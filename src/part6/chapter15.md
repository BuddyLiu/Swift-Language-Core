# 第15章 内存布局与访问控制

在系统编程和性能敏感的应用中，理解内存布局是写出高效代码的关键。Swift 在保持高级语言安全性的同时，提供了底层内存操作的接口，让开发者可以在必要时进行精细控制。本章将深入探讨 Swift 的内存模型、值类型与引用类型的存储差异、独占访问原则，以及指针操作和安全集合的内部策略。

## 15.1 栈与堆：值类型与引用类型的内存本质

### 栈（Stack）

栈是一种后进先出（LIFO）的数据结构，用于存储局部变量和函数调用信息。栈的分配和释放仅通过移动栈指针完成，速度极快。

**栈的特点**：
- 分配速度极快（仅移动指针）
- 内存自动管理（函数返回时自动出栈）
- 大小有限（通常为 8MB，可通过线程配置调整）
- 适合小体积、生命周期确定的数据

```swift
struct Point {
    var x: Double
    var y: Double
}

func draw() {
    let p1 = Point(x: 0, y: 0)  // 栈上分配，8 字节 × 2 = 16 字节
    let p2 = p1                  // 复制到栈上的新位置
    print(p1, p2)
    // 函数返回时自动回收
}
```

### 堆（Heap）

堆用于存储生命周期不确定或大小可变的数据。堆上的分配需要查找空闲内存块，释放需要跟踪，成本远高于栈。

**堆的特点**：
- 分配速度较慢（需要查找空闲块）
- 需要手动或自动释放（ARC 跟踪引用计数）
- 空间大（受限于物理内存和虚拟地址空间）
- 适合大体积、生命周期跨作用域的数据

```swift
class Canvas {
    var points: [Point]
    init() { points = [] }
}

func createCanvas() {
    let canvas = Canvas()  // 堆上分配，栈上持有引用
    canvas.points.append(Point(x: 1, y: 2))
    // 函数返回时，栈上的引用被销毁
    // 如果没有任何其他引用，ARC 释放堆上的 Canvas
}
```

### 内存布局对比

```swift
struct StructPoint {  // 值类型
    var x: Double     // 8 字节
    var y: Double     // 8 字节
}                     // 总计 16 字节，对齐 8 字节

class ClassPoint {    // 引用类型
    var x: Double     // 8 字节
    var y: Double     // 8 字节
}                     // 堆上 16 字节 + 元数据（isa + refCount ≈ 16 字节）
```

引用类型在堆上有额外的元数据开销（isa 指针和引用计数），一共约 16 字节的额外消耗。

### 嵌套类型的内存布局

```swift
struct Line {
    let start: Point  // Point 是结构体，内联存储
    let end: Point    // 两个 Point 直接嵌在 Line 中
}                     // Line 占据 32 字节

struct Shape {
    var type: String  // String 是结构体，但内部持有堆上缓冲区
    var bounds: CGRect // CGRect 是结构体，内联存储
    var points: [Point] // Array 是结构体，内部持有堆上缓冲区
}
```

`String`、`Array` 等看似值类型的结构体，内部可能持有指向堆内存的指针。它们的"值语义"通过写时复制保证。

## 15.2 内存独占访问原则

Swift 引入了一个独特的内存安全机制——独占访问原则（Exclusive Access to Memory）。这个规则防止在同一时间内对同一内存位置进行冲突的读写操作。

### 什么是独占访问

独占访问原则规定：如果一个变量正在被写入，那么在写入完成之前，任何对该变量的其他访问（读或写）都是不允许的。

```swift
var score = 100

// 同时读写同一内存——违反独占访问
let newScore = score + score  // 这是安全的，读操作之间有重叠没问题

// 下面的操作存在问题
func addScore(_ value: inout Int, to other: inout Int) {
    value += other
}

addScore(&score, to: &score)  // 错误：同时以 inout 方式访问 score
```

### 典型冲突场景

**场景一：inout 参数别名**

```swift
var array = [1, 2, 3]

func modifyArray(_ arr: inout [Int], at index: Int, with value: Int) {
    arr[index] = value
}

// 将同一数组的不同元素传入——编译器可能报错
modifyArray(&array, at: 0, with: array[1])
```

**场景二：结构体中的 mutating 方法**

```swift
struct Player {
    var health: Int
    var energy: Int

    mutating func shareHealth(with teammate: inout Player) {
        health += teammate.health
    }
}

var player = Player(health: 100, energy: 50)
player.shareHealth(with: &player)  // 错误：同时访问 player
```

**场景三：闭包捕获与 inout**

```swift
var x = 10
let closure = {
    x += 1  // 闭包捕获了 x
}

func process(_ value: inout Int) {
    value += 1
}

// 在闭包捕获 x 的同时使用 inout 传递 x
process(&x)  // 可能错误：x 被闭包捕获，形成重叠访问
```

### 编译时与运行时检查

编译器可以在编译时检测到部分冲突，对于无法静态分析的情况，Swift 会在运行时插入检查：

```swift
func processSafely(_ value: inout Int) {
    // 编译器插入了运行时检查
    value += 1
}
```

如果冲突在运行时被检测到，程序会崩溃（而非产生不可预期的行为），这正是 Swift"安全第一"哲学的体现。

### 独占访问对性能的影响

独占访问原则允许编译器做出更强的优化假设：

```swift
struct Vector {
    var x: Int, y: Int, z: Int

    mutating func scaleAll(by factor: Int) {
        // 编译器知道没有重叠访问，可以自由优化
        x *= factor
        y *= factor
        z *= factor
    }
}
```

如果没有独占访问保证，编译器在处理结构体修改时必须更保守，每次写入前都要重新读取。

## 15.3 指针：UnsafePointer 族

Swift 在设计上默认屏蔽了指针操作，但在与 C 语言交互或进行底层性能优化时，仍然提供了名为 `Unsafe` 的指针系列。

### UnsafePointer 与 UnsafeMutablePointer

```swift
// 不可变指针
func inspect(ptr: UnsafePointer<Int>) {
    print(ptr.pointee)  // 读取指针指向的值
}

// 可变指针
func mutate(ptr: UnsafeMutablePointer<Int>) {
    ptr.pointee = 42   // 修改指针指向的值
}

var value = 10
mutate(ptr: &value)
print(value)  // 42
```

### 内存分配与释放

```swift
// 手动分配内存
let ptr = UnsafeMutablePointer<Int>.allocate(capacity: 3)

// 初始化
ptr.initialize(to: 0)              // 第一个元素
ptr.advanced(by: 1).initialize(to: 1)
ptr.advanced(by: 2).initialize(to: 2)

// 访问
for i in 0..<3 {
    print(ptr.advanced(by: i).pointee)  // 0, 1, 2
}

// 使用缓冲区语法
let buffer = UnsafeMutableBufferPointer(start: ptr, count: 3)
for (index, value) in buffer.enumerated() {
    print("index \(index): \(value)")
}

// 反初始化并释放
ptr.deinitialize(count: 3)
ptr.deallocate()
```

### UnsafeRawPointer

当不需要类型信息时，可以使用原始指针：

```swift
let data = Data([0x41, 0x42, 0x43])

// 使用原始指针遍历字节
data.withUnsafeBytes { rawPtr in
    guard let base = rawPtr.baseAddress else { return }
    for i in 0..<data.count {
        let byte = base.load(fromByteOffset: i, as: UInt8.self)
        print(String(format: "%02x", byte))
    }
}
```

### 与 C 语言的交互

```swift
import Darwin

// 调用 C 的 strcmp
func compare(_ s1: String, _ s2: String) -> Int32 {
    return s1.withCString { ptr1 in
        s2.withCString { ptr2 in
            return strcmp(ptr1, ptr2)
        }
    }
}
```

### 安全使用指南

**黄金法则**：仅在与 C API 交互或极致性能优化时使用指针，其他情况优先使用 Swift 的安全抽象。

```swift
// 错误：悬垂指针
var ptr: UnsafeMutablePointer<Int>?
do {
    var value = 10
    ptr = withUnsafeMutablePointer(to: &value) { $0 }
}
// 离开作用域后，ptr 指向的内存已无效
// ptr!.pointee  // 未定义行为！
```

```swift
// 正确做法：在 withUnsafePointer 的闭包内完成操作
var value = 10
withUnsafeMutablePointer(to: &value) { ptr in
    ptr.pointee += 1
}
```

## 15.4 Array、Dictionary、Set 的内部策略

Swift 的标准库集合类型在外表现为值类型，内部实现却是复杂的引用计数和写时复制（COW）机制。

### Array 的内部结构

`Array` 的结构体布局可以大致表示为：

```swift
struct Array<Element> {
    var buffer: ContiguousArrayStorage<Element>  // 对堆上缓冲区的引用
}

// 缓冲区包含：
// - isa + refCount（元数据）
// - count（元素数量）
// - capacity（容量）
// - elements（连续存储的元素）
```

存储容量始终 >= 计数，当元素超出容量时，Array 会分配更大的缓冲区并复制元素。

### Array 的写时复制

```swift
var a = [1, 2, 3, 4, 5]
// a 的缓冲区引用计数 = 1

var b = a
// b 共享 a 的缓冲区，引用计数 = 2

b[0] = 100
// 检测到引用计数 > 1，创建新缓冲区
// a 仍然使用原缓冲区：[1, 2, 3, 4, 5]
// b 使用新缓冲区：[100, 2, 3, 4, 5]
```

写时复制将复制的开销从"赋值"推迟到"写入"，大幅提升了性能。

### Array 的容量增长策略

```swift
var numbers = [Int]()
print(numbers.capacity)  // 0

numbers.append(1)
print(numbers.capacity)  // 初始容量通常为 2

numbers.append(2)
print(numbers.capacity)  // 2

numbers.append(3)
print(numbers.capacity)  // 4（翻倍增长）

numbers.append(4)
numbers.append(5)
print(numbers.capacity)  // 8（每次翻倍）
```

每次超出容量时，Array 大约按 2 倍增长，这是时间与空间开销的折中方案。了解这个策略可以帮助你预估性能特征。

### Dictionary 的哈希表结构

`Dictionary` 内部使用开放寻址哈希表（open addressing hash table）：

```swift
var dict = ["name": "Alice", "age": "30"]
```

- 底层是一个连续的存储桶数组（buckets）
- 每个存储桶存储键值对或标记为空
- 通过键的哈希值计算存储位置
- 冲突时通过线性探测解决

```swift
// Dictionary 性能关键因素
// 1. 哈希函数的质量（避免冲突）
// 2. 负载因子（load factor）
// 3. 初始化容量

// 预分配容量可避免频繁 rehash
var dict2 = Dictionary<String, String>(minimumCapacity: 100)
```

### Set 的内部策略

`Set` 的内部结构与 `Dictionary` 几乎相同，也是基于哈希表实现。实际上，在 Swift 源码层面，`Set` 和 `Dictionary` 共享了大量的哈希表代码：

```swift
// Set 和 Dictionary 使用同一个哈希表引擎
var set: Set<Int> = [1, 2, 3, 4, 5]

// Set 也支持预分配容量
var set2 = Set<Int>(minimumCapacity: 100)
```

`Set` 的元素必须遵循 `Hashable` 协议，这在底层决定了哈希表的查找效率。

### 集合类型的性能总结

| 操作 | Array | Dictionary | Set |
|------|-------|------------|-----|
| 读取（索引/键） | O(1) | 平均 O(1) | O(1) |
| 写入（末尾添加） | 平均 O(1) | - | - |
| 插入 | O(n) | 平均 O(1) | 平均 O(1) |
| 删除 | O(n) | 平均 O(1) | 平均 O(1) |
| 查找 | O(n) | - | 平均 O(1) |
| 容量增长 | 翻倍 | rehash | rehash |

### 优化建议

```swift
// 1. 预分配容量
var items: [Int] = []
items.reserveCapacity(1000)
for i in 0..<1000 {
    items.append(i)  // 不会触发多次扩容
}

// 2. 使用 contiguously stored 类型
// Array 的元素是连续存储的，缓存友好

// 3. 大数组使用 lazy 操作链
let result = largeArray.lazy
    .filter { $0 > 100 }
    .map { $0 * 2 }
    .prefix(10)  // 只处理前 10 个符合条件的元素

// 4. 选择正确的集合类型
// 需要唯一性检查 → Set（O(1)）而非 Array（O(n)）
// 需要键值映射 → Dictionary
// 需要有序索引 → Array
```

## 本章小结

Swift 的内存模型在安全与性能之间取得了精妙的平衡。栈与堆的区分决定了值类型与引用类型的性能特征；独占访问原则在编译期和运行时保障数据一致性；UnsafePointer 系列为底层操作提供了接口；而 Array、Dictionary、Set 等集合类型通过写时复制等机制在值语义下实现接近引用类型的高性能。理解这些底层机制，能够帮助开发者在日常编码中做出更明智的设计决策。
