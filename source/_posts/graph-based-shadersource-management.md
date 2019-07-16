---
title: Graph based shader source management in artgl
date: 2019/7/16
---

目前绝大多数的webgl库都使用字符串替换，并使用预编译指令来控制shader行为。

这种一般的基于字符串替换的方案存在以下缺陷：

* 没有可组合性，只是复用代码，抽象能力弱
* 代码包含大量不需要的实现，使用 #include #define 等做区分，调试和阅读困难， 开发效率低
* 问题定位只依赖最后编译报错，排错效率低
* 修改代码需要在浏览器运行所有依赖的shader，代码难维护

这些缺陷导致的各自难受之处之前也深有体会, 所以在设计artgl的shader体系时，我一直想要解决这个问题：

**如何找到更好的方案来管理一个webgl框架内的shader代码？**

[glslify](https://github.com/glslify/glslify) 可能是一个好的解决方案，但我个人并不是很喜欢这种方式： 在shader里写require, transform shader code。 这种方式是以shader code作为核心，而我希望以框架code为核心，这样做1是ts代码能做更多的事情，而shader只是字符串，即便使用了完整的shader parser，能做的事情还是比较有限，扩展起来也不方便。 最好是使用ts代码，框架的代码，对着色流程进行抽象，shader只是这种抽象的target。2是，glslify太重了，我还是倾向于暂时把事情看的简单一些。

正常我们写shader，大多数写的是一些function，然后组合起来，执行出结果。所以我设计的组合单元是、ShaderFunction： 比如下面这个shader function，pack float depth到 四通道 rgba中

 ```ts
export const depthPackFunction = new ShaderFunction({
  description: 'pack depth to RGBA output',
  source: `
    vec4 depthPack(float frag_depth){
      vec4 bitSh = vec4(256.0 * 256.0 * 256.0, 256.0 * 256.0, 256.0, 1.0);
      vec4 bitMsk = vec4(0.0, 1.0 / 256.0, 1.0 / 256.0, 1.0 / 256.0);
      vec4 enc = fract(frag_depth * bitSh);
      enc -= enc.xxyz * bitMsk;
      return enc;
    }
  `,
})
 ```

ShaderFunction 只是描述这个shader的function，一个shader可能会调用多次shaderFunction进行计算，所以某个使用ShaderFunction的计算我们用 ShaderFunctionNode 来表示：ShaderFunction 事实上是 ShaderFunctionNode 的 factory。！

```ts
const packedDepth = depthPackFunction.make()
```

function的调用总归要传参吧：参数从哪里来呢？ 要么是来自shader外部，比如uniform，attribute，texture，对于片元着色器还有vary，要么来自其他function调用的结果。

我们先看外部的参数可以如何如何构造：

```ts

// 这是一个引擎内置支持的全局uniform
const VPMatrix = innerUniform(InnerSupportUniform.VPMatrix)

// 这时一个用户自定义的shader uniform
const sampleCount = uniform("u_sampleCount", GLDataType.float).default(0);

// 同理一个position attribute
const position = attribute("position", GLDataType.floatVec3);

// 同理一个depth vary
const depth = vary("depth", GLDataType.float);
```

然后我们可以给 packedDepth 喂数据了

```ts
packedDepth = depthPackFunction.make().input(
  vary("depth", GLDataType.float)
)
```

当然，链式调用，我们可以一个一个input， 栗子：

```ts
const worldPosition = MVPTransform.make()
  .input("VPMatrix", innerUniform(InnerSupportUniform.VPMatrix))
  .input("MMatrix", innerUniform(InnerSupportUniform.MMatrix))
  .input("position", attribute("position", GLDataType.floatVec3))
```

如果 ShaderFunctionNode 依赖其他 ShaderFunctionNode 的计算结果，这也是可以的：

```ts
// 比如 先生成一个动态的2d平面上的随机数
const Random2D1 = rand2DT.make()
  .input("cood", vUV)
  .input("t", sampleCount)

// 再拿它去搞出另一个维度的随机数
const Random2D2 = rand.make()
.input("n", Random2D1)

// 然后在拿这两个随机数生成随机方向
const randDir = dir3D.make()
  .input("x", Random2D1)
  .input("y", Random2D2)
```

通过这些node的连接，我们构造了一个graph，现在有了这个graph，我们要把shader搞出来。

OK， 我们再回过头来看shader，我们需要的是shader，webgl shader，两个着色器：

对于顶点着色器，总归有个 gl_Position ，或者 vary 这些。很简单，我们只要指定说：哪个node作为gl_Position，或者哪个node最为 叫什么名字的 vary就可以了。

对于顶点着色器，总归有个 gl_FragColor ，或者 mrt的输出 这些。很简单，我们只要指定说：哪个node作为 gl_FragColor ，mrt就先从简不考虑了，虽然挺对称的。

因为其实我们 vary 是构建顶点着色器指定的，所以前面vary也应该从整个graph get，api要改一下。

这是个画normal颜色的shader，应该好理解：

```ts
const graph = new ShaderGraph();

graph.reset()
  .setVertexRoot(
    MVPTransform.make()
      .input("VPMatrix", innerUniform(InnerSupportUniform.VPMatrix))
      .input("MMatrix", innerUniform(InnerSupportUniform.MMatrix))
      .input("position", attribute(
        { name: 'position', type: GLDataType.floatVec3, usage: AttributeUsage.position }
      ))
  )
  .setVary("color", attribute(
    { name: 'normal', type: GLDataType.floatVec3, usage: AttributeUsage.normal }
  ))
  .setFragmentRoot(
    normalShading.make().input("color", this.graph.getVary("color"))
  )
```

为了开发方便，很多小功能也被加入进来，比如你可以这么写：而不用自己包装function

```ts
const node1 = vec4(attribute("position", GLDataType.floatVec3), constValue(1))
const node2 = node1.swizzling("zyx")
```


这就是 artgl ShaderGraph API的核心接口啦，目前这么写，已经可以work了！

在通过这些接口构建graph的过程中，每一步都有运行时参数类型的检查。link错的地方很早就能发现，解决了： 「问题定位只依赖最后编译报错，排错效率低」 的问题

虽然不是静态的检查，至少可以保证只要function写的不错，能link就能运行。所以你可以直接把每个shader构建加在单测里，单测没挂shader就没挂，不用起浏览器环境。解决了： 「修改代码需要在浏览器运行所有依赖的shader，代码难维护」 的问题

在编译shader的过程中，linker只会收集使用到的function和input依赖，没有任何预处理指令干扰视线，所以不存在： 「代码包含大量不需要的实现，使用 #include #define 等做区分，调试和阅读困难， 开发效率低」 的问题

shader graph 每一个node其实就是一个有类型的数值节点，所以可组合性非常强。不同的node类型实现不同的功能，易于扩展，比字符串拼接水平高多了。graph更是提供了整个shader的抽象，解决了「没有可组合性，只是复用代码，抽象能力弱」的问题

框架内置提供了常用的function供用户使用，也可以从独立的package import，shader 代码也正式成为框架的可扩展实现之一。 基于这样的抽象，甚至能解决不同shader version 编译的问题，这个就不展开了//

基于动态link，使用这种api，构造一个shader，类似于使用react，构造一个view，其实是相似的，有很多想象空间。基本上是我认为是实现框架抽象shader代码的一个好的方向。当然目前的解法还有一些限制，一些问题还有待解决，一些特性还有待设计和支持，如果有反馈和建议欢迎在 [https://github.com/mikialex/artgl](https://github.com/mikialex/artgl) 反馈和交流。


```ts
export class TSSAOShading extends Shading {
  update() {
    const VPMatrix = innerUniform(InnerSupportUniform.VPMatrix);
    const sampleCount = uniform("u_sampleCount", GLDataType.float).default(0);
    const depthTex = texture("depthResult");
    this.graph.reset()
      .setVertexRoot(
        vec4(attribute(
        { name: 'position', type: GLDataType.floatVec3, usage: AttributeUsage.position }
      ), constValue(1)))
      .setVary("v_uv", attribute(
        { name: 'uv', type: GLDataType.floatVec2, usage: AttributeUsage.uv }
      ))

    const vUV = this.graph.getVary("v_uv");
    const depth = unPackDepth.make().input("enc", depthTex.fetch(vUV))

    const worldPosition = getWorldPosition.make()
      .input("uv", vUV)
      .input("depth", depth)
      .input("VPMatrix", VPMatrix)
      .input("VPMatrixInverse", uniform("VPMatrixInverse", GLDataType.Mat4).default(new Matrix4()))

    const Random2D1 = rand2DT.make()
      .input("cood", vUV)
      .input("t", sampleCount)
    
    const Random2D2 = rand.make()
    .input("n", Random2D1)
    
    const randDir = dir3D.make()
      .input("x", Random2D1)
      .input("y", Random2D2)

    const newPositionRand = newSamplePosition.make()
      .input("positionOld", worldPosition.swizzling("xyz"))
      .input("distance", uniform("u_aoRadius", GLDataType.float).default(1))
      .input("dir", randDir)

    const newDepth = unPackDepth.make()
      .input("enc",
        depthTex.fetch(
          NDCxyToUV.make()
            .input(
              "ndc", NDCFromWorldPositionAndVPMatrix.make()
                .input(
                  "position", newPositionRand
                ).input(
                  "matrix", VPMatrix
                )
            )
        )
      )

    this.graph.setFragmentRoot(
      tssaoMix.make()
        .input("oldColor", texture("AOAcc").fetch(vUV).swizzling("xyz"))
        .input("newColor",
          sampleAO.make()
            .input("depth", depth)
            .input("newDepth", newDepth)
        )
        .input("sampleCount", sampleCount)
    )
  }
}
```