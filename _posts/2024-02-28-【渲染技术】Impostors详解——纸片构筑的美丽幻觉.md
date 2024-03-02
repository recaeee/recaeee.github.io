---
layout:     post
title:      "【渲染技术】Impostors详解——纸片构筑的美丽幻觉"
subtitle:   ""
date:       2024-02-28 12:00:00
author:     "recaeee"
header-img: "img/in-post/default-bg.jpg"
tags:
    - 渲染技术
---
# 【渲染技术】Impostors详解——纸片构筑的美丽幻觉

![20240228180756](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240228180756.png)

#### 0 写在前面

早前，我截帧分析了《Call of Dragons》，具体内容可以看[《Call of Dragons》渲染截帧分析与迷思](https://zhuanlan.zhihu.com/p/681057070)，在其中提到了《Call of Dragons》中使用了3D场景与部分2D渲染结合的方式来优化性能，而这种方式的一个核心思路就是“三转二”，关于“三转二”的具体内容也可以参考我更早早前写的一篇[《三渲二（三转二）2D动画方案调研》](https://zhuanlan.zhihu.com/p/621733441)。而这种将3D物体转换成2D的思想，同样应用于一项渲染技术——Impostors，中译为冒名顶替者（感觉怪怪的，所以后文还是说Impostors吧）。

Impostors简单来说就是使用Billboard的2D片来表示复杂3D模型，可以说《Call of Dragons》中使用的三转二算是Impostors中的一种简化版。而《Call of Dragons》中摄像机不会旋转，因此它处理的情况更简单，甚至都用不到billboard渲染，只需要将2D片平行于摄像机近平面摆放就行了。而广泛意义上的Impostors会在各类游戏中广泛应用（通常用于表达远景），它需要考虑摄像机观察角度的变化、透视关系、渲染效果还原、过渡等众多因素，因此Impostors的实践绝不是一件简单的事情。

在Unity Asset Store中存在着一个名为[Amplify Impostors](https://assetstore.unity.com/packages/tools/utilities/amplify-impostors-119877#description)的工具，它提供了非常好的Impostors技术的实践，因此本文的主要目的是从Amplify Impostors中详细分析Impostors的实践原理与使用到的技术，由此对Impostors渲染技术进行学习与分析。

在本文中，你可以了解到：

1. Impostor概念。
2. Amplify Impostors的简单食用方法，包含界面参数介绍、配合LOD Group等。
3. 官方手册的要点提炼，包括支持、不支持的功能、不同Impostors类型、适合的使用场景等。
4. Amplify Impostors中烘培Impostors的过程详解，包括Billboard Mesh的创建、GBufferTextures的生成等。
5. 对Amplify Impostors提供的Demo场景的简要分析。

测试环境：Unity2021.3.12f1、Amplify Impostors 0.9.9.3、URP 12.1.7

#### 1 Impostor概念

参考《Real-Time Rendering, Fourth Edition》（感谢[Morakito翻译的中文版](https://zhuanlan.zhihu.com/p/667993188)）中的13.6章节，介绍了**广告牌技术Billboarding**，其原理是**基于摄像机观察方向来修改纹理矩形朝向**，通常会让矩形总是与屏幕平行。Billboard技术可以用于表示草、烟、能量盾、云等（比如原神中的远景云使用了Billboard实现），原书中也举例了一个用Billboard模拟的云层。

![20240228170510](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240228170510.png)

而Impostor是Billboard技术的一种应用，**通过将当前观察点中的复杂物体渲染成纹理，并将改纹理映射到一个Billboard Mesh上，从而创建一个Impostor**。它可以用于同一个物体的若干实例，或者在几帧中重复使用，从而分摊它的创建成本。

在渲染物体存在的地方，Impostor纹理是不透明的，而在其他任何地方都是完全透明的，它可以**将复杂的模型简化成单个的纹理**，因此Impostor对于快速渲染远处的物体十分有用。说到渲染远处的物体，当然还有一项耳熟能详的技术——**LOD**，即对于远处的物体使用更简化的Mesh。然而这种**简化的Mesh往往会丢失物体的形状信息和颜色信息**，。而Impostor则没有这个缺点，因为可以使生成的纹理分辨率与显示器的分辨率近似匹配。另一种使用Impostor的情况是，**当观察者只能看到这个物体的同一个侧面的时候**，可以使用Impostor（在《Call of Dragons》中就是这种使用场景）。

在渲染物体创建Impostor纹理的过程中，我们需要将Camera朝向物体包围盒的中心，并将视锥体设定为物体投影包围盒的最小矩形。Impostor纹理的Alpha值一开始会被清除为0，而在物体渲染的地方，Alpha值则会设置为1.0。然后将生成的纹理用作面向视点的Billboard Mesh的纹理贴图。

![20240228171807](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240228171807.png)

参考[UE4的Document——Render 3D Imposter Sprites](https://docs.unrealengine.com/4.26/en-US/RenderingAndGraphics/RenderToTextureTools/3/)，**Impostors是使用翻页书风格的Textures的Sprites**，用于存储一个静态Mesh的每一个可能的观察角度下的渲染结果，尽可能让sprite的渲染效果在材质和光照方面与原始Mesh匹配。但是，它也可能带来**帧之间的Popping**，意味着它不一定适用于所有场景，**当渲染大量不会靠近相机且移动缓慢的物体时**，它非常有用。

翻页书风格的Textures如下图所示，通过将不同角度下材质属性（Albedo、Normal等）存储到Textures中，用于渲染Sprite时尽可能模拟原始Mesh，这些就是Impostor所用的GBufferTextures。

![20240228172711](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240228172711.png)

#### 2 Amplify Impostors简介

Amplify Impostors是一个强大的工具，通过使用经典的billboard技术的现代版本来构建几何简化单体积准确的复杂模型表示。

拿官方的一张图片来举例，我们就可以知道Impostors做的事情，使用2D片代替3D Mesh来营造视觉欺骗，以此来优化渲染上的性能与表现。通常，Impostors可以应用于远景的表达。

![20240228155053](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240228155053.png)

在官方提供的Demo中，有一个场景对比了Impostors和3D Mesh的渲染效果，如下图所示（左下是3DMesh，右上是Impostors）。从中可以看到，Amplify Impostors的效果可以做到几乎与原始Mesh无法区分，在距离渲染物体较远的情况下，这个效果完全可以以假乱真。

![20240228173116](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240228173116.png)

但在距离渲染物体近的情况下，可以看到Impostors会有比较明显的穿帮，并且在视角转动时会有poping与不自然的过渡鬼影。

![ImpostorDem](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGoImpostorDem.gif)

#### 3 Amplify Impostors的Start Screen

导入Amplify Impostors.unitypackage后，会自动弹出一个**Amlify Impostors Start Screen**，如下图所示。

![20240219145002](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240219145002.png)

通过Manual按钮，可以快速跳转到官方提供的[使用手册](https://wiki.amplify.pt/index.php?title=Unity_Products:Amplify_Impostors/Manual)。

同时，Start Screen中提供了3种渲染管线HDRP、URP、Built-In下的Demo用例，点击对应的按钮即可导入对应的Demo用例。原文中提到了不要手动解压用例，否则不能保证导入正确，请**使用Start Screen导入Demo用例**。

官方提供了对Unity 2019.4 LTS以上、URP 10或HDRP 10（SRP 10）以上以及Built-in管线的一键导入支持。注意，对于URP和SRP，**官方只确保了对（10以上）原生URP和HDRP的支持**，而在实际项目中，我们很可能会对URP或HDRP进行修改，可能会导致导入用例失败。在我一开始使用实际项目的URP管线时，导入就失败了，原因是丢失了Decal Feature和SSAO Feature，因为Decal被我们从URP中删除了。

所以对于初步上手Amplify Impostors，**建议**先使用原生URP管线（HDRP也行，但我用的是URP，所以后文均会在URP的情况下实践）。在了解Amplify Impostors后，可以按照需要移植到实际项目的URP管线中。

#### 4 Manual提炼

##### 4.1 Impostor类型

Amplify Impostors中提供了**三种预烘培Impostor类型**：

1. **Spherical**：以经典的**经纬度**划分烘培角度进行快照。这种情况的Impostor **shader非常简单**，因此性能很好，并且Impostor近距离看下来表现基本没问题，但是**在视角发生变化的多帧之间，抖动很明显**。可以通过牺牲渲染分辨率以提高帧率来减少该问题（虽然我想应该不会有人这么做）。
   
	Spherical的图示如下，左边的猴子是真实3D模型，右边的猴子是Impostor效果，灰色几何体代表预烘培的所有角度（后两种类型的图示也是这种排布，后续不再赘述）。

	![SphericalExample](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGoSphericalExample.gif)

	![20240221115156](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240221115156.png)

2. **Octahedron**：以**icosphere**（短程线性多面体[Geodesic polyhedron](https://en.wikipedia.org/wiki/Geodesic_polyhedron)）划分烘培角度进行快照。这种方式烘培的好处是，给出摄像机的任意坐标，都可以在多面体上找到最接近该位置的3个frame，然后对3个frame进行混合。这种方法很好地**解决了视角移动时多个frame之间的抖动**。但是，**shader复杂度会更高**，当近距离观察Impostor时，混合会产生明显的**鬼影artifacts**。

	![OctahedronExample](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGoOctahedronExample.gif)

	![20240221120020](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240221120020.png)

3. **HemiOctahedron**：Octahedron的变体，只从“icosphere的上半球”进行**相同数量的快照**，从而有效地将混合精度提高了一倍。其缺点在于，如果不烘培下半球，从下面看时会产生不正确的结果。只有当我们已知永远只会从上方观察该物体时，该方法才适用。

	![HemiOctahedronExample](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGoHemiOctahedronExample.gif)

	![20240222160046](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240222160046.png)

##### 4.2 实时渲染Feature

1. SRP(HD and LW)(v4.9.0+)
2. 标准/Legacy的前向渲染和延迟渲染
3. 动态光照和阴影
4. 物体相交时的深度写入
5. 全局光照
6. 烘培的Ligthmaps（通过自定义烘培）
7. GPU Instancing
8. 抖动穿插过渡

##### 4.3 其他支持的功能与不支持的功能

1. 支持自定义打包贴图：渲染最多4个贴图用于standard材质和自定义材质，也可以使用自定义烘培最多8个不同的贴图。
2. 支持自定义形状编辑器：自动生成Impostor的自定义形状，减少Impostor的透明区域以减少overdraw，同时也**支持手动编辑**。
3. 支持烘培预设：定义了烘培Impostors的预设，其中还包括各种导入和导出选项。
4. **不支持Skinned mesh的烘培**，目前不支持Animated skinned meshes比如角色。作者计划提供一个实时烘培变体，允许动画物体以指定速率渲染为Impostors。
5. 如果想在近距离下改善质量，**只能提高纹理分辨率**，因为近距离下渲染Impostor会产生artifacts。注意，**impostors不适用于近距离渲染**，它不能代替真实的Mesh，只能作为远距离下渲染的性能优化。2K尺寸的纹理通常效果OK，对于移动端，甚至可以使用1K（感觉1K也很大啊）。
6. **仅当shader包含Deferred Pass时才支持标准烘培**，比如Unity Standard Shaders。但是**创建出的Impostors可以同时用于前向和延迟渲染两种模式**。如果原始的Mesh使用了自定义的前向shader（比如卡通渲染），我们需要使用**custom baking shader**来烘培它。

#### 5 Impostor烘培界面

Impostor的烘培操作是通过在Inspector视图中**Amplify Impostor**脚本提供的界面下进行的，如下图所示。

![20240222172947](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240222172947.png)

参考Manual，对**几个重要参数**进行解释：

1. **Impostor Asset**：对烘培impostor生成的**资源文件**的引用，其中包含了该对象大部分的impostor信息。使用资源文件形式的好处是如果我们有多个相同对象的impostor，资源文件可以共享。
2. **LOD Group**：引用对象身上的LOD Group组件，作者似乎更建议Impostor要和LOD Group搭配使用。
3. **References**：要烘培成impostor的renderer的引用。
4. **BakeType**：烘培和渲染impostor使用的技术，即上文提到的三种**impostor类型**，Spherical、Octahedron和HemiOctahedron。
5. **Texture Size**：最终烘培出的image尺寸。更高的尺寸意味着在近距离下有更好的表现，但会带来更高的内存占用和运行时渲染消耗。
6. **Axis Frames**：每个轴上的frames数量。例如，16代表会对单个impostor执行256（16x16）次快照。（这个轴指最终Impostor对应的2DTextures的横轴纵轴）对于spherical impostors，每个轴上可以指定不同的frame数量。
7. **Pixel Padding**：对于每个快照的边缘像素的填充padding，以**避免由mipmap引起的渲染伪影**。
8. **Billboard Mesh**：可用于编辑烘培的Billboard Mesh形状，使用了Unity内置的shape editor来自动估计形状，同时提供了手动编辑功能。
9. **Bake Preset**：提供了Standard烘培预设以及可自定义的预设，在预设中，可以配置**烘培用的shader**以及运行时**渲染impostor用的shader**（可自定义），也可以配置**烘培输出的Impostor对象的材质Textures**（_AlbedoAlpha、_SpecularSmoothness等等）以及Textures的一些属性。

#### 6 Impostor的烘培过程

讲完烘培Impostor的操作界面，接下来看下Impostor的**烘培过程**。首先，在前面提到过，**仅当shader包含Deferred Pass时才支持标准烘培**，同时在Manual中提到过，默认烘培会使用原始物体的shader中的**deferred pass**。

整个烘培过程可以简单描述成3个步骤：

1. 生成Billboard Mesh。
2. 烘培GBufferTextures。
3. 创建Impo Billboard。

##### 6.1 生成Billboard Mesh

因为最终**Impostor是渲染在Billboard方式的Mesh上的**，所以得先思考Mesh长什么样。可能我们会首先想到，直接用一个矩形Quad不就行了吗？没错，矩形片完全OK可用，但实际渲染对象可能是不规则的形状，矩形片上会有许多完全透明的区域，这意味着我们需要在Billboard Mesh使用的纹理上也空出这些透明的区域，因为uv坐标是在整个Mesh的所有顶点之间插值的，由此**造成了纹理空间的浪费**、纹理的**实际精度下降**。因此，对于Billboard Mesh的构建，**使用最贴近原渲染对象边缘的简单不规则几何片**是最合理的，这样能**充分利用纹理**，也能**减少overdraw**。

![20240226110320](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240226110320.png)

接下来就是考虑**如何生成最贴近原渲染对象边缘的简单不规则几何片**，主要思路就是**找到所有frame下的渲染对象边界，然后使用其中的最大（小）值作为最终Billboard Mesh的边界**。在Amplify Impsotors中，使用的方法可以简单描述成以下两个步骤（为了易于理解，我描述的步骤和代码实现会有略微不同，但原理完全是一致的）：

1. 创建一张默认大小为256x256的**AlphaTexture**，**对于每个frame**，将渲染对象使用**正交投影**（并确定合适的VP矩阵，确保渲染对象尽量保持在画面的正中心）**渲染到AlphaTexture**，渲染的Pass是按DEFERRED、Deferred、GBuffer的顺序寻找到一个可用的**延迟渲染Pass**。简单理解就是**将所有frame下渲染对象的Alpha值（相当于Mask）叠加渲染到一个Mask上，该Mask可以覆盖到所有frame下渲染对象的不透明区域**。可以在代码中配置AlphaTexture的尺寸，但是太大没有意义，这里只是用来生成简单多边形。同时这里我们也看到，在Amplify Impostors中**烘培用的shader必须包含延迟渲染Pass**。
2. 将上一步生成的AlphaTexture的Alpha通道值渲染（使用Blit）到一张**同尺寸256x256**的**CombinedAlphaTexture**上（Combined顾名思义）。使用的Pass代码如下，非常简单。重新Blit的主要目的是**过滤出AlphaTexture的Alpha通道值**（因为上一步走的延迟渲染Pass还会让AlphaTexture包含RGB值）。

    ```hlsl
		Pass
		{
			CGPROGRAM
			#pragma target 3.0
			#pragma vertex vert_img
			#pragma fragment frag
			#include "UnityCG.cginc"

			uniform sampler2D _MainTex;

			fixed4 frag(v2f_img i) : SV_Target
			{
				fixed4 finalColor = tex2Dlod(_MainTex, float4(i.uv, 0, 0));
				return float4(0, 0, 0, finalColor.a);
			}
			ENDCG
		}
    ```

3. 根据CombinedAlphaTexture（相当于一个最终的Mask）使用一定算法（只要能找到近似几何边缘的算法就行）**求出原渲染对象的近似几何边缘**，由这个近似几何边缘的顶点生成最终的Billboard Mesh。在Amplify Impostors中使用的求边缘方法为UnityEditor下的**SpriteUtility**的**GenerateOutline**方法，Sprite本身提供了符合这个功能的方法，省的自己实现了。

接下来详细介绍这三个步骤的**具体实现**，核心逻辑在于**叠加渲染所有frame下的3D物体来生成AlphaTexture**，**该逻辑和最终生成Impostor的GBufferTextures的逻辑大部分相同**，很多逻辑相通（比如确定当前Frame下的CameraVP矩阵），因此了解了生成AlphaTexture的逻辑就必然能理解最终烘培GBufferTextures的逻辑。

###### 6.1.1 生成AlphaTexure

生成AlphaTexure详细过程如下：

1. 首先通过AmlifyImpostor.cs引用的AmplifyImpostorAsset**获取烘培用到的所有信息**，包括Impostor类型、烘培用的shader、预设等等。
2. **计算3DMesh在所有快照角度下的Bound的最大值**（对应CalculatePixelBounds函数），生成各个快照角度下的CamearView矩阵的旋转部分，应用于mesh.bounds，得到每个角度旋转后的meshBounds信息，然后在所有角度中对**xy方向Bound**和**depth方向Bound**找到最大值。
3. **初始化创建要烘培的AlphaTexture**（如果是烘培GBufferTextures创建的就是_AlbedoAlpha、_SpecularSmoothness这些GBufferTextures），以及一张**trueDepth Texture**（用于在烘培时执行深度测试），**texture的尺寸均为minAlphaResolution默认256值**（如果是烘培GBufferTextures则创建的尺寸为配置的Texel Size，比如2048）。
4. **开启sRGB写入**。
5. **记录**原renderer的transform信息（position、rotation、localscale这些）并存储到成员变量，并将其**重置**为默认值（位置归零、缩放为1）。
6. 开始**烘培Impostor**（对应RenderImpostor函数）。首先判断Bake shader是否为空：是，则启用标准烘培；不是，则启用自定义烘培。
7. 创建一个commandBuffer，**SetRenderTarget**为所有要烘培的AlphaTexture（如果是烘培GBufferTextures则设置为GBufferTextures数组，利用MRT一次烘培多张textures）和trueDepth（DepthRT，临时资源，用于深度测试），进行一次**ClearRenderTarget**对所有textures初始化清理。
8. **筛选下要烘培的Mesh**，去除所有为空的、Disable的、ShadowsOnly的Renderer。
9. 遍历所有快照角度，**在每个frame角度下进行快照**，每次快照具体步骤如下。
10. 对于每个frame角度，首先计算CameraView矩阵的旋转部分，计算旋转后mesh.bounds作为**frameBounds**（当前frame的边界信息）。然后根据frameBounds**计算烘培需要使用的Camera的V矩阵和正交P矩阵**。其中**V矩阵**包含了一些信息，摄像机从哪看——从bounds中心退后一定距离，以及要看向哪——frameBounds的中心。**P矩阵**包含了一些信息，投影模式为正交（必须是正交），根据bounds以及第2步计算出的最大值计算视锥体各个平面（即near、far、left、right等），主要目的就是找到一个合适大小的视锥体。
11. 得到当前frame角度的V矩阵和P矩阵后，通过commandBuffer设置对应pipeline状态（VP矩阵、视口），注意**对于AlphaTexture的渲染，视口统一为(0, 0, 256, 256)**。然后，**选择烘培用的材质**：如果启用了标准烘培，则在当前renderer使用的材质中**按DEFERRED、Deferred、GBuffer的顺序寻找到一个可用的Pass**，如果全部都没找到，则fallback到**LightMode为Deferred的Pass**，并且如果存在DepthOnly的PrePass，则记录下该PrePass；如果启用了自定义烘培，则将<**当前Renderer的材质，自定义烘培使用的材质**>的键值对关系保存到bakeMats字典中，在自定义烘培的情况下，**默认会使用Pass 0**。
12. 根据Renderer的属性，**设置Lightmap、全局光照相关的关键字和值**。**关闭LightProbe关键字**，因为LightProbe原本作用是影响动态物体，即使renderer本身是动态的，但在烘培时也不需要让renderer包含LightProbe提供的的光照信息，否则物体所处的位置、周围环境的变化会影响烘培结果。虽然对于烘培AlphaTexture这一步是多余的，因为我们只需要Alpha值，但是对于烘培GBufferTextures是必要的。
13. **对Renderer进行烘培**：在标准烘培的情况下，调用commandBuffer.DrawRenderer，如果包含DepthOnly的PrePass则先渲染PrePass，再渲染GBUFFER，将GBUFFER渲染到AlphaTexture（如果是烘培GBufferTextures，则渲染到多张GBufferTextures）上；在自定义烘培的情况下，调用commandBuffer.DrawRenderer，使用自定义的烘培材质的Pass 0将物体烘培到AlphaTexture（如果是烘培GBufferTextures，则渲染到多张GBufferTextures）上。
14. 重复10~13步，烘培得到**所有frame下渲染对象Alpha值合并后的AlphaTexture**。
15. **清理所有临时资源**，包括commandBuffer、烘培材质。

###### 6.1.2 生成CombinedAlphaTexture

有了所有frame下的AlphaTextures，**生成CombinedAlphaTexture**就很简单了，即**过滤出AlphaTexture的Alpha通道值**。

1. 开启sRGB写入。
2. 加载采样Alpha值用的shader及材质（Shader名为ShaderPacker）。
3. 创建一张RenderTexture，名为CombinedAlphaTexture，尺寸为minAlphaResolution默认256值。
4. 将AlphaTexture使用上述步骤2的材质Blit到CombinedAlphaTexture上，Pass代码如下（上面提到过，这里再提一次）。
    ```hlsl
		Pass
		{
			CGPROGRAM
			#pragma target 3.0
			#pragma vertex vert_img
			#pragma fragment frag
			#include "UnityCG.cginc"

			uniform sampler2D _MainTex;

			fixed4 frag(v2f_img i) : SV_Target
			{
				fixed4 finalColor = tex2Dlod(_MainTex, float4(i.uv, 0, 0));
				return float4(0, 0, 0, finalColor.a);
			}
			ENDCG
		}
    ```
5. 将类型为RenderTexture的CombinedAlphaTexture存储到**Texture2D**类型的CombinedAlphaTexture中，方法很简单，使用**Texuture2D.ReadPixels**函数，这里不过多描述。

###### 6.1.3 生成Billboard Mesh

传入参数CombinedAlphaTexture以及一些必要的参数，直接调用SpriteUtility.GenerateOutline生成BillboardMesh。

如下图所示，木桶（3D Mesh）最后生成的CombinedAlphaTexture（256x256）和Billboard Mesh长这样。

![20240227104116](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240227104116.png)

##### 6.2 烘培GBufferTextures

烘培GBufferTextures可以简单分为以下几个步骤：

1. 生成GBufferTextures。
2. 对GBufferTextures执行一些可选操作。

###### 6.2.1 生成GBufferTextures

接下来就是生成GBufferTextures实际运行时渲染Billboard Mesh要采样的GBufferTextures了，在了解了AlphaTexture如何生成后，生成GBufferTextures的原理我们就几乎完全知道了。

**生成GBufferTextures的步骤和7.1.1节生成AlphaTexture几乎完全相同**，除了**视口Viewport设置**会有所区别。在渲染GBufferTextures时，每个frame的渲染结果不会叠加在RenderTarget上，而是会在GBufferTextures上**划分出快照数量（通常是frame x frame）个Grid区域**，渲染每个frame时将视口指定为对应frame的区域（有点像阴影级联的逻辑）。逻辑在这里就不赘述了，因为只有视口的区别以及一些细枝末节的区别。

如下图即为一个木桶生成的16x16的GBufferTextures中的AlbedoAlpha纹理。
 
![20240228174901](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240228174901.png)

###### 6.2.2 可选操作

根据烘培的预设、提供的Feature以及单次烘培的配置，可以有一些额外的步骤，不是很重要的点，因此略过详细内容。

1. （可选操作）在标准烘培下，对GBufferTextures的值做一定修正（Depth、Albedo等）。
2. （可选操作）如果输出GBufferTextures的图片格式为TGA，则将通道布局改为BGRA。
3. （可选操作）若烘培预设中PixelPadding值大于0，则使用ImpostorDilate Shader对GBufferTextures进行边缘扩张。
4. （可选操作）若在烘培预设的基础上Override了当前烘培的GBufferTextures尺寸，则对GBufferTextures进行Resize，（也就是说可以单独修改单次烘培中个别GBufferTexture的尺寸）。

##### 6.3 生成Impostor Billboard

目前我们生成了Billboard Mesh和GBufferTextures，接下来就是组合成最后的**Impostor Billboard（GameObject**）。

1. 配置**Runtime下渲染用的shader**并创建对应材质保存到AmlifyImpostorAsset中：如果在烘培预设中指定了Runtime Shader，则使用指定的Runtime Shader；如果没指定，则根据Impostor Type使用Amplify Impostors提供的对应Runtime Shader。
2. 将（RAM中的）GBufferTextures创建并保存到Assets中对应文件夹（磁盘空间）中，**生成贴图资源文件**。
3. 创建ImpostorGameObject，为其创建并配置**SharedMesh**（Billboard Mesh）、LOD Group。
4. **为Runtime下的材质设置正确的GBufferTextures properties**，根据Impostor类型开启关闭对应关键字，将一些必要的properties设置为正确值。

由此，最终的Impostor Billboard就生成完毕了。

如下图所示，最后的AmplifyImpostorAsset的资源包括以下内容：Runtime材质、Billboard Mesh、GBufferTextures。这些资源也就是渲染一个Impostor需要的所有东西。

![20240228174937](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240228174937.png)

#### 7 Runtime渲染

上一节完整介绍了Impostor Billboard的离线烘培生成，接下来我们来看向**运行时如何渲染Impostor**。

在这一节中，我们会重点关注**Impostor的frames选择与插值**，而不会关注Impostor Shader的前向渲染Pass和延迟渲染Pass，因为其不是本文的主要内容。接下来，会以效果最好的OctachedronImpostorURP.shader为例，对shader进行分析。

首先分析在vertex shader中，**Billboard片的实现**和**如何确定使用哪些frame以及对应顶点uv**。

1. 首先是**Billboard片的实现**，将CameraPosition从世界空间变换到模型空间**objectCameraPosition**，求出**模型空间物体指向摄像机的单位向量objectCameraDirection**。
2. 使用简单的叉乘计算，求出**定义Billboard Mesh的（单位）正交向量**，即计算BillboardMesh朝向objectCameraDirection时local的上向量（Billboard Local Y）与右向量（Billboard Local X）。如下图所示，只考虑相机在Object Space的XY平面内的情况下，Billboard正交向量示意图。
    ![20240227111600](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240227111600.png)
3. 直接对Billboard Mesh的Vertex的X分量乘以右向量（Billboard Local X），Y分量同理。这样做是正确的，因为Vertex的X分量确定了顶点水平方向到空间中心的距离，因此该距离等于Billboard显示时顶点水平方向到Billboard空间中心的距离。因此该操作保证了**Billboard显示时从Camera看到的Mesh和模型空间下从Z轴负方向看向Mesh是一致的**。以上三步即为Amplify Impostors中Billboard的实现，该方法利用了**Billboard Mesh只关注顶点的XY分量（Z分量始终为0）**的特性，从而避免了矩阵与向量的运算，而是直接使用**标量乘以向量**，简化了计算。
4. 接下来**确定使用哪些frame以及对应顶点uv**，首先计算**从camera指向变换后的Billboard顶点的向量localDir**，对于localDir使用**八面体映射**投影到二维平面上。**八面体映射**的逻辑可以参考[《八面体参数化球面映射方式及实现》](https://zhuanlan.zhihu.com/p/408898601)，该技术主要用于将球面参数化映射到平面，通常会用于Cubemap的纹理映射，这篇文章中对该方法介绍的很详细，在此不做过多赘述。
5. 将localDir映射到二维平面之后，就相当于将localDir映射到GBufferTextures上的UV坐标OctaUV（范围0到1），此时**OctaUV必定落在某一个frame的Grid内部**，那么容易得到**floor(OctaUV x framesHorizontal)就是当前落到的frame索引baseOctaFrame**。
6. 通过baseOctaFrame，很容易计算出该frame对应的Grid在GBufferTextures上的UV范围，比如对于2x2的Grid，左下角的Grid对应的UV范围为[0, 0.5]。同时，我们也可以得到该frame的**起始uv点**（即Grid的左下角），将起始uv点使用八面体映射的逆运算映射回三维空间下的向量，该向量即对应**该frame下烘培相机的近平面plane的法线normal**。
7. 接下来就是求当前顶点对应的uv坐标。已知该frame下烘培相机近平面plane的法线normal、objectCameraPosition、localDir，求出**以objectCameraPosition为起点、localDir为方向的射线与法线为normal的平面的交点p**。
8. 创建一个与plane正交的**本地切线空间坐标系TBN**，向量tangent与bitangent与normal正交。
9. 已知交点p、以plane为XZ平面的切线空间的X、Z方向单位向量tangent、bitangent，求出p在X、Z方向上的分量frameX、frameZ，由此求得**当前顶点对应的uv坐标uvs=-(frameX, frameZ)**（为什么是负的？）。同时，我们计算局部法线**localNormal**。
10. 对uvs应用scale和offset。
11. 由于OctachedronImpostor中，**会对3个frame进行采样插值**，所以接下来会**找到与当前落到的frame索引baseOctaFrame相邻的2个frame**进行6~10步的计算，**得到另2个uv坐标**，关于相邻frame的选择在此不做赘述。
12. 最终，得到Octachedron多面体上找到最接近该相机位置的3个frame对应的**3组顶点uv**。

fragment shader就没什么好说的了，主要就是采样三个frame然后根据权重插值，其余就是正常的延迟渲染Pass了。

#### 8 【番外篇】Amplify Impostors Demo分析

终于，我们分析完了Amplify Impostors中所有核心逻辑。接下来是一些我在分析Amplify Impostors提供的Demo时的一些**分析过程**，就当作番外篇了，有兴趣的同学可以阅读，跳过也完全OK。

因为之前提到了我在使用自己的URP管线时导入失败，那么就首先看下官方Demo用例中的**URP Asset和Renderer**都有些什么特点吧。

Demo中提供了三种质量的URP Asset，分为高保真度High Fidelity、均衡模式Balanced和性能至上Performant。三种Asset使用的Renderer均为同一个AI Universal Render Pipeline Asset Renderer。

![20240219150438](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240219150438.png)

在这里就拿**Balanced**分析吧（鲁迅先生说过，中国人的性情总是喜欢调和和折中的）。

首先看**AI Universal Render Pipeline Asset Balanced**（AI显然是Amplify Impostors的缩写）。

![20240219150856](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240219150856.png)

Emmmm，真是一个毫无特点而言的URP Asset呢（主要也是因为URP Asset本身玩不出什么花样啦）。注意到一点是，**当前使用的Renderer并不支持OpenGLES3**，这个对于手游项目而言还是挺致命的，很多手机设备都可能用的是OpenGLES3（快都给我去用Vulkan啦！），之后查一下Renderer不支持OpenGLES3的原因，埋坑ing~

在该URP Asset中，**启用了Depth Texture和Opaque Texture特性**。Depth Texture特性会在渲染完Opaque Pass（取决于Renderer上的Depth Texture Mode，在这里是After Opaques）之后Copy Camera深度图到一张_CameraDepthTexture中，而Opaque Texture特性会在渲染完Opaque Pass之后使用Opaque Downsampling（可配置的降采样模式）将ColorAttachment降采样到一张_CameraOpaqueTexture中。

对于URP Asset，基本上没别的好说的了。接下来看AI **Universal Render Pipeline Renderer**。

![20240219160644](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240219160644.png)

这里看到**Rendering Path为Deferred**，即使用了**延迟渲染**。**Depth Texture Mode为After Opaques**，上文中提到过，会在Opaque Pass之后Copy Depth到一张Texture。

分析完了URP管线，接下来看一个Demo场景**Barriels & Boulders**（木桶和巨石）。

![20240219161500](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240219161500.png)

这个场景非常简洁，放了5个木桶和3块石头。就拿一个木桶来分析吧。

对于一个木桶Game Object，其使用了**LOD Group**组件：**LOD0是个正常的3D模型**，有2106个三角面；**LOD1是Impostor**，只有6个三角面。也就是说，当摄像机距离木桶很远时，木桶会切到Impostor显示。

![20240219161610](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240219161610.png)

这样是Impostor的一个很恰当的实践方法——即**近景使用3D模型，远景使用Impostor**。正如官方手册中提到的，"The common purpose of using such technique is to be able to represent far distant objects with a very low polycount, for instance, trees, bushes, rocks, ruins, buildings, props, etc."**Imposter适合用于代替3D模型表现远景**。虽然Impostor在近景表现也还算可以，但在一些情况（如视角转动）下还是会穿帮，以及一些过渡的artifacts，**是否在近景使用Impostor取决于项目需求**吧。

#### 9 小结

好啦，目前我已经介绍了我对Impostor的全部认识，包括Impostor的概念、以及Amplify Impostors中对该技术的实现细节。相信在足够了解之后，无论是自己去实践Impostor，还是基于Amplify Impostors进行改造，都会变得不那么遥不可及。至于Impostors的实际性能怎么样，这里我尚且没有关注，一是因为我目前也仍然在Impostor学习过程中，二是对于不同的场景、不同的Impostor使用，性能都是未知的，不能一概而言。本文也告一段落了，未来如果有对Impostors新的认知，后面会及时更新到本文中。

#### 参考
1. https://assetstore.unity.com/packages/tools/utilities/amplify-impostors-119877#description
2. https://wiki.amplify.pt/index.php?title=Unity_Products:Amplify_Impostors/Manual
3. https://en.wikipedia.org/wiki/Geodesic_polyhedron
4. https://zhuanlan.zhihu.com/p/408898601
5. https://zhuanlan.zhihu.com/p/667993188
6. https://docs.unrealengine.com/4.26/en-US/RenderingAndGraphics/RenderToTextureTools/3/
7. 题图来自画师wlop
