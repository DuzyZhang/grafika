# SurfaceFlinger VSync 深度解析：线程模型与请求机制

本文旨在解答关于 `SurfaceFlinger` 中 VSync 线程模型（`app`, `sf`, `appSf`）及其与 `Choreographer` 交互机制的深度疑问。

## 1. `vsync-appSf` 线程：它是谁？什么场景下使用？

在 SurfaceFlinger 的初始化代码中，我们看到了一段关键的分支逻辑：

```cpp
if (vsyncPhaseOffsetNs != sfVsyncPhaseOffsetNs) {
    // 路径 A：相位不同 -> 创建两个独立的线程
    mEventThread = new EventThread(vsyncSrc, ... "app");
    mSFEventThread = new EventThread(sfVsyncSrc, ... "sf");
} else {
    // 路径 B：相位相同 -> 创建一个共用的线程
    mEventThread = new EventThread(vsyncSrc, ... "sf-app"); // 注意名字
    mEventQueue.setEventThread(mEventThread); // SF 也用这个
}
```

### 它是谁？
**`vsync-appSf`** (或者叫 `sf-app`) 本质上是一个 **“合体”线程**。当系统配置中 App 的 VSync 唤醒时间点（Offset）和 SurfaceFlinger 的唤醒时间点**完全一致**时，SurfaceFlinger 就不会浪费资源去创建两个线程了，而是让大家共用一个线程来分发信号。

### 场景分析
1.  **未调优的设备 (Default Configuration)**：
    *   许多中低端设备或早期的 Android 版本，厂商没有针对屏幕特性去精细调整 `vsyncPhaseOffsetNs`（相位偏移），默认都是 0。
    *   在这种情况下，App 和 SF 都应该在 VSync 信号到来的那个瞬间同时开工。此时就会进入这个 `else` 分支，使用 `vsync-appSf`。
2.  **特殊的省电模式**：
    *   某些省电策略可能会强制将 Offset 归零，减少系统唤醒频次，强制 App 和 SF 串行工作（虽然略微增加帧延迟，但减少了上下文切换）。

### 与其他两个线程的不同？
*   **`vsync-app` + `vsync-sf`**: 代表**“流水线模式” (Pipelined)**。App 在 T时刻醒来，SF 在 T+Offset 时刻醒来。两人错峰工作，并行度高，Touch-to-Display 延迟低。
*   **`vsync-appSf`**: 代表**“同步/串行模式”**。App 和 SF 被同一个线程唤醒。这通常意味着 SF 必须等下一帧才能合成 App 这一帧产生的内容（增加了一帧的显示延迟）。

---

## 2. 为什么需要 `vsync-app` 和 `vsync-sf` 分离？(The "Pipeline" Philosophy)

将它们分离的核心目的是为了**性能流水线 (Pipelining)** 和 **降低延迟 (Latency Reduction)**。

### 并没有“更早触发”那么简单
直觉上我们认为“App 先画，画完 SF 再合成”，所以 App 应该更早触发。但事实可能恰恰相反或者更复杂（App 往往晚触发，SF 早触发针对的是**下一帧**的截止时间）。

但核心逻辑是为了**错峰出行**。

#### 关键概念：Phase Offset (相位偏移)
想象 VSync 周期是 16.6ms。

*   **如果不分离 (Offset = 0)**:
    *   0ms: VSync 信号到。App 开始画 (UI Thread -> RenderThread)。
    *   0ms: SF 也被唤醒了。但此时 App 新的 Buffer 还没画好（刚开始画），SF 只能拿 App **上一帧** (N-1) 的旧 Buffer 去合成。
    *   **结果**: App 这一帧 (N) 的内容，最早也要等到 **下一个 VSync** 才能被 SF 拿走。显示延迟 = **2帧**。

*   **如果分离 (理想流水线)**:
    *   **App Offset (比如 2ms)**: VSync 后 2ms，App 被唤醒开始画。它有 14ms 的时间画图。
    *   **SF Offset (比如 16ms/下个VSync前夕)**: 当 App 快画完的时候（或者刚好画完），SF 被唤醒。
    *   **理想状态**: SF 醒来时，App 刚好把帧 N 的 Buffer 塞入队列。SF 直接拿走帧 N 进行合成。
    *   **结果**: App 画完立刻被合成显示。显示延迟 = **1帧**。

**结论**：分离线程是为了让 App 和 SF 拥有**独立的 wakeup 自定义能力**。通过精心调整这两个 Offset，可以让 CPU（App绘图）和 GPU（SF合成）在时间轴上完美错开，极致压缩“点击-显示”延迟。

---

## 3. Choreographer 为什么要“每次”都请求？(One-Shot Mechanism)

你观察非常敏锐。`BitTube` 确实是一个长连接管道，但在 Android 图形设计中，VSync 分发采用的是 **"One-Shot" (单次触发)** 机制，而不是 **"Continuous" (广播)** 机制。

### 为什么不一直发？(第一性原理：省电)
如果 `EventThread` 只要有 VSync 信号就通过 BitTube 往外发：
1.  **CPU 无法休眠**: 即使屏幕是静止的（比如用户在看电子书），App 的 `Choreographer` 也会每秒收到 60 次信号，`Looper` 会被唤醒 60 次，CPU 必须从 Idle 状态切出。这对电池是毁灭性的。
2.  **无效做功**: 屏幕静止时，App 不需要重绘，`doFrame` 空跑没有任何意义。

### 请求机制详解
1.  **默认状态 (Mute)**:
    *   虽然 BitTube 连着，但 `EventThread` 里的逻辑是：只要没有请求 (`count <= 0`)，我就不往管子里写数据。
    *   默认情况下，App 的 Connection `count` 是 0。VSync 信号来了，`EventThread` 直接跳过这个连接。

2.  **`scheduleFrameLocked` (Unmute)**:
    *   当你的 View 调用 `invalidate()` 或者有 Input 事件时，Choreographer 调用 `scheduleFrameLocked`。
    *   这会通过 JNI 调用 `DisplayEventReceiver::requestNextVsync()`。
    *   **底层逻辑**: 它通过 Binder 告诉 `EventThread`：“我要 **1个** 信号”。(`count++` 或 `setVsyncRate(1)`)

3.  **触发与重置**:
    *   下一个 VSync 来临时，`EventThread` 看到 `count > 0`。
    *   它往 BitTube 里塞**一个**事件。
    *   **关键**: 它立刻把 `count` 重置回 0 (或 `-1`, 取决于版本实现，通常叫 `vsyncRequest = Single`)。

**结论**: Android 图形系统的默认状态是 **“静默”** 的。只有 App 主动举手说“我要画下一帧”（requestNextVsync），系统才会给它发且只发一个信号。这就是为什么即使 BitTube 一直通着，Choreographer 也必须每次都去“乞求”一下。
