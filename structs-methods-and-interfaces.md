# Structs, methods & interfaces

**[You can find all the code for this chapter here](https://github.com/quii/learn-go-with-tests/tree/main/structs)**

假设我们需要一些几何代码来计算给定高度和宽度的矩形的周长。
我们可以编写一个 `Perimeter(width float64, height float64)` 函数，其中 `float64` 用于浮点数，如 `123.45`。

到现在为止，您对 TDD cycle 应该很熟悉了。

## 首先写测试代码

```go
func TestPerimeter(t *testing.T) {
    got := Perimeter(10.0, 10.0)
    want := 40.0

    if got != want {
        t.Errorf("got %.2f want %.2f", got, want)
    }
}
```

注意其中的 format string。这个 `f` 代表的是我们的 `float64`，`.2` 代表打印 2 位小数。

## Try to run the test

`./shapes_test.go:6:9: undefined: Perimeter`

## Write the minimal amount of code for the test to run and check the failing test output

```go
func Perimeter(width float64, height float64) float64 {
    return 0
}
```

Results in `shapes_test.go:10: got 0.00 want 40.00`.

## Write enough code to make it pass

```go
func Perimeter(width float64, height float64) float64 {
    return 2 * (width + height)
}
```

现在我们创建一个 `Area(width, height float64)` 函数，它将返回矩形的面积。

Try to do it yourself, following the TDD cycle.

You should end up with tests like this

```go
func TestPerimeter(t *testing.T) {
    got := Perimeter(10.0, 10.0)
    want := 40.0

    if got != want {
        t.Errorf("got %.2f want %.2f", got, want)
    }
}

func TestArea(t *testing.T) {
    got := Area(12.0, 6.0)
    want := 72.0

    if got != want {
        t.Errorf("got %.2f want %.2f", got, want)
    }
}
```

And code like this

```go
func Perimeter(width float64, height float64) float64 {
    return 2 * (width + height)
}

func Area(width float64, height float64) float64 {
    return width * height
}
```

## Refactor

 我们的代码完成了任务，但它没有包含任何关于矩形的显式内容。
 粗心的开发人员可能试图为这些函数提供三角形的宽度和高度，而没有意识到它们将返回错误的答案。


我们可以给函数起一个更具体的名字，比如 `RectangleArea`。一个更简洁的解决方案是定义我们自己的名为 `Rectangle` 的类型，它为我们封装了这个概念。

我们可以使用 **struct** 创建一个简单的类型。[A struct](https://golang.org/ref/spec#Struct_types) 只是用于存储数据的指定字段集合。

声明一个结构如下：

```go
type Rectangle struct {
    Width float64
    Height float64
}
```

现在重构测试，使用 `Rectangle` 替代 `float64`。

```go
func TestPerimeter(t *testing.T) {
    rectangle := Rectangle{10.0, 10.0}
    got := Perimeter(rectangle)
    want := 40.0

    if got != want {
        t.Errorf("got %.2f want %.2f", got, want)
    }
}

func TestArea(t *testing.T) {
    rectangle := Rectangle{12.0, 6.0}
    got := Area(rectangle)
    want := 72.0

    if got != want {
        t.Errorf("got %.2f want %.2f", got, want)
    }
}
```

Remember to run your tests before attempting to fix, you should get a helpful error like

```text
./shapes_test.go:7:18: not enough arguments in call to Perimeter
    have (Rectangle)
    want (float64, float64)
```

You can access the fields of a struct with the syntax of `myStruct.field`.

Change the two functions to fix the test.

```go
func Perimeter(rectangle Rectangle) float64 {
    return 2 * (rectangle.Width + rectangle.Height)
}

func Area(rectangle Rectangle) float64 {
    return rectangle.Width * rectangle.Height
}
```


我希望你会同意，传递一个 `Rectangle` 给函数更清楚地传达了我们的意图，但使用结构体还有更多的好处，我们将继续讨论。

接下来我们将为原型编写一个 `Area` 函数。

## Write the test first

```go
func TestArea(t *testing.T) {

    t.Run("rectangles", func(t *testing.T) {
        rectangle := Rectangle{12, 6}
        got := Area(rectangle)
        want := 72.0

        if got != want {
            t.Errorf("got %g want %g", got, want)
        }
    })

    t.Run("circles", func(t *testing.T) {
        circle := Circle{10}
        got := Area(circle)
        want := 314.1592653589793

        if got != want {
            t.Errorf("got %g want %g", got, want)
        }
    })

}
```

 如你所见，`f` 已经被 `g` 取代了，
 使用 `f` 可能很难知道精确的小数，
 使用 `g`，我们在错误消息中得到一个完整的十进制数
  \([fmt options](https://golang.org/pkg/fmt/)\).

## Try to run the test

`./shapes_test.go:28:13: undefined: Circle`

## Write the minimal amount of code for the test to run and check the failing test output

We need to define our `Circle` type.

```go
type Circle struct {
    Radius float64
}
```

现在运行测试

`./shapes_test.go:29:14: cannot use circle (type Circle) as type Rectangle in argument to Area`

有些变成语言允许你编写如下的代码：

```go
func Area(circle Circle) float64 { ... }
func Area(rectangle Rectangle) float64 { ... }
```

但是在 Go 里面不行

`./shapes.go:20:32: Area redeclared in this block`

我们有两个选择：

* 你可以在不同的 _packages_ 中声明相同的函数名。所以我们可以在一个新包中创建 `Area(Circle)`，但在这里感觉有点过分了。
* 可以在新定义的类型上定义 [_methods_](https://golang.org/ref/spec#Method_declarations)。

### What are methods?

目前为止我们还只写了函数，但是我们使用了方法。当我们调用 `t.Errorf`，调用的是 `testing.T` 实例 `t` 的 `Errorf` 方法。

方法是一个带有接收器的函数。方法声明将标识符(方法名)绑定到方法，并将方法与接收方的基类型关联起来。

方法与函数非常相似，但它们是通过在特定类型的实例上调用它们来调用的。你可以在任何你喜欢的地方调用函数，比如 `Area(rectangle)`，你只能在“things”上调用方法。

一个示例将会有所帮助，因此让我们先更改测试，以调用方法，然后修复代码。

```go
func TestArea(t *testing.T) {

    t.Run("rectangles", func(t *testing.T) {
        rectangle := Rectangle{12, 6}
        got := rectangle.Area()
        want := 72.0

        if got != want {
            t.Errorf("got %g want %g", got, want)
        }
    })

    t.Run("circles", func(t *testing.T) {
        circle := Circle{10}
        got := circle.Area()
        want := 314.1592653589793

        if got != want {
            t.Errorf("got %g want %g", got, want)
        }
    })

}
```

如果现在运行测试，将得到

```text
./shapes_test.go:19:19: rectangle.Area undefined (type Rectangle has no field or method Area)
./shapes_test.go:29:16: circle.Area undefined (type Circle has no field or method Area)
```

> 类型 Circle 没有字段或方法 Area

我想重申一下这里的编译器有多棒。花时间慢慢地阅读你得到的错误消息是非常重要的，从长远来看，这将帮助你。

## Write the minimal amount of code for the test to run and check the failing test output

为我们的类型添加一些方法

```go
type Rectangle struct {
    Width  float64
    Height float64
}

func (r Rectangle) Area() float64  {
    return 0
}

type Circle struct {
    Radius float64
}

func (c Circle) Area() float64  {
    return 0
}
```

声明方法的语法几乎与函数相同，这是因为它们非常相似。唯一的区别是方法 receiver `func (receiverName ReceiverType) MethodName(args)` 的语法。

当你的方法被这种类型的变量调用时，你通过 `receiverName` 变量获得对它数据的引用。在许多其他编程语言中，这是隐式完成的，你通过 `this` 访问接收器。

Go 中的约定是让 receiver 变量是该类型的第一个字母。

```go
r Rectangle
```

如果您尝试重新运行测试，它们现在应该编译并给出一些失败的输出。

## Write enough code to make it pass

现在让我们通过固定我们的新方法来通过矩形测试

```go
func (r Rectangle) Area() float64  {
    return r.Width * r.Height
}
```

如果重新运行测试，矩形测试应该通过，但圆形测试仍然失败。

为了让 circle 的 `Area` 函数通过，我们将从 `math` 包中借用 `Pi`常量(记住要导入它)。

```go
func (c Circle) Area() float64  {
    return math.Pi * c.Radius * c.Radius
}
```

## Refactor

我们的测试有些重复。

我们所要做的就是取一个 _shapes_ 集合，对它们调用 `Area()` 方法，然后检查结果。

我们希望能够编写一些 `checkArea` 函数，我们可以传递 `Rectangle` 和 `Circle`，但如果我们试图传递一些不是形状的东西，则无法编译。

有了 Go，我们可以通过接口来实现这个意图。

[Interfaces](https://golang.org/ref/spec#Interface_types) 在像 Go 这样的静态类型语言中是一个非常强大的概念，因为它们允许您创建可用于不同类型的函数，并创建高度解耦的代码，同时仍然保持类型安全。

让我们通过重构测试来引入这一点。

```go
func TestArea(t *testing.T) {

    checkArea := func(t testing.TB, shape Shape, want float64) {
        t.Helper()
        got := shape.Area()
        if got != want {
            t.Errorf("got %g want %g", got, want)
        }
    }

    t.Run("rectangles", func(t *testing.T) {
        rectangle := Rectangle{12, 6}
        checkArea(t, rectangle, 72.0)
    })

    t.Run("circles", func(t *testing.T) {
        circle := Circle{10}
        checkArea(t, circle, 314.1592653589793)
    })

}
```

我们正在创建一个 helper 函数，就像我们在其他练习中所做的那样，但这次我们要求传入一个 `Shape`。如果我们试图用非形状的东西调用它，那么它将无法编译。

物体是如何变成形状的?我们只需要使用接口声明告诉 Go 一个 Shape 是什么

```go
type Shape interface {
    Area() float64
}
```

我们正在创建一个新的 `type`，就像我们对 `Rectangle` 和 `Circle` 做的那样，但这次它是一个 `interface` 而不是一个 `struct`。

一旦将此添加到代码中，测试就会通过。

### Wait, what?

这与大多数其他编程语言中的接口非常不同。通常情况下，你必须写代码说 `My type Foo implements interface Bar`。

但在我们的案例中

* `Rectangle` 有一个方法 `Area` 它返回 `float64`，因此它实现了 `Shape` 接口
* `Circle` 有一个方法 `Area` 它返回 `float64`，因此它实现了 `Shape` 接口
* `string` 没有这个方法，因此它没有实现 `Shape` 接口
* etc.

在Go中，接口分辨是隐式的。如果传入的类型与接口所要求的类型匹配，则接口将编译。

### Decoupling

请注意，我们的 helper 并不需要关心形状是 `Rectangle`、 `Circle` 还是 `Triangle`。
通过声明一个接口，helper 就可以从具体的类型中分离出来，并且只拥有完成工作所需的方法。

这种使用接口来声明“只需要”的方法在软件设计中是非常重要的，在后面的小节中将详细介绍。

## Further refactoring

现在你已经对结构有了一些了解，我们可以介绍“表驱动测试”了。

当您想要构建一个可以以相同方式测试的测试用例列表时，[Table driven tests](https://github.com/golang/go/wiki/TableDrivenTests) 是有用的。

```go
func TestArea(t *testing.T) {

    areaTests := []struct {
        shape Shape
        want  float64
    }{
        {Rectangle{12, 6}, 72.0},
        {Circle{10}, 314.1592653589793},
    }

    for _, tt := range areaTests {
        got := tt.shape.Area()
        if got != tt.want {
            t.Errorf("got %g want %g", got, tt.want)
        }
    }

}
```

这里唯一的新语法是创建一个“匿名结构” areaTests。我们使用 `[]struct` 来声明一个结构块，它有两个字段: `shape` 和 `want` 。

然后，就像对任何其他切片一样，使用 struct 字段对它们进行迭代，以运行测试。

您可以看到，对于开发人员来说，引入一个新的形状，实现 `Area`，然后将其添加到测试用例中是非常容易的。
此外，如果在 `Area` 中发现了一个 bug，那么在修复它之前添加一个新的测试用例来测试它是非常容易的。

基于表的测试可能是工具箱中的一个重要项目，但请确保您需要在测试中添加额外的噪声。
如果您希望测试接口的各种实现，或者如果传入函数的数据有许多不同的需求需要测试，那么它们是非常合适的。

让我们通过添加另一个形状并测试它来演示所有这些;一个三角形。

## Write the test first

Adding a new test for our new shape is very easy. Just add `{Triangle{12, 6}, 36.0},` to our list.

为我们的新形状添加一个新的测试是非常容易的。只需将 `{Triangle{12, 6}, 36.0},` 添加到我们的列表中。



```go
func TestArea(t *testing.T) {

    areaTests := []struct {
        shape Shape
        want  float64
    }{
        {Rectangle{12, 6}, 72.0},
        {Circle{10}, 314.1592653589793},
        {Triangle{12, 6}, 36.0},
    }

    for _, tt := range areaTests {
        got := tt.shape.Area()
        if got != tt.want {
            t.Errorf("got %g want %g", got, tt.want)
        }
    }

}
```

## Try to run the test

Remember, keep trying to run the test and let the compiler guide you toward a solution.

## Write the minimal amount of code for the test to run and check the failing test output

`./shapes_test.go:25:4: undefined: Triangle`

We have not defined Triangle yet

```go
type Triangle struct {
    Base   float64
    Height float64
}
```

Try again

```text
./shapes_test.go:25:8: cannot use Triangle literal (type Triangle) as type Shape in field value:
    Triangle does not implement Shape (missing Area method)
```

It's telling us we cannot use a Triangle as a shape because it does not have an `Area()` method, so add an empty implementation to get the test working

```go
func (t Triangle) Area() float64 {
    return 0
}
```

Finally the code compiles and we get our error

`shapes_test.go:31: got 0.00 want 36.00`

## Write enough code to make it pass

```go
func (t Triangle) Area() float64 {
    return (t.Base * t.Height) * 0.5
}
```

And our tests pass!

## Refactor

Again, the implementation is fine but our tests could do with some improvement.

When you scan this

```go
{Rectangle{12, 6}, 72.0},
{Circle{10}, 314.1592653589793},
{Triangle{12, 6}, 36.0},
```

It's not immediately clear what all the numbers represent and you should be aiming for your tests to be easily understood.

So far you've only been shown syntax for creating instances of structs `MyStruct{val1, val2}` but you can optionally name the fields.

Let's see what it looks like

```go
        {shape: Rectangle{Width: 12, Height: 6}, want: 72.0},
        {shape: Circle{Radius: 10}, want: 314.1592653589793},
        {shape: Triangle{Base: 12, Height: 6}, want: 36.0},
```

In [Test-Driven Development by Example](https://g.co/kgs/yCzDLF) Kent Beck refactors some tests to a point and asserts:

> The test speaks to us more clearly, as if it were an assertion of truth, **not a sequence of operations**

\(emphasis mine\)

Now our tests \(at least the list of cases\) make assertions of truth about shapes and their areas.

## Make sure your test output is helpful

Remember earlier when we were implementing `Triangle` and we had the failing test? It printed `shapes_test.go:31: got 0.00 want 36.00`.

We knew this was in relation to `Triangle` because we were just working with it, but what if a bug slipped in to the system in one of 20 cases in the table? How would a developer know which case failed? This is not a great experience for the developer, they will have to manually look through the cases to find out which case actually failed.

We can change our error message into `%#v got %g want %g`. The `%#v` format string will print out our struct with the values in its field, so the developer can see at a glance the properties that are being tested.

To increase the readability of our test cases further we can rename the `want` field into something more descriptive like `hasArea`.

One final tip with table driven tests is to use `t.Run` and to name the test cases.

By wrapping each case in a `t.Run` you will have clearer test output on failures as it will print the name of the case

```text
--- FAIL: TestArea (0.00s)
    --- FAIL: TestArea/Rectangle (0.00s)
        shapes_test.go:33: main.Rectangle{Width:12, Height:6} got 72.00 want 72.10
```

And you can run specific tests within your table with `go test -run TestArea/Rectangle`.

Here is our final test code which captures this

```go
func TestArea(t *testing.T) {

    areaTests := []struct {
        name    string
        shape   Shape
        hasArea float64
    }{
        {name: "Rectangle", shape: Rectangle{Width: 12, Height: 6}, hasArea: 72.0},
        {name: "Circle", shape: Circle{Radius: 10}, hasArea: 314.1592653589793},
        {name: "Triangle", shape: Triangle{Base: 12, Height: 6}, hasArea: 36.0},
    }

    for _, tt := range areaTests {
        // using tt.name from the case to use it as the `t.Run` test name
        t.Run(tt.name, func(t *testing.T) {
            got := tt.shape.Area()
            if got != tt.hasArea {
                t.Errorf("%#v got %g want %g", tt.shape, got, tt.hasArea)
            }
        })

    }

}
```

## Wrapping up

This was more TDD practice, iterating over our solutions to basic mathematic problems and learning new language features motivated by our tests.

* Declaring structs to create your own data types which lets you bundle related data together and make the intent of your code clearer
* Declaring interfaces so you can define functions that can be used by different types \([parametric polymorphism](https://en.wikipedia.org/wiki/Parametric_polymorphism)\)
* Adding methods so you can add functionality to your data types and so you can implement interfaces
* Table based tests to make your assertions clearer and your suites easier to extend & maintain

This was an important chapter because we are now starting to define our own types. In statically typed languages like Go, being able to design your own types is essential for building software that is easy to understand, to piece together and to test.

Interfaces are a great tool for hiding complexity away from other parts of the system. In our case our test helper _code_ did not need to know the exact shape it was asserting on, only how to "ask" for its area.

As you become more familiar with Go you start to see the real strength of interfaces and the standard library. You'll learn about interfaces defined in the standard library that are used _everywhere_ and by implementing them against your own types you can very quickly re-use a lot of great functionality.
