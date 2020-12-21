# 64位内存

## 概述

本页描述了支持大小大于2<sup>32</sup>位的线性内存的提案。
它没有引入了新的指令，但是扩展了已有指令以支持64位索引。

## 动机

[WebAssembly线性内存对象][memory object]有通过[页][memory page]定义的大小。 
每个页的大小为65536 (2<sup>16</sup>)个字节。
在WebAssembly版本1中，线性内存最多有65536个页，总共2<sup>32</sup>字节(4 [Gb][gibibyte])。

除了页的限制以外，所有[内存指令][memory instructions]当前都是用[`i32`类型][i32]作为内存索引。
这意味着它们对多能够寻址2<sup>32</sup>字节。

对于很多应用来说，4Gb内存已经足够了。
对于这些例子使用32位内存索引就足够了，而且带来了指针大小更小的优点，这样可以节省内存。
但是，对于另外一些应用它们需要更多的内存，对于当前的WebAssembly功能集，没有简单的解决方法。
允许WebAssembly模块在32位和64位内存索引寻址之间做出选择是很重要的。

同样地，因为WebAssembly是一个虚拟[指令集架构][ISA]
(ISA)，一些宿主可能将WebAssembly二进制格式用作可执行格式，为了支持其他非虚拟ISA。
几乎所有的ISA对64位内存寻址有支持，且宿主可能不想在它们的ABI里面支持32位内存寻址。

## 概述

### 结构

* [limits][syntax limits]结构改为使用`u64`
  - `limits ::= {min u64, max u64?}`

* 新的`idxtype`同时适用`i32`或`i64`
  - `idxtype ::= i32 | i64`

* [memtype][syntax memtype]结构扩展为有索引类型
  - `memtype ::= limits idxtype`

* [memarg][syntax memarg]立即数改为允许64位offset
  - `memarg ::= {offset u64, align u32}`


### 验证

* [内存页限制][valid limits]扩展为`i64`索引
  - ```
    ⊦ limits : 2**16
    ----------------
    ⊦ limits i32 ok
    ```
  - ```
    ⊦ limits : 2**48
    ----------------
    ⊦ limits i64 ok
    ```

* 所有[内存指令][valid meminst]改为使用索引类型,
  且offset范围扩展到支持索引类型
  - t.load memarg
    - ```
      C.mems[0] = limits it   2**memarg.align <= |t|/8   memarg.offset < 2**|it|
      --------------------------------------------------------------------------
                          C ⊦ t.load memarg : [it] → [t]
      ```
  - t.loadN_sx memarg
    - ```
      C.mems[0] = limits it   2**memarg.align <= N/8   memarg.offset < 2**|it|
      ------------------------------------------------------------------------
                        C ⊦ t.loadN_sx memarg : [it] → [t]
      ```
  - t.store memarg
    - ```
      C.mems[0] = limits it   2**memarg.align <= |t|/8   memarg.offset < 2**|it|
      --------------------------------------------------------------------------
                         C ⊦ t.store memarg : [it t] → []
      ```
  - t.storeN_sx memarg
    - ```
      C.mems[0] = limits it   2**memarg.align <= N/8   memarg.offset < 2**|it|
      ------------------------------------------------------------------------
                       C ⊦ t.storeN_sx memarg : [it t] → []
      ```
  - memory.size
    - ```
         C.mems[0] = limits it
      ---------------------------
      C ⊦ memory.size : [] → [it]
      ```
  - memory.grow
    - ```
          C.mems[0] = limits it
      -----------------------------
      C ⊦ memory.grow : [it] → [it]
      ```
  - memory.fill
    - ```
          C.mems[0] = limits it
      -----------------------------
      C ⊦ memory.fill : [it i32 it] → []
      ```
  - memory.copy
    - ```
          C.mems[0] = limits it
      -----------------------------
      C ⊦ memory.copy : [it it it] → []
      ```
  - memory.init x
    - ```
          C.mems[0] = limits it   C.datas[x] = ok
      -------------------------------------------
          C ⊦ memory.init : [it i32 i32] → []
      ```
  - (其他提案中的内存指令也类似)

* [多内存提案][multi memory]给这些指令扩展了一到两个立即数。
  对于这些内存，索引类型会被用掉。例如，
  - memory.size x
    - ```
         C.mems[x] = limits it
      ---------------------------
      C ⊦ memory.size x : [] → [it]
      ```

  `memory.copy`有两个内存索引立即数，所以会有多个可能的签名:
  - memory.copy d s
    - ```
          C.mems[d] = limits it_d   C.mems[s] = limits it_s
      --------------------------------------------------------
          C ⊦ memory.copy d s : [it_d it_s f(it_d, it_s)] → []

      有:

        f(i32, i32) = i32
        f(i64, i32) = i32
        f(i32, i64) = i32
        f(i64, i64) = i64
      ```

* [数据节验证][valid data]使用索引类型
  - ```
    C.mems[0] = limits it   C ⊦ expr: [it]   C ⊦ expr const
    -------------------------------------------------------
          C ⊦ {data x, offset expr, init b*} ok
    ```


### 执行

* [内存实例][exec mem]被扩展为拥有64位数组和`u64`最大大小
  - `meminst ::= { data vec64(byte), max u64? }`

* [内存指令][exec meminst]使用索引类型替代`i32`
  - `t.load memarg`
  - `t.loadN_sx  memarg`
  - `t.store memarg`
  - `t.storeN memarg`
  - `memory.size`
  - `memory.grow`
  - (规范文本省略)

* [memory.grow][exec memgrow]其行为取决于索引类型:
  - 对于`i32`: 不变
  - 对于`i64`: 检查大小是否大于2<sup>64</sup> - 1，在执行失败时返回2<sup>64</sup> - 1。

* [内存导入匹配][exec memmatch]要求索引类型匹配
  - ```
      ⊦ limits_1 <= limits_2   it_1 = it_2
    ----------------------------------------
    ⊦ mem limits_1 it_1 <= mem limits_2 it_2
    ```


### 二进制格式

* [limits][binary limits]结构将编码附加值以代表索引类型
  - ```
    limits ::= 0x00 n:u32        ⇒ {min n, max ϵ}, 0
            |  0x01 n:u32 m:u32  ⇒ {min n, max m}, 0
            |  0x02 n:u32        ⇒ {min n, max ϵ}, 1  ;; 来自线程提案
            |  0x03 n:u32 m:u32  ⇒ {min n, max m}, 1  ;; 来自线程提案
            |  0x04 n:u64        ⇒ {min n, max ϵ}, 2
            |  0x05 n:u64 m:u64  ⇒ {min n, max m}, 2
    ```

* [memtype][binary memtype]结构采用上述limits编码
  - ```
    memtype ::= lim, 0:limits ⇒ lim i32
             |  lim, 2:limits ⇒ lim i64
    ```

* [memarg][binary memarg]的offset被读为`u64`
  - `memarg ::= a:u32 o:u64`

### 文本格式

*  这是新的索引类型:
   - ```
     indextype ::= 'i32' | 'i64'
     ```

*  [memtype][text memtype]被扩展为允许一个可选的索引类型，其必须是`i32`或`i64`
   - ```
     memtype ::= lim:limits               ⇒ lim i32
              |  it:indextype lim:limits  ⇒ lim it
     ```

* [内存缩写][text memabbrev]定义扩展为允许一个可选的索引类型，其必须是`i32`或`i64`
  - ```
    '(' 'memory' id? index_type? '(' 'data' b_n:datastring ')' ')' === ...
    ```


[memory object]: https://webassembly.github.io/spec/core/syntax/modules.html#memories
[memory page]: https://webassembly.github.io/spec/core/exec/runtime.html#page-size
[gibibyte]: https://en.wikipedia.org/wiki/Gibibyte
[i32]: https://webassembly.github.io/spec/core/syntax/types.html#syntax-valtype
[memory instructions]: https://webassembly.github.io/spec/core/syntax/instructions.html#memory-instructions
[ISA]: https://en.wikipedia.org/wiki/Instruction_set_architecture
[syntax limits]: https://webassembly.github.io/spec/core/syntax/types.html#syntax-limits
[syntax memtype]: https://webassembly.github.io/spec/core/syntax/types.html#memory-types
[syntax memarg]: https://webassembly.github.io/spec/core/syntax/instructions.html#syntax-memarg
[valid limits]: https://webassembly.github.io/spec/core/valid/types.html#limits
[valid meminst]: https://webassembly.github.io/spec/core/valid/instructions.html#memory-instructions
[valid data]: https://webassembly.github.io/spec/core/valid/modules.html#data-segments
[exec mem]: https://webassembly.github.io/spec/core/exec/runtime.html#memory-instances
[exec meminst]: https://webassembly.github.io/spec/core/exec/instructions.html#memory-instructions
[exec memgrow]: https://webassembly.github.io/spec/core/exec/instructions.html#exec-memory-grow
[exec memmatch]: https://webassembly.github.io/spec/core/exec/modules.html#memories
[binary limits]: https://webassembly.github.io/spec/core/binary/types.html#limits
[binary memtype]: https://webassembly.github.io/spec/core/binary/types.html#memory-types
[binary memarg]: https://webassembly.github.io/spec/core/binary/instructions.html#binary-memarg
[text memtype]: https://webassembly.github.io/spec/core/text/types.html#text-memtype
[text memabbrev]: https://webassembly.github.io/spec/core/text/modules.html#text-mem-abbrev
[multi memory]: https://github.com/webassembly/multi-memory
