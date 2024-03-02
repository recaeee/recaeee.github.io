---
layout:     post
title:      "【VulkanTutorial学习笔记——9】Drawing a triangle-Swap chain recreation"
subtitle:   ""
date:       2024-02-06 12:00:00
author:     "recaeee"
header-img: "img/in-post/default-bg.jpg"
tags:
    - Vulkan
---
# 【VulkanTutorial学习笔记——9】Drawing a triangle-Swap chain recreation

![20240202180839](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240202180839.png)

#### 0 写在前面
本系列为个人在跟随Vulkan Tutorial学习过程中记录的一些笔记，一是防止自己看着看着弃坑，二是希望能在看过之后多少留些印象，三是想在啥时候忘记了能快速地回忆一些Vulkan知识点。

出于以上这些目的，该系列的学习笔记可能会写得较随意一些，并且对于主观判断不重要的东西可能会略过，因此也不保证该系列笔记的学习质量，可能有些东西不经过脑子就会写出来。

我是一个Vulkan的小白，对于Vulkan尚不熟悉，笔记中可能存在错误，或者概念不明确的地方，也希望看的人可以批评斧正~

这一章比较简短（考虑到和后续内容的区分，还是单独分为了一章），主要是完善我们的三角形绘制程序，考虑一些窗口变动等情况下swap chain重建的情况。在这一章之后，我们的vulkan程序变得足够健壮，然后从下一章开始，我们会开始学习使用各种buffer资源了（真是越来越好了呢~）

原教程链接：
https://vulkan-tutorial.com/Introduction
本章节链接：
https://vulkan-tutorial.com/Drawing_a_triangle/Swap_chain_recreation

#### 1 重建Swap chain Recreating the swap chain

现在我们的程序可以成功地绘制三角形了，但是目前它还会有一些情况无法处理。**当window surface改变时，导致swap chain不再兼容它是有可能的**。可能导致这种情况的一个成因是**window的尺寸改变**了。我们必须捕获到这些事件，并且重建swap chain。

创建一个**recreateSwapChain**函数，调用**createSwapChain**和所有依赖swap chain或window尺寸的创建函数。

```c++
	//当window surface和swap chain不再兼容时重建swap chain
	void recreateSwapChain()
	{
		vkDeviceWaitIdle(device);

		createSwapChain();
		createImageViews();
		createFramebuffers();
	}
```

首先调用**vkDeviceWaitIdle**，因为正如上一章一样，我们不能接触可能正在被使用的资源。显然，我们需要重建**swap chain**本身。**images views**需要被重建，因为它们时基于swap chain images创建的（即swap chain images的句柄）。最后，**framebuffers**直接依赖于swap chain images，因此也需要被重建。

为了**保证在重建它们之前让相应的旧资源被清理**，我们需要移动一些cleanup代码到一个单独的函数中，然后在recreateSwapChain中调用，称之为**cleanupSwapChain**。

```c++
	//重建swap chain前清理旧资源
	void cleanupSwapChain()
	{

	}

	//当window surface和swap chain不再兼容时重建swap chain
	void recreateSwapChain()
	{
		vkDeviceWaitIdle(device);

		cleanupSwapChain();

		createSwapChain();
		createImageViews();
		createFramebuffers();
	}
```

需要注意，为了简单起见，我们不会在此处重新创建render pass。理论上，**swap chain image的格式可能在程序生命周期内发生变化**，比如将一个window从SDR的显示屏上移动到一个HDR的显示屏上。这也许会要求程序重建render pass来保证正确表现动态范围（SDR和HDR）之间的变化。

我们把重建的所有这些对象的cleanup代码从cleanup移动到cleanupSwapChain。

```c++
	//重建swap chain前清理旧资源
	void cleanupSwapChain()
	{
		//销毁framebuffers，需要在renderpass、imageviews销毁前、一帧绘制完后销毁
		for (auto framebuffer : swapChainFramebuffers)
		{
			vkDestroyFramebuffer(device, framebuffer, nullptr);
		}
		//销毁swap chain对应的image views，image view是我们手动创建的，因此也要手动销毁
		for (auto imageView : swapChainImageViews)
		{
			vkDestroyImageView(device, imageView, nullptr);
		}
		//销毁Swap chain
		vkDestroySwapchainKHR(device, swapChain, nullptr);
	}

	//一旦window被关闭，我们将在**cleanup**函数中确保释放我们用到的所有资源
	void cleanup()
	{
		cleanupSwapChain();

		//销毁pipeline
		vkDestroyPipeline(device, graphicsPipeline, nullptr);
		//销毁pipeline layout
		vkDestroyPipelineLayout(device, pipelineLayout, nullptr);

		//销毁render pass
		vkDestroyRenderPass(device, renderPass, nullptr);

		//当程序结束，并且所有commands都完成了，并且没有更多的同步被需要时，semaphores和fence需要被销毁
		for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHTS; i++)
		{
			vkDestroySemaphore(device, imageAvailableSemaphores[i], nullptr);
			vkDestroySemaphore(device, renderFinishedSemaphores[i], nullptr);
			vkDestroyFence(device, inFlightFences[i], nullptr);
		}

		//销毁command pool
		vkDestroyCommandPool(device, commandPool, nullptr);
		
		//销毁VkDevice
		vkDestroyDevice(device, nullptr);
		//销毁DebugUtilsMessenger
		if (enableValidationLayers)
		{
			DestroyDebugUtilsMeseengerEXT(instance, debugMessenger, nullptr);
		}
		//销毁Window surface
		vkDestroySurfaceKHR(instance, surface, nullptr);
		//VkInstance只应该在程序退出前一刻销毁，所有其他的Vulkan资源都应该在instance销毁前释放
		vkDestroyInstance(instance, nullptr);
		//销毁window
		glfwDestroyWindow(window);
		//结束GLFW本身
		glfwTerminate();
	}
```

注意到在创建Swapchain的函数的**chooseSwapExtent**中，我们已经询问了新窗口的分辨率，来保证swap chain images拥有正确的尺寸，所以没有必要修改chooseSwapExtent。（记住在创建swap chain时，我们已经使用**glfwGetFramebufferSize**来获取surface的以像素为单位的分辨率）

这就是重建swap chain的所有工作！但是，这个方法的**缺点是我们在创建新swap chain之前需要暂停所有的渲染**。**在旧swap chain的image上绘制命令进行的同时创建新的swap chian是有可行的**。我们需要将旧的swap chain传入**VkSwapchainCreateInfoKHR**中的**oldSwapChain**字段，然后在完成使用它后立即销毁旧的swap chain。

#### 2 次优或过时的swap chain Suboptimal or out-of date swap chain

现在我们只需要弄清楚什么时候需要重建swap chain，然后调用**recreateSwapChain**函数。幸运的是，Vulkan通常只会告诉我们**swap chain在presentation的过程中不足够**。**vkAcquireNextImageKHR**和**vkQueuePresentKHR**函数可以返回表示这个情况的特殊值。

**VK_ERROR_OUT_OF_DATE_KHR**：swap chain已经不再和surface兼容，并且不能再用于渲染。通常发生在window resize。

**VK_SUBOPTIMAL_KHR**：swap chain仍然可以成功用于present，但是surface属性不再完全匹配。

```c++
		//swap chain过时的话重建
		if (result == VK_ERROR_OUT_OF_DATE_KHR)
		{
			//swap chain已经不再和surface兼容，并且不能再用于渲染。通常发生在window resize。
			recreateSwapChain();
			return;
		}
		else if (result != VK_SUCCESS && result != VK_SUBOPTIMAL_KHR)
		{
			throw std::runtime_error("failed to acquire swap chain image!");
		}
```

当尝试获取image时，如果swap chain变成out of date，那么它不再能用于present。因此我们立刻重建swap chain并且在下一次drawFrame调用中重新尝试。

我们也可以在swap chain变成suboptimal时做这件事，但是在这种情况下我们选择继续执行下去，因为我们已经得到了一张image。**VK_SUCCESS**和**VK_SUBOPTIMAL_KHR**都被认为是success。

```c++
		//向swap chain提交一个present an image的请求
		result = vkQueuePresentKHR(presentQueue, &presentInfo);

		if (result == VK_ERROR_OUT_OF_DATE_KHR || result == VK_SUBOPTIMAL_KHR)
		{
			recreateSwapChain();
		}
		else if (result != VK_SUCCESS)
		{
			throw std::runtime_error("failed to present swap chain image!");
		}

		//递进到下一帧
		currentFrame = (currentFrame + 1) % MAX_FRAMES_IN_FLIGHTS;
```

**vkQueuePresentKHR**函数也返回同样意义的result。在这个情况下，当swap chain变成suboptimal时我们也重建swap chain，因为我们想要最佳的结果。

#### 3 修复一个死锁 Fixing a deadlock

如果我们现在尝试运行代码，我们可能会遇到一个死锁。调试代码可以发现，程序到达了vkWaitForFences但是不再继续过去。这是因为当vkAcquireNextImageKHR返回了VK_ERROR_OUT_OF_DATE_KHR，我们重建了swap chain然后从drawFrame中直接返回。但是在它发生前，当前帧的fence已被等待并且重置为unsignaled（这里回顾下，waitForFence会等待fence被激活为signal时才会通过）。因为我们立即返回了，没有工作被提交去执行，因此fence永远不会被激活，造成了vkWaitForFences永远在等待。

好在我们可以非常简单地修复它，**将fence的重置推迟到我们可以确认将会submit工作**。因此，如果我们提前返回了，fence也会是被激活的，vkWaitForFences不会再下一次被用到的时候遇到死锁。

drawFrame开头的代码如下：

```c++
		//等待上一帧完成
		vkWaitForFences(device, 1, &inFlightFences[currentFrame], VK_TRUE, UINT64_MAX);
		
		//从swap chain中获取一张image
		uint32_t imageIndex;
		VkResult result = vkAcquireNextImageKHR(device, swapChain, UINT64_MAX, imageAvailableSemaphores[currentFrame], VK_NULL_HANDLE, &imageIndex);
		//swap chain过时的话重建
		if (result == VK_ERROR_OUT_OF_DATE_KHR)
		{
			//swap chain已经不再和surface兼容，并且不能再用于渲染。通常发生在window resize。
			recreateSwapChain();
			return;
		}
		else if (result != VK_SUCCESS && result != VK_SUBOPTIMAL_KHR)
		{
			throw std::runtime_error("failed to acquire swap chain image!");
		}

		//在确保我们会执行submit之后再重置fence到unsignaled
		vkResetFences(device, 1, &inFlightFences[currentFrame]);
```

#### 4 显式处理resize Handling resizes explicitly

虽然许多驱动和平台会在window resize之后自动触发VK_ERROR_OUT_OF_DATE_KHR，但是它并不能保证一定发生。所以我们将会增加一些额外的代码来显式地处理resize。首先增加一个成员变量标识一个resize发生了。

```c++
	//存储保证一次只渲染一帧的fence
	std::vector<VkFence> inFlightFences;
	//当前帧索引
	uint32_t currentFrame = 0;
	//标识一个window resize发生的flag
	bool framebufferResized = false;
```

**drawFrame**函数应该进行如下修改，需要检查该flag。

```c++
		if (result == VK_ERROR_OUT_OF_DATE_KHR || result == VK_SUBOPTIMAL_KHR || framebufferResized)
		{
			framebufferResized = false;
			recreateSwapChain();
		}
		else if (result != VK_SUCCESS)
		{
			throw std::runtime_error("failed to present swap chain image!");
		}
```

在**vkQueuePresentKHR**函数之后做这个是很重要的，因为需要确保semaphores处于一致状态，否则可能永远不会正确等待一个signaled的semapore。现在为了实际测试resize，我们可以使用GLFW框架中的**glfwSetFramebufferSizeCallback**函数来设置一个回调。

```c++
	//始化GLFW并且创建一个window
	void initWindow()
	{
		//初始化GLFW库
		glfwInit();

		//由于GLFW最开始是被用于创建OpenGL context的，所以我们需要告诉它不要创建一个OpenGL context
		glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
		//disable resize功能
		//glfwWindowHint(GLFW_RESIZABLE, GLFW_FALSE);

		//创建真正的window
		//前三个参数确定了window的宽、高和标题。第四个参数允许我们选择指定打开window的监视器，最后一个参数仅与OpenGL相关。
		window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);

		//注册resize回调
		glfwSetFramebufferSizeCallback(window, framebufferResizeCallback);
	}

	static void framebufferResizeCallback(GLFWwindow* window, int wdth, int height)
	{

	}
```

我们创建一个**静态函数**作为回调的原因是**GLFW不会知道如何正确通过HelloTriangleApplication实例的this指针调用一个函数成员**。

但是，我们确实在回调中获得了对GLFWwindow的引用，并且还有另一个GLFW函数允许我们在其中存储任意指针：**glfwSetWindowUserPointer**。

```c++
		//创建真正的window
		//前三个参数确定了window的宽、高和标题。第四个参数允许我们选择指定打开window的监视器，最后一个参数仅与OpenGL相关。
		window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
		//将helloTriangleApplication的this指针存储到window中
		glfwSetWindowUserPointer(window, this);
		//注册resize回调
		glfwSetFramebufferSizeCallback(window, framebufferResizeCallback);
```

现在在回调中，我们可以通过**glfwGetWindowUserPointer**来正确获取变量，设置flag。

```c++
	static void framebufferResizeCallback(GLFWwindow* window, int wdth, int height)
	{
		//设置flag为true，发生resize
		auto app = reinterpret_cast<HelloTriangleApplication*>(glfwGetWindowUserPointer(window));
		app->framebufferResized = true;
	}
```
现在尝试运行程序，并且resize window，可以看到framebuffer正确随window调整了大小。

#### 4 处理最小化 Handling minimization

这里有另外一个案例可能导致swap chain变成out of date，并且有**一种特殊的window resize：窗口最小化**。这个案例特殊的原因是它**会造成size为0的frame buffer**。在该教程中，我们通过拓展recreateSwapChain函数来暂停直到窗口回到前台，来处理该问题。

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
		...
	}
```

对**glfwGetFramebufferSize**的初始调用处理大小已经正确切glfwWaitEvents没有任何等待的情况。

好耶，我们现在完成了出色的Vulkan程序。在下一章，我们会摆脱顶点着色器中硬编码的顶点数据，并且实际使用vertex buffer。

#### 参考
1. 题图来自画师wlop
