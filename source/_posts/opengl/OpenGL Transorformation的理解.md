---
title: OpenGL Transorformation的理解
urlname: Transorformation的理解
date: 2023-08-11 22:40:32
tags: OpenGLES
categories: OpenGLES
description: OpenGLES
---

#### OpenGL Transformation



#### Object Coordinates

#### Eye Coordinates

1. Model transform用来从object space转换为world space;
2. View transform用来从world space转换为eye space。

![eye_coordinates](/images/eye_coordinates.png)

#### Clip Coordinates

![clip_coordinates](/images/clip_coordinates.png)

#### Normalized Device Coordinates (NDC)

![ndc_coordinates](/images/ndc_coordinates.png)

####

![OpenGL Transformation](/images/opengl_transformation.jpg)



Matrix的perspectiveM方法可以生成一个透视投影

```java
/**
 * Defines a projection matrix in terms of a field of view angle, an
 * aspect ratio, and z clip planes.
 *
 * @param m the float array that holds the perspective matrix
 * @param offset the offset into float array m where the perspective
 *        matrix data is written
 * @param fovy field of view in y direction, in degrees
 * @param aspect width to height aspect ratio of the viewport
 * @param zNear
 * @param zFar
 */
perspectiveM(float[] m, int offset, float fovy, float aspect, float near, float far);
```

焦距：

```java
a = 1.0f / Math.tan((fovy * Math.PI / 180.0f) / 2.0f)
```




```java
    /**
     * 生成透视投影
     * Defines a projection matrix in terms of six clip planes.
     *
     * @param m 要填充矩阵元素的float[]类型数组
     * @param 要填充其实偏移量
     * @param near面的left
     * @param near面的right
     * @param near面的bottom
     * @param near面的top
     * @param near面与透视点的距离
     * @param far面与透视点的距离
     */
    public static void frustumM(float[] m, int offset, float left, float right, float bottom, float top,
            float near, float far);
```

如果仅仅使用projection matrix时

Vertex~clip~ = ProjectionMatrix * Vertex~eye~

添加了model matrix之后，

Vertexeye = ModelMatrix * Vertex~model~
vertex
Clip = ProjectionMatrix * Vertex~eye~

### Perspective Projection

![perspective_projection](/images/perspective_projection.png)


Perspective Frustum and Normalized Device Coordinates (NDC)
