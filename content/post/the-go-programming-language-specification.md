---
title: "Go 编程语言规范"
date: 2024-08-24T12:00:00+08:00
isCJKLanguage: true
Description: "Go 编程语言规范"
Tags: ["go", "golang", "spec"]
Categories: ["go"]
DisableComments: false
---

原文地址: [The Go Programming Language Specification](https://go.dev/ref/spec)

Go 官方编程语言规范中文翻译版本（Go 版本 go1.23 (2024年8月24日)）
<!--more-->

# 引言

这是一份Go编程语言的参考手册。没有泛型的 Go1.18 之前的版本请看 [这里](https://go.dev/doc/go1.17_spec.html)。其他更多信息和文档，请参阅 [go.dev](https://go.dev)。

Go是一个面向系统编程设计的通用语言。它是强类型语言且有垃圾回收机制，并且支持并发编程。程序由 *包(packages)* 组成，其特性使得依赖关系的管理非常高效。

语法紧凑且简单，易于解析，这样可以方便地使用自动化工具如集成开发环境进行分析。

# 符号表示法

文法使用[扩展巴科斯-瑙尔范式（Extended Backus-Naur Form，EBNF）](https://zh.wikipedia.org/zh-cn/%E6%89%A9%E5%B1%95%E5%B7%B4%E7%A7%91%E6%96%AF%E8%8C%83%E5%BC%8F)
的[变体](https://en.wikipedia.org/wiki/Wirth_syntax_notation)来描述：

```
Syntax      = { Production } .
Production  = production_name "=" [ Expression ] "." .
Expression  = Term { "|" Term } .
Term        = Factor { Factor } .
Factor      = production_name | token [ "…" token ] | Group | Option | Repetition .
Group       = "(" Expression ")" .
Option      = "[" Expression "]" .
Repetition  = "{" Expression "}" .
```

产生式(Production) 是由 Term 和以下操作符构建的 表达式(Expression)，按优先级递增：

```
|   alternation
()  grouping
[]  option (0 or 1 times)
{}  repetition (0 to n times)
```

小写的产生式名称用来标识终结词法符号，分终结的使用大驼峰格式。词法符号使用双引号 ""  或反引号 `` 包含起来

The form a … b represents the set of characters from a through b as alternatives. The horizontal ellipsis … is also used elsewhere in the spec to informally denote various enumerations or code snippets that are not further specified. The character … (as opposed to the three characters ...) is not a token of the Go language.

A link of the form [Go 1.xx] indicates that a described language feature (or some aspect of it) was changed or added with language version 1.xx and thus requires at minimum that language version to build. For details, see the linked section in the appendix.

# 源代码表示

源代码是使用 UTF-8 编码的 Unicode 文本。文本未进行规范化，因此一个带重音的代码点与通过组合重音和字母构建的相同字符是不同的；
它们被视为两个代码点。为了简化，本文件将使用未限定的术语 *字符(charactor)* 来指代源文本中的 Unicode 代码点。


每个代码点都是独特的；例如，大写和小写字母是不同的字符。

实现限制：为了与其他工具兼容，编译器可能会禁止源文本中的空字符（U+0000）。

实现限制：为了与其他工具兼容，编译器可能在源文本的第一 Unicode 代码点是 UTF-8 编码的字节顺序标记（U+FEFF）时忽略该字节顺序标记。在源文本的其他任何位置，字节顺序标记可能被禁止。

## 字符（Characters）

以下术语用于表示特定的 Unicode 字符类别：

```
newline        = /* the Unicode code point U+000A */ .
unicode_char   = /* an arbitrary Unicode code point except newline */ .
unicode_letter = /* a Unicode code point categorized as "Letter" */ .
unicode_digit  = /* a Unicode code point categorized as "Number, decimal digit" */ .
```

在 [Unicode 标准 8.0](https://www.unicode.org/versions/Unicode8.0.0/) 第 4.5 节“通用类别”中，定义了一组字符类别。
Go 将任何属于 Lu、Ll、Lt、Lm 或 Lo 字母类别中的字符视为 Unicode 字母，并将属于 Nd 数字类别的字符视为 Unicode 数字。

## 字母和数字

下划线字符 _ (U+005F) 被认为是小写字母。

```
letter        = unicode_letter | "_" .
decimal_digit = "0" … "9" .
binary_digit  = "0" | "1" .
octal_digit   = "0" … "7" .
hex_digit     = "0" … "9" | "A" … "F" | "a" … "f" .
```

# 词法元素

## 注释

注释作为程序文档。有两种形式：

1. 行注释以字符序列 // 开始，并在行末结束。

2. 通用注释以字符序列 /* 开始，并在随后的第一个字符序列 */ 处结束。

注释不能出现在 rune 或字符串字面量内，也不能出现在注释内部。
没有换行的通用注释（一般注释）就像空白字符一样作用。其他类型的注释，就像换行一样作用。

## 词法单元(Tokens)

词法单元是构成Go语言词汇的基本单位。
有四个类别：标识符、关键字、运算符和标点符号、字面量。
空白字符（包括空格、水平制表符、回车和换行）是被忽略的，除非它们之间作为分隔符令牌合并成一个令牌。 
另外，如果遇到回车或文件结束符，则可能会触发插入一个`;`。
当将输入分解为词法单元时，下一个词法单元是能形成有效词法单元的最长子字符序列。

## 分号

正式语法使用分号";"作为一些产生式的结束符号。Go程序可以省略大部分这些分号，按照以下两个规则：

1. 当输入被分解为词元时，将自动插入一个分号，如果该词元是

   - 标识符
   - 整数、浮点、虚数、Rune 或字符串字面量
   - 关键字 break、continue、fallthrough 或 return
   - 运算符和标点符号 ++、--、）、] 或 }

2. 为了允许复杂语句占据一行，可以在结束语“）”或“}”之前省略分号。

为了反映惯用法，本文档中的代码示例使用这些规则省略了分号。

## 标识符

标识符为程序实体命名，例如变量和类型。标识符是由一个或多个字母和数字组成的序列。标识符中的第一个字符必须是字母。

```
identifier = letter { letter | unicode_digit } .
```

```
a
_x9
ThisVariableIsExported
α
```

译注：下划线和中文都被当作小写字母，可以作为开头，不可导出
```
你好abc
_42
```

某些标识符是预先声明的。

## 关键字

以下关键字是保留的，不能用作标识符。

```
break        default      func         interface    select
case         defer        go           map          struct
chan         else         goto         package      switch
const        fallthrough  if           range        type
continue     for          import       return       var
```

## 操作符和标点符号

下字符序列表示运算符（包括赋值运算符）和标点符号 [Go 1.18]：

```
+    &     +=    &=     &&    ==    !=    (    )
-    |     -=    |=     ||    <     <=    [    ]
*    ^     *=    ^=     <-    >     >=    {    }
/    <<    /=    <<=    ++    =     :=    ,    ;
%    >>    %=    >>=    --    !     ...   .    :
     &^          &^=          ~
```

## 整数字面量

整数文字是表示整数常数的数字序列。可选前缀设置非十进制基数：0b 或 0B 表示二进制，0、0o 或 0O 表示八进制，0x 或 0X 表示十六进制 [Go 1.13]。
单个 0 被视为十进制零。在十六进制文字中，字母 a 到 f 和 A 到 F 表示值 10 到 15。

为了便于阅读，下划线字符 _ 可以出现在基本前缀之后或连续数字之间;此类下划线不会更改文本的值。

```
int_lit        = decimal_lit | binary_lit | octal_lit | hex_lit .
decimal_lit    = "0" | ( "1" … "9" ) [ [ "_" ] decimal_digits ] .
binary_lit     = "0" ( "b" | "B" ) [ "_" ] binary_digits .
octal_lit      = "0" [ "o" | "O" ] [ "_" ] octal_digits .
hex_lit        = "0" ( "x" | "X" ) [ "_" ] hex_digits .

decimal_digits = decimal_digit { [ "_" ] decimal_digit } .
binary_digits  = binary_digit { [ "_" ] binary_digit } .
octal_digits   = octal_digit { [ "_" ] octal_digit } .
hex_digits     = hex_digit { [ "_" ] hex_digit } .
```

```
42
4_2
0600
0_600
0o600
0O600       // second character is capital letter 'O'
0xBadFace
0xBad_Face
0x_67_7a_2f_cc_40_c6
170141183460469231731687303715884105727
170_141183_460469_231731_687303_715884_105727

_42         // an identifier, not an integer literal
42_         // invalid: _ must separate successive digits
4__2        // invalid: only one _ at a time
0_xBadFace  // invalid: _ must separate successive digits
```

## 浮点数字面量

浮点字面量是十进制或十六进制表示的浮点常量。

## 复数字面量

## rune字面量

## 字符串字面量

# 常量

# 变量

# 类型

## 布尔类型

## 数字类型

## 字符串类型

## 数组类型

## 切片类型

## 结构体类型

## 指针类型

## 函数类型

## 接口类型

## Map类型

## Channel 类型



# 类型和值的属性

# 块

# 声明和范围

## 标签范围

## 空白标识符

## 预先声明的标识符

## 导出的标识符

## 标识符的唯一性

## 常量声明

## Iota

## 类型声明

## 类型参数声明

## 变量声明

## 短变量声明

## 函数声明

## 方法声明


# 表达式

## 操作

## 限定的标识符

## Composite literals

## Function literals

## 基础表达式

## Selectors

## 方法表达式

## 方法值

## 索引表达式

## 切片表达式

## 类型断言

## 调用

## 将参数传递给

## Instantiations

## 类型推断

## Operators

## 算术运算符

## 比较运算符

## 逻辑运算符

## 地址运算符

## 接受运算符

## 转换

## 常量表达式

## 执行顺序

# 语句

## 终止语句

## 空语句

## 标记语句

## 表达式语句

## 发送语句

## 增减声明

## 复制语句

## If 语句

## switch 语句

## For 语句

## Go 语句

## 选择语句

## 返回语句

## 中断语句

## Continue 语句

## goto 语句

## Fallthrough 语句

## defer 语句

# 内置函数

## 追加或复制切片

# 包

# 程序初始化和执行

# 错误

# 运行时恐慌

# 系统考虑

# 附录
