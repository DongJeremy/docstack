# 第13章：错误处理与测试

Go 没有像 Java 和 .NET 那样的 `try/catch` 异常机制：不能执行抛异常操作。但是有一套 `defer-panic-and-recover` 机制（参见 13.2-13.3 节）。

Go 的设计者觉得 `try/catch` 机制的使用太泛滥了，而且从底层向更高的层级抛异常太耗费资源。他们给 Go 设计的机制也可以 “捕捉” 异常，但是更轻量，并且只应该作为（处理错误的）最后的手段。

Go 是怎么处理普通错误的呢？通过在函数和方法中返回错误对象作为它们的唯一或最后一个返回值——如果返回 nil，则没有错误发生——并且主调（calling）函数总是应该检查收到的错误。

**永远不要忽略错误，否则可能会导致程序崩溃！！**

处理错误并且在函数发生错误的地方给用户返回错误信息：照这样处理就算真的出了问题，你的程序也能继续运行并且通知给用户。`panic and recover` 是用来处理真正的异常（无法预测的错误）而不是普通的错误。

库函数通常必须返回某种错误提示给主调（calling）函数。

在前面的章节中我们了解了 Go 检查和报告错误条件的惯有方式：

- 产生错误的函数会返回两个变量，一个值和一个错误码；如果后者是 nil 就是成功，非 nil 就是发生了错误。

- 为了防止发生错误时正在执行的函数（如果有必要的话甚至会是整个程序）被中止，在调用函数后必须检查错误。

下面这段来自 pack1 包的代码 Func1 测试了它的返回值：

```go
if value, err := pack1.Func1(param1); err != nil {
	fmt.Printf("Error %s in pack1.Func1 with parameter %v", err.Error(), param1)
	return    // or: return err
} else {
	// Process(value)
}
```

*为了更清晰的代码，应该总是使用包含错误值变量的 if 复合语句*

上例除了 `fmt.Printf` 还可以使用 log 中对应的方法（参见 13.3 节 和 15.2 节），如果程序中止也没关系的话甚至可以使用 `panic`（参见后面的章节）。

## 13.1 错误处理

Go 有一个预先定义的 error 接口类型

```go
type error interface {
	Error() string
}
```

错误值用来表示异常状态；我们可以在 [5.2 节](05.2.md)中看到它的标准用法。处理文件操作的例子可以在 12 章找到；我们将在 15 章看到网络操作的例子。errors 包中有一个 errorString 结构体实现了 error 接口。当程序处于错误状态时可以用 `os.Exit(1)` 来中止运行。

### 13.1.1 定义错误

任何时候当你需要一个新的错误类型，都可以用 `errors`（必须先 import）包的 `errors.New` 函数接收合适的错误信息来创建，像下面这样：

```go
err := errors.New("math - square root of negative number")
```

在示例 13.1 中你可以看到一个简单的用例：

示例 13.1 [errors.go](examples/chapter_13/errors.go)：

```go
// errors.go
package main

import (
	"errors"
	"fmt"
)

var errNotFound error = errors.New("Not found error")

func main() {
	fmt.Printf("error: %v", errNotFound)
}
// error: Not found error
```

可以把它用于计算平方根函数的参数测试：

```go
func Sqrt(f float64) (float64, error) {
	if f < 0 {
		return 0, errors.New ("math - square root of negative number")
	}
   // implementation of Sqrt
}
```

你可以像下面这样调用 Sqrt 函数：

```go
if f, err := Sqrt(-1); err != nil {
	fmt.Printf("Error: %s\n", err)
}
```

由于 `fmt.Printf` 会自动调用 `String()` 方法 （参见 [10.7 节](10.7.md)），所以错误信息 “Error: math - square root of negative number” 会打印出来。通常（错误信息）都会有像 “Error:” 这样的前缀，所以你的错误信息不要以大写字母开头。

在大部分情况下自定义错误结构类型很有意义的，可以包含除了（低层级的）错误信息以外的其它有用信息，例如，正在进行的操作（打开文件等），全路径或名字。看下面例子中 os.Open 操作触发的 PathError 错误：

```go
// PathError records an error and the operation and file path that caused it.
type PathError struct {
	Op string    // "open", "unlink", etc.
	Path string  // The associated file.
	Err error  // Returned by the system call.
}

func (e *PathError) String() string {
	return e.Op + " " + e.Path + ": "+ e.Err.Error()
}
```

如果有不同错误条件可能发生，那么对实际的错误使用类型断言或类型判断（type-switch）是很有用的，并且可以根据错误场景做一些补救和恢复操作。

```go
//  err != nil
if e, ok := err.(*os.PathError); ok {
	// remedy situation
}
```

或：

```go
switch err := err.(type) {
	case ParseError:
		PrintParseError(err)
	case PathError:
		PrintPathError(err)
	...
	default:
		fmt.Printf("Not a special error, just %s\n", err)
}
```

作为第二个例子考虑用 json 包的情况。当 json.Decode 在解析 JSON 文档发生语法错误时，指定返回一个 SyntaxError 类型的错误：

```go
type SyntaxError struct {
	msg    string // description of error
// error occurred after reading Offset bytes, from which line and columnnr can be obtained
	Offset int64
}

func (e *SyntaxError) String() string { return e.msg }
```

在调用代码中你可以像这样用类型断言测试错误是不是上面的类型：

```go
if serr, ok := err.(*json.SyntaxError); ok {
	line, col := findLine(f, serr.Offset)
	return fmt.Errorf("%s:%d:%d: %v", f.Name(), line, col, err)
}
```

包也可以用额外的方法（methods）定义特定的错误，比如 net.Error：

```go
package net
type Error interface {
	Timeout() bool   // Is the error a timeout?
	Temporary() bool // Is the error temporary?
}
```

在 [15.1 节](15.1.md) 我们可以看到怎么使用它。

正如你所看到的一样，所有的例子都遵循同一种命名规范：错误类型以 “Error” 结尾，错误变量以 “err” 或 “Err” 开头。

syscall 是低阶外部包，用来提供系统基本调用的原始接口。它们返回封装整数类型错误码的syscall.Errno；类型 syscall.Errno 实现了 Error 接口。

大部分 syscall 函数都返回一个结果和可能的错误，比如：

```go
r, err := syscall.Open(name, mode, perm)
if err != nil {
	fmt.Println(err.Error())
}
```

os 包也提供了一套像 os.EINAL 这样的标准错误，它们基于 syscall 错误：

```go
var (
	EPERM		Error = Errno(syscall.EPERM)
	ENOENT		Error = Errno(syscall.ENOENT)
	ESRCH		Error = Errno(syscall.ESRCH)
	EINTR		Error = Errno(syscall.EINTR)
	EIO			Error = Errno(syscall.EIO)
	...
)
```

### 13.1.2 用 fmt 创建错误对象

通常你想要返回包含错误参数的更有信息量的字符串，例如：可以用 `fmt.Errorf()` 来实现：它和 fmt.Printf() 完全一样，接收一个或多个格式占位符的格式化字符串和相应数量的占位变量。和打印信息不同的是它用信息生成错误对象。

比如在前面的平方根例子中使用：

```go
if f < 0 {
	return 0, fmt.Errorf("math: square root of negative number %g", f)
}
```

第二个例子：从命令行读取输入时，如果加了 help 标志，我们可以用有用的信息产生一个错误：

```go
if len(os.Args) > 1 && (os.Args[1] == "-h" || os.Args[1] == "--help") {
	err = fmt.Errorf("usage: %s infile.txt outfile.txt", filepath.Base(os.Args[0]))
	return
}
```

## 13.2 运行时异常和 panic

当发生像数组下标越界或类型断言失败这样的运行错误时，Go 运行时会触发*运行时 panic*，伴随着程序的崩溃抛出一个 `runtime.Error` 接口类型的值。这个错误值有个 `RuntimeError()` 方法用于区别普通错误。

`panic` 可以直接从代码初始化：当错误条件（我们所测试的代码）很严苛且不可恢复，程序不能继续运行时，可以使用 `panic` 函数产生一个中止程序的运行时错误。`panic` 接收一个做任意类型的参数，通常是字符串，在程序死亡时被打印出来。Go 运行时负责中止程序并给出调试信息。在示例 13.2 [panic.go](examples/chapter_13/panic.go) 中阐明了它的工作方式：

```go
package main

import "fmt"

func main() {
	fmt.Println("Starting the program")
	panic("A severe error occurred: stopping the program!")
	fmt.Println("Ending the program")
}
```

输出如下：

```
Starting the program
panic: A severe error occurred: stopping the program!
panic PC=0x4f3038
runtime.panic+0x99 /go/src/pkg/runtime/proc.c:1032
       runtime.panic(0x442938, 0x4f08e8)
main.main+0xa5 E:/Go/GoBoek/code examples/chapter 13/panic.go:8
       main.main()
runtime.mainstart+0xf 386/asm.s:84
       runtime.mainstart()
runtime.goexit /go/src/pkg/runtime/proc.c:148
       runtime.goexit()
---- Error run E:/Go/GoBoek/code examples/chapter 13/panic.exe with code Crashed
---- Program exited with code -1073741783
```

一个检查程序是否被已知用户启动的具体例子：

```go
var user = os.Getenv("USER")

func check() {
	if user == "" {
		panic("Unknown user: no value for $USER")
	}
}
```

可以在导入包的 init() 函数中检查这些。

当发生错误必须中止程序时，`panic` 可以用于错误处理模式：

```go
if err != nil {
	panic("ERROR occurred:" + err.Error())
}
```

<u>Go panicking</u>：

在多层嵌套的函数调用中调用 panic，可以马上中止当前函数的执行，所有的 defer 语句都会保证执行并把控制权交还给接收到 panic 的函数调用者。这样向上冒泡直到最顶层，并执行（每层的） defer，在栈顶处程序崩溃，并在命令行中用传给 panic 的值报告错误情况：这个终止过程就是 *panicking*。

标准库中有许多包含 `Must` 前缀的函数，像 `regexp.MustComplie` 和 `template.Must`；当正则表达式或模板中转入的转换字符串导致错误时，这些函数会 panic。

不能随意地用 panic 中止程序，必须尽力补救错误让程序能继续执行。

## 13.3 从 panic 中恢复（Recover）

正如名字一样，这个（recover）内建函数被用于从 panic 或 错误场景中恢复：让程序可以从 panicking 重新获得控制权，停止终止过程进而恢复正常执行。

`recover` 只能在 defer 修饰的函数（参见 [6.4 节](06.4.md)）中使用：用于取得 panic 调用中传递过来的错误值，如果是正常执行，调用 `recover` 会返回 nil，且没有其它效果。

<u>总结</u>：panic 会导致栈被展开直到 defer 修饰的 recover() 被调用或者程序中止。

下面例子中的 protect 函数调用函数参数 g 来保护调用者防止从 g 中抛出的运行时 panic，并展示 panic 中的信息：

```go
func protect(g func()) {
	defer func() {
		log.Println("done")
		// Println executes normally even if there is a panic
		if err := recover(); err != nil {
			log.Printf("run time panic: %v", err)
		}
	}()
	log.Println("start")
	g() //   possible runtime-error
}
```

这跟 Java 和 .NET 这样的语言中的 catch 块类似。

log 包实现了简单的日志功能：默认的 log 对象向标准错误输出中写入并打印每条日志信息的日期和时间。除了 `Println` 和 `Printf` 函数，其它的致命性函数都会在写完日志信息后调用 os.Exit(1)，那些退出函数也是如此。而 Panic 效果的函数会在写完日志信息后调用 panic；可以在程序必须中止或发生了临界错误时使用它们，就像当 web 服务器不能启动时那样（参见 [15.4 节](15.4.md)中的例子）。

log 包用那些方法（methods）定义了一个 Logger 接口类型，如果你想自定义日志系统的话可以参考（参见 [http://golang.org/pkg/log/#Logger](http://golang.org/pkg/log/#Logger)）。

这是一个展示 panic，defer 和 recover 怎么结合使用的完整例子：

示例 13.3 [panic_recover.go](examples/chapter_13/panic_recover.go)：

```go
// panic_recover.go
package main

import (
	"fmt"
)

func badCall() {
	panic("bad end")
}

func test() {
	defer func() {
		if e := recover(); e != nil {
			fmt.Printf("Panicing %s\r\n", e)
		}
	}()
	badCall()
	fmt.Printf("After bad call\r\n") // <-- wordt niet bereikt
}

func main() {
	fmt.Printf("Calling test\r\n")
	test()
	fmt.Printf("Test completed\r\n")
}
```

输出：

```
Calling test
Panicing bad end
Test completed
```

`defer-panic-recover` 在某种意义上也是一种像 `if`，`for` 这样的控制流机制。

Go 标准库中许多地方都用了这个机制，例如，json 包中的解码和 regexp 包中的 Complie 函数。Go 库的原则是即使在包的内部使用了 panic，在它的对外接口（API）中也必须用 recover 处理成返回显式的错误。

## 13.4 自定义包中的错误处理和 panicking

这是所有自定义包实现者应该遵守的最佳实践：

1）*在包内部，总是应该从 panic 中 recover*：不允许显式的超出包范围的 panic()

2）*向包的调用者返回错误值（而不是 panic）。*

在包内部，特别是在非导出函数中有很深层次的嵌套调用时，将 panic 转换成 error 来告诉调用方为何出错，是很实用的（且提高了代码可读性）。

下面的代码则很好地阐述了这一点。我们有一个简单的 parse 包（示例 13.4）用来把输入的字符串解析为整数切片；这个包有自己特殊的 `ParseError`。

当没有东西需要转换或者转换成整数失败时，这个包会 panic（在函数 fields2numbers 中）。但是可导出的 Parse 函数会从 panic 中 recover 并用所有这些信息返回一个错误给调用者。为了演示这个过程，在 [panic_recover.go](examples/chapter_13/panic_recover.go) 中 调用了 parse 包（示例 13.5）；不可解析的字符串会导致错误并被打印出来。

示例 13.4 [parse.go](examples/chapter_13/parse/parse.go)：

```go
// parse.go
package parse

import (
	"fmt"
	"strings"
	"strconv"
)

// A ParseError indicates an error in converting a word into an integer.
type ParseError struct {
    Index int      // The index into the space-separated list of words.
    Word  string   // The word that generated the parse error.
    Err error // The raw error that precipitated this error, if any.
}

// String returns a human-readable error message.
func (e *ParseError) String() string {
    return fmt.Sprintf("pkg parse: error parsing %q as int", e.Word)
}

// Parse parses the space-separated words in in put as integers.
func Parse(input string) (numbers []int, err error) {
    defer func() {
        if r := recover(); r != nil {
            var ok bool
            err, ok = r.(error)
            if !ok {
                err = fmt.Errorf("pkg: %v", r)
            }
        }
    }()

    fields := strings.Fields(input)
    numbers = fields2numbers(fields)
    return
}

func fields2numbers(fields []string) (numbers []int) {
    if len(fields) == 0 {
        panic("no words to parse")
    }
    for idx, field := range fields {
        num, err := strconv.Atoi(field)
        if err != nil {
            panic(&ParseError{idx, field, err})
        }
        numbers = append(numbers, num)
    }
    return
}
```

示例 13.5 [panic_package.go](examples/chapter_13/panic_package.go)：

```go
// panic_package.go
package main

import (
	"fmt"
	"./parse/parse"
)

func main() {
    var examples = []string{
            "1 2 3 4 5",
            "100 50 25 12.5 6.25",
            "2 + 2 = 4",
            "1st class",
            "",
    }

    for _, ex := range examples {
        fmt.Printf("Parsing %q:\n  ", ex)
        nums, err := parse.Parse(ex)
        if err != nil {
            fmt.Println(err) // here String() method from ParseError is used
            continue
        }
        fmt.Println(nums)
    }
}
```

输出：

```
Parsing "1 2 3 4 5":
  [1 2 3 4 5]
Parsing "100 50 25 12.5 6.25":
  pkg: pkg parse: error parsing "12.5" as int
Parsing "2 + 2 = 4":
  pkg: pkg parse: error parsing "+" as int
Parsing "1st class":
  pkg: pkg parse: error parsing "1st" as int
Parsing "":
  pkg: no words to parse
```

## 13.5 一种用闭包处理错误的模式

每当函数返回时，我们应该检查是否有错误发生：但是这会导致重复乏味的代码。结合 defer/panic/recover 机制和闭包可以得到一个我们马上要讨论的更加优雅的模式。不过这个模式只有当所有的函数都是同一种签名时可用，这样就有相当大的限制。一个很好的使用它的例子是 web 应用，所有的处理函数都是下面这样：

```go
func handler1(w http.ResponseWriter, r *http.Request) { ... }
```

假设所有的函数都有这样的签名：

```go
func f(a type1, b type2)
```

参数的数量和类型是不相关的。

我们给这个类型一个名字：

```go
fType1 = func f(a type1, b type2)
```

在我们的模式中使用了两个帮助函数：

1）check：这是用来检查是否有错误和 panic 发生的函数：

```go
func check(err error) { if err != nil { panic(err) } }
```

2）errorhandler：这是一个包装函数。接收一个 fType1 类型的函数 fn 并返回一个调用 fn 的函数。里面就包含有 defer/recover 机制，这在 [13.3 节](13.3.md)中有相应描述。

```go
func errorHandler(fn fType1) fType1 {
	return func(a type1, b type2) {
		defer func() {
			if err, ok := recover().(error); ok {
				log.Printf("run time panic: %v", err)
			}
		}()
		fn(a, b)
	}
}
```

当错误发生时会 recover 并打印在日志中；除了简单的打印，应用也可以用 template 包（参见 [15.7 节](15.7.md)）为用户生成自定义的输出。check() 函数会在所有的被调函数中调用，像这样：

```go
func f1(a type1, b type2) {
	...
	f, _, err := // call function/method
	check(err)
	t, err := // call function/method
	check(err)
	_, err2 := // call function/method
	check(err2)
	...
}
```

通过这种机制，所有的错误都会被 recover，并且调用函数后的错误检查代码也被简化为调用 check(err) 即可。在这种模式下，不同的错误处理必须对应不同的函数类型；它们（错误处理）可能被隐藏在错误处理包内部。可选的更加通用的方式是用一个空接口类型的切片作为参数和返回值。

我们会在 [15.5 节](15.5.md)的 web 应用中使用这种模式。

<u>练习</u>

**练习 13.1**：[recover_dividebyzero.go](exercises/chapter_13/recover_divbyzero.go)

用示例 13.3 中的编码模式通过整数除以 0 触发一个运行时 panic。

**练习 13.2**：[panic_defer.go](exercises/chapter_13/panic_defer.go)

阅读下面的完整程序。不要执行它，写出程序的输出结果。然后编译执行并验证你的预想。

```go
// panic_defer.go
package main

import "fmt"

func main() {
	f()
	fmt.Println("Returned normally from f.")
}

func f() {
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("Recovered in f", r)
		}
	}()
	fmt.Println("Calling g.")
	g(0)
	fmt.Println("Returned normally from g.")
}

func g(i int) {
	if i > 3 {
		fmt.Println("Panicking!")
		panic(fmt.Sprintf("%v", i))
	}
	defer fmt.Println("Defer in g", i)
	fmt.Println("Printing in g", i)
	g(i + 1)
}
```

输出：

```
Calling g.
Printing in g 0
Printing in g 1
Printing in g 2
Printing in g 3
Panicking!
Defer in g 3
Defer in g 2
Defer in g 1
Defer in g 0
Recovered in f 4
Returned normally from f.
```

**练习 13.3**：[panic_defer_convint.go](exercises/chapter_13/panic_defer_convint.go)

写一个 ConvertInt64ToInt 函数把 int64 值转换为 int 值，如果发生错误（提示：参见 [4.5.2.1 节](04.5.md#4521-整型-int-和浮点型-float)）就 panic。然后在函数 IntFromInt64 中调用这个函数并 recover，返回一个整数和一个错误。请测试这个函数！

## 13.6 启动外部命令和程序

os 包有一个 `StartProcess` 函数可以调用或启动外部系统命令和二进制可执行文件；它的第一个参数是要运行的进程，第二个参数用来传递选项或参数，第三个参数是含有系统环境基本信息的结构体。

这个函数返回被启动进程的 id（pid），或者启动失败返回错误。

exec 包中也有同样功能的更简单的结构体和函数；主要是 `exec.Command(name string, arg ...string)` 和 `Run()`。首先需要用系统命令或可执行文件的名字创建一个 `Command` 对象，然后用这个对象作为接收者调用 `Run()`。下面的程序（因为是执行 Linux 命令，只能在 Linux 下面运行）演示了它们的使用：

示例 13.6 [exec.go](examples/chapter_13/exec.go)：

```go
// exec.go
package main
import (
	"fmt"
    "os/exec"
	"os"
)

func main() {
// 1) os.StartProcess //
/*********************/
/* Linux: */
env := os.Environ()
procAttr := &os.ProcAttr{
			Env: env,
			Files: []*os.File{
				os.Stdin,
				os.Stdout,
				os.Stderr,
			},
		}
// 1st example: list files
pid, err := os.StartProcess("/bin/ls", []string{"ls", "-l"}, procAttr)  
if err != nil {
		fmt.Printf("Error %v starting process!", err)  //
		os.Exit(1)
}
fmt.Printf("The process id is %v", pid)
```

输出：

``` go
The process id is &{2054 0}total 2056
-rwxr-xr-x 1 ivo ivo 1157555 2011-07-04 16:48 Mieken_exec
-rw-r--r-- 1 ivo ivo    2124 2011-07-04 16:48 Mieken_exec.go
-rw-r--r-- 1 ivo ivo   18528 2011-07-04 16:48 Mieken_exec_go_.6
-rwxr-xr-x 1 ivo ivo  913920 2011-06-03 16:13 panic.exe
-rw-r--r-- 1 ivo ivo     180 2011-04-11 20:39 panic.go
```

```go
// 2nd example: show all processes
pid, err = os.StartProcess("/bin/ps", []string{"ps", "-e", "-opid,ppid,comm"}, procAttr)  

if err != nil {
		fmt.Printf("Error %v starting process!", err)  //
		os.Exit(1)
}

fmt.Printf("The process id is %v", pid)
```

```go
// 2) exec.Run //
/***************/
// Linux:  OK, but not for ls ?
// cmd := exec.Command("ls", "-l")  // no error, but doesn't show anything ?
// cmd := exec.Command("ls")  		// no error, but doesn't show anything ?
	cmd := exec.Command("gedit")  // this opens a gedit-window
	err = cmd.Run()
	if err != nil {
		fmt.Printf("Error %v executing command!", err)
		os.Exit(1)
	}
	fmt.Printf("The command is %v", cmd)
// The command is &{/bin/ls [ls -l] []  <nil> <nil> <nil> 0xf840000210 <nil> true [0xf84000ea50 0xf84000e9f0 0xf84000e9c0] [0xf84000ea50 0xf84000e9f0 0xf84000e9c0] [] [] 0xf8400128c0}
}
// in Windows: uitvoering: Error fork/exec /bin/ls: The system cannot find the path specified. starting process!
```

## 13.7 Go 中的单元测试和基准测试

首先所有的包都应该有一定的必要文档，然后同样重要的是对包的测试。

在第 3 章中提到了 Go 的测试工具 gotest， 我们已经在 [9.8 节](09.8.md)中使用过了。这里我们会用更多的例子进行详细说明。

名为 testing 的包被专门用来进行自动化测试，日志和错误报告。并且还包含一些基准测试函数的功能。

<u>备注：</u>gotest 是 Unix bash 脚本，所以在 Windows 下你需要配置 MINGW 环境（参见 [2.5 节](02.5.md)）；在 Windows 环境下把所有的 pkg/linux_amd64 替换成 pkg/windows。

对一个包做（单元）测试，需要写一些可以频繁（每次更新后）执行的小块测试单元来检查代码的正确性。于是我们必须写一些 Go 源文件来测试代码。测试程序必须属于被测试的包，并且文件名满足这种形式 `*_test.go`，所以测试代码和包中的业务代码是分开的。

`_test` 程序不会被普通的 Go 编译器编译，所以当放应用部署到生产环境时它们不会被部署；只有 gotest 会编译所有的程序：普通程序和测试程序。

测试文件中必须导入 "testing" 包，并写一些名字以 `TestZzz` 打头的全局函数，这里的 `Zzz` 是被测试函数的字母描述，如 TestFmtInterface，TestPayEmployees 等。

测试函数必须有这种形式的头部：

```go
func TestAbcde(t *testing.T)
```

T 是传给测试函数的结构类型，用来管理测试状态，支持格式化测试日志，如 t.Log，t.Error，t.ErrorF 等。在函数的结尾把输出跟想要的结果对比，如果不等就打印一个错误。成功的测试则直接返回。

用下面这些函数来通知测试失败：

1）```func (t *T) Fail()```

		标记测试函数为失败，然后继续执行（剩下的测试）。

2）```func (t *T) FailNow()```

		标记测试函数为失败并中止执行；文件中别的测试也被略过，继续执行下一个文件。

3）```func (t *T) Log(args ...interface{})```

		args 被用默认的格式格式化并打印到错误日志中。

4）```func (t *T) Fatal(args ...interface{})```

		结合 先执行 3），然后执行 2）的效果。

运行 go test 来编译测试程序，并执行程序中所有的 TestZZZ 函数。如果所有的测试都通过会打印出 PASS。

gotest 可以接收一个或多个函数程序作为参数，并指定一些选项。

结合 --chatty 或 -v 选项，每个执行的测试函数以及测试状态会被打印。

例如：

```bash
go test fmt_test.go --chatty
=== RUN fmt.TestFlagParser
--- PASS: fmt.TestFlagParser
=== RUN fmt.TestArrayPrinter
--- PASS: fmt.TestArrayPrinter
...
```

testing 包中有一些类型和函数可以用来做简单的基准测试；测试代码中必须包含以 `BenchmarkZzz` 打头的函数并接收一个 `*testing.B` 类型的参数，比如：

```go
func BenchmarkReverse(b *testing.B) {
	...
}
```

命令 ```go test –test.bench=.*``` 会运行所有的基准测试函数；代码中的函数会被调用 N 次（N是非常大的数，如 N = 1000000），并展示 N 的值和函数执行的平均时间，单位为 ns（纳秒，ns/op）。如果是用 testing.Benchmark 调用这些函数，直接运行程序即可。

具体可以参见 [14.16 节](14.16.md)中用 goroutines 运行基准测试的例子以及练习 13.4：[string_reverse_test.go](exercises/chapter_13/string_reverse_test.go)

## 13.8 测试的具体例子

在练习 9.4 中你写了一个叫 [main_oddeven.go](exercises/chapter_9/main_oddeven.go) 的程序用来测试前 100 个整数是否是偶数。这个函数属于 even 包。

下面是一种可能的方案：

示例 13.7 [even_main.go](examples/chapter_13/even/even_main/even_main.go)：

```go
package main

import (
	"fmt"
	"even/even"
)

func main() {
	for i:=0; i<=100; i++ {
		fmt.Printf("Is the integer %d even? %v\n", i, even.Even(i))
	}
}
```

上面使用了 even.go 中的 even 包：

示例 13.8 [even/even.go](examples/chapter_13/even/even/even.go)：

```go
package even

func Even(i int) bool {		// Exported function
	return i%2 == 0
}

func Odd(i int) bool {		// Exported function
	return i%2 != 0
}
```

在 even 包的路径下，我们创建一个名为 oddeven_test.go 的测试程序：

示例 13.9 [even/oddeven_test.go](examples/chapter_13/even/even/oddeven_test.go)：

```go
package even

import "testing"

func TestEven(t *testing.T) {
	if !Even(10) {
		t.Log(" 10 must be even!")
		t.Fail()
	}
	if Even(7) {
		t.Log(" 7 is not even!")
		t.Fail()
	}

}

func TestOdd(t *testing.T) {
	if !Odd(11) {
		t.Log(" 11 must be odd!")
		t.Fail()
	}
	if Odd(10) {
		t.Log(" 10 is not odd!")
		t.Fail()
	}
}
```

由于测试需要具体的输入用例且不可能测试到所有的用例（非常像一个无穷的数），所以我们必须对要使用的测试用例思考再三。

至少应该包括：

- 正常的用例
- 反面的用例（错误的输入，如用负数或字母代替数字，没有输入等）
- 边界检查用例（如果参数的取值范围是 0 到 1000，检查 0 和 1000 的情况）

可以直接执行 go install 安装 even 或者创建一个 以下内容的 Makefile：

```bash
include $(GOROOT)/src/Make.inc
TARG=even
GOFILES=\
       even.go\
include $(GOROOT)/src/Make.pkg
```

然后执行 make（或 gomake）命令来构建归档文件 even.a

测试代码不能在 GOFILES 参数中引用，因为我们不希望生成的程序中有测试代码。如果包含了测试代码，go test 会给出错误提示！go test 会生成一个单独的包含测试代码的 `_test` 程序。

现在我们可以用命令：`go test`（或 `make test`）来测试 even 包。

因为示例 13.5 中的测试函数不会调用 t.Log 和 t.Fail，所以会得到一个 PASS 的结果。在这个简单例子中一切都正常执行。

为了看到失败时的输出，把函数 TestEven 改为：

```go
func TestEven(t *testing.T) {
	if Even(10) {
		t.Log("Everything OK: 10 is even, just a test to see failed output!")
		t.Fail()
 	}
}
```

现在会调用 t.Log 和 t.Fail，得到的结果如下：

```go
--- FAIL: even.TestEven (0.00 seconds)
Everything OK: 10 is even, just a test to see failed output!
FAIL
```

**练习 13.4：**[string_reverse_test.go](exercises/chapter_13/string_reverse_test.go)

为练习 7.14 [string_reverse.go](exercises/chapter_7/string_reverse.go) 写一个单元测试。

把 string_reverse 放到自己的包 strev 中，只包含一个可导出函数 reverse。

实现并测试它。

## 13.9 用（测试数据）表驱动测试

编写测试代码时，一个较好的办法是把测试的输入数据和期望的结果写在一起组成一个数据表：表中的每条记录都是一个含有输入和期望值的完整测试用例，有时还可以结合像测试名字这样的额外信息来让测试输出更多的信息。

实际测试时简单迭代表中的每条记录，并执行必要的测试。这在练习 [13.4](exercises/chapter_13/string_reverse_test.go) 中有具体的应用。

可以抽象为下面的代码段：

```go
var tests = []struct{ 	// Test table
	in  string
	out string

}{
	{"in1", "exp1"},
	{"in2", "exp2"},
	{"in3", "exp3"},
...
}

func TestFunction(t *testing.T) {
	for i, tt := range tests {
		s := FuncToBeTested(tt.in)
		if s != tt.out {
			t.Errorf("%d. %q => %q, wanted: %q", i, tt.in, s, tt.out)
		}
	}
}
```

如果大部分函数都可以写成这种形式，那么写一个帮助函数 verify 对实际测试会很有帮助：

```go
func verify(t *testing.T, testnum int, testcase, input, output, expected string) {
	if expected != output {
		t.Errorf("%d. %s with input = %s: output %s != %s", testnum, testcase, input, output, expected)
	}
}
```

TestFunction 则变为：

```go
func TestFunction(t *testing.T) {
	for i, tt := range tests {
		s := FuncToBeTested(tt.in)
		verify(t, i, “FuncToBeTested: “, tt.in, s, tt.out)
	}
}
```

## 13.10 性能调试：分析并优化 Go 程序

### 13.10.1 时间和内存消耗

可以用这个便捷脚本 *xtime* 来测量：

```sh
#!/bin/sh
/usr/bin/time -f '%Uu %Ss %er %MkB %C' "$@"
```

在 Unix 命令行中像这样使用 ```xtime goprogexec```，这里的 progexec 是一个 Go 可执行程序，这句命令行输出类似：56.63u 0.26s 56.92r 1642640kB progexec，分别对应用户时间，系统时间，实际时间和最大内存占用。

### 13.10.2 用 go test 调试

如果代码使用了 Go 中 testing 包的基准测试功能，我们可以用 gotest 标准的 `-cpuprofile` 和 `-memprofile` 标志向指定文件写入 CPU 或 内存使用情况报告。

使用方式：```go test -x -v -cpuprofile=prof.out -file x_test.go```

编译执行 x_test.go 中的测试，并向 prof.out 文件中写入 cpu 性能分析信息。

### 13.10.3 用 pprof 调试

你可以在单机程序 progexec 中引入 runtime/pprof 包；这个包以 pprof 可视化工具需要的格式写入运行时报告数据。对于 CPU 性能分析来说你需要添加一些代码：

```go
var cpuprofile = flag.String("cpuprofile", "", "write cpu profile to file")

func main() {
	flag.Parse()
	if *cpuprofile != "" {
		f, err := os.Create(*cpuprofile)
		if err != nil {
			log.Fatal(err)
		}
		pprof.StartCPUProfile(f)
		defer pprof.StopCPUProfile()
	}
...
```

代码定义了一个名为 cpuprofile 的 flag，调用 Go flag 库来解析命令行 flag，如果命令行设置了 cpuprofile flag，则开始 CPU 性能分析并把结果重定向到那个文件。（os.Create 用拿到的名字创建了用来写入分析数据的文件）。这个分析程序最后需要在程序退出之前调用 StopCPUProfile 来刷新挂起的写操作到文件中；我们用 defer 来保证这一切会在 main 返回时触发。

现在用这个 flag 运行程序：```progexec -cpuprofile=progexec.prof```

然后可以像这样用 gopprof 工具：```gopprof progexec progexec.prof```

gopprof 程序是 Google pprofC++ 分析器的一个轻微变种；关于此工具更多的信息，参见[https://github.com/gperftools/gperftools](https://github.com/gperftools/gperftools)。

如果开启了 CPU 性能分析，Go 程序会以大约每秒 100 次的频率阻塞，并记录当前执行的 goroutine 栈上的程序计数器样本。

此工具一些有趣的命令：

1）`topN`

用来展示分析结果中最开头的 N 份样本，例如：```top5```
它会展示在程序运行期间调用最频繁的 5 个函数，输出如下：

```
Total: 3099 samples
626 20.2% 20.2% 626 20.2% scanblock
309 10.0% 30.2% 2839 91.6% main.FindLoops
...
```

第 5 列表示函数的调用频度。

2）`web` 或 `web 函数名`

该命令生成一份 SVG 格式的分析数据图表，并在网络浏览器中打开它（还有一个 gv 命令可以生成 PostScript 格式的数据，并在 GhostView 中打开，这个命令需要安装 graphviz）。函数被表示成不同的矩形（被调用越多，矩形越大），箭头指示函数调用链。

3）`list 函数名` 或 `weblist 函数名`

展示对应函数名的代码行列表，第 2 列表示当前行执行消耗的时间，这样就很好地指出了运行过程中消耗最大的代码。

如果发现函数 `runtime.mallocgc`（分配内存并执行周期性的垃圾回收）调用频繁，那么是应该进行内存分析的时候了。找出垃圾回收频繁执行的原因，和内存大量分配的根源。

为了做到这一点必须在合适的地方添加下面的代码：

```go
var memprofile = flag.String("memprofile", "", "write memory profile to this file")
...

CallToFunctionWhichAllocatesLotsOfMemory()
if *memprofile != "" {
	f, err := os.Create(*memprofile)
	if err != nil {
		log.Fatal(err)
	}
	pprof.WriteHeapProfile(f)
	f.Close()
	return
}
```

用 -memprofile flag 运行这个程序：```progexec -memprofile=progexec.mprof```

然后你可以像这样再次使用 gopprof 工具：```gopprof progexec progexec.mprof```

`top5`，`list 函数名` 等命令同样适用，只不过现在是以 Mb 为单位测量内存分配情况，这是 top 命令输出的例子：

```
Total: 118.3 MB
	66.1 55.8% 55.8% 103.7 87.7% main.FindLoops
	30.5 25.8% 81.6% 30.5 25.8% main.*LSG·NewLoop
	...
```

从第 1 列可以看出，最上面的函数占用了最多的内存。

同样有一个报告内存分配计数的有趣工具：

```sh
gopprof --inuse_objects progexec progexec.mprof
```

对于 web 应用来说，有标准的 HTTP 接口可以分析数据。在 HTTP 服务中添加

```go
import _ "http/pprof"
```

会为 /debug/pprof/ 下的一些 URL 安装处理器。然后你可以用一个唯一的参数——你服务中的分析数据的 URL 来执行 gopprof 命令——它会下载并执行在线分析。

```sh
gopprof http://localhost:6060/debug/pprof/profile # 30-second CPU profile
gopprof http://localhost:6060/debug/pprof/heap # heap profile
```

在 Go-blog（引用 15）中有一篇很好的文章用具体的例子进行了分析：分析 Go 程序（2011年6月）。