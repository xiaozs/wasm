# Wasm多内存

## 概述

本提案赋予单个wasm模块以使用多个内存的能力。
在当前版本的Wasm中，一个应用程序已经能够生成多个内存了，但只能将其分割到多个模块中。
单个模块或者函数不能同时引用多个内存。
因此，无法完成一些操作，例如有效率地把数据从一个内存转换到另外一个内存里，因为这必然设计及到对不同模块的每个值进行单独的函数调用。

## 动机

有一系列的单个应用中使用多个内存的场景:

* *安全性.* 一个模块可能希望分割和外界进行数据交换的公共内存和被包裹在模块内部的私有内存。

* *隔离性.* 即使是在单个模块内部，将两个线程之间共享的内存与以单线程方式使用的内存分开也是有利的。

* *持久性.* 应用程序可能希望在两次运行之间保持一些内存状态的持久性，例如，将状态保存在文件里。但我们未必想要保存整个内存，因此，通过多个内存来分离生命周期是一种自然的设置。

* *链接.* 有很多工具可以将多个Wasm模块合并为一个模块，作为静态链接的一种形式。这几乎可以适应所有情况，除非一组模块定义多个内存。在一个模块中允许多个内存可以弥补这个缺失。

* *可伸缩性.* 只要Wasm内存被限制在32位地址空间内，就没有办法有效地扩展到4GB内存以外。
 在64位内存可用之前，多个内存至少提供了一个有效的解决方案(这可能还要持续一段时间)。

* *仿真.* 一些提案，例如，[垃圾收集](https://github.com/WebAssembly/gc)或者[接口类型](https://github.com/WebAssembly/interface-types)如果它们能够添加一个不同于模块自身地址空间的辅助内存，则可以在当前的Wasm中进行仿真。


## 概述

最初的Wasm设计已经预料到了定义和引用多个内存的能力。特别是，已经有了记忆索引空间的概念(当前最多只能支持一个实例)，大多数内存结构已经在二进制编码中为它留下了空间。
本提案相应地填补了漏洞。

这种泛化完全对称于[reference types](https://github.com/WebAssembly/reference-types)提案中对多表的扩展。

这个扩展的设计几乎完全是规范的。具体地说:

* 允许在单个模块中导入和定义多个内存。

* 向所有与内存相关的指令添加内存索引。
  - 装载和存储有memop字段，其中，我们可以分配一个二进制格式的表示内存索引立即数的位。
  - 所有其他内存指令都已经有内存索引立即数(当前必须为0)。

* 数据段和导出段也已经有了内存索引。

* 以明显的方式扩展验证和执行语义。


### 指令

除非另有说明，以下更改适用于每个`load`和`store`指令，还有`memory.size`和`memory.grow`。
大容量内存提案引入的新指令，也需要做相应的改变。

抽象语法:

* 添加内存索引立即数。

验证:

* 检查内存索引立即数而不是索引0.

执行:

* 根据它们的索引立即访问内存，而不是内存0。

二进制格式:

* 对于`load`和`store`指令: 将`memarg`中的对齐值重新解释为位字段;
如果第6位(第一个LEB的MSB)已设置，那么立即数后将跟一个`i32`内存索引
(即使使用SIMD，在对数编码中，alignment值当前不能大于4，例如，占据低3位，这不仅关系到安全性)。

* 对于其他内存指令: 用`i32`内存索引更换硬编码`0x00`。

文本格式:

* 添加可选的`memidx`立即数，其省略的默认值是0(为了后向兼容)。
 对于`load`和`store`指令，这个索引必须出现在可选的alignment和offset立即数之前。


### 模块

#### 数据段和内存导出

数据段和内存导出的抽象语法已经带上了内存索引(尽管它现在只能是0)。
二进制和文本格式都已经有了相应的索引参数(在数据段中它是可选，默认为0的)。

从技术上讲，没有必要改变规范，尽管实现上需要处理索引不为0的情况。


#### 模块

* 模块验证不再检查是否最多定义或导入了1个内存。


## 实现

引擎已经需要处理多个内存，但是到目前为止，任何给定的代码只能处理一个内存。
因此，在当前的引擎中，为基址保留一个寄存器是一种常见的技术。
多个内存通常需要额外的间接寻址(一些引擎可能已经用上了)。

引擎可以保守地继续通过专用寄存器优化对内存索引0的访问。
大概，我们最终还可以添加一个带有优化提示的自定义部分，例如标记“主”内存的索引。
