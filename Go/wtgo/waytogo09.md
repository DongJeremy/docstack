# 第9章：包（package）

本章主要针对 Go 语言的包展开讲解。

## 9.1 标准库概述

像 `fmt`、`os` 等这样具有常用功能的内置包在 Go 语言中有 150 个以上，它们被称为标准库，大部分(一些底层的除外)内置于 Go 本身。完整列表可以在 [Go Walker](https://gowalker.org/search?q=gorepos) 查看。

在贯穿本书的例子和练习中，我们都是用标准库的包。可以通过查阅第 350 页包中的内容快速找到相关的包的实例。这里我们只是按功能进行分组来介绍这些包的简单用途，我们不会深入讨论他们的内部结构。

- `unsafe`: 包含了一些打破 Go 语言“类型安全”的命令，一般的程序中不会被使用，可用在 C/C++ 程序的调用中。
- `syscall`-`os`-`os/exec`:  
	- `os`: 提供给我们一个平台无关性的操作系统功能接口，采用类Unix设计，隐藏了不同操作系统间差异，让不同的文件系统和操作系统对象表现一致。  
	- `os/exec`: 提供我们运行外部操作系统命令和程序的方式。  
	- `syscall`: 底层的外部包，提供了操作系统底层调用的基本接口。

通过一个 Go 程序让Linux重启来体现它的能力。

示例 9.1 [reboot.go](examples/chapter_9/reboot.go)：

```go
package main
import (
	"syscall"
)

const LINUX_REBOOT_MAGIC1 uintptr = 0xfee1dead
const LINUX_REBOOT_MAGIC2 uintptr = 672274793
const LINUX_REBOOT_CMD_RESTART uintptr = 0x1234567

func main() {
	syscall.Syscall(syscall.SYS_REBOOT,
		LINUX_REBOOT_MAGIC1,
		LINUX_REBOOT_MAGIC2,
		LINUX_REBOOT_CMD_RESTART)
}
```

- `archive/tar` 和 `/zip-compress`：压缩(解压缩)文件功能。
- `fmt`-`io`-`bufio`-`path/filepath`-`flag`:  
	- `fmt`: 提供了格式化输入输出功能。  
	- `io`: 提供了基本输入输出功能，大多数是围绕系统功能的封装。  
	- `bufio`: 缓冲输入输出功能的封装。  
	- `path/filepath`: 用来操作在当前系统中的目标文件名路径。  
	- `flag`: 对命令行参数的操作。　　
- `strings`-`strconv`-`unicode`-`regexp`-`bytes`:  
	- `strings`: 提供对字符串的操作。  
	- `strconv`: 提供将字符串转换为基础类型的功能。
	- `unicode`: 为 `unicode` 型的字符串提供特殊的功能。
	- `regexp`: 正则表达式功能。  
	- `bytes`: 提供对字符型分片的操作。  
	- `index/suffixarray`: 子字符串快速查询。
- `math`-`math/cmath`-`math/big`-`math/rand`-`sort`:  
	- `math`: 基本的数学函数。  
	- `math/cmath`: 对复数的操作。  
	- `math/rand`: 伪随机数生成。  
	- `sort`: 为数组排序和自定义集合。  
	- `math/big`: 大数的实现和计算。  　　
- `container`-`/list-ring-heap`: 实现对集合的操作。  
	- `list`: 双链表。
	- `ring`: 环形链表。

下面代码演示了如何遍历一个链表(当 l 是 `*List`)：

```go
for e := l.Front(); e != nil; e = e.Next() {
	//do something with e.Value
}
```

- `time`-`log`:  
	- `time`: 日期和时间的基本操作。  
	- `log`: 记录程序运行时产生的日志,我们将在后面的章节使用它。
- `encoding/json`-`encoding/xml`-`text/template`:
	- `encoding/json`: 读取并解码和写入并编码 JSON 数据。  
	- `encoding/xml`:简单的 XML1.0 解析器,有关 JSON 和 XML 的实例请查阅第 12.9/10 章节。  
	- `text/template`:生成像 HTML 一样的数据与文本混合的数据驱动模板（参见第 15.7 节）。  
- `net`-`net/http`-`html`:（参见第 15 章）
	- `net`: 网络数据的基本操作。  
	- `http`: 提供了一个可扩展的 HTTP 服务器和基础客户端，解析 HTTP 请求和回复。  
	- `html`: HTML5 解析器。  
- `runtime`: Go 程序运行时的交互操作，例如垃圾回收和协程创建。  
- `reflect`: 实现通过程序运行时反射，让程序操作任意类型的变量。  

`exp` 包中有许多将被编译为新包的实验性的包。它们将成为独立的包在下次稳定版本发布的时候。如果前一个版本已经存在了，它们将被作为过时的包被回收。然而 Go1.0 发布的时候并不包含过时或者实验性的包。

**练习 9.1**

使用 `container/list` 包实现一个双向链表，将 101、102 和 103 放入其中并打印出来。

```go

```



**练习 9.2**

通过使用 `unsafe` 包中的方法来测试你电脑上一个整型变量占用多少个字节。

```go

```

## 9.2 regexp 包

正则表达式语法和使用的详细信息请参考 [维基百科](http://en.wikipedia.org/wiki/Regular_expression)。

在下面的程序里，我们将在字符串中对正则表达式模式（`pattern`）进行匹配。

如果是简单模式，使用 `Match` 方法便可：

```go
ok, _ := regexp.Match(pat, []byte(searchIn))
```

变量 `ok` 将返回 `true` 或者 `false`,我们也可以使用 `MatchString`：

```go
ok, _ := regexp.MatchString(pat, searchIn)
```

更多方法中，必须先将正则模式通过 `Compile` 方法返回一个 `Regexp` 对象。然后我们将掌握一些匹配，查找，替换相关的功能。

示例 9.2 [pattern.go](examples/chapter_9/pattern.go)：

```go
package main
import (
	"fmt"
	"regexp"
	"strconv"
)
func main() {
	//目标字符串
	searchIn := "John: 2578.34 William: 4567.23 Steve: 5632.18"
	pat := "[0-9]+.[0-9]+" //正则

	f := func(s string) string{
    	v, _ := strconv.ParseFloat(s, 32)
    	return strconv.FormatFloat(v * 2, 'f', 2, 32)
	}

	if ok, _ := regexp.Match(pat, []byte(searchIn)); ok {
    fmt.Println("Match Found!")
	}

	re, _ := regexp.Compile(pat)
	//将匹配到的部分替换为"##.#"
	str := re.ReplaceAllString(searchIn, "##.#")
	fmt.Println(str)
	//参数为函数时
	str2 := re.ReplaceAllStringFunc(searchIn, f)
	fmt.Println(str2)
}
```

输出结果：

```go
Match Found!
John: ##.# William: ##.# Steve: ##.#
John: 5156.68 William: 9134.46 Steve: 11264.36
```

`Compile` 函数也可能返回一个错误，我们在使用时忽略对错误的判断是因为我们确信自己正则表达式是有效的。当用户输入或从数据中获取正则表达式的时候，我们有必要去检验它的正确性。另外我们也可以使用 `MustCompile` 方法，它可以像 `Compile` 方法一样检验正则的有效性，但是当正则不合法时程序将 `panic`（详情查看第 13.2 节)。

## 9.3 锁和 sync 包

在一些复杂的程序中，通常通过不同线程执行不同应用来实现程序的并发。当不同线程要使用同一个变量时，经常会出现一个问题：无法预知变量被不同线程修改的顺序！(这通常被称为资源竞争,指不同线程对同一变量使用的竞争)显然这无法让人容忍，那我们该如何解决这个问题呢？

经典的做法是一次只能让一个线程对共享变量进行操作。当变量被一个线程改变时(临界区)，我们为它上锁，直到这个线程执行完成并解锁后，其他线程才能访问它。

特别是我们之前章节学习的 `map` 类型是不存在锁的机制来实现这种效果(出于对性能的考虑)，所以 `map` 类型是非线程安全的。当并行访问一个共享的 `map` 类型的数据，`map` 数据将会出错。

在 Go 语言中这种锁的机制是通过 `sync` 包中 `Mutex` 来实现的。`sync` 来源于 "synchronized" 一词，这意味着线程将有序的对同一变量进行访问。

`sync.Mutex` 是一个互斥锁，它的作用是守护在临界区入口来确保同一时间只能有一个线程进入临界区。

假设 info 是一个需要上锁的放在共享内存中的变量。通过包含 `Mutex` 来实现的一个典型例子如下：

```go
import  "sync"

type Info struct {
	mu sync.Mutex
	// ... other fields, e.g.: Str string
}
```

如果一个函数想要改变这个变量可以这样写:

```go
func Update(info *Info) {
	info.mu.Lock()
    // critical section:
    info.Str = // new value
    // end critical section
    info.mu.Unlock()
}
```

还有一个很有用的例子是通过 `Mutex` 来实现一个可以上锁的共享缓冲器:

```go
type SyncedBuffer struct {
	lock 	sync.Mutex
	buffer  bytes.Buffer
}
```

在 `sync` 包中还有一个 `RWMutex` 锁：他能通过 `RLock()` 来允许同一时间多个线程对变量进行读操作，但是只能一个线程进行写操作。如果使用 `Lock()` 将和普通的 `Mutex` 作用相同。包中还有一个方便的 `Once` 类型变量的方法 `once.Do(call)`，这个方法确保被调用函数只能被调用一次。

相对简单的情况下，通过使用 `sync` 包可以解决同一时间只能一个线程访问变量或 `map` 类型数据的问题。如果这种方式导致程序明显变慢或者引起其他问题，我们要重新思考来通过 `goroutines` 和 `channels` 来解决问题，这是在 Go 语言中所提倡用来实现并发的技术。我们将在第 14 章对其深入了解，并在第 14.7 节中对这两种方式进行比较。

## 9.4 精密计算和 big 包

我们知道有些时候通过编程的方式去进行计算是不精确的。如果你使用 Go 语言中的 `float64` 类型进行浮点运算，返回结果将精确到 15 位，足以满足大多数的任务。当对超出 `int64` 或者 `uint64` 类型这样的大数进行计算时，如果对精度没有要求，`float32` 或者 `float64` 可以胜任，但如果对精度有严格要求的时候，我们不能使用浮点数，在内存中它们只能被近似的表示。

对于整数的高精度计算 Go 语言中提供了 `big` 包，被包含在 math 包下：有用来表示大整数的 `big.Int` 和表示大有理数的 `big.Rat` 类型（可以表示为 2/5 或 3.1416 这样的分数，而不是无理数或 π）。这些类型可以实现任意位类型的数字，只要内存足够大。缺点是更大的内存和处理开销使它们使用起来要比内置的数字类型慢很多。

大的整型数字是通过 `big.NewInt(n)` 来构造的，其中 n 为 int64 类型整数。而大有理数是通过 `big.NewRat(n, d)` 方法构造。n（分子）和 d（分母）都是 int64 型整数。因为 Go 语言不支持运算符重载，所以所有大数字类型都有像是 `Add()` 和 `Mul()` 这样的方法。它们作用于作为 `receiver` 的整数和有理数，大多数情况下它们修改 `receiver` 并以 `receiver` 作为返回结果。因为没有必要创建 `big.Int` 类型的临时变量来存放中间结果，所以运算可以被链式地调用，并节省内存。

示例 9.2 [big.go](examples/chapter_9/big.go)：

``` go
// big.go
package main

import (
	"fmt"
	"math"
	"math/big"
)

func main() {
	// Here are some calculations with bigInts:
	im := big.NewInt(math.MaxInt64)
	in := im
	io := big.NewInt(1956)
	ip := big.NewInt(1)
	ip.Mul(im, in).Add(ip, im).Div(ip, io)
	fmt.Printf("Big Int: %v\n", ip)
	// Here are some calculations with bigInts:
	rm := big.NewRat(math.MaxInt64, 1956)
	rn := big.NewRat(-1956, math.MaxInt64)
	ro := big.NewRat(19, 56)
	rp := big.NewRat(1111, 2222)
	rq := big.NewRat(1, 1)
	rq.Mul(rm, rn).Add(rq, ro).Mul(rq, rp)
	fmt.Printf("Big Rat: %v\n", rq)
}

/* Output:
Big Int: 43492122561469640008497075573153004
Big Rat: -37/112
*/
```

输出结果：

	Big Int: 43492122561469640008497075573153004
	Big Rat: -37/112

## 9.5 自定义包和可见性

包是 Go 语言中代码组织和代码编译的主要方式。关于它们的很多基本信息已经在 4.2 章节中给出，最引人注目的便是可见性。现在我们来看看具体如何来使用自己写的包。在下一节，我们将回顾一些标准库中的包，自定义的包和标准库以外的包。

当写自己包的时候，要使用短小的不含有 `_`(下划线)的小写单词来为文件命名。这里有个简单例子来说明包是如何相互调用以及可见性是如何实现的。

当前目录下（`examples/chapter_9/book/`）有一个名为 `package_test.go` 的程序, 它使用了自定义包 `pack1` 中 `pack1.go` 的代码。这段程序(连同编译链接生成的 `pack1.a`)存放在当前目录下一个名为 `pack1` 的文件夹下。所以链接器将包的对象和主程序对象链接在一起。

示例 9.4 [pack1.go](examples/chapter_9/book/pack1/pack1.go)：

```go
package pack1
var Pack1Int int = 42
var pack1Float = 3.14

func ReturnStr() string {
	return "Hello main!"
}
```

它包含了一个整型变量 `Pack1Int` 和一个返回字符串的函数 `ReturnStr`。这段程序在运行时不做任何的事情，因为它不包含有一个 `main` 函数。

在主程序 `package_test.go` 中这个包通过声明的方式被导入

```go
import "./pack1/pack1"
```

`import` 的一般格式如下:

```go
import "包的路径或 URL 地址" 
```

例如：

```go
import "github.com/org1/pack1”
```

路径是指当前目录的相对路径。

示例 9.5 [package_test.go](examples/chapter_9/book/package_test.go)：

```go
package main

import (
	"fmt"
	"./pack1/pack1"
)

func main() {
	var test1 string
	test1 = pack1.ReturnStr()
	fmt.Printf("ReturnStr from package1: %s\n", test1)
	fmt.Printf("Integer from package1: %d\n", pack1.Pack1Int)
	// fmt.Printf("Float from package1: %f\n", pack1.pack1Float)
}
```

输出结果：

	ReturnStr from package1: Hello main!
	Integer from package1: 42

如果包 pack1 和我们的程序在同一路径下，我们可以通过 `"import ./pack1"` 这样的方式来引入，但这不被视为一个好的方法。

下面的代码试图访问一个未引用的变量或者函数，甚至没有编译。将会返回一个错误：

```go
fmt.Printf("Float from package1: %f\n", pack1.pack1Float)
```

错误：
	cannot refer to unexported name pack1.pack1Float

主程序利用的包必须在主程序编写之前被编译。主程序中每个 pack1 项目都要通过包名来使用：`pack1.Item`。具体使用方法请参见示例 4.6 和 4.7。

因此，按照惯例,子目录和包之间有着密切的联系：为了区分,不同包存放在不同的目录下，每个包(所有属于这个包中的 go 文件)都存放在和包名相同的子目录下：

Import with `.` :  
```go
import . "./pack1"
```

当使用`.`来做为包的别名时，你可以不通过包名来使用其中的项目。例如：`test := ReturnStr()`。

在当前的命名空间导入 pack1 包，一般是为了具有更好的测试效果。

Import with `_` : 

```go
import _ "./pack1/pack1"
```

pack1包只导入其副作用，也就是说，只执行它的`init`函数并初始化其中的全局变量。

**导入外部安装包:**

如果你要在你的应用中使用一个或多个外部包，首先你必须使用 `go install`（参见第 9.7 节）在你的本地机器上安装它们。

假设你想使用 `http://codesite.ext/author/goExample/goex` 这种托管在 Google Code、GitHub 和 Launchpad 等代码网站上的包。

你可以通过如下命令安装：

```go
go install codesite.ext/author/goExample/goex
```

将一个名为 `codesite.ext/author/goExample/goex` 的 map 安装在 `$GOROOT/src/` 目录下。

通过以下方式，一次性安装，并导入到你的代码中：

```go
import goex "codesite.ext/author/goExample/goex"
```

因此该包的 URL 将用作导入路径。

在 `http://golang.org/cmd/goinstall/` 的 `go install` 文档中列出了一些广泛被使用的托管在网络代码仓库的包的导入路径

**包的初始化:**

程序的执行开始于导入包，初始化 `main` 包然后调用 `main` 函数。

一个没有导入的包将通过分配初始值给所有的包级变量和调用源码中定义的包级 `init` 函数来初始化。一个包可能有多个 `init` 函数甚至在一个源码文件中。它们的执行是无序的。这是最好的例子来测定包的值是否只依赖于相同包下的其他值或者函数。

`init` 函数是不能被调用的。

导入的包在包自身初始化前被初始化，而一个包在程序执行中只能初始化一次。

**编译并安装一个包(参见第 9.7 节):**

在 Linux/OS X 下可以用类似第 3.9 节的 `Makefile` 脚本做到这一点：

	include $(GOROOT)/src/Make.inc
	TARG=pack1
	GOFILES=\
	 	pack1.go\
	 	pack1b.go\
	include $(GOROOT)/src/Make.pkg

通过 `chmod 777 ./Makefile`确保它的可执行性。

上面脚本内的include语引入了相应的功能，将自动检测机器的架构并调用正确的编译器和链接器。

然后终端执行 make 或 `gomake` 工具：他们都会生成一个包含静态库 pack1.a 的 _obj 目录。

go install(参见第 9.7 节，从 Go1 的首选方式)同样复制 pack1.a 到本地的 $GOROOT/pkg 的目录中一个以操作系统为名的子目录下。像 `import "pack1"` 代替 `import "path to pack1"`，这样只通过名字就可以将包在程序中导入。

当第 13 章我们遇到使用测试工具进行测试的时候我们将重新回到自己的包的制作和编译这个话题。

**问题 9.1**

a）一个包能分成多个源文件么？

b）一个源文件是否能包含多个包？

**练习 9.3**

创建一个程序 `main_greetings.go` 能够和用户说 "Good Day" 或者 "Good Night"。不同的问候应该放到单独的 greetings 包中。

在同一个包中创建一个 `IsAM` 函数返回一个布尔值用来判断当前时间是 AM 还是 PM，同样创建 `IsAfternoon` 和 `IsEvening` 函数。

使用 main_greetings 作出合适的问候(提示：使用 time 包)。

**练习 9.4** 创建一个程序 `main_oddven.go` 判断前 100 个整数是不是偶数，将判断所用的函数编写在 even 包里。

**练习 9.5** 使用第 6.6 节的斐波那契程序：

1）将斐波那契功能放入自己的 fibo 包中并通过主程序调用它，存储最后输入的值在函数的全局变量。

2）扩展 fibo 包将通过调用斐波那契的时候，操作也作为一个参数。实验 "+" 和 “*”

main_fibo.go / fibonacci.go

## 9.6 为自定义包使用 godoc

godoc工具（第 3.6 节）在显示自定义包中的注释也有很好的效果：注释必须以 `//` 开始并无空行放在声明（包，类型，函数）前。godoc 会为每个文件生成一系列的网页。

例如：

- 在 [doc_examples](examples/chapter_9/doc_example) 目录下我们有第 11.7 节中的用来排序的 go 文件，文件中有一些注释（文件需要未编译）
- 命令行下进入目录下并输入命令：

	`godoc -http=:6060 -goroot="."`

（`.` 是指当前目录，-goroot 参数可以是 `/path/to/my/package1` 这样的形式指出 package1 在你源码中的位置或接受用冒号形式分隔的路径，无根目录的路径为相对于当前目录的相对路径）

- 在浏览器打开地址：http://localhost:6060

然后你会看到本地的 godoc 页面（详见第 3.6 节）从左到右一次显示出目录中的包：

doc_example:

doc_example | Packages | Commands | Specification

下面是链接到源码和所有对象时有序概述（所以是很好的浏览和查找源代码的方式），连同文件/注释：

sort 包

```go
func Float64sAreSorted

type IntArray

func IntsAreSortedfunc IsSortedfunc Sort

func (IntArray) Len

func SortFloat64s

func (IntArray) Less

func SortInts

func (IntArray) Swap

func SortStrings type Interface

func StringsAreSorted type StringArray type Float64Array

func (StringArray) Len

func (Float64Array) Len

func (StringArray) Less

func (Float64Array) Less

func (StringArray) Swap

func (Float64Array) Swap

// Other packages
import "doc_example" 
```

使用通用的接口排序:
```
func Float64sAreSorted[Top]
func Float64sAreSorted(a []float64) bool

func IntsAreSorted[Top]
func IntsAreSorted(a []int) bool

func IsSorted[Top]
func IsSorted(data Interface) bool
Test if data is sorted

func Sort[Top]
func Sort(data Interface)
General sort function

func SortInts[Top]
func SortInts(a []int)

Convenience wrappers for common cases: type IntArray[Top]
Convenience types for common cases: IntArray type IntArray []int  
```

如果你在一个团队中工作，并且源代码树被存储在网络硬盘上，就可以使用 godoc 给所有团队成员连续文档的支持。通过设置 `sync_minutes=n`，你甚至可以让它每 n 分钟自动更新您的文档！

## 9.7 使用 go install 安装自定义包

go install 是 Go 中自动包安装工具：如需要将包安装到本地它会从远端仓库下载包：检出、编译和安装一气呵成。

在包安装前的先决条件是要自动处理包自身依赖关系的安装。被依赖的包也会安装到子目录下，但是没有文档和示例：可以到网上浏览。

go install 使用了 GOPATH 变量(详见第 2.2 节)。

远端包(详见第 9.5 节)：

假设我们要安装一个有趣的包 tideland（它包含了许多帮助示例，参见 [项目主页](http://code.google.com/p/tideland-cgl)）。

因为我们需要创建目录在 Go 安装目录下，所以我们需要使用 root 或者 su 的身份执行命令。

确保 Go 环境变量已经设置在 root 用户下的 `./bashrc` 文件中。

使用命令安装：`go install tideland-cgl.googlecode.com/hg`。

可执行文件 `hg.a` 将被放到 `$GOROOT/pkg/linux_amd64/tideland-cgl.googlecode.com` 目录下，源码文件被放置在 `$GOROOT/src/tideland-cgl.googlecode.com/hg` 目录下，同样有个 `hg.a` 放置在 `_obj` 的子目录下。

现在就可以在 go 代码中使用这个包中的功能了，例如使用包名 cgl 导入：

```go
import cgl "tideland-cgl.googlecode.com/hg"
```

从 Go1 起 go install 安装 Google Code 的导入路径形式是：`"code.google.com/p/tideland-cgl"`

升级到新的版本：

更新到新版本的 Go 之后本地安装包的二进制文件将全被删除。如果你想更新，重编译、重安装所有的go安装包可以使用：`go install -a`。

go 的版本发布的很频繁，所以需要注意发布版本和包的兼容性。go1 之后都是自己编译自己了。

go install 同样可以使用 go install 编译链接并安装本地自己的包（详见第 9.8.2 节）。

更多信息可以在 [官方网站](http://golang.org/cmd/go/) 找到。

## 9.8 自定义包的目录结构、go install 和 go test

为了示范，我们创建了一个名为 `uc` 的简单包，它含有一个 `UpperCase` 函数将字符串的所有字母转换为大写。当然这并不值得创建一个自己包，同样的功能已被包含在 `strings` 包里，但是同样的技术也可以应用在更复杂的包中。

### 9.8.1 自定义包的目录结构

下面的结构给了你一个好的示范(`uc` 代表通用包名, 名字为粗体的代表目录，斜体代表可执行文件):

```go
/home/user/goprograms
	ucmain.go	(uc包主程序)
	Makefile (ucmain的makefile)
	ucmain
	src/uc	 (包含uc包的go源码)
		uc.go
	 	uc_test.go
	 	Makefile (包的makefile)
	 	uc.a
	 	_obj
			uc.a
		_test
			uc.a
	bin		(包含最终的执行文件)
		ucmain
	pkg/linux_amd64
		uc.a	(包的目标文件)
```

将你的项目放在 `goprograms` 目录下(你可以创建一个环境变量 GOPATH，详见第 2.2/3 章节：在 `.profile` 和 `.bashrc` 文件中添加 `export GOPATH=/home/user/goprograms`)，而你的项目将作为 `src` 的子目录。`uc` 包中的功能在 `uc.go` 中实现。

示例 9.6 [uc.go](examples/chapter_9/uc.go)：

```go
package uc
import "strings"

func UpperCase(str string) string {
	return strings.ToUpper(str)
}
```

包通常附带一个或多个测试文件，在这我们创建了一个 uc_test.go 文件，如第 9.8 节所述。

示例 9.7 [test.go](examples/chapter_9/test.go)

```go
package uc
import "testing"

type ucTest struct {
	in, out string
}

var ucTests = []ucTest {
	ucTest{"abc", "ABC"},
	ucTest{"cvo-az", "CVO-AZ"},
	ucTest{"Antwerp", "ANTWERP"},
}

func TestUC(t *testing.T) {
	for _, ut := range ucTests {
		uc := UpperCase(ut.in)
		if uc != ut.out {
			t.Errorf("UpperCase(%s) = %s, must be %s", ut.in, uc,
			ut.out)
		}
	}
}
```

通过指令编译并安装包到本地：`go install uc`, 这会将 `uc.a` 复制到 `pkg/linux_amd64` 下面。

另外，使用 make ，通过以下内容创建一个包的 `Makefile` 在 `src/uc` 目录下:

```
include $(GOROOT)/src/Make.inc

TARG=uc
GOFILES=\
		uc.go\

include $(GOROOT)/src/Make.pkg
```

在该目录下的命令行调用: gomake

这将创建一个 _obj 目录并将包编译生成的存档 uc.a 放在该目录下。

这个包可以通过 go test 测试。

创建一个 uc.a 的测试文件在目录下，输出为 PASS 时测试通过。

在第 13.8 节我们将给出另外一个测试例子并进行深入研究。

备注：有可能你当前的用户不具有足够的资格使用 go　install(没有权限)。这种情况下，选择 root 用户 su。确保 Go 环境变量和 Go 源码路径也设置给 su，同样也适用你的普通用户(详见第 2.3 节)。

接下来我们创建主程序 ucmain.go:

示例 9.8 [ucmain.go](/examples/chapter_9/ucmain.go)：

```go
package main
import (
	"./src/uc"
	"fmt"
)

func main() {
	str1 := "USING package uc!"
	fmt.Println(uc.UpperCase(str1))
}
```

然后在这个目录下输入 `go install`。

另外复制 uc.a 到 /home/user/goprograms 目录并创建一个 Makefile 并写入文本：

```
include $(GOROOT)/src/Make.inc
TARG=ucmain
GOFILES=\
	ucmain.go\

include $(GOROOT)/src/Make.cmd
```

执行 gomake 编译 `ucmain.go` 生成可执行文件ucmain

运行 `./ucmain` 显示: `USING PACKAGE UC!`。

### 9.8.2 本地安装包

本地包在用户目录下，使用给出的目录结构，以下命令用来从源码安装本地包：

	go install /home/user/goprograms/src/uc # 编译安装uc
	cd /home/user/goprograms/uc
	go install ./uc 	# 编译安装uc（和之前的指令一样）
	cd ..
	go install .	# 编译安装ucmain

安装到 `$GOPATH` 下：

如果我们想安装的包在系统上的其他 Go 程序中被使用，它一定要安装到 `$GOPATH` 下。
这样做，在 .profile 和 .bashrc 中设置 `export GOPATH=/home/user/goprograms`。

然后执行 go install uc 将会复制包存档到 `$GOPATH/pkg/LINUX_AMD64/uc`。

现在，uc 包可以通过 `import "uc"` 在任何 Go 程序中被引用。

### 9.8.3 依赖系统的代码

在不同的操作系统上运行的程序以不同的代码实现是非常少见的：绝大多数情况下语言和标准库解决了大部分的可移植性问题。

你有一个很好的理由去写平台特定的代码，例如汇编语言。这种情况下，按照下面的约定是合理的：

	prog1.go
	prog1_linux.go
	prog1_darwin.go
	prog1_windows.go

prog1.go 定义了不同操作系统通用的接口，并将系统特定的代码写到 prog1_os.go 中。
对于 Go 工具你可以指定 `prog1_$GOOS.go` 或 `prog1_$GOARCH.go`
或在平台 Makefile 中：`prog1_$(GOOS).go\` 或 `prog1_$(GOARCH).go\`。

## 9.9 通过 Git 打包和安装

### 9.9.1 安装到 GitHub

以上的方式对于本地包来说是可以的，但是我们如何打包代码到开发者圈子呢？那么我们需要一个云端的源码的版本控制系统，比如著名的 Git。

在 Linux 和 OS X 的机器上 Git 是默认安装的，在 Windows 上你必须先自行安装，参见 [GitHub 帮助页面](http://help.github.com/win-set-up-git/)。

这里将通过为第 9.8 节中的 uc 包创建一个 git 仓库作为演示

进入到 uc 包目录下并创建一个 Git 仓库在里面: `git init`。

信息提示: `Initialized empty git repository in $PWD/uc`。

每一个 Git 项目都需要一个对包进行描述的 README.md 文件，所以需要打开你的文本编辑器（gedit、notepad 或 LiteIde）并添加一些说明进去。

- 添加所有文件到仓库：`git add README.md uc.go uc_test.go Makefile`。
- 标记为第一个版本：`git commit -m "initial rivision"`。

现在必须登录 [GitHub 网站](https://github.com)。 

如果您还没有账号，可以去注册一个开源项目的免费帐号。输入正确的帐号密码和有效的邮箱地址并进一步创建用户。然后你将获得一个 Git 命令的列表。本地仓库的操作命令已经完成。一个优秀的系统在你遇到任何问题的时候将 [引导你](http://help.github.com/)。

在云端创建一个新的 uc 仓库;发布的指令为(`NNNN` 替代用户名):

```
git remote add origin git@github.com:NNNN/uc.git  
git push -u origin master
```

操作完成后检查 GitHub 上的包页面: `http://github.com/NNNN/uc`。

### 9.9.2 从 GitHub 安装

如果有人想安装您的远端项目到本地机器，打开终端并执行（NNNN 是你在 GitHub 上的用户名）：`go get github.com/NNNN/uc`。

这样现在这台机器上的其他 Go 应用程序也可以通过导入路径：`"github.com/NNNN/uc"` 代替 `"./uc/uc"` 来使用。

也可以将其缩写为：`import uc "github.com/NNNN/uc"`。

然后修改 Makefile: 将 `TARG=uc` 替换为 `TARG=github.com/NNNN/uc`。

Gomake（和 go install）将通过 `$GOPATH` 下的本地版本进行工作。

网站和版本控制系统的其他的选择(括号中为网站所使用的版本控制系统)：

- BitBucket(hg/Git)
- GitHub(Git)
- Google Code(hg/Git/svn)
- Launchpad(bzr)

版本控制系统可以选择你熟悉的或者本地使用的代码版本控制。Go 核心代码的仓库是使用 Mercurial(hg) 来控制的，所以它是一个最可能保证你可以得到开发者项目中最好的软件。Git 也很出名，同样也适用。如果你从未使用过版本控制，这些网站有一些很好的帮助并且你可以通过在谷歌搜索 "{name} tutorial"，(name为你想要使用的版本控制系统),得到许多很好的教程。

## 9.10 Go 的外部包和项目

现在我们知道如何使用 Go 以及它的标准库了，但是 Go 的生态要比这大的多。当着手自己的 Go 项目时，最好先查找下是否有些存在的第三方的包或者项目能不能使用。大多数可以通过 go install 来进行安装。

[Go Walker](https://gowalker.org) 支持根据包名在海量数据中查询。

目前已经有许多非常好的外部库，如：

- MySQL(GoMySQL), PostgreSQL(go-pgsql), MongoDB (mgo, gomongo), CouchDB (couch-go), ODBC (godbcl), Redis (redis.go) and SQLite3 (gosqlite) database drivers
- SDL bindings
- Google's Protocal Buffers(goprotobuf)
- XML-RPC(go-xmlrpc)
- Twitter(twitterstream)
- OAuth libraries(GoAuth)

## 9.11 在 Go 程序中使用外部库

（本节我们将创建一个 Web 应用和它的 Google App Engine 版本,在第 19 和 21 章分别说明，当你阅读到这些章节时可以再回到这个例子。)

当开始一个新项目或增加新的功能到现有的项目，你可以通过在应用程序中使用已经存在的库来节省开发时间。为了做到这一点，你必须理解库的 API（应用编程接口），那就是：库中有哪些方法可以调用，如何调用。你可能没有这个库的源代码，但作者肯定有记载的 API 以及详细介绍了如何使用它。

作为一个例子，我们将使用谷歌的 API 的 urlshortener 编写一个小程序：你可以尝试一下在 http://goo.gl/ 输入一个像 "http://www.destandaard.be" 这样的URL，你会看到一个像 "http://goo.gl/O9SUO" 这样更短的 URL 返回，也就是说，在 Twitter 之类的服务中这是非常容易嵌入的。谷歌 urlshortener 服务的文档可以在 "http://code.google.com/apis/urlshortener/" 找到。(第 19 章，我们将开发自己版本的 urlshortener)。

谷歌将这项技术提供给其他开发者，作为 API 我们可以在我们自己的应用程序中调用（释放到指定的限制）。他们也生成了一个 Go 语言客户端库使其变得更容易。

备注：谷歌让通过使用 Google API Go 客户端服务的开发者生活变得更简单，Go 客户端程序自动生成于 Google 库的 JSON 描述。更多详情在 [项目页面](http://code.google.com/p/google-api-go-client/) 查看。

下载并安装 Go 客户端库:
将通过 go install 实现。但是首先要验证环境变量中是否含有 `GOPATH` 变量，因为外部源码将被下载到 `$GOPATH/src` 目录下并被安装到 `$GOPATH/PKG/"machine_arch"/` 目录下。

我们将通过在终端调用以下命令来安装 API:

```go
go install google.golang.org/api/urlshortener/v1
```

go install 将下载源码，编译并安装包

使用 `urlshortener` 服务的 web 程序:
现在我们可以通过导入并赋予别名来使用已安装的包：

```go
import  "google.golang.org/api/urlshortener/v1"
```

现在我们写一个 Web 应用(参见第 15 章 4-8 节)通过表单实现短地址和长地址的相互转换。我们将使用 `template` 包并写三个处理函数：`root` 函数通过执行表单模板来展示表单。`short` 函数将长地址转换为短地址，`long` 函数逆向转换。

要调用 `urlshortener` 接口必须先通过 http 包中的默认客户端创建一个服务实例 `urlshortenerSvc`：  

```go
urlshortenerSvc, _ := urlshortener.New(http.DefaultClient)
```

我们通过调用服务中的 `Url.Insert` 中的 `Do` 方法传入包含长地址的 `Url` 数据结构从而获取短地址：

```go
url, _ := urlshortenerSvc.Url.Insert(&urlshortener.Url{LongUrl: longUrl}).Do()
```

返回 `url` 的 `Id` 便是我们需要的短地址。

我们通过调用服务中的 `Url.Get` 中的 `Do` 方法传入包含短地址的Url数据结构从而获取长地址：

```go
url, error := urlshortenerSvc.Url.Get(shwortUrl).Do()
```

返回的长地址便是转换前的原始地址。

示例 9.9	[urlshortener.go](examples/chapter_9/use_urlshortener.go)

```go
package main

import (
	 "fmt"
	 "net/http"
	 "text/template"

	 "google.golang.org/api/urlshortener/v1"
)
func main() {
	 http.HandleFunc("/", root)
	 http.HandleFunc("/short", short)
	 http.HandleFunc("/long", long)

	 http.ListenAndServe("localhost:8080", nil)
}
// the template used to show the forms and the results web page to the user
var rootHtmlTmpl = template.Must(template.New("rootHtml").Parse(`
<html><body>
<h1>URL SHORTENER</h1>
{{if .}}{{.}}<br /><br />{{end}}
<form action="/short" type="POST">
Shorten this: <input type="text" name="longUrl" />
<input type="submit" value="Give me the short URL" />
</form>
<br />
<form action="/long" type="POST">
Expand this: http://goo.gl/<input type="text" name="shortUrl" />
<input type="submit" value="Give me the long URL" />
</form>
</body></html>
`))
func root(w http.ResponseWriter, r *http.Request) {
	rootHtmlTmpl.Execute(w, nil)
}
func short(w http.ResponseWriter, r *http.Request) {
	 longUrl := r.FormValue("longUrl")
	 urlshortenerSvc, _ := urlshortener.New(http.DefaultClient)
	 url, _ := urlshortenerSvc.Url.Insert(&urlshortener.Url{LongUrl:
	 longUrl,}).Do()
	 rootHtmlTmpl.Execute(w, fmt.Sprintf("Shortened version of %s is : %s",
	 longUrl, url.Id))
}

func long(w http.ResponseWriter, r *http.Request) {
	 shortUrl := "http://goo.gl/" + r.FormValue("shortUrl")
	 urlshortenerSvc, _ := urlshortener.New(http.DefaultClient)
	 url, err := urlshortenerSvc.Url.Get(shortUrl).Do()
	 if err != nil {
		 fmt.Println("error: %v", err)
		 return

	 }
	 rootHtmlTmpl.Execute(w, fmt.Sprintf("Longer version of %s is : %s",
	 shortUrl, url.LongUrl))
}
```

执行这段代码：

```go
go run urlshortener.go
```

通过浏览 `http://localhost:8080/` 的页面来测试。

为了代码的简洁我们并没有检测返回的错误状态，但是在真实的生产环境的应用中一定要做检测。

将应用放入 Google App Engine，我们只需要在之前的代码中作出如下改变：

	package main -> package urlshort
	func main() -> func init()

创建一个和包同名的目录 `urlshort`，并将以下两个安装目录复制到这个目录：

```go
google.golang.org/api/urlshortener
google.golang.org/api/googleapi
```

此外还要配置下配置文件 `app.yaml`，内容如下：

```yaml
application: urlshort
version: 0-1-test
runtime: go
api_version: 3
handlers:
- url: /.*
script: _go_app
```

现在你可以去到你的项目目录并在终端运行：`dev_appserver.py urlshort`

在浏览器打开你的 Web应用：http://localhost:8080。