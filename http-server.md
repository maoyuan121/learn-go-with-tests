# HTTP Server

**[You can find all the code for this chapter here](https://github.com/quii/learn-go-with-tests/tree/main/http-server)**

你被要求写一个  web server 可以 用来追踪玩家的输赢情况。

-   `GET /players/{name}` 返回玩家赢的次数
-   `POST /players/{name}` 记录玩家赢了一次

我们将遵循 TDD 方法，尽可能快地获得可工作的软件，然后进行小规模的迭代改进，直到我们有了解决方案。通过采用这种方法，我们

- 在任何给定的时间保持问题空间较小
- 不进入到未知
- 如果我们被卡住/迷失了，做一个恢复不会损失大量的工作。

## Red, green, refactor

在这本书中，我们强调了 TDD 的过程，即编写一个测试并看着它失败(红色)，编写最小数量的代码使其工作(绿色)，然后重构。

就 TDD 给您带来的安全性而言，编写最少代码的原则非常重要。你应该努力尽快摆脱“红色”。

Kent Beck 是这样描述的:

> 让测试快速进行，在过程中犯任何必要的错误。

您可以犯这些错误，因为您将在测试的安全性支持下进行重构。

### What if you don't do this?

红色标记的更改越多，就越有可能添加更多测试没有覆盖的问题。

这个想法是用小步骤迭代地编写有用的代码，由测试驱动，这样您就不会陷入数小时的兔子洞。

### Chicken and egg

我们如何增量地构建它?我们不能 `GET` 一个玩家没有存储的东西，似乎很难知道 `POST` 是否工作没有 `GET` 端点已经存在。

这就是 _mocking_ 的光芒。

- `GET` 将需要一个 `PlayerStore` 为玩家获得分数。这应该是一个接口，所以当我们测试时，我们可以创建一个简单的存根来测试我们的代码，而不需要实现任何实际的存储代码
- 对于 `POST`，我们可以监听它对 `PlayerStore` 的调用，以确保它正确存储玩家。我们的保存实现不会与检索耦合
- 为了快速获得一些工作软件，我们可以做一个非常简单的内存实现，然后我们可以创建一个由我们喜欢的任何存储机制支持的实现

## Write the test first

我们可以编写一个测试，并通过返回一个硬编码的值来启动测试。肯特·贝克(Kent Beck)称之为“Faking it”。一旦我们有了一个可以工作的测试，我们就可以编写更多的测试来帮助我们移除这个常量。

通过完成这个非常小的步骤，我们可以在不需要过多担心应用程序逻辑的情况下，获得一个正确工作的整体项目结构的重要开端。

在 golang 中创建一个  web server 一般是调用 [ListenAndServe](https://golang.org/pkg/net/http/#ListenAndServe)。

```go
func ListenAndServe(addr string, handler Handler) error
```

这将启动一个 web 服务器侦听一个端口，为每个请求创建一个 goroutine，并在 [`Handler`](https://golang.org/pkg/net/http/#Handler) 上运行它。

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

类型通过实现 `ServeHTTP` 方法来实现 Handler 接口，该方法需要两个参数，第一个是写入 response 的地方，第二个是发送到服务器的 HTTP 请求。

创建一个文件 `server_test.go`，为 `PlayerServer` 写一个测试，它接收两个参数。发送的请求将得到一个玩家的分数，我们希望是“20”。


```go
func TestGETPlayers(t *testing.T) {
	t.Run("returns Pepper's score", func(t *testing.T) {
		request, _ := http.NewRequest(http.MethodGet, "/players/Pepper", nil)
		response := httptest.NewRecorder()

		PlayerServer(response, request)

		got := response.Body.String()
		want := "20"

		if got != want {
			t.Errorf("got %q, want %q", got, want)
		}
	})
}
```

为了测试我们的服务器，我们将需要一个 `Request` 发送进来，我们想要监视我们的处理器写入 `ResponseWriter` 的内容。

-  我们使用 `http.NewRequest` 创建一个请求。第一个参数是请求方式，第二是请求路径。`nil` 参数代表请求体，在这里我们不需要请求体。
- `net/http/httptest` 已经为我们做了一个间谍 `ResponseRecorder` 所以我们可以使用它。它有许多有用的方法来检查作为回应所写的内容。

## Try to run the test

`./server_test.go:13:2: undefined: PlayerServer`

## Write the minimal amount of code for the test to run and check the failing test output

创建一个文件  `server.go`，在其中定义 `PlayerServer`

```go
func PlayerServer() {}
```

Try again

```
./server_test.go:13:14: too many arguments in call to PlayerServer
    have (*httptest.ResponseRecorder, *http.Request)
    want ()
```

为函数添加参数

```go
import "net/http"

func PlayerServer(w http.ResponseWriter, r *http.Request) {

}
```

现在可以编译了，但是测试还是失败

```
=== RUN   TestGETPlayers/returns_Pepper's_score
    --- FAIL: TestGETPlayers/returns_Pepper's_score (0.00s)
        server_test.go:20: got '', want '20'
```

## Write enough code to make it pass

从 DI 章节，我们知道了 net/http 的 `ResponseWriter` 实现了 io `Writer`, 因此我们可以使用 `fmt.Fprint` 来发送字符串作为 HTTP 响应。


```go
func PlayerServer(w http.ResponseWriter, r *http.Request) {
	fmt.Fprint(w, "20")
}
```

现在测试应该能通过了。

## Complete the scaffolding


我们希望将其连接到应用程序中。这很重要，因为

- 我们将拥有实际工作的软件，我们不想为了它而编写测试，看到代码运行是很好的
- 当我们重构代码时，很可能会改变程序的结构。我们希望确保这也反映在我们的应用程序中，作为增量方法的一部分

为我们的应用创建一个 `main.go` 文件，代码如下

```go
package main

import (
	"log"
	"net/http"
)

func main() {
	handler := http.HandlerFunc(PlayerServer)
    log.Fatal(http.ListenAndServe(":5000", handler))
}
```

到目前为止，我们所有的应用程序代码都在一个文件中，但是，对于希望将内容分离到不同文件中的大型项目来说，这不是最佳实践。

运行这个，`go build` 将获取目录下所有的 `.go` 文件，然后构建一个程序。然后你可以用 `./myprogram` 来执行它。

### `http.HandlerFunc`

早些时候，我们探讨了 `Handler` 接口是我们需要实现的以便创建服务器。我们通常是通过创建一个 `struct`，并通过实现 ServeHTTP 方法来实现接口。然而，结构体的用例是用来保存数据的，但是我们目前还没有状态，所以创建一个状态是不对的。

[HandlerFunc](https://golang.org/pkg/net/http/#HandlerFunc) 能让我们避免这种情况。

> HandlerFunc类型是一个适配器，允许使用普通函数作为 HTTP 处理程序。如果 f 是具有适当签名的函数，则 HandlerFunc(f) 是调用 f 的处理程序。

```go
type HandlerFunc func(ResponseWriter, *Request)
```

从文档中，我们看到类型 `HandlerFunc` 已经实现了 `ServeHTTP` 方法。
通过类型铸造我们的 `PlayerServer` 函数，我们现在已经实现了所需的 `Handler`。

### `http.ListenAndServe(":5000"...)`

`ListenAndServe` 需要一个端口来监听 `Handler`。如果出现问题，web 服务器将返回一个错误，一个例子可能是已经被监听的端口。出于这个原因，我们将调用 `log.Fatal` 将错误记录给用户。

我们现在要做的是编写另一个测试来迫使我们做出积极的改变，尝试远离硬编码的值。

## Write the test first

我们将在套件中添加另一个子测试，它试图获取不同玩家的分数，这将破坏我们的硬编码方法。

```go
t.Run("returns Floyd's score", func(t *testing.T) {
	request, _ := http.NewRequest(http.MethodGet, "/players/Floyd", nil)
	response := httptest.NewRecorder()

	PlayerServer(response, request)

	got := response.Body.String()
	want := "10"

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
})
```

你可能在想

> 当然，我们需要一些存储概念来控制玩家获得多少分数。奇怪的是，在我们的测试中，这些值似乎是任意的。

记住，我们只是在尝试尽可能小的步骤，所以我们现在只是试图打破常数。

## Try to run the test

```
=== RUN   TestGETPlayers/returns_Pepper's_score
    --- PASS: TestGETPlayers/returns_Pepper's_score (0.00s)
=== RUN   TestGETPlayers/returns_Floyd's_score
    --- FAIL: TestGETPlayers/returns_Floyd's_score (0.00s)
        server_test.go:34: got '20', want '10'
```

## Write enough code to make it pass

```go
//server.go
func PlayerServer(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	if player == "Pepper" {
		fmt.Fprint(w, "20")
		return
	}

	if player == "Floyd" {
		fmt.Fprint(w, "10")
		return
	}
}
```

这个测试迫使我们查看请求的 URL 并做出决定。所以在我们的脑海中，我们可能一直在担心玩家 store 和界面，而下一个逻辑步骤似乎是关于路由。

如果我们从存储代码开始，那么我们需要做的更改量将比这要大得多。**这是我们朝着最终目标迈出的一小步，是由测试驱动的**。

我们现在抵制住了使用任何路由库的诱惑，只是让我们的测试通过的最小步骤。

`r.URL.Path` 返回请求的路径，然后我们可以使用[`strings.TrimPrefix`](https://golang.org/pkg/strings/#TrimPrefix)修剪掉 `/players/` `以获得请求的玩家。它不是很强大，但目前可以做到。

## Refactor

我们可以通过将分数检索分离到一个函数中来简化 `PlayerServer`

```go
//server.go
func PlayerServer(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	fmt.Fprint(w, GetPlayerScore(player))
}

func GetPlayerScore(name string) string {
	if name == "Pepper" {
		return "20"
	}

	if name == "Floyd" {
		return "10"
	}

	return ""
}
```

我们可以通过制作一些 helpers 来 DRY 测试中的一些代码

```go
//server_test.go
func TestGETPlayers(t *testing.T) {
	t.Run("returns Pepper's score", func(t *testing.T) {
		request := newGetScoreRequest("Pepper")
		response := httptest.NewRecorder()

		PlayerServer(response, request)

		assertResponseBody(t, response.Body.String(), "20")
	})

	t.Run("returns Floyd's score", func(t *testing.T) {
		request := newGetScoreRequest("Floyd")
		response := httptest.NewRecorder()

		PlayerServer(response, request)

		assertResponseBody(t, response.Body.String(), "10")
	})
}

func newGetScoreRequest(name string) *http.Request {
	req, _ := http.NewRequest(http.MethodGet, fmt.Sprintf("/players/%s", name), nil)
	return req
}

func assertResponseBody(t testing.TB, got, want string) {
	t.Helper()
	if got != want {
		t.Errorf("response body is wrong, got %q want %q", got, want)
	}
}
```

然而，我们仍然不应该快乐。让我们的服务器知道比分感觉不太对。

我们的重构已经非常清楚该做什么了。

我们将分数计算从处理程序的主体移到了函数 `GetPlayerScore` 中。这似乎是使用接口分离关注点的正确地方。

让我们把重构后的函数移到接口

```go
type PlayerStore interface {
	GetPlayerScore(name string) int
}
```

为了我们的 `PlayerServer` 能够使用 `PlayerStore`，它将需要一个引用。现在是时候改变我们的架构了，这样我们的 `PlayerServer` 就变成了一个 `struct`。

```go
type PlayerServer struct {
	store PlayerStore
}
```

最后，我们现在将通过向新结构添加一个方法并放入现有的处理程序代码来实现 `Handler` 接口。

```go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")
	fmt.Fprint(w, p.store.GetPlayerScore(player))
}
```

唯一的另一个变化是我们现在调用 `store.GetPlayerScore`，而不是我们定义的本地函数(现在可以删除)。

下面是完整的代码

```go
//server.go
type PlayerStore interface {
	GetPlayerScore(name string) int
}

type PlayerServer struct {
	store PlayerStore
}

func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")
	fmt.Fprint(w, p.store.GetPlayerScore(player))
}
```

### Fix the issues

这是相当多的变化，我们知道我们的测试和应用程序将不再编译，放松，让编译器通过它。

`./main.go:9:58: type PlayerServer is not an expression`

We need to change our tests to instead create a new instance of our `PlayerServer` and then call its method `ServeHTTP`.

我们需要修改我们的测试，创建 `PlayerServer` 实例，然后调用它的方法 `ServeHTTP`。

```go
//server_test.go
func TestGETPlayers(t *testing.T) {
	server := &PlayerServer{}

	t.Run("returns Pepper's score", func(t *testing.T) {
		request := newGetScoreRequest("Pepper")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertResponseBody(t, response.Body.String(), "20")
	})

	t.Run("returns Floyd's score", func(t *testing.T) {
		request := newGetScoreRequest("Floyd")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertResponseBody(t, response.Body.String(), "10")
	})
}
```

注意，我们现在不需要太担心 store，我们只是希望编译器尽快通过。

你应该养成一个习惯，先编译代码，再通过测试。

在代码还没有编译的时候添加更多的功能(比如存根存储)，我们可能会遇到更多的编译问题。

现在出于同样的原因 `main.go` 也无法编译。

```go
func main() {
	server := &PlayerServer{}
    log.Fatal(http.ListenAndServe(":5000", server))
}
```

最终，编译通过了但是测试失败。

```
=== RUN   TestGETPlayers/returns_the_Pepper's_score
panic: runtime error: invalid memory address or nil pointer dereference [recovered]
    panic: runtime error: invalid memory address or nil pointer dereference
```

这是因为我们没有在 `PlayerStore` 中通过测试。我们需要做一个存根。



```go
//server_test.go
type StubPlayerStore struct {
	scores map[string]int
}

func (s *StubPlayerStore) GetPlayerScore(name string) int {
	score := s.scores[name]
	return score
}
```

`map` 是为我们的测试创建存根键/值存储的一种快速而简单的方法。现在让我们为我们的测试创建一个 store，并将其发送到我们的 `PlayerServer`。

```go
//server_test.go
func TestGETPlayers(t *testing.T) {
	store := StubPlayerStore{
		map[string]int{
			"Pepper": 20,
			"Floyd":  10,
		},
	}
	server := &PlayerServer{&store}

	t.Run("returns Pepper's score", func(t *testing.T) {
		request := newGetScoreRequest("Pepper")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertResponseBody(t, response.Body.String(), "20")
	})

	t.Run("returns Floyd's score", func(t *testing.T) {
		request := newGetScoreRequest("Floyd")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertResponseBody(t, response.Body.String(), "10")
	})
}
```

我们的测试现在通过了，看起来好多了。由于引入了store，我们代码背后的意图现在更加清晰了。我们告诉 reader，因为我们在 `PlayerStore` 有 this data，当你使用它与`PlayerServer`，你应该得到以下响应。

### Run the application

现在我们的测试已经通过了，要完成这个重构，我们需要做的最后一件事就是检查应用程序是否工作正常。程序应该会启动，但如果你试着访问 `http://localhost:5000/players/Pepper` 上点击服务器，你会得到一个可怕的响应。

原因是我们没有传递 `PlayerStore`。

我们需要做一个实现，但现在这是困难的，因为我们没有存储任何有意义的数据，所以它将不得不暂时硬编码。

```go
//main.go
type InMemoryPlayerStore struct{}

func (i *InMemoryPlayerStore) GetPlayerScore(name string) int {
	return 123
}

func main() {
	server := &PlayerServer{&InMemoryPlayerStore{}}
    log.Fatal(http.ListenAndServe(":5000", server))
}
```

如果你再次运行 `go build`，并点击相同的 URL，你应该得到 `“123”`。不是很好，但在我们存储数据之前这是我们能做的最好的了。

关于下一步该做什么，我们有几个选择

-   处理玩家不存在的场景
-   处理 `POST /players/{name}` 场景
-   我们的主应用程序启动了，但实际上没有工作，这让人感觉不太好。我们必须手动测试才能发现问题。

虽然 `POST` 场景让我们更接近“快乐之路”，但我觉得先解决玩家缺失的场景会更容易，因为我们已经处在这种情境中。剩下的我们稍后再讲。

## Write the test first

在我们现有的套件中添加一个缺失的玩家场景

```go
//server_test.go
t.Run("returns 404 on missing players", func(t *testing.T) {
	request := newGetScoreRequest("Apollo")
	response := httptest.NewRecorder()

	server.ServeHTTP(response, request)

	got := response.Code
	want := http.StatusNotFound

	if got != want {
		t.Errorf("got status %d want %d", got, want)
	}
})
```

## Try to run the test

```
=== RUN   TestGETPlayers/returns_404_on_missing_players
    --- FAIL: TestGETPlayers/returns_404_on_missing_players (0.00s)
        server_test.go:56: got status 200 want 404
```

## Write enough code to make it pass

```go
//server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	w.WriteHeader(http.StatusNotFound)

	fmt.Fprint(w, p.store.GetPlayerScore(player))
}
```

有时候，当 TDD 的支持者说“确保你只写了最少的代码就能通过测试”时，我就会翻白眼，因为这听起来很迂弱。

但是这个场景很好地说明了这个示例。我已经做了最低限度(知道它是不正确的)，这是在所有相应中写一个 `StatusNotFound` ，但我们所有的测试都通过了!

**通过最低限度的测试，可以突出测试中的差距**。在我们的例子中，我们并没有断言当玩家在商店中存在时，我们应该获得 `StatusOK`。

更新其他两个测试以断言状态并修复代码。

这是新的测试

```go
//server_test.go
func TestGETPlayers(t *testing.T) {
	store := StubPlayerStore{
		map[string]int{
			"Pepper": 20,
			"Floyd":  10,
		},
	}
	server := &PlayerServer{&store}

	t.Run("returns Pepper's score", func(t *testing.T) {
		request := newGetScoreRequest("Pepper")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusOK)
		assertResponseBody(t, response.Body.String(), "20")
	})

	t.Run("returns Floyd's score", func(t *testing.T) {
		request := newGetScoreRequest("Floyd")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusOK)
		assertResponseBody(t, response.Body.String(), "10")
	})

	t.Run("returns 404 on missing players", func(t *testing.T) {
		request := newGetScoreRequest("Apollo")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusNotFound)
	})
}

func assertStatus(t testing.TB, got, want int) {
	t.Helper()
	if got != want {
		t.Errorf("did not get correct status, got %d, want %d", got, want)
	}
}

func newGetScoreRequest(name string) *http.Request {
	req, _ := http.NewRequest(http.MethodGet, fmt.Sprintf("/players/%s", name), nil)
	return req
}

func assertResponseBody(t testing.TB, got, want string) {
	t.Helper()
	if got != want {
		t.Errorf("response body is wrong, got %q want %q", got, want)
	}
}
```

我们现在在所有的测试中检查状态，所以我做了一个 helper `assertStatus` 来促进这一点。

现在我们的前两个测试失败了，因为 404 而不是 200，所以我们可以修复 `PlayerServer`，只在分数为 0 时返回 not found。

```go
//server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	score := p.store.GetPlayerScore(player)

	if score == 0 {
		w.WriteHeader(http.StatusNotFound)
	}

	fmt.Fprint(w, score)
}
```

### Storing scores

既然我们可以从存储中检索分数，那么存储新分数就有意义了。

## Write the test first

```go
//server_test.go
func TestStoreWins(t *testing.T) {
	store := StubPlayerStore{
		map[string]int{},
	}
	server := &PlayerServer{&store}

	t.Run("it returns accepted on POST", func(t *testing.T) {
		request, _ := http.NewRequest(http.MethodPost, "/players/Pepper", nil)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusAccepted)
	})
}
```

首先，让我们检查一下，如果我们使用 POST 访问特定的路由，我们是否得到了正确的状态代码。这让我们可以抛弃接受不同类型请求和以不同方式处理请求的功能。一旦这个工作完成，我们就可以开始断言处理程序与 store 的交互。

## Try to run the test

```
=== RUN   TestStoreWins/it_returns_accepted_on_POST
    --- FAIL: TestStoreWins/it_returns_accepted_on_POST (0.00s)
        server_test.go:70: did not get correct status, got 404, want 202
```

## Write enough code to make it pass

记住，我们是故意犯错误的，所以一个基于请求方法的 `if` 语句就可以达到目的。

```go
//server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {

	if r.Method == http.MethodPost {
		w.WriteHeader(http.StatusAccepted)
		return
	}

	player := strings.TrimPrefix(r.URL.Path, "/players/")

	score := p.store.GetPlayerScore(player)

	if score == 0 {
		w.WriteHeader(http.StatusNotFound)
	}

	fmt.Fprint(w, score)
}
```

## Refactor

handler 现在看起来有点糊涂了。让我们将代码分解，使其更容易理解，并将不同的功能隔离为新功能。

```go
//server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {

	switch r.Method {
	case http.MethodPost:
		p.processWin(w)
	case http.MethodGet:
		p.showScore(w, r)
	}

}

func (p *PlayerServer) showScore(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	score := p.store.GetPlayerScore(player)

	if score == 0 {
		w.WriteHeader(http.StatusNotFound)
	}

	fmt.Fprint(w, score)
}

func (p *PlayerServer) processWin(w http.ResponseWriter) {
	w.WriteHeader(http.StatusAccepted)
}
```

这使得 `ServeHTTP` 的路由方面更清晰，意味着我们的下一个存储迭代可以只在 `processWin` 里面。

接下来，我们要检查当我们执行 `POST /players/{name}`时，`PlayerStore` 是否被告知记录获胜情况。

## Write the test first

我们可以通过扩展我们的 `StubPlayerStore` 与一个新的 `RecordWin` 方法，然后监视它的调用来实现这一点。

```go
//server_test.go
type StubPlayerStore struct {
	scores   map[string]int
	winCalls []string
}

func (s *StubPlayerStore) GetPlayerScore(name string) int {
	score := s.scores[name]
	return score
}

func (s *StubPlayerStore) RecordWin(name string) {
	s.winCalls = append(s.winCalls, name)
}
```

现在，首先扩展我们的测试，检查调用的数量

```go
//server_test.go
func TestStoreWins(t *testing.T) {
	store := StubPlayerStore{
		map[string]int{},
	}
	server := &PlayerServer{&store}

	t.Run("it records wins when POST", func(t *testing.T) {
		request := newPostWinRequest("Pepper")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusAccepted)

		if len(store.winCalls) != 1 {
			t.Errorf("got %d calls to RecordWin want %d", len(store.winCalls), 1)
		}
	})
}

func newPostWinRequest(name string) *http.Request {
	req, _ := http.NewRequest(http.MethodPost, fmt.Sprintf("/players/%s", name), nil)
	return req
}
```

## Try to run the test

```
./server_test.go:26:20: too few values in struct initializer
./server_test.go:65:20: too few values in struct initializer
```

## Write the minimal amount of code for the test to run and check the failing test output

我们需要更新我们的代码，我们创建了一个 `StubPlayerStore`，因为我们已经添加了一个新的字段

```go
//server_test.go
store := StubPlayerStore{
	map[string]int{},
	nil,
}
```

```
--- FAIL: TestStoreWins (0.00s)
    --- FAIL: TestStoreWins/it_records_wins_when_POST (0.00s)
        server_test.go:80: got 0 calls to RecordWin want 1
```

## Write enough code to make it pass

因为我们只断言调用的数量而不是特定的值，所以初始迭代会稍微小一些。

我们需要更新 `PlayerServer` 的想法什么是 `PlayerStore` 是通过改变接口，如果我们要能够调用 `RecordWin`。



```go
//server.go
type PlayerStore interface {
	GetPlayerScore(name string) int
	RecordWin(name string)
}
```

通过这样做 `main` 不再编译通过

```
./main.go:17:46: cannot use InMemoryPlayerStore literal (type *InMemoryPlayerStore) as type PlayerStore in field value:
    *InMemoryPlayerStore does not implement PlayerStore (missing RecordWin method)
```

编译器告诉我们哪里出错了。让我们更新 `InMemoryPlayerStore` 来拥有那个方法。

```go
//main.go
type InMemoryPlayerStore struct{}

func (i *InMemoryPlayerStore) RecordWin(name string) {}
```

尝试运行测试，我们应该回到编译代码 —— 但测试仍然失败。

现在 `PlayerStore` 中有 `RecordWin` 我们可以在 `PlayerServer` 中调用它

```go
//server.go
func (p *PlayerServer) processWin(w http.ResponseWriter) {
	p.store.RecordWin("Bob")
	w.WriteHeader(http.StatusAccepted)
}
```

运行测试，它应该会通过!显然 `Bob` 并不是我们想要发送给 `RecordWin` 的内容，所以让我们进一步完善测试。

## Write the test first

```go
//server_test.go
t.Run("it records wins on POST", func(t *testing.T) {
	player := "Pepper"

	request := newPostWinRequest(player)
	response := httptest.NewRecorder()

	server.ServeHTTP(response, request)

	assertStatus(t, response.Code, http.StatusAccepted)

	if len(store.winCalls) != 1 {
		t.Fatalf("got %d calls to RecordWin want %d", len(store.winCalls), 1)
	}

	if store.winCalls[0] != player {
		t.Errorf("did not store correct winner got %q want %q", store.winCalls[0], player)
	}
})
```

现在我们知道在我们的 `winCalls` 切片中有一个元素，我们可以安全地引用第一个元素并检查它是否等于 `player`。

## Try to run the test

```
=== RUN   TestStoreWins/it_records_wins_on_POST
    --- FAIL: TestStoreWins/it_records_wins_on_POST (0.00s)
        server_test.go:86: did not store correct winner got 'Bob' want 'Pepper'
```

## Write enough code to make it pass

```go
//server.go
func (p *PlayerServer) processWin(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")
	p.store.RecordWin(player)
	w.WriteHeader(http.StatusAccepted)
}
```

我们修改 `processWin` 使得其接收 `http.Request` 这样我们就可以通过 URL 来提取玩家的名字。一旦我们有了这个，我们就可以用正确的值调用 `store`，使测试通过。

## Refactor

我们可以用相同的方法在两个地方提取玩家名，从而将代码干化

```go
//server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	switch r.Method {
	case http.MethodPost:
		p.processWin(w, player)
	case http.MethodGet:
		p.showScore(w, player)
	}
}

func (p *PlayerServer) showScore(w http.ResponseWriter, player string) {
	score := p.store.GetPlayerScore(player)

	if score == 0 {
		w.WriteHeader(http.StatusNotFound)
	}

	fmt.Fprint(w, score)
}

func (p *PlayerServer) processWin(w http.ResponseWriter, player string) {
	p.store.RecordWin(player)
	w.WriteHeader(http.StatusAccepted)
}
```

即使我们的测试通过了，我们也没有真正的工作软件。如果你尝试着运行 `main` 并按照预期使用软件，那么它便不会奏效，
因为我们还没有找到时间去正确执行 `PlayerStore`。这很好;通过关注处理程序，我们已经确定了需要的接口，而不是试图预先设计它。

我们可以开始围绕我们的 `InMemoryPlayerStore` 编写一些测试，但这只是暂时的，直到我们执行一种更强大的持久化玩家分数的方法(如数据库)。

我们现在要做的是在 `PlayerServer` 和 `InMemoryPlayerStore` 之间编写一个集成测试来完成功能。这将让我们达到确信我们的应用程序正在工作的目标，而不必直接测试 `InMemoryPlayerStore`。不仅如此，当我们使用数据库执行 `PlayerStore` 时，我们还可以使用相同的集成测试来测试该执行。

### Integration tests

集成测试可以用于测试更大范围的系统工作，但您必须记住:

-   它们更难写
-   当它们失败时，很难知道原因(通常是集成测试组件中的一个 bug)，因此很难修复
-   有时它们的运行速度较慢(因为它们经常与“真正的”组件一起使用，如数据库)

出于这个原因，建议你研究一下“测试金字塔”。

## Write the test first

为了简洁起见，我将向您展示最终的重构集成测试。

```go
//server_integration_test.go
func TestRecordingWinsAndRetrievingThem(t *testing.T) {
	store := InMemoryPlayerStore{}
	server := PlayerServer{&store}
	player := "Pepper"

	server.ServeHTTP(httptest.NewRecorder(), newPostWinRequest(player))
	server.ServeHTTP(httptest.NewRecorder(), newPostWinRequest(player))
	server.ServeHTTP(httptest.NewRecorder(), newPostWinRequest(player))

	response := httptest.NewRecorder()
	server.ServeHTTP(response, newGetScoreRequest(player))
	assertStatus(t, response.Code, http.StatusOK)

	assertResponseBody(t, response.Body.String(), "3")
}
```

-   我们正在创建两个试图集成的组件: `InMemoryPlayerStore` 和 `PlayerServer`。
-   然后我们发出 3 次请求，为 `player` 记录 3 次胜利。我们不太关心这个测试中的状态代码，因为它与它们是否集成良好无关。
-   我们所关心的下一个响应(所以我们储存了一个变量 `response`)，因为我们将尝试着获得 `player` 的分数。

## Try to run the test

```
--- FAIL: TestRecordingWinsAndRetrievingThem (0.00s)
    server_integration_test.go:24: response body is wrong, got '123' want '3'
```

## Write enough code to make it pass

在这里，我将采取一些自由的做法，编写更多的代码，而您可能不需要编写测试。

这是允许的！我们仍然有一个测试，检查事情是否正常工作，但它不是围绕我们正在工作的特定单位(`InMemoryPlayerStore`)。

如果我陷入这种情况，我会将更改恢复到失败的测试，然后围绕 `InMemoryPlayerStore` 编写更具体的单元测试，以帮助我找到解决方案。

```go
//in_memory_player_store.go
func NewInMemoryPlayerStore() *InMemoryPlayerStore {
	return &InMemoryPlayerStore{map[string]int{}}
}

type InMemoryPlayerStore struct {
	store map[string]int
}

func (i *InMemoryPlayerStore) RecordWin(name string) {
	i.store[name]++
}

func (i *InMemoryPlayerStore) GetPlayerScore(name string) int {
	return i.store[name]
}
```

-   我们需要存储数据，所以我添加了一个 `map[string]int` 到 `InMemoryPlayerStore` 结构体
-   为了方便起见，我设置了 `NewInMemoryPlayerStore` 来初始化商店，并更新了集成测试来使用它:
    ```go
    //server_integration_test.go
    store := NewInMemoryPlayerStore()
    server := PlayerServer{store}
    ```
-   剩下的代码只是围绕着 `map`

集成测试通过了，现在我们只需要改变 `main` 来使用 `NewInMemoryPlayerStore()`

```go
//main.go
package main

import (
	"log"
	"net/http"
)

func main() {
	server := &PlayerServer{NewInMemoryPlayerStore()}
    log.Fatal(http.ListenAndServe(":5000", server))
}
```

构建它，运行它，然后使用 `curl` 测试它。

-   运行几次，如果你喜欢 `curl -X POST http://localhost:5000/players/Pepper`，请更改 player 名称
-   使用 `curl http://localhost:5000/players/Pepper` 检查得分

太棒了!您已经创建了一个 rest 式服务。要实现这一点，您可能需要选择一个数据存储，以便将分数持久化到比程序运行时间更长的位置。

-   选择一个 store (Bolt? Mongo? Postgres? File system?)
-   使 `PostgresPlayerStore` 实现 `PlayerStore`
-   对功能进行 TDD，这样你就能确保它能工作
-   将它插入到集成测试中，检查它是否仍然正常
-   最后将它插入 `main`

## Refactor

我们快到了!让我们努力防止类似这样的并发错误

```
fatal error: concurrent map read and map write
```

通过添加互斥锁，我们加强了并发安全性，特别是对于 `RecordWin` 函数中的计数器。有关互斥锁的更多信息，请参阅同步章节。

## 总结

### `http.Handler`

-   实现这个接口来创建 web 服务器
-   使用 `http.HandlerFunc` 将普通函数转换为 `http.Handler` 的
-   使用 `httptest.NewRecorder` 作为 `ResponseWriter` 传入，让您监视处理程序发送的响应
-   使用 `http.NewRequest` 来构造您希望进入系统的请求

### Interfaces, Mocking and DI

-   让您可以在更小的块中迭代地构建系统
-   允许您开发需要存储而不需要实际存储的处理程序
-   TDD来驱动您需要的接口

### Commit sins, then refactor (and then commit to source control)

-   您需要将编译失败或测试失败视为需要尽快摆脱的红色情况。
-   只需要编写必要的代码。然后重构代码。
-   在代码没有编译或测试失败的时候尝试做太多的更改，会使问题复杂化。
-   坚持这种方法会迫使您编写小的测试，这意味着小的更改，这有助于保持复杂系统的工作可管理。
