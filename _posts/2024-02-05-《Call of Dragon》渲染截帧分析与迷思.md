---
layout:     post
title:      "《Call of Dragons》渲染截帧分析与迷思"
subtitle:   ""
date:       2024-02-05 12:00:00
author:     "recaeee"
header-img: "img/in-post/default-bg.jpg"
tags:
    - 渲染截帧分析
    - 三转二
---
# 《Call of Dragons》渲染截帧分析与迷思

![20240201015110](https://raw.githubusercontent.com/recaeee/PicGo/main/20240201015110.png)

#### 0 写在前面

《Call of Dragons》是由乐狗研发、莉莉丝发行的一款热门SLG游戏，在这款游戏之前，同样由乐狗研发、莉莉丝发行，推出过另一款热门的SLG游戏《万国觉醒》（也就是ROK），所以《Call of Dragons》也被俗称为ROK2。但不同于ROK的纯2D渲染方案，《Call of Dragons》中使用了3D场景与部分2D渲染结合的方式，从结果上看，取得了较好的画面与性能效果。

那作为引擎/图程/TA，截帧分析已经成了当下必备的技能（别卷了啊喂！再下去要手撕glsl了），所以在这里我也来分析一下《Call of Dragons》中的渲染管线流程吧。

个人掌握的知识与水平有所局限，一定会有没分析到的地方，也希望大家集思广益、一起讨论，如有不对的地方，欢迎批评斧正~

在本文中，你可以了解到：
1. 《Call of Dragons》中一帧绘制中的每个Pass的具体内容。
2. 从截帧分析得到的，我所知的一些，《Call of Dragons》中做出的优化，比如GPU动画方案、图集方案等。
3. 我认为的一些《Call of Dragons》中不足、可以优化的一些点，比如阴影贴图的CloseFit、Bloom的实现等。
4. 《Call of Dragons》中的贴图精度、分辨率的选择与取舍。
5. 三转二方案面临的问题与我的一些迷思。

#### 1 测试环境

模拟器：MuMu模拟器12 （V3.7.5）
截帧工具：Intel GPA
游戏：Call of Dragons (Version:1.0.20.32:247868)
游戏内图像质量：高（共有5个档位，极低、低、中、高、极高）
硬件：i7-10700，RTX2060

#### 2 概要

截帧场景为《Call of Dragons》的内城场景，如下图所示。

![20240130111822](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240130111822.png)

在具体展开之前简单概括下《Call of Dragons》中的渲染管线**要点**：使用了较为传统的前向渲染，对阴影贴图的精度做了优化，使用了较高精度的LUT，内城建筑使用三转二2D片的方案并且结合了图集使用，3D角色使用了GPU动画方案，2D片的贴图精度较高，Bloom采用了传统方案，没有使用抗锯齿，大部分UI元素直接绘制到Framebuffer上。

整个渲染管线流程分析如下。

#### 3 主方向光阴影贴图Pass

整个渲染管线的第一个Pass为**主方向光的ShadowCaster Pass**，**阴影贴图的尺寸为2048x2048**，**阴影贴图的精度为D16_UNROM**，采用D16而不是D24、D32，**节省了阴影贴图的着色压力，同时也降低了移动端的带宽**，这种优化需要考虑到实际的精度是否足够。该Pass渲染了整个内城内的3D物件，主要是内城周围一圈山，从这里也可以知道**内城的建筑都是2D的片**（使用了三转二的方案，建筑没渲染到阴影贴图，关于三转二后面会展开一些内容），同时注意到内城的3D角色也没渲染到阴影贴图。**阴影贴图**的尺寸比较常规，为**2048x2048**，**精度做了优化，使用了D16而不是D24**。ShadowCaster Pass的**视锥体裁剪范围比较大**，实际有用的像素数量会比较少，可能会导致阴影贴图的精度不够。关于视锥体裁剪的方案看起来没有做CloseFit这种优化，无用的空间比较多，但相对保守的方式也能减少一些问题吧，不会出现超出范围的情况。

![20240130113517](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240130113517.png)

#### 4 实时LUT图烘培Pass

第二个Pass为**实时LUT图烘培Pass**，**LUT图的尺寸为1024x32**，精度还是算比较高的（更考虑性能的话，一般可以选择256x16）。LUT图的实时烘培只用了1个Draw Call，没有对这张RT执行Clear操作，因为是整张RT完全覆盖，也不需要考虑Blend，所以不需要Clear，合理。（如果运行时不需要修改LUT的话，这里可以**考虑在首帧烘培LUT图**，后续不再每帧执行LUT pass，或者做一些Lazy处理）。关于LUT的一些知识点可以看我之前的一篇文章[《浅谈Unity URP中LUT的应用》](https://zhuanlan.zhihu.com/p/635956437)。

![20240130112700](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240130112700.png)

#### 5 不透明物体Pass

第四个Pass是**MainCamera Opaque Pass**，也就是URP里渲染主摄像机通常使用的Draw Object Pass（Opaque部分），用于绘制不透明物体。**ColorAttachment格式为R11G11B10_Float**，标准HDR格式，在模拟器上的尺寸为**1920x1080**。**DepthAttachment格式为D24_UNORM_S8_UINT**，尺寸当然也是**1920x1080**，很标准的深度模板缓冲格式。

![20240130114052](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240130114052.png)

![20240130114110](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240130114110.png)

该Pass中首先2条渲染指令为**清理ColorAttachment和DepthAttachment**，常规操作。

![20240130113443](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240130113443.png)

在Opaque Pass中，首先绘制了内城地面上的石块，许多石块采用了Draw Instance的合批，再绘制了内城周围的地皮、草之类的，内城大部分的三角面数都在周围的山、树、石头等，这些的物件最少的到60面，最多的到1.1w面，对于这些物件，顶点Attribute中slot0精度是Float32，存储PositionOS，**slot1、2、3均为Float16的精度**，推测是存储了一些uv信息、法线信息，并且这些物件**使用了法线贴图**（大部分是512x512）来实现更精细的光照。

**内城的角色是3D模型**，使用Draw Instance合批，对相同的Mesh进行了合批。

![20240130161843](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240130161843.png)

内城小型的3D角色的面数在688面，**面数非常少**，顶点Attribute中slot0和slot1使用了Float32精度，**slot2和slot3使用了Float16的精度**（顶点属性推测包含了PositionOS、Normal、Tangent、UV0、骨骼信息？不太确定），角色的贴图中没有法线贴图。

![20240201115544](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240201115544.png)

**角色的动画方案使用了GPU动画方案**，即**使用贴图存储帧动画的M矩阵数组，每帧根据帧索引采样贴图得到对应帧的M矩阵，在顶点着色器中做动画**，从CBuffer中可以明确看到_GPUSkinning2BoneInstancing的CBuffer字段。

**对于移动端**，GPU动画的方案已经算是比较成熟的了，其CPU耗时、GPU耗时、带宽性能目前凭经验来说算是比较好的。传统的**CPU Skinning Mesh**使用多线程实现Mesh动画，其压力点来自于设备的多线程能力，对于一些低端的机器CPU耗时会产生瓶颈。而Unity默认的**Compute Shader计算Skinning**会导致非常严重的带宽问题，并且部分移动端对Compute Shader支持不友好，所以该方案对于移动端设备的带宽不友好，容易导致发热降频，在移动端这种GPU Skinning方法基本很少被采用。

如下图即用贴图存储的帧动画的M矩阵数组。

![20240130162050](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240130162050.png)

接下来绘制**使用Draw Instance绘制了大量的草、树**，一共大概20多个DrawCall用来绘制这些。

绘制完Instance的草、树之后，还会绘制内城的3D人物，应该和之前的人物有所区别（英雄之类？）**人物身上的贴图精度为256x256**，这也算是比较低的一个精度，应该是更偏考虑到性能，牺牲了贴图精度，降低GPU渲染压力和带宽。不过对于这种占屏比低的人物，确实用不上512x512，合理。

#### 6 深度Copy Pass

绘制完内城的3D模型（石头、草、树、角色）之后，应该就是**绘制完Opaque Object之后**，使用一个Draw Call**将D24的DepthAttachment Copy 到了一张尺寸同为1920x1080的D16_UNORM贴图上**。这张Copy出来的深度纹理应该是供后续采样摄像机深度图实现各种效果的地方（透明物体）用的。这里会有一次切RenderTarget的消耗，**需要将Color和Depth都Store下来**，但这里的消耗在所难免，也是**使用D16的RenderTarget来降低消耗**。

![20240130164934](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240130164934.png)

#### 7 透明物体Pass

接下来RenderTarget切回ColorAttachment和DepthAttachment（需要再Load回来），绘制主摄像机的透明物体部分，也是URP中的Draw Object Pass（透明部分），包括内城的水塘等。

![20240130165008](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240130165008.png)

接下来是**内城建筑的渲染**，首先会渲染每个建筑的底座片，底座片贴在地面上，不是Billboard，会采样阴影贴图实现阴影。

![20240130165512](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240130165512.png)

然后是内城地面的道路，**道路用的贴图尺寸为512x1024**，这里的大尺寸比较意外，应该是考虑到道路的占屏比比较大，贴图需要较高精度。

接下来会**使用透明物体Blend的方式绘制3D角色的trick阴影**（之前看到阴影贴图并没有渲染到阴影贴图上，所以绘制内城地面的时候是没有3D角色的阴影的，所以这里把3D角色的阴影使用Blend的方式补回来了，但具体为什么要这么做还不太清楚），算是一种trick吧。关于具体怎么实现的trick阴影，猜测是在顶点着色器中根据传入的顶点和光源方向，在平面上做投影，得到影子的顶点，再在片元着色器中Blend。

![20240131172551](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240131172551.png)

接着是一些透明粒子。

![20240131173718](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240131173718.png)

接下来是**内城建筑2D片的绘制**，大部分建筑片都进行了合批。多个建筑的贴图存储到了一个贴图上，应该也是为合批服务的，图集尺寸为576x1016，不确定这个尺寸是否是动态变化的。

![20240131174508](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240131174508.png)

图集上的每个单个建筑的贴图的边缘看起来都比较怪，类似于做了一圈Clamp，猜测是防止采样超出范围而采到黑色的部分，所以手动在烘培的时候实现了贴图上的Clamp。

![20240131174052](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240131174052.png)

每个建筑还有一个遮罩贴图，每个通道存储了了不同的mask，也是存储在一张同尺寸的图集上（之前误以为是2D法线）。

![20240131174322](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240131174322.png)

接下来是一些灌木丛上的粒子DrawCall，这些DrawCall没合批，估计是没关注到这个问题。

![20240131175317](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240131175317.png)

#### 8 后处理Pass

透明Pass绘制完水塘、内城建筑2D片、粒子结束后，接下来是**后处理Pass**。

上来先是**Bloom**，从半分辨率开始渲染，降采样分辨率为960x540、480x270、240x135、120x67、60x33、30x16、15x8，再依次上采样，**整个过程耗费19个DrawCall**，**Bloom看起来是比较传统的方案**，没做什么优化（没有使用像原神那种合并多张降采样纹理到图集上再一次模糊的方案，这种方案DrawCall会少很多）。

Bloom完之后就是**Uber**了，包括**应用LUT、Bloom**，没有采用额外的DrawCall去实现抗锯齿，看起来是没有使用抗锯齿的（整个画面比较锐利整洁），如果有的话应该也是FXAA。注意到**Uber的RenderTarget是另一张R11G11B10格式的ColorAttachment，而并不是Framebuffer**。

![20240131181009](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240131181009.png)

关于后处理其实也有一些可以做的细枝末节的优化，比如对于后处理里这种全屏的渲染操作，**使用一个三角形代替两个三角形组成的矩形进行渲染**。具体可以参考[《用三角形代替四边形-后处理优化》](https://zhuanlan.zhihu.com/p/128023876)，**使用单个三角形可以增加Cache命中率**，并且提升GPU利用率。像之前截帧《原神》是看到统一使用单个三角形实现的全屏渲染，这个优化在《Call of Dragons》中没有看到。

#### 9 UICamera HUD Pass

后处理结束后是UI摄像机的Pass，首先是**渲染HUD的Pass**，RenderTarget仍然是HDR格式的ColorAttachment。这里是比较奇怪的一点

![20240131191859](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240131191859.png)

**HUD的贴图使用了1024x1024的图集**，图集的好处一方面是**方便合批**，另一方面是可以用于**优化运行时内存**（按界面区分模块，同一界面的UI图片存储到一张图集中，从而图集资源可以随界面开关在内存中加载卸载，不需要常驻）。

![20240131192002](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240131192002.png)

#### 10 UICamera UI Pass

绘制完HUD之后，再绘制用户界面UI。

**在内城的HUD绘制完成之后，首先将ColorAttachment Blit到了R8G8B8A8_UNORM_SRGB格式的Framebuffer上**。通常来说URP的Framebuffer格式为R8G8B8A8_UNROM_SRGB，sRGB格式保证了片元着色器的输出会自动转换到Gamma0.45空间，关于伽马矫正具体可以参考[《Gamma、Linear、sRGB 和Unity Color Space，你真懂了吗？》](https://zhuanlan.zhihu.com/p/66558476)。在输出到屏幕时经过屏幕校准会输出真实的光照。

![20240131193006](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240131193006.png)

接下来**Clear了一下Framebuffer的深度模板缓冲**，深度应该是用不到的，应该是考虑到需要清理模板值，给后续UI使用。

![20240131193313](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240131193313.png)

接下来**绘制用户界面的UI**，从这里开始也就是直接绘制到Framebuffer上了。

![20240131193458](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240131193458.png)

大部分UI贴图资源都整合到了图集，有2048x1024的，有2048x4096的，**整体UI图集的精度是比较高的**。

![20240131193528](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240131193528.png)

#### 11 Built-in输出

最后一个DrawCall将**R8G8B8A8_UNORM_SRGB格式的Framebuffer Blit到 R8G8B8A8_UNORM格式的纹理上**，最后输出到屏幕上。这一步应该是Built-in的，即DX层内置的DrawCall。

![20240131194305](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240131194305.png)

#### 12 （番外篇）三转二2D方案下的一些迷思

在《Call of Dragons》中实现地很出色的一点是一些**3D模型的三转二**，比如内城的建筑，不仔细看的话基本上看不出是2D的片。关于三转二的一些概念可以转到我之前写过的一篇简单文章[《三渲二（三转二）2D动画方案调研》](https://zhuanlan.zhihu.com/p/621733441)，其主要原理就是**使用2D片代替3D模型的渲染**（就是一种Trick的手段），以**在获得较好的美术效果的同时减轻渲染压力**。

**在获得较好的美术效果的同时减轻渲染压力**，这一点是非常重要的（至少在我的感受下），谁没经历过美术制作了一个非常精致的3D模型，然后因为性能不达标，需要再去优化减面呢？通过三转二，性能优化的时候再也不用和美术Battle模型减面的事情了，转，都可以转！

在《Call of Dragons》中，内城建筑的渲染就采用了2D片的方案，但其还稍有一些不同之处，一般来说，**三转二后的2D片更适合搭配正交投影的摄像机渲染**，在正交投影下，无论2D片位于画面中的任何位置，其渲染结果都是一样的，最典型的例子是《部落冲突》。

![20240201010003](https://raw.githubusercontent.com/recaeee/PicGo/main/20240201010003.png)

但是，**在《Call of Dragons》中则使用了透视投影的摄像机渲染**，当然，这也是必然的，因为除了建筑的2D片，游戏中还存在着大量3D物体的渲染（山、树、河流、石头、角色等等），这些3D物体需要透视投影（除非你想要《部落冲突》这种四平八方渲染风格）。

使用透视投影的摄像机搭配三转二的2D片则需要面临一些问题——**三转二2D片搭配透视投影的视觉穿帮**。简单来说，**当真实3D场景与2D片同时出现时，会出现透视关系不统一的情况，3D物体具有强烈的透视关系（近大远小），而三转二的2D片却是正交投影的（无远近关系），当摄像机移动或者两者同时出现并且参照明显时，容易造成穿帮或者视觉观感不佳的情况**。

举一个强透视关系的典型例子，应该有不少人玩过FPS游戏吧（Apex、Csgo、OW等），里面有一项设置是Fov，当Fov调大时，视野内看到的东西会越多，越靠近画面中心的物体会缩得越小，而画面边缘的物体往往会向外拉伸，原本3D空间下的平行线的收缩程度越强，这就是因为透视投影导致的。

再举一个生活中的例子，在摄影中，通常认为50mm左右的焦段是比较符合人眼的观感的（因此比较容易出片），而一些摄影艺术家会为了让画面更有张力、让画面中的线条更为奔放，则会选择一个广角的焦段，比如下面这张照片，可以看到画面中（原本3D空间下）平行的线之间的角度很大，具有强烈的收缩感。

![20240201012408](https://raw.githubusercontent.com/recaeee/PicGo/main/20240201012408.png)

说完强透视关系的现象，再接下来聊聊**三转二2D片搭配透视投影的视觉穿帮原因**（拿出我贫瘠的美术知识），说的可能比较抽象，见谅。

使用2D片代替3D物体的一个限制点在于，**2D片在被烘培时摄像机必须是正交投影的**，原本平行的线在烘培后依然是平行的线，即不具有实际的消失点（从绘画方面来说，平行线的长宽方向的消失点都是在无限远，没有消失点）。

而实际游戏中的**场景中的3D物体因为透视投影的主摄像机的关系，所有平行的线最后会相交到同一消失点**。

这就出现了一个矛盾，场景中的**3D物体上平行线会相交**，而场景中原本应该是3D、但却**被trick成三转二的2D片的物体上的平行线会平行**，那如果理论上有一组平行线，一条在3D物体上，一条在三转二的2D片物体上，这个时候就会因为投影模式的不同，导致**两条线相交在了错误的消失点上**，造成穿帮。并且正交投影下的物体和透视投影下的物体同时出现时，在相机移动相同距离的情况下，偏移量也不同，会造成**视觉上的偏差**，该问题在有参照物的情况下会很明显。

那么如何应对这个问题呢？

**降低透视摄像机的Fov**，降低摄像机Fov意味着透视摄像机越来越接近正交摄像机，即透视关系变弱，整体的透视关系变弱了，镜头边缘的透视关系也就相应变弱了（但这个方法无法彻底解决该问题）。但fov降低意味着在同一条件下视野内的东西会变少，如果想让画面中囊括到和原来差不多的东西，摄像机就需要“离远”原来的物体范围。因此**如何调整相机的Fov和位置成了该应对方法需要攻克的主要难题**。

在《Call of Dragons》中，通过截帧分析，根据VP矩阵反推Fov，推测**内城默认视角下主摄像机的fov为6.9，大世界下主摄像机的Fov为30**，在内城到大世界的过程中会对fov插值过渡（可能不对啊，这里我推测的方法不是很严谨），其中内城fov的降低应该就是为了减少视觉穿帮的程度。

#### 参考与引用
1. https://zhuanlan.zhihu.com/p/66558476
2. https://play.google.com/store/apps/details?id=com.supercell.clashofclans&hl=zh&gl=US
3. https://www.sj33.cn/dphoto/other/201009/24547.html
4. https://store.epicgames.com/es-MX/p/call-of-dragons-d85ee6
5. https://zhuanlan.zhihu.com/p/128023876

#### 日志
更新于2024/2/5：之前误认为了2D片建筑上使用了2D法线贴图，实际上是各通道下的mask信息。同时重新修正了关于三转二2D片遇到的问题的现象解释和分析，之前对该问题有些误解，以为是镜头边缘的畸变。以上均已更新到正文内。

更新于2024/2/24：修正《Call of Dragons》以及《万国觉醒》的研发团队、发行团队信息。