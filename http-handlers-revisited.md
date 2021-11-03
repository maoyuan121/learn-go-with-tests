# HTTP Handlers Revisited

**[You can find all the code here](https://github.com/quii/learn-go-with-tests/tree/main/q-and-a/http-handlers-revisited)**

这本书已经有一章是关于[测试HTTP处理程序](HTTP -server.md)的，但是这一章将对设计它们进行更广泛的讨论，所以它们很容易测试。

我们将看一个真实的例子，以及如何通过应用单一责任原则和关注点分离等原则来改进它的设计。这些原则可以通过使用[interfaces](structs-methods-and-interfaces.md)和[dependency injection](dependency-injection.md)来实现。通过这样做，我们将展示测试处理程序实际上是非常简单的。

![Common question in Go community illustrated](amazing-art.png)

在 Go 社区中，测试 HTTP 处理程序似乎是一个反复出现的问题，我认为它指向了一个更广泛的问题，即人们误解了如何设计它们。

因此，人们在测试方面的困难往往源于他们的代码设计，而不是实际编写测试。正如我在这本书中经常强调的:

> 如果您的测试给您带来了痛苦，请倾听这个信号并考虑代码的设计。

## An example

[Santosh Kumar tweeted me](https://twitter.com/sntshk/status/1255559003339284481)

> 我如何测试一个依赖 mongodb 的 http handler？


```go
func Registration(w http.ResponseWriter, r *http.Request) {
	var res model.ResponseResult
	var user model.User

	w.Header().Set("Content-Type", "application/json")

	jsonDecoder := json.NewDecoder(r.Body)
	jsonDecoder.DisallowUnknownFields()
	defer r.Body.Close()

	// check if there is proper json body or error
	if err := jsonDecoder.Decode(&user); err != nil {
		res.Error = err.Error()
		// return 400 status codes
		w.WriteHeader(http.StatusBadRequest)
		json.NewEncoder(w).Encode(res)
		return
	}

	// Connect to mongodb
	client, _ := mongo.NewClient(options.Client().ApplyURI("mongodb://127.0.0.1:27017"))
	ctx, _ := context.WithTimeout(context.Background(), 10*time.Second)
	err := client.Connect(ctx)
	if err != nil {
		panic(err)
	}
	defer client.Disconnect(ctx)
	// Check if username already exists in users datastore, if so, 400
	// else insert user right away
	collection := client.Database("test").Collection("users")
	filter := bson.D{{"username", user.Username}}
	var foundUser model.User
	err = collection.FindOne(context.TODO(), filter).Decode(&foundUser)
	if foundUser.Username == user.Username {
		res.Error = UserExists
		// return 400 status codes
		w.WriteHeader(http.StatusBadRequest)
		json.NewEncoder(w).Encode(res)
		return
	}

	pass, err := bcrypt.GenerateFromPassword([]byte(user.Password), bcrypt.DefaultCost)
	if err != nil {
		res.Error = err.Error()
		// return 400 status codes
		w.WriteHeader(http.StatusBadRequest)
		json.NewEncoder(w).Encode(res)
		return
	}
	user.Password = string(pass)

	insertResult, err := collection.InsertOne(context.TODO(), user)
	if err != nil {
		res.Error = err.Error()
		// return 400 status codes
		w.WriteHeader(http.StatusBadRequest)
		json.NewEncoder(w).Encode(res)
		return
	}

	// return 200
	w.WriteHeader(http.StatusOK)
	res.Result = fmt.Sprintf("%s: %s", UserCreated, insertResult.InsertedID)
	json.NewEncoder(w).Encode(res)
	return
}
```

让我们列出这个函数要做的所有事情:

1. 编写 HTTP 响应、发送报头、状态码等
2. 将请求体解码为 `User`
3. 连接到数据库(以及与此相关的所有细节)
4. 查询数据库并根据结果应用一些业务逻辑
5. 生成密码
6. 插入记录

干的事情太多了。

## 什么是 HTTP 处理程序，它应该做什么?

忘记具体细节一会儿,不管什么语言我一直为我工作是思考[关注点分离](https://en.wikipedia.org/wiki/Separation_of_concerns)和[单一责任原则](https://en.wikipedia.org/wiki/Single-responsibility_principle)。

根据你要解决的问题，这可能会非常棘手。责任到底是什么?

界限可能会模糊，这取决于你的思维有多抽象，有时你的第一个猜测可能是错误的。

谢天谢地，有了 HTTP 处理程序，我觉得我很清楚它们应该做什么，不管我在做什么项目：

1. 接受一个HTTP请求，解析并验证它。
2. 用我从第一步得到的数据调用一些 `ServiceThing` 来做 `ImportantBusinessLogic`。
3. 根据 `ServiceThing` 返回的内容发送一个适当的 `HTTP` 响应。

我并不是说每个HTTP处理程序都应该大致具有这种形状，但对我来说，这似乎是 99% 的情况。

当你把这些关注点分开时:

 - 测试处理程序变得轻而易举，只关注少数问题。
 - 重要的是，测试 `ImportantBusinessLogic` 不再需要关注 `HTTP` 本身，您可以干净地测试业务逻辑。
 - 你可以在其他情况下使用 `ImportantBusinessLogic` 而不需要修改它。
 - 如果 `ImportantBusinessLogic` 改变了它所做的事情，只要接口保持不变，您就不必更改处理程序。

## Go's Handlers

[`http.HandlerFunc`](https://golang.org/pkg/net/http/#HandlerFunc)

> HandlerFunc 类型是一个适配器，允许使用普通函数作为 HTTP 处理程序。

`type HandlerFunc func(ResponseWriter, *Request)`

读者，深呼吸，看看上面的代码。你注意到了什么?

**它是一个带有一些参数的函数**

没有神奇的框架，没有注释，没有魔豆，什么都没有。

它只是一个函数，我们知道如何测试函数。

这正好符合上面的评论:

- 它接收一个 [`http.Request`](https://golang.org/pkg/net/http/#Request) 这只是一组数据供我们检查，解析和验证。
- > [A `http.ResponseWriter` interface is used by an HTTP handler to construct an HTTP response.](https://golang.org/pkg/net/http/#ResponseWriter)

### Super basic example test

```go
func Teapot(res http.ResponseWriter, req *http.Request) {
	res.WriteHeader(http.StatusTeapot)
}

func TestTeapotHandler(t *testing.T) {
	req := httptest.NewRequest(http.MethodGet, "/", nil)
	res := httptest.NewRecorder()

	Teapot(res, req)

	if res.Code != http.StatusTeapot {
		t.Errorf("got status %d but wanted %d", res.Code, http.StatusTeapot)
	}
}
```

To test our function, we _call_ it.

为了我们测试，我们传入一个 `httptest.ResponseRecorder` 作为我们的 `http.ResponseWriter` 参数，我们的函数将使用它来编写 `HTTP` 响应。recorder 将记录(或监视)发送的内容，然后我们可以作出断言。

## Calling a `ServiceThing` in our handler


关于 TDD 教程的一个常见抱怨是，它们总是“太简单”，“不够真实”。我的回答是:

> 如果您的所有代码都像您提到的示例那样易于阅读和测试，那不是很好吗?

这是我们面临的最大挑战之一，但需要继续努力。设计代码是可能的(虽然不一定容易)，所以如果我们实践和应用好的软件工程原则，它可以很容易阅读和测试。

回顾一下前面的处理程序的作用:

1. 编写 HTTP 响应、发送报头、状态码等
2. 将请求体解码为 `User`
3. 连接到数据库(以及与此相关的所有细节)
4. 查询数据库并根据结果应用一些业务逻辑
5. 生成密码
6. 插入记录

考虑一个更理想的关注点分离的想法，我希望它是这样的:

1. 将请求体解码为 `User`
2. 调用 `UserService.Register(user)` (this is our `ServiceThing`)
3. 如果在它上有一个错误行为(示例总是发送一个 `400 BadRequest`，我认为是不对的，目前我们使用 `500 Internal Server Error` 兜底。我必须强调，对于所有错误返回 `500` 将导致一个糟糕的 API!稍后，我们可以使用[error types](error-types.md)使错误处理更加复杂。
4. 如果没有错误，则使用 ID 作为响应体的 `201 Created` (同样是为了简洁/懒惰)

为了简洁起见，我将不详细介绍通常的 TDD 过程，请查看所有其他章节中的示例。

### New design

```go
type UserService interface {
	Register(user User) (insertedID string, err error)
}

type UserServer struct {
	service UserService
}

func NewUserServer(service UserService) *UserServer {
	return &UserServer{service: service}
}

func (u *UserServer) RegisterUser(w http.ResponseWriter, r *http.Request) {
	defer r.Body.Close()

	// request parsing and validation
	var newUser User
	err := json.NewDecoder(r.Body).Decode(&newUser)

	if err != nil {
		http.Error(w, fmt.Sprintf("could not decode user payload: %v", err), http.StatusBadRequest)
		return
	}

	// call a service thing to take care of the hard work
	insertedID, err := u.service.Register(newUser)

	// depending on what we get back, respond accordingly
	if err != nil {
		//todo: handle different kinds of errors differently
		http.Error(w, fmt.Sprintf("problem registering new user: %v", err), http.StatusInternalServerError)
		return
	}

	w.WriteHeader(http.StatusCreated)
	fmt.Fprint(w, insertedID)
}
```

我们的 `RegisterUser` 方法匹配 `http.HandlerFunc` 这样就可以了。我们已经将它作为一个方法附加到一个新类型 `UserServer` 上，该类型包含一个对 `UserService` 的依赖，该依赖被捕获为一个接口。

接口是确保我们的 `HTTP` 关注点与任何具体实现解耦的奇妙方式;我们可以只调用依赖项上的方法，而不必关心用户是如何注册的。

如果您希望在 TDD 之后更详细地探索这种方法，请阅读[依赖注入](dependency-injection.md)章节和[HTTP服务器章节的“构建应用程序”章节](http-server.md)。

现在我们已经将自己与注册的任何具体实现细节解耦，为处理程序编写代码就很简单了，并遵循前面描述的职责。

### The tests!

这种简单性反映在我们的测试中。

```go
type MockUserService struct {
	RegisterFunc    func(user User) (string, error)
	UsersRegistered []User
}

func (m *MockUserService) Register(user User) (insertedID string, err error) {
	m.UsersRegistered = append(m.UsersRegistered, user)
	return m.RegisterFunc(user)
}

func TestRegisterUser(t *testing.T) {
	t.Run("can register valid users", func(t *testing.T) {
		user := User{Name: "CJ"}
		expectedInsertedID := "whatever"

		service := &MockUserService{
			RegisterFunc: func(user User) (string, error) {
				return expectedInsertedID, nil
			},
		}
		server := NewUserServer(service)

		req := httptest.NewRequest(http.MethodGet, "/", userToJSON(user))
		res := httptest.NewRecorder()

		server.RegisterUser(res, req)

		assertStatus(t, res.Code, http.StatusCreated)

		if res.Body.String() != expectedInsertedID {
			t.Errorf("expected body of %q but got %q", res.Body.String(), expectedInsertedID)
		}

		if len(service.UsersRegistered) != 1 {
			t.Fatalf("expected 1 user added but got %d", len(service.UsersRegistered))
		}

		if !reflect.DeepEqual(service.UsersRegistered[0], user) {
			t.Errorf("the user registered %+v was not what was expected %+v", service.UsersRegistered[0], user)
		}
	})

	t.Run("returns 400 bad request if body is not valid user JSON", func(t *testing.T) {
		server := NewUserServer(nil)

		req := httptest.NewRequest(http.MethodGet, "/", strings.NewReader("trouble will find me"))
		res := httptest.NewRecorder()

		server.RegisterUser(res, req)

		assertStatus(t, res.Code, http.StatusBadRequest)
	})

	t.Run("returns a 500 internal server error if the service fails", func(t *testing.T) {
		user := User{Name: "CJ"}

		service := &MockUserService{
			RegisterFunc: func(user User) (string, error) {
				return "", errors.New("couldn't add new user")
			},
		}
		server := NewUserServer(service)

		req := httptest.NewRequest(http.MethodGet, "/", userToJSON(user))
		res := httptest.NewRecorder()

		server.RegisterUser(res, req)

		assertStatus(t, res.Code, http.StatusInternalServerError)
	})
}
```

现在我们的处理程序没有耦合到特定的存储实现，所以编写一个 `MockUserService` 来帮助我们编写简单、快速的单元测试来执行它所承担的特定职责是很简单的。

### What about the database code? You're cheating!

这都是经过深思熟虑的。我们不希望HTTP处理程序关注我们的业务逻辑、数据库、连接等。

通过这样做，我们将处理程序从混乱的细节中解放出来，我们也使得测试持久性层和业务逻辑变得更容易，因为它也不再与无关的 HTTP 细节相耦合。

我们现在需要做的就是使用我们想要使用的数据库来实现我们的 `UserService`

```go
type MongoUserService struct {
}

func NewMongoUserService() *MongoUserService {
	//todo: pass in DB URL as argument to this function
	//todo: connect to db, create a connection pool
	return &MongoUserService{}
}

func (m MongoUserService) Register(user User) (insertedID string, err error) {
	// use m.mongoConnection to perform queries
	panic("implement me")
}
```

我们可以单独测试这个，一旦我们在 `main`，我们可以将这两个单元结合在一起，以实现我们的工作应用程序。

```go
func main() {
	mongoService := NewMongoUserService()
	server := NewUserServer(mongoService)
	http.ListenAndServe(":8000", http.HandlerFunc(server.RegisterUser))
}
```

### 一个更健壮和可扩展的设计

这些原则不仅使我们的生活在短期内更容易，而且使系统在未来更容易扩展。

在这个系统的进一步迭代中，我们希望通过电子邮件向用户确认注册，这一点也不奇怪。

在旧的设计中，我们必须改变处理程序和周围的测试。这通常是部分代码变得不可维护性的原因，越来越多的功能因为它已经这样设计了;为“HTTP handler”处理…一切!

通过使用接口分离关注点，我们不必编辑处理程序，因为它与注册的业务逻辑无关。

## 总结

测试 Go 的 HTTP 处理程序并不具有挑战性，但设计好的软件却很有挑战性!

人们错误地认为 HTTP 处理程序是特殊的，在编写它们时抛弃了良好的软件工程实践，从而使测试具有挑战性。

再次重申,**Go 的 http 处理程序只是函数**。如果您像编写其他函数一样编写它们，职责明确，关注点分离良好，那么测试它们就不会有问题，您的代码库也会因此变得更加健康。

