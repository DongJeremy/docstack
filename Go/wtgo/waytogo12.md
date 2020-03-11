# 第12章：读写数据

除了 `fmt` 和 `os` 包，我们还需要用到 `bufio` 包来处理缓冲的输入和输出。

## 12.1 读取用户的输入

我们如何读取用户的键盘（控制台）输入呢？从键盘和标准输入 `os.Stdin` 读取输入，最简单的办法是使用 `fmt` 包提供的 `Scan` 和 `Sscan` 开头的函数。请看以下程序：

示例 12.1 [readinput1.go](examples/chapter_12/readinput1.go)：

```go
// 从控制台读取输入:
package main
import "fmt"

var (
   firstName, lastName, s string
   i int
   f float32
   input = "56.12 / 5212 / Go"
   format = "%f / %d / %s"
)

func main() {
   fmt.Println("Please enter your full name: ")
   fmt.Scanln(&firstName, &lastName)
   // fmt.Scanf("%s %s", &firstName, &lastName)
   fmt.Printf("Hi %s %s!\n", firstName, lastName) // Hi Chris Naegels
   fmt.Sscanf(input, format, &f, &i, &s)
   fmt.Println("From the string we read: ", f, i, s)
    // 输出结果: From the string we read: 56.12 5212 Go
}
```

`Scanln` 扫描来自标准输入的文本，将空格分隔的值依次存放到后续的参数内，直到碰到换行。`Scanf` 与其类似，除了 `Scanf` 的第一个参数用作格式字符串，用来决定如何读取。`Sscan` 和以 `Sscan` 开头的函数则是从字符串读取，除此之外，与 `Scanf` 相同。如果这些函数读取到的结果与您预想的不同，您可以检查成功读入数据的个数和返回的错误。

您也可以使用 `bufio` 包提供的缓冲读取（`buffered reader`）来读取数据，正如以下例子所示：

示例 12.2 [readinput2.go](examples/chapter_12/readinput2.go)：

```go
package main
import (
    "fmt"
    "bufio"
    "os"
)

var inputReader *bufio.Reader
var input string
var err error

func main() {
    inputReader = bufio.NewReader(os.Stdin)
    fmt.Println("Please enter some input: ")
    input, err = inputReader.ReadString('\n')
    if err == nil {
        fmt.Printf("The input was: %s\n", input)
    }
}
```

`inputReader` 是一个指向 `bufio.Reader` 的指针。`inputReader := bufio.NewReader(os.Stdin)` 这行代码，将会创建一个读取器，并将其与标准输入绑定。

`bufio.NewReader()` 构造函数的签名为：`func NewReader(rd io.Reader) *Reader`

该函数的实参可以是满足 `io.Reader` 接口的任意对象（任意包含有适当的 `Read()` 方法的对象，请参考[章节11.8](11.8.md)），函数返回一个新的带缓冲的 `io.Reader` 对象，它将从指定读取器（例如 `os.Stdin`）读取内容。

返回的读取器对象提供一个方法 `ReadString(delim byte)`，该方法从输入中读取内容，直到碰到 `delim` 指定的字符，然后将读取到的内容连同 `delim` 字符一起放到缓冲区。

`ReadString` 返回读取到的字符串，如果碰到错误则返回 `nil`。如果它一直读到文件结束，则返回读取到的字符串和 `io.EOF`。如果读取过程中没有碰到 `delim` 字符，将返回错误 `err != nil`。

在上面的例子中，我们会读取键盘输入，直到回车键（`\n`）被按下。

屏幕是标准输出 `os.Stdout`；`os.Stderr` 用于显示错误信息，大多数情况下等同于 `os.Stdout`。

一般情况下，我们会省略变量声明，而使用 `:=`，例如：

```go
inputReader := bufio.NewReader(os.Stdin)
input, err := inputReader.ReadString('\n')
```

我们将从现在开始使用这种写法。

第二个例子从键盘读取输入，使用了 `switch` 语句：

示例 12.3 [switch_input.go](examples/chapter_12/switch_input.go)：

```go
package main
import (
    "fmt"
    "os"
    "bufio"
)

func main() {
    inputReader := bufio.NewReader(os.Stdin)
    fmt.Println("Please enter your name:")
    input, err := inputReader.ReadString('\n')

    if err != nil {
        fmt.Println("There were errors reading, exiting program.")
        return
    }

    fmt.Printf("Your name is %s", input)
    // For Unix: test with delimiter "\n", for Windows: test with "\r\n"
    switch input {
    case "Philip\r\n":  fmt.Println("Welcome Philip!")
    case "Chris\r\n":   fmt.Println("Welcome Chris!")
    case "Ivo\r\n":     fmt.Println("Welcome Ivo!")
    default: fmt.Printf("You are not welcome here! Goodbye!")
    }

    // version 2:   
    switch input {
    case "Philip\r\n":  fallthrough
    case "Ivo\r\n":     fallthrough
    case "Chris\r\n":   fmt.Printf("Welcome %s\n", input)
    default: fmt.Printf("You are not welcome here! Goodbye!\n")
    }

    // version 3:
    switch input {
    case "Philip\r\n", "Ivo\r\n":   fmt.Printf("Welcome %s\n", input)
    default: fmt.Printf("You are not welcome here! Goodbye!\n")
    }
}
```

注意：Unix和Windows的行结束符是不同的！

**练习**

**练习 12.1:** [word_letter_count.go](exercises/chapter_12/word_letter_count.go)

编写一个程序，从键盘读取输入。当用户输入 'S' 的时候表示输入结束，这时程序输出 3 个数字：  

i) 输入的字符的个数，包括空格，但不包括 '`\r`' 和 '`\n`'  

ii) 输入的单词的个数  

iii) 输入的行数

**练习 12.2:** [calculator.go](exercises/chapter_12/calculator.go)

编写一个简单的逆波兰式计算器，它接受用户输入的整型数（最大值 999999）和运算符 `+`、`-`、`*`、`/`。  

输入的格式为：`number1 ENTER number2 ENTER operator ENTER` --> 显示结果  

当用户输入字符 'q' 时，程序结束。请使用您在练习11.13中开发的 `stack` 包。

## 12.2 文件读写

### 12.2.1 读文件

在 Go 语言中，文件使用指向 `os.File` 类型的指针来表示的，也叫做文件句柄。我们在前面章节使用到过标准输入 `os.Stdin` 和标准输出 `os.Stdout`，他们的类型都是 `*os.File`。让我们来看看下面这个程序：

示例 12.4 [fileinput.go](examples/chapter_12/fileinput.go)：

```go
package main

import (
    "bufio"
    "fmt"
    "io"
    "os"
)

func main() {
    inputFile, inputError := os.Open("input.dat")
    if inputError != nil {
        fmt.Printf("An error occurred on opening the inputfile\n" +
            "Does the file exist?\n" +
            "Have you got acces to it?\n")
        return // exit the function on error
    }
    defer inputFile.Close()

    inputReader := bufio.NewReader(inputFile)
    for {
        inputString, readerError := inputReader.ReadString('\n')
        fmt.Printf("The input was: %s", inputString)
        if readerError == io.EOF {
            return
        }      
    }
}
```

变量 `inputFile` 是 `*os.File` 类型的。该类型是一个结构，表示一个打开文件的描述符（文件句柄）。然后，使用 `os` 包里的 `Open` 函数来打开一个文件。该函数的参数是文件名，类型为 `string`。在上面的程序中，我们以只读模式打开 `input.dat` 文件。

如果文件不存在或者程序没有足够的权限打开这个文件，Open函数会返回一个错误：`inputFile, inputError = os.Open("input.dat")`。如果文件打开正常，我们就使用 `defer inputFile.Close()` 语句确保在程序退出前关闭该文件。然后，我们使用 `bufio.NewReader` 来获得一个读取器变量。

通过使用 `bufio` 包提供的读取器（写入器也类似），如上面程序所示，我们可以很方便的操作相对高层的 string 对象，而避免了去操作比较底层的字节。

接着，我们在一个无限循环中使用 `ReadString('\n')` 或 `ReadBytes('\n')` 将文件的内容逐行（行结束符 '\n'）读取出来。

**注意：** 在之前的例子中，我们看到，Unix和Linux的行结束符是 \n，而Windows的行结束符是 \r\n。在使用 `ReadString` 和 `ReadBytes` 方法的时候，我们不需要关心操作系统的类型，直接使用 \n 就可以了。另外，我们也可以使用 `ReadLine()` 方法来实现相同的功能。

一旦读取到文件末尾，变量 `readerError` 的值将变成非空（事实上，其值为常量 `io.EOF`），我们就会执行 `return` 语句从而退出循环。

**其他类似函数：**

**1) 将整个文件的内容读到一个字符串里：**

如果您想这么做，可以使用 `io/ioutil` 包里的 `ioutil.ReadFile()` 方法，该方法第一个返回值的类型是 `[]byte`，里面存放读取到的内容，第二个返回值是错误，如果没有错误发生，第二个返回值为 nil。请看示例 12.5。类似的，函数 `WriteFile()` 可以将 `[]byte` 的值写入文件。

示例 12.5 [read_write_file1.go](examples/chapter_12/read_write_file1.go)：

```go
package main
import (
    "fmt"
    "io/ioutil"
    "os"
)

func main() {
    inputFile := "products.txt"
    outputFile := "products_copy.txt"
    buf, err := ioutil.ReadFile(inputFile)
    if err != nil {
        fmt.Fprintf(os.Stderr, "File Error: %s\n", err)
        // panic(err.Error())
    }
    fmt.Printf("%s\n", string(buf))
    err = ioutil.WriteFile(outputFile, buf, 0644) // oct, not hex
    if err != nil {
        panic(err.Error())
    }
}
```

**2) 带缓冲的读取**

在很多情况下，文件的内容是不按行划分的，或者干脆就是一个二进制文件。在这种情况下，`ReadString()`就无法使用了，我们可以使用 `bufio.Reader` 的 `Read()`，它只接收一个参数：

```go
buf := make([]byte, 1024)
...
n, err := inputReader.Read(buf)
if (n == 0) { break}
```

变量 n 的值表示读取到的字节数.

**3) 按列读取文件中的数据**

如果数据是按列排列并用空格分隔的，你可以使用 `fmt` 包提供的以 FScan 开头的一系列函数来读取他们。请看以下程序，我们将 3 列的数据分别读入变量 v1、v2 和 v3 内，然后分别把他们添加到切片的尾部。

示例 12.6 [read_file2.go](examples/chapter_12/read_file2.go)：

```go
package main
import (
    "fmt"
    "os"
)

func main() {
    file, err := os.Open("products2.txt")
    if err != nil {
        panic(err)
    }
    defer file.Close()

    var col1, col2, col3 []string
    for {
        var v1, v2, v3 string
        _, err := fmt.Fscanln(file, &v1, &v2, &v3)
        // scans until newline
        if err != nil {
            break
        }
        col1 = append(col1, v1)
        col2 = append(col2, v2)
        col3 = append(col3, v3)
    }

    fmt.Println(col1)
    fmt.Println(col2)
    fmt.Println(col3)
}
```

输出结果：

```
[ABC FUNC GO]
[40 56 45]
[150 280 356]
```

**注意：** `path` 包里包含一个子包叫 `filepath`，这个子包提供了跨平台的函数，用于处理文件名和路径。例如 Base() 函数用于获得路径中的最后一个元素（不包含后面的分隔符）：

```go
import "path/filepath"
filename := filepath.Base(path)
```

**练习 12.3**：[read_csv.go](exercises/chapter_12/read_csv.go)

文件 products.txt 的内容如下：

```
"The ABC of Go";25.5;1500
"Functional Programming with Go";56;280
"Go for It";45.9;356
"The Go Way";55;500
```
每行的第一个字段为 title，第二个字段为 price，第三个字段为 quantity。内容的格式基本与 示例 12.3c 的相同，除了分隔符改成了分号。请读取出文件的内容，创建一个结构用于存取一行的数据，然后使用结构的切片，并把数据打印出来。

关于解析 CSV 文件，`encoding/csv` 包提供了相应的功能。具体请参考 [http://golang.org/pkg/encoding/csv/](http://golang.org/pkg/encoding/csv/)

### 12.2.2 `compress`包：读取压缩文件

`compress`包提供了读取压缩文件的功能，支持的压缩文件格式为：bzip2、flate、gzip、lzw 和 zlib。

下面的程序展示了如何读取一个 gzip 文件。

示例 12.7 [gzipped.go](examples/chapter_12/gzipped.go)：

```go
package main

import (
	"fmt"
	"bufio"
	"os"
	"compress/gzip"
)

func main() {
	fName := "MyFile.gz"
	var r *bufio.Reader
	fi, err := os.Open(fName)
	if err != nil {
		fmt.Fprintf(os.Stderr, "%v, Can't open %s: error: %s\n", os.Args[0], fName,
			err)
		os.Exit(1)
	}
	defer fi.Close()
	fz, err := gzip.NewReader(fi)
	if err != nil {
		r = bufio.NewReader(fi)
	} else {
		r = bufio.NewReader(fz)
	}

	for {
		line, err := r.ReadString('\n')
		if err != nil {
			fmt.Println("Done reading file")
			os.Exit(0)
		}
		fmt.Println(line)
	}
}
```

### 12.2.3 写文件

请看以下程序：

示例 12.8 [fileoutput.go](examples/chapter_12/fileoutput.go)：

```go
package main

import (
	"os"
	"bufio"
	"fmt"
)

func main () {
	// var outputWriter *bufio.Writer
	// var outputFile *os.File
	// var outputError os.Error
	// var outputString string
	outputFile, outputError := os.OpenFile("output.dat", os.O_WRONLY|os.O_CREATE, 0666)
	if outputError != nil {
		fmt.Printf("An error occurred with file opening or creation\n")
		return  
	}
	defer outputFile.Close()

	outputWriter := bufio.NewWriter(outputFile)
	outputString := "hello world!\n"

	for i:=0; i<10; i++ {
		outputWriter.WriteString(outputString)
	}
	outputWriter.Flush()
}
```

除了文件句柄，我们还需要 `bufio` 的 `Writer`。我们以只写模式打开文件 `output.dat`，如果文件不存在则自动创建：

```go
outputFile, outputError := os.OpenFile("output.dat", os.O_WRONLY|os.O_CREATE, 0666)
```

可以看到，`OpenFile` 函数有三个参数：文件名、一个或多个标志（使用逻辑运算符“|”连接），使用的文件权限。

我们通常会用到以下标志：

- `os.O_RDONLY`：只读  
- `os.O_WRONLY`：只写  
- `os.O_CREATE`：创建：如果指定文件不存在，就创建该文件。  
- `os.O_TRUNC`：截断：如果指定文件已存在，就将该文件的长度截为0。

在读文件的时候，文件的权限是被忽略的，所以在使用 `OpenFile` 时传入的第三个参数可以用0。而在写文件时，不管是 Unix 还是 Windows，都需要使用 0666。

然后，我们创建一个写入器（缓冲区）对象：

```go
outputWriter := bufio.NewWriter(outputFile)
```

接着，使用一个 for 循环，将字符串写入缓冲区，写 10 次：`outputWriter.WriteString(outputString)`

缓冲区的内容紧接着被完全写入文件：`outputWriter.Flush()`

如果写入的东西很简单，我们可以使用 `fmt.Fprintf(outputFile, "Some test data.\n")` 直接将内容写入文件。`fmt` 包里的 F 开头的 Print 函数可以直接写入任何 `io.Writer`，包括文件（请参考[章节12.8](12.8.md))。

程序 `filewrite.go` 展示了不使用 `fmt.FPrintf` 函数，使用其他函数如何写文件：

示例 12.8 [filewrite.go](examples/chapter_12/filewrite.go)：

```go
package main

import "os"

func main() {
	os.Stdout.WriteString("hello, world\n")
	f, _ := os.OpenFile("test", os.O_CREATE|os.O_WRONLY, 0666)
	defer f.Close()
	f.WriteString("hello, world in a file\n")
}
```

使用 `os.Stdout.WriteString("hello, world\n")`，我们可以输出到屏幕。

我们以只写模式创建或打开文件"test"，并且忽略了可能发生的错误：`f, _ := os.OpenFile("test", os.O_CREATE|os.O_WRONLY, 0666)`

我们不使用缓冲区，直接将内容写入文件：`f.WriteString( )`

**练习 12.4**：[wiki_part1.go](exercises/chapter_12/wiki_part1.go)

（这是一个独立的练习，但是同时也是为[章节15.4](15.4.md)做准备）

程序中的数据结构如下，是一个包含以下字段的结构:

```go
type Page struct {
    Title string
    Body  []byte
}
```

请给这个结构编写一个 `save` 方法，将 Title 作为文件名、Body作为文件内容，写入到文本文件中。

再编写一个 `load` 函数，接收的参数是字符串 title，该函数读取出与 title 对应的文本文件。请使用 `*Page` 做为参数，因为这个结构可能相当巨大，我们不想在内存中拷贝它。请使用 `ioutil` 包里的函数（参考章节12.2.1）。

## 12.3 文件拷贝

如何拷贝一个文件到另一个文件？最简单的方式就是使用 io 包：

示例 12.10 [filecopy.go](examples/chapter_12/filecopy.go)：

```go
// filecopy.go
package main

import (
	"fmt"
	"io"
	"os"
)

func main() {
	CopyFile("target.txt", "source.txt")
	fmt.Println("Copy done!")
}

func CopyFile(dstName, srcName string) (written int64, err error) {
	src, err := os.Open(srcName)
	if err != nil {
		return
	}
	defer src.Close()

	dst, err := os.Create(dstName)
	if err != nil {
		return
	}
	defer dst.Close()

	return io.Copy(dst, src)
}
```

注意 `defer` 的使用：当打开dst文件时发生了错误，那么 `defer` 仍然能够确保 `src.Close()` 执行。如果不这么做，src文件会一直保持打开状态并占用资源。

## 12.4 从命令行读取参数

### 12.4.1 os 包

os 包中有一个 string 类型的切片变量 `os.Args`，用来处理一些基本的命令行参数，它在程序启动后读取命令行输入的参数。来看下面的打招呼程序：

示例 12.11 [os_args.go](examples/chapter_12/os_args.go)：

```go
// os_args.go
package main

import (
	"fmt"
	"os"
	"strings"
)

func main() {
	who := "Alice "
	if len(os.Args) > 1 {
		who += strings.Join(os.Args[1:], " ")
	}
	fmt.Println("Good Morning", who)
}
```

我们在 IDE 或编辑器中直接运行这个程序输出：`Good Morning Alice`

我们在命令行运行 `os_args` 或 `./os_args` 会得到同样的结果。

但是我们在命令行加入参数，像这样：`os_args John Bill Marc Luke`，将得到这样的输出：`Good Morning Alice John Bill Marc Luke`

这个命令行参数会放置在切片 `os.Args[]` 中（以空格分隔），从索引1开始（`os.Args[0]` 放的是程序本身的名字，在本例中是 `os_args`）。函数 `strings.Join` 以空格为间隔连接这些参数。

**练习 12.5**：[hello_who.go](exercises/chapter_12/hello_who.go)

写一个"Hello World"的变种程序：把人的名字作为程序命令行执行的一个参数，比如： `hello_who Evan Michael Laura` 那么会输出`Hello Evan Michael Laura`!

### 12.4.2 flag 包

flag 包有一个扩展功能用来解析命令行选项。但是通常被用来替换基本常量，例如，在某些情况下我们希望在命令行给常量一些不一样的值。（参看 19 章的项目）

在 flag 包中有一个 Flag 被定义成一个含有如下字段的结构体：

```go
type Flag struct {
	Name     string // name as it appears on command line
	Usage    string // help message
	Value    Value  // value as set
	DefValue string // default value (as text); for usage message
}
```

下面的程序 `echo.go` 模拟了 Unix 的 echo 功能：

```go
package main

import (
	"flag" // command line option parser
	"os"
)

var NewLine = flag.Bool("n", false, "print newline") // echo -n flag, of type *bool

const (
	Space   = " "
	Newline = "\n"
)

func main() {
	flag.PrintDefaults()
	flag.Parse() // Scans the arg list and sets up flags
	var s string = ""
	for i := 0; i < flag.NArg(); i++ {
		if i > 0 {
			s += " "
			if *NewLine { // -n is parsed, flag becomes true
				s += Newline
			}
		}
		s += flag.Arg(i)
	}
	os.Stdout.WriteString(s)
}
```

`flag.Parse()` 扫描参数列表（或者常量列表）并设置 flag, `flag.Arg(i)` 表示第i个参数。`Parse()` 之后 `flag.Arg(i)` 全部可用，`flag.Arg(0)` 就是第一个真实的 flag，而不是像 `os.Args(0)` 放置程序的名字。

`flag.Narg()` 返回参数的数量。解析后 flag 或常量就可用了。
`flag.Bool()` 定义了一个默认值是 `false` 的 flag：当在命令行出现了第一个参数（这里是 "n"），flag 被设置成 `true`（NewLine 是 `*bool` 类型）。flag 被解引用到 `*NewLine`，所以当值是 `true` 时将添加一个 Newline（"\n"）。

`flag.PrintDefaults()` 打印 flag 的使用帮助信息，本例中打印的是：

```go
-n=false: print newline
```

`flag.VisitAll(fn func(*Flag))` 是另一个有用的功能：按照字典顺序遍历 flag，并且对每个标签调用 fn （参考 15.8 章的例子）

当在命令行（Windows）中执行：`echo.exe A B C`，将输出：`A B C`；执行 `echo.exe -n A B C`，将输出：

```
A
B
C
```

每个字符的输出都新起一行，每次都在输出的数据前面打印使用帮助信息：`-n=false: print newline`。

对于 `flag.Bool` 你可以设置布尔型 flag 来测试你的代码，例如定义一个 flag `processedFlag`:

```go
var processedFlag = flag.Bool("proc", false, "nothing processed yet")
```

在后面用如下代码来测试：

```go
if *processedFlag { // found flag -proc
	r = process()
}
```

要给 flag 定义其它类型，可以使用 `flag.Int()`，`flag.Float64()`，`flag.String()`。

在第 15.8 章你将找到一个具体的例子。

## 12.5 用 buffer 读取文件

在下面的例子中，我们结合使用了缓冲读取文件和命令行 flag 解析这两项技术。如果不加参数，那么你输入什么屏幕就打印什么。

参数被认为是文件名，如果文件存在的话就打印文件内容到屏幕。命令行执行 `cat test` 测试输出。

示例 12.11 [cat.go](examples/chapter_12/cat.go)：

```go
package main

import (
	"bufio"
	"flag"
	"fmt"
	"io"
	"os"
)

func cat(r *bufio.Reader) {
	for {
		buf, err := r.ReadBytes('\n')
		if err == io.EOF {
			break
		}
		fmt.Fprintf(os.Stdout, "%s", buf)
	}
	return
}

func main() {
	flag.Parse()
	if flag.NArg() == 0 {
		cat(bufio.NewReader(os.Stdin))
	}
	for i := 0; i < flag.NArg(); i++ {
		f, err := os.Open(flag.Arg(i))
		if err != nil {
			fmt.Fprintf(os.Stderr, "%s:error reading from %s: %s\n", os.Args[0], flag.Arg(i), err.Error())
			continue
		}
		cat(bufio.NewReader(f))
	}
}
```

在 12.6 章节，我们将看到如何使用缓冲写入。

**练习 12.6**：[cat_numbered.go](exercises/chapter_12/cat_numbered.go)

扩展 cat.go 例子，使用 flag 添加一个选项，目的是为每一行头部加入一个行号。使用 `cat -n test` 测试输出。

## 12.6 用切片读写文件

切片提供了 Go 中处理 I/O 缓冲的标准方式，下面 `cat` 函数的第二版中，在一个切片缓冲内使用无限 for 循环（直到文件尾部 EOF）读取文件，并写入到标准输出（`os.Stdout`）。

```go
func cat(f *os.File) {
	const NBUF = 512
	var buf [NBUF]byte
	for {
		switch nr, err := f.Read(buf[:]); true {
		case nr < 0:
			fmt.Fprintf(os.Stderr, "cat: error reading: %s\n", err.Error())
			os.Exit(1)
		case nr == 0: // EOF
			return
		case nr > 0:
			if nw, ew := os.Stdout.Write(buf[0:nr]); nw != nr {
				fmt.Fprintf(os.Stderr, "cat: error writing: %s\n", ew.Error())
			}
		}
	}
}
```

上面的代码来自于 `cat2.go`，使用了 os 包中的 `os.File` 和 `Read` 方法；`cat2.go` 与 `cat.go` 具有同样的功能。

示例 12.14 [cat2.go](examples/chapter_12/cat2.go)：

```go
package main

import (
	"flag"
	"fmt"
	"os"
)

func cat(f *os.File) {
	const NBUF = 512
	var buf [NBUF]byte
	for {
		switch nr, err := f.Read(buf[:]); true {
		case nr < 0:
			fmt.Fprintf(os.Stderr, "cat: error reading: %s\n", err.Error())
			os.Exit(1)
		case nr == 0: // EOF
			return
		case nr > 0:
			if nw, ew := os.Stdout.Write(buf[0:nr]); nw != nr {
				fmt.Fprintf(os.Stderr, "cat: error writing: %s\n", ew.Error())
			}
		}
	}
}

func main() {
	flag.Parse() // Scans the arg list and sets up flags
	if flag.NArg() == 0 {
		cat(os.Stdin)
	}
	for i := 0; i < flag.NArg(); i++ {
		f, err := os.Open(flag.Arg(i))
		if f == nil {
			fmt.Fprintf(os.Stderr, "cat: can't open %s: error %s\n", flag.Arg(i), err)
			os.Exit(1)
		}
		cat(f)
		f.Close()
	}
}
```

## 12.7 用 defer 关闭文件

`defer` 关键字（参看 6.4）对于在函数结束时关闭打开的文件非常有用，例如下面的代码片段：

```go
func data(name string) string {
	f, _ := os.OpenFile(name, os.O_RDONLY, 0)
	defer f.Close() // idiomatic Go code!
	contents, _ := ioutil.ReadAll(f)
	return string(contents)
}
```

在函数 return 后执行了 `f.Close()`

## 12.8 使用接口的实际例子：fmt.Fprintf

例子程序 `io_interfaces.go` 很好的阐述了 io 包中的接口概念。

示例 12.15 [io_interfaces.go](examples/chapter_12/io_interfaces.go)：

```go
// interfaces being used in the GO-package fmt
package main

import (
	"bufio"
	"fmt"
	"os"
)

func main() {
	// unbuffered
	fmt.Fprintf(os.Stdout, "%s\n", "hello world! - unbuffered")
	// buffered: os.Stdout implements io.Writer
	buf := bufio.NewWriter(os.Stdout)
	// and now so does buf.
	fmt.Fprintf(buf, "%s\n", "hello world! - buffered")
	buf.Flush()
}
```

输出：

```
hello world! - unbuffered
hello world! - buffered
```

下面是 `fmt.Fprintf()` 函数的实际签名

```go
func Fprintf(w io.Writer, format string, a ...interface{}) (n int, err error)
```
其不是写入一个文件，而是写入一个 `io.Writer` 接口类型的变量，下面是 `Writer` 接口在 io 包中的定义：

```go
type Writer interface {
	Write(p []byte) (n int, err error)
}
```

`fmt.Fprintf()` 依据指定的格式向第一个参数内写入字符串，第一个参数必须实现了 `io.Writer` 接口。`Fprintf()` 能够写入任何类型，只要其实现了 `Write` 方法，包括 `os.Stdout`,文件（例如 `os.File`），管道，网络连接，通道等等，同样的也可以使用 `bufio` 包中缓冲写入。`bufio` 包中定义了 `type Writer struct{...}`。

`bufio.Writer` 实现了 `Write` 方法：

```go
func (b *Writer) Write(p []byte) (nn int, err error)
```

它还有一个工厂函数：传给它一个 `io.Writer` 类型的参数，它会返回一个带缓冲的 `bufio.Writer` 类型的 `io.Writer`:

```go
func NewWriter(wr io.Writer) (b *Writer)
```

其适合任何形式的缓冲写入。

在缓冲写入的最后千万不要忘了使用 `Flush()`，否则最后的输出不会被写入。

在 15.2-15.8 章节，我们将使用 `fmt.Fprint` 函数向 `http.ResponseWriter` 写入，其同样实现了 io.Writer 接口。

**练习 12.7**：[remove_3till5char.go](exercises/chapter_12/remove_3till5char.go)

下面的代码有一个输入文件 `goprogram`，然后以每一行为单位读取，从读取的当前行中截取第 3 到第 5 的字节写入另一个文件。然而当你运行这个程序，输出的文件却是个空文件。找出程序逻辑中的 bug，修正它并测试。

```go
package main

import (
	"bufio"
	"fmt"
	"os"
	"io"
)

func main() {
	inputFile, _ := os.Open("goprogram")
	outputFile, _ := os.OpenFile("goprogramT", os.O_WRONLY|os.O_CREATE, 0666)
	defer inputFile.Close()
	defer outputFile.Close()
	inputReader := bufio.NewReader(inputFile)
	outputWriter := bufio.NewWriter(outputFile)
	for {
		inputString, _, readerError := inputReader.ReadLine()
		if readerError == io.EOF {
			fmt.Println("EOF")
			return
		}
		outputString := string(inputString[2:5]) + "\r\n"
		_, err := outputWriter.WriteString(outputString)
		if err != nil {
			fmt.Println(err)
			return
		}
	}
	fmt.Println("Conversion done")
}
```

## 12.9 JSON 数据格式

数据结构要在网络中传输或保存到文件，就必须对其编码和解码；目前存在很多编码格式：JSON，XML，gob，Google 缓冲协议等等。Go 语言支持所有这些编码格式；在后面的章节，我们将讨论前三种格式。

结构可能包含二进制数据，如果将其作为文本打印，那么可读性是很差的。另外结构内部可能包含匿名字段，而不清楚数据的用意。

通过把数据转换成纯文本，使用命名的字段来标注，让其具有可读性。这样的数据格式可以通过网络传输，而且是与平台无关的，任何类型的应用都能够读取和输出，不与操作系统和编程语言的类型相关。

下面是一些术语说明：

- 数据结构 --> 指定格式 = `序列化` 或 `编码`（传输之前）
- 指定格式 --> 数据格式 = `反序列化` 或 `解码`（传输之后）

序列化是在内存中把数据转换成指定格式（data -> string），反之亦然（string -> data structure）

编码也是一样的，只是输出一个数据流（实现了 io.Writer 接口）；解码是从一个数据流（实现了 io.Reader）输出到一个数据结构。

我们都比较熟悉 XML 格式(参阅 [12.10](12.9.md))；但有些时候 JSON（JavaScript Object Notation，参阅 [http://json.org](http://json.org)）被作为首选，主要是由于其格式上非常简洁。通常 JSON 被用于 web 后端和浏览器之间的通讯，但是在其它场景也同样的有用。

这是一个简短的 JSON 片段：

```javascript
{
    "Person": {
        "FirstName": "Laura",
        "LastName": "Lynn"
    }
}
```

尽管 XML 被广泛的应用，但是 JSON 更加简洁、轻量（占用更少的内存、磁盘及网络带宽）和更好的可读性，这也使它越来越受欢迎。

Go 语言的 json 包可以让你在程序中方便的读取和写入 JSON 数据。

我们将在下面的例子里使用 json 包，并使用练习 10.1 [vcard.go](exercises/chapter_10/vcard.go) 中一个简化版本的 Address 和 VCard 结构（为了简单起见，我们忽略了很多错误处理，不过在实际应用中你必须要合理的处理这些错误，参阅 13 章）

示例 12.16 [json.go](examples/chapter_12/json.go)：

```go
// json.go
package main

import (
	"encoding/json"
	"fmt"
	"log"
	"os"
)

type Address struct {
	Type    string
	City    string
	Country string
}

type VCard struct {
	FirstName string
	LastName  string
	Addresses []*Address
	Remark    string
}

func main() {
	pa := &Address{"private", "Aartselaar", "Belgium"}
	wa := &Address{"work", "Boom", "Belgium"}
	vc := VCard{"Jan", "Kersschot", []*Address{pa, wa}, "none"}
	// fmt.Printf("%v: \n", vc) // {Jan Kersschot [0x126d2b80 0x126d2be0] none}:
	// JSON format:
	js, _ := json.Marshal(vc)
	fmt.Printf("JSON format: %s", js)
	// using an encoder:
	file, _ := os.OpenFile("vcard.json", os.O_CREATE|os.O_WRONLY, 0666)
	defer file.Close()
	enc := json.NewEncoder(file)
	err := enc.Encode(vc)
	if err != nil {
		log.Println("Error in encoding json")
	}
}
```

`json.Marshal()` 的函数签名是 `func Marshal(v interface{}) ([]byte, error)`，下面是数据编码后的 JSON 文本（实际上是一个 `[]byte`）：

```javascript
{
    "FirstName": "Jan",
    "LastName": "Kersschot",
    "Addresses": [{
        "Type": "private",
        "City": "Aartselaar",
        "Country": "Belgium"
    }, {
        "Type": "work",
        "City": "Boom",
        "Country": "Belgium"
    }],
    "Remark": "none"
}
```

出于安全考虑，在 web 应用中最好使用 `json.MarshalforHTML()` 函数，其对数据执行HTML转码，所以文本可以被安全地嵌在 HTML `<script>` 标签中。

`json.NewEncoder()` 的函数签名是 `func NewEncoder(w io.Writer) *Encoder`，返回的Encoder类型的指针可调用方法 `Encode(v interface{})`，将数据对象 v 的json编码写入 `io.Writer` w 中。

JSON 与 Go 类型对应如下：

- bool 对应 JSON 的 boolean
- float64 对应 JSON 的 number
- string 对应 JSON 的 string
- nil 对应 JSON 的 null

不是所有的数据都可以编码为 JSON 类型：只有验证通过的数据结构才能被编码：

- JSON 对象只支持字符串类型的 key；要编码一个 Go map 类型，map 必须是 map[string]T（T是 `json` 包中支持的任何类型）
- Channel，复杂类型和函数类型不能被编码
- 不支持循环数据结构；它将引起序列化进入一个无限循环
- 指针可以被编码，实际上是对指针指向的值进行编码（或者指针是 nil）

### 反序列化：

`UnMarshal()` 的函数签名是 `func Unmarshal(data []byte, v interface{}) error` 把 JSON 解码为数据结构。

示例12.16中对 vc 编码后的数据为 `js` ，对其解码时，我们首先创建结构 VCard 用来保存解码的数据：`var v VCard` 并调用 `json.Unmarshal(js, &v)`，解析 []byte 中的 JSON 数据并将结果存入指针 &v 指向的值。

虽然反射能够让 JSON 字段去尝试匹配目标结构字段；但是只有真正匹配上的字段才会填充数据。字段没有匹配不会报错，而是直接忽略掉。

（练习 15.2b [twitter_status_json.go](exercises/chapter_15/twitter_status_json.go) 中用到了 UnMarshal）

### 解码任意的数据：

json 包使用 `map[string]interface{}` 和 `[]interface{}` 储存任意的 JSON 对象和数组；其可以被反序列化为任何的 JSON blob 存储到接口值中。

来看这个 JSON 数据，被存储在变量 b 中：

```go
b := []byte(`{"Name": "Wednesday", "Age": 6, "Parents": ["Gomez", "Morticia"]}`)
```

不用理解这个数据的结构，我们可以直接使用 Unmarshal 把这个数据编码并保存在接口值中：

```go
var f interface{}
err := json.Unmarshal(b, &f)
```

f 指向的值是一个 map，key 是一个字符串，value 是自身存储作为空接口类型的值：

```go
map[string]interface{} {
	"Name": "Wednesday",
	"Age":  6,
	"Parents": []interface{} {
		"Gomez",
		"Morticia",
	},
}
```

要访问这个数据，我们可以使用类型断言

```go
m := f.(map[string]interface{})
```

我们可以通过 for range 语法和 type switch 来访问其实际类型：

```go
for k, v := range m {
	switch vv := v.(type) {
	case string:
		fmt.Println(k, "is string", vv)
	case int:
		fmt.Println(k, "is int", vv)

	case []interface{}:
		fmt.Println(k, "is an array:")
		for i, u := range vv {
			fmt.Println(i, u)
		}
	default:
		fmt.Println(k, "is of a type I don’t know how to handle")
	}
}
```

通过这种方式，你可以处理未知的 JSON 数据，同时可以确保类型安全。

### 解码数据到结构

如果我们事先知道 JSON 数据，我们可以定义一个适当的结构并对 JSON 数据反序列化。下面的例子中，我们将定义：

```go
type FamilyMember struct {
	Name    string
	Age     int
	Parents []string
}

```

并对其反序列化：

```go
var m FamilyMember
err := json.Unmarshal(b, &m)
```

程序实际上是分配了一个新的切片。这是一个典型的反序列化引用类型（指针、切片和 map）的例子。

### 编码和解码流

`json` 包提供 `Decoder` 和 `Encoder` 类型来支持常用 JSON 数据流读写。`NewDecoder` 和 `NewEncoder` 函数分别封装了 `io.Reader` 和 `io.Writer` 接口。

```go
func NewDecoder(r io.Reader) *Decoder
func NewEncoder(w io.Writer) *Encoder
```

要想把 JSON 直接写入文件，可以使用 `json.NewEncoder` 初始化文件（或者任何实现 `io.Writer` 的类型），并调用 `Encode()`；反过来与其对应的是使用 `json.NewDecoder` 和 `Decode()` 函数：

```go
func NewDecoder(r io.Reader) *Decoder
func (dec *Decoder) Decode(v interface{}) error
```

来看下接口是如何对实现进行抽象的：数据结构可以是任何类型，只要其实现了某种接口，目标或源数据要能够被编码就必须实现 io.Writer 或 io.Reader 接口。由于 Go 语言中到处都实现了 Reader 和 Writer，因此 Encoder 和 Decoder 可被应用的场景非常广泛，例如读取或写入 HTTP 连接、websockets 或文件。

## 12.10 XML 数据格式

下面是与 12.9 节 JSON 例子等价的 XML 版本：

```xml
<Person>
    <FirstName>Laura</FirstName>
    <LastName>Lynn</LastName>
</Person>
```

如同 json 包一样，也有 `Marshal()` 和 `UnMarshal()` 从 XML 中编码和解码数据；但这个更通用，可以从文件中读取和写入（或者任何实现了 io.Reader 和 io.Writer 接口的类型）

和 JSON 的方式一样，XML 数据可以序列化为结构，或者从结构反序列化为 XML 数据；这些可以在例子 15.8（twitter_status.go）中看到。

encoding/xml 包实现了一个简单的 XML 解析器（SAX），用来解析 XML 数据内容。下面的例子说明如何使用解析器：

示例 12.17 [xml.go](examples/chapter_12/xml.go)：

```go
// xml.go
package main

import (
	"encoding/xml"
	"fmt"
	"strings"
)

var t, token xml.Token
var err error

func main() {
	input := "<Person><FirstName>Laura</FirstName><LastName>Lynn</LastName></Person>"
	inputReader := strings.NewReader(input)
	p := xml.NewDecoder(inputReader)

	for t, err = p.Token(); err == nil; t, err = p.Token() {
		switch token := t.(type) {
		case xml.StartElement:
			name := token.Name.Local
			fmt.Printf("Token name: %s\n", name)
			for _, attr := range token.Attr {
				attrName := attr.Name.Local
				attrValue := attr.Value
				fmt.Printf("An attribute is: %s %s\n", attrName, attrValue)
				// ...
			}
		case xml.EndElement:
			fmt.Println("End of token")
		case xml.CharData:
			content := string([]byte(token))
			fmt.Printf("This is the content: %v\n", content)
			// ...
		default:
			// ...
		}
	}
}
```

输出：

```
Token name: Person
Token name: FirstName
This is the content: Laura
End of token
Token name: LastName
This is the content: Lynn
End of token
End of token
```

包中定义了若干 XML 标签类型：StartElement，Chardata（这是从开始标签到结束标签之间的实际文本），EndElement，Comment，Directive 或 ProcInst。

包中同样定义了一个结构解析器：`NewParser` 方法持有一个 io.Reader（这里具体类型是 strings.NewReader）并生成一个解析器类型的对象。还有一个 `Token()` 方法返回输入流里的下一个 XML token。在输入流的结尾处，会返回（nil，io.EOF）

XML 文本被循环处理直到 `Token()` 返回一个错误，因为已经到达文件尾部，再没有内容可供处理了。通过一个 type-switch 可以根据一些 XML 标签进一步处理。Chardata 中的内容只是一个 []byte，通过字符串转换让其变得可读性强一些。

## 12.11 用 Gob 传输数据

Gob 是 Go 自己的以二进制形式序列化和反序列化程序数据的格式；可以在 `encoding` 包中找到。这种格式的数据简称为 Gob （即 Go binary 的缩写）。类似于 Python 的 "pickle" 和 Java 的 "Serialization"。

Gob 通常用于远程方法调用（RPCs，参见 [15.9节](15.9.md) 的 rpc 包）参数和结果的传输，以及应用程序和机器之间的数据传输。
它和 JSON 或 XML 有什么不同呢？Gob 特定地用于纯 Go 的环境中，例如，两个用 Go 写的服务之间的通信。这样的话服务可以被实现得更加高效和优化。
Gob 不是可外部定义，语言无关的编码方式。因此它的首选格式是二进制，而不是像 JSON 和 XML 那样的文本格式。
Gob 并不是一种不同于 Go 的语言，而是在编码和解码过程中用到了 Go 的反射。

Gob 文件或流是完全自描述的：里面包含的所有类型都有一个对应的描述，并且总是可以用 Go 解码，而不需要了解文件的内容。

只有可导出的字段会被编码，零值会被忽略。在解码结构体的时候，只有同时匹配名称和可兼容类型的字段才会被解码。当源数据类型增加新字段后，Gob 解码客户端仍然可以以这种方式正常工作：解码客户端会继续识别以前存在的字段。并且还提供了很大的灵活性，比如在发送者看来，整数被编码成没有固定长度的可变长度，而忽略具体的 Go 类型。

假如在发送者这边有一个有结构 T：

```go
type T struct { X, Y, Z int }
var t = T{X: 7, Y: 0, Z: 8}
```

而在接收者这边可以用一个结构体 U 类型的变量 u 来接收这个值：

```go
type U struct { X, Y *int8 }
var u U
```

在接收者中，X 的值是7，Y 的值是0（Y的值并没有从 t 中传递过来，因为它是零值）


和 JSON 的使用方式一样，Gob 使用通用的 `io.Writer` 接口，通过 `NewEncoder()` 函数创建 `Encoder` 对象并调用 `Encode()`；相反的过程使用通用的 `io.Reader` 接口，通过 `NewDecoder()` 函数创建 `Decoder` 对象并调用 `Decode()`。


我们把示例 12.12 的信息写进名为 vcard.gob 的文件作为例子。这会产生一个文本可读数据和二进制数据的混合，当你试着在文本编辑中打开的时候会看到。

在示例 12.18 中你会看到一个编解码，并且以字节缓冲模拟网络传输的简单例子：

示例 12.18 [gob1.go](examples/chapter_12/gob1.go)：

```go
// gob1.go
package main

import (
	"bytes"
	"fmt"
	"encoding/gob"
	"log"
)

type P struct {
	X, Y, Z int
	Name    string
}

type Q struct {
	X, Y *int32
	Name string
}

func main() {
	// Initialize the encoder and decoder.  Normally enc and dec would be      
	// bound to network connections and the encoder and decoder would      
	// run in different processes.      
	var network bytes.Buffer   // Stand-in for a network connection      
	enc := gob.NewEncoder(&network) // Will write to network.      
	dec := gob.NewDecoder(&network)	// Will read from network.      
	// Encode (send) the value.      
	err := enc.Encode(P{3, 4, 5, "Pythagoras"})
	if err != nil {
		log.Fatal("encode error:", err)
	}
	// Decode (receive) the value.      
	var q Q
	err = dec.Decode(&q)
	if err != nil {
		log.Fatal("decode error:", err)
	}
	fmt.Printf("%q: {%d,%d}\n", q.Name, *q.X, *q.Y)
}
// Output:   "Pythagoras": {3,4}
```

示例 12.19 [gob2.go](examples/chapter_12/gob2.go) 编码到文件：

```go
// gob2.go
package main

import (
	"encoding/gob"
	"log"
	"os"
)

type Address struct {
	Type             string
	City             string
	Country          string
}

type VCard struct {
	FirstName	string
	LastName	string
	Addresses	[]*Address
	Remark		string
}

var content	string

func main() {
	pa := &Address{"private", "Aartselaar","Belgium"}
	wa := &Address{"work", "Boom", "Belgium"}
	vc := VCard{"Jan", "Kersschot", []*Address{pa,wa}, "none"}
	// fmt.Printf("%v: \n", vc) // {Jan Kersschot [0x126d2b80 0x126d2be0] none}:
	// using an encoder:
	file, _ := os.OpenFile("vcard.gob", os.O_CREATE|os.O_WRONLY, 0666)
	defer file.Close()
	enc := gob.NewEncoder(file)
	err := enc.Encode(vc)
	if err != nil {
		log.Println("Error in encoding gob")
	}
}
```

**练习 12.8**：[degob.go](exercises/chapter_12/degob.go)：

写一个程序读取 vcard.gob 文件，解码并打印它的内容。

## 12.12 Go 中的密码学

通过网络传输的数据必须加密，以防止被 hacker（黑客）读取或篡改，并且保证发出的数据和收到的数据检验和一致。
鉴于 Go 母公司的业务，我们毫不惊讶地看到 Go 的标准库为该领域提供了超过 30 个的包：

- `hash` 包：实现了 `adler32`、`crc32`、`crc64` 和 `fnv` 校验；
- `crypto` 包：实现了其它的 hash 算法，比如 `md4`、`md5`、`sha1` 等。以及完整地实现了 `aes`、`blowfish`、`rc4`、`rsa`、`xtea` 等加密算法。

下面的示例用 `sha1` 和 `md5` 计算并输出了一些校验值。

示例 12.20 [hash_sha1.go](examples/chapter_12/hash_sha1.go)：

```go
// hash_sha1.go
package main

import (
	"fmt"
	"crypto/sha1"
	"io"
	"log"
)

func main() {
	hasher := sha1.New()
	io.WriteString(hasher, "test")
	b := []byte{}
	fmt.Printf("Result: %x\n", hasher.Sum(b))
	fmt.Printf("Result: %d\n", hasher.Sum(b))
	//
	hasher.Reset()
	data := []byte("We shall overcome!")
	n, err := hasher.Write(data)
	if n!=len(data) || err!=nil {
		log.Printf("Hash write error: %v / %v", n, err)
	}
	checksum := hasher.Sum(b)
	fmt.Printf("Result: %x\n", checksum)
}
```

输出：

```
Result: a94a8fe5ccb19ba61c4c0873d391e987982fbbd3
Result: [169 74 143 229 204 177 155 166 28 76 8 115 211 145 233 135 152 47 187 211]
Result: e2222bfc59850bbb00a722e764a555603bb59b2a
```

通过调用 `sha1.New()` 创建了一个新的 `hash.Hash` 对象，用来计算 SHA1 校验值。`Hash` 类型实际上是一个接口，它实现了 `io.Writer` 接口：

```go
type Hash interface {
	// Write (via the embedded io.Writer interface) adds more data to the running hash.
	// It never returns an error.
	io.Writer

	// Sum appends the current hash to b and returns the resulting slice.
	// It does not change the underlying hash state.
	Sum(b []byte) []byte

	// Reset resets the Hash to its initial state.
	Reset()

	// Size returns the number of bytes Sum will return.
	Size() int

	// BlockSize returns the hash's underlying block size.
	// The Write method must be able to accept any amount
	// of data, but it may operate more efficiently if all writes
	// are a multiple of the block size.
	BlockSize() int
}
```

通过 io.WriteString 或 hasher.Write 将给定的 []byte 附加到当前的 `hash.Hash` 对象中。

**练习 12.9**：[hash_md5.go](exercises/chapter_12/hash_md5.go)：

在示例 12.20 中检验 md5 算法。
