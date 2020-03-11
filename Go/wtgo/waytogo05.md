# 第5章：控制结构

到目前为止，我们看到的 Go 程序都是从 `main()` 函数开始执行，然后按顺序执行该函数体中的代码。但我们经常会需要只有在满足一些特定情况时才执行某些代码，也就是说在代码里进行条件判断。针对这种需求，Go 提供了下面这些条件结构和分支结构：

- `if-else` 结构
- `switch` 结构
- `select` 结构，用于 `channel` 的选择（第 14.4 节）

可以使用迭代或循环结构来重复执行一次或多次某段代码（任务）：

- `for (range)` 结构

一些如 `break` 和 `continue` 这样的关键字可以用于中途改变循环的状态。

此外，你还可以使用 `return` 来结束某个函数的执行，或使用 `goto` 和标签来调整程序的执行位置。

Go 完全省略了 `if`、`switch` 和 `for` 结构中条件语句两侧的括号，相比 Java、C++ 和 C# 中减少了很多视觉混乱的因素，同时也使你的代码更加简洁。

## 5.1 if-else 结构

`if` 是用于测试某个条件（布尔型或逻辑型）的语句，如果该条件成立，则会执行 `if` 后由大括号括起来的代码块，否则就忽略该代码块继续执行后续的代码。

```go
if condition {
	// do something	
}
```

如果存在第二个分支，则可以在上面代码的基础上添加 `else` 关键字以及另一代码块，这个代码块中的代码只有在条件不满足时才会执行。`if` 和 `else` 后的两个代码块是相互独立的分支，只可能执行其中一个。

```go
if condition {
	// do something	
} else {
	// do something	
}
```

如果存在第三个分支，则可以使用下面这种三个独立分支的形式：

```go
if condition1 {
	// do something	
} else if condition2 {
	// do something else	
} else {
	// catch-all or default
}
```

`else-if` 分支的数量是没有限制的，但是为了代码的可读性，还是不要在 `if` 后面加入太多的 `else-if` 结构。如果你必须使用这种形式，则把尽可能先满足的条件放在前面。

即使当代码块之间只有一条语句时，大括号也不可被省略(尽管有些人并不赞成，但这还是符合了软件工程原则的主流做法)。

关键字 `if` 和 `else` 之后的左大括号 `{` 必须和关键字在同一行，如果你使用了 `else-if` 结构，则前段代码块的右大括号 `}` 必须和 `else-if` 关键字在同一行。这两条规则都是被编译器强制规定的。

非法的 Go 代码:

```go
if x{
}
else {	// 无效的
}
```

要注意的是，在你使用 `gofmt` 格式化代码之后，每个分支内的代码都会缩进 4 个或 8 个空格，或者是 1 个 tab，并且右大括号与对应的 `if` 关键字垂直对齐。

在有些情况下，条件语句两侧的括号是可以被省略的；当条件比较复杂时，则可以使用括号让代码更易读。条件允许是符合条件，需使用 `&&`、`||` 或 `!`，你可以使用括号来提升某个表达式的运算优先级，并提高代码的可读性。

一种可能用到条件语句的场景是测试变量的值，在不同的情况执行不同的语句，不过将在第 5.3 节讲到的 `switch` 结构会更适合这种情况。

示例 5.1 [booleans.go](examples/chapter_5/booleans.go)

```go
package main
import "fmt"
func main() {
	bool1 := true
	if bool1 {
		fmt.Printf("The value is true\n")
	} else {
		fmt.Printf("The value is false\n")
	}
}
```

输出：

```go
The value is true
```

> **注意事项** 这里不需要使用 `if bool1 == true` 来判断，因为 `bool1` 本身已经是一个布尔类型的值。

这种做法一般都用在测试 `true` 或者有利条件时，但你也可以使用取反 `!` 来判断值的相反结果，如：`if !bool1` 或者 `if !(condition)`。后者的括号大多数情况下是必须的，如这种情况：`if !(var1 == var2)`。

当 `if` 结构内有 `break`、`continue`、`goto` 或者 `return` 语句时，Go 代码的常见写法是省略 `else` 部分（另见第 5.2 节）。无论满足哪个条件都会返回 `x` 或者 `y` 时，一般使用以下写法：

```go
if condition {
	return x
}
return y
```

> **注意事项** 不要同时在 if-else 结构的两个分支里都使用 return 语句，这将导致编译报错 `function ends without a return statement`（你可以认为这是一个编译器的 Bug 或者特性）。（ **译者注：该问题已经在 Go 1.1 中被修复或者说改进** ）

这里举一些有用的例子：

1. 判断一个字符串是否为空：

   - `if str == "" { ... }`
   - `if len(str) == 0 {...}`	

2. 判断运行 Go 程序的操作系统类型，这可以通过常量 `runtime.GOOS` 来判断(第 2.2 节)。

   ```go
   if runtime.GOOS == "windows" {
   	...
   } else { // Unix-like
   	...
   }
   ```

   这段代码一般被放在 `init()` 函数中执行。这儿还有一段示例来演示如何根据操作系统来决定输入结束的提示：

   ```go
   var prompt = "Enter a digit, e.g. 3 "+ "or %s to quit."
   
   func init() {
   	if runtime.GOOS == "windows" {
   		prompt = fmt.Sprintf(prompt, "Ctrl+Z, Enter")		
   	} else { //Unix-like
   		prompt = fmt.Sprintf(prompt, "Ctrl+D")
   	}
   }
   ```

3. 函数 `Abs()` 用于返回一个整型数字的绝对值:

   ```go
   func Abs(x int) int {
   	if x < 0 {
   		return -x
   	}
   	return x	
   }
   ```

4. `isGreater` 用于比较两个整型数字的大小:

   ```go
   func isGreater(x, y int) bool {
   	if x > y {
   		return true	
   	}
   	return false
   }
   ```

在第四种情况中，`if` 可以包含一个初始化语句（如：给一个变量赋值）。这种写法具有固定的格式（在初始化语句后方必须加上分号）：

```go
if initialization; condition {
	// do something
}
```

例如:

```go
val := 10
if val > max {
	// do something
}
```

你也可以这样写:

```go
if val := 10; val > max {
	// do something
}
```

但要注意的是，使用简短方式 `:=` 声明的变量的作用域只存在于 `if` 结构中（在 `if` 结构的大括号之间，如果使用 `if-else` 结构则在 `else` 代码块中变量也会存在）。如果变量在 `if` 结构之前就已经存在，那么在 `if` 结构中，该变量原来的值会被隐藏。最简单的解决方案就是不要在初始化语句中声明变量（见 5.2 节的例 3 了解更多)。

示例 5.2 [ifelse.go](examples/chapter_5/ifelse.go)

```go
package main

import "fmt"

func main() {
	var first int = 10
	var cond int

	if first <= 0 {
		fmt.Printf("first is less than or equal to 0\n")
	} else if first > 0 && first < 5 {
		fmt.Printf("first is between 0 and 5\n")
	} else {
		fmt.Printf("first is 5 or greater\n")
	}
	if cond = 5; cond > 10 {
		fmt.Printf("cond is greater than 10\n")
	} else {
		fmt.Printf("cond is not greater than 10\n")
	}
}
```

输出：

```go
first is 5 or greater
cond is not greater than 10
```

下面的代码片段展示了如何通过在初始化语句中获取函数 `process()` 的返回值，并在条件语句中作为判定条件来决定是否执行 if 结构中的代码：

```go
if value := process(data); value > max {
	...
}
```

## 5.2 测试多返回值函数的错误

Go 语言的函数经常使用两个返回值来表示执行是否成功：返回某个值以及 `true` 表示成功；返回零值（或 `nil`）和 `false` 表示失败（第 4.4 节）。当不使用 `true` 或 `false` 的时候，也可以使用一个 `error` 类型的变量来代替作为第二个返回值：成功执行的话，`error` 的值为 `nil`，否则就会包含相应的错误信息（Go 语言中的错误类型为 error: `var err error`，我们将会在第 13 章进行更多地讨论）。这样一来，就很明显需要用一个 `if` 语句来测试执行结果；由于其符号的原因，这样的形式又称之为 `comma,ok` 模式（pattern）。

在第 4.7 节的程序 `string_conversion.go` 中，函数 `strconv.Atoi` 的作用是将一个字符串转换为一个整数。之前我们忽略了相关的错误检查：

```go
anInt, _ = strconv.Atoi(origStr)
```

如果 `origStr` 不能被转换为整数，`anInt` 的值会变成 0 而 `_` 无视了错误，程序会继续运行。

这样做是非常不好的：程序应该在最接近的位置检查所有相关的错误，至少需要暗示用户有错误发生并对函数进行返回，甚至中断程序。

我们在第二个版本中对代码进行了改进：

示例 5.3 [string_conversion2.go](examples/chapter_5/string_conversion2.go)

```go
package main

import (
	"fmt"
	"strconv"
)

func main() {
	var orig string = "ABC"
	// var an int
	var newS string
	// var err error

	fmt.Printf("The size of ints is: %d\n", strconv.IntSize)	  
	// anInt, err = strconv.Atoi(origStr)
	an, err := strconv.Atoi(orig)
	if err != nil {
		fmt.Printf("orig %s is not an integer - exiting with error\n", orig)
		return
	} 
	fmt.Printf("The integer is %d\n", an)
	an = an + 5
	newS = strconv.Itoa(an)
	fmt.Printf("The new string is: %s\n", newS)
}
```

这是测试 `err` 变量是否包含一个真正的错误（`if err != nil`）的习惯用法。如果确实存在错误，则会打印相应的错误信息然后通过 `return` 提前结束函数的执行。我们还可以使用携带返回值的 `return` 形式，例如 `return err`。这样一来，函数的调用者就可以检查函数执行过程中是否存在错误了。

**习惯用法**

```go
value, err := pack1.Function1(param1)
if err != nil {
	fmt.Printf("An error occured in pack1.Function1 with parameter %v", param1)
	return err
}
// 未发生错误，继续执行：
```

由于本例的函数调用者属于 `main` 函数，所以程序会直接停止运行。

如果我们想要在错误发生的同时终止程序的运行，我们可以使用 `os` 包的 `Exit` 函数：

**习惯用法**

```go
if err != nil {
	fmt.Printf("Program stopping with error %v", err)
	os.Exit(1)
}
```

（此处的退出代码 1 可以使用外部脚本获取到）

有时候，你会发现这种习惯用法被连续重复地使用在某段代码中。

当没有错误发生时，代码继续运行就是唯一要做的事情，所以 if 语句块后面不需要使用 else 分支。

示例 2：我们尝试通过 `os.Open` 方法打开一个名为 `name` 的只读文件：

```go
f, err := os.Open(name)
if err != nil {
	return err
}
doSomething(f) // 当没有错误发生时，文件对象被传入到某个函数中
doSomething
```

**练习 5.1** 尝试改写 [string_conversion2.go](examples/chapter_5/string_conversion2.go) 中的代码，要求使用 `:=` 方法来对 err 进行赋值，哪些地方可以被修改？

示例 3：可以将错误的获取放置在 if 语句的初始化部分：

**习惯用法**

```go
if err := file.Chmod(0664); err != nil {
	fmt.Println(err)
	return err
}
```

示例 4：或者将 ok-pattern 的获取放置在 if 语句的初始化部分，然后进行判断：

**习惯用法**

```go
if value, ok := readData(); ok {
…
}
```

**注意事项**

如果您像下面一样，没有为多返回值的函数准备足够的变量来存放结果：

```go
func mySqrt(f float64) (v float64, ok bool) {
	if f < 0 { return } // error case
	return math.Sqrt(f),true
}

func main() {
	t := mySqrt(25.0)
	fmt.Println(t)
}
```

您会得到一个编译错误：`multiple-value mySqrt() in single-value context`。

正确的做法是：

```go
t, ok := mySqrt(25.0)
if ok { fmt.Println(t) }
```

**注意事项 2**

当您将字符串转换为整数时，且确定转换一定能够成功时，可以将 `Atoi` 函数进行一层忽略错误的封装：

```go
func atoi (s string) (n int) {
	n, _ = strconv.Atoi(s)
	return
}
```

实际上，`fmt` 包（第 4.4.3 节）最简单的打印函数也有 2 个返回值：

```go
count, err := fmt.Println(x) // number of bytes printed, nil or 0, error
```

当打印到控制台时，可以将该函数返回的错误忽略；但当输出到文件流、网络流等具有不确定因素的输出对象时，应该始终检查是否有错误发生（另见练习 6.1b）。

## 5.3 switch 结构

相比较 C 和 Java 等其它语言而言，Go 语言中的 switch 结构使用上更加灵活。它接受任意形式的表达式：

```go
switch var1 {
	case val1:
		...
	case val2:
		...
	default:
		...
}
```

变量 var1 可以是任何类型，而 val1 和 val2 则可以是同类型的任意值。类型不被局限于常量或整数，但必须是相同的类型；或者最终结果为相同类型的表达式。前花括号 `{` 必须和 switch 关键字在同一行。

您可以同时测试多个可能符合条件的值，使用逗号分割它们，例如：`case val1, val2, val3`。

每一个 `case` 分支都是唯一的，从上至下逐一测试，直到匹配为止。（ Go 语言使用快速的查找算法来测试 switch 条件与 case 分支的匹配情况，直到算法匹配到某个 case 或者进入 default 条件为止。）

一旦成功地匹配到某个分支，在执行完相应代码后就会退出整个 switch 代码块，也就是说您不需要特别使用 `break` 语句来表示结束。

因此，程序也不会自动地去执行下一个分支的代码。如果在执行完每个分支的代码后，还希望继续执行后续分支的代码，可以使用 `fallthrough` 关键字来达到目的。

因此：

```go
switch i {
	case 0: // 空分支，只有当 i == 0 时才会进入分支
	case 1:
		f() // 当 i == 0 时函数不会被调用
}
```

并且：

```go
switch i {
	case 0: fallthrough
	case 1:
		f() // 当 i == 0 时函数也会被调用
}
```

在 `case ...:` 语句之后，您不需要使用花括号将多行语句括起来，但您可以在分支中进行任意形式的编码。当代码块只有一行时，可以直接放置在 `case` 语句之后。

您同样可以使用 `return` 语句来提前结束代码块的执行。当您在 switch 语句块中使用 `return` 语句，并且您的函数是有返回值的，您还需要在 switch 之后添加相应的 `return` 语句以确保函数始终会返回。

可选的 `default` 分支可以出现在任何顺序，但最好将它放在最后。它的作用类似与 `if-else` 语句中的 `else`，表示不符合任何已给出条件时，执行相关语句。

示例 5.4 [switch1.go](examples/chapter_5/switch1.go)：

```go
package main

import "fmt"

func main() {
	var num1 int = 100

	switch num1 {
	case 98, 99:
		fmt.Println("It's equal to 98")
	case 100: 
		fmt.Println("It's equal to 100")
	default:
		fmt.Println("It's not equal to 98 or 100")
	}
}

```

输出：

	It's equal to 100

在第 12.1 节，我们会使用 switch 语句判断从键盘输入的字符（详见第 12.2 节的 switch.go）。switch 语句的第二种形式是不提供任何被判断的值（实际上默认为判断是否为 true），然后在每个 case 分支中进行测试不同的条件。当任一分支的测试结果为 true 时，该分支的代码会被执行。这看起来非常像链式的 `if-else` 语句，但是在测试条件非常多的情况下，提供了可读性更好的书写方式。

```go
switch {
	case condition1:
		...
	case condition2:
		...
	default:
		...
}
```

例如：

```go
switch {
	case i < 0:
		f1()
	case i == 0:
		f2()
	case i > 0:
		f3()
}
```

任何支持进行相等判断的类型都可以作为测试表达式的条件，包括 int、string、指针等。

示例 5.4 [switch2.go](examples/chapter_5/switch2.go)：

```go
package main

import "fmt"

func main() {
	var num1 int = 7

	switch {
	    case num1 < 0:
		    fmt.Println("Number is negative")
	    case num1 > 0 && num1 < 10:
		    fmt.Println("Number is between 0 and 10")
	    default:
		    fmt.Println("Number is 10 or greater")
	}
}
```

输出：

	Number is between 0 and 10

switch 语句的第三种形式是包含一个初始化语句：

```go
switch initialization {
	case val1:
		...
	case val2:
		...
	default:
		...
}
```

这种形式可以非常优雅地进行条件判断：

```go
switch result := calculate() {
	case result < 0:
		...
	case result > 0:
		...
	default:
		// 0
}
```

在下面这个代码片段中，变量 a 和 b 被平行初始化，然后作为判断条件：

```go
switch a, b := x[i], y[j] {
	case a < b: t = -1
	case a == b: t = 0
	case a > b: t = 1
}
```

switch 语句还可以被用于 type-switch（详见第 11.4 节）来判断某个 interface 变量中实际存储的变量类型。

**问题 5.1：**

请说出下面代码片段输出的结果：

```go
	k := 6
	switch k {
	case 4:
		fmt.Println("was <= 4")
		fallthrough
	case 5:
		fmt.Println("was <= 5")
		fallthrough
	case 6:
		fmt.Println("was <= 6")
		fallthrough
	case 7:
		fmt.Println("was <= 7")
		fallthrough
	case 8:
		fmt.Println("was <= 8")
		fallthrough
	default:
		fmt.Println("default case")
	}
```

**练习 5.2：** [season.go](exercises/chapter_5/season.go)：

写一个 Season 函数，要求接受一个代表月份的数字，然后返回所代表月份所在季节的名称（不用考虑月份的日期）。

## 5.4 for 结构

如果想要重复执行某些语句，Go 语言中您只有 for 结构可以使用。不要小看它，这个 for 结构比其它语言中的更为灵活。

**注意事项** 其它许多语言中也没有发现和 do while 完全对等的 for 结构，可能是因为这种需求并不是那么强烈。

### 5.4.1 基于计数器的迭代

文件 for1.go 中演示了最简单的基于计数器的迭代，基本形式为：

	for 初始化语句; 条件语句; 修饰语句 {}

示例 5.6 [for1.go](examples/chapter_5/for1.go)：

```go
package main

import "fmt"

func main() {
	for i := 0; i < 5; i++ {
		fmt.Printf("This is the %d iteration\n", i)
	}
}
```

输出：

	This is the 0 iteration
	This is the 1 iteration
	This is the 2 iteration
	This is the 3 iteration
	This is the 4 iteration

由花括号括起来的代码块会被重复执行已知次数，该次数是根据计数器（此例为 i）决定的。循环开始前，会执行且仅会执行一次初始化语句 `i := 0;`；这比在循环之前声明更为简短。紧接着的是条件语句 `i < 5;`，在每次循环开始前都会进行判断，一旦判断结果为 false，则退出循环体。最后一部分为修饰语句 `i++`，一般用于增加或减少计数器。

这三部分组成的循环的头部，它们之间使用分号 `;` 相隔，但并不需要括号 `()` 将它们括起来。例如：`for (i = 0; i < 10; i++) { }`，这是无效的代码！

同样的，左花括号 `{` 必须和 for 语句在同一行，计数器的生命周期在遇到右花括号 `}` 时便终止。一般习惯使用 i、j、z 或 ix 等较短的名称命名计数器。

特别注意，永远不要在循环体内修改计数器，这在任何语言中都是非常差的实践！

您还可以在循环中同时使用多个计数器：

```go
for i, j := 0, N; i < j; i, j = i+1, j-1 {}
```

这得益于 Go 语言具有的平行赋值的特性（可以查看第 7 章 string_reverse.go 中反转数组的示例）。

您可以将两个 for 循环嵌套起来：

```go
for i:=0; i<5; i++ {
	for j:=0; j<10; j++ {
		println(j)
	}
}
```

如果您使用 for 循环迭代一个 Unicode 编码的字符串，会发生什么？

示例 5.7 [for_string.go](examples/chapter_5/for_string.go)：

```go
package main

import "fmt"

func main() {
	str := "Go is a beautiful language!"
	fmt.Printf("The length of str is: %d\n", len(str))
	for ix :=0; ix < len(str); ix++ {
		fmt.Printf("Character on position %d is: %c \n", ix, str[ix])
	}
	str2 := "日本語"
	fmt.Printf("The length of str2 is: %d\n", len(str2))
	for ix :=0; ix < len(str2); ix++ {
		fmt.Printf("Character on position %d is: %c \n", ix, str2[ix])
	}
}
```

输出：

	The length of str is: 27
	Character on position 0 is: G 
	Character on position 1 is: o 
	Character on position 2 is:   
	Character on position 3 is: i 
	Character on position 4 is: s 
	Character on position 5 is:   
	Character on position 6 is: a 
	Character on position 7 is:   
	Character on position 8 is: b 
	Character on position 9 is: e 
	Character on position 10 is: a 
	Character on position 11 is: u 
	Character on position 12 is: t 
	Character on position 13 is: i 
	Character on position 14 is: f 
	Character on position 15 is: u 
	Character on position 16 is: l 
	Character on position 17 is:   
	Character on position 18 is: l 
	Character on position 19 is: a 
	Character on position 20 is: n 
	Character on position 21 is: g 
	Character on position 22 is: u 
	Character on position 23 is: a 
	Character on position 24 is: g 
	Character on position 25 is: e 
	Character on position 26 is: ! 
	The length of str2 is: 9
	Character on position 0 is: æ 
	Character on position 1 is:  
	Character on position 2 is: ¥ 
	Character on position 3 is: æ 
	Character on position 4 is:  
	Character on position 5 is: ¬ 
	Character on position 6 is: è 
	Character on position 7 is: ª 
	Character on position 8 is:  

如果我们打印 str 和 str2 的长度，会分别得到 27 和 9。

由此我们可以发现，ASCII 编码的字符占用 1 个字节，既每个索引都指向不同的字符，而非 ASCII 编码的字符（占有 2 到 4 个字节）不能单纯地使用索引来判断是否为同一个字符。我们会在第 5.4.4 节解决这个问题。

### 练习题

**练习 5.4** [for_loop.go](exercises/chapter_5/for_loop.go)

1. 使用 `for` 结构创建一个简单的循环。要求循环 15 次然后使用 `fmt` 包来打印计数器的值。
2. 使用 `goto` 语句重写循环，要求不能使用 `for` 关键字。

**练习 5.5** [for_character.go](exercises/chapter_5/for_character.go)

创建一个程序，要求能够打印类似下面的结果（尾行达 25 个字符为止）：

	G
	GG
	GGG
	GGGG
	GGGGG
	GGGGGG

1. 使用 2 层嵌套 for 循环。
2. 仅用 1 层 for 循环以及字符串连接。

**练习 5.6** [bitwise_complement.go](exercises/chapter_5/bitwise_complement.go)

使用按位补码从 0 到 10，使用位表达式 `%b` 来格式化输出。

**练习 5.7** Fizz-Buzz 问题：[fizzbuzz.go](exercises/chapter_5/fizzbuzz.go)

写一个从 1 打印到 100 的程序，但是每当遇到 3 的倍数时，不打印相应的数字，但打印一次 "Fizz"。遇到 5 的倍数时，打印 `Buzz` 而不是相应的数字。对于同时为 3 和 5 的倍数的数，打印 `FizzBuzz`（提示：使用 switch 语句）。

**练习 5.8** [rectangle_stars.go](exercises/chapter_5/rectangle_stars.go)

使用 `*` 符号打印宽为 20，高为 10 的矩形。

### 5.4.2 基于条件判断的迭代

for 结构的第二种形式是没有头部的条件判断迭代（类似其它语言中的 while 循环），基本形式为：`for 条件语句 {}`。

您也可以认为这是没有初始化语句和修饰语句的 for 结构，因此 `;;` 便是多余的了。

Listing 5.8 [for2.go](examples/chapter_5/for2.go)：

```go
package main

import "fmt"

func main() {
	var i int = 5

	for i >= 0 {
		i = i - 1
		fmt.Printf("The variable i is now: %d\n", i)
	}
}
```

输出：

	The variable i is now: 4
	The variable i is now: 3
	The variable i is now: 2
	The variable i is now: 1
	The variable i is now: 0
	The variable i is now: -1

### 5.4.3 无限循环

条件语句是可以被省略的，如 `i:=0; ; i++` 或 `for { }` 或 `for ;; { }`（`;;` 会在使用 gofmt 时被移除）：这些循环的本质就是无限循环。最后一个形式也可以被改写为 `for true { }`，但一般情况下都会直接写 `for { }`。

如果 `for` 循环的头部没有条件语句，那么就会认为条件永远为 `true`，因此循环体内必须有相关的条件判断以确保会在某个时刻退出循环。

想要直接退出循环体，可以使用 `break` 语句（第 5.5 节）或 `return` 语句直接返回（第 6.1 节）。

但这两者之间有所区别，`break` 只是退出当前的循环体，而 `return` 语句提前对函数进行返回，不会执行后续的代码。

无限循环的经典应用是服务器，用于不断等待和接受新的请求。

```go
for t, err = p.Token(); err == nil; t, err = p.Token() {
	...
}
```

### 5.4.4 for-range 结构

这是 Go 特有的一种的迭代结构，您会发现它在许多情况下都非常有用。它可以迭代任何一个集合（包括数组和 `map`，详见第 7 和 8 章）。语法上很类似其它语言中 `foreach` 语句，但您依旧可以获得每次迭代所对应的索引。一般形式为：`for ix, val := range coll { }`。

要注意的是，`val` 始终为集合中对应索引的值拷贝，因此它一般只具有只读性质，对它所做的任何修改都不会影响到集合中原有的值（**译者注：如果 `val` 为指针，则会产生指针的拷贝，依旧可以修改集合中的原值**）。一个字符串是 Unicode 编码的字符（或称之为 `rune`）集合，因此您也可以用它迭代字符串：

```go
for pos, char := range str {
...
}
```

每个 `rune` 字符和索引在 `for-range` 循环中是一一对应的。它能够自动根据 UTF-8 规则识别 Unicode 编码的字符。

示例 5.9 [range_string.go](examples/chapter_5/range_string.go)：

```go
package main

import "fmt"

func main() {
	str := "Go is a beautiful language!"
	fmt.Printf("The length of str is: %d\n", len(str))
	for pos, char := range str {
		fmt.Printf("Character on position %d is: %c \n", pos, char)
	}
	fmt.Println()
	str2 := "Chinese: 日本語"
	fmt.Printf("The length of str2 is: %d\n", len(str2))
	for pos, char := range str2 {
    	fmt.Printf("character %c starts at byte position %d\n", char, pos)
	}
	fmt.Println()
	fmt.Println("index int(rune) rune    char bytes")
	for index, rune := range str2 {
    	fmt.Printf("%-2d      %d      %U '%c' % X\n", index, rune, rune, rune, []byte(string(rune)))
	}
}
```

输出：

```
The length of str is: 27
Character on position 0 is: G 
Character on position 1 is: o 
Character on position 2 is:   
Character on position 3 is: i 
Character on position 4 is: s 
Character on position 5 is:   
Character on position 6 is: a 
Character on position 7 is:   
Character on position 8 is: b 
Character on position 9 is: e 
Character on position 10 is: a 
Character on position 11 is: u 
Character on position 12 is: t 
Character on position 13 is: i 
Character on position 14 is: f 
Character on position 15 is: u 
Character on position 16 is: l 
Character on position 17 is:   
Character on position 18 is: l 
Character on position 19 is: a 
Character on position 20 is: n 
Character on position 21 is: g 
Character on position 22 is: u 
Character on position 23 is: a 
Character on position 24 is: g 
Character on position 25 is: e 
Character on position 26 is: ! 

The length of str2 is: 18
character C starts at byte position 0
character h starts at byte position 1
character i starts at byte position 2
character n starts at byte position 3
character e starts at byte position 4
character s starts at byte position 5
character e starts at byte position 6
character : starts at byte position 7
character   starts at byte position 8
character 日 starts at byte position 9
character 本 starts at byte position 12
character 語 starts at byte position 15

index int(rune) rune    char bytes
0       67      U+0043 'C' 43
1       104      U+0068 'h' 68
2       105      U+0069 'i' 69
3       110      U+006E 'n' 6E
4       101      U+0065 'e' 65
5       115      U+0073 's' 73
6       101      U+0065 'e' 65
7       58      U+003A ':' 3A
8       32      U+0020 ' ' 20
9       26085      U+65E5 '日' E6 97 A5
12      26412      U+672C '本' E6 9C AC
15      35486      U+8A9E '語' E8 AA 9E
```

请将输出结果和 Listing 5.7（for_string.go）进行对比。

我们可以看到，常用英文字符使用 1 个字节表示，而汉字（**译者注：严格来说，“Chinese: 日本語”的Chinese应该是Japanese**）使用 3 个字符表示。

**练习 5.9** 以下程序的输出结果是什么？

```go
for i := 0; i < 5; i++ {
	var v int
	fmt.Printf("%d ", v)
	v = 5
}
```

**问题 5.2：** 请描述以下 for 循环的输出结果：

1.

```go
for i := 0; ; i++ {
	fmt.Println("Value of i is now:", i)
}
```

2.

```go
for i := 0; i < 3; {
	fmt.Println("Value of i:", i)
}
```

3.

```go
s := ""
for ; s != "aaaaa"; {
	fmt.Println("Value of s:", s)
	s = s + "a"
}
```

4.

```go
for i, j, s := 0, 5, "a"; i < 3 && j < 100 && s != "aaaaa"; i, j,
	s = i+1, j+1, s + "a" {
	fmt.Println("Value of i, j, s:", i, j, s)
}
```

## 5.5 Break 与 continue

您可以使用 break 语句重写 for2.go 的代码：

示例 5.10 [for3.go](examples/chapter_5/for3.go)：

```go
for {
	i = i - 1
	fmt.Printf("The variable i is now: %d\n", i)
	if i < 0 {
		break
	}
}
```

因此每次迭代都会对条件进行检查（i < 0），以此判断是否需要停止循环。如果退出条件满足，则使用 break 语句退出循环。

一个 break 的作用范围为该语句出现后的最内部的结构，它可以被用于任何形式的 for 循环（计数器、条件判断等）。但在 switch 或 select 语句中（详见第 13 章），break 语句的作用结果是跳过整个代码块，执行后续的代码。

下面的示例中包含了嵌套的循环体（for4.go），break 只会退出最内层的循环：

示例 5.11 [for4.go](examples/chapter_5/for4.go)：

```go
package main

func main() {
	for i:=0; i<3; i++ {
		for j:=0; j<10; j++ {
			if j>5 {
			    break   
			}
			print(j)
		}
		print("  ")
	}
}
```

输出：

	012345 012345 012345

关键字 continue 忽略剩余的循环体而直接进入下一次循环的过程，但不是无条件执行下一次循环，执行之前依旧需要满足循环的判断条件。

示例 5.12 [for5.go](examples/chapter_5/for5.go)：

```go
package main

func main() {
	for i := 0; i < 10; i++ {
		if i == 5 {
			continue
		}
		print(i)
		print(" ")
	}
}
```

输出：

```
0 1 2 3 4 6 7 8 9
```

显然，5 被跳过了。

另外，关键字 continue 只能被用于 for 循环中。

## 5.6 标签与 goto

for、switch 或 select 语句都可以配合标签（label）形式的标识符使用，即某一行第一个以冒号（`:`）结尾的单词（gofmt 会将后续代码自动移至下一行）。

示例 5.13 [for6.go](examples/chapter_5/for6.go)：

（标签的名称是大小写敏感的，为了提升可读性，一般建议使用全部大写字母）

```go
package main

import "fmt"

func main() {

LABEL1:
	for i := 0; i <= 5; i++ {
		for j := 0; j <= 5; j++ {
			if j == 4 {
				continue LABEL1
			}
			fmt.Printf("i is: %d, and j is: %d\n", i, j)
		}
	}

}
```

本例中，continue 语句指向 LABEL1，当执行到该语句的时候，就会跳转到 LABEL1 标签的位置。

您可以看到当 j==4 和 j==5 的时候，没有任何输出：标签的作用对象为外部循环，因此 i 会直接变成下一个循环的值，而此时 j 的值就被重设为 0，即它的初始值。如果将 continue 改为 break，则不会只退出内层循环，而是直接退出外层循环了。另外，还可以使用 goto 语句和标签配合使用来模拟循环。

示例 5.14 [goto.go](examples/chapter_5/goto.go)：

```go
package main

func main() {
	i:=0
	HERE:
		print(i)
		i++
		if i==5 {
			return
		}
		goto HERE
}
```

上面的代码会输出 `01234`。

使用逆向的 goto 会很快导致意大利面条式的代码，所以不应当使用而选择更好的替代方案。

> **特别注意** 使用标签和 goto 语句是不被鼓励的：它们会很快导致非常糟糕的程序设计，而且总有更加可读的替代方案来实现相同的需求。

一个建议使用 goto 语句的示例会在第 15.1 章的 simple_tcp_server.go 中出现：示例中在发生读取错误时，使用 goto 来跳出无限读取循环并关闭相应的客户端链接。

定义但未使用标签会导致编译错误：`label … defined and not used`。

如果您必须使用 goto，应当只使用正序的标签（标签位于 goto 语句之后），但注意标签和 goto 语句之间不能出现定义新变量的语句，否则会导致编译失败。

示例 5.15 [goto2.go](examples/chapter_5/got2o.go)：

```go
// compile error goto2.go:8: goto TARGET jumps over declaration of b at goto2.go:8
package main

import "fmt"

func main() {
		a := 1
		goto TARGET // compile error
		b := 9
	TARGET:  
		b += a
		fmt.Printf("a is %v *** b is %v", a, b)
}
```

**问题 5.3** 请描述下面 for 循环的输出：

1.

```go
i := 0
for { //since there are no checks, this is an infinite loop
	if i >= 3 { break }
	//break out of this for loop when this condition is met
	fmt.Println("Value of i is:", i)
	i++
}
fmt.Println("A statement just after for loop.")
```

2.

```go
for i := 0; i<7 ; i++ {
	if i%2 == 0 { continue }
	fmt.Println("Odd:", i)
}
```
