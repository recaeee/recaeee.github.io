---
layout:     post
title:      "浅谈Unity URP中LUT的应用"
subtitle:   ""
date:       2023-06-19 12:00:00
author:     "recaeee"
header-img: "img/in-post/default-bg.jpg"
tags:
    - 浅谈系列
    - LUT
---
# 浅谈Unity URP中LUT的应用



![20230609180012](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20230609180012.png)



---

#### 0 写在前面

环境：
Windows10
Unity版本：Unity 2021.3.12f1
URP版本：Universal RP 12.1.7
PS版本：Adobe PhotoShop CC 2019

本文简单介绍了LUT的概念、用途，LUT贴图的结构，使用LUT的调色原理，在Unity URP中使用内置LUT以及使用自己制作的LUT的方法，其中第2章涉及了URP管线代码部分的解读（偏向程序），第3章涉及URP应用LUT的工作流（偏向美术、TA）。第1章、第2章涉及LUT原理，第3章涉及LUT应用，仅应用需要可直接跳转至第3章。

#### 1 LUT Lookup Texture

#### 1.1 LUT概念

LUT的全称为Lookup Texture（也有叫做Look-up Table颜色查找表的，通常Look-up Table会以纹理的形式存储），**LUT包含了一系列的颜色映射关系**，将输入颜色值映射到输出颜色值。通过使用LUT，可以对图像快速调色、颜色矫正、色彩分级等。

参考[《Unreal4.27 Documentation-Using Lookup Tables (LUTs) for Color Grading》](https://docs.unrealengine.com/4.27/en-US/RenderingAndGraphics/PostProcessEffects/UsingLUTs/)，可以看到LUT可以应用于Color Grading（颜色分级）的效果。



![20230609164032](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20230609164032.png)



题外话，之前一直不太清楚Color Grading、Tone Mapping之间的区别，这里也简单查了下：颜色分级Color Grading是指对整个图像进行颜色调整，包括颜色曲线、饱和度、对比度、色调等等；色调映射Tone Mapping是指将HDR图像映射到LDR显示设备上，即将HDR图像的亮度范围压缩到LDR范围内，尽可能保留细节和对比度。

#### 1.2 LUT贴图的组成

通常来说LUT贴图有两种尺寸256x16和1024x32，尺寸越大，颜色映射的准确度更高。如下图所示为一张1024x32的**原始LUT贴图**（也叫做Neutral LUT，原始LUT贴图的特点是，我们通过一个RGB值采样这张LUT贴图，不考虑离散的误差的话得到的就是这个RGB值）（通过了解原始LUT图，基本上就可以理解所有LUT图的组成和作用了）。



![NewLut32](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGoNewLut32.png)



从上图中可以看到**1024x32的LUT贴图由32个Tile组成，每个Tile的尺寸为32x32**。
首先分析每个Tile的共同点，以下图展示的其中一个Tile为例，其x轴的R值满足从0到1，那已知一个Tile尺寸为32x32，可以得到x轴上共32个像素将R通道0->1均匀离散成32个值，即每向右步进一个像素R通道值增加1/32。同理，y轴将G通道0->1均匀离散成了32个值。
由此可见，**每个Tile的x轴和y轴均会将RG通道0->1离散成32个值**。

然后分析每个Tile的不同点，从1024x32的LUT图中可以看到，越往右的LUT图偏蓝，实际上，**一个32x32的Tile内B通道值都是相同的，而不同Tile的B通道值不同**，一共32个Tile将B通道0->1均匀离散成了32个值。



![20230609151348](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20230609151348.png)



由此，一张1024x32的原始LUT贴图一共将会存储32x32x32=32768个颜色，其将RGB各通道的0->1都均匀离散成32个值，组成了这32768个颜色。可以将**1024x32的LUT贴图理解成一个32x32x32的立体贴图的平铺**。

通过输入一个RGB值，就可以按一定方式采样这张原始LUT图，得到输入的RGB值（如果LUT贴图更改，那就会得到不同的颜色，达到滤镜的效果）。接下来便说说LUT贴图的采样方式。

#### 1.3 LUT贴图的采样方式

在前面提到过，原始LUT贴图的特点是**通过一个RGB值采样这张LUT贴图，不考虑离散的误差的话得到的就是这个RGB值**，从这一点就不能看出LUT贴图的采样方式了。

LUT贴图的采样方式：首先输入一个RGB值，已知LUT贴图上每个Tile对应一个B值（0->1的32**均匀**离散值），根据输入的B值找到最接近的Tile。在该Tile中，直接使用输入的RG值作为该Tile下的uv坐标对Tile进行采样，得到一个RGB值的输出。

原始LUT每个像素其实就反应了其采样坐标，即采样该像素时输入的RGB值。如果将原始LUT贴图的每个像素值做一些改变，再按以上方式采样，就会得到一个不同的颜色值，达到滤镜的效果。


#### 2 URP管线中的LUT LUT in URP

在默认的URP管线中，**只要当Camera开启后处理并且管线的后处理Pass正在被启用，就会执行ColorGradingLUT Pass，无论我们是否真的在调色**。这是一套基于URP自己提供的LUT调色系统，ColorGradingLUT Pass会根据URP内置的后处理调色相关组件每帧生成一张LUT贴图用于最终调色，如果我们没有使用任何后处理调色，该Pass可能会依然存在，并生成一张原始LUT贴图作用于最后调色，即没有任何变化。这里其实挺不科学的，应该是可以针对管线进行一定优化的，在不需要调色的时候直接关闭这个Pass。

ColorGradingLUT Pass在BeforeRenderingPrePasses渲染阶段被执行。

```
UniversalRenderer.Setup()函数中

// TODO: We could cache and generate the LUT before rendering the stack
bool generateColorGradingLUT = cameraData.postProcessEnabled && m_PostProcessPasses.isCreated;

...

if (generateColorGradingLUT)
{
    colorGradingLutPass.Setup(colorGradingLut);
    EnqueuePass(colorGradingLutPass);
}

```



![20230608110712](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20230608110712.png)



从Frame Debugger中可以看到，**该Pass生成了一张256x16的_InternalGradingLut纹理**，纹理格式为R8G8B8A8_UNorm。从ShaderProperties中可以看到生成这张LUT纹理使用到的参数，其中_CurveMaster、_CurveRed、_CurveGreen、_CurveBlue、_CurveHueVsHue、_CurveHueVsSat、_CurveSatVsSat、_CurveLumVsSat这8张纹理是通过后处理组件ColorCurves获得的，在后处理组件ColorCurves中进行参数调整即可改变这几张纹理，从而改变生成的LUT纹理。另外能影响生成这张_InternalGradingLut纹理的后处理组件还包括ChannelMixer、ColorAdjustments、LifgGammaGain、ShadowsMidtonesHighlights、SplitToning、Tonemapping、WhiteBalance。

这张_InternalGradingLut纹理会在UberPost（即后处理合并环节）被应用，在UberPost的Fragment中该_InternalGradingLut用于Color Grading。

```hlsl
// Color grading is always enabled when post-processing/uber is active
{
    color = ApplyColorGrading(color, PostExposure, TEXTURE2D_ARGS(_InternalLut, sampler_LinearClamp), LutParams, TEXTURE2D_ARGS(_UserLut, sampler_LinearClamp), UserLutParams, UserLutContribution);
}
```

在使用LDR Grading的情况下，ApplyColorGrading函数中，会先**执行Tonemap将HDR图像转换到LDR空间**，如果启用了自定义的LUT（即Color Lookup组件，在第3章会提到），会**执行一次自定义的LUT**，最后**执行_InternalGradingLut的调色**。

#### 3 结合PS在URP中应用LUT的工作流 Workflow of applying LUT in URP with PhotoShop

URP提供的后处理组件ChannelMixer、ColorAdjustments、LifgGammaGain、ShadowsMidtonesHighlights、SplitToning、Tonemapping、WhiteBalance、ColorCurves提供了一套URP自己管理的应用LUT方式，在该方法中LUT纹理(即_InternalGradingLut纹理)会在**每帧根据这些后处理组件的参数生成**，并在后处理阶段用于调色。这种方法的好处在于我们**不用手动去PS工具之类的制作LUT贴图，并且可以在运行时动态修改LUT贴图实现动态修改调色效果**；但缺点也很明显，**每帧都需要一个Pass去生成LUT贴图**，如果在游戏运行过程中LUT贴图一直不变，或者不需要高频率修改，其实完全没必要每帧生成LUT贴图（每帧可能生成的都是一样的LUT贴图）。这里可以根据需求通过更改URP代码以优化管线的性能。

因此URP提供了一个**Color Looku**p组件可以让我们**手动配置LUT贴图以应用于调色**，并且可以和前面所说的URP自己管理的LUT一起使用。在Color Lookup中，我们就可以使用我们自己制作的LUT贴图资源。

其实在[Unity5.4的遗留文档](https://docs.unity3d.com/540/Documentation/Manual/script-ColorCorrectionLookup.html)中提到过LUT应用于Unity的工作流，结合PS在URP中应用LUT的主要工作流包括：截取一张游戏截图，导入到PS调色，对原始LUT图应用相同调色生成一张新LUT图，导出新LUT图并导入到Unity中使用。

主要参考了文章[《游戏开发技术杂谈7：Unity后期处理特效·颜色查找表的运用》](https://zhuanlan.zhihu.com/p/373819558)，在该文章中对LUT使用的程序部分、美术部分、TA部分都进行了比较详细的介绍，建议阅读，由于**生成LUT图需要原始LUT图**，在该文章中也提供了脚本生成原始LUT图(1024*32)。

在这里我也对代码简单调整，以生成256*16的LUT图，考虑性能和效果可以自行选择不同尺寸的LUT图。
给出生成256*16的原始LUT图代码如下：
```python
import numpy as np
import cv2


def make_gray_source_16():
    '''生成步长为16的灰度信息'''

    o = [0]
    step = 0
    for _ in range(1, 16):
        step += 17
        o.append(step)
    return o

def make_lattice(source):
    '''生成大小为16的格子'''

    tex2d = np.ndarray((16, 16, 3), dtype=np.int)

    for _ in range(16):tex2d[0:16, _, 2] = source[_]
    for _ in range(16):tex2d[_, 0:16, 1] = source[15 - _]
    return tex2d

def genlut16():

    source = make_gray_source_16()
    lattice = make_lattice(source)

    o = list()
    for _ in range(16):
        _lattice = np.copy(lattice)
        _lattice[0:16, 0:16, 0] = source[_]
        o.append(_lattice)
    return np.hstack(o)
    
tex2d = genlut16()
cv2.imwrite("./NaturalLut16.png", tex2d)
```

这里也直接提供脚本生成的256x16和1024x32的**原始LUT图**。

256x16原始LUT图如下。



![NaturalLut16](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGoNaturalLut16.png)



1024x32原始LUT图如下。



![NewLut32](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGoNewLut32.png)



在说具体工作流之前，首先提一点关键点，在Unity中，**任何一张LUT贴图都需要关闭纹理压缩，并且需要关闭sRGB模式**。在Unity中，**任何一张LUT贴图都需要关闭纹理压缩，并且需要关闭sRGB模式**。在Unity中，**任何一张LUT贴图都需要关闭纹理压缩，并且需要关闭sRGB模式**。（重要的事情说3遍~）



![20230609161639](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20230609161639.png)



**结合PS生成LUT图并在Unity中使用的工作流：**（以1024*32的LUT图为例，以下步骤可以由美术同学完成，关于LUT图在PS中的生成也可以参考[UE4的官方文档](https://docs.unrealengine.com/4.27/en-US/RenderingAndGraphics/PostProcessEffects/UsingLUTs/)。）

(1)任意截取一张原始游戏画面，导入PS，这张游戏画面只是用于参考模拟游戏内的滤镜效果。



![20230609101846](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20230609101846.png)



(2)**新建背景图层**："图层"->"新建"->"图层背景"



![20230609102141](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20230609102141.png)



(3)**接下来制作想要的滤镜效果**："调整"->各种效果。这里也参考着调整了色阶、色彩平衡、亮度/对比度（美术部分我不太熟悉）。



![20230609103005](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20230609103005.png)



(4)**制作新LUT图**：使用PS打开原始LUT图（即不对颜色做任何变化的那张LUT），将第3步得到的所有调整图层复制到原始LUT图，使这些效果作用到原始LUT图，这样就得到了新LUT图。

原始LUT图如下。



![NaturalLut32](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGoNaturalLut32.png)



复制图层制作新LUT图如下。



![20230609103423](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20230609103423.png)



(5)**导出并使用新LUT图**：导出新LUT图，导入Unity，并配置到URP后处理PostProcess Volume的Color Lookup组件的Texture中（没有的话Add Override）。由此实现了新的滤镜效果，并且可以通过Contribution参数调节滤镜的权重。



![20230609110821](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20230609110821.png)

![20230609165136](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20230609165136.png)



如果配置Color Lookup Texture后出现黄色警告，这时候LUT不会生效，根据描述可以看到可能是**LUT贴图开启了sRGB模式**或者其尺寸和管线Asset配置中的**LUT Size值不匹配**，检查一下是哪个问题。



![20230609165407](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20230609165407.png)



如果是LUT Size不匹配，则需要到当前启用的Universal Render Pipeline Assset中设置正确的LUT size（256x16对应16，1024x32对应32）。



![20230609165525](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20230609165525.png)



#### 参考

1. https://zhuanlan.zhihu.com/p/623210058
2. https://zhuanlan.zhihu.com/p/373819558
3. https://docs.unity3d.com/540/Documentation/Manual/script-ColorCorrectionLookup.html
4. https://docs.unrealengine.com/4.27/en-US/RenderingAndGraphics/PostProcessEffects/UsingLUTs/
5. https://zhuanlan.zhihu.com/p/623210058
6. 题图来自画师wlop