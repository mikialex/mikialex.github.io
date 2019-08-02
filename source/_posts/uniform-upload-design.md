---
title: Uniform upload optimization and UBO support in Artgl
date: 2019/8/3
---

本文主要讨论和介绍一下artgl的uniform上传优化，到如何支持ubo的设计。 以及，因为ubo的优化，shadergraph的api和其概念模型的调整。

## Uniform Upload

一个经典的图形框架设计问题是:如何减少 uniform 上传?

在最简单的实现中，⽤户在绘制某个物体前，需要对当前uniform进⾏更新。这些通过调用gl.uniformXXX 等接⼝来完成。当⼀个场景drawcall ⾮常多，draw的东西也⽐较复杂，需要调⽤这个接⼝进行上传的次数也⾮常多，这里会造成cpu端的性能瓶颈。 即便当前gl状态机中的uniform值已经是正确的，并不需要调⽤这个接⼝进⾏更新，但如果不加判断的调⽤，经过实测，依然会造成额外成本。原因可能是浏览器的实现或者驱动的实现，需要对js端传来的数据做额外的检查和同步，但出于某些原因，并没有cache机制。所以，WebGL框架应当提供相应的cache，在上传之前，进⾏diff，如果判断出当前并不需要上传，那么直接略过。引⼊diff机制会带来额外性能成本，⼀般⽽言，这个成本会小于直接调⽤接⼝的成本，但不排除会有⼀些情况不能产⽣优化效果。

具体的diff的机制是这样的: 如果当前要draw的东⻄，没有切换过program， 且cache的新的数值，和cache的旧数值一致，则不进⾏上传。如果不一致，则copy更新cache旧的数值，进⾏上传。当切换了program，⽆论如何都要进⾏上传。

如此即可做到：不会上传不需要更新的uniform。但上⾯提到，uniform diff和copy是有成本的，can we do better? 我们能想办法避免⼀部分uniform diff的成本吗? 想想看似乎是可能。⽐如:在⼀批的绘制过程 中，camera的VP matrix 从来都没有变过，我们是知道的，所以可以不⽤diff。⼜⽐如:在⼀批的绘制过程中，light的参数假设也是没有变过的，那么我们也可以提前跳过。这些情况都可以被识别出来，然后在renderer⾥特殊处理掉。

但是，我认为这样做是糟糕的，即便three.js就是这样做的。原因是: 

1 这样做破坏了引擎设计的⼀致性， 只有被你认可的case才能得到优化。 

2 为了实现这样的优化，渲染逻辑和场景对象强耦合在⼀起， renderer⾥存在⼤量的针对处理这种特殊情况的代码。

3 为了实现这样的优化，在你使⽤场景的api时，你又必须非常了解这些假设，如果不了解，要么优化不会起作用，要么你期望的API调⽤却竟然没有效果。

所以， how can we do better than that? 我们应该找到⼀个通⽤的解法。如果从本质上来看这个问题，⼀般⽽⾔，判断数据变化就两种⽅案: 

1 diff，这种⽅案，⽤户在修改和访问数据时，不会有成本，但是在获取数据是否变更时需要付出成本。

2 watch，这种⽅案，⽤户在修改和访问数据时，会有成本，但是在获取数据变更时没有成本。

上⽂提到的three的这种特殊的做法，其实还是diff，只不过通过约定来省略⼀部分diff。

我的解决⽅案是通过结合前⼀种做法，尽可能的减少diff的发⽣。

⼀处优化是: 假设对于⼀个vector，或者matrix，我们对这个值添加⼀个容器进行包装，实现watch机制，知道了它有没有被change，我们就可以通过diff boolean值，其实就是判断有没有change，来节约逐个数据的diff成本。

renderer对于⼀个draw的若⼲uniform，会先拿到需要set对应uniform的容器，还会从⼀个cache的Set⾥里里 拿到上⼀个对应该uniform且set过值的容器，如果两个容器本身不是⼀个引⽤，这时候我就知道，这个uniform数值的提供者发⽣了改变，那这种情况下，我需要做diff，容器上⾯的可变标记就⽆效了。如果两个容器是同⼀个引用，那么如果容器上的change标记显示已经发⽣了变化，那么我就可以不diff，直接进行上传，如果标记显示没有发⽣变化，那么我也可以跳过diff，直接不上传。⼀旦发⽣了上传，则更新 change标记false。如果program发⽣生了切换，那我就清空上⼀个draw的容器Set缓存，如果拿不到上⼀个draw的对应uniform的cache容器，那我也可以跳过diff，直接上传。

这个解法可以缓解⼤部分不需要的diff，⽽而对于watch的成本，在图形开发的常⽤环境下⽽而⾔言，不变的东⻄
往往⼤⼤多于变的东⻄，所以是⾮常合算的!

## Uniform Block & ShaderGraph decorator API

WebGL2⽀持了UBO，UBO可以让⽤户将多个uniform合并成⼀个进⾏绑定，减少了gl调⽤次数，同时也能作为整体在多个WebGL program之间进⾏复⽤。对性能的提升影响很⼤。 我们怎么⽤好这个api呢，can we do better?

从字⾯意义上讲，既然要UBO，我们需要把⼀个shader中几个相关联的的uniform看成⼀个整体，或者知道他们是有联系的。把它们group起来。如果你⽐较贪，想把它们全部group起来，这样是不合适的，这样没有办法进⾏组合，只能draw⼀个死的东⻄。所以我的想法是把它们之前有关系的东⻄进⾏group:⽐ 如light的位置，颜色，就是⼀个整体，⼀个block。很显然的是，这些需要group的uniform的提供者，似乎都是⼀个对象，所以这个设计就⽐较明确了:

在artgl的设计中:每个这种可以提供uniform的对象，需要实现uniform provider的接⼝，其中包含⼀个 uniform-key to uniform-watch容器的map， 和⼀个标示uniform block是否change的字段。对象⾃身的⼀些属性，可以被装饰器进⾏装饰，使得⾃动转化成getter和setter， setter会⾃动向uniform-watch容器进⾏更新，以及和block change字段进⾏更新。由此，⽤户可以⾮常⾮常⽅便的接⼊uniform的优化系统。只要对原先的字段添加装饰器并继承⼀个实现了这个接⼝的类就可!

<img src="/images/uniform-upload-design/provider.png" style="max-width:500px">

在之前的⼀篇⽂章，介绍了artgl的shadergraph的设计，在artgl中，shader是通过graph api显示的构建出来的，shader的uniform依赖，其实就是graph上的uniform input node，所以 can we do better on api design?

对于那些作为uniform provider的对象，它们之所以能作为uniform的 provider，是因为shader中有需要它们提供的uniform，所以，它们其实是和shader息息相关的。在此，介绍⼀下artgl shadergraph decorator的API，它很好的解决的这⼀层的抽象。本质上uniform provider声明了某个shader需要这些 uniform，所以，provider需要亲⾃修改shadergraph，⾃⼰添加⾃⼰provide的uniform input和⾃身 shader实现的计算逻辑。API上，uniform provider需要实现 decorate接⼝，decorate接⼝输⼊需要装饰 的shadergraph，实现装饰逻辑。

<img src="/images/uniform-upload-design/decorate.png" style="max-width:500px">

在graph上添加⼀个uniform的input node， 需要名字和类型，默认值等参数，这些显然和provider⾃身的字段声明是冗余的，所以我在provider的基类实现了⼀个util函数，可以直接运⾏时读取属性值，根据数据类型和数据，return你需要的uniform input node，同时，利⽤ts的key of，可以实现静态的字段检查，防止你引⽤⼀个不存在字段。

比如这个简单的点光源的实现：

```ts

export class PointLight extends Light<PointLight> {

  decorate(decorated: ShaderGraph) {
    decorated
      .setFragmentRoot(
        AddCompose.make()
          .input("base", decorated.getFragRoot())
          .input("light", pointLightShading.make()
            .input("fragPosition", decorated.getVary(WorldPositionFragVary))
            .input("FragNormal", decorated.getVary(NormalFragVary))
            .input("lightPosition", this.getPropertyUniform('position'))
            .input("color", this.getPropertyUniform('color'))
            .input("radius", this.getPropertyUniform('radius'))
          )
      )
  }

  @MapUniform("lightColor")
  color: Vector3 = new Vector3(1, 1, 1)

  @MapUniform("lightPosition")
  position: Vector3 = new Vector3(0, 0, 0)

  @MapUniform("lightRadius")
  radius: number = 3
}

```

其实不止光照系统，camera， 渲染曝光参数，等等，如果你想要引擎内置的这些功能，那就让他们装饰你的shadergraph吧，他们依赖的uniform，ubo，shader的逻辑，直接全都有了～ 所以shadergraph的api现在将全部以decorator的形式提供：

```ts
export class PureShading extends BaseEffectShading<PureShading> {

  decorate(graph: ShaderGraph) {
    graph
      .setVertexRoot(MVPWorld())
      .setFragmentRoot(uniform("baseColor", GLDataType.floatVec4))
  }

}
```

在api的另⼀侧，场景接⼝的⽤户，现在构造某个绘制所需要的shading对象，则需要⼿⼯的进⾏依赖的装饰，虽然使⽤上繁琐了⼀些。但是本质上其实就是把更多的⾃由度开给了⽤户，让⽤户更清楚的看到这个draw依赖哪些对象，如何能更好的在数据层⾯上优化绘制。在过去，这样的逻辑是写死在渲染引擎内部的，对于使⽤者的理理解其实也是⼀层不便。

通过 uniform provider的 API， ⼀⽅⾯，对UBO的引擎层⾯有了很好的抽象，直接⽀持是件易如反掌的事情。另⼀⽅⾯，进⼀步减少了上传时check的成本，毕竟你有了⼀个group的change标记可以略略过⼀整个group的check过程。更重要的是和shadergraph的结合使得在设计上，开发上，概念模型上达成了不错的⼀致和统⼀。