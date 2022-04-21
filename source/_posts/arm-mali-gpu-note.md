---
title: arm mali gpu note
date: 2022/4/21
categories:
- techology
tags:
- arm, optimization, graphics
---

[https://www.youtube.com/watch?v=tnR4mExVClY&list=PLKjl7IFAwc4QUTejaX2vpIwXstbgf8Ik7](https://www.youtube.com/watch?v=tnR4mExVClY&list=PLKjl7IFAwc4QUTejaX2vpIwXstbgf8Ik7)

摘录此系列视频中提到的一些点，复习一下移动端图形优化（视频提到的顺序）

- 功耗问题
    - 整个系统的功耗是有预算的（3w），需要在cpu，gpu，isp，npu...之间share
    - 移动平台cpu，频率约低，能效越高
    - 带宽就是能耗，per gb per second ⇒ watt
    - 尽可能多线程，降低每个线程的负载，使得cpu尽可能低频率运行
- 硬件实现hidden surface removal使得，即便drawcall提交顺序不利于earlyz，优化也能起作用
- TBDR： Immediate模式，cheap几何，expensive着色（高带宽）更好更容易的并行能力。tbdr：expensive几何（分块，带宽），cheap着色（低带宽），较低（粗粒度）的并行能力（pipline各个阶段以renderpass为界进行工作切换），更需注意的api使用。
- 基本上大部分移动平台都是tbdr一个原因是，分辨率高，大部分应用的成本在着色上。有了onchip的tile，很多带宽昂贵的操作都free了
- 处理器特性：
    - cpu：低线程数量 ⇒ 依赖单核性能 ⇒ 通过降低延迟提高性能
    - gpu：高线程数量 ⇒ 依赖整体吞吐量 ⇒ 通过提升能效提高性能
- gpu有较多的线程可利用，大量线程在多个任务间切换来避免计算单元闲置，提高并行能力，提高吞吐，即便这样使得延迟大大提升（无所谓）。这种吞吐量提升的方式，避免了硬件设计上比如cpu端对分支预测，乱序预测执行（这些东西的存在其实就是为了主动发现额外的并行能力），和大的多级缓存的支持，降低了复杂程度和能耗。
- mali gpu市占率： 50% 手机设备， 85%智能电视
- constant tile elimination : 跳过没有变化的tile写入来减少带宽
- AFBC arm frame buffer compression：压缩fb减少带宽
- IVDS index driven vertex shading，mali gpu会将顶点着色器提前编译处理成两部分，提取位置计算靠前以提早进行图元剔除
- 有限资源下优化的策略： 严格计算budget，有多少带宽可用：对应能支持多大分辨率，多少目标fps，使用什么纹理格式？ 有多少计算能力可用： shader cycles per pixel
- 压缩纹理
- 仔细设计和考虑render/frame graph 数据流，尽可能避免tile memory外的存储访问
- tile上的操作，可以将多个pass合并，所以事实上，msaa的source target，渲染的中间target只要使用得当，事实上是根本不存在主存里，也不会访问主存，memoryless
- 对于vk
    - 简单的使用pass load的clear即可，而不单独使用clear command
    - 简单的使用pass resolve来resolve mass， 而不单独使用resolve command
- 对于opengl
    - pass的概念在驱动内还是存在的，等于说驱动会推测出pass，load，store的各种operation
    - 一些操作会导致pass被split，等于说需要重新进行昂贵的load，需要避免，比如fbo绑定状态的修改，同步状态的修改。
    - 一些操作是有利于驱动作出优化的，例如在绑定完fbo时，立刻调用clear，驱动会推测出此pass是loadop 不需要load。
    - AFBC只支持同时attach depth23 stencil8格式的fbo，所以为了减少带宽消耗，故意不绑定stencil之类的做法是错误的。类似的，f32支持但是f16不支持
- 从graph的角度，可以轻易识别出不必要的写（没有后续pass读的输出），可以识别出可以进行merge的pass。
- AFBC的使用，对于opengl而言，是完全驱动控制的，但是对于vk而言，是通过textureview的flag（IMAGE_TILING_OPtiMAL, IMAGE_USAFE_STORAGE...）显式控制的
- tile的存储大小是固定的，但是物理大小是可变的。使用mrt，msaa，会导致tile物理尺寸变小，降低性能
- 移动设备横放和竖放其实是有影响的，如果swapchain的输入方向和屏幕不一致，可能会有昂贵的读取成本。opengl驱动会处理，vk需要用户处理
- prez pass一般不适合移动端，不过例外是可以用来绘制那些不能用hsr优化的动态物体
- 尽可能减少几何数据的带宽，甚至使用f16，2 float normal
- 由于IVDS的存在，position是先被计算的，所以传统的interleave组织attribute的方式并不是最好的，因为position的访问被interleave的，缓存命中率低。所以最优是将非position的数据interleave，position相关的单独interleave，做两个array of structure
- 2d渲染，如果有能力预处理资源，可以在几何上将物体切成需要alphablend的不需要的部分来减少overdraw
- shader的优化一般被高估了，很多优化看起来没有生效其实是因为他们是不合法的（浮点数上界的边界问题）。
- workgroup share memory在mali gpu上和主存没有性能区别