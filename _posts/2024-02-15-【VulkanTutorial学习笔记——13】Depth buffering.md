---
layout:     post
title:      "【VulkanTutorial学习笔记——13】Depth buffering"
subtitle:   ""
date:       2024-02-15 12:00:00
author:     "recaeee"
header-img: "img/in-post/default-bg.jpg"
tags:
    - Vulkan
---
# 【VulkanTutorial学习笔记——13】Depth buffering

![20240215154631](https://raw.githubusercontent.com/recaeee/PicGo/main/20240215154631.png)

#### 0 写在前面
本系列为个人在跟随Vulkan Tutorial学习过程中记录的一些笔记，一是防止自己看着看着弃坑，二是希望能在看过之后多少留些印象，三是想在啥时候忘记了能快速地回忆一些Vulkan知识点。

出于以上这些目的，该系列的学习笔记可能会写得较随意一些，并且对于主观判断不重要的东西可能会略过，因此也不保证该系列笔记的学习质量，可能有些东西不经过脑子就会写出来。

我是一个Vulkan的小白，对于Vulkan尚不熟悉，笔记中可能存在错误，或者概念不明确的地方，也希望看的人可以批评斧正~

这一章，我们会实现Depth buffer，在Unity中通常对应CameraDepthAttachment。Depth buffer在实践中很重要，我们也经常会和深度图打交道，无论是深度测试还是想要实现一些特殊的渲染效果等等。幸运的是，在我们目前的程序中，实现Depth buffer并没有很复杂，得益于之前写的框架与函数，我们可以很方便且出色地实现Depth buffer，难度系数骤然降低~那么就开冲！

原教程链接：
https://vulkan-tutorial.com/Introduction
本章节链接：
https://vulkan-tutorial.com/Depth_buffering

#### 1 深度缓冲 Depth buffering

目前虽然我们使用的几何体都在3D空间中被投影，但我们没有关心过3D空间的前后关系，传入的Mesh顶点position也只有x、y坐标。在这一章，我们将为Mesh的position增加Z坐标，为3D网格做准备。首先，我们将会使用Z轴的坐标，在当前的正方形上放置另一个正方形，由此，我们可以看到**目前几何体由于未按照深度排序时出现的问题**。

#### 2 3D几何 3D geometry

首先，改变Vertex结构体，对position使用一个3D向量，并且更新相关的**VkVertexInputAttributeDescription**中的**format**。

```c++
	//顶点着色器输入
	struct Vertex {
		glm::vec3 pos;
		glm::vec3 color;
		glm::vec2 texCoord;
	};

			static std::array<VkVertexInputAttributeDescription, 3> getAttributeDescriptions()
		{
			//属性描述Attribute description结构体描述了如何从源自绑定描述Binding descrription的顶点数据块中提取顶点属性
			std::array<VkVertexInputAttributeDescription, 3> attributeDescriptions{};

			attributeDescriptions[0].binding = 0;
			attributeDescriptions[0].location = 0;
			attributeDescriptions[0].format = VK_FORMAT_R32G32B32_SFLOAT;
			attributeDescriptions[0].offset = offsetof(Vertex, pos);

			...
		}
```

接下来，**更新vertex shader**来接受3D坐标作为输入并且进行变换。不要忘记重新编译shader~

```c++
layout(location = 0) in vec3 inPosition;

...

void main()
{
	gl_Position = ubo.proj * ubo.view * ubo.model * vec4(inPosition, 1.0);
	fragColor = inColor;
	fragTexCoord = inTexCoord;
}
```

最后，**更新vertices**容器来增加Z坐标。

```c++
	const std::vector<Vertex> vertices = {
		{{-0.5f, -0.5f, 0.0f}, {1.0f, 0.0f, 0.0f}, {1.0f, 0.0f}},
		{{0.5f, -0.5f, 0.0f}, {0.0f, 1.0f, 0.0f}, {0.0f, 0.0f}},
		{{0.5f, 0.5f, 0.0f}, {0.0f, 0.0f, 1.0f}, {0.0f, 1.0f}},
		{{-0.5f, 0.5f, 0.0f}, {1.0f, 1.0f, 1.0f}, {1.0f, 1.0f}}
	};
```

如果现在运行程序，那么我们可以看到和之前一模一样的渲染结果。接下来，让我们在场景中增加一些额外的几何体，然后来演示我们开头说到的那个问题（几何体未按深度排序）。复制vertices来定义一个新的正方形，让它位于旧的正方形的下方，如下图所示。

![20240214200028](https://raw.githubusercontent.com/recaeee/PicGo/main/20240214200028.png)

**z坐标使用-0.5f**，并且为新的正方形增加正确的**indices**。

```c++
	const std::vector<Vertex> vertices = {
		{{-0.5f, -0.5f, 0.0f}, {1.0f, 0.0f, 0.0f}, {1.0f, 0.0f}},
		{{0.5f, -0.5f, 0.0f}, {0.0f, 1.0f, 0.0f}, {0.0f, 0.0f}},
		{{0.5f, 0.5f, 0.0f}, {0.0f, 0.0f, 1.0f}, {0.0f, 1.0f}},
		{{-0.5f, 0.5f, 0.0f}, {1.0f, 1.0f, 1.0f}, {1.0f, 1.0f}},
		//下面的正方形
		{{-0.5f, -0.5f, -0.5f}, {1.0f, 0.0f, 0.0f}, {1.0f, 0.0f}},
		{{0.5f, -0.5f, -0.5f}, {0.0f, 1.0f, 0.0f}, {0.0f, 0.0f}},
		{{0.5f, 0.5f, -0.5f}, {0.0f, 0.0f, 1.0f}, {0.0f, 1.0f}},
		{{-0.5f, 0.5f, -0.5f}, {1.0f, 1.0f, 1.0f}, {1.0f, 1.0f}}
	};

	const std::vector<uint16_t> indices = {
		0, 1, 2, 2, 3, 0,
		//下面的正方形
		4, 5, 6, 6, 7, 4
	};
```

现在运行程序，我们会看啊都类似Escher插图（视错觉艺术家Escher，著名画作是《瀑布》，很多人应该都看到过）的东西。

![20240214200321](https://raw.githubusercontent.com/recaeee/PicGo/main/20240214200321.png)

我们遇到的问题是，下面的正方形的片元覆盖在了上面的正方形上方进行了绘制，仅仅是因为**它位于index数组中的后面**。有两种方法来解决该问题：

1. 使用从后往前的深度对所有Draw Calls进行排序（即画家算法）
2. 使用**depth buffer**进行深度测试。

前一种方法通常用来绘制透明物体，因为与顺序无关的透明度是一个很难解决的问题。但是（第一种方法overdraw会很严重，并且还有复杂mesh交叠的问题），按深度对片元排序的问题更通常会使用**深度缓冲depth buffer**来解决。

**Depth buffer是一个额外的attachment用来存储每个像素上的深度**，好比color attachment用来存储每个像素上的颜色。每次光栅化器生产了一个新片元，深度测试就会检查新片元是否比前一个（同一个坐标下的）片元离（摄像机）更近（原文写的不太清楚哈，但depth buffer相信大火很熟悉了）。如果不是，新片元会被丢弃。如果一个片元通过了深度测试，它就会将它自身的深度值写入depth buffer。可以从fragment shader中操作写入的深度值，就像操作颜色输出一样。

```c++
//数学库，包含vector和matrix等，用于声明顶点数据、3D MVP变换等
#define GLM_FORCE_RADIANS
//使GLM的类型满足内存对齐
#define GLM_FORCE_DEFAULT_ALIGNED_GENTYPES
//使用Vulkan的深度范围0.0到1.0
#define GLM_FORCE_DEPTH_ZERO_TO_ONE
#include <glm/glm.hpp>
#include <glm/gtc/matrix_transform.hpp>
```

GLM生成的透视投影矩阵会默认使用OpenGL的深度范围-1.0到1.0。我们需要通过定义**GLM_FORCE_DEPTH_ZERO_TO_ONE**配置它，使用Vulkan的**深度范围0.0到1.0**。

#### 3 深度图像和视图 Depth image and view

Depth attachment是基于一个image运作的，就像color attachment一样。不同之处在于，swap chain不会为我们自动创建depth images。我们**只需要一张单独的depth image**，因为在同一时间只会有一个绘制操作在执行。Depth image将再次需要三个资源：**image、memory和image view**。（这么说来，才发现color attachment海比较特殊，我们并不会存储color attachment的VkDeviceMemory，猜测是因为swap chain自动为我们做了内存分配）

```c++
	//Depth buffer相关
	VkImage depthImage;
	VkDeviceMemory depthImageMemory;
	VkImageView depthImageView;
```

创建一个新函数**createDepthResources**来创建这些资源。

```c++
	//initVulkan函数来用于实例化Vulkan objects私有成员
	void initVulkan()
	{
		...
		createCommandPool();
		createDepthResources();
		createTextureImage();
		...
	}

	void createDepthResources()
	{

	}
```

创建一个depth image非常简单。它应该和color attachment具有相同的（由swap chain拓展定义）分辨路，并且具有适合depth attachment的image usage、optimal tiling模式和device local memory。问题点在于**depth image正确的format是什么？**该format应该包含一个depth component，在VK_FORMAT_中通过_D??_指示。

不像纹理贴图，我们不需要一个特定的format，因为我们不会从程序中直接访问这些深度texels。它只**需要有一个合理的精度**，通常在实践中**至少为24bits**。有许多format符合这个要求：

1. **VK_FORMAT_D32_SFLOAT**：32位浮点深度。
2. **VK_FORMAT_D32_SFLOAT_S8_UINT**：32位浮点深度以及8位模板值。
3. **VK_FORMAT_D24_UNORM_S8_UINT**：24位浮点深度以及8位模板值。

模板值stencil component用于**模板测试**[stencil tests](https://en.wikipedia.org/wiki/Stencil_buffer)，是另一个可以和深度测试结合使用的测试。我们会在后面章节看到它。

我们可以简单使用**VK_FORMAT_D32_SFLOAT**格式，因为极大部分硬件都支持它，但是为我们的程序尽可能增加一些额外的灵活性也是不错的。我们会写一个函数**findSupportedFormat**，按从最理想到最不理想的顺序获取候选格式列表，并且从第一个开始检查是否支持。

```c++
	VkFormat findSupportedFormat(const std::vector<VkFormat>& candidates, VkImageTiling tiling, VkFormatFeatureFlags features)
	{

	}
```

**format是否支持还取决于tiling mode和usage**，所以我们必须也包含它们作为函数传入参数。format是否支持可以使用**vkGetPhysicalDeviceFormatProperties**函数进行查询。

```c++
		for (VkFormat format : candidates)
		{
			VkFormatProperties props;
			vkGetPhysicalDeviceFormatProperties(physicalDevice, format, &props);
		}
```

**VkFormatProperties**结构体包含三个字段：

1. **linearTilingFeatures**：线性平铺支持的cases。
2. **optimalTilingFeatures**：最佳平铺支持的cases。
3. **bufferFeatures**：buffers支持的cases。

只有前两个和我们这里的参数相关，我们检查哪个取决于函数传入的tiling参数。

```c++
			if (tiling == VK_IMAGE_TILING_LINEAR && (props.linearTilingFeatures & features) == features)
			{
				return format;
			}
			else if (tiling == VK_IMAGE_TILING_OPTIMAL && (props.optimalTilingFeatures & features) == features)
			{
				return format;
			}
```

如果没有候选formats支持期望的usage，那么我们可以返回一个特殊值或者抛出一个异常。

```c++
	VkFormat findSupportedFormat(const std::vector<VkFormat>& candidates, VkImageTiling tiling, VkFormatFeatureFlags features)
	{
		for (VkFormat format : candidates)
		{
			VkFormatProperties props;
			//查询format是否支持
			vkGetPhysicalDeviceFormatProperties(physicalDevice, format, &props);

			if (tiling == VK_IMAGE_TILING_LINEAR && (props.linearTilingFeatures & features) == features)
			{
				return format;
			}
			else if (tiling == VK_IMAGE_TILING_OPTIMAL && (props.optimalTilingFeatures & features) == features)
			{
				return format;
			}
		}

		//如果没有支持的format，则抛出异常
		throw std::runtime_error("failed to find supported format");
	}
```

我们现在再创建一个**findDepthFormat**函数作为heler function，在其中调用该函数，用来**选择一个支持depth attachment usage的包含一个depth component的format**。

```c++
	VkFormat findDepthFormat()
	{
		return findSupportedFormat({ VK_FORMAT_D32_SFLOAT, VK_FORMAT_D32_SFLOAT_S8_UINT, VK_FORMAT_D24_UNORM_S8_UINT },
			VK_IMAGE_TILING_OPTIMAL,
			VK_FORMAT_FEATURE_DEPTH_STENCIL_ATTACHMENT_BIT);
	}
```

在这里，确保**使用VK_FORMAT_FEATURE_标识而不是VK_IMAGE_USAGE**。所有候选的format都包含depth component，但是后两者也包含一个stencil component。我们目前不会用到stencil，但是当对这些格式的image执行layout transitions的时候，我们需要将其考虑在内。增加一个简单的helper function来告诉我们选择的depth format是否包含stencil component。

```c++
	bool hasStencilComponent(VkFormat format)
	{
		return format == VK_FORMAT_D32_SFLOAT_S8_UINT || format == VK_FORMAT_D24_UNORM_S8_UINT;
	}
```

在**createDepthResources**中调用该函数来找到depth format。

```c++
		//找到适合的depth format
		VkFormat depthFormat = findDepthFormat();
```

我们现在拥有了调用**createImage**和**createImageView**这两个helper function需要的所有信息。

```c++
		createImage(swapChainExtent.width, swapChainExtent.height, depthFormat,
			VK_IMAGE_TILING_OPTIMAL, VK_IMAGE_USAGE_DEPTH_STENCIL_ATTACHMENT_BIT,
			VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, depthImage, depthImageMemory);
		depthImageView = createImageView(depthImage, depthFormat);
```

但是，**createImageView**函数现在**假设subresource总是VK_IMAGE_ASPECT_COLOR_BIT**，所以我们需要将该字段变成一个参数。

```c++
	//创建image view的抽象
	VkImageView createImageView(VkImage image, VkFormat format, VkImageAspectFlags aspectFlags)
	{
		...
		viewInfo.subresourceRange.aspectMask = aspectFlags;
		...
	}
```

更新该函数的所有调用处来使用正确的aspect。

```c++
	swapChainImageViews[i] =createImageView(swapChainImages[i],swapChainImageFormat,VK_IMAGE_ASPECT_COLOR_BIT);
	
	textureImageView =createImageVi(textureImageVK_FORMAT_R8G8B8A8_SRGBVK_IMAGE_ASPECT_COLOR_BIT);

	depthImageView = createImageView(depthImage, depthFormat, VK_IMAGE_ASPECT_DEPTH_BIT);
```

这就是创建depth image的过程（用helper function果然方便了很多）。我们不需要映射它或者拷贝其他的image到它，因为我们会**在render pass的开始清理它**，就像color attachment一样。

#### 4 显式转换深度图像 Explicitly transitioning the depth image

我们不需要显式转换作为depth attachment的image的layout，因为我们会在render pass中处理这个问题。但是，为了完整性，我们还是在本节中描述该过程。我们可以选择性跳过。

在**createDepthResources**函数中调用**transitionImageLayout**如下。

```c++
		//显式转换image layout
		transitionImageLayout(depthImage, depthFormat, VK_IMAGE_LAYOUT_UNDEFINED, VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL);
```

**Undefined layout可以用作初始化的layout**，因为没有会有所影响的depth image内容。我们需要**更新transitionImageLayout**中的一些逻辑来**使用正确的subresources aspect**。

```c++
		//使用正确的subresource aspect
		if (newLayout == VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL)
		{
			barrier.subresourceRange.aspectMask = VK_IMAGE_ASPECT_DEPTH_BIT;

			if (hasStencilComponent(format))
			{
				barrier.subresourceRange.aspectMask |= VK_IMAGE_ASPECT_STENCIL_BIT;
			}
		}
		else
		{
			barrier.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
		}
```

虽然我们不会使用到**stencil component**，但我们还需要在depth image的layout transition中考虑到它。

最后，增加正确的**access masks和pipeline stages**。

```c++
		if (oldLayout == VK_IMAGE_LAYOUT_UNDEFINED && newLayout == VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL)
		{
			barrier.srcAccessMask = 0;//write不需要等待任何事情
			barrier.dstAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;//transfer write必须发生在pipeline transfer stage

			sourceStage = VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT;//最早的可能的pipeline stage
			destinationStage = VK_PIPELINE_STAGE_TRANSFER_BIT;
		}
		else if (oldLayout == VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL && newLayout == VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL)
		{
			barrier.srcAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;
			barrier.dstAccessMask = VK_ACCESS_SHADER_READ_BIT;//image将在同一pipeline stage写入，随后由片元着色器读取

			sourceStage = VK_PIPELINE_STAGE_TRANSFER_BIT;
			destinationStage = VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT;
		}
		else if (oldLayout == VK_IMAGE_LAYOUT_UNDEFINED && newLayout == VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL)
		{
			barrier.srcAccessMask = 0;
			barrier.dstAccessMask = VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_READ_BIT | VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT;

			sourceStage = VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT;
			destinationStage = VK_PIPELINE_STAGE_EARLY_FRAGMENT_TESTS_BIT;
		}
		else
		{
			throw std::invalid_argument("unsupported layout transition!");
		}
```

depth buffer会被读取用于深度测试来判断一个片元是否可见，并且当一个新片元被绘制时会写入深度值。读取depth buffer发生在**VK_PIPELINE_STAGE_EARLY_FRAGMENT_TESTS_BIT**阶段，写入发生在**VK_PIPELINE_STAGE_LATE_FRAGMENT_TESTS_BIT**阶段。我们需要选择与指定操作匹配的最早的pipeline stage（在这里就是EARLY_FRAGMENT_TESTS），以便在需要时可以用作depth attachment。

#### 5 渲染通道 Render pass

现在我们来修改**createRenderPass**函数来包含一个depth attachment。首先**指定VkAttachmentDescription**。

```c++
		//创建depth attachment描述符
		VkAttachmentDescription depthAttachment{};
		depthAttachment.format = findDepthFormat();
		depthAttachment.samples = VK_SAMPLE_COUNT_1_BIT;
		depthAttachment.loadOp = VK_ATTACHMENT_LOAD_OP_CLEAR;
		depthAttachment.storeOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
		depthAttachment.stencilLoadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
		depthAttachment.stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
		depthAttachment.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
		depthAttachment.finalLayout = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;
```

**format**字段应该和depth image的format相同。我们**不需要存储depth数据**（storeOp），因为在绘制完成之后它不会被用到。这也许会**允许硬件来执行额外的优化**（移动端会对带宽优化明显，不需要把attachment传会到内存）。和color attachment很像，我们**不关心之前的depth内容**，所以我们可以**使用VK_IMAGE_LAYOUT_UNDEFINED作为initialLayout**。

```c++
		//subpass使用的depth attachment引用
		VkAttachmentReference depthAttachmentRef{};
		depthAttachmentRef.attachment = 1;
		depthAttachmentRef.layout = VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL;
```

为第一个（并且仅有）的subpass增加**对attachment的引用**。

```c++
		//指定subpass的attachment引用
		subpass.colorAttachmentCount = 1;
		subpass.pColorAttachments = &colorAttachmentRef;
		subpass.pDepthStencilAttachment = &depthAttachmentRef;
```

不像color attachments，**一个subpass只可以使用单独的一个depth（+stencil） attachment**，对多个buffers进行深度测试实际上没有任何意义。

```c++
		//创建render pass
		std::array<VkAttachmentDescription, 2> attachments = { colorAttachment, depthAttachment };
		VkRenderPassCreateInfo renderPassInfo{};
		renderPassInfo.sType = VK_STRUCTURE_TYPE_RENDER_PASS_CREATE_INFO;
		renderPassInfo.attachmentCount = static_cast<uint32_t>(attachments.size());
		renderPassInfo.pAttachments = attachments.data();
		renderPassInfo.subpassCount = 1;
		renderPassInfo.pSubpasses = &subpass;
		renderPassInfo.dependencyCount = 1;
		renderPassInfo.pDependencies = &dependency;
```

接下来，更新**VkSubpassDependency**结构体来指向两个attachments。

```c++
		//配置subpass dependency
		VkSubpassDependency dependency{};
		dependency.srcSubpass = VK_SUBPASS_EXTERNAL; // 被依赖的subpass
		dependency.dstSubpass = 0; //依赖的subpass
		dependency.srcStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT | VK_PIPELINE_STAGE_EARLY_FRAGMENT_TESTS_BIT;
		dependency.srcAccessMask = 0;
		dependency.dstStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT | VK_PIPELINE_STAGE_EARLY_FRAGMENT_TESTS_BIT;
		dependency.dstAccessMask = VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT | VK_ACCESS_DEPTH_STENCIL_ATTACHMENT_WRITE_BIT;
```

最后，我们需要拓展我们的subpass dependencies，**确保depth image的转换与在load操作时的clear之间不存在冲突**。Depth image会在early fragment test pipeline stage被第一次访问，因为我们在load的时候执行clear操作，所以我们需要为写入指定access mask。

#### 6 帧缓冲 Framebuffer

下一步是修改framebuffer的创建，**将depth image绑定到depth attachment**。转到createFramebuffers，**指定depth image view作为第二个attachment**。

```c++
			std::array<VkImageView, 2> attachments = {
				swapChainImageViews[i],
				depthImageView
			};

			VkFramebufferCreateInfo framebufferInfo{};
			framebufferInfo.sType = VK_STRUCTURE_TYPE_FRAMEBUFFER_CREATE_INFO;
			framebufferInfo.renderPass = renderPass; // 要兼容的render pass
			framebufferInfo.attachmentCount = static_cast<uint32_t>(attachments.size());
			framebufferInfo.pAttachments = attachments.data(); // 对应的imageviews
			framebufferInfo.width = swapChainExtent.width;
			framebufferInfo.height = swapChainExtent.height;
			framebufferInfo.layers = 1;
```

color attachment对于每个swap chain image不同，但是**同一个depth image可以用于所有sawp chain iamge**，因为我们的semaphores，**同一时刻只有一个subpass在运行**。

我们也需要移动createFramebuffers的调用，**确保在depth image view被实际创建之后再调用**。

```c++
	//initVulkan函数来用于实例化Vulkan objects私有成员
	void initVulkan()
	{
		...
		createDepthResources();
		createFramebuffers();
		...
	}
```

#### 7 清理值 Clear values

因为我们现在有多个VK_ATTACHMENT_LOAD_OP_CLEAR的attachment，所以我们也需要**指定多个clear values**。转到recordCommandBuffer，创建一个VkClearValue数组。

```c++
		//在使用VK_ATTACHMENT_LOAD_OP_CLEAR时使用的Clear Values
		std::array<VkClearValue, 2> clearValues{};
		clearValues[0].color = {{0.0f, 0.0f, 0.0f, 1.0f}};
		clearValues[1].depthStencil = { 1.0f, 0 };
		renderPassInfo.clearValueCount = static_cast<uint32_t>(clearValues.size());
		renderPassInfo.pClearValues = clearValues.data();
```

在Vulkan中，**depth buffer中depths值的范围是0.0到1.0，1.0位于摄像机远平面，0.0位于摄像机近平面**。Depth buffer中的每个像素的初始值应该是最远的深度1.0。

#### 8 深度和模板状态 Depth and stencil state

现在，depth attachment可以被使用了，但是**深度测试需要在graphics pipeline中开启**。它通过**VkPipelineDepthStencilStateCreateInfo**结构体来配置。

```c++
		//开启深度测试
		VkPipelineDepthStencilStateCreateInfo depthStencil{};
		depthStencil.sType = VK_STRUCTURE_TYPE_PIPELINE_DEPTH_STENCIL_STATE_CREATE_INFO;
		depthStencil.depthTestEnable = VK_TRUE;
		depthStencil.depthWriteEnable = VK_TRUE;
```

**depthTestEnable**字段指定了新片元的深度是否应该和depth buffer比较来判断其是否应该被丢弃。**depthWriteEnable**字段指定了通过深度测试的新片元的深度值是否应该写入depth buffer。

```c++
		depthStencil.depthCompareOp = VK_COMPARE_OP_LESS;
```

**depthCompareOp**字段指定了判断留下或者丢弃片元的比较方式。我们坚持较小深度=更接近的约定，因此新片元的深度应该更小。

```c++
		depthStencil.depthBoundsTestEnable = VK_FALSE;
		depthStencil.minDepthBounds = 0.0f; // Optional
		depthStencil.maxDepthBounds = 1.0f; // Optional
```

**depthBoundsTestEnable**、**minDepthBounds**和**maxDepthBounds**字段用于可选的深度限制测试。简单来说，它允许我们只保留指定深度范围内的片元。我们不会用到该功能。

```c++
		depthStencil.stencilTestEnable = VK_FALSE;
		depthStencil.front = {}; // Optional
		depthStencil.back = {}; // Optional
```

最后三个字段配置**模板缓冲操作**，在教程中我们也不会用到。如果我们想要使用这些操作，我们就必须要正depth/stencil image的格式包含stencil component。

```c++
		pipelineInfo.pDepthStencilState = &depthStencil;
```

更新**VkGraphicsPipelineCreateInfo**结构体，引用我们刚填充的depth stencil state。**如果render pass包含一个depth stencil attachment，depth stencil state必须总是被指定**。

如果我们现在运行程序，我们会看到几何体的片元被正确地排序了~（好耶，Depth buffer配置起来很简单，因为大部分工作之前都已经铺垫了）。

![20240215153624](https://raw.githubusercontent.com/recaeee/PicGo/main/20240215153624.png)

#### 9 处理窗口尺寸变化 Handling window resize

depth buffer的分辨率应该在window发生resize时改变，以匹配新的color attachment的分辨率。拓展**recreateSwapChain**函数，在这种情况下**重建depth resources**。

```c++
	//当window surface和swap chain不再兼容时重建swap chain
	void recreateSwapChain()
	{
		//当窗口最小化时暂停运行
		int width = 0, height = 0;
		glfwGetFramebufferSize(window, &width, &height);
		while (width == 0 || height == 0)
		{
			glfwGetFramebufferSize(window, &width, &height);
			glfwWaitEvents();
		}

		vkDeviceWaitIdle(device);

		cleanupSwapChain();

		createSwapChain();
		createImageViews();
		createDepthResources();
		createFramebuffers();
	}
```

**Depth资源的清理**操作应该在**swap chain的cleanup**函数中发生。

```c++
	//重建swap chain前清理旧资源
	void cleanupSwapChain()
	{
		//销毁depth resources
		vkDestroyImageView(device, depthImageView, nullptr);
		vkDestroyImage(device, depthImage, nullptr);
		vkFreeMemory(device, depthImageMemory, nullptr);
		...
	}
```

塔诺西~我们的应用现在可以准备渲染任意的3D几何体并且能够显示正确了。下期预告，我们会尝试绘制一个带纹理的模型~

#### 参考
1. 题图来自画师wlop
2. https://zh.wikipedia.org/zh-cn/%E8%8E%AB%E9%87%8C%E8%8C%A8%C2%B7%E7%A7%91%E5%86%85%E5%88%A9%E6%96%AF%C2%B7%E5%9F%83%E8%88%8D%E5%B0%94
3. https://en.wikipedia.org/wiki/Stencil_buffer


