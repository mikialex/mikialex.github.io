---
title: Build 3dmax scene from vrscene
date: 2019/3/6
cover_image: ../images/vrscene-in-3dsmax/cover.png
categories:
- techology
tags:
- 3dsmax
---

最近公司的业务需要将vrscene格式的文件导入到3dmax中。 3dsmax是广泛应用的建模软件， vrscene是 VRay 渲染器的场景描述文件，而 VRay 是当下最流行的离线渲染器。3dmax在安装了vray渲染器的插件之后，是支持vrsene文件的导入的，但是导入之后，在3dmax中是以vray的proxy形式存在，无法进行任何编辑操作。 根据vray的官方文档，vrscene仅仅是作为vray渲染器在不同建模平台进行数据交换的格式，并不支持反向导入到各个平台，形成可编辑的场景数据。我在maya上的测试结果也显示，vrscene是不支持反向导入成为可编辑数据的。而公司的业务要求后续的工作流需要vrscene的导入结果是正常的3dmax场景图对象，如何支持vrscene在3dmax中导入成可编辑对象，是我们主要讨论和解决的问题。

## 关于vrscene格式

即便vray非常流行，但是vrscene这个格式鲜有人知道，以至于网上这方面的资料很少。vrscene是VRay渲染引擎用来进行渲染的专用文件格式，本质上是一个字符串文本。这个文件格式包含了所有渲染所必须的信息，包括模型三维空间信息、材质信息、相机视角、渲染设置等。

我们的业务需要，只提取了部分关心的数据，其中，最主要最的部分如下图所示：

<img src="/images/vrscene-in-3dsmax/hierachy.png" style="width: 80%">


## 为什么vrscene不支持导入成为可编辑对象?

vrscene不支持这样的特性，我认为是有一定原因的。vray以插件形式支持了非常多种类的建模软件，每一种建模软件，都有自己的场景系统，材质系统，vray对渲染的支持，其实是将每一种不同的描述体系，映射到自己的体系中，vrscene的材质／场景描述体系，包括扩展能力，一般都比具体一种建模软件大，所以vrscene的描述体系，事实上几乎是各个建模软件的超集。 从软件的体系，导出到vrscene的体系，是没什么问题的，但是反过来，就是从超集回到子集的过程，不同的建模软件，子集是不同的，所以势必是有一些数据，vrscne可以描述，但是建模软件不行，所以就会丢失数据。 索性这部分都不做了。

## 解决方案

vrscene 本身场景格式易于解析，我调研了一些具体的文件，以及3dmax的api，最后的解决方案是，**直接在3dmax中解析vrscene，并使用相关api进行构建**。除此之外没有太多其他可行的做法。毕竟根据我们的需求，性能并不重要, 正确性只要基本满足就可以。

正确性理论上是无法满足的，正如上文提到的，我举几个例子。3dmax的blendmaterial和vraymaterial导出的vrscene，plugin object 都是BRDFLayered，但是他们无论是用法和效果都有很大不同。又比如你画一个几何体，这时候3dmax里它是一个无材质的东西，不过在viewport里是有颜色的，这个无材质的东西导出却是有材质的BRDFDiffuse。。 所以很多东西事实上只能模拟, 不必强求。

## 主要解析流程

1 从字符串解析出数据结构，这部分使用基于pyparing单文件的parser combinator库，构建vrscene文法，然后直接解析到嵌套的字典结构。 这部分主要参考了[vb30](https://github.com/ChaosGroup/blender_with_vray_additions)（一个python的blender插件，也是导入vrscene，不过不是用来构建场景）的解析部分， 这部分在它的 [submodule](https://github.com/ChaosGroup/vray_for_blender_python_utils) 里， 主要是这个[文件](https://github.com/ChaosGroup/vray_for_blender_python_utils/blob/master/VRaySceneParser.py), 不过有一些bug，而且不支持include指令，不支持中文（3dmax的python执行环境是2.7），不过这些都不难解决。

2 遍历字典提取几何数据，和关心的plugin object 到集中的map。

3 针对关心的plugin object，以自底向上的顺序，构建贴图，材质，xx等中间资源。**因为缺乏python api，具体的创建使用拼接maxscript字符串并执行实现**。材质名，vrscene字段名，mapName，等通过查表批量生成，特殊情况特殊对待。plugin之间的依赖以vrscene中的name为uuid。

针对每一种pluginObject，lookuptable的构造和流程基本如下图所示。

<img src="/images/vrscene-in-3dsmax/lookuptable.png" style="width: 100%">


自底向上构建，是为了防止资源重复创建。在子plugin的构建中，使用 [currentMaterialLibrary](http://help.autodesk.com/view/3DSMAX/2017/ENU/?guid=__files_GUID_D10CEFAB_E451_4A63_968E_01AEDEFCE928_htm) 来存储创建的对象，虽然这个东西是materiallib，但实测是可以作为通用的map来存储贴图，dirt和falloff等plugin的

4 最后遍历node map， 使用3dmax python api构建场景节点，并解压和去编码几何数据，创建几何对象，材质部分直接assign之前构建的中间对象。完成场景的构建。

这部分最坑的是，算是3dmax的python api了，3dmax主要支持三种语言，c++写lib插件，python或者maxscript写脚本，其中c++的sdk，算是一等公民的支持， maxscript作为平台独占的脚本，也是一等支持，但是python真的是不行，无论是文档，还是开发体验，都特别垃圾。文档只有接口列表，使用全部靠猜，至今还没有找到正确实现mesh的normal导入方法。开发调试时，代码有改动需要重启整个应用才能生效，似乎是因为在某个地方缓存了pyc文件。每当看着3dmax重启，界面巨卡无比，老态龙钟的样子，非常无奈。 并不要因为3dmax是多么的著名而认为它是一个好的软件，在很多地方上，我觉得充斥着糟粕。很多功能，就比如python的支持，根本就是为了支持而支持，就是完成任务，根本没有考虑用户怎么使用，好不好用，也事实上鲜有用户在使用这些功能。



### 其他参考资料

[3dmax python API 2014](http://docs.autodesk.com/3DSMAX/16/ENU/3ds-Max-Python-API-Documentation/index.html)

[build geometry use python](http://help.autodesk.com/view/3DSMAX/2018/ENU/?guid=__developer_maxplus_python_api_introduction_working_with_objects_html)

[clara.io vrscene](https://clara.io/learn/user-guide/data_exchange/vrscene_import)

