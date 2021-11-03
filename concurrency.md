# Concurrency

**[You can find all the code for this chapter here](https://github.com/quii/learn-go-with-tests/tree/main/concurrency)**

一个同事写了这么个函数 `CheckWebsites`，用来检查列表中 url 的状态。

```go
package concurrency

type WebsiteChecker func(string) bool

func CheckWebsites(wc WebsiteChecker, urls []string) map[string]bool {
	results := make(map[string]bool)

	for _, url := range urls {
		results[url] = wc(url)
	}

	return results
}
```

它返回一个 map， key 为 url，值是 boolean 类型，`true` 代表正常响应，`false` 代表非正常响应。

你要传递一个 `WebsiteChecker` 它接收一个 url，返回一个 boolean 值。它用来检查网站的状态。

使用 [dependency injection][DI] 能让我们测试这个函数，而不要发起真正的 HTTP 请求。

我们可以这样写测试代码：

```go
package concurrency

import (
	"reflect"
	"testing"
)

func mockWebsiteChecker(url string) bool {
	if url == "waat://furhurterwe.geds" {
		return false
	}
	return true
}

func TestCheckWebsites(t *testing.T) {
	websites := []string{
		"http://google.com",
		"http://blog.gypsydave5.com",
		"waat://furhurterwe.geds",
	}

	want := map[string]bool{
		"http://google.com":          true,
		"http://blog.gypsydave5.com": true,
		"waat://furhurterwe.geds":    false,
	}

	got := CheckWebsites(mockWebsiteChecker, websites)

	if !reflect.DeepEqual(want, got) {
		t.Fatalf("Wanted %v, got %v", want, got)
	}
}
```

这个函数是在生成环境中使用过的，检查了上百个网站的状态。但是你的同事开始抱怨变慢了，他们要求你让它搞快点。

## Write a test


使用 benchmark 来测试下 `CheckWebsites` 的速度，这样我们可以看到我们修改的效果。

```go
package concurrency

import (
	"testing"
	"time"
)

func slowStubWebsiteChecker(_ string) bool {
	time.Sleep(20 * time.Millisecond)
	return true
}

func BenchmarkCheckWebsites(b *testing.B) {
	urls := make([]string, 100)
	for i := 0; i < len(urls); i++ {
		urls[i] = "a url"
	}

	for i := 0; i < b.N; i++ {
		CheckWebsites(slowStubWebsiteChecker, urls)
	}
}
```

benchmark 测试 `CheckWebsites` 通过使用一个 `WebsiteChecker` 的 fake 实现，来检查 100 个网址。
`slowStubWebsiteChecker` 通过 Sleep 故意高慢速度，然后直接返回 true。

当我们使用 `go test -bench=.`（or if you're in Windows Powershell `go test -bench="."`）：

```sh
pkg: github.com/gypsydave5/learn-go-with-tests/concurrency/v0
BenchmarkCheckWebsites-4               1        2249228637 ns/op
PASS
ok      github.com/gypsydave5/learn-go-with-tests/concurrency/v0        2.268s
```

`CheckWebsites` has been benchmarked at 2249228637 nanoseconds - about two and
a quarter seconds.

让我们现在来搞快点。

### 写足够的代码使得测试通过Write enough code to make it pass

现在我们终于可以讨论并发了，出于 Following 的意思是“有不止一件事情在进行中”。这是我们每天都在自然地做。

例如，今天早上我泡了一杯茶。我把水壶放上，然后，等它煮开的时候，我把牛奶从冰箱里拿出来，拿了把茶从橱柜里拿出来，找到我最喜欢的杯子，把茶包放进杯子里然后，当壶开了，我把水放在杯子里。

我没有做的是把水壶放在上面，然后站在那里发呆先把水壶煮开，等它煮开后再做其他的事情。

如果你能理解为什么第一种方法泡茶更快，那么你就可以理解我们将如何使 `CheckWebsites` 更快。
而不是等待一个网站在发送请求到下一个网站之前要做出回应，我们会告诉我们的计算机在等待时，无法发出下一个请求。

通常在 Go 中，当我们调用函数 `doSomething()` 时，我们会等待它返回（即使它没有返回值，我们仍然等待它完成）。我们说这个操作是“阻塞”的 —— 它让我们等待它完成。
在 Go 中不阻塞的操作将在一个叫做 `goroutine` 的单独的 `process` 中运行。
把一个进程想象成从上到下阅读 Go 代码页，在每个函数被调用时“进入”每个函数来阅读它做了什么。
当一个单独的进程启动时，它就像另一个读取器开始读取函数，让原来的读者继续往下看。

为了告诉 Go 开始一个新的 goroutine，我们将一个函数调用转换为一个 `Go` 语句，将关键字 `Go` 放在它前面:`go doSomething()`。

```go
package concurrency

type WebsiteChecker func(string) bool

func CheckWebsites(wc WebsiteChecker, urls []string) map[string]bool {
	results := make(map[string]bool)

	for _, url := range urls {
		go func() {
			results[url] = wc(url)
		}()
	}

	return results
}
```

因为要开始一个 goroutine，我们将 `go` 放在调用函数的开始，在开始一个 goroutine 时我们通常使用 *匿名函数*。


匿名函数有许多特性，这使得它们很有用，我们上面正在使用其中两个。
首先，它们可以在声明的同时执行 —— 这就是匿名函数末尾的 `()` 所做的。
其次，它们维护了对定义它们的词法作用域的访问 —— 在声明匿名函数时可用的所有变量在函数体中也可用。

上面的匿名函数体和之前的循环体是一样的。唯一的区别是每次循环迭代都会启动一个新的 goroutine，与当前进程并发( `WebsiteChecker` 函数)每一个都会将其结果添加到结果映射中。

But when we run `go test`:

```sh
--- FAIL: TestCheckWebsites (0.00s)
        CheckWebsites_test.go:31: Wanted map[http://google.com:true http://blog.gypsydave5.com:true waat://furhurterwe.geds:false], got map[]
FAIL
exit status 1
FAIL    github.com/gypsydave5/learn-go-with-tests/concurrency/v1        0.010s

```

### A quick aside into a parallel(ism) universe...

你可能得不到这个结果。你可能会得到一个 panic 信息，我们稍后会讲到。
如果你得到了，不要担心，只要继续运行测试，直到你得到上面的结果。
或者假装你得到了。欢迎使用并发：如果没有正确处理，就很难预测将会发生什么。
别担心，这就是我们编写测试的原因，测试将帮助我们知道何时可预测地处理并发。

### ... and we're back.

我们在最初的测试中发现 `CheckWebsites` 现在返回的是一个空 map。到底是哪里出了错?

我们的 `for` 循环开始的 goroutine 没有足够的时间将它们的结果添加到 `results` 映射中;
`WebsiteChecker` 函数对他们来说太快了，它返回的仍然是空的 map。


为了解决这个问题，我们可以等着所有的 goroutines 完成它们的工作，然后再回来。两秒钟就够了，对吧?

```go
package concurrency

import "time"

type WebsiteChecker func(string) bool

func CheckWebsites(wc WebsiteChecker, urls []string) map[string]bool {
	results := make(map[string]bool)

	for _, url := range urls {
		go func() {
			results[url] = wc(url)
		}()
	}

	time.Sleep(2 * time.Second)

	return results
}
```

Now when we run the tests you get (or don't get - see above):

```sh
--- FAIL: TestCheckWebsites (0.00s)
        CheckWebsites_test.go:31: Wanted map[http://google.com:true http://blog.gypsydave5.com:true waat://furhurterwe.geds:false], got map[waat://furhurterwe.geds:false]
FAIL
exit status 1
FAIL    github.com/gypsydave5/learn-go-with-tests/concurrency/v1        0.010s
```

这不是很好 —— 为什么只有一个结果？我们可以尝试通过增加等待时间来解决这个问题 —— 如果你喜欢的话，可以尝试一下。
它不会工作。这里的问题是变量 `url` 在 `for` 循环的每次迭代中都被重用 - 它每次从 `urls` 获取一个新值。
但是我们的每个 goroutine 都有一个指向 `url` 变量的引用 —— 它们没有自己独立的副本。
所以他们都在写 `url` 在迭代的最后一个 url 的值。
这就是为什么我们得到的结果是最后一个url。

修正：

```go
package concurrency

import (
	"time"
)

type WebsiteChecker func(string) bool

func CheckWebsites(wc WebsiteChecker, urls []string) map[string]bool {
	results := make(map[string]bool)

	for _, url := range urls {
		go func(u string) {
			results[u] = wc(u)
		}(url)
	}

	time.Sleep(2 * time.Second)

	return results
}
```

通过给每个匿名函数一个 url 的参数 - `u` - 然后用 `url` 作为参数调用匿名函数，
我们确保 `u` 的值固定为 `url` 的值，在我们启动 goroutine 的循环的迭代中。
`u` 是 `url` 值的副本，因此不能更改。

如果你比较幸运将得到下面的结果：

```sh
PASS
ok      github.com/gypsydave5/learn-go-with-tests/concurrency/v1        2.012s
```

但如果您运气不好（如果您使用基准运行它们，则更有可能出现这种情况，因为您将得到更多的尝试）

```sh
fatal error: concurrent map writes

goroutine 8 [running]:
runtime.throw(0x12c5895, 0x15)
        /usr/local/Cellar/go/1.9.3/libexec/src/runtime/panic.go:605 +0x95 fp=0xc420037700 sp=0xc4200376e0 pc=0x102d395
runtime.mapassign_faststr(0x1271d80, 0xc42007acf0, 0x12c6634, 0x17, 0x0)
        /usr/local/Cellar/go/1.9.3/libexec/src/runtime/hashmap_fast.go:783 +0x4f5 fp=0xc420037780 sp=0xc420037700 pc=0x100eb65
github.com/gypsydave5/learn-go-with-tests/concurrency/v3.WebsiteChecker.func1(0xc42007acf0, 0x12d3938, 0x12c6634, 0x17)
        /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:12 +0x71 fp=0xc4200377c0 sp=0xc420037780 pc=0x12308f1
runtime.goexit()
        /usr/local/Cellar/go/1.9.3/libexec/src/runtime/asm_amd64.s:2337 +0x1 fp=0xc4200377c8 sp=0xc4200377c0 pc=0x105cf01
created by github.com/gypsydave5/learn-go-with-tests/concurrency/v3.WebsiteChecker
        /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:11 +0xa1

        ... many more scary lines of text ...
```

这很长很吓人，但我们所需要做的就是深呼吸并阅读 stacktrace: `fatal error: concurrent map write`。
有时候，当我们运行我们的测试时，两个 goroutine 在完全相同的时间写入结果 map。
golang 中的 map 不喜欢同时有多个东西试图向它写入，所以这是 `fatal error`。

这是一个 _race condition_，当我们的软件输出时发生的错误取决于我们无法控制的事件的时间和顺序。
因为我们不能准确地控制每个 goroutine 写入结果 map 的时间， 我们很容易受到两个 goroutines 同时向它写入的攻击。

Go 可以通过其内置的 [_race detector_][godoc_race_detector] 帮助我们发现 race conditions。
要启用此功能，请使用 `race` 标志运行测试: `go test -race`。

你应该得到如下的输出:

```sh
==================
WARNING: DATA RACE
Write at 0x00c420084d20 by goroutine 8:
  runtime.mapassign_faststr()
      /usr/local/Cellar/go/1.9.3/libexec/src/runtime/hashmap_fast.go:774 +0x0
  github.com/gypsydave5/learn-go-with-tests/concurrency/v3.WebsiteChecker.func1()
      /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:12 +0x82

Previous write at 0x00c420084d20 by goroutine 7:
  runtime.mapassign_faststr()
      /usr/local/Cellar/go/1.9.3/libexec/src/runtime/hashmap_fast.go:774 +0x0
  github.com/gypsydave5/learn-go-with-tests/concurrency/v3.WebsiteChecker.func1()
      /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:12 +0x82

Goroutine 8 (running) created at:
  github.com/gypsydave5/learn-go-with-tests/concurrency/v3.WebsiteChecker()
      /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:11 +0xc4
  github.com/gypsydave5/learn-go-with-tests/concurrency/v3.TestWebsiteChecker()
      /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker_test.go:27 +0xad
  testing.tRunner()
      /usr/local/Cellar/go/1.9.3/libexec/src/testing/testing.go:746 +0x16c

Goroutine 7 (finished) created at:
  github.com/gypsydave5/learn-go-with-tests/concurrency/v3.WebsiteChecker()
      /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:11 +0xc4
  github.com/gypsydave5/learn-go-with-tests/concurrency/v3.TestWebsiteChecker()
      /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker_test.go:27 +0xad
  testing.tRunner()
      /usr/local/Cellar/go/1.9.3/libexec/src/testing/testing.go:746 +0x16c
==================
```

细节也很难读懂，但是 `WARNING: DATA RACE` 是非常明确的。
从错误体中读取，我们可以看到两个不同的 goroutine 在 map 上执行写操作:

`Write at 0x00c420084d20 by goroutine 8:`

写到相同的内存块

`Previous write at 0x00c420084d20 by goroutine 7:`

在上面，我们可以看到写代码的代码行:

`/Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:12`

以及 goroutines 7 和 8 开始的那行代码:

`/Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:11`

你需要知道的一切都打印到你的终端上了 —— 你所要做的就是有足够的耐心去阅读它。

### Channels

我们可以通过使用 _channels_ 协调 goroutine 来解决这个 data race。
通道是一种可以接收和发送值的 Go 数据结构。
这些操作及其细节允许不同进程之间进行通信。

在本例中，我们想要考虑父进程和它创建的每一个 goroutine 之间的通信。

```go
package concurrency

type WebsiteChecker func(string) bool
type result struct {
	string
	bool
}

func CheckWebsites(wc WebsiteChecker, urls []string) map[string]bool {
	results := make(map[string]bool)
	resultChannel := make(chan result)

	for _, url := range urls {
		go func(u string) {
			resultChannel <- result{u, wc(u)}
		}(url)
	}

	for i := 0; i < len(urls); i++ {
		r := <-resultChannel
		results[r.string] = r.bool
	}

	return results
}
```

在 `results` map 旁边，我们现在有一个 `resultChannel`，我们用同样的方法 `make` 它。
`chan result` 是通道类型 - a channel of `result`.
新的类型，`result` 已经被用来关联 `WebsiteChecker` 的返回值与被检查的url - 它是 `string` 和 `bool` 结构体。
因为我们不需要对这两个值进行命名，所以它们在结构中都是匿名的;当很难知道该命名什么时，这可能很有用一个值。

现在，当我们遍历 url 时，不是直接写入 `map` 而是使用 _send statement_ 为每次调用 `wc` 将 `result` 结构发送给 `resultChannel`。
这使用了 `<-` 操作符，在左边取一个通道，在右边取一个值:

```go
// Send statement
resultChannel <- result{u, wc(u)}
```

下一个 `for` 循环为每个 url 迭代一次。
在内部，我们使用了 _receive expression_，它将从通道接收到的值赋给一个变量。
这也使用了 `<-` 操作符，但现在两个操作数是相反的:通道现在在右边，而要赋值的变量在左边:

```go
// Receive expression
r := <-resultChannel
```

然后我们使用收到的 `result` 来更新 map。

通过将结果发送到一个通道，我们可以控制每次写入 results 映射的时间，确保每次写入一个结果。
尽管每次调用 `wc` 和每次发送到结果通道，在它自己的进程中并行地发生，当我们用接收表达式从结果通道中取值时，每个结果都会被一次处理一个。

我们已经并行化了我们想让代码更快的部分，同时确保不能并行发生的部分仍然线性发生。
我们已经通过使用通道在涉及的多个进程之间进行了通信。

运行 benchmark：

```sh
pkg: github.com/gypsydave5/learn-go-with-tests/concurrency/v2
BenchmarkCheckWebsites-8             100          23406615 ns/op
PASS
ok      github.com/gypsydave5/learn-go-with-tests/concurrency/v2        2.377s
```
23406615 nanoseconds - 0.023 seconds, about one hundred times as fast as
original function. A great success.

## 总结

这个练习在 TDD 上比通常要轻一些。
在某种程度上，我们参与了对 `CheckWebsites` 的长重构;
输入和输出没有变，只是变快了。
但我们的测试，以及我们编写的基准，允许我们重构 `CheckWebsites`，以保持软件仍在工作的信心，同时证明它实际上变得更快了。

为了让它更快，我们学到了

- *goroutines*, Go 中并发的基本单元，它让我们可以同时查看多个网站。
- *anonymous functions*, 我们用它来启动每个检查网站的并发进程。
- *channels*, 帮助组织和控制不同进程之间的通信，使我们避免*竞争条件*错误。
- *the race detector* 它帮助我们调试并发代码的问题

### Make it fast

一种构建软件的敏捷方法经常被误认为是 KentBeck 提出的:

> [Make it work, make it right, make it fast][wrf]

“工作”是让测试通过，“正确”是重构代码，
“快速”是指优化代码，使之快速运行。
只有当我们让它运转并正确时，我们才能“让它快速运转”。
我们很幸运，我们得到的代码已经被证明是有效的，而且不需要重构。
我们永远不应该在其他两个步骤完成之前就试图快速完成，因为

> [过早的优化是万恶之源][popt]
> -- Donald Knuth

[DI]: dependency-injection.md
[wrf]: http://wiki.c2.com/?MakeItWorkMakeItRightMakeItFast
[godoc_race_detector]: https://blog.golang.org/race-detector
[popt]: http://wiki.c2.com/?PrematureOptimization
