---
layout:     post
title:      "【VulkanTutorial学习笔记——17（完结篇）】Compute Shader"
subtitle:   ""
date:       2024-03-02 12:00:00
author:     "recaeee"
header-img: "img/in-post/default-bg.jpg"
tags:
    - Vulkan
---
# 【VulkanTutorial学习笔记——17（完结篇）】Compute Shader

![20240302110450](https://raw.githubusercontent.com/recaeee/PicGo/main/20240302110450.png)

#### 0 写在前面
本系列为个人在跟随Vulkan Tutorial学习过程中记录的一些笔记，一是防止自己看着看着弃坑，二是希望能在看过之后多少留些印象，三是想在啥时候忘记了能快速地回忆一些Vulkan知识点。

出于以上这些目的，该系列的学习笔记可能会写得较随意一些，并且对于主观判断不重要的东西可能会略过，因此也不保证该系列笔记的学习质量，可能有些东西不经过脑子就会写出来。

我是一个Vulkan的小白，对于Vulkan尚不熟悉，笔记中可能存在错误，或者概念不明确的地方，也希望看的人可以批评斧正~

在这一章，我们会学习最后一个内容Compute shaders。这一章的风格和前面所有章节都相差甚远，甚至原教程中的代码会让我们废除很多之前已经完成的代码，这无疑是让人沮丧的。因此我也花了些时间，对本章的代码和之前的代码做了兼容，确保两边都可以正常运行。

最后，这也是Vulkan Tutorial系列的最后一章啦~完结撒花！在这里也啰嗦些想说的话吧。

起初学习Vulkan契机是因为处理项目中的Vulkan渲染兼容性问题，但当时对Vulkan完全不熟悉的我看到那些问题只能感觉到无比的茫然，不知道要干什么，对它的毫不了解让我生活在Vulkan的恐惧之中，即使只是帮大佬打杂，但也会因为不熟悉Vulkan、对问题一知半解而自卑。因此在2023年9月30日开启了这个系列的学习，今天是2024年3月2日了，没想到一学就是5个月。一开始只是想记录一些笔记，但逐渐变成了对原教程的个人翻译版，我想我大概翻译了整个Vulkan Tutorial的95%以上的内容吧。虽然我知道之前已经有人翻译过，但自己翻译一遍可以让我的Vulkan基础更加扎实（不想再经历学习OpenGL学完之后几乎忘光了），另外也可以锻炼自己的英语能力。经过这5个月的学习，有一点是让我深刻感觉到变化的，就是我不会再对图形API的底层内容感到恐惧了，让我对自己更自信了（要不要脸了），我觉得自信这点还是很重要的，很多时候不自信会让人打退堂鼓，看到一些前沿技术就感觉自己一定学不会，就不去学了。当然，学习的过程是让自己感觉变得无知的过程，我变得更自信也说明我学的还不够深，哈哈哈哈。

另外，如果让我形容一下Vulkan的学习过程的话，我觉得是枯燥但值得。枯燥是因为图形API只是图形学的工具，并不像图形学中各种充满魔力的数学公式（渲染方程等）一样让人向往。但它也是值得的，因为我觉得它是学习渲染的必经之路，学习图形API可以感受到渲染在实践中的设计哲学，更能帮我们为渲染的底层实践去黑箱化（要想再深就是去学习GPU硬件咯）。

对于如何学习Vulkan，我觉得先去初步了解下Vulkan的各个概念，推荐[这位大佬的文章](https://zhuanlan.zhihu.com/p/616082929)，再跟随这个[Vulkan Tutorial系列](https://vulkan-tutorial.com/Introduction)完全足以，但推荐的学习Vulkan Tutorial的方法是以原英文教程为主（如果看英文感觉头疼，或者有些地方感觉疑惑的话，可以再看看我的翻译辅助），英文的一些内容翻译成中文味道总会变得怪怪的。

最后也附上我最终的代码仓库，在其中包含了整个系列的所有代码，并且我都添加了详细的中文注释（个人认为写代码注释非常重要），同时对最后一章的代码进行了改良，分离了前后内容，不会让读者因为最后一章代码和之前章节代码相差甚远而感觉沮丧。

这一章的成果也很符合完结撒花哈哈哈~Vulkan系列圆满落幕，我们下个系列再见！

![ComputeShadersShort](https://raw.githubusercontent.com/recaeee/PicGo/main/ComputeShadersShort.gif)

原教程链接：
https://vulkan-tutorial.com/Introduction

本章节链接：
https://vulkan-tutorial.com/Compute_Shader

包含系列全部内容的代码仓库（包含详细中文注释）:
https://github.com/recaeee/VulkanTutorialProject

#### 1 介绍 Introduction

在这一章中，我们将了解**计算着色器compute shaders**。之前的所有章节都涉及Vulkan pileline的传统图形部分。但是不像OpenGL这类老APIs，Vulkan中的compute shader支持是强制性的。这意味着我可以在每个可用的Vulkan实现上使用compute shaders，无论是高端桌面GPU还是低功耗嵌入式设备。

这开启了**图形处理器单元（GPGPU）实现通用计算**的世界，无论我们的应用在何处运行。GPGPU意味着我们**可以在GPU上做一般的数学计算**，这在传统上是CPU的领域。但是随着GPUs变得更加强大和灵活，许多需要CPU通用功能的工作负载现在都可以在GPU上实时完成。

GPU的计算能力能被使用到的例子包括图像处理、可见性测试、后处理、高级光照计算、动画、物理（粒子）等等。甚至，可以将计算用于不需要任何图形输出的非视觉计算工作，例如数学运算或AI相关的活。这就是所谓的"**headless compute**（中译还真是无头计算？？即无须显示器）"。

#### 2 优势 Advantages

在GPU上做一些计算量大的计算有许多优势。最显著的一个优势是**降低CPU的工作负载**。另一个优势是**不需要在CPU主存和GPU内存之间移动数据**。所有的数据可以驻留在GPU上，而不需要等待从主存过来的非常慢的传输工作。

除了以上这些，GPUs是**高度并行化**的，其中一些拥有数万个小型计算单元。这通常使它们比具有一些大型计算单元的CPU更适合高度并行的工作流程。

#### 3 Vulkan管线 The Vulkan pipeline

有一点很重要，**compute是完全与pipeline的graphics部分完全分开的**。这可以从官方规范中的以下Vulkan pipeline框图中看出。

![20240220195635](https://raw.githubusercontent.com/recaeee/PicGo/main/20240220195635.png)

在该图表中，我们可以看到pipeline的传统graphics部分位于左边，而在右边的许多Shader stages（红框）并不是graphics pipeline的一部分，包括compute shader（stage）。因为**compute shader stage与graphics pipeline分离**，所以我们**能够在任何我们认为合适的地方使用它**。这和fragment shader总是会被应用于vertex shader的变换输出是不同的。

该图表的中心也展示了例如descriptor sets也可以用于compute，因此我们学到的有关**descriptors layouts、descriptor sets和descriptors也适用于此处**。

#### 4 一个栗子 An example

一个易于我们理解的例子是，我们会在这一章实现一个**基于GPU的粒子系统particle system**。这样的粒子系统会在许多游戏中被使用到，并且经常由数千个需要以交互式帧率更新的粒子组成。渲染这样一个system需要2个主要部分：**顶点vertices**（作为vertex buffer传递）和**基于某些等式更新它们的方法**。

“经典”的**基于CPU的particle system**会在系统主存中存储particle数据，并且使用CPU来更新它们。在CPU更新之后，顶点需要再次被传递到GPU的内存（显存），以便在下一帧显式更新后的particle。最直接的方法是每一帧使用新的数据重建vertex buffer。这显然是非常耗的。根据我们的实现，有一些其他的选择，比如**映射GPU内存来让CPU可以直接写入GPU内存**（在桌面系统上被叫做"**resizable BAR**"，或者在集显上被称为**统一内存unified memory**），或者**使用一个host local buffer**（由于PCI-E带宽，这将是最慢的方法）。但是，无论我们选择哪种buffer更新方式，我们总会需要与CPU进行“**往返round-trip**”来更新particles。

而在**基于GPU的particle system**中，这种往返round-trip不再需要。Vertices只会在最初上传到GPU，**所有的更新都会通过compute shaders在GPU内存中运行**。这更快的一个主要原因是**GPU和它的local memory具有更高的带宽**。在基于CPU的场景下，我们会**受限于主存和PCI-express带宽**，它的数值通常是GPU内存的冰山一角。

在具有专门compute queue的GPU上执行此操作时，我们可以与graphics pipeline的渲染部分并行更新particles。这被称为"**异步计算async compute**"，是本教程未涵盖的高级主题。

以下是这一章代码的效果截图，这里显示的particles都在GPU上通过一个compute shader直接更新，没有任何CPU交互。

![20240220201927](https://raw.githubusercontent.com/recaeee/PicGo/main/20240220201927.png)

这里再附上我录的Gif。

![ComputeShadersShort](https://raw.githubusercontent.com/recaeee/PicGo/main/ComputeShadersShort.gif)

#### 5 数据处理 Data manipulation

在教程中，我们已经学习了不同的buffer类型，比如vertex和index buffers（用于传输图元）、uniform buffers（用于传递数据到shader），并且我们使用images来实现纹理映射。但是到目前为止，我们总是使用CPU写入数据，并且只在GPU上读取数据。

**compute shaders**中要引入的一个重要概念是，任意读取和**写入**buffers的能力。为此，Vulkan提供了两种专用存储类型。

##### 5.1 着色器存储缓冲区对象（SSBO） Shader storage buffer objects(SSBO)

**shader storage buffer(SSBO)允许shader读取和写入一个buffer**。它的使用方法和unform buffer objects类似。最大的不同点在于，我们**可以将其他buffer类型别名alias为SSBO**，并且它们可以任意大。

回到基于GPU的particle system，我们也许想知道如何使用compute shader更新（写入）vertices，并且让vertex shader读取来进行（绘制），因为这两种用法似乎需要不同的buffer类型。

但事实并非如此。在VUlkan中，我们**可以为buffers和images指定多种usages**。因此，为了让particle vertex buffer能够作为（graphics pass中的）**vertex buffer**，同时作为（compute pass）中的**storage buffer**，我们只需要创建同时具有这两种usage flags的buffer。

```c++
VkBufferCreateInfo bufferInfo{};
...
bufferInfo.usage = VK_BUFFER_USAGE_VERTEX_BUFFER_BIT | VK_BUFFER_USAGE_STORAGE_BUFFER_BIT | VK_BUFFER_USAGE_TRANSFER_DST_BIT;
...

if (vkCreateBuffer(device, &bufferInfo, nullptr, &shaderStorageBuffers[i]) != VK_SUCCESS) {
    throw std::runtime_error("failed to create vertex buffer!");
}
```

将这两个flags **VK_BUFFER_USAGE_VERTEX_BUFFER_BIT**和**VK_BUFFER_USAGE_STORAGE_BUFFER_BIT**设置到bufferInfo.usage会告诉Vulkan我们希望在两个不同的场景下使用该buffer：作为vertex shader中的vertex buffer、作为store buffer。注意，我们也增加了**VK_BUFFER_USAGE_TRANSFER_DST_BIT** flag，由此我们可以从host传输数据到GPU。这一点至关重要，因为我们希望shader storage buffer仅保留在GPU内存中（**VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT**），我们需要将数据从host传输到该buffer。

这是使用createBuffer helper function**创建shaderStorageBuffer**的代码。

```c++
createBuffer(bufferSize, VK_BUFFER_USAGE_STORAGE_BUFFER_BIT | VK_BUFFER_USAGE_VERTEX_BUFFER_BIT | VK_BUFFER_USAGE_TRANSFER_DST_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, shaderStorageBuffers[i], shaderStorageBuffersMemory[i]);
```

**GLSL** shader对这样一个buffer的声明如下。

```c++
struct Particle {
  vec2 position;
  vec2 velocity;
  vec4 color;
};

layout(std140, binding = 1) readonly buffer ParticleSSBOIn {
   Particle particlesIn[ ];
};

layout(std140, binding = 2) buffer ParticleSSBOOut {
   Particle particlesOut[ ];
};
```

在这个例子中，我们有一个**类型化的SSBO**，同时每个particle拥有一个position和velocity值（Particle结构体中所示）。该SSBO**使用[]来包含不受约束数量的particles**。与uniform buffers相比，不需要指定SSBO中的元素数量是其有点之一。**std140是一个内存布局限定符**，它确定shader storage buffer的成员元素如何在内存中对齐。这为我们提供了**在host和GPU之间映射buffers**所需的某些保证。

在compute shader中**写入**这样一个storage buffer object是非常简单的，与在C++端写入buffer的方式类似。

```c++
particlesOut[index].position = particlesIn[index].position + particlesIn[index].velocity.xy * ubo.deltaTime;
```

##### 5.2 存储图像 Storage images

注意我们在这一章不会实现图像处理。该段旨在让读者意识到**compute shader也可以用于图像处理**。

**storage image**允许我们读取和写入一个image。典型的用例是对纹理应用image effects，实现后处理（和前者非常相似）或生成mipmaps。

storage image的创建和image非常相似。

```c++
VkImageCreateInfo imageInfo {};
...
imageInfo.usage = VK_IMAGE_USAGE_SAMPLED_BIT | VK_IMAGE_USAGE_STORAGE_BIT;
...

if (vkCreateImage(device, &imageInfo, nullptr, &textureImage) != VK_SUCCESS) {
    throw std::runtime_error("failed to create image!");
}
```

将两个flags **VK_IMAGE_USAGE_SAMPLED_BIT**和**VK_IMAGE_USAGE_STORAGE_BIT**设置到imageInfo.usage告诉Vulkan我们想在两个场景下使用该image：在fragment shader中采样该image，在compute shader中作为storage image。

**GLSL shader对storage image的声明**和sampled images很类似，如下为fragment中声明storage image的例子。

```c++
layout (binding = 0, rgba8) uniform readonly image2D inputImage;
layout (binding = 1, rgba8) uniform writeonly image2D outputImage;
```

主要的区别是一些**additional attributes**，比如图像格式的rgba8、readonly和writeonly限定符、告诉implementation我们将只从input image读取并写入到output iamge。还有一点重要的是，我们需要**使用image2D类型来声明storage image**。

在compute shader中读取和写入storage image的方法是使用**imageLoad**和**imageStore**。

```c++
vec3 pixel = imageLoad(inputImage, ivec2(gl_GlobalInvocationID.xy)).rgb;
imageStore(outputImage, ivec2(gl_GlobalInvocationID.xy), pixel);
```

#### 6 计算队列族 Compute queue familiies

在physical device and queue families chapter，我们已经学习了queue families和如何选择一个graphics queue family。Compute使用的queue family properties flag标识位为**VK_QUEUE_COMPUTE_BIT**。所以，如果我们想要做些计算工作，我们需要从**支持compute的queue family**中获取一个queue。

注意，Vulkan需要一种支持graphics操作的implementation（implementation我在这里理解为硬件对于vulkan接口的实现），以**拥有至少一个同时支持graphics和compute操作的queue family**，但是implementation提供一个专门的compute queue是可能的。专门的compute queue（不包含graphics bit）意味着一个**异步的compute queue**。为了让该教程更友好，我们使用同时支持graphics和compute操作的queue。这也会使我们免于处理一些高级同步机制。

为了我们的compute用例，我们需要改变一些device创建的代码。

```c++
		//获取PhysicalDevice支持的所有Queue family
		uint32_t queueFamilyCount = 0;
		vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, nullptr);
		std::vector<VkQueueFamilyProperties> queueFamilies(queueFamilyCount);
		vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, queueFamilies.data());

		//我们需要支持Graphics Queue Family
		int i = 0;
		for (const auto& queueFamily : queueFamilies)
		{
			//判断physical device是否支持graphics family，并且该family同时支持compute
			if ((queueFamily.queueFlags & VK_QUEUE_GRAPHICS_BIT) && (queueFamily.queueFlags & VK_QUEUE_COMPUTE_BIT))
			{
				indices.graphicsAndComputeFamily = i;
			}
			//判断physical device是否支持present family
			VkBool32 presentSupport = false;
			vkGetPhysicalDeviceSurfaceSupportKHR(device, i, surface, &presentSupport);
			if (presentSupport)
			{
				//注意present family和graphics family实际上很有可能指向同一个family，但我们依然将它们视为单独的两个
				indices.presentFamily = i;
			}
			...
		}
```

现在选择queue family index的代码会尝试找到一个**同时支持graphics和compute的queue family**。

然后，我们可以在**createLogicalDevice**中从该queue family中获取一个compute queue。

```c++
	//存储Compute queue的Handle
	VkQueue computeQueue;

	...

	//compute queue也是同一个family
	vkGetDeviceQueue(device, indices.graphicsAndQueueFamily.value(), 0, &computeQueue);
```

#### 7 计算着色器阶段 The compute shader stage

在graphics samples中，我们使用不同的pipeline stages来加载shaders和访问descriptors。**Compute shaders**以类似的方法通过**VK_SHADER_STAGE_COMPUTE_BIT** pipeline被访问。所以加载一个compute shader和加载一个vertex shader是一样的，只是在不同的shader stage。我们会在下一个段落来讨论具体细节。Compute也会为descriptors和pipelines引入一个新的binding point类型，名为**VK_PIPELINE_BIND_POINT_COMPUTE**，我们稍后会使用它。

#### 8 加载计算着色器 Loading compute shaders

在我们的程序中加载compute shader和加载其他shader是一样的。实际的不同点只有，我们需要使用**VK_SHADER_STAGE_COMPUTE_BIT**。

```c++
		auto computeShaderCode = readFile("shaders/compute.spv");
			
		VkShaderModule computeShaderModule = createShaderModule(computeShaderCode);

		VkPipelineShaderStageCreateInfo computeShaderStageInfo{};
		computeShaderStageInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_SHADER_STAGE_CREATE_INFO;
		computeShaderStageInfo.stage = VK_SHADER_STAGE_COMPUTE_BIT;
		computeShaderStageInfo.module = computeShaderModule;
		computeShaderStageInfo.pName = "main";

		VkPipelineShaderStageCreateInfo shaderStages[] = { vertShaderStageInfo, fragShaderStageInfo, computeShaderStageInfo };
```

#### 9 准备着色器存储缓冲区 Preparing the shader storage buffers

早前，我们了解到可以使用shader storage buffers来传递任意数据到compute shaders。在该案例中，我们将会上传一个particles数组到GPU，由此我们可以在GPU内存中直接操作它们。

在frames in flight章节，我们讨论了在运行中的每一帧的复制的资源，由此可以让CPU和GPU忙碌。首先，我们**声明一个buffer对象以及支持它们的device memory的vectors**。

```c++
	//传递到Compute shader的storage buffers，存储particle数据
	std::vector<VkBuffer> shaderStorageBuffers;
	std::vector<VkDeviceMemory> shaderStorageBuffersMemory;
```

在（新创建的）**createShaderStorageBuffers**中，我们resize这些vectors来匹配运行帧的最大数量。

```c++
		shaderStorageBuffers.resize(MAX_FRAMES_IN_FLIGHTS);
		shaderStorageBuffersMemory.resize(MAX_FRAMES_IN_FLIGHTS);
```

完成此设置后，我们可以开始将初始particle信息移动到GPU。首先，我们**在host端初始化一个particles向量**。

```c++
struct Particle {
	glm::vec2 position;
	glm::vec2 velocity;
	glm::vec4 color;

	static VkVertexInputBindingDescription getBindingDescription()
	{
		VkVertexInputBindingDescription bindingDescription{};
		bindingDescription.binding = 0;
		bindingDescription.stride = sizeof(Particle);
		bindingDescription.inputRate = VK_VERTEX_INPUT_RATE_VERTEX;
		
		return bindingDescription;
	}

	static std::array<VkVertexInputAttributeDescription, 2> getAttributeDescriptions()
	{
		std::array<VkVertexInputAttributeDescription, 2> attributeDescriptions{};

		attributeDescriptions[0].binding = 0;
		attributeDescriptions[0].location = 0;
		attributeDescriptions[0].format = VK_FORMAT_R32G32_SFLOAT;
		attributeDescriptions[0].offset = offsetof(Particle, position);

		attributeDescriptions[1].binding = 0;
		attributeDescriptions[1].location = 1;
		attributeDescriptions[1].format = VK_FORMAT_R32G32B32A32_SFLOAT;
		attributeDescriptions[1].offset = offsetof(Particle, color);

		return attributeDescriptions;
	}
};

const uint32_t PARTICLE_COUNT = 8192;

	void createShaderStorageBuffers()
	{
		shaderStorageBuffers.resize(MAX_FRAMES_IN_FLIGHTS);
		shaderStorageBuffersMemory.resize(MAX_FRAMES_IN_FLIGHTS);

		// Initialize particles
		std::default_random_engine rndEngine((unsigned)time(nullptr));
		std::uniform_real_distribution<float> rndDist(0.0f, 1.0f);

		// Initial particle positions on a circle
		std::vector<Particle> particles(PARTICLE_COUNT);
		for (auto& particle : particles) {
			float r = 0.25f * sqrt(rndDist(rndEngine));
			float theta = rndDist(rndEngine) * 2 * 3.14159265358979323846;
			float x = r * cos(theta) * HEIGHT / WIDTH;
			float y = r * sin(theta);
			particle.position = glm::vec2(x, y);
			particle.velocity = glm::normalize(glm::vec2(x, y)) * 0.00025f;
			particle.color = glm::vec4(rndDist(rndEngine), rndDist(rndEngine), rndDist(rndEngine), 1.0f);
		}
	}
```

然后，我们在host memory中创建一个**staging buffer**来控制初始的particle属性（这里代码很熟悉了，和vertex buffer一样）。

```c++
		VkDeviceSize bufferSize = sizeof(Particle) * PARTICLE_COUNT;

		VkBuffer stagingBuffer;
		VkDeviceMemory stagingBufferMemory;
		createBuffer(bufferSize, VK_BUFFER_USAGE_TRANSFER_SRC_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, stagingBuffer, stagingBufferMemory);

		void* data;
		vkMapMemory(device, stagingBufferMemory, 0, bufferSize, 0, &data);
		memcpy(data, particles.data(), (size_t)bufferSize);
		vkUnmapMemory(device, stagingBufferMemory);
```

使用staging buffer作为源，创建每帧的shader storage buffers，然后**将particle properties从staging buffer中拷贝到每个shader storage buffer**。

```c++
		//copyBuffer到shader storage buffers
		for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHTS; i++)
		{
			createBuffer(bufferSize, VK_BUFFER_USAGE_STORAGE_BUFFER_BIT | VK_BUFFER_USAGE_VERTEX_BUFFER_BIT | VK_BUFFER_USAGE_TRANSFER_DST_BIT,
				VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, shaderStorageBuffers[i], shaderStorageBuffersMemory[i]);
			// Copy data from the staging buffer (host) to the shader storage buffer (GPU)
			copyBuffer(stagingBuffer, shaderStorageBuffers[i], bufferSize);
		}
```

#### 10 描述符 Descriptors

设置compute的描述符Descriptors和graphics几乎一样。唯一的区别是该descriptors需要拥有**VK_SHADER_STAGE_COMPUTE_BIT**设置来让它们在compute stage可以被访问。

```c++
	void createComputeDescriptorSetLayout()
	{
		std::array<VkDescriptorSetLayoutBinding, 3> layoutBindings{};
		layoutBindings[0].binding = 0;
		layoutBindings[0].descriptorCount = 1;
		layoutBindings[0].descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
		layoutBindings[0].pImmutableSamplers = nullptr;
		layoutBindings[0].stageFlags = VK_SHADER_STAGE_COMPUTE_BIT;
		...
	}
```

注意，我们可以在这里组合shader stages，如果我们希望descriptor在vertex和compute stage可以被访问，举例来说，对于uniform buffer，我们可以如下设置bits。

```c++
layoutBindings[0].stageFlags = VK_SHADER_STAGE_VERTEX_BIT | VK_SHADER_STAGE_COMPUTE_BIT;
```

以下是我们的例子中的descriptor设置，**layouts**（回顾下知识点，描述符布局descriptor layout指定了pipeline将访问的资源类型）如下。（我为了和上几章兼容，代码和原教程不太一样，在这里单独创建了一份新的compute前缀的资源）

```c++
	VkDescriptorSetLayout computeDescriptorSetLayout;

	void createComputeDescriptorSetLayout()
	{
		std::array<VkDescriptorSetLayoutBinding, 3> layoutBindings{};
		//compute的uniform buffer
		layoutBindings[0].binding = 0;
		layoutBindings[0].descriptorCount = 1;
		layoutBindings[0].descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
		layoutBindings[0].pImmutableSamplers = nullptr;
		layoutBindings[0].stageFlags = VK_SHADER_STAGE_COMPUTE_BIT;

		//上一帧storage shader buffer
		layoutBindings[1].binding = 1;
		layoutBindings[1].descriptorCount = 1;
		layoutBindings[1].descriptorType = VK_DESCRIPTOR_TYPE_STORAGE_BUFFER;
		layoutBindings[1].pImmutableSamplers = nullptr;
		layoutBindings[1].stageFlags = VK_SHADER_STAGE_COMPUTE_BIT;

		//当前帧storage shader buffer
		layoutBindings[2].binding = 2;
		layoutBindings[2].descriptorCount = 1;
		layoutBindings[2].descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
		layoutBindings[2].pImmutableSamplers = nullptr;
		layoutBindings[2].stageFlags = VK_SHADER_STAGE_COMPUTE_BIT;

		VkDescriptorSetLayoutCreateInfo layoutInfo{};
		layoutInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO;
		layoutInfo.bindingCount = 3;
		layoutInfo.pBindings = layoutBindings.data();

		if (vkCreateDescriptorSetLayout(device, &layoutInfo, nullptr, &computeDescriptorSetLayout) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to create compute descriptor set layout!");
		}
	}
```

观察该设置，我们也许想知道为什么我们有两个layout bindings给shader storage buffer objects，即使我们只渲染一个单独的particle system。这是因为**particle的position会在每一帧根据帧时间间隔delta time更新**。这意味着，**每一帧需要知道上一帧的particle positions**，由此它可以使用一个新的delta time来更新它们，并且将它们写入自己的SSBO中，示意图如下。

![20240221222343](https://raw.githubusercontent.com/recaeee/PicGo/main/20240221222343.png)

为此，compute shader需要访问上一帧和当前帧的SSBOs。这通过**在descriptor setup中将两者都传递给compute shader**来实现。在**createComputeDescriptorSets**函数（在这一章，很多函数的实现被省略，需要的同学可以查看[这一章源码](https://vulkan-tutorial.com/code/31_compute_shader.cpp)，总体来说和之前代码类似，不是很难的）中对应的两个buffers资源为**storageBufferInfoLastFrame**和**storageBufferInfoCurrentFrame**。

```c++
	void createComputeDescriptorSets()
	{
		std::vector<VkDescriptorSetLayout> layouts(MAX_FRAMES_IN_FLIGHTS, computeDescriptorSetLayout);
		VkDescriptorSetAllocateInfo allocInfo{};
		allocInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_ALLOCATE_INFO;
		allocInfo.descriptorPool = computeDescriptorPool;//使用同一个pool
		allocInfo.descriptorSetCount = static_cast<uint32_t>(MAX_FRAMES_IN_FLIGHTS);
		allocInfo.pSetLayouts = layouts.data();

		computeDescriptorSets.resize(MAX_FRAMES_IN_FLIGHTS);
		if (vkAllocateDescriptorSets(device, &allocInfo, computeDescriptorSets.data()) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to allocate descriptor sets!");
		}

		for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHTS; i++)
		{
			std::array<VkWriteDescriptorSet, 3> descriptorWrites{};

			//compute的uniform buffer
			VkDescriptorBufferInfo uniformBufferInfo{};
			uniformBufferInfo.buffer = uniformBuffers[i];
			uniformBufferInfo.offset = 0;
			uniformBufferInfo.range = sizeof(UniformBufferObjectForParticle);

			descriptorWrites[0].sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
			descriptorWrites[0].dstSet = computeDescriptorSets[i];
			descriptorWrites[0].dstBinding = 0;
			descriptorWrites[0].dstArrayElement = 0;
			descriptorWrites[0].descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
			descriptorWrites[0].descriptorCount = 1;
			descriptorWrites[0].pBufferInfo = &uniformBufferInfo;

			//上一帧的particle数据shader storage buffer
			VkDescriptorBufferInfo storageBufferInfoLastFrame{};
			storageBufferInfoLastFrame.buffer = shaderStorageBuffers[(i - 1) % MAX_FRAMES_IN_FLIGHTS];
			storageBufferInfoLastFrame.offset = 0;
			storageBufferInfoLastFrame.range = sizeof(Particle) * PARTICLE_COUNT;

			descriptorWrites[1].sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
			descriptorWrites[1].dstSet = computeDescriptorSets[i];
			descriptorWrites[1].dstBinding = 1;
			descriptorWrites[1].dstArrayElement = 0;
			descriptorWrites[1].descriptorType = VK_DESCRIPTOR_TYPE_STORAGE_BUFFER;
			descriptorWrites[1].descriptorCount = 1;
			descriptorWrites[1].pBufferInfo = &storageBufferInfoLastFrame;

			//当前帧的particle数据shader storage buffer
			VkDescriptorBufferInfo storageBufferInfoCurrentFrame{};
			storageBufferInfoCurrentFrame.buffer = shaderStorageBuffers[i];
			storageBufferInfoCurrentFrame.offset = 0;
			storageBufferInfoCurrentFrame.range = sizeof(Particle) * PARTICLE_COUNT;

			descriptorWrites[2].sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
			descriptorWrites[2].dstSet = computeDescriptorSets[i];
			descriptorWrites[2].dstBinding = 2;
			descriptorWrites[2].dstArrayElement = 0;
			descriptorWrites[2].descriptorType = VK_DESCRIPTOR_TYPE_STORAGE_BUFFER;
			descriptorWrites[2].descriptorCount = 1;
			descriptorWrites[2].pBufferInfo = &storageBufferInfoCurrentFrame;

			vkUpdateDescriptorSets(device, 3, descriptorWrites.data(), 0, nullptr);
		}
	}
```

记住，我们也需要从我们的**descriptor pool**中为SSBOs请求descriptor types。

```c++
	void createComputeDescriptorPool()
	{
		std::array<VkDescriptorPoolSize, 3> poolSizes{};

		//uniform buffer
		poolSizes[0].type = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;//需要包含哪些descriptor类型
		poolSizes[0].descriptorCount = static_cast<uint32_t>(MAX_FRAMES_IN_FLIGHTS);//数量
		//combined image sampler
		poolSizes[1].type = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
		poolSizes[1].descriptorCount = static_cast<uint32_t>(MAX_FRAMES_IN_FLIGHTS);
		//storage buffers
		poolSizes[2].type = VK_DESCRIPTOR_TYPE_STORAGE_BUFFER;
		poolSizes[2].descriptorCount = static_cast<uint32_t>(MAX_FRAMES_IN_FLIGHTS) * 2;

		VkDescriptorPoolCreateInfo poolInfo{};
		poolInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_POOL_CREATE_INFO;
		poolInfo.poolSizeCount = static_cast<uint32_t>(poolSizes.size());
		poolInfo.pPoolSizes = poolSizes.data();

		poolInfo.maxSets = static_cast<uint32_t>(MAX_FRAMES_IN_FLIGHTS);//可能被分配的descriptor sets的最大数量

		if (vkCreateDescriptorPool(device, &poolInfo, nullptr, &computeDescriptorPool) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to create descriptor pool!");
		}
	}
```

我们需要让**VK_DESCRIPTOR_TYPE_STORAGE_BUFFER类型的descriptor申请最大数量**翻倍，因为我们的sets会引用当前帧和上一帧的SSBOs。

#### 11 计算管线 Compute pipelines

因为compute不是graphics pipeline的一部分，所以我们不能使用vkCreateGraphicsPipelines。为运行计算指令我们需要通过**vkCreateComputePipelines**创建专门的**compute pipeline**。因为compute pipeline不会涉足任何光栅化状态，所以比起graphics pipeline，它的state数量很少。

```c++
		//创建pipeline
		VkComputePipelineCreateInfo pipelineInfo{};
		pipelineInfo.sType = VK_STRUCTURE_TYPE_COMPUTE_PIPELINE_CREATE_INFO;
		//shader stages
		pipelineInfo.layout = computePipelineLayout;;
		pipelineInfo.stage = computeShaderStageInfo;
		

		if (vkCreateComputePipelines(device, VK_NULL_HANDLE, 1, &pipelineInfo, nullptr, &computePipeline) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to create compute pipeline!");
		}
```

设置的过程很简单，我们只需要**一个shader stage**和**一个pipeline layout**。pipeline layout的工作方式与graphics pipeline相同。

```c++
		//创建pipeline layout
		VkPipelineLayoutCreateInfo pipelineLayoutInfo{};
		pipelineLayoutInfo.sType = VK_STRUCTURE_TYPE_PIPELINE_LAYOUT_CREATE_INFO;
		pipelineLayoutInfo.setLayoutCount = 1; // Optional
		pipelineLayoutInfo.pSetLayouts = &computeDescriptorSetLayout; // Optional

		if (vkCreatePipelineLayout(device, &pipelineLayoutInfo, nullptr, &computePipelineLayout) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to create pipeline layout!");
		}
```

#### 12 计算空间 Compute space

在我们开始了解compute shader如何工作以及如何将compute workloads提交到GPU之前，我们需要先谈谈两个重要的compute概念：**工作组work groups**和**调用invocations**。它们定义了一个抽象的执行模型，用于说明**GPU的计算硬件如何在三个维度（x，y和z）上处理compute workloads**。

**工作组work groups定义了GPU的计算硬件如何形成和处理compute workloads**。我们可以将它们视为GPU必须完成的工作项。**Work group dimensions**被应用程序通过command buffer中使用**dispatch command**来设置。

**每个work group都是执行相同compute shader的invocations的集合**。Invocations可能**并行运行**，并且**它们的dimensions在compute shader中设置**。**在单个work group中的所有invocations可以访问共享的内存**。

下图展示了在三个维度下work groups和invocations的关系。

![20240223232833](https://raw.githubusercontent.com/recaeee/PicGo/main/20240223232833.png)

**work groups的dimension数量（由vkCmdDispatch定义）和invocations的dimension数量取决于输入数据的结构（由compute shader中的local sizes定义）**。举例来说，如果我们要处理一维数组，那么我们只需要对两者指定x dimension。

在该例中，如果我们dispatch一个[64, 1, 1]的work group，并且compute shader的local size为[32, 32, 1]，那么我们的compute shader会被调用64x32x32=65536次。

注意，**对于不同的（硬件）实现，workgroups和local sizes的最大数量会有所不同**，所以我们需要总是在**VkPhysicalDeviceLimits**中查询compute相关的**maxComputeWorkGroupCount**、**maxComputeWorkGroupInvocations**和**maxComputeWorkGroupSize**限制。

#### 13 计算着色器 Compute shaders

现在，我们已经了解了设置一个compute shader pipeline的所有必要内容，是时候来看看compute shaders了。我们学习的有关GLSL shaders（vertex和fragmnet）的所有内容都适用于compute shaders。语法是一样的，以及许多概念比如程序和shader之间的数据传递是相同的。但同时也有一些重要的不同点。

一个非常基础的用于更新particle线性数组的compute shader的代码大致如下。

```c++
#version 450

layout (binding = 0) uniform ParameterUBO {
    float deltaTime;
} ubo;

struct Particle {
    vec2 position;
    vec2 velocity;
    vec4 color;
};

layout(std140, binding = 1) readonly buffer ParticleSSBOIn {
   Particle particlesIn[ ];
};

layout(std140, binding = 2) buffer ParticleSSBOOut {
   Particle particlesOut[ ];
};

layout (local_size_x = 256, local_size_y = 1, local_size_z = 1) in;

void main() 
{
    uint index = gl_GlobalInvocationID.x;  

    Particle particleIn = particlesIn[index];

    particlesOut[index].position = particleIn.position + particleIn.velocity.xy * ubo.deltaTime;
    particlesOut[index].velocity = particleIn.velocity;
    ...
}
```

shader的开头部分包含了shader input的声明。首先是一个uniform buffer object位于**binding 0**（shader binding对应descriptor的binding index），我们在该教程中已经学习过它。
在它的下方，我们声明了Particle结构，匹配C++端的声明。**Binding 1**指向shader storage buffer object，对应上一帧的particle data。**Binding 2**指向当前帧的SSBO，对应我们将要通过shader更新的数据。

有趣的东西是**compute独有的声明**，和compute space相关。

```c++
layout (local_size_x = 256, local_size_y = 1, local_size_z = 1) in;
```

它定义了**在当前work group中该compute shader的invocations数量**。正如之前提到的，这是compute space的local部分，因此拥有**local_前缀**。因为我们处理线性一维数组的particles，我们只需要在**local_size_x**中指定x dimension上的值。

在main函数中，会读取上一帧的SSBO，并且将更新后的particle positions写入当前帧的SSBO。和其他shader类型相似，**compute shaders拥有其独有的一组内置input变量**。**内置变量几乎总是以gl_作为前缀**。一个内置变量的例子是**gl_GlobalInvocationID**，它是一个**唯一标识当前dispatch中当前compute shader invocation的变量**。我们使用它来索引particles数组。

#### 14 运行compute指令

##### 14.1 Dispatch

现在是时候来告诉GPU真正来做一些计算了。这是通过在command buffer中调用**vkCmdDispatch**来完成的。虽然不完全正确，**compute的一个dispatch就好比graphics的一个draw call**（比如vkCmdDraw）。**该指令会dispatch最大三个dimensions下的给定数量的compute work items**。

```c++
	void recordComputeCommandBuffer(VkCommandBuffer commandBuffer, uint32_t imageIndex)
	{
		//录制绘制指令
		VkCommandBufferBeginInfo beginInfo{};
		beginInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;
		beginInfo.flags = 0; // Optional
		beginInfo.pInheritanceInfo = nullptr; // Optional

		if (vkBeginCommandBuffer(commandBuffer, &beginInfo) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to begin recording compute command buffer!");
		}

		//Compute不需要绑定render pass
		//绑定compute pipeline
		vkCmdBindPipeline(commandBuffer, VK_PIPELINE_BIND_POINT_COMPUTE, computePipeline);

		//为每一帧的正确的descriptor set实际绑定到着色器中的descriptor，好绕，实际就是为每一帧绑定正确的descriptor set
		vkCmdBindDescriptorSets(commandBuffer, VK_PIPELINE_BIND_POINT_COMPUTE, computePipelineLayout, 0, 1, &computeDescriptorSets[currentFrame], 0, nullptr);

		//dispatch，发送compute指令
		vkCmdDispatch(commandBuffer, PARTICLE_COUNT / 256, 1, 1);

		//结束command buffer录制
		if (vkEndCommandBuffer(commandBuffer) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to record compute command buffer!");
		}
	}
```

**vkCmdDispatch**会dispatch **PARTICLE_COUNT / 256**个**local work groups**在x dimension 。因为我们的particles数组是现行的，所以我们将其他两个dimensions置为1，形成了一维的dispatch。但是为什么我们要将particles的数量除以256呢？这是因为在之前的段落，我们**定义了在一个work group中每个compute shader会进行256次invocations**。所以如果我们有4096个particles，我们会dispatch 16次work groups，每个work group会运行256次compute shader invocations（每一次invocation执行一次main函数，更新一个particle）。为了让这两个数字（**要dispatch的work groups数量和compute shader中的invocations数量**）正确，往往需要花费一些思考和测试，取决于我们的workload和运行的硬件。如果我们的particle数量是动态变化的，并且不能总是被256（举例来说）整除，我们可以总是**在compute shader的开头使用gl_GlobalInvocationID做个判断，如果其索引值大于particles的数量，则直接从main函数返回**。

正如上面看到的compute pipeline绑定过程，compute command buffer包含的**state**比graphics command buffer少得多。**不需要启动render pass或者设置viewport**。

##### 14.2 提交工作 Submitting work

因为在我们的案例中同时进行了compute和graphics操作，我们会在（drawFrame函数中）**每一帧提交graphics queue和compute queue两者**。

```c++
...
if (vkQueueSubmit(computeQueue, 1, &submitInfo, nullptr) != VK_SUCCESS) {
    throw std::runtime_error("failed to submit compute command buffer!");
};
...
if (vkQueueSubmit(graphicsQueue, 1, &submitInfo, inFlightFences[currentFrame]) != VK_SUCCESS) {
    throw std::runtime_error("failed to submit draw command buffer!");
}
```

第一次对compute queue的submit使用了compute shader**对particle positions进行更新**，第二次submit将会使用更新后的数据**绘制particle system**。

##### 14.3 同步图形和计算 Synchronizing graphics and compute

**同步**是VUlkan中很重要的一个话题，在与graphics结合进行compute时更是如此。错误或者缺少同步可能会造成**在verte stage开始绘制（=读取）particles的时候compute shader还没有完成更新（=写入）particles的数据**（read-after-write hazard），或者**compute shader开始更新particles，但particles仍然在被pipeline的vertex部分使用**（write-after-read hazard）。

所以我们必须通过恰到地同步graphics和compute负载保证这些情况不会发生。取决于我们如何提交compute workload，有许多不同的方法来实现同步，但是在我们的例子中，我们使用了两个单独的submits，所以我们会**使用semaphores和fences来确保vertex shader在compute shader更新完particles前不会开始访问顶点数据**。

即使在C++代码中我们按顺序提交了这两次submits，但同步还是必须的，因为不保证它们会在GPU中按该顺序执行。增加wait和signal semaphores可以确保此执行顺序。

所以我们首先**在createSyncObjects中为compute工作添加一组新的同步原语**。compute fences和graphics fences类似，创建时为signaled状态，否则第一帧会因为等待该fence变为signaled而堵塞。

```c++
std::vector<VkFence> computeInFlightFences;
std::vector<VkSemaphore> computeFinishedSemaphores;
...
computeInFlightFences.resize(MAX_FRAMES_IN_FLIGHT);
computeFinishedSemaphores.resize(MAX_FRAMES_IN_FLIGHT);

VkSemaphoreCreateInfo semaphoreInfo{};
semaphoreInfo.sType = VK_STRUCTURE_TYPE_SEMAPHORE_CREATE_INFO;

VkFenceCreateInfo fenceInfo{};
fenceInfo.sType = VK_STRUCTURE_TYPE_FENCE_CREATE_INFO;
fenceInfo.flags = VK_FENCE_CREATE_SIGNALED_BIT;

for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHT; i++) {
    ...
    if (vkCreateSemaphore(device, &semaphoreInfo, nullptr, &computeFinishedSemaphores[i]) != VK_SUCCESS ||
        vkCreateFence(device, &fenceInfo, nullptr, &computeInFlightFences[i]) != VK_SUCCESS) {
        throw std::runtime_error("failed to create compute synchronization objects for a frame!");
    }
}
```

然后，我们使用这些同步原语来同步compute buffer的提交和graphics的提交。

```c++
	void drawFrameForParticles()
	{
		//compute和graphics可以共用一个submitInfo
		VkSubmitInfo submitInfo{};
		submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;

		//等待上一次compute完成
		vkWaitForFences(device, 1, &computeInFlightFences[currentFrame], VK_TRUE, UINT64_MAX);
		//更新compute shader的uniform buffer
		updateComputeUniformBuffer(currentFrame);
		//在确保我们会执行submit之后再重置fence到unsignaled
		vkResetFences(device, 1, &computeInFlightFences[currentFrame]);
		//重置compute command buffer，确保其可以被录制
		vkResetCommandBuffer(computeCommandBuffers[currentFrame], 0);
		//录制compute command buffer
		recordComputeCommandBuffer(computeCommandBuffers[currentFrame], currentFrame);

		submitInfo.commandBufferCount = 1;
		submitInfo.pCommandBuffers = &computeCommandBuffers[currentFrame];
		submitInfo.signalSemaphoreCount = 1;
		submitInfo.pSignalSemaphores = &computeFinishedSemaphores[currentFrame];

		if (vkQueueSubmit(computeQueue, 1, &submitInfo, computeInFlightFences[currentFrame]) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to submit compute command buffer!");
		}

		//接下来是graphics部分
		//等待上一帧完成
		vkWaitForFences(device, 1, &inFlightFences[currentFrame], VK_TRUE, UINT64_MAX);

		...
		//提交command buffer到queue中，并且配置同步
		//配置同步，注意这里要让STAGE_VERTEX_INPUT等待compute计算完成
		VkSemaphore waitSemaphores[] = { computeFinishedSemaphores[currentFrame], imageAvailableSemaphores[currentFrame] };
		VkPipelineStageFlags waitStages[] = { VK_PIPELINE_STAGE_VERTEX_INPUT_BIT, VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT };
		//重置submitInfo
		submitInfo = {};
		submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;

		submitInfo.waitSemaphoreCount = 2;
		submitInfo.pWaitSemaphores = waitSemaphores;
		submitInfo.pWaitDstStageMask = waitStages;
		...
		//提交Command buffer
		if (vkQueueSubmit(graphicsQueue, 1, &submitInfo, inFlightFences[currentFrame]) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to submit draw command buffer!");
		}

		...
	}
```

和semaphores章节的样例很像，在该设置中会立即运行compute shader因为我们没有指定任何要等待的semaphores（Fence除外）。这是正确的，因为我们**使用vkWaitForFences等待上一帧的compute command buffer完成执行后再提交新一轮的compute command buffer**。

在另一边，graphics的提交需要等待compute工作完成，所以当compute buffer还在更新particles的vertices数据时graphics并不会开始访问vertices。所以我们**等待当前帧的computeFinishedSemaphores**，让graphics提交在**VK_PIPELINE_STAGE_VERTEX_INPUT_BIT**阶段等待该semaphore，因为在该阶段会获取vertices。

但我们也需要等待presentation，所以**fragment shader在image被present之前不会输出到color attachments**。所以我们也会在**VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT**阶段等待当前帧的imageAvailableSemaphores。

#### 15 绘制粒子系统 Drawing the particle system

早前，我们学习了在Vulkan中，**buffers可以有多种使用场景**，所以我们创建用来包含particles的shader storage buffer同时启用shader storage buffer bit和vertex buffer bit。这意味着我们可以使用该shader storage buffer来绘制，正如我们之前章节用的“纯净”的vertex buffers。

首先我们**设置vertex input state来匹配particle结构体**。

```c++
struct Particle {
	...

	static std::array<VkVertexInputAttributeDescription, 2> getAttributeDescriptions()
	{
		std::array<VkVertexInputAttributeDescription, 2> attributeDescriptions{};

		attributeDescriptions[0].binding = 0;
		attributeDescriptions[0].location = 0;
		attributeDescriptions[0].format = VK_FORMAT_R32G32_SFLOAT;
		attributeDescriptions[0].offset = offsetof(Particle, position);

		attributeDescriptions[1].binding = 0;
		attributeDescriptions[1].location = 1;
		attributeDescriptions[1].format = VK_FORMAT_R32G32B32A32_SFLOAT;
		attributeDescriptions[1].offset = offsetof(Particle, color);

		return attributeDescriptions;
	}
};
```

注意，我们**没有在vertex input attributes中增加velocity**，因为它只用于compute shader。然后我们绑定并且绘制它们，就和所有vertex buffer一样。

最后，虽然省略了很多代码细节（细节可以看原教程代码或者我的仓库），但是我们成功实现Particle System啦！附上效果图~

![ComputeShadersShort](https://raw.githubusercontent.com/recaeee/PicGo/main/ComputeShadersShort.gif)

#### 16 完结撒花！ Conclusion

在这一章，我们学习了如何使用compute shaders将工作从CPU卸载到GPU。如果没有compute shaders，现代游戏和程序中的许多效果可能不可行或者运行地很慢。但除了graphics之外，compute还有很多用例，本章仅让我们了解了其可能性。所以，现在我们学会了如何使用compute shaders，我们可能会想关注一些更高级的compute主题，比如

1. Shared memory
2. [Asynchronous compute](https://github.com/KhronosGroup/Vulkan-Samples/tree/main/samples/performance/async_compute)
3. Atomic operations
4. [Subgroups](https://www.khronos.org/blog/vulkan-subgroup-tutorial)

我们可以在[Khronos Vulkan Samples repository](https://github.com/KhronosGroup/Vulkan-Samples/tree/master/samples/api)中找到更高级的compute samples。

完结撒花！Vulkan之路漫漫不止，渲染之路漫漫不止。5个月间的Vulkan学习之旅无疑是折磨、痛苦、无聊的，但收获也是满满的。之后打算做一些更有意思的东西，下个系列，我们再见！！

#### 参考
1. 题图来自画师wlop
2. https://zh.wikipedia.org/zh-cn/%E6%97%A0%E5%A4%B4%E8%AE%A1%E7%AE%97%E6%9C%BA
3. https://zhuanlan.zhihu.com/p/616082929






