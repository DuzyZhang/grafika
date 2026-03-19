# Android Window 变化通知机制总结：从 `relayoutWindow` 到一帧真正显示

这份文档不再从“窗口变了会通知谁”这种零散问题出发，而是围绕一条完整主线来理解：

`窗口条件变化 -> ViewRootImpl 发起 relayoutWindow -> WMS / SurfaceControl 更新窗口几何契约 -> 应用更新可用 Surface -> ThreadedRenderer 提交新 buffer -> SurfaceFlinger 在显示节拍上消费 layer state + buffer state`

只要这条主线清楚，很多看似分散的知识点都会自动归位：

1. 为什么 `relayoutWindow` 不是“重画一帧”。
2. 为什么系统既要同步返回，又要异步回调。
3. 为什么窗口位置变化和内容绘制明明都和“一帧显示”有关，却不是同一条调用链。
4. 为什么后来越来越强调 `SurfaceControl.Transaction`、BLAST 和 layer / buffer 一致性。

---

## 1. 第一性原理：为什么窗口变化不能只靠应用自己处理

Window 不是普通 View，它是系统级显示资源的抽象。

它天然受三类约束：

- **跨进程约束**：应用可以生产自己的内容，但不能自己决定系统里所有窗口的叠放顺序与最终显示方式。
- **全局协调约束**：状态栏、导航栏、输入法、应用窗口、弹窗共享同一块屏幕，必须由 `WindowManagerService` 统一裁决。
- **显示一致性约束**：窗口位置、尺寸、Insets、可见区域变化，必须尽量和内容更新对齐，否则就会出现黑边、错位、拉伸、跳变。

所以 Android 的设计不是“View 变了就自己重画”，而是：

- App 负责提出“我这扇窗当前希望长什么样”。
- WMS 负责决定“系统允许它最终长什么样”。
- 渲染系统负责往这扇窗当前绑定的 `Surface` 里提交内容。
- SurfaceFlinger 负责把这扇窗对应 layer 的状态和 buffer 一起合成显示。

---

## 2. 先抓总主线：`relayoutWindow` 为什么是最佳切口

`ViewRootImpl` 是应用侧与窗口系统之间最重要的桥，而 `relayoutWindow()` 正好站在两条链路的交界点：

- 一条是 **窗口状态链**
  关注 frame、Insets、可见区域、surface 是否需要 resize / recreate，以及窗口对应的 `SurfaceControl/layer` 应该处于什么状态。
- 一条是 **内容绘制链**
  关注应用后续到底应该往哪个 `Surface` 里画，以及这一帧内容何时通过 `queueBuffer` 提交。

所以 `relayoutWindow()` 的价值不在于“它触发了一次重绘”，而在于：

**它负责让应用和系统重新确认当前这扇窗的显示契约。**

更直白地说：

- 这扇窗现在几何条件是什么。
- 它当前对应哪个 `SurfaceControl/layer`。
- 后续内容应该继续提交到哪个 producer `Surface`。

这就是为什么从 `relayoutWindow` 出发，很容易把 `ViewRootImpl`、WMS、`SurfaceControl`、BLAST、`ThreadedRenderer`、`queueBuffer`、SurfaceFlinger 串成一条线。

---

## 3. `relayoutWindow` 到底做了什么

当 `ViewRootImpl` 调用 `mWindowSession.relayout(...)` 时，背后通常在完成四类事情。

### 3.1 重新确认窗口几何状态

例如：

- window frame
- 可见区域
- Insets
- 系统栏 / 输入法带来的遮挡与可用区变化
- surface 是否需要 resize / recreate

这部分回答的是：

**“这扇窗现在在系统里应该占哪块区域。”**

### 3.2 重新确认窗口在系统中的控制对象

也就是：

- 这个窗口当前对应哪个 `SurfaceControl`
- 这个 `SurfaceControl` 背后的 layer 状态是否需要更新

这里要特别分清：

- `Surface` 更像是“内容生产入口”
- `SurfaceControl` 更像是“系统里这个图层怎么显示的控制句柄”

### 3.3 决定应用后续应该往哪里画

这是最容易被忽略的一点。

`relayoutWindow` 不只是同步返回一些 frame / Insets，它还会影响应用后续绘制绑定的 `Surface` 是否需要更新。

在启用 BLAST 的模型下，这一步常常会继续牵涉：

- `BLASTBufferQueue` 的初始化或更新
- `createSurface()`
- `mSurface.transferFrom(...)`

所以更准确地说：

**`relayoutWindow` 不直接交付这一帧像素，但它会重新确定“从下一帧开始你该往哪个 producer surface 里画”。**

### 3.4 返回同步结果，并埋好后续异步通知链

这就是为什么你既会看到 `relayoutWindow` 的同步返回值，又会在后续看到 `IWindow.resized()` 之类的异步通知。

---

## 4. 为什么同时需要“同步返回”和“异步回调”

这是窗口通知机制最容易被误读的地方。

### 4.1 同步返回解决什么

同步返回主要解决：

- 当前这次 `performTraversals()` 立刻就要用到的结果
- 例如 frame、Insets、surface 变化结果

因为遍历继续往下走时，应用必须马上知道：

- 当前该按多大尺寸布局
- 当前可见区域是多少
- 当前绘制目标是否发生了变化

所以这些信息必须同步拿回来。

### 4.2 异步回调解决什么

异步回调例如 `IWindow.resized()` 更适合处理：

- 某些窗口状态变化稍后才稳定
- 某些 frame / Insets / 配置变化需要系统侧进一步推进后再通知应用

所以它们不是重复机制，而是分工：

- **同步返回**：保证当前遍历能继续。
- **异步回调**：补充后续状态变化。

---

## 5. 真正必须分开的两条链：窗口状态链 vs 内容绘制链

理解 `relayoutWindow` 时，最重要的不是盯某个函数，而是分清下面两条链。

### 5.1 窗口状态链

这条链关注的是：

- `WindowManagerService`
- `SurfaceControl`
- `SurfaceControl.Transaction`
- layer 的位置、尺寸、裁剪、可见性

它回答的问题是：

**“这扇窗作为系统中的一个 layer，应该怎么摆、怎么裁、怎么显示。”**

### 5.2 内容绘制链

这条链关注的是：

- `ThreadedRenderer`
- `HardwareRenderer.nSyncAndDrawFrame(...)`
- `RenderProxy`
- `CanvasContext`
- `ANativeWindow`
- `queueBuffer`

它回答的问题是：

**“这扇窗里这一帧的内容是什么。”**

### 5.3 为什么两条链必须对齐

因为最终上屏看到的不是 buffer 本身，而是：

**buffer 内容 + 它所属 layer 的当前显示状态**

一旦两条链不同步，就容易出现：

- 新位置 + 旧 buffer
- 旧位置 + 新 buffer

最终表现为：

- 黑边
- 错位
- 拉伸
- 一帧跳变

所以 Android 后来强调的重点不是“让 `draw()` 顺手去改窗口属性”，而是：

**让窗口 layer state 和后续提交的 buffer state 最终围绕同一个显示对象更一致地被 SurfaceFlinger 消费。**

---

## 6. BLAST / Transaction 到底“绑定”了什么

这部分最容易在字面上理解错。

很多人会以为：

- `relayoutWindow` 改窗口位置
- `draw` 提交内容
- 那是不是 Java 层里有一个“把两者手动打包成单一事务”的大函数

更准确的理解不是这样。

BLAST / `SurfaceControl.Transaction` 真正做的事情是：

- `relayoutWindow` / WMS / `SurfaceControl.Transaction`
  负责更新这扇窗对应 layer 的显示状态
- `draw`
  负责通过当前绑定的 `Surface` 提交新的 buffer
- BLAST
  负责让“这个 layer 的几何 / 显示状态变化”和“这个 layer 的 buffer 更新”更稳定地围绕同一个显示对象收敛

所以它绑定的不是两个 Java 方法调用本身，而是：

- **layer state**
- **buffer state**

你可以记成一句特别重要的话：

- `Buffer` 解决“画了什么”
- `Layer` 解决“怎么显示”
- `Transaction` 解决“这些状态何时一起生效”

---

## 7. `ThreadedRenderer.draw()` 在这条链里的角色

Java 层里最像“开始真正交付这一帧内容”的入口，通常是：

- `ThreadedRenderer.draw(...)`
- 最终进入 JNI：`HardwareRenderer.nSyncAndDrawFrame(...)`

这一段的职责不是改窗口几何，而是：

- 同步 RenderNode 树
- 组织这一帧 GPU 工作
- 把结果提交到与当前窗口绑定的 `Surface`

所以更准确地说：

**`draw` 做的是“往当前已绑定好的渲染目标里交这一帧内容”，不是“重新定义这扇窗的系统几何状态”。**

---

## 8. 如果在 SurfaceFlinger 消费时，窗口位置又变了，会发生什么

这是理解 layer / buffer 一致性的关键场景。

假设某次 SurfaceFlinger 即将 latch / 合成时，`relayoutWindow` 又触发了一次窗口位置变化，那么系统里其实有两条变化在赛跑：

- **buffer 内容链**：这一帧的内容何时 ready、何时 fence 满足、何时可被 latch
- **layer 状态链**：新的位置 / 裁剪 / 尺寸何时通过 `SurfaceControl.Transaction` 提交到 SF 可见

此时结果取决于：

- 新位置是否赶上了当前这轮 SF 合成
- 新 buffer 是否也与之匹配地 ready

可能出现三种情况：

### 8.1 新位置赶上了，本帧也有匹配的新 buffer

这一帧显示正常，用户看到的就是新位置和新内容。

### 8.2 新位置没赶上，本帧仍沿用旧位置

这一帧继续按旧位置显示，下一帧才跳到新位置。通常表现为一帧滞后，而不一定是视觉错误。

### 8.3 最糟糕的错配：新旧状态混搭

例如：

- 新位置 + 旧 buffer
- 旧位置 + 新 buffer

这时候就会出现：

- 黑边
- 错位
- 边缘裁剪不对
- 一帧跳动

所以问题的本质不是“SF 会不会消费失败”，而是：

**SF 在这次合成里拿到的是不是一组版本一致的 layer state + buffer state。**

---

## 9. `reportDrawFinished()` 应该怎么理解

这一点很容易被说得过重。

更稳妥的理解是：

- 它不是普通高性能每帧提交流程的核心主路径
- 它更多是在某些窗口同步场景下，向系统说明：
  - 这一轮 draw 已经完成
  - 某些依赖“窗口已完成绘制”的系统逻辑可以继续推进

所以不要把它理解成：

- “app 每帧内容都是靠它提交给 SF 的”

更接近的是：

- 它是窗口同步协议中的一个确认点
- 而不是替代 `queueBuffer` 的主提交流程

---

## 10. 一条最值得记住的闭环

如果只记一条闭环，我建议记下面这 8 步：

1. `ViewRootImpl.performTraversals()`
2. `relayoutWindow()`
3. WMS 计算窗口几何与 surface 相关结果
4. `ViewRootImpl` 更新 `SurfaceControl` / BLAST / `mSurface`
5. `ThreadedRenderer.draw()`
6. `nSyncAndDrawFrame(...)`
7. app 往当前绑定的 `Surface` 提交新 buffer
8. SurfaceFlinger 在显示节拍上消费：
   - 这个 layer 的当前状态
   - 这个 layer 的当前可用 buffer

如果你能把这 8 步讲顺，`relayoutWindow` 这块就已经不是“背到了名词”，而是真正入门了。

---

## 11. 作为应用开发者，读这条链最该学到什么

### 11.1 Window 变化不是“顺手改个布局”

它背后会牵涉：

- Binder IPC
- WMS 计算
- `SurfaceControl`
- BLAST
- 渲染目标更新

所以复杂窗口变化天然比普通 View 内容刷新重。

### 11.2 定位异常显示时，不要只盯某一个点

看到错位、黑边、跳变时，不要只问：

- `draw()` 有没有跑
- WMS 有没有通知

更应该问：

- 这一帧的 buffer 到位了吗
- 与之匹配的 layer 状态到位了吗
- SurfaceFlinger 在本次合成时消费到的是不是同一版本

### 11.3 `relayoutWindow` 是理解“为什么 Window 比普通 View 特殊”的最佳入口

因为它把下面这些模块一次性串了起来：

- `ViewRootImpl`
- WMS
- `SurfaceControl`
- BLAST
- `ThreadedRenderer`
- `queueBuffer`
- SurfaceFlinger

---

## 12. 建议插入总纲的部分内容

这份专题稿不适合整篇并入总纲，但很适合往总纲里插一小段“窗口变化主结论”。

建议保留的摘要是：

`relayoutWindow` 的本质不是“重画一帧”，而是让应用与系统重新确认当前窗口的几何状态、surface 绑定关系和显示契约；真正的一帧内容仍在 `ThreadedRenderer.draw -> nSyncAndDrawFrame -> queueBuffer` 这条内容链里提交。BLAST / Transaction 的价值在于让窗口 layer state 与后续提交的 buffer state 尽量在同一个显示周期内保持一致。`

---

## 13. 源码阅读抓手

如果接下来继续读源码，建议优先盯这几个点：

- `ViewRootImpl.performTraversals()`
- `ViewRootImpl.relayoutWindow()`
- `ViewRootImpl.updateBlastSurfaceIfNeeded()`
- `ViewRootImpl.getOrCreateBLASTSurface()`
- `ThreadedRenderer.draw(...)`
- `HardwareRenderer.nSyncAndDrawFrame(...)`
- `SurfaceControl.Transaction`

这几个点最容易把“窗口状态链”和“内容绘制链”先立起来。
