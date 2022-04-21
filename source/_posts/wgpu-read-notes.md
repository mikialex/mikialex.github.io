---
title: wgpu read notes
date: 2022/4/21
categories:
- techology
tags:
- webgpu, wgpu, graphics
---

长期以来，各个操作系统，平台，厂商的发展出了多套现代的图形API，使得跨平台图形开发存在巨大的开发成本和兼容性问题。成熟的渲染引擎，一般都实现了一套跨平台的渲染抽象层用以封装和抹平不同图形API接口和使用的差异。但这样一套成体系的抽象层并没有一个公认的实现标准。web graphics，作为跨平台渲染最重要的应用场景，长期以来使用老旧的opengl es标准，无法满足未来图形应用的发展需求，webgpu成为大家的众望所归，不仅解决了web上下一代图形api的形态问题，更重要的是为桌面端跨平台渲染提供了一个最稳固的标准实现。

wgpu是rust的webgpu实现，目前被应用在firefox，deno等项目中实现webgpu支持。本文将介绍一下wgpu体系的整体架构，并针对性的对核心模块实现进行分析和摘录。此文基于wgpu 0.12.0的实现。

整个wgpu体系是较为复杂的(core 2w，hal 2w)，无论其核心模块还是hal实现都包含大量的细节，本文不会面面俱到指出所有细节。仅仅是从笔者自己关心的方向，进行探索性的阅读，重点将放在wgpu-core的资源管理上，不保证分析完全的正确性或是是否符合作者的本意。

![big-picture](/images/wgpu-read-notes/big-picture.png)

## 整体架构

根据上图：整个体系最重要的有3个crate组成：wgpu-hal：设计和实现了渲染抽象层：同时支持vk，metal，dx，opengl，多个后端。向外暴露出一组unsafe的需要手工管理资源的接口。wgpu-core：wgpu的核心实现，主要完成对标准的实现以及资源管理。对外暴露出真正的native版本的webgpu接口。wgpu：为rust应用提供的上层webgpu接口，完成native，web多实现的封装。

使用wgpu，可以使得你的rust程序编译为native的应用程序（使用wgpu-core的native实现），或是wasm版本的web应用程序（使用浏览器接口，如果是firefox，firefox底层还是会使用wgpu-core）或是native版的其他语言实现（比如chrome的dawn，这一点通过wgpu-native/wgpu headers完成）wgpu不仅承担了选择native还是web的逻辑，还有具体使用哪个图形api的逻辑，如果不加指定，wgpu会根据你的编译平台自行选择。

naga是个重要的模块，其完成的职责是shader的转换翻译和validation，webgpu可以使用wgsl，spirv等shader格式，但对于各个后端支持而言，需要进行shader翻译工作。同时webgpu作为safe的api，需要对shader和相关shader绑定的声明进行运行时验证，甚至对shader内部逻辑进行调整以满足安全性需求。naga的内部架构主要是一套树形的IR，支持多个shader语言的前后端，来完成转译工作。naga会基于IR执行一些分析和验证工作，但不会涉及到优化。

对于有更高性能要求，不希望wgpu-core做任何验证和资源管理的，可以直接使用wgpu-hal unsafe的api，wgpu-hal其实就是gfx-rs的简约版

## 抽象层

### web/native抽象层

这一层是wgpu rust api的主要内容。我们日常使用的对象如buffer，texture，device等就是该层提供的。可以看到，这一层的实现其实都是一些不透明的handle对象。实际的实现通过Context这个trait的抽象层进行转发。ctx定义了若干关联类型和操作的方法。

```rust
pub struct Buffer {
    context: Arc<C>,
    id: <C as Context>::BufferId,
    map_context: Mutex<MapContext>,
    usage: BufferUsages,
}

pub struct Texture {
    context: Arc<C>,
    id: <C as Context>::TextureId,
    owned: bool,
}

trait Context: Debug + Send + Sized + Sync {
    type AdapterId: Debug + Send + Sync + 'static;
    type DeviceId: Debug + Send + Sync + 'static;
    type QueueId: Debug + Send + Sync + 'static;
    type ShaderModuleId: Debug + Send + Sync + 'static;
    type BindGroupLayoutId: Debug + Send + Sync + 'static;
	...

    fn init(backends: Backends) -> Self;
    fn instance_create_surface(
        &self,
        handle: &impl raw_window_handle::HasRawWindowHandle,
    ) -> Self::SurfaceId;
    fn instance_request_adapter(
        &self,
        options: &RequestAdapterOptions<'_>,
    ) -> Self::RequestAdapterFuture;
    fn adapter_request_device(
        &self,
        adapter: &Self::AdapterId,
        desc: &DeviceDescriptor,
        trace_dir: Option<&std::path::Path>,
    ) -> Self::RequestDeviceFuture;
    fn instance_poll_all_devices(&self, force_wait: bool);
    fn adapter_is_surface_supported(
        &self,
        adapter: &Self::AdapterId,
        surface: &Self::SurfaceId,
    ) -> bool;
    fn adapter_features(&self, adapter: &Self::AdapterId) -> Features;
...

```

在wgpu/src/backend下可以找到direct和web版本的实现。

对于direct的实现，会直接使用wgpu-core的类型，对于web的实现，会直接使用web sys提供的类型（其实就是js对象）。

```rust
pub struct Context(wgc::hub::Global<wgc::hub::IdentityManagerFactory>);

impl crate::Context for Context {
    type AdapterId = wgc::id::AdapterId;
    type DeviceId = Device;
    type QueueId = wgc::id::QueueId;
    type ShaderModuleId = wgc::id::ShaderModuleId;
    type BindGroupLayoutId = wgc::id::Bi
	...
```

```rust
pub(crate) struct Context(web_sys::Gpu);

impl crate::Context for Context {
    type AdapterId = Sendable<web_sys::GpuAdapter>;
    type DeviceId = Sendable<web_sys::GpuDevice>;
    type QueueId = Sendable<web_sys::GpuQueue>;
    type ShaderModuleId = Sendable<web_sys::GpuShaderModule>;
    type BindGroupLayoutId = Sendable<web_sys::GpuBindGroupLayout>;
    type BindGroupId = Sendable<web_sys::GpuBindGroup>;
    type TextureViewId = Sendable<web_sys::GpuTextureView>;
```

### 渲染抽象层

另一套抽象层就是wgpu-core要使用的wgpu-hal，这其实是多套trait,定义了很多类型和他们之间需要交互的protocal。其后端实现在wgpu-hal下可以找到。

```rust
pub trait Api: Clone + Sized {
    type Instance: Instance<Self>;
    type Surface: Surface<Self>;
    type Adapter: Adapter<Self>;
    type Device: Device<Self>;
		...
}

pub trait Instance<A: Api>: Sized + Send + Sync {
    unsafe fn init(desc: &InstanceDescriptor) -> Result<Self, InstanceError>;
    unsafe fn create_surface(
...
}

pub trait Surface<A: Api>: Send + Sync {
    unsafe fn configure(
        &mut self,
        device: &A::Device,
        config: &SurfaceConfiguration,
    ) -> Result<(), SurfaceError>;
...
}

pub trait Adapter<A: Api>: Send + Sync {
...
}
```

**如何在上层做类型擦除？**

wgpu的前身的gfx-rs，这个项目是将所有api封装成一套unsafe的vulkan like api。这个项目如果使用不当的话，可能会造成的麻烦的是，后端的类型作为泛型参数，如果不加控制，会一路pop到最上层的用户接口，比如camera。这是非常不合理的。简而言之，将这个类型进行擦除，wgpu的做法是通过cargo feature对指定的类型别名进行条件编译，除了类型别名，大量direct后端实现会直接调用接口会通过gfx_select！宏来转发，而宏展开的内部也会应用相同的feature逻辑

## 资源管理

使用webgpu，而不是其他抽象层，重要的原因，其实是安全性/便捷性。用户不需要手工管理图形资源，不用担心正在使用的资源被错误的释放，不用担心没有插入正确的同步调用，等等。自然这些职责就落在在实现者上。

理解wgpu如何做好资源管理是比较重要的，原因其一是作为上层用户的使用者，webgpu内部的资源管理显然是一个具有overhead的黑盒，通过阅读实现可以帮助理解和避免高成本的api错误用法。其二是通过阅读实现也能学习到如何高效的安全的合理的对现代unsafe图形api进行资源管理。

### 资源管理器结构

从理解数据结构开始。

```rust
pub struct Global<G: GlobalIdentityHandlerFactory> {
// 实际渲染后端的instance类型
    pub instance: Instance,
// 管理swapchain相关
    pub surfaces: Registry<Surface, id::SurfaceId, G>,
    hubs: Hubs<G>,
}

// 实际存储后端的图形资源的容器集合
pub struct Hub<A: hal::Api, F: GlobalIdentityHandlerFactory> {
    pub adapters: Registry<Adapter<A>, id::AdapterId, F>,
    pub devices: Registry<Device<A>, id::DeviceId, F>,
    pub pipeline_layouts: Registry<PipelineLayout<A>, id::PipelineLayoutId, F>,
    pub shader_modules: Registry<ShaderModule<A>, id::ShaderModuleId, F>,
    pub bind_group_layouts: Registry<BindGroupLayout<A>, id::BindGroupLayoutId, F>,
    pub bind_groups: Registry<BindGroup<A>, id::BindGroupId, F>,
    pub command_buffers: Registry<CommandBuffer<A>, id::CommandBufferId, F>,
    pub render_bundles: Registry<RenderBundle, id::RenderBundleId, F>,
    pub render_pipelines: Registry<RenderPipeline<A>, id::RenderPipelineId, F>,
    pub compute_pipelines: Registry<ComputePipeline<A>, id::ComputePipelineId, F>,
    pub query_sets: Registry<QuerySet<A>, id::QuerySetId, F>,
    pub buffers: Registry<Buffer<A>, id::BufferId, F>,
    pub textures: Registry<Texture<A>, id::TextureId, F>,
    pub texture_views: Registry<TextureView<A>, id::TextureViewId, F>,
    pub samplers: Registry<Sampler<A>, id::SamplerId, F>,
}

// 存储单个资源类型的容器
// 实际的F实现IdentityHandlerFactory事实上和Storage是配套的
// 由下面若干类型展开看到，某个类型的资源是以generational arena方式进行管理的
pub struct Registry<T: Resource, I: id::TypedId, F: IdentityHandlerFactory<I>> {
    identity: F::Filter,
    data: RwLock<Storage<T, I>>,
    backend: Backend, // 后端类型的枚举
}

pub struct Storage<T, I: id::TypedId> {
    map: Vec<Element<T>>,
    kind: &'static str,
    _phantom: PhantomData<I>,
}

enum Element<T> {
    Vacant,
    Occupied(T, Epoch), // Eopch就是generation
    Error(Epoch, String),
}
// 管理generational arena的freelist
pub struct IdentityManager {
    free: Vec<Index>,
    epochs: Vec<Epoch>,
}
```

所有的资源读写都需要从registry访问，考虑到后续多线程支持，registry本身是添加读写锁的。为了从根本上避免死锁问题，这里采用类型系统的技巧来从编译期避免死锁：即通过限制资源获取锁的顺序，如何限制呢：通过trait，为资源类型之间定义出一整套DAG的关系，每解开锁需要某个前序类型的token，并返回后序类型的token，以此来限制资源获取顺序。

```rust
// 所有registry获取锁的都需要一个token，某个token的存在意味着当前某个类型的锁已经被打开了
pub(crate) struct Token<'a, T: 'a> {
    level: PhantomData<&'a T>,
}

/// Type system for enforcing the lock order on shared HUB structures.
/// If type A implements `Access<B>`, that means we are allowed to proceed
/// with locking resource `B` after we lock `A`.
///
/// The implenentations basically describe the edges in a directed graph
/// of lock transitions. As long as it doesn't have loops, we can have
/// multiple concurrent paths on this graph (from multiple threads) without
/// deadlocks, i.e. there is always a path whose next resource is not locked
/// by some other path, at any time.
pub trait Access<A> {}

pub enum Root {}
//TODO: establish an order instead of declaring all the pairs.
impl Access<Instance> for Root {}
impl Access<Surface> for Root {}
impl Access<Surface> for Instance {}
impl<A: hal::Api> Access<Adapter<A>> for Root {}
impl<A: hal::Api> Access<Adapter<A>> for Surface {}
impl<A: hal::Api> Access<Device<A>> for Root {}
impl<A: hal::Api> Access<Device<A>> for Surface {}
impl<A: hal::Api> Access<Device<A>> for Adapter<A> {}
impl<A: hal::Api> Access<PipelineLayout<A>> for Root {}
impl<A: hal::Api> Access<PipelineLayout<A>> for Device<A> {}
impl<A: hal::Api> Access<PipelineLayout<A>> for RenderBundle {}
impl<A: hal::Api> Access<BindGroupLayout<A>> for Root {}
impl<A: hal::Api> Access<BindGroupLayout<A>> for Device<A> {}
......
```

### 生命周期管理

我们知道，当用户drop了某个buffer，但是如果某个buffer还继续被bindgroup或是其他资源继续引用，是不能在底层直接销毁的。为此，wgpu实现需要考虑如何管理相关资源生命周期的问题。

整个问题的不能全部使用传统的引用计数来解决，是因为，1 wgpu的资源是以arena方式进行存储的，通信和计算都使用handle类型，2 某个资源即便没有任何引用，如果还在gpu运行中被使用，也是不能删除的。 针对1 wgpu通过自己的refcount raii类型，实现了在arena基础之上提供一套refcount机制，针对2 wgpu为每个resource绑定最后一次commandbuffer submission的index，decive端通过gpu fence来获取当前command执行情况（哪些commnadbuffer已经被执行完毕了），通过比较last submission index便可以知道该资源是否还在gpu中正常运行需要使用。

数据结构上：对于所有资源都是由device创建和销毁，每个device own这些resource，每个device实例上有此 lifetime tracker，每次resource上面都有对应的LifeGuard。

引用关系建立，refcount raii clone可以在device创建资源的代码中找到，drop就是各个资源的drop。由于refcount是附加在resource上的，并不是包裹在resource外，所以refcount置空也不会直接释放资源，所以事实上有个类似垃圾回收的pass会定期的回收。实现位于device.maintain. tracker不仅管理资源清理，还管理buffer mapping。maintain的执行结果会返回实际要做的资源清理列表以及map成功的列表。

用户drop资源，会将资源加入tracker的 suspected_resources中，queue write texture / buffer，创建的临时buffer，会加入到future_suspected_buffers/texture中，用户异步map的buffer会被加入到mapped中。在maintain逻辑中，会调用triage_suspected（检查所有用户当前指定销毁的资源，检查是否被gpu还在使用，如果是的装在active submission list中）， triage_mapped（检查当然map的请求，哪些buffer是否gpu已经使用完毕，装在ready_to_map中）， triage_submissions（检查active submission list，即过去的待销毁但被gpu使用中的资源是否还在被gpu使用）

maintain的调用时机是每次queue submit command buffer时，或者用户显式的poll device。对于native实现如果没有queue submit，整个event loop是需要定期持续poll的，对于web，浏览器会自动做这件事情，不需要js进行调用。在queue submit时，会更新最新的submission index到各个resource，同时检查引用计数并尝试添加到suspected_resources以供下一波triage_suspected来检查清理资源。

```rust
/// A struct responsible for tracking resource lifetimes.
///
/// Here is how host mapping is handled:
///   1. When mapping is requested we add the buffer to the life_tracker list of `mapped` buffers.
///   2. When `triage_suspected` is called, it checks the last submission index associated with each of the mapped buffer,
/// and register the buffer with either a submission in flight, or straight into `ready_to_map` vector.
///   3. When `ActiveSubmission` is retired, the mapped buffers associated with it are moved to `ready_to_map` vector.
///   4. Finally, `handle_mapping` issues all the callbacks.
pub(super) struct LifetimeTracker<A: hal::Api> {
    /// Resources that the user has requested be mapped, but are still in use.
/// 用户请求异步map的buffer会添加至此
    mapped: Vec<Stored<id::BufferId>>,
    /// Buffers can be used in a submission that is yet to be made, by the
    /// means of `write_buffer()`, so we have a special place for them.
    pub future_suspected_buffers: Vec<Stored<id::BufferId>>,
    /// Textures can be used in the upcoming submission by `write_texture`.
    pub future_suspected_textures: Vec<Stored<id::TextureId>>,
    /// Resources that are suspected for destruction.
// 任何资源在用户侧被删除，会添加至此集合
    pub suspected_resources: SuspectedResources,
    /// Resources that are not referenced any more but still used by GPU.
    /// Grouped by submissions associated with a fence and a submission index.
    /// The active submissions have to be stored in FIFO order: oldest come first.
    active: Vec<ActiveSubmission<A>>,
    /// Resources that are neither referenced or used, just life_tracker
    /// actual deletion.
    free_resources: NonReferencedResources<A>,
    ready_to_map: Vec<id::Valid<id::BufferId>>,
}

// resource本身会持有此lifeguard
pub struct LifeGuard {
    ref_count: Option<RefCount>,
// 记录该resource最后一次被commandbuffer submit的index
    submission_index: AtomicUsize,
}

struct ActiveSubmission<A: hal::Api> {
    index: SubmissionIndex,
    last_resources: NonReferencedResources<A>,
    mapped: Vec<id::Valid<id::BufferId>>,
    encoders: Vec<EncoderInFlight<A>>,
    work_done_closures: SmallVec<[SubmittedWorkDoneClosure; 1]>,
}

struct NonReferencedResources<A: hal::Api> {
    buffers: Vec<A::Buffer>,
    textures: Vec<A::Texture>,
...
}

pub(super) struct SuspectedResources {
    pub(super) buffers: Vec<id::Valid<id::BufferId>>,
    pub(super) textures: Vec<id::Valid<id::TextureId>>,
...
}
```

### Barrier生成

webgpu和metal类似，而不是和dx12，vulkan，一样需要用户自行设置resource/execution barriar。对于使用dx12/vulkan等后端的实现而言，需要wgpu自行设置正确的barrier，这需要对所有resource有一套精细的状态管理工作。这部分的实现可以在wgpu-core下的 tracker看到。

**标准回顾**

为了能让底层实现正确的处理barrier，所有的资源都有usage属性需要在用户创建时提供，usage定义了/限制了资源的使用方式，具体的spec是： [https://www.w3.org/TR/webgpu/#programming-model-resource-usages](https://www.w3.org/TR/webgpu/#programming-model-resource-usages)  核心规则是：在任意**usage scope**中，对任意**sub resource**，包含的**internal usage**是必须要相**兼容**的, 满足这样的规则，实现才可能针对所有平台做好barrier，否则是validation bug。

这里边有四个概念：

scope：scope可以理解是在某一段api调用范围，标准在 [https://www.w3.org/TR/webgpu/#programming-model-synchronization](https://www.w3.org/TR/webgpu/#programming-model-synchronization) 这一节中定义了这些scope：

- 在pass之外每个command独立为一个scope
- 在compute pass中，每个dispatch是一个scope，凡是被引用和间接引用到的资源都是被使用到的
- 每个render pass是一个独立的scope，以整个renderpass为单位，所有引用和间接引用到的资源都是被使用到的

定义这个scope是必要的，因为在底层api的实现中，可能是无法做到在任何时刻要求资源进行状态转换（比如vulkan里barrier没法在renderpass(subpass)中途生效）

sub resource：这个主要是针对texture的处理，因为显然，某个texture可能是有多个layer和level的，某个pass是可以读一个level写另一个level。所以对于一个texture不能使用一个usage来进行区分。简单来说，对于buffer，sub resource就是buffer，对于texture，sub resource是其下的某个level，layer的组合。

internal usage: 可以理解是对usage的进一步分类，分类为 input（如vertex/index buffer） constant（如uniform/texture） storage storage-read  storage-write attachment attachment-read

兼容的规则：某个resource在某个scope下，所有usage，要么只有作为一个attachment（写），要么只作为storage（写），要么只作为读（input constant storage-read）

总结一下这个限制就是**resource在某些使用阶段，要求他们没法同时支持读写的同时使用。**

实现：

**state抽象：**

```rust
/// The main trait that abstracts away the tracking logic of
/// a particular resource type, like a buffer or a texture.
pub(crate) trait ResourceState: Clone + Default {
    /// Corresponding `HUB` identifier.
    type Id: Copy + fmt::Debug + TypedId;
    /// A type specifying the sub-resources.
    type Selector: fmt::Debug;
    /// Usage type for a `Unit` of a sub-resource.
    type Usage: fmt::Debug;

    /// Check if all the selected sub-resources have the same
    /// usage, and return it.
    ///
    /// Returns `None` if no sub-resources
    /// are intersecting with the selector, or their usage
    /// isn't consistent.
    fn query(&self, selector: Self::Selector) -> Option<Self::Usage>;

    /// Change the last usage of the selected sub-resources.
    ///
    /// If `output` is specified, it's filled with the
    /// `PendingTransition` objects corresponding to smaller
    /// sub-resource transitions. The old usage is replaced by
    /// the new one.
    ///
    /// If `output` is `None`, the old usage is extended with
    /// the new usage. The error is returned if it's not possible,
    /// specifying the conflicting transition. Extension can only
    /// be done for read-only usages.
    fn change(
        &mut self,
        id: Valid<Self::Id>,
        selector: Self::Selector,
        usage: Self::Usage,
        output: Option<&mut Vec<PendingTransition<Self>>>,
    ) -> Result<(), PendingTransition<Self>>;

    /// Merge the state of this resource tracked by a different instance
    /// with the current one.
    ///
    /// Same rules for `output` apply as with `change()`: last usage state
    /// is either replaced (when `output` is provided) with a
    /// `PendingTransition` pushed to this vector, or extended with the
    /// other read-only usage, unless there is a usage conflict, and
    /// the error is generated (returning the conflict).
    fn merge(
        &mut self,
        id: Valid<Self::Id>,
        other: &Self,
        output: Option<&mut Vec<PendingTransition<Self>>>,
    ) -> Result<(), PendingTransition<Self>>;

    /// Try to optimize the internal representation.
    fn optimize(&mut self);
}
```

**数据结构：**

```rust
	// state tracking 的基本单元
pub struct Unit<U> {
    first: Option<U>,
    last: U,
}

// 描述某个类型的资源引用关系
struct Resource<S> {
    ref_count: RefCount,// 引用计数clone
    state: S, // 实现一般是上面的unit
    epoch: Epoch,
}

// tracking 同一类型多个资源的引用关系
pub(crate) struct ResourceTracker<S: ResourceState> {
    /// An association of known resource indices with their tracked states.
    map: FastHashMap<Index, Resource<S>>,
    /// Temporary storage for collecting transitions.
    temp: Vec<PendingTransition<S>>,
    /// The backend variant for all the tracked resources.
    backend: wgt::Backend,
}

// 描述所有类型资源的引用关系
pub(crate) struct TrackerSet {
    pub buffers: ResourceTracker<BufferState>,
    pub textures: ResourceTracker<TextureState>,
    pub views: ResourceTracker<PhantomData<id::TextureViewId>>,
	...
}
```

TrackerSet 由bindgroup，renderpassInfo，commandbuffer， bundle，device分别持有，来管理资源的状态和引用关系。

**流程：**

hal command buffer上的insert_barriers是最终的pass上主要的barrier生成（调用底层api）的方法（当然其他地方也有手工barrier设置，比如copy，clear）。从这个方法可以看到，barrier主要是由base和target的trackerset交互生成的，merge_replace会将目标的state，转移到base上，并且生成transition信息，交由底层生成barrier。insert_barriers 会在renderpass录制结束，每次compute dispatch，queue submit 时提交，正好对应spec中usage scope的定义。

以renderpass录制为例，会通过renderpassInfo上的tracker来收集当前动态提交的资源绑定命令，比如会merge_extend 来自bindgroup上trackerset缓存的usage，index/vertex buffer的usage，merge_extend bundle的trackerset缓存的usage，这个renderpassInfo上的tracker就是下面insert_barriers使用的target set。同时，当barrier被insert之后，意味着resource的state要被同步，意味着cmdbuffer上的state要被target merge。当cmdbuffer被submit时，同样的，会和device上的tracker state进行merge，并生成正确的barrier。

```rust
pub(crate) fn insert_barriers(
        raw: &mut A::CommandEncoder,
        base: &mut TrackerSet,
        head_buffers: &ResourceTracker<BufferState>,
        head_textures: &ResourceTracker<TextureState>,
        buffer_guard: &Storage<Buffer<A>, id::BufferId>,
        texture_guard: &Storage<Texture<A>, id::TextureId>,
    ) {
        profiling::scope!("insert_barriers");
        debug_assert_eq!(A::VARIANT, base.backend());

        let buffer_barriers = base.buffers.merge_replace(head_buffers).map(|pending| {
            let buf = &buffer_guard[pending.id];
            pending.into_hal(buf)
        });
        let texture_barriers = base.textures.merge_replace(head_textures).map(|pending| {
            let tex = &texture_guard[pending.id];
            pending.into_hal(tex)
        });

        unsafe {
            raw.transition_buffers(buffer_barriers);
            raw.transition_textures(texture_barriers);
        }
    }
```

**想法**

wgpu将对象引用关系和state维护建模在一起，是好的做法。可以理解整个state管理的架构是非常清晰的：同一个资源，在整个绘制流程中，可能会涉及到状态的切换，所以：

- 在创建资源时，同步引用关系的同时，就指定好合适的目标状态
- 时刻都存在一个当前真实的状态记录（tracker set），开始是是device的，然后是commandbuffer的，然后真实绘制时是passinfo的
- 在使用资源时，凡是遇到真实的状态记录需要切换即同步的，意味着需要插入barrier
- 标准对资源限定的用法保证了我们总是可以在正确的时机插入barrier

resource上面标记的usage仅仅是为了帮助配合spec做validation，实际usage在进行相关调用时当场决定，实际usage的存在是为了正确生成barrier调用。所以可以说根据现在的实现，如果在使用过程中，对资源标记额外的usage类型，即便没有使用，也不会有性能影响。

renderpass中收集use，并生成target的trackerset，其实是比较重的操作（可能仅次于validation）。bundle的trackerset其实是被缓存的，也是为什么使用bundle cpu的消耗较低的原因之一。

### buffer/texture初始化

webgpu标准要求，为了安全起见，所有的新创建的buffer/texture初始化必须是全零的（至少对于用户感知而言），但是从后端分配的资源，很可能是被重用、或是存在其他脏数据。需要实现某机制以lazy的方式控制初始化的流程来保证符合这一要求。具体的实现可以在wgpu-core下的init_tracker中看到，这里以buffer举例。

```rust
// Most of the time a resource is either fully uninitialized (one element) or initialized (zero elements).
type UninitializedRangeVec<Idx> = SmallVec<[Range<Idx>; 1]>;

/// Tracks initialization status of a linear range from 0..size
// 记录需要初始化的范围
pub(crate) struct InitTracker<Idx: Ord + Copy + Default> {
    // Ordered, non overlapping list of all uninitialized ranges.
    uninitialized_ranges: UninitializedRangeVec<Idx>,
}

pub(crate) enum MemoryInitKind {
    // The memory range is going to be written by an already initialized source, thus doesn't need extra attention other than marking as initialized.
    ImplicitlyInitialized,
    // The memory range is going to be read, therefore needs to ensure prior initialization.
    NeedsInitializedMemory,
}

pub(crate) struct BufferInitTrackerAction {
    pub id: BufferId,
    pub range: Range<wgt::BufferAddress>,
    pub kind: MemoryInitKind,
}

pub struct Buffer<A: hal::Api> {
    pub(crate) raw: Option<A::Buffer>,
// 每一个buffer都持有此tracker
    pub(crate) initialization_status: BufferInitTracker,
		...
}
```

其中 InitTracker记录未初始化范围并不是全部的，即所谓按需是指，在command submit阶段，会根据各个资源的使用范围，以及之前保存在tracker上未初始化的范围，收集需要手工初始化的范围，并在commandbuffer bake结束后，执行init(wgpu-core/src/command)，init会直接调用HAL encoder的clear方法，同时会处理好barrier方面的问题。

未初始化的数据也有可能被用户通过copy的方式隐式初始化，这种情况就不需要wgpu进行初始化，相关的逻辑可以从MemoryInitKind 的枚举值看出。

## 总结

wgpu-core的实现，除了资源管理，另一个重要部分就是validation，这部分比较无趣，涉及到的基本都是spec的细节，这里就不展开介绍了。我想，另一个有趣的问题可能更多的位于hal的后端实现部分，webgl2目前已经被部分支持了，考虑到我们做项目的需求，如果能直接使用那将对维护性有很大好处（不需要自行开发webgl/webgpu的抽象层），那么，支持的局限性如何？降级的成本如何？有没有可能进一步做到webgl1的支持？可能是我们下一个讨论的话题