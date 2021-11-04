# JSON, routing & embedding

**[You can find all the code for this chapter here](https://github.com/quii/learn-go-with-tests/tree/main/json)**

[In the previous chapter](http-server.md) we created a web server to store how many games players have won.


我们的产品所有者有一个新的要求;有一个名为 `/league` 的新端点，它返回存储的所有球员的列表。她希望将其作为 JSON 返回。

## Here is the code we have so far

```go
// server.go
package main

import (
	"fmt"
	"net/http"
)

type PlayerStore interface {
	GetPlayerScore(name string) int
	RecordWin(name string)
}

type PlayerServer struct {
	store PlayerStore
}

func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	player := r.URL.Path[len("/players/"):]

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

```go
// InMemoryPlayerStore.go
package main

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

```go
// main.go
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

您可以在本章顶部的链接中找到相应的测试。

我们将从联赛积分榜的 endpoint 开始。

## Write the test first

我们将扩展现有的套件，因为我们有一些有用的测试功能和一个假的 `PlayerStore` 来使用。


```go
//server_test.go
func TestLeague(t *testing.T) {
	store := StubPlayerStore{}
	server := &PlayerServer{&store}

	t.Run("it returns 200 on /league", func(t *testing.T) {
		request, _ := http.NewRequest(http.MethodGet, "/league", nil)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusOK)
	})
}
```

在担心实际分数和 JSON 之前，我们将尝试保持计划的小变化，以迭代达到我们的目标。
最简单的开始便是检查我们是否能够点击 `/league` 并获得 `OK`。

## Try to run the test

```
=== RUN   TestLeague/it_returns_200_on_/league
panic: runtime error: slice bounds out of range [recovered]
    panic: runtime error: slice bounds out of range

goroutine 6 [running]:
testing.tRunner.func1(0xc42010c3c0)
    /usr/local/Cellar/go/1.10/libexec/src/testing/testing.go:742 +0x29d
panic(0x1274d60, 0x1438240)
    /usr/local/Cellar/go/1.10/libexec/src/runtime/panic.go:505 +0x229
github.com/quii/learn-go-with-tests/json-and-io/v2.(*PlayerServer).ServeHTTP(0xc420048d30, 0x12fc1c0, 0xc420010940, 0xc420116000)
    /Users/quii/go/src/github.com/quii/learn-go-with-tests/json-and-io/v2/server.go:20 +0xec
```

你的 `PlayerServer` 应该像这样 panic。转到堆栈跟踪中指向 `server.go` 的那行代码。



```go
player := r.URL.Path[len("/players/"):]
```

在前一章中，我们提到过这是一种非常简单的路由方式。
它试图分割路径的字符串从 `/league` `以外的索引开始所以它是 `slice bounds out of range`。

## Write enough code to make it pass

Go 有一个内置的路由机制叫 [`ServeMux`](https://golang.org/pkg/net/http/#ServeMux)(请求多路复用器)，它允许你附加 `http.Handler` 到特定的请求路径。

让我们犯一些错误，以最快的方式通过测试，知道一旦我们知道测试通过了，我们就可以安全地重构它。

```go
//server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {

	router := http.NewServeMux()

	router.Handle("/league", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	}))

	router.Handle("/players/", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		player := r.URL.Path[len("/players/"):]

		switch r.Method {
		case http.MethodPost:
			p.processWin(w, player)
		case http.MethodGet:
			p.showScore(w, player)
		}
	}))

	router.ServeHTTP(w, r)
}
```

- 当请求开始时，我们创建一个路由器，然后我们告诉它对于 `x` 路径使用 `y` 处理器。
- 对于新端点，我们使用  `http.HandlerFunc` 和匿名函数来 `w.WriteHeader(http.StatusOK)` 当 `/league` 被请求使我们的新测试通过。
- 对于 `/players/` 路径，我们只是剪切并粘贴我们的代码到另一个 `http.HandlerFunc`。
- 最后，我们通过调用新路由器的 `ServeHTTP` 来处理请求(注意 `ServeMux` 如何也是一个 `http.Handler` ?)

测试现在应该能通过了。

## Refactor

`ServerHttp` 现在看起来有点大，我们可以通过将处理程序重构为单独的方法来将事情分开。

```go
//server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {

	router := http.NewServeMux()
	router.Handle("/league", http.HandlerFunc(p.leagueHandler))
	router.Handle("/players/", http.HandlerFunc(p.playersHandler))

	router.ServeHTTP(w, r)
}

func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
}

func (p *PlayerServer) playersHandler(w http.ResponseWriter, r *http.Request) {
	player := r.URL.Path[len("/players/"):]

	switch r.Method {
	case http.MethodPost:
		p.processWin(w, player)
	case http.MethodGet:
		p.showScore(w, player)
	}
}
```

当请求传入时设置一个路由器，然后调用它，这很奇怪(而且效率低)。理想情况下，我们想要做的是有某种 `NewPlayerServer` 函数，它会取走我们的依赖并做创建路由器的一次性设置。然后，每个请求就可以只使用路由器的一个实例。



```go
//server.go
type PlayerServer struct {
	store  PlayerStore
	router *http.ServeMux
}

func NewPlayerServer(store PlayerStore) *PlayerServer {
	p := &PlayerServer{
		store,
		http.NewServeMux(),
	}

	p.router.Handle("/league", http.HandlerFunc(p.leagueHandler))
	p.router.Handle("/players/", http.HandlerFunc(p.playersHandler))

	return p
}

func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	p.router.ServeHTTP(w, r)
}
```

- `PlayerServer` 现在需要一个 router
- 我们已经将路由创建从 `ServeHTTP` 移到了 `NewPlayerServer`，所以这只需要做一次，而不是每个请求。
- 你将需要更新所有的测试和生产代码，我们过去用 `PlayerServer{&store}` 的改为用 `NewPlayerServer(&store)`。

### One final refactor

修改代码如下。

```go
type PlayerServer struct {
	store PlayerStore
	http.Handler
}

func NewPlayerServer(store PlayerStore) *PlayerServer {
	p := new(PlayerServer)

	p.store = store

	router := http.NewServeMux()
	router.Handle("/league", http.HandlerFunc(p.leagueHandler))
	router.Handle("/players/", http.HandlerFunc(p.playersHandler))

	p.Handler = router

	return p
}
```

在 `server_test.go`, `server_integration_test.go`, 和 `main.go` 中用 `server := NewPlayerServer(&store)` 替代 `server := &PlayerServer{&store}`。

最后，确保你**删除**  `func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request)` 因为它不再需要!

## Embedding

我们修改了 `PlayerServer` 的第二个属性，删除了 `router http.ServeMux` 的命名属性将其替换为 `http.Handler` 这叫做嵌入。

> Go没有提供典型的、类型驱动的子类概念，但它确实能够通过在结构或接口中嵌入类型来“借用”实现的各个部分。

[Effective Go - Embedding](https://golang.org/doc/effective_go.html#embedding)

这意味着我们的 `PlayerServer` 现在有 `http.Handler` 的所有方法，也就是 `ServeHTTP`。

To "fill in" the `http.Handler` we assign it to the `router` we create in `NewPlayerServer`.
我们能这样做是因为 `http.ServerMux` 有 `ServeHTTP` 方法。

这让我们可以删除自己的 `ServeHTTP` 方法，因为我们已经通过嵌入式类型公开了一个方法。

嵌入是一个非常有趣的语言特性。您可以将它与接口一起使用，以组成新的接口。

```go
type Animal interface {
	Eater
	Sleeper
}
```

您也可以将它用于具体类型，而不仅仅是接口。正如您所期望的那样，如果您嵌入了一个具体类型，您就可以访问它的所有公共方法和字段。

### Any downsides?

您必须小心嵌入类型，因为您将公开所嵌入类型的所有公共方法和字段。在我们的例子中，这是可以的，因为我们只嵌入了我们想要公开的 _interface_ (`http.Handler`)。

如果我们懒惰地嵌入了 `http.ServeMux` 代替(具体类型)它将仍然工作但是用户 `PlayerServer` 将能够添加新的路由到我们的服务器，因为 `Handle(path, handler)` 将是公共的。

**当嵌入类型时，真的要考虑它对你的公共 API 有什么影响**

滥用嵌入是一个非常常见的错误，最终会污染 api 并暴露类型的内部。

现在我们已经重新构造了我们的应用程序，我们可以很容易地添加新的路由，并有了 `/league` 端点的开始。现在我们需要让它返回一些有用的信息。

我们应该返回类似这样的 JSON。

```json
[
   {
      "Name":"Bill",
      "Wins":10
   },
   {
      "Name":"Alice",
      "Wins":15
   }
]
```

## Write the test first

首先，我们将尝试将响应解析为有意义的内容。

```go
//server_test.go
func TestLeague(t *testing.T) {
	store := StubPlayerStore{}
	server := NewPlayerServer(&store)

	t.Run("it returns 200 on /league", func(t *testing.T) {
		request, _ := http.NewRequest(http.MethodGet, "/league", nil)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		var got []Player

		err := json.NewDecoder(response.Body).Decode(&got)

		if err != nil {
			t.Fatalf("Unable to parse response from server %q into slice of Player, '%v'", response.Body, err)
		}

		assertStatus(t, response.Code, http.StatusOK)
	})
}
```

### Why not test the JSON string?

您可以认为，更简单的初始步骤是断言响应主体具有特定的 JSON 字符串。

根据我的经验，断言 JSON 字符串的测试有以下问题。

- *脆弱性*. 如果更改数据模型，测试将失败。
- *难以调试*. 在比较两个 JSON 字符串时，理解实际问题是很棘手的。
- *Poor intention*. 虽然输出应该是 JSON，但真正重要的是数据是什么，而不是它是如何编码的。
- *Re-testing the standard library*. 不需要测试标准库如何输出 JSON，它已经被测试过了。不要测试别人的代码。

相反，我们应该将 JSON 解析为与测试相关的数据结构。

### Data modelling

考虑到 JSON 数据模型，我们似乎需要一个带有一些字段的 `Player` 数组，所以我们创建了一个新类型来捕获它。

```go
//server.go
type Player struct {
	Name string
	Wins int
}
```

### JSON decoding

```go
//server_test.go
var got []Player
err := json.NewDecoder(response.Body).Decode(&got)
```

为了将 JSON 解析到我们的数据模型中，我们从 `encoding/json` 包中创建了一个 `Decoder`，然后调用它的 `Decode` 方法。
要创建一个 `Decoder` 它需要一个 `io.Reader` 从哪里读取。

`Decode` 获取我们试图解码的东西的地址，这就是为什么我们在前面声明一个空的 `Player` 切片。

解析 JSON 可能会失败，所以 `Decode` 会返回一个 `error`。如果失败了，继续测试就没有意义了，所以我们检查错误并使用 `t.Fatalf` 停止测试。如果真的发生的话。请注意，我们打印了伴随错误的响应体，因为这对运行测试的人来说很重要，以查看哪些字符串不能被解析。



## Try to run the test

```
=== RUN   TestLeague/it_returns_200_on_/league
    --- FAIL: TestLeague/it_returns_200_on_/league (0.00s)
        server_test.go:107: Unable to parse response from server '' into slice of Player, 'unexpected end of JSON input'
```

我们的端点目前没有返回一个 body，所以它不能被解析成 JSON。

## Write enough code to make it pass

```go
//server.go
func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	leagueTable := []Player{
		{"Chris", 20},
	}

	json.NewEncoder(w).Encode(leagueTable)

	w.WriteHeader(http.StatusOK)
}
```

现在测试通过了。

### Encoding and Decoding

注意标准库中可爱的对称。

- 要创建一个 `Encoder`，你需要一个 `io.Writer`，`http.ResponseWriter` 实现了它。
- 要创建一个 `Decoder`，你需要一个 `io.Reader`。

## Refactor

在我们的处理程序和获取 `leagueTable` 之间引入一个关注点分离是很好的，因为我们知道我们不会很快进行硬编码。

```go
//server.go
func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	json.NewEncoder(w).Encode(p.getLeagueTable())
	w.WriteHeader(http.StatusOK)
}

func (p *PlayerServer) getLeagueTable() []Player {
	return []Player{
		{"Chris", 20},
	}
}
```

接下来，我们将扩展测试，以便能够精确地控制想要返回的数据。

## Write the test first

我们可以更新测试，以确定 league 表中包含一些我们将在 store stub 的玩家。

更新 `StubPlayerStore`，让它存储一个 league，这只是 `Player` 的一部分。我们会把我们想要的数据存储在那里。

```go
//server_test.go
type StubPlayerStore struct {
	scores   map[string]int
	winCalls []string
	league   []Player
}
```

接下来，通过将一些玩家放到我们存根的 league 属性中，并断言他们从我们的服务器返回来更新我们当前的测试。

```go
//server_test.go
func TestLeague(t *testing.T) {

	t.Run("it returns the league table as JSON", func(t *testing.T) {
		wantedLeague := []Player{
			{"Cleo", 32},
			{"Chris", 20},
			{"Tiest", 14},
		}

		store := StubPlayerStore{nil, nil, wantedLeague}
		server := NewPlayerServer(&store)

		request, _ := http.NewRequest(http.MethodGet, "/league", nil)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		var got []Player

		err := json.NewDecoder(response.Body).Decode(&got)

		if err != nil {
			t.Fatalf("Unable to parse response from server %q into slice of Player, '%v'", response.Body, err)
		}

		assertStatus(t, response.Code, http.StatusOK)

		if !reflect.DeepEqual(got, wantedLeague) {
			t.Errorf("got %v want %v", got, wantedLeague)
		}
	})
}
```

## Try to run the test

```
./server_test.go:33:3: too few values in struct initializer
./server_test.go:70:3: too few values in struct initializer
```

## Write the minimal amount of code for the test to run and check the failing test output

你需要更新其他测试，因为我们在 `StubPlayerStore` 中有一个新字段;在其他测试中设置为 nil。

再次运行测试

```
=== RUN   TestLeague/it_returns_the_league_table_as_JSON
    --- FAIL: TestLeague/it_returns_the_league_table_as_JSON (0.00s)
        server_test.go:124: got [{Chris 20}] want [{Cleo 32} {Chris 20} {Tiest 14}]
```

## Write enough code to make it pass

我们知道数据在我们的 `StubPlayerStore` 中，我们已经把它抽象到一个接口 `PlayerStore` 中。
我们需要更新它，这样任何传递给我们 `PlayerStore` 的都可以为我们提供联赛的数据。

```go
//server.go
type PlayerStore interface {
	GetPlayerScore(name string) int
	RecordWin(name string)
	GetLeague() []Player
}
```

现在我们可以更新处理程序代码来调用它，而不是返回一个硬编码的列表。删除我们的方法 `getLeagueTable()`，然后更新 `leagueHandler` 来调用 `GetLeague()`。

```go
//server.go
func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	json.NewEncoder(w).Encode(p.store.GetLeague())
	w.WriteHeader(http.StatusOK)
}
```

运行测试。

```
# github.com/quii/learn-go-with-tests/json-and-io/v4
./main.go:9:50: cannot use NewInMemoryPlayerStore() (type *InMemoryPlayerStore) as type PlayerStore in argument to NewPlayerServer:
    *InMemoryPlayerStore does not implement PlayerStore (missing GetLeague method)
./server_integration_test.go:11:27: cannot use store (type *InMemoryPlayerStore) as type PlayerStore in argument to NewPlayerServer:
    *InMemoryPlayerStore does not implement PlayerStore (missing GetLeague method)
./server_test.go:36:28: cannot use &store (type *StubPlayerStore) as type PlayerStore in argument to NewPlayerServer:
    *StubPlayerStore does not implement PlayerStore (missing GetLeague method)
./server_test.go:74:28: cannot use &store (type *StubPlayerStore) as type PlayerStore in argument to NewPlayerServer:
    *StubPlayerStore does not implement PlayerStore (missing GetLeague method)
./server_test.go:106:29: cannot use &store (type *StubPlayerStore) as type PlayerStore in argument to NewPlayerServer:
    *StubPlayerStore does not implement PlayerStore (missing GetLeague method)
```

编译器抱怨，因为 `InMemoryPlayerStore` 和 `StubPlayerStore` 没有我们添加到接口的新方法。

对于 `StubPlayerStore` 很简单, 只需返回我们之前添加的 `league` 字段。

```go
//server_test.go
func (s *StubPlayerStore) GetLeague() []Player {
	return s.league
}
```

下面是关于 `InMemoryStore` 是如何实现的提示。

```go
//in_memory_player_store.go
type InMemoryPlayerStore struct {
	store map[string]int
}
```

虽然通过迭代 map “正确”执行 `GetLeague` 非常简单，但请记住，我们只是试图编写最少的代码以使测试通过。

所以现在让编译器高兴一下，在我们的 `InMemoryStore` 中接受一个不完整实现的不舒服感觉。

```go
//in_memory_player_store.go
func (i *InMemoryPlayerStore) GetLeague() []Player {
	return nil
}
```

它真正告诉我们的是，稍后我们将对它进行测试，但我们先把它搁置一边。

尝试并运行测试，编译器应该通过，测试也应该通过!

## Refactor

测试代码不能很好地表达意图，并且有很多我们可以重构的样板。

```go
//server_test.go
t.Run("it returns the league table as JSON", func(t *testing.T) {
	wantedLeague := []Player{
		{"Cleo", 32},
		{"Chris", 20},
		{"Tiest", 14},
	}

	store := StubPlayerStore{nil, nil, wantedLeague}
	server := NewPlayerServer(&store)

	request := newLeagueRequest()
	response := httptest.NewRecorder()

	server.ServeHTTP(response, request)

	got := getLeagueFromResponse(t, response.Body)
	assertStatus(t, response.Code, http.StatusOK)
	assertLeague(t, got, wantedLeague)
})
```

下面是一些新的 helpers

```go
//server_test.go
func getLeagueFromResponse(t testing.TB, body io.Reader) (league []Player) {
	t.Helper()
	err := json.NewDecoder(body).Decode(&league)

	if err != nil {
		t.Fatalf("Unable to parse response from server %q into slice of Player, '%v'", body, err)
	}

	return
}

func assertLeague(t testing.TB, got, want []Player) {
	t.Helper()
	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %v want %v", got, want)
	}
}

func newLeagueRequest() *http.Request {
	req, _ := http.NewRequest(http.MethodGet, "/league", nil)
	return req
}
```

为了让服务器正常工作，我们需要做的最后一件事是确保在响应中返回一个 `content-type` 报头，以便机器能够识别我们返回的是 `JSON`。

## Write the test first

将此断言添加到现有测试中

```go
//server_test.go
if response.Result().Header.Get("content-type") != "application/json" {
	t.Errorf("response did not have content-type of application/json, got %v", response.Result().Header)
}
```

## Try to run the test

```
=== RUN   TestLeague/it_returns_the_league_table_as_JSON
    --- FAIL: TestLeague/it_returns_the_league_table_as_JSON (0.00s)
        server_test.go:124: response did not have content-type of application/json, got map[Content-Type:[text/plain; charset=utf-8]]
```

## Write enough code to make it pass

更新 `leagueHandler`

```go
//server.go
func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("content-type", "application/json")
	json.NewEncoder(w).Encode(p.store.GetLeague())
}
```

测试应该能通过了。

## Refactor

为 `application/json` 创建一个常量，并在 `leagueHandler` 里面使用

```go
//server.go
const jsonContentType = "application/json"

func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("content-type", jsonContentType)
	json.NewEncoder(w).Encode(p.store.GetLeague())
}
```

为 `assertContentType` 添加一个 helper。

```go
//server_test.go
func assertContentType(t testing.TB, response *httptest.ResponseRecorder, want string) {
	t.Helper()
	if response.Result().Header.Get("content-type") != want {
		t.Errorf("response did not have content-type of %s, got %v", want, response.Result().Header)
	}
}
```

在测试中使用它。

```go
//server_test.go
assertContentType(t, response, jsonContentType)
```

现在我们已经解决了 `PlayerServer`，现在我们可以将注意力转向 `InMemoryPlayerStore`，因为现在如果我们试图向产品所有者演示这个，`/league` 将不起作用。

对于我们来说，获得一些信心的最快方法是添加到我们的集成测试中，我们可以点击新的端点，并检查我们从 `/league` 得到正确的响应。

## Write the test first

我们可以用 `t.Run` 来分解这个测试，我们可以重用我们的服务器测试的助手 —— 再次显示了重构测试的重要性。

```go
//server_integration_test.go
func TestRecordingWinsAndRetrievingThem(t *testing.T) {
	store := NewInMemoryPlayerStore()
	server := NewPlayerServer(store)
	player := "Pepper"

	server.ServeHTTP(httptest.NewRecorder(), newPostWinRequest(player))
	server.ServeHTTP(httptest.NewRecorder(), newPostWinRequest(player))
	server.ServeHTTP(httptest.NewRecorder(), newPostWinRequest(player))

	t.Run("get score", func(t *testing.T) {
		response := httptest.NewRecorder()
		server.ServeHTTP(response, newGetScoreRequest(player))
		assertStatus(t, response.Code, http.StatusOK)

		assertResponseBody(t, response.Body.String(), "3")
	})

	t.Run("get league", func(t *testing.T) {
		response := httptest.NewRecorder()
		server.ServeHTTP(response, newLeagueRequest())
		assertStatus(t, response.Code, http.StatusOK)

		got := getLeagueFromResponse(t, response.Body)
		want := []Player{
			{"Pepper", 3},
		}
		assertLeague(t, got, want)
	})
}
```

## Try to run the test

```
=== RUN   TestRecordingWinsAndRetrievingThem/get_league
    --- FAIL: TestRecordingWinsAndRetrievingThem/get_league (0.00s)
        server_integration_test.go:35: got [] want [{Pepper 3}]
```

## Write enough code to make it pass

`InMemoryPlayerStore` 会在你调用 `GetLeague()` 时返回 `nil` 所以我们需要修正这个。

```go
//in_memory_player_store.go
func (i *InMemoryPlayerStore) GetLeague() []Player {
	var league []Player
	for name, wins := range i.store {
		league = append(league, Player{name, wins})
	}
	return league
}
```

我们所需要做的就是遍历 map 并将每个键/值转换为 `Player`。

测试现在应该能通过了。

## Wrapping up

我们继续使用 TDD 安全地迭代我们的程序，使它以一种可维护的方式通过路由器支持新的端点，现在它可以为我们的消费者返回 JSON。在下一章中，我们将介绍数据持久化和联赛排序。


What we've covered:

- **Routing**. 标准库为您提供了一种易于使用的类型来进行路由。它完全包含了 `http.Handler` 接口，你把路由分配给 `Handler`，路由器本身也是一个 `Handler`。它没有一些你可能期望的特性，比如路径变量(例如 `/users/{id}`)。您可以自己轻松地解析这些信息，但如果它成为负担，您可能需要考虑查看其他路由库。大多数流行的程序都坚持标准库的理念，即同时实现`http.Handler`。
- **Type embedding**. 我们接触了一些关于类型嵌套的技术，你可以[从这看到关于这个更多的介绍](https://golang.org/doc/effective_go.html#embedding)，如果有一件事你应该从中学到，那就是它可能非常有用，但要始终考虑你的公共 API，只暴露适当的。
- **JSON deserializing and serializing**. 标准库使得序列化和反序列化数据变得非常简单。它还可以进行配置，如果需要，您可以定制这些数据转换的工作方式。

