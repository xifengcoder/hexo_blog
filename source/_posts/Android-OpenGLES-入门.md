---
title: Android OpenGLES 入门
urlname: opengles_android
date: 2022-04-26 22:38:10
tags: OpenGL ES
categories: Android
description: Android平台OpenGL ES2.0入门
---

```java
/**
 * 指定了渲染时索引值为index的顶点属性数组的数据格式和位置。
 * @param index 指定要修改的顶点属性的索引值。
 * @param size 指定顶点属性的大小。顶点属性是一个vec3，它由3个值组成，所以大小是3。
 * @param type 指定数据的类型，这里是GL_FLOAT。
 * @param normalized 是否希望数据被标准化(Normalize)。
 * @param stride 步长，它告诉我们在连续的顶点属性组之间的间隔。
 * @param ptr 位置数据缓冲区。
 */
public static void glVertexAttribPointer(
    int index,
    int size,
    int type,
    boolean normalized,
    int stride,
    java.nio.Buffer ptr
);
```

return the location of an attribute variable

#### OpenGL 坐标系

按照约定，OpenGL是一个右手坐标系。最基本的就是说正x轴在你的右手边，正y轴往上而正z轴是往后的。想象你的屏幕处于三个轴的中心且正z轴穿过你的屏幕朝向你。坐标系画起来如下：

![coordinate_systems_right_handed](http://learnopengl.com/img/getting-started/coordinate_systems_right_handed.png)



```java
// 实际加载纹理
GLUtils.texImage2D(GLES20.GL_TEXTURE_2D, // 纹理类型，在OpenGLES中必须为GL10.GL_TEXTURE_2D
                   0, // 纹理的层次，0表示基本图像层，可以理解为直接贴图
                   bmp, // 纹理图像
                   0 // 纹理边框尺寸
                  );
```



加载图片纹理：

```java
    /**
     * 加载图片纹理
     *
     * @param bmp
     * @return
     */
    public static int getTextureIdByBitmap(Bitmap bmp) {
        int[] textures = new int[1];
        GLES20.glGenTextures(1, textures, 0);
        int textureId = textures[0];
        GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, textureId);
        GLES20.glTexParameterf(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_MIN_FILTER, GLES20.GL_NEAREST);
        GLES20.glTexParameterf(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_MAG_FILTER, GLES20.GL_LINEAR);
        GLES20.glTexParameterf(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_WRAP_S, GLES20.GL_CLAMP_TO_EDGE);
        GLES20.glTexParameterf(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_WRAP_T, GLES20.GL_CLAMP_TO_EDGE);
        GLUtils.texImage2D(GLES20.GL_TEXTURE_2D, 0, bmp, 0);
        bmp.recycle();
        return textureId;
    }
```



```java
// 设置相机位置/设置的是 View矩阵.决定摄影机的空间位置.
// 我们把相机放在世界坐标系的(0, 0, -3)这个点，
// 然后观察的目标位置在点(0.0, 0.0, 0.0)，即世界坐标系的原点。
// 同时我们还需要指定一个「头朝上」的方向，这在代码里设置的是向量(0.0, -1.0, 0.0)指向「上」的方向。
Matrix.setLookAtM(mViewMatrix, 0, 0, 0, -3,
                0f, 0f, 0f,
                0f, -1.0f, 0.0f);


public static void setLookAtM(float[] rm, int rmOffset,
            float eyeX, float eyeY, float eyeZ,
            float centerX, float centerY, float centerZ, float upX, float upY,
            float upZ){
  ...
}
```



设置透视投影

```java
    /**
     * 设置透视投影
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
    public static void frustumM(float[] m, int offset,
            float left, float right, float bottom, float top,
            float near, float far) {
      ...
    }
```



```java
/**
 * define an array of generic vertex attribute data
 * index: mPositionHandle, Specifies the index of the generic vertex attribute to be modified.
 * size: 数据的维数(COORDS_PER_VERTEX)
 * type: 数据的类型, 即GLES20.GL_FLOAT
 * normalized: 是否需要归一化(false)
 * stride: 步长，即连续顶点偏移量(COORDS_PER_VERTEX * 4)
 * pointer: 起始位置在缓冲区的偏移量(mVertexBuffer)
 */
public static void glVertexAttribPointer(
    int indx,
    int size,
    int type,
    boolean normalized,
    int stride,
    java.nio.Buffer ptr
) {
```
