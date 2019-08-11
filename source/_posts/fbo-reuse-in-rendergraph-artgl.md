---
title: FBO reuse in artgl rendergraph
date: 2019/8/11
---

这周末做了基于RenderGraph的Framebuffer重用。借此机会撰写一些关于reuse fbo的设计。

当我们在写后处理的时候，我们会写很多pass， 有的pass单纯的使用场景的shader画场景，更多的使用特殊的shader覆盖场景的shader进行绘制，有的pass处理之前pass绘制的结果，还有的pass处理多个之前pass的结果。你构建整个后处理的工作，事实上就是给这些pass分配绘制的target，并把这些target再提供给后续依赖的pass，来组合这些pass得到最终的屏幕效果。

很显然，整个后处理流程可以使用有向无环图进行抽象， RenderGraph就是我在artgl里完成的实现。

记得当时在我从前端转职到图形的时候，我最大的感慨是，没有一个顺手的webgl框架可用，对于之前vue什么写惯了的前端来说，组件化的模块化的抽象是深入人心的。市面上很难找到一个webgl框架，提供合理的针对渲染这个行为的模块化抽象。这大概是最早的一些想法。

后来，公司的项目里开始有一些后处理要做，这些后处理流程越来越多，充斥着大量非常丑陋的实现。于是我看不下去了，设计了一个graph的系统，通过声明式的api模块化后处理管线，希望能重构公司项目里整个后处理流程，我保证说整个后处理管线代码量下降一半，开发效率翻倍，但是我并没有得到机会做这个事情。于是乎我只能在artgl里做了。

为什么这周末突然做了fbo重用的实现呢？因为上周同事反馈移动端因为fbo大量申请导致崩溃的问题，这显然原因就是fbo没有复用的后果。我不敢想象在公司的项目里，没有graph的抽象，去人肉分辨出用到的十个fbo，谁在什么时候可以释放，谁可以给谁用，和动态配置如何配合，这种代码实在是不敢写，也没法写。 

而这时我想到正因为artgl做了graph的后处理，整个graph，这个pass依赖哪个pass的结果，那个pass的结果在哪一步再也不需要保留内容了，这些优化信息全部可以自动收集，所以基于graph做fbo的自动reuse是很简单的事情。

如果所有fbo都是一个格式和大小，那么大概是这个效果：

<img src="/images/fbo-reuse-in-rendergraph-artgl/fbo-reuse.png" style="width: 100%; max-height: 600px">

具体的做法是三部分：

1 每个 renderTargetNode 不再有一一对应的fbo，这些fbo也不在图构建的时候直接申请，取而代之，向一个fbo pool请求，fbo pool会根据 renderTargetNode 上的格式信息，比如长宽等，生成formatKey字符串，pool持有一个formatKey -> fbo[] 的map，取一个就pop一个，如果没有符合格式的直接向gl申请，这是取的过程。还有一个还回来的接口表示这个fbo我用完了，上面的内容后续没有依赖了，可以继续重用， push到对应的map里的array里。

2 每个renderPass不再向对应的output target和dependency的nodes索取fbo，而是 rendergraph的执行器，effectComposer， 在执行pass前完成相应的工作。 composer自己会持有一份targetName到fbo的map， 表示当前哪些fbo的内容需要keep。composer会在renderPass执行前，获取所需的fbo，如果这个是之前keep住的，那么就取keep的fbo，如果没有就问pool申请，并在pass结束后，按需归还不再使用的fbo。

3 composer需要知道这个pass结束后，哪些kept的fbo可以return给pool，那么在构建图的时候，拓扑排序完，得到的renderPass array，需要计算一个droplist array， 就是走到哪一个pass，哪些之前用的fbo再也不需要保留内容了。根据这个list，composer便可以实现按需归还fbo。

在某些情况下，某个pass的结果即便后续没有任何引用，依然需要一直keep住。比如temporal的ping pong buffer， 需要让这个target隔帧的进行reuse。 又比如 如果你需要某个target在render之外能支持readPixel， 那么你也需要一直keep。为了支持这种情况， renderGraph api在声明target节点时，提供了 keepContent的 getter， 默认永远返回false

```ts
export interface RenderTargetDefine {
  name: string,
  from: () => Nullable<string>
  keepContent?: () => boolean
  format?: {
    pixelFormat?: PixelFormat,
    dimensionType?: DimensionType,
    width?: number,
    height?: number,
    enableDepthBuffer?: boolean
  }
}
```

这是目前的带两个temporal的demo，你可以看到跨帧交替释放是如何支持的，我现在已经可以放心的继续往后叠ao的blur和glow的效果，不用担心要多出四个屏幕大小的fbo了。

```ts
graph.defineGraph({
    renderTargets: [
    {
        name: RenderGraph.screenRoot,
        from: () => 'CopyToScreen',
    },
    {
        name: 'sceneResult',
        format: {
        enableDepthBuffer: true,
        },
        from: () => 'SceneOrigin',
    },
    {
        name: 'depthResult',
        format: {
        enableDepthBuffer: true,
        },
        from: () => 'Depth',
    },
    {
        name: 'TAAHistoryA',
        keepContent: () => !this.isEvenTick,
        from: () => this.isEvenTick ? null : 'TAA',
    },
    {
        name: 'TAAHistoryB',
        keepContent: () => this.isEvenTick,
        from: () => this.isEvenTick ? 'TAA' : null,
    },
    {
        name: 'TSSAOHistoryA',
        keepContent: () => !this.isEvenTick,
        from: () => this.isEvenTick ? null : 'TSSAO',
    },
    {
        name: 'TSSAOHistoryB',
        keepContent: () => this.isEvenTick,
        from: () => this.isEvenTick ? 'TSSAO' : null,
    },
    ],
    passes: [
    { // general scene origin
        name: "SceneOrigin",
        source: [scene],
    },
    { // depth
        name: "Depth",
        shading: this.depthShader,
        source: [scene],
    },
    { // mix new render and old samples
        name: "TAA",
        inputs: () => {
        return {
            sceneResult: "sceneResult",
            depthResult: "depthResult",
            TAAHistoryOld: this.isEvenTick ? "TAAHistoryA" : "TAAHistoryB",
        }
        },
        shading: this.taaShader,
        source: [RenderGraph.quadSource],
        enableColorClear: false,
        beforePassExecute: () => {
        this.engine.unJit();
        const VP: Matrix4 = this.engine.getGlobalUniform(InnerSupportUniform.VPMatrix).value
        this.taaShading.VPMatrixInverse = this.taaShading.VPMatrixInverse.getInverse(VP, true); // TODO maybe add watch
        this.taaShading.sampleCount = this.sampleCount;
        },
    },
    {
        name: "TSSAO",
        inputs: () => {
        return {
            depthResult: "depthResult",
            AOAcc: this.isEvenTick ? "TSSAOHistoryA" : "TSSAOHistoryB",
        }
        },
        shading: this.tssaoShader,
        source: [RenderGraph.quadSource],
        enableColorClear: false,
        beforePassExecute: () => {
        const VP: Matrix4 = this.engine.getGlobalUniform(InnerSupportUniform.VPMatrix).value
        this.tssaoShading.VPMatrixInverse = this.tssaoShading.VPMatrixInverse.getInverse(VP, true);
        this.tssaoShading.sampleCount = this.sampleCount;
        },
    },
    { // copy to screen
        name: "CopyToScreen",
        enableColorClear: false,
        inputs: () => {
        let basic: string;
        let tssao: string;
        if (this.enableTAA) {
            basic = this.isEvenTick ? "TAAHistoryB" : "TAAHistoryA"
        } else {
            basic = "sceneResult"
        }
        if (this.enableTSSAO) {
            tssao = this.isEvenTick ? "TSSAOHistoryB" : "TSSAOHistoryA"
        } else {
            tssao = "sceneResult" // TODO consider design a way to bind default empty source? or recompile shader?
        }
        return { basic, tssao }
        },
        beforePassExecute: () => {
        this.composeShading.sampleCount = this.sampleCount;
        },
        afterPassExecute: () => {
        this.sampleCount++;``
        },
        shading: this.composeShader,
        source: [RenderGraph.quadSource],
    },
    ]
})

```