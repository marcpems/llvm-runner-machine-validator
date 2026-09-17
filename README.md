# llvm-runner-machine-validator

Fast, no-build machine-readiness validator for **self-hosted Windows runners** used to
build the `llvm-project` "Release Binaries" workflow.

Every check here maps directly to a real machine-configuration failure that has
actually broken a run in this environment, and/or a documented prerequisite of
[`llvm/utils/release/build_llvm_release.bat`](https://github.com/llvm/llvm-project/blob/main/llvm/utils/release/build_llvm_release.bat) -
the official Windows release-build script (Visual Studio, CMake, Ninja, Git, Bash,
Python, Perl, SWIG, `mv`/`tar`, curl, 7-Zip). It never runs (or waits on) an LLVM build -
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
| 1 | Chocolatey (`choco.exe`) installed and configured for non-interactive (CI) use | winget is not present by default on many Windows Server-based self-hosted runner images, and several LLVM build prerequisites (GNUWin32 `patch`/`diff`, Subversion, NSIS for the installer packaging step) are not published on winget at all - Chocolatey is the standard fallback package manager for these. Also verifies the `allowGlobalConfirmation` feature is enabled, since without it every `choco install` hangs on an interactive `[Y]es/[A]ll/[N]o/[P]rint` prompt and eventually fails with `Too many bad attempts. Stopping before application crash.` on a non-interactive runner. Auto-fix bootstraps Chocolatey via the official install script (requires an elevated/Administrator process) and enables the feature. |
| 2 | No stale Chocolatey pending-install lock files | A `.chocolateyPending` marker left behind by a previously interrupted `choco install`/`choco upgrade` blocks EVERY subsequent Chocolatey install (not just the originally-affected package) with an error like `the process cannot access the file ...\.chocolateyPending because it is being used by another process`, silently breaking every other check whose auto-fix relies on Chocolatey. Auto-fix removes any stale marker files found under `lib`/`lib-bad`. |
| 3 | VS2022 Build Tools (C++ workload, host-architecture-matched toolset) installed | Build needs an MSVC toolchain; missing = immediate cmake/MSBuild failure. **Architecture-aware:** on x64 hosts requires the `VC.Tools.x86.x64` component; on ARM64 hosts requires `VC.Tools.ARM64` specifically (the x86.x64 component alone does not guarantee a working native `HostARM64\ARM64\cl.exe`) and verifies `cl.exe` actually exists on disk for the host's `Host<Arch>\<Arch>` toolset directory, not just that vswhere reports the component as installed. |
| 4 | No conflicting newer VS toolchain (e.g. VS2026) present | A VS2026 install on the same machine was picked up ahead of VS2022, silently changing the MSVC ABI/runtime and breaking ASan interceptor tests (`Asan-x86_64-*-Dynamic-Test`, `memset_test.cpp`, `intercept_memcpy.cpp`, `dll_intercept_memcpy_indirect.cpp`). **Intel (x64) only** - auto-passes (skipped) on ARM64, since ASan interceptor tests are not built/run there and this failure mode cannot occur. |
| 5 | CMake installed (minimum version) | LLVM's configure step is driven by CMake. Missing or too-old `cmake.exe` fails the configure step immediately with `'cmake' is not recognized` or a `CMake x.y or higher is required` error, before any compilation starts. |
| 6 | Ninja installed | The LLVM release build is configured with `-G Ninja`. If `ninja.exe` is missing/not on PATH, CMake configure fails immediately with `CMake Error: ... generator Ninja is not installed`, before any compilation starts. |
| 7 | clang-cl (recent LLVM release) available | If `clang-cl --version` and `lld-link --version` both succeed, `build_llvm_release.bat` uses clang-cl+lld-link (instead of plain MSVC) to build the stage0 bootstrap compiler - noticeably faster, and how upstream official Windows releases are actually built. Not fatal if missing (silently falls back to MSVC), but a missing/stale clang-cl silently degrades every build to the slower path with no warning ever surfaced. |
| 8 | Python 3 installed | CMake's Python3 detection (used by LLDB and other components) requires a working `python.exe`. The official `build_llvm_release.bat` expects Python 3.11 at a fixed per-user path unless run with `--local-python`. A missing/broken Python install fails CMake configuration. |
| 9 | Perl installed | LLVM's OpenMP runtime build shells out to `perl` during its configure/generation steps. Without `perl.exe` on PATH, building the `runtimes` (openmp) component fails. |
| 10 | SWIG installed | LLDB's build generates its Python scripting bindings via SWIG. Without `swig.exe` on PATH, configuring/building the `lldb` project fails. The official script notes SWIG 4.1.1 specifically should be used. |
| 11 | Git for Windows installed and first on PATH | Wrong git.exe / missing Git for Windows broke checkout/build tooling that shells out to `git`. |
| 12 | Bash available | Some LLVM release build/test steps shell out to `bash` (e.g. lit-driven test-suite scripts and helper wrappers). If `bash.exe` isn't resolvable, those steps fail with `'bash' is not recognized` partway through a multi-hour build/test run. Bash normally ships with Git for Windows at `C:\Program Files\Git\bin\bash.exe`. |
| 13 | GNU-style `mv` and `tar` utilities available | `build_llvm_release.bat` directly shells out to `mv` (to rename the extracted source archive) and `tar` (to unpack downloaded libxml2/zlib/zstd tarballs). Neither is a built-in cmd.exe command; `mv` in particular has no Windows-native equivalent and is normally supplied by Git for Windows' `usr\bin`. |
| 14 | curl available | The official release script uses `curl` to download the LLVM source archive and dependency tarballs (libxml2/zlib/zstd). Without `curl.exe` on PATH, the build fails immediately at the first download step. Ships built-in on Windows 10 (1803+)/11. |
| 15 | 7-Zip (`7z.exe`) installed (correct version) | The final release-packaging step failed with `'7z' is not recognized as an internal or external command` **after** a full multi-hour build+test run had already completed. Also enforces the official script's requirement that 7-Zip be 20.x or older UNLESS the process is elevated, since 7-Zip 21.x+ tries to extract symlinks from LLVM's git archive (which needs Administrator rights). |
| 16 | 7-Zip directory present on machine-level PATH | `7z.exe` existed but wasn't reachable, same failure as above. **Known quirk:** a machine PATH change is not picked up by an already-running process (including an already-running runner) until it is restarted - the script calls this out explicitly. |
| 17 | GitHub Actions Runner process running | If the listener isn't running, the machine can't pick up any dispatched job at all. |
| 18 | ASan known-failing-test exclusion overlay (git hook) installed | 5 specific ASan interceptor tests are known-failing in this environment (not real code regressions). A machine-local, never-committed `post-checkout` git hook appends them to `LIT_FILTER_OUT` in `build_llvm_release.bat` after every checkout, since that script hard-overwrites the variable with a literal string. This is a transient, per-machine overlay - it never touches the repo/branch. **Intel (x64) only** - auto-passes (skipped) on ARM64, since these ASan tests are not built/run there and the overlay is unnecessary. |

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
* **Architecture-aware.** Checks #4 and #18 exist solely to prevent ASan interceptor
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
