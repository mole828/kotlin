# Revision 2: explicit Float32 test boundaries

Source commit: `a1e50f1f36ad0ad5d36379b5a24743aa11d4dffc`.
Parent: `f389eea7a8bb1c4fd88e09c3eb6037811361eeae`.
Date: 2026-09-22. Host: Linux x64, 5.8 GiB RAM.

## Change

Construct the Float test inputs from `0x4f000000` (2^31) and `0x5f000000`
(2^63). Kotlin/JS can retain 2147483647 when converting Int.MAX_VALUE to
Float, so the previous test incorrectly expected its negation to round
to Int.MIN_VALUE. The explicit bit patterns give all backends the same
boundary values. Expected results and test coverage are unchanged.

Only MathTest.kt changes in this revision. Native/Wasm implementation
files are byte-identical to the parent; no JS implementation is changed.

## Local results

| Check | Passed | Failed | Skipped |
| --- | ---: | ---: | ---: |
| Parent JS latestTarget, method-level filter | 34 | 1 | 1 |
| Revised JS ordinary target | 35 | 0 | 1 |
| Revised JS latestTarget | 35 | 0 | 1 |
| Revised Wasm-JS | 35 | 0 | 1 |
| Revised JVM | 35 | 0 | 1 |
| Revised MathTest + old Native library, default | 31 | 4 | 1 |
| Revised MathTest + fixed Native library, default | 35 | 0 | 1 |
| Revised MathTest + fixed Native library, -opt | 35 | 0 | 1 |

The parent JS run exited 1 with the reported failure:

```text
-2147483647.roundToInt(). Expected <-2147483648>, actual <-2147483647>.
```

The four revised Gradle tasks completed together with exit 0 in 7m 24s.
Each XML report contains exactly 36 math tests, and all six added methods
were individually checked to have executed without failures or skips.
The sole skip is the existing FloatMathTest.floatRangeConversion.

## Commands and tools

Baseline JS task:

```bash
./gradlew :kotlin-stdlib:jsNodeLatestTargetTest --tests 'test.numbers.*MathTest.*'
```

Revised tasks, explicitly filtering both JS executions:

```bash
./gradlew \
  :kotlin-stdlib:jsNodeTest --tests 'test.numbers.*MathTest.*' \
  :kotlin-stdlib:jsNodeLatestTargetTest --tests 'test.numbers.*MathTest.*' \
  :kotlin-stdlib:wasmJsNodeTest --tests 'test.numbers.*MathTest' \
  :kotlin-stdlib:jvmTest --tests 'test.numbers.*MathTest'
```

Both commands additionally used:

```text
--info --no-daemon --no-configure-on-demand --no-configuration-cache
-Pkotlin.native.enabled=false
-Pkotlin.compiler.execution.strategy=in-process
-Pkotlin.build.internal.gradle.setup=false
--max-workers=1
-Dorg.gradle.jvmargs=-Xmx1536m -XX:MaxMetaspaceSize=1536m -XX:+UseSerialGC
```

The JVM-arguments option was passed as one quoted shell argument.
JAVA_HOME and JDK_17_0 referred to JDK 17.0.2; the Gradle daemon used its
provisioned JDK 21. Gradle 9.7.1; Kotlin bootstrap 2.5.0-dev-7307.
Actual Node executables: 24.16.0 for JS and 26.2.0 for Wasm-JS.
The JVM test executor used Temurin 1.8.0_504, confirmed from its launch path.
These Gradle tasks built/linked the checkout's stdlib, not an installed
release library.

Native runs compile the revised complete MathTest.kt with the existing
source-built 2.5.255-SNAPSHOT compiler/runtime and JDK 17, explicitly using
`-target linux_x64 -nostdlib -no-default-libs -library <selected-stdlib>`.
The language flags are `-language-version 2.5
-Xname-based-destructuring=complete -generate-test-runner`; the optimized
run adds `-opt`. Each generated executable is actually run.

The selected Native libraries are the separate, source-built artifacts
recorded in RESULTS.md: test-only c82eca7a0 and fixed f389eea7a. Reusing them
is intentional: this revision changes only test construction, and the
Native/Wasm implementation files were checked unchanged. The same new
MathTest source is used against both libraries. This is targeted library
validation, not a new full Native distribution or full-suite run.

## Scope and historical reports

The user's macOS and Windows reports concern f389eea7a. They report Native
math checks passing on macos_arm64 and mingw_x64; Windows additionally
reports the official full Native stdlibTest passing (3332 passed, one
skip). macOS also reports a passing Wasm-WASI task. Those raw remote logs
and XML were not supplied here, and those runs must not be relabeled as
executions of a1e50f1f3.

This revision does not claim a new Windows/macOS run, a new full Native
suite run, or a new Wasm-WASI run. The local XML, Native console output and
machine-readable summaries are stored under evidence/revision-2/.

README now uses method-level JS filters, requires nonzero discovered test
counts, and treats the WASI task as an explicit full-task run rather than
assuming the reported filtering behavior works. Runtime versions must be
read from the actual launched executable, not inferred from PATH.

Console logs in Git have trailing whitespace removed; test outcomes are unchanged.
