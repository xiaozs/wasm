# 异常处理

这里有两个异常处理的备选方案
([第一个](https://github.com/WebAssembly/exception-handling/blob/master/proposals/old/Exceptions.md)
和
[第二个](https://github.com/WebAssembly/exception-handling/blob/master/proposals/old/Level-1.md))
然后我们
[决定](https://github.com/WebAssembly/meetings/blob/master/2018/TPAC.md#exception-handling-ben-titzer)
采用第二个方案，因为它会使用一等异常类型，更昂贵且对其他类型的事件更容易拓展。

本提案需要下列提案作为前提：

- [引用类型提案](https://github.com/WebAssembly/reference-types/blob/master/proposals/reference-types/Overview.md)，
  因为[`exnref`](#the-exception-reference-data-type)类型应该表现为`anyref`的子类型。

- [多值提案](https://github.com/WebAssembly/multi-value/blob/master/proposals/multi-value/Overview.md)，
  因为[`br_on_exn`](#exception-data-extraction)指令只和包含一个值的异常工作。
  此外，通过使用[多值](https://github.com/WebAssembly/multi-value/blob/master/proposals/multi-value/Overview.md)，
  [`try`块](#Try和catch块)可能会使用栈中的值，
  又或者往栈里推入多个值。

---

## 预览

异常处理允许代码在异常发生时打破控制流。
异常可以是WebAssembly模块已知的任何异常，
又或者它可能是被调用的导入函数引发的未知异常。

异常处理存在的问题是，WebAssembly和宿主对异常有不同的定义，但是双方都需要意识到对方的存在。

很难在WebAssembly中定义异常，因为(一般来说)它不知道其宿主为何。
再者，在WebAssembly中增加这些信息，会限制其他宿主对WebAssembly异常的支持。

有一个问题是，当双方都需要直到一个异常是否是由对方抛出的，因为需要做相应的清理操作。

另一个问题是，WebAssembly不能直接访问宿主的内存。
结果，WebAssembly将异常处理延迟到主机虚拟机。

为了访问异常，WebAssembly提供指令去判断该异常能否被WebAssembly识别。
如果能，则WebAssembly异常的数据将被提取、复制到栈上，从而允许后续指令去处理该数据。

最后，异常的声明周期可能为宿主所维护，所以它能收集和重用异常所占的内存。
这意味这宿主需要知道异常保存在哪里，所以这决定了异常什么时候能被垃圾收集。

这也意味这宿主需要给异常提供垃圾收集。
对于那些有垃圾收集的宿主(例如JavaScript)，
这不成问题。

但是，不是所有的宿主都会有垃圾收集器的。
因此，WebAssembly异常被设计为允许其他的内存管理方法，
例如引用计数，以在宿主中提供垃圾收集。

为此，WebAssembly异常一旦生成，就不能修改，以阻止引用计数所不能及的循环引用。
这也意味着异常不能保存到线性内存里面。
其理由有二:

* 为了安全。加载和存储不保证读取的数据与存储的数据类型相同。这允许欺骗异常引用，从而允许WebAssembly模块访问宿主虚拟机中不应该知道的数据。

* 宿主不知道线性内存中的数据的分布，所以它不知道异常引用保存在什么地方。

因此，当一个异常引用一个一等类型时，本提案不允许它们使用线性内存。
异常的引用类型可以表示为`anyref`类型的子类型，其于[引用类型提案](https://github.com/WebAssembly/reference-types/blob/master/proposals/reference-types/Overview.md)中引入。

当你使用`throw`指令时，会生成一个WebAssembly异常。
抛出的异常做如下处理:

1. 他们会被catch块捕获。捕获的异常被推入栈中。

1. 未被捕获的异常沿着调用栈，弹出调用帧，直到找到封闭的try块。

1. 如果调用栈中没找到任何封闭的try块，则由宿主决定如何处理未被捕获的异常。

### 事件处理

本提案为WebAssembly添加异常处理。
其一部分是为异常的声明定义新的段。
但是，与其让这个段只定义异常，它被定义为允许定义其他事件形式的更加通用的格式。

一般来讲，事件处理允许处理由代码生成的事件。
事件暂停当前执行并查找相应的事件处理程序。
如果找到了，运行相应的事件处理程序。
一些事件处理程序可能会将值发送回挂起的指令，允许源代码进行恢复。

异常是一种不会恢复的特殊的事件。
同样地，抛出指令是异常的挂起事件。
和try块关联的catch块定义了如何去处理抛出的异常。

WebAssembly事件(例如，异常)是由新的`event`段定义的。
`event`段定义了一系列与模块相关的事件。

每个事件都有`attribute`和`type`。
当前，`attribute`仅能将事件指定为异常。
在未来，新增的`attribute`值会为WebAssembly增加其他的事件。

为了支持未来的扩展，我们在二进制的异常定义中保留一个位，设其为0时意味着是异常，但目前我们不会在正式规范中使用术语event。

### 异常

`exception`是WebAssembly中的内部构造。
WebAssembly异常定义在模块的`event`和`import`段中。

异常的类型由一个索引定义，其指向一个定义在`type`段中函数签名。
其参数定义了一系列与异常相关的值。
其返回类型必须为空。

`exception tag`是一个用于识别不同异常的值，
而一个`exception index`是一个数字名称指向(导入或定义的)`exception tag`(详见[异常索引空间](#异常索引空间))。

异常索引为以下所用:

1. `throw`指令，通过对应的`exception tag`来新增一个WebAssembly异常，然后将其抛出。

2. `br_on_exn`指令，查询一个异常是否与对应的`exception tag`匹配。
   如果为真，执行对应分支并将相应的参数推入栈中。

### 异常引用数据类型

数据类型中新增了`exnref`类型，它引用了一个异常。
异常的表现留给了其实现。

### Try和catch块

一个_try块_定义了一系列用于处理异常的指令并会在异常抛出时对状态进行清理。
和其他高等结构一样，一个try块由`try`指令开始，以`end`指令结束。
换句话说，一个try块是一连串以下格式的代码:

```
try blocktype
  instruction*
catch
  instruction*
end
```

一个try块以`catch块`结束，其为一系列定义在`catch`指令后的指令所定义。

Try块和控制流块一样，有一个_块类型_。
try块的块类型由try块没有抛出异常时返回的值决定，或由catch块成功捕获后返回的值所决定。
因为`try`和`end`定义了控制流块，所以它们也适用`br`和`br_if`。

在最初的实现中，try块仅能返回0或1。

### 抛出异常

`throw`指令要一个异常索引作为立即数参数。
这个索引用于区分要生成并抛出相对应的异常。

栈顶的值必须与关联的异常类型对应。
这些值从堆栈中弹出并使用(与对应的异常标记一同)以生成对应的异常。
该异常随后被抛出。

当抛出一个异常时，宿主寻找最近在执行的try块体。
该try块被称为_捕获_try块。

如果抛出出现在一个try块中，则它就是捕获try块。

如果抛出出现在一个函数体中，且其调用不在一个try块中，未被捕获的异常沿着调用栈，弹出调用帧，直到找到封闭的try块或调用栈退空。
如果调用栈退空，则由宿主决定如何处理未被捕获的异常。
否则，找到封闭的try块就为捕获try块。

如果抛出出现在一个catch块中，它不会被对应的try块捕获，
因为catch块中的指令不在其对应的try块中。

一旦找到了捕获try块，操作数栈就会退回到进入该try块时的大小，然后一个指向异常的`exnref`会被推入到操作数栈中。

如果控制权转移到catch块，块体中的最后一个指令被执行后，控制权退出try块。

如果选中的catch块没有抛出异常，它必须返回捕获try块定义的值。
包括将捕获的异常退栈。

注意，被捕获的异常能通过`rethrow`指令重新抛出。

### 重抛异常

`rethrow`指令通过使用栈顶的`exnref`，来重新抛出异常。
除了没有新增异常意外，`rethrow`作用和`throw`一样。
将弹出堆栈顶部的引用异常，然后抛出。
如果栈顶值为null，`rethrow`指令会导致陷入。

### 异常数据获取

`br_on_exn`指令检查栈顶异常，以进行分支处理，其形式为:

```
br_on_exn label except_index
```

`br_on_exn`指令检查栈顶`exnref`是否与给出的异常索引匹配。
如果是，则跳到label指示的代码处
(在二进制格式里，label会转化成深度立即数，就像其他的分支指令一样)，
这时，从栈中弹出`exnref`值，并推入异常的参数值到栈顶。
为了使用这些值，目标分支的块签名必须与异常类型相匹配 - 因为它接收异常参数作为分支操作数。
如果异常标记不匹配，则`exnref`值留在栈中。
例如，当`exnref`包含一个类型为(i32 i64)的异常时，目标块的签名也应该为(i32 i64)，如下例:

```
block $l (result i32 i64)
  ...
  ;; exnref $e此时已经入栈
  br_on_exn $l ;; 带上$e的参数跳转到$l
  ...
end
```

这个现在可以用来构筑处理switch，以达到标准的`br_table`一样的功能:

```
block $end
  block $l1
    ...
      block $lN
        br_on_exn $l1
        ...
        br_on_exn $lN
        rethrow
      end $lN
      ;; 在这里处理$eN
      br $end
    ...
  end $l1
  ;; 在这里处理$e1
end $end
```

如果查询失败，则控制流被击穿，没有值入栈。
如果栈顶值为null，`br_on_exn`指令会导致陷入。

### 栈追踪

在抛出异常时，运行时会在穿越函数时退栈，直到找到对应的try块。
此时可应用对应的栈追踪，以报告未捕获的异常。
但是，其实现细节留给宿主。

### 陷入与JS API

`catch`捕获由`throw`指令生成的异常，但不会捕获陷入。
这样做的原因是，一般的陷入是无法在本地进行恢复的
且不需要在本地域中处理，就像try-catch。

`catch`指令也能捕获由引入函数生成的外部异常，包括JavaScript异常，除了一些例外:
1. 为了在陷入到达JavaScript帧之前和之后保持一致，
   `catch`指令不会捕获由陷入生成的异常。
1. `catch`指令不会捕获JavaScript栈溢出、内存溢出异常。

应该通过一个对JavaScript不透明的谓词，来过滤这些异常。
陷入当前会生成[`WebAssembly.RuntimeError`](https://webassembly.github.io/reference-types/js-api/#exceptiondef-runtimeerror)的实例，
但这个细节并不是用来决定类型的。
实现应该特别标记不可捕获的异常。
([`instanceof`](https://tc39.es/ecma262/#sec-instanceofoperator)谓词在JS中能被拦截，而栈溢出、内存溢出异常的类型是由实现定义的。)

## 对文本格式的修改

本节的描述放到[指令语法文档](https://github.com/WebAssembly/spec/blob/master/document/core/text/instructions.rst)中。

### 新指令

*instructions*中添加了如下新规则:

```
  try blocktype instruction* catch instruction* end |
  throw (exception except_index) |
  rethrow |
  br_on_exn label (exception except_index)
```

和`block`，`loop`，还有`if`指令一样，`try`指令是*结构化*控制流指令，且能上标签。
则允许分支指令退出try块。

`throw`和`br_on_exn`指令的`except_index`定义了应该生成/获取哪个异常。详见[异常索引空间](#异常索引空间)。

## 对模块文档的修改

本节的描述放到[模块文档](https://github.com/WebAssembly/design/blob/master/Modules.md)中。

### 异常索引空间

`异常索引空间`索引了所有导入的和内部定义的异常，根据导入和异常段中定义的顺序分配单调递增的索引。
因此，索引空间由0开始，以导入异常为头，跟着是定义在[异常段](#异常段)里面的内部定义异常。

异常索引空间定义了运行时异常标记的(模块)静态版本。
对于那些不是导入/导出的异常索引，相应的异常标记在所有加载的模块中都是唯一的。
导入或导出的异常将别名其他地方定义的相应异常，并使用相同的标记。

## 对二进制模式的修改

本节的描述放到[二进制编码设计文档](https://github.com/WebAssembly/design/blob/master/BinaryEncoding.md)中。

### 数据类型

#### exnref

对异常的引用指针。

### 语言类型

| 操作码 | 类型构造器 |
|--------|------------------|
| -0x18  |  `exnref`        |

#### value_type

由`varint7`描述的`value_type`包含了以上编码的`exnref`。

#### 其他类型

##### exception_type

我们预留了一位以指定异常特性:

| 名称      | 值 |
|-----------|-------|
| Exception | 0     |

每个异常都有以下字段:

| 字段 | 类型 | 描述 |
|-------|------|-------------|
| `attribute` | `varuint32` | 异常的属性 |
| `type` | `varuint32` | 对应的类型签名的类型索引 |

##### external_kind

由单字节无符号整数指定的类型:

* `0` 意味着 `Function` [导入](Modules.md#imports) 或
[定义](Modules.md#function-and-code-sections)
* `1` 意味着 `Table` [导入](Modules.md#imports) 或
[定义](Modules.md#table-section)
* `2` 意味着 `Memory` [导入](Modules.md#imports) 或
[定义](Modules.md#linear-memory-section)
* `3` 意味着 `Global` [导入](Modules.md#imports) 或
[定义](Modules.md#global-section)
* `4` 意味着 `Exception` [导入](#import-section) 或
[定义](#exception-section)

### 模块结构

#### 高等结构

新增了`exception`段。
如果存在，必须出现在`memory`段后。

##### 异常段

为了进行验证，此段出现在[内存段](https://github.com/WebAssembly/design/blob/master/BinaryEncoding.md#memory-section)后
、[全局段](https://github.com/WebAssembly/design/blob/master/BinaryEncoding.md#global-section)前。
所以，所有段的列表如下:

| 段名 | 代码 | 描述 |
| ------------ | ---- | ----------- |
| Type | `1` | 函数签名定义 |
| Import | `2` | 引入定义 |
| Function | `3` | 函数定义 |
| Table | `4` | 间接函数表和其他表 |
| Memory | `5` | 内存属性 |
| Exception | `13` | 异常定义 |
| Global | `6` | 全局变量定义 |
| Export | `7` | 导出 |
| Start | `8` | 开始函数定义 |
| Element | `9` | 元素节 |
| Code | `10` | 函数体(代码) |
| Data | `11` | 数据节 |

异常段定义了一系列异常类型，其描述如下:

| 字段 | 类型 | 描述 |
|-------|------|-------------|
| count | `varuint32` | 异常个数 |
| type | `exception_type*` | 异常类型的定义 |

##### 导入段

导入段被拓展为包含异常的定义，通过新增`import_entry`如下:

如果`kind`为`Exception`:

| 字段 | 类型 | 描述 |
|-------|------|-------------|
| `type` | `exception_type` | 被导入的异常 |

##### 导出段

导出段被拓展为包含异常的定义，通过新增`export_entry`如下:

如果`kind`为`Exception`:

| 字段 | 类型 | 描述 |
|-------|------|-------------|
| `index` | `varuint32` | 异常索引空间中对应的索引 |

##### 名称段

名称段的`name_type`被扩展为如下:

| 名称类型 | 代码 | 描述 |
| --------- | ---- | ----------- |
| [Function](#function-names) | `1` | 函数名称 |
| [Local](#local-names) | `2` | 函数中的局部变量名称 |
| [Exception](#exception-names) | `3` | 异常类型的名称 |

###### 异常名称

异常名称子段是一个`name_map`，它指定的名称对应的异常索引(为导入和模块内部定义所共用)。

### 控制流指令

控制流指令扩展为try块，catch块，throws，和rethrows如下:

| 名称 | 操作码 | 立即数 | 描述 |
| ---- | ---- | ---- | ---- |
| `try` | `0x06` | sig : `blocktype` | 开始一个能够处理抛出异常的块 |
| `catch` | `0x07` | | 开始try块对应的catch块 |
| `throw` | `0x08` | index : `varint32` | 新增一个由异常索引`index`定义的异常，然后将其抛出 |
| `rethrow` | `0x09` | | 弹出栈顶的`exnref`，然后将其抛出 |
| `br_on_exn` | `0x0a` |  relative_depth : `varuint32`，index : `varuint32` | 如果异常是由对应的异常索引`index`生成的，则跳转到给定的标签处，然后在栈顶入栈`exnref`的数据|

`block`，`if`，和`try`指令的*sig*参数是块签名，其描述了它们所用的操作栈。
