# GraalVM almost works on RISC-V. Almost.

*Part 2 of 3: Pulling the Thread — GraalVM Native Image on RISC-V 64*

---

One URL returning 404. That's all it took to make GraalVM native-image completely unbuildable on RISC-V 64. Not a missing compiler backend, not unsupported CPU instructions — a tarball that was never uploaded.

But I'm getting ahead of myself. First: the archaeology.

---

## Two backends, one platform gap

GraalVM Native Image supports two ways to generate machine code:

**The native backend.** AMD64 and AArch64 each have a purpose-built code generator written in Java — around 10,000 lines per architecture, translating GraalVM's intermediate representation directly to machine instructions. This is the fast path.

**The LLVM backend.** Instead of generating machine code directly, GraalVM emits LLVM IR and hands off to LLVM for the final compilation step. Slower, larger binaries, but portable: any architecture with LLVM support can, in principle, use this path.

For riscv64, there's no native backend. That's months of work and, frankly, not needed to get useful things shipped. What matters is the LLVM backend — and LLVM has had excellent riscv64 support for years. The intent is already in the code. `SubstrateRISCV64Feature.java` makes it explicit:

```java
if (!SubstrateOptions.useLLVMBackend()) {
    throw GraalError.unimplemented(
        "The RISC-V native backend is currently unimplemented. Use the LLVM backend.");
}
```

This is a checkpoint, not a failure. It means: you're on the right path, use LLVM. The question is whether the LLVM path actually connects to anything.

---

## What's already there

Before writing a single line of code, I spent a few hours reading through `graalvm/graal`. The most important lesson from working on platform ports: always do the archaeology first. Three hours of `grep` and `git log` can save three months of redundant work.

What I found in `substratevm/src/.../riscv64/`:

- `SubstrateRISCV64Feature.java` — the entrypoint that routes to the LLVM backend
- `SubstrateRISCV64RegisterConfig.java` — full register configuration for the riscv64 ABI
- `RISCV64CPUFeatureAccess.java` — CPU feature detection (RV64GC, RV64IMAFDCV variants including RVV)
- `RISCV64ArithmeticSnippets.java` — arithmetic lowerings for native-image compilation
- `PosixRISCV64VaListSnippets.java` — varargs handling per the riscv64 ABI
- `SubstrateRISCV64RegisterConfig.java` — HotSpot integration stubs
- Platform registration: `Platform$LINUX_RISCV64` already in the registry

And on the dependency server (`lafo.ssw.uni-linz.ac.at`), the LLVM JARs that GraalVM needs for riscv64:

```
https://lafo.ssw.uni-linz.ac.at/pub/graal-external-deps/native-image/
llvm-shadowed-13.0.1-1.5.7-linux-riscv64.jar     → HTTP 200
javacpp-shadowed-1.5.7-linux-riscv64.jar          → HTTP 200
```

Uploaded September 2022. Still there. Both already registered in `substratevm/mx.substratevm/suite.py`.

Someone started this work years ago. The skeleton is present. The LLVM dependencies exist. What's missing is the wiring that connects them to a buildable product.

---

## The hardware

The Banana Pi F3 is a single-board computer with a SpacemiT K1 chip — rv64gc, 16 GB RAM, 113 GB disk. It's one of the CI runners in the RISE riscv64 lab. Running Debian 13 (Trixie).

This matters for what comes later: builds happen natively on riscv64 hardware, not under QEMU or cross-compilation. Native riscv64 builds are slow by x86 standards — that's expected, and I'll flag it whenever a number comes up. Knowing it's native hardware is context for those numbers, not an apology for them.

The RISE runner program exists specifically because having real hardware changes what's feasible. You can do a lot with emulation; you can do everything with hardware.

---

## Fork, clone, first error

Fork `graalvm/graal`. Clone `graalvm/mx`. Set up the environment:

```bash
sudo apt-get install -y openjdk-25-jdk-headless
export JAVA_HOME=/usr/lib/jvm/java-25-openjdk-riscv64
git clone --depth=1 https://github.com/graalvm/mx.git ~/mx
export PATH=$HOME/mx:$PATH
git clone --depth=1 https://github.com/gounthar/graal.git ~/graal
cd ~/graal/substratevm
mx build
```

First error, immediately:

```
AssertionError: translation to configure style arch is not supported yet for riscv64
  File "truffle/mx.truffle/mx_truffle.py", line 2334
```

libffi is built as part of the Truffle native function interface, compiled via autotools. The `--host` argument is constructed from a dictionary:

```python
configure_arch = {"amd64": "x86_64", "aarch64": "aarch64"}.get(toolchain.spec.target.arch)
```

`riscv64` is not in that dict. One-line fix:

```python
configure_arch = {"amd64": "x86_64", "aarch64": "aarch64", "riscv64": "riscv64"}.get(toolchain.spec.target.arch)
```

Committed to `gounthar/graal`. Build proceeds. This is the kind of error that makes you optimistic — it's so small it means everything else was already handled.

---

## The real knot

Second error:

```
JvmFuncsFallbacksBuildTask svm-jvmfuncs-fallback-builder... [config was changed]
Could not find any unresolved JVM_* symbols in static JDK libraries
JvmFuncsFallbacksBuildTask svm-jvmfuncs-fallback-builder: Failed due to error: 1
```

This one took longer to understand.

SubstrateVM needs to generate a C file called `JvmFuncsFallbacks.c`. The purpose: Native Image can't call JVM internal functions directly, so it generates stub implementations. To know which stubs to generate, it scans static JDK `.a` libraries with `objdump --wide --syms`, looking for unresolved `JVM_*` symbols.

The task looks for static libs at `$JAVA_HOME/lib/static/linux-riscv64/glibc/lib*.a`. No standard riscv64 JDK ships these. I checked:

| Distribution | riscv64? | Static libs? |
|---|---|---|
| Debian OpenJDK 25 | yes | no |
| Temurin 25 (Adoptium) | yes | no |
| GraalVM labs-openjdk jvmci-25.1-b17 | **no** | yes (amd64/aarch64 only) |

I tried various workarounds. The build logs on the F3 tell the story across an afternoon of attempts:

- **12:45** — hard failure: "Could not find any unresolved JVM_* symbols"
- **15:51** — mx fell back to generating an empty `JvmFuncsFallbacks.c`, then choked trying to glob `lib*.a` from Debian's JDK (the glob expands to nothing)
- **15:52** — clearer error: "Please use a JDK that contains static JDK libraries."
- **15:56 / 16:00** — mx skipped JvmFuncsFallbacks entirely with a warning; build got further but was producing an incomplete native-image

Skipping isn't a real fix. The stubs exist for a reason. The right answer is to have the static libs.

---

## The 404

There's a distribution that ships static JDK libraries: `graalvm/labs-openjdk`. This is OpenJDK with JVMCI patches, built specifically for GraalVM's use. Every supported platform has a release artifact with the static libraries bundled inside.

GraalVM's build tool `mx` already knows the riscv64 URL. It's hardcoded in `mx`:

```
https://github.com/graalvm/labs-openjdk/releases/download/jvmci-25.1-b17/
labsjdk-ce-25.0.2+10-jvmci-25.1-b17-linux-riscv64.tar.gz
```

The LLVM JARs that mx downloads during the build? They're at `lafo` and they're present — HTTP 200. But this URL:

```
HTTP 404
```

The artifact has never been built. The URL pattern exists in mx. The platform recognition exists in labs-openjdk's CI configuration. The release tag `jvmci-25.1-b17` exists, with builds for linux-amd64, linux-aarch64, darwin-amd64, darwin-aarch64, windows-amd64.

No linux-riscv64.

This is the knot. Everything else — the riscv64 LLVM JARs, the SubstrateVM skeleton, the register configs, the CPU feature detection — all of that is in place. It's all waiting on one missing release artifact.

---

## What the fix actually involves

To fix the 404: build labs-openjdk for riscv64, package it in the expected format, publish it to the expected URL. The build is standard OpenJDK + JVMCI patches — OpenJDK 25 has full riscv64 support upstream, and JVMCI patches are maintained in `graalvm/labs-openjdk`. The hard part isn't the code; it's the build time on riscv64 hardware.

The next article covers the build: three configure flags that the documentation gets wrong, two builds (the first one from the wrong commit), and what it looks like when the chain finally connects.

---

*Part of the "Pulling the Thread" series.*
*← [Part 1: I just wanted one binary.](article-1-draft.md)*
*→ [Part 3: The 404 that blocked an entire ecosystem (and how we fixed it).](article-3-draft.md)*

*Tags: riscv64, graalvm, native-image, rise, substratevm*
