---
layout:     post
title:      "【VulkanTutorial学习笔记——11】Uniform buffers"
subtitle:   ""
date:       2024-02-09 12:00:00
author:     "recaeee"
header-img: "img/in-post/default-bg.jpg"
tags:
    - Vulkan
---
# 【VulkanTutorial学习笔记——11】Uniform buffers

![20240209124607](https://raw.githubusercontent.com/recaeee/PicGo/main/20240209124607.png)

#### 0 写在前面
本系列为个人在跟随Vulkan Tutorial学习过程中记录的一些笔记，一是防止自己看着看着弃坑，二是希望能在看过之后多少留些印象，三是想在啥时候忘记了能快速地回忆一些Vulkan知识点。

出于以上这些目的，该系列的学习笔记可能会写得较随意一些，并且对于主观判断不重要的东西可能会略过，因此也不保证该系列笔记的学习质量，可能有些东西不经过脑子就会写出来。

我是一个Vulkan的小白，对于Vulkan尚不熟悉，笔记中可能存在错误，或者概念不明确的地方，也希望看的人可以批评斧正~

今天是除夕了，提前祝大家新年快乐~（感觉每年的1月过得都好快）VulkanTutorial的学习进度也表现的不错，虽然一开始只是怕自己学完之后全忘了，才写的这个笔记，但每一章也几乎都是90%以上的翻译，导致了龟速般的学习速度，但也算是锻炼自己翻译英语的能力吧，何况Vulkan如此之复杂，值得这么久的学习来稳固基础。（另外，实践证明学习笔记确实能防止自己很早弃坑）

在这一章，我们会处理Uniform buffer，虽然uniform buffer可能是我们的老朋友，但我们也会遇到Vulkan中的新朋友descriptor、descriptor layout、descriptor set和descriptor pool。其实，在学习过程中可以发现，我们基本只需要关注descriptor layout、descriptor set和descriptor pool这三者就行，基本我们的代码接触不到descriptor，这样对我们理解起来可能更容易。

从我个人的经验来说，这一章节的知识还是比较重要的，在实际的项目实践中，设备会存在descriptor set的最大数量限制，而在Unity中，通常这种descriptor set数量溢出错误可以触发一些兼容性问题，需要我们去减少创建的descriptor sets来保证兼容。

原教程链接：
https://vulkan-tutorial.com/Introduction
本章节链接：
https://vulkan-tutorial.com/Uniform_buffers/Descriptor_layout_and_buffer
https://vulkan-tutorial.com/Uniform_buffers/Descriptor_pool_and_sets

#### 1 描述符布局和缓冲区 Descriptor layout and buffer

现在我们能为每个顶点传递任意属性给顶点着色器，但是对于**全局的值**呢？我们将要在这一章涉足3D图形，其需要model-view-projection矩阵（即MVP矩阵）。我们的确可以将其包含在顶点数内，但是它会造成内存浪费，并且一旦变换矩阵改变了，需要我们更新vertex buffer。变换矩阵理论上应该在每一帧之间能够轻松地改变。

在Vulkan中解决这个问题的正确方法是使用**资源描述符resource descriptors**。**一个descriptor是shader去自由访问比如buffers和images等资源的一个方式**。我们将会建立一个buffer来包含变换矩阵，并且让顶点着色器通过一个descriptor来访问它们。descriptors的使用包含以下三部分：

1. 在pipeline创建期间指定一个**descriptor layout**。
2. 从一个**descriptor pool**中分配一个**descriptor set**。
3. 在渲染期间绑定**descriptor**。

**描述符布局descriptor layout指定了pipeline将访问的资源类型**，就像render pass指定将要访问的attachments类型一样。**描述符集descriptor set指定了实际将要被绑定到descriptors的真实buffer或者image资源**，就像framebuffer指定了要绑定到render pass attachments上的真实的image views。然后**将descriptor set绑定到绘制指令**，就像vertex buffers和framebuffer一样。

有许多类型的descriptors，但是在这一章，我们只会用到**uniform buffer objects（UBO）**，也是老朋友了噢。我们将会在未来章节关注其他类型的descriptors，但是其基本过程是相同的。

假设我们希望顶点着色器在C结构体中拥有这样的数据：

```c++
struct UniformBufferObject {
    glm::mat4 model;
    glm::mat4 view;
    glm::mat4 proj;
};
```

然后我们可以将数据拷贝到一个VkBuffer中，然后使用一个UBO descriptor来在顶点着色器中访问它，如下：

```c++
layout(binding = 0) uniform UniformBufferObject {
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;

void main() {
    gl_Position = ubo.proj * ubo.view * ubo.model * vec4(inPosition, 0.0, 1.0);
    fragColor = inColor;
}
```

我们将会每一帧更新MVP矩阵，来让矩形在3D空间中旋转。

#### 2 顶点着色器 Vertex shader

修改顶点着色器来包含上文中提到的uniform buffer object。

```c++
#version 450

layout(binding = 0) uniform UniformBufferObject {
	mat4 model;
	mat4 view;
	mat4 proj;
} ubo;

layout(location = 0) in vec2 inPosition;
layout(location = 1) in vec3 inColor;

layout(location = 0) out vec3 fragColor;

void main()
{
	gl_Position = ubo.proj * ubo.view * ubo.model * vec4(inPosition, 0.0, 1.0);
	fragColor = inColor;
}
```

**uniform、in和out声明的顺序是没要求的**。binding指令就和属性的location指令类似。我们将会在descriptor layout中引用该binding。gl_Position的代码将会变成计算裁剪空间下的最终坐标。不像2D三角形，裁剪空间下的w分量也许不是1，在屏幕上转换成NDC坐标时会应用透视除法。这在透视投影中称为**透视除法perspective division**，其对于近大远小的关系非常重要。

#### 3 描述符集布局 Descriptor set layout

下一步是在C++端定义UBO，并且在顶点着色器中告诉Vulkan这个descriptor。

```c++
	struct UniformBufferObject {
		glm::mat4 model;
		glm::mat4 view;
		glm::mat4 proj;
	};
```

我们可以使用GLM中的数据类型来完全匹配shader中的定义。矩阵中的数据与着色器期望的方式二进制兼容，因此我们稍后可以**将UniformBufferOjbect使用memcopy拷贝到一个VkBuffer中**。

我们需要**提供有关着色器中用于创建pipeline的每个descriptor binding的详细信息**，就像我们需要为每个顶点属性和其location索引做的。我们将会创建一个新的函数**createDescriptorSetLayout**来定义所有的信息。它应该**在pipeline创建前被调用**，因为在pipeline创建时我们需要用到它。

```c++
	//initVulkan函数来用于实例化Vulkan objects私有成员
	void initVulkan()
	{
		...
		createDescriptorSetLayout();
		createGraphicsPipeline();
		...
	}

	void createDescriptorSetLayout()
	{

	}
```

任何binding都需要通过一个**VkDescriptorSetLayoutBinding**结构体来描述。

```c++
	void createDescriptorSetLayout()
	{
		VkDescriptorSetLayoutBinding uboLayoutBinding{};
		uboLayoutBinding.binding = 0;
		uboLayoutBinding.descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
		uboLayoutBinding.descriptorCount = 1;
	}
```

前两个字段指定了**shader中用到的binding**，和**descriptor的类型**，在这里是uniform buffer object。**着色器中的变量可以表示uniform buffer objects的数组，descriptorCount可以指定数组中值的数量**。例如，这可以用于指定骨骼动画中每一个骨骼的变换。我们的MVP变换是单个uniform buffer object，所以我们使用1作为descriptorCount的值。

```c++
		uboLayoutBinding.stageFlags = VK_SHADER_STAGE_VERTEX_BIT;
```

我们也需要**指定在哪个shader阶段该discriptor会被引用**。**stageFlags**字段可以是一个VkShaderStageFlagBits值的组合，或者值VK_SHADER_STAGE_ALL_GRAPHICS。在这里，我们只在顶点着色器中引用该descriptor。

```c++
		uboLayoutBinding.pImmutableSamplers = nullptr;// Optional
```

**pImmutableSamplers**字段只和image sampling相关的descriptors有关，后面我们会看到。在这里，将其置为默认值。

**所有的descriptor bindings会被组合到一个单独的VkDscriptorSetLayout对象中**。定义一个新的类成员在pipelineLayout之上。

```c++
	//存储descriptor set layout
	VkDescriptorSetLayout descriptorSetLayout;
	//存储pipeline layout，用来指定创建管线时的uniform值
	VkPipelineLayout pipelineLayout;
```

接下来我们使用**vkCreateDescriptorSetLayout**来创建它。该函数接受一个简单的VkDescriptorSetLayoutCreateInfo和bindings数组。

```c++
		//创建descriptor set layout
		VkDescriptorSetLayoutCreateInfo layoutInfo{};
		layoutInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO;
		layoutInfo.bindingCount = 1;
		layoutInfo.pBindings = &uboLayoutBinding;

		if (vkCreateDescriptorSetLayout(device, &layoutInfo, nullptr, &descriptorSetLayout) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to create descriptor set layout!");
		}
```

我们需要**在pipeline创建时指定descriptor set layout来告诉Vulkan着色器将会用到哪些descriptors**。**Descriptor set layouts在pipeline layout对象中被指定**。修改VkPipelineLayoutCreateInfo来引用该layout对象。

```c++
		//创建pipeline layout，指定shader uniform值
		VkPipelineLayoutCreateInfo pipelineLayoutInfo{};
		pipelineLayoutInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO;
		pipelineLayoutInfo.setLayoutCount = 1; // Optional
		pipelineLayoutInfo.pSetLayouts = &descriptorSetLayout; // Optional
```

我们也许想知道为什么可以在这里**指定多个descriptor set layouts**，因为一个单独的layout已经可以包含所有bindings了。我们会在下一节眼界descriptor pools和descriptor sets时回到这。

当我们创建新的graphics pipeline时，descriptor set layout应该保留下来，直到程序结束。

```c++
	//一旦window被关闭，我们将在**cleanup**函数中确保释放我们用到的所有资源
	void cleanup()
	{
		cleanupSwapChain();

		//销毁descriptor set layout
		vkDestroyDescriptorSetLayout(device, descriptorSetLayout, nullptr);
		...
	}
```

#### 4 全局缓冲区 Uniform buffer

在下一节，我们会指定包含给Shader的UBO数据的buffer，但是我们需要先创建该buffer。我们将会**在每一帧拷贝新数据到该uniform buffer中**，所以使用staging buffer实际上没有意义。在这种情况下，它只会增加额外的开销，并且可能会降低性能，而不是提高性能。

我们应该有许多buffers，因为同一时间会有多帧同时运行，并且我们不希望在前一帧仍在读取时更新buffer以准备下一帧。因此，我们**需要和同时运行的frame数量一致的uniform buffers**，并且写入到GPU不正在读取的uniform buffer。

因此，增加新的类成员**uniformBuffers**和**uniformBuffersMemory**。

```c++
	//Index buffer
	VkBuffer indexBuffer;
	//实际分配的index buffer的内存
	VkDeviceMemory indexBufferMemory;
	//uniform buffers，数量和同时运行的frame数量一致
	std::vector<VkBuffer> uniformBuffers;
	std::vector<VkDeviceMemory> uniformBuffersMemory;
	std::vector<void*> uniformBuffersMapped;
```

类似的，创建一个新函数**createUniformBuffers**，在createIndexBuffer之后调用，并且在其中为buffer分配内存。

```c++
	//initVulkan函数来用于实例化Vulkan objects私有成员
	void initVulkan()
	{
		...
		createVertexBuffer();
		createIndexBuffer();
		createUniformBuffers();
		...
	}

	void createUniformBuffers()
	{
		VkDeviceSize bufferSize = sizeof(UniformBufferObject);

		uniformBuffers.resize(MAX_FRAMES_IN_FLIGHTS);
		uniformBuffersMemory.resize(MAX_FRAMES_IN_FLIGHTS);
		uniformBuffersMapped.resize(MAX_FRAMES_IN_FLIGHTS);

		for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHTS; i++)
		{
			createBuffer(bufferSize, VK_BUFFER_USAGE_UNIFORM_BUFFER_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, uniformBuffers[i], uniformBuffersMemory[i]);

			vkMapMemory(device, uniformBuffersMemory[i], 0, bufferSize, 0, &uniformBuffersMapped[i]);
		}
	}
```

我们在创建完之后**使用vkMapMemory来映射buffer来获取一个指针然我们可以之后写入数据**。**在整个程序的生命周期内，buffer会持续映射到该指针**。该技巧称为"**持久映射persistent mapping**"，适用于所有Vulkan实现。不需要每次在我们更新它时才映射该buffer，这提高了性能，因为**mapping是有消耗的**。

uniform数据将会被用于所有的draw call，所以在我们**停止渲染时才销毁包含该unform数据的buffer**。

```c++
	//一旦window被关闭，我们将在**cleanup**函数中确保释放我们用到的所有资源
	void cleanup()
	{
		cleanupSwapChain();
		//销毁Uniform buffer和对应Memory
		for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHTS; i++)
		{
			vkDestroyBuffer(device, uniformBuffers[i], nullptr);
			vkFreeMemory(device, uniformBuffersMemory[i], nullptr);
		}

		//销毁descriptor set layout
		vkDestroyDescriptorSetLayout(device, descriptorSetLayout, nullptr);
		...
	}
```

#### 5 上传全局数据 Updating uniform data

创建一个新函数**updateUniformBuffer**，在drawFrame函数中的Submit之前调用它。

```c++
	void drawFrame()
	{
		...

		//更新uniform数据
		updateUniformBuffer(currentFrame);
		
		...

		//提交command buffer到queue中，并且配置同步
		VkSubmitInfo submitInfo{};
		submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;
		...
	}

	void updateUniformBuffer(uint32_t currentImage)
	{

	}
```

该函数会在每一帧生成一个新的变换，来让几何物体旋转。我们需要include两个新的头文件来实现该功能。

```c++
//数学库，包含vector和matrix等，用于声明顶点数据、3D MVP变换等
#define GLM_FORCE_RADIANS
#include <glm/glm.hpp>
#include <glm/gtc/matrix_transform.hpp>

#include <chrono>
```

**glm/gtc/matrix_transform.hpp**头文件暴露了可用于生成Model变换的函数（如glm::rotate）、View变换（如glm::lookAt）和Projection变换（如glm::perspective）。**GLM_FORCE_RADIANS**定义被用于保证如glm::rotate这些函数使用radians单位作为参数，避免可能的困惑。

**chrono**标准库头文件暴露了进行精确计时的函数。我们使用它来保证每秒渲染90°，使其与帧率无关。

```c++
	void updateUniformBuffer(uint32_t currentImage)
	{
		static auto startTime = std::chrono::high_resolution_clock::now();

		auto currentTime = std::chrono::high_resolution_clock::now();
		float time = std::chrono::duration<float, std::chrono::seconds::period>(currentTime - startTime).count();
	}
```

**updateUniformBuffer**函数将会从一些逻辑开始，使用浮点数精度计算从渲染开始后的时间，（以秒为单位）。

现在我们将**在uniform buffer object中定义MVP变换**。model矩阵将会是一个使用到time变量的绕着Z轴的简单旋转。

```c++
		//在uniform buffer object中定义MVP变换
		UniformBufferObject ubo{};
		ubo.model = glm::rotate(glm::mat4(1.0f), time * glm::radians(90.0f), glm::vec3(0.0f, 0.0f, 1.0f));
```

glm::rotate函数使用一个现有的变换矩阵、旋转角度、旋转轴作为参数。glm::mat4(1.0f)构造函数返回单位矩阵。使用time * glm::radians(90.0f)作为旋转角度实现了每秒旋转90°。

```c++
		ubo.view = glm::lookAt(glm::vec3(2.0f, 2.0f, 2.0f), glm::vec3(0.0f, 0.0f, 0.0f), glm::vec3(0.0f, 0.0f, 1.0f));
```

对于view变换，教程中决定从上45°俯视该几何图形。glm::lookAt函数使用摄像机位置、中心位置、上方向作为参数。

```c++
		ubo.proj = glm::perspective(glm::radians(45.0f), swapChainExtent.width / (float)swapChainExtent.height, 0.1f, 10.0f);
```

在教程中，选择使用一个45°fov的透视投影，其他的参数为aspect、近平面、远平面。**使用当前的swap chain extent来计算aspect**是非常重要的，因为window的width和height会resize。

```c++
		ubo.proj[1][1] *= -1;
```

**GLM库起初是为OpenGL设计的，其裁剪空间下的Y轴坐标与Vulkan下是相反的**。最简单的补偿方式是翻转project矩阵下Y轴上的scaling要素。如果不这么做，渲染出来的image会上下颠倒。

所有的变换现在都定义好了，我们可以将uniform buffer object对象中的数据拷贝到当前的uniform buffer中。其实现和我们为vertex buffer做的几乎一样，除了不用staging buffer。正如之前提到的，**我们只映射一次uniform buffer，所以我们可以直接写入它而不需要再次映射**。

```c++
		//将数据拷贝到uniform buffer中
		memcpy(uniformBuffersMapped[currentImage], &ubo, sizeof(ubo));
```

以这种方式使用UBO并不是将频繁更改的值传递给着色器的最有效方法，**一个更有效的将小buffer的数据传递给shader的方法是push constants**。我们会在未来章节看到它。

在下一章，我们会关注**descriptor sets**，其实际将VkBuffer绑定到uniform buffer descriptors，由此shader可以访问变换矩阵数据。


##### 6 描述符池 Descriptor popl

在之前章节的decriptor set layout描述了可以绑定的descriptor类型。在这一章节，我们将会**为每个VkBuffer资源创建一个descriptor set，来将VkBuffer绑定到uniform buffer descriptor**。

**Descriptor sets不能被直接创建，它们必须从池中创建**，就像command buffers。毫不奇怪，descriptor sets的池称为**描述符池descriptor pool**。我们将写一个新函数**createDescriptorPool**来建立它。

```c++
	//initVulkan函数来用于实例化Vulkan objects私有成员
	void initVulkan()
	{
		...
		createUniformBuffers();
		createDescriptorPool();
		...
	}

		void createDescriptorPool()
	{

	}
```

我们首先需要描述我们的descriptor sets需要包含哪些descriptor类型，以及数量，使用**VkDescriptorPoolSize**结构体。

```c++
		VkDescriptorPoolSize poolSize{};
		poolSize.type = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;//需要包含哪些descriptor类型
		poolSize.descriptorCount = static_cast<uint32_t>(MAX_FRAMES_IN_FLIGHTS);//数量
```

我们将会为每一帧分配一个descriptor。poolSize结构体被**VkDescriptorPoolCreateInfo**引用。

```c++
		VkDescriptorPoolCreateInfo poolInfo{};
		poolInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_POOL_CREATE_INFO;
		poolInfo.poolSizeCount = 1;
		poolInfo.pPoolSizes = &poolSize;
```

除了**可用的单个描述符的最大数量**之外，我们也需要指定**可能被分配的descriptor sets的最大数量**。

```c++
		poolInfo.maxSets = static_cast<uint32_t>(MAX_FRAMES_IN_FLIGHTS);
```

该结构体有一个可选的flag，就和command pools一样，决定了是否单个descriptor sets可以被释放：**VK_DESCRIPTOR_POOL_CREATE_FREE_DESCRIPTOR_SET_BIT**。我们在创建完之后不会接触到descriptor set，所以我们不需要该flag。我们可以将flags置为默认值0。

```c++
	//存储descriptor pool，用于分配descriptor sets
	VkDescriptorPool descriptorPool;

	if (vkCreateDescriptorPool(device, &poolInfo, nullptr, descriptorPool) != VK_SUCCESS)
	{
		throw std::runtime_error("failed to create descriptor pool!");
	}
```

增加一个新的类成员来存储descriptor pool的句柄，并且调用**vkCreateDescriptorPool**来创建它。

#### 7 描述符集 Descriptor set

现在我们可以分配**描述符集descriptor sets**自身了。增加一个**createDescriptorSets**函数。

```c++
	//initVulkan函数来用于实例化Vulkan objects私有成员
	void initVulkan()
	{
		...
		createDescriptorPool();
		createDescriptorSets();
		...
	}

		void createDescriptorSets()
	{

	}
```

descriptor set的分配通过**VkDescriptorSetAllocateInfo**结构体描述。我们需要指定要从哪个descriptor pool中分配，要分配的descriptor sets的数量和它们基于的descriptor set layout。

```c++
	void createDescriptorSets()
	{
		std::vector<VkDescriptorSetLayout> layouts(MAX_FRAMES_IN_FLIGHTS, descriptorSetLayout);
		VkDescriptorSetAllocateInfo allocInfo{};
		allocInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_ALLOCATE_INFO;
		allocInfo.descriptorPool = descriptorPool;
		allocInfo.descriptorSetCount = static_cast<uint32_t>(MAX_FRAMES_IN_FLIGHTS);
		allocInfo.pSetLayouts = layouts.data();
	}
```

在我们的案例中，我们将会**为每一个运行中的帧创建一个descriptor set**，使用同一个layout。不幸的是，我们确实需要拷贝所有的layout，因为下一个函数期望与sets数量相匹配的layouts数组。

增加一个类成员来存储descriptor set的句柄，并且使用**vkAllocateDescriptorSets**来为它们分配内存。

```c++
	//每个运行帧的descriptor set
	std::vector<VkDescriptorSet> descriptorSets;

	//为运行中的每一帧创建一个descriptor set
	descriptorSets.resiz(MAX_FRAMES_IN_FLIGHTS);
	if (vkAllocateDescriptorSets(device, allocInfo, descriptorSets.data()) !=VK_SUCCESS)
	{
		throw std::runtime_error("failed to allocate descriptor sets!");
	}
```

我们**不需要显式地销毁descriptor sets**，因为它们会在descriptor pool销毁时自动释放。对vkAllocateDescriptorSets的调用会分配descriptor sets，**每个descriptor sets都有一个uniform buffer dedscriptor**。

```c++
	//一旦window被关闭，我们将在**cleanup**函数中确保释放我们用到的所有资源
	void cleanup()
	{
		...
		//销毁descriptor pool
		vkDestroyDescriptorPool(device, descriptorPool, nullptr);

		//销毁descriptor set layout
		vkDestroyDescriptorSetLayout(device, descriptossrSetLayout, nullptr);
		...
	}
```

现在descriptor sets已经被分配了，但是**其中的descriptor需要被配置**，我们现在增加一个循环来填充每个descriptor。

```c++
		for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHTS; i++)
		{

		}
```

**descriptors指向buffers**，比如我们的uniform buffer descriptor，通过**VkDescriptorBufferInfo**结构体来配置。该结构体指定了**buffer**和**其中包含了该descriptor数据的区域**。

```c++
		for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHTS; i++)
		{
			//为每个descriptor set配置其中的descriptor，每个descriptor引用一个uniform buffer
			VkDescriptorBufferInfo bufferInfo{};
			bufferInfo.buffer = uniformBuffers[i];
			bufferInfo.offset = 0;
			bufferInfo.range = sizeof(UniformBufferObject);
		}
```


如果我们会重写整个buffer，比如在这里我们会做的，那么我们也可以对range使用值VK_WHOLE_SIZE。descriptors的配置使用**vkUpdateDescriptorSets**函数来更新，其使用一个**VkWriteDescriptorSet数组**作为参数。

```c++
			//更新descriptor set
			VkWriteDescriptorSet descriptorWrite{};
			descriptorWrite.sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
			descriptorWrite.dstSet = descriptorSets[i];
			descriptorWrite.dstBinding = 0;
			descriptorWrite.dstArrayElement = 0;
```

前两个字段指定了**要更新的descriptor set**和**对应binding**。我们给uniform buffer的binding索引是0。记住descriptors可以是数组，所以我们需要指定要更新的数组中的第一个索引（dstArrayElement）。我们现在不使用数组，所以索引为0。

```c++
			descriptorWrite.descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
			descriptorWrite.descriptorCount = 1;
```

我们需要再次指定descriptor的类型（感觉有点冗余）。**使用数组来一次更新多个descriptor sets是可以的**，从dstArrayElement索引开始。**descriptorCount**字段指定了我们想更新多少个元素。

```c++
			descriptorWrite.pBufferInfo = &bufferInfo;
			descriptorWrite.pImageInfo = nullptr; // Optional
			descriptorWrite.pTexelBufferView = nullptr; // Optional
```

最后一个字段引用一个包含实际配置descriptors的descriptorCount数量的数组。我们实际需要使用三种descriptor之一，取决于descriptor的类型。**pBufferInfo**字段用于指向buffer数据的descriptors，**pImageInfo**用于指向image数据的descriptors，**pTexelBufferView**用于指向buffer views的descriptors。我们的descriptor是基于buffers的，所以我们使用**pBufferInfo**。

```c++
			vkUpdateDescriptorSets(device, 1, &descriptorWrite, 0, nullptr);
```

我们使用**vkUpdateDescriptorSets**来应用更新，它接受两类数组作为参数：一个**VkWriteDescriptorSet**数组和一个**VkCopyDescriptorSet**数组，后者可以用于**互相拷贝descriptors**。

#### 8 使用描述符集 Using descriptor sets

现在我们需要更新recordCommandBuffer函数，**使用vkCmdBindDescriptorSets来实际为每一帧的正确的descriptor set实际绑定到着色器中的descriptor**（原文好绕，实际就是**为每一帧绑定正确的descriptor set**）。其需要在**vkCmdDrawIndexed**之前完成。

```c++
		//绑定index buffer
		vkCmdBindIndexBuffer(commandBuffer, indexBuffer, 0, VK_INDEX_TYPE_UINT16);
		//为每一帧的正确的descriptor set实际绑定到着色器中的descriptor，好绕，实际就是为每一帧绑定正确的descriptor set
		vkCmdBindDescriptorSets(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, pipelineLayout, 0, 1, &descriptorSets[currentFrame], 0, nullptr);
```

不像vertex buffer和index buffer，**descriptor sets不是graphics pipelines独有的**。 因此我们需要**指定想把descriptor sets绑定到graphics pipeline还是compute pipeline**。下一个参数descriptors基于的**pipelineLayout**。后三个参数指定了**第一个descriptor set的索引**，**要绑定的sets的数量**，**要绑定的sets的数组**。我们后面会回到这里。最后两个参数指定了**一个offsets数组，用于dynamic descriptors**。后面的章节我们会关注它们。

如果我们现在运行程序，我们会注意到没有任何可见的东西。问题是因为我们在project矩阵中做的Y轴翻转，现在顶点的绘制顺序是逆时针而不是顺时针。这会导致**背面剔除**，并阻止绘制任何几何体。到**createGraphicsPipeline**函数中，然后修改**VkPipelineRasterizationStateCreateInfo**中修正**frontFace**。

```c++
		rasterizer.cullMode = VK_CULL_MODE_BACK_BIT; //面剔除类型
		rasterizer.frontFace = VK_FRONT_FACE_COUNTER_CLOCKWISE; //确定正面方向
```

现在运行程序，我们会看下如下图的图像。

![20240209113420](https://raw.githubusercontent.com/recaeee/PicGo/main/20240209113420.png)

矩形已更正为正方形，因为projection矩阵会校正aspect。updateUniformBuffer考虑到了窗口的resize，所以我们不需要在recreateSwapChain中重建descriptor set。

#### 9 对齐要求 Alignment requirements

到目前为止我们忽略了一件事，**C++结构体中的数据应该如何与着色器中的uniform定义相匹配**。看起来在两者中使用相同的类型似乎很简单明了。

```c++
struct UniformBufferObject {
    glm::mat4 model;
    glm::mat4 view;
    glm::mat4 proj;
};

layout(binding = 0) uniform UniformBufferObject {
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;
```

但是，实际上没想的这么简单。例如，我们将结构体和shader修改成如下。

```c++
struct UniformBufferObject {
    glm::vec2 foo;
    glm::mat4 model;
    glm::mat4 view;
    glm::mat4 proj;
};

layout(binding = 0) uniform UniformBufferObject {
    vec2 foo;
    mat4 model;
    mat4 view;
    mat4 proj;
} ubo;
```
重新编译shader，然后运行程序，我们会发现我们色彩缤纷的方块消失了，这是因为我们没考虑到**对齐要求alignment requirements**。

Vulkan期望我们结构体中的数据有特定的内存对齐，例如：

1. 标量必须按N（=4个字节（即32位浮点数）对齐）。
2. vec2必须按2N（=8个字节）对齐。
3. vec3或者vec必须按4N（=16个字节）对齐。
4. 嵌套结构必须通过其成员的基本对齐方式进行对齐，四舍五入到16的倍数。
5. mat4矩阵的对齐方式必须和vec4一样。

我们可以在[the specification](https://www.khronos.org/registry/vulkan/specs/1.3-extensions/html/chap15.html#interfaces-resources-layout)中找到所有的对齐要求。（对齐要求很重要吖，这种错误记得是Validation也查不出来的）。

原本我们的shader只有3个mat4字段，已经满足了对齐要求。因为每个mat4是4x4x4=64字节大小，model偏移为0，view偏移为64，proj偏移为128。所有的偏移都是16的整数倍。

而新的结构体从vec2开始，其大小为8字节，因此会改变所有的偏移量。现在，model偏移为8，view偏移为72，proj偏移为136，没有一个是16的整数倍。为了修复该问题，我们**可以使用C++11中引入的alignas说明符**。

```c++
struct UniformBufferObject {
    glm::vec2 foo;
    alignas(16) glm::mat4 model;
    glm::mat4 view;
    glm::mat4 proj;
};
```

如果我们现在再编译并运行程序，我们会看到shader正确地得到了矩阵值。

幸运的是，有个办法可以让我们几乎不用考虑对齐要求问题，我们可以**定义GLM_FORCE_DEFAULT_ALIGNED_GENTYPES**在include GLM之前。

```c++
//数学库，包含vector和matrix等，用于声明顶点数据、3D MVP变换等
#define GLM_FORCE_RADIANS
//使GLM的类型满足内存对齐
#define GLM_FORCE_DEFAULT_ALIGNED_GENTYPES
#include <glm/glm.hpp>
```

这会让GLM使用已为我们指定内存对齐要求的vec2和mat4版本。如果我们增加了该定义，我们就可以移除alignas说明符了。

不幸的是，如果我们使用了嵌套结构，此方法可能会崩溃。考虑如下C++定义。

```c++
struct Foo {
    glm::vec2 v;
};

struct UniformBufferObject {
    Foo f1;
    Foo f2;
};
```

以及如下shader定义。

```c++
struct Foo {
    vec2 v;
};

layout(binding = 0) uniform UniformBufferObject {
    Foo f1;
    Foo f2;
} ubo;
```

在这个例子中，f2的offset为8，而它应该具有16的offset，这是因为它是嵌套结构。在这种情况下，我们必须自己指定alignment。

```c++
struct UniformBufferObject {
    Foo f1;
    alignas(16) Foo f2;
};
```

考虑到会有这样的陷阱，所以我们需要养成**始终显式地明确alignment**的习惯。这样，我们就不会因为对齐错误的奇怪症状而感受苦痛。

```c++
struct UniformBufferObject {
    alignas(16) glm::mat4 model;
    alignas(16) glm::mat4 view;
    alignas(16) glm::mat4 proj;
};
```

#### 10 多个描述符集 Multiple descriptor sets

正如一些结构体和函数暗示的一样，实际上我们**可以同时绑定多个descriptor sets**。我们需要在创建pipeline layout的时候为每个descriptor set指定一个descriptor layout。Shader可以通过如下方式引用指定的descriptor sets。

```c++
layout(set = 0, binding = 0) uniform UniformBufferObject { ... }
```

我们可以使用该特**性将每个对象不同的descriptor和共享的descriptor放入单独的descriptor sets中**。在这种情况下，我们**可以避免在draw call之间重新绑定大多数descriptors**，这可能会更高效。

#### 参考
1. 题图来自画师wlop
2. https://www.khronos.org/registry/vulkan/specs/1.3-extensions/html/chap15.html#interfaces-resources-layout

