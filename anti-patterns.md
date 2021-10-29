# TDD Anti-patterns

不时地回顾你的 TDD 技术并提醒自己要避免的行为是很有必要的。

TDD process 在概念上很容易遵循，但当您这样做时，您会发现它对您的设计技能构成了挑战。**不要误以为TDD是困难的，真正困难的是设计!**

本章列出了许多 TDD 和测试反模式，以及如何纠正它们。

## 根本不做 TDD

当然，没有 TDD 也可以写出优秀的软件，
如果使用了严格的 TDD 方法，我所看到的代码设计和测试质量方面的许多问题将很难解决。

TDD 的优点之一是，它给你一个正式的过程来分解问题，理解你想要达到的目标(红色)，完成目标(绿色)，然后好好想想如何做对(蓝色/重构)。
如果没有这些，这个过程通常是 ad-hoc 的和松散的，这可能会使工程比原来更加困难。

## 误解重构步骤的约束

我参加过许多研讨会、聚会或结对会议，在这些会议中，有人通过了测试，并处于重构阶段。经过一番思考，他们认为把一些代码抽象成一个新的结构体是很好的;一个萌芽中的学究喊道:

> 你不能这么做！您应该首先为此编写一个测试，我们正在进行TDD！

这似乎是一个常见的误解。**当测试为绿色时，您可以对代码做任何您想做的事情**，唯一不允许你做的就是**增加或改变行为**。                          
          
这些测试的目的是给你重构的自由，找到正确的抽象，使代码更容易更改和理解。

## 拥有不会失败的测试(或常青测试)

这种情况出现的频率令人吃惊。您开始调试或更改一些测试，并意识到：没有这种测试可能失败的情况。或者至少，它不会以测试应该防止的方式失败。

如果你遵循第一步，那么这几乎是不可能的。

> 编写一个测试，看到它失败

当开发人员在编写完代码之后再编写测试时，这几乎总是要做的，并且/或追逐测试覆盖率而不是创建一个有用的测试套件。

## 无用的断言

曾经在一个系统上工作过，但是你失败了一个测试，然后你看到了这个?

> `false was not equal to true`

我知道 false 不等于 true。但是这个是没有任何帮助的信息。它没有告诉我什么东西出错了。这是没有遵循 TDD 过程和没有读取失败错误消息的症状。

> 编写一个测试，看到它失败了(不要为错误消息感到羞愧)

## 断言无关的细节

这方面的一个例子是对一个复杂对象进行断言，而实际上您在测试中只关心其中一个字段的值。

```go
// 不是这样的，现在您的测试与整个对象紧密耦合
if !cmp.Equal(complexObject, want) {
    t.Error("got %+v, want %+v", complexObject, want)
}

// 具体一点，松耦合
got := complexObject.fieldYouCareAboutForThisTest
if got != want{
    t.Error("got %q, want %q", got, want)
}
```
 
额外的断言不仅通过在文档中创建“噪音”使您的测试更难以阅读，但也不必要地将测试与它不关心的数据结合起来。
这意味着，如果您碰巧更改了对象的字段，或者它们的行为方式可能会导致测试出现意外的编译问题或失败。

这是一个没有严格遵循红色阶段的例子。

- 让现有的设计影响您编写测试的方式，而不是考虑所需的行为
- 对失败测试的错误消息没有给予足够的考虑

## 单元测试的单个场景中有很多断言

许多断言会使测试难以阅读，并且在测试失败时很难调试。

它们通常是慢慢地潜入，
特别是如果测试设置很复杂，因为您不愿意复制相同的可怕设置来断言其他东西。
相反，你应该解决设计中的问题，这些问题会让你难以断言新事物。

一个有用的经验法则是，每次测试都要做一个断言。
在 Go 中，在需要的情况下，利用子测试来清楚地描述断言之间的关系。
这也是一种方便的技术来分离行为和实现细节上的断言。

对于其他设置或执行时间可能受到限制的测试(例如驱动 web 浏览器的验收测试)，您需要权衡调试测试与测试执行时间之间的利弊。



## Not listening to your tests

[Dave Farley in his video "When TDD goes wrong"](https://www.youtube.com/watch?v=UWtEVKVPBQ0&feature=youtu.be) points out,

> TDD 为您的设计提供最快的反馈

从我自己的经验来看，许多开发人员试图实践 TDD，但经常忽略 TDD 过程返回给他们的信号。所以他们仍然被脆弱、恼人的系统和糟糕的测试套件所困。

简单地说，如果测试你的代码很困难，那么使用你的代码也很困难。将您的测试视为代码的第一个用户，然后您将看到您的代码是否易于使用。

我在书中强调了很多，我再说一遍**听你的测试**。

### 过多的设置，太多的测试重复，等等。

在测试中发生任何有趣的事情之前，你是否曾经看过一个有 20,50,100,200 行设置代码的测试?
然后，你是否不得不修改代码，重新审视混乱的局面，并希望自己有一个不同的职业?

这里的信号是什么?复杂测试 `==` 复杂代码。为什么你的代码很复杂?一定要这样吗?

- 当您的测试中有很多测试重复时，这意味着您正在测试的代码有很多依赖项 —— 这意味着您的设计需要工作。
- 如果您的测试依赖于设置与模拟的各种交互，这意味着您的代码正在与它的依赖项进行大量交互。问问自己这些互动是否可以更简单。

#### Leaky interfaces

如果你已经声明了一个有很多方法的 `interface`，那就会指向一个有漏洞的抽象。
考虑如何用一组更统一的方法来定义协作。

#### 想想你使用的测试类型

- Mock 有时是有用的，但它们非常强大，因此很容易被误用。试着限制自己使用 stub。
- 用 spies 验证实现细节有时是有帮助的，但要尽量避免。记住，实现细节通常不重要，如果可能的话，您不希望测试与它们耦合。把你的测试与“有用的行为”联系起来，而不是附带的细节。
- 如果测试double的分类有点不清楚 [Read my posts on naming test doubles](https://quii.dev/Start_naming_your_test_doubles_correctly) 。

#### Consolidate dependencies

这里是一个 `http.HandlerFunc` 的一些代码，用来处理一个网站的用户注册功能。

```go
type User struct {
	// Some user fields
}

type UserStore interface {
	CheckEmailExists(email string) (bool, error)
	StoreUser(newUser User) error
}

type Emailer interface {
	SendEmail(to User, body string, subject string) error
}

func NewRegistrationHandler(userStore UserStore, emailer Emailer) http.HandlerFunc {
	return func(writer http.ResponseWriter, request *http.Request) {
		// extract out the user from the request body (handle error)
		// check user exists (handle duplicates, errors)
		// store user (handle errors)
		// compose and send confirmation email (handle error)
		// if we got this far, return 2xx response
	}
}
```

设计还不算坏。只有两个依赖！

通过考虑 handler 的职责来重新评估设计:

- 将请求体解析到一个 `User` ✅
- 使用 `UserStore` 检查用户是否已经存在？
- 使用 `UserStore` 存储用户？
- Compose an email ❓
- 使用 `Emailer` 发送 email？
- 返回合适的 http 响应，基于操作成功还是失败 etc ✅

为了测试这些代码，你需要编写许多测试，包括不同程度的测试、双重设置、spies 等

- 如果需求扩展了怎么办？需要翻译 email？需要发送短信确认？您认为必须更改 HTTP 处理程序来适应这种更改有道理吗？

- “我们应该发送电子邮件”的重要规则驻留在 HTTP 处理程序中，这感觉对吗?
    - 为什么您必须通过创建HTTP请求和读取响应来验证该规则?

以 TDD 的方式为这些代码编写测试应该很快就会让您感到不舒服(或者至少会让您的懒惰开发人员感到恼火)。如果感觉疼痛，停下来思考。

如果设计如下呢？

```go
type UserService interface {
	Register(newUser User) error
}

func NewRegistrationHandler(userService UserService) http.HandlerFunc {
	return func(writer http.ResponseWriter, request *http.Request) {
		// parse user
		// register user
		// check error, send response
	}
}
```

- 测试处理程序很简单
- 对注册规则的更改与 HTTP 是隔离的，因此测试也更简单

## 违反封装

封装非常重要。我们不把包中的所有东西都导出(或公开)是有原因的。
我们需要具有小表面积的一致性 api，以避免紧密耦合。

人们有时会为了测试某些东西而将函数或方法公开。
这样做会使您的设计变得更糟，并向代码的维护者和用户发送令人困惑的消息。


这样做的结果可能是开发人员试图调试一个测试，然后最终意识到被测试的函数只能从 tests 中调用。
这显然是一个糟糕的结果，也是浪费时间。

在 Go 中，从包的使用者的角度考虑编写测试的默认位置。
你可以将你的测试放在一个测试包中，例如 `package gocoin_test`，从而使它成为一个编译时约束。
如果这样做，您将只能访问包中导出的成员，因此不可能将自己与实现细节耦合在一起。

## Complicated table tests

当测试设置相同，而您只希望改变输入时，表测试是测试许多不同场景的好方法。

但是，当你试图以拥有一个光荣的表的名义强行塞进其他类型的测试时，阅读和理解它们可能会很麻烦。

```go
cases := []struct {
    X int
    Y int
    Z int
    err error
    IsFullMoon bool
    IsLeapYear bool
    AtWarWithEurasia bool
}
```

**不要害怕拆分你的表并编写新的测试** 而不是向表 `struct` 添加新的字段和布尔值。

在编写软件时要记住的一件事是，

> [Simple is not easy](https://www.infoq.com/presentations/Simple-Made-Easy/)

“只是”向表添加一个字段可能很容易，但它会使事情变得远不简单。


## Summary

单元测试的大多数问题通常可以追溯到:

- 开发人员没有遵循TDD流程
- 糟糕的设计

所以，学习优秀的软件设计吧!

好消息是 TDD 可以帮助你提高你的设计技能，因为正如开头所述:

**TDD的主要目的是为你的设计提供反馈。** 我已经说过一百万次了，倾听你的测试，它们反映了你的设计。

通过听取他们给你的反馈，诚实对待你的测试质量，你会因此成为一个更好的开发者。
