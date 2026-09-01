# 架构会议评审：CString 空字节检查

* CJEEP ID：CJEEP-0001
* Author(s)：熊洲
* Status：Reviewing
* Implementation：待补充（Implementation PR Link）

> 关联 issue：[Cangjie/UsersForum#2730](https://gitcode.com/Cangjie/UsersForum/issues/2730)
> 【缺陷】创建 CString 时，没有检查传入字符串是否包含空字节

# 1、特性需求/问题/动机的来源与价值

1. **问题来源与背景：本特性要做什么、为什么做**

   问题的直接来源是仓颉社区issue。
   - 依据 C 标准（C11 §7.1.1），NTBS（以空字节结尾的字节串）作为字符串使用必须满足两个条件：
     1. 以空字节（`\0`）结尾；
     2. 字节串内部**不包含**空字节。
   - 现有 `LibC.mallocCString(str)` 按字符串完整长度拷贝所有字节，**不做内部空字节检查**。传入含空字节的 `String`（如 `"\0aaa"`、`"a\0b"`）时，生成的 `CString` 会被 C 侧按 NTBS 语义从首个 `\0` 处**静默截断**，数据语义错误、`\0` 后内容不可达且难以调试。
   - 鉴于 `mallocCString` 为既有公开接口，直接改变其行为会破坏兼容性，故**新增一个安全创建接口**：创建 CString 时检查空字节，**有则报错**。

2. **用户视角：实现前 vs 实现后**

   | 实现前 | 实现后 |
   |--------|--------|
   | `LibC.mallocCString("a\0b")` 静默生成内容为 `"a"` 的 CString，无任何提示，后续使用出错难以定位 | 新增 `LibC.mallocCStringSafe("a\0b")` 立即抛 `IllegalArgumentException`；既有 `mallocCString` 行为保持不变 |
   | 用户无手段在创建阶段获知输入含空字节 | 需要保证 NTBS 语义的用户可"创建即校验"，问题在源头显式暴露 |

   **期望效果**：为需要保证 NTBS 语义的用户提供"创建即校验"的安全入口，同时不破坏既有接口兼容性。

# 2、特性影响分析

1. **系统位置及周边接口（架构图标注）**

   所属模块：`std.core`，文件 `libc.cj`。

   ```
                     ┌────────────────────────┐
                     │   LibC (std.core)      │
                     │   libc.cj              │
                     │                        │
                     │  mallocCStringSafe  ←─新增：空字节校验（str.contains("\0")）→ 复用 mallocCString
                     │        │               │
                     │        ▼               │
                     │  mallocCString  ←─既有：分配 + memcpy_s 拷贝（行为完全不变）
                     │        │               │
                     │        ▼               │
                     │  CString               │
                     └────────────────────────┘
   ```

2. **环境依赖**

   - 校验直接复用 `String.contains("\0")`，仅依赖标准库 String 既有能力（内部 O(n) 子串/字符搜索，无需自行扫描字节）。
   - **不涉及多 target / 多后端 / 多 OS 差异化实现**，全部后端行为一致。

3. **涉及的模块及依赖关系、关键约束**

   - 仅修改 `std.core` 单模块，无跨模块新依赖，无特性冲突。
   - 关键约束：不改动既有公开接口签名与行为；仅新增函数。
   - 已提前沟通：issue 内官方维护者已表态（"预计会加一个告警，并在下个版本修复"）。

4. **前后版本兼容性（特别是 ABI/API）**

   - **完全向后兼容**：`mallocCString` 及其余既有接口签名、行为均不改变，存量二进制/源码无需改动。
   - 仅**新增** `LibC.mallocCStringSafe`，ABI/API 兼容性无破坏。

5. **外部用户是否感知**

   - **无破坏性感知**：既有接口行为不变。安全接口供用户按需选用，是否迁移由用户自行决定。

6. **性能影响（时间/空间）**

   - 时间：`mallocCStringSafe` = `String.contains` 一次 O(n) 检查 + `mallocCString` 一次 O(n) 拷贝，整体 O(n)，相比原创建流程仅多一次字符串搜索，可忽略。
   - 空间：O(1)，无额外分配。

# 3、业界竞品分析(可选)

1. **用户视角**

   - **Rust**：`CString::new(s)` 返回 `Result<CString, NulError>`，字符串内部含 `\0` 时返回 `Err(NulError)`，**创建即强校验**；用户可用 `expect`/`unwrap_or` 等显式处理。
   - **C/C++**：`std::string` 为长度语义，允许内部含空字节（`size` 记录真实长度）；裸 C 字符串按 NTBS 语义，遇 `\0` 即截断——正是本缺陷的根源。
   - **Python**：`ctypes.c_char_p` 接收 bytes 后 C 侧按 NTBS 使用，同样存在静默截断，社区普遍建议调用方显式校验。

2. **设计者视角**

   - Rust 采用"创建即校验 + 显式报错"，与本设计思路一致。差异在错误表达方式：
     - Rust 用 `Result<T, E>`（调用方必须分支处理）；
     - 仓颉标准库习惯对非法入参抛异常（如 `LibC.malloc` 对负数 `count` 抛 `IllegalArgumentException`），故新增接口采用**抛异常**。
   - 相对 Rust `CString::new` 是唯一创建路径，本设计的约束是"与既有 `mallocCString` 兼容并存"，因此采用**新增 API 而非修改旧 API**。

3. **对比小结**

   - "创建时校验"优于"使用时校验"（后者无法预防、需改 C 侧调用方）；
   - "新增校验接口"优于"修改既有接口"（保持兼容，符合仓颉标准库演进风格）；
   - "抛异常"优于"Result 返回值"（符合仓颉标准库既有风格）。

# 4、本特性的设计/实现方案

1. **流程图与复杂度评估**

   ```
   mallocCStringSafe(str)
     ├─ str.contains("\0")             O(n) 子串/字符搜索
     │     ├─ true   → throw IllegalArgumentException （不分配、不拷贝）
     │     └─ false  → 复用 mallocCString(str)         既有分配与拷贝逻辑
     └─ 返回 CString
   ```

   - 时间复杂度：O(n)；空间复杂度：O(1)。整体流程简单，无分支复杂性。

2. **用户接口（名称/参数/正常异常情况返回值）**

   - **新增接口**：`LibC.mallocCStringSafe(str: String): CString`
     - 参数：`str`，待创建为 CString 的字符串。
     - **正常情况**：返回与 `mallocCString(str)` 完全一致的结果（`CString`，`\0` 结尾，内容与入参一致）；内存分配失败抛 `IllegalMemoryException`，与 `mallocCString` 行为一致。
     - **异常情况**：字符串内部含空字节（`\0`）时抛 **`IllegalArgumentException`**，异常消息明确说明原因，调用方可正常 catch。
   - **既有接口**：`LibC.mallocCString(str)` 保持原行为不变。

3. **涉及模块、源文件及增删改内容**

   - 文件：`libc.cj`（`std.core` 的 `LibC`）。
   - 改动：**新增** `LibC.mallocCStringSafe`（约 7 行），其余内容零改动。实现如下：

     ```cangjie
     public unsafe static func mallocCStringSafe(str: String): CString {
         if (str.contains("\0")) {
             throw IllegalArgumentException("The string contains a null byte, which is illegal for a CString.")
         }
         return mallocCString(str)
     }
     ```

4. **多个方案对比与选择**

   - **方案 A（本设计采用）**：新增安全创建接口 `mallocCStringSafe`，内部用 `str.contains("\0")` 校验，含则抛 `IllegalArgumentException`，否则复用 `mallocCString`。
     - 利：**完全向后兼容**；实现极简（复用既有拷贝逻辑 + 标准库 `contains`，不自写字节扫描）；语义与 Rust `CString::new` 对齐（check-then-act）。
     - 弊：新增一个公开 API 面，需按标准库 API 流程评审。
   - **方案 B**：修改 `mallocCString` 增加检查并抛错。
     - 利：调用方无需感知新接口。
     - 弊：**破坏既有行为兼容性**（存量调用可能因含空字节输入从"静默成功"变为"抛异常"），被本需求明确否决。
   - **方案 C**：新增独立布尔检查接口（如 `containsNullByte(str): Bool`）。
     - 利：可复用于任意 C 互操作场景。
     - 弊：仅返回 `Bool`，"有则报错"需调用方自行 `if...throw`，语义不完整；与本方案 `contains` 校验能力重叠。可作为后续辅助接口，本设计暂不引入。
   - **方案 D**：编译期对「含 `\0` 的字符串字面量」告警/报错（对应维护者"加一个告警"的表态）。
     - 利：零运行时开销，编译期即可暴露问题。
     - 弊：只能覆盖编译期字面量，**无法覆盖运行时构造的字符串**（如拼接、`substring`），且需改动编译器/诊断链路，周期长、范围大。作为后续增强项（见第 8 节遗留问题）。
   - **结论**：本版本采用**方案 A**，运行时校验保证所有场景语义正确、错误显式化、且完全兼容。

# 5、DFX分析

- **故障/异常路径**：命中空字节时在内存分配之前即抛异常（不产生内存泄漏、无无效拷贝），异常消息明确，可一级定位。
- **内存安全**：校验阶段只读字符串（`String.contains` 内部稳健），实际分配/拷贝全部委托给 `mallocCString` 既有逻辑（`acquireRaw/releaseRaw` 配对使用），无新增内存风险。
- **可观测性**：异常即反馈，库内无需额外日志；行为可被调用方测试用例稳定断言。

# 6、可信分析

阐述是否涉及可信六性：

- **安全性（Security）**：避免将截断后的字符串传入 C 侧造成逻辑绕过（如路径校验、白名单匹配等场景的 `\0` 注入）。
- **可靠性（Reliability）**：将"看似成功、实则截断"的隐性问题显式化、可选化（用与不用由用户选择，不影响既有路径）。
- **韧性（Resilience）**：无影响（不涉及故障恢复路径）。
- **可用性（Availability）**：无影响（不涉及服务可用性）。
- **隐私（Privacy）**：无影响（不涉及数据保护）。
- **无害（Safety）**：为需要 NTBS 语义的用户提供创建即校验入口，从源头消除数据截断导致的语义错误。

# 7、关键DT用例简述

> 注：DT 用例代码随 MR 一起提交，Committer 审核特性代码的同时需审核 DT 用例代码，经审核后才能上库。

| **测试用例名称** | **预置条件** | **用例关键步骤** | **预期结果** |
|-|-|-|-|
| 正常字符串创建 CString | 仓颉标准库 CString 测试环境 | 调用 `mallocCStringSafe("hello")` | 正常返回 CString，`\0` 结尾，内容与入参一致 |
| 中间含空字节报错 | 同上 | 调用 `mallocCStringSafe("a\0b")` | 抛 `IllegalArgumentException`，消息含 null byte 提示 |
| 开头含空字节报错 | 同上 | 调用 `mallocCStringSafe("\0aaa")`（issue 场景） | 抛 `IllegalArgumentException` |
| 仅含空字节报错 | 同上 | 调用 `mallocCStringSafe("\0")` | 抛 `IllegalArgumentException` |
| 空字符串不报错 | 同上 | 调用 `mallocCStringSafe("")` | 正常返回空 CString，不抛异常 |
| 内存分配失败路径 | 同上 | 在内存不足环境下调用 `mallocCStringSafe("hello")` | 抛 `IllegalMemoryException`，与 `mallocCString` 行为一致 |
| 既有接口兼容回归 | 同上 | 调用 `mallocCString("a\0b")`、`mallocCString("abc")` | 行为与旧版本完全一致（含空字节静默截断，正常串正常返回） |
| 超长字符串回归 | 同上 | 构造跨 `SECUREC_MEM_MAX_LEN` 分片超长字符串，分别调用新/旧接口 | 校验通过，分片拷贝结果与源串一致 |

# 8、结论

1. 评审结束时，总结达成一致的主要观点和评审结论(通过/不通过)。
2. 记录遗留问题(若有)，要写明责任人及预估闭环时间。

# 附录：

## 一、可测试性设计(建议组内评审)

主要测试覆盖场景：
- 空字节分布在开头 / 中间 / 结尾 / 全空串 / 空串等所有位置形态；
- 断言异常类型（`IllegalArgumentException` / `IllegalMemoryException`）与消息内容；
- 断言正常路径返回 CString 内容与源串逐字节一致、以 `\0` 结尾；
- 断言 `mallocCStringSafe` 与 `mallocCString` 在正常路径结果一致；
- 断言 `mallocCString` 既有行为（含空字节静默截断）不变，保证兼容回归；
- 超长字符串（跨 `SECUREC_MEM_MAX_LEN` 分片）回归；
- 并发/多线程下创建 CString 无数据竞争（实现无共享可变状态）。

## 二、评审范围

本特性对应评审范围判定：
- **属于"标准库的所有 API 设计及实现"**，需上架构会议评审（本次提交的目标）；
- 属于"标准库的所有实现"，功能实现评审范围。

**架构会议需要上会评审范围：**
1. SR 级别及以上的技术架构或解决方案
2. 影响架构流程(影响编译或者执行 pipeline)的解决方案
3. 涉及多种场景，涉及多个后端，多个 OS 的技术方案，有性能影响(时间或空间)的关键方案
4. **标准库的所有 API 设计及实现 ← 本特性归属**

**功能实现评审会议评审范围：**
1. 大粒度的 AR 特性实现
2. 对周边有依赖或者有交互的 AR
3. 有多种实现方式供选择，需要决策更优实现方式的 AR
4. **标准库的所有实现**

**不需要上会评审范围：**
1. 除上会评审外的简单的可以自闭环(不影响/不依赖周边特性，不影响外部开发者)的 AR(这种 AR 需要组内评审)