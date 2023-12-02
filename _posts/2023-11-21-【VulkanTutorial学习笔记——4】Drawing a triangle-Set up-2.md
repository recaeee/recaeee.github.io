---
layout:     post
title:      "【VulkanTutorial学习笔记——4】Drawing a triangle-Set up-2"
subtitle:   ""
date:       2023-11-21 12:00:00
author:     "recaeee"
header-img: "img/in-post/default-bg.jpg"
tags:
    - Vulkan
---
# 【VulkanTutorial学习笔记——4】Drawing a triangle-Set up-2


![20231121212926](https://raw.githubusercontent.com/recaeee/PicGo/main/20231121212926.png)



#### 0 写在前面
本系列为个人在跟随Vulkan Tutorial学习过程中记录的一些笔记，一是防止自己看着看着弃坑，二是希望能在看过之后多少留些印象，三是想在啥时候忘记了能快速地回忆一些Vulkan知识点。

出于以上这些目的，该系列的学习笔记可能会写得较随意一些，并且对于主观判断不重要的东西可能会略过，因此也不保证该系列笔记的学习质量，可能有些东西不经过脑子就会写出来。

我是一个Vulkan的小白，对于Vulkan尚不熟悉，笔记中可能存在错误，或者概念不明确的地方，也希望看的人可以批评斧正~

另外这篇鸽了比较久，虽然期间大部分时间在摸鱼，但也系统性地看了一些Vulkan相关的文章，以免在之后的笔记中出现过多错误。在这里也墙裂推荐[不知名书杯的Vulkan系列文章](https://www.zhihu.com/people/lllzwj)，基本上看过一遍之后就能对Vulkan有了比较全面的了解。（同时，也体会到了官方的Tutorials是真的不说人话。）

另外本篇为Validation Layers的搭建，内容依然较少（算是前传吧），从下一章开始就是Vulkan的正片了~

原教程链接：
https://vulkan-tutorial.com/Introduction
本章节链接：
https://vulkan-tutorial.com/Drawing_a_triangle/Setup/Validation_layers

### 验证层 Validation layers
#### 1 什么是验证层 What are validation layers?

我们都知道Vulkan的设计思想之一是最小化驱动的花销（已经不知道提了多少次）。其中一项实践就是**默认情况下Vulkan提供了非常非常有限的Error checking**。（同时再提醒自己一次，另一项实践是将同步等工作都交给上层开发者而不再由驱动自己管理。）而这么少的错误检查就意味着，可能把枚举设成错误值或者参数传递空指针这么傻冒的操作，都不会被显式捕获，并且会**导致Crash和未知行为**。（有利有弊吧~）举个实际的犯傻例子就是在使用一个新的GPU特性时忘记在logic device创建时请求它。

那有什么办法能知道我们做了这些傻冒操作呢？答案是**Validation layers**，原文中谓之为优雅的system。Validation layers通过hook到一些vulkan函数调用中来增加一些其他的检查操作，并且它是可选的。（显而易见，best practice是在debug下启用它，在release下完全剥离它）。

Validation layers中一些常见的**检查操作**包括：
1. 根据规范检查参数以检测错误使用
2. 跟踪对象的创建和销毁来查资源泄露
3. 通过跟踪线程来检查线程安全性
4. 将每个调用和其参数输出到log
5. 追踪Vulkan调用来分析和replay

接下来，来看下Validation layer在一个函数中做了啥的一个例子。

```c++
VkResult vkCreateInstance(
    const VkInstanceCreateInfo* pCreateInfo,
    const VkAllocationCallbacks* pAllocator,
    VkInstance* instance) {

    if (pCreateInfo == nullptr || instance == nullptr) {
        log("Null pointer passed to required parameter!");
        return VK_ERROR_INITIALIZATION_FAILED;
    }

    return real_vkCreateInstance(pCreateInfo, pAllocator, instance);
}
```

各种validation layers可以自由堆叠，可以任君加到想加的函数中。再提一遍，显而易见**best practice是在debug下启用它，在release下完全剥离它**。

Vulkan不包含任何built-in的validation layers，但好在LunarG Vulkan SDK提供了一个非常nice的layers集合来检查一些常见的errors。它们完全开源，[github地址](https://github.com/KhronosGroup/Vulkan-ValidationLayers)，我们可以用来在debug时详细参考。**使用Validation layers是避免app因为我们的一些傻冒操作在不同驱动上终端的最佳伙伴（没有之一）**。

使用Validation layers还有一个条件：需要将Validation安装到OS上。例如，LunarG validation layers只可以用在安装了Vulkan SDK的PC上。

在以前，validation layers还分为了两大类：instance和device specific。instance只检查global Vulkan objects的调用如vkInstance相关的。device specific只检查特定GPU相关的调用。后来device specific类被启用，然后instance就干了全部的工作了。

#### 2 使用验证层 Using validation layers
好，到了使用validation layers的实战环节。好比使用extension一样，我们需要声明要用的validation layers的名字。基本所有有用的check都捆绑到了一个layer下，其名为**VK_LAYERS_KHRONOS_validation**(这是甚魔命名法？)

首先定义要用的Validation layers和是否启用。

```c++
//要使用的validation layers
const std::vector<const char*> validationLayers = {
	"VK_LAYER_KHRONOS_validation"
};

//debug模式下开启Validation layers，release则不启用
#ifdef NDEBUG
	const bool enableValidationLayers = false;
#else
	const bool enableValidationLayers = true;
#endif
```

然后我们需要检查需要使用到的Validation layers是否都支持，这里写法和申请extension的时候不能说毫不相关，只能说几乎一样。代码不贴了，很简单的。

#### 3 消息回调 Message callback

Validation layers默认会把debug信息输出到standard output上，但我们也可以**自己管理它们并提供一个显式回调**。**自己管理的另一个好处是可以做过滤**，有些不重要的error就可以不显示了。

为了自己管理，我们需要创建一个debug messenger，并且需要用到**VK_EXT_debug_utils**拓展。同样，通过一些简单的代码申请VK_EXT_debug_utils拓展。

我们通过一系列宏定义一个消息回调的静态函数，通过它我们可以从Validation layers种获取4种类型的消息：**VERBOSE诊断信息**、**INFO普通信息**（比如创建一个资源）、**WARNING警告信息**（潜在bug）、**ERROR错误信息**（非法行为，可能造成Crash）。这样我们就可以根据消息的类型对Validation layers传来的信息进行过滤。

我们甚至可以从Validation layers的消息回调中直接拿到出问题的Log所关联的Vulkan Objects（通过pCallbaackData），感觉这个Debug工具确实很强啊。同时我们还可以根据messageType了解到行为是否对性能有影响。

在这里，我们需要通过创建**DebugUtilsMessenger**来管理Debug消息，其创建流程和其他Vulkan Objects类似，需要先去构建CreateInfo。但有一点不同的在于，其创建函数不会随着Vulkan自动加载到内存，因此我们需要通过instance主动寻址找到其创建函数进行执行，销毁同理。

#### 4 调试Instance的创建和销毁 Debugging instance creation and destruction

由于DebugUtilsMessenger的创建和销毁需要一个合法的VkInstance作为传入参数，这导致了我们目前只能在VkInstance创建后到其销毁前的这段时间内进行Debug，而不能对VkInstance其自身的创建和销毁进行Debug。

但如果我们正好是读了extension documentation的小麻瓜，那我们就能知道我们可以为VkInstance的创建和销毁创建单独的DebugUtilsMessenger。

```c++
	void createInstance()
	{
		...

		VkInstanceCreateInfo createInfo{};
		createInfo.sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO;
		createInfo.pApplicationInfo = &appInfo;

		...

		//额外创建一个DebugUtilsMessenger，让vkInstance的pNext拓展指向它，这样可以debug vkCreateInstance和vkDestroyInstance的信息。
		VkDebugUtilsMessengerCreateInfoEXT debugCreateInfo{};
		if (enableValidationLayers)
		{
			createInfo.enabledLayerCount = static_cast<uint32_t>(validationLayers.size());
			createInfo.ppEnabledLayerNames = validationLayers.data();

			polulateDebugMessengerCreateInfo(debugCreateInfo);
			createInfo.pNext = (VkDebugUtilsMessengerCreateInfoEXT*)&debugCreateInfo;
		}
		else
		{
			createInfo.enabledLayerCount = 0;
		}

        ...
		if (vkCreateInstance(&createInfo, nullptr, &instance) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to create instance!");
		}
	}
```

好欸，这样我们就能全程安心地Debug了~

#### 5 测试 Testing

好，现在我们不小心故意地犯个错误来测试下Validation。我们在cleanup函数中注释掉DestroyDebugUtilsMessengerEXT。在我们结束进程时，我们将会看到如下图的Log。



![20231121211725](https://raw.githubusercontent.com/recaeee/PicGo/main/20231121211725.png)



#### 6 配置 Configuration

其实我们可以有很多很多设置来拓展Validation Layers的Debug功能，通过在VkDebugUtilsMessengerCreateInfoEXT的flags中声明来启用。我们可以通过浏览Vulkan SDK的Config字典，在那我们可以找到一个叫vk_layer_settings.txt的魔法书用来了解如何配置这些Layers。

我们可以自行去了解如何开启和配置这些layers，但在该教程的剩下部分将会使用默认的设置。

在教程中，我们会时而故意不小心做错饭去展示Validation layers有多好用，以及它对于程序员有多重要。

#### 参考
1. https://vulkan-tutorial.com/Drawing_a_triangle/Setup/Validation_layers
2. https://zhuanlan.zhihu.com/p/616082929
3. 题图来自画师wlop