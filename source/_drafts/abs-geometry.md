# 使用Rust trait 抽象三维几何数据类型

在对rust的trait， generics等概念有了一定理解后，我尝试集中编写一些三位几何数据的容器类作为渲染引擎的基础库。

如果你对three.js比较了解的话，我这里做的，其实就是使用rust编写Geometry，BufferGeometry等容器类，同时支持一些简单的操作，比如有index的geometry 转化为无index的geometry，以及支持raycast。

three的geometry写的是很挫的，寄希望于rust这门优秀的语言，我试图提升一下自我要求，比如：

* 尝试将拓扑信息编码在类型之中，这样假设我有一个描述以triangleList为布局的几何数据，不应该赋值给一个以trianglestrip为布局的几何数据。
* 可以使用任何用户自定义的顶点数据结构
* 提供一个统一的图元迭代器，图元的类型由拓扑类型决定
* 同时支持index/无index， interleaved/非interleaved的buffer数据

## 统一interleaved和非interleaved的数据

我们正常在opengl/webgl中使用的数据都是非interleaved的，比如一般来说position是一个单独的buffer，normal也是一个单独的buffer，uv也是。而interleaved版本是position + normal + uv交替存储，只使用一个buffer。

虽然我一般都是使用非interleave版本，但事实上interleaved才是较为原始的数据存储，假设你如下定义一个struct，标记内存布局使用C语言标准。对于一个`Vec<Vertex>`, 它的内存事实上完全就是interleaving的，你可以直接transmute一下上传，没有多余的拷贝。

```rust
#[repr(C)]
#[derive(Clone, Copy, soa_derive::StructOfArray)]
pub struct Vertex {
  pub position: Vec3<f32>,
  pub normal: Vec3<f32>,
  pub uv: Vec2<f32>,
}

let my_geometry_data: Vec<Vertex>;
```

而我们常用的非interleave版本本质上是这个：

```rust
pub struct VertexArray {
  pub position: Vec<Vec3<f32>>,
  pub normal: Vec3<f32>,
  pub uv: Vec2<f32>,
}

let my_geometry_data: VertexArray;
```

本质上就是struct of array 和 array of struct 的区别。你也看到，我在Vertex上标记`soa_derive::StructOfArray`, 这个库可以为我们自动生成 struct of array版本的类型，并实现`As<[T]> + Index<usize>`这两个trait, 这使得两种容器类型在使用上一模一样。

所以和这个抹平差异的思想一致。实际数据的存储上，容器要满足的trait是：

```rust
pub trait GeometryDataContainer<T>:
  AsRef<[T]> + Clone + Index<usize, Output = T> + FromIterator<T>
{
}

// 为普通vec实现
impl<T: Clone> GeometryDataContainer<T> for Vec<T> {}
// 为自动的soa derive实现
impl<T: StructOfArray, A: T::ArrayType> GeometryDataContainer<T> for A {}
```

## 顶点，拓扑，和图元的trait约束

一个几何由很多顶点构成，先不考虑有没有index，这些顶点每一个/两个/三个 形成一个三角形/线段/点，如果再考虑是list还是strip，又可分为每过一个图元步进几个顶点数据。这些如何使用trait来抽象呢？

先看顶点，这里我们把问题再放窄一点，要求顶点必须是能提供3d空间点的信息， 3d空间点也不再加一层泛型，就f32吧，所以所有顶点应该实现这个trait：

```rust
pub trait Positioned3D: Copy {
  fn position(&self) -> Vec3<f32>;
}
```

顶点构成图元， 图元的约束可以是这样：

```rust
pub trait PrimitiveData<T: Positioned3D> {
  type IndexIndicator;
  const DATA_STRIDE: usize;
  fn from_indexed_data(index: &[u16], data: &[T], offset: usize) -> Self;
  fn create_index_indicator(index: &[u16], offset: usize) -> Self::IndexIndicator;
  fn from_data(data: &[T], offset: usize) -> Self;
}
```

来看几个impl可能会更清楚：

```rust
impl<T: Positioned3D> PrimitiveData<T> for Triangle<T> {
  type IndexIndicator = Triangle<u16>;
  const DATA_STRIDE: usize = 3;
  fn from_indexed_data(index: &[u16], data: &[T], offset: usize) -> Self {
    let a = data[index[offset] as usize];
    let b = data[index[offset + 1] as usize];
    let c = data[index[offset + 2] as usize];
    Triangle { a, b, c }
  }

  fn create_index_indicator(index: &[u16], offset: usize) -> Self::IndexIndicator {
    let a = index[offset];
    let b = index[offset + 1];
    let c = index[offset + 2];
    Triangle { a, b, c }
  }

  fn from_data(data: &[T], offset: usize) -> Self {
    let a = data[offset];
    let b = data[offset + 1];
    let c = data[offset + 2];
    Triangle { a, b, c }
  }
}

impl<T: Positioned3D> PrimitiveData<T> for LineSegment<T> {
  type IndexIndicator = LineSegment<u16>;
  const DATA_STRIDE: usize = 2;
  ...
}
```

`DATA_STRIDE`就是上文提到的数据宽度，三个方法就是具体从一个slice里读取并构造出图元的实现，IndexIndicator辅助支持indexgeometry。这个trait配合上面的容器抽象，我们就可以在容器的指定位置读取图元。

基于顶点和图元，我们还需要描述拓扑的信息，显然，拓扑可以使用零尺寸类型表达

```rust
pub trait PrimitiveTopology<T: Positioned3D> {
  type Primitive: PrimitiveData<T>;
  const STRIDE: usize;
}

// 这里不多举例，可以看到拓扑这个trait同时link了对顶点的约束，以及对应的图元类型，和步进长度
pub struct TriangleList;
impl<T: Positioned3D> PrimitiveTopology<T> for TriangleList {
  type Primitive = Triangle<T>;
  const STRIDE: usize = 3;
}

pub struct TriangleStrip;
impl<T: Positioned3D> PrimitiveTopology<T> for TriangleStrip {
  type Primitive = Triangle<T>;
  const STRIDE: usize = 1;
}
```

## 构造我们的几何类型

数据结构和约束如下：

```rust
pub struct IndexedGeometry<
  V: Positioned3D = Vertex,
  T: PrimitiveTopology<V> = TriangleList,
  U: GeometryDataContainer<V> = Vec<V>,
> {
  pub data: U,
  pub index: Vec<u16>,
  _v_phantom: PhantomData<V>,
  _phantom: PhantomData<T>,
}

pub struct NoneIndexedGeometry<
  V: Positioned3D = Vertex,
  T: PrimitiveTopology<V> = TriangleList,
  U: GeometryDataContainer<V> = Vec<V>,
> {
  pub data: U,
  _v_phantom: PhantomData<V>,
  _phantom: PhantomData<T>,
}
```

基于这两种geometry，配合上面的一些内容，可以直接实现出图元的迭代器，包括ExactSizeIterator。很简单，这里就不放出实现了。

可以看到我们有两个geometry，有index的和无index的，这应该需要统一一下。他们应该要实现同一个trait，这个trait应该是几何数据最基础的trait。 从抽象的角度来看，我们只关心图元，应该是要能返回图元的ExactSizeIterator，以及图元的随机访问能力。

```rust
pub trait AbstractGeometry: Sized {
  type Vertex: Positioned3D;
  type Topology: PrimitiveTopology<Self::Vertex>;

  fn wrap<'a>(&'a self) -> AbstractGeometryRef<'a, Self> {
    AbstractGeometryRef { wrapped: self }
  }

  fn primitive_iter<'a>(&'a self) -> AbstractPrimitiveIter<'a, Self> {
    AbstractPrimitiveIter(self)
  }

  fn primitive_at(
    &self,
    primitive_index: usize,
  ) -> Option<<Self::Topology as PrimitiveTopology<Self::Vertex>>::Primitive>;
}
```

因为返回的迭代器势必要对源数据保持一个借用关系，如果我要把这个迭代器的类型写进AbstractGeometry的关联类型的话，会有个生命周期参数要填。为了解决这个问题我参考了一些做法做了一些尝试，最后做法是再间接return中间的迭代器结构，让这个中间的迭代器实现IntoIterator，而这个实现我们在为AbstractGeometry实现一些通用方法的时候加上这个trait约束即可，其中生命周期参数通过HRTB注入。比如下面这个为**所有几何类型实现获得光线相交点的列表**的实现：

```rust
impl<'a, V, P, T, G> IntersectAble<AbstractGeometryRef<'a, G>, IntersectionList3D, Config> for Ray3
where
  V: Positioned3D,
  P: IntersectAble<Ray3, NearestPoint3D, Config> + PrimitiveData<V>,
  T: PrimitiveTopology<V, Primitive = P>,
  G: AbstractGeometry<Vertex = V, Topology = T>,
  for<'b> AbstractPrimitiveIter<'b, G>: IntoIterator<Item = T::Primitive>,
{
  fn intersect(&self, geometry: &AbstractGeometryRef<'a, G>, conf: &Config) -> IntersectionList3D {
    IntersectionList3D(
      geometry
        .primitive_iter()
        .into_iter()
        .filter_map(|p| p.intersect(self, conf).0)
        .collect(),
    )
  }
}
```

这个实现等价于threejs分散在各个class内的raycast方法，在我们的抽象体系下只有短短这么一点。从类型上可以看到，除了为了支持AbstractGeometry的trait约束，我只要要求图元实现了和光线相交的trait，那么任意满足这组约束的几何就可以直接支持这个行为。而由于完全是静态的trait的分发，编译时rust会根据你实际使用到的所有可能的图元/顶点/拓扑/几何/类型的组合生成代码，然后再层层内联高度优化，而这些代码原先都是手写的，科学技术真是第一生产力。

通用渲染用的mesh几何数据的实现还在开发中，实际实现可参考：
<https://github.com/mikialex/rendiation/tree/master/mesh-buffer/src/geometry>
