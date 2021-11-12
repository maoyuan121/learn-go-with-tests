# Iteration

**[You can find all the code for this chapter here](https://github.com/quii/learn-go-with-tests/tree/main/for)**

在 go 中重复做一些事情，你需要 `for`。在 go 中没有 `while`、`do`、`until` 等关键字，你只能使用 `for`。这是一件好事!

让我们为一个重复字符 5 次的函数编写一个测试。

到目前为止没有什么新的，所以试着自己写下来练习。

## Write the test first

```go
package iteration

import "testing"

func TestRepeat(t *testing.T) {
	repeated := Repeat("a")
	expected := "aaaaa"

	if repeated != expected {
		t.Errorf("expected %q but got %q", expected, repeated)
	}
}
```

## Try and run the test

`./repeat_test.go:6:14: undefined: Repeat`

## Write the minimal amount of code for the test to run and check the failing test output

保持纪律!您现在不需要知道任何新的东西就可以使测试正确地失败。

您现在需要做的就是使它能够编译，以便检查您的测试是否编写良好。

```go
package iteration

func Repeat(character string) string {
	return ""
}
```

知道您已经掌握了足够的 Go 知识，可以为一些基本问题编写测试，不是很好吗?这意味着您现在可以随心所欲地使用生产代码，并知道它的行为与您希望的一致。

`repeat_test.go:10: expected 'aaaaa' but got ''`

## Write enough code to make it pass

`for` 语法非常普通，它遵循大多数 c 类语言。

```go
func Repeat(character string) string {
	var repeated string
	for i := 0; i < 5; i++ {
		repeated = repeated + character
	}
	return repeated
}
```

与其他语言(如C、Java或JavaScript)不同，for 语句的三个组件周围没有括号，括号 `{}` 总是必需的。你可能想知道这一行发生了什么

```go
	var repeated string
```

到目前为止，我们一直使 `:=` 来声明和初始化变量。然而， `:=` 仅仅是[两个步骤的简写](https://gobyexample.com/variables)。
这里我们只声明了一个 `string` 变量。因此，显式版本。我们还可以使用 `var` 来声明函数，我们将在后面看到。

运行测试现在应该能通过了。

[这里](https://gobyexample.com/for) 对于 for 循环的其他变体进行了描述。

## Refactor

现在是重构和引入另一个构造 `+=` 赋值操作符的时候了。

```go
const repeatCount = 5

func Repeat(character string) string {
	var repeated string
	for i := 0; i < repeatCount; i++ {
		repeated += character
	}
	return repeated
}
```

`+=` 称为"加法和赋值操作符"，将右操作数加到左操作数上，并将结果赋值给左操作数。它适用于其他类型，如整数。

### Benchmarking

在 Go 中编写 [benchmarks](https://golang.org/pkg/testing/#hdr-Benchmarks) 是该语言的另一个一流特性，它与编写测试非常相似。

```go
func BenchmarkRepeat(b *testing.B) {
	for i := 0; i < b.N; i++ {
		Repeat("a")
	}
}
```

您将看到代码与测试非常相似。

`testing.B` 让你可以访问 `b.N`.。

When the benchmark code is executed, it runs `b.N` times and measures how long it takes.

当基准代码被执行时，它运行 `b.N` 次，并计算所需的时间。

代码运行的次数对你来说不重要，框架将决定什么是“好的”值，让你有一些像样的结果。

要运行 benchmarks 测试，请执行 `go test -bench=`。(或者如果你在Windows Powershell中 `go test -bench="."`)

```text
goos: darwin
goarch: amd64
pkg: github.com/quii/learn-go-with-tests/for/v4
10000000           136 ns/op
PASS
```

`136 ns/op` 的意思是我们的函数运行\(在我的计算机上\)平均需要 136 纳秒。这很好!为了测试，它运行了 10000000 次。

默认情况下基准测试是按顺序运行的。

## Practice exercises

* 更改测试，以便调用者可以指定字符重复的次数，然后修复代码
* 编写 `ExampleRepeat` 来文档化函数
* 查看 [strings](https://golang.org/pkg/strings) 包。找到您认为可能有用的函数，并通过编写像这里一样的测试来试验它们。随着时间的推移，花时间学习标准库真的会有回报。

## Wrapping up

* More TDD practice
* Learned `for`
* Learned how to write benchmarks
