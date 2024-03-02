---
layout:     post
title:      "【VulkanTutorial学习笔记——10】Vertex buffers"
subtitle:   ""
date:       2024-02-08 12:00:00
author:     "recaeee"
header-img: "img/in-post/default-bg.jpg"
tags:
    - Vulkan
---
# 【VulkanTutorial学习笔记——10】Vertex buffers

![20240208143903](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240208143903.png)

#### 0 写在前面
本系列为个人在跟随Vulkan Tutorial学习过程中记录的一些笔记，一是防止自己看着看着弃坑，二是希望能在看过之后多少留些印象，三是想在啥时候忘记了能快速地回忆一些Vulkan知识点。

出于以上这些目的，该系列的学习笔记可能会写得较随意一些，并且对于主观判断不重要的东西可能会略过，因此也不保证该系列笔记的学习质量，可能有些东西不经过脑子就会写出来。

我是一个Vulkan的小白，对于Vulkan尚不熟悉，笔记中可能存在错误，或者概念不明确的地方，也希望看的人可以批评斧正~

这一章，我们终于结束了Draw a Triangle的部分，来到了Vertex buffers。对于Vulkan，咱也越来越熟悉了呢~这一节中，我们会学会如何使用vertex buffers以及index buffer，其中我们会接触到一个新的概念VkDeviceMemory，其包括了Vulkan中不同的可分配的内存类型（比如只供GPU访问的显存、CPU可见的随时同步的内存等等）。话不多说，开冲~

原教程链接：
https://vulkan-tutorial.com/Introduction
本章节链接：
https://vulkan-tutorial.com/Vertex_buffers/Vertex_input_description
https://vulkan-tutorial.com/Vertex_buffers/Vertex_buffer_creation
https://vulkan-tutorial.com/Vertex_buffers/Staging_buffer
https://vulkan-tutorial.com/Vertex_buffers/Index_buffer

#### 1 顶点着色器 Vertex shader

在接下来的几个小结，我们将会使用**内存中的vertex buffer**代替在顶点着色器中硬编码的顶点数据。我们会使用最简单的方法创建按一个CPU可见的buffer，并且使用**memcpy**将顶点数据直接复制到其中，然后我们会了解到如何使用**暂存缓冲区staging buffer**来将顶点数据复制到更高性能的内存中。

首先改变顶点着色器让其代码中不再包含顶点数据。在顶点着色器中，**使用in关键字来从顶点缓冲区vertex buffer中获取输入**。

```c++
#version 450

layout(location = 0) in vec2 inPosition;
layout(location = 1) in vec3 inColor;

layout(location = 0) out vec3 fragColor;

void main()
{
	gl_Position = vec4(inPosition, 0.0, 1.0);
	fragColor = inColor;
}
```

**inPosition**和**inColor**变量是**顶点属性vertex attributes**。它们是在vertex buffer中指定的每顶点属性，就像我们手动使用两个数组指定的每顶点位置和颜色。接下来重新编译顶点着色器。

就像**fragColor**一样，**layout(locatio n= x)**将索引分配给我们，稍后我们可以用来引用片元着色器的输入。了解**有一些类型使用多个slots非常重要**的，比如**dvec3（64bit的向量）**。这意味着在它之后的索引必须至少增加2（也就是说1个slot占用32bit）。我们可以在[OpenGL wiki](https://www.khronos.org/opengl/wiki/Layout_Qualifier_(GLSL))中查看layout限定符的更多信息。

使用dvec3时location的用例如下。

```c++
layout(location = 0) in dvec3 inPosition;
layout(location = 2) in vec3 inColor;
```

#### 2 顶点数据 Vertex data

我们现在要**将顶点数据从shader代码中移动到程序代码中的一个数组**。首先**include GLM库**，它提供给了我们线性代数相关的类型，比如向量和矩阵。我们将会使用这些类型声明顶点的position和color向量。

```c++
//数学库，包含vector和matrix等，用于声明顶点数据
#include <glm/glm.hpp>
```

创建一个新的名为Vertex的结构体，包含我们将会在顶点着色器中使用到的2个属性。

```c++
	//顶点着色器输入
	struct Vertex {
		glm::vec2 pos;
		glm::vec3 color;
	};
```

GLM方便地提供给了我们**完全匹配shader语言的C++ vector类型**.

```c++
	const std::vector<Vertex> vertices = {
		{{0.0f, -0.5f}, {1.0f, 0.0f, 0.0f}},
		{{0.5f, 0.5f}, {0.0f, 1.0f, 0.0f}},
		{{-0.5f, 0.5f}, {0.0f, 0.0f, 1.0f}}
	}；
```

现在我们使用Vertex结构体声明一个向量数组。我们使用和之前一样的position和color值，但是现在它们被组合到了一个顶点数组中，其被称为**交错顶点属性interleaving vertex attributes**（交错在这里的意思应该是数组中的数据是position和color交错排布的）。

#### 3 绑定描述 Binding descriptions

下一步是告诉Vulkan**在上传到GPU显存后如何将此数据格式传递给顶点着色器**。我们需要有两个结构体类型来传递这些信息。

第一个结构体是**VkVertexInputBindingDescription**，我们将会增加一个成员函数到Vertex结构体以使用正确的数据来传递它。

```c++
	//顶点着色器输入
	struct Vertex {
		glm::vec2 pos;
		glm::vec3 color;

		static VkVertexInputBindingDescription getBindingDescription()
		{
			VkVertexInputBindingDescription bindingDescription{};

			return bindingDescription;
		}
	};
```

**顶点绑定Vertex binding**描述了**在整个顶点中从内存加载数据的速率**。它指定了**data entries之间的字节数以及是否在每个顶点后或每个实例之后移动到下一个data entry**。（主要就是明确每个顶点之间的步长等）

```c++
			VkVertexInputBindingDescription bindingDescription{};
			bindingDescription.binding = 0;
			bindingDescription.stride = sizeof(Vertex);
			bindingDescription.inputRate = VK_VERTEX_INPUT_RATE_VERTEX;
```

我们所有的每顶点数据一起被pack到了一个数组里，所以我们只需要有一个binding。**binding**参数指定了**当前bingding在binding数组中的索引**。**stride**参数指定了**从一个entry到下一个entry的字节数**（也就是步长）。**inputRate**参数可以是以下值：

1. **VK_VERTEX_INPUT_RATE_VERTEX**：每顶点后移动到下一个data entry。
2. **VK_VERTEX_INPUT_RATE_INSTANCE**：每实例之后移动到下一个data entry。

我们现在不会使用instance渲染，所以我们使用每顶点数据。

#### 4 属性描述 Attribute descriptions

第二个结构体**VkVertexInputAttributeDescription**描述了**如何处理顶点输入**。我们将增加另一个helper函数到Vertex中来填充这些结构体。

```c++
		static std::array<VkVertexInputAttributeDescription, 2> getAttributeDescriptions()
		{
			std::array<VkVertexInputAttributeDescription, 2> attributeDescriptions{};

			return attributeDescriptions;
		}
```

正如函数原型所示，我们将有2个这样的结构体。一个**属性描述Attribute description结构体**描述了**如何从源自绑定描述Binding descrription的顶点数据块中提取顶点属性**。我们有2个Attribute，位置和颜色，所以我们需要2个attribute description。

```c++
		static std::array<VkVertexInputAttributeDescription, 2> getAttributeDescriptions()
		{
			//属性描述Attribute description结构体描述了如何从源自绑定描述Binding descrription的顶点数据块中提取顶点属性
			std::array<VkVertexInputAttributeDescription, 2> attributeDescriptions{};

			attributeDescriptions[0].binding = 0;
			attributeDescriptions[0].location = 0;
			attributeDescriptions[0].format = VK_FORMAT_R32G32_SFLOAT;
			attributeDescriptions[0].offset = offsetof(Vertex, pos);

			return attributeDescriptions;
		}
```

**binding**参数告诉了Vulkan**每顶点数据来自哪个binding**。**location**参数**引用顶点着色器中input的location指令**（即数据对应顶点着色器中哪个location）。顶点着色器的输入location 0是position，其拥有2个32bit浮点分量。

format参数描述了attribute的数据类型。有点令人困惑的是，**format是使用和color formats相同的枚举值来指定的**。以下是一些常用的shader类型和formats：

- float:VK_FORMAT_R32_SFLOAT
- vec2:VK_FORMAT_R32G32_SFLOAT
- vec3:VK_FORMAT_R32G32B32_SFLOAT
- vec4:VK_FORMAT_R32G32B32A32_SFLOAT

正如以上的例子，我们应该使用颜色通道数量与着色器数据类型中的components数量相匹配的format。**允许使用多于着色器中components数量的通道**，但它们将被默认丢弃。如果通道的数量少于着色器中components的数量，BGAcomponents会使用默认值（0，0，1）。**颜色类型（SFLOAT，UINT，SINT）和位宽应该匹配shader输入的类型**。例子如下：

- ivec2:VK_FORMAT_R32G32_SINT，32位有符号整数的2分量向量
- uvec4:VK_FORMAT_R32G32B32A32_UINT，32位无符号整数的4分量向量
- double:VK_FORMAT_R64_SFLOAT，双精度（64位）的浮点数。

**format**参数隐式地定义了**attribute数据的字节大小**，并且**offset**参数指定了**从每顶点数据的开头偏移多少字节开始读取**。binding一次只会加载一个Vertex，position attribute在Vertex结构体中的偏移量为0字节。偏移量可以使用**offsetof**宏自动计算。

```c++
			attributeDescriptions[1].binding = 0;
			attributeDescriptions[1].location = 1;
			attributeDescriptions[1].format = VK_FORMAT_R32G32B32_SFLOAT;
			attributeDescriptions[1].offset = offsetof(Vertex, color);
```

color attribute也使用同样的方式描述。

#### 5 管线顶点输入 Pipeline vertex input

我们现在需要通过**在createGraphicsPipeline中设置graphics pipeline接收以上格式的顶点数据**。找到**vertexInputInfo**结构体，然后修改它来引用2个descriptions。

```c++
		//顶点输入
		VkPipelineVertexInputStateCreateInfo vertexInputInfo{};
		vertexInputInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_VERTEX_INPUT_STATE_CREATE_INFO;

		auto bindingDescription = Vertex::getBindingDescription();
		auto attributeDescriptions = Vertex::getAttributeDescriptions();

		vertexInputInfo.vertexBindingDescriptionCount = 1;
		vertexInputInfo.vertexAttributeDescriptionCount = static_cast<uint32_t>(attributeDescriptions.size());
		vertexInputInfo.pVertexBindingDescriptions = &bindingDescription;
		vertexInputInfo.pVertexAttributeDescriptions = attributeDescriptions.data();
```

现在，pipeline准备接受vertices容器的顶点数据，并将其传递给我们的顶点着色器。如果我们现在在Validation启用的情况下运行程序，我们 将会看到它会上报binding中无绑定的vertex buffer。所以，我们的下一步是**创建一个vertex buffer，然后将顶点数据移动到vertex buffer中**，之后GPU就可以获取它了。

#### 5 Buffer创建 buffer creation

Vulkan中的缓冲区Buffers是**用于存储显卡可以读取的任意数据的内存区域**（即显存）。它们可以用于存储顶点数据（即我们这一节要做的），但它们也可以用于许多其他的用途，我们会在后面的章节来探索。不像我们之前处理的Vulkan objects，**buffers不会自动为自己分配内存**。之前几章的工作展示了Vulkan API让编程者控制几乎所有事情，内存管理也是其中一个。

创建一个新函数**createVertexBuffer**，在initVulkan的createCommandBuffers之前调用它。

```c++
	//initVulkan函数来用于实例化Vulkan objects私有成员
	void initVulkan()
	{
		createInstance();
		setupDebugMessenger();
		createSurface();
		pickPhysicalDevice();
		createLogicalDevice();
		createSwapChain();
		createImageViews();
		createRenderPass();
		createGraphicsPipeline();
		createFramebuffers();
		createCommandPool();
		createVertexBuffer();
		createCommandBuffers();
		createSyncObjects();
	}

	void createVertexBuffer()
	{

	}
```

创建一个buffer需要我们填充一个**VkBufferCreateInfo**结构体。

```c++
		VkBufferCreateInfo bufferInfo{};
		bufferInfo.sType = VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO;
		bufferInfo.size = sizeof(vertices[0]) * vertices.size();
```

第一个字段**size**指定了**buffer以字节为单位的大小**。我们直接使用sizeof计算顶点数据的字节大小。

```c++
		bufferInfo.usage = VK_BUFFER_USAGE_VERTEX_BUFFER_BIT;
```
第二个字段**usage**表明了**buffer中的数据的用途**，可以使用按位或运算来指定多个用途。我们将会用作vertex buffer，后续章节我们会关注其他用途。

```c++
		bufferInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;
```

和swap chain images一样，**buffer也可以被一个指定的queue family占用，或者同时被多个queue family共享**。在这里，buffer只会被graphics queue使用，所以我们使用独占访问exclusive access。

**flags**参数用来**配置稀疏缓冲内存**，目前我们用不到，我们将其置为默认值0。

现在我们可以使用**vkCreateBuffer**来创建buffer，定义一个类成员来存储buffer的句柄。

```c++
	//Vertex buffer
	VkBuffer vertexBuffer;

	void createVertexBuffer()
	{
		VkBufferCreateInfo bufferInfo{};
		bufferInfo.sType = VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO;
		bufferInfo.size = sizeof(vertices[0]) * vertices.size();//size指定了buffer以字节为单位的大小
		bufferInfo.usage = VK_BUFFER_USAGE_VERTEX_BUFFER_BIT;//usage表明了buffer中的数据的用途，可以使用按位或运算来指定多个用途
		bufferInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;//buffer也可以被一个指定的queue family占用，或者同时被多个queue family共享

		if (vkCreateBuffer(device, &bufferInfo, nullptr, &vertexBuffer) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to create vertex buffer!");
		}
	}
```

buffer应该可用于渲染指令，直到程序结束，并且它不依赖于swap chain，所以我们在原始的cleanup中消耗它。

```c++
	//一旦window被关闭，我们将在**cleanup**函数中确保释放我们用到的所有资源
	void cleanup()
	{
		cleanupSwapChain();

		//销毁vertex buffer
		vkDestroyBuffer(device, vertexBuffer, nullptr);
		...
	}
```

#### 6 内存要求 Memory requirements

**buffer已经被创建了，但实际上并未其分配任何内存**。给该buffer分配内存的第一步是**使用恰当命名的vkGetBufferMemoryRequirements函数查询其内存需求**。

```c++
		//buffer已经被创建了，但实际上并未其分配任何内存
		//使用恰当命名的vkGetBufferMemoryRequirements函数查询其内存需求
		VkMemoryRequirements memRequirements;
		vkGetBufferMemoryRequirements(device, vertexBuffer, &memRequirements);
```

**VkMemoryRequirements**结构体拥有3个字段：
- **size**：以字节为单位，需要的内存大小，可能会和bufferInfo.size不一致。
- **alignment**：buffer在分配的内存区域中开始的偏移量（以字节位单位），取决于bufferInfo.usage和bufferInfo.flags。
- **memoryTypeBits**：适合该buffer的内存类型的位字段

**显卡可以提供不同类型的内存来分配，每种类型的内存在允许的操作和性能特征方面有所不同**。我们需要结合buffer的要求和我们的程序的需求找到正确的内存类型累使用，所以我们接下来创建**findMemoryType**函数。

```c++
	//为要分配的buffer找到正确的合适的显存类型
	uint32_t findMemoryType(uint32_t typeFilter, VkMemoryPropertyFlags properties)
	{

	}
```

首先我们需要使用**vkGetPhysicalDeviceMemoryProperties**来查询所有可用的内存类型。

```c++
		//查询所有可用的内存类型
		VkPhysicalDeviceMemoryProperties memProperties;
		vkGetPhysicalDeviceMemoryProperties(physicalDevice, &memProperties);
```

**VkPhysicalDeviceMemoryProperties**结构体有2个数组**memoryTpyes**和**memoryHeaps**。**内存堆Memory heaps是不同的内存资源，例如专用VRAM、RAM中用于VRAM耗尽时的交换内存**（快进到爆显存）。目前我们只关注内存的类型，不关注它来自哪个内存堆，但是我们可以想到它会影响到性能。

首先找到适合该buffer自身的内存类型。

```c++
		//找到buffer适合的内存类型
		for (uint32_t i = 0; i < memProperties.memoryTypeCount; i++)
		{
			if (typeFilter & (1 << i))
			{
				return i;
			}
		}

		throw std::runtime_error("failed to find suitable memory type!");
```

**typeFilter**参数将会被用于指定**适合的内存类型的位字段**。这意味着我们可以通过简单的遍历并检查相应的位是否设置为1来找到合适的内存类型索引。

但是，我们并不只对适合于vertex buffer的内存类型感兴趣。我们需要能够将顶点数据写入到该内存中。memoryTypes数组由**VkMemoryType**结构组成，这些结构指定每种内存的heap和properties。这些properties定义了内存的特殊feature，比如能够**映射**它，由此我们可以从CPU写入它（将显存中的数据映射到CPU内存地址空间中，以便CPU能够直接读取或写入显存中的数据，而不需要复制或者传输。好比映射完之后，CPU对显存还是内存是无感的，直接当作内存写入就行。具体的写入实现交给显卡驱动）。可从CPU写入的properties标识为**VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT**，但是我们也需要使用**VK_MEMORY_PROPERTY_HOST_COHERENT_BIT** property。我们将会在映射内存时看到为什么。

现在我们修改这个循环，来检查这两个property。

```c++
		//找到buffer适合的内存类型
		for (uint32_t i = 0; i < memProperties.memoryTypeCount; i++)
		{
			if (typeFilter & (1 << i) && (memProperties.memoryTypes[i].propertyFlags & properties) == properties)
			{
				return i;
			}
		}
```

我们可能有超过一个期望的property，所以我们使用位与运算结果等于期望的properties位字段来检查。如果有一个适合buffer的内存类型，并且拥有所有我们需要的properties，然后我们返回它的索引，否则我们抛出一个异常。

#### 7 内存分配 Memory allocation

现在我们有办法确定正确的内存类型，所以我们可以**通过填充VkMemoryAllocateInfo结构体来实际分配内存**。

```c++
		//实际分配显存
		VkMemoryAllocateInfo allocInfo{};
		allocInfo.sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
		allocInfo.allocationSize = memRequirements.size;
		allocInfo.memoryTypeIndex = findMemoryType(memRequirements.memoryTypeBits, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT);
```

内存分配现在就只需要指定size和type了，这两者都源自vertex buffer的内存要求和需要的property。创建一个类成员来存储实际分配的内存（显存）的句柄，并且使用**vkAllocateMemory**来实际分配内存。

```c++
	//Vertex buffer
	VkBuffer vertexBuffer;
	//实际分配的vertex buffer的显存
	VkDeviceMemory vertexBufferMemory;

	...

	
	if (vkAllocateMemory(device, &allocInfo, nullptr, vertexBufferMemory) != VK_SUCCESS)
	{
		throw std::runtime_error("failed to allocate vertex buffer memory!");
	}
```

如果内存分配成功，那么我们现在可以**使用vkBindBufferMemory将此内存与buffer关联起来**（可以看到buffer本身只是个句柄，vkDeviceMemory才拥有实际分配的内存）。

```c++
		//使用vkBindBufferMemory将此内存与buffer关联起来
		vkBindBufferMemory(device, vertexBuffer, vertexBufferMemory, 0);
```

前三个参数非常明确了，第四个参数是内存区域的偏移。因为我们分配的内存是单独给vertex buffer的，所以offset为0.如果offset非0，则**需要能被memRequirements.alignment整除**。

当然，正如C++分配的动态内存，我们分配的内存也需要在某些时间点释放。绑定到一个buffer的内存通常在buffer不再被使用后释放，所以我们在buffer销毁时释放它。

```c++
	//一旦window被关闭，我们将在**cleanup**函数中确保释放我们用到的所有资源
	void cleanup()
	{
		cleanupSwapChain();

		//销毁vertex buffer
		vkDestroyBuffer(device, vertexBuffer, nullptr);
		//销毁与vertex buffer绑定的内存
		vkFreeMemory(device, vertexBufferMemory, nullptr);
		...
	}
```

#### 8 填充顶点缓冲区 Filling the vertex buffer

现在让我们将顶点数据拷贝到buffer中（也就是内存->显存）。我们通过**vkMapMemory**来**将buffer对应的显存memory映射到CPU可以访问的内存RAM中**。

```c++
		//将顶点数据拷贝到buffer中
		//首先将显存映射到CPU可以访问的内存中
		void* data;
		vkMapMemory(device, vertexBufferMemory, 0, bufferInfo.size, 0, &data);
```

该函数允许我们访问由offset和size定义的指定内存资源的区域。在这里，offset为0，size为bufferInfo.size，显而易见。也可以**通过指定特殊值VK_WHOLE_SIZE来映射到内存的全部空间**。倒数第二个参数可以用于指定flags，但是在目前的API中还没有可用的flag，它必须是0。最后一个参数指定了mapped memory的指针输出。

```c++
		//将顶点数据拷贝到buffer中
		//首先将显存映射到CPU可以访问的内存中
		void* data;
		vkMapMemory(device, vertexBufferMemory, 0, bufferInfo.size, 0, &data);
		//将内存中的顶点数据拷贝到显存
		memcpy(data, vertices.data(), (size_t)bufferInfo.size);
		//解除映射
		vkUnmapMemory(device, vertexBufferMemory);
```

现在我们使用**memcpy**将顶点数据拷贝到映射内存中，然后使用**vkUnmapMemory**再对其解映射。不幸的是，**驱动也许不会立刻将数据拷贝到buffer对应的显存memory中**，例如因为caching。也有可能**对buffer的写入在映射内存中还不可见**。有两个方法来处理这个问题：
- 使用与主机一致的堆内存，使用**VK_MEMORY_PROPERTY_HOST_COHERENT_BIT**指示。
- 在写入到映射内存后调用**vkFlushMappedMemoryRanges**，并且在从映射内存读取前调用**vkInvalidateMappedMemoryRanges**。

我们使用第一个方法，它**保证了mapped memory永远与分配的内存的内容匹配**。要记住比起显式的flushing，这也许会导致轻微的性能变差，但是我们会在后续章节看到为什么它没什么关系。

Flushing内存范围或者使用与主机一致的内存堆意味着驱动会意识到我们对buffer的写入，但它**不意味着它们对GPU实际可见**。将数据传输到GPU上是发生在后台的操作，规范只是告诉我们，**保证在下次调用vkQueueSubmit时完成**。

#### 9 绑定顶点缓冲区 Binding the vertex buffer

剩下的全部工作是**在渲染操作期间绑定vertex buffer**。我们拓展**recordCommandBuffer**函数来实现它。

```c++
		//绑定graphics pipeline
		vkCmdBindPipeline(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, graphicsPipeline);
        ...
		//绑定vertex buffer
		VkBuffer vertexBuffers[] = { vertexBuffer };
		VkDeviceSize offsets[] = { 0 };
		vkCmdBindVertexBuffers(commandBuffer, 0, 1, vertexBuffers, offsets);
		//绘制指令
		vkCmdDraw(commandBuffer, static_cast<uint32_t>(vertices.size()), 1, 0, 0);
```

**vkCmdBindVertexBuffers**函数用于**绑定vertex buffer到bindings**。其第二、第三个参数指定了vertex buffers绑定的bindings的offset和数量。最后两个参数制定了要绑定的vertex buffers数组和开始读取顶点数据时的字节offset。我们也需要改变调用vkCmdDraw时的参数，传递buffer中的顶点数量，而不是硬编码的数字3。

现在运行程序，我们会再次看到和之前一样的三角形。

![20240207101627](https://raw.githubusercontent.com/recaeee/PicGo/main/20240207101627.png)

我们可以尝试通过改变vertices数组来改变上顶点的颜色。（还挺好看）

![20240207101735](https://raw.githubusercontent.com/recaeee/PicGo/main/20240207101735.png)

在下一小节，我们会关注另一个将顶点数据拷贝到vertex buffer中的方法，它拥有更好的性能，但是需要额外做一些工作。

#### 10 暂存缓冲区 Staging buffer

我们现在拥有的vertex buffer可以正常工作，但这种允许CPU访问的内存类型也许不是对于显卡自身读取最优的内存类型。最优的内存拥有**VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT** flag，并且通常在使用独显时CPU无法访问这种内存。在这一节，我们将会创建2个vertex buffers。**在CPU可访问的内存中的一个暂存缓冲区stating buffer**，用于将数据从顶点数组上传到**device local memory（显存）中的最终vertex buffer**。然后我们将使用一个buffer拷贝指明来将数据从staging buffer移动到实际的vertex buffer。

#### 11 传输队列 Transfer queue

**buffer copy指令**需要一个支持tranfer操作的queue family，其使用**VK_QUEUE_TRANSFER_BIT**指示。好消息是任何与VK_QUEUE_GRAPHICS_BIT或者VK_QUEUE_COMPUTE_BIT兼容的queue family已经隐式地支持了VK_QUEUE_TRANSFER_BIT操作。在这些情况下，我们并不需要将其显式列出在queueFlags中。

如果我们想要挑战一下，我们仍然可以尝试为transfer操作使用单独的一个queue family。它需要我们对我们的程序做出以下修改：
- 修改QueueFamilyIndices和findQueueFamilies来显式寻找一个包含VK_QUEUE_TRANSFER_BIT而不是VK_QUEUE_GRAPHICS_BIT的queue family。
- 修改createLogicalDevice来申请一个transfer queue的句柄。
- 为command buffers创建第二个command pool用于提交到transfer queue family。
- 将资源的sharingMode改为VK_SHARING_MODE_CONCURRENT并且同时指定graphics和transfer queue families。
- 提交传输指令，比如vkCmdCopyBuffer（我们将会用到），到transfer queue，而不是graphics queue。

这是有些工作量的，但是它会告诉我们一些关于queue families之间资源如何共享的知识。

#### 12 抽象缓冲区创建 Abstracting buffer creation

因为我们在这一章节会创建许多buffers，因此将buffer的创建移动到helper function是一个好主意。创建一个新函数**createBuffer**，并且将createVertexBuffer（除了映射）的代码移动到其中。

```c++
	//创建buffer
	void createBuffer(VkDeviceSize size, VkBufferUsageFlags usage, VkMemoryPropertyFlags properties, VkBuffer& buffer, VkDeviceMemory& bufferMemory)
	{
		VkBufferCreateInfo bufferInfo{};
		bufferInfo.sType = VK_STRUCTURE_TYPE_BUFFER_CREATE_INFO;
		bufferInfo.size = size;//size指定了buffer以字节为单位的大小
		bufferInfo.usage = usage;//usage表明了buffer中的数据的用途，可以使用按位或运算来指定多个用途
		bufferInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;//buffer也可以被一个指定的queue family占用，或者同时被多个queue family共享

		if (vkCreateBuffer(device, &bufferInfo, nullptr, &buffer) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to create vertex buffer!");
		}

		//buffer已经被创建了，但实际上并未其分配任何内存
		//使用恰当命名的vkGetBufferMemoryRequirements函数查询其内存需求
		VkMemoryRequirements memRequirements;
		vkGetBufferMemoryRequirements(device, buffer, &memRequirements);

		//实际分配显存
		VkMemoryAllocateInfo allocInfo{};
		allocInfo.sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
		allocInfo.allocationSize = memRequirements.size;
		allocInfo.memoryTypeIndex = findMemoryType(memRequirements.memoryTypeBits, properties);

		if (vkAllocateMemory(device, &allocInfo, nullptr, &bufferMemory) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to allocate vertex buffer memory!");
		}

		//使用vkBindBufferMemory将此内存与buffer关联起来
		vkBindBufferMemory(device, buffer, bufferMemory, 0);
	}
```

确保增加了参数buffer size、memory properties和usage，由此我们可以使用这个函数创建许多不同类型的buffer。最后两个参数是要写入句柄的输出变量。

我们现在可以将buffer的创建和memory的分配代码从createVertexBuffer中移除，然后调用createBuffer。

```c++
	void createVertexBuffer()
	{
		//创建buffer与关联的VkdeviceMemory
		VkDeviceSize bufferSize = sizeof(vertices[0]) * vertices.size();
		createBuffer(bufferSize, VK_BUFFER_USAGE_VERTEX_BUFFER_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, vertexBuffer, vertexBufferMemory);

		//将顶点数据拷贝到buffer中
		//首先将显存映射到CPU可以访问的内存中
		void* data;
		vkMapMemory(device, vertexBufferMemory, 0, bufferSize, 0, &data);
		//将内存中的顶点数据拷贝到显存
		memcpy(data, vertices.data(), (size_t)bufferSize);
		//解除映射
		vkUnmapMemory(device, vertexBufferMemory);
	}
```

运行下我们的程序来保证vertex buffer正常工作。good~

#### 13 使用暂存缓冲区 Using a staging buffer

接下来，我们来改变createVertexBuffer函数，**使用一个host visible buffer作为临时buffer**，**并且使用一个device local buffer作为实际的vertex buffer**。

```c++
	void createVertexBuffer()
	{
		VkDeviceSize bufferSize = sizeof(vertices[0]) * vertices.size();

		//创建staging buffer
		VkBuffer stagingBuffer;
		VkDeviceMemory stagingBufferMemory;
		createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_SRC_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, stagingBuffer, stagingBufferMemory);

		//将顶点数据拷贝到staging buffer中
		//首先将内存映射到CPU可以访问的内存中
		void* data;
		vkMapMemory(device, stagingBufferMemory, 0, bufferSize, 0, &data);
		//将内存中的顶点数据拷贝到staging buffer的内存中
		memcpy(data, vertices.data(), (size_t)bufferSize);
		//解除映射
		vkUnmapMemory(device, stagingBufferMemory);
		
		//创建实际的vertex buffer，注意是device local的
		createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_DST_BIT | VK_BUFFER_USAGE_VERTEX_BUFFER_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, vertexBuffer, vertexBufferMemory);
	}
```

现在我们使用一个新的**stagingBuffer**和**stagingBufferMemory**来映射和拷贝顶点数据。在这一章，我们会使用两个新的buffer usage flags：
- **VK_BUFFER_USAGE_TRANSFER_SRC_BIT**：Buffer可以在内存传输操作中用作源。
- **VK_BUFFER_USAGE_TRANSFER_DST_BIT**：Buffer可以在内存传输操作中用作目的地。

现在，**vertex buffer**从**device local**的内存类型被分配，它通常意味着我们不能对它使用vkMapMemory（即CPU无法访问）。但是，我们**可以从stagingBuffer将数据拷贝到vertexBuffer**。我们必须对staging buffer指示传输源flag、对vertex buffer指示传输目的地flag与vertex buffer usage flag。

我们现在来写一个函数来将一个buffer中的内容拷贝到另一个buffer，叫做**copyBuffer**。

```c++
	//将一个buffer的内容拷贝到另一个buffer
	void copyBuffer(VkBuffer srcBuffer, VkBuffer dstBuffer, VkDeviceSize size)
	{

	}
```

内存传输操作通过command buffers来执行，就好比绘制指令。因此，我们必须首先**分配一个临时的command buffer**。我们也许想为了这种短生命周期的command buffers创建一个单独的command pool，因为这样也许会优化内存分配。如果真的想这么做，我们需要在这个command pool创建时，使用**VK_COMMAND_POLL_CREATE_TRANSIENT_BIT**（Transient即短暂的）。但在教程中，我们还是简单起见，使用同一个command pool。

```c++

	//将一个buffer的内容拷贝到另一个buffer
	void copyBuffer(VkBuffer srcBuffer, VkBuffer dstBuffer, VkDeviceSize size)
	{
		//创建一个临时的command buffer用于内存传输操作
		VkCommandBufferAllocateInfo allocInfo{};
		allocInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO;
		allocInfo.level = VK_COMMAND_BUFFER_LEVEL_PRIMARY;
		allocInfo.commandPool = commandPool;
		allocInfo.commandBufferCount = 1;

		VkCommandBuffer commandBuffer;
		vkAllocateCommandBuffers(device, &allocInfo, &commandBuffer);
	}
```

然后立刻开始录制该command buffer。

```c++
		//立即开始录制
		VkCommandBufferBeginInfo beginInfo{};
		beginInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;
		beginInfo.flags = VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT;

		vkBeginCommandBuffer(commandBuffer, &beginInfo);
```

我们**只会使用一次该command buffer，并且在拷贝操作完成执行时从该函数返回**。因此，使用**VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT**告诉驱动我们的想法是一个好做法。

```c++
		//拷贝
		VkBufferCopy copyRegion{};
		copyRegion.srcOffset = 0;// Optional
		copyRegion.dstOffset = 0;// Optional
		copyRegion.size = size;
		vkCmdCopyBuffer(commandBuffer, srcBuffer, dstBuffer, 1, &copyRegion);
```

**buffer内容的传输是通过vkCmdCopyBuffer实现的**。它使用源buffer、目的地buffer、以及要拷贝的region数组作为参数。regions在VkBufferCopy结构体中被定义，其由源buffer偏移、目的地buffer偏移和size组成。不像vkMapMemory，这里**不能指定为VK_WHOLE_SIZE**。

```c++
		//结束录制
		vkEndCommandBuffer(commandBuffer);
```

该command buffer只包含了拷贝操作，所以我们在拷贝完之后结束录制。现在我们执行该command buffer来完成传输。

```c++
		//提交到queue来执行
		VkSubmitInfo submitInfo{};
		submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;
		submitInfo.commandBufferCount = 1;
		submitInfo.pCommandBuffers = &commandBuffer;

		vkQueueSubmit(graphicsQueue, 1, &submitInfo, VK_NULL_HANDLE);
		vkQueueWaitIdle(graphicsQueue);
```

不像绘制指令，这次我们**不需要先等待任何事件再执行**，我们希望立刻执行buffer传输操作。这里有两种方式来等待传输操作完成。我们可以**使用一个fence并且使用vkWaitForFences来等待**，或者简单地**使用vkQueueWaitIdle来等待transfer queue变成空闲**。使用一个fence允许我们同时安排多个传输，并且一次等待它们全部完成，而不是执行一次传输等待一次，这也许会给驱动更多空间来优化。（但在教程中，使用了第二个方法，牺牲了点性能）

```c++
		//释放command buffer
		vkFreeCommandBuffers(device, commandPool, 1, &commandBuffer);
```

不要忘了**销毁**这个用于传输操作的command buffer。

现在我们可以在createVertexBuffer中调用**copyBuffer**来将顶点数据拷贝到device local buffer。

```c++
		//创建实际的vertex buffer，注意是device local的
		createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_DST_BIT | VK_BUFFER_USAGE_VERTEX_BUFFER_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, vertexBuffer, vertexBufferMemory);

		//将顶点数据从staging buffer拷贝到device local的vertex buffer
		copyBuffer(stagingBuffer, vertexBuffer, bufferSize);
```

在从staging buffer拷贝数据到device buffer之后，我们需要销毁staging buffer。

```c++
		//将顶点数据从staging buffer拷贝到device local的vertex buffer
		copyBuffer(stagingBuffer, vertexBuffer, bufferSize);

		//销毁staging buffer及其VkDeviceMemory
		vkDestroyBuffer(device, stagingBuffer, nullptr);
		vkFreeMemory(device, stagingBufferMemory, nullptr);
```

运行程序来保证我们可以看到熟悉的三角形。这个优化现在也许不是很明显，但是**顶点数据现在是从高性能的内存中上传到GPU中用于绘制的**。在渲染更复杂的几何场景时该优化会变得更明显。

#### 14 小结 Conclusion

需要注意的是，在实际的程序中，我们**不应该为每个单独的buffer实际调用vkAllocateMemory**。**同一时间内存分配的最大数量被物理设备上的maxMemoryAllocateCount所限制**，也许是4096这样的很少的数量，或者在像NVIDIA GTX 1080上有更高的数量（大人，时代变啦~现在是4090的天下了）。同一时间为大量对象分配内存的正确方式是**创建一个自定义的allocator**，通过使用我们在许多函数中看到的**offset**参数，**使用单次分配给许多不同的对象分配内存**。

我们可以自己实现这样一个allocator，或者直接使用GPUOpen计划提供的**VulkanMemoryAllocator**库。但是，在该教程中，为每个资源使用单独的分配操作是ok的，因为我们不会接近分配次数限制。

#### 15 索引缓冲区 Index buffer

在实际的程序中，一个3D Mesh的渲染通常会在多个三角形中共享顶点。在绘制一个矩形时其实就已经发生了该现象。

![20240208115231](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240208115231.png)

绘制一个矩形需要两个三角形，意味着我们需要一个包含6个顶点的vertex buffer。问题是这两个三角形的顶点数据需要重复，导致50%的冗余。在更复杂mesh的情况下冗余会变得更严重，其中1个顶点在平均3个三角形中宏重复使用。该问题的解决方法是使用索引缓冲区**index buffer**（老朋友了噢~）。

**index buffefer本质上是指向veretx buffer的指针数组**。它允许我们**重新排序顶点数据**，并且重用多个顶点的现有数据。上图演示了如果我们有一个包含四个唯一顶点的vertex buffer，则矩形的index buffer会是什么样子。前三个索引定义了右上方的三角形，后三个索引 定义了左下方的三角形。

#### 16 索引缓冲区创建 Index buffer creation

在这一节，我们将会修改顶点数据，并且为绘制一个矩形增加索引数据。将顶点数据修改为如下，代表矩形的四个顶点。

```c++
	const std::vector<Vertex> vertices = {
		{{-0.5f, -0.5f}, {1.0f, 0.0f, 0.0f}},
		{{0.5f, -0.5f}, {0.0f, 1.0f, 0.0f}},
		{{0.5f, 0.5f}, {0.0f, 0.0f, 1.0f}},
		{{-0.5f, 0.5f}, {1.0f, 1.0f, 1.0f}}
	};
```

左上角是红色，右上角是绿色，右下角是蓝色，左下角是白色（注意**y轴正方向是朝下的**）。我们将增加一个新的数组**indices**来表示index buffer的内容。就和上图中的一样，我们需要将索引匹配来绘制右上三角形和左下三角形。

```c++
	const std::vector<uint16_t> indices = {
		0, 1, 2, 2, 3, 0
	};
```

**取决于vertices中的entries数量，可以为index buffer使用uint16_t或者uint32_t**。我们使用uint16_t。因为我们现在不会超过65535个单独顶点。

就像顶点数据一样，**索引需要被上传到一个VkBuffer中来给GPU访问**。定义两个类成员来管理index buffeer的资源。

```c++
	//Vertex buffer
	VkBuffer vertexBuffer;
	//实际分配的vertex buffer的内存（device local，可以认为是显存）
	VkDeviceMemory vertexBufferMemory;
	//Index buffer
	VkBuffer indexBuffer;
	//实际分配的index buffer的内存
	VkDeviceMemory indexBufferMemory;
```

createIndexBuffer函数几乎和createVertexBuffer一样。

```c++
	//initVulkan函数来用于实例化Vulkan objects私有成员
	void initVulkan()
	{
		...
		createVertexBuffer();
		createIndexBuffer();
		...
	}

	//创建index buffer
	void createIndexBuffer()
	{
		VkDeviceSize bufferSize = sizeof(indices[0]) * indices.size();

		//创建staging buffer
		VkBuffer stagingBuffer;
		VkDeviceMemory stagingBufferMemory;
		createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_SRC_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, stagingBuffer, stagingBufferMemory);

		//将顶点数据拷贝到staging buffer中
		//首先将内存映射到CPU可以访问的内存中
		void* data;
		vkMapMemory(device, stagingBufferMemory, 0, bufferSize, 0, &data);
		//将内存中的顶点数据拷贝到staging buffer的内存中
		memcpy(data, indices.data(), (size_t)bufferSize);
		//解除映射
		vkUnmapMemory(device, stagingBufferMemory);

		//创建实际的index buffer，注意是device local的
		createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_DST_BIT | VK_BUFFER_USAGE_INDEX_BUFFER_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, indexBuffer, indexBufferMemory);

		//将索引数据从staging buffer拷贝到device local的index buffer
		copyBuffer(stagingBuffer, indexBuffer, bufferSize);

		//销毁staging buffer和对应VkDeviceMemory
		vkDestroyBuffer(device, stagingBuffer, nullptr);
		vkFreeMemory(device, stagingBufferMemory, nullptr);
	}
```

只有两处明显的不同。bufferSize现在等于indices数量乘以index类型的大小，uint16_t或者uint32_t。indexBuffer的usage应该是VK_UBFFER_USAGE_INDEX_BUFFER_BIT而不是VK_BUFFER_USAGE_VERTEX_BUFFER_BIT。此外，都是相同的。我们创建一个staging buffer将indices中的内容拷贝到其中，然后最后拷贝到真正的device local的index buffer中。

```c++
	//一旦window被关闭，我们将在**cleanup**函数中确保释放我们用到的所有资源
	void cleanup()
	{
		cleanupSwapChain();

		//销毁index buffer
		vkDestroyBuffer(device, indexBuffer, nullptr);
		//销毁与index buffer绑定的内存
		vkFreeMemory(device, indexBufferMemory, nullptr);

		//销毁vertex buffer
		vkDestroyBuffer(device, vertexBuffer, nullptr);
		//销毁与vertex buffer绑定的内存
		vkFreeMemory(device, vertexBufferMemory, nullptr);
		...
	}
```

#### 17 使用索引缓冲区 Using an index buffer

使用index buffer来绘制需要在recordCommandBuffer函数中进行两处改动。我们**首先需要绑定index buffer**，就像我们对vertex buffer做的一样。不同之处在于我们只可以拥有一个单独的index buffer。不幸的是，我们**不可以对每个顶点属性使用不同的索引**，因此即使只有一个属性发生变化（如position），我们仍然需要完全复制顶点数据。

```c++
		//绑定vertex buffer
		VkBuffer vertexBuffers[] = { vertexBuffer };
		VkDeviceSize offsets[] = { 0 };
		vkCmdBindVertexBuffers(commandBuffer, 0, 1, vertexBuffers, offsets);

		//绑定index buffer
		vkCmdBindIndexBuffer(commandBuffer, indexBuffer, 0, VK_INDEX_TYPE_UINT16);
```

**index buffer的绑定通过vkCmdBindIndexBuffer实现**，其拥有index buffer、字节为单位的offset、index数据的类型作为参数。正如之前提到的，可能的配型是VK_INDEX_TYPE_UINT16和VK_INDEX_TYPE_UINT32。

只绑定index buffer不会改变任何事，我们还需要**改变绘制指令来告诉Vulkan使用index buffer**。移除vkCmdDraw，并使用**vkCmdDrawIndexed**代替。

```c++
		//绘制指令，使用index buffer
		vkCmdDrawIndexed(commandBuffer, static_cast<uint32_t>(indices.size()), 1, 0, 0, 0);
```

其调用和vkCmdDraw非常像。第二、第三个参数指定了索引的数量和instance的数量。我们不会使用instancing，所以指定1个instance。索引的数量代表了将会传送到顶点着色器中的顶点数量。下一个参数指定了index buffer中的offset，使用1将会造成显卡从第二个索引开始读取。倒数第二个参数指定了index buffer中对索引值增加的一个offset。最后一个参数指定了instancing的offset，我们不会用到。

现在运行程序，我们可以看到如下一个矩形。

![20240208142306](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240208142306.png)

现在我们知道了如何**通过index buffer来重用顶点来节省内存**。这在未来章节当我们加载复杂的3D模型会变得非常重要。

前几章已经提到了，我们应该使用一次单独的内存分配来给许多的资源比如buffers来分配内存，但是实际上应该更进一步。驱动开发者建议我们也**应该存储许多buffer，比如vertex和index buffer，到一个单独的VkBuffer中**，并且使用指令中的offset，比如vkCmdBindVertexBuffers。这样的好处是数据**对于cache更加友好**，因为它们物理上紧凑。**如果在相同渲染操作期间不同时使用多个资源，甚至可以将相同的内存块重用于多个资源**，前提是它们的数据被刷新。这称为**别名aliasing**，一些Vulkan函数具有显式的flags来指定我们要执行此操作。

#### 参考
1. 题图来自画师wlop
2. https://www.khronos.org/opengl/wiki/Layout_Qualifier_(GLSL)
