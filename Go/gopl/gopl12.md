# 第十二章　反射

Go语言提供了一种机制，能够在运行时更新变量和检查它们的值、调用它们的方法和它们支持的内在操作，而不需要在编译时就知道这些变量的具体类型。这种机制被称为反射。反射也可以让我们将类型本身作为第一类的值类型处理。

在本章，我们将探讨Go语言的反射特性，看看它可以给语言增加哪些表达力，以及在两个至关重要的API是如何使用反射机制的：一个是`fmt`包提供的字符串格式化功能，另一个是类似`encoding/json`和`encoding/xml`提供的针对特定协议的编解码功能。对于我们在4.6节中看到过的`text/template`和`html/template`包，它们的实现也是依赖反射技术的。然后，反射是一个复杂的内省技术，不应该随意使用，因此，尽管上面这些包内部都是用反射技术实现的，但是它们自己的API都没有公开反射相关的接口。
## 12.1. 为何需要反射?

有时候我们需要编写一个函数能够处理一类并不满足普通公共接口的类型的值，也可能是因为它们并没有确定的表示方式，或者是在我们设计该函数的时候这些类型可能还不存在。

一个大家熟悉的例子是`fmt.Fprintf`函数提供的字符串格式化处理逻辑，它可以用来对任意类型的值格式化并打印，甚至支持用户自定义的类型。让我们也来尝试实现一个类似功能的函数。为了简单起见，我们的函数只接收一个参数，然后返回和`fmt.Sprint`类似的格式化后的字符串。我们实现的函数名也叫`Sprint`。

我们首先用`switch`类型分支来测试输入参数是否实现了`String`方法，如果是的话就调用该方法。然后继续增加类型测试分支，检查这个值的动态类型是否是`string`、`int`、`bool`等基础类型，并在每种情况下执行相应的格式化操作。

```Go
func Sprint(x interface{}) string {
	type stringer interface {
		String() string
	}
	switch x := x.(type) {
	case stringer:
		return x.String()
	case string:
		return x
	case int:
		return strconv.Itoa(x)
	// ...similar cases for int16, uint32, and so on...
	case bool:
		if x {
			return "true"
		}
		return "false"
	default:
		// array, chan, func, map, pointer, slice, struct
		return "???"
	}
}
```

但是我们如何处理其它类似`[]float64`、`map[string][]string`等类型呢？我们当然可以添加更多的测试分支，但是这些组合类型的数目基本是无穷的。还有如何处理类似`url.Values`这样的具名类型呢？即使类型分支可以识别出底层的基础类型是`map[string][]string`，但是它并不匹配`url.Values`类型，因为它们是两种不同的类型，而且`switch`类型分支也不可能包含每个类似`url.Values`的类型，这会导致对这些库的依赖。

没有办法来检查未知类型的表示方式，我们被卡住了。这就是我们为何需要反射的原因。
## 12.2. reflect.Type 和 reflect.Value

反射是由 `reflect` 包提供的。它定义了两个重要的类型，`Type` 和 `Value`。一个 `Type` 表示一个Go类型。它是一个接口，有许多方法来区分类型以及检查它们的组成部分，例如一个结构体的成员或一个函数的参数等。唯一能反映 `reflect.Type` 实现的是接口的类型描述信息（§7.5），也正是这个实体标识了接口值的动态类型。

函数 `reflect.TypeOf` 接受任意的 `interface{}` 类型，并以 `reflect.Type` 形式返回其动态类型：

```Go
t := reflect.TypeOf(3)  // a reflect.Type
fmt.Println(t.String()) // "int"
fmt.Println(t)          // "int"
```

其中 `TypeOf(3)` 调用将值 3 传给 `interface{}` 参数。回到 7.5节 的将一个具体的值转为接口类型会有一个隐式的接口转换操作，它会创建一个包含两个信息的接口值：操作数的动态类型（这里是 `int`）和它的动态的值（这里是 3）。

因为 `reflect.TypeOf` 返回的是一个动态类型的接口值，它总是返回具体的类型。因此，下面的代码将打印 "`*os.File`" 而不是 "`io.Writer`"。稍后，我们将看到能够表达接口类型的 `reflect.Type`。

```Go
var w io.Writer = os.Stdout
fmt.Println(reflect.TypeOf(w)) // "*os.File"
```

要注意的是 `reflect.Type` 接口是满足 `fmt.Stringer` 接口的。因为打印一个接口的动态类型对于调试和日志是有帮助的， `fmt.Printf` 提供了一个缩写 `%T` 参数，内部使用 `reflect.TypeOf` 来输出：

```Go
fmt.Printf("%T\n", 3) // "int"
```

`reflect` 包中另一个重要的类型是 `Value`。一个 `reflect.Value` 可以装载任意类型的值。函数 `reflect.ValueOf` 接受任意的 `interface{}` 类型，并返回一个装载着其动态值的 `reflect.Value`。和 `reflect.TypeOf` 类似，`reflect.ValueOf` 返回的结果也是具体的类型，但是 `reflect.Value` 也可以持有一个接口值。

```Go
v := reflect.ValueOf(3) // a reflect.Value
fmt.Println(v)          // "3"
fmt.Printf("%v\n", v)   // "3"
fmt.Println(v.String()) // NOTE: "<int Value>"
```

和 `reflect.Type` 类似，`reflect.Value` 也满足 `fmt.Stringer` 接口，但是除非 `Value` 持有的是字符串，否则 `String` 方法只返回其类型。而使用 `fmt` 包的 `%v` 标志参数会对 `reflect.Values` 特殊处理。

对 `Value` 调用 `Type` 方法将返回具体类型所对应的 `reflect.Type`：

```Go
t := v.Type()           // a reflect.Type
fmt.Println(t.String()) // "int"
```

`reflect.ValueOf` 的逆操作是 `reflect.Value.Interface` 方法。它返回一个 `interface{}` 类型，装载着与 `reflect.Value` 相同的具体值：

```Go
v := reflect.ValueOf(3) // a reflect.Value
x := v.Interface()      // an interface{}
i := x.(int)            // an int
fmt.Printf("%d\n", i)   // "3"
```

`reflect.Value` 和 `interface{}` 都能装载任意的值。所不同的是，一个空的接口隐藏了值内部的表示方式和所有方法，因此只有我们知道具体的动态类型才能使用类型断言来访问内部的值（就像上面那样），内部值我们没法访问。相比之下，一个 `Value` 则有很多方法来检查其内容，无论它的具体类型是什么。让我们再次尝试实现我们的格式化函数 `format.Any`。

我们使用 `reflect.Value` 的 `Kind` 方法来替代之前的类型 `switch`。虽然还是有无穷多的类型，但是它们的 `kinds` 类型却是有限的：`Bool`、`String` 和 所有数字类型的基础类型；`Array` 和 `Struct` 对应的聚合类型；`Chan`、`Func`、`Ptr`、`Slice` 和 `Map` 对应的引用类型；`interface` 类型；还有表示空值的 `Invalid` 类型。（空的 `reflect.Value` 的 `kind` 即为 `Invalid`。）

<u><i>gopl.io/ch12/format</i></u>
```Go
package format

import (
	"reflect"
	"strconv"
)

// Any formats any value as a string.
func Any(value interface{}) string {
	return formatAtom(reflect.ValueOf(value))
}

// formatAtom formats a value without inspecting its internal structure.
func formatAtom(v reflect.Value) string {
	switch v.Kind() {
	case reflect.Invalid:
		return "invalid"
	case reflect.Int, reflect.Int8, reflect.Int16,
		reflect.Int32, reflect.Int64:
		return strconv.FormatInt(v.Int(), 10)
	case reflect.Uint, reflect.Uint8, reflect.Uint16,
		reflect.Uint32, reflect.Uint64, reflect.Uintptr:
		return strconv.FormatUint(v.Uint(), 10)
	// ...floating-point and complex cases omitted for brevity...
	case reflect.Bool:
		return strconv.FormatBool(v.Bool())
	case reflect.String:
		return strconv.Quote(v.String())
	case reflect.Chan, reflect.Func, reflect.Ptr, reflect.Slice, reflect.Map:
		return v.Type().String() + " 0x" +
			strconv.FormatUint(uint64(v.Pointer()), 16)
	default: // reflect.Array, reflect.Struct, reflect.Interface
		return v.Type().String() + " value"
	}
}
```

到目前为止，我们的函数将每个值视作一个不可分割没有内部结构的物品，因此它叫 `formatAtom`。对于聚合类型（结构体和数组）和接口，只是打印值的类型，对于引用类型（`channels`、`functions`、`pointers`、`slices` 和 `maps`），打印类型和十六进制的引用地址。虽然还不够理想，但是依然是一个重大的进步，并且 `Kind` 只关心底层表示，`format.Any` 也支持具名类型。例如：

```Go
var x int64 = 1
var d time.Duration = 1 * time.Nanosecond
fmt.Println(format.Any(x))                  // "1"
fmt.Println(format.Any(d))                  // "1"
fmt.Println(format.Any([]int64{x}))         // "[]int64 0x8202b87b0"
fmt.Println(format.Any([]time.Duration{d})) // "[]time.Duration 0x8202b87e0"
```
## 12.3. Display，一个递归的值打印器

接下来，让我们看看如何改善聚合数据类型的显示。我们并不想完全克隆一个`fmt.Sprint`函数，我们只是构建一个用于调试用的`Display`函数：给定任意一个复杂类型 `x`，打印这个值对应的完整结构，同时标记每个元素的发现路径。让我们从一个例子开始。

```Go
e, _ := eval.Parse("sqrt(A / pi)")
Display("e", e)
```

在上面的调用中，传入`Display`函数的参数是在7.9节一个表达式求值函数返回的语法树。`Display`函数的输出如下：

```Go
Display e (eval.call):
e.fn = "sqrt"
e.args[0].type = eval.binary
e.args[0].value.op = 47
e.args[0].value.x.type = eval.Var
e.args[0].value.x.value = "A"
e.args[0].value.y.type = eval.Var
e.args[0].value.y.value = "pi"
```

你应该尽量避免在一个包的API中暴露涉及反射的接口。我们将定义一个未导出的display函数用于递归处理工作，导出的是Display函数，它只是display函数简单的包装以接受interface{}类型的参数：

`gopl.io/ch12/display`

```Go
func Display(name string, x interface{}) {
	fmt.Printf("Display %s (%T):\n", name, x)
	display(name, reflect.ValueOf(x))
}
```

在display函数中，我们使用了前面定义的打印基础类型——基本类型、函数和chan等——元素值的formatAtom函数，但是我们会使用`reflect.Value`的方法来递归显示复杂类型的每一个成员。在递归下降过程中，path字符串，从最开始传入的起始值（这里是“e”），将逐步增长来表示是如何达到当前值（例如“`e.args[0].value`”）的。

因为我们不再模拟`fmt.Sprint`函数，我们将直接使用`fmt`包来简化我们的例子实现。

```Go
func display(path string, v reflect.Value) {
	switch v.Kind() {
	case reflect.Invalid:
		fmt.Printf("%s = invalid\n", path)
	case reflect.Slice, reflect.Array:
		for i := 0; i < v.Len(); i++ {
			display(fmt.Sprintf("%s[%d]", path, i), v.Index(i))
		}
	case reflect.Struct:
		for i := 0; i < v.NumField(); i++ {
			fieldPath := fmt.Sprintf("%s.%s", path, v.Type().Field(i).Name)
			display(fieldPath, v.Field(i))
		}
	case reflect.Map:
		for _, key := range v.MapKeys() {
			display(fmt.Sprintf("%s[%s]", path,
				formatAtom(key)), v.MapIndex(key))
		}
	case reflect.Ptr:
		if v.IsNil() {
			fmt.Printf("%s = nil\n", path)
		} else {
			display(fmt.Sprintf("(*%s)", path), v.Elem())
		}
	case reflect.Interface:
		if v.IsNil() {
			fmt.Printf("%s = nil\n", path)
		} else {
			fmt.Printf("%s.type = %s\n", path, v.Elem().Type())
			display(path+".value", v.Elem())
		}
	default: // basic types, channels, funcs
		fmt.Printf("%s = %s\n", path, formatAtom(v))
	}
}
```

让我们针对不同类型分别讨论。

**Slice和数组：** 两种的处理逻辑是一样的。Len方法返回slice或数组值中的元素个数，Index(i)获得索引i对应的元素，返回的也是一个reflect.Value；如果索引i超出范围的话将导致panic异常，这与数组或slice类型内建的len(a)和a[i]操作类似。display针对序列中的每个元素递归调用自身处理，我们通过在递归处理时向path附加“[i]”来表示访问路径。

虽然reflect.Value类型带有很多方法，但是只有少数的方法能对任意值都安全调用。例如，Index方法只能对Slice、数组或字符串类型的值调用，如果对其它类型调用则会导致panic异常。

**结构体：** NumField方法报告结构体中成员的数量，Field(i)以reflect.Value类型返回第i个成员的值。成员列表也包括通过匿名字段提升上来的成员。为了在path添加“.f”来表示成员路径，我们必须获得结构体对应的reflect.Type类型信息，然后访问结构体第i个成员的名字。

**Maps:** MapKeys方法返回一个reflect.Value类型的slice，每一个元素对应map的一个key。和往常一样，遍历map时顺序是随机的。MapIndex(key)返回map中key对应的value。我们向path添加“[key]”来表示访问路径。（我们这里有一个未完成的工作。其实map的key的类型并不局限于formatAtom能完美处理的类型；数组、结构体和接口都可以作为map的key。针对这种类型，完善key的显示信息是练习12.1的任务。）

**指针：** Elem方法返回指针指向的变量，依然是reflect.Value类型。即使指针是nil，这个操作也是安全的，在这种情况下指针是Invalid类型，但是我们可以用IsNil方法来显式地测试一个空指针，这样我们可以打印更合适的信息。我们在path前面添加“*”，并用括弧包含以避免歧义。

**接口：** 再一次，我们使用IsNil方法来测试接口是否是nil，如果不是，我们可以调用v.Elem()来获取接口对应的动态值，并且打印对应的类型和值。

现在我们的Display函数总算完工了，让我们看看它的表现吧。下面的Movie类型是在4.5节的电影类型上演变来的：

```Go
type Movie struct {
	Title, Subtitle string
	Year            int
	Color           bool
	Actor           map[string]string
	Oscars          []string
	Sequel          *string
}
```

让我们声明一个该类型的变量，然后看看Display函数如何显示它：

```Go
strangelove := Movie{
	Title:    "Dr. Strangelove",
	Subtitle: "How I Learned to Stop Worrying and Love the Bomb",
	Year:     1964,
	Color:    false,
	Actor: map[string]string{
		"Dr. Strangelove":            "Peter Sellers",
		"Grp. Capt. Lionel Mandrake": "Peter Sellers",
		"Pres. Merkin Muffley":       "Peter Sellers",
		"Gen. Buck Turgidson":        "George C. Scott",
		"Brig. Gen. Jack D. Ripper":  "Sterling Hayden",
		`Maj. T.J. "King" Kong`:      "Slim Pickens",
	},

	Oscars: []string{
		"Best Actor (Nomin.)",
		"Best Adapted Screenplay (Nomin.)",
		"Best Director (Nomin.)",
		"Best Picture (Nomin.)",
	},
}
```

Display("strangelove", strangelove)调用将显示（strangelove电影对应的中文名是《奇爱博士》）：

```Go
Display strangelove (display.Movie):
strangelove.Title = "Dr. Strangelove"
strangelove.Subtitle = "How I Learned to Stop Worrying and Love the Bomb"
strangelove.Year = 1964
strangelove.Color = false
strangelove.Actor["Gen. Buck Turgidson"] = "George C. Scott"
strangelove.Actor["Brig. Gen. Jack D. Ripper"] = "Sterling Hayden"
strangelove.Actor["Maj. T.J. \"King\" Kong"] = "Slim Pickens"
strangelove.Actor["Dr. Strangelove"] = "Peter Sellers"
strangelove.Actor["Grp. Capt. Lionel Mandrake"] = "Peter Sellers"
strangelove.Actor["Pres. Merkin Muffley"] = "Peter Sellers"
strangelove.Oscars[0] = "Best Actor (Nomin.)"
strangelove.Oscars[1] = "Best Adapted Screenplay (Nomin.)"
strangelove.Oscars[2] = "Best Director (Nomin.)"
strangelove.Oscars[3] = "Best Picture (Nomin.)"
strangelove.Sequel = nil
```

我们也可以使用Display函数来显示标准库中类型的内部结构，例如`*os.File`类型：

```Go
Display("os.Stderr", os.Stderr)
// Output:
// Display os.Stderr (*os.File):
// (*(*os.Stderr).file).fd = 2
// (*(*os.Stderr).file).name = "/dev/stderr"
// (*(*os.Stderr).file).nepipe = 0
```

可以看出，反射能够访问到结构体中未导出的成员。需要当心的是这个例子的输出在不同操作系统上可能是不同的，并且随着标准库的发展也可能导致结果不同。（这也是将这些成员定义为私有成员的原因之一！）我们甚至可以用Display函数来显示reflect.Value 的内部构造（在这里设置为`*os.File`的类型描述体）。`Display("rV", reflect.ValueOf(os.Stderr))`调用的输出如下，当然不同环境得到的结果可能有差异：

```Go
Display rV (reflect.Value):
(*rV.typ).size = 8
(*rV.typ).hash = 871609668
(*rV.typ).align = 8
(*rV.typ).fieldAlign = 8
(*rV.typ).kind = 22
(*(*rV.typ).string) = "*os.File"

(*(*(*rV.typ).uncommonType).methods[0].name) = "Chdir"
(*(*(*(*rV.typ).uncommonType).methods[0].mtyp).string) = "func() error"
(*(*(*(*rV.typ).uncommonType).methods[0].typ).string) = "func(*os.File) error"
...
```

观察下面两个例子的区别：

```Go
var i interface{} = 3

Display("i", i)
// Output:
// Display i (int):
// i = 3

Display("&i", &i)
// Output:
// Display &i (*interface {}):
// (*&i).type = int
// (*&i).value = 3
```

在第一个例子中，Display函数调用`reflect.ValueOf(i)`，它返回一个`Int`类型的值。正如我们在12.2节中提到的，`reflect.ValueO`f总是返回一个具体类型的 `Value`，因为它是从一个接口值提取的内容。

在第二个例子中，`Display`函数调用的是`reflect.ValueOf(&i)`，它返回一个指向`i`的指针，对应`Ptr`类型。在`switch`的`Ptr`分支中，对这个值调用 `Elem` 方法，返回一个`Value`来表示变量 `i` 本身，对应`Interface`类型。像这样一个间接获得的`Value`，可能代表任意类型的值，包括接口类型。`display`函数递归调用自身，这次它分别打印了这个接口的动态类型和值。

对于目前的实现，如果遇到对象图中含有回环，Display将会陷入死循环，例如下面这个首尾相连的链表：

```Go
// a struct that points to itself
type Cycle struct{ Value int; Tail *Cycle }
var c Cycle
c = Cycle{42, &c}
Display("c", c)
```

Display会永远不停地进行深度递归打印：

```Go
Display c (display.Cycle):
c.Value = 42
(*c.Tail).Value = 42
(*(*c.Tail).Tail).Value = 42
(*(*(*c.Tail).Tail).Tail).Value = 42
...ad infinitum...
```

许多Go语言程序都包含了一些循环的数据。让`Display`支持这类带环的数据结构需要些技巧，需要额外记录迄今访问的路径；相应会带来成本。通用的解决方案是采用 `unsafe` 的语言特性，我们将在13.3节看到具体的解决方案。

带环的数据结构很少会对`fmt.Sprint`函数造成问题，因为它很少尝试打印完整的数据结构。例如，当它遇到一个指针的时候，它只是简单地打印指针的数字值。在打印包含自身的slice或map时可能卡住，但是这种情况很罕见，不值得付出为了处理回环所需的开销。

**练习 12.1：** 扩展Display函数，使它可以显示包含以结构体或数组作为map的key类型的值。

```go
// display.go
package display

import (
	"bytes"
	"fmt"
	"reflect"
	"strconv"
)

func Display(name string, x interface{}) {
	fmt.Printf("Display %s (%T):\n", name, x)
	display(name, reflect.ValueOf(x))
}

// formatAtom formats a value without inspecting its internal structure.
// It is a copy of the the function in gopl.io/ch11/format.
func formatAtom(v reflect.Value) string {
	switch v.Kind() {
	case reflect.Invalid:
		return "invalid"
	case reflect.Int, reflect.Int8, reflect.Int16,
		reflect.Int32, reflect.Int64:
		return strconv.FormatInt(v.Int(), 10)
	case reflect.Uint, reflect.Uint8, reflect.Uint16,
		reflect.Uint32, reflect.Uint64, reflect.Uintptr:
		return strconv.FormatUint(v.Uint(), 10)
	// ...floating-point and complex cases omitted for brevity...
	case reflect.Bool:
		if v.Bool() {
			return "true"
		}
		return "false"
	case reflect.String:
		return strconv.Quote(v.String())
	case reflect.Chan, reflect.Func, reflect.Ptr,
		reflect.Slice, reflect.Map:
		return v.Type().String() + " 0x" +
			strconv.FormatUint(uint64(v.Pointer()), 16)
	default: // reflect.Array, reflect.Struct, reflect.Interface
		return v.Type().String() + " value"
	}
}

// formatMapKey includes special behaviour for struct and array values to
// format one level down, but defers to formatAtom for other types.
func formatMapKey(v reflect.Value) string {
	switch v.Kind() {
	case reflect.Struct:
		b := &bytes.Buffer{}
		b.WriteByte('{')
		for i := 0; i < v.NumField(); i++ {
			if i != 0 {
				b.WriteString(", ")
			}
			fmt.Fprintf(b, "%s: %s", v.Type().Field(i).Name, formatAtom(v.Field(i)))
		}
		b.WriteByte('}')
		return b.String()
	case reflect.Array:
		b := &bytes.Buffer{}
		b.WriteByte('{')
		for i := 0; i < v.Len(); i++ {
			if i != 0 {
				b.WriteString(", ")
			}
			b.WriteString(formatAtom(v.Index(i)))
		}
		b.WriteByte('}')
		return b.String()
	default:
		return formatAtom(v)
	}
}

func display(path string, v reflect.Value) {
	switch v.Kind() {
	case reflect.Invalid:
		fmt.Printf("%s = invalid\n", path)
	case reflect.Slice, reflect.Array:
		for i := 0; i < v.Len(); i++ {
			display(fmt.Sprintf("%s[%d]", path, i), v.Index(i))
		}
	case reflect.Struct:
		for i := 0; i < v.NumField(); i++ {
			fieldPath := fmt.Sprintf("%s.%s", path, v.Type().Field(i).Name)
			display(fieldPath, v.Field(i))
		}
	case reflect.Map:
		for _, key := range v.MapKeys() {
			display(fmt.Sprintf("%s[%s]", path, formatMapKey(key)), v.MapIndex(key))
		}
	case reflect.Ptr:
		if v.IsNil() {
			fmt.Printf("%s = nil\n", path)
		} else {
			display(fmt.Sprintf("(*%s)", path), v.Elem())
		}
	case reflect.Interface:
		if v.IsNil() {
			fmt.Printf("%s = nil\n", path)
		} else {
			fmt.Printf("%s.type = %s\n", path, v.Elem().Type())
			display(path+".value", v.Elem())
		}
	default: // basic types, channels, funcs
		fmt.Printf("%s = %s\n", path, formatAtom(v))
	}
}
```

```go
// display_test.go
package display

import (
	"io"
	"net"
	"os"
	"reflect"
	"sync"
	"testing"

	"gopl.io/ch7/eval"
)

func Example_expr() {
	e, _ := eval.Parse("sqrt(A / pi)")
	Display("e", e)
	// Output:
	// Display e (eval.call):
	// e.fn = "sqrt"
	// e.args[0].type = eval.binary
	// e.args[0].value.op = 47
	// e.args[0].value.x.type = eval.Var
	// e.args[0].value.x.value = "A"
	// e.args[0].value.y.type = eval.Var
	// e.args[0].value.y.value = "pi"
}

func Example_slice() {
	Display("slice", []*int{new(int), nil})
	// Output:
	// Display slice ([]*int):
	// (*slice[0]) = 0
	// slice[1] = nil
}

func Example_nilInterface() {
	var w io.Writer
	Display("w", w)
	// Output:
	// Display w (<nil>):
	// w = invalid
}

func Example_ptrToInterface() {
	var w io.Writer
	Display("&w", &w)
	// Output:
	// Display &w (*io.Writer):
	// (*&w) = nil
}

func Example_struct() {
	Display("x", struct{ x interface{} }{3})
	// Output:
	// Display x (struct { x interface {} }):
	// x.x.type = int
	// x.x.value = 3
}

func Example_interface() {
	var i interface{} = 3
	Display("i", i)
	// Output:
	// Display i (int):
	// i = 3
}

func Example_ptrToInterface2() {
	var i interface{} = 3
	Display("&i", &i)
	// Output:
	// Display &i (*interface {}):
	// (*&i).type = int
	// (*&i).value = 3
}

func Example_array() {
	Display("x", [1]interface{}{3})
	// Output:
	// Display x ([1]interface {}):
	// x[0].type = int
	// x[0].value = 3
}

func Example_movie() {
	//!+movie
	type Movie struct {
		Title, Subtitle string
		Year            int
		Color           bool
		Actor           map[string]string
		Oscars          []string
		Sequel          *string
	}
	//!-movie
	//!+strangelove
	strangelove := Movie{
		Title:    "Dr. Strangelove",
		Subtitle: "How I Learned to Stop Worrying and Love the Bomb",
		Year:     1964,
		Color:    false,
		Actor: map[string]string{
			"Dr. Strangelove":            "Peter Sellers",
			"Grp. Capt. Lionel Mandrake": "Peter Sellers",
			"Pres. Merkin Muffley":       "Peter Sellers",
			"Gen. Buck Turgidson":        "George C. Scott",
			"Brig. Gen. Jack D. Ripper":  "Sterling Hayden",
			`Maj. T.J. "King" Kong`:      "Slim Pickens",
		},

		Oscars: []string{
			"Best Actor (Nomin.)",
			"Best Adapted Screenplay (Nomin.)",
			"Best Director (Nomin.)",
			"Best Picture (Nomin.)",
		},
	}
	//!-strangelove
	Display("strangelove", strangelove)

	// We don't use an Output: comment since displaying
	// a map is nondeterministic.
	/*
		//!+output
		Display strangelove (display.Movie):
		strangelove.Title = "Dr. Strangelove"
		strangelove.Subtitle = "How I Learned to Stop Worrying and Love the Bomb"
		strangelove.Year = 1964
		strangelove.Color = false
		strangelove.Actor["Gen. Buck Turgidson"] = "George C. Scott"
		strangelove.Actor["Brig. Gen. Jack D. Ripper"] = "Sterling Hayden"
		strangelove.Actor["Maj. T.J. \"King\" Kong"] = "Slim Pickens"
		strangelove.Actor["Dr. Strangelove"] = "Peter Sellers"
		strangelove.Actor["Grp. Capt. Lionel Mandrake"] = "Peter Sellers"
		strangelove.Actor["Pres. Merkin Muffley"] = "Peter Sellers"
		strangelove.Oscars[0] = "Best Actor (Nomin.)"
		strangelove.Oscars[1] = "Best Adapted Screenplay (Nomin.)"
		strangelove.Oscars[2] = "Best Director (Nomin.)"
		strangelove.Oscars[3] = "Best Picture (Nomin.)"
		strangelove.Sequel = nil
		//!-output
	*/
}

// This test ensures that the program terminates without crashing.
func Test(t *testing.T) {
	// Some other values (YMMV)
	Display("os.Stderr", os.Stderr)
	// Output:
	// Display os.Stderr (*os.File):
	// (*(*os.Stderr).file).fd = 2
	// (*(*os.Stderr).file).name = "/dev/stderr"
	// (*(*os.Stderr).file).nepipe = 0

	var w io.Writer = os.Stderr
	Display("&w", &w)
	// Output:
	// Display &w (*io.Writer):
	// (*&w).type = *os.File
	// (*(*(*&w).value).file).fd = 2
	// (*(*(*&w).value).file).name = "/dev/stderr"
	// (*(*(*&w).value).file).nepipe = 0

	var locker sync.Locker = new(sync.Mutex)
	Display("(&locker)", &locker)
	// Output:
	// Display (&locker) (*sync.Locker):
	// (*(&locker)).type = *sync.Mutex
	// (*(*(&locker)).value).state = 0
	// (*(*(&locker)).value).sema = 0

	Display("locker", locker)
	// Output:
	// Display locker (*sync.Mutex):
	// (*locker).state = 0
	// (*locker).sema = 0
	// (*(&locker)) = nil

	locker = nil
	Display("(&locker)", &locker)
	// Output:
	// Display (&locker) (*sync.Locker):
	// (*(&locker)) = nil

	ips, _ := net.LookupHost("golang.org")
	Display("ips", ips)
	// Output:
	// Display ips ([]string):
	// ips[0] = "173.194.68.141"
	// ips[1] = "2607:f8b0:400d:c06::8d"

	// Even metarecursion!  (YMMV)
	Display("rV", reflect.ValueOf(os.Stderr))
	// Output:
	// Display rV (reflect.Value):
	// (*rV.typ).size = 8
	// (*rV.typ).ptrdata = 8
	// (*rV.typ).hash = 871609668
	// (*rV.typ)._ = 0
	// ...

	// a pointer that points to itself
	type P *P
	var p P
	p = &p
	if false {
		Display("p", p)
		// Output:
		// Display p (display.P):
		// ...stuck, no output...
	}

	// a map that contains itself
	type M map[string]M
	m := make(M)
	m[""] = m
	if false {
		Display("m", m)
		// Output:
		// Display m (display.M):
		// ...stuck, no output...
	}

	// a slice that contains itself
	type S []S
	s := make(S, 1)
	s[0] = s
	if false {
		Display("s", s)
		// Output:
		// Display s (display.S):
		// ...stuck, no output...
	}

	// a linked list that eats its own tail
	type Cycle struct {
		Value int
		Tail  *Cycle
	}
	var c Cycle
	c = Cycle{42, &c}
	if false {
		Display("c", c)
		// Output:
		// Display c (display.Cycle):
		// c.Value = 42
		// (*c.Tail).Value = 42
		// (*(*c.Tail).Tail).Value = 42
		// ...ad infinitum...
	}
}

func TestMapKeys(t *testing.T) {
	sm := map[struct{ x int }]int{
		{1}: 2,
		{2}: 3,
	}
	Display("sm", sm)
	// Output:
	// Display sm (map[struct { x int }]int):
	// sm[{x: 2}] = 3
	// sm[{x: 1}] = 2

	am := map[[3]int]int{
		{1, 2, 3}: 3,
		{2, 3, 4}: 4,
	}
	Display("am", am)
	// Output:
	// Display am (map[[3]int]int):
	// am[{1, 2, 3}] = 3
	// am[{2, 3, 4}] = 4
}
```

**练习 12.2：** 增强display函数的稳健性，通过记录边界的步数来确保在超出一定限制后放弃递归。（在13.3节，我们会看到另一种探测数据结构是否存在环的技术。）

```go
// display.go
package display

import (
	"bytes"
	"fmt"
	"reflect"
	"strconv"
)

func Display(name string, x interface{}) {
	fmt.Printf("Display %s (%T):\n", name, x)
	display(name, reflect.ValueOf(x), 0)
}

// formatAtom formats a value without inspecting its internal structure.
// It is a copy of the the function in gopl.io/ch11/format.
func formatAtom(v reflect.Value) string {
	switch v.Kind() {
	case reflect.Invalid:
		return "invalid"
	case reflect.Int, reflect.Int8, reflect.Int16,
		reflect.Int32, reflect.Int64:
		return strconv.FormatInt(v.Int(), 10)
	case reflect.Uint, reflect.Uint8, reflect.Uint16,
		reflect.Uint32, reflect.Uint64, reflect.Uintptr:
		return strconv.FormatUint(v.Uint(), 10)
	// ...floating-point and complex cases omitted for brevity...
	case reflect.Bool:
		if v.Bool() {
			return "true"
		}
		return "false"
	case reflect.String:
		return strconv.Quote(v.String())
	case reflect.Chan, reflect.Func, reflect.Ptr,
		reflect.Slice, reflect.Map:
		return v.Type().String() + " 0x" +
			strconv.FormatUint(uint64(v.Pointer()), 16)
	default: // reflect.Array, reflect.Struct, reflect.Interface
		return v.Type().String() + " value"
	}
}

// formatMapKey includes special behaviour for struct and array values to
// format one level down, but defers to formatAtom for other types.
func formatMapKey(v reflect.Value) string {
	switch v.Kind() {
	case reflect.Struct:
		b := &bytes.Buffer{}
		b.WriteByte('{')
		for i := 0; i < v.NumField(); i++ {
			if i != 0 {
				b.WriteString(", ")
			}
			fmt.Fprintf(b, "%s: %s", v.Type().Field(i).Name, formatAtom(v.Field(i)))
		}
		b.WriteByte('}')
		return b.String()
	case reflect.Array:
		b := &bytes.Buffer{}
		b.WriteByte('{')
		for i := 0; i < v.Len(); i++ {
			if i != 0 {
				b.WriteString(", ")
			}
			b.WriteString(formatAtom(v.Index(i)))
		}
		b.WriteByte('}')
		return b.String()
	default:
		return formatAtom(v)
	}
}

func display(path string, v reflect.Value, level int) {
	if level > 5 {
		fmt.Printf("%s = %s\n", path, formatAtom(v))
		return
	}
	switch v.Kind() {
	case reflect.Invalid:
		fmt.Printf("%s = invalid\n", path)
	case reflect.Slice, reflect.Array:
		for i := 0; i < v.Len(); i++ {
			display(fmt.Sprintf("%s[%d]", path, i), v.Index(i), level+1)
		}
	case reflect.Struct:
		for i := 0; i < v.NumField(); i++ {
			fieldPath := fmt.Sprintf("%s.%s", path, v.Type().Field(i).Name)
			display(fieldPath, v.Field(i), level+1)
		}
	case reflect.Map:
		for _, key := range v.MapKeys() {
			display(fmt.Sprintf("%s[%s]", path, formatMapKey(key)), v.MapIndex(key), level+1)
		}
	case reflect.Ptr:
		if v.IsNil() {
			fmt.Printf("%s = nil\n", path)
		} else {
			display(fmt.Sprintf("(*%s)", path), v.Elem(), level+1)
		}
	case reflect.Interface:
		if v.IsNil() {
			fmt.Printf("%s = nil\n", path)
		} else {
			fmt.Printf("%s.type = %s\n", path, v.Elem().Type())
			display(path+".value", v.Elem(), level+1)
		}
	default: // basic types, channels, funcs
		fmt.Printf("%s = %s\n", path, formatAtom(v))
	}
}
```

```go
// display_test.go
package display

import (
	"io"
	"net"
	"os"
	"reflect"
	"sync"
	"testing"

	"gopl.io/ch7/eval"
)

func Example_expr() {
	e, _ := eval.Parse("sqrt(A / pi)")
	Display("e", e)
	// Output:
	// Display e (eval.call):
	// e.fn = "sqrt"
	// e.args[0].type = eval.binary
	// e.args[0].value.op = 47
	// e.args[0].value.x.type = eval.Var
	// e.args[0].value.x.value = "A"
	// e.args[0].value.y.type = eval.Var
	// e.args[0].value.y.value = "pi"
}

func Example_slice() {
	Display("slice", []*int{new(int), nil})
	// Output:
	// Display slice ([]*int):
	// (*slice[0]) = 0
	// slice[1] = nil
}

func Example_nilInterface() {
	var w io.Writer
	Display("w", w)
	// Output:
	// Display w (<nil>):
	// w = invalid
}

func Example_ptrToInterface() {
	var w io.Writer
	Display("&w", &w)
	// Output:
	// Display &w (*io.Writer):
	// (*&w) = nil
}

func Example_struct() {
	Display("x", struct{ x interface{} }{3})
	// Output:
	// Display x (struct { x interface {} }):
	// x.x.type = int
	// x.x.value = 3
}

func Example_interface() {
	var i interface{} = 3
	Display("i", i)
	// Output:
	// Display i (int):
	// i = 3
}

func Example_ptrToInterface2() {
	var i interface{} = 3
	Display("&i", &i)
	// Output:
	// Display &i (*interface {}):
	// (*&i).type = int
	// (*&i).value = 3
}

func Example_array() {
	Display("x", [1]interface{}{3})
	// Output:
	// Display x ([1]interface {}):
	// x[0].type = int
	// x[0].value = 3
}

func Example_movie() {
	//!+movie
	type Movie struct {
		Title, Subtitle string
		Year            int
		Color           bool
		Actor           map[string]string
		Oscars          []string
		Sequel          *string
	}
	//!-movie
	//!+strangelove
	strangelove := Movie{
		Title:    "Dr. Strangelove",
		Subtitle: "How I Learned to Stop Worrying and Love the Bomb",
		Year:     1964,
		Color:    false,
		Actor: map[string]string{
			"Dr. Strangelove":            "Peter Sellers",
			"Grp. Capt. Lionel Mandrake": "Peter Sellers",
			"Pres. Merkin Muffley":       "Peter Sellers",
			"Gen. Buck Turgidson":        "George C. Scott",
			"Brig. Gen. Jack D. Ripper":  "Sterling Hayden",
			`Maj. T.J. "King" Kong`:      "Slim Pickens",
		},

		Oscars: []string{
			"Best Actor (Nomin.)",
			"Best Adapted Screenplay (Nomin.)",
			"Best Director (Nomin.)",
			"Best Picture (Nomin.)",
		},
	}
	//!-strangelove
	Display("strangelove", strangelove)

	// We don't use an Output: comment since displaying
	// a map is nondeterministic.
	/*
		//!+output
		Display strangelove (display.Movie):
		strangelove.Title = "Dr. Strangelove"
		strangelove.Subtitle = "How I Learned to Stop Worrying and Love the Bomb"
		strangelove.Year = 1964
		strangelove.Color = false
		strangelove.Actor["Gen. Buck Turgidson"] = "George C. Scott"
		strangelove.Actor["Brig. Gen. Jack D. Ripper"] = "Sterling Hayden"
		strangelove.Actor["Maj. T.J. \"King\" Kong"] = "Slim Pickens"
		strangelove.Actor["Dr. Strangelove"] = "Peter Sellers"
		strangelove.Actor["Grp. Capt. Lionel Mandrake"] = "Peter Sellers"
		strangelove.Actor["Pres. Merkin Muffley"] = "Peter Sellers"
		strangelove.Oscars[0] = "Best Actor (Nomin.)"
		strangelove.Oscars[1] = "Best Adapted Screenplay (Nomin.)"
		strangelove.Oscars[2] = "Best Director (Nomin.)"
		strangelove.Oscars[3] = "Best Picture (Nomin.)"
		strangelove.Sequel = nil
		//!-output
	*/
}

// This test ensures that the program terminates without crashing.
func Test(t *testing.T) {
	// Some other values (YMMV)
	Display("os.Stderr", os.Stderr)
	// Output:
	// Display os.Stderr (*os.File):
	// (*(*os.Stderr).file).fd = 2
	// (*(*os.Stderr).file).name = "/dev/stderr"
	// (*(*os.Stderr).file).nepipe = 0

	var w io.Writer = os.Stderr
	Display("&w", &w)
	// Output:
	// Display &w (*io.Writer):
	// (*&w).type = *os.File
	// (*(*(*&w).value).file).fd = 2
	// (*(*(*&w).value).file).name = "/dev/stderr"
	// (*(*(*&w).value).file).nepipe = 0

	var locker sync.Locker = new(sync.Mutex)
	Display("(&locker)", &locker)
	// Output:
	// Display (&locker) (*sync.Locker):
	// (*(&locker)).type = *sync.Mutex
	// (*(*(&locker)).value).state = 0
	// (*(*(&locker)).value).sema = 0

	Display("locker", locker)
	// Output:
	// Display locker (*sync.Mutex):
	// (*locker).state = 0
	// (*locker).sema = 0
	// (*(&locker)) = nil

	locker = nil
	Display("(&locker)", &locker)
	// Output:
	// Display (&locker) (*sync.Locker):
	// (*(&locker)) = nil

	ips, _ := net.LookupHost("golang.org")
	Display("ips", ips)
	// Output:
	// Display ips ([]string):
	// ips[0] = "173.194.68.141"
	// ips[1] = "2607:f8b0:400d:c06::8d"

	// Even metarecursion!  (YMMV)
	Display("rV", reflect.ValueOf(os.Stderr))
	// Output:
	// Display rV (reflect.Value):
	// (*rV.typ).size = 8
	// (*rV.typ).ptrdata = 8
	// (*rV.typ).hash = 871609668
	// (*rV.typ)._ = 0
	// ...

	// a pointer that points to itself
	type P *P
	var p P
	p = &p
	Display("p", p)
	// Output:
	// Display p (display.P):
	// (*(*(*(*(*(*p)))))) = display.P 0xc820028018

	// a map that contains itself
	type M map[string]M
	m := make(M)
	m[""] = m
	Display("m", m)
	// Output:
	// Display m (display.M):
	// m[""][""][""][""][""][""] = display.M 0xc8200c0840

	// a slice that contains itself
	type S []S
	s := make(S, 1)
	s[0] = s
	Display("s", s)
	// Output:
	// Display s (display.S):
	// s[0][0][0][0][0][0] = display.S 0xc8200d6800

	// a linked list that eats its own tail
	type Cycle struct {
		Value int
		Tail  *Cycle
	}
	var c Cycle
	c = Cycle{42, &c}
	Display("c", c)
	// Output:
	// Display c (display.Cycle):
	// c.Value = 42
	// (*c.Tail).Value = 42
	// (*(*c.Tail).Tail).Value = 42
	// (*(*(*c.Tail).Tail).Tail) = display.Cycle value
}

func TestMapKeys(t *testing.T) {
	sm := map[struct{ x int }]int{
		{1}: 2,
		{2}: 3,
	}
	Display("sm", sm)
	// Output:
	// Display sm (map[struct { x int }]int):
	// sm[{x: 2}] = 3
	// sm[{x: 1}] = 2

	am := map[[3]int]int{
		{1, 2, 3}: 3,
		{2, 3, 4}: 4,
	}
	Display("am", am)
	// Output:
	// Display am (map[[3]int]int):
	// am[{1, 2, 3}] = 3
	// am[{2, 3, 4}] = 4
}
```

## 12.4. 示例: 编码为S表达式

Display是一个用于显示结构化数据的调试工具，但是它并不能将任意的Go语言对象编码为通用消息然后用于进程间通信。

正如我们在4.5节中中看到的，Go语言的标准库支持了包括JSON、XML和ASN.1等多种编码格式。还有另一种依然被广泛使用的格式是S表达式格式，采用Lisp语言的语法。但是和其他编码格式不同的是，Go语言自带的标准库并不支持S表达式，主要是因为它没有一个公认的标准规范。

在本节中，我们将定义一个包用于将任意的Go语言对象编码为S表达式格式，它支持以下结构：

```go
42          integer
"hello"     string（带有Go风格的引号）
foo         symbol（未用引号括起来的名字）
(1 2 3)     list  （括号包起来的0个或多个元素）
```

布尔型习惯上使用`t`符号表示`true`，空列表或`nil`符号表示`false`，但是为了简单起见，我们暂时忽略布尔类型。同时忽略的还有`chan`管道和函数，因为通过反射并无法知道它们的确切状态。我们忽略的还有浮点数、复数和`interface`。支持它们是练习12.3的任务。

我们将Go语言的类型编码为S表达式的方法如下。整数和字符串以显而易见的方式编码。空值编码为nil符号。数组和slice被编码为列表。

结构体被编码为成员对象的列表，每个成员对象对应一个有两个元素的子列表，子列表的第一个元素是成员的名字，第二个元素是成员的值。Map被编码为键值对的列表。传统上，S表达式使用点状符号列表(key . value)结构来表示key/value对，而不是用一个含双元素的列表，不过为了简单我们忽略了点状符号列表。

编码是由一个`encode`递归函数完成，如下所示。它的结构本质上和前面的`Display`函数类似：

*gopl.io/ch12/sexpr*

```Go
func encode(buf *bytes.Buffer, v reflect.Value) error {
	switch v.Kind() {
	case reflect.Invalid:
		buf.WriteString("nil")

	case reflect.Int, reflect.Int8, reflect.Int16,
		reflect.Int32, reflect.Int64:
		fmt.Fprintf(buf, "%d", v.Int())

	case reflect.Uint, reflect.Uint8, reflect.Uint16,
		reflect.Uint32, reflect.Uint64, reflect.Uintptr:
		fmt.Fprintf(buf, "%d", v.Uint())

	case reflect.String:
		fmt.Fprintf(buf, "%q", v.String())

	case reflect.Ptr:
		return encode(buf, v.Elem())

	case reflect.Array, reflect.Slice: // (value ...)
		buf.WriteByte('(')
		for i := 0; i < v.Len(); i++ {
			if i > 0 {
				buf.WriteByte(' ')
			}
			if err := encode(buf, v.Index(i)); err != nil {
				return err
			}
		}
		buf.WriteByte(')')

	case reflect.Struct: // ((name value) ...)
		buf.WriteByte('(')
		for i := 0; i < v.NumField(); i++ {
			if i > 0 {
				buf.WriteByte(' ')
			}
			fmt.Fprintf(buf, "(%s ", v.Type().Field(i).Name)
			if err := encode(buf, v.Field(i)); err != nil {
				return err
			}
			buf.WriteByte(')')
		}
		buf.WriteByte(')')

	case reflect.Map: // ((key value) ...)
		buf.WriteByte('(')
		for i, key := range v.MapKeys() {
			if i > 0 {
				buf.WriteByte(' ')
			}
			buf.WriteByte('(')
			if err := encode(buf, key); err != nil {
				return err
			}
			buf.WriteByte(' ')
			if err := encode(buf, v.MapIndex(key)); err != nil {
				return err
			}
			buf.WriteByte(')')
		}
		buf.WriteByte(')')

	default: // float, complex, bool, chan, func, interface
		return fmt.Errorf("unsupported type: %s", v.Type())
	}
	return nil
}
```

`Marshal`函数是对`encode`的包装，以保持和`encoding/...`下其它包有着相似的API：

```Go
// Marshal encodes a Go value in S-expression form.
func Marshal(v interface{}) ([]byte, error) {
	var buf bytes.Buffer
	if err := encode(&buf, reflect.ValueOf(v)); err != nil {
		return nil, err
	}
	return buf.Bytes(), nil
}
```

下面是Marshal对12.3节的`strangelove`变量编码后的结果：

```go
((Title "Dr. Strangelove") (Subtitle "How I Learned to Stop Worrying and Lo
ve the Bomb") (Year 1964) (Actor (("Grp. Capt. Lionel Mandrake" "Peter Sell
ers") ("Pres. Merkin Muffley" "Peter Sellers") ("Gen. Buck Turgidson" "Geor
ge C. Scott") ("Brig. Gen. Jack D. Ripper" "Sterling Hayden") ("Maj. T.J. \
"King\" Kong" "Slim Pickens") ("Dr. Strangelove" "Peter Sellers"))) (Oscars
("Best Actor (Nomin.)" "Best Adapted Screenplay (Nomin.)" "Best Director (N
omin.)" "Best Picture (Nomin.)")) (Sequel nil))
```

整个输出编码为一行中以减少输出的大小，但是也很难阅读。下面是对`S`表达式手动格式化的结果。编写一个`S`表达式的美化格式化函数将作为一个具有挑战性的练习任务；不过 http://gopl.io 也提供了一个简单的版本。

```go
((Title "Dr. Strangelove")
 (Subtitle "How I Learned to Stop Worrying and Love the Bomb")
 (Year 1964)
 (Actor (("Grp. Capt. Lionel Mandrake" "Peter Sellers")
         ("Pres. Merkin Muffley" "Peter Sellers")
         ("Gen. Buck Turgidson" "George C. Scott")
         ("Brig. Gen. Jack D. Ripper" "Sterling Hayden")
         ("Maj. T.J. \"King\" Kong" "Slim Pickens")
         ("Dr. Strangelove" "Peter Sellers")))
 (Oscars ("Best Actor (Nomin.)"
          "Best Adapted Screenplay (Nomin.)"
          "Best Director (Nomin.)"
          "Best Picture (Nomin.)"))
 (Sequel nil))
```

和`fmt.Print`、`json.Marshal`、`Display`函数类似，`sexpr.Marshal`函数处理带环的数据结构也会陷入死循环。

在12.6节中，我们将给出S表达式解码器的实现步骤，但是在那之前，我们还需要先了解如何通过反射技术来更新程序的变量。

**练习 12.3：** 实现`encode`函数缺少的分支。将布尔类型编码为`t`和`nil`，浮点数编码为Go语言的格式，复数`1+2i`编码为#C(1.0 2.0)格式。接口编码为类型名和值对，例如（"[]int" (1 2 3)），但是这个形式可能会造成歧义：`reflect.Type.String`方法对于不同的类型可能返回相同的结果。



**练习 12.4：** 修改`encode`函数，以上面的格式化形式输出S表达式。

```

```



**练习 12.5：** 修改encode函数，用JSON格式代替S表达式格式。然后使用标准库提供的`json.Unmarshal`解码器来验证函数是正确的。

```

```



**练习 12.6：** 修改encode，作为一个优化，忽略对是零值对象的编码。

```

```



**练习 12.7：** 创建一个基于流式的API，用于S表达式的解码，和`json.Decoder`(§4.5)函数功能类似。

```

```



## 12.5. 通过reflect.Value修改值

到目前为止，反射还只是程序中变量的另一种读取方式。然而，在本节中我们将重点讨论如何通过反射机制来修改变量。

回想一下，Go语言中类似`x`、`x.f[1]`和`*p`形式的表达式都可以表示变量，但是其它如x + 1和f(2)则不是变量。一个变量就是一个可寻址的内存空间，里面存储了一个值，并且存储的值可以通过内存地址来更新。

对于`reflect.Values`也有类似的区别。有一些`reflect.Values`是可取地址的；其它一些则不可以。考虑以下的声明语句：

```Go
x := 2                   // value   type    variable?
a := reflect.ValueOf(2)  // 2       int     no
b := reflect.ValueOf(x)  // 2       int     no
c := reflect.ValueOf(&x) // &x      *int    no
d := c.Elem()            // 2       int     yes (x)
```

其中a对应的变量不可取地址。因为a中的值仅仅是整数2的拷贝副本。b中的值也同样不可取地址。c中的值还是不可取地址，它只是一个指针`&x`的拷贝。实际上，所有通过`reflect.ValueOf(x)`返回的`reflect.Value`都是不可取地址的。但是对于d，它是c的解引用方式生成的，指向另一个变量，因此是可取地址的。我们可以通过调用`reflect.ValueOf(&x).Elem()`，来获取任意变量x对应的可取地址的Value。

我们可以通过调用`reflect.Value`的`CanAddr`方法来判断其是否可以被取地址：

```Go
fmt.Println(a.CanAddr()) // "false"
fmt.Println(b.CanAddr()) // "false"
fmt.Println(c.CanAddr()) // "false"
fmt.Println(d.CanAddr()) // "true"
```

每当我们通过指针间接地获取的`reflect.Value`都是可取地址的，即使开始的是一个不可取地址的Value。在反射机制中，所有关于是否支持取地址的规则都是类似的。例如，slice的索引表达式e[i]将隐式地包含一个指针，它就是可取地址的，即使开始的e表达式不支持也没有关系。以此类推，`reflect.ValueOf(e).Index(i)`对应的值也是可取地址的，即使原始的`reflect.ValueOf(e)`不支持也没有关系。

要从变量对应的可取地址的`reflect.Value`来访问变量需要三个步骤。第一步是调用Addr()方法，它返回一个Value，里面保存了指向变量的指针。然后是在Value上调用Interface()方法，也就是返回一个interface{}，里面包含指向变量的指针。最后，如果我们知道变量的类型，我们可以使用类型的断言机制将得到的interface{}类型的接口强制转为普通的类型指针。这样我们就可以通过这个普通指针来更新变量了：

```Go
x := 2
d := reflect.ValueOf(&x).Elem()   // d refers to the variable x
px := d.Addr().Interface().(*int) // px := &x
*px = 3                           // x = 3
fmt.Println(x)                    // "3"
```

或者，不使用指针，而是通过调用可取地址的reflect.Value的reflect.Value.Set方法来更新对应的值：

```Go
d.Set(reflect.ValueOf(4))
fmt.Println(x) // "4"
```

Set方法将在运行时执行和编译时进行类似的可赋值性约束的检查。以上代码，变量和值都是int类型，但是如果变量是int64类型，那么程序将抛出一个panic异常，所以关键问题是要确保改类型的变量可以接受对应的值：

```Go
d.Set(reflect.ValueOf(int64(5))) // panic: int64 is not assignable to int
```

同样，对一个不可取地址的reflect.Value调用Set方法也会导致panic异常：

```Go
x := 2
b := reflect.ValueOf(x)
b.Set(reflect.ValueOf(3)) // panic: Set using unaddressable value
```

这里有很多用于基本数据类型的Set方法：SetInt、SetUint、SetString和SetFloat等。

```Go
d := reflect.ValueOf(&x).Elem()
d.SetInt(3)
fmt.Println(x) // "3"
```

从某种程度上说，这些Set方法总是尽可能地完成任务。以SetInt为例，只要变量是某种类型的有符号整数就可以工作，即使是一些命名的类型、甚至只要底层数据类型是有符号整数就可以，而且如果对于变量类型值太大的话会被自动截断。但需要谨慎的是：对于一个引用interface{}类型的reflect.Value调用SetInt会导致panic异常，即使那个interface{}变量对于整数类型也不行。

```Go
x := 1
rx := reflect.ValueOf(&x).Elem()
rx.SetInt(2)                     // OK, x = 2
rx.Set(reflect.ValueOf(3))       // OK, x = 3
rx.SetString("hello")            // panic: string is not assignable to int
rx.Set(reflect.ValueOf("hello")) // panic: string is not assignable to int

var y interface{}
ry := reflect.ValueOf(&y).Elem()
ry.SetInt(2)                     // panic: SetInt called on interface Value
ry.Set(reflect.ValueOf(3))       // OK, y = int(3)
ry.SetString("hello")            // panic: SetString called on interface Value
ry.Set(reflect.ValueOf("hello")) // OK, y = "hello"
```

当我们用Display显示os.Stdout结构时，我们发现反射可以越过Go语言的导出规则的限制读取结构体中未导出的成员，比如在类Unix系统上os.File结构体中的fd int成员。然而，利用反射机制并不能修改这些未导出的成员：

```Go
stdout := reflect.ValueOf(os.Stdout).Elem() // *os.Stdout, an os.File var
fmt.Println(stdout.Type())                  // "os.File"
fd := stdout.FieldByName("fd")
fmt.Println(fd.Int()) // "1"
fd.SetInt(2)          // panic: unexported field
```

一个可取地址的reflect.Value会记录一个结构体成员是否是未导出成员，如果是的话则拒绝修改操作。因此，CanAddr方法并不能正确反映一个变量是否是可以被修改的。另一个相关的方法CanSet是用于检查对应的reflect.Value是否是可取地址并可被修改的：

```Go
fmt.Println(fd.CanAddr(), fd.CanSet()) // "true false"
```


## 12.6. 示例: 解码S表达式

标准库中encoding/...下每个包中提供的Marshal编码函数都有一个对应的Unmarshal函数用于解码。例如，我们在4.5节中看到的，要将包含JSON编码格式的字节slice数据解码为我们自己的Movie类型（§12.3），我们可以这样做：

```Go
data := []byte{/* ... */}
var movie Movie
err := json.Unmarshal(data, &movie)
```

Unmarshal函数使用了反射机制类修改movie变量的每个成员，根据输入的内容为Movie成员创建对应的map、结构体和slice。

现在让我们为S表达式编码实现一个简易的Unmarshal，类似于前面的json.Unmarshal标准库函数，对应我们之前实现的sexpr.Marshal函数的逆操作。我们必须提醒一下，一个健壮的和通用的实现通常需要比例子更多的代码，为了便于演示我们采用了精简的实现。我们只支持S表达式有限的子集，同时处理错误的方式也比较粗暴，代码的目的是为了演示反射的用法，而不是构造一个实用的S表达式的解码器。

词法分析器lexer使用了标准库中的text/scanner包将输入流的字节数据解析为一个个类似注释、标识符、字符串面值和数字面值之类的标记。输入扫描器scanner的Scan方法将提前扫描和返回下一个记号，对于rune类型。大多数记号，比如“(”，对应一个单一rune可表示的Unicode字符，但是text/scanner也可以用小的负数表示记号标识符、字符串等由多个字符组成的记号。调用Scan方法将返回这些记号的类型，接着调用TokenText方法将返回记号对应的文本内容。

因为每个解析器可能需要多次使用当前的记号，但是Scan会一直向前扫描，所以我们包装了一个lexer扫描器辅助类型，用于跟踪最近由Scan方法返回的记号。

<u><i>gopl.io/ch12/sexpr</i></u>
```Go
type lexer struct {
	scan  scanner.Scanner
	token rune // the current token
}

func (lex *lexer) next()        { lex.token = lex.scan.Scan() }
func (lex *lexer) text() string { return lex.scan.TokenText() }

func (lex *lexer) consume(want rune) {
	if lex.token != want { // NOTE: Not an example of good error handling.
		panic(fmt.Sprintf("got %q, want %q", lex.text(), want))
	}
	lex.next()
}
```

现在让我们转到语法解析器。它主要包含两个功能。第一个是read函数，用于读取S表达式的当前标记，然后根据S表达式的当前标记更新可取地址的reflect.Value对应的变量v。

```Go
func read(lex *lexer, v reflect.Value) {
	switch lex.token {
	case scanner.Ident:
		// The only valid identifiers are
		// "nil" and struct field names.
		if lex.text() == "nil" {
			v.Set(reflect.Zero(v.Type()))
			lex.next()
			return
		}
	case scanner.String:
		s, _ := strconv.Unquote(lex.text()) // NOTE: ignoring errors
		v.SetString(s)
		lex.next()
		return
	case scanner.Int:
		i, _ := strconv.Atoi(lex.text()) // NOTE: ignoring errors
		v.SetInt(int64(i))
		lex.next()
		return
	case '(':
		lex.next()
		readList(lex, v)
		lex.next() // consume ')'
		return
	}
	panic(fmt.Sprintf("unexpected token %q", lex.text()))
}
```

我们的S表达式使用标识符区分两个不同类型，结构体成员名和nil值的指针。read函数值处理nil类型的标识符。当遇到scanner.Ident为“nil”是，使用reflect.Zero函数将变量v设置为零值。而其它任何类型的标识符，我们都作为错误处理。后面的readList函数将处理结构体的成员名。

一个“(”标记对应一个列表的开始。第二个函数readList，将一个列表解码到一个聚合类型中（map、结构体、slice或数组），具体类型依然于传入待填充变量的类型。每次遇到这种情况，循环继续解析每个元素直到遇到于开始标记匹配的结束标记“)”，endList函数用于检测结束标记。

最有趣的部分是递归。最简单的是对数组类型的处理。直到遇到“)”结束标记，我们使用Index函数来获取数组每个元素的地址，然后递归调用read函数处理。和其它错误类似，如果输入数据导致解码器的引用超出了数组的范围，解码器将抛出panic异常。slice也采用类似方法解析，不同的是我们将为每个元素创建新的变量，然后将元素添加到slice的末尾。

在循环处理结构体和map每个元素时必须解码一个(key value)格式的对应子列表。对于结构体，key部分对于成员的名字。和数组类似，我们使用FieldByName找到结构体对应成员的变量，然后递归调用read函数处理。对于map，key可能是任意类型，对元素的处理方式和slice类似，我们创建一个新的变量，然后递归填充它，最后将新解析到的key/value对添加到map。

```Go
func readList(lex *lexer, v reflect.Value) {
	switch v.Kind() {
	case reflect.Array: // (item ...)
		for i := 0; !endList(lex); i++ {
			read(lex, v.Index(i))
		}

	case reflect.Slice: // (item ...)
		for !endList(lex) {
			item := reflect.New(v.Type().Elem()).Elem()
			read(lex, item)
			v.Set(reflect.Append(v, item))
		}

	case reflect.Struct: // ((name value) ...)
		for !endList(lex) {
			lex.consume('(')
			if lex.token != scanner.Ident {
				panic(fmt.Sprintf("got token %q, want field name", lex.text()))
			}
			name := lex.text()
			lex.next()
			read(lex, v.FieldByName(name))
			lex.consume(')')
		}

	case reflect.Map: // ((key value) ...)
		v.Set(reflect.MakeMap(v.Type()))
		for !endList(lex) {
			lex.consume('(')
			key := reflect.New(v.Type().Key()).Elem()
			read(lex, key)
			value := reflect.New(v.Type().Elem()).Elem()
			read(lex, value)
			v.SetMapIndex(key, value)
			lex.consume(')')
		}

	default:
		panic(fmt.Sprintf("cannot decode list into %v", v.Type()))
	}
}

func endList(lex *lexer) bool {
	switch lex.token {
	case scanner.EOF:
		panic("end of file")
	case ')':
		return true
	}
	return false
}
```

最后，我们将解析器包装为导出的`Unmarshal`解码函数，隐藏了一些初始化和清理等边缘处理。内部解析器以`panic`的方式抛出错误，但是`Unmarshal`函数通过在`defer`语句调用`recover`函数来捕获内部`panic`（§5.10），然后返回一个对`panic`对应的错误信息。

```Go
// Unmarshal parses S-expression data and populates the variable
// whose address is in the non-nil pointer out.
func Unmarshal(data []byte, out interface{}) (err error) {
	lex := &lexer{scan: scanner.Scanner{Mode: scanner.GoTokens}}
	lex.scan.Init(bytes.NewReader(data))
	lex.next() // get the first token
	defer func() {
		// NOTE: this is not an example of ideal error handling.
		if x := recover(); x != nil {
			err = fmt.Errorf("error at %s: %v", lex.scan.Position, x)
		}
	}()
	read(lex, reflect.ValueOf(out).Elem())
	return nil
}
```

生产实现不应该对任何输入问题都用panic形式报告，而且应该报告一些错误相关的信息，例如出现错误输入的行号和位置等。尽管如此，我们希望通过这个例子来展示类似encoding/json等包底层代码的实现思路，以及如何使用反射机制来填充数据结构。

**练习 12.8：** `sexpr.Unmarshal`函数和`json.Unmarshal`一样，都要求在解码前输入完整的字节`slice`。定义一个和`json.Decoder`类似的`sexpr.Decoder`类型，支持从一个`io.Reader`流解码。修改`sexpr.Unmarshal`函数，使用这个新的类型实现。



**练习 12.9：** 编写一个基于标记的API用于解码S表达式，参考`xml.Decoder`（7.14）的风格。你将需要五种类型的标记：`Symbol`、`String`、`Int`、`StartList`和`EndList`。



**练习 12.10：** 扩展`sexpr.Unmarshal`函数，支持布尔型、浮点数和`interface`类型的解码，使用 **练习 12.3：** 的方案。（提示：要解码接口，你需要将name映射到每个支持类型的`reflect.Type`。）



## 12.7. 获取结构体字段标签

在4.5节我们使用构体成员标签用于设置对应JSON对应的名字。其中`json`成员标签让我们可以选择成员的名字和抑制零值成员的输出。在本节，我们将看到如何通过反射机制类获取成员标签。

对于一个web服务，大部分HTTP处理函数要做的第一件事情就是展开请求中的参数到本地变量中。我们定义了一个工具函数，叫`params.Unpack`，通过使用结构体成员标签机制来让HTTP处理函数解析请求参数更方便。

首先，我们看看如何使用它。下面的search函数是一个HTTP请求处理函数。它定义了一个匿名结构体类型的变量，用结构体的每个成员表示HTTP请求的参数。其中结构体成员标签指明了对于请求参数的名字，为了减少URL的长度这些参数名通常都是神秘的缩略词。`Unpack`将请求参数填充到合适的结构体成员中，这样我们可以方便地通过合适的类型类来访问这些参数。

*gopl.io/ch12/search*

```Go
import "gopl.io/ch12/params"

// search implements the /search URL endpoint.
func search(resp http.ResponseWriter, req *http.Request) {
	var data struct {
		Labels     []string `http:"l"`
		MaxResults int      `http:"max"`
		Exact      bool     `http:"x"`
	}
	data.MaxResults = 10 // set default
	if err := params.Unpack(req, &data); err != nil {
		http.Error(resp, err.Error(), http.StatusBadRequest) // 400
		return
	}

	// ...rest of handler...
	fmt.Fprintf(resp, "Search: %+v\n", data)
}
```

下面的`Unpack`函数主要完成三件事情。第一，它调用`req.ParseForm()`来解析HTTP请求。然后，`req.Form`将包含所有的请求参数，不管HTTP客户端使用的是GET还是POST请求方法。

下一步，Unpack函数将构建每个结构体成员有效参数名字到成员变量的映射。如果结构体成员有成员标签的话，有效参数名字可能和实际的成员名字不相同。`reflect.Type`的`Field`方法将返回一个`reflect.StructField`，里面含有每个成员的名字、类型和可选的成员标签等信息。其中成员标签信息对应`reflect.StructTag`类型的字符串，并且提供了`Get`方法用于解析和根据特定key提取的子串，例如这里的http:"..."形式的子串。

<u><i>gopl.io/ch12/params</i></u>

```Go
// Unpack populates the fields of the struct pointed to by ptr
// from the HTTP request parameters in req.
func Unpack(req *http.Request, ptr interface{}) error {
	if err := req.ParseForm(); err != nil {
		return err
	}

	// Build map of fields keyed by effective name.
	fields := make(map[string]reflect.Value)
	v := reflect.ValueOf(ptr).Elem() // the struct variable
	for i := 0; i < v.NumField(); i++ {
		fieldInfo := v.Type().Field(i) // a reflect.StructField
		tag := fieldInfo.Tag           // a reflect.StructTag
		name := tag.Get("http")
		if name == "" {
			name = strings.ToLower(fieldInfo.Name)
		}
		fields[name] = v.Field(i)
	}

	// Update struct field for each parameter in the request.
	for name, values := range req.Form {
		f := fields[name]
		if !f.IsValid() {
			continue // ignore unrecognized HTTP parameters
		}
		for _, value := range values {
			if f.Kind() == reflect.Slice {
				elem := reflect.New(f.Type().Elem()).Elem()
				if err := populate(elem, value); err != nil {
					return fmt.Errorf("%s: %v", name, err)
				}
				f.Set(reflect.Append(f, elem))
			} else {
				if err := populate(f, value); err != nil {
					return fmt.Errorf("%s: %v", name, err)
				}
			}
		}
	}
	return nil
}
```

最后，Unpack遍历HTTP请求的name/valu参数键值对，并且根据更新相应的结构体成员。回想一下，同一个名字的参数可能出现多次。如果发生这种情况，并且对应的结构体成员是一个slice，那么就将所有的参数添加到slice中。其它情况，对应的成员值将被覆盖，只有最后一次出现的参数值才是起作用的。

populate函数小心用请求的字符串类型参数值来填充单一的成员v（或者是slice类型成员中的单一的元素）。目前，它仅支持字符串、有符号整数和布尔型。其中其它的类型将留做练习任务。

```Go
func populate(v reflect.Value, value string) error {
	switch v.Kind() {
	case reflect.String:
		v.SetString(value)

	case reflect.Int:
		i, err := strconv.ParseInt(value, 10, 64)
		if err != nil {
			return err
		}
		v.SetInt(i)

	case reflect.Bool:
		b, err := strconv.ParseBool(value)
		if err != nil {
			return err
		}
		v.SetBool(b)

	default:
		return fmt.Errorf("unsupported kind %s", v.Type())
	}
	return nil
}
```

如果我们上上面的处理程序添加到一个web服务器，则可以产生以下的会话：

```
$ go build gopl.io/ch12/search
$ ./search &
$ ./fetch 'http://localhost:12345/search'
Search: {Labels:[] MaxResults:10 Exact:false}
$ ./fetch 'http://localhost:12345/search?l=golang&l=programming'
Search: {Labels:[golang programming] MaxResults:10 Exact:false}
$ ./fetch 'http://localhost:12345/search?l=golang&l=programming&max=100'
Search: {Labels:[golang programming] MaxResults:100 Exact:false}
$ ./fetch 'http://localhost:12345/search?x=true&l=golang&l=programming'
Search: {Labels:[golang programming] MaxResults:10 Exact:true}
$ ./fetch 'http://localhost:12345/search?q=hello&x=123'
x: strconv.ParseBool: parsing "123": invalid syntax
$ ./fetch 'http://localhost:12345/search?q=hello&max=lots'
max: strconv.ParseInt: parsing "lots": invalid syntax
```

**练习 12.11：** 编写相应的`Pack`函数，给定一个结构体值，`Pack`函数将返回合并了所有结构体成员和值的URL。

```go

```



**练习 12.12：** 扩展成员标签以表示一个请求参数的有效值规则。例如，一个字符串可以是有效的email地址或一个信用卡号码，还有一个整数可能需要是有效的邮政编码。修改`Unpack`函数以检查这些规则。

```go

```



**练习 12.13：** 修改S表达式的编码器（§12.4）和解码器（§12.6），采用和`encoding/json`包（§4.5）类似的方式使用成员标签中的sexpr:"..."字串。

````go

````



## 12.8. 显示一个类型的方法集

我们的最后一个例子是使用`reflect.Type`来打印任意值的类型和枚举它的方法：

<u><i>gopl.io/ch12/methods</i></u>
```Go
// Print prints the method set of the value x.
func Print(x interface{}) {
	v := reflect.ValueOf(x)
	t := v.Type()
	fmt.Printf("type %s\n", t)

	for i := 0; i < v.NumMethod(); i++ {
		methType := v.Method(i).Type()
		fmt.Printf("func (%s) %s%s\n", t, t.Method(i).Name,
			strings.TrimPrefix(methType.String(), "func"))
	}
}
```

`reflect.Type`和`reflect.Value`都提供了一个Method方法。每次`t.Method(i)`调用将一个reflect.Method的实例，对应一个用于描述一个方法的名称和类型的结构体。每次`v.Method(i)`方法调用都返回一个`reflect.Value`以表示对应的值（§6.4），也就是一个方法是帮到它的接收者的。使用`reflect.Value.Call`方法（我们这里没有演示），将可以调用一个`Func`类型的Value，但是这个例子中只用到了它的类型。

这是属于`time.Duration`和`*strings.Replacer`两个类型的方法：

```Go
methods.Print(time.Hour)
// Output:
// type time.Duration
// func (time.Duration) Hours() float64
// func (time.Duration) Minutes() float64
// func (time.Duration) Nanoseconds() int64
// func (time.Duration) Seconds() float64
// func (time.Duration) String() string

methods.Print(new(strings.Replacer))
// Output:
// type *strings.Replacer
// func (*strings.Replacer) Replace(string) string
// func (*strings.Replacer) WriteString(io.Writer, string) (int, error)
```
## 12.9. 几点忠告

虽然反射提供的API远多于我们讲到的，我们前面的例子主要是给出了一个方向，通过反射可以实现哪些功能。反射是一个强大并富有表达力的工具，但是它应该被小心地使用，原因有三。

第一个原因是，基于反射的代码是比较脆弱的。对于每一个会导致编译器报告类型错误的问题，在反射中都有与之相对应的误用问题，不同的是编译器会在构建时马上报告错误，而反射则是在真正运行到的时候才会抛出`panic`异常，可能是写完代码很久之后了，而且程序也可能运行了很长的时间。

以前面的`readList`函数（§12.6）为例，为了从输入读取字符串并填充`int`类型的变量而调用的`reflect.Value.SetString`方法可能导致`panic`异常。绝大多数使用反射的程序都有类似的风险，需要非常小心地检查每个`reflect.Value`的对应值的类型、是否可取地址，还有是否可以被修改等。

避免这种因反射而导致的脆弱性的问题的最好方法，是将所有的反射相关的使用控制在包的内部，如果可能的话避免在包的API中直接暴露`reflect.Value`类型，这样可以限制一些非法输入。如果无法做到这一点，在每个有风险的操作前指向额外的类型检查。以标准库中的代码为例，当`fmt.Printf`收到一个非法的操作数时，它并不会抛出`panic`异常，而是打印相关的错误信息。程序虽然还有BUG，但是会更加容易诊断。

```Go
fmt.Printf("%d %s\n", "hello", 42) // "%!d(string=hello) %!s(int=42)"
```

反射同样降低了程序的安全性，还影响了自动化重构和分析工具的准确性，因为它们无法识别运行时才能确认的类型信息。

避免使用反射的第二个原因是，即使对应类型提供了相同文档，但是反射的操作不能做静态类型检查，而且大量反射的代码通常难以理解。总是需要小心翼翼地为每个导出的类型和其它接受`interface{}`或`reflect.Value`类型参数的函数维护说明文档。

第三个原因，基于反射的代码通常比正常的代码运行速度慢一到两个数量级。对于一个典型的项目，大部分函数的性能和程序的整体性能关系不大，所以当反射能使程序更加清晰的时候可以考虑使用。测试是一个特别适合使用反射的场景，因为每个测试的数据集都很小。但是对于性能关键路径的函数，最好避免使用反射。

