# 第一章　入门

本章介绍Go语言的基础组件。本章提供了足够的信息和示例程序，希望可以帮你尽快入门，写出有用的程序。本章和之后章节的示例程序都针对你可能遇到的现实案例。先了解几个Go程序，涉及的主题从简单的文件处理、图像处理到互联网客户端和服务端并发。当然，第一章不会解释细枝末节，但用这些程序来学习一门新语言还是很有效的。

学习一门新语言时，会有一种自然的倾向，按照自己熟悉的语言的套路写新语言程序。学习Go语言的过程中，请警惕这种想法，尽量别这么做。我们会演示怎么写好Go语言程序，所以，请使用本书的代码作为你自己写程序时的指南。
## 1.1. Hello, World

我们以现已成为传统的“hello world”案例来开始吧，这个例子首次出现于1978年出版的C语言圣经[《The C Programming Language》](http://s3-us-west-2.amazonaws.com/belllabs-microsite-dritchie/cbook/index.html)（译注：本书作者之一Brian W. Kernighan也是《The C Programming Language》一书的作者）。C语言是直接影响Go语言设计的语言之一。这个例子体现了Go语言一些核心理念。

*gopl.io/ch1/helloworld*

```go
package main

import "fmt"

func main() {
	fmt.Println("Hello, 世界")
}
```

Go是一门编译型语言，Go语言的工具链将源代码及其依赖转换成计算机的机器指令（译注：静态编译）。Go语言提供的工具都通过一个单独的命令`go`调用，`go`命令有一系列子命令。最简单的一个子命令就是`run`。这个命令编译一个或多个以`.go`结尾的源文件，链接库文件，并运行最终生成的可执行文件。（本书使用`$`表示命令行提示符。）

```go
$ go run helloworld.go
```

毫无意外，这个命令会输出：

```go
Hello, 世界
```

Go语言原生支持`Unicode`，它可以处理全世界任何语言的文本。

如果不只是一次性实验，你肯定希望能够编译这个程序，保存编译结果以备将来之用。可以用`build`子命令：

```go
$ go build helloworld.go
```

这个命令生成一个名为`helloworld`的可执行的二进制文件（译注：Windows系统下生成的可执行文件是`helloworld.exe`，增加了`.exe`后缀名），之后你可以随时运行它（译注：在Windows系统下在命令行直接输入`helloworld.exe`命令运行），不需任何处理（译注：因为静态编译，所以不用担心在系统库更新的时候冲突，幸福感满满）。

```go
$ ./helloworld
Hello, 世界
```

本书中所有示例代码上都有一行标记，利用这些标记可以从[gopl.io](http://gopl.io)网站上本书源码仓库里获取代码：

```go
gopl.io/ch1/helloworld
```

执行 `go get gopl.io/ch1/helloworld` 命令，就会从网上获取代码，并放到对应目录中（需要先安装**Git**或**Hg**之类的版本管理工具，并将对应的命令添加到`PATH`环境变量中。序言已经提及，需要先设置好`GOPATH`环境变量，下载的代码会放在`$GOPATH/src/gopl.io/ch1/helloworld`目录）。2.6和10.7节有这方面更详细的介绍。

来讨论下程序本身。Go语言的代码通过**包**（`package`）组织，包类似于其它语言里的库（`libraries`）或者模块（`modules`）。一个包由位于单个目录下的一个或多个`.go`源代码文件组成，目录定义包的作用。每个源文件都以一条`package`声明语句开始，这个例子里就是`package main`，表示该文件属于哪个包，紧跟着一系列导入（`import`）的包，之后是存储在这个文件里的程序语句。

Go的标准库提供了100多个包，以支持常见功能，如输入、输出、排序以及文本处理。比如`fmt`包，就含有格式化输出、接收输入的函数。`Println`是其中一个基础函数，可以打印以空格间隔的一个或多个值，并在最后添加一个换行符，从而输出一整行。

`main`包比较特殊。它定义了一个独立可执行的程序，而不是一个库。在`main`里的`main` *函数* 也很特殊，它是整个程序执行时的入口（译注：C系语言差不多都这样）。`main`函数所做的事情就是程序做的。当然了，`main`函数一般调用其它包里的函数完成很多工作（如：`fmt.Println`）。

必须告诉编译器源文件需要哪些包，这就是跟随在`package`声明后面的`import`声明扮演的角色。`hello world`例子只用到了一个包，大多数程序需要导入多个包。

必须恰当导入需要的包，缺少了必要的包或者导入了不需要的包，程序都无法编译通过。这项严格要求避免了程序开发过程中引入未使用的包（译注：Go语言编译过程没有警告信息，争议特性之一）。

`import`声明必须跟在文件的`package`声明之后。随后，则是组成程序的函数、变量、常量、类型的声明语句（分别由关键字`func`、`var`、`const`、`type`定义）。这些内容的声明顺序并不重要（译注：最好还是定一下规范）。这个例子的程序已经尽可能短了，只声明了一个函数，其中只调用了一个其他函数。为了节省篇幅，有些时候示例程序会省略`package`和`import`声明，但是，这些声明在源代码里有，并且必须得有才能编译。

一个函数的声明由`func`关键字、函数名、参数列表、返回值列表（这个例子里的`main`函数参数列表和返回值都是空的）以及包含在大括号里的函数体组成。第五章进一步考察函数。

Go语言不需要在语句或者声明的末尾添加分号，除非一行上有多条语句。实际上，编译器会主动把特定符号后的换行符转换为分号，因此换行符添加的位置会影响Go代码的正确解析（译注：比如行末是标识符、整数、浮点数、虚数、字符或字符串文字、关键字`break`、`continue`、`fallthrough`或`return`中的一个、运算符和分隔符`++`、`--`、`)`、`]`或`}`中的一个）。举个例子，函数的左括号`{`必须和`func`函数声明在同一行上，且位于末尾，不能独占一行，而在表达式`x + y`中，可在`+`后换行，不能在`+`前换行（译注：以`+`结尾的话不会被插入分号分隔符，但是以x结尾的话则会被分号分隔符，从而导致编译错误）。

Go语言在代码格式上采取了很强硬的态度。`gofmt`工具把代码格式化为标准格式（译注：这个格式化工具没有任何可以调整代码格式的参数，Go语言就是这么任性），并且`go`工具中的`fmt`子命令会对指定包，否则默认为当前目录中所有`.go`源文件应用`gofmt`命令。本书中的所有代码都被`gofmt`过。你也应该养成格式化自己的代码的习惯。以法令方式规定标准的代码格式可以避免无尽的无意义的琐碎争执（译注：也导致了Go语言的TIOBE排名较低，因为缺少撕逼的话题）。更重要的是，这样可以做多种自动源码转换，如果放任Go语言代码格式，这些转换就不大可能了。

很多文本编辑器都可以配置为保存文件时自动执行`gofmt`，这样你的源代码总会被恰当地格式化。还有个相关的工具，`goimports`，可以根据代码需要，自动地添加或删除`import`声明。这个工具并没有包含在标准的分发包中，可以用下面的命令安装：

```go
$ go get golang.org/x/tools/cmd/goimports
```

对于大多数用户来说，下载、编译包、运行测试用例、察看Go语言的文档等等常用功能都可以用`go`的工具完成。10.7节详细介绍这些知识。
## 1.2. 命令行参数

大多数的程序都是处理输入，产生输出；这也正是“计算”的定义。但是，程序如何获取要处理的输入数据呢？一些程序生成自己的数据，但通常情况下，输入来自于程序外部：文件、网络连接、其它程序的输出、敲键盘的用户、命令行参数或其它类似输入源。下面几个例子会讨论其中几个输入源，首先是命令行参数。

`os`包以跨平台的方式，提供了一些与操作系统交互的函数和变量。程序的命令行参数可从`os`包的`Args`变量获取；`os`包外部使用`os.Args`访问该变量。

`os.Args`变量是一个字符串（string）的*切片*（slice）（译注：slice和Python语言中的切片类似，是一个简版的动态数组），切片是Go语言的基础概念，稍后详细介绍。现在先把切片s当作数组元素序列，序列的长度动态变化，用`s[i]`访问单个元素，用`s[m:n]`获取子序列（译注：和python里的语法差不多）。序列的元素数目为`len(s)`。和大多数编程语言类似，区间索引时，Go言里也采用**左闭右开**形式，即，区间包括第一个索引元素，不包括最后一个，因为这样可以简化逻辑。（译注：比如`a = [1, 2, 3, 4, 5]`, `a[0:3] = [1, 2, 3]`，不包含最后一个元素）。比如`s[m:n]`这个切片，`0 ≤ m ≤ n ≤ len(s)`，包含`n-m`个元素。

`os.Args`的第一个元素：`os.Args[0]`，是命令本身的名字；其它的元素则是程序启动时传给它的参数。`s[m:n]`形式的切片表达式，产生从第`m`个元素到第`n-1`个元素的切片，下个例子用到的元素包含在`os.Args[1:len(os.Args)]`切片中。如果省略切片表达式的`m`或`n`，会默认传入`0`或`len(s)`，因此前面的切片可以简写成`os.Args[1:]`。

下面是Unix里`echo`命令的一份实现，`echo`把它的命令行参数打印成一行。程序导入了两个包，用括号把它们括起来写成列表形式，而没有分开写成独立的`import`声明。两种形式都合法，列表形式习惯上用得多。包导入顺序并不重要；`gofmt`工具格式化时按照字母顺序对包名排序。（示例有多个版本时，我们会对示例编号，这样可以明确当前正在讨论的是哪个。）

*gopl.io/ch1/echo1*

```go
// Echo1 prints its command-line arguments.
package main

import (
	"fmt"
	"os"
)

func main() {
	var s, sep string
	for i := 1; i < len(os.Args); i++ {
		s += sep + os.Args[i]
		sep = " "
	}
	fmt.Println(s)
}
```

注释语句以`//`开头。对于程序员来说，`//`之后到行末之间所有的内容都是注释，被编译器忽略。按照惯例，我们在每个包的包声明前添加注释；对于`main package`，注释包含一句或几句话，从整体角度对程序做个描述。

`var`声明定义了两个string类型的变量`s`和`sep`。变量会在声明时直接初始化。如果变量没有显式初始化，则被隐式地赋予其类型的*零值*（zero value），数值类型是`0`，字符串类型是空字符串`""`。这个例子里，声明把`s`和`sep`隐式地初始化成空字符串。第2章再来详细地讲解变量和声明。

对数值类型，Go语言提供了常规的数值和逻辑运算符。而对`string`类型，`+`运算符连接字符串（译注：和C++或者JS是一样的）。所以表达式：

```go
sep + os.Args[i]
```

表示连接字符串`sep`和`os.Args`。程序中使用的语句：

```go
s += sep + os.Args[i]
```

是一条*赋值语句*，将`s`的旧值跟`sep`与`os.Args[i]`连接后赋值回`s`，等价于：

```go
s = s + sep + os.Args[i]
```

运算符`+=`是赋值运算符（assignment operator），每种数值运算符或逻辑运算符，如`+`或`*`，都有对应的赋值运算符。

`echo`程序可以每循环一次输出一个参数，这个版本却是不断地把新文本追加到末尾来构造字符串。字符串`s`开始为空，即值为`""`，每次循环会添加一些文本；第一次迭代之后，还会再插入一个空格，因此循环结束时每个参数中间都有一个空格。这是一种二次加工（quadratic process），当参数数量庞大时，开销很大，但是对于`echo`，这种情形不大可能出现。本章会介绍`echo`的若干改进版，下一章解决低效问题。

循环索引变量`i`在for循环的第一部分中定义。符号`:=`是*短变量声明*（short variable declaration）的一部分，这是定义一个或多个变量并根据它们的初始值为这些变量赋予适当类型的语句。下一章有这方面更多说明。

自增语句`i++`给`i`加1；这和`i += 1`以及`i = i + 1`都是等价的。对应的还有`i--`给`i`减1。它们是语句，而不像C系的其它语言那样是表达式。所以`j = i++`非法，而且`++`和`--`都只能放在变量名后面，因此`--i`也非法。

Go语言只有`for`循环这一种循环语句。`for`循环有多种形式，其中一种如下所示：

```go
for initialization; condition; post {
	// zero or more statements
}
```

for循环三个部分不需括号包围。大括号强制要求，左大括号必须和*post*语句在同一行。

*`initialization`*语句是可选的，在循环开始前执行。`initalization`如果存在，必须是一条*简单语句*（`simple statement`），即，短变量声明、自增语句、赋值语句或函数调用。`condition`是一个布尔表达式（`boolean expression`），其值在每次循环迭代开始时计算。如果为`true`则执行循环体语句。`post`语句在循环体执行结束后执行，之后再次对`condition`求值。`condition`值为`false`时，循环结束。

`for`循环的这三个部分每个都可以省略，如果省略`initialization`和`post`，分号也可以省略：

```go
// a traditional "while" loop
for condition {
	// ...
}
```

如果连`condition`也省略了，像下面这样：

```go
// a traditional infinite loop
for {
	// ...
}
```

这就变成一个无限循环，尽管如此，还可以用其他方式终止循环，如一条`break`或`return`语句。

`for`循环的另一种形式，在某种数据类型的区间（`range`）上遍历，如字符串或切片。`echo`的第二版本展示了这种形式：

*gopl.io/ch1/echo2*

```go
// Echo2 prints its command-line arguments.
package main

import (
	"fmt"
    "os"
)

func main() {
	s, sep := "", ""
	for _, arg := range os.Args[1:] {
		s += sep + arg
		sep = " "
	}
	fmt.Println(s)
}
```

每次循环迭代，`range`产生一对值；索引以及在该索引处的元素值。这个例子不需要索引，但`range`的语法要求，要处理元素，必须处理索引。一种思路是把索引赋值给一个临时变量（如`temp`）然后忽略它的值，但Go语言**不允许**使用无用的局部变量（local variables），因为这会导致编译错误。

Go语言中这种情况的解决方法是用`空标识符`（blank identifier），即`_`（也就是下划线）。空标识符可用于在任何语法需要变量名但程序逻辑不需要的时候（如：在循环里）丢弃不需要的循环索引，并保留元素值。大多数的Go程序员都会像上面这样使用`range`和`_`写`echo`程序，因为隐式地而非显式地索引`os.Args`，容易写对。

`echo`的这个版本使用一条短变量声明来声明并初始化`s`和`seps`，也可以将这两个变量分开声明，声明一个变量有好几种方式，下面这些都等价：

```go
s := ""
var s string
var s = ""
var s string = ""
```

用哪种不用哪种，为什么呢？第一种形式，是一条短变量声明，最简洁，但只能用在函数内部，而不能用于包变量。第二种形式依赖于字符串的默认初始化零值机制，被初始化为`""`。第三种形式用得很少，除非同时声明多个变量。第四种形式显式地标明变量的类型，当变量类型与初值类型相同时，类型冗余，但如果两者类型不同，变量类型就必须了。实践中一般使用前两种形式中的某个，初始值重要的话就显式地指定变量的类型，否则使用隐式初始化。

如前文所述，每次循环迭代字符串s的内容都会更新。`+=`连接原字符串、空格和下个参数，产生新字符串，并把它赋值给`s`。`s`原来的内容已经不再使用，将在适当时机对它进行垃圾回收。

如果连接涉及的数据量很大，这种方式代价高昂。一种简单且高效的解决方案是使用`strings`包的`Join`函数：

*gopl.io/ch1/echo3*

```go
func main() {
	fmt.Println(strings.Join(os.Args[1:], " "))
}
```

最后，如果不关心输出格式，只想看看输出值，或许只是为了调试，可以用`Println`为我们格式化输出。

```go
fmt.Println(os.Args[1:])
```

这条语句的输出结果跟`strings.Join`得到的结果很像，只是被放到了一对方括号里。切片都会被打印成这种格式。

**练习 1.1：** 修改`echo`程序，使其能够打印`os.Args[0]`，即被执行命令本身的名字。

```go
// main.go
package main

import (
	"fmt"
	"os"
)

func main() {
	fmt.Println(os.Args)
}
```

**练习 1.2：** 修改`echo`程序，使其打印每个参数的索引和值，每个一行。

```go
// main.go
package main

import (
	"fmt"
	"os"
)

func main() {
	for i, arg := range os.Args {
		fmt.Printf("%d: %s\n", i, arg)
	}
}
```

**练习 1.3：** 做实验测量潜在低效的版本和使用了`strings.Join`的版本的运行时间差异。（1.6节讲解了部分`time`包，11.4节展示了如何写标准测试程序，以得到系统性的性能评测。）

```go
// concat_test.go
// Run with go test -bench=.
package concat_test

import (
	"strings"
	"testing"
)

var args = []string{"hi", "there", "buddy", "boy", "5", "6", "7", "8", "9"}

func concat(args []string) {
	r, sep := "", ""
	for _, a := range args {
		r += sep + a
		sep = " "
	}
}

func BenchmarkConcat(b *testing.B) {
	for i := 0; i < b.N; i++ {
		concat(args)
	}
}

func BenchmarkJoin(b *testing.B) {
	for i := 0; i < b.N; i++ {
		strings.Join(args, " ")
	}
}
```

## 1.3. 查找重复的行

对文件做拷贝、打印、搜索、排序、统计或类似事情的程序都有一个差不多的程序结构：一个处理输入的循环，在每个元素上执行计算处理，在处理的同时或最后产生输出。我们会展示一个名为`dup`的程序的三个版本；灵感来自于Unix的`uniq`命令，其寻找相邻的重复行。该程序使用的结构和包是个参考范例，可以方便地修改。

`dup`的第一个版本打印标准输入中多次出现的行，以重复次数开头。该程序将引入`if`语句，`map`数据类型以及`bufio`包。

*gopl.io/ch1/dup1*

```go
// Dup1 prints the text of each line that appears more than
// once in the standard input, preceded by its count.
package main

import (
	"bufio"
	"fmt"
	"os"
)

func main() {
	counts := make(map[string]int)
	input := bufio.NewScanner(os.Stdin)
	for input.Scan() {
		counts[input.Text()]++
	}
	// NOTE: ignoring potential errors from input.Err()
	for line, n := range counts {
		if n > 1 {
			fmt.Printf("%d\t%s\n", n, line)
		}
	}
}
```

正如`for`循环一样，`if`语句条件两边也不加括号，但是主体部分需要加。`if`语句的`else`部分是可选的，在`if`的条件为`false`时执行。

**map**存储了键/值（key/value）的集合，对集合元素，提供常数时间的存、取或测试操作。键可以是任意类型，只要其值能用`==`运算符比较，最常见的例子是字符串；值则可以是任意类型。这个例子中的键是字符串，值是整数。内置函数`make`创建空`map`，此外，它还有别的作用。4.3节讨论`map`。

> 译注：从功能和实现上说，`Go`的`map`类似于`Java`语言中的`HashMap`，Python语言中的`dict`，`Lua`语言中的`table`，通常使用`hash`实现。遗憾的是，对于该词的翻译并不统一，数学界术语为`映射`，而计算机界众说纷纭莫衷一是。为了防止对读者造成误解，保留不译。

每次`dup`读取一行输入，该行被当做键存入`map`，其对应的值递增。`counts[input.Text()]++`语句等价下面两句：

```go
line := input.Text()
counts[line] = counts[line] + 1
```

`map`中不含某个键时不用担心，首次读到新行时，等号右边的表达式`counts[line]`的值将被计算为其类型的零值，对于`int`即`0`。

为了打印结果，我们使用了基于`range`的循环，并在`counts`这个`map`上迭代。跟之前类似，每次迭代得到两个结果，键和其在`map`中对应的值。`map`的迭代顺序并不确定，从实践来看，该顺序随机，每次运行都会变化。这种设计是有意为之的，因为能防止程序依赖特定遍历顺序，而这是无法保证的。

> （译注：具体可以参见这里http://stackoverflow.com/questions/11853396/google-go-lang-assignment-order）

继续来看`bufio`包，它使处理输入和输出方便又高效。`Scanner`类型是该包最有用的特性之一，它读取输入并将其拆成行或单词；通常是处理行形式的输入最简单的方法。

程序使用短变量声明创建`bufio.Scanner`类型的变量`input`。

```go
input := bufio.NewScanner(os.Stdin)
```

该变量从程序的标准输入中读取内容。每次调用`input.Scan()`，即读入下一行，并移除行末的换行符；读取的内容可以调用`input.Text()`得到。`Scan`函数在读到一行时返回`true`，不再有输入时返回`false`。

类似于C或其它语言里的`printf`函数，`fmt.Printf`函数对一些表达式产生格式化输出。该函数的首个参数是个格式字符串，指定后续参数被如何格式化。各个参数的格式取决于“转换字符”（conversion character），形式为百分号后跟一个字母。举个例子，`%d`表示以十进制形式打印一个整型操作数，而`%s`则表示把字符串型操作数的值展开。

`Printf`有一大堆这种转换，Go程序员称之为*动词（verb）*。下面的表格虽然远不是完整的规范，但展示了可用的很多特性：

```go
%d          十进制整数
%x, %o, %b  十六进制，八进制，二进制整数。
%f, %g, %e  浮点数： 3.141593 3.141592653589793 3.141593e+00
%t          布尔：true或false
%c          字符（rune） (Unicode码点)
%s          字符串
%q          带双引号的字符串"abc"或带单引号的字符'c'
%v          变量的自然形式（natural format）
%T          变量的类型
%%          字面上的百分号标志（无操作数）
```

`dup1`的格式字符串中还含有制表符`\t`和换行符`\n`。字符串字面上可能含有这些代表不可见字符的**转义字符（escape sequences）**。默认情况下，`Printf`不会换行。按照惯例，以字母`f`结尾的格式化函数，如`log.Printf`和`fmt.Errorf`，都采用`fmt.Printf`的格式化准则。而以`ln`结尾的格式化函数，则遵循`Println`的方式，以跟`%v`差不多的方式格式化参数，并在最后添加一个换行符。（译注：后缀`f`指`format`，`ln`指`line`。）

很多程序要么从标准输入中读取数据，如上面的例子所示，要么从一系列具名文件中读取数据。`dup`程序的下个版本读取标准输入或是使用`os.Open`打开各个具名文件，并操作它们。

*gopl.io/ch1/dup2*

```go
// Dup2 prints the count and text of lines that appear more than once
// in the input.  It reads from stdin or from a list of named files.
package main

import (
	"bufio"
	"fmt"
	"os"
)

func main() {
	counts := make(map[string]int)
	files := os.Args[1:]
	if len(files) == 0 {
		countLines(os.Stdin, counts)
	} else {
		for _, arg := range files {
			f, err := os.Open(arg)
			if err != nil {
				fmt.Fprintf(os.Stderr, "dup2: %v\n", err)
				continue
			}
			countLines(f, counts)
			f.Close()
		}
	}
	for line, n := range counts {
		if n > 1 {
			fmt.Printf("%d\t%s\n", n, line)
		}
	}
}

func countLines(f *os.File, counts map[string]int) {
	input := bufio.NewScanner(f)
	for input.Scan() {
		counts[input.Text()]++
	}
	// NOTE: ignoring potential errors from input.Err()
}
```

`os.Open`函数返回两个值。第一个值是被打开的文件(`*os.File`），其后被`Scanner`读取。

`os.Open`返回的第二个值是内置`error`类型的值。如果`err`等于内置值`nil`（译注：相当于其它语言里的`NULL`），那么文件被成功打开。读取文件，直到文件结束，然后调用`Close`关闭该文件，并释放占用的所有资源。相反的话，如果`err`的值不是`nil`，说明打开文件时出错了。这种情况下，错误值描述了所遇到的问题。我们的错误处理非常简单，只是使用`Fprintf`与表示任意类型默认格式值的动词`%v`，向标准错误流打印一条信息，然后`dup`继续处理下一个文件；`continue`语句直接跳到`for`循环的下个迭代开始执行。

为了使示例代码保持合理的大小，本书开始的一些示例有意简化了错误处理，显而易见的是，应该检查`os.Open`返回的错误值，然而，使用`input.Scan`读取文件过程中，不大可能出现错误，因此我们忽略了错误处理。我们会在跳过错误检查的地方做说明。5.4节中深入介绍错误处理。

注意`countLines`函数在其声明前被调用。函数和包级别的变量（package-level entities）可以任意顺序声明，并不影响其被调用。（译注：最好还是遵循一定的规范）

`map`是一个由`make`函数创建的数据结构的引用。`map`作为参数传递给某函数时，该函数接收这个引用的一份拷贝（`copy`，或译为副本），被调用函数对`map`底层数据结构的任何修改，调用者函数都可以通过持有的`map`引用看到。在我们的例子中，`countLines`函数向`counts`插入的值，也会被`main`函数看到。（译注：类似于C++里的引用传递，实际上指针是另一个指针了，但内部存的值指向同一块内存）

`dup`的前两个版本以"流”模式读取输入，并根据需要拆分成多个行。理论上，这些程序可以处理任意数量的输入数据。还有另一个方法，就是一口气把全部输入数据读到内存中，一次分割为多行，然后处理它们。下面这个版本，`dup3`，就是这么操作的。这个例子引入了`ReadFile`函数（来自于`io/ioutil`包），其读取指定文件的全部内容，`strings.Split`函数把字符串分割成子串的切片。（`Split`的作用与前文提到的`strings.Join`相反。）

我们略微简化了`dup3`。首先，由于`ReadFile`函数需要文件名作为参数，因此只读指定文件，不读标准输入。其次，由于行计数代码只在一处用到，故将其移回`main`函数。

*gopl.io/ch1/dup3*

```go
package main

import (
	"fmt"
	"io/ioutil"
	"os"
	"strings"
)

func main() {
	counts := make(map[string]int)
	for _, filename := range os.Args[1:] {
		data, err := ioutil.ReadFile(filename)
		if err != nil {
			fmt.Fprintf(os.Stderr, "dup3: %v\n", err)
			continue
		}
		for _, line := range strings.Split(string(data), "\n") {
			counts[line]++
		}
	}
	for line, n := range counts {
		if n > 1 {
			fmt.Printf("%d\t%s\n", n, line)
		}
	}
}
```

`ReadFile`函数返回一个字节切片（byte slice），必须把它转换为`string`，才能用`strings.Split`分割。我们会在3.5.4节详细讲解字符串和字节切片。

实现上，`bufio.Scanner`、`ioutil.ReadFile`和`ioutil.WriteFile`都使用`*os.File`的`Read`和`Write`方法，但是，大多数程序员很少需要直接调用那些低级（lower-level）函数。高级（higher-level）函数，像`bufio`和`io/ioutil`包中所提供的那些，用起来要容易点。

**练习 1.4：** 修改`dup2`，出现重复的行时打印文件名称。

```go
package main

import (
	"bufio"
	"fmt"
	"os"
)

func main() {
	counts := make(map[string]int)
	foundIn := make(map[string][]string)
	files := os.Args[1:]
	if len(files) == 0 {
		countLines(os.Stdin, counts, foundIn)
	} else {
		for _, arg := range files {
			f, err := os.Open(arg)
			if err != nil {
				fmt.Fprintf(os.Stderr, "dup2: %v\n", err)
				continue
			}
			countLines(f, counts, foundIn)
			f.Close()
		}
	}
	for line, n := range counts {
		if n > 1 {
			fmt.Printf("%d\t%v\t%s\n", n, foundIn[line], line)
		}
	}
}

func in(needle string, strings []string) bool {
	for _, s := range strings {
		if needle == s {
			return true
		}
	}
	return false
}

func countLines(f *os.File, counts map[string]int, foundIn map[string][]string) {
	input := bufio.NewScanner(f)
	for input.Scan() {
		line := input.Text()
		counts[line]++
		if !in(f.Name(), foundIn[line]) {
			foundIn[line] = append(foundIn[line], f.Name())
		}
	}
	// NOTE: ignoring potential errors from input.Err()
}
```

## 1.4. GIF动画

下面的程序会演示Go语言标准库里的`image`这个`package`的用法，我们会用这个包来生成一系列的`bit-mapped`图，然后将这些图片编码为一个GIF动画。我们生成的图形名字叫利萨如图形（`Lissajous figures`），这种效果是在1960年代的老电影里出现的一种视觉特效。它们是协振子在两个纬度上振动所产生的曲线，比如两个`sin`正弦波分别在`x`轴和`y`轴输入会产生的曲线。图1.1是这样的一个例子：

![GIF动画 - 图1](images\ch1-01.png)

译注：要看这个程序的结果，需要将标准输出重定向到一个GIF图像文件（使用 `./lissajous > output.gif` 命令）。下面是GIF图像动画效果：

![GIF动画 - 图2](images\ch1-01.gif)

这段代码里我们用了一些新的结构，包括`const`声明，`struct`结构体类型，复合声明。和我们举的其它的例子不太一样，这一个例子包含了浮点数运算。这些概念我们只在这里简单地说明一下，之后的章节会更详细地讲解。

*gopl.io/ch1/lissajous*

```go
// Lissajous generates GIF animations of random Lissajous figures.
package main

import (
	"image"
	"image/color"
	"image/gif"
	"io"
	"math"
	"math/rand"
	"os"
	"time"
)

var palette = []color.Color{color.White, color.Black}

const (
	whiteIndex = 0 // first color in palette
	blackIndex = 1 // next color in palette
)

func main() {
	// The sequence of images is deterministic unless we seed
	// the pseudo-random number generator using the current time.
	// Thanks to Randall McPherson for pointing out the omission.
	rand.Seed(time.Now().UTC().UnixNano())
	lissajous(os.Stdout)
}

func lissajous(out io.Writer) {
	const (
		cycles  = 5     // number of complete x oscillator revolutions
		res     = 0.001 // angular resolution
		size    = 100   // image canvas covers [-size..+size]
		nframes = 64    // number of animation frames
		delay   = 8     // delay between frames in 10ms units
	)

	freq := rand.Float64() * 3.0 // relative frequency of y oscillator
	anim := gif.GIF{LoopCount: nframes}
	phase := 0.0 // phase difference
	for i := 0; i < nframes; i++ {
		rect := image.Rect(0, 0, 2*size+1, 2*size+1)
		img := image.NewPaletted(rect, palette)
		for t := 0.0; t < cycles*2*math.Pi; t += res {
			x := math.Sin(t)
			y := math.Sin(t*freq + phase)
			img.SetColorIndex(size+int(x*size+0.5), size+int(y*size+0.5),
				blackIndex)
		}
		phase += 0.1
		anim.Delay = append(anim.Delay, delay)
		anim.Image = append(anim.Image, img)
	}
	gif.EncodeAll(out, &anim) // NOTE: ignoring encoding errors
}

```

当我们`import`了一个包路径包含有多个单词的`package`时，比如`image/color`（`image`和`color`两个单词），通常我们只需要用最后那个单词表示这个包就可以。所以当我们写`color.White`时，这个变量指向的是`image/color`包里的变量，同理`gif.GIF`是属于`image/gif`包里的变量。

这个程序里的常量声明给出了一系列的常量值，常量是指在程序编译后运行时始终都不会变化的值，比如圈数、帧数、延迟值。常量声明和变量声明一般都会出现在包级别，所以这些常量在整个包中都是可以共享的，或者你也可以把常量声明定义在函数体内部，那么这种常量就只能在函数体内用。目前常量声明的值必须是一个数字值、字符串或者一个固定的`boolean`值。

`[]color.Color{...}`和`gif.GIF{...}`这两个表达式就是我们说的复合声明（4.2和4.4.1节有说明）。这是实例化Go语言里的复合类型的一种写法。这里的前者生成的是一个`slice`切片，后者生成的是一个`struct`结构体。

`gif.GIF`是一个struct类型（参考4.4节）。struct是一组值或者叫字段的集合，不同的类型集合在一个struct可以让我们以一个统一的单元进行处理。`anim`是一个`gif.GIF`类型的struct变量。这种写法会生成一个struct变量，并且其内部变量`LoopCount`字段会被设置为`nframes`；而其它的字段会被设置为各自类型默认的零值。struct内部的变量可以以一个点（`.`）来进行访问，就像在最后两个赋值语句中显式地更新了`anim`这个struct的`Delay`和`Image`字段。

`lissajous`函数内部有两层嵌套的`for`循环。外层循环会循环64次，每一次都会生成一个单独的动画帧。它生成了一个包含两种颜色的`201*201`大小的图片，白色和黑色。所有像素点都会被默认设置为其零值（也就是调色板palette里的第0个值），这里我们设置的是白色。每次外层循环都会生成一张新图片，并将一些像素设置为黑色。其结果会`append`到之前结果之后。这里我们用到了`append`(参考4.2.1)内置函数，将结果`append`到`anim`中的帧列表末尾，并设置一个默认的80ms的延迟值。循环结束后所有的延迟值被编码进了GIF图片中，并将结果写入到输出流。`out`这个变量是`io.Writer`类型，这个类型支持把输出结果写到很多目标，很快我们就可以看到例子。

内层循环设置两个偏振值。`x`轴偏振使用`sin`函数。`y`轴偏振也是正弦波，但其相对x轴的偏振是一个`0-3`的随机值，初始偏振值是一个零值，随着动画的每一帧逐渐增加。循环会一直跑到`x`轴完成五次完整的循环。每一步它都会调用`SetColorIndex`来为`(x,y)`点来染黑色。

`main`函数调用`lissajous`函数，用它来向标准输出流打印信息，所以下面这个命令会像图1.1中产生一个GIF动画。

```bash
$ go build gopl.io/ch1/lissajous
$ ./lissajous >out.gif
```

**练习 1.5：** 修改前面的`Lissajous`程序里的调色板，由黑色改为绿色。我们可以用`color.RGBA{0xRR, 0xGG, 0xBB, 0xff}`来得到`#RRGGBB`这个色值，三个十六进制的字符串分别代表红、绿、蓝像素。

```go
package main

import (
	"image"
	"image/color"
	"image/gif"
	"io"
	"math"
	"math/rand"
	"os"
)

// Packages not needed by version in book.
import (
	"log"
	"net/http"
	"time"
)

var palette = []color.Color{color.Black, color.RGBA{0, 255, 0, 255}}

const (
	blackIndex = 0 // first color in palette
	greenIndex = 1 // next color in palette
)

func main() {
	// The sequence of images is deterministic unless we seed
	// the pseudo-random number generator using the current time.
	// Thanks to Randall McPherson for pointing out the omission.
	rand.Seed(time.Now().UTC().UnixNano())

	if len(os.Args) > 1 && os.Args[1] == "web" {
		handler := func(w http.ResponseWriter, r *http.Request) {
			lissajous(w)
		}
		http.HandleFunc("/", handler)
		log.Fatal(http.ListenAndServe("localhost:8000", nil))
		return
	}
	lissajous(os.Stdout)
}

func lissajous(out io.Writer) {
	const (
		cycles  = 5     // number of complete x oscillator revolutions
		res     = 0.001 // angular resolution
		size    = 100   // image canvas covers [-size..+size]
		nframes = 64    // number of animation frames
		delay   = 8     // delay between frames in 10ms units
	)
	freq := rand.Float64() * 3.0 // relative frequency of y oscillator
	anim := gif.GIF{LoopCount: nframes}
	phase := 0.0 // phase difference
	for i := 0; i < nframes; i++ {
		rect := image.Rect(0, 0, 2*size+1, 2*size+1)
		img := image.NewPaletted(rect, palette)
		for t := 0.0; t < cycles*2*math.Pi; t += res {
			x := math.Sin(t)
			y := math.Sin(t*freq + phase)
			img.SetColorIndex(size+int(x*size+0.5), size+int(y*size+0.5),
				greenIndex)
		}
		phase += 0.1
		anim.Delay = append(anim.Delay, delay)
		anim.Image = append(anim.Image, img)
	}
	gif.EncodeAll(out, &anim) // NOTE: ignoring encoding errors
}
```

**练习 1.6：** 修改`Lissajous`程序，修改其调色板来生成更丰富的颜色，然后修改`SetColorIndex`的第三个参数，看看显示结果吧。

```go
package main

import (
	"image"
	"image/color"
	"image/gif"
	"io"
	"math"
	"math/rand"
	"os"
)

// Packages not needed by version in book.
import (
	"log"
	"net/http"
	"time"
)

func main() {
	// The sequence of images is deterministic unless we seed
	// the pseudo-random number generator using the current time.
	// Thanks to Randall McPherson for pointing out the omission.
	rand.Seed(time.Now().UTC().UnixNano())

	if len(os.Args) > 1 && os.Args[1] == "web" {
		handler := func(w http.ResponseWriter, r *http.Request) {
			lissajous(w)
		}
		http.HandleFunc("/", handler)
		log.Fatal(http.ListenAndServe("localhost:8000", nil))
		return
	}
	lissajous(os.Stdout)
}

func lissajous(out io.Writer) {
	const (
		cycles  = 5     // number of complete x oscillator revolutions
		res     = 0.001 // angular resolution
		size    = 100   // image canvas covers [-size..+size]
		nframes = 64    // number of animation frames
		delay   = 8     // delay between frames in 10ms units
	)
	palette := make([]color.Color, 0, nframes)
	palette = append(palette, color.RGBA{0, 0, 0, 255})
	for i := 0; i < nframes; i++ {
		scale := float64(i) / float64(nframes)
		c := color.RGBA{uint8(55 + 200*scale), uint8(55 + 200*scale), uint8(55 + 200*scale), 255}
		palette = append(palette, c)
	}
	freq := rand.Float64() * 3.0 // relative frequency of y oscillator
	anim := gif.GIF{LoopCount: nframes}
	phase := 0.0 // phase difference
	for i := 0; i < nframes; i++ {
		rect := image.Rect(0, 0, 2*size+1, 2*size+1)
		img := image.NewPaletted(rect, palette)
		for t := 0.0; t < cycles*2*math.Pi; t += res {
			x := math.Sin(t)
			y := math.Sin(t*freq + phase)
			img.SetColorIndex(size+int(x*size+0.5), size+int(y*size+0.5), uint8((i%(len(palette)-1))+1))
		}
		phase += 0.1
		anim.Delay = append(anim.Delay, delay)
		anim.Image = append(anim.Image, img)
	}
	gif.EncodeAll(out, &anim) // NOTE: ignoring encoding errors
}
```

## 1.5. 获取URL

对于很多现代应用来说，访问互联网上的信息和访问本地文件系统一样重要。Go语言在`net`这个强大package的帮助下提供了一系列的package来做这件事情，使用这些包可以更简单地用网络收发信息，还可以建立更底层的网络连接，编写服务器程序。在这些情景下，Go语言原生的并发特性（在第八章中会介绍）显得尤其好用。

为了最简单地展示基于HTTP获取信息的方式，下面给出一个示例程序`fetch`，这个程序将获取对应的`url`，并将其源文本打印出来；这个例子的灵感来源于`curl`工具（译注：**UNIX**下的一个用来发http请求的工具，具体可以`man curl`）。当然，`curl`提供的功能更为复杂丰富，这里只编写最简单的样例。这个样例之后还会多次被用到。

*gopl.io/ch1/fetch*

```go
// Fetch prints the content found at a URL.
package main

import (
	"fmt"
	"io/ioutil"
	"net/http"
	"os"
)

func main() {
	for _, url := range os.Args[1:] {
		resp, err := http.Get(url)
		if err != nil {
			fmt.Fprintf(os.Stderr, "fetch: %v\n", err)
			os.Exit(1)
		}
		b, err := ioutil.ReadAll(resp.Body)
		resp.Body.Close()
		if err != nil {
			fmt.Fprintf(os.Stderr, "fetch: reading %s: %v\n", url, err)
			os.Exit(1)
		}
		fmt.Printf("%s", b)
	}
}
```

这个程序从两个package中导入了函数，`net/http`和`io/ioutil`包，`http.Get`函数是创建HTTP请求的函数，如果获取过程没有出错，那么会在`resp`这个结构体中得到访问的请求结果。`resp`的Body字段包括一个可读的服务器响应流。`ioutil.ReadAll`函数从`response`中读取到全部内容；将其结果保存在变量`b`中。`resp.Body.Close`关闭`resp`的Body流，防止资源泄露，`Printf`函数会将结果`b`写出到标准输出流中。

```go
$ go build gopl.io/ch1/fetch
$ ./fetch http://gopl.io
<html>
<head>
<title>The Go Programming Language</title>title>
...
```

HTTP请求如果失败了的话，会得到下面这样的结果：

```go
$ ./fetch http://bad.gopl.io
fetch: Get http://bad.gopl.io: dial tcp: lookup bad.gopl.io: no such host
```

> 译注：在大天朝的网络环境下很容易重现这种错误，下面是Windows下运行得到的错误信息：

```go
$ go run main.go http://gopl.io
fetch: Get http://gopl.io: dial tcp: lookup gopl.io: getaddrinfow: No such host is known.
```

无论哪种失败原因，我们的程序都用了`os.Exit`函数来终止进程，并且返回一个`status`错误码，其值为1。

**练习 1.7：** 函数调用`io.Copy(dst, src)`会从`src`中读取内容，并将读到的结果写入到`dst`中，使用这个函数替代掉例子中的`ioutil.ReadAll`来拷贝响应结构体到`os.Stdout`，避免申请一个缓冲区（例子中的b）来存储。记得处理`io.Copy`返回结果中的错误。

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"os"
)

func main() {
	for _, url := range os.Args[1:] {
		resp, err := http.Get(url)
		if err != nil {
			fmt.Fprintf(os.Stderr, "fetch: %v\n", err)
			os.Exit(1)
		}
		_, err = io.Copy(os.Stdout, resp.Body)
		resp.Body.Close()
		if err != nil {
			fmt.Fprintf(os.Stderr, "fetch: reading %s: %v\n", url, err)
			os.Exit(1)
		}
	}
}
```

**练习 1.8：** 修改fetch这个范例，如果输入的`url`参数没有 `http://` 前缀的话，为这个`url`加上该前缀。你可能会用到`strings.HasPrefix`这个函数。

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"os"
	"strings"
)

func main() {
	for _, url := range os.Args[1:] {
		if !strings.HasPrefix(url, "http://") {
			url = "http://" + url
		}
		resp, err := http.Get(url)
		if err != nil {
			fmt.Fprintf(os.Stderr, "fetch: %v\n", err)
			os.Exit(1)
		}
		_, err = io.Copy(os.Stdout, resp.Body)
		resp.Body.Close()
		if err != nil {
			fmt.Fprintf(os.Stderr, "fetch: reading %s: %v\n", url, err)
			os.Exit(1)
		}
	}
}
```

**练习 1.9：** 修改fetch打印出HTTP协议的状态码，可以从`resp.Status`变量得到该状态码。

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"os"
	"strings"
)

func main() {
	for _, url := range os.Args[1:] {
		if !strings.HasPrefix(url, "http://") {
			url = "http://" + url
		}
		resp, err := http.Get(url)
		if err != nil {
			fmt.Fprintf(os.Stderr, "fetch: %v\n", err)
			os.Exit(1)
		}
		_, err = io.Copy(os.Stdout, resp.Body)
		fmt.Printf("HTTP status: %d\n", resp.StatusCode)
		resp.Body.Close()
		if err != nil {
			fmt.Fprintf(os.Stderr, "fetch: reading %s: %v\n", url, err)
			os.Exit(1)
		}
	}
}
```

## 1.6. 并发获取多个URL

Go语言最有意思并且最新奇的特性就是对并发编程的支持。并发编程是一个大话题，在第八章和第九章中会专门讲到。这里我们只浅尝辄止地来体验一下Go语言里的`goroutine`和`channel`。

下面的例子`fetchall`，和前面小节的`fetch`程序所要做的工作基本一致，`fetchall`的特别之处在于它会同时去获取所有的URL，所以这个程序的总执行时间不会超过执行时间最长的那一个任务，前面的`fetch`程序执行时间则是所有任务执行时间之和。`fetchall`程序只会打印获取的内容大小和经过的时间，不会像之前那样打印获取的内容。

*gopl.io/ch1/fetchall*

```go
// Fetchall fetches URLs in parallel and reports their times and sizes.
package main

import (
	"fmt"
	"io"
	"io/ioutil"
	"net/http"
	"os"
	"time"
)

func main() {
	start := time.Now()
	ch := make(chan string)
	for _, url := range os.Args[1:] {
		go fetch(url, ch) // start a goroutine
	}
	for range os.Args[1:] {
		fmt.Println(<-ch) // receive from channel ch
	}
	fmt.Printf("%.2fs elapsed\n", time.Since(start).Seconds())
}

func fetch(url string, ch chan<- string) {
	start := time.Now()
	resp, err := http.Get(url)
	if err != nil {
		ch <- fmt.Sprint(err) // send to channel ch
		return
	}
	nbytes, err := io.Copy(ioutil.Discard, resp.Body)
	resp.Body.Close() // don't leak resources
	if err != nil {
		ch <- fmt.Sprintf("while reading %s: %v", url, err)
		return
	}
	secs := time.Since(start).Seconds()
	ch <- fmt.Sprintf("%.2fs  %7d  %s", secs, nbytes, url)
}
```

下面使用`fetchall`来请求几个地址：

```go
$ go build gopl.io/ch1/fetchall
$ ./fetchall https://golang.org http://gopl.io https://godoc.org
0.14s     6852  https://godoc.org
0.16s     7261  https://golang.org
0.48s     2475  http://gopl.io
0.48s elapsed
```

`goroutine`是一种函数的并发执行方式，而`channel`是用来在`goroutine`之间进行参数传递。`main`函数本身也运行在一个`goroutine`中，而`go function`则表示创建一个新的`goroutine`，并在这个新的`goroutine`中执行这个函数。

`main`函数中用`make`函数创建了一个传递`string`类型参数的`channel`，对每一个命令行参数，我们都用`go`这个关键字来创建一个`goroutine`，并且让函数在这个`goroutine`异步执行`http.Get`方法。这个程序里的`io.Copy`会把响应的Body内容拷贝到`ioutil.Discard`输出流中（译注：可以把这个变量看作一个垃圾桶，可以向里面写一些不需要的数据），因为我们需要这个方法返回的字节数，但是又不想要其内容。每当请求返回内容时，`fetch`函数都会往`ch`这个`channel`里写入一个字符串，由`main`函数里的第二个`for`循环来处理并打印`channel`里的这个字符串。

当一个`goroutine`尝试在一个`channel`上做`send`或者`receive`操作时，这个`goroutine`会阻塞在调用处，直到另一个`goroutine`从这个`channel`里接收或者写入值，这样两个`goroutine`才会继续执行`channel`操作之后的逻辑。在这个例子中，每一个`fetch`函数在执行时都会往`channel`里发送一个值（`ch <- expression`），主函数负责接收这些值（`<-ch`）。这个程序中我们用`main`函数来接收所有`fetch`函数传回的字符串，可以避免在`goroutine`异步执行还没有完成时`main`函数提前退出。

**练习 1.10：** 找一个数据量比较大的网站，用本小节中的程序调研网站的缓存策略，对每个URL执行两遍请求，查看两次时间是否有较大的差别，并且每次获取到的响应内容是否一致，修改本节中的程序，将响应结果输出，以便于进行对比。

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"net/url"
	"os"
	"time"
)

func main() {
	start := time.Now()
	ch := make(chan string)
	for _, url := range os.Args[1:] {
		go fetch(url, ch) // start a goroutine
	}
	for range os.Args[1:] {
		fmt.Println(<-ch) // receive from channel ch
	}
	fmt.Printf("%.2fs elapsed\n", time.Since(start).Seconds())
}

func fetch(uri string, ch chan<- string) {
	start := time.Now()
	resp, err := http.Get(uri)
	if err != nil {
		ch <- fmt.Sprint(err) // send to channel ch
		return
	}

	f, err := os.Create(url.QueryEscape(uri))
	if err != nil {
		ch <- err.Error()
	}
	nbytes, err := io.Copy(f, resp.Body)
	resp.Body.Close() // don't leak resources

	if closeErr := f.Close(); err == nil { // example from chapter 5.8, gopl.io/ch5/fetch
		err = closeErr
	}

	if err != nil {
		ch <- fmt.Sprintf("while reading %s: %v", uri, err)
		return
	}
	secs := time.Since(start).Seconds()
	ch <- fmt.Sprintf("%.2fs  %7d  %s", secs, nbytes, uri)
}
```

**练习 1.11：** 在`fetchall`中尝试使用长一些的参数列表，比如使用在alexa.com的上百万网站里排名靠前的。如果一个网站没有回应，程序将采取怎样的行为？（Section8.9 描述了在这种情况下的应对机制）。

## 1.7. Web服务

Go语言的内置库使得写一个类似`fetch`的`web`服务器变得异常地简单。在本节中，我们会展示一个微型服务器，这个服务器的功能是返回当前用户正在访问的URL。比如用户访问的是 http://localhost:8000/hello ，那么响应是`URL.Path = "hello"`。

gopl.io/ch1/server1

```go
// Server1 is a minimal "echo" server.
package main

import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/", handler) // each request calls handler
	log.Fatal(http.ListenAndServe("localhost:8000", nil))
}

// handler echoes the Path component of the request URL r.
func handler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "URL.Path = %q\n", r.URL.Path)
}
```

我们只用了八九行代码就实现了一个Web服务程序，这都是多亏了标准库里的方法已经帮我们完成了大量工作。`main`函数将所有发送到`/`路径下的请求和`handler`函数关联起来，`/`开头的请求其实就是所有发送到当前站点上的请求，服务监听`8000`端口。发送到这个服务的“请求”是一个`http.Request`类型的对象，这个对象中包含了请求中的一系列相关字段，其中就包括我们需要的URL。当请求到达服务器时，这个请求会被传给`handler`函数来处理，这个函数会将`/hello`这个路径从请求的URL中解析出来，然后把其发送到响应中，这里我们用的是标准输出流的`fmt.Fprintf`。Web服务会在第7.7节中做更详细的阐述。

让我们在后台运行这个服务程序。如果你的操作系统是Mac OS X或者Linux，那么在运行命令的末尾加上一个`&`符号，即可让程序简单地跑在后台，windows下可以在另外一个命令行窗口去运行这个程序。

```go
$ go run src/gopl.io/ch1/server1/main.go &
```

现在可以通过命令行来发送客户端请求了：

```go
$ go build gopl.io/ch1/fetch
$ ./fetch http://localhost:8000
URL.Path = "/"
$ ./fetch http://localhost:8000/help
URL.Path = "/help"
```

还可以直接在浏览器里访问这个URL，然后得到返回结果，如图1.2：

![Web服务 - 图1](images\ch1-02.png)

在这个服务的基础上叠加特性是很容易的。一种比较实用的修改是为访问的`url`添加某种状态。比如，下面这个版本输出了同样的内容，但是会对请求的次数进行计算；对URL的请求结果会包含各种URL被访问的总次数，直接对`/count`这个URL的访问要除外。

<u><i>gopl.io/ch1/server2</i></u>

```go
// Server2 is a minimal "echo" and counter server.
package main

import (
	"fmt"
	"log"
	"net/http"
	"sync"
)

var mu sync.Mutex
var count int

func main() {
	http.HandleFunc("/", handler)
	http.HandleFunc("/count", counter)
	log.Fatal(http.ListenAndServe("localhost:8000", nil))
}

// handler echoes the Path component of the requested URL.
func handler(w http.ResponseWriter, r *http.Request) {
	mu.Lock()
	count++
	mu.Unlock()
	fmt.Fprintf(w, "URL.Path = %q\n", r.URL.Path)
}

// counter echoes the number of calls so far.
func counter(w http.ResponseWriter, r *http.Request) {
	mu.Lock()
	fmt.Fprintf(w, "Count %d\n", count)
	mu.Unlock()
}
```

这个服务器有两个请求处理函数，根据请求的`url`不同会调用不同的函数：对`/count`这个`url`的请求会调用到`counter`这个函数，其它的`url`都会调用默认的处理函数。如果你的请求`pattern`是以`/`结尾，那么所有以该`url`为前缀的`url`都会被这条规则匹配。在这些代码的背后，服务器每一次接收请求处理时都会另起一个`goroutine`，这样服务器就可以同一时间处理多个请求。然而在并发情况下，假如真的有两个请求同一时刻去更新`count`，那么这个值可能并不会被正确地增加；这个程序可能会引发一个严重的bug：**竞态条件**（参见9.1）。为了避免这个问题，我们必须保证每次修改变量的最多只能有一个`goroutine`，这也就是代码里的`mu.Lock()`和`mu.Unlock()`调用将修改`count`的所有行为包在中间的目的。第九章中我们会进一步讲解共享变量。

下面是一个更为丰富的例子，`handler`函数会把请求的`http`头和请求的`form`数据都打印出来，这样可以使检查和调试这个服务更为方便：

gopl.io/ch1/server3

```go
// handler echoes the HTTP request.
func handler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "%s %s %s\n", r.Method, r.URL, r.Proto)
	for k, v := range r.Header {
		fmt.Fprintf(w, "Header[%q] = %q\n", k, v)
	}
	fmt.Fprintf(w, "Host = %q\n", r.Host)
	fmt.Fprintf(w, "RemoteAddr = %q\n", r.RemoteAddr)
	if err := r.ParseForm(); err != nil {
		log.Print(err)
	}
	for k, v := range r.Form {
		fmt.Fprintf(w, "Form[%q] = %q\n", k, v)
	}
}
```

我们用`http.Request`这个struct里的字段来输出下面这样的内容：

```go
GET /?q=query HTTP/1.1
Header["Accept-Encoding"] = ["gzip, deflate, sdch"]
Header["Accept-Language"] = ["en-US,en;q=0.8"]
Header["Connection"] = ["keep-alive"]
Header["Accept"] = ["text/html,application/xhtml+xml,application/xml;..."]
Header["User-Agent"] = ["Mozilla/5.0 (Macintosh; Intel Mac OS X 10_7_5)..."]
Host = "localhost:8000"
RemoteAddr = "127.0.0.1:59911"
Form["q"] = ["query"]
```

可以看到这里的`ParseForm`被嵌套在了`if`语句中。Go语言允许这样的一个简单的语句结果作为局部的变量声明出现在`if`语句的最前面，这一点对错误处理很有用处。我们还可以像下面这样写（当然看起来就长了一些）：

```go
err := r.ParseForm()
if err != nil {
	log.Print(err)
}
```

用`if`和`ParseForm`结合可以让代码更加简单，并且可以限制`err`这个变量的作用域，这么做是很不错的。我们会在2.7节中讲解作用域。

在这些程序中，我们看到了很多不同的类型被输出到标准输出流中。比如前面的`fetch`程序，把HTTP的响应数据拷贝到了`os.Stdout`，`lissajous`程序里我们输出的是一个文件。`fetchall`程序则完全忽略到了HTTP的响应Body，只是计算了一下响应Body的大小，这个程序中把响应Body拷贝到了`ioutil.Discard`。在本节的web服务器程序中则是用`fmt.Fprintf`直接写到了`http.ResponseWriter`中。

尽管三种具体的实现流程并不太一样，他们都实现一个共同的接口，即当它们被调用需要一个标准流输出时都可以满足。这个接口叫作`io.Writer`，在7.1节中会详细讨论。

Go语言的接口机制会在第7章中讲解，为了在这里简单说明接口能做什么，让我们简单地将这里的web服务器和之前写的`lissajous`函数结合起来，这样GIF动画可以被写到HTTP的客户端，而不是之前的标准输出流。只要在web服务器的代码里加入下面这几行。

```Go
handler := func(w http.ResponseWriter, r *http.Request) {
	lissajous(w)
}
http.HandleFunc("/", handler)
```

或者另一种等价形式：

```Go
http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
	lissajous(w)
})
```

`HandleFunc`函数的第二个参数是一个函数的字面值，也就是一个在使用时定义的匿名函数。这些内容我们会在5.6节中讲解。

做完这些修改之后，在浏览器里访问 http://localhost:8000 。每次你载入这个页面都可以看到一个像图1.3那样的动画。

![Web服务 - 图2](images\ch1-03.png)

**练习 1.12：** 修改`Lissajour`服务，从URL读取变量，比如你可以访问 http://localhost:8000/?cycles=20 这个URL，这样访问可以将程序里的`cycles`默认的5修改为20。字符串转换为数字可以调用`strconv.Atoi`函数。你可以在`godoc`里查看`strconv.Atoi`的详细说明。

```go
package main

import (
	"fmt"
	"image"
	"image/color"
	"image/gif"
	"io"
	"log"
	"math"
	"math/rand"
	"net/http"
	"os"
	"strconv"
	"time"
)

// Packages not needed by version in book.

func main() {
	var cycles = 5 // number of complete x oscillator revolutions

	// The sequence of images is deterministic unless we seed
	// the pseudo-random number generator using the current time.
	// Thanks to Randall McPherson for pointing out the omission.
	rand.Seed(time.Now().UTC().UnixNano())

	if len(os.Args) > 1 && os.Args[1] == "web" {
		handler := func(w http.ResponseWriter, r *http.Request) {
			cyclesStr := r.FormValue("cycles")
			if cyclesStr != "" {
				var err error
				cycles, err = strconv.Atoi(cyclesStr)
				if err != nil {
					fmt.Fprintf(w, "bad cycles param: %s %s", cyclesStr, err)
					return
				}
			}
			lissajous(cycles, w)
		}
		http.HandleFunc("/", handler)
		log.Fatal(http.ListenAndServe("localhost:8000", nil))
		return
	}
	lissajous(cycles, os.Stdout)
}

func lissajous(cycles int, out io.Writer) {
	const (
		res     = 0.001 // angular resolution
		size    = 100   // image canvas covers [-size..+size]
		nframes = 64    // number of animation frames
		delay   = 8     // delay between frames in 10ms units
	)
	palette := make([]color.Color, 0, nframes)
	palette = append(palette, color.RGBA{0, 0, 0, 255})
	for i := 0; i < nframes; i++ {
		scale := float64(i) / float64(nframes)
		c := color.RGBA{uint8(55 + 200*scale), uint8(55 + 200*scale), uint8(55 + 200*scale), 255}
		palette = append(palette, c)
	}
	freq := rand.Float64() * 3.0 // relative frequency of y oscillator
	anim := gif.GIF{LoopCount: nframes}
	phase := 0.0 // phase difference
	for i := 0; i < nframes; i++ {
		rect := image.Rect(0, 0, 2*size+1, 2*size+1)
		img := image.NewPaletted(rect, palette)
		for t := 0.0; t < float64(cycles)*2*math.Pi; t += res {
			x := math.Sin(t)
			y := math.Sin(t*freq + phase)
			img.SetColorIndex(size+int(x*size+0.5), size+int(y*size+0.5), uint8((i%(len(palette)-1))+1))
		}
		phase += 0.1
		anim.Delay = append(anim.Delay, delay)
		anim.Image = append(anim.Image, img)
	}
	gif.EncodeAll(out, &anim) // NOTE: ignoring encoding errors
}
```

## 1.8. 本章要点

本章对Go语言做了一些介绍，Go语言很多方面在有限的篇幅中无法覆盖到。本节会把没有讲到的内容也做一些简单的介绍，这样读者在读到完整的内容之前，可以有个简单的印象。

**控制流：** 在本章我们只介绍了`if`控制和`for`，但是没有提到`switch`多路选择。这里是一个简单的`switch`的例子：

```go
switch coinflip() {
case "heads":
	heads++
case "tails":
	tails++
default:
	fmt.Println("landed on edge!")
}
```

在翻转硬币的时候，例子里的`coinflip`函数返回几种不同的结果，每一个`case`都会对应一个返回结果，这里需要注意，Go语言并不需要显式地在每一个`case`后写`break`，语言默认执行完`case`后的逻辑语句会自动退出。当然了，如果你想要相邻的几个`case`都执行同一逻辑的话，需要自己显式地写上一个`fallthrough`语句来覆盖这种默认行为。不过`fallthrough`语句在一般的程序中很少用到。

Go语言里的`switch`还可以不带操作对象（译注：`switch`不带操作对象时默认用`true`值代替，然后将每个`case`的表达式和`true`值进行比较）；可以直接罗列多种条件，像其它语言里面的多个`if else`一样，下面是一个例子：

```go
func Signum(x int) int {
	switch {
	case x > 0:
		return +1
	default:
		return 0
	case x < 0:
		return -1
	}
}
```

这种形式叫做无`tag switch`(tagless switch)；这和switch true是等价的。

像`for`和if控制语句一样，`switch`也可以紧跟一个简短的变量声明，一个自增表达式、赋值语句，或者一个函数调用（译注：比其它语言丰富）。

`break`和`continue`语句会改变控制流。和其它语言中的`break`和`continue`一样，`break`会中断当前的循环，并开始执行循环之后的内容，而`continue`会跳过当前循环，并开始执行下一次循环。这两个语句除了可以控制`for`循环，还可以用来控制`switch`和`select`语句（之后会讲到），在1.3节中我们看到，`continue`会跳过内层的循环，如果我们想跳过的是更外层的循环的话，我们可以在相应的位置加上`label`，这样`break`和`continue`就可以根据我们的想法来`continue`和`break`任意循环。这看起来甚至有点像`goto`语句的作用了。当然，一般程序员也不会用到这种操作。这两种行为更多地被用到机器生成的代码中。

**命名类型：** 类型声明使得我们可以很方便地给一个特殊类型一个名字。因为~类型声明通常非常地长，所以我们总要给这种`struct`取一个名字。本章中就有这样一个例子，二维点类型：

```go
type Point struct {
	X, Y int
}
var p Point
```

类型声明和命名类型会在第二章中介绍。

**指针：** Go语言提供了指针。指针是一种直接存储了变量的内存地址的数据类型。在其它语言中，比如C语言，指针操作是完全不受约束的。在另外一些语言中，指针一般被处理为“引用”，除了到处传递这些指针之外，并不能对这些指针做太多事情。Go语言在这两种范围中取了一种平衡。指针是可见的内存地址，`&`操作符可以返回一个变量的内存地址，并且`*`操作符可以获取指针指向的变量内容，但是在Go语言里没有指针运算，也就是不能像c语言里可以对指针进行加或减操作。我们会在2.3.2中进行详细介绍。

**方法和接口：** 方法是和命名类型关联的一类函数。Go语言里比较特殊的是方法可以被关联到任意一种命名类型。在第六章我们会详细地讲方法。接口是一种抽象类型，这种类型可以让我们以同样的方式来处理不同的固有类型，不用关心它们的具体实现，而只需要关注它们提供的方法。第七章中会详细说明这些内容。

**包（packages）：** Go语言提供了一些很好用的package，并且这些package是可以扩展的。Go语言社区已经创造并且分享了很多很多。所以Go语言编程大多数情况下就是用已有的package来写我们自己的代码。通过这本书，我们会讲解一些重要的标准库内的package，但是还是有很多限于篇幅没有去说明，因为我们没法在这样的厚度的书里去做一部代码大全。

在你开始写一个新程序之前，最好先去检查一下是不是已经有了现成的库可以帮助你更高效地完成这件事情。你可以在 https://golang.org/pkg 和 https://godoc.org 中找到标准库和社区写的package。`godoc`这个工具可以让你直接在本地命令行阅读标准库的文档。比如下面这个例子。

```go
$ go doc http.ListenAndServe
package http // import "net/http"
func ListenAndServe(addr string, handler Handler) error
    ListenAndServe listens on the TCP network address addr and then
    calls Serve with handler to handle requests on incoming connections.
...
```

**注释：** 我们之前已经提到过了在源文件的开头写的注释是这个源文件的文档。在每一个函数之前写一个说明函数行为的注释也是一个好习惯。这些惯例很重要，因为这些内容会被像`godoc`这样的工具检测到，并且在执行命令时显示这些注释。具体可以参考10.7.4。

多行注释可以用 `/* ... */` 来包裹，和其它大多数语言一样。在文件一开头的注释一般都是这种形式，或者一大段的解释性的注释文字也会被这符号包住，来避免每一行都需要加`//`。在注释中`//`和`/*`是没什么意义的，所以不要在注释中再嵌入注释。

