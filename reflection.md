# Reflection

**[You can find all the code for this chapter here](https://github.com/quii/learn-go-with-tests/tree/main/reflection)**

[From Twitter](https://twitter.com/peterbourgon/status/1011403901419937792?s=09)

> golang挑战：编写一个函数 `walk(x interface{}， fn func(string))`，它接受一个struct `x`，并调用 `fn` 用于所有在其中找到的字符串字段。难度等级:递归。
  
要做到这个我们需要使用 _反射_。

> 计算机中的反射是程序检查自身结构的能力，特别是通过类型;它是元编程的一种形式。这也是一大困惑之源。
  
From [The Go Blog: Reflection](https://blog.golang.org/laws-of-reflection)

## What is `interface`?

我们很喜欢 Go 提供的类型安全功能，这些功能可以处理已知类型，如 `string`、`int` 和我们自己的类型，如 `BankAccount`。

这意味着我们可以免费获得一些文档，如果你试图将错误的类型传递给函数，编译器将会报错。

不过，您可能会遇到这样的情况：您想编写一个在编译时不知道类型的函数。

Go 让我们用类型 `interface{}` 来解决这个问题，你可以把它看作是 _任何_ 类型。

因此 `walk(x interface{}, fn func(string))` 将接收 `x` 的任何值。

### 所以为什么不把 `interface` 用于所有的事情，并拥有真正灵活的功能呢?

- As a user of a function that takes `interface` you lose type safety. 
What if you meant to pass `Foo.bar` of type `string` into a function but instead did `Foo.baz` which is an `int`? 
The compiler won't be able to inform you of your mistake. 
You also have no idea _what_ you're allowed to pass to a function.
 Knowing that a function takes a `UserService` for instance is very useful.
- As a writer of such a function, 
you have to be able to inspect _anything_ that has been passed to you and try and figure out what the type is and what you can do with it. 
This is done using _reflection_. 
This can be quite clumsy and difficult to read and is generally less performant (as you have to do checks at runtime).

简而言之，只有在真正需要时才使用反射。

如果你想要多态函数，考虑是否可以围绕一个 interface(不是 `interface`，令人困惑)来设计它，这样用户就可以使用多种类型的函数，如果他们实现了任何你需要的方法，让你的函数工作。

我们的函数需要能够处理许多不同的东西。一如既往，我们将采用迭代的方法，为我们想要支持的每一个新事物编写测试，并在此过程中进行重构，直到完成为止。

## Write the test first

我们想用一个结构体来调用函数，这个结构体中有一个字符串字段(`x`)。然后我们可以监视传入的函数(`fn`)，看看它是否被调用。

```go
func TestWalk(t *testing.T) {

    expected := "Chris"
    var got []string

    x := struct {
        Name string
    }{expected}

    walk(x, func(input string) {
        got = append(got, input)
    })

    if len(got) != 1 {
        t.Errorf("wrong number of function calls, got %d want %d", len(got), 1)
    }
}
```

- 我们想要存储一个字符串切片(`got`)，它存储通过 `walk` 传递给 `fn` 的字符串。通常在前面的章节中，我们已经为这个做了专门的类型来监视函数/方法调用，
但在这种情况下，我们可以传递一个匿名函数给 `fn`，它 closes over `got`。
- 我们使用一个匿名的 `struct`，带有字符串类型的 `Name` 字段，以获得最简单的"快乐"路径。
- 最后，调用 `walk` 与 `x`，现在只是检查 `got` 的长度，一旦我们有一些非常基本的工作，我们将更具体地使用我们的断言。
 
  

## Try to run the test

```
./reflection_test.go:21:2: undefined: walk
```

## Write the minimal amount of code for the test to run and check the failing test output

我们需要定义 `walk`

```go
func walk(x interface{}, fn func(input string)) {

}
```

再次运行测试

```
=== RUN   TestWalk
--- FAIL: TestWalk (0.00s)
    reflection_test.go:19: wrong number of function calls, got 0 want 1
FAIL
```

## Write enough code to make it pass

我们使用任意字符出调用 spy 让测试通过。

```go
func walk(x interface{}, fn func(input string)) {
    fn("I still can't believe South Korea beat Germany 2-0 to put them last in their group")
}
```

现在测试应该是通过了。我们需要做的下一件事是对 `fn` 被调用的对象做一个更具体的断言。

## Write the test first

在现有的测试中添加以下内容，以检查传递给 `fn` 的字符串是否正确

```go
if got[0] != expected {
    t.Errorf("got %q, want %q", got[0], expected)
}
```

## Try to run the test

```
=== RUN   TestWalk
--- FAIL: TestWalk (0.00s)
    reflection_test.go:23: got 'I still can't believe South Korea beat Germany 2-0 to put them last in their group', want 'Chris'
FAIL
```

## Write enough code to make it pass

```go
func walk(x interface{}, fn func(input string)) {
    val := reflect.ValueOf(x)
    field := val.Field(0)
    fn(field.String())
}
```

这段代码非常不安全，也非常幼稚，但是请记住，当我们处于“红色”状态(测试失败)时，我们的目标是尽可能少地编写代码。然后我们编写更多的测试来解决我们所关注的问题。

我们需要用反射来看看 `x`，看看它的属性。

[reflect package](https://godoc.org/reflect)有一个函数 `ValueOf`，它返回给定变量的 `Value`。它可以让我们检查一个值，包括在下一行中使用的字段。

然后我们对传入的值做一些非常乐观的假设

- 我们看第一个也是唯一的字段，可能根本没有字段会引起 panic
- 然后我们调用 `String()`，它返回基础值作为一个字符串，但我们知道，如果字段不是字符串，它将是错误的。


## Refactor

我们的代码传递的是简单的情况，但我们知道我们的代码有很多缺点。

我们会写一些测试在这里我们传递不同的值并检查调用 `fn` 的字符串数组。

我们应该将测试重构为基于表的测试，以使继续测试新场景变得更容易。


```go
func TestWalk(t *testing.T) {

    cases := []struct{
        Name string
        Input interface{}
        ExpectedCalls []string
    } {
        {
            "Struct with one string field",
            struct {
                Name string
            }{ "Chris"},
            []string{"Chris"},
        },
    }

    for _, test := range cases {
        t.Run(test.Name, func(t *testing.T) {
            var got []string
            walk(test.Input, func(input string) {
                got = append(got, input)
            })

            if !reflect.DeepEqual(got, test.ExpectedCalls) {
                t.Errorf("got %v, want %v", got, test.ExpectedCalls)
            }
        })
    }
}
```

现在我们可以很容易地添加一个场景，看看如果有多个字符串字段会发生什么。

## Write the test first

将以下场景添加到 `cases` 中。

```go
{
    "Struct with two string fields",
    struct {
        Name string
        City string
    }{"Chris", "London"},
    []string{"Chris", "London"},
}
```

## Try to run the test

```
=== RUN   TestWalk/Struct_with_two_string_fields
    --- FAIL: TestWalk/Struct_with_two_string_fields (0.00s)
        reflection_test.go:40: got [Chris], want [Chris London]
```

## Write enough code to make it pass

```go
func walk(x interface{}, fn func(input string)) {
    val := reflect.ValueOf(x)

    for i:=0; i<val.NumField(); i++ {
        field := val.Field(i)
        fn(field.String())
    }
}
```

`val` 有一个方法 `NumField` ，它返回值中的字段数量。这让我们可以遍历字段并调用 `fn`，从而通过我们的测试。



## Refactor

这里似乎没有任何明显的重构可以改进代码，所以让我们继续。

`walk` 的下一个缺点是它假定每个字段都是一个 `字符串`。让我们为这个场景编写一个测试。



## Write the test first

Add the following case

```go
{
    "Struct with non string field",
    struct {
        Name string
        Age  int
    }{"Chris", 33},
    []string{"Chris"},
},
```

## Try to run the test

```
=== RUN   TestWalk/Struct_with_non_string_field
    --- FAIL: TestWalk/Struct_with_non_string_field (0.00s)
        reflection_test.go:46: got [Chris <int Value>], want [Chris]
```

## Write enough code to make it pass

我们需要检查字段的类型是否为 `string`。

```go
func walk(x interface{}, fn func(input string)) {
    val := reflect.ValueOf(x)

    for i := 0; i < val.NumField(); i++ {
        field := val.Field(i)

        if field.Kind() == reflect.String {
            fn(field.String())
        }
    }
}
```

我们可以通过查看它的[`Kind`](https://godoc.org/reflect#Kind)来做到这一点。

## Refactor

现在看起来代码已经足够合理了。

下一个场景是如果它不是一个“扁平的” `struct`呢?换句话说，如果我们有一个带有嵌套字段的 `struct` 会发生什么?



## Write the test first

我们一直在使用匿名结构语法来为我们的测试特别声明类型，所以我们可以继续这样做

```go
{
    "Nested fields",
    struct {
        Name string
        Profile struct {
            Age  int
            City string
        }
    }{"Chris", struct {
        Age  int
        City string
    }{33, "London"}},
    []string{"Chris", "London"},
},
```

但我们可以看到，当你得到内部匿名结构时语法会有点混乱。[有人提议让它的语法更好](https://github.com/golang/go/issues/12854)。

让我们通过为这个场景创建一个已知类型并在测试中引用它来重构它。有一点间接的，我们的测试的一些代码是在测试之外的，但读者应该能够通过查看初始化推断 `struct` 的结构。

在您的测试文件中添加以下类型声明

```go
type Person struct {
    Name    string
    Profile Profile
}

type Profile struct {
    Age  int
    City string
}
```

现在我们可以把这个添加到我们的案例中，比以前更清楚了

```go
{
    "Nested fields",
    Person{
        "Chris",
        Profile{33, "London"},
    },
    []string{"Chris", "London"},
},
```

## Try to run the test

```
=== RUN   TestWalk/Nested_fields
    --- FAIL: TestWalk/Nested_fields (0.00s)
        reflection_test.go:54: got [Chris], want [Chris London]
```

问题是我们只对类型层次结构的第一级的字段进行迭代。

## Write enough code to make it pass

```go
func walk(x interface{}, fn func(input string)) {
    val := reflect.ValueOf(x)

    for i := 0; i < val.NumField(); i++ {
        field := val.Field(i)

        if field.Kind() == reflect.String {
            fn(field.String())
        }

        if field.Kind() == reflect.Struct {
            walk(field.Interface(), fn)
        }
    }
}
```

解决方案很简单，我们再次检查它的 `Kind`，如果它碰巧是 `struct`，我们只在内部 `struct` 调用 `walk`。

## Refactor

```go
func walk(x interface{}, fn func(input string)) {
    val := reflect.ValueOf(x)

    for i := 0; i < val.NumField(); i++ {
        field := val.Field(i)

        switch field.Kind() {
        case reflect.String:
            fn(field.String())
        case reflect.Struct:
            walk(field.Interface(), fn)
        }
    }
}
```

当你对同一个值进行多次比较时，一般来说，重构为一个 `switch` 将提高可读性，使代码更容易扩展。

如果传入的结构体的值是一个指针呢?

## Write the test first

Add this case

```go
{
    "Pointers to things",
    &Person{
        "Chris",
        Profile{33, "London"},
    },
    []string{"Chris", "London"},
},
```

## Try to run the test

```
=== RUN   TestWalk/Pointers_to_things
panic: reflect: call of reflect.Value.NumField on ptr Value [recovered]
    panic: reflect: call of reflect.Value.NumField on ptr Value
```

## Write enough code to make it pass

```go
func walk(x interface{}, fn func(input string)) {
    val := reflect.ValueOf(x)

    if val.Kind() == reflect.Ptr {
        val = val.Elem()
    }

    for i := 0; i < val.NumField(); i++ {
        field := val.Field(i)

        switch field.Kind() {
        case reflect.String:
            fn(field.String())
        case reflect.Struct:
            walk(field.Interface(), fn)
        }
    }
}
```

你不能在指针 `Value` 上使用 `NumField`，我们需要在使用 `Elem()` 之前提取底层值。


## Refactor

让我们封装提取从给定的 `interface{}` 提取 `reflect.Value` 的职责，将它封装成一个函数。

```go
func walk(x interface{}, fn func(input string)) {
    val := getValue(x)

    for i := 0; i < val.NumField(); i++ {
        field := val.Field(i)

        switch field.Kind() {
        case reflect.String:
            fn(field.String())
        case reflect.Struct:
            walk(field.Interface(), fn)
        }
    }
}

func getValue(x interface{}) reflect.Value {
    val := reflect.ValueOf(x)

    if val.Kind() == reflect.Ptr {
        val = val.Elem()
    }

    return val
}
```

这实际上增加了更多的代码，但我觉得抽象层是正确的。

- 得到 `x` 的 `reflect.Value`，所以我可以检查它，我不在乎怎么检查。
- 遍历字段，根据其类型执行任何需要执行的操作。

接下来，我们需要覆盖切片。

## Write the test first

```go
{
    "Slices",
    []Profile {
        {33, "London"},
        {34, "Reykjavík"},
    },
    []string{"London", "Reykjavík"},
},
```

## Try to run the test

```
=== RUN   TestWalk/Slices
panic: reflect: call of reflect.Value.NumField on slice Value [recovered]
    panic: reflect: call of reflect.Value.NumField on slice Value
```

## Write the minimal amount of code for the test to run and check the failing test output

这类似于之前的指针场景，我们试图在 `reflect.Value` 上调用 `NumField`。但它没有，因为它不是 struct。


## Write enough code to make it pass

```go
func walk(x interface{}, fn func(input string)) {
    val := getValue(x)

    if val.Kind() == reflect.Slice {
        for i:=0; i< val.Len(); i++ {
            walk(val.Index(i).Interface(), fn)
        }
        return
    }

    for i := 0; i < val.NumField(); i++ {
        field := val.Field(i)

        switch field.Kind() {
        case reflect.String:
            fn(field.String())
        case reflect.Struct:
            walk(field.Interface(), fn)
        }
    }
}
```

## Refactor

这个有用，但很恶心。不用担心，我们有测试支持的工作代码，所以我们可以随意修改。

如果你想得稍微抽象一点，we want to call `walk` on either

- 一个结构中的所有字段
- slice 中的每个 _thing_ 

我们的代码目前就是这样做的，但并没有很好地反映出来。我们只是在开始时检查它是否是一个切片(用一个 `return` 来停止其余的代码执行)，如果不是，我们就假设它是一个结构。

让我们重新编写代码，以先检查类型，然后继续工作。

```go
func walk(x interface{}, fn func(input string)) {
    val := getValue(x)

    switch val.Kind() {
    case reflect.Struct:
        for i:=0; i<val.NumField(); i++ {
            walk(val.Field(i).Interface(), fn)
        }
    case reflect.Slice:
        for i:=0; i<val.Len(); i++ {
            walk(val.Index(i).Interface(), fn)
        }
    case reflect.String:
        fn(val.String())
    }
}
```

看起来好多了!如果它是一个结构体或一个切片，则对其值进行迭代，并对每个值调用 `walk`。否则，如果是 `reflect.String` 我们可以调用 `fn`

尽管如此，对我来说，感觉还可以更好。这里重复了遍历字段/值的操作，然后调用 `walk`，但概念上它们是相同的。

```go
func walk(x interface{}, fn func(input string)) {
    val := getValue(x)

    numberOfValues := 0
    var getField func(int) reflect.Value

    switch val.Kind() {
    case reflect.String:
        fn(val.String())
    case reflect.Struct:
        numberOfValues = val.NumField()
        getField = val.Field
    case reflect.Slice:
        numberOfValues = val.Len()
        getField = val.Index
    }

    for i:=0; i< numberOfValues; i++ {
        walk(getField(i).Interface(), fn)
    }
}
```

如果 `value` 是一个 `reflect.String`，我们就像平常一样调用 `fn`。

否则，我们的 `switch` 将根据类型提取两个东西

- 他们有多少个字段
- 如何提取 `Value` (`Field` 还是 `Index`)

一旦我们确定了这些东西，我们可以通过 `numberOfValues` 调用 `walk` 与 `getField` 函数的结果。

现在我们已经完成了这个，处理数组应该是很简单的。

## Write the test first

添加 case

```go
{
    "Arrays",
    [2]Profile {
        {33, "London"},
        {34, "Reykjavík"},
    },
    []string{"London", "Reykjavík"},
},
```

## Try to run the test

```
=== RUN   TestWalk/Arrays
    --- FAIL: TestWalk/Arrays (0.00s)
        reflection_test.go:78: got [], want [London Reykjavík]
```

## Write enough code to make it pass

可以用与片相同的方式处理数组，所以只需用逗号将它添加到 case 中

```go
func walk(x interface{}, fn func(input string)) {
    val := getValue(x)

    numberOfValues := 0
    var getField func(int) reflect.Value

    switch val.Kind() {
    case reflect.String:
        fn(val.String())
    case reflect.Struct:
        numberOfValues = val.NumField()
        getField = val.Field
    case reflect.Slice, reflect.Array:
        numberOfValues = val.Len()
        getField = val.Index
    }

    for i:=0; i< numberOfValues; i++ {
        walk(getField(i).Interface(), fn)
    }
}
```

下一个我们向处理的类型是 `map`。

## Write the test first

```go
{
    "Maps",
    map[string]string{
        "Foo": "Bar",
        "Baz": "Boz",
    },
    []string{"Bar", "Boz"},
},
```

## Try to run the test

```
=== RUN   TestWalk/Maps
    --- FAIL: TestWalk/Maps (0.00s)
        reflection_test.go:86: got [], want [Bar Boz]
```

## Write enough code to make it pass

再一次，如果你稍微抽象地思考一下，你会发现 `map` 和 `struct` 非常相似，只是键在编译时是未知的。

```go
func walk(x interface{}, fn func(input string)) {
    val := getValue(x)

    numberOfValues := 0
    var getField func(int) reflect.Value

    switch val.Kind() {
    case reflect.String:
        fn(val.String())
    case reflect.Struct:
        numberOfValues = val.NumField()
        getField = val.Field
    case reflect.Slice, reflect.Array:
        numberOfValues = val.Len()
        getField = val.Index
    case reflect.Map:
        for _, key := range val.MapKeys() {
            walk(val.MapIndex(key).Interface(), fn)
        }
    }

    for i:=0; i< numberOfValues; i++ {
        walk(getField(i).Interface(), fn)
    }
}
```

但是，按照设计，不能通过索引从映射中获取值。它只能通过 _key_ 完成，这就打破了我们的抽象，该死。

## Refactor

你现在感觉如何?这在当时可能是一个很好的抽象，但现在代码感觉有点不稳定。

重构是一个过程，有时我们会犯错误。TDD 的一个主要特点是它让我们能够自由地尝试这些东西。

采取以试验为后盾的小步骤，这绝不是一种不可逆转的局面。让我们回到重构之前的状态。

```go
func walk(x interface{}, fn func(input string)) {
    val := getValue(x)

    walkValue := func(value reflect.Value) {
        walk(value.Interface(), fn)
    }

    switch val.Kind() {
    case reflect.String:
        fn(val.String())
    case reflect.Struct:
        for i := 0; i< val.NumField(); i++ {
            walkValue(val.Field(i))
        }
    case reflect.Slice, reflect.Array:
        for i:= 0; i<val.Len(); i++ {
            walkValue(val.Index(i))
        }
    case reflect.Map:
        for _, key := range val.MapKeys() {
            walkValue(val.MapIndex(key))
        }
    }
}
```

我们已经引入了 `walkValue`，它会在我们的 `switch` 中停止调用 `walk`，这样它们只需要提取从 `val` 提取 `reflect.Value`。

### One final problem

记住，golang 中的 map 并不保证顺序。所以您的测试有时会失败，因为我们断言调用 `fn` 是按照特定的顺序完成的。

要解决这个问题，我们需要将带有映射的断言移动到一个新的测试中，在这个测试中我们不关心顺序。

```go
t.Run("with maps", func(t *testing.T) {
    aMap := map[string]string{
        "Foo": "Bar",
        "Baz": "Boz",
    }

    var got []string
    walk(aMap, func(input string) {
        got = append(got, input)
    })

    assertContains(t, got, "Bar")
    assertContains(t, got, "Boz")
})
```

`assertContains` 定义如下：

```go
func assertContains(t testing.TB, haystack []string, needle string)  {
    t.Helper()
    contains := false
    for _, x := range haystack {
        if x == needle {
            contains = true
        }
    }
    if !contains {
        t.Errorf("expected %+v to contain %q but it didn't", haystack, needle)
    }
}
```

我们下一个想处理的类型是 `chan`。

## Write the test first

```go
t.Run("with channels", func(t *testing.T) {
		aChannel := make(chan Profile)

		go func() {
			aChannel <- Profile{33, "Berlin"}
			aChannel <- Profile{34, "Katowice"}
			close(aChannel)
		}()

		var got []string
		want := []string{"Berlin", "Katowice"}

		walk(aChannel, func(input string) {
			got = append(got, input)
		})

		if !reflect.DeepEqual(got, want) {
			t.Errorf("got %v, want %v", got, want)
		}
	})
```

## Try to run the test

```
--- FAIL: TestWalk (0.00s)
    --- FAIL: TestWalk/with_channels (0.00s)
        reflection_test.go:115: got [], want [Berlin Katowice]
```

## Write enough code to make it pass

我们可以遍历所有通过 channel 发送的值，直到它被 `Recv()` 关闭。



```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	walkValue := func(value reflect.Value) {
		walk(value.Interface(), fn)
	}

	switch val.Kind() {
	case reflect.String:
		fn(val.String())
	case reflect.Struct:
		for i := 0; i < val.NumField(); i++ {
			walkValue(val.Field(i))
		}
	case reflect.Slice, reflect.Array:
		for i := 0; i < val.Len(); i++ {
			walkValue(val.Index(i))
		}
	case reflect.Map:
		for _, key := range val.MapKeys() {
			walkValue(val.MapIndex(key))
		}
	case reflect.Chan:
		for v, ok := val.Recv(); ok; v, ok = val.Recv() {
			walk(v.Interface(), fn)
		}
	}
}
```

接下来要处理的是类型是 `func`。

## Write the test first

```go
t.Run("with function", func(t *testing.T) {
		aFunction := func() (Profile, Profile) {
			return Profile{33, "Berlin"}, Profile{34, "Katowice"}
		}

		var got []string
		want := []string{"Berlin", "Katowice"}

		walk(aFunction, func(input string) {
			got = append(got, input)
		})

		if !reflect.DeepEqual(got, want) {
			t.Errorf("got %v, want %v", got, want)
		}
	})
```

## Try to run the test

```
--- FAIL: TestWalk (0.00s)
    --- FAIL: TestWalk/with_function (0.00s)
        reflection_test.go:132: got [], want [Berlin Katowice]
```

## Write enough code to make it pass

在这种情况下，非零参数函数似乎没有多大意义。但是我们应该允许任意的返回值。

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	walkValue := func(value reflect.Value) {
		walk(value.Interface(), fn)
	}

	switch val.Kind() {
	case reflect.String:
		fn(val.String())
	case reflect.Struct:
		for i := 0; i < val.NumField(); i++ {
			walkValue(val.Field(i))
		}
	case reflect.Slice, reflect.Array:
		for i := 0; i < val.Len(); i++ {
			walkValue(val.Index(i))
		}
	case reflect.Map:
		for _, key := range val.MapKeys() {
			walkValue(val.MapIndex(key))
		}
	case reflect.Chan:
		for v, ok := val.Recv(); ok; v, ok = val.Recv() {
			walk(v.Interface(), fn)
		}
	case reflect.Func:
		valFnResult := val.Call(nil)
		for _, res := range valFnResult {
			walk(res.Interface(), fn)
		}
	}
}
```

## Wrapping up

- 介绍了一些来自 `reflect` 包的东西
- 使用递归遍历任意数据结构
- 做了一个事后看来很糟糕的重构，但并没有为此感到太沮丧。通过迭代地使用测试，这并不是什么大问题。
- 这只是反射的一个小方面。[Go 博客有一篇精彩的文章，内容涉及更多细节](https://blog.golang.org/laws-of-reflection)。
- 既然您已经了解了反射，就尽量避免使用它。
