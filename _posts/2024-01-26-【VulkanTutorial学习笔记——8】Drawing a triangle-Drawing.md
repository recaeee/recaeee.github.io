---
layout:     post
title:      "【VulkanTutorial学习笔记——8】Drawing a triangle-Drawing"
subtitle:   ""
date:       2023-12-31 12:00:00
author:     "recaeee"
header-img: "img/in-post/default-bg.jpg"
tags:
    - Vulkan
---
# 【VulkanTutorial学习笔记——8】Drawing a triangle-Drawing

![20240126194046](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240126194046.png)

#### 0 写在前面
本系列为个人在跟随Vulkan Tutorial学习过程中记录的一些笔记，一是防止自己看着看着弃坑，二是希望能在看过之后多少留些印象，三是想在啥时候忘记了能快速地回忆一些Vulkan知识点。

出于以上这些目的，该系列的学习笔记可能会写得较随意一些，并且对于主观判断不重要的东西可能会略过，因此也不保证该系列笔记的学习质量，可能有些东西不经过脑子就会写出来。

我是一个Vulkan的小白，对于Vulkan尚不熟悉，笔记中可能存在错误，或者概念不明确的地方，也希望看的人可以批评斧正~

在本章节，我们终于可以画出我们的第一个三角形了，在这个章节之前，我们创建了许许多多的Vulkan Objects，并且现在基本上也对Vulkan Object的创建熟悉很多了，在这一章，会将所有创建的Vulkan Object串联起来，在主渲染循环中实际使用它们，实现一帧的渲染。在其中我也整理了我关于Vulkan绘制一帧的大致脑图，供大家参考~

![Vulkan-DrawATriangle.drawio](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGoVulkan-DrawATriangle.drawio.png)

原教程链接：
https://vulkan-tutorial.com/Introduction
本章节链接：
https://vulkan-tutorial.com/Drawing_a_triangle/Drawing/Framebuffers
https://vulkan-tutorial.com/Drawing_a_triangle/Drawing/Rendering_and_presentation
https://vulkan-tutorial.com/Drawing_a_triangle/Drawing/Frames_in_flight

#### 1 帧缓冲区 Framebuffers

我们在之前的一些章节已经讨论过很多关于framebuffers的内容了，并且我们设置了render pass来期望一张和swap chain images格式相同的framebuffer，但是我们仍然并没有创建任何framebuffer。

我们通过将render pass创建期间指定的attachments包装到**VkFramebuffer**中实现绑定。一个framebuffer对象引用了代表这些attachment的所有VkImageView对象。在我们的项目中，我们只有一个attachment，即color attachment。但是，我们为这个attachment实际使用的image取决于我们进行presentation时从swap chain中取回的那张image。这意味着**我们必须为swap chain中的所有image都创建framebuffer，并且在绘制时使用swap chain返回的image所对应的framebuffer**。

我们使用vector成员来存储这些framebuffers，并且为它们写一个**createFramebuffers**函数，在创建完管线后调用它。

```c++
	//存储framebuffers
	std::vector<VkFramebuffer> swapChainFramebuffers;

    //**initVulkan**函数来用于实例化Vulkan objects私有成员
	void initVulkan()
	{
		...
		createFramebuffers();
	}

    //一旦window被关闭，我们将在**cleanup**函数中确保释放我们用到的所有资源
	void cleanup()
	{
		//销毁framebuffers，需要在renderpass、imageviews销毁前、一帧绘制完后销毁
		for (auto framebuffer : swapChainFramebuffers)
		{
			vkDestroyFramebuffer(device, framebuffer, nullptr);
		}
		...
	}

    //为swap chain的每个image创建framebuffers
	void createFramebuffers()
	{
		swapChainFramebuffers.resize(swapChainImageViews.size());
		for (size_t i = 0; i < swapChainImageViews.size(); i++)
		{
			VkImageView attachments[] = {
				swapChainImageViews[i]
			};

			VkFramebufferCreateInfo framebufferInfo{};
			framebufferInfo.sType = VK_STRUCTURE_TYPE_FRAMEBUFFER_CREATE_INFO;
			framebufferInfo.renderPass = renderPass; // 要兼容的render pass
			framebufferInfo.attachmentCount = 1;
			framebufferInfo.pAttachments = attachments; // 对应的imageviews
			framebufferInfo.width = swapChainExtent.width;
			framebufferInfo.height = swapChainExtent.height;
			framebufferInfo.layers = 1;

			if (vkCreateFramebuffer(device, &framebufferInfo, nullptr, &swapChainFramebuffers[i]) != VK_SUCCESS)
			{
				throw std::runtime_error("failed to create framebuffer!");
			}
		}
	}
```

创建framebuffer非常简洁明了，首先我们需要**指定framebuffer需要与哪个renderPass兼容**。我们必须保证使用的的render passes和对应的framebuffer兼容，粗略来说就是**framebuffer需要和render pass使用相同数量和类型的attachment**。

我们需要在framebuffers对应的image views和render pass之前销毁framebuffers，但是销毁必须是在完成渲染后。

目前我们就拥有了渲染需要用到的的所有对象，下一张我们将会写实际的绘制指令。

#### 2 命令缓冲区 Command buffers

Vulkan中的指令，如绘制操作和内存传输，并不是通过函数调用直接实现的。我们必须**在command buffer对象中录制我们想执行的所有指令**。这样的好处是当我们准备好告诉Vulkan我们想做什么时，所有指令会被一起提交，并且由于这些指令被group到了一起，**Vulkan可以更高效地处理这些指令**。另外，这也**允许我们多线程录制指令**。

#### 3 命令池 Command pools

在我们创建**command buffer**之前，我们需要创建**command pool**。Command pools会管理用来**存储command buffer的内存**，并且**command buffers是从其中分配的**。添加一个类成员来存储一个**VkCommandPool**，并且新写一个**createCommandPool**函数在framebuffers创建完之后调用。

```c++
	void createCommandPool()
	{
		QueueFamilyIndices queueFamilyIndices = findQueueFamilies(physicalDevice);

		VkCommandPoolCreateInfo poolInfo{};
		poolInfo.sType = VK_STRUCTURE_TYPE_COMMAND_POOL_CREATE_INFO; // Allow command buffers to be rerecorded individually, without this flag they all have to be reset together
		poolInfo.flags = VK_COMMAND_POOL_CREATE_RESET_COMMAND_BUFFER_BIT;
	
		poolInfo.queueFamilyIndex = queueFamilyIndices.graphicsFamily.value(); // 单个pool的指令只能提交到一个queue

		if (vkCreateCommandPool(device, &poolInfo, nullptr, &commandPool) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to create command pool!");
		}
	}
```

创建Command pool只需要2个参数。参数**flags**有两个值可选：

1. **VK_COMMAND_POOL_CREATE_TRANSIENT_BIT**：表明command buffers频繁会被新指令重新录制（可能会改变内存分配行为）。
2. **VK_COMMAND_POLL_CREATE_RESET_COMMAND_BUFFER_BIT**：
允许command buffers被单独录制，若没启用此flag则所有指令需要被一起重置。

我们将会每帧录制一个command buffer，所以我们想要能够重置以及重录制它。因此，我们需要为command pool置为VK_COMMAND_POLL_CREATE_RESET_COMMAND_BUFFER_BIT。

我们通过将Command buffers发送到其中一个device queue来执行它们，比如我们取得的graphics和presentation queues。**每个command pool只能分配会被发送到单独一个类型的queue中的command buffers**。我们将要为绘制录制指令，因此我们选择graphics queue family。

创建CommandPool的函数格式我们已经再熟悉不过了。因为Commands会在程序整个生命周期内用于绘制，因此在cleanup中销毁它。

#### 4 命令缓冲区分配 Command buffer allocation

接下来可以分配command buffers啦~创建一个类成员**VkCommandBuffer**，**当Command pool销毁时，Command buffers会被自动释放**，因此我们不需要显式销毁它们。

我们通过**vkAllocateCommandBuffers**来创建Command buffer，其中需要填充一个**VkCommandBufferAllocateInfo**来指定command pool和要分配的buffer数量。

```c++
	void createCommandBuffer()
	{
		VkCommandBufferAllocateInfo allocInfo{};
		allocInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_ALLOCATE_INFO;
		allocInfo.commandPool = commandPool;
		allocInfo.level = VK_COMMAND_BUFFER_LEVEL_PRIMARY;
		allocInfo.commandBufferCount = 1;

		if (vkAllocateCommandBuffers(device, &allocInfo, &commandBuffer) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to allocate command buffers!");
		}
	}
```

其中，**level**参数指定了command buffers是primary还是secondary command buffers。
1. **VK_COMMAND_BUFFER_LEVEL_PRIMARY**：可以被提交给queue来执行，但是不能被其他Command buffers调用。
2. **VK_COMMAND_BUFFER_LEVEL_SECONDARY**：不能直接被提交给queue，但是能被primary command buffers调用。

我们在这里不会用到secondary command buffer的功能，但是我们可以想象到它有助于重用command operations。

#### 5 录制指令 Command buffer recording

我们现在写一个**recordCommandBuffer**函数来将我们想要执行的指令写入一个command buffer。传入参数包括要使用的VkCommandBuffer和当前swapchain image的index。

我们总是通过调用**vkBeginCommandBuffer**来开始录制指令，它需要一个**VkCommandBufferBeginInfo**来指定当前Command buffer的用途的一些细节。

```c++
	void recordCommandBuffer(VkCommandBuffer commandBuffer, uint32_t imageIndex)
	{
		VkCommandBufferBeginInfo beginInfo{};
		beginInfo.sType = VK_STRUCTURE_TYPE_COMMAND_BUFFER_BEGIN_INFO;
		beginInfo.flags = 0; // Optional
		beginInfo.pInheritanceInfo = nullptr; // Optional

		if (vkBeginCommandBuffer(commandBuffer, &beginInfo) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to begin recording command buffer!");
		}
	}
```

**flags**参数指定了我们将如何使用该command buffer，其可以是以下值：
1. **VK_COMMAND_BUFFER_USAGE_ONE_TIME_SUBMIT_BIT**：该command buffer将会在执行过一次后被重录制。
2. **VK_COMMAND_BUFFER_USAGE_RENDER_PASS_CONTINUE_BIT**：该command buffer是一个secondary command buffer，并且只会在一个render pass中使用。
3. **VK_COMMAND_BUFFER_USAGE_SIMULTANEOUS_USE_BIT**：该command buffer可以在等待执行时被重新提交。

这些flags目前都不适合我们。

**pInteritanceInfo**参数只和secondary command buffers有关，它指定从调用primary command buffers中继承哪个状态。

一旦一个command buffer已经完成了录制，此时再对它调用vkBeginCommandBuffer将会隐式地重置它。在录制完后追加指令到一个buffer中时不可能的。

#### 6 启动一个渲染通道 Starting a render pass

**整个绘制过程的开始是通过vkCmdBeginRenderPass启动一个render pass**。render pass会在**VkRenderPassBeginInfo**中被配置。

```c++
	void recordCommandBuffer(VkCommandBuffer commandBuffer, uint32_t imageIndex)
	{
		...

		//录制开始，先启动一个render pass
		VkRenderPassBeginInfo renderPassInfo{};
		renderPassInfo.sType = VK_STRUCTURE_TYPE_RENDER_PASS_BEGIN_INFO;
		//render pass自身和要绑定的attachments
		renderPassInfo.renderPass = renderPass;
		renderPassInfo.framebuffer = swapChainFramebuffers[imageIndex];
		//渲染区域的尺寸
		renderPassInfo.renderArea.offset = { 0, 0 };
		renderPassInfo.renderArea.extent = swapChainExtent;
		//在使用VK_ATTACHMENT_LOAD_OP_CLEAR时使用的Clear Values
		VkClearValue clearColor = { {0.0f, 0.0f, 0.0f, 1.0f} };
		renderPassInfo.clearValueCount = 1;
		renderPassInfo.pClearValues = &clearColor;

		vkCmdBeginRenderPass(commandBuffer, &renderPassInfo, VK_SUBPASS_CONTENTS_INLINE);
	}
```

**VkRenderPassBeginInfo**的前2个参数是**render pass自身和要绑定的attachments**。我们为每个swap chain image都创建了一个framebuffer，将其视为color attachment。因此我们需要绑定需要绘制到的swap chain image对应的frame buffer，使用传入参数imageIndex来选择正确的framebuffer。

接下来2个参数定义**渲染区域的尺寸**。渲染区域定义shader加载和存储的位置。超出区域的像素会拥有未定义值。它应该匹配attachment的尺寸来获得最好的性能。

最后2个参数定义了**在使用VK_ATTACHMENT_LOAD_OP_CLEAR时使用的Clear Values**，在Load Color attachment时用于清除。在这里定义clear color为完全不透明的纯黑色。

渲染通道现在可以开启了。所有用于录制指令的函数都会以**vkCmd**作为前缀。它们均返回void，因此在录制完成之前都不会有错误处理。

每个指令的第一个参数都是要被记录到的command buffer。第二个参数指定了render pass的细节。最后一个参数控制render pass内的指令将会如何被提供，它可以有两个值：
1. **VK_SUBPASS_CONTENTS_INLINE**：render pass commands将嵌入到primary command buffer本身中，并且不会执行secondary command buffers。
2. **VK_SUBPASS_CONTENTS_SECONDARY_COMMAND_BUFFERS**：render pass commands将会从secondary command buffers执行。

#### 7 基础绘制指令 Basic drawing commands

现在我们可以绑定graphics pipeline。

```c++
	void recordCommandBuffer(VkCommandBuffer commandBuffer, uint32_t imageIndex)
	{
		...
		//绑定graphics pipeline
		vkCmdBindPipeline(commandBuffer, VK_PIPELINE_BIND_POINT_GRAPHICS, graphicsPipeline);
		//配置dynamic states
		VkViewport viewport{};
		viewport.x = 0.0f;
		viewport.y = 0.0f;
		viewport.width = static_cast<float>(swapChainExtent.width);
		viewport.height = static_cast<float>(swapChainExtent.height);
		viewport.minDepth = 0.0f;
		viewport.maxDepth = 1.0f;
		vkCmdSetViewport(commandBuffer, 0, 1, &viewport);

		VkRect2D scissor{};
		scissor.offset = { 0, 0 };
		scissor.extent = swapChainExtent;
		vkCmdSetScissor(commandBuffer, 0, 1, &scissor);
	}
```

第二个参数指定了该pipeline是graphics还是compute pipeline。我们现在已经告诉Vulkan在graphics pipeline中执行哪些操作以及在片元着色器中使用哪些attachment。

正如之前fixed functions小节提到的，我们已经为管线指定了viewport和scissor state为dynamic state。因此我们需要在处理绘制指令前**通过command buffer**设置它们。

现在我们准备为三角形录制绘制指令。

```c++
	vkCmdDraw(commandBuffer, 3, 1, 0, 0);
```

实际的**vkCmdDraw**函数有些虎头蛇尾，它非常简单因为所有的信息我们都提前确定了。除了第一个参数为command buffer，它还有以下几个参数：
1. **vertexCount**：虽然我们并没有vertex buffer，但技术上我们还是需要绘制3个顶点。
2. **instanceCount**：用于instanced rendering，使用1表示不做instance渲染。
3. **firstVertex**：用作vertex buffer的偏移，决定了**gl_VertexIndex**的最小值。
4. **firstInstance**：用作instance渲染的偏移，决定了**gl_InstanceIndex**的最小值。

#### 8 收尾 Finishing up

render pass现在可以结束了，我们也完成了command buffer的录制。

```c++
		//结束render pass
		vkCmdEndRenderPass(commandBuffer);
		//结束command buffer录制
		if (vkEndCommandBuffer(commandBuffer) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to record command buffer!");
		}
```
在下一章节，我们将会为主循环编写代码，在其中将会从swap chain中获取一张image，录制和执行一个command buffer，然后将渲染完成的image返回给swap chain。冲~

#### 9 渲染和呈现 Rendering and presentation

这一节我们会把所有东西串联起来。我们将会编写**drawFrame**函数，它会在主线程中被调用来将三角形呈现在屏幕上。

```c++
	//mainloop来开始渲染每一帧
	void mainLoop()
	{
		//为了确保应用直到发生错误或者关闭window才结束运行，我们需要增加一个事件循环在mainLoop中，如下
		while (!glfwWindowShouldClose(window))
		{
			//处理窗口事件并触发事件回调函数、如鼠标、键盘事件、窗口尺寸的调整、窗口关闭等
			glfwPollEvents();
			//绘制一帧
			drawFrame();
		}
	}

	void drawFrame()
	{

	}
```

#### 10 一帧的流程 Outline of a frame

在high level，在Vulkan中**渲染一帧**由以下几个步骤组成：
1. 等待上一帧完成。
2. 从swap chain中取得一张image。
3. 录制一个command buffer用于将场景绘制到image上。
4. 提交录制好的command buffer。
5. 呈现swap chain image。

#### 11 同步 Synchronization

**Vulkan的一个核心设计哲学就是GPU上的同步执行是显式明确的**。操作的顺序由我们使用的各种**同步原语Synchronization primitives**来定义，它们告诉驱动我们想要的执行顺序。这也意味着开始在GPU上执行工作的许多Vulkan API调用都是异步的，函数将会在其操作完成之前返回。

在这一节中，我们需要对许多事件进行显式地排序，因为它们发生在GPU上，比如：
1. 从swap chain中取得一张image。
2. 执行绘制到image的command buffers。
3. 将image返回到swap chain中从而显示在屏幕上。

这些事件中的每一个都是使用单个函数调用来启动的，但是都会被异步地执行。函数将会在其实际操作完成前返回，并且其执行顺序也是未定义地。这是“不幸”的，因为以上举例的三步都依赖于前一步的完成。因此我们接下来需要探讨以下为了实现期望的执行顺序，我们需要使用哪些同步原语。

#### 11 信号量 Semaphores

**一个信号量Semaphores用于在队列操作之间增加顺序关系**。队列操作指的是我们提交到一个Queue上的工作，无论是在Command buffer中还是在一个后面我们会遇到的函数内。Queues的例子有Graphics queue和presentation queue。**Semaphores可以用于同一个Queue内对工作排序，也可以用于不同的Queue之间的工作排序**。

Vulkan中有2种Semaphores，**binary**和**timeline**。由于我们在教程中只会用到binary semaphores，我们并不会讨论timeline semaphores。**之后提到的semaphores均指代binary semaphores**。

**一个semaphore只有2种状态，unsignal或者signal**。**在它的生命周期开始，它是unsignaled**。我们使用一个semaphores来对工作排序的方式是在一个队列操作中提供一个'signal'的semaphore，并在另一个队列操作中提供'wait'的同一个semaphore。例如，我们有一个semaphore命名为S，以及队列操作A和B。当我们告诉Vulkan当操作A完成时它将会'signal'（大概理解成标记激活的意思？）Semaphore S，并且操作B会在开始执行之前'wait' Semaphore S。当操作A完成，semaphore S被signaled，同时操作B直到Semaphore S为'signal'之前都不会开始。在操作B开始执行后，smaphore S会自动被重置为'unsignaled'，允许它再次被使用。

伪代码如下。

```c++
VkCommandBuffer A, B = ... // record command buffers
VkSemaphore S = ... // create a semaphore

// enqueue A, signal S when done - starts executing immediately
vkQueueSubmit(work: A, signal: S, wait: None)

// enqueue B, wait on S to start
vkQueueSubmit(work: B, signal: None, wait: S)
```

注意以上代码段落，**两次vkQueueSubmit()的调用都会立即返回，等待操作只会发生在GPU上**，CPU将会畅通无阻地继续执行下去。为了让CPU等待，我们需要另一个同步原语，也就是接下来要说地同步原语。

#### 12 栅栏 Fences

栅栏**Fence**（直译有些怪~）和semaphore有相似的作用，即用于执行同步，但是它**用于在CPU**（也称为host，从这个意义上来说，GPU也被视为了server？）**上排序**。如果host需要知道GPU何时完成了某些操作，我们就需要用到Fences。

一个具体的例子是截图操作。假设我们已经在GPU上完成了必要的工作，现在需要将image从GPU传输到host上，并且将image的内存存储到一个文件中。我们有一个command buffer A来执行传输工作，和一个fence F。我们让command buffer A和fence F一起提交，然后立刻告诉host等待fence F变为signal。这就可以让host直到command buffer A执行完成之前都被阻断。这是我们就可以安全地让host将文件保存到磁盘，因为内存传输操作已经完成了。

伪代码如下。

```c++
VkCommandBuffer A = ... // record command buffer with the transfer
VkFence F = ... // create the fence

// enqueue A, start work immediately, signal F when done
vkQueueSubmit(work: A, fence: F)

vkWaitForFence(F) // blocks execution until A has finished executing

save_screenshot_to_disk() // can't run until the transfer has finished
```

和semaphore的案例不同，这个案例会导致host执行被阻断。这意味着host除了等待fence变为signal之前不会做任何事情。对于这种情况，我们必须保证内存传输已完成，然后才能将屏幕截图保存到磁盘。

通常来说，**非必要的情况下，我们尽量不要阻断host**（阻断cpu的行为代价很大呢~）。我们希望喂饱GPU和host，让它们一直做有效的工作（不准偷懒！）。等待fence变为signal并不是一个有效的工作。因此我们**更偏向于使用semaphores**，或者其他我们目前未涉及到的同步原语**来同步我们的工作**。

**Fences必须被手动重置来让其进入unsignaled状态**。这是因为fence用于控制host的执行，因此host需要决定什么时候重置这个fence。对比来说，semaphores用于在GPU上执行同步，而无需host参与。

那我们选择哪个同步原语捏？小孩子才做选择，大人们早就已经全都要了（

我们有2个同步原语可以使用，并且有2个地方可以方便地执行同步：**Swapchain操作**和**等待上一帧完成**。我们希望**使用semaphores来同步swapchain操作**，因为它们发生在GPU上，无需host参与。**对于等待上一帧完成，我们希望使用fences**，因为我们需要让host等待，这样我们一次就不会绘制超过一帧（当然有更高效的决策，之后小节会说）。因为我们每帧都会重录制command buffer，我们在直到当前帧的工作执行完成之前，我们都不能将下一帧的工作录制到command buffer中，因为我们不能在GPU正在使用command buffer时覆写掉它中当前的内容。

#### 13 创建同步对象 Creating the synchronization objects

我们需要一个semaphore来标记一个image已经从swap chain中被取到了，并且准备被用于渲染；另一个semaphore用于标记渲染已经完成了，可以用于presentation；以及一个fence用于保证一次只有一帧正在渲染。

创建3个类成员来存储这些semaphores和fence。

```c++
	//存储从swapchain中取得image的semaphore
	VkSemaphore imageAvailableSemaphore;
	//存储image渲染完毕的semaphore
	VkSemaphore renderFinishedSemaphore;
	//存储保证一次只渲染一帧的fence
	VkFence inFlightFence;

	void createSyncObjects()
	{
		VkSemaphoreCreateInfo semaphoreInfo{};
		semaphoreInfo.sType = VK_STRUCTURE_TYPE_SEMAPHORE_CREATE_INFO;

		VkFenceCreateInfo fenceInfo{};
		fenceInfo.sType = VK_STRUCTURE_TYPE_FENCE_CREATE_INFO;

		if (vkCreateSemaphore(device, &semaphoreInfo, nullptr, &imageAvailableSemaphore) != VK_SUCCESS
			||
			vkCreateSemaphore(device, &semaphoreInfo, nullptr, &renderFinishedSemaphore) != VK_SUCCESS
			||
			vkCreateFence(device, &fenceInfo, nullptr, &inFlightFence) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to create semaphores!");
		}
	}

	//一旦window被关闭，我们将在**cleanup**函数中确保释放我们用到的所有资源
	void cleanup()
	{
		//当程序结束，并且所有commands都完成了，并且没有更多的同步被需要时，semaphores和fence需要被销毁
		vkDestroySemaphore(device, imageAvailableSemaphore, nullptr);
		vkDestroySemaphore(device, renderFinishedSemaphore, nullptr);
		vkDestroyFence(device, inFlightFence, nullptr);
		...
	}

```

为了创建semaphores，我们将会增加教程这部分中最后一个create函数**createSyncObjects**。创建semaphores需要填充**VkSemaphoreCreateInfo**结构体，但在当前版本的API中，我们除了**sType**并不需要其他任何fields。

未来版本的Vulkan API或者拓展可能会增加功能给**flags**和**pNext**参数，就像其为其他结构体做的那样。

一样，创建一个Fence需要填充**VkFenceCreateInfo**。创建semaphores和fence的函数都很简单。同时，**当程序结束，并且所有commands都完成了，并且没有更多的同步被需要时，semaphores和fence需要被销毁**。

#### 14 等待前一帧 Waiting for the previous frame

接下来终于开始写drawFrame函数了！在一帧的开始，我们希望等待到上一帧完毕，只有这时command buffer和semaphores才可以被使用，所以我们先调用**vkWaitForFences**。

```c++
	void drawFrame()
	{
		//等待上一帧完成
		vkWaitForFences(device, 1, &inFlightFence, VK_TRUE, UINT64_MAX);
		//重置fence
		vkResetFences(device, 1, &inFlightFence);
	}

	void createSyncObjects()
	{
		...

		VkFenceCreateInfo fenceInfo{};
		fenceInfo.sType = VK_STRUCTURE_TYPE_FENCE_CREATE_INFO;
		//创建时置为signaled，避免第一帧block
		fenceInfo.flags = VK_FENCE_CREATE_SIGNALED_BIT;

		...
	}
```

vkWaitForFences函数**传入一个fence数组**，它**让host等待到其中任何一个或者所有fences为signaled时再返回**。**传入的VK_TRUE表示我们希望等待所有的fences**，虽然在这里只有一个fence。该函数还有一个**timeout参数**，我们将其设置为UINT64的最大值，其会disable超时返回返回（即不会触发超时返回）。

在等待完毕后，我们需要通过**vkResetFences**手动重置fence到unsignaled。

在我们继续前，在我们的设计中有一个小问题。在第一帧我们调用drawFrame()会立即等待inFlightFence变为signaled。而inFlightFence只会在一帧完成渲染后变为signaled，而第一帧之前没有任何帧来signal该fence，因此**第一帧会永远block住**。

在众多解决方法中，API内置了一个巧妙的解决方法，**在创建该fence时初始化为signaled状态**，由此第一次调用vkWaitForFences()会立即返回，因为fence已经是signaled。为了实现它，我们**在VkFenceCreateInfo中增加VK_FENCE_CREATE_SIGNALED_BIT标识符**。

#### 15 从swap chain获取image Acquiring an image from the swap chain

接下来需要在drawFrame中做的事情是**从swap chain中获取一张image**。再次回顾下，swap chain是一个拓展feature，所以我们使用的函数会以KHR作为后缀。

```c++
	void drawFrame()
	{
		//等待上一帧完成
		vkWaitForFences(device, 1, &inFlightFence, VK_TRUE, UINT64_MAX);
		//重置fence
		vkResetFences(device, 1, &inFlightFence);
		//从swap chain中获取一张image
		uint32_t imageIndex;
		vkAcquireNextImageKHR(device, swapChain, UINT64_MAX, imageAvailableSemaphore, VK_NULL_HANDLE, &imageIndex);
	}
```

**vkAcquireNextImageKHR**的前两个参数是**logical device**和**swap chain**。第三个参数定义了一个纳秒单位下的**timeout**，同样在这里我们使用UINT64的最大值disable超时功能。接下来两个参数指定了**当presentation engine使用完image时要发出信号的同步对象**，这就是我们可以开始绘制的时间点。指定一个semaphore或fence或两者都有是都可行的。我们将在这里使用imageAvailableSemaphore。最后一个参数指定一个变量来**输出可用的swap chain image的索引**，该索引对应了我们的swapChainImages数组中VkImage的索引，我们可以通过该索引选取到对应的VkFrameBuffer。

#### 16 录制command buffer Recording the command buffer

有了imageIndex，我们现在可以实际录制command buffer了。首先，我们调用**vkResetCommandBuffer**来**确保command buffer可以被录制**。vkResetComandBuffer的第二个参数是一个**VkCommandBufferReseFlagBits**的标识符。因为我们不想做其他特殊的事情，我们将其置为0。

接下来我们就可以调用之前写好的**recordCommandBuffer**来录制我们想要的指令。

```c++
	void drawFrame()
	{
		...
		//重置command buffer，确保其可以被录制
		vkResetCommandBuffer(commandBuffer, 0);
		//录制指令
		recordCommandBuffer(commandBuffer, imageIndex);
	}
```

有了已经充分录制好的command buffer之后，我们就可以提交它了。

#### 17 提交command buffer Submitting the command buffer

**Queue submission**和**Synchronization**都会在**VkSubmitInfo**结构体中通过参数被配置。这里也是重要知识点的地方，务必认真思考，从中体会Vulkan的设计哲学~

```c++
		//提交command buffer到queue中，并且配置同步
		VkSubmitInfo submitInfo{};
		submitInfo.sType = VK_STRUCTURE_TYPE_SUBMIT_INFO;
		//配置同步
		VkSemaphore waitSemaphores[] = { imageAvailableSemaphore };
		VkPipelineStageFlags waitStages[] = { VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT };
		submitInfo.waitSemaphoreCount = 1;
		submitInfo.pWaitSemaphores = waitSemaphores;
		submitInfo.pWaitDstStageMask = waitStages;
		//配置要提交的command buffer
		submitInfo.commandBufferCount = 1;
		submitInfo.pCommandBuffers = &commandBuffer;
		//配置command buffer执行完毕后要signal的semaphores
		VkSemaphore signalSemaphores[] = { renderFinishedSemaphore };
		submitInfo.signalSemaphoreCount = 1;
		submitInfo.pSignalSemaphores = signalSemaphores;
		//提交Command buffer
		if (vkQueueSubmit(graphicsQueue, 1, &submitInfo, inFlightFence) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to submit draw command buffer!");
		}
```

前三个参数指定了**在command buffer开始执行前需要等待哪几个semaphores**，以及**在哪个pipeline stage(s)等待**。我们**希望直到image可用之后才将颜色写入到image中，所以我们指定管线阶段为写入到color attachment**。这意味着理论上在实际工作中，可以在image不可用时先开始执行顶点着色器（这里其实算是乍一看比较难想明白的一点，也就是说**一个command buffer中的所有指令都可以被归类进几个管线阶段，我们可以从管线阶段这个粒度去控制同步**，这一思想非常关键）。waitStages数组中的每个entry都对应pWaitSemaphores中具有相同索引的信号量。

接下来两个参数很简单，指定**要提交的command buffer数量以及对应指针**。

再接下来的**signalSemaphoreCount**和**pSignalSemaphores**参数指定了**一旦command buffer完成执行后要signal的semaphores**。在我们的案例中使用**renderFinishedSemaphore**来标记渲染完成的时机。

最后使用**vkQueueSubmit**将command buffer提交到graphics queue。该函数第一个参数为**VkSubmitInfo数组**，以在工作负载较大时提高效率。最后一个参数指向一个**可选的fence**，**当command buffers执行完毕后将被signal的fence**。该fence允许我们知道什么时候我们能安全地重用command buffer，因此我们将其置为inFlightFence。现在在下一帧，CPU将会等待command buffer执行完毕再开始录制新指令到command buffer中。

#### 18 子通道依赖 Subpass dependencies

注意**在一个render pass中的subpasses会自动进行image layout的转换**，这些转换被**subpass dependencies**控制，其**指定了subpasses之间的内存依赖和执行依赖**。我们现在只有一个subpass，但是该subpass之前和之后的操作都被认定为隐式的"subpasses"。

**有两个内置依赖负责处理render pass开始时和结束时的transition**，但前者没有在正确的时间发生。它会假设transition发生在pipeline的开始处，但次数我们尚未获取image。有两个方法可以解决该问题。一，我们可以将waitStages中imageAvailableSemaphore的同步阶段改成VK_PIPELINE_STAGE_TOP_OF_PIPE_BIT，来保证render pass在image可用前都不会开始（但这样vertex shader就不能先执行了？）。二，我们可以**让render pass等待VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT**。我们使用第二个方法，这样我们更清楚subpass dependencies的工作原理。我们希望在需要将颜色写入到image时才对它的layout进行transition。

subpass dependencies在VkSubpassDenpendecy结构体中被指定，我们到**createRenderPass**函数中增加它。

```c++
		//配置subpass dependency
		VkSubpassDependency dependency{};
		dependency.srcSubpass = VK_SUBPASS_EXTERNAL; // 被依赖的subpass
		dependency.dstSubpass = 0; //依赖的subpass
		dependency.srcStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
		dependency.srcAccessMask = 0;
		dependency.dstStageMask = VK_PIPELINE_STAGE_COLOR_ATTACHMENT_OUTPUT_BIT;
		dependency.dstAccessMask = VK_ACCESS_COLOR_ATTACHMENT_WRITE_BIT;

		//创建render pass
		...
		renderPassInfo.dependencyCount = 1;
		renderPassInfo.pDependencies = &dependency;
```

前两个fields指定了**依赖项**（被依赖的subpass，注意隐式subpass也是可能值，事实上这里就为隐式subpass）和**依赖subpass的索引**。特殊值**VK_SUBPASS_EXTERNAL指render pass之前或之后的隐式subpass**，具体取决于它是在srcSubpass还是dstSubpass中指定。索引0指向我们的唯一的subpass。dstSubpass必须总是高于srcSubpass来避免dependency graph中出现循环（除非dstSubpass之一是VK_SUBPASS_EXTERNAL）。

接下来两个fields指定**要等待的操作以及这些操作发生的阶段**。我们需要等待swap chain完成image的读取（即acquire完毕）然后才能访问它。这可以通过**等待color attachment output stage本身**来实现（也就是**当需要将颜色输出到color attachment时才对image layout进行transition**）。

需要等待的阶段是color attachment阶段，涉及到color attachment的写入。这些设置将会阻止发生transition，知道实际需要（并允许）时：**当我们想要开始向其写入颜色时**。

**VkRenderPassCreateInfo**结构体中有两个filed来明确一个依赖数组。

#### 19 呈现 Presentation

绘制一帧的最后一步是将结果发送回swap chain来让它最后显示在屏幕上。Presentation是通过一个位于drawFrame函数最后的**VkPresentInfoKHR**结构体来配置的。

```c++
	void drawFrame()
	{
		//等待上一帧完成
		//重置fence
		//从swap chain中获取一张image
		//重置command buffer，确保其可以被录制
		//录制指令
		//提交command buffer到queue中，并且配置同步
		//配置同步
		//配置要提交的command buffer
		//配置command buffer执行完毕后要signal的semaphores
		//提交Command buffer
		...

		//Presentation
		VkPresentInfoKHR presentInfo{};
		presentInfo.sType = VK_STRUCTURE_TYPE_PRESENT_INFO_KHR;
		//等待command buffer执行完毕
		presentInfo.waitSemaphoreCount = 1;
		presentInfo.pWaitSemaphores = signalSemaphores;
		//呈现image的swap chains和每个swap chain的image index
		VkSwapchainKHR swapChains[] = { swapChain };
		presentInfo.swapchainCount = 1;
		presentInfo.pSwapchains = swapChains;
		presentInfo.pImageIndices = &imageIndex;
		//指定一个VkResult数组，来检查每个swap chain的presentation是否成功
		presentInfo.pResults = nullptr; // Optional
		//向swap chain提交一个present an image的请求
		vkQueuePresentKHR(presentQueue, &presentInfo);
	}
```

其前两个参数明确了**在呈现前需要等待哪些semaphores**，就像VkSubmitInfo一样。因为我们想等待command buffer执行完成后才绘制三角形，因此使用signalSemaphores。

接下来两个参数指定了**呈现image的swap chains和每个swap chain的image index**。

最后一个**可选参数pResults**。它允许我们可以指定一个VkResult数组，来检查每个swap chain的presentation是否成功。使用单个swap chain则没必要用到它，因为我们可以直接使用present函数的返回值。

**vkQueuePresentKHR函数向swap chain提交一个present an image的请求**。在下一节，我们会对vkAcquireNextImageKHR和vkQueuePresentKHR函数增加错误处理，因为它们的failure不一定意味着程序应该终止，不像我们之前写的函数。

现在，让我们运行下程序，Ohhhhhhhhhhhhh~（发疯）

![20240117214627](https://raw.githubusercontent.com/recaeee/PicGo/main/20240117214627.png)

泪目，耗费了1268行代码，终于把三角形画出来了。

这个彩色三角形的颜色可能和其他graphics tutorials中看到的有一丢丢不同。这是因为该教程中让**着色器在线性颜色空间中进行插值，然后转换成sRGB颜色空间**。有关差异的讨论，可以参考[以下博客](https://medium.com/@heypete/hello-triangle-meet-swift-and-wide-color-6f9e246616d9)。

当然，我更推荐这篇[《Gamma、Linear、sRGB 和Unity Color Space，你真懂了吗？》](https://zhuanlan.zhihu.com/p/66558476)~

不幸的是，当开启Validation layers后，程序会在关闭时crash，terminal上的信息告诉了我们为什么。

![20240117215313](https://raw.githubusercontent.com/recaeee/PicGo/main/20240117215313.png)

要记住**drawFrame中的所有操作都是异步的**，这意味着当我们从mainLoop中推出的时候，drawing和presentation操作也许会继续，在这个时候清理资源是个坏主意。

为了修复该问题，我们应该**在退出mainLoop之前等待logical device完成所有操作，然后销毁window**。

```c++
	//mainloop来开始渲染每一帧
	void mainLoop()
	{
		//为了确保应用直到发生错误或者关闭window才结束运行，我们需要增加一个事件循环在mainLoop中，如下
		while (!glfwWindowShouldClose(window))
		{
			//处理窗口事件并触发事件回调函数、如鼠标、键盘事件、窗口尺寸的调整、窗口关闭等
			glfwPollEvents();
			//绘制一帧
			drawFrame();
		}

		//等待logical device完成所有操作，再退出mainLoop
		vkDeviceWaitIdle(device);
	}
```

我们也可以使用vkQueueWaitIdle来等待一个特定command queue中所有操作执行完毕。这些函数可以用作执行同步的非常基本的方法。现在关闭window的时候不会crash啦~

#### 20 小结 Conclusion

超过900行代码（实际算注释我写了1268行），我们终于画出点啥了。引导Vulkan程序绝对是一项繁重的工作，但要点是，Vulkan通过其明确性为我们提供了巨大的控制权。原教程中建议我们花一些时间重新阅读代码，并构建程序中所有Vulkan对象的用途以及它们之间的相互关系的脑图。从现在开始，我们将在这些知识的基础上拓展程序的功能。

好啦，虽然心中无限想偷懒，但人有的时候总得逼下自己，呈上我的图~

![Vulkan-DrawATriangle.drawio](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGoVulkan-DrawATriangle.drawio.png)

下一节，我们将拓展render loop以处理“飞行中的多个帧”（让子弹飞一会~）

#### 21 让帧飞一会 Frames in flights

现在我们的渲染循环还有个明显的缺点。我们需要**等待前一帧完成再开始渲染下一帧**，导致了没必要的**host（CPU）空闲**。

修复方法是**允许多个帧同时运行**（原来不是让帧飞一会，哈哈哈），也就是说，**允许一帧的渲染完全不干扰下一次录制**。怎么做呢？**在渲染时任何被获取和修改的资源都需要被复制**。因此，我们需要多个command buffers、semaphores、fences。在之后的章节，我们也会增加其他资源的多个实例。

首先在程序顶增加一个常量定义多少帧应该被同时进行。

```c++
//多少帧应该同时工作
const int MAX_FRAMES_IN_FLIGHTS = 2;
```

我们选择2帧同时运行，因为我们不想让CPU领先GPU太多（想了想应该是在GPU瓶颈的场景下会有这种情况，不过大部分游戏场景也都是GPU瓶颈）。**当2个帧同时运行时，CPU和GPU可以同时处理自己的任务**。如果CPU提前完成了，他将会等到GPU完成渲染后再提交更多工作。当3帧以上同时运行时，CPU将会领先GPU，从而增加帧延迟。通常来说，延迟是不期望的。但是让程序控制同时运行的帧数量是Vulkan显式的另一个案例。

每一帧应该有它自己的command buffer、sempahores和fence。重命名并且用数组持有这些对象。

```c++
	//存储command buffer
	std::vector<VkCommandBuffer> commandBuffers;
	//存储从swapchain中取得image的semaphore
	std::vector<VkSemaphore> imageAvailableSemaphores;
	//存储image渲染完毕的semaphore
	std::vector<VkSemaphore> renderFinishedSemaphores;
	//存储保证一次只渲染一帧的fence
	std::vector<VkFence> inFlightFences;
```

然后我们需要创建多个command buffers，将**createCommandBuffer**重命名为**createCommandBuffers**。然后我们需要将command buffer数组resize到MAX_FRAME_IN_FLIGHT，改变VkCommandBufferAllocateInfo来包含多个command buffers。**createSyncOjbects**、**cleanup**同理。

```c++
	void createCommandBuffers()
	{
		//从command pool中分配command buffer
		commandBuffers.resize(MAX_FRAMES_IN_FLIGHTS);
		...
		allocInfo.commandBufferCount = (uint32_t)commandBuffers.size();

		if (vkAllocateCommandBuffers(device, &allocInfo, commandBuffers.data()) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to allocate command buffers!");
		}
	}

	void createSyncObjects()
	{
		imageAvailableSemaphores.resize(MAX_FRAMES_IN_FLIGHTS);
		renderFinishedSemaphores.resize(MAX_FRAMES_IN_FLIGHTS);
		inFlightFences.resize(MAX_FRAMES_IN_FLIGHTS);

		...

		for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHTS; i++)
		{
			if (vkCreateSemaphore(device, &semaphoreInfo, nullptr, &imageAvailableSemaphores[i]) != VK_SUCCESS
				||
				vkCreateSemaphore(device, &semaphoreInfo, nullptr, &renderFinishedSemaphores[i]) != VK_SUCCESS
				||
				vkCreateFence(device, &fenceInfo, nullptr, &inFlightFences[i]) != VK_SUCCESS)
			{
				throw std::runtime_error("failed to create semaphores!");
			}
		}
	}

		//一旦window被关闭，我们将在**cleanup**函数中确保释放我们用到的所有资源
	void cleanup()
	{
		//当程序结束，并且所有commands都完成了，并且没有更多的同步被需要时，semaphores和fence需要被销毁
		for (size_t i = 0; i < MAX_FRAMES_IN_FLIGHTS; i++)
		{
			vkDestroySemaphore(device, imageAvailableSemaphores[i], nullptr);
			vkDestroySemaphore(device, renderFinishedSemaphores[i], nullptr);
			vkDestroyFence(device, inFlightFences[i], nullptr);
		}
		...
	}
```

需要注意，因为**command buffer pool销毁时，command buffers会自动释放**，因此command buffer并不需要做额外的清除工作。

为了使用正确的objects，我们需要**使用一个帧索引追踪当前帧**。

```c++
	//当前帧索引
	uint32_t currentFrame = 0;
```

drawFrame函数也做出对应修改，使用正确的objects，并且在最后递进到下一帧。

```c++
	void drawFrame()
	{
		//等待上一帧完成
		vkWaitForFences(device, 1, &inFlightFences[currentFrame], VK_TRUE, UINT64_MAX);
		//重置fence
		vkResetFences(device, 1, &inFlightFences[currentFrame]);
		...
		vkAcquireNextImageKHR(device, swapChain, UINT64_MAX, imageAvailableSemaphores[currentFrame], VK_NULL_HANDLE, &imageIndex);
		//重置command buffer，确保其可以被录制
		vkResetCommandBuffer(commandBuffers[currentFrame], 0);
		//录制指令
		recordCommandBuffer(commandBuffers[currentFrame], imageIndex);
		...
		//配置同步
		VkSemaphore waitSemaphores[] = { imageAvailableSemaphores[currentFrame]};
		...
		submitInfo.pCommandBuffers = &commandBuffers[currentFrame];
		//配置command buffer执行完毕后要signal的semaphores
		VkSemaphore signalSemaphores[] = { renderFinishedSemaphores[currentFrame]};
		...
		if (vkQueueSubmit(graphicsQueue, 1, &submitInfo, inFlightFences[currentFrame]) != VK_SUCCESS)
		{
			throw std::runtime_error("failed to submit draw command buffer!");
		}

		...
		//递进到下一帧
		currentFrame = (currentFrame + 1) % MAX_FRAMES_IN_FLIGHTS;
	}
```

通过使用模操作，我们可以保证帧索引在MAX_FRAMES_IN_FLIGHT内循环。

我们现在执行了所有需要的同步来保证工作帧数不超过MAX_FRAMES_IN_FLIGHTS数量，并且它们之间互不重叠。注意，代码的其他部分（比如cleanup）可以依赖更粗略的同步（比如vkDeviceWaitIdle）。我们需要根据性能需求来决定使用哪种方式。

在下一章，我们将会处理一个好的Vulkan程序中需要的琐碎工作。呼，累~道阻且长，加油！

#### 参考
1. 题图来自画师wlop
2. https://medium.com/@heypete/hello-triangle-meet-swift-and-wide-color-6f9e246616d9
3. https://zhuanlan.zhihu.com/p/66558476