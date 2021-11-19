# Sync

**[You can find all the code for this chapter here](https://github.com/quii/learn-go-with-tests/tree/main/sync)**

我们想在并发的情况下安全的使用 counter.

我们将从一个不安全的计数器开始，并验证它是否在单线程环境中能正常工作。

然后，我们将通过多个 goroutine 来重现它的不安全性，尝试通过测试来使用它并修复它。

## Write the test first

我们希望 API 给我们一个方法来增加计数器，然后检索它的值。

```go
func TestCounter(t *testing.T) {
	t.Run("incrementing the counter 3 times leaves it at 3", func(t *testing.T) {
		counter := Counter{}
		counter.Inc()
		counter.Inc()
		counter.Inc()

		if counter.Value() != 3 {
			t.Errorf("got %d, want %d", counter.Value(), 3)
		}
	})
}
```

## Try to run the test

```
./sync_test.go:9:14: undefined: Counter
```

## 为要运行的测试编写最小数量的代码，并检查失败的测试输出

Let's define `Counter`.

```go
type Counter struct {

}
```

再试一次，它会以以下方式失败

```
./sync_test.go:14:10: counter.Inc undefined (type Counter has no field or method Inc)
./sync_test.go:18:13: counter.Value undefined (type Counter has no field or method Value)
```

为了最终运行测试，我们可以定义这些方法

```go
func (c *Counter) Inc() {

}

func (c *Counter) Value() int {
	return 0
}
```

它现在应该运行并失败

```
=== RUN   TestCounter
=== RUN   TestCounter/incrementing_the_counter_3_times_leaves_it_at_3
--- FAIL: TestCounter (0.00s)
    --- FAIL: TestCounter/incrementing_the_counter_3_times_leaves_it_at_3 (0.00s)
    	sync_test.go:27: got 0, want 3
```

## Write enough code to make it pass

对于像我们这样的 go 专家来说，这应该是微不足道的。我们需要在数据类型中为计数器保留一些状态，然后在每次 `Inc` 调用时增加它


```go
type Counter struct {
	value int
}

func (c *Counter) Inc() {
	c.value++
}

func (c *Counter) Value() int {
	return c.value
}
```

## Refactor

这里没有太多需要重构的东西，但考虑到我们将围绕 `Counter` 编写更多测试，我们将编写一个小断言函数 `assertCount`，这样测试读起来更清楚一些。

```go
t.Run("incrementing the counter 3 times leaves it at 3", func(t *testing.T) {
    counter := Counter{}
    counter.Inc()
    counter.Inc()
    counter.Inc()

    assertCounter(t, counter, 3)
})

func assertCounter(t testing.TB, got Counter, want int)  {
	t.Helper()
	if got.Value() != want {
		t.Errorf("got %d, want %d", got.Value(), want)
	}
}
```

## Next steps

这很简单，但现在我们要求它必须在并发环境中安全使用。我们将需要编写一个失败的测试来练习这一点。

## Write the test first

```go
t.Run("it runs safely concurrently", func(t *testing.T) {
    wantedCount := 1000
    counter := Counter{}

    var wg sync.WaitGroup
    wg.Add(wantedCount)

    for i := 0; i < wantedCount; i++ {
        go func(w *sync.WaitGroup) {
            counter.Inc()
            w.Done()
        }(&wg)
    }
    wg.Wait()

    assertCounter(t, counter, wantedCount)
})
```

这将循环遍历 `wantedCount` 并触发一个调用 `counter.Inc()` 的 goroutine。


我们使用 [`sync.WaitGroup`](https://golang.org/pkg/sync/#WaitGroup) 这是同步并发进程的一种方便的方法。
                                                                

> WaitGroup 等待一组 goroutine 完成。
主 goroutine 调用 Add 来设置要等待的 goroutine 的数量。
然后每个 goroutine 运行并在完成时调用Done。
同时，可以使用 Wait 来阻塞，直到所有 goroutin e完成。

在执行断言之前等待 `wg.Wait()` 完成，我们可以确保所有 goroutine 都试图 `Inc` 这个 `Counter`。

## Try to run the test

```
=== RUN   TestCounter/it_runs_safely_in_a_concurrent_envionment
--- FAIL: TestCounter (0.00s)
    --- FAIL: TestCounter/it_runs_safely_in_a_concurrent_envionment (0.00s)
    	sync_test.go:26: got 939, want 1000
FAIL
```

这个测试 _可能_ 会失败输出了不同的数字，但尽管如此，它证明了当多个 goroutine 同时试图改变计数器的值时，它是不起作用的。

## Write enough code to make it pass

一个简单的解决方案是给我们的 `Counter`添加一个锁，一个 [`Mutex`](https://golang.org/pkg/sync/#Mutex)

> Mutex 是一种互斥锁。互斥锁的零值是一个未锁定的互斥锁。

```go
type Counter struct {
	mu sync.Mutex
	value int
}

func (c *Counter) Inc() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.value++
}
```

这个是什么意思呢?任意 goroutine 调用 `Inc` 将获得 `Counter` 的锁, 如果这个 goroutine 是第一个的话.其它的 goroutine 将等待它被 `Unlock` 后才能进入。

如果您现在重新运行测试，那么它现在应该通过了，因为每个 goroutine 在进行更改之前都必须等待轮到自己。

## 我还见过其他同步的例子。`sync.Mutex` 嵌入到结构体中。
 
你可能看过类似下面的列子:

```go
type Counter struct {
	sync.Mutex
	value int
}
```

可以这样说，它可以使代码更优雅一些。

```go
func (c *Counter) Inc() {
	c.Lock()
	defer c.Unlock()
	c.value++
}
```

这看起来不错，但编程是一个非常主观的学科，这是糟糕的和错误的。

有时人们忘记了嵌入类型意味着该类型的方法成为公共接口的一部分;你通常不会想要那样。
记住，我们应该非常小心我们的公共 api，当我们让一些东西成为公共的时候，其他代码就可以把自己和它结合起来。我们总是希望避免不必要的耦合。

显示“锁定”和“解锁”最好的情况是令人困惑，但在最坏的情况下，如果您的类型的调用者开始调用这些方法，则可能对您的软件非常有害。
暴露 `Lock` 和 `Unlock` 最好的情况是令人困惑，但在最坏的情况下，如果您的类型的调用者开始调用这些方法，则可能对您的软件非常有害。

![Showing how a user of this API can wrongly change the state of the lock](https://i.imgur.com/SWYNpwm.png)

_This seems like a really bad idea_

## Copying mutexes

测试通过了,但是我们的代码还是有一点风险.

如果你运行 `go vet`, 你应该会得到下面的错误

```
sync/v2/sync_test.go:16: call of assertCounter copies lock value: v1.Counter contains sync.Mutex
sync/v2/sync_test.go:39: assertCounter passes lock by value: v1.Counter contains sync.Mutex
```

查看 [`sync.Mutex`](https://golang.org/pkg/sync/#Mutex) 文档

> Mutex 互斥锁在第一次使用后不能被复制。

当我们传递 `Counter` (by value) 给 `assertCounter`, 它将试着创建一个 mutex 的副本.

为了解决这个问题, 我们应该传递指向 `Counter` 的指针, 因此修改 `assertCounter` 的签名

```go
func assertCounter(t *testing.T, got *Counter, want int)
```

我们的测试将不再编译，因为我们试图传递 `Counter` 而不是 `*Counter`。
为了解决这个问题，我更喜欢创建一个构造函数，让 API 的读者知道最好不要自己初始化类型。

```go
func NewCounter() *Counter {
	return &Counter{}
}
```

在初始化 `Counter` 时，请在测试中使用此函数。

## Wrapping up

我们已经介绍了 [sync package](https://golang.org/pkg/sync/) 中的一些内容

- `Mutex` 能让我们给我们的数据添加锁
- `Waitgroup` 表示等待 goroutine 完成

### 什么时候在通道和 goroutine 上使用锁?

[We've previously covered goroutines in the first concurrency chapter](concurrency.md) which let us write safe concurrent code so why would you use locks?
[The go wiki has a page dedicated to this topic; Mutex Or Channel](https://github.com/golang/go/wiki/MutexOrChannel)

> 一个常见的 Go 新手错误是过度使用 channel 和 goroutine，仅仅因为它是可能的，或者因为它很有趣。
不要害怕使用 sync.Mutex，如果它最适合你的问题。
Go 是实用的，它让你使用最能解决问题的工具，而不是强迫你使用一种代码风格。

Paraphrasing:

- **在传递数据所有权时使用通道**
-- **使用 mutexes 来管理状态**

### go vet

记住，在构建脚本中使用 go vet，因为它可以在代码中出现一些微妙的错误时提醒您，以免它们影响到可怜的用户。

### 不要因为方便而使用 embedding

- 考虑一下内嵌对公共API的影响。
- 您真的想公开这些方法，并让人们将自己的代码耦合到这些方法上吗?
- 对于互斥锁来说，这可能会以非常不可预测和奇怪的方式带来潜在的灾难，想象一下一些邪恶的代码在不应该解锁互斥锁的时候解锁它;这将导致一些非常奇怪的错误，将很难跟踪。
   
