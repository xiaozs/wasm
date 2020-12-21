# js类型

## 动机

Wasm是类型化的，且它的[类型](https://webassembly.github.io/spec/core/syntax/types.html)携带的信息对于通过JS API与Wasm模块和对象交互的宿主来说是非常有用和重要的。
例如，对于导入导出的类型描述，包括内存和表的大小限制，或者是全局变量的可变性。
通过JS查询这一类的信息的需求已经出现过好几次了，例如，下面的问题:

* [WebAssembly/design#1046](https://github.com/WebAssembly/design/issues/1046)
* [WebAssembly/threads#87](https://github.com/WebAssembly/threads/issues/87)
* 其他?

例如，需要为模块编写一个JS托管的链接器或适配器机制。

本提案以系统的方式向JS API添加各项功能。


## 概述

一言以蔽之，本提案包含三部分:

* 将Wasm类型的表示形式定义为JS对象

* 用`type`方法扩展API类以检索底层Wasm对象的类型

* 在最后，`WebAssembly.Function`作为一个新类引入，其为JavaScript的`Function`的子类，用于表示Wasm导出的函数

最后一个同时提供一个构造函数用于通过标准的JS函数显示创建Wasm导出函数。


## 类型表示

所有的Wasm类型都能通过一个简单的语法定义。
这个语法能使用一种直接且可拓展的方式映射到JSON-style JS对象上。
例如，使用TypeScript-style类型定义：

```TypeScript
type ValueType = "i32" | "i64" | "f32" | "f64"
type ElemType = "anyfunc"
type GlobalType = {value: ValueType, mutable: boolean}
type MemoryType = {limits: Limits}
type TableType = {limits: Limits, element: ElemType}
type Limits = {min: number, max?: number}  // see below
type FunctionType = {parameters: ValueType[], results: ValueType[]}
type ExternType =
  {kind: "function", type: FunctionType} |
  {kind: "memory",   type: MemoryType} |
  {kind: "table",    type: TableType} |
  {kind: "global",   type: GlobalType}
```

给出已存在的JS API，我们可以将API的现有描述符接口重新调整用途（并重命名）为类型，并为函数和外部类型添加缺少的描述符接口。
与上述方法的唯一区别是，限制是被内联到内存和表类型中的(且有更长的名字)。

更具体地说:

* 重命名[ImportExportKind](https://webassembly.github.io/spec/js-api/index.html#modules)为ExternKind

* 重命名[MemoryDescriptor](https://webassembly.github.io/spec/js-api/index.html#memories)为MemoryType

* 重命名[TableDescriptor](https://webassembly.github.io/spec/js-api/index.html#tables)为TableType

* 重命名[TableKind](https://webassembly.github.io/spec/js-api/index.html#tables)为ElemType

  注意: 这些规范内部定义的重命名纯粹是表面的，不会影响可观察的API。

* 为函数类型添加字典:
  ```WebIDL
  dictionary FunctionType {
    required sequence<ValueType> parameters;
    required sequence<ValueType> results;
  };
  ```

* 为外部类型添加词典:
  ```WebIDL
  dictionary ExternType {
    required ExternKind kind;
    required (FunctionType or TableType or MemoryType or GlobalType) type;
  };
  ```
  作为附加约束，`type`字段的内容必须与`kind`字段的内容匹配。

### 对大小限制的命名

还有一个问题。
MemoryDescriptor和TableDescriptor的当前定义将表示最小大小的属性命名为`initial`。
这对于将其用作各个构造函数的参数是有意义的，但另一方面: 对于更一般的类型用法，此属性只反映当前或最小的所需大小，可能是在增长后。特别是对于导入，其类型中的最小大小可能大于匹配该导入的对象的初始大小（并小于其当前大小）。

因此，当前用于表和内存构造函数的描述符不能正确地表示类型的概念。
另一方面，构造函数可以直接理解反射函数传递的类型(详见下面的[例子](#例子))。

因此，我提议允许将`minimum`和`initial`作为该字段的名称。
也就是说，它们都是接口的可选字段，但有两个中必须有一个。
但是，这种约束不能直接在WebIDL中实现，而是需要使用辅助接口，如下所示：

* 在[MemoryDescriptor/Type](https://webassembly.github.io/spec/js-api/index.html#memories)和 [TableDescriptor/Type](https://webassembly.github.io/spec/js-api/index.html#tables)中，将`initial`命名为`minimum`

* 将内存构造函数的参数类型更改为`(MemoryType or InitialMemoryType)`，其中InitialMemoryType对应当前的MemoryDescriptor

* 将表构造函数的参数类型更改为`(TableType or InitialTableType)`，其中InitialTableType对应当前的TableDescriptor

注意: 最后两点只是一个向后兼容性度量，使构造函数能够继续将`initial`而不是`minimum`理解为字段名。


## 对API函数的扩展

可以通过向API中添加以下方法来查询类型。

* 令[ModuleExportDescriptor](https://webassembly.github.io/spec/js-api/index.html#modules)和[ModuleImportDescriptor](https://webassembly.github.io/spec/js-api/index.html#modules)继承自`ExternType`:
  ```WebIDL
  dictionary ModuleExportDescriptor : ExternType { ... };
  dictionary ModuleImportDescriptor : ExternType { ... };
  ```
  `kind`字段将从这两个定义中删除，转而继承自新增的`type`字段。

* 用属性扩展接口[Memory](https://webassembly.github.io/spec/js-api/index.html#memories)
  ```WebIDL
  MemoryType type();
  ```

* 用属性扩展接口[Table](https://webassembly.github.io/spec/js-api/index.html#tables)
  ```WebIDL
  TableType type();
  ```

* 用属性扩展接口[Global](https://github.com/WebAssembly/mutable-global/blob/master/proposals/mutable-global/Overview.md#webassemblyglobal-objects)
  ```WebIDL
  GlobalType type();
  ```

* 重载构造函数[Memory](https://webassembly.github.io/spec/js-api/index.html#memories)
  ```WebIDL
  Constructor(MemoryType or InitialMemoryType type)
  ```

* 重载构造函数[Table](https://webassembly.github.io/spec/js-api/index.html#tables)
  ```WebIDL
  Constructor(TableType or InitialTableType type)
  ```

* 调整构造函数[Global](https://github.com/WebAssembly/mutable-global/blob/master/proposals/mutable-global/Overview.md#webassemblyglobal-objects)以接受GlobalType和它的初始化值:
  ```WebIDL
  Constructor(GlobalType type, any value)
  ```


## 对`WebAssembly.Function`的新增

当前，Wasm[导出函数](https://webassembly.github.io/spec/js-api/index.html#exported-function-exotic-objects)没有被分配一个特殊的类。
相反，它们只是具有JavaScript的内置类`Function`。

提案的这一部分对Wasm导出的函数进行了改进，使其具有合适的子类，具有以下优点:

* `type`属性可以添加到这个类中，以与上面提出的其他类型反射属性一致的方式反映Wasm函数的类型。

* 此类的构造函数可用于显式构造Wasm导出的函数，弥补了当前API中的一个空白，为JavaScript提供一种将普通JS函数放入表中的方法(同时能于Wasm内部做同样的操作)。

* Wasm导出的函数可以通过`instanceof`检查加以识别。

具体来说，变化如下：

* 引入新类型`WebAssembly.Function`，其为`Function`的子类型，如下
  ```WebIDL
  [LegacyNamespace=WebAssembly, Constructor(FunctionType type, function func), Exposed=(Window,Worker,Worklet)]
  interface Function : global.Function {
    FunctionType type();
  };
  ```

* 所有导出的函数属于类`WebAssembly.Function`。

* 由`WebAssembly.Function`构造的函数和其他从模块中导出的函数无异。
更具体地说，它们有一个[[FunctionAddress]]内部插槽，它将它们标识为导出函数。


## 例子

以下函数需要用到`WebAssembly.Module并创建一个合适的模拟导入对象来实例化它:
```JavaScript
function mockImports(module) {
  let mock = {};
  for (let imp of WebAssembly.Module.imports(module)) {
    let value;
    switch (imp.kind) {
      case "table":
        value = new WebAssembly.Table(imp.type);
        break;
      case "memory":
        value = new WebAssembly.Memory(imp.type);
        break;
      case "global":
        value = new WebAssembly.Global(imp.type, undefined);
        break;
      case "function":
        value = () => { throw "unimplemented" };
        break;
    }
    if (! (imp.module in mock)) mock[imp.module] = {};
    mock[imp.module][imp.name] = value;
  }
  return mock;
}

let module = ...;
let instance = WebAssembly.instantiate(module, mockImports(module));
```

下面的示例演示如何使用`WebAssembly.Function``构造函数，以使用多种不同类型将JavaScript函数添加到表中：
```JavaScript
function print(...args) {
  for (let x of args) console.log(x + "\n")
}

let table = new Table({element: "anyfunc", minimum: 10});

let print_i32 = new WebAssembly.Function({parameters: ["i32"], results: []}, print);
table.set(0, print_i32);
let print_f64 = new WebAssembly.Function({parameters: ["f64"], results: []}, print);
table.set(1, print_f64);
let print_i32_i32 = new WebAssembly.Function({parameters: ["i32", "i32"], results: []}, print);
table.set(2, print_i32_i32);
```
