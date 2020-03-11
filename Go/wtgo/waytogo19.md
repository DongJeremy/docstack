# 第19章：构建一个完整的应用程序

# 19.1 简介

由于 web 无处不在，本章我们将开发一个完整的程序：`goto`，它是一个 web 缩短网址应用程序。示例来自 Andrew Gerrand 的讲座（见参考资料 22）。我们将把项目分成 3 个阶段，每一个都会比之前阶段包含更多的功能，并逐渐展示更多 Go 语言中的特性。我们会大量使用在 [15章](15.0.md) 所学的网页应用程序的知识。

**版本 1：** 利用映射和结构体，与 `sync` 包的 `Mutex` 一起使用，以及一个结构体工厂。

**版本 2：** 数据以 `gob` 格式写入文件以实现持久化。

**版本 3：** 利用协程和通道重写应用（见 [14章](14.0.md)）。

**版本 4：** 如果我们要使用 json 格式的文件该如何修改？

**版本 5：** 用 rpc 协议实现的分布式版本。

由于代码变更频繁，不会展示在此处，仅给出访问地址。

# 19.2 短网址项目简介

你肯定知道有些浏览器中的地址（称为 URL）非常长且/或复杂，在网上有一些将他们转换成简短 URL 来使用的服务。我们的项目与此类似：它是具有 2 个功能的 *web 服务*（web service）：

## 添加 (Add)

给定一个较长的 URL，会将其转换成较短的版本，例如：
```
http://maps.google.com/maps?f=q&source=s_q&hl=en&geocode=&q=tokyo&sll=37.0625,-95.677068&sspn=68.684234,65.566406&ie=UTF8&hq=&hnear=Tokyo,+Japan&t=h&z=9
```
- (A) 转变为：`http://goto/UrcGq`
- (B) 并保存这对数据

## 重定向 (Redirect)

短网址被请求时，会把用户重定向到原始的长 URL。因此如果你在浏览器输入网址 (B)，会被重定向到页面 (A)。

# 版本 1 - 数据结构和前端界面

第 1 个版本的代码 *goto_v1* 见 [goto_v1](examples/chapter_19/goto_v1)。

# 19.3 数据结构

（本节代码见 [goto_v1/store.go](examples/chapter_19/goto_v1/store.go)。）

当程序运行在生产环境时，会收到很多短网址的请求，同时会有一些将长 URL 转换成短 URL 的请求。我们的程序要以什么样的结构存储这些数据呢？[19.2 节](19.2.md) 中 (A) 和 (B) 两种 URL 都是字符串，此外，它们相互关联：给定键 (B) 能获取到值 (A)，他们互相*映射*（map）。要将数据存储在内存中，我们需要这种结构，它们几乎存在于所有的编程语言中，只是名称有所不同，例如“哈希表”或“字典”等。

Go 语言就有这种内建的映射（map）：`map[string]string`。

键的类型写在 `[` 和 `]` 之间，紧接着是值的类型。有关映射的所有知识详见 [8 章](08.0.md)。为特定类型指定一个别名在严谨的程序中非常实用。Go 语言中通过关键字 `type` 来定义，因此有定义：
```go
type URLStore map[string]string
```
它从短 URL 映射到长 URL，两者都是字符串。

要创建那种类型的变量，并命名为 m，使用：
```go
m := make(URLStore)
```

假设 *http://goto/a* 映射到 *http://google.com/* ，我们要把它们存储到 m 中，可以用如下语句：
```go
m["a"] = "http://google.com/"
```
（键只是 *http://goto/* 的后缀，其前缀总是不变的。）

要获得给定 "a" 对应的长 URL，可以这么写：
```go
url := m["a"]
```
此时 `url` 的值等于 `http://google.com/`。

注意，使用了 `:=` 就不需要指明 url 的类型为 `string`，编译器会从右侧的值中推断出来。

## 使程序线程安全

这里，变量 `URLStore` 是中心化的内存存储。当收到网络流量时，会有很多 `Redirect` 服务的请求。这些请求其实只涉及读操作：以给定的短 URL 作为键，返回对应的长 URL 的值。然而，对 `Add` 服务的请求则大不相同，它们会更改 `URLStore`，添加新的键值对。当在瞬间收到大量更新请求时，可能会产生如下问题：添加操作可能被另一个同类请求打断，写入的长 URL 值可能会丢失；另外，读取和更改同时进行，导致可能读到脏数据。代码中的 map 并不保证当开始更新数据时，会彻底阻止另一个更新操作的启动。也就是说，map 不是线程安全的，goto 会并发地为很多请求提供服务。因此必须使 `URLStore` 是线程安全的，以便可以从不同的线程访问它。最简单和经典的方法是为其增加一个锁，它是 Go 标准库 `sync` 包中的 `Mutex` 类型，必须导入到我们的代码中（关于锁详见 [9.3 节](09.3.md)）。

现在，我们把 `URLStore` 类型的定义更改为一个结构体（就是字段的集合，类似 C 或 Java ，[10 章](10.0.md) 介绍了结构体），它含有两个字段：`map` 和 `sync` 包的 `RWMutex`：
```go
import "sync"
type URLStore struct {
	urls map[string]string		// map from short to long URLs
	mu sync.RWMutex
}
```

`RWMutex` 有两种锁：分别对应读和写。多个客户端可以同时设置读锁，但只有一个客户端可以设置写锁（以排除所有的读锁），有效地串行化变更，使他们按顺序生效。

我们将在 `Get` 函数中实现 `Redirect` 服务的读请求，在 `Set` 函数中实现 `Add` 服务的写请求。`Get` 函数类似下面这样：
```go
func (s *URLStore) Get(key string) string {
	s.mu.RLock()
	url := s.urls[key]
	s.mu.RUnlock()
	return url
}
```

函数按照键（短 URL）返回对应映射后的 URL。它所处理的变量是指针类型（见 [4.9 节](04.9.md)），指向 `URLStore`。但在读取值之前，先用 `s.mu.RLock()` 放置一个读锁，这样就不会有更新操作妨碍读取。数据读取后撤销锁定，以便挂起的更新操作可以开始。如果键不存在于 map 中会怎样？会返回字符串的零值（空字符串）。注意点号（`.`）类似面向对象的语言：在 `s` 的 `mu` 字段上调用方法 `RLock()`。

`Set` 函数同时需要 URL 的键值对，且必须放置写锁 `Lock()` 来排除同一时刻任何其他更新操作。函数返回布尔值 `true` 或 `false` 来表示 `Set` 操作是否成功：
```go
func (s *URLStore) Set(key, url string) bool {
	s.mu.Lock()
	_, present := s.urls[key]
	if present {
		s.mu.Unlock()
		return false
	}
	s.urls[key] = url
	s.mu.Unlock()
	return true
}
```

形式 `_, present := s.urls[key]` 可以测试 map 中是否已经包含该键，包含则 `present` 为 `true`，否则为 `false`。这种形式称为“逗号 ok 模式”，在 Go 代码中会频繁出现。如果键已存在，`Set` 函数直接返回布尔值 `false`，map 不会被更新（这样可以保证短 URL 不会重复）。如果键不存在，把它加入 map 中并返回 `true`。左侧 `_` 是一个值的占位符，赋值给 `_` 来表明我们不会使用它。注意在更新后尽早调用 `Unlock()` 来释放对 `URLStore` 的锁定。

## 使用 defer 简化代码

目前代码还比较简单，容易记得操作完成后调用 `Unlock()` 解锁。然而在代码更复杂时很容易忘记解锁，或者放置在错误的位置，往往导致问题很难追踪。对于这种情况 Go 提供了一个特殊关键字 `defer`（见 [6.4 节](06.4.md)）。在本例中，可以在 `Lock` 之后立即示意 `Unlock`，不过其效果是 `Unlock()` 只会在函数返回之前被调用。

`Get` 可以简化成以下代码（我们消除了本地变量 `url`）：
```go
func (s *URLStore) Get(key string) string {
	s.mu.RLock()
	defer s.mu.RUnlock()
	return s.urls[key]
}
```

`Set` 的逻辑在某种程度上也变得清晰了（我们不用再考虑解锁的事了）：
```go
func (s *URLStore) Set(key, url string) bool {
	s.mu.Lock()
	defer s.mu.Unlock()
	_, present := s.urls[key]
	if present {
		return false
	}
	s.urls[key] = url
	return true
}
```

## URLStore 工厂函数

`URLStore` 结构体中包含 map 类型的字段，使用前必须先用 `make` 初始化。在 Go 中创建一个结构体实例，一般是通过定义一个前缀为 `New`，能返回该类型已初始化实例的函数（通常是指向实例的指针）。
```go
func NewURLStore() *URLStore {
	return &URLStore{ urls: make(map[string]string) }
}
```

在 `return` 语句中，创建了 `URLStore` 字面量实例，其中包含初始化了的 map 映射。锁无需特别指明初始化，这是 Go 创建结构体实例的惯例。`&` 是取址运算符，它将我们要返回的内容变成指针，因为 `NewURLStore` 返回类型是 `*URLStore`。然后调用该函数来创建 `URLStore` 变量：
```go
var store = NewURLStore()
```

## 使用 URLStore

要新增一对短/长 URL 到 map 中，我们只需调用 s 上的 `Set` 方法，由于返回布尔值，可以把它包裹在 `if` 语句中：
```go
if s.Set("a", "http://google.com") {
	// 成功
}
```

要获取给定短 URL 对应的长 URL，调用 s 上的 `Get` 方法，将返回值放入变量 `url`：
```go
if url := s.Get("a"); url != "" {
	// 重定向到 url
} else {
	// 键未找到
}
```

这里我们利用 Go 语言 `if` 语句的特性，可以在起始部分、条件判断前放置初始化语句。另外还需要一个 `Count` 方法以获取 map 中键值对的数量，可以使用内建的 `len` 函数：
```go
func (s *URLStore) Count() int {
	s.mu.RLock()
	defer s.mu.RUnlock()
	return len(s.urls)
}
```

如何根据给定的长 URL 计算出短 URL 呢？为此我们创建一个函数 `genKey(n int) string {…}`，将 `s.Count()` 的当前值作为其整型参数传入。（具体算法并不重要，示例代码可以在 [key.go](examples/chapter_19/goto_v1/key.go) 找到。）

现在，我们可以创建一个 `Put` 方法，接收一个长 URL，用 `genKey` 生成其短 URL 键，调用 `Set` 方法在此键下存储长 URL 数据，然后返回这个键：
```go
func (s *URLStore) Put(url string) string {
	for {
		key := genKey(s.Count())
		if s.Set(key, url) {
			return key
		}
	}
	// shouldn’t get here
	return ""
}
```

`for` 循环会一直尝试调用 `Set` 直到成功为止（意味着生成了一个尚未存在的短网址）。现在我们定义好了数据存储，以及配套的可工作的函数（见代码 [store.go](examples/chapter_19/goto_v1/store.go)）。但这本身并不能完成任务，我们还需要开发 web 服务器以交付 `Add` 和 `Redirect` 服务。

# 19.4 用户界面：web 服务端

（本节代码见 [goto_v1/main.go](examples/chapter_19/goto_v1/main.go)。）

我们尚未编写启动程序的必要函数。它们（总是）类似 C，C++ 或 Java 中的 `main()` 函数，我们的 web 服务器由它启动，例如用如下命令在本地 8080 端口启动 web 服务器：
```go
http.ListenAndServe(":8080", nil)
```

（web 服务器的功能来自于 `http` 包，[15 章](15.0.md) 做了深入介绍）。web 服务器会在一个无限循环中监听到来的请求，但我们必须定义针对这些请求，服务器该如何响应。可以用被称为 HTTP 处理器的 `HandleFunc` 函数来办到，例如代码：
```go
http.HandleFunc("/add", Add)
```
如此，每个以 `/add` 结尾的请求都会调用 `Add` 函数（尚未完成）。

程序有两个 HTTP 处理器：
- `Redirect`，用于对短 URL 重定向
- `Add`，用于处理新提交的 URL

示意图：

![](images/19.4_fig19.1.jpg?raw=true)

最简单的 `main()` 函数类似这样：
```go
func main() {
	http.HandleFunc("/", Redirect)
	http.HandleFunc("/add", Add)
	http.ListenAndServe(":8080", nil)
}
```

对 `/add` 的请求由 `Add` 处理器处理，所有其他请求会被 `Redirect` 处理器处理。处理函数从到来的请求（一个类型为 `*http.Request` 的变量）中获取信息，然后产生响应并写入 `http.ResponseWriter` 类型变量 `w`。

`Add` 函数必须做的事有：
1. 读取长 URL，即：用 `r.FormValue("url")` 从 HTML 表单提交的 HTTP 请求中读取 URL
2. 使用 store 上的 `Put` 方法存储长 URL
3. 将对应的短 URL 发送给用户

每个需求都转化为一行代码：
```go
func Add(w http.ResponseWriter, r *http.Request) {
	url := r.FormValue("url")
	key := store.Put(url)
	fmt.Fprintf(w, "http://localhost:8080/%s", key)
}
```

这里 `fmt` 包的 `Fprintf` 函数用来替换字符串中的关键字 `%s`，然后将结果作为响应发送回客户端。注意 `Fprintf` 把数据写到了 `ResponseWriter` 中，其实 `Fprintf` 可以将数据写到任何实现了 `io.Writer` 的数据结构，即该结构实现了 `Write` 方法。Go 中 `io.Writer` 称为接口，可见 `Fprintf` 利用接口变得十分通用，可以对很多不同的类型写入数据。Go 中接口的使用十分普遍，它使代码更通用（见 [11 章](11.0.md)）。

还需要一个表单，仍然可以用 `Fprintf` 来输出，这次将常量写入 `w`。让我们来修改 `Add`，当未指定 URL 时显示 HTML 表单：
```go
func Add(w http.ResponseWriter, r *http.Request) {
	url := r.FormValue("url")
	if url == "" {
		fmt.Fprint(w, AddForm)
		return
	}
	key := store.Put(url)
	fmt.Fprintf(w, "http://localhost:8080/%s", key)
}

const AddForm = `
<form method="POST" action="/add">
URL: <input type="text" name="url">
<input type="submit" value="Add">
</form>
`
```

在那种情况下，发送字符串常量 `AddForm` 到客户端，它是 html 表单，包含一个 `url` 输入域和一个提交按钮，点击后发送 POST 请求到 `/add`。这样 `Add` 处理函数被再次调用，此时 `url` 的值来自文本域。（` `` ` 用来创建原始字符串，否则按惯例 `""` 将成为字符串边界。）

`Redirect` 函数在 HTTP 请求路径中找到键（短 URL 的键是请求路径去除首字符，在 Go 中可以写为 `[1:]`。例如请求 "/abc"，键就是 "abc"），用 `Get` 函数从 `store` 检索到对应的长 URL，对用户发送 HTTP 重定向。如果没找到 URL，发送 404 "Not Found" 错误取而代之：
```go
func Redirect(w http.ResponseWriter, r *http.Request) {
	key := r.URL.Path[1:]
	url := store.Get(key)
	if url == "" {
		http.NotFound(w, r)
		return
	}
	http.Redirect(w, r, url, http.StatusFound)
}
```

（`http.NotFound` 和 `http.Redirect` 是发送通用 HTTP 响应的工具函数。）

我们已经完整地遍历了 [goto_v1](examples/chapter_19/goto_v1) 的代码。

## 编译和运行

可执行程序已包含在示例代码下，如果你想立即测试可以跳过本节。其中包含 3 个 go 源文件和一个 Makefile 文件，通过它应用可以被编译和链接，只须如下操作：
- **Linux 和 OSX 平台：** 在终端窗口源码目录下启动 `make` 命令，或在 LiteIDE 中构建项目。
- **Windows 平台：** 启动 MINGW 环境，步骤为：开始菜单，所有程序，MinGW，MinGW Shell（见 [2.5.5 节](02.5.md)），在命令行窗口输入 `make` 并回车，源代码被编译并链接为原生 exe 可执行程序。

生成内容为可执行程序，Linux/OS X 下为 `goto`，Windows 下为 `goto.exe`。

要启动并运行 web 服务器，那么：
- **Linux 和 OSX 平台：** 输入命令 `./goto`。
- **Windows 平台：** 从 Go IDE 启动程序（如果 Windows 防火墙阻止程序启动，设置允许该程序）

## 测试该程序

打开浏览器并请求  url：`http://localhost:8080/add`

这会激活 `Add` 处理函数。请求还未包含 url 变量，所以响应会输出 html 表单询问输入：

![](images/19.4_fig19.2.png?raw=true)

添加一个长 URL 以获取等价的缩短版本，例如 `http://golang.org/pkg/bufio/#Writer`，然后单击按钮。应用会为你产生一个短 URL 并打印出来，例如 `http://
localhost:8080/2`。

![](images/19.4_fig19.3.jpg?raw=true)

复制该 URL 并在浏览器地址栏粘贴以发出请求，现在轮到 `Redirect` 处理函数上场了，对应长 URL 的页面被显示了出来。

![](images/19.4_fig19.4.jpg?raw=true)

# 版本 2 - 添加持久化存储

第 2 个版本的代码 *goto_v2* 见 [goto_v2](examples/chapter_19/goto_v2)。

# 19.5 持久化存储：gob

（本节代码见 [goto_v2/store.go](examples/chapter_19/goto_v2/store.go) 和 [goto_v2/main.go](examples/chapter_19/goto_v2/main.go)。）

当 goto 进程（监听在 8080 端口的 web 服务器）终止，这迟早会发生，内存 map 中缩短的 URL 就会丢失。要保留这些数据，就得将其保存到磁盘文件中。我们将修改 `URLStore`，使它可以保存数据到文件，且在 goto 启动时还原这些数据。为此我们使用 Go 标准库的 `encoding/gob` 包：它用于序列化和反序列化，将数据结构转换为字节数组（确切地说是切片），反之亦然（见 [12.11 节](12.11.md)）。

通过 `gob` 包的 `NewEncoder` 和 `NewDecoder` 函数，可以指定数据要写入或读取的位置。返回的 `Encoder` 和 `Decoder` 对象提供了 `Encode` 和 `Decode` 方法，用于对文件写入和从中读取 Go 数据结构。提示：`Encoder` 实现了 `Writer` 接口，同样 `Decoder` 实现了 `Reader` 接口。我们在 `URLStore` 上增加一个新的 `file` 字段（`*os.File` 类型），它是用于读写已打开文件的句柄。


```go
type URLStore struct {
	urls map[string]string
	mu sync.RWMutex
	file *os.File
}
```

我们把这个文件命名为 store.gob，当初始化 `URLStore` 时将其作为参数传入：
```go
var store = NewURLStore("store.gob")
```

接着，调整 `NewURLStore` 函数：
```go
func NewURLStore(filename string) *URLStore {
	s := &URLStore{urls: make(map[string]string)}
	f, err := os.OpenFile(filename, os.O_RDWR|os.O_CREATE|os.O_APPEND, 0644)
	if err != nil {
		log.Fatal("URLStore:", err)
	}
	s.file = f
	return s
}
```

现在，更新后的 `NewURLStore` 函数接受一个文件名参数，它会打开该文件（见 [12 章](12.0.md)），将返回的 `*os.File` 作为 `file` 字段的值存储在 `URLStore` 变量 `store` 中，即这里的本地变量 `s` 。

对 `OpenFile` 的调用可能会失败（例如文件可能被删除或改名）。它会返回一个错误 err，注意 Go 是如何处理这种情况的：
```go
f, err := os.OpenFile(filename, os.O_RDWR|os.O_CREATE|os.O_APPEND, 0644)
if err != nil {
	log.Fatal("URLStore:", err)
}
```

当 err 不为 `nil`，表示确实发生了错误，那么输出一条消息并停止程序执行。这是处理错误的一种方式，大多数情况下错误应该返回给调用函数，但这种检测错误的模式在 Go 代码中也很普遍。在 `}` 之后可以确定文件被成功打开了。

打开该文件时启用了写入标志，更精确地说是“追加模式”。每当一对新的短/长 URL 在程序中创建后，我们通过 `gob` 把它存储到文件 "store.gob" 中。

为达到目的，定义一个新的结构体类型 `record`：
```go
type record struct {
	Key, URL string
}
```

以及新的 `save` 方法，将给定的键和 URL 组成 `record` ，以 `gob` 编码的形式写入磁盘。
```go
func (s *URLStore) save(key, url string) error {
	e := gob.NewEncoder(s.file)
	return e.Encode(record{key, url})
}
```

goto 程序启动时，磁盘上存储的数据必须读取到 `URLStore` 的 map 中。为此，我们编写 `load` 方法：
```go
func (s *URLStore) load() error {
	if _, err := s.file.Seek(0, 0); err != nil {
		return err
	}
	d := gob.NewDecoder(s.file)
	var err error
	for err == nil {
		var r record
		if err = d.Decode(&r); err == nil {
			s.Set(r.Key, r.URL)
		}
	}
	if err == io.EOF {
		return nil
	}
	return err
}
```

这个新的 `load` 方法会寻址（`Seek`）到文件的起始位置，读取并解码（`Decode`）每一条记录（`record`），然后用 `Set` 方法将数据存储到 map 中。再次注意无处不在的错误处理模式。文件的解码由一个无限循环完成，只要没有错误就会一直继续：
```go
for err == nil {
	…
}
```

如果得到了一个错误，可能是刚解码了最后一条记录，于是产生了 `io.EOF`（EndOfFile） 错误。若并非此种错误，表示产生了解码错误，用 `return err` 来返回它。对该方法的调用必须加入到 `NewURLStore` 中：
```go
func NewURLStore(filename string) *URLStore {
	s := &URLStore{urls: make(map[string]string)}
	f, err := os.OpenFile(filename, os.O_RDWR|os.O_CREATE|os.O_APPEND, 0644)
	if err != nil {
		log.Fatal("Error opening URLStore:", err)
	}
	s.file = f
	if err := s.load(); err != nil {
		log.Println("Error loading data in URLStore:", err)
	}
	return s
}
```

同时在 `Put` 方法中，当新的 URL 对加入到 map 中，也应该立即将它们保存到数据文件中：
```go
func (s *URLStore) Put(url string) string {
	for {
		key := genKey(s.Count())
		if s.Set(key, url) {
			if err := s.save(key, url); err != nil {
				log.Println("Error saving to URLStore:", err)
			}
			return key
		}
	}
	panic("shouldn’t get here")
}
```

编译并测试这第二个版本的程序，或直接使用现有的可执行程序，验证关闭服务器（在终端窗口可以按 CTRL/C）并重启后，短 URL 仍然有效。goto 程序第一次启动时，文件 store.gob 还不存在，因此当载入数据时会得到错误：

	2011/09/11 11:08:11 Error loading URLStore: open store.gob: The system cannot find the file specified.


结束进程并重启后，就能正常工作了。或者，可以在 goto 启动前先创建空的 store.gob 文件。

**备注：** 当第二次启动 goto 时，可能会产生错误：

	Error loading URLStore: extra data in buffer

这是由于 `gob` 是基于流的协议，它不支持重新开始。在版本 4 中，会用 json 作为存储协议来补救此问题。

# 版本 3 - 添加协程

第 3 个版本的代码 *goto_v3* 见 [goto_v3](examples/chapter_19/goto_v3)。

# 19.6 用协程优化性能

如果有太多客户端同时尝试添加 URL，第 2 个版本依旧存在性能问题。得益于锁机制，我们的 map 可以在并发访问环境下安全地更新，但每条新产生的记录都要立即写入磁盘，这种机制成为了瓶颈。写入操作可能同时发生，根据不同操作系统的特性，可能会产生数据损坏。就算不产生写入冲突，每个客户端在 `Put` 函数返回前，必须等待数据写入磁盘。因此，在一个 I/O 负载很高的系统中，客户端为了完成 `Add` 请求，将等待更长的不必要的时间。

为缓解该问题，必须对 `Put` 和存储进程*解耦*：我们将使用 Go 的并发机制。我们不再将记录直接写入磁盘，而是发送到一个*通道*中，它是某种形式的缓冲区，因而发送函数不必等待它完成。

保存进程会从该通道读取数据并写入磁盘。它是以 `saveLoop` 协程启动的独立线程。现在 `main` 和 `saveLoop` 并行地执行，不会再发生阻塞。

将 `URLStore` 的 `file` 字段替换为 `record` 类型的通道：`save chan record`。
```go
type URLStore struct {
	urls map[string]string
	mu sync.RWMutex
	save chan record
}
```

通道和 map 一样，必须用 `make` 创建。我们会以此修改 `NewURLStore` 工厂函数，并给定缓冲区大小为1000，例如：`save := make(chan record, saveQueueLength)`。为解决性能问题，`Put` 可以发送记录 record 到带缓冲的 `save` 通道：
```go
func (s *URLStore) Put(url string) string {
	for {
		key := genKey(s.Count())
		if s.Set(key, url) {
			s.save <- record{key, url}
			return key
		}
	}
	panic("shouldn't get here")
}
```

`save` 通道的另一端必须有一个接收者：新的 `saveLoop` 方法在独立的协程中运行，它接收 record 值并将它们写入到文件。`saveLoop` 是在 `NewURLStore()` 函数中用 `go` 关键字启动的。现在，可以移除不必要的打开文件的代码。以下是修改后的 `NewURLStore()`：
```go
const saveQueueLength = 1000
func NewURLStore(filename string) *URLStore {
	s := &URLStore{
		urls: make(map[string]string),
		save: make(chan record, saveQueueLength),
	}
	if err := s.load(filename); err != nil {
		log.Println("Error loading URLStore:", err)
	}
	go s.saveLoop(filename)
	return s
}
```

以下是 `saveLoop` 方法的代码：
```go
func (s *URLStore) saveLoop(filename string) {
	f, err := os.Open(filename, os.O_WRONLY|os.O_CREATE|os.O_APPEND, 0644)
	if err != nil {
		log.Fatal("URLStore:", err)
	}
	defer f.Close()
	e := gob.NewEncoder(f)
	for {
		// taking a record from the channel and encoding it
		r := <-s.save
		if err := e.Encode(r); err != nil {
			log.Println("URLStore:", err)
		}
	}
}
```

在无限循环中，记录从 `save` 通道读取，然后编码到文件中。

我们在 [14 章](14.0.md) 深入学习了协程和通道，但在这里我们见到了实用的案例，更好地管理程序的不同部分。注意现在 `Encoder` 对象只被创建一次，而不是每次保存时都创建，这也可以节省了一些内存和运算处理。

还有一个改进可以使 goto 更灵活：我们可以将文件名、监听地址和主机名定义为标志（flag），来代替在程序中硬编码或定义常量。这样当程序启动时，可以在命令行中指定它们的新值，如果没有指定，将采用 flag 的默认值。该功能来自另一个包，所以需要 `import "flag"`（这个包的更详细信息见 [12.4 节](12.4.md)）。

先创建一些全局变量来保存 flag 的值：
```go
var (
	listenAddr = flag.String("http", ":8080", "http listen address")
	dataFile = flag.String("file", "store.gob", "data store file name")
	hostname = flag.String("host", "localhost:8080", "host name and port")
)
```

为了处理命令行参数，必须把 `flag.Parse()` 添加到 `main` 函数中，在 flag 解析后才能实例化 `URLStore`，一旦得知了 `dataFile` 的值（在代码中使用了 `*dataFile`，因为 flag 是指针类型必须解除引用来获取值，见 [4.9 节](04.9.md)）：
```go
var store *URLStore
func main() {
	flag.Parse()
	store = NewURLStore(*dataFile)
	http.HandleFunc("/", Redirect)
	http.HandleFunc("/add", Add)
	http.ListenAndServe(*listenAddr, nil)
}
```

现在 `Add` 处理函数中须用 `*hostname` 替换 `localhost:8080`：
```go
fmt.Fprintf(w, "http://%s/%s", *hostname, key)
```

编译或直接使用现有的可执行程序测试第 3 个版本。

# 版本 4 - 用 JSON 持久化存储

第 4 个版本的代码 *goto_v4* 见 [goto_v4](examples/chapter_19/goto_v4)。

# 19.7 以 json 格式存储

如果你是个敏锐的测试者也许已经注意到了，当 goto 程序启动 2 次，第 2 次启动后能读取短 URL 且完美地工作。然而从第 3 次开始，会得到错误：

	Error loading URLStore: extra data in buffer

这是由于 gob 是基于流的协议，它不支持重新开始。为补救该问题，这里我们使用 json 作为存储协议（见 [12.9 节](12.9.md)），它以纯文本形式存储数据，因此也可以被非 Go 语言编写的进程读取。同时也显示了更换一种不同的持久化协议是多么简单，因为与存储打交道的代码被清晰地隔离在 2 个方法中，即 `load` 和 `saveLoop`。

从创建新的空文件 store.json 开始，更改 main.go 中声明文件名变量的那一行：
```go
var dataFile = flag.String("file", "store.json", "data store file name")
```

在 store.go 中导入 `json` 取代 `gob`。然后在 `saveLoop` 中唯一需要被修改的行：
```go
e := gob.NewEncoder(f)
```

更改为：
```go
e := json.NewEncoder(f)
```

类似的，在 `load` 方法中：
```go
d := gob.NewDecoder(f)
```

修改为：
```go
d := json.NewDecoder(f)
```

这就是所有要改动的地方！编译，启动并测试，你会发现之前的错误不会再发生了。

# 版本 5 - 分布式程序

第 5 个版本的代码 *goto_v5*（19.8 节和 19.9 节讨论）见 [goto_v5](examples/chapter_19/goto_v5)。该版本仍然基于 `gob` 存储，但很容易调整为使用 json，正如版本 4 演示的那样。

# 19.8 多服务器处理架构

目前为止 goto 以单线程运行，但即使用协程，在一台机器上运行的单一进程，也只能为一定数量的并发请求提供服务。一个缩短网址服务，相对于 `Add`（用 `Put()` 写入），通常 `Redirect` 服务（用 `Get()` 读取）要多得多。因此我们应该可以创建任意数量的只读的从（slave）服务器，提供服务并缓存 `Get` 方法调用的结果，将 `Put` 请求转发给主（master）服务器，类似如下架构：

![图 19.5 跨越主从计算机的分布式负载](images/19.8_fig19.5.jpg?raw=true)

对于 slave 进程，要在网络上运行 goto 应用的一个 master 节点实例，它们必须能相互通信。Go 的 `rpc` 包为跨越网络发起函数调用提供了便捷的途径。这里将把 `URLStore` 变为 RPC 服务（[15.9 节](15.9.md) 详细讨论了 rpc 包）。slave 进程将应对 `Get` 请求以交付长 URL。当一个长 URL 要被转换为缩短版本（使用 `Put` 方法）时，它们通过 rpc 连接把任务委托给 master 进程，因此只有 master 节点会写入数据文件。

截至目前 `URLStore` 上基本的 `Get()` 和 `Put()` 方法具有如下签名：
```go
func (s *URLStore) Get(key string) string
func (s *URLStore) Put(url string) string
```

而 RPC 调用仅能使用如下形式的方法（t 是 T 类型的值）：
```go
func (t T) Name(args *ArgType, reply *ReplyType) error
```

要使 `URLStore` 成为 RPC 服务，需要修改 `Put` 和 `Get` 方法使它们符合上述函数签名。以下是修改后的签名：
```go
func (s *URLStore) Get(key, url *string) error
func (s *URLStore) Put(url, key *string) error
```

`Get()` 代码变更为：
```go
func (s *URLStore) Get(key, url *string) error {
	s.mu.RLock()
	defer s.mu.RUnlock()
	if u, ok := s.urls[*key]; ok {
		*url = u
		return nil
	}
	return errors.New("key not found")
}
```

现在，键和长 URL 都变成了指针，必须加上前缀 `*` 来取得它们的值，例如 `*key` 这种形式。`u` 是一个值，可以用 `*url = u` 来将其赋值给指针。

接着对 `Put()` 代码做同样的改动：
```go
func (s *URLStore) Put(url, key *string) error {
	for {
		*key = genKey(s.Count())
			if err := s.Set(key, url); err == nil {
			break
		}
	}
	if s.save != nil {
		s.save <- record{*key, *url}
	}
	return nil
}
```

`Put()` 调用 `Set()`，由于后者也要做调整，`key` 和 `url` 参数现在是指针类型，还必须返回 `error` 取代 `boolean`：
```go
func (s *URLStore) Set(key, url *string) error {
	s.mu.Lock()
	defer s.mu.Unlock()
	if _, present := s.urls[*key]; present {
		return errors.New("key already exists")
	}
	s.urls[*key] = *url
	return nil
}
```

同样，当从 `load()` 调用 `Set()` 时，也必须做调整：
```go
s.Set(&r.Key, &r.URL)
```

还必须修改 HTTP 处理函数以适应 `URLStore` 上的更改。`Redirect` 处理函数现在返回 `URLStore` 给出错误的字符串形式：
```go
func Redirect(w http.ResponseWriter, r *http.Request) {
	key := r.URL.Path[1:]
	var url string
	if err := store.Get(&key, &url); err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
	http.Redirect(w, r, url, http.StatusFound)
}
```

`Add` 处理函数也以基本相同的方式修改：
```go
func Add(w http.ResponseWriter, r *http.Request) {
	url := r.FormValue("url")
	if url == "" {
		fmt.Fprint(w, AddForm)
		return
	}
	var key string
	if err := store.Put(&url, &key); err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
	fmt.Fprintf(w, "http://%s/%s", *hostname, key)
}
```

要使应用程序更灵活，正如之前章节所为，可以添加一个命令行标志（flag）来决定是否在 `main()` 函数中启用 RPC 服务器：
```go
var rpcEnabled = flag.Bool("rpc", false, "enable RPC server")
```

要使 RPC 工作，还要用 `rpc` 包来注册 `URLStore`，并用 `HandleHTTP` 创建基于 HTTP 的 RPC 处理器：
```go
func main() {
	flag.Parse()
	store = NewURLStore(*dataFile)
	if *rpcEnabled { // flag has been set
		rpc.RegisterName("Store", store)
		rpc.HandleHTTP()
	}
	... (set up http like before)
}
```

# 19.9 使用代理缓存

`URLStore` 已经成为了有效的 RPC 服务，现在可以创建另一种代表 RPC 客户端的类型，它会转发请求到 RPC 服务器，我们称它为 `ProxyStore`。
```go
type ProxyStore struct {
	client *rpc.Client
}
```

一个 RPC 客户端必须使用 `DialHTTP()` 方法连接到服务器，所以我们把这句加入 `NewProxyStore` 函数，它用于创建 `ProxyStore` 对象。
```go
func NewProxyStore(addr string) *ProxyStore {
	client, err := rpc.DialHTTP("tcp", addr)
	if err != nil {
		log.Println("Error constructing ProxyStore:", err)
	}
	return &ProxyStore{client: client}
}
```

`ProxyStore` 有 `Get` 和 `Put` 方法，它们利用 RPC 客户端的 `Call` 方法，将请求直接传递给服务器：
```go
func (s *ProxyStore) Get(key, url *string) error {
	return s.client.Call("Store.Get", key, url)
}

func (s *ProxyStore) Put(url, key *string) error {
	return s.client.Call("Store.Put", url, key)
}
```

## 带缓存的 ProxyStore

可是，如果 slave 进程只是简单地代理所有的工作到 master 节点，不会得到任何增益！我们打算用 slave 节点来应对 `Get` 请求。要做到这点，它们必须有 `URLStore` 中 map 的一份副本（缓存）。因此我们对 `ProxyStore` 的定义进行扩展，将 `URLStore` 包含在其中：
```go
type ProxyStore struct {
	urls *URLStore
	client *rpc.Client
}
```

`NewProxyStore` 也必须做修改：
```go
func NewProxyStore(addr string) *ProxyStore {
	client, err := rpc.DialHTTP("tcp", addr)
	if err != nil {
		log.Println("ProxyStore:", err)
	}
	return &ProxyStore{urls: NewURLStore(""), client: client}
}
```

还必须修改 `NewURLStore` 以便给出空文件名时，不会尝试从磁盘写入或读取文件：
```go
func NewURLStore(filename string) *URLStore {
	s := &URLStore{urls: make(map[string]string)}
	if filename != "" {
		s.save = make(chan record, saveQueueLength)
		if err := s.load(filename); err != nil {
			log.Println("Error loading URLStore: ", err)
		}
		go s.saveLoop(filename)
	}
	return s
}
```

`ProxyStore` 的 `Get` 方法需要扩展：**它应该首先检查缓存中是否有对应的键**。如果有，`Get` 返回已缓存的结果。否则，应该发起 RPC 调用，然后用返回结果更新其本地缓存：
```go
func (s *ProxyStore) Get(key, url *string) error {
	if err := s.urls.Get(key, url); err == nil { // url found in local map
		return nil
	}
	// url not found in local map, make rpc-call:
	if err := s.client.Call("Store.Get", key, url); err != nil {
		return err
	}
	s.urls.Set(key, url)
	return nil
}
```

同样地，`Put` 方法仅当成功完成了远程 RPC `Put` 调用，才更新本地缓存：
```go
func (s *ProxyStore) Put(url, key *string) error {
	if err := s.client.Call("Store.Put", url, key); err != nil {
		return err
	}
	s.urls.Set(key, url)
	return nil
}
```

## 汇总

slave 节点使用 `ProxyStore`，只有 master 使用 `URLStore`。有鉴于创造它们的方式，它们看上去十分一致：两者都实现了相同签名的 `Get` 和 `Put` 方法，因此我们可以指定一个 `Store` 接口来概括它们的行为：
```go
type Store interface {
	Put(url, key *string) error
	Get(key, url *string) error
}
```

现在全局变量 `store` 可以成为 `Store` 类型：
```go
var store Store
```

最后，我们改写 `main()` 函数以便程序只作为 master 或 slave 启动（我们只能这么做，因为现在 store 是 `Store` 接口类型！）。

为此我们添加一个没有默认值的新命令行标志 `masterAddr`。
```go
var masterAddr = flag.String("master", "", "RPC master address")
```

如果给出 master 地址，就启动一个 slave 进程并创建新的 `ProxyStore`；否则启动 master 进程并创建新的 `URLStore`：
```go
func main() {
	flag.Parse()
	if *masterAddr != "" { // we are a slave
		store = NewProxyStore(*masterAddr)
	} else { // we are the master
		store = NewURLStore(*dataFile)
	}
	...
}
```

这样，我们已启用了 `ProxyStore` 作为 web 前端，以代替 `URLStore`。

其余的前端代码继续和之前一样地工作，它们不必在意 `Store` 接口。只有 master 进程会写数据文件。

现在可以加载一个 master 节点和数个 slave 节点，对 slave 进行压力测试。

编译这个版本 4 或直接使用现有的可执行程序。

要进行测试，首先在命令行用以下命令启动 master 节点：
```bash
./goto -http=:8081 -rpc=true	# （Windows 平台用 goto 代替 ./goto）
```
这里提供了 2 个标志：master 监听 8081 端口，已启用 RPC。

slave 节点用以下命令启动：
```bash
./goto -master=127.0.0.1:8081
```

它获取到 master 的地址，并在 8080 端口接受客户端请求。

在源码目录下已包含了以下 shell 脚本 [demo.sh](examples/chapter_19/goto_v5/demo.sh)，用来在类 Unix 系统下自动启动程序：
```bash
#!/bin/sh
gomake
./goto -http=:8081 -rpc=true &
master_pid=$!
sleep 1
./goto -master=127.0.0.1:8081 &
slave_pid=$!
echo "Running master on :8081, slave on :8080."
echo "Visit: http://localhost:8080/add"
echo "Press enter to shut down"
read
kill $master_pid
kill $slave_pid
```

要在 Windows 下测试，启动 MINGW shell 并启动 master，然后每个 slave 都要单独启动新的 MINGW shell 并启动 slave 进程。

# 19.10 总结和增强

通过逐步构建 goto 应用程序，我们遇到了几乎所有的 Go 语言特性。

虽然这个程序按照我们的目标行事，仍然有一些可改进的途径：
- *审美*：用户界面可以（极大地）美化。为此可以使用 Go 的 `template` 包（见 [15.7 节](15.7.md)）。
- *可靠性*：master/slave 之间的 RPC 连接应该可以更可靠：如果客户端到服务器之间的连接中断，客户端应该尝试重连。用一个 "dialer" 协程可以达成。
- *资源减负*：由于 URL 数据库大小不断增长，内存占用可能会成为一个问题。可以通过多台 master 服务器按照键分片来解决。
- *删除*：要支持删除短 URL，master 和 slave 之间的交互将变得更复杂。