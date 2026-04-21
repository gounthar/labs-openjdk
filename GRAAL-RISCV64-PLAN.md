# GraalVM Native Image on RISC-V 64: Investigation Log

Running log of the attempt to build and run GraalVM native-image on riscv64.
Intended to inform a blog post and provide a clear path for upstream maintainers.

Hardware: Banana Pi F3 (SpacemiT K1, rv64gc, 16 GB RAM) - one of the RISE riscv64 CI runners.

---

## Background

JReleaser merged [jreleaser/jreleaser#2062](https://github.com/jreleaser/jreleaser/pull/2062) which adds
`linux-riscv_64` platform recognition, but JReleaser's own release pipeline does not produce riscv64
native binaries. The reason: native binaries are built with GraalVM Native Image, and GraalVM has no
riscv64 distribution.

Filed [oracle/graal#13351](https://github.com/oracle/graal/issues/13351) requesting riscv64 native-image
support. This document tracks the hands-on investigation.

Forks:
- https://github.com/gounthar/graal
- https://github.com/gounthar/labs-openjdk (planned)

---

## Architecture: how GraalVM native-image works on a new platform

Before diving into errors, the key insight from reading the code:

**GraalVM supports two backends for native-image:**

1. **Native backend** (AMD64, AArch64): custom LIR code generator in Java, ~10,000 lines per arch
2. **LLVM backend**: GraalVM emits LLVM IR, LLVM does the machine code generation

For riscv64, the code explicitly routes to the LLVM backend:

```java
// substratevm/src/.../riscv64/SubstrateRISCV64Feature.java
if (!SubstrateOptions.useLLVMBackend()) {
    throw GraalError.unimplemented(
        "The RISC-V native backend is currently unimplemented. Use the LLVM backend.");
}
```

This is good news: LLVM has excellent riscv64 support. The work is build system wiring,
not compiler implementation.

---

## Pre-existing riscv64 work (already in oracle/graal)

Someone at Oracle started this work. The following already exists:

**SubstrateVM skeleton (`com.oracle.svm.core.graal.riscv64`)**
- `SubstrateRISCV64Feature.java` - routes to LLVM backend
- `SubstrateRISCV64RegisterConfig.java` - register configuration
- `RISCV64ReservedRegisters.java` - reserved register definitions

**SubstrateVM core (`com.oracle.svm.core`, riscv64 package)**
- `RISCV64CPUFeatureAccess.java` - CPU feature detection
- `RISCV64FrameAccess.java` - stack frame layout
- `RISCV64LibCHelper.java` / `RISCV64LibCHelperDirectives.java` - libc integration

**SubstrateVM snippets**
- `RISCV64ArithmeticSnippets.java` - arithmetic lowerings
- `RISCV64NonSnippetLowerings.java` - non-snippet lowerings
- `RISCV64SnippetsFeature.java` - snippet registration
- `PosixRISCV64VaListSnippets.java` - varargs support

**Compiler backend**
- `RISCV64LoweringProviderMixin.java`
- `RISCV64NodeMatchRules.java`
- HotSpot integration stubs (4 files)

**Platform registration**
- `Platform$LINUX_RISCV64` already registered
- `CPUTypeRISCV64.java` with RV64GC, RV64IMAFDCV variants

**LLVM JARs on lafo server** (uploaded Sep 2022 - someone already built these)
```
https://lafo.ssw.uni-linz.ac.at/pub/graal-external-deps/native-image/llvm-shadowed-13.0.1-1.5.7-linux-riscv64.jar  → HTTP 200
https://lafo.ssw.uni-linz.ac.at/pub/graal-external-deps/native-image/javacpp-shadowed-1.5.7-linux-riscv64.jar     → HTTP 200
```

Both entries are already in `substratevm/mx.substratevm/suite.py`.

---

## Environment

| Item | Value |
|------|-------|
| Board | Banana Pi F3 (SpacemiT K1) |
| ISA | rv64gc |
| RAM | 16 GB |
| Disk | 113 GB (50 GB free at start) |
| OS | Debian 13 (Trixie) |
| Java | OpenJDK 25.0.2 (`openjdk-25-jdk-headless` from apt) |
| mx | 7.79.1 (graalvm/mx, shallow clone) |
| graal | gounthar/graal shallow clone |

**Note:** GraalVM normally builds with a custom "labs-openjdk" (OpenJDK + JVMCI patches + static
library build). Debian's OpenJDK 25 works as the host JVM but is missing static `.a` libraries
(see Error 2 below).

---

## Build log

### Setup

```bash
# JDK 25 required (mx enforces this; JDK 21 is rejected)
sudo apt-get install -y openjdk-25-jdk-headless
export JAVA_HOME=/usr/lib/jvm/java-25-openjdk-riscv64

# mx build tool
git clone --depth=1 https://github.com/graalvm/mx.git ~/mx
export PATH=$HOME/mx:$PATH

# graal fork
git clone --depth=1 https://github.com/gounthar/graal.git ~/graal
```

### Error 1: libffi arch mapping missing riscv64

```
AssertionError: translation to configure style arch is not supported yet for riscv64
  File "truffle/mx.truffle/mx_truffle.py", line 2334
```

**Root cause:** libffi is built as part of Truffle/NFI using autotools. The `--host` argument
is constructed from a dict that only knew about `amd64` and `aarch64`. riscv64 was absent.

**Fix** (commit `gounthar/graal@2b5e4bf9`):

```python
# truffle/mx.truffle/mx_truffle.py, line ~2334
# Before:
configure_arch = {"amd64": "x86_64", "aarch64": "aarch64"}.get(toolchain.spec.target.arch)
# After:
configure_arch = {"amd64": "x86_64", "aarch64": "aarch64", "riscv64": "riscv64"}.get(toolchain.spec.target.arch)
```

**Status:** Fixed, committed, build proceeded.

---

### Error 2: No static JDK libraries (JvmFuncsFallbacks)

```
JvmFuncsFallbacksBuildTask svm-jvmfuncs-fallback-builder failed
Could not find any unresolved JVM_* symbols in static JDK libraries
```

**Root cause:** SubstrateVM generates a C file (`JvmFuncsFallbacks.c`) that stubs out JVM
functions. To do this, it scans static JDK `.a` libraries for undefined `JVM_*` symbols using
`objdump --wide --syms`. No standard JDK distribution for riscv64 includes these static libraries.

The task looks for static libs at:
1. `$JAVA_HOME/lib/static/linux-riscv64/glibc/lib*.a` (preferred)
2. `$JAVA_HOME/lib/lib*.a` (fallback)

Checked all riscv64 JDK distributions - none include static libraries:

| Distribution | riscv64? | Static libs? |
|---|---|---|
| Debian OpenJDK 25 | yes | no |
| Temurin 25 (Adoptium) | yes | no |
| GraalVM labs-openjdk 25.1-b17 | **no** | yes (amd64/aarch64 only) |

**Root fix required:** Build `graalvm/labs-openjdk` for riscv64 with `--enable-static-build`.
This is the correct long-term fix and is a contribution target in its own right.

**mx knows about riscv64:** When running `mx fetch-jdk labsjdk-ce-latest`, it tries:
```
https://github.com/graalvm/labs-openjdk/releases/download/jvmci-25.1-b17/labsjdk-ce-25.0.2+10-jvmci-25.1-b17-linux-riscv64.tar.gz
```
...which 404s. The URL pattern exists; the artifact does not.

---

## Target 2: labs-openjdk for riscv64

Building and publishing labs-openjdk for riscv64 would:
1. Provide the static libraries needed by GraalVM's build system
2. Enable GraalVM native-image to be built natively on riscv64
3. Be a standalone contribution to the ecosystem (RISE submission candidate)

The build is standard OpenJDK + JVMCI patches + `--enable-static-build`. The JVMCI patches
are maintained in `graalvm/labs-openjdk`. OpenJDK 25 already has full riscv64 support upstream.

Build plan (to be executed):
```bash
git clone https://github.com/graalvm/labs-openjdk.git
cd labs-openjdk
# Standard OpenJDK build + static libs
bash configure --with-jvm-features=graal --enable-static-build \
    --with-jvm-variants=server --disable-warnings-as-errors
make images static-libs-image
```

Expected build time on F3: several hours.

---

## Non-fatal warnings (to address later)

```
WARNING: No platform-specific definition is available for distribution MUSL_CMAKE_TOOLCHAIN for your architecture (riscv64)
WARNING: No platform-specific definition is available for distribution MUSL_NINJA_TOOLCHAIN for your architecture (riscv64)
```

These affect musl libc static builds only. Not needed for the initial glibc-based native-image.
Fix: add riscv64 entries to the MUSL toolchain definitions in `substratevm/mx.substratevm/suite.py`.

---

## Summary of fixes needed

| Fix | File | Complexity | Status |
|-----|------|-----------|--------|
| libffi arch mapping | `truffle/mx.truffle/mx_truffle.py` | Trivial (1 line) | Done |
| labs-openjdk riscv64 build | `graalvm/labs-openjdk` | Multi-hour build | Planned |
| MUSL toolchain riscv64 | `substratevm/mx.substratevm/suite.py` | Small config | Pending |
| (potential) Further build errors | TBD | TBD | Pending |

---

## Open questions

1. What further errors appear after static libs are available?
2. Does the bundled LLVM 13 JAR for riscv64 function correctly (built Sep 2022)?
3. Can `native-image --llvm` compile a Hello World on riscv64?
4. Does LLVM 13 support RVV (RISC-V Vector extension)? (SpacemiT K1 has RVV 1.0)

---

## Related issues / PRs

- [oracle/graal#13351](https://github.com/oracle/graal/issues/13351) - feature request (filed by us)
- [jreleaser/jreleaser#2062](https://github.com/jreleaser/jreleaser/pull/2062) - the original motivation
- [Rise-dev-appreciation/Rise-dev-appreciation#30](https://github.com/Rise-dev-appreciation/Rise-dev-appreciation/issues/30) - RISE submission, marked Future

---

*Log started: 2026-04-17. Updated incrementally as experiment progresses.*
