# Integers

**[You can find all the code for this chapter here](https://github.com/quii/learn-go-with-tests/tree/main/integers)**

整数的工作方式与您期望的一样。让我们写一个 `Add` 函数来尝试一下。创建一个名为 `adder_test` 的测试文件。去写这个代码吧。

**注意:** 每个目录只能有一个 `package`，请确保您的文件是单独组织的。[这里有一个很好的解释](https://dave.cheney.net/2014/12/01/five-suggestions-for-setting-up-a-go-project)

## Write the test first

```go
package integers

import "testing"

func TestAdder(t *testing.T) {
	sum := Add(2, 2)
	expected := 4

	if sum != expected {
		t.Errorf("expected '%d' but got '%d'", expected, sum)
	}
}
```

你将注意到我们使用了 `%d` 用来格式化字符出，而不是用 `%q`。这是因为我们想打印出一个整形而不是字符出。

我们不在使用 main package，我们定义了一个名为 `integers` 的包，顾名思义，这将对处理整数的函数进行分组，如 `Add`。


## Try and run the test

运行测试 `go test`

将出现编译错误

`./adder_test.go:6:9: undefined: Add`

## Write the minimal amount of code for the test to run and check the failing test output

编写足够的代码来满足编译器的要求 —— 这就是全部 —— 记住，我们要检查我们的测试是否因为正确的原因而失败。

```go
package integers

func Add(x, y int) int {
	return 0
}
```


当你有多个相同类型的参数 \(在本例中是两个整数\) 而不是 `(x int, y int)`，你可以把它缩短为 `(x, y int)`。

现在运行测试，我们应该很高兴测试正确地报告了错误。

`adder_test.go:10: expected '4' but got '0'`

如果你注意到我们在[last](hello-world.md#one…last…refactor?)小节中学习了 _named return value_，但在这里没有使用相同的方法。
它通常应该用于结果的含义从上下文不清楚时，在我们的情况下，“Add”函数将添加参数非常清楚。你可以参考[this](https://github.com/golang/go/wiki/CodeReviewComments#named-result-parameters) wiki
了解更多细节。



## Write enough code to make it pass

从 TDD 的最严格意义上讲，我们现在应该编写最小数量的代码以使测试通过。一个学究气的程序员可能会这样做

```go
func Add(x, y int) int {
	return 4
}
```

呀哈!又失败了，TDD是假的，对吧?

我们可以编写另一个测试，使用一些不同的数字来迫使测试失败，但这感觉就像[猫捉老鼠的游戏](https://en.m.wikipedia.org/wiki/Cat_and_mouse)。

一旦我们更加熟悉 Go 的语法，我将介绍一种名为“基于属性的测试”的技术，它将停止打扰开发人员，并帮助您找到bug。

现在，让我们正确地修复它

```go
func Add(x, y int) int {
	return x + y
}
```

重新运行测试，测试通过了。

## Refactor

在 _actual_ 代码中我们可以改进的地方不多。

我们在前面探讨了如何通过命名 return 参数，它不仅出现在文档中，而且也出现在大多数开发人员的文本编辑器中。

这很好，因为它有助于您编写的代码的可用性。用户最好能通过查看类型签名和文档来理解代码的用法。

你可以用注释给函数添加文档，这些注释会出现在 Go Doc 中，就像你查看标准库的文档一样。

```go
// Add takes two integers and returns the sum of them.
func Add(x, y int) int {
	return x + y
}
```

### Examples

如果你真的想走得更远，你可以做 [examples](https://blog.golang.org/examples)。您可以在标准库的文档中找到许多示例。

通常可以在代码库之外找到的代码示例，比如自述文件，与实际的代码相比，经常会变得过时和不正确，因为它们没有被检查。

Go 示例的执行就像测试一样，因此您可以确信示例反映了代码的实际操作。

作为包测试套件的一部分，示例被编译\(或可选地执行\)。

与典型的测试一样，示例是驻留在包的 `_test` 中的函数。将下面的 `ExampleAdd` 函数添加到 `adder_test.go` 中。


```go
func ExampleAdd() {
	sum := Add(1, 5)
	fmt.Println(sum)
	// Output: 6
}
```

(如果你的编辑器没有自动为你导入包，编译步骤会失败，因为你会在 `adder_test.go` 中缺少 `import "fmt"`。强烈建议你研究如何在任何你正在使用的编辑器中自动修复这些错误。)

如果您的代码发生了更改，使示例不再有效，则构建将失败。

运行包的测试套件，我们可以看到示例函数在没有进一步安排的情况下执行:

```bash
$ go test -v
=== RUN   TestAdder
--- PASS: TestAdder (0.00s)
=== RUN   ExampleAdd
--- PASS: ExampleAdd (0.00s)
```

请注意，如果你删除注释' //Output: 6 '，示例函数将不会被执行。虽然函数将被编译，但它不会被执行。

通过添加这段代码，示例将出现在文档中的 `godoc` 中，使您的代码更容易访问。

要尝试这个，请运行 `godoc -http=:6060` 并导航到 `http://localhost:6060/pkg/`

在这里，你会看到 `$GOPATH` 中的所有包的列表，所以假设你在 `$GOPATH/src/github.com/{your_id}` 之类的地方写了这段代码，你就可以找到你的示例文档。

如果你将你的代码和示例发布到一个公共 URL，你可以在[pkg.go.dev](https://pkg.go.dev/)分享你的代码文档。例如，[here](https://pkg.go.dev/github.com/quii/learn-go-with-tests/integers/v2)
是本章最终确定的 API。这个 web 界面允许您搜索标准库包和第三方包的文档。

## Wrapping up

我们已经涵盖了:

* 更多TDD工作流程的实践
* 整形, 加法
* 编写更好的文档，以便我们的代码的用户能够快速理解它的用法
* 示例说明如何使用我们的代码，这些代码将作为测试的一部分进行检查
