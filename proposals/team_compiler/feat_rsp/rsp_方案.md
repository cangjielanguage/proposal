# Response File 特性设计

# **1、特性需求/问题/动机的来源与价值**

## 1.1 背景与目标

考虑以下场景:在 Windows 中,当用户向 `gcc` 传递的命令行参数较多时,进程可能直接启动失败,而 `cjc` 也会存在类似的问题。

``` cmd
gcc C:\opt\buildtools\rsp_wintest\main.c -o test  -O2 -pipe -time -dumpmachine  -print-libgcc-file-name -print-sysroot -print-sysroot-headers-suffix -print-multi-os-directory  -o test  -O2 -pipe -time -dumpmachine  -print-libgcc-file-name -print-sysroot -print-sysroot-headers-suffix -print-multi-os-director -o test  -O2 -pipe -time -dumpmachine  -print-libgcc-file-name -print-sysroot -print-sysroot-headers-suffix 
xxx
输入行太长。
```

**根因**:Windows `CreateProcessA` 命令行字符串长度上限为 32767 字符,（cmd 命令行中为 8191）一旦 `cjc` 收到的参数总长超过该值,操作系统直接拒绝启动进程,且无任何兜底。Linux/macOS 虽有 `ARG_MAX`(典型 2 MB)限制,常规工程不易触发,但 Windows 大工程(尤其是链接阶段参数多)极易撞到该硬限制。

### 1.1.1 目标

为绕开上述系统命令行长度限制,本特性为 `cjc` 提供一种"从文件读取命令行参数"的入口。大型工程中,rsp 通常由构建系统根据已有参数数组自动生成,而不是要求用户手写；`cjc` 读取该文件内容、按既定规则切分为参数 token。

**术语**:该文件即业界通称的 **response file(响应文件)**,常取扩展名 `.rsp`(故简称 rsp 文件)——一份纯文本参数清单,由工具自己读取并展开,以绕开 shell/OS 的命令行长度限制。MSVC `link.exe @link.rsp`、GCC/Clang、GNU ld、lld 的 `@file` 都是同一机制。

**关键术语速览**:

| 术语 | 一句话解释 |
| --- | --- |

| **CRT** | C 运行时库(C Runtime Library)。Windows 上程序被启动后,由程序内的 CRT 启动代码把那**一整条字符串**切分成 argv——这套切分规则即后文所说的"CRT 规则 / CRT 风格 / CRT 套",也是 `CommandLineToArgvW` 的规则。MSVC `link.exe`、CMake/Ninja 生成的 rsp、Windows 上的 ld.lld 全部遵循它,它是 Windows 用户"反斜杠路径不转义"习惯的来源(详见附录 A0.3) |
| CRT 风格 | 上文 CRT 规则的引号/反斜杠行为:仅 `"` 是引号;`\` 仅在紧贴 `"` 时特殊(2n 配对),其余位置一律普通字符 |
| bash 风格 | bash shell 的引号/转义语义(不含变量/命令展开):无引号 `\X`→X、单引号内全字面、双引号内仅 5 种转义、`#` 注释、行尾 `\` 续行 |

https://gcc.gnu.org/wiki/Response_Files 



## 1.2 适用场景
- Windows 命令行长度上限 32767 字符,进程可能启动失败;
- 大型工程中源文件、依赖库、链接输入或编译选项数量过多,由构建系统/driver 自动生成 rsp;
- 用户手写 rsp 仅作为调试、复现和小规模配置的辅助场景,不是本特性的主要目标;

## 1.3 实现前 vs 实现后对比

**实现前**:用户必须把所有参数挤在命令行里,Windows 大工程链接时一旦超 32767 字符直接启动失败,无任何兜底。

```bash
# 实现前:命令行参数过长,不便维护
cjc -L"third party/lib" -lfoo -lbar -lbaz -g -O0 src/main.cj
```

**实现后**:构建系统或上游 driver 根据参数数组自动生成 rsp,再启动 cjc。

```bash
# 构建系统生成 compile.rsp 后启动 cjc
cjc @build/compile.rsp
cjc -O0 @build/common.rsp -O2 src/main.cj   # 仍可与普通参数混用
```

# **2、特性影响分析**

本特性只影响 driver 读取命令行选项的阶段:rsp 展开发生在选项解析之前,展开后行为与参数直写命令行完全一致,不改动编译、链接等后续 pipeline。





# **3、业界竞品分析**

## 3.1 主要实现

| 工具 | 用户手册 | 语法真正定义处(源码) |
| --- | --- | --- |
| GCC | <https://gcc.gnu.org/onlinedocs/gcc/Overall-Options.html>(页末 `@file` 段) | libiberty `argv.c` 的 `expandargv` |
| GNU ld | <https://sourceware.org/binutils/docs/ld/Options.html> | binutils `ld/`(与 libiberty 同源) |
| clang / lld | 无专门手册页 | `llvm/lib/Support/CommandLine.cpp`(`ExpansionContext::expandResponseFiles`)+ `lld/Common/Args.cpp` |
| MSVC `link.exe` | <https://learn.microsoft.com/en-us/cpp/build/reference/at-specify-a-linker-response-file> | 闭源,以文档为准 |
| Rust `rustc` | [`@path` 参数文件](https://doc.rust-lang.org/rustc/command-line-arguments.html#path-load-command-line-flags-from-a-path) | [`rustc_driver_impl/src/args.rs`](https://github.com/rust-lang/rust/blob/master/compiler/rustc_driver_impl/src/args.rs)；稳定化 Issue [#63576](https://github.com/rust-lang/rust/issues/63576) / Commit [446a43d](https://github.com/rust-lang/rust/commit/446a43da0ee23b3a69e214f030d6d7dada345296) |
| Rust Cargo | 无独立用户格式文档；构建时自动兜底 | [`cargo-util/process_builder.rs`](https://github.com/rust-lang/cargo/blob/master/crates/cargo-util/src/process_builder.rs) 的 `retry_with_argfile` |
| Swift Driver | [Swift Driver 仓库](https://github.com/swiftlang/swift-driver)；`swiftc` 用户手册未单独定义统一 rsp 语法 | [`Driver.swift`](https://github.com/swiftlang/swift-driver/blob/main/Sources/SwiftDriver/Driver/Driver.swift)、[`ArgsResolver.swift`](https://github.com/swiftlang/swift-driver/blob/main/Sources/SwiftDriver/Execution/ArgsResolver.swift)；演进提交见 [response-file 初始支持](https://github.com/swiftlang/swift-driver/commit/b86325a9)、[Windows 支持](https://github.com/swiftlang/swift-driver/commit/402b22a9) |
| Go `cmd/go`/`cmd/compile` | 无稳定的 `go build @file` 用户文档；主要通过 Issue/CL 演进 | 初始支持 [Issue #18468](https://github.com/golang/go/issues/18468)/[CL 110395](https://go-review.googlesource.com/c/go/+/110395)；转义补丁 [Issue #42295](https://github.com/golang/go/issues/42295)/[CL 272870](https://go-review.googlesource.com/c/go/+/272870)；GCC 兼容 [Issue #77177](https://github.com/golang/go/issues/77177)/[CL 737500](https://go-review.googlesource.com/c/go/+/737500) |
| CMake / Ninja | [Ninja `rspfile` 文档](https://ninja-build.org/manual.html)；CMake 平台模块中的 `CMAKE_<LANG>_USE_RESPONSE_FILE_FOR_*` 配置 | CMake 生成规则，Ninja 在执行前写入并在成功后删除 `rspfile` |
具体字符行为及实验数据见附录 B。

## 3.2 生成者、消费者与触发方式

命令行长度限制发生在父进程创建子进程时，因此**持有完整参数数组的父进程负责生成 rsp，接收参数的子进程负责定义 rsp 格式**。不能要求尚未启动的子进程为自己生成 rsp。

| 调用链 | rsp 生成者 | rsp 消费者 | 典型触发方式 |
| --- | --- | --- | --- |
| Cargo → rustc/rustdoc | Cargo | rustc/rustdoc | Cargo 先按普通 argv 启动；遇到“命令行过长”错误后，将参数逐行写入临时 `cargo-argfile.*`，以 `@file` 重试，执行后清理 |
| Swift Driver → swift-frontend/clang/linker | 待补充 | 待补充| 待补充 |
| Go `go` → compile/link | `go` 构建驱动或相应工具链组件 | compile/link/外部链接器 | Go 没有面向 `go build` 用户的统一 `@file` 协议；具体参数文件属于内部工具链机制,不能直接作为 cjc 的格式先例 |
| CMake + Ninja → compiler/linker | CMake 生成构建规则；**执行时由 Ninja 写入 rsp** | compiler/linker | CMake按编译器/平台能力产生 `rspfile`、`rspfile_content` 规则；Ninja 在执行命令前写文件，成功后删除 |



## 3.3 四类格式及其设计动机

业界主流实现可归纳为以下四类。本文所称“GNU 格式”和“GCC 格式”通常指同一来源：GCC、GNU ld 等 binutils 工具通过 libiberty 的 `buildargv`/`expandargv` 解析 rsp，因此统一记为 **GNU/GCC（libiberty）格式**。不过，libiberty 的规则会随版本演进；LLVM 的 “GNU tokenizer” 是兼容实现，也不保证与任意版本的 libiberty 在所有边界行为上完全一致。

| 格式 | 主要使用者 | 核心规则 | 设计动机与指导思想 |
| --- | --- | --- | --- |
| MSVC/CRT | `link.exe`、`cl.exe`、`lld-link`、`clang-cl`、Windows Swift 工具链 | 空白分隔；仅双引号分组；反斜杠只在紧邻 `"` 时使用 2n/2n+1 规则，不支持 shell 变量、glob、注释和续行。 | 镜像 Windows 命令行和 CRT 的 `argv` 语义，受既有生态兼容约束 |
| GNU/GCC（libiberty） | GCC、GNU ld、`ar` 等 binutils 工具；Go 1.27 开始向其兼容 | 空白分隔；单双引号分组；反斜杠转义；不执行变量、命令和 glob 展开；引号内转义和换行行为以具体 libiberty 版本为准 | 接近 shell 的参数引用子集，兼顾手写和工具生成，已形成 Unix 工具链事实标准 |
| LLVM GNU 兼容变体 | 非 CL 模式的 Clang、lld、Linux/macOS Swift Driver | 使用 LLVM GNU tokenizer，兼容 GNU/GCC 的常用写法，但单双引号、反斜杠、空参数和续行等边界不保证完全一致；Windows/CL 场景切换 Windows tokenizer | 以共享实现服务多个工具和平台；保证主流 GNU rsp 互操作，同时为 Windows 保留独立规则 |
| Rust 零语法 | `rustc` 默认 argfile | 一行一个参数；无引号、无转义、无嵌套；UTF-8；参数内换行不可表示 | 写端和读端受控时，以最小语法换取简单、确定的序列化；增强格式通过显式入口增加 |

三种现代工具链的演进重点不同：Rust 保持基础格式稳定，通过显式入口增加 POSIX shell 风格（并非严格 GNU/GCC 兼容）的格式；Go 从私有逐行协议转向 GNU/GCC 兼容格式；Swift 的 tokenizer 基本稳定，主要演进生成时机、能力声明和进程内/进程外执行边界。详细过程见附录 B.3。


# **4、本特性的设计/实现方案**

## 4.1 选项定位与设计定位

cjc 提供编译选项 `@<file>`:以 `@` 开头、后跟 rsp 文件路径构成一个命令行参数,如 `@build/options.rsp`,用于在编译时从文件中读取编译参数。cjc 读取 `<file>` 中的参数序列,**就地替换**该 `@<file>` 参数;替换后 cjc 的行为,与用户把这些参数手工写在命令行该位置**完全等价**。


本特性首先解决“大型工程中父进程无法用普通命令行启动 cjc”的问题。

```text
cjpm / Ninja / CMake
    └─ 根据完整 argv 生成 rsp
         └─ cjc 读取 rsp 并恢复 argv
              └─ 进入既有选项解析和编译流程
```

因此，rsp 的首要定位是构建系统与 cjc 之间的参数传递协议。

## 4.2 接口与展开流程

cjc 接受 `@<file>` 参数：

```text
cjc -O0 @build/compile.rsp -O2
```

rsp 在选项解析前就地展开，展开后的参数顺序与直接写在命令行中一致：

```text
命令行： cjc -O0 @build/compile.rsp -O2
rsp：    -Lthird-party/lib
         src/main.cj
结果：   cjc -O0 -Lthird-party/lib src/main.cj -O2
```

通用处理流程如下：

1. 识别 `@<file>`
2. 读取文件并按所选格式恢复参数序列；
3. 在原位置替换 `@<file>`；
4. 将完整参数数组交给现有选项解析器。

文件不存在、不可读、编码非法或格式错误时，在展开阶段直接报错。具体错误码和嵌套路径规则见附录 A0.6、A0.7。

## 4.3 候选方案与业界格式的关系

第三章的四类格式是业界协议分类，下面的四个方案是 cjc 的候选设计，两者不是一一对应关系。各方案与业界格式的关系如下：

| 候选方案 | MSVC/CRT | GNU/GCC | LLVM GNU 变体 | Rust 零语法 | 定位 |
| --- | --- | --- | --- | --- | --- |
| 方案一：LLVM GNU + Windows 裸路径扩展 | 不兼容 | 常用子集兼容 | 除无引号 `\` 外对齐 | 不兼容 | 以 LLVM GNU 为基线，仅将无引号 `\` 改为字面字符，使 Windows 裸路径可直接书写 |
| 方案二：LLVM GNU | 不兼容 | 常用子集兼容 | 兼容 | 不兼容 | 全平台直接采用 LLVM GNU tokenizer，优先兼容 Clang/lld/Swift 及其上游生成器 |
| 方案三：Swift/LLVM 分平台规则 | Windows 端兼容 | POSIX 端常用子集兼容 | POSIX 端兼容 | 不兼容 | Windows 使用 LLVM Windows/CRT tokenizer，Linux/macOS 使用 LLVM GNU tokenizer |
| 方案四：Rust 默认 argfile | 不兼容 | 不兼容 | 不兼容 | 兼容 | 一行一个参数，整行原样作为 payload；不解析引号、转义和嵌套 |

由此可见，现有候选方案中：

- 方案一是 LLVM GNU 的单点扩展，不再是独立设计的一整套 shell 语法；
- 方案二严格采用 LLVM GNU 格式；
- 方案三采用 Swift/LLVM 的双 tokenizer 结构；
- 方案四严格采用 Rust 默认 argfile 格式。

需要注意，LLVM GNU 与 GNU/GCC（libiberty）只有常用写法兼容，不应宣称逐项等价。Ninja 本身只负责写入 `rspfile_content`，上游生成规则仍需明确按 LLVM GNU、Windows CRT 或 Rust 逐行格式编码。

详细字符规则、示例和逐项比较见附录 A0。

## 4.4 方案选择依据

方案选择取决于 rsp 是否属于公开互操作协议。

### 4.4.1 封闭链路：生成端与读取端均由项目控制

如果首版仅服务 `cjpm → cjc`，且两端同步升级，可以采用严格的一行一参格式。建议遵循以下约束：

- 一行对应一个参数，除行终止符外不裁剪内容；
- 无引号、转义、注释、续行和自动嵌套；
- UTF-8 编码，兼容 LF/CRLF；
- 参数内换行和 NUL 不可表示，生成时直接报错；
- 空参数是否支持需单独决定；若支持，必须明确区分空记录与文件末尾换行；
- 生成器和读取器以 `decode(encode(argv)) == argv` 为验收标准。

这一路线即方案四（Rust 默认 argfile）。

### 4.4.2 公开链路：需要接收外部构建系统生成的 rsp

如果要求 Ninja、CMake、Bazel 或其他工具直接生成 cjc rsp，应选择明确的生态格式，而不是新增混合规则：

- POSIX/GNU 生态采用 GCC 或 LLVM GNU tokenizer；
- Windows/MSVC 生态采用 CRT tokenizer；
- 必须明确具体兼容目标和版本差异，并建立相应兼容测试。

此时方案二可作为 LLVM 生态兼容格式；方案三适合同时覆盖 Windows CRT 和 POSIX LLVM GNU。方案一属于带扩展的 LLVM GNU，必须让生成端知道其无引号反斜杠差异。

### 4.4.3 推荐的演进方向

建议将稳定基础格式与生态兼容格式显式分层，不根据文件内容自动猜测：

```text
@args.rsp          基础格式
@llvm:args.rsp     可选：LLVM GNU 兼容格式
@windows:args.rsp  可选：MSVC/CRT 兼容格式
```

首版只需实现已经确认的基础格式；格式前缀作为扩展空间预留。是否立即实现 `@gcc:` 或 `@windows:`，取决于 cjpm/Ninja/CMake 的接入要求。

## 4.5 生成策略

格式定义与生成时机相互独立。不同调用链可采用不同策略：

| 调用链 | 建议策略 |
| --- | --- |
| cjpm → cjc | 构建规则式或预判式；cjpm 持有完整 argv，负责写入并清理 rsp |
| Ninja/CMake → cjc | 由生成器声明 rsp 规则，执行器在启动 cjc 前落盘 |
| cjc → linker | cjc 根据目标 linker 能力和平台限制预判生成，或捕获命令行过长错误后重试 |
| 用户直接调用 cjc | 用户可显式提供 `@file`，但不承担自动生成职责 |

临时文件应使用唯一名称并避免并发冲突；正常结束后清理，调试或 `save-temps` 模式下可选择保留。

## 4.6 分阶段范围

为避免把读取协议、构建系统集成和链接器兜底混在一个特性中，建议分阶段实施：

1. **cjc 读端**：确定基础格式，实现 `@file` 展开、校验和诊断；
2. **构建系统写端**：cjpm/Ninja/CMake 集成按阈值或规则自动生成 cjc rsp；
3. **下游工具写端**：cjc 为 linker/frontend 生成其原生格式的 rsp；
4. **兼容方言**：仅在存在明确外部互操作需求时增加，并通过显式前缀选择。


# **5、结论**

## 5.1 评审结论

(待评审后填写:通过 / 不通过 + 达成一致的主要观点。)

## 5.2 遗留问题


# **附录 A0:章节4的方案细节**

本附录保留四个候选方案的字符级规则、用例和诊断细节；正文第四章只讨论方案定位与选择依据。

## A0.1 方案一：LLVM GNU + Windows 裸路径扩展

本方案以 LLVM `TokenizeGNUCommandLine` 为基线，仅修改**无引号反斜杠**的处理：LLVM GNU 中无引号 `\X` 会消费反斜杠并保留 `X`；本方案改为原样保留 `\X`。其他空白分隔、单双引号、引号拼接等规则保持 LLVM GNU 语义。

| 元素 | 规则 |
| --- | --- |
| 编码与换行 | UTF-8；忽略 UTF-8 BOM；CRLF 按空白处理；非法 UTF-8 或 NUL 报错 |
| 分隔 | 空格、Tab、LF、CR、FF、VT 等空白切分参数 |
| 引号 | `'...'`、`"..."` 均用于分组，引号本身不进入参数；允许与无引号片段拼接 |
| 反斜杠（引号内） | 沿用 LLVM GNU tokenizer 的转义规则 |
| 反斜杠（无引号） | **扩展点**：一律字面保留，因此 `C:\Users\x` 原样；`a\ b` 被切成 `a\` 和 `b` |
| 注释、变量、glob | 不支持注释；`#`、`$`、反引号、`~`、glob 均为普通参数字符，不执行展开 |
| 嵌套 | 是否支持 `@child.rsp` 属于文件级能力，不属于 LLVM GNU tokenizer；若首版面向工具生成，可不支持 |

该方案能够直接读取大量 LLVM/GNU 常见 rsp，但凡依赖无引号反斜杠转义的输入都会产生不同结果，因此只能称为“LLVM GNU + 扩展”，不能声明严格 LLVM GNU 兼容。

典型差异：

```text
输入                    LLVM GNU              方案一
C:\Users\x              C:Usersx              C:\Users\x
a\ b                    a b（一个参数）        a\、b（两个参数）
\#flag                  #flag                  \#flag
```

## A0.2 方案二：LLVM GNU

本方案全平台直接采用 LLVM 的 GNU tokenizer，不增加 cjc 私有字符规则。其目标是使 cjc 对 Clang、lld、Linux/macOS Swift Driver 及按 LLVM GNU 规则生成的 rsp 得到相同参数序列。

| 元素 | 规则 |
| --- | --- |
| 编码与分隔 | UTF-8；空白分隔；CRLF 兼容 |
| 引号 | 单引号和双引号均可分组，引号本身不进入参数；支持引号片段拼接 |
| 反斜杠 | 按 LLVM `TokenizeGNUCommandLine` 处理；无引号 Windows 路径中的反斜杠可能被消费 |
| 注释 | 不支持；行首 `#` 是普通参数 |
| 展开 | 不执行变量、命令、波浪号或 glob 展开 |
| 格式错误 | 由 cjc 在 LLVM tokenizer 结果之外补充文件路径、行号和内容校验诊断 |

常见写法：

```text
-O2 -g                         → 两个参数
-L"third party/lib"            → 一个参数 -Lthird party/lib
a\ b                          → 一个参数 a b
# comment                     → 两个普通参数 #、comment（不是注释）
```

LLVM GNU 与 GNU/GCC（libiberty）共享大量常见写法，但在单引号内反斜杠、空参数、续行和未闭合引号等边界上可能不同，因此本方案兼容目标应明确写作 **LLVM GNU**。

## A0.3 方案三：Swift/LLVM 分平台规则

本方案参照 Swift Driver/LLVM 的做法，由宿主平台选择 tokenizer：

- Windows：LLVM Windows tokenizer，目标语义为 MSVC/CRT；
- Linux/macOS：LLVM GNU tokenizer，与方案二相同。

文件级机制（UTF-8、CRLF、读取错误和内容校验）保持一致；字符级解析由所选 tokenizer 决定。因此同一 rsp 若使用平台特有写法，不保证跨平台得到相同参数序列。

### A0.3.1 Windows：MSVC/CRT 规则

| 字符 / 序列 | 处理规则 | 示例 → 解析结果 |
| --- | --- | --- |
| 空白 | 引号外切分参数 | `-O2 -g` → 两个参数 |
| `"` | 引号开合，引号内空白不切分 | `-L"C:\Program Files\lib"` → `-LC:\Program Files\lib` |
| 2n 个 `\` + `"` | 输出 n 个 `\`，引号正常开合 | `"C:\lib\\"` → `C:\lib\` |
| 2n+1 个 `\` + `"` | 输出 n 个 `\` 和字面 `"` | `\"` → `"` |
| `\` + 非 `"` | 反斜杠字面保留 | `C:\Users\x` → 原样 |
| `'` | 普通字符，不参与分组 | `-L'x y'` → 两个参数 |
| `#`、`$`、glob | 普通字符，不执行注释或展开 | `#` → 普通参数 |

### A0.3.2 POSIX：LLVM GNU 规则

与方案二完全相同，直接使用 LLVM GNU tokenizer，不再维护一套 cjc bash 变体。

方案三与 Swift 的共同点在于：格式选择由宿主/消费者确定，Windows 硬兼容边界使用 CRT 规则，POSIX 使用 LLVM GNU 规则。代价是同一文件可能因宿主平台不同而得到不同结果。

## A0.4 方案四：Rust 默认 argfile

本方案严格采用 Rust 默认 argfile 的模型：**每行一个参数，行内容原样作为 payload**。

| 元素 | 规则 |
| --- | --- |
| 编码 | UTF-8；非法 UTF-8 直接报错 |
| 分隔 | 使用文本行作为参数边界；兼容 LF/CRLF |
| 行内容 | 除行终止符外全部原样保留，包括行首尾空格、引号、反斜杠和 `#` |
| 引号与转义 | 不存在；`"`、`'`、`\\` 都是普通字符 |
| 空参数 | 空行表示一个空参数；文件末尾换行本身不额外产生参数 |
| 参数内换行 | 不可表示；生成器遇到含 LF/CR 的参数时直接报错 |
| 嵌套 | 不支持；rsp 内的 `@child.rsp` 是普通参数，不递归展开 |
| 注释 | 不支持；`# comment` 是一个普通参数 |
| NUL | 不支持，读取时直接报错 |

示例：

```text
-O2                             → -O2
-Lthird party/lib               → -Lthird party/lib（一个参数）
  -DMSG=a  b                    → 保留行首、行尾和中间空格
C:\Users\test\main.c            → 原样
"-DMSG=hello world"            → 引号也进入参数值
@child.rsp                      → 普通参数，不递归展开
<空行>                          → 空参数
```

方案四的优点是写端与读端都不需要 quoting 算法；限制是无法直接读取 Ninja/CMake 按 GNU/MSVC quoting 生成的 rsp。若后续需要兼容格式，应通过显式方言入口增加，而不是修改默认逐行格式。

## A0.5 四方案对比

| 维度 | 方案一：LLVM GNU + 裸路径扩展 | 方案二：LLVM GNU | 方案三：Swift/LLVM 分平台 | 方案四：Rust argfile |
| --- | --- | --- | --- | --- |
| 一行多个参数 | 支持 | 支持 | 支持 | 不支持 |
| Windows 裸反斜杠路径 | 支持 | 不支持，反斜杠可能被消费 | Windows 支持；POSIX 不支持 | 支持，整行原样 |
| LLVM GNU rsp | 除无引号反斜杠外兼容 | 兼容 | POSIX 兼容 | 不兼容 |
| MSVC/CRT rsp | 不兼容 | 不兼容 | Windows 兼容 | 不兼容 |
| 引号、转义 | LLVM GNU，唯独无引号 `\` 字面 | LLVM GNU | 按宿主选择 CRT/LLVM GNU | 无语法，引号和 `\` 均为 payload |
| 注释 | 不支持 | 不支持 | 不支持 | 不支持 |
| 空参数 | 按 LLVM GNU 行为 | 按 LLVM GNU 行为 | 按所选 tokenizer | 空行表示空参数 |
| rsp 内嵌套 | 可作为 cjc 文件级扩展，但非 tokenizer 固有能力 | 同左 | 同左 | 不支持，`@` 行为普通参数 |
| 跨平台同义 | 是 | 是 | 否 | 是 |
| 实现成本 | LLVM GNU tokenizer 单点修改 | 直接复用 LLVM GNU tokenizer | 同时复用 LLVM Windows/GNU tokenizer | 最低，逐行读取 |
| 适用场景 | 希望兼容 LLVM 常见写法并优先支持 Windows 裸路径 | 需要直接读取 LLVM 生态 rsp | 同时对齐 Windows 与 POSIX 本地生态 | cjpm 与 cjc 成对控制的机器协议 |

关键取舍：

- 方案一以少量 LLVM 兼容性换取 Windows 裸路径直写；
- 方案二优先复用和兼容 LLVM 生态；
- 方案三优先平台原生兼容，接受跨平台语义不同；
- 方案四优先协议简单和可逆，不承诺读取外部 GNU/MSVC rsp。

## A0.6 错误诊断

| 错误类型 | 触发条件 | 报错信息 |
| --- | --- | --- |
| `response_file_read_failed` | 文件存在但读不了(权限、IO 错误) | 含路径与系统错误原因 |
| `not_a_file` | `@xxx` 指向目录 | 含路径 |
| `response_file_not_found` | `@xxx` 指向的文件不存在(`@` 起始一律按 rsp 引用,无兜底) | 含路径 |
| `response_file_cyclic_include` | 当前引用链栈中再次出现同一文件 | 含循环文件路径 |
| `response_file_max_depth_exceeded` | 嵌套超过 32 层 | 含文件路径与最大深度 |
| `response_file_unterminated_quote` | 引号内遇裸 `\n`、文件末尾引号未闭合、或双引号末尾 `\` 后无字符(转义序列未完成) | 含文件路径与行号 |
| `response_file_invalid_content` | 文件解码失败(非法 UTF-8 字节)或含 NUL 字节(误传二进制文件的兜底) | 含文件路径与行号 |

**与业界的有意差异:"文件不存在"直接报错**:`@` 起始参数一律按 rsp 引用,指向的文件不存在时报 `response_file_not_found`。业界做法不同:GCC(libiberty)对"不存在**或不可读**"都原样保留,lld 只对"不存在"保留——本设计有意收紧,理由:①`@` 语义唯一("是 rsp 引用"),不存在就响亮失败,避免把 `@xxx` 静默当输入文件名流出;②字面 `@` 文件名有统一的转义表达(`\@`),不再需要兜底(决策记录见附录 A0.1)。

## A0.7 嵌套引用与路径解析基准例子

嵌套引用只有一条核心规则:**`@child.rsp` 的相对路径按"父 rsp 文件所在目录"解析,而不是按 cwd**;只有命令行顶层的 `@<file>` 才按 cwd 解析。`@` 后写绝对路径(如 `@D:\cfg\common.rsp`、`@/home/x/common.rsp`)时忽略基准目录、直接按该路径打开——使用者 cd 到任何目录都不影响绝对路径引用;相对路径引用则保持"整个 rsp 目录树拷到别处仍可用"的自包含特性。

假设工程目录如下,在 `project/` 下执行 `cjc @build/debug.rsp src/main.cj`:

```
project/
├── build/
│   ├── debug.rsp      内容: @common.rsp -g -O0
│   ├── common.rsp     内容: -L"third party/lib" -lfoo @sub/deps.rsp
│   └── sub/
│       └── deps.rsp   内容: -lbar -lbaz
└── src/
    └── main.cj
```

三次 rsp 引用的路径解析:

| 引用 | 写在哪 | 解析基准 | 实际打开的文件 |
| --- | --- | --- | --- |
| `@build/debug.rsp` | 命令行 | cwd = `project/` | `project/build/debug.rsp` |
| `@common.rsp` | debug.rsp 内 | 父 rsp(debug.rsp)所在目录 = `build/` | `project/build/common.rsp` |
| `@sub/deps.rsp` | common.rsp 内 | 父 rsp(common.rsp)所在目录 = `build/`(注意不是 cwd `project/`) | `project/build/sub/deps.rsp` |

就地按序展开:

```
cjc @build/debug.rsp src/main.cj
→ cjc @common.rsp -g -O0 src/main.cj                            # 展开 debug.rsp
→ cjc -Lthird party/lib -lfoo @sub/deps.rsp -g -O0 src/main.cj  # 展开 common.rsp(双引号去掉,空格保留)
→ cjc -Lthird party/lib -lfoo -lbar -lbaz -g -O0 src/main.cj    # 展开 deps.rsp
```

最终等价于:`cjc -Lthird party/lib -lfoo -lbar -lbaz -g -O0 src/main.cj`

注:展开过程中嵌套的 `@common.rsp`、`@sub/deps.rsp` 虽然形式上出现在了参数序列里,解析基准仍以"父 rsp 所在目录"为准(见上表),不随展开位置变成 cwd。

# **附录 A1:方案一规则的辅助文法**

> **定位**：本附录不是用户规范，也不是实现接口；附录 A0.1 的规则表才是方案一的规范来源。本附录仅提供一份简化辅助文法，便于描述规则结构。



## A.1 简化辅助文法（仅供参考）

```ebnf
(* 为便于阅读，非终结符名称采用“作用_内容”命名。 *)
rsp_file  = { line_with_lf } [ final_line ] ;
line_with_lf = { whitespace_char } [ line_comment | argument_line ] LF ;
final_line = { whitespace_char } ( line_comment | argument_line )
           | whitespace_char { whitespace_char } ; (* 末行可无 LF、以 EOF 结束；必须至少消费一个字符，避免空串歧义 *)
line_comment = "#" { any - LF } ;
argument_line = argument { whitespace_run argument } [ whitespace_run line_comment ] { whitespace_char } ;
whitespace_run = whitespace_char { whitespace_char } ;
whitespace_char = " " | TAB | VT | FF | CR ;
argument = ( unquoted_start_char | quoted_segment ) { unquoted_char | quoted_segment } ;
unquoted_start_char = any - ( whitespace_char | LF | '"' | "'" | "#" ) ;
unquoted_char = any - ( whitespace_char | LF | '"' | "'" ) ;
quoted_segment = double_quoted | single_quoted ;
double_quoted = '"' { double_quoted_char } '"' ;
double_quoted_char = any - ( '"' | '\\' | LF )
                   | '\\' double_quote_escape ;
double_quote_escape = '\\' | '"' | '`' | '$' | '@'
                    | LF
                    | any - ( '\\' | '"' | '`' | '$' | '@' | LF ) ;
single_quoted = "'" { any - ( "'" | LF ) } "'" ;
```

> **记号约定**：本节使用的是“扩展 EBNF”。其中 `any - X` 表示字符集合差集；`TAB`、`VT`、`FF`、`CR`、`LF` 分别表示对应的控制字符；产生式中的 `'\\'`（即 BS）表示反斜杠字符。词法分析前先把 CRLF 规范化为单个 LF，孤立 CR 仍按 `sep` 处理；这样状态表无需额外增加 CRLF 组合状态。实现时按字符码点匹配这些终结符，不再进行第二层字符串转义。

> 本附录的后续内容仅作理解辅助；历史性的完整证明、状态机和测试记录不属于方案规范正文。

### A.1.1 关键产生式说明

`argument` 表示一个完整参数：首片段不能以裸 `#` 开始，后面可以继续拼接普通字符或引号片段。因此：

```text
pre"middle text"post  →  一个参数：premiddle textpost
pre "middle text" post →  三个参数
```

`quoted_segment` 只是参数中的一个片段，不是完整参数：

```ebnf
quoted_segment = double_quoted | single_quoted ;
```

所以 `"..."`、`'...'` 可以单独构成参数，也可以和普通字符拼接。`""`、`''` 的内容可以为空，因此各自产生一个空参数。

双引号内部由以下两条规则描述：

```ebnf
double_quoted_char = any - ( '"' | '\\' | LF )
                   | '\\' double_quote_escape ;
double_quote_escape = '\\' | '"' | '`' | '$' | '@'
                    | LF
                    | any - ( '\\' | '"' | '`' | '$' | '@' | LF ) ;
```

`double_quoted_char` 表示双引号中的一个“字符单元”，有两种情况：

- 不是 `"`、`\\`、`LF` 的普通字符，直接保留；双引号内的空格也属于这一类，不会切分参数；
- 以 `\\` 开头的转义单元，交给 `double_quote_escape` 判断后续字符。

`double_quote_escape` 将反斜杠后的字符分成三类：

| 类别 | 写法 | 含义 |
| --- | --- | --- |
| 白名单转义 | `\\\\`、`\\"`、``\\` ``、`\\$`、`\\@` | 反斜杠被消费，输出后面的字符 |
| 双引号内续行 | `\\` + `LF` | 消费反斜杠和换行，不把换行放入参数 |
| 其他字符 | `\\X` | 反斜杠和 `X` 都保留，例如 `\\U`、`\\h` |

单引号内容则完全按字面处理：

```ebnf
single_quoted = "'" { any - ( "'" | LF ) } "'" ;
```

## A.2 EBNF 记号说明

| 记号 | 含义 |
| --- | --- |
| `名字 = 规则 ;` | 产生式定义(`名字` 为非终结符) |
| `"x"` | 终结符(输入中真实出现的字符) |
| `\|` | 或——多个可选分支 |
| `{ X }` | X 重复零次或多次 |
| `[ X ]` | 可选:X 出现零次或一次 |
| `any - X` | 差集:除 X 以外的任意字符(全集 = UTF-8 校验后的字符流) |
| `(* *)` | 注释 |

## A.3 逐产生式说明（阅读辅助）

| 产生式 | 解释 |
| --- | --- |
| `rsp_file` | 文件 = 一串 LF 结尾的行 + 至多一个无 LF 的末行；空文件合法。`final_line` 必须至少消费一个字符，不能匹配空串，因此不会产生歧义 |
| `line_with_lf` | 一行 = 行首空白 +（注释或参数行）+ LF；空行/纯空白行合法 |
| `final_line` | 无 LF 的末行；可以是注释、参数行或至少一个尾随空白 |
| `line_comment` | `#` 起始，吞到行尾/EOF；注释内引号和反斜杠均为普通字符 |
| `argument_line` | 参数列表 + 可选的“空白+行尾注释” + 尾随空白 |
| `whitespace_run` | 一段连续空白 |
| `whitespace_char` | 空格、Tab、VT、FF、CR；LF 单独作为行终止符 |
| `argument` | 一个完整参数：首字符不能是裸 `#`，后面可拼接无引号字符或引号片段 |
| `unquoted_start_char` | 参数首个无引号字符；排除空白、换行、引号和 `#` |
| `unquoted_char` | 无引号字符；除空白、换行和两种引号外均按字面处理 |
| `quoted_segment` | 一个双引号或单引号片段 |
| `double_quoted` | 双引号片段；引号本身不进入参数 |
| `double_quoted_char` | 双引号内非转义普通字符 |
| `double_quote_escape` | 双引号内反斜杠转义、续行以及未列入白名单的 `\X` |
| `single_quoted` | 单引号片段；除单引号和 LF 外全部按字面处理 |

### A.4 设计要点（阅读辅助）

这套文法按“文件 → 行 → 参数 → 片段”分层，结构边界与字符转义分开表达：

1. `rsp_file`/`line_with_lf` 负责文件和行边界；`final_line` 专门接收没有结尾 `LF` 的最后一行，并且禁止匹配空串，避免空文件或末尾换行出现两种语法树。
2. `argument_line` 用 `whitespace_run` 明确表示“至少一个空白”。因此 `a#b` 中的 `#` 是参数内容，而 `a #b` 中的 `#` 位于参数起始位置，会进入注释分支；这对应“`#` 仅在参数起始处注释”。
3. `argument` 只保留一个必要区别：首字符用 `unquoted_start_char` 排除裸 `#`，后续统一使用 `unquoted_char | quoted_segment`，因此仍支持 `pre"mid dle"post` 引号拼接，但减少了中间非终结符。
4. `double_quoted`、`single_quoted` 分别对应双引号和单引号；`quoted_segment` 只是二者的组合名称，不增加额外语义。


<!-- 详细证明和状态机记录已移出本附录。
1. 无引号上下文:`unquoted_part` 的补集恰好是空白/LF/引号(首位置再排除 `#`)→ 分别落入分隔、行结束、引号、注释产生式;`argument_start` 禁 `#` 使注释分支与参数分支 first 集互斥,无遗漏亦无重叠;
2. 双引号上下文:`double_quote_escape` 三分支互斥且穷尽反斜杠后的所有字符,非反斜杠/双引号/LF 字符由 `double_quoted_char` 普通分支覆盖;裸 LF 由错误处理，反斜杠 + LF 则由续接分支消费;
3. 单引号上下文:同理;
4. LF 与 EOF 在全部上下文(无引号/双引号/单引号/注释/转义/文件尾)均有显式出口(行结束/报错/接受;末行无 LF 由 `final_line` 显式接受)。

故任意输入的每一步都有唯一确定的后继,直到接受或报错。∎

**定理 2(表达充分性,往返)**:对意图域内任意参数序列 σ,存在规范编码 W(σ) 使 parse(W(σ)) = σ。

W 的构造(逆运算,逐字符,与 A.9.1 定义 5 一致):不含空白/`"` 且非空、且不以 `#`/`@` 起始的参数原样输出;否则包双引号——内部 `"` → `\"`、`\` 一律翻倍(保守编码)、首字符 `@` 写作 `\@`(`#` 在双引号内本就字面,无需转义)。证明对 σ 的字符序列归纳,依据 `double_quote_escape` 分支穷尽性;多参数序列 W(σ₁) LF … LF W(σₖ) 无尾 LF,末段由 `final_line` 接受。∎

-->

# **附录 B:竞品调研与现代 rsp 设计原则**

> **结论先行**:不存在单一"GNU 风格"。解析器按"版本 × 宿主"分四类行为,但存在一个所有解析器共同遵守的**安全公共子集**(§3)——项目读写 rsp 按它来即可。

## B.1 各解析器规则总表

| 规则 | GCC 11.4¹ | clang 14(全平台) | GNU ld 2.38 | ld.lld/ld64.lld(Linux/macOS 宿主) | ld.lld(MinGW 驱动 × Windows 宿主)² | bash(仅参照) |
| --- | --- | --- | --- | --- | --- | --- |
| 空白切 token | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `"` 包空格、可跨行 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `'` 引号 | ✓(内 `\` 仍转义) | 同左 | 同左 | 同左 | ✗(`'` 是普通字符) | ✓(内全字面) |
| `\X` → 字面 `X` | 引号内外都转 | 引号内外都转 | 引号内外都转 | 引号内外都转 | ✗(`\` 字面,仅紧邻 `"` 时特殊) | 双引号内仅 `` $ ` " \ `` 5 种 |
| `\<newline>` | 换行进 token | 换行进 token | 换行进 token | 换行进 token | 字面 | 续行拼接 |
| `#` 注释 | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |
| `$` / `~` / glob 展开 | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |
| 空串 `""` | 保留 | **丢弃** | 保留 | 保留 | 未测 | 保留 |
| 未闭合 `"` | 吞到 EOF | 吞到 EOF | 吞到 EOF | 吞到 EOF | 吞到 EOF | 语法错误 |

¹ GCC 在向 bash 演进:WinLibs GCC 16.1.0 初测已部分吻合新 libiberty(双引号内仅 5 种转义、单引号字面化),与 GCC 11 已分叉。
² lld 的 tokenizer **跟着驱动走**:MachO 驱动(ld64.lld)无条件 GNU 规则(`lld/MachO/DriverUtils.cpp:86`),故 macOS 上与 clang 同款;GNU/ELF 驱动与 MinGW 驱动按宿主 OS 分支(Windows→CRT,其他→GNU),ELF 驱动另可用 `--rsp-quoting=posix|windows` 显式指定(`lld/ELF/DriverUtils.cpp:77-89`)。**Windows 宿主上的 ld.lld(MinGW)是全家唯一 CRT 规则者**。GNU ld 与 gcc 同源 libiberty(`expandargv`/`buildargv`)。

**未闭合 `"`(EOF)**:开了 `"` 却一直等不到收尾的 `"`,此时 GCC/clang/GNU ld/lld(含 Windows CRT)谁都不报错,而是把**从开引号到文件末尾的全部内容**当作同一个参数,然后正常结束。例如 `"test.c` → 参数 `test.c`(后面恰好没别的内容,所以看不出问题;若后面还有几十行参数,会被全部并进这一个参数);只有 bash 视为语法错误。方案一/二有意加严报 `unterminated_quote`(见附录 A0.6)。

**常见误区**:clang 的单引号**不是** bash 单引号——引号内外 `\` 一律转义(`'c:\users'` → `c:users`),引号只负责包空格与闭合。CRT(ld.lld-on-Windows)更极端:`'` 根本不是引号——两个 `'` 之间没有字面保护区,`"` 照样开合、空白照样切分(详见附录 A0.3 的 `'` 与 `''` 规则)。

## B.2 实验例表（写法 → 解析结果）

### B.2.1 Linux/WSL 宿主(clang 14 / gcc 11.4 / GNU ld 2.38 / bash 5)

| rsp 写法 | clang 14 | gcc 11.4 | GNU ld 2.38 | bash 5 |
| --- | --- | --- | --- | --- |
| `a  b` / TAB / 首尾空白 | 切 token 去空白 | 同 | 同 | 同 |
| `"a b"` | `a b` | 同 | 同 | 同 |
| `a\ b` | `a b` | 同 | 同 | 同 |
| `"a\\b"` | `a\b` | 同 | 同 | 同 |
| `"a\"b"` | `a"b` | 同 | 同 | 同 |
| `a"b c"d'e f'` | 一个 token `ab cde f` | 同 | 同 | 同 |
| `a\nb`(未引号 `\`+字母) | `anb` | 同 | 同 | `anb` |
| `"a\$b"` | `a$b`(碰巧一致,机制不同) | 同 | 同 | `a$b` |
| `a#b`(# 词中) | 字面 | 同 | 同 | 字面 |
| `"a<换行>b"` | 一个含真实换行的 token(clang 到宏层才警告截断) | 同 | 同 | 同 |
| 空行/缩进行 | 忽略 | 同 | 同 | 同 |
| `a\`(EOF 孤立反斜杠) | 保留 `a\` 不报错 | 同 | 同 | 同 |
| `'c:\users\liso'` | `c:usersliso`(**单引号内仍转义**) | 同 | 同 | `c:\users\liso`(全字面) |
| `'a\"b'` | `a"b` | 同 | 同 | `a\"b` |
| `a\`↵`b` 行尾反斜杠 | `a<换行>b`(换行进 token) | 同 | 同 | `ab`(续行拼接) |
| `# comment`(行首) | `#`、`comment` 成输入文件 | 同(链接期才报 `cannot find #`,编译期无征兆) | 同 | 注释,0 token |
| `$HOME` / `$(echo hi)` / `*.rsp` / `~/x` | 全部字面,当输入文件报错 | 同 | 同 | 变量/命令/glob/波浪号展开 |
| `""` / `''` 空串 | **丢弃**(cc1 argv 无空元素实证) | **保留**(成空名输入文件) | **保留**(`cannot find : ` 空名报错) | 保留 |
| `"abc` 未闭合引号 | 吞到 EOF(含换行)成一个 token,rsp 层不报错 | 同 | 同 | 语法错误 |
| `a\r\nb` CRLF | `\r` 当空白切分(反而兼容 Windows 格式) | 同 | 同 | `\r` 留在 token 里 |
| `'-De=a"b c"'` | 完整 token | 完整(早轮"裂 3 段"系展示层假象,argv 级复测不复现) | 完整(`unrecognized option` 反证) | 完整 |
| `"a'b'c"` / `'a"b"'` 异类引号嵌套 | 完整,不裂 | 同 | 同 | 完整 |

@ 机制(Linux 三家):嵌套展开、`@'带 空格.rsp'`、`"@x.rsp"`、同文件多次引用重复展开均支持;自引用不挂死(gcc/ld 报 `too many @-files encountered`,clang 达上限后 `@token` 原样透传,与"文件不存在"同文案);`@` 指向不存在文件三家都硬错(gcc 伴生误导性警告 `linker input file unused`);`@ file`(@ 后空格)→ `@` 单独成 token 报错,含空格路径必须 `@'...'`。

### B.2.2 Windows 宿主(llvm-mingw 15:clang / gcc 16.1.0 / ld.lld)

| rsp 写法 | clang | gcc 16.1.0 | ld.lld |
| --- | --- | --- | --- |
| `"C:\\opt\\rsp wintest\\main.c"` | `C:\opt\rsp wintest\main.c` | `C:\opt\rsp wintest\main.c` | `C:\\opt\\rsp wintest\\main.c`(**原样保留**) |
| `"C:\opt\buildtools\rsp_wintest\main2.c"` | `C:optbuildtoolsrsp_wintestmain2.c` | 原样保留 ⚠ | 原样保留 |
| `C:\opt\buildtools\rsp_wintest\main.c`(裸) | `C:optbuildtoolsrsp_wintestmain.c` | `C:optbuildtoolsrsp_wintestmain.c` | 原样保留 |
| `"C:\opt\'\rsp_wintest\main.c"` | `C:opt'rsp_wintestmain.c` | `C:\opt\'rsp_wintest\main.c` ⚠ | 原样保留(`'` 字面) |
| `"C:\opt\"\rsp_wintest\main.c"` | `C:opt"rsp_wintestmain.c`(`\"`→`"` 且不闭合) | `C:opt"rsp_wintestmain.c` ⚠ | `C:\opt\rsp_wintest\main2.c` |
| `"C:\opt\...\main.c  \`↵`"`(引号跨行) | token 含换行 | token 含换行 | `\` 与换行都进 token |
| `C:\\opt\\buildtools\\rsp_wintest\\main2.c`(未引号) | `C:\opt\buildtools\rsp_wintest\main2.c` | 同 clang | 原样保留,且该路径被**成功打开**(实证 Win32 容忍重复分隔符) |
| 上一行 + `@build.rsp`(不存在) | `no such file or directory: '@build.rsp'` | `too many @-files encountered` | `@build.rsp: No such file or directory` |
| `@test.exe`(rsp 指向二进制) | 三家均把 PE 头当文本词法解析:`MZ`、`This program cannot be run in DOS mode`、随机字节全成 token(gcc 到链接期才报,clang/lld 逐 token 报) | 同 | 同 |
| '-De=a"b c"' | 见上 | 加上 | -De=ab c |
| 'a\\"b' | 见上 | 加上 | a"b |
| "abc  | 见上 | 加上 | abc|
| "a'b'c" | 见上 | 加上 | a'b'c|

> 注:表中 `'...'` 是测试脚本的书写记号,**不等于文件字节**——外层 `'` 经 `echo` 写入时会被 shell 剥离。例:本表 `'-De=a"b c"'` 行,由 ld.lld(CRT)结果 `-De=ab c` 可反推文件实际字节为 `-De=a"b c"`(无 `'`);若 `'` 真在文件中,CRT 结果应为 `'-De=ab c'`(`'` 原样保留,见附录 A0.3 的 `'` 与 `''` 规则)。本表末四行为占位残文,需按真实字节复测后订正。




## B.3 现代语言工具链的 rsp 约定与演进

现代语言工具链通常把 rsp 当作“参数载体”，而不是 shell 脚本。需要区分语言驱动器是否公开 `@file` 入口，以及驱动器为后端工具内部生成的临时 rsp；后者不能直接推导出用户语法。更重要的是，格式往往会随命令行限制和生态兼容需求逐步演进，不能只看某个版本的当前行为。

| 工具链 | 用户侧入口 | 已观察/公开的格式倾向 | 证据/对本设计的启示 |
| --- | --- | --- | --- |
| Rust `rustc` | `rustc @args.rsp` | 默认按**每行一个参数**读取，行内容原样保留，不执行 shell 引号/反斜杠解析；官方源码另有不稳定的 `-Zshell-argfiles`，开启后才使用 shell 风格解析；默认不递归展开 rsp 内的 `@child.rsp` | **rustc 定义格式，Cargo 负责生成和重试**：Cargo 遇到命令行过长时生成临时 `cargo-argfile.*`。方案四借鉴逐行模型，但其“去首尾空白”与 Rust 不同。[rustc 源码](https://github.com/rust-lang/rust/blob/master/compiler/rustc_driver_impl/src/args.rs)、[Cargo 源码](https://github.com/rust-lang/cargo/blob/master/crates/cargo-util/src/process_builder.rs) |
| Swift `swiftc`/Swift driver | Swift driver 支持用户参数中的 `@file`，也会按工具能力和命令行长度**自动/强制生成临时 rsp** | **官方源码**：[`Driver.swift`](https://github.com/swiftlang/swift-driver/blob/main/Sources/SwiftDriver/Driver/Driver.swift) 负责递归展开；POSIX 行解析支持反斜杠、shell 引号、空白分隔和 `//` 行首注释；Windows 使用 CRT 风格；[`ArgsResolver.swift`](https://github.com/swiftlang/swift-driver/blob/main/Sources/SwiftDriver/Execution/ArgsResolver.swift) 按每个已序列化参数一行写出，并按目标工具选择 quoting |
| Go `go` 命令 | `go build` 没有稳定承诺的通用用户侧 `@file` 语法；但 Go 工具链内部的 compile/link 参数文件经历过多次演进 | **Go issue/CL 演进记录**。不能把工具内部参数文件当作用户格式标准，但其迁移过程可作为兼容性设计先例 |

Rust 官方实现的关键语义是“`contents.lines()` 的每一行直接成为一个参数”；默认模式不会把引号当语法，也不会消费反斜杠。Rust 还保留了显式的 `-Zshell-argfiles` 能力，说明**基础格式和增强格式应通过显式开关区分**，不应自动猜测。

Swift 官方实现提供了另一条重要经验：驱动器会根据目标工具是否支持 rsp、命令行是否超限来决定是否生成临时 rsp；生成器把参数逐项序列化，Windows 与 POSIX 使用不同 quoting 规则，并通过 `@path` 传入。这说明“用户读取 rsp”和“工具内部生成 rsp”可以共享容器机制，但词法规则必须由接收工具决定。

### B.3.1 Rust：Cargo 负责兜底，rustc 负责读取

Rust 的职责边界很清晰：`rustc` 只定义和读取 argfile；Cargo 的 `ProcessBuilder` 负责检测 `E2BIG`/Windows 命令行过长错误，创建临时 `cargo-argfile.*`，把每个参数逐行写入，再以 `@file` 重试，完成后删除临时文件。Rust 默认格式要求参数为 UTF-8 且不能包含换行；引号和反斜杠不具备 shell 语义。Rust 还提供不稳定的 `-Zshell-argfiles` 显式切换到 shell 风格解析。

Rust 本身也有清晰的协议演进：

1. **初始实现（2019，Issue [#63576](https://github.com/rust-lang/rust/issues/63576)）**：增加 `@path` 参数文件，采用最小的“一行一个参数”格式；空行表示空参数；支持 Unix/Windows 换行；文件必须是 UTF-8。该设计明确承认不能表达参数内部换行。
2. **稳定化（2019，Commit [446a43d](https://github.com/rust-lang/rust/commit/446a43da0ee23b3a69e214f030d6d7dada345296)）**：将 `@file` 入口稳定下来，但没有扩展基础格式，保持逐行、无引号、无转义的机器协议。
3. **显式增强（2023，Commit [684aa2c](https://github.com/rust-lang/rust/commit/684aa2c9d11d9e818ea65f7e4ebd194b97318ac7)）**：新增不稳定 `-Zshell-argfiles`，仅在显式开启时使用 shell 风格解析；默认格式保持兼容。这是“新增语义使用开关隔离，而不是改变旧格式”的典型做法。
4. **构建侧兜底**：Cargo 在执行 rustc/rustdoc 时负责生成 argfile 和失败重试；`rustc` 不负责为“启动自己”生成 argfile。这验证了“参数数组持有者负责写端”的生成者原则。

这条演进路线体现了“基础格式最小化、增强语义显式开启”：机器生成场景不需要为了手写便利引入复杂 tokenizer；确需 shell 兼容时，通过显式开关而不是猜测文件内容来启用。

### B.3.2 Swift：从支持 response file 到按工具能力自动选择

Swift Driver 的演进重点不在单一词法，而在“按下游工具能力选择协议”：

1. 早期先增加 response-file 展开和参数 tokenization（[初始支持](https://github.com/swiftlang/swift-driver/commit/b86325a9)）；
2. 随后支持相对路径、递归引用检测、临时文件原子写入和确定性命名（[相对路径](https://github.com/swiftlang/swift-driver/commit/df0e9e19)、[原子写入](https://github.com/swiftlang/swift-driver/commit/6f5e8fed)）；
3. Windows 场景增加 CRT 风格 tokenizer，并对 `clang.exe` 显式传递 `--rsp-quoting=windows`（[Windows response file](https://github.com/swiftlang/swift-driver/commit/402b22a9)、[Windows quoting](https://github.com/swiftlang/swift-driver/commit/1bd69775)）；POSIX 场景使用另一套 quoting；
4. 当前 Driver 根据命令行长度和目标工具是否支持 rsp 自动/强制生成临时文件，同时对源文件列表使用独立的 `-filelist` 机制。

因此 Swift 的经验不是“所有 rsp 都一行一参”，而是：**rsp 的写端必须面向具体消费者选择 quoting；专用 filelist 与通用 response file 分离；自动生成和用户手写是两条不同链路。**

### B.3.3 Go：从私有逐行格式逐步走向 GCC 兼容

Go 的 response-file 演进可以概括为三阶段（版本和 CL 以官方 issue/review 记录为准）：

1. **Windows 初始支持（[Issue #18468](https://github.com/golang/go/issues/18468)，[CL 110395](https://go-review.googlesource.com/c/go/+/110395)）**：为解决约 32KB 命令行限制，`cmd/go`/`cmd/compile` 使用临时 response file；格式为每行一个参数，按 `\\n` 切分并去掉 `\\r`，不支持引号和转义。
2. **转义协议补丁（[Issue #42295](https://github.com/golang/go/issues/42295)，[CL 272870](https://go-review.googlesource.com/c/go/+/272870)，Go 1.17）**：为保留参数中的反斜杠和换行，写端 `encodeArg`、读端 `DecodeArg` 增加专用编码；未知 `\\x` 序列视为非法输入。这是对第一版逐行格式的兼容性补强。
3. **GCC 兼容重写（[Issue #77177](https://github.com/golang/go/issues/77177)，[CL 737500](https://go-review.googlesource.com/c/go/+/737500)，Go 1.27）**：为兼容 Bazel/Buck 等构建系统生成的标准 rsp，并修复 Windows 裸路径导致的解析问题，改用接近 GCC `libiberty/argv.c` 的 tokenizer，支持空白分隔、引号、转义和续行；旧版 `\\n` 转义格式因此存在 breaking change，需要迁移。

Go 的教训是：私有格式可以快速解决平台限制，但一旦需要接入外部构建生态，就会被迫向事实标准迁移。因此首版应明确兼容目标，并为未来格式升级预留版本/入口，不能把临时内部格式直接承诺为长期公共协议。

CMake/Ninja 则体现了“配置者与执行者分离”：CMake根据编译器和平台生成包含 `rspfile`/`rspfile_content` 的 Ninja 规则，实际构建时由 Ninja 写入 rsp、执行 `tool @file`，成功后删除。换用 Makefile generator 时，rsp 由 CMake生成的 Makefile/辅助命令负责。因此可以说“CMake支持生成 rsp”，但更准确的表述是：**CMake负责决定和描述 rsp 的用法，底层构建执行器负责在构建时落盘**。

## B.4 设计选择与语法后果

| 设计选择 | 必然带来的语法/实现要求 | 设计理念 |
| --- | --- | --- |
| 支持一行多个参数 | 以 `isspace()`（至少覆盖空格、Tab、LF、CR，并明确是否含 VT/FF）识别分隔符；因此参数内部需要空格时必须有引号或转义机制 | 优先兼容 GCC/Clang/MSVC 和构建工具生成物，降低迁移成本 |
| 支持含空格、引号、反斜杠的单个参数 | 引入引号分组、引号拼接和反斜杠转义；必须定义 `\`、`\"`、孤立 `\` 及引号未闭合的行为 | 把文件文本稳定映射为一个 argv 元素，避免路径和宏值静默变形 |
| Windows 裸路径可直接书写 | 无引号上下文不能沿用 bash 的 `\X` 吃字符规则；需要将无引号 `\` 字面化，或采用独立 CRT 词法 | 首要平台痛点优先，但不能破坏引号内的可逆转义 |
| 支持空参数 | 保留 `""`/`''` 或定义等价的显式编码；空行是否为空参数必须单独规定，不能靠实现习惯 | 空值是参数序列的一部分，不能与“没有参数”混淆 |
| 支持注释 | 规定 `#` 只在 token 起始生效，或完全不支持；一旦支持就必须提供表达字面 `#` 的方法 | 可读性和可复制性要服从参数确定性，避免把 GCC 文件误当脚本 |
| 支持嵌套 `@file` | 规定相对路径基准、循环检测、最大深度、字面 `@` 的编码和错误码 | 组合能力必须可控，不能让递归或 cwd 造成隐式行为 |
| 支持 CRLF/跨平台共享 | 读取时明确 CRLF 归一化、编码和 BOM；写出时统一 UTF-8/LF | 同一份配置在不同宿主上应得到同一参数序列 |
| 允许语法演进 | 使用版本标记、新扩展名或命令行开关；旧版本遇到新版本应明确拒绝 | 宁可显式失败，也不静默改变既有参数含义 |

## B.5 rsp 格式演进与兼容原则

1. **参数序列优先**：rsp 的唯一职责是确定地生成 `argv`，不引入变量、命令替换、glob 或环境依赖。
2. **生成与读取对称**：生成器必须能对任意合法参数进行可逆序列化，读取器不得依赖调用 shell 的二次转义。
3. **分隔模型稳定**：一旦选择空白分隔或一行一参，后续版本不得默默切换；空行、纯空白行、行首尾空格的含义必须固定。
4. **能力按版本增加**：新增引号、转义、注释、嵌套等能力时，使用显式版本标记或新入口，不靠猜测格式启用。
5. **旧版本可识别失败**：遇到不支持的版本或非法语法，在 rsp 展开阶段报告文件路径、行号和错误类型。
6. **嵌套语义显式化**：支持嵌套就完整定义路径基准、循环和深度；不支持就按普通参数处理，不能出现半展开状态。
7. **规范文件可移植**：推荐 UTF-8、LF 和稳定路径表示；宿主 shell、cwd、本地编码不能成为隐含输入。
8. **兼容优先于“看起来简单”**：解析器简单不是唯一目标；应同时衡量生态兼容、可逆序列化、错误可诊断性和长期演进成本。

## B.6 对本方案的归纳

- 方案一选择空白分隔，是为了兼容“单行多个参数”的业界模型，因此必须定义 `isspace()` 分隔集合，并提供引号/转义来表达空格、引号和反斜杠。
- 方案四选择一行一参，借鉴 Rust 的逐行模型，换取更简单的解析器；但它不是 Rust 的完全复制，也不应宣称兼容 GCC/Clang/MSVC 的通用 rsp。
- 无论采用哪一方案，文件编码、换行、空参数、`@` 展开、错误诊断和版本演进都必须先于实现确定。

# **附录 c:评审范围**

**架构会议需要上会评审范围:**
1. SR级别及以上的技术架构或解决方案
2. 影响架构流程(影响编译或者执行pipeline)的解决方案
3. 涉及多种场景,涉及多个后端,多个OS的技术方案,有性能影响(时间或空间)的关键方案
4. 标准库的所有API设计及实现

**功能实现评审会议评审范围:**

1. 大粒度的AR特性实现
2. 对周边有依赖或者有交互的AR
3. 有多种实现方式供选择,需要决策更优实现方式的AR

4. 标准库的所有实现

**不需要上会评审范围:**
1. 除上会评审外的简单的可以自闭环(不影响/不依赖周边特性,不影响外部开发者)的AR(这种AR需要组内评审)。

> 本特性涉及多平台(Windows/Linux/macOS)、多链接器(`ld.lld`/`ld64.lld`/GNU `ld`)、影响编译 pipeline 的链接阶段,且有多种实现方案需决策(见 §4.3),建议上**架构会议**评审。
