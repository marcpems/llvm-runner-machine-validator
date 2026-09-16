# llvm-runner-machine-validator

Fast, no-build machine-readiness validator for **self-hosted Windows runners** used to
build the `llvm-project` "Release Binaries" workflow.

Every check here maps directly to a real machine-configuration failure that has
actually broken a run in this environment. It never runs (or waits on) an LLVM build -
it only inspects and, where safe, repairs local machine state - so a full run takes
seconds, not hours. Use it to validate a brand-new runner machine **before** spending
6-10 hours discovering a missing tool at the very last step.

## Usage

```powershell
# Detect-only (safe, no changes made)
.\Validate-RunnerMachine.ps1

# Detect AND automatically fix anything that can be fixed safely
.\Validate-RunnerMachine.ps1 -ApplyFixes

# Point at a non-default runner install directory
.\Validate-RunnerMachine.ps1 -ApplyFixes -RunnerDir "E:\actions-runner"
```

Exit codes: `0` = ready, `1` = manual action required (see console output / report
JSON), `2` = the validator itself hit an internal error on a check.

A timestamped JSON report is written to `.\reports\validate-<timestamp>.json` on every
run for later reference / auditing.

## What it checks, and why (incident history)

| # | Check | Real incident it prevents |
|---|-------|----------------------------|
| 1 | VS2022 Build Tools (C++ workload) installed | Build needs an MSVC toolchain; missing = immediate cmake/MSBuild failure. |
| 2 | No conflicting newer VS toolchain (e.g. VS2026) present | A VS2026 install on the same machine was picked up ahead of VS2022, silently changing the MSVC ABI/runtime and breaking ASan interceptor tests (`Asan-x86_64-*-Dynamic-Test`, `memset_test.cpp`, `intercept_memcpy.cpp`, `dll_intercept_memcpy_indirect.cpp`). **Intel (x64) only** - auto-passes (skipped) on ARM64, since ASan interceptor tests are not built/run there and this failure mode cannot occur. |
| 3 | Git for Windows installed and first on PATH | Wrong git.exe / missing Git for Windows broke checkout/build tooling that shells out to `git`. |
| 4 | 7-Zip (`7z.exe`) installed | The final release-packaging step failed with `'7z' is not recognized as an internal or external command` **after** a full multi-hour build+test run had already completed. |
| 5 | 7-Zip directory present on machine-level PATH | `7z.exe` existed but wasn't reachable, same failure as above. **Known quirk:** a machine PATH change is not picked up by an already-running process (including an already-running runner) until it is restarted - the script calls this out explicitly. |
| 6 | GitHub Actions Runner process running | If the listener isn't running, the machine can't pick up any dispatched job at all. |
| 7 | ASan known-failing-test exclusion overlay (git hook) installed | 5 specific ASan interceptor tests are known-failing in this environment (not real code regressions). A machine-local, never-committed `post-checkout` git hook appends them to `LIT_FILTER_OUT` in `build_llvm_release.bat` after every checkout, since that script hard-overwrites the variable with a literal string. This is a transient, per-machine overlay - it never touches the repo/branch. **Intel (x64) only** - auto-passes (skipped) on ARM64, since these ASan tests are not built/run there and the overlay is unnecessary. |

## Design principles

* **One check failing never stops the others.** Every check is wrapped so an
  exception in Detect or Fix is caught, logged as `CheckError`, and the script moves
  on to the next check.
* **Nothing destructive is auto-applied.** Fixes that would be slow/interactive
  (installing Visual Studio) or destructive/risky (uninstalling a VS instance, freeing
  disk space) are always reported as **manual action required**, with a clear
  explanation of *why it matters* and *exactly what to do*, never silently skipped.
* **Detect-only by default.** Nothing changes on the machine unless you pass
  `-ApplyFixes`.
* **Architecture-aware.** Checks #2 and #7 exist solely to prevent ASan interceptor
  test failures, and ASan interceptor tests are only built/run on Intel (x64). The
  script detects ARM64 machines (`$env:PROCESSOR_ARCHITECTURE` /
  `RuntimeInformation.ProcessArchitecture`) and automatically skips (auto-passes)
  those two checks there, so an ARM64 runner is never flagged for, or has fixes
  applied for, an issue that cannot occur on it.
* **Self-contained.** Single PowerShell script, no external dependencies beyond
  tools already expected on a build machine (`git`, `winget`, `vswhere.exe` if VS is
  present).

## Known limitations / things intentionally NOT automated

* Visual Studio installation/uninstallation (checks 1 & 2) - too slow/interactive/
  destructive to run unattended. The script tells you exactly what's wrong and what to
  install/remove.
* Disk space cleanup (check 8) - flagged only; freeing space requires human judgment
  about what's safe to delete.
* If a PATH-related fix is applied while the runner is already running, **you must
  restart the runner** for it to take effect - the script prints this reminder and the
  "GitHub Actions Runner is running" check's `-ApplyFixes` fix will relaunch it with
  the corrected PATH explicitly applied to that process tree, sidestepping the PATH-
  propagation quirk observed in this environment.
