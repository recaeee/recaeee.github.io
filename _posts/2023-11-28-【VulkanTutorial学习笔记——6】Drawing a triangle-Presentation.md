---
layout:     post
title:      "【VulkanTutorial学习笔记——6】Drawing a triangle-Presentation"
subtitle:   ""
date:       2023-11-28 12:00:00
author:     "recaeee"
header-img: "img/in-post/default-bg.jpg"
tags:
    - Vulkan
---
# 【VulkanTutorial学习笔记——6】Drawing a triangle-Presentation


![20231128205827](https://raw.githubusercontent.com/recaeee/PicGo/main/20231128205827.png)



#### 0 写在前面
本系列为个人在跟随Vulkan Tutorial学习过程中记录的一些笔记，一是防止自己看着看着弃坑，二是希望能在看过之后多少留些印象，三是想在啥时候忘记了能快速地回忆一些Vulkan知识点。

出于以上这些目的，该系列的学习笔记可能会写得较随意一些，并且对于主观判断不重要的东西可能会略过，因此也不保证该系列笔记的学习质量，可能有些东西不经过脑子就会写出来。

我是一个Vulkan的小白，对于Vulkan尚不熟悉，笔记中可能存在错误，或者概念不明确的地方，也希望看的人可以批评斧正~

新手村的第二个任务是！当当当~**Window surface、Swap chain、Image views**，也就是涉及到如何将我们渲染完的画面呈现到屏幕上所需要的一些必要装备，当然这一次的任务难度不小，因此接下来，长文警告！！

原教程链接：
https://vulkan-tutorial.com/Introduction
本章节链接：
https://vulkan-tutorial.com/Drawing_a_triangle/Presentation/Window_surface
https://vulkan-tutorial.com/Drawing_a_triangle/Presentation/Swap_chain
https://vulkan-tutorial.com/Drawing_a_triangle/Presentation/Image_views

#### 1 窗口表面 Window surface

由于Vulkan是个平台无关的API，因此**它不能直接与实际设备上的Window system直接交互**。为了建议Vulkan和Window system之间的连接让Image呈现在屏幕上，我们需要使用WSI(Window System Integration拓展)。在这一章节中，我们第一个要讨论的东西是**VK_KHR_surface拓展**。它暴露了**VkSurfaceKHR**对象，可以用来**表示一个用于呈现Image的Surafece抽象**。在我们的程序中，该Surface将会通过已经创建好的GLFW窗口支持。

**VK_KHR_surface拓展是一个VkInstance层面的拓展**，并且我们已经通过glfwGetRequiredInstanceExtensions开启了它。

Window surface需要在VkInstance创建后立即创建，因为它会影响Physical device的选择。同时值得注意的是，如果我们只需要离屏渲染，我们可以完全不需要创建Window surface，它只是Vulkan的一个可选组件。

#### 2 创建窗口表面 Window surface creation

**尽管VkSurfaceKHR是平台无关的抽象对象，但是它的创建过程并不是平台无关的**，例如在Windows平台它需要HWND和HMODULE。好在glfw提供了一个兼容多平台的创建函数**glfwCreateWindowSurface**帮我们处理好了平台问题，因此我们可以通过它继续写平台无关的代码。当然原教程中也提供了window平台的手动实现，有兴趣的同学可以去看看。

```c++
	//创建window surface，它需要在创建完VkInstance和DebugMessenger后立即创建，因为它会影响Physical device的选择
	void createSurface()
	{
		//直接通过glfw提供的跨平台接口创建surface
		if (glfwCreateWindowSurface(instance, window, nullptr, &surface) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to create window surface!");
		}
	}
```

glfw并没有提供特殊的销毁函数来销毁VkSurfaceKHR，但我们可以简单地调用Vulkan的API来销毁它，记得**要在VkInstance销毁前销毁VkSurfaceKHR**。

#### 3 查询呈现支持 Querying for presentation support

虽然Vulkan可能支持Window system，但是它不意味系统中的任何device（显卡）都支持它。因此我们需要拓展isDeviceSuitable函数来确保设备可以将Images呈现到我们创建的Surface上。因为Presetation是一个对于Queue的特性，因此我们实际要处理的是要**找到一个Queue family支持present我们创建的Surface**。

实际上，一个Queue family只支持绘制指令，而不支持呈现指令是有可能的。因此咱需要修改QueueFamilyIndices结构体，**加入一个的单独用于present的famliy的索引（其实际的familiy可能和graphics是同一个），用于创建Present的Queue**。

```c++
	//我们需要的所有Queue families
	struct QueueFamilyIndices
	{
		//optional允许uint32_t在实际分配值之前让其保持no value，并且可以通过has_value()来查询是否有值
		std::optional<uint32_t> graphicsFamily;
		//用于present image to surface的family
		std::optional<uint32_t> presentFamily;

		//快速判断当前PhysicalDevices是否支持所有我们需要的Queue Families
		bool isComplete()
		{
			return graphicsFamily.has_value() && presentFamily.has_value();
		}
	};
```

我们可以通过**vkGetPhysicalDeviceSurfaceSupportKHR**来判断queue family是否支持present。**注意，Present family和Graphics family很有可能实际上指向同一个famliy，但我们依然将它们视为单独的两个**。

```c++
	//从PhysicalDevice得到我们需要的Queue family，并且可以用来判断physical device是否支持我们需要的所有queue famliy
	QueueFamilyIndices findQueueFamilies(VkPhysicalDevice device)
	{
		QueueFamilyIndices indices;

		...
		
		//我们需要支持Graphics Queue Family
		int i = 0;
		for (const auto& queueFamily : queueFamilies)
		{
			//判断physical device是否支持graphics family
			...
			//判断physical device是否支持present family
			VkBool32 presentSupport = false;
			vkGetPhysicalDeviceSurfaceSupportKHR(device, i, surface, &presentSupport);
			if (presentSupport)
			{
				//注意present family和graphics family实际上很有可能指向同一个family，但我们依然将它们视为单独的两个
				indices.presentFamily = i;
			}

			i++;
		}

		return indices;
	}
```

这里我让GraphicsQueueFamily和PresentQueueFamily的值完全相同了，然后在创建PhysicalDevice的时候Validation提示Error，**The queueFamilyIndex member of each element of pQueueCreateInfos must be unique within pQueueCreateInfos, except that two members can share the same queueFamilyIndex**，咱这里正好是Except情况，所以我就先不管这个Error啦，之后再接这个回旋镖好了。

#### 4 创建呈现队列 Creating the presentation queue

在获取完PresentFamily之后，理所当然我们就接着**创建Present Queue**。由于我们需要同时创建GraphicsQueue和PresentQueue，因此我们改用vector存储queueCreateInfos，并在循环中创建queueCreateInfos。最后将整个vector传递给VkDeviceCreateInfo。

在创建完VkDevice之后，我们仍然需要**获取到Present Queue的handle**，就和Graphics Queue一样。注意，这里如果GraphicsQueueFamily和PresentQueueFamily值相同，那么返回的GraphicsQueue和PresentQueue的Handle值也是相同的，但这并不妨碍咱单独使用它们。

```c++
	void createLogicalDevice()
	{
		//先填充VkDeviceQueueCreateInfo，描述了我们想要的单个Queue family中的Queue数量。
		QueueFamilyIndices indices = findQueueFamilies(physicalDevice);
		// 目前，我们需要Graphics Queue和Present Queue，因此使用Vector存储queueCreateInfos。
		std::vector<VkDeviceQueueCreateInfo> queueCreateInfos;
		std::vector<uint32_t> uniqueQueueFamilies = { indices.graphicsFamily.value(),
			indices.presentFamily.value() };
		float queuePriority = 1.0f;
		//使用一个循环创建所有QueueCreateInfo
		for (uint32_t queueFamily : uniqueQueueFamilies)
		{
			VkDeviceQueueCreateInfo queueCreateInfo{};
			queueCreateInfo.sType = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO;
			queueCreateInfo.queueFamilyIndex = queueFamily;
			queueCreateInfo.queueCount = 1;
			//Vulkan允许我们为Queues分配一个优先级来影响它们的Commandbuffer的执行调度，其值范围为0.0到1.0
			queueCreateInfo.pQueuePriorities = &queuePriority;
			queueCreateInfos.push_back(queueCreateInfo);
		}

	
		//填充需要的设备特性
		...
		//填充VkDeviceCreateInfo
		...
		//填充针对于device的enabled extensions和（被废弃的）ValidationLayers
		...
		//创建VkDevice
		...

		//Queue会在Logical device创建时自动创建，我们需要拿到它的句柄（它也会伴随VkDevice销毁而自动销毁）
		//目前两个Queue family相同，因此返回的两个handle也是相同值的。
		vkGetDeviceQueue(device, indices.graphicsFamily.value(), 0, &graphicsQueue);
		vkGetDeviceQueue(device, indices.presentFamily.value(), 0, &presentQueue);
	}
```

下一节，我们将会探讨Swap chains以及它们呈现images到surface上的过程。

#### 5 交换链 Swap chain

Vulkan并不拥有“默认Framebuffer”的概念，因此它需要一个基础概念来持有一些buffers，这些buffers用来在呈现到屏幕之前存储我们的渲染结果。这个基础概念叫做**交换链Swap chain**。它必须在Vulkan中被显式创建，Swap chain实际上是**一个Images的队列**，这些Images就是等待被呈现到屏幕上的Images。**我们的应用需要这样一张Image用来绘制，绘制完之后返回给Images队列**。这个Images队列如何工作和其中images呈现到屏幕上的条件取决于Swap chain如何被设置，通常Swap chain的**一大作用是将Images的呈现频率和屏幕刷新率同步**。（长难文好难翻译啦~）

#### 6 检查交换链支持 Checking for swap chain support

不是所有的显卡都支持直接将Images呈现到屏幕上，例如为服务器设计的没有显示输出的显卡。另外，因为Images presentation和window system、surfaces紧密关联，它并不属于Vulkan core的一部分。我们必须在查询交换链支持后开启VK_KHR_swapchain这一device extension。

出于以上原因，我们首先**在isDeviceSuitable函数中检查是否支持Vk_KHR_swapchain这一device extension**。（回旋镖接到一个！我们要在创建Physical device的时候填充要启用的device extensions了。）咱通过VK_KHR_SWAPCHAIN_EXTENSION_NAME宏代替VK_KHR_swapchain，该宏会帮我们检查拼写错误。

检查Physical device是否支持我们需要的Device extensions（VK_KHR_swapchain）的代码如下。

```c++
	//所有要启用的device extensions
	const std::vector<const char*> deviceExtensions = {
		VK_KHR_SWAPCHAIN_EXTENSION_NAME
	};

	//判断该显卡是否支持我们想要的功能
	bool isDeviceSuitable(VkPhysicalDevice device)
	{
		QueueFamilyIndices indices = findQueueFamilies(device);

		bool extensionsSupported = checkDeviceExtensionSupport(device);

		return indices.isComplete() && extensionsSupported;
	}

		//判断Physical device是否支持我们需要开启的device extensions
	bool checkDeviceExtensionSupport(VkPhysicalDevice device)
	{
		//获取physical device支持的所有device extensions
		uint32_t extensionCount;
		vkEnumerateDeviceExtensionProperties(device, nullptr, &extensionCount, nullptr);
		std::vector<VkExtensionProperties> availableExtensions(extensionCount);
		vkEnumerateDeviceExtensionProperties(device, nullptr, &extensionCount, availableExtensions.data());

		//检查是否所有需要的device extensions都支持
		std::set<std::string> requiredExtensions(deviceExtensions.begin(), deviceExtensions.end());
		for (const auto& extension : availableExtensions)
		{
			requiredExtensions.erase(extension.extensionName);
		}

		return requiredExtensions.empty();
	}
```

好了，代码其实并不难，接下来运行下程序看看咱的显卡是否支持SwapChain拓展。需要注意的是，**可以得到Presentation queue就暗示了swap chain拓展是被支持的**，但是这里我们仍然显式去做这些事。

#### 7 启用设备拓展 Enabling device extensions

使用SwapChian需要开启VK_KHR_swapchain，我们**在创建Physical device的时候检查了其是否支持该拓展**，接下来，我们**在创建Logical device的时候真正启用这些拓展**。

```c++
		//填充针对于device的enabled extensions和（被废弃的）ValidationLayers
		createInfo.enabledExtensionCount = static_cast<uint32_t>(deviceExtensions.size());
		createInfo.ppEnabledExtensionNames = deviceExtensions.data();
```

#### 8 查询交换链支持的细节 Querying details of swap chain support

只检查swap chain是否可以被获取是不足够的，因为它**可能和我们的window system不兼容**。创建一个Swap chain除了VkInstance和Device的创建还需要更多的设置，所以我们需要查询一些细节。

我们需要检查最基础的**三种属性**：
1. **Basic surface capabilities**（swap chain中images的最小最大数量，images的最小最大长宽）。
2. **Surface formats**（像素格式，颜色空间）。
3. **Available presentation modes**。

检查的方式比较简单，直接上代码，调用的几个函数**前两个参数基本都是Physical device和surface**，因为它俩是Swapchain的重要组成部分。

```c++
	//检查physical device的swap chain支持的属性
	SwapChainSupportDetails querySwapChainSupport(VkPhysicalDevice device)
	{
		SwapChainSupportDetails details;
		//Basic surface capabilities
		vkGetPhysicalDeviceSurfaceCapabilitiesKHR(device, surface, &details.capabilities);
		//Surface formats
		uint32_t formatCount;
		vkGetPhysicalDeviceSurfaceFormatsKHR(device, surface, &formatCount, nullptr);
		if (formatCount != 0)
		{
			details.formats.resize(formatCount);
			vkGetPhysicalDeviceSurfaceFormatsKHR(device, surface, &formatCount, details.formats.data());
		}
		//Available presentation modes
		uint32_t presentModeCount;
		vkGetPhysicalDeviceSurfacePresentModesKHR(device, surface, &presentModeCount, nullptr);
		if (presentModeCount != 0)
		{
			details.presentModes.resize(presentModeCount);
			vkGetPhysicalDeviceSurfacePresentModesKHR(device, surface, &presentModeCount, details.presentModes.data());
		}

		return details;
	}

		//判断该显卡是否支持我们想要的功能
	bool isDeviceSuitable(VkPhysicalDevice device)
	{
		...

		bool swapChainAdequate = false;
		if (extensionsSupported)
		{
			SwapChainSupportDetails swapChainSupport = querySwapChainSupport(device);
			//在该教程中，我们需要至少一个支持的image format和一个presentation mode。
			swapChainAdequate = !swapChainSupport.formats.empty() && !swapChainSupport.presentModes.empty();
		}

		return indices.isComplete() && extensionsSupported && swapChainAdequate;
	}
```

#### 9 选择交换链的正确设置 Choosing the right settings for the swap chain

如果swapChainAdequate条件满足，那么我们就已经可以创建swap chain了，但是我们仍然能做一些优化。我们将会写一些函数来**寻找可用的swap chain设置中最佳的那一套设置**，主要由以下三个条件作为判断标准：
1. **Surface format**（颜色、深度）
2. **Presentation mode**（交换Images的模式）
3. **Swap extent**（swap chain中images的分辨率）

对于这三个衡量指标，我们有一套理想值，如果达不到这些理想值，再寻找一些fallback。

**首先来选择Surface format**。每个**VkSurfaceFormatKHR**包含一个**format**和一个**colorSpace**。**format确定了颜色通道和类型**，例如VK_FORMAT_B8G8R8A8_SRGB，意义很简单啦，不解释了。**colorspace表示SRGB颜色空间是否支持**。

对于颜色空间，**如果SRGB可用我们就使用SRGB**，因为它更符合人的感知。它也几乎是大部分Images的标准颜色空间，比如Textures。因此，我们也需要使用支持SRGB的Surface format，最常见的是**VK_FORMAT_B8G8R8A8_SRGB**。

```c++
	//选择Swap chain最佳的surface format
	VkSurfaceFormatKHR chooseSwapSurfaceFormat(const std::vector<VkSurfaceFormatKHR>& availableFormats)
	{
		//优先使用VK_FORMAT_B8R8G8A8_SRGB
		for (const auto& availableFormat : availableFormats)
		{
			if (availableFormat.format == VK_FORMAT_B8G8R8A8_SRGB && availableFormat.colorSpace == VK_COLOR_SPACE_SRGB_NONLINEAR_KHR)
			{
				return availableFormat;
			}
		}

		//fallback使用第一个支持的格式
		return availableFormats[0];
	}
```

**接下来选择Presentation mode**。Presentation mode可以说是swap chain中最重要的设置，因为它**代表了实际展现Images到屏幕的条件**。在Vulkan中，有4个可能的模式：
1. **VK_PRESENT_MODE_IMMEDIATE_KHR**：应用提交的Images会被立刻传输到屏幕，可能会造成撕裂。
2. **VK_PRESENT_MODE_FIFO_KHR**：swap chain是一个**先进先出的Images队列**，每当屏幕刷新，从队头取出一张Images；应用每渲染完一张Images，将其插入到队尾。它与现代游戏中的垂直同步类似。**显示刷新的瞬间称为垂直空白“Vertical blank”**。
3. **VK_PRESENT_MODE_FIFO_RELAXED_KHR**：这个模式和上一个模式只有一个不同点，如果应用延迟了，在最后一次Vertical blank时队列为空时，在下一张Image进队时会立刻把它传输到屏幕而不是等待下一个Verticle blank，可能会造成画面撕裂。
4. **VK_PRESENT_MODE_MAILBOX_KHR**：这是第二个模式的一个变体。当队列已满时，在下一个Images到达时，会立刻出队将新的Image传输到屏幕上。这个模式可以用于尽快渲染画面，同时避免画面撕裂，比起垂直同步会有更少的延迟问题。它通常被叫做"**三重缓冲Triple buffering**"，但三重缓冲也并不意味着帧率不被锁定。

在这4个模式中，**只有VK_PRESENT_MODE_FIFO_KHR是一定可用的**，所以我们写个函数来寻找更好的可用模式。

在**不考虑功耗**的情况下，推荐**优先使用VK_PRESENT_MODE_MAILBOX_KHR**，少撕裂、低延迟，用过了都说好。但是在功耗受限的移动端，可能还是推荐使用VK_PRESENT_MODE_FIFO_KHR。

```c++
//选择Swap chain最佳的Presentation mode
	VkPresentModeKHR chooseSwapPresentMode(const std::vector<VkPresentModeKHR>& availablePresentModes)
	{
		//优先使用VK_PRESENT_MODE_MAILBOX_KHR三重缓冲
		for (const auto& availablePresentMode : availablePresentModes)
		{
			if (availablePresentMode == VK_PRESENT_MODE_MAILBOX_KHR)
			{
				return availablePresentMode;
			}
		}
		return VK_PRESENT_MODE_FIFO_KHR;
	}
```

**接下来选择最佳的Swap extent**。**Swap extent决定了swap chain的images的分辨率，它大部分情况来说都等于window的分辨率（以pixel为单位）**。我们可以通过**VkSurfaceCapabilitiesKHR**结构体来得到可用的分辨率范围，并且可以从它的**currentExtent成员**得到目前Vulkan自动匹配的分辨率。那我们要选择的分辨率一定是**优先匹配window分辨率**的，我们会遇到两种情况：
1. currentExtent返回的宽高为实际window的宽高，这个时候咱就用这个宽高就行啦，Vulkan默认匹配到了window的分辨率，咱也改不了。
2. currentExtent返回的宽高为uint32_t的无穷大值，这个时候意味着window manager告诉我们swap chain的分辨率是可调的，这个时候我们通过glfw的接口获得window以pixel为单位的分辨率，并将swap chain的分辨率在可用范围内尽可能匹配该值。（**注意，Vulkan中的分辨率是以pixel为单位的，而glfw中的分辨率有两种，以pixel为单位和以屏幕坐标为单位**。

```c++
	//选择Swap chain最佳的Swap extent
	VkExtent2D chooseSwapExtent(const VkSurfaceCapabilitiesKHR& capabilities)
	{
		//capabilities的currentExtent成员是Vulkan返回的宽高用于匹配window的分辨率
		//但如果其值为uint32_t的最大值，意味着其允许我们主动设置成将swap chain设置成其他分辨率
		if (capabilities.currentExtent.width != std::numeric_limits<uint32_t>::max())
		{
			//意味着不允许我们主动改swap chain分辨率，一定要适配window
			return capabilities.currentExtent;
		}
		else
		{
			//Vulkan的分辨率以pixel为单位，而不是屏幕坐标
			int width, height;
			//获取window以pixel为单位的分辨率
			glfwGetFramebufferSize(window, &width, &height);
			
			VkExtent2D actualExtent = {
				static_cast<uint32_t>(width),
				static_cast<uint32_t>(height)
			};
			//在可用范围内尽可能贴近window分辨率
			actualExtent.width = std::clamp(actualExtent.width, capabilities.minImageExtent.width, capabilities.maxImageExtent.width);
			actualExtent.height = std::clamp(actualExtent.height, capabilities.minImageExtent.height, capabilities.maxImageExtent.height);

			return actualExtent;
		}
	}
```

#### 10 创建交换链 Creating the swap chain

现在，我们拥有了所有帮助我们运行时做选择的函数，最后我们用这些信息来创建实际工作的swap chain。

除了以上属性，创建swap chain的过程中，我们需要**指定其中image的数量**，我们使用最小数量+1以确保一定的并行度。创建Swap chain的过程和创建其他Vulkan object类似，需要填充一个庞大的createInfo。在其中我们还需要填写swapchain的**imageArrayLayers**，它用于**确定每张Image组成layer的数量**，非3D情况下应该一直为1。**imageUsage**用于**确定swap chain的使用目的**，在教程中我们会将直接在swap chain提供的image上进行渲染，因此使用**VK_IMAGE_USAGE_TRANSFER_DST_BIT**。如果我们希望渲染在单独的一张image来执行一些后处理，那我们会选择将swap chain的imageUsage设置为VK_IMAGE_USAGE_TRANSFER_DST_BIT，并且使用内存操作来将渲染完的image传输到swap chain image上。

我们还需要**确定swap chain在多Queue families情况下的工作模式**，一共有两种模式：
1. **VK_SHARING_MODE_EXCLUSIVE**：一个Image同一时间只属于一个Queue family，在其他queue family使用前，拥有权需要显式转移，该模式性能最佳。
2. **VK_SHARING_MODE_CONCURRENT**：Images可以被多queue families同时使用，不需要显式转移。

咱们在Graphics和Present的Queue family相同的情况下使用VK_SHARING_MODE_EXCLUSIVE，否则使用VK_SHARING_MODE_CONCURRENT。

另外我们还需要**确定Swap chain中的image被应用的变换preTransform**，我们使用默认值不对其变换（通常会有旋转90°或者水平翻转之类的）。

另另外，我们需要**确定Swap chain的presentMode**，这个之前说过啦。另另另外，我们需要**确定clipped值**，通常为VK_TRUE，其意味着我们不关心被其他window遮挡的像素，会被裁剪掉。

最最最后，我们需要**确定oldSwapchain**。在Vulkan运行时，swap chain可能变得非法或者非最优，比如window被resize。在这种情况下，swap chain需要重建，并且我们需要在重建时明确老的swap chain的引用。（突然感觉好累是怎么回事（）

接下来真正创建swap chain啦，记得在cleanup中主动销毁它。

```c++
	//创建Swap chain
	void CreateSwapChain()
	{
		SwapChainSupportDetails swapChainSupport = querySwapChainSupport(physicalDevice);

		VkSurfaceFormatKHR surfaceFormat = chooseSwapSurfaceFormat(swapChainSupport.formats);
		VkPresentModeKHR presentMode = chooseSwapPresentMode(swapChainSupport.presentModes);
		VkExtent2D extent = chooseSwapExtent(swapChainSupport.capabilities);

		//确定swap chain中的Images数量，至少多一张以确保并行度
		uint32_t imageCount = swapChainSupport.capabilities.minImageCount + 1;
		//确保数量不要超过最大数量（maxImageCount为0意味着没上限。
		if (swapChainSupport.capabilities.maxImageCount > 0 && imageCount > swapChainSupport.capabilities.maxImageCount)
		{
			imageCount = swapChainSupport.capabilities.maxImageCount;
		}

		//创建Swap chain
		VkSwapchainCreateInfoKHR createInfo{};
		createInfo.sType = VK_STRUCTURE_TYPE_SWAPCHAIN_CREATE_INFO_KHR;
		createInfo.surface = surface;
		createInfo.minImageCount = imageCount;
		createInfo.imageFormat = surfaceFormat.format;
		createInfo.imageColorSpace = surfaceFormat.colorSpace;
		createInfo.imageExtent = extent;
		createInfo.imageArrayLayers = 1; // 确定了每一个image组成的layer数量
		createInfo.imageUsage = VK_IMAGE_USAGE_COLOR_ATTACHMENT_BIT; // 确定swap chain的使用目的

		//明确swap chain images在多queue families情况下的使用
		QueueFamilyIndices indices = findQueueFamilies(physicalDevice);
		uint32_t queueFamilyIndices[] = { indices.graphicsFamily.value(), indices.presentFamily.value() };

		if (indices.graphicsFamily != indices.presentFamily)
		{
			//如果graphics和present为两个queue family，则使用CONCURRENT模式
			createInfo.imageSharingMode = VK_SHARING_MODE_CONCURRENT;
			createInfo.queueFamilyIndexCount = 2;
			createInfo.pQueueFamilyIndices = queueFamilyIndices;
		}
		else
		{
			//否则使用EXCLUSIVE模式，swap chain同一时间只属于一个queue family
			createInfo.imageSharingMode = VK_SHARING_MODE_EXCLUSIVE;
			createInfo.queueFamilyIndexCount = 0; // Optional
			createInfo.pQueueFamilyIndices = nullptr; // Optional
		}
		//确定images被应用的变换（通常为旋转90°、水平翻转等），我们不需要任何变换
		createInfo.preTransform = swapChainSupport.capabilities.currentTransform;
		//确定和window system中其他window的透明度混合模式
		createInfo.compositeAlpha = VK_COMPOSITE_ALPHA_OPAQUE_BIT_KHR;
		//确定present mode
		createInfo.presentMode = presentMode;
		//确定被其他window遮挡的像素模式
		createInfo.clipped = VK_TRUE;
		//确定老的swap chain
		createInfo.oldSwapchain = VK_NULL_HANDLE;

		if (vkCreateSwapchainKHR(device, &createInfo, nullptr, &swapChain) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to create swap chain!");
		}
	}
```

现在运行下游戏确保swap chain被成功创建。我们可以主动注释掉createInfo.imageExtent = extent，然后可以看到Validation layers会主动报错。

#### 11 检索交换链图像 Retrieving the swap chain images

Swap chain现在被成功创建了，剩下的工作就是拿到其中的**VkImage**们的handle了。

这些**Images在Swapchain创建时会被自动创建，同样在Swapchain销毁时会被自动销毁**，因此我们不需要任何cleanup代码。获取handle的方式老生常谈了，值得注意的是我们在创建swap chain的时候只确定了images的最小数量，实际数量可能更多。

```c++
		//创建完swap chain后存储Images的handle
		//注意我们之前设置的imageCount只是确定swapchain中image的最小数量，实际可能会更多，因此需要重新取值
		vkGetSwapchainImagesKHR(device, swapChain, &imageCount, nullptr);
		swapChainImages.resize(imageCount);
		vkGetSwapchainImagesKHR(device, swapChain, &imageCount, swapChainImages.data());
		//存储format和extent
		swapChainImageFormat = surfaceFormat.format;
		swapChainExtent = extent;
```

我们现在有了一个Images集合用于绘制，同时它们可以被呈现到window上。下一节将会讨论我们如何将这些images设置为render targets，然后我们将会实践graphics pipeline和drawing commoands！

#### 12 Image views

为了使用swap chain中的VkImage,在Render pipeline中我们必须创建**VkImageView**对象。一个ImageView就相当于对Image的观察（ImageView实在不知道译成什么了）。**它描述了如何访问Image和能访问的区域**，例如一个ImageView描述了一个VkImage应该被视作一个不带任何mipmap等级的2D深度纹理。

在本教程中，我们将会为swap chain中的VkImages写一个**createImageViews**函数来给每个VkImage创建一个基本的ImageView，由此我们可以将它们用于Color targets。

具体代码如下，在其createInfo中，**viewType**和**format**字段指定如何解释图像数据。viewType包括1D、2D、3D纹理和贴图。**components**字段允许我们混合颜色通道，可以实现不同通道映射，例如可以对一个单通道纹理将其4个通道都映射到R通道上，在这里我们使用默认映射。**subsresourceRange**描述Image的用途和可以访问image的哪些部分，我们在这里将其作为Color target，并且无Mipmap，只包含一层layer。

```c++
	//为swap chain中每个VkImage创建ImageView
	void createImageViews()
	{
		swapChainImageViews.resize(swapChainImages.size());
		for (size_t i = 0; i < swapChainImages.size(); i++)
		{
			VkImageViewCreateInfo createInfo{};
			createInfo.sType = VK_STRUCTURE_TYPE_IMAGE_VIEW_CREATE_INFO;
			//对应的Image
			createInfo.image = swapChainImages[i];
			//viewType和format字段指定如何解释图像数据
			//viewType包括1D、2D、3D纹理和贴图
			createInfo.viewType = VK_IMAGE_VIEW_TYPE_2D;
			createInfo.format = swapChainImageFormat;
			//components字段允许我们混合颜色通道，可以实现不同通道映射，在这里我们使用默认映射
			createInfo.components.r = VK_COMPONENT_SWIZZLE_IDENTITY;
			createInfo.components.g = VK_COMPONENT_SWIZZLE_IDENTITY;
			createInfo.components.b = VK_COMPONENT_SWIZZLE_IDENTITY;
			createInfo.components.a = VK_COMPONENT_SWIZZLE_IDENTITY;
			//subsresourceRange描述Image的用途和可以访问image的哪些部分，我们在这里将其作为Color target
			createInfo.subresourceRange.aspectMask = VK_IMAGE_ASPECT_COLOR_BIT;
			createInfo.subresourceRange.baseMipLevel = 0;
			createInfo.subresourceRange.levelCount = 1;
			createInfo.subresourceRange.baseArrayLayer = 0;
			createInfo.subresourceRange.layerCount = 1;

			//创建ImageView
			if (vkCreateImageView(device, &createInfo, nullptr, &swapChainImageViews[i]) != VK_SUCCESS)
			{
				throw std::runtime_error("failed to create image views!");
			}
		}
	}
```

不像Images，由于ImageViews是我们手动显式创建的，因此我们需要在cleanup中手动销毁它们。

```c++

	//一旦window被关闭，我们将在**cleanup**函数中确保释放我们用到的所有资源
	void cleanup()
	{
		//销毁swap chain对应的image views，image view是我们手动创建的，因此也要手动销毁
		for (auto imageView : swapChainImageViews)
		{
			vkDestroyImageView(device, imageView, nullptr);
		}
		...
	}
```

一个ImageView足以让我们将一个Image作为一张纹理来使用，但是它不足与让我们将一个Image用作Render target。我们需要一个中间步骤，称为**framebuffer**，但是首先我们需要建立**graphics pipeline**。（呼，我们已经从lv1升级到了lv5了，接下来要对战pipeline了！）

#### 参考
1. https://zhuanlan.zhihu.com/p/555509777
2. 题图来自画师wlop