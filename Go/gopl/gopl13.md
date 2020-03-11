# 第十三章　底层编程

Go语言的设计包含了诸多安全策略，限制了可能导致程序运行出错的用法。编译时类型检查可以发现大多数类型不匹配的操作，例如两个字符串做减法的错误。字符串、map、slice和chan等所有的内置类型，都有严格的类型转换规则。

对于无法静态检测到的错误，例如数组访问越界或使用空指针，运行时动态检测可以保证程序在遇到问题的时候立即终止并打印相关的错误信息。自动内存管理（垃圾内存自动回收）可以消除大部分野指针和内存泄漏相关的问题。

Go语言的实现刻意隐藏了很多底层细节。我们无法知道一个结构体真实的内存布局，也无法获取一个运行时函数对应的机器码，也无法知道当前的`goroutine`是运行在哪个操作系统线程之上。事实上，Go语言的调度器会自己决定是否需要将某个`goroutine`从一个操作系统线程转移到另一个操作系统线程。一个指向变量的指针也并没有展示变量真实的地址。因为垃圾回收器可能会根据需要移动变量的内存位置，当然变量对应的地址也会被自动更新。

总的来说，Go语言的这些特性使得Go程序相比较低级的C语言来说更容易预测和理解，程序也不容易崩溃。通过隐藏底层的实现细节，也使得Go语言编写的程序具有高度的可移植性，因为语言的语义在很大程度上是独立于任何编译器实现、操作系统和CPU系统结构的（当然也不是完全绝对独立：例如`int`等类型就依赖于CPU机器字的大小，某些表达式求值的具体顺序，还有编译器实现的一些额外的限制等）。

有时候我们可能会放弃使用部分语言特性而优先选择具有更好性能的方法，例如需要与其他语言编写的库进行互操作，或者用纯Go语言无法实现的某些函数。

在本章，我们将展示如何使用`unsafe`包来摆脱Go语言规则带来的限制，讲述如何创建C语言函数库的绑定，以及如何进行系统调用。

本章提供的方法不应该轻易使用（译注：属于黑魔法，虽然功能很强大，但是也容易误伤到自己）。如果没有处理好细节，它们可能导致各种不可预测的并且隐晦的错误，甚至连有经验的C语言程序员也无法理解这些错误。使用`unsafe`包的同时也放弃了Go语言保证与未来版本的兼容性的承诺，因为它必然会有意无意中使用很多非公开的实现细节，而这些实现的细节在未来的Go语言中很可能会被改变。

要注意的是，`unsafe`包是一个采用特殊方式实现的包。虽然它可以和普通包一样的导入和使用，但它实际上是由编译器实现的。它提供了一些访问语言内部特性的方法，特别是内存布局相关的细节。将这些特性封装到一个独立的包中，是为在极少数情况下需要使用的时候，同时引起人们的注意（译注：因为看包的名字就知道使用`unsafe`包是不安全的）。此外，有一些环境因为安全的因素可能限制这个包的使用。

不过`unsafe`包被广泛地用于比较低级的包，例如`runtime`、`os`、`syscall`还有`net`包等，因为它们需要和操作系统密切配合，但是对于普通的程序一般是不需要使用unsafe包的。
## 13.1. unsafe.Sizeof, Alignof 和 Offsetof

`unsafe.Sizeof`函数返回操作数在内存中的字节大小，参数可以是任意类型的表达式，但是它并不会对表达式进行求值。一个`Sizeof`函数调用是一个对应`uintptr`类型的常量表达式，因此返回的结果可以用作数组类型的长度大小，或者用作计算其他的常量。

```Go
import "unsafe"
fmt.Println(unsafe.Sizeof(float64(0))) // "8"
```

`Sizeof`函数返回的大小只包括数据结构中固定的部分，例如字符串对应结构体中的指针和字符串长度部分，但是并不包含指针指向的字符串的内容。Go语言中非聚合类型通常有一个固定的大小，尽管在不同工具链下生成的实际大小可能会有所不同。考虑到可移植性，引用类型或包含引用类型的大小在32位平台上是4个字节，在64位平台上是8个字节。

计算机在加载和保存数据时，如果内存地址合理地对齐的将会更有效率。例如2字节大小的`int16`类型的变量地址应该是偶数，一个4字节大小的`rune`类型变量的地址应该是4的倍数，一个8字节大小的`float64`、`uint64`或64-bit指针类型变量的地址应该是8字节对齐的。但是对于再大的地址对齐倍数则是不需要的，即使是`complex128`等较大的数据类型最多也只是8字节对齐。

由于地址对齐这个因素，一个聚合类型（结构体或数组）的大小至少是所有字段或元素大小的总和，或者更大因为可能存在内存空洞。内存空洞是编译器自动添加的没有被使用的内存空间，用于保证后面每个字段或元素的地址相对于结构或数组的开始地址能够合理地对齐（译注：内存空洞可能会存在一些随机数据，可能会对用unsafe包直接操作内存的处理产生影响）。


类型                            | 大小
------------------------------- | -----------------------------
`bool`                          | 1个字节
`intN, uintN, floatN, complexN` | N/8个字节（例如float64是8个字节）
`int, uint, uintptr`            | 1个机器字
`*T`                            | 1个机器字
`string`                        | 2个机器字（data、len）
`[]T`                           | 3个机器字（data、len、cap）
`map`                           | 1个机器字
`func`                          | 1个机器字
`chan`                          | 1个机器字
`interface`                     | 2个机器字（type、value）

Go语言的规范并没有要求一个字段的声明顺序和内存中的顺序是一致的，所以理论上一个编译器可以随意地重新排列每个字段的内存位置，虽然在写作本书的时候编译器还没有这么做。下面的三个结构体虽然有着相同的字段，但是第一种写法比另外的两个需要多50%的内存。

```Go
                               // 64-bit  32-bit
struct{ bool; float64; int16 } // 3 words 4words
struct{ float64; int16; bool } // 2 words 3words
struct{ bool; int16; float64 } // 2 words 3words
```

关于内存地址对齐算法的细节超出了本书的范围，也不是每一个结构体都需要担心这个问题，不过有效的包装可以使数据结构更加紧凑（译注：未来的Go语言编译器应该会默认优化结构体的顺序，当然应该也能够指定具体的内存布局，相同讨论请参考 [Issue10014](https://github.com/golang/go/issues/10014) ），内存使用率和性能都可能会受益。

`unsafe.Alignof` 函数返回对应参数的类型需要对齐的倍数。和 `Sizeof` 类似， `Alignof` 也是返回一个常量表达式，对应一个常量。通常情况下布尔和数字类型需要对齐到它们本身的大小（最多8个字节），其它的类型对齐到机器字大小。

`unsafe.Offsetof` 函数的参数必须是一个字段 `x.f`，然后返回 `f` 字段相对于 `x` 起始地址的偏移量，包括可能的空洞。

图 13.1 显示了一个结构体变量 x 以及其在32位和64位机器上的典型的内存。灰色区域是空洞。

```Go
var x struct {
	a bool
	b int16
	c []int
}
```

下面显示了对x和它的三个字段调用unsafe包相关函数的计算结果：

![unsafe.Sizeof, Alignof 和 Offsetof - 图1](images\ch13-01-1579426631995.png)

32位系统：

```
Sizeof(x)   = 16  Alignof(x)   = 4
Sizeof(x.a) = 1   Alignof(x.a) = 1 Offsetof(x.a) = 0
Sizeof(x.b) = 2   Alignof(x.b) = 2 Offsetof(x.b) = 2
Sizeof(x.c) = 12  Alignof(x.c) = 4 Offsetof(x.c) = 4
```

64位系统：

```
Sizeof(x)   = 32  Alignof(x)   = 8
Sizeof(x.a) = 1   Alignof(x.a) = 1 Offsetof(x.a) = 0
Sizeof(x.b) = 2   Alignof(x.b) = 2 Offsetof(x.b) = 2
Sizeof(x.c) = 24  Alignof(x.c) = 8 Offsetof(x.c) = 8
```

虽然这几个函数在不安全的unsafe包，但是这几个函数调用并不是真的不安全，特别在需要优化内存空间时它们返回的结果对于理解原生的内存布局很有帮助。

## 13.2. unsafe.Pointer

大多数指针类型会写成`*T`，表示是“一个指向T类型变量的指针”。`unsafe.Pointer`是特别定义的一种指针类型（译注：类似C语言中的`void*`类型的指针），它可以包含任意类型变量的地址。当然，我们不可以直接通过`*p`来获取`unsafe.Pointer`指针指向的真实变量的值，因为我们并不知道变量的具体类型。和普通指针一样，`unsafe.Pointer`指针也是可以比较的，并且支持和nil常量比较判断是否为空指针。

一个普通的`*T`类型指针可以被转化为`unsafe.Pointer`类型指针，并且一个`unsafe.Pointer`类型指针也可以被转回普通的指针，被转回普通的指针类型并不需要和原始的`*T`类型相同。通过将`*float64`类型指针转化为`*uint64`类型指针，我们可以查看一个浮点数变量的位模式。

```Go
package math

func Float64bits(f float64) uint64 { return *(*uint64)(unsafe.Pointer(&f)) }

fmt.Printf("%#016x\n", Float64bits(1.0)) // "0x3ff0000000000000"
```

通过转为新类型指针，我们可以更新浮点数的位模式。通过位模式操作浮点数是可以的，但是更重要的意义是指针转换语法让我们可以在不破坏类型系统的前提下向内存写入任意的值。

一个`unsafe.Pointer`指针也可以被转化为`uintptr`类型，然后保存到指针型数值变量中（译注：这只是和当前指针相同的一个数字值，并不是一个指针），然后用以做必要的指针数值运算。（第三章内容，`uintptr`是一个无符号的整型数，足以保存一个地址）这种转换虽然也是可逆的，但是将`uintptr`转为`unsafe.Pointer`指针可能会破坏类型系统，因为并不是所有的数字都是有效的内存地址。

许多将`unsafe.Pointer`指针转为原生数字，然后再转回为`unsafe.Pointer`类型指针的操作也是不安全的。比如下面的例子需要将变量x的地址加上b字段地址偏移量转化为`*int16`类型指针，然后通过该指针更新`x.b`：

*gopl.io/ch13/unsafeptr*

```Go
var x struct {
	a bool
	b int16
	c []int
}

// 和 pb := &x.b 等价
pb := (*int16)(unsafe.Pointer(
	uintptr(unsafe.Pointer(&x)) + unsafe.Offsetof(x.b)))
*pb = 42
fmt.Println(x.b) // "42"
```

上面的写法尽管很繁琐，但在这里并不是一件坏事，因为这些功能应该很谨慎地使用。不要试图引入一个uintptr类型的临时变量，因为它可能会破坏代码的安全性（译注：这是真正可以体会unsafe包为何不安全的例子）。下面段代码是错误的：

```Go
// NOTE: subtly incorrect!
tmp := uintptr(unsafe.Pointer(&x)) + unsafe.Offsetof(x.b)
pb := (*int16)(unsafe.Pointer(tmp))
*pb = 42
```

产生错误的原因很微妙。有时候垃圾回收器会移动一些变量以降低内存碎片等问题。这类垃圾回收器被称为移动GC。当一个变量被移动，所有的保存该变量旧地址的指针必须同时被更新为变量移动后的新地址。从垃圾收集器的视角来看，一个`unsafe.Pointer`是一个指向变量的指针，因此当变量被移动时对应的指针也必须被更新；但是`uintptr`类型的临时变量只是一个普通的数字，所以其值不应该被改变。上面错误的代码因为引入一个非指针的临时变量`tmp`，导致垃圾收集器无法正确识别这个是一个指向变量x的指针。当第二个语句执行时，变量x可能已经被转移，这时候临时变量`tmp`也就不再是现在的`&x.b`地址。第三个向之前无效地址空间的赋值语句将彻底摧毁整个程序！

还有很多类似原因导致的错误。例如这条语句：

```Go
pT := uintptr(unsafe.Pointer(new(T))) // 提示: 错误!
```

这里并没有指针引用`new`新创建的变量，因此该语句执行完成之后，垃圾收集器有权马上回收其内存空间，所以返回的`pT`将是无效的地址。

虽然目前的Go语言实现还没有使用移动GC（译注：未来可能实现），但这不该是编写错误代码侥幸的理由：当前的Go语言实现已经有移动变量的场景。在5.2节我们提到goroutine的栈是根据需要动态增长的。当发生栈动态增长的时候，原来栈中的所有变量可能需要被移动到新的更大的栈中，所以我们并不能确保变量的地址在整个使用周期内是不变的。

在编写本文时，还没有清晰的原则来指引Go程序员，什么样的`unsafe.Pointer`和`uintptr`的转换是不安全的（参考 [Issue7192](https://github.com/golang/go/issues/7192) ）. 译注: 该问题已经关闭），因此我们强烈建议按照最坏的方式处理。将所有包含变量地址的`uintptr`类型变量当作BUG处理，同时减少不必要的`unsafe.Pointer`类型到`uintptr`类型的转换。在第一个例子中，有三个转换——字段偏移量到`uintptr`的转换和转回`unsafe.Pointer`类型的操作——所有的转换全在一个表达式完成。

当调用一个库函数，并且返回的是`uintptr`类型地址时（译注：普通方法实现的函数尽量不要返回该类型。下面例子是`reflect`包的函数，`reflect`包和`unsafe`包一样都是采用特殊技术实现的，编译器可能给它们开了后门），比如下面反射包中的相关函数，返回的结果应该立即转换为`unsafe.Pointer`以确保指针指向的是相同的变量。

```Go
package reflect

func (Value) Pointer() uintptr
func (Value) UnsafeAddr() uintptr
func (Value) InterfaceData() [2]uintptr // (index 1)
```
## 13.3. 示例: 深度相等判断

来自`reflect`包的`DeepEqual`函数可以对两个值进行深度相等判断。`DeepEqual`函数使用内建的`==`比较操作符对基础类型进行相等判断，对于复合类型则递归该变量的每个基础类型然后做类似的比较判断。因为它可以工作在任意的类型上，甚至对于一些不支持`==`操作运算符的类型也可以工作，因此在一些测试代码中广泛地使用该函数。比如下面的代码是用`DeepEqual`函数比较两个字符串数组是否相等。

```Go
func TestSplit(t *testing.T) {
	got := strings.Split("a:b:c", ":")
	want := []string{"a", "b", "c"};
	if !reflect.DeepEqual(got, want) { /* ... */ }
}
```

尽管`DeepEqual`函数很方便，而且可以支持任意的数据类型，但是它也有不足之处。例如，它将一个`nil`值的`map`和非`nil`值但是空的`map`视作不相等，同样`nil`值的`slice` 和非`nil`但是空的`slice`也视作不相等。

```Go
var a, b []string = nil, []string{}
fmt.Println(reflect.DeepEqual(a, b)) // "false"

var c, d map[string]int = nil, make(map[string]int)
fmt.Println(reflect.DeepEqual(c, d)) // "false"
```

我们希望在这里实现一个自己的`Equal`函数，用于比较类型的值。和`DeepEqual`函数类似的地方是它也是基于`slice`和`map`的每个元素进行递归比较，不同之处是它将`nil`值的`slice`（`map`类似）和非`nil`值但是空的`slice`视作相等的值。基础部分的比较可以基于`reflect`包完成，和12.3章的`Display`函数的实现方法类似。同样，我们也定义了一个内部函数`equal`，用于内部的递归比较。读者目前不用关心`seen`参数的具体含义。对于每一对需要比较的`x`和`y`，`equal`函数首先检测它们是否都有效（或都无效），然后检测它们是否是相同的类型。剩下的部分是一个巨大的`switch`分支，用于相同基础类型的元素比较。因为页面空间的限制，我们省略了一些相似的分支。

*gopl.io/ch13/equal*

```Go
func equal(x, y reflect.Value, seen map[comparison]bool) bool {
	if !x.IsValid() || !y.IsValid() {
		return x.IsValid() == y.IsValid()
	}
	if x.Type() != y.Type() {
		return false
	}

	// ...cycle check omitted (shown later)...

	switch x.Kind() {
	case reflect.Bool:
		return x.Bool() == y.Bool()
	case reflect.String:
		return x.String() == y.String()

	// ...numeric cases omitted for brevity...

	case reflect.Chan, reflect.UnsafePointer, reflect.Func:
		return x.Pointer() == y.Pointer()
	case reflect.Ptr, reflect.Interface:
		return equal(x.Elem(), y.Elem(), seen)
	case reflect.Array, reflect.Slice:
		if x.Len() != y.Len() {
			return false
		}
		for i := 0; i < x.Len(); i++ {
			if !equal(x.Index(i), y.Index(i), seen) {
				return false
			}
		}
		return true

	// ...struct and map cases omitted for brevity...
	}
	panic("unreachable")
}
```

和前面的建议一样，我们并不公开`reflect`包相关的接口，所以导出的函数需要在内部自己将变量转为`reflect.Value`类型。

```Go
// Equal reports whether x and y are deeply equal.
func Equal(x, y interface{}) bool {
	seen := make(map[comparison]bool)
	return equal(reflect.ValueOf(x), reflect.ValueOf(y), seen)
}

type comparison struct {
	x, y unsafe.Pointer
	treflect.Type
}
```

为了确保算法对于有环的数据结构也能正常退出，我们必须记录每次已经比较的变量，从而避免进入第二次的比较。Equal函数分配了一组用于比较的结构体，包含每对比较对象的地址（`unsafe.Pointer`形式保存）和类型。我们要记录类型的原因是，有些不同的变量可能对应相同的地址。例如，如果x和y都是数组类型，那么x和x[0]将对应相同的地址，y和y[0]也是对应相同的地址，这可以用于区分x与y之间的比较或x[0]与y[0]之间的比较是否进行过了。

```Go
// cycle check
if x.CanAddr() && y.CanAddr() {
	xptr := unsafe.Pointer(x.UnsafeAddr())
	yptr := unsafe.Pointer(y.UnsafeAddr())
	if xptr == yptr {
		return true // identical references
	}
	c := comparison{xptr, yptr, x.Type()}
	if seen[c] {
		return true // already seen
	}
	seen[c] = true
}
```

这是Equal函数用法的例子:

```Go
fmt.Println(Equal([]int{1, 2, 3}, []int{1, 2, 3}))        // "true"
fmt.Println(Equal([]string{"foo"}, []string{"bar"}))      // "false"
fmt.Println(Equal([]string(nil), []string{}))             // "true"
fmt.Println(Equal(map[string]int(nil), map[string]int{})) // "true"
```

Equal函数甚至可以处理类似12.3章中导致Display陷入死循环的带有环的数据。

```Go
// Circular linked lists a -> b -> a and c -> c.
type link struct {
	value string
	tail *link
}
a, b, c := &link{value: "a"}, &link{value: "b"}, &link{value: "c"}
a.tail, b.tail, c.tail = b, a, c
fmt.Println(Equal(a, a)) // "true"
fmt.Println(Equal(b, b)) // "true"
fmt.Println(Equal(c, c)) // "true"
fmt.Println(Equal(a, b)) // "false"
fmt.Println(Equal(a, c)) // "false"
```

**练习 13.1：** 定义一个深比较函数，对于十亿以内的数字比较，忽略类型差异。

```

```



**练习 13.2：** 编写一个函数，报告其参数是否为循环数据结构。

```

```



## 13.4. 通过cgo调用C代码

Go程序可能会遇到要访问C语言的某些硬件驱动函数的场景，或者是从一个C++语言实现的嵌入式数据库查询记录的场景，或者是使用Fortran语言实现的一些线性代数库的场景。C语言作为一个通用语言，很多库会选择提供一个C兼容的API，然后用其他不同的编程语言实现（译者：Go语言需要也应该拥抱这些巨大的代码遗产）。

在本节中，我们将构建一个简易的数据压缩程序，使用了一个Go语言自带的叫`cgo`的用于支援C语言函数调用的工具。这类工具一般被称为 *foreign-function interfaces* （简称`ffi`），并且在类似工具中`cgo`也不是唯一的。SWIG（<http://swig.org>）是另一个类似的且被广泛使用的工具，SWIG提供了很多复杂特性以支援C++的特性，但SWIG并不是我们要讨论的主题。

在标准库的`compress/...`子包有很多流行的压缩算法的编码和解码实现，包括流行的LZW压缩算法（Unix的`compress`命令用的算法）和DEFLATE压缩算法（GNU `gzip`命令用的算法）。这些包的API的细节虽然有些差异，但是它们都提供了针对 `io.Writer`类型输出的压缩接口和提供了针对`io.Reader`类型输入的解压缩接口。例如：

```Go
package gzip // compress/gzip
func NewWriter(w io.Writer) io.WriteCloser
func NewReader(r io.Reader) (io.ReadCloser, error)
```

`bzip2`压缩算法，是基于优雅的Burrows-Wheeler变换算法，运行速度比`gzip`要慢，但是可以提供更高的压缩比。标准库的`compress/bzip2`包目前还没有提供`bzip2`压缩算法的实现。完全从头开始实现一个压缩算法是一件繁琐的工作，而且 http://bzip.org 已经有现成的`libbzip2`的开源实现，不仅文档齐全而且性能又好。

如果是比较小的C语言库，我们完全可以用纯Go语言重新实现一遍。如果我们对性能也没有特殊要求的话，我们还可以用`os/exec`包的方法将C编写的应用程序作为一个子进程运行。只有当你需要使用复杂而且性能更高的底层C接口时，就是使用`cgo`的场景了（译注：用`os/exec`包调用子进程的方法会导致程序运行时依赖那个应用程序）。下面我们将通过一个例子讲述`cgo`的具体用法。

> 译注：本章采用的代码都是最新的。因为之前已经出版的书中包含的代码只能在Go1.5之前使用。从Go1.6开始，Go语言已经明确规定了哪些Go语言指针可以直接传入C语言函数。新代码重点是增加了`bz2alloc`和`bz2free`的两个函数，用于`bz_stream`对象空间的申请和释放操作。下面是新代码中增加的注释，说明这个问题：

```Go
// The version of this program that appeared in the first and second
// printings did not comply with the proposed rules for passing
// pointers between Go and C, described here:
// https://github.com/golang/proposal/blob/master/design/12416-cgo-pointers.md
//
// The rules forbid a C function like bz2compress from storing 'in'
// and 'out' (pointers to variables allocated by Go) into the Go
// variable 's', even temporarily.
//
// The version below, which appears in the third printing, has been
// corrected.  To comply with the rules, the bz_stream variable must
// be allocated by C code.  We have introduced two C functions,
// bz2alloc and bz2free, to allocate and free instances of the
// bz_stream type.  Also, we have changed bz2compress so that before
// it returns, it clears the fields of the bz_stream that contain
// pointers to Go variables.
```

要使用libbzip2，我们需要先构建一个`bz_stream`结构体，用于保持输入和输出缓存。然后有三个函数：`BZ2_bzCompressInit`用于初始化缓存，`BZ2_bzCompress`用于将输入缓存的数据压缩到输出缓存，`BZ2_bzCompressEnd`用于释放不需要的缓存。（目前不要担心包的具体结构，这个例子的目的就是演示各个部分如何组合在一起的。）

我们可以在Go代码中直接调用`BZ2_bzCompressInit`和`BZ2_bzCompressEnd`，但是对于`BZ2_bzCompress`，我们将定义一个C语言的包装函数，用它完成真正的工作。下面是C代码，对应一个独立的文件。

*gopl.io/ch13/bzip*

```C
/* This file is gopl.io/ch13/bzip/bzip2.c,         */
/* a simple wrapper for libbzip2 suitable for cgo. */
#include <bzlib.h>

int bz2compress(bz_stream *s, int action,
                char *in, unsigned *inlen, char *out, unsigned *outlen) {
	s->next_in = in;
	s->avail_in = *inlen;
	s->next_out = out;
	s->avail_out = *outlen;
	int r = BZ2_bzCompress(s, action);
	*inlen -= s->avail_in;
	*outlen -= s->avail_out;
	s->next_in = s->next_out = NULL;
	return r;
}
```

现在让我们转到Go语言部分，第一部分如下所示。其中`import "C"`的语句是比较特别的。其实并没有一个叫C的包，但是这行语句会让Go编译程序在编译之前先运行`cgo`工具。

```Go
// Package bzip provides a writer that uses bzip2 compression (bzip.org).
package bzip

/*
#cgo CFLAGS: -I/usr/include
#cgo LDFLAGS: -L/usr/lib -lbz2
#include <bzlib.h>
#include <stdlib.h>
bz_stream* bz2alloc() { return calloc(1, sizeof(bz_stream)); }
int bz2compress(bz_stream *s, int action,
                char *in, unsigned *inlen, char *out, unsigned *outlen);
void bz2free(bz_stream* s) { free(s); }
*/
import "C"

import (
	"io"
	"unsafe"
)

type writer struct {
	w      io.Writer // underlying output stream
	stream *C.bz_stream
	outbuf [64 * 1024]byte
}

// NewWriter returns a writer for bzip2-compressed streams.
func NewWriter(out io.Writer) io.WriteCloser {
	const blockSize = 9
	const verbosity = 0
	const workFactor = 30
	w := &writer{w: out, stream: C.bz2alloc()}
	C.BZ2_bzCompressInit(w.stream, blockSize, verbosity, workFactor)
	return w
}
```

在预处理过程中，`cgo`工具生成一个临时包用于包含所有在Go语言中访问的C语言的函数或类型。例如`C.bz_stream`和`C.BZ2_bzCompressInit`。`cgo`工具通过以某种特殊的方式调用本地的C编译器来发现在Go源文件导入声明前的注释中包含的C头文件中的内容（译注：`import "C"`语句前紧挨着的注释是对应`cgo`的特殊语法，对应必要的构建参数选项和C语言代码）。

在`cgo`注释中还可以包含`#cgo`指令，用于给C语言工具链指定特殊的参数。例如CFLAGS和LDFLAGS分别对应传给C语言编译器的编译参数和链接器参数，使它们可以从特定目录找到`bzlib.h`头文件和`libbz2.a`库文件。这个例子假设你已经在`/usr`目录成功安装了`bzip2`库。如果`bzip2`库是安装在不同的位置，你需要更新这些参数（译注：这里有一个从纯C代码生成的`cgo`绑定，不依赖`bzip2`静态库和操作系统的具体环境，具体请访问 https://github.com/chai2010/bzip2 ）。

`NewWriter`函数通过调用C语言的`BZ2_bzCompressInit`函数来初始化`stream`中的缓存。在`writer`结构中还包括了另一个`buffer`，用于输出缓存。

下面是`Write`方法的实现，返回成功压缩数据的大小，主体是一个循环中调用C语言的`bz2compress`函数实现的。从代码可以看到，Go程序可以访问C语言的`bz_stream`、`char`和`uint`类型，还可以访问`bz2compress`等函数，甚至可以访问C语言中像`BZ_RUN`那样的宏定义，全部都是以`C.x`语法访问。其中`C.uint`类型和Go语言的`uint`类型并不相同，即使它们具有相同的大小也是不同的类型。

```Go
func (w *writer) Write(data []byte) (int, error) {
	if w.stream == nil {
		panic("closed")
	}
	var total int // uncompressed bytes written

	for len(data) > 0 {
		inlen, outlen := C.uint(len(data)), C.uint(cap(w.outbuf))
		C.bz2compress(w.stream, C.BZ_RUN,
			(*C.char)(unsafe.Pointer(&data[0])), &inlen,
			(*C.char)(unsafe.Pointer(&w.outbuf)), &outlen)
		total += int(inlen)
		data = data[inlen:]
		if _, err := w.w.Write(w.outbuf[:outlen]); err != nil {
			return total, err
		}
	}
	return total, nil
}
```

在循环的每次迭代中，向`bz2compress`传入数据的地址和剩余部分的长度，还有输出缓存`w.outbuf`的地址和容量。这两个长度信息通过它们的地址传入而不是值传入，因为`bz2compress`函数可能会根据已经压缩的数据和压缩后数据的大小来更新这两个值。每个块压缩后的数据被写入到底层的`io.Writer`。

`Close`方法和`Write`方法有着类似的结构，通过一个循环将剩余的压缩数据刷新到输出缓存。

```Go
// Close flushes the compressed data and closes the stream.
// It does not close the underlying io.Writer.
func (w *writer) Close() error {
	if w.stream == nil {
		panic("closed")
	}
	defer func() {
		C.BZ2_bzCompressEnd(w.stream)
		C.bz2free(w.stream)
		w.stream = nil
	}()
	for {
		inlen, outlen := C.uint(0), C.uint(cap(w.outbuf))
		r := C.bz2compress(w.stream, C.BZ_FINISH, nil, &inlen,
			(*C.char)(unsafe.Pointer(&w.outbuf)), &outlen)
		if _, err := w.w.Write(w.outbuf[:outlen]); err != nil {
			return err
		}
		if r == C.BZ_STREAM_END {
			return nil
		}
	}
}
```

压缩完成后，`Close`方法用了`defer`函数确保函数退出前调用`C.BZ2_bzCompressEnd`和`C.bz2free`释放相关的C语言运行时资源。此刻`w.stream`指针将不再有效，我们将它设置为nil以保证安全，然后在每个方法中增加了nil检测，以防止用户在关闭后依然错误使用相关方法。

上面的实现中，不仅仅写是非并发安全的，甚至并发调用`Close`和`Write`方法也可能导致程序的的崩溃。修复这个问题是练习13.3的内容。

下面的`bzipper`程序，使用我们自己包实现的`bzip2`压缩命令。它的行为和许多Unix系统的`bzip2`命令类似。

*gopl.io/ch13/bzipper*

```Go
// Bzipper reads input, bzip2-compresses it, and writes it out.
package main

import (
	"io"
	"log"
	"os"
	"gopl.io/ch13/bzip"
)

func main() {
	w := bzip.NewWriter(os.Stdout)
	if _, err := io.Copy(w, os.Stdin); err != nil {
		log.Fatalf("bzipper: %v\n", err)
	}
	if err := w.Close(); err != nil {
		log.Fatalf("bzipper: close: %v\n", err)
	}
}
```

在上面的场景中，我们使用`bzipper`压缩了`/usr/share/dict/words`系统自带的词典，从938,848字节压缩到335,405字节。大约是原始数据大小的三分之一。然后使用系统自带的`bunzip2`命令进行解压。压缩前后文件的SHA256哈希码是相同了，这也说明了我们的压缩工具是正确的。（如果你的系统没有`sha256sum`命令，那么请先按照练习4.2实现一个类似的工具）

```go
$ go build gopl.io/ch13/bzipper
$ wc -c < /usr/share/dict/words
938848
$ sha256sum < /usr/share/dict/words
126a4ef38493313edc50b86f90dfdaf7c59ec6c948451eac228f2f3a8ab1a6ed -
$ ./bzipper < /usr/share/dict/words | wc -c
335405
$ ./bzipper < /usr/share/dict/words | bunzip2 | sha256sum
126a4ef38493313edc50b86f90dfdaf7c59ec6c948451eac228f2f3a8ab1a6ed -
```

我们演示了如何将一个C语言库链接到Go语言程序。相反，将Go编译为静态库然后链接到C程序，或者将Go程序编译为动态库然后在C程序中动态加载也都是可行的（译注：在Go1.5中，Windows系统的Go语言实现并不支持生成C语言动态库或静态库的特性。不过好消息是，目前已经有人在尝试解决这个问题，具体请访问 [Issue11058](https://github.com/golang/go/issues/11058) ）。这里我们只展示的`cgo`很小的一些方面，更多的关于内存管理、指针、回调函数、中断信号处理、字符串、`errno`处理、终结器，以及`goroutines`和系统线程的关系等，有很多细节可以讨论。特别是如何将Go语言的指针传入C函数的规则也是异常复杂的（译注：简单来说，要传入C函数的Go指针指向的数据本身不能包含指针或其他引用类型；并且C函数在返回后不能继续持有Go指针；并且在C函数返回之前，Go指针是被锁定的，不能导致对应指针数据被移动或栈的调整），部分的原因在13.2节有讨论到，但是在Go1.5中还没有被明确（译注：Go1.6将会明确`cgo`中的指针使用规则）。如果要进一步阅读，可以从 https://golang.org/cmd/cgo 开始。

**练习 13.3：** 使用`sync.Mutex`以保证`bzip2.writer`在多个`goroutines`中被并发调用是安全的。

```

```



**练习 13.4：** 因为C库依赖的限制。 使用`os/exec`包启动`/bin/bzip2`命令作为一个子进程，提供一个纯Go的`bzip.NewWriter`的替代实现（译注：虽然是纯Go实现，但是运行时将依赖`/bin/bzip2`命令，其他操作系统可能无法运行）。

```

```



## 13.5. 几点忠告

我们在前一章结尾的时候，我们警告要谨慎使用`reflect`包。那些警告同样适用于本章的`unsafe`包。

高级语言使得程序员不用再关心真正运行程序的指令细节，同时也不再需要关注许多如内存布局之类的实现细节。因为高级语言这个绝缘的抽象层，我们可以编写安全健壮的，并且可以运行在不同操作系统上的具有高度可移植性的程序。

但是`unsafe`包，它让程序员可以透过这个绝缘的抽象层直接使用一些必要的功能，虽然可能是为了获得更好的性能。但是代价就是牺牲了可移植性和程序安全，因此使用`unsafe`包是一个危险的行为。我们对何时以及如何使用`unsafe`包的建议和我们在11.5节提到的Knuth对过早优化的建议类似。大多数Go程序员可能永远不会需要直接使用`unsafe`包。当然，也永远都会有一些需要使用`unsafe`包实现会更简单的场景。如果确实认为使用`unsafe`包是最理想的方式，那么应该尽可能将它限制在较小的范围，这样其它代码就可以忽略`unsafe`的影响。

现在，赶紧将最后两章抛入脑后吧。编写一些实实在在的应用是真理。请远离`reflect`的`unsafe`包，除非你确实需要它们。

最后，用Go快乐地编程。我们希望你能像我们一样喜欢Go语言。

