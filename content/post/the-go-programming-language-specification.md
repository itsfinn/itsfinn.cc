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

Go 官方编程语言规范中文翻译版本（Go 版本 go1.23 (2024年8月24日)）[翻译进度 14%]
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

浮点字面量是浮点常量的十进制或十六进制表示形式。

十进制浮点文本由整数部分（十进制数字）、小数点、小数部分（十进制数字）和指数部分（e 或 E 后跟可选符号和十进制数字）组成。整数部分或小数部分之一可以省略;小数点后的一个或指数部分可以省略。指数值 exp 将尾数（整数和小数部分）缩放 10exp。

十六进制浮点文本由 0x 或 0X 前缀、整数部分（十六进制数字）、基数点、小数部分（十六进制数字）和指数部分（p 或 P 后跟可选符号和十进制数字）组成。整数部分或小数部分之一可以省略;基数点也可以省略，但指数部分是必需的。

（此语法与 IEEE 754-2008 §5.12.3 中给出的语法匹配。指数值 exp 将尾数（整数和小数部分）缩放 2exp [Go 1.13]。

为了便于阅读，下划线字符 _ 可能出现在基本前缀之后或连续数字之间;此类下划线不会更改字面值。

```float_lit         = decimal_float_lit | hex_float_lit .

decimal_float_lit = decimal_digits "." [ decimal_digits ] [ decimal_exponent ] |
                    decimal_digits decimal_exponent |
                    "." decimal_digits [ decimal_exponent ] .
decimal_exponent  = ( "e" | "E" ) [ "+" | "-" ] decimal_digits .

hex_float_lit     = "0" ( "x" | "X" ) hex_mantissa hex_exponent .
hex_mantissa      = [ "_" ] hex_digits "." [ hex_digits ] |
                    [ "_" ] hex_digits |
                    "." hex_digits .
hex_exponent      = ( "p" | "P" ) [ "+" | "-" ] decimal_digits .
```

```
0.
72.40
072.40       // == 72.40
2.71828
1.e+0
6.67428e-11
1E6
.25
.12345E+5
1_5.         // == 15.0
0.15e+0_2    // == 15.0

0x1p-2       // == 0.25
0x2.p10      // == 2048.0
0x1.Fp+0     // == 1.9375
0X.8p-0      // == 0.5
0X_1FFFP-16  // == 0.1249847412109375
0x15e-2      // == 0x15e - 2 (integer subtraction)

0x.p1        // invalid: mantissa has no digits
1p-2         // invalid: p exponent requires hexadecimal mantissa
0x1.5e-2     // invalid: hexadecimal mantissa requires p exponent
1_.5         // invalid: _ must separate successive digits
1._5         // invalid: _ must separate successive digits
1.5_e1       // invalid: _ must separate successive digits
1.5e_1       // invalid: _ must separate successive digits
1.5e1_       // invalid: _ must separate successive digits
```

## 虚数字面量

虚数文本表示复数常量的虚数部分。它由一个整数或浮点文本组成，后跟小写字母 i。虚数字面的值是相应的整数或浮点字面乘以虚数单位 *i* [Go 1.13]

```
imaginary_lit = (decimal_digits | int_lit | float_lit) "i" .
```

为了向后兼容，完全由十进制数字（可能还有下划线）组成的虚构文本的整数部分被视为十进制整数，即使它以前导 0 开头。

```
0i
0123i         // == 123i for backward-compatibility
0o123i        // == 0o123 * 1i == 83i
0xabci        // == 0xabc * 1i == 2748i
0.i
2.71828i
1.e+0i
6.67428e-11i
1E6i
.25i
.12345E+5i
0x1p-2i       // == 0x1p-2 * 1i == 0.25i
```

## rune字面量

A rune literal represents a rune constant, an integer value identifying a Unicode code point. A rune literal is expressed as one or more characters enclosed in single quotes, as in 'x' or '\n'. Within the quotes, any character may appear except newline and unescaped single quote. A single quoted character represents the Unicode value of the character itself, while multi-character sequences beginning with a backslash encode values in various formats.
rune 文本表示 rune 常量，即标识 Unicode 代码点的整数值。rune 文本表示为用单引号括起来的一个或多个字符，如 'x' 或 '\n'。在引号中，可以显示除换行符和未转义的单引号之外的任何字符。单引号引起来的一个字符表示字符本身的 Unicode 值，而以反斜杠开头的多字符序列以各种格式对值进行编码。

最简单的形式就是单引号引起来的一个字符，由于 Go 源文本是用 UTF-8 编码的 Unicode 字符，因此多个 UTF-8 编码的字节可能表示一个整数值。例如，文本 'a' 包含一个字节，表示文本 a，Unicode U+0061，值 0x61，而 'ä' 包含两个字节 （0xc30xa4），表示文本 a-dieresis，U+00E4，值 0xe4。

几个反斜杠转义允许将任意值编码为 ASCII 文本。有四种方法可以将整数值表示为数字常量：\x 后跟两个十六进制数字;\u 后跟四个十六进制数字;\U 后跟 8 个十六进制数字，一个普通的反斜杠 \ 后跟 3 个八进制数字。在每种情况下，文本的值都是相应基数中的数字表示的值。

Although these representations all result in an integer, they have different valid ranges. Octal escapes must represent a value between 0 and 255 inclusive. Hexadecimal escapes satisfy this condition by construction. The escapes \u and \U represent Unicode code points so within them some values are illegal, in particular those above 0x10FFFF and surrogate halves.

尽管这些表示形式都生成一个整数，但它们具有不同的有效范围。八进制转义必须表示介于 0 和 255 之间（含 0 和 255）的值。十六进制转义通过构造满足此条件。转义符 \u 和 \U 表示 Unicode 码位，因此在它们中，某些值是非法的，特别是 0x10FFFF 和代理部分上方的值。

在反斜杠之后，某些单字符转义表示特殊值：

```
\a   U+0007 alert or bell
\b   U+0008 backspace
\f   U+000C form feed
\n   U+000A line feed or newline
\r   U+000D carriage return
\t   U+0009 horizontal tab
\v   U+000B vertical tab
\\   U+005C backslash
\'   U+0027 single quote  (valid escape only within rune literals)
\"   U+0022 double quote  (valid escape only within string literals)
```

在 rune 字面量中反斜杠后面的无法识别的字符是非法的。

```
rune_lit         = "'" ( unicode_value | byte_value ) "'" .
unicode_value    = unicode_char | little_u_value | big_u_value | escaped_char .
byte_value       = octal_byte_value | hex_byte_value .
octal_byte_value = `\` octal_digit octal_digit octal_digit .
hex_byte_value   = `\` "x" hex_digit hex_digit .
little_u_value   = `\` "u" hex_digit hex_digit hex_digit hex_digit .
big_u_value      = `\` "U" hex_digit hex_digit hex_digit hex_digit
                           hex_digit hex_digit hex_digit hex_digit .
escaped_char     = `\` ( "a" | "b" | "f" | "n" | "r" | "t" | "v" | `\` | "'" | `"` ) .
```

```
'a'
'ä'
'本'
'\t'
'\000'
'\007'
'\377'
'\x07'
'\xff'
'\u12e4'
'\U00101234'
'\''         // rune literal containing single quote character
'aa'         // illegal: too many characters
'\k'         // illegal: k is not recognized after a backslash
'\xa'        // illegal: too few hexadecimal digits
'\0'         // illegal: too few octal digits
'\400'       // illegal: octal value over 255
'\uDFFF'     // illegal: surrogate half
'\U00110000' // illegal: invalid Unicode code point
```

## 字符串字面量

字符串字面量表示通过连接字符序列获得的字符串常量。有两种形式：原始字符串字面量和解释的字符串字面量。

原始字符串字面量是反引号之间的字符序列，如 \`foo\` 的。 两个反引号中间可以出现除了反引号以外的任何字符。

原始字符串字面量的值是两个反引号中间未解释（隐式 UTF-8 编码）字符组成的字符串; 特别注意是，反斜杠没有特殊含义，字符串可以包含换行符。

原始字符串文字中的回车符 （'\r'） 从原始字符串值中丢弃。

解释的字符串文字是双引号之间的字符序列，如 "bar" 一样。在引号中，可以出现除换行符和未转义的双引号之外的任何字符。引号之间的文本构成了 字面量的值，反斜杠转义解释为 rune 字面量中的含义（除了 \' 是非法的，\" 是合法的），具有相同的限制。三位数八进制 （\nnn） 和两位十六进制 （\xnn） 转义表示结果字符串的单个字节;所有其他转义符表示单个字符的（可能是多字节的）UTF-8 编码。因此，在字符串文本中，\377 和 \xFF 表示值为 0xFF=255 的单个字节，而 ÿ、\u00FF、\U000000FF 和 \xc3\xbf 表示字符 U+00FF 的 UTF-8 编码0xc30xbf的两个字节。

```
string_lit             = raw_string_lit | interpreted_string_lit .
raw_string_lit         = "`" { unicode_char | newline } "`" .
interpreted_string_lit = `"` { unicode_value | byte_value } `"` .
```

```
`abc`                // same as "abc"
`\n
\n`                  // same as "\\n\n\\n"
"\n"
"\""                 // same as `"`
"Hello, world!\n"
"日本語"
"\u65e5本\U00008a9e"
"\xff\u00FF"
"\uD800"             // illegal: surrogate half
"\U00110000"         // illegal: invalid Unicode code point
```

# 常量

有布尔常量、rune 常量、整数常量、浮点常量、复数常量和字符串常量。rune 常量、整数常量、浮点常量、复数常量统称为数值常量。

常量值可以以下形式表示，包括 rune、整数、浮点、虚数或字符串字面量、表示常量的标识符、常量表达式、结果为常量的转换或某些内置函数（如 min 或 max）的结果值、unsafe.Sizeof() 应用于某些值，cap 或 len 应用于某些表达式，real 和 imag 应用于复数常量，complex 应用于数值常量。布尔值真值由预先声明的常量 true 和 false 表示。预先声明的标识符 iota 表示一个整数常量。

通常，复数常量是常量表达式的一种形式，将在常量表达式的部分中讨论。

数字常量表示任意精度的精确值，并且不会溢出。因此，不存在表示 IEEE 754 负零、无穷大和非数字值的常量。

常量可以是类型化的，也可以是非类型的。文本常量、true、false、iota 和某些仅包含无类型常量操作数的常量表达式是无类型的。

常量可以通过常量声明 或转换 显式地指定类型，或者在 变量声明或 赋值语句中使用或作为表达式中的操作数时隐式地指定类型。如果常量值不能表示为相应类型的值，则会出现错误。如果类型是类型参数，则常量将转换为类型参数的非常量值。

非类型化常量具有默认类型，该类型是常量在需要类型化值的上下文中隐式转换为的类型，例如，在没有显式类型的短变量声明（如 i ：= 0）中。无类型常量的默认类型分别为 bool、rune、int、float64、complex128 或 string，具体取决于它是布尔值、rune、整数、浮点数、complex 还是字符串常量。

实现限制：尽管语言中的数字常量具有任意精度，但编译器可以使用精度有限的内部表示来实现它们。也就是说，每个实现都必须：

- 用至少 256 位来表示整数常量。
- 用至少 256 位的尾数和至少 16 位的有符号二进制指数表示浮点常数（包括复数常数的各个部分）。
- 如果无法准确表示整数常数，则会出现错误。
- 如果由于溢出而无法表示浮点数或复数常数，则会出错。
- 如果由于精度限制而无法表示浮点数或复数常数，则四舍五入到最接近的可表示常数。

这些要求既适用于文本常量，也适用于常量表达式 的求值结果。

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
