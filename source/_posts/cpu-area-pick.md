---
title: 设计工具的框选实现
date: 2019/6/4
---

做设计的朋友们对设计软件中的框选是非常熟悉的。整理一下最近在产品中调研的cpu框选的技术实现。

## 需求

所谓框选，就是用户交互式的拖出一个屏幕上的矩形区域用来选择场景对象。这里根据不同的业务存在几种不同的需求： 

比如：遮挡情况：是否需要只选中最前面的物体，而排除那些完全被遮挡住的物体， 还是即便是被完全遮挡住的物体也要选中？ 某些在空间上完全处于被遮挡的情况，在某些渲染模式下（比如线框），被遮挡的物体依然完全可见，这种情况下又如何处理？某些情况下，物体没有被完全遮挡，只有一个像素暴露在外，甚至因为渲染的原因，只有半个像素糊在边界上，这种情况用户似乎又不是需要选中的，那实现上又如何应该处理呢？ 当我们实现点选，选择最前面的，是很好实现和分辨的，因为总归是一个ray的事情，而对于框选来说，遮挡/最前面，从实现的角度而言，这个需求就显得复杂和含糊。从我过去作为一个设计出身的经历而言，我已经很适应在一个mesh上框出一堆face，但是反面也会被框住的事情。事实上因为这些种种的原因，这设计软件中，灵巧的调整各种视图，使用不同的工具，快速的选中需要的东西其实也是关于生产力的技巧。

又比如：相交还是包含？ 相交是指框选框和物体相接触就认为选中，包含是指框选框完整包含物体才认为选中。设计软件一般这两种模式都会提供，某些情况下，使用相交会使得选择变得容易，不过又容易多选，在我看来是更符合我的直觉的，当然，这其实是使用习惯上的分别

## 实现

在不考虑遮挡关系，考虑如何实现一个cpu版的框选。

由于我们的实现是在threejs的基础上，three的场景数据种类比较丰富，对象的类型mesh？ line？ points？几何的类型，Geometry？ BufferGeometry？ BufferGeometry里有没有index， Material是array还是一个，Attributes是interleaved怎么办？各种事情导致code写起来非常复杂。three也没有提供一个简单的针对drawable的东西foreach primitive的通用方法，这些因素归结于codebase提供的复杂性我们不必深究。 核心还是mesh， line， points三种不同的primitive的讨论。

通过屏幕空间范围和相机的参数，可以确定出一个世界空间的frustum。

bounding的提前退出：如果一个物体的bounding完全处于这个frustum之外，那么肯定是选不中的，提前退出。 如果一个物体的bounding完全处于这个frustum之内，在包含模式下，肯定是选中了，提前退出。这里需要额外考虑，bounding需要使用数据的，而不是渲染的，因为shadow之类的会产生影响。

如果frustum和box相交，那么计算就变得额外的多，简而言之：

相交模式：foreach primitive，依次测试和frustum是否相交， 相交则提前退出，该物体框选成功。否则失败。
包含模式：foreach primitive，依次测试和frustum是否包含， 不包含则提前退出，该物体框选不成功。否则成功。
以上的 primitive 需要根据情况做面序剔除。

在使用BVH的情况下，利用空间结构我们可以做更多的提前退出：

相交模式：traverse bvh tree每一个tree节点， 如果node被frustum contain，且该node的所有的pimitives中至少有一个可见，则bvh提前退出框选成功。
包含模式：traverse bvh tree每一个tree节点， 如果node被frustum contain，那么此node被剪枝。

<!-- 需要实现的现在数学库没有的东西：

frustum contain box

frustum contain shpere

frustum contain triangle

frustum intersect triangle

frustum contain line segment

frustum intersect line segment -->

性能评估：

框选的成本比点选高很多，估计可能在5-10倍。contain模式可能开销会高。

frustum有6个面要判断。每个面和primitive和位置关系判断似乎都比ray要开销高。需要深入进入mesh内进行判断的情况也比raycast多很多。

其他社区实现或者讨论：

有用的：

https://cesiumjs.org/Cesium/Build/Documentation/Scene.html#pick

https://cesiumjs.org/Cesium/Build/Documentation/Scene.html#drillPick 

https://github.com/AnalyticalGraphicsInc/cesium/blob/1.57/Source/Scene/PickFramebuffer.js

cesium主要是应该gpu实现的， gpu做不到遮挡后面的pick


osg intersector, 抽象了普通的多面体求交， 使用kd tree

https://github.com/openscenegraph/OpenSceneGraph/blob/master/src/osgUtil/PolytopeIntersector.cpp



其他：

https://github.com/mrdoob/three.js/issues/1826

https://github.com/vasturiano/react-force-graph/issues/43 2d的，简单，没有太多可比性

http://output.jsbin.com/tamoce/3/  three的，实现不是很正确？

https://community.khronos.org/t/intersecting-a-3d-segment-with-perspective-frustum/59537/3

