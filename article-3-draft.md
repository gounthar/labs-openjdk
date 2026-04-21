# The 404 That Blocked an Entire Ecosystem (and How We Fixed It)

*Article 3 of 3 — Pulling the Thread: GraalVM Native Image on RISC-V 64*

---

At the end of the last article, the problem was clear: GraalVM's build system
tries to fetch a specific tarball before it can build native-image for
riscv64. The URL is hardcoded. The file doesn't exist. The build stops.

One 404. That's it.

The fix is to build the tarball. That sounds simple. It is not.

---

## Fork, Clone, Configure

The artifact mx wants is a `labsjdk-ce-*-linux-riscv64.tar.gz` built from
[graalvm/labs-openjdk](https://github.com/graalvm/labs-openjdk) — OpenJDK
with JVMCI patches applied, plus static `.a` libraries that SubstrateVM
needs to generate its JVM function stubs.

Fork the repo. SSH into the Banana Pi F3. Clone. Run configure.

The CLAUDE.md I'd written for myself, summarizing what I thought I knew about
the build, had three mistakes in it. I'll name them explicitly because someone
else will hit the same wall.

**Mistake 1: `--enable-static-build` is not a configure flag.**

The documentation suggests it is. It isn't — I'd written it down somewhere
and trusted it. Running configure with that flag produces a warning and
silently drops it. Static libraries are not controlled by configure at all.
They're a make target: `make static-libs-image`. The configure flag I'd
imagined does not exist.

**Mistake 2: `--with-jvm-features=graal` is invalid.**

The correct flag for JVMCI support is `--with-jvm-features=jvmci`. The string
`graal` is not in the list of valid feature names. Configure rejects it. I'd
copied it from somewhere, didn't verify it, and it cost me a configure cycle.

**Mistake 3: `libxrandr-dev` is missing from the prerequisite list.**

Configure fails with "Could not find Xrandr.h" if it's not installed. The
published prerequisite list in the repository doesn't include it. Add it:

```bash
sudo apt-get install -y libxrandr-dev
```

The corrected configure invocation, confirmed from the actual `configure.log`:

```bash
bash configure \
    --with-jvm-features=jvmci \
    --with-jvm-variants=server \
    --disable-warnings-as-errors \
    --with-extra-cflags="-fcommon"

make images static-libs-image
```

That's it. No exotic flags. The static libs are a make target, not a
configure option.

---

## Build #1: Correctly Built, Wrong Version

The build runs. On the Banana Pi F3 — SpacemiT K1 processor, rv64gc,
16 GB RAM — this takes a while. 2h56m wall time for the first complete build.

38 static `.a` files produced under `build/linux-riscv64-server-release/images/static-libs/lib/`.

Then I checked the version:

```
openjdk version "26-internal" ...
```

Wrong. mx is hardcoded to `jvmci-25.1-b17`, which corresponds to JDK 25.0.2.
I'd built from master. The build was technically successful. The artifact
was useless.

This is the kind of mistake that costs an afternoon on slow hardware. The fix
was obvious in retrospect: check the version string before starting a three-hour
build. The build output is correct. The input was wrong.

---

## Build #2: The Right Tag

Delete the build directory. Clone `graalvm/labs-openjdk` at tag
`jvmci-25.1-b17`. Configure reports `25.0.2-internal`. Run the build.

Wait another 2h56m.

```
openjdk version "25.0.2-internal" 2026-01-20
OpenJDK Runtime Environment (build 25.0.2-internal-adhoc.poddingue.labs-openjdk-25)
OpenJDK 64-Bit Server VM (build 25.0.2-internal-adhoc.poddingue.labs-openjdk-25, mixed mode, sharing)
```

That's the right version. 38 static `.a` files. The build succeeded.

---

## Packaging: Match the Reference Structure Exactly

The build drops static libs flat into `images/static-libs/lib/*.a`. The mx
fetch expects a different layout — one that matches the existing aarch64
artifact in the upstream release.

Check the reference:

```bash
tar -tz labsjdk-ce-25.0.2+10-jvmci-25.1-b17-linux-aarch64.tar.gz | grep static | head -3
# labsjdk-ce-25.0.2-jvmci-25.1-b17/lib/static/linux-aarch64/glibc/libattach.a
# labsjdk-ce-25.0.2-jvmci-25.1-b17/lib/static/linux-aarch64/glibc/libdt_socket.a
# labsjdk-ce-25.0.2-jvmci-25.1-b17/lib/static/linux-aarch64/glibc/libj2pcsc.a
```

Two things to notice: the directory structure inside the tarball is
`lib/static/linux-{arch}/glibc/`, and the top-level directory name is
`labsjdk-ce-25.0.2-jvmci-25.1-b17/` — no `+10`. The `+10` appears in the
filename but not the directory. Copy that structure exactly for riscv64.

The tarball came out at 442 MB. Uploading from the F3 over a home connection
took 4m19s.

---

## Publish

The release goes to `gounthar/labs-openjdk` under tag `jvmci-25.1-b17` via
`gh release create` directly from the F3:

```bash
gh release create jvmci-25.1-b17 \
    --repo gounthar/labs-openjdk \
    --title "jvmci-25.1-b17" \
    labsjdk-ce-25.0.2+10-jvmci-25.1-b17-linux-riscv64.tar.gz
```

The URL that was returning 404:

```
https://github.com/graalvm/labs-openjdk/releases/download/jvmci-25.1-b17/
labsjdk-ce-25.0.2+10-jvmci-25.1-b17-linux-riscv64.tar.gz
```

Now returns 302 from `gounthar/labs-openjdk`. mx can be pointed at the
gounthar URL. The file exists.

---

## Does mx Actually Accept It?

Having the file is not enough. The structure has to be right, the static libs
have to contain the `JVM_*` symbols SubstrateVM is scanning for, and the
version string has to match what mx expects.

Stage the static libs at the path JvmFuncsFallbacksBuildTask will scan:

```bash
mkdir -p $JAVA_HOME/lib/static/linux-riscv64/glibc/
cp images/static-libs/lib/*.a $JAVA_HOME/lib/static/linux-riscv64/glibc/
```

Run `mx build` in gounthar/graal substratevm with that `JAVA_HOME`.

The build log from the first attempt without static libs, 12:45 on
2026-04-17:

```
JvmFuncsFallbacksBuildTask svm-jvmfuncs-fallback-builder: Failed due to error: 1
Could not find any unresolved JVM_* symbols in static JDK libraries
```

After staging the labs-openjdk static libs, at 23:03:

```
JvmFuncsFallbacksBuildTask svm-jvmfuncs-fallback-builder...
[build continues]
```

No failure line. The build proceeds through Truffle, SubstrateVM compiler,
NFI native, GraalVM compiler. Full success. `JvmFuncsFallbacksBuildTask`
is unblocked.

It took seven distinct mx build attempts across the day to get there — the
log timestamps tell the story of the wrong JDKs tried, the workarounds
attempted, the detours through Debian's OpenJDK (no static libs), through
Temurin (no static libs either), before the labs-openjdk artifact was in place.

The static libs are what the build task is looking for. That's all it needed.

---

## CI on RISE Runners

Doing this once manually is necessary. Doing it reproducibly, on real riscv64
hardware, on every tag — that's a CI workflow.

The workflow (`.github/workflows/build-riscv64.yml`) runs on the
`ubuntu-24.04-riscv` label — the RISE riscv64 GitHub Actions runners. It
triggers on `jvmci-*` tag pushes, reads version strings dynamically from
`make/conf/version-numbers.conf`, finds a suitable boot JDK via the Adoptium
API (falling back to a previously published artifact from the same repo), and
publishes the tarball to the release automatically. Not QEMU. Not
cross-compilation. Native riscv64 hardware.

The initial CI run — before the workflow was made fully dynamic — completed in
**3h38m44s** on the RISE runner. Green. PRs open against both `master` and
`jdk25` in the fork ([#2](https://github.com/gounthar/labs-openjdk/pull/2),
[#3](https://github.com/gounthar/labs-openjdk/pull/3)).

The wall time is what it is. A SpacemiT K1 is not a server-class machine.
It is, however, real hardware producing a genuine riscv64 artifact. For a
platform that until recently had no CI footprint at all, a passing 3.5-hour
build is not a problem to fix — it's infrastructure that didn't exist before.

---

## Where This Leaves Things

The dependency chain from Article 1:

```
labs-openjdk riscv64 artifact  ← was 404, now exists
  └─ static .a libs bundled inside  ← 38 libs, verified
       └─ GraalVM mx: JvmFuncsFallbacksBuildTask  ← passes
            └─ native-image --llvm works on riscv64  ← next step
                 └─ JReleaser ships linux-riscv64 native binaries
                      └─ RISE submission #30 reopens
```

The first three links are confirmed. The fourth — a full `native-image --llvm`
run producing a working HelloWorld binary — is the next thing to attempt. The
LLVM JARs for riscv64 have been on the lafo server since September 2022. The
SubstrateVM skeleton for riscv64 predates this effort. The wiring is close.

This pattern is consistent across the riscv64 ecosystem. The code paths are
written. The release pipelines were never wired for riscv64. The work is
usually smaller than it looks.

If your tool has a release pipeline and riscv64 is a registered platform,
check whether the artifact actually exists. Not in the code — in the release.
A 404 in a release is not a missing feature. It's a missing build job. Those
are fixable.

One tarball. That's how much of the GraalVM native-image stack for riscv64
was blocked.

---

*The gounthar/labs-openjdk release artifact:*
*`labsjdk-ce-25.0.2+10-jvmci-25.1-b17-linux-riscv64.tar.gz`*

*CI workflow: `.github/workflows/build-riscv64.yml` on ubuntu-24.04-riscv (RISE)*

*Upstream: issue filed at [graalvm/labs-openjdk#33](https://github.com/graalvm/labs-openjdk/issues/33) — PR after response*

*Next: `native-image --llvm HelloWorld` on riscv64*

