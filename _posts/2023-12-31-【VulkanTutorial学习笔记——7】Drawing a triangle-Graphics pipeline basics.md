---
layout:     post
title:      "【VulkanTutorial学习笔记——7】Drawing a triangle-Hraphics pipeline basics"
subtitle:   ""
date:       2023-12-31 12:00:00
author:     "recaeee"
header-img: "img/in-post/default-bg.jpg"
tags:
    - Vulkan
---
# 【VulkanTutorial学习笔记——7】Drawing a triangle-Graphics pipeline basics

![20231231122247](https://raw.githubusercontent.com/recaeee/PicGo/main/20231231122247.png)

#### 0 写在前面
本系列为个人在跟随Vulkan Tutorial学习过程中记录的一些笔记，一是防止自己看着看着弃坑，二是希望能在看过之后多少留些印象，三是想在啥时候忘记了能快速地回忆一些Vulkan知识点。

出于以上这些目的，该系列的学习笔记可能会写得较随意一些，并且对于主观判断不重要的东西可能会略过，因此也不保证该系列笔记的学习质量，可能有些东西不经过脑子就会写出来。

我是一个Vulkan的小白，对于Vulkan尚不熟悉，笔记中可能存在错误，或者概念不明确的地方，也希望看的人可以批评斧正~

首先在这祝小伙伴们2024年新年快乐~这也是2023年我的最后一篇文章了，回顾这一年的经历虽然多灾多难，但也从Unity的萌新，到熟练使用SRP、URP，再到开始学习Vulkan，也算是有所成长吧。今年可以说是游戏行业的寒冬吧，时常听到哪哪又裁员了，不禁让人怀疑自己到底能在游戏行业待多久。但至少我目前还有很多想学习的渲染技术，至少2024年够我学的了，就像我的steam库里还剩那么多没玩的游戏，新的一年想必也是挖坑填坑的一年，虽然每年开头都想鼓励自己，但想摸鱼是刻在DNA里的，所以新的一年也要努力摸鱼！（不是

原教程链接：
https://vulkan-tutorial.com/Introduction
本章节链接：
https://vulkan-tutorial.com/Drawing_a_triangle/Graphics_pipeline_basics/Introduction

#### 1 任务介绍 Introduction

在接下来的几个小节，我们将会建立一个图形管线**Graphics pipeline**，其会被配置用于绘制我们的第一个三角形。Graphics pipeline是一系列操作，它将网格的顶点和纹理一直传递到render targets的像素。其实就是我们了解的硬件层面的渲染管线啦~如下图所示。

![20231130211414](https://raw.githubusercontent.com/recaeee/PicGo/main/20231130211414.png)

既然说到这了，就简单复习下硬件层面的渲染管线吧。

**输入装配器input assembler**会从我们指定的缓冲区收集原始顶点数据，同时也可能使用索引缓冲区来重复利用顶点。

**顶点着色器vertex shader**对每个顶点执行矩阵变换操作，通常是从模型空间到裁剪空间，同样会传递每顶点数据到下一阶段。

**曲面细分着色器tessellation shaders**允许我们以指定规则细分几何体以提高mesh质量。通常用于使砖墙或者楼梯等表面在近距离下看起来不那么平坦。

**几何着色器geometry shader**为每个图元primitive（三角形，线，点）执行，可以对图元进行丢弃或者增加更多输出。它和曲面细分着色器很像，但是更加灵活。但是，如今它很少被使用，因为大部分显卡上它的表现都不尽如人意，除了英特尔的集显。

**光栅化阶段rasterization stage**将图元离散成片元。屏幕外的片元会被丢弃，并且顶点着色传出的属性会在片元之间插值。同时，深度检测会将被其他片元遮挡的片元剔除。在这里也复习一个最基础的概念吧，片元和像素的区别是什么，片元带有一系列属性（比如颜色、法线等），并且可能会被剔除。而framebuffer上最终的像素只包含RGB值，只是单纯的像素罢了。

**颜色混合Color blending**对映射到framebuffer上的同一个像素的多个不同片元进行一系列操作。通常可以直接覆盖、相加或者基于透明度混合。

图中绿色部分阶段被称为**Fixed-function阶段**，我们只能通过改变参数简单调整它们，但是它们的行为是被预定义好的。而图中橘色阶段为可编程的，不必多言。

在OpenGL或者D3D中，我们可以随时调用函数来改变这些管线设置，但是**在Vulkan中，graphics pipeline几乎是完全不可变的，因此如果要更改着色器、绑定不同的framebuffer或者改变颜色混合模式，我们必须从头开始重新创建管线**。**缺点是我们必须为渲染需要的所有不同排列组合创建一堆的pipelines。但是，因为我们事先知道了在管线中所有要做的事情，驱动可以做更多的优化**。

一些可编程阶段是可选的，例如我们可能不需要曲面细分着色器和几何着色器，还比如我们在渲染shadowmap时可以关闭片元着色阶段。

在接下来的小节，我们会先创建绘制三角形最基本的两个阶段：顶点着色器和片元着色器。fixed functions如混合模式、视口、光栅化会被设置。最后我们会为Vulkan中的graphics pipeline指定输入和framebuffer输出。

好了，让我们开始吧，任务出发！

我们先创建createGraphicsPipeline函数，在initVulkan中调用它。

```c++
	//**initVulkan**函数来用于实例化Vulkan objects私有成员
	void initVulkan()
	{
		createInstance();
		setupDebugMessenger();
		createSurface();
		pickPhysicalDevice();
		createLogicalDevice();
		createSwapChain();
		createImageViews();
		createGraphicsPipeline();
	}

	//创建pipeline
	void createGraphicsPipeline()
	{

	}
```

#### 2 着色器模块 Shader modules

不像早期的APIs，Vulkan中的Shader代码必须为字节码格式，而不是人类可读的语法如GLSL和HLSL。这种字节码格式被称为**SPIR-V**，被设计用于Vulkan和OpenGL（都是Khronos APIs），可以被用于graphics和compute shader。

**使用字节码的优点是由GPU厂商写的将shader转换为本地指令的解释器会更简单、快捷、统一**。过去情况表明由人类可读语法如GLSL，硬件厂商对标准的解释相当灵活。如果咱写了一个重要的着色器，对于一个厂商的GPU来说能正常工作，但咱也有很大概率在其他厂商的GPU上因为语法错误而无法编译，或者工作不正常。这些情况可以通过直接提供如SPIR-V格式的字节码来进行避免。

但我们不需要手写这些字节码。**Khronos提供了它自己的厂商无关的编译器，可以将GLSL编译成SPIR-V**。该编译器旨在验证我们的着色器代码是否完全符合标准，并且会生成SPIR-V二进制文件。我们也可以将这个编译器作为库引入以在运行时生成SPIR-V，但教程中咱不会这么做。虽然我们可以通过glslangValidator.exe来直接使用该编译器，但我们会用Google的**glslc.exe**取而代之。glslc的优点是它使用了和GCC、Clang相同的参数格式，并且包含了许多额外的功能。这两者都已经被包含在了Vulkan SDK中，所以我们不需要额外下任何东西。

**GLSL是一个使用C风格语法的Shading语言**。GLSL语法的程序会有一个main函数会对每个object调用。GLSL使用全局变量来管理输入和输出，而不是函数参数和返回值的格式。GLSL包含了许多有助于图形编程的功能，例如内置vector和matrix，向量矩阵的点积函数、反射函数等。其他关于GLSL的细节在此不做过多赘述啦，有兴趣的自己看看文档吧~

正如之前章节所说，我们需要写一个顶点着色器和一个片元着色器来绘制三角形。任务开始！

#### 3 顶点着色器 Vertex shader

顶点着色器处理传入的每个顶点，它接收顶点的属性Attributes如世界坐标、颜色、法线和纹理坐标作为输入。同时，它输出裁剪空间下的顶点坐标，并且传递需要给片元着色器的顶点属性Attributes，如颜色和纹理坐标。这些值会在光栅化阶段的片元中被插值。

中间是一些关于坐标系、硬件层面的渲染流程的基础知识，在这里不做过多翻译。Skip！

对于咱的第一个三角形，我们不会应用任何变换，而是直接确定三个顶点的NDC空间下的坐标。咱直接在顶点着色器中输出NDC坐标作为裁剪空间下的坐标。

通常来说传入的顶点数据会存储在顶点缓冲区中，但以我们目前的菜鸟水平尚不足以在Vulkan中创建顶点缓冲区并用数据填充它。所以在这里先写死在shader里。

GLSL语法在这里不多说了，直接上代码吧！

```glsl
#version 450

layout(location = 0) out vec3 fragColor;

vec2 positions[3] = vec2[](
	vec2(0.0, -0.5),
	vec2(0.5, 0.5),
	vec2(-0.5, 0.5)
);

vec3 colors[3] = vec3[](
	vec3(1.0, 0.0, 0.0),
	vec3(0.0, 1.0, 0.0),
	vec3(0.0, 0.0, 1.0)
);

void main()
{
	gl_Position = vec4(positions[gl_VertexIndex], 0.0, 1.0);
	fragColor = colors[gl_VertexIndex];
}
```

#### 4 片元着色器 Fragment shader

同上，简单的GLSL语法，上代码！

```glsl
#version 450

layout(location = 0) in vec3 fragColor;

layout(location = 0) out vec4 outColor;

void main()
{
	outColor = vec4(fragColor, 1.0);
}
```

#### 5 编译着色器 Compile Shaders

我们将顶点着色器代码和片元着色器代码保存到shader.vert和shader.frag文件中，接下来写一个bat文件来编译着色器，windows平台的bat代码如下。

```
glslcPath shader.vert -o vert.spv
glslcPath shader.frag -o frag.spv
pause
```

**这两个指令告诉编译器读取GLSL源文件并且通过-o(output)标识来输出SPIR-V字节码文件**。运行bat即可得到香喷喷的vert.spv和frag.spv文件。

如果咱的shader代码中包含语法错误，编译器会提示行号和问题。它还有些其他功能，比如**可以将字节码翻译成人类可读的格式，我们可以从中看到shader具体在做些什么操作，并且其中做了什么优化**。

从命令行编译shader是最直接的方式，Vulkan SDK也包含了libshaderc库让我们可以在程序内通过代码将GLSL编译成SPIR-V。

#### 6 加载一个着色器 Loading a shader

OK，食材准备好了（指编译得到的spv文件），锅热倒油！我们将在程序中加载编译得到的SPIR-V格式的shaders并且将其插入到Graphics pipeline中的一处。

首先**加载shader资源文件**。

```c++
	//创建pipeline
	void createGraphicsPipeline()
	{
		//加载shader资源
		auto vertShaderCode = readFile("shaders/vert.spv");
		auto fragShaderCode = readFile("shaders/frag.spv");
	}

	//读取文件，并以vector返回byte array
	static std::vector<char> readFile(const std::string& filename)
	{
		//ate表明从文件末尾开始读取
		std::ifstream file(filename, std::ios::ate | std::ios::binary);

		if (!file.is_open())
		{
			throw std::runtime_error("failed to open file!");
		}

		size_t fileSize = (size_t)file.tellg();
		std::vector<char> buffer(fileSize);
		file.seekg(0);
		file.read(buffer.data(), fileSize);
		file.close();

		return buffer;
	}
```

#### 7 创建着色器模块 Creating shader modules

在我们实际将shader code传递给pipeline之前，我们还需要将其封装到一个**VkShaderModule**对象中。

```c++
	//将shader code包装到VkShaderModule中
	VkShaderModule createShaderModule(const std::vector<char>& code)
	{
		VkShaderModuleCreateInfo createInfo{};
		createInfo.sType = VK_STRUCTURE_TYPE_SHADER_MODULE_CREATE_INFO;
		createInfo.codeSize = code.size();
		//注意需要传入uint32_t类型的指针
		createInfo.pCode = reinterpret_cast<const uint32_t*>(code.data());

		VkShaderModule shaderModule;
		if (vkCreateShaderModule(device, &createInfo, nullptr, &shaderModule) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to create shader module!");
		}

		return shaderModule;
	}
```

Shader Modules只是对shader字节码和其中定义的函数的一个薄封装。**SPIR-V字节码的编译和链接到机器码给GPU执行会在graphics pipeline实际创建时发生**。这意味着我们**可以在pipeline创建完毕后立刻销毁shader modules**。

#### 8 着色器阶段创建 Shader stage creation

要实际使用这些shaders，我们需要通过**vkPipelineShaderStageCreateInfo**结构体将它们**分配到特定的管线阶段**，，作为实际管线创建过程的一部分。

创建vkPipelineShaderStageCreateInfo的过程和之前也十分类似，在其中，我们需要指定shader被使用到的管线阶段、shaderModule以及主函数入口**entrypoint**。可以指定entrypoint意味着我们**可以在一个Shader资源文件中写多个不同的片元着色器函数**，通过使用不同的entrypoint来作为区分。

同时还有一个可选的**成员pSpecializationInfo**值得一聊，虽然我们在这里用不着它。它**允许我们确定着色器常量值**。我们可以使用单个shader module，为其中使用的常量指定不同的值，可以在创建管线的时候配置其行为。这比在渲染时使用变量配置shader更加高效，因为编译器可以做一些优化，比如省去一些依赖这些常量值的if判断语法。如果我们没有用到任何这样的常量，我们可以将其置为nullptr，这在createInfo初始化时自动会做。

```c++
	//创建pipeline
	void createGraphicsPipeline()
	{
		//加载shader资源
		auto vertShaderCode = readFile("shaders/vert.spv");
		auto fragShaderCode = readFile("shaders/frag.spv");

		//封装到Shader modules
		VkShaderModule vertShaderModule = createShaderModule(vertShaderCode);
		VkShaderModule fragShaderModule = createShaderModule(fragShaderCode);

		//将shader分配到特定管线阶段
		VkPipelineShaderStageCreateInfo vertShaderStageInfo{};
		vertShaderStageInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO;
		vertShaderStageInfo.stage = VK_SHADER_STAGE_VERTEX_BIT; // 要分配的管线阶段
		vertShaderStageInfo.module = vertShaderModule;
		vertShaderStageInfo.pName = "main"; //entrypoint

		VkPipelineShaderStageCreateInfo fragShaderStageInfo{};
		fragShaderStageInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO;
		fragShaderStageInfo.stage = VK_SHADER_STAGE_FRAGMENT_BIT; // 要分配的管线阶段
		fragShaderStageInfo.module = fragShaderModule;
		fragShaderStageInfo.pName = "main"; //entrypoint

		VkPipelineShaderStageCreateInfo shaderStages[] = { vertShaderStageInfo, fragShaderStageInfo };

		//创建完pipeline后可以立刻销毁shader modules
		vkDestroyShaderModule(device, fragShaderModule, nullptr);
		vkDestroyShaderModule(device, vertShaderModule, nullptr);
	}
```

以上便是描述一个管线中可编程部分的所有内容。在下一小节我们将关注Fixed-function阶段。

#### 9 固定功能 Fixed functions

一些老的图形API会为图形管线的大部分阶段提供默认的状态。但在Vulkan中，我们必须显式面对大部分管线状态，因为它将会被烘培成一个不可变的管线状态对象。在这一小节中，我们将完成配置这些Fixed-function operations的所有结构体。

#### 10 动态状态 Dynamic state

虽然大部分的管线状态都需要烘培到管线状态中，但还是**有一小些状态是可以在绘制时改变的，不需要重建管线**。举例来说，比如Viewport的尺寸、线宽、混合常量。如果我们想要使用动态状态并且保留这些属性，那么我们必须填充一个**VkPipelineDynamicStateCreateInfo**结构体。它会让这些值的配置被忽略，同时我们可以并且必须在绘制时明确这些数据。这会带来更加灵活的配置，对于视口或者裁剪状态来说非常常见。

```c++
std::vector<VkDynamicState> dynamicStates = {
    VK_DYNAMIC_STATE_VIEWPORT,
    VK_DYNAMIC_STATE_SCISSOR
};

VkPipelineDynamicStateCreateInfo dynamicState{};
dynamicState.sType = VK_STRUCTURE_TYPE_PIPELINE_DYNAMIC_STATE_CREATE_INFO;
dynamicState.dynamicStateCount = static_cast<uint32_t>(dynamicStates.size());
dynamicState.pDynamicStates = dynamicStates.data();
```

#### 11 顶点输入 Vertex input

**VkPipelineVertexInputStateCreateInfo**结构体描述了**将要被传输给顶点着色器的顶点数据的格式**。它大致以两种方式描述了这一点：
1. **绑定Bindings**：数据之间的间距，以及数据是每顶点的还是每实例的。
2. **属性描述Attribute descriptions**：传递给顶点着色器的属性的类型，从哪个绑定加载它们以及以什么偏移。

因为我们之前在顶点着色器中硬编码了顶点数据，因此目前我们先不加载任何顶点数据。

```c++
		//顶点输入
		VkPipelineVertexInputStateCreateInfo vertexInputInfo{};
		vertexInputInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_VERTEX_INPUT_STATE_CREATE_INFO;
		vertexInputInfo.vertexBindingDescriptionCount = 0;
		vertexInputInfo.pVertexBindingDescriptions = nullptr;
		vertexInputInfo.vertexAttributeDescriptionCount = 0;
		vertexInputInfo.pVertexAttributeDescriptions = nullptr;
```

#### 12 输入装配 Input assembly

**VkPipelineInputAssemblyStateCreateInfo**结构体描述了两个事情：**对于顶点数据什么样的几何形状会被绘制**，以及**是否启用Primitive Restart**。前者在**Topology**成员上确定，可以是以下这些值：
1. VK_PRIMITIVE_TOPOLOGY_POINT_LIST：每个顶点构成一个点。
2. VK_PRIMITIVE_TOPOLOGY_LINE_LIST：每两个顶点构成一条线，顶点不被重用。
3. VK_PRIMITIVE_TOPOLOGY_LINE_STRIP：每条线的终点会被作为下一条线的起点。
4. VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST：每三个顶点组成一个三角形，顶点不被重用。
5. VK_PRIMITIVE_TOPOLOGY_TRIANGLE_STRIP：每个三角形的后两个顶点会作为下一个三角形的前两个顶点。

通常来说顶点会从vertex buffer中按顺序索引加载，但是**通过element buffer我们可以自定义顶点索引**。它允许我们**通过重用顶点来优化性能**。如果将primitiveRestartEnable成员置为VK_TRUE，则可以通过使用特殊索引0xFFFF或0xFFFFFFFF来分解_STRIP拓扑模式中的直线和三角形。

在教程中，我们将图元装配模式设置为绘制三角形。

```c++
		//图元装配模式
		VkPipelineInputAssemblyStateCreateInfo inputAssembly{};
		inputAssembly.sType = VK_STRUCTURE_TYPE_PIPELINE_INPUT_ASSEMBLY_STATE_CREATE_INFO;
		inputAssembly.topology = VK_PRIMITIVE_TOPOLOGY_TRIANGLE_LIST;
		inputAssembly.primitiveRestartEnable = VK_FALSE;
```

#### 13 视口和裁剪 Viewports and scissors

**视口viewport描述了渲染输出的framebuffer的范围**。通常来说几乎都是从(0,0)到(width, height)，在该教程中也是如此。**注意swap chain和它其中的images的尺寸可能与window的长宽不同**。在后续，swap chain images将会被用作framebuffer，所以我们需要一直使用其相同的尺寸。

```c++
		//视口
		VkViewport viewport{};
		viewport.x = 0.0f;
		viewport.y = 0.0f;
		viewport.width = (float)swapChainExtent.width;
		viewport.height = (float)swapChainExtent.height;
		viewport.minDepth = 0.0f;
		viewport.maxDepth = 1.0f;
```

**视口Viewport决定从Image到Framebuffer的转换，而裁剪矩形Scissor rectangle决定了哪个范围的像素会被实际存储**。任何超出裁剪矩形的像素都会被光栅器丢弃。scissor的功能更像过滤的Mask，viewport的功能更像执行一个形变。如下图所示，可以看出两者区别。

![20231228212205](https://raw.githubusercontent.com/recaeee/PicGo/main/20231228212205.png)

所以如果我们想要绘制完整的framebuffer，我们需要定义一个完全覆盖它的scissor。

```c++
		//定义裁剪矩形
		VkRect2D scissor{};
		scissor.offset = { 0, 0 };
		scissor.extent = swapChainExtent;
```

Viewport和scissor rectangle可以被定义在管线的静态部分，也可以通过Command buffer被定义在**Dynamic state**集合中。我们让视口和裁剪状态动态化，因为它提供了更大的灵活性。这是很常见的，所有实现都可以处理这种动态状态，并且不会造成性能损失。由此，**在创建管线时只需要确定viewportState的数量，在绘制时再确定实际使用的viewport和scissor**。

```c++
		//在创建管线时只需要确定viewportState的数量，在绘制时再确定实际使用的viewport和scissor
		VkPipelineViewportStateCreateInfo viewportState{};
		viewportState.sType = VK_STRUCTURE_TYPE_PIPELINE_VIEWPORT_STATE_CREATE_INFO;
		viewportState.viewportCount = 1;
		viewportState.scissorCount = 1;
```

通过动态状态，我们甚至可以在一个Command buffer中指定不同的viewport或者scissor rectangle。如果不使用动态状态，我们需要在管线中使用**VkPipelineViewportStateCreateInfo**设置它们。这会让管线的视口和裁剪矩形不可变。对于这些值的改变都需要完全重建一个新的管线。

我们**可以在一些显卡上使用多个视口和裁剪矩形，因此传入给CreateInfo的Viewport和Scissor是以数组的形式**。使用多个需要启用GPU feature。

#### 14 光栅化器 Rasterizer

**光栅化器Rasterizer将顶点着色器输出的几何图形转变成片元传递给片元着色器**。它也会执行**深度检测**、**面剔除**和**裁剪检测**，同时它可以配置输出片元为填充模式还是线框模式。以上这些模式均由**VkPipelineRasterizationStateCreateInfo**配置。

```c++
		//配置光栅化器
		VkPipelineRasterizationStateCreateInfo rasterizer{};
		rasterizer.sType = VK_STRUCTURE_TYPE_PIPELINE_RASTERIZATION_STATE_CREATE_INFO;
		rasterizer.depthClampEnable = VK_FALSE; // 如果depthClampEnable为true，超出近、远平面的片元会被Clamp而不是丢弃它们。这在绘制深度图等一些特殊情况有用，使用它需要一个GPU feature。
		rasterizer.rasterizerDiscardEnable = VK_FALSE; // 该值为true时，会在光栅化阶段丢弃所有片元。
		rasterizer.polygonMode = VK_POLYGON_MODE_FILL; // 决定根据几何图形如何生成片元，可以是FILL、LINE、POINT，后两者需要GPU feature
		rasterizer.lineWidth = 1.0f; //超过1.0时，需要启用wideLines GPU feature
		rasterizer.cullMode = VK_CULL_MODE_BACK_BIT; //面剔除类型
		rasterizer.frontFace = VK_FRONT_FACE_CLOCKWISE; //确定正面方向
		rasterizer.depthBiasEnable = VK_FALSE; // 可以设置深度偏移
		rasterizer.depthBiasConstantFactor = 0.0f; // Optional
		rasterizer.depthBiasClamp = 0.0f; // Optional
		rasterizer.depthBiasSlopeFactor = 0.0f; // Optional
```

具体成员在代码中均有解释，我们需要在光栅化状态中配置**几何模式、线宽、面剔除模式、正面方向、深度偏移**等等。一些特性需要启用对应的GPU features。

#### 15 多重采样 Multisampling

**VkPipelineMultisampleStateCreateInfo**配置了多重采样，用来进行抗锯齿。它的工作原理是**混合光栅化到同一个像素的多个多边形的片元结果**。它通常发生在几何边缘，也是最容易出现锯齿的地方。因为它不会在仅有一个几何图形覆盖的区域多次计算片元着色，因此它的性能比单纯放大分辨率再降采样好很多。同样，启用它需要启用一个GPU feature。在后续章节中会详细讨论多重采样，目前我们先关闭它。

```c++
		//配置多重采样
		VkPipelineMultisampleStateCreateInfo multisampling{};
		multisampling.sType = VK_STRUCTURE_TYPE_PIPELINE_MULTISAMPLE_STATE_CREATE_INFO;
		multisampling.sampleShadingEnable = VK_FALSE;
		multisampling.rasterizationSamples = VK_SAMPLE_COUNT_1_BIT;
		multisampling.minSampleShading = 1.0f; // Optional
		multisampling.pSampleMask = nullptr; // Optional
		multisampling.alphaToCoverageEnable = VK_FALSE; // Optional
		multisampling.alphaToOneEnable = VK_FALSE; // Optional
```

#### 16 深度、模板检测 Depth and stencil testing

如果我们正在使用深度或模板缓冲，我们需要使用**VkPipelineDepthStencilStateCreateInfo**来配置深度、模板检测模式。我们目前用不到，所以会置一个nullptr，后续章节会重新讨论它。

#### 17 颜色混合 Color blending

在片元着色器输出一个颜色后，它需要**与已经在Framebuffer上的颜色做一个混合**。这个过程叫做**颜色混合Color Blending**。我们有两个方法操作它：

1. 混合新、旧两个值来产生一个最终颜色。
2. 使用位运算混合新、旧两个值。

我们需要两个结构体用来配置颜色混合。第一个是**VkPipelineColorBlendingAttachmentState用来配置每个attached framebuffer**，第二个是**VkPipelineColorBlendStateCreateInfo用来配置全局颜色混合设置**。在我们的项目中，我们只有一个framebuffer。

```c++
		//配置颜色混合
		VkPipelineColorBlendAttachmentState colorBlendAttachment{};
		colorBlendAttachment.colorWriteMask = VK_COLOR_COMPONENT_R_BIT | VK_COLOR_COMPONENT_G_BIT | VK_COLOR_COMPONENT_B_BIT | VK_COLOR_COMPONENT_A_BIT;
		colorBlendAttachment.blendEnable = VK_FALSE;
		colorBlendAttachment.srcColorBlendFactor = VK_BLEND_FACTOR_ONE; // Optional
		colorBlendAttachment.dstColorBlendFactor = VK_BLEND_FACTOR_ZERO; // Optional
		colorBlendAttachment.colorBlendOp = VK_BLEND_OP_ADD; // Optional
		colorBlendAttachment.srcAlphaBlendFactor = VK_BLEND_FACTOR_ONE; // Optional
		colorBlendAttachment.dstAlphaBlendFactor = VK_BLEND_FACTOR_ZERO; // Optional
		colorBlendAttachment.alphaBlendOp = VK_BLEND_OP_ADD; // Optional
```

实际的颜色混合过程可以参考如下伪代码。

```c++
if (blendEnable) {
    finalColor.rgb = (srcColorBlendFactor * newColor.rgb) <colorBlendOp> (dstColorBlendFactor * oldColor.rgb);
    finalColor.a = (srcAlphaBlendFactor * newColor.a) <alphaBlendOp> (dstAlphaBlendFactor * oldColor.a);
} else {
    finalColor = newColor;
}

finalColor = finalColor & colorWriteMask;
```

如果blendEnable为VK_FALSE，则片元着色器输出的颜色会直接覆盖原Framebuffer上的颜色。否则，这两个混合操作会执行计算并得到一个新颜色。颜色计算结果回合colorWriteMask进行与操作来决定哪些通道会被实际传递。

第二个结构体会引用所有的framebuffer，并且我们可以在其中设置混合常量。

```c++
VkPipelineColorBlendStateCreateInfo colorBlending{};
colorBlending.sType = VK_STRUCTURE_TYPE_PIPELINE_COLOR_BLEND_STATE_CREATE_INFO;
colorBlending.logicOpEnable = VK_FALSE;
colorBlending.logicOp = VK_LOGIC_OP_COPY; // Optional
colorBlending.attachmentCount = 1;
colorBlending.pAttachments = &colorBlendAttachment;
colorBlending.blendConstants[0] = 0.0f; // Optional
colorBlending.blendConstants[1] = 0.0f; // Optional
colorBlending.blendConstants[2] = 0.0f; // Optional
colorBlending.blendConstants[3] = 0.0f; // Optional
```

如果我们想要使用位运算混合操作，我们需要将**logicOpEnable**设置为VK_TRUE。

在我们的教程中，我们会disable这两个模式，意味着**输出的片元颜色会不经任何操作直接写入framebuffer**。

#### 18 管线布置 Pipeline layout

我们可以在shader中使用**uniform变量**，其原理和dynamic state很类似，可以在绘制时修改，以此来改变shader的行为，而不需要重建它们。它们通常用来传递变换矩阵给顶点着色器，或者在片元着色器中创建texture samplers。

这些uniform值需要在创建管线时通过创建**VkPipelineLayout**来指定。虽然后续章节暂时用不到它，但我们仍然需要创建一个空的pipeline layout。创建一个类成员来持有它，后续章节会引用到它。

```c++
	//存储pipeline layout，用来指定创建管线时的uniform值
	VkPipelineLayout pipelineLayout;

	//一旦window被关闭，我们将在**cleanup**函数中确保释放我们用到的所有资源
	void cleanup()
	{
		//销毁pipeline layout
		vkDestroyPipelineLayout(device, pipelineLayout, nullptr);
		...
	}

	//创建pipeline
	void createGraphicsPipeline()
	{
		...

		//创建pipeline layout，指定shader uniform值
		VkPipelineLayoutCreateInfo pipelineLayoutInfo{};
		pipelineLayoutInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO;
		pipelineLayoutInfo.setLayoutCount = 0; // Optional
		pipelineLayoutInfo.pSetLayouts = nullptr; // Optional
		pipelineLayoutInfo.pushConstantRangeCount = 0; // Optional
		pipelineLayoutInfo.pPushConstantRanges = nullptr; // Optinal

		if (vkCreatePipelineLayout(device, &pipelineLayoutInfo, nullptr, &pipelineLayout) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to create pipeline layout!");
		}
	}
```

这个结构体也指定了**push constants**，这是将动态值传递给着色器的另一种方式，在后续章节会介绍到。Pipeline layout会在程序的生命周期内持续引用，因此需要在cleanup中销毁它。

#### 19 小结 Conclusion

以上便是所有的fixed-function状态。从头设置它们需要许多工作，但因此我们也几乎知道了graphic pipeline中发生的所有事情。在最后创建Graphic pipeline之前，我们还需要创建一个对象，它就是**Render Pass**。呼，创建个管线好累！

#### 20 设置渲染通道 Setup Render passes

在我们完成pipeline创建之前，我们需要告诉Vulkan我们将要用于渲染的framebuffer attachments。我们需要明确会**有多少color和depth buffers**，**每个buffer使用多少个samples**，**以及在整个渲染操作中如何处理它们的内容**。以上所有信息都被封装到一个**render pass**对象中，我们会创建一个**createRenderPass**函数，在initVulkan函数中createGraphicsPipeline之前调用它。

#### 21 Attachment描述符 Attachment description

在我们的项目中，我们只有一个color buffer attachment，由swap chain中的一个image表示。并且我们目前不会使用多重采样，因此使用1 sample。

```c++
	//创建Render pass
	void createRenderPass()
	{
		VkAttachmentDescription colorAttachment{};
		colorAttachment.format = swapChainImageFormat; //color attachment的format需要和swap chain images的format匹配
		colorAttachment.samples = VK_SAMPLE_COUNT_1_BIT;
		//loadOp和storeOp应用于color和depth
		colorAttachment.loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR;
		colorAttachment.storeOp = VK_ATTACHMENT_STORE_OP_STORE;
		//stencilLoadOp和stencilStoreOp应用于stencil
		colorAttachment.stencilLoadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
		colorAttachment.stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
		//render pass前image期望的格式，结束后image将要转变的格式
		colorAttachment.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
		colorAttachment.finalLayout = VK_IMAGE_LAYOUT_PRESENT_SRC_KHR;
	}
```

**loadOp**和**storeOp**相信经常使用URP的小伙伴应该都很熟悉了，我们在SetRenderTarget的时候会经常对这些值做文章。它们**决定了在渲染前和渲染后我们会对attachment的数据做什么**，我们具有以下选项给loadOp：
1. **VK_ATTACHMENT_LOAD_OP_LOAD**：保留attachment上的当前内容。
2. **VK_ATTACHMENT_LOAD_OP_CLEAR**：在开始时使用一个常值清理attachment的内容。
3. **VK_ATTACHMENT_LOAD_OP_DONT_CARE**：当前attachment上的内容被认为是未定义的，我们不关心它。

在我们的项目中，我们会使用Clear操作，在绘制新一帧前用黑色来清理framebuffer。

**storeOp**只有2个可选项：
1. **VK_ATTACHMENT_STORE_OP_STORE**：渲染完的内容将被存储到内存中，之后可以读取。
2. **VK_ATTACHMENT_STORE_OP_DONT_CARE**：在渲染完后，framebuffer的内容被视为未定义的。

我们想在渲染完三角形后看到它显示在屏幕上，因此我们使用store操作。

loadOp和storeOp应用于color和depth。stencilLoadOp和stencilStoreOp应用于stencil。

接下来看**initialLayout**和**finalLayout**。**Vulkan中的textures和framebuffers都是用VKImages对象来表示的**，而VKImage会包含一个特定的pixel format，**我们可以根据我们要对Image做的操作改变内存中的pixel layout，以获取更好的性能**。

以下为几个常见的Layouts：

1. **VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL**：Images被用作color attachment。
2. **VK_IMAGE_LAYOUT_PRESENT_SRC_KHR**：Images在swap chain中被用作present。
3. **VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL**：Images被用作memory copy的destination。

我们将在texturing章节对该话题展开，但目前更重要的是，我们需要知道**Images需要转换为适合接下来要涉及的操作的特定Layouts**。

**initialLayout指定了在render pass开始前image将使用的layout**。**finalLayout指定了当render pass结束时image会自动切换成的layout**。对initialLayout使用VK_IMAGE_LAYOUT_UNDEFINED意味着我们并不关心image之前的layout是什么样的。需要注意的是，这个**UNDEFINED意味着image的内容不能保证被preserved**，但这并不重要，因为我们会清理它。我们希望这个image在渲染完后被用于在swap chain中呈现，因此我们使用VK_IMAGE_LAYOUT_PRESENT_SRC_KHR作为finalLayout。

#### 22 子通道和attachment引用 Subpasses and attachment references

**一个单独的渲染通道render pass可以由多个子通道subpasses组成**。**Subpasses是紧随其后的渲染操作，其关联于先前passes中framebuffer的内容**，例如依次应用的一系列后处理操作。如果我们将这些渲染操作group到一个render pass中，**Vulkan就可以对这些操作重新排序并节省内存带宽**，从而可能获得更好的性能。在我们绘制一个三角形的项目中，我们只会使用一个subpass。

```c++
		//subpass引用的attachment
		VkAttachmentReference colorAttachmentRef{};
		colorAttachmentRef.attachment = 0;
		colorAttachmentRef.layout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;
		//subpass
		VkSubpassDescription subpass{};
		subpass.pipelineBindPoint = VK_PIPELINE_BIND_POINT_GRAPHICS; // Vulkan未来会支持compute subpasses，所以我们需要显式告诉它是一个graphics subpass。
		//指定subpass的attachment引用
		subpass.colorAttachmentCount = 1;
		subpass.pColorAttachments = &colorAttachmentRef;
```

每个subpass都引用我们之前描述过的一个或多个attachments。这些引用本身就是**VkAttachmentReference**结构。

**attachment**参数通过attachment在descriptions array中的索引指定了要引用的attachment。我们的array由一个单独的VkAttachmentDescription组成，所以索引为0。**layout**制定了在subpass中attachment被期望的layout。**Vulkan会在subpass开始前自动将attachment转换成这个layout**。我们期望attachment被用作color buffer，而VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL会给我们更好的性能，就如它的名字。

Vulkan未来会支持compute subpasses，所以我们需要显**式告诉它是一个graphics subpass**。

**attachment array中的索引是通过layout(location = 0) out vec4 outColor指令直接从片元着色器中引用的**。attachment array也可以理解成subpass中的shader需要in和out使用到的attachment吧。

以下是**一个subpass可以引用的其他attachment类型**：
1. **pInputAttachments**：从shader中读取的attachments。
2. **pResolveAttachments**：用于多重采样color attachments的attachments
3. **pDepthStencilAttachment**：depth和stencil数据的attachment。
4. **pPreserveAttachment**：attachment不会被这个subpass引用，但是其数据需要保留。

#### 23 渲染通道 Render pass

现在我们已经描述了attachment和引用它的subpass，我们可以创建**render pass**了！我们创建一个类成员来持有**VkRenderPass**对象。

```c++
		//创建render pass
		VkRenderPassCreateInfo renderPassInfo{};
		renderPassInfo.sType = VK_STRUCTURE_TYPE_RENDER_PASS_CREATE_INFO;
		renderPassInfo.attachmentCount = 1;
		renderPassInfo.pAttachments = &colorAttachment;
		renderPassInfo.subpassCount = 1;
		renderPassInfo.pSubpasses = &subpass;

		if (vkCreateRenderPass(device, &renderPassInfo, nullptr, &renderPass) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to create render pass!");
		}
```

就像pipeline layout，render pass也会在程序运行期间被引用，因此需要在cleanup中销毁。

好了，下一节我们就会创建graphics pipeline了！

#### 24 总结 Conclusion

现在，我们可以根据我们之前创建的那么多结构体和对象来创建graphics pipeline了。我们先快速回顾下我们拥有的对象：
1. **Shader stages**：定义了管线中可编程部分功能的shader modules。
2. **Fixed-function state**：所有管线的固定功能状态结构体，比如输入装配、光栅化器、视口和颜色混合。
3. **Pipeline layout**：绘制时可更改的shader的uniform和push值。
4. **Render pass**：管线阶段引用的attachments和它们的用途。

接下来在createGrahpicsPipeline函数中创建**VkGraphicsPipelineCreateInfo**结构体，记得需要在销毁shader modules之前完成创建，因为在创建过程中还需要用到它们。

```c++
		//创建pipeline
		VkGraphicsPipelineCreateInfo pipelineInfo{};
		pipelineInfo.sType = VK_STRUCTURE_TYPE_GRAPHICS_PIPELINE_CREATE_INFO;
		//shader stages
		pipelineInfo.stageCount = 2;
		pipelineInfo.pStages = shaderStages;
		//fixed-function
		pipelineInfo.pVertexInputState = &vertexInputInfo;
		pipelineInfo.pInputAssemblyState = &inputAssembly;
		pipelineInfo.pViewportState = &viewportState;
		pipelineInfo.pRasterizationState = &rasterizer;
		pipelineInfo.pMultisampleState = &multisampling;
		pipelineInfo.pDepthStencilState = nullptr; // Optional
		pipelineInfo.pColorBlendState = &colorBlending;
		pipelineInfo.pDynamicState = &dynamicState;
		//pipeline layout
		pipelineInfo.layout = pipelineLayout;
		//render pass
		pipelineInfo.renderPass = renderPass;
		pipelineInfo.subpass = 0;//subpass index
		pipelineInfo.basePipelineHandle = VK_NULL_HANDLE; // Optional
		pipelineInfo.basePipelineIndex = -1; // Optional

		if (vkCreateGraphicsPipelines(device, VK_NULL_HANDLE, 1, &pipelineInfo, nullptr, &graphicsPipeline) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to create graphics pipeline!");
		}

		//创建完pipeline后立刻销毁shader modules
		vkDestroyShaderModule(device, fragShaderModule, nullptr);
		vkDestroyShaderModule(device, vertShaderModule, nullptr)
```

我们可以在这个管线中使用其他render passes，但是它们必须和该renderPass兼容。最后还有两个额外参数**basePipelineHandle**和**basePipelineIndex**。Vulkan允许我们从现有管线派生来创建新的管线。**管线派生**的思想是，**如果要创建的管线和一个现有的管线有许多相同的功能，比起从零重新设置管线，管线派生更加低成本**，并且**在相同parent的管线之间做切换会更加快**。我们可以使用basePipelineHandle指定现有管线的handle，也可以使用basePipelineIndex引用即将通过索引创建的另一个管线。显然我们现在只有一个管线，因此置为nullpHandle和-1。**这两个值只有在VkGraphicsPipelineCreateInfo的flags中明确VK_PIPELINE_CREATE_DERIVATIVE_BIT才会生效**。

vkCreateGraphicsPipelines函数比起一般的VulkanObject创建函数有更多参数，因为它被设计成**可以使用多个VkGraphicsPipelineCreateInfo来在一次调用中创建多个VkPipeline**。

第二个VK_NULL_HANDLE参数引用了一个可选的**VkPipelineCache**对象。**pipeline cache用来存储和复用在多次管线创建调用之间相关的数据**。如果cache存储到文件中，甚至可以跨程序执行存储和重用与管线创建相关的数据。这个可以让管线创建过程液氮加速，我们在之后的pipeline cahce章节详细展开。

graphics pipeline在所有常用的绘制操作都会被用到，因此它应该在程序结束时才被销毁，我们在cleanup中销毁它。

```c++
	//一旦window被关闭，我们将在**cleanup**函数中确保释放我们用到的所有资源
	void cleanup()
	{
		//销毁pipeline
		vkDestroyPipeline(device, graphicsPipeline, nullptr);
		//销毁pipeline layout
		vkDestroyPipelineLayout(device, pipelineLayout, nullptr);
		...
	}
```

呼！这一章终于结束了，创建Graphics pipeline的过程居然是如此的复杂。

最后，祝小伙伴们2024年新年快乐！

#### 参考
1. 题图来自画师wlop
