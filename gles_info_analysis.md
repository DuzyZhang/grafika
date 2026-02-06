# OpenGL ES 与 EGL 信息详解

这份文档结合了 `gles-info.txt` 的实际输出与 `GlesInfoActivity.java` 的源代码，详细解释了每一行信息的来源及其在图形开发中的意义。

## 1. OpenGL ES 信息 (GL Information)

这部分信息描述了你的设备 GPU 对 OpenGL ES 标准的支持情况。

### 基础厂商信息
*   **Code**: `GLES20.glGetString(GLES20.GL_VENDOR)`
*   **Output**: `vendor : Imagination Technologies`
*   **Code**: `GLES20.glGetString(GLES20.GL_RENDERER)`
*   **Output**: `renderer : PowerVR Rogue GE8320`
*   **Code**: `GLES20.glGetString(GLES20.GL_VERSION)`
*   **Output**: `version : OpenGL ES 3.2 build 1.13@5776728`
*   **解析**:
    *   **Vendor/Renderer**: 告诉你谁制造了 GPU 以及具体的型号。这里的 **PowerVR Rogue GE8320** 是一款常见于联发科芯片（如 Helio P22/A22）的入门级 GPU。
    *   **Version**: 你的设备支持到 **OpenGL ES 3.2**。这意味着你可以使用曲面细分 (Tessellation) 和几何着色器 (Geometry Shader) 等高级功能。

### 扩展列表 (Extensions)
*   **Code**: `GLES20.glGetString(GLES20.GL_EXTENSIONS)`
*   **输出示例**:
    *   `GL_EXT_texture_sRGB_decode`: 支持 sRGB 纹理自动解码（无需在 Shader 中手动转换）。
    *   `GL_OES_EGL_image_external`: **非常重要！** 这就是支持 `TextureView` 和相机预览所需的扩展，允许 GPU 直接读取 Android 的 `SurfaceTexture` 数据。
    *   `GL_EXT_debug_marker`: 允许你在 GL 命令流中插入标记，方便在 GPU 分析器（如 RenderDoc）中调试。
*   **解析**: 扩展是 GPU 对标准 OpenGL ES 的补充。如果你的程序需要某些高级特性（如特定的纹理压缩格式 `GL_IMG_texture_compression_pvrtc`），必须先检查这里是否存在。因为这是 PowerVR 的 GPU，所以你看到很多 `GL_IMG_` 开头的私有扩展。

---

## 2. EGL 信息 (EGL Information)

EGL 是 OpenGL ES 和 Android 窗口系统之间的“胶水”。

### 基础信息
*   **Code**: `eglCore.queryString(EGL14.EGL_VENDOR)`
*   **Output**: `vendor : Android`
*   **Code**: `eglCore.queryString(EGL14.EGL_VERSION)`
*   **Output**: `version : 1.4 Android META-EGL`
*   **解析**: 这里的 Vendor 是 Android，因为 Android 实现了一个 EGL 的中间层 (EGL Loader)，它会根据需要加载真正的 GPU 驱动。

### 关键扩展 (Key Extensions)
*   **Code**: `eglCore.queryString(EGL14.EGL_EXTENSIONS)`
*   **输出重点解析**:
    *   `EGL_ANDROID_presentation_time`: **核心能力！** 这个扩展允许 `ScheduledSwapActivity` 控制每一帧具体的显示时间（PTS）。没有它，你只能“尽快”显示，不可控。
    *   `EGL_ANDROID_recordable`: 允许将 EGLSurface 作为 MediaCodec 的输入进行硬编码录制。
    *   `EGL_KHR_swap_buffers_with_damage`: 允许“局部刷新”（Partial Update），告诉系统哪部分变了，只合成变了的部分，省电神技。

---

## 3. 系统信息 (System Information)

这部分不是 GL 的，而是 Android 系统的基本 Build 信息。

*   **Code**: `Build.MANUFACTURER`, `Build.MODEL`, etc.
*   **Output**:
    *   `model : G6-TY`
    *   `release : 11` (Android 11)
*   **解析**: 虽然不直接通过 GL API 获取，但对于图形调试至关重要。很多 GPU 驱动的 Bug 是特定于某个系统版本的。在做适配时，通常组合 `Renderer` + `Android Version` 来判断是否需要开启某些 Workaround（变通方案）。

## 总结
这份日志证明了你的设备：
1.  是一台 **Android 11** 设备，搭载 **PowerVR GE8320** GPU。
2.  支持先进的 **OpenGL ES 3.2**。
3.  **核心扩展齐全**: 支持 Android 图形架构所需的所有关键扩展（如 `EGL_image_external` 和 `presentation_time`），这意味着 Grafika 中的所有高级用例（视频录制、相机预览、时间戳控制）都能在这台设备上完美运行。
