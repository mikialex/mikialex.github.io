---
title: Side project retro in 2019 and future plan
date: 2019/12/28
---

## artgl

今年的side project主要是这个 <https://github.com/mikialex/artgl> 这个项目基本占据了我今年主要的业余编程时间。 <del>我写坑过许多项目，从目前的发展来看，ARTGL应该是不会坑掉了。</del> 今年完成了主要的api设计和开发，主要的关于场景和渲染流程构建方面的API已经成熟。我对目前的整体架构还是较为满意的：经过年底的一番调整，目前整个repo已经完成了模块化的迁移，不同的功能被分割在不同的package里，并使用lerna进行管理。在模块化的迁移中，为了拆解不必要的依赖和耦合关系，做出了不少设计上的变动。这些变动提升了架构上的质量。现在整个项目中，每一个package，每一个类的的职责，界限，已经非常明确，干净。

详细来说：

* 实现了一个独立的WebGL封装库： @artgl/webgl。 包装原始api，对webgl状态切换做diff，记录，调试，以及提供了功能级别的webgl1/2的自动降级（比如使用此layer提供的instance draw/VAO，可以无缝切换webgl1/2）。独立性非常高，可以被用于任何其他的项目

* 数学库，@artgl/math, 这里主要是three的数学库，ts化。但还有很多没有完成迁移，基本上是需要什么迁移什么。

* utils， @artgl/shared， 工具util

* @artgl/shader-graph, a shader linker。主要是用户可以包装任意glsl字符串函数，我们会简单提取输入输出，以节点的形式，抽象shader中的计算。用户可以使用builder style的API，构建一个完整shader的计算图，两个计算图合在一起，配合一些特殊的IO节点，便可以描述一个program的所有信息。shader graph提供面向 @artgl/webgl的完整program的代码生成。

* @artgl/core 核心模块。 提供了engine的实现，renderable的抽象。其中renderable中描述shading方式的部分使用的就是 @artgl/shader-graph。当然还引入了构筑于此的更上层的概念，比如shader graph的装饰器，和shader graph uniform的提供器。这部分的抽象我个人认为比较精彩，甚至light， camera等抽象是构建于次的。 当然除了shading，还有其他的诸如geometry。这一模块基本上定义了在artgl里，如何描述一个绘制的物体，以及实际上去进行绘制的实现。

* @artgl/lib-geometry 和 @artgl/lib-shading 主要是核心实现和库实现的分离。前者放各种geometry， 后者放各种shading decorator，以及glsl片段。

* @artgl/scene-graph。 当然就是场景图了，将各种renderable以层次结构组合在一起，实现典型的场景整体描述。初次之外还有IO的一些东西，I可以有各种controller loader，O可以有各种exporter。因为现在就没写几种，就没有必要再独立成包了。

* @artgl/render-graph。 同样是builder style的API，以声明式的方式构建后处理pass graph。

除了这些主要模块之外，还有若干实际应用测试分别是：

* @artgl/example， 会有各种小example，兼顾回归测试，可能可以以后codegen出文档，网站。

* @artgl/viewer， 会是一个比较完善的场景编辑器。

再除了这些，还有些高度实验性质的部分：

* wasm-scene， 测试和发掘 Webassembly Rust 潜力，包括框架集成的可能性。

* parser, 目前有个简单的lr1 parser，主要是想解决shading language在框架集成方面的问题，以及更多可能性的探索。

### 目前的不足

#### 性能一般

目前简单的benchmark下来是不如three的。主要可能是这些原因： 1 过于灵活和丰富的抽象，这导致在three里功能是用代码写死的，直接被JIT了，而我这里要用代码自己走各种数据结构。 2 使用了不利于JIT的特性，在chrome下某些情况相较于three可能慢一倍，然而去fireFox，这个差距就远远没有这么多，至于为什么，那就很难说得清了，总之就是没有走到什么fast path上。 3 细节上没有使用高性能js的最佳实践，比如我就是要用instance of，而不是硬判断标志属性，这样ts会立刻收窄类型。 又比如我没有为了性能而特意去掉foreach。

#### 测试和用例不足， 完善程度低

由于时间精力有限的原因，几乎没有什么单元测试，毕竟没法全职搞这个。example太少，不利于找到设计上的缺陷。搜索一下todo，就知道在各个角落都有各种case没有处理全，有些甚至涉及基本功能。只要我自己example没有写到那里我就不会补的那种。其实也主要是上述原因。

---

### 2020 计划

我最早的考虑是整个项目用rust/wasm 重写，以及first class WebGPU支持。后来经过具体的一番操作和研究，我认为这个几乎是另一个项目了。但是，我依然觉得如果提供js/ts良好的支持，依然是有价值的。这一点比较矛盾。wasm之前也是调研许久，只不过是scope的问题。最早是计划用于加速场景树drawcall生成的。后来发现render的js也很慢，所以engine，renderable也得做进去。 那么webgl也没必要是ts了，所以需要全部重写。

目前的考虑是，artgl会依然是一个ts项目，不会有任何wasm的东西存在。目前只会维持一个对渲染最基本的东西的轻量级抽象，而不再提供更多的实质性实现。简而言之，如果没有实际应用的化，我在明年不会有太多精力投入到这个项目中来，短期之内不会有新的功能开发。

## rendiation

取而代之，明年主要focus的side project会是这个 <https://github.com/mikialex/rendiation>。这会是一个跨平台的rust的图形渲染库，后端是WebGPU，会支持所有常用桌面端移动端平台以及web。

语言上是的，我现在决定all in rust了。选择一门语言是非常重要的，就像决定在哪里买房子一样。1，我彻底受够了脚本语言的低效，2，我需要极致的性能，所以任何GC的语言都再见了，那么似乎只有c/c++/rust三者可选。经过我一年多的观察和学习，我几乎认定，如果我之后要做特别care性能的项目，都会尽可能的选择rust。我暂时不想写很多内容安利这门语言，懂的人自然懂。 另一个考虑是 rust的跨平台图形的基础设施还不错，即便不完善也是蓬勃发展，特别是会主要使用到的wgpu这个crate，会直接作为firefox自己的webgpu实现。在后端支持上，选择webgpu是为了在使用最现代的图形API且保持最大化的跨平台支持。

这个项目很大程度更像是在重做我今年做的东西，只不过会有极致的性能，使用最先进的API，具备完美的跨平台能力。我以后的任何业务图形积累都会使用这个作为底层平台，比如GUI，可视化，等等。 这其中会遇到很大的挑战，一方面主要是真正的系统级编程，包括语言上的，平台上的新东西，另一方面是新一代的图形API适应。希望在明年这个时候，成果依然如今年一般丰厚。

<img src="/images/side-project-in-2019/github.png" style="width: 100%">