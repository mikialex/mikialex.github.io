---
title: 设计工具的框选实现
date: 2019/6/4
---

做设计的朋友们对设计软件中的框选是非常熟悉的。整理一下最近在产品中调研的cpu框选的技术实现。

通过屏幕空间范围和相机的参数，可以确定出一个世界空间的frustum。

相交模式
（以下相交包含contain）：for 场景物体，与框选frustum contain，且包含至少一个可见三角形则添加到结果集， 与框选frustum不相交的，剔除。 未剔除的进入obj内做详细测试：

obj是mesh：

无bvh：每一个可见面（正反剔除后的）依次测试和frustum是否相交， 相交则提前退出，该物体框选成功。
有bvh：bvh tree每一个tree节点， 如果node bbox被frustum contain，且该node的所有的三角形中至少有一个可见，则bvh提前退出框选成功。若进入叶子节点，则和无bvh退出模式相同。
obj是line：

每一个linesegment依次测试和frustum是否相交， 相交则提前退出，该物体框选成功。
若框选没有提前成功退出，均框选失败

包含模式
（以下相交包含contain）：for 场景物体，与框选frustum contain，且包含至少一个可见三角形则添加到结果集，与框选frustum不相交的，剔除。 未剔除的进入obj内做详细测试：

obj是mesh：

无bvh：每一个可见面（正反剔除后的）依次测试和frustum是否包含， 只要有一个不包含则提前退出，该物体框选不成功。
有bvh：bvh tree每一个tree节点， 如果node bbox被frustum contain，则该node可以被剪枝。若进入叶子节点， 提前退出和无bvh相同。
obj是line：

每一个linesegment依次测试和frustum是否包含， 只要有一个不包含则提前退出，该物体框选不成功。
若框选没有提前失败退出，均框选成功

需要实现的现在数学库没有的东西：
frustum contain box

frustum contain shpere

frustum contain triangle

frustum intersect triangle

frustum contain line segment

frustum intersect line segment

性能评估：
框选的成本比点选高很多，估计可能在5-10倍。contain模式可能开销会高。

frustum有6个面要判断。每个面和primitive和位置关系判断似乎都比ray要开销高。需要深入进入mesh内进行判断的情况也比raycast多很多。

其他社区实现或者讨论：
有用的：

https://cesiumjs.org/Cesium/Build/Documentation/Scene.html#pick

https://cesiumjs.org/Cesium/Build/Documentation/Scene.html#drillPick 

https://github.com/AnalyticalGraphicsInc/cesium/blob/1.57/Source/Scene/PickFramebuffer.js

cesium主要是应该gpu实现的

gpu做不到遮挡后面的pick，这个需要看产品需求。



osg intersector, 抽象了普通的多面体求交， 使用kd tree

https://github.com/openscenegraph/OpenSceneGraph/blob/master/src/osgUtil/PolytopeIntersector.cpp



其他：

https://github.com/mrdoob/three.js/issues/1826

https://github.com/vasturiano/react-force-graph/issues/43 2d的，简单，没有太多可比性

http://output.jsbin.com/tamoce/3/  three的，实现不是很正确？