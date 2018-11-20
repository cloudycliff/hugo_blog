---
title: "Mac上搭建OpenGL开发环境"
date: 2018-10-23T14:58:40+08:00
draft: false
tags: [OpenGL]
categories: [development]
---

Mac上搭建OpenGL开发环境
=

## 编译环境

安装xcode，然后在命令行里运行```xcode-select --install```安装命令行工具

通过```brew install cmake```安装cmake及相关工具

## GLFW

GLFW是一个开源库，为OpenGL、OpenGL ES等提供了跨平台的窗口、输入事件等的封装，简化了开发流程，使开发者可以集中精力在OpenGL本身的开发上。

首先在官网上下载源码包 [下载页](http://www.glfw.org/download.html) 这里使用的是`3.2.1`版本

解压源码后，使用`cmake .` `make` `make install`编译安装即可

## GLAD

由于OpenGL的实现是由显卡驱动提供的，所以在编译时并不能获取到函数的地址，所以需要在开发时通过函数指针的形式查找并保存这些函数。代码写起来就会很冗长，所以会有很多工具来解决这个问题。

GLAD通过一种web服务的形式解决这个问题 [官网](https://glad.dav1d.de/) 

在网站上，选择API gl Version 3.3以上，Profile选择Core，其他选项暂时使用默认值，然后点击GENERATE，下载生成的压缩包。

把压缩包内的两个文件夹include、src复制到项目目录下。

## GLM

GLM是为OpenGL提供的一套数学库

在[GLM的github上](https://github.com/g-truc/glm/tags)下载最新版的源码，这里使用的是`0.9.9.2`版本

解压后，把压缩包内的glm文件夹复制到项目下即可

## macOS Mojave的坑

在最新的mac系统上，直接编译项目会报错 `fatal error: stdio.h: No such file or directory`

不知道为啥，xcode 10没有安装`/usr/include`文件夹，但是提供了一个安装包`/Library/Developer/CommandLineTools/Packages/macOS_SDK_headers_for_macOS_10.14.pkg`，安装以后就好了。。。

## macOS Mojave的坑2

看来apple真的是要放弃OpenGL了。。现在程序跑起来会直接黑屏，必须手动移动一下才能真正的渲染，目前不确定是OpenGL库的问题还是glfw的锅。在glfw的github上找到一个 [issue](https://github.com/glfw/glfw/issues/1334)

目前临时的解决方案，是在渲染的循环中，在渲染一次后手动修改一下窗口位置。。。

```c++

    bool init = false;

    do {
        glClearColor(0.2f, 0.3f, 0.3f, 1.0f);
        glClear(GL_COLOR_BUFFER_BIT);

        glfwSwapBuffers(m_pWindow);
        glfwPollEvents();

        if(!init)
        {
            int x,y;
            glfwGetWindowPos(m_pWindow, &x, &y);
            glfwSetWindowPos(m_pWindow, x+1, y);

            init = true;
        }
    } while (!glfwWindowShouldClose(m_pWindow));
    
```
