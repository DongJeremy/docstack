# 第7章：数组与切片

这章我们开始剖析 **容器**, 它是可以包含大量条目（item）的数据结构, 例如数组、切片和 `map`。从这看到 Go 明显受到 Python 的影响。

以 `[]` 符号标识的数组类型几乎在所有的编程语言中都是一个基本主力。Go 语言中的数组也是类似的，只是有一些特点。Go 没有 C 那么灵活，但是拥有切片（`slice`）类型。这是一种建立在 Go 语言数组类型之上的抽象，要想理解切片我们必须先理解数组。数组有特定的用处，但是却有一些呆板，所以在 Go 语言的代码里并不是特别常见。相对的，切片确实随处可见的。它们构建在数组之上并且提供更强大的能力和便捷。

## 7.1 声明和初始化

### 7.1.1 概念
数组是具有相同 **唯一类型** 的一组已编号且长度固定的数据项序列（这是一种同构的数据结构）；这种类型可以是任意的原始类型例如整型、字符串或者自定义类型。数组长度必须是一个常量表达式，并且必须是一个非负整数。数组长度也是数组类型的一部分，所以`[5]int`和`[10]int`是属于不同类型的。数组的编译时值初始化是按照数组顺序完成的（如下）。

**注意事项** 如果我们想让数组元素类型为任意类型的话可以使用空接口作为类型（参考 [第 11 章](11.9.md)）。当使用值时我们必须先做一个类型判断（参考 [第 11 章](11.3.md)）。

数组元素可以通过 **索引**（位置）来读取（或者修改），索引从 `0` 开始，第一个元素索引为 `0`，第二个索引为 1，以此类推。（数组以 0 开始在所有类 C 语言中是相似的）。元素的数目，也称为长度或者数组大小必须是固定的并且在声明该数组时就给出（编译时需要知道数组长度以便分配内存）；数组长度最大为 2Gb。

声明的格式是： 

```go
var identifier [len]type
```

例如： 

```go
var arr1 [5]int
```

在内存中的结构是：![](images/7.1_fig7.1.png?raw=true)

每个元素是一个整型值，当声明数组时所有的元素都会被自动初始化为默认值 `0`。

`arr1` 的长度是 `5`，索引范围从 `0` 到 `len(arr1)-1`。

第一个元素是 `arr1[0]`，第三个元素是 `arr1[2]`；总体来说索引 `i` 代表的元素是 `arr1[i]`，最后一个元素是 `arr1[len(arr1)-1]`。

对索引项为 `i` 的数组元素赋值可以这么操作：`arr[i] = value`，所以数组是 **可变的**。

只有有效的索引可以被使用，当使用等于或者大于 `len(arr1)` 的索引时：如果编译器可以检测到，会给出索引超限的提示信息；如果检测不到的话编译会通过而运行时会 panic:（参考 [第 13 章](13.0.md)）

```go
runtime error: index out of range
```

由于索引的存在，遍历数组的方法自然就是使用 `for` 结构:

- 通过 `for` 初始化数组项
- 通过 `for` 打印数组元素
- 通过 `for` 依次处理元素

示例 7.1 [for_arrays.go](examples/chapter_7/for_arrays.go)

```go
package main
import "fmt"

func main() {
	var arr1 [5]int

	for i:=0; i < len(arr1); i++ {
		arr1[i] = i * 2
	}

	for i:=0; i < len(arr1); i++ {
		fmt.Printf("Array at index %d is %d\n", i, arr1[i])
	}
}
```

输出结果：

```go
Array at index 0 is 0
Array at index 1 is 2
Array at index 2 is 4
Array at index 3 is 6
Array at index 4 is 8
```

`for` 循环中的条件非常重要：`i < len(arr1)`，如果写成 `i <= len(arr1)` 的话会产生越界错误。

语法:

```go
for i:=0; i < len(arr1); i++｛
	arr1[i] = ...
}
```

也可以使用 `for-range` 的生成方式：

语法:

```go
for i,_:= range arr1 {
...
}
```

在这里`i`也是数组的索引。当然这两种 `for` 结构对于切片（`slices`）（参考 [第 7 章](07.2.md)）来说也同样适用。

**问题 7.1** 下面代码段的输出是什么？

```go
a := [...]string{"a", "b", "c", "d"}
for i := range a {
	fmt.Println("Array item", i, "is", a[i])
}
```

Go 语言中的数组是一种 **值类型**（不像 C/C++ 中是指向首元素的指针），所以可以通过 `new()` 来创建： `var arr1 = new([5]int)`。

那么这种方式和 `var arr2 [5]int` 的区别是什么呢？arr1 的类型是 `*[5]int`，而 arr2的类型是 `[5]int`。

这样的结果就是当把一个数组赋值给另一个时，需要再做一次数组内存的拷贝操作。例如：

```go
arr2 := *arr1
arr2[2] = 100
```

这样两个数组就有了不同的值，在赋值后修改 `arr2` 不会对 `arr1` 生效。

所以在函数中数组作为参数传入时，如 `func1(arr2)`，会产生一次数组拷贝，`func1` 方法不会修改原始的数组 `arr2`。

如果你想修改原数组，那么 `arr2` 必须通过`&`操作符以引用方式传过来，例如 `func1(&arr2)`，下面是一个例子

示例 7.2 [pointer_array.go](examples/chapter_7/pointer_array.go):

```go
package main

import "fmt"

func f(a [3]int)   { fmt.Println(a) }
func fp(a *[3]int) { fmt.Println(a) }

func main() {
	var ar [3]int
	f(ar)   // passes a copy of ar
	fp(&ar) // passes a pointer to ar
}
```

输出结果：

```go
[0 0 0]
&[0 0 0]
```

另一种方法就是生成数组切片并将其传递给函数（详见第 7.1.4 节）。

**练习**

练习7.1：`array_value.go`: 证明当数组赋值时，发生了数组内存拷贝。

```go
package main

import "fmt"

func main() {
	var arr1 [5]int

	for i := 0; i < len(arr1); i++ {
		arr1[i] = i * 2
	}

	arr2 := arr1
	arr2[2] = 100

	for i := 0; i < len(arr1); i++ {
		fmt.Printf("Array arr1 at index %d is %d\n", i, arr1[i])
	}
	fmt.Println()
	for i := 0; i < len(arr2); i++ {
		fmt.Printf("Array arr2 at index %d is %d\n", i, arr2[i])
	}
}
/* Output:
Array arr1 at index 0 is 0
Array arr1 at index 1 is 2
Array arr1 at index 2 is 4
Array arr1 at index 3 is 6
Array arr1 at index 4 is 8

Array arr2 at index 0 is 0
Array arr2 at index 1 is 2
Array arr2 at index 2 is 100
Array arr2 at index 3 is 6
Array arr2 at index 4 is 8
*/
```

练习7.2：`for_array.go`: 写一个循环并用下标给数组赋值（从 0 到 15）并且将数组打印在屏幕上。

```go
package main

import "fmt"

func main() {
	var arr [15]int
	for i := 0; i < 15; i++ {
		arr[i] = i
	}
	fmt.Println(arr) // [0 1 2 3 4 5 6 7 8 9 10 11 12 13 14]

}
```

练习7.3：`fibonacci_array.go`: 在第 6.6 节我们看到了一个递归计算 `Fibonacci` 数值的方法。但是通过数组我们可以更快的计算出 `Fibonacci` 数。完成该方法并打印出前 50 个 `Fibonacci` 数字。

```go
package main

import "fmt"

var fibs [50]int64

func main() {
	fibs[0] = 1
	fibs[1] = 1

	for i := 2; i < 50; i++ {
		fibs[i] = fibs[i-1] + fibs[i-2]
	}

	for i := 0; i < 50; i++ {
		fmt.Printf("The %d-th Fibonacci number is: %d\n", i, fibs[i])
	}
}
```

### 7.1.2 数组常量

如果数组值已经提前知道了，那么可以通过 **数组常量** 的方法来初始化数组，而不用依次使用 `[]=` 方法（所有的组成元素都有相同的常量语法）。

示例 7.3 [array_literals.go](examples/chapter_7/array_literals.go)

```go
package main

import "fmt"

func main() {
	// var arrAge = [5]int{18, 20, 15, 22, 16}
	// var arrLazy = [...]int{5, 6, 7, 8, 22}
	// var arrLazy = []int{5, 6, 7, 8, 22}	//注：初始化得到的实际上是切片slice
	var arrKeyValue = [5]string{3: "Chris", 4: "Ron"}
	// var arrKeyValue = []string{3: "Chris", 4: "Ron"}	//注：初始化得到的实际上是切片slice

	for i:=0; i < len(arrKeyValue); i++ {
		fmt.Printf("Person at %d is %s\n", i, arrKeyValue[i])
	}
}
```

第一种变化：

```go
var arrAge = [5]int{18, 20, 15, 22, 16}
```

注意 `[5]int` 可以从左边起开始忽略：`[10]int {1, 2, 3}` :这是一个有 10 个元素的数组，除了前三个元素外其他元素都为 `0`。

第二种变化：

```go
var arrLazy = [...]int{5, 6, 7, 8, 22}
```

`...` 可同样可以忽略，从技术上说它们其实变化成了切片。

第三种变化：`key: value 语法`

```go
var arrKeyValue = [5]string{3: "Chris", 4: "Ron"}
```

只有索引 `3` 和 `4` 被赋予实际的值，其他元素都被设置为空的字符串，所以输出结果为：

```go
Person at 0 is
Person at 1 is
Person at 2 is
Person at 3 is Chris
Person at 4 is Ron
```

在这里数组长度同样可以写成 `...`。

你可以取任意数组常量的地址来作为指向新实例的指针。

示例 7.4 [pointer_array2.go](examples/chapter_7/pointer_array2.go)

```go
package main

import "fmt"

func fp(a *[3]int) { fmt.Println(a) }

func main() {
	for i := 0; i < 3; i++ {
		fp(&[3]int{i, i * i, i * i * i})
	}
}
```

输出结果：
```go
&[0 0 0]
&[1 1 1]
&[2 4 8]
```

几何点（或者数学向量）是一个使用数组的经典例子。为了简化代码通常使用一个别名：

```go
type Vector3D [3]float32
var vec Vector3D
```

### 7.1.3 多维数组

数组通常是一维的，但是可以用来组装成多维数组，例如：`[3][5]int`，`[2][2][2]float64`。

内部数组总是长度相同的。Go 语言的多维数组是矩形式的（唯一的例外是切片的数组，参见第 7.2.5 节）。

示例 7.5 [multidim_array.go](examples/chapter_7/multidim_array.go) 
```go
package main
const (
	WIDTH  = 1920
	HEIGHT = 1080
)

type pixel int
var screen [WIDTH][HEIGHT]pixel

func main() {
	for y := 0; y < HEIGHT; y++ {
		for x := 0; x < WIDTH; x++ {
			screen[x][y] = 0
		}
	}
}
```

### 7.1.4 将数组传递给函数

把一个大数组传递给函数会消耗很多内存。有两种方法可以避免这种现象：

- 传递数组的指针
- 使用数组的切片

接下来的例子阐明了第一种方法：

示例 7.6 [array_sum.go](examples/chapter_7/array_sum.go)
```go
package main
import "fmt"

func main() {
	array := [3]float64{7.0, 8.5, 9.1}
	x := Sum(&array) // Note the explicit address-of operator
	// to pass a pointer to the array
	fmt.Printf("The sum of the array is: %f", x)
}

func Sum(a *[3]float64) (sum float64) {
	for _, v := range a { // derefencing *a to get back to the array is not necessary!
		sum += v
	}
	return
}
```

输出结果：

	The sum of the array is: 24.600000

但这在 Go 中并不常用，通常使用切片（参考 [第 7.2 节](07.2.md)）。

## 7.2 切片

### 7.2.1 概念

切片（`slice`）是对数组一个连续片段的引用（该数组我们称之为相关数组，通常是匿名的），所以切片是一个引用类型（因此更类似于 C/C++ 中的数组类型，或者 Python 中的 `list` 类型）。这个片段可以是整个数组，或者是由起始和终止索引标识的一些项的子集。需要注意的是，终止索引标识的项不包括在切片内。切片提供了一个相关数组的动态窗口。

切片是可索引的，并且可以由 `len()` 函数获取长度。

给定项的切片索引可能比相关数组的相同元素的索引小。和数组不同的是，切片的长度可以在运行时修改，最小为 0 最大为相关数组的长度：切片是一个 **长度可变的数组**。

切片提供了计算容量的函数 `cap()` 可以测量切片最长可以达到多少：它等于切片的长度 + 数组除切片之外的长度。如果 `s` 是一个切片，`cap(s)` 就是从 `s[0]` 到数组末尾的数组长度。切片的长度永远不会超过它的容量，所以对于 切片 `s` 来说该不等式永远成立：`0 <= len(s) <= cap(s)`。

多个切片如果表示同一个数组的片段，它们可以共享数据；因此一个切片和相关数组的其他切片是共享存储的，相反，不同的数组总是代表不同的存储。数组实际上是切片的构建块。

**优点** 因为切片是引用，所以它们不需要使用额外的内存并且比使用数组更有效率，所以在 Go 代码中*切片比数组更常用*。

声明切片的格式是： `var identifier []type`（不需要说明长度）。

一个切片在未初始化之前默认为 `nil`，长度为 `0`。

切片的初始化格式是：`var slice1 []type = arr1[start:end]`。

这表示 `slice1` 是由数组 `arr1` 从 `start` 索引到 `end-1` 索引之间的元素构成的子集（切分数组，`start:end` 被称为 `slice` 表达式）。所以 `slice1[0]` 就等于 `arr1[start]`。这可以在 `arr1` 被填充前就定义好。

如果某个人写：`var slice1 []type = arr1[:]` 那么 slice1 就等于完整的 arr1 数组（所以这种表示方式是 `arr1[0:len(arr1)]` 的一种缩写）。另外一种表述方式是：`slice1 = &arr1`。

`arr1[2:]` 和 `arr1[2:len(arr1)]` 相同，都包含了数组从第三个到最后的所有元素。

`arr1[:3]` 和 `arr1[0:3]` 相同，包含了从第一个到第三个元素（不包括第四个）。

如果你想去掉 `slice1` 的最后一个元素，只要 `slice1 = slice1[:len(slice1)-1]`。

一个由数字 1、2、3 组成的切片可以这么生成：`s := [3]int{1,2,3}[:]`(注: 应先用`s := [3]int{1, 2, 3}`生成数组, 再使用`s[:]`转成切片) 甚至更简单的 `s := []int{1,2,3}`。

`s2 := s[:]` 是用切片组成的切片，拥有相同的元素，但是仍然指向相同的相关数组。

一个切片 `s` 可以这样扩展到它的大小上限：`s = s[:cap(s)]`，如果再扩大的话就会导致运行时错误（参见第 7.7 节）。

对于每一个切片（包括 `string`），以下状态总是成立的：

```go
s == s[:i] + s[i:] // i是一个整数且: 0 <= i <= len(s)
len(s) <= cap(s)
```

切片也可以用类似数组的方式初始化：`var x = []int{2, 3, 5, 7, 11}`。这样就创建了一个长度为 5 的数组并且创建了一个相关切片。

切片在内存中的组织方式实际上是一个有 3 个域的结构体：指向相关数组的指针，切片长度以及切片容量。下图给出了一个长度为 2，容量为 4 的切片y。

- `y[0] = 3` 且 `y[1] = 5`。
- 切片 `y[0:4]` 由 元素 3，5，7 和 11 组成。

![](images/7.2_fig7.2.png?raw=true)

示例 7.7 [array_slices.go](examples/chapter_7/array_slices.go)

```go
package main
import "fmt"

func main() {
	var arr1 [6]int
	var slice1 []int = arr1[2:5] // item at index 5 not included!

	// load the array with integers: 0,1,2,3,4,5
	for i := 0; i < len(arr1); i++ {
		arr1[i] = i
	}

	// print the slice
	for i := 0; i < len(slice1); i++ {
		fmt.Printf("Slice at %d is %d\n", i, slice1[i])
	}

	fmt.Printf("The length of arr1 is %d\n", len(arr1))
	fmt.Printf("The length of slice1 is %d\n", len(slice1))
	fmt.Printf("The capacity of slice1 is %d\n", cap(slice1))

	// grow the slice
	slice1 = slice1[0:4]
	for i := 0; i < len(slice1); i++ {
		fmt.Printf("Slice at %d is %d\n", i, slice1[i])
	}
	fmt.Printf("The length of slice1 is %d\n", len(slice1))
	fmt.Printf("The capacity of slice1 is %d\n", cap(slice1))

	// grow the slice beyond capacity
	//slice1 = slice1[0:7 ] // panic: runtime error: slice bound out of range
}
```

输出：  

	Slice at 0 is 2  
	Slice at 1 is 3  
	Slice at 2 is 4  
	The length of arr1 is 6  
	The length of slice1 is 3  
	The capacity of slice1 is 4  
	Slice at 0 is 2  
	Slice at 1 is 3  
	Slice at 2 is 4  
	Slice at 3 is 5  
	The length of slice1 is 4  
	The capacity of slice1 is 4  

如果 `s2` 是一个 `slice`，你可以将 `s2` 向后移动一位 `s2 = s2[1:]`，但是末尾没有移动。切片只能向后移动，`s2 = s2[-1:]` 会导致编译错误。切片不能被重新分片以获取数组的前一个元素。

> **注意** 绝对不要用指针指向 `slice`。切片本身已经是一个引用类型，所以它本身就是一个指针!!

问题 7.2： 给定切片 `b:= []byte{'g', 'o', 'l', 'a', 'n', 'g'}`，那么 `b[1:4]`、`b[:2]`、`b[2:]` 和 `b[:]` 分别是什么？

### 7.2.2 将切片传递给函数

如果你有一个函数需要对数组做操作，你可能总是需要把参数声明为切片。当你调用该函数时，把数组分片，创建为一个 切片引用并传递给该函数。这里有一个计算数组元素和的方法:

```go
func sum(a []int) int {
	s := 0
	for i := 0; i < len(a); i++ {
		s += a[i]
	}
	return s
}

func main() {
	var arr = [5]int{0, 1, 2, 3, 4}
	sum(arr[:])
}
```

### 7.2.3 用 make() 创建一个切片

当相关数组还没有定义时，我们可以使用 `make()` 函数来创建一个切片 同时创建好相关数组：`var slice1 []type = make([]type, len)`。

也可以简写为 `slice1 := make([]type, len)`，这里 `len` 是数组的长度并且也是 `slice` 的初始长度。

所以定义 `s2 := make([]int, 10)`，那么 `cap(s2) == len(s2) == 10`。

`make` 接受 2 个参数：元素的类型以及切片的元素个数。

如果你想创建一个 `slice1`，它不占用整个数组，而只是占用以 `len` 为个数个项，那么只要：`slice1 := make([]type, len, cap)`。

`make` 的使用方式是：`func make([]T, len, cap)`，其中 cap 是可选参数。

所以下面两种方法可以生成相同的切片:

```go
make([]int, 50, 100)
new([100]int)[0:50]
```

下图描述了使用 `make` 方法生成的切片的内存结构：![](images/7.2_fig7.2.1.png?raw=true)

示例 7.8 [make_slice.go](examples/chapter_7/make_slice.go)

```go
package main
import "fmt"

func main() {
	var slice1 []int = make([]int, 10)
	// load the array/slice:
	for i := 0; i < len(slice1); i++ {
		slice1[i] = 5 * i
	}

	// print the slice:
	for i := 0; i < len(slice1); i++ {
		fmt.Printf("Slice at %d is %d\n", i, slice1[i])
	}
	fmt.Printf("\nThe length of slice1 is %d\n", len(slice1))
	fmt.Printf("The capacity of slice1 is %d\n", cap(slice1))
}
```

输出：  

	Slice at 0 is 0  
	Slice at 1 is 5  
	Slice at 2 is 10  
	Slice at 3 is 15  
	Slice at 4 is 20  
	Slice at 5 is 25  
	Slice at 6 is 30  
	Slice at 7 is 35  
	Slice at 8 is 40  
	Slice at 9 is 45  
	
	The length of slice1 is 10  
	The capacity of slice1 is 10  

因为字符串是纯粹不可变的字节数组，它们也可以被切分成 切片。

练习 7.4： `fobinacci_funcarray.go`: 为练习 7.3 写一个新的版本，主函数调用一个使用序列个数作为参数的函数，该函数返回一个大小为序列个数的 `Fibonacci` 切片。

```go
package main

import "fmt"

var term = 15

func main() {
	result := fibarray(term)
	for ix, fib := range result {
		fmt.Printf("The %d-th Fibonacci number is: %d\n", ix, fib)
	}
}

func fibarray(term int) []int {
	farr := make([]int, term)
	farr[0], farr[1] = 1, 1

	for i := 2; i < term; i++ {
		farr[i] = farr[i-1] + farr[i-2]
	}
	return farr
}
```

### 7.2.4 new() 和 make() 的区别

看起来二者没有什么区别，都在堆上分配内存，但是它们的行为不同，适用于不同的类型。

- `new(T)` 为每个新的类型T分配一片内存，初始化为 `0` 并且返回类型为`*T`的内存地址：这种方法 **返回一个指向类型为 T，值为 0 的地址的指针**，它适用于值类型如*数组*和*结构体*（参见第 10 章）；它相当于 `&T{}`。
- `make(T)` **返回一个类型为 T 的初始值**，它只适用于3种内建的引用类型：*切片*、*map* 和 *channel*（参见第 8 章，第 13 章）。

换言之，`new` 函数分配内存，`make` 函数初始化；下图给出了区别：

![](images/7.2_fig7.3.png?raw=true)

在图 7.3 的第一幅图中：

```go
var p *[]int = new([]int) // *p == nil; with len and cap 0
p := new([]int)
```

在第二幅图中， `p := make([]int, 0)` ，*切片*已经被初始化，但是指向一个空的数组。

以上两种方式实用性都不高。下面的方法：

```go
var v []int = make([]int, 10, 50)
```

或者
```go
v := make([]int, 10, 50)
```

这样分配一个有 50 个 `int` 值的数组，并且创建了一个长度为 10，容量为 50 的 切片 `v`，该 切片 指向数组的前 10 个元素。

**问题 7.3** 给定 `s := make([]byte, 5)`，`len(s)` 和 `cap(s)` 分别是多少？`s = s[2:4]`，`len(s)` 和 `cap(s)` 又分别是多少？

**问题 7.4** 假设 `s1 := []byte{'p', 'o', 'e', 'm'}` 且 `s2 := s1[2:]`，`s2` 的值是多少？如果我们执行 `s2[1] = 't'`，`s1` 和 `s2` 现在的值又分别是多少？

> 译者注：如何理解new、make、slice、map、channel的关系
>
> *1.slice、map以及channel都是 GoLang 内建的一种引用类型，三者在内存中存在多个组成部分，
> 需要对内存组成部分初始化后才能使用，而make就是对三者进行初始化的一种操作方式*
>
> *2. new 获取的是存储指定变量内存地址的一个变量，对于变量内部结构并不会执行响应的初始化操作，
> 所以slice、map、channel需要make进行初始化并获取对应的内存地址，而非new简单的获取内存地址*

### 7.2.5 多维 切片

和数组一样，切片通常也是一维的，但是也可以由一维组合成高维。通过分片的分片（或者切片的数组），长度可以任意动态变化，所以 Go 语言的多维切片可以任意切分。而且，内层的切片必须单独分配（通过 `make` 函数）。

### 7.2.6 bytes 包

类型 `[]byte` 的切片十分常见，Go 语言有一个 `bytes` 包专门用来解决这种类型的操作方法。

`bytes` 包和字符串包十分类似（参见第 4.7 节）。而且它还包含一个十分有用的类型 `Buffer`:

```go
import "bytes"

type Buffer struct {
	...
}
```

这是一个长度可变的 `bytes` 的 `buffer`，提供 `Read` 和 `Write` 方法，因为读写长度未知的 `bytes` 最好使用 `buffer`。

`Buffer` 可以这样定义：`var buffer bytes.Buffer`。

或者使用 `new` 获得一个指针：`var r *bytes.Buffer = new(bytes.Buffer)`。

或者通过函数：`func NewBuffer(buf []byte) *Buffer`，创建一个 `Buffer` 对象并且用 `buf` 初始化好；`NewBuffer` 最好用在从 `buf` 读取的时候使用。

**通过 buffer 串联字符串**

类似于 Java 的 `StringBuilder` 类。

在下面的代码段中，我们创建一个 `buffer`，通过 `buffer.WriteString(s)` 方法将字符串 `s` 追加到后面，最后再通过 `buffer.String()` 方法转换为 `string`：

```go
var buffer bytes.Buffer

for {
	if s, ok := getNextString(); ok { //method getNextString() not shown here
		buffer.WriteString(s)
	} else {
		break
	}
}
fmt.Print(buffer.String(), "\n")
```

这种实现方式比使用 `+=` 要更节省内存和 CPU，尤其是要串联的字符串数目特别多的时候。

**练习 7.5** 给定切片 `sl`，将一个 `[]byte` 数组追加到 `sl` 后面。写一个函数 `Append(slice, data []byte) []byte`，该函数在 `sl` 不能存储更多数据的时候自动扩容。

```go
package main

import "fmt"

func main() {
	sl := []byte{'a', 'u', 'z', 's'}
	data := []byte{'g', 'o', 'l', 'a', 'n', 'g'}
	sliceVal := Append(sl, data)
	fmt.Printf("slice = %c \n", sliceVal)
}

func Append(slice, data []byte) []byte {
	lengthNewSlice := len(slice) + len(data)
	capNewSlice := cap(slice)
	if lengthNewSlice > cap(slice) {
		capNewSlice = lengthNewSlice
	}
	newSlice := make([]byte, lengthNewSlice, capNewSlice)

	copy(newSlice, slice)
	for dataKey, item := range data {
		newSlice[dataKey+len(slice)] = item
	}
	return newSlice
}
/* Output:
slice = [a u z s g o l a n g]
*/
```

**练习 7.6** 把一个缓存 `buf` 分片成两个切片：第一个是前 n 个 bytes，后一个是剩余的，用一行代码实现。

```go
a, b := buf[:n], buf[n:]
```

## 7.3 For-range 结构

这种构建方法可以应用于数组和切片:

```go
for ix, value := range slice1 {
	...
}
```

第一个返回值 `ix` 是数组或者切片的索引，第二个是在该索引位置的值；他们都是仅在 `for` 循环内部可见的局部变量。`value` 只是 `slice1` 某个索引位置的值的一个拷贝，不能用来修改 `slice1` 该索引位置的值。

示例 7.9 [slices_forrange.go](examples/chapter_7/slices_forrange.go)

```go
package main

import "fmt"

func main() {
	var slice1 []int = make([]int, 4)

	slice1[0] = 1
	slice1[1] = 2
	slice1[2] = 3
	slice1[3] = 4

	for ix, value := range slice1 {
		fmt.Printf("Slice at %d is: %d\n", ix, value)
	}
}
```

示例 7.10 [slices_forrange2.go](examples/chapter_7/slices_forrange2.go)

```go
package main
import "fmt"

func main() {
	seasons := []string{"Spring", "Summer", "Autumn", "Winter"}
	for ix, season := range seasons {
		fmt.Printf("Season %d is: %s\n", ix, season)
	}

	var season string
	for _, season = range seasons {
		fmt.Printf("%s\n", season)
	}
}
```

`slices_forrange2.go` 给出了一个关于字符串的例子， `_` 可以用于忽略索引。

如果你只需要索引，你可以忽略第二个变量，例如：

```go
for ix := range seasons {
	fmt.Printf("%d", ix)
}
// Output: 0 1 2 3
```

如果你需要修改 `seasons[ix]` 的值可以使用这个版本。

**多维切片下的 for-range：**

通过计算行数和矩阵值可以很方便的写出如（参考第 7.1.3 节）的 for 循环来，例如（参考第 7.5 节的例子 `multidim_array.go`）：

```go
for row := range screen {
	for column := range screen[row] {
		screen[row][column] = 1
	}
}
```

**问题 7.5** 假设我们有如下数组：`items := [...]int{10, 20, 30, 40, 50}`

a) 如果我们写了如下的 for 循环，那么执行完 for 循环后的 `items` 的值是多少？如果你不确定的话可以测试一下:)

```go
for _, item := range items {
	item *= 2
}
```

b) 如果 a) 无法正常工作，写一个 for 循环让值可以 double。

**问题 7.6** 通过使用省略号操作符 `...` 来实现累加方法。

**练习 7.7** `sum_array.go`

a) 写一个 `Sum` 函数，传入参数为一个 32 位 `float` 数组成的数组 `arrF`，返回该数组的所有数字和。

如果把数组修改为切片的话代码要做怎样的修改？如果用切片形式方法实现不同长度数组的的和呢？

b) 写一个 `SumAndAverage` 方法，返回两个 `int` 和 `float32` 类型的未命名变量的和与平均值。

```go
package main

import "fmt"

func main() {
	// var a = [4]float32 {1.0,2.0,3.0,4.0}
	var a = []float32{1.0, 2.0, 3.0, 4.0}
	fmt.Printf("The sum of the array is: %f\n", Sum(a))
	var b = []int{1, 2, 3, 4, 5}
	sum, average := SumAndAverage(b)
	fmt.Printf("The sum of the array is: %d, and the average is: %f", sum, average)
}

/*
func Sum(a [4]float32) (sum float32) {
	for _, item := range a {
		sum += item
	}
	return
}
*/

func Sum(a []float32) (sum float32) {
	for _, item := range a {
		sum += item
	}
	return
}

func SumAndAverage(a []int) (int, float32) {
	sum := 0
	for _, item := range a {
		sum += item
	}
	return sum, float32(sum / len(a))
}
```

**练习 7.8** `min_max.go`

写一个 `minSlice` 方法，传入一个 `int` 的切片并且返回最小值，再写一个 `maxSlice` 方法返回最大值。

```go
package main

import (
	"fmt"
	"math"
)

func main() {
	sl1 := []int{78, 34, 643, 12, 90, 492, 13, 2}
	max := maxSlice(sl1)
	fmt.Printf("The maximum is %d\n", max)
	min := minSlice(sl1)
	fmt.Printf("The minimum is %d\n", min)
}

func maxSlice(sl []int) (max int) {
	for _, v := range sl {
		if v > max {
			max = v
		}
	}
	return
}

func minSlice(sl []int) (min int) {
	// min = int(^uint(0) >> 1)
	min = math.MaxInt32
	for _, v := range sl {
		if v < min {
			min = v
		}
	}
	return
}

/* Output:
The maximum is 643
The minimum is 2
*/
```


## 7.4 切片重组（reslice）

我们已经知道切片创建的时候通常比相关数组小，例如：

```go
slice1 := make([]type, start_length, capacity)
```

其中 `start_length` 作为切片初始长度而 `capacity` 作为相关数组的长度。

这么做的好处是我们的切片在达到容量上限后可以扩容。改变切片长度的过程称之为切片重组 **reslicing**，做法如下：`slice1 = slice1[0:end]`，其中 end 是新的末尾索引（即长度）。

将切片扩展 1 位可以这么做：

```go
sl = sl[0:len(sl)+1]
```

切片可以反复扩展直到占据整个相关数组。

示例 7.11 [reslicing.go](examples/chapter_7/reslicing.go)

```go
package main
import "fmt"

func main() {
	slice1 := make([]int, 0, 10)
	// load the slice, cap(slice1) is 10:
	for i := 0; i < cap(slice1); i++ {
		slice1 = slice1[0:i+1]
		slice1[i] = i
		fmt.Printf("The length of slice is %d\n", len(slice1))
	}

	// print the slice:
	for i := 0; i < len(slice1); i++ {
		fmt.Printf("Slice at %d is %d\n", i, slice1[i])
	}
}
```

输出结果：

	The length of slice is 1
	The length of slice is 2
	The length of slice is 3
	The length of slice is 4
	The length of slice is 5
	The length of slice is 6
	The length of slice is 7
	The length of slice is 8
	The length of slice is 9
	The length of slice is 10
	Slice at 0 is 0
	Slice at 1 is 1
	Slice at 2 is 2
	Slice at 3 is 3
	Slice at 4 is 4
	Slice at 5 is 5
	Slice at 6 is 6
	Slice at 7 is 7
	Slice at 8 is 8
	Slice at 9 is 9

另一个例子：

```go
var ar = [10]int{0,1,2,3,4,5,6,7,8,9}
var a = ar[5:7] // reference to subarray {5,6} - len(a) is 2 and cap(a) is 5
```

将 a 重新分片：

```go
a = a[0:4] // ref of subarray {5,6,7,8} - len(a) is now 4 but cap(a) is still 5
```

**问题 7.7**

1) 如果 a 是一个切片，那么 `a[n:n]` 的长度是多少？

2) `a[n:n+1]` 的长度又是多少？          

## 7.5 切片的复制与追加

如果想增加切片的容量，我们必须创建一个新的更大的切片并把原分片的内容都拷贝过来。下面的代码描述了从拷贝切片的 `copy` 函数和向切片追加新元素的 `append` 函数。

示例 7.12 [copy_append_slice.go](examples/chapter_7/copy_append_slice.go)

```go
package main
import "fmt"

func main() {
	sl_from := []int{1, 2, 3}
	sl_to := make([]int, 10)

	n := copy(sl_to, sl_from)
	fmt.Println(sl_to)
	fmt.Printf("Copied %d elements\n", n) // n == 3

	sl3 := []int{1, 2, 3}
	sl3 = append(sl3, 4, 5, 6)
	fmt.Println(sl3)
}
```

`func append(s[]T, x ...T) []T` 其中 `append` 方法将 0 个或多个具有相同类型 `s` 的元素追加到切片后面并且返回新的切片；追加的元素必须和原切片的元素同类型。如果 `s` 的容量不足以存储新增元素，`append` 会分配新的切片来保证已有切片元素和新增元素的存储。因此，返回的切片可能已经指向一个不同的相关数组了。`append` 方法总是返回成功，除非系统内存耗尽了。

如果你想将切片 `y` 追加到切片 `x` 后面，只要将第二个参数扩展成一个列表即可：`x = append(x, y...)`。

**注意**： `append` 在大多数情况下很好用，但是如果你想完全掌控整个追加过程，你可以实现一个这样的 `AppendByte` 方法：

```go
func AppendByte(slice []byte, data ...byte) []byte {
	m := len(slice)
	n := m + len(data)
	if n > cap(slice) { // if necessary, reallocate
		// allocate double what's needed, for future growth.
		newSlice := make([]byte, (n+1)*2)
		copy(newSlice, slice)
		slice = newSlice
	}
	slice = slice[0:n]
	copy(slice[m:n], data)
	return slice
}
```

`func copy(dst, src []T) int` `copy` 方法将类型为 T 的切片从源地址 `src` 拷贝到目标地址 `dst`，覆盖 `dst` 的相关元素，并且返回拷贝的元素个数。源地址和目标地址可能会有重叠。拷贝个数是 `src` 和 `dst` 的长度最小值。如果 `src` 是字符串那么元素类型就是 `byte`。如果你还想继续使用 `src`，在拷贝结束后执行 `src = dst`。

**练习 7.9**

给定一个slice`s []int` 和一个 `int` 类型的因子factor，扩展 s 使其长度为 `len(s) * factor`。

```go
package main

import "fmt"

var s []int

func main() {
	s = []int{1, 2, 3}
	fmt.Println("The length of s before enlarging is:", len(s))
	fmt.Println(s)
	s = enlarge(s, 5)
	fmt.Println("The length of s after enlarging is:", len(s))
	fmt.Println(s)
}

func enlarge(s []int, factor int) []int {
	ns := make([]int, len(s)*factor)
	// fmt.Println("The length of ns  is:", len(ns))
	copy(ns, s)
	//fmt.Println(ns)
	s = ns
	//fmt.Println(s)
	//fmt.Println("The length of s after enlarging is:", len(s))
	return s
}
```

**练习 7.10**

用顺序函数过滤容器：`s` 是前 10 个整型的切片。构造一个函数 `Filter`，第一个参数是 `s`，第二个参数是一个 `fn func(int) bool`，返回满足函数 `fn` 的元素切片。通过 `fn` 测试方法测试当整型值是偶数时的情况。

```go
package main

import (
	"fmt"
)

func main() {
	s := []int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9}
	s = Filter(s, even)
	fmt.Println(s)
}

// Filter returns a new slice holding only
// the elements of s that satisfy f()
func Filter(s []int, fn func(int) bool) []int {
	var p []int // == nil
	for _, i := range s {
		if fn(i) {
			p = append(p, i)
		}
	}
	return p
}

func even(n int) bool {
	if n%2 == 0 {
		return true
	}
	return false
}
/* [0 2 4 6 8] */
```

**练习 7.11**

写一个函数 `InsertStringSlice` 将切片插入到另一个切片的指定位置。

```go
// insert_slice.go
package main

import (
	"fmt"
)

func main() {
	s := []string{"M", "N", "O", "P", "Q", "R"}
	in := []string{"A", "B", "C"}
	res := InsertStringSlice(s, in, 0) // at the front
	fmt.Println(res)                   // [A B C M N O P Q R]
	res = InsertStringSlice(s, in, 3)  // [M N O A B C P Q R]
	fmt.Println(res)
}

func InsertStringSlice(slice, insertion []string, index int) []string {
	result := make([]string, len(slice)+len(insertion))
	at := copy(result, slice[:index])
	at += copy(result[at:], insertion)
	copy(result[at:], slice[index:])
	return result
}
```

**练习 7.12**

写一个函数 `RemoveStringSlice` 将从 `start` 到 `end` 索引的元素从切片 中移除。

```go
// remove_slice.go
package main

import (
	"fmt"
)

func main() {
	s := []string{"M", "N", "O", "P", "Q", "R"}
	res := RemoveStringSlice(s, 2, 4)
	fmt.Println(res) // [M N Q R]
}

func RemoveStringSlice(slice []string, start, end int) []string {
	result := make([]string, len(slice)-(end-start)-1)
	at := copy(result, slice[:start])
	copy(result[at:], slice[end+1:])
	return result
}
```

## 7.6 字符串、数组和切片的应用

### 7.6.1 从字符串生成字节切片

假设 `s` 是一个字符串（本质上是一个字节数组），那么就可以直接通过 `c := []byte(s)` 来获取一个字节的切片 `c`。另外，您还可以通过 `copy` 函数来达到相同的目的：`copy(dst []byte, src string)`。

同样的，还可以使用 `for-range` 来获得每个元素（Listing 7.13—`for_string.go`）：

```go
package main

import "fmt"

func main() {
    s := "\u00ff\u754c"
    for i, c := range s {
        fmt.Printf("%d:%c ", i, c)
    }
}
```

输出：

    0:ÿ 2:界

我们知道，Unicode 字符会占用 2 个字节，有些甚至需要 3 个或者 4 个字节来进行表示。如果发现错误的 UTF8 字符，则该字符会被设置为 `U+FFFD` 并且索引向前移动一个字节。和字符串转换一样，您同样可以使用 `c := []int32(s)` 语法，这样切片中的每个 `int` 都会包含对应的 Unicode 代码，因为字符串中的每次字符都会对应一个整数。类似的，您也可以将字符串转换为元素类型为 `rune` 的切片：`r := []rune(s)`。

可以通过代码 `len([]int32(s))` 来获得字符串中字符的数量，但使用 `utf8.RuneCountInString(s)` 效率会更高一点。(参考[count_characters.go](exercises/chapter_4/count_characters.go))

您还可以将一个字符串追加到某一个字节切片的尾部：

```go
var b []byte
var s string
b = append(b, s...)
```

### 7.6.2 获取字符串的某一部分

使用 `substr := str[start:end]` 可以从字符串 `str` 获取到从索引 `start` 开始到 `end-1` 位置的子字符串。同样的，`str[start:]` 则表示获取从 `start` 开始到 `len(str)-1` 位置的子字符串。而 `str[:end]` 表示获取从 `0` 开始到 `end-1` 的子字符串。

### 7.6.3 字符串和切片的内存结构

在内存中，一个字符串实际上是一个双字结构，即一个指向实际数据的指针和记录字符串长度的整数（见图 7.4）。因为指针对用户来说是完全不可见，因此我们可以依旧把字符串看做是一个值类型，也就是一个字符数组。

字符串 `string s = "hello"` 和子字符串 `t = s[2:3]` 在内存中的结构可以用下图表示：

![](images/7.6_fig7.4.png)

### 7.6.4 修改字符串中的某个字符

Go 语言中的字符串是不可变的，也就是说 `str[index]` 这样的表达式是不可以被放在等号左侧的。如果尝试运行 `str[i] = 'D'` 会得到错误：`cannot assign to str[i]`。

因此，您必须先将字符串转换成字节数组，然后再通过修改数组中的元素值来达到修改字符串的目的，最后将字节数组转换回字符串格式。

例如，将字符串 "hello" 转换为 "cello"：

```go
s := "hello"
c := []byte(s)
c[0] = 'c'
s2 := string(c) // s2 == "cello"
```

所以，您可以通过操作切片来完成对字符串的操作。

### 7.6.5 字节数组对比函数

下面的 `Compare` 函数会返回两个字节数组字典顺序的整数对比结果，即 `0 if a == b, -1 if a < b, 1 if a > b`。

```go
func Compare(a, b[]byte) int {
    for i:=0; i < len(a) && i < len(b); i++ {
        switch {
        case a[i] > b[i]:
            return 1
        case a[i] < b[i]:
            return -1
        }
    }
    // 数组的长度可能不同
    switch {
    case len(a) < len(b):
        return -1
    case len(a) > len(b):
        return 1
    }
    return 0 // 数组相等
}
```

### 7.6.6 搜索及排序切片和数组

标准库提供了 `sort` 包来实现常见的搜索和排序操作。您可以使用 `sort` 包中的函数 `func Ints(a []int)` 来实现对 `int` 类型的切片排序。例如 `sort.Ints(arri)`，其中变量 `arri` 就是需要被升序排序的数组或切片。为了检查某个数组是否已经被排序，可以通过函数 `IntsAreSorted(a []int) bool` 来检查，如果返回 `true` 则表示已经被排序。

类似的，可以使用函数 `func Float64s(a []float64)` 来排序 `float64` 的元素，或使用函数 `func Strings(a []string)` 排序字符串元素。

想要在数组或切片中搜索一个元素，该数组或切片必须先被排序（因为标准库的搜索算法使用的是二分法）。然后，您就可以使用函数 `func SearchInts(a []int, n int) int` 进行搜索，并返回对应结果的索引值。

当然，还可以搜索 `float64` 和字符串：

```go
func SearchFloat64s(a []float64, x float64) int
func SearchStrings(a []string, x string) int
```

您可以通过查看 [官方文档](http://golang.org/pkg/sort/) 来获取更详细的信息。

这就是如何使用 `sort` 包的方法，我们会在第 11.6 节对它的细节进行深入，并实现一个属于我们自己的版本。

### 7.6.7 append 函数常见操作

我们在第 7.5 节提到的 `append` 非常有用，它能够用于各种方面的操作：

1. 将切片 `b` 的元素追加到切片 `a` 之后：`a = append(a, b...)`
2. 复制切片 `a` 的元素到新的切片 `b` 上：

    ```go
    b = make([]T, len(a))
    copy(b, a)
    ```

3. 删除位于索引 `i` 的元素：`a = append(a[:i], a[i+1:]...)`
4. 切除切片 `a` 中从索引 `i` 至 `j` 位置的元素：`a = append(a[:i], a[j:]...)`
5. 为切片 `a` 扩展 `j` 个元素长度：`a = append(a, make([]T, j)...)`
6. 在索引 `i` 的位置插入元素 `x`：`a = append(a[:i], append([]T{x}, a[i:]...)...)`
7. 在索引 `i` 的位置插入长度为 `j` 的新切片：`a = append(a[:i], append(make([]T, j), a[i:]...)...)`
8. 在索引 `i` 的位置插入切片 `b` 的所有元素：`a = append(a[:i], append(b, a[i:]...)...)`
9. 取出位于切片 `a` 最末尾的元素 `x`：`x, a = a[len(a)-1], a[:len(a)-1]`
10. 将元素 `x` 追加到切片 `a`：`a = append(a, x)`

因此，您可以使用切片和 `append` 操作来表示任意可变长度的序列。

从数学的角度来看，切片相当于向量，如果需要的话可以定义一个向量作为切片的别名来进行操作。

如果您需要更加完整的方案，可以学习一下 Eleanor McHugh 编写的几个包：[slices](http://github.com/feyeleanor/slices)、[chain](http://github.com/feyeleanor/chain) 和 [lists](http://github.com/feyeleanor/lists)。

### 7.6.8 切片和垃圾回收

切片的底层指向一个数组，该数组的实际容量可能要大于切片所定义的容量。只有在没有任何切片指向的时候，底层的数组内存才会被释放，这种特性有时会导致程序占用多余的内存。

**示例** 函数 `FindDigits` 将一个文件加载到内存，然后搜索其中所有的数字并返回一个切片。

```go
var digitRegexp = regexp.MustCompile("[0-9]+")

func FindDigits(filename string) []byte {
    b, _ := ioutil.ReadFile(filename)
    return digitRegexp.Find(b)
}
```

这段代码可以顺利运行，但返回的 `[]byte` 指向的底层是整个文件的数据。只要该返回的切片不被释放，垃圾回收器就不能释放整个文件所占用的内存。换句话说，一点点有用的数据却占用了整个文件的内存。

想要避免这个问题，可以通过拷贝我们需要的部分到一个新的切片中：

```go
func FindDigits(filename string) []byte {
   b, _ := ioutil.ReadFile(filename)
   b = digitRegexp.Find(b)
   c := make([]byte, len(b))
   copy(c, b)
   return c
}
```
事实上，上面这段代码只能找到第一个匹配正则表达式的数字串。要想找到所有的数字，可以尝试下面这段代码：
```go
func FindFileDigits(filename string) []byte {
   fileBytes, _ := ioutil.ReadFile(filename)
   b := digitRegexp.FindAll(fileBytes, len(fileBytes))
   c := make([]byte, 0)
   for _, bytes := range b {
      c = append(c, bytes...)
   }
   return c
}
```

**练习 7.12**

编写一个函数，要求其接受两个参数，原始字符串 `str` 和分割索引 `i`，然后返回两个分割后的字符串。

```go
package main

import (
	"fmt"
)

func main() {
	rawString := "Google"
	index := 3
	sp1, sp2 := splitStringbyIndex(rawString, index)
	fmt.Printf("The string %s split at position %d is: %s / %s\n", rawString, index, sp1, sp2)
}

func splitStringbyIndex(str string, i int) (sp1, sp2 string) {
	rawStrSlice := []byte(str)
	sp1 = string(rawStrSlice[:i])
	sp2 = string(rawStrSlice[i:])
	return
}
```

**练习 7.13**

假设有字符串 `str`，那么 `str[len(str)/2:] + str[:len(str)/2]` 的结果是什么？

```go
package main

import "fmt"

func main() {
	str := "Google"
	str2 := Split2(str)
	fmt.Printf("The string %s transformed is: %s\n", str, str2)
}

func Split2(s string) string {
	mid := len(s) / 2
	return s[mid:] + s[:mid]
}

// Output: The string Google transformed is: gleGoo
```

**练习 7.14**

编写一个程序，要求能够反转字符串，即将 “Google” 转换成 “elgooG”（提示：使用 `[]byte` 类型的切片）。

如果您使用两个切片来实现反转，请再尝试使用一个切片（提示：使用交换法）。

如果您想要反转 Unicode 编码的字符串，请使用 `[]int32` 类型的切片。

```go
package main

import "fmt"

func reverse(s string) string {
	runes := []rune(s)
	n, h := len(runes), len(runes)/2
	for i := 0; i < h; i++ {
		runes[i], runes[n-1-i] = runes[n-1-i], runes[i]
	}
	return string(runes)
}

func main() {
	// reverse a string:
	str := "Google"
	sl := []byte(str)
	var rev [100]byte
	j := 0
	for i := len(sl) - 1; i >= 0; i-- {
		rev[j] = sl[i]
		j++
	}
	str_rev := string(rev[:])
	fmt.Printf("The reversed string is -%s-\n", str_rev)
	// variant: "in place" using swapping
	str2 := "Google"
	sl2 := []byte(str2)
	for i, j := 0, len(sl2)-1; i < j; i, j = i+1, j-1 {
		sl2[i], sl2[j] = sl2[j], sl2[i]
	}
	fmt.Printf("The reversed string is -%s-\n", string(sl2))
	// variant: using [] int for runes (necessary for Unicode-strings!):
	s := "My Test String!"
	fmt.Println(s, " --> ", reverse(s))
}

/* Output:
The reversed string is -elgooG-
The reversed string is -elgooG-
My Test String!  -->  !gnirtS tseT yM
*/
```

**练习 7.15**

编写一个程序，要求能够遍历一个字符数组，并将当前字符和前一个字符不相同的字符拷贝至另一个数组。

```go
// Q29_uniq.go
package main

import (
	"fmt"
)

var arr []byte = []byte{'a', 'b', 'a', 'a', 'a', 'c', 'd', 'e', 'f', 'g'}

func main() {
	arru := make([]byte, len(arr)) // this will contain the unique items
	ixu := 0                       // index in arru
	tmp := byte(0)
	for _, val := range arr {
		if val != tmp {
			arru[ixu] = val
			fmt.Printf("%c ", arru[ixu])
			ixu++
		}
		tmp = val
	}
	// fmt.Println(arru)
}
```

**练习 7.16**

编写一个程序，使用冒泡排序的方法排序一个包含整数的切片（算法的定义可参考 [维基百科](http://en.wikipedia.org/wiki/Bubble_sort)）。

```go
// Q14_Bubblesort.go
package main

import (
	"fmt"
)

func main() {
	sla := []int{2, 6, 4, -10, 8, 89, 12, 68, -45, 37}
	fmt.Println("before sort: ", sla)
	// sla is passed via call by value, but since sla is a reference type
	// the underlying slice is array is changed (sorted)
	bubbleSort(sla)
	fmt.Println("after sort:  ", sla)
}

func bubbleSort(sl []int) {
	// passes through the slice:
	for pass := 1; pass < len(sl); pass++ {
		// one pass:
		for i := 0; i < len(sl)-pass; i++ { // the bigger value 'bubbles up' to the last position
			if sl[i] > sl[i+1] {
				sl[i], sl[i+1] = sl[i+1], sl[i]
			}
		}
	}
}
```

**练习 7.17**

在函数式编程语言中，一个 `map-function` 是指能够接受一个函数原型和一个列表，并使用列表中的值依次执行函数原型，公式为：`map ( F(), (e1,e2, . . . ,en) ) = ( F(e1), F(e2), ... F(en) )`。

编写一个函数 `mapFunc` 要求接受以下 2 个参数：

- 一个将整数乘以 10 的函数
- 一个整数列表

最后返回保存运行结果的整数列表。

```go
package main

import "fmt"

func main() {
	list := []int{0, 1, 2, 3, 4, 5, 6, 7}
	mf := func(i int) int {
		return i * 10
	}
	/*
		result := mapFunc(mf, list)
		for _, v := range result {
			fmt.Println(v)
		}
	*/
	println()
	// shorter:
	fmt.Printf("%v", mapFunc(mf, list))
}

func mapFunc(mf func(int) int, list []int) []int {
	result := make([]int, len(list))
	for ix, v := range list {
		result[ix] = mf(v)
	}
	/*
		for ix := 0; ix<len(list); ix++ {
			result[ix] = mf(list[ix])
		}
	*/
	return result
}
```
