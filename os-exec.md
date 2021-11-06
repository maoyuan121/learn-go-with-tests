# OS Exec

**[You can find all the code here](https://github.com/quii/learn-go-with-tests/tree/main/q-and-a/os-exec)**

[keith6014](https://www.reddit.com/user/keith6014) asks on [reddit](https://www.reddit.com/r/golang/comments/aaz8ji/testdata_and_function_setup_help/)

> 我正在使用 os/exec.Command() 执行生成 XML 数据的命令。该命令将在一个名为 GetData() 的函数中执行。

> 为了测试 GetData()，我创建了一些 testdata。

> 在我的 _test.go 中有一个 TestGetData 调用 GetData()，但它将使用 os.exec。我希望它使用我的 testdata。
  
> 实现这一目标的好方法是什么?当调用 GetData 时，我应该有一个“测试”标志模式，以便它将读取文件即 GetData(模式字符串)?
 
A few things

- 当某些东西难以测试时，通常是因为关注点的分离不是很正确
- 不要在你的代码中添加“测试模式”，而是使用 [Dependency Injection](/dependency-injection.md)，这样你就可以对你的依赖进行建模并分离关注。

我冒昧地猜测了一下代码的样子

```go
type Payload struct {
	Message string `xml:"message"`
}

func GetData() string {
	cmd := exec.Command("cat", "msg.xml")

	out, _ := cmd.StdoutPipe()
	var payload Payload
	decoder := xml.NewDecoder(out)

	// these 3 can return errors but I'm ignoring for brevity
	cmd.Start()
	decoder.Decode(&payload)
	cmd.Wait()

	return strings.ToUpper(payload.Message)
}
```

- 使用 `exec.Command`，它允许您对进程执行外部命令
- 我们在 `cmd.StdoutPipe` 中捕获输出。它返回 `io.ReadCloser` (这将变得很重要)
- 代码的其余部分或多或少是从 [excellent documentation](https://golang.org/pkg/os/exec/#example_Cmd_StdoutPipe) copy 过来的.
    - 我们将标准输出的任何输出捕获到 `io.ReadCloser` 中，然后我们 `Start` 命令，然后通过调用 `Wait` 等待读取所有数据。在这两个调用之间，我们将数据解码到 `Payload` 结构体中。
     
下面是 `msg.xml` 的内容

```xml
<payload>
    <message>Happy New Year!</message>
</payload>
```

我编写了一个简单的测试来显示它的实际作用

```go
func TestGetData(t *testing.T) {
	got := GetData()
	want := "HAPPY NEW YEAR!"

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

## Testable code

可测试代码是解耦的和单一用途的。对我来说，这段代码有两个主要的问题

1. 检索原始XML数据
2. 解码 XML 数据并应用业务逻辑(在本例中是 `strings.ToUpper` on the `<message>`)

第一部分只是从标准库中复制示例。

第二部分是我们的业务逻辑，通过查看代码，我们可以看到逻辑的“接缝”在哪里开始;这是我们获得 `io.ReadCloser` 的地方。我们可以使用这个现有的抽象来分离关注点，并使代码可测试。

**GetData 的问题是业务逻辑与获取 XML 的方法相结合。为了使我们的设计更好，我们需要将它们解耦**

我们的 `TestGetData` 可以作为我们两个关注点之间的集成测试，所以我们将保持它，以确保它继续工作。

下面是新分离的代码

```go
type Payload struct {
	Message string `xml:"message"`
}

func GetData(data io.Reader) string {
	var payload Payload
	xml.NewDecoder(data).Decode(&payload)
	return strings.ToUpper(payload.Message)
}

func getXMLFromCommand() io.Reader {
	cmd := exec.Command("cat", "msg.xml")
	out, _ := cmd.StdoutPipe()

	cmd.Start()
	data, _ := ioutil.ReadAll(out)
	cmd.Wait()

	return bytes.NewReader(data)
}

func TestGetDataIntegration(t *testing.T) {
	got := GetData(getXMLFromCommand())
	want := "HAPPY NEW YEAR!"

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

现在 `GetData` 只从 `io.Reader` 获取输入。我们使它可测试，它不再关心数据如何检索;人们可以对任何返回 `io.Reader` 的函数进行重用。例如，我们可以从 URL 而不是命令行获取 XML。

```go
func TestGetData(t *testing.T) {
	input := strings.NewReader(`
<payload>
    <message>Cats are the best animal</message>
</payload>`)

	got := GetData(input)
	want := "CATS ARE THE BEST ANIMAL"

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}

```

下面是一个 `GetData` 单元测试的例子。

通过分离关注点和使用 Go 中现有的抽象，测试我们重要的业务逻辑是轻而易举的事。
