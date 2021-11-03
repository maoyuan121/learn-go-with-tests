# Context

**[You can find all the code for this chapter here](https://github.com/quii/learn-go-with-tests/tree/main/context)**


软件经常启动长时间运行的、资源密集型的进程(通常是goroutines)。
如果导致此操作的操作因某种原因被取消或失败，则需要在应用程序中以一致的方式停止这些进程。

如果您不管理这一点，您引以为傲的时髦 Go 应用程序可能会开始难以调试性能问题。

在本章中，我们将使用包 `context` 来帮助我们管理长时间运行的 process。

我们将从一个经典的 web 服务器的例子开始，当点击时，启动一个潜在的长时间运行的进程来获取一些数据，让它在响应中返回。


我们将演练一个场景，其中用户在可以检索数据之前取消请求，我们将确保流程被告知放弃。

我已经设置了一些代码，让我们开始。这是我们的服务器代码。

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprint(w, store.Fetch())
	}
}
```

`Server` 函数接收一个 `Store` 参数，返回一个 `http.HandlerFunc`。Store 定义如下：

```go
type Store interface {
	Fetch() string
}
```

这个返回得函数调用 `store` 的 `Fetch` 方法来获取一些数据并写到 response 中。

在测试中，我们有一个对应的 Store 存根。

```go
type StubStore struct {
	response string
}

func (s *StubStore) Fetch() string {
	return s.response
}

func TestServer(t *testing.T) {
	data := "hello, world"
	svr := Server(&StubStore{data})

	request := httptest.NewRequest(http.MethodGet, "/", nil)
	response := httptest.NewRecorder()

	svr.ServeHTTP(response, request)

	if response.Body.String() != data {
		t.Errorf(`got "%s", want "%s"`, response.Body.String(), data)
	}
}
```

我们想要创建一个更现实的场景，即 `Store` 在用户取消请求之前无法完成 `fetch`。

## Write the test first

我们的处理程序需要一种方法来告诉 `Store` 取消工作。更新接口。

```go
type Store interface {
	Fetch() string
	Cancel()
}
```

我们将需要调整我们的间谍，以便它需要一些时间返回 `data` 和一种知道它已被告知取消的方式。我们还将把它重命名为 `SpyStore`，
因为我们现在正在观察它的叫法。它必须添加 `Cancel` 作为一个方法来实现 `Store` 接口。


```go
type SpyStore struct {
	response string
	cancelled bool
}

func (s *SpyStore) Fetch() string {
	time.Sleep(100 * time.Millisecond)
	return s.response
}

func (s *SpyStore) Cancel() {
	s.cancelled = true
}
```

让我们添加一个新的测试，在 100 毫秒之前取消请求，并检查存储，看看它是否被取消。

```go
t.Run("tells store to cancel work if request is cancelled", func(t *testing.T) {
      data := "hello, world"
      store := &SpyStore{response: data}
      svr := Server(store)

      request := httptest.NewRequest(http.MethodGet, "/", nil)

      cancellingCtx, cancel := context.WithCancel(request.Context())
      time.AfterFunc(5 * time.Millisecond, cancel)
      request = request.WithContext(cancellingCtx)

      response := httptest.NewRecorder()

      svr.ServeHTTP(response, request)

      if !store.cancelled {
          t.Errorf("store was not told to cancel")
      }
  })
```

From the [Go Blog: Context](https://blog.golang.org/context)

> context 包提供了从现有值派生新的上下文值的函数。
这些值形成一个树: 当取消上下文时，从它派生的所有上下文也将被取消。

获取上下文非常重要，以便在给定请求的整个调用堆栈中传播取消。

我们所做的是从 `request` 派生一个新的 `cancellingCtx`，它返回一个 `cancel` 函数。
然后我们使用 `time.AfterFunc` 来安排在 5 毫秒内调用该函数。
最后，我们通过调用 `request.WithContext` 在请求中使用这个新上下文。

## Try to run the test

The test fails as we'd expect.

```go
--- FAIL: TestServer (0.00s)
    --- FAIL: TestServer/tells_store_to_cancel_work_if_request_is_cancelled (0.00s)
    	context_test.go:62: store was not told to cancel
```

## Write enough code to make it pass

记住要遵守TDD。编写最小数量的代码使我们的测试通过。


```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		store.Cancel()
		fmt.Fprint(w, store.Fetch())
	}
}
```

这使得这个测试通过了，但是感觉不太好!我们当然不应该在获取 _every request_ 之前取消 `Store`。

它突出了我们测试中的一个缺陷，这是一件好事!

我们需要更新测试，以确保它没有被取消。


```go
t.Run("returns data from store", func(t *testing.T) {
    data := "hello, world"
    store := &SpyStore{response: data}
    svr := Server(store)

    request := httptest.NewRequest(http.MethodGet, "/", nil)
    response := httptest.NewRecorder()

    svr.ServeHTTP(response, request)

    if response.Body.String() != data {
        t.Errorf(`got "%s", want "%s"`, response.Body.String(), data)
    }

    if store.cancelled {
        t.Error("it should not have cancelled the store")
    }
})
```

测试现在应该失败，现在我们被迫做一个更合理的实现。

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		ctx := r.Context()

		data := make(chan string, 1)

		go func() {
			data <- store.Fetch()
		}()

		select {
		case d := <-data:
			fmt.Fprint(w, d)
		case <-ctx.Done():
			store.Cancel()
		}
	}
}
```

我们在这里做了什么?

`context` 有一个方法 `Done()`，它返回一个通道，当上下文被 `Done` 或 `cancelled` 时发送一个信号。
我们想要听到这个信号并调用 `store.Cancel`，如果我们得到它但我们想忽略它如果我们的 `Store` 成功地在它之前 `Fetch`。

为了管理这个，我们在 goroutine 中运行 `Fetch`，它将结果写入一个新的通道 `data`。
然后我们使用 `select`  有效地 race 两个异步进程，然后我们要么写一个响应，要么写一个 `Cancel`。

## Refactor

我们可以通过在间谍上创建断言方法来重构测试代码


```go
type SpyStore struct {
	response  string
	cancelled bool
	t         *testing.T
}

func (s *SpyStore) assertWasCancelled() {
	s.t.Helper()
	if !s.cancelled {
		s.t.Errorf("store was not told to cancel")
	}
}

func (s *SpyStore) assertWasNotCancelled() {
	s.t.Helper()
	if s.cancelled {
		s.t.Errorf("store was told to cancel")
	}
}
```

记得当创建 spy  的时候将 `*testing.T` 给传递进去。

```go
func TestServer(t *testing.T) {
	data := "hello, world"

	t.Run("returns data from store", func(t *testing.T) {
		store := &SpyStore{response: data, t: t}
		svr := Server(store)

		request := httptest.NewRequest(http.MethodGet, "/", nil)
		response := httptest.NewRecorder()

		svr.ServeHTTP(response, request)

		if response.Body.String() != data {
			t.Errorf(`got "%s", want "%s"`, response.Body.String(), data)
		}

		store.assertWasNotCancelled()
	})

	t.Run("tells store to cancel work if request is cancelled", func(t *testing.T) {
		store := &SpyStore{response: data, t: t}
		svr := Server(store)

		request := httptest.NewRequest(http.MethodGet, "/", nil)

		cancellingCtx, cancel := context.WithCancel(request.Context())
		time.AfterFunc(5*time.Millisecond, cancel)
		request = request.WithContext(cancellingCtx)

		response := httptest.NewRecorder()

		svr.ServeHTTP(response, request)

		store.assertWasCancelled()
	})
}
```

这种方法是可行的，但它是否具有习惯用法?

我们的 web 服务器手动取消 `Store` 有意义吗?
如果 `Store` 碰巧也依赖于其他运行缓慢的进程呢?
我们必须确保 `Store.Cancel` 正确地将取消传播到它的所有依赖项。

`context` 的一个要点是，它是提供取消的一致方式。

[From the go doc](https://golang.org/pkg/context/)


> 向服务器发出的传入请求应该创建一个 Context，而向服务器发出的调用应该接受一个 Context。
它们之间的函数调用链必须传播上下文，可以选择用派生上下文替换它，派生上下文使用 WithCancel、WithDeadline、WithTimeout 或 WithValue 创建。
当取消上下文时，从它派生的所有上下文也将被取消。

From the [Go Blog: Context](https://blog.golang.org/context) again:

> 在谷歌，我们要求 Go 程序员将 Context 参数作为第一个参数传递给传入和传出请求之间的调用路径上的每个函数。
这使得许多不同团队开发的 Go 代码能够很好地互操作。
它提供了对超时和取消的简单控制，并确保安全凭据等关键值能够正确地传输 Go 程序。

(Pause for a moment and think of the ramifications of every function having to send in a context, and the ergonomics of that.)

感觉有点不舒服?好。让我们尝试着遵循这种方法，而不是通过 `context` 传递给我们的 `Store`，让它负责任。通过这种方式，它也可以将 `context` 传递给它的依赖者，它们也可以负责停止自己。


## Write the test first

我们将不得不改变现有的测试，因为它们的职责正在发生变化。
我们的处理器现在唯一负责的事情是确保它发送一个 context 到下游的 `Store`，它处理的错误将来自 `Store` 当它被取消。

让我们更新 `Store` 接口以显示新的职责。

```go
type Store interface {
	Fetch(ctx context.Context) (string, error)
}
```

删除 handler 里面的代码

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
	}
}
```

更新我们的 `SpyStore`

```go
type SpyStore struct {
	response string
	t        *testing.T
}

func (s *SpyStore) Fetch(ctx context.Context) (string, error) {
	data := make(chan string, 1)

	go func() {
		var result string
		for _, c := range s.response {
			select {
			case <-ctx.Done():
				s.t.Log("spy store got cancelled")
				return
			default:
				time.Sleep(10 * time.Millisecond)
				result += string(c)
			}
		}
		data <- result
	}()

	select {
	case <-ctx.Done():
		return "", ctx.Err()
	case res := <-data:
		return res, nil
	}
}
```

我们必须让我们的 spy 的行为像一个真正的方法，与 `context` 一起工作。

我们正在模拟一个缓慢的过程，通过在 goroutine 中一个字符一个字符地添加字符串来缓慢地构建结果。
当 goroutine 完成它的工作时，它将字符串写入 `data` 通道。
goroutine监听 `ctx.Done`，并在该通道中发送信号时停止工作。

最后，代码使用另一个 `select` 来等待 goroutine 完成它的工作或取消发生。

这与之前的方法类似，我们使用 Go 的并发原语使两个异步进程相互竞争以确定返回的内容。

在编写接受 `context` 的函数和方法时，您将采用类似的方法，因此请确保您理解发生了什么。

我们终于可以更新测试了。注释掉我们的取消测试，这样我们可以先修复快乐路径测试。

```go
t.Run("returns data from store", func(t *testing.T) {
    data := "hello, world"
    store := &SpyStore{response: data, t: t}
    svr := Server(store)

    request := httptest.NewRequest(http.MethodGet, "/", nil)
    response := httptest.NewRecorder()

    svr.ServeHTTP(response, request)

    if response.Body.String() != data {
        t.Errorf(`got "%s", want "%s"`, response.Body.String(), data)
    }
})
```

## Try to run the test

```
=== RUN   TestServer/returns_data_from_store
--- FAIL: TestServer (0.00s)
    --- FAIL: TestServer/returns_data_from_store (0.00s)
    	context_test.go:22: got "", want "hello, world"
```

## Write enough code to make it pass

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		data, _ := store.Fetch(r.Context())
		fmt.Fprint(w, data)
	}
}
```

我们的幸福之路应该是……快乐。现在我们可以修改另一个测试了。

## Write the test first

我们需要测试我们没有对错误情况编写任何类型的响应。
不幸的是 `httptest.ResponseRecorder` 没有办法解决这个问题，所以我们必须扮演我们自己的间谍来测试。


```go
type SpyResponseWriter struct {
	written bool
}

func (s *SpyResponseWriter) Header() http.Header {
	s.written = true
	return nil
}

func (s *SpyResponseWriter) Write([]byte) (int, error) {
	s.written = true
	return 0, errors.New("not implemented")
}

func (s *SpyResponseWriter) WriteHeader(statusCode int) {
	s.written = true
}
```

我们的 `SpyResponseWriter` 实现了 `http.ResponseWriter`，因此可以在测试中使用它。

```go
t.Run("tells store to cancel work if request is cancelled", func(t *testing.T) {
    store := &SpyStore{response: data, t: t}
    svr := Server(store)

    request := httptest.NewRequest(http.MethodGet, "/", nil)

    cancellingCtx, cancel := context.WithCancel(request.Context())
    time.AfterFunc(5*time.Millisecond, cancel)
    request = request.WithContext(cancellingCtx)

    response := &SpyResponseWriter{}

    svr.ServeHTTP(response, request)

    if response.written {
        t.Error("a response should not have been written")
    }
})
```

## Try to run the test

```
=== RUN   TestServer
=== RUN   TestServer/tells_store_to_cancel_work_if_request_is_cancelled
--- FAIL: TestServer (0.01s)
    --- FAIL: TestServer/tells_store_to_cancel_work_if_request_is_cancelled (0.01s)
    	context_test.go:47: a response should not have been written
```

## Write enough code to make it pass

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		data, err := store.Fetch(r.Context())

		if err != nil {
			return // todo: log error however you like
		}

		fmt.Fprint(w, data)
	}
}
```

我们可以看到，在这之后，服务器代码变得简化了，因为它不再显式地负责取消，
它只是通过 `context`，并依赖于下游功能来尊重任何可能发生的取消。

## 总结

### What we've covered

- 如何测试 client 取消了请求的 HTTP handler。
- 如何使用 context 管理 cancellation。
- 如何编写一个函数，接受 `context`，并使用它来通过goroutines， `select` 和通道取消自己。
- 遵循谷歌关于如何通过调用堆栈传播请求范围上下文来管理取消的指导方针。
- How to roll your own spy for `http.ResponseWriter` if you need it.

### What about context.Value ?

[Michal Štrba](https://faiface.github.io/post/context-should-go-away-go2/) 和我有类似的观点。

> 如果你在我的公司使用 ctx.Value，你将被开除

一些工程师提倡通过 `context` 传递值，因为它感觉方便。

方便性往往是导致糟糕代码的原因。

`context.Values` 的问题是，它只是一个无类型映射，所以你没有类型安全，你必须处理它，实际上不包含你的值。
你必须创建一个从一个模块到另一个模块的映射键耦合，如果有人改变了一些东西，就会开始破坏。

简而言之，**如果函数需要一些值，将它们作为类型化参数，而不是试图从 `context.Value` 获取它们**。
这使得每个人都可以静态地检查和记录它。

#### But...

另一方面，在上下文中包含与请求正交的信息可能会很有帮助，例如 trace id。
这个信息可能不会被调用堆栈中的每个函数所需要，并且会使函数签名非常混乱。

[Jack Lindamood 说 **Context.Value 应该通知，而不是控制**](https://medium.com/@cep21/how-to-correctly-use-context-context-in-go-1-7-8f2c0fafdf39)

> context.Value 是针对维护者而不是用户。它永远不应该被要求输入文件或预期的结果。



### Additional material

- 我真的很喜欢阅读 [Context should go away for Go 2 by Michal Štrba](https://faiface.github.io/post/context-should-go-away-go2/)。
他的论点是，到处传递 `context` 是一种气味，
它指出了语言在取消方面的缺陷。
他说，如果能在语言层面解决这个问题，而不是在库层面，那就更好了。
在此之前，如果您想管理长时间运行的流程，您将需要 `context`。
- The [Go blog further describes the motivation for working with `context` and has some examples](https://blog.golang.org/context)
