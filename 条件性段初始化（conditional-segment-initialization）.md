# 条件性段初始化提案

本页描述了为模块初始化提供跳过数据及元素段初始化机制的提案。

虽然以下基本原理仅适用于数据节，但本提案建议其解决方案也适用于单元节，一致性。

## 原理

在当前的线程提案下,
为了在多个代理间分享模块，模块必须进行多次初始化: 
每个代理一次。
通过模块中的数据段来初始化线性内存。
如果内存在多个代理间分享，它会被多次初始化，可能会覆盖先前初始化时写入的数据。

例如:

```webassembly
;; 该模块
(module
  (memory (export "memory") 1)

  ;; 一个作为counter的值
  (data (i32.const 0) "\0")

  ;; 给counter加一
  (func (export "addOne")
    (i32.store8
      (i32.const 0)
      (i32.add
        (i32.load8_u (i32.const 0))
        (i32.const 1)))
  )
)
```

```javascript
// main.js
let moduleBytes = ...;

WebAssembly.instantiate(moduleBytes).then(
  ({module, instance}) => {
    // 给counter加一
    instance.exports.addOne();

    // 新建一个Worker
    let worker = new Worker('worker.js');

    // 把模块发送给新Worker
    worker.postMessage(module);
  });

// worker.js

function onmessage(event) {
  let module = event.data;

  // 使用该模块生成新实例
  WebAssembly.instantiate(module).then(
    (instance) => {
      // 糟了，我们的counter被覆盖了
    });
}

```

这个例子可以运行在只初始化一次的独立模块上，然后把其内存导出给其他只包含代码的模块使用。
这能行，这很麻烦，因为本来只用一个模块就应该可以的。

## 提案: 用于初始化数据及元素节的新指令

[数据段的二进制格式](https://webassembly.github.io/spec/binary/modules.html#data-section)
中有一系列的节，它们每个都有一个内存索引，一个offset的初始表达式，还有数据。

因为当前WebAssembly不允许多内存，每个节的内存索引都必须为0。
我们可以把这个32位整数作为一个flag值。

当flag值的低位为`1`时，这个节就是_被动的_。
一个被动节在初始化时，不会自动复制到内存或者时表里，而必须要使用如下的新指令:

* `mem.init`: 从数据段中复制一个区域
* `table.init`: 从元素段中复制一个区域

被动节没有初始化表达式，因为它会作为`mem.init`或者`table.init`的操作数。

可以通过下列新指令缩减被动节:

* `mem.drop`: 阻止未来对数据节的使用
* `table.drop`: 阻止未来对元素节的使用

删除主动节会导致验证错误。

数据段编码如下:

```
datasec ::= seg*:section_11(vec(data))   => seg
data    ::= 0x00 e:expr b*:vec(byte)     => {data 0, offset e, init b*, active true}
data    ::= 0x01 b*:vec(byte)            => {data 0, offset empty, init b*, active false}
```

元素段编码类似。

### `mem.init`指令

`mem.init`指令从给定的被动节中复制数据到目标内存。
目标内存和源节会作为立即数给出。
该指令有3个`i32`操作数: 源节的offset，目标内存的offset，和复制的长度

当`mem.init`执行后，它的行为符合[指令](https://webassembly.github.io/spec/exec/modules.html#instantiation)中第11步,
但它的行为由`mem.init`操作数，源offset,目标offset，和复制的长度指定。

如下情况会发生陷入:
* 该节为被动
* 该节通过`mem.drop`删除后，被用到
* 访问任何源数据段或目标内存之外的字节

注意，允许对同一数据节多次使用`mem.init`。

### `mem.drop`指令

`mem.drop`指令阻止未来对给定节的使用。
一个数据节被删除后，再也不能对它使用`mem.init`指令。
这个指令会作为wasm实现的一个优化提示。
当一个内存节被删除后，其数据将不能恢复，所以此节所使用的内存将被释放。

### `table.init`和`table.drop`指令

`table.init`和`table.drop`指令其行为跟`mem.init`和`mem.drop`指令类似，
不同点在于一个操作元素节和表，一个操作数据节和内存。
`table.init`的offset和length操作数，描述的都是元素，而非字节。

### 示例

假设有两个数据节，第一个总为主动，而第二个在全局变量0为非零值时为主动。
应通过如下方法实现:

```webassembly
(import "a" "global" (global i32))  ;; 全局变量0
(memory 1)
(data (i32.const 0) "hello")   ;; 数据节0，其为主动，所以总会被复制
(data passive "goodbye")       ;; 数据节1，其为被动

(func $start
  (if (get_global 0)

    ;; copy data segment 1 into memory
    (mem.init 1
      (i32.const 0)     ;; 源offset
      (i32.const 16)    ;; 目标offset
      (i32.const 7))    ;; 长度

    ;; 这个节所占有的内存已经没用了，所以可以删除这个节了
    (mem.drop 1))
)
```
