# Pointers & errors

**[You can find all the code for this chapter here](https://github.com/quii/learn-go-with-tests/tree/main/pointers)**

我们在上一节中学习了结构体，它让我们捕获与概念相关的许多值。

在某些时候，您可能希望使用结构来管理状态，公开方法以允许用户以您可以控制的方式更改状态。


**金融科技喜欢 GO**和比特币？所以，让我们展示一下我们可以打造一个多么神奇的银行系统。

创建一个 `Wallet` 结构，它能让我们存 `Bitcoin`。

## Write the test first

```go
func TestWallet(t *testing.T) {

    wallet := Wallet{}

    wallet.Deposit(10)

    got := wallet.Balance()
    want := 10

    if got != want {
        t.Errorf("got %d want %d", got, want)
    }
}
```

在 [上个例子中](./structs-methods-and-interfaces.md) 我们直接用字段名访问字段，然而在我们非常安全的钱包里，我们不想把自己的内部状态暴露给外界。我们想通过方法来控制访问。

## Try to run the test

`./wallet_test.go:7:12: undefined: Wallet`

## Write the minimal amount of code for the test to run and check the failing test output

编辑器不知道 `Wallet`，现在让我们来创建它。

```go
type Wallet struct { }
```

再次运行

```go
./wallet_test.go:9:8: wallet.Deposit undefined (type Wallet has no field or method Deposit)
./wallet_test.go:11:15: wallet.Balance undefined (type Wallet has no field or method Balance)
```

我们需要定义一些方法。

记住，只做足以使测试运行的事情。我们需要确保我们的测试正确失败，并带有明确的错误消息。

```go
func (w Wallet) Deposit(amount int) {

}

func (w Wallet) Balance() int {
    return 0
}
```

如果你不熟悉这些语法，回头去看看 struct 章节。

测试现在应该能编译通过了。

`wallet_test.go:15: got 0 want 10`

## Write enough code to make it pass

我们需要在结构体中使用某种 _balance_ 变量来存储状态

```go
type Wallet struct {
    balance int
}
```

在 Go 中，如果一个符号（比如变量、类型、函数）以小写符号开始，那么它就是 private _outside 包，它被定义为 in_。

在我们的例子中，我们希望我们的方法能够操作这个值，而不是其他人。

记住，我们可以使用“receiver”变量在结构体中访问内部的 `balance` 字段。


```go
func (w Wallet) Deposit(amount int) {
    w.balance += amount
}

func (w Wallet) Balance() int {
    return w.balance
}
```

随着我们在金融科技领域的职业生涯的稳定，运行我们的测试，并享受通过测试的乐趣

`wallet_test.go:15: got 0 want 10`

### ????

这很让人困惑，我们的代码看起来应该可以工作，我们将新金额添加到 balance 中然后 balance 方法应该返回它的当前状态。

在Go中，当你调用一个函数或方法时，参数被复制。

当调用 `func (w Wallet) Deposit(amount int)` 时，`w` 是我们调用该方法的对象的副本。

不用太过计算机科学，当你创造一个值时 —— 就像钱包一样，它被存储在内存中的某个地方。你可以通过 `&myVal` 找到内存中的 _address_。

尝试在代码中添加一些打印

```go
func TestWallet(t *testing.T) {

    wallet := Wallet{}

    wallet.Deposit(10)

    got := wallet.Balance()

    fmt.Printf("address of balance in test is %v \n", &wallet.balance)

    want := 10

    if got != want {
        t.Errorf("got %d want %d", got, want)
    }
}
```

```go
func (w Wallet) Deposit(amount int) {
    fmt.Printf("address of balance in Deposit is %v \n", &w.balance)
    w.balance += amount
}
```

`\n` 转义字符，在输出内存地址后打印新行。我们得到一个指向地址为symbol的对象的指针; `&`。

重新运行测试

```text
address of balance in Deposit is 0xc420012268
address of balance in test is 0xc420012260
```
  
你可以看到两个 balance 的地址不同，当我们在代码中改变余额的值时，
我们正在测试结果的副本中工作。因此测试中的 balance 是不变的。

我们可以使用 _pointers_ 修复这个问题。[Pointers](https://gobyexample.com/pointers) 让我们指向一些值，然后让我们改变它们。
因此，与其使用钱包的副本，不如使用指向钱包的指针，这样我们就可以更改它。                                                                  


```go
func (w *Wallet) Deposit(amount int) {
    w.balance += amount
}

func (w *Wallet) Balance() int {
    return w.balance
}
```

不同之处在于 receiver type 是 `*Wallet` 而不再是 `Wallet`，你可以将这看成是 “指向 wallet 的指针”。

再次运行测试，现在应该能通过了。

现在你可能会想，为什么他们通过了?我们没有像这样在函数中解引用指针:

```go
func (w *Wallet) Balance() int {
    return (*w).balance
}
```

而且似乎是直接指向对象的。事实上，上面使用 `(*w)`的代码是绝对有效的。
然而，Go 的制造者认为这种符号很麻烦，所以语言允许我们写 `w.balance`，
这些指向结构体的指针甚至有自己的名字:_struct pointers_，而且它们 [automatically dereferenced](https://golang.org/ref/spec#Method_values)。

从技术上讲，你不需要使用指针接收器作为 balance 的副本修改 `Balance`。
但是，按照约定，您应该保持方法接收方类型相同以保持一致性。


## Refactor

我们说过我们在做一个比特币钱包，但到目前为止我们还没有提到它们。
我们一直在使用 `int`，因为它们是计数的好类型!

为它创建一个 `struct` 似乎有点小题大做。`int` 在它的工作方式上是好的，但它不是描述性的。

Go 允许您从现有类型创建新的类型。

语法是 `type MyName OriginalType`

```go
type Bitcoin int

type Wallet struct {
    balance Bitcoin
}

func (w *Wallet) Deposit(amount Bitcoin) {
    w.balance += amount
}

func (w *Wallet) Balance() Bitcoin {
    return w.balance
}
```

```go
func TestWallet(t *testing.T) {

    wallet := Wallet{}

    wallet.Deposit(Bitcoin(10))

    got := wallet.Balance()

    want := Bitcoin(10)

    if got != want {
        t.Errorf("got %d want %d", got, want)
    }
}
```

要 make `Bitcoin`，你只需使用 `Bitcoin(999)` 的语法。


通过这样做，我们创建了一个新类型，我们可以在它们上声明 _methods_。
当您想要在现有类型上添加一些特定领域的功能时，这是非常有用的。

让我们在 Bitcoin 上实现 [Stringer](https://golang.org/pkg/fmt/#Stringer)

```go
type Stringer interface {
        String() string
}
```

这个接口定义在 `fmt` 包中。允许您定义在打印中与 `%s` 格式字符串一起使用时如何打印类型。

```go
func (b Bitcoin) String() string {
    return fmt.Sprintf("%d BTC", b)
}
```

如您所见，在类型别名上创建方法的语法与在结构上相同。

接下来，我们需要更新我们的测试格式字符串，以便它们将使用 `String()` 代替。

```go
    if got != want {
        t.Errorf("got %s want %s", got, want)
    }
```

为了看到它的实际作用，故意破坏测试以便我们能看到它

`wallet_test.go:18: got 10 BTC want 20 BTC`

这让我们更清楚地知道我们的测试。

下一个要求是 `Withdraw` 功能。

## Write the test first

几乎和 `Deposit()` 相反

```go
func TestWallet(t *testing.T) {

    t.Run("Deposit", func(t *testing.T) {
        wallet := Wallet{}

        wallet.Deposit(Bitcoin(10))

        got := wallet.Balance()

        want := Bitcoin(10)

        if got != want {
            t.Errorf("got %s want %s", got, want)
        }
    })

    t.Run("Withdraw", func(t *testing.T) {
        wallet := Wallet{balance: Bitcoin(20)}

        wallet.Withdraw(Bitcoin(10))

        got := wallet.Balance()

        want := Bitcoin(10)

        if got != want {
            t.Errorf("got %s want %s", got, want)
        }
    })
}
```

## Try to run the test

`./wallet_test.go:26:9: wallet.Withdraw undefined (type Wallet has no field or method Withdraw)`

## Write the minimal amount of code for the test to run and check the failing test output

```go
func (w *Wallet) Withdraw(amount Bitcoin) {

}
```

`wallet_test.go:33: got 20 BTC want 10 BTC`

## Write enough code to make it pass

```go
func (w *Wallet) Withdraw(amount Bitcoin) {
    w.balance -= amount
}
```

## Refactor

我们的测试有一些重复代码，现在重构下。

```go
func TestWallet(t *testing.T) {

    assertBalance := func(t testing.TB, wallet Wallet, want Bitcoin) {
        t.Helper()
        got := wallet.Balance()

        if got != want {
            t.Errorf("got %s want %s", got, want)
        }
    }

    t.Run("Deposit", func(t *testing.T) {
        wallet := Wallet{}
        wallet.Deposit(Bitcoin(10))
        assertBalance(t, wallet, Bitcoin(10))
    })

    t.Run("Withdraw", func(t *testing.T) {
        wallet := Wallet{balance: Bitcoin(20)}
        wallet.Withdraw(Bitcoin(10))
        assertBalance(t, wallet, Bitcoin(10))
    })

}
```

如果你试图 `Withdraw` 超过账户剩余的钱，会发生什么?目前，我们的要求是假设没有透支功能。

当使用 `Withdraw` 时，我们如何发出问题信号?

在 Go 中，如果你想指出一个错误，按照惯例，你的函数会返回一个 `err`，让调用者检查并采取行动。

## Write the test first

```go
t.Run("Withdraw insufficient funds", func(t *testing.T) {
    startingBalance := Bitcoin(20)
    wallet := Wallet{startingBalance}
    err := wallet.Withdraw(Bitcoin(100))

    assertBalance(t, wallet, startingBalance)

    if err == nil {
        t.Error("wanted an error but didn't get one")
    }
})
```

如果你试图取出超过你余额的钱，我们希望 `Withdraw` 返回一个错误, 并且余额应该保持不变。

然后我们检查一个返回的错误，如果它是 `nil` 测试失败。

`nil` 在其他编程语言中是 `null` 的同义词。
Errors 可以是 `nil`，因为 `Withdraw` 的返回类型将是 `error`，这是一个接口。
如果您看到一个函数接受参数或返回的值是接口，则它们可以为空。

像 `null`，如果你试图访问一个值是 `nil`，它会抛出一个 **runtime panic**。这是不好的!你应该确保你检查了 nil。

## Try and run the test

`./wallet_test.go:31:25: wallet.Withdraw(Bitcoin(100)) used as value`

措辞可能有点不清楚，但我们之前的意图是 `Withdraw` 只是调用它，它将永远不会返回值。
要进行此编译，我们需要更改它，使其具有返回类型。

## Write the minimal amount of code for the test to run and check the failing test output

```go
func (w *Wallet) Withdraw(amount Bitcoin) error {
    w.balance -= amount
    return nil
}
```
同样，只编写足够的代码来满足编译器的要求是非常重要的。
我们纠正了 `Withdraw` 方法，以返回 `error` 现在我们必须返回 _something_ 所以我们只返回 `nil`。

## Write enough code to make it pass

```go
func (w *Wallet) Withdraw(amount Bitcoin) error {

    if amount > w.balance {
        return errors.New("oh no")
    }

    w.balance -= amount
    return nil
}
```

记得导入 `errors` 包。

`errors.New` creates a new `error` with a message of your choosing.

## Refactor


让我们为错误检查创建一个快速测试助手，以帮助我们的测试更清晰

```go
assertError := func(t testing.TB, err error) {
    t.Helper()
    if err == nil {
        t.Error("wanted an error but didn't get one")
    }
}
```

And in our test

```go
t.Run("Withdraw insufficient funds", func(t *testing.T) {
    startingBalance := Bitcoin(20)
    wallet := Wallet{startingBalance}
    err := wallet.Withdraw(Bitcoin(100))

    assertBalance(t, wallet, startingBalance)
    assertError(t, err)
})
```

希望当返回一个错误“oh no”时，你在想我们可能会迭代它，因为它似乎没有返回的用处。

假设错误最终返回给用户，让我们更新测试以断言某种错误消息，而不仅仅是错误的存在。

## Write the test first

更新我们的 helper 以对 `string` 进行比较。

```go
assertError := func(t testing.TB, got error, want string) {
    t.Helper()
    if got == nil {
        t.Fatal("didn't get an error but wanted one")
    }

    if got.Error() != want {
        t.Errorf("got %q, want %q", got, want)
    }
}
```

更新调用方

```go
t.Run("Withdraw insufficient funds", func(t *testing.T) {
    startingBalance := Bitcoin(20)
    wallet := Wallet{startingBalance}
    err := wallet.Withdraw(Bitcoin(100))

    assertBalance(t, wallet, startingBalance)
    assertError(t, err, "cannot withdraw, insufficient funds")
})
```

我们引入的 `t.Fatal`，它将在被调用时停止测试。
这是因为如果没有返回的错误，我们不想对返回的错误进行任何断言。
如果没有这个，测试将继续进行到下一个步骤，并因为一个 nil 指针而 panic。


## Try to run the test

`wallet_test.go:61: got err 'oh no' want 'cannot withdraw, insufficient funds'`

## Write enough code to make it pass

```go
func (w *Wallet) Withdraw(amount Bitcoin) error {

    if amount > w.balance {
        return errors.New("cannot withdraw, insufficient funds")
    }

    w.balance -= amount
    return nil
}
```

## Refactor

我们在 `Withdraw` 和测试代码中有 error message 的重复代码。

如果有人想重述错误，测试失败将是非常恼人的，这对我们的测试来说太详细了。
我们并不真正关心确切的措辞是什么，只关心在给定条件下返回一些有意义的关于退出的错误。

在 Go 中，**错误就是值**，所以我们可以把它重构成一个变量，并有一个单一的真理来源。

```go
var ErrInsufficientFunds = errors.New("cannot withdraw, insufficient funds")

func (w *Wallet) Withdraw(amount Bitcoin) error {

    if amount > w.balance {
        return ErrInsufficientFunds
    }

    w.balance -= amount
    return nil
}
```

`var` 关键字能让我们定义包的全局变量。

这本身就是一个积极的变化，因为现在我们的 `Withdraw` 功能看起来非常清晰。

接下来，我们可以重构测试代码，使用这个值而不是特定的字符串。


```go
func TestWallet(t *testing.T) {

    t.Run("Deposit", func(t *testing.T) {
        wallet := Wallet{}
        wallet.Deposit(Bitcoin(10))
        assertBalance(t, wallet, Bitcoin(10))
    })

    t.Run("Withdraw with funds", func(t *testing.T) {
        wallet := Wallet{Bitcoin(20)}
        wallet.Withdraw(Bitcoin(10))
        assertBalance(t, wallet, Bitcoin(10))
    })

    t.Run("Withdraw insufficient funds", func(t *testing.T) {
        wallet := Wallet{Bitcoin(20)}
        err := wallet.Withdraw(Bitcoin(100))

        assertBalance(t, wallet, Bitcoin(20))
        assertError(t, err, ErrInsufficientFunds)
    })
}

func assertBalance(t testing.TB, wallet Wallet, want Bitcoin) {
    t.Helper()
    got := wallet.Balance()

    if got != want {
        t.Errorf("got %q want %q", got, want)
    }
}

func assertError(t testing.TB, got error, want error) {
    t.Helper()
    if got == nil {
        t.Fatal("didn't get an error but wanted one")
    }

    if got != want {
        t.Errorf("got %q, want %q", got, want)
    }
}
```

现在这个测试也更容易遵循了。

我将帮助程序移出了主测试函数，这样当有人打开一个文件时，他们可以首先读取我们的断言，而不是一些帮助程序。

测试的另一个有用的特性是，它帮助我们了解代码的实际使用情况，这样我们就可以编写出符合实际情况的代码。
我们可以在这里看到，开发人员可以简单地调用我们的代码，并对 `ErrInsufficientFunds` 进行等号检查，并相应地采取行动。

### Unchecked errors

虽然 Go 编译器帮了你很多，但有时你还是会错过一些东西，错误处理有时会很棘手。

有一种情况我们没有测试过。要找到它，在终端中运行以下命令安装 `errcheck`，这是 Go 可用的许多检查程序之一。

`go get -u github.com/kisielk/errcheck`

然后在你代码的目录下运行 `errcheck`。

你应该能得到下面的东西

`wallet_test.go:17:18: wallet.Withdraw(Bitcoin(10))`

这告诉我们，我们没有检查在那行代码上返回的错误。
我的计算机上的这行代码对应于我们正常的取款场景，因为我们没有检查如果 err 的情况。

下面是最终的测试代码。

```go
func TestWallet(t *testing.T) {

    t.Run("Deposit", func(t *testing.T) {
        wallet := Wallet{}
        wallet.Deposit(Bitcoin(10))

        assertBalance(t, wallet, Bitcoin(10))
    })

    t.Run("Withdraw with funds", func(t *testing.T) {
        wallet := Wallet{Bitcoin(20)}
        err := wallet.Withdraw(Bitcoin(10))

        assertBalance(t, wallet, Bitcoin(10))
        assertNoError(t, err)
    })

    t.Run("Withdraw insufficient funds", func(t *testing.T) {
        wallet := Wallet{Bitcoin(20)}
        err := wallet.Withdraw(Bitcoin(100))

        assertBalance(t, wallet, Bitcoin(20))
        assertError(t, err, ErrInsufficientFunds)
    })
}

func assertBalance(t testing.TB, wallet Wallet, want Bitcoin) {
    t.Helper()
    got := wallet.Balance()

    if got != want {
        t.Errorf("got %s want %s", got, want)
    }
}

func assertNoError(t testing.TB, got error) {
    t.Helper()
    if got != nil {
        t.Fatal("got an error but didn't want one")
    }
}

func assertError(t testing.TB, got error, want error) {
    t.Helper()
    if got == nil {
        t.Fatal("didn't get an error but wanted one")
    }

    if got != want {
        t.Errorf("got %s, want %s", got, want)
    }
}
```

## Wrapping up

### Pointers

* 当你把它们传递给函数/方法时，Go 会复制值，所以如果你在写一个需要改变状态的函数，你需要它接受一个指向你想改变的东西的指针。
* Go 获取值的副本在很多时候是有用的，但有时你不想让你的系统复制某些东西，在这种情况下，你需要传递一个引用。示例可能是非常大的数据，或者可能是只打算有一个实例的数据(如数据库连接池)。

### nil

* 指针可以是 nil
* 当一个函数返回一个指向某物的指针时，你需要确保你检查它是否是 nil，否则你可能会引发一个运行时异常，编译器在这里不会帮你。
* 当你想要描述一个可能丢失的值时很有用

### Errors

* error 是调用函数/方法时表示失败的方式。
* 通过听我们的测试，我们得出结论，在错误中检查字符串会导致测试不稳定。因此，我们进行了重构，使用了一个有意义的值，这使得代码更容易测试，并得出结论，这对我们 API 的用户来说也更容易。
* 这还不是错误处理的结束，你可以做更复杂的事情，但这只是一个介绍。后面的部分将介绍更多的策略。
* [不要只是检查错误，要优雅地处理它们](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully)

### Create new types from existing ones

* Useful for adding more domain specific meaning to values
* Can let you implement interfaces

指针和错误是编写 Go 的重要部分，你需要熟悉它们。幸运的是，如果你做错了，编译器通常会帮助你，只要花时间阅读错误。
