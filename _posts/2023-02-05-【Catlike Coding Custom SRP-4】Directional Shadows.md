---
layout:     post
title:      "【Catlike Coding Custom SRP学习之旅——4】Directional Shadows"
subtitle:   ""
date:       2023-02-05 12:00:00
author:     "recaeee"
header-img: "img/in-post/default-bg.jpg"
tags:
    - Custom SRP
---
# 【Catlike Coding Custom SRP学习之旅——4】Directional Shadows
#### 写在前面
接下来到方向光的阴影实现了，阴影实现本身不难，难点在于其锯齿的消除以及阴影级联技术的实现，在很多实际项目中，会针对阴影贴图Shadow Map做性能优化，例如分离动静态物体渲染多张阴影图、旋转阴影图方向等等。另外，阴影也是提升游戏质感的一项重要技术，在完成这一章后，我们就能获取一个比较好的渲染效果了。另外，这一章我会尽力压缩篇幅，主要拓展和深入关键知识点，希望做到即使不是实际去搭建SRP管线，大家也能得到一些收获。

前3章节为长篇文章，考虑到篇幅问题与工作量，从第4章节后半部分开始以及未来章节，考虑以提炼原教程为主，尽量减少篇幅与实际代码，在我的Github工程中包含了对源代码的详细注释，如需深入代码细节可以查看我的Github工程。

以下是原教程链接与我的Github工程（Github上会实时同步最新进度）：

[CatlikeCoding-SRP-Tutorial](https://catlikecoding.com/unity/tutorials/custom-srp/)

[我的Github工程](https://github.com/recaeee/CatlikeCoding-Custom-RP)

--- 



![20230126233237](https://raw.githubusercontent.com/recaeee/PicGo/main/20230126233237.png)



#### 方向光阴影 Directional Shadows

在这一章，我们将实现以下多个效果：渲染和采样Shadow Maps、支持多个方向光阴影、阴影级联、混合渐变过滤阴影。

#### 1 渲染阴影 Rendering Shadows

实现阴影有很多方法，最常见的**实时阴影**技术就是**Shadow Map**（阴影贴图）了，其原理非常简单（推荐观看[《GAMES 101课程相关章节》](https://www.bilibili.com/video/BV1X7411F744/?spm_id_from=333.999.0.0)）。简单来说，就是从光源方向渲染一张深度图，然后根据这张深度图，我们就知道渲染画面用的摄像机中每一个片元处是否被光照射到了，如果没照射到，光照计算结果就是0。

[《Unity官方文档：阴影贴图》](https://docs.unity3d.com/cn/2021.3/Manual/shadow-mapping.html)中对其描述为：**阴影贴图类似于深度纹理，光源生成阴影贴图的方式与摄像机生成深度纹理的方式类似。Unity会在阴影贴图中填充与光线在射到表面之前传播的距离有关的信息（就是光线从光源出发打到物体的距离，类似于摄像机深度），然后对阴影贴图进行采样，以便计算光线射中的游戏对象的实时阴影**。

#### 1.1 阴影设置 Shadow Settings

首先创建一个序列化的ShadowSettings类来提供对阴影贴图的配置属性，主要包括两个属性，一个是**maxDistance**（阴影距离），一个是**TextureSize**（Shadow Map尺寸）。

参考[《Unity官方文档：阴影距离》](https://docs.unity3d.com/cn/2021.3/Manual/shadow-distance.html)，maxDistance决定**Unity渲染实时阴影的最大距离**（与摄像机之间的距离）。另外，**如果当前摄像机远平面小于阴影距离，Unity将使用摄像机远平面而不是maxDistance**。这也是非常合理的，毕竟超出摄像机远平面的物体本身就不会被渲染的。

这里我提出我对**Unity渲染阴影贴图底层逻辑**的简单猜测。

1. 根据maxDistance（或者摄像机远平面）得到一个BoundingBox（也可能是个球型），这个BoundingBox容纳了所有要渲染阴影的物体。
2. 根据这个BoundingBox（也可能是个球星）和方向光源的方向，确定渲染阴影贴图用的正交摄像机的视锥体，渲染阴影贴图。

ShadowSettings.cs代码如下。

```c#
using UnityEngine;

//单纯用来存放阴影配置选项的容器
[System.Serializable]
public class ShadowSettings
{
    //maxDistance决定视野内多大范围会被渲染到阴影贴图上，距离主摄像机超过maxDistance的物体不会被渲染在阴影贴图上
    //其具体逻辑猜测如下：
    //1.根据maxDistance（或者摄像机远平面）得到一个BoundingBox（也可能是个球型），这个BoundingBox容纳了所有要渲染阴影的物体
    //2.根据这个BoundingBox（也可能是个球型）和方向光源的方向，确定渲染阴影贴图用的正交摄像机的视锥体，渲染阴影贴图
    [Min(0f)] public float maxDistance = 100f;

    //阴影贴图的所有尺寸，使用枚举防止出现其他数值，范围为256-8192。
    public enum TextureSize
    {
        _256 = 256,
        _512 = 512,
        _1024 = 1024,
        _2048 = 2048,
        _4096 = 4096,
        _8192 = 8192
    }

    //定义方向光源的阴影贴图配置
    [System.Serializable]
    public struct Directional
    {
        public TextureSize atlasSize;
    }

    //创建一个1024大小的Directional Shadow Map
    public Directional directional = new Directional()
    {
        atlasSize = TextureSize._1024
    };
}
```

#### 1.2 传递配置 Passing Along Settings

我们将会每帧向摄像机传递阴影相关的配置，这样我们可以在运行时实时对其进行修改（虽然教程中不会这么做）。

我们首先需要**将shadowSettings.maxDistance传递给camera的cullingParameters.shadowDistance**，同时为了实现“如果当前摄像机远平面小于阴影距离，Unity将使用摄像机远平面而不是maxDistance”，我们将取shadowSetting.maxDistance和camera.farClipPlane的较小值。

```c#
    bool Cull(float maxShadowDistance)
    {
        //获取摄像机用于剔除的参数
        if (camera.TryGetCullingParameters(out ScriptableCullingParameters p))
        {
            //实际shadowDistance取maxShadowDistance和camera.farClipPlane中较小值
            p.shadowDistance = Mathf.Min(maxShadowDistance, camera.farClipPlane);
            cullingResults = context.Cull(ref p);
            return true;
        }

        return false;
    }
```

ScriptableCullingParameters.shadowDistance即**用于剔除的阴影距离**。我们上一节说过，我们需要**根据主摄像机去生成一个用于渲染阴影贴图的Bounding Box**（要进行剔除，和常规渲染剔除不一样，远平面是shadowDistance）。

#### 1.3 阴影类 Shadows Class

我们将处理方向光阴影的逻辑都放在一个新的Shadows.cs中，然后在Lighting中创建一个Shadows的实例，在Lighting的Setup中执行shadows.Setup。

因此目前逻辑上的包含关系为**CameraRenderer->Lighting->Shadows**。

目前，我们在Shadows中什么都没干，只是获取到需要的context、cullingResult和shadowSettings。

```c#
using UnityEngine;
using UnityEngine.Rendering;

//所有Shadow Map相关逻辑，其上级为Lighting类
public class Shadows
{
    private const string bufferName = "Shadows";

    private CommandBuffer buffer = new CommandBuffer()
    {
        name = bufferName
    };

    private ScriptableRenderContext context;

    private CullingResults cullingResults;

    private ShadowSettings settings;

    public void Setup(ScriptableRenderContext context, CullingResults cullingResults,
        ShadowSettings settings)
    {
        this.context = context;
        this.cullingResults = cullingResults;
        this.settings = settings;
    }

    void ExecuteBuffer()
    {
        context.ExecuteCommandBuffer(buffer);
        buffer.Clear();
    }
}
```

#### 1.4 支持阴影的光源 Lights with Shadows

接下来，我们需要在Shadows类中配置所有支持阴影的光源，并设置一个最大光源数，我们将其设置为1，意味着我们**最多只有一个支持阴影的方向光源**（这通常来说足够了，但注意我们依然可以有多个方向光源，只是只有一个支持阴影）。

在配置过程中，我们始终通过索引号（一个int）来标识每个光源（光源的索引号和cullingResult中光源索引号相同，非常不错~），因此我们也始终通过索引号来表示每个要配置的光源的阴影信息。

在代码中，我们主要通过**ShadowedDirectionalLight**结构体来存储**支持阴影的光源相关信息**。通过ReserveDirectionalShadows方法来配置每个支持阴影的光源，在该方法中，我们会限制光源最大数量、忽略不开启阴影或者阴影强度等于0的光源、忽略不需要渲染阴影的光源。

```c#
    //用于获取当前支持阴影的方向光源的一些信息
    struct ShadowedDirectionalLight
    {
        //当前光源的索引，猜测该索引为CullingResults中光源的索引(也是Lighting类下的光源索引，它们都是统一的，非常不错~）
        public int visibleLightIndex;
    }

    //虽然我们目前最大光源数为1，但依然用数组存储，因为最大数量可配置嘛~
    private ShadowedDirectionalLight[] ShadowedDirectionalLights =
        new ShadowedDirectionalLight[maxShadowedDirectionalLightCount];

    //当前已配置完毕的方向光源数
    private int ShadowedDirectionalLightCount;

    ...

        //每帧执行，用于为light配置shadow altas（shadowMap）上预留一片空间来渲染阴影贴图，同时存储一些其他必要信息
    public void ReserveDirectionalShadows(Light light, int visibleLightIndex)
    {
        //配置光源数不超过最大值
        //只配置开启阴影且阴影强度大于0的光源
        //忽略不需要渲染任何阴影的光源（通过cullingResults.GetShadowCasterBounds方法）
        if (ShadowedDirectionalLightCount < maxShadowedDirectionalLightCount && light.shadows != LightShadows.None && light.shadowStrength > 0f
            && cullingResults.GetShadowCasterBounds(visibleLightIndex, out Bounds b))
        {
            ShadowedDirectionalLights[ShadowedDirectionalLightCount++] = new ShadowedDirectionalLight()
            {
                visibleLightIndex = visibleLightIndex
            };
        }
    }
```

这里使用到了cullingResults.GetShadowCasterBounds((int lightIndex, out Bounds outBounds)方法，该方法传入lightIndex为阴**影投射光源的索引**，outBounds为**要计算的边界**，其返回值为一个bool值，**如果光源影响了场景中至少一个阴影投射对象，则为true**。该方法返回封装了可见阴影投射物的包围盒（包围盒里的物体就是所有需要渲染到阴影贴图上的物体），其可用于动态调整级联范围。

#### 1.5 创建阴影图集 Creating the Shadow Atlas

首先说明一点，为什么这里把阴影贴图叫做ShadowAtlas（阴影图集）而不是ShadowMap。因为，**我们的这一张用作阴影贴图的Texture上，不一定只渲染一个光源的ShadowMap**，可能左上角一块是给光源1的，右上角一块是给光源2的，因此其更符合图集的概念，因此叫做ShadowAtlas。

在这一章节里，我们通过每帧调用CommandBuffer.GetTemporaryRT来**每帧申请一张Render Texture**用作ShadowAtlas。默认申请的RT为RGBA的格式，但对于这张RT，**我们只需要DepthBUffer**，我们通过传入参数RenderTextureFormat.Shadowmap来设置该RT的格式，并且我们将其精度设置为**32bits**。

在这里，我们详细了解下**CommandBuffer.GetTempooraryRT**这一非常重要的方法，我们先来看下我们的实际使用形式。

```c#
        //使用CommandBuffer.GetTemporaryRT来申请一张RT用于Shadow Atlas，注意我们每帧自己管理其释放
        //第一个参数为该RT的标识，第二个参数为RT的宽，第三个参数为RT的高
        //第四个参数为depthBuffer的位宽，第五个参数为过滤模式，第六个参数为RT格式
        //我们使用32bits的Float位宽，URP使用的是16bits
        buffer.GetTemporaryRT(dirShadowAtlasId, atlasSize, atlasSize,
            32, FilterMode.Bilinear, RenderTextureFormat.Shadowmap);
```

CommandBuffer.GetTemporaryRT的作用是**添加“获取临时渲染纹理”命令**。

对于其传入参数，依次进行解释。

1. int nameID:此纹理的着色器属性名称(Shader.PropertyToID)，该方法自动会**使用nameID将这张RT设置为全局着色器属性**。
2. int width:RT像素宽度，-1表示摄像机像素宽度。
3. int height:RT像素高度，-1表示摄像机像素高度。
4. int depthBuffer:深度缓冲区位数（精度）
5. FilterMode filter:纹理过滤模式，对于ShadowAtlas我们使用Linear模式。
6. RenderTextureFormat format:渲染纹理的格式，其默认位ARGB32，但ShadowAtlas不应该使用ARGB32。我们使用**RenderTextureFormat.Shadowmap**，其表示使用原生阴影贴图纹理渲染格式，GPU会自动根据当前显卡支持的格式设置ShadowMap的格式（**对于不支持Shadowmap格式的显卡，阴影将会使用RenderTextrueFormat.Depth格式**）。

另外，我们通过CommandBuffer.GetTemporaryRT申请的RT必须**由我们手动管理其释放**，**使用ReleaseTemporaryRT 并传递相同的 nameID 可释放临时渲染纹理**。未显式释放的所有RT将会在摄像机完成渲染后或在Graphics.ExecuteCommandBuffer完成后被删除。

另外，我们在不渲染阴影时生成一张1x1的ShadowAtlas防止在WebGL 2.0平台上的一些Error（具体Error在代码注释中，不是很重要）。

```c#
    //渲染阴影贴图
    public void Render()
    {
        if (ShadowedDirectionalLightCount > 0)
        {
            RenderDirectionalShadows();
        }
        else
        {
            //如果因为某种原因不需要渲染阴影，我们也需要生成一张1x1大小的ShadowAtlas
            //因为WebGL 2.0下如果某个材质包含ShadowMap但在加载时丢失了ShadowMap会报错
            buffer.GetTemporaryRT(dirShadowAtlasId, 1, 1, 32, FilterMode.Bilinear, RenderTextureFormat.Shadowmap);
        }
    }
```

我们在申请完ShadowAtlas之后，我们需要**告诉GPU接下来的一系列操作的目标RT是ShadowAtlas这张RT**，而不是摄像机的RenderTarget。我们通过**buffer.SetRenderTarget**这一方法来实现。我们来看下我们在代码中的具体使用，然后来分析其逻辑。

```c#
        //告诉GPU接下来操作的RT是ShadowAtlas
        //RenderBufferLoadAction.DontCare意味着在将其设置为RenderTarget之后，我们不关心它的初始状态，不对其进行任何预处理
        //RenderBufferStoreAction.Store意味着完成这张RT上的所有渲染指令之后（要切换为下一个RenderTarget时），我们会将其存储到显存中为后续采样使用
        buffer.SetRenderTarget(dirShadowAtlasId, RenderBufferLoadAction.DontCare, RenderBufferStoreAction.Store);
```

CommandBuffer.SetRenderTarget的作用是**添加“设置活动的渲染目标”命令**。

对于其传入参数，依次进行解释。

1. RenderTargetIdentifier rt:**RT的唯一标识**，我们有多种方式来标识RT，例如RenderTexture类型对象、TemporaryRT的nameID（我们使用的就是nameID，即dirShadowAtlasId）
2. RenderBufferLoadAction loadAction:此枚举描述**在RT激活（加载）时应对其执行的操作**。也就是当GPU开始渲染到该RT时，应对RT中现有内容执行的操作。其枚举值有三个：1，**Load**，保留其现有内容（对于Tile Based的GPU，其成本高昂，应尽量避免）；2，**Clear**，清除其内容；3，**DontCare**，不考虑该RT的现有内容（在Tile Based的GPU上意味着不需要将RenderBuffer内容加载到区块内存中，从而实现性能提升）。
3. Rendering.RenderBufferStoreAction storeAction:此枚举值描述**在GPU渲染到RT后应对其执行的操作**。其枚举值有4个：1，**Store**，需要存储到RAM中的RenderBuffer内容；2，**Resolve**，MSAA相关，先不深入了解；3，**StoreAndResolve**，同2；4，**DontCare**，**不再需要RenderBuffer内容，（在该RT不再作为RenderTarget时）可以直接丢弃**（Tile Base的GPU不会将该RT写出到RAM，从而提供性能提升）。

**RenderBufferLoadAction和RenderBufferStorAction**非常重要，尤其是对于移动端的大多GPU（Tile Based）。其具体食用用法可以参考[《UWA社区的一篇讨论》](https://answer.uwa4d.com/question/5e856425acb49302349a1119)。

我对**RenderTexture与RenderTarget的工作流**的猜测如下：

1. CommandBuffer.GetTemporaryRT会在**显存**上申请一张RT的内存。
2. CommandBuffer.SetRenderTargetRT会将显存上的RT加载到GPU中一块空间非常小的渲染时使用的**高速缓存**中，高速缓存我感觉是类似于**OnChip**（具体文章可参考这篇[《移动端gpu架构中的onchip memory具体是如何运作的？》](https://www.zhihu.com/question/499462755)）这样的东西。
3. 如果loadAction为Clear，在将RT加载到高速缓存后会先对其清除；如果为DontCare则不进行任何预处理。
4. 在我们对这块高速缓存中的RT执行完**所有对它的渲染指令之后**（即下一个RT要使用这块高速缓存了），我们通常会有两种处理方式：1，Store，将其写会到RAM，意味着我们之后还能用它；2，DontCare，直接丢弃这张**高速缓存中的RT**，意味着我们之后就访问不到这张RT了，我想最不容易理解的就是这里的DontCare，因为我们通常会认为我们既然要渲染它，我们之后怎么可能用不到它呢，打个比方吧，我们在渲染完这张RT后，将其Blit到另一张RT上后，可能我们就用不到这张RT了，因为其信息已经在另一张RT上了。

**这些在GPU高速缓存和显存上的加载、存储操作会影响GPU带宽**，因此其概念非常重要（我们也要尽量减少RenderTarget的切换，以此来减少GPU带宽）。

最后，我们会**在完成阴影的一切工作后立刻**手动释放ShadowAtlas这张RT，其具体使用代码如下。

```c#
    //完成因ShadowAtlas所有工作后，释放ShadowAtlas RT
    public void Cleanup()
    {
        buffer.ReleaseTemporaryRT(dirShadowAtlasId);
        ExecuteBuffer();
    }
```

最后给出这节的部分关键代码。

```c#
    //方向光源Shadow Atlas的标识
    private static int dirShadowAtlasId = Shader.PropertyToID("_DirectionalShadowAtlas");

    ...

    //渲染阴影贴图
    public void Render()
    {
        if (ShadowedDirectionalLightCount > 0)
        {
            RenderDirectionalShadows();
        }
        else
        {
            //如果因为某种原因不需要渲染阴影，我们也需要生成一张1x1大小的ShadowAtlas
            //因为WebGL 2.0下如果某个材质包含ShadowMap但在加载时丢失了ShadowMap会报错
            buffer.GetTemporaryRT(dirShadowAtlasId, 1, 1, 32, FilterMode.Bilinear, RenderTextureFormat.Shadowmap);
        }
    }

    //渲染方向光源的Shadow Map到ShadowAtlas上
    void RenderDirectionalShadows()
    {
        //Shadow Atlas阴影图集的尺寸，默认为1024
        int atlasSize = (int)settings.directional.atlasSize;
        //使用CommandBuffer.GetTemporaryRT来申请一张RT用于Shadow Atlas，注意我们每帧自己管理其释放
        //第一个参数为该RT的标识，第二个参数为RT的宽，第三个参数为RT的高
        //第四个参数为depthBuffer的位宽，第五个参数为过滤模式，第六个参数为RT格式
        //我们使用32bits的Float位宽，URP使用的是16bits
        buffer.GetTemporaryRT(dirShadowAtlasId, atlasSize, atlasSize,
            32, FilterMode.Bilinear, RenderTextureFormat.Shadowmap);
        //告诉GPU接下来操作的RT是ShadowAtlas
        //RenderBufferLoadAction.DontCare意味着在将其设置为RenderTarget之后，我们不关心它的初始状态，不对其进行任何预处理
        //RenderBufferStoreAction.Store意味着完成这张RT上的所有渲染指令之后（要切换为下一个RenderTarget时），我们会将其存储到显存中为后续采样使用
        buffer.SetRenderTarget(dirShadowAtlasId, RenderBufferLoadAction.DontCare, RenderBufferStoreAction.Store);
        //清理ShadowAtlas的DepthBuffer（我们的ShadowAtlas也只有32bits的DepthBuffer）,第一次参数true表示清除DepthBuffer，第二个false表示不清除ColorBuffer
        buffer.ClearRenderTarget(true, false, Color.clear);
        ExecuteBuffer();
    }

    //完成因ShadowAtlas所有工作后，释放ShadowAtlas RT
    public void Cleanup()
    {
        buffer.ReleaseTemporaryRT(dirShadowAtlasId);
        ExecuteBuffer();
    }
```

目前为止，虽然我们的摄像机画面会变黑，但我们使用Frame Debugger可以看到目前我们会在渲染物体前先申请一张ShadowAtlas的RT，然后对其执行清除Depth+Stencil的预处理操作。并且，我们可以在Frame Debugger上看到该RT的加载、存储模式。



![20230126233008](https://raw.githubusercontent.com/recaeee/PicGo/main/20230126233008.png)



好啦~到现在为止，我们已经了解对于一个RT的申请、管理、释放操作了，这些操作非常有用，不只是阴影贴图，很多地方我们都需要进行这些操作，因此是非常重要的知识点。

#### 1.6 先渲染阴影贴图 Shadows First



![20230126233431](https://raw.githubusercontent.com/recaeee/PicGo/main/20230126233431.png)



我们需要**在渲染实际摄像机画面之前渲染阴影贴图**，因此我们要先将ShadowAtlas设置为Render Target，渲染阴影贴图，再将摄像机画面设置为Render Target，渲染实际画面。因此在CameraRenderer.Render中调整摄像机Setup的位置。

```c#
        //将光源信息传递给GPU，在其中也会完成阴影贴图的渲染
        lighting.Setup(context, cullingResults, shadowSettings);
        //设置当前摄像机Render Target，准备渲染摄像机画面
        Setup();
```

现在我们能正常渲染出画面了，打开FrameDebugger截帧观察渲染过程，可以看到在MainCamera相关渲染操作之前，会先进行Shadows的相关操作。



![20230127142002](https://raw.githubusercontent.com/recaeee/PicGo/main/20230127142002.png)



我们希望在Frame Debugger中让Shadows标签被Main Camera标签囊括。

```c#
        //在Frame Debugger中将Shadows buffer下的操作囊括到Camera标签下
        buffer.BeginSample(SampleName);
        ExecuteBuffer();
        //将光源信息传递给GPU，在其中也会完成阴影贴图的渲染
        lighting.Setup(context, cullingResults, shadowSettings);
        buffer.EndSample(SampleName);
```



![20230127142526](https://raw.githubusercontent.com/recaeee/PicGo/main/20230127142526.png)



#### 1.7 渲染阴影贴图 Rendering

为了渲染阴影贴图，我们先**根据cullingResults和光源索引来构建一个shadowDrawingSettings**。

ShadowDrawingSettings用来描述**使用哪种splitData渲染哪个阴影光源**。splitData为给定阴影级联等级的剔除信息，目前我们还不需要深入splitData。

接下来调用**cullingResults.ComputeDirectionalShadowMatricesAndCullingPrimitives**来计算当前方向光源渲染阴影贴图用的VP矩阵，然后通过buffer.SetViewProjectionMatrices来配置当前使用的VP矩阵，最后调用context.DrawShadows来渲染阴影贴图。

我们先来看下实际代码。

```c#
    /// <summary>
    /// 渲染单个光源的阴影贴图到ShadowAtlas上
    /// </summary>
    /// <param name="index">光源的索引</param>
    /// <param name="tileSize">该光源在ShadowAtlas上分配的Tile块大小</param>
    void RenderDirectionalShadows(int index, int tileSize)
    {
        //获取当前要配置光源的信息
        ShadowedDirectionalLight light = ShadowedDirectionalLights[index];
        //根据cullingResults和当前光源的索引来构造一个ShadowDrawingSettings
        var shadowSettings = new ShadowDrawingSettings(cullingResults, light.visibleLightIndex);
        //使用Unity提供的接口来为方向光源计算出其渲染阴影贴图用的VP矩阵和splitData
        cullingResults.ComputeDirectionalShadowMatricesAndCullingPrimitives(light.visibleLightIndex,
            0, 1, Vector3.zero,
            tileSize, 0f,
            out Matrix4x4 viewMatrix, out Matrix4x4 projectionMatrix, out ShadowSplitData splitData);
        //splitData包括投射阴影物体应该如何被裁剪的信息，我们需要把它传递给shadowSettings
        shadowSettings.splitData = splitData;
        //将当前VP矩阵设置为计算出的VP矩阵，准备渲染阴影贴图
        buffer.SetViewProjectionMatrices(viewMatrix, projectionMatrix);
        ExecuteBuffer();
        //使用context.DrawShadows来渲染阴影贴图，其需要传入一个shadowSettings
        context.DrawShadows(ref shadowSettings);
    }
```

接下来，我们详细分析下**cullingResults.ComputeDirectionalShadowMatricesAndCullingPrimitives**方法。

根据官方文档，该函数的作用是**计算方向光的视图和投影矩阵以及阴影分割数据**。

接下来看下其传入参数：
1. int activeLightIndex:**当前要计算的光源索引**。
2. int splitIndex:级联索引，阴影级联相关，暂时不深入。
3. int splitCount:级联的数量，阴影级联相关，暂时不深入。
4. Vector3 splitRatio:级联比率，阴影级联相关，暂时不深入。
5. int shadowResolution:**阴影贴图（tile）的分辨率**。
6. float shadowNearPlaneOffset:光源的近平面偏移。
7. **out** Matrix4x4 viewMatrix:**计算出的视图矩阵**。
8. **out** Matrix4x4 projMatrix:**计算出的投影矩阵**。
9. **out** Rendering.ShadowSplitData:计算的级联数据，阴影级联相关，暂时不深入。

我们需要知道，**方向光源是没有位置数据的**（被定义为无限远），因此，我们通过VP矩阵构造一个立方体裁剪空间，将所有要投射阴影的物体通过该VP矩阵变换到该裁剪空间中，然后根据深度信息渲染阴影贴图。

最后，我们通过**context.DrawShadows**来渲染阴影贴图。

context.DrawShadows的作用是为**单个光源调度阴影投射物的绘制**，其传入一个ShadowDrawingSettings，该settings指定了要绘制哪组阴影投射物（级联相关）以及绘制方式。

由此，我们就基本上完成了渲染阴影贴图所用的cpu端发送的一系列渲染指令。

#### 1.8 阴影投射通道 Shadow Caster Pass

虽然我们目前已经每帧告诉GPU去渲染阴影贴图，但我们通过Frame Debugger可以看到在Shadows标签下我们没有渲染任何物体，这是因为**context.DrawShadows只渲染包含ShadowCaster Pass的材质**。因此我们需要在Lit.Shader中增加一个ShadowCaster Pass。**该Pass的"LightMode" = "ShadowCaster"**,ShadowCaster通道用于将对象渲染到阴影贴图中。

新增Pass具体代码如下。

```c#
        //渲染阴影的Pass
        Pass
        {
            //阴影Pass的LightMode为ShadowCaster
            Tags
            {
                "LightMode" = "ShadowCaster"
            }
            //因为只需要写入深度，关闭对颜色通道的写入
            ColorMask 0

            HLSLPROGRAM
            //支持的最低平台
            #pragma target 3.5
            //支持Alpha Test的裁剪
            #pragma shader_feature _CLIPPING
            //定义diffuse项是否使用Premultiplied alpha的关键字
            #pragma multi_compile_instancing
            #pragma vertex ShadowCasterPassVertex
            #pragma fragment ShadowCasterPassFragment
            //阴影相关方法写在ShadowCasterPass.hlsl
            #include "ShadowCasterPass.hlsl"
            ENDHLSL
        }
```

ShadowCasterPass.hlsl代码如下。

```c#
#ifndef CUSTOM_SHADOW_CASTER_PASS_INCLUDED
#define CUSTOM_SHADOW_CASTER_PASS_INCLUDED

#include "../ShaderLibrary/Common.hlsl"

//获取BaseMap用于Clip
TEXTURE2D(_BaseMap);
SAMPLER(sampler_BaseMap);

//为了使用GPU Instancing，每实例数据要构建成数组,使用UNITY_INSTANCING_BUFFER_START(END)来包裹每实例数据
UNITY_INSTANCING_BUFFER_START(UnityPerMaterial)
    //纹理坐标的偏移和缩放可以是每实例数据
    UNITY_DEFINE_INSTANCED_PROP(float4,_BaseMap_ST)
    //_BaseColor在数组中的定义格式
    UNITY_DEFINE_INSTANCED_PROP(float4,_BaseColor)
    //透明度测试阈值
    UNITY_DEFINE_INSTANCED_PROP(float,_Cutoff)
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

Varyings ShadowCasterPassVertex(Attributes input)
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

void ShadowCasterPassFragment(Varyings input)
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
    //到这里就结束了，我们不需要返回任何值，其片元深度会写入阴影贴图的DepthBuffer
}

#endif
```

**ShadowCaster Pass的顶点着色器和片元着色器都非常简单**，我们只需要传递顶点在裁剪空间下的坐标，插值成片元，Clip裁剪掉Alpha Test不通过的片元，最后记录其深度信息到ShadowAtlas的DepthBuffer中就行了，在片元着色器中我们不需要返回值，其深度会自动记录。

此时，我们默认的maxDistance为100，我们打开Frame Debugger可以看到阴影贴图的渲染。



![20230127202452](https://raw.githubusercontent.com/recaeee/PicGo/main/20230127202452.png)



此时，我们阴影贴图的利用率十分低，只有一小块用上了，此时我们降低maxDistance，会使Main Camera上用于阴影渲染的视锥体的远平面拉近，虽然包括的物体可能不变，但会减少视锥体空余的部分。此时，渲染阴影贴图用的立方体裁剪空间自然也就变小，阴影贴图利用率就变大了，以下为maxDistance为15的情况。



![20230127202941](https://raw.githubusercontent.com/recaeee/PicGo/main/20230127202941.png)



再提醒一次，渲染结果是正交模式的，因为渲染的是方向光源的阴影。

#### 1.9 支持更多光源 Multiple Lights

我们将支持阴影的方向光源最大数量设置为4，因此当我们实际有4个方向光源需要渲染阴影时，我们需要将ShadowAltas分为4个Tile，每个光源将其阴影贴图渲染到一个Tile上（**这时就体现了ShadowAtlas图集的概念**）。分块的逻辑就不说了，计算出偏移很简单，唯一比较重要的就是我们通过**buffer.SetViewport**来实际告诉GPU渲染到RenderTarget上的哪一块区域。

首先看下实际应用的部分关键代码。

```c#
    //支持阴影的方向光源最大数（注意这里，我们可以有多个方向光源，但支持的阴影的最多只有4个）
    private const int maxShadowedDirectionalLightCount = 4;
    
    ...

    /// <summary>
    /// 设置当前要渲染的Tile区域
    /// </summary>
    /// <param name="index">Tile索引</param>
    /// <param name="split">Tile一个方向上的总数</param>
    /// <param name="tileSize">一个Tile的宽度（高度）</param>
    void SetTileViewport(int index, int split, float tileSize)
    {
        Vector2 offset = new Vector2(index % split, index / split);
        buffer.SetViewport(new Rect(offset.x * tileSize, offset.y * tileSize,tileSize,tileSize));
    }

```

我们来详细分析下**buffer.SetViewport**方法。

CommandBuffer.SetViewport的作用是**添加用于渲染视口的命令**。默认情况下，在Render Target更改后，视口将设置为整个Render Target。**此命令可用于将未来的渲染限制为指定的像素矩阵**，其传入参数为Rect pixelRect:视口矩形（像素坐标）。

由此，我们放置4个支持阴影的方向光源得到的ShadowAtlas如下图所示。



![20230127211722](https://raw.githubusercontent.com/recaeee/PicGo/main/20230127211722.png)



#### 2 采样阴影贴图 Sampling Shadows



![20230127205507](https://raw.githubusercontent.com/recaeee/PicGo/main/20230127205507.png)



目前我们已经能成功渲染出阴影贴图，接下来就是需要在光照Pass里采样阴影贴图，来判断片元是否在阴影中了。

#### 2.1 阴影矩阵 Shadow Matrices

为了判断Lit中的片元是否在阴影中，我们首先**需要根据片元的世界坐标，将其转换成ShadowAtlas上的像素坐标**，采样，然后比较值。

坐标转换的方法很简单，我们首先让片元的世界坐标左乘光源裁剪空间的VP矩阵，转换到光源裁剪空间，然后将其坐标从[-1,1]缩放到[0,1]，然后根据Tile偏移和缩放到对应光源的Tile上，就可以进行采样了。

在这里给出关键代码。

```c#
    //方向光源Shadow Atlas、阴影变化矩阵数组的标识
    private static int dirShadowAtlasId = Shader.PropertyToID("_DirectionalShadowAtlas"),
        dirShadowMatricesId = Shader.PropertyToID("_DirectionalShadowMatrices");
    //将世界坐标转换到阴影贴图上的像素坐标的变换矩阵
    private static Matrix4x4[] dirShadowMatrices = new Matrix4x4[maxShadowedDirectionalLightCount];

    ...

    /// <summary>
    /// 设置当前要渲染的Tile区域
    /// </summary>
    /// <param name="index">Tile索引</param>
    /// <param name="split">Tile一个方向上的总数</param>
    /// <param name="tileSize">一个Tile的宽度（高度）</param>
    Vector2 SetTileViewport(int index, int split, float tileSize)
    {
        Vector2 offset = new Vector2(index % split, index / split);
        buffer.SetViewport(new Rect(offset.x * tileSize, offset.y * tileSize,tileSize,tileSize));
        return offset;
    }

        Matrix4x4 ConvertToAtlasMatrix(Matrix4x4 m, Vector2 offset, int split)
    {
        //如果使用反向Z缓冲区，为Z取反
        if (SystemInfo.usesReversedZBuffer)
        {
            m.m20 = -m.m20;
            m.m21 = -m.m21;
            m.m22 = -m.m22;
            m.m23 = -m.m23;
        }
        //光源裁剪空间坐标范围为[-1,1]，而纹理坐标和深度都是[0,1]，因此，我们将裁剪空间坐标转化到[0,1]内
        //然后将[0,1]下的x,y偏移到光源对应的Tile上
        float scale = 1f / split;
        m.m00 = (0.5f * (m.m00 + m.m30) + offset.x * m.m30) * scale;
        m.m01 = (0.5f * (m.m01 + m.m31) + offset.x * m.m31) * scale;
        m.m02 = (0.5f * (m.m02 + m.m32) + offset.x * m.m32) * scale;
        m.m03 = (0.5f * (m.m03 + m.m33) + offset.x * m.m33) * scale;
        m.m10 = (0.5f * (m.m10 + m.m30) + offset.y * m.m30) * scale;
        m.m11 = (0.5f * (m.m11 + m.m31) + offset.y * m.m31) * scale;
        m.m12 = (0.5f * (m.m12 + m.m32) + offset.y * m.m32) * scale;
        m.m13 = (0.5f * (m.m13 + m.m33) + offset.y * m.m33) * scale;
        m.m20 = 0.5f * (m.m20 + m.m30);
        m.m21 = 0.5f * (m.m21 + m.m31);
        m.m22 = 0.5f * (m.m22 + m.m32);
        m.m23 = 0.5f * (m.m23 + m.m33);
        return m;
    }
```

其中，我们会考虑到有些图形API会采用**反向深度缓冲**（为了充分利用深度精度），因此我们判断这种情况并进行矫正。另外采用多行乘法和加法的原因是避免矩阵操作带来的无效运算。

到了这一步，我们就实现了在CPU端将至多4个阴影变换矩阵传递给GPU（虽然在GPU端还没接收），这样，我们在GPU中就可以将片元的世界坐标通过这4个阴影变换矩阵转换到ShadowAtlas上每个Tile上的像素坐标。

#### 2.2 存储每个光源的阴影数据 Storing Shadow Data Per Light

我们虽然告诉了GPU如何对片元采样到每个shadow Tile，但我们没有告诉GPU**哪个光源对应哪个Tile的阴影变换矩阵**。因此，我们还需要在CPU端传递每个光源的阴影数据_DirectionalLightShadowData给GPU，其数据类型为**Vector2**，包含光源的**阴影强度**（0为不渲染阴影，以及其**Shadow Tile索引**（也可以认为是支持阴影的光源索引，默认值为0）。

代码很简单，就是多传递个Vector2的数据给GPU，就不贴代码了~

#### 2.3 阴影HLSL文件 Shadows HLSL File

我们创建一个Shadows.hlsl文件，在其中我们会接收cpu端传来的阴影贴图和每个Shadow Tile的阴影变换矩阵，我们也会在其中实现采样阴影贴图。其代码如下所示。

```c#
//用来采样阴影贴图
#ifndef CUSTOM_SHADOWS_INCLUDED
#define CUSTOM_SHADOWS_INCLUDED

//宏定义最大支持阴影的方向光源数，要与CPU端同步，为4
#define MAX_SHADOWED_DIRECTIONAL_LIGHT_COUNT 4

//接收CPU端传来的ShadowAtlas
//使用TEXTURE2D_SHADOW来明确我们接收的是阴影贴图
TEXTURE2D_SHADOW(_DirectionalShadowAtlas);
//阴影贴图只有一种采样方式，因此我们显式定义一个阴影采样器状态（不需要依赖任何纹理），其名字为sampler_linear_clamp_compare(使用宏定义它为SHADOW_SAMPLER)
//由此，对于任何阴影贴图，我们都可以使用SHADOW_SAMPLER这个采样器状态
//sampler_linear_clamp_compare这个取名十分有讲究，Unity会将这个名字翻译成使用Linear过滤、Clamp包裹的用于深度比较的采样器
#define SHADOW_SAMPLER sampler_linear_clamp_compare
SAMPLER_CMP(SHADOW_SAMPLER);

//接收CPU端传来的每个Shadow Tile的阴影变换矩阵
CBUFFER_START(_CustonShadows)
    float4x4 _DirectionalShadowMatrices[MAX_SHADOWED_DIRECTIONAL_LIGHT_COUNT];
CBUFFER_END

#endif

```

在其中，我们通过宏定义了一个SAMLLER_CMP，对于任何阴影贴图，我们都可以使用这一个采样器状态来进行采样。参考[知乎文章《Unity SRP搞事（四）影》](https://zhuanlan.zhihu.com/p/67749158)，**通常来说，纹理和其采样器是成对出现的**，即一个纹理对应一个采样器。但是，因为对于任何阴影贴图，只有一种采样方法，因此我们只需要定义唯一一个SAMPLER_CMP。

```c#
#define SAMPLER(samplerName)                  SamplerState samplerName
#define SAMPLER_CMP(samplerName)              SamplerComparisonState samplerName
```

根据以上宏定义可以看到，SAMPLE_CMP实际会定义一个SamplerComparisonState采样器状态.

**SAMPLER_CMP**是一个特殊的采样器，我们可以通过与其匹配的SampleCmp函数来采样深度图。参考[《微软官方文档：SampleCmp》](https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-to-samplecmp)，该函数将采样**一块纹素区域**（不是仅仅一个纹素），对于每个纹素，将其采样出来的深度值与给定比较值进行比较，返回0或者1。最后**将这些纹素的每个0或1结果通过纹理过滤模式混合在一起**（不是平均值这么简单的混合），最后将**一个[0,1]的float类型混合结果**返回给着色器。这个深度比较结果，相比只采样一个纹素效果更好。

注意，我们的**定义的比较采样器名称为sampler_linear_clamp_compare**，这个名称是具有实际意义的。参考[《Unity官方文档：内联采样器状态》](https://docs.unity3d.com/cn/current/Manual/SL-SamplerStates.html)，**Unity会识别采样器名称中的某些模式**，其对于直接在着色器中声明简单硬编码采样状态很有用。sampler_linear_clamp_compare意味着我们会创建一个采用**Linear过滤模式**、**Clamp纹理包裹模式**的**用于深度比较**的采样器。

#### 2.4 采样阴影

在这一节，我们就需要实现采样阴影了。在采样阴影之前，我们需要先接收到CPU端发送过来的每个光源的**阴影强度**和**Tile索引**。此外，我们还需要得到片元的**世界坐标**用于转换到阴影贴图上的像素坐标。

在接收完这些信息后，我们通过**SAMPLE_TEXTURE2D_SHADOW**来采样ShadowAtlas，其会返回[0,1]的值，也就是上一步SampleCmp函数得到的结果，该值反映了阴影的衰减度**ShadowAttenuation**,0意味着片元完全在阴影中，1意味着片元完全不在阴影中，而**中间值意味着片元有部分在阴影中**（上一节说过SampleCmp的原理）。

Shadows.hlsl中关键代码如下。

```c#
//每个方向光源的的阴影信息（包括不支持阴影的光源，不支持，其阴影强度就是0）
struct DirectionalShadowData
{
    float strength;
    int tileIndex;
};

//采样ShadowAtlas，传入positionSTS（STS是Shadow Tile Space，即阴影贴图对应Tile像素空间下的片元坐标）
float SampleDirectionalShadowAtlas(float3 positionSTS)
{
    //使用特定宏来采样阴影贴图
    return SAMPLE_TEXTURE2D_SHADOW(_DirectionalShadowAtlas,SHADOW_SAMPLER,positionSTS);
}

//计算阴影衰减值，返回值[0,1]，0代表阴影衰减最大（片元完全在阴影中），1代表阴影衰减最少，片元完全被光照射。而[0,1]的中间值代表片元有一部分在阴影中
float GetDirectionalShadowAttenuation(DirectionalShadowData data, Surface surfaceWS)
{
    //忽略不开启阴影和阴影强度为0的光源
    if(data.strength <= 0.0)
    {
        return 1.0;
    }
    //根据对应Tile阴影变换矩阵和片元的世界坐标计算Tile上的像素坐标STS
    float3 positionSTS = mul(_DirectionalShadowMatrices[data.tileIndex], float4(surfaceWS.position,1.0)).xyz;
    //采样Tile得到阴影强度值
    float shadow = SampleDirectionalShadowAtlas(positionSTS);
    //考虑光源的阴影强度，strength为0，依然没有阴影
    return lerp(1.0,shadow,data.strength);
}
```

这里，注意我们在GetDirectionalShadowAttenuation方法中使用了**if分支语句**，普遍来说，**分支语句会大幅降低Shader的性能**，这是因为GPU的高度并行化导致的，其SIMD或SIMT的硬件架构决定了图元上的每个片元都需要根据其自己的数据执行相同的指令，这也就意味着，如果有一些片元走进了if的一个分支，那么另一些没走进if的片元也会跟着一起去执行分支里的指令（即使它们在空转，即不实际执行分支中的语句）。具体的GPU硬件原理可以参考[向往大佬的《深入GPU硬件架构及运行机制》](https://zhuanlan.zhihu.com/p/545056819)，即使没完全看懂这篇文章，也会收获到许多GPU知识，是一篇非常好的文章，强烈推荐~

#### 2.5 光源衰减 Attenuating Light

目前，我们已经知道了ShadowAttenuation（阴影衰减），因此我们就可以计算出**考虑阴影后，光源打在片元上的能量比例**Light Attenuation（光源衰减），根据定义显而易见，LightAttenuation就等于ShadowAttenuation。在这一节要明确，我们会**对每个片元去构造属于这个片元自身的光源Light结构体**，对于每个片元的Light，Light的color和direction通常相同，但是attenuation不同（即考虑阴影），而不是将Light作为Shader的Uniform变量（虽然CPU传进来少量关于光源的CBUFFER数据，但是依然会对每个片元去构造器自己的Light)。最后，我们让物体表面接收到的光能量乘以Light Attenuation就成功让物体表面根据阴影衰减度接收到不同的光能量，因此实现阴影的效果。

这里给出Light.hlsl中的关键代码。

```c#
//构造一个光源的ShadowData
DirectionalShadowData GetDirectionalShadowData(int lightIndex)
{
    DirectionalShadowData data;
    //阴影强度
    data.strength = _DirectionalLightShadowData[lightIndex].x;
    //Tile索引
    data.tileIndex = _DirectionalLightShadowData[lightIndex].y;
    return data;
}

//对于每个片元，构造一个方向光源并返回，其颜色与方向取自常量缓冲区的数组中index下标处
Light GetDirectionalLight(int index, Surface surfaceWS)
{
    Light light;
    //float4的rgb和xyz完全等效
    light.color = _DirectionalLightColors[index].rgb;
    light.direction = _DirectionalLightDirections[index].xyz;
    //构造光源阴影信息
    DirectionalShadowData shadowData = GetDirectionalShadowData(index);
    //根据片元的强度
    light.attenuation = GetDirectionalShadowAttenuation(shadowData, surfaceWS);
    return light;
}
```

到此为止，我们就成功实现了初步的阴影效果，如下图所示。



![20230129225657](https://raw.githubusercontent.com/recaeee/PicGo/main/20230129225657.png)



从效果图中可以看到，我们的阴影渲染有些问题。在本该没有阴影的物体表面产生了类似于摩尔纹一般的阴影花纹，以及还会存在一些潜在的问题（一个光源的阴影可能采样到别的光源的Tile上）。我们会在后续的小节解决这些问题。

#### 3 级联阴影贴图 Cascaded Shadow Maps

参考[《Unity官方文档——阴影级联》](https://docs.unity3d.com/cn/2021.3/Manual/shadow-cascades.html)阴影级联Shadow Cascades是一项非常有效的技术来充分利用阴影贴图生成高质量的阴影效果，并且有效减少阴影锯齿，其在很多商业项目中都有所应用。**阴影级联是只针对方向光源的技术**，首先来说为什么需要阴影级联。

由于方向光源在渲染阴影贴图时使用正交投影，因此其阴影贴图每个Texel覆盖的（世界空间下的）实际面积都是相同的。而游戏中的主摄像机往往是透视投影的，而透视投影会造成这样的现象：视锥体内，距离主摄像机近的区域最终会被更多像素覆盖，而远处则相反。但在采样阴影贴图时，近处远处的像素都会去采样相同精度的Texel，这样就导致近处的像素可能产生阴影锯齿，也就是导致了**摄像机像素与阴影贴图纹素实际覆盖面积不匹配**，如下图所示。



![20230201222912](https://raw.githubusercontent.com/recaeee/PicGo/main/20230201222912.png)



因此，对于一个方向光源，我们对距离摄像机的近端区域（面积较小）和远端区域（面积较大）渲染两张相同尺寸的阴影贴图（级联区域Cascades），就会导致近端区域的阴影贴图精度高，远端的精度低，使采样时纹素与像素覆盖面积匹配。这就是阴影级联的大致工作原理，其示意图如下图所示。



![20230201223536](https://raw.githubusercontent.com/recaeee/PicGo/main/20230201223536.png)



使用的级联越多，透视锯齿对阴影的影响就越小，增加级联数量会增加渲染开销，但**此开销仍然会低于使用整张高分辨率的阴影贴图**。

#### 3.1 设置 Settings

我们通过控制每次渲染阴影贴图时的maxShadowDistance来控制级联阴影渲染的区域。

对于方向光源阴影信息，增加参数cascadeCount来控制其级联数量（Unity默认最多支持4个级联，因此cascadeCount最大值为4），同时定义每一级级联的覆盖范围。

#### 3.2 渲染级联阴影图 Rendering Cascades

因为最多支持4个带阴影的方向光源，每个方向光源最多有4个级联，因此ShadowAtlas最多会被分成16个Tile。Unity自带的ComputeDirectionalShadowMatricesAndCullingPrimitives方法本身就很方便地支持了我们绘制一个方向光源的所有级联图，因此在代码中，我们只需要完成对Shadow Atlas的切分工作，以及把级联相关的参数传入该方法。关键代码如下所示。

```c#
    /// <summary>
    /// 渲染单个光源的阴影贴图到ShadowAtlas上
    /// </summary>
    /// <param name="index">光源的索引</param>
    /// /// <param name="split">分块量（一个方向）</param>
    /// <param name="tileSize">该光源在ShadowAtlas上分配的Tile块大小</param>
    void RenderDirectionalShadows(int index, int split, int tileSize)
    {
        //获取当前要配置光源的信息
        ShadowedDirectionalLight light = ShadowedDirectionalLights[index];
        //根据cullingResults和当前光源的索引来构造一个ShadowDrawingSettings
        var shadowSettings = new ShadowDrawingSettings(cullingResults, light.visibleLightIndex);
        //当前配置的阴影级联数
        int cascadeCount = settings.directional.cascadeCount;
        //当前要渲染的第一个tile在ShadowAtlas中的索引
        int tileOffset = index * cascadeCount;
        //级联Ratios（控制渲染区域）
        Vector3 ratios = settings.directional.CascadeRatios;
        //渲染每个级联的阴影贴图
        for (int i = 0; i < cascadeCount; i++)
        {
            //使用Unity提供的接口来为方向光源计算出其渲染阴影贴图用的VP矩阵和splitData
            cullingResults.ComputeDirectionalShadowMatricesAndCullingPrimitives(light.visibleLightIndex,
                i, cascadeCount, ratios,
                tileSize, 0f,
                out Matrix4x4 viewMatrix, out Matrix4x4 projectionMatrix, out ShadowSplitData splitData);
            //splitData包括投射阴影物体应该如何被裁剪的信息，我们需要把它传递给shadowSettings
            shadowSettings.splitData = splitData;
            int tileIndex = tileOffset + i;
            //设置当前要渲染的Tile区域
            //设置阴影变换矩阵(世界空间到光源裁剪空间）
            dirShadowMatrices[tileIndex] =
                ConvertToAtlasMatrix(projectionMatrix * viewMatrix, SetTileViewport(tileIndex, split, tileSize), split);
            //将当前VP矩阵设置为计算出的VP矩阵，准备渲染阴影贴图
            buffer.SetViewProjectionMatrices(viewMatrix, projectionMatrix);
            ExecuteBuffer();
            //使用context.DrawShadows来渲染阴影贴图，其需要传入一个shadowSettings
            context.DrawShadows(ref shadowSettings);
        }
    }
```

放置4个方向光源后，渲染出的Shadow Atlas如下图所示。



![20230201231026](https://raw.githubusercontent.com/recaeee/PicGo/main/20230201231026.png)



#### 3.3 剔除球空间 Culling Spheres

在确定每个级联图要渲染的实际区域时，Unity会为根据级联的阴影裁剪长方体创建一个球型空间，该球形空间会包裹整个阴影裁剪长方体，因此球形的空间会比原长方体多包裹住一些空间，这可能会导致有时在裁剪长方体区域外也会绘制一些阴影。下图为Culling Spheres的可视化图。



![20230201231702](https://raw.githubusercontent.com/recaeee/PicGo/main/20230201231702.png)



**Culling Spheres的作用是让Shader确定相机渲染的每个片元需要采样哪个级联图**。原理很简单，**对于相机要渲染的一个片元，计算出其光源空间下的坐标，通过它计算片元与每个Culling Sphere球心的距离，最后确定属于哪个球空间内，采样对应级联图**。

因此，在这一节中，主要实现了将级联信息（级联数、每个级联Culling Shpere信息）传递给GPU（代码不贴了，控制篇幅，后续将继续提炼）。

#### 3.4 采样级联图

上述小节已经把采样哪张级联图的逻辑讲的很清楚了，这一节就是具体在shader中实现上述逻辑。这里不禁感慨，之前一直以为阴影级联有多么高大上，这么看来，不过是只纸老虎，逻辑真的太简单了。

计算使用哪张级联图的代码如下，非常简单。

```c#
//计算给定片元将要使用的级联信息
ShadowData GetShadowData(Surface surfaceWS)
{
    ShadowData data;
    int i;
    for(i=0;i<_CascadeCount;i++)
    {
        float4 sphere = _CascadeCullingSpheres[i];
        float distanceSqr = DistanceSquared(surfaceWS.position,sphere.xyz);
        if(distanceSqr < sphere.w)
        {
            break;
        }
    }
    data.cascadeIndex = i;
    return data;
}
```

完成后，效果图如下，目前还是比较难看出来其带来的效果，因为大家的目光都会被精神污染的自阴影给吸引哈哈哈。



![20230202000651](https://raw.githubusercontent.com/recaeee/PicGo/main/20230202000651.png)



#### 3.5 剔除阴影采样 Culling Shadow Sampling

这一节主要实现了当片元的位置超出了最大的级联Culling Sphere时，不对其渲染阴影，将其阴影强度设置为0。

#### 3.6 最大阴影距离 Max Distance

由于最大级联的Culling Sphere会比实际的阴影裁剪长方体多出一些空间，可能会导致一些最远处绘制的阴影突然消失。因此，我们通过判断**片元在观察空间下的深度值是否超过maxShadowDistance**，如果超过，则不对其渲染阴影，这样，范围外的阴影会自然消失。我们需要将maxShadowDistance传递给GPU，并且为每个片元计算观察空间下的z值作为深度值。对于TransformWorldToView(input.positionWS).z，我们需要将其取反作为真正的深度值，因为观察空间的z轴是指向摄像机正后方的。其实我并没有看出3.5节和3.6节的阴影有多大区别，但是，该深度值在下一章节做渐变阴影的时候会发挥很大作用。

#### 3.7 渐变阴影 Fading Shadows

目前，我们的阴影会在超出maxShadowDistance的地方突然消失，因此我们考虑让越远的阴影强度越淡，从而自然地消失。我们通过设计一个函数来控制级联强度在不同片元深度下的结果，原教程使用的函数形状如下所示。



![20230202224728](https://raw.githubusercontent.com/recaeee/PicGo/main/20230202224728.png)



方法如下，比较简单。

```c#
/**
 * \brief 计算考虑渐变的级联阴影强度
 * \param distance 当前片元深度
 * \param scale 1/maxDistance 将当前片元深度缩放到 [0,1]内
 * \param fade 渐变比例，值越大，开始衰减的距离越远，衰减速度越大
 * \return 级联阴影强度
 */
float FadedShadowStrength(float distance,float scale,float fade)
{
    //saturate抑制了近处的级联阴影强度到1
    return saturate((1.0 - distance * scale) * fade);
}
```

其效果如下图所示。



![20230202225445](https://raw.githubusercontent.com/recaeee/PicGo/main/20230202225445.png)



#### 3.8 渐变最大级联 Fading Cascades

上一节我们实现了对所有级联等级下的阴影的过渡，这一节中，我们对最大级联下的阴影再乘以一个系数来控制，由此我们可以更好地**控制在最远处的阴影是如何自然消失的**。其函数和上一节稍有不同，另外会在_ShadowDistanceFade中配置第三个参数来控制该项。由于效果肉眼比较不可见，就不贴图了。

#### 4 阴影质量 Shadow Quality



![20230202230944](https://raw.githubusercontent.com/recaeee/PicGo/main/20230202230944.png)



目前，我们就拥有了一个可控且有效的阴影级联系统，接下来来解决那些掉san的自阴影，通常其被称为**阴影痤疮shadow acne**，其形成的原因可以解释为**物体不正确的自阴影**。由于阴影贴图上的一个像素覆盖的实际区域并不一定完全垂直于光源，但其都被该像素定义成了一个平面上的深度值，因此，实际不同深度的片元在采样该阴影贴图上的像素时产生了错误的比较结果，也就产生了这些阴影痤疮shadow acne。

即使我们增大阴影贴图的精度，也只会使这些shadow acne变得更小更多，并不能完全解决该问题。

#### 4.1 深度偏移 Depth Bias

因为阴影痤疮的产生是多个相邻的片元对应同一个深度，导致其中部分片元大于深度，部分片元小于深度，那我们就可以让这些相邻的片元对应的这个深度值手动偏移增大一些，让这些相邻的片元都小于该深度，就得到了统一的结果，不会产生阴影痤疮，这种方法就叫做**深度偏移Depth Bias**。这也是实际场景中经常使用的一个方法，简单并且高效。但其带来的副作用就是真正需要投射阴影的地方也产生一定的偏移，毕竟总体深度偏移了，这种现象叫做**Peter-Panning**，如下图所示，导致的效果就是看起来物体像浮在地面上一样。



![20230203000124](https://raw.githubusercontent.com/recaeee/PicGo/main/20230203000124.png)



除了Depth Bias，还有一种更“人性化”一点的方法:**斜率偏差Slope Bias**。其原理我没有深入了解，只是做了一定猜测。对于阴影贴图上要渲染的每个片元，计算方向光源与片元对应实际物体表面（可能用的是片元的法线信息做依据）的夹角，对于完全垂直于光源方向的片元（物体表面），不对其偏移；而当夹角增大，则设置不同程度的偏移。

以上，Depth Bias和Slope Bias两种方法的一大缺点就是，实际使用的值需要经过人为多次尝试后才能确定出一个合理值，因此我们目前不采用这两种方法。

#### 4.2 级联数据 Cascade Data

像DepthBias这种方法操作的是一整个ShadowAtlas，但由于我们使用了级联阴影，如果统一对每个级联图都使用同样的偏移来去除阴影痤疮是不合理的，因此需要**对每个级联等级来执行不同的去除痤疮**。这一节中，构建了一个float4Array的cascadeData，代表每个级联自身的相关信息，来传递给GPU。首先，每个元素的x分量表示其级联球半径的倒数。

#### 4.3 法线偏移 Normal Bias

造成阴影痤疮的主要原因就是摄像机像素和阴影贴图纹素的大小不匹配，确切的说是**多个摄像机像素会对应一个阴影贴图纹素**，也就意味着阴影贴图纹素覆盖的实际区域偏大。而增大阴影贴图精度无法彻底解决该问题，将阴影贴图的所有深度值都增大一些也不太合理。因此，我们考虑使用**法线偏移Normal Bias**。其原理就是，我们在采样阴影贴图之前，**将片元的位置沿着物体表面法线向外偏移一些**，再对阴影贴图采样。这也就意味着，在采样阴影贴图时，**对于那些原本会产生阴影痤疮的片元**，实际采样的出来的会比当前片元的深度偏大一些，造成这个结果的原因挺简单的，但我很难组织出语言来描述，因此手绘了下其原理，如下图所示。



![20230204193150](https://raw.githubusercontent.com/recaeee/PicGo/main/20230204193150.png)



实际上，我们只要让原本会产生阴影痤疮的摄像机像素**偏移一个级联Tile的纹素**采样就足够了，因此，我们将具体沿着法线偏移的量设置为**surfaceWS.position+normalBias**，其中normalBias为级联Tile大小的根号2倍。

应用上Normal Bias后，效果图如下所示。



![20230204193740](https://raw.githubusercontent.com/recaeee/PicGo/main/20230204193740.png)



但是，Normal Bias也不是完全没问题的。其会造成一些比阴影痤疮更不明显的问题，比如“光渗”等。

#### 4.4 可配置偏移 Configurable Biases

将slopeShadowBias和NormalBias设置成每个光源的可配置属性，教程里直接使用了Light组件里的Bias和Normal Bias来作为这两个属性的配置（这里其实是偷懒，这两个值在默认管线中不完全代表这两个含义）。配置slopeShadowBias的目的在于避免NormalBias带来的一些阴影问题，比如墙面和地面夹角的一些阴影问题。通常来说，slopeShadowBias设置为0，NormalBias设置为1比较合理。

#### 4.5 阴影平坠 Shadow Pancaking

参考[《Unity官方文档——阴影故障排除》](https://docs.unity.cn/cn/2020.2/Manual/ShadowPerformance.html)，为了进一步防止阴影痤疮，Unity采用了一种称为**阴影平坠Shadow Pancaking**的技术，该技术旨在减少沿光源方向渲染阴影贴图时使用的光照空间范围，这可以提高阴影贴图的精度，减少阴影痤疮。

简单来说，该技术让光源的阴影裁剪长方体的近裁面尽可能远离光源，只保留摄像机视锥体内的物体。但是如果有一个Shadow Caster（带阴影的物体）穿过了该近裁面，Unity默认管线会将超出的物体顶点钳制到裁剪长方体内，具体可以参考官方文档。我们需要在我们的管线中也实现顶点钳制，否则顶点会直接被裁剪掉，产生不正确的阴影。

但是将顶点钳制到近平面内当然会使阴影也发生扭曲，虽然很少会有这样特殊的物体出现，但是我们可以通过将光源的阴影裁剪近裁面拉近光源来避免该问题，将其设置为可配置属性nearPlaneOffset（Light组件的shadowNearPlane）。

#### 4.6 PCF过滤 PCF Filtering

目前我们实现的阴影效果属于硬阴影，而软阴影的实现可以通过PCF过滤来实现，PCF过滤即**Percentage Closer Filtering**，在2.3节中SAMPLER_CMP其内部就是PCF过滤实现采样的，其原理在2.3节中已讲过，在此不做过多赘述。值得注意的是，3x3的PCF是通过4次2x2的PCF来实现的，由此类推更高等级的PCF。

在增大PCF过滤的采样范围时，会导致阴影痤疮再次出现，因此需要根据PCF等级动态调整NormalBias来消除它。

PCF7x7的效果图如下所示。



![20230204215836](https://raw.githubusercontent.com/recaeee/PicGo/main/20230204215836.png)



#### 4.7 混合级联 Blending Cascades

在加入软阴影后，不同级联之间的阴影过渡处又产生了明显的不自然的变化，对于这些级联交界处，我们对其采样这两张级联的阴影结果，并根据一定值插值两者，得到较为自然的过渡。效果有些，但是也不是很明显，不贴效果图了。

#### 4.8 抖动过渡 Dithered Transition

上一节中的软阴影混合级联，对于交界处的片元，需要对两个级联图进行采样，每个级联图内又要进行PCF采样多次，最后性能可能会变得很差。因此引入**抖动过渡Dithered Transition**，其过渡质量比软阴影混合稍差，但**对于一个片元只需要采样一个级联图**，性能提升非常大。其原理为，对于级联交界处的片元，根据特定的抖动算法（CoreRP库自带有）与片元的位置计算出一个抖动值，然后在计算其需要使用的级联图时，根据片元的抖动值让部分片元采样下一级的级联图。其思路有点类似于早期透明混合的棋盘算法，在高分辨率下很难穿帮，并且配合AA后处理效果很好。

下图为低分辨率下的抖动过渡，可以看出其棋盘状的阴影。



![20230204224855](https://raw.githubusercontent.com/recaeee/PicGo/main/20230204224855.png)



#### 4.9 剔除偏移 Culling Bias

对于一个ShadowCaster，它可能被渲染到多个级联图上，但大部分时候，对于一个ShaderCaster的物体，可能只会采样到一个级联图上。这就导致了ShadowCaster多余的被多次渲染。因此，Unity原生提供了级联剔除功能，对于那些不会永远被当前级联图采样的ShadowCaster进行剔除，不对其进行渲染。其提供了可配置的剔除范围参数splitData.shadowCascadeBlendCullingFactor，参考官方API，其为应用于Culling Sphere半径的系数，范围[0,1]，其值越小，Unity剔除的对象越多，级联共享的渲染对象越多。考虑到需要让不同级联交界处的物体不能被剔除，将其值设置为与cascadeFade相关的保守的一个数值。以下给出原教程的效果图（左图值为0，右图值为1），我自己实验出来没剔除多少。



![20230204230836](https://raw.githubusercontent.com/recaeee/PicGo/main/20230204230836.png)



#### 5 透明物体 Transparency

目前，只有Clip模式的透明物体会被正确地渲染阴影，而Blend模式会被当作Opaque物体一样渲染阴影。

#### 5.1 阴影模式 Shadow Modes

我们一共需要4种阴影模式，分别为On、Clip、Dither、Off。在Lit.shader的阴影Pass中增加对应关键字的定义，并在预设中设置对应值，后续会在阴影Pass中使用这些关键字。

#### 5.2 裁剪阴影 Clipped Shadows

对于阴影模式的关键字，使用的关键字定义为#pragma shader_feature _ _SHADOWS_CLIP _SHADOWS_DITHER。在ShadowCasterPassFragment中，对于_SHADOWS_CLIP启用clip裁剪片元。

#### 5.3 抖动阴影 Dithered Shadows

对于_SHADOWS_DITHER，让片元的alpha值与根据物体坐标得到的抖动值进行比较来clip，得到抖动的阴影结果，类似透明物体的棋盘算法，具体可参考[毛星云大佬《【《Real-Time Rendering 3rd》 提炼总结】(四) 第五章 · 图形渲染与视觉外观 The Visual Appearance》](https://zhuanlan.zhihu.com/p/27234482)，配合PCF可以得到比较好的阴影效果，但是该算法不支持重叠的多个透明物体的阴影效果，以及对动态场景支持效果较差。因此对于半透明物体，通常使用Clip或者Off。

以下为Dither配合PCF7x7的阴影效果。



![20230204235452](https://raw.githubusercontent.com/recaeee/PicGo/main/20230204235452.png)



#### 5.4 无阴影 No Shadows

在这一节，通过编辑器拓展实现了材质的批量开关ShadowCasterPass。开关Pass使用m.SetShaderPassEnabled("ShadowCaster", enabled)。

#### 5.5 无光照阴影投射器 Unlit Shadow Casters

直接将ShadowCaster Pass复制到Unlit.shader，并且增加_Shadows Property。

#### 5.6 接收阴影 Receiving Shadows

在光照Pass中增加_RECEIVE_SHADOWS关键字，当它启用时，将阴影衰减值设置为1返回，达到不接收阴影的效果。

#### 结束语

这一章应该算最后一篇半长篇吧，主要是考虑了篇幅与毕设进度，后续章节会考虑以本节后半章的形式简写或者进行提炼，如需具体代码的详细注释可查看我的Github工程。终于又写完了一章，后面可能考虑开始同步进行一些风格化渲染的尝试。这几天也一直在看毛星云的文章，感叹真是国内图形学的明灯啊，可惜。

#### 参考

1. https://www.bilibili.com/video/BV1X7411F744/?spm_id_from=333.999.0.0
2. https://docs.unity3d.com/cn/2021.3/Manual/shadow-mapping.html
3. https://docs.unity3d.com/cn/2021.3/Manual/shadow-distance.html
4. https://answer.uwa4d.com/question/5e856425acb49302349a1119
5. https://zhuanlan.zhihu.com/p/67749158
6. https://learn.microsoft.com/en-us/windows/win32/direct3dhlsl/dx-graphics-hlsl-to-samplecmp
7. https://docs.unity3d.com/cn/current/Manual/SL-SamplerStates.html
8. https://zhuanlan.zhihu.com/p/545056819
9. https://docs.unity3d.com/cn/2021.3/Manual/shadow-cascades.html
10. https://zhuanlan.zhihu.com/p/498067785
11. https://zhuanlan.zhihu.com/p/27234482
12. 所有涩图均来自wlop大大
