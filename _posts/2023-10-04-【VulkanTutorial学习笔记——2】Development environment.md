---
layout:     post
title:      "【VulkanTutorial学习笔记——2】Development environment"
subtitle:   ""
date:       2023-10-04 12:00:00
author:     "recaeee"
header-img: "img/in-post/default-bg.jpg"
tags:
    - Vulkan
---
# 【VulkanTutorial学习笔记——2】Development environment


![20231004002852](https://raw.githubusercontent.com/recaeee/PicGo/main/20231004002852.png)



#### 0 写在前面
本系列为个人在跟随Vulkan Tutorial学习过程中记录的一些笔记，一是防止自己看着看着弃坑，二是希望能在看过之后多少留些印象，三是想在啥时候忘记了能快速地回忆一些Vulkan知识点。

出于以上这些目的，该系列的学习笔记可能会写得较随意一些，并且对于主观判断不重要的东西可能会略过，因此也不保证该系列笔记的学习质量，可能有些东西不经过脑子就会写出来。

我是一个Vulkan的小白，对于Vulkan尚不熟悉，笔记中可能存在错误，或者概念不明确的地方，也希望看的人可以批评斧正~

原教程链接：https://vulkan-tutorial.com/Introduction
本章节链接：https://vulkan-tutorial.com/Development_environment

本篇是配置环境，主要包括了几个Vulkan的依赖库，以及配置项目，知识点较少，不过一些死去的C++回忆开始攻击我（

#### 1 Vulkan SDK
为了开发Vulkan app，我们需要的最重要的组件是**SDK**。它包括了头文件、Standard validation layers、调试工具和Vulkan函数的加载程序。加载程序会在运行时查找驱动程序中的函数，类似于OpenGL的GLFW。

SDK下载地址：[the LunarG website](https://vulkan.lunarg.com/)
我使用的版本：1.3.261.1

安装完成之后第一件事是验证我们的显卡和驱动正确支持Vulkan。在安装SDK的文件夹下，打开Bin文件夹，运行vkcube.exe，显示如下图所示（给我眼转花了）。如果收到了错误信息，请确保驱动程序是最新的，并且显卡支持Vulkan Runtime。



![20231003232418](https://raw.githubusercontent.com/recaeee/PicGo/main/20231003232418.png)



同时，该文件夹下有另一个有助于开发的程序。**glslangValidator.exe和glslc.exe程序可以用于将可读语言GLSL的Shader编译成bytecode**。Bin文件下还包含了Vulkan loader和Validation layers的二进制文件，而Lib文件夹包含了很多库。

#### 2 GLFW
正如之前所说，Vulkan本身是与平台无关的API，并且不包含**创建window来显示渲染结果的工具**。为了利用Vulkan的跨平台优势并且避开Win32，我们将**使用GLFW库来创建一个Window**，其支持Windows、Linux和MacOS平台。除了GLFW，还有其他库也可以用来创建window显示，比如SDL，但GLFW的优势在于除了创建window外，它还抽象了Vulkan中的一些其他特定于平台的内容。

[GLFW下载地址](https://www.glfw.org/download.html)

本教程中使用了64位的GLFW二进制文件。下载GLFW后，将archive解压到一个方便的地方。教程中建议在Visual Studio目录下创建一个Libraries文件夹来存放。

#### 3 GLM
不像DX12，Vulkan并不包含一个**线性代数计算的库**，所以我们需要下载一个。**GLM**是一个被设计用于图形API的数学库，并且也通常搭配OpenGL食用。

GLM下载地址

GLM是一个仅包含Header的库，因此只需要下载最新版，并且放到Libraries下就行了（目录结构参考原教程就行）。

#### 4 配置Visual Studio Setting up Visual Studio

目前我们安装Vulkan所有需要的依赖项，接下来配置Visual Studio中的环境。这里跟着原教程图文教程走就行，这里简单提一下就过去了。

我使用的Visual Studio版本：2022

创建一个windows桌面向导程序（Windows Desktop Wizard），应用类型选择控制台应用（Console Application），勾选Empty Project。

新建一个Main.cpp，先键入下列代码（好熟悉的风格，梦回OpenGL Tutorial。

```c++
#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>

#define GLM_FORCE_RADIANS
#define GLM_FORCE_DEPTH_ZERO_TO_ONE
#include <glm/vec4.hpp>
#include <glm/mat4x4.hpp>

#include <iostream>

int main() {
    glfwInit();

    glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
    GLFWwindow* window = glfwCreateWindow(800, 600, "Vulkan window", nullptr, nullptr);

    uint32_t extensionCount = 0;
    vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, nullptr);

    std::cout << extensionCount << " extensions supported\n";

    glm::mat4 matrix;
    glm::vec4 vec;
    auto test = matrix * vec;

    while(!glfwWindowShouldClose(window)) {
        glfwPollEvents();
    }

    glfwDestroyWindow(window);

    glfwTerminate();

    return 0;
}
```

接下来配置项目来解决这一堆Errors。跟着原教程图文走ing~
死去的C++回忆突然开始攻击我（
还好有原教程的保姆级教学~

在成功配置完之后，编译运行，可以得到一个命令行窗口和一个window，如下图所示。



![20231004002339](https://raw.githubusercontent.com/recaeee/PicGo/main/20231004002339.png)



好耶，配置完毕，Linux和MacOS就略了。之后就是愉快的Vulan之旅了~

#### 参考
1. 题图来自画师wlop