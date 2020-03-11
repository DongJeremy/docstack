# 第六章　方法

从90年代早期开始，面向对象编程（OOP）就成为了称霸工程界和教育界的编程范式，所以之后几乎所有大规模被应用的语言都包含了对OOP的支持，go语言也不例外。

尽管没有被大众所接受的明确的OOP的定义，从我们的理解来讲，一个对象其实也就是一个简单的值或者一个变量，在这个对象中会包含一些方法，而一个方法则是一个一个和特殊类型关联的函数。一个面向对象的程序会用方法来表达其属性和对应的操作，这样使用这个对象的用户就不需要直接去操作对象，而是借助方法来做这些事情。

在早些的章节中，我们已经使用了标准库提供的一些方法，比如`time.Duration`这个类型的`Seconds`方法：

```go
const day = 24 * time.Hour
fmt.Println(day.Seconds()) // "86400"
```

并且在2.5节中，我们定义了一个自己的方法，`Celsius`类型的`String`方法:

```go
func (c Celsius) String() string { return fmt.Sprintf("%g°C", c) }
```

在本章中，OOP编程的第一方面，我们会向你展示如何有效地定义和使用方法。我们会覆盖到OOP编程的两个关键点，封装和组合。

## 6.1. 方法声明

在函数声明时，在其名字之前放上一个变量，即是一个方法。这个附加的参数会将该函数附加到这种类型上，即相当于为这种类型定义了一个独占的方法。

下面来写我们第一个方法的例子，这个例子在`package geometry`下：

*gopl.io/ch6/geometry*

```go
package geometry

import "math"

type Point struct{ X, Y float64 }

// traditional function
func Distance(p, q Point) float64 {
	return math.Hypot(q.X-p.X, q.Y-p.Y)
}

// same thing, but as a method of the Point type
func (p Point) Distance(q Point) float64 {
	return math.Hypot(q.X-p.X, q.Y-p.Y)
}
```

上面的代码里那个附加的参数`p`，叫做方法的接收器（`receiver`），早期的面向对象语言留下的遗产将调用一个方法称为“*向一个对象发送消息*”。

在Go语言中，我们并不会像其它语言那样用`this`或者`self`作为接收器；我们可以任意的选择接收器的名字。由于接收器的名字经常会被使用到，所以保持其在方法间传递时的一致性和简短性是不错的主意。这里的建议是可以使用其类型的第一个字母，比如这里使用了`Point`的首字母`p`。

在方法调用过程中，接收器参数一般会在方法名之前出现。这和方法声明是一样的，都是接收器参数在方法名字之前。下面是例子：

```go
p := Point{1, 2}
q := Point{4, 6}
fmt.Println(Distance(p, q)) // "5", function call
fmt.Println(p.Distance(q))  // "5", method call
```

可以看到，上面的两个函数调用都是`Distance`，但是却没有发生冲突。第一个`Distance`的调用实际上用的是包级别的函数`geometry.Distance`，而第二个则是使用刚刚声明的`Point`，调用的是`Point`类下声明的`Point.Distance`方法。

这种`p.Distance`的表达式叫做**选择器**，因为他会选择合适的对应`p`这个对象的`Distance`方法来执行。选择器也会被用来选择一个`struct`类型的字段，比如`p.X`。由于方法和字段都是在同一命名空间，所以如果我们在这里声明一个`X`方法的话，编译器会报错，因为在调用`p.X`时会有歧义（译注：这里确实挺奇怪的）。

因为每种类型都有其方法的命名空间，我们在用`Distance`这个名字的时候，不同的`Distance`调用指向了不同类型里的`Distance`方法。让我们来定义一个`Path`类型，这个`Path`代表一个线段的集合，并且也给这个`Path`定义一个叫`Distance`的方法。

```go
// A Path is a journey connecting the points with straight lines.
type Path []Point
// Distance returns the distance traveled along the path.
func (path Path) Distance() float64 {
	sum := 0.0
	for i := range path {
		if i > 0 {
			sum += path[i-1].Distance(path[i])
		}
	}
	return sum
}
```

Path是一个命名的`slice`类型，而不是Point那样的`struct`类型，然而我们依然可以为它定义方法。在能够给任意类型定义方法这一点上，Go和很多其它的面向对象的语言不太一样。因此在Go语言里，我们为一些简单的数值、字符串、slice、map来定义一些附加行为很方便。我们可以给同一个包内的任意命名类型定义方法，只要这个命名类型的底层类型（译注：这个例子里，底层类型是指`[]Point`这个slice，Path就是命名类型）不是指针或者`interface`。

两个`Distance`方法有不同的类型。他们两个方法之间没有任何关系，尽管Path的`Distance`方法会在内部调用`Point.Distance`方法来计算每个连接邻接点的线段的长度。

让我们来调用一个新方法，计算三角形的周长：

```go
perim := Path{
	{1, 1},
	{5, 1},
	{5, 4},
	{1, 1},
}
fmt.Println(perim.Distance()) // "12"
```

在上面两个对Distance名字的方法的调用中，编译器会根据方法的名字以及接收器来决定具体调用的是哪一个函数。第一个例子中`path[i-1]`数组中的类型是`Point`，因此`Point.Distance`这个方法被调用；在第二个例子中`perim`的类型是`Path`，因此Distance调用的是`Path.Distance`。

对于一个给定的类型，其内部的方法都必须有唯一的方法名，但是不同的类型却可以有同样的方法名，比如我们这里`Point`和`Path`就都有`Distance`这个名字的方法；所以我们没有必要非在方法名之前加类型名来消除歧义，比如`PathDistance`。这里我们已经看到了方法比之函数的一些好处：方法名可以简短。当我们在包外调用的时候这种好处就会被放大，因为我们可以使用这个短名字，而可以省略掉包的名字，下面是例子：

```go
import "gopl.io/ch6/geometry"

perim := geometry.Path{{1, 1}, {5, 1}, {5, 4}, {1, 1}}
fmt.Println(geometry.PathDistance(perim)) // "12", standalone function
fmt.Println(perim.Distance())             // "12", method of geometry.Path
```

> **译注：** 如果我们要用方法去计算`perim`的`distance`，还需要去写全`geometry`的包名，和其函数名，但是因为`Path`这个类型定义了一个可以直接用的`Distance`方法，所以我们可以直接写`perim.Distance()`。相当于可以少打很多字，作者应该是这个意思。因为在Go里包外调用函数需要带上包名，还是挺麻烦的。

## 6.2. 基于指针对象的方法

当调用一个函数时，会对其每一个参数值进行拷贝，如果一个函数需要更新一个变量，或者函数的其中一个参数实在太大我们希望能够避免进行这种默认的拷贝，这种情况下我们就需要用到指针了。对应到我们这里用来更新接收器的对象的方法，当这个接受者变量本身比较大时，我们就可以用其指针而不是对象来声明方法，如下：

```go
func (p *Point) ScaleBy(factor float64) {
	p.X *= factor
	p.Y *= factor
}
```

这个方法的名字是`(*Point).ScaleBy`。这里的括号是必须的；没有括号的话这个表达式可能会被理解为`*(Point.ScaleBy)`。

在现实的程序里，一般会约定如果Point这个类有一个指针作为接收器的方法，那么所有Point的方法都必须有一个指针接收器，即使是那些并不需要这个指针接收器的函数。我们在这里打破了这个约定只是为了展示一下两种方法的异同而已。

只有类型（`Point`）和指向他们的指针(`*Point`)，才可能是出现在接收器声明里的两种接收器。此外，为了避免歧义，在声明方法时，如果一个类型名本身是一个指针的话，是不允许其出现在接收器中的，比如下面这个例子：

```go
type P *int
func (P) f() { /* ... */ } // compile error: invalid receiver type
```

想要调用指针类型方法`(*Point).ScaleBy`，只要提供一个`Point`类型的指针即可，像下面这样。

```go
r := &Point{1, 2}
r.ScaleBy(2)
fmt.Println(*r) // "{2, 4}"
```

或者这样：

```go
p := Point{1, 2}
pptr := &p
pptr.ScaleBy(2)
fmt.Println(p) // "{2, 4}"
```

或者这样:

```go
p := Point{1, 2}
(&p).ScaleBy(2)
fmt.Println(p) // "{2, 4}"
```

不过后面两种方法有些笨拙。幸运的是，go语言本身在这种地方会帮到我们。如果接收器`p`是一个`Point`类型的变量，并且其方法需要一个`Point`指针作为接收器，我们可以用下面这种简短的写法：

```go
p.ScaleBy(2)
```

编译器会隐式地帮我们用`&p`去调用`ScaleBy`这个方法。这种简写方法只适用于“变量”，包括`struct`里的字段比如`p.X`，以及array和slice内的元素比如`perim[0]`。我们不能通过一个无法取到地址的接收器来调用指针方法，比如临时变量的内存地址就无法获取得到：

```go
Point{1, 2}.ScaleBy(2) // compile error: can't take address of Point literal
```

但是我们可以用一个`*Point`这样的接收器来调用`Point`的方法，因为我们可以通过地址来找到这个变量，只要用解引用符号`*`来取到该变量即可。编译器在这里也会给我们隐式地插入`*`这个操作符，所以下面这两种写法等价的：

```go
pptr.Distance(q)
(*pptr).Distance(q)
```

这里的几个例子可能让你有些困惑，所以我们总结一下：在每一个合法的方法调用表达式中，也就是下面三种情况里的任意一种情况都是可以的：

要么接收器的实际参数和其形式参数是相同的类型，比如两者都是类型T或者都是类型`*T`：

```go
Point{1, 2}.Distance(q) //  Point
pptr.ScaleBy(2)         // *Point
```

或者接收器实参是类型`T`，但接收器形参是类型`*T`，这种情况下编译器会隐式地为我们取变量的地址：

```go
p.ScaleBy(2) // implicit (&p)
```

或者接收器实参是类型`*T`，形参是类型`T`。编译器会隐式地为我们解引用，取到指针指向的实际变量：

```go
pptr.Distance(q) // implicit (*pptr)
```

如果命名类型`T`（译注：用`type xxx`定义的类型）的所有方法都是用`T`类型自己来做接收器（而不是`*T`），那么拷贝这种类型的实例就是安全的；调用他的任何一个方法也就会产生一个值的拷贝。比如`time.Duration`的这个类型，在调用其方法时就会被全部拷贝一份，包括在作为参数传入函数的时候。但是如果一个方法使用指针作为接收器，你需要避免对其进行拷贝，因为这样可能会破坏掉该类型内部的不变性。比如你对`bytes.Buffer`对象进行了拷贝，那么可能会引起原始对象和拷贝对象只是别名而已，实际上它们指向的对象是一样的。紧接着对拷贝后的变量进行修改可能会有让你有意外的结果。

> **译注：** 作者这里说的比较绕，其实有两点：
>
> 1. 不管你的`method`的`receiver`是指针类型还是非指针类型，都是可以通过指针/非指针类型进行调用的，编译器会帮你做类型转换。
> 2. 在声明一个`method`的`receiver`该是指针还是非指针类型时，你需要考虑两方面的因素，第一方面是这个对象本身是不是特别大，如果声明为非指针变量时，调用会产生一次拷贝；第二方面是如果你用指针类型作为`receiver`，那么你一定要注意，这种指针类型指向的始终是一块内存地址，就算你对其进行了拷贝。熟悉C或者C++的人这里应该很快能明白。

### 6.2.1. Nil也是一个合法的接收器类型

就像一些函数允许`nil`指针作为参数一样，方法理论上也可以用`nil`指针作为其接收器，尤其当`nil`对于对象来说是合法的零值时，比如`map`或者`slice`。在下面的简单`int`链表的例子里，`nil`代表的是空链表：

```go
// An IntList is a linked list of integers.
// A nil *IntList represents the empty list.
type IntList struct {
	Value int
	Tail  *IntList
}
// Sum returns the sum of the list elements.
func (list *IntList) Sum() int {
	if list == nil {
		return 0
	}
	return list.Value + list.Tail.Sum()
}
```

当你定义一个允许`nil`作为接收器值的方法的类型时，在类型前面的注释中指出`nil`变量代表的意义是很有必要的，就像我们上面例子里做的这样。

下面是`net/url`包里Values类型定义的一部分。

*net/url*

```GO
package url

// Values maps a string key to a list of values.
type Values map[string][]string
// Get returns the first value associated with the given key,
// or "" if there are none.
func (v Values) Get(key string) string {
	if vs := v[key]; len(vs) > 0 {
		return vs[0]
	}
	return ""
}
// Add adds the value to key.
// It appends to any existing values associated with key.
func (v Values) Add(key, value string) {
	v[key] = append(v[key], value)
}
```

这个定义向外部暴露了一个`map`的命名类型，并且提供了一些能够简单操作这个`map`的方法。这个`map`的`value`字段是一个`string`的`slice`，所以这个`Values`是一个多维`map`。客户端使用这个变量的时候可以使用`map`固有的一些操作（make，切片，m[key]等等），也可以使用这里提供的操作方法，或者两者并用，都是可以的：

*gopl.io/ch6/urlvalues*

```go
m := url.Values{"lang": {"en"}} // direct construction
m.Add("item", "1")
m.Add("item", "2")

fmt.Println(m.Get("lang")) // "en"
fmt.Println(m.Get("q"))    // ""
fmt.Println(m.Get("item")) // "1"      (first value)
fmt.Println(m["item"])     // "[1 2]"  (direct map access)

m = nil
fmt.Println(m.Get("item")) // ""
m.Add("item", "3")         // panic: assignment to entry in nil map
```

对Get的最后一次调用中，`nil`接收器的行为即是一个空map的行为。我们可以等价地将这个操作写成`Value(nil).Get("item")`，但是如果你直接写`nil.Get("item")`的话是无法通过编译的，因为`nil`的字面量编译器无法判断其准确类型。所以相比之下，最后的那行`m.Add`的调用就会产生一个`panic`，因为他尝试更新一个空`map`。

由于`url.Values`是一个`map`类型，并且间接引用了其`key/value`对，因此`url.Values.Add`对这个`map`里的元素做任何的更新、删除操作对调用方都是可见的。实际上，就像在普通函数中一样，虽然可以通过引用来操作内部值，但在方法想要修改引用本身时是不会影响原始值的，比如把他置换为nil，或者让这个引用指向了其它的对象，调用方都不会受影响。（译注：因为传入的是存储了内存地址的变量，你改变这个变量本身是影响不了原始的变量的，想想C语言，是差不多的）

## 6.3. 通过嵌入结构体来扩展类型

来看看`ColoredPoint`这个类型：

*gopl.io/ch6/coloredpoint*

```go
import "image/color"

type Point struct{ X, Y float64 }

type ColoredPoint struct {
	Point
	Color color.RGBA
}
```

我们完全可以将`ColoredPoint`定义为一个有三个字段的`struct`，但是我们却将`Point`这个类型嵌入到`ColoredPoint`来提供`X`和`Y`这两个字段。像我们在4.4节中看到的那样，内嵌可以使我们在定义`ColoredPoint`时得到一种句法上的简写形式，并使其包含`Point`类型所具有的一切字段，然后再定义一些自己的。如果我们想要的话，我们可以直接认为通过嵌入的字段就是`ColoredPoint`自身的字段，而完全不需要在调用时指出`Point`，比如下面这样。

```go
var cp ColoredPoint
cp.X = 1
fmt.Println(cp.Point.X) // "1"
cp.Point.Y = 2
fmt.Println(cp.Y) // "2"
```

对于`Point`中的方法我们也有类似的用法，我们可以把`ColoredPoint`类型当作接收器来调用`Point`里的方法，即使`ColoredPoint`里没有声明这些方法：

```go
red := color.RGBA{255, 0, 0, 255}
blue := color.RGBA{0, 0, 255, 255}
var p = ColoredPoint{Point{1, 1}, red}
var q = ColoredPoint{Point{5, 4}, blue}
fmt.Println(p.Distance(q.Point)) // "5"
p.ScaleBy(2)
q.ScaleBy(2)
fmt.Println(p.Distance(q.Point)) // "10"
```

`Point`类的方法也被引入了`ColoredPoint`。用这种方式，内嵌可以使我们定义字段特别多的复杂类型，我们可以将字段先按小类型分组，然后定义小类型的方法，之后再把它们组合起来。

读者如果对基于类来实现面向对象的语言比较熟悉的话，可能会倾向于将`Point`看作一个基类，而`ColoredPoint`看作其子类或者继承类，或者将`ColoredPoint`看作`"is a" Point`类型。但这是错误的理解。请注意上面例子中对`Distance`方法的调用。`Distance`有一个参数是`Point`类型，但`q`并不是一个`Point`类，所以尽管`q`有着`Point`这个内嵌类型，我们也必须要显式地选择它。尝试直接传`q`的话你会看到下面这样的错误：

```go
p.Distance(q) // compile error: cannot use q (ColoredPoint) as Point
```

一个`ColoredPoint`并不是一个`Point`，但他`"has a"Point`，并且它有从`Point`类里引入的`Distance`和`ScaleBy`方法。如果你喜欢从实现的角度来考虑问题，内嵌字段会指导编译器去生成额外的包装方法来委托已经声明好的方法，和下面的形式是等价的：

```go
func (p ColoredPoint) Distance(q Point) float64 {
	return p.Point.Distance(q)
}

func (p *ColoredPoint) ScaleBy(factor float64) {
	p.Point.ScaleBy(factor)
}
```

当`Point.Distance`被第一个包装方法调用时，它的接收器值是`p.Point`，而不是`p`，当然了，在Point类的方法里，你是访问不到`ColoredPoint`的任何字段的。

在类型中内嵌的匿名字段也可能是一个命名类型的指针，这种情况下字段和方法会被间接地引入到当前的类型中（译注：访问需要通过该指针指向的对象去取）。添加这一层间接关系让我们可以共享通用的结构并动态地改变对象之间的关系。下面这个`ColoredPoint`的声明内嵌了一个`*Point`的指针。

```go
type ColoredPoint struct {
	*Point
	Color color.RGBA
}

p := ColoredPoint{&Point{1, 1}, red}
q := ColoredPoint{&Point{5, 4}, blue}
fmt.Println(p.Distance(*q.Point)) // "5"
q.Point = p.Point                 // p and q now share the same Point
p.ScaleBy(2)
fmt.Println(*p.Point, *q.Point) // "{2 2} {2 2}"
```

一个`struct`类型也可能会有多个匿名字段。我们将`ColoredPoint`定义为下面这样：

```go
type ColoredPoint struct {
	Point
	color.RGBA
}
```

然后这种类型的值便会拥有`Point`和`RGBA`类型的所有方法，以及直接定义在`ColoredPoint`中的方法。当编译器解析一个选择器到方法时，比如`p.ScaleBy`，它会首先去找直接定义在这个类型里的`ScaleBy`方法，然后找被`ColoredPoint`的内嵌字段们引入的方法，然后去找`Point`和`RGBA`的内嵌字段引入的方法，然后一直递归向下找。如果选择器有二义性的话编译器会报错，比如你在同一级里有两个同名的方法。

方法只能在命名类型（像`Point`）或者指向类型的指针上定义，但是多亏了内嵌，有些时候我们给匿名`struct`类型来定义方法也有了手段。

下面是一个小trick。这个例子展示了简单的`cache`，其使用两个包级别的变量来实现，一个`mutex`互斥量（§9.2）和它所操作的`cache`：

```go
var (
	mu sync.Mutex // guards mapping
	mapping = make(map[string]string)
)

func Lookup(key string) string {
	mu.Lock()
	v := mapping[key]
	mu.Unlock()
	return v
}
```

下面这个版本在功能上是一致的，但将两个包级别的变量放在了`cache`这个`struct`一组内：

```go
var cache = struct {
	sync.Mutex
	mapping map[string]string
}{
	mapping: make(map[string]string),
}


func Lookup(key string) string {
	cache.Lock()
	v := cache.mapping[key]
	cache.Unlock()
	return v
}
```

我们给新的变量起了一个更具表达性的名字：`cache`。因为`sync.Mutex`字段也被嵌入到了这个`struct`里，其`Lock`和`Unlock`方法也就都被引入到了这个匿名结构中了，这让我们能够以一个简单明了的语法来对其进行加锁解锁操作。

## 6.4. 方法值和方法表达式

我们经常选择一个方法，并且在同一个表达式里执行，比如常见的`p.Distance()`形式，实际上将其分成两步来执行也是可能的。`p.Distance`叫作“选择器”，选择器会返回一个方法“值”->一个将方法（`Point.Distance`）绑定到特定接收器变量的函数。这个函数可以不通过指定其接收器即可被调用；即调用时不需要指定接收器（译注：因为已经在前文中指定过了），只要传入函数的参数即可：

```go
p := Point{1, 2}
q := Point{4, 6}

distanceFromP := p.Distance        // method value
fmt.Println(distanceFromP(q))      // "5"
var origin Point                   // {0, 0}
fmt.Println(distanceFromP(origin)) // "2.23606797749979", sqrt(5)

scaleP := p.ScaleBy // method value
scaleP(2)           // p becomes (2, 4)
scaleP(3)           //      then (6, 12)
scaleP(10)          //      then (60, 120)
```

在一个包的API需要一个函数值、且调用方希望操作的是某一个绑定了对象的方法的话，方法“值”会非常实用（`=_=`真是绕）。举例来说，下面例子中的`time.AfterFunc`这个函数的功能是在指定的延迟时间之后来执行一个（译注：另外的）函数。且这个函数操作的是一个`Rocket`对象`r`

```go
type Rocket struct { /* ... */ }
func (r *Rocket) Launch() { /* ... */ }
r := new(Rocket)
time.AfterFunc(10 * time.Second, func() { r.Launch() })
```

直接用方法“值”传入`AfterFunc`的话可以更为简短：

```go
time.AfterFunc(10 * time.Second, r.Launch)
```

> 译注：省掉了上面那个例子里的匿名函数。

和方法“值”相关的还有方法表达式。当调用一个方法时，与调用一个普通的函数相比，我们必须要用选择器（`p.Distance`）语法来指定方法的接收器。

当T是一个类型时，方法表达式可能会写作`T.f`或者`(*T).f`，会返回一个函数“值”，这种函数会将其第一个参数用作接收器，所以可以用通常（译注：不写选择器）的方式来对其进行调用：

```go
p := Point{1, 2}
q := Point{4, 6}

distance := Point.Distance   // method expression
fmt.Println(distance(p, q))  // "5"
fmt.Printf("%T\n", distance) // "func(Point, Point) float64"

scale := (*Point).ScaleBy
scale(&p, 2)
fmt.Println(p)            // "{2 4}"
fmt.Printf("%T\n", scale) // "func(*Point, float64)"

// 译注：这个Distance实际上是指定了Point对象为接收器的一个方法func (p Point) Distance()，
// 但通过Point.Distance得到的函数需要比实际的Distance方法多一个参数，
// 即其需要用第一个额外参数指定接收器，后面排列Distance方法的参数。
// 看起来本书中函数和方法的区别是指有没有接收器，而不像其他语言那样是指有没有返回值。
```

当你根据一个变量来决定调用同一个类型的哪个函数时，方法表达式就显得很有用了。你可以根据选择来调用接收器各不相同的方法。下面的例子，变量`op`代表`Point`类型的`addition`或者`subtraction`方法，`Path.TranslateBy`方法会为其`Path`数组中的每一个`Point`来调用对应的方法：

```go
type Point struct{ X, Y float64 }

func (p Point) Add(q Point) Point { return Point{p.X + q.X, p.Y + q.Y} }
func (p Point) Sub(q Point) Point { return Point{p.X - q.X, p.Y - q.Y} }

type Path []Point

func (path Path) TranslateBy(offset Point, add bool) {
	var op func(p, q Point) Point
	if add {
		op = Point.Add
	} else {
		op = Point.Sub
	}
	for i := range path {
		// Call either path[i].Add(offset) or path[i].Sub(offset).
		path[i] = op(path[i], offset)
	}
}
```

## 6.5. 示例: Bit数组

Go语言里的集合一般会用`map[T]bool`这种形式来表示，`T`代表元素类型。集合用`map`类型来表示虽然非常灵活，但我们可以以一种更好的形式来表示它。例如在数据流分析领域，集合元素通常是一个非负整数，集合会包含很多元素，并且集合会经常进行并集、交集操作，这种情况下，bit数组会比map表现更加理想。（译注：这里再补充一个例子，比如我们执行一个`http`下载任务，把文件按照`16kb`一块划分为很多块，需要有一个全局变量来标识哪些块下载完成了，这种时候也需要用到bit数组。）

一个bit数组通常会用一个无符号数或者称之为“字”的slice来表示，每一个元素的每一位都表示集合里的一个值。当集合的第`i`位被设置时，我们才说这个集合包含元素`i`。下面的这个程序展示了一个简单的bit数组类型，并且实现了三个函数来对这个bit数组来进行操作：

*gopl.io/ch6/intset*

```go
// An IntSet is a set of small non-negative integers.
// Its zero value represents the empty set.
type IntSet struct {
	words []uint64
}

// Has reports whether the set contains the non-negative value x.
func (s *IntSet) Has(x int) bool {
	word, bit := x/64, uint(x%64)
	return word < len(s.words) && s.words[word]&(1<<bit) != 0
}

// Add adds the non-negative value x to the set.
func (s *IntSet) Add(x int) {
	word, bit := x/64, uint(x%64)
	for word >= len(s.words) {
		s.words = append(s.words, 0)
	}
	s.words[word] |= 1 << bit
}

// UnionWith sets s to the union of s and t.
func (s *IntSet) UnionWith(t *IntSet) {
	for i, tword := range t.words {
		if i < len(s.words) {
			s.words[i] |= tword
		} else {
			s.words = append(s.words, tword)
		}
	}
}
```

因为每一个字都有64个二进制位，所以为了定位x的bit位，我们用了`x/64`的商作为字的下标，并且用`x%64`得到的值作为这个字内的bit的所在位置。`UnionWith`这个方法里用到了bit位的“或”逻辑操作符号|来一次完成64个元素的或计算。（在练习6.5中我们还会有程序用到这个64位字的例子。）

当前这个实现还缺少了很多必要的特性，我们把其中一些作为练习题列在本小节之后。但是有一个方法如果缺失的话我们的bit数组可能会比较难混：将`IntSet`作为一个字符串来打印。这里我们来实现它，让我们来给上面的例子添加一个`String`方法，类似2.5节中做的那样：

```go
// String returns the set as a string of the form "{1 2 3}".
func (s *IntSet) String() string {
	var buf bytes.Buffer
	buf.WriteByte('{')
	for i, word := range s.words {
		if word == 0 {
			continue
		}
		for j := 0; j < 64; j++ {
			if word&(1<<uint(j)) != 0 {
				if buf.Len() > len("{") {
					buf.WriteByte(' ')
				}
				fmt.Fprintf(&buf, "%d", 64*i+j)
			}
		}
	}
	buf.WriteByte('}')
	return buf.String()
}
```

这里留意一下`String`方法，是不是和3.5.4节中的`intsToString`方法很相似；`bytes.Buffer`在`String`方法里经常这么用。当你为一个复杂的类型定义了一个`String`方法时，`fmt`包就会特殊对待这种类型的值，这样可以让这些类型在打印的时候看起来更加友好，而不是直接打印其原始的值。`fmt`会直接调用用户定义的`String`方法。这种机制依赖于接口和类型断言，在第7章中我们会详细介绍。

现在我们就可以在实战中直接用上面定义好的`IntSet`了：

```go
var x, y IntSet
x.Add(1)
x.Add(144)
x.Add(9)
fmt.Println(x.String()) // "{1 9 144}"

y.Add(9)
y.Add(42)
fmt.Println(y.String()) // "{9 42}"

x.UnionWith(&y)
fmt.Println(x.String()) // "{1 9 42 144}"
fmt.Println(x.Has(9), x.Has(123)) // "true false"
```

这里要注意：我们声明的`String`和`Has`两个方法都是以指针类型`*IntSet`来作为接收器的，但实际上对于这两个类型来说，把接收器声明为指针类型也没什么必要。不过另外两个函数就不是这样了，因为另外两个函数操作的是`s.words`对象，如果你不把接收器声明为指针对象，那么实际操作的是拷贝对象，而不是原来的那个对象。因此，因为我们的String方法定义在`IntSet`指针上，所以当我们的变量是`IntSet`类型而不是`IntSet`指针时，可能会有下面这样让人意外的情况：

```go
fmt.Println(&x)         // "{1 9 42 144}"
fmt.Println(x.String()) // "{1 9 42 144}"
fmt.Println(x)          // "{[4398046511618 0 65536]}"
```

在第一个`Println`中，我们打印一个`*IntSet`的指针，这个类型的指针确实有自定义的`String`方法。第二`Println`，我们直接调用了`x`变量的`String()`方法；这种情况下编译器会隐式地在`x`前插入`&`操作符，这样相当于我们还是调用的`IntSet`指针的`String`方法。在第三个`Println`中，因为`IntSet`类型没有`String`方法，所以`Println`方法会直接以原始的方式理解并打印。所以在这种情况下`&`符号是不能忘的。在我们这种场景下，你把`String`方法绑定到`IntSet`对象上，而不是`IntSet`指针上可能会更合适一些，不过这也需要具体问题具体分析。

**练习6.1:** 为bit数组实现下面这些方法

```go
func (*IntSet) Len() int      // return the number of elements
func (*IntSet) Remove(x int)  // remove x from the set
func (*IntSet) Clear()        // remove all elements from the set
func (*IntSet) Copy() *IntSet // return a copy of the set
```

```go
// intset.go
package main

import (
	"bytes"
	"fmt"
)

// An IntSet is a set of small non-negative integers.
// Its zero value represents the empty set.
type IntSet struct {
	words []uint64
}

// Has reports whether the set contains the non-negative value x.
func (s *IntSet) Has(x int) bool {
	word, bit := x/64, uint(x%64)
	return word < len(s.words) && s.words[word]&(1<<bit) != 0
}

// Add adds the non-negative value x to the set.
func (s *IntSet) Add(x int) {
	word, bit := x/64, uint(x%64)
	for word >= len(s.words) {
		s.words = append(s.words, 0)
	}
	s.words[word] |= 1 << bit
}

// UnionWith sets s to the union of s and t.
func (s *IntSet) UnionWith(t *IntSet) {
	for i, tword := range t.words {
		if i < len(s.words) {
			s.words[i] |= tword
		} else {
			s.words = append(s.words, tword)
		}
	}
}

func popcount(x uint64) int {
	count := 0
	for x != 0 {
		count++
		x &= x - 1
	}
	return count
}

// return the number of elements
func (s *IntSet) Len() int {
	count := 0
	for _, word := range s.words {
		count += popcount(word)
	}
	return count
}

// remove x from the set
func (s *IntSet) Remove(x int) {
	word, bit := x/64, uint(x%64)
	s.words[word] &^= 1 << bit
}

// remove all elements from the set
func (s *IntSet) Clear() {
	for i := range s.words {
		s.words[i] = 0
	}
}

// return a copy of the set
func (s *IntSet) Copy() *IntSet {
	new := &IntSet{}
	new.words = make([]uint64, len(s.words))
	copy(new.words, s.words)
	return new
}

// String returns the set as a string of the form "{1 2 3}".
func (s *IntSet) String() string {
	var buf bytes.Buffer
	buf.WriteByte('{')
	for i, word := range s.words {
		if word == 0 {
			continue
		}
		for j := 0; j < 64; j++ {
			if word&(1<<uint(j)) != 0 {
				if buf.Len() > len("{") {
					buf.WriteByte(' ')
				}
				fmt.Fprintf(&buf, "%d", 64*i+j)
			}
		}
	}
	buf.WriteByte('}')
	return buf.String()
}
```

```go
// intset_test.go
package main

import (
	"testing"
)

func TestLenZeroInitially(t *testing.T) {
	s := &IntSet{}
	if s.Len() != 0 {
		t.Logf("%d != 0", s.Len())
		t.Fail()
	}
}

func TestLenAfterAddingElements(t *testing.T) {
	s := &IntSet{}
	s.Add(0)
	s.Add(2000)
	if s.Len() != 2 {
		t.Logf("%d != 2", s.Len())
		t.Fail()
	}
}

func TestRemove(t *testing.T) {
	s := &IntSet{}
	s.Add(0)
	s.Remove(0)
	if s.Has(0) {
		t.Log(s)
		t.Fail()
	}
}

func TestClear(t *testing.T) {
	s := &IntSet{}
	s.Add(0)
	s.Add(1000)
	s.Clear()
	if s.Has(0) || s.Has(1000) {
		t.Log(s)
		t.Fail()
	}
}

func TestCopy(t *testing.T) {
	orig := &IntSet{}
	orig.Add(1)
	copy := orig.Copy()
	copy.Add(2)
	if !copy.Has(1) || orig.Has(2) {
		t.Log(orig, copy)
		t.Fail()
	}
}
```

**练习 6.2：** 定义一个变参方法`(*IntSet).AddAll(...int)`，这个方法可以添加一组`IntSet`，比如`s.AddAll(1,2,3)`。

```go
// intset.go
package main

import (
	"bytes"
	"fmt"
)

// An IntSet is a set of small non-negative integers.
// Its zero value represents the empty set.
type IntSet struct {
	words []uint64
}

// Has reports whether the set contains the non-negative value x.
func (s *IntSet) Has(x int) bool {
	word, bit := x/64, uint(x%64)
	return word < len(s.words) && s.words[word]&(1<<bit) != 0
}

// Add adds the non-negative value x to the set.
func (s *IntSet) Add(x int) {
	word, bit := x/64, uint(x%64)
	for word >= len(s.words) {
		s.words = append(s.words, 0)
	}
	s.words[word] |= 1 << bit
}

func (s *IntSet) AddAll(nums ...int) {
	for _, n := range nums {
		s.Add(n)
	}
}

// UnionWith sets s to the union of s and t.
func (s *IntSet) UnionWith(t *IntSet) {
	for i, tword := range t.words {
		if i < len(s.words) {
			s.words[i] |= tword
		} else {
			s.words = append(s.words, tword)
		}
	}
}

func popcount(x uint64) int {
	count := 0
	for x != 0 {
		count++
		x &= x - 1
	}
	return count
}

// return the number of elements
func (s *IntSet) Len() int {
	count := 0
	for _, word := range s.words {
		count += popcount(word)
	}
	return count
}

// remove x from the set
func (s *IntSet) Remove(x int) {
	word, bit := x/64, uint(x%64)
	s.words[word] &^= 1 << bit
}

// remove all elements from the set
func (s *IntSet) Clear() {
	for i := range s.words {
		s.words[i] = 0
	}
}

// return a copy of the set
func (s *IntSet) Copy() *IntSet {
	new := &IntSet{}
	new.words = make([]uint64, len(s.words))
	copy(new.words, s.words)
	return new
}

// String returns the set as a string of the form "{1 2 3}".
func (s *IntSet) String() string {
	var buf bytes.Buffer
	buf.WriteByte('{')
	for i, word := range s.words {
		if word == 0 {
			continue
		}
		for j := 0; j < 64; j++ {
			if word&(1<<uint(j)) != 0 {
				if buf.Len() > len("{") {
					buf.WriteByte(' ')
				}
				fmt.Fprintf(&buf, "%d", 64*i+j)
			}
		}
	}
	buf.WriteByte('}')
	return buf.String()
}
```

```go
// intset_test.go
package main

import (
	"testing"
)

func TestLenZeroInitially(t *testing.T) {
	s := &IntSet{}
	if s.Len() != 0 {
		t.Logf("%d != 0", s.Len())
		t.Fail()
	}
}

func TestLenAfterAddingElements(t *testing.T) {
	s := &IntSet{}
	s.Add(0)
	s.Add(2000)
	if s.Len() != 2 {
		t.Logf("%d != 2", s.Len())
		t.Fail()
	}
}

func TestRemove(t *testing.T) {
	s := &IntSet{}
	s.Add(0)
	s.Remove(0)
	if s.Has(0) {
		t.Log(s)
		t.Fail()
	}
}

func TestClear(t *testing.T) {
	s := &IntSet{}
	s.Add(0)
	s.Add(1000)
	s.Clear()
	if s.Has(0) || s.Has(1000) {
		t.Log(s)
		t.Fail()
	}
}

func TestCopy(t *testing.T) {
	orig := &IntSet{}
	orig.Add(1)
	copy := orig.Copy()
	copy.Add(2)
	if !copy.Has(1) || orig.Has(2) {
		t.Log(orig, copy)
		t.Fail()
	}
}

func TestAddAll(t *testing.T) {
	s := &IntSet{}
	s.AddAll(0, 2, 4)
	if !s.Has(0) || !s.Has(2) || !s.Has(4) {
		t.Log(s)
		t.Fail()
	}
}
```

**练习 6.3：** `(*IntSet).UnionWith`会用`|`操作符计算两个集合的并集，我们再为`IntSet`实现另外的几个函数`IntersectWith`（交集：元素在A集合B集合均出现），`DifferenceWith`（差集：元素出现在A集合，未出现在B集合），`SymmetricDifference`（并差集：元素出现在A但没有出现在B，或者出现在B没有出现在A）。

```go
// intset.go
package main

import (
	"bytes"
	"fmt"
)

// An IntSet is a set of small non-negative integers.
// Its zero value represents the empty set.
type IntSet struct {
	words []uint64
}

// Has reports whether the set contains the non-negative value x.
func (s *IntSet) Has(x int) bool {
	word, bit := x/64, uint(x%64)
	return word < len(s.words) && s.words[word]&(1<<bit) != 0
}

// Add adds the non-negative value x to the set.
func (s *IntSet) Add(x int) {
	word, bit := x/64, uint(x%64)
	for word >= len(s.words) {
		s.words = append(s.words, 0)
	}
	s.words[word] |= 1 << bit
}

func (s *IntSet) AddAll(nums ...int) {
	for _, n := range nums {
		s.Add(n)
	}
}

// UnionWith sets s to the union of s and t.
func (s *IntSet) UnionWith(t *IntSet) {
	for i, tword := range t.words {
		if i < len(s.words) {
			s.words[i] |= tword
		} else {
			s.words = append(s.words, tword)
		}
	}
}

// Set s to the intersection of s and t.
func (s *IntSet) IntersectWith(t *IntSet) {
	for i, tword := range t.words {
		if i < len(s.words) {
			s.words[i] &= tword
		} else {
			s.words = append(s.words, tword)
		}
	}
}

// Set s to the difference of s and t.
func (s *IntSet) DifferenceWith(t *IntSet) {
	for i, tword := range t.words {
		if i < len(s.words) {
			s.words[i] &^= tword
		} else {
			s.words = append(s.words, tword)
		}
	}
}

// Set s to the symmetric difference of s and t.
func (s *IntSet) SymmetricDifference(t *IntSet) {
	for i, tword := range t.words {
		if i < len(s.words) {
			s.words[i] ^= tword
		} else {
			s.words = append(s.words, tword)
		}
	}
}

func popcount(x uint64) int {
	count := 0
	for x != 0 {
		count++
		x &= x - 1
	}
	return count
}

// return the number of elements
func (s *IntSet) Len() int {
	count := 0
	for _, word := range s.words {
		count += popcount(word)
	}
	return count
}

// remove x from the set
func (s *IntSet) Remove(x int) {
	word, bit := x/64, uint(x%64)
	s.words[word] &^= 1 << bit
}

// remove all elements from the set
func (s *IntSet) Clear() {
	for i := range s.words {
		s.words[i] = 0
	}
}

// return a copy of the set
func (s *IntSet) Copy() *IntSet {
	new := &IntSet{}
	new.words = make([]uint64, len(s.words))
	copy(new.words, s.words)
	return new
}

// String returns the set as a string of the form "{1 2 3}".
func (s *IntSet) String() string {
	var buf bytes.Buffer
	buf.WriteByte('{')
	for i, word := range s.words {
		if word == 0 {
			continue
		}
		for j := 0; j < 64; j++ {
			if word&(1<<uint(j)) != 0 {
				if buf.Len() > len("{") {
					buf.WriteByte(' ')
				}
				fmt.Fprintf(&buf, "%d", 64*i+j)
			}
		}
	}
	buf.WriteByte('}')
	return buf.String()
}
```

```go
// intset_test.go
package main

import (
	"testing"
)

func TestLenZeroInitially(t *testing.T) {
	s := &IntSet{}
	if s.Len() != 0 {
		t.Logf("%d != 0", s.Len())
		t.Fail()
	}
}

func TestLenAfterAddingElements(t *testing.T) {
	s := &IntSet{}
	s.Add(0)
	s.Add(2000)
	if s.Len() != 2 {
		t.Logf("%d != 2", s.Len())
		t.Fail()
	}
}

func TestRemove(t *testing.T) {
	s := &IntSet{}
	s.Add(0)
	s.Remove(0)
	if s.Has(0) {
		t.Log(s)
		t.Fail()
	}
}

func TestClear(t *testing.T) {
	s := &IntSet{}
	s.Add(0)
	s.Add(1000)
	s.Clear()
	if s.Has(0) || s.Has(1000) {
		t.Log(s)
		t.Fail()
	}
}

func TestCopy(t *testing.T) {
	orig := &IntSet{}
	orig.Add(1)
	copy := orig.Copy()
	copy.Add(2)
	if !copy.Has(1) || orig.Has(2) {
		t.Log(orig, copy)
		t.Fail()
	}
}

func TestAddAll(t *testing.T) {
	s := &IntSet{}
	s.AddAll(0, 2, 4)
	if !s.Has(0) || !s.Has(2) || !s.Has(4) {
		t.Log(s)
		t.Fail()
	}
}

func TestIntersectWith(t *testing.T) {
	s := &IntSet{}
	s.AddAll(0, 2, 4)
	u := &IntSet{}
	u.AddAll(1, 2, 3)
	s.IntersectWith(u)
	if !s.Has(2) || s.Len() != 1 {
		t.Log(s)
		t.Fail()
	}
}

func TestDifferenceWith(t *testing.T) {
	s := &IntSet{}
	s.AddAll(0, 2, 4)
	u := &IntSet{}
	u.AddAll(1, 2, 3)
	s.DifferenceWith(u)
	expected := &IntSet{}
	expected.AddAll(0, 4)
	if s.String() != expected.String() {
		t.Log(s)
		t.Fail()
	}
}

func TestSymmetricDifference(t *testing.T) {
	s := &IntSet{}
	s.AddAll(0, 2, 4)
	u := &IntSet{}
	u.AddAll(1, 2, 3)
	s.SymmetricDifference(u)
	expected := &IntSet{}
	expected.AddAll(0, 1, 3, 4)
	if s.String() != expected.String() {
		t.Log(s)
		t.Fail()
	}
}
```

**练习6.4: ** 实现一个`Elems`方法，返回集合中的所有元素，用于做一些`range`之类的遍历操作。

```go
// intset.go
package main

import (
	"bytes"
	"fmt"
)

// An IntSet is a set of small non-negative integers.
// Its zero value represents the empty set.
type IntSet struct {
	words []uint64
}

// Has reports whether the set contains the non-negative value x.
func (s *IntSet) Has(x int) bool {
	word, bit := x/64, uint(x%64)
	return word < len(s.words) && s.words[word]&(1<<bit) != 0
}

// Add adds the non-negative value x to the set.
func (s *IntSet) Add(x int) {
	word, bit := x/64, uint(x%64)
	for word >= len(s.words) {
		s.words = append(s.words, 0)
	}
	s.words[word] |= 1 << bit
}

func (s *IntSet) AddAll(nums ...int) {
	for _, n := range nums {
		s.Add(n)
	}
}

// UnionWith sets s to the union of s and t.
func (s *IntSet) UnionWith(t *IntSet) {
	for i, tword := range t.words {
		if i < len(s.words) {
			s.words[i] |= tword
		} else {
			s.words = append(s.words, tword)
		}
	}
}

// Set s to the intersection of s and t.
func (s *IntSet) IntersectWith(t *IntSet) {
	for i, tword := range t.words {
		if i < len(s.words) {
			s.words[i] &= tword
		} else {
			s.words = append(s.words, tword)
		}
	}
}

// Set s to the difference of s and t.
func (s *IntSet) DifferenceWith(t *IntSet) {
	for i, tword := range t.words {
		if i < len(s.words) {
			s.words[i] &^= tword
		} else {
			s.words = append(s.words, tword)
		}
	}
}

// Set s to the symmetric difference of s and t.
func (s *IntSet) SymmetricDifference(t *IntSet) {
	for i, tword := range t.words {
		if i < len(s.words) {
			s.words[i] ^= tword
		} else {
			s.words = append(s.words, tword)
		}
	}
}

func popcount(x uint64) int {
	count := 0
	for x != 0 {
		count++
		x &= x - 1
	}
	return count
}

// return the number of elements
func (s *IntSet) Len() int {
	count := 0
	for _, word := range s.words {
		count += popcount(word)
	}
	return count
}

// remove x from the set
func (s *IntSet) Remove(x int) {
	word, bit := x/64, uint(x%64)
	s.words[word] &^= 1 << bit
}

// remove all elements from the set
func (s *IntSet) Clear() {
	for i := range s.words {
		s.words[i] = 0
	}
}

// return a copy of the set
func (s *IntSet) Copy() *IntSet {
	new := &IntSet{}
	new.words = make([]uint64, len(s.words))
	copy(new.words, s.words)
	return new
}

// String returns the set as a string of the form "{1 2 3}".
func (s *IntSet) String() string {
	var buf bytes.Buffer
	buf.WriteByte('{')
	for i, word := range s.words {
		if word == 0 {
			continue
		}
		for j := 0; j < 64; j++ {
			if word&(1<<uint(j)) != 0 {
				if buf.Len() > len("{") {
					buf.WriteByte(' ')
				}
				fmt.Fprintf(&buf, "%d", 64*i+j)
			}
		}
	}
	buf.WriteByte('}')
	return buf.String()
}

// Return set elements.
func (s *IntSet) Elems() []int {
	e := make([]int, 0)
	for i, word := range s.words {
		for j := 0; j < 64; j++ {
			if word&(1<<uint(j)) != 0 {
				e = append(e, i*64+j)
			}
		}
	}
	return e
}
```

```go
// intset_test.go
package main

import (
	"testing"
)

func TestLenZeroInitially(t *testing.T) {
	s := &IntSet{}
	if s.Len() != 0 {
		t.Logf("%d != 0", s.Len())
		t.Fail()
	}
}

func TestLenAfterAddingElements(t *testing.T) {
	s := &IntSet{}
	s.Add(0)
	s.Add(2000)
	if s.Len() != 2 {
		t.Logf("%d != 2", s.Len())
		t.Fail()
	}
}

func TestRemove(t *testing.T) {
	s := &IntSet{}
	s.Add(0)
	s.Remove(0)
	if s.Has(0) {
		t.Log(s)
		t.Fail()
	}
}

func TestClear(t *testing.T) {
	s := &IntSet{}
	s.Add(0)
	s.Add(1000)
	s.Clear()
	if s.Has(0) || s.Has(1000) {
		t.Log(s)
		t.Fail()
	}
}

func TestCopy(t *testing.T) {
	orig := &IntSet{}
	orig.Add(1)
	copy := orig.Copy()
	copy.Add(2)
	if !copy.Has(1) || orig.Has(2) {
		t.Log(orig, copy)
		t.Fail()
	}
}

func TestAddAll(t *testing.T) {
	s := &IntSet{}
	s.AddAll(0, 2, 4)
	if !s.Has(0) || !s.Has(2) || !s.Has(4) {
		t.Log(s)
		t.Fail()
	}
}

func TestIntersectWith(t *testing.T) {
	s := &IntSet{}
	s.AddAll(0, 2, 4)
	u := &IntSet{}
	u.AddAll(1, 2, 3)
	s.IntersectWith(u)
	if !s.Has(2) || s.Len() != 1 {
		t.Log(s)
		t.Fail()
	}
}

func TestDifferenceWith(t *testing.T) {
	s := &IntSet{}
	s.AddAll(0, 2, 4)
	u := &IntSet{}
	u.AddAll(1, 2, 3)
	s.DifferenceWith(u)
	expected := &IntSet{}
	expected.AddAll(0, 4)
	if s.String() != expected.String() {
		t.Log(s)
		t.Fail()
	}
}

func TestSymmetricDifference(t *testing.T) {
	s := &IntSet{}
	s.AddAll(0, 2, 4)
	u := &IntSet{}
	u.AddAll(1, 2, 3)
	s.SymmetricDifference(u)
	expected := &IntSet{}
	expected.AddAll(0, 1, 3, 4)
	if s.String() != expected.String() {
		t.Log(s)
		t.Fail()
	}
}

func TestElems(t *testing.T) {
	elems := []int{0, 4, 8, 9}
	s := &IntSet{}
	s.AddAll(elems...)
	for i, n := range s.Elems() {
		if elems[i] != n {
			t.Log(s.Elems())
			t.Fail()
		}
	}
}
```

**练习 6.5：** 我们这章定义的`IntSet`里的每个字都是用的`uint64`类型，但是64位的数值可能在32位的平台上不高效。修改程序，使其使用`uint`类型，这种类型对于32位平台来说更合适。当然了，这里我们可以不用简单粗暴地除`64`，可以定义一个常量来决定是用32还是64，这里你可能会用到平台的自动判断的一个智能表达式：`32 << (^uint(0) >> 63)`

```go
// intset.go
package main

import (
	"bytes"
	"fmt"
)

const wordSize = 32 << (^uint(0) >> 63)

// An IntSet is a set of small non-negative integers.
// Its zero value represents the empty set.
type IntSet struct {
	words []uint
}

// Has reports whether the set contains the non-negative value x.
func (s *IntSet) Has(x int) bool {
	word, bit := x/wordSize, uint(x%wordSize)
	return word < len(s.words) && s.words[word]&(1<<bit) != 0
}

// Add adds the non-negative value x to the set.
func (s *IntSet) Add(x int) {
	word, bit := x/wordSize, uint(x%wordSize)
	for word >= len(s.words) {
		s.words = append(s.words, 0)
	}
	s.words[word] |= 1 << bit
}

func (s *IntSet) AddAll(nums ...int) {
	for _, n := range nums {
		s.Add(n)
	}
}

// UnionWith sets s to the union of s and t.
func (s *IntSet) UnionWith(t *IntSet) {
	for i, tword := range t.words {
		if i < len(s.words) {
			s.words[i] |= tword
		} else {
			s.words = append(s.words, tword)
		}
	}
}

// Set s to the intersection of s and t.
func (s *IntSet) IntersectWith(t *IntSet) {
	for i, tword := range t.words {
		if i < len(s.words) {
			s.words[i] &= tword
		} else {
			s.words = append(s.words, tword)
		}
	}
}

// Set s to the difference of s and t.
func (s *IntSet) DifferenceWith(t *IntSet) {
	for i, tword := range t.words {
		if i < len(s.words) {
			s.words[i] &^= tword
		} else {
			s.words = append(s.words, tword)
		}
	}
}

// Set s to the symmetric difference of s and t.
func (s *IntSet) SymmetricDifference(t *IntSet) {
	for i, tword := range t.words {
		if i < len(s.words) {
			s.words[i] ^= tword
		} else {
			s.words = append(s.words, tword)
		}
	}
}

func popcount(x uint) int {
	count := 0
	for x != 0 {
		count++
		x &= x - 1
	}
	return count
}

// return the number of elements
func (s *IntSet) Len() int {
	count := 0
	for _, word := range s.words {
		count += popcount(word)
	}
	return count
}

// remove x from the set
func (s *IntSet) Remove(x int) {
	word, bit := x/wordSize, uint(x%wordSize)
	s.words[word] &^= 1 << bit
}

// remove all elements from the set
func (s *IntSet) Clear() {
	for i := range s.words {
		s.words[i] = 0
	}
}

// return a copy of the set
func (s *IntSet) Copy() *IntSet {
	new := &IntSet{}
	new.words = make([]uint, len(s.words))
	copy(new.words, s.words)
	return new
}

// String returns the set as a string of the form "{1 2 3}".
func (s *IntSet) String() string {
	var buf bytes.Buffer
	buf.WriteByte('{')
	for i, word := range s.words {
		if word == 0 {
			continue
		}
		for j := 0; j < wordSize; j++ {
			if word&(1<<uint(j)) != 0 {
				if buf.Len() > len("{") {
					buf.WriteByte(' ')
				}
				fmt.Fprintf(&buf, "%d", wordSize*i+j)
			}
		}
	}
	buf.WriteByte('}')
	return buf.String()
}

// Return set elements.
func (s *IntSet) Elems() []int {
	e := make([]int, 0)
	for i, word := range s.words {
		for j := 0; j < wordSize; j++ {
			if word&(1<<uint(j)) != 0 {
				e = append(e, i*wordSize+j)
			}
		}
	}
	return e
}
```

```go
// intset_test.go
package main

import (
	"testing"
)

func TestLenZeroInitially(t *testing.T) {
	s := &IntSet{}
	if s.Len() != 0 {
		t.Logf("%d != 0", s.Len())
		t.Fail()
	}
}

func TestLenAfterAddingElements(t *testing.T) {
	s := &IntSet{}
	s.Add(0)
	s.Add(2000)
	if s.Len() != 2 {
		t.Logf("%d != 2", s.Len())
		t.Fail()
	}
}

func TestRemove(t *testing.T) {
	s := &IntSet{}
	s.Add(0)
	s.Remove(0)
	if s.Has(0) {
		t.Log(s)
		t.Fail()
	}
}

func TestClear(t *testing.T) {
	s := &IntSet{}
	s.Add(0)
	s.Add(1000)
	s.Clear()
	if s.Has(0) || s.Has(1000) {
		t.Log(s)
		t.Fail()
	}
}

func TestCopy(t *testing.T) {
	orig := &IntSet{}
	orig.Add(1)
	copy := orig.Copy()
	copy.Add(2)
	if !copy.Has(1) || orig.Has(2) {
		t.Log(orig, copy)
		t.Fail()
	}
}

func TestAddAll(t *testing.T) {
	s := &IntSet{}
	s.AddAll(0, 2, 4)
	if !s.Has(0) || !s.Has(2) || !s.Has(4) {
		t.Log(s)
		t.Fail()
	}
}

func TestIntersectWith(t *testing.T) {
	s := &IntSet{}
	s.AddAll(0, 2, 4)
	u := &IntSet{}
	u.AddAll(1, 2, 3)
	s.IntersectWith(u)
	if !s.Has(2) || s.Len() != 1 {
		t.Log(s)
		t.Fail()
	}
}

func TestDifferenceWith(t *testing.T) {
	s := &IntSet{}
	s.AddAll(0, 2, 4)
	u := &IntSet{}
	u.AddAll(1, 2, 3)
	s.DifferenceWith(u)
	expected := &IntSet{}
	expected.AddAll(0, 4)
	if s.String() != expected.String() {
		t.Log(s)
		t.Fail()
	}
}

func TestSymmetricDifference(t *testing.T) {
	s := &IntSet{}
	s.AddAll(0, 2, 4)
	u := &IntSet{}
	u.AddAll(1, 2, 3)
	s.SymmetricDifference(u)
	expected := &IntSet{}
	expected.AddAll(0, 1, 3, 4)
	if s.String() != expected.String() {
		t.Log(s)
		t.Fail()
	}
}

func TestElems(t *testing.T) {
	elems := []int{0, 4, 8, 9}
	s := &IntSet{}
	s.AddAll(elems...)
	for i, n := range s.Elems() {
		if elems[i] != n {
			t.Log(s.Elems())
			t.Fail()
		}
	}
}
```

## 6.6. 封装

一个对象的变量或者方法如果对调用方是不可见的话，一般就被定义为“封装”。封装有时候也被叫做信息隐藏，同时也是面向对象编程最关键的一个方面。

Go语言只有一种控制可见性的手段：大写首字母的标识符会从定义它们的包中被导出，小写字母的则不会。这种限制包内成员的方式同样适用于`struct`或者一个类型的方法。因而如果我们想要封装一个对象，我们必须将其定义为一个`struct`。

这也就是前面的小节中`IntSet`被定义为`struct`类型的原因，尽管它只有一个字段：

```go
type IntSet struct {
    words []uint64
}
```

当然，我们也可以把`IntSet`定义为一个slice类型，但这样我们就需要把代码中所有方法里用到的`s.words`用`*s`替换掉了：

```go
type IntSet []uint64
```

尽管这个版本的`IntSet`在本质上是一样的，但它也允许其它包中可以直接读取并编辑这个slice。换句话说，相对于`*s`这个表达式会出现在所有的包中，`s.words`只需要在定义`IntSet`的包中出现（译注：所以还是推荐后者吧的意思）。

这种基于名字的手段使得在语言中最小的封装单元是package，而不是像其它语言一样的类型。一个struct类型的字段对同一个包的所有代码都有可见性，无论你的代码是写在一个函数还是一个方法里。

封装提供了三方面的优点。首先，因为调用方不能直接修改对象的变量值，其只需要关注少量的语句并且只要弄懂少量变量的可能的值即可。

第二，隐藏实现的细节，可以防止调用方依赖那些可能变化的具体实现，这样使设计包的程序员在不破坏对外的`api`情况下能得到更大的自由。

把`bytes.Buffer`这个类型作为例子来考虑。这个类型在做短字符串叠加的时候很常用，所以在设计的时候可以做一些预先的优化，比如提前预留一部分空间，来避免反复的内存分配。又因为`Buffer`是一个`struct`类型，这些额外的空间可以用附加的字节数组来保存，且放在一个小写字母开头的字段中。这样在外部的调用方只能看到性能的提升，但并不会得到这个附加变量。Buffer和其增长算法我们列在这里，为了简洁性稍微做了一些精简：

```go
type Buffer struct {
    buf     []byte
    initial [64]byte
    /* ... */
}

// Grow expands the buffer's capacity, if necessary,
// to guarantee space for another n bytes. [...]
func (b *Buffer) Grow(n int) {
    if b.buf == nil {
        b.buf = b.initial[:0] // use preallocated space initially
    }
    if len(b.buf)+n > cap(b.buf) {
        buf := make([]byte, b.Len(), 2*cap(b.buf) + n)
        copy(buf, b.buf)
        b.buf = buf
    }
}
```

封装的第三个优点也是最重要的优点，是阻止了外部调用方对对象内部的值任意地进行修改。因为对象内部变量只可以被同一个包内的函数修改，所以包的作者可以让这些函数确保对象内部的一些值的不变性。比如下面的`Counter`类型允许调用方来增加`counter`变量的值，并且允许将这个值`reset`为`0`，但是不允许随便设置这个值（译注：因为压根就访问不到）：

```go
type Counter struct { n int }
func (c *Counter) N() int     { return c.n }
func (c *Counter) Increment() { c.n++ }
func (c *Counter) Reset()     { c.n = 0 }
```

只用来访问或修改内部变量的函数被称为`setter`或者`getter`，例子如下，比如`log`包里的`Logger`类型对应的一些函数。在命名一个`getter`方法时，我们通常会省略掉前面的`Get`前缀。这种简洁上的偏好也可以推广到各种类型的前缀比如`Fetch`，`Find`或者`Lookup`。

```go
package log
type Logger struct {
	flags  int
	prefix string
	// ...
}
func (l *Logger) Flags() int
func (l *Logger) SetFlags(flag int)
func (l *Logger) Prefix() string
func (l *Logger) SetPrefix(prefix string)
```

Go的编码风格不禁止直接导出字段。当然，一旦进行了导出，就没有办法在保证API兼容的情况下去除对其的导出，所以在一开始的选择一定要经过深思熟虑并且要考虑到包内部的一些不变量的保证，未来可能的变化，以及调用方的代码质量是否会因为包的一点修改而变差。

封装并不总是理想的。 虽然封装在有些情况是必要的，但有时候我们也需要暴露一些内部内容，比如：`time.Duration`将其表现暴露为一个`int64`数字的纳秒，使得我们可以用一般的数值操作来对时间进行对比，甚至可以定义这种类型的常量：

```go
const day = 24 * time.Hour
fmt.Println(day.Seconds()) // "86400"
```

另一个例子，将`IntSet`和本章开头的`geometry.Path`进行对比。Path被定义为一个slice类型，这允许其调用slice的字面方法来对其内部的`points`用`range`进行迭代遍历；在这一点上，`IntSet`是没有办法让你这么做的。

这两种类型决定性的不同：`geometry.Path`的本质是一个坐标点的序列，不多也不少，我们可以预见到之后也并不会给他增加额外的字段，所以在`geometry`包中将`Path`暴露为一个`slice`。相比之下，`IntSet`仅仅是在这里用了一个`[]uint64`的`slice`。这个类型还可以用`[]uint`类型来表示，或者我们甚至可以用其它完全不同的占用更小内存空间的东西来表示这个集合，所以我们可能还会需要额外的字段来在这个类型中记录元素的个数。也正是因为这些原因，我们让`IntSet`对调用方不透明。

在这章中，我们学到了如何将方法与命名类型进行组合，并且知道了如何调用这些方法。尽管方法对于OOP编程来说至关重要，但他们只是OOP编程里的半边天。为了完成OOP，我们还需要接口。Go里的接口会在下一章中介绍。
