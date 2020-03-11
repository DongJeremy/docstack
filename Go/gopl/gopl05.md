# 第五章　函数

函数可以让我们将一个语句序列打包为一个单元，然后可以从程序中其它地方多次调用。函数的机制可以让我们将一个大的工作分解为小的任务，这样的小任务可以让不同程序员在不同时间、不同地方独立完成。一个函数同时对用户隐藏了其实现细节。由于这些因素，对于任何编程语言来说，函数都是一个至关重要的部分。

我们已经见过许多函数了。现在，让我们多花一点时间来彻底地讨论函数特性。本章的运行示例是一个网络蜘蛛，也就是web搜索引擎中负责抓取网页部分的组件，它们根据抓取网页中的链接继续抓取链接指向的页面。一个网络蜘蛛的例子给我们足够的机会去探索递归函数、匿名函数、错误处理和函数其它的很多特性。

## 5.1. 函数声明

函数声明包括函数名、形式参数列表、返回值列表（可省略）以及函数体。

```go
func name(parameter-list) (result-list) {
	body
}
```

形式参数列表描述了函数的参数名以及参数类型。这些参数作为局部变量，其值由参数调用者提供。返回值列表描述了函数返回值的变量名以及类型。如果函数返回一个无名变量或者没有返回值，返回值列表的括号是可以省略的。如果一个函数声明不包括返回值列表，那么函数体执行完毕后，不会返回任何值。在`hypot`函数中：

```go
func hypot(x, y float64) float64 {
	return math.Sqrt(x*x + y*y)
}
fmt.Println(hypot(3,4)) // "5"
```

`x`和`y`是形参名，`3`和`4`是调用时的传入的实参，函数返回了一个`float64`类型的值。 返回值也可以像形式参数一样被命名。在这种情况下，每个返回值被声明成一个局部变量，并根据该返回值的类型，将其初始化为该类型的零值。 如果一个函数在声明时，包含返回值列表，该函数必须以 `return`语句结尾，除非函数明显无法运行到结尾处。例如函数在结尾时调用了`panic`异常或函数中存在无限循环。

正如`hypot`一样，如果一组形参或返回值有相同的类型，我们不必为每个形参都写出参数类型。下面2个声明是等价的：

```go
func f(i, j, k int, s, t string)                 { /* ... */ }
func f(i int, j int, k int,  s string, t string) { /* ... */ }
```

下面，我们给出4种方法声明拥有2个`int`型参数和1个`int`型返回值的函数`.blank identifier`(译者注：即下文的`_`符号)可以强调某个参数未被使用。

```go
func add(x int, y int) int   {return x + y}
func sub(x, y int) (z int)   { z = x - y; return}
func first(x int, _ int) int { return x }
func zero(int, int) int      { return 0 }

fmt.Printf("%T\n", add)   // "func(int, int) int"
fmt.Printf("%T\n", sub)   // "func(int, int) int"
fmt.Printf("%T\n", first) // "func(int, int) int"
fmt.Printf("%T\n", zero)  // "func(int, int) int"
```

函数的类型被称为函数的**签名**。如果两个函数*形式参数列表*和*返回值列表中的变量类型*一一对应，那么这两个函数被认为有相同的类型或签名。形参和返回值的变量名不影响函数签名，也不影响它们是否可以以省略参数类型的形式表示。

每一次函数调用都必须按照声明顺序为所有参数提供实参（参数值）。在函数调用时，Go语言没有默认参数值，也没有任何方法可以通过参数名指定形参，因此形参和返回值的变量名对于函数调用者而言没有意义。

在函数体中，函数的形参作为局部变量，被初始化为调用者提供的值。函数的形参和有名返回值作为函数最外层的局部变量，被存储在相同的词法块中。

实参通过<u>值的方式</u>传递，因此函数的形参是实参的拷贝。对形参进行修改不会影响实参。但是，如果实参包括引用类型，如指针，slice(切片)、map、function、channel等类型，实参可能会由于函数的间接引用被修改。

你可能会偶尔遇到没有函数体的函数声明，这表示该函数不是以Go实现的。这样的声明定义了函数签名。

```go
package math

func Sin(x float64) float //implemented in assembly language
```

## 5.2. 递归

函数可以是递归的，这意味着函数可以直接或间接的调用自身。对许多问题而言，递归是一种强有力的技术，例如处理递归的数据结构。在4.4节，我们通过遍历二叉树来实现简单的插入排序，在本章节，我们再次使用它来处理HTML文件。

下文的示例代码使用了非标准包 `golang.org/x/net/html` ，解析HTML。`golang.org/x/...` 目录下存储了一些由Go团队设计、维护，对网络编程、国际化文件处理、移动平台、图像处理、加密解密、开发者工具提供支持的扩展包。未将这些扩展包加入到标准库原因有二，一是部分包仍在开发中，二是对大多数Go语言的开发者而言，扩展包提供的功能很少被使用。

例子中调用`golang.org/x/net/html`的部分`api`如下所示。`html.Parse`函数读入一组`bytes`解析后，返回`html.Node`类型的HTML页面树状结构根节点。HTML拥有很多类型的结点如`text`（文本）、`commnets`（注释）类型，在下面的例子中，我们只关注`<name key='value'>`形式的结点。

*golang.org/x/net/html*

```go
package html

type Node struct {
	Type                    NodeType
	Data                    string
	Attr                    []Attribute
	FirstChild, NextSibling *Node
}

type NodeType int32

const (
	ErrorNode NodeType = iota
	TextNode
	DocumentNode
	ElementNode
	CommentNode
	DoctypeNode
)

type Attribute struct {
	Key, Val string
}

func Parse(r io.Reader) (*Node, error)
```

`main`函数解析HTML标准输入，通过递归函数`visit`获得`links`（链接），并打印出这些`links`：

*gopl.io/ch5/findlinks1*

```go
// Findlinks1 prints the links in an HTML document read from standard input.
package main

import (
	"fmt"
	"os"

	"golang.org/x/net/html"
)

func main() {
	doc, err := html.Parse(os.Stdin)
	if err != nil {
		fmt.Fprintf(os.Stderr, "findlinks1: %v\n", err)
		os.Exit(1)
	}
	for _, link := range visit(nil, doc) {
		fmt.Println(link)
	}
}
```

`visit`函数遍历HTML的节点树，从每一个`anchor`元素的`href`属性获得`link`,将这些`links`存入字符串数组中，并返回这个字符串数组。

```go
// visit appends to links each link found in n and returns the result.
func visit(links []string, n *html.Node) []string {
	if n.Type == html.ElementNode && n.Data == "a" {
		for _, a := range n.Attr {
			if a.Key == "href" {
				links = append(links, a.Val)
			}
		}
	}
	for c := n.FirstChild; c != nil; c = c.NextSibling {
		links = visit(links, c)
	}
	return links
}
```

为了遍历结点`n`的所有后代结点，每次遇到`n`的孩子结点时，`visit`递归的调用自身。这些孩子结点存放在`FirstChild`链表中。

让我们以Go的主页（`golang.org`）作为目标，运行`findlinks`。我们以`fetch`（1.5章）的输出作为`findlinks`的输入。下面的输出做了简化处理。

```go
$ go build gopl.io/ch1/fetch
$ go build gopl.io/ch5/findlinks1
$ ./fetch https://golang.org | ./findlinks1
#
/doc/
/pkg/
/help/
/blog/
http://play.golang.org/
//tour.golang.org/
https://golang.org/dl/
//blog.golang.org/
/LICENSE
/doc/tos.html
http://www.google.com/intl/en/policies/privacy/
```

注意在页面中出现的链接格式，在之后我们会介绍如何将这些链接，根据根路径（ [https://golang.org](https://golang.org/) ）生成可以直接访问的`url`。

在函数`outline`中，我们通过递归的方式遍历整个HTML结点树，并输出树的结构。在`outline`内部，每遇到一个HTML元素标签，就将其入栈，并输出。

*gopl.io/ch5/outline*

```go
func main() {
	doc, err := html.Parse(os.Stdin)
	if err != nil {
		fmt.Fprintf(os.Stderr, "outline: %v\n", err)
		os.Exit(1)
	}
	outline(nil, doc)
}
func outline(stack []string, n *html.Node) {
	if n.Type == html.ElementNode {
		stack = append(stack, n.Data) // push tag
		fmt.Println(stack)
	}
	for c := n.FirstChild; c != nil; c = c.NextSibling {
		outline(stack, c)
	}
}
```

有一点值得注意：`outline`有入栈操作，但没有相对应的出栈操作。当`outline`调用自身时，被调用者接收的是`stack`的拷贝。被调用者对`stack`的元素追加操作，修改的是`stack`的拷贝，其可能会修改`slice`底层的数组甚至是申请一块新的内存空间进行扩容；但这个过程并不会修改调用方的`stack`。因此当函数返回时，调用方的`stack`与其调用自身之前完全一致。

下面是 [https://golang.org](https://golang.org/) 页面的简要结构:

```go
$ go build gopl.io/ch5/outline
$ ./fetch https://golang.org | ./outline
[html]
[html head]
[html head meta]
[html head title]
[html head link]
[html body]
[html body div]
[html body div]
[html body div div]
[html body div div form]
[html body div div form div]
[html body div div form div a]
...
```

正如你在上面实验中所见，大部分HTML页面只需几层递归就能被处理，但仍然有些页面需要深层次的递归。

大部分编程语言使用固定大小的函数调用栈，常见的大小从64KB到2MB不等。固定大小栈会限制递归的深度，当你用递归处理大量数据时，需要避免栈溢出；除此之外，还会导致安全性问题。与此相反，Go语言使用可变栈，栈的大小按需增加（初始时很小）。这使得我们使用递归时不必考虑溢出和安全问题。

**练习 5.1：** 修改`findlinks`代码中遍历`n.FirstChild`链表的部分，将循环调用`visit`，改成递归调用。

```go
// main.go
package main

import (
	"fmt"
	"os"

	"golang.org/x/net/html"
)

func main() {
	doc, err := html.Parse(os.Stdin)
	if err != nil {
		fmt.Fprintf(os.Stderr, "findlinks1: %v\n", err)
		os.Exit(1)
	}
	for _, link := range visit(nil, doc) {
		fmt.Println(link)
	}
}

// visit appends to links each link found in n and returns the result.
func visit(links []string, n *html.Node) []string {
	if n.Type == html.ElementNode && n.Data == "a" {
		for _, a := range n.Attr {
			if a.Key == "href" {
				fmt.Println(a.Val)
				links = append(links, a.Val)
			}
		}
	}
	if n.FirstChild != nil {
		links = visit(links, n.FirstChild)
	}
	if n.NextSibling != nil {
		links = visit(links, n.NextSibling)
	}
	return links
}

/*
//!+html
package html

type Node struct {
	Type                    NodeType
	Data                    string
	Attr                    []Attribute
	FirstChild, NextSibling *Node
}

type NodeType int32

const (
	ErrorNode NodeType = iota
	TextNode
	DocumentNode
	ElementNode
	CommentNode
	DoctypeNode
)

type Attribute struct {
	Key, Val string
}

func Parse(r io.Reader) (*Node, error)
//!-html
*/
```

**练习 5.2：** 编写函数，记录在HTML树中出现的同名元素的次数。

```go
// main.go
package main

import (
	"fmt"
	"io"
	"log"
	"os"

	"golang.org/x/net/html"
)

func tagFreq(r io.Reader) (map[string]int, error) {
	freq := make(map[string]int, 0)
	z := html.NewTokenizer(os.Stdin)
	var err error
	for {
		type_ := z.Next()
		if type_ == html.ErrorToken {
			break
		}
		name, _ := z.TagName()
		if len(name) > 0 {
			freq[string(name)]++
		}
	}
	if err != io.EOF {
		return freq, err
	}
	return freq, nil
}

func main() {
	freq, err := tagFreq(os.Stdin)
	if err != nil {
		log.Fatal(err)
	}
	for tag, count := range freq {
		fmt.Printf("%4d %s\n", count, tag)
	}
}

```

**练习 5.3：** 编写函数输出所有text结点的内容。注意不要访问``和``元素，因为这些元素对浏览者是不可见的。

```go
// main.go
package main

import (
	"fmt"
	"io"
	"log"
	"os"
	"strings"

	"golang.org/x/net/html"
)

func printTagText(r io.Reader, w io.Writer) error {
	z := html.NewTokenizer(os.Stdin)
	var err error
	stack := make([]string, 20)
Tokenize:
	for {
		switch z.Next() {
		case html.ErrorToken:
			break Tokenize
		case html.StartTagToken:
			b, _ := z.TagName()
			stack = append(stack, string(b))
		case html.TextToken:
			cur := stack[len(stack)-1]
			if cur == "script" || cur == "style" {
				continue
			}
			text := z.Text()
			if len(strings.TrimSpace(string(text))) == 0 {
				continue
			}
			w.Write([]byte(fmt.Sprintf("<%s>", cur)))
			w.Write(text)
			if text[len(text)-1] != '\n' {
				io.WriteString(w, "\n")
			}
		case html.EndTagToken:
			stack = stack[:len(stack)-1]
		}
	}
	if err != io.EOF {
		return err
	}
	return nil
}

func main() {
	err := printTagText(os.Stdin, os.Stdout)
	if err != nil {
		log.Fatal(err)
	}
}
```

**练习 5.4：** 扩展`visit`函数，使其能够处理其他类型的结点，如`images`、`scripts`和`style sheets`。

```go
// main.go
package main

import (
	"fmt"
	"os"

	"golang.org/x/net/html"
)

func main() {
	doc, err := html.Parse(os.Stdin)
	if err != nil {
		fmt.Fprintf(os.Stderr, "findlinks1: %v\n", err)
		os.Exit(1)
	}
	for _, link := range visit(nil, doc) {
		fmt.Println(link)
	}
}

// visit appends to links each link found in n and returns the result.
func visit(links []string, n *html.Node) []string {
	if n.Type == html.ElementNode && n.Data == "a" {
		for _, a := range n.Attr {
			if a.Key == "href" {
				links = append(links, a.Val)
			}
		}
	}
	if n.Type == html.ElementNode && (n.Data == "img" || n.Data == "script") {
		for _, a := range n.Attr {
			if a.Key == "src" {
				links = append(links, a.Val)
			}
		}
	}
	if n.FirstChild != nil {
		links = visit(links, n.FirstChild)
	}
	if n.NextSibling != nil {
		links = visit(links, n.NextSibling)
	}
	return links
}

/*
//!+html
package html

type Node struct {
	Type                    NodeType
	Data                    string
	Attr                    []Attribute
	FirstChild, NextSibling *Node
}

type NodeType int32

const (
	ErrorNode NodeType = iota
	TextNode
	DocumentNode
	ElementNode
	CommentNode
	DoctypeNode
)

type Attribute struct {
	Key, Val string
}

func Parse(r io.Reader) (*Node, error)
//!-html
*/
```

## 5.3. 多返回值

在Go中，一个函数可以返回多个值。我们已经在之前例子中看到，许多标准库中的函数返回2个值，一个是期望得到的返回值，另一个是函数出错时的错误信息。下面的例子会展示如何编写多返回值的函数。

下面的程序是`findlinks`的改进版本。修改后的`findlinks`可以自己发起HTTP请求，这样我们就不必再运行`fetch`。因为HTTP请求和解析操作可能会失败，因此`findlinks`声明了2个返回值：*链接列表和错误信息*。一般而言，HTML的解析器可以处理HTML页面的错误结点，构造出HTML页面结构，所以解析HTML很少失败。这意味着如果`findlinks`函数失败了，很可能是由于I/O的错误导致的。

*gopl.io/ch5/findlinks2*

```go
func main() {
	for _, url := range os.Args[1:] {
		links, err := findLinks(url)
		if err != nil {
			fmt.Fprintf(os.Stderr, "findlinks2: %v\n", err)
			continue
		}
		for _, link := range links {
			fmt.Println(link)
		}
	}
}

// findLinks performs an HTTP GET request for url, parses the
// response as HTML, and extracts and returns the links.
func findLinks(url string) ([]string, error) {
	resp, err := http.Get(url)
	if err != nil {
		return nil, err
	}
	if resp.StatusCode != http.StatusOK {
		resp.Body.Close()
		return nil, fmt.Errorf("getting %s: %s", url, resp.Status)
	}
	doc, err := html.Parse(resp.Body)
	resp.Body.Close()
	if err != nil {
		return nil, fmt.Errorf("parsing %s as HTML: %v", url, err)
	}
	return visit(nil, doc), nil
}
```

在`findlinks`中，有4处`return`语句，每一处`return`都返回了一组值。前三处`return`，将`http`和`html`包中的错误信息传递给`findlinks`的调用者。第一处`return`直接返回错误信息，其他两处通过`fmt.Errorf`（§7.8）输出详细的错误信息。如果`findlinks`成功结束，最后的`return`语句将一组解析获得的连接返回给用户。

在`findlinks`中，我们必须确保`resp.Body`被关闭，释放网络资源。虽然Go的垃圾回收机制会回收不被使用的内存，但是这不包括操作系统层面的资源，比如打开的文件、网络连接。因此我们必须显式的释放这些资源。

调用多返回值函数时，返回给调用者的是一组值，调用者必须显式的将这些值分配给变量:

```go
links, err := findLinks(url)
```

如果某个值不被使用，可以将其分配给`blank identifier`:

```go
links, _ := findLinks(url) // errors ignored
```

一个函数内部可以将另一个有多返回值的函数调用作为返回值，下面的例子展示了与`findLinks`有相同功能的函数，两者的区别在于下面的例子先输出参数：

```go
func findLinksLog(url string) ([]string, error) {
	log.Printf("findLinks %s", url)
	return findLinks(url)
}
```

当你调用接受多参数的函数时，可以将一个返回多参数的函数调用作为该函数的参数。虽然这很少出现在实际生产代码中，但这个特性在`debug`时很方便，我们只需要一条语句就可以输出所有的返回值。下面的代码是等价的：

```go
log.Println(findLinks(url))
links, err := findLinks(url)
log.Println(links, err)
```

准确的变量名可以传达函数返回值的含义。尤其在返回值的类型都相同时，就像下面这样：

```go
func Size(rect image.Rectangle) (width, height int)
func Split(path string) (dir, file string)
func HourMinSec(t time.Time) (hour, minute, second int)
```

虽然良好的命名很重要，但你也不必为每一个返回值都取一个适当的名字。比如，按照惯例，函数的最后一个`bool`类型的返回值表示函数是否运行成功，`error`类型的返回值代表函数的错误信息，对于这些类似的惯例，我们不必思考合适的命名，它们都无需解释。

如果一个函数所有的返回值都有显式的变量名，那么该函数的`return`语句可以省略操作数。这称之为`bare return`。

```go
// CountWordsAndImages does an HTTP GET request for the HTML
// document url and returns the number of words and images in it.
func CountWordsAndImages(url string) (words, images int, err error) {
	resp, err := http.Get(url)
	if err != nil {
		return
	}
	doc, err := html.Parse(resp.Body)
	resp.Body.Close()
	if err != nil {
		err = fmt.Errorf("parsing HTML: %s", err)
		return
	}
	words, images = countWordsAndImages(doc)
	return
}
func countWordsAndImages(n *html.Node) (words, images int) { /* ... */ }
```

按照返回值列表的次序，返回所有的返回值，在上面的例子中，每一个`return`语句等价于：

```go
return words, images, err
```

当一个函数有多处`return`语句以及许多返回值时，`bare return` 可以减少代码的重复，但是使得代码难以被理解。举个例子，如果你没有仔细的审查代码，很难发现前2处`return`等价于 `return 0,0,err`（Go会将返回值 `words`和`images`在函数体的开始处，根据它们的类型，将其初始化为0），最后一处`return`等价于 `return words, image, nil`。基于以上原因，不宜过度使用`bare return`。

**练习 5.5：** 实现`countWordsAndImages`。（参考练习4.9如何分词）

```go
// ex5.5 counts the number of words and images at a url.
package main

import (
	"bufio"
	"fmt"
	"log"
	"net/http"
	"os"
	"strings"

	"golang.org/x/net/html"
)

// CountWordsAndImages does an HTTP GET request for the HTML
// document url and returns the number of words and images in it.
func CountWordsAndImages(url string) (words, images int, err error) {
	resp, err := http.Get(url)
	if err != nil {
		return
	}
	doc, err := html.Parse(resp.Body)
	resp.Body.Close()
	if err != nil {
		err = fmt.Errorf("parsing HTML: %s", err)
		return
	}
	words, images = countWordsAndImages(doc)
	return
}

func countWordsAndImages(n *html.Node) (words, images int) {
	u := make([]*html.Node, 0) // unvisited
	u = append(u, n)
	for len(u) > 0 {
		n = u[len(u)-1]
		u = u[:len(u)-1]
		switch n.Type {
		case html.TextNode:
			words += wordCount(n.Data)
		case html.ElementNode:
			if n.Data == "img" {
				images++
			}
		}
		for c := n.FirstChild; c != nil; c = c.NextSibling {
			u = append(u, c)
		}
	}
	return
}

func wordCount(s string) int {
	n := 0
	scan := bufio.NewScanner(strings.NewReader(s))
	scan.Split(bufio.ScanWords)
	for scan.Scan() {
		n++
	}
	return n
}

func main() {
	if len(os.Args) != 2 {
		fmt.Fprintln(os.Stderr, "usage: PROG URL")
	}
	url := os.Args[1]
	words, images, err := CountWordsAndImages(url)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("words: %d\nimages: %d\n", words, images)
}
```

**练习 5.6：** 修改`gopl.io/ch3/surface`（§3.2）中的`corner`函数，将返回值命名，并使用`bare return`。

```go
package main

import (
	"fmt"
	"math"
)

const (
	width, height = 600, 320            // canvas size in pixels
	cells         = 100                 // number of grid cells
	xyrange       = 30.0                // axis ranges (-xyrange..+xyrange)
	xyscale       = width / 2 / xyrange // pixels per x or y unit
	zscale        = height * 0.4        // pixels per z unit
	angle         = math.Pi / 6         // angle of x, y axes (=30°)
)

var sin30, cos30 = math.Sin(angle), math.Cos(angle) // sin(30°), cos(30°)

func main() {
	fmt.Printf("<svg xmlns='http://www.w3.org/2000/svg' "+
		"style='stroke: grey; fill: white; stroke-width: 0.7' "+
		"width='%d' height='%d'>", width, height)
	for i := 0; i < cells; i++ {
		for j := 0; j < cells; j++ {
			ax, ay := corner(i+1, j)
			bx, by := corner(i, j)
			cx, cy := corner(i, j+1)
			dx, dy := corner(i+1, j+1)
			fmt.Printf("<polygon points='%g,%g %g,%g %g,%g %g,%g'/>\n",
				ax, ay, bx, by, cx, cy, dx, dy)
		}
	}
	fmt.Println("</svg>")
}

func corner(i, j int) (sx float64, sy float64) {
	// Find point (x,y) at corner of cell (i,j).
	x := xyrange * (float64(i)/cells - 0.5)
	y := xyrange * (float64(j)/cells - 0.5)

	// Compute surface height z.
	z := f(x, y)

	// Project (x,y,z) isometrically onto 2-D SVG canvas (sx,sy).
	sx = width/2 + (x-y)*cos30*xyscale
	sy = height/2 + (x+y)*sin30*xyscale - z*zscale
	return
}

func f(x, y float64) float64 {
	r := math.Hypot(x, y) // distance from (0,0)
	return math.Sin(r) / r
}
```

## 5.4. 错误

在Go中有一部分函数总是能成功的运行。比如`strings.Contains`和`strconv.FormatBool`函数，对各种可能的输入都做了良好的处理，使得运行时几乎不会失败，除非遇到灾难性的、不可预料的情况，比如运行时的内存溢出。导致这种错误的原因很复杂，难以处理，从错误中恢复的可能性也很低。

还有一部分函数只要输入的参数满足一定条件，也能保证运行成功。比如`time.Date`函数，该函数将年月日等参数构造成`time.Time`对象，除非最后一个参数（时区）是`nil`。这种情况下会引发`panic`异常。`panic`是来自被调用函数的信号，表示发生了某个已知的bug。一个良好的程序永远不应该发生`panic`异常。

对于大部分函数而言，永远无法确保能否成功运行。这是因为错误的原因超出了程序员的控制。举个例子，任何进行`I/O`操作的函数都会面临出现错误的可能，只有没有经验的程序员才会相信读写操作不会失败，即使是简单的读写。因此，当本该可信的操作出乎意料的失败后，我们必须弄清楚导致失败的原因。

在Go的错误处理中，错误是软件包API和应用程序用户界面的一个重要组成部分，程序运行失败仅被认为是几个预期的结果之一。

对于那些将运行失败看作是预期结果的函数，它们会返回一个额外的返回值，通常是最后一个，来传递错误信息。如果导致失败的原因只有一个，额外的返回值可以是一个布尔值，通常被命名为ok。比如，`cache.Lookup`失败的唯一原因是`key`不存在，那么代码可以按照下面的方式组织：

```go
value, ok := cache.Lookup(key)
if !ok {
	// ...cache[key] does not exist…
}
```

通常，导致失败的原因不止一种，尤其是对`I/O`操作而言，用户需要了解更多的错误信息。因此，额外的返回值不再是简单的布尔类型，而是`error`类型。

内置的`error`是接口类型。我们将在第七章了解接口类型的含义，以及它对错误处理的影响。现在我们只需要明白`error`类型可能是`nil`或者`non-nil`。`nil`意味着函数运行成功，`non-nil`表示失败。对于`non-nil`的`error`类型，我们可以通过调用`error`的`Error`函数或者输出函数获得字符串类型的错误信息。

```go
fmt.Println(err)
fmt.Printf("%v", err)
```

通常，当函数返回`non-nil`的`error`时，其他的返回值是未定义的（`undefined`），这些未定义的返回值应该被忽略。然而，有少部分函数在发生错误时，仍然会返回一些有用的返回值。比如，当读取文件发生错误时，`Read`函数会返回可以读取的字节数以及错误信息。对于这种情况，正确的处理方式应该是先处理这些不完整的数据，再处理错误。因此对函数的返回值要有清晰的说明，以便于其他人使用。

在Go中，函数运行失败时会返回错误信息，这些错误信息被认为是一种预期的值而非异常（`exception`），这使得Go有别于那些将函数运行失败看作是异常的语言。虽然Go有各种异常机制，但这些机制仅被使用在处理那些未被预料到的错误，即bug，而不是那些在健壮程序中应该被避免的程序错误。对于Go的异常机制我们将在5.9介绍。

Go这样设计的原因是由于对于某个应该在控制流程中处理的错误而言，将这个错误以异常的形式抛出会混乱对错误的描述，这通常会导致一些糟糕的后果。当某个程序错误被当作异常处理后，这个错误会将堆栈跟踪信息返回给终端用户，这些信息复杂且无用，无法帮助定位错误。

正因此，Go使用控制流机制（如`if`和`return`）处理错误，这使得编码人员能更多的关注错误处理。

### 5.4.1. 错误处理策略

当一次函数调用返回错误时，调用者应该选择合适的方式处理错误。根据情况的不同，有很多处理方式，让我们来看看常用的**五种**方式。

首先，也是最常用的方式是传播错误。这意味着函数中某个子程序的失败，会变成该函数的失败。下面，我们以5.3节的`findLinks`函数作为例子。如果`findLinks`对`http.Get`的调用失败，`findLinks`会直接将这个HTTP错误返回给调用者：

```go
resp, err := http.Get(url)
if err != nil{
	return nil, err
}
```

当对`html.Parse`的调用失败时，`findLinks`不会直接返回`html.Parse`的错误，因为缺少两条重要信息：

1. 发生错误时的解析器（html parser）；
2. 发生错误的`url`。因此，`findLinks`构造了一个新的错误信息，既包含了这两项，也包括了底层的解析出错的信息。

```go
doc, err := html.Parse(resp.Body)
resp.Body.Close()
if err != nil {
	return nil, fmt.Errorf("parsing %s as HTML: %v", url,err)
}
```

`fmt.Errorf`函数使用`fmt.Sprintf`格式化错误信息并返回。我们使用该函数添加额外的前缀上下文信息到原始错误信息。当错误最终由`main`函数处理时，错误信息应提供清晰的从原因到后果的因果链，就像美国宇航局事故调查时做的那样：

```go
genesis: crashed: no parachute: G-switch failed: bad relay orientation
```

由于错误信息经常是以链式组合在一起的，所以错误信息中应避免大写和换行符。最终的错误信息可能很长，我们可以通过类似`grep`的工具处理错误信息（译者注：`grep`是一种文本搜索工具）。

编写错误信息时，我们要确保错误信息对问题细节的描述是详尽的。尤其是要注意错误信息表达的一致性，即相同的函数或同包内的同一组函数返回的错误在构成和处理方式上是相似的。

以`os`包为例，`os`包确保文件操作（如`os.Open`、`Read`、`Write`、`Close`）返回的每个错误的描述不仅仅包含错误的原因（如无权限，文件目录不存在）也包含文件名，这样调用者在构造新的错误信息时无需再添加这些信息。

一般而言，被调用函数`f(x)`会将调用信息和参数信息作为发生错误时的上下文放在错误信息中并返回给调用者，调用者需要添加一些错误信息中不包含的信息，比如添加`url`到`html.Parse`返回的错误中。

让我们来看看处理错误的第二种策略。如果错误的发生是偶然性的，或由不可预知的问题导致的。一个明智的选择是重新尝试失败的操作。在重试时，我们需要限制重试的时间间隔或重试的次数，防止无限制的重试。

*gopl.io/ch5/wait*

```go
// WaitForServer attempts to contact the server of a URL.
// It tries for one minute using exponential back-off.
// It reports an error if all attempts fail.
func WaitForServer(url string) error {
	const timeout = 1 * time.Minute
	deadline := time.Now().Add(timeout)
	for tries := 0; time.Now().Before(deadline); tries++ {
		_, err := http.Head(url)
		if err == nil {
			return nil // success
		}
		log.Printf("server not responding (%s);retrying…", err)
		time.Sleep(time.Second << uint(tries)) // exponential back-off
	}
	return fmt.Errorf("server %s failed to respond after %s", url, timeout)
}
```

如果错误发生后，程序无法继续运行，我们就可以采用第三种策略：输出错误信息并结束程序。需要注意的是，这种策略只应在`main`中执行。对库函数而言，应仅向上传播错误，除非该错误意味着程序内部包含不一致性，即遇到了bug，才能在库函数中结束程序。

```go
// (In function main.)
if err := WaitForServer(url); err != nil {
	fmt.Fprintf(os.Stderr, "Site is down: %v\n", err)
	os.Exit(1)
}
```

调用`log.Fatalf`可以更简洁的代码达到与上文相同的效果。`log`中的所有函数，都默认会在错误信息之前输出时间信息。

```go
if err := WaitForServer(url); err != nil {
	log.Fatalf("Site is down: %v\n", err)
}
```

长时间运行的服务器常采用默认的时间格式，而交互式工具很少采用包含如此多信息的格式。

```go
2006/01/02 15:04:05 Site is down: no such domain:
bad.gopl.io
```

我们可以设置`log`的前缀信息屏蔽时间信息，一般而言，前缀信息会被设置成命令名。

```go
log.SetPrefix("wait: ")
log.SetFlags(0)
```

第四种策略：有时，我们只需要输出错误信息就足够了，不需要中断程序的运行。我们可以通过`log`包提供函数

```go
if err := Ping(); err != nil {
	log.Printf("ping failed: %v; networking disabled",err)
}
```

或者标准错误流输出错误信息。

```go
if err := Ping(); err != nil {
	fmt.Fprintf(os.Stderr, "ping failed: %v; networking disabled\n", err)
}
```

`log`包中的所有函数会为没有换行符的字符串增加换行符。

第五种，也是最后一种策略：我们可以直接忽略掉错误。

```go
dir, err := ioutil.TempDir("", "scratch")
if err != nil {
	return fmt.Errorf("failed to create temp dir: %v",err)
}
// ...use temp dir…
os.RemoveAll(dir) // ignore errors; $TMPDIR is cleaned periodically
```

尽管`os.RemoveAll`会失败，但上面的例子并没有做错误处理。这是因为操作系统会定期的清理临时目录。正因如此，虽然程序没有处理错误，但程序的逻辑不会因此受到影响。我们应该在每次函数调用后，都养成考虑错误处理的习惯，当你决定忽略某个错误时，你应该清晰地写下你的意图。

在Go中，错误处理有一套独特的编码风格。检查某个子函数是否失败后，我们通常将处理失败的逻辑代码放在处理成功的代码之前。如果某个错误会导致函数返回，那么成功时的逻辑代码不应放在`else`语句块中，而应直接放在函数体中。Go中大部分函数的代码结构几乎相同，首先是一系列的初始检查，防止错误发生，之后是函数的实际逻辑。

### 5.4.2. 文件结尾错误（EOF）

函数经常会返回多种错误，这对终端用户来说可能会很有趣，但对程序而言，这使得情况变得复杂。很多时候，程序必须根据错误类型，作出不同的响应。让我们考虑这样一个例子：从文件中读取`n`个字节。如果`n`等于文件的长度，读取过程的任何错误都表示失败。如果`n`小于文件的长度，调用者会重复的读取固定大小的数据直到文件结束。这会导致调用者必须分别处理由文件结束引起的各种错误。基于这样的原因，`io`包保证任何由文件结束引起的读取失败都返回同一个错误——`io.EOF`，该错误在`io`包中定义：

```go
package io

import "errors"

// EOF is the error returned by Read when no more input is available.
var EOF = errors.New("EOF")
```

调用者只需通过简单的比较，就可以检测出这个错误。下面的例子展示了如何从标准输入中读取字符，以及判断文件结束。（4.3的`chartcount`程序展示了更加复杂的代码）

```go
in := bufio.NewReader(os.Stdin)
for {
	r, _, err := in.ReadRune()
	if err == io.EOF {
		break // finished reading
	}
	if err != nil {
		return fmt.Errorf("read failed:%v", err)
	}
	// ...use r…
}
```

因为文件结束这种错误不需要更多的描述，所以`io.EOF`有固定的错误信息——“`EOF`”。对于其他错误，我们可能需要在错误信息中描述错误的类型和数量，这使得我们不能像`io.EOF`一样采用固定的错误信息。在7.11节中，我们会提出更系统的方法区分某些固定的错误值。

## 5.5. 函数值

在Go中，函数被看作第一类值（`first-class values`）：函数像其他值一样，拥有类型，可以被赋值给其他变量，传递给函数，从函数返回。对函数值（`function value`）的调用类似函数调用。例子如下：

```go
	func square(n int) int { return n * n }
	func negative(n int) int { return -n }
	func product(m, n int) int { return m * n }

	f := square
	fmt.Println(f(3)) // "9"

	f = negative
	fmt.Println(f(3))     // "-3"
	fmt.Printf("%T\n", f) // "func(int) int"

	f = product // compile error: can't assign func(int, int) int to func(int) int
```

函数类型的零值是`nil`。调用值为`nil`的函数值会引起`panic`错误：

```go
	var f func(int) int
	f(3) // 此处f的值为nil, 会引起panic错误
```

函数值可以与`nil`比较：

```go
	var f func(int) int
	if f != nil {
		f(3)
	}
```

但是函数值之间是不可比较的，也不能用函数值作为`map`的`key`。

函数值使得我们不仅仅可以通过数据来参数化函数，亦可通过行为。标准库中包含许多这样的例子。下面的代码展示了如何使用这个技巧。`strings.Map`对字符串中的每个字符调用`add1`函数，并将每个`add1`函数的返回值组成一个新的字符串返回给调用者。

```go
	func add1(r rune) rune { return r + 1 }

	fmt.Println(strings.Map(add1, "HAL-9000")) // "IBM.:111"
	fmt.Println(strings.Map(add1, "VMS"))      // "WNT"
	fmt.Println(strings.Map(add1, "Admix"))    // "Benjy"
```

5.2节的`findLinks`函数使用了辅助函数`visit`，遍历和操作了HTML页面的所有结点。使用函数值，我们可以将遍历结点的逻辑和操作结点的逻辑分离，使得我们可以复用遍历的逻辑，从而对结点进行不同的操作。

*gopl.io/ch5/outline2*

```go
// forEachNode针对每个结点x，都会调用pre(x)和post(x)。
// pre和post都是可选的。
// 遍历孩子结点之前，pre被调用
// 遍历孩子结点之后，post被调用
func forEachNode(n *html.Node, pre, post func(n *html.Node)) {
	if pre != nil {
		pre(n)
	}
	for c := n.FirstChild; c != nil; c = c.NextSibling {
		forEachNode(c, pre, post)
	}
	if post != nil {
		post(n)
	}
}
```

该函数接收2个函数作为参数，分别在结点的孩子被访问前和访问后调用。这样的设计给调用者更大的灵活性。举个例子，现在我们有`startElemen`和`endElement`两个函数用于输出HTML元素的开始标签和结束标签`...`：

```go
var depth int
func startElement(n *html.Node) {
	if n.Type == html.ElementNode {
		fmt.Printf("%*s<%s>\n", depth*2, "", n.Data)
		depth++
	}
}
func endElement(n *html.Node) {
	if n.Type == html.ElementNode {
		depth--
		fmt.Printf("%*s</%s>\n", depth*2, "", n.Data)
	}
}
```

上面的代码利用`fmt.Printf`的一个小技巧控制输出的缩进。`%*s`中的`*`会在字符串之前填充一些空格。在例子中，每次输出会先填充`depth*2`数量的空格，再输出`""`，最后再输出HTML标签。

如果我们像下面这样调用forEachNode：

```go
forEachNode(doc, startElement, endElement)
```

与之前的`outline`程序相比，我们得到了更加详细的页面结构：

```go
$ go build gopl.io/ch5/outline2
$ ./outline2 http://gopl.io
<html>
  <head>
    <meta>
    </meta>
    <title>
	</title>
	<style>
	</style>
  </head>
  <body>
    <table>
      <tbody>
        <tr>
          <td>
            <a>
              <img>
              </img>
...
```

**练习 5.7：** 完善`startElement`和`endElement`函数，使其成为通用的HTML输出器。要求：输出注释结点，文本结点以及每个元素的属性（`< a href='...'>`）。使用简略格式输出没有孩子结点的元素（即用``代替``）。编写测试，验证程序输出的格式正确。（详见11章）

```go
// pretty.go
package main

import (
	"fmt"
	"io"
	"log"
	"os"
	"strings"

	"golang.org/x/net/html"
)

var depth int

type PrettyPrinter struct {
	w   io.Writer
	err error
}

func NewPrettyPrinter() PrettyPrinter {
	return PrettyPrinter{}
}

func (pp PrettyPrinter) Pretty(w io.Writer, n *html.Node) error {
	pp.w = w
	pp.err = nil
	pp.forEachNode(n, pp.start, pp.end)
	return pp.Err()
}

func (pp PrettyPrinter) Err() error {
	return pp.err
}

func (pp PrettyPrinter) forEachNode(n *html.Node, pre, post func(n *html.Node)) {
	if pre != nil {
		pre(n)
	}
	if pp.Err() != nil {
		return
	}
	for c := n.FirstChild; c != nil; c = c.NextSibling {
		pp.forEachNode(c, pre, post)
	}
	if post != nil {
		post(n)
	}
	if pp.Err() != nil {
		return
	}
}

func (pp PrettyPrinter) printf(format string, args ...interface{}) {
	_, err := fmt.Fprintf(pp.w, format, args...)
	pp.err = err
}

func (pp PrettyPrinter) startElement(n *html.Node) {
	end := ">"
	if n.FirstChild == nil {
		end = "/>"
	}

	attrs := make([]string, 0, len(n.Attr))
	for _, a := range n.Attr {
		attrs = append(attrs, fmt.Sprintf(`%s="%s"`, a.Key, a.Val))
	}
	attrStr := ""
	if len(n.Attr) > 0 {
		attrStr = " " + strings.Join(attrs, " ")
	}

	name := n.Data

	pp.printf("%*s<%s%s%s\n", depth*2, "", name, attrStr, end)
	depth++
}

func (pp PrettyPrinter) endElement(n *html.Node) {
	depth--
	if n.FirstChild == nil {
		return
	}
	pp.printf("%*s</%s>\n", depth*2, "", n.Data)
}

func (pp PrettyPrinter) startText(n *html.Node) {
	text := strings.TrimSpace(n.Data)
	if len(text) == 0 {
		return
	}
	pp.printf("%*s%s\n", depth*2, "", n.Data)
}

func (pp PrettyPrinter) startComment(n *html.Node) {
	pp.printf("<!--%s-->\n", n.Data)
}

func (pp PrettyPrinter) start(n *html.Node) {
	switch n.Type {
	case html.ElementNode:
		pp.startElement(n)
	case html.TextNode:
		pp.startText(n)
	case html.CommentNode:
		pp.startComment(n)
	}
}

func (pp PrettyPrinter) end(n *html.Node) {
	switch n.Type {
	case html.ElementNode:
		pp.endElement(n)
	}
}

func main() {
	doc, err := html.Parse(os.Stdin)
	if err != nil {
		log.Fatal(err)
	}
	pp := NewPrettyPrinter()
	pp.Pretty(os.Stdout, doc)
}
```

```go
// pretty_test.go
package main

import (
	"bytes"
	"strings"
	"testing"

	"golang.org/x/net/html"
)

// But html.Parse parses pretty much anything, so this test is useless.
func TestPrettyOutputCanBeParsed(t *testing.T) {
	input := `
<html>
<body>
	<p class="something" id="short"><span class="special">hi</span></p><br/>
</body>
</html>
`
	doc, err := html.Parse(strings.NewReader(input))
	if err != nil {
		t.Error(err)
	}
	pp := NewPrettyPrinter()
	b := &bytes.Buffer{}
	err = pp.Pretty(b, doc)
	if err != nil {
		t.Log(err)
		t.Fail()
	}
	_, err = html.Parse(bytes.NewReader(b.Bytes()))
	if err != nil {
		t.Log(err)
		t.Fail()
	}
}
```

**练习 5.8：** 修改`pre`和`post`函数，使其返回布尔类型的返回值。返回`false`时，中止`forEachNoded`的遍历。使用修改后的代码编写`ElementByID`函数，根据用户输入的id查找第一个拥有该id元素的HTML元素，查找成功后，停止遍历。

```go
func ElementByID(doc *html.Node, id string) *html.Node
```

```go
package main

import (
	"fmt"
	"log"
	"os"

	"golang.org/x/net/html"
)

func ElementByID(n *html.Node, id string) *html.Node {
	pre := func(n *html.Node) bool {
		if n.Type != html.ElementNode {
			return true
		}
		for _, a := range n.Attr {
			if a.Key == "id" && a.Val == id {
				return false
			}

		}
		return true
	}
	return forEachElement(n, pre, nil)
}

func forEachElement(n *html.Node, pre, post func(n *html.Node) bool) *html.Node {
	u := make([]*html.Node, 0) // unvisited
	u = append(u, n)
	for len(u) > 0 {
		n = u[0]
		u = u[1:]
		if pre != nil {
			if !pre(n) {
				return n
			}
		}
		for c := n.FirstChild; c != nil; c = c.NextSibling {
			u = append(u, c)
		}
		if post != nil {
			if !post(n) {
				return n
			}
		}
	}
	return nil
}

func main() {
	if len(os.Args) != 3 {
		fmt.Fprintln(os.Stderr, "usage: ex5.8 HTML_FILE ID")
	}
	filename := os.Args[1]
	id := os.Args[2]
	file, err := os.Open(filename)
	if err != nil {
		log.Fatal(err)
	}
	doc, err := html.Parse(file)
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("%+v\n", ElementByID(doc, id))
}
```

**练习 5.9：** 编写函数expand，将s中的"foo"替换为f("foo")的返回值。

```go
func expand(s string, f func(string) string) string
```

```go
package main

import (
	"bytes"
	"fmt"
	"log"
	"os"
	"regexp"
	"strings"
)

var pattern = regexp.MustCompile(`\$\w+|\${\w+}`)

func expand(s string, f func(string) string) string {
	wrapper := func(s string) string {
		if strings.HasPrefix(s, "${") {
			s = s[2 : len(s)-1]
		} else {
			s = s[1:]
		}
		return f(s)
	}
	return pattern.ReplaceAllStringFunc(s, wrapper)
}

func main() {
	log.SetFlags(0)
	log.SetPrefix("ex5.9: ")

	subs := make(map[string]string, 0)
	if len(os.Args) > 1 {
		for _, arg := range os.Args[1:] {
			pieces := strings.Split(arg, "=")
			if len(pieces) != 2 {
				fmt.Fprintln(os.Stderr, "usage: ex5.9 KEY=VAL ...")
				os.Exit(1)
			}
			k, v := pieces[0], pieces[1]
			subs[k] = v
		}
	}

	missing := make([]string, 0)
	used := make(map[string]bool, 0)

	f := func(s string) string {
		v, ok := subs[s]
		if !ok {
			missing = append(missing, s)
			return "$" + s
		}
		used[s] = true
		return v
	}

	b := &bytes.Buffer{}
	b.ReadFrom(os.Stdin)
	fmt.Print(expand(b.String(), f))

	unused := make([]string, 0)
	for k, _ := range subs {
		if !used[k] {
			unused = append(unused, k)
		}
	}

	if len(unused) > 0 {
		log.Printf("unused bindings: %s", strings.Join(unused, " "))
	}
	if len(missing) > 0 {
		log.Printf("missing bindings: %s", strings.Join(missing, " "))
	}
}
```

## 5.6. 匿名函数

拥有函数名的函数只能在包级语法块中被声明，通过函数字面量（`function literal`），我们可绕过这一限制，在任何表达式中表示一个函数值。函数字面量的语法和函数声明相似，区别在于`func`关键字后没有函数名。函数值字面量是一种表达式，它的值被称为匿名函数（`anonymous function`）。

函数字面量允许我们在使用函数时，再定义它。通过这种技巧，我们可以改写之前对`strings.Map`的调用：

```go
strings.Map(func(r rune) rune { return r + 1 }, "HAL-9000")
```

更为重要的是，通过这种方式定义的函数可以访问完整的词法环境（lexical environment），这意味着在函数中定义的内部函数可以引用该函数的变量，如下例所示：

*gopl.io/ch5/squares*

```go
// squares返回一个匿名函数。
// 该匿名函数每次被调用时都会返回下一个数的平方。
func squares() func() int {
	var x int
	return func() int {
		x++
		return x * x
	}
}
func main() {
	f := squares()
	fmt.Println(f()) // "1"
	fmt.Println(f()) // "4"
	fmt.Println(f()) // "9"
	fmt.Println(f()) // "16"
}
```

函数`squares`返回另一个类型为 `func() int` 的函数。对`squares`的一次调用会生成一个局部变量`x`并返回一个匿名函数。每次调用匿名函数时，该函数都会先使`x`的值加1，再返回`x`的平方。第二次调用`squares`时，会生成第二个`x`变量，并返回一个新的匿名函数。新匿名函数操作的是第二个`x`变量。

`squares`的例子证明，函数值不仅仅是一串代码，还记录了状态。在`squares`中定义的匿名内部函数可以访问和更新`squares`中的局部变量，这意味着匿名函数和`squares`中，存在变量引用。这就是函数值属于引用类型和函数值不可比较的原因。Go使用闭包（`closures`）技术实现函数值，Go程序员也把函数值叫做闭包。

通过这个例子，我们看到变量的生命周期不由它的作用域决定：`squares`返回后，变量`x`仍然隐式的存在于`f`中。

接下来，我们讨论一个有点学术性的例子，考虑这样一个问题：给定一些计算机课程，每个课程都有前置课程，只有完成了前置课程才可以开始当前课程的学习；我们的目标是选择出一组课程，这组课程必须确保按顺序学习时，能全部被完成。每个课程的前置课程如下：

*gopl.io/ch5/toposort*

```go
// prereqs记录了每个课程的前置课程
var prereqs = map[string][]string{
	"algorithms": {"data structures"},
	"calculus": {"linear algebra"},
	"compilers": {
		"data structures",
		"formal languages",
		"computer organization",
	},
	"data structures":       {"discrete math"},
	"databases":             {"data structures"},
	"discrete math":         {"intro to programming"},
	"formal languages":      {"discrete math"},
	"networks":              {"operating systems"},
	"operating systems":     {"data structures", "computer organization"},
	"programming languages": {"data structures", "computer organization"},
}
```

这类问题被称作拓扑排序。从概念上说，前置条件可以构成有向图。图中的顶点表示课程，边表示课程间的依赖关系。显然，图中应该无环，这也就是说从某点出发的边，最终不会回到该点。下面的代码用深度优先搜索了整张图，获得了符合要求的课程序列。

```go
func main() {
	for i, course := range topoSort(prereqs) {
		fmt.Printf("%d:\t%s\n", i+1, course)
	}
}

func topoSort(m map[string][]string) []string {
	var order []string
	seen := make(map[string]bool)
	var visitAll func(items []string)
	visitAll = func(items []string) {
		for _, item := range items {
			if !seen[item] {
				seen[item] = true
				visitAll(m[item])
				order = append(order, item)
			}
		}
	}
	var keys []string
	for key := range m {
		keys = append(keys, key)
	}
	sort.Strings(keys)
	visitAll(keys)
	return order
}
```

当匿名函数需要被递归调用时，我们必须首先声明一个变量（在上面的例子中，我们首先声明了 `visitAll`），再将匿名函数赋值给这个变量。如果不分成两步，函数字面量无法与`visitAll`绑定，我们也无法递归调用该匿名函数。

```go
visitAll := func(items []string) {
	// ...
	visitAll(m[item]) // compile error: undefined: visitAll
	// ...
}
```

在`topsort`中，首先对`prereqs`中的key排序，再调用`visitAll`。因为`prereqs`映射的是切片而不是更复杂的`map`，所以数据的遍历次序是固定的，这意味着你每次运行`topsort`得到的输出都是一样的。 `topsort`的输出结果如下:

```go
1: intro to programming
2: discrete math
3: data structures
4: algorithms
5: linear algebra
6: calculus
7: formal languages
8: computer organization
9: compilers
10: databases
11: operating systems
12: networks
13: programming languages
```

让我们回到`findLinks`这个例子。我们将代码移动到了`links`包下，将函数重命名为`Extract`，在第八章我们会再次用到这个函数。新的匿名函数被引入，用于替换原来的`visit`函数。该匿名函数负责将新连接添加到切片中。在`Extract`中，使用`forEachNode`遍历HTML页面，由于`Extract`只需要在遍历结点前操作结点，所以`forEachNode`的`post`参数被传入`nil`。

*gopl.io/ch5/links*

```go
// Package links provides a link-extraction function.
package links
import (
	"fmt"
	"net/http"
	"golang.org/x/net/html"
)
// Extract makes an HTTP GET request to the specified URL, parses
// the response as HTML, and returns the links in the HTML document.
func Extract(url string) ([]string, error) {
	resp, err := http.Get(url)
	if err != nil {
		return nil, err
	}
	if resp.StatusCode != http.StatusOK {
	resp.Body.Close()
		return nil, fmt.Errorf("getting %s: %s", url, resp.Status)
	}
	doc, err := html.Parse(resp.Body)
	resp.Body.Close()
	if err != nil {
		return nil, fmt.Errorf("parsing %s as HTML: %v", url, err)
	}
	var links []string
	visitNode := func(n *html.Node) {
		if n.Type == html.ElementNode && n.Data == "a" {
			for _, a := range n.Attr {
				if a.Key != "href" {
					continue
				}
				link, err := resp.Request.URL.Parse(a.Val)
				if err != nil {
					continue // ignore bad URLs
				}
				links = append(links, link.String())
			}
		}
	}
	forEachNode(doc, visitNode, nil)
	return links, nil
}
```

上面的代码对之前的版本做了改进，现在`links`中存储的不是`href`属性的原始值，而是通过`resp.Request.URL`解析后的值。解析后，这些连接以绝对路径的形式存在，可以直接被`http.Get`访问。

网页抓取的核心问题就是如何遍历图。在`topoSort`的例子中，已经展示了深度优先遍历，在网页抓取中，我们会展示如何用广度优先遍历图。在第8章，我们会介绍如何将深度优先和广度优先结合使用。

下面的函数实现了广度优先算法。调用者需要输入一个初始的待访问列表和一个函数f。待访问列表中的每个元素被定义为`string`类型。广度优先算法会为每个元素调用一次f。每次f执行完毕后，会返回一组待访问元素。这些元素会被加入到待访问列表中。当待访问列表中的所有元素都被访问后，`breadthFirst`函数运行结束。为了避免同一个元素被访问两次，代码中维护了一个`map`。

*gopl.io/ch5/findlinks3*

```go
// breadthFirst calls f for each item in the worklist.
// Any items returned by f are added to the worklist.
// f is called at most once for each item.
func breadthFirst(f func(item string) []string, worklist []string) {
	seen := make(map[string]bool)
	for len(worklist) > 0 {
		items := worklist
		worklist = nil
		for _, item := range items {
			if !seen[item] {
				seen[item] = true
				worklist = append(worklist, f(item)...)
			}
		}
	}
}
```

就像我们在章节3解释的那样，`append`的参数“`f(item)...`”，会将`f`返回的一组元素一个个添加到`worklist`中。

在我们网页抓取器中，元素的类型是`url`。`crawl`函数会将URL输出，提取其中的新链接，并将这些新链接返回。我们会将crawl作为参数传递给`breadthFirst`。

```go
func crawl(url string) []string {
	fmt.Println(url)
	list, err := links.Extract(url)
	if err != nil {
		log.Print(err)
	}
	return list
}
```

为了使抓取器开始运行，我们用命令行输入的参数作为初始的待访问`url`。

```go
func main() {
	// Crawl the web breadth-first,
	// starting from the command-line arguments.
	breadthFirst(crawl, os.Args[1:])
}
```

让我们从 [https://golang.org](https://golang.org/) 开始，下面是程序的输出结果：

```go
$ go build gopl.io/ch5/findlinks3
$ ./findlinks3 https://golang.org
https://golang.org/
https://golang.org/doc/
https://golang.org/pkg/
https://golang.org/project/
https://code.google.com/p/go-tour/
https://golang.org/doc/code.html
https://www.youtube.com/watch?v=XCsL89YtqCs
http://research.swtch.com/gotour
```

当所有发现的链接都已经被访问或电脑的内存耗尽时，程序运行结束。

**练习5.10：** 重写`topoSort`函数，用`map`代替切片并移除对`key`的排序代码。验证结果的正确性（结果不唯一）。

```go
package main

import (
	"fmt"
)

// prereqs maps computer science courses to their prerequisites.
var prereqs = map[string][]string{
	"algorithms": {"data structures"},
	"calculus":   {"linear algebra"},

	"compilers": {
		"data structures",
		"formal languages",
		"computer organization",
	},

	"data structures":       {"discrete math"},
	"databases":             {"data structures"},
	"discrete math":         {"intro to programming"},
	"formal languages":      {"discrete math"},
	"networks":              {"operating systems"},
	"operating systems":     {"data structures", "computer organization"},
	"programming languages": {"data structures", "computer organization"},
}

func main() {
	for i, course := range topoSort(prereqs) {
		fmt.Printf("%d:\t%s\n", i+1, course)
	}
}

func topoSort(m map[string][]string) []string {
	var order []string
	seen := make(map[string]bool)
	var visitAll func([]string)

	visitAll = func(items []string) {
		for _, v := range items {
			if !seen[v] {
				seen[v] = true
				visitAll(m[v])
				order = append(order, v)
			}
		}
	}

	for k := range m {
		visitAll([]string{k})
	}
	return order
}
```

```go
1: intro to programming
2: discrete math
3: data structures
4: algorithms
5: linear algebra
6: calculus
7: formal languages
8: computer organization
9: compilers
10: databases
11: operating systems
12: networks
13: programming languages
```

**练习5.11：** 现在线性代数的老师把微积分设为了前置课程。完善`topSort`，使其能检测有向图中的环。

```go
package main

import (
	"fmt"
	"os"
	"strings"
)

// prereqs maps computer science courses to their prerequisites.
var prereqs = map[string][]string{
	"algorithms":     {"data structures"},
	"calculus":       {"linear algebra"},
	"linear algebra": {"calculus"},
	// uncomment line below to introduce a dependency cycle.
	//"intro to programming": {"data structures"},

	"compilers": {
		"data structures",
		"formal languages",
		"computer organization",
	},

	"data structures":       {"discrete math"},
	"databases":             {"data structures"},
	"discrete math":         {"intro to programming"},
	"formal languages":      {"discrete math"},
	"networks":              {"operating systems"},
	"operating systems":     {"data structures", "computer organization"},
	"programming languages": {"data structures", "computer organization"},
}

func main() {
	order, err := topoSort(prereqs)
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	for i, course := range order {
		fmt.Printf("%d:\t%s\n", i+1, course)
	}
}

func index(s string, slice []string) (int, error) {
	for i, v := range slice {
		if s == v {
			return i, nil
		}
	}
	return 0, fmt.Errorf("not found")
}

func topoSort(m map[string][]string) (order []string, err error) {
	resolved := make(map[string]bool)
	var visitAll func([]string, []string)

	visitAll = func(items []string, parents []string) {
		for _, v := range items {
			vResolved, seen := resolved[v]
			if seen && !vResolved {
				start, _ := index(v, parents) // Ignore error since v has to be in parents.
				err = fmt.Errorf("cycle: %s", strings.Join(append(parents[start:], v), " -> "))
			}
			if !seen {
				resolved[v] = false
				visitAll(m[v], append(parents, v))
				resolved[v] = true
				order = append(order, v)
			}
		}
	}

	for k := range m {
		if err != nil {
			return nil, err
		}
		visitAll([]string{k}, nil)
	}
	return order, nil
}
```

**练习5.12：** `gopl.io/ch5/outline2`（5.5节）的`startElement`和`endElement`共用了全局变量`depth`，将它们修改为匿名函数，使其共享`outline`中的局部变量。

```go
package main

import (
	"fmt"
	"net/http"
	"os"

	"golang.org/x/net/html"
)

func main() {
	for _, url := range os.Args[1:] {
		outline(url)
	}
}

func outline(url string) error {
	resp, err := http.Get(url)
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	doc, err := html.Parse(resp.Body)
	if err != nil {
		return err
	}

	var depth int

	startElement := func(n *html.Node) {
		if n.Type == html.ElementNode {
			fmt.Printf("%*s<%s>\n", depth*2, "", n.Data)
			depth++
		}
	}

	endElement := func(n *html.Node) {
		if n.Type == html.ElementNode {
			depth--
			fmt.Printf("%*s</%s>\n", depth*2, "", n.Data)
		}
	}

	forEachNode(doc, startElement, endElement)

	return nil
}

// forEachNode calls the functions pre(x) and post(x) for each node
// x in the tree rooted at n. Both functions are optional.
// pre is called before the children are visited (preorder) and
// post is called after (postorder).
func forEachNode(n *html.Node, pre, post func(n *html.Node)) {
	if pre != nil {
		pre(n)
	}

	for c := n.FirstChild; c != nil; c = c.NextSibling {
		forEachNode(c, pre, post)
	}

	if post != nil {
		post(n)
	}
}
```

**练习5.13：** 修改`crawl`，使其能保存发现的页面，必要时，可以创建目录来保存这些页面。只保存来自原始域名下的页面。假设初始页面在`golang.org`下，就不要保存`vimeo.com`下的页面。

```go
package main

import (
	"fmt"
	"io"
	"log"
	"net/http"
	"net/url"
	"os"
	"path/filepath"

	"gopl.io/ch5/links"
)

// breadthFirst calls f for each item in the worklist.
// Any items returned by f are added to the worklist.
// f is called at most once for each item.
func breadthFirst(f func(item string) []string, worklist []string) {
	seen := make(map[string]bool)
	for len(worklist) > 0 {
		items := worklist
		worklist = nil
		for _, item := range items {
			if !seen[item] {
				seen[item] = true
				worklist = append(worklist, f(item)...)
			}
		}
	}
}

var origHost string

func save(rawurl string) error {
	url, err := url.Parse(rawurl)
	if err != nil {
		return fmt.Errorf("bad url: %s", err)
	}
	if origHost == "" {
		origHost = url.Host
	}
	if origHost != url.Host {
		return nil
	}
	dir := url.Host
	var filename string
	if filepath.Ext(filename) == "" {
		dir = filepath.Join(dir, url.Path)
		filename = filepath.Join(dir, "index.html")
	} else {
		dir = filepath.Join(dir, filepath.Dir(url.Path))
		filename = url.Path
	}
	err = os.MkdirAll(dir, 0777)
	if err != nil {
		return err
	}
	resp, err := http.Get(rawurl)
	if err != nil {
		return err
	}
	defer resp.Body.Close()
	file, err := os.Create(filename)
	if err != nil {
		return err
	}
	_, err = io.Copy(file, resp.Body)
	if err != nil {
		return err
	}
	// Check for delayed write errors, as mentioned at the end of section 5.8.
	err = file.Close()
	if err != nil {
		return err
	}
	return nil
}

func crawl(url string) []string {
	fmt.Println(url)
	err := save(url)
	if err != nil {
		log.Printf(`can't cache "%s": %s`, url, err)
	}
	list, err := links.Extract(url)
	if err != nil {
		log.Printf(`can't extract links from "%s": %s`, url, err)
	}
	return list
}

func main() {
	// Crawl the web breadth-first,
	// starting from the command-line arguments.
	breadthFirst(crawl, os.Args[1:])
}
```

**练习5.14：** 使用`breadthFirst`遍历其他数据结构。比如，`topoSort`例子中的课程依赖关系（有向图）、个人计算机的文件层次结构（树）；你所在城市的公交或地铁线路（无向图）。

```go
package main

import (
	"fmt"
)

// course preregs from ex5.10
var prereqs = map[string][]string{
	"algorithms": {"data structures"},
	"calculus":   {"linear algebra"},

	"compilers": {
		"data structures",
		"formal languages",
		"computer organization",
	},

	"data structures":       {"discrete math"},
	"databases":             {"data structures"},
	"discrete math":         {"intro to programming"},
	"formal languages":      {"discrete math"},
	"networks":              {"operating systems"},
	"operating systems":     {"data structures", "computer organization"},
	"programming languages": {"data structures", "computer organization"},
}

// breadthFirst calls f for each item in the worklist.
// Any items returned by f are added to the worklist.
// f is called at most once for each item.
func breadthFirst(f func(item string) []string, worklist []string) {
	seen := make(map[string]bool)
	for len(worklist) > 0 {
		items := worklist
		worklist = nil
		for _, item := range items {
			if !seen[item] {
				seen[item] = true
				worklist = append(worklist, f(item)...)
			}
		}
	}
}

func deps(course string) []string {
	fmt.Println(course)
	return prereqs[course]
}

func main() {
	var course string
	for course = range prereqs { // get random key
		break
	}
	breadthFirst(deps, []string{course})
}
```

### 5.6.1. 警告：捕获迭代变量

本节，将介绍Go词法作用域的一个陷阱。请务必仔细的阅读，弄清楚发生问题的原因。即使是经验丰富的程序员也会在这个问题上犯错误。

考虑这样一个问题：你被要求首先创建一些目录，再将目录删除。在下面的例子中我们用函数值来完成删除操作。下面的示例代码需要引入`os`包。为了使代码简单，我们忽略了所有的异常处理。

```go
var rmdirs []func()
for _, d := range tempDirs() {
	dir := d // NOTE: necessary!
	os.MkdirAll(dir, 0755) // creates parent directories too
	rmdirs = append(rmdirs, func() {
		os.RemoveAll(dir)
	})
}
// ...do some work…
for _, rmdir := range rmdirs {
	rmdir() // clean up
}
```

你可能会感到困惑，为什么要在循环体中用循环变量d赋值一个新的局部变量，而不是像下面的代码一样直接使用循环变量`dir`。需要注意，下面的代码是错误的。

```go
var rmdirs []func()
for _, dir := range tempDirs() {
	os.MkdirAll(dir, 0755)
	rmdirs = append(rmdirs, func() {
		os.RemoveAll(dir) // NOTE: incorrect!
	})
}
```

问题的原因在于循环变量的作用域。在上面的程序中，`for`循环语句引入了新的词法块，循环变量`dir`在这个词法块中被声明。在该循环中生成的所有函数值都共享相同的循环变量。需要注意，函数值中记录的是循环变量的内存地址，而不是循环变量某一时刻的值。以`dir`为例，后续的迭代会不断更新`dir`的值，当删除操作执行时，`for`循环已完成，`dir`中存储的值等于最后一次迭代的值。这意味着，每次对`os.RemoveAll`的调用删除的都是相同的目录。

通常，为了解决这个问题，我们会引入一个与循环变量同名的局部变量，作为循环变量的副本。比如下面的变量`dir`，虽然这看起来很奇怪，但却很有用。

```go
for _, dir := range tempDirs() {
	dir := dir // declares inner dir, initialized to outer dir
	// ...
}
```

这个问题不仅存在基于`range`的循环，在下面的例子中，对循环变量`i`的使用也存在同样的问题：

```go
var rmdirs []func()
dirs := tempDirs()
for i := 0; i < len(dirs); i++ {
	os.MkdirAll(dirs[i], 0755) // OK
	rmdirs = append(rmdirs, func() {
		os.RemoveAll(dirs[i]) // NOTE: incorrect!
	})
}
```

如果你使用`go`语句（第八章）或者`defer`语句（5.8节）会经常遇到此类问题。这不是`go`或`defer`本身导致的，而是因为它们都会等待循环结束后，再执行函数值。

## 5.7. 可变参数

参数数量可变的函数称为**可变参数函数**。典型的例子就是`fmt.Printf`和类似函数。`Printf`首先接收一个必备的参数，之后接收任意个数的后续参数。

在声明可变参数函数时，需要在参数列表的最后一个参数类型之前加上省略符号“`...`”，这表示该函数会接收任意数量的该类型参数。

*gopl.io/ch5/sum*

```go
func sum(vals...int) int {
	total := 0
	for _, val := range vals {
		total += val
	}
	return total
}
```

`sum`函数返回任意个`int`型参数的和。在函数体中，`vals`被看作是类型为`[]int`的切片。`sum`可以接收任意数量的`int`型参数：

```go
fmt.Println(sum())           // "0"
fmt.Println(sum(3))          // "3"
fmt.Println(sum(1, 2, 3, 4)) // "10"
```

在上面的代码中，调用者隐式的创建一个数组，并将原始参数复制到数组中，再把数组的一个切片作为参数传给被调用函数。如果原始参数已经是切片类型，我们该如何传递给`sum`？只需在最后一个参数后加上省略符。下面的代码功能与上个例子中最后一条语句相同。

```go
values := []int{1, 2, 3, 4}
fmt.Println(sum(values...)) // "10"
```

虽然在可变参数函数内部，`...int` 型参数的行为看起来很像切片类型，但实际上，可变参数函数和以切片作为参数的函数是不同的。

```go
func f(...int) {}
func g([]int) {}
fmt.Printf("%T\n", f) // "func(...int)"
fmt.Printf("%T\n", g) // "func([]int)"
```

可变参数函数经常被用于格式化字符串。下面的`errorf`函数构造了一个以行号开头的，经过格式化的错误信息。函数名的后缀`f`是一种通用的命名规范，代表该可变参数函数可以接收`Printf`风格的格式化字符串。

```go
func errorf(linenum int, format string, args ...interface{}) {
	fmt.Fprintf(os.Stderr, "Line %d: ", linenum)
	fmt.Fprintf(os.Stderr, format, args...)
	fmt.Fprintln(os.Stderr)
}
linenum, name := 12, "count"
errorf(linenum, "undefined: %s", name) // "Line 12: undefined: count"
```

`interface{}`表示函数的最后一个参数可以接收任意类型，我们会在第7章详细介绍。

**练习5.15：** 编写类似`sum`的可变参数函数`max`和`min`。考虑不传参时，`max`和`min`该如何处理，再编写至少接收1个参数的版本。

```go
package main

import (
	"fmt"
)

func min(nums ...int) int {
	if len(nums) == 0 {
		return 0
	}
	m := nums[0]
	for _, n := range nums {
		if n < m {
			m = n
		}
	}
	return m
}

func max(nums ...int) int {
	if len(nums) == 0 {
		return 0
	}
	m := nums[0]
	for _, n := range nums {
		if n > m {
			m = n
		}
	}
	return m
}

func min2(first int, nums ...int) int {
	m := first
	for _, n := range nums {
		if n < m {
			m = n
		}
	}
	return m
}

func max2(first int, nums ...int) int {
	m := first
	for _, n := range nums {
		if n > m {
			m = n
		}
	}
	return m
}

func main() {
	fmt.Println(min(3, -1, 4))
	fmt.Println(max(3, -1, 4))
	fmt.Println(min2(3, -1, 4))
	fmt.Println(max2(3, -1, 4))
}
```

**练习5.16：**编写多参数版本的`strings.Join`。

```go
package main

import (
	"bytes"
	"fmt"
)

func join(sep string, strs ...string) string {
	if len(strs) == 0 {
		return ""
	}
	b := bytes.Buffer{}
	for _, s := range strs[:len(strs)-1] {
		b.WriteString(s)
		b.WriteString(sep)
	}
	b.WriteString(strs[len(strs)-1])
	return b.String()
}

func main() {
	fmt.Println(join("/", "hi", "there"))
	fmt.Println(join("/"))
}
```

**练习5.17：**编写多参数版本的`ElementsByTagName`，函数接收一个HTML结点树以及任意数量的标签名，返回与这些标签名匹配的所有元素。下面给出了2个例子：

```go
func ElementsByTagName(doc *html.Node, name...string) []*html.Node
images := ElementsByTagName(doc, "img")
headings := ElementsByTagName(doc, "h1", "h2", "h3", "h4")
```

```go
package main

import (
	"fmt"
	"log"
	"os"

	"golang.org/x/net/html"
)

func ElementsByTag(n *html.Node, tags ...string) []*html.Node {
	nodes := make([]*html.Node, 0)
	keep := make(map[string]bool, len(tags))
	for _, t := range tags {
		keep[t] = true
	}

	pre := func(n *html.Node) bool {
		if n.Type != html.ElementNode {
			return true
		}
		_, ok := keep[n.Data]
		if ok {
			nodes = append(nodes, n)
		}
		return true
	}
	forEachElement(n, pre, nil)
	return nodes
}

func forEachElement(n *html.Node, pre, post func(n *html.Node) bool) *html.Node {
	u := make([]*html.Node, 0) // unvisited
	u = append(u, n)
	for len(u) > 0 {
		n = u[0]
		u = u[1:]
		if pre != nil {
			if !pre(n) {
				return n
			}
		}
		for c := n.FirstChild; c != nil; c = c.NextSibling {
			u = append(u, c)
		}
		if post != nil {
			if !post(n) {
				return n
			}
		}
	}
	return nil
}

func main() {
	if len(os.Args) < 3 {
		fmt.Fprintln(os.Stderr, "usage: ex5.17 HTML_FILE TAG ...")
	}
	filename := os.Args[1]
	tags := os.Args[2:]
	file, err := os.Open(filename)
	if err != nil {
		log.Fatal(err)
	}
	doc, err := html.Parse(file)
	if err != nil {
		log.Fatal(err)
	}
	for _, n := range ElementsByTag(doc, tags...) {
		fmt.Printf("%+v\n", n)
	}
}
```

## 5.8. Deferred函数

在`findLinks`的例子中，我们用`http.Get`的输出作为`html.Parse`的输入。只有`url`的内容的确是HTML格式的，`html.Parse`才可以正常工作，但实际上，`url`指向的内容很丰富，可能是图片，纯文本或是其他。将这些格式的内容传递给`html.parse`，会产生不良后果。

下面的例子获取HTML页面并输出页面的标题。title函数会检查服务器返回的Content-Type字段，如果发现页面不是HTML，将终止函数运行，返回错误。

*gopl.io/ch5/title1*

```go
func title(url string) error {
	resp, err := http.Get(url)
	if err != nil {
		return err
	}
	// Check Content-Type is HTML (e.g., "text/html;charset=utf-8").
	ct := resp.Header.Get("Content-Type")
	if ct != "text/html" && !strings.HasPrefix(ct,"text/html;") {
		resp.Body.Close()
		return fmt.Errorf("%s has type %s, not text/html",url, ct)
	}
	doc, err := html.Parse(resp.Body)
	resp.Body.Close()
	if err != nil {
		return fmt.Errorf("parsing %s as HTML: %v", url,err)
	}
	visitNode := func(n *html.Node) {
		if n.Type == html.ElementNode && n.Data == "title"&&n.FirstChild != nil {
			fmt.Println(n.FirstChild.Data)
		}
	}
	forEachNode(doc, visitNode, nil)
	return nil
}
```

下面展示了运行效果：

```go
$ go build gopl.io/ch5/title1
$ ./title1 http://gopl.io
The Go Programming Language
$ ./title1 https://golang.org/doc/effective_go.html
Effective Go - The Go Programming Language
$ ./title1 https://golang.org/doc/gopher/frontpage.png
title1: https://golang.org/doc/gopher/frontpage.png has type image/png, not text/html
```

`resp.Body.close`调用了多次，这是为了确保title在所有执行路径下（即使函数运行失败）都关闭了网络连接。随着函数变得复杂，需要处理的错误也变多，维护清理逻辑变得越来越困难。而Go语言独有的defer机制可以让事情变得简单。

你只需要在调用普通函数或方法前加上关键字defer，就完成了defer所需要的语法。当执行到该条语句时，函数和参数表达式得到计算，但直到包含该defer语句的函数执行完毕时，defer后的函数才会被执行，不论包含defer语句的函数是通过return正常结束，还是由于panic导致的异常结束。你可以在一个函数中执行多条defer语句，它们的执行顺序与声明顺序相反。

defer语句经常被用于处理成对的操作，如打开、关闭、连接、断开连接、加锁、释放锁。通过defer机制，不论函数逻辑多复杂，都能保证在任何执行路径下，资源被释放。释放资源的defer应该直接跟在请求资源的语句后。在下面的代码中，一条defer语句替代了之前的所有`resp.Body.Close`

*gopl.io/ch5/title2*

```go
func title(url string) error {
	resp, err := http.Get(url)
	if err != nil {
		return err
	}
	defer resp.Body.Close()
	ct := resp.Header.Get("Content-Type")
	if ct != "text/html" && !strings.HasPrefix(ct,"text/html;") {
		return fmt.Errorf("%s has type %s, not text/html",url, ct)
	}
	doc, err := html.Parse(resp.Body)
	if err != nil {
		return fmt.Errorf("parsing %s as HTML: %v", url,err)
	}
	// ...print doc's title element…
	return nil
}
```

在处理其他资源时，也可以采用`defer`机制，比如对文件的操作：

*io/ioutil*

```go
package ioutil
func ReadFile(filename string) ([]byte, error) {
	f, err := os.Open(filename)
	if err != nil {
		return nil, err
	}
	defer f.Close()
	return ReadAll(f)
}
```

或是处理互斥锁（9.2章）

```go
var mu sync.Mutex
var m = make(map[string]int)
func lookup(key string) int {
	mu.Lock()
	defer mu.Unlock()
	return m[key]
}
```

调试复杂程序时，`defer`机制也常被用于记录何时进入和退出函数。下例中的`bigSlowOperation`函数，直接调用`trace`记录函数的被调情况。`bigSlowOperation`被调时，`trace`会返回一个函数值，该函数值会在`bigSlowOperation`退出时被调用。通过这种方式， 我们可以只通过一条语句控制函数的入口和所有的出口，甚至可以记录函数的运行时间，如例子中的`start`。需要注意一点：不要忘记`defer`语句后的圆括号，否则本该在进入时执行的操作会在退出时执行，而本该在退出时执行的，永远不会被执行。

*gopl.io/ch5/trace*

```go
func bigSlowOperation() {
	defer trace("bigSlowOperation")() // don't forget the extra parentheses
	// ...lots of work…
	time.Sleep(10 * time.Second) // simulate slow operation by sleeping
}
func trace(msg string) func() {
	start := time.Now()
	log.Printf("enter %s", msg)
	return func() { 
		log.Printf("exit %s (%s)", msg,time.Since(start)) 
	}
}
```

每一次`bigSlowOperation`被调用，程序都会记录函数的进入，退出，持续时间。（我们用`time.Sleep`模拟一个耗时的操作）

```go
$ go build gopl.io/ch5/trace
$ ./trace
2015/11/18 09:53:26 enter bigSlowOperation
2015/11/18 09:53:36 exit bigSlowOperation (10.000589217s)
```

我们知道，`defer`语句中的函数会在`return`语句更新返回值变量后再执行，又因为在函数中定义的匿名函数可以访问该函数包括返回值变量在内的所有变量，所以，对匿名函数采用`defer`机制，可以使其观察函数的返回值。

以`double`函数为例：

```go
func double(x int) int {
	return x + x
}
```

我们只需要首先命名`double`的返回值，再增加`defer`语句，我们就可以在`double`每次被调用时，输出参数以及返回值。

```go
func double(x int) (result int) {
	defer func() { fmt.Printf("double(%d) = %d\n", x,result) }()
	return x + x
}
_ = double(4)
// Output:
// "double(4) = 8"
```

可能`double`函数过于简单，看不出这个小技巧的作用，但对于有许多`return`语句的函数而言，这个技巧很有用。

被延迟执行的匿名函数甚至可以修改函数返回给调用者的返回值：

```go
func triple(x int) (result int) {
	defer func() { result += x }()
	return double(x)
}
fmt.Println(triple(4)) // "12"
```

在循环体中的`defer`语句需要特别注意，因为只有在函数执行完毕后，这些被延迟的函数才会执行。下面的代码会导致系统的文件描述符耗尽，因为在所有文件都被处理之前，没有文件会被关闭。

```go
for _, filename := range filenames {
	f, err := os.Open(filename)
	if err != nil {
		return err
	}
	defer f.Close() // NOTE: risky; could run out of file descriptors
	// ...process f…
}
```

一种解决方法是将循环体中的`defer`语句移至另外一个函数。在每次循环时，调用这个函数。

```go
for _, filename := range filenames {
	if err := doFile(filename); err != nil {
		return err
	}
}
func doFile(filename string) error {
	f, err := os.Open(filename)
	if err != nil {
		return err
	}
	defer f.Close()
	// ...process f…
}
```

下面的代码是`fetch`（1.5节）的改进版，我们将`http`响应信息写入本地文件而不是从标准输出流输出。我们通过`path.Base`提出`url`路径的最后一段作为文件名。

*gopl.io/ch5/fetch*

```go
// Fetch downloads the URL and returns the
// name and length of the local file.
func fetch(url string) (filename string, n int64, err error) {
	resp, err := http.Get(url)
	if err != nil {
		return "", 0, err
	}
	defer resp.Body.Close()
	local := path.Base(resp.Request.URL.Path)
	if local == "/" {
		local = "index.html"
	}
	f, err := os.Create(local)
	if err != nil {
		return "", 0, err
	}
	n, err = io.Copy(f, resp.Body)
	// Close file, but prefer error from Copy, if any.
	if closeErr := f.Close(); err == nil {
		err = closeErr
	}
	return local, n, err
}
```

对`resp.Body.Close`延迟调用我们已经见过了，在此不做解释。上例中，通过`os.Create`打开文件进行写入，在关闭文件时，我们没有对`f.close`采用`defer`机制，因为这会产生一些微妙的错误。许多文件系统，尤其是NFS，写入文件时发生的错误会被延迟到文件关闭时反馈。如果没有检查文件关闭时的反馈信息，可能会导致数据丢失，而我们还误以为写入操作成功。如果`io.Copy`和`f.close`都失败了，我们倾向于将`io.Copy`的错误信息反馈给调用者，因为它先于`f.close`发生，更有可能接近问题的本质。

**练习5.18：**不修改fetch的行为，重写fetch函数，要求使用defer机制关闭文件。

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"os"
	"path"
)

// Fetch downloads the URL and returns the
// name and length of the local file.
func fetch(url string) (filename string, n int64, err error) {
	resp, err := http.Get(url)
	if err != nil {
		return "", 0, err
	}
	defer resp.Body.Close()

	local := path.Base(resp.Request.URL.Path)
	if local == "/" {
		local = "index.html"
	}
	f, err := os.Create(local)
	if err != nil {
		return "", 0, err
	}
	defer func() {
		// Close file, but prefer error from Copy, if any.
		if closeErr := f.Close(); err == nil {
			err = closeErr
		}
	}()

	n, err = io.Copy(f, resp.Body)
	return local, n, err
}

func main() {
	for _, url := range os.Args[1:] {
		local, n, err := fetch(url)
		if err != nil {
			fmt.Fprintf(os.Stderr, "fetch %s: %v\n", url, err)
			continue
		}
		fmt.Fprintf(os.Stderr, "%s => %s (%d bytes).\n", url, local, n)
	}
}
```

## 5.9. Panic异常

Go的类型系统会在编译时捕获很多错误，但有些错误只能在运行时检查，如数组访问越界、空指针引用等。这些运行时错误会引起`painc`异常。

一般而言，当`panic`异常发生时，程序会中断运行，并立即执行在该`goroutine`（可以先理解成线程，在第8章会详细介绍）中被延迟的函数（`defer` 机制）。随后，程序崩溃并输出日志信息。日志信息包括`panic value`和函数调用的堆栈跟踪信息。`panic value`通常是某种错误信息。对于每个`goroutine`，日志信息中都会有与之相对的，发生`panic`时的函数调用堆栈跟踪信息。通常，我们不需要再次运行程序去定位问题，日志信息已经提供了足够的诊断依据。因此，在我们填写问题报告时，一般会将`panic`异常和日志信息一并记录。

不是所有的panic异常都来自运行时，直接调用内置的panic函数也会引发panic异常；panic函数接受任何值作为参数。当某些不应该发生的场景发生时，我们就应该调用panic。比如，当程序到达了某条逻辑上不可能到达的路径：

```go
switch s := suit(drawCard()); s {
	case "Spades":                                // ...
	case "Hearts":                                // ...
	case "Diamonds":                              // ...
	case "Clubs":                                 // ...
	default:
		panic(fmt.Sprintf("invalid suit %q", s)) // Joker?
}
```

断言函数必须满足的前置条件是明智的做法，但这很容易被滥用。除非你能提供更多的错误信息，或者能更快速的发现错误，否则不需要使用断言，编译器在运行时会帮你检查代码。

```go
func Reset(x *Buffer) {
	if x == nil {
		panic("x is nil") // unnecessary!
	}
	x.elements = nil
}
```

虽然Go的panic机制类似于其他语言的异常，但panic的适用场景有一些不同。由于panic会引起程序的崩溃，因此panic一般用于严重错误，如程序内部的逻辑不一致。勤奋的程序员认为任何崩溃都表明代码中存在漏洞，所以对于大部分漏洞，我们应该使用Go提供的错误机制，而不是panic，尽量避免程序的崩溃。在健壮的程序中，任何可以预料到的错误，如不正确的输入、错误的配置或是失败的`I/O`操作都应该被优雅的处理，最好的处理方式，就是使用Go的错误机制。

考虑`regexp.Compile`函数，该函数将正则表达式编译成有效的可匹配格式。当输入的正则表达式不合法时，该函数会返回一个错误。当调用者明确的知道正确的输入不会引起函数错误时，要求调用者检查这个错误是不必要和累赘的。我们应该假设函数的输入一直合法，就如前面的断言一样：当调用者输入了不应该出现的输入时，触发panic异常。

在程序源码中，大多数正则表达式是字符串字面值（string literals），因此`regexp`包提供了包装函数`regexp.MustCompile`检查输入的合法性。

```go
package regexp
func Compile(expr string) (*Regexp, error) { /* ... */ }
func MustCompile(expr string) *Regexp {
	re, err := Compile(expr)
	if err != nil {
		panic(err)
	}
	return re
}
```

包装函数使得调用者可以便捷的用一个编译后的正则表达式为包级别的变量赋值：

```
var httpSchemeRE = regexp.MustCompile(`^https?:`) //"http:" or "https:"
```

显然，`MustCompile`不能接收不合法的输入。函数名中的`Must`前缀是一种针对此类函数的命名约定，比如`template.Must`（4.6节）

```go
func main() {
	f(3)
}
func f(x int) {
	fmt.Printf("f(%d)\n", x+0/x) // panics if x == 0
	defer fmt.Printf("defer %d\n", x)
	f(x - 1)
}
```

上例中的运行输出如下：

```go
f(3)
f(2)
f(1)
defer 1
defer 2
defer 3
```

当`f(0)`被调用时，发生`panic`异常，之前被延迟执行的3个`fmt.Printf`被调用。程序中断执行后，`panic`信息和堆栈信息会被输出（下面是简化的输出）：

```go
panic: runtime error: integer divide by zero
main.f(0)
src/gopl.io/ch5/defer1/defer.go:14
main.f(1)
src/gopl.io/ch5/defer1/defer.go:16
main.f(2)
src/gopl.io/ch5/defer1/defer.go:16
main.f(3)
src/gopl.io/ch5/defer1/defer.go:16
main.main()
src/gopl.io/ch5/defer1/defer.go:10
```

我们在下一节将看到，如何使程序从`panic`异常中恢复，阻止程序的崩溃。

为了方便诊断问题，`runtime`包允许程序员输出堆栈信息。在下面的例子中，我们通过在`main`函数中延迟调用`printStack`输出堆栈信息。

*gopl.io/ch5/defer2*

```go
func main() {
	defer printStack()
	f(3)
}
func printStack() {
	var buf [4096]byte
	n := runtime.Stack(buf[:], false)
	os.Stdout.Write(buf[:n])
}
```

`printStack`的简化输出如下（下面只是`printStack`的输出，不包括`panic`的日志信息）：

```go
goroutine 1 [running]:
main.printStack()
src/gopl.io/ch5/defer2/defer.go:20
main.f(0)
src/gopl.io/ch5/defer2/defer.go:27
main.f(1)
src/gopl.io/ch5/defer2/defer.go:29
main.f(2)
src/gopl.io/ch5/defer2/defer.go:29
main.f(3)
src/gopl.io/ch5/defer2/defer.go:29
main.main()
src/gopl.io/ch5/defer2/defer.go:15
```

将`panic`机制类比其他语言异常机制的读者可能会惊讶，`runtime.Stack`为何能输出已经被释放函数的信息？在Go的`panic`机制中，延迟函数的调用在释放堆栈信息之前。

## 5.10. Recover捕获异常

通常来说，不应该对panic异常做任何处理，但有时，也许我们可以从异常中恢复，至少我们可以在程序崩溃前，做一些操作。举个例子，当web服务器遇到不可预料的严重问题时，在崩溃前应该将所有的连接关闭；如果不做任何处理，会使得客户端一直处于等待状态。如果web服务器还在开发阶段，服务器甚至可以将异常信息反馈到客户端，帮助调试。

如果在deferred函数中调用了内置函数recover，并且定义该defer语句的函数发生了panic异常，recover会使程序从panic中恢复，并返回panic value。导致panic异常的函数不会继续运行，但能正常返回。在未发生panic时调用recover，recover会返回nil。

让我们以语言解析器为例，说明recover的使用场景。考虑到语言解析器的复杂性，即使某个语言解析器目前工作正常，也无法肯定它没有漏洞。因此，当某个异常出现时，我们不会选择让解析器崩溃，而是会将panic异常当作普通的解析错误，并附加额外信息提醒用户报告此错误。

```go
func Parse(input string) (s *Syntax, err error) {
	defer func() {
		if p := recover(); p != nil {
			err = fmt.Errorf("internal error: %v", p)
		}
	}()
	// ...parser...
}
```

`deferred`函数帮助Parse从`panic`中恢复。在`deferred`函数内部，`panic value`被附加到错误信息中；并用`err`变量接收错误信息，返回给调用者。我们也可以通过调用`runtime.Stack`往错误信息中添加完整的堆栈调用信息。

不加区分的恢复所有的panic异常，不是可取的做法；因为在panic之后，无法保证包级变量的状态仍然和我们预期一致。比如，对数据结构的一次重要更新没有被完整完成、文件或者网络连接没有被关闭、获得的锁没有被释放。此外，如果写日志时产生的panic被不加区分的恢复，可能会导致漏洞被忽略。

虽然把对panic的处理都集中在一个包下，有助于简化对复杂和不可以预料问题的处理，但作为被广泛遵守的规范，你不应该试图去恢复其他包引起的panic。公有的API应该将函数的运行失败作为error返回，而不是panic。同样的，你也不应该恢复一个由他人开发的函数引起的panic，比如说调用者传入的回调函数，因为你无法确保这样做是安全的。

有时我们很难完全遵循规范，举个例子，net/http包中提供了一个web服务器，将收到的请求分发给用户提供的处理函数。很显然，我们不能因为某个处理函数引发的panic异常，杀掉整个进程；web服务器遇到处理函数导致的panic时会调用recover，输出堆栈信息，继续运行。这样的做法在实践中很便捷，但也会引起资源泄漏，或是因为recover操作，导致其他问题。

基于以上原因，安全的做法是有选择性的recover。换句话说，只恢复应该被恢复的panic异常，此外，这些异常所占的比例应该尽可能的低。为了标识某个panic是否应该被恢复，我们可以将panic value设置成特殊类型。在recover时对panic value进行检查，如果发现panic value是特殊类型，就将这个panic作为errror处理，如果不是，则按照正常的panic进行处理（在下面的例子中，我们会看到这种方式）。

下面的例子是title函数的变形，如果HTML页面包含多个``，该函数会给调用者返回一个错误（error）。在soleTitle内部处理时，如果检测到有多个``，会调用panic，阻止函数继续递归，并将特殊类型bailout作为panic的参数。

```go
// soleTitle returns the text of the first non-empty title element
// in doc, and an error if there was not exactly one.
func soleTitle(doc *html.Node) (title string, err error) {
	type bailout struct{}
	defer func() {
		switch p := recover(); p {
		case nil:       // no panic
		case bailout{}: // "expected" panic
			err = fmt.Errorf("multiple title elements")
		default:
			panic(p) // unexpected panic; carry on panicking
		}
	}()
	// Bail out of recursion if we find more than one nonempty title.
	forEachNode(doc, func(n *html.Node) {
		if n.Type == html.ElementNode && n.Data == "title" &&
			n.FirstChild != nil {
			if title != "" {
				panic(bailout{}) // multiple titleelements
			}
			title = n.FirstChild.Data
		}
	}, nil)
	if title == "" {
		return "", fmt.Errorf("no title element")
	}
	return title, nil
}
```

在上例中，`deferred`函数调用`recover`，并检查`panic value`。当`panic value`是`bailout{}`类型时，`deferred`函数生成一个`error`返回给调用者。当`panic value`是其他`non-nil`值时，表示发生了未知的`panic`异常，`deferred`函数将调用panic函数并将当前的`panic value`作为参数传入；此时，等同于`recover`没有做任何操作。（请注意：在例子中，对可预期的错误采用了`panic`，这违反了之前的建议，我们在此只是想向读者演示这种机制。）

有些情况下，我们无法恢复。某些致命错误会导致Go在运行时终止程序，如内存不足。

**练习5.19：** 使用`panic`和`recover`编写一个不包含`return`语句但能返回一个非零值的函数。

```go
package main

import (
	"fmt"
)

func weird() (ret string) {
	defer func() {
		recover()
		ret = "hi"
	}()
	panic("omg")
}

func main() {
	fmt.Println(weird())
}
```
