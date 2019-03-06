---
title: 根据ANGLE 实现原理的 WebGL优化建议
date: 2018/11/17
---


<img src="/images/webgl-opti-guide/anglelayer.png" style="width: 100%; max-width:300px">

> 本文来自 webGL insights 的第一章节阅读整理

chrome 浏览器目前在使用份额上表现出统治性的地位，而 windows 则是最常用的个人桌面操作系统，所以 windows 平台下 chrome 的一些 webgl 的优化知识需要被重点关注。在windows平台下，chrome 和 firefox 的 webgl 后端支持是 ANGLE 。ANGLE 是 Google 发起的开源项目，最初是为了使用 D3D 9 来实现 OpenGL es 2.0，而 WebGL 正是基于前者。当然， chrome不仅把 ANGLE 作为 webGL 的后端，还用来支持整个浏览器的渲染加速。现在 ANGLE 不仅拥有 D3D 9的后端， 还有 D3D 11 以及 OpenGL的后端，可以说是将 opengl es 的 API 移植到桌面的首选支持。

ANGLE 会将 webgl 使用的 glsl 翻译成 d3d 的 hlsl， 并且运行时验证 gl 的调用是否合法。所有的基于状态机的光栅化 API，都无非是设置状态，和执行渲染命令。ANGLE 会将等价的状态设置和等价的渲染调用进行翻译，直接使用底层 API，所以只有很小的性能损失。ANGLE 并不是一个模拟器。

因为 OpenGL 和 D3D 在本身的设计和支持上存在差异，所以一些 OpenGL 原先轻而易举可以做的事情，不一定能很好的映射到对应的高效的 D3D 实现， 所以在 ANGLE 的基础上，我们可以总结出一些最佳实践。归纳如下。

##### 避免使用LINE_LOOP 和 TRIANGLE_FAN

D3D 里没有 LINE_LOOP 这种 DrawMode，TRIANGLE_FAN 在 D3D 11 里不被支持。ANGLE 需要重写 index buffer来支持。

##### 总是使用新的纹理，而不是对旧的做出修改

D3D 的纹理 API 需要在创建纹理的时候，就确定所有参数。而为了支持纹理属性的修改，ANGLE 需要维护多份纹理拷贝，并且在绘制时产生额外的延迟。

##### 在剪切和蒙板开启时，不要 执行Clear操作

D3D 的 clear 操作会无视剪切和蒙板，而 GL clear 时会考虑剪切和蒙板。 ANGLE 需要绘制一个四边形来支持剪切效果，并且产生额外的状态切换成本。

##### 使用多边形来实现带宽度的线

chrome 的 webgl 线宽不被支持时一个很经典的事情，甚至在汇报 chrome bug 的网站，chrome 的开发者直接回应说，“线的宽度是个没有意义的概念” 这个事情想想真的比较搞笑。不过从数学的意义上讲，线的宽度的确还真的没有什么意义。

任何现代图形硬件上，带宽度的线都不是原生的基本图元。所以需要模拟实现，这个模拟，要么应用层来做，要么驱动层来做。带宽度的线其实很复杂，joint style各种各样，今天支持了这种，明天还会有人问为什么不支持那一种，所以其实我也是认同 chrome 的做法的，这个事情就应该放在应用层实现。

##### 在顶点 Buffer 中，避免使用三通道的 Uint8Array／Uint16Array

主要是 D3D 底层对三通道的数据支持有限， ANGLE 会自己转成四通道的数据。如果是 dynamic 的， 甚至每次画之前都会转换。

##### 在索引 Buffer 中， 避免使用 Uint8Array

D3D 不支持在索引 Buffer 中使用 8位的数据。ANGLE 会转化回 16 位格式。

##### 避免在 16位索引 Buffer 中，使用 0xFFFF 值

在 D3D 11 中， 这个值具有特殊意义，代表三角形 strip-cut， 当是，这个是个 bug 被发现才被改正的。 ANGLE 一旦发现有这个值，会直接转化使用 32 位的数据。

##### 正确使用 buffer 标记

需要 STATIC_DRAW , 就指明。不然，ANGLE 会反复在内部执行数据转换。

##### 总是指明片元着色器浮点精度

之前据说写不写无所谓，后来强制必须要写了。。

##### 不要使用 Rendering Feedback Loops

Rendering Feedback Loops 是指同时在同一个纹理上进行渲染和采样，／／我都不知道可以这样？？ 总之 只有D3D 9 支持，现在一定都不支持。


