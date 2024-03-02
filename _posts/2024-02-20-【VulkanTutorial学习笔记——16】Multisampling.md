---
layout:     post
title:      "【VulkanTutorial学习笔记——16】Multisampling"
subtitle:   ""
date:       2024-02-20 12:00:00
author:     "recaeee"
header-img: "img/in-post/default-bg.jpg"
tags:
    - Vulkan
---
# 【VulkanTutorial学习笔记——16】Multisampling

![20240220111839](https://raw.githubusercontent.com/recaeee/PicGo/main/20240220111839.png)

#### 0 写在前面
本系列为个人在跟随Vulkan Tutorial学习过程中记录的一些笔记，一是防止自己看着看着弃坑，二是希望能在看过之后多少留些印象，三是想在啥时候忘记了能快速地回忆一些Vulkan知识点。

出于以上这些目的，该系列的学习笔记可能会写得较随意一些，并且对于主观判断不重要的东西可能会略过，因此也不保证该系列笔记的学习质量，可能有些东西不经过脑子就会写出来。

我是一个Vulkan的小白，对于Vulkan尚不熟悉，笔记中可能存在错误，或者概念不明确的地方，也希望看的人可以批评斧正~

在这一章，我们会学习如何在Vulkan中启用MSAA，启用MSAA的代码实现是很简单的。我们会从其中接触到一些渲染中的重要概念，如resolve、aliasing、几何锯齿、着色锯齿等等。对于Vulkan中的MSAA实现，会使用到resolveAttachment，这一点还是比较重要的，我们经常可以在开启MSAA的游戏截帧中看到多出的这一张attachment，以前还一直不清楚为什么会有这一张attachment，在学习完这一章之后也做了一定程度的去黑箱化了~关于MSAA的详细原理，网上也有很多技术文章，感兴趣的同学可以很方便地了解到。

原教程链接：
https://vulkan-tutorial.com/Introduction
本章节链接：
https://vulkan-tutorial.com/Multisampling

#### 1 介绍 Introduction

现在我们的程序可以加载多个LOD的textures了，当渲染的物体距离摄像机很远时减少了伪影artifacts。画面现在变得更加平滑，但是仔细观察后，我们会发现绘制的几何图形的**边缘有锯齿状的图案**，尤其是在我们程序早期渲染一个正方形片的时候尤为明显。

![20240218173143](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240218173143.png)

这种不期望的效果叫做“**走样aliasing**”（或者混叠，梦回Games101啊），它是**可用于渲染的像素数量有限**的结果。由于没有分辨率无限大的显示器，因此在某种程度上它总是可见的（毕竟，永远是用离散的采样模拟连续的结果嘛）。有许多的方法来修复该问题，在这一章，我们会着重介绍其中一种比较流行的方法：**多重采样抗锯齿Multisampling anti-aliasing(MSAA)**。（PS：现在感觉是TAA的天下呀~）

在普通的渲染中，像素颜色是根据**单个采样点**确定的，该采样点在大多数情况下是屏幕上**目标像素的中心**。如果绘制的线的一部分经过某个像素，但没有覆盖像素中心的采样点，则将该像素留空，从而导致了锯齿状的“楼梯”效果。

![20240218173916](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240218173916.png)

MSAA做的事情是，**对于每个像素它使用多个采样点来决定该像素的最终颜色**。当然，更多的samples会带来更好的结果，但是计算成本也更高。

![20240218174255](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240218174255.png)

在我们的实现中，我们会关注使用最大可用sample数。取决于我们的程序，这也许不是最好的方法，如果最终结果满足我们的质量要求，为了获取更高的性能，使用更少的samples可能会更好。

#### 2 获取可用样本数 Getting available sample count

首先，让我们**确定硬件可以使用多少个samples**。大部分现代GPUs都支持至少8 samples，但是不保证任何硬件的samples数量都一样。我们会**增加一个新的类成员**来追踪它。

```c++
	//MSAA的samples数量
	VkSampleCountFlagBits msaaSamples = VK_SAMPLE_COUNT_1_BIT;
```

默认情况下，我们会使用每个像素1个sample，等效于没有multisampling，意味着最终渲染结果保持不变。实际确切的**最大samples数**可以通过于我们选择的physical device关联的**VkPhysicalDeviceProperties**中获取。我们使用了一个depth buffer，所以我们必须**同时考虑color和depth的sample数量**。与操作&后支持的最高sample数将是我们可以支持的最大值。增加一个函数来获取该信息。

```c++
	VkSampleCountFlagBits getMaxUsableSampleCount()
	{
		VkPhysicalDeviceProperties physicalDeviceProperties;
		vkGetPhysicalDeviceProperties(physicalDevice, &physicalDeviceProperties);

		VkSampleCountFlags counts = physicalDeviceProperties.limits.framebufferColorSampleCounts & physicalDeviceProperties.limits.framebufferDepthSampleCounts;
		if (counts & VK_SAMPLE_COUNT_64_BIT) { return VK_SAMPLE_COUNT_64_BIT; }
		if (counts & VK_SAMPLE_COUNT_32_BIT) { return VK_SAMPLE_COUNT_32_BIT; }
		if (counts & VK_SAMPLE_COUNT_16_BIT) { return VK_SAMPLE_COUNT_16_BIT; }
		if (counts & VK_SAMPLE_COUNT_8_BIT) { return VK_SAMPLE_COUNT_8_BIT; }
		if (counts & VK_SAMPLE_COUNT_4_BIT) { return VK_SAMPLE_COUNT_4_BIT; }
		if (counts & VK_SAMPLE_COUNT_2_BIT) { return VK_SAMPLE_COUNT_2_BIT; }

		return VK_SAMPLE_COUNT_1_BIT;
	}
```

现在，在physical device选择期间，我们使用该函数设置**msaaSamples**变量。为此，我们需要略微修改**pickPhysicalDevice**函数。

```c++
	//启用一张合适的显卡
	void pickPhysicalDevice()
	{
		...
		//找到合适的显卡
		for (const auto& device : devices)
		{
			if (isDeviceSuitable(device))
			{
				physicalDevice = device;
				//设置最大msaa samples数
				msaaSamples = getMaxUsableSampleCount();
				break;
			}
		}
		...
	}
```

#### 3 设置渲染目标 Setting up a render target

在MSAA中，**每个像素在一个离屏的buffer中采样**，然后渲染到屏幕上。这个新的buffer和我们渲染的常规的images有些许不同——它们必须能够**为每个像素存储多个samples**。multisampled buffer被创建了之后，必须**将其解析resolve为默认framebuffer**（每个像素仅存储一个sample）。这就是为什么我们必须**创建一个额外的render target**，并且修改我们现有的绘制过程。我们只需要一个render target，因为一次只有一个绘制操作处于活动状态，就像depth buffer一样。增加以下类成员。

这里提到了**resolve的概念**，由于我之前一直对该概念比较模糊，因此在这里根据资料了解下resolve的概念。超采样，如SSAA和MSAA相当于以一个比默认framebuffer更大的分辨率渲染场景（实际上增加samples数）），**将更大分辨率的attachment中相邻的像素做一个过滤（比如平均等）得到最终的attachment，这个过程就称为resolve**。其实感觉和降采样的概念很相似，但**resolve是通过显卡的固定硬件实现的**，一般采用的方法是一像素的box过滤器。不同的硬件厂商可能会使用不同的resolve算法，比如NVIDIA的"Quincunx"AA等。随着显卡的不断升级，我们现在还可以通过自定义的shader来做MSAA的resolve。

```c++
	//MSAA render target
	VkImage colorImage;
	VkDeviceMemory colorImageMemory;
	VkImageView colorImageView;
```

这个**新的image会为每个像素存储期望数量的samples**，所以我们需要在image创建过程中，将该数量传递到VkImageCraeteInfo。修改**createImage**函数，**增加一个numSamples参数**。

```c++
	void createImage(uint32_t width, uint32_t height, uint32_t mipLevels, VkSampleCountFlagBits numSamples, VkFormat format, VkImageTiling tiling, VkImageUsageFlags usage, VkMemoryPropertyFlags properties, VkImage& image, VkDeviceMemory& imageMemory)
	{
		...
		imageInfo.samples = numSamples;
		...
	}
```

现在，使用**VK_SAMPLE_COUNT_1_BIT**更新该函数的所有调用，随着实现的进展，我们将会使用合适的值替换它。

```c++
		createImage(swapChainExtent.width, swapChainExtent.height, 1, VK_SAMPLE_COUNT_1_BIT, depthFormat,
			VK_IMAGE_TILING_OPTIMAL, VK_IMAGE_USAGE_DEPTH_STENCIL_ATTACHMENT_BIT,
			VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, depthImage, depthImageMemory);

		//创建textureImage及其内存
		createImage(texWidth, texHeight, mipLevels, VK_SAMPLE_COUNT_1_BIT, VK_FORMAT_R8G8B8A8_SRGB, VK_IMAGE_TILING_OPTIMAL,
			VK_IMAGE_USAGE_TRANSFER_SRC_BIT | VK_IMAGE_USAGE_TRANSFER_DST_BIT | VK_IMAGE_USAGE_SAMPLED_BIT, VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT,
			textureImage, textureImageMemory);
```

现在，我们来**创建一个multisampled color buffer**。增加一个**createColorResources**函数，注意我们会在这里使用msaaSamples作为createImage的传入参数。我们也会使用仅1级mip level，因为Vulkan specification中强制要求**每像素超过1个sample的images的mip level必须为1**。当然，这张color buffer也不需要mipmaps，因为它不会用作texture。

```c++
	void createColorResources()
	{
		VkFormat colorFormat = swapChainImageFormat;

		createImage(swapChainExtent.width, swapChainExtent.height, 1, msaaSamples,
			colorFormat, VK_IMAGE_TILING_OPTIMAL, VK_IMAGE_USAGE_TRANSIENT_ATTACHMENT_BIT | VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT,
			VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, colorImage, colorImageMemory);
		colorImageView = createImageView(colorImage, colorFormat, VK_IMAGE_ASPECT_COLOR_BIT, 1);
	}
```

为了保持一致性，我们在createDepthResources之前调用该函数。

```c++
	//initVulkan函数来用于实例化Vulkan objects私有成员
	void initVulkan()
	{
		...
		createColorResources();
		createDepthResources();
		...
	}
```

现在，我们有一个multisampled color buffer，是时候**处理深度**了。修改**createDepthResources**，更新depth buffer使用的sample数。

```c++
	void createDepthResources()
	{
		...
		createImage(swapChainExtent.width, swapChainExtent.height, 1, msaaSamples, depthFormat,
			VK_IMAGE_TILING_OPTIMAL, VK_IMAGE_USAGE_DEPTH_STENCIL_ATTACHMENT_BIT,
			VK_MEMORY_PROPERTY_DEVICE_LOCAL_BIT, depthImage, depthImageMemory);
		...
	}
```

现在我们创建了一对新的Vulkan资源，所以别忘了在需要的时候**释放它们**。

```c++
	//重建swap chain前清理旧资源
	void cleanupSwapChain()
	{
		//销毁msaa使用的color resources
		vkDestroyImageView(device, colorImageView, nullptr);
		vkDestroyImage(device, colorImage, nullptr);
		vkFreeMemory(device, colorImageMemory, nullptr);
		...
	}
```

更新**recreateSwapChain**让新的color image可以在window resize时创建到正确的分辨率。

```c++
	//当window surface和swap chain不再兼容时重建swap chain
	void recreateSwapChain()
	{
		...
		createColorResources();
		createDepthResources();
		...
	}
```

我们已经完成了最初的MSAA设置，现在我们需要开始在**graphics pipeline**、**framebuffer**、**render pass**中使用这个新资源，并查看渲染结果！

#### 4 增加新attachments Adding new attachments

首先，看向render pass。修改**createRenderPass**，更新color和depth attachment的**createInfo**。

```c++
	//创建Render pass
	void createRenderPass()
	{
		...
		colorAttachment.samples = msaaSamples;
		...
		colorAttachment.finalLayout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;
		...
		depthAttachment.samples = msaaSamples;
		...
	}
```

我们可以注意到我们将colorAttachment的**finalLayout**从VK_IMAGE_LAYOUT_PRESENT_SRC_KHR更改为了**VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL**。这是因为multisampled images不能被直接present。我们首先需要**将其resolve到一个常规的image上**。这个要求不适用于depth buffer，因为它不会在任何时候present。因此，我们只需要增加一个新的color attachment，即所谓的**resolve attachment**。

```c++
		//MSAA resolve到的attachment
		VkAttachmentDescription colorAttachmentResolve{};
		colorAttachmentResolve.format = swapChainImageFormat;
		colorAttachmentResolve.samples = VK_SAMPLE_COUNT_1_BIT;
		colorAttachmentResolve.loadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
		colorAttachmentResolve.storeOp = VK_ATTACHMENT_STORE_OP_STORE;
		colorAttachmentResolve.stencilLoadOp = VK_ATTACHMENT_LOAD_OP_DONT_CARE;
		colorAttachmentResolve.stencilStoreOp = VK_ATTACHMENT_STORE_OP_DONT_CARE;
		colorAttachmentResolve.initialLayout = VK_IMAGE_LAYOUT_UNDEFINED;
		colorAttachmentResolve.finalLayout = VK_IMAGE_LAYOUT_PRESENT_SRC_KHR;
```

现在必须指示render pass**将multisampled color image解析resolve到常规的attachment**。创建一个新的attachment reference，它会**指向用作resolve target的color buffer**。

```c++
		//msaa使用的resolve到的attachment引用
		VkAttachmentReference colorAttachmentResolveRef{};
		colorAttachmentResolveRef.attachment = 2;
		colorAttachmentResolveRef.layout = VK_IMAGE_LAYOUT_COLOR_ATTACHMENT_OPTIMAL;
```

设置subpass结构体的**pResolveAttachments**成员来指向新创建的attachment reference。这足以让render pass定义一个multisample resolve操作，它会让我们将image渲染到屏幕。

```c++
		...
		subpass.pResolveAttachments = &colorAttachmentResolveRef;
		...
```

现在使用这张新的color attachment更新**render pass info**。

```c++
		...
		//创建render pass
		std::array<VkAttachmentDescription, 3> attachments = { colorAttachment, depthAttachment, colorAttachmentResolve };
		...
```

render pass准备完毕后，修改**createFramebuffers**，将新的image view增加到list中。

```c++
	//为swap chain的每个image创建framebuffers
	void createFramebuffers()
	{
			...
			std::array<VkImageView, 3> attachments = {
				colorImageView,
				depthImageView,
				swapChainImageViews[i]
			};
			...
	}
```

最后，通过修改**createGraphicsPipeline**告诉新创建的pipeline使用多个samples。

```c++
	void createGraphicsPipeline()
	{
		...
		multisampling.rasterizationSamples = msaaSamples;
		...
	}
```

现在，运行程序，我们可以看到如下渲染结果。

![20240220104015](https://raw.githubusercontent.com/recaeee/PicGo/main/20240220104015.png)

就像mipmapping一样，和之前的区别可能并不是很明显。仔细观察，我们可以看到边缘不再会有明显锯齿，整个图像看起来比原始的更加平滑。

![20240220104129](https://raw.githubusercontent.com/recaeee/PicGo/main/20240220104129.png)

仔细观察边缘时可以看出一些明显的区别。

![20240220104206](https://raw.githubusercontent.com/recaeee/PicGo/main/20240220104206.png)

#### 5 质量改进 Quality improvements

我们现在的MSAA实现有一些明确的限制，在包含更多细节的场景下会影响输出图像的质量。例如，我们现在不会处理潜在的**shader aliasing**导致的问题（着色锯齿），即**MSAA只会平滑几何的边缘，并不平滑内部填充**。这可能会导致一种情况，我们在屏幕上渲染得到了平滑的几何图形，但是应用的纹理看起来仍然会走样，如果textures包含强对比度的颜色。处理该问题的一种方法是启用[Sample Shading](https://registry.khronos.org/vulkan/specs/1.3-extensions/html/chap27.html#primsrast-sampleshading)，通过额外的性能小号来进一步改善图像的质量。

```c++
	void createLogicalDevice()
	{
		...
		deviceFeatures.sampleRateShading = VK_TRUE;// enable sample shading feature for the device
		...
	}

	void createGraphicsPipeline()
	{
		...
		multisampling.sampleShadingEnable = VK_TRUE;// enable sample shading in the pipeline
		multisampling.minSampleShading = .2f; // min fraction for sample shading; closer to one is smoother
		...
	}
```

在教程中，我们会关闭sample shading，但是在特定场景下它会对质量有明显的改善。

![20240220105011](https://raw.githubusercontent.com/recaeee/PicGo/main/20240220105011.png)

#### 6 小结 Conclusion

到了这一步，我们已经做了很多工作，但是我们现在对一个Vulkan程序有了很好的基础。我们现在掌握的Vulkan基本原理知识应该足以开始探索更多功能，比如：

1. Push constants
2. Instanced rendering
3. Dynamic uniforms
4. Seperate images and sampler descriptors
5. Pipeline cache
6. Multi-threaded command buffer generation
7. Multiple subpasses
8. Compute shaders

现在的程序可以以许多方式拓展，比如增加Blinn-Phong光照、后处理效果、阴影贴图。我们应该在其他APIs中能够学到这些效果如何实现，但许多概念仍然相同。

还剩最后一章啦，泪目~

#### 参考
1. 题图来自画师wlop
2. https://registry.khronos.org/vulkan/specs/1.3-extensions/html/chap27.html#primsrast-sampleshading
3. https://zhuanlan.zhihu.com/p/32823370
4. https://mynameismjp.wordpress.com/2012/10/28/msaa-resolve-filters/
5. https://zhuanlan.zhihu.com/p/619295431





