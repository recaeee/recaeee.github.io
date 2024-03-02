---
layout:     post
title:      "【VulkanTutorial学习笔记——14】Loading models"
subtitle:   ""
date:       2024-02-16 12:00:00
author:     "recaeee"
header-img: "img/in-post/default-bg.jpg"
tags:
    - Vulkan
---
# 【VulkanTutorial学习笔记——14】Loading models

![20240216013825](https://raw.githubusercontent.com/recaeee/PicGo/main/20240216013825.png)

#### 0 写在前面
本系列为个人在跟随Vulkan Tutorial学习过程中记录的一些笔记，一是防止自己看着看着弃坑，二是希望能在看过之后多少留些印象，三是想在啥时候忘记了能快速地回忆一些Vulkan知识点。

出于以上这些目的，该系列的学习笔记可能会写得较随意一些，并且对于主观判断不重要的东西可能会略过，因此也不保证该系列笔记的学习质量，可能有些东西不经过脑子就会写出来。

我是一个Vulkan的小白，对于Vulkan尚不熟悉，笔记中可能存在错误，或者概念不明确的地方，也希望看的人可以批评斧正~

这一章，我们会实践加载一个3D模型，在其中，我们会了解到一个OBJ文件的组成，使用库加载OBJ文件，以及使用unordered_map来进行顶点去重。和上一章一样，依然是难度系数很低，同时正反馈很高的一章，因为我们会加载出好看的模型啦~

原教程链接：
https://vulkan-tutorial.com/Introduction
本章节链接：
https://vulkan-tutorial.com/Loading_models

#### 1 介绍 Introduction

现在我们准备让程序渲染带纹理的3D meshes，但是目前vertices和indices数组离存储的几何图形并不是十分因吹斯听。在这一章，我们会拓展程序，从一个实际的文件加载顶点和索引数据，让显卡实际地干些活~

许多图形API教程会有一章让读者自己写一个OBJ Loader。这样做的问题是，任何有趣的3D应用程序很快就会需要此文件格式不支持的功能，例如骨骼动画Skeletal animation。在这一章，我们会从一个OBJ模型中加载mesh数据，但是我们会更关注将mesh数据和程序本身的集成，而不是从文件加载它的细节。

#### 2 库 Library

我们将会使用[tinyobjloader](https://github.com/tinyobjloader/tinyobjloader)库来**从一个OBJ文件加载顶点和三角面**。它非常快，并且容易集成，因为它是单个文件，和stb_image一样。

配置过程略，可配合原文食用~

#### 3 样品网格 Sample mesh

在这一章，我们还不会启用光照，所以我们使用一个Sample mesh，它的光照烘培在了texture中。一个找到这样的3D模型的简单方法是在[Sketchfab](https://sketchfab.com/)上查找3D扫描物体。该网站的很多模型都可以获取到OBJ格式的文件，并且拥有许可证。

在教程中，我们使用[nigelgoh](https://sketchfab.com/nigelgoh)(CC BY 4.0)的[Viking room](https://sketchfab.com/3d-models/viking-room-a49f1b8e4f5c4ecf9e1fe7d81915ad38)模型。教程作者调整了模型的大小和方向，将其用作当前几何体的替代品。

obj和png资源文件请到[原教程](https://vulkan-tutorial.com/Loading_models)中下载~


我们可以随意使用自己的模型，但是确保其只由一个材质组成，并且尺寸约为1.5x1.5x1.5单位。如果比这个尺寸大，那么我们需要改变view矩阵。将该模型文件放入新的models文件夹，将texture放到textures文件夹。

在程序中，我们定义两个新的配置变量表示**模型和texture的路径**。

```c++
//使用常量而不是硬编码的width和height，因为我们会多次引用这些值
const uint32_t WIDTH = 800;
const uint32_t HEIGHT = 600;

//模型和纹理路径
const std::string MODEL_PATH = "models/viking_room.obj";
const std::string TEXTURE_PATH = "textures/viking_room.png";
```

更新**createTextureImage**来使用该路径变量（之前的不用啦）。

```c++
		stbi_uc* pixels = stbi_load(TEXTURE_PATH.c_str(), &texWidth, &texHeight, &texChannels, STBI_rgb_alpha);
```

#### 4 加载顶点和索引 Loading vertices and indices

接下来，我们从model文件中加载顶点和索引，我我们现在可以移除全局的vertices和indices了。使用非常值的容器作为类成员来代替它们。

```c++
	//mesh顶点和索引数据
	std::vector<Vertex> vertices;
	std::vector<uint32_t> indices;
	//Vertex buffer
	VkBuffer vertexBuffer;
	//实际分配的vertex buffer的内存（device local，可以认为是显存）
	VkDeviceMemory vertexBufferMemory;
```

我们需要**将indices的类型从uint16_t改成uint32_t**，因为我们将会有**超过65535个顶点**。要记得也要改变**vkCmdBindIndexBuffer**参数。

```c++
		//绑定index buffer
		vkCmdBindIndexBuffer(commandBuffer, indexBuffer, 0, VK_INDEX_TYPE_UINT32);
```

tinyobjloader库和STB库被include的方式一样。我们直接include tiny_obj_loader.h文件，确保在一个源文件中**定义TINYOBJLOADER_IMPLEMENTATION**以包含函数体，并且避免链接器错误。

```c++
//OBJ加载库
#define TINYOBJLOADER_IMPLEMENTATION
#include <tiny_obj_loader.h>
```

我们现在来写一个**loadModel**函数，使用该库来使用mesh中的顶点数据填充**vertices**和**indices**容器。它应该在vertex和index buffer创建之前被调用。

```c++
	//initVulkan函数来用于实例化Vulkan objects私有成员
	void initVulkan()
	{
		...
		loadModel();
		createVertexBuffer();
		createIndexBuffer();
		...
	}

	void loadModel()
	{

	}
```

我们通过调用**tinyobj::LoadObj**函数来加载一个model到该库中的数据结构。

```c++
	void loadModel()
	{
		//加载一个model到该库中的数据结构
		tinyobj::attrib_t attrib;
		std::vector<tinyobj::shape_t> shapes;
		std::vector<tinyobj::material_t> materials;
		std::string warn, err;

		if (!tinyobj::LoadObj(&attrib, &shapes, &materials, &warn, &err, MODEL_PATH.c_str()))
		{
			throw std::runtime_error(warn + err);
		}
	}
```

**一个OBJ文件由positions、normals、texture coordinates和faces组成**。**Faces**由任意数量的顶点组成，其中每个顶点通过**索引index**引用一个position、normal和/或texture coordinate。着使得不仅可以**重用整个顶点**，还可以**重用单个属性attribute**。

**attrib**容器包含了所有的positions、normals和texture coordinates在它的**attrib.vertices**、**attrib.normals**和**attrib.texcoords**向量。**shapes**容器包含了**所有单独的物体和它们的faces**。每个**face**由一个顶点数组组成，每个顶点包含了position、normal和texture coordinate属性的索引index。OBJ模型也可以为每个face定义一个材质和纹理，但我们会忽略这些功能。

当加载文件时会发生错误，err字符串包含了错误，warn字符串包含了警告，比如丢失材质定义。只有当LoadObj函数**返回false**时，才发生实际的**加载失败**。正如上文提到的，OBJ文件的**faces可以包含任意数量的vertices**，但我们的程序只会渲染三角形。幸运的是，LoadObj有一个可选的参数来自动对这些faces进行三角测量，默认情况下会启用该参数。

我们要将文件中的所有faces组合成一个模型，因此只需迭代所有shapes即可。

```c++
		for (const auto& shape : shapes)
		{

		}
```

**三角测量特性已经保证了每个face有3个顶点**，所以我们现在可以直接遍历每个顶点，将其直接复制到我们的verices向量中。

```c++
		for (const auto& shape : shapes)
		{
			//遍历每个顶点，将其直接复制到我们的verices向量中
			for (const auto& index : shape.mesh.indices)
			{
				Vertex vertex{};

				vertices.push_back(vertex);
				indices.push_back(indices.size());
			}
		}
```

为了简单起见，我们假设每个顶点都是唯一的，因此简单地自动递增索引。**tinyobj::index_t**类型的index变量，包含了**vertex_index**、**normal_index**和**texcoord_index**成员。我们需要使用这些索引来**在attrib数组中查找实际的顶点属性**。

```c++
				vertex.pos = {
					attrib.vertices[3 * index.vertex_index + 0],
					attrib.vertices[3 * index.vertex_index + 1],
					attrib.vertices[3 * index.vertex_index + 2]
				};

				vertex.texCoord = {
					attrib.texcoords[2 * index.texcoord_index + 0],
					attrib.texcoords[2 * index.texcoord_index + 1]
				};

				vertex.color = { 1.0f, 1.0f, 1.0f };
```

不幸的是，**attrib.vertices数组是一个float数组**，而不是像glm::vec3，所以我们必须手动对index乘以3。类似的，每个entry有2个纹理坐标分量。0，1，2的便宜用于访问X、Y和Z分量，或者纹理坐标的U、V分量。

现在**开启优化**（比如Visual Studio的Release模式）之后，运行程序。这是必要的，否则加载模型会非常慢（实际测试差距不大）。我们可以看到如下渲染结果。

![20240216005342](https://raw.githubusercontent.com/recaeee/PicGo/main/20240216005342.png)

这里我踩了一个坑，如果将obj文件包含到项目中，因为编译过程中产生的目标文件后缀为.obj，链接器会将模型文件当成目标文件与其他目标文件一起链接来生成最后的可执行文件，导致报错“LNK1136 无效或损坏的文件”。所以，注意**不要将模型obj文件包含到项目中**。

好耶~现在几何图形看起来正确，但是Texture怎么样呢？OBJ格式假设这样一个坐标系统，垂直方向的0意味着image的底部，但是我们**以从上到下的方向上传image到Vulkan中**，其中0表示image的顶部。通过**翻转纹理坐标的垂直分量**来解决该问题。

```c++
				vertex.texCoord = {
					attrib.texcoords[2 * index.texcoord_index + 0],
					1.0f - attrib.texcoords[2 * index.texcoord_index + 1]
				};
```

当我们再次运行程序，我们会看到正确的渲染结果。

![20240216005917](https://raw.githubusercontent.com/recaeee/PicGo/main/20240216005917.png)

经历了如此漫长的修炼，我们终于渲染出了一个好看的模型，泪目了，还是有些成就感的吧。

PS：当模型旋转时，我们可能会注意到背面（墙壁的背面）看起来有些有趣。这是正常现象，只是因为该模型并非真正设计用于从该侧面查看。（其实在实践中，也会有对模型进行背面预先剔除掉一些永远不可见的面来优化性能的方法，尤其对于固定视角的游戏。）

#### 5 顶点去重 Vertex deduplication

不幸的是，我们实际上还没有利用index buffer的优势。vertices向量包含了很多**重复的顶点数据**，因为许多顶点被包含在多个三角面中。我们应该**保留唯一的顶点**，并且使用index buffer重用它们。一个实现的最直接的方法是**使用map或者unordered_map来追踪唯一的顶点和各自的索引**。

```c++
//用于顶点去重
#include <unordered_map>

...

std::unordered_map<Vertex, uint32_t> uniqueVertices{};

for (const auto& shape : shapes)
{
	//遍历每个顶点，将其直接复制到我们的verices向量中
	for (const auto& index : shape.mesh.indices)
	{
		...
		if (uniqueVertices.count(vertex) == 0)
		{
			uniqueVertices[vertex] = static_cast<uint32_t>(vertices.size());
			vertices.push_back(vertex);
		}
		indices.push_back(uniqueVertices[vertex]);
	}
}
```

每次我们从OBJ文件读取一个顶点，我们检查我们是否已经见过具有相同的position和纹理坐标的顶点。如果不是，我们将其增加到**vertices**，并且存储它的索引到**uniqueVertices**容器中，在之后，我们增加该新顶点的索引到**indices**中。如果我们已经见过该顶点，我们就在uniqueVertices中**查找该顶点的索引**，并且只将该索引存储到indices中（vertices不会增加新元素）。

现在程序会编译失败，因为我们**使用了用户定义的类型Vertex结构体作为hash table的key**，它要求我们实现两个函数：**相等性测试**和**哈希计算**。前者可以通过在Vertex结构体中**重写==操作符**简单地实现。

```c++
		bool operator==(const Vertex& other) const {
			return pos == other.pos && color == other.color && texCoord == other.texCoord;
		}
```

Vertex的**哈希函数**是通过**指定std::hash的模板特化**来实现的。哈希函数是一个复杂的主题，但是[cppreference.com](http://en.cppreference.com/w/cpp/utility/hash)建议以下方法来组合一个结构体中的字段来创建质量不错的哈希函数。

```c++
namespace std {
	template<> struct hash<Vertex> {
		size_t operator()(Vertex const& vertex) const {
			return ((hash<glm::vec3>()(vertex.pos) ^
				(hash<glm::vec3>()(vertex.color) << 1)) >> 1) ^
				(hash<glm::vec2>()(vertex.texCoord) << 1);
		}
	};
}
```

该代码需要被放到Vertex结构体外部。GLM类型的哈希函数需要使用以下宏定义来包含。

```c++
//包含GLM类型的哈希函数
#define GLM_ENABLE_EXPERIMENTAL
#include <glm/gtx/hash.hpp>
```

哈希函数被定义在gtx文件夹，其意味着它在技术上仍然是GLM的实验性拓展。因此，我们需要定义**GLM_ENABLE_EXPERIMENTAL**来使用它。它意味着在未来API会改变成一个新版的GLM，但是实践中该API非常稳定。

现在，我们应该可以成功编译并运行程序。如果我们检查vertices的大小，我们可以看到**它从1500000降低到了265645**！这意味着每个顶点平均被6个三角形重用。这节省了很多GPU内存。

加载模型，Get！

#### 参考
1. 题图来自画师wlop
2. https://github.com/tinyobjloader/tinyobjloader
3. https://sketchfab.com/
4. https://sketchfab.com/3d-models/viking-room-a49f1b8e4f5c4ecf9e1fe7d81915ad38
5. https://developer.huawei.com/consumer/cn/forum/topic/0204137413099897772
6. http://en.cppreference.com/w/cpp/utility/hash



