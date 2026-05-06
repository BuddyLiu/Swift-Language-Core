# 第3章 初始化与解构

## 3.1 初始化器与初始化器委托

在 Swift 中，初始化器（Initializer）是创建类或结构体实例的入口。它的核心职责是**确保所有存储属性在被使用前都拥有初始值**。Swift 的初始化系统以严格和完整著称——编译器会检查每个属性是否都被正确初始化。

### 基础初始化器

结构体默认会获得一个**成员逐一初始化器**（Memberwise Initializer），但类没有：

```swift
struct Point {
    var x: Double
    var y: Double
}

// 结构体自动获得成员逐一初始化器
let p = Point(x: 10, y: 20)

class PointClass {
    var x: Double
    var y: Double
    // 类没有默认的成员逐一初始化器，必须自定义
    init(x: Double, y: Double) {
        self.x = x
        self.y = y
    }
}
```

### Designated 初始化器

**指定初始化器**（Designated Initializer）是类的主要初始化器，负责初始化当前类的所有属性，并调用父类的初始化器。一个类至少需要有一个指定初始化器。

```swift
class Vehicle {
    var wheels: Int
    // 指定初始化器
    init(wheels: Int) {
        self.wheels = wheels
    }
}

class Car: Vehicle {
    var brand: String
    // 指定初始化器
    init(brand: String, wheels: Int) {
        self.brand = brand         // 第一步：初始化自身属性
        super.init(wheels: wheels) // 第二步：调用父类初始化器
    }
}
```

### Convenience 初始化器

**便利初始化器**（Convenience Initializer）是辅助性质的初始化器。它必须**委托到同一类中的另一个初始化器**（最终委托到一个指定初始化器），不能直接调用父类的 `init`。

```swift
class Car: Vehicle {
    var brand: String
    
    init(brand: String, wheels: Int) {
        self.brand = brand
        super.init(wheels: wheels)
    }
    
    // 便利初始化器：为品牌提供默认值
    convenience init(brand: String) {
        self.init(brand: brand, wheels: 4)  // 委托到指定初始化器
    }
    
    // 便利初始化器：全部使用默认值
    convenience init() {
        self.init(brand: "Unknown")  // 委托到另一个便利初始化器
    }
}

let car1 = Car(brand: "Tesla", wheels: 4)
let car2 = Car(brand: "BMW")       // 使用便利初始化器，wheels 默认为 4
let car3 = Car()                   // 全部默认
```

### 初始化器委托规则

Swift 为初始化器委托设定了三条规则，确保初始化过程安全可靠：

1. **指定初始化器必须调用父类的指定初始化器**
2. **便利初始化器必须调用同类中的另一个初始化器**
3. **便利初始化器最终必须调用到指定初始化器**

```
指定初始化器 → 父类的指定初始化器
便利初始化器 → 同类中的指定初始化器（或另一个便利初始化器）
```

这形成了一条清晰的委托链（Initializer Delegation Chain），确保了初始化过程的完整性和安全性。

## 3.2 可失败初始化器

有些初始化过程可能失败——比如从字符串解析数字、加载不存在的文件、或验证输入的有效性。Swift 通过**可失败初始化器**（Failable Initializer）优雅地处理这种情况。

```swift
struct Temperature {
    var celsius: Double
    
    // 可失败初始化器：检查温度不能低于绝对零度
    init?(celsius: Double) {
        guard celsius >= -273.15 else {
            return nil  // 初始化失败，返回 nil
        }
        self.celsius = celsius
    }
}

let temp1 = Temperature(celsius: 100)     // 成功：Optional(Temperature)
let temp2 = Temperature(celsius: -300)    // 失败：nil
```

可失败初始化器返回的是**可选类型**（`Optional<Self>`），而非直接返回实例。

### 枚举的可失败初始化器

可失败初始化器在枚举中非常常见，尤其是在需要从原始值创建枚举时：

```swift
enum HTTPStatus: Int {
    case ok = 200
    case notFound = 404
    case serverError = 500
    
    // 如果原始值不匹配任何 case，初始化失败
    init?(code: Int) {
        self.init(rawValue: code)  // 使用默认的 rawValue 初始化器
    }
}

let status1 = HTTPStatus(code: 200)  // Optional(.ok)
let status2 = HTTPStatus(code: 999)  // nil
```

### init! — 隐式解包的可失败初始化器

除了 `init?`，Swift 还提供了 `init!`，创建一个**隐式解包可选类型**的实例：

```swift
class Document {
    var name: String?
    init!(name: String) {
        if name.isEmpty { return nil }  // 返回 nil 但类型是 Document!
        self.name = name
    }
}

let doc: Document = Document(name: "report")  // 可当作非可选使用
// 但如果为 nil，使用 doc 时会导致崩溃
```

一般建议优先使用 `init?`，除非你有非常明确的理由需要使用 `init!`。

## 3.3 必需初始化器

**必需初始化器**（Required Initializer）强制所有子类必须实现该初始化器。这在需要确保多态初始化行为时非常有用。

```swift
class Animal {
    var name: String
    
    required init(name: String) {
        self.name = name
    }
}

class Dog: Animal {
    var breed: String
    
    required init(name: String) {
        self.breed = "Unknown"
        super.init(name: name)
    }
}

let dog = Dog(name: "Buddy")
print(dog.name)   // "Buddy"
```

### 必需初始化器的应用场景

1. **协议要求**：当协议声明了 `init` 要求时，实现必须使用 `required` 关键字
2. **工厂模式**：需要确保所有子类都能通过统一的 `init` 创建
3. **框架设计**：框架可能要求子类实现特定的初始化路径

```swift
protocol Identifiable {
    init(id: String)
}

class User: Identifiable {
    var id: String
    required init(id: String) {
        self.id = id
    }
}

// 即使在子类中，也强制实现该初始化器
class AdminUser: User {
    var role: String
    required init(id: String) {
        self.role = "admin"
        super.init(id: id)
    }
}
```

### 子类可以不实现 required init 吗？

如果子类没有任何其他指定初始化器，且继承的 `required init` 能满足需求，可以省略。但一旦子类添加了自定义的指定初始化器，就必须同时实现 `required init`。

## 3.4 反初始化器 deinit

**反初始化器**（Deinitializer）是类实例在销毁前执行的清理方法。结构体和枚举没有反初始化器——只有类才有。这是因为只有类（引用类型）拥有可管理的生命周期。

```swift
class FileHandler {
    var filename: String
    
    init(filename: String) {
        self.filename = filename
        print("打开文件：\(filename)")
    }
    
    deinit {
        // 关闭文件、释放资源等清理工作
        print("关闭文件：\(filename)")
    }
}

// 使用
var handler: FileHandler? = FileHandler(filename: "data.txt")
handler = nil  // "关闭文件：data.txt" — 实例被销毁
```

### deinit 的特点

1. **自动调用**：实例被销毁前自动调用，**不能手动调用**
2. **没有参数**：`deinit` 不接受参数，也不加括号
3. **继承**：父类的 `deinit` 会在子类的 `deinit` 之后自动调用
4. **顺序保证**：子类先执行 `deinit`，再执行父类的 `deinit`

### 常见用途

```swift
class NetworkConnection {
    let url: String
    var isConnected = false
    
    init(url: String) {
        self.url = url
        connect()
    }
    
    func connect() {
        isConnected = true
        print("已连接：\(url)")
    }
    
    func disconnect() {
        isConnected = false
        print("已断开：\(url)")
    }
    
    deinit {
        disconnect()  // 确保断开连接
        print("NetworkConnection 被销毁")
    }
}
```

## 3.5 两阶段初始化与内存安全

两阶段初始化（Two-Phase Initialization）是 Swift 初始化系统的核心安全机制。它确保所有属性在被访问前都已正确初始化，尤其在类继承的场景下。

### 第一阶段：初始化自身属性

1. 子类的指定初始化器初始化自己引入的所有存储属性
2. 调用父类的指定初始化器，让父类完成它自己属性的初始化
3. 沿着继承链向上，直到根类
4. 到达根类后，根类完成自身的初始化

### 第二阶段：自定义操作

1. 从根类返回，沿着继承链向下，每个类可以访问 `self` 并修改属性
2. 调用实例方法、读取属性值等

### 安全检查

Swift 编译器会执行四道安全检查，确保两阶段初始化的正确性：

**检查 1**：子类的指定初始化器在调用父类初始化器之前，必须完成自身所有属性的初始化。

```swift
class Vehicle {
    var wheels: Int
    init(wheels: Int) { self.wheels = wheels }
}

class Car: Vehicle {
    var brand: String
    init(brand: String, wheels: Int) {
        // ✅ 先初始化自己的属性
        self.brand = brand
        // 然后才能调用父类初始化器
        super.init(wheels: wheels)
    }
}
```

**检查 2**：在调用父类初始化器之前，不能访问 `self` 的属性和方法。

```swift
class Car: Vehicle {
    var brand: String
    init(brand: String, wheels: Int) {
        // ❌ 错误：不能先使用 self
        // print(self.brand)
        
        self.brand = brand
        super.init(wheels: wheels)
        
        // ✅ 在调用 super.init 之后才能访问 self
        print(self.brand)
    }
}
```

**检查 3**：便利初始化器在委托到其他初始化器之前，不能给任何属性赋值。

```swift
class Car: Vehicle {
    var brand: String
    init(brand: String, wheels: Int) {
        self.brand = brand
        super.init(wheels: wheels)
    }
    
    convenience init() {
        // ❌ 错误：便利初始化器在委托前不能设置属性
        // self.brand = "Unknown"
        self.init(brand: "Unknown", wheels: 4)
    }
}
```

**检查 4**：第一阶段完成后（即父类的 `init` 返回后），才能读取属性和调用方法。

```swift
class Car: Vehicle {
    var brand: String
    init(brand: String, wheels: Int) {
        self.brand = brand
        // 第一阶段：调用 super.init 之前
        // ❌ 不能读取属性
        // print(self.wheels) — 错误
        
        super.init(wheels: wheels)
        
        // 第二阶段：父类初始化完成
        // ✅ 可以安全访问
        print(self.wheels)
        print(self.brand)
    }
}
```

### 为什么需要两阶段初始化？

考虑一个继承链中的场景：如果子类可以在父类初始化完成之前访问父类的属性，那么子类可能读取到未初始化的内存，导致灾难性后果。

```swift
class Rectangle {
    var width: Double
    var height: Double
    
    init(width: Double, height: Double) {
        self.width = width
        self.height = height
    }
    
    var area: Double {
        return width * height
    }
}

class Square: Rectangle {
    var isRegular: Bool
    
    init(side: Double) {
        self.isRegular = true
        super.init(width: side, height: side)
        
        // 现在可以安全使用 area
        print("面积：\(self.area)")
    }
}
```

如果没有两阶段初始化的保护，`Square` 可能在 `width` 和 `height` 尚未设置时就调用 `self.area`——那将是一场灾难。

### 内存安全与初始化

Swift 的初始化系统与 ARC（自动引用计数）协同工作，确保内存管理在初始化过程中也是安全的：

1. **所有存储属性都有确定的值**：没有未初始化内存
2. **引用计数在初始化完成后正确建立**：避免了过早释放或引用循环
3. **常量属性（let）可以在初始化器中只赋值一次**：即使在类中，`let` 属性也可以在 `init` 中赋值

```swift
class View {
    let identifier: String
    let createdAt: Date
    
    init(identifier: String) {
        // 即使是 let 常量，也可以在 init 中设置
        self.identifier = identifier
        self.createdAt = Date()  // 创建时的时间戳
        // 之后不能再修改
    }
}
```

### 小结

本章深入探讨了 Swift 的初始化系统。指定初始化器与便利初始化器的委托规则确保了初始化流程的清晰；可失败初始化器优雅地处理了初始化中的失败场景；必需初始化器强制执行特定的初始化契约；反初始化器为资源清理提供了自然的接入点。而两阶段初始化机制则是这一切的安全基石，它确保在你访问任何属性之前，所有内存都已被正确初始化。

这些设计共同构成了 Swift 初始化系统的核心——**安全、完整、可预测**。
