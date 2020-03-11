# 第10章：结构（struct）与方法（method）

Go 通过类型别名（alias types）和结构体的形式支持用户自定义类型，或者叫定制类型。一个带属性的结构体试图表示一个现实世界中的实体。结构体是复合类型（composite types），当需要定义一个类型，它由一系列属性组成，每个属性都有自己的类型和值的时候，就应该使用结构体，它把数据聚集在一起。然后可以访问这些数据，就好像它是一个独立实体的一部分。结构体也是值类型，因此可以通过 **new** 函数来创建。

组成结构体类型的那些数据称为 **字段（fields）**。每个字段都有一个类型和一个名字；在一个结构体中，字段名字必须是唯一的。

结构体的概念在软件工程上旧的术语叫 ADT（抽象数据类型：Abstract Data Type），在一些老的编程语言中叫 **记录（Record）**，比如 Cobol，在 C 家族的编程语言中它也存在，并且名字也是 **struct**，在面向对象的编程语言中，跟一个无方法的轻量级类一样。不过因为 Go 语言中没有类的概念，因此在 Go 中结构体有着更为重要的地位。

## 10.1 结构体定义

结构体定义的一般方式如下：

```go
type identifier struct {
    field1 type1
    field2 type2
    ...
}
```

`type T struct {a, b int}` 也是合法的语法，它更适用于简单的结构体。

结构体里的字段都有 **名字**，像 `field1`、`field2` 等，如果字段在代码中从来也不会被用到，那么可以命名它为 `_`。

结构体的字段可以是任何类型，甚至是结构体本身（参考第 [10.5](10.5.md) 节），也可以是函数或者接口（参考第 11 章）。可以声明结构体类型的一个变量，然后像下面这样给它的字段赋值：

```go
var s T
s.a = 5
s.b = 8
```

数组可以看作是一种结构体类型，不过它使用下标而不是具名的字段。

**使用 new**

使用 **new** 函数给一个新的结构体变量分配内存，它返回指向已分配内存的指针：`var t *T = new(T)`，如果需要可以把这条语句放在不同的行（比如定义是包范围的，但是分配却没有必要在开始就做）。

```go
var t *T
t = new(T)
```

写这条语句的惯用方法是：`t := new(T)`，变量 `t` 是一个指向 `T`的指针，此时结构体字段的值是它们所属类型的零值。

声明 `var t T` 也会给 `t` 分配内存，并零值化内存，但是这个时候 `t` 是类型T。在这两种方式中，`t` 通常被称做类型 `T` 的一个实例（`instance`）或对象（`object`）。

示例 10.1 [structs_fields.go](examples/chapter_10/structs_fields.go) 给出了一个非常简单的例子：

```go
package main
import "fmt"

type struct1 struct {
    i1  int
    f1  float32
    str string
}

func main() {
    ms := new(struct1)
    ms.i1 = 10
    ms.f1 = 15.5
    ms.str= "Chris"

    fmt.Printf("The int is: %d\n", ms.i1)
    fmt.Printf("The float is: %f\n", ms.f1)
    fmt.Printf("The string is: %s\n", ms.str)
    fmt.Println(ms)
}
```

输出：

```go
The int is: 10
The float is: 15.500000
The string is: Chris
&{10 15.5 Chris}
```

使用 `fmt.Println` 打印一个结构体的默认输出可以很好的显示它的内容，类似使用 **%v** 选项。

就像在面向对象语言所作的那样，可以使用点号符给字段赋值：`structname.fieldname = value`。

同样的，使用点号符可以获取结构体字段的值：`structname.fieldname`。

在 Go 语言中这叫 **选择器（selector）**。无论变量是一个结构体类型还是一个结构体类型指针，都使用同样的 **选择器符（selector-notation）** 来引用结构体的字段：

```go
type myStruct struct { i int }
var v myStruct    // v是结构体类型变量
var p *myStruct   // p是指向一个结构体类型变量的指针
v.i
p.i
```

初始化一个结构体实例（一个结构体字面量：`struct-literal`）的更简短和惯用的方式如下：

```go
    ms := &struct1{10, 15.5, "Chris"}
    // 此时ms的类型是 *struct1
```

或者：

```go
    var ms struct1
    ms = struct1{10, 15.5, "Chris"}
```

混合字面量语法（composite literal syntax）`&struct1{a, b, c}` 是一种简写，底层仍然会调用 `new ()`，这里值的顺序必须按照字段顺序来写。在下面的例子中能看到可以通过在值的前面放上字段名来初始化字段的方式。表达式 `new(Type)` 和 `&Type{}` 是等价的。

时间间隔（开始和结束时间以秒为单位）是使用结构体的一个典型例子：

```go
type Interval struct {
    start int
    end   int
}
```

初始化方式：

```go
intr := Interval{0, 3}            (A)
intr := Interval{end:5, start:1}  (B)
intr := Interval{end:5}           (C)
```

在（A）中，值必须以字段在结构体定义时的顺序给出，**&** 不是必须的。（B）显示了另一种方式，字段名加一个冒号放在值的前面，这种情况下值的顺序不必一致，并且某些字段还可以被忽略掉，就像（C）中那样。

结构体类型和字段的命名遵循可见性规则（第 [4.2](04.2.md) 节），一个导出的结构体类型中有些字段是导出的，另一些不是，这是可能的。

下图说明了结构体类型实例和一个指向它的指针的内存布局：

```go
type Point struct { x, y int }
```

使用 new 初始化：

![](images/10.1_fig10.1-1.jpg?raw=true)

作为结构体字面量初始化：

![](images/10.1_fig10.1-2.jpg?raw=true)

类型 `struct1` 在定义它的包 `pack1` 中必须是唯一的，它的完全类型名是：`pack1.struct1`。

下面的例子 [Listing 10.2—person.go](examples/chapter_10/person.go) 显示了一个结构体 `Person`，一个方法，方法有一个类型为 `*Person` 的参数（因此对象本身是可以被改变的），以及三种调用这个方法的不同方式：

```go
package main
import (
    "fmt"
    "strings"
)

type Person struct {
    firstName   string
    lastName    string
}

func upPerson(p *Person) {
    p.firstName = strings.ToUpper(p.firstName)
    p.lastName = strings.ToUpper(p.lastName)
}

func main() {
    // 1-struct as a value type:
    var pers1 Person
    pers1.firstName = "Chris"
    pers1.lastName = "Woodward"
    upPerson(&pers1)
    fmt.Printf("The name of the person is %s %s\n", pers1.firstName, pers1.lastName)

    // 2—struct as a pointer:
    pers2 := new(Person)
    pers2.firstName = "Chris"
    pers2.lastName = "Woodward"
    (*pers2).lastName = "Woodward"  // 这是合法的
    upPerson(pers2)
    fmt.Printf("The name of the person is %s %s\n", pers2.firstName, pers2.lastName)

    // 3—struct as a literal:
    pers3 := &Person{"Chris","Woodward"}
    upPerson(pers3)
    fmt.Printf("The name of the person is %s %s\n", pers3.firstName, pers3.lastName)
}
```

输出：

```go
The name of the person is CHRIS WOODWARD
The name of the person is CHRIS WOODWARD
The name of the person is CHRIS WOODWARD
```

在上面例子的第二种情况中，可以直接通过指针，像 `pers2.lastName="Woodward"` 这样给结构体字段赋值，没有像 C++ 中那样需要使用 `->` 操作符，Go 会自动做这样的转换。

注意也可以通过解指针的方式来设置值：`(*pers2).lastName = "Woodward"`

**结构体的内存布局**

Go 语言中，结构体和它所包含的数据在内存中是以连续块的形式存在的，即使结构体中嵌套有其他的结构体，这在性能上带来了很大的优势。不像 Java 中的引用类型，一个对象和它里面包含的对象可能会在不同的内存空间中，这点和 Go 语言中的指针很像。下面的例子清晰地说明了这些情况：

```go
type Rect1 struct {Min, Max Point }
type Rect2 struct {Min, Max *Point }
```

![](images/10.1_fig10.2.jpg?raw=true)

**递归结构体**

结构体类型可以通过引用自身来定义。这在定义链表或二叉树的元素（通常叫节点）时特别有用，此时节点包含指向临近节点的链接（地址）。如下所示，链表中的 `su`，树中的 `ri` 和 `le` 分别是指向别的节点的指针。

链表：

![](images/10.1_fig10.3.jpg?raw=true)

这块的 `data` 字段用于存放有效数据（比如 float64），`su` 指针指向后继节点。

Go 代码：

```go
type Node struct {
    data    float64
    su      *Node
}
```

链表中的第一个元素叫 `head`，它指向第二个元素；最后一个元素叫 `tail`，它没有后继元素，所以它的 `su` 为 `nil` 值。当然真实的链接会有很多数据节点，并且链表可以动态增长或收缩。

同样地可以定义一个双向链表，它有一个前趋节点 `pr` 和一个后继节点 `su`：

```go
type Node struct {
    pr      *Node
    data    float64
    su      *Node
}
```

二叉树：

![](images/10.1_fig10.4.jpg?raw=true)

二叉树中每个节点最多能链接至两个节点：左节点（`le`）和右节点（`ri`），这两个节点本身又可以有左右节点，依次类推。树的顶层节点叫根节点（**root**），底层没有子节点的节点叫叶子节点（**leaves**），叶子节点的 `le` 和 `ri` 指针为 `nil` 值。在 Go 中可以如下定义二叉树：

```go
type Tree struct {
    le      *Tree
    data    float64
    ri      *Tree
}
```

**结构体转换**

Go 中的类型转换遵循严格的规则。当为结构体定义了一个 `alias` 类型时，此结构体类型和它的 `alias` 类型都有相同的底层类型，它们可以如示例 10.3 那样互相转换，同时需要注意其中非法赋值或转换引起的编译错误。

示例 10.3：

```go
package main
import "fmt"

type number struct {
    f float32
}

type nr number   // alias type

func main() {
    a := number{5.0}
    b := nr{5.0}
    // var i float32 = b   // compile-error: cannot use b (type nr) as type float32 in assignment
    // var i = float32(b)  // compile-error: cannot convert b (type nr) to type float32
    // var c number = b    // compile-error: cannot use b (type nr) as type number in assignment
    // needs a conversion:
    var c = number(b)
    fmt.Println(a, b, c)
}
```

输出：

    {5} {5} {5}

**练习 10.1** `vcard.go`：

定义结构体 `Address` 和 `VCard`，后者包含一个人的名字、地址编号、出生日期和图像，试着选择正确的数据类型。构建一个自己的 `vcard` 并打印它的内容。

> 提示：
> `VCard` 必须包含住址，它应该以值类型还是以指针类型放在 `VCard` 中呢？
> 第二种会好点，因为它占用内存少。包含一个名字和两个指向地址的指针的 `Address` 结构体可以使用 `%v` 打印：
> `{Kersschot 0x126d2b80 0x126d2be0}`

```go
package main

import (
	"fmt"
	"time"
)

type Address struct {
	Street           string
	HouseNumber      uint32
	HouseNumberAddOn string
	POBox            string
	ZipCode          string
	City             string
	Country          string
}

type VCard struct {
	FirstName string
	LastName  string
	NickName  string
	BirtDate  time.Time
	Photo     string
	Addresses map[string]*Address
}

func main() {
	addr1 := &Address{"Elfenstraat", 12, "", "", "2600", "Mechelen", "België"}
	addr2 := &Address{"Heideland", 28, "", "", "2640", "Mortsel", "België"}
	addrs := make(map[string]*Address)
	addrs["youth"] = addr1
	addrs["now"] = addr2
	birthdt := time.Date(1956, 1, 17, 15, 4, 5, 0, time.Local)
	photo := "MyDocuments/MyPhotos/photo1.jpg"
	vcard := &VCard{"Ivo", "Balbaert", "", birthdt, photo, addrs}
	fmt.Printf("Here is the full VCard: %v\n", vcard)
	fmt.Printf("My Addresses are:\n %v\n %v", addr1, addr2)
}

/* Output:
Here is the full VCard: &{Ivo Balbaert  Sun Jan 17 15:04:05 +0000 1956 MyDocuments/MyPhotos/photo1.jpg map[now:0x126d57c0 youth:0x126d5500]}
My Addresses are:
 &{Elfenstraat 12   2600 Mechelen België}
 &{Heideland 28   2640 Mortsel België}
*/
```

**练习 10.2** `personex1.go`：

修改 `personex1.go`，使它的参数 `upPerson` 不是一个指针，解释下二者的区别。

```go
package main

import (
	"fmt"
	"strings"
)

type Person struct {
	firstName string
	lastName  string
}

func upPerson(p Person) {
	p.firstName = strings.ToUpper(p.firstName)
	p.lastName = strings.ToUpper(p.lastName)
}

func main() {
	// 1- struct as a value type:
	var pers1 Person
	pers1.firstName = "Chris"
	pers1.lastName = "Woodward"
	upPerson(pers1)
	fmt.Printf("The name of the person is %s %s\n", pers1.firstName, pers1.lastName)
	// 2 - struct as a pointer:
	pers2 := new(Person)
	pers2.firstName = "Chris"
	pers2.lastName = "Woodward"
	upPerson(*pers2)
	fmt.Printf("The name of the person is %s %s\n", pers2.firstName, pers2.lastName)
	// 3 - struct as a literal:
	pers3 := &Person{"Chris", "Woodward"}
	upPerson(*pers3)
	fmt.Printf("The name of the person is %s %s\n", pers3.firstName, pers3.lastName)
}

/* Output:
The name of the person is Chris Woodward
The name of the person is Chris Woodward
The name of the person is Chris Woodward
*/
```

**练习 10.3** `point.go`：

使用坐标 X、Y 定义一个二维 `Point` 结构体。同样地，对一个三维点使用它的极坐标定义一个 `Polar` 结构体。实现一个 `Abs()` 方法来计算一个 `Point` 表示的向量的长度，实现一个 `Scale` 方法，它将点的坐标乘以一个尺度因子（提示：使用 `math` 包里的 `Sqrt` 函数）（function Scale that multiplies the coordinates of a point with a scale factor）。

```go
// point.go
package main

import (
	"fmt"
	"math"
)

type Point struct {
	X, Y float64
}

type Point3 struct {
	X, Y, Z float64
}

type Polar struct {
	R, T float64
}

func Abs(p *Point) float64 {
	return math.Sqrt(float64(p.X*p.X + p.Y*p.Y))
}

func Scale(p *Point, s float64) (q Point) {
	q.X = p.X * s
	q.Y = p.Y * s
	return
}

func main() {
	p1 := new(Point)
	p1.X = 3
	p1.Y = 4
	fmt.Printf("The length of the vector p1 is: %f\n", Abs(p1))

	p2 := &Point{4, 5}
	fmt.Printf("The length of the vector p2 is: %f\n", Abs(p2))

	q := Scale(p1, 5)
	fmt.Printf("The length of the vector q is: %f\n", Abs(&q))
	fmt.Printf("Point p1 scaled by 5 has the following coordinates: X %f - Y %f", q.X, q.Y)
}

/* Output:
The length of the vector p1 is: 5.000000
The length of the vector p2 is: 6.403124
The length of the vector q is: 25.000000
Point p1 scaled by 5 has the following coordinates: X 15.000000 - Y 20.000000
*/
```

**练习 10.4** `rectangle.go`：

定义一个 `Rectangle` 结构体，它的长和宽是 `int` 类型，并定义方法 `Area()` 和 `Perimeter()`，然后进行测试。

```go
// rectangle.go
package main

import "fmt"

type Rectangle struct {
	length, width int
}

func (r *Rectangle) Area() int {
	return r.length * r.width
}

func (r *Rectangle) Perimeter() int {
	return 2 * (r.length + r.width)
}

func main() {
	r1 := Rectangle{4, 3}
	fmt.Println("Rectangle is: ", r1)
	fmt.Println("Rectangle area is: ", r1.Area())
	fmt.Println("Rectangle perimeter is: ", r1.Perimeter())
}

/* Output:
Rectangle is:  {4 3}
Rectangle area is:  12
Rectangle perimeter is:  14
*/
```

## 10.2 使用工厂方法创建结构体实例

### 10.2.1 结构体工厂

Go 语言不支持面向对象编程语言中那样的构造子方法，但是可以很容易的在 Go 中实现 “构造子工厂”方法。为了方便通常会为类型定义一个工厂，按惯例，工厂的名字以 `new` 或 `New` 开头。假设定义了如下的 `File` 结构体类型：

```go
type File struct {
    fd      int     // 文件描述符
    name    string  // 文件名
}
```

下面是这个结构体类型对应的工厂方法，它返回一个指向结构体实例的指针：

```go
func NewFile(fd int, name string) *File {
    if fd < 0 {
        return nil
    }

    return &File{fd, name}
}
```

然后这样调用它：

```go
f := NewFile(10, "./test.txt")
```

在 Go 语言中常常像上面这样在工厂方法里使用初始化来简便的实现构造函数。

如果 `File` 是一个结构体类型，那么表达式 `new(File)` 和 `&File{}` 是等价的。

这可以和大多数面向对象编程语言中笨拙的初始化方式做个比较：`File f = new File(...)`。

我们可以说是工厂实例化了类型的一个对象，就像在基于类的OO语言中那样。

如果想知道结构体类型T的一个实例占用了多少内存，可以使用：`size := unsafe.Sizeof(T{})`。

**如何强制使用工厂方法**

通过应用可见性规则参考[4.2.1节](04.2.md)、[9.5 节](09.5.md)就可以禁止使用 new 函数，强制用户使用工厂方法，从而使类型变成私有的，就像在面向对象语言中那样。

```go
type matrix struct {
    ...
}

func NewMatrix(params) *matrix {
    m := new(matrix) // 初始化 m
    return m
}
```

在其他包里使用工厂方法：

```go
package main
import "matrix"
...
wrong := new(matrix.matrix)     // 编译失败（matrix 是私有的）
right := matrix.NewMatrix(...)  // 实例化 matrix 的唯一方式
```

### 10.2.2 map 和 struct vs new() 和 make()

`new` 和 `make` 这两个内置函数已经在第 [7.2.4](07.2.md) 节通过切片的例子说明过一次。

现在为止我们已经见到了可以使用 `make()` 的三种类型中的其中两个：

```go
slices  /  maps / channels（见第 14 章）
```

下面的例子说明了在映射上使用 `new` 和 `make` 的区别以及可能发生的错误：

示例 10.4 `new_make.go`（不能编译）

```go
package main

type Foo map[string]string
type Bar struct {
    thingOne string
    thingTwo int
}

func main() {
    // OK
    y := new(Bar)
    (*y).thingOne = "hello"
    (*y).thingTwo = 1

    // NOT OK
    z := make(Bar) // 编译错误：cannot make type Bar
    (*z).thingOne = "hello"
    (*z).thingTwo = 1

    // OK
    x := make(Foo)
    x["x"] = "goodbye"
    x["y"] = "world"

    // NOT OK
    u := new(Foo)
    (*u)["x"] = "goodbye" // 运行时错误!! panic: assignment to entry in nil map
    (*u)["y"] = "world"
}
```

试图 `make()` 一个结构体变量，会引发一个编译错误，这还不是太糟糕，但是 `new()` 一个映射并试图使用数据填充它，将会引发运行时错误！ 因为 `new(Foo)` 返回的是一个指向 `nil` 的指针，它尚未被分配内存。所以在使用 `map` 时要特别谨慎。

## 10.3 使用自定义包中的结构体

下面的例子中，`main.go` 使用了一个结构体，它来自 `struct_pack` 下的包 `structPack`。

示例 10.5 [structPack.go](examples/chapter_10/struct_pack/structPack.go)：

```go
package structPack

type ExpStruct struct {
    Mi1 int
    Mf1 float32
}
```

示例 10.6 [main.go](examples/chapter_10/main.go)：

```go
package main
import (
    "fmt"
    "./struct_pack/structPack"
)

func main() {
    struct1 := new(structPack.ExpStruct)
    struct1.Mi1 = 10
    struct1.Mf1 = 16.

    fmt.Printf("Mi1 = %d\n", struct1.Mi1)
    fmt.Printf("Mf1 = %f\n", struct1.Mf1)
}
```

输出：

    Mi1 = 10
    Mf1 = 16.000000

## 10.4 带标签的结构体

结构体中的字段除了有名字和类型外，还可以有一个可选的标签（`tag`）：它是一个附属于字段的字符串，可以是文档或其他的重要标记。标签的内容不可以在一般的编程中使用，只有包 `reflect` 能获取它。我们将在下一章（第 [11.10 节](11.10.md)）中深入的探讨 `reflect`包，它可以在运行时自省类型、属性和方法，比如：在一个变量上调用 `reflect.TypeOf()` 可以获取变量的正确类型，如果变量是一个结构体类型，就可以通过 `Field` 来索引结构体的字段，然后就可以使用 `Tag` 属性。

示例 10.7 [struct_tag.go](examples/chapter_10/struct_tag.go)：

```go
package main

import (
	"fmt"
	"reflect"
)

type TagType struct { // tags
	field1 bool   "An important answer"
	field2 string "The name of the thing"
	field3 int    "How much there are"
}

func main() {
	tt := TagType{true, "Barak Obama", 1}
	for i := 0; i < 3; i++ {
		refTag(tt, i)
	}
}

func refTag(tt TagType, ix int) {
	ttType := reflect.TypeOf(tt)
	ixField := ttType.Field(ix)
	fmt.Printf("%v\n", ixField.Tag)
}
```

输出：

    An important answer
    The name of the thing
    How much there are

## 10.5 匿名字段和内嵌结构体

### 10.5.1 定义

结构体可以包含一个或多个 **匿名（或内嵌）字段**，即这些字段没有显式的名字，只有字段的类型是必须的，此时类型就是字段的名字。匿名字段本身可以是一个结构体类型，即 **结构体可以包含内嵌结构体**。

可以粗略地将这个和面向对象语言中的继承概念相比较，随后将会看到它被用来模拟类似继承的行为。Go 语言中的继承是通过内嵌或组合来实现的，所以可以说，在 Go 语言中，相比较于继承，组合更受青睐。

考虑如下的程序：

示例 10.8 [structs_anonymous_fields.go](examples/chapter_10/structs_anonymous_fields.go)：

```go
package main

import "fmt"

type innerS struct {
	in1 int
	in2 int
}

type outerS struct {
	b    int
	c    float32
	int  // anonymous field
	innerS //anonymous field
}

func main() {
	outer := new(outerS)
	outer.b = 6
	outer.c = 7.5
	outer.int = 60
	outer.in1 = 5
	outer.in2 = 10

	fmt.Printf("outer.b is: %d\n", outer.b)
	fmt.Printf("outer.c is: %f\n", outer.c)
	fmt.Printf("outer.int is: %d\n", outer.int)
	fmt.Printf("outer.in1 is: %d\n", outer.in1)
	fmt.Printf("outer.in2 is: %d\n", outer.in2)

	// 使用结构体字面量
	outer2 := outerS{6, 7.5, 60, innerS{5, 10}}
	fmt.Println("outer2 is:", outer2)
}
```

输出：

    outer.b is: 6
    outer.c is: 7.500000
    outer.int is: 60
    outer.in1 is: 5
    outer.in2 is: 10
    outer2 is:{6 7.5 60 {5 10}}

通过类型 `outer.int` 的名字来获取存储在匿名字段中的数据，于是可以得出一个结论：在一个结构体中对于每一种数据类型**只能有一个匿名字段**。

### 10.5.2 内嵌结构体

同样地结构体也是一种数据类型，所以它也可以作为一个匿名字段来使用，如同上面例子中那样。外层结构体通过 `outer.in1` 直接进入内层结构体的字段，内嵌结构体甚至可以来自其他包。内层结构体被简单的插入或者内嵌进外层结构体。这个简单的“继承”机制提供了一种方式，使得可以从另外一个或一些类型继承部分或全部实现。

另外一个例子：

示例 10.9 [embedd_struct.go](examples/chapter_10/embedd_struct.go)：

```go
package main

import "fmt"

type A struct {
	ax, ay int
}

type B struct {
	A
	bx, by float32
}

func main() {
	b := B{A{1, 2}, 3.0, 4.0}
	fmt.Println(b.ax, b.ay, b.bx, b.by)
	fmt.Println(b.A)
}
```

输出：

    1 2 3 4
    {1 2}

**练习 10.5** `anonymous_struct.go`：

创建一个结构体，它有一个具名的 `float` 字段，2 个匿名字段，类型分别是 `int` 和 `string`。通过结构体字面量新建一个结构体实例并打印它的内容。

```go
package main

import "fmt"

type C struct {
	x float32
	int
	string
}

func main() {
	c := C{3.14, 7, "hello"}
	fmt.Println(c.x, c.int, c.string) // output: 3.14 7 hello
	fmt.Println(c)                    // output: {3.14 7 hello}
}
```

### 10.5.3 命名冲突

当两个字段拥有相同的名字（可能是继承来的名字）时该怎么办呢？

1. 外层名字会覆盖内层名字（但是两者的内存空间都保留），这提供了一种重载字段或方法的方式；
2. 如果相同的名字在同一级别出现了两次，如果这个名字被程序使用了，将会引发一个错误（不使用没关系）。没有办法来解决这种问题引起的二义性，必须由程序员自己修正。

例子：

```go
type A struct {a int}
type B struct {a, b int}

type C struct {A; B}
var c C
```

规则 2：使用 `c.a` 是错误的，到底是 `c.A.a` 还是 `c.B.a` 呢？会导致编译器错误：`ambiguous DOT reference c.a disambiguate with either c.A.a or c.B.a`。

```go
type D struct {B; b float32}
var d D
```

规则1：使用 `d.b` 是没问题的：它是 `float32`，而不是 `B` 的 `b`。如果想要内层的 `b` 可以通过 `d.B.b` 得到。

## 10.6 方法

### 10.6.1 方法是什么

在 Go 语言中，结构体就像是类的一种简化形式，那么面向对象程序员可能会问：类的方法在哪里呢？在 Go 中有一个概念，它和方法有着同样的名字，并且大体上意思相同：Go 方法是作用在接收者（`receiver`）上的一个函数，接收者是某种类型的变量。因此方法是一种特殊类型的函数。

接收者类型可以是（几乎）任何类型，不仅仅是结构体类型：任何类型都可以有方法，甚至可以是函数类型，可以是 `int`、`bool`、`string` 或数组的别名类型。但是接收者不能是一个接口类型（参考 第 11 章），因为接口是一个抽象定义，但是方法却是具体实现；如果这样做会引发一个编译错误：`invalid receiver type…`。

最后接收者不能是一个指针类型，但是它可以是任何其他允许类型的指针。

一个类型加上它的方法等价于面向对象中的一个类。一个重要的区别是：在 Go 中，类型的代码和绑定在它上面的方法的代码可以不放置在一起，它们可以存在在不同的源文件，唯一的要求是：它们必须是同一个包的。

类型 `T`（或 `\*T`）上的所有方法的集合叫做类型 `T`（或 `\*T`）的方法集（`method set`）。

因为方法是函数，所以同样的，不允许方法重载，即对于一个类型只能有一个给定名称的方法。但是如果基于接收者类型，是有重载的：具有同样名字的方法可以在 2 个或多个不同的接收者类型上存在，比如在同一个包里这么做是允许的：

```go
func (a *denseMatrix) Add(b Matrix) Matrix
func (a *sparseMatrix) Add(b Matrix) Matrix
```

别名类型没有原始类型上已经定义过的方法。

定义方法的一般格式如下：

```go
func (recv receiver_type) methodName(parameter_list) (return_value_list) { ... }
```

在方法名之前，`func` 关键字之后的括号中指定 `receiver`。

如果 `recv` 是 `receiver` 的实例，`Method1` 是它的方法名，那么方法调用遵循传统的 `object.name` 选择器符号：**recv.Method1()**。

如果 `recv` 是一个指针，Go 会自动解引用。

如果方法不需要使用 `recv` 的值，可以用 **_** 替换它，比如：

```go
func (_ receiver_type) methodName(parameter_list) (return_value_list) { ... }
```

`recv` 就像是面向对象语言中的 `this` 或 `self`，但是 Go 中并没有这两个关键字。随个人喜好，你可以使用 `this` 或 `self` 作为 `receiver` 的名字。下面是一个结构体上的简单方法的例子：

示例 10.10 method .go：

```go
package main

import "fmt"

type TwoInts struct {
	a int
	b int
}

func main() {
	two1 := new(TwoInts)
	two1.a = 12
	two1.b = 10

	fmt.Printf("The sum is: %d\n", two1.AddThem())
	fmt.Printf("Add them to the param: %d\n", two1.AddToParam(20))

	two2 := TwoInts{3, 4}
	fmt.Printf("The sum is: %d\n", two2.AddThem())
}

func (tn *TwoInts) AddThem() int {
	return tn.a + tn.b
}

func (tn *TwoInts) AddToParam(param int) int {
	return tn.a + tn.b + param
}
```

输出：

    The sum is: 22
    Add them to the param: 42
    The sum is: 7

下面是非结构体类型上方法的例子：

示例 10.11 method2.go：

```go
package main

import "fmt"

type IntVector []int

func (v IntVector) Sum() (s int) {
	for _, x := range v {
		s += x
	}
	return
}

func main() {
	fmt.Println(IntVector{1, 2, 3}.Sum()) // 输出是6
}
```

**练习 10.6** `employee_salary.go`

定义结构体 `employee`，它有一个 `salary` 字段，给这个结构体定义一个方法 `giveRaise` 来按照指定的百分比增加薪水。

```go
// methods1.go
package main

import "fmt"

/* basic data structure upon with we'll define methods */
type employee struct {
	salary float32
}

/* a method which will add a specified percent to an
   employees salary */
func (this *employee) giveRaise(pct float32) {
	this.salary += this.salary * pct
}

func main() {
	/* create an employee instance */
	var e = new(employee)
	e.salary = 100000
	/* call our method */
	e.giveRaise(0.04)
	fmt.Printf("Employee now makes %f", e.salary)
}

// Employee now makes 104000.000000
```

**练习 10.7** `iteration_list.go`

下面这段代码有什么错？

```go
package main

import "container/list"

func (p *list.List) Iter() {
	// ...
}

func main() {
	lst := new(list.List)
	for _= range lst.Iter() {
	}
}
```

类型和作用在它上面定义的方法必须在同一个包里定义，这就是为什么不能在 `int`、`float` 或类似这些的类型上定义方法。试图在 `int` 类型上定义方法会得到一个编译错误：

```go
cannot define new methods on non-local type int
```

比如想在 `time.Time` 上定义如下方法：

```go
func (t time.Time) first3Chars() string {
	return time.LocalTime().String()[0:3]
}
```

类型在其他的，或是非本地的包里定义，在它上面定义方法都会得到和上面同样的错误。

但是有一个间接的方式：可以先定义该类型（比如：`int` 或 `float`）的别名类型，然后再为别名类型定义方法。或者像下面这样将它作为匿名类型嵌入在一个新的结构体中。当然方法只在这个别名类型上有效。

示例 10.12 `method_on_time.go`：

```go
package main

import (
	"fmt"
	"time"
)

type myTime struct {
	time.Time //anonymous field
}

func (t myTime) first3Chars() string {
	return t.Time.String()[0:3]
}
func main() {
	m := myTime{time.Now()}
	// 调用匿名Time上的String方法
	fmt.Println("Full time now:", m.String())
	// 调用myTime.first3Chars
	fmt.Println("First 3 chars:", m.first3Chars())
}

/* Output:
Full time now: Mon Oct 24 15:34:54 Romance Daylight Time 2011
First 3 chars: Mon
*/
```

### 10.6.2 函数和方法的区别

函数将变量作为参数：**`Function1(recv)`**

方法在变量上被调用：**`recv.Method1()`**

在接收者是指针时，方法可以改变接收者的值（或状态），这点函数也可以做到（当参数作为指针传递，即通过引用调用时，函数也可以改变参数的状态）。

**不要忘记 Method1 后边的括号 ()，否则会引发编译器错误：`method recv.Method1 is not an expression, must be called`**

接收者必须有一个显式的名字，这个名字必须在方法中被使用。

**receiver_type** 叫做 **（接收者）基本类型**，这个类型必须在和方法同样的包中被声明。

在 Go 中，（接收者）类型关联的方法不写在类型结构里面，就像类那样；耦合更加宽松；类型和方法之间的关联由接收者来建立。

**方法没有和数据定义（结构体）混在一起：它们是正交的类型；表示（数据）和行为（方法）是独立的。**

### 10.6.3 指针或值作为接收者

鉴于性能的原因，`recv` 最常见的是一个指向 `receiver_type` 的指针（因为我们不想要一个实例的拷贝，如果按值调用的话就会是这样），特别是在 `receiver` 类型是结构体时，就更是如此了。

如果想要方法改变接收者的数据，就在接收者的指针类型上定义该方法。否则，就在普通的值类型上定义方法。

下面的例子 `pointer_value.go` 作了说明：`change()`接受一个指向 B 的指针，并改变它内部的成员；`write()` 通过拷贝接受 B 的值并只输出B的内容。注意 Go 为我们做了探测工作，我们自己并没有指出是否在指针上调用方法，Go 替我们做了这些事情。`b1` 是值而 `b2` 是指针，方法都支持运行了。

示例 10.13 `pointer_value.go`：

```go
package main

import (
	"fmt"
)

type B struct {
	thing int
}

func (b *B) change() { b.thing = 1 }

func (b B) write() string { return fmt.Sprint(b) }

func main() {
	var b1 B // b1是值
	b1.change()
	fmt.Println(b1.write())

	b2 := new(B) // b2是指针
	b2.change()
	fmt.Println(b2.write())
}

/* 输出：
{1}
{1}
*/
```

试着在 `write()` 中改变接收者`b`的值：将会看到它可以正常编译，但是开始的 `b` 没有被改变。

我们知道方法将指针作为接收者不是必须的，如下面的例子，我们只是需要 `Point3` 的值来做计算：

```go
type Point3 struct { x, y, z float64 }
// A method on Point3
func (p Point3) Abs() float64 {
    return math.Sqrt(p.x*p.x + p.y*p.y + p.z*p.z)
}
```

这样做稍微有点昂贵，因为 `Point3` 是作为值传递给方法的，因此传递的是它的拷贝，这在 Go 中是合法的。也可以在指向这个类型的指针上调用此方法（会自动解引用）。

假设 `p3` 定义为一个指针：`p3 := &Point{ 3, 4, 5}`。

可以使用 `p3.Abs()` 来替代 `(*p3).Abs()`。

像例子 10.10（`method1.go`）中接收者类型是 `*TwoInts` 的方法 `AddThem()`，它能在类型 `TwoInts` 的值上被调用，这是自动间接发生的。

因此 `two2.AddThem` 可以替代 `(&two2).AddThem()`。

在值和指针上调用方法：

可以有连接到类型的方法，也可以有连接到类型指针的方法。

但是这没关系：对于类型 T，如果在 \*T 上存在方法 `Meth()`，并且 `t` 是这个类型的变量，那么 `t.Meth()` 会被自动转换为 `(&t).Meth()`。

**指针方法和值方法都可以在指针或非指针上被调用**，如下面程序所示，类型 `List` 在值上有一个方法 `Len()`，在指针上有一个方法 `Append()`，但是可以看到两个方法都可以在两种类型的变量上被调用。

示例 10.14 methodset1.go：

```go
package main

import (
	"fmt"
)

type List []int

func (l List) Len() int        { return len(l) }
func (l *List) Append(val int) { *l = append(*l, val) }

func main() {
	// 值
	var lst List
	lst.Append(1)
	fmt.Printf("%v (len: %d)", lst, lst.Len()) // [1] (len: 1)

	// 指针
	plst := new(List)
	plst.Append(2)
	fmt.Printf("%v (len: %d)", plst, plst.Len()) // &[2] (len: 1)
}
```

### 10.6.4 方法和未导出字段

考虑 `person2.go` 中的 `person` 包：类型 `Person` 被明确的导出了，但是它的字段没有被导出。例如在 `use_person2.go` 中 `p.firstName` 就是错误的。该如何在另一个程序中修改或者只是读取一个 `Person` 的名字呢？

这可以通过面向对象语言一个众所周知的技术来完成：提供 `getter` 和 `setter` 方法。对于 `setter` 方法使用 `Set` 前缀，对于 `getter` 方法只使用成员名。

示例 10.15 person2.go：

```go
package person

type Person struct {
	firstName string
	lastName  string
}

func (p *Person) FirstName() string {
	return p.firstName
}

func (p *Person) SetFirstName(newName string) {
	p.firstName = newName
}
```

示例 10.16—use_person2.go：

```go
package main

import (
	"./person"
	"fmt"
)

func main() {
	p := new(person.Person)
	// p.firstName undefined
	// (cannot refer to unexported field or method firstName)
	// p.firstName = "Eric"
	p.SetFirstName("Eric")
	fmt.Println(p.FirstName()) // Output: Eric
}
```

**并发访问对象**

对象的字段（属性）不应该由 2 个或 2 个以上的不同线程在同一时间去改变。如果在程序发生这种情况，为了安全并发访问，可以使用包 `sync`（参考第 9.3 节）中的方法。在第 14.17 节中我们会通过 goroutines 和 channels 探索另一种方式。

### 10.6.5 内嵌类型的方法和继承

当一个匿名类型被内嵌在结构体中时，匿名类型的可见方法也同样被内嵌，这在效果上等同于外层类型 **继承** 了这些方法：**将父类型放在子类型中来实现亚型**。这个机制提供了一种简单的方式来模拟经典面向对象语言中的子类和继承相关的效果，也类似 Ruby 中的混入（`mixin`）。

下面是一个示例（可以在练习 10.8 中进一步学习）：假定有一个 `Engine` 接口类型，一个 `Car` 结构体类型，它包含一个 `Engine` 类型的匿名字段：

```go
type Engine interface {
	Start()
	Stop()
}

type Car struct {
	Engine
}
```

我们可以构建如下的代码：

```go
func (c *Car) GoToWorkIn() {
	// get in car
	c.Start()
	// drive to work
	c.Stop()
	// get out of car
}
```

下面是 `method3.go` 的完整例子，它展示了内嵌结构体上的方法可以直接在外层类型的实例上调用：

```go
package main

import (
	"fmt"
	"math"
)

type Point struct {
	x, y float64
}

func (p *Point) Abs() float64 {
	return math.Sqrt(p.x*p.x + p.y*p.y)
}

type NamedPoint struct {
	Point
	name string
}

func main() {
	n := &NamedPoint{Point{3, 4}, "Pythagoras"}
	fmt.Println(n.Abs()) // 打印5
}
```

内嵌将一个已存在类型的字段和方法注入到了另一个类型里：匿名字段上的方法“晋升”成为了外层类型的方法。当然类型可以有只作用于本身实例而不作用于内嵌“父”类型上的方法，

可以覆写方法（像字段一样）：和内嵌类型方法具有同样名字的外层类型的方法会覆写内嵌类型对应的方法。

在示例 10.18 method4.go 中添加：

```go
func (n *NamedPoint) Abs() float64 {
	return n.Point.Abs() * 100.
}
```

现在 `fmt.Println(n.Abs())` 会打印 `500`。

因为一个结构体可以嵌入多个匿名类型，所以实际上我们可以有一个简单版本的多重继承，就像：`type Child struct { Father; Mother}`。在第 10.6.7 节中会进一步讨论这个问题。

结构体内嵌和自己在同一个包中的结构体时，可以彼此访问对方所有的字段和方法。

**练习 10.8** `inheritance_car.go`

创建一个上面 `Car` 和 `Engine` 可运行的例子，并且给 `Car` 类型一个 `wheelCount` 字段和一个 `numberOfWheels()` 方法。

创建一个 `Mercedes` 类型，它内嵌 `Car`，并新建 `Mercedes` 的一个实例，然后调用它的方法。

然后仅在 `Mercedes` 类型上创建方法 `sayHiToMerkel()` 并调用它。

```go
// inheritance_car.go
package main

import (
	"fmt"
)

type Engine interface {
	Start()
	Stop()
}

type Car struct {
	wheelCount int
	Engine
}

// define a behavior for Car
func (car Car) numberOfWheels() int {
	return car.wheelCount
}

type Mercedes struct {
	Car //anonymous field Car
}

// a behavior only available for the Mercedes
func (m *Mercedes) sayHiToMerkel() {
	fmt.Println("Hi Angela!")
}

func (c *Car) Start() {
	fmt.Println("Car is started")
}

func (c *Car) Stop() {
	fmt.Println("Car is stopped")
}

func (c *Car) GoToWorkIn() {
	// get in car
	c.Start()
	// drive to work
	c.Stop()
	// get out of car
}

func main() {
	m := Mercedes{Car{4, nil}}
	fmt.Println("A Mercedes has this many wheels: ", m.numberOfWheels())
	m.GoToWorkIn()
	m.sayHiToMerkel()
}

/* Output:
A Mercedes has this many wheels:  4
Car is started
Car is stopped
Hi Angela!
*/
```

### 10.6.6 如何在类型中嵌入功能

主要有两种方法来实现在类型中嵌入功能：

A：聚合（或组合）：包含一个所需功能类型的具名字段。

B：内嵌：内嵌（匿名地）所需功能类型，像前一节 10.6.5 所演示的那样。

为了使这些概念具体化，假设有一个 `Customer` 类型，我们想让它通过 `Log` 类型来包含日志功能，`Log` 类型只是简单地包含一个累积的消息（当然它可以是复杂的）。如果想让特定类型都具备日志功能，你可以实现一个这样的 `Log` 类型，然后将它作为特定类型的一个字段，并提供 `Log()`，它返回这个日志的引用。

方式 A 可以通过如下方法实现（使用了第 10.7 节中的 `String()` 功能）：

示例 10.19 embed_func1.go：

```go
package main

import (
	"fmt"
)

type Log struct {
	msg string
}

type Customer struct {
	Name string
	log  *Log
}

func main() {
	c := new(Customer)
	c.Name = "Barak Obama"
	c.log = new(Log)
	c.log.msg = "1 - Yes we can!"
	// shorter
	c = &Customer{"Barak Obama", &Log{"1 - Yes we can!"}}
	// fmt.Println(c) &{Barak Obama 1 - Yes we can!}
	c.Log().Add("2 - After me the world will be a better place!")
	//fmt.Println(c.log)
	fmt.Println(c.Log())

}

func (l *Log) Add(s string) {
	l.msg += "\n" + s
}

func (l *Log) String() string {
	return l.msg
}

func (c *Customer) Log() *Log {
	return c.log
}
```

输出：

    1 - Yes we can!
    2 - After me the world will be a better place!

相对的方式 B 可能会像这样：

```go
package main

import (
	"fmt"
)

type Log struct {
	msg string
}

type Customer struct {
	Name string
	Log
}

func main() {
	c := &Customer{"Barak Obama", Log{"1 - Yes we can!"}}
	c.Add("2 - After me the world will be a better place!")
	fmt.Println(c)

}

func (l *Log) Add(s string) {
	l.msg += "\n" + s
}

func (l *Log) String() string {
	return l.msg
}

func (c *Customer) String() string {
	return c.Name + "\nLog:" + fmt.Sprintln(c.Log)
}
```

输出：

    Barak Obama
    Log:{1 - Yes we can!
    2 - After me the world will be a better place!}

内嵌的类型不需要指针，`Customer` 也不需要 `Add` 方法，它使用 `Log` 的 `Add` 方法，`Customer` 有自己的 `String` 方法，并且在它里面调用了 `Log` 的 `String` 方法。

如果内嵌类型嵌入了其他类型，也是可以的，那些类型的方法可以直接在外层类型中使用。

因此一个好的策略是创建一些小的、可复用的类型作为一个工具箱，用于组成域类型。

### 10.6.7 多重继承

多重继承指的是类型获得多个父类型行为的能力，它在传统的面向对象语言中通常是不被实现的（C++ 和 Python 例外）。因为在类继承层次中，多重继承会给编译器引入额外的复杂度。但是在 Go 语言中，通过在类型中嵌入所有必要的父类型，可以很简单的实现多重继承。

作为一个例子，假设有一个类型 `CameraPhone`，通过它可以 `Call()`，也可以 `TakeAPicture()`，但是第一个方法属于类型 `Phone`，第二个方法属于类型 `Camera`。

只要嵌入这两个类型就可以解决这个问题，如下所示：

```go
package main

import (
	"fmt"
)

type Camera struct{}

func (c *Camera) TakeAPicture() string {
	return "Click"
}

type Phone struct{}

func (p *Phone) Call() string {
	return "Ring Ring"
}

type CameraPhone struct {
	Camera
	Phone
}

func main() {
	cp := new(CameraPhone)
	fmt.Println("Our new CameraPhone exhibits multiple behaviors...")
	fmt.Println("It exhibits behavior of a Camera: ", cp.TakeAPicture())
	fmt.Println("It works like a Phone too: ", cp.Call())
}
```

输出：

    Our new CameraPhone exhibits multiple behaviors...
    It exhibits behavior of a Camera: Click
    It works like a Phone too: Ring Ring

**练习 10.9** `point_methods.go`：

从 `point.go` 开始（第 10.1 节的练习）：使用方法来实现 `Abs()` 和 `Scale()`函数，`Point` 作为方法的接收者类型。也为 `Point3` 和 `Polar` 实现 `Abs()` 方法。完成了 `point.go` 中同样的事情，只是这次通过方法。

```go
// float64 is necessary as input to math.Sqrt()
package main

import (
	"fmt"
	"math"
)

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
	R, T float64
}

func (p Polar) Abs() float64 { return p.R }

func main() {
	p1 := new(Point)
	p1.X = 3
	p1.Y = 4
	fmt.Printf("The length of the vector p1 is: %f\n", p1.Abs())

	p2 := &Point{4, 5}
	fmt.Printf("The length of the vector p2 is: %f\n", p2.Abs())

	p1.Scale(5)
	fmt.Printf("The length of the vector p1 after scaling is: %f\n", p1.Abs())
	fmt.Printf("Point p1 after scaling has the following coordinates: X %f - Y %f", p1.X, p1.Y)
}

/* Output:
The length of the vector p1 is: 5.000000
The length of the vector p2 is: 6.403124
The length of the vector p1 after scaling is: 25.000000
Point p1 after scaling has the following coordinates: X 15.000000 - Y 20.000000
*/
```

**练习 10.10** `inherit_methods.go`：

定义一个结构体类型 `Base`，它包含一个字段 `id`，方法 `Id()` 返回 `id`，方法 `SetId()` 修改 `id`。结构体类型 `Person` 包含 `Base`，及 `FirstName` 和 `LastName` 字段。结构体类型 `Employee` 包含一个 `Person` 和 `salary` 字段。

创建一个 `employee` 实例，然后显示它的 `id`。

```go
package main

import "fmt"

type Base struct {
	id string
}

func (b *Base) Id() string {
	return b.id
}

func (b *Base) SetId(id string) {
	b.id = id
}

type Person struct {
	Base
	FirstName string
	LastName  string
}

type Employee struct {
	Person
	salary float32
}

func main() {
	idjb := Base{"007"}
	jb := Person{idjb, "James", "Bond"}
	e := &Employee{jb, 100000.}
	fmt.Printf("ID of our hero: %v\n", e.Id())
	// Change the id:
	e.SetId("007B")
	fmt.Printf("The new ID of our hero: %v\n", e.Id())
}

/* Output:
ID of our hero: 007
The new ID of our hero: 007B
*/
```

**练习 10.11** `magic.go`：

首先预测一下下面程序的结果，然后动手实验下：

```go
package main

import (
	"fmt"
)

type Base struct{}

func (Base) Magic() {
	fmt.Println("base magic")
}

func (self Base) MoreMagic() {
	self.Magic()
	self.Magic()
}

type Voodoo struct {
	Base
}

func (Voodoo) Magic() {
	fmt.Println("voodoo magic")
}

func main() {
	v := new(Voodoo)
	v.Magic()
	v.MoreMagic()
}
```

### 10.6.8 通用方法和方法命名

在编程中一些基本操作会一遍又一遍的出现，比如打开（`Open`）、关闭（`Close`）、读（`Read`）、写（`Write`）、排序（`Sort`）等等，并且它们都有一个大致的意思：打开（`Open`）可以作用于一个文件、一个网络连接、一个数据库连接等等。具体的实现可能千差万别，但是基本的概念是一致的。在 Go 语言中，通过使用接口（参考 第 11 章），标准库广泛的应用了这些规则，在标准库中这些通用方法都有一致的名字，比如 `Open()`、`Read()`、`Write()`等。想写规范的 Go 程序，就应该遵守这些约定，给方法合适的名字和签名，就像那些通用方法那样。这样做会使 Go 开发的软件更加具有一致性和可读性。比如：如果需要一个 `convert-to-string` 方法，应该命名为 `String()`，而不是 `ToString()`（参考第 10.7 节）。

### 10.6.9 和其他面向对象语言比较 Go 的类型和方法

在如 C++、Java、C# 和 Ruby 这样的面向对象语言中，方法在类的上下文中被定义和继承：在一个对象上调用方法时，运行时会检测类以及它的超类中是否有此方法的定义，如果没有会导致异常发生。

在 Go 语言中，这样的继承层次是完全没必要的：如果方法在此类型定义了，就可以调用它，和其他类型上是否存在这个方法没有关系。在这个意义上，Go 具有更大的灵活性。

下面的模式就很好的说明了这个问题：

![](images/10.6.9_fig10.4.jpg?raw=true)

Go 不需要一个显式的类定义，如同 Java、C++、C# 等那样，相反地，“类”是通过提供一组作用于一个共同类型的方法集来隐式定义的。类型可以是结构体或者任何用户自定义类型。

比如：我们想定义自己的 `Integer` 类型，并添加一些类似转换成字符串的方法，在 Go 中可以如下定义：

```go
type Integer int
func (i *Integer) String() string {
    return strconv.Itoa(int(*i))
}
```

在 Java 或 C# 中，这个方法需要和类 `Integer` 的定义放在一起，在 Ruby 中可以直接在基本类型 `int` 上定义这个方法。

**总结**

在 Go 中，类型就是类（数据和关联的方法）。Go 不知道类似面向对象语言的类继承的概念。继承有两个好处：代码复用和多态。

在 Go 中，代码复用通过组合和委托实现，多态通过接口的使用来实现：有时这也叫 **组件编程（Component Programming）**。

许多开发者说相比于类继承，Go 的接口提供了更强大、却更简单的多态行为。

> **备注**
>
> 如果真的需要更多面向对象的能力，看一下 [`goop`](https://github.com/losalamos/goop) 包（Go Object-Oriented Programming），它由 Scott Pakin 编写: 它给 Go 提供了 JavaScript 风格的对象（基于原型的对象），并且支持多重继承和类型独立分派，通过它可以实现你喜欢的其他编程语言里的一些结构。

**问题 10.1**

我们在某个类型的变量上使用点号调用一个方法：`variable.method()`，在使用 Go 以前，在哪儿碰到过面向对象的点号？

**问题 10.2**

a）假设定义： `type Integer int`，完成 `get()` 方法的方法体: `func (p Integer) get() int { ... }`。

b）定义： `func f(i int) {}; var v Integer` ，如何就 v 作为参数调用f？

c）假设 `Integer` 定义为 `type Integer struct {n int}`，完成 `get()` 方法的方法体：`func (p Integer) get() int { ... }`。

d）对于新定义的 `Integer`，和 b）中同样的问题。

## 10.7 类型的 String() 方法和格式化描述符

当定义了一个有很多方法的类型时，十之八九你会使用 `String()` 方法来定制类型的字符串形式的输出，换句话说：一种可阅读性和打印性的输出。如果类型定义了 `String()` 方法，它会被用在 `fmt.Printf()` 中生成默认的输出：等同于使用格式化描述符 `%v` 产生的输出。还有 `fmt.Print()` 和 `fmt.Println()` 也会自动使用 `String()` 方法。

我们使用第 10.4 节中程序的类型来进行测试：

示例 10.22 `method_string.go`：

```go
package main

import (
	"fmt"
	"strconv"
)

type TwoInts struct {
	a int
	b int
}

func main() {
	two1 := new(TwoInts)
	two1.a = 12
	two1.b = 10
	fmt.Printf("two1 is: %v\n", two1)
	fmt.Println("two1 is:", two1)
	fmt.Printf("two1 is: %T\n", two1)
	fmt.Printf("two1 is: %#v\n", two1)
}

func (tn *TwoInts) String() string {
	return "(" + strconv.Itoa(tn.a) + "/" + strconv.Itoa(tn.b) + ")"
}
```

输出：

    two1 is: (12/10)
    two1 is: (12/10)
    two1 is: *main.TwoInts
    two1 is: &main.TwoInts{a:12, b:10}

当你广泛使用一个自定义类型时，最好为它定义 `String()`方法。从上面的例子也可以看到，格式化描述符 `%T` 会给出类型的完全规格，`%#v` 会给出实例的完整输出，包括它的字段（在程序自动生成 `Go` 代码时也很有用）。

**备注**

不要在 `String()` 方法里面调用涉及 `String()` 方法的方法，它会导致意料之外的错误，比如下面的例子，它导致了一个无限递归调用（`TT.String()` 调用 `fmt.Sprintf`，而 `fmt.Sprintf` 又会反过来调用 `TT.String()`...），很快就会导致内存溢出：

```go
type TT float64

func (t TT) String() string {
    return fmt.Sprintf("%v", t)
}
t.String()
```

**练习 10.12** `type_string.go`

给定结构体类型 T:

```go
type T struct {
    a int
    b float32
    c string
}
```

值 `t`: `t := &T{7, -2.35, "abc\tdef"}`。给 T 定义 `String()`，使得 `fmt.Printf("%v\n", t)` 输出：`7 / -2.350000 / "abc\tdef"`。

```go
package main

import "fmt"

type T struct {
	a int
	b float32
	c string
}

func main() {
	t := &T{7, -2.35, "abc\tdef"}
	fmt.Printf("%v\n", t)
}

func (t *T) String() string {
	return fmt.Sprintf("%d / %f / %q", t.a, t.b, t.c)
}

// Output: 7 / -2.350000 / "abc\tdef"
```

**练习 10.13** `celsius.go`

为 `float64` 定义一个别名类型 `Celsius`，并给它定义 `String()`，它输出一个十进制数和 °C 表示的温度值。

```go
// celsius.go
package main

import (
	"fmt"
	"strconv"
)

type Celsius float64

func (c Celsius) String() string {
	return "The temperature is: " + strconv.FormatFloat(float64(c), 'f', 1, 32) + " °C"
}

func main() {
	var c Celsius = 18.36
	fmt.Println(c)
}

// The temperature is: 18.4 °C
```

**练习 10.14** `days.go`

为 `int` 定义一个别名类型 `Day`，定义一个字符串数组它包含一周七天的名字，为类型 `Day` 定义 `String()` 方法，它输出星期几的名字。使用 `iota` 定义一个枚举常量用于表示一周的中每天（MO、TU...）。

```go
package main

import "fmt"

type Day int

const (
	MO Day = iota
	TU
	WE
	TH
	FR
	SA
	SU
)

var dayName = []string{"Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday", "Sunday"}

func (day Day) String() string {
	return dayName[day]
}

func main() {
	var th Day = 3
	fmt.Printf("The 3rd day is: %s\n", th)
	// If index > 6: panic: runtime error: index out of range
	// but use the enumerated type to work with valid values:
	var day = SU
	fmt.Println(day) // prints Sunday
	fmt.Println(0, MO, 1, TU)
}

/* Output:
The 3rd day is: Thursday
Sunday
0 Monday 1 Tuesday
*/
```

**练习 10.15** `timezones.go`

为 `int` 定义别名类型 `TZ`，定义一些常量表示时区，比如 UTC，定义一个 `map`，它将时区的缩写映射为它的全称，比如：`UTC -> "Universal Greenwich time"`。为类型 `TZ` 定义 `String()` 方法，它输出时区的全称。

```go
package main

import "fmt"

type TZ int

const (
	HOUR TZ = 60 * 60
	UTC  TZ = 0 * HOUR
	EST  TZ = -5 * HOUR
	CST  TZ = -6 * HOUR
)

var timeZones = map[TZ]string{
	UTC: "Universal Greenwich time",
	EST: "Eastern Standard time",
	CST: "Central Standard time"}

func (tz TZ) String() string { // Method on TZ (not ptr)
	if zone, ok := timeZones[tz]; ok {
		return zone
	}
	return ""
}

func main() {
	fmt.Println(EST) // Print* knows about method String() of type TZ
	fmt.Println(0 * HOUR)
	fmt.Println(-6 * HOUR)
}

/* Output:
Eastern Standard time
Universal Greenwich time
Central Standard time
*/
```

**练习 10.16** `stack_arr.go/stack_struct.go`

实现栈（`stack`）数据结构：

![](images/10.7_fig.jpg?raw=true)

它的格子包含数据，比如整数 `i`、`j`、`k` 和 `l` 等等，格子从底部（索引 0）至顶部（索引 n）来索引。这个例子中假定 `n=3`，那么一共有 4 个格子。

一个新栈中所有格子的值都是 0。

将一个新值放到栈的最顶部一个空（包括零）的格子中，这叫做`push`。

获取栈的最顶部一个非空（非零）的格子的值，这叫做`pop`。
现在可以理解为什么栈是一个后进先出（`LIFO`）的结构了吧。

为栈定义一个`Stack` 类型，并为它定义 `Push` 和 `Pop` 方法，再为它定义 `String()` 方法（用于调试）输出栈的内容，比如：`[0:i] [1:j] [2:k] [3:l]`。

1）`stack_arr.go`：使用长度为 4 的 `int` 数组作为底层数据结构。

2）`stack_struct.go`：使用包含一个索引和一个 `int` 数组的结构体作为底层数据结构，索引表示第一个空闲的位置。

3）使用常量 `LIMIT` 代替上面表示元素个数的 4 重新实现上面的 1）和 2），使它们更具有一般性。

```go
// stack_arr.go
package main

import (
	"fmt"
	"strconv"
)

const LIMIT = 4

type Stack [LIMIT]int

func main() {
	st1 := new(Stack)
	fmt.Printf("%v\n", st1)
	st1.Push(3)
	fmt.Printf("%v\n", st1)
	st1.Push(7)
	fmt.Printf("%v\n", st1)
	st1.Push(10)
	fmt.Printf("%v\n", st1)
	st1.Push(99)
	fmt.Printf("%v\n", st1)
	p := st1.Pop()
	fmt.Printf("Popped %d\n", p)
	fmt.Printf("%v\n", st1)
	p = st1.Pop()
	fmt.Printf("Popped %d\n", p)
	fmt.Printf("%v\n", st1)
	p = st1.Pop()
	fmt.Printf("Popped %d\n", p)
	fmt.Printf("%v\n", st1)
	p = st1.Pop()
	fmt.Printf("Popped %d\n", p)
	fmt.Printf("%v\n", st1)
}

// put value on first position which contains 0, starting from bottom
func (st *Stack) Push(n int) {
	for ix, v := range st {
		if v == 0 {
			st[ix] = n
			break
		}
	}
}

// take value from first position which contains !=0, starting from top
func (st *Stack) Pop() int {
	v := 0
	for ix := len(st) - 1; ix >= 0; ix-- {
		if v = st[ix]; v != 0 {
			st[ix] = 0
			return v
		}
	}
	return 0
}

func (st Stack) String() string {
	str := ""
	for ix, v := range st {
		str += "[" + strconv.Itoa(ix) + ":" + strconv.Itoa(v) + "] "
	}
	return str
}

```

```go
// stack_struct.go
package main

import (
	"fmt"
	"strconv"
)

const LIMIT = 4

type Stack struct {
	ix   int // first free position, so data[ix] == 0
	data [LIMIT]int
}

func main() {
	st1 := new(Stack)
	fmt.Printf("%v\n", st1)
	st1.Push(3)
	fmt.Printf("%v\n", st1)
	st1.Push(7)
	fmt.Printf("%v\n", st1)
	st1.Push(10)
	fmt.Printf("%v\n", st1)
	st1.Push(99)
	fmt.Printf("%v\n", st1)
	p := st1.Pop()
	fmt.Printf("Popped %d\n", p)
	fmt.Printf("%v\n", st1)
	p = st1.Pop()
	fmt.Printf("Popped %d\n", p)
	fmt.Printf("%v\n", st1)
	p = st1.Pop()
	fmt.Printf("Popped %d\n", p)
	fmt.Printf("%v\n", st1)
	p = st1.Pop()
	fmt.Printf("Popped %d\n", p)
	fmt.Printf("%v\n", st1)
}

func (st *Stack) Push(n int) {
	if st.ix+1 > LIMIT {
		return // stack is full!
	}
	st.data[st.ix] = n
	st.ix++
}

func (st *Stack) Pop() int {
	st.ix--
	return st.data[st.ix]
}

func (st Stack) String() string {
	str := ""
	for ix := 0; ix < st.ix; ix++ {
		str += "[" + strconv.Itoa(ix) + ":" + strconv.Itoa(st.data[ix]) + "] "
	}
	return str
}
```

## 10.8 垃圾回收和 SetFinalizer

Go 开发者不需要写代码来释放程序中不再使用的变量和结构占用的内存，在 Go 运行时中有一个独立的进程，即垃圾收集器（GC），会处理这些事情，它搜索不再使用的变量然后释放它们的内存。可以通过 `runtime` 包访问 GC 进程。

通过调用 `runtime.GC()` 函数可以显式的触发 GC，但这只在某些罕见的场景下才有用，比如当内存资源不足时调用 `runtime.GC()`，它会在此函数执行的点上立即释放一大片内存，此时程序可能会有短时的性能下降（因为 `GC` 进程在执行）。

如果想知道当前的内存状态，可以使用：

```go
// fmt.Printf("%d\n", runtime.MemStats.Alloc/1024)
// 此处代码在 Go 1.5.1下不再有效，更正为
var m runtime.MemStats
runtime.ReadMemStats(&m)
fmt.Printf("%d Kb\n", m.Alloc / 1024)
```

上面的程序会给出已分配内存的总量，单位是 `Kb`。进一步的测量参考 [文档页面](http://golang.org/pkg/runtime/#MemStatsType)。

如果需要在一个对象 `obj` 被从内存移除前执行一些特殊操作，比如写到日志文件中，可以通过如下方式调用函数来实现：

```go
runtime.SetFinalizer(obj, func(obj *typeObj))
```

`func(obj *typeObj)` 需要一个 `typeObj` 类型的指针参数 `obj`，特殊操作会在它上面执行。`func` 也可以是一个匿名函数。

在对象被 GC 进程选中并从内存中移除以前，`SetFinalizer` 都不会执行，即使程序正常结束或者发生错误。

**练习 10.17**

从练习 10.16 开始（它基于结构体实现了一个栈结构），为栈的实现（`stack_struct.go`）创建一个单独的包 `stack`，并从 `main` 包 `main.stack.go` 中调用它。

```go
// Q15.go
package main

import (
	"./stack/stack"
	"fmt"
)

func main() {
	st1 := new(stack.Stack)
	fmt.Printf("%v\n", st1)
	st1.Push(3)
	fmt.Printf("%v\n", st1)
	st1.Push(7)
	fmt.Printf("%v\n", st1)
	st1.Push(10)
	fmt.Printf("%v\n", st1)
	st1.Push(99)
	fmt.Printf("%v\n", st1)
	p := st1.Pop()
	fmt.Printf("Popped %d\n", p)
	fmt.Printf("%v\n", st1)
	p = st1.Pop()
	fmt.Printf("Popped %d\n", p)
	fmt.Printf("%v\n", st1)
	p = st1.Pop()
	fmt.Printf("Popped %d\n", p)
	fmt.Printf("%v\n", st1)
	p = st1.Pop()
	fmt.Printf("Popped %d\n", p)
	fmt.Printf("%v\n", st1)
}

/* Output:
[0:3]
[0:3] [1:7]
[0:3] [1:7] [2:10]
[0:3] [1:7] [2:10] [3:99]
Popped 99
[0:3] [1:7] [2:10]
Popped 10
[0:3] [1:7]
Popped 7
[0:3]
Popped 3
*/
```

