# WeKan fork of Node.js — fork changelog

This file lists what the [wekan/node](https://github.com/wekan/node) fork adds on
top of upstream Node.js. Upstream's own changelog is untouched and lives in
`CHANGELOG.md` and `doc/changelogs/`.

**Why the fork exists.** Some CPU/OS combinations have no published Node.js at
all: nodejs.org does not build them and
[unofficial-builds](https://unofficial-builds.nodejs.org/) does not either. A
machine with no runtime cannot run WeKan, however well WeKan is packaged, and
[wekan/FerretDB](https://github.com/wekan/FerretDB) already builds a database
for those same platforms. So this fork builds the runtime to go with it.

**Base.** `v24.x`, on upstream
[v24.18.1 'Krypton' (LTS)](https://github.com/nodejs/node/commit/9623d9ad85d37d2f0610ec4a82b48182cf2c6061),
2026-07-29. The fork is current with `nodejs/node` `v24.x` as of 2026-08-03 —
that release commit is the branch tip upstream, so there is nothing newer to
merge.

**What is built.** One executable per platform, attached to the GitHub Release
for that version — individual per-arch assets, no archive, so a consumer
downloads only the file it needs.

| Asset | Platform | How it is built |
| --- | --- | --- |
| `node-i386` | 32-bit x86 Linux | native, in an `i386/debian:bookworm` container |
| `node-armhf` | 32-bit ARM Linux, hard-float, VFPv3-D16 | cross, from a 32-bit x86 host container |
| `node-armv7` | 32-bit ARM Linux, hard-float, NEON | cross, from a 32-bit x86 host container |
| `node-loong64` | LoongArch64 Linux | cross, in a `debian:trixie` container |
| `node-win32.exe` | 32-bit Windows | native on a Windows runner, ClangCL |

`armhf` and `armv7` are both 32-bit hard-float ARM; the difference is the FPU
baseline, which is the difference that matters on the boards this exists for.

**Deliberately not built**, so the gaps are decisions rather than oversights:
**armel** (ARMv5), because V8 dropped it years ago and a target that can only
fail is worse than an honest gap; and **macOS** and **FreeBSD**, which need
their own hosts rather than a cross compiler on Linux.

**Almost all of this is build configuration.** Two commits touch shipped source
— the zlib SIMD flags and the V8 template disambiguator below — and both are
fixes for 32-bit targets that upstream no longer builds and therefore no longer
compiles.

## Changes

Newest first.

<details>
<summary><a href="https://github.com/wekan/node/commit/61e9c936c5556a50ec10a3ac570a5e2862f7b728">Fix what the fifth workflow run found: the host linker, a V8 template, and ICU's name for ia32</a>. Thanks to xet7.</summary>

Five builds, five failures, three causes.

**armhf, armv7 and loong64** all died linking `node_js2c`, a *host* tool, with
the *cross* compiler — the host's own word-size flag (`-m32`, and `-m64` in the
loong64 container) handed to a compiler that has never heard of it. The cause
is a fallback rather than a default: gyp's make generator resolves each host
tool as `GetEnvironFallback(("<TOOL>_host", "<TOOL>"), default)`, so an unset
`LINK_host` does not fall back to `$(CXX.host)` — it falls back to `LINK`,
which these jobs set to the cross linker. Setting `CC_host`/`CXX_host` had
therefore fixed the host *compiles* and left the host *link* on the cross
toolchain. `LINK_host` and `AR_host` are named explicitly now.

**i386** stopped in V8, on two lines of `int64-lowering-reducer.h` that write
`__ Tuple<Word32, Word32>(...)`. `__` expands to `Asm().` and the reducer is a
template, so C++ wants `__ template Tuple<...>`. The idiom is used 113 times
elsewhere in that directory; these two were missed, and nothing noticed because
the header is only compiled for 32-bit targets.

**win32** failed `MSB8066` on the ICU data object, from `genccode` answering
`CPU architecture "ia32" is unknown.` — `-c` takes ICU's name for the CPU, and
node's and ICU's agree on x64, arm64 and arm only by coincidence. `ia32` maps
to `x86` now, which ICU maps to `IMAGE_FILE_MACHINE_I386`.

</details>

<details>
<summary><a href="https://github.com/wekan/node/commit/5a96e87974dd512015be45337e3a3eb786a2e426">Fix what the second workflow run found: zlib SIMD flags, an ARMv8 dependency, loong64 openssl, and the ARM builds off QEMU</a>. Thanks to xet7.</summary>

Five builds, five failures, none the same one twice.

**i386 and armv7** died in `deps/zlib`: the targets define
`DEFLATE_SLIDE_HASH_SSE2`/`_NEON` and compile the intrinsics, but never pass
the compiler the flag that *enables* those instructions. On x86_64 and arm64
that goes unnoticed because SSE2 and NEON are part of those architectures; on
32-bit they are extensions. The same NEON condition also pulled in
`zlib_arm_crc32`, whose `-march=armv8-a+aes` a 32-bit gcc rejects outright —
only arm64 implies ARMv8, so only arm64 takes that dependency now.

**loong64** died in OpenSSL: `deps/openssl` ships exactly one LoongArch config
and it is `no-asm`, so an asm build fell through to `linux-x86_64/asm` and
handed the LoongArch compiler x86_64 flags.

**win32** asked msbuild for `Release|x64` against a solution that has
`Release|Win32` — gyp names the ia32 configuration `Win32`, and the line saying
so was removed with 32-bit Windows upstream and missed by the restore. The job
installs NASM now too, without which configure quietly falls back to the slow C
implementation of every cipher.

**armhf** was killed by the timeout with five hours and fifty minutes of work
in it, still compiling V8 under QEMU — which was never going to finish, six
hours being GitHub's ceiling. Both ARM builds cross compile from a 32-bit x86
host container now, which an x86_64 runner runs natively. V8 defines
`USE_SIMULATOR` itself for a non-ARM host, so `mksnapshot` simulates the ARM it
generates for while every compile runs at the runner's own speed.

</details>

<details>
<summary><a href="https://github.com/wekan/node/commit/22f6367caad2c67576878943d830eda1e5ff8bc4">A release carries exactly one run's binaries</a>. Thanks to xet7.</summary>

After uploading, the publish step deletes every `node-*` asset this run did not
produce, so a release cannot mix binaries from different runs — which would
mean comparing against a binary whose provenance nobody knows.

It sweeps *after* the upload, never before, so a failed upload leaves the
release with what it had. Only `node-*` is swept; anything else was attached on
purpose. The comparison is by asset name, so `node-win32.exe` keeps its
extension.

</details>

<details>
<summary><a href="https://github.com/wekan/node/commit/f5b65320a75329e3a17e17f4bd2464a865b09eb7">A re-run replaces the binaries it rebuilds, and says what it kept</a>. Thanks to xet7.</summary>

Replacing was already the behaviour; the notes were the missing half. They were
written from `dist/` alone, so a re-run fixing two platforms out of five would
name only those two, and somebody downloading one of the other three would find
a binary the notes never mention. The platform list is the union now — what
this run built, plus what the release already carries — and the log prints the
two sets separately, because "this one is older than the notes" is worth
knowing.

</details>

<details>
<summary><a href="https://github.com/wekan/node/commit/d099de4c2823b06a7cdd6db3f2a72b319f7171c3">Fix what the first workflow run found: i386 openssl assembly, and the ARM host V8 insists on</a>. Thanks to xet7.</summary>

**i386** got through configure and V8 and then hit OpenSSL, where every line of
the pregenerated x86 assembly was rejected by GNU as: the file opens in NASM
syntax, because it was generated for another assembler. Nobody had compiled it
since nodejs.org stopped building linux-x86, so it had rotted unnoticed.

**armhf and armv7** died building the host toolset on `v8config.h`: "Target
architecture arm is only supported on arm and ia32 host". V8 refuses an ARM32
target on an x86_64 host, which is why nodejs.org's own armv7 binaries are
built on ARM machines.

</details>

<details>
<summary><a href="https://github.com/wekan/node/commit/0b950b403345092a9fe9ad8b272351eb88be33a3">Put 32-bit Windows back, and build it</a>. Thanks to xet7.</summary>

Unlike the Linux platforms — where the support was always there and only the
build was missing — upstream removed 32-bit Windows itself, in
[7ad0cc3e571](https://github.com/nodejs/node/commit/7ad0cc3e571). That removal
was twelve files; four matter for producing a `node.exe` and are restored:
`configure.py` (the host-arch table knows what x86 means again, while the
default stays x64), `vcbuild.bat`, `node.gyp` (`/SAFESEH` off, x86-only) and
`toolchain.gypi` (`/arch:SSE2`, so doubles are 64-bit rather than x87's
80-bit-then-rounded).

The fifth needed re-deriving. The removal took out an ia32 case pointing at
`push_registers_masm.asm`, and that file no longer exists in this V8 — all that
is left for ia32 is `push_registers_asm.cc`, whose `_WIN32` branch is written
in Clang's inline-asm syntax. So the case points at the `.cc` and the build uses
ClangCL rather than `cl.exe`: not a preference, the only thing that compiles it.

The MSI/installer and test-status parts of that removal are *not* restored —
this workflow publishes a bare binary, and an installer nobody builds is a file
to keep in step for nothing.

</details>

<details>
<summary><a href="https://github.com/wekan/node/commit/86bd37b6a74c40df4816b07b152ba51ea5be30ed">The workflow is called Node.js, not the three platforms it started with</a>. Thanks to xet7.</summary>

It was named after i386, armhf and armv7, and it builds loong64 too now — and
will build whatever else nodejs.org and unofficial-builds leave out. A workflow
named after a list has to be renamed every time the list grows, and the list is
the matrix, which the Actions page shows anyway.

</details>

<details>
<summary><a href="https://github.com/wekan/node/commit/f1fdb0903c71541f7f248e7aefec6280989a206a">Build LoongArch64 too</a>. Thanks to xet7.</summary>

`wekan/FerretDB` builds for LoongArch64 and nobody publishes a Node.js for it,
so such a machine had a database and no runtime. Node supports the target —
`configure.py`'s `valid_arch` carries `loong64` and V8 has the full port in
`deps/v8/src/codegen/loong64` — so it only needed building, which is what this
fork is for. Cross compiled with the `loongarch64-linux-gnu` toolchain.

This is also where the deliberate gaps got written down: armel, and the
platforms that need their own hosts.

</details>

<details>
<summary><a href="https://github.com/wekan/node/commit/e248bdfa7615d7c2d8dc065cd0d5fd3c99ff5ebd">Add the build workflow: i386, armhf and armv7, attached to the release</a>. Thanks to xet7.</summary>

One job per platform, so they build in parallel with fail-fast off: a platform
that breaks is one red job with its own log, and the ones that worked are still
published.

i386 builds *inside* an i386 container rather than cross compiling — an x86_64
CPU runs i386 natively, so it is a native build at native speed and configure
sees a real i386 host. Through `docker run` and not the job-level `container:`
key, because the Actions runner injects its own x86_64 Node.js into a job
container and an i386 image has no 64-bit loader to start it, so checkout
itself would fail.

The release notes are built from what actually landed in `dist/`, with
upstream's own notes for that exact version lifted out of
`doc/changelogs/CHANGELOG_V<major>.md` by its anchor — so a platform that failed
is absent from the notes rather than promised and missing.

Nothing had to be restored for these three: `configure.py`'s `valid_arch` still
carries arm, ia32 and x86, `common.gypi` still branches on them, and V8's
32-bit ports are all present. What was missing was the build, not the code.

</details>

## Running it

From the Actions page, or:

```
gh workflow run node.yml -f version=v24.18.1
```
