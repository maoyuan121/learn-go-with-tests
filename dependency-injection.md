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

**Our function doesn't need to care **_**where**_** or **_**how**_** the printing happens, so we should accept an **_**interface**_** rather than a concrete type.**

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

Run the program and go to [http://localhost:5000](http://localhost:5000). You'll see your greeting function being used.

HTTP servers will be covered in a later chapter so don't worry too much about the details.

When you write an HTTP handler, you are given an `http.ResponseWriter` and the `http.Request` that was used to make the request. 
When you implement your server you _write_ your response using the writer.

You can probably guess that `http.ResponseWriter` also implements `io.Writer` so this is why we could re-use our `Greet` function inside our handler.

## Wrapping up

Our first round of code was not easy to test because it wrote data to somewhere we couldn't control.

_Motivated by our tests_ we refactored the code so we could control _where_ the data was written by **injecting a dependency** which allowed us to:

* **Test our code** If you can't test a function _easily_, it's usually because of dependencies hard-wired into a function _or_ global state. If you have a global database connection pool for instance that is used by some kind of service layer, it is likely going to be difficult to test and they will be slow to run. DI will motivate you to inject in a database dependency \(via an interface\) which you can then mock out with something you can control in your tests.
* **Separate our concerns**, decoupling _where the data goes_ from _how to generate it_. If you ever feel like a method/function has too many responsibilities \(generating data _and_ writing to a db? handling HTTP requests _and_ doing domain level logic?\) DI is probably going to be the tool you need.
* **Allow our code to be re-used in different contexts** The first "new" context our code can be used in is inside tests. But further on if someone wants to try something new with your function they can inject their own dependencies.

### What about mocking? I hear you need that for DI and also it's evil

Mocking will be covered in detail later \(and it's not evil\). You use mocking to replace real things you inject with a pretend version that you can control and inspect in your tests. In our case though, the standard library had something ready for us to use.

### The Go standard library is really good, take time to study it

By having some familiarity with the `io.Writer` interface we are able to use `bytes.Buffer` in our test as our `Writer` and then we can use other `Writer`s from the standard library to use our function in a command line app or in web server.

The more familiar you are with the standard library the more you'll see these general purpose interfaces which you can then re-use in your own code to make your software reusable in a number of contexts.

This example is heavily influenced by a chapter in [The Go Programming language](https://www.amazon.co.uk/Programming-Language-Addison-Wesley-Professional-Computing/dp/0134190440), so if you enjoyed this, go buy it!
