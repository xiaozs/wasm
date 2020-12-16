# 导入/导出可变全局变量提案

本页描述了导入/导出可变全局变量提案。

## 原理

如果没有导入/导出可变全局变量，在提供可动态链接的可变的线程局部变量时会很不便，例如C++栈指针(SP).

下面的例子使用了SP，但对于其他线程局部变量，也可以使用类似的参数(又或者时TLS指针)。

### 例子: 通过单个代理动态链接SP

假设我们拥有两个动态链接的模块，`m1`和`m2`。
它们都使用了C++栈指针。在MVP中，我们没有办法导入/导出可变全局变量。
在单代理程序中,我们可以把栈指针保存在线性内存中以达到目的:

```
(module $m1
  (memory (export "memory") 1)
  ;; 地址0处为栈指针地址
  ;; 栈底在0x100
  (data (i32.const 0) "\00\01\00\00)

  (func
    ;; SP = SP + 64
    (i32.store
      (i32.const 0)  ;; SP
      (i32.add
        (i32.load (i32.const 0))  ;; SP
        (i32.const 64)))
    ...
  )
  ...
)

(module $m2
  (import "env" "memory" (memory 1))
  ...
)
```

然后模块实例化:

```
WebAssembly.instantiate(m1Bytes, {}).then(
    ({instance}) => {
        let imports = {env: {memory: instance.exports.memory}};
        WebAssembly.instantiate(m2Bytes, imports).then(...);
    });
```

在线性内存是共享的时候，这个例子不能运行，因为我要要给每一个栈指针分配一个不同的地址(每个代理都要一个)。

### 办法 1: 每个模块一个不可变全局变量

一个办法是给每个模块一个不可变全局变量来存储栈指针地址:

```
(module $m1
  (import "env" "spAddr" (global $spAddr i32))
  (memory (export "memory") 1)

  (data (i32.const 0) "\00\01\00\00")  ;; 代理0的栈指针
  (data (i32.const 4) "\00\02\00\00")  ;; 代理1的栈指针

  (func
    ;; SP = SP + 64
    (i32.store
      (get_global $spAddr)
      (i32.add
        (i32.load (get_global $spAddr))
        (i32.const 64)))
    ...
  )
  ...
)

(module $m2
  (import "env" "memory" (memory 1))
  (import "env" "spAddr" (global $spAddr i32))
  ...
)
```

然后通过以下代码进行实例化:

```
let agentIndex = ...;  // 0或1，取决于是主线程还是子线程
let spAddrs = [0x0, 0x4];
let imports = {env: {spAddr: spAddrs[agentIndex]}};
WebAssembly.instantiate(m1Bytes, imports).then(
    ({instance}) => {
        let imports = {env: {
          memory: instance.exports.memory,
          spAddr: spAddrs[agentIndex],
        }};
        WebAssembly.instantiate(m2Bytes, imports).then(...);
    });
```

这个原则能拓展到其他的线程局部变量，通过改变spAddr的指向，指到代理的TLS的起始位置。

这能跑，但有些缺点:

* 每次访问SP时，都要先读取全局变量
* SP是存在线性内存里面的，所以它能轻易被其他线程修改

### 办法 2: 使用内部可变全局变量 w/ Shadow in Linear Memory

为了节省访问SP的损耗，我们可以把SP保存在一个可变的全局变量里面。
为了能让其穿过模块的边界，我们把通过调用者把SP填充到线性内存里然后通过被调用这加载SP。

这可被优化为仅在必要时填充SP(尽管这里被间接函数调用复杂化了)，
但为了简单起见，这个例子只会显示函数调用之前的填充和函数入口处的加载:

```
(module $m1
  (import "env" "shadowSpAddr" (global $shadowSpAddr i32))
  (memory (export "memory") 1)
  (global $sp (mut i32) (i32.const 0))

  (data (i32.const 0) "\00\01\00\00")  ;; 代理0的影子SP
  (data (i32.const 4) "\00\02\00\00")  ;; 代理1的影子SP

  (func
    ;; 加载影子SP
    (set_global $sp (i32.load (get_global $shadowSpAddr)))

    ;; SP = SP + 64
    (set_global $sp (i32.add (get_global $sp) (i32.const 64)))
    ...
    ;; 调用方法，填充SP
    (i32.store (get_global $shadowSpAddr) (get_global $sp))
  )
  ...
)

(module $m2
  (import "env" "memory" (memory 1))
  (import "env" "spAddr" (global $shadowSpAddr i32))

  (global $sp (mut i32) (i32.const 0))

  (func
    ;; 加载影子SP
    (set_global $sp (i32.load (get_global $shadowSpAddr)))
    ...
  )
  ...
)
```

这些模块的实例化会和上面的办法1相同:

```
let agentIndex = ...;  // 0或1，取决于是主线程还是子线程
let shadowSpAddrs = [0x0, 0x4];
let imports = {env: {shadowSpAddr: shadowSpAddrs[agentIndex]}};
WebAssembly.instantiate(m1Bytes, imports).then(
    ({instance}) => {
        let imports = {env: {
          memory: instance.exports.memory,
          shadowSpAddr: shadowSpAddrs[agentIndex],
        }};
        WebAssembly.instantiate(m2Bytes, imports).then(...);
    });
```

这个方案可以扩展到其他的线程局部变量，但是每个函数调用都有额外消耗。这可能只适合那些经常使用的线程局部变量。对于其他案例，最好直接使用阴影值。

这个办法有以下缺点:

* 多数情况下都只是办法1的(潜在)优化方案
* 所有的load/store SP的函数调用都必须要在函数入口处填充SP
* 额外的线程局部变量也必须用同样的方法填充/加载，以获得同样的好处
* SP还是在线性内存里，所以它能轻易地被其他的线程修改

### 办法 3: 修改函数签名，将SP当成参数传递

于其将SP填充到线性缓存，不如将SP当成参数传递。 
因为我们不知道哪个导入的函数会使用SP，所以我们必须修改每一个导出的函数。

SP最终会被保存到可变全局变量中，但是会在函数入口的参数中被加载。
这只是一个优化;
我们能把SP传递给所有函数，但是只需要在被导出的函数上做:

```
(module $m1
  (memory (export "memory") 1)
  (global $sp (mut i32) (i32.const 0))

  (func $exported (export "exported") (param $sp i32)
    ;; 从参数中加载SP
    (set_global $sp (get_local $sp))

    ;; SP = SP + 64
    (set_global $sp (i32.add (get_global $sp) (i32.const 64)))
    ...
  )
  ...
)

(module $m2
  (import "env" "memory" (memory 1))
  (import "env" "exported" (func $exported (param $sp i32)))

  (global $sp (mut i32) (i32.const 0))

  (func $internal
    ;; 因为这个函数是内部的，所以不需要加载SP

    ;; SP = SP + 4
    (set_global $sp (i32.add (get_global $sp) (i32.const 4)))

    (call $exported (get_global $sp))
  )
)
```

模块初始化如下:

```
WebAssembly.instantiate(m1Bytes, {}).then(
    ({instance}) => {
        let imports = {env: {
          memory: instance.exports.memory,
          exported: instance.exports.exported,
        }};
        WebAssembly.instantiate(m2Bytes, imports).then(...);
    });
```

但是现在JavaScript代码必须保存一个传输到导出的函数的全局SP。
除此以外，任何回调JavaScript的函数必须更新此SP:

```
let sp = 0x200;

function importedFunction(newSp) {
  sp = newSp;
  ...
}

m1.exports.exported(sp);
```

这个方法的缺点是所有导出的函数必须有一个额外的参数给线程局部变量。
这个方法可以扩展到其他的线程局部变量，但是马上就会变得非常臃肿。

### 计划方案: 导出/导入可变全局变量

在MVP里面，可变全局变量不能被导出或导入。
如果我们放开这个限制，我们就能够为线程局部变量提供一个更好的解决方案:

```
(module $m1
  (import "env" "sp" (global $sp (mut i32)))
  (memory (export "memory") 1)

  (func
    ;; SP = SP + 64
    (set_global $sp (i32.add (get_global $sp) (i32.const 64)))
    ...
  )
  ...
)

(module $m2
  (import "env" "memory" (memory 1))
  (import "env" "sp" (global $sp (mut i32)))
  ...

  (func
    ;; SP = SP + 4
    (set_global $sp (i32.add (get_global $sp (i32.const 4))))
  )
)
```

模块初始化如下:

```
let agentSp = new WebAssembly.Global({type: 'i32', mutable: true}, 0x100);

let imports = {env: {sp: agentSp}};
WebAssembly.instantiate(m1Bytes, {}).then(
    ({instance}) => {
        let imports = {env: {
          memory: instance.exports.memory,
          sp: agentSp,
        }};
        WebAssembly.instantiate(m2Bytes, imports).then(...);
    });
```

JavaScript宿主只需给每个代理提供一个SP，然后在动态链接模块间共享它，就无需额外存储SP或修改函数签名。

类似地，如果导入的JavaScript函数想要在栈中分配内存，他能修改全局值:

```

let agentSp = ...;  // 同上

function importedFunction() {
  let addr = agentSp.value;
  // 在栈上分配8字节
  agentSp.value += 8;

  // 在地址处填充数据...
  ...

  // 调用回WebAssembly，传递栈里填充的数据
  m1.exports.anotherFunction(addr);
  ...
}
```

## 导出/导入可变全局变量

导出/导入全局变量现在可以修改了。
在Web binding里，导入全局变量现在的类型是`WebAssembly.Global`，而不是转换为JavaScript数字。

对[代理][agent]来说全局变量是本地的，且不能在代理间共享。
因此，全局变量也能用作[thread-local storage](https://en.wikipedia.org/wiki/Thread-local_storage).

## `WebAssembly.Global` 对象

`WebAssembly.Global`对象包含一个`global`值，这个值能同时被多个`Instance`对象引用。
每个`Global`对象拥有两个内部插槽:

* \[\[Global\]\]: a [`global instance`]
* \[\[GlobalType\]\]: a [`global_type`]

#### `WebAssembly.Global`构造器

`WebAssembly.Global`构造器的签名如下:

```
new Global(globalDescriptor, value=0)
```

如果NewTarget是`undefined`，将会抛出一个[`TypeError`]异常
(例如，这个构造函数调用时不能省略`new`).

如果`Type(globalDescriptor)`不是对象，将会抛出一个[`TypeError`]异常

[`ToString`] ([`Get`] (`globalDescriptor`，`"value"`))应为`typeName`

如果`typeName`不是`"i32"`，`"f32"`，或者`"f64"`其中之一，将会抛出一个[`TypeError`]异常

[`value type`]应为`type`:

* 如果`typeName`是`"i32"`，让`type`为`i32`.
* 如果`typeName`是`"f32"`，让`type`为`f32`.
* 如果`typeName`是`"f64"`，让`type`为`f64`.

[`ToBoolean`] ([`Get`] (`globalDescriptor`，`"mutable"`))应为`mutable`

如果`mutable`为真，让`mut`为`var`；如果`mutable`为假，让`mut`为`const`。

设`v`为能转换成`type`的`value`。

返回`CreateGlobalObject`(`v`，`mut`，`type`)的结果。

#### CreateGlobalObject

给出初始值`v`，可变性`m`，和类型`t`以生成`WebAssembly.Global`:

设`g`为一个新的[`global instance`]带有`value` `v`和`mut` `m`.
设`gt`为一个新的[`global_type`] 带有`mut` `m`和`type` `t`.

返回一个新的`WebAssembly.Global`，它有设为`g`的\[\[Global\]\]和设为`gt`的\[\[GlobalType\]\]。

#### `WebAssembly.Global.prototype.valueOf()` 方法

1. 如果\[\[GlobalType\]\]的`valtype`是`i64`，将会抛出一个[`TypeError`]异常。
1. 返回[`ToJSValue`] (\[\[Global\]\].`value`).

#### `WebAssembly.Global.prototype.value` 属性

这是一个访问器属性。[[Set]]访问器函数，通过`V`值调用,
执行步骤如果下:

1. 如果\[\[Global\]\]的`mut`是`const`，将会抛出一个[`TypeError`]异常。
1. 设`type`为\[\[GlobalType\]\].`valtype`。
1. 如果`type`是`i64`，将会抛出一个[`TypeError`]异常。
1. 设`value`为[`ToWebAssemblyValue`] (`V`)，它能转换为`type`。
1. 将\[\[Global\]\].`value`设置为`value`。

[[Get]]访问器函数执行步骤如果下:

1. 如果\[\[GlobalType\]\].`valtype`是`i64`，将会抛出一个[`TypeError`]异常。
1. 返回[`ToJSValue`] (\[\[Global\]\].`value`).

### `WebAssembly.Instance` 构造器

对每一个在`module.imports`中的[`import`] `i`:

1. ...
1. ...
1. ...
1. ...
1. 如果`i`是全局变量:
   1. 如果`Type(v)`是数字:
      1. 如果`i`的`global_type`是`i64`，将会抛出一个`WebAssembly.LinkError`异常。
      1. 设`globalinst`为[`global instance`]带[`ToWebAssemblyValue`] (`v`) 和 mut `i.mut`。
      1. 将`imports`加入到`globalinst`。
   1. 如果`Type(v)`是`WebAssembly.Global`，将`v.[[Global]]`追加到`imports`。
   1. 其他: 抛出一个`WebAssembly.LinkError`异常。

...

设`exports`是(字符串，JS值)键值对列表，其映射为`instance.exports`里的每个[`external`]值`e`:

1. ...
1. 如果`e`是一个[`global instance`] `g`带有[`global_type`] `gt`:
   1. 返回一个新的`WebAssembly.Global`带有设为`g`的\[\[Global\]\]和设为`gt`的\[\[GlobalType\]\]。

[agent]: Overview.md#agents-and-agent-clusters
[`external`]: https://github.com/WebAssembly/spec/blob/master/interpreter/spec/instance.ml#L24
[`Get`]: https://tc39.github.io/ecma262/#sec-get-o-p
[global]: https://github.com/WebAssembly/spec/blob/master/interpreter/spec/instance.ml#L15
[`global_type`]: https://webassembly.github.io/spec/syntax/types.html#global-types
[`global instance`]: http://webassembly.github.io/spec/execution/runtime.html#global-instances
[`import`]: https://github.com/WebAssembly/spec/blob/master/interpreter/spec/ast.ml#L168
[`ToBoolean`]: https://tc39.github.io/ecma262/#sec-toboolean
[`ToJSValue`]: https://github.com/WebAssembly/design/blob/master/JS.md#tojsvalue
[`ToString`]: https://tc39.github.io/ecma262/#sec-tostring
[`@@toStringTag`]: https://tc39.github.io/ecma262/#sec-well-known-symbols
[`ToWebAssemblyValue`]: https://github.com/WebAssembly/design/blob/master/JS.md#towebassemblyvalue
[`TypeError`]: https://tc39.github.io/ecma262/#sec-native-error-types-used-in-this-standard-typeerror
[`value type`]: https://github.com/WebAssembly/spec/blob/master/interpreter/spec/types.ml#L3
