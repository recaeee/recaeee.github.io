---
layout:     post
title:      "【VulkanTutorial学习笔记——5】Drawing a triangle-Setup-3"
subtitle:   ""
date:       2023-11-24 12:00:00
author:     "recaeee"
header-img: "img/in-post/default-bg.jpg"
tags:
    - Vulkan
---
# 【VulkanTutorial学习笔记——5】Drawing a triangle-Setup-3


![20231124211715](https://raw.githubusercontent.com/recaeee/PicGo/main/20231124211715.png)



#### 0 写在前面
本系列为个人在跟随Vulkan Tutorial学习过程中记录的一些笔记，一是防止自己看着看着弃坑，二是希望能在看过之后多少留些印象，三是想在啥时候忘记了能快速地回忆一些Vulkan知识点。

出于以上这些目的，该系列的学习笔记可能会写得较随意一些，并且对于主观判断不重要的东西可能会略过，因此也不保证该系列笔记的学习质量，可能有些东西不经过脑子就会写出来。

我是一个Vulkan的小白，对于Vulkan尚不熟悉，笔记中可能存在错误，或者概念不明确的地方，也希望看的人可以批评斧正~

在本章节中，我们终于踏上了Vulkan取经之路的正传，到新手村了，让我们首先来了解Physical Device、Queue families、Logical Device、Queue这几个概念。

原教程链接：
https://vulkan-tutorial.com/Introduction
本章节链接：
https://vulkan-tutorial.com/Drawing_a_triangle/Setup/Physical_devices_and_queue_families
https://vulkan-tutorial.com/Drawing_a_triangle/Setup/Logical_device_and_queues

#### 1 选择一个物理设备 Selecting a physical device

在通过VkInstance初始化我们的Vulkan library之后，我们需要在系统中挑选一个支持我们所有需要的特性的显卡。当然，我们可以挑选任意个显卡并且同时使用它们，但在教程中，我们只会选择第一个合适的显卡（毕竟要考虑到像我一样只有一个破卡的麻瓜）。

我们最终选择的这张显卡将会被存储到一个**VkPhysicalDevice**句柄中。该对象将会在VkInstance销毁时被隐式销毁，因此我们不需要在cleanup函数中主动做任何事。

选择显卡的第一个步骤是列举所有能用的显卡，方法就和列出Extension类似，先执行一遍函数得到数量，再创建Vector得到所有显卡，代码如下。

```c++
	//启用一张合适的显卡
	void pickPhysicalDevice()
	{
		//列出所有能用的显卡
		uint32_t deviceCount = 0;
		vkEnumeratePhysicalDevices(instance, &deviceCount, nullptr);
		//如果没有一张支持Vulkan的显卡可用，则抛出异常
		if (deviceCount == 0)
		{
			throw std::runtime_error("failed to find GPUs with Vulkan support!");
		}
		std::vector<VkPhysicalDevice> devices(deviceCount);
		vkEnumeratePhysicalDevices(instance, &deviceCount, devices.data());
		//找到合适的显卡
		for (const auto& device : devices)
		{
			if (isDeviceSuitable(device))
			{
				physicalDevice = device;
				break;
			}
		}
		
		if (physicalDevice == VK_NULL_HANDLE)
		{
			throw std::runtime_error("failed to find a suitable GPU!");
		}
	}

	//判断该显卡是否支持我们想要的功能
	bool isDeviceSuitable(VkPhysicalDevice device)
	{
		return true;
	}
```

#### 2 基本设备适用检查 Base device suitability checks

我们可以获取到PhysicalDevice的一些基本属性，例如名称、类型、支持的Vulkan版本等，同时也可以获取到它们的一些特性，例如纹理压缩、多视口渲染等等。通过这些信息，我们可以匹配到我们需要的显卡。

```c++
	//判断该显卡是否支持我们想要的功能
	bool isDeviceSuitable(VkPhysicalDevice device)
	{
		//获取显卡的基本属性，比如名字、类型、支持的vulkan版本
		VkPhysicalDeviceProperties deviceProperties;
		vkGetPhysicalDeviceProperties(device, &deviceProperties);
		//获取显卡支持的特性，例如纹理压缩、64比特浮点数、多视口渲染
		VkPhysicalDeviceFeatures deviceFeatures;
		vkGetPhysicalDeviceFeatures(device, &deviceFeatures);
		//在之后的章节我们还会查询显存、Queue families


		return true;
	}
```

我们可以简单地只要求显卡需要支持几何着色器，也可以采用复杂地评分机制，并且优先使用独显，若无可用再fallback到集显，等等。原文中给出了示例代码，但在本教程中只需要一个支持Vulkan的显卡，因此不做过于复杂的策略。

#### 3 队列族 Queue families

之前也提到过，几乎Vulkan中的任何操作，从绘制到上传纹理，都需要将Commands提交到一个Queue中来实现。关于Queue，存在很多不同类型的**Queues**，源自不同的队列族**Queue families**，**每个Queue families只允许用于Commands中的一个子集**。例如，存在只允许Compute commands的queue familiy，也存在只允许内存转移相关命令的queue family。

我们需要检查PhysicalDevice支持哪些Queue family，并且检查是否支持我们需要的Queue family。而目前，我们只需要让启用的Physical Device支持Graphics Queue，我们通过queueFamily.queueFlags & VK_QUEUE_GRAPHICS_BIT来判断。

```c++
	//判断该显卡是否支持我们想要的功能
	bool isDeviceSuitable(VkPhysicalDevice device)
	{
		QueueFamilyIndices indices = findQueueFamilies(device);

		return indices.isComplete();
	}

	//我们需要的所有Queue families
	struct QueueFamilyIndices
	{
		//optional允许uint32_t在实际分配值之前让其保持no value，并且可以通过has_value()来查询是否有值
		std::optional<uint32_t> graphicsFamily;

		//快速判断当前PhysicalDevices是否支持所有我们需要的Queue Families
		bool isComplete()
		{
			return graphicsFamily.has_value();
		}
	};

	//判断PhysicalDevice是否支持我们需要的Queue family
	QueueFamilyIndices findQueueFamilies(VkPhysicalDevice device)
	{
		QueueFamilyIndices indices;

		//获取PhysicalDevice支持的所有Queue family
		uint32_t queueFamilyCount = 0;
		vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, nullptr);
		std::vector<VkQueueFamilyProperties> queueFamilies(queueFamilyCount);
		vkGetPhysicalDeviceQueueFamilyProperties(device, &queueFamilyCount, queueFamilies.data());

		//我们需要支持Graphics Queue Family
		int i = 0;
		for (const auto& queueFamily : queueFamilies)
		{
			if (queueFamily.queueFlags & VK_QUEUE_GRAPHICS_BIT)
			{
				indices.graphicsFamily = i;
				break;
			}

			i++;
		}

		return indices;
	}
```

好，我们目前完成了获取正确PhysicalDevice的所有步骤，接下来创建一个Logic device来和它交互。

#### 4 逻辑设备和队列 Logical device and queues

在我们选择了一个正确的Physical device之后，我们需要创建一个**Logical device**来与之交互。当然，Logical device的创建过程就和VkInstance的创建过程类似（填充一个CreateInfo，描述我们想要的features巴拉巴拉）。既然我们查询了哪些Queue families可用，我们当然也要指定创建哪些队列**Queue**。甚至，我们可以对一个Physical device创建多个Logical devices，只要我们需要。

#### 5 指定要创建的队列 Specifying the queues to be created

Logical device的创建需要填充一个CreateInfo，首当其冲要填的一个信息是**VkDeviceQueueCreateInfo**，这个结构体描述了我们想要的单个Queue family中的Queue数量。目前，我们只需要一个具有图形功能的Queue。

**当前可用的驱动只允许我们为每个Queue family创建一个小数量的queues，事实上我们也用不着多个**。我们可以在多线程上创建所有Command buffers，并且通过一次低开销调用来在主线程上一次性提交它们。

Vulkan允许我们为Queues分配一个优先级来影响它们的Commandbuffer的执行调度，其值范围为0.0到1.0。需要注意，即使咱只有一个队列，也要设置这个值。

```c++
	void createLogicalDevice()
	{
		//先填充VkDeviceQueueCreateInfo，描述了我们想要的单个Queue family中的Queue数量。目前，我们只需要一个具有图形功能的Queue。
		QueueFamilyIndices indices = findQueueFamilies(physicalDevice);

		VkDeviceQueueCreateInfo queueCreateInfo{};
		queueCreateInfo.sType = VK_STRUCTURE_TYPE_DEVICE_QUEUE_CREATE_INFO;
		queueCreateInfo.queueFamilyIndex = indices.graphicsFamily.value();
		queueCreateInfo.queueCount = 1;
		//Vulkan允许我们为Queues分配一个优先级来影响它们的Commandbuffer的执行调度，其值范围为0.0到1.0
		float queuePriority = 1.0f;
		queueCreateInfo.pQueuePriorities = &queuePriority;
	}
```

#### 6 确定使用的设备特性 Specifying used devices features

接下来要填充的信息是我们将要用到的device features的一个集合。目前，咱不需要用到任何features，因此把它摆那就行啦~未来总有一天会接到这个回旋镖的！

#### 7 创建逻辑设备 Creating the logical device

好，我们目前创建并填充了俩结构体了，接下来咱就可以创建并填充最重要的**VkDeviceCreateInfo**了。

在VkDeviceCreateInfo中，我们还要明确对具体Device要启用的Extensions和ValidationLayers（**这里不同于创建VkInstance时启用的，而是对应具体Device启用的**）。对于Device需要启用的Extensions，一个很好的例子就是**VK_KHR_swapchain**，用来让我们渲染出来的Image呈现在windows上。一个Vulkan devices缺少该功能是有可能的，因为可能存在完全为了计算用的device。在之后的章节我们会填入该Extensions。

另外，对于具体device要启用的Validation layers，这里的值其实是无用的，因为**新一些的Vulkan SDK不再区分VkInstance和Device的Layers**，统一使用VkInstance的Layers，但为了保险，咱在这还是填一遍一样的值。

创建VkDevice的过程和创建VkInstance的过程类似，调用**vkCreateDevice**函数，参数格式也是类似的。为了熟记这种写法，咱再来复习一遍，第一个参数为我们想要与之交互的Physical device，第二个参数为CreateInfo，第三个参数为可选的Allocation callbacks pointer（在教程中始终为nullptr），最后一个参数为我们用来存储创建的Logical device的handle。

```c++
		//填充VkDeviceCreateInfo
		VkDeviceCreateInfo createInfo{};
		createInfo.sType = VK_STRUCTURE_TYPE_DEVICE_CREATE_INFO;
		createInfo.pQueueCreateInfos = &queueCreateInfo;
		createInfo.queueCreateInfoCount = 1;

		createInfo.pEnabledFeatures = &deviceFeatures;

		//填充针对于device的enabled extensions和（被废弃的）ValidationLayers
		createInfo.enabledExtensionCount = 0;

		if (enableValidationLayers)
		{
			createInfo.enabledLayerCount = static_cast<uint32_t>(validationLayers.size());
			createInfo.ppEnabledLayerNames = validationLayers.data();
		}
		else
		{
			createInfo.enabledLayerCount = 0;
		}
		
		//创建VkDevice
		if (vkCreateDevice(physicalDevice, &createInfo, nullptr, &device) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to create logical device!");
		}
```

另外，**我们需要在cleanup函数中显式销毁VkDevice**。Logical device并不会直接与VkInstance交互，这就是它不做为被包含的参数的原因。

#### 8 检索队列句柄 Retrieving queue handles

Queue会在我们创建VkDevice时自动创建，但我们需要存储一个Handle来与之交互，并且Queue会在VkDevice销毁时自动销毁，即**Queue的生命周期完全贴合VkDevice内部**。

咱要使用vkGetDeviceQueue来拿到这个Handle，其参数列表如下：
1. VkDevice：逻辑设备。
2. Queue family：要从哪个Queue family中获取Queue。
3. Queue Index：Queue的索引，因为我们只创建一个queue，所以为0。
4. Queue handle：返回的句柄

```c++
//Queue会在Logical device创建时自动创建，我们需要拿到它的句柄（它也会伴随VkDevice销毁而自动销毁）
		vkGetDeviceQueue(device, indices.graphicsFamily.value(), 0, &graphicsQueue);
```

在有了VkDevice和Queue handles之后，我们终于真正可以拿显卡来干些事情了。在下一章我们将会设置资源以将渲染结果呈现到窗口系统上。

#### 参考
1. https://zhuanlan.zhihu.com/p/555509777