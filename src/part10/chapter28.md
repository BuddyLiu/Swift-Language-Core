# 第28章 自定义视图与绘图

SwiftUI 提供了丰富的内置视图，但真正的灵活性体现在自定义视图和绘图能力上。无论是通过 `Shape` 协议创建自定义图形，使用 `Canvas` 进行高性能绘制，还是通过自定义 `Layout` 协议实现复杂布局，SwiftUI 都为开发者提供了强大的底层工具。本章将深入探讨这些进阶技术。

## 28.1 Shape 协议与 Path

`Shape` 协议是所有可绘制形状的基础。通过实现 `path(in:)` 方法，我们可以创建任意形状。

### Shape 协议基础

```swift
struct Diamond: Shape {
    func path(in rect: CGRect) -> Path {
        var path = Path()

        let center = CGPoint(x: rect.midX, y: rect.midY)

        path.move(to: CGPoint(x: center.x, y: rect.minY))
        path.addLine(to: CGPoint(x: rect.maxX, y: center.y))
        path.addLine(to: CGPoint(x: center.x, y: rect.maxY))
        path.addLine(to: CGPoint(x: rect.minX, y: center.y))
        path.closeSubpath()

        return path
    }
}

// 使用
struct ContentView: View {
    var body: some View {
        Diamond()
            .fill(.blue)
            .frame(width: 200, height: 200)
    }
}
```

### Path 的绘制命令

`Path` 提供了一系列绘制命令，类似于 Core Graphics 的 API：

```swift
struct CustomPathShape: Shape {
    func path(in rect: CGRect) -> Path {
        Path { path in
            // 移动起点
            path.move(to: CGPoint(x: rect.minX, y: rect.midY))

            // 直线
            path.addLine(to: CGPoint(x: rect.midX, y: rect.minY))

            // 弧线
            path.addArc(
                center: CGPoint(x: rect.midX, y: rect.midY),
                radius: 50,
                startAngle: .degrees(0),
                endAngle: .degrees(180),
                clockwise: false
            )

            // 二次贝塞尔曲线
            path.addQuadCurve(
                to: CGPoint(x: rect.maxX, y: rect.midY),
                control: CGPoint(x: rect.midX, y: rect.maxY)
            )

            // 三次贝塞尔曲线
            path.addCurve(
                to: CGPoint(x: rect.midX, y: rect.maxY),
                control1: CGPoint(x: rect.maxX * 0.75, y: rect.midY),
                control2: CGPoint(x: rect.maxX * 0.75, y: rect.maxY)
            )

            path.closeSubpath()
        }
    }
}
```

### 带参数的可动画形状

形状可以通过 `Animatable` 协议实现动画：

```swift
struct AnimatablePolygon: Shape {
    var sides: Double

    var animatableData: Double {
        get { sides }
        set { sides = newValue }
    }

    func path(in rect: CGRect) -> Path {
        let radius = min(rect.width, rect.height) / 2
        let center = CGPoint(x: rect.midX, y: rect.midY)
        let angleIncrement = (2 * .pi) / sides

        var path = Path()

        for i in 0..<Int(sides) {
            let angle = angleIncrement * Double(i) - (.pi / 2)
            let point = CGPoint(
                x: center.x + radius * cos(angle),
                y: center.y + radius * sin(angle)
            )

            if i == 0 {
                path.move(to: point)
            } else {
                path.addLine(to: point)
            }
        }
        path.closeSubpath()
        return path
    }
}

// 使用
struct PolygonView: View {
    @State private var sides = 3.0

    var body: some View {
        VStack {
            AnimatablePolygon(sides: sides)
                .stroke(.blue, lineWidth: 2)
                .frame(width: 200, height: 200)

            Slider(value: $sides, in: 3...12, step: 1)
                .padding()
        }
        .animation(.spring(), value: sides)
    }
}
```

### 使用 Shape 进行裁剪和遮罩

Shape 不仅可绘制，还可作为裁剪和遮罩的工具：

```swift
Image("photo")
    .resizable()
    .frame(width: 200, height: 200)
    .clipShape(Circle())

// 自定义裁剪
Image("photo")
    .resizable()
    .frame(width: 200, height: 200)
    .clipShape(Diamond())

// 使用形状作为遮罩
Color.blue
    .frame(width: 200, height: 200)
    .mask {
        Text("SwiftUI")
            .font(.system(size: 60, weight: .bold))
    }
```

## 28.2 Canvas 与图形上下文

`Canvas` 是 SwiftUI 中直接操作图形上下文的视图，适用于高性能绘图场景。

### 基本用法

```swift
struct CanvasExample: View {
    var body: some View {
        Canvas { context, size in
            // 绘制矩形
            context.fill(
                Path(CGRect(x: 0, y: 0, width: 100, height: 100)),
                with: .color(.blue)
            )

            // 绘制椭圆
            context.fill(
                Path(ellipseIn: CGRect(x: 50, y: 50, width: 150, height: 100)),
                with: .color(.green)
            )

            // 绘制文本
            context.draw(
                Text("Hello Canvas")
                    .font(.title)
                    .foregroundColor(.white),
                at: CGPoint(x: size.width / 2, y: size.height / 2)
            )
        }
        .frame(width: 300, height: 300)
    }
}
```

### 图形上下文的高级操作

```swift
Canvas { context, size in
    // 保存上下文状态
    context.saveGraphicsState()

    // 平移
    context.translateBy(x: 50, y: 50)
    // 旋转
    context.rotate(by: .degrees(45))
    // 缩放
    context.scaleBy(x: 1.5, y: 1.5)

    // 设置不透明度
    context.opacity = 0.8

    // 绘制带阴影
    context.fill(
        Path(ellipseIn: CGRect(x: 0, y: 0, width: 100, height: 100)),
        with: .color(.purple)
    )

    // 恢复上下文状态
    context.restoreGraphicsState()

    // 使用 blend mode
    context.blendMode = .multiply
}
.frame(width: 300, height: 300)
```

### 性能优化

Canvas 适合绘制大量图形元素，比使用单个视图叠加更高效：

```swift
struct ParticleCanvas: View {
    let particles: [Particle]

    var body: some View {
        Canvas { context, size in
            for particle in particles {
                let rect = CGRect(
                    x: particle.x - 2,
                    y: particle.y - 2,
                    width: 4,
                    height: 4
                )
                context.fill(
                    Path(ellipseIn: rect),
                    with: .color(.blue.opacity(particle.opacity))
                )
            }
        }
    }
}
```

### 与 TimelineView 结合实现动画

```swift
struct AnimatedCanvas: View {
    var body: some View {
        TimelineView(.animation) { timeline in
            Canvas { context, size in
                let time = timeline.date.timeIntervalSince1970
                let phase = cos(time * 2)

                for i in 0..<10 {
                    let x = size.width / 2 + CGFloat(i - 5) * 30
                    let y = size.height / 2 + CGFloat(phase * 50)
                    let rect = CGRect(x: x - 15, y: y - 15, width: 30, height: 30)

                    context.fill(
                        Path(ellipseIn: rect),
                        with: .color(.blue.opacity(0.5 - Double(i) * 0.05))
                    )
                }
            }
        }
        .frame(width: 300, height: 300)
    }
}
```

## 28.3 进阶布局：自定义 Layout 协议

`Layout` 协议（iOS 16+）是 SwiftUI 布局系统的最高抽象，允许开发者实现自定义的容器布局算法。

### Layout 协议

```swift
struct CustomLayout: Layout {
    // 计算布局大小
    func sizeThatFits(
        proposal: ProposedViewSize,
        subviews: Subviews,
        cache: inout ()
    ) -> CGSize {
        // 根据子视图计算容器大小
        let maxWidth = proposal.width ?? .infinity
        var height: CGFloat = 0

        for subview in subviews {
            let size = subview.sizeThatFits(.unspecified)
            height += size.height
        }

        return CGSize(width: maxWidth, height: height)
    }

    // 放置子视图
    func placeSubviews(
        in bounds: CGRect,
        proposal: ProposedViewSize,
        subviews: Subviews,
        cache: inout ()
    ) {
        var y = bounds.minY

        for subview in subviews {
            let size = subview.sizeThatFits(.unspecified)
            subview.place(
                at: CGPoint(x: bounds.midX, y: y + size.height / 2),
                anchor: .center,
                proposal: ProposedViewSize(size)
            )
            y += size.height
        }
    }
}
```

### 实现流式布局（Flow Layout）

流式布局是自定义 Layout 的典型应用——子视图从左到右排列，放不下时换行：

```swift
struct FlowLayout: Layout {
    var spacing: CGFloat = 8
    var lineSpacing: CGFloat = 8

    struct Cache {
        var sizes: [CGSize] = []
    }

    func makeCache(subviews: Subviews) -> Cache {
        return Cache(sizes: subviews.map { $0.sizeThatFits(.unspecified) })
    }

    func sizeThatFits(
        proposal: ProposedViewSize,
        subviews: Subviews,
        cache: inout Cache
    ) -> CGSize {
        let maxWidth = proposal.width ?? .infinity
        var width: CGFloat = 0
        var height: CGFloat = 0
        var currentX: CGFloat = 0
        var currentY: CGFloat = 0
        var maxLineHeight: CGFloat = 0

        for (index, subview) in subviews.enumerated() {
            let size = cache.sizes[safe: index] ?? subview.sizeThatFits(.unspecified)

            if currentX + size.width > maxWidth {
                // 换行
                currentX = 0
                currentY += maxLineHeight + lineSpacing
                maxLineHeight = 0
            }

            currentX += size.width + spacing
            maxLineHeight = max(maxLineHeight, size.height)
            width = max(width, currentX)
            height = currentY + maxLineHeight
        }

        return CGSize(width: width, height: height)
    }

    func placeSubviews(
        in bounds: CGRect,
        proposal: ProposedViewSize,
        subviews: Subviews,
        cache: inout Cache
    ) {
        let maxWidth = bounds.width
        var currentX = bounds.minX
        var currentY = bounds.minY
        var maxLineHeight: CGFloat = 0

        for (index, subview) in subviews.enumerated() {
            let size = cache.sizes[safe: index] ?? subview.sizeThatFits(.unspecified)

            if currentX + size.width > maxWidth + bounds.minX {
                currentX = bounds.minX
                currentY += maxLineHeight + lineSpacing
                maxLineHeight = 0
            }

            subview.place(
                at: CGPoint(x: currentX + size.width / 2, y: currentY + size.height / 2),
                anchor: .center,
                proposal: ProposedViewSize(size)
            )

            currentX += size.width + spacing
            maxLineHeight = max(maxLineHeight, size.height)
        }
    }
}

// 使用
FlowLayout(spacing: 10, lineSpacing: 15) {
    ForEach(tags, id: \.self) { tag in
        Text(tag)
            .padding(.horizontal, 12)
            .padding(.vertical, 6)
            .background(.blue.opacity(0.1))
            .clipShape(Capsule())
    }
}
```

### Cache 机制

`Layout` 协议内置了缓存机制，避免重复计算子视图尺寸：

```swift
func updateCache(_ cache: inout Cache, subviews: Subviews) {
    cache.sizes = subviews.map { $0.sizeThatFits(.unspecified) }
}
```

缓存会在子视图发生变化时自动更新，是优化布局性能的关键。

## 28.4 视图首选项：PreferenceKey

`PreferenceKey` 是 SwiftUI 中向上传递数据的机制——子视图向父视图传递信息。

### PreferenceKey 协议

```swift
struct WidthPreferenceKey: PreferenceKey {
    static let defaultValue: CGFloat = 0

    static func reduce(value: inout CGFloat, nextValue: () -> CGFloat) {
        value = max(value, nextValue())
    }
}

// 子视图写入首选项
struct ChildView: View {
    var body: some View {
        Text("测量我的宽度")
            .background(
                GeometryReader { proxy in
                    Color.clear
                        .preference(key: WidthPreferenceKey.self,
                                   value: proxy.size.width)
                }
            )
    }
}

// 父视图读取首选项
struct ParentView: View {
    @State private var childWidth: CGFloat = 0

    var body: some View {
        VStack {
            ChildView()
                .onPreferenceChange(WidthPreferenceKey.self) { value in
                    childWidth = value
                }

            Text("子视图宽度: \(childWidth)")
        }
    }
}
```

### 多子视图的 Preference 合并

当多个子视图设置相同的 PreferenceKey 时，`reduce` 方法决定如何合并：

```swift
struct MaxWidthPreferenceKey: PreferenceKey {
    static let defaultValue: CGFloat = 0

    static func reduce(value: inout CGFloat, nextValue: () -> CGFloat) {
        // 取所有子视图宽度的最大值
        value = max(value, nextValue())
    }
}

// 多个子视图分别上报
HStack {
    ForEach(items) { item in
    ItemView(item: item)
        .background(
            GeometryReader { proxy in
                Color.clear
                    .preference(key: MaxWidthPreferenceKey.self,
                               value: proxy.size.width)
            }
        )
    }
}
.onPreferenceChange(MaxWidthPreferenceKey.self) { maxWidth in
    print("最宽的子视图宽度: \(maxWidth)")
}
```

### 典型应用：实现等宽子视图

PreferenceKey 最常见的应用场景之一是让多个子视图保持等宽：

```swift
struct EqualWidthHStack<Content: View>: View {
    let content: Content
    @State private var maxWidth: CGFloat = 0

    init(@ViewBuilder content: () -> Content) {
        self.content = content()
    }

    var body: some View {
        HStack(spacing: 8) {
            content
                .background(
                    GeometryReader { proxy in
                        Color.clear
                            .preference(key: MaxWidthPreferenceKey.self,
                                       value: proxy.size.width)
                    }
                )
        }
        .onPreferenceChange(MaxWidthPreferenceKey.self) { width in
            maxWidth = width
        }
        .onPreferenceChange(MaxWidthPreferenceKey.self) { width in
            forEachSubView {
                $0.frame(width: maxWidth)
            }
        }
    }
}
```

### AnchorPreferenceKey

对于需要传递视图几何位置信息的场景，使用 `AnchorPreferenceKey`：

```swift
struct CenterAnchorPreference: PreferenceKey {
    static let defaultValue: [Int: Anchor<CGPoint>] = [:]
    static func reduce(value: inout [Int: Anchor<CGPoint>],
                      nextValue: () -> [Int: Anchor<CGPoint>]) {
        value.merge(nextValue()) { $1 }
    }
}

// 在子视图中
.anchorPreference(key: CenterAnchorPreference.self,
                 value: .center) { [item.id: $0] }

// 在父视图中通过 GeometryReader 使用锚点
.backgroundPreferenceValue(CenterAnchorPreference.self) { anchors in
    GeometryReader { proxy in
        ForEach(Array(anchors.keys), id: \.self) { id in
            if let anchor = anchors[id] {
                let point = proxy[anchor]
                Circle()
                    .frame(width: 8, height: 8)
                    .position(point)
            }
        }
    }
}
```

## 本章小结

本章深入探讨了 SwiftUI 自定义视图与绘制的四大核心技术。`Shape` 协议提供了创建任意几何形状的能力，Path 的绘制命令集可以构建复杂图形。`Canvas` 视图直接操作图形上下文，适用于高性能绘图场景。`Layout` 协议是布局系统的最高抽象，支持创建完全自定义的容器布局算法。`PreferenceKey` 实现了从子视图到父视图的数据传递，是构建自适应布局的重要工具。掌握这些技术，你将能够突破 SwiftUI 内置视图的限制，实现任意复杂的界面效果。
