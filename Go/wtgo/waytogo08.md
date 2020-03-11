# 第8章：Map

`map` 是一种特殊的数据结构：一种元素对（`pair`）的无序集合，`pair` 的一个元素是 `key`，对应的另一个元素是 `value`，所以这个结构也称为关联数组或字典。这是一种快速寻找值的理想结构：给定 `key`，对应的 `value` 可以迅速定位。

`map` 这种数据结构在其他编程语言中也称为字典（Python）、hash 和 HashTable 等。

## 8.1 声明、初始化和 make

### 8.1.1 概念

map 是引用类型，可以使用如下声明：

```go
var map1 map[keytype]valuetype
var map1 map[string]int
```

（`[keytype]` 和 `valuetype` 之间允许有空格，但是 `gofmt` 移除了空格）

在声明的时候不需要知道 `map` 的长度，`map` 是可以动态增长的。

未初始化的 `map` 的值是 `nil`。

`key` 可以是任意可以用 `==` 或者 `!=` 操作符比较的类型，比如 `string`、`int`、`float`。所以数组、切片和结构体不能作为 `key` (译者注：含有数组切片的结构体不能作为 `key`，只包含内建类型的 `struct` 是可以作为 `key` 的），但是指针和接口类型可以。如果要用结构体作为 `key` 可以提供 `Key()` 和 `Hash()` 方法，这样可以通过结构体的域计算出唯一的数字或者字符串的 `key`。

`value` 可以是任意类型的；通过使用空接口类型（详见第 11.9 节），我们可以存储任意值，但是使用这种类型作为值时需要先做一次类型断言（详见第 11.3 节）。

`map` 传递给函数的代价很小：在 32 位机器上占 4 个字节，64 位机器上占 8 个字节，无论实际上存储了多少数据。通过 `key` 在 `map` 中寻找值是很快的，比线性查找快得多，但是仍然比从数组和切片的索引中直接读取要慢 100 倍；所以如果你很在乎性能的话还是建议用切片来解决问题。

`map` 也可以用函数作为自己的值，这样就可以用来做分支结构（详见第 5 章）：`key` 用来选择要执行的函数。

如果 `key1` 是 `map1` 的`key`，那么 `map1[key1]` 就是对应 `key1` 的值，就如同数组索引符号一样（数组可以视为一种简单形式的 `map`，`key` 是从 `0` 开始的整数）。

`key1` 对应的值可以通过赋值符号来设置为 `val1`：`map1[key1] = val1`。

令 `v := map1[key1]` 可以将 `key1` 对应的值赋值给 `v`；如果 `map` 中没有 `key1` 存在，那么 `v` 将被赋值为 `map1` 的值类型的空值。

常用的 `len(map1)` 方法可以获得 `map` 中的 `pair` 数目，这个数目是可以伸缩的，因为 `map-pairs` 在运行时可以动态添加和删除。

示例 8.1 [make_maps.go](examples/chapter_8/make_maps.go)

```go
package main
import "fmt"

func main() {
	var mapLit map[string]int
	//var mapCreated map[string]float32
	var mapAssigned map[string]int

	mapLit = map[string]int{"one": 1, "two": 2}
	mapCreated := make(map[string]float32)
	mapAssigned = mapLit

	mapCreated["key1"] = 4.5
	mapCreated["key2"] = 3.14159
	mapAssigned["two"] = 3

	fmt.Printf("Map literal at \"one\" is: %d\n", mapLit["one"])
	fmt.Printf("Map created at \"key2\" is: %f\n", mapCreated["key2"])
	fmt.Printf("Map assigned at \"two\" is: %d\n", mapLit["two"])
	fmt.Printf("Map literal at \"ten\" is: %d\n", mapLit["ten"])
}
```

输出结果：

```go
Map literal at "one" is: 1
Map created at "key2" is: 3.141590
Map assigned at "two" is: 3
Mpa literal at "ten" is: 0
```

`mapLit` 说明了 `map literals` 的使用方法： `map` 可以用 `{key1: val1, key2: val2}` 的描述方法来初始化，就像数组和结构体一样。

`map` 是 **引用类型** 的： 内存用 `make` 方法来分配。

`map` 的初始化：`var map1 = make(map[keytype]valuetype)`。

或者简写为：`map1 := make(map[keytype]valuetype)`。

上面例子中的 `mapCreated` 就是用这种方式创建的：`mapCreated := make(map[string]float32)`。

相当于：`mapCreated := map[string]float32{}`。

`mapAssigned` 也是 `mapList` 的引用，对 `mapAssigned` 的修改也会影响到 `mapList` 的值。

**不要使用 new，永远用 make 来构造 map**

> **注意** 如果你错误的使用 `new()` 分配了一个引用对象，你会获得一个空引用的指针，相当于声明了一个未初始化的变量并且取了它的地址：

```go
mapCreated := new(map[string]float32)
```

接下来当我们调用：`mapCreated["key1"] = 4.5` 的时候，编译器会报错：

```go
invalid operation: mapCreated["key1"] (index of type *map[string]float32).
```

为了说明值可以是任意类型的，这里给出了一个使用 `func() int` 作为值的 `map`：

示例 8.2 [map_func.go](examples/chapter_8/map_func.go)

```go
package main

import "fmt"

func main() {
	mf := map[int]func() int{
		1: func() int { return 10 },
		2: func() int { return 20 },
		5: func() int { return 50 },
	}
	fmt.Println(mf)
}
```

输出结果为：`map[1:0x10903be0 5:0x10903ba0 2:0x10903bc0]`: 整形都被映射到函数地址。

### 8.1.2 map 容量

和数组不同，`map` 可以根据新增的 `key-value` 对动态的伸缩，因此它不存在固定长度或者最大限制。但是你也可以选择标明 `map` 的初始容量 `capacity`，就像这样：`make(map[keytype]valuetype, cap)`。例如：

```go
map2 := make(map[string]float32, 100)
```

当 `map` 增长到容量上限的时候，如果再增加新的 `key-value` 对，`map` 的大小会自动加 1。所以出于性能的考虑，对于大的 `map` 或者会快速扩张的 `map`，即使只是大概知道容量，也最好先标明。

这里有一个 map 的具体例子，即将音阶和对应的音频映射起来：

```go
noteFrequency := map[string]float32 {
	"C0": 16.35, "D0": 18.35, "E0": 20.60, "F0": 21.83,
	"G0": 24.50, "A0": 27.50, "B0": 30.87, "A4": 440}
```

### 8.1.3 用切片作为 map 的值

既然一个 `key` 只能对应一个 `value`，而 `value` 又是一个原始类型，那么如果一个 `key` 要对应多个值怎么办？例如，当我们要处理UNIX机器上的所有进程，以父进程（`pid` 为整形）作为 `key`，所有的子进程（以所有子进程的 `pid` 组成的切片）作为 `value`。通过将 value 定义为 `[]int` 类型或者其他类型的切片，就可以优雅的解决这个问题。

这里有一些定义这种 map 的例子：

```go
mp1 := make(map[int][]int)
mp2 := make(map[int]*[]int)
```

## 8.2 测试键值对是否存在及删除元素

测试 `map1` 中是否存在 `key1`：

在例子 8.1 中，我们已经见过可以使用 `val1 = map1[key1]` 的方法获取 `key1` 对应的值 `val1`。如果 `map` 中不存在 `key1`，`val1` 就是一个值类型的空值。

这就会给我们带来困惑了：现在我们没法区分到底是 `key1` 不存在还是它对应的 `value` 就是空值。

为了解决这个问题，我们可以这么用：`val1, isPresent = map1[key1]`

`isPresent` 返回一个 `bool` 值：如果 `key1` 存在于 `map1`，`val1` 就是 `key1` 对应的 `value` 值，并且 `isPresent`为`true`；如果 `key1` 不存在，`val1` 就是一个空值，并且 `isPresent` 会返回 `false`。

如果你只是想判断某个 `key` 是否存在而不关心它对应的值到底是多少，你可以这么做：

```go
_, ok := map1[key1] // 如果key1存在则ok == true，否则ok为false
```

或者和 `if` 混合使用：

```go
if _, ok := map1[key1]; ok {
	// ...
}
```

从 `map1` 中删除 `key1`：

直接 `delete(map1, key1)` 就可以。

如果 `key1` 不存在，该操作不会产生错误。

示例 8.4 [map_testelement.go](examples/chapter_8/map_testelement.go)

```go
package main

import "fmt"

func main() {
	var value int
	var isPresent bool

	map1 := make(map[string]int)
	map1["New Delhi"] = 55
	map1["Beijing"] = 20
	map1["Washington"] = 25
	value, isPresent = map1["Beijing"]
	if isPresent {
		fmt.Printf("The value of \"Beijing\" in map1 is: %d\n", value)
	} else {
		fmt.Printf("map1 does not contain Beijing")
	}

	value, isPresent = map1["Paris"]
	fmt.Printf("Is \"Paris\" in map1 ?: %t\n", isPresent)
	fmt.Printf("Value is: %d\n", value)

	// delete an item:
	delete(map1, "Washington")
	value, isPresent = map1["Washington"]
	if isPresent {
		fmt.Printf("The value of \"Washington\" in map1 is: %d\n", value)
	} else {
		fmt.Println("map1 does not contain Washington")
	}
}
```

输出结果：

```go
The value of "Beijing" in map1 is: 20
Is "Paris" in map1 ?: false
Value is: 0
map1 does not contain Washington
```

## 8.3 for-range 的配套用法

可以使用 `for` 循环构造 `map`：

```go
for key, value := range map1 {
	...
}
```

第一个返回值 `key` 是 `map` 中的 `key` 值，第二个返回值则是该 `key` 对应的 `value` 值；这两个都是仅 `for` 循环内部可见的局部变量。其中第一个返回值`key`值是一个可选元素。如果你只关心值，可以这么使用：

```go
for _, value := range map1 {
	...
}
```

如果只想获取 key，你可以这么使用：

```go
for key := range map1 {
	fmt.Printf("key is: %d\n", key)
}
```

示例 8.5 [maps_forrange.go](examples/chapter_8/maps_forrange.go)：

```go
package main
import "fmt"

func main() {
	map1 := make(map[int]float32)
	map1[1] = 1.0
	map1[2] = 2.0
	map1[3] = 3.0
	map1[4] = 4.0
	for key, value := range map1 {
		fmt.Printf("key is: %d - value is: %f\n", key, value)
	}
}
```

输出结果：

```go
key is: 3 - value is: 3.000000
key is: 1 - value is: 1.000000
key is: 4 - value is: 4.000000
key is: 2 - value is: 2.000000
```

注意 `map` 不是按照 `key` 的顺序排列的，也不是按照 `value` 的序排列的。

问题 8.1： 下面这段代码的输出是什么？

```go
capitals := map[string] string {"France":"Paris", "Italy":"Rome", "Japan":"Tokyo" }
for key := range capitals {
	fmt.Println("Map item: Capital of", key, "is", capitals[key])
}
```

**练习 8.1**

创建一个 `map` 来保存每周 7 天的名字，将它们打印出来并且测试是否存在 `Tuesday` 和 `Hollyday`。

```go
// map_days.go
package main

import (
	"fmt"
)

var Days = map[int]string{
	1: "Monday",
	2: "Tuesday",
	3: "Wednesday",
	4: "Thursday",
	5: "Friday",
	6: "Saturday",
	7: "Sunday",
}

func main() {
	fmt.Println(Days)
	// fmt.Printf("%v", Days)
	flagHolliday := false
	for k, v := range Days {
		if v == "Thursday" || v == "Holliday" {
			fmt.Println(v, "is the", k, "th day in the week")
			if v == "Holliday" {
				flagHolliday = true
			}
		}
	}
	if !flagHolliday {
		fmt.Println("holliday is not a day!")
	}
}

/* Output:
map[1:Monday 2:Tuesday 3:Wednesday 4:Thursday 5:Friday 6:Saturday 7:Sunday]
Thursday  is the  4 th day in the week
holliday is not a day!
*/
```

## 8.4 map 类型的切片

假设我们想获取一个 `map` 类型的切片，我们必须使用两次 `make()` 函数，第一次分配切片，第二次分配 切片中每个 `map` 元素（参见下面的例子 8.4）。

示例 8.4 [maps_forrange2.go](examples/chapter_8/maps_forrange2.go)：

```go
package main
import "fmt"

func main() {
	// Version A:
	items := make([]map[int]int, 5)
	for i:= range items {
		items[i] = make(map[int]int, 1)
		items[i][1] = 2
	}
	fmt.Printf("Version A: Value of items: %v\n", items)

	// Version B: NOT GOOD!
	items2 := make([]map[int]int, 5)
	for _, item := range items2 {
		item = make(map[int]int, 1) // item is only a copy of the slice element.
		item[1] = 2 // This 'item' will be lost on the next iteration.
	}
	fmt.Printf("Version B: Value of items: %v\n", items2)
}
```

输出结果：

```go
Version A: Value of items: [map[1:2] map[1:2] map[1:2] map[1:2] map[1:2]]
Version B: Value of items: [map[] map[] map[] map[] map[]]
```

需要注意的是，应当像 A 版本那样通过索引使用切片的 `map` 元素。在 B 版本中获得的项只是 `map` 值的一个拷贝而已，所以真正的 `map` 元素没有得到初始化。

## 8.5 map 的排序

`map` 默认是无序的，不管是按照 `key` 还是按照 `value` 默认都不排序（详见第 8.3 节）。

如果你想为 `map` 排序，需要将 `key`（或者 `value`）拷贝到一个切片，再对切片排序（使用 `sort` 包，详见第 7.6.6 节），然后可以使用切片的 `for-range` 方法打印出所有的 `key` 和 `value`。

下面有一个示例：

示例 8.6 [sort_map.go](examples/chapter_8/sort_map.go)：

```go
// the telephone alphabet:
package main

import (
	"fmt"
	"sort"
)

var (
	barVal = map[string]int{
		"alpha": 34, "bravo": 56, "charlie": 23,
		"delta": 87, "echo": 56, "foxtrot": 12,
		"golf": 34, "hotel": 16, "indio": 87,
		"juliet": 65, "kili": 43, "lima": 98}
)

func main() {
	fmt.Println("unsorted:")
	for k, v := range barVal {
		fmt.Printf("Key: %v, Value: %v / ", k, v)
	}
	keys := make([]string, len(barVal))
	i := 0
	for k := range barVal {
		keys[i] = k
		i++
	}
	sort.Strings(keys)
	fmt.Println()
	fmt.Println("sorted:")
	for _, k := range keys {
		fmt.Printf("Key: %v, Value: %v / ", k, barVal[k])
	}
}
```

输出结果：

	unsorted:
	Key: bravo, Value: 56 / Key: echo, Value: 56 / Key: indio, Value: 87 / Key: juliet, Value: 65 / Key: alpha, Value: 34 / Key: charlie, Value: 23 / Key: delta, Value: 87 / Key: foxtrot, Value: 12 / Key: golf, Value: 34 / Key: hotel, Value: 16 / Key: kili, Value: 43 / Key: lima, Value: 98 /
	sorted:
	Key: alpha, Value: 34 / Key: bravo, Value: 56 / Key: charlie, Value: 23 / Key: delta, Value: 87 / Key: echo, Value: 56 / Key: foxtrot, Value: 12 / Key: golf, Value: 34 / Key: hotel, Value: 16 / Key: indio, Value: 87 / Key: juliet, Value: 65 / Key: kili, Value: 43 / Key: lima, Value: 98 /

但是如果你想要一个排序的列表你最好使用结构体切片，这样会更有效：

```go
type name struct {
	key string
	value int
}
```

## 8.6 将 map 的键值对调

这里对调是指调换 `key` 和 `value`。如果 `map` 的值类型可以作为 `key` 且所有的 `value` 是唯一的，那么通过下面的方法可以简单的做到键值对调。

示例 8.7 [invert_map.go](examples/chapter_8/invert_map.go)：

```go
package main
import (
	"fmt"
)

var (
	barVal = map[string]int{"alpha": 34, "bravo": 56, "charlie": 23,
							"delta": 87, "echo": 56, "foxtrot": 12,
							"golf": 34, "hotel": 16, "indio": 87,
							"juliet": 65, "kili": 43, "lima": 98}
)

func main() {
	invMap := make(map[int]string, len(barVal))
	for k, v := range barVal {
		invMap[v] = k
	}
	fmt.Println("inverted:")
	for k, v := range invMap {
		fmt.Printf("Key: %v, Value: %v / ", k, v)
	}
}
```

输出结果：

	inverted:
	Key: 34, Value: golf / Key: 23, Value: charlie / Key: 16, Value: hotel / Key: 87, Value: delta / Key: 98, Value: lima / Key: 12, Value: foxtrot / Key: 43, Value: kili / Key: 56, Value: bravo / Key: 65, Value: juliet /

如果原始 `value` 值不唯一那这么做肯定会出问题；这种情况下不会报错，但是当遇到不唯一的 `key` 时应当直接停止对调，且此时对调后的 `map` 很可能没有包含原 `map` 的所有键值对！一种解决方法就是仔细检查唯一性并且使用多值 `map`，比如使用 `map[int][]string` 类型。

**练习 8.2**

构造一个将英文饮料名映射为法语（或者任意你的母语）的集合；先打印所有的饮料，然后打印原名和翻译后的名字。接下来按照英文名排序后再打印出来。

```go
// map_drinks.go
package main

import (
	"fmt"
	"sort"
)

func main() {
	drinks := map[string]string{
		"beer":   "bière",
		"wine":   "vin",
		"water":  "eau",
		"coffee": "café",
		"thea":   "thé"}
	sdrinks := make([]string, len(drinks))
	ix := 0

	fmt.Printf("The following drinks are available:\n")
	for eng := range drinks {
		sdrinks[ix] = eng
		ix++
		fmt.Println(eng)
	}

	fmt.Println("")
	for eng, fr := range drinks {
		fmt.Printf("The french for %s is %s\n", eng, fr)
	}

	// SORTING:
	fmt.Println("")
	fmt.Println("Now the sorted output:")
	sort.Strings(sdrinks)

	fmt.Printf("The following sorted drinks are available:\n")
	for _, eng := range sdrinks {
		fmt.Println(eng)
	}

	fmt.Println("")
	for _, eng := range sdrinks {
		fmt.Printf("The french for %s is %s\n", eng, drinks[eng])
	}
}

/* Output:
The following drinks are available:
wine
beer
water
coffee
thea

The french for wine is vin
The french for beer is bière
The french for water is eau
The french for coffee is café
The french for thea is thé

Now the sorted output:
The following sorted drinks are available:
beer
coffee
thea
water
wine

The french for beer is bière
The french for coffee is café
The french for thea is thé
The french for water is eau
The french for wine is vin
*/
```

