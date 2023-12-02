---
layout:     post
title:      "【VulkanTutorial学习笔记——3】Drawing a triangle-Set up-1"
subtitle:   ""
date:       2023-10-06 12:00:00
author:     "recaeee"
header-img: "img/in-post/default-bg.jpg"
tags:
    - Vulkan
---
# 【VulkanTutorial学习笔记——3】Drawing a triangle-Set up-1


![20231006221002](https://raw.githubusercontent.com/recaeee/PicGo/main/20231006221002.png)



#### 0 写在前面
本系列为个人在跟随Vulkan Tutorial学习过程中记录的一些笔记，一是防止自己看着看着弃坑，二是希望能在看过之后多少留些印象，三是想在啥时候忘记了能快速地回忆一些Vulkan知识点。

出于以上这些目的，该系列的学习笔记可能会写得较随意一些，并且对于主观判断不重要的东西可能会略过，因此也不保证该系列笔记的学习质量，可能有些东西不经过脑子就会写出来。

我是一个Vulkan的小白，对于Vulkan尚不熟悉，笔记中可能存在错误，或者概念不明确的地方，也希望看的人可以批评斧正~

原教程链接：
https://vulkan-tutorial.com/Introduction
本章节链接：
https://vulkan-tutorial.com/Drawing_a_triangle/Setup/Base_code
https://vulkan-tutorial.com/Drawing_a_triangle/Setup/Instance

终于正式开始要画三角形了~虽然本章节内容还是window的生命周期管理和vkInstance的创建（不过开始有重要的知识点了）

### Base code
#### 1 通用结构 General structure
在前面的章节我们创建了一个拥有正确配置的Vulkan项目并且成功测试了样例代码。在这一节中，我们重新书写如下代码。

```c++
#include <vulkan/vulkan.h>

#include <iostream>
#include <stdexcept>
#include <cstdlib>

class HelloTriangleApplication
{
public:
	void run()
	{
		initVulkan();
		mainLoop();
		cleanup();
	}

private:
	void initVulkan()
	{

	}

	void mainLoop()
	{

	}

	void cleanup()
	{

	}
};

int main()
{
	HelloTriangleApplication app;

	try 
	{
		app.run();
	}
	catch(const std::exception& e)
	{
		std::cerr << e.what() << std::endl;
		return EXIT_FAILURE;
	}

	return EXIT_SUCCESS;
}
```

我们首先Include LunarG SDK的Vulkan头文件，它提供了函数、结构体和枚举类型。stdexcept和iostream头文件用于报告和输出errors。cstdlib头文件提供了EXIT_SUCCESS和EXIT_FAILURE宏。

程序自身被封装在了一个类中，在其中我们存放Vulkan相关对象作为私有成员，并且创建**initVulkan**函数来用于实例化它们。当初始化完毕，我们进入mainloop来开始渲染每一帧。我们将在**mainLoop**函数中写一个循环，循环退出条件为当window关闭。一旦window被关闭，我们将在**cleanup**函数中确保释放我们用到的所有资源。

如果在程序执行期间遇到了任何一种致命错误，我们将会抛出一个std::runtime_error异常和一段描述信息，它将返回到主函数并且被print到命令行。为了处理各种标准异常类型，我们catch更通用的std::exception。一个在不久之后我们会处理到的一个Error例子就是发现某个需要的Extension不支持。

大约在这之后的每一章都会新增一个函数在initVulkan中被调用，和增加一到多个Vulkan Objects到私有成员，同时也需要在cleanup中释放它们。

#### 2 资源管理 Resource management
就像使用malloc分配的任何memory chunk都需要调用一次free，任何我们创建的Vulkan object都需要在我们不再需要它们时被显式释放。在C++中，资源管理可以通过使用RAII（构造函数完成分配，析构函数完成释放）或者memory头文件下提供的smart pointers。但是，在该教程中选择了显示操作任何Vulkan objects的分配和释放。毕竟Vulkan的优势在于明确每项操作以避免错误，因此明确objects的生命周期对于学习API如何工作是有好处的。

在本教程中，我们当然可以在构造函数中获取Vulkan对象并且在析构函数中释放它们来实现自动资源管理，或者给std::unique_ptr或std::shared_ptr提供一个custom deleter。RAII对于更大的Vulkan程序来说是比较推荐的一个资源管理模型，但出于学习目的，我们需要了解其背后的运作。

Vulkan objects通常有两种创建方式，一种是**通过名为vkCreateXXX的函数直接create**，另一种是**通过其他object的vkAllocateXXX函数来allocate**。在确保一个object不再会被使用后，我们需要**通过配对的vkDestroyXXX和vkFreeXXX函数来释放它们**。不同类型对象的创建、销毁函数的参数通常有所不同，但它们都有一个共同的参数：**pAllocator**。这是一个可选的参数，允许我们指向自定义内存分配器的回调。在该教程中，我们将始终无视该参数，并一直置为nullptr。

#### 3 集成GLFW Integrating GLFW
Vulkan可以在离屏渲染下正常工作，但显示出来更令人有学下去的动力（。
因此首先把#include <vulkan/vulkan,h>用如下代码代替。
```c++
#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>
```

这样，GLFW将会包含其自己的一些定义，并且自动加载Vulkan头文件。接下来在run函数的开头增加一个initWindow函数，通过该函数来**初始化GLFW并且创建一个window**。

```c++
	void run()
	{
		initWindow();
		initVulkan();
		mainLoop();
		cleanup();
	}

private:
	void initWindow()
	{

	}
```

在initWindow中，最开始的调用应该是**glfwInit()**，用来**初始化GLFW库**。由于GLFW最开始是被用于创建OpenGL context的，所以我们需要告诉它**不要创建一个OpenGL context**。

```c++
glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
```

由于处理可调整尺寸的window需要特别小心，我们之后会深入，因此在这里通过以下代码先**disable resize功能**。

```c++
glfwWindowHint(GLFW_RESIZABLE, GLFW_FALSE);
```

接下来就**创建真正的window**。在私有成员中新增一个**GLFWwindow* window**来存储window的指针，并且通过以下函数来初始化它。

```c++
window = glfwCreateWindow(800, 600, "Vulkan", nullptr, nullptr);
```

前三个参数确定了window的宽、高和标题。第四个参数允许我们选择指定打开window的监视器，最后一个参数仅与OpenGL相关。

更加建议使用常量而不是硬编码的width和height，因为我们会多次引用这些值。

```c++
const uint32_t WIDTH = 800;
const uint32_t HEIGHT = 600;
```

到目前为止，initWindow的代码如下。

```c++
	void initWindow()
	{
		glfwInit();

		glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
		glfwWindowHint(GLFW_RESIZABLE, GLFW_FALSE);

		window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
	}
```

为了确保应用直到发生错误或者关闭window才结束运行，我们需要增加一个事件循环在mainLoop中，如下。

```c++
	void mainLoop()
	{
		while (!glfwWindowShouldClose(window))
		{
			glfwPollEvents();
		}
	}
```

该段代码逻辑非常明确，循环并且检查window结束事件，比如用户按下“X”按钮。在之后，我们也会在这个循环中调用渲染一帧的函数。

一旦window被关闭，我们需要在cleanup函数中**销毁已分配的资源**，目前需要销毁window，并且**结束GLFW本身**。

好了，现在运行程序我们就可以看到一个标题为Vulkan的window。一个Vulkan的应用程序框架就搭好了。



![20231006122854](https://raw.githubusercontent.com/recaeee/PicGo/main/20231006122854.png)



好了这一小节基本结束，在这一小节中完成了**Vulkan程序基础框架的搭建，并且管理window的生命周期**，review一下整体代码，并且再简单加上一些注释（一切不写注释的代码都是耍流氓~）。

趁着现在代码量少，先粘在文中方便看吧，后面应该就都放在github工程中了（之后我会粘上github仓库地址）。

```c++
//GLFW将会包含其自己的一些定义，并且自动加载Vulkan头文件
#define GLFW_INCLUDE_VULKAN
#include <GLFW/glfw3.h>

//stdexcept和iostream头文件用于报告和输出errors
#include <iostream>
#include <stdexcept>
//cstdlib头文件提供了EXIT_SUCCESS和EXIT_FAILURE宏
#include <cstdlib>

//使用常量而不是硬编码的width和height，因为我们会多次引用这些值
const uint32_t WIDTH = 800;
const uint32_t HEIGHT = 600;

class HelloTriangleApplication
{
public:
	void run()
	{
		initWindow();
		initVulkan();
		mainLoop();
		cleanup();
	}

private:
	GLFWwindow* window;

	//始化GLFW并且创建一个window
	void initWindow()
	{
		//初始化GLFW库
		glfwInit();

		//由于GLFW最开始是被用于创建OpenGL context的，所以我们需要告诉它不要创建一个OpenGL context
		glfwWindowHint(GLFW_CLIENT_API, GLFW_NO_API);
		//disable resize功能
		glfwWindowHint(GLFW_RESIZABLE, GLFW_FALSE);

		//创建真正的window
		//前三个参数确定了window的宽、高和标题。第四个参数允许我们选择指定打开window的监视器，最后一个参数仅与OpenGL相关。
		window = glfwCreateWindow(WIDTH, HEIGHT, "Vulkan", nullptr, nullptr);
	}

	//**initVulkan**函数来用于实例化Vulkan objects私有成员
	void initVulkan()
	{

	}

	//mainloop来开始渲染每一帧
	void mainLoop()
	{
		//为了确保应用直到发生错误或者关闭window才结束运行，我们需要增加一个事件循环在mainLoop中，如下
		while (!glfwWindowShouldClose(window))
		{
			//处理窗口事件并触发事件回调函数、如鼠标、键盘事件、窗口尺寸的调整、窗口关闭等
			glfwPollEvents();
		}
	}

	//一旦window被关闭，我们将在**cleanup**函数中确保释放我们用到的所有资源
	void cleanup()
	{
		//销毁window
		glfwDestroyWindow(window);

		//结束GLFW本身
		glfwTerminate();
	}
};

int main()
{
	HelloTriangleApplication app;

	try 
	{
		app.run();
	}
	catch(const std::exception& e)
	{
		std::cerr << e.what() << std::endl;
		return EXIT_FAILURE;
	}

	return EXIT_SUCCESS;
}
```

接下来创建第一个Vulkan object！

### Instance
#### 4 创建一个实例 Creating an instance

首先我们需要做的第一件事情是**通过创建一个instance来初始化Vulkan library**。这个**Instance**是我们的**应用与Vulkan libaray之间的联系**，创建它需要**向驱动**指定和我们的应用相关的一些详细信息。

创建一个createInstance函数，并且在initVulkan函数中调用它。并且增加一个**VkInstance**私有成员。

```c++
	//**initVulkan**函数来用于实例化Vulkan objects私有成员
	void initVulkan()
	{
		createInstance();
	}

	private:
	GLFWwindow* window;
	VkInstance instance;
```

接下来在createInstance函数中，我们需要填充一个struct，其中**包含了我们的应用的信息**。这些信息是可选的，但它们可能会**向驱动提供一些有用的信息来优化应用**。这个struct叫做**VkApplicationInfo**。

```c++
	void createInstance()
	{
		VkApplicationInfo appInfo{};
		appInfo.sType = VK_STRUCTURE_TYPE_APPLICATION_INFO;
		appInfo.pApplicationName = "Hello Triangle";
		appInfo.applicationVersion = VK_MAKE_VERSION(1, 0, 0);
		appInfo.pEngineName = "No Engine";
		appInfo.engineVersion = VK_MAKE_VERSION(1, 0, 0);
		appInfo.apiVersion = VK_API_VERSION_1_0;
	}
```

正如之前所说，**在Vulkan中的许多structs都需要我们显式地在成员sType中确定其类型**。VkApplicationInfo也是成员pNext可以指向extension的众多成员之一。目前，我们用nullptr来初始化它。

Vulkan中的很多信息都是通过struct而不是函数参数来传递的，我们必须再填写一个struct来创建VkInstance实例。

下一个struct是必须要填写的，它告诉了Vulkan驱动我们想要使用哪些**全局extensions和validation layers**。在这里的“全局”指它们将作用于整个程序而不是一个特定的device。

```c++
		VkInstanceCreateInfo createInfo{};
		createInfo.sType = VK_STRUCTURE_TYPE_INSTANCE_CREATE_INFO;
		createInfo.pApplicationInfo = &appInfo;
```

前两个参数是简单明确的。正如overview章节所说，Vulkan是一个平台无关的API，这意味着**我们需要一个extension来与window system交互**。GLFW有一个方便的Built-in函数，返回了它所需要的extensions，我们可以将其传递给VkInstanceCreateInfo。

```c++
		uint32_t glfwExtensionCount = 0;
		const char** glfwExtensions;

		glfwExtensions = glfwGetRequiredInstanceExtensions(&glfwExtensionCount);

		createInfo.enabledExtensionCount = glfwExtensionCount;
		createInfo.ppEnabledExtensionNames = glfwExtensions;
```

VkInstanceCreateInfo中最后两个成员决定了开启的global validation layers。我们将会在后续章节展开它，所以目前我们先置为0。

```c++
createInfo.enabledLayerCount = 0;
```

目前我们定义了Vulkan创建一个instance需要的任何东西，接下来我们调用**vkCreateInstance**。

```c++
		VkResult result = vkCreateInstance(&createInfo, nullptr, &instance);
```

正如我们所见，**Vulkan中的对象创建函数的参数遵循的一般模式**是：
1. **指向带有创建信息的结构体的指针**
2. **指向自定义分配器回调的指针，在本教程中始终为nullptr**
3. **指向存储新object的handle变量的指针**

如果一切顺利，这个instance的handle将会被存储到VkInstance的成员中。几乎所有的Vulkan函数都返回一个**VkResult**类型，其值为VK_SUCCESS或者一个Error code。要检查instance是否创建成功，我们不需要存储返回的VkResult，只需要增加一层if判断就行。因此创建Instance应该这样写。

```c++
		if (vkCreateInstance(&createInfo, nullptr, &instance) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to create instance!");
		}
```

#### 5 处理VK_ERROR_INCOMPATIBLE_DRIVER Encountered VK_ERROR_INCOMPATIBLE_DRIVER

如果使用MacOS和最新的MoltenVK sdk，我们也许会从vkCreateInstance中得到VK_ERROR_INCOMPATIBLE_DRIVER。从1.3.216 Vulkan SDK开始，VK_KHR_PORTABILITY_subset拓展是强制性的。

教程中中提到了该Error的解决方法，由于我用的windows，所以略过这部分。

#### 6 检查Extension支持 Checking for extension support

如果我们看了vkCreateInstance的文档，我们会看到一个可能的error code是**VK_ERROR_EXTENSION_NOT_PRESENT**。我们可以简单地指定我们需要的extension，并且在error code返回时终止程序。这对于像window system这样的基础extension是有意义的，但是如果我们想要检查可选的extension功能怎么办？

要**在创建Instance前检索支持的extension列表**，需要使用**vkEnumerateInstanceExtensionProperties**函数。它需要一个指向extension数量的指针和一个VkExtensionProperties数组来存储这些extensions的详细信息。它还需要一个可选的第一个参数，允许偶们通过特定的validation layer来过滤extensions，我们现在会忽略它。

要分配一个数组来保存extension details，我们首先需要知道它们有多少个。我们可以通过将最后一个参数置为空来先只获取extensions的数量。

```c++
		uint32_t extensionCount = 0;
		vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, nullptr);
```

再分配一个数组来存储extension details，如下。

```c++
		uint32_t extensionCount = 0;
		vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, nullptr);
		std::vector<VkExtensionProperties> extensions(extensionCount);
		vkEnumerateInstanceExtensionProperties(nullptr, &extensionCount, extensions.data());
```

**每一个VkExtensionProperties包含了一个extension的name和version**。我们可以在一个循环中简单输出它们。

```c++
		std::cout << "available extensions\n";

		for (const auto& extension : extensions)
		{
			std::cout << '\t' << extension.extensionName << '\n';
		}
```

我们可以将这段代码加到createInstance函数中来输出Vulkan支持的一些信息。作为挑战，尝试创建一个函数来检查glfwGetRequiredInstanceExtensions返回的所有extension是否都包含在支持的extension列表中。参考评论区，我这也简单写了一个。

```c++
	void checkRequiredExtensionsSupported(const char** requiredExtensions, uint32_t requiredExtensionCount, std::vector<VkExtensionProperties> supportedExtensions)
	{
		for (int i = 0; i < requiredExtensionCount; i++)
		{
			bool found = false;
			for (const auto& extension : supportedExtensions)
			{
				if (strcmp(requiredExtensions[i], extension.extensionName))
				{
					found = true;
				}
			}
			if (!found)
			{
				throw std::runtime_error("Required extension not supported.");
			}
		}
		std::cout << "All required extensions are supported." << std::endl;
	}
```

运行程序后可以在命令行看到如下输出。



![20231006162310](https://raw.githubusercontent.com/recaeee/PicGo/main/20231006162310.png)



#### 7 清除 Cleaning up

VkInstance只应该在程序退出前一刻销毁。可以通过vkDestroyInstance函数在cleanup函数中销毁它。

```c++
	//一旦window被关闭，我们将在**cleanup**函数中确保释放我们用到的所有资源
	void cleanup()
	{
		vkDestroyInstance(instance, nullptr);

		//销毁window
		glfwDestroyWindow(window);

		//结束GLFW本身
		glfwTerminate();
	}
```

vkDestroyInstance的函数参数非常明确。正如前文所说，Vulkan中的分配和释放函数都有一个可选的自定义分配器回调，我们将在这里置为nullptr。我们在接下来的章节创建的**所有其他的Vulkan资源都应该在instance销毁前释放**。

ok，到这里我们完成了vkInstance的创建，vkInstance的作用是应用与Vulkan libaray之间的联系。分p现场~

最后再review一遍代码，这里我也粘上我的工程，其中包括了详细的注释，Vulkan项目工程地址：https://github.com/recaeee/VulkanTutorialProject。

#### 参考
1. 题图来自画师wlop