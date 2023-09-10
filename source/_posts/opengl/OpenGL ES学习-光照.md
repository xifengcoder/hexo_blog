---
title: OpenGL学习-光照
urlname: opengl_lighting
date: 2023-01-25 23:15:41
tags: OpenGLES
categories: OpenGLES
description: OpenGLES
---

![img](https://img-blog.csdnimg.cn/d1cef96742da4121860d648ba2575f8e.png)

#### 环境光（Ambient）

环境光照：物体永远不会是完全黑暗，使用一个环境光照常量，永远给物体一些颜色。

​							环境光照射结果 = 材质的反射系数 X 环境光照强度

Vertex Shader

```java
uniform mat4 uMVPMatrix; //总变换矩阵
attribute vec3 aPosition;  //顶点位置
varying vec3 vPosition;//用于传递给片元着色器的顶点位置
varying vec4 vAmbient;//用于传递给片元着色器的环境光分量
void main()
{
   //根据总变换矩阵计算此次绘制此顶点位置
   gl_Position = uMVPMatrix * vec4(aPosition,1);
   //将顶点的位置传给片元着色器
   vPosition = aPosition;
   //将的环境光分量传给片元着色器
   vAmbient = vec4(0.15,0.15,0.15,1.0);
}
```

Fragment Shader

```java
precision mediump float;
uniform float uR;
varying vec3 vPosition;//接收从顶点着色器过来的顶点位置
varying vec4 vAmbient;//接收从顶点着色器过来的环境光分量
void main()                         
{
   vec3 color;
   float n = 8.0;//一个坐标分量分的总份数
   float span = 2.0 * uR / n;//每一份的长度
   //每一维在立方体内的行列数
   int i = int((vPosition.x + uR)/span);
   int j = int((vPosition.y + uR)/span);
   int k = int((vPosition.z + uR)/span);
   //计算当点应位于白色块还是黑色块中
   int whichColor = int(mod(float(i + j + k), 2.0));
   if(whichColor == 1) {//奇数时为红色
   		color = vec3(0.678, 0.231, 0.129);//红色
   }
   else {//偶数时为白色
   		color = vec3(1.0,1.0,1.0);//白色
   }
   //最终颜色
   vec4 finalColor=vec4(color,0);
   //给此片元颜色值
   gl_FragColor= finalColor * vAmbient;
}     
```

#### 散射光（Diffuse）

漫反射光照：模拟光源对物体方向性影响。物体某一部分越是对着光源越亮。


#### 镜面光（Specular）

```c
uniform mat4 uMVPMatrix; 						//总变换矩阵
uniform mat4 uMMatrix; 							//变换矩阵(包括平移、旋转、缩放)
uniform vec3 uLightLocation;						//定位光源位置
attribute vec3 aPosition;  						//顶点位置
attribute vec3 aNormal;    						//顶点法向量
varying vec3 vPosition;							//用于传递给片元着色器的顶点位置
varying vec4 vDiffuse;							//用于传递给片元着色器的散射光分量

void main(){
   vec3 normal = normalize(aNormal); //法向量
   vec4 lightDiffuse = vec4(0.8,0.8,0.8,1.0); //散射光强度
   //计算顶点在世界空间中的位置
   vec3 fragPos = vec3(uMMatrix * vec4(aPosition, 1));
   //计算光源和片元位置之间的方向向量
   vec3 lightDirection = normalize(uLightLocation - fragPos);
   vDiffuse = lightDiffuse * max(0.0, dot(normal, lightDirection)); //计算散射光的最终强度
   vPosition = aPosition; //将顶点的位置传给片元着色器
   gl_Position = uMVPMatrix * vec4(aPosition,1); //根据总变换矩阵计算此次绘制此顶点的位置
}
```

注意：

从表面顶点到光源位置的向量的计算：

首先，需要计算出顶点在世界空间中的位置。我们可以通过把顶点位置属性乘以模型矩阵(Model Matrix, 只用模型矩阵不需要用观察和投影矩阵)来把它变换到世界空间坐标。

即：uMMatrix * vec4(aPosition, 1)



#### 镜面光照

