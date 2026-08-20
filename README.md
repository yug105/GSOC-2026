# GSoC 2026 Final Work Submission

**Contributor:** Yug Agarwal ([@yug105](https://github.com/yug105))

**Organization:** MetaCall Technologies

**Project:** Extended Platform and Architecture Support for MetaCall

**Mentors:** [@pkspyder](https://github.com/pkspyder), [@Ashpect](https://github.com/Ashpect)

**Project size:** Medium (175 hours)

**Technologies:** C, C++, CMake, POSIX, Bash, GitHub Actions, ELF

**Topics:** embedded systems, systems programming, build systems, CI/CD, dynamic linking

**Synopsis:** [metacall/gsoc-2026 - Extended Platform and Architecture Support for MetaCall](https://github.com/metacall/gsoc-2026#9-extended-platform-and-architecture-support-for-metacall)

**Contact:** 2023uee1378@mnit.ac.in

---

## 1. Project Goals

MetaCall is a polyglot runtime that loads language runtimes as shared-library plugins and marshals calls between them. Because it does that with `dlopen`, PLT/GOT hooking and per-platform dynamic linker behaviour, it is unusually sensitive to the operating system it runs on. At the start of this project MetaCall built and tested on Linux (GCC), Windows (MSVC) and macOS. That leaves out most of the platforms where a polyglot runtime is actually interesting: BSD servers, embedded and mobile targets, and the whole MinGW/MSYS2 toolchain on Windows.

Some of these platforms had partial support already sitting in the core layers, but partial support that nobody tests is indistinguishable from no support. There was no CI, no section in the environment script, and no way to know whether a given platform still built.

The project set out to:

- Add first-class support for new platform targets, driven by CI rather than by claims in a README.
- Keep every piece of platform logic inside `tools/metacall-environment.sh`, with no hardcoded setup in the CI YAML, so that a developer can reproduce a FreeBSD or Haiku build locally with the same commands CI runs.
- Verify CMake platform detection and the build actually work on each target, not just that they configure.
- Get the test suite running on each new platform and fix what it uncovers.
- Fix `metacall/plthook`, the PLT/GOT hooking library MetaCall's detour layer depends on, for the platforms that need it.

### Method

I followed a TDD approach at the level of platforms rather than functions, in three passes per target:

1. **CI first.** Add the workflow and let it fail. This produces a real, reproducible failure log instead of a guess about what the platform needs.
2. **Environment script and build system.** Add the platform's section to `metacall-environment.sh` and fix CMake detection until the build completes.
3. **Fix the test failures.** Work through what the suite reports, which is where the genuinely interesting bugs were.

Then move to the next platform. Going platform by platform rather than in parallel kept each PR small enough to be reviewed properly.

## 2. What I Did

### HP-UX dynamic linking (January)

**[core #617](https://github.com/metacall/core/pull/617), merged.** HP-UX does not use `dlopen`. It uses the `shl_load` family (`shl_load`, `shl_findsym`, `shl_unload`), a completely separate API from POSIX `dlfcn`. I added a new dynlink backend for it: `dynlink_impl_hpux.c` (158 lines) plus its header, wired through `dynlink_interface.h`, and added `PROJECT_OS_HPUX` detection with a distinct `hpux` OS family in `cmake/Portability.cmake` so `source/dynlink/CMakeLists.txt` selects the right implementation.

This was the first piece of the project and it set the pattern for everything after: MetaCall's dynlink layer is an interface with one implementation per OS family, and adding a platform means adding an implementation, not adding `#ifdef`s to an existing one.

### Android dynlink (January to February)

**[core #620](https://github.com/metacall/core/pull/620), open.** Android's Bionic linker behaves differently enough from glibc that the Unix dynlink path needs its own handling. This PR adds that support (+1101 lines across 10 files). It is still open pending review.

**[plthook #17](https://github.com/metacall/plthook/pull/17), merged.** Before the core side could be tested, plthook's own Android CI was broken. I rewrote the Android test runner as a proper shell script (`test/android/run_tests.sh`, replacing the extensionless `run_tests`), fixed `Android.mk` and `Application.mk`, and repaired the CI workflow.

### FreeBSD (March to June)

This was the largest single thread of the project and it went through the full three-pass cycle.

**[plthook #19](https://github.com/metacall/plthook/pull/19), merged.** FreeBSD support in plthook itself, +277 lines in `plthook_elf.c`. The FreeBSD dynamic linker exposes its link map differently from glibc, so the ELF walking code needed a separate path, plus a CI workflow to prove it works.

**[plthook-poc #1](https://github.com/metacall/plthook-poc/pull/1), merged.** FreeBSD and NetBSD support in the proof-of-concept harness.

**[core #762](https://github.com/metacall/core/pull/762), merged.** The FreeBSD CI pipeline: a 46-line `freebsd-test.yml` running a real FreeBSD VM through `cross-platform-actions`, with 29 lines of FreeBSD setup added to `metacall-environment.sh` and nothing platform-specific left in the YAML. Getting the build green also required fixing `cmake/FindNodeJS.cmake` and `source/portability/portability_executable_path.c`, since FreeBSD retrieves the running executable's path through a `sysctl` rather than `/proc/self/exe`.

**[core #772](https://github.com/metacall/core/pull/772), merged.** Extended the FreeBSD environment section by another 27 lines and fixed `FindWasmtime.cmake` and the Rust `RustProject.cmake` so more loaders build on FreeBSD.

**[plthook #23](https://github.com/metacall/plthook/pull/23), merged.** The most instructive bug of the project. plthook's tests passed on FreeBSD at `-O0` and `-O1` but failed at `-O2` and `-O3`. The cause was not a FreeBSD bug and not a compiler bug: it was undefined behaviour in plthook's own pointer arithmetic, which the optimiser was entitled to assume could not happen. A 13-line change in `plthook_elf.c` made the arithmetic well-defined and the failures disappeared at every optimisation level.

**[core #805](https://github.com/metacall/core/pull/805), merged.** FreeBSD CI with sanitizers enabled, which is where the harder failures surface. Included a fix in `plthook_detour_impl.c` and in the WASM test.

The FreeBSD pipeline on `develop` now runs three build types across x86-64 and arm64, with clean, AddressSanitizer and ThreadSanitizer variants, loading Python, C, WASM, Ruby, Go, Java, COBOL, file and RPC loaders.

### Haiku (June to July)

**[plthook #24](https://github.com/metacall/plthook/pull/24), merged.** Haiku support in plthook, +189 lines in `plthook_elf.c` plus a Haiku CI workflow. Haiku is BeOS-derived rather than Unix-derived, so assumptions that hold on every other target here do not hold: no `/proc`, no `getconf`, and a runtime loader with its own API.

**[core #833](https://github.com/metacall/core/pull/833), merged.** Haiku CI for MetaCall core, plus the dynlink and serialization fixes needed to get it building. Three parts:

- A 47-line `haiku-test.yml` running Haiku r1beta5 in a VM across debug, relwithdebinfo and release.
- Three lines in `source/dynlink/CMakeLists.txt` selecting the `unix` dynlink implementation for Haiku. This is the non-obvious part: Haiku's OS family is `beos`, which would normally select the legacy `load_add_on` backend, but Haiku does provide a working `dlopen`. The right fix was to keep the family correct and special-case the dynlink selection, mirroring how macOS is handled a few lines above.
- A fix in `rapid_json_serial_impl.cpp` moving the RapidJSON allocator into the document struct, so the allocator's lifetime is tied to the document rather than outliving the plugin that owns it.

That last change came out of a crash class that took most of July to understand, described in section 6.

### MinGW / MSYS2 (July)

**[core #846](https://github.com/metacall/core/pull/846), open.** MinGW is Windows without MSVC, which means Windows headers and semantics but a GCC toolchain and a POSIX-ish shell. This PR adds a 58-line `windows-mingw-test.yml` and works through what breaks: `CompileOptions.cmake` and `InstallGTest.cmake` flags, `log_policy_stream_syslog.c` (no syslog under MinGW), `metacall_link.c`, a guard so the backtrace plugin is skipped where it cannot build, and fixes in the fork and serial tests. Currently open for review.

### Teardown race investigation (July, ongoing)

**[plthook #25](https://github.com/metacall/plthook/pull/25).** A standalone ThreadSanitizer reproducer for the destructor-after-unload crash class described in section 6, built so the bug can be discussed against a minimal case instead of against a full MetaCall CI log.

## 3. Current State

Merged and running on `metacall/core:develop` today:

- **`freebsd-test.yml`** runs FreeBSD 14.2 across three build types, x86-64 and arm64, with clean, AddressSanitizer and ThreadSanitizer variants, exercising nine loaders. This workflow did not exist before this project. The plain builds pass on both architectures; the sanitizer and release variants are currently red on the outstanding teardown issues listed in section 4, which is exactly what the pipeline was built to expose.
- **`haiku-test.yml`** runs Haiku r1beta5 on x86-64 across three build types. Did not exist before this project. It is currently red on the destructor-after-unload crash described in section 6; the workflow reports a crash dump rather than hanging, which is what makes that bug tractable at all.
- **FreeBSD and Haiku sections in `tools/metacall-environment.sh`**, so both platforms are reproducible locally with the same three-script pipeline used everywhere else. No platform setup lives in the workflow YAML.
- **HP-UX dynlink backend** (`dynlink_impl_hpux.c`) and `PROJECT_OS_HPUX` detection.
- **Haiku dynlink selection** and the RapidJSON allocator lifetime fix.
- **plthook** has merged FreeBSD, Haiku and Android CI support, and the `-O2`/`-O3` undefined-behaviour fix.

Open and awaiting review:

- MinGW/MSYS2 support ([core #846](https://github.com/metacall/core/pull/846)).
- Android dynlink support ([core #620](https://github.com/metacall/core/pull/620)).

## 4. What's Left

| Item | Status |
|---|---|
| MinGW / MSYS2 support ([#846](https://github.com/metacall/core/pull/846)) | Implemented, workflow added, awaiting maintainer review |
| Android dynlink support ([#620](https://github.com/metacall/core/pull/620)) | Implemented, awaiting review |
| Android CI for core via `CMAKE_CROSSCOMPILING_EMULATOR` + `adb` | In progress. Running the test suite on a device or emulator from CMake, rather than only cross-compiling |
| FreeBSD Python teardown segfault | Root cause identified (see section 6). The fix touches loader shutdown ordering, which is a maintainer decision rather than mine to make unilaterally |
| Haiku TLS destructor crash | Same root cause family. Partially mitigated by the RapidJSON fix in [#833](https://github.com/metacall/core/pull/833); the general case remains |
| Cygwin support | Not started |
| NetBSD support in core | Landed in `plthook-poc` only; not yet carried into core |

I want to be straightforward about the two crash items. Both are understood, both are documented with reproducers, and neither is a case of the platform being unsupported. They are pre-existing lifetime bugs in MetaCall's teardown path that only these platforms were strict enough to expose.

## 5. Pull Requests

### GSoC coding period

| # | Repo | Title | State | Date | Diff |
|---|---|---|---|---|---|
| [#846](https://github.com/metacall/core/pull/846) | core | MinGW / MSYS2 support | Open | Jul 16, 2026 | +99 / -11 |
| [#833](https://github.com/metacall/core/pull/833) | core | Haiku CI, dynlink selection and RapidJSON allocator fix | Merged | Jul 8, 2026 | +59 / -13 |
| [#24](https://github.com/metacall/plthook/pull/24) | plthook | Add Haiku OS support | Merged | Jun 14, 2026 | +240 / -8 |
| [#805](https://github.com/metacall/core/pull/805) | core | FreeBSD CI with sanitizers | Merged | Jun 10, 2026 | +18 / -6 |

### Contributor period (before the coding period)

| # | Repo | Title | State | Date | Diff |
|---|---|---|---|---|---|
| [#23](https://github.com/metacall/plthook/pull/23) | plthook | Fix undefined behaviour in address arithmetic causing -O2/-O3 failures on FreeBSD | Merged | May 19, 2026 | +15 / -13 |
| [#772](https://github.com/metacall/core/pull/772) | core | FreeBSD build support, additional loaders | Merged | Apr 18, 2026 | +41 / -4 |
| [#762](https://github.com/metacall/core/pull/762) | core | Add FreeBSD CI | Merged | Apr 14, 2026 | +100 / -22 |
| [#1](https://github.com/metacall/plthook-poc/pull/1) | plthook-poc | FreeBSD and NetBSD support | Merged | Apr 7, 2026 | +48 / -1 |
| [#19](https://github.com/metacall/plthook/pull/19) | plthook | FreeBSD support | Merged | Apr 4, 2026 | +343 / -24 |
| [#17](https://github.com/metacall/plthook/pull/17) | plthook | Android CI fix | Merged | Mar 2, 2026 | +94 / -78 |
| [#620](https://github.com/metacall/core/pull/620) | core | Add Android platform support for dynlink module | Open | Jan 22, 2026 | +1101 / -1 |
| [#617](https://github.com/metacall/core/pull/617) | core | Add HP-UX platform support for dynlink module | Merged | Jan 19, 2026 | +229 / -0 |

Live lists:

- [metacall/core](https://github.com/metacall/core/pulls?q=is%3Apr+author%3Ayug105)
- [metacall/plthook](https://github.com/metacall/plthook/pulls?q=is%3Apr+author%3Ayug105)

## 6. Challenges and Lessons Learned

**Optimisation levels expose undefined behaviour, they do not cause it.** plthook passed its tests on FreeBSD at `-O0` and `-O1` and failed at `-O2` and `-O3`. The tempting conclusion is that the optimiser is wrong, or that FreeBSD is doing something unusual, and the tempting fix is to lower the optimisation level for that platform. Both are wrong. The real cause was undefined pointer arithmetic in `plthook_elf.c` that the compiler was allowed to assume never happened. [plthook #23](https://github.com/metacall/plthook/pull/23) fixed the arithmetic in 13 lines and the failures went away at every level. The general lesson is that a bug which appears only at higher optimisation is almost always a latent bug in your own code that was previously being masked.

**Two crashes on two operating systems can be one bug.** The FreeBSD Python segfault and the Haiku crash looked unrelated: different OS, different language runtime, different stack trace. They are the same failure mode. Something with a destructor, a `thread_local` object or a runtime's own thread state, is destroyed after the shared object that owns its code has already been unloaded, so the destructor call jumps into memory that is no longer mapped. FreeBSD and Haiku both surface it because their linkers unmap more eagerly than glibc does. Recognising these as one family, rather than two platform bugs, is what turned an intractable pair of CI failures into a single question about teardown ordering. The RapidJSON allocator fix in [#833](https://github.com/metacall/core/pull/833) is one instance of it fixed properly, by tying the allocator's lifetime to the document that uses it.

**Platform logic belongs in the environment script, not in CI.** This was a hard constraint from the project description and it turned out to be the single most useful rule in the project. It is much faster to put `pkg install` lines directly in `freebsd-test.yml`, and it produces something that works. But then the only way to build MetaCall on FreeBSD is to push a commit and wait for a VM, and the only person who can debug a FreeBSD failure is someone who can read the workflow file. Keeping every platform section inside `tools/metacall-environment.sh` means the CI job is three lines that anyone can run locally, and the platform knowledge lives somewhere a contributor will actually find it.

**Not every platform is Unix.** Haiku is BeOS-derived. It has no `/proc`, no `getconf`, and its own loader API, so a large amount of code that is portable across Linux, macOS and the BSDs simply does not apply. But it is also not uniformly different: it does provide a working `dlopen`, which is why the right fix in [#833](https://github.com/metacall/core/pull/833) was three lines selecting the `unix` dynlink implementation for Haiku specifically, rather than writing a whole Haiku backend. Working out which of a platform's differences are real and which are assumed is most of the work in porting.

**Get the CI failing before writing the fix.** Adding the workflow first felt backwards, since it means deliberately pushing something that goes red. It was the right order every time. A real failure log from a real FreeBSD 14.2 VM tells you exactly which header is missing and which CMake check failed. Guessing from the source and writing a speculative fix produces changes that look plausible, pass review on vibes, and then break something else. Every platform in this project followed the same loop: red CI, then environment script, then build system, then test failures.

**Never disable a feature to make CI green.** The fastest route to a green MinGW build is to switch off whatever does not compile. I did that early on and was rightly pulled up for it. Turning off the backtrace plugin under MinGW because it genuinely cannot build there is a documented limitation; turning off a test because it fails is hiding a bug and shifting it onto whoever hits it next. The distinction matters and it is the reviewer's call, not the contributor's. When I could not find the real fix, the correct move was to report the error and ask.

**Small PRs get reviewed, large ones do not.** My first attempts at both Android ([#619](https://github.com/metacall/core/pull/619), [#621](https://github.com/metacall/core/pull/621)) and Haiku ([#819](https://github.com/metacall/core/pull/819), 23 files) were closed. The Haiku work that eventually merged was [#833](https://github.com/metacall/core/pull/833) at three files. Same problem, same understanding, a fraction of the diff, and it went in. A PR that touches 23 files across dynlink, CMake, CI and the serialization layer is not one change, it is six changes that a reviewer has to untangle before they can even start.

---

Thanks to [@viferga](https://github.com/viferga) for reviewing and merging this work, and to [@pkspyder](https://github.com/pkspyder) and [@Ashpect](https://github.com/Ashpect) for mentoring the project. The review style here is Socratic rather than prescriptive, which was uncomfortable at first and is the reason I understand the dynamic linker instead of just having patched around it.
