---
layout:     post
title:      "【VulkanTutorial学习笔记——12】Texture mapping"
subtitle:   ""
date:       2024-02-11 12:00:00
author:     "recaeee"
header-img: "img/in-post/default-bg.jpg"
tags:
    - Vulkan
---
# 【VulkanTutorial学习笔记——12】Texture mapping

![20240211204909](https://raw.githubusercontent.com/recaeee/PicGo/main/20240211204909.png)

#### 0 写在前面
本系列为个人在跟随Vulkan Tutorial学习过程中记录的一些笔记，一是防止自己看着看着弃坑，二是希望能在看过之后多少留些印象，三是想在啥时候忘记了能快速地回忆一些Vulkan知识点。

出于以上这些目的，该系列的学习笔记可能会写得较随意一些，并且对于主观判断不重要的东西可能会略过，因此也不保证该系列笔记的学习质量，可能有些东西不经过脑子就会写出来。

我是一个Vulkan的小白，对于Vulkan尚不熟悉，笔记中可能存在错误，或者概念不明确的地方，也希望看的人可以批评斧正~

首先祝大家新年快乐~

这一章来到了纹理映射，我们会在这一章中了解到Vulkan中Texture资源的存储方式，以及如何在shader中采样texture。涉及到的内容很多，包括texture image、texture image view、sampler、combined image sampler descriptor等等内容，长文警告~

原教程链接：
https://vulkan-tutorial.com/Introduction
本章节链接：
https://vulkan-tutorial.com/Texture_mapping/Images
https://vulkan-tutorial.com/Texture_mapping/Image_view_and_sampler
https://vulkan-tutorial.com/Texture_mapping/Combined_image_sampler

#### 1 纹理映射 Texture mapping

迄今为止，我们使用每顶点的颜色给几何图形带来了色彩，但它是一个很有限制的方法。在接下来的章节，我们将会实践**纹理映射Texture mapping**，让几何图形看起来更有趣~在未来章节，我们可以加载和绘制基础的3D模型。

在我们的程序中**增加一个纹理Texutre**将会遵循以下步骤：

1. 创建**由device memory支持的image对象**。
2. 用**图像文件**中的像素填充它。
3. 创建一个**采样器image sampler**。
4. 添加一个combined **image sampler descriptor**来从texture中采样颜色。

在之前，我们已经和image对象打过交道，但是它们是通过swap chain拓展自动创建的。这次，我们将自己创建一个image。创建一个image并向其中填入数据和vertex buffer的创建过程非常像。首先，我们**创建一个暂存资源staging resource并且向其中填入像素数据**，然后我们**将其拷贝到最终用于渲染的image对象中**。虽然我们可以为此创建一个staging image，但Vulkan也允许我们**从一个VKBuffer拷贝像素到一个image**，并且在一些硬件上这么做实际上更快速。我们首先创建该buffer，然后向其中填入像素值，然后我们将创建一个image将像素再拷贝进去。创建一个image和创建buffer并没啥不同。它包括了**询问内存要求、分配device memory和绑定**，就我们之前看到的类似。

但是，当我们处理image时，我们需要考虑额外的一些事情。Images可以有不同的**布局layout**，**影响像素在内存中如何排布**。考虑到显卡的工作方式，**一行一行地存储像素数据并不会得到最佳的性能**。**对images执行任何操作时，我们都需要保证它们是处于该操作的最优layout**。在指定render pass时，我们已经看到过一些layout：

1. **VK_IMAGE_LAYOUT_PRESENT_SRC_KHR**：最适合presentation。
2. **VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL**：最适合作为attachment从片元着色器写入颜色。
3. **VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL**：最适合作为传输操作的源，比如vkCmdCopyImageToBuffer。
4. **VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL**：最适合作为传输操作的目的地，比如vkCmdCopyBufferToImage。
5. **VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL**：最适合从shader中采样。

**改变image的layout**最普遍的方式是**管线屏障pipeline barrier**。**Pipeline barrier**主要用于**同步资源的访问**，比如保证一个image在被读取之前已经被写入，但是它们也可以用于改变layout。在这一章，我们将会看到如何使用pipeline barriers来改变layout。使用VK_SHARING_MODE_EXCLUSIVE时，barriers还可用于改变queue family所有权。

#### 2 图像库 Image library

有许多库可以用于加载图像，我们甚至可以自己写代码来加载BMP和PPM这种简单恶的图像格式。在教程中，我们会使用[stb collection](https://github.com/nothings/stb)中的**stb_image库**。它的好处是所有代码只在一个文件中，所以不需要任何棘手的构建配置。

配置过程略~（可以去看原教程哦，非常EZ）

#### 3 加载一个图像 Loading an image

首先，include图像库。

```c++
//图像加载库
#define STB_IMAGE_IMPLEMENTATION
#include <stb_image.h>
```

头文件只定义了函数的原型，我们需要在include前定义STB_IMAGE_IMPLEMENTATION，以包含函数体，否则我们会得到linking错误。

```c++
	//initVulkan函数来用于实例化Vulkan objects私有成员
	void initVulkan()
	{
		...
		createCommandPool();
		createTextureImage();
		createVertexBuffer();
		...
	}

	void createTextureImage()
	{

	}
```

创建一个新函数**createTextureImage**，我们将会在其中**加载一张图像，然后将其上传到一个Vulkan image对象中**。我们会用到command bufer，所以它需要在**createCommandPool**之后调用。

在shaders文件夹同目录下创建一个新的文件夹textures来存储纹理图像。我们将会**从该文件夹中加载一张名为texture.jpg的图像**。教程中使用以下CC0许可的图像，大小调整为512x512，但是我们也可以随意选择想要的任何图像。该图像库支持大部分常见的图像文件格式，比如JPEG、PNG、BMP和GIF。

![20240209174141](https://raw.githubusercontent.com/recaeee/PicGo/main/20240209174141.png)

通过该库加载一张图像很简单，如下。（注意，我这里使用了png，原教程使用了jpg，个人不太喜欢jpg这种有损压缩的图片，容易有坑）

```c++
	void createTextureImage()
	{
		int texWidth, texHeight, texChannels;
		stbi_uc* pixels = stbi_load("textures/texture.png", &texWidth, &texHeight, &texChannels, STBI_rgb_alpha);
		VkDeviceSize  imageSize = texWidth * texHeight * 4;

		if (!pixels)
		{
			throw std::runtime_error("failed to load texture image!");
		}
	}
```

**stbi_load**函数使用文件路径和要加载的通道数作为参数。**STBI_rgb_alpha**值强迫图像加载一个aplha通道，即使它本身没有，这对于将来与其他纹理的一致性很有好处。中间的三个参数是图像的宽、高、实际的通道数的输出。返回的指针是像素值数组的第一个元素。像素是一行一行排布的，在STBI_rgb_alpha的情况下，每个像素占用4个字节。

#### 4 暂存缓冲区 Staging buffer

接下来，我们**在host visible memory中创建一个buffer**，由此我们就可以使用**vkMapMemory**，然后将像素数据拷贝进buffer。在**createTextureImage**中增加一个临时buffer变量。

```c++
		//staging buffer
		VkBuffer stagingBuffer;
		VkDeviceMemory stagingBufferMemory;
```

这个buffer（即staging buffer）应该位于**host visible memory**（即CPU可以访问），所以我们可以映射它，并且它应该作为一个**传输源**，由此我们之后可以从它拷贝数据到一个Vulkan image对象。

```c++
		createBuffer(imageSize, VK_BUFFER_USAGE_TRANSFER_SRC_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, stagingBuffer, stagingBufferMemory);
```

然后我们可以直接**将从加载图像文件得到的像素值数据拷贝到该staging buffer**。

```c++
		//将加载的图像文件像素数据拷贝到staging buffer
		void* data;
		vkMapMemory(device, stagingBufferMemory, 0, imageSize, 0, &data);
		memcpy(data, pixels, static_cast<size_t>(imageSize));
		vkUnmapMemory(device, stagingBufferMemory);
```

不要忘记销毁原始的像素数组。

```c++
		//销毁原始的像素数组
		stbi_image_free(pixels);
```

#### 5 纹理图像 Texture Image

虽然我们可以设置着色器来访问buffer中的像素值，但是使用一个Vulkan的image对象更好。**Image对象让获取颜色更加简单和快速，它允许我们使用2D纹理坐标**。**一个image对象中的像素pixel也被称为纹素texel**，从现在开始，我们将会该名称。增加以下新的类成员。

```c++
	//存储纹理image及其对应device memory
	VkImage textureImage;
	VkDeviceMemory textureImageMemory;
```

一个image的参数在**VkImageCreateInfo**结构体中被指定。

```c++
		//创建image
		VkImageCreateInfo imageInfo{};
		imageInfo.sType = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO;
		imageInfo.imageType = VK_IMAGE_TYPE_2D;
		imageInfo.extent.width = static_cast<uint32_t>(texWidth);
		imageInfo.extent.height = static_cast<uint32_t>(texHeight);
		imageInfo.extent.depth = 1;
		imageInfo.mipLevels = 1;
		imageInfo.arrayLayers = 1;
```

**imageType**字段指定image类型，**告诉Vulkan image中的texels将会使用哪种坐标系**。创建1D、2D和3D image是可行的。一维的images可以用于存储数据或梯度的数组，二维的images主要用于纹理textures，三维的images可以用于存储体素。**extent**字段指定了image的尺寸，即每个轴上有多少texels，这也是为什么depth必须是1，而不是0（即Z轴上是1个texel）。我们的texture不会是一个数组，因此我们现在不会使用**mipmapping**。

```c++
		imageInfo.format = VK_FORMAT_R8G8B8A8_SRGB;
```

Vulkan支持许多可能的image formats，但是我们会**使用和buffer中的pixels相同的texels格式**，**否则拷贝操作将会失败**。

```c++
		imageInfo.tiling = VK_IMAGE_TILING_OPTIMAL;
```

**tiling**字段可以有以下两个值：

1. **VK_IMAGE_TILING_LINEAR**：texels按照行主序排列，就像我们的pixels数组一样。
2. **VK_IMAGE_TILING_OPTIMAL**：texels按照实现定义的顺序进行排布，以实现最佳访问。

不像image的layout，**tiling mode在之后不可以被改变**。**如果我们想要直接访问内存中image中的texels，则我们必须使用VK_IMAGE_TILING_LINEAR**。我们将会使用一个staging buffer而不是staging image，所以它是没必要的。我们将会使用VK_IMAGE_TILING_OPTIMAL来让shader更高效地访问image。

```c++
		imageInfo.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
```

一个image的**initialLayout**只有两个可能值：

1. **VK_IMAGE_LAYOUT_UNDEFINED**：GPU无法使用，并且第一个transition将会丢弃texels。
2. **VK_IMAGE_LAYOUT_PREINITIALIZED**：GPU无法使用，但是第一个transition将保留texels。

**只有很少的情况下，我们需要在第一次transition时保留texels**。一个例子是，如果我们想将一个image与VK_IMAGE_TILING_LINEAR布局结合作为**staging image**。在这种情况下，我们想要向其中上传texel数据，然后将该image作为传输源，并且不让它丢失数据。但在我们的情况下，我们将会**把image作为传输目的地**，从一个buffer对象中拷贝texel数据到image中，所以我们不需要该property，可以安全地使用**VK_IMAGE_LAYOUT_UNDEFINED**。

```c++
		imageInfo.usage = VK_IMAGE_USAGE_TRANSFER_DST_BIT | VK_IMAGE_USAGE_SAMPLED_BIT;
```

**usage**字段与buffer创建期间地的语义相同。image将会被用作buffer拷贝的目的地，所以它应该被设置为**传输目的地**。我们也希望在shader中访问该image来为mesh着色，因此usage需要包含**VK_IMAGE_USAGE_SAMPLED_BIT**。

```c++
		imageInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;
```

**image将只在一个queue family中被使用**：即支持graphics（当然也支持transfer操作）的queue family。

```c++
		imageInfo.samples = VK_SAMPLE_COUNT_1_BIT;
		imageInfo.flags = 0; // Optional
```

**samples**标识符和**多重采样multisampling**相关，它只和被用作attachment的images相关，所以我们使用1。这里有许多可选的**flags**和**稀疏image**相关。**稀疏图像Sparse Image是实际上仅在某些区域在内存中的image**。例如，如果我们想要为体素使用一张3D texture，我们可以使用它来避免为存储大量“air”值的体素信息而分配内存。在教程中，我们不会使用它，所以将其值置为默认值0。

```c++
		if (vkCreateImage(device, &imageInfo, nullptr, &textureImage) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to create image!");
		}
```

通过**vkCreateImage**来创建image，没有任何值得关注的新参数。可能的情况是**显卡不支持VK_FORMAT_R8G8B8A8_SRGB格式**。我们应该有一个支持的可选格式列表，然后选择显卡支持的最合适的那个格式。但是，对于这个格式的支持非常普遍，因此我们跳过该步骤。使用不同格式还需要烦人的转换。我们将会在depth buffer章节回到这，那是我们会需要这样一个system。

```c++
		//获取image对应的内存要求（找到对应的内存类型）
		VkMemoryRequirements memRequirements;
		vkGetImageMemoryRequirements(device, textureImage, &memRequirements);
		//分配内存
		VkMemoryAllocateInfo allocInfo{};
		allocInfo.sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
		allocInfo.allocationSize = memRequirements.size;
		allocInfo.memoryTypeIndex = findMemoryType(memRequirements.memoryTypeBits, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT);

		if (vkAllocateMemory(device, &allocInfo, nullptr, &textureImageMemory) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to allocate image memory!");
		}
		//绑定image和为其分配的实际内存
		vkBindImageMemory(device, textureImage, textureImageMemory, 0);
```

为一个image分配内存的过程几乎和为一个buffer分配内存的过程一模一样。使用**vkGetImageMemoryRequirements**而不是vkGetBufferMemoryRequirements，并且使用**vkBindImageMemory**而不是vkBindBufferMemory。

该函数现在变得非常庞大，并且后续章节我们会创建其他更多的images，所以我们**将image的创建抽象到一个createImage函数中**，就像我们为buffer做的。创建一个函数，并且将image对象的创建和内存分配移动到其中。

```c++
	void createImage(uint32_t width, uint32_t height, VkFormat format, VkImageTiling tiling, VkImageUsageFlags usage, VkMemoryPropertyFlags properties, VkImage& image, VkDeviceMemory& imageMemory)
	{
		//创建image
		VkImageCreateInfo imageInfo{};
		imageInfo.sType = VK_STRUCTURE_TYPE_IMAGE_CREATE_INFO;
		imageInfo.imageType = VK_IMAGE_TYPE_2D;
		imageInfo.extent.width = width;
		imageInfo.extent.height = height;
		imageInfo.extent.depth = 1;
		imageInfo.mipLevels = 1;
		imageInfo.arrayLayers = 1;
		imageInfo.format = format;
		imageInfo.tiling = tiling;
		imageInfo.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;//GPU无法使用，但是第一个transition将保留texels
		imageInfo.usage = usage;
		imageInfo.samples = VK_SAMPLE_COUNT_1_BIT;
		imageInfo.sharingMode = VK_SHARING_MODE_EXCLUSIVE;

		if (vkCreateImage(device, &imageInfo, nullptr, &image) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to create image!");
		}

		//获取image对应的内存要求（找到对应的内存类型）
		VkMemoryRequirements memRequirements;
		vkGetImageMemoryRequirements(device, image, &memRequirements);
		//分配内存
		VkMemoryAllocateInfo allocInfo{};
		allocInfo.sType = VK_STRUCTURE_TYPE_MEMORY_ALLOCATE_INFO;
		allocInfo.allocationSize = memRequirements.size;
		allocInfo.memoryTypeIndex = findMemoryType(memRequirements.memoryTypeBits, properties);

		if (vkAllocateMemory(device, &allocInfo, nullptr, &imageMemory) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to allocate image memory!");
		}
		//绑定image和为其分配的实际内存
		vkBindImageMemory(device, image, imageMemory, 0);
	}
```

我们让width、height、format、tiling mode、usage和memory properties作为参数，因为在教程中这些可能因为image而发生变换。

**createTextureImage**现在应该被简化成如下代码。

```c++
	void createTextureImage()
	{
		//加载图像文件
		int texWidth, texHeight, texChannels;
		stbi_uc* pixels = stbi_load("textures/texture.png", &texWidth, &texHeight, &texChannels, STBI_rgb_alpha);
		VkDeviceSize  imageSize = texWidth * texHeight * 4;

		if (!pixels)
		{
			throw std::runtime_error("failed to load texture image!");
		}

		//创建staging buffer即对应的host visible memory
		VkBuffer stagingBuffer;
		VkDeviceMemory stagingBufferMemory;
		createBuffer(imageSize, VK_BUFFER_USAGE_TRANSFER_SRC_BIT, VK_MEMORY_PROPERTY_HOST_VISIBLE_BIT | VK_MEMORY_PROPERTY_HOST_COHERENT_BIT, stagingBuffer, stagingBufferMemory);
		
		//将加载的图像文件像素数据拷贝到staging buffer
		void* data;
		vkMapMemory(device, stagingBufferMemory, 0, imageSize, 0, &data);
		memcpy(data, pixels, static_cast<size_t>(imageSize));
		vkUnmapMemory(device, stagingBufferMemory);

		//销毁原始的像素数组
		stbi_image_free(pixels);
		//创建textureImage及其内存
		createImage(texWidth, texHeight, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_TILING_OPTIMAL,
			VK_IMAGE_USAGE_TRANSFER_DST_BIT | VK_IMAGE_USAGE_SAMPLED_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT,
			textureImage, textureImageMemory);
	}
```

#### 6 布局转换 Layout transitions

现在我们要写的函数再次涉及到录制和执行一个单独的command buffer，所以现在是个好机会将这种single time command buffer相关逻辑移动到两个函数中。

```c++
	//抽象出Single time command buffer的录制与结束
	VkCommandBuffer beginSingleTimeCommands()
	{
		VkCommandBufferAllocateInfo allocInfo{};
		allocInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO;
		allocInfo.level = VK_COMMAND_BUFFER_LEVEL_PRIMARY;
		allocInfo.commandPool = commandPool;
		allocInfo.commandBufferCount = 1;

		VkCommandBuffer commandBuffer;
		vkAllocateCommandBuffers(device, &allocInfo, &commandBuffer);

		VkCommandBufferBeginInfo beginInfo{};
		beginInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;
		beginInfo.flags = VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT;
		
		vkBeginCommandBuffer(commandBuffer, &beginInfo);

		return commandBuffer;
	}


	void endSingleTimeCommands(VkCommandBuffer commandBuffer)
	{
		vkEndCommandBuffer(commandBuffer);

		VkSubmitInfo submitInfo{};
		submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;
		submitInfo.commandBufferCount = 1;
		submitInfo.pCommandBuffers = &commandBuffer;

		vkQueueSubmit(graphicsQueue, 1, &submitInfo, VK_NULL_HANDLE);
		vkQueueWaitIdle(graphicsQueue);

		vkFreeCommandBuffers(device, commandPool, 1, &commandBuffer);
	}
```

以上函数的代码时基于现在copyBuffer中的代码写的。现在我们可以简化copyBuffer函数：

```c++
	//将一个buffer的内容拷贝到另一个buffer
	void copyBuffer(VkBuffer srcBuffer, VkBuffer dstBuffer, VkDeviceSize size)
	{
		//创建一个临时的command buffer用于内存传输操作
		VkCommandBuffer commandBuffer = beginSingleTimeCommands();

		//拷贝
		VkBufferCopy copyRegion{};
		copyRegion.srcOffset = 0;// Optional
		copyRegion.dstOffset = 0;// Optional
		copyRegion.size = size;
		vkCmdCopyBuffer(commandBuffer, srcBuffer, dstBuffer, 1, &copyRegion);

		endSingleTimeCommands(commandBuffer);
	}
```

如果我们仍使用buffers作为staging，现在我们可以写一个函数记录并执行**vkCmdCopyBufferToImage**来完成工作，但是**该command要求image首先出于正确的layout**。创建一个新函数来处理**layout转换**。

```c++
	//处理layout转换
	void transitionImageLayout(VkImage image, VkFormat format, VkImageLayout oldLayout, VkImageLayout newLayout)
	{
		VkCommandBuffer commandBuffer = beginSingleTimeCommands();

		endSingleTimeCommands(commandBuffer);
	}
```

一个执行layout转换的常用方法时使用一个**image memory barrier**。像这样的**pipeline barrier**通常用于**同步资源的访问**，比如确保在读取buffer之前完成对buffer的写入，但是当使用VK_SHARING_MODE_EXCLUSIVE时，它也**可以用于转换image layout和转换queue family所有权**。有一个等效的buffer memory barrier可以为buffer执行此操作。

```c++
		VkImageMemoryBarrier barrier{};
		barrier.sType = VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER;
		barrier.oldLayout = oldLayout;
		barrier.newLayout = newLayout;
```

前两个字段指定了layout转换。**如果我们不关心image的现有的内容，对于oldLayout，使用VK_IMAGE_LAYOUT_UNDEFINED时可行的**。

```c++
		barrier.srcQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
		barrier.dstQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
```

如果我们想要使用barrier来**转移queue family所有权**，那么这两个字段应该是相应queue family的索引。如果我们不希望这样做，**那么其值一定要设置成VK_QUEUE_FAMILY_IGNORED（不是默认值！！）**。

```c++
		barrier.image = image;
		barrier.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
		barrier.subresourceRange.baseMipLevel = 0;
		barrier.subresourceRange.levelCount = 1;
		barrier.subresourceRange.baseArrayLayer = 0;
		barrier.subresourceRange.layerCount = 1;
```

**image和subresourceRange指定了影响的image和特定的image部分**。我们的image不是数组，并且不包含mipmapping levels，所以只有一级level和一个layer。

```c++
		barrier.srcAccessMask = 0; // TODO
		barrier.dstAccessMask = 0; // TODO
```

**barriers的主要用途是同步，所以我们必须指定哪些类型的涉及资源的操作必须在barrier之前发生，以及哪些涉及资源的操作必须在barrier上等待**。尽管我们已经使用vkQueueWaitIdle来手动同步，但我们仍然需要这么做。正确的值取决于旧的和新的layout，所以一旦我们弄清楚要使用哪些转变换，我们就会回到这个问题。

```c++
		vkCmdPipelineBarrier(commandBuffer, 
			0/*TODO*/, 0/*TODO*/, 
			0,
			0, nullptr,
			0, nullptr,
			1, &barrier);
```

所有类型的pipeline barrier都会使用同一个函数来提交。在command buffer之后的第一个参数指定了**在barrier之前发生的操作属于哪个pipeine stage**。第二个参数指定了**属于哪个pipeline stage的操作会等待barrier**。我们允许哪些pipeline stage在barrier之前和之后，取决于我们在barrier之前和之后如何使用资源。其允许的值在[this table](https://www.khronos.org/registry/vulkan/specs/1.3-extensions/html/chap7.html#synchronization-access-types-supported)中被列出。例如，如果我们想要在barrier之后读取一个uniform，那么我们会指定VK_ACCESS_UNIFORM_READ_BIT用途以及最早会读取该uniform的shader（顶点or片元）作为pipeline stage，例如**VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT**。为这种类型的使用指定非着色器pipeline stage是没有意义的，**validation layers会警告我们正在指定一个不匹配usage类型的pipeline stage**。

第三个参数是**0或者VK_DEPENDENCY_BY_REGION_BIT**。后者**将barrier转变成per-region的条件**，例如，它意味着允许执行开始读取已经写入的资源部分。

最后三对参数引用了**三种可用类型的pipeline barriers数组**：**memory barriers、buffer memory barriers和image memory barriers**。请注意我们还没用到VkFormat参数，但我们将在depth buffer章节使用该参数进行特殊的transition。

#### 7 拷贝buffer到image Copying buffer to image

在我们回到**createTextureImage**之前，我们会再写一个helper function：**copyBufferToImage**。

```c++
	void copyBufferToImage(VkBuffer buffer, VkImage image, uint32_t width, uint32_t height)
	{
		VkCommandBuffer commandBuffer = beginSingleTimeCommands();

		endSingleTimeCommands(commandBuffer);
	}
```

就像buffer copy一样，我们需要指定buffer的哪部分会被拷贝到image的哪部分。其通过**VkBufferImageCopy**结构体来确定。

```c++
		VkBufferImageCopy region{};
		region.bufferOffset = 0;
		region.bufferRowLength = 0;
		region.bufferImageHeight = 0;

		region.imageSubresource.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
		region.imageSubresource.mipLevel = 0;
		region.imageSubresource.baseArrayLayer = 0;
		region.imageSubresource.layerCount = 1;

		region.imageOffset = { 0, 0, 0 };
		region.imageExtent = {
			width,
			height,
			1
		};
```

大部分字段是不言自明的。**bufferOffset**指定了buffer中pixel值的起始字节偏移。**bufferRowLength**和**bufferImageHeight**字段指定了image的pixels将会如何在内存中排布。例如，我们可以在image的行之间添加一些填充字节。对这两者指定0表明pixels之间是紧密排布的。**imageSubresource**、**imageOffset**和**imageExtent**字段表明我们希望将pixels拷贝到image的哪部分。

Buffer到image的拷贝操作通过**vkCmdCopyBufferToImage**函数实现。

```c++
		//拷贝buffer到image
		vkCmdCopyBufferToImage(
			commandBuffer,
			buffer,
			image,
			VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL,
			1,
			&region
		);
```

第四个参数指示了**当前image使用的layout**。我们在这里假设image已经转换为最适合将pixel复制到的layout。现在我们只将一大块pixel复制到整个image，但是可以指定一个**VkBufferImageCopy数组**来在一次操作中执行从该buffer到该image的许多不同copy操作。

#### 8 准备纹理图像 Preparing the texture image

我们现在有所有用于设置texture image的工具，接下来我们回到**createTexture**函数，来完整创建texture image。下一步是**从staging buffer拷贝到texture image**。它包含以下两个步骤：

1. 将texture image转换到**VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL**。
2. 执行buffer到image的copy函数。

使用我们之前创建的函数很简单。

```c++
		//将texture image转换到VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL
		transitionImageLayout(textureImage, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_LAYOUT_UNDEFINED, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL);
		//执行buffer到image的copy函数
		copyBufferToImage(stagingBuffer, textureImage, static_cast<uint32_t>(texWidth), static_cast<uint32_t>(texHeight));
```

**image创建时其layout为VK_IMAGE_LAYOUT_UNDEFINED**，所以在转换texture image时应该将其指定为old layout。记住，我们能这么做是因为我们不关心执行拷贝操作前image的内容是什么。

为了能在shader中采样texture image，我们需要**在copy之后进行一次transition来准备让shader访问**。

```c++
		//在copy之后进行一次transition来准备让shader访问
		transitionImageLayout(textureImage, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL);
```

#### 9 过渡屏障遮罩 Transition barrier masks

如果我们现在启用validation layers再运行程序，我们会看到它上报**transitionImageLayout的access mask和pipeline stages不合法**。我们仍然需要在transition中基于layout来设置它们。

有两个transition我们需要处理：

1. Undefined->transfer destination：传输写入不需要等待任何事情。
2. Transfer destination-shader reading：shader读取需要等待传输写入完成，特别是片元着色器中的读取，因为我们要在片元着色器中使用texture。

这些规则通过以下access masks和pipeline stages来指定。

```c++
		VkPipelineStageFlags sourceStage;
		VkPipelineStageFlags destinationStage;

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
		else
		{
			throw std::invalid_argument("unsupported layout transition!");
		}

		vkCmdPipelineBarrier(commandBuffer, 
			sourceStage, destinationStage, 
			0,
			0, nullptr,
			0, nullptr,
			1, &barrier);

		endSingleTimeCommands(commandBuffer);
```

正如我们在之前提到的表格中看到的，**transfer write必须发生在pipeline transfer stage**。因为**write不需要等待任何事情**，所以我们可以指定一个空的access mask和最早的可能的pipeline stage **VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT**。需要注意的是，**VK_PIPELINE_STAGE_TRANSFER_BIT实际上在graphics和compute pipelines中并不是一个真实的stage**，这更像是发生transfer的一个**伪阶段**。有关伪阶段的更多信息和其他示例，请参考[文档](https://www.khronos.org/registry/vulkan/specs/1.3-extensions/html/chap7.html#VkPipelineStageFlagBits)。

**image将在同一pipeline stage写入，随后由片元着色器读取**，这就是为什么我们在片元着色器阶段指定shader reading access。

如果我们未来想要做更多transition，那么我们就拓展该函数。程序现在应该能成功运行，虽然不会有可见的改变。

一个需要注意的事情是，**command buffer的提交会在一开始就导致隐式VK_ACCESS_HOST_WRITE_BIT同步**。因为transitionImageLayout函数执行了一个单独的command buffer，如果我们在layout transition中需要VK_ACCESS_HOST_WRITE_BIT依赖项，则可以使用该隐式的同步，并且将srcAccessMesk设置为0。是否要显式地实现取决于我们，但是教程中认为我们不应该依赖这些隐式操作。

实际上，有一个特殊类型的image layout**支持所有的操作**，**VK_IMAGE_LAYOUT_GENERAL**。它的问题是，它**不会为任何操作提供最佳的性能**。在一些特殊情况下需要使用它，比如使用一个image同时作为输入和输出，或者在image离开预初始化布局后被读取。

迄今为止所有提交command buffer的helper function都已设置为通过等待queue空闲来同步执行。对于实际的程序，更建议**将这些操作组合在单个command buffer中，并且异步执行以获得更高的吞吐量**，尤其是createTextureImage函数中的transitions和copy。尝试通过创建一个setupCommandBuffer来进行实验，helper function将命令录制到其中，并增加一个**flushSetupCommands**来**执行到目前为止已录制到的命令**。最好在纹理映射工作后执行此操作，以检查纹理资源是否仍然正确设置。

#### 10 清理 Cleanup

清理staging buffer和其对应memory来完成**createTextureImag**e函数。

```c++
		//在copy之后进行一次transition来准备让shader访问
		transitionImageLayout(textureImage, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL);
	
		//清理staging buffer和对应memory
		vkDestroyBuffer(device, stagingBuffer, nullptr);
		vkFreeMemory(device, stagingBufferMemory, nullptr);
```

在程序的最后清理texture image。

```c++
	//一旦window被关闭，我们将在**cleanup**函数中确保释放我们用到的所有资源
	void cleanup()
	{
		cleanupSwapChain();

		//销毁texture image和对应memory
		vkDestroyImage(device, textureImage, nullptr);
		vkFreeMemory(device, textureImageMemory, nullptr);

		...
	}
```

image现在包含了texture，但是我们仍然需要一个方法来**在graphics pipeline中访问它**。在下一节，我们来解决这个问题。

#### 11 纹理图像视图 Texture image view

在这一节，我们将会创建两个额外的资源，用于graphics pipeline采样一张image。第一个资源是我们之前处理swap chain images时已经见过的（image view），但是第二个资源是新朋友——它关系到**shader如何从image上读取texels**。

我们之前已经看到过，在处理swap chain images和framebuffer时，**images通过image views被访问**，而不是直接访问。我们也需要为texture image创建这样一个image view。

增加一个类成员来存储texture image的**VkImageView**，并且创建一个新函数**createTextureImageView**，在其中来创建该image view。

```c++
	//Texture iamge对应的image view
	VkImageView textureImageView;

	void initVulkan()
	{
		...
		createTextureImage();
		createTextureImageView();
		createVertexBuffer();
		...
	}

	void createTextureImageView()
	{

	}
```

我们可以直接基于**createImageViews**来编写该函数的代码。我们只需要改变两处，**format**和**image**。

```c++
		VkImageViewCreateInfo viewInfo{};
		viewInfo.sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO;
		viewInfo.image = textureImage;
		viewInfo.viewType = VK_IMAGE_VIEW_TYPE_2D;
		viewInfo.format = VK_FORMAT_R8G8B8A8_SRGB;
		viewInfo.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
		viewInfo.subresourceRange.baseMipLevel = 0;
		viewInfo.subresourceRange.levelCount = 1;
		viewInfo.subresourceRange.baseArrayLayer = 0;
		viewInfo.subresourceRange.layerCount = 1;
```

我们**省略了显式的viewInfo.components初始化**，因为VK_COMPONENT_SWIZZLE_IDENTITY无论如何都被定义为0。通过调用**vkCreateImageView**来完成image view的创建。

```c++
		if (vkCreateImageView(device, &viewInfo, nullptr, &textureImageView) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to create texture image view!");
		}
```

因为有这么多逻辑和createImageViews重复，我们也许向抽象出一个新的createImageView函数（刚白写了说是）。

```c++
	//创建image view的抽象
	VkImageView createImageView(VkImage image, VkFormat format)
	{
		VkImageViewCreateInfo viewInfo{};
		viewInfo.sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO;
		viewInfo.image = image;
		viewInfo.viewType = VK_IMAGE_VIEW_TYPE_2D;
		viewInfo.format = format;
		viewInfo.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
		viewInfo.subresourceRange.baseMipLevel = 0;
		viewInfo.subresourceRange.levelCount = 1;
		viewInfo.subresourceRange.baseArrayLayer = 0;
		viewInfo.subresourceRange.layerCount = 1;

		VkImageView imageView;
		if (vkCreateImageView(device, &viewInfo, nullptr, &imageView) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to create texture image view!");
		}

		return imageView;
	}
```

**createTextureImageView**函数现在可以被简化。

```c++
	void createTextureImageView()
	{
		textureImageView = createImageView(textureImage, VK_FORMAT_R8G8B8A8_SRGB);
	}
```

并且**createImageViews**函数可以被简化。

```c++
	//为swap chain中每个VkImage创建ImageView
	void createImageViews()
	{
		swapChainImageViews.resize(swapChainImages.size());
		for (size_t i = 0; i < swapChainImages.size(); i++)
		{
			swapChainImageViews[i] = createImageView(swapChainImages[i], swapChainImageFormat);
		}
	}
```

我们还需要保证在程序的最后**销毁该image view**，而且要在销毁image之前。

```c++
	//一旦window被关闭，我们将在**cleanup**函数中确保释放我们用到的所有资源
	void cleanup()
	{
		cleanupSwapChain();

		//销毁texture image view
		vkDestroyImageView(device, textureImageView, nullptr);

		//销毁texture image和对应memory
		vkDestroyImage(device, textureImage, nullptr);
		vkFreeMemory(device, textureImageMemory, nullptr);
		...
	}
```

#### 12 采样器 Samplers

shaders直接从images上读取texels是可以的，但是在images被用作textures的时候这不是一个普遍的方式。Textures通常通过**采样器samplers**被访问，它会**应用过滤filtering和转换transformations来计算得到的最终颜色**。

这些filters对于处理像**过采样oversampling**这种问题非常有效。考虑一个texture映射到一个片元比texels数量更多的几何图形上。如果我们简单地对每个片元使用最邻近纹理坐标的纹素颜色值，那么我们就会得到如下左图。

![20240211001130](https://raw.githubusercontent.com/recaeee/PicGo/main/20240211001130.png)

如果我们**通过线性插值来组合4个最邻近的texels**，我们就可以得到像右图一样的更平滑的结果。当然我们的程序可能会想要左图般的艺术效果（比如Minecraft），但是在传统的图形程序中会更偏向于右图的效果。当从一个texture上读取颜色时，一个**sampler对象自动会应用filtering**。

**欠采样Undersampling**是其对立的问题（感觉回到了GAMES101课程~），这是我们的texels数量远超了片元数量。当以锐角采样高频图案（例如棋盘纹理）时，这会导致**伪影artifacts**。

![20240211001741](https://raw.githubusercontent.com/recaeee/PicGo/main/20240211001741.png)

正如左图所示，texture在远处变得模糊混乱。其解决方法是[**各向异性过滤anisotropic filtering**](https://en.wikipedia.org/wiki/Anisotropic_filtering)，它也可以被sampler自动应用。

除了以上这些filters，一个sampler也可以应用一些**变换transformations**。它决定了当我们尝试通过其寻址模式addressing mode读取image外部的texels时会发生什么（这个也很熟悉了，GAMES101的回忆啊）。以下图片展示了几种可能：

![20240211002104](https://raw.githubusercontent.com/recaeee/PicGo/main/20240211002104.png)

现在我们创建一个函数**createTextureSampler**来设置这样一个sampler对象。我们将会使用该sampler来在shader中读取texture上的颜色。

```c++
	//initVulkan函数来用于实例化Vulkan objects私有成员
	void initVulkan()
	{
		...
		createTextureImage();
		createTextureImageView();
		createTextureSampler();
		...
	}

	void createTextureSampler()
	{

	}
```

Samplers通过**VkSamplerCreateInfo**结构体来被配置，其指定了它应该应用的所有**filters**和**transformations**。

```c++
	void createTextureSampler()
	{
		VkSamplerCreateInfo samplerInfo{};
		
		samplerInfo.sType = VK_STRUCTURE_TYPE_SAMPLER_CREATE_INFO;
		//指定了它应该应用的所有filters和transformations
		samplerInfo.magFilter = VK_FILTER_LINEAR;
		samplerInfo.minFilter = VK_FILTER_LINEAR;
	}
```

**magFilter**和**minFilter**字段指定了**如何对放大或缩小的texels进行插值**。放大涉及上面描述的过采样问题，缩小涉及欠采样问题。可选项包括VK_FILTER_NEAREST和VK_FILTER_LINEAR，对应上图中演示的模式。

```c++
		samplerInfo.addressModeU = VK_SAMPLER_ADDRESS_MODE_REPEAT;
		samplerInfo.addressModeV = VK_SAMPLER_ADDRESS_MODE_REPEAT;
		samplerInfo.addressModeW = VK_SAMPLER_ADDRESS_MODE_REPEAT;
```

**寻址模式address mode**可以使用**addressMode**字段在**每个轴**上进行指定。可选值包含以下值，大部分值都在上图中演示过了。注意到坐标轴被命名为U、V、W而不是X、Y、Z，这是纹理空间坐标的规定。

1. **VK_SAMPLER_ADDRESS_MODE_REPEAT**：当超出image尺寸时重复纹理。
2. **VK_SAMPLER_ADDRESS_MODE_MIRRORED_REPEAT**：和REPEAT类似，但超出尺寸时会反转坐标以镜像image。
3. **VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_EDGE**：取image尺寸之外最接近坐标的边缘颜色。
4. **VK_SAMPLER_ADDRESS_MODE_MIRROR_CLAMP_TO_EDGE**：和clamp到边缘类似，但是使用与最近边缘相对的边缘。
5. **VK_SAMPLER_ADDRESS_MODE_CLAMP_TO_BORDER**：当采样超出image尺寸时返回一个固定的颜色值。

在这里，我们实际上不关心使用哪种寻址模式，因为我们在教程中不会采样超出image。但是，repeat模式可能是最常见的模式，因为它可以用于像地板或者墙壁这样的纹理。

```c++
		//各向异性过滤
		samplerInfo.anisotropyEnable = VK_TRUE;
		samplerInfo.maxAnisotropy = ? ? ? ;
```

这两个字段指定了**各向异性过滤anisotropic filtering**是否启用。除非考虑性能，否则没有理由不使用它。**maxAnisotropy字段**限制了**可用于计算最终颜色的texel samples的数量**。更小的值意味着更好的氢能，但是更低质量的结果。为了了解我们可以使用哪些值，我们需要从physical device来获取该properties。

```c++
		//从physical device来获取该properties
		VkPhysicalDeviceProperties properties{};
		vkGetPhysicalDeviceProperties(physicalDevice, &properties);
```

如果我们查询**VkPhysicalDeviceProperties**结构体的文档，我们可以看到它包含一个**VkPhysicalDeviceLimits**类型成员，名为**limits**。该结构体拥有一个成员名为**maxSamplerAnisotropy**，这就是我们可以指定给maxAnisotropy的最大值。如果我们想要最好的效果，那么我们就直接使用该最大值。

```c++
		//从properties中获取各向异性过滤最大值
		samplerInfo.maxAnisotropy = properties.limits.maxSamplerAnisotropy;
```

我们可以在程序的最开始查询该properties，并且传递给需要它们的函数，或者在createTextureSampler函数内部查询它们。

```c++
		samplerInfo.borderColor = VK_BORDER_COLOR_INT_OPAQUE_BLACK;
```

**borderColor**字段指定了**在使用clamp to border寻址模式下采样超出image时的固定颜色值**，可以以float或者int格式返回黑色、白色或者透明，但**不能指定任意颜色**。

```c++
		//当对image寻址texel的时候使用哪种坐标系统
		samplerInfo.unnormalizedCoordinates = VK_FALSE;
```

**unnormalizedCoordinates**字段指定了当对image寻址texel的时候使用哪种坐标系统。**如果该字段为VK_TRUE**，那么我们可以使用[0, texWidth)和[0, texHeight)范围内的坐标。**如果该字段为VK_FALSE**，那么对于所有轴，texels使用[0, 1)范围来进行寻址。实际实践中的程序几乎总会使用**normalized coordinates**，因为这样就可以使用具有完全相同坐标的不同分辨率的textures。

```c++
		//是否启用比较函数，可用于PCF过滤实现软阴影
		samplerInfo.compareEnable = VK_FALSE;
		samplerInfo.compareOp = VK_COMPARE_OP_ALWAYS;
```

如果一个比较函数被启用，那么texels首先会和一个值进行比较，比较的结果被用与过滤操作。这通常被用于**shadow maps的PCF过滤软阴影**[percentage-closer filtering](https://developer.nvidia.com/gpugems/gpugems/part-ii-lighting-and-shadows/chapter-11-shadow-map-antialiasing)。我们会在后续章节来关注它。

```c++
		//mipmapping相关字段
		samplerInfo.mipmapMode = VK_SAMPLER_MIPMAP_MODE_LINEAR;
		samplerInfo.mipLodBias = 0.0f;
		samplerInfo.minLod = 0.0f;
		samplerInfo.maxLod = 0.0f;
```

以上这些字段都应用于**mipmapping**。我们将会在后续章节关注mipmapping，基本上它是可以应用的另一种类型的filter。

现在一个sampler的功能基本上都被定义完了（功能好多）。增加一个类成员来存储该sampler对象的句柄，并且通过**vkCreateSampler**来创建该sampler。

```c++
	//Texture image对应的sampler
	VkSampler textureSampler;

	void createTextureSampler()
	{
		...

		//创建sampler
		if (vkCreateSampler(device, &samplerInfo, nullptr, &textureSampler) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to create texture sampler!");
		}
	}
```

注意到**sampler并不会在任何地方引用到VkImage**。sampler是一个独特的对象，它提供了从texture中提取颜色的接口。它**可以被用于任何我们想要的image**，无论是1D、2D还是3D。这和许多老式的APIs不同，在老式的APIs中通常会组合texture images和filtering到一个单独的state中。

在程序的最后，当我们不再访问image时，销毁sampler。

```c++
	//一旦window被关闭，我们将在**cleanup**函数中确保释放我们用到的所有资源
	void cleanup()
	{
		cleanupSwapChain();

		//销毁sampler
		vkDestroySampler(device, textureSampler, nullptr);

		//销毁texture image view
		vkDestroyImageView(device, textureImageView, nullptr);
		...
	}
```

#### 13 各向异性过滤设备特性 Anisotropy device feature

如果我们现在运行程序，我们会看到如下的validation layer信息。

![20240211192316](https://raw.githubusercontent.com/recaeee/PicGo/main/20240211192316.png)

这是因为各向异性过滤实际上是一个可选的device feature。我们需要更新**createLogicalDevice**来申请该特性。

```c++
		//填充需要的设备特性
		VkPhysicalDeviceFeatures deviceFeatures{};
		deviceFeatures.samplerAnisotropy = VK_TRUE;//开启各向异性过滤
```

即使现代显卡不太可能不支持它，但我们也应该更新**isDeviceSuitable**函数来检查它是否可用。

```c++
	//判断该显卡是否支持我们想要的功能
	bool isDeviceSuitable(VkPhysicalDevice device)
	{
		...

		//检查是否支持各向异性过滤
		VkPhysicalDeviceFeatures supportedFeatures;
		vkGetPhysicalDeviceFeatures(device, &supportedFeatures);

		return indices.isComplete() && extensionsSupported && swapChainAdequate && supportedFeatures.samplerAnisotropy;
	}
```

**VkGetPhysicalDeviceFeatures**重新调整了**VkPhysicalDeviceFeatures**结构体的用途，**用来指示哪些features被支持**，而不是通过设置布尔值来请求哪些功能。

除了强制使用各向异性过滤之外，还可以通过条件来设置不使用它，设置代码如下。

```c++
samplerInfo.anisotropyEnable = VK_FALSE;
samplerInfo.maxAnisotropy = 1.0f;
```

在下一章，我们会接触到shader中的image和sampler对象，来在平面上绘制texture。

#### 14 更新描述符 Updating the descriptors

接下来，我们会在教程中的uniform buffer部分第一次看向descriptor。在这一节，我们会关注一种新的descriptor类型：**combined image sampler**。该descriptor**让shader通过一个sampler对象（比如我们上一节创建的）访问image资源成为可能**。

我们将会从**修改descriptor layout、descriptor pool和descriptor set**来包含这样一个combined image sampler descriptor开始。在那之后，我们会**在Vertex中增加纹理坐标**，并且**修改fragment shader**来从texture上读取颜色。

回到**createDescriptorSetLayout**函数，为combined image sampler descriptor增加一个**VkDescriptorSetLayoutBinding**。我们会简单地将其放在uniform buffer之后的一个binding。

```c++
		//创建所有bindings
		//Uniform buffer的binding
		VkDescriptorSetLayoutBinding uboLayoutBinding{};
		uboLayoutBinding.binding = 0;
		uboLayoutBinding.descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
		uboLayoutBinding.descriptorCount = 1;

		uboLayoutBinding.stageFlags = VK_SHADER_STAGE_VERTEX_BIT;//指定在哪个shader阶段该discriptor会被引用
		uboLayoutBinding.pImmutableSamplers = nullptr;// Optional

		//combined image sampler的binding
		VkDescriptorSetLayoutBinding samplerLayoutBinding{};
		samplerLayoutBinding.binding = 1;//ubo的binding为0，combined image sampler的binding为1
		samplerLayoutBinding.descriptorCount = 1;
		samplerLayoutBinding.descriptorType = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
		samplerLayoutBinding.pImmutableSamplers = nullptr;// Optional
		samplerLayoutBinding.stageFlags = VK_SHADER_STAGE_FRAGMENT_BIT;//片元着色器中使用

		//创建binding数组
		std::array<VkDescriptorSetLayoutBinding, 2> bindings = { uboLayoutBinding, samplerLayoutBinding };

		//创建descriptor set layout
		VkDescriptorSetLayoutCreateInfo layoutInfo{};
		layoutInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_SET_LAYOUT_CREATE_INFO;
		layoutInfo.bindingCount = static_cast<uint32_t>(bindings.size());
		layoutInfo.pBindings = bindings.data();
```

确保正确设置**stageFlags**来指示我们会在**fragment shader**中使用该combined image sampler descriptor，在片元着色器中会决定片元的颜色。当然**也可以在vertex shader中使用texture sampling**，例如通过[高度图heightmap](https://en.wikipedia.org/wiki/Heightmap)动态变形顶点网格（我知道的还有GPU顶点动画~）。

我们必须创建一个更大的descriptor pool，通过将另一个**VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER**类型的**VkPoolSize**添加到**VkDescriptorPoolCreateInfo**来为combined image sampler的分配腾出空间。转到**createDescriptorPool**函数，修改它来为该descriptor包含一个VkDescriptorPoolSize。

```c++
		std::array<VkDescriptorPoolSize, 2> poolSizes{};

		//uniform buffer
		poolSizes[0].type = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;//需要包含哪些descriptor类型
		poolSizes[0].descriptorCount = static_cast<uint32_t>(MAX_FRAMES_IN_FLIGHTS);//数量
		//combined image sampler
		poolSizes[1].type = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
		poolSizes[1].descriptorCount = static_cast<uint32_t>(MAX_FRAMES_IN_FLIGHTS);

		VkDescriptorPoolCreateInfo poolInfo{};
		poolInfo.sType = VK_STRUCTURE_TYPE_DESCRIPTOR_POOL_CREATE_INFO;
		poolInfo.poolSizeCount = static_cast<uint32_t>(poolSizes.size());
		poolInfo.pPoolSizes = poolSizes.data();

		poolInfo.maxSets = static_cast<uint32_t>(MAX_FRAMES_IN_FLIGHTS);//可能被分配的descriptor sets的最大数量
```

**descriptor pools不足是一个validation layers无法捕获到的一个典型的问题**：因为在Vulkan 1.1中，如果pool不够大，**vkAllocateDescriptorSets可能会伴随错误码VK_ERROR_POOL_OUT_OF_MEMORY失败**，但是驱动也许会尝试在内部解决该问题。这意味着有些时候（取决于硬件、poolSize和分配Size），**驱动程序会让我们脱离超出descriptor pool限制的分配**。另一些时候，vkAllocateDescriptorSets会失败，并返回VK_ERROR_POOL_OUT_OF_MEMORY。如果分配在某些机器上成功，但在其他机器上失败，这可能会特别令人沮丧（我太懂这种痛苦了）。

由于Vulkan将分配的责任转移给了驱动程序，因此**不再严格要求**只分配由相应的descriptorCount成员指定的特定类型的descriptor（比如VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER等）来创建descriptor pool。但是，**这样做仍然是最佳实践**，在未来，如果我们启用Best Practice Validation，VK_LAYER_KHRONOS_validation会警告这种类型的错误。

最后一步是**在descriptor set中绑定实际的image和sampler资源到descriptors**。转到**createDescriptorSets**函数。

```c++
		for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHTS; i++)
		{
			//为每个descriptor set配置其中的descriptor，每个descriptor引用一个uniform buffer
			VkDescriptorBufferInfo bufferInfo{};
			bufferInfo.buffer = uniformBuffers[i];
			bufferInfo.offset = 0;
			bufferInfo.range = sizeof(UniformBufferObject);

			//在descriptor set中绑定实际的image和sampler资源到descriptors
			VkDescriptorImageInfo imageInfo{};
			imageInfo.imageLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
			imageInfo.imageView = textureImageView;
			imageInfo.sampler = textureSampler;

			...
		}
```

一个combined image sampler结构体的资源必须通过一个**VkDescriptorImageInfo**结构体来指定，就像uniform buffer descriptor的buffer资源通过VkDescriptorBufferInfo结构体来指定。这里，我们会让上一节中的对象组合到一起。

```c++
			//更新descriptor set
			//uniform buffer
			std::array<VkWriteDescriptorSet, 2> descriptorWrites{};
			descriptorWrites[0].sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
			descriptorWrites[0].dstSet = descriptorSets[i];
			descriptorWrites[0].dstBinding = 0;
			descriptorWrites[0].dstArrayElement = 0;
			descriptorWrites[0].descriptorType = VK_DESCRIPTOR_TYPE_UNIFORM_BUFFER;
			descriptorWrites[0].descriptorCount = 1;   
			descriptorWrites[0].pBufferInfo = &bufferInfo;
			descriptorWrites[0].pImageInfo = nullptr; // Optional
			descriptorWrites[0].pTexelBufferView = nullptr; // Optional
			//combined image sampler
			descriptorWrites[1].sType = VK_STRUCTURE_TYPE_WRITE_DESCRIPTOR_SET;
			descriptorWrites[1].dstSet = descriptorSets[i];
			descriptorWrites[1].dstBinding = 1;
			descriptorWrites[1].dstArrayElement = 0;
			descriptorWrites[1].descriptorType = VK_DESCRIPTOR_TYPE_COMBINED_IMAGE_SAMPLER;
			descriptorWrites[1].descriptorCount = 1;
			descriptorWrites[1].pImageInfo = &imageInfo;

			vkUpdateDescriptorSets(device, static_cast<uint32_t>(descriptorWrites.size()), descriptorWrites.data(), 0, nullptr);
```

就和buffer类似，descriptors需要通过imageInfo来更新。这次我们使用pImageInfo数组，而不是pBufferInfo。descriptors现在可以给shader使用了！（好耶~）

#### 15 纹理坐标 Texture coordinates

目前，纹理映射还缺少一个重要要素，那就是**每个顶点的纹理坐标**。该坐标决定了image会如何映射到几何图形上。

```c++
	//顶点着色器输入
	struct Vertex {
		glm::vec2 pos;
		glm::vec3 color;
		glm::vec2 texCoord;

		static VkVertexInputBindingDescription getBindingDescription()
		{
			//顶点绑定Vertex binding描述了在整个顶点中从内存加载数据的速率。它指定了data entries之间的字节数以及是否在每个顶点后或每个实例之后移动到下一个data entry
			VkVertexInputBindingDescription bindingDescription{};
			bindingDescription.binding = 0;
			bindingDescription.stride = sizeof(Vertex);
			bindingDescription.inputRate = VK_VERTEX_INPUT_RATE_VERTEX;
			return bindingDescription;
		}

		static std::array<VkVertexInputAttributeDescription, 3> getAttributeDescriptions()
		{
			//属性描述Attribute description结构体描述了如何从源自绑定描述Binding descrription的顶点数据块中提取顶点属性
			std::array<VkVertexInputAttributeDescription, 3> attributeDescriptions{};

			attributeDescriptions[0].binding = 0;
			attributeDescriptions[0].location = 0;
			attributeDescriptions[0].format = VK_FORMAT_R32G32_SFLOAT;
			attributeDescriptions[0].offset = offsetof(Vertex, pos);

			attributeDescriptions[1].binding = 0;
			attributeDescriptions[1].location = 1;
			attributeDescriptions[1].format = VK_FORMAT_R32G32B32_SFLOAT;
			attributeDescriptions[1].offset = offsetof(Vertex, color);

			attributeDescriptions[2].binding = 0;
			attributeDescriptions[2].location = 2;
			attributeDescriptions[2].format = VK_FORMAT_R32G32_SFLOAT;
			attributeDescriptions[2].offset = offsetof(Vertex, texCoord);

			return attributeDescriptions;
		}
	};
```

修改Vertex结构体来包含一个vec2类型的纹理坐标。需要保证**增加一个VkVertexInputAttributeDescription**，这样我们就可以在vertex shader中获得作为输入的纹理坐标。这对于能够将他们传递到fragment shader以在正方形表面上进行插值是必要的。

```c++
	const std::vector<Vertex> vertices = {
		{{-0.5f, -0.5f}, {1.0f, 0.0f, 0.0f}, {1.0f, 0.0f}},
		{{0.5f, -0.5f}, {0.0f, 1.0f, 0.0f}, {0.0f, 0.0f}},
		{{0.5f, 0.5f}, {0.0f, 0.0f, 1.0f}, {0.0f, 1.0f}},
		{{-0.5f, 0.5f}, {1.0f, 1.0f, 1.0f}, {1.0f, 1.0f}}
	};
```

在教程中，使用**从左上角的0，0到右下角的1，1**的坐标，简单地用纹理填充正方形。我们也可以随意尝试不同的坐标，尝试使用低于0或高于1的坐标来查看实际的寻址模式。

#### 16 着色器 Shaders

最后一步是**修改shaders来采样texture上的颜色**（终于到最后一步了，一口老血）。我们先**修改vertex shader来将纹理坐标传递给fragment shader**。

```c++
#version 450

layout(binding = 0) uniform UniformBufferObject {
	mat4 model;
	mat4 view;
	mat4 proj;
} ubo;

layout(location = 0) in vec2 inPosition;
layout(location = 1) in vec3 inColor;
layout(location = 2) in vec2 inTexCoord;

layout(location = 0) out vec3 fragColor;
layout(location = 1) out vec2 fragTexCoord;

void main()
{
	gl_Position = ubo.proj * ubo.view * ubo.model * vec4(inPosition, 0.0, 1.0);
	fragColor = inColor;
	fragTexCoord = inTexCoord;
}
```

就和每顶点颜色一样，fragTexCoord值将由光栅化器平滑地插值到正方形区域。我们可以将其可视化，通过在fragment shader中输出纹理坐标作为颜色。

```c++
#version 450

layout(location = 0) in vec3 fragColor;
layout(location = 1) in vec2 fragTexCoord;

layout(location = 0) out vec4 outColor;

void main()
{
	outColor = vec4(fragTexCoord, 0.0, 1.0);
}
```

噔噔噔噔~重编译shader运行程序结果如下。

![20240211202241](https://raw.githubusercontent.com/recaeee/PicGo/main/20240211202241.png)

绿色通道表示水平方向的坐标，红色通标表示垂直方向的坐标。黑色和黄色的角证明了纹理坐标在正方形上正确地从0，0插值到1，1。使用颜色可视化数据相当于printf调试shader，可惜缺少更好的选择（其实RenderDoc的单个像素断点调试很好用~推荐使用）。

**combined image sampler descriptor在GLSL中由sampler uniform表示**。在片元着色器中增加一个对其的引用。

```c++
layout(binding = 1) uniform sampler2D texSampler;
```

对于其他类型的image，有等效的sampler1D和sampler3D类型。确保使用正确的binding~

```c++
void main()
{
	outColor = texture(texSampler, fragTexCoord);
}
```

现在使用**内置的texture函数**来采样texture。它使用一个sampler和纹理坐标作为参数。sampler会自动在后台使用filtering和transformations。我们现在运行程序，可以看到正方形上的纹理（泪目了）。

![20240211202953](https://raw.githubusercontent.com/recaeee/PicGo/main/20240211202953.png)

尝试通过将纹理坐标缩放到高于1的值来尝试不同的寻址模式。例如，当使用VK_SAMPLER_ADDRESS_MODE_REPEAT以及如下fragment shader后，会渲染出如下图像。

```c++
void main()
{
	outColor = texture(texSampler, fragTexCoord * 2.0);
}
```

![20240211203549](https://raw.githubusercontent.com/recaeee/PicGo/main/20240211203549.png)

我们也可以让纹理颜色乘以顶点颜色。

```c++
void main()
{
	outColor = vec4(fragColor * texture(texSampler, fragTexCoord),rgb, 1.0);
}
```

![20240211203734](https://raw.githubusercontent.com/recaeee/PicGo/main/20240211203734.png)

现在我们知道了如何在shader中访问images。当与写入帧缓冲区的图像结合使用时，这是一种非常强大的技术。我们可以使用这些images作为输入来实现很酷的效果，比如后处理和3D世界中的相机显示（类似于监控吧）。

呼，Texture mapping这篇长文终于写完了！！！内容好多，慢慢吸收~

#### 参考
1. 题图来自画师wlop
2. https://github.com/nothings/stb
3. https://www.khronos.org/registry/vulkan/specs/1.3-extensions/html/chap7.html#synchronization-access-types-supported
4. https://www.khronos.org/registry/vulkan/specs/1.3-extensions/html/chap7.html#VkPipelineStageFlagBits
5. https://en.wikipedia.org/wiki/Anisotropic_filtering
6. https://developer.nvidia.com/gpugems/gpugems/part-ii-lighting-and-shadows/chapter-11-shadow-map-antialiasing
7. https://en.wikipedia.org/wiki/Heightmap


