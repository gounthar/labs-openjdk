# I just wanted one binary. How hard can it be?

*Part 1 of 3: Pulling the Thread — GraalVM Native Image on RISC-V 64*

---

The ask was simple. Or so it seemed.

Through the RISE program — a cross-industry initiative to grow the RISC-V software ecosystem — I've been contributing to riscv64 support across the Java toolchain. The goal isn't academic: RISC-V hardware is real, CI runners exist, and the gap between "runs under emulation" and "ships native binaries" is exactly the kind of gap the program exists to close.

JReleaser is a release automation tool written in Java. It handles changelogs, artifact publishing, cross-platform release coordination. It ships native binaries — fast-starting, no JVM required — using GraalVM Native Image. One binary per platform: linux-x86_64, linux-aarch64, macOS, Windows.

No linux-riscv64. That's the string.

---

## The pull request that didn't solve the problem

JReleaser had already done the right thing in code. Pull request [jreleaser/jreleaser#2062](https://github.com/jreleaser/jreleaser/pull/2062) added `linux-riscv_64` to the list of recognized platforms. Merged. Done.

Except: no riscv64 binary appeared in the next release.

This is a pattern I've seen repeatedly in the riscv64 ecosystem. Platform recognition gets added to the code — someone adds an `if riscv64` branch, or adds the string to a list, or adds the entry to a configuration table. Then the PR merges. Then nothing ships, because the release pipeline was never updated, or the release pipeline depends on something that doesn't exist for riscv64 yet, or that dependency depends on something else.

The recognition is real. The artifact isn't.

JReleaser's native binaries are produced by GraalVM Native Image. The build process is: take a Java application, point `native-image` at it, get a standalone executable. For that to work on linux-riscv64, you need GraalVM Native Image to run on riscv64.

So: does GraalVM support riscv64?

---

## Filed: oracle/graal#13351

The answer GraalVM gives is: not officially. There's no riscv64 distribution on the GraalVM releases page. I filed [oracle/graal#13351](https://github.com/oracle/graal/issues/13351) asking about it. The maintainer marked it Future.

That's not a no. "Future" means: the intent exists, but no one has prioritized it, and no one has done the work to complete it. It means: the thread is there, but no one has pulled it.

I pulled it.

---

## What's actually inside GraalVM

Reading the issue response and then reading the code are two different things. I cloned [graalvm/graal](https://github.com/graalvm/graal) and started looking.

What I found was not a missing feature. It was incomplete plumbing around a feature that someone had started years ago and never finished connecting.

GraalVM Native Image supports two code generation backends:

1. **A native backend** — AMD64 and AArch64 each have a custom code generator written in Java, around 10,000 lines per architecture. This is what runs when you do `native-image` on the platforms you actually ship to production.

2. **An LLVM backend** — instead of generating machine code directly, GraalVM emits LLVM IR and lets LLVM handle the final compilation. This is slower and produces larger binaries, but it means any architecture that LLVM supports can, in principle, be used.

For riscv64, the native backend doesn't exist. But here's what does exist, in `substratevm/src/.../riscv64/`:

- `SubstrateRISCV64Feature.java` — the entrypoint that routes to the LLVM backend
- `SubstrateRISCV64RegisterConfig.java` — register configuration for the riscv64 ABI
- `RISCV64CPUFeatureAccess.java` — CPU feature detection (RV64GC, RV64IMAFDCV)
- `RISCV64ArithmeticSnippets.java` — arithmetic lowerings
- `PosixRISCV64VaListSnippets.java` — varargs support
- HotSpot integration stubs
- Platform registration: `Platform$LINUX_RISCV64` already registered

Someone at Oracle started this. The skeleton is there. `SubstrateRISCV64Feature.java` even has the guard that makes the intent explicit:

```java
if (!SubstrateOptions.useLLVMBackend()) {
    throw GraalError.unimplemented(
        "The RISC-V native backend is currently unimplemented. Use the LLVM backend.");
}
```

This is a checkpoint, not a bug. LLVM has excellent riscv64 support — the SpacemiT K1 chip on the Banana Pi F3 I'm using as a CI runner is rv64gc, and LLVM has handled that ISA for years. The riscv64 LLVM JARs that GraalVM needs were uploaded to the build server in September 2022. They're still there. HTTP 200.

So the code paths exist. The compiler dependency exists. What's missing is the wire that connects them to an actual build.

---

## The dependency chain

Pull on "I want a JReleaser riscv64 binary" and here's what you find attached to the other end:

```
JReleaser linux-riscv64 native binary
  └─ native-image --llvm succeeds on riscv64
       └─ GraalVM substratevm builds on riscv64
            └─ JvmFuncsFallbacksBuildTask needs static .a libs
                 └─ labs-openjdk riscv64 artifact with static libs
                      └─ 404: artifact never built
```

The deepest knot is a single missing tarball. Not a missing compiler. Not unsupported hardware. Not a fundamental architectural gap.

A file that was never uploaded to a GitHub release.

The URL is even hardcoded in GraalVM's build system. When you run `mx fetch-jdk` on riscv64, it tries:

```
https://github.com/graalvm/labs-openjdk/releases/download/jvmci-25.1-b17/
labsjdk-ce-25.0.2+10-jvmci-25.1-b17-linux-riscv64.tar.gz
```

HTTP 404. That's it. That one URL, returning 404, is why GraalVM native-image is completely unbuildable on RISC-V 64.

---

## Why this matters beyond JReleaser

The pattern here is not unique to JReleaser or GraalVM.

Across the riscv64 ecosystem, this repeats: the code for a new platform gets written, sometimes years in advance. Register configs, ABI definitions, compiler hooks, platform strings. The code is correct. It's tested in CI on the platforms that have CI. But the release pipeline — the part that actually produces an artifact, uploads it, makes it downloadable — was never updated for riscv64.

The result: a platform that's "supported" in every meaningful code sense, but that no one can actually use, because there's nothing to download.

The fix, once you find it, is usually smaller than you expect. One URL. One CI job. One entry in a matrix. One missing package in a build script.

In this case: one tarball. `labsjdk-ce-25.0.2+10-jvmci-25.1-b17-linux-riscv64.tar.gz`. Build it. Upload it. The 404 becomes a 200. The chain unblocks.

---

## What labs-openjdk is, and why it matters

labs-openjdk is not a standard JDK distribution. It's OpenJDK with JVMCI patches — the Java-Level JVM Compiler Interface that GraalVM uses to hook into the JVM's compilation pipeline. It's also built with static libraries: `.a` files for every JDK native library, compiled with `-fPIC`.

Those static libraries are what GraalVM's build system (`mx`) needs to generate `JvmFuncsFallbacks.c` — a C file that stubs out JVM functions that Native Image can't call directly. The build task `JvmFuncsFallbacksBuildTask` scans the static `.a` files with `objdump`, finds unresolved `JVM_*` symbols, and generates stubs for them.

No static libs: build fails. Every riscv64 JDK distribution — Debian's OpenJDK, Adoptium's Temurin — ships a normal JDK. No static libraries. The only distribution that includes them is labs-openjdk, and labs-openjdk doesn't have a riscv64 build.

So: build labs-openjdk for riscv64. Get the static libs. Put them where mx expects them. The chain unblocks.

The next article covers what that actually looks like.

---

## The cliffhanger

The code for riscv64 Native Image support exists inside GraalVM. Months of groundwork, already done by someone at Oracle, sitting in the repository, waiting for the missing piece.

One missing artifact. One 404. One tarball that needs to be built and uploaded.

In the next article: forking graalvm/graal, hitting the first two build errors, and tracking the 404 to its source.

---

*Part of the "Pulling the Thread" series. [Part 2: GraalVM almost works on RISC-V. Almost.](article-2-draft.md)*

*Tags: riscv64, graalvm, jreleaser, rise*
