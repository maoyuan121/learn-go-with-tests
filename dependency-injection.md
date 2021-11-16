# Dependency Injection

**[You can find all the code for this chapter here](https://github.com/quii/learn-go-with-tests/tree/main/di)**

在编程社区中，对于依赖注入有很多误解。希望本指南能告诉你如何做

* 你不需要一个框架
* 它不会使你的设计过于复杂
* 它促进测试
* 它允许您编写伟大的、通用的函数。

我们想要编写一个函数来问候某人，就像我们在 hello-world 一章中所做的那样，但这次我们将测试 _actual printing_。

回顾一下，有下面一个函数

```go
func Greet(name string) {
	fmt.Printf("Hello, %s", name)
}
```

但我们如何测试呢?调用的 `fmt.Printf` 打印到 stdout，这对于我们使用测试框架来说是相当困难的。

我们需要做的是能够**注入** \(这只是传递\的一个花哨的词)打印依赖。

我们的函数不需要关心打印发生在哪里或如何发生，所以我们应该接受接口而不是具体类型。

如果我们这样做，我们就可以将实现改为打印到我们控制的东西，这样我们就可以测试它。
在“现实生活”中，您将注入一些写入 stdout 的内容。

如果你看一下 `fmt.Printf` 你可以看到一个让我们 hook 的方法

```go
// It returns the number of bytes written and any write error encountered.
func Printf(format string, a ...interface{}) (n int, err error) {
	return Fprintf(os.Stdout, format, a...)
}
```

有意思。`Printf` 里面调用了 `Fprintf`，并且传递进去了一个 `os.Stdout`。

`os.Stdout` 到底是什么？`Fprintf` 期望传递给它的第一个参数是什么?

```go
func Fprintf(w io.Writer, format string, a ...interface{}) (n int, err error) {
	p := newPrinter()
	p.doPrintf(format, a)
	n, err = w.Write(p.buf)
	p.free()
	return
}
```

一个 `io.Writer`

```go
type Writer interface {
	Write(p []byte) (n int, err error)
}
```

当你编写更多的 Go 代码时，你会发现这个 interface 经常出现，因为它是一个用于“把数据放到某个地方”的通用接口。

所以我们知道在底层，我们最终会使用 `Writer` 来发送我们的问候。
让我们使用这个现有的抽象来让我们的代码更可测试和更可重用。

## Write the test first

```go
func TestGreet(t *testing.T) {
	buffer := bytes.Buffer{}
	Greet(&buffer, "Chris")

	got := buffer.String()
	want := "Hello, Chris"

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}
```

`bytes` 包中的 `buffer` 类型实现了 `Writer` 接口。

所以我们会在测试中使用它作为 `Writer` 发送然后我们可以在调用 `Greet` 之后检查写入了什么

## Try and run the test

测试编译失败

```text
./di_test.go:10:7: too many arguments in call to Greet
    have (*bytes.Buffer, string)
    want (string)
```

## Write the minimal amount of code for the test to run and check the failing test output

监听编译器并修复问题。

```go
func Greet(writer *bytes.Buffer, name string) {
	fmt.Printf("Hello, %s", name)
}
```

`Hello, Chris di_test.go:16: got '' want 'Hello, Chris'`

测试失败。注意，名字会被打印出来，但它会被 stdout。

## Write enough code to make it pass

在我们的测试中，使用 wrtier 将问候语发送到 buffer。
记住 `fmt.Fprintf` 就像 `fmt.Printf`，但它接受一个 `Writer` 来发送字符串，
而 `fmt.Printf` 默认为标准输出。

```go
func Greet(writer *bytes.Buffer, name string) {
	fmt.Fprintf(writer, "Hello, %s", name)
}
```

测试现在可以通过了。

## Refactor

之前编译器告诉我们传入一个指向 `bytes.Buffer` 的指针。这在技术上是正确的，但不是很有用。

为了演示这一点，请尝试将 `Greet` 函数连接到 Go 应用程序中，以便将其打印到 stdout。


```go
func main() {
	Greet(os.Stdout, "Elodie")
}
```

`./di.go:14:7: cannot use os.Stdout (type *os.File) as type *bytes.Buffer in argument to Greet`

正如前面讨论的 `fmt.Fprintf` 允许传入一个 `io.Writer`，我们都知道 `os.Stdout` 和 `bytes.Buffer` 实现了它。

如果我们更改代码以使用更通用的接口，我们现在可以在测试和应用程序中使用它。

```go
package main

import (
    "fmt"
    "os"
    "io"
)

func Greet(writer io.Writer, name string) {
    fmt.Fprintf(writer, "Hello, %s", name)
}

func main() {
	Greet(os.Stdout, "Elodie")
}
```

## More on io.Writer

还有哪些地方可以使用 `io.Writer` 写入数据?我们的 `Greet`功能有多普遍?

### The internet

Run the following

```go
package main

import (
    "fmt"
    "io"
    "log"
    "net/http"
)

func Greet(writer io.Writer, name string) {
    fmt.Fprintf(writer, "Hello, %s", name)
}

func MyGreeterHandler(w http.ResponseWriter, r *http.Request) {
    Greet(w, "world")
}

func main() {
    log.Fatal(http.ListenAndServe(":5000", http.HandlerFunc(MyGreeterHandler)))
}
```

运行程序，跳转至  [http://localhost:5000](http://localhost:5000)。你将看到 greeting 函数被使用了。

HTTP服务器将在后面的章节中介绍，所以不必太担心细节。

当你写一个HTTP处理程序时，你会得到一个 `http.ResponseWriter` and the `http.Request`  用于发出请求。
当你实现你的服务器时，你使用 writer 写入你的响应。

你可能会猜到 `http.ResponseWriter` 也实现了 `io.Writer`，这就是为什么我们可以在 handler中重用 `Greet` 函数。

## 总结

我们的第一轮代码不容易测试，因为它将数据写到我们无法控制的地方。

_受测试的驱动_，我们重构了代码，这样我们就可以通过注入依赖来控制数据写入的位置，这样我们就可以:

* **测试我们的代码** 如果你不能很容易地测试一个函数，那通常是因为依赖关系硬连接到函数或者全局状态。
    例如，如果您有一个全局数据库连接池，该连接池被某种服务层使用，那么测试起来可能会很困难，而且运行起来也会很慢。
    DI 会激发你注入一个数据库依赖项(通过一个接口)，然后你可以用你可以在测试中控制的东西模拟出来。
* **Separate our concerns**, 解耦数据去哪里和如何生成它。如果你曾经觉得一个方法/函数有太多的责任（生成数据和写入数据库?处理HTTP请求做域级逻辑）DI 可能是你需要的工具。
* **允许我们的代码在不同的上下文中被重用** 我们的代码可以使用的第一个“新”上下文是在测试内部。但如果有人想在你的函数中尝试一些新的东西，他们可以注入自己的依赖项。

### What about mocking? I hear you need that for DI and also it's evil

Mocking 将在后面详细讨论\(它不邪恶\)。您可以使用 mock 将注入的真实内容替换为可以在测试中控制和检查的模拟版本。在我们的例子中，标准库已经准备好了一些东西供我们使用。

### Go 标准库真的很好，花点时间学习一下吧

通过对  `io.Writer` 接口的一些熟悉，我们能够在我们的测试使用 `bytes.Buffer` 作为我们的 `Writer`，然后我们可以使用其他 `Writer` 从标准库在命令行应用程序或web服务器中使用我们的函数。

您对标准库越熟悉，就越容易看到这些通用接口，然后您可以在自己的代码中重用这些接口，使您的软件在许多上下文中可重用。

这个例子很大程度上受到了 [The Go Programming language](https://www.amazon.co.uk/Programming-Language-Addison-Wesley-Professional-Computing/dp/0134190440)的影响, 如果你喜欢，那就去买吧!
