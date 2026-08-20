# GSoC 2026 Final Work Submission

**Contributor:** Yug Agarwal ([@yug105](https://github.com/yug105))

**Organization:** MetaCall Technologies

**Project:** Extended Platform and Architecture Support for MetaCall

**Mentors:** [@pkspyder](https://github.com/pkspyder), [@Ashpect](https://github.com/Ashpect)

**Project size:** Medium (175 hours)

**Technologies:** C, C++, CMake, POSIX, Bash, GitHub Actions, ELF

**Topics:** embedded systems, systems programming, build systems, CI/CD, dynamic linking

**Synopsis:** [metacall/gsoc-2026 - Extended Platform and Architecture Support for MetaCall](https://github.com/metacall/gsoc-2026#9-extended-platform-and-architecture-support-for-metacall)

**Contact:** yugagarwal105@gmail.com

---

## 1. Project Goals

MetaCall is a polyglot runtime. It loads language runtimes as shared libraries and calls between them, which means it leans on `dlopen`, PLT/GOT hooking, and whatever the platform's dynamic linker happens to do. That makes it more sensitive to the operating system than most projects.

When I started, MetaCall built and tested on Linux (GCC), Windows (MSVC) and macOS. A few other platforms had bits of support sitting in the core layers, but none of it was tested. No CI, no section in the environment script, no way to know whether it still worked.

Goals:

- Add support for new platforms, with CI to prove it works.
- Keep all platform logic in `tools/metacall-environment.sh` and none in the CI YAML, so anyone can reproduce a build locally.
- Make sure CMake detection and the build actually work on each target.
- Get the test suite running on each platform and fix what it finds.
- Fix `metacall/plthook`, which MetaCall's detour layer depends on, for the platforms that need it.

### How I worked

Three passes per platform:

1. Add the CI workflow and let it fail. A real failure log beats guessing from the source.
2. Add the platform's section to `metacall-environment.sh` and fix CMake until it builds.
3. Fix whatever the tests report.

Then move to the next platform. Doing them one at a time kept the PRs small enough to review.

## 2. What I Did

### HP-UX

**[core #617](https://github.com/metacall/core/pull/617), merged.** HP-UX has no `dlopen`, it uses `shl_load`, `shl_findsym` and `shl_unload`. I added `dynlink_impl_hpux.c` and its header, wired through `dynlink_interface.h`, plus `PROJECT_OS_HPUX` and its own `hpux` OS family in `cmake/Portability.cmake`.

This set the pattern for the rest. Dynlink is an interface with one implementation per OS family, so adding a platform means adding an implementation, not `#ifdef`s.

### Android

**[core #620](https://github.com/metacall/core/pull/620), open.** Bionic's linker differs enough from glibc that the Unix dynlink path needs its own handling. This PR adds that handling. Still open.

**[plthook #17](https://github.com/metacall/plthook/pull/17), merged.** plthook's Android CI was broken before I could test the core side, so I fixed that first. Rewrote the test runner as `test/android/run_tests.sh`, fixed `Android.mk` and `Application.mk`, repaired the workflow.

### FreeBSD

The biggest part of the project.

**[plthook #19](https://github.com/metacall/plthook/pull/19), merged.** FreeBSD support in plthook, in `plthook_elf.c`. FreeBSD's linker exposes its link map differently from glibc, so the ELF walking needed its own path. Added CI alongside it.

**[plthook-poc #1](https://github.com/metacall/plthook-poc/pull/1), merged.** FreeBSD and NetBSD in the proof-of-concept harness.

**[core #762](https://github.com/metacall/core/pull/762), merged.** The FreeBSD CI pipeline. A `freebsd-test.yml` running a real FreeBSD VM through `cross-platform-actions`, with the FreeBSD setup added to `metacall-environment.sh` and nothing platform-specific in the YAML. Getting it to build also needed fixes in `cmake/FindNodeJS.cmake` and `portability_executable_path.c`, since FreeBSD gets the running executable's path from a `sysctl` rather than `/proc/self/exe`.

**[core #772](https://github.com/metacall/core/pull/772), merged.** More FreeBSD setup, plus fixes to `FindWasmtime.cmake` and `RustProject.cmake` so more loaders build.

**[plthook #23](https://github.com/metacall/plthook/pull/23), merged.** Tests passed on FreeBSD at `-O0` and `-O1` and failed at `-O2` and `-O3`. It was not a FreeBSD bug and not a compiler bug. It was undefined pointer arithmetic in plthook's own code. Making that arithmetic well-defined in `plthook_elf.c` fixed it at every level.

**[core #805](https://github.com/metacall/core/pull/805), merged.** FreeBSD CI with sanitizers turned on, which is where the harder failures show up. Included fixes in `plthook_detour_impl.c` and the WASM test.

### Haiku

**[plthook #24](https://github.com/metacall/plthook/pull/24), merged.** Haiku support in plthook, in `plthook_elf.c`, plus a Haiku workflow. Haiku is BeOS-derived rather than Unix-derived, so there is no `/proc`, no `getconf`, and the loader has its own API.

**[core #833](https://github.com/metacall/core/pull/833), merged.** Haiku CI for core plus the fixes needed to get it building. Three parts:

- A `haiku-test.yml` running Haiku r1beta5 across debug, relwithdebinfo and release.
- A change in `source/dynlink/CMakeLists.txt` selecting the `unix` dynlink implementation for Haiku. Haiku's OS family is `beos`, which would otherwise pick the old `load_add_on` backend, but Haiku does provide a working `dlopen`. Special-casing the selection was the right fix, the same way macOS is handled a few lines above.
- A fix in `rapid_json_serial_impl.cpp` moving the RapidJSON allocator into the document struct, so it dies with the document instead of outliving the plugin that owns it.

### MinGW / MSYS2

**[core #846](https://github.com/metacall/core/pull/846), open.** MinGW is Windows with a GCC toolchain and a POSIX-ish shell instead of MSVC. Adds a `windows-mingw-test.yml` and works through what breaks: flags in `CompileOptions.cmake` and `InstallGTest.cmake`, `log_policy_stream_syslog.c` (no syslog under MinGW), `metacall_link.c`, a guard so the backtrace plugin is skipped where it cannot build, and fixes in the fork and serial tests.

### Teardown reproducer

**[plthook #25](https://github.com/metacall/plthook/pull/25).** A standalone ThreadSanitizer reproducer for the destructor-after-unload crash, so the bug can be discussed against a small case instead of a full CI log.

## 3. Pull Requests

### GSoC coding period

| # | Repo | Title | State | Date |
|---|---|---|---|---|
| [#846](https://github.com/metacall/core/pull/846) | core | MinGW / MSYS2 support | Open | Jul 16, 2026 |
| [#833](https://github.com/metacall/core/pull/833) | core | Haiku CI, dynlink selection and RapidJSON allocator fix | Merged | Jul 8, 2026 |
| [#24](https://github.com/metacall/plthook/pull/24) | plthook | Add Haiku OS support | Merged | Jun 14, 2026 |
| [#805](https://github.com/metacall/core/pull/805) | core | FreeBSD CI with sanitizers | Merged | Jun 10, 2026 |

### Contributor period (before the coding period)

| # | Repo | Title | State | Date |
|---|---|---|---|---|
| [#23](https://github.com/metacall/plthook/pull/23) | plthook | Fix undefined behaviour in address arithmetic causing -O2/-O3 failures on FreeBSD | Merged | May 19, 2026 |
| [#772](https://github.com/metacall/core/pull/772) | core | FreeBSD build support, additional loaders | Merged | Apr 18, 2026 |
| [#762](https://github.com/metacall/core/pull/762) | core | Add FreeBSD CI | Merged | Apr 14, 2026 |
| [#1](https://github.com/metacall/plthook-poc/pull/1) | plthook-poc | FreeBSD and NetBSD support | Merged | Apr 7, 2026 |
| [#19](https://github.com/metacall/plthook/pull/19) | plthook | FreeBSD support | Merged | Apr 4, 2026 |
| [#17](https://github.com/metacall/plthook/pull/17) | plthook | Android CI fix | Merged | Mar 2, 2026 |
| [#620](https://github.com/metacall/core/pull/620) | core | Add Android platform support for dynlink module | Open | Jan 22, 2026 |
| [#617](https://github.com/metacall/core/pull/617) | core | Add HP-UX platform support for dynlink module | Merged | Jan 19, 2026 |

Live lists: [metacall/core](https://github.com/metacall/core/pulls?q=is%3Apr+author%3Ayug105) and [metacall/plthook](https://github.com/metacall/plthook/pulls?q=is%3Apr+author%3Ayug105).

## 4. Challenges and Lessons Learned

**Optimisation exposes undefined behaviour, it does not cause it.** plthook passed on FreeBSD at `-O0` and `-O1` and failed at `-O2` and `-O3`. The obvious conclusions are that the optimiser is wrong or that FreeBSD is doing something strange, and the obvious fix is to lower the optimisation level for that platform. Both are wrong. The real cause was undefined pointer arithmetic in plthook's own code that the compiler was allowed to assume never happened. If something only breaks at higher optimisation, it is usually a bug that was being masked, not one that was introduced.

**Two crashes on two operating systems can be the same bug.** The FreeBSD Python segfault and the Haiku crash looked unrelated. Different OS, different runtime, different stack trace. They are the same failure: something with a destructor, either a `thread_local` object or a runtime's own thread state, gets destroyed after the shared object holding its code has already been unloaded, so the call lands in memory that is no longer mapped. FreeBSD and Haiku both show it because their linkers unmap sooner than glibc does. Treating them as one bug instead of two turned a pair of unrelated CI failures into a single question about shutdown ordering. The RapidJSON fix in [#833](https://github.com/metacall/core/pull/833) is one case of it fixed properly.

**Platform logic belongs in the environment script, not in CI.** This was a constraint from the project description and it turned out to be the most useful rule in the project. Putting `pkg install` lines straight into `freebsd-test.yml` is faster and it works. But then the only way to build on FreeBSD is to push a commit and wait for a VM, and the only person who can debug a FreeBSD failure is someone willing to read the workflow file. Keeping it in `tools/metacall-environment.sh` means the CI job is a couple of commands anyone can run locally.

**Not every platform is Unix.** Haiku is BeOS-derived, so no `/proc`, no `getconf`, and its own loader API. Plenty of code that is portable across Linux, macOS and the BSDs does not apply. But it is not different in every way either. It does have a working `dlopen`, which is why the fix in [#833](https://github.com/metacall/core/pull/833) was to pick the `unix` implementation rather than write a whole new Haiku backend. Working out which differences are real and which are assumed is most of the work in a port.

**Break the CI first.** Adding a workflow that you know will fail feels backwards, but it was the right order every time. A failure log from a real FreeBSD 14.2 VM tells you which header is missing and which CMake check failed. Reading the source and writing a fix from a guess produces changes that look reasonable and break something else.

**Never disable something to get CI green.** The quickest way to a green MinGW build is to switch off whatever does not compile. Skipping the backtrace plugin under MinGW because it genuinely cannot build there is a documented limitation. Skipping a test because it fails is hiding a bug and handing it to whoever hits it next. The two look identical in a diff and the difference matters, and it is the maintainer's call rather than mine. Where I could not find the real fix, the right move was to report the error and ask.

---

Thanks to [@viferga](https://github.com/viferga), [@pkspyder](https://github.com/pkspyder) and [@Ashpect](https://github.com/Ashpect) for the mentorship.
