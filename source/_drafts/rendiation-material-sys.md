# Rendiation in 2020

Rendiation是我长期开发维护的一个图形渲染引擎，自去年年底12月提出了这个新的宏伟计划之后，我的今年大部分的业余编码时间全部投入这个项目之中，于今已由正好一年时间了，在此同步一些进度，做一些总结。

下文将rendiation简称为RRF.

## 逐模块介绍

整个项目使用rust语言，由多个划分清晰的模块构成，我将在下文逐一介绍这些模块的角色，设计实现特色，不足及后续开发计划。

### 基础数学库：

#### math

一个多维线性代数数学库，提供了Vector Matrix等相关数学类。

其中的设计上最大的成就是实现了**数学库的维度常量泛型标记**，这使得**维数直接融合进数学计算的抽象中，使得使用者可以直接实现出高维度/任意维度的实现，或是在实现中对类型约束添加维数约束**，如果你还不能意识到这是多么酷的一点可以稍后看后文的例子。

在实现维度抽象这一特性上，依赖rust还未稳定的const generics和specialization特性，保守估计这两个特性三年之内都无法进入稳定版，所以RRF将长期使用nightly版本编译器。

在实现维度抽象这一特性上， 我最早参考了[aljabar](https://github.com/maplant/aljabar)这个高维线代库的实现。但是最后放弃了它的做法，aljabar使用`[T; const N]`的array作为vector的容器，`[Vector<const N>; const N]`来作为matrix的容器，基本的针对n维的运算和操作都已实现，这并无不妥。但是因为算法是针对n维进行实现的，性能非常不理想，作者也表示会手工加入234维的特化实现，这也并无不妥。但是我实际自行加了一些特化的实现，和我当前的实现使用[criterion.rs](https://github.com/bheisler/criterion.rs)进行benchmark对比，结果依然在某些地方不理想。举一个差异较大的例子：aljabar的n维矩阵求逆使用的是LU分解，3维情况下时间消耗比直接的实现要慢30倍左右，我手工加入了直接做法的特化，但是性能也有数倍的差异。我判断原因可能基于array的数据结构在matrix这种case让编译器优化产生困难，或者disable了一些自动向量化的编译优化。我并不愿意为了这个抽象而牺牲性能，所以只能尝试其他的做法。

经过反复实验，最终我成功尝试出了真正零开销的维数抽象。做法并不复杂：使用trait的associate type的specialization：

```rust
pub trait DimensionalVec<T: Scalar, const D: usize> {
  type Type: Vector<T> + VectorDimension<D> + SpaceEntity<T, D>;
}

pub struct VectorMark<T>(PhantomData<T>);

// 标记vec2维度
impl<T: Scalar> DimensionalVec<T, 2> for VectorMark<T> {
  type Type = Vec2<T>;
}
// 标记vec3维度
impl<T: Scalar> DimensionalVec<T, 3> for VectorMark<T> {
  type Type = Vec3<T>;
}

...

// 标记vecn维度, 这里我都没有真的实现n维vec，如果实现也会直接使用aljabar的代码
impl<T: Scalar, const D: usize> DimensionalVec<T, D> for VectorMark<T> {
  default type Type = FakeHyperVec<T, D>;
}

// alias一下type，方便使用
pub type VectorType<T, const D: usize> = <VectorMark<T> as DimensionalVec<T, D>>::Type;

let a: Vector<f32, 3> = Vec3::new(...);

```

成功将维度标记在类型上带来的另一个好处是，这提高了代码复用的能力：

```rust
pub trait Vector<T: Scalar>:
  Sized + Mul<T, Output = Self> + Sub<Self, Output = Self> + Add<Self, Output = Self> + Copy
{
  #[inline]
  fn normalize(&self) -> Self {
    let mag_sq = self.length2();
    if mag_sq > T::zero() {
      let inv_sqrt = T::one() / mag_sq.sqrt();
      return *self * inv_sqrt;
    }
    *self
  }

  #[inline]
  fn length(&self) -> T {
    self.length2().sqrt()
  }

  #[inline]
  fn distance(&self, b: Self) -> T {
    return (*self - b).length();
  }

  #[inline]
  fn length2(&self) -> T {
    self.dot(*self)
  }

  fn dot(&self, b: Self) -> T;
  fn cross(&self, b: Self) -> Self;
}
```

你可以看到我只要针对具体的维度实现dot和cross，向量空间就有了距离的定义：这个不就是内积空间的定义吗？https://en.wikipedia.org/wiki/Inner_product_space。（这太酷了，我仿佛是在定义什么是数学。）于此我根本不需要针对各个维度的vector写normalize，length等实现，我的实现是直接针对任意维度的，大大减少了代码上的冗余。

数学库的进一步创新计划是准备将更多信息类型化，比如将矩阵的row/colum major编码进类型中，比如将vector是否是normalize的信息编码进类型中，比如将坐标系的手性编码进类型中。希望随着越来越严格的类型约束使得更多的逻辑错误在编译期被检查出来，同时不会带来任何运行时成本。

目前的实现，rotation相关的部分还没有被纳入体系中来，需要改进和完善。

####  math-entity

math-entity主要是基于math的几何库。比如box，sphere，circle，ray，plane等等类似于空间中实体 (entity) 的东西。

受益于math库的维度抽象能力。自然而然sphere和circle使用的是同一个数据结构：HyperSphere，box和rectangle使用的也是同一个数据结构：HyperAABB，line和plane：HyperPlane，无非只是维度的不同。

```rust
pub struct HyperSphere<T: Scalar, const D: usize> {
  pub center: VectorType<T, D>,
  pub radius: T,
}
pub type Circle = HyperSphere<f32, 2>;
pub type Sphere = HyperSphere<f32, 3>;

pub struct HyperAABB<T: Scalar, const D: usize> {
  pub min: VectorType<T, D>,
  pub max: VectorType<T, D>,
}
pub type Box3 = HyperAABB<f32, 3>;
pub type Rectangle = HyperAABB<f32, 2>;

pub struct HyperPlane<T: Scalar, const D: usize> {
  pub normal: VectorType<T, D>,
  pub constant: T,
}
...
```

math entity还定义了一些重要的trait：这里挑选几个重要的

```rust
// 用以标记这个几何对象的维度
// 需要实现对应维度的matrix的变换，以支持几何体在空间中的平移缩放选择
pub trait SpaceEntity<T: Scalar, const D: usize> {
  fn apply_matrix(&mut self, mat: &SquareMatrixType<T, D>) -> &mut Self;
}

// 为这个几何对象实现n维的Lebesgue测度（1维长度，2维面积，3维体积）
/// https://en.wikipedia.org/wiki/Lebesgue_measure
pub trait LebesgueMeasurable<T: Scalar, const D: usize> {
  fn measure(&self) -> T;
}

// 用以标记这个几何对象是否在n维是封闭的
// 封闭的几何体需要实现对应维度的测度，比如3维封闭就要提供体积的实现，2维封闭要提供面积的实现
pub trait SolidEntity<T: Scalar, const D: usize>:
  SpaceEntity<T, D> + LebesgueMeasurable<T, D>
{}

// 各种真正用户接口的trait，比如长度，面积， 周长， 表面积，体积
pub trait LengthMeasurable<T: Scalar>: LebesgueMeasurable<T, 1> {...}
impl<T: Scalar> LengthMeasurable<T> for T where T: LebesgueMeasurable<T, 1> {}

pub trait AreaMeasurable<T: Scalar>: LebesgueMeasurable<T, 2> {...}
impl<T: Scalar> AreaMeasurable<T> for T where T: LebesgueMeasurable<T, 2> {}

pub trait VolumeMeasurable<T: Scalar>: LebesgueMeasurable<T, 3> {...}
impl<T: Scalar> VolumeMeasurable<T> for T where T: LebesgueMeasurable<T, 3> {}

// 只有3维物体实现2维Lebesgue测度才能生成表面积方法，这样就和面积区分出来了
pub trait SurfaceAreaMeasure<T: Scalar>: SpaceEntity<T, 3> + LebesgueMeasurable<T, 2> {...}
impl<T: Scalar> SurfaceAreaMeasure<T> for T where T: SpaceEntity<T, 3> + LebesgueMeasurable<T, 2> {}

// 只有2维物体实现1维Lebesgue测度才能生成周长方法，这样就和长度区分出来了
pub trait PerimeterMeasure<T: Scalar>: SpaceEntity<T, 2> + LebesgueMeasurable<T, 1> {...}
impl<T: Scalar> PerimeterMeasure<T> for T where T: SpaceEntity<T, 2> + LebesgueMeasurable<T, 1> {}

// 实现图元对Target的包含关系判断
// 需要两者是统一维度的，且自己是封闭的
pub trait ContainAble<T: Scalar, Target: SpaceEntity<T, D>, const D: usize>:
  SolidEntity<T, D>
{
  fn contains(&self, items_to_contain: &Target) -> bool;
}

// 实现图元到给定类型的封闭包围体转化
// 需要两者是统一维度的，且目标是封闭的
pub trait SpaceBounding<T: Scalar, Bound: SolidEntity<T, D>, const D: usize>:
  SpaceEntity<T, D>
{
  fn to_bounding(&self) -> Bound;
}

// 实现图元相交
pub trait IntersectAble<Target, Result, Parameter = ()> {
  fn intersect(&self, other: &Target, param: &Parameter) -> Result;
}

```

想必这可以感受到math库提供的维度抽象的力量了吧。

### 组件及算法实现：

#### render-entity

构建与math及math entity之上的是render-entity。render-entity集合了若干常用于渲染引擎中的数据结构，目前只有相机，控制器组件，其他组件待补充

另外颜色相关的实现也存在于此，颜色方面主要的特色是将色彩空间信息编码进类型中，可以避免错误混淆不同颜色空间中的颜色计算，同时也能一更优美的方式编写色彩空间转化的实现。

#### space-indexer

实现了若干空间加速/空间索引数据结构，BVH，八叉树，四叉树等。尽可能的提供优美，通用性高，性能卓越的实现。

举例说：

```rust
// 例如八叉树和四叉树和二叉树共用了一个实现，只是维度不同
pub type BinaryTree = BSTTree<Binary, 2, 1>;
pub type QuadTree = BSTTree<Quad, 4, 2>;
pub type OcTree = BSTTree<Oc, 8, 3>;

...

// 例如BVH内的Bounding可以是任意类型
// 只要实现了以下约束，就可以直接生成以该类型为Bounding的BVH的Tree
//（当然我这里做的还不够彻底，理论上维数也是可以抽象出来的
pub trait BVHBounding:
  Sized + Copy 
  + FromIterator<Self> // 能够从一组自己中生成Union的Bounding
  + CenterAblePrimitive // 能够获得自身的中心位置
  + SolidEntity<f32, 3> // 自己是个三位封闭图形
{
  // 标记划分轴的类型
  type AxisType: Copy;
  // 获得最佳划分轴的方法
  fn get_partition_axis(&self) -> Self::AxisType;
}

// 你也可以实现这个Trait 直接自己定义BVH的划分策略！
pub trait BVHBuildStrategy<B: BVHBounding> {
  fn build(
    &mut self,
    option: &TreeBuildOption,
    build_source: &[BuildPrimitive<B>],
    index_source: &mut Vec<usize>,
    nodes: &mut Vec<FlattenBVHNode<B>>,
    depth: usize,
  ) -> usize {
    ...
  }

  fn split(
    &mut self,
    parent_node: &FlattenBVHNode<B>,
    build_source: &[BuildPrimitive<B>],
    index_source: &mut Vec<usize>,
  ) -> ((B, Range<usize>), B::AxisType, (B, Range<usize>));
}

```

在后续会继续完成开发BSP， R tree，KdTree等数据结构的实现。并尝试在这些结构之上抽象出更加一般的空间相关算法。

#### river-mesh

这部分目前完成度很低。这块主要想做一个可编辑mesh数据结构，类似于blender之类建模软件的几何部分。毕竟我能搞定渲染，我再搞定建模，我就可以自己进入一种无中生有的境界。。

主要是Half edge，后续会考虑做其他的数据结构，翼边什么的。

当时实现是想尝试使用unsafe代码提升性能，但是unsafe非常难写，非常容易写出UB。rust对于unsafe代码有一套非常晦涩的是不是UB的原则，叫stack borrow，整个paper我就没有真的读下去，给我的感受是实现上真的太难了。在当时，我是知道有miri这个解释器可以检查unsafe代码UB的问题，但是那时候我还没有决心上nightly(miri只有nigtly有)，后续就直接搁置了。当然现在RRF已经是nightly编译了，这块我想还是可以拾起来继续做的。另一个想法是，即便不适用unsafe，通过合适的编程技巧，依然能够作出高性能的实现，所以可能后续会直接重写。

这个仓库还躺了没写完的mesh QEM简模的实现，主要是想实践我另一篇介绍简模算法的文章，但是依赖于mesh数据结构，并未完成。

#### mesh-buffer

除了可编辑mesh，mesh的另一种，也可以说是最通用的表达方式是图元的buffer。即那些可以直接被图形API使用的数据。这个crate大量应用了泛型，定义了诸多这样的buffer容器，同时将图元的访问，遍历，抽象出具体的trait，并基于这些抽象，在这些容器上实现出了高性能的几何转化/射线求交/BVH等算法。除了基于容器的算法，还有一部分归类于tessellation的主题下，这部分是几何生成的工具。

具体的实现可以参考我之前另一篇文章的介绍。但是我要指出的是：很遗憾那一篇文章已经过时了，很多抽象和实现都做了调整，不过这不影响对我做的事情本质上的描述。

#### nozie

噪声及生成式纹理库，完成度也很低。目前只实现了两种最基本的噪声纹理，也并没有抽象上的设计。

最早我在实现Minecraft的地形部分的时候，需要噪声纹理，那么我很快我就找到了社区的包并成功使用上了，但生成式纹理一直也是我很感兴趣的主题，所以这不妨碍我建立一个独立的crate来进行开发和研究。

在将来，计划会有一个类似于mesh-buffer的crate，类比于mesh-buffer，提供贴图的容器/算法集合，而类比于tessellation的部分就是nozie。

### 跨平台渲染库：

#### ral 跨平台渲染抽象层

ral作为跨平台渲染抽象层，目的是解决跨平台渲染的问题，用户使用RAL而不是具体某个图形api可以实现跨平台能力，理论上通过重新编译，应用可以无缝迁移到其他平台进行渲染。

这个库主要包含了三个内容：

- 一套渲染相关描述符(例如贴图格式，drawmode格式)的中间格式。
- 一个RAL Trait，用来对渲染进行抽象
- 一个资源管理器的实现：ResourceManager<T: RAL>

中间格式现在直接将

RAL这个 trait，可以说是RRF目前最重要的trait了。

```rust
pub trait RAL: 'static + Sized {
  type RenderTarget;
  type RenderPass;
  type Renderer;
  type ShaderBuildSource;
  type Shading;
  type BindGroup;
  type IndexBuffer;
  type VertexBuffer;
  type UniformBuffer;
  type Texture;
  type Sampler;

  fn create_shading(renderer: &mut Self::Renderer, des: &Self::ShaderBuildSource) -> Self::Shading;
  fn dispose_shading(renderer: &mut Self::Renderer, shading: Self::Shading);
  fn apply_shading(pass: &mut Self::RenderPass, shading: &Self::Shading);
  fn apply_bindgroup(pass: &mut Self::RenderPass, index: usize, bindgroup: &Self::BindGroup);

  fn apply_vertex_buffer(pass: &mut Self::RenderPass, index: i32, vertex: &Self::VertexBuffer);
  fn apply_index_buffer(pass: &mut Self::RenderPass, index: &Self::IndexBuffer);

  fn create_uniform_buffer(renderer: &mut Self::Renderer, data: &[u8]) -> Self::UniformBuffer;
  fn dispose_uniform_buffer(renderer: &mut Self::Renderer, uniform: Self::UniformBuffer);
  fn update_uniform_buffer(
    renderer: &mut Self::Renderer,
    gpu: &mut Self::UniformBuffer,
    data: &[u8],
    range: Range<usize>,
  );

  fn create_index_buffer(renderer: &mut Self::Renderer, data: &[u8]) -> Self::IndexBuffer;
  fn dispose_index_buffer(renderer: &mut Self::Renderer, buffer: Self::IndexBuffer);

  fn create_vertex_buffer(
    renderer: &mut Self::Renderer,
    data: &[u8],
    layout: VertexBufferDescriptor<'static>,
  ) -> Self::VertexBuffer;
  fn dispose_vertex_buffer(renderer: &mut Self::Renderer, buffer: Self::VertexBuffer);

  fn set_viewport(pass: &mut Self::RenderPass, viewport: &Viewport);

  fn draw_indexed(pass: &mut Self::RenderPass, topology: PrimitiveTopology, range: Range<u32>);
  fn draw_none_indexed(pass: &mut Self::RenderPass, topology: PrimitiveTopology, range: Range<u32>);

  fn render_drawcall(
    drawcall: &Drawcall<Self>,
    pass: &mut Self::RenderPass,
    resources: &ResourceManager<Self>,
  );
}
```

和gfx-rs的定位差异：

在设计和实现rrf ral的过程中，我并没有参考gfx-rs的实现。后来随着思考和实现的推进，我发现ral有完全朝gfx-rs转型的苗头。gfx这个项目是rust图形生态中元老级的核心项目，其抽象程度，实现的完备程度，是不可能超越的。所以后来我尝试赋予ral新的一个定位，新的理解：ral只做场景资源类型的抽象，不涉及到底层实现渲染的数据结构的抽象，举例说，RAL的关联类型中不会有Encoder，Queue等类型，至于实际绘制中，这些场景资源如何进行渲染，如何进行同步的细节完全由RAL的实现者完成。这个新的定位目前还在转型之中，所以现在的RAL的接口还会有变化。

#### shadergraph 着色器链接框架

#### derives

#### webgl

渲染抽象层的webgl实现。主要是对webgl2的上层封装，使其实现RAL trait。实现的完整程度还是处于prototype的级别。

#### webgpu

渲染抽象层的webgpu实现。主要是对wgpu-rs的上层封装，使其实现RAL trait。实现的完整程度还是处于prototype的级别。

### 渲染框架层：

#### scene-graph

#### render-graph

#### nyxt

#### rendium

### 框架测试应用：

#### rainray

Yet another ray tracer. 

纯CPU实现的光线跟踪渲染器。使用rayon进行多线程加速。支持多种几何图元/mesh进行渲染，

#### rinecraft

Rendiation在渲染抽象层之上核心作出了的一套材质系统的框架性的实现。经过实践，相关的想法已经成形，虽然具体的实现依然在开发之中，但我自认为这套体系浓缩了我今年对材质框架新的独立思考，那么于此我做一些介绍和输出。

About the dirty works we knew

长期以来，我们编写渲染引擎的特效实现，遵从的模式是：写shader字符串，import/include 其他可用的shader的片段。然后将完整的shader拼接出来，配合一些其他信息交由图形API生成对应的pipeline。这种模式其实是一种四分五裂的状态，带来诸多不便：

* 构建于字符串拼接之上的抽象
* 跨平台困难
* 实现冗余

构建于字符串拼接之上的抽象

因为着色器语言静态强类型，其表达能力是强烈限制的，所以我们可以理解编程语言的割裂这个问题。

所谓构建于字符串拼接之上的抽象，举例说我们一般会上层封装出很多概念和抽象，比如光源，材质等，这些抽象的运作过程是通过在底层拼接shader字符串完成的。在我第一次学习图形学，看到这种字符串拼接的通用做法，是比较诧异的，我以为有更高级的做法，但事实上并没有。写web的会手工拼接html吗，过去的确是会，但现在我们不是有更好的做法吗？

我认为web是一个好的比喻，html比于glsl：浏览器提供了直接解析显示html字符串为页面的能力，浏览器也提供了将glsl变为一个可以执行图形绘制的shader的能力。
html是dom tree，浏览器提供了一整套API用以操作dom树节点，构建于dom API的那些前端框架再进一步为前端开发者提供了真正好用的抽象能力。但是我们看到glsl这边就完全没有了。我们畅想一下对应的是，**我们需要一套用以操作shader的控制流/数据流图节点的API，然后构建于这套流图的API之上，封装出材质框架，提供真正好用的抽象能力。**

html描述了一个tree的结构，所以重要的是tree，tree才是核心，html只是这个tree的一种输入方式而已，如果我高兴我可以用json同样来表述tree。 glsl描述了一个程序的执行逻辑，从程序的表达方式而言，显然这里match tree的就是流图了，tree毕竟也是graph的子集，graph才是核心，glsl只是这个这个graph的一种方式而已，我完全可以使用其他的着色器语言，来对应到这一个graph。

我们不断通过调整拼接字符串的逻辑来进行dom操作吗，显然是不可能的。但是我们在shader的处理上做的事情恰恰如此。低效，易错，dirty。而构建于字符串之上的抽象是如此的不足，直接造成了下面的核心问题：

跨平台

跨平台指的是跨图形API的问题，我们有如此多的target要支持，每个target又有不同的版本，每个版本都会给shader添加一些新的能力，甚至每个target又有一堆extension，每个extension又给shader添加了一些新的能力。这就变成了一个工程上的难题。

理智的说，我大概需要知道这么两件事情，1 针对某一个shader module，或者是逻辑上的着色单元。对于每个target，对于每个version，能不能支持的了，2 如果不能，有没有extension可以支持，如果没有extension，那么有没有可能做一些性能上的牺牲，框架层面自动生成polyfill，有没有可能提供一种工程上漂亮的优雅降级的能力？

我想如果框架是运行在字符串拼接上，那么恐怕1都搞不定，因为我们都给不出一个严谨的shader的feature set的描述。有了这个描述，我们才能和各个target/version去match 他们的feature set。我们不能指望用户给标记这些东西。

第二个目标就更加困难了，因为这本质上要让我们对shader的程序进行变换和改写。讲道理我shader能靠字符串操作自动展开for循环已经不容易了，基于字符串，真的不能做更多事情。

**字符串是没有结构的东西**，没有结构是问题的核心。材质系统是编译器技术和图形学技术融合的地方，做的好真的比较难，是逃不掉写compiler的，只有搞定编译器前端，你才能真正的获得完整的结构化的着色逻辑，才能真正开始搞事情。

设计一门新的着色语言，然后compile到其他target，我现在认为不是一个好的做法，或者说不是问题的核心。我认为核心是要作出着色逻辑的结构化描述出来，就是IR，针对这个IR，去为你要支持的target建模feature set，feature set的识别和降级算法。然后为你想支持的语言写前端，来compile到IR上。你可以同时支持glsl和hlsl，一个着色组件用glsl，另一个用hlsl，大家share 着色组件的实现，做成库，再也不care shader是什么语言的问题。当然你也可以设计自己的语言。 另外我推崇直接将框架语言的子集作为前端，这样，同一份语言可以同时跑在CPU和GPU上，完全没有跨语言的问题。

冗余

pipeline layout descriptor类型的信息和shader内的信息是冗余的



其他能力

为什么使用流图，而不是AST？

## 


