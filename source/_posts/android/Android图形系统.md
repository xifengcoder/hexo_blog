---
title: Android图形系统概述
urlname: android_display
date: 2024-03-03 16:51:18
tags:
categories: Android
description:
---



Android 中从应用程序绘制 View 到最终在屏幕上显示的流程。

Android App开发者可通过三种方式将图像绘制到屏幕上：

- Canvas
- OpenGL ES
- Vulkan

**1. 应用程序层：**

- 开发者在应用程序层通过 XML 布局文件或程序代码定义 UI 元素，如 TextView、ImageView 等。这些 UI 元素最终会被封装成 View 对象。

**2. View的绘制：**

- 通过调用View的onDraw()方法进行View的绘制，

**`Canvas`** 是 Android 中的绘图工具，而 **`Bitmap`** 可以看作是 `Canvas` 的绘图目标。在绘制时，`Canvas` 可以将图形元素绘制到关联的 `Bitmap` 上。



**3. 绘制缓冲区（Drawing buffer）**

绘制的内容通常存储在一个绘制缓冲区中，这个缓冲区使用**`Bitmap`**表示。`Bitmap` 是 Android 中表示位图的类，用于存储图像的像素数据。它可以被视为一个绘图缓冲区，用于保存绘制操作的结果。

使用 `Canvas` 对象可以将图形元素绘制到 `Bitmap` 上。例如，在 `onDraw` 方法中：

```java
@Override
protected void onDraw(Canvas canvas) {
    super.onDraw(canvas);

    Bitmap bitmap = Bitmap.createBitmap(width, height, Bitmap.Config.ARGB_8888);
    Canvas bitmapCanvas = new Canvas(bitmap);

    // 在 bitmapCanvas 上绘制图形元素
    // ...

    // 将结果绘制到 View 的 Canvas 上
    canvas.drawBitmap(bitmap, 0, 0, null);
}
```

**4. Hardware Acceleration（硬件加速）**

如果启用了硬件加速，绘制的过程可能会在 GPU（图形处理单元）上执行，以提高绘制性能。Android 的硬件加速系统（Hwui）可以处理图形的光栅化和混合等操作。 

**5. DisplayList 和 RenderNode：**

- 绘制的操作可以被记录到一个 DisplayList 中，它是一系列绘制指令的容器。RenderNode 是 DisplayList 的底层实现。

**6. 图层合成：**

- View 最终与一个 Surface 相关联，Surface 包含了绘制内容的像素数据。

- SurfaceFlinger 负责合成所有的 Layers 并送显到 Display 

- SurfaceFlinger 将不同的Layers按照 Z-Order的顺序进行合成，生成最终的屏幕显示内容。每个Layer都可以是一个 View。

- SurfaceFlinger 根据 WMS 提供的窗口信息合成所有的 Layers（对应不同的Surface ），具体的合成策略由 `hwcomposer` HAL 模块决定并实施，最后也是由该模块送显到 Display，而 Gralloc 模块则负责分配图形缓冲区。

- `hwcomposer` （全称为 Hardware Composer）的主要功能是在硬件层面执行图形合成操作。它接受来自 SurfaceFlinger 的图层数据，将它们合成到帧缓冲区（Framebuffer）上，以在屏幕上显示。


`hwcomposer` 利用硬件加速来提高图形合成的性能。它能够利用 GPU等硬件加速设备执行图形操作，从而加速图层的合成和呈现

- **Layer**

Layer 是 SurfaceFlinger 中的一个概念，它表示一个图形元素或一个图形层。每个应用程序窗口、系统 UI 元素等都是一个Layer。Layers 按照它们的 Z 轴值堆叠在一起，形成最终的显示。每个 Layer 通常与一个 Surface 对象相关联，这个 Surface 包含了图层的像素数据。

- **Surface**

Surface 是Layer的一个抽象，它包含了一个或多个 Buffer，这些 Buffer 存储了图层的像素数据。

Surface由应用程序的 UI 线程或其他相关组件使用。

```java
SurfaceView surfaceView = (SurfaceView) findViewById(R.id.surfaceView);
SurfaceHolder surfaceHolder = surfaceView.getHolder();
Canvas canvas = surfaceHolder.lockCanvas();
// 在 canvas 上进行绘制操作
surfaceHolder.unlockCanvasAndPost(canvas);
```



- **Display**

Display 表示一个物理或虚拟的显示屏幕。在 Android 中，一个设备可以有多个显示屏幕，例如，主屏幕、外部显示器等。每个 Display 包含一个或多个 Surface，这些 Surface 可以显示不同的图形内容。



**BufferQueue**

用于显示 Surface 的 BufferQueue 通常配置为三重缓冲（triple-buffering）。缓冲区是按需分配的，因此，如果生产者足够缓慢地生成缓冲区（例如在 60 fps 的显示屏上以 30 fps 的速度缓冲），队列中可能只有两个分配的缓冲区。按需分配缓冲区有助于最大限度地减少内存消耗。您可以在 `dumpsys SurfaceFlinger` 的输出中看到每个层级关联的缓冲区的摘要。





