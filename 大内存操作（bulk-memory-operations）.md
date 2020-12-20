# 大内存操作与条件性段初始化

## 大内存操作的动机

当对WebAssembly进行基准测试时，有人提到`memcpy`和`memmove`函数非常热门。
一些例子:

- https://github.com/WebAssembly/design/issues/236#issuecomment-283279499

> 我最近查看了wasm一些基准测试的配置，发现一些最热门的函数都在做类似于`memcpy`和`memmove`的事情。
  如果这是正常wasm代码模式，我认为我们可以看到一个内在的显著改善，所以它可能值得优先考虑。

- https://github.com/WebAssembly/design/issues/977#issue-204960079

> 在许多游戏引擎中，我一直在优化和基准测试，有趣的是，memcpy()的性能在配置文件中显示得相对较高。(~2%-5%的总体执行时间)

### 大内存操作原型

我在v8实现了一个原型级`memory.copy`指令，它只能由v8的[`MemMove`函数](https://cs.chromium.org/chromium/src/v8/src/utils.h?l=446)调用。
我把它和[由emscripten生成的](https://gist.github.com/binji/c57dc945bba60985439ef8e5b574eee0)且使用在最近的单元测试demo里面的实现进行比较。
这个实现对齐然后通过`i32.load`和`i32.store`进行复制。
我同时记录了将循环展开和把大小增大到`i64`的性能测试。

每个测试都复制了`size`字节从一处到另一处，且无重叠。
重复了`N`次。
每次总共复制1Gb数据，并且在源和目标范围内只涉及1Mb的内存。

这个是核心循环:

```javascript
  let mask = Mib - 1;
  let start = performance.now();
  for (let i = 0; i < N; ++i) {
    f(dst_base + dst, src_base + src, size);
    dst = (dst + size) & mask;
    src = (src + size) & mask;
  }
  let end = performance.now();
```

基准测试的代码可以在[here](https://gist.github.com/binji/b8e8bc0c0121235d9f1668bc447c7f8c)找到。
注意，这个执行前要实现`memory.copy`。
在我的测试中，我把v8暴露出来的`memcpy`和`memmove`函数都替换为如下函数:

```wasm
(func (param $dst i32) (param $src i32) (param $size i32) (result i32)
  local.get $dst
  local.get $src
  local.get $size
  memory.copy
  local.get $dst)
```

这是在我的电脑上的结果 (x86_64，2.9GHz，L1 32k，L2 256k，L3 256k):

| | 原有的 | i64 load/store x 4 | i64 load/store x 2 | i32 load/store x 2 | i32 load/store |
| --- | --- | --- | --- | --- | --- |
| size=32b，N=33554432 | 1.382 Gib/s | 1.565 Gib/s | 1.493 Gib/s | 1.275 Gib/s | 1.166 Gib/s |
| size=64b，N=16777216 | 3.285 Gib/s | 2.669 Gib/s | 2.383 Gib/s | 1.861 Gib/s | 1.639 Gib/s |
| size=128b，N=8388608 | 6.162 Gib/s | 3.993 Gib/s | 3.480 Gib/s | 2.433 Gib/s | 2.060 Gib/s |
| size=256b，N=4194304 | 9.939 Gib/s | 5.323 Gib/s | 4.462 Gib/s | 2.724 Gib/s | 2.213 Gib/s |
| size=512b，N=2097152 | 15.777 Gib/s | 6.377 Gib/s | 4.913 Gib/s | 3.231 Gib/s | 2.457 Gib/s |
| size=1.0Kib，N=1048576 | 17.902 Gib/s | 7.010 Gib/s | 6.112 Gib/s | 3.568 Gib/s | 2.614 Gib/s |
| size=2.0Kib，N=524288 | 19.870 Gib/s | 8.248 Gib/s | 6.915 Gib/s | 3.764 Gib/s | 2.699 Gib/s |
| size=4.0Kib，N=262144 | 20.940 Gib/s | 9.145 Gib/s | 7.400 Gib/s | 3.871 Gib/s | 2.729 Gib/s |
| size=8.0Kib，N=131072 | 21.162 Gib/s | 9.258 Gib/s | 7.672 Gib/s | 3.925 Gib/s | 2.763 Gib/s |
| size=16.0Kib，N=65536 | 20.991 Gib/s | 9.758 Gib/s | 7.756 Gib/s | 3.945 Gib/s | 2.773 Gib/s |
| size=32.0Kib，N=32768 | 22.504 Gib/s | 9.956 Gib/s | 7.861 Gib/s | 3.966 Gib/s | 2.780 Gib/s |
| size=64.0Kib，N=16384 | 22.534 Gib/s | 10.088 Gib/s | 7.931 Gib/s | 3.974 Gib/s | 2.782 Gib/s |
| size=128.0Kib，N=8192 | 29.728 Gib/s | 10.032 Gib/s | 7.934 Gib/s | 3.975 Gib/s | 2.782 Gib/s |
| size=256.0Kib，N=4096 | 29.742 Gib/s | 10.116 Gib/s | 7.625 Gib/s | 3.886 Gib/s | 2.781 Gib/s |
| size=512.0Kib，N=2048 | 29.994 Gib/s | 10.090 Gib/s | 7.627 Gib/s | 3.985 Gib/s | 2.785 Gib/s |
| size=1.0Mib，N=1024 | 11.760 Gib/s | 10.091 Gib/s | 7.959 Gib/s | 3.989 Gib/s | 2.787 Gib/s |


## 条件性段初始化的动机

在当前的[threading proposal](https://github.com/WebAssembly/threads)下,
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


## 大内存操作+条件性段初始化的组合提案的动机

在[讨论条件性段初始化的设计](https://github.com/WebAssembly/threads/issues/62)时，
我们发现从只读数据段初始化程序内存(通过`memory.init`指令，下述)的行为与从线性内存中复制内存的提案指令(`memory.copy`，下述)相似。

## 设计

在内存和表中进行复制，要使用新的`*.copy`指令:

* `memory.copy`: 将线性内存中的一个区域，复制到另一个
* `table.copy`: 将表中的一个区域，复制到另一个

可以通过`memory.fill`来填充内存的一个区域:

* `memory.fill`: 用一个给定的字节填充线性内存的一个区域

[数据段的二进制格式](https://webassembly.github.io/spec/core/binary/modules.html#binary-datasec)
中有一系列的节，它们每个都有一个内存索引，一个offset的初始表达式，还有数据。

因为当前WebAssembly不允许多内存，每个节的内存索引都必须为0。
我们可以把这个32位整数作为一个flag值。

当flag值的低位为`1`时，这个节就是_被动的_。
一个被动节在初始化时，不会自动复制到内存或者时表里，而必须要使用如下的新指令:

* `memory.init`: 从数据段中复制一个区域
* `table.init`: 从元素段中复制一个区域

被动节没有初始化表达式，因为它会作为`memory.init`或者`table.init`的操作数。

节的大小也可以通过下面的新指令缩减为0:

* `data.drop`: 抛弃一个数据节中的数据
* `elem.drop`: 抛弃一个元素节中的数据

一个主动节和一个被动节相当，但是需要隐式地把一个`memory.init`跟着一个`data.drop`(或者一个`table.init`跟着一个`elem.drop`)添加到模块的开始函数的开头。

另外，引用类型提案介绍了引用函数的概念(函数的地址为一个程序值)。
为了支持这个协议，元素节应该支持多种编码，且能应用到其地址能被获得的前向声明函数上;下述。

引用类型提案同时介绍了`table.fill`和`table.grow`指令，它们都需要一个函数引用作为一个初始化参数。

### 数据节

数据节中的flag字段(a `varuint32`)的位含义如下:

| 位  | 含义                             |
| -   | -                                |
| 0   | 0=主动的，1=被动的                |
| 1   | 如果位0为0: 0=内存0，1=有内存索引 |

这将生成此视图:

| Flags | 含义          | 内存索引      | 在内存中的Offset  | 个数         | 荷载   |
| -     | -             | -            | -                | -           | -       |
| 0     | 主动          |              | `init_expr`      | `varuint32` | `u8`*   |
| 1     | 被动          |              |                  | `varuint32` | `u8`*   |
| 2     | 主动带内存索引 | `varuint32`  | `init_expr`      | `varuint32` | `u8`*   |

其余flag值都为非法。
当前内存索引值必须为0，但在未来的多内存提案中这将会改变。


### 元素节

元素节中的flag字段(a `varuint32`)的位含义如下:

| 位  | 含义                             |
| -   | -                                |
| 0   | 0=主动的，1=被动的                |
| 1   | 如果位0为0: 0=表0，1=有表索引     | 
|     | 如果位0为1: 0=主动的，1=已定义    |
| 2   | 0=带索引; 1= 带元素表达式         |

这将生成此视图:

| Flag | 含义                             | 表索引       | 在表中的in table | 编码          | 个数        | 荷载          |
| -    | -                                | -           | -               | -             | -           | -            |
| 0    | 旧版主动的，外部函数引用 |             | `init_expr`     |               | `varuint32` | `idx`*       |
| 1    | 被动的，外部               |             |                 | `extern_kind` | `varuint32` | `idx`*       |
| 2    | 主动的，外部                | `varuint32` | `init_expr`     | `extern_kind` | `varuint32` | `idx`*       |
| 3    | 已声明的，外部                  |             |                 | `extern_kind` | `varuint32` | `idx`*       |
| 4    | 旧版主动的，函数引用元素表达式  |             | `init_expr`     |               | `varuint32` | `elem_expr`* |
| 5    | 被动的，元素表达式                |             |                 | `elem_type`   | `varuint32` | `elem_expr`* |
| 6    | 主动的，元素表达式                 | `varuint32` | `init_expr`     | `elem_type`   | `varuint32` | `elem_expr`* |
| 7    | 已声明的，元素表达式               |             |                 | `elem_type`   | `varuint32` | `elem_expr`* |

其余flag值都为非法。
注意，"已声明的"属性不是用于本提案的，是用于引用类型提案的。

表示函数定义时，`extern_kind`必须为零。
`idx`是`varuint32`，它引用了模块中的一个实体，当前仅指向函数表。

当前表索引值必须为0，但在引用类型提案中会引入多表的概念。

`elem_expr`类似于`init_expr`，不过仅支持下列的形式:

| 二进制                | 文本                    | 描述                                |
| -                     | -                       | -                                          |
| `0xd0 0x0b`           | `ref.null end`          | 返回null引用                   |
| `0xd2 varuint32 0x0b` | `ref.func $funcidx end` | 返回函数`$funcidx`的引用 |

### 节的初始化

在MVP中，节是在模块初始化期间进行初始化的。
如果有任意节被初始化超出界限了，那么内存或表实例不会被修改。

这个行为将在大内存提案里改变:

每个主动节按照模块中定义的顺序进行初始化。
对每个节，如果对源的读取或对目标的写入超出界限，那么初始化会在该处失败。
而之前的数据写入会仍然保留

### `memory.init`指令

`memory.init`指令从给定的被动节中复制数据到目标内存。
目标内存和源节会作为立即数给出。

该指令有签名`[i32 i32 i32] -> []`。其参数按序如下:

- top-2: 目标地址
- top-1: 源节的offset
- top-0: 内存区域大小（单位为字节）

使用超出界限的节索引的时，`memory.init`会验证失败。

如下情况会发生陷入:

* 源的offset加上size，大于源数据节的长度;
  这里包括通过`data.drop`移除节的情况
* 目标的offset加上size，大于目标内存的长度

写入的顺序未作要求，当前无法进行观察。

注意，允许对同一数据节多次使用`memory.init`。

### `data.drop`指令

`data.drop`指令会把节的大小缩减为0。
当一个数据节被删除后，仍然可以对其使用`memory.init`指令，
但是仅有在offset0处的0长度访问不会引发陷入。
这个指令会作为wasm实现的一个优化提示。
当一个内存节被删除后，其数据将不能恢复，所以此节所使用的内存将被释放。

使用超出界限的节索引的时，`data.drop`会验证失败。

### `memory.copy`指令

将数据从源内存区域复制到目标区域。
如果这些区域在同一个内存中，并且一个区域的起始地址是另一个区域中读或写（通过复制操作）的地址之一，则称为重叠。

这个指令有两个立即数: 源和目标内存索引。当前它们都必须为0。

如果源与目标重叠，允许使用中间缓存。

该指令有签名`[i32 i32 i32] -> []`。其参数按序如下:

- top-2: 目标地址
- top-1: 源地址
- top-0: 内存区域大小（单位为字节）

如下情况会发生陷入:

* 源的offset加上size，大于源内存的长度; 
* 目标的offset加上size，大于目标内存的长度

边界检查发生于写入之前。

### `memory.fill` instruction

将一个内存区域里面的所有字节设置为给定的字节。
该指令有一个立即数，用于指定操作的内存，当前必须为0。

该指令有签名`[i32 i32 i32] -> []`。其参数按序如下:

- top-2: 目标地址
- top-1: 要设的值（字节）
- top-0: 内存区域大小（单位为字节）

如下情况会发生陷入:

* 目标的offset加上size，大于目标内存的长度

边界检查发生于写入之前。


### `table.init`，`elem.drop`，和`table.copy`指令

`table.*`指令的行为于`memory.*`指令相近，不同之处在于它们会对元素节和表进行操作，
而不是数据节和内存。
`table.init`与`table.copy`的offset和length操作数，描述的都是元素。

## 被动节的初始化示例

假设有两个数据节，第一个总为主动，而第二个在全局变量0为非零值时为主动。
应通过如下方法实现:

```webassembly
(import "a" "global" (global i32))  ;; 全局变量0
(memory 1)
(data (i32.const 0) "hello")   ;; 数据节0，其为主动，所以总会被复制
(data passive "goodbye")       ;; 数据节1，其为被动

(func $start
  (if (global.get 0)

    ;; 将数据节1复制到内存0里 (0是隐式的)
    (memory.init 1
      (i32.const 16)    ;; 目标offset
      (i32.const 0)     ;; 源offset
      (i32.const 7))    ;; 长度

    ;; 这个节所占有的内存已经没用了，所以可以删除这个节了
    (data.drop 1))
)
```

### 指令编码

所有的大内存指令其编码以0xfc开头，接着是操作码，可选的其他立即数:

```
instr ::= ...
        | 0xfc operation:uint8 ...
```

| 名称 | 操作码 | 立即数 | 描述 |
| ---- | ---- | ---- | ---- |
| `memory.init` | `0xfc 0x08` | `segment:varuint32`，`memory:0x00` | 复制一个被动数据节到线性内存中 |
| `data.drop` | `0xfc 0x09` | `segment:varuint32` | 阻止未来对被动数据节的使用 |
| `memory.copy` | `0xfc 0x0a` | `memory_dst:0x00` `memory_src:0x00` | 复制线性内存的一个区域到另一个区域 |
| `memory.fill` | `0xfc 0x0b` | `memory:0x00` | 将一个内存区域里面的所有字节设置为给定的字节 |
| `table.init` | `0xfc 0x0c` | `segment:varuint32`，`table:0x00` | 复制一个被动元素节到表中 |
| `elem.drop` | `0xfc 0x0d` | `segment:varuint32` | 阻止未来对被动元素节的使用 |
| `table.copy` | `0xfc 0x0e` | `table_dst:0x00` `table_src:0x00` | 复制表的一个区域到另一个区域 |

### `DataCount`段

WebAssembly的二进制格式被设计为一趟就能验证完。
如果一个段需要信息去验证，那么这些信息必然出现在前一个段里。

`memory.{init,drop}`打破了这个规矩。
这两个指令都应用在`Code`段里面。
它们都有数据节索引的立即数，但是此时还不能拿个数据节，直到`Data`段完成了转换，其转换发生在`Code`段之后。

为了维持一趟内完成验证，
`Data`段中定义的数据节的个数必须在`Code`段之前提供。
这个信息将在新的`DataCount`段中提供，其代码危机`12`。

和所有段一样，`DataCount`段是可选的。
如果有，必须以一下顺序出现:

| 段名称 | 代码 | 描述 |
| ------------ | ---- | ----------- |
| Type | `1` | 函数签名定义 |
| Import | `2` | 引入定义 |
| Function | `3` | 函数定义 |
| Table | `4` | 间接函数表和其他表 |
| Memory | `5` | 内存属性 |
| Global | `6` | 全局变量定义 |
| Export | `7` | 导出 |
| Start | `8` | 开始函数定义 |
| Element | `9` | 元素节 |
| DataCount | `12` | 数据节个数 |
| Code | `10` | 函数体(代码) |
| Data | `11` | 数据节 |

`DataCount`段只有一个字段，用于指定`Data`段中数据节的个数:

| 字段 | 类型 | 描述 |
| ----- | ---- | ----------- |
| count | `varuint32` | `Data`段中数据节的个数 |

如果`count`和`Data`段中数据节的个数不等，则其二进制格式是畸形的。
如果`DataCount`段不存在，且用上了`memory.init`或`data.drop`指令，则其二进制格式也是畸形的。 
