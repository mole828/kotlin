# KT-89074 跨平台验证说明（个人 fork 草案）

目的：验证 `roundToInt` / `roundToLong` 在 Float、Double 上的行为，尤其是
半整数相邻值、已经为整数的大数、Int/Long 范围边界、NaN 和无穷大。
这份说明不是上游 PR 描述，也不代表所有平台已通过。

算法说明见 [REASONING.md](REASONING.md)，最新本机记录见 [REVISION-2.md](REVISION-2.md)，首版记录见 [RESULTS.md](RESULTS.md)。

## 固定版本

- 仓库：<https://github.com/mole828/kotlin>
- 改动分支：`fix/kt-89074-multiplatform`
- 上游基线：`9b478d0341465fabaad64b3701d40b504c1a047f`
- 只有测试的提交：`c82eca7a0`（Native/Wasm 仍是旧实现）
- 当前源码提交：a1e50f1f36ad0ad5d36379b5a24743aa11d4dffc
- 首版实现提交：`f389eea7a`（其 Float 边界测试在 JS 上有错误假设）
- 本说明放在另一个分支 `validation/kt-89074-multiplatform`，不属于待评审源码补丁。

保护已有工作，使用独立 checkout/worktree。确认实际 HEAD，不要顺手更新到新的
master 或混入旧 PR 的其他改动。先阅读仓库规范、ReadMe.md、kotlin-native/README.md。
没有 JetBrains IDE MCP 时，本验证任务允许使用 CLI。

## 机器分工

| 机器 | 首要运行目标 | 额外的通用后端检查 |
| --- | --- | --- |
| macOS M2 | Native `macos_arm64`，默认模式和 `-opt` | 可补 JVM、JS |
| Windows x64 | Native `mingw_x64`，默认模式和 `-opt` | 优先补 JVM、JS |
| Linux x64 | Native `linux_x64`，默认模式和 `-opt` | Wasm-JS、Wasm-WASI，必要时 JVM/JS |

三个宿主运行结果不能代替 iOS、Android、Linux ARM64 等其他 Native 目标。
WSL 的 Native 结果属于 Linux，不能算作 Windows `mingw_x64`。
M2 请检查 JDK/编译器/产物架构，避免将 Rosetta x64 结果记作 arm64。

先记录 OS、CPU 架构、RAM、可用磁盘、JDK 和 Gradle 版本。依赖以本提交的
构建配置为准。macOS 源码构建要求见 kotlin-native/README.md（本基线写的是
Xcode 26.4+，ARM64 JDK）；Windows 见同文件的 MSVC/SDK 要求。
JDK 可由仓库自动供应；失败时再按错误配置 JDK_17_0、JDK_25_0 等变量。
不要直接照搬另一台机器的绝对路径。

## JVM / JS / Wasm

在仓库根目录执行真实 Gradle 任务，测试文件保持为仓库中的原始完整
`libraries/stdlib/test/numbers/MathTest.kt`。Gradle 会构建本分支 stdlib。

本基线配置和任务清单中存在的任务（启动前核对环境）：

```text
:kotlin-stdlib:jvmTest --tests test.numbers.*MathTest
:kotlin-stdlib:jsNodeTest --tests 'test.numbers.*MathTest.*' :kotlin-stdlib:jsNodeLatestTargetTest --tests 'test.numbers.*MathTest.*'
:kotlin-stdlib:wasmJsNodeTest --tests test.numbers.*MathTest
:kotlin-stdlib:wasmWasiWasmtimeTest
```

JS 的 `jsNodeTest` 还依赖 latestTarget 测试运行。上面显式列出两个任务并
分别传入筛选参数，不依赖命令行选项自动传播。两组报告分别记录。
JS 使用包含方法名的 `test.numbers.*MathTest.*`；旧的类级模式可能零测试成功。
两组 JS 报告都必须实际发现 36 项数学测试，六个新增方法必须执行，不能只看退出码。
在 shell 中请给筛选表达式加引号，避免通配符被 shell 展开。
Wasm-WASI 在此基线使用 Wasmtime。macOS 报告中的筛选没有收窄运行范围，
因此上面明确运行此任务全量，再单独统计三个 MathTest 类。记录实际执行数量，
不要将任务全量结果称为仅运行 MathTest。筛选失效的运行器原因尚未独立确认。
Gradle 可以供应 Node/Wasmtime；PATH 中没有它们不代表任务无法运行。
从实际启动命令核对运行时路径和版本，不要只记录系统 node --version。

Linux/macOS 使用 `./gradlew`，Windows PowerShell 使用 `./gradlew.bat`。
PowerShell 中将每个 `-P...` 参数整体放入单引号，避免属性名被拆分。
以下参数是本次资源受限主机的参考，不是统一构建要求：

```text
--no-daemon --no-configure-on-demand --no-configuration-cache
-Pkotlin.native.enabled=false
-Pkotlin.compiler.execution.strategy=in-process
-Pkotlin.build.internal.gradle.setup=false
--max-workers=1
```

JVM 参数应作为一个带引号的参数传入，例如：
`"-Dorg.gradle.jvmargs=-Xmx2g -XX:MaxMetaspaceSize=1536m"`。
按机器资源调整；不要为小内存机器直接使用仓库默认的 6 GiB 堆。
不要并行启动多个重型构建。保留完整日志和 XML 测试报告。

## Native

官方完整 stdlib 测试入口：

```text
./gradlew :native:native.tests:stdlibTest -Pkotlin.native.enabled=true
```

需要固定宿主目标时，加 `-Pkotlin.internal.native.test.target=macos_arm64`
（Windows 为 `mingw_x64`，Linux 为 `linux_x64`）。
其来源是 kotlin-native/runtime/README.md。它会带入编译器、runtime、缓存和
测试设施的大量构建；资源不足时报告未完成，不能用下述定向测试冒充完整套件。
也不要假设 `:kotlin-stdlib:linuxX64Test` 等是此仓库真实验证入口。

源码构建宿主分发包的官方入口：

```text
./gradlew :kotlin-native:dist -Pkotlin.native.enabled=true
```

之后用 `kotlin-native/dist/bin/konanc`（Windows 为 `konanc.bat`）编译完整 MathTest：

```text
kotlin-native/dist/bin/konanc libraries/stdlib/test/numbers/MathTest.kt -language-version 2.5 -Xname-based-destructuring=complete -generate-test-runner -o <输出目录>/math
```

然后实际执行生成的测试程序；再加 `-opt` 编译到另一个文件并执行。
记录默认模式的实际选项，不要把没有 `-g` 的命令称作调试构建。
执行文件通常为 `.kexe`，Windows 为 `.exe`，以实际输出为准。
用编译日志及产物头核对架构（例如 macOS/Linux 的 `file`、Windows 的
`dumpbin /headers`）；需要时在 konanc 命令中显式传入 `-target <目标>`。

如果完整分发包构建资源不足，但已有兼容的 Native 编译器，可以做范围更窄的验证：

1. 运行 `:kotlin-native:runtime:nativeStdlib -Pkotlin.native.enabled=true`，
   从当前源码生成 `kotlin-native/runtime/build/nativeStdlib`。
2. 编译完整 MathTest 时显式传入
   `-nostdlib -no-default-libs -library <该nativeStdlib的绝对路径>`，
   另加上面的语言版本、解构和 test runner 参数，分别默认/`-opt` 运行。
3. 清楚记录编译器/runtime 的来源和版本；这只能称为“当前 stdlib + 指定工具链
   的定向验证”，不能称为“整个当前分支的 Native 分发包/完整套件通过”。

不要只编译一个复制了修复函数的小程序；不要只运行系统里安装的发行版 stdlib。
如果修复前后结果完全一致，先核查链接库和缓存，确认测试执行到了改过的代码。

## 修复前后对照

本轮修订只明确了 Float32 测试边界输入。JS 上比较 `f389eea7a` 与当前提交，
是在验证测试构造修正，不是在修复 JS 的 roundToInt 实现。
Native/Wasm 上这些边界输入的值保持相同；仍可用 `c82eca7a0` 对照取整算法。
注意当前 MathTest 文件与旧提交不同，不要声称两者文件哈希一致。

保留修复版日志之后，切换到只有测试的提交 `c82eca7a0`，重新构建该版本 stdlib
并执行同样测试。也可先测试旧版，再切到修复版。两个版本的日志/产物分目录存放。
不要同时在两个 worktree 上重型构建，也不要将一个版本的 stdlib 产物拿给另一个版本。

预期 Native/Wasm 的以下测试在旧实现失败，在修复版通过：

- DoubleMathTest.roundingConversionNearHalfIntegers
- FloatMathTest.roundingConversionNearHalfIntegers
- DoubleMathTest.roundingConversionAtIntegralPrecision
- FloatMathTest.roundingConversionAtIntegralPrecision

新增 IntegerLimits 测试是保持饱和语义的保护，不要求旧实现失败。
JVM/JS 旧版可能本来就通过；这里用来确认共有契约。
最后回到修复提交，检查工作树无意外源码修改。

## 结果格式

生成 `ROUNDING-VALIDATION-<os>-<arch>.md`，附日志/报告位置：

1. HEAD、OS/架构、内存、工具链/runtime 版本、Native target。
2. 每条完整命令、退出码，以及实际使用的 stdlib 来源。
3. 每组 passed/failed/skipped 数量；列出新测试名称和结果。
4. 修复前失败的具体输入、期望值、实际值；修复后对应结果。
5. 区分编译失败、依赖/环境错误、执行断言失败、未运行。
6. 默认/优化 Native 模式分别记录；完整 stdlibTest 是否完成单列。
7. 不同架构或其他未运行目标明确写“未验证”。

不要为了通过测试而改预期值、跳过新测试或使用旧库。发现问题先定位并报告，
建议代码变更单独保存待审查。此任务不授权创建/重开上游 PR、发评论或自动推送。
