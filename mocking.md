# Mocking

**[You can find all the code for this chapter here](https://github.com/quii/learn-go-with-tests/tree/main/mocking)**

你被要求写一个程序从 3 开始倒数，每一个数字在一个新的行（有 1 秒钟的暂停），当到 0 的时候会打印出  "Go!" 并退出。

```
3
2
1
Go!
```

我们将通过编写一个名为 `Countdown` 的函数来解决这个问题，然后将其放入 `main` 程序中，使其看起来像这样：

```go
package main

func main() {
    Countdown()
}
```

虽然这是一个非常简单的程序，但要完全测试它，我们需要像往常一样采用 _iterative_， _test-driven_ 方法。

我说的迭代是什么意思？我们确保我们采取最小的步骤，我们可以得到有用的软件。

我们不想花很长时间编写那些理论上可以在黑客攻击后正常工作的代码，因为这通常是开发人员掉进兔子洞的原因。
这是一个重要的技能，能够分割需求尽可能小，这样你就可以有  _working software_

以下是我们如何划分工作并进行迭代:

- Print 3
- Print 3, 2, 1 and Go!
- Wait a second between each line

## Write the test first

我们的软件需要打印到标准输出 stdout，我们在 DI 部分看到了如何使用 DI 来促进测试。

```go
func TestCountdown(t *testing.T) {
    buffer := &bytes.Buffer{}

    Countdown(buffer)

    got := buffer.String()
    want := "3"

    if got != want {
        t.Errorf("got %q want %q", got, want)
    }
}
```

如果你不熟悉 `buffer`， 可以重读 [the previous section](dependency-injection.md)。

我们知道我们向让我们的 `Countdown` 函数在某处写一些数据，`io.Writer` 实际上是在 go 中捕获它作为界面的方法。
                                   
- 在 `main` 中我们将发送到 `os.Stdout`，以便我们的用户看到倒计时打印到终端。
- 在测试中我们将发送到 `bytes.Buffer`，这样我们的测试就能捕捉到什么生成了什么数据。

## Try and run the test

`./countdown_test.go:11:2: undefined: Countdown`

## Write the minimal amount of code for the test to run and check the failing test output

定义 `Countdown`

```go
func Countdown() {}
```

再试

```go
./countdown_test.go:11:11: too many arguments in call to Countdown
    have (*bytes.Buffer)
    want ()
```

编译器告诉你函数的签名应该是什么样的，因此更新。

```go
func Countdown(out *bytes.Buffer) {}
```

`countdown_test.go:17: got '' want '3'`

Perfect!

## Write enough code to make it pass

```go
func Countdown(out *bytes.Buffer) {
    fmt.Fprint(out, "3")
}
```

我们使用 `fmt.Fprint`，它接收一个 `io.Writer`（如 `*bytes.Buffer`）传递一个 `string` 给它。测试应该能通过了。

## Refactor

我们知道 `*bytes.Buffer` 能正常工作了，使用通用接口会更好。

```go
func Countdown(out io.Writer) {
    fmt.Fprint(out, "3")
}
```

重新运行测试，应该也能通过。

为了完成任务，现在让我们将函数连接到一个 `main` 中，这样我们就有了一些可以工作的软件来确保我们正在取得进展。


```go
package main

import (
    "fmt"
    "io"
    "os"
)

func Countdown(out io.Writer) {
    fmt.Fprint(out, "3")
}

func main() {
    Countdown(os.Stdout)
}
```

尝试运行这个程序，并为您的手工工作感到惊奇。

是的，这似乎是微不足道的，我为任何项目都推荐这种方法。**取一小部分功能，让它端到端工作，并在测试的支持下。**

接下来我们让它打印出 2，1 然后是"Go!"。

## Write the test first

通过投资使整个管道正常工作，我们可以安全地轻松地迭代我们的解决方案。
我们将不再需要停止并重新运行程序，以确信它能工作，因为所有的逻辑都已经过测试。

```go
func TestCountdown(t *testing.T) {
    buffer := &bytes.Buffer{}

    Countdown(buffer)

    got := buffer.String()
    want := `3
2
1
Go!`

    if got != want {
        t.Errorf("got %q want %q", got, want)
    }
}
```

backtick 语法是另一种创建 `string` 的方法，但让你把像换行符这样的东西，这对我们的测试来说是完美的。


## Try and run the test

```
countdown_test.go:21: got '3' want '3
        2
        1
        Go!'
```
## Write enough code to make it pass

```go
func Countdown(out io.Writer) {
    for i := 3; i > 0; i-- {
        fmt.Fprintln(out, i)
    }
    fmt.Fprint(out, "Go!")
}
```

Use a `for` loop counting backwards with `i--` and use `fmt.Fprintln` to print to `out` with our number followed by a newline character. 
Finally use `fmt.Fprint` to send "Go!" aftward.

## Refactor

除了将一些神奇的值重构为命名常量之外，没有什么需要重构的。

```go
const finalWord = "Go!"
const countdownStart = 3

func Countdown(out io.Writer) {
    for i := countdownStart; i > 0; i-- {
        fmt.Fprintln(out, i)
    }
    fmt.Fprint(out, finalWord)
}
```

如果您现在运行程序，您应该得到所需的输出，但我们没有将其作为一个引人注目的倒计时 1 秒暂停。

Go让你用 `time.Sleep` 来实现这个目标。试着把它添加到我们的代码中。

```go
func Countdown(out io.Writer) {
    for i := countdownStart; i > 0; i-- {
        time.Sleep(1 * time.Second)
        fmt.Fprintln(out, i)
    }

    time.Sleep(1 * time.Second)
    fmt.Fprint(out, finalWord)
}
```

If you run the program it works as we want it to.

## Mocking

测试能通过了，软件也按照预想一样运行了，但是我们还有些问题：
- 我们的测试花了 4 秒钟来运行。
    - 每一篇关于软件开发的前瞻性文章都强调快速反馈循环的重要性。
    - **缓慢的测试会破坏开发人员的生产力**。
    - 想象一下，如果需求变得更加复杂，需要进行更多的测试。我们对每一次“倒计时”的新测试都添加4s试运行感到满意吗?
- 我们还没有测试函数的一个重要性质。

我们对 `Sleep` 有依赖性，我们需要提取它，这样我们就可以在测试中控制它。

如果我们要 mock `time.Sleep` 我们可以使用 _dependency injection_ 来代替真正的 `time.Sleep`，然后我们可以“监视调用”，对他们做出断言。


## Write the test first

让我们将依赖定义为一个接口。这让我们可以在 `main` 中使用 _real_ Sleeper，在测试中使用 _spy Sleeper_。
通过使用一个接口，我们的 `Countdown` 函数是不在意的，并为调用者增加了一些灵活性。
                                                                         
```go
type Sleeper interface {
    Sleep()
}
```

我做了一个设计决定，我们的 `Countdown` 函数将不负责多长时间的睡眠。
至少到目前为止，这稍微简化了我们的代码，这意味着我们函数的用户可以按照他们喜欢的方式配置 sleep。

现在我们需要创建一个 _mock_ 以供测试使用。

```go
type SpySleeper struct {
    Calls int
}

func (s *SpySleeper) Sleep() {
    s.Calls++
}
```

_Spies_ 是一种 _mock_，它可以记录依赖是如何使用的。
他们可以记录发送的参数，调用的次数，等等。
在我们的例子中，我们跟踪 `Sleep()` 被调用的次数，这样我们就可以在测试中检查它。

更新测试，注入对我们的间谍的依赖，并断言睡眠已被调用 4 次。


```go
func TestCountdown(t *testing.T) {
    buffer := &bytes.Buffer{}
    spySleeper := &SpySleeper{}

    Countdown(buffer, spySleeper)

    got := buffer.String()
    want := `3
2
1
Go!`

    if got != want {
        t.Errorf("got %q want %q", got, want)
    }

    if spySleeper.Calls != 4 {
        t.Errorf("not enough calls to sleeper, want 4 got %d", spySleeper.Calls)
    }
}
```

## Try and run the test

```
too many arguments in call to Countdown
    have (*bytes.Buffer, *SpySleeper)
    want (io.Writer)
```

## Write the minimal amount of code for the test to run and check the failing test output

我们需要更新 `Countdown` 接收一个 `Sleeper` 参数

```go
func Countdown(out io.Writer, sleeper Sleeper) {
    for i := countdownStart; i > 0; i-- {
        time.Sleep(1 * time.Second)
        fmt.Fprintln(out, i)
    }

    time.Sleep(1 * time.Second)
    fmt.Fprint(out, finalWord)
}
```

如果您再次尝试，你的 `main` 将不再编译的原因相同


```
./main.go:26:11: not enough arguments in call to Countdown
    have (*os.File)
    want (io.Writer, Sleeper)
```

让我们创建一个真正的 sleeper， 它实现了我们需要的接口

```go
type DefaultSleeper struct {}

func (d *DefaultSleeper) Sleep() {
    time.Sleep(1 * time.Second)
}
```

我们可以在真正的应用中如下一样使用

```go
func main() {
    sleeper := &DefaultSleeper{}
    Countdown(os.Stdout, sleeper)
}
```

## Write enough code to make it pass

现在测试可以编译通过了，但是没有通过，我们任然调用了 `time.Sleep` 还不是用的依赖注入的。现在修复。

```go
func Countdown(out io.Writer, sleeper Sleeper) {
    for i := countdownStart; i > 0; i-- {
        sleeper.Sleep()
        fmt.Fprintln(out, i)
    }

    sleeper.Sleep()
    fmt.Fprint(out, finalWord)
}
```

测试现在应该能通过，并且不再需要花费 4 秒钟了。

### Still some problems

还有个重要的属性我们没有测试。

`Countdown` 应该在每次打印前睡一下，例如：

- `Sleep`
- `Print N`
- `Sleep`
- `Print N-1`
- `Sleep`
- `Print Go!`
- etc

我们最新的更改只断言它已经休眠了 4 次，但是这些休眠可能会不按顺序发生。

在编写测试时，如果您不相信测试给了您足够的信心，那么就打破它!
(确保您已经首先向源代码控制提交了更改)。将代码更改为以下内容

```go
func Countdown(out io.Writer, sleeper Sleeper) {
    for i := countdownStart; i > 0; i-- {
        sleeper.Sleep()
    }

    for i := countdownStart; i > 0; i-- {
        fmt.Fprintln(out, i)
    }

    sleeper.Sleep()
    fmt.Fprint(out, finalWord)
}
```

如果您运行您的测试，它们仍然应该通过，即使实现是错误的。

让我们再用一种新的测试来检查行动的顺序是否正确。

我们有两个不同的依赖项，我们希望将它们的所有操作记录到一个列表中。所以我们将为他们两个都创建一个间谍。

```go
type CountdownOperationsSpy struct {
    Calls []string
}

func (s *CountdownOperationsSpy) Sleep() {
    s.Calls = append(s.Calls, sleep)
}

func (s *CountdownOperationsSpy) Write(p []byte) (n int, err error) {
    s.Calls = append(s.Calls, write)
    return
}

const write = "write"
const sleep = "sleep"
```

我们的 `CountdownOperationsSpy` 实现了 `io.Writer` 和 `Sleeper`。

记录每一个调用到一个切片。在这个测试中，我们只关心操作的顺序，所以只要将它们记录为命名操作列表就足够了。

我们现在可以在我们的测试套件中添加一个子测试，以验证我们的睡眠和打印操作的顺序

```go
t.Run("sleep before every print", func(t *testing.T) {
    spySleepPrinter := &CountdownOperationsSpy{}
    Countdown(spySleepPrinter, spySleepPrinter)

    want := []string{
        sleep,
        write,
        sleep,
        write,
        sleep,
        write,
        sleep,
        write,
    }

    if !reflect.DeepEqual(want, spySleepPrinter.Calls) {
        t.Errorf("wanted calls %v got %v", want, spySleepPrinter.Calls)
    }
})
```

这个测试现在应该失败了。将 `Countdown`  恢复到修复测试时的状态。

我们现在有两个测试监视 `Sleeper`，所以我们现在可以重构我们的测试，所以一个是测试正在打印什么，另一个是确保我们在打印之间睡觉。我们终于可以删除第一个间谍了，因为它不再被使用了。

```go
func TestCountdown(t *testing.T) {

    t.Run("prints 3 to Go!", func(t *testing.T) {
        buffer := &bytes.Buffer{}
        Countdown(buffer, &CountdownOperationsSpy{})

        got := buffer.String()
        want := `3
2
1
Go!`

        if got != want {
            t.Errorf("got %q want %q", got, want)
        }
    })

    t.Run("sleep before every print", func(t *testing.T) {
        spySleepPrinter := &CountdownOperationsSpy{}
        Countdown(spySleepPrinter, spySleepPrinter)

        want := []string{
            sleep,
            write,
            sleep,
            write,
            sleep,
            write,
            sleep,
            write,
        }

        if !reflect.DeepEqual(want, spySleepPrinter.Calls) {
            t.Errorf("wanted calls %v got %v", want, spySleepPrinter.Calls)
        }
    })
}
```

现在我们已经正确地测试了函数和它的两个重要性质。

## 扩展 Sleeper 可配置

一个不错的功能是 `Sleeper` 可以配置。这意味着我们可以在主程序中调整睡眠时间。

### Write the test first

让我们首先为 `ConfigurableSleeper` 创建一个新类型，它接受我们配置和测试所需要的东西。

```go
type ConfigurableSleeper struct {
    duration time.Duration
    sleep    func(time.Duration)
}
```

We are using `duration` to configure the time slept and `sleep` as a way to pass in a sleep function. 
The signature of `sleep` is the same as for `time.Sleep` allowing us to use `time.Sleep` in our real implementation and the following spy in our tests:


```go
type SpyTime struct {
    durationSlept time.Duration
}

func (s *SpyTime) Sleep(duration time.Duration) {
    s.durationSlept = duration
}
```

我们的 spy 就位后，我们可以为可配置的睡眠者创建一个新的测试。

```go
func TestConfigurableSleeper(t *testing.T) {
    sleepTime := 5 * time.Second

    spyTime := &SpyTime{}
    sleeper := ConfigurableSleeper{sleepTime, spyTime.Sleep}
    sleeper.Sleep()

    if spyTime.durationSlept != sleepTime {
        t.Errorf("should have slept for %v but slept for %v", sleepTime, spyTime.durationSlept)
    }
}
```

这个测试中应该没有新的内容，而且它的设置与以前的模拟测试非常相似。

### Try and run the test
```
sleeper.Sleep undefined (type ConfigurableSleeper has no field or method Sleep, but does have sleep)

```

你应该看到一个非常清晰的错误消息，表明我们没有在我们的 `ConfigurableSleeper` 上创建一个 `Sleep` 方法。

### Write the minimal amount of code for the test to run and check failing test output
```go
func (c *ConfigurableSleeper) Sleep() {
}
```

With our new `Sleep` function implemented we have a failing test.

```
countdown_test.go:56: should have slept for 5s but slept for 0s
```

### Write enough code to make it pass

All we need to do now is implement the `Sleep` function for `ConfigurableSleeper`.

```go
func (c *ConfigurableSleeper) Sleep() {
    c.sleep(c.duration)
}
```

With this change all of the tests should be passing again and you might wonder why all the hassle as the main program didn't change at all. Hopefully it becomes clear after the following section.

### Cleanup and refactor

The last thing we need to do is to actually use our `ConfigurableSleeper` in the main function.

```go
func main() {
    sleeper := &ConfigurableSleeper{1 * time.Second, time.Sleep}
    Countdown(os.Stdout, sleeper)
}
```

If we run the tests and the program manually, we can see that all the behavior remains the same.

Since we are using the `ConfigurableSleeper`, it is now safe to delete the `DefaultSleeper` implementation. Wrapping up our program and having a more [generic](https://stackoverflow.com/questions/19291776/whats-the-difference-between-abstraction-and-generalization) Sleeper with arbitrary long countdowns.

## But isn't mocking evil?

You may have heard mocking is evil. Just like anything in software development it can be used for evil, just like [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself).

People normally get in to a bad state when they don't _listen to their tests_ and are _not respecting the refactoring stage_.

If your mocking code is becoming complicated or you are having to mock out lots of things to test something, you should _listen_ to that bad feeling and think about your code. Usually it is a sign of

- The thing you are testing is having to do too many things (because it has too many dependencies to mock)
  - Break the module apart so it does less
- Its dependencies are too fine-grained
  - Think about how you can consolidate some of these dependencies into one meaningful module
- Your test is too concerned with implementation details
  - Favour testing expected behaviour rather than the implementation

Normally a lot of mocking points to _bad abstraction_ in your code.

**What people see here is a weakness in TDD but it is actually a strength**, more often than not poor test code is a result of bad design or put more nicely, well-designed code is easy to test.

### But mocks and tests are still making my life hard!

Ever run into this situation?

- You want to do some refactoring
- To do this you end up changing lots of tests
- You question TDD and make a post on Medium titled "Mocking considered harmful"

This is usually a sign of you testing too much _implementation detail_. Try to make it so your tests are testing _useful behaviour_ unless the implementation is really important to how the system runs.

It is sometimes hard to know _what level_ to test exactly but here are some thought processes and rules I try to follow:

- **The definition of refactoring is that the code changes but the behaviour stays the same**. If you have decided to do some refactoring in theory you should be able to make the commit without any test changes. So when writing a test ask yourself
  - Am I testing the behaviour I want, or the implementation details?
  - If I were to refactor this code, would I have to make lots of changes to the tests?
- Although Go lets you test private functions, I would avoid it as private functions are implementation detail to support public behaviour. Test the public behaviour. Sandi Metz describes private functions as being "less stable" and you don't want to couple your tests to them.
- I feel like if a test is working with **more than 3 mocks then it is a red flag** - time for a rethink on the design
- Use spies with caution. Spies let you see the insides of the algorithm you are writing which can be very useful but that means a tighter coupling between your test code and the implementation. **Be sure you actually care about these details if you're going to spy on them**

#### Can't I just use a mocking framework?

Mocking requires no magic and is relatively simple; using a framework can make mocking seem more complicated than it is. We don't use automocking in this chapter so that we get:

- a better understanding of how to mock
- practise implementing interfaces

In collaborative projects there is value in auto-generating mocks. In a team, a mock generation tool codifies consistency around the test doubles. This will avoid inconsistently written test doubles which can translate to inconsistently written tests.

You should only use a mock generator that generates test doubles against an interface. Any tool that overly dictates how tests are written, or that use lots of 'magic', can get in the sea.

## Wrapping up

### More on TDD approach

- When faced with less trivial examples, break the problem down into "thin vertical slices". Try to get to a point where you have _working software backed by tests_ as soon as you can, to avoid getting in rabbit holes and taking a "big bang" approach.
- Once you have some working software it should be easier to _iterate with small steps_ until you arrive at the software you need.

> "When to use iterative development? You should use iterative development only on projects that you want to succeed."

Martin Fowler.

### Mocking

- **Without mocking important areas of your code will be untested**. In our case we would not be able to test that our code paused between each print but there are countless other examples. Calling a service that _can_ fail? Wanting to test your system in a particular state? It is very hard to test these scenarios without mocking.
- Without mocks you may have to set up databases and other third parties things just to test simple business rules. You're likely to have slow tests, resulting in **slow feedback loops**.
- By having to spin up a database or a webservice to test something you're likely to have **fragile tests** due to the unreliability of such services.

Once a developer learns about mocking it becomes very easy to over-test every single facet of a system in terms of the _way it works_ rather than _what it does_. Always be mindful about **the value of your tests** and what impact they would have in future refactoring.

In this post about mocking we have only covered **Spies** which are a kind of mock. The "proper" term for mocks though are "test doubles"

[> Test Double is a generic term for any case where you replace a production object for testing purposes.](https://martinfowler.com/bliki/TestDouble.html)

Under test doubles, there are various types like stubs, spies and indeed mocks! Check out [Martin Fowler's post](https://martinfowler.com/bliki/TestDouble.html) for more detail.
