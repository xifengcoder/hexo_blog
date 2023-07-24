---
title: OpenGL学习-光照
urlname: opengl_lighting
date: 2023-01-25 23:15:41
tags: OpenGLES
categories: OpenGLES
description: OpenGLES
---

#### 漫反射光照

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

