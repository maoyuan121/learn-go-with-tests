# Error types

**[You can find all the code here](https://github.com/quii/learn-go-with-tests/tree/main/q-and-a/error-types)**

**创建自己的错误类型是一种整理代码的优雅方式，可以使代码更易于使用和测试。**

“Gopher Slack”上的 Pedro 问道

> 如果我创建了一个错误类似于 `fmt.Errorf("%s must be foo, got %s", bar, baz)`，有没有一种方法来测试相等而不比较字符串值?

让我们创建一个函数来帮助探索这个想法。

```go
// DumbGetter will get the string body of url if it gets a 200
func DumbGetter(url string) (string, error) {
	res, err := http.Get(url)

	if err != nil {
		return "", fmt.Errorf("problem fetching from %s, %v", url, err)
	}

	if res.StatusCode != http.StatusOK {
		return "", fmt.Errorf("did not get 200 from %s, got %d", url, res.StatusCode)
	}

	defer res.Body.Close()
	body, _ := ioutil.ReadAll(res.Body) // ignoring err for brevity

	return string(body), nil
}
```

编写一个可能因为不同原因而失败的函数并不少见，我们希望确保正确地处理每个场景。

正如Pedro所说，我们可以像这样为状态错误编写一个测试。


```go
t.Run("when you don't get a 200 you get a status error", func(t *testing.T) {

	svr := httptest.NewServer(http.HandlerFunc(func(res http.ResponseWriter, req *http.Request) {
		res.WriteHeader(http.StatusTeapot)
	}))
	defer svr.Close()

	_, err := DumbGetter(svr.URL)

	if err == nil {
		t.Fatal("expected an error")
	}

	want := fmt.Sprintf("did not get 200 from %s, got %d", svr.URL, http.StatusTeapot)
	got := err.Error()

	if got != want {
		t.Errorf(`got "%v", want "%v"`, got, want)
	}
})
```

这个测试创建一个总是返回 `StatusTeapot` 的服务器，然后我们使用它的 URL 作为 `DumbGetter` 的参数，所以我们可以看到它正确地处理非 `200` 响应。

## 这种测试方式存在的问题

这本书试图强调倾听你的测试，而这个测试感觉不好:

- 我们正在构建与产品代码相同的字符串来测试它
- 读和写都很烦人
- 确切的错误消息字符串是我们真正关心的吗?

这告诉我们什么?我们测试的人机工程学将反映在另一个试图使用我们代码的代码上。

我们的代码的用户如何对我们返回的特定类型的错误作出反应?他们能做的最好的事情就是查看错误字符串，因为它非常容易出错，而且编写起来很糟糕。

## 我们应该怎么做

有了 TDD，我们就有了这样的心态:

> 我想如何使用这段代码?

我们可以为 `DumbGetter` 提供一种方法，让用户使用类型系统来理解发生了什么类型的错误。

如果 `DumbGetter` 能够返回给我们一些类似的东西会怎么样呢

```go
type BadStatusError struct {
	URL    string
	Status int
}
```

而不是一个神奇的字符串，我们有实际的 _data_ 工作。

让我们改变现有的测试以反映这种需求

```go
t.Run("when you don't get a 200 you get a status error", func(t *testing.T) {

	svr := httptest.NewServer(http.HandlerFunc(func(res http.ResponseWriter, req *http.Request) {
		res.WriteHeader(http.StatusTeapot)
	}))
	defer svr.Close()

	_, err := DumbGetter(svr.URL)

	if err == nil {
		t.Fatal("expected an error")
	}

	got, isStatusErr := err.(BadStatusError)

	if !isStatusErr {
		t.Fatalf("was not a BadStatusError, got %T", err)
	}

	want := BadStatusError{URL: svr.URL, Status: http.StatusTeapot}

	if got != want {
		t.Errorf("got %v, want %v", got, want)
	}
})
```

我们应该使 `BadStatusError` 实现 error 接口。

```go
func (b BadStatusError) Error() string {
	return fmt.Sprintf("did not get 200 from %s, got %d", b.URL, b.Status)
}
```

### What does the test do?

我们不是检查错误的确切字符串，而是在错误上执行 [type assertion](https://tour.golang.org/methods/15)，以查看它是否是 `BadStatusError`。这反映了我们对那种错误比较清楚的愿望。假设断言通过，我们就可以检查错误的属性是否正确。

当我们运行测试时，它告诉我们我们没有返回正确的错误类型

```
--- FAIL: TestDumbGetter (0.00s)
    --- FAIL: TestDumbGetter/when_you_dont_get_a_200_you_get_a_status_error (0.00s)
    	error-types_test.go:56: was not a BadStatusError, got *errors.errorString
```

让我们通过更新错误处理代码来使用我们的类型来修复 `DumbGetter`

```go
if res.StatusCode != http.StatusOK {
	return "", BadStatusError{URL: url, Status: res.StatusCode}
}
```

这种变化产生了一些真正的积极影响

- 我们的 `DumbGetter` 函数变得更简单了，它不再关心一个复杂的错误字符串，它只是创建一个 `BadStatusError`。
- 我们的测试现在反映(并记录)我们代码的用户可以做什么，如果他们决定他们想做一些更复杂的错误处理，而不仅仅是记录。只需要做一个类型断言，就可以很容易地访问错误的属性。
- 它仍然“只是”一个 `error`，所以如果他们选择，他们可以将它传递到调用堆栈或记录它，就像任何其他的 `error`。

## 总结

如果您发现自己在测试多个错误条件，就不会陷入比较错误消息的陷阱。

这就导致了读写测试的不稳定和困难，这也反映了如果你的代码的用户也需要根据所发生的错误的类型来进行不同的操作，那么他们将会遇到的困难。

一定要确保你的测试反映出你想如何使用你的代码，所以在这方面，考虑创建错误类型来封装你的错误类型。这使得代码的用户更容易处理不同类型的错误，也使得编写错误处理代码更简单、更容易阅读。

## Addendum

从 Go 1.13 开始，在[Go 博客](https://blog.golang.org/go1.13-errors)中介绍了标准库中处理错误的新方法。

```go
t.Run("when you don't get a 200 you get a status error", func(t *testing.T) {

	svr := httptest.NewServer(http.HandlerFunc(func(res http.ResponseWriter, req *http.Request) {
		res.WriteHeader(http.StatusTeapot)
	}))
	defer svr.Close()

	_, err := DumbGetter(svr.URL)

	if err == nil {
		t.Fatal("expected an error")
	}

	var got BadStatusError
	isBadStatusError := errors.As(err, &got)
	want := BadStatusError{URL: svr.URL, Status: http.StatusTeapot}

	if !isBadStatusError {
		t.Fatalf("was not a BadStatusError, got %T", err)
	}

	if got != want {
		t.Errorf("got %v, want %v", got, want)
	}
})
```

In this case we are using [`errors.As`](https://golang.org/pkg/errors/#example_As) to try and extract our error into our custom type. It returns a `bool` to denote success and extracts it into `got` for us.
在本例中，我们使用 [`errors.As`](https://golang.org/pkg/errors/#example_As) 尝试将错误提取到自定义类型中。它返回一个 `bool` 来表示成功并为我们将其提取到 `got`中。

