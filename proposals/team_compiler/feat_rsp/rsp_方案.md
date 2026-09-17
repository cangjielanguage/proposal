# Response File 特性设计

# **1、特性需求/问题/动机的来源与价值**

## 1.1 背景与目标

之前一线同事反馈当cjpm编译大型项目（包结构复杂，嵌套层级过多）会出现编译失败的问题：

``` cmd
C:\rsp_e2e\ws_multigkg_demo>cjpm build --jobs 8
error: ""C:\\hlt\\cangjie\\third_party\\llvm\\bin\\ld.lld.exe" "-LC:\\hlt\\cangjie\\third_party\\mingw\\lib\\" "-LC:\\rsp_e2e\\ws_multigkg_demo\\target\\release\\pkg_dep_module_0355\\"

...

le_0061" "-lpkg_dep_module_0060" "-lpkg_dep_module_0059" "-lpkg_dep_module_0030" "-lpkg_dep_module_0029" "-lpkg_dep_module_0028" "-lpkg_dep_module_0027" "-lpkg_dep_module_0002" "-lpkg_dep_module_0001" "--start-group" "-l:libcangjie-std-core.a" "--end-group" "-l:libcangjie-runtime.dll" "-lclang_rt-builtins" "-lboundscheck" "-lm" "-lmsvcrt" "-lmingw32" "-lmoldname" "-lmingwex" "-lmingw32" "-lmsvcrt" "-lpthread" "-ladvapi32" "-lshell32" "-luser32" "-lkernel32" "-lmingw32" "-lmoldname" "-lmingwex" "-lmsvcrt" "C:\\hlt\\cangjie\\third_party\\mingw\\lib\\crtend.o" ": command failed (The parameter is incorrect.)
Error: failed to compile package `app`, return code is 1
Error: cjpm build failed
```

**根因**:Windows `CreateProcessA` 命令行字符串长度上限为 32767 字符(https://learn.microsoft.com/zh-cn/windows/win32/api/processthreadsapi/nf-processthreadsapi-createprocessa)，（cmd 命令行中为 8191）一旦 `cjc` 收到的参数总长超过该值,操作系统直接拒绝启动进程。Linux/macOS 虽有 `ARG_MAX`(典型 2 MB)限制,常规工程不易触发,但 Windows 大工程(尤其是链接阶段参数多）容易遇到该限制。

### 1.1.1 目标


为绕开上述系统命令行长度限制,本特性为 `cjc` 提供一种"从文件读取命令行参数"的能力。出现该问题一定是因为涉及编译大型项目，输入了太多的自定义的内容，例如链接的路径、链接所需要的库（包括静态库、动态库、bc文件），涉及太多的中间产物。因此，我们更加倾向将该文件看作是编译过程的中间产物，而非需要开发者过度关注。


**术语**:该文件即业界通称的 **response file(响应文件)**,常取扩展名 `.rsp`(故简称 rsp 文件)——一份纯文本参数清单,由工具自己读取并展开,以绕开 shell/OS 的命令行长度限制。
https://gcc.gnu.org/wiki/Response_Files 



## 1.2 适用场景
- Windows 命令行长度上限 32767 字符,进程可能启动失败;
- 大型工程中源文件、依赖库、链接输入或编译选项数量过多,由构建系统/包管理工具自动生成 rsp;

本特性只影响 driver 读取命令行选项的阶段:rsp 展开发生在选项解析之前,展开后行为与参数直写命令行完全一致,不改动编译、链接等后续 pipeline。





# **2、业界竞品分析**

## 2.1 业界产品支持情况
目前调研了下列工具，全部都支持response file机制
| 工具 | 用户手册 |
| --- | --- |
| GCC | <https://gcc.gnu.org/onlinedocs/gcc/Overall-Options.html> | libiberty `argv.c` 的 `expandargv` |
| GNU ld | <https://sourceware.org/binutils/docs/ld/Options.html> | binutils `ld/`(与 libiberty 同源) |
| clang / lld | 无专门手册页 | `llvm/lib/Support/CommandLine.cpp`(`ExpansionContext::expandResponseFiles`)+ `lld/Common/Args.cpp` |
| MSVC `link.exe` | <https://learn.microsoft.com/en-us/cpp/build/reference/at-specify-a-linker-response-file> | 闭源,以文档为准 |
| Rust `rustc` | [`@path` 参数文件](https://doc.rust-lang.org/rustc/command-line-arguments.html#path-load-command-line-flags-from-a-path) | [`rustc_driver_impl/src/args.rs`](https://github.com/rust-lang/rust/blob/master/compiler/rustc_driver_impl/src/args.rs)；稳定化 Issue [#63576](https://github.com/rust-lang/rust/issues/63576) / Commit [446a43d](https://github.com/rust-lang/rust/commit/446a43da0ee23b3a69e214f030d6d7dada345296) |
| Rust Cargo | 无独立用户格式文档 | [`cargo-util/process_builder.rs`](https://github.com/rust-lang/cargo/blob/master/crates/cargo-util/src/process_builder.rs) 的 `retry_with_argfile` |
| Swift Driver | [Swift Driver 仓库](https://github.com/swiftlang/swift-driver)；`swiftc` 用户手册未单独定义统一 rsp 语法 | [`Driver.swift`](https://github.com/swiftlang/swift-driver/blob/main/Sources/SwiftDriver/Driver/Driver.swift)、[`ArgsResolver.swift`](https://github.com/swiftlang/swift-driver/blob/main/Sources/SwiftDriver/Execution/ArgsResolver.swift)；演进提交见 [response-file 初始支持](https://github.com/swiftlang/swift-driver/commit/b86325a9)、[Windows 支持](https://github.com/swiftlang/swift-driver/commit/402b22a9) |
| Go  | 无稳定用户文档；主要通过 Issue/CL 演进 | 初始支持 [Issue #18468](https://github.com/golang/go/issues/18468)/[CL 110395](https://go-review.googlesource.com/c/go/+/110395)；转义补丁 [Issue #42295](https://github.com/golang/go/issues/42295)/[CL 272870](https://go-review.googlesource.com/c/go/+/272870)；GCC 兼容 [Issue #77177](https://github.com/golang/go/issues/77177)/[CL 737500](https://go-review.googlesource.com/c/go/+/737500) |
| CMake / Ninja | [Ninja `rspfile` 文档](https://ninja-build.org/manual.html)；CMake 平台模块中的 `CMAKE_<LANG>_USE_RESPONSE_FILE_FOR_*` 配置 | CMake 生成规则，Ninja 在执行前写入并在成功后删除 `rspfile` |
具体字符行为及实验数据见附录 B。

## 2.2 rsp文件生成时机

目前业界工具在构建，编译过程中生成Rsp文件，开发者并不需要过多关注。这也启示我们无需为rsp设计太多的能力。

| 语言/工具 | rsp 生成者 | rsp 消费者 |
| --- | --- | --- |
| rust | Cargo | rustc/rustdoc |
| Swift | swift-driver（用于编译相关） | swiftc以及下游工具| 待补充 |
| Go  | `go` 构建驱动或相应工具链组件 | go 编译器 |
| CMake + Ninja | CMake 生成构建规则；**执行时由 Ninja 写入 rsp** | compiler/linker | CMake按编译器/平台能力产生 `rspfile`、`rspfile_content` 规则；Ninja 在执行命令前写文件，成功后删除 |



## 2.3 业界产品rsp文件主要规则

业界主流实现可归纳为以下几类。本文所称“GNU 格式”和“GCC 格式”通常指同一来源：GCC、GNU ld 等 binutils 工具通过 libiberty 的 `buildargv`/`expandargv` 解析 rsp，因此统一记为 **GNU/GCC 格式**。LLVM GNU 是另一套兼容实现，与 GNU/GCC 常用写法相通，但边界行为不完全一致。

下表中，“跨平台性”表示同一文件能否在 Windows/Linux/macOS 上保持相同参数语义；“兼容性”表示能否直接读取其他工具链按其规则生成的 rsp。

| 格式 | 主要使用者 | 核心规则 | 跨平台性 | 兼容性 | 设计动机与指导思想 |
| --- | --- | --- | --- | --- | --- |
| MSVC/CRT（Windows 规则） | `link.exe`、`cl.exe`、`lld-link`、`clang-cl` | 空白分隔；仅双引号分组；反斜杠只在紧邻 `"` 时使用 2n/2n+1 规则；不支持 shell 变量、glob、注释和续行 | **低**。主要服务 Windows | Windows 生态内兼容性高；与 GNU/GCC、Rust 格式不兼容 | Windows 生态兼容约束 |
| GNU/GCC（libiberty） | GCC、GNU ld、`ar` 等 binutils 工具；Go 1.27 开始向其兼容 | 空白分隔；单双引号分组；反斜杠转义；不执行变量、命令和 glob 展开；引号内转义和换行行为以具体 libiberty 版本为准 | **中** | GCC 生态内兼容性高；与 LLVM GNU 常用子集兼容，但不能保证空参数、续行、引号等边界完全一致；与 MSVC、Rust 不兼容 | 接近 shell，Unix 工具链标准 |
| LLVM GNU | 非 CL 模式的 Clang、lld；Linux/macOS Swift | 空白分隔；单双引号分组；反斜杠转义；不支持注释和变量/glob 展开；`""`/`''` 不产生空参数 | **中** | LLVM 生态内兼容性高；与 GNU/GCC 常用输入兼容；与 MSVC、Rust 不兼容 | LLVM 生态  |
| Swift分平台格式 | Swift | macOS/Linux 使用 LLVM GNU tokenizer；Windows 使用 LLVM Windows/CRT tokenizer | **低** | POSIX 端与 LLVM GNU 兼容，并与 GNU/GCC 常用输入互通；Windows 端与 MSVC/CRT 兼容 | 根据平台选择已有生态规则 |
| Rust 零语法 | `rustc` 默认 argfile | 一行一个参数；无引号、无转义、无嵌套；UTF-8；参数内换行不可表示 | **高**  | 最低| 规则简单，跨平台 |

三种现代工具链的演进重点不同：Rust 保持基础格式稳定，通过显式入口增加 POSIX shell 风格（并非严格 GNU/GCC 兼容）的格式；Go 从私有逐行协议转向 GNU/GCC 兼容格式；Swift 采用 macOS/Linux LLVM GNU、Windows CRT 的分平台策略，主要演进生成时机、能力声明和进程内/进程外执行边界。详细过程见附录 B.3。


## 2.4 业界产品主要演进过程
详细过程见附录 B.3

（1）rust 2019年支持rsp文件并合入main分支， 2023年解决ninja 输入的文件的不兼容问题，引入了 `-Zshell-argfiles`以支持 shell语法。该特性以实验特性开放。

（2）go 语言为解决 windows 32kb 限制开始支持，格式为一行一个参数，后续为为保留参数中的反斜杠和换行开始引入转义，于2026年为兼容 Bazel/Buck 等构建系统生成的标准 rsp，改用GCC方式。

（3）swift则较为稳定，mac/linux上使用 llvm Gnu方式，而windows 上使用MSVC/CRT方式。

# **3、本特性的设计/实现方案**

## 3.1 选项定位与设计定位

cjc 提供编译选项 `@<file>`:以 `@` 开头、后跟 rsp 文件路径构成一个命令行参数,如 `@build/options.rsp`,用于在编译时从文件中读取编译参数。cjc 读取 `<file>` 中的参数序列,**就地替换**该 `@<file>` 参数;替换后 cjc 的行为,与用户把这些参数手工写在命令行该位置**完全等价**。
> 但是这里会引入一个兼容性问题 ，就是之前可以编译的@file.cj 此时会被识别成参数文件，但是考虑到这种场景并不常见， 因此想沿用业界普遍用法以@标识rsp文件。


本特性首先解决“大型工程中无法用普通命令行启动 cjc”的问题。

```text
cjpm / Ninja / CMake
    └─ 根据完整 argv 生成 rsp
         └─ cjc 读取 rsp 并恢复 argv
              └─ 进入既有选项解析和编译流程
```


## 3.2 接口与展开流程

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

## 3.3 方案设计

参照第二章业界情况，类似于rust一行一参数的情况兼容性不好，可以参考go rust都处于演进过程 ，特别是go 已经取消了一行一参数的设计；而GCC 风格对于windows 的场景并不友好，因此目前倾向于使用 LLVM GNU的方式，即参考 swift实现或者基于swift实现一个仓颉特有跨平台。目前识别到cangjie rsp的主要生成者是cjpm 以及cmake。cjpm在windows中并不使用gcc风格。 cmake（就算是之后需要适配Bazel等构建工具）等构建工具都支持定义rsp的写法，例如cmake的 CMAKE_<LANG>_USE_RESPONSE_FILE_FOR_* 。

详细字符规则、示例和逐项比较见附录 A0。
### 3.3.1 分平台
参考Swift做法，使用 mac/linux 使用llvm GNU, window使用CRT规则。举个例子：

```text
rsp 内容                         macOS/Linux             Windows
-L"third party/lib"             -Lthird party/lib        -Lthird party/lib
C:\Users\test\main.cj           C:Userstestmain.cj       C:\Users\test\main.cj
'a b'                           a b（一个参数）           'a、b'（两个参数，单引号保留）
a\ b                            a b（一个参数）           a\、b（两个参数）
"C:\lib\\"                     C:lib\                   C:\lib\
# comment                       #、comment                #、comment
pre"a b"post                   prea bpost（一个参数）     prea bpost（一个参数）
```

### 3.3.2 跨平台
基于LLVM GNU 进行定制化，主要修改点将无引号 \ 改为字面字符，使 Windows 裸路径可直接书写。举个例子：

```text
rsp 内容                         解析结果
-L"third party/lib"             -Lthird party/lib
C:\Users\test\main.cj           C:\Users\test\main.cj
a\ b                            a\、b（两个参数）
"C:\lib\\"                     C:lib\
# comment                       #、comment（不是注释）
pre"a b "post                  prea b post（一个参数，末尾空格保留）
--output=\                      --output=\（无续行）
```

# **4、结论**

## 4.1 评审结论

(待评审后填写:通过 / 不通过 + 达成一致的主要观点。)

## 4.2 遗留问题


# **附录 A0:章节3的方案细节**
## A0.1 3.3.1 分平台 LLVM GNU

本方案mac , linux直接采用 LLVM 的 GNU tokenizer，不增加 cjc 私有字符规则。
| 元素 | 规则 |
| --- | --- |
| 编码与分隔 | UTF-8；空白分隔；CRLF 兼容 |
| 引号 | 单引号和双引号均可分组，引号本身不进入参数；支持引号片段拼接 |
| 反斜杠 | 按 LLVM `TokenizeGNUCommandLine` 处理；无引号 Windows 路径中的反斜杠可能被消费 |
| 展开 | 不执行变量、命令、波浪号或 glob 展开 |

常见写法：

```text
-O2 -g                         → 两个参数
-L"third party/lib"            → 一个参数 -Lthird party/lib
a\ b                          → 一个参数 a b
# comment                     → 两个普通参数 #、comment（不是注释）
```
windows 采取如下规则：

| 字符 / 序列 | 处理规则 | 示例 → 解析结果 |
| --- | --- | --- |
| 空白 | 引号外切分参数 | `-O2 -g` → 两个参数 |
| `"` | 引号开合，引号内空白不切分 | `-L"C:\Program Files\lib"` → `-LC:\Program Files\lib` |
| 2n 个 `\` + `"` | 输出 n 个 `\`，引号正常开合 | `"C:\lib\\"` → `C:\lib\` |
| 2n+1 个 `\` + `"` | 输出 n 个 `\` 和字面 `"` | `\"` → `"` |
| `\` + 非 `"` | 反斜杠字面保留 | `C:\Users\x` → 原样 |
| `'` | 普通字符，不参与分组 | `-L'x y'` → 两个参数 |
| `#`、`$`、glob | 普通字符，不执行注释或展开 | `#` → 普通参数 |


## A0.2： 3.3.2 跨平台 LLVM GNU + Windows 裸路径扩展

本方案以 LLVM `TokenizeGNUCommandLine` 为基线，仅修改**无引号反斜杠**的处理：LLVM GNU 中无引号 `\X` 会消费反斜杠并保留 `X`；本方案改为原样保留 `\X`。其他空白分隔、单双引号、引号拼接等规则保持 LLVM GNU 语义。

| 元素 | 规则 |
| --- | --- |
| 编码与换行 | UTF-8；忽略 UTF-8 BOM；CRLF 按空白处理；非法 UTF-8 或 NUL 报错 |
| 分隔 | 空格、Tab、LF、CR、FF、VT 等空白切分参数 |
| 引号 | `'...'`、`"..."` 均用于分组，引号本身不进入参数；允许与无引号片段拼接 |
| 反斜杠（引号内） | 沿用 LLVM GNU tokenizer 的转义规则 |
| 反斜杠（无引号） | **扩展点**：一律字面保留，因此 `C:\Users\x` 原样；`a\ b` 被切成 `a\` 和 `b` |


## A0.3 嵌套引用与路径解析基准例子

**`@child.rsp` 的相对路径按"父 rsp 文件所在目录"解析,而不是按 cwd**;只有命令行顶层的 `@<file>` 才按 cwd 解析。`@` 后写绝对路径(如 `@D:\cfg\common.rsp`、`@/home/x/common.rsp`)时忽略基准目录、直接按该路径打开——使用者 cd 到任何目录都不影响绝对路径引用。

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

| 引用 | 实际打开的文件 |
| --- | --- |
| `@build/debug.rsp` | `project/build/debug.rsp` |
| `@common.rsp` | `project/build/common.rsp` |
| `@sub/deps.rsp`  | `project/build/sub/deps.rsp` |

就地按序展开:

```
cjc @build/debug.rsp src/main.cj
→ cjc @common.rsp -g -O0 src/main.cj                            # 展开 debug.rsp
→ cjc -Lthird party/lib -lfoo @sub/deps.rsp -g -O0 src/main.cj  # 展开 common.rsp(双引号去掉,空格保留)
→ cjc -Lthird party/lib -lfoo -lbar -lbaz -g -O0 src/main.cj    # 展开 deps.rsp
```

最终等价于:`cjc -Lthird party/lib -lfoo -lbar -lbaz -g -O0 src/main.cj`

注:展开过程中嵌套的 `@common.rsp`、`@sub/deps.rsp` 虽然形式上出现在了参数序列里,解析基准仍以"父 rsp 所在目录"为准(见上表),不随展开位置变成 cwd。

# **附录 A1：方案一的辅助词法模型**

> **定位**：方案一的规范来源是附录 A0.1。本附录只描述“LLVM GNU tokenizer + 无引号反斜杠字面化”的实现结构，不另行定义一套 bash 语法。LLVM GNU 的边界行为以项目实际采用版本的 `llvm::cl::TokenizeGNUCommandLine` 为准。

## A1.1 分层模型

方案一分为三层：

1. 文件层：读取 UTF-8 文本并完成 BOM、CRLF、NUL 和 I/O 校验；
2. tokenizer 层：按 LLVM GNU 规则切分参数，仅覆盖无引号 `\` 的处理；
3. 展开层：对 tokenizer 输出的完整 token 判断是否触发 `@file` 展开。

因此，`@`、相对路径、循环检测和最大深度不属于字符词法；`#` 也不是注释起始符，而是普通字符。

## A1.2 简化辅助产生式

下面的 EBNF 只描述 token 的结构。引号内反斜杠如何消费后继字符、EOF 如何收束等操作语义见 A1.3。

```ebnf
rsp_file          = { separator | argument } ;
separator         = SPACE | TAB | LF | CR | VT | FF ;
argument          = argument_part { argument_part } ;
argument_part     = unquoted_run | unquoted_backslash
                  | single_quoted | double_quoted ;
unquoted_run      = unquoted_char { unquoted_char } ;
unquoted_char     = any - ( separator | "'" | '"' | '\\' ) ;
unquoted_backslash = '\\' ;
single_quoted     = "'" { quoted_unit("'") } "'" ;
double_quoted     = '"' { quoted_unit('"') } '"' ;
quoted_unit(Q)    = any - ( Q | '\\' )
                  | '\\' any ;
```

这里最关键的覆盖点是：

```ebnf
unquoted_backslash = '\\' ;
```

也就是在无引号状态下，反斜杠本身进入 token，不与后一个字符组成转义单元。除这一点外，单/双引号、片段拼接以及引号内反斜杠行为沿用 LLVM GNU tokenizer。

## A1.3 解析示例

```text
输入                         结果
-O2 -g                       两个参数：-O2、-g
-L"third party/lib"          一个参数：-Lthird party/lib
pre"mid dle"post            一个参数：premid dlepost
# comment                    两个参数：#、comment（不是注释）
C:\Users\test               一个参数，反斜杠原样保留
a\ b                         两个参数：a\、b
'a\b'                        一个参数；引号内按 LLVM GNU 规则处理反斜杠
```

与原 LLVM GNU tokenizer 的差异仅发生在无引号状态：

```text
输入       LLVM GNU          方案一
C:\x        C:x               C:\x
a\ b        a b（一个参数）   a\、b（两个参数）
\#flag      #flag             \#flag
```

# **附录 B:竞品调研与现代 rsp 设计原则**

> **结论先行**:不存在单一"GNU 风格"。解析器按"版本 × 宿主"分四类行为,但存在一个所有解析器共同遵守的**安全公共子集**(§3)——项目读写 rsp 按它来即可。

## B.1 各解析器规则总表

| 规则 | GCC 11.4 | clang 14 | GNU ld 2.38 | ld.lld/ld64.lld（Linux/macOS） | ld.lld（MinGW/Windows） | bash（参照） |
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

> 本表只汇总结论；版本、宿主和驱动模式造成的差异见 B.2。

## B.2 实验例表（写法 → 解析结果）

本节只保留能够区分解析规则的用例。空白分隔、普通参数等共同行为不再重复。

### B.2.1 Linux/WSL 宿主(clang 14 / gcc 11.4 / GNU ld 2.38 / bash 5)

| rsp 写法 | clang 14 | gcc 11.4 | GNU ld 2.38 | bash 5 |
| --- | --- | --- | --- | --- |
| `a\ b` | `a b` | 同 | 同 | 同 |
| `a"b c"d'e f'` | 一个 token `ab cde f` | 同 | 同 | 同 |
| `a\nb`(未引号 `\`+字母) | `anb` | 同 | 同 | `anb` |
| `"a<换行>b"` | 一个含真实换行的 token(clang 到宏层才警告截断) | 同 | 同 | 同 |
| `a\`(EOF 孤立反斜杠) | 保留 `a\` 不报错 | 同 | 同 | 同 |
| `'c:\users\liso'` | `c:usersliso`(**单引号内仍转义**) | 同 | 同 | `c:\users\liso`(全字面) |
| `'a\"b'` | `a"b` | 同 | 同 | `a\"b` |
| `a\`↵`b` 行尾反斜杠 | `a<换行>b`(换行进 token) | 同 | 同 | `ab`(续行拼接) |
| `# comment`(行首) | `#`、`comment` 成输入文件 | 同(链接期才报 `cannot find #`,编译期无征兆) | 同 | 注释,0 token |
| `$HOME` / `$(echo hi)` / `*.rsp` / `~/x` | 全部字面,当输入文件报错 | 同 | 同 | 变量/命令/glob/波浪号展开 |
| `""` / `''` 空串 | **丢弃**(cc1 argv 无空元素实证) | **保留**(成空名输入文件) | **保留**(`cannot find : ` 空名报错) | 保留 |
| `"abc` 未闭合引号 | 吞到 EOF(含换行)成一个 token,rsp 层不报错 | 同 | 同 | 语法错误 |
| `a\r\nb` CRLF | `\r` 当空白切分(反而兼容 Windows 格式) | 同 | 同 | `\r` 留在 token 里 |
| `"a'b'c"` / `'a"b"'` | 完整，不拆分 | 同 | 同 | 完整 |

补充说明：

- clang/GCC/GNU ld 的单引号不是 bash 单引号，单引号内的反斜杠仍会参与转义；
- 未闭合引号会一直读取到 EOF，不在 rsp 解析阶段报错；
- Linux 下 GCC、clang、GNU ld 均支持嵌套 `@file`；不存在的文件会报错，自引用由各工具的深度限制终止。

### B.2.2 Windows 宿主(llvm-mingw 15:clang / gcc 16.1.0 / ld.lld)

| rsp 写法 | clang | gcc 16.1.0 | ld.lld |
| --- | --- | --- | --- |
| `"C:\\opt\\rsp wintest\\main.c"` | `C:\opt\rsp wintest\main.c` | `C:\opt\rsp wintest\main.c` | `C:\\opt\\rsp wintest\\main.c`（原样保留） |
| `"C:\opt\buildtools\rsp_wintest\main2.c"` | `C:optbuildtoolsrsp_wintestmain2.c` | 原样保留 ⚠ | 原样保留 |
| `C:\opt\buildtools\rsp_wintest\main.c`(裸) | `C:optbuildtoolsrsp_wintestmain.c` | `C:optbuildtoolsrsp_wintestmain.c` | 原样保留 |
| `"C:\opt\'\rsp_wintest\main.c"` | `C:opt'rsp_wintestmain.c` | `C:\opt\'rsp_wintest\main.c` ⚠ | 原样保留(`'` 字面) |
| `"C:\opt\"\rsp_wintest\main.c"` | `C:opt"rsp_wintestmain.c`(`\"`→`"` 且不闭合) | `C:opt"rsp_wintestmain.c` ⚠ | `C:\opt\rsp_wintest\main2.c` |
| `"C:\opt\...\main.c  \`↵`"`(引号跨行) | token 含换行 | token 含换行 | `\` 与换行都进 token |
| `C:\\opt\\buildtools\\rsp_wintest\\main2.c`(未引号) | `C:\opt\buildtools\rsp_wintest\main2.c` | 同 clang | 原样保留；Win32 可接受重复分隔符 |
| 上一行 + `@build.rsp`(不存在) | `no such file or directory: '@build.rsp'` | `too many @-files encountered` | `@build.rsp: No such file or directory` |
| `@test.exe`(rsp 指向二进制) | 三家均把 PE 头当文本词法解析:`MZ`、`This program cannot be run in DOS mode`、随机字节全成 token(gcc 到链接期才报,clang/lld 逐 token 报) | 同 | 同 |

补充说明：Windows 下的 ld.lld（MinGW 驱动）使用 CRT 规则；clang 和测试版本的 GCC 使用 GNU 类规则。lld 的具体选择还与驱动模式有关，不能只按工具名判断。




## B.3 Rust、Swift、Go 的 rsp 演进

### B.3.1 Rust

- 2019 年，[Issue #63576](https://github.com/rust-lang/rust/issues/63576) 引入 `@path`：每行一个参数，空行表示空参数，支持 LF/CRLF，文件必须是 UTF-8；参数中不能包含换行。
- 同年，[Commit 446a43d](https://github.com/rust-lang/rust/commit/446a43da0ee23b3a69e214f030d6d7dada345296) 将该功能稳定下来，原有格式未改变。
- 2023 年，[Commit 684aa2c](https://github.com/rust-lang/rust/commit/684aa2c9d11d9e818ea65f7e4ebd194b97318ac7) 增加实验性的 `-Zshell-argfiles`。启用后，`@shell:path` 按 POSIX shell 风格读取；普通 `@path` 仍按每行一个参数读取。
- Cargo 先用普通命令行启动 rustc/rustdoc；遇到命令行过长时，生成临时 `cargo-argfile.*` 后重试，执行结束后删除。实现见 [rustc args.rs](https://github.com/rust-lang/rust/blob/master/compiler/rustc_driver_impl/src/args.rs) 和 [Cargo process_builder.rs](https://github.com/rust-lang/cargo/blob/master/crates/cargo-util/src/process_builder.rs)。

Rust 没有修改已经稳定的默认格式，而是另加 `@shell:path`。

### B.3.2 Swift

- 早期先实现 `swiftc @file` 的读取（[初始提交](https://github.com/swiftlang/swift-driver/commit/b86325a9)）。
- 后续增加相对路径、递归检测和临时文件原子写入（[相对路径](https://github.com/swiftlang/swift-driver/commit/df0e9e19)、[原子写入](https://github.com/swiftlang/swift-driver/commit/6f5e8fed)）。
- Windows 使用 CRT 规则；macOS/Linux 使用 LLVM GNU 规则。Windows 调用 `clang.exe` 时会显式传入 `--rsp-quoting=windows`（[Windows 支持](https://github.com/swiftlang/swift-driver/commit/402b22a9)、[Windows quoting](https://github.com/swiftlang/swift-driver/commit/1bd69775)）。
- 当前 Swift Driver 根据命令行长度和目标工具能力决定是否生成临时 rsp；`-filelist` 继续专门用于输入文件清单。

Swift 的规则基本没有改，主要变化是何时生成 rsp，以及哪些任务允许使用 rsp。源码见 [Driver.swift](https://github.com/swiftlang/swift-driver/blob/main/Sources/SwiftDriver/Driver/Driver.swift) 和 [ArgsResolver.swift](https://github.com/swiftlang/swift-driver/blob/main/Sources/SwiftDriver/Execution/ArgsResolver.swift)。

### B.3.3 Go

1. [Issue #18468](https://github.com/golang/go/issues/18468) / [CL 110395](https://go-review.googlesource.com/c/go/+/110395)：为解决 Windows 命令行长度限制，引入每行一个参数的临时 rsp，不支持引号和转义。
2. [Issue #42295](https://github.com/golang/go/issues/42295) / [CL 272870](https://go-review.googlesource.com/c/go/+/272870)：增加反斜杠和换行编码，写入时编码，读取时还原。
3. [Issue #77177](https://github.com/golang/go/issues/77177) / [CL 737500](https://go-review.googlesource.com/c/go/+/737500)：为兼容 Bazel、Buck 等工具生成的 rsp，改用接近 GCC 的空白、引号和转义规则；旧的私有转义格式需要迁移。

Go 的变化发生在文件格式本身：从内部逐行格式逐步转向外部工具普遍使用的 GCC 格式。

### B.3.4 CMake 与 Ninja

CMake 根据编译器和平台生成 `rspfile`、`rspfile_content` 规则；Ninja 在执行命令前写入 rsp，命令成功后删除。文件采用哪套引号和转义规则，由 CMake 中对应的工具链配置决定，而不是由 Ninja 固定。

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
