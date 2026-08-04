# WeKan fork of Node.js — fork changelog

This file lists what the [wekan/node](https://github.com/wekan/node) fork adds on
top of upstream Node.js. Upstream's own changelog is untouched and lives in
`CHANGELOG.md` and `doc/changelogs/`.

**Why the fork exists.** It began with a gap: some CPU/OS combinations have no
published Node.js at all — nodejs.org does not build them and
[unofficial-builds](https://unofficial-builds.nodejs.org/) does not either. A
machine with no runtime cannot run WeKan, however well WeKan is packaged, and
[wekan/FerretDB](https://github.com/wekan/FerretDB) already builds a database
for those same platforms, so this fork builds the runtime to go with it.

It has since become the place **WeKan takes its Node.js from for every
platform**, not only the ones with a gap, for two reasons. The fixes below are
build-configuration fixes that any platform benefits from, and a set of WeKan
bundles should not be half built on one Node.js and half on another depending on
which CPU it is for. And a backstop only works if it covers everything: when
nodejs.org or unofficial-builds has not published a version yet, this fork is
what fills the gap, and it can only fill a gap it builds.

**Base.** `v24.x`, on upstream
[v24.19.0 'Krypton' (LTS)](https://github.com/nodejs/node/commit/cdc1b38d40c),
2026-08-03, merged in as of 2026-08-03. Every change listed below was checked
after that merge: the set of files this fork touches is identical before and
after, and every line it adds is byte for byte the same.

**What is built.** One executable per platform, attached to the GitHub Release
for that version — individual per-arch assets, no archive, so a consumer
downloads only the file it needs.

| Asset | Platform | How it is built |
| --- | --- | --- |
| `node-x64` | 64-bit x86 Linux | native, on an x86_64 runner |
| `node-arm64` | 64-bit ARM Linux | native, on an ARM runner |
| `node-i386` | 32-bit x86 Linux | native, in an `i386/debian:bookworm` container |
| `node-armhf` | 32-bit ARM Linux, hard-float, VFPv3-D16 | cross, from a 32-bit x86 host container |
| `node-armv7` | 32-bit ARM Linux, hard-float, NEON | cross, from a 32-bit x86 host container |
| `node-ppc64le` | PowerPC 64 LE Linux | cross, in a `debian:trixie` container |
| `node-s390x` | IBM Z Linux | cross, in a `debian:trixie` container |
| `node-riscv64` | RISC-V 64 Linux | cross, in a `debian:trixie` container |
| `node-loong64` | LoongArch64 Linux | cross, in a `debian:trixie` container |
| `node-win64.exe` | 64-bit Windows | native on a Windows runner, ClangCL |
| `node-win32.exe` | 32-bit Windows | native on a Windows runner, ClangCL |
| `node-mac-x64` | macOS Intel | cross, on an arm64 macos-15 runner (Xcode 16) |
| `node-mac-arm64` | macOS Apple silicon | native, on a macos-15 runner (Xcode 16) |

`armhf` and `armv7` are both 32-bit hard-float ARM; the difference is the FPU
baseline, which is the difference that matters on the boards this exists for.

**Deliberately not built**, so the gaps are decisions rather than oversights:
**armel** (ARMv5), because V8 dropped it years ago and a target that can only
fail is worse than an honest gap; and **FreeBSD**, which needs a FreeBSD host
and has no runner - FreeBSD builds Node from ports, which is the answer there
anyway.

**Almost all of this is build configuration.** A handful of commits touch
shipped source — the zlib SIMD flags and macro, the V8 template disambiguator and
s390 simulator cast, and one `crypto_kmac.cc` aggregate-init form below — and
every one is a fix for a target (32-bit ARM, s390x, macOS) whose compiler upstream
no longer exercises, not a change to what Node.js does.

## Changes

Newest first.

<details>
<summary><a href="https://github.com/wekan/node/commit/900ec746ea87257159d9334f8b6d4a60b38d7b6d">The macOS builds move to Xcode 16 (macos-15), because Node 24's V8 needs C++20 aggregate init</a>. Thanks to xet7.</summary>

The mac-arm64 build failed compiling V8:
`deps/v8/src/wasm/value-type.h:701: error: no matching conversion for
functional-style cast from 'unsigned int' to 'TypeIndex'`. That is C++20
parenthesized aggregate initialization (P0960) - `TypeIndex(uint)` on an
aggregate - which macos-14's default Apple Clang (Xcode 15) does not support and
Xcode 16 does; it is the same feature that broke `crypto_kmac.cc`. Node 24
requires Xcode >= 16.1 on macOS (BUILDING.md).

`mac-arm64` now runs on **macos-15**, which ships Xcode 16 by default. `mac-x64`
cannot use a newer Intel runner - Xcode 16 needs macOS 14+, and every macos-14/15
GitHub runner is Apple silicon - so it **cross-compiles** x64 on an arm64 macos-15
runner (`--dest-cpu=x64 --cross-compiling`, universal SDK), guarded by the
existing `file` arch check. Node's own deployment target keeps both binaries
compatible with older macOS. The mac-x64 cross-compile still needs confirming on
CI.

</details>

<details>
<summary><a href="https://github.com/wekan/node/commit/11687784ddc3ce424bcbb81862f57262974decb4">Release all missing built nothing: it deadlocked on the concurrency group it shared with the workflow it calls</a>. Thanks to xet7.</summary>

`release-all-missing.yml` finds which per-arch binaries a release is short of and
builds only those, by calling `release-all.yml` in its `build` job with `uses:`.
It also declared **the same** concurrency group as `release-all.yml` —
`node-32bit-<version>` — deliberately, so a "fill the gaps" run and a full run
could not upload to one release at once. But a reusable workflow that requests a
concurrency group **already held by its caller** deadlocks: the caller holds the
group and waits for the callee, the callee waits for the group. GitHub detects it
and cancels the whole run —

    Canceling since a deadlock was detected for concurrency group
    'node-32bit-v24.19.0' between a top level workflow and 'build what is missing'

— after the `plan` job had listed what was missing and before the `build` job
could start. So the workflow only ever *listed* the gaps and never filled one.

The caller now uses a **distinct** group, `node-missing-<version>`. The mutual
exclusion that the shared group was for is preserved where it actually matters:
the build and the `gh release upload` happen inside `release-all.yml`, which
keeps its own `node-32bit-<version>` group, so this workflow's inner call and a
direct full run still serialize on it and cannot write to one release at the same
time. Two "fill the gaps" runs serialize on `node-missing-<version>`. With the
deadlock gone the `build` job runs and the three build fixes below actually
produce their binaries.

</details>

<details>
<summary><a href="https://github.com/wekan/node/commit/9d7fbf3249f016063a11e8498880f5819734eb38">Fix the v24.19.0 builds for macOS, s390x and armv7: a C++20 aggregate init, a const cast, and an ARM zlib macro</a>. Thanks to xet7.</summary>

The v24.19.0 "build what is missing" run left six targets unbuilt. Two of them —
**win64** and **win32** — in fact built and published; they were carried by the
same run and only looked missing in a later check. The other four were genuine
compile failures, three distinct ones:

**mac-arm64 and mac-x64** failed in `src/crypto/crypto_kmac.cc`, which
constructed `ncrypto::Buffer<const void>` with parentheses —
`Buffer<const void>(key_data, key_size)`. `Buffer` is an aggregate
(`{ T* data; size_t len; }`), so that is C++20 parenthesized aggregate
initialization (P0960): GCC and modern Clang accept it, which is why every Linux
build passed, but the macOS runners' Apple Clang does not, so both mac builds
failed with `no matching constructor for 'ncrypto::Buffer<const void>'`. Switched
to brace initialization — `Buffer<const void>{.data = …, .len = …}` — the
portable form `crypto_hash.cc` right beside it already uses.

**s390x** failed building the V8 s390 *simulator* (a host tool, and the only
place the templated `Instruction::SetInstructionBits<T>` member wrapper in
`deps/v8/src/codegen/s390/constants-s390.h` is instantiated). The wrapper handed
a `const uint8_t*` to the static overload that takes a non-const `uint8_t*`, so
it failed with `invalid conversion from const uint8_t* to uint8_t*`. The static
overload writes through the pointer; this `const` accessor patches instructions
in place the way V8 does elsewhere, so it strips the const rather than passing a
pointer the overload cannot bind.

**armv7** failed at link, not compile: `openssl-cli` ended with
`adler32.c: undefined reference to cpu_check_features`. 32-bit ARM+NEON enables
`ADLER32_SIMD_NEON`, so `adler32.c` calls `cpu_check_features()`, but
`cpu_features.c` only *defines* that function when an `ARMV8_OS_*` macro names
the platform — and on arm64 that macro arrives with the ARMv8-only
`zlib_arm_crc32` dependency, which a 32-bit target deliberately does not take
(its `-march=armv8-a+aes+crc` is AArch64-only). So `ARMV8_OS_LINUX` is now set on
the main `zlib` target for 32-bit ARM+NEON directly, independent of that
dependency. This sits next to the earlier zlib SIMD-flags fix, the same
"32-bit is not a subset of 64-bit here" shape.

</details>

<details>
<summary><a href="https://github.com/wekan/node/commit/8c4de123a1cabe66e08fa022251c07e6cbee31b7">A job-level `if:` cannot see `matrix`, and a workflow that tries does not load</a>. Thanks to xet7.</summary>

The `platforms` filter added with the workflow below was written as a job-level
condition:

    if: ${{ inputs.platforms == '' || contains(fromJSON(inputs.platforms), matrix.platform) }}

GitHub refuses to load a workflow that does that:

    Invalid workflow file
    (Line: 135, Col: 9): Unrecognized named-value: 'matrix'

`matrix` is available to a job's `runs-on`, `env`, `name`, `container`,
`services`, `continue-on-error`, `timeout-minutes`, `strategy` and `steps` — but
not to `jobs.<id>.if`, which is evaluated before the matrix is expanded. It
looks entirely reasonable, which is why it was written the same way in several
repositories at once.

It is worse than a job that does not run: a workflow that will not load takes
every workflow that CALLS it down with it, so `release-all-missing.yml` failed
at startup and built nothing.

The decision moves to the job's `env:`, which can see matrix, and every step
asks `if: ${{ env.BUILD_THIS == 'true' }}`. The seven steps that already had a
condition keep it, ANDed inside parentheses — `matrix.mode == 'windows'`,
`matrix.mode == 'cross-container'` and the rest are unchanged.

</details>

<details>
<summary><a href="https://github.com/wekan/node/commit/0b724bd7602a2d28526d12eff05645c4481db0e5">Only the platforms a release is missing can be built, and the full build is release-all.yml</a>. Thanks to xet7.</summary>

Thirteen platforms, and several of them take hours: armhf and armv7 are cross
builds that compile V8 twice and have been killed at the 360-minute mark,
loong64 takes about 150 minutes and i386 about 115. When one platform failed, or
was added, or a run was cancelled, the only way to get that one binary was to
run the whole thing again and rebuild all thirteen.

`release-all-missing.yml` works out which platforms the release is short of and
builds those. It carries no second copy of the build: `node.yml` is renamed
`release-all.yml` and gained a `workflow_call` trigger and a `platforms` filter,
so the missing-only workflow calls it with the subset it wants and the thirteen
platforms' compile flags stay in one place. That matters more here than
anywhere: a duplicated matrix would drift, and a binary added to a release
months later would then differ from the ones beside it.

A platform counts as present only when BOTH `node-<platform>[.exe]` and its
`.sha256sum` are on the release, so a binary whose checksum upload failed is
rebuilt rather than left half published. "Nothing is missing" is a notice and a
summary line, not a failure.

The selection was checked against three seeded releases: a complete one gives
`[]`; one missing armhf, with win32's checksum absent and no mac-x64 at all,
gives exactly `["armhf","win32","mac-x64"]`; and no release at all gives all
thirteen. The `.exe` naming of win64/win32 is part of that check.

The name change is so this repository calls its release workflows what the
other WeKan repositories call theirs.

</details>

<details>
<summary><a href="https://github.com/wekan/node/commit/e52958d30f884a1c8d7d1c15f267117228f64d27">A checksum file beside every binary</a>. Thanks to xet7.</summary>

nodejs.org publishes a `SHASUMS256.txt` for its releases and signs it; this fork
published nothing. So WeKan's build could verify a download from nodejs.org and
could not verify one from here, which meant the fork was the least trustworthy
of the three sources for a reason that was entirely fixable.

Every binary now gets a `node-<platform>.sha256sum` beside it on the release,
in the same `<sum>  <file>` format nodejs.org uses, so `sha256sum -c` works on
it directly and WeKan's build reads it the same way it reads the other two.

One file per binary rather than one file for the release, because that is the
shape the rest of these releases already have: a consumer downloads only the
platform it needs, and should not have to fetch a list covering twelve others to
check it.

This does not make the fork's binaries signed - that needs a key, which is a
decision and a secret rather than something a workflow can invent. It makes them
checkable, which is what catches the truncated download that is the realistic
failure.

</details>

<details>
<summary><a href="https://github.com/wekan/node/commit/0e319e128f440881d698f7cfbcc815f0c182fb48">Every platform is built here now, not only the ones nobody else builds</a>. Thanks to xet7.</summary>

The fork started as "the CPUs nobody publishes a Node.js for". That is no
longer what it is for: **WeKan takes its Node.js from here for every bundle**,
which changes the requirement twice over.

**The fixes.** What this fork carries that upstream does not is not 32-bit-only
— the ICU genccode architecture name, the x86 `/SAFESEH` opt-out, the zlib SSE2
and NEON flags, the V8 template disambiguator are build-configuration fixes. A
set of WeKan bundles should not be half built on one Node.js and half on another
depending on which CPU it is for.

**The backstop.** A fallback only works if it covers everything. nodejs.org and
unofficial-builds each publish on their own schedule, and when one is behind
this fork is what fills the gap — which it can only do for a platform it builds.
As this was written, unofficial-builds had only `x64-musl` for v24.19.0 while it
had riscv64 and loong64 for v24.18.1, leaving riscv64 two releases behind.

Eight platforms added, taking it from five to thirteen: **x64**, **arm64**,
**ppc64le**, **s390x**, **riscv64**, **win64**, **mac-x64** and **mac-arm64**.
x64, arm64 and both macOS builds run on a runner that IS the target CPU, so they
need no container, no toolchain and no cross flags. riscv64 gets
`--openssl-no-asm` for the same reason loong64 already did: `deps/openssl` has
no asm config for it.

`vcbuild.bat` is no longer called with a hardcoded `x86` — the Windows
architecture comes from the matrix, so win32 and win64 share one step.

</details>

<details>
<summary><a href="https://github.com/wekan/node/commit/62686ecf126f4bd66aa0acc8e4203b2771aad762">Merged upstream v24.19.0, with every fork change checked afterwards</a>. Thanks to xet7.</summary>

240 commits from upstream `v24.x`, up to the
[v24.19.0](https://github.com/nodejs/node/commit/cdc1b38d40c) release commit of
2026-08-03. It merged with no conflicts.

A clean merge is not by itself evidence that a fork survived it, so this was
checked rather than assumed, two ways. The set of files this fork changes
against its upstream base is **identical** before and after — the same eleven.
And the fork's diff against its base is byte for byte the same: 164 added and
removed lines before, 164 after, with every added line identical.

So all of it is still here: the node.yml workflow, the `/SAFESEH:NO` opt-out in
`common.gypi`, `configure.py`'s Windows `x86` → `ia32` host mapping, the V8
`__ template Tuple` disambiguator, the zlib NEON and SSE2 flags, ICU's
architecture name for genccode, and the `vcbuild.bat` Win32 configuration.

</details>

<details>
<summary><a href="https://github.com/wekan/node/commit/0767aa19aefb72c0fe24b1638a4ed341b37fa3d9">Each platform gets the build timeout its build actually needs</a>. Thanks to xet7.</summary>

`armhf` and `armv7` were both cancelled at exactly three hours, mid-compile,
with nothing wrong with them: `timeout-minutes` was a flat 180, set as a
*prediction* before any ARM build had ever run to completion.

The prediction missed that a **cross build compiles V8 twice** — once for the
host, to get `mksnapshot`, and once for the target — and that for ARM the host
half is itself 32-bit, because V8 refuses anything else: *"Target architecture
arm is only supported on arm and ia32 host"* (`deps/v8/include/v8config.h`). An
hour of armhf's three went on `obj.host/` before a single ARM object was built.

Measured from the sixth run: win32 22 min, i386 115 (native container — the
only Linux job that is *not* a cross build, so it has no host toolset to build),
loong64 148 (cross, but its host half is native amd64), armhf and armv7 killed
at 180 with ~85% and ~88% of their target compiles done.

The timeout comes from the matrix now, keeping the rule the flat number was
reaching for — roughly twice a good build, so a hang is stopped rather than left
to burn the runner's whole allowance. i386 240, loong64 300, armhf and armv7
360. ARM gets 360 rather than twice its build because 360 is the ceiling: a job
on a GitHub-hosted runner is killed at six hours whatever this file says.

</details>

<details>
<summary><a href="https://github.com/wekan/node/commit/9a057a4284edad38fbd442cd337283400690362a">Fix what the sixth workflow run found: x86 links opt out of /SAFESEH everywhere, not only in node.gyp</a>. Thanks to xet7.</summary>

The ICU data object the fifth run taught `genccode` to write correctly is now
written correctly — and the linker refuses it:

```
lld-link : error : /safeseh: ../obj/global_intermediate/icudt78l_dat.obj
is not compatible with SEH [tools\v8_gypfiles\gen-regexp-special-case.vcxproj]
```

`icudt78l_dat.obj` is generated **data**. It holds no code, so it carries no
safe-exception-handler table, and nothing can give it one. Opting out of
`/SAFESEH` is the only way to link it — which is why node.gyp already carried
`ImageHasSafeExceptionHandlers: 'false'`, restored with 32-bit Windows.

The opt-out was in the wrong place. A `target_defaults` in `node.gyp` reaches
only the targets **node.gyp defines**, and this object is linked by targets in
`tools/v8_gypfiles/v8.gyp` and `tools/icu/icu-generic.gyp`, which never saw it.
Nothing had to opt those in before, because gyp turns `/SAFESEH` on by itself
for every x86 link — `msvs_emulation.py` sets `safeseh_default = "true"` when
the arch is x86 — and upstream stopped building the one architecture where that
default means anything.

It sits in `common.gypi` now, scoped to `ia32`. `tools/gyp_node.py` passes that
file to gyp with `-I`, and gyp hands its includes down to every dependency
build file it loads, so one rule covers node, V8 and ICU alike.

</details>

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
