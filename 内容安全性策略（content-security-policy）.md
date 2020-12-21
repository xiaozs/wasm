# WebAssembly内容安全性策略

本提案试图统一现有的WebAssembly实现的内容安全性策略(CSP)。

同时试图扩展CSP，使其更好地支持WebAssembly用例。

## 背景: CSP线程模型及用例

本节描述CSP要对抗的攻击。
WebAssembly对CSP的处理应该符合这个线程模型。

CSP，广义上，允许开发者控制什么资源能被加载到网站上。
这些资源包括图片，音频，视频，和脚本。
加载不受信任的资源可能会导致各种不良结果。
恶意脚本可能会从站点窃取数据。
图像可能显示误导或不正确的信息。
获取资源会将有关用户的信息泄露给不受信任的第三方。

虽然这些威胁可以通过其他方式得到保护，但CSP允许一小部分安全专家在一个地方定义一个全面的策略，以防止意外地将不可信的资源加载到站点中。

### 超出范围的威胁

* **浏览器中的Bug**。我们假设图像解码器、脚本编译器等的正确实现。
  CSP不能防止恶意输入，
  例如，触发器缓冲区溢出。
* **资源枯竭**。脚本执行的计算占用内存和CPU时间，因此可能导致浏览器拒绝服务。
  防止这种情况发生是站点所有者使用CSP的原因之一，但是拒绝服务并不是CSP的首要考虑因素。 
  脚本是危险的，不是因为它们的资源消耗，而是因为其他可能引起的影响。


## WebAssembly与CSP

WebAssembly实例由代码（Wasm字节）和导入对象组成。
导入对象定义了实例的功能，因此定义了最坏情况下的安全行为。
具有空导入对象的实例不会造成任何影响，因此可以安全运行。
如果可以检查导入对象，那么创建实例并从不受信任的Wasm代码运行是安全的，
因为代码的行为将受到导入对象功能的限制。 
实际上，在JavaScript中审查导入对象是非常困难的；
很容易意外地授予对全局对象的访问权限。

CSP扭转了问题。
假设功能不受限制，开发人员愿意在他们的站点上运行什么代码？
因此，对于WebAssembly，CSP将用于定义哪些Wasm字节的源可以被信任实例化和运行。

### WebAssembly api及其风险概述

本节介绍WebAssembly提供的与内容安全策略相关的api。

执行WebAssembly有几个步骤。
首先是原始的WebAssembly字节，通常使用[fetch API](https://fetch.spec.whatwg.org/)加载。 
接下来，使用`WebAssembly.compile`或`WebAssembly.compileStreaming`成为`WebAssembly.Module`。
这个模块还不能执行，但是WebAssembly实现可以选择在这一步将WebAssembly代码转换成机器代码
（Chrome做这个，WebKit不做）。
最后，WebAssembly模块和一个_引入对象_组合，通过`WebAssembly.instantiate`生成一个`WebAssembly.Instance`对象。
导入对象大体上定义了结果实例的功能，可能包含一个`WebAssembly.Memory`，WebAssembly函数导入和间接函数调用表的绑定。
此时，WebAssembly代码实际上是可执行的，因为宿主代码可以调用通过实例的导出WebAssembly函数。

这些步骤提供了WebAssembly的核心API，但也提供了其他几种方法。
这些风险及其与CSP相关的风险总结如下。

[**`WebAssembly.validate`**](https://webassembly.github.io/spec/js-api/index.html#dom-webassembly-validate)
检查给定字节是否包含有效的WebAssembly程序。 
换句话说，它根据WebAssembly类型系统检查字节的语法是否正确和有效。

_风险:_ 没有。

[**`new WebAssembly.Module`**](https://webassembly.github.io/spec/js-api/index.html#dom-module-module)
通过WebAssembly字节同步创建`WebAssembly.Module`。
这是的一个同步版本的`WebAssembly.compile`。

_风险:_ 许多实现在这一步将生成机器代码，即使它还没有作为可执行代码公开给周围的程序。
通过利用实现中的另一个错误，此代码有可能被用来构建攻击。
这种风险显然超出了我们正在研究的威胁模型的范围。

[**`WebAssembly.compile`**](https://webassembly.github.io/spec/js-api/index.html#dom-webassembly-compile)
提供一个`Promise`，它将从提供的WebAssembly字节生成`WebAssembly.Module`。 
这是一个异步版本的`new WebAssembly.Module`。

_风险:_ 和`new WebAssembly.Module`相同。

[**`WebAssembly.compileStreaming`**](https://webassembly.github.io/spec/web-api/index.html#dom-webassembly-compilestreaming)
从`Response`对象提供的WebAssembly字节中生成一个`WebAssembly.Module`。

_风险:_ 和`new WebAssembly.Module`相同。

[**`WebAssembly.instantiate`**](https://webassembly.github.io/spec/js-api/index.html#dom-webassembly-instantiate)
接受WebAssembly字节或一个`WebAssembly.Module`和一个引入对象。
此函数返回一个`WebAssembly.Instance`，它允许执行WebAssembly代码。
如果提供的是WebAssembly字节，`instantiate`将首先执行`WebAssembly.compile`。

_风险:_ 将可执行代码加载到正在运行的程序中。
这个代码是有限制的，只能访问可从导入对象访问的对象。
此实例没有不受限制访问JavaScript全局对象的权限。

[**`WebAssembly.instantiateStreaming`**](https://webassembly.github.io/spec/web-api/index.html#dom-webassembly-instantiatestreaming)
接受一个`Response`包含WebAssembly字节和一个引入对象，执行`WebAssembly.compileStreaming` on these 后面的操作然后创建一个`WebAssembly.Instance`。

_风险:_ 和`WebAssembly.instantiate`相同。

### CSP推荐应用

CSP政策可用于限制`WebAssembly.Module`的构造。
给出CSP的威胁模型，通过网络下载wasm字节或通过字节新增`WebAssembly.Module`对象的操作应该遵守CSP限制。

实例化`WebAssembly.Module`对象被认为是安全的。
不像JavaScript的`eval`，WebAssembly基于功能: 实例只能访问作为导入显式提供给它的功能，而不能访问环境状态，如JavaScript全局对象。
保护JavaScript提供给WebAssembly模块的导入与WebAssembly的CSP指令是正交的。
因此实例化`WebAssembly.Module`对象不必受CSP限制。

## 当前实现的行为

如果没有指定内容安全策略，则所有实现当前都允许它们支持的所有WebAssembly操作。

如果指定了内容安全策略且`script-src`中包含了'unsafe-eval'指令，
则当前所有的实现都允许执行它们支持的所有WebAssembly操作。

如果指定了内容安全策略且`script-src`中不包含'unsafe-eval'指令，
则，根据不同的实现允许执行的WebAssembly操作各有不同。

操作 | Chrome | Safari | Firefox | Edge
--- | --- | --- | --- | --- 
WebAssembly.validate | yes | yes | yes | yes
new WebAssembly.Module | no | yes | yes | yes
WebAssembly.compile | no | yes | yes | yes
WebAssembly.compileStreaming | no | yes | yes | yes
new WebAssembly.Instance | ? | ? | ? | ?
WebAssembly.instantiate(WebAssembly.Module，...) | no | no | yes | yes
WebAssembly.instantiate(BufferSource，...) | no | no | yes | yes
WebAssembly.instantiateStreaming | no | no | yes | yes
new WebAssembly.CompileError | yes | yes | yes | yes
new WebAssembly.LinkError | yes | yes | yes | yes
new WebAssembly.Table | yes | no | yes | yes
new WebAssembly.Memory | yes | no | yes | yes

根据实现的不同，上述操作之一不被允许时引发的异常的类型也不同。
此表列出了每个实现的异常类型:

浏览器 | 不被允许时引发的异常
--- | --
Chrome | WebAssembly.CompileError: 此上下文中不允许Wasm代码生成
Safari | EvalError
Firefox | N/A
Edge | ??

下面是每个浏览器如何处理eval():

浏览器 | 不被允许时引发的异常
--- | --
Chrome | EvalError
Safari | EvalError
Firefox | 不被允许的脚本(无法捕获)
Edge | ??


## 现有行为的同质化建议

行动原则:

* 对允许的事情要保守。
* 允许在当前API表面执行那些不能和源绑定的操作(Chrome的行为)。
   * 允许内存和表对象，因为当编译/实例化是源代码绑定时它们与当前源绑定，没有允许明确的源的参数。
   * 不允许编译，因为如果存在注入攻击，它会被用于执行WebAssembly代码编译。
* 抛出EvalError (Safari的行为)，Chrome和Safari都对eval()这样做。
  注意: Firefox的行为更加保守，但这对其他人来说可能是个挑战，因为它对这个比对eval()更严格。

此表说明当指定了内容安全策略且`script-src`不包含'unsafe-eval'指令时，应允许哪些操作:

操作 | 结果
--- | ---
WebAssembly.validate | yes
new WebAssembly.Module | no
WebAssembly.compile | no
WebAssembly.compileStreaming | no
new WebAssembly.Instance | yes
WebAssembly.instantiate(WebAssembly.Module，...) | yes
WebAssembly.instantiate(BufferSource，...) | no
WebAssembly.instantiateStreaming | no
new WebAssembly.CompileError | yes
new WebAssembly.LinkError | yes
new WebAssembly.Table | yes
new WebAssembly.Memory | yes


## 建议的'wasm-unsafe-eval'指令

WebAssembly编译不太容易像JavaScript那样被欺骗。
此外，WebAssembly有一个显式指定的作用域，
进一步降低注入攻击的可能性。

虽然源绑定/已知哈希操作总是更安全，
在CSP策略中，存在一个机制以允许WebAssembly的内容是很有用的。
否则将不允许它，而不需要同时允许JavaScript eval（）。

注意: 提供一个允许JavaScript eval()而不使用WebAssembly的指令似乎不会立即有用，因此被有意忽略了。

我们提议:
* 每个支持'unsafe-eval'的指令都允许'wasm-unsafe-eval'(这是目前所有的指令，因为指令可以互相尊重)。
* 对于`script-src`指令(直接或者是引用的),
  将'wasm-unsafe-eval'解释为允许所有WebAssembly操作。
  (不允许JavaScript eval()).


## 建议的源绑定许可

为了使WebAssembly在CSP的精神下更加有用，
我们应该允许`Response`对象携带可信的源信息。
这会允许WebAssembly以一种自然的方式在CSP中进行编译和实例化。

建议的更改:
* 将被“强化”以允许它（或带有污点跟踪位的它）被信任携带关于获取响应的起源的信息。
* 如果WebAssembly编译/实例化请求是非内联JavaScript的脚本src请求，那么WebAssembly也将被允许。
   * 这适用于哈希或源白名单。
* 这个子提案只作用于WebAssembly.compileStreaming和WebAssembly.instatiateStreaming
  
### CSP应用程序策略摘要

对wasm操作执行的检查总结如下：

操作 | default | no unsafe-eval | with wasm-unsafe-eval | with unsafe-eval and wasm-unsafe-eval 
--- | --- | --- | --- | ---
JavaScript eval                                  | allow | SRI-hash | SRI-hash | allow
new WebAssembly.Module(bytes)                    | allow | SRI-hash | allow | allow 
WebAssembly.compile(bytes)                       | allow | SRI-hash | allow | allow 
WebAssembly.instantiate(bytes，...)              | allow | SRI-hash | allow | allow 
WebAssembly.instantiate(module，...)             | allow | allow | allow | allow 
WebAssembly.compileStreaming(Response)           | allow | script-src | script-src | script-src 
WebAssembly.instantiateStreaming(Response，...)  | allow | script-src | script-src | script-src
WebAssembly.validate(bytes)                      | allow | allow | allow | allow 
new WebAssembly.Instance(module)                 | allow | allow | allow | allow 
new WebAssembly.CompileError                     | allow | allow | allow | allow 
new WebAssembly.LinkError                        | allow | allow | allow | allow 
new WebAssembly.Table                            | allow | allow | allow | allow 
new WebAssembly.Memory                           | allow | allow | allow | allow 

其中SRI-hash表示基于所提供字节的哈希应用子资源完整性（sub-resource-integrity）检查，
如果哈希与白名单哈希不匹配，则拒绝该操作，
且script-src意味着拒绝CSP策略对脚本源的指令不允许的操作，例如，script-src限制源。
注意，`unsafe-eval`实际上*意味着*`wasm-unsafe-eval`.

### 例子

```
Content-Security-Policy: script-src 'self';

WebAssembly.compileStreaming(fetch('/foo.wasm'));  // OK
WebAssembly.instantiateStreaming(fetch('/foo.wasm')); // OK
WebAssembly.compileStreaming(fetch('/foo.js'));  // BAD: mime type
WebAssembly.instantiateStreaming(fetch('/foo.js')); // BAD: mime type
WebAssembly.compileStreaming(fetch('http://yo.com/foo.wasm'));  // BAD: cross origin
WebAssembly.instantiateStreaming(fetch('http://yo.com/foo.wasm')); // BAD: cross origin
```

```
Content-Security-Policy: script-src http://yo.com;

WebAssembly.compileStreaming(fetch('http://yo.com/foo.wasm'));  // OK
WebAssembly.instantiateStreaming(fetch('http://yo.com/foo.wasm')); // OK
```

```
Content-Security-Policy: script-src 'sha256-123...456';

WebAssembly.compileStreaming(fetch('http://baz.com/hash123..456.wasm'));  // OK
WebAssembly.instantiateStreaming(fetch('http://baz.com/hash123..456.wasm')); // OK
```
