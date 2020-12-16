# 非陷入的Float到int转换

## 介绍

### 动机

基础动机位:

 - LLVM的Float到int转换存在未定义结果，而不是未定义行为，而且看起来LLVM确实在一定条件下进行了推测。
 - 对于SIMD提案来说，如果SIMD操作没有陷入，它就更像是硬件支持的SIMD。
  本提案将为饱和操作确定一种转换方式，其能够被SIMD共享，以避免陷入。

本提案不为性能。

LLVM可能会被改变，例如给"fptosi"指令加一个"nsw"flag，特别是通过一些最近的提案对LLVM中的一些坏理念进行修改。
虽说如此，但是现在还没人来做这个。

### 背景

很多地方都讨论了这个问题:

https://github.com/WebAssembly/design/issues/986

虽然最初的讨论是出于对性能的考虑，但是性能影响是由一个特定的实现细节造成的，这个细节后来被修复了。

从此，没有与此问题相关的实际性能问题的报告。

这个话题在CG-05会议中被讨论到:

https://github.com/WebAssembly/meetings/pull/3

还有

https://github.com/WebAssembly/meetings/blob/master/2017/CG-05.md#non-trapping-float-to-int

讨论了应该选用什么语义，和什么编码策略.

这些决定都记录到了设计中:

https://github.com/WebAssembly/design/pull/1089

在CG-07-06会议中，我们决定，一个完整规范应该遵循新流程来获得新特性:

https://github.com/WebAssembly/meetings/blob/master/2017/CG-07-06.md#float-to-int-conversion

这导致了本提案的产生:

https://github.com/WebAssembly/nontrapping-float-to-int-conversions

### 设计

本提案新增8条指令:

 - `i32.trunc_sat_f32_s`
 - `i32.trunc_sat_f32_u`
 - `i32.trunc_sat_f64_s`
 - `i32.trunc_sat_f64_u`
 - `i64.trunc_sat_f32_s`
 - `i64.trunc_sat_f32_u`
 - `i64.trunc_sat_f64_s`
 - `i64.trunc_sat_f64_u`

它们的语义和没有`_sat`的指令一一对应，除了:
 - 将正溢出或负溢出陷入分别替换成返回最大或最小整数值。 (这种行为也被称为"饱和")
 - 将NaN陷入，替换成返回0。

### 编码

本提案新增一个新的前缀字节:

| 前缀   | 名称    | 描述 |
| ------ | ------- | ----------- |
| `0xfc` | misc    | 其他操作 |

它也将用作将来其他杂项操作的前缀.

新指令的编码使用了这个新的前缀:

| 名称 | 操作码 | 立即数 | 描述 |
| ---- | ---- | ---- | ---- |
| `i32.trunc_sat_f32_s` | `0xfc` `0x00` | | 饱和形式的`i32.trunc_f32_s` |
| `i32.trunc_sat_f32_u` | `0xfc` `0x01` | | 饱和形式的`i32.trunc_f32_u` |
| `i32.trunc_sat_f64_s` | `0xfc` `0x02` | | 饱和形式的`i32.trunc_f64_s` |
| `i32.trunc_sat_f64_u` | `0xfc` `0x03` | | 饱和形式的`i32.trunc_f64_u` |
| `i64.trunc_sat_f32_s` | `0xfc` `0x04` | | 饱和形式的`i64.trunc_f32_s` |
| `i64.trunc_sat_f32_u` | `0xfc` `0x05` | | 饱和形式的`i64.trunc_f32_u` |
| `i64.trunc_sat_f64_s` | `0xfc` `0x06` | | 饱和形式的`i64.trunc_f64_s` |
| `i64.trunc_sat_f64_u` | `0xfc` `0x07` | | 饱和形式的`i64.trunc_f64_u` |
