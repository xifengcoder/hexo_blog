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

```java
/**
 * 围绕xyz组成的轴旋转角度a创建一个矩阵rm，把结果放到矩阵rm中。
 * Creates a matrix for rotation by angle a (in degrees) around the axis (x, y, z).
 */

public static void setRotateM(float[] rm, int rmOffset,
            float a, float x, float y, float z) {
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



##### 向量（Vectors）

向量的长度

![vector_length](/images/vector_length.png)

<center><font size=2  color=gray>图1：LeakCanary内存泄漏检测图</font></center>

有一个特殊类型的向量叫做单位向量(Unit Vector)。单位向量有一个特别的性质——它的长度是1。我们可以用任意向量的每个分量除以向量的长度得到它的单位向量n̂ ：

![unit_vector](/images/unit_vector.png)

我们把这种方法叫做一个向量的标准化(Normalizing)。单位向量头上有一个^样子的记号。通常单位向量会变得很有用，特别是在我们只关心方向不关心长度的时候（如果改变向量的长度，它的方向并不会改变）。



向量点乘

两个向量的点乘等于它们的数乘结果乘以两个向量之间夹角的余弦值。

![vector_dot_product](/images/vector_dot_product.png)

#### 叉乘

叉乘只在3D空间中有定义，它需要两个不平行向量作为输入，生成一个正交于两个输入向量的第三个向量。如果输入的两个向量也是正交的，那么叉乘之后将会产生3个互相正交的向量。接下来的教程中这会非常有用。下面的图片展示了3D空间中叉乘的样子：

![vector_cross_product](/images/vector_cross_product1.png)

不同于其他运算，如果你没有钻研过线性代数，可能会觉得叉乘很反直觉，所以只记住公式就没问题啦（记不住也没问题）。下面你会看到两个正交向量A和B叉积：

![vector_cross_product2](/images/vector_cross_product2.png)

####

##### 标量（Scalar）







#### 初识纹理映射

纹理（Texture）是一个2D图片（甚至也有1D和3D的纹理），它可以用来添加物体的细节；你可以想象纹理是一张绘有砖块的纸，无缝折叠贴合到你的3D的房子上，这样你的房子看起来就像有砖墙外表了。因为我们可以在一张图片上插入非常多的细节，这样就可以让物体非常精细而不用指定额外的顶点。

为了能够把纹理映射(Map)到相应的几何图元（geometric primitive），就需要为图元中的每个顶点指定恰当的纹理坐标（Texture Coordinate），用来标明该从纹理图像的哪个部分采样。



##### Vertex Shader

 ```c
 uniform mat4 uMVPMatrix; //总变换矩阵
 attribute vec3 aPosition;  //顶点位置
 attribute vec2 aTexCoor;    //顶点纹理坐标(Texture Coordinate)
 varying vec2 vTextureCoord;  //用于传递给Fragment Shader的varying变量
 void main() { 
    gl_Position = uMVPMatrix * vec4(aPosition,1); //根据总变换矩阵计算此次绘制此顶点位置
    vTextureCoord = aTexCoor;//将接收的纹理坐标(Texture Coordinate)传递给Fragment Shader
 }   
 ```

第7行将被处理顶点的纹理坐标（Texture Coordinate）从attribute变量aTexCoor赋值给了varying变量vTextureCoord，供渲染管线（Render pipeline）进行插值计算后传递个Fragment Shader使用。



##### Fragment Shader

```C
precision mediump float; //指定默认浮点精度
varying vec2 vTextureCoord; //接收从Vertex Shader过来的纹理坐标varying变量
uniform sampler2D sTexture; //纹理采样器，代表一副纹理
void main() {           
   //给此片元从纹理中采样出颜色值            
   gl_FragColor = texture2D(sTexture, vTextureCoord); 
}              
```

此Fragment Shader的主要功能根据从Vertex Shader传递过来的varying变量vTextureCoord，调用texture2D内置函数从采样器中进行纹理采样，得到此片元的颜色。最后，将采样到的颜色值传给内置变量gl_FragColor，完成片元的着色。











#### 纹理采样

所谓纹理采样就是根据片元（Fragment）纹理坐标（Texture Coordinate）到纹理图中提取对应位置的颜色的过程。由于被渲染图元（Primitive）中的片元数量与其对应纹理区域中像素的数量并不一定相同，也就是说图元中的片元与纹理图中的像素并不总是一一对应的。





## 纹理过滤

GL_NEAREST（也叫邻近过滤，Nearest Neighbor Filtering）是OpenGL默认的纹理过滤方式。当设置为GL_NEAREST的时候，OpenGL会选择中心点最接近纹理坐标的那个像素。下图中你可以看到四个像素，加号代表纹理坐标。左上角那个纹理像素的中心距离纹理坐标最近，所以它会被选择为样本颜色：

![img](https://learnopengl-cn.readthedocs.io/zh/latest/img/01/06/filter_nearest.png)

GL_LINEAR（也叫线性过滤，(Bi)linear Filtering）它会基于纹理坐标附近的纹理像素，计算出一个插值，近似出这些纹理像素之间的颜色。一个纹理像素的中心距离纹理坐标越近，那么这个纹理像素的颜色对最终的样本颜色的贡献越大。下图中你可以看到返回的颜色是邻近像素的混合色：

![img](https://learnopengl-cn.readthedocs.io/zh/latest/img/01/06/filter_linear.png)
