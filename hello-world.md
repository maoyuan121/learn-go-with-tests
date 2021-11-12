# Hello, World

**[You can find all the code for this chapter here](https://github.com/quii/learn-go-with-tests/tree/main/hello-world)**

- 创建一个你喜欢的目录
- 在这个目录里创建一个文件 `hello.go`，内容如下

```go
package main

import "fmt"

func main() {
	fmt.Println("Hello, world")
}
```

然后运行这个 type `go run hello.go`。

## How it works

当你用 Go 写程序的时候，你会有一个 `main` 包，里面有一个 `main` func。
包是将相关的 Go 代码组合在一起的方法。

`func` 关键字用来定义一个函数。

通过使用 `import "fmt"` 我们导入了一个包，它里面有一个 `Println` 函数，我们用来打印。

## How to test

你怎么测试这个?将你的“域”代码与外部世界分开是很好的(副作用)。
`fmt.Println` 是一个副作用\(打印到stdout\)，我们发送的字符串是我们的域。

让我们分离这些问题，以便于测试

```go
package main

import "fmt"

func Hello() string {
	return "Hello, world"
}

func main() {
	fmt.Println(Hello())
}
```

我们用 `func` 再次创建了一个新函数，但这次我们在定义中添加了另一个关键字 `string`。这意味着这个函数返回一个 `string`。

现在在创建一个 `hello_test.go` 文件，在这里面我们将为 `Hello` 函数写测试

```go
package main

import "testing"

func TestHello(t *testing.T) {
	got := Hello()
	want := "Hello, world"

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}
```

## Go modules?

下一步是运行测试。在终端中输入 `go test`。
如果测试通过，那么您可能使用的是较早版本的 Go。
但是，如果您使用的是 Go 1.16 或更高版本，那么测试可能根本无法运行。相反，你会在终端中看到这样的错误信息:

```shell
$ go test
go: cannot find main module; see 'go help modules'
```

这个是什么问题？简而言之，[modules](https://blog.golang.org/go116-module-changes)。
幸运的是这个问题很容易修复。在终端输入 `go mod init hello`。
这将创建一个新的文件，内容如下：

```go
module hello

go 1.16
```

 
这个文件告诉 `go` 工具关于代码的基本信息。
如果您计划分发您的应用程序，您应该包括代码可下载的位置以及关于依赖关系的信息。
现在，您的模块文件是最小的，您可以保持这种方式。要阅读更多关于模块的内容，[you can check out the reference in the Golang documentation](https://golang.org/doc/modules/gomod-ref)。
我们现在可以回到测试和学习 Go 了，因为即使在 Go 1.16 上也应该运行测试。

在以后的章节中，你需要在运行命令如 `go test` 或 `go build` 之前，在每个新文件夹中运行 `go mod init SOMENAME`。


## Back to Testing

在终端中运行 `go test`。它应该已经通过了!只是为了检查，尝试通过改变 `want` 字符串来故意破坏测试。

请注意，您不必在多个测试框架之间进行选择，然后找出如何安装。您所需要的一切都内置在语言中，语法与您将编写的其他代码相同。

### Writing tests

编写测试就像编写函数一样，只有一些规则

* 它需要文件名如 `xxx_test.go`
* 测试函数必须以 `Test` 开头
* 测试函数接收一个参数 `t *testing.T`
* 为了使用 `*testing.T` 类型，你需要 `import "testing"`

现在，只要知道类型为 `*testing.T` 的 `t` 测试就足够了。`t` 是你进入测试框架的“钩子”，所以当你想要失败时，你可以做像 `t.Fail()` 这样的事情。

We've covered some new topics:

#### `if`
If statements in Go are very much like other programming languages.

#### Declaring variables

We're declaring some variables with the syntax `varName := value`, which lets us re-use some values in our test for readability.

#### `t.Errorf`

我们调用 `t` 的 `Errorf` 方法将会打印一段消息，并且测试失败。
`f` 代表格式（format），它允许我们构建一个包含插入到占位符 `%q` 中的值的字符串。

你可以在 [fmt go doc](https://golang.org/pkg/fmt/#hdr-Printing) 中阅读关于占位符字符串的更多信息。
对于测试 `%q` 非常有用，因为它将值包装在双引号中。

稍后我们将探讨方法和函数之间的区别。

### Go doc

Go 的另一个重要特性是文档。

你可以通过运行 `godoc -http:8000` 来运行文档。 
如果你跳转到 [localhost:8000/pkg](http://localhost:8000/pkg) 将看到你系统上安装的所有 package。

绝大多数标准库都有很好的示例文档。
导航到 [http://localhost:8000/pkg/testing/](http://localhost:8000/pkg/testing/) 很值得看看你能找到什么。
                                                                             
如果你没有 `godoc` 命令，那么可能你使用了新版本的 Go (1.14 or later)，[新版本不在内置 `godoc`](https://golang.org/doc/go1.14#godoc)。
你可以通过 `go get golang.org/x/tools/cmd/godoc` 手动安装。

### Hello, YOU

现在我们有一个测试了，可以安全的迭代我们的软件了。

在最后一个例子中，我们在编写代码后编写了编写了测试，这样你就可以得到一个如何编写测试和声明函数的例子。
从现在开始，我们将首先编写测试。

我们下一个需求是让我们可以指定问候的接收者。

让我们从在测试中捕获这些需求开始。
这是基本的测试驱动开发，允许我们确保我们的测试是真正测试我们想要的。
当您回顾性地编写测试时，即使代码没有按照预期工作，您的测试也可能继续通过。

```go
package main

import "testing"

func TestHello(t *testing.T) {
	got := Hello("Chris")
	want := "Hello, Chris"

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}
```

现在运行 `go test`，应该是编译失败了

```text
./hello_test.go:6:18: too many arguments in call to Hello
    have (string)
    want ()
```

当使用像 Go 这样的静态类型语言时，监听编译器是很重要的。
编译器知道你的代码应该如何拼接和工作，所以你不需要。

在这种情况下，编译器会告诉您需要做什么才能继续。我们必须改变函数 `Hello` 以接受一个参数。

编辑 `Hello` 函数接收一个字符串类型的参数

```go
func Hello(name string) string {
	return "Hello, world"
}
```

如果你再次运行测试，编译器也会报错，因为你没有传入一个参数。现在把 "world" 传进去。

```go
func main() {
	fmt.Println(Hello("world"))
}
```

现在你运行测试，应该能看到

```text
hello_test.go:10: got 'Hello, world' want 'Hello, Chris''
```

我们最终有了一个编译的程序，但根据测试，它不符合我们的要求。


让我们使用 name 参数使测试通过，并将它与 `Hello,` 连接起来



```go
func Hello(name string) string {
	return "Hello, " + name
}
```

现在测试应该能通过了。通常，作为 TDD 周期的一部分，我们现在应该进行重构。

### A note on source control

到这里，如果你使用了源代码控制器，我将 `commit` 代码。

不过我不会急于 push 到 master，因为我接下来打算重构。
如果你在重构的过程中陷入了混乱，那么最好在此时提交 —— 你总可以回到可以工作的版本。

这里没有太多需要重构的东西，但我们可以引入另一种语言特性 _常量_。

### Constants

常量定义如下

```go
const englishHelloPrefix = "Hello, "
```

现在我们可以重构代码

```go
const englishHelloPrefix = "Hello, "

func Hello(name string) string {
	return englishHelloPrefix + name
}
```

重构后，重新运行测试确保你没有破坏任何东西。

常量可以提高应用程序的性能，因为它可以在每次调用 Hello 时节省创建字符串实例的时间。

需要说明的是，在这个例子中，性能提升是非常微不足道的!
但值得考虑创建常量来捕获值的含义，有时还有助于提高性能。

## Hello, world... again

下一个需求是，当我们的函数被一个空字符串调用时，它默认打印“Hello, World”，而不是“Hello，”。

从编写一个新的失败测试开始

```go
func TestHello(t *testing.T) {

	t.Run("saying hello to people", func(t *testing.T) {
		got := Hello("Chris")
		want := "Hello, Chris"

		if got != want {
			t.Errorf("got %q want %q", got, want)
		}
	})

	t.Run("say 'Hello, World' when an empty string is supplied", func(t *testing.T) {
		got := Hello("")
		want := "Hello, World"

		if got != want {
			t.Errorf("got %q want %q", got, want)
		}
	})

}
```

在这里，我们将介绍我们的测试库中的另一个工具，子测试。
有时，围绕一个“事物”对测试进行分组，然后使用描述不同场景的子测试是很有用的。

这种方法的一个好处是，您可以设置可在其他测试中使用的共享代码。

当我们检查消息是否符合预期时，会有重复的代码。

重构不仅仅是为了生产代码!

重要的是，你的测试必须清楚说明代码需要做什么。

我们能并且也应该重构我们的测试。

```go
func TestHello(t *testing.T) {

	assertCorrectMessage := func(t testing.TB, got, want string) {
		t.Helper()
		if got != want {
			t.Errorf("got %q want %q", got, want)
		}
	}

	t.Run("saying hello to people", func(t *testing.T) {
		got := Hello("Chris")
		want := "Hello, Chris"
		assertCorrectMessage(t, got, want)
	})

	t.Run("empty string defaults to 'World'", func(t *testing.T) {
		got := Hello("")
		want := "Hello, World"
		assertCorrectMessage(t, got, want)
	})

}
```

我们在这做了些什么？

我们已经将断言重构为一个函数。这减少了重复并提高了测试的可读性。在 Go 中，你可以在其他函数中声明函数，并将它们赋值给变量。
然后您可以调用它们，就像普通函数一样。我们需要通过测试。这样我们就可以在需要的时候告诉测试代码失败。

对于帮助函数，接受 `testing.TB` 是个好主意，它同时实现了 `*testing.T` 和 `*testing.B` 接口。
因此你可以在测试或者 benchmark 中调用帮助函数。

需要使用 `t.Helper()` 来告诉测试套件这个方法是一个帮助函数。
通过这样做，当它失败时，报告的行号将在我们的 _function call_ 中，而不是在我们的测试帮助函数中。
这将帮助其他开发人员更容易地跟踪问题。
如果您仍然不理解，就将其注释掉，使测试失败并观察测试输出。
Go 中的注释是向代码添加额外信息的好方法，或者在这种情况下，是告诉编译器忽略一行代码的一种快速方法。
可以通过在行首添加两个斜杠 `//` 来注释 `t.Helper()` 代码。
您应该会看到该行变成灰色或更改为其他颜色，而不是代码的其余部分，以表明它现在已被注释掉。

现在我们已经有了一个编写良好的失败测试，让我们使用 `if` 来修复代码。

```go
const englishHelloPrefix = "Hello, "

func Hello(name string) string {
	if name == "" {
		name = "World"
	}
	return englishHelloPrefix + name
}
```

如果我们运行我们的测试，我们应该看到它满足了新的需求，并且我们没有意外地破坏其他功能。

### Back to source control

现在我们对我要修改之前提交的代码很满意，所以我们只签入带有测试的可爱版本的代码。

### Discipline

让我们再来回顾一下这个循环

* 编写测试
* 确保编译通过
* 运行测试，查看它是否失败，并检查错误消息是否有意义
* 编写足够的代码使测试通过
* 重构

从表面上看，这似乎很乏味，但坚持反馈循环很重要。

它不仅能确保你有相关的测试，还能通过重构测试的安全性来帮助你设计出好的软件。

看到测试失败是一个重要的检查，因为它还让您看到错误消息是什么样子的。
作为一个开发人员，当测试失败不能明确指出问题所在时，使用代码库是非常困难的。

通过确保您的测试是快速的，并设置您的工具以使运行测试变得简单，您可以在编写代码时进入一种流状态。

如果不编写测试，你就只能通过运行软件来手动检查代码，这会破坏你的流程状态，而且你不会节省任何时间，特别是从长远来看。

## Keep going! More requirements

天哪，我们还有更多的要求。现在我们需要支持第二个参数，指定问候语的语言。
如果传入的语言我们不认识，就默认为英语。

我们应该有信心，我们可以使用 TDD 轻松地充实这个功能!

为通过西班牙语测试的用户编写测试。将其添加到现有套件中。

```go
	t.Run("in Spanish", func(t *testing.T) {
		got := Hello("Elodie", "Spanish")
		want := "Hola, Elodie"
		assertCorrectMessage(t, got, want)
	})
```

记住不要作弊! _Test first_。当你尝试运行测试时，编译器应该会报错，因为你用两个参数而不是一个参数调用 `Hello`。

```text
./hello_test.go:27:19: too many arguments in call to Hello
    have (string, string)
    want (string)
```

通过向 `Hello` 添加另一个字符串参数来修复编译问题

```go
func Hello(name string, language string) string {
	if name == "" {
		name = "World"
	}
	return englishHelloPrefix + name
}
```

当你再次尝试运行测试时，它会抱怨在其他测试和 `hello.go` 中没有传递足够的参数给 `Hello`

```text
./hello.go:15:19: not enough arguments in call to Hello
    have (string)
    want (string, string)
```

通过传递空字符串来修复它们。现在，除了我们的新场景，所有的测试都应该编译并通过

```text
hello_test.go:29: got 'Hello, Elodie' want 'Hola, Elodie'
```

我们可以在这里使用 `if` 来检查语言是否等于 `Spanish`，如果是的话就改变信息

```go
func Hello(name string, language string) string {
	if name == "" {
		name = "World"
	}

	if language == "Spanish" {
		return "Hola, " + name
	}

	return englishHelloPrefix + name
}
```

测试现在应该能通过了。

现在是重构的时候了。你应该会在代码中看到一些问题，"magic"字符串，其中一些是重复的。尝试自己重构它，每次修改都要重新运行测试，以确保重构不会破坏任何东西。

```go
const spanish = "Spanish"
const englishHelloPrefix = "Hello, "
const spanishHelloPrefix = "Hola, "

func Hello(name string, language string) string {
	if name == "" {
		name = "World"
	}

	if language == spanish {
		return spanishHelloPrefix + name
	}

	return englishHelloPrefix + name
}
```

### French

* 写一个测试，断言如果你传入了 `"French"`，你会得到 `"Bonjour"`，
* 看到它失败了，检查错误信息很容易读取
* 在代码中做最小的合理更改吗


你可能写过类似这样的东西

```go
func Hello(name string, language string) string {
	if name == "" {
		name = "World"
	}

	if language == spanish {
		return spanishHelloPrefix + name
	}

	if language == french {
		return frenchHelloPrefix + name
	}

	return englishHelloPrefix + name
}
```

## `switch`

当你有很多 `if` 语句检查一个特定的值时，通常使用 `switch` 语句来代替。如果我们希望以后添加更多的语言支持，我们可以使用 `switch` 来重构代码，使其更容易阅读和更可扩展

```go
func Hello(name string, language string) string {
	if name == "" {
		name = "World"
	}

	prefix := englishHelloPrefix

	switch language {
	case french:
		prefix = frenchHelloPrefix
	case spanish:
		prefix = spanishHelloPrefix
	}

	return prefix + name
}
```


现在编写一个测试，用您选择的语言包含问候语，您应该会看到扩展 _amazing_ 函数是多么简单。

### one...last...refactor?

你可能会说函数变大了一点。最简单的重构方法是将一些功能提取到另一个函数中。

```go
func Hello(name string, language string) string {
	if name == "" {
		name = "World"
	}

	return greetingPrefix(language) + name
}

func greetingPrefix(language string) (prefix string) {
	switch language {
	case french:
		prefix = frenchHelloPrefix
	case spanish:
		prefix = spanishHelloPrefix
	default:
		prefix = englishHelloPrefix
	}
	return
}
```

一些新的概念：

* 在我们的函数前面中我们用了一个 _命名返回值_ `(prefix string)`
* 这将在函数中创建一个名为 `prefix` 的变量。
  * 它将被赋值为“零”。这取决于类型，例如 `int` 是 0，`string` 是 `""`。
    * 你可以通过调用 `return` 而不是 `return prefix` 来返回它设置的值。
  * 这将显示在你的函数的 Go Doc 中，这样它可以使你的代码意图更清楚。
* 如果其他 `case` 语句都不匹配，switch case 中的 `default` 将被分支到。
* 函数名以小写字母开头。在 Go 中，公共函数以大写字母开头，私有函数以小写字母开头。我们不想让算法的内部信息暴露给外界，所以我们把这个函数设为私有。

## Wrapping up

谁知道你能从 `Hello, world` 中学到这么多?

到目前为止，你应该对以下内容有所了解:

### Some of Go's syntax around

* 写测试
* 声明带有参数和返回类型的函数
* `if`, `const` 和 `switch`
* 声明变量和常量

### The TDD process and _why_ the steps are important

* 写一个失败的测试，然后看到它失败了，这样我们就知道我们已经为我们的需求写了一个相关的测试，并且看到它产生了一个容易理解的失败描述
* 用最少的代码让它通过，这样我们就知道我们的软件是可以工作的
* 然后重构，以我们测试的安全性为后盾，以确保我们有易于使用的精心制作的代码
 
在我们的例子中，我们从 `Hello()` 到 `Hello("name")`，再到 `Hello("name"， "French")`，这些步骤很小，很容易理解。

与“真实世界”的软件相比，这当然是微不足道的，但原则仍然有效。TDD 是一种需要实践才能开发的技能，但是通过将问题分解成更小的组件进行测试，您将会更容易地编写软件。

