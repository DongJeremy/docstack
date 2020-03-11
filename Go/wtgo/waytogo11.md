# 第11章：接口（Interfaces）与反射（reflection）

本章介绍 Go 语言中接口和反射的相关内容。

﻿## 11.1 接口是什么

Go 语言不是一种 *“传统”* 的面向对象编程语言：它里面没有类和继承的概念。

但是 Go 语言里有非常灵活的 **接口** 概念，通过它可以实现很多面向对象的特性。接口提供了一种方式来 **说明** 对象的行为：如果谁能搞定这件事，它就可以用在这儿。

接口定义了一组方法（方法集），但是这些方法不包含（实现）代码：它们没有被实现（它们是抽象的）。接口里也不能包含变量。

通过如下格式定义接口：

```go
type Namer interface {
    Method1(param_list) return_type
    Method2(param_list) return_type
    ...
}
```

上面的 `Namer` 是一个 **接口类型**。

（按照约定，只包含一个方法的）接口的名字由方法名加 `[e]r` 后缀组成，例如 `Printer`、`Reader`、`Writer`、`Logger`、`Converter` 等等。还有一些不常用的方式（当后缀 `er` 不合适时），比如 `Recoverable`，此时接口名以 `able` 结尾，或者以 `I` 开头（像 `.NET` 或 `Java` 中那样）。

Go 语言中的接口都很简短，通常它们会包含 0 个、最多 3 个方法。

不像大多数面向对象编程语言，在 Go 语言中接口可以有值，一个接口类型的变量或一个 **接口值** ：`var ai Namer`，`ai` 是一个多字（`multiword`）数据结构，它的值是 `nil`。它本质上是一个指针，虽然不完全是一回事。指向接口值的指针是非法的，它们不仅一点用也没有，还会导致代码错误。

![](images/11.1_fig11.1.jpg?raw=true)

此处的方法指针表是通过运行时反射能力构建的。

类型（比如结构体）实现接口方法集中的方法，每一个方法的实现说明了此方法是如何作用于该类型的：**即实现接口**，同时方法集也构成了该类型的接口。实现了 `Namer` 接口类型的变量可以赋值给 `ai` （接收者值），此时方法表中的指针会指向被实现的接口方法。当然如果另一个类型（也实现了该接口）的变量被赋值给 `ai`，这二者（译者注：指针和方法实现）也会随之改变。

**类型不需要显式声明它实现了某个接口：接口被隐式地实现。多个类型可以实现同一个接口**。

**实现某个接口的类型（除了实现接口方法外）可以有其他的方法**。

**一个类型可以实现多个接口**。

**接口类型可以包含一个实例的引用， 该实例的类型实现了此接口（接口是动态类型）**。

即使接口在类型之后才定义，二者处于不同的包中，被单独编译：只要类型实现了接口中的方法，它就实现了此接口。

所有这些特性使得接口具有很大的灵活性。

第一个例子：

示例 11.1 [interfaces.go](examples/chapter_11/interfaces.go)：

```go
package main

import "fmt"

type Shaper interface {
	Area() float32
}

type Square struct {
	side float32
}

func (sq *Square) Area() float32 {
	return sq.side * sq.side
}

func main() {
	sq1 := new(Square)
	sq1.side = 5

	var areaIntf Shaper
	areaIntf = sq1
	// shorter,without separate declaration:
	// areaIntf := Shaper(sq1)
	// or even:
	// areaIntf := sq1
	fmt.Printf("The square has area: %f\n", areaIntf.Area())
}
```

输出：

    The square has area: 25.000000

上面的程序定义了一个结构体 `Square` 和一个接口 `Shaper`，接口有一个方法 `Area()`。

在 `main()` 方法中创建了一个 `Square` 的实例。在主程序外边定义了一个接收者类型是 `Square` 方法的 `Area()`，用来计算正方形的面积：结构体 `Square` 实现了接口 `Shaper` 。

所以可以将一个 `Square` 类型的变量赋值给一个接口类型的变量：`areaIntf = sq1` 。

现在接口变量包含一个指向 `Square` 变量的引用，通过它可以调用 `Square` 上的方法 `Area()`。当然也可以直接在 `Square` 的实例上调用此方法，但是在接口实例上调用此方法更令人兴奋，它使此方法更具有一般性。接口变量里包含了接收者实例的值和指向对应方法表的指针。

这是 **多态** 的 Go 版本，多态是面向对象编程中一个广为人知的概念：根据当前的类型选择正确的方法，或者说：同一种类型在不同的实例上似乎表现出不同的行为。

如果 `Square` 没有实现 `Area()` 方法，编译器将会给出清晰的错误信息：

```go
cannot use sq1 (type *Square) as type Shaper in assignment:
*Square does not implement Shaper (missing Area method)
```

如果 `Shaper` 有另外一个方法 `Perimeter()`，但是`Square` 没有实现它，即使没有人在 `Square` 实例上调用这个方法，编译器也会给出上面同样的错误。

扩展一下上面的例子，类型 `Rectangle` 也实现了 `Shaper` 接口。接着创建一个 `Shaper` 类型的数组，迭代它的每一个元素并在上面调用 `Area()` 方法，以此来展示多态行为：

示例 11.2 [interfaces_poly.go](examples/chapter_11/interfaces_poly.go)：

```go
package main

import "fmt"

type Shaper interface {
	Area() float32
}

type Square struct {
	side float32
}

func (sq *Square) Area() float32 {
	return sq.side * sq.side
}

type Rectangle struct {
	length, width float32
}

func (r Rectangle) Area() float32 {
	return r.length * r.width
}

func main() {

	r := Rectangle{5, 3} // Area() of Rectangle needs a value
	q := &Square{5}      // Area() of Square needs a pointer
	// shapes := []Shaper{Shaper(r), Shaper(q)}
	// or shorter
	shapes := []Shaper{r, q}
	fmt.Println("Looping through shapes for area ...")
	for n, _ := range shapes {
		fmt.Println("Shape details: ", shapes[n])
		fmt.Println("Area of this shape is: ", shapes[n].Area())
	}
}
```

输出：

```go
Looping through shapes for area ...
Shape details:  {5 3}
Area of this shape is:  15
Shape details:  &{5}
Area of this shape is:  25
```

在调用 `shapes[n].Area() ` 这个时，只知道 `shapes[n]` 是一个 `Shaper` 对象，最后它摇身一变成为了一个 `Square` 或 `Rectangle` 对象，并且表现出了相对应的行为。

也许从现在开始你将看到通过接口如何产生 **更干净**、**更简单** 及 **更具有扩展性** 的代码。在 11.12.3 中将看到在开发中为类型添加新的接口是多么的容易。

下面是一个更具体的例子：有两个类型 `stockPosition` 和 `car`，它们都有一个 `getValue()` 方法，我们可以定义一个具有此方法的接口 `valuable`。接着定义一个使用 `valuable` 类型作为参数的函数 `showValue()`，所有实现了 `valuable` 接口的类型都可以用这个函数。

示例 11.3 [valuable.go](examples/chapter_11/valuable.go)：

```go
package main

import "fmt"

type stockPosition struct {
	ticker     string
	sharePrice float32
	count      float32
}

/* method to determine the value of a stock position */
func (s stockPosition) getValue() float32 {
	return s.sharePrice * s.count
}

type car struct {
	make  string
	model string
	price float32
}

/* method to determine the value of a car */
func (c car) getValue() float32 {
	return c.price
}

/* contract that defines different things that have value */
type valuable interface {
	getValue() float32
}

func showValue(asset valuable) {
	fmt.Printf("Value of the asset is %f\n", asset.getValue())
}

func main() {
	var o valuable = stockPosition{"GOOG", 577.20, 4}
	showValue(o)
	o = car{"BMW", "M3", 66500}
	showValue(o)
}
```

输出：

```go
Value of the asset is 2308.800049
Value of the asset is 66500.000000
```

**一个标准库的例子**

`io` 包里有一个接口类型 `Reader`:

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

定义变量 `r`：` var r io.Reader`

那么就可以写如下的代码：

```go
	var r io.Reader
	r = os.Stdin    // see 12.1
	r = bufio.NewReader(r)
	r = new(bytes.Buffer)
	f,_ := os.Open("test.txt")
	r = bufio.NewReader(f)
```

上面 `r` 右边的类型都实现了 `Read()` 方法，并且有相同的方法签名，`r` 的静态类型是 `io.Reader`。

> **备注**
>
> 有的时候，也会以一种稍微不同的方式来使用接口这个词：从某个类型的角度来看，它的接口指的是：它的所有导出方法，只不过没有显式地为这些导出方法额外定一个接口而已。

**练习 11.1** simple_interface.go：

定义一个接口 `Simpler`，它有一个 `Get()` 方法和一个 `Set()`，`Get()`返回一个整型值，`Set()` 有一个整型参数。创建一个结构体类型 `Simple` 实现这个接口。

接着定一个函数，它有一个 `Simpler` 类型的参数，调用参数的 `Get()` 和 `Set()` 方法。在 `main` 函数里调用这个函数，看看它是否可以正确运行。

```go
// simple_interface.go
package main

import (
	"fmt"
)

type Simpler interface {
	Get() int
	Put(int)
}

type Simple struct {
	i int
}

func (p *Simple) Get() int {
	return p.i
}

func (p *Simple) Put(u int) {
	p.i = u
}

func fI(it Simpler) int {
	it.Put(5)
	return it.Get()
}

func main() {
	var s Simple
	fmt.Println(fI(&s)) // &s is required because Get() is defined with a receiver type pointer
}

// Output: 5
```

**练习 11.2** interfaces_poly2.go：

a) 扩展 `interfaces_poly.go` 中的例子，添加一个 `Circle` 类型

b) 使用一个抽象类型 `Shape`（没有字段） 实现同样的功能，它实现接口 `Shaper`，然后在其他类型里内嵌此类型。扩展 10.6.5 中的例子来说明覆写。

```go
// simple_interface2.go
package main

import (
	"fmt"
)

type Simpler interface {
	Get() int
	Set(int)
}

type Simple struct {
	i int
}

func (p *Simple) Get() int {
	return p.i
}

func (p *Simple) Set(u int) {
	p.i = u
}

type RSimple struct {
	i int
	j int
}

func (p *RSimple) Get() int {
	return p.j
}

func (p *RSimple) Set(u int) {
	p.j = u
}

func fI(it Simpler) int {
	switch it.(type) {
	case *Simple:
		it.Set(5)
		return it.Get()
	case *RSimple:
		it.Set(50)
		return it.Get()
	default:
		return 99
	}
	return 0
}

func main() {
	var s Simple
	fmt.Println(fI(&s)) // &s is required because Get() is defined with a receiver type pointer
	var r RSimple
	fmt.Println(fI(&r))
}

/* Output:
5
50
*/
```

﻿## 11.2 接口嵌套接口

一个接口可以包含一个或多个其他的接口，这相当于直接将这些内嵌接口的方法列举在外层接口中一样。

比如接口 `File` 包含了 `ReadWrite` 和 `Lock` 的所有方法，它还额外有一个 `Close()` 方法。

```go
type ReadWrite interface {
    Read(b Buffer) bool
    Write(b Buffer) bool
}

type Lock interface {
    Lock()
    Unlock()
}

type File interface {
    ReadWrite
    Lock
    Close()
}
```

## 11.3 类型断言：如何检测和转换接口变量的类型

一个接口类型的变量 `varI` 中可以包含任何类型的值，必须有一种方式来检测它的 **动态** 类型，即运行时在变量中存储的值的实际类型。在执行过程中动态类型可能会有所不同，但是它总是可以分配给接口变量本身的类型。通常我们可以使用 **类型断言** 来测试在某个时刻 `varI` 是否包含类型 `T` 的值：

```go
v := varI.(T)       // unchecked type assertion
```

**varI 必须是一个接口变量**，否则编译器会报错：`invalid type assertion: varI.(T) (non-interface type (type of varI) on left)` 。

类型断言可能是无效的，虽然编译器会尽力检查转换是否有效，但是它不可能预见所有的可能性。如果转换在程序运行时失败会导致错误发生。更安全的方式是使用以下形式来进行类型断言：

```go
if v, ok := varI.(T); ok {  // checked type assertion
    Process(v)
    return
}
// varI is not of type T
```

如果转换合法，`v` 是 `varI` 转换到类型 `T` 的值，`ok` 会是 `true`；否则 `v` 是类型 `T` 的零值，`ok` 是 `false`，也没有运行时错误发生。

**应该总是使用上面的方式来进行类型断言**。

多数情况下，我们可能只是想在 `if` 中测试一下 `ok` 的值，此时使用以下的方法会是最方便的：

```go
if _, ok := varI.(T); ok {
    // ...
}
```

示例 11.4 [type_interfaces.go](examples/chapter_11/type_interfaces.go)：

```go
package main

import (
	"fmt"
	"math"
)

type Square struct {
	side float32
}

type Circle struct {
	radius float32
}

type Shaper interface {
	Area() float32
}

func main() {
	var areaIntf Shaper
	sq1 := new(Square)
	sq1.side = 5

	areaIntf = sq1
	// Is Square the type of areaIntf?
	if t, ok := areaIntf.(*Square); ok {
		fmt.Printf("The type of areaIntf is: %T\n", t)
	}
	if u, ok := areaIntf.(*Circle); ok {
		fmt.Printf("The type of areaIntf is: %T\n", u)
	} else {
		fmt.Println("areaIntf does not contain a variable of type Circle")
	}
}

func (sq *Square) Area() float32 {
	return sq.side * sq.side
}

func (ci *Circle) Area() float32 {
	return ci.radius * ci.radius * math.Pi
}
```

输出：

    The type of areaIntf is: *main.Square
    areaIntf does not contain a variable of type Circle

程序中定义了一个新类型 `Circle`，它也实现了 `Shaper` 接口。 `if t, ok := areaIntf.(*Square); ok ` 测试 `areaIntf` 里是否有一个包含 `*Square` 类型的变量，结果是确定的；然后我们测试它是否包含一个 `*Circle` 类型的变量，结果是否定的。

> **备注**
>
> 如果忽略 `areaIntf.(*Square)` 中的 `*` 号，会导致编译错误：`impossible type assertion: Square does not implement Shaper (Area method has pointer receiver)`。

﻿## 11.4 类型判断：type-switch

接口变量的类型也可以使用一种特殊形式的 `switch` 来检测：**type-switch** （下面是示例 11.4 的第二部分）：

```go
switch t := areaIntf.(type) {
case *Square:
	fmt.Printf("Type Square %T with value %v\n", t, t)
case *Circle:
	fmt.Printf("Type Circle %T with value %v\n", t, t)
case nil:
	fmt.Printf("nil value: nothing to check?\n")
default:
	fmt.Printf("Unexpected type %T\n", t)
}
```

输出：

```go
Type Square *main.Square with value &{5}
```

变量 `t` 得到了 `areaIntf` 的值和类型， 所有 `case` 语句中列举的类型（`nil` 除外）都必须实现对应的接口（在上例中即 `Shaper`），如果被检测类型没有在 `case` 语句列举的类型中，就会执行 `default` 语句。

可以用 `type-switch` 进行运行时类型分析，但是在 `type-switch` 不允许有 `fallthrough` 。

如果仅仅是测试变量的类型，不用它的值，那么就可以不需要赋值语句，比如：

```go
switch areaIntf.(type) {
case *Square:
	// TODO
case *Circle:
	// TODO
...
default:
	// TODO
}
```

下面的代码片段展示了一个类型分类函数，它有一个可变长度参数，可以是任意类型的数组，它会根据数组元素的实际类型执行不同的动作：

```go
func classifier(items ...interface{}) {
	for i, x := range items {
		switch x.(type) {
		case bool:
			fmt.Printf("Param #%d is a bool\n", i)
		case float64:
			fmt.Printf("Param #%d is a float64\n", i)
		case int, int64:
			fmt.Printf("Param #%d is a int\n", i)
		case nil:
			fmt.Printf("Param #%d is a nil\n", i)
		case string:
			fmt.Printf("Param #%d is a string\n", i)
		default:
			fmt.Printf("Param #%d is unknown\n", i)
		}
	}
}
```

可以这样调用此方法：`classifier(13, -14.3, "BELGIUM", complex(1, 2), nil, false)` 。

在处理来自于外部的、类型未知的数据时，比如解析诸如 `JSON` 或 `XML` 编码的数据，类型测试和转换会非常有用。

在示例 12.17（`xml.go`）中解析 XML 文档时，我们就会用到 `type-switch`。

**练习 11.4** `simple_interface2.go`：

接着练习 11.1 中的内容，创建第二个类型 `RSimple`，它也实现了接口 `Simpler`，写一个函数 `fi`，使它可以区分 `Simple` 和 `RSimple` 类型的变量。

```go
// simple_interface2.go
package main

import (
	"fmt"
)

type Simpler interface {
	Get() int
	Set(int)
}

type Simple struct {
	i int
}

func (p *Simple) Get() int {
	return p.i
}

func (p *Simple) Set(u int) {
	p.i = u
}

type RSimple struct {
	i int
	j int
}

func (p *RSimple) Get() int {
	return p.j
}

func (p *RSimple) Set(u int) {
	p.j = u
}

func fI(it Simpler) int {
	switch it.(type) {
	case *Simple:
		it.Set(5)
		return it.Get()
	case *RSimple:
		it.Set(50)
		return it.Get()
	default:
		return 99
	}
}

func main() {
	var s Simple
	fmt.Println(fI(&s)) // &s is required because Get() is defined with a receiver type pointer
	var r RSimple
	fmt.Println(fI(&r))
}

/* Output:
5
50
*/
```

﻿## 11.5 测试一个值是否实现了某个接口

这是 11.3 类型断言中的一个特例：假定 `v` 是一个值，然后我们想测试它是否实现了 `Stringer` 接口，可以这样做：

```go
type Stringer interface {
    String() string
}

if sv, ok := v.(Stringer); ok {
    fmt.Printf("v implements String(): %s\n", sv.String()) // note: sv, not v
}
```

`Print` 函数就是如此检测类型是否可以打印自身的。

接口是一种契约，实现类型必须满足它，它描述了类型的行为，规定类型可以做什么。接口彻底将类型能做什么，以及如何做分离开来，使得相同接口的变量在不同的时刻表现出不同的行为，这就是多态的本质。

编写参数是接口变量的函数，这使得它们更具有一般性。

**使用接口使代码更具有普适性。**

标准库里到处都使用了这个原则，如果对接口概念没有良好的把握，是不可能理解它是如何构建的。

在接下来的章节中，我们会讨论两个重要的例子，试着去深入理解它们，这样你就可以更好的应用上面的原则。

## 11.6 使用方法集与接口

在第 10.6.3 节及例子 `methodset1.go` 中我们看到，作用于变量上的方法实际上是不区分变量到底是指针还是值的。当碰到接口类型值时，这会变得有点复杂，原因是接口变量中存储的具体值是不可寻址的，幸运的是，如果使用不当编译器会给出错误。考虑下面的程序：

示例 11.5 [methodset2.go](examples/chapter_11/methodset2.go)：

```go
package main

import (
	"fmt"
)

type List []int

func (l List) Len() int {
	return len(l)
}

func (l *List) Append(val int) {
	*l = append(*l, val)
}

type Appender interface {
	Append(int)
}

func CountInto(a Appender, start, end int) {
	for i := start; i <= end; i++ {
		a.Append(i)
	}
}

type Lener interface {
	Len() int
}

func LongEnough(l Lener) bool {
	return l.Len()*10 > 42
}

func main() {
	// A bare value
	var lst List
	// compiler error:
	// cannot use lst (type List) as type Appender in argument to CountInto:
	//       List does not implement Appender (Append method has pointer receiver)
	// CountInto(lst, 1, 10)
	if LongEnough(lst) { // VALID:Identical receiver type
		fmt.Printf("- lst is long enough\n")
	}

	// A pointer value
	plst := new(List)
	CountInto(plst, 1, 10) //VALID:Identical receiver type
	if LongEnough(plst) {
		// VALID: a *List can be dereferenced for the receiver
		fmt.Printf("- plst is long enough\n")
	}
}
```

**讨论**

在 `lst` 上调用 `CountInto` 时会导致一个编译器错误，因为 `CountInto` 需要一个 `Appender`，而它的方法 `Append` 只定义在指针上。 在 `lst` 上调用 `LongEnough` 是可以的，因为 `Len` 定义在值上。

在 `plst` 上调用 `CountInto` 是可以的，因为 `CountInto` 需要一个 `Appender`，并且它的方法 `Append` 定义在指针上。 在 `plst` 上调用 `LongEnough` 也是可以的，因为指针会被自动解引用。

**总结**

在接口上调用方法时，必须有和方法定义时相同的接收者类型或者是可以从具体类型 `P` 直接可以辨识的：

- 指针方法可以通过指针调用
- 值方法可以通过值调用
- 接收者是值的方法可以通过指针调用，因为指针会首先被解引用
- 接收者是指针的方法不可以通过值调用，因为存储在接口中的值没有地址

将一个值赋值给一个接口时，编译器会确保所有可能的接口方法都可以在此值上被调用，因此不正确的赋值在编译期就会失败。

> **译注**
>
> Go 语言规范定义了接口方法集的调用规则：
>
> - 类型 `*T` 的可调用方法集包含接受者为 `*T` 或 `T` 的所有方法集
> - 类型 `T` 的可调用方法集包含接受者为 `T` 的所有方法
> - 类型 `T` 的可调用方法集不包含接受者为 `*T` 的方法

﻿## 11.7 第一个例子：使用 Sorter 接口排序

一个很好的例子是来自标准库的 `sort` 包，要对一组数字或字符串排序，只需要实现三个方法：反映元素个数的 `Len()`方法、比较第 `i` 和 `j` 个元素的 `Less(i, j)` 方法以及交换第 `i` 和 `j` 个元素的 `Swap(i, j)` 方法。

排序函数的算法只会使用到这三个方法（可以使用任何排序算法来实现，此处我们使用冒泡排序）：

```go
func Sort(data Sorter) {
    for pass := 1; pass < data.Len(); pass++ {
        for i := 0;i < data.Len() - pass; i++ {
            if data.Less(i+1, i) {
                data.Swap(i, i + 1)
            }
        }
    }
}
```

`Sort` 函数接收一个接口类型的参数：`Sorter` ，它声明了这些方法：

```go
type Sorter interface {
    Len() int
    Less(i, j int) bool
    Swap(i, j int)
}
```

参数中的 `int` 是待排序序列长度的类型，而不是说要排序的对象一定要是一组 `int`。`i` 和 `j` 表示元素的整型索引，长度也是整型的。

现在如果我们想对一个 `int` 数组进行排序，所有必须做的事情就是：为数组定一个类型并在它上面实现 `Sorter` 接口的方法：

```go
type IntArray []int
func (p IntArray) Len() int           { return len(p) }
func (p IntArray) Less(i, j int) bool { return p[i] < p[j] }
func (p IntArray) Swap(i, j int)      { p[i], p[j] = p[j], p[i] }
```

下面是调用排序函数的一个具体例子：

```go
data := []int{74, 59, 238, -784, 9845, 959, 905, 0, 0, 42, 7586, -5467984, 7586}
a := sort.IntArray(data) //conversion to type IntArray from package sort
sort.Sort(a)
```

完整的、可运行的代码可以在 `sort.go` 和 `sortmain.go` 里找到。

同样的原理，排序函数可以用于一个浮点型数组，一个字符串数组，或者一个表示每周各天的结构体 `dayArray`。

示例 11.6 [sort.go](examples/chapter_11/sort/sort.go)：

```go
package sort

type Sorter interface {
	Len() int
	Less(i, j int) bool
	Swap(i, j int)
}

func Sort(data Sorter) {
	for pass := 1; pass < data.Len(); pass++ {
		for i := 0; i < data.Len()-pass; i++ {
			if data.Less(i+1, i) {
				data.Swap(i, i+1)
			}
		}
	}
}

func IsSorted(data Sorter) bool {
	n := data.Len()
	for i := n - 1; i > 0; i-- {
		if data.Less(i, i-1) {
			return false
		}
	}
	return true
}

// Convenience types for common cases
type IntArray []int

func (p IntArray) Len() int           { return len(p) }
func (p IntArray) Less(i, j int) bool { return p[i] < p[j] }
func (p IntArray) Swap(i, j int)      { p[i], p[j] = p[j], p[i] }

type StringArray []string

func (p StringArray) Len() int           { return len(p) }
func (p StringArray) Less(i, j int) bool { return p[i] < p[j] }
func (p StringArray) Swap(i, j int)      { p[i], p[j] = p[j], p[i] }

// Convenience wrappers for common cases
func SortInts(a []int)       { Sort(IntArray(a)) }
func SortStrings(a []string) { Sort(StringArray(a)) }

func IntsAreSorted(a []int) bool       { return IsSorted(IntArray(a)) }
func StringsAreSorted(a []string) bool { return IsSorted(StringArray(a)) }
```

示例 11.7 [sortmain.go](examples/chapter_11/sortmain.go)：

```go
package main

import (
	"./sort"
	"fmt"
)

func ints() {
	data := []int{74, 59, 238, -784, 9845, 959, 905, 0, 0, 42, 7586, -5467984, 7586}
	a := sort.IntArray(data) //conversion to type IntArray
	sort.Sort(a)
	if !sort.IsSorted(a) {
		panic("fails")
	}
	fmt.Printf("The sorted array is: %v\n", a)
}

func strings() {
	data := []string{"monday", "friday", "tuesday", "wednesday", "sunday", "thursday", "", "saturday"}
	a := sort.StringArray(data)
	sort.Sort(a)
	if !sort.IsSorted(a) {
		panic("fail")
	}
	fmt.Printf("The sorted array is: %v\n", a)
}

type day struct {
	num       int
	shortName string
	longName  string
}

type dayArray struct {
	data []*day
}

func (p *dayArray) Len() int           { return len(p.data) }
func (p *dayArray) Less(i, j int) bool { return p.data[i].num < p.data[j].num }
func (p *dayArray) Swap(i, j int)      { p.data[i], p.data[j] = p.data[j], p.data[i] }

func days() {
	Sunday    := day{0, "SUN", "Sunday"}
	Monday    := day{1, "MON", "Monday"}
	Tuesday   := day{2, "TUE", "Tuesday"}
	Wednesday := day{3, "WED", "Wednesday"}
	Thursday  := day{4, "THU", "Thursday"}
	Friday    := day{5, "FRI", "Friday"}
	Saturday  := day{6, "SAT", "Saturday"}
	data := []*day{&Tuesday, &Thursday, &Wednesday, &Sunday, &Monday, &Friday, &Saturday}
	a := dayArray{data}
	sort.Sort(&a)
	if !sort.IsSorted(&a) {
		panic("fail")
	}
	for _, d := range data {
		fmt.Printf("%s ", d.longName)
	}
	fmt.Printf("\n")
}

func main() {
	ints()
	strings()
	days()
}
```

输出：

    The sorted array is: [-5467984 -784 0 0 42 59 74 238 905 959 7586 7586 9845]
    The sorted array is: [ friday monday saturday sunday thursday tuesday wednesday]
    Sunday Monday Tuesday Wednesday Thursday Friday Saturday 

**备注**：

`panic("fail")` 用于停止处于在非正常情况下的程序（详细请参考 第13章），当然也可以先打印一条信息，然后调用 `os.Exit(1)` 来停止程序。

上面的例子帮助我们进一步了解了接口的意义和使用方式。对于基本类型的排序，标准库已经提供了相关的排序函数，所以不需要我们再重复造轮子了。对于一般性的排序，`sort` 包定义了一个接口：

```go
type Interface interface {
	Len() int
	Less(i, j int) bool
	Swap(i, j int)
}
```

这个接口总结了需要用于排序的抽象方法，函数 `Sort(data Interface)` 用来对此类对象进行排序，可以用它们来实现对其他类型的数据（非基本类型）进行排序。在上面的例子中，我们也是这么做的，不仅可以对 `int` 和 `string` 序列进行排序，也可以对用户自定义类型 `dayArray` 进行排序。

**练习 11.5** `interfaces_ext.go`：

a). 继续扩展程序，定义类型 `Triangle`，让它实现 `AreaInterface` 接口。通过计算一个特定三角形的面积来进行测试（`三角形面积=0.5 * (底 * 高)`）

b). 定义一个新接口 `PeriInterface`，它有一个 `Perimeter` 方法。让 `Square` 实现这个接口，并通过一个 `Square` 示例来测试它。

```go
package main

import "fmt"

type Square struct {
	side float32
}

type Triangle struct {
	base   float32
	height float32
}

type AreaInterface interface {
	Area() float32
}

type PeriInterface interface {
	Perimeter() float32
}

func main() {
	var areaIntf AreaInterface
	var periIntf PeriInterface

	sq1 := new(Square)
	sq1.side = 5
	tr1 := new(Triangle)
	tr1.base = 3
	tr1.height = 5

	areaIntf = sq1
	fmt.Printf("The square has area: %f\n", areaIntf.Area())

	periIntf = sq1
	fmt.Printf("The square has perimeter: %f\n", periIntf.Perimeter())

	areaIntf = tr1
	fmt.Printf("The triangle has area: %f\n", areaIntf.Area())
}

func (sq *Square) Area() float32 {
	return sq.side * sq.side
}

func (sq *Square) Perimeter() float32 {
	return 4 * sq.side
}

func (tr *Triangle) Area() float32 {
	return 0.5 * tr.base * tr.height
}
```

**练习 11.6** `point_interfaces.go`：

继续 10.3 中的练习 point_methods.go，定义接口 `Magnitude`，它有一个方法 `Abs()`。让 `Point`、`Point3` 及`Polar` 实现此接口。通过接口类型变量使用方法做`point.go`中同样的事情。

```go
// float64 is necessary as input to math.Sqrt()
package main

import (
	"fmt"
	"math"
)

type Magnitude interface {
	Abs() float64
}

var m Magnitude

type Point struct {
	X, Y float64
}

func (p *Point) Scale(s float64) {
	p.X *= s
	p.Y *= s
}

func (p *Point) Abs() float64 {
	return math.Sqrt(float64(p.X*p.X + p.Y*p.Y))
}

type Point3 struct {
	X, Y, Z float64
}

func (p *Point3) Abs() float64 {
	return math.Sqrt(float64(p.X*p.X + p.Y*p.Y + p.Z*p.Z))
}

type Polar struct {
	R, Theta float64
}

func (p Polar) Abs() float64 { return p.R }

func main() {
	p1 := new(Point)
	p1.X = 3
	p1.Y = 4
	m = p1 // p1 is type *Point, has method Abs()
	fmt.Printf("The length of the vector p1 is: %f\n", m.Abs())

	p2 := &Point{4, 5}
	m = p2
	fmt.Printf("The length of the vector p2 is: %f\n", m.Abs())

	p1.Scale(5)
	m = p1
	fmt.Printf("The length of the vector p1 after scaling is: %f\n", m.Abs())
	fmt.Printf("Point p1 after scaling has the following coordinates: X %f - Y %f\n", p1.X, p1.Y)

	mag := m.Abs()
	m = &Point3{3, 4, 5}
	mag += m.Abs()
	m = Polar{2.0, math.Pi / 2}
	mag += m.Abs()
	fmt.Printf("The float64 mag is now: %f", mag)
}

/* Output:
The length of the vector p1 is: 5.000000
The length of the vector p2 is: 6.403124
The length of the vector p1 after scaling is: 25.000000
Point p1 after scaling has the following coordinates: X 15.000000 - Y 20.000000
The float64 mag is now: 34.071068

-- instead of:
func (p *Point) Abs() float64 {
	return math.Sqrt(float64(p.X*p.X + p.Y*p.Y))
}
m = p1
we can write:
func (p Point) Abs() float64 {
	return math.Sqrt(float64(p.X*p.X + p.Y*p.Y))
}
m = p1
and instead of:
func (p Polar) Abs() float64 { return p.R }
m = Polar{2.0, math.Pi / 2}
we can write:
func (p *Polar) Abs() float64 { return p.R }
m = &Polar{2.0, math.Pi / 2}
with the same output
*/
```

**练习 11.7** `float_sort.go` / `float_sortmain.go`：

类似11.7和示例11.3/4，定义一个包 `float64`，并在包里定义类型 `Float64Array`，然后让它实现 `Sorter` 接口用来对 `float64` 数组进行排序。

另外提供如下方法：

- `NewFloat64Array()`：创建一个包含25个元素的数组变量（参考10.2）
- `List()`：返回数组格式化后的字符串，并在 `String()` 方法中调用它，这样就不用显式地调用 `List()` 来打印数组（参考10.7）
- `Fill()`：创建一个包含10个随机浮点数的数组（参考4.5.2.6）

在主程序中新建一个此类型的变量，然后对它排序并进行测试。

```go
// float_sort.go
package float64

import (
	"fmt"
	"math/rand"
	"time"
)

type Sorter interface {
	Len() int
	Less(i, j int) bool
	Swap(i, j int)
}

func Sort(data Sorter) {
	for pass := 1; pass < data.Len(); pass++ {
		for i := 0; i < data.Len()-pass; i++ {
			if data.Less(i+1, i) {
				data.Swap(i, i+1)
			}
		}
	}
}

func IsSorted(data Sorter) bool {
	n := data.Len()
	for i := n - 1; i > 0; i-- {
		if data.Less(i, i-1) {
			return false
		}
	}
	return true
}

type Float64Array []float64

func (p Float64Array) Len() int           { return len(p) }
func (p Float64Array) Less(i, j int) bool { return p[i] < p[j] }
func (p Float64Array) Swap(i, j int)      { p[i], p[j] = p[j], p[i] }

func NewFloat64Array() Float64Array {
	return make([]float64, 25)
}

func (p Float64Array) Fill(n int) {
	rand.Seed(int64(time.Now().Nanosecond()))
	for i := 0; i < n; i++ {
		p[i] = 100 * (rand.Float64())
	}
}

func (p Float64Array) List() string {
	s := "{ "
	for i := 0; i < p.Len(); i++ {
		if p[i] == 0 {
			continue
		}
		s += fmt.Sprintf("%3.1f ", p[i])
	}
	s += " }"
	return s
}

func (p Float64Array) String() string {
	return p.List()
}
```

```go
// float_sortmain.go
package main

import (
	"./float64"
	"fmt"
)

func main() {
	f1 := float64.NewFloat64Array()
	f1.Fill(10)
	fmt.Printf("Before sorting %s\n", f1)
	float64.Sort(f1)
	fmt.Printf("After sorting %s\n", f1)
	if float64.IsSorted(f1) {
		fmt.Println("The float64 array is sorted!")
	} else {
		fmt.Println("The float64 array is NOT sorted!")
	}
}

/* Output:
Before sorting { 55.0 82.3 36.4 66.6 25.3 82.7 47.4 21.5 4.6 81.6  }
After sorting { 4.6 21.5 25.3 36.4 47.4 55.0 66.6 81.6 82.3 82.7  }
The float64 array is sorted!
*/
```

**练习 11.8** `sort.go`/`sort_persons.go`：

定义一个结构体 `Person`，它有两个字段：`firstName` 和 `lastName`，为 `[]Person` 定义类型 `Persons` 。让 `Persons` 实现 `Sorter` 接口并进行测试。

```go
package sort

type Sorter interface {
	Len() int
	Less(i, j int) bool
	Swap(i, j int)
}

/*
func Sort(Sorter Interface) {
	for i := 1; i < data.Len(); i++ {
		for j := i; j > 0 && data.Less(j, j-1); j-- {
			data.Swap(j, j-1)
		}
	}
}
*/

func Sort(data Sorter) {
	for pass := 1; pass < data.Len(); pass++ {
		for i := 0; i < data.Len()-pass; i++ {
			if data.Less(i+1, i) {
				data.Swap(i, i+1)
			}
		}
	}
}

func IsSorted(data Sorter) bool {
	n := data.Len()
	for i := n - 1; i > 0; i-- {
		if data.Less(i, i-1) {
			return false
		}
	}
	return true
}

// Convenience types for common cases
type IntArray []int

func (p IntArray) Len() int           { return len(p) }
func (p IntArray) Less(i, j int) bool { return p[i] < p[j] }
func (p IntArray) Swap(i, j int)      { p[i], p[j] = p[j], p[i] }

type StringArray []string

func (p StringArray) Len() int           { return len(p) }
func (p StringArray) Less(i, j int) bool { return p[i] < p[j] }
func (p StringArray) Swap(i, j int)      { p[i], p[j] = p[j], p[i] }

// Convenience wrappers for common cases
func SortInts(a []int)       { Sort(IntArray(a)) }
func SortStrings(a []string) { Sort(StringArray(a)) }

func IntsAreSorted(a []int) bool       { return IsSorted(IntArray(a)) }
func StringsAreSorted(a []string) bool { return IsSorted(StringArray(a)) }
```

```go
// sort_persons.go
package main

import (
	"./sort"
	"fmt"
)

type Person struct {
	firstName string
	lastName  string
}

type Persons []Person

func (p Persons) Len() int { return len(p) }

func (p Persons) Less(i, j int) bool {
	in := p[i].lastName + " " + p[i].firstName
	jn := p[j].lastName + " " + p[j].firstName
	return in < jn
}

func (p Persons) Swap(i, j int) {
	p[i], p[j] = p[j], p[i]
}

func main() {
	p1 := Person{"Xavier", "Papadopoulos"}
	p2 := Person{"Chris", "Naegels"}
	p3 := Person{"John", "Doe"}
	arrP := Persons{p1, p2, p3}
	fmt.Printf("Before sorting: %v\n", arrP)
	sort.Sort(arrP)
	fmt.Printf("After sorting: %v\n", arrP)
}
```

﻿## 11.8 第二个例子：读和写

读和写是软件中很普遍的行为，提起它们会立即想到读写文件、缓存（比如字节或字符串切片）、标准输入输出、标准错误以及网络连接、管道等等，或者读写我们的自定义类型。为了让代码尽可能通用，Go 采取了一致的方式来读写数据。

`io` 包提供了用于读和写的接口 `io.Reader` 和 `io.Writer`：

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}
```

只要类型实现了读写接口，提供 `Read()` 和 `Write` 方法，就可以从它读取数据，或向它写入数据。一个对象要是可读的，它必须实现 `io.Reader` 接口，这个接口只有一个签名是 `Read(p []byte) (n int, err error)` 的方法，它从调用它的对象上读取数据，并把读到的数据放入参数中的字节切片中，然后返回读取的字节数和一个 `error` 对象，如果没有错误发生返回 `nil`，如果已经到达输入的尾端，会返回 `io.EOF("EOF")`，如果读取的过程中发生了错误，就会返回具体的错误信息。类似地，一个对象要是可写的，它必须实现 `io.Writer` 接口，这个接口也只有一个签名是 `Write(p []byte) (n int, err error)` 的方法，它将指定字节切片中的数据写入调用它的对象里，然后返回实际写入的字节数和一个 `error` 对象（如果没有错误发生就是 `nil`）。

`io` 包里的 `Readers` 和 `Writers` 都是不带缓冲的，`bufio` 包里提供了对应的带缓冲的操作，在读写 `UTF-8` 编码的文本文件时它们尤其有用。在 第12章 我们会看到很多在实战中使用它们的例子。

在实际编程中尽可能的使用这些接口，会使程序变得更通用，可以在任何实现了这些接口的类型上使用读写方法。

例如一个 `JPEG` 图形解码器，通过一个 `Reader` 参数，它可以解码来自磁盘、网络连接或以 `gzip` 压缩的 `HTTP` 流中的 `JPEG` 图形数据，或者其他任何实现了 `Reader` 接口的对象。 

## 11.9 空接口

### 11.9.1 概念

**空接口或者最小接口** 不包含任何方法，它对实现不做任何要求：

```go
type Any interface {}
```

任何其他类型都实现了空接口（它不仅仅像 `Java/C#` 中 `Object` 引用类型），`any` 或 `Any` 是空接口一个很好的别名或缩写。

空接口类似 `Java/C#` 中所有类的基类： `Object` 类，二者的目标也很相近。

可以给一个空接口类型的变量 `var val interface {}` 赋任何类型的值。

示例 11.8 [empty_interface.go](examples/chapter_11/empty_interface.go)：

```go
package main
import "fmt"

var i = 5
var str = "ABC"

type Person struct {
	name string
	age  int
}

type Any interface{}

func main() {
	var val Any
	val = 5
	fmt.Printf("val has the value: %v\n", val)
	val = str
	fmt.Printf("val has the value: %v\n", val)
	pers1 := new(Person)
	pers1.name = "Rob Pike"
	pers1.age = 55
	val = pers1
	fmt.Printf("val has the value: %v\n", val)
	switch t := val.(type) {
	case int:
		fmt.Printf("Type int %T\n", t)
	case string:
		fmt.Printf("Type string %T\n", t)
	case bool:
		fmt.Printf("Type boolean %T\n", t)
	case *Person:
		fmt.Printf("Type pointer to Person %T\n", t)
	default:
		fmt.Printf("Unexpected type %T", t)
	}
}
```

输出：

    val has the value: 5
    val has the value: ABC
    val has the value: &{Rob Pike 55}
    Type pointer to Person *main.Person

在上面的例子中，接口变量 `val` 被依次赋予一个 `int`，`string` 和 `Person` 实例的值，然后使用 `type-switch` 来测试它的实际类型。每个 `interface {}` 变量在内存中占据两个字长：一个用来存储它包含的类型，另一个用来存储它包含的数据或者指向数据的指针。

示例 [emptyint_switch.go](examples/chapter_11/emptyint_switch.go) 说明了空接口在 `type-switch` 中联合 `lambda` 函数的用法：

```go
package main

import "fmt"

type specialString string

var whatIsThis specialString = "hello"

func TypeSwitch() {
	testFunc := func(any interface{}) {
		switch v := any.(type) {
		case bool:
			fmt.Printf("any %v is a bool type", v)
		case int:
			fmt.Printf("any %v is an int type", v)
		case float32:
			fmt.Printf("any %v is a float32 type", v)
		case string:
			fmt.Printf("any %v is a string type", v)
		case specialString:
			fmt.Printf("any %v is a special String!", v)
		default:
			fmt.Println("unknown type!")
		}
	}
	testFunc(whatIsThis)
}

func main() {
	TypeSwitch()
}
```

输出：

    any hello is a special String!

**练习 11.9** simple_interface3.go：

继续 练习11.2，在它中添加一个 `gI` 函数，它不再接受 `Simpler` 类型的参数，而是接受一个空接口参数。然后通过类型断言判断参数是否是 `Simpler` 类型。最后在 `main` 使用 `gI` 取代 `fI` 函数并调用它。确保你的代码足够安全。

```go
// simple_interface2.go
package main

import (
	"fmt"
)

type Simpler interface {
	Get() int
	Set(int)
}

type Simple struct {
	i int
}

func (p *Simple) Get() int {
	return p.i
}

func (p *Simple) Set(u int) {
	p.i = u
}

type RSimple struct {
	i int
	j int
}

func (p *RSimple) Get() int {
	return p.j
}

func (p *RSimple) Set(u int) {
	p.j = u
}

func fI(it Simpler) int {
	switch it.(type) {
	case *Simple:
		it.Set(5)
		return it.Get()
	case *RSimple:
		it.Set(50)
		return it.Get()
	default:
		return 99
	}
	return 0
}

func gI(any interface{}) int {
	// return any.(Simpler).Get()   // unsafe, runtime panic possible
	if v, ok := any.(Simpler); ok {
		return v.Get()
	}
	return 0 // default value
}

/* Output:
6
60
*/

func main() {
	var s Simple = Simple{6}
	fmt.Println(gI(&s)) // &s is required because Get() is defined with a receiver type pointer
	var r RSimple = RSimple{60, 60}
	fmt.Println(gI(&r))
}

/* Output:
6
60
*/
```

### 11.9.2 构建通用类型或包含不同类型变量的数组

在 7.6.6 中我们看到了能被搜索和排序的 `int` 数组、`float` 数组以及 `string` 数组，那么对于其他类型的数组呢，是不是我们必须得自己编程实现它们？

现在我们知道该怎么做了，就是通过使用空接口。让我们给空接口定一个别名类型 `Element`：`type Element interface{}`

然后定义一个容器类型的结构体 `Vector`，它包含一个 `Element` 类型元素的切片：

```go
type Vector struct {
	a []Element
}
```

`Vector` 里能放任何类型的变量，因为任何类型都实现了空接口，实际上 `Vector` 里放的每个元素可以是不同类型的变量。我们为它定义一个 `At()` 方法用于返回第 `i` 个元素：

```go
func (p *Vector) At(i int) Element {
	return p.a[i]
}
```

再定一个 `Set()` 方法用于设置第 `i` 个元素的值：

```go
func (p *Vector) Set(i int, e Element) {
	p.a[i] = e
}
```

`Vector` 中存储的所有元素都是 `Element` 类型，要得到它们的原始类型（unboxing：拆箱）需要用到类型断言。TODO：The compiler rejects assertions guaranteed to fail，类型断言总是在运行时才执行，因此它会产生运行时错误。

**练习 11.10** `min_interface.go` / `minmain.go`：

仿照11.7中开发的 `Sorter` 接口，创建一个 `Miner` 接口并实现一些必要的操作。函数 `Min` 接受一个 `Miner` 类型变量的集合，然后计算并返回集合中最小的元素。

```go
// min_interface.go
package min

type Miner interface {
	Len() int
	ElemIx(ix int) interface{}
	Less(i, j int) bool
	Swap(i, j int)
}

func Min(data Miner) interface{} {
	min := data.ElemIx(0)
	for i := 1; i < data.Len(); i++ {
		if data.Less(i, i-1) {
			min = data.ElemIx(i)
		} else {
			data.Swap(i, i-1)
		}
	}
	return min
}

type IntArray []int

func (p IntArray) Len() int                  { return len(p) }
func (p IntArray) ElemIx(ix int) interface{} { return p[ix] }
func (p IntArray) Less(i, j int) bool        { return p[i] < p[j] }
func (p IntArray) Swap(i, j int)             { p[i], p[j] = p[j], p[i] }

type StringArray []string

func (p StringArray) Len() int                  { return len(p) }
func (p StringArray) ElemIx(ix int) interface{} { return p[ix] }
func (p StringArray) Less(i, j int) bool        { return p[i] < p[j] }
func (p StringArray) Swap(i, j int)             { p[i], p[j] = p[j], p[i] }
```

```go
// minmain.go
package main

import (
	"./min"
	"fmt"
)

func ints() {
	data := []int{74, 59, 238, -784, 9845, 959, 905, 0, 0, 42, 7586, -5467984, 7586}
	a := min.IntArray(data) //conversion to type IntArray
	m := min.Min(a)
	fmt.Printf("The minimum of the array is: %v\n", m)
}

func strings() {
	data := []string{"ddd", "eee", "bbb", "ccc", "aaa"}
	a := min.StringArray(data)
	m := min.Min(a)
	fmt.Printf("The minimum of the array is: %v\n", m)
}

func main() {
	ints()
	strings()
}

/* Output:
The minimum of the array is: -5467984
The minimum of the array is: aaa
*/
```

### 11.9.3 复制数据切片至空接口切片

假设你有一个 `myType` 类型的数据切片，你想将切片中的数据复制到一个空接口切片中，类似：

```go
var dataSlice []myType = FuncReturnSlice()
var interfaceSlice []interface{} = dataSlice
```

可惜不能这么做，编译时会出错：`cannot use dataSlice (type []myType) as type []interface { } in assignment`。

原因是它们俩在内存中的布局是不一样的（参考 [Go wiki](https://github.com/golang/go/wiki/InterfaceSlice)）。

必须使用 `for-range` 语句来一个一个显式地复制：

```go
var dataSlice []myType = FuncReturnSlice()
var interfaceSlice []interface{} = make([]interface{}, len(dataSlice))
for i, d := range dataSlice {
    interfaceSlice[i] = d
}
```

### 11.9.4 通用类型的节点数据结构

在10.1中我们遇到了诸如列表和树这样的数据结构，在它们的定义中使用了一种叫节点的递归结构体类型，节点包含一个某种类型的数据字段。现在可以使用空接口作为数据字段的类型，这样我们就能写出通用的代码。下面是实现一个二叉树的部分代码：通用定义、用于创建空节点的 `NewNode` 方法，及设置数据的 `SetData` 方法。

示例 11.10 [node_structures.go](examples/chapter_11/node_structures.go)：

```go
package main

import "fmt"

type Node struct {
	le   *Node
	data interface{}
	ri   *Node
}

func NewNode(left, right *Node) *Node {
	return &Node{left, nil, right}
}

func (n *Node) SetData(data interface{}) {
	n.data = data
}

func main() {
	root := NewNode(nil, nil)
	root.SetData("root node")
	// make child (leaf) nodes:
	a := NewNode(nil, nil)
	a.SetData("left node")
	b := NewNode(nil, nil)
	b.SetData("right node")
	root.le = a
	root.ri = b
	fmt.Printf("%v\n", root) // Output: &{0x125275f0 root node 0x125275e0}
}
```

### 11.9.5 接口到接口

一个接口的值可以赋值给另一个接口变量，只要底层类型实现了必要的方法。这个转换是在运行时进行检查的，转换失败会导致一个运行时错误：这是 `Go` 语言动态的一面，可以拿它和 `Ruby` 和 `Python` 这些动态语言相比较。

假定：

```go
var ai AbsInterface // declares method Abs()
type SqrInterface interface {
    Sqr() float
}
var si SqrInterface
pp := new(Point) // say *Point implements Abs, Sqr
var empty interface{}
```

那么下面的语句和类型断言是合法的：

```go
empty = pp                // everything satisfies empty
ai = empty.(AbsInterface) // underlying value pp implements Abs()
// (runtime failure otherwise)
si = ai.(SqrInterface) // *Point has Sqr() even though AbsInterface doesn’t
empty = si             // *Point implements empty set
// Note: statically checkable so type assertion not necessary.
```

下面是函数调用的一个例子：

```go
type myPrintInterface interface {
	print()
}

func f3(x myInterface) {
	x.(myPrintInterface).print() // type assertion to myPrintInterface
}
```

`x` 转换为 `myPrintInterface` 类型是完全动态的：只要 `x` 的底层类型（动态类型）定义了 `print` 方法这个调用就可以正常运行。

## 11.10 反射包

### 11.10.1 方法和类型的反射

在 10.4 节我们看到可以通过反射来分析一个结构体。本节我们进一步探讨强大的反射功能。反射是用程序检查其所拥有的结构，尤其是类型的一种能力；这是元编程的一种形式。反射可以在运行时检查类型和变量，例如它的大小、方法和 `动态`的调用这些方法。这对于没有源代码的包尤其有用。这是一个强大的工具，除非真得有必要，否则应当避免使用或小心使用。

变量的最基本信息就是类型和值：反射包的 `Type` 用来表示一个 Go 类型，反射包的 `Value` 为 Go 值提供了反射接口。

两个简单的函数，`reflect.TypeOf` 和 `reflect.ValueOf`，返回被检查对象的类型和值。例如，`x` 被定义为：`var x float64 = 3.4`，那么 `reflect.TypeOf(x)` 返回 `float64`，`reflect.ValueOf(x)` 返回 `<float64 Value>`

实际上，反射是通过检查一个接口的值，变量首先被转换成空接口。这从下面两个函数签名能够很明显的看出来：

```go
func TypeOf(i interface{}) Type
func ValueOf(i interface{}) Value
```

接口的值包含一个 `type` 和 `value`。

反射可以从接口值反射到对象，也可以从对象反射回接口值。

`reflect.Type` 和 `reflect.Value` 都有许多方法用于检查和操作它们。一个重要的例子是 `Value` 有一个 `Type` 方法返回 `reflect.Value` 的 `Type`。另一个是 `Type` 和 `Value` 都有 `Kind` 方法返回一个常量来表示类型：`Uint`、`Float64`、`Slice` 等等。同样 `Value` 有叫做 `Int` 和 `Float` 的方法可以获取存储在内部的值（跟 `int64` 和 `float64` 一样）

```go
const (
	Invalid Kind = iota
	Bool
	Int
	Int8
	Int16
	Int32
	Int64
	Uint
	Uint8
	Uint16
	Uint32
	Uint64
	Uintptr
	Float32
	Float64
	Complex64
	Complex128
	Array
	Chan
	Func
	Interface
	Map
	Ptr
	Slice
	String
	Struct
	UnsafePointer
)
```

对于 `float64` 类型的变量 `x`，如果 `v:=reflect.ValueOf(x)`，那么 `v.Kind()` 返回 `reflect.Float64` ，所以下面的表达式是 `true`
`v.Kind() == reflect.Float64`

`Kind` 总是返回底层类型：

```go
type MyInt int
var m MyInt = 5
v := reflect.ValueOf(m)
```

方法 `v.Kind()` 返回 `reflect.Int`。

变量 v 的 `Interface()` 方法可以得到还原（接口）值，所以可以这样打印 v 的值：`fmt.Println(v.Interface())`


尝试运行下面的代码：

示例 11.11 [reflect1.go](examples/chapter_11/reflect1.go)：

```go
// blog: Laws of Reflection
package main

import (
	"fmt"
	"reflect"
)

func main() {
	var x float64 = 3.4
	fmt.Println("type:", reflect.TypeOf(x))
	v := reflect.ValueOf(x)
	fmt.Println("value:", v)
	fmt.Println("type:", v.Type())
	fmt.Println("kind:", v.Kind())
	fmt.Println("value:", v.Float())
	fmt.Println(v.Interface())
	fmt.Printf("value is %5.2e\n", v.Interface())
	y := v.Interface().(float64)
	fmt.Println(y)
}
```

输出：

```
type: float64
value: 3.4
type: float64
kind: float64
value: 3.4
3.4
value is 3.40e+00
3.4
```

`x` 是一个 `float64` 类型的值，`reflect.ValueOf(x).Float()` 返回这个 `float64` 类型的实际值；同样的适用于 `Int(), Bool(), Complex(), String()`

### 11.10.2 通过反射修改(设置)值

继续前面的例子（参阅 11.9 [reflect2.go](examples/chapter_11/reflect2.go)），假设我们要把 `x` 的值改为 3.1415。`Value` 有一些方法可以完成这个任务，但是必须小心使用：`v.SetFloat(3.1415)`。

这将产生一个错误：`reflect.Value.SetFloat using unaddressable value`。

为什么会这样呢？问题的原因是 v 不是可设置的（这里并不是说值不可寻址）。是否可设置是 Value 的一个属性，并且不是所有的反射值都有这个属性：可以使用 `CanSet()` 方法测试是否可设置。

在例子中我们看到 `v.CanSet()` 返回 `false`： `settability of v: false`

当 `v := reflect.ValueOf(x)` 函数通过传递一个 `x` 拷贝创建了 `v`，那么 `v` 的改变并不能更改原始的 `x`。要想 v 的更改能作用到 `x`，那就必须传递 `x` 的地址 `v = reflect.ValueOf(&x)`。

通过 `Type()` 我们看到 `v` 现在的类型是 `*float64` 并且仍然是不可设置的。

要想让其可设置我们需要使用 `Elem()` 函数，这间接的使用指针：`v = v.Elem()`

现在 `v.CanSet()` 返回 `true` 并且 `v.SetFloat(3.1415)` 设置成功了！


示例 11.12 [reflect2.go](examples/chapter_11/reflect2.go)：

```go
package main

import (
	"fmt"
	"reflect"
)

func main() {
	var x float64 = 3.4
	v := reflect.ValueOf(x)
	// setting a value:
	// v.SetFloat(3.1415) // Error: will panic: reflect.Value.SetFloat using unaddressable value
	fmt.Println("settability of v:", v.CanSet())
	v = reflect.ValueOf(&x) // Note: take the address of x.
	fmt.Println("type of v:", v.Type())
	fmt.Println("settability of v:", v.CanSet())
	v = v.Elem()
	fmt.Println("The Elem of v is: ", v)
	fmt.Println("settability of v:", v.CanSet())
	v.SetFloat(3.1415) // this works!
	fmt.Println(v.Interface())
	fmt.Println(v)
}
```

输出：

```
settability of v: false
type of v: *float64
settability of v: false
The Elem of v is:  <float64 Value>
settability of v: true
3.1415
<float64 Value>
```

反射中有些内容是需要用地址去改变它的状态的。

### 11.10.3 反射结构

有些时候需要反射一个结构类型。`NumField()` 方法返回结构内的字段数量；通过一个 for 循环用索引取得每个字段的值 `Field(i)`。

我们同样能够调用签名在结构上的方法，例如，使用索引 n 来调用：`Method(n).Call(nil)`。

示例 11.13 [reflect_struct.go](examples/chapter_11/reflect_struct.go)：

```go
package main

import (
	"fmt"
	"reflect"
)

type NotknownType struct {
	s1, s2, s3 string
}

func (n NotknownType) String() string {
	return n.s1 + " - " + n.s2 + " - " + n.s3
}

// variable to investigate:
var secret interface{} = NotknownType{"Ada", "Go", "Oberon"}

func main() {
	value := reflect.ValueOf(secret) // <main.NotknownType Value>
	typ := reflect.TypeOf(secret)    // main.NotknownType
	// alternative:
	//typ := value.Type()  // main.NotknownType
	fmt.Println(typ)
	knd := value.Kind() // struct
	fmt.Println(knd)

	// iterate through the fields of the struct:
	for i := 0; i < value.NumField(); i++ {
		fmt.Printf("Field %d: %v\n", i, value.Field(i))
		// error: panic: reflect.Value.SetString using value obtained using unexported field
		//value.Field(i).SetString("C#")
	}

	// call the first method, which is String():
	results := value.Method(0).Call(nil)
	fmt.Println(results) // [Ada - Go - Oberon]
}
```

输出：

```
main.NotknownType
struct
Field 0: Ada
Field 1: Go
Field 2: Oberon
[Ada - Go - Oberon]
```

但是如果尝试更改一个值，会得到一个错误：

```
panic: reflect.Value.SetString using value obtained using unexported field
```

这是因为结构中只有被导出字段（首字母大写）才是可设置的；来看下面的例子：

示例 11.14 [reflect_struct2.go](examples/chapter_11/reflect_struct2.go)：

```go
package main

import (
	"fmt"
	"reflect"
)

type T struct {
	A int
	B string
}

func main() {
	t := T{23, "skidoo"}
	s := reflect.ValueOf(&t).Elem()
	typeOfT := s.Type()
	for i := 0; i < s.NumField(); i++ {
		f := s.Field(i)
		fmt.Printf("%d: %s %s = %v\n", i,
			typeOfT.Field(i).Name, f.Type(), f.Interface())
	}
	s.Field(0).SetInt(77)
	s.Field(1).SetString("Sunset Strip")
	fmt.Println("t is now", t)
}
```

输出：

```
0: A int = 23
1: B string = skidoo
t is now {77 Sunset Strip}
```

附录 37 深入阐述了反射概念。

## 11.11 Printf 和反射

在 Go 语言的标准库中，前几节所述的反射的功能被大量地使用。举个例子，`fmt` 包中的 `Printf`（以及其他格式化输出函数）都会使用反射来分析它的 `...` 参数。

`Printf` 的函数声明为：

```go
func Printf(format string, args ... interface{}) (n int, err error)
```

`Printf` 中的 `...` 参数为空接口类型。`Printf` 使用反射包来解析这个参数列表。所以，`Printf` 能够知道它每个参数的类型。因此格式化字符串中只有`%d`而没有 `%u` 和 `%ld`，因为它知道这个参数是 `unsigned` 还是 `long`。这也是为什么 `Print` 和 `Println` 在没有格式字符串的情况下还能如此漂亮地输出。

为了让大家更加具体地了解 `Printf` 中的反射，我们实现了一个简单的通用输出函数。其中使用了 `type-switch` 来推导参数类型，并根据类型来输出每个参数的值（这里用了 10.7 节中练习 10.13 的部分代码）

示例 11.15 [print.go](examples/chapter_11/print.go)：

```go
package main

import (
	"os"
	"strconv"
)

type Stringer interface {
	String() string
}

type Celsius float64

func (c Celsius) String() string {
	return strconv.FormatFloat(float64(c),'f', 1, 64) + " °C"
}

type Day int

var dayName = []string{"Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday", "Sunday"}

func (day Day) String() string {
	return dayName[day]
}

func print(args ...interface{}) {
	for i, arg := range args {
		if i > 0 {os.Stdout.WriteString(" ")}
		switch a := arg.(type) { // type switch
			case Stringer:	os.Stdout.WriteString(a.String())
			case int:		os.Stdout.WriteString(strconv.Itoa(a))
			case string:	os.Stdout.WriteString(a)
			// more types
			default:		os.Stdout.WriteString("???")
		}
	}
}

func main() {
	print(Day(1), "was", Celsius(18.36))  // Tuesday was 18.4 °C
}
```

在 12.8 节中我们将阐释 `fmt.Fprintf()` 是怎么运用同样的反射原则的。

## 11.12 接口与动态类型

### 11.12.1 Go 的动态类型

在经典的面向对象语言（像 C++，Java 和 C#）中数据和方法被封装为 `类` 的概念：类包含它们两者，并且不能剥离。

Go 没有类：数据（结构体或更一般的类型）和方法是一种松耦合的正交关系。

Go 中的接口跟 Java/C# 类似：都是必须提供一个指定方法集的实现。但是更加灵活通用：任何提供了接口方法实现代码的类型都隐式地实现了该接口，而不用显式地声明。

和其它语言相比，Go 是唯一结合了接口值，静态类型检查（是否该类型实现了某个接口），运行时动态转换的语言，并且不需要显式地声明类型是否满足某个接口。该特性允许我们在不改变已有的代码的情况下定义和使用新接口。

接收一个（或多个）接口类型作为参数的函数，其**实参**可以是任何实现了该接口的类型的变量。 `实现了某个接口的类型可以被传给任何以此接口为参数的函数` 。

类似于 Python 和 Ruby 这类动态语言中的 `动态类型（duck typing）`；这意味着对象可以根据提供的方法被处理（例如，作为参数传递给函数），而忽略它们的实际类型：它们能做什么比它们是什么更重要。

这在程序 `duck_dance.go` 中得以阐明，函数 `DuckDance` 接受一个 `IDuck` 接口类型变量。仅当 `DuckDance` 被实现了 `IDuck` 接口的类型调用时程序才能编译通过。

示例 11.16 [duck_dance.go](examples/chapter_11/duck_dance.go)：

```go
package main

import "fmt"

type IDuck interface {
	Quack()
	Walk()
}

func DuckDance(duck IDuck) {
	for i := 1; i <= 3; i++ {
		duck.Quack()
		duck.Walk()
	}
}

type Bird struct {
	// ...
}

func (b *Bird) Quack() {
	fmt.Println("I am quacking!")
}

func (b *Bird) Walk()  {
	fmt.Println("I am walking!")
}

func main() {
	b := new(Bird)
	DuckDance(b)
}
```

输出：

```
I am quacking!
I am walking!
I am quacking!
I am walking!
I am quacking!
I am walking!
```

如果 `Bird` 没有实现 `Walk()`（把它注释掉），会得到一个编译错误：

```
cannot use b (type *Bird) as type IDuck in function argument:
*Bird does not implement IDuck (missing Walk method)
```

如果对 `cat` 调用函数 `DuckDance()`，Go 会提示编译错误，但是 Python 和 Ruby 会以运行时错误结束。

### 11.12.2 动态方法调用

像 Python，Ruby 这类语言，动态类型是延迟绑定的（在运行时进行）：方法只是用参数和变量简单地调用，然后在运行时才解析（它们很可能有像 `responds_to` 这样的方法来检查对象是否可以响应某个方法，但是这也意味着更大的编码量和更多的测试工作）

Go 的实现与此相反，通常需要编译器静态检查的支持：当变量被赋值给一个接口类型的变量时，编译器会检查其是否实现了该接口的所有函数。如果方法调用作用于像 `interface{}` 这样的“泛型”上，你可以通过类型断言（参见 11.3 节）来检查变量是否实现了相应接口。

例如，你用不同的类型表示 XML 输出流中的不同实体。然后我们为 XML 定义一个如下的“写”接口（甚至可以把它定义为私有接口）：

```go
type xmlWriter interface {
	WriteXML(w io.Writer) error
}
```

现在我们可以实现适用于该流类型的任何变量的 `StreamXML` 函数，并用类型断言检查传入的变量是否实现了该接口；如果没有，我们就调用内建的 `encodeToXML` 来完成相应工作：

```go
// Exported XML streaming function.
func StreamXML(v interface{}, w io.Writer) error {
	if xw, ok := v.(xmlWriter); ok {
		// It’s an  xmlWriter, use method of asserted type.
		return xw.WriteXML(w)
	}
	// No implementation, so we have to use our own function (with perhaps reflection):
	return encodeToXML(v, w)
}

// Internal XML encoding function.
func encodeToXML(v interface{}, w io.Writer) error {
	// ...
}
```

Go 在这里用了和 `gob` 相同的机制：定义了两个接口 `GobEncoder` 和 `GobDecoder`。这样就允许类型自己实现从流编解码的具体方式；如果没有实现就使用标准的反射方式。

因此 Go 提供了动态语言的优点，却没有其他动态语言在运行时可能发生错误的缺点。

对于动态语言非常重要的单元测试来说，这样即可以减少单元测试的部分需求，又可以发挥相当大的作用。

Go 的接口提高了代码的分离度，改善了代码的复用性，使得代码开发过程中的设计模式更容易实现。用 Go 接口还能实现 `依赖注入模式`。

### 11.12.3 接口的提取

`提取接口` 是非常有用的设计模式，可以减少需要的类型和方法数量，而且不需要像传统的基于类的面向对象语言那样维护整个的类层次结构。

Go 接口可以让开发者找出自己写的程序中的类型。假设有一些拥有共同行为的对象，并且开发者想要抽象出这些行为，这时就可以创建一个接口来使用。
我们来扩展 11.1 节的示例 11.2 interfaces_poly.go，假设我们需要一个新的接口 `TopologicalGenus`，用来给 shape 排序（这里简单地实现为返回 int）。我们需要做的是给想要满足接口的类型实现 `Rank()` 方法：

示例 11.17 [multi_interfaces_poly.go](examples/chapter_11/multi_interfaces_poly.go)：

```go
//multi_interfaces_poly.go
package main

import "fmt"

type Shaper interface {
	Area() float32
}

type TopologicalGenus interface {
	Rank() int
}

type Square struct {
	side float32
}

func (sq *Square) Area() float32 {
	return sq.side * sq.side
}

func (sq *Square) Rank() int {
	return 1
}

type Rectangle struct {
	length, width float32
}

func (r Rectangle) Area() float32 {
	return r.length * r.width
}

func (r Rectangle) Rank() int {
	return 2
}

func main() {
	r := Rectangle{5, 3} // Area() of Rectangle needs a value
	q := &Square{5}      // Area() of Square needs a pointer
	shapes := []Shaper{r, q}
	fmt.Println("Looping through shapes for area ...")
	for n, _ := range shapes {
		fmt.Println("Shape details: ", shapes[n])
		fmt.Println("Area of this shape is: ", shapes[n].Area())
	}
	topgen := []TopologicalGenus{r, q}
	fmt.Println("Looping through topgen for rank ...")
	for n, _ := range topgen {
		fmt.Println("Shape details: ", topgen[n])
		fmt.Println("Topological Genus of this shape is: ", topgen[n].Rank())
	}
}
```

输出：

```
Looping through shapes for area ...
Shape details:  {5 3}
Area of this shape is:  15
Shape details:  &{5}
Area of this shape is:  25
Looping through topgen for rank ...
Shape details:  {5 3}
Topological Genus of this shape is:  2
Shape details:  &{5}
Topological Genus of this shape is:  1
```

所以你不用提前设计出所有的接口；`整个设计可以持续演进，而不用废弃之前的决定`。类型要实现某个接口，它本身不用改变，你只需要在这个类型上实现新的方法。

### 11.12.4 显式地指明类型实现了某个接口

如果你希望满足某个接口的类型显式地声明它们实现了这个接口，你可以向接口的方法集中添加一个具有描述性名字的方法。例如：

```go
type Fooer interface {
	Foo()
	ImplementsFooer()
}
```

类型 Bar 必须实现 `ImplementsFooer` 方法来满足 `Fooer` 接口，以清楚地记录这个事实。

```go
type Bar struct{}
func (b Bar) ImplementsFooer() {}
func (b Bar) Foo() {}
```

大部分代码并不使用这样的约束，因为它限制了接口的实用性。

但是有些时候，这样的约束在大量相似的接口中被用来解决歧义。

### 11.12.5 空接口和函数重载

在 6.1 节中, 我们看到函数重载是不被允许的。在 Go 语言中函数重载可以用可变参数 `...T` 作为函数最后一个参数来实现（参见 6.3 节）。如果我们把 T 换为空接口，那么可以知道任何类型的变量都是满足 T (空接口）类型的，这样就允许我们传递任何数量任何类型的参数给函数，即重载的实际含义。

函数 `fmt.Printf` 就是这样做的：

```go
fmt.Printf(format string, a ...interface{}) (n int, errno error)
```

这个函数通过枚举 `slice` 类型的实参动态确定所有参数的类型。并查看每个类型是否实现了 `String()` 方法，如果是就用于产生输出信息。我们可以回到 11.10 节查看这些细节。

### 11.12.6 接口的继承

当一个类型包含（内嵌）另一个类型（实现了一个或多个接口）的指针时，这个类型就可以使用（另一个类型）所有的接口方法。

例如：

```go
type Task struct {
	Command string
	*log.Logger
}
```

这个类型的工厂方法像这样：

```go
func NewTask(command string, logger *log.Logger) *Task {
	return &Task{command, logger}
}
```

当 `log.Logger` 实现了 `Log()` 方法后，Task 的实例 task 就可以调用该方法：

```go
task.Log()
```

类型可以通过继承多个接口来提供像 `多重继承` 一样的特性：

```go
type ReaderWriter struct {
	*io.Reader
	*io.Writer
}
```

上面概述的原理被应用于整个 Go 包，多态用得越多，代码就相对越少（参见 12.8 节）。这被认为是 Go 编程中的重要的最佳实践。

有用的接口可以在开发的过程中被归纳出来。添加新接口非常容易，因为已有的类型不用变动（仅仅需要实现新接口的方法）。已有的函数可以扩展为使用接口类型的约束性参数：通常只有函数签名需要改变。对比基于类的 OO 类型的语言在这种情况下则需要适应整个类层次结构的变化。

**练习 11.11**：[map_function_interface.go](exercises/chapter_11/map_function_interface.go)：

在练习 7.13 中我们定义了一个 `map` 函数来使用 `int` 切片 （`map_function.go`）。

通过空接口和类型断言，现在我们可以写一个可以应用于许多类型的 `泛型` 的 `map` 函数，为 `int` 和 `string` 构建一个把 `int` 值加倍和将字符串值与其自身连接（译者注：即`"abc"`变成`"abcabc"`）的 map 函数 `mapFunc`。

提示：为了可读性可以定义一个 `interface{}` 的别名，比如：`type obj interface{}`

```go
package main

import "fmt"

type obj interface{}

func main() {
	// define a generic lambda function mf:
	mf := func(i obj) obj {
		switch i.(type) {
		case int:
			return i.(int) * 2
		case string:
			return i.(string) + i.(string)
		}
		return i
	}

	isl := []obj{0, 1, 2, 3, 4, 5}
	res1 := mapFunc(mf, isl)
	for _, v := range res1 {
		fmt.Println(v)
	}
	println()

	ssl := []obj{"0", "1", "2", "3", "4", "5"}
	res2 := mapFunc(mf, ssl)
	for _, v := range res2 {
		fmt.Println(v)
	}
}

func mapFunc(mf func(obj) obj, list []obj) []obj {
	result := make([]obj, len(list))

	for ix, v := range list {
		result[ix] = mf(v)
	}

	// Equivalent:
	/*
		for ix := 0; ix<len(list); ix++ {
			result[ix] = mf(list[ix])
		}
	*/
	return result
}

/* Output:
0
2
4
6
8
10

00
11
22
33
44
55
*/
```

**练习 11.12**：[map_function_interface_var.go](exercises/chapter_11/map_function_interface_var.go)：

稍微改变练习 11.11，允许 `mapFunc` 接收不定数量的 `items`。

```go
package main

import "fmt"

type obj interface{}

func main() {
	// define a generic lambda function mf:
	mf := func(i obj) obj {
		switch i.(type) {
		case int:
			return i.(int) * 2
		case string:
			return i.(string) + i.(string)
		}
		return i
	}

	res1 := mapFunc(mf, 0, 1, 2, 3, 4, 5)
	for _, v := range res1 {
		fmt.Println(v)
	}
	println()
	res2 := mapFunc(mf, "0", "1", "2", "3", "4", "5")
	for _, v := range res2 {
		fmt.Println(v)
	}
}

func mapFunc(mf func(obj) obj, list ...obj) []obj {
	result := make([]obj, len(list))

	for ix, v := range list {
		result[ix] = mf(v)
	}

	// Equivalent:
	/*
		for ix := 0; ix<len(list); ix++ {
			result[ix] = mf(list[ix])
		}
	*/
	return result
}

/* Output:
0
2
4
6
8
10

00
11
22
33
44
55
*/

```

**练习 11.13**：[main_stack.go](exercises/chapter_11/main_stack.go)—[stack/stack_general.go](exercises/chapter_11/stack/stack_general.go)：

在练习 10.16 和 10.17 中我们开发了一些栈结构类型。但是它们被限制为某种固定的内建类型。现在用一个元素类型是 `interface{}`（空接口）的切片开发一个通用的栈类型。

实现下面的栈方法：

```go
Len() int
IsEmpty() bool
Push(x interface{})
Pop() (interface{}, error)
```

`Pop()` 改变栈并返回最顶部的元素；`Top()` 只返回最顶部元素。

在主程序中构建一个充满不同类型元素的栈，然后弹出并打印所有元素的值。

```go
// main_stack.go
package main

import (
	"./stack/stack"
	"fmt"
)

var st1 stack.Stack

func main() {
	st1.Push("Brown")
	st1.Push(3.14)
	st1.Push(100)
	st1.Push([]string{"Java", "C++", "Python", "C#", "Ruby"})
	for {
		item, err := st1.Pop()
		if err != nil {
			break
		}
		fmt.Println(item)
	}
}

/* Output:
[Java C++ Python C# Ruby]
100
3.14
Brown
*/
```

## 11.13 总结：Go 中的面向对象

我们总结一下前面看到的：Go 没有类，而是松耦合的类型、方法对接口的实现。

OO 语言最重要的三个方面分别是：封装，继承和多态，在 Go 中它们是怎样表现的呢？

- 封装（数据隐藏）：和别的 OO 语言有 4 个或更多的访问层次相比，Go 把它简化为了 2 层（参见 4.2 节的可见性规则）:

  1）包范围内的：通过标识符首字母小写，`对象` 只在它所在的包内可见

  2）可导出的：通过标识符首字母大写，`对象` 对所在包以外也可见

类型只拥有自己所在包中定义的方法。

- 继承：用组合实现：内嵌一个（或多个）包含想要的行为（字段和方法）的类型；多重继承可以通过内嵌多个类型实现
- 多态：用接口实现：某个类型的实例可以赋给它所实现的任意接口类型的变量。类型和接口是松耦合的，并且多重继承可以通过实现多个接口实现。Go 接口不是 Java 和 C# 接口的变体，而且接口间是不相关的，并且是大规模编程和可适应的演进型设计的关键。

## 11.14 结构体、集合和高阶函数

通常你在应用中定义了一个结构体，那么你也可能需要这个结构体的（指针）对象集合，比如：

```go
type Any interface{}
type Car struct {
	Model        string
	Manufacturer string
	BuildYear    int
	// ...
}

type Cars []*Car
```

然后我们就可以使用高阶函数，实际上也就是把函数作为定义所需方法（其他函数）的参数，例如：

1）定义一个通用的 `Process()` 函数，它接收一个作用于每一辆 car 的 f 函数作参数：

```go
// Process all cars with the given function f:
func (cs Cars) Process(f func(car *Car)) {
	for _, c := range cs {
		f(c)
	}
}
```

2）在上面的基础上，实现一个查找函数来获取子集合，并在 `Process()` 中传入一个闭包执行（这样就可以访问局部切片 `cars`）：

```go
// Find all cars matching a given criteria.
func (cs Cars) FindAll(f func(car *Car) bool) Cars {

	cars := make([]*Car, 0)
	cs.Process(func(c *Car) {
		if f(c) {
			cars = append(cars, c)
		}
	})
	return cars
}
```

3）实现 Map 功能，产出除 car 对象以外的东西：

```go
// Process cars and create new data.
func (cs Cars) Map(f func(car *Car) Any) []Any {
	result := make([]Any, 0)
	ix := 0
	cs.Process(func(c *Car) {
		result[ix] = f(c)
		ix++
	})
	return result
}
```

现在我们可以定义下面这样的具体查询：

```go
allNewBMWs := allCars.FindAll(func(car *Car) bool {
	return (car.Manufacturer == "BMW") && (car.BuildYear > 2010)
})
```

4）我们也可以根据参数返回不同的函数。也许我们想根据不同的厂商添加汽车到不同的集合，但是这（这种映射关系）可能会是会改变的。所以我们可以定义一个函数来产生特定的添加函数和 map 集：

```go
func MakeSortedAppender(manufacturers []string)(func(car *Car),map[string]Cars) {
	// Prepare maps of sorted cars.
	sortedCars := make(map[string]Cars)
	for _, m := range manufacturers {
		sortedCars[m] = make([]*Car, 0)
	}
	sortedCars["Default"] = make([]*Car, 0)
	// Prepare appender function:
	appender := func(c *Car) {
		if _, ok := sortedCars[c.Manufacturer]; ok {
			sortedCars[c.Manufacturer] = append(sortedCars[c.Manufacturer], c)
		} else {
			sortedCars["Default"] = append(sortedCars["Default"], c)
		}

	}
	return appender, sortedCars
}
```

现在我们可以用它把汽车分类为独立的集合，像这样：

```go
manufacturers := []string{"Ford", "Aston Martin", "Land Rover", "BMW", "Jaguar"}
sortedAppender, sortedCars := MakeSortedAppender(manufacturers)
allUnsortedCars.Process(sortedAppender)
BMWCount := len(sortedCars["BMW"])
```

我们让这些代码在下面的程序 cars.go 中执行：

示例 11.18 [cars.go](examples/chapter_11/cars.go)：

```go
// cars.go
package main

import (
	"fmt"
)

type Any interface{}
type Car struct {
	Model        string
	Manufacturer string
	BuildYear    int
	// ...
}
type Cars []*Car

func main() {
	// make some cars:
	ford := &Car{"Fiesta", "Ford", 2008}
	bmw := &Car{"XL 450", "BMW", 2011}
	merc := &Car{"D600", "Mercedes", 2009}
	bmw2 := &Car{"X 800", "BMW", 2008}
	// query:
	allCars := Cars([]*Car{ford, bmw, merc, bmw2})
	allNewBMWs := allCars.FindAll(func(car *Car) bool {
		return (car.Manufacturer == "BMW") && (car.BuildYear > 2010)
	})
	fmt.Println("AllCars: ", allCars)
	fmt.Println("New BMWs: ", allNewBMWs)
	//
	manufacturers := []string{"Ford", "Aston Martin", "Land Rover", "BMW", "Jaguar"}
	sortedAppender, sortedCars := MakeSortedAppender(manufacturers)
	allCars.Process(sortedAppender)
	fmt.Println("Map sortedCars: ", sortedCars)
	BMWCount := len(sortedCars["BMW"])
	fmt.Println("We have ", BMWCount, " BMWs")
}

// Process all cars with the given function f:
func (cs Cars) Process(f func(car *Car)) {
	for _, c := range cs {
		f(c)
	}
}

// Find all cars matching a given criteria.
func (cs Cars) FindAll(f func(car *Car) bool) Cars {
	cars := make([]*Car, 0)

	cs.Process(func(c *Car) {
		if f(c) {
			cars = append(cars, c)
		}
	})
	return cars
}

// Process cars and create new data.
func (cs Cars) Map(f func(car *Car) Any) []Any {
	result := make([]Any, len(cs))
	ix := 0
	cs.Process(func(c *Car) {
		result[ix] = f(c)
		ix++
	})
	return result
}

func MakeSortedAppender(manufacturers []string) (func(car *Car), map[string]Cars) {
	// Prepare maps of sorted cars.
	sortedCars := make(map[string]Cars)

	for _, m := range manufacturers {
		sortedCars[m] = make([]*Car, 0)
	}
	sortedCars["Default"] = make([]*Car, 0)

	// Prepare appender function:
	appender := func(c *Car) {
		if _, ok := sortedCars[c.Manufacturer]; ok {
			sortedCars[c.Manufacturer] = append(sortedCars[c.Manufacturer], c)
		} else {
			sortedCars["Default"] = append(sortedCars["Default"], c)
		}
	}
	return appender, sortedCars
}
```

输出：

```
AllCars:  [0xf8400038a0 0xf840003bd0 0xf840003ba0 0xf840003b70]
New BMWs:  [0xf840003bd0]
Map sortedCars:  map[Default:[0xf840003ba0] Jaguar:[] Land Rover:[] BMW:[0xf840003bd0 0xf840003b70] Aston Martin:[] Ford:[0xf8400038a0]]
We have  2  BMWs
```
