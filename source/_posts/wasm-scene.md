---
title: WebAssembly 3D 场景树加速原型验证
date: 2019/4/30
---

继上次 「WebAssembly零拷贝批量数据交换和计算」 一文中的想法，我在我的webgl引擎项目中进行了初步实践。相应的代码可以在 [artgl](https://github.com/mikialex/artgl) 中找到。基本验证了上次的设想（使用wasm加速渲染数据生成更新并batch drawcall）是可行的。

## TLDR

主要实现了一些基本的场景数据和渲染数据存储在wasm 的memory中，场景节点提供普通的 js api。

**实测使用6层6叉的超大场景树，约6w节点，全部计算更新世界矩阵，耗时是threejs的不到一半。**（～6ms ／ ～14ms）

这基本证明了其性能优势是无法忽视的。

local矩阵的更新，世界bounding信息的更新，以及这些的按需行为，视锥剔除 细节剔除 drawcall排序 基数排序 这些将在后续完成实现

## 基本设计

scene对象负责管理用户描述场景的sceneNode节点，以及渲染数据的更新。

实际的数据存储在多个float32array／ int32array 中，这些array的视图建立在wasm内存arraybuffer上，初始化和扩容时调用scene对应的wasmscene的扩容方法，完成wasm内的扩容，返回这些array的ptr，用以更新typearray的视图。sceneNode上的场景数据，会以getter和setter的形式直接访问wasm内存中的数据。例如用户设置了某个节点的position， 数据会直接通过typedarray写入wasm内存中。

比如目前就已经实现了下列数据的包装：

```rust

local_transform_array: Vec<f32>,
local_position_array: Vec<f32>,
local_rotation_array: Vec<f32>,
local_scale_array: Vec<f32>,
world_transform_array: Vec<f32>，

local_aabb_array: Vec<f32>,
world_aabb_array: Vec<f32>,
local_bsphere_array: Vec<f32>,
world_bsphere_array: Vec<f32>,

empty_array: Vec<u8>,
empty_list_array: Vec<u16>,
empty_count: u16,

// [parent, left brother, right brother, first child]
nodes_indexs: Vec<i32>,

```

为了实现将tree存储于array中，tree的children使用双链表实现，每一个node存储前后一个兄弟，父亲，第一个child的索引位置。由于用户几乎没有随机访问某一个节点的children的需求，查询操作基本不受影响。sceneNode的js api 暴露add和remove方法会自动维护相关的索引。

empty_array和empty_list_array用以做标记删除

在后续会加入：

一个uint32的array使用位信息来表示每一个数据的change情况用以按需优化。

一个f32表示距离，一个f32表示屏幕投影大小。  一个uint32表示gl状态，包括shader／blend／culling ／visibility。这些可以组合起来做基数排序。

每次渲染时，scene通知wasmscene batch drawcall， 在wasm中完成所有计算，生成一个index list ，返回这个result indexlist的ptr和count， 然后js renderer加个接口直接读取就可以。  另外可以考虑的是直接使用wasm webgl的bindgen 这方面没有了解，可能需要调研和测试。

## 遇到的坑·已知风险·考虑：

wasm 的部分在chrome pref测量会导致性能问题。对实际运行无影响。

rust要开o3编译。

需要考虑asmjs的fallback，uc qq浏览器支持不好。

wasm scene和其他的比如three的scene有一个区别是，sceneNode 有attach状态并且只能属于一个scene，这个可能会引起上层设计改动。
