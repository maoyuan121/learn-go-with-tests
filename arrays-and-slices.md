# Arrays and slices

**[You can find all the code for this chapter here](https://github.com/quii/learn-go-with-tests/tree/main/arrays)**

Arrays 允许你用一个变量以特定顺序存储同一个类型的多个元素。

当您有一个数组时，对它们进行迭代是很常见的。
所以让我们用 [for](iteration.md) 来做一个 Sum 函数。Sum 将接受一个数字数组并返回总数。

## 先写测试

创建一个新的工作文件夹。创建一个名为 `sum_test` 的新文件。然后插入以下内容：

```go
package main

import "testing"

func TestSum(t *testing.T) {

	numbers := [5]int{1, 2, 3, 4, 5}

	got := Sum(numbers)
	want := 15

	if got != want {
		t.Errorf("got %d want %d given, %v", got, want, numbers)
	}
}
```

数组有 _固定的容量_，在声明变量时定义。
有两种方式初始化一个数组：

* \[N\]type{value1, value2, ..., valueN} e.g. `numbers := [5]int{1, 2, 3, 4, 5}`
* \[...\]type{value1, value2, ..., valueN} e.g. `numbers := [...]int{1, 2, 3, 4, 5}`

有时在错误消息中打印函数的输入也是有用的，我们使用的是 `%v` 占位符，这是“默认”格式，这对数组很有效。

[关于 format 字符串的更多内容](https://golang.org/pkg/fmt/)

## 运行测试

运行 `go test`, 编译失败 `./sum_test.go:10:15: undefined: Sum`

## 为要运行的测试编写最小数量的代码，并检查失败的测试输出

In `sum.go`

```go
package main

func Sum(numbers [5]int) int {
	return 0
}
```

现在测试失败了，并且有一个 _清晰的错误消息_

`sum_test.go:13: got 0 want 15 given, [1 2 3 4 5]`

## 编写足够的代码使测试通过

```go
func Sum(numbers [5]int) int {
	sum := 0
	for i := 0; i < 5; i++ {
		sum += numbers[i]
	}
	return sum
}
```

要从特定索引处的数组中获取值，只需使用 `array[index]` 语法。
在本例中，我们使用 `for` 迭代 5 次，遍历数组并将每一项加到 `sum` 上。

## 重构

引入 [`range`](https://gobyexample.com/range) 帮助使得代码更整洁

```go
func Sum(numbers [5]int) int {
	sum := 0
	for _, number := range numbers {
		sum += number
	}
	return sum
}
```

`range` 允许你遍历数组。每次调用它都会返回两个值，index 和 value。
我们选择使用 `_` [空白标识符](https://golang.org/doc/effective_go.html#blank)来忽略索引值。

### 数组及其类型

数组的一个有趣的特性是，数组的大小是数组类型的一部分。
如果你试图将一个 `[4]int` 传递给一个期望 `[5]int` 的函数，它将无法编译。
它们是不同的类型，所以这就像试图传递一个 `string` 给一个想要 `int` 的函数一样。

你可能会认为数组有固定长度很麻烦，而且大多数时候你可能不会使用它们!

Go 有 _slices_，它不编码集合的大小，而是可以有任何大小。

下一个要求是对大小不同的集合进行求和。

## 先写测试

我们将使用 [slice type][slice]，它允许我们有任意大小的集合。语法和数组非常相似，只需要在声明的时候忽略大小
 `mySlice := []int{1,2,3}` 而不是 `myArray := [3]int{1,2,3}`


```go
func TestSum(t *testing.T) {

	t.Run("collection of 5 numbers", func(t *testing.T) {
		numbers := [5]int{1, 2, 3, 4, 5}

		got := Sum(numbers)
		want := 15

		if got != want {
			t.Errorf("got %d want %d given, %v", got, want, numbers)
		}
	})

	t.Run("collection of any size", func(t *testing.T) {
		numbers := []int{1, 2, 3}

		got := Sum(numbers)
		want := 6

		if got != want {
			t.Errorf("got %d want %d given, %v", got, want, numbers)
		}
	})

}
```

## 运行测试

编译失败

`./sum_test.go:22:13: cannot use numbers (type []int) as type [5]int in argument to Sum`

## 为要运行的测试编写最小数量的代码，并检查失败的测试输出

问题是我们也可以

* 通过将 `Sum` 的参数改为 slice 来破坏现有的 API。当我们这样做时，我们就会知道我们可能毁了别人的一天，因为我们的 _其它_ 测试将无法编译!
* 创建一个新的函数

在我们的例子中，没有其他人使用我们的函数，所以与其有两个函数要维护，我们只需要一个。

```go
func Sum(numbers []int) int {
	sum := 0
	for _, number := range numbers {
		sum += number
	}
	return sum
}
```

如果您试图运行它们仍然不能编译的测试，则必须更改第一个测试以通过切片而不是数组。

## 编写足够的代码使得测试通过

事实证明，修复编译器问题是我们在这里所需要做的一切，并且测试通过了!

## 重构

我们已经重构了 `Sum` 我们所做的只是从数组变成了切片，所以这里没有太多要做的。请记住，在重构阶段我们一定不能忽视测试代码，这里我们还有一些工作要做。

```go
func TestSum(t *testing.T) {

	t.Run("collection of 5 numbers", func(t *testing.T) {
		numbers := []int{1, 2, 3, 4, 5}

		got := Sum(numbers)
		want := 15

		if got != want {
			t.Errorf("got %d want %d given, %v", got, want, numbers)
		}
	})

	t.Run("collection of any size", func(t *testing.T) {
		numbers := []int{1, 2, 3}

		got := Sum(numbers)
		want := 6

		if got != want {
			t.Errorf("got %d want %d given, %v", got, want, numbers)
		}
	})

}
```

对测试的价值提出质疑是很重要的。有尽可能多的测试不应该是一个目标，
但在您的代码库中要有可能足够的信心。太多的测试可能会变成一个真正的问题这只会增加维护的开销。**每个测试都有成本**。


在我们的例子中，您可以看到对这个函数进行两次测试是多余的。
如果它对一个大小的切片有效那么它很可能对任何尺寸大小的切片有效
(在合理范围内)。

Go's built-in testing toolkit features a [coverage
tool](https://blog.golang.org/cover), which can help identify areas of your code
you have not covered. I do want to stress that having 100% coverage should not
be your goal, it's just a tool to give you an idea of your coverage. If you have
been strict with TDD, it's quite likely you'll have close to 100% coverage
anyway.

Go 的内置测试工具包具有 [coveragetool](https://blog.golang.org/cover)，
它可以帮助识别你没有覆盖的代码区域。我想强调的是，100% 的保险覆盖是不应该的成为你的目标，它只是一个工具，让你了解你的覆盖范围。
如果你有如果严格使用 TDD，很有可能你将拥有接近 100% 的覆盖率无论如何。

运行

`go test -cover`

你将看到

```bash
PASS
coverage: 100.0% of statements
```

现在删除其中一个测试并再次检查覆盖率。

现在我们很高兴我们有一个经过良好测试的功能，你应该在接受下一个挑战之前提交你的伟大作品。

我们需要一个名为 `SumAll` 的新函数，它需要不同数量的切片，返回一个新的切片，其中包含传入的每个片的相加总和。

例子

`SumAll([]int{1,2}, []int{0,9})` would return `[]int{3, 9}`

或者

`SumAll([]int{1,1,1})` would return `[]int{3}`

## 先写测试

```go
func TestSumAll(t *testing.T) {

	got := SumAll([]int{1, 2}, []int{0, 9})
	want := []int{3, 9}

	if got != want {
		t.Errorf("got %v want %v", got, want)
	}
}
```

## 运行测试

`./sum_test.go:23:9: undefined: SumAll`

## 为要运行的测试编写最小数量的代码，并检查失败的测试输出

我们需要根据我们的测试需要定义 `SumAll`。

我们可以在 Go 中写 [_variadic functions_](https://gobyexample.com/variadic-functions)，它可以使用可变数量的参数。

```go
func SumAll(numbersToSum ...[]int) (sums []int) {
	return
}
```

尝试编译我们的测试，仍然编译失败！

`./sum_test.go:26:9: invalid operation: got != want (slice can only be compared to nil)`

Go 不允许对切片使用相等运算符。
你可以编写一个函数来迭代每个 `got` 和 `want` 的切片并检查它们的值
但是为了方便起见，我们可以用 [`reflect.DeepEqual`][deepEqual] 用于查看任意两个变量是否相同。

```go
func TestSumAll(t *testing.T) {

	got := SumAll([]int{1, 2}, []int{0, 9})
	want := []int{3, 9}

	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %v want %v", got, want)
	}
}
```

\(确保你的文件顶部有 `import reflect` 以访问 `DeepEqual`\)

重要的是要注意 `reflect.DeepEqual` 不是“类型安全”的代码即使你做了一些愚蠢的事情，也可以编译。
为了看到这一点，暂时将测试改为:

```go
func TestSumAll(t *testing.T) {

	got := SumAll([]int{1, 2}, []int{0, 9})
	want := "bob"

	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %v want %v", got, want)
	}
}
```

我们在这里做的是比较 `slice` 和 `string`。
这没有任何意义，但是测试可以编译!
所以在使用 `reflect.DeepEqual` 是比较片\(和其他东西\)的一种方便的方法
你使用它的时候一定要小心。

再次更改测试并运行它，您的测试输出应该如下所示

`sum_test.go:30: got [] want [3 9]`

## 编写足够的代码使测试通过

我们需要做的是遍历变量，使用 `Sum` 函数计算总和，然后将其加到返回的切片中

```go
func SumAll(numbersToSum ...[]int) []int {
	lengthOfNumbers := len(numbersToSum)
	sums := make([]int, lengthOfNumbers)

	for i, numbers := range numbersToSum {
		sums[i] = Sum(numbers)
	}

	return sums
}
```

有很多新东西要学!

有一种新的方法来制作切片。`make` 允许你用我们需要处理的 `numbersToSum` 的 `len` 的作为新切片的起始容量。

你可以用 `mySlice[N]` 像数组这样的切片来获取值或用 `=` 给它赋一个新值

测试现在应该能通过了

## Refactor

如前所述，切片具有容量。如果你有一个容量为 2 的切片，然后尝试执行 `mySlice[10] = 1`，你会得到一个 _runtime_ 错误。

然而，你可以使用 `append` 函数，它接受一个切片和一个新值，返回一个包含所有项目的新切片。

```go
func SumAll(numbersToSum ...[]int) []int {
	var sums []int
	for _, numbers := range numbersToSum {
		sums = append(sums, Sum(numbers))
	}

	return sums
}
```

在这个实现中，我们不太担心容量。我们从一个空切片 `Sum` 开始，并在处理变量时将 `Sum` 的结果附加到它后面。

我们的下一个要求是将 `SumAll` 改为 `SumAllTails` 计算每个切片“tails”的总数。集合的尾部是
除了第一个\(“头”\)以外的所有项目。

## Write the test first

```go
func TestSumAllTails(t *testing.T) {
	got := SumAllTails([]int{1, 2}, []int{0, 9})
	want := []int{2, 9}

	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %v want %v", got, want)
	}
}
```

## 尝试运行测试

`./sum_test.go:26:9: undefined: SumAllTails`

## 为要运行的测试编写最小数量的代码，并检查失败的测试输出

将函数重命名为 `SumAllTails`，重新运行测试

`sum_test.go:30: got [3 9] want [2 9]`

## 写足够的代码使得测试通过

```go
func SumAllTails(numbersToSum ...[]int) []int {
	var sums []int
	for _, numbers := range numbersToSum {
		tail := numbers[1:]
		sums = append(sums, Sum(tail))
	}

	return sums
}
```

切片可以被 sliced！语法是 `slice[low:high]`，如果你省略了 `:` 一面的值，它抓住了它的一面的一切。
在我们的在 Case 中，我们用 `numbers[1:]` 表示“从1到最后”。
你可能想花一些时间编写关于切片的其他测试，并用切片运算符，你们可以熟悉一下。

## 重构

这次没有太多需要重构的东西。

你觉得如果你一个空的切片传递给我们的函数会发生什么？当你告诉 Go 从 `myEmptySlice[1:]` 中捕获所有元素?

## 先写测试

```go
func TestSumAllTails(t *testing.T) {

	t.Run("make the sums of some slices", func(t *testing.T) {
		got := SumAllTails([]int{1, 2}, []int{0, 9})
		want := []int{2, 9}

		if !reflect.DeepEqual(got, want) {
			t.Errorf("got %v want %v", got, want)
		}
	})

	t.Run("safely sum empty slices", func(t *testing.T) {
		got := SumAllTails([]int{}, []int{3, 4, 5})
		want := []int{0, 9}

		if !reflect.DeepEqual(got, want) {
			t.Errorf("got %v want %v", got, want)
		}
	})

}
```

## 运行测试

```text
panic: runtime error: slice bounds out of range [recovered]
    panic: runtime error: slice bounds out of range
```

噢,不！重要的是要注意测试 _has compiled_，它是一个运行时错误。
编译时错误是我们的朋友，因为它们帮助我们编写这样的软件
运行时错误是我们的敌人，因为它们影响我们的用户。

## 写足够的代码使得测试通过

```go
func SumAllTails(numbersToSum ...[]int) []int {
	var sums []int
	for _, numbers := range numbersToSum {
		if len(numbers) == 0 {
			sums = append(sums, 0)
		} else {
			tail := numbers[1:]
			sums = append(sums, Sum(tail))
		}
	}

	return sums
}
```

## Refactor

我们的测试有一些关于断言的重复代码，让我们将其提取到一个函数中

```go
func TestSumAllTails(t *testing.T) {

	checkSums := func(t testing.TB, got, want []int) {
		t.Helper()
		if !reflect.DeepEqual(got, want) {
			t.Errorf("got %v want %v", got, want)
		}
	}

	t.Run("make the sums of tails of", func(t *testing.T) {
		got := SumAllTails([]int{1, 2}, []int{0, 9})
		want := []int{2, 9}
		checkSums(t, got, want)
	})

	t.Run("safely sum empty slices", func(t *testing.T) {
		got := SumAllTails([]int{}, []int{3, 4, 5})
		want := []int{0, 9}
		checkSums(t, got, want)
	})

}
```

这样做的一个方便的副作用是，它为我们的代码增加了一点类型安全。
如果一个愚蠢的开发人员在编译器中添加了一个 `checkSums(t, got, "dave")` 的新测试就能阻止他们

```bash
$ go test
./sum_test.go:52:21: cannot use "dave" (type string) as type []int in argument to checkSums
```

## Wrapping up

我们已经介绍了

* Arrays
* Slices
  * 创建 slice 的几种方式
  * 如何创建一个有固定容量的 slice。但是你可以使用 `append` 从一个 slice 创建一个新的 slice。
  * 如何对切片进行切片
* `len` 用来获取数组或者切片的长度
* Test coverage tool
* `reflect.DeepEqual` 为什么它很有用，但却会降低代码的类型安全性

我们已经对整数使用了切片和数组，但它们适用于任何其他类型也包括数组/切片本身。
所以你可以声明一个变量 `[][]string`，如果需要的话。

想深入了解 slice 请查看[Check out the Go blog post on slices][blog-slice]。
尝试为 demo 写更多的测试。

除了编写测试之外，另一种方便的使用 Go 的方法是 Go playground。你可以尝试很多东西，你可以很容易地分享你的代码你需要问问题。
[I have made a go playground with a slice in it for you to experiment with.](https://play.golang.org/p/ICCWcRGIO68)

[Here is an example](https://play.golang.org/p/bTrRmYfNYCp) of slicing an array
and how changing the slice affects the original array; but a "copy" of the slice
will not affect the original array.
[Another example](https://play.golang.org/p/Poth8JS28sc) of why it's a good idea
to make a copy of a slice after slicing a very large slice.

[for]: ../iteration.md#
[blog-slice]: https://blog.golang.org/go-slices-usage-and-internals
[deepEqual]: https://golang.org/pkg/reflect/#DeepEqual
[slice]: https://golang.org/doc/effective_go.html#slices
