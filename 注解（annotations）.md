# Wasm文本格式的自定义注解的语法

## 动机

问题

* Wasm二进制格式支持自定义段，以令Wasm模块能够关联任意元数据。

* 不存在等价的文本模式。 特别是，我们没有办法：
  - 以文本的方式表示自定义段，请参考 WebAssembly/design#1153 和 https://gist.github.com/binji/d1cfff7faaebb2aa4f8b1c995234e5a0
  - 以文本的方式表示任意名称，请参考 WebAssembly/spec#617
  - 表达像Web IDL bindings那样的信息，请参考 https://github.com/WebAssembly/webidl-bindings/blob/master/proposals/webidl-bindings/Explainer.md

解决方案

* 本提案增加以文本的方式装饰模块的能力，格式如`(@id ...)`的任意注解。

* 虽然句法形式和语义都不是由Wasm规范规定，但是附录中可能包含Name段注解和一般的自定义段的可选的支持的描述.

* 本提案仅影响文本格式。


## 细节

通过如下方式拓展文本格式:

* 只要是允许空格的地方，允许一下格式的*注解*:
  ```
  annot ::= "(@"idchar+ annotelem* ")"
  annotelem ::= keyword | reserved | uN | sN | fN | string | id | "(" annotelem* ")" | "(@"idchar+ annotelem* ")"
  ```
  换句话说，只要括号括好，一个注解允许任何形式的token。
  不允许任何空白符号在最初的分隔符`(@idchar+`中。

* 最初的`idchar+`作为分类拓展的标识符，并且扮演着和自定义段的名称相似的角色。
  按照惯例，与自定义段对应的注释应该使用相同的id。

拓展自定义段的附录:

* 定义反映Name段的注释，应该使用`(@name "name")`的形式。
  它们可以放在任何可以由Name段命名的结构的绑定器之后。

* 定义表示任意自定义段的注释语法; 请参考 https://gist.github.com/binji/d1cfff7faaebb2aa4f8b1c995234e5a0
  注解中存在的任何问题，都取决于实现是如何去处理显式的自定义段和与它重叠的单独注解。


## 例子

表示普通自定义段 (请参考 https://gist.github.com/binji/d1cfff7faaebb2aa4f8b1c995234e5a0)
```wasm
(module
  (@custom "my-fancy-section" (after function) "contents-bytes")
)
```

表示名称
```wasm
(module (@name "Gümüsü")
  (func $lambda (@name "λ") (param $x (@name "α βγ δ") i32) (result i32) (get_local $x))
)
```

Web IDL bindings (请参考 https://github.com/WebAssembly/webidl-bindings/blob/master/proposals/webidl-bindings/Explainer.md)
```wasm
(module
  (func (export "f") (param i32 (@js unsigned)) ...)                        ;; 参数转换为unsigned
  (func (export "method") (param $x anyref (@js this)) (param $y i32) ...)  ;; 将第一个参数映射为this
  (func (import "m" "constructor") (@js new) (param i32) (result anyref)    ;; 作为构造函数调用
)
```
