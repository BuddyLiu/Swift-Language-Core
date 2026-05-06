# 第27章 动画与过渡

动画是提升用户体验的关键因素。SwiftUI 提供了声明式的动画系统，让开发者可以用最少的代码实现流畅、自然的动画效果。从最简单的隐式动画到复杂的自定义动画曲线，SwiftUI 的动画能力覆盖了现代应用开发的全部需求。本章将系统性地介绍 SwiftUI 的动画与过渡体系。

## 27.1 隐式动画：.animation 修饰符

隐式动画是最简单的动画形式——只要状态发生变化，视图就会自动以动画的方式过渡到新状态。

### 基础用法

```swift
struct ImplicitAnimationView: View {
    @State private var scale = 1.0

    var body: some View {
        Button("缩放") {
            scale = scale == 1.0 ? 1.5 : 1.0
        }
        .scaleEffect(scale)
        .animation(.spring(response: 0.5, dampingFraction: 0.6), value: scale)
    }
}
```

`.animation(_:value:)` 接受两个参数：动画描述和监听的数值。当 `value` 发生变化时，视图中依赖该数值的所有可动画属性都会以指定的动画方式过渡。

### 不同动画效果

SwiftUI 内置了多种动画预设：

```swift
VStack(spacing: 20) {
    RoundedRectangle(cornerRadius: 10)
        .fill(.blue)
        .frame(width: isExpanded ? 300 : 100, height: 100)
        .animation(.linear(duration: 0.3), value: isExpanded)

    RoundedRectangle(cornerRadius: 10)
        .fill(.green)
        .frame(width: isExpanded ? 300 : 100, height: 100)
        .animation(.easeInOut(duration: 0.5), value: isExpanded)

    RoundedRectangle(cornerRadius: 10)
        .fill(.orange)
        .frame(width: isExpanded ? 300 : 100, height: 100)
        .animation(.bouncy(extraBounce: 0.2), value: isExpanded)

    RoundedRectangle(cornerRadius: 10)
        .fill(.purple)
        .frame(width: isExpanded ? 300 : 100, height: 100)
        .animation(.snappy, value: isExpanded)
}
```

常用的动画类型包括：

- `.spring`：弹簧动画，可自定义响应时间和阻尼
- `.bouncy`：弹跳动画，iOS 17+ 引入
- `.snappy`：干脆利落的动画，iOS 17+
- `.smooth`：平滑动画，iOS 17+
- `.easeInOut` / `.easeIn` / `.easeOut`：缓动动画
- `.linear`：线性动画
- `.interpolatingSpring`：插值弹簧动画

### 多个可动画属性

当一个视图有多个可动画属性时，可以为每个属性分别设置动画：

```swift
struct MultiAnimationView: View {
    @State private var active = false

    var body: some View {
        Circle()
            .fill(active ? .red : .blue)
            .frame(width: active ? 200 : 100, height: active ? 200 : 100)
            .animation(.spring(response: 0.4), value: active)
            .animation(.easeInOut(duration: 0.5), value: active)
    }
}
```

### 动画事务

动画事务（Transaction）是 SwiftUI 动画系统的底层机制，可以通过 `transaction` 修饰符精细控制：

```swift
Circle()
    .fill(active ? .red : .blue)
    .frame(width: active ? 200 : 100, height: active ? 200 : 100)
    .transaction { transaction in
        transaction.animation = transaction.animation?
            .speed(2)
            .delay(0.1)
    }
```

## 27.2 显式动画：withAnimation

隐式动画自动响应状态变化，而显式动画则明确指定在状态变更时应用动画效果。

### 基本用法

```swift
struct ExplicitAnimationView: View {
    @State private var position = CGPoint.zero

    var body: some View {
        Circle()
            .fill(.teal)
            .frame(width: 80, height: 80)
            .position(position)
            .onTapGesture {
                withAnimation(.spring(duration: 0.6)) {
                    position = CGPoint(
                        x: CGFloat.random(in: 50...350),
                        y: CGFloat.random(in: 100...600)
                    )
                }
            }
    }
}
```

`withAnimation` 包裹的闭包内的状态变更都会以动画方式呈现。与隐式动画不同，显式动画不依赖于 `.animation` 修饰符，而是直接作用于状态变化本身。

### 复杂动画组合

```swift
withAnimation(.easeInOut(duration: 0.5)) {
    isExpanded.toggle()
    opacity = isExpanded ? 1.0 : 0.3
    rotation = isExpanded ? 45 : 0
}
```

多个状态变量可以在同一个 `withAnimation` 中同时更新，所有变化将协同动画。

### 嵌套显式动画

通过在闭包内部嵌套 `withAnimation`，可以为不同属性指定不同的动画效果：

```swift
withAnimation(.spring()) {
    isExpanded.toggle()
}
withAnimation(.linear(duration: 0.3).delay(0.2)) {
    showContent.toggle()
}
```

### 动画完成回调

SwiftUI 没有直接提供动画完成回调，但可以通过 `onAnimationCompleted` 或组合 `Task` 来实现类似效果：

```swift
struct AnimatedWithCompletion: View {
    @State private var animate = false

    var body: some View {
        Text("动画完成示例")
            .scaleEffect(animate ? 2 : 1)
            .onTapGesture {
                withAnimation(.spring()) {
                    animate.toggle()
                }
                // 使用 Task 延迟执行
                Task {
                    try? await Task.sleep(nanoseconds: 600_000_000)
                    print("动画已完成")
                }
            }
    }
}
```

> 更可靠的方式是实现 `Animatable` 协议或使用 `AnimationCompleted` 修饰符。

## 27.3 转场：transition 与 asymmetric

转场（Transition）控制视图出现和消失时的动画效果。

### 内置转场

```swift
struct TransitionExample: View {
    @State private var showDetail = false

    var body: some View {
        VStack {
            Button("切换详情") {
                withAnimation(.spring()) {
                    showDetail.toggle()
                }
            }

            if showDetail {
                DetailCard()
                    .transition(.slide)
            }
        }
    }
}
```

常用的内置转场：

```swift
.transition(.slide)          // 从右侧滑入
.transition(.opacity)        // 渐变
.transition(.scale)          // 缩放
.transition(.move(edge: .bottom)) // 从指定边缘移入
.transition(.push(from: .leading)) // 推入效果（iOS 16+）
.transition(.offset(x: 100, y: 0)) // 自定义偏移
```

### 组合转场

多个转场可以组合使用：

```swift
.transition(.opacity.combined(with: .slide))
```

### Asymmetric 转场

入场和出场使用不同效果：

```swift
.transition(.asymmetric(
    insertion: .scale(scale: 0.5).combined(with: .opacity),
    removal: .slide.combined(with: .opacity)
))
```

`asymmetric` 让你精确控制视图的进入和离开动画，这对于构建复杂的 UI 交互非常有用。

### 转场与动画修饰符

转场效果需要与 `withAnimation` 或 `.animation` 配合使用。如果没有动画上下文，转场将不会产生视觉效果：

```swift
// 正确：使用 withAnimation
Button("切换") {
    withAnimation(.spring()) {
        showView.toggle()
    }
}

// 错误：没有动画上下文，转场不会生效
Button("切换") {
    showView.toggle() // 没有动画效果
}
```

### 匹配几何效果（Matched Geometry Effect）

`matchedGeometryEffect` 是一种特殊的转场，在两个视图之间创建平滑的过渡：

```swift
struct MatchedGeometryExample: View {
    @Namespace private var animation
    @State private var isExpanded = false

    var body: some View {
        VStack {
            if isExpanded {
                ExpandedView()
                    .matchedGeometryEffect(id: "card", in: animation)
            } else {
                CompactView()
                    .matchedGeometryEffect(id: "card", in: animation)
            }
        }
        .onTapGesture {
            withAnimation(.spring()) {
                isExpanded.toggle()
            }
        }
    }
}
```

## 27.4 手势驱动的动画

将手势与动画结合可以创造出高度交互性的用户体验。

### Drag 手势与动画

```swift
struct DraggableCard: View {
    @State private var offset = CGSize.zero
    @State private var isDragging = false

    var body: some View {
        RoundedRectangle(cornerRadius: 16)
            .fill(.linearGradient(
                colors: [.blue, .purple],
                startPoint: .topLeading,
                endPoint: .bottomTrailing
            ))
            .frame(width: 200, height: 250)
            .offset(offset)
            .scaleEffect(isDragging ? 1.05 : 1.0)
            .gesture(
                DragGesture()
                    .onChanged { value in
                        withAnimation(.interactiveSpring()) {
                            offset = value.translation
                            isDragging = true
                        }
                    }
                    .onEnded { value in
                        withAnimation(.spring(response: 0.5, dampingFraction: 0.7)) {
                            offset = .zero
                            isDragging = false
                        }
                    }
            )
    }
}
```

`interactiveSpring()` 是专门为手势交互设计的动画曲线，它会在用户拖拽时立即响应，同时保持流畅的物理感。

### 手势与转场联动

```swift
struct SwipeToDismiss: View {
    @State private var offset = CGSize.zero
    @State private var showCard = true

    var body: some View {
        if showCard {
            CardView()
                .offset(offset)
                .gesture(
                    DragGesture()
                        .onChanged { offset = $0.translation }
                        .onEnded { value in
                            if abs(value.translation.width) > 100 {
                                withAnimation(.easeInOut) {
                                    showCard = false
                                }
                            } else {
                                withAnimation(.spring()) {
                                    offset = .zero
                                }
                            }
                        }
                )
                .transition(.slide.combined(with: .opacity))
        }
    }
}
```

### 滚动动画

利用 `GeometryReader` 结合滚动位置实现视差效果：

```swift
struct ParallaxScrollView: View {
    var body: some View {
        ScrollView {
            ForEach(0..<10) { index in
                GeometryReader { proxy in
                    let offset = proxy.frame(in: .global).minY
                    let scale = 1 + (offset / 1000)

                    CardView(index: index)
                        .scaleEffect(max(0.8, min(1.2, scale)))
                        .opacity(Double(scale))
                }
                .frame(height: 200)
            }
        }
    }
}
```

## 27.5 Animatable 协议与自定义动画曲线

对于需要精细控制动画细节的场景，SwiftUI 提供了 `Animatable` 协议。

### Animatable 协议

```swift
struct AnimatableProgress: View, Animatable {
    var progress: CGFloat

    var animatableData: CGFloat {
        get { progress }
        set { progress = newValue }
    }

    var body: some View {
        GeometryReader { geo in
            ZStack(alignment: .leading) {
                Rectangle()
                    .fill(.gray.opacity(0.2))

                Rectangle()
                    .fill(.blue)
                    .frame(width: geo.size.width * progress)
            }
            .clipShape(RoundedRectangle(cornerRadius: 8))
        }
    }
}

// 使用
AnimatableProgress(progress: 0.75)
    .frame(height: 20)
    .padding()
```

通过实现 `animatableData`，SwiftUI 可以在动画帧之间插值，从而驱动自定义视图的动画。

### 复杂动画数据

对于包含多个可动画属性的视图，可以使用 `AnimatablePair`：

```swift
struct ComplexShape: View, Animatable {
    var start: CGFloat
    var end: CGFloat

    var animatableData: AnimatablePair<CGFloat, CGFloat> {
        get { AnimatablePair(start, end) }
        set {
            start = newValue.first
            end = newValue.second
        }
    }

    var body: some View {
        Path { path in
            // 使用 start 和 end 构建路径
        }
        .stroke(.blue, lineWidth: 2)
    }
}
```

### 自定义动画曲线

除了系统预设的动画类型，还可以通过 `TimingCurve` 创建自定义缓动曲线：

```swift
// iOS 17+ 使用 UnitCurve
let customCurve = UnitCurve.easeInOut
let customSpring = Spring(duration: 0.5, bounce: 0.3)

withAnimation(.spring(customSpring)) {
    // 状态变化
}

// 自定义三次贝塞尔曲线
let timing = Animation.timingCurve(0.68, -0.6, 0.32, 1.6)
withAnimation(timing) {
    isAnimating.toggle()
}
```

### 自定义动画协议

高级用法可以实现自定义的 `CustomAnimation` 协议（iOS 17+）：

```swift
struct BounceAnimation: CustomAnimation {
    let amplitude: Double
    let frequency: Double

    func animate<V>(value: V, time: TimeInterval, context: inout AnimationContext<V>) -> V? where V: VectorArithmetic {
        // 实现自定义的动画逻辑
        let decay = exp(-time * frequency)
        let oscillation = cos(time * frequency * .pi * 2)
        let progress = 1 - decay * oscillation
        return value.scaled(by: progress)
    }

    func shouldMerge<V>(previous: Animation, value: V, time: TimeInterval, context: inout AnimationContext<V>) -> Bool where V: VectorArithmetic {
        return true
    }
}
```

### 性能优化

动画性能优化是实际开发中的关键考量：

```swift
// 使用 .drawingGroup 将视图渲染到离屏位图
ModifiedContent(content: view, modifier: AnyViewModifier { _ in })
    .drawingGroup()

// 使用 .compositingGroup 组合视图层级
ZStack {
    // 子视图
}
.compositingGroup()
.opacity(isVisible ? 1 : 0)

// 避免不必要的重绘
LazyVStack {
    // 大量数据使用懒加载
}
```

## 本章小结

SwiftUI 的动画系统可以分为三个层次：隐式动画通过 `.animation` 修饰符自动响应状态变化；显式动画通过 `withAnimation` 在状态变更时触发；转场控制视图的插入和移除效果。手势驱动的动画将用户交互与视觉响应无缝结合，而 `Animatable` 协议和自定义动画曲线则为高级场景提供了无限可能。在实际开发中，合理选择动画方式、注意性能优化，就能为用户创造出流畅、自然的交互体验。
