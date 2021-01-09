# WebAssembly的带符号扩展操作符提案

本页描述了post-MVP的[带符号扩展操作符特性][future sext]提案。

本提案新增了五个整数指令 带符号扩展8位，16位，和32位值。

## 新的带符号扩展操作符

为了支持带符号扩展，新增了五个带符号扩展操作符:

  * `i32.extend8_s`: 将一个带符号8位整数扩展为32位整数
  * `i32.extend16_s`: 将一个带符号16位整数扩展为32位整数
  * `i64.extend8_s`: 将一个带符号8位整数扩展为64位整数
  * `i64.extend16_s`: 将一个带符号16位整数扩展为64位整数
  * `i64.extend32_s`: 将一个带符号32位整数扩展为64位整数
  
注意`i64.extend32_s`并没有在2017年5月的CG会议中被讨论到。
原因是它的行为和`i64.extend_s/i32`匹配。
在之后的讨论中发现这是错误的，因为`i64.extend_s/i32`会将一个`i32`值带符号拓展为`i64`,
然而`i64.extend32_s`会将一个`i64`值带符号拓展为`i64`。
`i64.extend32_s`的行为能通过`i64.extend_s/i32`+`i32.wrap/i64`模拟
，但同样的情况也适用于带符号拓展加载操作符。
因此，加入`i64.extend32_s`以保持一致性。

## [规范改变][spec]

[指令语法][instruction syntax]做如下修改:

```
instr ::= ... |
          inn.extend8_s | inn.extend16_s | i64.extend32_s
```

[指令二进制格式][instruction binary format]做如下修改:

```
instr ::= ...
        | 0xC0                  =>  i32.extend8_s
        | 0xC1                  =>  i32.extend16_s
        | 0xC2                  =>  i64.extend8_s
        | 0xC3                  =>  i64.extend16_s
        | 0xC4                  =>  i64.extend32_s
```

[future sext]: https://github.com/WebAssembly/design/blob/master/FutureFeatures.md#additional-integer-operators
[instruction syntax]: https://webassembly.github.io/spec/syntax/instructions.html
[instruction binary format]: https://webassembly.github.io/spec/binary/instructions.html
[spec]: https://webassembly.github.io/sign-extension-ops/
