# Local validation — KT-89074 multiplatform draft

Historical results for f389eea7a. See [REVISION-2.md](REVISION-2.md) for
the corrected shared tests and new local runs, including both JS targets.

Date: 2026-09-22. Host: Linux x64, 5.8 GiB RAM, no swap.
Base: `9b478d0341465fabaad64b3701d40b504c1a047f`.
Tests: `c82eca7a00cefe9ef1a7fecbd73203d570046671`.
Fix: `f389eea7a8bb1c4fd88e09c3eb6037811361eeae`.

| Check | Result |
| --- | --- |
| Native stdlib compiled directly from test-only commit | Exit 0, no compiler diagnostics |
| Native stdlib compiled directly from fixed commit | Exit 0, no compiler diagnostics |
| Full MathTest.kt, test-only stdlib, default compilation | 31 passed, 4 failed, 1 skipped; exit 1 |
| Full MathTest.kt, fixed stdlib, default compilation | 35 passed, 0 failed, 1 skipped; exit 0 |
| Full MathTest.kt, fixed stdlib, `-opt` | 35 passed, 0 failed, 1 skipped; exit 0 |
| Official Wasm-JS Gradle MathTest task, fixed commit | 35 passed, 0 failed/errors, 1 skipped; exit 0 |
| Full Native Gradle stdlibTest | Not run on this branch |
| macOS arm64 / Windows x64 Native | Not run here |
| JVM / JS / Wasm-WASI on this branch | Not run here |

The skipped Native test is the pre-existing
`FloatMathTest.floatRangeConversion`. All six new test methods run and pass
on the fixed Native library. Native test compilation produced no diagnostics.

## What was actually built

Two initial attempts to run the official Wasm-JS task ended with
“Gradle build daemon disappeared unexpectedly” during build-tool
configuration/compilation, before the math tests ran. Another project was
using the same small host. The kernel log was unavailable, so these runs
are not described as confirmed OOM kills or test assertion failures.

For Native, the full stdlib was compiled directly with the cached official
bootstrap compiler, `2.5.0-dev-7307`, using the arguments recorded by a
previous successful `:kotlin-native:runtime:stdlibBuildTask` run. Both the
Gradle task implementation and its configuration were verified unchanged
from that run. Checkout/output paths were updated; all 501 current source
files and their fragment assignments were checked against the task's
source directories. No functions were extracted or reimplemented.

The repository was checked out at the test-only commit for the first
build, then at the fix commit for the second. Input SHA-256 inventories
show that only `kotlin-native/runtime/src/main/kotlin/kotlin/math/math.kt`
changed among those 501 inputs. Outputs were stored in separate directories.

MathTest.kt was then compiled against each freshly built library with:

```text
konanc libraries/stdlib/test/numbers/MathTest.kt
  -nostdlib -no-default-libs -library <freshly-built-stdlib>
  -language-version 2.5 -Xname-based-destructuring=complete
  -generate-test-runner -o <separate-output>
```

The fixed test was also compiled with `-opt` and executed separately.
The test compiler/runtime came from the existing source-built Native
`2.5.255-SNAPSHOT` distribution; JRE `17.0.2+8-86`. They were not rebuilt at
the new commit. These are **current stdlib + existing toolchain targeted
runs**, not successful runs of the full Native Gradle test infrastructure.

## Official Wasm-JS run

A third run completed successfully in 14m 24s after other workloads had
released memory. It rebuilt the checkout's Wasm stdlib and kotlin-test,
compiled the full stdlib test sources, and ran the selected math classes.
Bootstrap compiler: 2.5.0-dev-7307; Gradle 9.7.1; Node 26.2.0.

```bash
./gradlew :kotlin-stdlib:wasmJsNodeTest --tests 'test.numbers.*MathTest' --info \
  --no-daemon --no-configure-on-demand --no-configuration-cache \
  -Pkotlin.native.enabled=false \
  -Pkotlin.compiler.execution.strategy=in-process \
  -Pkotlin.build.internal.gradle.setup=false \
  '-Dorg.gradle.jvmargs=-Xmx1536m -XX:MaxMetaspaceSize=1536m -XX:+UseSerialGC' \
  -Dorg.gradle.internal.http.connectionTimeout=15000 \
  -Dorg.gradle.internal.http.socketTimeout=15000 --max-workers=1
```

JDK_17_0 and JAVA_HOME pointed to the installed JDK 17; the Gradle daemon
used the repository-provisioned JDK 21. XML reports are attached under
`evidence/wasmJsNodeTest/`. All six added methods passed. The only skip is
the same pre-existing FloatMathTest.floatRangeConversion.

The test-only Wasm commit did not reach execution in the two earlier
attempts. Before/after regression evidence on this branch is from Native.

## Reproduced failures

In the test-only version, the first failing assertion in each regression
method was:

| Type / input | Operation | Expected | Actual |
| --- | --- | --- | --- |
| Double `0.49999999999999994` | roundToLong | 0 | 1 |
| Double `4503599627370497` | roundToLong | 4503599627370497 | 4503599627370498 |
| Float `0.49999997` | roundToLong | 0 | 1 |
| Float `8388609` | roundToLong | 8388609 | 8388610 |

Each method checks both integer result types; an early failing assertion
ends that method on the old implementation. The fixed run executes all
remaining cases as well. The new integer-limit methods pass before and
after, protecting saturation rather than demonstrating the old defect.

## Evidence and remaining work

The attached Native console logs show the before/after test results;
trailing whitespace was removed for storage in Git.
The accompanying README gives host-specific verification instructions.
Do not infer other Native architectures or Wasm-WASI coverage from Linux
x64 or Wasm-JS results. No performance benchmark was run.
