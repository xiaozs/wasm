# 多值扩展

## 介绍

### 背景

* 目前，函数和指令消费多个操作数，但最多只能产生一个结果
  - 函数: `value* -> value?`
  - 指令: `value* -> value?`
  - 块: `[] -> value?`

* 在堆栈机中，这些不对称是人为的限制
  - 为了简化最初的WebAssembly版本 (多返回值被推迟到post-MVP)
  - 通过归并为 value* -> value*，能轻松进行升级

* 广义语义是很好理解的
  - https://github.com/WebAssembly/spec/tree/master/papers/pldi2017.pdf

* V8中对多返回值进行了半实现


### 动机

* 让函数能返回多个值:
  - 实现对元组或结构体返回值的拆箱
  - 多个返回值的高效编译

* 让指令能返回多个值:
  - 实现指令生成多个结果(divmod，进位运算)

* 带输入的块:
  - loop标签能带上参数
  - 可以在后边界表示phis
  - 现象未来的`pick`操作以跨越块边界
  - 带输入的指令的宏可定义性
    * `i32.select3` = `dup if ... else ... end`


## 例子

### 让函数能返回多个值

简单的交换函数。
```wasm
(func $swap (param i32 i32) (result i32 i32)
	(get_local 1) (get_local 0)
)
```

一个返回附加进位的加法函数。
```wasm
(func $add64_u_with_carry (param $i i64) (param $j i64) (param $c i32) (result i64 i32)
	(local $k i64)
	(set_local $k
		(i64.add (i64.add (get_local $i) (get_local $j)) (i64.extend_u/i32 (get_local $c)))
	)
	(return (get_local $k) (i64.lt_u (get_local $k) (get_local $i)))
)
```

### 让指令能返回多个值

* `iNN.divrem` : \[iNN iNN\] -> \[iNN iNN\]
* `iNN.add_carry` : \[iNN iNN i32\] -> \[iNN i32\]
* `iNN.sub_carry` : \[iNN iNN i32\] -> \[iNN i32\]
* 其他。


### 带输入的块

有条件地操作堆栈操作数而不使用局部变量。
```wasm
(func $add64_u_saturated (param i64 i64) (result i64)
	($i64.add_u_carry (get_local 0) (get_local 1) (i32.const 0))
	(if (param i64) (result i64)
		(then (drop) (i64.const 0xffff_ffff_ffff_ffff))
	)
)
```

迭代阶乘函数，循环不使用局部变量，转而使用块参数。
```wasm
(func $fac (param i64) (result i64)
	(i64.const 1) (get_local 0)
	(loop $l (param i64 i64) (result i64)
		(pick 1) (pick 1) (i64.mul)
		(pick 1) (i64.const 1) (i64.sub)
		(pick 0) (i64.const 0) (i64.gt_u)
		(br_if $l)
		(pick 1) (return)
	)
)
```

一个能展开成`if`指令的宏定义。
```
i64.select3  =
     dup if (param i64 i64 i64 i32) (result i64) … select ... else … end
```

`if`自身的宏展开.
```
if (param t*) (result u*) A else B end  =
      block (param t* i32) (result u*)
          block (param t* i32) (result t*) (br_if 0)  B  (br 1) end  A
      end
```


## 规范修改

### 结构

语言的结构基本上不受影响。唯一受影响的是类型的语法:

* *resulttype* 由 \[*valtype*?\] 改为 \[*valtype*\*\]
* 块类型(`block`，`loop`，`if`指令)由 *resulttype* 改为 *functype*


### 验证

删除Arity限制:

* 对于合法*functype*不再进行Arity验证
* 所有现有的 "?" 替换为 "\*" (例如，blocks，calls，return)

Validation for block instructions is generalised:

* `block`，`loop`，和`if`的类型变为*functype* \[t1\*\] -> \[t2\*\]，由块类型给出
* `block`和`if`的标签的类型变成\[t2\*\]
* `loop`的标签的类型变成\[t1\*\]


### 执行

对于多个结果，不需要做太多的工作:

* 所有现有的 "?" 替换为 "\*"。

唯一的非机械变化是使用操作数输入块:

* 操作数值从堆栈中退栈，并在标签后向重新入栈。
* 正式的简化规则的制定，请见论文


### 二进制格式

二进制文件需要更改以允许函数类型作为块类型。
这需要扩展当前的临时编码以允许对函数类型的引用。

* `blocktype`被扩展为一下格式:
  ```
  blocktype ::= 0x40       => [] -> []
             |  t:valtype  => [] -> [t]
             |  ft:typeidx => ft
  ```

### 文本格式

文本格式基本不受影响，除了块类型的语法:

* `resulttype` 被替换为 `blocktype`，它的语法是
  ```
  blocktype ::= vec(param) vec(result)
  ```

* `block`，`loop`，和 `if` 指令用 `blocktype` 以替代 `resulttype`。

* 现有的函数缩写适用于块类型。


### 可靠性性证明

需要归纳指令的类型，请见论文。


## 可能的替代品和扩展

### 更灵活的块和函数类型

* 可以使用对type段的引用来代替内联函数类型
  - 语义上更官僚化，但其他方面没问题

* 也可以同时允许
  - 对于一次性使用，内联函数类型稍微更紧凑一些

* 甚至可以将块类型的编码与任何地方的函数类型统一起来
  - 即使对于函数也允许内联类型，也有同样的好处


## 开放性问题

* 解构或重组多个值需要局部变量，这就足够了吗?
  - 可以添加`pick`指令(或者叫`dup`)
  - 可以添加`let`指令(或者叫`swap`)
  - 其他用例?

