---
title: "Go Code Review Comments"
date: 2021-04-18T13:04:21+08:00
isCJKLanguage: true
Description: "本页面收集了Go代码评审的常见评论。并提供了简单的细节解释，可以代码评审时引用。这只是一个常见错误的列表，并不是一个全面的风格指南。你可以把这些当作对 [Effective Go](https://golang.org/doc/effective_go.html)的补充."
Tags: ["Go", "code review"]
Categories: ["Go"]
---

原文地址：https://github.com/golang/go/wiki/CodeReviewComments


* [Gofmt](#gofmt)
* [声明的注释应该是句子](#声明的注释应该是句子)
* [Contexts](#contexts)
* [复制](#复制)
* [Crypto Rand](#crypto-rand)
* [声明空数组](#声明空切片)
* [文档注释](#文档注释)
* [不要使用panic](#不要使用panic)
* [错误字符串](#错误字符串)
* [例子](#例子)
* [Goroutine 生命周期](#Goroutine-生命周期)
* [处理错误](#处理错误)
* [导入](#导入)
* [作为空的导入](#作为空的导入)
* [作为句点的导入](#作为句点的导入)
* [带内错误](#带内错误)
* [缩进错误处理流](#缩进错误处理流)
* [首字母缩写](#首字母缩写)
* [Interfaces](#interfaces)
* [Line Length](#line-length)
* [混合大小写(驼峰)](#混合大小写(驼峰))
* [具名返回参数](#具名返回参数)
* [Naked Returns](#naked-returns)
* [Package Comments](#package-comments)
* [Package Names](#package-names)
* [Pass Values](#pass-values)
* [Receiver Names](#receiver-names)
* [Receiver Type](#receiver-type)
* [Synchronous Functions](#synchronous-functions)
* [有意义的测试错误信息](#有意义的测试错误信息)
* [变量名](#变量名)


## Gofmt

在你的代码上运行[gofmt](https://golang.org/cmd/gofmt/)可以解决大多数固定的代码风格问题。几乎所有的Go代码都在使用 `gofmt`。本文档的其他部分会讨论非固定格式的代码风格问题。

你也可以使用[goimports](https://godoc.org/golang.org/x/tools/cmd/goimports), 它是 `gofmt` 的超集，可以根据需要添加或删除 import 行。

## 注释应该是句子

参见 https://golang.org/doc/effective_go.html#commentary 。针对声明的代码注释应该是完整的句子，即使它看起来有些多余。 这样你生成 godoc 文档可读性更好。注释应该以所描述事物的名称开始，并以句号结束:

```go
// Request represents a request to run a command.
type Request struct { ...

// Encode writes the JSON encoding of req to w.
func Encode(w io.Writer, req *Request) { ...
```

## Contexts

`context.Context` 类型的 Value 可以携带跨越API或处理逻辑边界的安全凭据，追踪信息，截止日期或取消信号。在 Go 程序里，Context 显式地在整个函数调用链中传递，从接受 RPC 或者 HTTP 调用，一直到对外的 RPC 或者 HTTP 调用。

大多数使用Context的函数，都应该把它作为第一个参数:

```go
func F(ctx context.Context, /* other arguments */) {}
```

A function that is never request-specific may use context.Background(),
but err on the side of passing a Context even if you think you don't need
to. The default case is to pass a Context; only use context.Background()
directly if you have a good reason why the alternative is a mistake.

Don't add a Context member to a struct type; instead add a ctx parameter
to each method on that type that needs to pass it along. The one exception
is for methods whose signature must match an interface in the standard library
or in a third party library.

Don't create custom Context types or use interfaces other than Context in function signatures.

If you have application data to pass around, put it in a parameter,
in the receiver, in globals, or, if it truly belongs there, in a Context value.

上下文是不可变的，所以可以放心把同一个ctx传递给多个调用，如果这些调用共享截止时间，取消信号，凭证，追踪信息等。

## 复制

 为了避免意外的别名，在从另一个 package 复制结构时，要尤其小心。 例如，bytes.Buffer 类型包含一个 `[]byte` 切片。如果你复制一个 `Buffer`， 副本中的切片只是原始数组的别名，从而导致后续方法调用可能会有意外的效果。 一般来说，如果类型 `T` 的方法，关联的是它的指针类型 `*T` ，那么不要复制 `T` 的值。


## Crypto Rand


不要使用 `math/rand` 来生成密钥，即使是一次性的密钥。 在没有种子的情况下，生成的密钥是完全可以预测的。使用 `time.Nanoseconds()` 作为种子，也只带来少量的熵。你应该使用 `crypto/rand` 的 Reader， 如果需要的是文本，可以打印为十六进制或base64:


``` go
import (
	"crypto/rand"
	// "encoding/base64"
	// "encoding/hex"
	"fmt"
)

func Key() string {
	buf := make([]byte, 16)
	_, err := rand.Read(buf)
	if err != nil {
		panic(err)  // out of randomness, should never happen
	}
	return fmt.Sprintf("%x", buf)
	// or hex.EncodeToString(buf)
	// or base64.StdEncoding.EncodeToString(buf)
}
```

## 声明空切片

当你创建空切片时推荐使用：

```go
var t []string
```

而不是

```go
t := []string{}
```
前者是一个值为nil的切片。后者只是长度为零，但不是nil。他们在功能上是完全相同的。他们的 `len` and `cap`都是零，但是nil切片是更加推荐的方式。

请注意，在有些特殊情况，更推荐使用不为nil，但长度为零但切片，例如编码
JSON对象但时候（一个`nil`切片被编码成`null`，但是`[]string{}`被比编码成JSON数组`[]`）。

在设计接口时，避免区分nil片和非nil、零长度的片，因为这可能会导致微妙的编程错误。

When designing interfaces, avoid making a distinction between a nil slice and a non-nil, zero-length slice, as this can lead to subtle programming errors.

Go中关于nil的更多讨论可以看一下 Francesc Campoy 的视频 [Understanding Nil](https://www.youtube.com/watch?v=ynoY2xz-F8s).

## 文档注释

所有顶级的、可导出的名称，都应该有文档注释，重要的未导出类型或函数声明也应该如此。有关注释约定的更多信息，请参见https://golang.org/doc/effective_go.html#commentary。

## 不要使用panic

参见 https://golang.org/doc/effective_go.html#errors. 对于正常的错误处理，不要使用panic。使用错误和多个返回值。

## 错误字符串

错误字符串不应该大写(除非以专有名词或首字母缩写开头)，也不应该以标点符号结尾，因为它们通常在其他上下文后面打印。也就是说，使用 

```go
fmt.Errorf("something bad")
```
而不是
```go
fmt.Errorf("Something bad.")
```
这样
```go
log.Printf("Reading %s: %v", filename, err)
```
的输出才不会出现大写字母。不过这不适用于日志记录，日志记录是隐式面向行的，不会在其他消息中合并。

## 例子

添加新的 package 时，同时添加一些使用示例，这些示例应该是可运行的，或者一个可以演示的，包含了完整调用过程的测试用例。更多内容参见 [testable Example() functions](https://blog.golang.org/examples).

## Goroutine 生命周期

当你起一个 goroutine 时，你要清楚它是否需要退出，或者什么时候退出。

Goroutines can leak by blocking on channel sends or receives: the garbage collector
will not terminate a goroutine even if the channels it is blocked on are unreachable.

Even when goroutines do not leak, leaving them in-flight when they are no longer
needed can cause other subtle and hard-to-diagnose problems. Sends on closed channels panic. Modifying still-in-use inputs "after the result isn't needed" can still lead to data races. And leaving goroutines in-flight for arbitrarily long can lead to unpredictable memory usage.
 
尽量保持并发代码足够简单，这样goroutine的生存期就会很明显。如果这是不可行的，请写好文档，记录goroutines 何时退出以及为什么退出。


## 处理错误

参见 https://golang.org/doc/effective_go.html#errors. 不要使用 `_` 变量丢弃错误。如果函数返回一个错误，明确地检查错误，确保函数执行成功。要么处理错误，要么向上一级返回它，又或者，确定是一个无法处理的异常的话，选择 `panic`。

## 导入

不要重命名导入的包名，除非为了避免包名冲突；好的包名不应该需要重命名。包名冲突时，优先重命名`最局部的`包名，或者项目特有的导入包。

多行导入使用空行对他们分组。标准库的包总是放在第一个组。

```go
package main

import (
	"fmt"
	"hash/adler32"
	"os"

	"appengine/foo"
	"appengine/user"

	"github.com/foo/bar"
	"rsc.io/goversion/version"
)
```

<a href="https://godoc.org/golang.org/x/tools/cmd/goimports">goimports</a> 可以自动为你做好分组排序。

## 作为空的导入

仅因其副作用而导入的包(使用语法 `import _ "pkg"` )应仅在程序的 `main` 包或测试包导入。


## 作为句点的导入

在写测试代码的时候，由于循环依赖的问题，导致包不能导入，这是导入成句点这种形式可能会有用：

```go
package foo_test

import (
	"bar/testutil" // also imports "foo"
	. "foo"
)
```
在这种情况下，这个测试文件不能在 foo 包中，因为它导入了 `bar/testutil`，而 `bar/testutil` 又导入了 foo。所以我们使用 `import .` 的形式，让文件假装是 foo 包的一部分，即使它不是。除了这一种情况，不要在你的程序中使用 `import .`。它使程序更难阅读，因为不清楚像 `Quux` 这样的名称是当前包中的顶级标识符还是导入包中的顶级标识符。

## 带内错误

在C和其他类似语言中，经常让函数返回 -1 表示信号错误，或者返回 null 表示没有结果：

```go
// Lookup 返回key对应的value，如果没有的话则返回 ""
func Lookup(key string) string

// Failing to check a for an in-band error value can lead to bugs:
Parse(Lookup(key))  // returns "parse failure for value" instead of "no value for key"
```
Go 的函数可以返回多个值，这是一个更好的方案。


Instead of requiring clients to check for an in-band error value, a function should return
an additional value to indicate whether its other return values are valid. This return
value may be an error, or a boolean when no explanation is needed.
It should be the final return value.

``` go
// Lookup returns the value for key or ok=false if there is no mapping for key.
func Lookup(key string) (value string, ok bool)
```
这防止了调用者错误地使用调用结果：

``` go
Parse(Lookup(key))  // 编译错误
```
并鼓励更健壮和可读的代码:
And encourages more robust and readable code:

``` go
value, ok := Lookup(key)
if !ok {
	return fmt.Errorf("no value for %q", key)
}
return Parse(value)
```

该规则适用于可导出的函数，对于不可到处的函数，也是有用的。

如果 nil， ""， 0 或者 -1 是有意义的返回结果，也就是说，函数的调用者不需要将其与其他值区别处理，那么函数直接返回这些值也是可以的。

Go 的函数标准库里，比如 `strings` 包，里面就有一些函数，直接返回了带内错误。 This greatly simplifies string-manipulation
code at the cost of requiring more diligence from the programmer.通常，Go代码应该返回额外的错误值。

## 缩进错误处理流

尽量将常规代码路径缩进到最小，并缩进错误处理，先处理它。通过允许可视化地快速扫描正常路径，提高了代码的可读性。例如，不要这样写:

```go
if err != nil {
	// error handling
} else {
	// normal code
}
```

而应该写成:

```go
if err != nil {
	// error handling
	return // or continue, etc.
}
// normal code
```

如果' If '语句有一个初始化语句，例如:

```go
if x, err := f(); err != nil {
	// error handling
	return
} else {
	// use x
}
```

这可能需要将短变量声明移到单独的行中，这样缩进的错误处理代码后面的正常代码逻辑才能使用变量 `x`:

```go
x, err := f()
if err != nil {
	// error handling
	return
}
// use x
```

## 首字母缩写

名字中的单词是缩写或首字母缩写(例如:“URL”或“NATO”)是，大小写要保持一致。例如，"URL" 应该写成 "URL" 或者 "url"，而不要写成"Url"。作为名字的一部分时，写作 "urlPony"，或者 "URLPony". 又例如，ServeHttp 应该写成 ServeHTTP。对于具有多个初始化“单词”的标识符，可以使用例如“xmlHTTPRequest”或“xmlHTTPRequest”。

当“ID”是“identifier”的缩写时，这条规则也适用，所以写成“appID”，而不是“appId”。

protocol buffer 编译器生成的代码不受此规则约束。人类编写的代码比机器编写的代码有更高的标准。

## Interfaces

Go接口通常属于使用 接口类型，而不是实现这些值的包。的 实现包应该返回具体的(通常是指针或结构) 类型:这样，新的方法可以添加到实现中而不需要 需要广泛的重构。

Go interfaces generally belong in the package that uses values of the
interface type, not the package that implements those values. The
implementing package should return concrete (usually pointer or struct)
types: that way, new methods can be added to implementations without
requiring extensive refactoring.

Do not define interfaces on the implementor side of an API "for mocking";
instead, design the API so that it can be tested using the public API of
the real implementation.

Do not define interfaces before they are used: without a realistic example
of usage, it is too difficult to see whether an interface is even necessary,
let alone what methods it ought to contain.

``` go
package consumer  // consumer.go

type Thinger interface { Thing() bool }

func Foo(t Thinger) string { … }
```

``` go
package consumer // consumer_test.go

type fakeThinger struct{ … }
func (t fakeThinger) Thing() bool { … }
…
if Foo(fakeThinger{…}) == "x" { … }
```

``` go
// DO NOT DO IT!!!
package producer

type Thinger interface { Thing() bool }

type defaultThinger struct{ … }
func (t defaultThinger) Thing() bool { … }

func NewThinger() Thinger { return defaultThinger{ … } }
```

Instead return a concrete type and let the consumer mock the producer implementation.
``` go
package producer

type Thinger struct{ … }
func (t Thinger) Thing() bool { … }

func NewThinger() Thinger { return Thinger{ … } }
```

## Line Length

There is no rigid line length limit in Go code, but avoid uncomfortably long lines.
Similarly, don't add line breaks to keep lines short when they are more readable long--for example,
if they are repetitive.

Most of the time when people wrap lines "unnaturally" (in the middle of function calls or
function declarations, more or less, say, though some exceptions are around), the wrapping would be
unnecessary if they had a reasonable number of parameters and reasonably short variable names.
Long lines seem to go with long names, and getting rid of the long names helps a lot.

In other words, break lines because of the semantics of what you're writing (as a general rule)
and not because of the length of the line. If you find that this produces lines that are too long,
then change the names or the semantics and you'll probably get a good result.

This is, actually, exactly the same advice about how long a function should be. There's no rule
"never have a function more than N lines long", but there is definitely such a thing as too long
of a function, and of too stuttery tiny functions, and the solution is to change where the function
boundaries are, not to start counting lines.

## 混合大小写(驼峰)

参见 https://golang.org/doc/effective_go.html#mixed-caps. 这可能与其他语言的约定不同，比如不可导出的常量应该命名为 `maxLength`，而不是 `MaxLength` 或 `MAX_LENGTH`.

Also see [Initialisms](https://github.com/golang/go/wiki/CodeReviewComments#initialisms).

## 具名返回参数

Consider what it will look like in godoc.  Named result parameters like:

```go
func (n *Node) Parent1() (node *Node) {}
func (n *Node) Parent2() (node *Node, err error) {}
```

will stutter in godoc; better to use:

```go
func (n *Node) Parent1() *Node {}
func (n *Node) Parent2() (*Node, error) {}
```

On the other hand, if a function returns two or three parameters of the same type, 
or if the meaning of a result isn't clear from context, adding names may be useful
in some contexts. Don't name result parameters just to avoid declaring a var inside
the function; that trades off a minor implementation brevity at the cost of
unnecessary API verbosity.


```go
func (f *Foo) Location() (float64, float64, error)
```

is less clear than:

```go
// Location returns f's latitude and longitude.
// Negative values mean south and west, respectively.
func (f *Foo) Location() (lat, long float64, err error)
```

Naked returns are okay if the function is a handful of lines. Once it's a medium
sized function, be explicit with your return values. Corollary: it's not worth it
to name result parameters just because it enables you to use naked returns.
Clarity of docs is always more important than saving a line or two in your function.

Finally, in some cases you need to name a result parameter in order to change
it in a deferred closure. That is always OK.


## Naked Returns

参见 [具名返回参数](#具名返回参数).

## 包注释

Package comments, like all comments to be presented by godoc, must appear adjacent to the package clause, with no blank line.

```go
// Package math provides basic constants and mathematical functions.
package math
```

```go
/*
Package template implements data-driven templates for generating textual
output such as HTML.
....
*/
package template
```

For "package main" comments, other styles of comment are fine after the binary name (and it may be capitalized if it comes first), For example, for a `package main` in the directory `seedgen` you could write:

``` go
// Binary seedgen ...
package main
```
or
```go
// Command seedgen ...
package main
```
or
```go
// Program seedgen ...
package main
```
or
```go
// The seedgen command ...
package main
```
or
```go
// The seedgen program ...
package main
```
or
```go
// Seedgen ..
package main
```

These are examples, and sensible variants of these are acceptable.

Note that starting the sentence with a lower-case word is not among the
acceptable options for package comments, as these are publicly-visible and
should be written in proper English, including capitalizing the first word
of the sentence. When the binary name is the first word, capitalizing it is
required even though it does not strictly match the spelling of the
command-line invocation.

See https://golang.org/doc/effective_go.html#commentary for more information about commentary conventions.

## Package Names

All references to names in your package will be done using the package name, 
so you can omit that name from the identifiers. For example, if you are in package chubby, 
you don't need type ChubbyFile, which clients will write as `chubby.ChubbyFile`.
Instead, name the type `File`, which clients will write as `chubby.File`.
Avoid meaningless package names like util, common, misc, api, types, and interfaces. See http://golang.org/doc/effective_go.html#package-names and
http://blog.golang.org/package-names for more.

## Pass Values

Don't pass pointers as function arguments just to save a few bytes.  If a function refers to its argument `x` only as `*x` throughout, then the argument shouldn't be a pointer.  Common instances of this include passing a pointer to a string (`*string`) or a pointer to an interface value (`*io.Reader`).  In both cases the value itself is a fixed size and can be passed directly.  This advice does not apply to large structs, or even small structs that might grow.

## Receiver Names

The name of a method's receiver should be a reflection of its identity; often a one or two letter abbreviation of its type suffices (such as "c" or "cl" for "Client"). Don't use generic names such as "me", "this" or "self", identifiers typical of object-oriented languages that gives the method a special meaning. In Go, the receiver of a method is just another parameter and therefore, should be named accordingly. The name need not be as descriptive as that of a method argument, as its role is obvious and serves no documentary purpose. It can be very short as it will appear on almost every line of every method of the type; familiarity admits brevity. Be consistent, too: if you call the receiver "c" in one method, don't call it "cl" in another.

## Receiver Type

Choosing whether to use a value or pointer receiver on methods can be difficult, especially to new Go programmers.  If in doubt, use a pointer, but there are times when a value receiver makes sense, usually for reasons of efficiency, such as for small unchanging structs or values of basic type. Some useful guidelines:

  * If the receiver is a map, func or chan, don't use a pointer to them. If the receiver is a slice and the method doesn't reslice or reallocate the slice, don't use a pointer to it.
  * If the method needs to mutate the receiver, the receiver must be a pointer.
  * If the receiver is a struct that contains a sync.Mutex or similar synchronizing field, the receiver must be a pointer to avoid copying.
  * If the receiver is a large struct or array, a pointer receiver is more efficient.  How large is large?  Assume it's equivalent to passing all its elements as arguments to the method.  If that feels too large, it's also too large for the receiver.
  * Can function or methods, either concurrently or when called from this method, be mutating the receiver? A value type creates a copy of the receiver when the method is invoked, so outside updates will not be applied to this receiver. If changes must be visible in the original receiver, the receiver must be a pointer.
  * If the receiver is a struct, array or slice and any of its elements is a pointer to something that might be mutating, prefer a pointer receiver, as it will make the intention more clear to the reader.
  * If the receiver is a small array or struct that is naturally a value type (for instance, something like the time.Time type), with no mutable fields and no pointers, or is just a simple basic type such as int or string, a value receiver makes sense.  A value receiver can reduce the amount of garbage that can be generated; if a value is passed to a value method, an on-stack copy can be used instead of allocating on the heap. (The compiler tries to be smart about avoiding this allocation, but it can't always succeed.) Don't choose a value receiver type for this reason without profiling first.
  * Don't mix receiver types. Choose either pointers or struct types for all available methods. 
  * Finally, when in doubt, use a pointer receiver.

## 同步函数

更推荐使用同步函数，而非异步函数。同步函数是那些直接返回结果，或者在返回结果之前，完成所有回调或信道操作的函数。

Synchronous functions keep goroutines localized within a call, making it easier to reason about their lifetimes and avoid leaks and data races. They're also easier to test: the caller can pass an input and check the output without the need for polling or synchronization.

如果调用者需要更高的并发性，他们可以在其他 goroutine 中来调用该函数。 But it is quite difficult - sometimes impossible - to remove unnecessary concurrency at the caller side.

## 有意义的测试错误信息

测试代码失败的时候，应该给出有用的信息，说明出了什么问题，输入是什么，期望得到什么，预期得到什么。你可能会写一些 `assertFoo` 工具函数，不过也要确保这些函数输出有用的错误信息。写测试代码时，要假定调试这些测试代码的人不是你，也不是你的团队成员。一个典型的Go测试失败如下:

```go
if got != tt.want {
	t.Errorf("Foo(%q) = %d; want %d", tt.in, got, tt.want) // or Fatalf, if test can't test anything more past this point
}
```

注意这里的顺序是 `actual != expected`, 输出的错误信息也是这个顺序。有些测试框架推荐相反的顺序：`0 != x`, `"expected 0, got x"`, 等等。Go 里面不这样写。

如果测试需要大量输入，那么你可能需要编写表格驱动的测试。

如果一个测试辅助函数需要调用多次，多次调用只是传不同参数，那么为了消除错误输出信息的歧义，方便运行测试的人确定测试失败位置，一种常见的方式，是用不同的 `TestFoo` 函数包装每次调用，并根据不同的输入命名对应的测试函数:

```go
func TestSingleValue(t *testing.T) {
	testHelper(t, []int{80}) 
}

func TestNoValues(t *testing.T) {
	testHelper(t, []int{}) 
}
```

无论如何，向调试您代码的人提供有用的消息，这个责任都在你身上。

## 变量名

变量名称应该尽可能的短。尤其是只在小范围使用短局部变量，例如 `c` 替换 `lineCount`。 `i` 替换 `sliceIndex`

基本规则：使用变量名的地方距离声明它的地方越远，变量名描述的越具体。
对于方法的 receiver，一两个字母就够了。常见的变量，例如循环的下标和接收值的变量名，一个字母就够了（`i`, `r`）。其他不常用的或者全局变量，则需要描述的更具体一些。