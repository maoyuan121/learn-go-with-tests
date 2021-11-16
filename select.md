# Select

**[You can find all the code for this chapter here](https://github.com/quii/learn-go-with-tests/tree/main/select)**

你已经被要求写一个名为 `WebsiteRacer` 的函数，它接受两个 URL 并通过使用 HTTP GET 来“赛跑”它们，并返回最先返回的 URL。如果在 10 秒内它们都没有返回，那么它应该返回一个 `error`。

为此，我们将使用

- `net/http` 发起 HTTP 请求
- `net/http/httptest` 帮助我们测试
- goroutines.
- `select` 同步 process

## 先写测试

让我们从一些简单的东西开始。

```go
func TestRacer(t *testing.T) {
	slowURL := "http://www.facebook.com"
	fastURL := "http://www.quii.co.uk"

	want := fastURL
	got := Racer(slowURL, fastURL)

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

我们知道这并不完美，也有问题，但它会让我们继续前进。重要的是不要过于执着于第一次就把事情做到完美。

## Try to run the test

`./racer_test.go:14:9: undefined: Racer`

## Write the minimal amount of code for the test to run and check the failing test output

```go
func Racer(a, b string) (winner string) {
	return
}
```

`racer_test.go:25: got '', want 'http://www.quii.co.uk'`

## Write enough code to make it pass

```go
func Racer(a, b string) (winner string) {
	startA := time.Now()
	http.Get(a)
	aDuration := time.Since(startA)

	startB := time.Now()
	http.Get(b)
	bDuration := time.Since(startB)

	if aDuration < bDuration {
		return a
	}

	return b
}
```

For each URL:

1. 我们使用 `time.Now()` 记录 http 请求的开始时间
1. 然后我们使用 [`http.Get`](https://golang.org/pkg/net/http/#Client.Get) 发起 http 请求。这个函数返回一个 [`http.Response`](https://golang.org/pkg/net/http/#Response) 
和一个 `error`，但是我们目前还不关心这些值。
1. `time.Since` 接收开始时间，返回时间差 `time.Duration`。

一旦我们这样做了，我们简单地比较持续时间，看看哪个是最快的。

### Problems

测试可能通过，也可能不通过。问题是，我们用真实的网站来测试我们自己的逻辑。
              
使用 HTTP 的测试代码是如此普遍，以至于 Go 在标准库中提供了工具来帮助您测试它。

在 mocking 和 DI 章节，我们讨论了理想情况下我们不希望依赖外部服务来测试代码，因为它们会
                  
- 慢
- Flaky
- Can't test edge cases

在标准库里，有个叫  [`net/http/httptest`](https://golang.org/pkg/net/http/httptest/) 的包，通过它可以很容易的创建一个 mock HTTP server。

让我们更改测试，使用 mock，这样我们就有了可以控制的可靠服务器进行测试。

```go
func TestRacer(t *testing.T) {

	slowServer := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(20 * time.Millisecond)
		w.WriteHeader(http.StatusOK)
	}))

	fastServer := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	}))

	slowURL := slowServer.URL
	fastURL := fastServer.URL

	want := fastURL
	got := Racer(slowURL, fastURL)

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}

	slowServer.Close()
	fastServer.Close()
}
```

语法可能看起来有点复杂，但请您慢慢来。

`httptest.NewServer` 接收一个 `http.HandlerFunc`，这个方法接收一个 _匿名函数_。

`http.HandlerFunc` 是一个类型，大概就是：`type HandlerFunc func(ResponseWriter, *Request)`。

它真正说的是，它需要一个函数，接受一个 `ResponseWriter` 和一个 `Request`，这对 HTTP 服务器来说并不奇怪。

实际上这里没有额外的魔法，**这也是在 Go 中编写_real_ HTTP 服务器的方法**。
唯一的区别是我们将它包装在一个 `httptest.NewServer` 中使它更容易使用与测试，
因为它找到了一个打开的端口来监听，然后您可以在完成测试后关闭它。

在我们的两个服务器中，我们让慢的那个有很短的时间。当我们收到一个让它比另一个慢的请求时，就进入睡眠状态。
然后两个服务器都用 `w.WriteHeader(http.StatusOK)` 写一个 `OK` 响应回给调用者。

如果您重新运行测试，它现在肯定会通过，并且应该会更快。使用这些睡眠来故意破坏测试。

## 重构

在生产代码和测试代码中都有一些重复代码。

```go
func Racer(a, b string) (winner string) {
	aDuration := measureResponseTime(a)
	bDuration := measureResponseTime(b)

	if aDuration < bDuration {
		return a
	}

	return b
}

func measureResponseTime(url string) time.Duration {
	start := time.Now()
	http.Get(url)
	return time.Since(start)
}
```

现在 `Racer` 的代码更易读了。

```go
func TestRacer(t *testing.T) {

	slowServer := makeDelayedServer(20 * time.Millisecond)
	fastServer := makeDelayedServer(0 * time.Millisecond)

	defer slowServer.Close()
	defer fastServer.Close()

	slowURL := slowServer.URL
	fastURL := fastServer.URL

	want := fastURL
	got := Racer(slowURL, fastURL)

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}

func makeDelayedServer(delay time.Duration) *httptest.Server {
	return httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(delay)
		w.WriteHeader(http.StatusOK)
	}))
}
```

我们重构了假服务器，将其创建为一个名为 `makeDelayedServer` 的函数，以将一些无趣的代码从测试中移出，并减少重复。

### `defer`

通过在函数调用前加上 `defer`，它现在会在包含函数的末尾调用函数。

有时您需要清理资源，例如关闭文件或在我们的情况下关闭服务器，使其不再继续侦听端口。

您希望它在函数的末尾执行，但要将指令保持在创建服务器的位置附近，以便以后的代码读者受益。

我们的重构是一种改进，鉴于到目前为止涉及的 Go 特性，它是一个合理的解决方案，但我们可以使解决方案更简单。

### Synchronising processes

- 为什么我们要一个接一个地测试网站的速度，而 Go 在并发性方面做得很好?我们应该可以同时检查。
- 我们并不真正关心请求的确切响应时间，我们只想知道哪个最先返回。

为了做到这一点，我们将引入一个名为 `select` 的新结构，它帮助我们非常容易和清晰地同步进程。

```go
func Racer(a, b string) (winner string) {
	select {
	case <-ping(a):
		return a
	case <-ping(b):
		return b
	}
}

func ping(url string) chan struct{} {
	ch := make(chan struct{})
	go func() {
		http.Get(url)
		close(ch)
	}()
	return ch
}
```

#### `ping`

我们定义了一个 `ping` 函数，它创建一个 `chat struct{}` 并返回它。

在我们的例子中，我们不关心是什么东西发送给了 channel，我们只是想要一个信号告诉我们完成了，直接关闭这个 channel 就可以了。

为什么是 `struct{}` 而不是其它类，比如 `bool`？，`struct{}` 不占内存。以为我们只需要关闭，而不需要发送任何东西给 channel。

在同一个函数里面，我们开始了一个 goroutine，在里面当我们完成了 `http.Get(url)` 后讲发送一个信号给 channel。

##### Always `make` channels

注意我们应该使用 `make` 创建一个 channel；而不是 `var ch chan struct{}`。当你使用 `var` 变量会初始化为这个类型的零值，对于 `string` 是 `"""`，`int` 是 0。

Channel 的零值是 `nil`，如果你尝试对零值的 channel 发送一个东西将永远被阻塞，以为你不能发送东西给 `nil` channel。

[You can see this in action in The Go Playground](https://play.golang.org/p/IIbeAox5jKA)

#### `select`

回顾一下 concurrency 章节，你可以使用 `myVar := <-ch` 来等待发送给 channel 的值。这将阻塞调用，因为你在等待一个值。

What `select` lets you do is wait on _multiple_ channels. The first one to send a value "wins" and the code underneath the `case` is executed.

`select` 让你做的是等待 _多个_ channels。第一个发送值的“赢”，然后执行 `case` 下面的代码。

我们在 `select` 中使用 `ping` 为两个通道分别建立通道。
无论哪个先写入它的通道，它的代码都会在 `select` 中执行，
结果返回它的 `URL`(并成为赢家)。

在这些更改之后，我们代码背后的意图就非常清楚了，而且实现实际上也更简单了。

### Timeouts

我们最后的需求是如果 `Racer` 花费时间超过 10 秒，那么返回 error。

## Write the test first

```go
t.Run("returns an error if a server doesn't respond within 10s", func(t *testing.T) {
	serverA := makeDelayedServer(11 * time.Second)
	serverB := makeDelayedServer(12 * time.Second)

	defer serverA.Close()
	defer serverB.Close()

	_, err := Racer(serverA.URL, serverB.URL)

	if err == nil {
		t.Error("expected an error but didn't get one")
	}
})
```

我们已经让我们的测试服务器花了超过 10 秒的时间来返回这个场景，我们希望 `Racer` 现在返回两个值，
获胜的 URL(我们在这个测试中用 `_` 忽略了它)和一个 `error`。

## Try to run the test

`./racer_test.go:37:10: assignment mismatch: 2 variables but 1 values`

## Write the minimal amount of code for the test to run and check the failing test output

```go
func Racer(a, b string) (winner string, error error) {
	select {
	case <-ping(a):
		return a, nil
	case <-ping(b):
		return b, nil
	}
}
```

更改 `Racer` 的签名以返回赢家和一个 `error`。返回 `nil` 作为我们满意的情况。

编译器会抱怨你的 _first test_ 只寻找一个值，所以把这一行改为 `got， _:= Racer(slowURL, fastURL)`，知道我们应该检查我们的_don't_得到一个错误。

运行 11 秒后将失败。

```
--- FAIL: TestRacer (12.00s)
    --- FAIL: TestRacer/returns_an_error_if_a_server_doesn't_respond_within_10s (12.00s)
        racer_test.go:40: expected an error but didn't get one
```

## Write enough code to make it pass

```go
func Racer(a, b string) (winner string, error error) {
	select {
	case <-ping(a):
		return a, nil
	case <-ping(b):
		return b, nil
	case <-time.After(10 * time.Second):
		return "", fmt.Errorf("timed out waiting for %s and %s", a, b)
	}
}
```

当使用 `select` 的时候 `time.After` 是一个非常方便的函数。尽管在我们的例子中没有发生这种情况，但如果您正在侦听的通道从未返回值，您可能会编写永远阻塞的代码。
`time.After` 返回一个 `chan` (像 `ping`)，并在你定义的时间之后发送一个信号。

对我们来说，这是完美的;如果 `a` 或 `b` 成功返回，他们就赢了，但如果我们到达 10 秒，那就是我们的 `time.After` 将发送一个信号，我们将返回一个 `error`。

### Slow tests

我们的问题是这个测试需要 10 秒来运行。对于这样一个简单的逻辑，这感觉不太好。

我们可以做的是让超时是可配置的。在我们的测试中，我们可以有一个很短的超时，然后当代码在现实世界中使用时，它可以设置为 10 秒。

```go
func Racer(a, b string, timeout time.Duration) (winner string, error error) {
	select {
	case <-ping(a):
		return a, nil
	case <-ping(b):
		return b, nil
	case <-time.After(timeout):
		return "", fmt.Errorf("timed out waiting for %s and %s", a, b)
	}
}
```

我们的测试现在无法编译，因为我们没有提供超时。

在匆忙将这个默认值添加到两个测试之前，让我们先听一下它们。

- 我们关心“快乐”测试中的超时吗?
- 关于超时的要求是明确的。

有了这些知识，让我们做一点重构，以同情我们的测试和代码的用户。

```go
var tenSecondTimeout = 10 * time.Second

func Racer(a, b string) (winner string, error error) {
	return ConfigurableRacer(a, b, tenSecondTimeout)
}

func ConfigurableRacer(a, b string, timeout time.Duration) (winner string, error error) {
	select {
	case <-ping(a):
		return a, nil
	case <-ping(b):
		return b, nil
	case <-time.After(timeout):
		return "", fmt.Errorf("timed out waiting for %s and %s", a, b)
	}
}
```

我们的用户和我们的第一个测试可以使用 `Racer` (它在内部使用 `ConfigurableRacer`)，而我们的悲伤路径测试可以使用 `ConfigurableRacer`。

```go
func TestRacer(t *testing.T) {

	t.Run("compares speeds of servers, returning the url of the fastest one", func(t *testing.T) {
		slowServer := makeDelayedServer(20 * time.Millisecond)
		fastServer := makeDelayedServer(0 * time.Millisecond)

		defer slowServer.Close()
		defer fastServer.Close()

		slowURL := slowServer.URL
		fastURL := fastServer.URL

		want := fastURL
		got, err := Racer(slowURL, fastURL)

		if err != nil {
			t.Fatalf("did not expect an error but got one %v", err)
		}

		if got != want {
			t.Errorf("got %q, want %q", got, want)
		}
	})

	t.Run("returns an error if a server doesn't respond within the specified time", func(t *testing.T) {
		server := makeDelayedServer(25 * time.Millisecond)

		defer server.Close()

		_, err := ConfigurableRacer(server.URL, server.URL, 20*time.Millisecond)

		if err == nil {
			t.Error("expected an error but didn't get one")
		}
	})
}
```

我在第一个测试中添加了一个最后的检查，以验证我们没有得到一个 `error`。

## 总结

### `select`

- 帮助您在多个通道等待。
- 有时你会想要在 `cases` 中包括 `time.After`，防止你的系统永远阻塞。
 

### `httptest`

- 创建测试服务器的一种方便的方法，这样您就可以拥有可靠和可控的测试。
- 使用与“真正的” `net/http` 服务器相同的接口，这是一致的，对你来说更少的学习。
