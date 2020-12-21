# WAT数值

## 概述

* 当前，数据节当中的数据值在WAT文件中只能写成字符串。 
For example:

    ```wat
    (data (offset (i32.const 0)) "1234")               ;; 3132 3334
    ```
    ```wat
    (data (offset (i32.const 0)) "\09\ab\cd\ef")       ;; 09ab cdef
    ```

* 本提案建议支持其他的书写方式，以允许对整数、浮点数的写入。

    ```wat
    (data (offset (i32.const 0))
        (f32 0.2 0.3 0.4)          ;; cdcc 4c3e 9a99 993e cdcc cc3e
    )
    ```
    ```wat
    (memory $1
        (data 
            (i8 1 2)               ;; 0102
            (i16 3 4)              ;; 0300 0400
        )
    )
    ```

* **原型:** https://wasmprop-numerical-data.netlify.app/wat2wasm/

* 实现了本提案的编译器:

    - [wasp](https://github.com/WebAssembly/wasp) (使用`--enable-numeric-values`标志)
    - [Rust `wat`/`wast` crates](https://github.com/bytecodealliance/wasm-tools)

## 动机

* 往数据节中写入任意数值(整数和浮点数)可不简单。
我们需要对数据进行编码，新增转义字符`\`，and write it as strings.

    例如，下面的片段意味着往数据节中写入一些浮点数。

    ```wat
    (data (i32.const 0)
        "\00\00\c8\42"
        "\00\00\80\3f"
        "\00\00\00\3f"
    )
    ```

* 如果我们需要回顾上面的数值，不经过解码我们无法轻易的把它识别出来。

    `"\00\00\c8\42"` => `0x42c80000` => `100.0`

* x86-64 GCC 和 x86-64 Clang 有[汇编指令](https://ftp.gnu.org/old-gnu/Manuals/gas-2.9.1/html_chapter/as_7.html)，可用于往数据段中写入任意数值。

    以下是一些例子:

    ```s
    data:
    .ascii  "abcd"         #; 61 62 63 64
    .byte   1, 2, 3, 4     #; 01 02 03 04
    .2byte  5, 6           #; 05 00 06 00
    .4byte  0x89ABCDEF     #; EF CD AB 89
    .8byte  -1             #; FF FF FF FF FF FF FF FF
    .float  62.5           #; 00 00 7A 42
    .double 62.5           #; 00 00 00 00 00 40 4F 40
    ```

    在NASM中，他们是[伪指令](http://www.tortall.net/projects/yasm/manual/html/nasm-pseudop.html)，例如:

    ```asm
    data:
    db          'abcd', 0x01, 2, 3, 4   ; 61 62 63 64 01 02 03 04
    dw          5, 6                    ; 05 00 06 00
    dd          62.5                    ; 00 00 7A 42
    dq          62.5                    ; 00 00 00 00 00 40 4F 40
    times 4 db  0xAB                    ; AB AB AB AB
    ```

    这些指令帮助程序员直接往代码里面读和写数据，通过一种人类可读的格式，而不是编码后的格式。
    本提案位WebAssembly带来类似的功能的文本格式。

## 预览

本提案提议给文本格式增加一些微小的修改，以支持往数据节中写入数值。

### 文本格式规范的改变

数据节中的数据值应该能够同时接受字符串和一系列数值(numlist)。

<pre>
data<sub>I</sub> ::= '(' 'data' x:<a href="https://webassembly.github.io/spec/core/text/modules.html#text-memidx">memidx</a><sub>I</sub> '(' 'offset' e:<a href="https://webassembly.github.io/spec/core/text/instructions.html#text-expr">expr</a><sub>I</sub> ')' b*:<a href="https://github.com/WebAssembly/wat-numeric-values/blob/master/proposals/wat-numeric-values/Overview.md#text-format-spec-changes">dataval</a> ')'
                => { <a href="https://webassembly.github.io/spec/core/syntax/modules.html#syntax-data">data</a> x'，<a href="https://webassembly.github.io/spec/core/syntax/modules.html#syntax-data">offset</a> e，<a href="https://webassembly.github.io/spec/core/syntax/modules.html#syntax-data">init</a> b* }

dataval ::= (b*:<a href="https://github.com/WebAssembly/wat-numeric-values/blob/master/proposals/wat-numeric-values/Overview.md#text-format-spec-changes">datavalelem</a>)*       => <a href="https://webassembly.github.io/spec/core/syntax/conventions.html#notation-concat">concat</a>((b*)*)

datavalelem ::= b*:<a href="https://webassembly.github.io/spec/core/text/values.html#text-string">string</a>           => b*
             |  b*:<a href="https://github.com/WebAssembly/wat-numeric-values/blob/master/proposals/wat-numeric-values/Overview.md#text-format-spec-changes">numlist</a>           => b*
</pre>

Numlist代表一系列字节。
它们用括号括起来，首先用关键字标识数字的类型，然后是数字列表。

numlist中的数字表示使用各自编码的字节序列。
它们使用整数补码编码和浮点值的IEEE754编码进行编码。
每个numlist符号表示这些数字的字节的连接。

<pre>
numlist ::= '(' 'i8'  (n:<a href="https://webassembly.github.io/spec/core/text/values.html#text-int">i8</a>)*  ')'      => <a href="https://webassembly.github.io/spec/core/syntax/conventions.html#notation-concat">concat</a>((<a href="https://webassembly.github.io/spec/core/exec/numerics.html#aux-bytes">bytes</a><sub>i8</sub>(n))*)    (if |<a href="https://webassembly.github.io/spec/core/syntax/conventions.html#notation-concat">concat</a>((<a href="https://webassembly.github.io/spec/core/exec/numerics.html#aux-bytes">bytes</a><sub>i8</sub>(n))*) | < 2<sup>32</sup>)
        |  '(' 'i16' (n:<a href="https://webassembly.github.io/spec/core/text/values.html#text-int">i16</a>)* ')'      => <a href="https://webassembly.github.io/spec/core/syntax/conventions.html#notation-concat">concat</a>((<a href="https://webassembly.github.io/spec/core/exec/numerics.html#aux-bytes">bytes</a><sub>i16</sub>(n))*)   (if |<a href="https://webassembly.github.io/spec/core/syntax/conventions.html#notation-concat">concat</a>((<a href="https://webassembly.github.io/spec/core/exec/numerics.html#aux-bytes">bytes</a><sub>i16</sub>(n))*)| < 2<sup>32</sup>)
        |  '(' 'i32' (n:<a href="https://webassembly.github.io/spec/core/text/values.html#text-int">i32</a>)* ')'      => <a href="https://webassembly.github.io/spec/core/syntax/conventions.html#notation-concat">concat</a>((<a href="https://webassembly.github.io/spec/core/exec/numerics.html#aux-bytes">bytes</a><sub>i32</sub>(n))*)   (if |<a href="https://webassembly.github.io/spec/core/syntax/conventions.html#notation-concat">concat</a>((<a href="https://webassembly.github.io/spec/core/exec/numerics.html#aux-bytes">bytes</a><sub>i32</sub>(n))*)| < 2<sup>32</sup>)
        |  '(' 'i64' (n:<a href="https://webassembly.github.io/spec/core/text/values.html#text-int">i64</a>)* ')'      => <a href="https://webassembly.github.io/spec/core/syntax/conventions.html#notation-concat">concat</a>((<a href="https://webassembly.github.io/spec/core/exec/numerics.html#aux-bytes">bytes</a><sub>i64</sub>(n))*)   (if |<a href="https://webassembly.github.io/spec/core/syntax/conventions.html#notation-concat">concat</a>((<a href="https://webassembly.github.io/spec/core/exec/numerics.html#aux-bytes">bytes</a><sub>i64</sub>(n))*)| < 2<sup>32</sup>)
        |  '(' 'f32' (n:<a href="https://webassembly.github.io/spec/core/text/values.html#text-float">f32</a>)* ')'      => <a href="https://webassembly.github.io/spec/core/syntax/conventions.html#notation-concat">concat</a>((<a href="https://webassembly.github.io/spec/core/exec/numerics.html#aux-bytes">bytes</a><sub>f32</sub>(n))*)   (if |<a href="https://webassembly.github.io/spec/core/syntax/conventions.html#notation-concat">concat</a>((<a href="https://webassembly.github.io/spec/core/exec/numerics.html#aux-bytes">bytes</a><sub>f32</sub>(n))*)| < 2<sup>32</sup>)
        |  '(' 'f64' (n:<a href="https://webassembly.github.io/spec/core/text/values.html#text-float">f64</a>)* ')'      => <a href="https://webassembly.github.io/spec/core/syntax/conventions.html#notation-concat">concat</a>((<a href="https://webassembly.github.io/spec/core/exec/numerics.html#aux-bytes">bytes</a><sub>f64</sub>(n))*)   (if |<a href="https://webassembly.github.io/spec/core/syntax/conventions.html#notation-concat">concat</a>((<a href="https://webassembly.github.io/spec/core/exec/numerics.html#aux-bytes">bytes</a><sub>f64</sub>(n))*)| < 2<sup>32</sup>)
</pre>

这个新的数据值形式也应该在内存模块的内联数据节中可用。

<pre>
'(' 'memory' <a href="https://webassembly.github.io/spec/core/text/values.html#text-id">id</a><sup>?</sup> '(' 'data' b<sup>n</sup>:<a href="https://github.com/WebAssembly/wat-numeric-values/blob/master/proposals/wat-numeric-values/Overview.md#text-format-spec-changes">dataval</a> ')' ')' ≡
    '(' 'memory' <a href="https://webassembly.github.io/spec/core/text/values.html#text-id">id</a>' m m ')' '(' 'data' <a href="https://webassembly.github.io/spec/core/text/values.html#text-id">id</a>' '(' 'i32.const' '0' <a href="https://github.com/WebAssembly/wat-numeric-values/blob/master/proposals/wat-numeric-values/Overview.md#text-format-spec-changes">dataval</a> ')'
        (if <a href="https://webassembly.github.io/spec/core/text/values.html#text-id">id</a>'=<a href="https://webassembly.github.io/spec/core/text/values.html#text-id">id</a><sup>?</sup> ≠ 𝜖 ∨ <a href="https://webassembly.github.io/spec/core/text/values.html#text-id">id</a>' <a href="https://webassembly.github.io/spec/core/text/values.html#text-id-fresh">fresh</a>，m=ceil(n/64Ki))
</pre>

#### SIMD v128

SIMD提案引入了[v128值类型](https://github.com/WebAssembly/simd/blob/master/proposals/simd/SIMD.md#simd-value-type)。
当支持SIMD特性时，`datavalelem`也应该能接受一系列的`v128`值类型。

<pre>
datavalelem ::= b*:<a href="https://webassembly.github.io/spec/core/text/values.html#text-string">string</a>           => b*
             |  b*:<a href="https://github.com/WebAssembly/wat-numeric-values/blob/master/proposals/wat-numeric-values/Overview.md#text-format-spec-changes">numlist</a>           => b*
             |  b*:v128list         => b*

v128list ::= '(' 'v128' (<i><a href="https://github.com/WebAssembly/simd/blob/master/proposals/simd/TextSIMD.md">v128 constant text format</a></i>)* ')'
</pre>

请看下面的用例。

### 用例

```wat
;; XYZ坐标点
(data (offset (i32.const 0))
    (f32 0.2 0.3 0.4)
    (f32 0.4 0.5 0.6)
    (f32 0.4 0.5 0.6)
)

;; 写入第1001个 ~ 第1010个质数
(data (offset (i32.const 0x100))
    (i16 7927 7933 7937 7949 7951 7963 7993 8009 8011 8017)
)

;; PI
(data (offset (i32.const 0x200))
    (f64 3.14159265358979323846264338327950288)
)

;; v128数组(SIMD)
(data (offset (i32.const 0x300))
    (v128
        i32x4 0 0 0 0
        f64x2 1.0 1.5
    )
)

;; 内联到内存模块里
(memory (data (i8 1 2 3 4)))
```

### 运行结果例子

将numlist转换为数据段中的数据这一步发生在文本格式被编译为二进制格式时。

因此，以下两个片段:

```wat
...
(memory 1)
(data (offset (i32.const 0))
    "abcd"
    (i16 -1)
    (f32 62.5)
)
...
```
```wat
...
(memory 1)
(data (offset (i32.const 0))
    "abcd"
    "\FF\FF"
    "\00\00\7a\42"
)
...
```

会输出同样的二进制代码:

```
...
; 数据节头部0
0000010: 00                                        ; 节flags
0000011: 41                                        ; i32.const
0000012: 00                                        ; i32 literal
0000013: 0b                                        ; end
0000014: 0a                                        ; 数据节大小
; 数据节数据0
0000015: 6162 6364 ffff 0000 7a42                  ; 数据节数据
000000e: 10                                        ; FIXUP 节大小
...
```

#### SIMD v128 扩展例子

下面3个片段会输出相同的二进制代码:
```wat
(data (offset (i32.const 0x00))
    (v128
        i32x4 0xA 0xB 0xC 0xD
        f64x2 1.0 0.5
    )
)
```
```wat
(data (offset (i32.const 0x00))
    (i32 0xA 0xB 0xC 0xD)
    (f64 1.0 0.5)
)
```
```wat
(data (offset (i32.const 0x00))
    "\0a\00\00\00\0b\00\00\00\0c\00\00\00\0d\00\00\00"
    "\00\00\00\00\00\00\f0?\00\00\00\00\00\00\e0?"
)
```

### 新增信息

#### 编码

如前所述，numlist中的数值使用的时整数补码编码和浮点值的IEEE754编码，这和`t.store`内存指令相同。
这个编码确保了我们从内存中加载数据时使用`load`内存指令，该值无论是使用`(data ... )`初始化，还是`t.store`指令都会保持一致。

#### 数据对齐

允许未对齐数据。例子:

```wat
...
(memory 1)
(data (offset (i32.const 0))
  (i8 1)    ;; 地址0
  (i16 2)   ;; 地址1
)
...
```
编译为: `0102 00`

#### 超出数值范围

超出数值范围时应该在文本格式编译为二进制格式时抛出错误。

```wat
(memory 1)
(data (offset (i32.const 0))
  (i8 256)        ;; 错误
  (i8 -129)       ;; 错误
)
```

#### 二进制格式转为文本格式

编译后的二进制数据节不能包含WAT中的任何原始信息。
因此，从二进制格式转回文本格式会使用默认的字符串格式。

#### 后向兼容

因为语法仍然支持字符串格式，所有已有的WAT代码都会正常工作。

## 链接

* 初始设计问题讨论: https://github.com/WebAssembly/design/issues/1348

* CG会议中的初次演示: https://github.com/WebAssembly/meetings/blob/master/main/2020/CG-06-09.md

* 进入阶段1的投票: https://github.com/WebAssembly/meetings/blob/master/main/2020/CG-06-23.md
