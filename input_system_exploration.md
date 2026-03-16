# Android Input 系统全链路总结：从 `/dev/input` 到 `View`

这份文档的目标不是堆概念，而是把 Android 输入系统回答成一条完整工作流：

1. 硬件事件是如何从内核进入 framework 的。
2. `InputReader` 和 `InputDispatcher` 为什么要拆成两个线程。
3. 窗口、`InputChannel`、焦点和目标窗口是如何关联起来的。
4. Input ANR 到底盯的是哪一段，为什么 `finishInputEvent` 能“解除警报”。

如果以后复习时始终围绕这 4 个问题去看，输入系统就不会再是一堆散点知识。

---

## 总主线：一次输入从哪里来，到哪里去

先记住这条主线：

`硬件中断 -> 内核 evdev -> /dev/input/eventX -> EventHub -> InputReader -> NotifyArgs -> InputDispatcher -> 找目标窗口 -> InputChannel -> 应用进程 InputConsumer -> ViewRootImpl -> View/Window 回调 -> finishInputEvent`

其中最容易混淆的 4 个边界是：

- `InputReader` 负责把“原始设备协议”翻译成 Android 认识的高层事件。
- `InputDispatcher` 负责决定“这件事该发给谁”。
- `InputChannel` 负责跨进程通信，不负责业务判断。
- Input ANR 盯的是“事件发给应用后，应用是否按时确认处理完”，不是盯内核、也不是盯 `InputReader`。

---

## 第一阶段：硬件与内核，事件的诞生

输入的起点不是 Android framework，而是 Linux 输入子系统。

### 1. 硬件如何上报

- 触摸屏、键盘、鼠标等硬件产生电信号。
- 驱动把这些信号转换成 Linux 标准输入事件。
- 内核通过 `evdev` 协议把事件暴露到 `/dev/input/eventX`。

### 2. 事件的基本格式

内核提供的是 `struct input_event`，典型字段有：

- `type`
- `code`
- `value`
- `time`

例如：

- `EV_KEY` 表示按键类事件
- `EV_ABS` 表示绝对坐标类事件
- `EV_REL` 表示相对位移
- `EV_SYN` 表示一次硬件状态上报的“帧边界”

### 3. 为什么 `EV_SYN` 很重要

`EV_SYN` 不是“用户停止操作”的信号，而是驱动在这一轮硬件状态上报结束时打的分隔符。

它的意义是：

- 前面的若干 `EV_ABS` / `EV_KEY` 是同一批状态变化
- `InputMapper` 应该在看到 `EV_SYN` 后，把前面累积的状态组装成一个完整的 Android 高层事件

这一阶段的关键理解：

- Android 读到的第一手数据不是 `MotionEvent`，而是一串 Linux 输入协议事件。

---

## 第二阶段：EventHub，framework 与内核设备文件的桥

`EventHub` 是 Native 输入系统接触内核事件的第一站。

### 核心职责

- 监控 `/dev/input/eventX`
- 感知设备增删
- 批量读取原始输入事件
- 把 `struct input_event` 包装成 Android 自己的 `RawEvent`

### 为什么它高效

`EventHub` 不是轮询所有设备，而是用 `epoll` 做阻塞等待：

- 没有事件时睡眠，不消耗 CPU
- 有事件时被唤醒，批量读取

### `RawEvent` 的价值

`RawEvent` 可以理解为 Android 对内核事件的第一层包装，它比 `struct input_event` 多了 framework 关心的上下文，例如：

- `deviceId`
- 更统一的事件来源信息

这里的“上下文”不是指它已经理解了“点击”“滑动”这种用户行为语义，而是指“这条原始事件来自哪个设备、该交给哪类 framework 设备对象继续处理”。

典型包括：

- 这条事件属于哪个 `deviceId`
- 这个设备是否刚刚被新增或移除
- 这个设备具备哪些能力，例如键盘、触摸、相对位移、绝对坐标
- 这个设备的标识信息，例如名称、vendor/product、总线类型

这些信息的价值在于：

- `InputReader` 能据此把事件路由给正确的 `InputDevice`
- `InputReader` 能在设备新增时选择合适的 `InputMapper`
- framework 能把“同样是 `EV_KEY`”区分为来自键盘、遥控器还是游戏手柄的协议流

所以更准确地说：

- `EventHub` 补的是“来源上下文”和“设备上下文”
- `InputReader/InputMapper` 才负责把这些原始事件翻译成 Android 输入语义

这一阶段的关键理解：

- `EventHub` 不做“语义翻译”，只负责高效拿到事件并补齐 Android 需要的上下文。

---

## 第三阶段：InputReader，把协议流翻译成 Android 事件

`InputReader` 是输入系统里的“翻译官”。

它面对的是：

- 来自 `EventHub` 的 `RawEvent`
- 其中既有设备增删事件，也有用户操作事件

它要输出的是：

- `NotifyKeyArgs`
- `NotifyMotionArgs`
- 其他可被 `InputDispatcher` 接收的高层通知

### 1. 线程模型

`InputReader` 跑在独立线程中，核心循环可以概括为：

- 从 `EventHub` 读取一批 `RawEvent`
- 逐个交给对应的 `InputDevice`
- 再由 `InputDevice` 分发给内部的 `InputMapper`

### 2. `InputDevice` 与 `InputMapper` 的分工

- `InputDevice`
  代表 framework 中的一个输入设备实例，维护设备能力、配置和 mapper 集合。
- `InputMapper`
  真正理解某类协议语义的状态机，比如：
  - `KeyboardInputMapper`
  - `TouchInputMapper`

一个 `InputDevice` 可以挂多个 `InputMapper`，这就是为什么一个物理设备可以同时提供多种输入能力。

### 3. 为什么有限的 Mapper 能支持无限设备

因为 Android 适配的不是某个厂家硬件本身，而是标准化后的输入协议。

也就是说：

- `Mapper` 理解的是 `EV_KEY` / `EV_ABS` / `EV_SYN` 这些协议语义
- 不需要为“每个品牌的触摸屏”写一套新逻辑

### 4. 设备增删为什么也必须走这条链

`DEVICE_ADDED` / `DEVICE_REMOVED` 很关键，因为它决定了：

- 要加载哪个 `.idc` 配置
- 要创建什么类型的 `InputMapper`
- 要不要清理已有状态
- 设备移除时是否要生成 `CANCEL`

### 5. `EV_SYN` 到高层事件的组装

`InputMapper` 不会见到一个 `EV_ABS` 就立刻吐一个 `MotionEvent`，而是：

- 先累积一批状态变化
- 等遇到 `EV_SYN`
- 再组装成一次完整的“触摸/按键结果”

这一阶段的关键理解：

- `InputReader` 做的是“从协议到语义”的翻译。
- 这一步之后，事件才开始脱离硬件细节，进入 Android 输入框架自己的世界。

---

## 第四阶段：从 Reader 到 Dispatcher，生产者-消费者模型

为什么 `InputReader` 和 `InputDispatcher` 要拆成两个线程？因为“采集输入”和“决定发给谁”是两类完全不同的工作。

### Reader 是生产者

`InputReader` 翻译完事件后，会通过监听器接口把结果发出去，例如：

- `notifyKey()`
- `notifyMotion()`

这些调用最终进入 `InputDispatcher`。

### Dispatcher 是消费者

`InputDispatcher` 收到通知后会：

1. 做必要的前置策略处理
2. 把事件包装成内部管理对象
3. 放入 `mInboundQueue`
4. 唤醒自己的分发线程

### 为什么这样拆

好处有两个：

- 输入采集不会因为窗口定位、策略判断等较重逻辑而阻塞
- 分发线程可以独立决定调度顺序、超时、唤醒和重试

这一阶段的关键理解：

- `InputReader` 关注“事件是什么”
- `InputDispatcher` 关注“事件发给谁”

---

## 第五阶段：InputDispatcher，输入分发的总调度中心

`InputDispatcher` 是整个输入系统最容易感觉“复杂但抓不住”的部分。其实只要抓住它的职责，就清楚了。

它的核心职责只有 4 个：

1. 决定目标窗口
2. 管理分发队列
3. 跟踪连接状态
4. 监控超时并上报 ANR

### 1. 事件进入 Dispatcher 后会变成什么

高层通知进入 `InputDispatcher` 后，会被包装成内部事件条目，例如：

- `KeyEntry`
- `MotionEntry`

它们都属于 `EventEntry` 体系。

可以这样理解：

- `KeyEvent` / `MotionEvent` 更像“事件内容”
- `KeyEntry` / `MotionEntry` 更像“带路由、超时、策略信息的内部调度档案”

### 2. 为什么会先入 `mInboundQueue`

因为 `InputDispatcher` 不希望在 Reader 调用栈里直接做完整分发，它要把事件先纳入自己的调度域，再统一处理：

- 焦点判断
- 目标窗口选择
- 特殊策略键处理
- ANR 计时
- 连接忙闲控制

### 3. Policy 拦截发生在哪里

有些键不能直接发给应用，例如：

- Home
- Power
- 某些系统保留按键

这时 `InputDispatcher` 会通过 policy 路径把问题交给系统策略层处理。Native 侧你看到的通常是 `NativeInputManager` 这一层代理，Java 侧常落到 `PhoneWindowManager`。

### 4. 为什么有 `postCommandLocked`

这是个很关键的设计点。

`InputDispatcher` 自己有一把核心锁 `mLock`。如果持锁时直接调用外部逻辑，例如：

- Policy
- Java 层
- 可能重入或耗时的回调

就很容易死锁。

所以它会把这类操作封装成 command，稍后在更安全的时机执行。这本质上是：

- 锁内只做快速状态变更
- 锁外再做可能耗时的外调

这一阶段的关键理解：

- `InputDispatcher` 的复杂度主要来自“调度”和“并发安全”，不是来自事件结构本身。

---

## 第六阶段：目标窗口是怎么找到的

输入系统真正难串起来的地方，往往在“事件到底是怎么落到某个窗口身上的”。

### 1. 先按 displayId 缩小范围

`displayId` 不是直接定位窗口，它先确定“这是哪块物理显示上的事件”。

### 2. 再找焦点窗口或命中窗口

不同事件的目标定位逻辑不同：

- 按键类事件通常找当前焦点窗口
- 触摸类事件更多依赖命中测试、窗口可触区域、触摸状态机等

在新版系统里，焦点解析已经抽象到类似 `FocusResolver` 的组件中，`InputDispatcher` 更偏向查询结果而不是自己维护全部查找细节。

这里要特别区分“按键”和“触摸”：

- 对按键来说，`InputDispatcher` 更像是在指定 `displayId` 上查询“当前焦点窗口是谁”。
- 对触摸来说，`InputDispatcher` 仍然要主动做目标选择，因为它需要在一笔手势开始时根据窗口输入信息做命中测试。

更准确的分工是：

- `WindowManagerService` 负责维护窗口输入相关信息，例如层级、可触区域、是否可接收触摸、窗口标志位等。
- `InputDispatcher` 基于这些窗口信息，在 `ACTION_DOWN` 到来时决定这笔触摸应该落到哪个窗口。
- 一旦目标窗口确定，后续同一笔手势的 `MOVE/UP` 通常会继续发给同一个目标窗口，以保证手势连续性，而不是每次移动都重新全量命中测试。

所以如果问题是“触摸事件到底是谁负责查该发给哪个窗口”，答案应该是：

- 窗口信息的维护者主要是 `WMS`
- 真正执行触摸目标选择和维护触摸分发状态机的是 `InputDispatcher`

### 3. 为什么目标是窗口，而不是 Activity

因为输入是发给“窗口”的，不是发给“Activity 逻辑对象”的。

这解释了很多现象：

- `Dialog` 有自己的窗口，也有自己的输入通道
- 子窗口、弹窗、系统窗口都能独立接收输入
- ANR 也能更准确地定位到具体窗口

这一阶段的关键理解：

- Android 输入系统的分发粒度是 window，不是 activity。

---

## 第七阶段：InputChannel，跨进程输入通道

当 `InputDispatcher` 决定了目标窗口后，真正把事件送到应用进程靠的是 `InputChannel`。

### 1. 它是什么

`InputChannel` 可以理解为一对跨进程通信端点：

- 服务端一端在 system_server / input dispatcher 侧
- 客户端一端在应用进程侧

### 2. 它和 Window 的关系

`InputChannel` 生命周期与 `Window` 绑定，不与 `Activity` 绑定。

所以：

- 每个可接收输入的窗口都有自己的 `InputChannel`
- 一个 Activity 里弹出 Dialog，本质上会多出新的窗口和新的输入通道

### 3. 为什么必须按窗口建通道

这能带来 3 个收益：

- 分发粒度清晰：事件知道该发给哪个窗口
- 连接状态独立：一个窗口卡住不会把所有窗口混成一锅
- ANR 定位准确：能知道到底是哪个窗口没及时响应

### 4. `InputDispatcher` 是怎么通过 `InputChannel` 把事件发到应用进程的

这一步是很多资料会一句带过，但实际上很关键的桥接过程。

可以把它拆成 4 步：

1. `InputDispatcher` 先为目标窗口找到对应的 `Connection`
2. 然后创建一个 `DispatchEntry`，表示“这次事件准备发往这个窗口”
3. 接着把事件序列化成底层可传输的 `InputMessage`
4. 最后通过 `Connection` 关联的服务端 `InputChannel` 把消息写出去

这里最重要的是理解：

- `InputDispatcher` 发给对端的不是 Java 层的 `MotionEvent` 对象
- 而是 Native 层的输入消息结构，再通过 `InputChannel` 这条跨进程通道送到应用侧

从调度角度看，常见会分成两个动作：

- `enqueueDispatchEntryLocked`
  负责把“发给这个窗口的这次事件”加入该连接的待发送队列
- `startDispatchCycleLocked`
  负责真正启动发送流程，把队列头部事件写入 `InputChannel`

之所以拆成两步，是因为 `InputDispatcher` 不只是“写 socket”那么简单，它还要同时维护：

- 这个连接当前是否空闲
- 前一个事件是否还没等到应用确认完成
- 当前这次发送是否应该开始 ANR 计时

所以你可以把 `InputChannel` 想成“管道”，但真正负责“什么时候发、能不能发、发出去后怎么记账”的，是 `InputDispatcher + Connection + DispatchEntry` 这套组合。

### 5. 为什么能跨进程送达

`InputChannel` 本质上是一对成对创建的通信端点：

- system_server / input dispatcher 持有服务端端点
- 应用进程持有客户端端点

系统在窗口建立时，把客户端那一端交给应用进程的 `ViewRootImpl`。

于是后续流程就成立了：

- `InputDispatcher` 往服务端 channel 写入 `InputMessage`
- 内核把消息送到对端端点
- 应用侧 `InputConsumer` 从客户端 channel 读出消息

所以跨进程输入并不是“系统直接调用了应用里的某个方法”，而是：

- 先通过 `InputChannel` 把底层消息送到应用进程
- 再由应用进程自己的输入消费链把它恢复成 `KeyEvent` / `MotionEvent`

---

## 第八阶段：WMS、IMS、ViewRootImpl 各自扮演什么角色

这一组类最容易背混，所以必须分职责记。

### `ViewRootImpl`

应用进程中窗口的总入口，负责：

- 与 `WindowManagerService` 通信
- 持有应用侧 `InputChannel`
- 接收从系统发来的输入
- 把输入继续交给 View 树

它是“应用窗口与系统服务的桥”。

### `WindowManagerService` (WMS)

负责窗口管理，关心的是：

- 窗口的创建、层级、属性、焦点
- 哪个窗口当前可接收输入
- 窗口 token 与系统状态的维护

### `InputManagerService` (IMS)

它是 Java 服务层面对 native 输入系统的封装和管理入口。

你可以把它理解为：

- Java 世界通往 native input manager 的服务门面
- WMS 和 native 输入子系统之间的重要桥梁

### 三者怎么协作

典型流程是：

- 应用侧 `ViewRootImpl` 请求创建/更新窗口
- 请求先到 `WMS`
- `WMS` 再协同 `IMS` / native input 侧创建并注册对应 `InputChannel`

这一阶段的关键理解：

- 应用不是直接和 `InputDispatcher` 交互
- 中间一定经过窗口管理与输入服务这套系统协作链

---

## 第九阶段：事件进入应用进程后发生什么

事件通过 `InputChannel` 进入应用进程后，并不会立刻直接调用某个 `View`。

中间还有一层应用侧输入消费链。

### 典型过程

- 应用侧 `InputConsumer` 从 channel 取出消息
- 转成 `KeyEvent` / `MotionEvent`
- `ViewRootImpl` 将其包装成内部待处理节点，例如 `QueuedInputEvent`
- 进入应用自己的输入处理流水线
- 再逐步分发到 `DecorView` / `ViewGroup` / `View`

### 为什么 Java 层还要再处理一次

因为 native 发到应用侧的还是“跨进程消息结果”，应用真正关心的是：

- 能直接调用的 `MotionEvent` / `KeyEvent`
- 能参与事件分发和拦截机制的 Java 对象

这一阶段的关键理解：

- `InputDispatcher` 负责把事件送达应用进程
- 但“事件在应用内部怎么走完 View 树”是应用侧输入管线的职责

这里再补一层非常关键的对应关系：

- system_server 侧看到的是 `Connection -> InputChannel(server) -> write InputMessage`
- 应用进程侧看到的是 `InputChannel(client) -> InputConsumer -> InputEvent`

这两边一写一读，中间靠的是同一对 `InputChannel` 端点。

所以如果以后你问自己“事件是怎么从 `InputDispatcher` 真正进到 app 的”，最短答案应该是：

`InputDispatcher` 把事件封装成 `InputMessage`，通过目标窗口的服务端 `InputChannel` 写出；应用进程中的 `InputConsumer` 再从客户端 `InputChannel` 读入并恢复成 `InputEvent`。

---

## 第十阶段：Input ANR 到底盯着哪里

这是你最初切入 Input 系统时关注的点，也是最适合反向串起整条链路的问题。

### ANR 不是盯谁“收到事件”

Input ANR 不是在问：

- 内核有没有收到事件
- `EventHub` 有没有读取
- `InputReader` 有没有翻译

它盯的是：

- `InputDispatcher` 已经把事件发给某个窗口对应的 `Connection`
- 但应用迟迟没有完成这次输入处理确认

### 关键中间对象：`DispatchEntry`

当事件即将发往某个目标窗口时，`InputDispatcher` 会创建一个 `DispatchEntry`。

它可以理解为：

- 某个目标窗口的一次“在途包裹”
- 里面记录了：
  - 对应的事件
  - 目标连接
  - 分发时间
  - 超时信息

### 为什么 `finishInputEvent` 能解除 ANR 风险

因为分发链路的最后，应用会通过对应机制向系统回告：

- 这个事件我处理完了

系统收到这个完成信号后，就能把相应的在途记录移出队列，说明：

- 这个窗口没有卡在这次输入上
- 这次输入不再构成 ANR 候选

### `processAnrsLocked` 在做什么

它本质上是在看：

- 某个连接队列头部的事件是否已经超时
- 如果超时还没完成，就说明这个窗口可能卡住了

因此它更像一个“超时巡检器”。

这一阶段的关键理解：

- Input ANR 的观察点在 `InputDispatcher -> Connection -> 应用确认完成` 这一段
- `finishInputEvent` 不是“重新激活输入系统”，而是“告诉系统这笔账结清了”

---

## 第十一阶段：批处理、合并、长按、重复键这些机制放在哪一层

这一部分最适合用“谁最适合做这件事”来记。

### 1. Motion 事件批处理

Native 侧更适合做 batching，因为它能减少 JNI 往返次数。

### 2. Java 层合并成最终 `MotionEvent`

Java 层更适合把一批输入变成一个带历史轨迹的 `MotionEvent`，因为应用侧最终消费的是这个对象模型。

### 3. 键盘长按与重复键

这是更偏系统级通用行为，所以通常由 `InputDispatcher` 处理更合理。

### 4. 触摸长按

这是强上下文相关行为，比如：

- 图标长按
- 文本选择长按
- 自定义手势长按

所以放在 View 体系里更合理。

这一阶段的关键理解：

- 判断一个机制该放哪层，不看“能不能做”，而看“谁最懂上下文，谁最适合统一处理”。

---

## 关键类职责速查表

### Native 输入采集与翻译

- `EventHub`
  负责监控 `/dev/input/eventX`，读取原始内核事件。
- `InputReader`
  负责把原始协议翻译成 Android 高层输入通知。
- `InputDevice`
  表示一个 framework 层输入设备实例，管理设备状态和 mapper 集合。
- `InputMapper`
  针对具体协议语义做状态机翻译，例如键盘、触摸。

### Native 输入分发

- `InputDispatcher`
  输入分发总调度中心，负责目标选择、排队、连接管理、超时监控。
- `EventEntry` / `KeyEntry` / `MotionEntry`
  Dispatcher 内部使用的事件管理对象。
- `DispatchEntry`
  一次“发往某个目标连接”的在途分发记录，也是 ANR 追踪的重要载体。
- `Connection`
  表示某个窗口对应的一条输入连接状态。
- `FocusResolver`
  帮助按 display 解析焦点目标。

### Java 服务与窗口管理

- `InputManagerService`
  Java 层输入服务门面，连接 native 输入子系统与系统服务层。
- `WindowManagerService`
  管理窗口、焦点、层级和可接收输入的目标窗口信息。
- `PhoneWindowManager`
  系统策略层，处理 Home、Power 等系统级按键策略。

### 应用进程接收与分发

- `ViewRootImpl`
  应用窗口总入口，接收输入并驱动 View 树分发。
- `InputChannel`
  系统与应用之间的输入通信通道。
- `InputConsumer`
  应用侧从 channel 读取消息并构造成 `InputEvent`。
- `QueuedInputEvent`
  `ViewRootImpl` 内部排队节点，帮助应用在主线程上有序处理输入。

---

## 为什么这套知识以前容易“串不起来”

通常是因为少了下面这 6 个因果钉子：

1. 没把“协议翻译”和“目标分发”分开。
2. 没把“窗口”与“Activity”分开。
3. 没把“跨进程送达应用”和“应用内部 View 分发”分开。
4. 没把“事件发出”与“事件完成确认”分开。
5. 没把 `DispatchEntry` / `Connection` 这类 ANR 关键对象纳入主线。
6. 只记类名，不记每个类的输入、输出、状态和阻塞点。

以后复习时，建议每一层都问自己 4 个问题：

- 这一层的输入是什么。
- 这一层的输出是什么。
- 这一层的状态由谁维护。
- 这一层的阻塞或超时由谁观察。

只要这 4 个问题能答清，输入系统就不会再散。

---

## 一页复习抓手

把整条链路压缩成一句话：

`内核产出协议事件，EventHub 读取，InputReader 翻译，InputDispatcher 定位目标窗口并通过 InputChannel 发给应用，应用经 ViewRootImpl 和 View 树处理后回告 finish，Dispatcher 以此判断是否超时。`

---

## 源码指路地图

- Native 设备读取：`EventHub.cpp`
- Native 事件翻译：`InputReader.cpp`
- 设备与 mapper：`InputDevice.cpp`、各类 `InputMapper`
- Native 分发核心：`InputDispatcher.cpp`
- Java 服务入口：`InputManagerService.java`
- 窗口管理：`WindowManagerService.java`
- 应用输入总入口：`ViewRootImpl.java`
- 事件对象：`InputEvent.java`、`MotionEvent.java`、`KeyEvent.java`
