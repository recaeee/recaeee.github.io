---
layout:     post
title:      "【VulkanTutorial学习笔记——15】Generating Mipmaps"
subtitle:   ""
date:       2024-02-18 12:00:00
author:     "recaeee"
header-img: "img/in-post/default-bg.jpg"
tags:
    - Vulkan
---
# 【VulkanTutorial学习笔记——15】Generating Mipmaps

![20240218142926](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240218142926.png)

#### 0 写在前面
本系列为个人在跟随Vulkan Tutorial学习过程中记录的一些笔记，一是防止自己看着看着弃坑，二是希望能在看过之后多少留些印象，三是想在啥时候忘记了能快速地回忆一些Vulkan知识点。

出于以上这些目的，该系列的学习笔记可能会写得较随意一些，并且对于主观判断不重要的东西可能会略过，因此也不保证该系列笔记的学习质量，可能有些东西不经过脑子就会写出来。

我是一个Vulkan的小白，对于Vulkan尚不熟悉，笔记中可能存在错误，或者概念不明确的地方，也希望看的人可以批评斧正~

这一章，我们会实现mipmap的生成。在实际游戏开发过程中，我们就会经常接触到mipmap，它可以处理摩尔纹，也可以降低渲染压力，对于额外多出的一点点内存消耗而言，其收益无疑是非常大的。而平时，我们很少有会接触到mipmap的生成原理，在这一章节，我们就会了解这些内容，对mipmap进行一定程度的去黑箱化~加油！

原教程链接：
https://vulkan-tutorial.com/Introduction
本章节链接：
https://vulkan-tutorial.com/Generating_Mipmaps

#### 1 介绍 Inttroduction

现在我们的程序可以加载并且渲染3D模型。在这一章，我们会增加一个新特性**mipmap generation**。Mipmaps在游戏和渲染软件中被广泛使用，并且Vulkan给予了我们如何来生成它们的完全控制权。

**Mipmap是图像的预先计算的缩小版本**（相信大火都了解，Games101也讲过），每一个新的图像的宽高是上一个的一半。Mipmap用作Level of Detail（LOD）的一种形式。远离相机的对象从较小的mip图像中采样其纹理。使用更小的图像**增加了渲染速度**，并且避免了像[摩尔纹](https://en.wikipedia.org/wiki/Moir%C3%A9_pattern)这样的**伪影artifacts**。一个mipmaps的例子如下。

![20240218104239](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240218104239.png)

#### 2 图像创建 Image creation

在Vulkan中，**每个mip images都存储在VkImage的不同mip levels中**。Mip level 0是原始image，在level 0之后的mip level通常称为**mip chain**。

**当VkImage被创建时，mip levels的数量会被指定**。到现在为止，我们一直都将该值设置为1。我们需要根据image的尺寸计算mip levels的数量。首先，**增加一个类成员来存储mip levels数量**。

```c++
	//存储texture的mip levels数量
	uint32_t mipLevels;
	//存储纹理image及其对应device memory
	VkImage textureImage;
```

mipLevels的值可以在**createTextureImage**加载texture时计算。

```c++
		//加载图像文件
		int texWidth, texHeight, texChannels;
		stbi_uc* pixels = stbi_load(TEXTURE_PATH.c_str(), &texWidth, &texHeight, &texChannels, STBI_rgb_alpha);
		...

		//计算mip Levels
		mipLevels = static_cast<uint32_t>(std::floor(std::log2(std::max(texWidth, texHeight)))) + 1;
```

上述代码计算了**mip chain中的levels数量**（数量包含了mip0）。max函数筛选最大的维度，log2函数计算了该维度可以除以2几次，floor函数控制了当最大维度不是2的整数次幂的情况，增加1为了让原始image有1级mip level。

为了使用该值，我们需要改变**createImage**、**createImageView**和**transitionImageLayout**函数，允许我们指定mip levels的数量。**增加一个mipLevels传入参数**到这些函数。

```c++
	void createImage(uint32_t width, uint32_t height, uint32_t mipLevels, VkFormat format, VkImageTiling tiling, VkImageUsageFlags usage, VkMemoryPropertyFlags properties, VkImage& image, VkDeviceMemory& imageMemory)
	{
		//创建image
		...
		imageInfo.mipLevels = mipLevels;
		...
	}

	//创建image view的抽象
	VkImageView createImageView(VkImage image, VkFormat format, VkImageAspectFlags aspectFlags, uint32_t mipLevels)
	{
		...
		viewInfo.subresourceRange.levelCount = mipLevels;
		...
	}

		//处理layout转换
	void transitionImageLayout(VkImage image, VkFormat format, VkImageLayout oldLayout, VkImageLayout newLayout, uint32_t mipLevels)
	{
		...
		barrier.subresourceRange.levelCount = mipLevels;
		...
	}
```

更新这些函数的所有调用来使用正确的值（改的地方还挺多的）。

```c++
		createImage(swapChainExtent.width, swapChainExtent.height, 1, depthFormat,
			VK_IMAGE_TILING_OPTIMAL, VK_IMAGE_USAGE_DEPTH_STENCIL_ATTACHMENT_BIT,
			VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, depthImage, depthImageMemory);

		...

		//创建textureImage及其内存
		createImage(texWidth, texHeight, mipLevels, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_TILING_OPTIMAL,
			VK_IMAGE_USAGE_TRANSFER_DST_BIT | VK_IMAGE_USAGE_SAMPLED_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT,
			textureImage, textureImageMemory);

		...

		swapChainImageViews[i] = createImageView(swapChainImages[i], swapChainImageFormat, VK_IMAGE_ASPECT_COLOR_BIT, 1);

		...

		depthImageView = createImageView(depthImage, depthFormat, VK_IMAGE_ASPECT_DEPTH_BIT, 1);

		...

		textureImageView = createImageView(textureImage, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_ASPECT_COLOR_BIT, mipLevels);

		...

		//显式转换image layout
		transitionImageLayout(depthImage, depthFormat, VK_IMAGE_LAYOUT_UNDEFINED, VK_IMAGE_LAYOUT_DEPTH_STENCIL_ATTACHMENT_OPTIMAL, 1);

		...

		//将texture image转换到VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL
		transitionImageLayout(textureImage, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_LAYOUT_UNDEFINED, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, mipLevels);

		...

		//在copy之后进行一次transition来准备让shader访问
		transitionImageLayout(textureImage, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL, mipLevels);
```

#### 3 生成Mipmaps Generating Mipmaps

现在我们的texture image拥有了多个mip levels，但是**stating buffer只可以用于填充mip level 0**，剩下的levels仍然是未定义的。为了填充这些levels，我们需要**从我们拥有的单个level生成数据**。我们会使用到**vkCmdBlitImage**指令，该指令执行copying、scaling和filtering操作。我们会**调用多次来blit数据到texture image的每一个level**。

**vkCmdBlitImage**被认为是一个**传输操作**，所以我们需要告知Vulkan我们打算**使用texture image作为传输源和目的地**。在**createTextureImage**中，对texture image的usage flags增加**VK_IMAGE_USAGE_TRANSFER_SRC_BIT**。

```c++
		...
		//创建textureImage及其内存
		createImage(texWidth, texHeight, mipLevels, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_TILING_OPTIMAL,
			VK_IMAGE_USAGE_TRANSFER_SRC_BIT | VK_IMAGE_USAGE_TRANSFER_DST_BIT | VK_IMAGE_USAGE_SAMPLED_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT,
			textureImage, textureImageMemory);
		...
```

就像其他image操作，**vkCmdBlitImage依赖于其操作的image的layout**。虽然我们能将整个image转换到VK_IMAGE_LAYOUT_GENERAL，但是这可能会非常慢。为了优化性能，源image应该处于VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL，并且目的地image应该处于VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL。**Vulkan允许我们单独地改变一个image的每一级mip level的layout**。每次blit只会同时处理两个mip levels，所以我们可以**在blit操作之间改变每一个level到最优的layout**。

**transitionImageLayout只会执行整个image的layout转换**，所以我们需要写更多的pipeline barrier指令。在createTextureImage中移除现在的到VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL的转换。

```c++
		//将texture image转换到VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL
		transitionImageLayout(textureImage, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_LAYOUT_UNDEFINED, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, mipLevels);
		//执行buffer到image的copy函数
		copyBufferToImage(stagingBuffer, textureImage, static_cast<uint32_t>(texWidth), static_cast<uint32_t>(texHeight));
		//在copy之后进行一次transition来准备让shader访问
		//在生成完mipmaps之后再transition到shader read only
		//transitionImageLayout(textureImage, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL, mipLevels);
```

移除了之后，现在texture image的每一个level会处于**VK_IMAGE_LAYOUT_TRANSFER_DST_LAYOUT**。每一个level会在blit操作完成对它的读取之后转换到**VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL**。

现在，我们来写生成mipmaps的函数。

```c++
	void generateMipmaps(VkImage image, int32_t texWidth, int32_t texHeight, uint32_t mipLevels)
	{
		VkCommandBuffer commandBuffer = beginSingleTimeCommands();
		//复用该barrier
		VkImageMemoryBarrier barrier{};
		//对于所有barrier，上面设置的字段将保持不变
		barrier.sType = VK_STRUCTURE_TYPE_IMAGE_MEMORY_BARRIER;
		barrier.image = image;
		barrier.srcQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
		barrier.dstQueueFamilyIndex = VK_QUEUE_FAMILY_IGNORED;
		barrier.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
		barrier.subresourceRange.baseArrayLayer = 0;
		barrier.subresourceRange.layerCount = 1;
		barrier.subresourceRange.levelCount = 1;

		endSingleTimeCommands(commandBuffer);
	}
```

我们将会进行多次transitions，所以我们会**复用该VkImageMemoryBarrier**，对于所有barrier，**上面设置的字段将保持不变**。subresourceRange.**mipLevel**、**oldLayout**、**newLayout**、**srcAccessMask**和**dstAccessMask**将会在每次transition之间改变。

```c++
		int32_t mipWidth = texWidth;
		int32_t mipHeight = texHeight;

		for (uint32_t i = 1; i < mipLevels; i++)
		{

		}
```

该循环会录制每个**VkCmdBlitImage**指令。注意循环变量是**从1开始**的，不是0。

```c++
			//将i-1 level转换到SRC_OPTIMAL
			barrier.subresourceRange.baseMipLevel = i - 1;
			barrier.oldLayout = VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL;
			barrier.newLayout = VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL;
			barrier.srcAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;//等待i-1被填充完毕
			barrier.dstAccessMask = VK_ACCESS_TRANSFER_READ_BIT;

			vkCmdPipelineBarrier(commandBuffer,
				VK_PIPELINE_STAGE_TRANSFER_BIT, VK_PIPELINE_STAGE_TRANSFER_BIT, 0,
				0, nullptr,
				0, nullptr,
				1, &barrier);
```

首先，我们将**i-1** level转换到**VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL**。该transition会**等待i-1被填充完毕**，无论是等待前一个blit指令还是vkCmdCopyBufferToImage。当前的blit指令会等待这次transition。

```c++
			VkImageBlit blit{};
			//srcOffsets数组的两个元素决定了数据会从哪被blit的3D区域
			blit.srcOffsets[0] = { 0, 0, 0 };
			blit.srcOffsets[1] = { mipWidth, mipHeight, 1 };//srcOffsets[1]和dstOffsets[1]的Z维度必须是1，因为2Dimage的深度为1
			blit.srcSubresource.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
			blit.srcSubresource.mipLevel = i - 1;
			blit.srcSubresource.baseArrayLayer = 0;
			blit.srcSubresource.layerCount = 1;
			//dstOffsets决定了数据会被blit到哪
			blit.dstOffsets[0] = { 0, 0, 0 };
			blit.dstOffsets[1] = { mipWidth > 1 ? mipWidth / 2 : 1, mipHeight > 1 ? mipHeight / 2 : 1, 1 };
			blit.dstSubresource.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
			blit.dstSubresource.mipLevel = i;
			blit.dstSubresource.baseArrayLayer = 0;
			blit.dstSubresource.layerCount = 1;
```

接下来，我们指定会在blit操作中用到的**regions**。源mip level是i-1，目的地mip level是i。**srcOffsets**数组的两个元素决定了数据会从哪被blit的**3D**区域。**dstOffsets**决定了数据会被blit到哪。dstOffsets[1]的X和Y维度除以了2，因为每一个mip level的尺寸是上一个level的一半。srcOffsets[1]和dstOffsets[1]的Z维度必须是1，因为**2Dimage的深度为1**。

```c++
			vkCmdBlitImage(commandBuffer,
				image, VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL,
				image, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL,
				1, &blit,
				VK_FILTER_LINEAR);
```

现在，我们录制blit指令。注意**textureImage被同时用作srcImage和dstImage参数**。这是因为我们是**在同一个image的不同levels之间进行blit操作**。源mip level在刚才被转换到了**VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL**，目的地level仍然在**createTextureImage**时的**VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL**。

如果我们使用的是专门的transfer queue（Vertex buffers章节中提到的），那么需要注意：**vkCmdBlitImage必须被提交到与graphics兼容的queue中**。

最后一个参数允许我们指定blit过程中使用的**VkFilter**。在这里，我们有和创建VkSampler时相同的过滤选项。我们**使用VK_FILTER_LINEAR来启用插值**。

```c++
			//将i-1 level转换到SHADER_READ_ONLY_OPTIMAL
			barrier.oldLayout = VK_IMAGE_LAYOUT_TRANSFER_SRC_OPTIMAL;
			barrier.newLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
			barrier.srcAccessMask = VK_ACCESS_TRANSFER_READ_BIT;//等待当前的blit操作完成
			barrier.dstAccessMask = VK_ACCESS_SHADER_READ_BIT;//采样操作会等待这次transition完成

			vkCmdPipelineBarrier(commandBuffer,
				VK_PIPELINE_STAGE_TRANSFER_BIT, VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT, 0,
				0, nullptr,
				0, nullptr,
				1, &barrier);
```

接下来重用该barrier，将i-1 level转换到**VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL**。这次transition会**等待当前的blit操作完成**，所有的**采样操作会等待这次transition完成**。

```c++
			...
			if (mipWidth > 1) mipWidth /= 2;
			if (mipHeight > 1) mipHeight /= 2;
```

在循环的最后，我们将当前mip的尺寸除以2。在除之前，我们检查每个尺寸确保其不会变成0，这可以处理image不是正方形的情况，因为其中一个mip将在另一个维度之前达到1。当一个维度到达了1，该维度会在剩下的levels中都保持1。

```c++
		//将最后一级mip level从VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL转换到VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL
		barrier.subresourceRange.baseMipLevel = mipLevels - 1;
		barrier.oldLayout = VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL;
		barrier.newLayout = VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL;
		barrier.srcAccessMask = VK_ACCESS_TRANSFER_WRITE_BIT;
		barrier.dstAccessMask = VK_ACCESS_SHADER_READ_BIT;

		vkCmdPipelineBarrier(commandBuffer,
			VK_PIPELINE_STAGE_TRANSFER_BIT, VK_PIPELINE_STAGE_FRAGMENT_SHADER_BIT, 0,
			0, nullptr,
			0, nullptr,
			1, &barrier);

		endSingleTimeCommands(commandBuffer);
```

在我们结束command buffer之前，我们需要插入**额外的一个pipeline barrier**。该barrier将**最后一级mip level**从**VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL**转换到**VK_IMAGE_LAYOUT_SHADER_READ_ONLY_OPTIMAL**。这不会在循环中处理，因为最后一级mip level不会作为blit源。

最后，在**createTextureImage**中调用**generateMipmaps**。

```c++
		//将texture image转换到VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL
		transitionImageLayout(textureImage, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_LAYOUT_UNDEFINED, VK_IMAGE_LAYOUT_TRANSFER_DST_OPTIMAL, mipLevels);
		//执行buffer到image的copy函数
		copyBufferToImage(stagingBuffer, textureImage, static_cast<uint32_t>(texWidth), static_cast<uint32_t>(texHeight));
		//在生成完mipmaps之后transition到shader read only
		generateMipmaps(textureImage, texWidth, texHeight, mipLevels);
```

现在，我们的texture image的mipmaps被完全填充了。

#### 4 线性过滤支持 Linear filtering support

使用一个内置函数像vkCmdBlitImage来生成所有的mip levels非常方便，但是不幸的是，它**不保证支持所有平台**。它**需要texture image格式支持线性过滤linaer filtering**，通过**vkGetPhysicalDeviceFormatProperties**可以检查。我们会对**generateMipmaps**函数增加该检查。

增加一个额外的参数指定image的格式。

```c++
	void createTextureImage()
	{
		...
		//在生成完mipmaps之后transition到shader read only
		generateMipmaps(textureImage, VK_FORMAT_R8G8B8A8_SRGB, texWidth, texHeight, mipLevels);
	}

	void generateMipmaps(VkImage image, VkFormat imageFormat, int32_t texWidth, int32_t texHeight, uint32_t mipLevels)
	{
		...
	}
```

在**generateMipmaps**函数中，使用**vkGetPhysicalDeviceFormatProperties**来查询texture image format的所有**properties**。

```c++
	void generateMipmaps(VkImage image, VkFormat imageFormat, int32_t texWidth, int32_t texHeight, uint32_t mipLevels)
	{
		//检查image format是否支持线性过滤
		VkFormatProperties formatProperties;
		vkGetPhysicalDeviceFormatProperties(physicalDevice, imageFormat, &formatProperties);
		...
	}
```

**VkFormatProperties**结构体有3个字段，名为**linearTilingFeature**、**optimalTilingFeatures**和**bufferFeatures**，每个字段根据格式的使用方式描述了如何使用该格式（之前也遇到过）。我们创建的texture image使用了optimal tiling format，所以我们需要检查optimalTilingFeatures。对线性过滤特性的支持可以用过**VK_FORMAT_FEATURE_SAMPLED_IMAGE_FILTER_LINEAR_BIT**来检查。

```c++
		if (!(formatProperties.optimalTilingFeatures & VK_FORMAT_FEATURE_SAMPLED_IMAGE_FILTER_LINEAR_BIT))
		{
			throw std::runtime_error("texture image format does not support linear blitting!");
		}
```

在这种情况下有2个选择。我们可以实现一个函数来寻找支持linear blitting的一个常见texture image format，或者我们可以使用像[stb_image_resize](https://github.com/nothings/stb/blob/master/stb_image_resize.h)这样的库来在软件层面实现mipmap生成。然后可以按照与加载原始image相同的方式将每个mip level加载到image中。

需要注意，在实践中，**任何运行时的mipmap levels生成都是不常见的**（虽然我知道一些哈哈哈，有些降采样可以通过实时mipmap生成代替）。通常，它们会预先生成，并且与base level一起存储在texture文件中以提高加载速度。软件层面实现resizing和从一个文件加载多个levels是留给读者的一个小练习（我是懒狗~）。

#### 5 采样器 Sampler

**VKImage**拥有mipmap数据，**VKSampler**控制在渲染时如何读取数据。Vulkan允许我们指定**minLod**、**maxLod**、**mipLodBias**和**mipmapMode**。当一个texture被采样，sampler会根据以下伪代码选择一个mip level。

```c++
lod = getLodLevelFromScreenSize(); //smaller when the object is close, may be negative
lod = clamp(lod + mipLodBias, minLod, maxLod);

level = clamp(floor(lod), 0, texture.mipLevels - 1);  //clamped to the number of mip levels in the texture

if (mipmapMode == VK_SAMPLER_MIPMAP_MODE_NEAREST) {
    color = sample(level);
} else {
    color = blend(sample(level), sample(level + 1));
}
```

如果**samplerInfo.mipmapMode**是**VK_SAMPLER_MIPMAP_MODE_NEAREST**，lod选择**一个**采样的mip level。如果mipmapMode是**VK_SAMPLER_MIPMAP_MODE_LINEAR**，lod会使用2个mip levels来采样。这些levels被采样，并且其结果会被**线性混合**。

采样操作同样被lod影响。

```c++
if (lod <= 0) {
    color = readTexture(uv, magFilter);
} else {
    color = readTexture(uv, minFilter);
}
```

如果物体离摄像机很近，**magFilter**会用作过滤器。如果物体离摄像机很远，**minFilter**会被使用。正常情况下，lod是**非负**的，并且只会在靠近摄像机的时候为0。**mipLodBias**让我们强制VUlkan来使用比正常情况下更低的lod和level。

为了看到这一章的效果，我们需要为我们的textureSampler设置对应的成员值。我们已经**将minFilter和magFilter设置成VK_FILTER_LINEAR**。所以，我们只需要给**minLod**、**maxLod**、**mipLodBias**和**mipmapMode**赋值。

```c++
	void createTextureSampler()
	{
		...
		//mipmapping相关字段
		samplerInfo.mipmapMode = VK_SAMPLER_MIPMAP_MODE_LINEAR;
		samplerInfo.minLod = 0.0f;
		samplerInfo.maxLod = static_cast<float>(mipLevels);
		samplerInfo.mipLodBias = 0.0f;
		...
	}
```

为了让全范围的mip levels被使用，我们将minLod设置成0.0f，将maxLod设置成mip levels的数量。我们没有理由改变lod值，所以让mipLodBias设置为0.0f。

现在运行程序，会看到以下渲染结果。

![20240218141842](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240218141842.png)

实际上并没有很大的不同，因为我们的场景很简单，仔细观察可以看到细微的差别（比如左边墙壁上报纸的图案）。

![20240218141942](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240218141942.png)

最显眼的区别是纸上的图案。因为mipmaps的原因，图案变得**更平滑**。没有mipmaps，图案有摩尔纹和粗糙的边缘。

我们可以玩一玩sampler的设置来看它们如何影响mipmapping。例如，改变minLod，我们可以让sampler不使用最低的mip levels。

```c++
		samplerInfo.minLod = static_cast<float>(mipLevels / 2);
```

以上设置之后，会渲染出如下结果。

![20240218142302](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240218142302.png)

这就是当物体离远摄像机后，更高级别的mip level被使用的效果。

#### 参考
1. 题图来自画师wlop
2. https://en.wikipedia.org/wiki/Moir%C3%A9_pattern
3. https://github.com/nothings/stb/blob/master/stb_image_resize.h




