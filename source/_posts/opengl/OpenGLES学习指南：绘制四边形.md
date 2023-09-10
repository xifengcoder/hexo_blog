---
title: OpenGLES学习指南：绘制四边形
urlname: draw_rectangle
date: 2022-10-06 12:38:58
tags: OpenGLES
categories: OpenGLES
description: Android平台使用OpenGLES绘制矩形
---

核心代码SquareRenderer类：

```java
package com.github.piasy.openglestutorial_android;

import android.opengl.GLES20;
import android.opengl.GLSurfaceView;
import android.opengl.Matrix;

import java.nio.ByteBuffer;
import java.nio.ByteOrder;
import java.nio.FloatBuffer;
import java.nio.ShortBuffer;

import javax.microedition.khronos.egl.EGLConfig;
import javax.microedition.khronos.opengles.GL10;

public class SquareRenderer implements GLSurfaceView.Renderer {
    private static final int TYPE_DRAW_ARRAYS = 1;
    private static final int TYPE_DRAW_ELEMENTS = 2;

    private int mDrawType = TYPE_DRAW_ELEMENTS;
    private static final int BYTES_PER_FLOAT = 4; //一个Float占用4Byte

    private final FloatBuffer mVertexBuffer;  //顶点位置缓存
    private final FloatBuffer mColorBuffer;  //顶点颜色缓存
    private final ShortBuffer mIndicesBuffer;  //顶点索引缓存
    private int mProgram;  //渲染程序

    private static final String VERTEX_SHADER =
            "uniform mat4 uMVPMatrix;\n"
                    + "attribute vec3 aPosition;\n"
                    + "attribute vec4 aColor;\n"
                    + "varying vec4 vColor; \n"
                    + "void main(){\n"
                    + "gl_Position = uMVPMatrix * vec4(aPosition,1);\n"
                    + "gl_PointSize=20.0;\n"
                    + "vColor = aColor;\n"
                    + "}";

    private static final String FRAGMENT_SHADER =
            "precision mediump float;\n" +
                    "varying vec4 vColor;\n" +
                    "void main() {\n" +
                    "  gl_FragColor = vColor;\n" +
                    "}";

    private final float[] mViewMatrix = new float[16];
    private final float[] mProjectMatrix = new float[16];
    private final float[] mMVPMatrix = new float[16];

    private int uMVPMatrixHandle;
    private int aPositionHandle;
    private int aColorHandle;

    //采用glDrawElements绘制时的顶点个数
    private static final float[] VERTICES = {
            -0.5f, 0.5f, 0.0f,//top left
            -0.5f, -0.5f, 0.0f, // bottom left
            0.5f, -0.5f, 0.0f, // bottom right
            0.5f, 0.5f, 0.0f // top right
    };

    //采用glDrawArrays绘制时的顶点个数
    private static final float[] VERTICES2 = {
            -0.5f, 0.5f, 0.0f,      // top left
            -0.5f, -0.5f, 0.0f,      // bottom left
            0.5f, -0.5f, 0.0f,      // bottom right

            -0.5f, 0.5f, 0.0f,      // top left
            0.5f, -0.5f, 0.0f,      // bottom right
            0.5f, 0.5f, 0.0f       // top right
    };

    /**
     * 顶点索引
     */
    private static final short[] INDICES = {
            0, 1, 2, 0, 2, 3
    };

    //四个顶点的颜色参数
    private static final float[] COLORS = {
            0.0f, 0.0f, 1.0f, 1.0f, //top left
            0.0f, 1.0f, 0.0f, 1.0f, // bottom left
            0.0f, 0.0f, 1.0f, 1.0f, // bottom right
            1.0f, 0.0f, 0.0f, 1.0f // top right
    };

    public SquareRenderer() {
        if (mDrawType == TYPE_DRAW_ARRAYS) {
            mVertexBuffer = ByteBuffer.allocateDirect(VERTICES2.length * BYTES_PER_FLOAT)
                    .order(ByteOrder.nativeOrder())
                    .asFloatBuffer();
            mVertexBuffer.put(VERTICES2);
        } else {
            mVertexBuffer = ByteBuffer.allocateDirect(VERTICES.length * BYTES_PER_FLOAT)
                    .order(ByteOrder.nativeOrder())
                    .asFloatBuffer();
            mVertexBuffer.put(VERTICES);
        }
        mVertexBuffer.position(0);

        mColorBuffer = ByteBuffer.allocateDirect(COLORS.length * BYTES_PER_FLOAT)
                .order(ByteOrder.nativeOrder())
                .asFloatBuffer();
        mColorBuffer.put(COLORS);
        mColorBuffer.position(0);

        mIndicesBuffer = ByteBuffer.allocateDirect(INDICES.length * 4)
                .order(ByteOrder.nativeOrder())
                .asShortBuffer();
        mIndicesBuffer.put(INDICES);
        mIndicesBuffer.position(0);
    }

    @Override
    public void onSurfaceCreated(GL10 gl, EGLConfig config) {
        GLES20.glClearColor(1.0f, 1.0f, 1.0f, 1.0f);
        int vertexShaderId = ShaderUtils.compileVertexShader(VERTEX_SHADER);
        int fragmentShaderId = ShaderUtils.compileFragmentShader(FRAGMENT_SHADER);
        mProgram = ShaderUtils.linkProgram(vertexShaderId, fragmentShaderId);
        GLES20.glUseProgram(mProgram);
        uMVPMatrixHandle = GLES20.glGetUniformLocation(mProgram, "uMVPMatrix");
        aPositionHandle = GLES20.glGetAttribLocation(mProgram, "aPosition");
        aColorHandle = GLES20.glGetAttribLocation(mProgram, "aColor");
    }

    @Override
    public void onSurfaceChanged(GL10 gl, int width, int height) {
        GLES20.glViewport(0, 0, width, height);
        float ratio = (float) width / height;
        Matrix.frustumM(mProjectMatrix, 0, -ratio, ratio, -1, 1, 3, 7);
        Matrix.setLookAtM(mViewMatrix, 0, 0, 0, 7.0f, 0f, 0f, 0f, 0f, 1.0f, 0.0f);
        Matrix.multiplyMM(mMVPMatrix, 0, mProjectMatrix, 0, mViewMatrix, 0);
    }

    @Override
    public void onDrawFrame(GL10 gl) {
        GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT);
        GLES20.glUniformMatrix4fv(uMVPMatrixHandle, 1, false, mMVPMatrix, 0);
        GLES20.glVertexAttribPointer(aPositionHandle, 3, GLES20.GL_FLOAT, false, 3 * 4, mVertexBuffer);
        GLES20.glEnableVertexAttribArray(aPositionHandle);
        GLES20.glVertexAttribPointer(aColorHandle, 4, GLES20.GL_FLOAT, false, 4 * 4, mColorBuffer);
        GLES20.glEnableVertexAttribArray(aColorHandle);
        if (mDrawType == TYPE_DRAW_ARRAYS) {
            GLES20.glDrawArrays(GLES20.GL_TRIANGLES, 0, 6);
        } else {
            GLES20.glDrawElements(GLES20.GL_TRIANGLES, INDICES.length, GLES20.GL_UNSIGNED_SHORT, mIndicesBuffer);
        }
        GLES20.glDisableVertexAttribArray(aPositionHandle);
        GLES20.glDisableVertexAttribArray(aColorHandle);
    }
}
```

其中ShaderUtils辅助工具类：

```java
package com.github.piasy.openglestutorial_android;

import android.opengl.GLES20;
import android.util.Log;

public class ShaderUtils {

    private static final String TAG = "ShaderUtils";

    public static int compileVertexShader(String shaderCode) {
        return compileShader(GLES20.GL_VERTEX_SHADER, shaderCode);
    }

    public static int compileFragmentShader(String shaderCode) {
        return compileShader(GLES20.GL_FRAGMENT_SHADER, shaderCode);
    }

    private static int compileShader(int type, String shaderCode) {
        final int shaderId = GLES20.glCreateShader(type);
        if (shaderId == 0) {
            return 0;
        }
        GLES20.glShaderSource(shaderId, shaderCode);
        GLES20.glCompileShader(shaderId);
        final int[] compileStatus = new int[1];
        GLES20.glGetShaderiv(shaderId, GLES20.GL_COMPILE_STATUS, compileStatus, 0);
        if (compileStatus[0] == 0) {
            String logInfo = GLES20.glGetShaderInfoLog(shaderId);
            Log.e(TAG, "compile failure, error: " + logInfo);
            GLES20.glDeleteShader(shaderId);
            return 0;
        }
        return shaderId;
    }

    public static int linkProgram(int vertexShaderId, int fragmentShaderId) {
        final int programId = GLES20.glCreateProgram();
        if (programId == 0) {
            return 0;
        }

        GLES20.glAttachShader(programId, vertexShaderId);
        GLES20.glAttachShader(programId, fragmentShaderId);
        GLES20.glLinkProgram(programId);
        final int[] linkStatus = new int[1];

        GLES20.glGetProgramiv(programId, GLES20.GL_LINK_STATUS, linkStatus, 0);
        if (linkStatus[0] == 0) {
            String logInfo = GLES20.glGetProgramInfoLog(programId);
            Log.e(TAG, "link failure, error: " + logInfo);
            GLES20.glDeleteProgram(programId);
            return 0;
        }
        return programId;
    }

    public static boolean validProgram(int programObjectId) {
        GLES20.glValidateProgram(programObjectId);
        final int[] programStatus = new int[1];
        GLES20.glGetProgramiv(programObjectId, GLES20.GL_VALIDATE_STATUS, programStatus, 0);
        return programStatus[0] != 0;
    }
}
```

