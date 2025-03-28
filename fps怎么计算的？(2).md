# fps怎么计算的？(2)
好的，我们来聊聊 FPS (Frames Per Second - 每秒帧数) 是如何计算的。

**核心概念:**

FPS 衡量的是图像处理器（GPU）或应用程序在**一秒钟内能够渲染并显示多少个独立的图像帧**。它是一个速率指标，表示画面更新的频率。更高的 FPS 通常意味着更流畅、更连贯的视觉体验。

**基本计算公式:**

最基础的计算方法是：

**FPS = 在特定时间段内渲染的总帧数 / 该时间段的总时长 (秒)**

如果这个时间段恰好是 1 秒，那么：

**FPS = 在过去 1 秒内渲染的总帧数**

**实际应用中的计算方法:**

在实时应用（如游戏、UI动画）中，我们通常不等待整整一秒来计算FPS，因为这样得到的反馈太慢。相反，我们计算的是**瞬时 FPS** 或 **近期平均 FPS**。

1.  **基于单帧耗时计算 (最常用):**
    *   **原理:** 计算渲染**单一一帧**所花费的时间 (Frame Time)。如果渲染一帧需要 `T` 秒，那么理论上在一秒内可以渲染 `1 / T` 帧。
    *   **公式:** `FPS ≈ 1.0 / FrameTime` (其中 `FrameTime` 的单位是秒)
    *   **示例:**
        *   如果渲染一帧需要 16.67 毫秒 (0.01667 秒)，那么 FPS ≈ 1 / 0.01667 ≈ 60 FPS。
        *   如果渲染一帧需要 33.33 毫秒 (0.03333 秒)，那么 FPS ≈ 1 / 0.03333 ≈ 30 FPS。
    *   **优点:** 可以提供非常实时的 FPS 数据。
    *   **缺点:** 这个值可能会非常不稳定，因为每一帧的渲染时间都可能波动。

2.  **基于时间窗口平均计算:**
    *   **原理:** 在一个固定的时间窗口内（例如，过去 0.5 秒或 1 秒），统计该窗口内总共渲染了多少帧。
    *   **公式:** `FPS = FrameCountInWindow / WindowDurationInSeconds`
    *   **示例:** 在过去的 0.5 秒内渲染了 25 帧，那么 FPS = 25 / 0.5 = 50 FPS。
    *   **优点:** 比单帧耗时计算更平滑，能反映一段时间内的平均性能。
    *   **缺点:** 更新频率受限于窗口大小，可能不如瞬时值灵敏。

3.  **基于固定帧数平均计算:**
    *   **原理:** 计算渲染最近 N 帧（例如，最近 60 帧）所花费的总时间。
    *   **公式:** `AverageFrameTime = TotalTimeForLastNFrames / N`
              `AverageFPS = 1.0 / AverageFrameTime = N / TotalTimeForLastNFrames`
    *   **示例:** 渲染最近 60 帧总共花费了 1.2 秒，那么平均 FPS = 60 / 1.2 = 50 FPS。
    *   **优点:** 平滑性好，更新频率可以较快（每帧都可以更新一次滚动平均值）。
    *   **缺点:** 需要存储最近 N 帧的时间戳或耗时。

**在 iOS/macOS 中如何测量帧时间？**

开发者通常使用 `CADisplayLink` 来获取与屏幕刷新同步的回调，并通过回调提供的时间戳来计算两帧之间的时间差，即 `FrameTime`。

**示例 (Swift 使用 CADisplayLink 计算 FPS):**

```swift
import UIKit

class FPSCounter {
    private var displayLink: CADisplayLink?
    private var lastTimestamp: CFTimeInterval = 0
    private var frameCount: Int = 0
    private var fpsUpdateInterval: TimeInterval = 0.5 // 每隔多少秒更新一次 FPS 显示
    private var accumulatedTime: TimeInterval = 0

    var currentFPS: Double = 0

    func start() {
        stop() // Ensure no existing link

        // 使用 weak self 避免循环引用
        displayLink = CADisplayLink(target: WeakProxy(target: self), selector: #selector(WeakProxy.onScreenUpdate(_:)))
        lastTimestamp = CACurrentMediaTime() // Record initial time
        displayLink?.add(to: .main, forMode: .common)
    }

    func stop() {
        displayLink?.invalidate()
        displayLink = nil
        frameCount = 0
        accumulatedTime = 0
        lastTimestamp = 0
        currentFPS = 0
    }

    @objc private func screenDidUpdate(displayLink: CADisplayLink) {
        let currentTimestamp = displayLink.timestamp // Use link's timestamp for accuracy

        // --- Method 1: Instantaneous FPS (less smooth) ---
        // let frameTime = currentTimestamp - lastTimestamp
        // if frameTime > 0 {
        //     currentFPS = 1.0 / frameTime
        // }
        // lastTimestamp = currentTimestamp
        // print("Instantaneous FPS: \(currentFPS)")

        // --- Method 2: Averaged over an interval (smoother) ---
        frameCount += 1
        let deltaTime = currentTimestamp - lastTimestamp
        accumulatedTime += deltaTime
        lastTimestamp = currentTimestamp

        if accumulatedTime >= fpsUpdateInterval {
            currentFPS = Double(frameCount) / accumulatedTime
            print("Average FPS (\(fpsUpdateInterval)s interval): \(String(format: "%.1f", currentFPS))")

            // Reset for next interval
            frameCount = 0
            accumulatedTime = 0
        }
    }

    // Helper proxy to break retain cycle with CADisplayLink target
    private class WeakProxy {
        weak var target: FPSCounter?
        init(target: FPSCounter) { self.target = target }
        @objc func onScreenUpdate(_ displayLink: CADisplayLink) { target?.screenDidUpdate(displayLink: displayLink) }
    }
}

// 如何使用:
// let fpsCounter = FPSCounter()
// fpsCounter.start()
// ...
// fpsCounter.stop() // When done
```

**总结:**

FPS 的计算核心是测量**单位时间内的帧数**或**单帧渲染耗时**。实际应用中常用基于单帧耗时（计算瞬时值）或在一定时间/帧数窗口内取平均值（计算平滑值）的方法。在iOS/macOS开发中，`CADisplayLink` 是获取精确帧时间戳以计算FPS的关键工具。