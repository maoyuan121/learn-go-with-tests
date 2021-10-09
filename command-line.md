# Command line and project structure

**[You can find all the code for this chapter here](https://github.com/quii/learn-go-with-tests/tree/main/command-line)**

我们的产品所有者现在想通过引入第二个应用程序 —— 一个命令行应用程序来进行数据透视。

现在，它只需要在用户输入 `Ruth wins` 时记录玩家的胜利。其目的是最终成为一种帮助用户玩扑克的工具。

产品所有者希望两个应用程序共享数据库，以便联赛根据新应用程序记录的胜利进行更新。

## A reminder of the code

我们有一个应用查询其中的 `main.go` 文件启动一个 HTTP 服务器。在本练习中，HTTP 服务器不会引起我们的兴趣，但它使用的抽象会引起我们的兴趣。它依赖于 `PlayerStore`。

```go
type PlayerStore interface {
	GetPlayerScore(name string) int
	RecordWin(name string)
	GetLeague() League
}
```

上一个章节中，我们创建了一个实现了这个接口的 `FileSystemPlayerStore`。我们应该能够在我们的新应用程序中重用其中的一些内容。

## Some project refactoring first

我们的项目现在需要创建两个二进制，一个是已经存在的 web server 另一个是命令行应用。

在我们陷入新的工作之前，我们应该构建我们的项目来适应这一点。

到目前为止所有的代码都在同一个文件夹下面，路径类似于

`$GOPATH/src/github.com/your-name/my-app`

为了让你在 Go 中创建一个应用程序，你需要在 `package main` 中包含一个 `main` 函数。到目前为止，我们所有的“域”代码都在 `package main` 中，而我们的 `func main` 可以引用所有内容。

到目前为止，这是一个很好的实践，不要过度使用包结构。如果你花点时间浏览一下标准库，你会发现其中很少有文件夹和结构。

幸运的是，在需要的时候添加结构是非常简单的。

在现有的项目中创建一个 `cmd` 目录，其中有一个 `webserver` 目录(例如 `mkdir -p cmd/webserver`)。

将 `main.go` 移动到此。

如果你已经安装了 `tree`，你应该运行它，你的结构应该是这样的


```
.
├── file_system_store.go
├── file_system_store_test.go
├── cmd
│   └── webserver
│       └── main.go
├── league.go
├── server.go
├── server_integration_test.go
├── server_test.go
├── tape.go
└── tape_test.go
```

我们现在有效地将应用程序和库代码分离了，但是我们现在需要更改一些包名。记住，当你构建一个 Go 应用程序时，它的包必须是 `main`。

更改所有其他代码，以拥有一个名为 `poker` 的包。

最后，我们需要将这个包导入 `main.go`。所以我们可以用它来创建我们的 web 服务器。然后我们可以使用 `poker.FunctionName` 来使用库代码。

路径在你的电脑上是不同的，但它应该是类似的:

```go
//cmd/webserver/main.go
package main

import (
	"github.com/quii/learn-go-with-tests/command-line/v1"
	"log"
	"net/http"
	"os"
)

const dbFileName = "game.db.json"

func main() {
	db, err := os.OpenFile(dbFileName, os.O_RDWR|os.O_CREATE, 0666)

	if err != nil {
		log.Fatalf("problem opening %s %v", dbFileName, err)
	}

	store, err := poker.NewFileSystemPlayerStore(db)

	if err != nil {
		log.Fatalf("problem creating file system player store, %v ", err)
	}

	server := poker.NewPlayerServer(store)

    log.Fatal(http.ListenAndServe(":5000", server))
}
```

完整的路径可能看起来有点不协调，但这就是将任何公共可用库导入到代码中的方法。

通过将我们的域代码分离成一个单独的包，并将其提交给像GitHub这样的公共仓储平台，任何 Go 开发者都可以编写自己的代码，并将我们编写的功能导入包中。你第一次尝试运行它会抱怨它不存在，但你需要做的就是运行 `go get`。

另外，用户可以查看 [the documentation at godoc.org](https://godoc.org/github.com/quii/learn-go-with-tests/command-line/v1)。

### Final checks

- 在根目录运行 `go test` 检查是否还能通过测试
- 转至 `cmd/webserver` 运行 `go run main.go`
  - 浏览 `http://localhost:5000/league` 你应该能看到工作正常

### Walking skeleton

在我们陷入编写测试之前，让我们添加一个项目将要构建的新应用程序。在 `cmd` 中创建另一个名为 `cli` (命令行界面)的目录，并添加一个 `main.go`。内容如下

```go
//cmd/cli/main.go
package main

import "fmt"

func main() {
	fmt.Println("Let's play poker")
}
```

我们要处理的第一个要求是当用户输入 `{PlayerName} wins` 时记录胜利。

## Write the test first

我们知道我们需要创建一个叫做 `CLI` 的东西，它将允许我们 `Play` 扑克。它需要读取用户输入，然后将胜利记录到 `PlayerStore`。

在我们进一步深入之前，让我们先编写一个测试，检查它是否如我们所愿与 `PlayerStore` 整合在一起。

在 `CLI_test.go` (在项目的根目录，不在 `cmd` 中)

```go
//CLI_test.go
package poker

import "testing"

func TestCLI(t *testing.T) {
	playerStore := &StubPlayerStore{}
	cli := &CLI{playerStore}
	cli.PlayPoker()

	if len(playerStore.winCalls) != 1 {
		t.Fatal("expected a win call but didn't get any")
	}
}
```

- 我们可以从其它的测试中使用我们的 `StubPlayerStore`
- 我们将依赖项传递给还不存在的 `CLI` 类型
- 通过还没写的 `PlayPoker` 方法触发游戏
- 检查赢家已经被记录下来了

## 运行测试

```
# github.com/quii/learn-go-with-tests/command-line/v2
./cli_test.go:25:10: undefined: CLI
```

## 为要运行的测试编写最小数量的代码，并检查失败的测试输出

现在，您应该可以使用各自的字段创建新的 `CLI` 结构，并添加一个方法了。

您应该以这样的代码结束

```go
//CLI.go
package poker

type CLI struct {
	playerStore PlayerStore
}

func (cli *CLI) PlayPoker() {}
```

记住，我们只是试图让测试运行，所以我们可以按照我们希望的方式检查测试失败



```
--- FAIL: TestCLI (0.00s)
    cli_test.go:30: expected a win call but didn't get any
FAIL
```

## Write enough code to make it pass

```go
//CLI.go
func (cli *CLI) PlayPoker() {
	cli.playerStore.RecordWin("Cleo")
}
```

测试应该能通过。

接下来，我们需要模拟从 `Stdin` (来自用户的输入)读取数据，这样我们就可以记录特定玩家的胜利。

让我们扩展测试来验证这一点。

## Write the test first

```go
//CLI_test.go
func TestCLI(t *testing.T) {
	in := strings.NewReader("Chris wins\n")
	playerStore := &StubPlayerStore{}

	cli := &CLI{playerStore, in}
	cli.PlayPoker()

	if len(playerStore.winCalls) < 1 {
		t.Fatal("expected a win call but didn't get any")
	}

	got := playerStore.winCalls[0]
	want := "Chris"

	if got != want {
		t.Errorf("didn't record correct winner, got %q, want %q", got, want)
	}
}
```                                    

在我们的测试中我们使用 `strings.NewReader` 创建一个 `io.Reader`，用我们希望用户输入的内容填充它。

## Try to run the test

`./CLI_test.go:12:32: too many values in struct initializer`

## Write the minimal amount of code for the test to run and check the failing test output

我们需要添加新的依赖的 `CLI` 中。

```go
//CLI.go
type CLI struct {
	playerStore PlayerStore
	in          io.Reader
}
```

## Write enough code to make it pass

```
--- FAIL: TestCLI (0.00s)
    CLI_test.go:23: didn't record the correct winner, got 'Cleo', want 'Chris'
FAIL
```

记住先做最简单的事情

```go
func (cli *CLI) PlayPoker() {
	cli.playerStore.RecordWin("Chris")
}
```

测试通过。接下来，我们将添加另一个测试，迫使我们编写一些真正的代码，但首先，让我们进行重构。

## Refactor

在 `server_test` 中，我们之前检查了获胜的记录是否与这里一样。让我们将断言 DRY 到 helper 中



```go
//server_test.go
func assertPlayerWin(t testing.TB, store *StubPlayerStore, winner string) {
	t.Helper()

	if len(store.winCalls) != 1 {
		t.Fatalf("got %d calls to RecordWin want %d", len(store.winCalls), 1)
	}

	if store.winCalls[0] != winner {
		t.Errorf("did not store correct winner got %q want %q", store.winCalls[0], winner)
	}
}
```

现在替换这个断言到 `server_test.go` 和 `CLI_test.go` 中。

现在测试应该如下

```go
//CLI_test.go
func TestCLI(t *testing.T) {
	in := strings.NewReader("Chris wins\n")
	playerStore := &StubPlayerStore{}

	cli := &CLI{playerStore, in}
	cli.PlayPoker()

	assertPlayerWin(t, playerStore, "Chris")
}
```

现在让我们用不同的用户输入编写另一个测试，迫使我们实际读取它。

## Write the test first

```go
//CLI_test.go
func TestCLI(t *testing.T) {

	t.Run("record chris win from user input", func(t *testing.T) {
		in := strings.NewReader("Chris wins\n")
		playerStore := &StubPlayerStore{}

		cli := &CLI{playerStore, in}
		cli.PlayPoker()

		assertPlayerWin(t, playerStore, "Chris")
	})

	t.Run("record cleo win from user input", func(t *testing.T) {
		in := strings.NewReader("Cleo wins\n")
		playerStore := &StubPlayerStore{}

		cli := &CLI{playerStore, in}
		cli.PlayPoker()

		assertPlayerWin(t, playerStore, "Cleo")
	})

}
```

## Try to run the test

```
=== RUN   TestCLI
--- FAIL: TestCLI (0.00s)
=== RUN   TestCLI/record_chris_win_from_user_input
    --- PASS: TestCLI/record_chris_win_from_user_input (0.00s)
=== RUN   TestCLI/record_cleo_win_from_user_input
    --- FAIL: TestCLI/record_cleo_win_from_user_input (0.00s)
        CLI_test.go:27: did not store correct winner got 'Chris' want 'Cleo'
FAIL
```

## Write enough code to make it pass

我们将使用一个 [`bufio.Scanner`](https://golang.org/pkg/bufio/) 从 `io.Reader` 中读取输入。

> 包 bufio 实现缓冲 I/O。它封装了一个 io.Reader或 io.Writer 对象，创建另一个对象(Reader 或 Writer)，该对象也实现了接口，但为文本 I/O 提供了缓冲和一些帮助。

更新代码如下

```go
//CLI.go
type CLI struct {
	playerStore PlayerStore
	in          io.Reader
}

func (cli *CLI) PlayPoker() {
	reader := bufio.NewScanner(cli.in)
	reader.Scan()
	cli.playerStore.RecordWin(extractWinner(reader.Text()))
}

func extractWinner(userInput string) string {
	return strings.Replace(userInput, " wins", "", 1)
}
```

测试现在应该能通过了。

- `Scanner.Scan()` 将一直读到换行符。
- 然后我们使用 `Scanner.Text()` 返回 scanner 读取的 `string`。
 
现在我们已经通过了一些测试，我们应该将其连接到 `main`。记住，我们应该尽可能快地努力拥有完全集成的工作软件。

在 `main.go` 中添加下面的代码，然后运行。（您可能需要调整第二个依赖项的路径以匹配您的计算机上的内容）
                          

```go
package main

import (
	"fmt"
	"github.com/quii/learn-go-with-tests/command-line/v3"
	"log"
	"os"
)

const dbFileName = "game.db.json"

func main() {
	fmt.Println("Let's play poker")
	fmt.Println("Type {Name} wins to record a win")

	db, err := os.OpenFile(dbFileName, os.O_RDWR|os.O_CREATE, 0666)

	if err != nil {
		log.Fatalf("problem opening %s %v", dbFileName, err)
	}

	store, err := poker.NewFileSystemPlayerStore(db)

	if err != nil {
		log.Fatalf("problem creating file system player store, %v ", err)
	}

	game := poker.CLI{store, os.Stdin}
	game.PlayPoker()
}
```

应该出现错误

```
command-line/v3/cmd/cli/main.go:32:25: implicit assignment of unexported field 'playerStore' in poker.CLI literal
command-line/v3/cmd/cli/main.go:32:34: implicit assignment of unexported field 'in' in poker.CLI literal
```

这里发生了什么？因为我们尝试给 `CLI` 的 `playStore` 和 `in` 赋值。这两个西端是未导出的字段（私有字段）。
我们可以在测试代码中这样做，因为我们的测试和 `CLI`(`poker`) 在一个包中。
但是我们的 `main` 在 `main` 包中，所以他没有访问权限。

这突出了整合你的工作的重要性。
我们正确地将 `CLI` 的依赖项设置为私有（因为我们不想让它们暴露给 `CLI` 的用户），但还没有为用户构造它创造一种方法。

有没有办法早点发现这个问题?

### `package mypackage_test`

在到目前为止的所有其他示例中，当我们创建一个测试文件时，我们将它声明为我们正在测试的同一个包中。

这很好，这意味着在我们想要测试包内部的某些内容时，我们可以访问未导出的类型。

But given we have advocated for _not_ testing internal things _generally_, can Go help enforce that? What if we could test our code where we only have access to the exported types (like our `main` does)?

但是考虑到我们一直提倡不要测试内部的东西，golang 能帮助执行吗？如果我们可以测试我们的代码，我们只能访问导出的类型(像我们的 `main` 做的那样)?

当你在编写一个包含多个包的项目时，我强烈建议你的测试包名末尾有 `_test`。
当您这样做时，您将只能访问包中的公共类型。这将有助于这种特定的情况，但也有助于加强只测试公共 api 的原则。
如果您仍然希望测试内部组件，您可以使用您想要测试的包进行单独的测试。

TDD 的一个格言是，如果您不能测试您的代码，那么您的代码的用户可能很难与它集成。使用 `package foo_test` 会帮助你测试你的代码，
就像你导入你的包的用户一样。

在修复 `main` 之前，让我们先修改 `CLI_test` 里面的测试包到 `poker_test`。

如果您有一个配置良好的 IDE，您会突然看到很多红色!如果您运行编译器，您将得到以下错误


```
./CLI_test.go:12:19: undefined: StubPlayerStore
./CLI_test.go:17:3: undefined: assertPlayerWin
./CLI_test.go:22:19: undefined: StubPlayerStore
./CLI_test.go:27:3: undefined: assertPlayerWin
```

现在我们遇到了更多关于包设计的问题。
为了测试我们的软件，我们制作了未导出的存根和 helper 函数，这些函数在 `CLI_test` 中不再可用，
因为 helper 函数是在`poker` 包中 `_test.go`中定义的。


#### Do we want to have our stubs and helpers 'public'?

这是一个主观的讨论。有人可能会说，您不想用代码污染包的API来方便测试。

In the presentation ["Advanced Testing with Go"](https://speakerdeck.com/mitchellh/advanced-testing-with-go?slide=53) 
by Mitchell Hashimoto, 

Mitchell Hashimoto 的演讲 ["Advanced Testing with Go"](https://speakerdeck.com/mitchellh/advanced-testing-with-go?slide=53) 
它描述了 HashiCorp 是如何提倡这样做的，这样包的用户就可以编写测试，而不必重新发明轮子写存根。
在我们的例子中，这意味着任何使用我们的 `poker` 包的人，如果他们希望使用我们的代码，就不需要创建他们自己的存根 `PlayerStore`。

有趣的是，我曾在其他共享包中使用过这种技术，事实证明它在用户与我们的包集成时节省了时间方面非常有用。

让我们创建一个名为 `testing.go` 的文件。然后加入我们的存根和 helper。


```go
//testing.go
package poker

import "testing"

type StubPlayerStore struct {
	scores   map[string]int
	winCalls []string
	league   []Player
}

func (s *StubPlayerStore) GetPlayerScore(name string) int {
	score := s.scores[name]
	return score
}

func (s *StubPlayerStore) RecordWin(name string) {
	s.winCalls = append(s.winCalls, name)
}

func (s *StubPlayerStore) GetLeague() League {
	return s.league
}

func AssertPlayerWin(t testing.TB, store *StubPlayerStore, winner string) {
	t.Helper()

	if len(store.winCalls) != 1 {
		t.Fatalf("got %d calls to RecordWin want %d", len(store.winCalls), 1)
	}

	if store.winCalls[0] != winner {
		t.Errorf("did not store correct winner got %q want %q", store.winCalls[0], winner)
	}
}

// todo for you - the rest of the helpers
```

如果您想让我们的包的进口商看到这些 helper，您需要将它们公开(记住，导出是在开始时使用大写字母)。

在我们的 `CLI` 测试中，您需要调用代码，就像您在不同的包中使用它一样。

```go
//CLI_test.go
func TestCLI(t *testing.T) {

	t.Run("record chris win from user input", func(t *testing.T) {
		in := strings.NewReader("Chris wins\n")
		playerStore := &poker.StubPlayerStore{}

		cli := &poker.CLI{playerStore, in}
		cli.PlayPoker()

		poker.AssertPlayerWin(t, playerStore, "Chris")
	})

	t.Run("record cleo win from user input", func(t *testing.T) {
		in := strings.NewReader("Cleo wins\n")
		playerStore := &poker.StubPlayerStore{}

		cli := &poker.CLI{playerStore, in}
		cli.PlayPoker()

		poker.AssertPlayerWin(t, playerStore, "Cleo")
	})

}
```

现在你会看到我们有和 `main` 一样的问题

```
./CLI_test.go:15:26: implicit assignment of unexported field 'playerStore' in poker.CLI literal
./CLI_test.go:15:39: implicit assignment of unexported field 'in' in poker.CLI literal
./CLI_test.go:25:26: implicit assignment of unexported field 'playerStore' in poker.CLI literal
./CLI_test.go:25:39: implicit assignment of unexported field 'in' in poker.CLI literal
```

解决这个问题的最简单方法是像处理其他类型一样创建一个构造函数。
我们还将更改 `CLI` ，以便它存储 `bufio.Scanner` 而不是阅读器，因为它现在在构造时自动包装。

```go
//CLI.go
type CLI struct {
	playerStore PlayerStore
	in          *bufio.Scanner
}

func NewCLI(store PlayerStore, in io.Reader) *CLI {
	return &CLI{
		playerStore: store,
		in:          bufio.NewScanner(in),
	}
}
```

通过这样做，我们可以简化和重构我们的阅读代码

```go
//CLI.go
func (cli *CLI) PlayPoker() {
	userInput := cli.readLine()
	cli.playerStore.RecordWin(extractWinner(userInput))
}

func extractWinner(userInput string) string {
	return strings.Replace(userInput, " wins", "", 1)
}

func (cli *CLI) readLine() string {
	cli.in.Scan()
	return cli.in.Text()
}
```

将测试改为使用构造函数，我们应该会回到通过测试的状态。

最后，我们可以回到新的 `main`。然后使用我们刚刚创建的构造函数


```go
//cmd/cli/main.go
game := poker.NewCLI(store, os.Stdin)
```

运行，输入 "Bob wins"

### Refactor

我们在各自的应用程序中有一些重复，我们打开一个文件并从其内容创建一个 `file_system_store`。
这似乎是我们包设计中的一个弱点，所以我们应该在其中创建一个函数来封装从路径打开文件并返回 `PlayerStore`。



```go
//file_system_store.go
func FileSystemPlayerStoreFromFile(path string) (*FileSystemPlayerStore, func(), error) {
	db, err := os.OpenFile(path, os.O_RDWR|os.O_CREATE, 0666)

	if err != nil {
		return nil, nil, fmt.Errorf("problem opening %s %v", path, err)
	}

	closeFunc := func() {
		db.Close()
	}

	store, err := NewFileSystemPlayerStore(db)

	if err != nil {
		return nil, nil, fmt.Errorf("problem creating file system player store, %v ", err)
	}

	return store, closeFunc, nil
}
```

现在重构我们的两个应用程序，使用这个函数来创建存储。



#### CLI application code

```go
//cmd/cli/main.go
package main

import (
	"fmt"
	"github.com/quii/learn-go-with-tests/command-line/v3"
	"log"
	"os"
)

const dbFileName = "game.db.json"

func main() {
	store, close, err := poker.FileSystemPlayerStoreFromFile(dbFileName)

	if err != nil {
		log.Fatal(err)
	}
	defer close()

	fmt.Println("Let's play poker")
	fmt.Println("Type {Name} wins to record a win")
	poker.NewCLI(store, os.Stdin).PlayPoker()
}
```

#### Web server application code

```go
//cmd/webserver/main.go
package main

import (
	"github.com/quii/learn-go-with-tests/command-line/v3"
	"log"
	"net/http"
)

const dbFileName = "game.db.json"

func main() {
	store, close, err := poker.FileSystemPlayerStoreFromFile(dbFileName)

	if err != nil {
		log.Fatal(err)
	}
	defer close()

	server := poker.NewPlayerServer(store)

	if err := http.ListenAndServe(":5000", server); err != nil {
		log.Fatalf("could not listen on port 5000 %v", err)
	}
}
```

注意它的对称性:尽管用户界面不同，但设置几乎是相同的。到目前为止，这感觉像是对我们设计的良好验证。
还要注意 `FileSystemPlayerStoreFromFile` 返回一个关闭函数，
因此，一旦我们使用完 Store，就可以关闭底层文件。

## Wrapping up

### Package structure

这一章意味着我们想要创建两个应用程序，重用我们迄今为止编写的域代码。为了做到这一点，我们需要更新我们的包结构，以便我们有各自的 `main` 的独立文件夹。

通过这样做，我们遇到了由于未导出值而导致的集成问题，因此这进一步证明了以小“片”工作并经常集成的价值。

我们了解了 `mypackage_test` 如何帮助我们创建一个测试环境，这与其他集成到您的代码中的包的体验是相同的，
以帮助您捕捉集成问题，并看看您的代码使用起来有多容易(或不容易!)

### Reading user input

我们看到了如何从 `os.Stdin` 阅读对我们来说很容易使用，因为它实现了 `io.Reader`。我们使用 `bufio.Scanner` 容易读取一行一行的用户输入。


### Simple abstractions leads to simpler code re-use

将 `PlayerStore` 整合到我们的新应用中几乎不费什么功夫(一旦我们调整了包)，随后的测试也很容易，因为我们决定也公开存根版本。

