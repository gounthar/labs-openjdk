# Blog Series: Pulling the Thread — GraalVM Native Image on RISC-V 64

## Series concept

Not a planned port. A discovery. You find a loose string in the ecosystem,
you pull it, you find a knot. You untangle it, pull again, find another knot.
Three articles, three knots. One string.

The string: "I want JReleaser riscv64 binaries."

**Format:** Technical narrative, first person, discovery-driven
**Tone:** Name the mistakes. Don't editorialize on build times. Don't frame
it as "I figured it all out" — honesty about in-progress work is part of
the value.
**Length:** ~1500-2000 words each (~12-15 min total read)
**Audience:** Java/GraalVM developers, riscv64 ecosystem people, RISE community
**Publication:** bruno.verachten.fr
**Series tags:** `riscv64`, `graalvm`, `jreleaser`, `rise`

---

## Article 1 — The String

**Working title:** "I just wanted one binary. How hard can it be?"

**Opening beat:** RISE context — sponsoring riscv64 ecosystem support across
the Java toolchain. JReleaser is a Java-based release tool that ships native
binaries via GraalVM native-image. You get the platform recognition PR merged
(jreleaser/jreleaser#2062). The platform is recognized. But no riscv64 binary
appears in the release.

**Why It Matters:** The gap between "we support this platform" in code and
"you can actually run this" in production is often a single missing artifact
in a release pipeline. JReleaser uses GraalVM native-image to build binaries.
GraalVM has no riscv64 distribution. Filed oracle/graal#13351. Marked Future
by maintainer. Dead end — or so it seems.

**Dependency chain (text diagram):**

```
labs-openjdk riscv64 artifact  ← the 404
  └─ static .a libs bundled inside
       └─ GraalVM mx: JvmFuncsFallbacksBuildTask succeeds
            └─ native-image --llvm works on riscv64
                 └─ JReleaser ships linux-riscv64 native binaries
                      └─ RISE submission #30 reopens
```

**Who cares:** RISE project, riscv64 CI infrastructure, anyone wanting to run
JVM tooling natively on riscv64 hardware rather than under emulation.

**Cliffhanger:** The code for riscv64 support exists inside GraalVM. Someone
at Oracle started this years ago. Pull that thread.

---

## Article 2 — First Tangle: GraalVM

**Working title:** "GraalVM almost works on RISC-V. Almost."

**Intro hook:**

> One URL returning 404. That's all it took to make GraalVM native-image
> completely unbuildable on RISC-V 64. Not a missing compiler backend, not
> unsupported CPU instructions — a tarball that was never uploaded.

**Section: Architecture in Two Minutes**

Two backends: native (AMD64, AArch64 — full LIR codegen, ~10k lines per
arch) and LLVM (used for riscv64 and everything else without a native
backend). The LLVM path is a deliberate design choice, not a fallback.

`SubstrateRISCV64Feature.java` — what it does and the key guard:

```java
if (!SubstrateOptions.useLLVMBackend()) {
    throw GraalError.unimplemented(
        "The RISC-V native backend is currently unimplemented. Use the LLVM backend.");
}
```

This is not a bug — it's a checkpoint. LLVM has excellent riscv64 support.
The missing piece is build system wiring, not compiler work.

**Key insight:** Native backend missing = months of work. LLVM wiring missing
= weeks or less. Knowing which backend a platform uses tells you the scope.

**Section: The Archaeology**

SubstrateVM already has the riscv64 skeleton: register config
(`SubstrateRISCV64RegisterConfig`), CPU feature detection
(`RISCV64CPUFeatureAccess`), arithmetic snippets, varargs support, HotSpot
integration stubs. LLVM JARs for riscv64 were uploaded to the lafo server in
September 2022 (HTTP 200 — they're still there). The platform is registered.
The infrastructure predates this effort.

**Key insight:** Before assuming something doesn't exist in a large codebase,
do the archaeology. Three hours of grep and git log can save three months of
redundant work.

**Section: The Hardware**

Banana Pi F3 (SpacemiT K1, rv64gc, 16 GB RAM) — one of the RISE riscv64 CI
runners. Real hardware, not QEMU. Native riscv64 builds are slow by x86
standards. That's expected. Flag it early so the 2h26m figure in the next
article doesn't read as a problem.

**Key insight:** Having native riscv64 hardware in your CI lab changes what's
possible. This is an argument for the RISE runner program, not just context.

**Section: Fork, clone, mx build, first error**

Fork graalvm/graal. Clone graalvm/mx. Run `mx build` in substratevm. First
error:

```
AssertionError: translation to configure style arch is not supported yet for riscv64
  File "truffle/mx.truffle/mx_truffle.py", line 2334
```

The dict that maps arch names for autotools `--host` knows `amd64` and
`aarch64`. riscv64 is absent. One-line fix. Commit to gounthar/graal.
Build proceeds.

**Second error — the real knot:**

```
JvmFuncsFallbacksBuildTask svm-jvmfuncs-fallback-builder failed
Could not find any unresolved JVM_* symbols in static JDK libraries
```

SubstrateVM generates C stubs by scanning static `.a` libraries with
`objdump`. It needs `$JAVA_HOME/lib/static/linux-riscv64/glibc/lib*.a`.

| Distribution | riscv64? | Static libs? |
|---|---|---|
| Debian OpenJDK 25 | yes | no |
| Temurin 25 (Adoptium) | yes | no |
| GraalVM labs-openjdk jvmci-25.1-b17 | **no** | yes (amd64/aarch64 only) |

mx tries to fetch the artifact. The URL pattern exists in the code:

```
https://github.com/graalvm/labs-openjdk/releases/download/jvmci-25.1-b17/
labsjdk-ce-25.0.2+10-jvmci-25.1-b17-linux-riscv64.tar.gz
```

HTTP 404. The artifact has never been built.

**Cliffhanger:** Fix the 404. The whole chain unblocks. Pull that thread.

---

## Article 3 — Second Tangle: labs-openjdk

**Working title:** "The 404 that blocked an entire ecosystem (and how we fixed it)"

**Opening:** Fork graalvm/labs-openjdk. SSH into the Banana Pi F3. Run
configure. Hit the wall — three times.

**Section: What the Docs Got Wrong (three concrete errors)**

Every error named explicitly because someone else will hit them:

1. `--enable-static-build` does not exist as a configure flag. Static libs
   are a make target (`make static-libs-image`), not a configure option. The
   documentation was aspirational.

2. `--with-jvm-features=graal` is invalid. The available feature names are
   `cds compiler1 compiler2 ... jvmci ...`. The correct flag is `jvmci`.

3. Configure fails: *"Could not find Xrandr.h"*. `libxrandr-dev` is missing
   from the documented prerequisite list. Install it.

**The corrected configure invocation:**

```bash
bash configure \
    --with-jvm-features=jvmci \
    --with-jvm-variants=server \
    --disable-warnings-as-errors \
    --with-extra-cflags="-fcommon"

make images static-libs-image
```

**Build #1 result:** 2h26m wall time. 39 static `.a` files produced. But the
JDK version is `26-internal` — built from master. mx is hardcoded to
`jvmci-25.1-b17`, which is JDK 25.0.2. Start over.

**Key insight:** In dependency chains with pinned versions, building from the
wrong commit produces a correctly-built artifact that solves nothing. Always
confirm the version string before a 2.5-hour build.

**Section: The Version Problem**

Clone graalvm/labs-openjdk at tag `jvmci-25.1-b17`. Configure reports
`25.0.2-internal` — correct. Build #2: 2h26m, 39 static `.a` files.

**Section: Packaging**

The build puts static libs at `images/static-libs/lib/*.a` (flat). The
aarch64 reference tarball uses `lib/static/linux-riscv64/glibc/lib*.a`
inside the JDK tree. Peek at the reference structure:

```bash
tar -tz labsjdk-ce-25.0.2+10-jvmci-25.1-b17-linux-aarch64.tar.gz | grep static
# labsjdk-ce-25.0.2-jvmci-25.1-b17/lib/static/linux-aarch64/glibc/libattach.a
# ...
```

Key detail: top-level dir is `labsjdk-ce-25.0.2-jvmci-25.1-b17/` (no `+10`
in dirname). Filename has `+10`. Copy that structure exactly for riscv64.

**Section: Publish and verify**

442 MB tarball. Uploaded to gounthar/labs-openjdk releases under tag
`jvmci-25.1-b17` via `gh release create` directly from the F3 (gh CLI
authenticated as gounthar). URL that was returning 404 now returns 302.

**Section: The mx test**

Stage static libs at `$JAVA_HOME/lib/static/linux-riscv64/glibc/`. Run
`mx build` in gounthar/graal substratevm with that JAVA_HOME. Result:

```
JvmFuncsFallbacksBuildTask svm-jvmfuncs-fallback-builder...
[continues — no failure]
```

The build continues through Truffle, SVM compiler, NFI native, GraalVM
compiler tests. Full success. `JvmFuncsFallbacksBuildTask` is unblocked.

**Section: CI on RISE runners**

Workflow `.github/workflows/build-riscv64.yml` on `ubuntu-24.04-riscv`.
Self-bootstrapping: uses the published jvmci-25.1-b17 artifact as boot JDK.
PR #1 on gounthar/labs-openjdk. CI run: ~2.5h. Path to upstream PR on
graalvm/labs-openjdk.

**Closing argument:**

This pattern repeats across the riscv64 ecosystem: x86/AArch64 toolchains
exist, code paths are written, but release pipelines were never wired for
riscv64. The work is usually smaller than it looks.

If your tool has a release pipeline and riscv64 is a registered platform,
check whether the artifact actually exists. A 404 is easier to fix than a
missing backend. One tarball, significant downstream surface area.

---

## Supporting material (gather before writing)

- [x] mx error log — exact failure text (buildlog-20260417-124550.html):
      `JvmFuncsFallbacksBuildTask svm-jvmfuncs-fallback-builder... [config was changed]`
      `Could not find any unresolved JVM_* symbols in static JDK libraries`
      `JvmFuncsFallbacksBuildTask svm-jvmfuncs-fallback-builder: Failed due to error: 1`
- [x] `SubstrateRISCV64Feature.java` LLVM guard snippet (in GRAAL-RISCV64-PLAN.md)
- [x] Build log timestamps — Build #2: configure done 19:09, build done 22:05 → **2:56:02 wall time**
      (Note: earlier estimate of ~2h26m was wrong; correct is 2h56m)
- [x] `ls images/static-libs/lib/*.a | wc -l` output: **38** (not 39 — earlier count was wrong)
- [x] `java -version` output (built JDK):
      `openjdk version "25.0.2-internal" 2026-01-20`
      `OpenJDK Runtime Environment (build 25.0.2-internal-adhoc.poddingue.labs-openjdk-25)`
      `OpenJDK 64-Bit Server VM (build 25.0.2-internal-adhoc.poddingue.labs-openjdk-25, mixed mode, sharing)`
- [x] mx build log showing `JvmFuncsFallbacksBuildTask` passing — in /tmp/mx-build.log on F3:
      task appears then build continues (no failure line); full GraalVM build completed
- [x] Configure invocation (from configure.log):
      `--with-jvm-features=jvmci --with-jvm-variants=server --disable-warnings-as-errors --with-extra-cflags=-fcommon`
- [ ] CI run URL: https://github.com/gounthar/labs-openjdk/actions/runs/24586557048 (in progress — wait for green)
- [x] PR #1 URL: https://github.com/gounthar/labs-openjdk/pull/1

## Additional material from F3 build logs

Build log progression (mx attempts before success):
- 12:44 — compiled mxtool.compilerserver (setup)
- 12:45 — **FAIL**: "Could not find any unresolved JVM_* symbols in static JDK libraries"
- 15:46 — dry-run only
- 15:51 — WARNING: "No static JDK libraries found for riscv64 - generating empty JvmFuncsFallbacks.c" then SVM_STATIC_LIBRARIES_SUPPORT failed (Debian JDK has no lib*.a glob to expand)
- 15:52 — FAIL: "Please use a JDK that contains static JDK libraries."
- 15:56 — WARNING: "No static JDK libraries found for riscv64 - skipping JvmFuncsFallbacks" (partial workaround)
- 16:00 — same skip workaround
- 23:03 — **SUCCESS** with labs-openjdk static libs staged at JAVA_HOME

## Cross-links between articles

- Art 1 → Art 2: "The code exists. Let's build it."
- Art 2 → Art 3: "One missing tarball. Let's build that."
- Art 3 → Art 1: Close the loop — JReleaser riscv64 binaries now unblocked
- Reference throughout: oracle/graal#13351, RISE #30, jreleaser#2062
