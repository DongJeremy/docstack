# 第十一章　测试

Maurice Wilkes，第一个存储程序计算机EDSAC的设计者，1949年他在实验室爬楼梯时有一个顿悟。在《计算机先驱回忆录》（Memoirs of a Computer Pioneer）里，他回忆到：“忽然间有一种醍醐灌顶的感觉，我整个后半生的美好时光都将在寻找程序BUG中度过了”。肯定从那之后的大部分正常的码农都会同情Wilkes过分悲观的想法，虽然也许会有人困惑于他对软件开发的难度的天真看法。

现在的程序已经远比Wilkes时代的更大也更复杂，也有许多技术可以让软件的复杂性可得到控制。其中有两种技术在实践中证明是比较有效的。第一种是代码在被正式部署前需要进行*代码评审*。第二种则是*测试*，也就是本章的讨论主题。

我们说测试的时候一般是指自动化测试，也就是写一些小的程序用来检测被测试代码（产品代码）的行为和预期的一样，这些通常都是精心设计的执行某些特定的功能或者是通过随机性的输入待验证边界的处理。

软件测试是一个巨大的领域。测试的任务可能已经占据了一些程序员的部分时间和另一些程序员的全部时间。和软件测试技术相关的图书或博客文章有成千上万之多。对于每一种主流的编程语言，都会有一打的用于测试的软件包，同时也有大量的测试相关的理论，而且每种都吸引了大量技术先驱和追随者。这些都足以说服那些想要编写有效测试的程序员重新学习一套全新的技能。

Go语言的测试技术是相对低级的。它依赖一个`go test`测试命令和一组按照约定方式编写的测试函数，测试命令可以运行这些测试函数。编写相对轻量级的纯测试代码是有效的，而且它很容易延伸到基准测试和示例文档。

在实践中，编写测试代码和编写程序本身并没有多大区别。我们编写的每一个函数也是针对每个具体的任务。我们必须小心处理边界条件，思考合适的数据结构，推断合适的输入应该产生什么样的结果输出。编写测试代码和编写普通的Go代码过程是类似的；它并不需要学习新的符号、规则和工具。
## 11.1. go test

`go test`命令是一个按照一定的约定和组织来测试代码的程序。在包目录内，所有以`_test.go`为后缀名的源文件在执行`go build`时不会被构建成包的一部分，它们是`go test`测试的一部分。

在`*_test.go`文件中，有三种类型的函数：

* **测试函数**、
* **基准测试**（benchmark）函数
* **示例函数**

一个测试函数是以`Test`为函数名前缀的函数，用于测试程序的一些逻辑行为是否正确；`go test`命令会调用这些测试函数并报告测试结果是`PASS`或`FAIL`。基准测试函数是以`Benchmark`为函数名前缀的函数，它们用于衡量一些函数的性能；`go test`命令会多次运行基准测试函数以计算一个平均的执行时间。示例函数是以`Example`为函数名前缀的函数，提供一个由编译器保证正确性的示例文档。我们将在11.2节讨论测试函数的所有细节，并在11.4节讨论基准测试函数的细节，然后在11.6节讨论示例函数的细节。

`go test`命令会遍历所有的`*_test.go`文件中符合上述命名规则的函数，生成一个临时的`main`包用于调用相应的测试函数，接着构建并运行、报告测试结果，最后清理测试中生成的临时文件。

## 11.2. 测试函数

每个测试函数必须导入`testing`包。测试函数有如下的签名：

```Go
func TestName(t *testing.T) {
	// ...
}
```

测试函数的名字必须以`Test`开头，可选的后缀名必须以大写字母开头：

```Go
func TestSin(t *testing.T) { /* ... */ }
func TestCos(t *testing.T) { /* ... */ }
func TestLog(t *testing.T) { /* ... */ }
```

其中t参数用于报告测试失败和附加的日志信息。让我们定义一个实例包`gopl.io/ch11/word1`，其中只有一个函数`IsPalindrome`用于检查一个字符串是否从前向后和从后向前读都是一样的。（下面这个实现对于一个字符串是否是回文字符串前后重复测试了两次；我们稍后会再讨论这个问题。）

*gopl.io/ch11/word1*

```Go
// Package word provides utilities for word games.
package word

// IsPalindrome reports whether s reads the same forward and backward.
// (Our first attempt.)
func IsPalindrome(s string) bool {
	for i := range s {
		if s[i] != s[len(s)-1-i] {
			return false
		}
	}
	return true
}
```

在相同的目录下，`word_test.go`测试文件中包含了`TestPalindrome`和`TestNonPalindrome`两个测试函数。每一个都是测试`IsPalindrome`是否给出正确的结果，并使用`t.Error`报告失败信息：

```Go
package word

import "testing"

func TestPalindrome(t *testing.T) {
	if !IsPalindrome("detartrated") {
		t.Error(`IsPalindrome("detartrated") = false`)
	}
	if !IsPalindrome("kayak") {
		t.Error(`IsPalindrome("kayak") = false`)
	}
}

func TestNonPalindrome(t *testing.T) {
	if IsPalindrome("palindrome") {
		t.Error(`IsPalindrome("palindrome") = true`)
	}
}
```

`go test`命令如果没有参数指定包那么将默认采用当前目录对应的包（和`go build`命令一样）。我们可以用下面的命令构建和运行测试。

```go
$ cd $GOPATH/src/gopl.io/ch11/word1
$ go test
ok   gopl.io/ch11/word1  0.008s
```

结果还比较满意，我们运行了这个程序， 不过没有提前退出是因为还没有遇到BUG报告。不过一个法国名为“`Noelle Eve Elleon`”的用户会抱怨`IsPalindrome`函数不能识别“`été`”。另外一个来自美国中部用户的抱怨则是不能识别“`A man, a plan, a canal: Panama.`”。执行特殊和小的BUG报告为我们提供了新的更自然的测试用例。

```Go
func TestFrenchPalindrome(t *testing.T) {
	if !IsPalindrome("été") {
		t.Error(`IsPalindrome("été") = false`)
	}
}

func TestCanalPalindrome(t *testing.T) {
	input := "A man, a plan, a canal: Panama"
	if !IsPalindrome(input) {
		t.Errorf(`IsPalindrome(%q) = false`, input)
	}
}
```

为了避免两次输入较长的字符串，我们使用了提供了有类似`Printf`格式化功能的 `Errorf`函数来汇报错误结果。

当添加了这两个测试用例之后，`go test`返回了测试失败的信息。

```go
$ go test
--- FAIL: TestFrenchPalindrome (0.00s)
    word_test.go:28: IsPalindrome("été") = false
--- FAIL: TestCanalPalindrome (0.00s)
    word_test.go:35: IsPalindrome("A man, a plan, a canal: Panama") = false
FAIL
FAIL    gopl.io/ch11/word1  0.014s
```

先编写测试用例并观察到测试用例触发了和用户报告的错误相同的描述是一个好的测试习惯。只有这样，我们才能定位我们要真正解决的问题。

先写测试用例的另外的好处是，运行测试通常会比手工描述报告的处理更快，这让我们可以进行快速地迭代。如果测试集有很多运行缓慢的测试，我们可以通过只选择运行某些特定的测试来加快测试速度。

参数`-v`可用于打印每个测试函数的名字和运行时间：

```go
$ go test -v
=== RUN TestPalindrome
--- PASS: TestPalindrome (0.00s)
=== RUN TestNonPalindrome
--- PASS: TestNonPalindrome (0.00s)
=== RUN TestFrenchPalindrome
--- FAIL: TestFrenchPalindrome (0.00s)
    word_test.go:28: IsPalindrome("été") = false
=== RUN TestCanalPalindrome
--- FAIL: TestCanalPalindrome (0.00s)
    word_test.go:35: IsPalindrome("A man, a plan, a canal: Panama") = false
FAIL
exit status 1
FAIL    gopl.io/ch11/word1  0.017s
```

参数`-run`对应一个正则表达式，只有测试函数名被它正确匹配的测试函数才会被`go test`测试命令运行：

```go
$ go test -v -run="French|Canal"
=== RUN TestFrenchPalindrome
--- FAIL: TestFrenchPalindrome (0.00s)
    word_test.go:28: IsPalindrome("été") = false
=== RUN TestCanalPalindrome
--- FAIL: TestCanalPalindrome (0.00s)
    word_test.go:35: IsPalindrome("A man, a plan, a canal: Panama") = false
FAIL
exit status 1
FAIL    gopl.io/ch11/word1  0.014s
```

当然，一旦我们已经修复了失败的测试用例，在我们提交代码更新之前，我们应该以不带参数的`go test`命令运行全部的测试用例，以确保修复失败测试的同时没有引入新的问题。

我们现在的任务就是修复这些错误。简要分析后发现第一个BUG的原因是我们采用了 `byte`而不是`rune`序列，所以像“`été`”中的`é`等非`ASCII`字符不能正确处理。第二个BUG是因为没有忽略空格和字母的大小写导致的。

针对上述两个BUG，我们仔细重写了函数：

*gopl.io/ch11/word2*

```Go
// Package word provides utilities for word games.
package word

import "unicode"

// IsPalindrome reports whether s reads the same forward and backward.
// Letter case is ignored, as are non-letters.
func IsPalindrome(s string) bool {
	var letters []rune
	for _, r := range s {
		if unicode.IsLetter(r) {
			letters = append(letters, unicode.ToLower(r))
		}
	}
	for i := range letters {
		if letters[i] != letters[len(letters)-1-i] {
			return false
		}
	}
	return true
}
```

同时我们也将之前的所有测试数据合并到了一个测试中的表格中。

```Go
func TestIsPalindrome(t *testing.T) {
	var tests = []struct {
		input string
		want  bool
	}{
		{"", true},
		{"a", true},
		{"aa", true},
		{"ab", false},
		{"kayak", true},
		{"detartrated", true},
		{"A man, a plan, a canal: Panama", true},
		{"Evil I did dwell; lewd did I live.", true},
		{"Able was I ere I saw Elba", true},
		{"été", true},
		{"Et se resservir, ivresse reste.", true},
		{"palindrome", false}, // non-palindrome
		{"desserts", false},   // semi-palindrome
	}
	for _, test := range tests {
		if got := IsPalindrome(test.input); got != test.want {
			t.Errorf("IsPalindrome(%q) = %v", test.input, got)
		}
	}
}
```

现在我们的新测试都通过了：

```go
$ go test gopl.io/ch11/word2
ok      gopl.io/ch11/word2      0.015s
```

这种表格驱动的测试在Go语言中很常见。我们可以很容易地向表格添加新的测试数据，并且后面的测试逻辑也没有冗余，这样我们可以有更多的精力去完善错误信息。

失败测试的输出并不包括调用`t.Errorf`时刻的堆栈调用信息。和其他编程语言或测试框架的`assert`断言不同，`t.Errorf`调用也没有引起`panic`异常或停止测试的执行。即使表格中前面的数据导致了测试的失败，表格后面的测试数据依然会运行测试，因此在一个测试中我们可能了解多个失败的信息。

如果我们真的需要停止测试，或许是因为初始化失败或可能是早先的错误导致了后续错误等原因，我们可以使用`t.Fatal`或`t.Fatalf`停止当前测试函数。它们必须在和测试函数同一个`goroutine`内调用。

测试失败的信息一般的形式是“`f(x) = y, want z`”，其中`f(x)`解释了失败的操作和对应的输入，`y`是实际的运行结果，`z`是期望的正确的结果。就像前面检查回文字符串的例子，实际的函数用于`f(x)`部分。显示`x`是表格驱动型测试中比较重要的部分，因为同一个断言可能对应不同的表格项执行多次。要避免无用和冗余的信息。在测试类似`IsPalindrome`返回布尔类型的函数时，可以忽略并没有额外信息的z部分。如果x、y或z是y的长度，输出一个相关部分的简明总结即可。测试的作者应该要努力帮助程序员诊断测试失败的原因。

**练习 11.1:** 为4.3节中的`charcount`程序编写测试。

```go
// charcount.go
package main

import (
	"bufio"
	"fmt"
	"io"
	"log"
	"os"
	"unicode"
)

func charCount(r io.Reader) (
	runes map[rune]int, props map[string]int, sizes map[int]int, invalid int) {
	runes = make(map[rune]int)   // char frequency
	props = make(map[string]int) // unicode.Properties frequency
	sizes = make(map[int]int)    // rune length frequency
	invalid = 0                  // invalid char frequency

	in := bufio.NewReader(r)
	for {
		r, n, err := in.ReadRune() // returns rune, nbytes, error
		if err == io.EOF {
			return
		}
		if err != nil {
			log.Fatal("charCount: %s", err)
		}
		if r == unicode.ReplacementChar && n == 1 {
			invalid++
			continue
		}
		for name, rangeTable := range unicode.Properties {
			if unicode.In(r, rangeTable) {
				props[name]++
			}
		}
		runes[r]++
		sizes[n]++
	}
}

func main() {
	runes, props, sizes, invalid := charCount(os.Stdin)
	fmt.Println(props)
	fmt.Printf("rune\tcount\n")
	for c, n := range runes {
		fmt.Printf("%q\t%d\n", c, n)
	}
	fmt.Print("\nlen\tcount\n")
	for i, n := range sizes {
		if i > 0 {
			fmt.Printf("%d\t%d\n", i, n)
		}
	}
	fmt.Printf("\n%-30s count\n", "category")
	for cat, n := range props {
		fmt.Printf("%-30s %d\n", cat, n)
	}
	if invalid > 0 {
		fmt.Printf("\n%d invalid UTF-8 characters\n", invalid)
	}
}
```

```go
// charcount_test.go
package main

import (
	"reflect"
	"strings"
	"testing"
)

func TestCharCount(t *testing.T) {
	tests := []struct {
		input   string
		runes   map[rune]int
		props   map[string]int
		sizes   map[int]int
		invalid int
	}{
		{
			input: "Hi, 世.",
			runes: map[rune]int{'H': 1, 'i': 1, ',': 1, ' ': 1, '世': 1, '.': 1},
			props: map[string]int{
				"Ideographic":          1,
				"Pattern_Syntax":       2,
				"Pattern_White_Space":  1,
				"STerm":                1,
				"Sentence_Terminal":    1,
				"Soft_Dotted":          1,
				"Terminal_Punctuation": 2,
				"Unified_Ideograph":    1,
				"White_Space":          1,
			},
			sizes:   map[int]int{1: 5, 3: 1},
			invalid: 0,
		},
	}
	for _, test := range tests {
		runes, props, sizes, invalid := charCount(strings.NewReader(test.input))
		if !reflect.DeepEqual(runes, test.runes) {
			t.Errorf("%q runes: got %v, want %v", test.input, runes, test.runes)
		}
		if !reflect.DeepEqual(props, test.props) {
			t.Errorf("%q props: got %v, want %v", test.input, props, test.props)
		}
		if !reflect.DeepEqual(sizes, test.sizes) {
			t.Errorf("%q sizes: got %v, want %v", test.input, sizes, test.sizes)
		}
		if invalid != test.invalid {
			t.Errorf("%q invalid: got %v, want %v", test.input, invalid, test.invalid)
		}
	}
}
```

**练习 11.2:** 为（§6.5）的`IntSet`编写一组测试，用于检查每个操作后的行为和基于内置`map`的集合等价，后面练习11.7将会用到。

```go
// intset.go
package main

type IntSet interface {
	Has(x int) bool
	Add(x int)
	AddAll(nums ...int)
	UnionWith(t IntSet)
	Len() int
	Remove(x int)
	Clear()
	Copy() IntSet
	String() string
	Ints() []int
}
```

```go
// bitintset.go
package main

import (
	"bytes"
	"fmt"
)

// An BitIntSet is a set of small non-negative integers.
// Its zero value represents the empty set.
type BitIntSet struct {
	words []uint64
}

// Has reports whether the set contains the non-negative value x.
func (s *BitIntSet) Has(x int) bool {
	word, bit := x/64, uint(x%64)
	return word < len(s.words) && s.words[word]&(1<<bit) != 0
}

// Add adds the non-negative value x to the set.
func (s *BitIntSet) Add(x int) {
	word, bit := x/64, uint(x%64)
	for word >= len(s.words) {
		s.words = append(s.words, 0)
	}
	s.words[word] |= 1 << bit
}

func (s *BitIntSet) AddAll(nums ...int) {
	for _, n := range nums {
		s.Add(n)
	}
}

// UnionWith sets s to the union of s and t.
func (s *BitIntSet) UnionWith(t IntSet) {
	if bis, ok := t.(*BitIntSet); ok {
		for i, tword := range bis.words {
			if i < len(s.words) {
				s.words[i] |= tword
			} else {
				s.words = append(s.words, tword)
			}
		}
	} else {
		for _, i := range t.Ints() {
			s.Add(i)
		}
	}
}

func popcount(x uint64) int {
	count := 0
	for x != 0 {
		count++
		x &= x - 1
	}
	return count
}

// return the number of elements
func (s *BitIntSet) Len() int {
	count := 0
	for _, word := range s.words {
		count += popcount(word)
	}
	return count
}

// remove x from the set
func (s *BitIntSet) Remove(x int) {
	word, bit := x/64, uint(x%64)
	s.words[word] &^= 1 << bit
}

// remove all elements from the set
func (s *BitIntSet) Clear() {
	for i := range s.words {
		s.words[i] = 0
	}
}

// return a copy of the set
func (s *BitIntSet) Copy() IntSet {
	new := &BitIntSet{}
	new.words = make([]uint64, len(s.words))
	copy(new.words, s.words)
	return new
}

// String returns the set as a string of the form "{1 2 3}".
func (s *BitIntSet) String() string {
	var buf bytes.Buffer
	buf.WriteByte('{')
	for i, word := range s.words {
		if word == 0 {
			continue
		}
		for j := 0; j < 64; j++ {
			if word&(1<<uint(j)) != 0 {
				if buf.Len() > len("{") {
					buf.WriteByte(' ')
				}
				fmt.Fprintf(&buf, "%d", 64*i+j)
			}
		}
	}
	buf.WriteByte('}')
	return buf.String()
}

func (s *BitIntSet) Ints() []int {
	var ints []int
	for i, word := range s.words {
		if word == 0 {
			continue
		}
		for j := 0; j < 64; j++ {
			if word&(1<<uint(j)) != 0 {
				ints = append(ints, 64*i+j)
			}
		}
	}
	return ints
}
```

````go
// mapintset.go
package main

import (
	"bytes"
	"fmt"
	"sort"
)

type MapIntSet struct {
	m map[int]bool
}

func NewMapIntSet() *MapIntSet {
	return &MapIntSet{map[int]bool{}}
}

func (s *MapIntSet) Has(x int) bool {
	return s.m[x]
}

func (s *MapIntSet) Add(x int) {
	s.m[x] = true
}

func (s *MapIntSet) AddAll(nums ...int) {
	for _, x := range nums {
		s.m[x] = true
	}
}

func (s *MapIntSet) UnionWith(t IntSet) {
	for _, x := range t.Ints() {
		s.m[x] = true
	}
}

func (s *MapIntSet) Len() int {
	return len(s.m)
}

func (s *MapIntSet) Remove(x int) {
	delete(s.m, x)
}

func (s *MapIntSet) Clear() {
	s.m = make(map[int]bool)
}

func (s *MapIntSet) Copy() IntSet {
	copy := make(map[int]bool)
	for k, v := range s.m {
		copy[k] = v
	}
	return &MapIntSet{copy}
}

func (s *MapIntSet) String() string {
	b := &bytes.Buffer{}
	b.WriteByte('{')
	for i, x := range s.Ints() {
		if i != 0 {
			b.WriteByte(' ')
		}
		fmt.Fprintf(b, "%d", x)
	}
	b.WriteByte('}')
	return b.String()
}

func (s *MapIntSet) Ints() []int {
	ints := make([]int, 0, len(s.m))
	for x := range s.m {
		ints = append(ints, x)
	}
	sort.IntSlice(ints).Sort()
	return ints
}
````

```go
// intset_test.go
package main

import (
	"testing"
)

func newIntSets() []IntSet {
	return []IntSet{&BitIntSet{}, NewMapIntSet()}
}

func TestLenZeroInitially(t *testing.T) {
	for _, s := range newIntSets() {
		if s.Len() != 0 {
			t.Errorf("%T.Len(): got %d, want 0", s, s.Len())
		}
	}
}

func TestLenAfterAddingElements(t *testing.T) {
	for _, s := range newIntSets() {
		s.Add(0)
		s.Add(2000)
		if s.Len() != 2 {
			t.Errorf("%T.Len(): got %d, want 2", s, s.Len())
		}
	}
}

func TestRemove(t *testing.T) {
	for _, s := range newIntSets() {
		s.Add(0)
		s.Remove(0)
		if s.Has(0) {
			t.Errorf("%T: want zero removed, got %s", s, s)
		}
	}
}

func TestClear(t *testing.T) {
	for _, s := range newIntSets() {
		s.Add(0)
		s.Add(1000)
		s.Clear()
		if s.Has(0) || s.Has(1000) {
			t.Errorf("%T: want empty set, got %s", s, s)
		}
	}
}

func TestCopy(t *testing.T) {
	for _, orig := range newIntSets() {
		orig.Add(1)
		copy := orig.Copy()
		copy.Add(2)
		if !copy.Has(1) || orig.Has(2) {
			t.Errorf("%T: want %s, got %s", orig, orig, copy)
		}
	}
}

func TestAddAll(t *testing.T) {
	for _, s := range newIntSets() {
		s.AddAll(0, 2, 4)
		if !s.Has(0) || !s.Has(2) || !s.Has(4) {
			t.Errorf("%T: want {2 4}, got %s", s, s)
		}
	}
}
```

### 11.2.1. 随机测试

表格驱动的测试便于构造基于精心挑选的测试数据的测试用例。另一种测试思路是随机测试，也就是通过构造更广泛的随机输入来测试探索函数的行为。

那么对于一个随机的输入，我们如何能知道希望的输出结果呢？这里有两种处理策略。第一个是编写另一个对照函数，使用简单和清晰的算法，虽然效率较低但是行为和要测试的函数是一致的，然后针对相同的随机输入检查两者的输出结果。第二种是生成的随机输入的数据遵循特定的模式，这样我们就可以知道期望的输出的模式。

下面的例子使用的是第二种方法：`randomPalindrome`函数用于随机生成回文字符串。

```Go
import "math/rand"

// randomPalindrome returns a palindrome whose length and contents
// are derived from the pseudo-random number generator rng.
func randomPalindrome(rng *rand.Rand) string {
	n := rng.Intn(25) // random length up to 24
	runes := make([]rune, n)
	for i := 0; i < (n+1)/2; i++ {
		r := rune(rng.Intn(0x1000)) // random rune up to '\u0999'
		runes[i] = r
		runes[n-1-i] = r
	}
	return string(runes)
}

func TestRandomPalindromes(t *testing.T) {
	// Initialize a pseudo-random number generator.
	seed := time.Now().UTC().UnixNano()
	t.Logf("Random seed: %d", seed)
	rng := rand.New(rand.NewSource(seed))

	for i := 0; i < 1000; i++ {
		p := randomPalindrome(rng)
		if !IsPalindrome(p) {
			t.Errorf("IsPalindrome(%q) = false", p)
		}
	}
}
```

虽然随机测试会有不确定因素，但是它也是至关重要的，我们可以从失败测试的日志获取足够的信息。在我们的例子中，输入`IsPalindrome`的`p`参数将告诉我们真实的数据，但是对于函数将接受更复杂的输入，不需要保存所有的输入，只要日志中简单地记录随机数种子即可（像上面的方式）。有了这些随机数初始化种子，我们可以很容易修改测试代码以重现失败的随机测试。

通过使用当前时间作为随机种子，在整个过程中的每次运行测试命令时都将探索新的随机数据。如果你使用的是定期运行的自动化测试集成系统，随机测试将特别有价值。

**练习 11.3:** `TestRandomPalindromes`测试函数只测试了回文字符串。编写新的随机测试生成器，用于测试随机生成的非回文字符串。

```go
// word.go
package word

import "unicode"

// IsPalindrome reports whether s reads the same forward and backward.
// Letter case is ignored, as are non-letters.
func IsPalindrome(s string) bool {
	var letters []rune
	for _, r := range s {
		if unicode.IsLetter(r) {
			letters = append(letters, unicode.ToLower(r))
		}
	}
	for i := range letters {
		if letters[i] != letters[len(letters)-1-i] {
			return false
		}
	}
	return true
}
```

```go
// palindrome_test.go
package word

import (
	"bytes"
	"fmt"
	"math/rand"
	"strings"
	"testing"
	"time"
	"unicode"
)

func TestRandomPalindromes(t *testing.T) {
	// Initialize a pseudo-random number generator.
	seed := time.Now().UTC().UnixNano()
	t.Logf("Random seed: %d", seed)
	rng := rand.New(rand.NewSource(seed))
	for i := 0; i < 1000; i++ {
		p := randomPalindrome(rng)
		if !IsPalindrome(p) {
			t.Errorf("IsPalindrome(%q) = false", p)
		}
	}
}

func TestRandomNonPalindromes(t *testing.T) {
	seed := time.Now().UTC().UnixNano()
	t.Logf("Random seed: %d", seed)
	rng := rand.New(rand.NewSource(seed))
	for i := 0; i < 500; i++ {
		np := randomNonPalindrome(rng)
		if IsPalindrome(np) {
			t.Errorf("IsPalindrome(%q) = true", np)
		}
	}
}

// randomPalindrome returns a palindrome whose length and contents are derived
// from the pseudo-random number generator rng.
func randomPalindrome(rng *rand.Rand) string {
	n := rng.Intn(25) // random length up to 24
	runes := make([]rune, n)
	for i := 0; i < (n+1)/2; i++ {
		r := rune(rng.Intn(0x1000)) // random rune up to '\u0999'
		runes[i] = r
		runes[n-1-i] = r
	}
	return string(runes)
}

// Using a bunch of ideas from Eli Bendersky's excellent article:
// http://eli.thegreenplace.net/2010/01/28/generating-random-sentences-from-a-context-free-grammar/
//
// NON = acb | ab | aNONa | aNONb | aPALb
// PAL = eps | a | aa | aPALa
//
// - a, b, and c are any letter, but b has a different value than a if they
//   appear together in a production, and multiple a's in a production share
//   the same value.
// - eps is the empty string
//
// Unlike the grammar Bendersky was playing with, the non-palindrome grammar
// here converges very quickly unless we increase the weights of the
// non-terminals. Also, decaying the probability of productions frequently chosen in
// a recursive call tree is unnecessary. We don't, but if we wanted to we could
// probably get a consistent range of lengths by adding decay and tweaking the
// weights, so that it's very likely for nonterminals to be chosen but then
// decay at a certain rate.
var grammar = map[string][]weighted{
	"NON": []weighted{
		{"a c b", 1},
		{"a b", 1},
		{"a NON a", 30},
		{"a NON b", 30},
		{"a PAL b", 30},
	}, "PAL": []weighted{
		{"eps ", 1},
		{"a ", 1},
		{"a a", 1},
		{"a PAL a", 40},
	},
}
var letters []rune

type weighted struct {
	s      string
	weight float64
}

func randomNonPalindrome(rng *rand.Rand) string {
	return expand("NON", rng)
}

func expand(symbol string, rng *rand.Rand) string {
	prod := choose(grammar[symbol], rng)
	buf := &bytes.Buffer{}

	var a rune
	for _, sym := range strings.Fields(prod) {
		if _, ok := grammar[sym]; ok { // recurse
			buf.WriteString(expand(sym, rng))
			continue
		}
		switch sym {
		case "a":
			if a == 0 {
				a = chooseLetter(rng)
			}
			buf.WriteRune(a)
		case "b":
			buf.WriteRune(chooseOtherLetter(a, rng))
		case "c":
			buf.WriteRune(chooseLetter(rng))
		case "eps":
			// nop: empty string
		default:
			panic(fmt.Sprintf("unexpected symbol %q", sym))
		}
	}
	return buf.String()
}

// choose returns a string from a slice, decaying the probability of those
// already-chosen.
func choose(choices []weighted, rng *rand.Rand) string {
	if len(choices) == 0 {
		panic("choose: no choices")
	}
	var sum float64
	for _, c := range choices {
		sum += c.weight
	}
	r := rng.Float64() * sum
	for _, c := range choices {
		r -= c.weight
		if r <= 0 {
			return c.s
		}
	}
	panic("choose: r was chosen incorrectly")
}

func chooseLetter(rng *rand.Rand) rune {
	return letters[rng.Intn(len(letters))]
}

// Choose a letter that isn't r or an upper/lowercase variant of r.
func chooseOtherLetter(r rune, rng *rand.Rand) rune {
	for {
		r2 := letters[rng.Intn(len(letters))]
		if unicode.ToLower(r2) == unicode.ToLower(r) {
			continue
		}
		return r2
	}
}

func init() {
	// Let's just stick with ASCII, since the odds of a font having some random
	// unicode codepoint seems pretty low.
	for r := rune(0x21); r < 0x7e; r++ {
		if unicode.IsLetter(r) {
			letters = append(letters, r)
		}
	}
	// Visualizing the algorithm might be easier with a small character set:
	// letters = []rune{'a', 'b'}
}
```

**练习 11.4:** 修改`randomPalindrome`函数，以探索`IsPalindrome`是否对标点和空格做了正确处理。

```go
// word.go
package word

import "unicode"

// IsPalindrome reports whether s reads the same forward and backward.
// Letter case is ignored, as are non-letters.
func IsPalindrome(s string) bool {
	var letters []rune
	for _, r := range s {
		if unicode.IsLetter(r) {
			letters = append(letters, unicode.ToLower(r))
		}
	}
	for i := range letters {
		if letters[i] != letters[len(letters)-1-i] {
			return false
		}
	}
	return true
}
```

```go
// palindrome_test.go
package word

import (
	"bytes"
	"fmt"
	"math/rand"
	"strings"
	"testing"
	"time"
	"unicode"
)

func TestRandomPalindromes(t *testing.T) {
	// Initialize a pseudo-random number generator.
	seed := time.Now().UTC().UnixNano()
	t.Logf("Random seed: %d", seed)
	rng := rand.New(rand.NewSource(seed))
	for i := 0; i < 1000; i++ {
		p := randomPalindrome(rng)
		if !IsPalindrome(p) {
			t.Errorf("IsPalindrome(%q) = false", p)
		}
	}
}

func TestRandomNonPalindromes(t *testing.T) {
	seed := time.Now().UTC().UnixNano()
	t.Logf("Random seed: %d", seed)
	rng := rand.New(rand.NewSource(seed))
	for i := 0; i < 500; i++ {
		np := randomNonPalindrome(rng)
		if IsPalindrome(np) {
			t.Errorf("IsPalindrome(%q) = true", np)
		}
		np = randomPunctuatedNonPalindrome(rng)
		if IsPalindrome(np) {
			t.Errorf("IsPalindrome(%q) = true", np)
		}
	}
}

// randomPalindrome returns a palindrome whose length and contents are derived
// from the pseudo-random number generator rng.
func randomPalindrome(rng *rand.Rand) string {
	n := rng.Intn(25) // random length up to 24
	runes := make([]rune, n)
	for i := 0; i < (n+1)/2; i++ {
		r := rune(rng.Intn(0x1000)) // random rune up to '\u0999'
		runes[i] = r
		runes[n-1-i] = r
	}
	return string(runes)
}

// Using a bunch of ideas from Eli Bendersky's excellent article:
// http://eli.thegreenplace.net/2010/01/28/generating-random-sentences-from-a-context-free-grammar/
//
// NON = acb | ab | aNONa | aNONb | aPALb
// PAL = eps | a | aa | aPALa
//
// - a, b, and c are any letter, but b has a different value than a if they
//   appear together in a production, and multiple a's in a production share
//   the same value.
// - eps is the empty string
//
// Unlike the grammar Bendersky was playing with, the non-palindrome grammar
// here converges very quickly unless we increase the weights of the
// non-terminals. Also, decaying the probability of productions frequently chosen in
// a recursive call tree is unnecessary. We don't, but if we wanted to we could
// probably get a consistent range of lengths by adding decay and tweaking the
// weights, so that it's very likely for nonterminals to be chosen but then
// decay at a certain rate.
var grammar = map[string][]weighted{
	"NON": []weighted{
		{"a c b", 1},
		{"a b", 1},
		{"a NON a", 30},
		{"a NON b", 30},
		{"a PAL b", 30},
	}, "PAL": []weighted{
		{"eps ", 1},
		{"a ", 1},
		{"a a", 1},
		{"a PAL a", 40},
	},
}
var letters []rune
var punctuation []rune
var punctProb = 0.1

type weighted struct {
	s      string
	weight float64
}

func randomNonPalindrome(rng *rand.Rand) string {
	return expand("NON", rng)
}

func randomPunctuatedNonPalindrome(rng *rand.Rand) string {
	b := &bytes.Buffer{}
	for _, r := range randomNonPalindrome(rng) {
		if rng.Float64() < punctProb {
			b.WriteRune(choosePunct(rng))
		}
		b.WriteRune(r)
	}
	return b.String()
}

func expand(symbol string, rng *rand.Rand) string {
	prod := choose(grammar[symbol], rng)
	buf := &bytes.Buffer{}

	var a rune
	for _, sym := range strings.Fields(prod) {
		if _, ok := grammar[sym]; ok { // recurse
			buf.WriteString(expand(sym, rng))
			continue
		}
		switch sym {
		case "a":
			if a == 0 {
				a = chooseLetter(rng)
			}
			buf.WriteRune(a)
		case "b":
			buf.WriteRune(chooseOtherLetter(a, rng))
		case "c":
			buf.WriteRune(chooseLetter(rng))
		case "eps":
			// nop: empty string
		default:
			panic(fmt.Sprintf("unexpected symbol %q", sym))
		}
	}
	return buf.String()
}

// choose returns a string from a slice, decaying the probability of those
// already-chosen.
func choose(choices []weighted, rng *rand.Rand) string {
	if len(choices) == 0 {
		panic("choose: no choices")
	}
	var sum float64
	for _, c := range choices {
		sum += c.weight
	}
	r := rng.Float64() * sum
	for _, c := range choices {
		r -= c.weight
		if r <= 0 {
			return c.s
		}
	}
	panic("choose: r was chosen incorrectly")
}

func chooseLetter(rng *rand.Rand) rune {
	return letters[rng.Intn(len(letters))]
}

// Choose a letter that isn't r or an upper/lowercase variant of r.
func chooseOtherLetter(r rune, rng *rand.Rand) rune {
	for {
		r2 := letters[rng.Intn(len(letters))]
		if unicode.ToLower(r2) == unicode.ToLower(r) {
			continue
		}
		return r2
	}
}

func choosePunct(rng *rand.Rand) rune {
	return punctuation[rng.Intn(len(punctuation))]
}

func init() {
	// Let's just stick with ASCII, since the odds of a font having some random
	// unicode codepoint seems pretty low.
	for r := rune(0x21); r < 0x7e; r++ {
		switch {
		case unicode.IsLetter(r):
			letters = append(letters, r)
		case unicode.IsPunct(r):
			punctuation = append(punctuation, r)
		}
	}
	// Visualizing the algorithm might be easier with a small character set:
	// letters = []rune{'a', 'b'}
}
```

> 译者注：**拓展阅读**感兴趣的读者可以再了解一下`go-fuzz`

### 11.2.2. 测试一个命令

对于测试包`go test`是一个有用的工具，但是稍加努力我们也可以用它来测试可执行程序。如果一个包的名字是 `main`，那么在构建时会生成一个可执行程序，不过`main`包可以作为一个包被测试器代码导入。

让我们为2.3.2节的`echo`程序编写一个测试。我们先将程序拆分为两个函数：`echo`函数完成真正的工作，`main`函数用于处理命令行输入参数和`echo`可能返回的错误。

*gopl.io/ch11/echo*

```Go
// Echo prints its command-line arguments.
package main

import (
	"flag"
	"fmt"
	"io"
	"os"
	"strings"
)

var (
	n = flag.Bool("n", false, "omit trailing newline")
	s = flag.String("s", " ", "separator")
)

var out io.Writer = os.Stdout // modified during testing

func main() {
	flag.Parse()
	if err := echo(!*n, *s, flag.Args()); err != nil {
		fmt.Fprintf(os.Stderr, "echo: %v\n", err)
		os.Exit(1)
	}
}

func echo(newline bool, sep string, args []string) error {
	fmt.Fprint(out, strings.Join(args, sep))
	if newline {
		fmt.Fprintln(out)
	}
	return nil
}
```

在测试中我们可以用各种参数和标志调用`echo`函数，然后检测它的输出是否正确，我们通过增加参数来减少`echo`函数对全局变量的依赖。我们还增加了一个全局名为`out`的变量来替代直接使用`os.Stdout`，这样测试代码可以根据需要将`out`修改为不同的对象以便于检查。下面就是`echo_test.go`文件中的测试代码：

```Go
package main

import (
	"bytes"
	"fmt"
	"testing"
)

func TestEcho(t *testing.T) {
	var tests = []struct {
		newline bool
		sep     string
		args    []string
		want    string
	}{
		{true, "", []string{}, "\n"},
		{false, "", []string{}, ""},
		{true, "\t", []string{"one", "two", "three"}, "one\ttwo\tthree\n"},
		{true, ",", []string{"a", "b", "c"}, "a,b,c\n"},
		{false, ":", []string{"1", "2", "3"}, "1:2:3"},
	}
	for _, test := range tests {
		descr := fmt.Sprintf("echo(%v, %q, %q)",
			test.newline, test.sep, test.args)

		out = new(bytes.Buffer) // captured output
		if err := echo(test.newline, test.sep, test.args); err != nil {
			t.Errorf("%s failed: %v", descr, err)
			continue
		}
		got := out.(*bytes.Buffer).String()
		if got != test.want {
			t.Errorf("%s = %q, want %q", descr, got, test.want)
		}
	}
}
```

要注意的是测试代码和产品代码在同一个包。虽然是`main`包，也有对应的`main`入口函数，但是在测试的时候`main`包只是`TestEcho`测试函数导入的一个普通包，里面`main`函数并没有被导出，而是被忽略的。

通过将测试放到表格中，我们很容易添加新的测试用例。让我通过增加下面的测试用例来看看失败的情况是怎么样的：

```Go
{true, ",", []string{"a", "b", "c"}, "a b c\n"}, // NOTE: wrong expectation!
```

`go test`输出如下：

```go
$ go test gopl.io/ch11/echo
--- FAIL: TestEcho (0.00s)
    echo_test.go:31: echo(true, ",", ["a" "b" "c"]) = "a,b,c", want "a b c\n"
FAIL
FAIL        gopl.io/ch11/echo         0.006s
```

错误信息描述了尝试的操作（使用Go类似语法），实际的结果和期望的结果。通过这样的错误信息，你可以在检视代码之前就很容易定位错误的原因。

要注意的是在测试代码中并没有调用`log.Fatal`或`os.Exit`，因为调用这类函数会导致程序提前退出；调用这些函数的特权应该放在`main`函数中。如果真的有意外的事情导致函数发生`panic`异常，测试驱动应该尝试用`recover`捕获异常，然后将当前测试当作失败处理。如果是可预期的错误，例如非法的用户输入、找不到文件或配置文件不当等应该通过返回一个非空的`error`的方式处理。幸运的是（上面的意外只是一个插曲），我们的`echo`示例是比较简单的也没有需要返回非空`error`的情况。
### 11.2.3. 白盒测试

一种测试分类的方法是基于测试者是否需要了解被测试对象的内部工作原理。黑盒测试只需要测试包公开的文档和API行为，内部实现对测试代码是透明的。相反，白盒测试有访问包内部函数和数据结构的权限，因此可以做到一些普通客户端无法实现的测试。例如，一个白盒测试可以在每个操作之后检测不变量的数据类型。（白盒测试只是一个传统的名称，其实称为clear box测试会更准确。）

黑盒和白盒这两种测试方法是互补的。黑盒测试一般更健壮，随着软件实现的完善测试代码很少需要更新。它们可以帮助测试者了解真实客户的需求，也可以帮助发现API设计的一些不足之处。相反，白盒测试则可以对内部一些棘手的实现提供更多的测试覆盖。

我们已经看到两种测试的例子。`TestIsPalindrome`测试仅仅使用导出的`IsPalindrome`函数，因此这是一个黑盒测试。`TestEcho`测试则调用了内部的`echo`函数，并且更新了内部的`out`包级变量，这两个都是未导出的，因此这是白盒测试。

当我们准备`TestEcho`测试的时候，我们修改了`echo`函数使用包级的`out`变量作为输出对象，因此测试代码可以用另一个实现代替标准输出，这样可以方便对比`echo`输出的数据。使用类似的技术，我们可以将产品代码的其他部分也替换为一个容易测试的伪对象。使用伪对象的好处是我们可以方便配置，容易预测，更可靠，也更容易观察。同时也可以避免一些不良的副作用，例如更新生产数据库或信用卡消费行为。

下面的代码演示了为用户提供网络存储的web服务中的配额检测逻辑。当用户使用了超过`90%`的存储配额之后将发送提醒邮件。

> （译注：一般在实现业务机器监控，包括磁盘、cpu、网络等的时候，需要类似的到达阈值=>触发报警的逻辑，所以是很实用的案例。）

*gopl.io/ch11/storage1*

```Go
package storage

import (
	"fmt"
	"log"
	"net/smtp"
)

func bytesInUse(username string) int64 { return 0 /* ... */ }

// Email sender configuration.
// NOTE: never put passwords in source code!
const sender = "notifications@example.com"
const password = "correcthorsebatterystaple"
const hostname = "smtp.example.com"

const template = `Warning: you are using %d bytes of storage,
%d%% of your quota.`

func CheckQuota(username string) {
	used := bytesInUse(username)
	const quota = 1000000000 // 1GB
	percent := 100 * used / quota
	if percent < 90 {
		return // OK
	}
	msg := fmt.Sprintf(template, used, percent)
	auth := smtp.PlainAuth("", sender, password, hostname)
	err := smtp.SendMail(hostname+":587", auth, sender,
		[]string{username}, []byte(msg))
	if err != nil {
		log.Printf("smtp.SendMail(%s) failed: %s", username, err)
	}
}
```

我们想测试这段代码，但是我们并不希望发送真实的邮件。因此我们将邮件处理逻辑放到一个私有的`notifyUser`函数中。

*gopl.io/ch11/storage2*

```Go
var notifyUser = func(username, msg string) {
	auth := smtp.PlainAuth("", sender, password, hostname)
	err := smtp.SendMail(hostname+":587", auth, sender,
		[]string{username}, []byte(msg))
	if err != nil {
		log.Printf("smtp.SendEmail(%s) failed: %s", username, err)
	}
}

func CheckQuota(username string) {
	used := bytesInUse(username)
	const quota = 1000000000 // 1GB
	percent := 100 * used / quota
	if percent < 90 {
		return // OK
	}
	msg := fmt.Sprintf(template, used, percent)
	notifyUser(username, msg)
}
```

现在我们可以在测试中用伪邮件发送函数替代真实的邮件发送函数。它只是简单记录要通知的用户和邮件的内容。

```Go
package storage

import (
	"strings"
	"testing"
)
func TestCheckQuotaNotifiesUser(t *testing.T) {
	var notifiedUser, notifiedMsg string
	notifyUser = func(user, msg string) {
		notifiedUser, notifiedMsg = user, msg
	}

	// ...simulate a 980MB-used condition...

	const user = "joe@example.org"
	CheckQuota(user)
	if notifiedUser == "" && notifiedMsg == "" {
		t.Fatalf("notifyUser not called")
	}
	if notifiedUser != user {
		t.Errorf("wrong user (%s) notified, want %s",
			notifiedUser, user)
	}
	const wantSubstring = "98% of your quota"
	if !strings.Contains(notifiedMsg, wantSubstring) {
		t.Errorf("unexpected notification message <<%s>>, "+
			"want substring %q", notifiedMsg, wantSubstring)
	}
}
```

这里有一个问题：当测试函数返回后，`CheckQuota`将不能正常工作，因为`notifyUsers`依然使用的是测试函数的伪发送邮件函数（当更新全局对象的时候总会有这种风险）。 我们必须修改测试代码恢复`notifyUsers`原先的状态以便后续其他的测试没有影响，要确保所有的执行路径后都能恢复，包括测试失败或`panic`异常的情形。在这种情况下，我们建议使用`defer`语句来延后执行处理恢复的代码。

```Go
func TestCheckQuotaNotifiesUser(t *testing.T) {
	// Save and restore original notifyUser.
	saved := notifyUser
	defer func() { notifyUser = saved }()

	// Install the test's fake notifyUser.
	var notifiedUser, notifiedMsg string
	notifyUser = func(user, msg string) {
		notifiedUser, notifiedMsg = user, msg
	}
	// ...rest of test...
}
```

这种处理模式可以用来暂时保存和恢复所有的全局变量，包括命令行标志参数、调试选项和优化参数；安装和移除导致生产代码产生一些调试信息的钩子函数；还有有些诱导生产代码进入某些重要状态的改变，比如超时、错误，甚至是一些刻意制造的并发行为等因素。

以这种方式使用全局变量是安全的，因为`go test`命令并不会同时并发地执行多个测试。
### 11.2.4. 外部测试包

考虑下这两个包：`net/url`包，提供了URL解析的功能；`net/http`包，提供了web服务和HTTP客户端的功能。如我们所料，上层的`net/http`包依赖下层的`net/url`包。然后，`net/url`包中的一个测试是演示不同URL和HTTP客户端的交互行为。也就是说，一个下层包的测试代码导入了上层的包。

![测试函数 - 图1](images\ch11-01-1579484646826.png)

这样的行为在`net/url`包的测试代码中会导致包的循环依赖，正如图11.1中向上箭头所示，同时正如我们在10.1节所讲的，Go语言规范是禁止包的循环依赖的。

不过我们可以通过外部测试包的方式解决循环依赖的问题，也就是在`net/url`包所在的目录声明一个独立的`url_test`测试包。其中包名的`_test`后缀告诉`go test`工具它应该建立一个额外的包来运行测试。我们将这个外部测试包的导入路径视作是`net/url_test`会更容易理解，但实际上它并不能被其他任何包导入。

因为外部测试包是一个独立的包，所以能够导入那些`依赖待测代码本身`的其他辅助包；包内的测试代码就无法做到这点。在设计层面，外部测试包是在所有它依赖的包的上层，正如图11.2所示。

![测试函数 - 图2](images\ch11-02-1579484658009.png)

通过避免循环的导入依赖，外部测试包可以更灵活地编写测试，特别是集成测试（需要测试多个组件之间的交互），可以像普通应用程序那样自由地导入其他包。

我们可以用`go list`命令查看包对应目录中哪些Go源文件是产品代码，哪些是包内测试，还有哪些是外部测试包。我们以`fmt`包作为一个例子：`GoFiles`表示产品代码对应的Go源文件列表；也就是`go build`命令要编译的部分。

```go
$ go list -f={{.GoFiles}} fmt
[doc.go format.go print.go scan.go]
```

`TestGoFiles`表示的是`fmt`包内部测试代码，以`_test.go`为后缀文件名，不过只在测试时被构建：

```go
$ go list -f={{.TestGoFiles}} fmt
[export_test.go]
```

包的测试代码通常都在这些文件中，不过`fmt`包并非如此；稍后我们再解释`export_test.go`文件的作用。

`XTestGoFiles`表示的是属于外部测试包的测试代码，也就是`fmt_test`包，因此它们必须先导入`fmt`包。同样，这些文件也只是在测试时被构建运行：

```go
$ go list -f={{.XTestGoFiles}} fmt
[fmt_test.go scan_test.go stringer_test.go]
```

有时候外部测试包也需要访问被测试包内部的代码，例如在一个为了避免循环导入而被独立到外部测试包的白盒测试。在这种情况下，我们可以通过一些技巧解决：我们在包内的一个`_test.go`文件中导出一个内部的实现给外部测试包。因为这些代码只有在测试时才需要，因此一般会放在`export_test.go`文件中。

例如，`fmt`包的`fmt.Scanf`函数需要`unicode.IsSpace`函数提供的功能。但是为了避免太多的依赖，`fmt`包并没有导入包含巨大表格数据的`unicode`包；相反`fmt`包有一个叫`isSpace`内部的简易实现。

为了确保`fmt.isSpace`和`unicode.IsSpace`函数的行为保持一致，`fmt`包谨慎地包含了一个测试。一个在外部测试包内的白盒测试，是无法直接访问到`isSpace`内部函数的，因此`fmt`通过一个后门导出了`isSpace`函数。`export_test.go`文件就是专门用于外部测试包的后门。

```Go
package fmt

var IsSpace = isSpace
```

这个测试文件并没有定义测试代码；它只是通过`fmt.IsSpace`简单导出了内部的`isSpace`函数，提供给外部测试包使用。这个技巧可以广泛用于位于外部测试包的白盒测试。

### 11.2.5. 编写有效的测试

许多Go语言新人会惊异于Go语言极简的测试框架。很多其它语言的测试框架都提供了识别测试函数的机制（通常使用反射或元数据），通过设置一些“`setup`”和“`teardown`”的钩子函数来执行测试用例运行的初始化和之后的清理操作，同时测试工具箱还提供了很多类似`assert`断言、值比较函数、格式化输出错误信息和停止一个失败的测试等辅助函数（通常使用异常机制）。虽然这些机制可以使得测试非常简洁，但是测试输出的日志却会像火星文一般难以理解。此外，虽然测试最终也会输出`PASS`或`FAIL`的报告，但是它们提供的信息格式却非常不利于代码维护者快速定位问题，因为失败信息的具体含义非常隐晦，比如“`assert: 0 == 1`”或成页的海量跟踪日志。

Go语言的测试风格则形成鲜明对比。它期望测试者自己完成大部分的工作，定义函数避免重复，就像普通编程那样。编写测试并不是一个机械的填空过程；一个测试也有自己的接口，尽管它的维护者也是测试仅有的一个用户。一个好的测试不应该引发其他无关的错误信息，它只要清晰简洁地描述问题的症状即可，有时候可能还需要一些上下文信息。在理想情况下，维护者可以在不看代码的情况下就能根据错误信息定位错误产生的原因。一个好的测试不应该在遇到一点小错误时就立刻退出测试，它应该尝试报告更多的相关的错误信息，因为我们可能从多个失败测试的模式中发现错误产生的规律。

下面的断言函数比较两个值，然后生成一个通用的错误信息，并停止程序。它很好用也确实有效，但是当测试失败的时候，打印的错误信息却几乎是没有价值的。它并没有为快速解决问题提供一个很好的入口。

```Go
import (
	"fmt"
	"strings"
	"testing"
)
// A poor assertion function.
func assertEqual(x, y int) {
	if x != y {
		panic(fmt.Sprintf("%d != %d", x, y))
	}
}
func TestSplit(t *testing.T) {
	words := strings.Split("a:b:c", ":")
	assertEqual(len(words), 3)
	// ...
}
```

从这个意义上说，断言函数犯了过早抽象的错误：仅仅测试两个整数是否相同，而没能根据上下文提供更有意义的错误信息。我们可以根据具体的错误打印一个更有价值的错误信息，就像下面例子那样。只有在测试中出现重复模式时才采用抽象。

```Go
func TestSplit(t *testing.T) {
	s, sep := "a:b:c", ":"
	words := strings.Split(s, sep)
	if got, want := len(words), 3; got != want {
		t.Errorf("Split(%q, %q) returned %d words, want %d",
			s, sep, got, want)
	}
	// ...
}
```

现在的测试不仅报告了调用的具体函数、它的输入和结果的意义；并且打印的真实返回的值和期望返回的值；并且即使断言失败依然会继续尝试运行更多的测试。一旦我们写了这样结构的测试，下一步自然不是用更多的`if`语句来扩展测试用例，我们可以用像`IsPalindrome`的表驱动测试那样来准备更多的`s`和`sep`测试用例。

前面的例子并不需要额外的辅助函数，如果有可以使测试代码更简单的方法我们也乐意接受。（我们将在13.3节看到一个类似`reflect.DeepEqual`辅助函数。）一个好的测试的关键是首先实现你期望的具体行为，然后才是考虑简化测试代码、避免重复。如果直接从抽象、通用的测试库着手，很难取得良好结果。

**练习11.5:** 用表格驱动的技术扩展`TestSplit`测试，并打印期望的输出结果。

```go
// split_test.go
package main

import (
	"strings"
	"testing"
)

func TestSplit(t *testing.T) {
	tests := []struct {
		s, sep string
		want   int
	}{
		{"a:b:c", ":", 3},
	}
	for _, test := range tests {
		words := strings.Split(test.s, test.sep)
		if got := len(words); got != test.want {
			t.Errorf("Split(%q, %q) returned %d words, want %d",
				test.s, test.sep, got, test.want)
		}
	}
}
```


### 11.2.6. 避免脆弱的测试

如果一个应用程序对于新出现的但有效的输入经常失败说明程序容易出bug（不够稳健）；同样，如果一个测试仅仅对程序做了微小变化就失败则称为脆弱。就像一个不够稳健的程序会挫败它的用户一样，一个脆弱的测试同样会激怒它的维护者。最脆弱的测试代码会在程序没有任何变化的时候产生不同的结果，时好时坏，处理它们会耗费大量的时间但是并不会得到任何好处。

当一个测试函数会产生一个复杂的输出如一个很长的字符串、一个精心设计的数据结构或一个文件时，人们很容易想预先写下一系列固定的用于对比的标杆数据。但是随着项目的发展，有些输出可能会发生变化，尽管很可能是一个改进的实现导致的。而且不仅仅是输出部分，函数复杂的输入部分可能也跟着变化了，因此测试使用的输入也就不再有效了。

避免脆弱测试代码的方法是只检测你真正关心的属性。保持测试代码的简洁和内部结构的稳定。特别是对断言部分要有所选择。不要对字符串进行全字匹配，而是针对那些在项目的发展中是比较稳定不变的子串。很多时候值得花力气来编写一个从复杂输出中提取用于断言的必要信息的函数，虽然这可能会带来很多前期的工作，但是它可以帮助迅速及时修复因为项目演化而导致的不合逻辑的失败测试。

## 11.3. 测试覆盖率

就其性质而言，测试不可能是完整的。计算机科学家Edsger Dijkstra曾说过：“测试能证明缺陷存在，而无法证明没有缺陷。”再多的测试也不能证明一个程序没有BUG。在最好的情况下，测试可以增强我们的信心：代码在很多重要场景下是可以正常工作的。

对待测程序执行的测试的程度称为测试的覆盖率。测试覆盖率并不能量化——即使最简单的程序的动态也是难以精确测量的——但是有启发式方法来帮助我们编写有效的测试代码。

这些启发式方法中，语句的覆盖率是最简单和最广泛使用的。语句的覆盖率是指在测试中至少被运行一次的代码占总代码数的比例。在本节中，我们使用`go test`命令中集成的测试覆盖率工具，来度量下面代码的测试覆盖率，帮助我们识别测试和我们期望间的差距。

下面的代码是一个表格驱动的测试，用于测试第七章的表达式求值程序：

*gopl.io/ch7/eval*

```Go
func TestCoverage(t *testing.T) {
	var tests = []struct {
		input string
		env   Env
		want  string // expected error from Parse/Check or result from Eval
	}{
		{"x % 2", nil, "unexpected '%'"},
		{"!true", nil, "unexpected '!'"},
		{"log(10)", nil, `unknown function "log"`},
		{"sqrt(1, 2)", nil, "call to sqrt has 2 args, want 1"},
		{"sqrt(A / pi)", Env{"A": 87616, "pi": math.Pi}, "167"},
		{"pow(x, 3) + pow(y, 3)", Env{"x": 9, "y": 10}, "1729"},
		{"5 / 9 * (F - 32)", Env{"F": -40}, "-40"},
	}

	for _, test := range tests {
		expr, err := Parse(test.input)
		if err == nil {
			err = expr.Check(map[Var]bool{})
		}
		if err != nil {
			if err.Error() != test.want {
				t.Errorf("%s: got %q, want %q", test.input, err, test.want)
			}
			continue
		}
		got := fmt.Sprintf("%.6g", expr.Eval(test.env))
		if got != test.want {
			t.Errorf("%s: %v => %s, want %s",
				test.input, test.env, got, test.want)
		}
	}
}
```

首先，我们要确保所有的测试都正常通过：

```go
$ go test -v -run=Coverage gopl.io/ch7/eval
=== RUN TestCoverage
--- PASS: TestCoverage (0.00s)
PASS
ok      gopl.io/ch7/eval         0.011s
```

下面这个命令可以显示测试覆盖率工具的使用用法：

```go
$ go tool cover
Usage of 'go tool cover':
Given a coverage profile produced by 'go test':
    go test -coverprofile=c.out

Open a web browser displaying annotated source code:
    go tool cover -html=c.out
...
```

`go tool`命令运行Go工具链的底层可执行程序。这些底层可执行程序放在`$GOROOT/pkg/tool/${GOOS}_${GOARCH}`目录。因为有`go build`命令的原因，我们很少直接调用这些底层工具。

现在我们可以用`-coverprofile`标志参数重新运行测试：

```
$ go test -run=Coverage -coverprofile=c.out gopl.io/ch7/eval
ok      gopl.io/ch7/eval         0.032s      coverage: 68.5% of statements
```

这个标志参数通过在测试代码中插入生成钩子来统计覆盖率数据。也就是说，在运行每个测试前，它将待测代码拷贝一份并做修改，在每个词法块都会设置一个布尔标志变量。当被修改后的被测试代码运行退出时，将统计日志数据写入`c.out`文件，并打印一部分执行的语句的一个总结。（如果你需要的是摘要，使用`go test -cover`。）

如果使用了`-covermode=count`标志参数，那么将在每个代码块插入一个计数器而不是布尔标志量。在统计结果中记录了每个块的执行次数，这可以用于衡量哪些是被频繁执行的热点代码。

为了收集数据，我们运行了测试覆盖率工具，打印了测试日志，生成一个HTML报告，然后在浏览器中打开（图11.3）。

```go
$ go tool cover -html=c.out
```

![测试覆盖率 - 图1](images\ch11-03-1579484680806.png)

绿色的代码块被测试覆盖到了，红色的则表示没有被覆盖到。为了清晰起见，我们将背景红色文本的背景设置成了阴影效果。我们可以马上发现`unary`操作的`Eval`方法并没有被执行到。如果我们针对这部分未被覆盖的代码添加下面的测试用例，然后重新运行上面的命令，那么我们将会看到那个红色部分的代码也变成绿色了：

```go
{"-x * -x", eval.Env{"x": 2}, "4"}
```

不过两个`panic`语句依然是红色的。这是没有问题的，因为这两个语句并不会被执行到。

实现`100%`的测试覆盖率听起来很美，但是在具体实践中通常是不可行的，也不是值得推荐的做法。因为那只能说明代码被执行过而已，并不意味着代码就是没有BUG的；因为对于逻辑复杂的语句需要针对不同的输入执行多次。有一些语句，例如上面的`panic`语句则永远都不会被执行到。另外，还有一些隐晦的错误在现实中很少遇到也很难编写对应的测试代码。测试从本质上来说是一个比较务实的工作，编写测试代码和编写应用代码的成本对比是需要考虑的。测试覆盖率工具可以帮助我们快速识别测试薄弱的地方，但是设计好的测试用例和编写应用代码一样需要严密的思考。
## 11.4. 基准测试

基准测试是测量一个程序在固定工作负载下的性能。在Go语言中，基准测试函数和普通测试函数写法类似，但是以`Benchmark`为前缀名，并且带有一个`*testing.B`类型的参数；`*testing.B`参数除了提供和`*testing.T`类似的方法，还有额外一些和性能测量相关的方法。它还提供了一个整数`N`，用于指定操作执行的循环次数。

下面是`IsPalindrome`函数的基准测试，其中循环将执行`N`次。

```Go
import "testing"

func BenchmarkIsPalindrome(b *testing.B) {
	for i := 0; i < b.N; i++ {
		IsPalindrome("A man, a plan, a canal: Panama")
	}
}
```

我们用下面的命令运行基准测试。和普通测试不同的是，默认情况下不运行任何基准测试。我们需要通过`-bench`命令行标志参数手工指定要运行的基准测试函数。该参数是一个正则表达式，用于匹配要执行的基准测试函数的名字，默认值是空的。其中“.”模式将可以匹配所有基准测试函数，但因为这里只有一个基准测试函数，因此和`-bench=IsPalindrome`参数是等价的效果。

```go
$ cd $GOPATH/src/gopl.io/ch11/word2
$ go test -bench=.
PASS
BenchmarkIsPalindrome-8 1000000                1035 ns/op
ok      gopl.io/ch11/word2      2.179s
```

结果中基准测试名的数字后缀部分，这里是8，表示运行时对应的`GOMAXPROCS`的值，这对于一些与并发相关的基准测试是重要的信息。

报告显示每次调用`IsPalindrome`函数花费1.035微秒，是执行1,000,000次的平均时间。因为基准测试驱动器开始时并不知道每个基准测试函数运行所花的时间，它会尝试在真正运行基准测试前先尝试用较小的`N`运行测试来估算基准测试函数所需要的时间，然后推断一个较大的时间保证稳定的测量结果。

循环在基准测试函数内实现，而不是放在基准测试框架内实现，这样可以让每个基准测试函数有机会在循环启动前执行初始化代码，这样并不会显著影响每次迭代的平均运行时间。如果还是担心初始化代码部分对测量时间带来干扰，那么可以通过`testing.B`参数提供的方法来临时关闭或重置计时器，不过这些一般很少会用到。

现在我们有了一个基准测试和普通测试，我们可以很容易测试改进程序运行速度的想法。也许最明显的优化是在`IsPalindrome`函数中第二个循环的停止检查，这样可以避免每个比较都做两次：

```Go
n := len(letters)/2
for i := 0; i < n; i++ {
	if letters[i] != letters[len(letters)-1-i] {
		return false
	}
}
return true
```

不过很多情况下，一个显而易见的优化未必能带来预期的效果。这个改进在基准测试中只带来了4%的性能提升。

```go
$ go test -bench=.
PASS
BenchmarkIsPalindrome-8 1000000              992 ns/op
ok      gopl.io/ch11/word2      2.093s
```

另一个改进想法是在开始为每个字符预先分配一个足够大的数组，这样就可以避免在`append`调用时可能会导致内存的多次重新分配。声明一个`letters`数组变量，并指定合适的大小，像下面这样，

```Go
letters := make([]rune, 0, len(s))
for _, r := range s {
	if unicode.IsLetter(r) {
		letters = append(letters, unicode.ToLower(r))
	}
}
```

这个改进提升性能约`35%`，报告结果是基于`2,000,000`次迭代的平均运行时间统计。

```go
$ go test -bench=.
PASS
BenchmarkIsPalindrome-8 2000000                      697 ns/op
ok      gopl.io/ch11/word2      1.468s
```

如这个例子所示，快的程序往往是伴随着较少的内存分配。`-benchmem`命令行标志参数将在报告中包含内存的分配数据统计。我们可以比较优化前后内存的分配情况：

```go
$ go test -bench=. -benchmem
PASS
BenchmarkIsPalindrome    1000000   1026 ns/op    304 B/op  4 allocs/op
```

这是优化之后的结果：

```go
$ go test -bench=. -benchmem
PASS
BenchmarkIsPalindrome    2000000    807 ns/op    128 B/op  1 allocs/op
```

用一次内存分配代替多次的内存分配节省了75%的分配调用次数和减少近一半的内存需求。

这个基准测试告诉了我们某个具体操作所需的绝对时间，但我们往往想知道的是两个不同的操作的时间对比。例如，如果一个函数需要1ms处理1,000个元素，那么处理10000或1百万将需要多少时间呢？这样的比较揭示了渐近增长函数的运行时间。另一个例子：I/O缓存该设置为多大呢？基准测试可以帮助我们选择在性能达标情况下所需的最小内存。第三个例子：对于一个确定的工作哪种算法更好？基准测试可以评估两种不同算法对于相同的输入在不同的场景和负载下的优缺点。

比较型的基准测试就是普通程序代码。它们通常是单参数的函数，由几个不同数量级的基准测试函数调用，就像这样：

```Go
func benchmark(b *testing.B, size int) { /* ... */ }
func Benchmark10(b *testing.B)         { benchmark(b, 10) }
func Benchmark100(b *testing.B)        { benchmark(b, 100) }
func Benchmark1000(b *testing.B)       { benchmark(b, 1000) }
```

通过函数参数来指定输入的大小，但是参数变量对于每个具体的基准测试都是固定的。要避免直接修改b.N来控制输入的大小。除非你将它作为一个固定大小的迭代计算输入，否则基准测试的结果将毫无意义。

比较型的基准测试反映出的模式在程序设计阶段是很有帮助的，但是即使程序完工了也应当保留基准测试代码。因为随着项目的发展，或者是输入的增加，或者是部署到新的操作系统或不同的处理器，我们可以再次用基准测试来帮助我们改进设计。

**练习 11.6:** 为2.6.2节的练习2.4和练习2.5的`PopCount`函数编写基准测试。看看基于表格算法在不同情况下对提升性能会有多大帮助。

```go
// popcount_test.go
package popcount

import (
	"testing"
)

func PopCountShiftMask(x uint64) int {
	count := 0
	mask := uint64(1)
	for i := 0; i < 64; i++ {
		if x&mask > 0 {
			count++
		}
		mask <<= 1
	}
	return count
}

func PopCountShiftValue(x uint64) int {
	count := 0
	mask := uint64(1)
	for i := 0; i < 64; i++ {
		if x&mask > 0 {
			count++
		}
		x >>= 1
	}
	return count
}

func PopCountClearRightmost(x uint64) int {
	count := 0
	for x != 0 {
		x &= x - 1
		count++
	}
	return count
}

// pc[i] is the population count of i.
var pc [256]byte

func init() {
	for i := range pc {
		pc[i] = pc[i/2] + byte(i&1)
	}
}

// PopCount returns the population count (number of set bits) of x.
func PopCountTable(x uint64) int {
	return int(pc[byte(x>>(0*8))] +
		pc[byte(x>>(1*8))] +
		pc[byte(x>>(2*8))] +
		pc[byte(x>>(3*8))] +
		pc[byte(x>>(4*8))] +
		pc[byte(x>>(5*8))] +
		pc[byte(x>>(6*8))] +
		pc[byte(x>>(7*8))])
}

func bench(b *testing.B, f func(uint64) int) {
	for i := 0; i < b.N; i++ {
		f(uint64(i))
	}
}

func benchN(b *testing.B, n int, f func(uint64) int) {
	for i := 0; i < b.N; i++ {
		for j := 0; j < n; j++ {
			f(uint64(j))
		}
	}
}

func benchTableN(b *testing.B, n int) {
	for i := range pc {
		pc[i] = pc[i/2] + byte(i&1)
	}
	for i := 0; i < b.N; i++ {
		for j := 0; j < n; j++ {
			PopCountTable(uint64(j))
		}
	}
}

func BenchmarkTable(b *testing.B) {
	bench(b, PopCountTable)
}

func BenchmarkShiftMask(b *testing.B) {
	bench(b, PopCountShiftMask)
}

func BenchmarkShiftValue(b *testing.B) {
	bench(b, PopCountShiftValue)
}

func BenchmarkClearRightmost(b *testing.B) {
	bench(b, PopCountClearRightmost)
}

// ClearRightMost should be cheaper than Table (including the table creation)
// if only a few shifts are being performed. Determine how many popcounts make
// Table worth it.
//
// Based on these results the break-even point for Table is slightly over 1000
// invocations versus ClearRightmost and is always cheaper than the shift-based
// methods.
//
// BenchmarkClearRightmost1-4      500000000                4.03 ns/op
// BenchmarkClearRightmost10-4     50000000                37.1 ns/op
// BenchmarkClearRightmost100-4     3000000               538 ns/op
// BenchmarkClearRightmost1000-4     200000              7494 ns/op
// BenchmarkClearRightmost10000-4     20000             92601 ns/op
// BenchmarkTable1-4               200000000                7.66 ns/op
// BenchmarkTable10-4              20000000                79.2 ns/op
// BenchmarkTable100-4              2000000               793 ns/op
// BenchmarkTable1000-4              200000              7975 ns/op
// BenchmarkTable10000-4              20000             79473 ns/op
// BenchmarkShiftValue1-4          20000000                78.3 ns/op
// BenchmarkShiftValue10-4          2000000               782 ns/op
// BenchmarkShiftValue100-4          200000              9203 ns/op
// BenchmarkShiftValue1000-4          10000            119000 ns/op
// BenchmarkShiftValue10000-4          1000           1289813 ns/op
// BenchmarkShiftMask1-4           20000000                78.3 ns/op
// BenchmarkShiftMask10-4           2000000               795 ns/op
// BenchmarkShiftMask100-4           200000              9573 ns/op
// BenchmarkShiftMask1000-4           10000            115398 ns/op
// BenchmarkShiftMask10000-4           1000           1245754 ns/op

func BenchmarkClearRightmost1(b *testing.B) {
	benchN(b, 1, PopCountClearRightmost)
}

func BenchmarkClearRightmost10(b *testing.B) {
	benchN(b, 10, PopCountClearRightmost)
}

func BenchmarkClearRightmost100(b *testing.B) {
	benchN(b, 100, PopCountClearRightmost)
}

func BenchmarkClearRightmost1000(b *testing.B) {
	benchN(b, 1000, PopCountClearRightmost)
}

func BenchmarkClearRightmost10000(b *testing.B) {
	benchN(b, 10000, PopCountClearRightmost)
}

func BenchmarkTable1(b *testing.B) {
	benchTableN(b, 1)
}

func BenchmarkTable10(b *testing.B) {
	benchTableN(b, 10)
}

func BenchmarkTable100(b *testing.B) {
	benchTableN(b, 100)
}

func BenchmarkTable1000(b *testing.B) {
	benchTableN(b, 1000)
}

func BenchmarkTable10000(b *testing.B) {
	benchTableN(b, 10000)
}

func BenchmarkShiftValue1(b *testing.B) {
	benchN(b, 1, PopCountShiftValue)
}

func BenchmarkShiftValue10(b *testing.B) {
	benchN(b, 10, PopCountShiftValue)
}

func BenchmarkShiftValue100(b *testing.B) {
	benchN(b, 100, PopCountShiftValue)
}

func BenchmarkShiftValue1000(b *testing.B) {
	benchN(b, 1000, PopCountShiftValue)
}

func BenchmarkShiftValue10000(b *testing.B) {
	benchN(b, 10000, PopCountShiftValue)
}

func BenchmarkShiftMask1(b *testing.B) {
	benchN(b, 1, PopCountShiftMask)
}

func BenchmarkShiftMask10(b *testing.B) {
	benchN(b, 10, PopCountShiftMask)
}

func BenchmarkShiftMask100(b *testing.B) {
	benchN(b, 100, PopCountShiftMask)
}

func BenchmarkShiftMask1000(b *testing.B) {
	benchN(b, 1000, PopCountShiftMask)
}

func BenchmarkShiftMask10000(b *testing.B) {
	benchN(b, 10000, PopCountShiftMask)
}
```

**练习 11.7:** 为`*IntSet`（§6.5）的`Add`、`UnionWith`和其他方法编写基准测试，使用大量随机输入。你可以让这些方法跑多快？选择字的大小对于性能的影响如何？`IntSet`和基于内建map的实现相比有多快？

```go

```


## 11.5. 剖析

基准测试（Benchmark）对于衡量特定操作的性能是有帮助的，但是当我们试图让程序跑的更快的时候，我们通常并不知道从哪里开始优化。每个码农都应该知道Donald Knuth在1974年的“Structured Programming with go to Statements”上所说的格言。虽然经常被解读为不重视性能的意思，但是从原文我们可以看到不同的含义：

> 毫无疑问，对效率的片面追求会导致各种滥用。程序员会浪费大量的时间在非关键程序的速度上，实际上这些尝试提升效率的行为反倒可能产生很大的负面影响，特别是当调试和维护的时候。我们不应该过度纠结于细节的优化，应该说约97%的场景：过早的优化是万恶之源。
>
> 当然我们也不应该放弃对那关键3%的优化。一个好的程序员不会因为这个比例小就裹足不前，他们会明智地观察和识别哪些是关键的代码；但是仅当关键代码已经被确认的前提下才会进行优化。对于很多程序员来说，判断哪部分是关键的性能瓶颈，是很容易犯经验上的错误的，因此一般应该借助测量工具来证明。

当我们想仔细观察我们程序的运行速度的时候，最好的方法是性能剖析。剖析技术是基于程序执行期间一些自动抽样，然后在收尾时进行推断；最后产生的统计结果就称为剖析数据。

Go语言支持多种类型的剖析性能分析，每一种关注不同的方面，但它们都涉及到每个采样记录的感兴趣的一系列事件消息，每个事件都包含函数调用时函数调用堆栈的信息。内建的`go test`工具对几种分析方式都提供了支持。

CPU剖析数据标识了最耗CPU时间的函数。在每个CPU上运行的线程在每隔几毫秒都会遇到操作系统的中断事件，每次中断时都会记录一个剖析数据然后恢复正常的运行。

堆剖析则标识了最耗内存的语句。剖析库会记录调用内部内存分配的操作，平均每512KB的内存申请会触发一个剖析数据。

阻塞剖析则记录阻塞`goroutine`最久的操作，例如系统调用、管道发送和接收，还有获取锁等。每当`goroutine`被这些操作阻塞时，剖析库都会记录相应的事件。

只需要开启下面其中一个标志参数就可以生成各种分析文件。当同时使用多个标志参数时需要当心，因为一项分析操作可能会影响其他项的分析结果。

```go
$ go test -cpuprofile=cpu.out
$ go test -blockprofile=block.out
$ go test -memprofile=mem.out
```

对于一些非测试程序也很容易进行剖析，具体的实现方式，与程序是短时间运行的小工具还是长时间运行的服务会有很大不同。剖析对于长期运行的程序尤其有用，因此可以通过调用Go的runtime API来启用运行时剖析。

一旦我们已经收集到了用于分析的采样数据，我们就可以使用`pprof`来分析这些数据。这是Go工具箱自带的一个工具，但并不是一个日常工具，它对应`go tool pprof`命令。该命令有许多特性和选项，但是最基本的是两个参数：生成这个概要文件的可执行程序和对应的剖析数据。

为了提高分析效率和减少空间，分析日志本身并不包含函数的名字；它只包含函数对应的地址。也就是说`pprof`需要对应的可执行程序来解读剖析数据。虽然`go test`通常在测试完成后就丢弃临时用的测试程序，但是在启用分析的时候会将测试程序保存为`foo.test`文件，其中`foo`部分对应待测包的名字。

下面的命令演示了如何收集并展示一个CPU分析文件。我们选择`net/http`包的一个基准测试为例。通常最好是对业务关键代码的部分设计专门的基准测试。因为简单的基准测试几乎没法代表业务场景，因此我们用`-run=NONE`参数禁止那些简单测试。

```go
$ go test -run=NONE -bench=ClientServerParallelTLS64 \
    -cpuprofile=cpu.log net/http
 PASS
 BenchmarkClientServerParallelTLS64-8  1000
    3141325 ns/op  143010 B/op  1747 allocs/op
ok       net/http       3.395s

$ go tool pprof -text -nodecount=10 ./http.test cpu.log
2570ms of 3590ms total (71.59%)
Dropped 129 nodes (cum <= 17.95ms)
Showing top 10 nodes out of 166 (cum >= 60ms)
    flat  flat%   sum%     cum   cum%
  1730ms 48.19% 48.19%  1750ms 48.75%  crypto/elliptic.p256ReduceDegree
   230ms  6.41% 54.60%   250ms  6.96%  crypto/elliptic.p256Diff
   120ms  3.34% 57.94%   120ms  3.34%  math/big.addMulVVW
   110ms  3.06% 61.00%   110ms  3.06%  syscall.Syscall
    90ms  2.51% 63.51%  1130ms 31.48%  crypto/elliptic.p256Square
    70ms  1.95% 65.46%   120ms  3.34%  runtime.scanobject
    60ms  1.67% 67.13%   830ms 23.12%  crypto/elliptic.p256Mul
    60ms  1.67% 68.80%   190ms  5.29%  math/big.nat.montgomery
    50ms  1.39% 70.19%    50ms  1.39%  crypto/elliptic.p256ReduceCarry
    50ms  1.39% 71.59%    60ms  1.67%  crypto/elliptic.p256Sum
```

参数`-text`用于指定输出格式，在这里每行是一个函数，根据使用CPU的时间长短来排序。其中`-nodecount=10`参数限制了只输出前10行的结果。对于严重的性能问题，这个文本格式基本可以帮助查明原因了。

这个概要文件告诉我们，HTTPS基准测试中`crypto/elliptic.p256ReduceDegree`函数占用了将近一半的CPU资源，对性能占很大比重。相比之下，如果一个概要文件中主要是`runtime`包的内存分配的函数，那么减少内存消耗可能是一个值得尝试的优化策略。

对于一些更微妙的问题，你可能需要使用`pprof`的图形显示功能。这个需要安装`GraphViz`工具，可以从 http://www.graphviz.org 下载。参数`-web`用于生成函数的有向图，标注有CPU的使用和最热点的函数等信息。

这一节我们只是简单看了下Go语言的数据分析工具。如果想了解更多，可以阅读Go官方博客的“Profiling Go Programs”一文。
## 11.6. 示例函数

第三种被`go test`特别对待的函数是示例函数，以`Example`为函数名开头。示例函数没有函数参数和返回值。下面是`IsPalindrome`函数对应的示例函数：

```Go
func ExampleIsPalindrome() {
	fmt.Println(IsPalindrome("A man, a plan, a canal: Panama"))
	fmt.Println(IsPalindrome("palindrome"))
	// Output:
	// true
	// false
}
```

示例函数有三个用处。最主要的一个是作为文档：一个包的例子可以更简洁直观的方式来演示函数的用法，比文字描述更直接易懂，特别是作为一个提醒或快速参考时。一个示例函数也可以方便展示属于同一个接口的几种类型或函数之间的关系，所有的文档都必须关联到一个地方，就像一个类型或函数声明都统一到包一样。同时，示例函数和注释并不一样，示例函数是真实的Go代码，需要接受编译器的编译时检查，这样可以保证源代码更新时，示例代码不会脱节。

根据示例函数的后缀名部分，`godoc`这个web文档服务器会将示例函数关联到某个具体函数或包本身，因此`ExampleIsPalindrome`示例函数将是`IsPalindrome`函数文档的一部分，`Example`示例函数将是包文档的一部分。

示例函数的第二个用处是，在`go test`执行测试的时候也会运行示例函数测试。如果示例函数内含有类似上面例子中的`// Output:`格式的注释，那么测试工具会执行这个示例函数，然后检查示例函数的标准输出与注释是否匹配。

示例函数的第三个目的提供一个真实的演练场。 http://golang.org 就是由`godoc`提供的文档服务，它使用了`Go Playground`让用户可以在浏览器中在线编辑和运行每个示例函数，就像图11.4所示的那样。这通常是学习函数使用或Go语言特性最快捷的方式。

![示例函数 - 图1](images\ch11-04-1579484872583.png)

本书最后的两章是讨论`reflect`和`unsafe`包，一般的Go程序员很少使用它们，事实上也很少需要用到。因此，如果你还没有写过任何真实的Go程序的话，现在可以先去写些代码了。

