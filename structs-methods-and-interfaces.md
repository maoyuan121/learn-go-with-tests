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

## 为要运行的测试编写最小数量的代码，并检查失败的测试输出

```go
func Perimeter(width float64, height float64) float64 {
    return 0
}
```

Results in `shapes_test.go:10: got 0.00 want 40.00`.

## 编写足够的代码让它通过
   
```go
func Perimeter(width float64, height float64) float64 {
    return 2 * (width + height)
}
```

现在我们创建一个 `Area(width, height float64)` 函数，它将返回矩形的面积。

试着自己去做，遵循 TDD 循环。

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

代码如下

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

记住在尝试修复之前运行测试，您应该会得到一个有用的错误，如

```text
./shapes_test.go:7:18: not enough arguments in call to Perimeter
    have (Rectangle)
    want (float64, float64)
```

你可以使用 `myStruct.field` 的语法来访问结构体的字段。

更改这两个函数以修复测试。

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

## 运行测试

`./shapes_test.go:28:13: undefined: Circle`

## 为要运行的测试编写最小数量的代码，并检查失败的测试输出

我们需要定义 `Circle` 类型。

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

Go 中的约定是让 receiver 变量是该类型的**第一个字母**。

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

在 Go 中，接口分辨是隐式的。如果传入的类型与接口所要求的类型匹配，则接口将编译。

### 解耦

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

请记住，不断尝试运行测试，并让编译器引导您找到解决方案。

## Write the minimal amount of code for the test to run and check the failing test output

`./shapes_test.go:25:4: undefined: Triangle`

我们现在还没有定义 Triangle

```go
type Triangle struct {
    Base   float64
    Height float64
}
```

再次运行

```text
./shapes_test.go:25:8: cannot use Triangle literal (type Triangle) as type Shape in field value:
    Triangle does not implement Shape (missing Area method)
```

它告诉我们不能使用 Triangle 作为一个 shape，因为它没有 `Area()` 方法，所以添加一个空的实现来让测试工作

```go
func (t Triangle) Area() float64 {
    return 0
}
```

最后代码编译通过，我们得到了错误

`shapes_test.go:31: got 0.00 want 36.00`

## Write enough code to make it pass

```go
func (t Triangle) Area() float64 {
    return (t.Base * t.Height) * 0.5
}
```

现在我们的测试通过了！

## Refactor

同样，实现很好，但是我们的测试需要一些改进。

当你扫描这个的时候

```go
{Rectangle{12, 6}, 72.0},
{Circle{10}, 314.1592653589793},
{Triangle{12, 6}, 36.0},
```

并不能立即明确所有的数字代表什么，你应该让你的测试更容易理解。

到目前为止，您只展示了创建结构实例 `MyStruct{val1, val2}` 的语法，但您可以选择命名字段。

让我们看看它是什么样的

```go
        {shape: Rectangle{Width: 12, Height: 6}, want: 72.0},
        {shape: Circle{Radius: 10}, want: 314.1592653589793},
        {shape: Triangle{Base: 12, Height: 6}, want: 36.0},
```

在 [Test-Driven Development by Example](https://g.co/kgs/yCzDLF) 中 Kent Beck 对一些测试进行了重构，并断言:

> 这个测试对我们讲得更清楚，就好像它是对真理的断言，而不是一系列操作

\(我强调\)

现在我们的测试(至少是一系列的用例)断言形状和它们的面积是真实的。

## 确保您的测试输出是有用的

还记得之前我们执行 `Triangle` 时遇到的失败测试吗？它打印的 `shapes_test.go:31: got 0.00 want 36.00`.。

我们知道这与 `Triangle` 有关，因为我们只是在使用它，但如果在表中20个案例中的一个案例中，系统中出现了漏洞怎么办?开发人员如何知道哪个案例失败了?
这对开发人员来说不是一个很好的体验，他们将不得不手动查看用例，以找出实际失败的用例。

我们可以将错误消息改为 `%#v got %g want %g`。`%#v` 格式字符串将打印出我们的结构，在它的字段中包含值，因此开发人员可以一眼看到正在测试的属性。

为了进一步增加测试用例的可读性，我们可以将 `want` 字段重命名为更具描述性的东西，比如 `hasArea`。

关于表驱动测试的最后一个技巧是使用 `t.Run`  并命名测试用例。

用 `t.Run` 包装每个用例，您将有更清晰的失败测试输出，因为它将打印用例的名称


```text
--- FAIL: TestArea (0.00s)
    --- FAIL: TestArea/Rectangle (0.00s)
        shapes_test.go:33: main.Rectangle{Width:12, Height:6} got 72.00 want 72.10
```

您还可以使用 `go test -run TestArea/Rectangle` 在表中运行特定的测试。

这是我们捕获这个的最终测试代码

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

## 总结

这是更多的 TDD 实践，迭代我们对基本数学问题的解决方案，并学习由测试激发的新语言特性。

* 声明结构来创建自己的数据类型，这样可以将相关数据捆绑在一起，并使代码的意图更清晰
* 声明接口以便定义可被不同类型使用的函数\([参数多态性](https://en.wikipedia.org/wiki/Parametric_polymorphism)\)
* 添加方法，这样你就可以向数据类型添加功能，这样你就可以实现接口
* 基于表的测试，使您的断言更清晰，您的套件更容易扩展和维护

这一章很重要，因为我们现在开始定义自己的类型。在像 Go 这样的静态类型语言中，能够设计自己的类型对于构建易于理解、组装和测试的软件至关重要。

接口是一个很好的工具，可以将复杂性隐藏起来，不让系统的其他部分看到。在我们的例子中，我们的测试助手 _code_ 不需要知道它断言的确切形状，只需要知道如何“ask”它的面积。

随着您对 Go 越来越熟悉，您开始看到接口和标准库的真正优势。您将了解在标准库中定义的接口，这些接口在任何地方都可以使用，通过针对您自己的类型实现它们，您可以非常快速地重用许多很棒的功能。

