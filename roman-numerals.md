# Roman Numerals

**[You can find all the code for this chapter here](https://github.com/quii/learn-go-with-tests/tree/main/roman-numerals)**

一些公司会要求你做[罗马数字 Kata](http://codingdojo.org/kata/RomanNumerals/) 作为面试过程的一部分。本章将展示如何使用 TDD 来解决这个问题。

我们将编写一个函数，它将[阿拉伯数字](https://en.wikipedia.org/wiki/Arabic_numerals)(数字 0 到 9)转换为罗马数字。

如果你没有听说过[罗马数字](https://en.wikipedia.org/wiki/Roman_numerals)，他们是罗马人记录数字的方式。

你把符号粘在一起，这些符号代表数字

`I` 代表一， `III` 代表三。

看起来很简单，但有一些有趣的规则。`V` 表示五，但 `IV` 是 4(不是 `IIII`)。

`MCMLXXXIV`是 1984。这看起来很复杂，很难想象我们如何从一开始就编写代码来解决这个问题。

正如本书所强调的，软件开发人员的一项关键技能是尝试并识别有用功能的“垂直薄片”，然后**迭代**。TDD 工作流有助于促进迭代开发。

我们从 1 开始，而不是 1984。

## 先写测试

```go
func TestRomanNumerals(t *testing.T) {
	got := ConvertToRoman(1)
	want := "I"

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

如果你们已经读到这里了，希望你们会觉得这很无聊，很老套。这是一件好事。

## 运行测试

`./numeral_test.go:6:9: undefined: ConvertToRoman`

让编译器来指导

## 为要运行的测试编写最小数量的代码，并检查失败的测试输出

创建我们的函数，但不要让测试通过，总是要确保测试以预期的方式失败

```go
func ConvertToRoman(arabic int) string {
	return ""
}
```

现在运行

```go
=== RUN   TestRomanNumerals
--- FAIL: TestRomanNumerals (0.00s)
    numeral_test.go:10: got '', want 'I'
FAIL
```

## 编写足够的代码使其通过

```go
func ConvertToRoman(arabic int) string {
	return "I"
}
```

## 重构

还没有太多需要重构的地方。

我知道，硬编码结果感觉很奇怪，但对于 TDD，我们希望尽可能远离“红色”。它可能感觉我们没有完成多少，但我们已经定义了我们的 API，并进行了一个测试捕捉我们的一个规则;即使“真正的”代码非常愚蠢。

现在用这种不安的感觉来编写一个新的测试，迫使我们编写稍微不那么愚蠢的代码。

## Write the test first

我们可以使用子测试来很好地分组测试

```go
func TestRomanNumerals(t *testing.T) {
	t.Run("1 gets converted to I", func(t *testing.T) {
		got := ConvertToRoman(1)
		want := "I"

		if got != want {
			t.Errorf("got %q, want %q", got, want)
		}
	})

	t.Run("2 gets converted to II", func(t *testing.T) {
		got := ConvertToRoman(2)
		want := "II"

		if got != want {
			t.Errorf("got %q, want %q", got, want)
		}
	})
}
```

## Try to run the test

```
=== RUN   TestRomanNumerals/2_gets_converted_to_II
    --- FAIL: TestRomanNumerals/2_gets_converted_to_II (0.00s)
        numeral_test.go:20: got 'I', want 'II'
```

Not much surprise there

## Write enough code to make it pass

```go
func ConvertToRoman(arabic int) string {
	if arabic == 2 {
		return "II"
	}
	return "I"
}
```

是的，感觉我们并没有真正解决这个问题。因此，我们需要编写更多的测试来推动我们前进。

## Refactor

我们的测试中有一些重复的内容。当你在测试一些东西时，感觉它是“给定输入X，我们期望Y”的问题，你可能应该使用基于表格的测试。

```go
func TestRomanNumerals(t *testing.T) {
	cases := []struct {
		Description string
		Arabic      int
		Want        string
	}{
		{"1 gets converted to I", 1, "I"},
		{"2 gets converted to II", 2, "II"},
	}

	for _, test := range cases {
		t.Run(test.Description, func(t *testing.T) {
			got := ConvertToRoman(test.Arabic)
			if got != test.Want {
				t.Errorf("got %q, want %q", got, test.Want)
			}
		})
	}
}
```

我们现在可以轻松地添加更多的用例，而不必编写更多的测试样板。

让我们添加 3 这个用例

## Write the test first

添加下面的用例

```go
{"3 gets converted to III", 3, "III"},
```

## Try to run the test

```
=== RUN   TestRomanNumerals/3_gets_converted_to_III
    --- FAIL: TestRomanNumerals/3_gets_converted_to_III (0.00s)
        numeral_test.go:20: got 'I', want 'III'
```

## Write enough code to make it pass

```go
func ConvertToRoman(arabic int) string {
	if arabic == 3 {
		return "III"
	}
	if arabic == 2 {
		return "II"
	}
	return "I"
}
```

## Refactor

好的，我开始不喜欢这些 if 语句了，如果你仔细看代码，你会发现我们建立了一个基于 `arabic` 大小的 `I` 字符串。

我们“知道”对于更复杂的数字，我们将进行某种算术和字符串连接。

让我们带着这些想法来尝试重构，它可能不适合最终的解决方案，但这是可以的。我们总是可以扔掉我们的代码，用我们必须指导的测试重新开始。


```go
func ConvertToRoman(arabic int) string {

	var result strings.Builder

	for i:=0; i<arabic; i++ {
		result.WriteString("I")
	}

	return result.String()
}
```

可能你之前没有使用过 [`strings.Builder`](https://golang.org/pkg/strings/#Builder)

> Builder 用于使用 Write 方法高效地构建字符串。它最小化了内存复制。

通常情况下，除非我遇到实际的性能问题，否则我不会为这种优化而烦恼，但代码量并不比“手动”追加字符串大多少，所以我们不妨使用更快的方法。

代码对我来说更好看，并且描述了我们现在所知道的领域。

### The Romans were into DRY too...

事情开始变得更复杂了。聪明的罗马人认为重复的字符会变得难以阅读和计算。所以罗马数字的一个规则是，同一个字符不能连续重复超过 3 次。

相反，你取下一个最高的符号，然后“subtract”，把一个符号放在它的左边。不是所有的符号都可以作为减法;只有I (1)， X(10)和C (100)

例如，`5` 在罗马数字中是 `V`。要创造 4，你不需要做 `IIII`，而是要做 `IV`。

## Write the test first

```go
{"4 gets converted to IV (can't repeat more than 3 times)", 4, "IV"},
```

## Try to run the test

```
=== RUN   TestRomanNumerals/4_gets_converted_to_IV_(cant_repeat_more_than_3_times)
    --- FAIL: TestRomanNumerals/4_gets_converted_to_IV_(cant_repeat_more_than_3_times) (0.00s)
        numeral_test.go:24: got 'IIII', want 'IV'
```

## Write enough code to make it pass

```go
func ConvertToRoman(arabic int) string {

	if arabic == 4 {
		return "IV"
	}

	var result strings.Builder

	for i:=0; i<arabic; i++ {
		result.WriteString("I")
	}

	return result.String()
}
```

## Refactor

我不“喜欢”我们打破了我们的 string building  模式，我想继续它。

```go
func ConvertToRoman(arabic int) string {

	var result strings.Builder

	for i := arabic; i > 0; i-- {
		if i == 4 {
			result.WriteString("IV")
			break
		}
		result.WriteString("I")
	}

	return result.String()
}
```
为了让4“适合”我现在的想法，我现在从阿拉伯数字倒数，在我们前进的时候添加符号到我们的字符串。不确定从长远来看这是否有效，但让我们看看!

Let's make 5 work

## Write the test first

```go
{"5 gets converted to V", 5, "V"},
```

## Try to run the test

```
=== RUN   TestRomanNumerals/5_gets_converted_to_V
    --- FAIL: TestRomanNumerals/5_gets_converted_to_V (0.00s)
        numeral_test.go:25: got 'IIV', want 'V'
```

## Write enough code to make it pass

复制 4 的方法

```go
func ConvertToRoman(arabic int) string {

	var result strings.Builder

	for i := arabic; i > 0; i-- {
		if i == 5 {
			result.WriteString("V")
			break
		}
		if i == 4 {
			result.WriteString("IV")
			break
		}
		result.WriteString("I")
	}

	return result.String()
}
```

## Refactor

像这样的循环中的重复通常是一个抽象等待调用的标志。短路循环是一种有效的可读性工具，但它也可以告诉你一些其他的东西。

我们在循环我们的阿拉伯数字，如果我们击中特定的符号，我们调用 `break`，但我们真正做的是用笨拙的方式减去 `i`。

```go
func ConvertToRoman(arabic int) string {

	var result strings.Builder

	for arabic > 0 {
		switch {
		case arabic > 4:
			result.WriteString("V")
			arabic -= 5
		case arabic > 3:
			result.WriteString("IV")
			arabic -= 4
		default:
			result.WriteString("I")
			arabic--
		}
	}

	return result.String()
}

```

- Given the signals I'm reading from our code, driven from our tests of some very basic scenarios I can see that to build a Roman Numeral I need to subtract from `arabic` as I apply symbols
- The `for` loop no longer relies on an `i` and instead we will keep building our string until we have subtracted enough symbols away from `arabic`.

I'm pretty sure this approach will be valid for 6 (VI), 7 (VII) and 8 (VIII) too. Nonetheless add the cases in to our test suite and check (I won't include the code for brevity, check the github for samples if you're unsure).

9 follows the same rule as 4 in that we should subtract `I` from the representation of the following number. 10 is represented in Roman Numerals with `X`; so therefore 9 should be `IX`.

## Write the test first

```go
{"9 gets converted to IX", 9, "IX"}
```
## Try to run the test

```
=== RUN   TestRomanNumerals/9_gets_converted_to_IX
    --- FAIL: TestRomanNumerals/9_gets_converted_to_IX (0.00s)
        numeral_test.go:29: got 'VIV', want 'IX'
```

## Write enough code to make it pass

We should be able to adopt the same approach as before

```go
case arabic > 8:
    result.WriteString("IX")
    arabic -= 9
```

## Refactor

感觉代码仍然在告诉我们某个地方需要重构，但对我来说不是很明显，所以让我们继续。

我将跳过这部分的代码，但在您的测试用例中添加一个 `10` 的测试，它应该是 `X`，并在阅读之前让它通过。

下面是我添加的一些测试，因为我确信到39，我们的代码应该可以工作

```go
{"10 gets converted to X", 10, "X"},
{"14 gets converted to XIV", 14, "XIV"},
{"18 gets converted to XVIII", 18, "XVIII"},
{"20 gets converted to XX", 20, "XX"},
{"39 gets converted to XXXIX", 39, "XXXIX"},
```

If you've ever done OO programming, you'll know that you should view `switch` statements with a bit of suspicion. Usually you are capturing a concept or data inside some imperative code when in fact it could be captured in a class structure instead.

Go isn't strictly OO but that doesn't mean we ignore the lessons OO offers entirely (as much as some would like to tell you).

Our switch statement is describing some truths about Roman Numerals along with behaviour.

We can refactor this by decoupling the data from the behaviour.

```go
type RomanNumeral struct {
	Value  int
	Symbol string
}

var allRomanNumerals = []RomanNumeral {
	{10, "X"},
	{9, "IX"},
	{5, "V"},
	{4, "IV"},
	{1, "I"},
}

func ConvertToRoman(arabic int) string {

	var result strings.Builder

	for _, numeral := range allRomanNumerals {
		for arabic >= numeral.Value {
			result.WriteString(numeral.Symbol)
			arabic -= numeral.Value
		}
	}

	return result.String()
}
```

这个感觉好多了。我们声明了一些关于数字的规则作为数据，而不是隐藏在算法中，我们可以看到我们是如何处理阿拉伯数字的，尝试在我们的结果中添加符号，如果它们符合。

这种抽象是否适用于更大的数字?扩展测试套件，使其能够适用于罗马数字 50，即 `L`。

这里有一些测试用例，试着让它们通过。


```go
{"40 gets converted to XL", 40, "XL"},
{"47 gets converted to XLVII", 47, "XLVII"},
{"49 gets converted to XLIX", 49, "XLIX"},
{"50 gets converted to L", 50, "L"},
```

需要帮忙吗?您可以看到要添加哪些符号，从这[this gist](https://gist.github.com/pamelafox/6c7b948213ba55332d86efd0f0b037de).

## And the rest!

Here are the remaining symbols

| Arabic        | Roman           |
| ------------- |:-------------:|
| 100     | C      |
| 500 | D      |
| 1000 | M      |

对其余的符号采用相同的方法，它应该只是向测试和符号数组中添加数据的问题。

你的代码能算出 `1984` 是 `MCMLXXXIV` 吗？

下面是最终的测试套件

```go
func TestRomanNumerals(t *testing.T) {
	cases := []struct {
		Arabic int
		Roman  string
	}{
		{Arabic: 1, Roman: "I"},
		{Arabic: 2, Roman: "II"},
		{Arabic: 3, Roman: "III"},
		{Arabic: 4, Roman: "IV"},
		{Arabic: 5, Roman: "V"},
		{Arabic: 6, Roman: "VI"},
		{Arabic: 7, Roman: "VII"},
		{Arabic: 8, Roman: "VIII"},
		{Arabic: 9, Roman: "IX"},
		{Arabic: 10, Roman: "X"},
		{Arabic: 14, Roman: "XIV"},
		{Arabic: 18, Roman: "XVIII"},
		{Arabic: 20, Roman: "XX"},
		{Arabic: 39, Roman: "XXXIX"},
		{Arabic: 40, Roman: "XL"},
		{Arabic: 47, Roman: "XLVII"},
		{Arabic: 49, Roman: "XLIX"},
		{Arabic: 50, Roman: "L"},
		{Arabic: 100, Roman: "C"},
		{Arabic: 90, Roman: "XC"},
		{Arabic: 400, Roman: "CD"},
		{Arabic: 500, Roman: "D"},
		{Arabic: 900, Roman: "CM"},
		{Arabic: 1000, Roman: "M"},
		{Arabic: 1984, Roman: "MCMLXXXIV"},
		{Arabic: 3999, Roman: "MMMCMXCIX"},
		{Arabic: 2014, Roman: "MMXIV"},
		{Arabic: 1006, Roman: "MVI"},
		{Arabic: 798, Roman: "DCCXCVIII"},
	}
	for _, test := range cases {
		t.Run(fmt.Sprintf("%d gets converted to %q", test.Arabic, test.Roman), func(t *testing.T) {
			got := ConvertToRoman(test.Arabic)
			if got != test.Roman {
				t.Errorf("got %q, want %q", got, test.Roman)
			}
		})
	}
}
```

- 我删除了 description ，因为我觉得 _data_ 描述了足够多的信息。
- 我添加了一些我发现的其他边缘情况，只是为了让我更有信心。使用基于表的测试，这是非常便宜的。

我没有改变算法，我只需要更新 `allRomanNumerals` 数组。

```go
var allRomanNumerals = []RomanNumeral{
	{1000, "M"},
	{900, "CM"},
	{500, "D"},
	{400, "CD"},
	{100, "C"},
	{90, "XC"},
	{50, "L"},
	{40, "XL"},
	{10, "X"},
	{9, "IX"},
	{5, "V"},
	{4, "IV"},
	{1, "I"},
}
```

## Parsing Roman Numerals

我们还没做完。接下来我们要写一个函数将罗马数字转换位 `int`

## Write the test first

我们可以通过一些重构来重用我们的测试用例

将 `cases` 变量作为 `var` 块中的包变量移动到测试之外。

```go
func TestConvertingToArabic(t *testing.T) {
	for _, test := range cases[:1] {
		t.Run(fmt.Sprintf("%q gets converted to %d", test.Roman, test.Arabic), func(t *testing.T) {
			got := ConvertToArabic(test.Roman)
			if got != test.Arabic {
				t.Errorf("got %d, want %d", got, test.Arabic)
			}
		})
	}
}
```

注意，我现在使用切片功能只是运行其中一个测试( `cases[:1]`)，因为试图让所有这些测试一次性通过太大了

## Try to run the test

```
./numeral_test.go:60:11: undefined: ConvertToArabic
```

## Write the minimal amount of code for the test to run and check the failing test output

定义新函数

```go
func ConvertToArabic(roman string) int {
	return 0
}
```

现在运行测试应该失败

```
--- FAIL: TestConvertingToArabic (0.00s)
    --- FAIL: TestConvertingToArabic/'I'_gets_converted_to_1 (0.00s)
        numeral_test.go:62: got 0, want 1
```

## Write enough code to make it pass

You know what to do

```go
func ConvertToArabic(roman string) int {
	return 1
}
```

接下来，在我们的测试中更改切片索引以移动到下一个测试用例(例如。“情况下[:2]”)。让它通过你能想到的最愚蠢的代码，继续为第三种情况编写愚蠢的代码(最好的书，对吗?)这是我愚蠢的代码。

```go
func ConvertToArabic(roman string) int {
	if roman == "III" {
		return 3
	}
	if roman == "II" {
		return 2
	}
	return 1
}
```

通过工作的代码的沉默，我们可以开始看到一个像以前一样的模式。我们需要遍历输入并构建 _something_，在本例中为 total。

```go
func ConvertToArabic(roman string) int {
	total := 0
	for range roman {
		total++
	}
	return total
}
```

## Write the test first

接下来我们移动到 `cases[:4]` (`IV`)，它现在失败了，因为它返回了 2，因为这是字符串的长度。

## Write enough code to make it pass

```go
// earlier..
type RomanNumerals []RomanNumeral

func (r RomanNumerals) ValueOf(symbol string) int {
	for _, s := range r {
		if s.Symbol == symbol {
			return s.Value
		}
	}

	return 0
}

// later..
func ConvertToArabic(roman string) int {
	total := 0

	for i := 0; i < len(roman); i++ {
		symbol := roman[i]

		// look ahead to next symbol if we can and, the current symbol is base 10 (only valid subtractors)
		if i+1 < len(roman) && symbol == 'I' {
			nextSymbol := roman[i+1]

			// build the two character string
			potentialNumber := string([]byte{symbol, nextSymbol})

			// get the value of the two character string
			value := allRomanNumerals.ValueOf(potentialNumber)

			if value != 0 {
				total += value
				i++ // move past this character too for the next loop
			} else {
				total++
			}
		} else {
			total++
		}
	}
	return total
}
```

This is horrible but it does work. It's so bad I felt the need to add comments.

- I wanted to be able to look up an integer value for a given roman numeral so I made a type from our array of `RomanNumeral`s and then added a method to it, `ValueOf`
- Next in our loop we need to look ahead _if_ the string is big enough _and the current symbol is a valid subtractor_. At the moment it's just `I` (1) but can also be `X` (10) or `C` (100).
    - If it satisfies both of these conditions we need to lookup the value and add it to the total _if_ it is one of the special subtractors, otherwise ignore it
    - Then we need to further increment `i` so we don't count this symbol twice

## Refactor

I'm not entirely convinced this will be the long-term approach and there's potentially some interesting refactors we could do, but I'll resist that in case our approach is totally wrong. I'd rather make a few more tests pass first and see. For the meantime I made the first `if` statement slightly less horrible.

```go
func ConvertToArabic(roman string) int {
	total := 0

	for i := 0; i < len(roman); i++ {
		symbol := roman[i]

		if couldBeSubtractive(i, symbol, roman) {
			nextSymbol := roman[i+1]

			// build the two character string
			potentialNumber := string([]byte{symbol, nextSymbol})

			// get the value of the two character string
			value := allRomanNumerals.ValueOf(potentialNumber)

			if value != 0 {
				total += value
				i++ // move past this character too for the next loop
			} else {
				total++
			}
		} else {
			total++
		}
	}
	return total
}

func couldBeSubtractive(index int, currentSymbol uint8, roman string) bool {
	return index+1 < len(roman) && currentSymbol == 'I'
}
```

## Write the test first

Let's move on to `cases[:5]`

```
=== RUN   TestConvertingToArabic/'V'_gets_converted_to_5
    --- FAIL: TestConvertingToArabic/'V'_gets_converted_to_5 (0.00s)
        numeral_test.go:62: got 1, want 5
```

## Write enough code to make it pass

Apart from when it is subtractive our code assumes that every character is a `I` which is why the value is 1. We should be able to re-use our `ValueOf` method to fix this.

```go
func ConvertToArabic(roman string) int {
	total := 0

	for i := 0; i < len(roman); i++ {
		symbol := roman[i]

		// look ahead to next symbol if we can and, the current symbol is base 10 (only valid subtractors)
		if couldBeSubtractive(i, symbol, roman) {
			nextSymbol := roman[i+1]

			// build the two character string
			potentialNumber := string([]byte{symbol, nextSymbol})

			if value := allRomanNumerals.ValueOf(potentialNumber); value != 0 {
				total += value
				i++ // move past this character too for the next loop
			} else {
				total++ // this is fishy...
			}
		} else {
			total+=allRomanNumerals.ValueOf(string([]byte{symbol}))
		}
	}
	return total
}
```

## Refactor

When you index strings in Go, you get a `byte`. This is why when we build up the string again we have to do stuff like `string([]byte{symbol})`. It's repeated a couple of times, let's just move that functionality so that `ValueOf` takes some bytes instead.

```go
func (r RomanNumerals) ValueOf(symbols ...byte) int {
	symbol := string(symbols)
	for _, s := range r {
		if s.Symbol == symbol {
			return s.Value
		}
	}

	return 0
}
```

Then we can just pass in the bytes as is, to our function

```go
func ConvertToArabic(roman string) int {
	total := 0

	for i := 0; i < len(roman); i++ {
		symbol := roman[i]

		if couldBeSubtractive(i, symbol, roman) {
			if value := allRomanNumerals.ValueOf(symbol, roman[i+1]); value != 0 {
				total += value
				i++ // move past this character too for the next loop
			} else {
				total++ // this is fishy...
			}
		} else {
			total+=allRomanNumerals.ValueOf(symbol)
		}
	}
	return total
}
```

It's still pretty nasty, but it's getting there.

If you start moving our `cases[:xx]` number through you'll see that quite a few are passing now. Remove the slice operator entirely and see which ones fail, here's some examples from my suite

```
=== RUN   TestConvertingToArabic/'XL'_gets_converted_to_40
    --- FAIL: TestConvertingToArabic/'XL'_gets_converted_to_40 (0.00s)
        numeral_test.go:62: got 60, want 40
=== RUN   TestConvertingToArabic/'XLVII'_gets_converted_to_47
    --- FAIL: TestConvertingToArabic/'XLVII'_gets_converted_to_47 (0.00s)
        numeral_test.go:62: got 67, want 47
=== RUN   TestConvertingToArabic/'XLIX'_gets_converted_to_49
    --- FAIL: TestConvertingToArabic/'XLIX'_gets_converted_to_49 (0.00s)
        numeral_test.go:62: got 69, want 49
```

I think all we're missing is an update to `couldBeSubtractive` so that it accounts for the other kinds of subtractive symbols

```go
func couldBeSubtractive(index int, currentSymbol uint8, roman string) bool {
	isSubtractiveSymbol := currentSymbol == 'I' || currentSymbol == 'X' || currentSymbol =='C'
	return index+1 < len(roman) && isSubtractiveSymbol
}
```

Try again, they still fail. However we left a comment earlier...

```go
total++ // this is fishy...
```

We should never be just incrementing `total` as that implies every symbol is a `I`. Replace it with:

```go
total += allRomanNumerals.ValueOf(symbol)
```

And all the tests pass! Now that we have fully working software we can indulge ourselves in some refactoring, with confidence.

## Refactor

Here is all the code I finished up with. I had a few failed attempts but as I keep emphasising, that's fine and the tests help me play around with the code freely.

```go
import "strings"

func ConvertToArabic(roman string) (total int) {
	for _, symbols := range windowedRoman(roman).Symbols() {
		total += allRomanNumerals.ValueOf(symbols...)
	}
	return
}

func ConvertToRoman(arabic int) string {
	var result strings.Builder

	for _, numeral := range allRomanNumerals {
		for arabic >= numeral.Value {
			result.WriteString(numeral.Symbol)
			arabic -= numeral.Value
		}
	}

	return result.String()
}

type romanNumeral struct {
	Value  int
	Symbol string
}

type romanNumerals []romanNumeral

func (r romanNumerals) ValueOf(symbols ...byte) int {
	symbol := string(symbols)
	for _, s := range r {
		if s.Symbol == symbol {
			return s.Value
		}
	}

	return 0
}

func (r romanNumerals) Exists(symbols ...byte) bool {
	symbol := string(symbols)
	for _, s := range r {
		if s.Symbol == symbol {
			return true
		}
	}
	return false
}

var allRomanNumerals = romanNumerals{
	{1000, "M"},
	{900, "CM"},
	{500, "D"},
	{400, "CD"},
	{100, "C"},
	{90, "XC"},
	{50, "L"},
	{40, "XL"},
	{10, "X"},
	{9, "IX"},
	{5, "V"},
	{4, "IV"},
	{1, "I"},
}

type windowedRoman string

func (w windowedRoman) Symbols() (symbols [][]byte) {
	for i := 0; i < len(w); i++ {
		symbol := w[i]
		notAtEnd := i+1 < len(w)

		if notAtEnd && isSubtractive(symbol) && allRomanNumerals.Exists(symbol, w[i+1]) {
			symbols = append(symbols, []byte{byte(symbol), byte(w[i+1])})
			i++
		} else {
			symbols = append(symbols, []byte{byte(symbol)})
		}
	}
	return
}

func isSubtractive(symbol uint8) bool {
	return symbol == 'I' || symbol == 'X' || symbol == 'C'
}
```

My main problem with the previous code is similar to our refactor from earlier. We had too many concerns coupled together. We wrote an algorithm which was trying to extract Roman Numerals from a string _and_ then find their values.

So I created a new type `windowedRoman` which took care of extracting the numerals, offering a `Symbols` method to retrieve them as a slice. This meant our `ConvertToArabic` function could simply iterate over the symbols and total them.

I broke the code down a bit by extracting some functions, especially around the wonky if statement to figure out if the symbol we are currently dealing with is a two character subtractive symbol.

There's probably a more elegant way but I'm not going to sweat it. The code is there and it works and it is tested. If I (or anyone else) finds a better way they can safely change it - the hard work is done.

## An intro to property based tests

There have been a few rules in the domain of Roman Numerals that we have worked with in this chapter

- Can't have more than 3 consecutive symbols
- Only I (1), X (10) and C (100) can be "subtractors"
- Taking the result of `ConvertToRoman(N)` and passing it to `ConvertToArabic` should return us `N`

The tests we have written so far can be described as "example" based tests where we provide the tooling some examples around our code to verify.

What if we could take these rules that we know about our domain and somehow exercise them against our code?

Property based tests help you do this by throwing random data at your code and verifying the rules you describe always hold true. A lot of people think property based tests are mainly about random data but they would be mistaken. The real challenge about property based tests is having a _good_ understanding of your domain so you can write these properties.

Enough words, let's see some code

```go
func TestPropertiesOfConversion(t *testing.T) {
	assertion := func(arabic int) bool {
		roman := ConvertToRoman(arabic)
		fromRoman := ConvertToArabic(roman)
		return fromRoman == arabic
	}

	if err := quick.Check(assertion, nil); err != nil {
		t.Error("failed checks", err)
	}
}
```

### Rationale of property

Our first test will check that if we transform a number into Roman, when we use our other function to convert it back to a number that we get what we originally had.

- Given random number (e.g `4`).
- Call `ConvertToRoman` with random number (should return `IV` if `4`).
- Take the result of above and pass it to `ConvertToArabic`.
- The above should give us our original input (`4`).

This feels like a good test to build us confidence because it should break if there's a bug in either. The only way it could pass is if they have the same kind of bug; which isn't impossible but feels unlikely.

### Technical explanation

 We're using the [testing/quick](https://golang.org/pkg/testing/quick/) package from the standard library

 Reading from the bottom, we provide `quick.Check` a function that it will run against a number of random inputs, if the function returns `false` it will be seen as failing the check.

 Our `assertion` function above takes a random number and runs our functions to test the property.

 ### Run our test

 Try running it; your computer may hang for a while, so kill it when you're bored :)

 What's going on? Try adding the following to the assertion code.

 ```go
assertion := func(arabic int) bool {
    if arabic <0 || arabic > 3999 {
        log.Println(arabic)
        return true
    }
    roman := ConvertToRoman(arabic)
    fromRoman := ConvertToArabic(roman)
    return fromRoman == arabic
}
```

You should see something like this:

```
=== RUN   TestPropertiesOfConversion
2019/07/09 14:41:27 6849766357708982977
2019/07/09 14:41:27 -7028152357875163913
2019/07/09 14:41:27 -6752532134903680693
2019/07/09 14:41:27 4051793897228170080
2019/07/09 14:41:27 -1111868396280600429
2019/07/09 14:41:27 8851967058300421387
2019/07/09 14:41:27 562755830018219185
```

Just running this very simple property has exposed a flaw in our implementation. We used `int` as our input but:
- You can't do negative numbers with Roman Numerals
- Given our rule of a max of 3 consecutive symbols we can't represent a value greater than 3999 ([well, kinda](https://www.quora.com/Which-is-the-maximum-number-in-Roman-numerals)) and `int` has a much higher maximum value than 3999.

This is great! We've been forced to think more deeply about our domain which is a real strength of property based tests.

Clearly `int` is not a great type. What if we tried something a little more appropriate?

### [`uint16`](https://golang.org/pkg/builtin/#uint16)

Go has types for _unsigned integers_, which means they cannot be negative; so that rules out one class of bug in our code immediately. By adding 16, it means it is a 16 bit integer which can store a max of `65535`, which is still too big but gets us closer to what we need.

Try updating the code to use `uint16` rather than `int`. I updated `assertion` in the test to give a bit more visibility.

```go
assertion := func(arabic uint16) bool {
    if arabic > 3999 {
        return true
    }
    t.Log("testing", arabic)
    roman := ConvertToRoman(arabic)
    fromRoman := ConvertToArabic(roman)
    return fromRoman == arabic
}
```

If you run the test they now actually run and you can see what is being tested. You can run multiple times to see our code stands up well to the various values! This gives me a lot of confidence that our code is working how we want.

The default number of runs `quick.Check` performs is 100 but you can change that with a config.

```go
if err := quick.Check(assertion, &quick.Config{
    MaxCount:1000,
}); err != nil {
    t.Error("failed checks", err)
}
```

### Further work

- Can you write property tests that check the other properties we described?
- Can you think of a way of making it so it's impossible for someone to call our code with a number greater than 3999?
    - You could return an error
    - Or create a new type that cannot represent > 3999
        - What do you think is best?

## Wrapping up

### More TDD practice with iterative development

Did the thought of writing code that converts 1984 into MCMLXXXIV feel intimidating to you at first? It did to me and I've been writing software for quite a long time.

The trick, as always, is to **get started with something simple** and take **small steps**.

At no point in this process did we make any large leaps, do any huge refactorings, or get in a mess.

I can hear someone cynically saying "this is just a kata". I can't argue with that, but I still take this same approach for every project I work on. I never ship a big distributed system in my first step, I find the simplest thing the team could ship (usually a "Hello world" website) and then iterate on small bits of functionality in manageable chunks, just like how we did here.

The skill is knowing _how_ to split work up, and that comes with practice and with some lovely TDD to help you on your way.

### Property based tests

- Built into the standard library
- If you can think of ways to describe your domain rules in code, they are an excellent tool for giving you more confidence
- Force you to think about your domain deeply
- Potentially a nice complement to your test suite

## Postscript

This book is reliant on valuable feedback from the community.
[Dave](http://github.com/gypsydave5) is an enormous help in practically every
chapter. But he had a real rant about my use of 'Arabic numerals' in this
chapter so, in the interests of full disclosure, here's what he said.

> Just going to write up why a value of type `int` isn't really an 'arabic
> numeral'. This might be me being way too precise so I'll completely understand
> if you tell me to f off.
>
> A _digit_ is a character used in the representation of numbers - from the Latin
> for 'finger', as we usually have ten of them. In the Arabic (also called
> Hindu-Arabic) number system there are ten of them. These Arabic digits are:
>
>     0 1 2 3 4 5 6 7 8 9
>
> A _numeral_ is the representation of a number using a collection of digits.
> An Arabic numeral is a number represented by Arabic digits in a base 10
> positional number system. We say 'positional' because each digit has
> a different value based upon its position in the numeral. So
>
>     1337
>
> The `1` has a value of one thousand because its the first digit in a four
> digit numeral.
>
> Roman are built using a reduced number of digits (`I`, `V` etc...) mainly as
> values to produce the numeral. There's a bit of positional stuff but it's
> mostly `I` always representing 'one'.
>
> So, given this, is `int` an 'Arabic number'? The idea of a number is not at
> all tied to its representation - we can see this if we ask ourselves what the
> correct representation of this number is:
>
>     255
>     11111111
>     two-hundred and fifty-five
>     FF
>     377
>
> Yes, this is a trick question. They're all correct. They're the representation
> of the same number in the decimal,  binary, English, hexadecimal and octal
> number systems respectively.
>
> The representation of a number as a numeral is _independent_ of its properties
> as a number - and we can see this when we look at integer literals in Go:
>
> ```go
>  0xFF == 255 // true
> ```
>
> And how we can print integers in a format string:
>
> ```go
> n := 255
> fmt.Printf("%b %c %d %o %q %x %X %U", n, n, n, n, n, n, n, n)
> // 11111111 ÿ 255 377 'ÿ' ff FF U+00FF
> ```
>
> We can write the same integer both as a hexadecimal and an Arabic (decimal)
> numeral.
>
> So when the function signature looks like `ConvertToRoman(arabic int) string`
> it's making a bit of an assumption about how it's being called. Because
> sometimes `arabic` will be written as a decimal integer literal
>
> ```go
> ConvertToRoman(255)
> ```
>
> But it could just as well be written
>
> ```go
> ConvertToRoman(0xFF)
> ```
>
> Really, we're not 'converting' from an Arabic numeral at all, we're 'printing'  -
> representing - an `int` as a Roman numeral - and `int`s are not numerals,
> Arabic or otherwise; they're just numbers. The `ConvertToRoman` function is
> more like `strconv.Itoa` in that it's turning an `int` into a `string`.
>
> But every other version of the kata doesn't care about this distinction so
> :shrug:
