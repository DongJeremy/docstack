# 第6章：函数

函数是 Go 里面的基本代码块：Go 函数的功能非常强大，以至于被认为拥有函数式编程语言的多种特性。在这一章，我们将对 [第 4.2.2 节](04.2.md) 所简要描述的函数进行详细的讲解。

## 6.1 介绍

每一个程序都包含很多的函数：函数是基本的代码块。

Go是编译型语言，所以函数编写的顺序是无关紧要的；鉴于可读性的需求，最好把 `main()` 函数写在文件的前面，其他函数按照一定逻辑顺序进行编写（例如函数被调用的顺序）。

编写多个函数的主要目的是将一个需要很多行代码的复杂问题分解为一系列简单的任务（那就是函数）来解决。而且，同一个任务（函数）可以被调用多次，有助于代码重用。

（事实上，好的程序是非常注意DRY原则的，即不要重复你自己（Don't Repeat Yourself），意思是执行特定任务的代码只能在程序里面出现一次。）

当函数执行到代码块最后一行（`}` 之前）或者 `return` 语句的时候会退出，其中 `return` 语句可以带有零个或多个参数；这些参数将作为返回值（参考 [第 6.2 节](06.2.md)）供调用者使用。简单的 `return ` 语句也可以用来结束 for 死循环，或者结束一个协程（goroutine）。

Go 里面有三种类型的函数：  

- 普通的带有名字的函数
- 匿名函数或者`lambda`函数（参考 [第 6.8 节](06.8.md)）
- 方法（`Methods`，参考 [第 10.6 节](10.6.md)）

除了`main()`、`init()`函数外，其它所有类型的函数都可以有参数与返回值。函数参数、返回值以及它们的类型被统称为函数签名。

作为提醒，提前介绍一个语法：

这样是不正确的 Go 代码：

```go
func g()
{
}
```

它必须是这样的：

```go
func g() {
}
```

函数被调用的基本格式如下：

```go
pack1.Function(arg1, arg2, …, argn)
```

`Function` 是 `pack1` 包里面的一个函数，括号里的是被调用函数的实参（`argument`）：这些值被传递给被调用函数的*形参*（`parameter`，参考 [第 6.2 节](06.2.md)）。函数被调用的时候，这些实参将被复制（简单而言）然后传递给被调用函数。函数一般是在其他函数里面被调用的，这个其他函数被称为调用函数（calling function）。函数能多次调用其他函数，这些被调用函数按顺序（简单而言）执行，理论上，函数调用其他函数的次数是无穷的（直到函数调用栈被耗尽）。

一个简单的函数调用其他函数的例子：

示例 6.1 [greeting.go](examples/chapter_6/greeting.go)

```go
package main

func main() {
    println("In main before calling greeting")
    greeting()
    println("In main after calling greeting")
}

func greeting() {
    println("In greeting: Hi!!!!!")
}
```

代码输出：

```go
In main before calling greeting
In greeting: Hi!!!!!
In main after calling greeting
```

函数可以将其他函数调用作为它的参数，只要这个被调用函数的返回值个数、返回值类型和返回值的顺序与调用函数所需求的实参是一致的，例如：

假设 `f1` 需要 3 个参数 `f1(a, b, c int)`，同时 `f2` 返回 3 个参数 `f2(a, b int) (int, int, int)`，就可以这样调用 `f1`：`f1(f2(a, b))`。

**函数重载**（function overloading）指的是可以编写多个同名函数，只要它们拥有不同的形参与/或者不同的返回值，在 Go 里面函数重载是不被允许的。这将导致一个编译错误：

```go
funcName redeclared in this book, previous declaration at lineno
```

Go 语言不支持这项特性的主要原因是函数重载需要进行多余的类型匹配影响性能；没有重载意味着只是一个简单的函数调度。所以你需要给不同的函数使用不同的名字，我们通常会根据函数的特征对函数进行命名（参考 [第 11.12.5 节](11.12.md)）。

如果需要申明一个在外部定义的函数，你只需要给出函数名与函数签名，不需要给出函数体：

```go
func flushICache(begin, end uintptr) // implemented externally
```

**函数也可以以申明的方式被使用，作为一个函数类型**，就像：

```go
type binOp func(int, int) int
```

在这里，不需要函数体 `{}`。

函数是一等值（`first-class value`）：它们可以赋值给变量，就像 `add := binOp` 一样。

这个变量知道自己指向的函数的签名，所以给它赋一个具有不同签名的函数值是不可能的。

函数值（`functions value`）之间可以相互比较：如果它们引用的是相同的函数或者都是 `nil` 的话，则认为它们是相同的函数。函数不能在其它函数里面声明（不能嵌套），不过我们可以通过使用匿名函数（参考 [第 6.8 节](06.8.md)）来破除这个限制。

目前 Go 没有泛型（`generic`）的概念，也就是说它不支持那种支持多种类型的函数。不过在大部分情况下可以通过接口（`interface`），特别是空接口与类型选择（`type switch`，参考 [第 11.12 节](11.12.md)）与/或者通过使用反射（`reflection`，参考 [第 6.8 节](06.8.md)）来实现相似的功能。使用这些技术将导致代码更为复杂、性能更为低下，所以在非常注意性能的的场合，最好是为每一个类型单独创建一个函数，而且代码可读性更强。

## 6.2 函数参数与返回值

函数能够接收参数供自己使用，也可以返回零个或多个值（我们通常把返回多个值称为返回一组值）。相比与 C、C++、Java 和 C#，多值返回是 Go 的一大特性，为我们判断一个函数是否正常执行（参考 [第 5.2 节](05.2.md)）提供了方便。

我们通过 `return` 关键字返回一组值。事实上，任何一个有返回值（单个或多个）的函数都必须以 `return ` 或 `panic`（参考 [第 13 章](13.0.md)）结尾。

在函数块里面，`return` 之后的语句都不会执行。如果一个函数需要返回值，那么这个函数里面的每一个代码分支（code-path）都要有 `return` 语句。

问题 6.1：下面的函数将不会被编译，为什么呢？大家可以试着纠正过来。

```go
func (st *Stack) Pop() int {
    v := 0
    for ix := len(st) - 1; ix >= 0; ix-- {
        if v = st[ix]; v != 0 {
            st[ix] = 0
            return v
        }
    }
}    
```

函数定义时，它的形参一般是有名字的，不过我们也可以定义没有形参名的函数，只有相应的形参类型，就像这样：`func f(int, int, float64)`。

没有参数的函数通常被称为 **niladic** 函数（niladic function），就像 `main.main()`。

### 6.2.1 按值传递（call by value） 按引用传递（call by reference）

Go 默认使用按值传递来传递参数，也就是传递参数的副本。函数接收参数副本之后，在使用变量的过程中可能对副本的值进行更改，但不会影响到原来的变量，比如 `Function(arg1)`。

如果你希望函数可以直接修改参数的值，而不是对参数的副本进行操作，你需要将参数的地址（变量名前面添加`&`符号，比如 `&variable`）传递给函数，这就是按引用传递，比如 `Function(&arg1)`，此时传递给函数的是一个指针。如果传递给函数的是一个指针，指针的值（一个地址）会被复制，但指针的值所指向的地址上的值不会被复制；我们可以通过这个指针的值来修改这个值所指向的地址上的值。

> （**译者注：指针也是变量类型，有自己的地址和值，通常指针的值指向一个变量的地址。所以，按引用传递也是按值传递。**）

几乎在任何情况下，传递指针（一个32位或者64位的值）的消耗都比传递副本来得少。

在函数调用时，像切片（`slice`）、字典（`map`）、接口（`interface`）、通道（`channel`）这样的引用类型都是默认使用引用传递（即使没有显式的指出指针）。

有些函数只是完成一个任务，并没有返回值。我们仅仅是利用了这种函数的副作用，就像输出文本到终端，发送一个邮件或者是记录一个错误等。

但是绝大部分的函数还是带有返回值的。

如下，`simple_function.go` 里的 `MultiPly3Nums` 函数带有三个形参，分别是 `a`、`b`、`c`，还有一个 `int` 类型的返回值（被注释的代码具有和未注释部分同样的功能，只是多引入了一个本地变量）：

示例 6.2 [simple_function.go](examples/chapter_6/simple_function.go)

```go
package main

import "fmt"

func main() {
    fmt.Printf("Multiply 2 * 5 * 6 = %d\n", MultiPly3Nums(2, 5, 6))
    // var i1 int = MultiPly3Nums(2, 5, 6)
    // fmt.Printf("MultiPly 2 * 5 * 6 = %d\n", i1)
}

func MultiPly3Nums(a int, b int, c int) int {
    // var product int = a * b * c
    // return product
    return a * b * c
}
```

输出显示：

```go
Multiply 2 * 5 * 6 = 60
```

如果一个函数需要返回四到五个值，我们可以传递一个切片给函数（如果返回值具有相同类型）或者是传递一个结构体（如果返回值具有不同的类型）。因为传递一个指针允许直接修改变量的值，消耗也更少。

问题 6.2：

如下的两个函数调用有什么不同：

```go
(A) func DoSomething(a *A) {
        b = a
    }

(B) func DoSomething(a A) {
        b = &a
    }
```

### 6.2.2 命名的返回值（named return variables）

如下，`multiple_return.go` 里的函数带有一个 `int` 参数，返回两个 `int` 值；其中一个函数的返回值在函数调用时就已经被赋予了一个初始零值。

`getX2AndX3` 与 `getX2AndX3_2` 两个函数演示了如何使用非命名返回值与命名返回值的特性。当需要返回多个非命名返回值时，需要使用 `()` 把它们括起来，比如 `(int, int)`。

命名返回值作为结果形参（result parameters）*被初始化为相应类型的零值*，当需要返回的时候，我们只需要一条简单的不带参数的`return`语句。需要注意的是，即使只有一个命名返回值，也需要使用 `()` 括起来（参考 [第 6.6 节](06.6.md)的 `fibonacci.go` 函数）。

示例 6.3 [multiple_return.go](examples/chapter_6/multiple_return.go)

```go
package main

import "fmt"

var num int = 10
var numx2, numx3 int

func main() {
    numx2, numx3 = getX2AndX3(num)
    PrintValues()
    numx2, numx3 = getX2AndX3_2(num)
    PrintValues()
}

func PrintValues() {
    fmt.Printf("num = %d, 2x num = %d, 3x num = %d\n", num, numx2, numx3)
}

func getX2AndX3(input int) (int, int) {
    return 2 * input, 3 * input
}

func getX2AndX3_2(input int) (x2 int, x3 int) {
    x2 = 2 * input
    x3 = 3 * input
    // return x2, x3
    return
}
```

输出结果：

```go
num = 10, 2x num = 20, 3x num = 30    
num = 10, 2x num = 20, 3x num = 30 
```

> 警告：
>
> - `return` 或 `return var` 都是可以的。
> - 不过 `return var = expression`（表达式） 会引发一个编译错误：`syntax error: unexpected =, expecting semicolon or newline or }`。     

即使函数使用了命名返回值，你依旧可以无视它而返回明确的值。

任何一个非命名返回值（使用非命名返回值是很糟的编程习惯）在 `return` 语句里面都要明确指出包含返回值的变量或是一个可计算的值（就像上面警告所指出的那样）。

**尽量使用命名返回值：会使代码更清晰、更简短，同时更加容易读懂。**

练习 6.1 [mult_returnval.go](exercises/chapter_6/mult_returnval.go)

编写一个函数，接收两个整数，然后返回它们的和、积与差。编写两个版本，一个是非命名返回值，一个是命名返回值。

```go
// mult_returnval.go
package main

import (
	"fmt"
)

func SumProductDiff(i, j int) (int, int, int) {
	return i + j, i * j, i - j
}

func SumProductDiffN(i, j int) (s int, p int, d int) {
	s, p, d = i+j, i*j, i-j
	return
}

func main() {
	sum, prod, diff := SumProductDiff(3, 4)
	fmt.Println("Sum:", sum, "| Product:", prod, "| Diff:", diff)
	sum, prod, diff = SumProductDiffN(3, 4)
	fmt.Println("Sum:", sum, "| Product:", prod, "| Diff:", diff)
}

// Sum: 7 | Product: 12 | Diff: -1
// Sum: 7 | Product: 12 | Diff: -1
```

练习 6.2 [error_returnval.go](exercises/chapter_6/error_returnval.go)

编写一个名字为 `MySqrt` 的函数，计算一个 `float64` 类型浮点数的平方根，如果参数是一个负数的话将返回一个错误。编写两个版本，一个是非命名返回值，一个是命名返回值。

```go
// error_returnval.go
package main

import (
	"errors"
	"fmt"
	"math"
)

func main() {
	fmt.Print("First example with -1: ")
	ret1, err1 := MySqrt(-1)
	if err1 != nil {
		fmt.Println("Error! Return values are: ", ret1, err1)
	} else {
		fmt.Println("It's ok! Return values are: ", ret1, err1)
	}

	fmt.Print("Second example with 5: ")
	//you could also write it like this
	if ret2, err2 := MySqrt(5); err2 != nil {
		fmt.Println("Error! Return values are: ", ret2, err2)
	} else {
		fmt.Println("It's ok! Return values are: ", ret2, err2)
	}
	// named return variables:
	fmt.Println(MySqrt2(5))
}

func MySqrt(f float64) (float64, error) {
	//return an error as second parameter if invalid input
	if f < 0 {
		return float64(math.NaN()), errors.New("I won't be able to do a sqrt of negative number!")
	}
	//otherwise use default square root function
	return math.Sqrt(f), nil
}

//name the return variables - by default it will have 'zero-ed' values i.e. numbers are 0, string is empty, etc.
func MySqrt2(f float64) (ret float64, err error) {
	if f < 0 {
		//then you can use those variables in code
		ret = float64(math.NaN())
		err = errors.New("I won't be able to do a sqrt of negative number!")
	} else {
		ret = math.Sqrt(f)
		//err is not assigned, so it gets default value nil
	}
	//automatically return the named return variables ret and err
	return
}

/* Output:
First example with -1: Error! Return values are: NaN I won't be able to do a sqrt of negative number!
Second example with 5: It's ok! Return values are: 2.23606797749979 <nil>
2.23606797749979 <nil>
*/
```

### 6.2.3 空白符（blank identifier）

空白符用来匹配一些不需要的值，然后丢弃掉，下面的 `blank_identifier.go` 就是很好的例子。

`ThreeValues` 是拥有三个返回值的不需要任何参数的函数，在下面的例子中，我们将第一个与第三个返回值赋给了 `i1` 与 `f1`。第二个返回值赋给了空白符 `_`，然后自动丢弃掉。

示例 6.4 [blank_identifier.go](examples/chapter_6/blank_identifier.go)

```go
package main

import "fmt"

func main() {
	var i1 int
	var f1 float32
	i1, _, f1 = ThreeValues()
	fmt.Printf("The int: %d, the float: %f \n", i1, f1)
}

func ThreeValues() (int, int, float32) {
	return 5, 6, 7.5
}
```

输出结果：

```go
The int: 5, the float: 7.500000
```

另外一个示例，函数接收两个参数，比较它们的大小，然后按小-大的顺序返回这两个数，示例代码为`minmax.go`。

示例 6.5 [minmax.go](examples/chapter_6/minmax.go)

```go
package main

import "fmt"

func main() {
	var min, max int
	min, max = MinMax(78, 65)
	fmt.Printf("Minmium is: %d, Maximum is: %d\n", min, max)
}

func MinMax(a int, b int) (min int, max int) {
	if a < b {
		min = a
		max = b
	} else { // a = b or a < b
		min = b
		max = a
	}
	return
}
```

输出结果：

```go
Minimum is: 65, Maximum is 78
```

### 6.2.4 改变外部变量（outside variable）

传递指针给函数不但可以节省内存（因为没有复制变量的值），而且赋予了函数直接修改外部变量的能力，所以被修改的变量不再需要使用 `return` 返回。如下的例子，`reply` 是一个指向 `int` 变量的指针，通过这个指针，我们在函数内修改了这个 `int` 变量的数值。

示例 6.6 [side_effect.go](examples/chapter_6/side_effect.go)

```go
package main

import (
	"fmt"
)

// this function changes reply:
func Multiply(a, b int, reply *int) {
	*reply = a * b
}

func main() {
	n := 0
	reply := &n
	Multiply(10, 5, reply)
	fmt.Println("Multiply:", *reply) // Multiply: 50
}
```

这仅仅是个指导性的例子，当需要在函数内改变一个占用内存比较大的变量时，性能优势就更加明显了。然而，如果不小心使用的话，传递一个指针很容易引发一些不确定的事，所以，我们要十分小心那些可以改变外部变量的函数，在必要时，需要添加注释以便其他人能够更加清楚的知道函数里面到底发生了什么。

## 6.3 传递变长参数

如果函数的最后一个参数是采用 `...type` 的形式，那么这个函数就可以处理一个变长的参数，这个长度可以为 `0`，这样的函数称为变参函数。

```go
func myFunc(a, b, arg ...int) {}
```

这个函数接受一个类似某个类型的 `slice` 的参数（详见第 7 章），该参数可以通过第 5.4.4 节中提到的 `for` 循环结构迭代。

示例函数和调用：

```go
func Greeting(prefix string, who ...string)
Greeting("hello:", "Joe", "Anna", "Eileen")
```

在 `Greeting` 函数中，变量 `who` 的值为 `[]string{"Joe", "Anna", "Eileen"}`。

如果参数被存储在一个 `slice` 类型的变量 `slice` 中，则可以通过 `slice...` 的形式来传递参数，调用变参函数。

示例 6.7 [varnumpar.go](examples/chapter_6/varnumpar.go)

```go
package main

import "fmt"

func main() {
	x := min(1, 3, 2, 0)
	fmt.Printf("The minimum is: %d\n", x)
	slice := []int{7,9,3,5,1}
	x = min(slice...)
	fmt.Printf("The minimum in the slice is: %d", x)
}

func min(s ...int) int {
	if len(s)==0 {
		return 0
	}
	min := s[0]
	for _, v := range s {
		if v < min {
			min = v
		}
	}
	return min
}
```

输出：

```go
The minimum is: 0
The minimum in the slice is: 1
```

**练习 6.3** `varargs.go`

写一个函数，该函数接受一个变长参数并对每个元素进行换行打印。

```go
// Q10_varargs.go
package main

import (
	"fmt"
)

func main() {
	printInts()
	println()
	printInts(2, 3)
	println()
	printInts(0, 1, 2, 3, 4, 5, 6, 7, 8, 9)
}

func printInts(list ...int) {
	//	for i:=0; i<len(list); i++ {
	//		fmt.Printf("%d\n", list[i])
	//	}
	for _, v := range list {
		fmt.Printf("%d\n", v)
	}
}
```

一个接受变长参数的函数可以将这个参数作为其它函数的参数进行传递：

```go
func F1(s ...string) {
	F2(s...)
	F3(s)
}

func F2(s ...string) { }
func F3(s []string) { }
```

变长参数可以作为对应类型的 `slice` 进行二次传递。

但是如果变长参数的类型并不是都相同的呢？使用 5 个参数来进行传递并不是很明智的选择，有 2 种方案可以解决这个问题：

1. 使用结构（详见第 10 章）：

   定义一个结构类型，假设它叫 `Options`，用以存储所有可能的参数：

   ```go
   type Options struct {
   	par1 type1,
   	par2 type2,
   	...
   }
   ```

   函数 `F1` 可以使用正常的参数 `a` 和 `b`，以及一个没有任何初始化的 `Options` 结构： `F1(a, b, Options {})`。如果需要对选项进行初始化，则可以使用 `F1(a, b, Options {par1:val1, par2:val2})`。

2. 使用空接口：

   如果一个变长参数的类型没有被指定，则可以使用默认的空接口 `interface{}`，这样就可以接受任何类型的参数（详见第 11.9 节）。该方案不仅可以用于长度未知的参数，还可以用于任何不确定类型的参数。一般而言我们会使用一个 `for-range` 循环以及 `switch` 结构对每个参数的类型进行判断：

   ```go
   func typecheck(..,..,values … interface{}) {
   	for _, value := range values {
   		switch v := value.(type) {
   			case int: …
   			case float: …
   			case string: …
   			case bool: …
   			default: …
   		}
   	}
   }
   ```
   
## 6.4 defer 和追踪

关键字 `defer` 允许我们推迟到函数返回之前（或任意位置执行 `return` 语句之后）一刻才执行某个语句或函数（为什么要在返回之后才执行这些语句？因为 `return` 语句同样可以包含一些操作，而不是单纯地返回某个值）。

关键字 `defer` 的用法类似于面向对象编程语言 Java 和 C# 的 `finally` 语句块，它一般用于释放某些已分配的资源。

示例 6.8 [defer.go](examples/chapter_6/defer.go)：

```go
package main

import "fmt"

func main() {
	function1()
}

func function1() {
	fmt.Printf("In function1 at the top\n")
	defer function2()
	fmt.Printf("In function1 at the bottom!\n")
}

func function2() {
	fmt.Printf("Function2: Deferred until the end of the calling function!")
}
```

输出：

```go
In Function1 at the top
In Function1 at the bottom!
Function2: Deferred until the end of the calling function!
```

请将 `defer` 关键字去掉并对比输出结果。

使用 `defer` 的语句同样可以接受参数，下面这个例子就会在执行 `defer` 语句时打印 `0`：

```go
func a() {
	i := 0
	defer fmt.Println(i)
	i++
	return
}
```

当有多个 `defer` 行为被注册时，它们会以逆序执行（类似栈，即后进先出）：

```go
func f() {
	for i := 0; i < 5; i++ {
		defer fmt.Printf("%d ", i)
	}
}
```

上面的代码将会输出：`4 3 2 1 0`。

关键字 `defer` 允许我们进行一些函数执行完成后的收尾工作，例如：

1. 关闭文件流 （详见 [第 12.2 节](12.2.md)）

```go
// open a file  
defer file.Close()
```

2. 解锁一个加锁的资源 （详见 [第 9.3 节](09.3.md)）

```go
mu.Lock()  
defer mu.Unlock() 
```

3. 打印最终报告

```go
printHeader()  
defer printFooter()
```

4. 关闭数据库链接

```go
// open a database connection  
defer disconnectFromDB()
```

合理使用 `defer` 语句能够使得代码更加简洁。

以下代码模拟了上面描述的第 4 种情况：

```go
package main

import "fmt"

func main() {
	doDBOperations()
}

func connectToDB() {
	fmt.Println("ok, connected to db")
}

func disconnectFromDB() {
	fmt.Println("ok, disconnected from db")
}

func doDBOperations() {
	connectToDB()
	fmt.Println("Defering the database disconnect.")
	defer disconnectFromDB() //function called here with defer
	fmt.Println("Doing some DB operations ...")
	fmt.Println("Oops! some crash or network error ...")
	fmt.Println("Returning from function here!")
	return //terminate the program
	// deferred function executed here just before actually returning, even if
	// there is a return or abnormal termination before
}
```

输出：

```go
ok, connected to db
Defering the database disconnect.
Doing some DB operations ...
Oops! some crash or network error ...
Returning from function here!
ok, disconnected from db
```

**使用 defer 语句实现代码追踪**

一个基础但十分实用的实现代码执行追踪的方案就是在进入和离开某个函数打印相关的消息，即可以提炼为下面两个函数：

```go
func trace(s string) { fmt.Println("entering:", s) }
func untrace(s string) { fmt.Println("leaving:", s) }
```

以下代码展示了何时调用这两个函数：

示例 6.10 [defer_tracing.go](examples/chapter_6/defer_tracing.go):

```go
package main

import "fmt"

func trace(s string)   { fmt.Println("entering:", s) }
func untrace(s string) { fmt.Println("leaving:", s) }

func a() {
	trace("a")
	defer untrace("a")
	fmt.Println("in a")
}

func b() {
	trace("b")
	defer untrace("b")
	fmt.Println("in b")
	a()
}

func main() {
	b()
}
```

输出：

```go
entering: b
in b
entering: a
in a
leaving: a
leaving: b
```

上面的代码还可以修改为更加简便的版本（示例 6.11 [defer_tracing2.go](examples/chapter_6/defer_tracing2.go)）：

```go
package main

import "fmt"

func trace(s string) string {
	fmt.Println("entering:", s)
	return s
}

func un(s string) {
	fmt.Println("leaving:", s)
}

func a() {
	defer un(trace("a"))
	fmt.Println("in a")
}

func b() {
	defer un(trace("b"))
	fmt.Println("in b")
	a()
}

func main() {
	b()
}
```

**使用 defer 语句来记录函数的参数与返回值**

下面的代码展示了另一种在调试时使用 `defer` 语句的手法（示例 6.12 [defer_logvalues.go](examples/chapter_6/defer_logvalues.go)）：

```go
package main

import (
	"io"
	"log"
)

func func1(s string) (n int, err error) {
	defer func() {
		log.Printf("func1(%q) = %d, %v", s, n, err)
	}()
	return 7, io.EOF
}

func main() {
	func1("Go")
}

```

输出：

```go
Output: 2011/10/04 10:46:11 func1("Go") = 7, EOF
```

## 6.5 内置函数

Go 语言拥有一些不需要进行导入操作就可以使用的内置函数。它们有时可以针对不同的类型进行操作，例如：`len`、`cap` 和 `append`，或必须用于系统级的操作，例如：`panic`。因此，它们需要直接获得编译器的支持。

以下是一个简单的列表，我们会在后面的章节中对它们进行逐个深入的讲解。

| 名称                   | 说明                                                         |
| ---------------------- | ------------------------------------------------------------ |
| `close`                | 用于管道通信                                                 |
| `len`、`cap`           | `len` 用于返回某个类型的长度或数量（字符串、数组、切片、map 和管道）；`cap` 是容量的意思，用于返回某个类型的最大容量（只能用于切片和 map） |
| `new`、`make`          | `new` 和 `make` 均是用于分配内存：`new` 用于值类型和用户定义的类型，如自定义结构，`make` 用于内置引用类型（切片、map 和管道）。它们的用法就像是函数，但是将类型作为参数：new(type)、make(type)。new(T) 分配类型 T 的零值并返回其地址，也就是指向类型 T 的指针（详见第 10.1 节）。它也可以被用于基本类型：`v := new(int)`。make(T) 返回类型 T 的初始化之后的值，因此它比 new 进行更多的工作（详见第 7.2.3/4 节、第 8.1.1 节和第 14.2.1 节）**new() 是一个函数，不要忘记它的括号** |
| `copy`、`append`       | 用于复制和连接切片                                           |
| `panic`、`recover`     | 两者均用于错误处理机制                                       |
| `print`、`println`     | 底层打印函数（详见第 4.2 节），在部署环境中建议使用 `fmt` 包 |
| `complex`、`real imag` | 用于创建和操作复数（详见第 4.5.2.2 节）                      |

## 6.6 递归函数

当一个函数在其函数体内调用自身，则称之为递归。最经典的例子便是计算斐波那契数列，即前两个数为1，从第三个数开始每个数均为前两个数之和。

数列如下所示：

	1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233, 377, 610, 987, 1597, 2584, 4181, 6765, 10946, …

下面的程序可用于生成该数列（示例 6.13 [fibonacci.go](examples/chapter_6/fibonacci.go)）：

```go
package main

import "fmt"

func main() {
	result := 0
	for i := 0; i <= 10; i++ {
		result = fibonacci(i)
		fmt.Printf("fibonacci(%d) is: %d\n", i, result)
	}
}

func fibonacci(n int) (res int) {
	if n <= 1 {
		res = 1
	} else {
		res = fibonacci(n-1) + fibonacci(n-2)
	}
	return
}
```

输出：

```go
fibonacci(0) is: 1
fibonacci(1) is: 1
fibonacci(2) is: 2
fibonacci(3) is: 3
fibonacci(4) is: 5
fibonacci(5) is: 8
fibonacci(6) is: 13
fibonacci(7) is: 21
fibonacci(8) is: 34
fibonacci(9) is: 55
fibonacci(10) is: 89
```

许多问题都可以使用优雅的递归来解决，比如说著名的快速排序算法。

在使用递归函数时经常会遇到的一个重要问题就是栈溢出：一般出现在大量的递归调用导致的程序栈内存分配耗尽。这个问题可以通过一个名为[懒惰求值](https://zh.wikipedia.org/wiki/惰性求值)的技术解决，在 Go 语言中，我们可以使用管道（`channel`）和 `goroutine`（详见第 14.8 节）来实现。练习 14.12 也会通过这个方案来优化斐波那契数列的生成问题。

Go 语言中也可以使用相互调用的递归函数：多个函数之间相互调用形成闭环。因为 Go 语言编译器的特殊性，这些函数的声明顺序可以是任意的。下面这个简单的例子展示了函数 `odd` 和 `even` 之间的相互调用（示例 6.14 [mut_recurs.go](examples/chapter_6/mut_recurs.go)）：

```go
package main

import (
	"fmt"
)

func main() {
	fmt.Printf("%d is even: is %t\n", 16, even(16)) // 16 is even: is true
	fmt.Printf("%d is odd: is %t\n", 17, odd(17))
	// 17 is odd: is true
	fmt.Printf("%d is odd: is %t\n", 18, odd(18))
	// 18 is odd: is false
}

func even(nr int) bool {
	if nr == 0 {
		return true
	}
	return odd(RevSign(nr) - 1)
}

func odd(nr int) bool {
	if nr == 0 {
		return false
	}
	return even(RevSign(nr) - 1)
}

func RevSign(nr int) int {
	if nr < 0 {
		return -nr
	}
	return nr
}
```

### 练习题

**练习 6.4**

重写本节中生成斐波那契数列的程序并返回两个命名返回值（详见第 6.2 节），即数列中的位置和对应的值，例如 5 与 4，89 与 10。

```go
package main

import "fmt"

func main() {
	pos := 4
	result, pos := fibonacci(pos)
	fmt.Printf("the %d-th fibonacci number is: %d\n", pos, result)
	pos = 10
	result, pos = fibonacci(pos)
	fmt.Printf("the %d-th fibonacci number is: %d\n", pos, result)
}

func fibonacci(n int) (val, pos int) {
	if n <= 1 {
		val = 1
	} else {
		v1, _ := fibonacci(n - 1)
		v2, _ := fibonacci(n - 2)
		val = v1 + v2
	}
	pos = n
	return
}
/* Output:
the 4-th fibonacci number is: 5
the 10-th fibonacci number is: 89
*/
```

**练习 6.5**

使用递归函数从 10 打印到 1。

```go
package main

import "fmt"

func main() {
	printrec(1)
}

func printrec(i int) {
	if i > 10 {
		return
	}
	printrec(i + 1)
	fmt.Printf("%d ", i)
}
/* Output:
10 9 8 7 6 5 4 3 2 1 
*/
```

**练习 6.6**

实现一个输出前 30 个整数的阶乘的程序。

n! 的阶乘定义为：`n! = n * (n-1)!, 0! = 1`，因此它非常适合使用递归函数来实现。

然后，使用命名返回值来实现这个程序的第二个版本。

特别注意的是，使用 `int` 类型最多只能计算到 12 的阶乘，因为一般情况下 `int` 类型的大小为 32 位，继续计算会导致溢出错误。那么，如何才能解决这个问题呢？

最好的解决方案就是使用 `big` 包（详见第 9.4 节）。

```go
package main

import (
	"fmt"
	"math/big"
)

func main() {
	var i int64
	for i = 1; i <= 30; i++ {
		fmt.Println(iterCount(i, big.NewInt(i)))
	}
}

func iterCount(num int64, count *big.Int) *big.Int {
	if num <= 1 || count.Int64() == 1 {
		return big.NewInt(1)
	}
	return count.Mul(count, iterCount(num-1, big.NewInt(count.Int64()-1)))
}
/* Output:
1
2
6
24
120
720
5040
40320
362880
3628800
39916800
479001600
6227020800
87178291200
1307674368000
20922789888000
355687428096000
6402373705728000
121645100408832000
2432902008176640000
51090942171709440000
1124000727777607680000
25852016738884976640000
620448401733239439360000
15511210043330985984000000
403291461126605635584000000
10888869450418352160768000000
304888344611713860501504000000
8841761993739701954543616000000
265252859812191058636308480000000
*/
```

## 6.7 将函数作为参数

函数可以作为其它函数的参数进行传递，然后在其它函数内调用执行，一般称之为回调。下面是一个将函数作为参数的简单例子（`function_parameter.go`）：

```go
package main

import (
	"fmt"
)

func main() {
	callback(1, Add)
}

func Add(a, b int) {
	fmt.Printf("The sum of %d and %d is: %d\n", a, b, a+b)
}

func callback(y int, f func(int, int)) {
	f(y, 2) // this becomes Add(1, 2)
}
```

输出：

```go
The sum of 1 and 2 is: 3
```

将函数作为参数的最好的例子是函数 `strings.IndexFunc()`：

该函数的签名是 `func IndexFunc(s string, f func(c rune) bool) int`，它的返回值是在函数 `f(c)` 返回 `true`、`-1` 或从未返回时的索引值。

例如 `strings.IndexFunc(line, unicode.IsSpace)` 就会返回 `line` 中第一个空白字符的索引值。当然，您也可以书写自己的函数：

```go
func IsAscii(c int) bool {
	if c > 255 {
		return false
	}
	return true
}
```

在第 14.10.1 节中，我们将会根据一个客户端/服务端程序作为示例对这个用法进行深入讨论。

```go
type binOp func(a, b int) int
func run(op binOp, req *Request) { … }
```

**练习 6.7**

包 `strings` 中的 `Map` 函数和 `strings.IndexFunc()` 一样都是非常好的使用例子。请学习它的源代码并基于该函数书写一个程序，要求将指定文本内的所有非 ASCII 字符替换成 `?` 或空格。您需要怎么做才能删除这些字符呢？

```go
// strings_map.go
package main

import (
	"fmt"
	"strings"
)

func main() {
	asciiOnly := func(c rune) rune {
		if c > 127 {
			return ' '
		}
		return c
	}
	fmt.Println(strings.Map(asciiOnly, "Jérôme Österreich"))
}

// J r me  sterreich
```

## 6.8 闭包

当我们不希望给函数起名字的时候，可以使用匿名函数，例如：`func(x, y int) int { return x + y }`。

这样的一个函数不能够独立存在（编译器会返回错误：`non-declaration statement
outside function body`），但可以被赋值于某个变量，即保存函数的地址到变量中：`fplus := func(x, y int) int { return x + y }`，然后通过变量名对函数进行调用：`fplus(3,4)`。

当然，您也可以直接对匿名函数进行调用：`func(x, y int) int { return x + y } (3, 4)`。

下面是一个计算从 1 到 1 百万整数的总和的匿名函数：

```go
func() {
	sum := 0
	for i := 1; i <= 1e6; i++ {
		sum += i
	}
}()
```

表示参数列表的第一对括号必须紧挨着关键字 `func`，因为匿名函数没有名称。花括号 `{}` 涵盖着函数体，最后的一对括号表示对该匿名函数的调用。

下面的例子展示了如何将匿名函数赋值给变量并对其进行调用（`function_literal.go`）：

```go
package main

import "fmt"

func main() {
	f()
}
func f() {
	for i := 0; i < 4; i++ {
		g := func(i int) { fmt.Printf("%d ", i) } //此例子中只是为了演示匿名函数可分配不同的内存地址，在现实开发中，不应该把该部分信息放置到循环中。
		g(i)
		fmt.Printf(" - g is of type %T and has value %v\n", g, g)
	}
}
```

输出：

```
0 - g is of type func(int) and has value 0x681a80
1 - g is of type func(int) and has value 0x681b00
2 - g is of type func(int) and has value 0x681ac0
3 - g is of type func(int) and has value 0x681400
```

我们可以看到变量 `g` 代表的是 `func(int)`，变量的值是一个内存地址。

所以我们实际上拥有的是一个函数值：匿名函数可以被赋值给变量并作为值使用。

**练习 6.8** 在 main 函数中写一个用于打印 `Hello World` 字符串的匿名函数并赋值给变量 `fv`，然后调用该函数并打印变量 `fv` 的类型。

```go
package main

import (
	"fmt"
	"reflect"
)

func main() {
	fv := func() {
		fmt.Println("Hello world")
	}
	fv()
	fmt.Println(reflect.TypeOf(fv))
}
/* Output:
Hello world
func()
*/
```

匿名函数像所有函数一样可以接受或不接受参数。下面的例子展示了如何传递参数到匿名函数中：

```go
func (u string) {
	fmt.Println(u)
	…
}(v)
```

请学习以下示例并思考（`return_defer.go`）：函数 `f` 返回时，变量 `ret` 的值是什么？

```go
package main

import "fmt"

func f() (ret int) {
	defer func() {
		ret++
	}()
	return 1
}
func main() {
	fmt.Println(f())
}
```

变量 `ret` 的值为 2，因为 `ret++` 是在执行 `return 1` 语句后发生的。

这可用于在返回语句之后修改返回的 `error` 时使用。

**defer 语句和匿名函数**

关键字 `defer` （详见第 6.4 节）经常配合匿名函数使用，它可以用于改变函数的命名返回值。

匿名函数还可以配合 `go` 关键字来作为 `goroutine` 使用（详见第 14 章和第 16.9 节）。

**匿名函数同样被称之为闭包**（函数式语言的术语）：它们被允许调用定义在其它环境下的变量。闭包可使得某个函数捕捉到一些外部状态，例如：函数被创建时的状态。另一种表示方式为：一个闭包继承了函数所声明时的作用域。这种状态（作用域内的变量）都被共享到闭包的环境中，因此这些变量可以在闭包中被操作，直到被销毁，详见第 6.9 节中的示例。闭包经常被用作包装函数：它们会预先定义好 1 个或多个参数以用于包装，详见下一节中的示例。另一个不错的应用就是使用闭包来完成更加简洁的错误检查（详见第 16.10.2 节）。

## 6.9 应用闭包：将函数作为返回值

在程序 `function_return.go` 中我们将会看到函数 `Add2` 和 `Adder` 均会返回签名为 `func(b int) int` 的函数：

```go
func Add2() (func(b int) int)
func Adder(a int) (func(b int) int)
```

函数 `Add2` 不接受任何参数，但函数 `Adder` 接受一个 `int` 类型的整数作为参数。

我们也可以将 `Adder` 返回的函数存到变量中（`function_return.go`）。

```go
package main

import "fmt"

func main() {
	// make an Add2 function, give it a name p2, and call it:
	p2 := Add2()
	fmt.Printf("Call Add2 for 3 gives: %v\n", p2(3))
	// make a special Adder function, a gets value 2:
	TwoAdder := Adder(2)
	fmt.Printf("The result is: %v\n", TwoAdder(3))
}

func Add2() func(b int) int {
	return func(b int) int {
		return b + 2
	}
}

func Adder(a int) func(b int) int {
	return func(b int) int {
		return a + b
	}
}
```

输出：

```go
Call Add2 for 3 gives: 5
The result is: 5
```

下例为一个略微不同的实现（`function_closure.go`）：

```go
package main

import "fmt"

func main() {
	var f = Adder()
	fmt.Print(f(1), " - ")
	fmt.Print(f(20), " - ")
	fmt.Print(f(300))
}

func Adder() func(int) int {
	var x int
	return func(delta int) int {
		x += delta
		return x
	}
}
```

函数 `Adder()` 现在被赋值到变量 `f` 中（类型为 `func(int) int`）。

输出：

```go
1 - 21 - 321
```

三次调用函数 `f` 的过程中函数 `Adder()` 中变量 `delta` 的值分别为：1、20 和 300。

我们可以看到，在多次调用中，变量 `x` 的值是被保留的，即 `0 + 1 = 1`，然后 `1 + 20 = 21`，最后 `21 + 300 = 321`：闭包函数保存并积累其中的变量的值，不管外部函数退出与否，它都能够继续操作外部函数中的局部变量。

这些局部变量同样可以是参数，例如之前例子中的 `Adder(as int)`。

这些例子清楚地展示了如何在 Go 语言中使用闭包。

在闭包中使用到的变量可以是在闭包函数体内声明的，也可以是在外部函数声明的：

```go
var g int
go func(i int) {
	s := 0
	for j := 0; j < i; j++ { s += j }
	g = s
}(1000) // Passes argument 1000 to the function literal.
```

这样闭包函数就能够被应用到整个集合的元素上，并修改它们的值。然后这些变量就可以用于表示或计算全局或平均值。

**练习 6.9** 不使用递归但使用闭包改写第 6.6 节中的斐波那契数列程序。

```go
package main

// fib returns a function that returns
// successive Fibonacci numbers.
func fib() func() int {
	a, b := 1, 1
	return func() int {
		a, b = b, a+b
		return b
	}
}

func main() {
	f := fib()
	// Function calls are evaluated left-to-right.
	// println(f(), f(), f(), f(), f())
	for i := 0; i <= 9; i++ {
		println(i+2, f())
	}
}
```

**练习 6.10** 

学习并理解以下程序的工作原理：

一个返回值为另一个函数的函数可以被称之为工厂函数，这在您需要创建一系列相似的函数的时候非常有用：书写一个工厂函数而不是针对每种情况都书写一个函数。下面的函数演示了如何动态返回追加后缀的函数：

```go
func MakeAddSuffix(suffix string) func(string) string {
	return func(name string) string {
		if !strings.HasSuffix(name, suffix) {
			return name + suffix
		}
		return name
	}
}
```

现在，我们可以生成如下函数：

```go
addBmp := MakeAddSuffix(".bmp")
addJpeg := MakeAddSuffix(".jpeg")
```

然后调用它们：

```go
addBmp("file") // returns: file.bmp
addJpeg("file") // returns: file.jpeg
```

可以返回其它函数的函数和接受其它函数作为参数的函数均被称之为高阶函数，是函数式语言的特点。我们已经在第 6.7 中得知函数也是一种值，因此很显然 Go 语言具有一些函数式语言的特性。闭包在 Go 语言中非常常见，常用于 `goroutine` 和管道操作（详见第 14.8-14.9 节）。在第 11.14 节的程序中，我们将会看到 Go 语言中的函数在处理混合对象时的强大能力。

## 6.10 使用闭包调试

当您在分析和调试复杂的程序时，无数个函数在不同的代码文件中相互调用，如果这时候能够准确地知道哪个文件中的具体哪个函数正在执行，对于调试是十分有帮助的。您可以使用 `runtime` 或 `log` 包中的特殊函数来实现这样的功能。包 `runtime` 中的函数 `Caller()` 提供了相应的信息，因此可以在需要的时候实现一个 `where()` 闭包函数来打印函数执行的位置：

```go
where := func() {
	_, file, line, _ := runtime.Caller(1)
	log.Printf("%s:%d", file, line)
}
where()
// some code
where()
// some more code
where()
```

您也可以设置 `log` 包中的 `flag` 参数来实现：

```go
log.SetFlags(log.Llongfile)
log.Print("")
```

或使用一个更加简短版本的 `where` 函数：

```go
var where = log.Print
func func1() {
where()
... some code
where()
... some code
where()
}
```

## 6.11 计算函数执行时间

有时候，能够知道一个计算执行消耗的时间是非常有意义的，尤其是在对比和基准测试中。最简单的一个办法就是在计算开始之前设置一个起始时候，再由计算结束时的结束时间，最后取出它们的差值，就是这个计算所消耗的时间。想要实现这样的做法，可以使用 `time` 包中的 `Now()` 和 `Sub` 函数：

```go
start := time.Now()
longCalculation()
end := time.Now()
delta := end.Sub(start)
fmt.Printf("longCalculation took this amount of time: %s\n", delta)
```

您可以查看示例 6.20 [fibonacci.go](examples/chapter_6/fibonacci.go) 作为实例学习。

如果您对一段代码进行了所谓的优化，请务必对它们之间的效率进行对比再做出最后的判断。在接下来的章节中，我们会学习如何进行有价值的优化操作。

## 6.12 通过内存缓存来提升性能

当在进行大量的计算时，提升性能最直接有效的一种方式就是避免重复计算。通过在内存中缓存和重复利用相同计算的结果，称之为内存缓存。最明显的例子就是生成斐波那契数列的程序（详见第 6.6 和 6.11 节）：

要计算数列中第 `n` 个数字，需要先得到之前两个数的值，但很明显绝大多数情况下前两个数的值都是已经计算过的。即每个更后面的数都是基于之前计算结果的重复计算，正如示例 6.11 [fibonnaci.go](examples/chapter_6/fibonacci.go) 所展示的那样。

而我们要做就是将第 `n` 个数的值存在数组中索引为 `n` 的位置（详见第 7 章），然后在数组中查找是否已经计算过，如果没有找到，则再进行计算。

程序 Listing 6.17 - `fibonacci_memoization.go` 就是依照这个原则实现的，下面是计算到第 40 位数字的性能对比：

- 普通写法：4.730270 秒
- 内存缓存：0.001000 秒

内存缓存的优势显而易见，而且您还可以将它应用到其它类型的计算中，例如使用 `map`（详见第 7 章）而不是数组或切片（Listing 6.21 - [fibonacci_memoization.go](examples/chapter_6/fibonacci_memoization.go)）：

```go
package main

import (
	"fmt"
	"time"
)

const LIM = 41

var fibs [LIM]uint64

func main() {
	var result uint64 = 0
	start := time.Now()
	for i := 0; i < LIM; i++ {
		result = fibonacci(i)
		fmt.Printf("fibonacci(%d) is: %d\n", i, result)
	}
	end := time.Now()
	delta := end.Sub(start)
	fmt.Printf("longCalculation took this amount of time: %s\n", delta)
}
func fibonacci(n int) (res uint64) {
	// memoization: check if fibonacci(n) is already known in array:
	if fibs[n] != 0 {
		res = fibs[n]
		return
	}
	if n <= 1 {
		res = 1
	} else {
		res = fibonacci(n-1) + fibonacci(n-2)
	}
	fibs[n] = res
	return
}
```

内存缓存的技术在使用计算成本相对昂贵的函数时非常有用（不仅限于例子中的递归），譬如大量进行相同参数的运算。这种技术还可以应用于纯函数中，即相同输入必定获得相同输出的函数。

