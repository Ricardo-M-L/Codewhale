# Environment-specific caveats

> 阅读简体中文版：[zh_hans/ENVIRONMENTS.md](zh_hans/ENVIRONMENTS.md)。

Standard build/test/run commands live in `AGENTS.md` and `CONTRIBUTING.md`.
This file records only the non-obvious quirks of particular environments, so
they do not cost context on machines that will never hit them.

## Cursor Cloud VMs

- **System build dep:** the build needs `libdbus-1-dev` (pulled in by
  `crates/secrets` for the OS keyring). It is installed by the startup update
  script; if a `cargo build` fails with a `dbus`/`pkg-config` error, that dep is
  missing.
- **`rustup default` must be set:** some tests and runtime paths spawn shells in
  temp dirs *outside* this checkout (e.g. `run_verifiers_background_*`, sub-agent
  worktrees). Those spawned shells only see the repo's `rust-toolchain.toml`
  override while inside `/workspace`, so without a global default they fail with
  "rustup could not choose a version of rustc to run". The update script runs
  `rustup default stable` to fix this.
- **Known env-specific test failures at `/workspace` (not code bugs):** because
  the checkout sits directly under `/`, two `codewhale-tui` subagent tests fail
  here — `git_repo_root_reports_attempted_paths_when_no_repo_found` (cannot
  create a temp dir in the unwritable parent `/`) and
  `create_isolated_worktree_reports_friendly_error_when_no_repo_found` (walking
  up to `/` discovers `/workspace` itself as a repo). Both pass when the repo is
  checked out under a normal, writable parent.

## Running the agent without provider API keys

Point Codewhale at any local OpenAI-compatible endpoint via the keyless
`vllm`/`ollama`/`sglang` providers:

```sh
CODEWHALE_PROVIDER=vllm VLLM_BASE_URL=http://127.0.0.1:8000/v1 VLLM_MODEL=<id> \
  codewhale exec --auto "..."
```

`codewhale exec` (add `--auto` for tool use) is the non-interactive path to
exercise the full agent loop.

## Keeping the host awake during a turn

While an interactive TUI turn is in flight, Codewhale holds the platform's
idle-sleep assertion, so an unattended machine does not idle into sleep
mid-turn and lose the work:

- macOS: `caffeinate -i`
- Linux: `systemd-inhibit --what=idle --why="Codewhale turn in flight" --mode=block cat`,
  where `cat` reads a pipe Codewhale holds for the turn

The assertion is released the moment the turn ends — on Linux by closing that
pipe, so `cat` exits and `systemd-inhibit` follows without leaving a process
behind — and it covers *idle* sleep only: an explicit `sleep` / `pmset sleepnow`, a closed lid, or a low battery
still suspends the machine. Headless hosts — `exec`, app-server, CI — never
hold it, so a shared runner's power policy is untouched. Windows is not
implemented: `SetThreadExecutionState` is thread-affine and needs a holder that
pins the thread, so the gap is deliberate rather than silent.

If a turn is suspended anyway, the engine notices on wake — wall-clock elapsed
diverging from monotonic elapsed by more than the suspend threshold — reports
`System sleep detected; connection lost — retrying request`, and re-issues the
request instead of failing the turn (#2990).

## Windows support and release qualification

Windows compatibility depends on the runtime architecture and terminal. The
native x64 CLI/TUI compatibility target is Windows 10 1607 (build 14393) or
later in the legacy Windows console. A release's actual executable and installer
must still pass startup and installation on that target; an import-table pass
alone does not prove it. Required Visual C++ runtime DLLs must also be present.

| Surface | Compatibility target or platform requirement | Qualification boundary |
| --- | --- | --- |
| Native x64 CLI/TUI | Windows 10 x64 1607, build 14393 | Final release bytes need an actual oldest-OS startup receipt |
| ConPTY API | Windows 10 1809, build 17763; Windows Server 2019 | This is Microsoft's API minimum; Codewhale's terminal-session routes remain Unix-only and return 501 on Windows |
| Current Windows Terminal | Windows 10 2004, build 19041, or later | This terminal has a newer prerequisite than the ConPTY API; use the legacy console for the x64 compatibility target |
| Native ARM64 CLI/TUI | Windows 11 ARM64 is the target | Candidate evidence exists on build 26300; no minimum Windows 11 build or Windows 10 ARM64 support is inferred from that run |
| NSIS installer | x86 setup/plugins under WOW64, installing x64 runtime bytes | Native ARM64 installation uses its archive/npm assets; generated Uninstall.exe and actual install behavior need separate receipts |

[Microsoft's ConPTY requirements](https://learn.microsoft.com/en-us/windows/console/createpseudoconsole)
and [current Terminal requirements](https://github.com/microsoft/terminal/blob/main/README.md)
are separate. Windows 11 supports x64 emulation on ARM64; an emulated x64 run
does not qualify native x64 execution or Windows 10 1607.

The release import guard uses separate per-machine profiles in
[`windows-import-floor.json`](../scripts/release/windows-import-floor.json).
Unknown import pairs, unlisted ordinals and delay-loaded dependencies stop the
gate. Its PASS covers the reviewed static import contract; it does not certify
all runtime features, the installed dependency closure or a minimum OS build.
The Native extension host still requires its runtime capability APIs and
isolation probes; where DeriveCapabilitySidsFromName is absent it reports
Unsupported rather than weakening its sandbox. This feature limit is distinct
from CLI startup compatibility.

### Recorded 0.10.2 candidate evidence

The 2026-10-13 qualification snapshot is source
[`cf7199b52e`](https://github.com/codewhale-hq/Codewhale/commit/cf7199b52ecb9147e4670ae7874924ffc30a6a67).
A default-feature debug ARM64 build on Windows 11 build 26300 produced executable
SHA256 `ca51f91562a0569ee43855715856e1cfee74e8a834bd6206183a3175373fdb2d`.
All 453 imported symbols resolved in a native ARM64 process using system-only
DLL lookup. The ordinary-user candidate checks passed 89 classifier tests and
one owner-crash recovery test, plus scoped editor wheel, Pinyin and undo checks.
This is neither a dist-profile release artifact nor installer acceptance, and
it establishes no oldest supported Windows 11 build.

The x64 profile comes from @LmeSzinc's
[#6953 export investigation](https://github.com/codewhale-hq/Codewhale/issues/6953)
on build 14393.447: all 411 v0.10.0 imports and 453 of 454 v0.10.1 imports resolved.
The missing DeriveCapabilitySidsFromName pair is excluded; the conservative
allowed union has 456 pairs. The x86 NSIS profile has 189 named Microsoft API
contracts predating 1607 and one explicitly identified ordinal: COMCTL32 #17 is
InitCommonControls in the x86 Windows SDK 10.0.26100.0 import library and stock
SysWOW64 DLL. Its [documented minimum is Vista](https://learn.microsoft.com/en-us/windows/win32/api/commctrl/nf-commctrl-initcommoncontrols).
The manifest retains official source URLs, image hashes, counts and the ordinal
identity; these qualify static contracts without claiming a 1607 installer run.

At this snapshot, startup/installation of the final release bytes on an actual
1607 machine and the candidate's generated uninstaller were still unqualified.
The final frozen source, release import dumps, archive/native/npm installation,
packaged/native-app owner recovery and oldest-OS receipts belong in the release
acceptance packet. See [the release checklist](RELEASE_CHECKLIST.md).

## Windows PowerShell execution policy

The shell tool runs PowerShell with `-ExecutionPolicy Bypass`. That sets only
the policy of the child process it launches: nothing is persisted, no
administrator rights are needed, and your own PowerShell windows keep their
policy. Without it, a machine whose local policy is `Restricted` (the Windows
client default) or `AllSigned` refuses the temporary `.ps1` script Codewhale
writes for multiline commands (#6745).

A policy set by Group Policy (the `MachinePolicy` or `UserPolicy` rows of
`Get-ExecutionPolicy -List`) outranks the process scope. On such a machine,
multiline commands are still refused and PowerShell's refusal is returned as
the command's error; Codewhale does not work around an administrator-enforced
policy. Single-line commands run through `-Command`, which the execution
policy does not govern. Scripts that a command itself calls run under the same
process scope; the shell tool's approval and sandbox settings, not the
execution policy, decide what may run.

To let the machine or user policy apply instead, set
`CODEWHALE_POWERSHELL_EXECUTION_POLICY=inherit` before starting Codewhale: the
shell tool then omits `-ExecutionPolicy` entirely, so a `Restricted` or
`AllSigned` policy refuses multiline commands that need the temporary `.ps1`
script. Unset, `bypass`, or any other value keeps the default `Bypass`.

## Consolidated runtime commands

The current `codewhale` binary runs the TUI in-process. Release installers copy
the same bytes to the optional `codew` short command; no sibling
`codewhale-tui` executable is required. `DEEPSEEK_TUI_BIN` remains a legacy
replay/migration setting, not a current install requirement.
