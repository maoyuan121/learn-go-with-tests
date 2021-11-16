# Maps

**[You can find all the code for this chapter here](https://github.com/quii/learn-go-with-tests/tree/main/maps)**

在 [arrays & slices](arrays-and-slices.md)，你看到了如何按照顺序存储值。现在，我们将看看一种通过 `key` 存储 item 并快速查找它们的方法。

Map 允许您以类似于字典的方式存储项。你可以把 `key` 看作单词，把 `value` 看作定义。还有什么比建立我们自己的字典更好的学习 map 的方法呢?

首先，假设我们已经在字典中有一些单词和它们的定义，如果我们搜索一个单词，它应该返回它的定义。


## Write the test first

In `dictionary_test.go`

```go
package main

import "testing"

func TestSearch(t *testing.T) {
    dictionary := map[string]string{"test": "this is just a test"}

    got := Search(dictionary, "test")
    want := "this is just a test"

    if got != want {
        t.Errorf("got %q want %q given, %q", got, want, "test")
    }
}
```

声明 Map 有点类似于数组。但是，它以 `map` 关键字开头，需要两种类型。第一个是键类型，它写在 `[]` 中。第二个是值类型，它紧跟在 `[]` 之后。

key 类型是特殊的。它只能是一个可比较的类型，因为如果不能判断两个键是否相等，我们就无法确保得到的是正确的值。类似的类型在[语言规范](https://golang.org/ref/spec#Comparison_operators)中有深入的解释。

另一方面，值类型可以是您想要的任何类型。它甚至可以是另一个 map。


## Try to run the test

运行 `go test` 编译失败 `./dictionary_test.go:8:9: undefined: Search`。

## Write the minimal amount of code for the test to run and check the output

In `dictionary.go`

```go
package main

func Search(dictionary map[string]string, word string) string {
    return ""
}
```

你的测试现在应该失败有一个*清晰的错误消息*

`dictionary_test.go:12: got '' want 'this is just a test' given, 'test'`.

## Write enough code to make it pass

```go
func Search(dictionary map[string]string, word string) string {
    return dictionary[word]
}
```

从 Map 中获取值与从 Array 中获取值相同 `map[key]`。

## 重构

```go
func TestSearch(t *testing.T) {
    dictionary := map[string]string{"test": "this is just a test"}

    got := Search(dictionary, "test")
    want := "this is just a test"

    assertStrings(t, got, want)
}

func assertStrings(t testing.TB, got, want string) {
    t.Helper()

    if got != want {
        t.Errorf("got %q want %q", got, want)
    }
}
```

我决定创建一个 `assertStrings` 帮助函数，使实现更通用。

### 使用自定义类型

我们可以通过在 map 之上创建一种新的类型，并将 `Search` 作为一种方法来改进词典的使用。

In `dictionary_test.go`:

```go
func TestSearch(t *testing.T) {
    dictionary := Dictionary{"test": "this is just a test"}

    got := dictionary.Search("test")
    want := "this is just a test"

    assertStrings(t, got, want)
}
```

我们开始使用 `Dictionary` 类型，我们现在还没有定义它。然后调用 `Dictionary` 实例的 `Search`。

In `dictionary.go`:

```go
type Dictionary map[string]string

func (d Dictionary) Search(word string) string {
    return d[word]
}
```

这里我们创建了一个 `Dictinary` 类型，它就像是 `map` 的一个薄薄的包装，通过定义自定义类型，我们创建 `Search` 方法。

## 先写测试

基本搜索很容易实现，但如果我们提供一个不在字典中的单词，会发生什么呢?

我们实际上什么也得不到。这很好，因为程序可以继续运行，但是有更好的方法。该函数可以报告该单词不在字典中。这样，用户就不会怀疑这个词是否不存在，
或者根本就没有定义(这对于字典来说似乎不是很有用。然而，这种场景在其他情况下可能是关键的)。

```go
func TestSearch(t *testing.T) {
    dictionary := Dictionary{"test": "this is just a test"}

    t.Run("known word", func(t *testing.T) {
        got, _ := dictionary.Search("test")
        want := "this is just a test"

        assertStrings(t, got, want)
    })

    t.Run("unknown word", func(t *testing.T) {
        _, err := dictionary.Search("unknown")
        want := "could not find the word you were looking for"

        if err == nil {
            t.Fatal("expected to get an error.")
        }

        assertStrings(t, err.Error(), want)
    })
}
```

在 Go 中处理这个场景的方法是返回第二个参数，它是一个 `Error` 类型。

`Error` 可以通过 `.Error()` 方法转换为字符串，这是我们在将其传递给断言时所做的。我们也用 `if` 来保护 `assertStrings`，以确保我们不会在 `nil` 上调用 `.error()`。

## Try and run the test

编译失败

```
./dictionary_test.go:18:10: assignment mismatch: 2 variables but 1 values
```

## Write the minimal amount of code for the test to run and check the output

```go
func (d Dictionary) Search(word string) (string, error) {
    return d[word], nil
}
```

现在测试应该失败，且有一个清晰的错误消息。

`dictionary_test.go:22: expected to get an error.`

## Write enough code to make it pass

```go
func (d Dictionary) Search(word string) (string, error) {
    definition, ok := d[word]
    if !ok {
        return "", errors.New("could not find the word you were looking for")
    }

    return definition, nil
}
```

为了测试通过，我们使用了 map 查找的一个有趣属性。它可以返回 2 个值。第二个值是一个布尔值，表示是否成功找到了键。

这个属性允许我们区分一个不存在的单词和一个没有定义的单词。

## Refactor

```go
var ErrNotFound = errors.New("could not find the word you were looking for")

func (d Dictionary) Search(word string) (string, error) {
    definition, ok := d[word]
    if !ok {
        return "", ErrNotFound
    }

    return definition, nil
}
```

我们可以通过将 `Search` 函数提取为一个变量来消除这个神奇的错误。这也将使我们有一个更好的测试。

```go
t.Run("unknown word", func(t *testing.T) {
    _, got := dictionary.Search("unknown")

    assertError(t, got, ErrNotFound)
})
}

func assertError(t testing.TB, got, want error) {
    t.Helper()

    if got != want {
        t.Errorf("got error %q want %q", got, want)
    }
}
```

通过创建一个新的帮助函数，我们可以简化测试，并开始使用 `ErrNotFound` 变量，这样在将来更改错误文本时，测试不会失败。

## Write the test first

我们有一个查找字典的好方法。然而，我们没有办法把新单词添加到字典中。

```go
func TestAdd(t *testing.T) {
    dictionary := Dictionary{}
    dictionary.Add("test", "this is just a test")

    want := "this is just a test"
    got, err := dictionary.Search("test")
    if err != nil {
        t.Fatal("should find added word:", err)
    }

    if got != want {
        t.Errorf("got %q want %q", got, want)
    }
}
```

在这个测试中，我们使用了 `Search` 函数，使字典的验证更容易一些。


## Write the minimal amount of code for the test to run and check output

In `dictionary.go`

```go
func (d Dictionary) Add(word, definition string) {
}
```

测试失败

```
dictionary_test.go:31: should find added word: could not find the word you were looking for
```

## Write enough code to make it pass

```go
func (d Dictionary) Add(word, definition string) {
    d[word] = definition
}
```

添加到 map 也类似于添加数组。您只需要指定一个键并将其设置为一个值。

### Pointers, copies, et al

maps 的一个有趣的特性是，你可以修改它们，而不需要传递地址(例如 `&myMap`)

这可能会让他们感觉像是“引用类型”，[但正如戴夫·切尼所描述的](https://dave.cheney.net/2017/04/30/if-a-map-isnt-a-reference-variable-what-is-it)他们不是。

> map 值是指向 runtime.hmap 结构。

因此，当您将 map 传递给一个函数/方法时，您实际上是在复制它，但只是指针部分，而不是包含数据的底层数据结构。

maps 的一个问题是，它们可以是一个 `nil` 值。一个 `nil` map 在读取时会表现得像一个空 map，但试图写入 `nil` map 会导致运行时的 panic。
你可以在这里阅读更多[关于 map 的信息](https://blog.golang.org/go-maps-in-action)。

因此，**永远不要初始化空的 map 变量**:

```go
var m map[string]string
```

相反，你可以像我们上面做的那样初始化一个空的 map，或者使用 `make` 关键字来为你创建一个map:


```go
var dictionary = map[string]string{}

// OR

var dictionary = make(map[string]string)
```

这两种方法都创建一个空的 `hash map`，并在其上指向 `dictionary`。这确保了您不会在运行时出现 panic。


## Refactor

在我们的实现中没有太多需要重构的东西，但是测试可以稍微简化一下。

```go
func TestAdd(t *testing.T) {
    dictionary := Dictionary{}
    word := "test"
    definition := "this is just a test"

    dictionary.Add(word, definition)

    assertDefinition(t, dictionary, word, definition)
}

func assertDefinition(t testing.TB, dictionary Dictionary, word, definition string) {
    t.Helper()

    got, err := dictionary.Search(word)
    if err != nil {
        t.Fatal("should find added word:", err)
    }

    if definition != got {
        t.Errorf("got %q want %q", got, definition)
    }
}
```

我们为 word 和 definition 创建了变量，并将定义断言移动到它自己的 helper 函数中。

我们的 `Add` 看起来不错。但是，我们没有考虑到当我们试图添加的值已经存在时会发生什么!

如果值已经存在，Map 不会抛出错误。相反，它们将继续使用新提供的值覆盖该值。这在实践中很方便，但使函数名不够准确。`Add` 不应该修改现有的值。它只会给我们的词典增加新词。

## Write the test first

```go
func TestAdd(t *testing.T) {
    t.Run("new word", func(t *testing.T) {
        dictionary := Dictionary{}
        word := "test"
        definition := "this is just a test"

        err := dictionary.Add(word, definition)

        assertError(t, err, nil)
        assertDefinition(t, dictionary, word, definition)
    })

    t.Run("existing word", func(t *testing.T) {
        word := "test"
        definition := "this is just a test"
        dictionary := Dictionary{word: definition}
        err := dictionary.Add(word, "new test")

        assertError(t, err, ErrWordExists)
        assertDefinition(t, dictionary, word, definition)
    })
}
...
func assertError(t testing.TB, got, want error) {
	t.Helper()
	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}
```

对于这个测试，我们修改了 `Add` 以返回一个错误，我们用一个新的错误变量 `ErrWordExists` 来验证这个错误。我们还修改了前面的测试，以检查 `nil` 错误，以及 `assertError` 函数。


## Try to run test

编译器将失败，因为我们没有返回 `Add` 的值。

```
./dictionary_test.go:30:13: dictionary.Add(word, definition) used as value
./dictionary_test.go:41:13: dictionary.Add(word, "new test") used as value
```

## 为要运行的测试编写最小数量的代码，并检查失败的测试输出

In `dictionary.go`

```go
var (
    ErrNotFound   = errors.New("could not find the word you were looking for")
    ErrWordExists = errors.New("cannot add word because it already exists")
)

func (d Dictionary) Add(word, definition string) error {
    d[word] = definition
    return nil
}
```

现在我们得到了两个错误。我们任然修改了值，返回了一个 `nil` 错误。

```
dictionary_test.go:43: got error '%!q(<nil>)' want 'cannot add word because it already exists'
dictionary_test.go:44: got 'new test' want 'this is just a test'
```

## Write enough code to make it pass

```go
func (d Dictionary) Add(word, definition string) error {
    _, err := d.Search(word)

    switch err {
    case ErrNotFound:
        d[word] = definition
    case nil:
        return ErrWordExists
    default:
        return err
    }

    return nil
}
```

这里我们使用 `switch` 语句来匹配错误。有一个像这样的 `switch` 提供了一个额外的安全网，以防 `Search` 返回一个错误而不是 `ErrNotFound`。

## Refactor

我们没有太多需要重构的东西，但是随着错误使用的增加，我们可以做一些修改。

```go
const (
    ErrNotFound   = DictionaryErr("could not find the word you were looking for")
    ErrWordExists = DictionaryErr("cannot add word because it already exists")
)

type DictionaryErr string

func (e DictionaryErr) Error() string {
    return string(e)
}
```

我们创建错误常量;这要求我们创建自己的 `DictionaryErr` 类型，它实现 `error`接口。
你可以在[戴夫·切尼的这篇优秀文章](https://dave.cheney.net/2016/04/07/constant-errors)中阅读更多细节。简单地说，它使错误更可重用和不可变。

接下来我们创建一个函数 `Update` 更新 word 的定义。

## Write the test first

```go
func TestUpdate(t *testing.T) {
    word := "test"
    definition := "this is just a test"
    dictionary := Dictionary{word: definition}
    newDefinition := "new definition"

    dictionary.Update(word, newDefinition)

    assertDefinition(t, dictionary, word, newDefinition)
}
```

`Update` 非常类似于 `Add`，我们将在下面实现。

## Try and run the test

```
./dictionary_test.go:53:2: dictionary.Update undefined (type Dictionary has no field or method Update)
```

## 为要运行的测试编写最小数量的代码，并检查失败的测试输出

我们已经知道如何处理这样的错误。我们需要定义函数。

```go
func (d Dictionary) Update(word, definition string) {}
```

有了这些，我们就能看到我们需要改变这个词的定义。

```
dictionary_test.go:55: got 'this is just a test' want 'new definition'
```

## 编写足够的代码让测试通过

我们已经看到了如何在修复 `Add` 问题时做到这一点。让我们实现一些类似于 `Add` 的东西。


```go
func (d Dictionary) Update(word, definition string) {
    d[word] = definition
}
```

我们不需要对它进行重构，因为它是一个简单的更改。然而，我们现在遇到了与 `Add` 相同的问题。如果我们传入一个新单词，`Update` 将把它添加到字典中。


## 先写测试

```go
t.Run("existing word", func(t *testing.T) {
    word := "test"
    definition := "this is just a test"
    newDefinition := "new definition"
    dictionary := Dictionary{word: definition}

    err := dictionary.Update(word, newDefinition)

    assertError(t, err, nil)
    assertDefinition(t, dictionary, word, newDefinition)
})

t.Run("new word", func(t *testing.T) {
    word := "test"
    definition := "this is just a test"
    dictionary := Dictionary{}

    err := dictionary.Update(word, definition)

    assertError(t, err, ErrWordDoesNotExist)
})
```

我们添加了另一种错误类型，用于当单词不存在时。我们还修改了 `Update` 以返回 `error` 值。

## 运行测试

```
./dictionary_test.go:53:16: dictionary.Update(word, "new test") used as value
./dictionary_test.go:64:16: dictionary.Update(word, definition) used as value
./dictionary_test.go:66:23: undefined: ErrWordDoesNotExist
```

这次我们有 3 个错误，但是我们知道如何去处理它。

## 为要运行的测试编写最小数量的代码，并检查失败的测试输出

```go
const (
    ErrNotFound         = DictionaryErr("could not find the word you were looking for")
    ErrWordExists       = DictionaryErr("cannot add word because it already exists")
    ErrWordDoesNotExist = DictionaryErr("cannot update word because it does not exist")
)

func (d Dictionary) Update(word, definition string) error {
    d[word] = definition
    return nil
}
```

我们添加了我们的错误类型，返回了一个 `nil` 错误。

现在我们只剩下一个错误了。

```
dictionary_test.go:66: got error '%!q(<nil>)' want 'cannot update word because it does not exist'
```

## 编写足够的代码让测试通过

```go
func (d Dictionary) Update(word, definition string) error {
    _, err := d.Search(word)

    switch err {
    case ErrNotFound:
        return ErrWordDoesNotExist
    case nil:
        d[word] = definition
    default:
        return err
    }

    return nil
}
```

这个函数看起来几乎与 `Add` 相同，除了我们在更新 `dictionary` 和返回错误时进行了切换。

### 注意为 Update 声明一个新错误

我们可以重用 `ErrNotFound` 而不添加新的错误。但是，最好在更新失败时给出一个精确的错误。

有了特定的错误，您就可以获得关于哪里出错的更多信息。以下是一个 web 应用程序中的例子:

> 当遇到 `ErrNotFound` 时，可以重定向用户，但当遇到 `ErrWordDoesNotExist` 时显示错误消息。
 
接下来，创建一个函数 `Delete` 从字典中删除。

## 先写测试

```go
func TestDelete(t *testing.T) {
    word := "test"
    dictionary := Dictionary{word: "test definition"}

    dictionary.Delete(word)

    _, err := dictionary.Search(word)
    if err != ErrNotFound {
        t.Errorf("Expected %q to be deleted", word)
    }
}
```

我们的测试创建一个带有单词的 `Dictionary`，然后检查这个单词是否已经被删除。

## 运行测试

By running `go test` we get:

```
./dictionary_test.go:74:6: dictionary.Delete undefined (type Dictionary has no field or method Delete)
```

## 为要运行的测试编写最小数量的代码，并检查失败的测试输出

```go
func (d Dictionary) Delete(word string) {

}
```

添加了后，测试告诉我们没有删除字。

```
dictionary_test.go:78: Expected 'test' to be deleted
```

## 写足够的代码让测试通过

```go
func (d Dictionary) Delete(word string) {
    delete(d, word)
}
```

Go 有一个内置的 `delete` 函数，可以在 map 上使用。它有两个参数。第一个是 map，第二个是要删除的 key。

`delete` 函数什么也不返回，我们的 `delete` 方法也是基于同样的概念。因为删除一个不存在的值是没有效果的，不像我们的 `Update` 和 `Add` 方法，我们不需要用错误使 API 复杂化。

## 总结

在本节中，我们讨论了很多内容。我们为字典创建了一个完整的 CRUD(创建、读取、更新和删除)API。在整个过程中，我们学会了如何:


* 创建 map
* 从 map 中搜索 item
* 添加一个 item 到 map 中
* 在 map 中更新 item
* 从 map 中删除 item
* 学习了关于 error 的知识
  * How to create errors that are constants
  * Writing error wrappers
