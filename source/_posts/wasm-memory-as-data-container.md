---
title: WebAssembly零拷贝批量数据交换和计算
date: 2019/3/28
---


最近在尝试将wasm用以webgl渲染引擎加速，有2个问题比较担心，1是据说js call wasm overhead很高，难以做函数级别的优化，最好批量处理，虽然mozila后来优化了相关问题，但是目前还没有bench过，包括其他浏览器也不是很确定，2是担心批量处理，需要从js端copy数据，再copy回来，这个开销比较难受。所以做了一翻调研。

假设我有一个很大的array需要传给webassembly，wasm-bindgen 可以生成 number slice的[接口](https://rustwasm.github.io/docs/wasm-bindgen/reference/types/number-slices.html), 大概类似这样：

```js
take_number_slice_by_shared_ref(new Float64Array(100));
take_number_slice_by_exclusive_ref(new Uint8Array(100));
```
大致是用户需要new一个typedarray来作为输入，需要在js端完成一次copy，临时内存分配和销毁带来的overhead并不合理，所以wasm能够真正应用得当需要寻找更合理的方式

[这个issue](https://github.com/rustwasm/wasm-bindgen/issues/270)，以及[这个](https://stackoverflow.com/questions/41875728/pass-a-javascript-array-as-argument-to-a-webassembly-function) 看到比较合理的从js端传送批量数据到webassembly进行处理的方案。

大致流程是：

1 调用wasm的方法，在wasm内存中分配空间，返回指针位置
2 js端在wasm的memory arraybuffer上，按指针位置和数据量建立view，把数据写入
3 调用wasm方法完成计算， 返回计算好的批量结果的指针位置和大小
4 js端在wasm的memory arraybuffer上，按指针位置和数据量建立view，把数据读出

主要情况是： wasm模块会有一个线性的内存，js端看就是一个arraybuffer，js端可以自由的读写。所以批量的数据写入可以通过直接在这个memory的arraybuffer上建立view来实现。甚至说，**我们可以直接将wasm的memory当作js部分的紧凑数据容器**，某些批量的数据处理和计算，可以直接调用wasm的方法来实现，js端可以直接在结果的读容器中获得。

在 rust 的webassembly的官方 game of life 的例子中我们可以看到这个[实现](https://rustwasm.github.io/docs/book/game-of-life/implementing.html), 直接访问wasm memory的数据

我自己测试了一下这种使用模式，似乎没有什么问题

```rust
#[wasm_bindgen]
pub struct Batcher {
    data: Vec<f32>,
}

#[wasm_bindgen]
impl Batcher {
    pub fn new() -> Batcher{
        Batcher {
            data:Vec::with_capacity(100)
        }
    }

    pub fn allocate(&mut self, capacity: u32) -> *const f32 {
        self.data = vec![0.0; capacity as usize];
        self.data.as_ptr()
    }

    pub fn batch(&self, batchLength: u32){
        for d in &self.data {
            log_f32(*d);
        }
    }
}
```

```js
const dataLength = 10;
const batcher = wasm.Batcher.new();
console.log(batcher)
const ptr = batcher.allocate(dataLength);
const dataview = new Float32Array(memory.buffer, ptr, dataLength);

dataview[0] = 1.5;
dataview[1] = 2;
dataview[2] = 3;
dataview[3] = 4;

batcher.batch(5);
```

这个原型大致可以推测出一种使用wasm memory作为数据容器以实现前端零拷贝高性能计算的模式： 将wasm的memory直接存储js数据。当然，我们在不考虑性能的情况下可以wrap一群js对象，通过getter setter，或者其他数据同步的设施，使得用户可以直接操作普通对象的方式操作wasm中的数据。在某些情况下，wasm可以直接对存储的数据进行重计算的操作，然后零拷贝的暴露出计算结果。这可能是wasm用法的一个比较好的实践。

实际的应用其实渲染引擎的确可以作为不错的尝试的例子，场景树，节点的js对象直接读写数据到wasm中，在每一帧渲染时，wasm模块自身负责高性能的渲染数据生成／同步，包括优化，排序，最后的renderlist直接暴露在memory中，js外层再一个batch读结果，完成gl调用。 在这个过程中js和wasm之间0数据拷贝，最小化直接交互调用，似乎没什么问题。

by the way, rust相关的wasm工具链包括rust自身的使用体验非常优秀。值得推荐。