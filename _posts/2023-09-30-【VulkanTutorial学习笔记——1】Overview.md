---
layout:     post
title:      "【VulkanTutorial学习笔记——1】Overview"
subtitle:   ""
date:       2023-09-30 12:00:00
author:     "recaeee"
header-img: "img/in-post/default-bg.jpg"
tags:
    - Vulkan
---
# 【VulkanTutorial学习笔记——1】Overview


![20230930125559](https://raw.githubusercontent.com/recaeee/PicGo/main/20230930125559.png)



#### 0 写在前面
本系列为个人在跟随Vulkan Tutorial学习过程中记录的一些笔记，一是防止自己看着看着弃坑，二是希望能在看过之后多少留些印象，三是想在啥时候忘记了能快速地回忆一些Vulkan知识点。

出于以上这些目的，该系列的学习笔记可能会写得较随意一些，并且对于主观判断不重要的东西可能会略过，因此也不保证该系列笔记的学习质量，可能有些东西不经过脑子就会写出来。

我是一个Vulkan的小白，对于Vulkan尚不熟悉，笔记中可能存在错误，或者概念不明确的地方，也希望看的人可以批评斧正~

原教程链接：https://vulkan-tutorial.com/Introduction
本章节链接：https://vulkan-tutorial.com/Overview

#### 1 绘制一个三角形 What it takes to draw a triangle
首先来了解一下使用Vulkan来绘制一个三角形需要了解的概念（任何图形API永远不变的新手村）。

##### 1.1 Instance and physical device selection
一个Vulkan应用通过一个**VkInstance**来配置Vulkan API。我们通过**描述我们的应用以及任何我们需要用到的API extensions来创建一个Instance**。创建Instance之后，我们可以查询Vulkan支持的**显卡硬件**并且选择一个或多个**VkPhysicalDevice**来进行操作（也就是说，VkPhysicalDevice对应显卡硬件）。我们可以通过查询一些属性，比如VRAM大小、设备功能来选择想用的设备，例如例如更想用独显而非核显。

##### 1.2 Logical devices and queue families
在选择了正确的硬件设备后，我们需要创建一个**VkDevice（逻辑设备）**，在其中，我们更具体地定义我们将使用哪些**VkPhysicalDeviceFeatures（特性）**，比如多视口渲染和64bit floats。我们也需要指定哪个**Queue families**来使用。Vulkan中的大部分操作，比如绘制指令和内存操作，都会通过将他们发送到一个*VkQueue*来异步执行。Queues从Queue families中被分配，每一个Queue Family都支持特定的一系列操作集在在它的queue中。比如，Queue family可以被分为Graphics、Compute和Memory Transfer Operations。Queue families的可用性同样也是一个选择物理设备时可判别的因素。一个支持Vulkan的设备是有可能不支持任何图形功能的，但现今所有支持Vulkan的显卡基本都支持我们感兴趣的所有的Queue操作。

##### 1.3 Window surface and swap chain
除非我们只对离屏渲染感兴趣，否则我们需要**创建一个Window来呈现渲染出来的图像**。Window的创建可以通过原生平台的API或者库，例如GLFW（学习OpenGL时也使用过这个）或者SDL。在该教程中将会使用GLFW。

我们还需要两个组件来实际去渲染一个Window：a window surface(**VkSurfaceKHR**)和a swap chain(**VkSwapchainKHR**)。注意，这些**带KHR后缀的Objects是Vulkan extension的一部分**。Vulkan API本身完全和平台无关，所以我们需要使用标准化的WSI(Window System Interface)拓展来与窗口管理器交互。**Surface是对要渲染的windows的一个跨平台抽象**，其在被实例化时通常会提供一个对Native window handle的引用，例如Windows平台上的HWND。GLFW库已经实现了一个Built-in的方法来处理不同平台的特性。

**Swap Chain**是一个Render Targets的集合。其基本**作用是确保我们当前正在渲染的Iamge并不是当前正显示在屏幕上的Image**（也就是双缓冲机制吧）。保证只有完全渲染完毕的Image才会被显示是很重要的。每次我们需要绘制一帧时，我们都需要让Swap Chain提供给我们一张Image来用于渲染。**当我们绘制完一帧后，这张Image会被返回到Swap Chain，以便在某些时间点来显示到屏幕上**。Render Targets的数量和将渲染完的Image显示到屏幕上的条件取决于Present Mode。常见的Present modes包括Double buffering(vsync)和Triple buffering。

**某些平台允许我们直接渲染到Display，不需要通过VK_KHR_display和VK_KHR_display_swapchain拓展与平台的Window Manager交互**。这允许我们创建一个代表全屏的一个Surface，并且可以用于实现自己的Window Manager。

##### 1.4 Image views and framebuffers
为了绘制从Swap Chain获取到的Image，我们需要将这张Image包装到一个VkImageView和VkFramebuffer。**一个ImageView用来引用当前Image的特定部分，一个Framebuffer用于引用用于Color、Depth、Stencil的Image Views**。因为在Swap Chain中会存在许多不同的Images，因此我们会预先为每一个Image创建一个ImageView和Framebuffer，并且在绘制时选择正确的一个。

##### 1.5 Render passes
**Vulkan中的Render pass用于描述渲染操作期间使用的Image type**，其决定了一些信息，如这些Images将会如何被使用和如何处理其内容。在绘制三角形的教程中，我们将会告诉Vulkan，我们将会使用一个Image作为Color Target，然后我们想在绘制操作执行前先使用一个固有色来清理它。一个Render pass仅仅描述Images的type，而一个VkFramebuffer实际上将特定的Images绑定到这些Slots上。（可以理解为Render pass与Image无关，只有在绑定到一个Image上才生效？）（Render passes用于确定Render target和其用途？）

##### 1.6 Graphics pipeline
Vulkan中的Graphics pipeline是通过创建一个**VkPipeline**对象来建立的。它描述了**显卡的可配置状态，例如视口大小、深度缓冲操作和使用VkShaderModule对象的可编程状态**。VkShaderModule对象是从Shader字节代码创建的。驱动同时也需要知道哪几张RenderTarget将会在Pipeline中被用到，我们通过引用Render pass来确定。

Vulkan相比于其他现存的图形API一个最大的显著不同点在于，几乎所有Graphics pipeline的配置都需要提前设置。这意味着如果我们需要切换到另一个不同的Shader或者改变定点布局，我们需要完全地重建Graphics pipeline。这意味着**我们需要预先为所有我们需要用到的渲染操作的不同组合预先创建好所有VkPipeline**。只有一些基础的配置，例如视口大小和清理用的颜色可以被动态改变。所有的State也都需要被显式地描述，例如颜色混合状态。

这么做有一个好处，因为预先创建Pileline相当于做了提前编译，而不是JIT（即时编译），因此驱动程序有更多优化的机会，并且运行时性能更容易预测，因为较大的State改变（例如切换到另一个Graphics pipeline）是很明确的。

##### 1.7 Command pools and command buffers
正如之前提到的，在VUlkan中我们想要执行的许多操作，例如绘制操作，需要被提交到一个Queue。这些操作在它们被提交前需要先录制到一个**VkCommandBuffer**中。这些Command buffers是从一个**关联到特定Queue Family的VkCommand Pool**中被分配的。为了绘制一个简单的三角形，我们需要在一个Command buffer中录制以下操作：
1. Begin the render pass
2. Bind the graphics pipeline
3. Draw 3 vertices
4. End the render pass

因为Framebuffer中的Image取决于Swap chain提供给我们的特定Image，所以我们需要为每个可能的Image录制一个Command buffer，并在绘制时选择正确的Image。另一个选择是每一帧重新录制Command buffer，这是不高效的。

##### 1.8 Main loop
由于绘制命令都被包装到了一个Command buffer中，因此主循环非常简单。首先，我们**通过vkAcquireNextImageKHR来从Swap chain中获取一张Image**。然后我们**为这张Image选择一个合适的Command buffer，并且通过vkQueueSubmit来执行它**。最后，我们**把这张Image返回给Swap chain，并通过vkQueuePresentKHR来将其显示到屏幕上**。

被提交到Queue中的操作将会被**异步执行**。因此我们需要**使用同步对象**，例如Semaphores（简单中译为信号量），来**确保正确的执行顺序**。绘制操作的执行必须被设置为等待Image acquisition完成后执行，否则会出现我们对一张正在被读取用于显示到屏幕上的Image进行渲染的情况。vkQueuePresentKHR的调用需要等待渲染完成，所以我们会使用第二个Semaphore来标记渲染完成。

##### 1.9 Summary
以上内容可以在绘制第一个三角形前给我们一些Vulkan工作流程上的基础理解。实际上的程序包含了更多的阶段，例如分配Vertex buffers、创建Uniform buffers和上传Texture images。为了简单入门，一开始，我们将在顶点着色器中确定顶点坐标，而不是使用一个Vertex buffer。这是因为管理Vertex buffers需要先熟悉Command buffers。

简单来说，为了绘制一个三角形，我们需要：
1. Create a **VkInstance**
2. Select a supported graphics card(**VkPhysicalDevices**显卡硬件)
3. Create a **VkDevice**(逻辑设备) and **VkQueue**(用于提交渲染) for drawing and presentation
4. Create a **window(用于呈现Image)**, **window surface(考虑到不同平台，对window的一个抽象)** and **swap chain(双缓冲、三缓冲等Present mode)**
5. Wrap the swap chain images into **VkImageView(引用Image的特定部分？)**
6. Create a **render pass** that specifies the render tarfgets and usage(确定render target和其用途)
7. Create **framebuffers**(指向一系列ImageView) for the render pass
8. Set up the **graphics pipeline**
9. Allocate and record a **command buffer** with the draw commands for every possible swap chain image
10. Draw frames by acquiring images, submitting the right draw command buffer and returning the images back to the swap chain

这里有许多阶段，但是每个阶段的目的都非常简单明确，后续章节会做解释。如果对单个步骤与整个程序的关系感到疑惑，可以回来参考本章。

#### 2 一些API概念 API concepts

本章节最后将简述Vulkan API如何在一个Lower level构建。

##### 2.1 代码约定 Coding conventions
所有Vulkan方法、枚举和结构体都将会被定义到vulkan.h中，vulkan.h包含在LunarG开发的Vulkan SDK中。我们将在下一章节安装该SDK。

所有Function包含一个**vk前缀**，类型例如枚举和结构体拥有一个**Vk前缀**，枚举值拥有**VK_前缀**。API使用了大量结构体来给Funtion提供参数。例如，对象创建一般遵循以下模式：
```c++
VkXXXCreateInfo createInfo{};
createInfo.sType = VK_STRUCTURE_TYPE_XXX_CREATE_INFO;
createInfo.pNext = nullptr;
createInfo.foo = ...;
createInfo.bar = ...;

VkXXX object;
if(vkCreateXXX(&createInfo, nullptr, &object) != VK_SUCCESS)
{
    std::cerr << "failed to create object" << std:endl;
    return false;
}
```
Vulkan中的许多结构都需要我们在sType成员中显式指定结构的类型。pNext成员可以指向拓展的一个结构，在该教程中会始终为nullptr。创建或销毁对象的Function中将会有一个**VkAllocationCallbacks**参数，其允许我们使用驱动程序内存的自定义分配器，在本教程中也将保留为nullptr。

几乎所有functions都会返回一个**VkResult**，其值可能为VK_SUCCESS或者一个Error code。规范中描述了每个函数可以返回哪些Error codes和其含义。

##### 2.2 验证层Validation layers
Vulkan旨在实现高性能和低驱动程序开销。因此，其包含了非常有限的错误检查和Debug功能。当我们做错了一些事情时，驱动经常会Crash而不是返回Error Code，或者在一部分显卡上无法运作（兼容性问题）。

Vulkan允许我们通过**Validation layers**来做一些更广泛的检查。Validation layers是一系列会被插入到Vulkan API中的代码，其会让驱动在函数参数中执行额外的检查，并且追踪内存管理问题。比较好的一点是，我们可以在开发期间一直启用它，并在release时完全关闭它们而不造成任何开销。任何人都可以自己写Validation layers，但是LunarG的Vulkan SDK提供了一个validation的标准集合，将在本教程中使用。我们也需要注册一个回调函数来接收这些Layers的debug信息。

由于Vulkan对于任何操作都非常显式明确，并且Validation layers功能非常强大，因此它实际上可以比OpenGL或者D3D更容易找到渲染出错的原因。

#### 参考
1. 题图来自画师wlop