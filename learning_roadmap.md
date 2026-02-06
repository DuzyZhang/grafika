# Grafika 学习路线图 (Grafika Learning Roadmap)

作为一个图形领域的初学者，直接阅读 Grafika 的所有代码可能会让人感到不知所措。这个项目涵盖了 Android 图形系统的许多方面：Canvas、OpenGL ES、MediaCodec、SurfaceView vs TextureView、EGL 上下文管理等。

为了最有效地学习，建议按照以下顺序进行学习。这个路线图从基础的 Surface 概念开始，逐步过渡到 OpenGL，最终进入视频编解码和高级系统特性。

## 第一阶段：理解 Surface 和 Canvas (基础)

在接触 OpenGL 之前，首先要理解 Android 的绘图基础：`Surface` 是什么，以及如何在上面绘制。

### 1. ColorBarActivity
*   **难度**: ⭐ (入门)
*   **核心任务**: 学习 `SurfaceView` 和 `SurfaceHolder`。
*   **先决知识**:
    *   Android Activity 基本生命周期。
    *   理解 View 系统。
*   **学习重点**:
    *   `SurfaceView` 有两个部分：View 本身（在 UI 线程）和背后的 Surface（由 SurfaceFlinger 合成）。
    *   `SurfaceHolder.Callback`: 监听 Surface 的创建 (`surfaceCreated`)、改变 (`surfaceChanged`) 和销毁 (`surfaceDestroyed`)。这是所有图形应用的基础。
    *   `lockCanvas()` 和 `unlockCanvasAndPost()`: CPU 绘图的基本循环。

### 2. TextureViewCanvasActivity
*   **难度**: ⭐⭐
*   **核心任务**: 在 `TextureView` 上使用 Canvas 绘图。
*   **先决知识**:
    *   理解 `SurfaceView` 和普通 View 的区别（TextureView 像一个普通 View，可以旋转、透明，但必须有硬件加速窗口）。
*   **学习重点**:
    *   `TextureView.SurfaceTextureListener`: 类似于 SurfaceHolder 的回调。
    *   `new Surface(surfaceTexture)`: 如何从 `SurfaceTexture` 创建一个 `Surface` 对象用于 Canvas 绘图。
    *   **思考**: 为什么这里需要自己手动开启一个渲染线程？(对比 `ColorBarActivity` 也会发现类似模式)。

---

## 第二阶段：OpenGL ES 入门 (EGL 环境搭建)

现在开始进入 GPU 绘图的世界。在画出任何东西之前，必须先搭建 EGL 环境。

### 3. GlesInfoActivity
*   **难度**: ⭐⭐
*   **核心任务**: 搭建 EGL 上下文，不显示任何 UI，只查询信息。
*   **先决知识**:
    *   什么是 OpenGL ES (GLES)？(它是绘图 API)。
    *   什么是 EGL？(它是连接 Android 窗口系统和 GLES 的桥梁)。
*   **学习重点**:
    *   `EglCore`: 学习 Grafika 封装的这个核心类。这是 Grafika 中最重要的工具类。
    *   `OffscreenSurface`: 即使没有屏幕，也可以创建一个 "Pbuffer" 表面来进行 GL 调用。
    *   理解 `eglMakeCurrent`: 切换当前线程的 GL 上下文。

### 4. TextureViewGLActivity
*   **难度**: ⭐⭐⭐
*   **核心任务**: 在 `TextureView` 上使用 OpenGL ES 绘图。
*   **先决知识**:
    *   基本的 GLES 命令 (`glClear`, `glViewport`)。
*   **学习重点**:
    *   **生产者-消费者模式**: 你的渲染线程是 GL 生产者，TextureView 是消费者。
    *   `WindowSurface`: 将 EGL 和 `SurfaceTexture` 绑定，使得 GL 绘制的内容能显示在 View 上。
    *   这是最标准的自定义 GL 渲染循环结构。

---

## 第三阶段：相机与视频播放 (Surface 的流动)

这一阶段学习如何将数据源（相机、视频文件）连接到 Surface。

### 5. LiveCameraActivity
*   **难度**: ⭐⭐
*   **核心任务**: 将相机预览直接显示在 TextureView 上。
*   **先决知识**:
    *   Android Camera API (Camera1)。
*   **学习重点**:
    *   不涉及 GL 也能预览：直接将 `SurfaceTexture` 传给 `Camera.setPreviewTexture()`。
    *   理解数据是如何在系统底层流动的（Camera 硬件 -> BufferQueue -> SurfaceTexture -> 屏幕）。

### 6. PlayMovieActivity & PlayMovieSurfaceActivity
*   **难度**: ⭐⭐⭐
*   **核心任务**: 使用 `MediaCodec` 解码视频并显示。
*   **先决知识**:
    *   数字视频基础 (H.264, 关键帧)。
    *   `MediaCodec` 基础概念 (输入缓冲 -> 解码 -> 输出缓冲)。
*   **学习重点**:
    *   对比 `TextureView` (PlayMovieActivity) 和 `SurfaceView` (PlayMovieSurfaceActivity) 的实现差异。
    *   **输出 Surface**: MediaCodec 可以直接配置一个 Surface 作为输出，这样解码后的数据不需要经过 CPU，直接在 GPU/硬件层显示，效率极高。

---

## 第四阶段：GL 处理与特效 (Grafika 的核心)

这是 Grafika 最精华的部分：拦截相机或视频流，用 GL 处理（如滤镜、缩放），然后再显示。

### 7. TextureFromCameraActivity
*   **难度**: ⭐⭐⭐⭐
*   **核心任务**: 相机 -> GL 纹理 -> 修改/变换 -> 屏幕。
*   **先决知识**:
    *   纹理坐标 (UV)。
    *   矩阵变换 (MVP Matrix)。
    *   GLES 着色器 (Vertex/Fragment Shader) 基础。
*   **学习重点**:
    *   `SurfaceTexture`: 这里它作为**纹理源** (GLES 纹理)，而不仅仅是显示目标。
    *   `GL_TEXTURE_EXTERNAL_OES`: Android 相机和视频专用的纹理类型。
    *   `SurfaceTexture.OnFrameAvailableListener`: 当相机有新帧时，通知 GL 线程去重绘。

### 8. RecordFBOActivity
*   **难度**: ⭐⭐⭐⭐
*   **核心任务**: 渲染到帧缓冲区对象 (FBO)，实现离屏绘制。
*   **先决知识**:
    *   Framebuffer Object (FBO) 概念。
*   **学习重点**:
    *   **三次拷贝**: 绘制到 FBO (纹理) -> 绘制到屏幕 -> 绘制到编码器。
    *   如何保持渲染与录制分辨率的解耦。

---

## 第五阶段：视频编码与录制 (MediaCodec 编码)

如何将我们画好的或者处理好的画面保存为 MP4？

### 9. SoftInputSurfaceActivity
*   **难度**: ⭐⭐⭐
*   **核心任务**: CPU 绘图 -> 视频编码。
*   **先决知识**: 
    *   `MediaMuxer` (混合器)。
*   **学习重点**:
    *   `MediaCodec.createInputSurface()`: 这是一个神奇的方法，它给你一个 Surface，你往上面画这一帧，它就把这一帧拿去编码。
    *   这是理解“渲染到视频”的最简模型。

### 10. CameraCaptureActivity
*   **难度**: ⭐⭐⭐⭐⭐ (大Boss)
*   **核心任务**: 实时预览 + 实时录像 + 滤镜。
*   **先决知识**: 
    *   前面所有的知识总和。
    *   EGL 共享上下文 (`EGL_CONTEXT_CLIENT_VERSION`)。
*   **学习重点**:
    *   **EGLContext 共享**: 如何在显示线程和录制线程之间共享纹理数据，避免数据拷贝。这是高性能视频应用的关键。
    *   **FullFrameRect** 和 **Texture2dProgram**: 深入理解 Grafika 的封装库，它们是如何简化着色器操作的。

---

## 第六阶段：高级系统特性 (选修)

理解 Android 图形系统的底层行为。

### 11. MultiSurfaceActivity
*   **学习**: Z-order (Z 轴顺序), `setSecure` (防截屏), 透明度合成。

### 12. HardwareScalerActivity
*   **学习**: `SurfaceHolder.setFixedSize()`。理解如果不设置这个，Surface 大小如果不匹配 View 大小会发生什么（硬件缩放 vs 软件缩放）。

### 13. ScheduledSwapActivity
*   **学习**: 深入理解 Vsync 和 `Choreographer`。如何控制帧显示的精确时间 (Presentation Time Stamp)。

### 14. ContinuousCaptureActivity
*   **学习**: 循环缓冲区 (Ring Buffer)。类似行车记录仪的功能，永远只保存最后 N 秒。
