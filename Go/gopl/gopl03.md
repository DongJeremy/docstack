# 第三章　基础数据类型

虽然从底层而言，所有的数据都是由比特组成，但计算机一般操作的是固定大小的数，如整数、浮点数、比特数组、内存地址等。进一步将这些数组织在一起，就可表达更多的对象，例如数据包、像素点、诗歌，甚至其他任何对象。Go语言提供了丰富的数据组织形式，这依赖于Go语言内置的数据类型。这些内置的数据类型，兼顾了硬件的特性和表达复杂数据结构的便捷性。

Go语言将数据类型分为四类：**基础类型**、**复合类型**、**引用类型**和**接口类型**。本章介绍基础类型，包括：数字、字符串和布尔型。复合数据类型——数组（§4.1）和结构体（§4.2）——是通过组合简单类型，来表达更加复杂的数据结构。引用类型包括**指针**（§2.3.2）、**切片**（§4.2)）、**字典**（§4.3）、**函数**（§5）、**通道**（§8），虽然数据种类很多，但它们都是对程序中一个变量或状态的间接引用。这意味着对任一引用类型数据的修改都会影响所有该引用的拷贝。我们将在第7章介绍接口类型。

## 3.1. 整型

Go语言的数值类型包括几种不同大小的整数、浮点数和复数。每种数值类型都决定了对应的大小范围和是否支持正负符号。让我们先从整数类型开始介绍。

Go语言同时提供了*有符号*和*无符号*类型的整数运算。这里有`int8`、`int16`、`int32`和`int64`四种截然不同大小的有符号整数类型，分别对应8、16、32、64bit大小的有符号整数，与此对应的是`uint8`、`uint16`、`uint32`和`uint64`四种无符号整数类型。

这里还有两种一般对应特定CPU平台机器字大小的有符号和无符号整数`int`和`uint`；其中`int`是应用最广泛的数值类型。这两种类型都有同样的大小，32或64bit，但是我们不能对此做任何的假设；因为不同的编译器即使在相同的硬件平台上可能产生不同的大小。

Unicode字符`rune`类型是和`int32`等价的类型，通常用于表示一个Unicode码点。这两个名称可以互换使用。同样`byte`也是`uint8`类型的等价类型，`byte`类型一般用于强调数值是一个原始的数据而不是一个小的整数。

最后，还有一种无符号的整数类型`uintptr`，没有指定具体的bit大小但是足以容纳指针。`uintptr`类型只有在底层编程时才需要，特别是Go语言和C语言函数库或操作系统接口相交互的地方。我们将在第十三章的`unsafe`包相关部分看到类似的例子。

不管它们的具体大小，`int`、`uint`和`uintptr`是不同类型的兄弟类型。其中`int`和`int32`也是不同的类型，即使`int`的大小也是`32bit`，在需要将`int`当作`int32`类型的地方需要一个显式的类型转换操作，反之亦然。

其中有符号整数采用2的补码形式表示，也就是最高`bit`位用来表示符号位，一个n-bit的有符号数的值域是从$-2^{n-1}$到 $2^{n-1}-1$。无符号整数的所有bit位都用于表示非负数，值域是0到 $2^n-1$。例如，`int8`类型整数的值域是从-128到127，而`uint8`类型整数的值域是从0到255。


下面是Go语言中关于算术运算、逻辑运算和比较运算的二元运算符，它们按照优先级递减的顺序排列：

```go
*      /      %      <<       >>     &       &^
+      -      |      ^
==     !=     <      <=       >      >=
&&
||
```

二元运算符有五种优先级。在同一个优先级，使用左优先结合规则，但是使用括号可以明确优先顺序，使用括号也可以用于提升优先级，例如`mask & (1 << 28)`。

对于上表中前两行的运算符，例如`+`运算符还有一个与赋值相结合的对应运算符`+=`，可以用于简化赋值语句。

算术运算符`+`、`-`、`*`和`/`可以适用于整数、浮点数和复数，但是取模运算符`%`仅用于整数间的运算。对于不同编程语言，`%`取模运算的行为可能并不相同。在Go语言中，`%`取模运算符的符号和被取模数的符号总是一致的，因此`-5%3`和`-5%-3`结果都是`-2`。除法运算符`/`的行为则依赖于操作数是否全为整数，比如`5.0/4.0`的结果是`1.25`，但是`5/4`的结果是`1`，因为整数除法会向着`0`方向截断余数。

一个算术运算的结果，不管是有符号或者是无符号的，如果需要更多的bit位才能正确表示的话，就说明计算结果是溢出了。超出的高位的bit位部分将被丢弃。如果原始的数值是有符号类型，而且最左边的bit位是1的话，那么最终结果可能是负的，例如`int8`的例子：

```go
var u uint8 = 255
fmt.Println(u, u+1, u*u) // "255 0 1"

var i int8 = 127
fmt.Println(i, i+1, i*i) // "127 -128 1"
```

两个相同的整数类型可以使用下面的二元比较运算符进行比较；比较表达式的结果是布尔类型。

```go
==    等于
!=    不等于
<     小于
<=    小于等于
>     大于
>=    大于等于
```

事实上，布尔型、数字类型和字符串等基本类型都是可比较的，也就是说两个相同类型的值可以用`==`和`!=`进行比较。此外，整数、浮点数和字符串可以根据比较结果排序。许多其它类型的值可能是不可比较的，因此也就可能是不可排序的。对于我们遇到的每种类型，我们需要保证规则的一致性。

这里是一元的加法和减法运算符：

```go
+      一元加法 (无效果)
-      负数
```

对于整数，`+x`是`0+x`的简写，`-x`则是`0-x`的简写；对于浮点数和复数，`+x`就是`x`，`-x`则是`x` 的负数。

Go语言还提供了以下的bit位操作运算符，前面4个操作运算符并不区分是有符号还是无符号数：

```go
&      位运算 AND
|      位运算 OR
^      位运算 XOR
&^     位清空 (AND NOT)
<<     左移
>>     右移
```

位操作运算符`^`作为二元运算符时是按位异或（XOR），当用作一元运算符时表示按位取反；也就是说，它返回一个每个bit位都取反的数。位操作运算符`&^`用于按位置零（AND NOT）：如果对应`y`中bit位为1的话, 表达式`z = x &^ y`结果`z`的对应的bit位为0，否则`z`对应的bit位等于`x`相应的bit位的值。

下面的代码演示了如何使用位操作解释`uint8`类型值的8个独立的bit位。它使用了`Printf`函数的`%b`参数打印二进制格式的数字；其中`%08b`中`08`表示打印至少`8`个字符宽度，不足的前缀部分用`0`填充。

```go
var x uint8 = 1<<1 | 1<<5
var y uint8 = 1<<1 | 1<<2

fmt.Printf("%08b\n", x) // "00100010", the set {1, 5}
fmt.Printf("%08b\n", y) // "00000110", the set {1, 2}

fmt.Printf("%08b\n", x&y)  // "00000010", the intersection {1}
fmt.Printf("%08b\n", x|y)  // "00100110", the union {1, 2, 5}
fmt.Printf("%08b\n", x^y)  // "00100100", the symmetric difference {2, 5}
fmt.Printf("%08b\n", x&^y) // "00100000", the difference {5}

for i := uint(0); i < 8; i++ {
    if x&(1<<i) != 0 { // membership test
        fmt.Println(i) // "1", "5"
    }
}

fmt.Printf("%08b\n", x<<1) // "01000100", the set {2, 6}
fmt.Printf("%08b\n", x>>1) // "00010001", the set {0, 4}
```

（6.5节给出了一个可以远大于一个字节的整数集的实现。）

在`x<<n`和`x>>n`移位运算中，决定了移位操作的bit数部分必须是无符号数；被操作的`x`可以是有符号数或无符号数。算术上，一个`x<<n`左移运算等价于乘以 $2^n$，一个`x>>n`右移运算等价于除以$2^n$。

左移运算用零填充右边空缺的bit位，无符号数的右移运算也是用`0`填充左边空缺的bit位，但是有符号数的右移运算会用符号位的值填充左边空缺的bit位。因为这个原因，最好用无符号运算，这样你可以将整数完全当作一个bit位模式处理。

尽管Go语言提供了无符号数的运算，但即使数值本身不可能出现负数，我们还是倾向于使用有符号的`int`类型，就像数组的长度那样，虽然使用`uint`无符号类型似乎是一个更合理的选择。事实上，内置的`len`函数返回一个有符号的`int`，我们可以像下面例子那样处理逆序循环。

```go
medals := []string{"gold", "silver", "bronze"}
for i := len(medals) - 1; i >= 0; i-- {
    fmt.Println(medals[i]) // "bronze", "silver", "gold"
}
```

另一个选择对于上面的例子来说将是灾难性的。如果`len`函数返回一个无符号数，那么`i`也将是无符号的`uint`类型，然后条件`i >= 0`则永远为真。在三次迭代之后，也就是`i == 0`时，`i--`语句将不会产生-1，而是变成一个`uint`类型的最大值（可能是$2^{64-1}$），然后`medals[i]`表达式运行时将发生`panic`异常（§5.9），也就是试图访问一个slice范围以外的元素。

出于这个原因，无符号数往往只有在位运算或其它特殊的运算场景才会使用，就像bit集合、分析二进制文件格式或者是哈希和加密操作等。它们通常并不用于仅仅是表达非负数量的场合。

一般来说，需要一个显式的转换将一个值从一种类型转化为另一种类型，并且算术和逻辑运算的二元操作中必须是相同的类型。虽然这偶尔会导致需要很长的表达式，但是它消除了所有和类型相关的问题，而且也使得程序容易理解。

在很多场景，会遇到类似下面代码的常见的错误：

```go
var apples int32 = 1
var oranges int16 = 2
var compote int = apples + oranges // compile error
```

当尝试编译这三个语句时，将产生一个错误信息：

```go
invalid operation: apples + oranges (mismatched types int32 and int16)
```

这种类型不匹配的问题可以有几种不同的方法修复，最常见方法是将它们都显式转型为一个常见类型：

```go
var compote = int(apples) + int(oranges)
```

如2.5节所述，对于每种类型`T`，如果转换允许的话，类型转换操作`T(x)`将`x`转换为`T`类型。许多整数之间的相互转换并不会改变数值；它们只是告诉编译器如何解释这个值。但是对于将一个大尺寸的整数类型转为一个小尺寸的整数类型，或者是将一个浮点数转为整数，可能会改变数值或丢失精度：

```go
f := 3.141 // a float64
i := int(f)
fmt.Println(f, i) // "3.141 3"
f = 1.99
fmt.Println(int(f)) // "1"
```

浮点数到整数的转换将丢失任何小数部分，然后向数轴零方向截断。你应该避免对可能会超出目标类型表示范围的数值做类型转换，因为截断的行为可能依赖于具体的实现：

```go
f := 1e100  // a float64
i := int(f) // 结果依赖于具体实现
```

任何大小的整数字面值都可以用以`0`开始的八进制格式书写，例如`0666`；或用以`0x`或`0X`开头的十六进制格式书写，例如`0xdeadbeef`。十六进制数字可以用大写或小写字母。如今八进制数据通常用于POSIX操作系统上的文件访问权限标志，十六进制数字则更强调数字值的bit位模式。

当使用`fmt`包打印一个数值时，我们可以用`%d`、`%o`或`%x`参数控制输出的进制格式，就像下面的例子：

```go
o := 0666
fmt.Printf("%d %[1]o %#[1]o\n", o) // "438 666 0666"
x := int64(0xdeadbeef)
fmt.Printf("%d %[1]x %#[1]x %#[1]X\n", x)
// Output:
// 3735928559 deadbeef 0xdeadbeef 0XDEADBEEF
```

请注意`fmt`的两个使用技巧。通常`Printf`格式化字符串包含多个`%`参数时将会包含对应相同数量的额外操作数，但是`%`之后的`[1]`副词告诉`Printf`函数再次使用第一个操作数。第二，`%`后的`#`副词告诉`Printf`在用`%o`、`%x`或`%X`输出时生成`0`、`0x`或`0X`前缀。

字符面值通过一对单引号直接包含对应字符。最简单的例子是ASCII中类似’a’写法的字符面值，但是我们也可以通过转义的数值来表示任意的Unicode码点对应的字符，马上将会看到这样的例子。

字符使用`%c`参数打印，或者是用`%q`参数打印带单引号的字符：

```go
ascii := 'a'
unicode := '国'
newline := '\n'
fmt.Printf("%d %[1]c %[1]q\n", ascii)   // "97 a 'a'"
fmt.Printf("%d %[1]c %[1]q\n", unicode) // "22269 国 '国'"
fmt.Printf("%d %[1]q\n", newline)       // "10 '\n'"
```

## 3.2. 浮点数

Go语言提供了两种精度的浮点数，`float32`和`float64`。它们的算术规范由**IEEE754**浮点数国际标准定义，该浮点数规范被所有现代的CPU支持。

这些浮点数类型的取值范围可以从很微小到很巨大。浮点数的范围极限值可以在`math`包找到。常量`math.MaxFloat32`表示`float32`能表示的最大数值，大约是 `3.4e38`；对应的`math.MaxFloat64`常量大约是`1.8e308`。它们分别能表示的最小值近似为`1.4e-45`和`4.9e-324`。

一个`float32`类型的浮点数可以提供大约6个十进制数的精度，而`float64`则可以提供约15个十进制数的精度；通常应该优先使用`float64`类型，因为`float32`类型的累计计算误差很容易扩散，并且`float32`能精确表示的正整数并不是很大（译注：因为`float32`的有效bit位只有`23`个，其它的bit位用于指数和符号；当整数大于`23bit`能表达的范围时，`float32`的表示将出现误差）：

```go
var f float32 = 16777216 // 1 << 24
fmt.Println(f == f+1)    // "true"!
```

浮点数的字面值可以直接写小数部分，像这样：

```go
const e = 2.71828 // (approximately)
```

小数点前面或后面的数字都可能被省略（例如.707或1.）。很小或很大的数最好用科学计数法书写，通过e或E来指定指数部分：

```go
const Avogadro = 6.02214129e23  // 阿伏伽德罗常数
const Planck   = 6.62606957e-34 // 普朗克常数
```

用`Printf`函数的`%g`参数打印浮点数，将采用更紧凑的表示形式打印，并提供足够的精度，但是对应表格的数据，使用`%e`（带指数）或`%f`的形式打印可能更合适。所有的这三个打印形式都可以指定打印的宽度和控制打印精度。

```go
for x := 0; x < 8; x++ {
    fmt.Printf("x = %d e^x = %8.3f\n", x, math.Exp(float64(x)))
}
```

上面代码打印`e`的幂，打印精度是小数点后三个小数精度和8个字符宽度：

```go
x = 0       e^x =    1.000
x = 1       e^x =    2.718
x = 2       e^x =    7.389
x = 3       e^x =   20.086
x = 4       e^x =   54.598
x = 5       e^x =  148.413
x = 6       e^x =  403.429
x = 7       e^x = 1096.633
```

`math`包中除了提供大量常用的数学函数外，还提供了**IEEE754**浮点数标准中定义的特殊值的创建和测试：正无穷大和负无穷大，分别用于表示太大溢出的数字和除零的结果；还有`NaN`非数，一般用于表示无效的除法操作结果`0/0`或`Sqrt(-1)`.

```go
var z float64
fmt.Println(z, -z, 1/z, -1/z, z/z) // "0 -0 +Inf -Inf NaN"
```

函数`math.IsNaN`用于测试一个数是否是非数`NaN`，`math.NaN`则返回非数对应的值。虽然可以用`math.NaN`来表示一个非法的结果，但是测试一个结果是否是非数`NaN`则是充满风险的，因为`NaN`和任何数都是不相等的（译注：在浮点数中，`NaN`、正无穷大和负无穷大都不是唯一的，每个都有非常多种的bit模式表示）：

```go
nan := math.NaN()
fmt.Println(nan == nan, nan < nan, nan > nan) // "false false false"
```

如果一个函数返回的浮点数结果可能失败，最好的做法是用单独的标志报告失败，像这样：

```go
func compute() (value float64, ok bool) {
    // ...
    if failed {
        return 0, false
    }
    return result, true
}
```

接下来的程序演示了通过浮点计算生成的图形。它是带有两个参数的`z = f(x, y)`函数的三维形式，使用了可缩放矢量图形（SVG）格式输出，SVG是一个用于矢量线绘制的XML标准。图3.1显示了`sin(r)/r`函数的输出图形，其中`r`是`sqrt(x*x+y*y)`。

![浮点数 - 图1](images\ch3-01.png)

*gopl.io/ch3/surface*

```go
// Surface computes an SVG rendering of a 3-D surface function.
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

func corner(i, j int) (float64, float64) {
    // Find point (x,y) at corner of cell (i,j).
    x := xyrange * (float64(i)/cells - 0.5)
    y := xyrange * (float64(j)/cells - 0.5)
    // Compute surface height z.
    z := f(x, y)
    // Project (x,y,z) isometrically onto 2-D SVG canvas (sx,sy).
    sx := width/2 + (x-y)*cos30*xyscale
    sy := height/2 + (x+y)*sin30*xyscale - z*zscale
    return sx, sy
}

func f(x, y float64) float64 {
    r := math.Hypot(x, y) // distance from (0,0)
    return math.Sin(r) / r
}
```

要注意的是`corner`函数返回了两个结果，分别对应每个网格顶点的坐标参数。

要解释这个程序是如何工作的需要一些基本的几何学知识，但是我们可以跳过几何学原理，因为程序的重点是演示浮点数运算。程序的本质是三个不同的坐标系中映射关系，如图3.2所示。第一个是`100x100`的二维网格，对应整数坐标(`i,j`)，从远处的(`0, 0`)位置开始。我们从远处向前面绘制，因此远处先绘制的多边形有可能被前面后绘制的多边形覆盖。

第二个坐标系是一个三维的网格浮点坐标(`x,y,z`)，其中`x`和`y`是`i`和`j`的线性函数，通过平移转换为网格单元的中心，然后用`xyrange`系数缩放。高度z是函数`f(x,y)`的值。

第三个坐标系是一个二维的画布，起点(0,0)在左上角。画布中点的坐标用(`sx, sy`)表示。我们使用等角投影将三维点

![浮点数 - 图2](images\ch3-02.png)

(`x,y,z`)投影到二维的画布中。画布中从远处到右边的点对应较大的x值和较大的`y`值。并且画布中`x`和`y`值越大，则对应的`z`值越小。`x`和`y`的垂直和水平缩放系数来自30度角的正弦和余弦值。z的缩放系数0.4，是一个任意选择的参数。

对于二维网格中的每一个网格单元，`main`函数计算单元的四个顶点在画布中对应多边形ABCD的顶点，其中B对应(`i,j`)顶点位置，A、C和D是其它相邻的顶点，然后输出SVG的绘制指令。

**练习 3.1：** 如果f函数返回的是无限制的`float64`值，那么SVG文件可能输出无效的多边形元素（虽然许多SVG渲染器会妥善处理这类问题）。修改程序跳过无效的多边形。

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
			if math.IsNaN(ax) || math.IsNaN(ay) || math.IsNaN(bx) || math.IsNaN(by) || math.IsNaN(cx) || math.IsNaN(cy) || math.IsNaN(dx) || math.IsNaN(dy) {
				continue
			}
			fmt.Printf("<polygon points='%g,%g %g,%g %g,%g %g,%g'/>\n",
				ax, ay, bx, by, cx, cy, dx, dy)
		}
	}
	fmt.Println("</svg>")
}

func corner(i, j int) (float64, float64) {
	// Find point (x,y) at corner of cell (i,j).
	x := xyrange * (float64(i)/cells - 0.5)
	y := xyrange * (float64(j)/cells - 0.5)

	// Compute surface height z.
	z := f(x, y)

	// Project (x,y,z) isometrically onto 2-D SVG canvas (sx,sy).
	sx := width/2 + (x-y)*cos30*xyscale
	sy := height/2 + (x+y)*sin30*xyscale - z*zscale
	return sx, sy
}

func f(x, y float64) float64 {
	r := math.Hypot(x, y) // distance from (0,0)
	return math.Sin(r) / r
}
```

**练习 3.2：** 试验`math`包中其他函数的渲染图形。你是否能输出一个`egg box`、`moguls`或`a saddle`图案?

```go
package main

import (
	"fmt"
	"io"
	"math"
	"os"
)

const (
	width, height = 600, 320            // canvas size in pixels
	cells         = 40                  // number of grid cells
	xyrange       = 30.0                // axis ranges (-xyrange..+xyrange)
	xyscale       = width / 2 / xyrange // pixels per x or y unit
	zscale        = height * 0.4        // pixels per z unit
	angle         = math.Pi / 6         // angle of x, y axes (=30°)
)

var sin30, cos30 = math.Sin(angle), math.Cos(angle) // sin(30°), cos(30°)

type zFunc func(x, y float64) float64

func svg(w io.Writer, f zFunc) {
	fmt.Fprintf(w, "<svg xmlns='http://www.w3.org/2000/svg' "+
		"style='stroke: grey; fill: white; stroke-width: 0.7' "+
		"width='%d' height='%d'>", width, height)
	for i := 0; i < cells; i++ {
		for j := 0; j < cells; j++ {
			ax, ay := corner(i+1, j, f)
			bx, by := corner(i, j, f)
			cx, cy := corner(i, j+1, f)
			dx, dy := corner(i+1, j+1, f)
			if math.IsNaN(ax) || math.IsNaN(ay) || math.IsNaN(bx) || math.IsNaN(by) || math.IsNaN(cx) || math.IsNaN(cy) || math.IsNaN(dx) || math.IsNaN(dy) {
				continue
			}
			fmt.Fprintf(w, "<polygon style='stroke: %s; fill: #222222' points='%g,%g %g,%g %g,%g %g,%g'/>\n",
				"#666666", ax, ay, bx, by, cx, cy, dx, dy)
		}
	}
	fmt.Fprintln(w, "</svg>")
}

func corner(i, j int, f zFunc) (float64, float64) {
	// Find point (x,y) at corner of cell (i,j).
	x := xyrange * (float64(i)/cells - 0.5)
	y := xyrange * (float64(j)/cells - 0.5)

	// Compute surface height z.
	z := f(x, y)

	// Project (x,y,z) isometrically onto 2-D SVG canvas (sx,sy).
	sx := width/2 + (x-y)*cos30*xyscale
	sy := height/2 + (x+y)*sin30*xyscale - z*zscale
	return sx, sy
}

func eggbox(x, y float64) float64 {
	return 0.2 * (math.Cos(x) + math.Cos(y))
}

func saddle(x, y float64) float64 {
	a := 25.0
	b := 17.0
	a2 := a * a
	b2 := b * b
	return (y*y/a2 - x*x/b2)
}

func main() {
	usage := "usage: ex3.2 saddle|eggbox"
	if len(os.Args) < 2 {
		fmt.Fprintln(os.Stderr, usage)
		os.Exit(1)
	}
	var f zFunc
	switch os.Args[1] {
	case "saddle":
		f = saddle
	case "eggbox":
		f = eggbox
	default:
		fmt.Fprintln(os.Stderr, usage)
		os.Exit(1)
	}
	svg(os.Stdout, f)
}
```

**练习 3.3：** 根据高度给每个多边形上色，那样峰值部将是红色`(#ff0000`)，谷部将是蓝色(`#0000ff`)。

```go
package main

import (
	"fmt"
	"io"
	"math"
	"os"
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

func svg(w io.Writer) {
	zmin, zmax := minmax()
	fmt.Fprintf(w, "<svg xmlns='http://www.w3.org/2000/svg' "+
		"style='stroke: grey; fill: white; stroke-width: 0.7' "+
		"width='%d' height='%d'>", width, height)
	for i := 0; i < cells; i++ {
		for j := 0; j < cells; j++ {
			ax, ay := corner(i+1, j)
			bx, by := corner(i, j)
			cx, cy := corner(i, j+1)
			dx, dy := corner(i+1, j+1)
			if math.IsNaN(ax) || math.IsNaN(ay) || math.IsNaN(bx) || math.IsNaN(by) || math.IsNaN(cx) || math.IsNaN(cy) || math.IsNaN(dx) || math.IsNaN(dy) {
				continue
			}
			fmt.Fprintf(w, "<polygon style='stroke: %s; fill: #222222' points='%g,%g %g,%g %g,%g %g,%g'/>\n",
				color(i, j, zmin, zmax), ax, ay, bx, by, cx, cy, dx, dy)
		}
	}
	fmt.Fprintln(w, "</svg>")
}

func main() {
	svg(os.Stdout)
}

// minmax returns the min and max values for z given the min/max values of x
// and y and assuming a square domain.
func minmax() (min float64, max float64) {
	min = math.NaN()
	max = math.NaN()
	for i := 0; i < cells; i++ {
		for j := 0; j < cells; j++ {
			for xoff := 0; xoff <= 1; xoff++ {
				for yoff := 0; yoff <= 1; yoff++ {
					x := xyrange * (float64(i+xoff)/cells - 0.5)
					y := xyrange * (float64(j+yoff)/cells - 0.5)
					z := f(x, y)
					if math.IsNaN(min) || z < min {
						min = z
					}
					if math.IsNaN(max) || z > max {
						max = z
					}
				}
			}
		}
	}
	return
}

func color(i, j int, zmin, zmax float64) string {
	min := math.NaN()
	max := math.NaN()
	for xoff := 0; xoff <= 1; xoff++ {
		for yoff := 0; yoff <= 1; yoff++ {
			x := xyrange * (float64(i+xoff)/cells - 0.5)
			y := xyrange * (float64(j+yoff)/cells - 0.5)
			z := f(x, y)
			if math.IsNaN(min) || z < min {
				min = z
			}
			if math.IsNaN(max) || z > max {
				max = z
			}
		}
	}

	color := ""
	if math.Abs(max) > math.Abs(min) {
		red := math.Exp(math.Abs(max)) / math.Exp(math.Abs(zmax)) * 255
		if red > 255 {
			red = 255
		}
		color = fmt.Sprintf("#%02x0000", int(red))
	} else {
		blue := math.Exp(math.Abs(min)) / math.Exp(math.Abs(zmin)) * 255
		if blue > 255 {
			blue = 255
		}
		color = fmt.Sprintf("#0000%02x", int(blue))
	}
	return color
}

func corner(i, j int) (float64, float64) {
	// Find point (x,y) at corner of cell (i,j).
	x := xyrange * (float64(i)/cells - 0.5)
	y := xyrange * (float64(j)/cells - 0.5)

	// Compute surface height z.
	z := f(x, y)

	// Project (x,y,z) isometrically onto 2-D SVG canvas (sx,sy).
	sx := width/2 + (x-y)*cos30*xyscale
	sy := height/2 + (x+y)*sin30*xyscale - z*zscale
	return sx, sy
}

func f(x, y float64) float64 {
	r := math.Hypot(x, y) // distance from (0,0)
	return math.Sin(r) / r
}
```

**练习 3.4：** 参考1.7节`Lissajous`例子的函数，构造一个web服务器，用于计算函数曲面然后返回`SVG`数据给客户端。服务器必须设置`Content-Type`头部：

```go
w.Header().Set("Content-Type", "image/svg+xml")
```

（这一步在`Lissajous`例子中不是必须的，因为服务器使用标准的PNG图像格式，可以根据前面的512个字节自动输出对应的头部。）允许客户端通过HTTP请求参数设置高度、宽度和颜色等参数。

```go
package main

import (
	"fmt"
	"io"
	"log"
	"math"
	"net/http"
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

func svg(w io.Writer) {
	zmin, zmax := minmax()
	fmt.Fprintf(w, "<svg xmlns='http://www.w3.org/2000/svg' "+
		"style='stroke: grey; fill: white; stroke-width: 0.7' "+
		"width='%d' height='%d'>", width, height)
	for i := 0; i < cells; i++ {
		for j := 0; j < cells; j++ {
			ax, ay := corner(i+1, j)
			bx, by := corner(i, j)
			cx, cy := corner(i, j+1)
			dx, dy := corner(i+1, j+1)
			if math.IsNaN(ax) || math.IsNaN(ay) || math.IsNaN(bx) || math.IsNaN(by) || math.IsNaN(cx) || math.IsNaN(cy) || math.IsNaN(dx) || math.IsNaN(dy) {
				continue
			}
			fmt.Fprintf(w, "<polygon style='stroke: %s; fill: #222222' points='%g,%g %g,%g %g,%g %g,%g'/>\n",
				color(i, j, zmin, zmax), ax, ay, bx, by, cx, cy, dx, dy)
		}
	}
	fmt.Fprintln(w, "</svg>")
}

func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "image/svg+xml")
		svg(w)
	})
	log.Fatal(http.ListenAndServe(":8080", nil))
}

func minmax() (min float64, max float64) {
	min = math.NaN()
	max = math.NaN()
	for i := 0; i < cells; i++ {
		for j := 0; j < cells; j++ {
			for xoff := 0; xoff <= 1; xoff++ {
				for yoff := 0; yoff <= 1; yoff++ {
					x := xyrange * (float64(i+xoff)/cells - 0.5)
					y := xyrange * (float64(j+yoff)/cells - 0.5)
					z := f(x, y)
					if math.IsNaN(min) || z < min {
						min = z
					}
					if math.IsNaN(max) || z > max {
						max = z
					}
				}
			}
		}
	}
	return
}

func color(i, j int, zmin, zmax float64) string {
	min := math.NaN()
	max := math.NaN()
	for xoff := 0; xoff <= 1; xoff++ {
		for yoff := 0; yoff <= 1; yoff++ {
			x := xyrange * (float64(i+xoff)/cells - 0.5)
			y := xyrange * (float64(j+yoff)/cells - 0.5)
			z := f(x, y)
			if math.IsNaN(min) || z < min {
				min = z
			}
			if math.IsNaN(max) || z > max {
				max = z
			}
		}
	}

	color := ""
	if math.Abs(max) > math.Abs(min) {
		red := math.Exp(math.Abs(max)) / math.Exp(math.Abs(zmax)) * 255
		if red > 255 {
			red = 255
		}
		color = fmt.Sprintf("#%02x0000", int(red))
	} else {
		blue := math.Exp(math.Abs(min)) / math.Exp(math.Abs(zmin)) * 255
		if blue > 255 {
			blue = 255
		}
		color = fmt.Sprintf("#0000%02x", int(blue))
	}
	return color
}

func corner(i, j int) (float64, float64) {
	// Find point (x,y) at corner of cell (i,j).
	x := xyrange * (float64(i)/cells - 0.5)
	y := xyrange * (float64(j)/cells - 0.5)

	// Compute surface height z.
	z := f(x, y)

	// Project (x,y,z) isometrically onto 2-D SVG canvas (sx,sy).
	sx := width/2 + (x-y)*cos30*xyscale
	sy := height/2 + (x+y)*sin30*xyscale - z*zscale
	return sx, sy
}

func f(x, y float64) float64 {
	r := math.Hypot(x, y) // distance from (0,0)
	return math.Sin(r) / r
}
```

## 3.3. 复数

Go语言提供了两种精度的复数类型：`complex64`和`complex128`，分别对应`float32`和`float64`两种浮点数精度。内置的`complex`函数用于构建复数，内建的`real`和`imag`函数分别返回复数的实部和虚部：

```go
var x complex128 = complex(1, 2) // 1+2i
var y complex128 = complex(3, 4) // 3+4i
fmt.Println(x*y)                 // "(-5+10i)"
fmt.Println(real(x*y))           // "-5"
fmt.Println(imag(x*y))           // "10"
```

如果一个浮点数面值或一个十进制整数面值后面跟着一个`i`，例如`3.141592i`或`2i`，它将构成一个复数的虚部，复数的实部是`0`：

```go
fmt.Println(1i * 1i) // "(-1+0i)", i^2 = -1
```

在常量算术规则下，一个复数常量可以加到另一个普通数值常量（整数或浮点数、实部或虚部），我们可以用自然的方式书写复数，就像`1+2i`或与之等价的写法`2i+1`。上面x和y的声明语句还可以简化：

```go
x := 1 + 2i
y := 3 + 4i
```

复数也可以用`==`和`!=`进行相等比较。只有两个复数的实部和虚部都相等的时候它们才是相等的（译注：浮点数的相等比较是危险的，需要特别小心处理精度问题）。

`math/cmplx`包提供了复数处理的许多函数，例如求复数的平方根函数和求幂函数。

```go
fmt.Println(cmplx.Sqrt(-1)) // "(0+1i)"
```

下面的程序使用`complex128`复数算法来生成一个`Mandelbrot`图像。

*gopl.io/ch3/mandelbrot*

```go
// Mandelbrot emits a PNG image of the Mandelbrot fractal.
package main

import (
    "image"
    "image/color"
    "image/png"
    "math/cmplx"
    "os"
)

func main() {
    const (
        xmin, ymin, xmax, ymax = -2, -2, +2, +2
        width, height          = 1024, 1024
    )
    img := image.NewRGBA(image.Rect(0, 0, width, height))
    for py := 0; py < height; py++ {
        y := float64(py)/height*(ymax-ymin) + ymin
        for px := 0; px < width; px++ {
            x := float64(px)/width*(xmax-xmin) + xmin
            z := complex(x, y)
            // Image point (px, py) represents complex value z.
            img.Set(px, py, mandelbrot(z))
        }
    }
    png.Encode(os.Stdout, img) // NOTE: ignoring errors
}

func mandelbrot(z complex128) color.Color {
    const iterations = 200
    const contrast = 15
    var v complex128
    for n := uint8(0); n < iterations; n++ {
        v = v*v + z
        if cmplx.Abs(v) > 2 {
            return color.Gray{255 - contrast*n}
        }
    }
    return color.Black
}
```

用于遍历`1024x1024`图像每个点的两个嵌套的循环对应`-2`到`+2`区间的复数平面。程序反复测试每个点对应复数值平方值加一个增量值对应的点是否超出半径为2的圆。如果超过了，通过根据预设置的逃逸迭代次数对应的灰度颜色来代替。如果不是，那么该点属于`Mandelbrot`集合，使用黑色颜色标记。最终程序将生成的PNG格式分形图像输出到标准输出，如图3.3所示。

![复数 - 图1](images\ch3-03.png)

**练习 3.5：** 实现一个彩色的Mandelbrot图像，使用`image.NewRGBA`创建图像，使用`color.RGBA`或`color.YCbCr`生成颜色。

```go
package main

import (
	"image"
	"image/color"
	"image/png"
	"math"
	"math/cmplx"
	"os"
)

func main() {
	const (
		xmin, ymin, xmax, ymax = -2, -2, +2, +2
		width, height          = 1024, 1024
	)

	img := image.NewRGBA(image.Rect(0, 0, width, height))
	for py := 0; py < height; py++ {
		y := float64(py)/height*(ymax-ymin) + ymin
		for px := 0; px < width; px++ {
			x := float64(px)/width*(xmax-xmin) + xmin
			z := complex(x, y)
			// Image point (px, py) represents complex value z.
			img.Set(px, py, mandelbrot(z))
		}
	}
	png.Encode(os.Stdout, img) // NOTE: ignoring errors
}

func mandelbrot(z complex128) color.Color {
	const iterations = 200
	const contrast = 15

	var v complex128
	for n := uint8(0); n < iterations; n++ {
		v = v*v + z
		if cmplx.Abs(v) > 2 {
			switch {
			case n > 50: // dark red
				return color.RGBA{100, 0, 0, 255}
			default:
				// logarithmic blue gradient to show small differences on the
				// periphery of the fractal.
				logScale := math.Log(float64(n)) / math.Log(float64(iterations))
				return color.RGBA{0, 0, 255 - uint8(logScale*255), 255}
			}
		}
	}
	return color.Black
}

// Some other interesting functions:

func acos(z complex128) color.Color {
	v := cmplx.Acos(z)
	blue := uint8(real(v)*128) + 127
	red := uint8(imag(v)*128) + 127
	return color.YCbCr{192, blue, red}
}

func sqrt(z complex128) color.Color {
	v := cmplx.Sqrt(z)
	blue := uint8(real(v)*128) + 127
	red := uint8(imag(v)*128) + 127
	return color.YCbCr{128, blue, red}
}

// f(x) = x^4 - 1
//
// z' = z - f(z)/f'(z)
//    = z - (z^4 - 1) / (4 * z^3)
//    = z - (z - 1/z^3) / 4
func newton(z complex128) color.Color {
	const iterations = 37
	const contrast = 7
	for i := uint8(0); i < iterations; i++ {
		z -= (z - 1/(z*z*z)) / 4
		if cmplx.Abs(z*z*z*z-1) < 1e-6 {
			return color.Gray{255 - contrast*i}
		}
	}
	return color.Black
}
```

**练习 3.6：** 升采样技术可以降低每个像素对计算颜色值和平均值的影响。简单的方法是将每个像素分成四个子像素，实现它。

```go
package main

import (
	"image"
	"image/color"
	"image/png"
	"math"
	"math/cmplx"
	"os"
)

func main() {
	const (
		xmin, ymin, xmax, ymax = -2, -2, +2, +2
		width, height          = 1024, 1024
		epsX                   = (xmax - xmin) / width
		epsY                   = (ymax - ymin) / height
	)
	offX := []float64{-epsX, epsX}
	offY := []float64{-epsY, epsY}

	img := image.NewRGBA(image.Rect(0, 0, width, height))
	for py := 0; py < height; py++ {
		y := float64(py)/height*(ymax-ymin) + ymin
		for px := 0; px < width; px++ {
			x := float64(px)/width*(xmax-xmin) + xmin
			// Supersampling:
			subPixels := make([]color.Color, 0)
			for i := 0; i < 2; i++ {
				for j := 0; j < 2; j++ {
					z := complex(x+offX[i], y+offY[j])
					subPixels = append(subPixels, mandelbrot(z))
				}
			}
			img.Set(px, py, avg(subPixels))
		}
	}
	png.Encode(os.Stdout, img) // NOTE: ignoring errors
}

func avg(colors []color.Color) color.Color {
	var r, g, b, a uint16
	n := len(colors)
	for _, c := range colors {
		r_, g_, b_, a_ := c.RGBA()
		r += uint16(r_ / uint32(n))
		g += uint16(g_ / uint32(n))
		b += uint16(b_ / uint32(n))
		a += uint16(a_ / uint32(n))
	}
	return color.RGBA64{r, g, b, a}
}

func mandelbrot(z complex128) color.Color {
	const iterations = 200
	const contrast = 15

	var v complex128
	for n := uint8(0); n < iterations; n++ {
		v = v*v + z
		if cmplx.Abs(v) > 2 {
			switch {
			case n > 50: // dark red
				return color.RGBA{100, 0, 0, 255}
			default:
				// logarithmic blue gradient to show small differences on the
				// periphery of the fractal.
				logScale := math.Log(float64(n)) / math.Log(float64(iterations))
				return color.RGBA{0, 0, 255 - uint8(logScale*255), 255}
			}
		}
	}
	return color.Black
}

// Some other interesting functions:

func acos(z complex128) color.Color {
	v := cmplx.Acos(z)
	blue := uint8(real(v)*128) + 127
	red := uint8(imag(v)*128) + 127
	return color.YCbCr{192, blue, red}
}

func sqrt(z complex128) color.Color {
	v := cmplx.Sqrt(z)
	blue := uint8(real(v)*128) + 127
	red := uint8(imag(v)*128) + 127
	return color.YCbCr{128, blue, red}
}

// f(x) = x^4 - 1
//
// z' = z - f(z)/f'(z)
//    = z - (z^4 - 1) / (4 * z^3)
//    = z - (z - 1/z^3) / 4
func newton(z complex128) color.Color {
	const iterations = 37
	const contrast = 7
	for i := uint8(0); i < iterations; i++ {
		z -= (z - 1/(z*z*z)) / 4
		if cmplx.Abs(z*z*z*z-1) < 1e-6 {
			return color.Gray{255 - contrast*i}
		}
	}
	return color.Black
}
```

**练习 3.7：** 另一个生成分形图像的方式是使用牛顿法来求解一个复数方程，例如$z^4-1=0$。每个起点到四个根的迭代次数对应阴影的灰度。方程根对应的点用颜色表示。

```go
package main

import (
	"image"
	"image/color"
	"image/png"
	"math"
	"math/cmplx"
	"os"
)

type Func func(complex128) complex128

var colorPool = []color.RGBA{
	{170, 57, 57, 255},
	{170, 108, 57, 255},
	{34, 102, 102, 255},
	{45, 136, 45, 255},
}

var chosenColors = map[complex128]color.RGBA{}

func main() {
	const (
		xmin, ymin, xmax, ymax = -2, -2, +2, +2
		width, height          = 1024, 1024
	)

	img := image.NewRGBA(image.Rect(0, 0, width, height))
	for py := 0; py < height; py++ {
		y := float64(py)/height*(ymax-ymin) + ymin
		for px := 0; px < width; px++ {
			x := float64(px)/width*(xmax-xmin) + xmin
			z := complex(x, y)
			// Image point (px, py) represents complex value z.
			img.Set(px, py, z4(z))
		}
	}
	png.Encode(os.Stdout, img) // NOTE: ignoring errors
}

func mandelbrot(z complex128) color.Color {
	const iterations = 200
	const contrast = 15

	var v complex128
	for n := uint8(0); n < iterations; n++ {
		v = v*v + z
		if cmplx.Abs(v) > 2 {
			switch {
			case n > 50: // dark red
				return color.RGBA{100, 0, 0, 255}
			default:
				// logarithmic blue gradient to show small differences on the
				// periphery of the fractal.
				logScale := math.Log(float64(n)) / math.Log(float64(iterations))
				return color.RGBA{0, 0, 255 - uint8(logScale*255), 255}
			}
		}
	}
	return color.Black
}

// Some other interesting functions:

func acos(z complex128) color.Color {
	v := cmplx.Acos(z)
	blue := uint8(real(v)*128) + 127
	red := uint8(imag(v)*128) + 127
	return color.YCbCr{192, blue, red}
}

func sqrt(z complex128) color.Color {
	v := cmplx.Sqrt(z)
	blue := uint8(real(v)*128) + 127
	red := uint8(imag(v)*128) + 127
	return color.YCbCr{128, blue, red}
}

// f(x) = x^4 - 1
//
// z' = z - f(z)/f'(z)
//    = z - (z^4 - 1) / (4 * z^3)
//    = z - (z - 1/z^3) / 4
func z4(z complex128) color.Color {
	f := func(z complex128) complex128 {
		return z*z*z*z - 1
	}
	fPrime := func(z complex128) complex128 {
		return (z - 1/(z*z*z)) / 4
	}
	return newton(z, f, fPrime)
}

// f(x) = x^4 + 2x^3 + 3x^2 + 4x - 5
//
// z' = z - f(z)/f'(z)
//    = z - (z^4 + 2z^3 + 3z^2 + 4z - 5) / (4z^3 + 6z^2 + 6z + 4)
func quartic(z complex128) color.Color {
	f := func(z complex128) complex128 {
		return z*z*z*z + 2*z*z*z + 3*z*z + 4*z - 5
	}
	fPrime := func(z complex128) complex128 {
		return (z*z*z*z + 2*z*z*z + 3*z*z + 4*z - 5) / (4*z*z*z + 6*z*z + 6*z + 4)
	}
	return newton(z, f, fPrime)
}

func newton(z complex128, f Func, fPrime Func) color.Color {
	const iterations = 37
	const contrast = 7
	for i := uint8(0); i < iterations; i++ {
		z -= fPrime(z)
		if cmplx.Abs(f(z)) < 1e-6 {
			root := complex(round(real(z), 4), round(imag(z), 4))
			c, ok := chosenColors[root]
			if !ok {
				if len(colorPool) == 0 {
					panic("no colors left")
				}
				c = colorPool[0]
				colorPool = colorPool[1:]
				chosenColors[root] = c
			}
			// Convert to YCbCr to make producing different shades easier.
			y, cb, cr := color.RGBToYCbCr(c.R, c.G, c.B)
			scale := math.Log(float64(i)) / math.Log(iterations)
			y -= uint8(float64(y) * scale)
			return color.YCbCr{y, cb, cr}
		}
	}
	return color.Black
}

func round(f float64, digits int) float64 {
	if math.Abs(f) < 0.5 {
		return 0
	}
	pow := math.Pow10(digits)
	return math.Trunc(f*pow+math.Copysign(0.5, f)) / pow
}
```

**练习 3.8：** 通过提高精度来生成更多级别的分形。使用四种不同精度类型的数字实现相同的分形：`complex64`、`complex128`、`big.Float`和`big.Rat`。（后面两种类型在`math/big`包声明。`Float`是有指定限精度的浮点数；`Rat`是无限精度的有理数。）它们间的性能和内存使用对比如何？当渲染图可见时缩放的级别是多少？

```go
// main.go
package main

import (
	"image"
	"image/color"
	"image/png"
	"math"
	"math/big"
	"math/cmplx"
	"os"
)

func main() {
	const (
		xmin, ymin, xmax, ymax = -2, -2, +2, +2
		width, height          = 1024, 1024
	)

	img := image.NewRGBA(image.Rect(0, 0, width, height))
	for py := 0; py < height; py++ {
		y := float64(py)/height*(ymax-ymin) + ymin
		for px := 0; px < width; px++ {
			x := float64(px)/width*(xmax-xmin) + xmin
			z := complex(x, y)
			// Image point (px, py) represents complex value z.
			img.Set(px, py, mandelbrotBigFloat(z))
		}
	}
	png.Encode(os.Stdout, img) // NOTE: ignoring errors
}

func mandelbrot(z complex128) color.Color {
	const iterations = 200
	const contrast = 15
	var v complex128
	for n := uint8(0); n < iterations; n++ {
		v = v*v + z
		if cmplx.Abs(v) > 2 {
			switch {
			case n > 50: // dark red
				return color.RGBA{100, 0, 0, 255}
			default:
				// logarithmic blue gradient to show small differences on the
				// periphery of the fractal.
				logScale := math.Log(float64(n)) / math.Log(float64(iterations))
				return color.RGBA{0, 0, 255 - uint8(logScale*255), 255}
			}
		}
	}
	return color.Black
}

func mandelbrot64(z complex128) color.Color {
	const iterations = 200
	const contrast = 15
	var v complex64
	for n := uint8(0); n < iterations; n++ {
		v = v*v + complex64(z)
		if cmplx.Abs(complex128(v)) > 2 {
			switch {
			case n > 50: // dark red
				return color.RGBA{100, 0, 0, 255}
			default:
				// logarithmic blue gradient to show small differences on the
				// periphery of the fractal.
				logScale := math.Log(float64(n)) / math.Log(float64(iterations))
				return color.RGBA{0, 0, 255 - uint8(logScale*255), 255}
			}
		}
	}
	return color.Black
}

func mandelbrotBigFloat(z complex128) color.Color {
	const iterations = 200
	const contrast = 15
	zR := (&big.Float{}).SetFloat64(real(z))
	zI := (&big.Float{}).SetFloat64(imag(z))
	var vR, vI = &big.Float{}, &big.Float{}
	for i := uint8(0); i < iterations; i++ {
		// (r+i)^2 = r^2 + 2ri + i^2
		vR2, vI2 := &big.Float{}, &big.Float{}
		vR2.Mul(vR, vR).Sub(vR2, (&big.Float{}).Mul(vI, vI)).Add(vR2, zR)
		vI2.Mul(vR, vI).Mul(vI2, big.NewFloat(2)).Add(vI2, zI)
		vR, vI = vR2, vI2
		squareSum := &big.Float{}
		squareSum.Mul(vR, vR).Add(squareSum, (&big.Float{}).Mul(vI, vI))
		if squareSum.Cmp(big.NewFloat(4)) == 1 {
			switch {
			case i > 50: // dark red
				return color.RGBA{100, 0, 0, 255}
			default:
				// logarithmic blue gradient to show small differences on the
				// periphery of the fractal.
				logScale := math.Log(float64(i)) / math.Log(float64(iterations))
				return color.RGBA{0, 0, 255 - uint8(logScale*255), 255}
			}
		}
	}
	return color.Black
}

func mandelbrotRat(z complex128) color.Color {
	// High-resolution images take an extremely long time to render with
	// iterations = 200. Multiplying arbitrary precision numbers has
	// algorithmic complexity of at least O(n*log(n)*log(log(n)))
	// (https://en.wikipedia.org/wiki/Arbitrary-precision_arithmetic#Implementation_issues).
	const iterations = 200
	const contrast = 15
	zR := (&big.Rat{}).SetFloat64(real(z))
	zI := (&big.Rat{}).SetFloat64(imag(z))
	var vR, vI = &big.Rat{}, &big.Rat{}
	for i := uint8(0); i < iterations; i++ {
		// (r+i)^2 = r^2 + 2ri + i^2
		vR2, vI2 := &big.Rat{}, &big.Rat{}
		vR2.Mul(vR, vR).Sub(vR2, (&big.Rat{}).Mul(vI, vI)).Add(vR2, zR)
		vI2.Mul(vR, vI).Mul(vI2, big.NewRat(2, 1)).Add(vI2, zI)
		vR, vI = vR2, vI2
		squareSum := &big.Rat{}
		squareSum.Mul(vR, vR).Add(squareSum, (&big.Rat{}).Mul(vI, vI))
		if squareSum.Cmp(big.NewRat(4, 1)) == 1 {
			switch {
			case i > 50: // dark red
				return color.RGBA{100, 0, 0, 255}
			default:
				// logarithmic blue gradient to show small differences on the
				// periphery of the fractal.
				logScale := math.Log(float64(i)) / math.Log(float64(iterations))
				return color.RGBA{0, 0, 255 - uint8(logScale*255), 255}
			}
		}
	}
	return color.Black
}

// Some other interesting functions:

func acos(z complex128) color.Color {
	v := cmplx.Acos(z)
	blue := uint8(real(v)*128) + 127
	red := uint8(imag(v)*128) + 127
	return color.YCbCr{192, blue, red}
}

func sqrt(z complex128) color.Color {
	v := cmplx.Sqrt(z)
	blue := uint8(real(v)*128) + 127
	red := uint8(imag(v)*128) + 127
	return color.YCbCr{128, blue, red}
}

// f(x) = x^4 - 1
//
// z' = z - f(z)/f'(z)
//    = z - (z^4 - 1) / (4 * z^3)
//    = z - (z - 1/z^3) / 4
func newton(z complex128) color.Color {
	const iterations = 37
	const contrast = 7
	for i := uint8(0); i < iterations; i++ {
		z -= (z - 1/(z*z*z)) / 4
		if cmplx.Abs(z*z*z*z-1) < 1e-6 {
			return color.Gray{255 - contrast*i}
		}
	}
	return color.Black
}
```

```go
// main_test.go
package main

import (
	"image/color"
	"testing"
)

func benchmarkMandelbrot(b *testing.B, f func(complex128) color.Color) {
	for i := 0; i < b.N; i++ {
		f(complex(float64(i), float64(i)))
	}
}

func BenchmarkMandelbrotComplex128(b *testing.B) {
	benchmarkMandelbrot(b, mandelbrot)
}

func BenchmarkMandelbrotComplex64(b *testing.B) {
	benchmarkMandelbrot(b, mandelbrot64)
}

func BenchmarkMandelbrotBigFloat(b *testing.B) {
	benchmarkMandelbrot(b, mandelbrotBigFloat)
}

func BenchmarkMandelbrotRat(b *testing.B) {
	benchmarkMandelbrot(b, mandelbrotRat)
}
```

**练习 3.9：** 编写一个web服务器，用于给客户端生成分形的图像。运行客户端通过HTTP参数指定x,y和zoom参数。

```go
package main

import (
	"fmt"
	"image"
	"image/color"
	"image/png"
	"log"
	"math"
	"math/cmplx"
	"net/http"
	"strconv"
)

func main() {
	const (
		width, height = 1024, 1024
	)
	params := map[string]float64{
		"xmin": -2,
		"xmax": 2,
		"ymin": -2,
		"ymax": 2,
		"zoom": 1,
	}

	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		for name := range params {
			s := r.FormValue(name)
			if s == "" {
				continue
			}
			f, err := strconv.ParseFloat(s, 64)
			if err != nil {
				http.Error(w, fmt.Sprintf("query param %s: %s", name, err), http.StatusBadRequest)
				return
			}
			params[name] = f
		}
		if params["xmax"] <= params["xmin"] || params["ymax"] <= params["ymin"] {
			http.Error(w, fmt.Sprintf("min coordinate greater than max"), http.StatusBadRequest)
			return
		}
		xmin := params["xmin"]
		xmax := params["xmax"]
		ymin := params["ymin"]
		ymax := params["ymax"]
		zoom := params["zoom"]

		lenX := xmax - xmin
		midX := xmin + lenX/2
		xmin = midX - lenX/2/zoom
		xmax = midX + lenX/2/zoom
		lenY := ymax - ymin
		midY := ymin + lenY/2
		ymin = midY - lenY/2/zoom
		ymax = midY + lenY/2/zoom

		img := image.NewRGBA(image.Rect(0, 0, width, height))
		for py := 0; py < height; py++ {
			y := float64(py)/height*(ymax-ymin) + ymin
			for px := 0; px < width; px++ {
				x := float64(px)/width*(xmax-xmin) + xmin
				z := complex(x, y)
				// Image point (px, py) represents complex value z.
				img.Set(px, py, mandelbrot(z))
			}
		}
		err := png.Encode(w, img)
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
		}
	})
	log.Fatal(http.ListenAndServe(":8080", nil))
}

func mandelbrot(z complex128) color.Color {
	const iterations = 200
	const contrast = 15

	var v complex128
	for n := uint8(0); n < iterations; n++ {
		v = v*v + z
		if cmplx.Abs(v) > 2 {
			switch {
			case n > 50: // dark red
				return color.RGBA{100, 0, 0, 255}
			default:
				// logarithmic blue gradient to show small differences on the
				// periphery of the fractal.
				logScale := math.Log(float64(n)) / math.Log(float64(iterations))
				return color.RGBA{0, 0, 255 - uint8(logScale*255), 255}
			}
		}
	}
	return color.Black
}

// Some other interesting functions:

func acos(z complex128) color.Color {
	v := cmplx.Acos(z)
	blue := uint8(real(v)*128) + 127
	red := uint8(imag(v)*128) + 127
	return color.YCbCr{192, blue, red}
}

func sqrt(z complex128) color.Color {
	v := cmplx.Sqrt(z)
	blue := uint8(real(v)*128) + 127
	red := uint8(imag(v)*128) + 127
	return color.YCbCr{128, blue, red}
}

// f(x) = x^4 - 1
//
// z' = z - f(z)/f'(z)
//    = z - (z^4 - 1) / (4 * z^3)
//    = z - (z - 1/z^3) / 4
func newton(z complex128) color.Color {
	const iterations = 37
	const contrast = 7
	for i := uint8(0); i < iterations; i++ {
		z -= (z - 1/(z*z*z)) / 4
		if cmplx.Abs(z*z*z*z-1) < 1e-6 {
			return color.Gray{255 - contrast*i}
		}
	}
	return color.Black
}
```

## 3.4. 布尔型

一个布尔类型的值只有两种：`true`和`false`。`if`和`for`语句的条件部分都是布尔类型的值，并且`==`和`<`等比较操作也会产生布尔型的值。一元操作符`!`对应逻辑非操作，因此`!true`的值为`false`，更罗嗦的说法是`(!true==false)==true`，虽然表达方式不一样，不过我们一般会采用简洁的布尔表达式，就像用`x`来表示`x==true`。

布尔值可以和`&&`（AND）和`||`（OR）操作符结合，并且有短路行为：如果运算符左边值已经可以确定整个布尔表达式的值，那么运算符右边的值将不再被求值，因此下面的表达式总是安全的：

```go
s != "" && s[0] == 'x'
```

其中`s[0]`操作如果应用于空字符串将会导致`panic`异常。

因为`&&`的优先级比`||`高（助记：`&&`对应逻辑乘法，`||`对应逻辑加法，乘法比加法优先级要高），下面形式的布尔表达式是不需要加小括弧的：

```go
if 'a' <= c && c <= 'z' ||
    'A' <= c && c <= 'Z' ||
    '0' <= c && c <= '9' {
    // ...ASCII letter or digit...
}
```

布尔值并不会隐式转换为数字值`0`或`1`，反之亦然。必须使用一个显式的`if`语句辅助转换：

```go
i := 0
if b {
    i = 1
}
```

如果需要经常做类似的转换, 包装成一个函数会更方便:

```go
// btoi returns 1 if b is true and 0 if false.
func btoi(b bool) int {
    if b {
        return 1
    }
    return 0
}
```

数字到布尔型的逆转换则非常简单, 不过为了保持对称, 我们也可以包装一个函数:

```go
// itob reports whether i is non-zero.
func itob(i int) bool { return i != 0 }
```

## 3.5. 字符串

一个字符串是一个不可改变的字节序列。字符串可以包含任意的数据，包括`byte`值`0`，但是通常是用来包含人类可读的文本。文本字符串通常被解释为采用UTF8编码的Unicode码点（`rune`）序列，我们稍后会详细讨论这个问题。

内置的`len`函数可以返回一个字符串中的字节数目（不是`rune`字符数目），索引操作`s[i]`返回第`i`个字节的字节值，`i`必须满足`0 ≤ i< len(s)`条件约束。

```go
s := "hello, world"
fmt.Println(len(s))     // "12"
fmt.Println(s[0], s[7]) // "104 119" ('h' and 'w')
```

如果试图访问超出字符串索引范围的字节将会导致`panic`异常：

```go
c := s[len(s)] // panic: index out of range
```

第`i`个字节并不一定是字符串的第`i`个字符，因为对于非ASCII字符的UTF8编码会要两个或多个字节。我们先简单说下字符的工作方式。

子字符串操作`s[i:j]`基于原始的`s`字符串的第`i`个字节开始到第`j`个字节（并不包含`j`本身）生成一个新字符串。生成的新字符串将包含`j-i`个字节。

```go
fmt.Println(s[0:5]) // "hello"
```

同样，如果索引超出字符串范围或者`j`小于`i`的话将导致`panic`异常。

不管`i`还是`j`都可能被忽略，当它们被忽略时将采用`0`作为开始位置，采用`len(s)`作为结束的位置。

```go
fmt.Println(s[:5]) // "hello"
fmt.Println(s[7:]) // "world"
fmt.Println(s[:])  // "hello, world"
```

其中`+`操作符将两个字符串连接构造一个新字符串：

```go
fmt.Println("goodbye" + s[5:]) // "goodbye, world"
```

字符串可以用`==`和`<`进行比较；比较通过逐个字节比较完成的，因此比较的结果是字符串自然编码的顺序。

字符串的值是不可变的：一个字符串包含的字节序列永远不会被改变，当然我们也可以给一个字符串变量分配一个新字符串值。可以像下面这样将一个字符串追加到另一个字符串：

```go
s := "left foot"
t := s
s += ", right foot"
```

这并不会导致原始的字符串值被改变，但是变量`s`将因为`+=`语句持有一个新的字符串值，但是`t`依然是包含原先的字符串值。

```go
fmt.Println(s) // "left foot, right foot"
fmt.Println(t) // "left foot"
```

因为字符串是不可修改的，因此尝试修改字符串内部数据的操作也是被禁止的：

```go
s[0] = 'L' // compile error: cannot assign to s[0]
```

不变性意味着如果两个字符串共享相同的底层数据的话也是安全的，这使得复制任何长度的字符串代价是低廉的。同样，一个字符串`s`和对应的子字符串切片`s[7:]`的操作也可以安全地共享相同的内存，因此字符串切片操作代价也是低廉的。在这两种情况下都没有必要分配新的内存。 图3.4演示了一个字符串和两个子串共享相同的底层数据。

### 3.5.1. 字符串面值

字符串值也可以用字符串面值方式编写，只要将一系列字节序列包含在双引号内即可：

```go
"Hello, 世界"
```

![字符串 - 图1](images\ch3-04.png)

因为Go语言源文件总是用UTF8编码，并且Go语言的文本字符串也以UTF8编码的方式处理，因此我们可以将Unicode码点也写到字符串面值中。

在一个双引号包含的字符串面值中，可以用以反斜杠`\`开头的转义序列插入任意的数据。下面的换行、回车和制表符等是常见的ASCII控制代码的转义方式：

```go
\a      响铃
\b      退格
\f      换页
\n      换行
\r      回车
\t      制表符
\v      垂直制表符
\'      单引号 (只用在 '\'' 形式的rune符号面值中)
\"      双引号 (只用在 "..." 形式的字符串面值中)
\\      反斜杠
```

可以通过十六进制或八进制转义在字符串面值中包含任意的字节。一个十六进制的转义形式是`\xhh`，其中两个h表示十六进制数字（大写或小写都可以）。一个八进制转义形式是`\ooo`，包含三个八进制的o数字（0到7），但是不能超过`\377`（译注：对应一个字节的范围，十进制为255）。每一个单一的字节表达一个特定的值。稍后我们将看到如何将一个Unicode码点写到字符串面值中。

一个原生的字符串面值形式是`…`，使用反引号代替双引号。在原生的字符串面值中，没有转义操作；全部的内容都是字面的意思，包含退格和换行，因此一个程序中的原生字符串面值可能跨越多行（译注：在原生字符串面值内部是无法直接写字符的，可以用八进制或十六进制转义或`+`连接字符串常量完成）。唯一的特殊处理是会删除回车以保证在所有平台上的值都是一样的，包括那些把回车也放入文本文件的系统（译注：Windows系统会把回车和换行一起放入文本文件中）。

原生字符串面值用于编写正则表达式会很方便，因为正则表达式往往会包含很多反斜杠。原生字符串面值同时被广泛应用于HTML模板、JSON面值、命令行提示信息以及那些需要扩展到多行的场景。

```go
const GoUsage = `Go is a tool for managing Go source code.

Usage:
    go command [arguments]
...`
```

### 3.5.2. Unicode

在很久以前，世界还是比较简单的，起码计算机世界就只有一个ASCII字符集：美国信息交换标准代码。ASCII，更准确地说是美国的ASCII，使用`7bit`来表示`128`个字符：包含英文字母的大小写、数字、各种标点符号和设备控制符。对于早期的计算机程序来说，这些就足够了，但是这也导致了世界上很多其他地区的用户无法直接使用自己的符号系统。随着互联网的发展，混合多种语言的数据变得很常见（译注：比如本身的英文原文或中文翻译都包含了ASCII、中文、日文等多种语言字符）。如何有效处理这些包含了各种语言的丰富多样的文本数据呢？

答案就是使用Unicode（ [http://unicode.org](http://unicode.org/) ），它收集了这个世界上所有的符号系统，包括重音符号和其它变音符号，制表符和回车符，还有很多神秘的符号，每个符号都分配一个唯一的Unicode码点，Unicode码点对应Go语言中的`rune`整数类型（译注：`rune`是`int32`等价类型）。

在第八版本的Unicode标准里收集了超过120,000个字符，涵盖超过100多种语言。这些在计算机程序和数据中是如何体现的呢？通用的表示一个Unicode码点的数据类型是`int32`，也就是Go语言中`rune`对应的类型；它的同义词`rune`符文正是这个意思。

我们可以将一个符文序列表示为一个`int32`序列。这种编码方式叫`UTF-32`或`UCS-4`，每个Unicode码点都使用同样大小的`32bit`来表示。这种方式比较简单统一，但是它会浪费很多存储空间，因为大多数计算机可读的文本是ASCII字符，本来每个ASCII字符只需要`8bit`或`1字节`就能表示。而且即使是常用的字符也远少于65,536个，也就是说用`16bit`编码方式就能表达常用字符。但是，还有其它更好的编码方法吗？

### 3.5.3. UTF-8

UTF8是一个将Unicode码点编码为字节序列的变长编码。UTF8编码是由Go语言之父Ken Thompson和Rob Pike共同发明的，现在已经是Unicode的标准。UTF8编码使用1到4个字节来表示每个Unicode码点，ASCII部分字符只使用1个字节，常用字符部分使用2或3个字节表示。每个符号编码后第一个字节的高端bit位用于表示编码总共有多少个字节。如果第一个字节的高端bit为0，则表示对应7bit的ASCII字符，ASCII字符每个字符依然是一个字节，和传统的ASCII编码兼容。如果第一个字节的高端bit是110，则说明需要2个字节；后续的每个高端bit都以10开头。更大的Unicode码点也是采用类似的策略处理。

```go
0xxxxxxx                             runes 0-127    (ASCII)
110xxxxx 10xxxxxx                    128-2047       (values <128 unused)
1110xxxx 10xxxxxx 10xxxxxx           2048-65535     (values <2048 unused)
11110xxx 10xxxxxx 10xxxxxx 10xxxxxx  65536-0x10ffff (other values unused)
```

变长的编码无法直接通过索引来访问第`n`个字符，但是UTF8编码获得了很多额外的优点。首先UTF8编码比较紧凑，完全兼容ASCII码，并且可以自动同步：它可以通过向前回朔最多3个字节就能确定当前字符编码的开始字节的位置。它也是一个前缀编码，所以当从左向右解码时不会有任何歧义也并不需要向前查看（译注：像GBK之类的编码，如果不知道起点位置则可能会出现歧义）。没有任何字符的编码是其它字符编码的子串，或是其它编码序列的字串，因此搜索一个字符时只要搜索它的字节编码序列即可，不用担心前后的上下文会对搜索结果产生干扰。同时UTF8编码的顺序和Unicode码点的顺序一致，因此可以直接排序UTF8编码序列。同时因为没有嵌入的`NUL(0)`字节，可以很好地兼容那些使用`NUL`作为字符串结尾的编程语言。

Go语言的源文件采用`UTF8`编码，并且Go语言处理`UTF8`编码的文本也很出色。`unicode`包提供了诸多处理`rune`字符相关功能的函数（比如区分字母和数字，或者是字母的大写和小写转换等），`unicode/utf8`包则提供了用于`rune`字符序列的`UTF8`编码和解码的功能。

有很多Unicode字符很难直接从键盘输入，并且还有很多字符有着相似的结构；有一些甚至是不可见的字符（译注：中文和日文就有很多相似但不同的字）。Go语言字符串面值中的Unicode转义字符让我们可以通过Unicode码点输入特殊的字符。有两种形式：`\uhhhh`对应`16bit`的码点值，`\Uhhhhhhhh`对应`32bit`的码点值，其中`h`是一个十六进制数字；一般很少需要使用32bit的形式。每一个对应码点的UTF8编码。例如：下面的字母串面值都表示相同的值：

```go
"世界"
"\xe4\xb8\x96\xe7\x95\x8c"
"\u4e16\u754c"
"\U00004e16\U0000754c"
```

上面三个转义序列都为第一个字符串提供替代写法，但是它们的值都是相同的。

Unicode转义也可以使用在rune字符中。下面三个字符是等价的：

```go
'世' '\u4e16' '\U00004e16'
```

对于小于256的码点值可以写在一个十六进制转义字节中，例如`\x41`对应字符’A’，但是对于更大的码点则必须使用`\u`或`\U`转义形式。因此，`\xe4\xb8\x96`并不是一个合法的rune字符，虽然这三个字节对应一个有效的UTF8编码的码点。

得益于UTF8编码优良的设计，诸多字符串操作都不需要解码操作。我们可以不用解码直接测试一个字符串是否是另一个字符串的前缀：

```go
func HasPrefix(s, prefix string) bool {
    return len(s) >= len(prefix) && s[:len(prefix)] == prefix
}
```

或者是后缀测试：

```go
func HasSuffix(s, suffix string) bool {
    return len(s) >= len(suffix) && s[len(s)-len(suffix):] == suffix
}
```

或者是包含子串测试：

```go
func Contains(s, substr string) bool {
    for i := 0; i < len(s); i++ {
        if HasPrefix(s[i:], substr) {
            return true
        }
    }
    return false
}
```

对于UTF8编码后文本的处理和原始的字节处理逻辑是一样的。但是对应很多其它编码则并不是这样的。（上面的函数都来自`strings`字符串处理包，真实的代码包含了一个用哈希技术优化的`Contains` 实现。）

另一方面，如果我们真的关心每个Unicode字符，我们可以使用其它处理方式。考虑前面的第一个例子中的字符串，它混合了中西两种字符。图3.5展示了它的内存表示形式。字符串包含13个字节，以UTF8形式编码，但是只对应9个Unicode字符：

```go
import "unicode/utf8"
s := "Hello, 世界"
fmt.Println(len(s))                    // "13"
fmt.Println(utf8.RuneCountInString(s)) // "9"
```

为了处理这些真实的字符，我们需要一个UTF8解码器。`unicode/utf8`包提供了该功能，我们可以这样使用：

```go
for i := 0; i < len(s); {
    r, size := utf8.DecodeRuneInString(s[i:])
    fmt.Printf("%d\t%c\n", i, r)
    i += size
}
```

每一次调用`DecodeRuneInString`函数都返回一个`r`和长度，`r`对应字符本身，长度对应`r`采用UTF8编码后的编码字节数目。长度可以用于更新第`i`个字符在字符串中的字节索引位置。但是这种编码方式是笨拙的，我们需要更简洁的语法。幸运的是，Go语言的`range`循环在处理字符串的时候，会自动隐式解码UTF8字符串。下面的循环运行如图3.5所示；需要注意的是对于非ASCII，索引更新的步长将超过1个字节。

![字符串 - 图2](images\ch3-05.png)

```go
for i, r := range "Hello, 世界" {
    fmt.Printf("%d\t%q\t%d\n", i, r, r)
}
```

我们可以使用一个简单的循环来统计字符串中字符的数目，像这样：

```go
n := 0
for _, _ = range s {
    n++
}
```

像其它形式的循环那样，我们也可以忽略不需要的变量：

```go
n := 0
for range s {
    n++
}
```

或者我们可以直接调用`utf8.RuneCountInString(s)`函数。

正如我们前面提到的，文本字符串采用UTF8编码只是一种惯例，但是对于循环的真正字符串并不是一个惯例，这是正确的。如果用于循环的字符串只是一个普通的二进制数据，或者是含有错误编码的UTF8数据，将会发生什么呢？

每一个UTF8字符解码，不管是显式地调用`utf8.DecodeRuneInString`解码或是在`range`循环中隐式地解码，如果遇到一个错误的UTF8编码输入，将生成一个特别的Unicode字符`\uFFFD`，在印刷中这个符号通常是一个黑色六角或钻石形状，里面包含一个白色的问号”?”。当程序遇到这样的一个字符，通常是一个危险信号，说明输入并不是一个完美没有错误的UTF8字符串。

UTF8字符串作为交换格式是非常方便的，但是在程序内部采用`rune`序列可能更方便，因为`rune`大小一致，支持数组索引和方便切割。

将`[]rune`类型转换应用到UTF8编码的字符串，将返回字符串编码的Unicode码点序列：

```go
// "program" in Japanese katakana
s := "プログラム"
fmt.Printf("% x\n", s) // "e3 83 97 e3 83 ad e3 82 b0 e3 83 a9 e3 83 a0"
r := []rune(s)
fmt.Printf("%x\n", r)  // "[30d7 30ed 30b0 30e9 30e0]"
```

（在第一个`Printf`中的`% x`参数用于在每个十六进制数字前插入一个空格。）

如果是将一个`[]rune`类型的Unicode字符`slice`或数组转为`string`，则对它们进行UTF8编码：

```go
fmt.Println(string(r)) // "プログラム"
```

将一个整数转型为字符串意思是生成以只包含对应Unicode码点字符的UTF8字符串：

```go
fmt.Println(string(65))     // "A", not "65"
fmt.Println(string(0x4eac)) // "京"
```

如果对应码点的字符是无效的，则用`\uFFFD`无效字符作为替换：

```go
fmt.Println(string(1234567)) // "?"
```

### 3.5.4. 字符串和Byte切片

标准库中有四个包对字符串处理尤为重要：`bytes`、`strings`、`strconv`和`unicode`包。

`strings`包提供了许多如字符串的查询、替换、比较、截断、拆分和合并等功能。

`bytes`包也提供了很多类似功能的函数，但是针对和字符串有着相同结构的`[]byte`类型。因为字符串是只读的，因此逐步构建字符串会导致很多分配和复制。在这种情况下，使用`bytes.Buffer`类型将会更有效，稍后我们将展示。

`strconv`包提供了布尔型、整型数、浮点数和对应字符串的相互转换，还提供了双引号转义相关的转换。

`unicode`包提供了`IsDigit`、`IsLetter`、`IsUpper`和`IsLower`等类似功能，它们用于给字符分类。每个函数有一个单一的`rune`类型的参数，然后返回一个布尔值。而像`ToUpper`和`ToLower`之类的转换函数将用于`rune`字符的大小写转换。所有的这些函数都是遵循Unicode标准定义的字母、数字等分类规范。`strings`包也有类似的函数，它们是`ToUpper`和`ToLower`，将原始字符串的每个字符都做相应的转换，然后返回新的字符串。

下面例子的`basename`函数灵感源于Unix shell的同名工具。在我们实现的版本中，`basename(s)`将看起来像是系统路径的前缀删除，同时将看似文件类型的后缀名部分删除：

```go
fmt.Println(basename("a/b/c.go")) // "c"
fmt.Println(basename("c.d.go"))   // "c.d"
fmt.Println(basename("abc"))      // "abc"
```

第一个版本并没有使用任何库，全部手工硬编码实现：

*gopl.io/ch3/basename1*

```go
// basename removes directory components and a .suffix.
// e.g., a => a, a.go => a, a/b/c.go => c, a/b.c.go => b.c
func basename(s string) string {
    // Discard last '/' and everything before.
    for i := len(s) - 1; i >= 0; i-- {
        if s[i] == '/' {
            s = s[i+1:]
            break
        }
    }
    // Preserve everything before last '.'.
    for i := len(s) - 1; i >= 0; i-- {
        if s[i] == '.' {
            s = s[:i]
            break
        }
    }
    return s
}
```

这个简化版本使用了`strings.LastIndex`库函数：

*gopl.io/ch3/basename2*

```go
func basename(s string) string {
    slash := strings.LastIndex(s, "/") // -1 if "/" not found
    s = s[slash+1:]
    if dot := strings.LastIndex(s, "."); dot >= 0 {
        s = s[:dot]
    }
    return s
}
```

`path`和`path/filepath`包提供了关于文件路径名更一般的函数操作。使用斜杠分隔路径可以在任何操作系统上工作。斜杠本身不应该用于文件名，但是在其他一些领域可能会用于文件名，例如URL路径组件。相比之下，`path/filepath`包则使用操作系统本身的路径规则，例如POSIX系统使用`/foo/bar`，而Microsoft Windows使用`c:\foo\bar`等。

让我们继续另一个字符串的例子。函数的功能是将一个表示整数值的字符串，每隔三个字符插入一个逗号分隔符，例如“`12345`”处理后成为“`12,345`”。这个版本只适用于整数类型；支持浮点数类型的留作练习。

*gopl.io/ch3/comma*

```go
// comma inserts commas in a non-negative decimal integer string.
func comma(s string) string {
    n := len(s)
    if n <= 3 {
        return s
    }
    return comma(s[:n-3]) + "," + s[n-3:]
}
```

输入`comma`函数的参数是一个字符串。如果输入字符串的长度小于或等于3的话，则不需要插入逗号分隔符。否则，`comma`函数将在最后三个字符前的位置将字符串切割为两个子串并插入逗号分隔符，然后通过递归调用自身来得出前面的子串。

一个字符串是包含只读字节的数组，一旦创建，是不可变的。相比之下，一个字节slice的元素则可以自由地修改。

字符串和字节slice之间可以相互转换：

```go
s := "abc"
b := []byte(s)
s2 := string(b)
```

从概念上讲，一个`[]byte(s)`转换是分配了一个新的字节数组用于保存字符串数据的拷贝，然后引用这个底层的字节数组。编译器的优化可以避免在一些场景下分配和复制字符串数据，但总的来说需要确保在变量`b`被修改的情况下，原始的`s`字符串也不会改变。将一个字节slice转换到字符串的`string(b)`操作则是构造一个字符串拷贝，以确保s2字符串是只读的。

为了避免转换中不必要的内存分配，`bytes`包和`strings`同时提供了许多实用函数。下面是`strings`包中的六个函数：

```go
func Contains(s, substr string) bool
func Count(s, sep string) int
func Fields(s string) []string
func HasPrefix(s, prefix string) bool
func Index(s, sep string) int
func Join(a []string, sep string) string
```

bytes包中也对应的六个函数：

```go
func Contains(b, subslice []byte) bool
func Count(s, sep []byte) int
func Fields(s []byte) [][]byte
func HasPrefix(s, prefix []byte) bool
func Index(s, sep []byte) int
func Join(s [][]byte, sep []byte) []byte
```

它们之间唯一的区别是字符串类型参数被替换成了字节slice类型的参数。

`bytes`包还提供了`Buffer`类型用于字节slice的缓存。一个`Buffer`开始是空的，但是随着`string`、`byte`或`[]byte`等类型数据的写入可以动态增长，一个`bytes.Buffer`变量并不需要初始化，因为零值也是有效的：

*gopl.io/ch3/printints*

```go
// intsToString is like fmt.Sprint(values) but adds commas.
func intsToString(values []int) string {
    var buf bytes.Buffer
    buf.WriteByte('[')
    for i, v := range values {
        if i > 0 {
            buf.WriteString(", ")
        }
        fmt.Fprintf(&buf, "%d", v)
    }
    buf.WriteByte(']')
    return buf.String()
}
func main() {
    fmt.Println(intsToString([]int{1, 2, 3})) // "[1, 2, 3]"
}
```

当向`bytes.Buffer`添加任意字符的UTF8编码时，最好使用`bytes.Buffer`的`WriteRune`方法，但是`WriteByte`方法对于写入类似’`[`‘和’`]`’等ASCII字符则会更加有效。

`bytes.Buffer`类型有着很多实用的功能，我们在第七章讨论接口时将会涉及到，我们将看看如何将它用作一个`I/O`的输入和输出对象，例如当做`Fprintf`的`io.Writer`输出对象，或者当作`io.Reader`类型的输入源对象。

**练习 3.10：** 编写一个非递归版本的comma函数，使用`bytes.Buffer`代替字符串链接操作。

```go
// main.go
package main

import (
	"bytes"
	"fmt"
	"os"
)

func main() {
	for i := 1; i < len(os.Args); i++ {
		fmt.Printf("%s\n", comma(os.Args[i]))
	}
}

// comma inserts commas in a non-negative decimal integer string.
func comma(s string) string {
	b := &bytes.Buffer{}
	pre := len(s) % 3
	// Write the first group of up to 3 digits.
	if pre == 0 {
		pre = 3
	}
	b.WriteString(s[:pre])
	// Deal with the rest.
	for i := pre; i < len(s); i += 3 {
		b.WriteByte(',')
		b.WriteString(s[i : i+3])
	}
	return b.String()
}

```

```go
// main_test.go
package main

import (
	"testing"
)

func TestComma(t *testing.T) {
	tests := []struct {
		s, want string
	}{
		{"1", "1"},
		{"12", "12"},
		{"123", "123"},
		{"1234", "1,234"},
		{"12345", "12,345"},
		{"123456", "123,456"},
		{"1234567", "1,234,567"},
		{"12345678", "12,345,678"},
		{"123456789", "123,456,789"},
		{"1234567890", "1,234,567,890"},
	}
	for _, test := range tests {
		got := comma(test.s)
		if got != test.want {
			t.Errorf("comma(%q), got %q, want %q", test.s, got, test.want)
		}
	}
}
```

**练习 3.11：** 完善comma函数，以支持浮点数处理和一个可选的正负号的处理。

```go
// main.go
package main

import (
	"bytes"
	"fmt"
	"os"
	"strings"
)

func main() {
	for i := 1; i < len(os.Args); i++ {
		fmt.Printf("%s\n", comma(os.Args[i]))
	}
}

func comma(s string) string {
	b := bytes.Buffer{}
	mantissaStart := 0
	if s[0] == '+' || s[0] == '-' {
		b.WriteByte(s[0])
		mantissaStart = 1
	}
	mantissaEnd := strings.Index(s, ".")
	if mantissaEnd == -1 {
		mantissaEnd = len(s)
	}
	mantissa := s[mantissaStart:mantissaEnd]
	pre := len(mantissa) % 3
	if pre > 0 {
		b.Write([]byte(mantissa[:pre]))
		if len(mantissa) > pre {
			b.WriteString(",")
		}
	}
	for i, c := range mantissa[pre:] {
		if i%3 == 0 && i != 0 {
			b.WriteString(",")
		}
		b.WriteRune(c)
	}
	b.WriteString(s[mantissaEnd:])
	return b.String()
}
```

```go
// main_test.go
package main

import (
	"testing"
)

func TestComma(t *testing.T) {
	tests := []struct {
		s, want string
	}{
		{"1", "1"},
		{"12", "12"},
		{"123", "123"},
		{"1234", "1,234"},
		{"12345", "12,345"},
		{"123456", "123,456"},
		{"1234567", "1,234,567"},
		{"12345678", "12,345,678"},
		{"123456789", "123,456,789"},
		{"1234567890", "1,234,567,890"},
		{"1.1234", "1.1234"},
		{"12.1234", "12.1234"},
		{"123.1234", "123.1234"},
		{"1234.1234", "1,234.1234"},
		{"12345.1234", "12,345.1234"},
		{"123456.1234", "123,456.1234"},
		{"1234567.1234", "1,234,567.1234"},
		{"12345678.1234", "12,345,678.1234"},
		{"123456789.1234", "123,456,789.1234"},
		{"1234567890.1234", "1,234,567,890.1234"},
		{"+123456789.1234", "+123,456,789.1234"},
		{"-1234567890.1234", "-1,234,567,890.1234"},
	}
	for _, test := range tests {
		got := comma(test.s)
		if got != test.want {
			t.Errorf("comma(%q), got %q, want %q", test.s, got, test.want)
		}
	}
}

```

**练习 3.12：** 编写一个函数，判断两个字符串是否是相互打乱的，也就是说它们有着相同的字符，但是对应不同的顺序。

```go
// anagram.go
// ex3.12 determines if strings are anagrams of each other.
package anagram

func isAnagram(a, b string) bool {
	aFreq := make(map[rune]int)
	for _, c := range a {
		aFreq[c]++
	}
	bFreq := make(map[rune]int)
	for _, c := range b {
		bFreq[c]++
	}
	for k, v := range aFreq {
		if bFreq[k] != v {
			return false
		}
	}
	for k, v := range bFreq {
		if aFreq[k] != v {
			return false
		}
	}
	return true
}
```

```go
// anagram_test.go
package anagram

import (
	"testing"
)

func TestIsAnagram(t *testing.T) {
	tests := []struct {
		a, b string
		want bool
	}{
		{"aba", "baa", true},
		{"aaa", "baa", false}, // same characters but different frequencies
	}
	for _, test := range tests {
		got := isAnagram(test.a, test.b)
		if got != test.want {
			t.Errorf("isAnagram(%q, %q), got %v, want %v",
				test.a, test.b, got, test.want)
		}
	}
}
```

### 3.5.5. 字符串和数字的转换

除了字符串、字符、字节之间的转换，字符串和数值之间的转换也比较常见。由`strconv`包提供这类转换功能。

将一个整数转为字符串，一种方法是用`fmt.Sprintf`返回一个格式化的字符串；另一个方法是用`strconv.Itoa(“整数到ASCII”)`：

```go
x := 123
y := fmt.Sprintf("%d", x)
fmt.Println(y, strconv.Itoa(x)) // "123 123"
```

`FormatInt`和`FormatUint`函数可以用不同的进制来格式化数字：

```go
fmt.Println(strconv.FormatInt(int64(x), 2)) // "1111011"
```

`fmt.Printf`函数的`%b`、`%d`、`%o`和`%x`等参数提供功能往往比`strconv`包的`Format`函数方便很多，特别是在需要包含有附加额外信息的时候：

```go
s := fmt.Sprintf("x=%b", x) // "x=1111011"
```

如果要将一个字符串解析为整数，可以使用`strconv`包的`Atoi`或`ParseInt`函数，还有用于解析无符号整数的`ParseUint`函数：

```go
x, err := strconv.Atoi("123")             // x is an int
y, err := strconv.ParseInt("123", 10, 64) // base 10, up to 64 bits
```

`ParseInt`函数的第三个参数是用于指定整型数的大小；例如16表示`int16`，0则表示`int`。在任何情况下，返回的结果`y`总是`int64`类型，你可以通过强制类型转换将它转为更小的整数类型。

有时候也会使用`fmt.Scanf`来解析输入的字符串和数字，特别是当字符串和数字混合在一行的时候，它可以灵活处理不完整或不规则的输入。

## 3.6. 常量

常量表达式的值在编译期计算，而不是在运行期。每种常量的潜在类型都是基础类型：`boolean`、`string`或数字。

一个常量的声明语句定义了常量的名字，和变量的声明语法类似，常量的值不可修改，这样可以防止在运行期被意外或恶意的修改。例如，常量比变量更适合用于表达像π之类的数学常数，因为它们的值不会发生变化：

```go
const pi = 3.14159 // approximately; math.Pi is a better approximation
```

和变量声明一样，可以批量声明多个常量；这比较适合声明一组相关的常量：

```go
const (
	e  = 2.71828182845904523536028747135266249775724709369995957496696763
	pi = 3.14159265358979323846264338327950288419716939937510582097494459
)
```

所有常量的运算都可以在编译期完成，这样可以减少运行时的工作，也方便其他编译优化。当操作数是常量时，一些运行时的错误也可以在编译时被发现，例如整数除零、字符串索引越界、任何导致无效浮点数的操作等。

常量间的所有算术运算、逻辑运算和比较运算的结果也是常量，对常量的类型转换操作或以下函数调用都是返回常量结果：`len`、`cap`、`real`、`imag`、`complex`和`unsafe.Sizeof`（§13.1）。

因为它们的值是在编译期就确定的，因此常量可以是构成类型的一部分，例如用于指定数组类型的长度：

```go
const IPv4Len = 4

// parseIPv4 parses an IPv4 address (d.d.d.d).
func parseIPv4(s string) IP {
	var p [IPv4Len]byte
	// ...
}
```

一个常量的声明也可以包含一个类型和一个值，但是如果没有显式指明类型，那么将从右边的表达式推断类型。在下面的代码中，`time.Duration`是一个命名类型，底层类型是`int64`，`time.Minute`是对应类型的常量。下面声明的两个常量都是`time.Duration`类型，可以通过`%T`参数打印类型信息：

```go
const noDelay time.Duration = 0
const timeout = 5 * time.Minute

fmt.Printf("%T %[1]v\n", noDelay)     // "time.Duration 0"
fmt.Printf("%T %[1]v\n", timeout)     // "time.Duration 5m0s"
fmt.Printf("%T %[1]v\n", time.Minute) // "time.Duration 1m0s"
```

如果是批量声明的常量，除了第一个外其它的常量右边的初始化表达式都可以省略，如果省略初始化表达式则表示使用前面常量的初始化表达式写法，对应的常量类型也一样的。例如：

```go
const (
	a = 1
	b
	c = 2
	d
)

fmt.Println(a, b, c, d) // "1 1 2 2"
```

如果只是简单地复制右边的常量表达式，其实并没有太实用的价值。但是它可以带来其它的特性，那就是`iota`常量生成器语法。

### 3.6.1. iota 常量生成器

常量声明可以使用`iota`常量生成器初始化，它用于生成一组以相似规则初始化的常量，但是不用每行都写一遍初始化表达式。在一个`const`声明语句中，在第一个声明的常量所在的行，`iota`将会被置为`0`，然后在每一个有常量声明的行加一。

下面是来自`time`包的例子，它首先定义了一个`Weekday`命名类型，然后为一周的每天定义了一个常量，从周日`0`开始。在其它编程语言中，这种类型一般被称为枚举类型。

```go
type Weekday int

const (
	Sunday Weekday = iota
	Monday
	Tuesday
	Wednesday
	Thursday
	Friday
	Saturday
)
```

周日将对应`0`，周一为`1`，如此等等。

我们也可以在复杂的常量表达式中使用`iota`，下面是来自`net`包的例子，用于给一个无符号整数的最低`5`bit的每个bit指定一个名字：

```go
type Flags uint

const (
	FlagUp Flags = 1 << iota // is up
	FlagBroadcast            // supports broadcast access capability
	FlagLoopback             // is a loopback interface
	FlagPointToPoint         // belongs to a point-to-point link
	FlagMulticast            // supports multicast access capability
)
```

随着`iota`的递增，每个常量对应表达式`1 << iota`，是连续的`2`的幂，分别对应一个bit位置。使用这些常量可以用于测试、设置或清除对应的bit位的值：

*gopl.io/ch3/netflag*

```go
func IsUp(v Flags) bool     { return v&FlagUp == FlagUp }
func TurnDown(v *Flags)     { *v &^= FlagUp }
func SetBroadcast(v *Flags) { *v |= FlagBroadcast }
func IsCast(v Flags) bool   { return v&(FlagBroadcast|FlagMulticast) != 0 }

func main() {
	var v Flags = FlagMulticast | FlagUp
	fmt.Printf("%b %t\n", v, IsUp(v)) // "10001 true"
	TurnDown(&v)
	fmt.Printf("%b %t\n", v, IsUp(v)) // "10000 false"
	SetBroadcast(&v)
	fmt.Printf("%b %t\n", v, IsUp(v))   // "10010 false"
	fmt.Printf("%b %t\n", v, IsCast(v)) // "10010 true"
}
```

下面是一个更复杂的例子，每个常量都是1024的幂：

```go
const (
	_ = 1 << (10 * iota)
	KiB // 1024
	MiB // 1048576
	GiB // 1073741824
	TiB // 1099511627776             (exceeds 1 << 32)
	PiB // 1125899906842624
	EiB // 1152921504606846976
	ZiB // 1180591620717411303424    (exceeds 1 << 64)
	YiB // 1208925819614629174706176
)
```

不过`iota`常量生成规则也有其局限性。例如，它并不能用于产生1000的幂（KB、MB等），因为Go语言并没有计算幂的运算符。

**练习 3.13：** 编写KB、MB的常量声明，然后扩展到YB。

```go
package byteconst

const (
	KB = 1000
	MB = 1000 * KB
	GB = 1000 * MB
	PB = 1000 * GB
)
```

### 3.6.2. 无类型常量

Go语言的常量有个不同寻常之处。虽然一个常量可以有任意一个确定的基础类型，例如`int`或`float64`，或者是类似`time.Duration`这样命名的基础类型，但是许多常量并没有一个明确的基础类型。编译器为这些没有明确基础类型的数字常量提供比基础类型更高精度的算术运算；你可以认为至少有`256bit`的运算精度。这里有六种未明确类型的常量类型，分别是

* 无类型的布尔型、
* 无类型的整数、
* 无类型的字符、
* 无类型的浮点数、
* 无类型的复数、
* 无类型的字符串。

通过延迟明确常量的具体类型，无类型的常量不仅可以提供更高的运算精度，而且可以直接用于更多的表达式而不需要显式的类型转换。例如，例子中的`ZiB`和`YiB`的值已经超出任何Go语言中整数类型能表达的范围，但是它们依然是合法的常量，而且像下面的常量表达式依然有效（译注：`YiB/ZiB`是在编译期计算出来的，并且结果常量是1024，是Go语言`int`变量能有效表示的）：

```go
fmt.Println(YiB/ZiB) // "1024"
```

另一个例子，`math.Pi`无类型的浮点数常量，可以直接用于任意需要浮点数或复数的地方：

```go
var x float32 = math.Pi
var y float64 = math.Pi
var z complex128 = math.Pi
```

如果`math.Pi`被确定为特定类型，比如`float64`，那么结果精度可能会不一样，同时对于需要`float32`或`complex128`类型值的地方则会强制需要一个明确的类型转换：

```go
const Pi64 float64 = math.Pi

var x float32 = float32(Pi64)
var y float64 = Pi64
var z complex128 = complex128(Pi64)
```

对于常量面值，不同的写法可能会对应不同的类型。例如`0`、`0.0`、`0i`和`\u0000`虽然有着相同的常量值，但是它们分别对应无类型的整数、无类型的浮点数、无类型的复数和无类型的字符等不同的常量类型。同样，`true`和`false`也是无类型的布尔类型，字符串面值常量是无类型的字符串类型。

前面说过除法运算符`/`会根据操作数的类型生成对应类型的结果。因此，不同写法的常量除法表达式可能对应不同的结果：

```go
var f float64 = 212

fmt.Println((f - 32) * 5 / 9)     // "100"; (f - 32) * 5 is a float64
fmt.Println(5 / 9 * (f - 32))     // "0";   5/9 is an untyped integer, 0
fmt.Println(5.0 / 9.0 * (f - 32)) // "100"; 5.0/9.0 is an untyped float
```

只有常量可以是无类型的。当一个无类型的常量被赋值给一个变量的时候，就像下面的第一行语句，或者出现在有明确类型的变量声明的右边，如下面的其余三行语句，无类型的常量将会被隐式转换为对应的类型，如果转换合法的话。

```go
var f float64 = 3 + 0i // untyped complex -> float64
f = 2                  // untyped integer -> float64
f = 1e123              // untyped floating-point -> float64
f = 'a'                // untyped rune -> float64
```

上面的语句相当于:

```go
var f float64 = float64(3 + 0i)
f = float64(2)
f = float64(1e123)
f = float64('a')
```

无论是隐式或显式转换，将一种类型转换为另一种类型都要求目标可以表示原始值。对于浮点数和复数，可能会有舍入处理：

```go
const (
	deadbeef = 0xdeadbeef // untyped int with value 3735928559
	a = uint32(deadbeef)  // uint32 with value 3735928559
	b = float32(deadbeef) // float32 with value 3735928576 (rounded up)
	c = float64(deadbeef) // float64 with value 3735928559 (exact)
	d = int32(deadbeef)   // compile error: constant overflows int32
	e = float64(1e309)    // compile error: constant overflows float64
	f = uint(-1)          // compile error: constant underflows uint
)
```

对于一个没有显式类型的变量声明（包括简短变量声明），常量的形式将隐式决定变量的默认类型，就像下面的例子：

```go
i := 0      // untyped integer;        implicit int(0)
r := '\000' // untyped rune;           implicit rune('\000')
f := 0.0    // untyped floating-point; implicit float64(0.0)
c := 0i     // untyped complex;        implicit complex128(0i)
```

注意有一点不同：无类型整数常量转换为`int`，它的内存大小是不确定的，但是无类型浮点数和复数常量则转换为内存大小明确的`float64`和`complex128`。 如果不知道浮点数类型的内存大小是很难写出正确的数值算法的，因此Go语言不存在整型类似的不确定内存大小的浮点数和复数类型。

如果要给变量一个不同的类型，我们必须显式地将无类型的常量转化为所需的类型，或给声明的变量指定明确的类型，像下面例子这样：

```go
var i = int8(0)
var i int8 = 0
```

当尝试将这些无类型的常量转为一个接口值时（见第7章），这些默认类型将显得尤为重要，因为要靠它们明确接口对应的动态类型。

```go
fmt.Printf("%T\n", 0)      // "int"
fmt.Printf("%T\n", 0.0)    // "float64"
fmt.Printf("%T\n", 0i)     // "complex128"
fmt.Printf("%T\n", '\000') // "int32" (rune)
```

现在我们已经讲述了Go语言中全部的基础数据类型。下一步将演示如何用基础数据类型组合成数组或结构体等复杂数据类型，然后构建用于解决实际编程问题的数据结构，这将是第四章的讨论主题。