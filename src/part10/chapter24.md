# 第24章 布局与容器视图

## 24.1 栈：HStack、VStack、ZStack

SwiftUI 提供了三种核心栈容器视图，用于在水平和垂直方向以及深度方向上排列子视图。

### VStack：垂直排列

`VStack` 将子视图沿垂直方向（从上到下）依次排列：

```swift
VStack(alignment: .leading, spacing: 12) {
    Text("第一行")
    Text("第二行")
    Text("第三行")
}
```

- `alignment`：子视图的水平对齐方式，默认 `.center`。可选 `.leading`、`.trailing` 等。
- `spacing`：子视图之间的间距，默认值由系统决定。

### HStack：水平排列

`HStack` 将子视图沿水平方向（从左到右）依次排列：

```swift
HStack(alignment: .top, spacing: 8) {
    Image(systemName: "star")
    Text("收藏")
    Spacer()
}
```

- `alignment`：子视图的垂直对齐方式，默认 `.center`。可选 `.top`、`.bottom`、`.firstTextBaseline` 等。
- `spacing`：子视图之间的间距。

### ZStack：深度排列

`ZStack` 在深度轴（Z 轴）上叠加子视图，先添加的视图在底层，后添加的在上层：

```swift
ZStack(alignment: .bottomTrailing) {
    Image("photo")
        .resizable()
        .scaledToFill()
    Text("© 2025")
        .padding(4)
        .background(Color.black.opacity(0.6))
        .foregroundColor(.white)
}
```

- `alignment`：所有子视图的对齐参考点，默认 `.center`。
- ZStack 常用于背景叠加、遮罩、角标等场景。

### 布局规则

栈容器的布局遵循三步过程：

1. **提案 (Proposal)**：父视图向栈容器提供一个建议尺寸。
2. **测量 (Measuring)**：栈容器根据自身的排列方向、间距和对齐方式，向每个子视图依次提议尺寸，收集它们的需求。
3. **分配 (Assignment)**：栈容器根据子视图的弹性 (flexibility) 和优先级，将可用空间分配给各个子视图，确定每个子视图的最终位置。

这个过程可以被 `fixedSize()`、`layoutPriority()` 等修饰符干预。

## 24.2 弹性空间：Spacer 与对齐方式

### Spacer：占据弹性空间

`Spacer` 是一个透明视图，它会尽可能多地占据可用空间，将周围的视图推开：

```swift
HStack {
    Text("左侧")
    Spacer()           // 占据中间所有弹性空间
    Text("右侧")
}
```

`Spacer` 也可以设置最小长度：

```swift
HStack {
    Text("左侧")
    Spacer(minLength: 20)  // 至少保留 20pt 空间
    Text("右侧")
}
```

多个 `Spacer` 会**均分**弹性空间：

```swift
HStack {
    Spacer()
    Text("中")
    Spacer()
    Text("右")
    Spacer()
}
// 结果：Text 均匀分布，形成类似 equal spacing 的效果
```

### divider：视觉分隔

`Divider()` 绘制一条分割线。在 `VStack` 中默认为水平线；在 `HStack` 中默认为垂直线：

```swift
VStack {
    Text("上")
    Divider()
    Text("下")
}
```

### 对齐方式进阶

栈容器支持精细的对齐控制，尤其是 `VStack` 配合 `alignment`：

```swift
VStack(alignment: .leading) {
    Text("短文本")
    Text("这是一段比较长的文本内容")
        .alignmentGuide(.leading) { d in
            d[.leading] + 20  // 自定义缩进
        }
}
```

`alignmentGuide` 修饰符允许我们覆盖默认的对齐行为，返回自定义的偏移量。这在实现复杂布局时非常有用。

## 24.3 滚动与懒加载

### ScrollView：通用滚动容器

`ScrollView` 允许内容在超出屏幕范围时滚动：

```swift
ScrollView(.vertical, showsIndicators: true) {
    VStack(spacing: 20) {
        ForEach(0..<100) { i in
            Text("Item \(i)")
                .frame(maxWidth: .infinity)
                .padding()
                .background(Color.blue.opacity(0.1))
        }
    }
    .padding()
}
```

- 第一个参数 `axis`：滚动方向，`.horizontal` 或 `.vertical`（默认）。
- `showsIndicators`：是否显示滚动指示器。

⚠️ **注意**：`ScrollView` 会一次性加载所有子视图。对于长列表，应使用懒加载容器。

### LazyVStack / LazyHStack：懒加载栈

`LazyVStack` 和 `LazyHStack` 与普通栈类似，但子视图**只在即将出现在屏幕上时才被创建**：

```swift
ScrollView {
    LazyVStack(spacing: 10) {
        ForEach(0..<10000) { i in
            Text("Item \(i)")
                .padding()
                .id(i)  // 提供稳定标识
        }
    }
}
```

关键区别：
- `VStack` 立即创建所有子视图。
- `LazyVStack` 根据需要动态创建子视图，大幅提升滚动性能。
- 懒加载栈**必须**放在 `ScrollView` 内才能正常工作。

### List：列表视图

`List` 是 SwiftUI 中最常用的滚动列表容器，在 iOS 上渲染为 `UITableView` 的等价物：

```swift
List {
    Section("收藏夹") {
        Text("书签 1")
        Text("书签 2")
    }
    Section("最近浏览") {
        ForEach(recentItems) { item in
            Text(item.title)
        }
    }
}
```

List 的特点：
- 原生支持选择、滑动删除、重排等操作。
- 在 iOS 上自动适配分组样式和分隔线。
- 支持 `.listStyle()` 修饰符改变样式（如 `.insetGrouped`、`.plain`）。

## 24.4 表单与分组

### Form：表单容器

`Form` 是专门为数据输入设计的容器，在 iOS 上呈现为分组列表样式：

```swift
Form {
    Section("个人信息") {
        TextField("姓名", text: $name)
        TextField("邮箱", text: $email)
    }

    Section("偏好设置") {
        Toggle("接收通知", isOn: $notificationsEnabled)
        Stepper("提醒频率：\(frequency) 分钟", value: $frequency, in: 1...60)
    }

    Section {
        Button("保存") { save() }
            .disabled(!isFormValid)
    }
}
```

`Form` 自动适配各平台的样式：
- iOS：分组样式，类似系统设置。
- macOS：表单样式。
- watchOS：滚动列表样式。

### Group：逻辑分组

`Group` 是一个**透明容器**，它不改变布局，但允许你将多个视图视为一个整体：

```swift
Group {
    Text("行 1")
    Text("行 2")
    Text("行 3")
}
.padding()
.background(Color.gray.opacity(0.1))
```

`Group` 的典型用途：
1. 突破 @ViewBuilder 的 10 个子视图限制。
2. 为一组视图统一应用修饰符。
3. 在条件语句中返回多个视图。

### Section：语义分组

`Section` 用于在 `List` 或 `Form` 中创建有标题和脚注的分组：

```swift
List {
    Section {
        Text("内容")
    } header: {
        Label("标题", systemImage: "star")
    } footer: {
        Text("这是脚注说明文字")
            .font(.caption)
    }
}
```

## 24.5 几何信息：GeometryReader 与 Layout 协议

### GeometryReader：读取几何信息

`GeometryReader` 是一个容器视图，它提供一个 `GeometryProxy` 对象，用于获取父视图的尺寸和安全区域：

```swift
GeometryReader { proxy in
    VStack {
        Text("容器宽度：\(proxy.size.width)")
        Text("容器高度：\(proxy.size.height)")
        Text("安全区域上边距：\(proxy.safeAreaInsets.top)")
    }
}
```

常见应用场景：

**1. 百分比布局：**
```swift
GeometryReader { proxy in
    Rectangle()
        .fill(Color.blue)
        .frame(width: proxy.size.width * 0.5,
               height: proxy.size.height * 0.3)
}
```

**2. 滚动偏移监听：**
```swift
ScrollView {
    GeometryReader { proxy in
        Color.clear
            .onAppear {
                let offsetY = proxy.frame(in: .global).minY
                print("当前偏移：\(offsetY)")
            }
    }
    .frame(height: 0)  // 隐藏几何读取器
}
```

⚠️ **使用建议**：`GeometryReader` 会占据尽可能多的空间，并且在几何信息变化时会触发视图刷新。过度使用可能导致性能问题，应谨慎使用。

### Layout 协议：自定义布局

iOS 16+ / macOS 13+ 引入了 `Layout` 协议，允许开发者完全自定义布局算法：

```swift
struct CircleLayout: Layout {
    let radius: Double

    func sizeThatFits(
        proposal: ProposedViewSize,
        subviews: Subviews,
        cache: inout ()
    ) -> CGSize {
        let diameter = radius * 2
        return CGSize(width: diameter, height: diameter)
    }

    func placeSubviews(
        in bounds: CGRect,
        proposal: ProposedViewSize,
        subviews: Subviews,
        cache: inout ()
    ) {
        let center = CGPoint(x: bounds.midX, y: bounds.midY)
        let count = subviews.count
        for (index, subview) in subviews.enumerated() {
            let angle = (2 * .pi / Double(count)) * Double(index)
            let x = center.x + radius * cos(angle)
            let y = center.y + radius * sin(angle)
            subview.place(
                at: CGPoint(x: x, y: y),
                anchor: .center,
                proposal: .unspecified
            )
        }
    }
}

// 使用
CircleLayout(radius: 100) {
    ForEach(0..<6) { i in
        Text("\(i)")
            .frame(width: 40, height: 40)
            .background(Color.blue)
            .clipShape(Circle())
    }
}
```

`Layout` 协议的核心方法：

1. **`sizeThatFits`**：计算容器在给定提议尺寸下的理想大小。
2. **`placeSubviews`**：将每个子视图放置在计算的位置。
3. 可选方法：`makeCache`、`updateCache` 用于缓存中间计算结果以优化性能。

## 总结

本章详细介绍了 SwiftUI 的布局体系。我们从 HStack、VStack、ZStack 三大栈容器入手，理解了布局的三步提交流程；掌握了 Spacer、对齐方式和 alignmentGuide 的灵活运用；学习了 ScrollView、LazyVStack/LazyHStack 和 List 的滚动与懒加载机制；以及 Form、Group、Section 在表单和分组布局中的角色；最后探讨了 GeometryReader 的几何信息读取和 Layout 协议的自定义布局能力。布局是 SwiftUI 应用的基础骨架，掌握好这些容器视图将让你能够搭建任何复杂的用户界面。
