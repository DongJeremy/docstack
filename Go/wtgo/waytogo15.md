# 第15章：网络，模板和网页应用

Go 在编写 web 应用方面非常得力。因为目前它还没有GUI（Graphic User Interface 即图形化用户界面）的框架，通过文本或者模板展现的 html 页面是目前 Go 编写界面应用程序的唯一方式。

> （**译者注：实际上在翻译的时候，已经有了一些不太成熟的GUI库例如：go ui。**）

## 15.1 tcp 服务器

这部分我们将使用 TCP 协议和在 14 章讲到的协程范式编写一个简单的客户端-服务器应用，一个（web）服务器应用需要响应众多客户端的并发请求：Go 会为每一个客户端产生一个协程用来处理请求。我们需要使用 `net` 包中网络通信的功能。它包含了处理 TCP/IP 以及 UDP 协议、域名解析等方法。

服务器端代码是一个单独的文件：

示例 15.1 [server.go](examples/chapter_15/server.go)

```go
package main

import (
	"fmt"
	"net"
)

func main() {
	fmt.Println("Starting the server ...")
	// 创建 listener
	listener, err := net.Listen("tcp", "localhost:50000")
	if err != nil {
		fmt.Println("Error listening", err.Error())
		return //终止程序
	}
	// 监听并接受来自客户端的连接
	for {
		conn, err := listener.Accept()
		if err != nil {
			fmt.Println("Error accepting", err.Error())
			return // 终止程序
		}
		go doServerStuff(conn)
	}
}

func doServerStuff(conn net.Conn) {
	for {
		buf := make([]byte, 512)
		len, err := conn.Read(buf)
		if err != nil {
			fmt.Println("Error reading", err.Error())
			return //终止程序
		}
		fmt.Printf("Received data: %v", string(buf[:len]))
	}
}
```

在 `main()` 中创建了一个 `net.Listener` 类型的变量 `listener`，他实现了服务器的基本功能：用来监听和接收来自客户端的请求（在 localhost 即 IP 地址为 127.0.0.1 端口为 50000 基于TCP协议）。`Listen()` 函数可以返回一个 `error` 类型的错误变量。用一个无限 `for` 循环的 `listener.Accept()` 来等待客户端的请求。客户端的请求将产生一个 `net.Conn` 类型的连接变量。然后一个独立的协程使用这个连接执行 `doServerStuff()`，开始使用一个 512 字节的缓冲 `data` 来读取客户端发送来的数据，并且把它们打印到服务器的终端，`len` 获取客户端发送的数据字节数；当客户端发送的所有数据都被读取完成时，协程就结束了。这段程序会为每一个客户端连接创建一个独立的协程。必须先运行服务器代码，再运行客户端代码。

客户端代码写在另一个文件 client.go 中：

示例 15.2 [client.go](examples/chapter_15/client.go)

```go
package main

import (
	"bufio"
	"fmt"
	"net"
	"os"
	"strings"
)

func main() {
	//打开连接:
	conn, err := net.Dial("tcp", "localhost:50000")
	if err != nil {
		//由于目标计算机积极拒绝而无法创建连接
		fmt.Println("Error dialing", err.Error())
		return // 终止程序
	}

	inputReader := bufio.NewReader(os.Stdin)
	fmt.Println("First, what is your name?")
	clientName, _ := inputReader.ReadString('\n')
	// fmt.Printf("CLIENTNAME %s", clientName)
	trimmedClient := strings.Trim(clientName, "\r\n") // Windows 平台下用 "\r\n"，Linux平台下使用 "\n"
	// 给服务器发送信息直到程序退出：
	for {
		fmt.Println("What to send to the server? Type Q to quit.")
		input, _ := inputReader.ReadString('\n')
		trimmedInput := strings.Trim(input, "\r\n")
		// fmt.Printf("input:--%s--", input)
		// fmt.Printf("trimmedInput:--%s--", trimmedInput)
		if trimmedInput == "Q" {
			return
		}
		_, err = conn.Write([]byte(trimmedClient + " says: " + trimmedInput))
	}
}
```
客户端通过 `net.Dial` 创建了一个和服务器之间的连接。

它通过无限循环从 `os.Stdin` 接收来自键盘的输入，直到输入了“`Q`”。注意裁剪 `\r` 和 `\n` 字符（仅 Windows 平台需要）。裁剪后的输入被 `connection` 的 `Write` 方法发送到服务器。

当然，服务器必须先启动好，如果服务器并未开始监听，客户端是无法成功连接的。

如果在服务器没有开始监听的情况下运行客户端程序，客户端会停止并打印出以下错误信息：`对tcp 127.0.0.1:50000发起连接时产生错误：由于目标计算机的积极拒绝而无法创建连接`。

打开命令提示符并转到服务器和客户端可执行程序所在的目录，Windows 系统下输入`server.exe`（或者只输入`server`），Linux系统下输入`./server`。

接下来控制台出现以下信息：`Starting the server ...`

在 Windows 系统中，我们可以通过 CTRL/C 停止程序。

然后开启 2 个或者 3 个独立的控制台窗口，分别输入 client 回车启动客户端程序

以下是服务器的输出：

```
Starting the Server ...
Received data: IVO says: Hi Server, what's up ?
Received data: CHRIS says: Are you busy server ?
Received data: MARC says: Don't forget our appointment tomorrow !
```
当客户端输入 Q 并结束程序时，服务器会输出以下信息：

```
Error reading WSARecv tcp 127.0.0.1:50000: The specified network name is no longer available.
```
在网络编程中 `net.Dial` 函数是非常重要的，一旦你连接到远程系统，函数就会返回一个 `Conn` 类型的接口，我们可以用它发送和接收数据。`Dial` 函数简洁地抽象了网络层和传输层。所以不管是 IPv4 还是 IPv6，TCP 或者 UDP 都可以使用这个公用接口。

以下示例先使用 TCP 协议连接远程 80 端口，然后使用 UDP 协议连接，最后使用 TCP 协议连接 IPv6 地址：

示例 15.3 [dial.go](examples/chapter_15/dial.go)

```go
// make a connection with www.example.org:
package main

import (
	"fmt"
	"net"
	"os"
)

func main() {
	conn, err := net.Dial("tcp", "192.0.32.10:80") // tcp ipv4
	checkConnection(conn, err)
	conn, err = net.Dial("udp", "192.0.32.10:80") // udp
	checkConnection(conn, err)
	conn, err = net.Dial("tcp", "[2620:0:2d0:200::10]:80") // tcp ipv6
	checkConnection(conn, err)
}
func checkConnection(conn net.Conn, err error) {
	if err != nil {
		fmt.Printf("error %v connecting!", err)
		os.Exit(1)
	}
	fmt.Printf("Connection is made with %v\n", conn)
}
```
下边也是一个使用 net 包从 socket 中打开，写入，读取数据的例子：

示例 15.4 [socket.go](examples/chapter_15/socket.go)

```go
package main

import (
	"fmt"
	"io"
	"net"
)

func main() {
	var (
		host          = "www.apache.org"
		port          = "80"
		remote        = host + ":" + port
		msg    string = "GET / \n"
		data          = make([]uint8, 4096)
		read          = true
		count         = 0
	)
	// 创建一个socket
	con, err := net.Dial("tcp", remote)
	// 发送我们的消息，一个http GET请求
	io.WriteString(con, msg)
	// 读取服务器的响应
	for read {
		count, err = con.Read(data)
		read = (err == nil)
		fmt.Printf(string(data[0:count]))
	}
	con.Close()
}
```

**练习 15.1** 

编写新版本的客户端和服务器（[client1.go](exercises/chapter_15/client1.go) / [server1.go](exercises/chapter_15/server1.go)）：

*	增加一个检查错误的函数 `checkError(error)`；讨论如下方案的利弊：为什么这个重构可能并没有那么理想？看看在 [示例15.14](examples/chapter_15/template_validation.go) 中它是如何被解决的
*	使客户端可以通过发送一条命令 SH 来关闭服务器
*	让服务器可以保存已经连接的客户端列表（他们的名字）；当客户端发送 WHO 指令的时候，服务器将显示如下列表：
```
This is the client list: 1:active, 0=inactive
User IVO is 1
User MARC is 1
User CHRIS is 1
```
注意：当服务器运行的时候，你无法编译/连接同一个目录下的源码来产生一个新的版本，因为 `server.exe` 正在被操作系统使用而无法被替换成新的版本。

下边这个版本的 simple_tcp_server.go 从很多方面优化了第一个tcp服务器的示例 server.go 并且拥有更好的结构，它只用了 80 行代码！

示例 15.5 [simple_tcp_server.go](examples/chapter_15/simple_tcp_server.go)：

```go
// Simple multi-thread/multi-core TCP server.
package main

import (
	"flag"
	"fmt"
	"net"
	"os"
)

const maxRead = 25

func main() {
	flag.Parse()
	if flag.NArg() != 2 {
		panic("usage: host port")
	}
	hostAndPort := fmt.Sprintf("%s:%s", flag.Arg(0), flag.Arg(1))
	listener := initServer(hostAndPort)
	for {
		conn, err := listener.Accept()
		checkError(err, "Accept: ")
		go connectionHandler(conn)
	}
}

func initServer(hostAndPort string) *net.TCPListener {
	serverAddr, err := net.ResolveTCPAddr("tcp", hostAndPort)
	checkError(err, "Resolving address:port failed: '"+hostAndPort+"'")
	listener, err := net.ListenTCP("tcp", serverAddr)
	checkError(err, "ListenTCP: ")
	println("Listening to: ", listener.Addr().String())
	return listener
}

func connectionHandler(conn net.Conn) {
	connFrom := conn.RemoteAddr().String()
	println("Connection from: ", connFrom)
	sayHello(conn)
	for {
		var ibuf []byte = make([]byte, maxRead+1)
		length, err := conn.Read(ibuf[0:maxRead])
		ibuf[maxRead] = 0 // to prevent overflow
		switch err {
		case nil:
			handleMsg(length, err, ibuf)
		case os.EAGAIN: // try again
			continue
		default:
			goto DISCONNECT
		}
	}
DISCONNECT:
	err := conn.Close()
	println("Closed connection: ", connFrom)
	checkError(err, "Close: ")
}

func sayHello(to net.Conn) {
	obuf := []byte{'L', 'e', 't', '\'', 's', ' ', 'G', 'O', '!', '\n'}
	wrote, err := to.Write(obuf)
	checkError(err, "Write: wrote "+string(wrote)+" bytes.")
}

func handleMsg(length int, err error, msg []byte) {
	if length > 0 {
		print("<", length, ":")
		for i := 0; ; i++ {
			if msg[i] == 0 {
				break
			}
			fmt.Printf("%c", msg[i])
		}
		print(">")
	}
}

func checkError(error error, info string) {
	if error != nil {
		panic("ERROR: " + info + " " + error.Error()) // terminate program
	}
}
```
（**译者注：应该是由于go版本的更新，会提示os.EAGAIN undefined ,修改后的代码：[simple_tcp_server_v1.go](examples/chapter_15/simple_tcp_server_v1.go)**）

都有哪些改进？

*	服务器地址和端口不再是硬编码，而是通过命令行参数传入，并通过 `flag` 包来读取这些参数。这里使用了 `flag.NArg()` 检查是否按照期望传入了2个参数：

```go
if flag.NArg() != 2 {
	panic("usage: host port")
}
```
传入的参数通过 `fmt.Sprintf` 函数格式化成字符串
```go
hostAndPort := fmt.Sprintf("%s:%s", flag.Arg(0), flag.Arg(1))
```
*	在 `initServer` 函数中通过 `net.ResolveTCPAddr` 得到了服务器地址和端口，这个函数最终返回了一个 `*net.TCPListener`
*	每一个连接都会以协程的方式运行 `connectionHandler` 函数。函数首先通过 `conn.RemoteAddr()` 获取到客户端的地址并显示出来
*	它使用 `conn.Write` 发送 Go 推广消息给客户端
*	它使用一个 25 字节的缓冲读取客户端发送的数据并一一打印出来。如果读取的过程中出现错误，代码会进入 `switch` 语句 `default` 分支，退出无限循环并关闭连接。如果是操作系统的 `EAGAIN` 错误，它会重试。
*	所有的错误检查都被重构在独立的函数 `checkError` 中，当错误产生时，利用错误上下文来触发 panic。

在命令行中输入 `simple_tcp_server localhost 50000` 来启动服务器程序，然后在独立的命令行窗口启动一些 client.go 的客户端。当有两个客户端连接的情况下服务器的典型输出如下，这里我们可以看到每个客户端都有自己的地址：
```
E:\Go\GoBoek\code examples\chapter 14>simple_tcp_server localhost 50000
Listening to: 127.0.0.1:50000
Connection from: 127.0.0.1:49346
<25:Ivo says: Hi server, do y><12:ou hear me ?>
Connection from: 127.0.0.1:49347
<25:Marc says: Do you remembe><25:r our first meeting serve><2:r?>
```

net.Error：
`net` 包返回的错误类型遵循惯例为 `error`，但有些错误实现包含额外的方法，他们被定义为 `net.Error` 接口：
```go
package net

type Error interface {
	Timeout() bool // 错误是否超时
	Temporary() bool // 是否是临时错误
}
```
通过类型断言，客户端代码可以测试 `net.Error`，从而区分是临时发生的还是必然会出现的错误。举例来说，一个网络爬虫程序在遇到临时发生的错误时可能会休眠或者重试，如果是一个必然发生的错误，则他会放弃继续执行。
```go
// in a loop - some function returns an error err
if nerr, ok := err.(net.Error); ok && nerr.Temporary() {
	time.Sleep(1e9)
	continue // try again
}
if err != nil {
	log.Fatal(err)
}
```

## 15.2 一个简单的网页服务器

http 是比 tcp 更高层的协议，它描述了网页服务器如何与客户端浏览器进行通信。Go 提供了 `net/http` 包，我们马上就来看下。先从一些简单的示例开始，首先编写一个“Hello world!”网页服务器：[查看示例15.6](examples/chapter_15/hello_world_webserver.go)

我们引入了 `http` 包并启动了网页服务器，和 [15.1节](15.1.md) 的 `net.Listen("tcp", "localhost:50000")` 函数的 tcp 服务器是类似的，使用 `http.ListenAndServe("localhost:8080", nil)` 函数，如果成功会返回空，否则会返回一个错误（地址 localhost 部分可以省略，8080 是指定的端口号）。

`http.URL` 用于表示网页地址，其中字符串属性 `Path` 用于保存 url 的路径；`http.Request` 描述了客户端请求，内含一个 `URL` 字段。

如果 `req` 是来自 html 表单的 POST 类型请求，“var1” 是该表单中一个输入域的名称，那么用户输入的值就可以通过 Go 代码 `req.FormValue("var1")` 获取到（见 [15.4节](15.4.md)）。还有一种方法是先执行 `request.ParseForm()`，然后再获取 `request.Form["var1"]` 的第一个返回参数，就像这样：
```go
var1, found := request.Form["var1"]
```
第二个参数 `found` 为 `true`。如果 `var1` 并未出现在表单中，`found` 就是 `false`。

表单属性实际上是 `map[string][]string` 类型。网页服务器发送一个 `http.Response` 响应，它是通过 `http.ResponseWriter` 对象输出的，后者组装了 HTTP 服务器响应，通过对其写入内容，我们就将数据发送给了 HTTP 客户端。

现在我们仍然要编写程序，以实现服务器必须做的事，即如何处理请求。这是通过 `http.HandleFunc` 函数完成的。在这个例子中，当根路径“/”（url地址是 `http://localhost:8080`）被请求的时候（或者这个服务器上的其他任意地址），`HelloServer` 函数就被执行了。这个函数是 `http.HandlerFunc` 类型的，它们通常被命名为 Prefhandler，和某个路径前缀 Pref 匹配。

`http.HandleFunc` 注册了一个处理函数（这里是 `HelloServer`）来处理对应 `/` 的请求。

`/` 可以被替换为其他更特定的 url，比如 `/create`，`/edit` 等等；你可以为每一个特定的 url 定义一个单独的处理函数。这个函数需要两个参数：第一个是 `ReponseWriter` 类型的 `w`；第二个是请求 `req`。程序向 `w` 写入了 `Hello` 和 `r.URL.Path[1:]` 组成的字符串：末尾的 `[1:]` 表示“创建一个从索引为 1 的字符到结尾的子切片”，用来丢弃路径开头的“/”，`fmt.Fprintf()` 函数完成了本次写入（见 [12.8节](12.8.md)）；另一种可行的写法是 `io.WriteString(w, "hello, world!\n")`。

总结：第一个参数是请求的路径，第二个参数是当路径被请求时，需要调用的处理函数的引用。

示例 15.6 [hello_world_webserver.go](examples/chapter_15/hello_world_webserver.go)：

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func HelloServer(w http.ResponseWriter, req *http.Request) {
	fmt.Println("Inside HelloServer handler")
	fmt.Fprintf(w, "Hello,"+req.URL.Path[1:])
}

func main() {
	http.HandleFunc("/", HelloServer)
	err := http.ListenAndServe("localhost:8080", nil)
	if err != nil {
		log.Fatal("ListenAndServe: ", err.Error())
	}
}
```
使用命令行启动程序，会打开一个命令窗口显示如下文字：
```
Starting Process E:/Go/GoBoek/code_examples/chapter_14/hello_world_webserver.exe...
```

然后打开浏览器并输入 url 地址：`http://localhost:8080/world`，浏览器就会出现文字：`Hello, world`，网页服务器会响应你在 `:8080/` 后边输入的内容。

`fmt.Println` 在服务器端控制台打印状态；在每个处理函数被调用时，把请求记录下来也许更为有用。

注：
1）前两行（没有错误处理代码）可以替换成以下写法：
```go
http.ListenAndServe(":8080", http.HandlerFunc(HelloServer))
```

2）`fmt.Fprint` 和 `fmt.Fprintf` 都是可以用来写入 `http.ResponseWriter` 的函数（他们实现了 `io.Writer`）。
比如我们可以使用
```go
fmt.Fprintf(w, "<h1>%s<h1><div>%s</div>", title, body)
```
来构建一个非常简单的网页并插入 `title` 和 `body` 的值。

如果你需要更多复杂的替换，使用模板包（见 [15.7节](15.7.md)）

3）如果你需要使用安全的 https 连接，使用 `http.ListenAndServeTLS()` 代替 `http.ListenAndServe()`

4）除了 `http.HandleFunc("/", Hfunc)`，其中的 `HFunc` 是一个处理函数，签名为：
```go
func HFunc(w http.ResponseWriter, req *http.Request) {
	...
}
```
也可以使用这种方式：`http.Handle("/", http.HandlerFunc(HFunc))`

`HandlerFunc` 只是定义了上述 HFunc 签名的别名：
```go
type HandlerFunc func(ResponseWriter, *Request)
```

它是一个可以把普通的函数当做 HTTP 处理器（`Handler`）的适配器。如果函数 `f` 声明的合适，`HandlerFunc(f)` 就是一个执行 `f` 函数的 `Handler` 对象。

`http.Handle` 的第二个参数也可以是 `T` 类型的对象 obj：`http.Handle("/", obj)`。

如果 T 有 `ServeHTTP` 方法，那就实现了http 的 `Handler` 接口：
```go
func (obj *Typ) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	...
}
```

这个用法也在 [15.8节](15.8.md) `Counter` 和 `Chan` 类型上使用。只要实现了 `http.Handler`，`http` 包就可以处理任何 HTTP 请求。

练习 15.2：[webhello2.go](exercises/chapter_15/webhello2.go)

编写一个网页服务器监听端口 9999，有如下处理函数：

*	当请求 `http://localhost:9999/hello/Name` 时，响应：`hello Name`（Name 需是一个合法的姓，比如 Chris 或者 Madeleine）

*	当请求 `http://localhost:9999/shouthello/Name` 时，响应：`hello NAME`

练习 15.3：[hello_server.go](exercises/chapter_15/hello_server.go)

创建一个空结构 `hello` 并为它实现 `http.Handler`。运行并测试。

## 15.3 访问并读取页面

在下边这个程序中，数组中的 url 都将被访问：会发送一个简单的 `http.Head()` 请求查看返回值；它的声明如下：`func Head(url string) (r *Response, err error)`

返回的响应 `Response` 其状态码会被打印出来。

示例 15.7 [poll_url.go](examples/chapter_15/poll_url.go)：

```go
package main

import (
	"fmt"
	"net/http"
)

var urls = []string{
	"http://www.google.com/",
	"http://golang.org/",
	"http://blog.golang.org/",
}

func main() {
	// Execute an HTTP HEAD request for all url's
	// and returns the HTTP status string or an error string.
	for _, url := range urls {
		resp, err := http.Head(url)
		if err != nil {
			fmt.Println("Error:", url, err)
		}
		fmt.Println(url, ": ", resp.Status)
	}
}
```

输出为：

	http://www.google.com/ : 302 Found
	http://golang.org/ : 200 OK
	http://blog.golang.org/ : 200 OK

***译者注*** 由于国内的网络环境现状，很有可能见到如下超时错误提示：
	Error: http://www.google.com/ Head http://www.google.com/: dial tcp 216.58.221.100:80: connectex: A connection attempt failed because the connected party did not properly respond after a period of time, or established connection failed because connected host has failed to respond.

在下边的程序中我们使用 `http.Get()` 获取并显示网页内容； `Get` 返回值中的 `Body` 属性包含了网页内容，然后我们用 `ioutil.ReadAll` 把它读出来：

示例 15.8 [http_fetch.go](examples/chapter_15/http_fetch.go)：

```go
package main

import (
	"fmt"
	"io/ioutil"
	"log"
	"net/http"
)

func main() {
	res, err := http.Get("http://www.google.com")
	checkError(err)
	data, err := ioutil.ReadAll(res.Body)
	checkError(err)
	fmt.Printf("Got: %q", string(data))
}

func checkError(err error) {
	if err != nil {
		log.Fatalf("Get : %v", err)
	}
}
```

当访问不存在的网站时，这里有一个`CheckError`输出错误的例子：

	2011/09/30 11:24:15 Get: Get http://www.google.bex: dial tcp www.google.bex:80:GetHostByName: No such host is known.

***译者注*** 和上一个例子相似，你可以把google.com更换为一个国内可以顺畅访问的网址进行测试

在下边的程序中，我们获取一个 twitter 用户的状态，通过 `xml` 包将这个状态解析成为一个结构：

示例 15.9 [twitter_status.go](examples/chapter_15/twitter_status.go)

```go
package main

import (
	"encoding/xml"
	"fmt"
	"net/http"
)

/*这个结构会保存解析后的返回数据。
他们会形成有层级的XML，可以忽略一些无用的数据*/
type Status struct {
	Text string
}

type User struct {
	XMLName xml.Name
	Status  Status
}

func main() {
	// 发起请求查询推特Goodland用户的状态
	response, _ := http.Get("http://twitter.com/users/Googland.xml")
	// 初始化XML返回值的结构
	user := User{xml.Name{"", "user"}, Status{""}}
	// 将XML解析为我们的结构
	xml.Unmarshal(response.Body, &user)
	fmt.Printf("status: %s", user.Status.Text)
}
```

输出：

	status: Robot cars invade California, on orders from Google: Google has been testing self-driving cars ... http://bit.ly/cbtpUN http://retwt.me/97p<exit code="0" msg="process exited normally"/>

**译者注** 和上边的示例相似，你可能无法获取到xml数据，另外由于go版本的更新，`xml.Unmarshal` 函数的第一个参数需是[]byte类型，而无法传入 `Body`。

我们会在 [15.4节](15.4.md) 中用到 `http` 包中的其他重要的函数：

*	`http.Redirect(w ResponseWriter, r *Request, url string, code int)`：这个函数会让浏览器重定向到 `url`（可以是基于请求 url 的相对路径），同时指定状态码。
*	`http.NotFound(w ResponseWriter, r *Request)`：这个函数将返回网页没有找到，HTTP 404错误。
*	`http.Error(w ResponseWriter, error string, code int)`：这个函数返回特定的错误信息和 HTTP 代码。
*	另一个 `http.Request` 对象 `req` 的重要属性：`req.Method`，这是一个包含 `GET` 或 `POST` 字符串，用来描述网页是以何种方式被请求的。

go为所有的HTTP状态码定义了常量，比如：
```go
http.StatusContinue		= 100
http.StatusOK			= 200
http.StatusFound		= 302
http.StatusBadRequest		= 400
http.StatusUnauthorized		= 401
http.StatusForbidden		= 403
http.StatusNotFound		= 404
http.StatusInternalServerError	= 500
```

你可以使用 `w.header().Set("Content-Type", "../..")` 设置头信息。

比如在网页应用发送 html 字符串的时候，在输出之前执行 `w.Header().Set("Content-Type", "text/html")`（通常不是必要的）。

练习 15.4：扩展 http_fetch.go 使之可以从控制台读取url，使用 [12.1节](12.1.md) 学到的接收控制台输入的方法（[http_fetch2.go](examples/chapter_15/http_fetch2.go)）

练习 15.5：获取 json 格式的推特状态，就像示例 15.9（[twitter_status_json.go](exercises/chapter_15/twitter_status_json.go)）

## 15.4 写一个简单的网页应用

下边的程序在端口 8088 上启动了一个网页服务器；`SimpleServer` 会处理 url `/test1` 使它在浏览器输出 `hello world`。`FormServer` 会处理 url `/test2`：如果 url 最初由浏览器请求，那么它是一个 `GET` 请求，返回一个 `form` 常量，包含了简单的 `input` 表单，这个表单里有一个文本框和一个提交按钮。当在文本框输入一些东西并点击提交按钮的时候，会发起一个 POST 请求。`FormServer` 中的代码用到了 `switch` 来区分两种情况。请求为 POST 类型时，`name` 属性 为 `inp` 的文本框的内容可以这样获取：`request.FormValue("inp")`。然后将其写回浏览器页面中。在控制台启动程序，然后到浏览器中打开 url `http://localhost:8088/test2` 来测试这个程序：

示例 15.10 [simple_webserver.go](examples/chapter_15/simple_webserver.go)

```go
package main

import (
	"io"
	"net/http"
)

const form = `
	<html><body>
		<form action="#" method="post" name="bar">
			<input type="text" name="in" />
			<input type="submit" value="submit"/>
		</form>
	</body></html>
`

/* handle a simple get request */
func SimpleServer(w http.ResponseWriter, request *http.Request) {
	io.WriteString(w, "<h1>hello, world</h1>")
}

func FormServer(w http.ResponseWriter, request *http.Request) {
	w.Header().Set("Content-Type", "text/html")
	switch request.Method {
	case "GET":
		/* display the form to the user */
		io.WriteString(w, form)
	case "POST":
		/* handle the form data, note that ParseForm must
		   be called before we can extract form data */
		//request.ParseForm();
		//io.WriteString(w, request.Form["in"][0])
		io.WriteString(w, request.FormValue("in"))
	}
}

func main() {
	http.HandleFunc("/test1", SimpleServer)
	http.HandleFunc("/test2", FormServer)
	if err := http.ListenAndServe(":8088", nil); err != nil {
		panic(err)
	}
}
```

注：当使用字符串常量表示 html 文本的时候，包含 `<html><body>...</body></html>` 对于让浏览器将它识别为 html 文档非常重要。

更安全的做法是在处理函数中，在写入返回内容之前将头部的 `content-type` 设置为`text/html`：`w.Header().Set("Content-Type", "text/html")`。

`content-type` 会让浏览器认为它可以使用函数 `http.DetectContentType([]byte(form))` 来处理收到的数据。

练习 15.6 [statistics.go](exercises/chapter_15/statistics.go)

编写一个网页程序，可以让用户输入一连串的数字，然后将它们打印出来，计算出这些数字的均值和中值，就像下边这张截图一样：

![](images/15.4_fig15.1.jpg?raw=true)

## 15.5 确保网页应用健壮

当网页应用的处理函数发生 panic，服务器会简单地终止运行。这可不妙：网页服务器必须是足够健壮的程序，能够承受任何可能的突发问题。

首先能想到的是在每个处理函数中使用 `defer/recover`，不过这样会产生太多的重复代码。[13.5节](13.5.md) 使用闭包的错误处理模式是更优雅的方案。我们把这种机制应用到前一章的简单网页服务器上。实际上，它可以被简单地应用到任何网页服务器程序中。

为增强代码可读性，我们为页面处理函数创建一个类型：
```go
type HandleFnc func(http.ResponseWriter, *http.Request)
```

我们的错误处理函数应用了[13.5节](13.5.md) 的模式，成为 `logPanics` 函数：
```go
func logPanics(function HandleFnc) HandleFnc {
	return func(writer http.ResponseWriter, request *http.Request) {
		defer func() {
			if x := recover(); x != nil {
				log.Printf("[%v] caught panic: %v", request.RemoteAddr, x)
			}
		}()
		function(writer, request)
	}
}
```

然后我们用 `logPanics` 来包装对处理函数的调用：
```go
http.HandleFunc("/test1", logPanics(SimpleServer))
http.HandleFunc("/test2", logPanics(FormServer))
```

处理函数现在可以恢复 panic 调用，类似[13.5节](13.5.md) 中的错误检测函数。完整代码如下：

示例 15.11 [robust_webserver.go](examples/chapter_15/robust_webserver.go)

```go
package main

import (
	"io"
	"log"
	"net/http"
)

const form = `<html><body><form action="#" method="post" name="bar">
		<input type="text" name="in"/>
		<input type="submit" value="Submit"/>
	</form></html></body>`

type HandleFnc func(http.ResponseWriter, *http.Request)

/* handle a simple get request */
func SimpleServer(w http.ResponseWriter, request *http.Request) {
	io.WriteString(w, "<h1>hello, world</h1>")
}

/* handle a form, both the GET which displays the form
   and the POST which processes it.*/
func FormServer(w http.ResponseWriter, request *http.Request) {
	w.Header().Set("Content-Type", "text/html")
	switch request.Method {
	case "GET":
		/* display the form to the user */
		io.WriteString(w, form)
	case "POST":
		/* handle the form data, note that ParseForm must
		   be called before we can extract form data*/
		//request.ParseForm();
		//io.WriteString(w, request.Form["in"][0])
		io.WriteString(w, request.FormValue("in"))
	}
}

func main() {
	http.HandleFunc("/test1", logPanics(SimpleServer))
	http.HandleFunc("/test2", logPanics(FormServer))
	if err := http.ListenAndServe(":8088", nil); err != nil {
		panic(err)
	}
}

func logPanics(function HandleFnc) HandleFnc {
	return func(writer http.ResponseWriter, request *http.Request) {
		defer func() {
			if x := recover(); x != nil {
				log.Printf("[%v] caught panic: %v", request.RemoteAddr, x)
			}
		}()
		function(writer, request)
	}
}
```

## 15.6 用模板编写网页应用

以下程序是用 100 行以内代码实现可行的 wiki 网页应用，它由一组页面组成，用于阅读、编辑和保存。它是来自 Go 网站 codelab 的 wiki 制作教程，我所知的最好的 Go 教程之一，非常值得进行完整的实验，以见证并理解程序是如何被构建起来的（[https://golang.org/doc/articles/wiki/](https://golang.org/doc/articles/wiki/)）。这里，我们将以自顶向下的视角，从整体上给出程序的补充说明。程序是网页服务器，它必须从命令行启动，监听某个端口，例如 8080。浏览器可以通过请求 URL 阅读 wiki 页面的内容，例如：`http://localhost:8080/view/page1`。

接着，页面的文本内容从一个文件中读取，并显示在网页中。它包含一个超链接，指向编辑页面（`http://localhost:8080/edit/page1`）。编辑页面将内容显示在一个文本域中，用户可以更改文本，点击“保存”按钮保存到对应的文件中。然后回到阅读页面显示更改后的内容。如果某个被请求阅读的页面不存在（例如：`http://localhost:8080/edit/page999`），程序可以作出识别，立即重定向到编辑页面，如此新的 wiki 页面就可以被创建并保存。

wiki 页面需要一个标题和文本内容，它在程序中被建模为如下结构体，Body 字段存放内容，由字节切片组成。
```go
type Page struct {
	Title string
	Body  []byte
}
```

为了在可执行程序之外维护 wiki 页面内容，我们简单地使用了文本文件作为持久化存储。程序、必要的模板和文本文件可以在 [wiki](examples/chapter_15/wiki) 中找到。

示例 15.12 [wiki.go](examples/chapter_15/wiki/wiki.go)

```go
package main

import (
	"net/http"
	"io/ioutil"
	"log"
	"regexp"
	"text/template"
)

const lenPath = len("/view/")

var titleValidator = regexp.MustCompile("^[a-zA-Z0-9]+$")
var templates = make(map[string]*template.Template)
var err error

type Page struct {
	Title string
	Body  []byte
}

func init() {
	for _, tmpl := range []string{"edit", "view"} {
		templates[tmpl] = template.Must(template.ParseFiles(tmpl + ".html"))
	}
}

func main() {
	http.HandleFunc("/view/", makeHandler(viewHandler))
	http.HandleFunc("/edit/", makeHandler(editHandler))
	http.HandleFunc("/save/", makeHandler(saveHandler))
	err := http.ListenAndServe("localhost:8080", nil)
	if err != nil {
		log.Fatal("ListenAndServe: ", err.Error())
	}
}

func makeHandler(fn func(http.ResponseWriter, *http.Request, string)) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		title := r.URL.Path[lenPath:]
		if !titleValidator.MatchString(title) {
			http.NotFound(w, r)
			return
		}
		fn(w, r, title)
	}
}

func viewHandler(w http.ResponseWriter, r *http.Request, title string) {
	p, err := load(title)
	if err != nil { // page not found
		http.Redirect(w, r, "/edit/"+title, http.StatusFound)
		return
	}
	renderTemplate(w, "view", p)
}

func editHandler(w http.ResponseWriter, r *http.Request, title string) {
	p, err := load(title)
	if err != nil {
		p = &Page{Title: title}
	}
	renderTemplate(w, "edit", p)
}

func saveHandler(w http.ResponseWriter, r *http.Request, title string) {
	body := r.FormValue("body")
	p := &Page{Title: title, Body: []byte(body)}
	err := p.save()
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
	http.Redirect(w, r, "/view/"+title, http.StatusFound)
}

func renderTemplate(w http.ResponseWriter, tmpl string, p *Page) {
	err := templates[tmpl].Execute(w, p)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
	}
}

func (p *Page) save() error {
	filename := p.Title + ".txt"
	// file created with read-write permissions for the current user only
	return ioutil.WriteFile(filename, p.Body, 0600)
}

func load(title string) (*Page, error) {
	filename := title + ".txt"
	body, err := ioutil.ReadFile(filename)
	if err != nil {
		return nil, err
	}
	return &Page{Title: title, Body: body}, nil
}
```

让我们来通读代码：

- 首先导入必要的包。由于我们在构建网页服务器，`http` 当然是必须的。不过还导入了 `io/ioutil` 来方便地读写文件，`regexp` 用于验证输入标题，以及 `template` 来动态创建 html 文档。

- 为避免黑客构造特殊输入攻击服务器，我们用如下正则表达式检查用户在浏览器上输入的 URL（同时也是 wiki 页面标题）：
  ```go
  var titleValidator = regexp.MustCompile("^[a-zA-Z0-9]+$")
  ```
  `makeHandler` 会用它对请求管控。
  
- 必须有一种机制把 `Page` 结构体数据插入到网页的标题和内容中，可以利用 `template` 包通过如下步骤完成：

  1. 先在文本编辑器中创建 html 模板文件，例如 view.html：

     ```html
     <h1>{{.Title |html}}</h1>
     <p>[<a href="/edit/{{.Title |html}}">edit</a>]</p>
     <div>{{printf "%s" .Body |html}}</div>
     ```
     把要插入的数据结构字段放在 `{{` 和 `}}` 之间，这里是把 `Page` 结构体数据 `{{.Title |html}}` 和 `{{printf "%s" .Body |html}}` 插入页面（当然可以是非常复杂的 html，但这里尽可能地简化了，以突出模板的原理。）（`{{.Title |html}}` 和 `{{printf "%s" .Body |html}}` 语法说明详见后续章节）。

  2. `template.Must(template.ParseFiles(tmpl + ".html"))` 把模板文件转换为 `*template.Template` 类型的对象，为了高效，在程序运行时仅做一次解析，在 `init()` 函数中处理可以方便地达到目的。所有模板对象都被保持在内存中，存放在以 html 文件名作为索引的 map 中：

     ```go
  	templates = make(map[string]*template.Template)
     ```
      此种技术被称为*模板缓存*，是推荐的最佳实践。

  3. 为了真正从模板和结构体构建出页面，必须使用：

     ```go
  	templates[tmpl].Execute(w, p)
     ```

      它基于模板执行，用 `Page` 结构体对象 p 作为参数对模板进行替换，并写入 `ResponseWriter` 对象 w。必须检查该方法的 error 返回值，万一有一个或多个错误，我们可以调用 `http.Error` 来明示。在我们的应用程序中，这段代码会被多次调用，所以把它提取为单独的函数 `renderTemplate`。

- 在 `main()` 中网页服务器用 `ListenAndServe` 启动并监听 8080 端口。但正如 [15.2节](15.2.md) 那样，需要先为紧接在 URL `localhost:8080/` 之后， 以`view`, `edit` 或 `save` 开头的 url 路径定义一些处理函数。在大多数网页服务器应用程序中，这形成了一系列 URL 路径到处理函数的映射，类似于 Ruby 和 Rails，Django 或 ASP.NET MVC 这样的 MVC 框架中的路由表。请求的 URL 与这些路径尝试匹配，较长的路径被优先匹配。如不与任何路径匹配，则调用 / 的处理程序。

  在此定义了 3 个处理函数，由于包含重复的启动代码，我们将其提取到单独的 `makeHandler` 函数中。这是一个值得研究的特殊高阶函数：其参数是一个函数，返回一个新的闭包函数：
```go
func makeHandler(fn func(http.ResponseWriter, *http.Request, string)) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		title := r.URL.Path[lenPath:]
		if !titleValidator.MatchString(title) {
			http.NotFound(w, r)
			return
		}
		fn(w, r, title)
	}
}
```
- 闭包封闭了函数变量 `fn` 来构造其返回值。但在此之前，它先用 `titleValidator.MatchString(title)` 验证输入标题 `title` 的有效性。如果标题包含了字母和数字以外的字符，就触发 NotFound 错误（例如：尝试 `localhost:8080/view/page++`）。`viewHandler`，`editHandler` 和 `saveHandler` 都是传入 `main()` 中 `makeHandler` 的参数，类型必须都与 `fn` 相同。
- `viewHandler` 尝试按标题读取文本文件，这是通过调用 `load()` 函数完成的，它会构建文件名并用 `ioutil.ReadFile` 读取内容。如果文件存在，其内容会存入字符串中。一个指向 `Page` 结构体的指针按字面量被创建：`&Page{Title: title, Body: body}`。

  另外，该值和表示没有 error 的 nil 值一起返回给调用者。然后在 `renderTemplate` 中将该结构体与模板对象整合。

  万一发生错误，也就是说 wiki 页面在磁盘上不存在，错误会被返回给 `viewHandler`，此时会自动重定向，跳转请求对应标题的编辑页面。

- `editHandler` 基本上也差不多：尝试读取文件，如果存在则用“编辑”模板来渲染；万一发生错误，创建一个新的包含指定标题的 `Page` 对象并渲染。
- 当在编辑页面点击“保存”按钮时，触发保存页面内容的动作。按钮须放在 html 表单中，它开头是这样的：
  ```html
  <form action="/save/{{.Title |html}}" method="POST">
  ```
  
  这意味着，当提交表单到类似 `http://localhost/save/{Title}` 这样的 URL 格式时，一个 POST 请求被发往网页服务器。针对这样的 URL 我们已经定义好了处理函数：`saveHandler()`。在 request 对象上调用 `FormValue()` 方法，可以提取名称为 body 的文本域内容，用这些信息构造一个 `Page` 对象，然后尝试通过调用 `save()` 方法保存其内容。万一运行失败，执行 `http.Error` 以将错误显示到浏览器。如果保存成功，重定向浏览器到该页的阅读页面。`save()` 函数非常简单，利用 `ioutil.WriteFile()`，写入 `Page` 结构体的 `Body` 字段到文件 `filename` 中，之后会被用于模板替换占位符 `{{printf "%s" .Body |html}}`。

## 15.7 探索 template 包

（template 包的文档可以在 [https://golang.org/pkg/text/template/](https://golang.org/pkg/text/template/) 找到。）

在前一章节，我们使用 template 对象把数据结构整合到 HTML 模板中。这项技术确实对网页应用程序非常有用，然而模板是一项更为通用的技术方案：数据驱动的模板被创建出来，以生成文本输出。HTML 仅是其中的一种特定使用案例。

模板通过与数据结构的整合来生成，通常为结构体或其切片。当数据项传递给 `tmpl.Execute()` ，它用其中的元素进行替换， 动态地重写某一小段文本。**只有被导出的数据项**才可以被整合进模板中。可以在 `{{` 和 `}}` 中加入数据求值或控制结构。数据项可以是值或指针，接口隐藏了他们的差异。

### 15.7.1 字段替换：`{{.FieldName}}`

要在模板中包含某个字段的内容，使用双花括号括起以点（`.`）开头的字段名。例如，假设 `Name` 是某个结构体的字段，其值要在被模板整合时替换，则在模板中使用文本 `{{.Name}}`。当 `Name` 是 map 的键时这么做也是可行的。要创建一个新的 Template 对象，调用 `template.New`，其字符串参数可以指定模板的名称。正如 [15.5节](15.5.md) 出现过的，`Parse` 方法通过解析模板定义字符串，生成模板的内部表示。当使用包含模板定义字符串的文件时，将文件路径传递给 `ParseFiles` 来解析。解析过程如产生错误，这两个函数第二个返回值 error != nil。最后通过 `Execute` 方法，数据结构中的内容与模板整合，并将结果写入方法的第一个参数中，其类型为 `io.Writer`。再一次地，可能会有 error 返回。以下程序演示了这些步骤，输出通过 `os.Stdout` 被写到控制台。

示例 15.13 [template_field.go](examples/chapter_15/template_field.go)

```go
package main

import (
	"fmt"
	"os"
	"text/template"
)

type Person struct {
	Name string
	nonExportedAgeField string
}

func main() {
	t := template.New("hello")
	t, _ = t.Parse("hello {{.Name}}!")
	p := Person{Name: "Mary", nonExportedAgeField: "31"}
	if err := t.Execute(os.Stdout, p); err != nil {
		fmt.Println("There was an error:", err.Error())
	}
}
```

输出：`hello Mary!`

数据结构中包含一个未导出的字段，当我们尝试把它整合到类似这样的定义字符串：
```go
t, _ = t.Parse("your age is {{.nonExportedAgeField}}!")
```
会产生错误：
```
There was an error: template: nonexported template hello:1: can’t evaluate field nonExportedAgeField in type main.Person.
```

如果只是想简单地把 `Execute()` 方法的第二个参数用于替换，使用 `{{.}}`。

当在浏览器环境中进行这些步骤，应首先使用 `html` 过滤器来过滤内容，例如 `{{html .}}`， 或者对 `FieldName` 过滤：`{{ .FieldName |html }}`。

`|html` 这部分代码，是请求模板引擎在输出 `FieldName` 的结果前把值传递给 html 格式化器，它会执行 HTML 字符转义（例如把 `>` 替换为 `&gt;`）。这可以避免用户输入数据破坏 HTML 文档结构。

### 15.7.2 验证模板格式

为了确保模板定义语法是正确的，使用 `Must` 函数处理 `Parse` 的返回结果。在下面的例子中 `tOK` 是正确的模板， `tErr` 验证时发生错误，会导致运行时 panic。

示例 15.14 [template_validation.go](examples/chapter_15/template_validation.go)

```go
package main

import (
	"text/template"
	"fmt"
)

func main() {
	tOk := template.New("ok")
	//a valid template, so no panic with Must:
	template.Must(tOk.Parse("/* and a comment */ some static text: {{ .Name }}"))
	fmt.Println("The first one parsed OK.")
	fmt.Println("The next one ought to fail.")
	tErr := template.New("error_template")
	template.Must(tErr.Parse(" some static text {{ .Name }"))
}
```

输出：

	The first one parsed OK.
	The next one ought to fail.
	panic: template: error_template:1: unexpected "}" in operand

模板语法出现错误比较少见，可以使用 [13.3节](13.3.md) 概括的 `defer/recover` 机制来报告并纠正错误。

在代码中常见到这 3 个基本函数被串联使用：
```go
var strTempl = template.Must(template.New("TName").Parse(strTemplateHTML))
```

练习 15.7 [template_validation_recover.go](exercises/chapter_15/template_validation_recover.go)

在上述示例代码上实现 defer/recover 机制。

### 15.7.3 `If-else`

运行 `Execute` 产生的结果来自模板的输出，它包含静态文本，以及被 `{{}}` 包裹的称之为*管道*的文本。例如，运行这段代码（示例 15.15 [pipline1.go](examples/chapter_15/pipeline1.go)）：
```go
t := template.New("template test")
t = template.Must(t.Parse("This is just static text. \n{{\"This is pipeline data - because it is evaluated within the double braces.\"}} {{`So is this, but within reverse quotes.`}}\n"))
t.Execute(os.Stdout, nil)
```

输出结果为：

	This is just static text.
	This is pipeline data—because it is evaluated within the double braces. So is this, but within reverse quotes.

现在我们可以对管道数据的输出结果用 `if-else-end` 设置条件约束：如果管道是空的，类似于：
```html
{{if ``}} Will not print. {{end}}
```
那么 `if` 条件的求值结果为 `false`，不会有输出内容。但如果是这样：
```html
{{if `anything`}} Print IF part. {{else}} Print ELSE part.{{end}}
```
会输出 `Print IF part.`。以下程序演示了这点：

示例 15.16 [template_ifelse.go](examples/chapter_15/template_ifelse.go)

```go
package main

import (
	"os"
	"text/template"
)

func main() {
	tEmpty := template.New("template test")
	tEmpty = template.Must(tEmpty.Parse("Empty pipeline if demo: {{if ``}} Will not print. {{end}}\n")) //empty pipeline following if
	tEmpty.Execute(os.Stdout, nil)

	tWithValue := template.New("template test")
	tWithValue = template.Must(tWithValue.Parse("Non empty pipeline if demo: {{if `anything`}} Will print. {{end}}\n")) //non empty pipeline following if condition
	tWithValue.Execute(os.Stdout, nil)

	tIfElse := template.New("template test")
	tIfElse = template.Must(tIfElse.Parse("if-else demo: {{if `anything`}} Print IF part. {{else}} Print ELSE part.{{end}}\n")) //non empty pipeline following if condition
	tIfElse.Execute(os.Stdout, nil)
}
```

输出：

	Empty pipeline if demo:
	Non empty pipeline if demo: Will print.
	if-else demo: Print IF part.

### 15.7.4 点号和 `with-end`

点号（`.`）可以在 Go 模板中使用：其值 `{{.}}` 被设置为当前管道的值。

`with` 语句将点号设为管道的值。如果管道是空的，那么不管 `with-end` 块之间有什么，都会被忽略。在被嵌套时，点号根据最近的作用域取得值。以下程序演示了这点：

示例 15.17 [template_with_end.go](examples/chapter_15/template_with_end.go)

```go
package main

import (
	"os"
	"text/template"
)

func main() {
	t := template.New("test")
	t, _ = t.Parse("{{with `hello`}}{{.}}{{end}}!\n")
	t.Execute(os.Stdout, nil)

	t, _ = t.Parse("{{with `hello`}}{{.}} {{with `Mary`}}{{.}}{{end}}{{end}}!\n")
	t.Execute(os.Stdout, nil)
}
```

输出：

	hello!
	hello Mary!

### 15.7.5 模板变量 `$`

可以在模板内为管道设置本地变量，变量名以 `$` 符号作为前缀。变量名只能包含字母、数字和下划线。以下示例使用了多种形式的有效变量名。

示例 15.18 [template_variables.go](examples/chapter_15/template_variables.go)

```go
package main

import (
	"os"
	"text/template"
)

func main() {
	t := template.New("test")
	t = template.Must(t.Parse("{{with $3 := `hello`}}{{$3}}{{end}}!\n"))
	t.Execute(os.Stdout, nil)

	t = template.Must(t.Parse("{{with $x3 := `hola`}}{{$x3}}{{end}}!\n"))
	t.Execute(os.Stdout, nil)

	t = template.Must(t.Parse("{{with $x_1 := `hey`}}{{$x_1}} {{.}} {{$x_1}}{{end}}!\n"))
	t.Execute(os.Stdout, nil)
}
```

输出：

	hello!
	hola!
	hey hey hey!

### 15.7.6 `range-end`

`range-end` 结构格式为：`{{range pipeline}} T1 {{else}} T0 {{end}}`。

`range` 被用于在集合上迭代：管道的值必须是数组、切片或 map。如果管道的值长度为零，点号的值不受影响，且执行 `T0`；否则，点号被设置为数组、切片或 map 内元素的值，并执行 `T1`。

如果模板为：
```html
{{range .}}
{{.}}
{{end}}
```
那么执行代码：
```go
s := []int{1,2,3,4}
t.Execute(os.Stdout, s)
```
会输出：
```
1
2
3
4
```

如需更实用的示例，请参考 [20.7节](20.7.md)，来自 App Engine 数据库的数据通过模板来显示：
```html
{{range .}}
	{{with .Author}}
		<p><b>{{html .}}</b> wrote:</p>
	{{else}}
		<p>An anonymous person wrote:</p>
	{{end}}
	<pre>{{html .Content}}</pre>
	<pre>{{html .Date}}</pre>
{{end}}
```

这里 `range .` 在结构体切片上迭代，每次都包含 `Author`、`Content` 和 `Date` 字段。

### 15.7.7 模板预定义函数

也有一些可以在模板代码中使用的预定义函数，例如 `printf` 函数工作方式类似于 `fmt.Sprintf`：

示例 15.19 [predefined_functions.go](examples/chapter_15/predefined_functions.go)

```go
package main

import (
	"os"
	"text/template"
)

func main() {
	t := template.New("test")
	t = template.Must(t.Parse("{{with $x := `hello`}}{{printf `%s %s` $x `Mary`}}{{end}}!\n"))
	t.Execute(os.Stdout, nil)
}
```
输出 `hello Mary!`。

预定义函数也在 [15.6节](15.6.md) 中使用：`{{ printf "%s" .Body|html}}`，否则字节切片 `Body` 会作为数字序列打印出来。

## 15.8 精巧的多功能网页服务器

为进一步深入理解 `http` 包以及如何构建网页服务器功能，让我们来学习和体会下面的例子：先列出代码，然后给出不同功能的实现方法，程序输出显示在表格中。

示例 15.20 [elaborated_webserver.go](examples/chapter_15/elaborated_webserver.go)

```go
package main

import (
	"bytes"
	"expvar"
	"flag"
	"fmt"
	"io"
	"log"
	"net/http"
	"os"
	"strconv"
)

// hello world, the web server
var helloRequests = expvar.NewInt("hello-requests")

// flags:
var webroot = flag.String("root", "/home/user", "web root directory")

// simple flag server
var booleanflag = flag.Bool("boolean", true, "another flag for testing")

// Simple counter server. POSTing to it will set the value.
type Counter struct {
	n int
}

// a channel
type Chan chan int

func main() {
	flag.Parse()
	http.Handle("/", http.HandlerFunc(Logger))
	http.Handle("/go/hello", http.HandlerFunc(HelloServer))
	// The counter is published as a variable directly.
	ctr := new(Counter)
	expvar.Publish("counter", ctr)
	http.Handle("/counter", ctr)
	// http.Handle("/go/", http.FileServer(http.Dir("/tmp"))) // uses the OS filesystem
	http.Handle("/go/", http.StripPrefix("/go/", http.FileServer(http.Dir(*webroot))))
	http.Handle("/flags", http.HandlerFunc(FlagServer))
	http.Handle("/args", http.HandlerFunc(ArgServer))
	http.Handle("/chan", ChanCreate())
	http.Handle("/date", http.HandlerFunc(DateServer))
	err := http.ListenAndServe(":12345", nil)
	if err != nil {
		log.Panicln("ListenAndServe:", err)
	}
}

func Logger(w http.ResponseWriter, req *http.Request) {
	log.Print(req.URL.String())
	w.WriteHeader(404)
	w.Write([]byte("oops"))
}

func HelloServer(w http.ResponseWriter, req *http.Request) {
	helloRequests.Add(1)
	io.WriteString(w, "hello, world!\n")
}

// This makes Counter satisfy the expvar.Var interface, so we can export
// it directly.
func (ctr *Counter) String() string { return fmt.Sprintf("%d", ctr.n) }

func (ctr *Counter) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	switch req.Method {
	case "GET": // increment n
		ctr.n++
	case "POST": // set n to posted value
		buf := new(bytes.Buffer)
		io.Copy(buf, req.Body)
		body := buf.String()
		if n, err := strconv.Atoi(body); err != nil {
			fmt.Fprintf(w, "bad POST: %v\nbody: [%v]\n", err, body)
		} else {
			ctr.n = n
			fmt.Fprint(w, "counter reset\n")
		}
	}
	fmt.Fprintf(w, "counter = %d\n", ctr.n)
}

func FlagServer(w http.ResponseWriter, req *http.Request) {
	w.Header().Set("Content-Type", "text/plain; charset=utf-8")
	fmt.Fprint(w, "Flags:\n")
	flag.VisitAll(func(f *flag.Flag) {
		if f.Value.String() != f.DefValue {
			fmt.Fprintf(w, "%s = %s [default = %s]\n", f.Name, f.Value.String(), f.DefValue)
		} else {
			fmt.Fprintf(w, "%s = %s\n", f.Name, f.Value.String())
		}
	})
}

// simple argument server
func ArgServer(w http.ResponseWriter, req *http.Request) {
	for _, s := range os.Args {
		fmt.Fprint(w, s, " ")
	}
}

func ChanCreate() Chan {
	c := make(Chan)
	go func(c Chan) {
		for x := 0; ; x++ {
			c <- x
		}
	}(c)
	return c
}

func (ch Chan) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	io.WriteString(w, fmt.Sprintf("channel send #%d\n", <-ch))
}

// exec a program, redirecting output
func DateServer(rw http.ResponseWriter, req *http.Request) {
	rw.Header().Set("Content-Type", "text/plain; charset=utf-8")
	r, w, err := os.Pipe()
	if err != nil {
		fmt.Fprintf(rw, "pipe: %s\n", err)
		return
	}

	p, err := os.StartProcess("/bin/date", []string{"date"}, &os.ProcAttr{Files: []*os.File{nil, w, w}})
	defer r.Close()
	w.Close()
	if err != nil {
		fmt.Fprintf(rw, "fork/exec: %s\n", err)
		return
	}
	defer p.Release()
	io.Copy(rw, r)
	wait, err := p.Wait()
	if err != nil {
		fmt.Fprintf(rw, "wait: %s\n", err)
		return
	}
	if !wait.Exited() {
		fmt.Fprintf(rw, "date: %v\n", wait)
		return
	}
}
```

处理函数 | 调用 URL | 浏览器获得响应
--------|----------|---------------
Logger | http://localhost:12345/ （根） | oops

`Logger` 处理函数用 `w.WriteHeader(404)` 来输出 “404 Not Found”头部。

此项技术通常很有用，无论何时服务器执行代码产生错误，都可以应用类似这样的代码：
```go
if err != nil {
	w.WriteHeader(400)
	return
}
```

另外利用 `logger` 包的函数，针对每个请求在服务器端命令行打印日期、时间和 URL。

处理函数 | 调用 URL | 浏览器获得响应
--------|----------|---------------
HelloServer | http://localhost:12345/go/hello | hello, world!

包 `expvar` 可以创建（Int，Float 和 String 类型）变量，并将它们发布为公共变量。它会在 HTTP URL `/debug/vars` 上以 JSON 格式公布。通常它被用于服务器操作计数。`helloRequests` 就是这样一个 `int64` 变量，该处理函数对其加 1，然后写入“hello world!”到浏览器。

处理函数 | 调用 URL | 浏览器获得响应
--------|----------|---------------
Counter | http://localhost:12345/counter | counter = 1
Counter | 刷新（GET 请求） | counter = 2

计数器对象 `ctr` 有一个 `String()` 方法，所以它实现了 `expvar.Var` 接口。这使其可以被发布，尽管它是一个结构体。`ServeHTTP` 函数使 `ctr` 成为处理器，因为它的签名正确实现了 `http.Handler` 接口。


处理函数 | 调用 URL | 浏览器获得响应
--------|----------|---------------
FileServer | http://localhost:12345/go/ggg.html | 404 page not found

`FileServer(root FileSystem) Handler` 返回一个处理器，它以 `root` 作为根，用文件系统的内容响应 HTTP 请求。要获得操作系统的文件系统，用 `http.Dir`，例如：
```go
http.Handle("/go/", http.FileServer(http.Dir("/tmp")))
```

处理函数 | 调用 URL | 浏览器获得响应
--------|----------|---------------
FlagServer | http://localhost:12345/flags | Flags: boolean = true root = /home/rsc

该处理函数使用了 `flag` 包。`VisitAll` 函数迭代所有的标签（flag），打印它们的名称、值和默认值（当不同于“值”时）。

处理函数 | 调用 URL | 浏览器获得响应
--------|----------|---------------
ArgServer | http://localhost:12345/args | ./elaborated_webserver.exe

该处理函数迭代 `os.Args` 以打印出所有的命令行参数。如果没有指定则只有程序名称（可执行程序的路径）会被打印出来。

处理函数 | 调用 URL | 浏览器获得响应
--------|----------|---------------
Channel | http://localhost:12345/chan | channel send #1
Channel | 刷新 | channel send #2

每当有新请求到达，通道的 `ServeHTTP` 方法从通道获取下一个整数并显示。由此可见，网页服务器可以从通道中获取要发送的响应，它可以由另一个函数产生（甚至是客户端）。下面的代码片段正是一个这样的处理函数，但会在 30 秒后超时：
```go
func ChanResponse(w http.ResponseWriter, req *http.Request) {
	timeout := make (chan bool)
	go func () {
		time.Sleep(30e9)
		timeout <- true
	}()
	select {
	case msg := <-messages:
		io.WriteString(w, msg)
	case stop := <-timeout:
		return
	}
}
```

处理函数 | 调用 URL | 浏览器获得响应
--------|----------|---------------
DateServer | http://localhost:12345/date | 显示当前时间（由于是调用 /bin/date，仅在 Unix 下有效）

可能的输出：`Thu Sep 8 12:41:09 CEST 2011`。

`os.Pipe()` 返回一对相关联的 `File`，从 `r` 读取数据，返回已读取的字节数来自于 `w` 的写入。函数返回这两个文件和错误，如果有的话：
```go
func Pipe() (r *File, w *File, err error)
```

## 15.9 用 rpc 实现远程过程调用

Go 程序之间可以使用 `net/rpc` 包实现相互通信，这是另一种客户端-服务器应用场景。它提供了一种方便的途径，通过网络连接调用远程函数。当然，仅当程序运行在不同机器上时，这项技术才实用。`rpc` 包建立在 `gob` 包之上（见 [12.11节](12.11.md)），实现了自动编码/解码传输的跨网络方法调用。

服务器端需要注册一个对象实例，与其类型名一起，使之成为一项可见的服务：它允许远程客户端跨越网络或其他 I/O 连接访问此对象已导出的方法。总之就是在网络上暴露类型的方法。

`rpc` 包使用了 http 和 tcp 协议，以及用于数据传输的 `gob` 包。服务器端可以注册多个不同类型的对象（服务），但同一类型的多个对象会产生错误。

我们讨论一个简单的例子：定义一个类型 `Args` 及其方法 `Multiply`，完美地置于单独的包中。方法必须返回可能的错误。

示例15.21 [rpc_objects.go](examples/chapter_15/rpc/rpc_objects.go)
```go
package rpc_objects

import "net"

type Args struct {
	N, M int
}

func (t *Args) Multiply(args *Args, reply *int) net.Error {
	*reply = args.N * args.M
	return nil
}
```

（**译注：Go 当前版本要求此方法返回类型为 `error`，以上示例中返回 `net.Error` 已无法通过编译，见更新后的[rpc_objects.go](examples/chapter_15/rpc_updated/rpc_objects/rpc_objects.go)。**）

服务器端产生一个 `rpc_objects.Args` 类型的对象 `calc`，并用 `rpc.Register(object)` 注册。调用 `HandleHTTP()`，然后用 `net.Listen` 在指定的地址上启动监听。也可以按名称来注册对象，例如：`rpc.RegisterName("Calculator", calc)`。

以协程启动 `http.Serve(listener, nil)` 后，会为每一个进入 `listener` 的 HTTP 连接创建新的服务线程。我们必须用诸如 `time.Sleep(1000e9)` 来使服务器在一段时间内保持运行状态。

示例 15.22 [rpc_server.go](examples/chapter_15/rpc/rpc_server.go)
```go
package main

import (
	"net/http"
	"log"
	"net"
	"net/rpc"
	"time"
	"./rpc_objects"
)

func main() {
	calc := new(rpc_objects.Args)
	rpc.Register(calc)
	rpc.HandleHTTP()
	listener, e := net.Listen("tcp", "localhost:1234")
	if e != nil {
		log.Fatal("Starting RPC-server -listen error:", e)
	}
	go http.Serve(listener, nil)
	time.Sleep(1000e9)
}
```

输出：

	Starting Process E:/Go/GoBoek/code_examples/chapter_14/rpc_server.exe ...
	** 5 秒后： **
	End Process exit status 0

客户端必须知晓对象类型及其方法的定义。执行 `rpc.DialHTTP()` 连接到服务器后，就可以用 `client.Call("Type.Method", args, &reply)` 调用远程对象的方法。`Type` 是远程对象的类型名，`Method` 是要调用的方法，`args` 是用 Args 类型初始化的对象，`reply` 是一个必须事先声明的变量，方法调用产生的结果将存入其中。

示例 15.23 [rpc_client.go](examples/chapter_15/rpc/rpc_client.go)
```go
package main

import (
	"fmt"
	"log"
	"net/rpc"
	"./rpc_objects"
)

const serverAddress = "localhost"

func main() {
	client, err := rpc.DialHTTP("tcp", serverAddress + ":1234")
	if err != nil {
		log.Fatal("Error dialing:", err)
	}
	// Synchronous call
	args := &rpc_objects.Args{7, 8}
	var reply int
	err = client.Call("Args.Multiply", args, &reply)
	if err != nil {
		log.Fatal("Args error:", err)
	}
	fmt.Printf("Args: %d * %d = %d", args.N, args.M, reply)
}
```

先启动服务器，再运行客户端，然后就能得到如下输出结果：

	Starting Process E:/Go/GoBoek/code_examples/chapter_14/rpc_client.exe ...
	
	Args: 7 * 8 = 56
	End Process exit status 0

该远程调用以同步方式进行，它会等待服务器返回结果。也可使用如下方式异步地执行调用：
```go
call1 := client.Go("Args.Multiply", args, &reply, nil)
replyCall := <- call1.Done
```

如果最后一个参数值为 `nil` ，调用完成后会创建一个新的通道。

如果你有一个以 root 管理员身份运行的 Go 服务器，想要以不同的用户身份运行某部分代码，Brad Fitz 利用 `rpc` 写的 `go-runas` 包可以完成任务：[https://github.com/bradfitz/go-runas](https://github.com/bradfitz/go-runas)。我们将会在 19 章看到一个完整的项目，它是一个使用了 `rpc` 的应用程序。

## 15.10 基于网络的通道 netchan

备注：Go 团队决定改进并重新打造 `netchan` 包的现有版本，它已被移至 `old/netchan`。`old/` 目录用于存放过时的包代码，它们不会成为 Go 1.x 的一部分。本节仅出于向后兼容性讨论 `netchan` 包的概念。

一项和 `rpc` 密切相关的技术是基于网络的通道。类似 14 章所使用的通道都是本地的，它们仅存在于被执行的机器内存空间中。`netchan` 包实现了类型安全的网络化通道：它允许一个通道两端出现由网络连接的不同计算机。其实现原理是，在其中一台机器上将传输数据发送到通道中，那么就可以被另一台计算机上同类型的通道接收。一个导出器（`exporter`）会按名称发布（一组）通道。导入器（`importer`）连接到导出的机器，并按名称导入这些通道。之后，两台机器就可按通常的方式来使用通道。网络通道不是同步的，它们类似于带缓存的通道。

发送端示例代码如下：

```go
exp, err := netchan.NewExporter("tcp", "netchanserver.mydomain.com:1234")
if err != nil {
	log.Fatalf("Error making Exporter: %v", err)
}
ch := make(chan myType)
err := exp.Export("sendmyType", ch, netchan.Send)
if err != nil {
	log.Fatalf("Send Error: %v", err)
}
```

接收端示例代码如下：

```go
imp, err := netchan.NewImporter("tcp", "netchanserver.mydomain.com:1234")
if err != nil {
	log.Fatalf("Error making Importer: %v", err)
}
ch := make(chan myType)
err = imp.Import("sendmyType", ch, netchan.Receive)
if err != nil {
	log.Fatalf("Receive Error: %v", err)
}
```

## 15.11 与 websocket 通信

备注：Go 团队决定从 Go 1 起，将 `websocket`  包移出 Go 标准库，转移到 `code.google.com/p/go` 下的子项目 `websocket`，同时预计近期将做重大更改。

`import "websocket"` 这行要改成：

```go
import websocket "code.google.com/p/go/websocket"
```

与 `http` 协议相反，`websocket` 是通过客户端与服务器之间的对话，建立的基于单个持久连接的协议。然而在其他方面，其功能几乎与 `http` 相同。在示例 15.24 中，我们有一个典型的 `websocket` 服务器，他会自启动并监听 `websocket` 客户端的连入。示例 15.25 演示了 5 秒后会终止的客户端代码。当连接到来时，服务器先打印 `new connection`，当客户端停止时，服务器打印 `EOF => closing connection`。

示例 15.24 [websocket_server.go](examples/chapter_15/websocket_server.go)

```go
package main

import (
	"fmt"
	"net/http"
	"websocket"
)

func server(ws *websocket.Conn) {
	fmt.Printf("new connection\n")
	buf := make([]byte, 100)
	for {
		if _, err := ws.Read(buf); err != nil {
			fmt.Printf("%s", err.Error())
			break
		}
	}
	fmt.Printf(" => closing connection\n")
	ws.Close()
}

func main() {
	http.Handle("/websocket", websocket.Handler(server))
	err := http.ListenAndServe(":12345", nil)
	if err != nil {
		panic("ListenAndServe: " + err.Error())
	}
}
```

示例 15.25 [websocket_client.go](examples/chapter_15/websocket_client.go)

```go
package main

import (
	"fmt"
	"time"
	"websocket"
)

func main() {
	ws, err := websocket.Dial("ws://localhost:12345/websocket", "",
		"http://localhost/")
	if err != nil {
		panic("Dial: " + err.Error())
	}
	go readFromServer(ws)
	time.Sleep(5e9)
    ws.Close()
}

func readFromServer(ws *websocket.Conn) {
	buf := make([]byte, 1000)
	for {
		if _, err := ws.Read(buf); err != nil {
			fmt.Printf("%s\n", err.Error())
			break
		}
	}
}
```

## 15.12 用 smtp 发送邮件

`smtp` 包实现了用于发送邮件的“简单邮件传输协议”（Simple Mail Transfer Protocol）。它有一个 `Client` 类型，代表一个连接到 SMTP 服务器的客户端：

- `Dial` 方法返回一个已连接到 SMTP 服务器的客户端 `Client`
- 设置 `Mail`（from，即发件人）和 `Rcpt`（to，即收件人）
- `Data` 方法返回一个用于写入数据的 `Writer`，这里利用 `buf.WriteTo(wc)` 写入

示例 15.26 [smtp.go](examples/chapter_15/smtp.go)

```go
package main

import (
	"bytes"
	"log"
	"net/smtp"
)

func main() {
	// Connect to the remote SMTP server.
	client, err := smtp.Dial("mail.example.com:25")
	if err != nil {
		log.Fatal(err)
	}
	// Set the sender and recipient.
	client.Mail("sender@example.org")
	client.Rcpt("recipient@example.net")
	// Send the email body.
	wc, err := client.Data()
	if err != nil {
		log.Fatal(err)
	}
	defer wc.Close()
	buf := bytes.NewBufferString("This is the email body.")
	if _, err = buf.WriteTo(wc); err != nil {
		log.Fatal(err)
	}
}
```

如果需要认证，或有多个收件人时，也可以用 `SendMail` 函数发送。它连接到地址为 `addr` 的服务器；如果可以，切换到 TLS（“传输层安全”加密和认证协议），并用 PLAIN 机制认证；然后以 `from` 作为发件人，`to` 作为收件人列表，`msg` 作为邮件内容，发出一封邮件：

```go
func SendMail(addr string, a Auth, from string, to []string, msg []byte) error
```

示例 15.27 [smtp_auth.go](examples/chapter_15/smtp_auth.go)

```go
package main

import (
	"log"
	"net/smtp"
)

func main() {
	// Set up authentication information.
	auth := smtp.PlainAuth(
		"",
		"user@example.com",
		"password",
		"mail.example.com",
	)
	// Connect to the server, authenticate, set the sender and recipient,
	// and send the email all in one step.
	err := smtp.SendMail(
		"mail.example.com:25",
		auth,
		"sender@example.org",
		[]string{"recipient@example.net"},
		[]byte("This is the email body."),
	)
	if err != nil {
		log.Fatal(err)
	}
}
```