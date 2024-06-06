---
layout:     post
title:      "【【RecaNoMaho】从零开始的体积光渲染——续"
subtitle:   ""
date:       2024-06-06 12:00:00
author:     "recaeee"
header-img: "img/in-post/default-bg.jpg"
tags:
    - RecaNoMaho
    - 体积光渲染
mathjax: true
---
# 【RecaNoMaho】从零开始的体积光渲染——续

![20240606225021](https://raw.githubusercontent.com/recaeee/PicGo/main/20240606225021.png)

#### 0 写在前面

RecaNoMaho是我的开源Unity URP项目，在其中我会致力于收录一些常见的、猎奇的、有趣的渲染效果与技术，并以源码实现与原理解析呈现，以让该系列内容具有一定的学习价值与参考意义。为了保证一定的可移植性、可迭代性和轻量性，在这些渲染效果与技术的实现中，第一原则是能写成RenderFeature的尽量写成RenderFeature，代价是原本一些修改URP代码很方便能做的事情可能要绕点路，以及丢失一些管线层面的可能优化点，这些优化可以在实际实践中再去实现。个人能力有限，文内如果有误或者有更好的想法，欢迎一起讨论~

RecaNoMaho项目仓库：https://github.com/recaeee/RecaNoMaho_P

目前使用的Unity版本：2022.3.17f1c1

目前使用的URP版本：14.0.9

在上一篇《【RecaNoMaho】从零开始的体积光渲染》中，我们学习了如何从零开始渲染体积光，其中包括什么是体积光（CG中对丁达尔效应的模拟）、体积光的理论模型（4类散射事件），以及在RecaNoMaho中的基于Ray-Marching的体积光实践。但上一篇我们只能算是得到了一个勉强够看的体积光，因此在这一篇中，我主要参考了SIGGRAPH2015《Frostbite Physically-based & Unified Volumetric Rendering》、《The Real-time Volumetric Cloudscapes of Horizon: Zero Dawn》这两篇关于体积光渲染的重要文章，对体积光进一步优化与完善，其中包括基于物理的参数定义、更合理的光照评估、模拟云雾的形态等内容（当然这篇文章结束后，差的东西还有很多）。

以下是本篇中目前实现的效果。

![云层翻涌](https://raw.githubusercontent.com/recaeee/PicGo/main/云层翻涌.gif)

#### 1 更科学地定义参与介质的材质

首先回顾一下上篇，在体积光的定量模型中，结合计算最后radiance$L_i(c,v)$、透光率$T_r$、消光系数$\sigma_t$（Extinction）的数学表达式看，**关于参与介质**，**影响体积光渲染效果的参数**包含以下几个：

1. **吸收系数$\sigma_a$**，光子被介质吸收并转化为热量或者其他形式的能量，是**减少光子**的一个因素，$\sigma_a$实际影响消光系数$\sigma_t$，从而影响透光率$T_r$。

2. **散射系数$\sigma_s$**，散射系数影响两个事件：一个是**外散射事件**（光子从当前光路流失），是**减少光子**的一个因素，和吸收系数$\sigma_a$一样，影响消光系数$\sigma_t$，从而影响透光率$T_r$；另一个是**内散射事件**（任何方向的光子进入当前光路），是**增加光子**的一个主要因素，从$L_i(c,v)$的表达式中可以看到，它直接作用于最终散射光线的计算，可以理解为影响最终散射光线总量的一个系数。

3. **自发光Emission**，从介质中发出的光线，和外散射事件一样，它是**增加光子**的一个因素。虽然在实践中，可能很少会有自发光的介质，但为了考虑更加周全，自发光也是影响体积光渲染效果的参数之一。如果实际用不到，可以省略掉该参数以进行优化。

4. **Phase function各向异性参数g**，评估内散射事件在各方向上的概率分布时我们会用到Phase function，在实践中我们选择了**Henyey-Greenstein(HG)相位函数**，其参数g为各向异性参数，控制向前和向后的散射概率分布。

在这里也重新贴上这些相关的数学表达式（也就是Shader代码的核心逻辑），计算radiance$L_i(c,v)$的表达式如。

![20240330170835](https://raw.githubusercontent.com/recaeee/PicGo/main/20240330170835.png)

透光率$T_r$的表达式如下。

![20240323004609](https://raw.githubusercontent.com/recaeee/PicGo/main/20240323004609.png)

消光系数$\sigma_t$（Extinction）的表达式如下。

![20240330171555](https://raw.githubusercontent.com/recaeee/PicGo/main/20240330171555.png)

OK，这里再总结一下，关于参与介质，影响体积光渲染效果的参数包含**吸收系数$\sigma_a$**、**散射系数$\sigma_s$**、**自发光Emission**、**Phase function各向异性参数g**。其实，这4个参数也是**基于Voxel实现的体积雾**（体积雾和体积光可以理解为一家人吧，可以混为一谈，后文中不再区分）中Voxel需要存储的数据，用来描述体素化参与介质的材质，关于基于Voxel实现的体积雾我之后会实践一番（埋坑ing）。

参考SIGGRAH2015的[《Frostbite Physically-based & Unified Volumetric Rendering》](https://www.ea.com/frostbite/news/physically-based-unified-volumetric-rendering-in-frostbite)这篇关于体积光渲染的著名文章（[在这里可以找到PDF](https://advances.realtimerendering.com/s2015/index.html)），对于**吸收系数$\sigma_a$**、**散射系数$\sigma_s$** 这两个参数，虽然它们的物理意义很明确，但是对于艺术家来说，这组参数不一定适合用来调整体积光的美术效果。因此，有一对**更符合直觉、更容易调控的参数**可以代替这两个参数——**消光系数Extnction（$\sigma_t=\sigma_s+\sigma_a$）和反照率Albedo（$\rho=\sigma_s/\sigma_t$）**。消光系数$\sigma_t$表示了随着光线经过介质的光子衰减，而反照率Albedo表示了介质散射出的光线比例（和PBR材质的Albedo是一个概念）。

![20240403195610](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240403195610.png)

当然选择(**吸收系数$\sigma_a$**、**散射系数$\sigma_s$**)或者选择(**消光系数$\sigma_t$**、**反照率Albedo**)都是可以的，在RecaNoMaho中选择使用后者作为参数，因为它更符合直觉且易于调试。

因此，在RecaNoMaho中，选择使用以下参数来更科学物理地定义参与介质的材质：

1. **反照率Albedo**

2. **消光系数Extinction($\sigma_t$)**

3. **各向异性参数Phase g**

4. **自发光Emission**

无论在基于Ray-Marching的体积光还是基于Voxel的体积雾中都可以**共用**（这两种方案对应的光照以及数据模型都是相同的）这组基于物理的参数，参考[《荒野大镖客2》中的体积云雾方案](https://zhuanlan.zhihu.com/p/638483468)，R星在SIGGRAPH2019上提出了一种较为创新的设计理念，就是**对于近景使用基于Voxel实现的体积雾，对于远景则使用可变步幅的Ray-Marching体积光**，可以用同一套参数去同时描述这2种方案下的体积光（雾）效果并结合使用，不会遇到不同方案调不同参数的问题，非常方便。

在参数界面，我们选择使用**反照率Albedo和消光系数$\sigma_t$**，而在实际运行时，我们将散射系数$\sigma_s$和消光系数$\sigma_t$传递给GPU（shader），因为它们才是shader中计算体积光需要的变量，因此对于参与介质的材质的定义，大致的C#代码如下。

```csharp
        [Tooltip("反照率Albedo")] public Color albedo = Color.white;
        [Tooltip("消光系数Extinction")] public float extinction = 1.0f;
        [Tooltip("各向异性Phase g")] [Range(-1f, 1f)]
        public float phaseG = -0.5f;
        [Tooltip("自发光Emission")][ColorUsage(true, true)] public Color emission = Color.black;

        public Vector4 GetScatteringExtinction()
        {
            float nonNegativeExtinction = Mathf.Max(0f, extinction);
            // scattering = albedo * extinction = (scattering / extinction) * extinction
            return new Vector4(albedo.r * nonNegativeExtinction, albedo.g * nonNegativeExtinction, albedo.b * nonNegativeExtinction, nonNegativeExtinction);
        }

        public Vector4 GetEmissionPhaseG()
        {
            return new Vector4(emission.r, emission.g, emission.b, phaseG);
        }
```

![20240404234213](https://raw.githubusercontent.com/recaeee/PicGo/main/20240404234213.png)

#### 2 更清晰的Shader代码来描述数学表达式

在重新确定了参与介质的参数之后，我们将目光投向Shader代码。

在上一篇文章中，关于体积光着色的主体Shader代码并没有完全严格地按照上述这些数学式去做，并且Shader代码与数学式的对应关系较为混淆，也存在一定错误书写。因此在本文中，我重新以更严谨的写法对整体的Shader代码进行了一次重构，**确保Shader代码与数学式基本对应**，**将代码变量与数学变量更加统一**。同时，我也参考了SIGGRAPH2015《Frostbite Physically-based & Unified Volumetric Rendering》**对其中的部分计算（离散渐进法求积分）进行了优化（部分解析）**。虽然Frostbite使用了基于Voxel的体渲染，而本文使用了Ray-Marching，但其中的数学计算式基本一致，与方案无关。

##### 2.1 离散渐进求Radiance优化

让我们首先看向**对视线v上的每个RayMarching采样点（c-vt）求光照Radiance的积分部分**（红框部分）。

![20240606025140](https://raw.githubusercontent.com/recaeee/PicGo/main/20240606025140.png)


在对totalRadiance的计算中，Frostbite比起常规地直接使用**离散渐进法**对积分式进行计算，**进一步对Transmittance进行了部分解析，使totalRadiance计算结果更接近理论正确值**，接下来进行一定说明。

我们非常容易知道，在Ray-marching过程中，我们会步进n次，每次步进距离为stepSize，同时每次步进对应一个sample采样点，在该点进行光照计算得到radiance，然后认为Ray-marching方向上对于以sample为中心，半径1/2 stepSize内，其光照计算得到的radiance相同，因此该段距离内radiance和为stepSize*radiance，将其累加到最终结果中，对应了**离散渐进法求积分numerically**的过程，求出的结果也就是**黎曼和Riemann sum**。因此，我们知道**Ray-marching的最终结果（离散化积分）肯定会与理论值（dx无限接近0）有差距，但当步进次数越多时，Ray-marching结果会越来越接近理论值**。

而在体积光理论模型中，其中的积分式部分为红框部分，可以看到积分式中包括**透射率$T_r$与采样点处的radiance（即$L_{scat}$）**，如果直接采用该式积分，那么就**相当于认为在每个dt这段距离内，$T_r$和$L_{scat}$都是定值**。

如果照着以上思路（纯粹的离散渐进法），Shader代码大致如下（主要是totalRadiance的计算）。

```c++
    //Ray-marching
    float4 scattering(float3 ray, float near, float far)
    {
        float3 totalRadiance = 0;
        float totalTransmittance = 1.0;
        float stepSize = (far - near) / _Steps;

        for(int i = 0; i < _Steps; i++)
        {
            float3 pos = _WorldSpaceCameraPos + ray * (near + stepSize * i);
            float density = SampleCloudDensity(pos);
            //dx的消光系数
            float transmittance = BeerLambert(extinctionAt(pos, density), stepSize);
            
            //“严谨地按照数学式红框中部分去离散渐进法求解”
            totalRadiance +=  totalTransmittance * scatteredLight(pos, ray, density) * scatteringAt(pos, density) * stepSize;
            
            totalTransmittance *= transmittance;
        }
        
        return float4(totalRadiance, totalTransmittance);
```

而在Frostbite中提出，**对于$L_{scat}$，我们的确可以按上述思路认为，对一段dx内$L_{scat}$始终为定值，否则评估成本太高**，但此时可以**让积分项对于透射率$T_r$进行一定程度上解析analytically**。如下图红框部分所示，将$S$（也就是$L_{scat}$）视为常数后，积分式变为了函数$T_r$在dx上的积分，此时因为$T_r$是关于dx的简单函数，可以对积分进行解析。

![20240419191947](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240419191947.png)

参考Frostbite中的做法，积分代码变为：

```c++
    //Ray-marching
    float4 scattering(float3 ray, float near, float far)
    {
        float3 totalRadiance = 0;
        float totalTransmittance = 1.0;
        float stepSize = (far - near) / _Steps;
        // [UNITY_LOOP]
        for(int i = 0; i < _Steps; i++)
        {
            float3 pos = _WorldSpaceCameraPos + ray * (near + stepSize * i);
            float density = SampleCloudDensity(pos);
            //dx的消光系数
            float transmittance = BeerLambert(extinctionAt(pos, density), stepSize);
            
            //参考SIGGRAPH2015 Frosbite PB and unified volumetrics中对积分式进行了一定求解。
            //Scattering评估成本太高，可以认为dx内为定值，即常数。
            //认为dx内Transmittance不是定值，因此将Transmittance在0~D上的积分求解，让Transmittance随dx连续变化，使计算结果更加接近正确值。
            float3 scattering =  scatteringAt(pos, density) * scatteredLight(pos, ray, density) * (1 - transmittance) / max(extinctionAt(pos, density), 0.00001f);
            totalRadiance += scattering * totalTransmittance;
            
            //如果按照RTR4的积分式积分，此时对于stepSize这段距离，认为其中的totalTransmittance是一个定值，这是不合理的，因此不直接使用积分式累积。
            // totalRadiance +=  totalTransmittance * scatteredLight(pos, ray, density) * scatteringAt(pos, density) * stepSize;
            
            totalTransmittance *= transmittance;
        }
        
        return float4(totalRadiance, totalTransmittance);
    }
```

##### 2.2 正确混合体积光Radiance与实体表面Radiance

由此，由于$T_r$在步进过程中可认为是精确值，因此比直接采用Ray-marching积分式得到的光照计算结果**更接近理论值**，从上图Frosbite的PPT中可以看到当采用改进过后的算法后，计算结果会更接近能量守恒，避免一些极端情况下的光照计算偏差过大。

另外，从$L_i(c,v)$可以看到，对于射线命中的实体表面，其贡献的radiance**需要乘上累计的$T_r$系数**。因此在渲染体积光的Draw Call时，我们返回float4(totalRadiance, totalTransmittance)，并将混合模式设置为<One, SrcAlpha>，从而使混合遵循数学式。题外话，实现该混合是非常基于物理的，但有时如果我们希望额外做一些不物理的体积光效果时，也可以考虑去掉混合，而是直接使用BlitAdd，即<One, One>，最终效果不那么物理一些，但通常来说是在**可接受范围**内，在《原神》中也对God Ray采用了直接Blit Add的方案。

##### 2.3 体积光Radiance计算代码整理

这一节中，我也根据数学式，对计算最终Radiance的代码按照数学表达式进行了整理。

首先是上面提到过的**对视线v上的每个RayMarching采样点（c-vt）求光照Radiance的积分部分**（这里再次贴一下）。

![20240606025140](https://raw.githubusercontent.com/recaeee/PicGo/main/20240606025140.png)

```c++
    //Ray-marching
    float4 scattering(float3 ray, float near, float far)
    {
        float3 totalRadiance = 0;
        float totalTransmittance = 1.0;
        float stepSize = (far - near) / _Steps;
        // [UNITY_LOOP]
        for(int i = 0; i < _Steps; i++)
        {
            float3 pos = _WorldSpaceCameraPos + ray * (near + stepSize * i);
            float density = SampleCloudDensity(pos);
            //dx的消光系数
            float transmittance = BeerLambert(extinctionAt(pos, density), stepSize);
            
            //参考SIGGRAPH2015 Frosbite PB and unified volumetrics中对积分式进行了一定求解。
            //Scattering评估成本太高，可以认为dx内为定值，即常数。
            //认为dx内Transmittance不是定值，因此将Transmittance在0~D上的积分求解，让Transmittance随dx连续变化，使计算结果更加接近正确值。
            float3 scattering =  scatteringAt(pos, density) * scatteredLight(pos, ray, density) * (1 - transmittance) / max(extinctionAt(pos, density), 0.00001f);
            totalRadiance += scattering * totalTransmittance;
            
            //如果按照RTR4的积分式积分，此时对于stepSize这段距离，认为其中的totalTransmittance是一个定值，这是不合理的，因此不直接使用积分式累积。
            // totalRadiance +=  totalTransmittance * scatteredLight(pos, ray, density) * scatteringAt(pos, density) * stepSize;
            
            totalTransmittance *= transmittance;
        }
        
        return float4(totalRadiance, totalTransmittance);
    }
```

对于场景中**给定位置x（RayMarching采样点）和方向v（观察方向的相反方向）**，对于精确光源，其在该x点v方向**内散射事件**的数学式与对应Shader代码如下。

![20240606041925](https://raw.githubusercontent.com/recaeee/PicGo/main/20240606041925.png)

```c++
    //返回介质中x处接收到的光线（RGB），以及x处到光源的方向
    float3 scatteredLight(float3 positionWS, float3 viewDir, float density)
    {
        //_LightPosition.w = 0时，为方向光，此时_LightPosition.xyx为方向光dir
        //_LightPosition.w = 1时，为SpotLight
        float3 lightDir = normalize(_LightPosition.xyz - positionWS * _LightPosition.w);
        //考虑Phase Function、可见性函数v、光源强度
        float3 radiance = PI * Phase(-lightDir, -viewDir, positionWS, density) * GetLightVisible(positionWS, lightDir) * _LightColor.rgb;
        
        return radiance;
    }
```

其中**相位函数Phase**为Henyey-Greenstein(HG)相位函数。

![20240606042555](https://raw.githubusercontent.com/recaeee/PicGo/main/20240606042555.png)

```c++
    //PhaseFunction，给定入射光线方向，根据概率分布，计算指定方向上的散射光线RGB，体积积分总是为1
    float3 Phase(float3 lightDir, float3 viewDir, float3 positionWS, float density)
    {
        // 采用Henyey-Greenstein phase function(即HG Phase)模拟米氏散射，即介质中微粒与入射光线波长的相对大小接近相等，模拟丁达尔效应时的情况
        float phaseG = phaseGAt(positionWS, density);
        return ( 1 - phaseG * phaseG) / ( 4 * PI * pow(1 + phaseG * phaseG- 2 * phaseG * dot(viewDir, lightDir) , 1.5));
    }
```

**可见性函数**$v(x.p_{light_i})$代表了从光源位置$p_{light_i}$处发出的光线最终到达位置$x$的比例。

![20240606042649](https://raw.githubusercontent.com/recaeee/PicGo/main/20240606042649.png)

```c++
    //可见性函数v，代表从光源位置到达采样点positionWS的比例
    float GetLightVisible(float3 positionWS, float3 lightDir)
    {
        //聚光灯衰减项
        float spotAttenuation = step(_LightCosHalfAngle, dot(lightDir, _LightDirection.xyz));
        //shadowmap阴影
        float shadowmapAttenuation = shadowAt(positionWS);
        //考虑体积阴影项，从光源到采样点的透光率Transmittance
        float transmittance = SampleAccumulatedCloudIrradiance(positionWS, lightDir);
        //不考虑体积阴影项（即使是聚光灯，我们暂时也先不考虑距离衰减）
        // float transmittance = BeerLambert(extinctionAt(positionWS, 1), 1);

        return spotAttenuation * shadowmapAttenuation * transmittance;
    }
```

以上代码基本清晰地描述了整个体积光着色过程的主要数学表达式。

#### 2 雾的形态

体积光通常会用来模拟**云、雾**的效果（我目前更偏向模拟雾的效果，主要是因为我想要实现舞台上的雾效），而**云和雾的密度分布通常会带有较大的随机性**，因此在模拟云雾的时候，我们通常会用到一些**噪声纹理**，最常见的比如**Perlin Noise、Worley Noise和Perlin-Worly Noise**，因此如何使用这些噪声模拟出更贴近现实中的云雾就成了一个课题。由于雾的模拟并不是一个常见的话题，因此我们来看向云的模拟（云应该也算一种雾吧！放舞台上效果应该也不错），说到云的模拟就不得不提到与《Frostbite Physically-based & Unified Volumetric Rendering》同年发表的另一篇SIG了——[《The Real-time Volumetric Cloudscapes of Horizon: Zero Dawn》](https://www.guerrilla-games.com/read/the-real-time-volumetric-cloudscapes-of-horizon-zero-dawn)，即《Horizon: Zero Dawn》中对体积云形状模拟的研究，因此我首先参考该篇SIG的实现对云雾的形态进行了塑造。

下图为2015年的《Horizon: Zero Dawn》云渲染实机画面效果，可以看到效果非常精细（难以相信是9年前的技术了）。

![20240606043746](https://raw.githubusercontent.com/recaeee/PicGo/main/20240606043746.png)

《Horizon: Zero Dawn》的云塑造技术（作者：Andrew Schneider）同样被写进了《GPU Pro 7》的体积云相关章节，这里提供下已有的一些中文参考。

[【【GPU Pro 7】Real-Time Volumetric Cloudscapes】](https://www.jianshu.com/p/ae1d13bb0d86)

[【GPU Pro 7——实时体积云（翻译，附Unity工程】](https://blog.csdn.net/b8566679/article/details/123446496)

##### 2.1 噪声的物理意义

首先，我们需要思考我们怎么用噪声来模拟云雾的形态，噪声值乘在哪？

从计算最终Radiance的数学表达式中，我们似乎很难去找到一些可以插入这个噪声值的地方。因此，我们重新回想下，从物理上，参与介质的什么属性会影响体积光效果？在[上篇](https://zhuanlan.zhihu.com/p/688803084)的1.4节中，我们提到影响散射的因素，可以认为是以下两个：

1. 参与介质的密度越大，其中粒子越多，散射程度越大，反应到Shader代码中对应的参数为**散射系数$\sigma_s$**。
2. 参与介质中单个粒子大小也会影响散射，即Phase Function，反应到Shader代码中对应的参数为**消光系数$\sigma_t$**。

因此，我们将Noise作为这两项参数的系数来塑造云的形状。

##### 2.2 使用什么噪声来Model云

使用什么噪声来塑造云的形状，这点就更偏向于经验谈与尝试了，本文也沿用了《Horizon: Zero Dawn》中给出的Model模型。

总共使用**2张3DTexture和1张2DTexture**来塑造云的形状。

**Noise3DTextureA**的尺寸为128x128x128，R通道存储**Perlin-Worley Noise**，GBA通道存储频率为4、8、16，FBM迭代3次的Worley Noise。其中，Perlin-Worley Noise的Base Perlin Noise的频率为4，FBM迭代次数为7，**用于模拟云的基本形状**。**Worley Noise**经常用于渲染焦散和水的效果，如果将其值取反（1-value），可以近似分型波浪效果，比较符合云的团簇状效果。单纯的Worley Noise形状又缺少一些细节，因此通过Worley Noise对Perlin Noise进行Remap，可以得到Perlin-Worley Noise，通过Perlin-Worley Noise塑造云的基本形状。

关于Perlin Noise的相关知识可以参考[【生成连续的2D、3D柏林噪声（Perlin Noise），技术美术教程】](https://zhuanlan.zhihu.com/p/620107368)。

关于Worley Noise的相关知识可以参考[【生成连续的2D、3D细胞噪声（Worley Noise），技术美术教程】](https://zhuanlan.zhihu.com/p/620316997)

![20240525123843](https://raw.githubusercontent.com/recaeee/PicGo/main/20240525123843.png)

**Noise3DTextureB**的尺寸为32x32x32，R、G、B通道分辨存储了频率为4、8、16，FBM迭代3次的**Worley Noise**，**用于对云层边缘进行腐蚀处理，增加更多细节**。

![20240525123858](https://raw.githubusercontent.com/recaeee/PicGo/main/20240525123858.png)

2DTexture的尺寸为128x128，为**Curl Noise**，在本文中暂时使用算法实时生成，后续可能考虑换成贴图（懒是原罪）。

![20240606171315](https://raw.githubusercontent.com/recaeee/PicGo/main/20240606171315.png)

##### 2.3 Model算法

好了，目前我们准备好了塑造云基本形状的所有噪声图，接下来来在Shader中使用这些噪声图，在算法中，我们大量使用了Remap函数。

```c++
//从一个范围Remap到另一个范围的工具函数
float Remap(float originValue, float originMin, float originMax, float newMin, float newMax)
{
    return newMin + (((originValue - originMin) / (originMax - originMin)) * (newMax - newMin));
}
```

第一步，我们使用Noise3DTextureA的R通道（低频Perlin-Worley Noise）作为云的基本形状，然后将GBA通道的Worley Noise构建FBM，对Perlin-Worley使用Remap函数进行膨胀，为云增加细节。

```c++
float SampleCloudDensity(float3 positionWS)
{
    float heightFraction = GetHeightFractionForPoint(positionWS, float2(_InCloudMin.y, _InCloudMax.y));
    //归一化xz用来计算Coverage
    float2 xzFraction = GetXZFractionForPoint(positionWS, float4(_InCloudMin.xz, _InCloudMax.xz));
    float3 uvw = positionWS * _CloudScale.xyz;
    uvw += float3(_Time.xxx * _CloudFlowSpeed);
    //采样Perlin-worley and Worley noises
    float4 lowFrequencyNoises = SAMPLE_TEXTURE3D(_CloudNoise3DTextureA, sampler_LinearRepeat, uvw).rgba;
    //gba通道再次FBM处理，用来刻画细节
    float lowFrequencyFbm = 0.625f * lowFrequencyNoises.g + 0.25f * lowFrequencyNoises.b + 0.125f * lowFrequencyNoises.a;
    //使用lowFrequencyFbm对Perlin-Worley进行膨胀，塑造云层基本形状
    float baseCloud = Remap(lowFrequencyNoises.r, -(1.0 - lowFrequencyFbm), 1.0, 0.0, 1.0);

    ...
}
```

接下来，我们应用一张**WeatherDataTexture**，这张图可以是噪声，也可以是美术产出，比较随意，该纹理的R通道代表云层覆盖信息CloudCoverage（XZ轴上哪些地方有云，哪些地方没云），G通道代表降水Precipitation（我的实现中没用到该参数），B通道代表云类型CloudType（也会影响Cloud在Y轴的上分布）。通过WeatherDataTexture的R、B通道，我们可以**确定云在XYZ三轴上的分布**，可以方便控制哪里有云，哪里没云。

参考SIG，此时的效果应该是这样（虽然我自己只到这里并做不出这样的效果，还涉及到一些高度分布函数、Coverage分布计算原文并没有详细给出，只能自己猜测，这里自己实现的效果并没有原文中那么好，因此也略过这部分代码的解释，有兴趣可以看我的代码）。

![20240606215850](https://raw.githubusercontent.com/recaeee/PicGo/main/20240606215850.png)

到这一步的代码如下。

```c++
float SampleCloudDensity(float3 positionWS)
{
    float heightFraction = GetHeightFractionForPoint(positionWS, float2(_InCloudMin.y, _InCloudMax.y));
    //归一化xz用来计算Coverage
    float2 xzFraction = GetXZFractionForPoint(positionWS, float4(_InCloudMin.xz, _InCloudMax.xz));
    float3 uvw = positionWS * _CloudScale.xyz;
    uvw += float3(_Time.xxx * _CloudFlowSpeed);
    //采样Perlin-worley and Worley noises
    float4 lowFrequencyNoises = SAMPLE_TEXTURE3D(_CloudNoise3DTextureA, sampler_LinearRepeat, uvw).rgba;
    //gba通道再次FBM处理，用来刻画细节
    float lowFrequencyFbm = 0.625f * lowFrequencyNoises.g + 0.25f * lowFrequencyNoises.b + 0.125f * lowFrequencyNoises.a;
    //使用lowFrequencyFbm对Perlin-Worley进行膨胀，塑造云层基本形状
    float baseCloud = Remap(lowFrequencyNoises.r, -(1.0 - lowFrequencyFbm), 1.0, 0.0, 1.0);
    
    //塑造云层在不同高度下的密度
    //在我的舞台场景下并不需要严格按照真实云层的高度-密度梯度去塑造云在不同高度下的形状，但这里保留这种写法，可以让该方法更通用。
    //weatherData:r->Coverage g->Precipitation b->CloudType
    float3 weatherData = SAMPLE_TEXTURE2D(_WeatherDataTexture, sampler_LinearRepeat, xzFraction).rgb;
    //暂时先不使用Cloud Type分布
    weatherData.z = 0;
    //给heightFraction增加点扰动，增加起伏效果（正好利用Worley Noise）
    heightFraction += (2 * weatherData.r - 1) * 0.03;
    //再手动Clip下
    weatherData.r += 0.25;
    
    float densityHeightGradient = GetDensityHeightGradientForPoint(positionWS, weatherData, heightFraction);
    baseCloud *= densityHeightGradient;
    
    //使用云层覆盖率的贴图来Clip
    float cloudCoverage = clamp(weatherData.r, 0, 0.999);
    float baseCloudWithCoverage = Remap(baseCloud, cloudCoverage, 1.0, 0.0, 1.0);
    //To ensure that density increases with coverage in an aesthetically pleasing way, we multiply this result by the cloud coverage attribute.
    baseCloudWithCoverage *= cloudCoverage;

    float finalCloud = baseCloudWithCoverage;

    ...
}
```

最后，我们使用CurlNoise和Noise3DTextureB对云层侵蚀，在云的上端使用Worley Noise收缩来产生翻滚的效果，在云的底部增加一些湍流效果。

完整的SampleCloudDensity函数代码如下。

```c++
float SampleCloudDensity(float3 positionWS)
{
    float heightFraction = GetHeightFractionForPoint(positionWS, float2(_InCloudMin.y, _InCloudMax.y));
    //归一化xz用来计算Coverage
    float2 xzFraction = GetXZFractionForPoint(positionWS, float4(_InCloudMin.xz, _InCloudMax.xz));
    float3 uvw = positionWS * _CloudScale.xyz;
    uvw += float3(_Time.xxx * _CloudFlowSpeed);
    //采样Perlin-worley and Worley noises
    float4 lowFrequencyNoises = SAMPLE_TEXTURE3D(_CloudNoise3DTextureA, sampler_LinearRepeat, uvw).rgba;
    //gba通道再次FBM处理，用来刻画细节
    float lowFrequencyFbm = 0.625f * lowFrequencyNoises.g + 0.25f * lowFrequencyNoises.b + 0.125f * lowFrequencyNoises.a;
    //使用lowFrequencyFbm对Perlin-Worley进行膨胀，塑造云层基本形状
    float baseCloud = Remap(lowFrequencyNoises.r, -(1.0 - lowFrequencyFbm), 1.0, 0.0, 1.0);
    
    //塑造云层在不同高度下的密度
    //在我的舞台场景下并不需要严格按照真实云层的高度-密度梯度去塑造云在不同高度下的形状，但这里保留这种写法，可以让该方法更通用。
    //weatherData:r->Coverage g->Precipitation b->CloudType
    float3 weatherData = SAMPLE_TEXTURE2D(_WeatherDataTexture, sampler_LinearRepeat, xzFraction).rgb;
    //暂时先不使用Cloud Type分布
    weatherData.z = 0;
    //给heightFraction增加点扰动，增加起伏效果（正好利用Worley Noise）
    heightFraction += (2 * weatherData.r - 1) * 0.03;
    //再手动Clip下
    weatherData.r += 0.25;
    
    float densityHeightGradient = GetDensityHeightGradientForPoint(positionWS, weatherData, heightFraction);
    baseCloud *= densityHeightGradient;
    
    //使用云层覆盖率的贴图来Clip
    float cloudCoverage = clamp(weatherData.r, 0, 0.999);
    float baseCloudWithCoverage = Remap(baseCloud, cloudCoverage, 1.0, 0.0, 1.0);
    //To ensure that density increases with coverage in an aesthetically pleasing way, we multiply this result by the cloud coverage attribute.
    baseCloudWithCoverage *= cloudCoverage;

    float finalCloud = baseCloudWithCoverage;

    if(baseCloudWithCoverage > 0.0)
    {
        //使用高频FBM噪声对云的边缘增加细节
        //TODO:Curl替换成离线的噪声图
        float2 curlNoise = ComputeCurl(uvw);
        //云的底部增加湍流效果
        uvw.xz += curlNoise.xy * (1.0 - heightFraction);
        float3 highFrequencyNoises = SAMPLE_TEXTURE3D(_CloudNoise3DTextureB, sampler_LinearRepeat, uvw).rgb;
        //构建FBM
        float highFrequencyFbm = 0.625f * highFrequencyNoises.r + 0.25f * highFrequencyNoises.g + 0.125f * highFrequencyNoises.b;
        float highFreqNoiseModifier = lerp(highFrequencyFbm, 1.0 - highFrequencyFbm, saturate(heightFraction * 10));
        //侵蚀（收缩）云的形状产生细节
        finalCloud = Remap(baseCloudWithCoverage, highFreqNoiseModifier * 0.2, 1.0, 0.0, 1.0);
    }
    
    return saturate(finalCloud);
}
```

达不到原文中那样好看的效果，自己调下来只能勉强看看（有待优化啊）。

![20240606221220](https://raw.githubusercontent.com/recaeee/PicGo/main/20240606221220.png)

![20240606221448](https://raw.githubusercontent.com/recaeee/PicGo/main/20240606221448.png)

云层翻涌效果如下。

![云层翻涌](https://raw.githubusercontent.com/recaeee/PicGo/main/云层翻涌.gif)

##### 2.4 暗边效应

到目前为止，我们仍然还缺少一个重要的东西——**云的暗边**。在《Horizon: Zero Dawn》之前并不存在对这个效果的研究文档，因此他们也根据实际现象做了一些思想实验。

观察下图，在现实中的云有一个规律：当视线接近光源投射光照方向时，云的凸起外轮廓部分会产生暗边，这种效果在凸团状的高密度云上更为明显。相对的，我们也可以这么理解，**云的每个凸起处之前的褶皱（凹点）会变得更亮**，这是由于内散射事件引起的（具体物理原因没怎么看明白）。

![20240606111213](https://raw.githubusercontent.com/recaeee/PicGo/main/20240606111213.png)

这种效果和一堆糖粉的效果很像，因此也称之为**糖粉效应**。

![20240606113008](https://raw.githubusercontent.com/recaeee/PicGo/main/20240606113008.png)

实际上，当我们意识到有这种现象（暗边效应），我们可以观察到它其实在很多情况下都能被观察到。

![20240606113147](https://raw.githubusercontent.com/recaeee/PicGo/main/20240606113147.png)

那么，我们如何实现该效果呢？在电影行业中，通常会使用更多的采样和更昂贵的Phase Function来收集一个点的光照贡献，可以通过暴力的方法来实现该效果。但是在游戏中，我们必须找到一个方法来近似它。

我们目前无法得到该效果的原因是我们的Transmittance公式（即BeerLambert定律）只是一个近似估计，只描述了光照强度随着消光系数、光学深度而衰减，并没有将暗边效果考虑在内，**云的表面总会散射出和它接收到光照相等的光能**，即光学深度Depth为0，Energy为1。

![20240606113547](https://raw.githubusercontent.com/recaeee/PicGo/main/20240606113547.png)

但根据暗边效果，我们可以认为当**越深入云的内部，内散射的概率会越大，从而有更多的光进入我们的眼睛**，如下图所示，红线Powder描述了该规律。如果我们将Powder和BeerLambert结合起来，噔噔，我们就获得了**BeerPowder函数**，即图中紫线。

![20240606113921](https://raw.githubusercontent.com/recaeee/PicGo/main/20240606113921.png)

从下图中可以直观地看到在不同的光照模型下的云效果的区别。

![20240606114026](https://raw.githubusercontent.com/recaeee/PicGo/main/20240606114026.png)

BeerPowder的Shader代码如下。

```c++
//Horizon中为实现暗边效应使用的Beer-Powder函数
float BeerPowder(float extinction, float depth)
{
    return exp(-extinction * depth) * (1 - exp(-extinction * depth * 2)) * 2;
}
```

##### 2.5 云的自阴影

在云的美术效果上，云的自阴影是让云变得好看的一个重要因素，因此我们需要在步进过程中考虑该点因素。**对于一个RayMarching采样点，自阴影的产生是因为从光源到该采样点的路线上存在高密度的云，从而产生遮挡的自阴影**。对于这点，其实在BeerLambert中对指数部分$\tau$的积分上是有所体现的。

![20240323004609](https://raw.githubusercontent.com/recaeee/PicGo/main/20240323004609.png)

当a-b上存在一段dx的消光系数$\sigma_t$特别大时，会让$\tau$变大，从而使透射率$T_r$变小，即采样点接收到光源的irradiance减少，从而产生变暗的效果。因此我们需要做的就很简单了——**在采样点向光源方向对$\tau$积分**。也可以理解成**二次的步进**，这部分的消耗其实也是比较大的。

Shader代码如下。

```c++
    //为了实现云的自阴影，需要从步进点向光源方向二次步进累积，计算positionWS接收到来自光源的irradiance，虽然费，但是效果好。
    float SampleAccumulatedCloudIrradiance(float3 positionWS, float3 lightDir, float3 viewDir)
    {
        float stepSize = SampleAccumulatedCloudDensityDistance / SampleAccumulatedCloudDensityStepCount;
        float cloudIrradianceMultiDepth = 0;
        for(int i = 0; i < SampleAccumulatedCloudDensityStepCount; i++)
        {
            float3 pos = positionWS + lightDir * stepSize * i;
            cloudIrradianceMultiDepth += extinctionAt(positionWS, SampleCloudDensity(pos)) * stepSize;
        }

        //只有当视线和光源方向一致时我们才使用BeerPowder
        //另外这里其实没有做到能量守恒，代码并不严谨
        if(dot(_LightDirection.xyz, viewDir) > 0)
        {
            return BeerPowder(cloudIrradianceMultiDepth, SampleAccumulatedCloudDensityDistance);
        }
        else
        {
            return BeerLambert(cloudIrradianceMultiDepth, SampleAccumulatedCloudDensityDistance);
        }
    }

    //可见性函数v，代表从光源位置到达采样点positionWS的比例
    float GetLightVisible(float3 positionWS, float3 lightDir, float3 viewDir)
    {
        //聚光灯衰减项
        float spotAttenuation = step(_LightCosHalfAngle, dot(lightDir, _LightDirection.xyz));
        //shadowmap阴影
        float shadowmapAttenuation = shadowAt(positionWS);
        //考虑体积阴影项，从光源到采样点的透光率Transmittance
        float transmittance = SampleAccumulatedCloudIrradiance(positionWS, lightDir, viewDir);
        //不考虑体积阴影项（即使是聚光灯，我们暂时也先不考虑距离衰减）
        // float transmittance = BeerLambert(extinctionAt(positionWS, 1), SampleAccumulatedCloudDensityDistance);

        return spotAttenuation * shadowmapAttenuation * transmittance;
    }
```

考虑自阴影与不考虑自阴影效果对比如下图所示。

![20240606224037](https://raw.githubusercontent.com/recaeee/PicGo/main/20240606224037.png)

##### 2.6 Noise生成工具

在RecaNoMaho中，提供了Perlin Noise、Worley Noise、Perlin-Worley Noise的代码生成工具，以及提供了将多张噪声纹理合并的工具，可以方便地产出Cloud需要的噪声资源。

Perlin Noise的生成代码参考了[【生成连续的2D、3D柏林噪声（Perlin Noise），技术美术教程】](https://zhuanlan.zhihu.com/p/620107368)。

Worley Noise的生成代码参考了[【生成连续的2D、3D细胞噪声（Worley Noise），技术美术教程】](https://zhuanlan.zhihu.com/p/620316997)。

Perlin-Worley Noise的生成代码参考了[【Tileable Perlin-Worley 3D】](https://www.shadertoy.com/view/3dVXDc)。


#### 3 【番外】再谈风格化

该节与本篇代码无关，仅仅是[上篇文章](https://zhuanlan.zhihu.com/p/688803084)对风格化的补充。写这部分的时候比较早，可能会和上文有割裂感，可以选择忽略！

在上篇文章中，提到了体积光效果的风格化，具体就是指让体积光变得更清晰明亮、明暗对比更明显，因此需要加入一些不那么物理的代码，因此上篇文章中选择的方法是对光源到介质的透光率$T_r$拉高，对从介质到相机的透光率$T_r$拉低。但实际使用下来，发现**参数比较难调控**——需要同时调整两个参数来控制亮部和暗部。

因此，我又想了一个没有道理的方法，**让没有处于阴影的亮部维持原来的亮度，只修改阴影中的体积光强度**，这样可以更方便地调整效果，方法比较简陋，仅供参考。因为即使抑制阴影中接受到的光子为0，相机光线还是会因为非阴影中的step而得到较多的亮度，因此主要思路就是让阴影中接受到的光子为负数，**让阴影中的step对最终的RGB值产生负贡献**，虽然一点道理都没有，但个人感觉效果还行？（如果有更好的方法，请告诉我！）

部分代码如下，主要就是最后return时根据阴影强度减去一个值(1 - shadowAt(positionWS)) * _ShadowIntensity。

```c++
    //返回介质中x处接收到的光线（RGB），以及x处到光源的方向
    float3 scatteredLight(float3 positionWS, float3 viewDir, float density)
    {
        //_LightPosition.w = 0时，为方向光，此时_LightPosition.xyx为方向光dir
        //_LightPosition.w = 1时，为SpotLight
        float3 lightDir = normalize(_LightPosition.xyz - positionWS * _LightPosition.w);
        float lightDistance = distance(_LightPosition.xyz, positionWS);
        //从光源到介质中x处的透射率，这里假设介质中x处到光源为均匀介质
        //TODO:SIMD?
        float transmittance = exp(-lightDistance * extinctionAt(positionWS, density));

        float3 radiance = _LightColor.rgb;
        float spotAttenuation = step(_LightCosHalfAngle, dot(lightDir, _LightDirection.xyz));
        float shadowAttenuation = shadowAt(positionWS);
        //考虑光源方向与片元到光源方向之间夹角的能量损失、阴影、透射率造成的衰减、散射系数、Phase Function
        radiance = radiance * spotAttenuation * shadowAttenuation * transmittance * scatteringAt(positionWS, density) * Phase(lightDir, -viewDir, positionWS, density);
        //考虑自发光、风格化_ShadowIntensity
        radiance = radiance + emissionAt(positionWS, density) * spotAttenuation - (1 - shadowAt(positionWS)) * _ShadowIntensity;
        
        return radiance;
    }
```

以下为效果对比，可以看到仅改变了阴影中的体积光亮度。

![20240411170804](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240411170804.png)

![20240411170820](https://raw.githubusercontent.com/recaeee/PicGo/main/recaeee/PicGo20240411170820.png)

#### 4 后记

这篇文章的制作跨度其实比较长，中间是断断续续写的，以及个人水平有限，文中以及代码中难免会存在问题（不，我相信是一定有问题），也希望同学们可以提出一些改进的建议。对于体积光渲染还有很多的工作没有做（我目前只参考SIG做出了云的简单效果，代码也比较糙，对于整个代码框架还有很多不合理的地方，同时之后还有AABBBox、TAA需要实现），但我最后还是决定先将本篇文章在这里结束，主要的原因是因为我已经太久没更新了（太怠惰了），另外之后对于体积光要做的内容也算比较多，总之先在这里存个档吧！

#### 参考

1. 【游戏开发相关实时渲染技术之体积光】https://zhuanlan.zhihu.com/p/21425792
2. 【光线相交算法与加速结构讨论】https://zhuanlan.zhihu.com/p/652102403
3. 【Unity实现体积雾】https://zhuanlan.zhihu.com/p/630539162
4. 【Frostbite Physically-based & Unified Volumetric Rendering】https://www.ea.com/frostbite/news/physically-based-unified-volumetric-rendering-in-frostbite
5. 【【Siggraph 2019】Creating the Atmospheric World of Red Dead Redemption 2】https://www.jianshu.com/p/01475e0c0baa
6. 【Vapor】https://vapor.rustltd.com/
7. 【《荒野大镖客2》中的体积云雾方案】https://zhuanlan.zhihu.com/p/638483468
8. 【SIG2015目录】https://advances.realtimerendering.com/s2015/index.html
9. 【《原神》主机版渲染技术要点和解决方案】https://zhuanlan.zhihu.com/p/356435019
10. 【CloudNoiseGen】https://github.com/Fewes/CloudNoiseGen
11. 【THE REAL-TIME VOLUMETRIC CLOUDSCAPES OF HORIZON ZERO DAWN】https://www.guerrilla-games.com/read/the-real-time-volumetric-cloudscapes-of-horizon-zero-dawn
12.  【【GPU Pro 7】Real-Time Volumetric Cloudscapes】https://www.jianshu.com/p/ae1d13bb0d86
13.  【GPU Pro 7——实时体积云（翻译，附Unity工程）】https://blog.csdn.net/b8566679/article/details/123446496
14.  【生成连续的2D、3D柏林噪声（Perlin Noise），技术美术教程】https://zhuanlan.zhihu.com/p/620107368
15.  【生成连续的2D、3D细胞噪声（Worley Noise），技术美术教程】https://zhuanlan.zhihu.com/p/620316997
16.  【Tileable Perlin-Worley 3D 】https://www.shadertoy.com/view/3dVXDc