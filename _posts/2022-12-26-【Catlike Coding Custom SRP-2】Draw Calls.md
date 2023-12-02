---
layout:     post
title:      "【Catlike Coding Custom SRP学习之旅——2】Draw Calls"
subtitle:   ""
date:       2022-12-26 12:00:00
author:     "recaeee"
header-img: "img/in-post/default-bg.jpg"
tags:
    - Custom SRP
---
# 【Catlike Coding Custom SRP学习之旅——2】Draw Calls
#### 写在前面
很高兴第一章有许多人看，因此我也不能懈怠，抓紧时间开始弄第二章（很想这么说，但后来因为实习+阳了，第二章拖得有点久了）。同时，这两天还在考虑毕设选题的事情，我在犹豫是搭建SRP+风格化渲染还是在URP的基础上直接着重开展风格化渲染（渲染的风格偏PBR+NPR的结合。前者由于搭建SRP会花大量的时间，所以工作量会比较大，而后者直接使用URP，但这样对管线的改造可能就会比较少，更多的时间会花在制作材质上。关于这方面我也问了大佬前辈，目前考虑的还是前者，肝起来肝起来~

以下是原教程链接与我的Github工程（Github上会实时同步最新进度）：

[CatlikeCoding-SRP-Tutorial](https://catlikecoding.com/unity/tutorials/custom-srp/)

[我的Github工程](https://github.com/recaeee/CatlikeCoding-Custom-RP)

废话少说，开始撕第二章吧！

--- 

![20221212230835](https://raw.githubusercontent.com/recaeee/PicGo/main/20221212230835.png)

#### Draw Calls

在第二章，我们的主题是DrawCalls，那还是首先唠唠什么是Draw Call吧。不管怎么说，Draw Call的含义比第一章的“自定义渲染管线”的含义理解起来容易多了。这里就直接参考《Shader入门精要》中对Draw Call的解释，**CPU通过调用Draw Call来告诉GPU开始一个渲染过程。一个Draw Call会指向本次调用需要渲染的图元列表**。

从这两句解释中，我们可以获得这些信息：1，Draw Call这个命令的发起方是CPU，接收方是GPU；2，Draw Call中传递的信息为“需要渲染的图元列表”。而**图元列表**其实就是一系列顶点、材质、纹理、着色器等数据。

更概念化地来说，**Draw Call是CPU调用图像编程接口，如OpenGL中地glDrawElements命令，以命令GPU进行渲染的操作**。我们可能会疑惑，在上一段话中我们说过Draw Call会传递顶点、纹理这些数据，但在这里我们又说Draw Call是一系列渲染命令，似乎不涉及数据的传递。

但其实我觉得两种说法都对，因为**一次Draw Call往往伴随着大量数据的传递**，这些大量的数据就是顶点、纹理这些数据。注意我说的是“伴随”，因为这些数据的传递其实是在Draw Call之前完成的。

在这里，我们梳理一下从CPU为起点，到Draw Call调用的流程。参考《Shader入门精要》，其经历了如下过程：1，把数据加载到显存中，把渲染所需的所有数据（顶点、法线、纹理坐标等）从硬盘加载到RAM，再从RAM加载到显存；2，设置渲染状态，设置着色器、光源属性、材质等；3，调用Draw Call，告诉GPU开始渲染。

由此可见，在Draw Call调用之前，我们会进行Mesh数据、材质数据、光源属性等等的传递，因此Draw Call的调用始终伴随着这些数据的传递。

总而言之，《Shader入门精要》对Draw Call进行了比较生动的解释，如果还不理解，可以看下原书。

而实际放到Unity中，我们在哪里体现出Draw Call呢？答案是，**一次DrawRenderer往往会产生一至多个Draw Call**。那我们知道，DrawRenderer这个函数是在CommandBuffer下的，那这里我们再谈回到CommandBuffer，思考这样一个问题，**为什么我们需要CommandBuffer？** 我们知道，CommandBuffer将一系列指令缓存在队列中一次提交给GPU，那为什么我们不是告诉GPU一条指令、GPU执行一条指令这样做呢？

其原因就是**Command Buffer（命令缓冲区）实现了让CPU和GPU并行工作**。命令缓冲区包含了一个命令队列，CPU向其中添加指令，GPU从中读取指令，添加和读取的过程是互相独立的。CommandBuffer中的指令有很多种，Draw Call是其中一种。

在每次调用Draw Call之前CPU都要向GPU发送许多内容（包括数据、状态、指令等），因此Draw Call的增多会使CPU压力过大，造成性能瓶颈。而为了减少Draw Call，我们就引入了**批处理（Batching）** 的方法，把多个小量Draw Call合并成一个大的Draw Call。

由此我们知道了何为Draw Call，Command Buffer的作用以及为什么我们要减少Draw Call。在这一章中，我们就会编写一系列Shader，使其支持**SRP Batcher、GPU Instancing和Dynamic Batching**这些批处理技术。

---

#### 1 着色器 Shaders

首先来唠唠Unity中的Shader吧。

我们在Unity中编写Shader使用的语法是Unity特有的**ShaderLab**，但它其实就是多种Shader语言（例如在其中可以使用HLSL、CG语言）的混合版，再加上一些特定的语法框架。如果想了解ShaderLab更底层的知识，请务必看下[这篇文章](https://zhuanlan.zhihu.com/p/400470713)，这个作者非常强，其他文章也值得一看。其中说到ShaderLab的作用，**ShaderLab是Unity构建的一种方便开发者做跨平台Shading开发的语言体系**，因为其底层会将我们所写的.Shader文件先使用FXC编译器编译成DXBC（一种可以直接被DX设备使用的二进制语言），如果目标平台不是DX，则会更进一步使用HLSLcc（交叉编译）将DXBC输出到对应平台（如转换成GLSL、Metal等）。

其实通常来说着色器就是Shader（在本文中也会这么认为，不做明确区分，但是还是要提一下），毕竟英文翻译过来就是这样。但是Unity对于Shader组织了一种新的数据类型，即**Shader对象**，我们在Unity中所编写的 **.shader文件**，其实是在编写**Shader类**，它和通常意义上的Shader最大的区别在于**一个Shader类之中可以定义多个通俗意义上的Shader（着色器程序）**，也就是多个**SubShader**，而这些SubShader其实才是我们通常所说的Shader。根据SubShader的官方说明，我们可以显而易见知道Unity组织Shader类这一数据结构的目的就在于**兼容不同的硬件、渲染管线和运行时设置**。在渲染管线实际运行时，Unity就会将我们编写的Shader类实例化成**Shader对象**使用。



![20221217135637](https://raw.githubusercontent.com/recaeee/PicGo/main/20221217135637.png)

*一个.shader文件*



而**一个SubShader的组成通常来说是一到多个Pass（也就是通道）**，而**一个Pass就是Shader对象中的最小单位**。官方对Pass的定义如下，**Pass是Shader对象的基本元素，它包含设置GPU状态的指令，以及在GPU上运行的着色器程序**。在一个Pass中，我们就会去定义我们最熟悉不过的**顶点着色器**（vertex函数）和**片元着色器**（fragment函数）。那如果大家编写过一些OpenGL，我们可能会感到诧异，因为在OpenGL中，我们会将Vertex Shader和Fragment Shader这些**着色器程序**定义为一个着色器，这么看来，在Unity中其实是一个Pass相当于一个完整的“着色器”（不是SubShader）？虽然说早期我也一直这么认为（认为一个Pass其实相当于一套Shader），的确很有道理，但其实这样的思想有些许欠妥。

对于这块的解释，且听我胡扯一番。我们首先看一下官方对Pass说过这样一句话，**简单的Shader对象可能只包括一个通道，但更复杂的着色器可以包含多个通道**。我们举一个很简单的例子就可以打破“Pass就是一套通俗意义上Shader”的思想。我们举例一个所谓的“简单的只包括一个Pass”的Shader对象，一个UnlitShader，在这个Pass中，我们让物体绘制成纯红色。简单思考一下，这个Pass里的vertex和fragment函数很简单吧，并且只要一个Pass就够了吧。那我们再举例一个“更复杂”的Shader对象，在这个SubShader中，我们还是把物体绘制成纯红色，**但是我们还希望这个物体能投射阴影**。这时候再思考一下，在这个Shader中，我们总共需要一个Pass来绘制纯红色，**还需要一个Pass来让物体渲染在阴影贴图上**，意味着在这个SubShader中我们需要定义2个Pass。通过这样一个例子（纯色渲染、纯色渲染+投射阴影），我们就能区分一个SubShader和一个Pass的区别，**SubShader决定了我们在整个渲染管线流程中渲染这个物体的所有行为（一个物体可能被渲染多次，可能一次屏幕上，一次阴影贴图上）**，而**一个Pass决定了我们对这个物体的一次渲染行为**。而在理解这点后，我们就会发现，如何理解SubShader和Pass就取决于我们对通俗意义上的Shader的理解，如果认为Shader确定了一次对物体的渲染行为（走一个顶点着色器，再走一个片元着色器），那Pass就是Shader；如果认为Shader确定了在整套渲染流程中物体的所有渲染行为，那SubShader就是Shader。曾经我更倾向前者，现在我更倾向后者。

好了，唠了这么多，大家是否对**通俗意义上的Shader、Unity中的Shader类、Shader对象、SubShader、Pass**这几个概念和它们之间的关系有了一定理解呢？

我想对于Shader的最好的学习方式，无非就是自己去写一些Shader，有时候说得再多都不如自己动手感受下直观。而通过这一章的学习，我们就会对Shader的基本组成和运作方式有一定理解。废话少说，开始写代码~

#### 1.1 无光照着色器 Unlit Shader

我们要写的第一个Shader是最简单的UnlitColor，也就是使用一个固定的颜色渲染一个mesh，通过编写这样一个简单的Shader，我们会去了解Shader的整体结构。

```c#
Shader "Custom RP/Unlit"
{
    Properties {}

    SubShader
    {
        Pass {}
    }
}
```

以上就是能编译通过的最最简单的Shader，在它的内部没有做任何事，只是写了一些关键字。但通过这个Shader，我们已经能够知道许多关于Shader的重要知识，因为通过它我们就可以知道Shader中所有必不可少的组成部分（因为它就是最简单的Shader了）。

先从结果来看，如果我们使用这个Shader创建一个材质，然后赋给一个物体，那么我们会看到这个物体变成了纯白色（有人或许会问为什么是纯白色，我也想知道，可能是默认的颜色？）。但我们更需要先看的是这个材质的Inspector视图。



![![20221216225449](httpsraw.githubusercontent.comrecaeeePicGomain20221216225449.png)](https://raw.githubusercontent.com/recaeee/PicGo/main/!%5B20221216225449%5D(httpsraw.githubusercontent.comrecaeeePicGomain20221216225449.png).png)



首先，我创建的这个材质命名为Unlit，Shader使用了上一步创建的Custom RP/Unlit，因此我们得知第一点，**每一个材质都需要其对应的Shader**（也就是说材质的创建依赖于Shader）。在我看来，Shader就好比定义一个C#类，而材质就是这个类的实例，在Shader中我们往往会定义一些参数，而在材质中我们需要赋予和确定这些参数的值。

其次，我们可以看到对于一个材质，它会拥有一个**Render Queue**的属性，其默认值是2000（From Shader意味着采用了Shader中的默认值，因为我们的Shader中没有设定这个默认值，所以Unity给它赋了个默认值2000）。

**Render Queue代表了此材质的渲染队列**，简单来说，**Render Queue意味着使用这个材质的物体在渲染管线中被渲染的顺序**。[官方文档](https://docs.unity3d.com/cn/2021.3/ScriptReference/Material-renderQueue.html)中说到，**Render Queue的值应该处于[0,5000]，或者为-1使用着色器的渲染队列**。

好了，说完了“简单来说”，接下来就是“复杂来说”了。

首先来具体了解一下Unity内置的渲染队列吧，这里直接引用了[这篇文章](https://qxsoftware.github.io/Unity-Rendering-Order.html)中对渲染队列的整理，感兴趣也可以去看下原文。从图中可以直观看出，Render Queue的值越大，渲染越靠后。



![20221216235314](https://raw.githubusercontent.com/recaeee/PicGo/main/20221216235314.png)



对于物体的渲染顺序，我们在第一章中知道了我们可以通过在DrawSettings中设置一次DrawRenderers中物体的渲染顺序（例如对于Opaque通常从前往后，对于Transparent通常从后往前）。那结合上图来看，Render Queue设置的渲染顺序和DrawSettings设置的渲染顺序之间的层级关系就显而易见了，**Render Queue确定了不同材质之间的渲染顺序（比如先渲染2000的Opaque物体再渲染3000的Transparent物体），而DrawSettings确定了同一Render Queue下的材质的渲染顺序（比如对于所有Render Queue=2000的Opaque物体从前往后渲染）**。

在[原文](https://qxsoftware.github.io/Unity-Rendering-Order.html)中还讲了一些其他决定渲染顺序的参数以及它们之间的层级关系，在此不多展开，有兴趣的可以看下原文。

好了，到了这里，我们基本上对一个材质的必要组成有所了解（Double Sided Global Illumination目前可忽略），总的来说，**一个材质必须有其使用的Shader，以及一个Render Queue的值**。接下来，我们回到Shader代码中，看一下一个Shader的必要组成部分（再次贴上Shader的代码）。

```c#
Shader "Custom RP/Unlit"
{
    Properties {}

    SubShader
    {
        Pass {}
    }
}
```

Shader的第一行首先为**Shader**的关键字，后面的字符串定义了该Shader的目录以及名字。在Shader中我们首先声明**Properties**关键字，**Properties代码块为Shader对象定义材质属性的信息**，具体定义和语法格式可以看下[官方文档](https://docs.unity3d.com/cn/2021.3/Manual/SL-Properties.html)，我们可以在材质的Inspector窗口中编辑这些值（主要目的），当然也可以通过代码设置其值。我们先不展开，在后续实际编写Shader的时候，我们自然而然就理解这些概念了。

声明完Properties之后，我们会声明**一至多个**SubShader。在一个SubShader中我们会声明**一至多个**Pass。更多的概念，我们从实际编写Shader时一点点理解。

#### 1.2 HLSL程序 HLSL Programs

HLSL的全称为High-Level Shading Language，在Unity中编写Shader类时，对于所有HLSL程序，我们都比u使用**HLSLPROGRAM**和**ENDHLSL**关键字来包裹HLSL代码，其原因是在一个Pass中我们可能还会使用其他语言（如CG）。在本教程中，我们会使用HLSL语言，而不是CG语言，一点原因是CG语言已经过时啦，它已经不更新了，HLSL可以说是现在的主流吧(URP用的主要也是HLSL）。

在这里顺便唠一下，从我们编写Unity代码到调度GPU执行渲染的大致层级架构吧，我们通过在Unity中编写C#（管线部分）、ShaderLab代码，这些代码会调用下一层的Unity C++层（要知道，Unity的底层是用C++写的），对于渲染部分的代码，这些C++代码就会调用下一层的图形API（Vulkan、OpenGL、Metal等），而这些图形API最后就驱动了GPU去执行渲染的一系列操作。简单来说，**Unity代码->Unity C++层->图形API->GPU**。

好了，接下来让我们继续编写Unlit这个Shader，代码如下所示。

```c#
Shader "Custom RP/Unlit"
{
    Properties {}

    SubShader
    {
        Pass {
            HLSLPROGRAM
            #pragma vertex UnlitPassVertex
            #pragma fragment UnlitPassFragment
            #include "UnlitPass.hlsl"
            ENDHLSL
            }
    }
}
```

这一步，我们在Pass中使用HLSLPROGRAM关键字包裹了一段代码，这段代码就是我们需要编写的HLSL代码部分，在其中，我们使用**#pragma**关键字声明了我们的Vertex Shader是“UnlitPassVertex”，Fragment Shader同理。

**pragma**这个词来自希腊语，指一个动作，或者需要做的事情，许多编程语言都使用它来发布特殊的编译器指令。

紧接着，我们需要具体定义UnlitPassVertex和UnlitPassFragment这两个函数，在这里，我们通过**include**关键字来插入一个hlsl文件中的代码内容，在这个hlsl文件中，我们会定义这两个函数。我们是可以直接在pragma下面直接编写这两个函数的，但是考虑代码清晰度等原因，我们会单独使用一个hlsl文件来管理它们。

#### 1.3 HLSL编译保护机制 Include Guard

HLSL文件用于像C#类一样group code，虽然HLSL没有类的概念。对于所有HLSL文件，它们除了代码块的局部作用域之外，只有一个全局作用域，所以一切内容都可以随处访问。

我们再细说一下**include**这个关键字，include会在其位置插入整个hlsl文件的代码，所以如果include同一个文件两次会导致编译错误，为了防止重复include的情况发生，我们在hlsl文件中增加include guard（不太清除中文是什么，总之是一种防止重复的保护机制吧）。

UnlitPass.hlsl文件代码如下。

```c#
#ifndef CUSTOM_UNLIT_PASS_INCLUDED
#define CUSTOM_UNLIT_PASS_INCLUDED
#endif
```
该段代码使用了**宏指令**（宏的定义可自行补充，总之其决定了代码编译时的一些规则）控制，它表示如果没有定义CUSTOM_UNLIT_PASS_INCLUDED这一标识符，则对其进行定义，这样在endif之前的代码，只会在第一次定义该标识符的时候被编译。在Shader类的编写中，我们会经常使用到宏指令，我们对其要比较敏感（Shader的编译也是一门学问）。

#### 1.4 着色器函数 Shader Functions

在这一节中，我们在UnlitPass中定义了顶点着色器函数和片段着色器函数，代码如下。

```c#
#ifndef CUSTOM_UNLIT_PASS_INCLUDED
#define CUSTOM_UNLIT_PASS_INCLUDED

float4 UnlitPassVertex() : SV_POSITION
{
    return 0.0;
}

float4 UnlitPassFragment() : SV_TARGET
{
    return 0.0;
}

#endif
```

在UnlitPassVertex中，我们返回一个float4的变量，通过**SV_POSITION**，我们赋予了这个函数返回值的语义（即返回了顶点的位置信息，至于为什么是float4而不是float3可以去看GAMES101前几课关于齐次坐标的讲解）。

在UnlitPassFragment中，我们也返回了一个float4的变量，代表这个像素的颜色值。

这一节代码很简单，但我们需要注意到两个信息，一个是**数据类型**，一个是**着色器语义**。

对于前者，我们在函数中使用float4的类型，除了**float**类型以外，还有**half**类型，两者区别在于浮点数的精度（float精度更高），half类型的存在意义主要是在移动端GPU上通过降低精度来获取性能上的提升。对于移动端开发，通常来说，我们对位置信息和纹理坐标使用float精度，其他信息都是用half精度。而对于桌面端，即使我们使用了half，GPU最后也是会使用float来代替这个half。（除了这两个精度，还有个fixed，通常等价于half）

对于后者，我们可以看到我们在函数命名的大括号后使用了“: XX_XXX”来告诉我们的GPU，该函数的返回值代表了什么意思，这就是**着色器语义**。[Unity官方文档](https://docs.unity3d.com/cn/2021.3/Manual/SL-ShaderSemantics.html)中对其做了如下说明，**编写HLSL着色器程序时，输入和输出变量需要通过语义来表明其“意图”**。语义是HLSL语言中的标准概念，[微软文档](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-semantics?redirectedfrom=MSDN)中对其做了如下定义：**语义是附加到着色器输入或输出的字符串，用于传达有关参数预期用途的信息**。对于不同的语义，GPU底层就会对这些数据做不同处理。

#### 1.5 空间变换 Space Transformation

在这一节中，我们主要实现的是对顶点Position做一些空间变换，如从模型空间转换到世界空间、从世界空间转换到裁剪空间。关于空间变换这部分知识，可以去看GAMES101,这里不过多展开，讲得非常好。这些函数的代码都比较简单，但是我们更值得注意的是，我们将一系列Pass的输入数据（如摄像机的M、VP矩阵）单独管理在一个UnityInput.hlsl文件中，将通用的空间变换函数管理在一个Common.hlsl文件中。

以下UnityInput.hlsl代码。
```c#
//存储Shader中的一些常用的输入数据
#ifndef CUSTOM_UNITY_INPUT_INCLUDED
#define CUSTOM_UNITY_INPUT_INCLUDED

float4x4 unity_ObjectToWorld;

float4x4 unity_MatrixVP;

#endif
```

以下为Common.hlsl代码。
```c#
//存储一些常用的函数，如空间变换
#ifndef CUSTOM_COMMON_INCLUDED
#define CUSTOM_COMMON_INCLUDED

#include "UnityInput.hlsl"

float3 TransformObjectToWorld(float3 positionOS)
{
    return mul(unity_ObjectToWorld,float4(positionOS,1.0)).xyz;
}

float4 TransformWorldToHClip(float3 positionWS)
{
    return mul(unity_MatrixVP,float4(positionWS,1.0));
}

#endif
```

通过在UnlitPassVertex函数中调用这两个空间变换函数，我们在Scene视图中就可以看到Unity正确绘制了Unlit材质的Mesh。



![20221218090513](https://raw.githubusercontent.com/recaeee/PicGo/main/20221218090513.png)



那我们可能会奇怪，我们在UnityInput.hlsl中只是声明了**unity_ObjectToWorld**和**unity_MatrixVP**，并没有对其进行赋值操作，但是我们从结果可以看出，这两个变换矩阵内已有值，并且是正确的数值。那这些变量是从哪得到的呢？

其实，这些变量被叫做**内置着色器变量**，这些变量的获取是在Unity内置文件中执行的，对于CGPROGRAM着色器，我们不必对其进行特定声明，可以直接使用，但对于HLSLPROGRAM着色器，我们需要自己声明这些变量名来获取当相应的内置变量。

在这里我们可以做一个大胆的尝试，在Unlit.Shader中直接把HLSLPROGRAM关键字替换为CGPROGRAM，然后在UnityInput.hlsl中把两个变换矩阵的声明直接注释掉，我们可以发现结果同上图。而这就是因为CGPROGRAM会自动include一些获取内置着色器变量的文件，而在HLSLPROGRAM中，则需要我们自己声明，更多信息可参考[官方文档对内置着色器变量的说明](https://docs.unity3d.com/cn/2021.3/Manual/SL-UnityShaderVariables.html)。

#### 1.6 SRP核心库 Core Library



![20221218100330](https://raw.githubusercontent.com/recaeee/PicGo/main/20221218100330.png)



我们上述编写的两个空间变换函数其实已经被包括在了一个叫**Core RP Pipeline**的Package中，这个Package同样也定义了许多其他我们常用的方法和功能，所以我们使用这个Package代替我们之前编写的两个函数（这些轮子就不用造啦）



![20221218101044](https://raw.githubusercontent.com/recaeee/PicGo/main/20221218101044.png)



为了使用库中的空间变换函数，我们需要在Common.hlsl中include它的SpcaceTransform.hlsl文件，但由于Core RP库中使用了**UNITY_MATRIX_M**代替了我们的unity_ObjectToWorld，所以在引入它前我们需要使用宏定义**#define UNITY_MATRIX_M unity_ObjectToWorld**，由此，之后**在编译SpaceTransform.hlsl时会自动使用unity_ObjectToWorld来代替UNITY_MATRIX_X**，其他几个变量同理（至于为什么变量名不同，之后会说）。

这里，我也遇到了使用Unity2021的坑，因为其对应Core RP Library为12.1.7，所以有两个新增的变量(UNITY_PREV_MATRIX_M和I_M)需要宏定义，但是我在网上并没找到其确切解释，只能猜测一下进行定义。

以下为Common.hlsl代码。

```c#
//存储一些常用的函数，如空间变换
#ifndef CUSTOM_COMMON_INCLUDED
#define CUSTOM_COMMON_INCLUDED

#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/Common.hlsl"
#include "UnityInput.hlsl"

// float3 TransformObjectToWorld(float3 positionOS)
// {
//     return mul(unity_ObjectToWorld,float4(positionOS,1.0)).xyz;
// }
//
// float4 TransformWorldToHClip(float3 positionWS)
// {
//     return mul(unity_MatrixVP,float4(positionWS,1.0));
// }
//将Unity内置着色器变量转换为SRP库需要的变量
#define UNITY_MATRIX_M unity_ObjectToWorld
#define UNITY_MATRIX_I_M unity_WorldToObject
#define UNITY_MATRIX_V unity_MatrixV
#define UNITY_MATRIX_VP unity_MatrixVP
#define UNITY_MATRIX_P glstate_matrix_projection
//使用2021版本的坑，我们还需要定义两个PREV标识符，才不会报错，但这两个变量具体代表什么未知
#define UNITY_PREV_MATRIX_M unity_ObjectToWorld
#define UNITY_PREV_MATRIX_I_M unity_WorldToObject
//我们直接使用SRP库中已经帮我们写好的函数
#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/SpaceTransforms.hlsl"

#endif
```

以下为UnityInput.hlsl代码。

```c#
//存储Shader中的一些常用的输入数据
#ifndef CUSTOM_UNITY_INPUT_INCLUDED
#define CUSTOM_UNITY_INPUT_INCLUDED

float4x4 unity_ObjectToWorld;
float4x4 unity_WorldToObject;
real4 unity_WorldTransformParams;

float4x4 unity_MatrixVP;
float4x4 unity_MatrixV;
float4x4 glstate_matrix_projection;

#endif
```

在这节，我们通过使用Core RP Library免去了一些通用函数的编写，同时为了编译不报错，需要对一些变量使用宏定义。

#### 1.7 颜色 Color

在这一节，我们的目的是让每个Unlit材质拥有自己的颜色，因此我们在HLSLPROGRAM的区域中声明一个float4类型的uniform变量叫_BaseColor，我们在UnlitPassFragment函数中返回这个颜色值。同时，我们需要让_BaseColor在材质的Inspector窗口中可编辑，我们需要在Unlit.Shader的Properties代码块中声明同名的_BaseColor变量，并赋予其在Inspector视图中的名字，同时赋予其默认值。

我们在Unlit.Shader中的代码如下。

```c#
Shader "Custom RP/Unlit"
{
    Properties
    {
        //[可选：特性]变量名(Inspector上的文本,类型名) = 默认值
        //[optional: attribute] name("display text in Inspector", type name) = default value
        _BaseColor("Color",Color) = (1.0,1.0,1.0,1.0)
    }

    SubShader
    {
        Pass
        {
            HLSLPROGRAM
            #pragma vertex UnlitPassVertex
            #pragma fragment UnlitPassFragment
            #include "UnlitPass.hlsl"
            ENDHLSL
        }
    }
}
```

在这里，我们在Shader的Properties中中定义了_BaseColor的一个属性，类型为**Color**（Color类型在着色器代码中会映射到float4，同时在材质Inspector上显示一个拾色器）。这个_BaseColor会在该Shader的HLSL代码中映射为同名为_BaseColor的float4变量。

这一节同样比较简单，更多关于Properties的信息可以参考[官方文档](https://docs.unity3d.com/cn/current/Manual/SL-Properties.html)。接下来就要正式进入批处理了。

#### 2 批处理 Batching

任何一次Draw Call都需要CPU和GPU之间的通信，如果有大量的数据需要从CPU传递到GPU，那么就会造成GPU闲置的情况（渲染完当前任务后等待CPU传递下一帧数据）。同时，CPU在传递数据的时候是不能做其他事情的。这两个问题都会造成帧率下降。在目前，我们的管线渲染的方法是这样的：**每一个物体对应一个Draw Call**。这样的方法显然是低效且不聪明的，一旦我们数据量大起来，性能是吃不消的。

那么如何提升性能呢？那就需要使用到**批处理（Batching）**技术，而在我看来了，批处理的思路就在于：**归纳出不同数据之间的特性（相似性），将相似（或者冗余）的数据合并或者一次传递给GPU**。也就是说，之前我们在Draw Call时，根本没考虑每个Object之间的关系，如果有两个一模一样的小球，那我们也会当成完全不同的两个小球进行两次Draw Call。

更具体来说，如下图，我在一个场景中拜访了3种颜色的小球，一共**17个小球**，从**Stats**界面中，我们可以看到进行了**18次Batches**（即18次Draw Call，多的一次为天空盒的绘制），每次Draw Call传递一个小球的所有数据，显然这是不聪明的。



![20221219213428](https://raw.githubusercontent.com/recaeee/PicGo/main/20221219213428.png)



#### 2.1 SRP批处理 SRP Batcher

重点来啦！！！我们一点一点理解。

首先我们需要了解什么是Draw Call、SetPass Call（[参考文章](https://zhuanlan.zhihu.com/p/76562300)）。

对于我们已经见过的Draw Call，我们再提一次Draw Call的定义为**CPU调用图像编程接口以命令GPU进行渲染操作**。以及，DrawCall本身只是一条指令，并不怎么耗时。

我们知道在发生一次Draw Call之前，CPU需要做两件事：1，把数据加载到显存，即把网格顶点、法线、纹理等加载到显存中；2，设置渲染状态，包括定义网格使用哪个纹理、哪个材质，还有光源属性等。

那么我们理解了，GPU想要进行一次渲染操作（Draw Call)，粗略看来需要两类信息：1，Mesh顶点；2，材质信息。（实际肯定更复杂）

对于第一类信息，CPU会在Draw Call前提交一次网格数据(顶点、顶点颜色、法线等）。

对于第二类信息，材质信息我们可以把它理解成渲染状态，CPU会在Draw Call前进行一次设置渲染状态。设置渲染状态流程：CPU首先会**判断当前GPU中当前渲染状态是否为需要的渲染状态**，如果是，则不用更改渲染状态，完成设置；如果不是，则设置为新的渲染状态，也叫做**更换渲染状态**，而更换渲染状态也就叫做**SetPass Call**。对于SetPass Call我们必须知道的一点是，**SetPass Call非常耗时！！！**

到此为止，我们就知道了Draw Call、（耗时的）SetPass Call分别都是什么。

而**SRP Batcher的作用就是尽可能减少耗时的SetPass Call**，也就是说SRP Batcher并不会减少Draw Call次数，而是减少每次Draw Call触发SetPass Call的可能性。而SRP Batcher之所以能够实现的大功臣就是CBUFFER。

**CBUFFER**，也就是Constant Buffer。我对CBUFFER的认识比较浅薄，具体大家可以参考[这篇文章](https://zhuanlan.zhihu.com/p/35830868)和[这篇文章](https://zhuanlan.zhihu.com/p/106234207)。"Constant"的含义是该Buffer在着色器程序执行期间其值是恒定的，而CPU可以对该Buffer进行修改。其用处主要是存储对单个物体或对一次绘制中所有顶点都相同的量（比如物体的变换矩阵等），并提供给GPU快速访问。它避免了多次重复传输这些较小的信息给GPU，只在需要更新的时候更新。从硬件层面看，GPU中一般会有一个64KB的常量寄存器用于专门存放Constant Buffer。

我们知道一次DrawRenderers会引发多次Draw Call，每次Draw Call可能会伴随着Set Pass Call（切换渲染状态）。如下图左半部分所示，我们可以看到一个**不使用SRP Batcher时一次DrawRenderers时CPU的（部分）行为**。（说“部分”是因为图中似乎没有提到Mesh数据的提交）

1. 设置SetShaderPass(SetPass Call)，即准备GPU要绘制的物体的材质数据（耗时）。
2. 在RAM中收集Builtin Data(Object的变换矩阵什么的），填充到一个Object CBUFFER（在这里虽然使用了CBuffer，但没利用其能长时间驻留的特性）；
3. 上传Object CBUFFER到GPU的常量缓冲区中；
4. 在RAM中收集材质信息，填充到一个Material CBUFFER（类似第二步）；
5. 上传Materi CBUFFER到GPU的常量缓冲区中；
6. 添加指令“绑定这个Material CBUFFER”；
7. 添加指令“绑定这个Object CBUFFER”;
8. 添加指令Draw Call。
9. 判断下一个要绘制的物体是否与上一个物体使用**同一材质**，如果是，转2；如果不是，转1。

从第9步可以看出，如果我连续绘制多个**不同材质**的物体，那么就会经常触发“1.SetShaderPass”，而第1步就是非常耗时的SetPass Call。



![20221220205859](https://raw.githubusercontent.com/recaeee/PicGo/main/20221220205859.png)



而SRP Batcher的流程就图中右半部分所示，对于一次Draw Renderers发生时CPU的（部分）行为如下。

1. 设置SetShaderPass(SetPass Call)，即准备GPU要绘制的物体的材质数据（耗时）。

2. 将一系列材质信息组成多个persistent Material CBUFFER一次加载到GPU的常量缓冲区中，再将一系列Object各自的一些信息（offset Object data）组成一个large CBUFFER一次加载到GPU的常量缓冲区中。

3. 添加指令“绑定这个persistent Material CBUFFER”。
   
4. 添加指令“绑定这个offset Object data”。

5. 添加指令Draw Call。
   
6. 判断下一个要绘制的物体是否与上一个物体使用**同一着色器变体**，如果是，转2；如果不是，转1。

在DrawRenderers开始时，SRP Batcher一次将本次DrawRenderers中所有（支持SRP Batcher）的材质和每个物体的部分信息（如变换矩阵）都提交到了显存中，然后CPU会在每次发送DrawCall前告诉GPU本次绘制需要使用哪一部分CBuffer。

从图中直观可以看出两点：**1，SRP Batcher减少了每次DrawCall前配置CBuffer的成本**（从准备和上传CBuffer变为了切换绑定CBuffer）；**2，降低了SetPass Call触发的的可能性**（触发条件从不同材质变为了不同着色器变体）。

以下为SRP Batcher的数据传递示意图。



![20221225211607](https://raw.githubusercontent.com/recaeee/PicGo/main/20221225211607.png)



对于SRP Batcher，其底层逻辑可参考[《【Unity】SRP底层渲染流程及原理》](https://zhuanlan.zhihu.com/p/378781638)，我认为的很重要的一点就在于：对于一次DrawRenderers，Unity会遍历所有要绘制的物体（Renderer），然后收集所有出现的材质，整理成一个大的CBuffer，一次传递给GPU的常量缓冲区，同时对这些物体的一些信息（Model矩阵等）也整理成一个大CBuffer，一次传递给GPU的常量缓冲区。

（到了这里我感觉我对SRP Batcher讲的有些混乱了，其中看了一些GPU硬件层面的知识，感觉牵扯到很多知识，我也不是很有把握，如果讲得不对，请多批评指正。我感觉SRP Batcher表面看起来是个很简单的东西，但其实背后原理也挺复杂的。）

对于理论部分，姑且写这么多了，接下来进入实战吧。

为了让我们编写的Unlit.Shader支持SRP Batcher，我们需要把Shader中Properties部分的变量在**HLSL代码段中**用名为**UnityPerMaterial**的CBUFFER段包裹（即定义Material CBUFFER的数据结构），再用名为**UnityPerDraw**的CUBBFER段包裹Shader输入数据中物体的一些变换矩阵（即定义Per Object large buffer中每一小段的结构）。

UnlitPass.hlsl文件代码如下。

```c#
#ifndef CUSTOM_UNLIT_PASS_INCLUDED
#define CUSTOM_UNLIT_PASS_INCLUDED

#include "../ShaderLibrary/Common.hlsl"

//使用Core RP Library的CBUFFER宏指令包裹材质属性，让Shader支持SRP Batcher，同时在不支持SRP Batcher的平台自动关闭它。
//CBUFFER_START后要加一个参数，参数表示该C buffer的名字(Unity内置了一些名字，如UnityPerMaterial，UnityPerDraw。
CBUFFER_START(UnityPerMaterial)
float4 _BaseColor;
CBUFFER_END

float4 UnlitPassVertex(float3 positionOS : POSITION) : SV_POSITION
{
    float3 positionWS = TransformObjectToWorld(positionOS.xyz);
    return TransformWorldToHClip(positionWS);
}

float4 UnlitPassFragment() : SV_TARGET
{
    return _BaseColor;
}

#endif
```

UnityInput.hlsl文件带如下。

```c#
//存储Shader中的一些常用的输入数据
#ifndef CUSTOM_UNITY_INPUT_INCLUDED
#define CUSTOM_UNITY_INPUT_INCLUDED

//这三个变量也使用CBUFFER，使用UnityPerDraw命名该Buffer（UnityPerDraw为Unity内置好的名字）
CBUFFER_START(UnityPerDraw)
float4x4 unity_ObjectToWorld;
float4x4 unity_WorldToObject;
//在定义（UnityPerDraw）CBuffer时，因为Unity对一组相关数据都归到一个Feature中，即使我们没用到unity_LODFade，我们也需要放到这个CBuffer中来构造一个完整的Feature
//如果不加这个unity_LODFade，不能支持SRP Batcher
float4 unity_LODFade;
real4 unity_WorldTransformParams;
CBUFFER_END

float4x4 unity_MatrixVP;
float4x4 unity_MatrixV;
float4x4 glstate_matrix_projection;

#endif
```

关于CBUFFER的命名，可参考文章[《Unity ConstantBuffer的一些解析和注意》](https://zhuanlan.zhihu.com/p/137455866)。简单来说，Unity内置了一些CBUFFER的名字，UnityPerMaterial用于所有材质相关的数据（通常是Properties中所有属性），UnityPerDraw用于所有Unity内置引擎变量（文中是这么说的，但我目前根据其名字理解为对于每个Object被绘制时各自的一些数据），对SRP Bathcer来说，UnityPerDraw中所有的变量如下图所示（图片截取自[视频](https://www.bilibili.com/video/BV1M54y1572J/?t=2673&vd_source=ff0e8ecb1d7ea963eef228f6c1cc6431))。



![20221225215318](https://raw.githubusercontent.com/recaeee/PicGo/main/20221225215318.png)



同时上图也给出了所有block feature的组成，我们在定义UnityPerDraw这个CBUFFER时需要注意，**所有变量在CBUFFER中都必须以组为单位被定义**，意味着CBUFFER中出现的一个变量其对应Block Feature中所有变量都需要同时出现。

此时，我们就支持了SRP Batcher，我们再抓一次帧看下绘制过程。



![20221225220012](https://raw.githubusercontent.com/recaeee/PicGo/main/20221225220012.png)



从图中可以看出，Frame Debuffer将17个使用同个Unlit.Shader但不同材质（不同颜色）的小球的绘制都合并到了一个叫**SRP Batch**的标签中，同时其父标签从原本的RenderLoop.Draw变为了**RenderLoop.DrawSRPBatcher**。从右侧的详细信息可以看到，本次SRP Batch中一共**包括了17次Draw Call**（即17个小球），同时下方还会显示一些合批的相关提示信息。

#### 2.2 更多颜色 Many Colors

在2.1啰嗦了这么多，再简单概括下SRP Batcher的工作流程，即将一些数据（材质、每个物体的变换矩阵等）缓存在GPU显存（常量缓冲区）中，在每次Draw Call时使用一个偏移找到正确的内存地址，得到这些数据用于绘制。（其背后原理并不简单啊，好难~）

接下来，来看一种情况，如果我们需要很多不同颜色的小球，那我们就必须得为每一个颜色都单独创建一个材质，这显然并不方便。Unity提供了一个类型可以用来设置**相同材质下每实例数据**，叫做**MaterialPropertyBlock**。

我们创建一个组件脚本（继承自MonoBehaviour），在其中创建一个静态的MaterialPropertyBlock对象（重复利用），以及Color对象（每个小球的颜色），给每个小球挂载该组件，代码如下。

```c#
using UnityEngine;

//特性：不允许同一物体挂多个该组件
[DisallowMultipleComponent]
public class PerObjectMaterialProperties : MonoBehaviour
{
    //获取名为"_BaseColor"的Shader属性（全局）
    private static int baseColorId = Shader.PropertyToID("_BaseColor");
    
    //每个物体自己的颜色
    [SerializeField] Color baseColor = Color.white;

    //MaterialPropertyBlock用于给每个物体设置材质属性，将其设置为静态，所有物体使用同一个block
    private static MaterialPropertyBlock block;

    //每当设置脚本的属性时都会调用 OnValidate（Editor下）
    private void OnValidate()
    {
        if (block == null)
        {
            block = new MaterialPropertyBlock();
        }

        //设置block中的baseColor属性(通过baseCalorId索引)为baseColor
        block.SetColor(baseColorId, baseColor);
        //将物体的Renderer中的颜色设置为block中的颜色
        GetComponent<Renderer>().SetPropertyBlock(block);
    }

    //Runtime时也执行
    private void Awake()
    {
        OnValidate();
    }
}
```

这里代码同样很简单，但是我们需要注意几个点。

首先，我们通过int类型的baseColorId去找到**名称叫做_BaseColor的着色器属性的唯一标识符**，我们需要牢记的是，**这个int只代表了属性名称，不代表属性本身**，在block.SetColor()方法中通过该int类型的标识符来设置属性比通过string类型的“_BaseColor”来设置属性更高效。Shader.PropertyToID方法具体描述见[官方文档](https://docs.unity3d.com/cn/2021.3/ScriptReference/Shader.PropertyToID.html)。值得一提的是，在一次运行游戏中，一个属性名称的Id是相同的，但是在不同次运行或不同机器上这些数字不同，所以**不要存储或通过网络发送这些数字**。

其次，我们需要知道对于一个带Renderer的Ojbect而言，它使用的着色器属性的优先顺序如下：每实例数据覆盖所有内容；然后使用材质数据；最后，如果这两个地方不存在着色器属性，则使用全局属性值。最终，如果在任何地方都没有定义着色器属性值，则将提供“默认值”（浮点数的默认值为零，颜色的默认值为黑色，纹理的默认值为空的白色纹理）。（该段参考[Unity文档《使用 Cg/HLSL 访问着色器属性》](https://docs.unity3d.com/cn/2021.3/Manual/SL-PropertiesInPrograms.html)）

这一节完毕，我们可以很方便使用同一个材质得到不同颜色小球，效果如下图所示。



![20221226094904](https://raw.githubusercontent.com/recaeee/PicGo/main/20221226094904.png)



但此时如果我们打开Frame Debugger截取一帧看，我们会失望地发现SRP Batcher失效了。



![20221226100641](https://raw.githubusercontent.com/recaeee/PicGo/main/20221226100641.png)



从Frame Debugger中的提示信息我们可以看到，SRP Batch失败的原因是Objects使用了不同的MaterialPropertyBlock的设置。这里我去网上也查了一下，MaterialPropertyBlock并不会产生新的材质实例（Material.SetColor之类的会产生，导致材质占用内存变大），MaterialPropertyBlock本身不支持SRP Batch。

#### 2.3 GPU实例化 GPU Instancing

针对于每个物体自身不同的材质属性，优化Draw Call性能的另一个方法是GPU Instancing，它的作用是**将多个使用相同Mesh相同Material的Objects放在一次Draw Call中绘制**。其中，**不同Object的材质属性可以不同**。CPU会收集每个物体的Transform信息和Material Properties然后构建成一个数组发送给GPU。GPU根据数组迭代绘制每个实体。

因为GPU实例化需要通过数组承载数据，所以我们首先需要在Vertex Shader和Fragment Shader之前添加#pragma multi_compile_instancing，这一指令会让Unity生成两个该Shader的变体，一个支持GPU Instancing，另一个不支持。在材质的Inspector视图下会多一个toggle提供选择使用哪个变体(是否启用GPU Instancing)。



![20221226103414](https://raw.githubusercontent.com/recaeee/PicGo/main/20221226103414.png)



为了使用GPU Instancing，我们会首先include UnityInstancing.hlsl，该hlsl文件为我们做了一些GPU Instancing的准备工作，它重新定义了一些宏用于访问实例化数据数组。

此外，我们使用struct去定义顶点着色器输入和片元着色器输入，一方面代码更整洁，一方面是为了支持GPU Instancing，在结构体中，我们会使用宏（来自UnityInstancing.hlsl）来定义每实例的唯一ID，当然也包括原本的输入数据。对于每实例的材质数据，我们使用**UNITY_INSTANCING_BUFFER**来包裹，类似于CBUFFER的写法，在着色器方法中，使用**UNITY_ACCESS_INSTANCED_PROP**来获取对应名称Buffer段下的属性。

Common.hlsl代码如下。

```c#
//存储一些常用的函数，如空间变换
#ifndef CUSTOM_COMMON_INCLUDED
#define CUSTOM_COMMON_INCLUDED

#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/Common.hlsl"
#include "UnityInput.hlsl"

//将Unity内置着色器变量转换为SRP库需要的变量
#define UNITY_MATRIX_M unity_ObjectToWorld
#define UNITY_MATRIX_I_M unity_WorldToObject
#define UNITY_MATRIX_V unity_MatrixV
#define UNITY_MATRIX_VP unity_MatrixVP
#define UNITY_MATRIX_P glstate_matrix_projection
//使用2021版本的坑，我们还需要定义两个PREV标识符，才不会报错，但这两个变量具体代表什么未知
#define UNITY_PREV_MATRIX_M unity_ObjectToWorld
#define UNITY_PREV_MATRIX_I_M unity_WorldToObject
//我们直接使用SRP库中已经帮我们写好的函数
//在include UnityInstancing.hlsl之前需要定义Unity_Matrix_M和其他宏，以及SpaceTransform.hlsl
//UnityInstancing.hlsl重新定义了一些宏用于访问实例化数据数组
#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/UnityInstancing.hlsl"
#include "Packages/com.unity.render-pipelines.core/ShaderLibrary/SpaceTransforms.hlsl"

#endif
```

UnlitPass.hlsl代码如下。

```c#
#ifndef CUSTOM_UNLIT_PASS_INCLUDED
#define CUSTOM_UNLIT_PASS_INCLUDED

#include "../ShaderLibrary/Common.hlsl"

//使用Core RP Library的CBUFFER宏指令包裹材质属性，让Shader支持SRP Batcher，同时在不支持SRP Batcher的平台自动关闭它。
//CBUFFER_START后要加一个参数，参数表示该C buffer的名字(Unity内置了一些名字，如UnityPerMaterial，UnityPerDraw。
// CBUFFER_START(UnityPerMaterial)
// float4 _BaseColor;
// CBUFFER_END

//为了使用GPU Instancing，每实例数据要构建成数组,使用UNITY_INSTANCING_BUFFER_START(END)来包裹每实例数据
UNITY_INSTANCING_BUFFER_START(UnityPerMaterial)
    //_BaseColor在数组中的定义格式
    UNITY_DEFINE_INSTANCED_PROP(float4,_BaseColor)
UNITY_INSTANCING_BUFFER_END(UnityPerMaterial)

//使用结构体定义顶点着色器的输入，一个是为了代码更整洁，一个是为了支持GPU Instancing（获取object的index）
struct Attributes
{
    float3 positionOS:POSITION;
    //定义GPU Instancing使用的每个实例的ID，告诉GPU当前绘制的是哪个Object
    UNITY_VERTEX_INPUT_INSTANCE_ID
};

//为了在片元着色器中获取实例ID，给顶点着色器的输出（即片元着色器的输入）也定义一个结构体
//命名为Varings是因为它包含的数据可以在同一三角形的片段之间变化
struct Varyings
{
    float4 positionCS:SV_POSITION;
    //定义每一个片元对应的object的唯一ID
    UNITY_VERTEX_INPUT_INSTANCE_ID
};

Varyings UnlitPassVertex(Attributes input)
{
    Varyings output;
    //从input中提取实例的ID并将其存储在其他实例化宏所依赖的全局静态变量中
    UNITY_SETUP_INSTANCE_ID(input);
    //将实例ID传递给output
    UNITY_TRANSFER_INSTANCE_ID(input,output);
    float3 positionWS = TransformObjectToWorld(input.positionOS);
    output.positionCS = TransformWorldToHClip(positionWS);
    return output;
}

float4 UnlitPassFragment(Varyings input) : SV_TARGET
{
    //从input中提取实例的ID并将其存储在其他实例化宏所依赖的全局静态变量中
    UNITY_SETUP_INSTANCE_ID(input);
    //通过UNITY_ACCESS_INSTANCED_PROP获取每实例数据
    return UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _BaseColor);
}

#endif
```

从代码中可见，对于GPU Instancing的实现，我们几乎都是通过UnityInstancing.hlsl定义好的宏指令来实现的，代码上看起来可能会比较让人心生畏惧（我害怕大量的宏指令）。其主要做的事就包括了**每实例数据的定义（数组形式）、每实例唯一ID的构建、每实例数据的获取**。

现在，为我们的材质打开Enable GPU Instancing，然后使用Frame Debugger抓帧一看，GPU Instancing已经实现了。在RenderLoop.Draw中，只产生了一条**Draw Mesh(instanced)**，从详细信息中可以看到，其执行了1次Draw Call，共绘制了17个实例。



![20221226112433](https://raw.githubusercontent.com/recaeee/PicGo/main/20221226112433.png)



参考文章[《关于静态批处理/动态批处理/GPU Instancing /SRP Batcher的详细剖析》](https://zhuanlan.zhihu.com/p/98642798)，GPU Instancing的底层原理也同样使用了GPU常量缓冲区（即Constant Buffer），Unity会将每个实例的位置、缩放、uv偏移等信息保存在显存中的常量缓冲区中，在绘制每个实例时抽取出其对应的信息使用。

但我们知道，显存的常量缓冲区的大小是有限的（通常为64KB），因此**一次GPU Instancing的可合批实例数量取决于目标平台（常量缓冲区的大小）和每实例数据的大小**。

最后再提一下GPU Instancing的使用条件：**使用相同Mesh、相同材质的物体，但各自的材质属性可以不同**。另外，绘制顺序也会打断批处理（例如中间出现其他材质的物体）。

相比SRP Batch来说，我觉得GPU Instancing比较容易理解。

#### 2.4 绘制更多实例网格 Drawing Many Instanced Meshes



![20221226184140](https://raw.githubusercontent.com/recaeee/PicGo/main/20221226184140.png)



GPU Instancing在一次绘制上百个相同物体的时候具有很大的优势，但是我们很少会手动摆放这样上百个物体，一般来说，GPU Instancing在程序化生成树木、草这些东西的时候很好用，在这里我们也试着去随机产生一些小球。

我们创建一个MeshBall组件脚本，这个随机生成小球的脚本很简单，注意我们在这里没有选择去创建每个小球的GameObject，而是选择只提供1个Mesh、1个材质、1023个Transform（每实例数据）、1023个颜色（每实例数据）的方式去**无GameObject化地绘制小球**。

MeshBall.cs代码如下所示。

```c#
using System;
using UnityEngine;
using Random = UnityEngine.Random;


public class MeshBall : MonoBehaviour
{
    //和之前一样，使用int类型的PropertyId代替属性名称
    private static int baseColorId = Shader.PropertyToID("_BaseColor");

    //GPU Instancing使用的Mesh
    [SerializeField] private Mesh mesh = default;
    //GPU Instancing使用的Material
    [SerializeField] private Material material = default;
    
    //我们可以new 1000个GameObject，但是我们也可以直接通过每实例数据去绘制GPU Instancing的物体
    //创建每实例数据
    private Matrix4x4[] matrices = new Matrix4x4[1023];
    private Vector4[] baseColors = new Vector4[1023];

    private MaterialPropertyBlock block;

    private void Awake()
    {
        
        for (int i = 0; i < matrices.Length; i++)
        {
            //在半径10米的球空间内随机实例小球的位置
            matrices[i] = Matrix4x4.TRS(Random.insideUnitSphere * 10f, Quaternion.identity, Vector3.one);
            baseColors[i] = new Vector4(Random.value, Random.value, Random.value, 1f);
        }
    }

    private void Update()
    {
        //由于没有创建GameObject，需要每帧绘制
        if (block == null)
        {
            block = new MaterialPropertyBlock();
            //设置向量属性数组
            block.SetVectorArray(baseColorId, baseColors);
        }

        //一帧绘制多个网格，并且没有创建不必要的游戏对象的开销（一次最多只能绘制1023个实例），材质必须支持GPU Instancing
        Graphics.DrawMeshInstanced(mesh, 0, material, matrices, 1023, block);
    }
}
```

在这里值得注意的有几点。

第一，我们对于MaterialPropertyBlock使用了SetVectorArray的方法来**设置向量属性数组**，参考[官方文档对该函数的描述](https://docs.unity3d.com/cn/2021.3/ScriptReference/MaterialPropertyBlock.SetVectorArray.html)，其向代码块添加向量数组属性，如果具有给定名称的向量数组属性已存在，则替换旧值。另外，**数组添加到代码块之后，其长度不可更改**。

第二，我们使用**Graphics.DrawMeshInstanced**来**每帧**绘制所有小球实例。参考[官方文档对该函数的描述](https://docs.unity3d.com/cn/2021.3/ScriptReference/Graphics.DrawMeshInstanced.html)，该函数用于**一帧绘制多个网格，并且没有创建不必要的游戏对象的开销**。值得注意的是，一次绘制最多只能绘制1023个实例，并且使用的材质必须支持GPU Instancing。这种无GameObject化的绘制方式在很多商业项目中也会被大量使用用于提升性能。

此时，我们运行游戏，我们可以在游戏中看到比较漂亮的景色了。



![20221226191225](https://raw.githubusercontent.com/recaeee/PicGo/main/20221226191225.png)



同时，我们可以看到Hierarchy视图中不存在这些小球的GameObject，我们在Scene场景中也无法选中这些小球。此时我抓帧看了下绘制情况，发现其实并没有只执行一次Draw Call，而是执行了3次，第一次绘制了511个，第二次绘制了511个，第三次绘制了1个。或许我的GPU的常量缓冲区不够大到支持一次绘制1024个小球？我算了一下，一个小球的每实例数据大小为80B，那511个小球需要的数据大小约为40KB，所以我的GPU的常量缓冲区大小大约就是40KB+Mesh数据+材质部分数据，估计总的大小也就是64KB吧。



![20221226191601](https://raw.githubusercontent.com/recaeee/PicGo/main/20221226191601.png)



此外，每个网格的绘制顺序与我们提供数据的顺序相同，另外在这次绘制中不存在排序（基于Camera那种）或视锥体剔除，也就是说即使我们视野中没有小球，这1023个小球依然会被绘制。

#### 2.5 动态批处理 Dynamic Batching

接下来，到了第三个优化Draw Call的方法——动态批处理Dynamic Batching。这是一个比较古老的技术，**它把多个共享相同材质的小Mesh合并成一个大的Mesh进行绘制**。另外，它不支持每Ojbect的材质属性(MaterialPropertyBlock)。

启用动态批处理是这三种方法里最简单的，在CameraRenderer.DrawVisibleGeometry的DrawSettings初始化时，将其成员enableDynamicBatching设置为true，将其成员enableInstancing设置为false，同样关闭SRP Batcher（SRP Batcher具有较高的优先级）。

这部分代码就不展示了。

因为动态批处理更适合小型的Mesh（顶点数比较少），我在场景中摆放了几个Cube，总共使用两种材质（动态批处理仅支持同一材质，我这里用作实验）。



![20221226193840](https://raw.githubusercontent.com/recaeee/PicGo/main/20221226193840.png)



这时候，我打开Frame Debugger一看，还是比较有意思的。



![20221226193956](https://raw.githubusercontent.com/recaeee/PicGo/main/20221226193956.png)



我使用了2种材质，理论上应该只存在2次Draw Call，但实际进行了4次Draw Call，观察每一次的绘制结果可以知道，DrawRenderers中的绘制顺序会打断动态批处理，但是它也不会完全按照DrawRenderers的绘制顺序来绘制（进行了网格合并），总的来说绘制顺序没看出很大规律，不过大致是从前往后的。

通常来说，GPU Instancing比Dynamic Batching效率更高。同时，Dynamic Batching有一些注意事项，比如**涉及不同Scale的网格合并时，不能保证较大网格的法向量是单位长度的**（原因未知），同时，**绘制顺序会发生变化**（上段提到）。

除了这三种方法，还有一个**静态批处理**技术，其工作方法类似，它会在游戏运行前就对标记为静态批处理的GameObjects合并网格。

更多批处理相关的原理，网上的文章也很多，大家有兴趣可以去看看，也可以去看看GPU硬件层面的知识（写这篇文章时，才发现我的硬件知识太弱了，很多地方都感觉写的没把握）。

批处理推荐文章：[SamUncle——《关于静态批处理/动态批处理/GPU Instancing /SRP Batcher的详细剖析》](https://zhuanlan.zhihu.com/p/98642798)

GPU硬件推荐文章：[向往——《深入GPU硬件架构及运行机制》](https://zhuanlan.zhihu.com/p/545056819)

另外，有篇讲SRP原理底层的好文章：[王江荣——《【Unity】SRP底层渲染流程及原理》](https://zhuanlan.zhihu.com/p/378781638)

#### 2.6 配置批处理 Configuring Batching

不同批处理方法在不同的情境下使用，效率都会不一样，我们希望我们的批处理技术是可变化、可配置的。

代码其实特别简单，添加几个配置的变量，然后设置一下就行了，只展示部分代码。

CameraRenderer.cs的Render函数代码如下，添加两个参数。

```c#
    public void Render(ScriptableRenderContext context, Camera camera, bool useDynamicBatching, bool useGPUInstancing)
    {
        //设定当前上下文和摄像机
        this.context = context;
        this.camera = camera;
        
        PrepareBuffer();
        PrepareForSceneWindow();
        
        if (!Cull())
        {
            return;
        }
        
        Setup();
        DrawVisibleGeometry(useDynamicBatching, useGPUInstancing);
        DrawUnsupportedShaders();
        DrawGizmos();
        Submit();
    }
```

CustomRenderPipeline.cs代码如下。

```c#
using UnityEngine;
using UnityEngine.Rendering;

public class CustomRenderPipeline : RenderPipeline
{
    //摄像机渲染器实例，用于管理所有摄像机的渲染
    private CameraRenderer renderer = new CameraRenderer();
    
    //批处理配置
    private bool useDynamicBatching, useGPUInstancing;

    //构造函数，初始化管线的一些属性
    public CustomRenderPipeline(bool useDynamicBatching, bool useGPUInstancing, bool useSRPBatcher)
    {
        this.useDynamicBatching = useDynamicBatching;
        this.useGPUInstancing = useGPUInstancing;
        //配置SRP Batch
        GraphicsSettings.useScriptableRenderPipelineBatching = useSRPBatcher;
    }
    
    //必须重写Render函数，渲染管线实例每帧执行Render函数
    protected override void Render(ScriptableRenderContext context, Camera[] cameras)
    {
        //按顺序渲染每个摄像机
        foreach (var camera in cameras)
        {
            renderer.Render(context, camera, useDynamicBatching, useGPUInstancing);
        }
    }
}
```

CustomRenderPipelineAsset.cs代码如下。

```c#
using UnityEngine;
using UnityEngine.Rendering;

[CreateAssetMenu(menuName = "Rendering/Custom Render Pipeline")]
public class CustomRenderPipelineAsset : RenderPipelineAsset
{
    [SerializeField] private bool useDynamicBatching = true, useGPUInstancing = true, useSRPBatcher = true;
    protected override RenderPipeline CreatePipeline()
    {
        return new CustomRenderPipeline(useDynamicBatching, useGPUInstancing, useSRPBatcher);
    }
}
```

PS：我记得我第一次跟着教程做到这的时候，已经分不清这三个脚本的区别了，当时也不太清楚RenderPipeline和RenderPipelineAsset的区别，现在终于比较清晰了。

最终效果如下所示。



![20221226200429](https://raw.githubusercontent.com/recaeee/PicGo/main/20221226200429.png)



好了，批处理这一坐山终于算爬过去了（其实还有好多没懂的）。

#### 3 透明 Transparency



![20221226204820](https://raw.githubusercontent.com/recaeee/PicGo/main/20221226204820.png)



目前我们的Unlit.shader只支持不透明物体的绘制，我们可以在其上做一些改造，使其同时支持不透明物体和透明物体的绘制。

#### 3.1 混合模式 Blend Modes

不透明物体和透明物体的渲染的主要区别在于是否完全覆盖原像素的颜色。我们可以使用source和destination的混合模式，source指将要被绘制的颜色，destination指当前像素的颜色。

我们首先在Properties中定义_SrcBlend和_DstBlend。这两个值本应该是枚举值，但我们使用Float来定义它们，同时增加特性，在Editor下更方便地设置它们。最后在Pass中设置混合模式，使用方括号包裹_SrcBlend和_DstBlend，这是种古老的语法。

Opaque物体的混合模式为Src=One、Dst=Zero，即新颜色会完全覆盖旧颜色，而Transparent物体的混合模式为Src=SrcAlhpa、Dst=OneMinusSrcAlpha，使用新片元的透明度为权值混合两者。这个比较简单，《Shader入门精要》中也对混合模式有较为详细的介绍。

#### 3.2 不写入深度 Not Writing Depth

透明物体的渲染通常不会写入深度缓冲，我们通过在Pass中设置ZWrite开启或关闭，来控制深度写入模式，同样需要在Properties中加入配置变量。写法类似3.2节。

Unlit.Shader代码如下。

```c#
Shader "Custom RP/Unlit"
{
    Properties
    {
        //[可选：特性]变量名(Inspector上的文本,类型名) = 默认值
        //[optional: attribute] name("display text in Inspector", type name) = default value
        _BaseColor("Color",Color) = (1.0,1.0,1.0,1.0)
        //混合模式使用的值，其值应该是枚举值，但是这里使用float
        //特性用于在Editor下更方便编辑
        [Enum(UnityEngine.Rendering.BlendMode)]_SrcBlend("Src Blend",Float) = 1
        [Enum(UnityEngine.Rendering.BlendMode)]_DstBlend("Dst Blend",Float) = 0
        //深度写入模式
        [Enum(Off,0,On,1)] _ZWrite("Z Write",Float) = 1
    }

    SubShader
    {
        Pass
        {
            //设置混合模式
            Blend [_SrcBlend] [_DstBlend]
            
            HLSLPROGRAM
            //这一指令会让Unity生成两个该Shader的变体，一个支持GPU Instancing，另一个不支持。
            #pragma multi_compile_instancing
            #pragma vertex UnlitPassVertex
            #pragma fragment UnlitPassFragment
            #include "UnlitPass.hlsl"
            ENDHLSL
        }
    }
}
```

#### 3.3 纹理采样 Texturing

我们还需要实现带alpha通道贴图的纹理采样。主要工作就是Properties中定义纹理，在Shader全局变量区中定义纹理的句柄和纹理采样器，在每实例数据中增加纹理坐标的ST变换，然后在着色器输入输出中添加对应变量，在顶点着色器中应用纹理坐标ST变换，在片元着色器中采样，并且使采样颜色结果和baseColor相乘。注意，还需要把透明材质的RenderQueue改为Transparency(3000)。

这里其实没什么值得讲的重点，主要就是注意HLSL的语法，多熟悉一下。

Unlit.shader代码如下。

```c#
Shader "Custom RP/Unlit"
{
    Properties
    {
        //[可选：特性]变量名(Inspector上的文本,类型名) = 默认值
        //[optional: attribute] name("display text in Inspector", type name) = default value
        
        //"white"为默认纯白贴图，{}在很久之前用于纹理的设置
        _BaseMap("Texture", 2D) = "white"{}
        _BaseColor("Color",Color) = (1.0,1.0,1.0,1.0)
        //混合模式使用的值，其值应该是枚举值，但是这里使用float
        //特性用于在Editor下更方便编辑
        [Enum(UnityEngine.Rendering.BlendMode)]_SrcBlend("Src Blend",Float) = 1
        [Enum(UnityEngine.Rendering.BlendMode)]_DstBlend("Dst Blend",Float) = 0
        //深度写入模式
        [Enum(Off,0,On,1)] _ZWrite("Z Write",Float) = 1
    }

    SubShader
    {
        Pass
        {
            //设置混合模式
            Blend [_SrcBlend] [_DstBlend]
            ZWrite [_ZWrite]
            
            HLSLPROGRAM
            //这一指令会让Unity生成两个该Shader的变体，一个支持GPU Instancing，另一个不支持。
            #pragma multi_compile_instancing
            #pragma vertex UnlitPassVertex
            #pragma fragment UnlitPassFragment
            #include "UnlitPass.hlsl"
            ENDHLSL
        }
    }
}
```

UnlitPass.hlsl代码如下。

```c#
#ifndef CUSTOM_UNLIT_PASS_INCLUDED
#define CUSTOM_UNLIT_PASS_INCLUDED

#include "../ShaderLibrary/Common.hlsl"

//使用Core RP Library的CBUFFER宏指令包裹材质属性，让Shader支持SRP Batcher，同时在不支持SRP Batcher的平台自动关闭它。
//CBUFFER_START后要加一个参数，参数表示该C buffer的名字(Unity内置了一些名字，如UnityPerMaterial，UnityPerDraw。
// CBUFFER_START(UnityPerMaterial)
// float4 _BaseColor;
// CBUFFER_END

//在Shader的全局变量区定义纹理的句柄和其采样器，通过名字来匹配
TEXTURE2D(_BaseMap);
SAMPLER(sampler_BaseMap);

//为了使用GPU Instancing，每实例数据要构建成数组,使用UNITY_INSTANCING_BUFFER_START(END)来包裹每实例数据
UNITY_INSTANCING_BUFFER_START(UnityPerMaterial)
    //纹理坐标的偏移和缩放可以是每实例数据
    UNITY_DEFINE_INSTANCED_PROP(float4,_BaseMap_ST)
    //_BaseColor在数组中的定义格式
    UNITY_DEFINE_INSTANCED_PROP(float4,_BaseColor)
UNITY_INSTANCING_BUFFER_END(UnityPerMaterial)

//使用结构体定义顶点着色器的输入，一个是为了代码更整洁，一个是为了支持GPU Instancing（获取object的index）
struct Attributes
{
    float3 positionOS:POSITION;
    float2 baseUV:TEXCOORD0;
    //定义GPU Instancing使用的每个实例的ID，告诉GPU当前绘制的是哪个Object
    UNITY_VERTEX_INPUT_INSTANCE_ID
};

//为了在片元着色器中获取实例ID，给顶点着色器的输出（即片元着色器的输入）也定义一个结构体
//命名为Varings是因为它包含的数据可以在同一三角形的片段之间变化
struct Varyings
{
    float4 positionCS:SV_POSITION;
    float2 baseUV:VAR_BASE_UV;
    //定义每一个片元对应的object的唯一ID
    UNITY_VERTEX_INPUT_INSTANCE_ID
};

Varyings UnlitPassVertex(Attributes input)
{
    Varyings output;
    //从input中提取实例的ID并将其存储在其他实例化宏所依赖的全局静态变量中
    UNITY_SETUP_INSTANCE_ID(input);
    //将实例ID传递给output
    UNITY_TRANSFER_INSTANCE_ID(input,output);
    float3 positionWS = TransformObjectToWorld(input.positionOS);
    output.positionCS = TransformWorldToHClip(positionWS);
    //应用纹理ST变换
    float4 baseST = UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial,_BaseMap_ST);
    output.baseUV = input.baseUV * baseST.xy + baseST.zw;
    return output;
}

float4 UnlitPassFragment(Varyings input) : SV_TARGET
{
    //从input中提取实例的ID并将其存储在其他实例化宏所依赖的全局静态变量中
    UNITY_SETUP_INSTANCE_ID(input);
    //获取采样纹理颜色
    float4 baseMap = SAMPLE_TEXTURE2D(_BaseMap,sampler_BaseMap,input.baseUV);
    //通过UNITY_ACCESS_INSTANCED_PROP获取每实例数据
    float4 baseColor =  UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _BaseColor);
    return baseMap * baseColor;
}

#endif
```

#### 3.4 透明度裁剪 Alpha Clipping

另一类透明物体应用了透明度裁剪，我们也叫Alpha Test，即透明度测试。其原理就是把不透明度低于一定阈值的片元直接舍弃（**透明度测试绘制的物体其实更偏向不透明物体**）。

对于使用透明度测试的物体，我们将起RenderQueue设置为AlphaTest(2450)，SrcBlend=One、DstBlend=Zero，ZWrite=On，很类似于Opaque。AlhpaTest的物体会在所有Opaque物体之后渲染。

透明度裁剪的代码也非常简单，其中Cutoff也是每实例数据。

AlphaTest和Transparent物体绘制效果如下。



![20221226223128](https://raw.githubusercontent.com/recaeee/PicGo/main/20221226223128.png)



#### 3.5 Shader Features

Clip函数会使一些GPU优化失效，我们不希望所有使用Unlit.shader的着色器都包含Clip函数，因为很多材质用不到AlphaTest，因此，我们选择使用**Shader关键字**来控制Shader变体的编译。

我们在Properties中增加一个Shader关键字的Toggle，关键字名为_CLIPPING。

```c#
//Clip的Shader关键字，启用该Toggle会将_Clipping关键字添加到该材质活动关键字列表中，而禁用该Toggle会将其删除
[Toggle(_CLIPPING)] _Clipping("Alpha Clipping",Float) = 0
```

同时在Pass中加入#pragma shader_feature _CLIPPING，这条指令告诉Unity，我们要通过_CLIPPING这个关键字来进行不同着色器变体的编译。

```c#
//告诉Unity启用_CLIPPING关键字时编译不同版本的Shader
#pragma shader_feature _CLIPPING
```

最后，用#if defined条件编译包裹片元着色器中的Clip函数，大功告成。

```c#
float4 UnlitPassFragment(Varyings input) : SV_TARGET
{
    //从input中提取实例的ID并将其存储在其他实例化宏所依赖的全局静态变量中
    UNITY_SETUP_INSTANCE_ID(input);
    //获取采样纹理颜色
    float4 baseMap = SAMPLE_TEXTURE2D(_BaseMap,sampler_BaseMap,input.baseUV);
    //通过UNITY_ACCESS_INSTANCED_PROP获取每实例数据
    float4 baseColor =  UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _BaseColor);
    float4 base = baseMap * baseColor;

    //只有在_CLIPPING关键字启用时编译该段代码
    #if defined(_CLIPPING)
    //clip函数的传入参数如果<=0则会丢弃该片元
    clip(base.a - UNITY_ACCESS_INSTANCED_PROP(UnityPerMaterial, _Cutoff));
    #endif
    
    return base;
}
```

官方文档对[着色器变体和关键字](https://docs.unity3d.com/cn/2019.4/Manual/SL-MultipleProgramVariants.html)做了详细的描述。

**Unity 编译着色器代码片段时，它将为已启用和已禁用关键字的不同组合创建单独的着色器程序。这些各个着色器程序被称为着色器变体**。（但关键字过多，往往会导致着色器变体占用的内存过大，增长为指数级别）

shader_feature指令会生成两个着色器变体，但Unity不会将shader_feature着色器的未用变体包括在最终构建中，会一定程度减少Shader内存。

与之相对的还有multi_compile指令，该指令与shader_feature类似，但最终构建时会将所有变体包括。

根据两者特性，**shader_feature用于材质中设置的关键字，而multi_compile更适合通过代码来全局设置的关键字**（毕竟Unity打包的时候不知道代码是否会启用这些关键字）。

#### 3.6 每实例Cutoff Cutoff Per Object

因为Cutoff时每实例数据，我们需要在PerObjectMaterialProperties脚本中增加该值，实现让每个物体有自己的cutoff值。

这里代码很简单，类似_BaseColor的设置，代码就不贴啦。

#### 3.7 透明度测试实例化 Ball of Alpha-Clipped Spheres

在MeshBall脚本（使用GPU Instancing程序化生成）中，我们同样可以使用透明度测试的材质。

在MeshBall的Awake函数中，我们随机化实例小球的旋转和尺寸，然后对其BaseColor的透明度也随机化。

Meshball.cs中Awake部分代码如下。

```c#
    private void Awake()
    {
        
        for (int i = 0; i < matrices.Length; i++)
        {
            //在半径10米的球空间内随机实例小球的位置
            matrices[i] = Matrix4x4.TRS(Random.insideUnitSphere * 10f,
                Quaternion.Euler(Random.value * 360f, Random.value * 360f, Random.value * 360f),
                Vector3.one * Random.Range(0.5f, 1.5f));
            baseColors[i] = new Vector4(Random.value, Random.value, Random.value, Random.Range(0.5f,1f));
        }
    }
```

最后效果如下，还是挺好看的。



![20221226223921](https://raw.githubusercontent.com/recaeee/PicGo/main/20221226223921.png)



最后需要注意的是，因为Cutoff也是每实例数据，虽然我们没在block中设置每个实例的Cutoff值，Unity也会使用材质的默认值构建成一个1023大小的float数组，其中每个值都是默认值，传递给GPU。

这一章写完~

#### 结束语

这一篇花了很久的时间，其中包括实习工作繁忙和阳了的原因，但更大的原因在于，我对批处理的底层理解的还是太浅，对于Shader在GPU层面的运作机制还不是很了解，接下来我也会多看一些硬件相关的文章，我相信在理解硬件之后，会对Shader理解更清晰一些。好啦，最后提醒自己千万不能怠惰。不知道这篇较长的文章有多少人看完呢，我相信肯定有很多讲的不对的地方，欢迎大家交流意见，或者进行批评指正！

#### 参考

1. 《Shader入门精要》——冯乐乐
2. https://qxsoftware.github.io/Unity-Rendering-Order.html
3. 涩图来自wlop大大 https://space.bilibili.com/26633150
4. 文中引用的知乎文章均已超链，在这里不再列举，感谢各位大佬。