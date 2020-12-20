# WebAssembly的引用类型

## 介绍

动机:

* 与宿主环境更简单更高效的互操作 (详见[接口类型提案](https://github.com/WebAssembly/interface-types/blob/master/proposals/interface-types/Explainer.md))
  - 允许使用类型`externref`代表宿主引用 (详见[这里](https://github.com/WebAssembly/interface-types/issues/9))
  - 无需遍历表、分配插槽和维护边界处的索引双射

* Wasm内部表格的基本操作
  - 通过将表重新调整为不透明数据类型的通用内存，允许表现的数据结构包含引用
  - 允许在wasm内操控函数表
  - 增加[大内存操作提案](https://github.com/WebAssembly/bulk-memory-operations/blob/master/proposals/bulk-memory-operations/Overview.md)中缺失的指令

* 为以后的添加做好准备：

  - 带类型函数引用(详见[下面](#带类型函数引用))
  - 异常引用(详见[异常处理提案](https://github.com/WebAssembly/exception-handling/blob/master/proposals/Exceptions.md)和[这里](https://github.com/WebAssembly/interface-types/issues/10))
  - 到GC的平滑过渡(详见[GC提案](https://github.com/WebAssembly/gc/blob/master/proposals/gc/Overview.md))

还有更多重要部分!

概述:
* 添加新类型`externref`，它可以同时作为值类型和表元素类型使用。

* 允许`funcref`作为值类型。

* 介绍get/set表插槽的指令。

* 添加缺失的表size，grow，fill指令。

* 允许多个表。

注意:

* 这个扩展本身并不意味着GC，仅当宿主引用是GC指针时，才和GC有关!

* 引用类型是*不透明的*，例如，它们的值是抽象的且不能存储在线性内存里。
  表也是。


## 语言扩展

类型扩展:

* 引入`funcref`和`externref`作为*引用类型*.
  - `reftype ::= funcref | externref`

* 值类型(局部变量，全局变量，函数的参数和结果)现在分为数字类型和引用类型。
  - `numtype ::= i32 | i64 | f32 | f64`
  - `valtype ::= <numtype> | <reftype>`
  - 引用类型的局部变量被初始化为`null`

* (表的)元素类型与引用类型相当。
  - `elemtype ::= <reftype>`


新/扩展指令:

* `select`指令现在可选一个值类型立即数。只有标记过的`select`能与引用类型共用。
  - `select : [t t i32] -> [t]`
    - 当且仅当`t`是`numtype`
  - `select t : [t t i32] -> [t]`

* 新指令`ref.null`为空引用常量。
  - `ref.null rt : [] -> [rtref]`
    - 当且仅当`rt = func`或`rt = extern`
  - 允许常量表达式

* 新指令`ref.is_null`检查是否为null。
  - `ref.is_null rt : [rtref] -> [i32]`
    - 当且仅当`rt = func` or `rt = extern`

* 新指令`ref.func`新增对指定函数的引用。
  - `ref.func $x : [] -> [funcref]`
    - 当且仅当`$x : func $t`
  - 允许常量表达式
  - 注意: 这个指令的返回类型可能被未来的提案改进(例如，`[(ref $t)]`)

* 新指令`table.get`和`table.set`用于访问表。
  - `table.get $x : [i32] -> [t]`
    - 当且仅当`$x : table t`
  - `table.set $x : [i32 t] -> []`
    - 当且仅当`$x : table t`

* 新指令`table.size`和`table.grow`操作表的大小。
  - `table.size $x : [] -> [i32]`
    - 当且仅当`$x : table t`
  - `table.grow $x : [t i32] -> [i32]`
    - 当且仅当`$x : table t`
  - `table.grow`的第一个操作数是一个初始化值(用于适配未来对类型系统的扩展，例如非null引用)

* 新指令`table.fill`将表的一定范围填充为某值。
  - `table.fill $x : [i32 t i32] -> []`
    - 当且仅当`$x : table t`
  - 第一个操作数是范围的开始索引，第三个操作数是他的长度(类似于`memory.fill`)
  - 当range+length > 表的size时，发生陷入(类似于`memory.fill`)

* `table.init`指令需要另外的表索引作为立即数。
  - `table.init $x $y : [i32 i32 i32] -> []`
    - 当且仅当`$x : table t`
    - 且 `$y : elem t'`
    - 且 `t' <: t`

* `table.copy`指令需要另外的两个表索引作为立即数。
  - `table.copy $x $y : [i32 i32 i32] -> []`
    - 当且仅当`$x : table t`
    - 且 `$y : table t'`
    - 且 `t' <: t`

* `call_indirect`指令需要另外的表索引作为立即数。
  - `call_indirect $x (type $t) : [t1* i32] -> [t2*]`
    - 当且仅当`$t = [t1*] -> [t2*]`
    - 且 `$x : table t'`
    - 且 `t' <: funcref`

* 所有指令，表索引都可以省略为0

注意:
- 在二进制格式中，已经为表索引保留了空间。
- 为了后向兼容，文本格式中的所有表索引都可省略，省略值为0(对于`table.copy`，两个索引需要同时提供或同时省略)。


表扩展:

* 一个模块可能定义，引入，和导出多个表。
  - 一般来说，引入在索引空间最前面。
  - 这已经在二进制格式中做了预留。

* 元素节需要一个表索引作为立即数，以指示其操控的表。
  - 在二进制格式中，已经为表索引保留了空间。
  - F为了后向兼容，文本格式中的所有表索引都可省略，省略值为0。


API扩展:

* 任何JS值都能当成`externref`传给Wasm函数，保存在一个全局变量，或一个表里。

* 任何Wasm导出函数对象或`null`都能当成`funcref`传给Wasm函数，保存在一个全局变量，或一个表里。


## 未来可能有的扩展


### 子类型

动机:

* 支持变量扩展(下述)。

新增:

* 在引用类型之间引入一个简单的子类型关系。
  - 下列规则的自反传递闭包
  - 对所有引用类型`t`有`t <: anyref`


### 引用的相等性

动机:

* 允许引用间通过id对比。
* 但是，并非所有引用类型都能对比，因为这会导致实现细节以不确定的方式被观察到(例如，宿主的JavaScript字符串)。


新增:

* 新增`eqref`作为可对比引用
  - `reftype ::= ... | eqref`
* 它是`anyref`的值类型
  - `eqref < anyref`
* 新增`ref.eq`指令
  - `ref.eq : [eqref eqref] -> [i32]`

API改变:

* 任何JS对象(非原始值)或symbol或`null`都能当成`eqref`传给Wasm函数，保存在一个全局变量，或一个表里。


问题:

* 与导入/导入类型的交互: 现在需要区分相等类型和不相等类型吗?

* 同样的，下面的`WebAssembly.Type`的JS API需要进行区分。


### 带类型函数引用

详见[带类型函数引用提案](https://github.com/WebAssembly/function-references/blob/master/proposals/function-references/Overview.md)

动机:

* 允许不经过表和动态类型检查就可以直接表示函数指针。
* 使函数可以轻松传递到其他模块。

新增:

* 新增`(ref $t)`作为引用类型
  - `reftype ::= ... | ref <typeidx>`
* 完善`(ref.func $f)`指令
  - `ref.func $f : [] -> (ref $t)` 当且仅当`$f : $t`
* 新增`(call_ref)`指令
  - `call_ref : [ts1 (ref $t)] -> [ts2]` 当且仅当`$t = [ts1] -> [ts2]`
* 引入子类型`ref <functype> < funcref`
* 具体引用类型和通用引用类型之间的微妙关系
  - `ref $t < anyref`
  - `ref <functype> < funcref`
  - 注意: 引用类型不一定是`eqref`的子类型，包括函数

* 带类型函数引用不能为null!

* `table.grow`指令(详见[大内存操作提案](https://github.com/WebAssembly/bulk-memory-operations/blob/master/proposals/bulk-memory-operations/Overview.md))需要一个初始化参数。

* 同样地`WebAssembly.Table#grow`需要新增一个初始化参数。
  - 为了后向兼容，默认为`null`


### 类型导入/导出

动机:

* 允许宿主(或Wasm模块)区分不同的引用类型。

新增:

* 新增`(type)`外部类型，以允许类型的导入导出
  - `externtype ::= ... | type`
  - `(ref $t)`现在可以表示抽象类型或函数引用
  - 导入的类型具有从0开始的索引。
  - 在二进制格式中预留字节，以便以后进行优化

* 在类型段中新增抽象类型定义
  - `deftype ::= <functype> | new`
  - 新增唯一抽象类型

* 在JS API中新增`WebAssembly.Type`类
  - 构造函数`new WebAssembly.Type(name)`新增唯一抽象类型

* 子类型`ref <abstype>` < `anyref`


问题:

* 我们要不要限制导入的顺序，以使节的依赖分层? 类型的导入导出应该分为不同的段吗?

* 我们需要让局部变量或别的什么支持可空`(optref $t)`类型吗? 还是说用一个`(nullable T)`类型?
  - 不清楚如何集成`nullable`构造函数。它仅支持(非空)引用类型作为参数吗?
    这个需要类型系统吗? 
    `anyref`应该和`(nullable anyref)`不同吗，还是不允许后者? `funcref`呢?
  - 语义上，想一下`(nullable T)`作为`T | nullref`能回答这些问题，但在wasm中我们不能支持任意联合。

* 我们是否应该添加`(new)`定义类型，以使Wasm模块也能定义新类型?

* `new`定义类型和`WebAssembly.Type`构造函数是否需要使用一个"comparable"标志来控制是否可以对类型进行比较?

* JS API允许在新类型之间指定子类型吗?


### 子类型转换

动机:

* 允许使用`anyref`作为顶层类型来实现泛型。

新增:

* 新增`cast`指令以检查其操作数是否能转换为子类型，如是，则转换; 否则，进入else分支。
  - `cast <resulttype> <reftype1> <reftype2> <instr1>* else <instr2>* end: [<reftypet1>] -> <resulttype>`
    - 当且仅当`<reftype2> < <reftype1>`
    - 且`<instr1>* : [<reftype2>] -> <resulttype>`
    - 且`<instr2>* : [<reftype1>] -> <resulttype>`
  - 以后可以泛化为非引用类型?

注意:

* 可以分解`call_indirect` (多值提案):
  - `(call_indirect $x (type $t))`展开为`(table.get $x) (cast $t anyref (ref $t) (then (call_ref (ref $t))) (else (unreachable)))`


### GC类型

详见[GC提案](https://github.com/WebAssembly/gc/blob/master/proposals/gc/Overview.md).


### 未来可能有扩展

* 引入指向表，内存，或全局变量的引用类型。
  - `deftype ::= ... | global <globaltype> | table <tabletype> | memory <memtype>`
  - `ref.global $g : [] -> (ref $t)` 当且仅当`$g : $t`
  - `ref.table $x : [] -> (ref $t)` 当且仅当`$x : $t`
  - `ref.mem $m : [] -> (ref $t)` 当且仅当`$m : $t`
  - 遍历一等表，内存，全局变量
  - 需要复制所有相应的指令

* 允许所有值类型作为元素类型使用。
  - `deftype := ... | globaltype | tabletype | memtype`
  - 需要将元素类型与值类型统一
