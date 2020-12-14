# 接口类型提案

本提案新增了一套**接口类型**以描述高等值。
本提案位于WebAssembly[核心规范][core spec]最上层而且可以根据一个未修改的wasm核心引擎实现。

本提案假定[multi-value]和[module linking]提案，以及[function references]提案的[`let`]指令可用.

1. [动机](#动机)
2. [额外要求](#额外要求)
3. [提案](#提案)
   1. [接口类型](#接口类型)
      1. [接口值是延迟求值的](#接口值是延迟求值的)
      2. [接口值具有析构函数](#接口值具有析构函数)
      3. [接口值最多消费一次](#接口值最多消费一次)
      4. [接口值只向前流动](#接口值只向前流动)
   2. [适配器](#适配器)
      1. [适配器函数](#适配器函数)
      2. [适配器模块](#适配器模块)
   3. [提升与降低指令](#提升与降低指令)
      1. [整数](#提升与降低整数)
      2. [字符](#提升与降低字符)
      3. [列表](#提升与降低列表)
      4. [记录](#提升与降低记录)
      5. [变体](#提升与降低变体)
4. [一个端对端实例](#一个端对端实例)
5. [适配器融合](#适配器融合)
6. [重新审视用例](#重新审视用例)
7. [FAQ](#FAQ)
8. [TODO](#TODO)


## 动机

作为编译目标，WebAssembly仅提供低等类型以尽可能贴近计算机底层，而允许源语言使用低等类型高效地实现它自己的高等类型。
但是，当不同语言编译成的模块互相之间，或与宿主之间进行交流时，会带来如何转换高等类型的问题。
 
为了解决方案，我们提出了4个用例。在介绍完提案之后，会对其进行[重新审视](#重新审视用例).

### 定义像是WASI那样的语言中立接口

很多[WASI]的签名能通过`i32`表达，
将来，依据[type imports]，我们仍然有必要让WASI函数使用、返回复合值，如字符串或列表。
当前，这些值指定于线性内存里。
例如，在方法[`path_open`]里，字符串是通过线性内存里的一对表示offset和length的`i32`传递的。

但是，指定单一的、固定的数据类型表示（如字符串）将成为问题:
* 在[GC]提案中，调用者需要传入`ref array u8`;
* 宿主中存在一种不透明的本地字符串类型，WASI会接受且不会复制进、出wasm线性内存或者是GC内存;
* 支持多种字符串编码。

另外的问题是，传递`i32`offset进线性内存意味着被调用者有权限访问调用者的内存。
因此，当前所有的WASI客户端模块都需要暴露出它的内存让WASI实现通过各种特别的方式去访问。

理论上，WASI API由不可预料的高等类型表示，且调用者不需要暴露出他们的内存。

### 优化Web API的调用

一个长时间存在于WebAssembly和其他Web平台之间的矛盾是，
通过WebAssembly调用Web API通常需要使用JavaScript，
这不单止损害了性能也给开发者带来了另外的复杂性。
技术上，因为它们被暴露为JavaScript函数，同过Web IDL定义的函数能够直接通过[JS API]引入并调用。 
但是，Web IDL函数通常使用并产生高级Web IDL类型，例如 [`DOMString`]和[Sequence]，
但是WebAssembly只能传入numbers。
通过使用最新增加的[`externref`]，WebAssembly能够引入JavaScript函数，
通过[ECMAScript Binding]产生JavaScript值并传给Web IDL函数，但是这个的性能可能比JIT优化过的胶水代码来得要糟糕。

理论上，Web API的调用应该进行静态类型检查并传递直接由wasm产生的Web IDL值。

### 生成最大可重用模块

为了充分发挥WebAssembly的潜力，一个WebAssembly模块作者应该让尽可能多的客户重用他的模块(这里的"客户"可能是另一个WebAssembly模块，一个嵌入了WebAssembly的本地语言运行时又或者是其他的一些宿主系统)。
作为一个最大可重用模块，模块作者所使用的语言和工具链应该保持对实现细节的包装让模块客户能够独立的选择它们自己的语言和工具链。
这反过来又允许出现 单个、统一的最大可重用模块的WebAssembly生态系统。
反之，如果在这上面不做功夫，就会让WebAssembly的生态系统被各种语言和工具链切割得支离破碎。

如何界定这个用例的范围以避免实现任意语言之间的自动无缝集成是非常重要的，因为这通常是个棘手的问题。
实际上，我们可以观察**不共享任何内容**这个成功的模式在不同语言的互操作上的使用。
有名的案例包括Unix管道链接不同进程，
HTTP API链接不同的(微)服务。 **不共享任何内容**模式将整个应用程序划分成不同的单元各自封装着它们的可变状态; 共享可变状态要么被禁止，要么明显受限。 在使用多种语言时，把不同语言分割到不同单元里面是非常自然的。
在原生WebAssembly中，一个**不共享任何内容**单元就是一个只引入和导出函数和不变值的模块，但不引入、导出内存，表和可变全局变量。

当存在**不共享任何内容**模式时容易因为系统调用而产生通信开销，上下文切换和数据复制，
WebAssembly能够通过它的轻量级沙箱和使用同步函数调用去阻止这些资源开销(通过宿主API提供异步通信)。
因此，比起管道API，
不共享任何内容的WebAssembly模块能通过方法引入直接调用其他不共享模块。操作系统方面，类似于
[synchronous IPC]在某些微内核基于性能考虑的应用。但是，这种操作在WebAssembly里有一个明显的问题: 在线性内存里，一个函数如何传递一个大于wasm核心类型(`i32`，`i64`，其他)的值?

一个解决办法时通过[GC]提案传递不可变GC对象给不共享任何内容的WebAssembly。但是，这会在客户不需要GC时，对GC增加一个不必要的依赖。此外，当双方都是用线性内存或可变GC内存，这样的方法会产生不必要的额外复制和垃圾。
最后，在GC中有`struct`和`array`之类的高等类型时，安全性、可移植性和性能允许的情况下，GC提案仍然会寻求接近汇编的方法。因此，语言和工具链的细节仍然可能反映在一个模块的公共接口上。
理论上，不共享任何内容的模块，能够在不需要GC或者增加额外的复制和垃圾的情况下，通过使用丰富的高等类型定义它们的函数引入和导出。


## 额外要求

为了正确处理以上用例，这里有一些会影响可用解决方案的额外的要求:
* 解决方案应该允许各种形式的值，以回避中间(反)序列化步骤。 
  理论上，这甚至包含惰性求值的数据结构 (例如，通过迭代器，生成器或comprehensions)。
* 解决方案不应该强制独占使用线性或[GC]内存。
* 解决方案应该允许对存在于任何wasm内存之外的不透明宿主类型的高效转换。
* 解决方案不应该允许O(n)大小的引擎内临时值的内存分配。
* 解决方案不应该有微妙的性能拐点或过于依赖编译器特性。
* 解决方案应该允许高效健壮的后向兼容的API签名。
* 本提案也应该允许"共享一切"模块的链接
  (类似于[native dynamic linking])作为一个底层机制，以将公共库代码分解为无共享模块。


## 提案

本提案将首先介绍新类型，然后是包含这个新类型的新模块和函数，最后是能够生产和消费新类型并用在新函数上的新指令。

### 接口类型

为了补足核心wasm的*低等*数据类型(`i32`，`i64`，...)，
本提案定义了一套称为**interface types***高等*数据类型。在核心标准中，核心类型是由
[`valtype`]语法规定的. 对于interface types，对应的就是`intertype`，
由一下语法定义:
```
intertype ::= f32 | f64
            | s8 | u8 | s16 | u16 | s32 | u32 | s64 | u64
            | char
            | (list <intertype>)
            | (record (field <name> <id>? <intertype>)*)
            | (variant (case <name> <id>? <intertype>?)*)
```
`f32`和`f64`是出现在'valtype'中的相同类型，因此'valtype'和'intertype'在这两个类型中相交。
`char`代表[Unicode scalar value](即，非[surrogate]  [code point]).

不同于核心wasm的`valtype`，`intertype`支持递归定义且一定是递归定义。 这是因为interface types用于跨语言间的值复制和匹配其他IPC/RPC语言。

此外，当核心wasm的`valtype`包含引用到可变状态引用类型时，`intertype`复合值是传递不变的。
关于这个不同有一个重要结论，当`valtype`引用的子类型是非强转换的，并且受低级内存布局的限制的时候，`intertype`子类型能具备高强转换性，因此允许固定的后向兼容的接口API。 由于复制是接口类型与生俱来的特性，强制转换可以折叠到copy里。下面是一个类型转换表:

| 类型             | 允许转换 |
| ---------------- | ---------------- |
| `f32`，`f64`     | 从`f32`到`f64`   |
| `s8`，`u8`，`s16`，`u16`，`s32`，`u32`，`s64`，`u64` | 如果原值包含在目标类型里面 |
| `char`           | 不能 |
| `list`           | 如果列表中的对象能够转换 |
| `record`         | 如果字段的类型能够转换，无视顺序以名称匹配字段，且允许原字段被忽略 |
| `variant`        | 如果`variant`类型能被转换，无视顺序以名称匹配字段，且允许目标字段被忽略 |

当这些类型足够通用，可以捕捉到一系列经常出现在接口中的更具体类型时，
在API签名中显式地表示专用类型仍然是有益的。
例如，这允许在自动生成的源代码级API声明中使用专门的源语言类型。 
因此，本提案包含几种文本格式和二进制格式[缩写][abbreviations]。
在下面的符号中，`≡`左侧的类型在解析/解码过程中扩展为右侧的抽象语法。
因此，验证和执行纯粹是根据一般类型定义的；而缩写只存在于具体的格式中。
```
                                      string ≡ (list char)
                        (tuple <intertype>*) ≡ (record ("𝒊" <intertype>)*) for 𝒊=0，1，...
                             (flags <name>*) ≡ (record (field <name> bool)*)
                                        bool ≡ (variant (case "false") (case "true"))
                              (enum <name>*) ≡ (variant (case <name>)*)
                        (option <intertype>) ≡ (variant (case "none") (case "some" <intertype>))
                        (union <intertype>*) ≡ (variant (case "𝒊" <intertype>)*) for 𝒊=0，1，...
(expected <intertype>? (error <intertype>)?) ≡ (variant (case "ok" <intertype>?) (case "error" <intertype>?))
```
到目前为止，接口类型是相当正常的，并且你可能看到一些你期望在函数式语言或协议模式中看到的内容。
然而，为了支持这个提案的特定用例和需求，接口类型具有一些非典型属性，下面将介绍。

#### 接口值是延迟求值的

[上面](#额外要求)列出的附加要求之一是避免接口值的中间O(n)拷贝（无论是时间或者是内存的使用中）
这一要求似乎与上述引入一组新的不可变的`intertype`的做法不符，因为一个标准的[急切的][Eager Evaluation]
对`intertype`的解释要求制作一个`intertype`的临时复制。
虽然人们可以期待这个复制可以通过足够智能的编译器得到优化，由于在生产和执行指令时任意核心wasm交替执行，这样的优化在实践中几乎永远不会应用。

为了满足性能要求，`intertype`被赋予了一个[懒惰的][Lazy Evaluation]解释。
因此，`intertype`用于格式`(instruction operands*)`，
其中`instruction`是一个生产指令[下述](#提升与降低指令)
然后`operands*`是一组核心wasm值。
当`intertype`被一个消费指令消费(同样[下述](#提升与降低指令))，
只有这样`instruction`接受到它的`operands*`. 
因此，排除了副作用，编译器永远不必创建中间O(n)复制。

举个例，一个懒惰的`string`应该有这样的格式:
```wasm
(list.lift string $liftChar (i32.const 100) (i32.const 20))
```
[`list.lift`](#提升与降低列表)指令产生了一个列表并给出一组立即数和操作数（下面详述）。
当这个懒惰的`string`被消费(例如，通过[`list.lower`](#提升与降低列表))，
只有这样`list.lift`指令才会被执行去生产一个抽象的字符序列。

和独立的提升与降低指令一起，下面将更详细的描述懒惰性。

#### 接口值具有析构函数

虽然懒惰求值解决了中间复制的问题，但是它带来了新的问题，懒惰值如果引用的是一个动态分配的对象时，必须在读取其值后进行释放。
解决方法就是给懒惰值一个额外的、可选的"析构函数",这个函数会在懒惰值退栈后执行(无论时显式的指令消费还是通过流隐式的控制指令退栈)。
因此，懒惰值的格式变成`(instruction operands* destructor?)`。
当`destructor`被调用时，也会传递`operands*`，允许`operands*`作为结构函数的"闭包状态"。
从C++或者Rust看来，接口类型像是具有析构函数的局部范围对象。

和独立的提升与降低指令一起，析构函数将会在[下面](#提升与降低指令)详述。

#### 接口值最多消费一次

考虑到这些惰性和所有权属性，接口类型受到限制成为[仿射][affine]，
意味着它们的值不能被多次复制或消费. 
这个限制解决了两个问题:
1. 复制懒惰接口值不应制造O(n)复制，但是有两个懒惰值引用相同的底层动态分配对象将导致双重释放。
2. 接口值延迟执行可以产生任意副作用，因此，消耗同一个接口值多次会在生产端和消费端都产生意外行为，由此导致整体生态系统稳健性的降低。

和适配器函数一起，仿射将会在[下面](#适配器函数)详述。

#### 接口值只向前流动

接口类型有一个最后限制，超越[仿射](#接口值最多消费一次):
接口类型不能作为循环块的签名`param`使用。
这个限制确保了接口值只能“向前”流动，这使得实现可以轻松地将接口值的`operands*`快照为函数局部变量，
而不需要将`operands*`作为一个一等元组传递。

此特性将会在[下面](#adapter-fusion)，通过一个简单的编译时消除懒惰性方案详述。


### 适配器

这个建议是基于核心wasm规范和接口类型的，不能直接用于核心模块。
相反，该提案定义了一个新的能够使用接口类型并可以嵌套核心模块的模块。
这种新模块被称为**适配器模块**并包含能够使用接口类型及其指令的**适配器函数**。
通过使用[module linking]提案中的概念，适配器模块嵌套核心模块。

#### 适配器函数

适配器函数的结构与核心函数相同，有三个主要差异:
* 适配器函数与核心函数的定义不同，因此:
   * 在文本格式中，适配器函数以`adapter_func`开头，而不是以`func`开头。
   * 在二进制格式中，适配器函数进入新的适配器函数段中并填充新的适配器函数索引空间。
* 适配器函数允许使用`adaptertype`代替核心wasm的[`valtype`]。
* 适配器函数允许使用`adapterinstr`代替核心wasm的[`instr`].

`adaptertype`和`adapterinstr`定义如下:
```
adaptertype ::= valtype | intertype
adapterinstr ::= instr | ... new instructions introduced below
```
因此，允许适配器函数同时包含核心和接口类型和指令，可以看作是核心函数的超集。

对核心函数的验证规则也进行了扩展，以验证新的接口类型及指令。 
一个所有规则都必须遵守的重要约束是[如上所述](#接口值最多消费一次)的仿射。
这是通过以下验证规则实现的:
* 局部变量中不允许使用接口类型(函数作用域或者`let`域)，因为局部变量可以多次读取。
* 函数参数从隐式局部变量改为描述操作数堆栈的初始内容，就像块参数一样。
  因此，函数中的`(param)`声明将丢失其标识符.

除了仿射类型限制，接口类型可以与[参数指令][parametric instructions]和[控制指令][control instructions]共用。
例如，以下适配器函数将接口类型与内核`if`、`return`还有`drop`结合使用。
```wasm
(adapter_func $return_one_of (param string string i32) (result string)
  i32.eqz
  (if (param string string) (result string)
    (then return)
    (else drop return))
)
```

##### `rotate`

因为局部变量不仅仅用于复制值，也用于重新排序堆栈，适配器函数引入了一个新的`rotate`指令用于将一个值移到栈顶:
```
rotate $n : [ Tn+1 ... T0 ] -> [ Tn ... T0 Tn+1 ]
```
`rotate`是对`let`可以实现的功能的纯粹限制，但由于其编码明显更小，可能是对核心wasm的一个有价值的补充，
降低验证成本并改进静态use-def信息(对于一个不生成[SSA]的第一层编译器). (也参考[design/#1381](https://github.com/WebAssembly/design/issues/1381).)

##### `call_adapter`

因为适配器函数与核心函数占用不同的索引空间，它们不能通过`call`，`call_indirect`调用或间接调用，
`ref.func`，`call_ref`，等也一样. 相反，一个新的`call_adapter`用于调用适配器函数:
```
call_adapter $f : [ P* ] -> [ R* ]
有:
 - $f : (adapter_func (param P*) (result R*))
 - 索引 $f 应该小于等于配器函数的最大索引
```
间接调用函数的核心指令不适用于适配器函数。
只允许直接、非递归适配器调用的原因是，
在实例化时，wasm引擎可以有一个完整的所有适配器函数调用的精确调用图，
以允许引擎在实例化时可预测地将惰性求值编译为急切求值。
更多讨论[如下](#适配器融合).

#### 适配器模块

与适配器函数一样，适配器模块的文本和二进制格式与核心模块具有相同的结构。主要区别在于:
* 适配器模块是与核心模块不同的定义，因此:
  * 在文本格式中，适配器模块以`adapter_module`开头取代`module`.
  * 二进制格式中，适配器模块[序言][preamble]会设置一位以表示这是一个适配器模块(向后兼容地将32位`version`字段重新构造为两个16位`version`和`kind`字段).
* 适配器模块不能包含`func`，`memory`，`table`，`global`，`elem`和`data`定义。
  只允许`type`，`import`，`export`，`module`，`instance`和`alias`定义。
* 适配器模块可另外包含`adapter_func`，`adapter_module`和`adapter_instance`定义。

例如，适配器模块的一个简单示例是:
```wasm
(adapter_module
  (import "duplicate" (adapter_func $dup (param string) (result string string)))
  (import "print" (adapter_func $print (param string)))
  (adapter_func (export "print_twice") (param string)
    call_adapter $dup
    call_adapter $print
    call_adapter $print
  )
)
```
这个例子说明了可以编写纯适配器模块，
适配器模块的要点是调整核心模块的导入或导出。为此，适配器模块使用并扩展了[module linking]提案以导入和嵌套核心模块然后本地实例化这些核心模块，将适配器函数作为导入进行赋值，并从适配器函数调用导出。

例如，以下适配器模块适配核心模块导出 (通过`...`填充[如下](#提升与降低指令))新指令:
```wasm
(adapter_module
  (module $CORE
    (func $transform (export "transform") (param i32 i32) (result i32 i32)
      ;; core code
    )
    (memory $memory (export "memory") 1)
  )
  (instance $core (instantiate $CORE))
  (adapter_func (export "transform") (param (list u8)) (result (list u8))
    ...
    call $core.$transform
    ...
  )
)
```
导入也可以进行调整，尽管这样做需要更多的管道（由于实例化的非周期性）:
```wasm
(adapter_module
  (import "print" (adapter_func $print (param string)))
  (adapter_module $ADAPTER
    (import "print" (adapter_func $originalPrint (param string)))
    (adapter_func $print (export "print") (param i32 i32)
      ...
      call_adapter $originalPrint
    )
  )
  (adapter_instance $adapter (instantiate $ADAPTER (adapter_func $print)))
  (module $CORE
    (import "print" (func $print (param i32 i32) (result i32 i32)))
    (func $run (export "run")
      ;; core code
    )
  )
  (instance $core (instantiate $CORE (adapter_func $adapter.$print)))
  (adapter_func (export "run")
    call $core.$run
  )
)
```
这里，里面的`$ADAPTER`模块先实例化，因此它的导出(`$adapter.$print`)能用于初始化里面的`$CORE` 模块。
注意，核心的`instantiate`指令已被扩展为另外接受适配器函数。

就像模块有模块类型一样，适配器模块也有适配器模块类型。
例如，前面的适配器模块的类型是：
```wasm
(adapter_module
  (import "print" (adapter_func (param string)))
  (export "run" (adapter_func))
)
```
类似地，正如模块实例具有*实例类型*一样，适配器模块的实例具有*适配器实例类型*：
```wasm
(adapter_instance
  (export "run" (adapter_func))
)
```

在上面的示例中，因为`run`适配器函数没有实际的自适应，底层的`$core.$run`函数也可以直接导出:
```wasm
(export "run" (func $core.$run))
```
根据[module linking]，`$core.$run`是一个[alias definition]的语法糖:
```wasm
(alias $core_run (func $core $run))
(export "run" (func $core_run))
```
对上述适配器模块进行此更改后，适配器模块类型将变为:
```wasm
(adapter_module
  (import "print" (adapter_func (param string)))
  (export "run" (func))
)
```
这个例子演示了适配器模块类型如何同时包含核心和适配器定义。
同样，可以导入和导出核心内存、表和全局变量。


### 提升与降低指令

对每种新的接口类型，本提案还为添加了生成和消费接口类型的新指令。
这些指令分为两类:
* **提升指令**生成接口类型，从低级类型到高级类型。
* **降低指令**消费接口类型，从高级类型到低级类型。

通过在`valtype`和`intertype`之间显式变换和可编程，此提案允许提升和降低各种各样的低级表示，而无需任何中间序列化步骤。
这甚至包括从迭代器、生成器或comprehension等构造中延迟或动态生成的数据结构。

#### 提升与降低整数

此提案包括一个整数转换指令矩阵，允许在2个无符号核心整数类型和8个显式有符号接口整数类型之间进行转换:
```
ct ::= i32 | i64
it ::= u8 | s8 | u16 | s16 | u32 | s32 | u64 | s64

<it>.lift_<ct> : [ ct ] -> [ it ]

<ct>.lower_<it> : [ it ] -> [ ct ]
有:
  - bitwidth(ct) >= bitwidth(it)
```
对于提升，如果`bitwidth(ct)`大于`bitwidth(it)`，那么只有最不重要的`bitwidth(it)`个位的`ct`的值会被使用到。
对于降低，位宽度验证间限制可确保不会发生超出范围的错误。
在这两种情况下，`ct`的符号都取决于`it`的符号。

因为没有动态分配可以通过整数提升延迟读取，这里没有[析构函数](#接口值具有析构函数)。

作为示例用法，下面的适配器模块转换隐式签名的`i32`转换为显式无符号的`u32`：
```wasm
(adapter_module $ADAPTER
  (module $CORE
    (func $get_num (export "get_num") (result i32)
      (i32.const 0xffffffff)
    )
  )
  (instance $core (instantiate $CORE))
  (adapter_func (export "get_num") (result u32)
    (u32.lift_i32 (call $core.get_num))
  )
)
```
`$CORE.$get_num`直接被[JS API]调用，`i32`值`0xffffffff`将被[`ToJSValue`]解释为有符号值-1。
通过适配器模块，`get_num`显式返回一个`u32`然后`ToJSValue`会把它解释为2<sup>32</sup>-1.


#### 提升与降低字符

`char`严格定义为[Unicode Scalar Value]  (USV)，意味着[0，0xD7FF]和[0xE000，0x10FFFF]范围内的正整数值。
因此，提升与降低简单的映射了`i32`和`char`:
```
char.lift : [ i32 ] -> [ char ]
char.lower : [ char ] -> [ i32 ]
```
如果传递到`char.lift`的`i32`不在USV范围内，发生指令陷阱。
因此，使用`char.lower`能静态地判断生成的`i32`是合法的、在范围内的USV。
即使接口类型通常是惰性的，`char.lift`范围检查也会表现为急切的。

因为不可能有动态分配被`char.lift`惰性地读取，这个指令没有[析构函数](#接口值具有析构函数)。

虽然最初看起来这些指令隐式地假设了一个4字节([UTF-32])编码，但事实并非如此。
实际上，`char.lift`会在线性存储器解码出一个code point后使用，
`char.lower`会在编码一个code point进线性存储器之前使用。
因此，包含适配器的函数完全控制所使用的编码。

比模块之间如何传递单个字符更有趣的是如何传递整串字符。
执行此操作需要传递`(list char)`和下面介绍的列表的提升与降低指令。

#### 提升与降低列表

因为列表包含接口类型的元素，列表升降指令必须递归定义如何提升和降低元素。
提升通过使用`list.lift`指令:
```
list.lift $List $done $liftElem $destructor? : [ T:<valtype>* ] -> [ $List ]
有:
  - $List 引用一个 (list E:<intertype>)
  - $done : [ T* ] -> [ done:i32，U* ]
  - $liftElem : [ U* ] -> [ E，T* ]
  - $destructor : [ T* ] -> []，如果存在
```
为了生成`$List`，`list.lift`会重复一下动作:
* 调用`$done`，判断迭代是否完成。
* 调用`$liftElem`，提升下一个值到列表中。

`T*`和`U*`元组用于定义迭代的*循环携带状态*，也可用于保存索引、指针或一般数据结构状态。
循环携带状态传递方式如下:
* 最初传递给`list.lift`的`T*`被用于调用第一次的`$done`。
* `$done`的结果`U*`被传递到`$liftElem` (如果`$done`不是返回非零的`done`).
* `$liftElem`的结果`T*`被用于调用下一次的`$done`.

通过这个设计，`list.lift`允许在没有任何准备序列化的情况下直接提升各种列表表示形式:
* 一个均匀数组，带`$liftElem`递增索引或指针;
* 一个链表，带`$liftElem`跟随指针找到下一个节点;
* 一个`string`，带任意可变长度编码`char`;
* 一个有序树，带`$liftElem`对父节点进行出入栈;
* 一个列表comprehension，带`$liftElem`调用任意用户定义的代码。

降低列表比较简单，因为对输入列表的每一个元素调用一次`$lowerElem`。
```
list.lower $List $lowerElem : [ T:<valtype>*，$List ] -> [ T* ]
有
  - $List 引用了一个 (list E:<intertype>)
  - $lowerElem : [ E，T* ] -> [ T* ]
```
`list.lift`，通过`T*`每次调用`$lowerElem`描述`T*`的循环携带状态。
通过`list.lower`两侧输入输出的`T*`，适配器函数可以降到各种各样的列表实现中，包括预先分配和降低期间增量（重新）分配的实现。

因为[接口值是延迟求值的](#接口值是延迟求值的)，`list.lift`也是延迟求值的。
这意味着，当`list.lift`被执行时，它在语义上生成一个包含`list.lift`指令（包括它的立即数）的元组和`T*`操作数的复制。
仅当`list`被`list.lower`消费时，`list.lift`才会调用`$liftElem`。
因此，任何被`$liftElem`读取的线性内存分配必须保持有效知道`list.lower`完成。
如果需要，一旦`list`出栈，`list.lift`的`$destructor`要马上用于释放动态分配。

作为示例，一个`(list s32)` 可以从动态分配的连续数组中提升，如下所示:
```wasm
(adapter_func $done (param i32 i32) (result i32 i32 i32)
  (let (local $ptr i32) (local $end i32)
    (return (i32.eq (local.get $ptr) (local.get $end))
            (local.get $ptr)
            (local.get $end)))
)
(adapter_func $liftElem (param i32 i32) (result s32 i32 i32)
  (let (local $ptr i32) (local $end i32)
    (return (s32.lift_i32 (i32.load (local.get $ptr)))
            (i32.add (local.get $ptr) (i32.const 4))
            (local.get $end)))
)
(adapter_func $free (param i32 i32)
  (let (local $ptr i32) (local $end)
    (call $libc.$free (local.get $ptr)))
)
(adapter_func $liftArray (param i32 i32) (result (list s32))
  (let (local $ptr i32) (local $end i32)
    (return (list.lift (list s32) $done $liftElem $free (local.get $ptr) (local.get $end))))
)
```
下降到`i32`链接中，如下所示：
```wasm
(adapter_func $lowerElem (param s32 i32) (result i32)
  (let (param s32) (local $prevNext i32) (result i32)
    i32.lower_s32
    (call $core.$malloc (i32.const 8))
    (let (local $srcVal i32) (local $dst i32)
      (i32.store (local.get $dst) (local.get $srcVal))
      (i32.store (local.get $prevNext) (local.get $dst))
      (i32.add (local.get $dst) (i32.const 4))))
)
(adapter_func $lowerLinkedList (param (list s32)) (result i32)
  (call $core.$malloc (i32.const 4))
  (let (param (list s32)) (local $container i32)
    (list.lower (list s32) $lowerElem (local.get $container))
    (i32.store (i32.const 0))
    (return (local.get $container)))
)
```
如果`$liftArray`在适配器模块A中调用以生成一个`(list s32)`用于传递到适配器模块B中的`$lowerLinkedList`，
最终的结果是从A的数组表示转换成B的链表表示，然后释放A的内存。

##### 优化: 元素个数

虽然上述升降指令允许多种列表表示，但是这种普遍性是以性能为代价的。
要知道原因，考虑一下如果我们想降低上面的`(list s32)`到一个连续的数组中会发生什么。
因为我们不知道前面的尺寸，我们需要一些动态的重新分配策略。
而再分配策略的成本是[amortized O(n)]，实际上，它仍然比我们可以预先分配所需内存慢。
但是，对于各种列表表示（如生成器、迭代器和字符串），要求所有列表预先提供长度是有问题的。

因此，添加了以下两条指令，使列表生产者可以选择提供预先的元素计数:
```
list.lift_count $List $liftElem $destructor? : [ T*，count:i32 ] -> [ $List ]
有:
  - $List 引用了一个 (list E:<intertype>)
  - $liftElem : [ T* ] -> [ E，T* ]
  - $destructor : [ T* ] -> []，如果存在
```
因为`count`预先定义了迭代次数，所以不需要使用`$done`函数，整个指令也变得简单多了。
例如，前面的`$liftArray`适配器函数可以更简洁地重写为:
```wasm
(adapter_func $liftElem (param i32) (result s32 i32)
  (let (local $ptr i32)
    (return (s32.lift_i32 (i32.load (local.get $ptr)))
            (i32.add (local.get $ptr) (i32.const 4))))
)
(adapter_func $liftArray (param i32 i32) (result (list s32))
  (let (local $ptr i32) (local $end i32)
    (return (list.lift (list s32) $liftElem
              (local.get $ptr)
              (i32.shr_u (i32.sub (local.get $end) (local.get $ptr)) (i32.const 2)))))
)
```
在消费端，`list`消费者无法依赖`list.lift_count`。
相反，消费者可以通过`list.has_count`查询计数是否存在:
```
list.has_count : [ (list T) ] -> [ (list T)，maybe_count:i32，condition:i32 ]
```
如果`condition`非零，则`maybe_count`包含`count`传递给`list.lift_count`。
注意，通过传递未修改的`(list T)`，
从视点[仿射类型规则](#接口值最多消费一次)看来，`list.has_count`不会"消费"列表。

通过使用`list.has_count`，可以更有效地将`(list s32)`降低为连续数组:
```wasm
(adapter_func $lowerElem (param s32 i32) (result i32)
  (let (param s32) (local $dst i32)
    (i32.store (local.get $dst) (i32.lower_s32))
    (return (i32.add (local.get $dst) (i32.const 4))))
)
(adapter_func $lowerArray (param (list s32)) (result i32 i32)
  list.has_count
  if (param (list s32) i32)
    (let (param (list s32)) (local $count i32)
      (call $core.$malloc (i32.mul (local.get $count) (i32.const 4)))
      (let (param (list s32)) (local $dst i32)
        (list.lower (list s32) $lowerElem (local.get $dst))
        (return (local.get $dst) (local.get $count))))
  else
    drop  ;; pop maybe_count
    ...   ;; use a dynamic reallocation strategy with list.lower
  end
)
```
在这里，then分支利用由`list.has_count`提供的`i32` `$count`去预先分配`$dst`数组。
如果没有计数，else分支必须为一个动态分配策略(为了简洁起见，这里省略了它，但是在[下面的一个端对端实例](#一个端对端实例)中完整地显示了它)。

##### 优化: 正则表示

还有一个性能关键的案例需要解决:字符串。 
回顾[上面](#接口类型)，`string`是`(list char)`的缩写，
因此，要提升或降低一个`string`，适配器函数应该能使用组合：`list.lift`+`char.lift`或者`list.lower`+`char.lower`。
这很好地处理了不同生产者/消费者字符串编码的一般情况最终结果是一个直接的字符串转码。
但是，在一般情况下，生产者和消费者有相同的表示，转码将是次优的，
因为字符串在大多数情况下无法使用`list.lift_count` (它的单位是USVs的个数)。
因此，字符串将需要使用动态重新分配策略，即便目标的分配大小可以简单地从源的分配大小中获取。

为了找出优化方案，让我们先考虑另一个优化方案:
当一个`list`被传递到宿主API(例如：[Web API](#优化Web API的调用)和[WASI](#定义像是WASI那样的语言中立接口)用例)，
我们想在线性内存里定义一个单一的标准的`list`以便宿主以它的内部表示进行对齐，在使用标准表示时，0复制就很重要了。
理论上，一个足够聪明的编译器可以模式匹配到`list.lift`的`$liftElem`代码以判断什么时候能用上0复制兼容表示，
但是这样的优化是非常脆弱的。

为了实现这两个优化，添加了最后一组可选指令:
```
list.lift_canon $List <memidx>? $destructor? : [ T*，offset:i32，byte_length:i32 ] -> [ $List ]
有:
  - $List 是一个 (list E)，且 E 是一个标量类型(float，integral or char)
  - $destructor : [ T* ] -> []，如果存在

list.is_canon : [ (list E) ] -> [ (list E)，maybe_byte_length:i32，condition:i32 ]
  - $List 是一个 (list E)，且 E 是一个标量类型  (float，integral or char)

list.lower_canon <memidx>? : [ offset:i32，(list E) ] -> []
```
这些指令用法如下:
* `list.lift_canon`从给定的`<memidx>`(默认是memory 0)中，从给定的`offset`起，读取`byte_length`个字节，生成一个列表。
在列表出栈时，调用给定的`$destructor`。
* `list.is_canon`用于查询给定的`(list E)`是否已被`list.lift_canon`提升。如果是，返回原始的`byte_length`。
* `list.lower_canon`将给定的列表从给定的`offset`起写入到给定的`<memidx>`(默认是memory 0)中。

[下面](#一个端对端实例)给出了这些指令的示例用法。

作为这些指令的标准的一部分，精确指定了`$List`的规范。
而理论上可以为所有种类的列表定义一个规范表示，这在复合元素类型（列表、变体和记录的列表）的情况下引发了有趣的问题。
因此，本提案一开始保守地只允许“标量”类型(numbers and characters)，符合类型将于以后讨论。

当所有数值类型的规范表示是显而易见的时候，
由于它们的大小都是固定的2^n，`char`要求本提案选择任意字符编码。
为了匹配核心wasm规范[对于UTF-8的选择][core-wasm-utf8]，
和更普遍的趋势["到处都是UTF-8"][UTF8-Everywhere]，
本提案也会选择UTF-8。
通过选择一个规范的字符串编码快乐路径
同时为高效的转码提供了一个优雅的回退，接口类型提供了一个温和的压力，最终在没有性能悬崖的情况下收敛。

应该注意，许多语言与[潜在的非法UTF-16][potentially ill-formed UTF-16]有着长期的联系，
在这些语言运行时间一个常见优化是将字符串表示为两个完整字节[WTF-16]或者是当字符串不包含任何超出一个字节表示范围的代码点时使用单字节表示。
而现在这种单字节表示法通常是与UTF-8不兼容的[Latin-1]，
和与UTF-8兼容的纯7位[ASCII]，`list.lift_canon`允许所有的这些格式的字符串使用。

#### 提升与降低记录

和列表类似，记录是包含能够递归地提升与降低的接口类型字段的复合类型 that。
`record.lift`的签名如下:
```
record.lift $Record $liftFields $destructor? : [ T:<valtype>* ] -> [ $Record ]
有:
  - $Record 引用了一个 (record (field <name> <id>? F:<intertype>)*)
  - $liftFields : [ T* ] -> [ F* ]
  - $destructor : [ T* ] -> []，如果存在
```
通过`list.lift`，`T*`元组捕获被传入到`$liftFields`中以提升个别字段的核心wasm状态，然后提升的记录出栈时调用`$destructor`。
降低记录与此对称:
```
record.lower $Record $lowerFields : [ T:<valtype>*，$Record ] -> [ U:<valtype>* ]
有:
  - $Record 引用了一个 (record (field <name> <id>? F:<intertype>)*)
  - $lowerFields : [ T*，F* ] -> [ U* ]
```
分开的`T*`输入和`U*`输出让`record.lower`拥有最大的灵活性去在`$lowerFields`之前或其中进行分配。

通过`list`，`record`有了惰性的表示，因此`record.lift`是惰性计算的。
因此，任何为`$liftFields`读取的线性内存都必须保持有效，直到`record.lower`，如果需要的话，再在之后调用`$destructor`以释放内存。

例如，给定记录类型的以下类型定义：
```wasm
(type $Coord (record (field "x" s32) (field "y" s32)))
```
通过以下适配函数,可将一个在线性内存中包含两个`i32`的C`struct`提升为一个`$Coord`值:
```wasm
(adapter_func $liftFields (param i32) (result s32 s32)
  (let (local $ptr i32) (result s32 s32)
    (s32.lift_i32 (i32.load (local.get $ptr)))
    (s32.lift_i32 (i32.load offset=4 (local.get $ptr))))
)
(adapter_func $liftCoord (param i32) (result $Coord)
  record.lift $Coord $liftFields
)
```
然后降低到另外一个线性内存表示中，例如，两个反转顺序的`i64`:
```wasm
(adapter_func $lowerFields (param i32 s32 s32)
  rotate 2  ;; move the i32 to the top of the stack
  (let (param s32 s32) (local $ptr i32)
    (i64.store (local.get $ptr) (i64.lower_s32))
    (i64.store offset=8 (local.get $ptr) (i64.lower_s32)))
)
(adapter_func $lowerCoord (param i32 $Coord)
  record.lower $Coord $lowerFields
)
```
在传递`$liftCoord`的结果到`$lowerCoord`时，最终效果是执行一个交换和签名扩展的副本。

#### 提升与降低变体

类似地，其他复合类型，变体递归地提升与降低它们的接口类型。
`variant.lift`的签名如下:
```
variant.lift $Variant $case $liftCase? $destructor? : [ T:<valtype>* ] -> [ $Variant ]
有
  - $Variant 引用了一个 (variant (case <name> <id>? C:<intertype>?)*)
  - $liftCase : [ T* ] -> [ C[$case] ]，仅当 C[$case] 存在时
  - $destructor : [ T* ] -> []，如果存在
```
通过提升单个case，分支选择case的责任被提升到能够使用核心控制指令（如`br_if`和`br_table`）的适配器函数。

相反，`variant.lower`指令通过对每个case使用一个降低函数来控制动态分支本身:
```
variant.lower $Variant $lowerCase* : [ T:<valtype>*，$Variant ] -> [ U:<valtype>* ]
where
  - $Variant 引用了一个 (variant (case <name> <id>? C:<intertype>?)*)
  - 对每一个case: $lowerCase : [ T*，C? ] -> [ U* ]
```
分开的`T*`输入和`U*`输出让`variant.lower`拥有最大的灵活性去在`$lowerCase`之前或其中进行分配.

通过`list`和`record`，`variant`有了惰性的表示，因此`variant.lift`是惰性计算的。
因此，任何为`$liftCase`读取的线性内存都必须保持有效，直到`variant.lower`，通过时`$destructor`在变体值退栈时进行可选的内存释放。

例如，给定的以下的变体的类型定义:
```wasm
(type $MaybeAge (variant (case "has_age" $hasAge u8) (case "no_age" $noAge)))
```
可以从表示一个被读取后必须进行释放的内存对象的null或指针提升出一个`$MaybeAge`值:
```wasm
(adapter_func $free (param i32)
  call $core.$free
)
(adapter_func $liftHasAge (param i32) (result u8)
  i32.load
  u8.lift_i32
)
(adapter_func $liftMaybeAge (param i32) (result $MaybeAge)
  (let (local $ptr i32)
    (if (result $MaybeAge) (i32.eqz (local.get $ptr))
      (then (variant.lift $MaybeAge $hasAge $liftHasAge $free (local.get $ptr)))
      (else (variant.lift $MaybeAge $noAge))))
)
```
降低为使用哨兵值`-1`的压缩字编码以代表`no_age`的情况:
```wasm
(adapter_func $lowerHasAge (param u8) (result i32)
  i32.lower_u8
)
(func $lowerNoAge (result i32)
  i32.const -1
)
(adapter_func $lowerMaybeAge (param $MaybeAge) (result i32)
  variant.lower $MaybeAge $lowerHasAge $lowerNoAge
)
```
在传递`$liftMaybeAge`的结果到`$lowerMaybeAge`时，最终效果是
将空或对象编码转换为压缩编码，
然后释放非空的情况下对象。


## 一个端对端实例

在介绍完以上内容后，我们来看一个更加完整的例子，在这个例子中模块`$A` 导出了一个返回`(list u8)`的函数，这个函数又被模块`$B`导入和调用。
这个例子同时展示了如何在不分享`libc`的状态的情况下，将其代码奉献给`$A`和`$B`。

`$A`很直接，引入了`libc`通过嵌套模块生成并引入一个`$libc`实例。
(这里遵循了[动态链接的例子][shared-everything-example]的一般模式。)
```wasm
(adapter_module $A
  (type $Libc (instance
    (export "memory" $memory (memory 1))
    (export "free" $free (func (param i32)))
  ))
  (import "libc" (module $LIBC (export $Libc)))
  (instance $libc (instantiate $LIBC))
  (alias (memory $libc $memory))  ;; make $libc.$memory the default memory

  (module $CORE_A
    (import "libc" (instance (type $Libc)))
    (func $getBytes (export "get_bytes") (result i32 i32)
      ;; core code，returning (ptr，length) of allocated std::vector
    )
  )
  (instance $core (instantiate $CORE_A (instance $libc)))

  (adapter_func $freeVector (param i32 i32)
    drop  ;; drop the byte length
    call $core.$free
  )
  (adapter_func $getBytes (export "get_bytes") (result (list u8))
    call $core.$getBytes
    list.lift_canon (list u8) $liftByte $freeVector
  )
)
```
因为`$core.$getBytes`返回了一个内部缓存`std::vector`的所有权，当`list`退栈时，一个析构函数(`$freeVector`)被用于释放内存。
如果没有这个析构函数，`get_bytes`就无法知道如何释放`$core.$getBytes`返回的`std::vector`的内存。

`$B`有点复杂，因为列表的提升需要同时处理优化的case和一般case，还有引入的适配器需要额外的管道才能在缺少循环引用的情况下正常工作。
```wasm
(adapter_module $B
  (type $Libc (instance
    (export "memory" $memory (memory 1))
    (export "malloc" $malloc (func (param i32) (result i32)))
    (export "free" $free (func (param i32)))
    (export "realloc" $realloc (func (param i32) (param i32) (result i32)))
  ))
  (import "libc" (module $LIBC (export $Libc)))
  (instance $libc (instantiate $LIBC))

  (import "./A.wasm" (adapter_module $A
    (import "libc" (module (export $Libc)))
    (adapter_func (export "get_bytes") $getBytes (result (list u8)))
  ))
  (adapter_instance $a (instantiate $A (module $LIBC)))

  (adapter_module $ADAPTER
    (import "get_bytes" (adapter_func $originalGetBytes (result (list u8))))

    (import "libc" (instance $libc (type $Libc)))
    (alias (memory $libc $memory))  ;; make $libc.$memory the default memory

    (adapter_func $growingLowerByte (param u8 i32 i32 i32) (result i32 i32 i32)
      (let (param u8) (result i32 i32 i32)
        (local $dst i32) (local $length i32) (local $capacity i32)
        (if (i32.eq (local.get $length) (local.get $capacity))
          (then
            (local.set $capacity (i32.mul (local.get $capacity) (i32.const 2)))
            (local.set $dstPtr (call $libc.$realloc (local.get $dstPtr) (local.get $capacity)))))
        (i32.store8 (local.get $dstPtr) (i32.lower_u8))
        (i32.add (local.get $dstPtr) (i32.const 1))
        (i32.add (local.get $length) (i32.const 1))
        (local.get $cap))
    )
    (adapter_func $getBytes (export "get_bytes") (result i32 i32)
      (list.is_canon (call_adapter $originalGetBytes))
      if (param (list u8) i32) (result i32 i32)
        (let (result i32 i32) (local $byteLength i32)
          (call $libc.$malloc (local.get $byteLength))
          (let (result i32 i32) (local $dstPtr i32)
            (list.lower_canon (list u8) (local.get $dstPtr))
            local.get $dstPtr
            local.get $byteLength))
      else
        drop  ;; pop maybe_count
        (list.lower (list u8) $growingLowerByte
          (call $core.$malloc (i32.const 8)) (i32.const 0) (i32.const 8))
        drop  ;; pop capacity，leave [ptr，length] pushed
      end
    )
  )
  (adapter_instance $adapter (instantiate $ADAPTER (instance $libc) (adapter_function $a.$getBytes)))

  (module $CORE_B
    (import "libc" (instance (type $Libc)))
    (import "get_bytes" (func (result i32 i32)))
    ;; core code
  )
  (instance $core (instantiate $CORE_B (instance $libc) (adapter_func $adapter.$getBytes)))
)
```
这个例子说明的另一件事是[额外要求](#额外要求)中描述的"共享所有内容"和"不共享任何内容"。
特别是，
`$LIBC`是一个共享所有内容的库，是从两个不共享的模块`$A`和`$B`中分离出来的公共代码。
注意，`$LIBC`只分享了无状态代码;
`$A`和`$B`有它们私有的`$LIBC`实例。
因此，从外观看来，`$A`和`$B`都产生了不共享的实例，通过共享所有的链接作为实现细节。


## 适配器融合

虽然上述的提升指令和接口类型的惰性求制阻止了中间O(n)复制，但是我们担心这会给运行时的实现带来了额外的复杂性以及性能开销。
例如，为了惰性求值，[Haskell使用了thunks][Haskell uses thunks]且它的运行时围绕着thunks来实现。

但是，通过这些组合:
* [仿射类型限制](#接口值最多消费一次)，
  确保了接口类型最多被消费一次，
* [循环参数限制](#接口值只向前流动)，
  确保了接口值只向前流动，
* [适配器调用限制](#适配器函数)，
  确保了所有适配器调用都是直接的和非递归的，

适配器函数的惰性的实现能被很好地简化:
在实例化时，当导入为已知，又因为适配器函数的调用关系已知，
我们能把所有适配函数和接口类型编译为*核心函数*和*核心类型*。
通过这个被称为"适配器融合"的编译方案，所有的运行时复杂度和性能开销都被避免了。

如下是一个高层次的适配器融合算法的描述。
这个程序通过一个由核心实例(通过`adapter_function`操作`instantiate`)引入的适配器函数(下称"根")描述的。
因此，每引入一个适配器函数，这个算法都需要重复执行一次。
1. 适配器函数递归地内联到根中，直到没有`call_adapter`指令保留。
2. 对于每个提升指令的每个操作数，合成一个新的操作数的类型的`local`到根上。
   (注意: 提升指令只有核心操作数类型。)
3. 给提升指令的每一次调用加一个唯一的整数id.
4. 将每一个提升指令重写为:
   1. 对每一个操作数，通过`local.set`设置到第2步生成的`local`里面。
   2. 通过`i32.const`入栈第3步生成的id。
5. 将所有接口类型重写为`i32`(因为第4步，它们现在都是提升指令的核心类型)。
6. 给所有降低指令计算[可达定义][Reaching Definitions]。 
   (为了验证，降低指令会被 一个或多个类型兼容的提升指令到达。)
7. 用`br_table`重写每一个降低指令，switch可达的`i32`提升指令id(4.2处入栈且在6处被发现)，
   每一个case的提升指令应该包含:
   1. 对每一个提升指令的函数立即数的操作数(例如，`$liftElem`/`$liftFields`/`$liftCase`)，
      通过`local.get`从第2步生成、第4.1步初始化的`local`获取值。
   2. 提升指令的函数立即数和降低指令的函数立即数的组合递归调用此算法产生的两者的内联融合。
8. 在每一个节点中，如果有接口值退栈(降低指令，`drop`指令和控制流指令)，通过`br_table`切换可达的`i32`提升指令id，
   case里面都包含对应提升指令的`$destructor`，将在4.2捕获到的闭包状态传递给它。

由此，该算法通过积极的内联消除了惰性的几乎所有动态性。
即使是仅存的一个动态源(`br_table`s)也常常会在一个下降指令被一个提升指令到达时被优化掉(下面的例子就是了)。
对于普通代码，这种编译策略可能会有代码膨胀的风险，但是适配器代码只是核心模块之间的很薄的一层。
如果代码膨胀称为了问题，有一系列不那么激进的专业化编译策略。
在限制内，不需要执行内联;
惰性值可以被装箱到保存函数引用的元组中。

作为示范，[上一节中的](#一个端对端实例)适配器模块`$A`和`$B`会被融合到`$fused_root`函数中，具体如下。
也将根据类似的自动融合规则，生成包含模块。
注意嵌套模块`$CORE_A`和`$CORE_B`与原来嵌套在`$A`和`$B`的相同;
融合算法只操作*适配器模块*并保持核心模块不变。
```wasm
(module
  (type $Libc (instance
    (export "memory" $memory (memory 1))
    (export "malloc" $malloc (func (param i32) (result i32)))
    (export "free" $free (func (param i32)))
    (export "realloc" $realloc (func (param i32) (result i32)))
  ))
  (import "libc" (module $LIBC (export $Libc)))
  (instance $libc_a (instantiate $LIBC))
  (module $CORE_A
    (import "libc" (instance (type $Libc)))
    (func (export "get_bytes") (result i32 i32)
      ;; core code
    )
  )
  (instance $core_a (instantiate $CORE_A (instance $libc_a))
  (module $GLUE
    (import "" (func $core_a_get_bytes (result i32 i32)))
    (import "libc_a" (instance $libc_a (type $Libc)))
    (import "libc_b" (instance $libc_b (type $Libc)))

    (func $fused_root (export "get_bytes") (result i32 i32)
      (local $list_lift_ptr i32)  ;; added by step 2 for list.lift_canon
      (local $list_lift_len i32)  ;; added by step 2 for list.lift_canon

      ;; $A.$getBytes:
      call $core_a_get_bytes
      local.set $list_lift_len    ;; rewritten from list.lift_canon by step 4.1
      local.set $list_lift_ptr    ;; rewritten from list.lift_canon by step 4.1
      ;; i32.const added by step 4.2 eliminated as dead code

      ;; $B.$getBytes_:
      ;; list.is_canon is a constant expression because only one reaching lift
      local.get $list_lift_len
      ;; `if` now has a constant expression，so the `if` and `else` are removed
      let (result i32 i32) (local $byteLength )
        (call $libc_b.$malloc (local.get $byteLength))
        let (result i32 i32) (local $dstPtr i32)

          ;; emitted by step 7.2 for matching list.lift_canon/lower_canon
          (memory.copy $libc_a.$memory $libc_b.$memory  ;; using multi-memory proposal
            (local.get $dstPtr)
            (local.get $list_lift_ptr)
            (local.get $byteLength))

          ;; emitted by step 8: inline $A.$freeVector
          (call $libc_a.$free (local.get $list_lift_ptr))

          ;; end of $B.$getBytes then-branch
          local.get $dstPtr
          local.get $list_lift_len
        end
      end
    )
  )
  (instance $libc_b (instantiate $LIBC))
  (instance $glue (instantiate $GLUE (instance $libc_a) (instance $libc_b) (func $core_a.$getBytes)))
  (module $CORE_B
    (import "libc" (instance (type $Libc)))
    (import "get_bytes" (func (result i32 i32)))
    ;; core code
  )
  (instance $core_b (instantiate $CORE_B (instance $libc_b) (func $glue.$fused_root)))
)
```
如这个例子所示，当各模块都使用了权威的列表表示时，融合能够产生高性能核心wasm代码。
但是，即使是非规范列表也会产生一个单一的能同时迭代源和目标的融合循环，允许直接复制而无任何中间缓存。
在嵌套列表的例子中，嵌套的提升和降低循环也会被融合，因此不存在中间复制。

总结，适配器融合将多个适配器模块转换为单个融合核心模块，核心模块包含所有内存(通过使用[multi-memory])，
而融合函数直接在内存间复制。
因此，融合可以作为接口类型的实现技术，将已有的核心wasm规范[`module_instantiate`]程序和新的做适配器融合`adapter_instantiate`程序包裹在一起，生成的核心模块然后输入到`module_instantiate`。
再者，如果在构建时已知导入(例如，通过webpack)，融合可实现在构建时，最终不同wasm引擎的支持。


## 重新审视用例

我们现在考虑如何使用这个提案来处理[上面](#动机)给出的用例。

### 定义像是WASI那样的语言中立接口(重新审视)

通过这个提案，一个WASI接口能通过[适配器模块类型](#适配器模块)进行定义。
编写一个`.wit`/[`.witx`]文件，每一种源语言的工具链能够自动地生成源语言定义和适配器函数。

例如，[`fd_pwrite`]签名定义如下:
```wasm
(adapter_instance
  (type $FD (export "fd"))
  (type $Errno (enum "acces" "badf" "busy" ...))
  (adapter_func $fd_pwrite (export "fd_pwrite")
    (param $fd (ref $FD))
    (param $iovs (list u8))
    (param $offset u64)
    (result (expected u32 $Errno)))
)
```
通过`.wit`，不同的源语言定义和适配器函数会被生成。
例如，一个C++定义生成器会输出类似的代码:
```C++
template <class T> class handle { int table_index_; ... };
enum class errno { acces，badf，busy，... };

std::expected<uint32_t，errno>
fd_pwrite(handle<FD> fd，
          const std::vector<uint8_t>& iovs，
          uint64_t offset);
```
基于这些签名，生成的适配器函数会是:
```wasm
(adapter_func $lowerOk (param u32) (result i32 i32)
  i32.const 0
  i32.lower_u32
)
(adapter_func $lowerError (param $Errno) (result i32 i32)
  i32.const 1
  call_adapter $lowerErrno  ;; $lowerErrno omitted for brevity
)
(adapter_func $fd_pwrite_adapter (param i32 i32 i64) (result i32 i32)
  (let (result i32 i32) (local $fd i32) (local $iovs i32) (local $offset i64)
    (call_adapter $fd_pwrite
      (table.get $fds (local.get $fd))
      (list.lift_canon $liftByte (i32.load (local.get $iovs)) (i32.load offset=4 (local.get $iovs)))
      (u64.lift_i64 (local.get $offset)))
    variant.lower (expected u32 $Errno) $lowerOk $lowerError)
)
```
我们假设`std::vector`前两个字是指针和长度，`std::expected`表现为能通过[multi-value]返回的一个bool的`i32`和一个负载的`i32`。
C++中对`fd_pwrite`的调用被编译成对引入的`$fd_pwrite_adapter`的调用。
在实例化时，wasm引擎会把`$fd_pwrite_adapter`编译到一个有效的蹦床（会将C++线性内存里的`fd_pwrite`的参数的表示直接转换为被调用者的表示）里。

### 优化Web API的调用 (重新审视)

通过接口类型，一个wasm[适配器模块](#适配器模块)通过传递适配的高等值能直接调用Web API。
程序会启动在适配器模块被[JS API]或[ESM-integration]实例化时。
在这两种情况中，[wasm模块的实例化][*instantiate a WebAssembly module*]规范例程会被调用，
并传入一组导入的JS值。这个例程规定了传入的JS值是如何强制匹配模块声明的导入类型的。
通过本提案，这些规则自然会被扩展以考虑接口类型的导入:
* 如果传入的JS值是一个由Web IDL生成的内置函数
  (例如，对于一个[operation]，[getter] or [setter])且它的Web IDL签名"匹配"接口类型的引入签名(如下描述)，
  然后一个会生成一个[适配器函数](#适配器函数);
* 否则，接口类型值会被转换为JS值([如下所述](#embedding-webassembly-in-language-runtimes-revisited))，
  然后Web IDL函数会被当成JS函数通过[ECMAScript Binding]调用.

对这两条强制路径的一个重要要求是，这两条路径之间应该几乎没有可观察到的差异，允许将前一条路径视为后一条路径的优化。
由于JS语义角度看来，可能无法确保100%的语义等价性 (例如，`unrestricted double`的NaN规范或者是对`dictionary`缺失的属性的原型链遍历)。
虽然如此，在一般情况下，必须保持这种等价性，以确保Web api的JS虚拟化仍然有效
(所以JS函数可以对一个Web IDL函数进行填充、修复、衰减审查、虚拟化等等)。

因此，下列Web IDL/接口类型会匹配:
* [`any`] 匹配 `externref`
* [`void`] 匹配 空的函数返回值
* [numeric types] 匹配 数字接口类型
* [`USVString`] 匹配 `string` (这两个类型是同构的)
* [`DOMString`] 匹配 `string` (使用有损[`DOMString`-to-`USVString`]转换)
* [`ByteString`] 匹配 `string` (使用有损[`TextDecoder.decode`]转换)
* [Dictionary] 匹配 `record`
* [Sequence] 和 [Frozen array] 匹配 `list`
* [`boolean`] 匹配 `bool` 变量
* [Enumeration] 匹配 `enum` 变量
* [Nullable] 匹配 `option` 变量
* [Union] 匹配 `union` 变量
* [Interface] 匹配 [TODO](#TODO)
* [`ArrayBufferView`] 匹配 [TODO](#TODO)
* [Callback] 匹配 [`funcref`]
* [`object`]，[`symbol`]，[`Promise`] 匹配 `externref`
* [`ArrayBuffer`] 匹配 `externref` (在[`BufferSource`]以外很少使用)
* [Record] 匹配 `externref` (很少使用; 实际上不是 `record`)
* [Annotated] 递归地匹配 [inner type]  (注释添加被调用方的运行时检查)

上面所列的，`externref`为不常用的类型或JS/Web特定类型提供了一个转义填充。
在使用`externref`时，适配器模块可以导入helper函数来生成和使用`externref`。
未来，通过[type imports]，`externref`能替代`(ref $Import)`，以消除实例化时的适配器函数中的动态类型检查。

### 生成最大可重用模块 (重新审视)

为了生成最大可重用模块，开发人员将生成一个适配器模块，该模块专门使用其签名中的接口类型来创建一个无共享接口。
接口类型的值语义为客户语言提供了极大的灵活性，可以强制互转客户语言和本机语言的值。

例如，[JS API]会被扩展以提供以下转换(通过[`ToJSValue`]和[`ToWebAssemblyValue`]):
* 除了`s64`和`u64`的值类型能与JS值类型互转。
* `s64`和`u64`能与JS BigInt值互转。
* `string`能与JS字符串(通过[`DOMString`-to-`USVString`])互转。
* 如果JS的[records and tuples]提案通过，接口`record`和`list`能与这些新类型互转。
* `tuple`能生成JS元组。
* `bool`能与JS boolean互转。
* `enum`能与JS string基于`enum`的标签进行互转。
* `option`能将JS `null`或`undefined`转换为`none`，把其他JS值转换为`some`。
* `result`能将函数调用结果通过`error`生成异常，通过`ok`返回荷载。
* `union`转换为JS时会直接转换荷载。
  将JS转换为`union`非常模糊，但是可能会使用和Web IDL相同的临时方案[`union` ECMASCript Binding][ES Union]。
* 一般的`variant`接口类型不存在规范的JS映射。
  TypeScript最初支持[tagged unions]，通过使用一个带有特殊属性`kind`的对象作为判断，但是这个特性后来被推广到[User-Defined Type Guards]。
  对于`variant`，和对象`{kind:'name'，value:<payload>}`的互转可能可行。

请注意，当WebAssembly嵌入到本机JavaScript引擎中且当客户端以WebAssembly的形式运行JavaScript(在不包含JavaScript引擎的宿主上)时，可以应用相同的强制转换。以类似的方式，其他高级语言可以在接口值之间定义自己的强转语义。


## FAQ

### 接口类型如何与ESM-integration交互?

本提案对[ESM-integration]没有直接影响，且适配器模块应只在通过`import`或`<script type='module'>`引入时工作。
原因是ESM-integration是用来选择要编译的wasm模块，和决定传递哪个JS导入值到[*wasm模块的实例化*][*instantiate a WebAssembly module*]，
而本提案改变了在这些步骤中发生的事情.

通过[ESM-integration]和接口类型，WebAssembly使WebAssembly模块能够更进一步地完全访问Web平台，而不需要JS"胶水代码"。
实现这一目标所需的剩余功能，比如[get-originals]提案，它可以把JS和Web IDL API映射为内建模块，从而直接导入。


## TODO

* "outparams"来捕获`read(buffer)` / 类型化数组视图用例 ([#68](https://github.com/WebAssembly/interface-types/issues/68))
* 接口类型和不透明引用类型之间的提升和降低，允许双方使用时零拷贝
* 用于描述处理资源生存期的资源句柄的类型 (见[#87](https://github.com/WebAssembly/interface-types/issues/87))
* 可选的/带默认值的记录字段或函数参数
* 对宿主函数的精确调用应该怎么做
* 导入/导出接口*值*(与适配器函数相反)的能力，从而允许适配器模块导入配置数据和[JSON modules]


[Core Spec]: https://webassembly.github.io/spec/core
[JS API]: https://webassembly.github.io/spec/js-api/index.html
[`ToJSValue`]: https://webassembly.github.io/spec/js-api/index.html#tojsvalue
[`ToWebAssemblyValue`]: https://webassembly.github.io/spec/js-api/index.html#towebassemblyvalue
[*instantiate a WebAssembly module*]: https://webassembly.github.io/spec/js-api/index.html#instantiate-a-webassembly-module
[Web API]: https://webassembly.github.io/spec/web-api/index.html
[C API]: https://github.com/WebAssembly/wasm-c-api
[Abbreviations]: https://webassembly.github.io/reference-types/core/text/conventions.html#abbreviations
[`name`]: https://webassembly.github.io/spec/core/text/values.html#names
[`valtype`]: https://webassembly.github.io/spec/core/syntax/types.html#syntax-valtype
[`instr`]: https://webassembly.github.io/spec/core/syntax/instructions.html#syntax-instr
[`hostfunc`]: https://webassembly.github.io/spec/core/exec/runtime.html#syntax-hostfunc
[`module_instantiate`]: https://webassembly.github.io/spec/core/appendix/embedding.html#mathrm-module-instantiate-xref-exec-runtime-syntax-store-mathit-store-xref-syntax-modules-syntax-module-mathit-module-xref-exec-runtime-syntax-externval-mathit-externval-ast-xref-exec-runtime-syntax-store-mathit-store-xref-exec-runtime-syntax-moduleinst-mathit-moduleinst-xref-appendix-embedding-embed-error-mathit-error
[Parametric Instructions]: https://webassembly.github.io/spec/core/syntax/instructions.html#parametric-instructions
[Control Instructions]: https://webassembly.github.io/spec/core/syntax/instructions.html#control-instructions
[Preamble]: https://webassembly.github.io/spec/core/binary/modules.html#binary-version
[core-wasm-utf8]: https://webassembly.github.io/spec/core/binary/values.html#binary-utf8

[`externref`]: https://webassembly.github.io/reference-types/core/syntax/types.html#syntax-reftype
[`funcref`]: https://webassembly.github.io/reference-types/core/syntax/types.html#syntax-reftype

[Function References]: https://github.com/WebAssembly/function-references/blob/master/proposals/function-references/Overview.md
[`let`]: https://github.com/WebAssembly/function-references/blob/master/proposals/function-references/Overview.md#local-bindings

[Type Imports]: https://github.com/WebAssembly/proposal-type-imports/blob/master/proposals/type-imports/Overview.md#imports

[Multi-value]: https://github.com/webassembly/multi-value
[Multi-memory]: https://github.com/webassembly/multi-memory

[Module Linking]: https://github.com/WebAssembly/module-linking/blob/master/proposals/module-linking/Explainer.md
[Alias Definition]: https://github.com/WebAssembly/module-linking/blob/master/proposals/module-linking/Explainer.md#instance-imports-and-aliases
[Shared-everything Dynamic Linking]: https://github.com/WebAssembly/module-linking/blob/master/proposals/module-linking/Explainer.md#shared-everything-dynamic-linking
[Shared-everything-example]: https://github.com/WebAssembly/module-linking/blob/master/proposals/module-linking/Example-SharedEverythingDynamicLinking.md

[GC]: https://github.com/WebAssembly/gc/blob/master/proposals/gc/Overview.md

[WASI]: https://github.com/webassembly/wasi
[`path_open`]: https://github.com/WebAssembly/WASI/blob/master/phases/snapshot/docs.md#-path_openfd-fd-dirflags-lookupflags-path-string-oflags-oflags-fs_rights_base-rights-fs_rights_inherting-rights-fdflags-fdflags---errno-fd
[`fd_pwrite`]: https://github.com/WebAssembly/WASI/blob/master/phases/snapshot/docs.md#-fd_pwritefd-fd-iovs-ciovec_array-offset-filesize---errno-size
[`.witx`]: https://github.com/WebAssembly/WASI/blob/master/docs/witx.md

[ESM-integration]: https://github.com/WebAssembly/esm-integration/tree/master/proposals/esm-integration

[get-originals]: https://github.com/domenic/get-originals/

[Records and Tuples]: https://github.com/tc39/proposal-record-tuple
[JSON Modules]: https://github.com/tc39/proposal-json-modules

[Operation]: https://heycam.github.io/webidl/#dfn-create-operation-function
[Setter]: https://heycam.github.io/webidl/#dfn-attribute-setter
[Getter]: https://heycam.github.io/webidl/#dfn-attribute-getter
[ECMAScript Binding]: https://heycam.github.io/webidl/#ecmascript-binding
[Numeric Types]: https://heycam.github.io/webidl/#dfn-numeric-type
[Dictionary]: https://heycam.github.io/webidl/#idl-dictionaries
[Callback]: https://heycam.github.io/webidl/#idl-callback-function
[Sequence]: https://heycam.github.io/webidl/#idl-sequence
[Record]: https://heycam.github.io/webidl/#idl-record
[Enumeration]: https://heycam.github.io/webidl/#idl-enumeration
[Interface]: https://heycam.github.io/webidl/#idl-interfaces
[Union]: https://heycam.github.io/webidl/#idl-union
[Nullable]: https://heycam.github.io/webidl/#idl-nullable-type
[Annotated]: https://heycam.github.io/webidl/#idl-annotated-types
[Inner Type]: https://heycam.github.io/webidl/#annotated-types-inner-type
[Typed Array View]: https://heycam.github.io/webidl/#dfn-typed-array-type
[Frozen array]: https://heycam.github.io/webidl/#idl-frozen-array
[ES Union]: https://heycam.github.io/webidl/#es-union
[`boolean`]: https://heycam.github.io/webidl/#idl-boolean
[`ArrayBufferView`]: https://heycam.github.io/webidl/#ArrayBufferView
[`ArrayBuffer`]: https://heycam.github.io/webidl/#idl-ArrayBuffer
[`BufferSource`]: https://heycam.github.io/webidl/#BufferSource
[`any`]: https://heycam.github.io/webidl/#idl-any
[`void`]: https://heycam.github.io/webidl/#idl-void
[`object`]: https://heycam.github.io/webidl/#idl-object
[`symbol`]: https://heycam.github.io/webidl/#idl-symbol
[`Promise`]: https://heycam.github.io/webidl/#idl-promise
[`DOMString`]: https://heycam.github.io/webidl/#idl-DOMString
[`ByteString`]: https://heycam.github.io/webidl/#idl-ByteString
[`USVString`]: https://heycam.github.io/webidl/#idl-USVString
[`DOMString`-to-`USVString`]: https://infra.spec.whatwg.org/#javascript-string-convert

[`TextDecoder.decode`]: https://encoding.spec.whatwg.org/#dom-textdecoder-decode
[Unicode Scalar Value]: https://unicode.org/glossary/#unicode_scalar_value
[Code Point]: https://unicode.org/glossary/#code_point
[Surrogate]: https://unicode.org/glossary/#surrogate_code_point
[UTF8-Everywhere]: http://utf8everywhere.org/
[Potentially ill-formed UTF-16]: http://simonsapin.github.io/wtf-8/#ill-formed-utf-16
[WTF-16]: http://simonsapin.github.io/wtf-8/#wtf-16

[Tagged Unions]: https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-0.html#tagged-union-types 
[User-Defined Type Guards]: https://www.typescriptlang.org/docs/handbook/advanced-types.html#user-defined-type-guards
[Haskell uses thunks]: https://en.wikibooks.org/wiki/Haskell/Laziness
[Native Dynamic Linking]: https://en.wikipedia.org/wiki/Dynamic_linker
[Affine]: https://en.wikipedia.org/wiki/Substructural_type_system#Affine_type_systems
[Eager Evaluation]: https://en.wikipedia.org/wiki/Eager_evaluation
[Lazy Evaluation]: https://en.wikipedia.org/wiki/Lazy_evaluation
[SSA]: https://en.wikipedia.org/wiki/Static_single_assignment_form
[Reaching Definitions]: https://en.wikipedia.org/wiki/Reaching_definition
[UTF-8]: https://en.wikipedia.org/wiki/UTF-8
[UTF-32]: https://en.wikipedia.org/wiki/UTF-32
[Latin-1]: https://en.wikipedia.org/wiki/ISO/IEC_8859-1
[ASCII]: https://en.wikipedia.org/wiki/ASCII
[Amortized O(n)]: https://en.wikipedia.org/wiki/Dynamic_array#Geometric_expansion_and_amortized_cost
[Synchronous IPC]: https://en.wikipedia.org/wiki/Microkernel#Inter-process_communication
