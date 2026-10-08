# 标准库适配 local mode 策略问题闭环.md

- CJEEP ID：CJEEP-005
- Author(s)：Zha Junpeng, Yujiahao, Chen Qian
- Status：Reviewing
- Implementation：UnImplemented

## 1、特性需求/问题/动机的来源与价值

上次评审 [标准库适配 local mode 整体策略及部分接口适配方案](./标准库适配%20local%20mode%20整体策略及部分接口适配方案.md) 存在如下遗留问题和改进建议，本次会议主要做：**部分遗留问题和全部改进建议闭环**。

上次架构评审我们主要讨论的标准库适配 local mode 的整体策略，将标准库 APIs 划分为了：获取内部数据、设置内部成员、判定/计数、列化和反序列化 四类方法，分别讨论适配策略，并给出 `Array` 和 `String` 类型作为例子展示适配 local mode 后的类型是什么样的。

本次架构评审，我们闭环下上次评审的遗留问题和建议，修改后再完整汇报一次适配策略。同时local mode今年依然是瞄准美团众包首页刷新JSON解析场景做峰值内存的优化，所以今年标准库适配 local mode 的类型为（JSON解释实现会使用到的类型）：`Array`、`ArrayList`、`String`、`HashMap`。

## 2、特性影响分析

1. 描述该特性在整个系统中的位置及周边接口

> 今年标准库适配 local mode 的类型为（JSON解释实现会使用到的类型）：`Array`、`ArrayList`、`String`、`HashMap`。其它类型暂时不做适配。

2. 环境依赖：包括是否涉及多 target、多后端、多 OS

> 与仓颉 local mode 特性支持的目标平台、后段、系统版本保持一致。

3. 其涉及的模块或服务，及其依赖关系。描述该特性有哪些关键约束或特性冲突。是否已提前沟通？

> 无

4. 前后版本是否兼容（特别是 ABI/API）

> 确保不影响标准库、扩展库已有的接口。

5. 外部用户是否感知

> 新增接口，外部开发者可感知。

6. 分析性能影响（空间/时间）

> - 空间：
> + 包体积：涉及新增接口，标准库和扩展库编译产物的体积会增大。并且新增接口数量巨大，方法数量会翻倍，部分方法数量 x 3。
> + 内存：编译产物增大导致的运行内存增大。
> - 时间：对当前已有接口，确保其执行性能不受影响。

## 3、详细设计

|序号|遗留问题|闭环情况|
|:---|:---|:---|
|1|适配是在原有类型上扩展，还是新增类型|新增类型的话 `@~local` 和 `@local!` 实例之间就比较分裂<br> `@local?` 的价值就不大了|
|2|需不需要增加 `extend<T> Array<T> where T <: Copyable`<br> 为部分类型的 `@local?` 提供 `set` 方法 |同意新增|
|3|直接在已有方法 `this` 上增加 `@local?` 是否 ABI 兼容 |明确不兼容|
|4|`@local?`的构造函数是否需要全适配？建议按需适配|已改为按需适配（详细判定原则见 3.1 中的**构造函数适配策略**）|

|序号|修改建议|闭环情况|
|:---|:---|:---|
|1|例如`Array`的`concat`、`splitAt`、`clone`等方法会创建新的实例<br> 适配策略参考构造函数，`@local?`的场景按需适配 |优先只提供 `@local!` 版本的适配|
|2|`String`提供 `@local!`、`@local?` 和 `@~local` 之间的转换方法 |新增 `String` 的构造函数实现三者的互转 <br> 详见 3.4.2 说明|
|3|`toString` 方法 `this` 使用 `@local?` 修饰 |已修改，删除 `func toString(this @local!): String @local!`|
|4|local mode 适配方法的默认实现抛 `UnImplementedException` |标准库新增`UnImplementedException`|
|5|`String` 的 `get` 和 `[]` 方法没必要提供 `@local!` 的版本 |只提供`@local?`版本的适配，判定标准：**返回值是否是 Copyable 的，或类成员全是 Copyable 的**。<br> `get`/`[]` 返回 `Option<Byte>`/`Byte`，内容纯 Copyable，`@local?` 版本以 `@~local` 交付的结果对三种接收者完全等价，`@local!` 版本冗余，已删除；<br> 切片 `[](range)` 返回 `String`（非 Copyable，结果共享底层存储、模式随接收者），模式有信息量，`@local!`/`@local?` 两个版本保留；<br> 按同一标准修正：`indexOf` 返回 `Option<Int64>`（内容同为纯 Copyable），其 `@local!` 版本一并删除、只保留 `@local?` 版本，与 `Array` 的 `indexOf` 处理对齐|
|6|`String` 类型方法中带默认值的参数，统一是 `@~local` 类型 | 当时统一成 `@~local` 是因为当时默认参数还不支持带 local mode 标注，但现在 Spec 改了默认值可以带 local mode 标记<br> 我们根据实际方法使用场景和语义决定默认值的local mode。<br> 对于 `String` 这几个场景，默认值可以是 `@~local`。|

### 3.1、整体适配策略中新增一条构造函数的适配策略

将标准库当前接口分类新增“构造函数”这类。

| 方法类型 | 功能概述 | 示例 | 适配策略 |
| :--- | :--- | :--- | :--- |
| <span style="color:red">构造函数</span> | 创建对象实例 | 类的 `init` 方法<br> `toArray`、`clone` 等方法 | 优先适配 `@local!` <br> 按需适配 `@local?` |
| 获取内部数据 | 返回获取的类的成员数据 | `ArrayList` 的 `get` 方法<br> `HashMap` 的 `get` 方法<br> 迭代器 `Iterator` 和 `Iterable` | 适配 `@local!` 和 `@local?` 两个版本 |
| 设置内部成员 | 使用外部数据，对类实例的内部成员做修改 | `ArrayList` 的 `set` 方法<br> `Array` 的 `fill` 方法<br> `ArrayList` 的 `reverse` 和 `sort` 方法 | 只适配 `@local!` 版本，<br> 成员是 `Copyable` 的情况特殊考虑 |
| 判定/计数 | 对象间比较，查询类实例的状态 | `compare` 方法<br> `ArrayList` 的 `isEmpty` 方法<br> `HashMap` 的 `contains` 方法 | 优先适配 `@local?`，<br> 根据特殊情况适配 `@local!` |
| 序列化和反序列化 | 类实例和 `String` 类型之间的互相转换 | `ToString` 接口的 `toString` 方法（序列化）<br> `Parsable` 接口（反序列化） | 序列化：入参是 `@local?` 返回 `String @local!` <br> 反序列化：入参是 `String @local?` 返回 `@local!` 实例 |

**构造函数适配策略** 对于计划适配 local mode 的类型，我们一定会提供 `@local!` 的版本；`@local?` 构造函数按需适配。

注意：对于类似 `Array` 的 `clone`、`concat`、`splitAt` 等方法，或者 `String` 的 `join`、`fromUtf8` 等方法，会创建新的 `Array` 或 `String` 实例，与构造函数类似。是否适配 `@local?` 的版本策略同 `@local?` 的构造函数，按需适配。

### 3.2、为 `Array` 和 `ArrayList` 类型增加泛型是 `Copyable` 基类时的扩展

**Array**

```swift
extend<T> Array<T> where T <: Copyable {

 // ============== [](index, value!) — 设置指定位置的值 ==============
 // T 是 Copyable，value 不需 local 标注；返回 Unit，无需 exclave
 @Frozen @OverflowWrapping
 public operator func [](this @local?, index: Int64, value!: T): Unit

 // ============== [](range, value!) — 范围设置 ==============
 // value 是只读源，标注 @local?；Range<Int64> 非 Copyable，标注 @local?（规则 ⑨）
 // 注意：intrinsicBuiltInCopyTo 的 src 参数不支持 @local?，
 //  改用 arrayGetUnchecked + arraySetUnchecked 逐元素复制
 @Frozen
 public operator func [](this @local?, range: Range<Int64> @local?, value!: Array<T> @local?): Unit

 // ============== fill — 填充 ==============
 // T 是 Copyable，value 不需 local 标注
 @Frozen
 public func fill(this @local?, value: T): Unit

 // ============== swap — 交换两个位置的元素 ==============
 // 纯内部操作，T 是 Copyable，读写都安全
 @Frozen
 public func swap(this @local?, index1: Int64, index2: Int64): Unit

 // ============== reverse — 原地反转 ==============
 // 纯内部操作，T 是 Copyable，读写都安全
 @Frozen 
 @OverflowWrapping
 public func reverse(this @local?): Unit

 // ============== copyTo — 复制到目标数组 ==============
 // src (this) 和 dst 都是 @local?，T 是 Copyable，双向读写安全
 // 注意：intrinsicBuiltInCopyTo 的 src 参数不支持 @local?，
 //  改用 arrayGetUnchecked + arraySetUnchecked 逐元素复制
 @Frozen
 public func copyTo(this @local?, dst: Array<T> @local?, srcStart: Int64, dstStart: Int64, copyLen: Int64): Unit

 @Frozen
 public func copyTo(this @local?, dst: Array<T> @local?): Unit
}
```

**ArrayList**

`ArrayList` 中的设置内部成员的方法继承自 `List` 接口，我们新增 `ListOfCopyable` 接口引入新增的 `@local?` 实例方法。

```swift
public interface ListOfCopyable<T> <: ReadOnlyList<T> where T <: Copyable {
 func add(this @local?, element: T): Unit

 func add(this @local?, all!: Collection<T> @local?): Unit

 func add(this @local?, element: T, at!: Int64): Unit

 func add(this @local?, all!: Collection<T> @local?, at!: Int64): Unit

 func remove(this @local?, at!: Int64): T

 func remove(this @local?, range: Range<Int64> @local?): Unit

 func removeIf(this @local?, predicate: ((T) -> Bool) @local?): Unit

 func clear(this @local?): Unit

 operator func [](this @local?, index: Int64, value!: T): Unit
}
```

```swift
extend<T> ArrayList<T> <: ListOfCopyable<T> where T <: Copyable {

 // ---- List 设置方法的 @local? 版本 ----

 // ---- add(element: T) ----
 public func add(this @local?, element: T): Unit

 // ---- add(all!: Collection<T>) ----
 public func add(this @local?, all!: Collection<T> @local?): Unit

 // ---- add(element: T, at!: Int64) ----
 @OverflowWrapping
 public func add(this @local?, element: T, at!: Int64): Unit

 // ---- add(all!: Collection<T>, at!: Int64) ----
 @OverflowWrapping
 public func add(this @local?, all!: Collection<T> @local?, at!: Int64): Unit

 // ---- remove(at!: Int64): T ----
 public func remove(this @local?, at!: Int64): T

 // ---- remove(range: Range<Int64>) ----
 public func remove(this @local?, range: Range<Int64> @local?): Unit

 // ---- removeIf(predicate) ----
 public func removeIf(this @local?, predicate: ((T) -> Bool) @local?): Unit

 // ---- clear ----
 public func clear(this @local?): Unit

 // ---- operator [](index, value!) ----
 public operator func [](this @local?, index: Int64, value!: T): Unit

 // ==================== ArrayList 特有方法（不在 List/ListOfCopyable 接口中） ====================

 // reverse：ArrayList 特有方法，不在 List 接口中
 @Frozen
 public func reverse(this @local?): Unit

 // reserve：ArrayList 特有方法，不在 List 接口中
 @Frozen
 public func reserve(this @local?, additional: Int64): Unit
}
```

### 3.3、标准库 `std.core` 新增 `UnimplementedException`

```swift
package std.core

public class UnimplementedException <: Exception {
 public init()

 public init(message: String)

 protected override func getClassName(): String {
  return "UnimplementedException"
 }
}
```

### 3.3、`toString` 只提供 `this` 是 `@local?` 的适配方法

```swift
public interface ToString {
 ... // 原有方法

 func toString(this @local?): String @local! {
  throw UnimplementedException("Current Type does not implement 'toString' method for its @local? instance")
 }
}
```

### 3.4、`Array`、`ArrayList`、`String`、`HashMap` 基于整改后的适配

#### 3.4.1、`Array`

```swift
@ConstSafe
public struct Array<T> {

 // ==================== 构造函数 ====================
 // 新增: local 构造函数
 @Frozen
 public const init(this @local!)

 @Frozen
 public init(this @local!, size: Int64, repeat!: T @local!)

 // 新增: local 构造函数
 @Frozen
 init(this @local!, elements: Collection<T> @local!)

 // 新增: local 构造函数
 @Frozen
 public init(this @local!, size: Int64, initElement: ((Int64) -> T @local!) @ local?)

 // 提供最基本的 @local? 的构造方法
 @Frozen
 init(this @local?, elements: Collection<T> @local?)

 @Frozen
 public init(this @local?, size: Int64, initElement: ((Int64) -> T @local?) @ local?)

 // ==================== 等价于构造函数的方法 =============

 // ---- slice ----
 // 新增: @local!
 @Frozen
 public func slice(this @local!, start: Int64, len: Int64): Array<T> @local!

 // 新增: @local? 版本
 // 修正：slice 返回与接收者共享底层 RawArray 的视图（stdlib 实现为 Array(this.rawptr, ...)），
 // 结果模式被接收者锁死；返回 Array<T> 非 Copyable、模式有信息量，
 // 两个接收者模式各需一个版本（与初始方案及 ArrayList.slice 的双版本对齐）
 @Frozen
 public func slice(this @local?, start: Int64, len: Int64): Array<T> @local?

 // ---- clone ----
 // 新增: @local! 版本
 @Frozen
 public func clone(this @local!): Array<T> @local!

 // ---- clone(range) ----
 // 新增: @local! 版本
 @Frozen
 @OverflowWrapping
 public func clone(this @local!, range: Range<Int64> @ local?): Array<T> @local!

 // ---- copyTo(dst, ...) ----
 // 新增: @local! 版本（src @local! → dst @local!）
 @Frozen
 public func copyTo(this @local!, dst: Array<T> @local!, srcStart: Int64, dstStart: Int64, copyLen: Int64): Unit

 // 新增: @local! 版本
 @Frozen
 public func copyTo(this @local!, dst: Array<T> @local!): Unit

 // ---- concat ----
 // 新增: @local! 版本
 @Frozen
 public func concat(this @local!, other: Array<T> @local!): Array<T> @local!

 // ---- splitAt — 获取内部数据（slice） ----
 // 新增: @local! 版本
 @Frozen
 public func splitAt(this @local!, mid: Int64): (Array<T>, Array<T>) @local!

 // 新增: @local? 版本
 // 修正：splitAt 返回的两个子数组同样是共享底层 RawArray 的视图（stdlib 实现为
 // Array(this.rawptr, this.start, mid) 与 Array(this.rawptr, this.start + mid, ...)），
 // 结果模式被接收者锁死，且视图无法由任何构造函数替代——必须提供 @local? 版本
 @Frozen
 public func splitAt(this @local?, mid: Int64): (Array<T>, Array<T>) @local?

 // ---- repeat ----
 // 新增: @local! 版本
 @Frozen
 public func repeat(this @local!, n: Int64): Array<T> @local!

 // ---- map ----
 // 新增: @local! 版本
 @Frozen
 public func map<R>(this @local!, transform: ((T @local!) -> R @local!) @local?): Array<R> @local!

 // ---- step ----
 // 新增: @local! 版本
 @When[env != "ohos"]
 public func step(this @local!, count: Int64): Array<T> @local!

 // ---- take ----
 // 新增: @local! 版本
 @When[env != "ohos"]
 public func take(this @local!, count: Int64): Array<T> @local!

 // ---- skip ----
 // 新增: @local! 版本
 @When[env != "ohos"]
 public func skip(this @local!, count: Int64): Array<T> @local!

 // ---- filter ----
 // 新增: @local! 版本
 @When[env != "ohos"]
 public func filter(this @local!, predicate: ((T @local?) -> Bool) @local?): Array<T> @local!

 // ---- flatMap ----
 // 新增: @local! 版本
 @When[env != "ohos"]
 public func flatMap<R>(this @local!, transform: ((T @local!) -> Array<R> @local!) @local?): Array<R> @local!

 // ---- filterMap ----
 // 新增: @local! 版本
 @When[env != "ohos"]
 public func filterMap<R>(this @local!, transform: ((T @local!) -> ?R @local!) @local?): Array<R> @local!

 // ---- intersperse ----
 // 新增: @local! 版本
 @When[env != "ohos"]
 public func intersperse(this @local!, separator: T @local!): Array<T> @local!

 // ---- forEach ----
 // 新增: @local! 版本
 @When[env != "ohos"]
 public func forEach(this @local!, action: ((T @local!) -> Unit) @local?): Unit

 // ==================== 获取内部数据 ====================

 // ---- prop first ----
 // 原有（不变）
 ...

 // 新增: @local! 版本
 @Frozen
 public prop first: Option<T> @local!

 @Frozen
 public prop first: Option<T> @local?

 // ---- prop last ----
 // 原有（不变）
 ...

 // 新增: @local! 版本
 @Frozen
 public prop last: Option<T> @local!

 @Frozen
 public prop last: Option<T> @local?

 // ==================== 获取内部数据 ====================

 // ---- get(index) ----
 // 新增: @local! 版本
 @Frozen
 @OverflowWrapping
 public func get(this @local!, index: Int64): Option<T> @local! 

 // 新增: @local? 版本（被 contains 等判定方法调用）
 @Frozen
 @OverflowWrapping
 public func get(this @local?, index: Int64): Option<T> @local? 

 // ---- [](index): T — 获取内部数据 ----
 // 新增: @local! 版本
 @Frozen
 @OverflowWrapping
 public operator func [](this @local!, index: Int64): T @local!

 // 新增: @local? 版本
 @Frozen
 @OverflowWrapping
 public operator func [](this @local?, index: Int64): T @local?

 // ---- [](range) — 获取内部数据（slice） ----
 // 新增: @local! 版本
 @Frozen
 public operator func [](this @local!, range: Range<Int64> @local?): Array<T> @local!

 // 新增: @local! 版本
 @Frozen
 public operator func [](this @local?, range: Range<Int64> @local?): Array<T> @local?

 // --- 返回 Copyable 类型，只适配 @local? 版本 ---
 @Frozen
 public func indexOf(this @local?, element: T @local?, fromIndex: Int64): Option<Int64>

 @Frozen
 public func indexOf(this @local?, elements: Array<T> @local?): Option<Int64>

 public func indexOf(this @local?, elements: Array<T> @local?, fromIndex: Int64): Option<Int64>

 @Frozen
 public func lastIndexOf(this @local?, element: T @local?): Option<Int64>

 @Frozen
 public func lastIndexOf(this @local?, element: T @local?, fromIndex: Int64): Option<Int64>

 @Frozen
 public func lastIndexOf(this @local?, elements: Array<T> @local?): Option<Int64>

 @Frozen
 public func lastIndexOf(this @local?, elements: Array<T> @local?, fromIndex: Int64): Option<Int64>

 // ==================== 设置内部成员 ====================

 // ---- [](index, value!) — 设置 ----
 // 新增: @local! 版本
 @Frozen
 @OverflowWrapping
 public operator func [](this @local!, index: Int64, value!: T @local!): Unit

 // ---- [](range, value!) — 设置 ----
 // 新增: @local! 版本（value 作为输入数据存入 @local! rawptr）
 @Frozen
 public operator func [](this @local!, range: Range<Int64> @local?, value!: Array<T> @local!): Unit

 // ---- fill — 设置内部成员 ----
 // 新增: @local! 版本（value 存为成员，需 @local!）
 @Frozen
 public func fill(this @local!, value: T @local!): Unit

 // ---- reverse ----
 // 新增: @local! 版本
 @Frozen
 @OverflowWrapping
 public func reverse(this @local!): Unit

 // ---- swap ----
 // 新增: @local! 版本
 @Frozen
 public func swap(this @local!, index1: Int64, index2: Int64): Unit

 // ==================== 判定类 ====================

 // 新增 @local?（不修改原方法，避免 ABI 问题）
 @Frozen
 public func isEmpty(this @local?): Bool {
  return this.len == 0
 }

 // ---- all / any / none — 判定类 ----
 // 特例：新增方法
 @When[env != "ohos"]
 public func all(this @ local?, predicate: ((T @ local?) -> Bool) @ local?): Bool

 // 特例：新增方法
 @When[env != "ohos"]
 public func any(this @ local?, predicate: ((T @ local?) -> Bool) @ local?): Bool

 // 特例：新增方法
 @When[env != "ohos"]
 public func none(this @ local?, predicate: ((T @ local?) -> Bool) @ local?): Bool

 // ---- fold / reduce — 数据转换 ----
 @When[env != "ohos"]
 public func fold<R>(this @local!, initial: R @ local!, operation: ((R, T) @ local! -> R @ local!) @local?): R @local!
}
```

#### 3.4.2、`String`

```swift
@When[backend == "cjnative"]
@ConstSafe
public struct String <: Collection<Byte> & Comparable<String> & Hashable & ToString {
 let myData: RawArray<UInt8>
 let start: UInt32
 let length: UInt32
 
 public static const empty: String = String()
 
 // ==================== 构造函数 ====================
 
 // 新增：@local! 构造函数
 @Frozen
 public const init(this @local!)
 
 // 新增：从 Array<Rune> 构造 @local! String
 @Frozen
 @OverflowWrapping
 public init(this @local!, value: Array<Rune> @ local?)
 
 // 新增：从 Collection<Rune> 构造 @local! String
 @Frozen
 @OverflowWrapping
 public init(this @local!, value: Collection<Rune> @local?)

 // 新增：从 @local? String 实例构造出 @local! String 实例的方法
 // 实现必须为深拷贝：在强制 exclave 内把 value 的字节内容复制到调用者区域新分配的存储。
 // 不能共享底层 RawArray（零拷贝）：value 的静态模式是 local?，实际对象可能是绑定当前函数
 // 区域的 local! 实例（底层 RawArray 随该区域消亡），共享会使生命周期更长的 @local! 新实例
 // 持有悬垂引用（use-after-free）。深拷贝后，新实例的存储与其实例模式对应区域的生命周期一致。
 // 对实际为 @~local 的来源也做复制，是静态健全性的代价：local? 的实际模式无法在编译期区分。
 public init(this @local!, value: String @ local?)

 // 新增：从 @local? String 实例构造出 non-local String 实例的方法
 // 实现同样必须为深拷贝（复制到堆存储）：@~local 实例可逃逸当前区域，若共享 local! 来源的
 // 区域存储，新实例同样会产生悬垂引用。
 public init(value: String @ local?)
 
 // ...原构造函数保持不变

 // ==================== 等价于构造函数 ====================

 // 新增：@local! 版本的 clone
 @Frozen
 public func clone(this @local!): String @local!

 // 新增：@local! 版本的 toArray
 @Frozen
 public func toArray(this @local!): Array<Byte> @local!
 
 // 新增：@local! 版本的 toRuneArray
 @Frozen
 @OverflowWrapping
 public func toRuneArray(this @local!): Array<Rune> @local!

 // 新增：@local! 版本的 operator +（拼接）
 @Frozen
 @OverflowWrapping
 public operator func +(this @local!, other: String @local?): String @local!
 
 // 新增：@local! 版本的 operator *（重复）
 @Frozen
 public operator func *(this @local!, count: Int64): String @local!
 
 // 新增：@local! 版本的 replace
 @Frozen
 public func replace(this @local!, old: String @local!, new: String @local?): String @local!
 
 // 新增：@local! 版本的 toAsciiLower
 @Frozen
 @OverflowWrapping
 public func toAsciiLower(this @local!): String @local!
 
 // 新增：@local! 版本的 toAsciiUpper
 @Frozen
 @OverflowWrapping
 public func toAsciiUpper(this @local!): String @local!
 
 // 新增：@local! 版本的 toAsciiTitle
 @Frozen
 @OverflowWrapping
 public func toAsciiTitle(this @local!): String @local!
 
 // 新增：@local! 版本的 trimAscii 系列
 @Frozen
 public func trimAscii(this @local!): String @local!
 
 @Frozen
 @OverflowWrapping
 public func trimAsciiStart(this @local!): String @local!
 
 @Frozen
 @OverflowWrapping
 public func trimAsciiEnd(this @local!): String @local!
 
 // 新增：@local! 版本的 removePrefix/removeSuffix
 @Frozen
 public func removePrefix(this @local!, prefix: String @local?): String @local!
 
 @Frozen
 public func removeSuffix(this @local!, suffix: String @local?): String @local!
 
 // 新增：@local! 版本的 split
 @Frozen
 public func split(this @local!, str: String @local?, removeEmpty!: Bool = false): Array<String> @local!
 
 @Frozen
 public func split(this @local!, str: String @local?, maxSplits: Int64, removeEmpty!: Bool = false): Array<String> @local!

 // 新增：@local? 版本的 fromUtf8，作为 local 和 non-local 实例之间的交互边界
 @Frozen
 public static func fromUtf8(utf8Data: Array<UInt8> @ local?): String @local!
 
 // 新增：@local! 版本的 fromUtf8Unchecked
 @Frozen
 public unsafe static func fromUtf8Unchecked(utf8Data: Array<UInt8> @ local?): String @local!
 
 // 新增：@local! 版本的 padStart/padEnd
 @Frozen
 public func padStart(this @local!, totalWidth: Int64, padding!: String = " "): String @local!
 
 @Frozen
 public func padEnd(this @local!, totalWidth: Int64, padding!: String = " "): String @local!
 
 // 新增：@local! 版本的 static join
 @Frozen
 @OverflowWrapping
 public static func join(strArray: Array<String> @local!, delimiter!: String): String @local!

 // 数据转换类
 @Frozen
 public unsafe func rawData(this @local!): Array<Byte> @local!
 
 // ==================== prop（判定/获取类） ====================
 
 // 新增
 @Frozen
 public prop size: Int64 @local?
 
 // ==================== 判定类方法（修改为 @local?） ====================
 // 因为原有方法标记了 @Frozen，这里选择新增
 
 @Frozen
 public func isEmpty(this @local?): Bool
 
 @Frozen
 public func isAscii(this @local?): Bool
 @Frozen
 public func isAsciiBlank(this @local?): Bool
 
 @Frozen
 public func contains(this @local?, str: String @local?): Bool
 
 @Frozen
 @OverflowWrapping
 public func startsWith(this @local?, prefix: String @local?): Bool
 
 @Frozen
 @OverflowWrapping
 public func endsWith(this @local?, suffix: String @local?): Bool
 
 @Frozen
 @OverflowWrapping
 public func equalsIgnoreAsciiCase(this @local?, other: String @local?): Bool
 
 // 新增：@local? 版本的 indexOf（获取内部数据）
 // 修正：返回 Option<Int64> 内容为纯 Copyable、模式无信息量，按建议5 的同一标准
 // 只提供 @local? 版本（原 @local! 版本删除，与 Array 的 indexOf 处理对齐）
 @Frozen
 public func indexOf(this @local?, b: Byte): Option<Int64>
 
 @Frozen
 public func indexOf(this @local?, str: String @local?): Option<Int64>
 
 // ... 其他原方法保持不变

 // ... 原有方法不变
 
 // ==================== Comparable 接口（判定类） ====================
 
 @Frozen
 public func compare(this @local?, str: String @local?): Ordering
 
 @Frozen
 public operator const func ==(this @local?, other: String @local?): Bool
 @Frozen
 public operator const func !=(this @local?, other: String @local?): Bool
 
 @Frozen
 public operator const func <(this @local?, other: String @local?): Bool
 
 @Frozen
 public operator const func <=(this @local?, other: String @local?): Bool
 
 @Frozen
 public operator const func >(this @local?, other: String @local?): Bool
 
 @Frozen
 public operator const func >=(this @local?, other: String @local?): Bool
 
 // ==================== Hashable 接口（判定类） ====================
 // 标记了 @Frozen，从兼容性角度考虑，选择新增
 
 @Frozen
 @OverflowWrapping
 public func hashCode(this @local?): Int64
 
 // ==================== ToString 接口 ====================
 
 // 新增：@local! 实例的 toString（跨越边界，返回 @~local）
 @Frozen
 public func toString(this @local?): String @local!
 
 // ==================== 获取内部数据 ====================
 
 // 新增：@local? 版本
 @Frozen
 @OverflowWrapping
 public func get(this @local?, index: Int64): Option<Byte>
 
 @Frozen
 @OverflowWrapping
 public operator const func [](this @local?, index: Int64): Byte
 
 // ==================== 获取内部数据 ====================
 
 // 新增：@local! 版本的 slice（获取内部数据/数据转换）
 @Frozen
 public operator const func [](this @local!, range: Range<Int64> @ local?): String @local!

 // 新增：@local! 版本的 slice（获取内部数据/数据转换）
 @Frozen
 public operator const func [](this @local?, range: Range<Int64> @ local?): String @local?
 
 // 新增：@local! 版本的 iterator
 @Frozen
 public func iterator(this @local!): Iterator<Byte> @local!

 // 新增：@local? 版本的 iterator
 @Frozen
 public func iterator(this @local?): Iterator<Byte> @local?
 
 // 新增：@local! 版本的 runes 迭代器
 @Frozen
 public func runes(this @local!): Iterator<Rune> @local!

 // 新增：@local? 版本的 runes 迭代器（迭代器惰性共享底层存储，结果模式锁死，与 iterator 对齐）
 @Frozen
 public func runes(this @local?): Iterator<Rune> @local?
 
 // 新增：@local! 版本的 lines 迭代器
 @Frozen
 public func lines(this @local!): Iterator<String> @local!

 // 新增：@local? 版本的 lines 迭代器（判定依据同 runes：惰性共享、结果模式锁死）
 @Frozen
 public func lines(this @local?): Iterator<String> @local?
 ...
}
```

`String` 不同 mode 之间的转换：
- `@~local -> @local!`：使用构造函数 `init(this @local!, value: String @ local?)`；
- `@local! -> @local?`: 天然成立；
- `@local? -> @local!`：使用构造函数 `init(this @local!, value: String @ local?)`；
- `@local? -> @~local`：使用构造函数 `init(value: String @ local?)`。

#### 3.4.3、`ArrayList`

```swift
public class ArrayList<T> <: List<T> {
 
 // ==================== 构造函数 ====================
 
 // 新增：@local! 构造函数
 @Frozen
 public init(this @local!)
 
 @Frozen
 public init(this @local!, capacity: Int64)
 
 @Frozen
 public init(this @local!, size: Int64, initElement: ((Int64) -> T @local!) @local?)
 
 @Frozen
 public init(this @local!, elements: Collection<T> @local!)

 // 新增：@local? 构造函数
 // 元素非 Copyable 时，local? 模式的元素既传不进原有 @~local 构造函数（local? 不是 ~local 的子模式），
 // 也传不进 @local! 构造函数（local? 不是 local! 的子模式）；且规范禁止在构造函数之外向 @local? 容器
 // 逐个写入非 Copyable 元素，构造函数内的初始化赋值（this 在构造时尚无别名，配合强制 exclave 保证健全性）
 // 是规范留出的唯一合法通道。元素类型不保证 Copyable 的容器必须提供本构造函数（与 3.4.1 的 Array 保持一致）。
 @Frozen
 public init(this @local?, elements: Collection<T> @local?)

 @Frozen
 public init(this @local?, size: Int64, initElement: ((Int64) -> T @local?) @ local?)

 // =================== 等价于构造函数 =================

 @Frozen
 public func toArray(this @local!): Array<T> @local!

 // 新增: @local? 版本
 // 修正：容器拷贝、元素引用共享，结果模式被接收者锁死；Collection 接口已声明本签名，实现必须补齐
 @Frozen
 public func toArray(this @local?): Array<T> @local?
 
 @Frozen
 public func clone(this @local!): ArrayList<T> @local!

 // 新增: @local? 版本（判定依据同 toArray/Array.clone：容器拷贝、元素引用共享、结果模式锁死；
 // stdlib 实现即 ArrayList<T>(this)，与 @local? 构造函数同构，此处保持方法形式一致性）
 @Frozen
 public func clone(this @local?): ArrayList<T> @local?

 // 高阶函数：数据转换类
 @When[env != "ohos"]
 public func filter(this @local!, predicate: ((T @local?) -> Bool) @local?): ArrayList<T> @local!
 
 @When[env != "ohos"]
 public func map<R>(this @local!, transform: ((T @local!) -> R @local!) @local?): ArrayList<R> @local!
 
 @When[env != "ohos"]
 public func flatMap<R>(this @local!, transform: ((T @local!) -> ArrayList<R> @local!) @local?): ArrayList<R> @local!
 
 @When[env != "ohos"]
 public func filterMap<R>(this @local!, transform: ((T @local!) -> ?R @local!) @local?): ArrayList<R> @local!
 
 @When[env != "ohos"]
 public func step(this @local!, count: Int64): ArrayList<T> @local!
 
 @When[env != "ohos"]
 public func take(this @local!, count: Int64): ArrayList<T> @local!
 
 @When[env != "ohos"]
 public func skip(this @local!, count: Int64): ArrayList<T> @local!
 
 @When[env != "ohos"]
 public func intersperse(this @local!, separator: T @local!): ArrayList<T> @local!
 
 @When[env != "ohos"]
 public func zip<R>(this @local!, other: ArrayList<R> @local!): ArrayList<(T, R)> @local!
 
 @When[env != "ohos"]
 public func enumerate(this @local!): ArrayList<(Int64, T)> @local!
 
 // ... 原构造函数保持不变
 
 // ==================== prop ====================
 
 // 新增
 public prop capacity: Int64 @local?
 // 新增
 public prop size: Int64 @local?

 ...

 // 新增
 public prop first: ?T @local!

 public prop first: ?T @local?

 // 新增
 public prop last: ?T @local!

 public prop last: ?T @local?
 
 // ==================== 获取内部数据 ====================
 
 // 新增：@local! 和 @local? 版本
 @Frozen
 public func get(this @local!, index: Int64): ?T @local!

 @Frozen
 public func get(this @local?, index: Int64): ?T @local?
 
 // 新增：@local! 和 @local? 版本
 @Frozen
 @OverflowWrapping
 public operator func [](this @local!, index: Int64): T @local!
 
 @Frozen
 @OverflowWrapping
 public operator func [](this @local?, index: Int64): T @local?
 
 // ... 原方法保持不变
 
 // ==================== 设置内部成员 ====================
 
 // 新增：@local! 版本
 @Frozen
 public func add(this @local!, element: T @local!): Unit
 
 @Frozen
 public func add(this @local!, all!: Collection<T> @local!): Unit
 
 @Frozen
 @OverflowWrapping
 public func add(this @local!, element: T @local!, at!: Int64): Unit
 
 @Frozen
 @OverflowWrapping
 public operator func [](this @local!, index: Int64, value!: T @local!): Unit
 
 // ... 原方法保持不变
 // ... 其他原 add 方法保持不变
 
 // 新增：@local! 版本
 @Frozen
 public func remove(this @local!, at!: Int64): T @local!
 
 @Frozen
 public func remove(this @local!, range: Range<Int64> @ local?): Unit
 
 @Frozen
 public func removeIf(this @local!, predicate: ((T @local!) -> Bool) @local?): Unit
 
 @Frozen
 public func clear(this @local!): Unit
 
 @Frozen
 public func reverse(this @local!): Unit

 @When[env != "ohos"]
 public func fold<R>(this @local!, initial: R @local!, operation: ((R, T) @local! -> R @local!) @local?): R @local!

 @When[env != "ohos"]
 public func reduce(this @local!, operation: ((T, T) @local! -> T @local!) @local?): Option<T> @local!
 
 // 原方法保持不变
 // ... 其他原 remove/clear/reverse 方法保持不变
 
 // ==================== 获取类内部成员 ====================
 
 // 新增：@local! 版本
 @Frozen
 public func iterator(this @local!): Iterator<T> @local!
 
 @Frozen
 public func slice(this @local!, range: Range<Int64> @local?): ArrayList<T> @local!
 
 @Frozen
 public operator func [](this @local!, range: Range<Int64> @local?): ArrayList<T> @local!

 // 新增：@local? 版本
 @Frozen
 public func iterator(this @local?): Iterator<T> @local?
 
 @Frozen
 public func slice(this @local?, range: Range<Int64> @local?): ArrayList<T> @local?
 
 @Frozen
 public operator func [](this @local?, range: Range<Int64> @local?): ArrayList<T> @local?
 
 // 原方法保持不变
 // ... 其他原数据转换方法保持不变
 
 // ==================== 判定类（修改为 @local?） ====================
 
 // 新增 @local?，目的是不破坏兼容性
 @Frozen
 public func isEmpty(this @local?): Bool

 // 新增
 @When[env != "ohos"]
 public func all(this @local?, predicate: ((T @local?) -> Bool) @local?): Bool
 
 // 新增
 @When[env != "ohos"]
 public func any(this @local?, predicate: ((T @local?) -> Bool) @local?): Bool
 
 // 新增
 @When[env != "ohos"]
 public func none(this @local?, predicate: ((T @local?) -> Bool) @local?): Bool
}
```

#### 3.4.4、`HashMap`

覆盖 `HashMap` 全部 public API（含通过 `Map`、`Equatable`、`ToString` 扩展暴露的成员）。`HashMap` 实现了 `Map` 接口：local 化的 `HashMap` 以 `Map` 类型使用时，调用的是接口成员，故 `Map` 接口族须与实现类同步适配（正如 `ArrayList` 之于 `List`）；经 `ReadOnlyMap <: Collection<(K, V)>` 继承的 `size`/`isEmpty`/`toArray`/`iterator` 沿用 `Collection` 接口既有适配（`E` 即 `(K, V)`），不重复列出。

两点约定：

- **高阶函数按"有无容器/累积产出"分组**：无产出的（forEach/all/any/none）提供 `this @local?` 单版本宽门；有产出的（mapValues/filter/fold/reduce）按 ArrayList 既有形态先落 `this @local!`，其 `@local?` 接收者版本涉及转换函数的模式策略（后续批次），决策后补齐。
- `mapValues`/`fold`/`reduce` 的回调输入标 `@local?`（只读来源），较 ArrayList.map 的 `((T @local!) -> ...)` 历史形态更宽且同样健全；ArrayList 的历史形态可在批次 C 一并放宽。

```swift
public interface Map<K, V> <: ReadOnlyMap<K, V> {
 // ---- 设置内部成员（Map 自有，只适配 @local!） ----
 // 新增：写入（元素传染）；返回被覆盖旧值随接收者锁死（?V 即 Option<V> 简写）
 func add(this @local!, key: K @local!, value: V @local!): ?V @local!
 // 新增：整批写入
 func add(this @local!, all!: Collection<(K, V)> @local!): Unit
 // 新增：删除是写操作→ this @local!；key 只读→ @local?
 func remove(this @local!, key: K @local?): Option<V> @local!
 // 新增：键集合只读 @local?（不被存储、无传染）
 func remove(this @local!, all!: Collection<K> @local?): Unit
 // 新增：谓词只读，删除动作在 @local! 接收者上
 func removeIf(this @local!, predicate: ((K @local?, V @local?) -> Bool) @local?): Unit
 // 新增：写操作
 func clear(this @local!): Unit
 // 新增：下标赋值（upsert），判定同 add
 operator func [](this @local!, key: K @local!, value!: V @local!): Unit
 // 新增：条件写入，key/value 均可能被存储；原成员自带 @Frozen
 @Frozen
 func addIfAbsent(this @local!, key: K @local!, value: V @local!): ?V @local!
 // 新增：替换写入，判定同 add；原成员自带 @Frozen
 @Frozen
 func replace(this @local!, key: K @local!, value: V @local!): ?V @local!

 // ---- 获取内部数据（ReadOnlyMap 声明， / / ） ----
 // 新增：读取双版本；key 只读
 func get(this @local!, key: K @local?): ?V @local!
 func get(this @local?, key: K @local?): ?V @local?
 // 新增：下标访问双版本（键不存在抛 NoneValueException）
 operator func [](this @local!, key: K @local?): V @local!
 operator func [](this @local?, key: K @local?): V @local?
 // 新增：Bool 可复制+ key 只读单版本
 func contains(this @local?, key: K @local?): Bool
 func contains(this @local?, all!: Collection<K> @local?): Bool
 // 新增：视图共享底层存储（双版本）
 func keys(this @local!): EquatableCollection<K> @local!
 func keys(this @local?): EquatableCollection<K> @local?
 func values(this @local!): Collection<V> @local!
 func values(this @local?): Collection<V> @local?
 // 新增：条目视图（双版本）
 func entryView(this @local!, k: K @local?): MapEntryView<K, V> @local!
 func entryView(this @local?, k: K @local?): MapEntryView<K, V> @local?
}

public interface MapEntryView<K, V> {
 // 新增：key 只读（单版本宽门）
 prop key: K @local?
 // 新增：mut prop 含 setter（写入会存储）→ 值以 @local! 交付/接收
 mut prop value: ?V @local!
}

public interface EquatableCollection<T> <: Collection<T> {
 // 新增：Bool 可复制+ 只读单版本；其余成员沿用 Collection 适配
 func contains(this @local?, element: T @local?): Bool
 func contains(this @local?, all!: Collection<T> @local?): Bool
}
```

```swift
public class HashMap<K, V> <: Map<K, V> where K <: Hashable & Equatable<K> {
 // ==================== 构造函数====================
 @Frozen
 public init(this @local!)

 @Frozen
 public init(this @local!, capacity: Int64)

 @Frozen
 public init(this @local!, elements: Array<(K, V)> @local!)

 @Frozen
 public init(this @local!, elements: Collection<(K, V)> @local!)

 // initElement 产出的键值对会被存储 → 返回值标 @local!；Int64 入参可复制不标
 @Frozen
 public init(this @local!, size: Int64, initElement: (Int64) -> (K, V) @local!)

 // 宽门：元素不保证可复制，必须提供；禁止 this @local! 配 @local? 来源（承诺撒谎）
 @Frozen
 public init(this @local?, elements: Array<(K, V)> @local?)

 @Frozen
 public init(this @local?, elements: Collection<(K, V)> @local?)

 @Frozen
 public init(this @local?, size: Int64, initElement: (Int64) -> (K, V) @local?)

 // ==================== 设置内部成员（只读 key 放宽）====================
 @Frozen
 public func add(this @local!, key: K @local!, value: V @local!): Option<V> @local!

 @Frozen
 public func add(this @local!, all!: Collection<(K, V)> @local!): Unit

 @Frozen
 public operator func [](this @local!, key: K @local!, value!: V @local!): Unit

 // remove 只做查找（key 只读 → @local?）；被删值 = 元素读出，随接收者锁死 @local!
 @Frozen
 public func remove(this @local!, key: K @local?): Option<V> @local!

 @Frozen
 public func remove(this @local!, all!: Collection<K> @local?): Unit

 @Frozen
 public func removeIf(this @local!, predicate: ((K @local?, V @local?) -> Bool) @local?): Unit

 @Frozen
 public func clear(this @local!): Unit

 @Frozen
 public func reserve(this @local!, additional: Int64): Unit

 // ==================== 获取内部数据====================
 @Frozen
 public func get(this @local!, key: K @local?): Option<V> @local!

 @Frozen
 public func get(this @local?, key: K @local?): Option<V> @local?

 @Frozen
 public operator func [](this @local!, key: K @local?): V @local!

 @Frozen
 public operator func [](this @local?, key: K @local?): V @local?

 // Bool 可复制 → 单版本；key 集合只读 → 
 @Frozen
 public func contains(this @local?, key: K @local?): Bool

 @Frozen
 public func contains(this @local?, all!: Collection<K> @local?): Bool

 @Frozen
 public func isEmpty(this @local?): Bool

 // 键/值集合视图、条目视图、迭代器：共享底层存储 → 双版本
 // （HashMapIterator 遵循 Iterator 同款适配：next 双版本；MapEntryView 的 mut prop value 按 mut prop 约定处理）
 @Frozen
 public func keys(this @local!): EquatableCollection<K> @local!

 @Frozen
 public func keys(this @local?): EquatableCollection<K> @local?

 @Frozen
 public func values(this @local!): Collection<V> @local!

 @Frozen
 public func values(this @local?): Collection<V> @local?

 @Frozen
 public func entryView(this @local!, key: K @local?): MapEntryView<K, V> @local!

 @Frozen
 public func entryView(this @local?, key: K @local?): MapEntryView<K, V> @local?

 @Frozen
 public func iterator(this @local!): HashMapIterator<K, V> @local!

 @Frozen
 public func iterator(this @local?): HashMapIterator<K, V> @local?

 // 容器拷贝（新数组/新映射共享元素引用）→ 双版本
 @Frozen
 public func toArray(this @local!): Array<(K, V)> @local!

 @Frozen
 public func toArray(this @local?): Array<(K, V)> @local?

 @Frozen
 public func clone(this @local!): HashMap<K, V> @local!

 @Frozen
 public func clone(this @local?): HashMap<K, V> @local?

 // 格式化生成全新字节，不共享 → 单版本
 @Frozen
 public func toString(this @local?): String @local! where V <: ToString, K <: ToString

 // Bool 可复制 + right 只读 → 单版本
 @Frozen
 public operator func ==(this @local?, right: HashMap<K, V> @local?): Bool where V <: Equatable<V>

 // ==================== prop====================
 @Frozen
 public prop size: Int64 @local?

 @Frozen
 public prop capacity: Int64 @local?

 // ==================== 高阶函数 ====================
 // 无容器/累积产出：只读遍历与判定 → 单版本宽门
 @Frozen
 public func forEach(this @local?, action: ((K @local?, V @local?) -> Unit) @local?): Unit

 @Frozen
 public func all(this @local?, predicate: ((K @local?, V @local?) -> Bool) @local?): Bool

 @Frozen
 public func any(this @local?, predicate: ((K @local?, V @local?) -> Bool) @local?): Bool

 @Frozen
 public func none(this @local?, predicate: ((K @local?, V @local?) -> Bool) @local?): Bool

 // 有容器/累积产出：按 ArrayList 既有形态落 this @local!；@local? 接收者版本待批次 C
 // 回调输入为只读来源（@local?），产出会被存储（→ @local!）；
 // reduce 的首个入参是跨调用传递的累积值，非只读来源 → @local!
 @Frozen
 public func mapValues<R>(this @local!, transform: ((K @local?, V @local?) -> R @local!) @local?): HashMap<K, R> @local!

 @Frozen
 public func mapValues<R>(this @local!, transform: ((V @local?) -> R @local!) @local?): HashMap<K, R> @local!

 @Frozen
 public func filter(this @local!, predicate: ((K @local?, V @local?) -> Bool) @local?): HashMap<K, V> @local!

 @Frozen
 public func fold<R>(this @local!, initial: R @local!, operation: ((R @local!, K @local?, V @local?) -> R @local!) @local?): R @local!

 @Frozen
 public func reduce(this @local!, operation: ((V @local!, V @local?) -> V @local!) @local?): Option<V> @local!
}
```

## 4、DFX 分析

## 5、关键 DT 用例简述

## 6、结论