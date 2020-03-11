# 第八章　Goroutines和Channels

并发程序指同时进行多个任务的程序，随着硬件的发展，并发程序变得越来越重要。Web服务器会一次处理成千上万的请求。平板电脑和手机app在渲染用户画面同时还会后台执行各种计算任务和网络请求。即使是传统的批处理问题——读取数据、计算、写输出，现在也会用并发来隐藏掉I/O的操作延迟以充分利用现代计算机设备的多个核心。计算机的性能每年都在以非线性的速度增长。

Go语言中的并发程序可以用两种手段来实现。本章讲解`goroutine`和`channel`，其支持“顺序通信进程”（communicating sequential processes）或被简称为*CSP*。*CSP*是一种现代的并发编程模型，在这种编程模型中值会在不同的运行实例（`goroutine`）中传递，尽管大多数情况下仍然是被限制在单一实例中。第9章覆盖更为传统的并发模型：多线程共享内存，如果你在其它的主流语言中写过并发程序的话可能会更熟悉一些。第9章也会深入介绍一些并发程序带来的风险和陷阱。

尽管Go对并发的支持是众多强力特性之一，但跟踪调试并发程序还是很困难，在线性程序中形成的直觉往往还会使我们误入歧途。如果这是读者第一次接触并发，推荐稍微多花一些时间来思考这两个章节中的样例。
## 8.1. Goroutines

在Go语言中，每一个并发的执行单元叫作一个`goroutine`。设想这里的一个程序有两个函数，一个函数做计算，另一个输出结果，假设两个函数没有相互之间的调用关系。一个线性的程序会先调用其中的一个函数，然后再调用另一个。如果程序中包含多个`goroutine`，对两个函数的调用则可能发生在同一时刻。马上就会看到这样的一个程序。

如果你使用过操作系统或者其它语言提供的线程，那么你可以简单地把`goroutine`类比作一个线程，这样你就可以写出一些正确的程序了。`goroutine`和线程的本质区别会在9.8节中讲。

当一个程序启动时，其主函数即在一个单独的`goroutine`中运行，我们叫它`main goroutine`。新的`goroutine`会用`go`语句来创建。在语法上，`go`语句是一个普通的函数或方法调用前加上关键字`go`。`go`语句会使其语句中的函数在一个新创建的`goroutine`中运行。而`go`语句本身会迅速地完成。

```go
f()    // call f(); wait for it to return
go f() // create a new goroutine that calls f(); don't wait
```

下面的例子，`main goroutine`将计算*菲波那契数列*的第45个元素值。由于计算函数使用低效的递归，所以会运行相当长时间，在此期间我们想让用户看到一个可见的标识来表明程序依然在正常运行，所以来做一个动画的小图标：

*gopl.io/ch8/spinner*

```go
func main() {
	go spinner(100 * time.Millisecond)
	const n = 45
	fibN := fib(n) // slow
	fmt.Printf("\rFibonacci(%d) = %d\n", n, fibN)
}

func spinner(delay time.Duration) {
	for {
		for _, r := range `-\|/` {
			fmt.Printf("\r%c", r)
			time.Sleep(delay)
		}
	}
}

func fib(x int) int {
	if x < 2 {
		return x
	}
	return fib(x-1) + fib(x-2)
}
```

动画显示了几秒之后，`fib(45)`的调用成功地返回，并且打印结果：

```go
Fibonacci(45) = 1134903170
```

然后主函数返回。主函数返回时，所有的`goroutine`都会被直接打断，程序退出。除了从主函数退出或者直接终止程序之外，没有其它的编程方法能够让一个`goroutine`来打断另一个的执行，但是之后可以看到一种方式来实现这个目的，通过`goroutine`之间的通信来让一个`goroutine`请求其它的`goroutine`，并让被请求的`goroutine`自行结束执行。

留意一下这里的两个独立的单元是如何进行组合的，`spinning`和菲波那契的计算。分别在独立的函数中，但两个函数会同时执行。
## 8.2. 示例: 并发的Clock服务

网络编程是并发大显身手的一个领域，由于服务器是最典型的需要同时处理很多连接的程序，这些连接一般来自于彼此独立的客户端。在本小节中，我们会讲解go语言的`net`包，这个包提供编写一个网络客户端或者服务器程序的基本组件，无论两者间通信是使用TCP、UDP或者Unix domain sockets。在第一章中我们使用过的`net/http`包里的方法，也算是`net`包的一部分。

我们的第一个例子是一个顺序执行的时钟服务器，它会每隔一秒钟将当前时间写到客户端：

*gopl.io/ch8/clock1*

```go
// Clock1 is a TCP server that periodically writes the time.
package main

import (
	"io"
	"log"
	"net"
	"time"
)

func main() {
	listener, err := net.Listen("tcp", "localhost:8000")
	if err != nil {
		log.Fatal(err)
	}

	for {
		conn, err := listener.Accept()
		if err != nil {
			log.Print(err) // e.g., connection aborted
			continue
		}
		handleConn(conn) // handle one connection at a time
	}
}

func handleConn(c net.Conn) {
	defer c.Close()
	for {
		_, err := io.WriteString(c, time.Now().Format("15:04:05\n"))
		if err != nil {
			return // e.g., client disconnected
		}
		time.Sleep(1 * time.Second)
	}
}
```

`Listen`函数创建了一个`net.Listener`的对象，这个对象会监听一个网络端口上到来的连接，在这个例子里我们用的是TCP的`localhost:8000`端口。`listener`对象的`Accept`方法会直接阻塞，直到一个新的连接被创建，然后会返回一个`net.Conn`对象来表示这个连接。

`handleConn`函数会处理一个完整的客户端连接。在一个`for`死循环中，用`time.Now()`获取当前时刻，然后写到客户端。由于`net.Conn`实现了`io.Writer`接口，我们可以直接向其写入内容。这个死循环会一直执行，直到写入失败。最可能的原因是客户端主动断开连接。这种情况下`handleConn`函数会用`defer`调用关闭服务器侧的连接，然后返回到主函数，继续等待下一个连接请求。

`time.Time.Format`方法提供了一种格式化日期和时间信息的方式。它的参数是一个格式化模板，标识如何来格式化时间，而这个格式化模板限定为`Mon Jan 2 03:04:05PM 2006 UTC-0700`。有8个部分（周几、月份、一个月的第几天……）。可以以任意的形式来组合前面这个模板；出现在模板中的部分会作为参考来对时间格式进行输出。在上面的例子中我们只用到了小时、分钟和秒。`time`包里定义了很多标准时间格式，比如`time.RFC1123`。在进行格式化的逆向操作`time.Parse`时，也会用到同样的策略。

> （译注：这是go语言和其它语言相比比较奇葩的一个地方。你需要记住格式化字符串是1月2日下午3点4分5秒零六年UTC-0700，而不像其它语言那样`Y-m-d H:i:s`一样，当然了这里可以用1234567的方式来记忆，倒是也不麻烦。）

为了连接例子里的服务器，我们需要一个客户端程序，比如`netcat`这个工具（`nc`命令），这个工具可以用来执行网络连接操作。

```go
$ go build gopl.io/ch8/clock1
$ ./clock1 &
$ nc localhost 8000
13:58:54
13:58:55
13:58:56
13:58:57
^C
```

客户端将服务器发来的时间显示了出来，我们用`Control+C`来中断客户端的执行，在Unix系统上，你会看到`^C`这样的响应。如果你的系统没有装`nc`这个工具，你可以用`telnet`来实现同样的效果，或者也可以用我们下面的这个用go写的简单的`telnet`程序，用`net.Dial`就可以简单地创建一个TCP连接：

*gopl.io/ch8/netcat1*

```go
// Netcat1 is a read-only TCP client.
package main

import (
	"io"
	"log"
	"net"
	"os"
)

func main() {
	conn, err := net.Dial("tcp", "localhost:8000")
	if err != nil {
		log.Fatal(err)
	}
	defer conn.Close()
	mustCopy(os.Stdout, conn)
}

func mustCopy(dst io.Writer, src io.Reader) {
	if _, err := io.Copy(dst, src); err != nil {
		log.Fatal(err)
	}
}
```

这个程序会从连接中读取数据，并将读到的内容写到标准输出中，直到遇到`end of file`的条件或者发生错误。`mustCopy`这个函数我们在本节的几个例子中都会用到。让我们同时运行两个客户端来进行一个测试，这里可以开两个终端窗口，下面左边的是其中的一个的输出，右边的是另一个的输出：

```go
$ go build gopl.io/ch8/netcat1
$ ./netcat1
13:58:54                               $ ./netcat1
13:58:55
13:58:56
^C
                                       13:58:57
                                       13:58:58
                                       13:58:59
                                       ^C
$ killall clock1
```

`killall`命令是一个Unix命令行工具，可以用给定的进程名来杀掉所有名字匹配的进程。

第二个客户端必须等待第一个客户端完成工作，这样服务端才能继续向后执行；因为我们这里的服务器程序同一时间只能处理一个客户端连接。我们这里对服务端程序做一点小改动，使其支持并发：在`handleConn`函数调用的地方增加`go`关键字，让每一次`handleConn`的调用都进入一个独立的`goroutine`。

<i>gopl.io/ch8/clock2</i>

```go
for {
	conn, err := listener.Accept()
	if err != nil {
		log.Print(err) // e.g., connection aborted
		continue
	}
	go handleConn(conn) // handle connections concurrently
}

```

现在多个客户端可以同时接收到时间了：

```go
$ go build gopl.io/ch8/clock2
$ ./clock2 &
$ go build gopl.io/ch8/netcat1
$ ./netcat1
14:02:54                               $ ./netcat1
14:02:55                               14:02:55
14:02:56                               14:02:56
14:02:57                               ^C
14:02:58
14:02:59                               $ ./netcat1
14:03:00                               14:03:00
14:03:01                               14:03:01
^C                                     14:03:02
                                       ^C
$ killall clock2
```

**练习 8.1：** 修改`clock2`来支持传入参数作为端口号，然后写一个`clockwall`的程序，这个程序可以同时与多个`clock`服务器通信，从多个服务器中读取时间，并且在一个表格中一次显示所有服务器传回的结果，类似于你在某些办公室里看到的时钟墙。如果你有地理学上分布式的服务器可以用的话，让这些服务器跑在不同的机器上面；或者在同一台机器上跑多个不同的实例，这些实例监听不同的端口，假装自己在不同的时区。像下面这样：

```go
$ TZ=US/Eastern    ./clock2 -port 8010 &
$ TZ=Asia/Tokyo    ./clock2 -port 8020 &
$ TZ=Europe/London ./clock2 -port 8030 &
$ clockwall NewYork=localhost:8010 Tokyo=localhost:8020 London=localhost:8030
```

```go
// clockwall.go
package main

import (
	"bufio"
	"fmt"
	"io"
	"log"
	"net"
	"os"
	"strings"
	"time"
)

type clock struct {
	name, host string
}

func (c *clock) watch(w io.Writer, r io.Reader) {
	s := bufio.NewScanner(r)
	for s.Scan() {
		fmt.Fprintf(w, "%s: %s\n", c.name, s.Text())
	}
	fmt.Println(c.name, "done")
	if s.Err() != nil {
		log.Printf("can't read from %s: %s", c.name, s.Err())
	}
}

func main() {
	if len(os.Args) == 1 {
		fmt.Fprintln(os.Stderr, "usage: clockwall NAME=HOST ...")
		os.Exit(1)
	}
	clocks := make([]*clock, 0)
	for _, a := range os.Args[1:] {
		fields := strings.Split(a, "=")
		if len(fields) != 2 {
			fmt.Fprintf(os.Stderr, "bad arg: %s\n", a)
			os.Exit(1)
		}
		clocks = append(clocks, &clock{fields[0], fields[1]})
	}
	for _, c := range clocks {
		conn, err := net.Dial("tcp", c.host)
		if err != nil {
			log.Fatal(err)
		}
		defer conn.Close()
		go c.watch(os.Stdout, conn)
	}
	// Sleep while other goroutines do the work.
	for {
		time.Sleep(time.Minute)
	}
}
```

**练习 8.2：** 实现一个并发FTP服务器。服务器应该解析客户端发来的一些命令，比如`cd`命令来切换目录，`ls`来列出目录内文件，`get`和`send`来传输文件，`close`来关闭连接。你可以用标准的`ftp`命令来作为客户端，或者也可以自己实现一个。

```go
// ftpd.go
package main

import (
	"bufio"
	"bytes"
	"flag"
	"fmt"
	"io"
	"log"
	"net"
	"os"
	"strconv"
	"strings"
)

type conn struct {
	rw           net.Conn // "Protocol Interpreter" connection
	dataHostPort string
	prevCmd      string
	pasvListener net.Listener
	cmdErr       error // Saved command connection write error.
	binary       bool
}

func NewConn(cmdConn net.Conn) *conn {
	return &conn{rw: cmdConn}
}

// hostPortToFTP returns a comma-separated, FTP-style address suitable for
// replying to the PASV command.
func hostPortToFTP(hostport string) (addr string, err error) {
	host, portStr, err := net.SplitHostPort(hostport)
	if err != nil {
		return "", err
	}
	ipAddr, err := net.ResolveIPAddr("ip4", host)
	if err != nil {
		return "", err
	}
	port, err := strconv.ParseInt(portStr, 10, 64)
	if err != nil {
		return "", err
	}
	ip := ipAddr.IP.To4()
	s := fmt.Sprintf("%d,%d,%d,%d,%d,%d", ip[0], ip[1], ip[2], ip[3], port/256, port%256)
	return s, nil
}

func hostPortFromFTP(address string) (string, error) {
	var a, b, c, d byte
	var p1, p2 int
	_, err := fmt.Sscanf(address, "%d,%d,%d,%d,%d,%d", &a, &b, &c, &d, &p1, &p2)
	if err != nil {
		return "", err
	}
	return fmt.Sprintf("%d.%d.%d.%d:%d", a, b, c, d, 256*p1+p2), nil
}

type logPairs map[string]interface{}

func (c *conn) log(pairs logPairs) {
	b := &bytes.Buffer{}
	fmt.Fprintf(b, "addr=%s", c.rw.RemoteAddr().String())
	for k, v := range pairs {
		fmt.Fprintf(b, " %s=%s", k, v)
	}
	log.Print(b.String())
}

func (c *conn) dataConn() (conn io.ReadWriteCloser, err error) {
	switch c.prevCmd {
	case "PORT":
		conn, err = net.Dial("tcp", c.dataHostPort)
		if err != nil {
			return nil, err
		}
	case "PASV":
		conn, err = c.pasvListener.Accept()
		if err != nil {
			return nil, err
		}
	default:
		return nil, fmt.Errorf("previous command not PASV or PORT")
	}
	return conn, nil
}

// list prints file information to a data connection specified by the
// immediately preceding PASV or PORT command.
func (c *conn) list(args []string) {
	var filename string
	switch len(args) {
	case 0:
		filename = "."
	case 1:
		filename = args[0]
	default:
		c.writeln("501 Too many arguments.")
		return
	}
	file, err := os.Open(filename)
	if err != nil {
		c.writeln("550 File not found.")
		return
	}
	c.writeln("150 Here comes the directory listing.")
	w, err := c.dataConn()
	if err != nil {
		c.writeln("425 Can't open data connection.")
		return
	}
	defer w.Close()
	stat, err := file.Stat()
	if err != nil {
		c.log(logPairs{"cmd": "LIST", "err": err})
		c.writeln("450 Requested file action not taken. File unavailable.")
	}
	// TODO: Print more than just the filenames.
	if stat.IsDir() {
		filenames, err := file.Readdirnames(0)
		if err != nil {
			c.writeln("550 Can't read directory.")
			return
		}
		for _, f := range filenames {
			_, err = fmt.Fprint(w, f, c.lineEnding())
			if err != nil {
				c.log(logPairs{"cmd": "LIST", "err": err})
				c.writeln("426 Connection closed: transfer aborted.")
				return
			}
		}
	} else {
		_, err = fmt.Fprint(w, filename, c.lineEnding())
		if err != nil {
			c.log(logPairs{"cmd": "LIST", "err": err})
			c.writeln("426 Connection closed: transfer aborted.")
			return
		}
	}
	c.writeln("226 Closing data connection. List successful.")
}

func (c *conn) writeln(s ...interface{}) {
	if c.cmdErr != nil {
		return
	}
	s = append(s, "\r\n")
	_, c.cmdErr = fmt.Fprint(c.rw, s...)
}

func (c *conn) lineEnding() string {
	if c.binary {
		return "\n"
	} else {
		return "\r\n"
	}
}

func (c *conn) CmdErr() error {
	return c.cmdErr
}

func (c *conn) Close() error {
	err := c.rw.Close()
	if err != nil {
		c.log(logPairs{"err": fmt.Errorf("closing command connection: %s", err)})
	}
	return err
}

func (c *conn) pasv(args []string) {
	if len(args) > 0 {
		c.writeln("501 Too many arguments.")
		return
	}
	var firstError error
	storeFirstError := func(err error) {
		if firstError == nil {
			firstError = err
		}
	}
	var err error
	c.pasvListener, err = net.Listen("tcp4", "")
	storeFirstError(err)
	_, port, err := net.SplitHostPort(c.pasvListener.Addr().String())
	storeFirstError(err)
	ip, _, err := net.SplitHostPort(c.rw.LocalAddr().String())
	storeFirstError(err)
	addr, err := hostPortToFTP(fmt.Sprintf("%s:%s", ip, port))
	storeFirstError(err)
	if firstError != nil {
		c.pasvListener.Close()
		c.pasvListener = nil
		c.log(logPairs{"cmd": "PASV", "err": err})
		c.writeln("451 Requested action aborted. Local error in processing.")
		return
	}
	// DJB recommends putting an extra character before the address.
	c.writeln(fmt.Sprintf("227 =%s", addr))
}

func (c *conn) port(args []string) {
	if len(args) != 1 {
		c.writeln("501 Usage: PORT a,b,c,d,p1,p2")
		return
	}
	var err error
	c.dataHostPort, err = hostPortFromFTP(args[0])
	if err != nil {
		c.log(logPairs{"cmd": "PORT", "err": err})
		c.writeln("501 Can't parse address.")
		return
	}
	c.writeln("200 PORT command successful.")
}

func (c *conn) type_(args []string) {
	if len(args) < 1 || len(args) > 2 {
		c.writeln("501 Usage: TYPE takes 1 or 2 arguments.")
		return
	}
	switch strings.ToUpper(strings.Join(args, " ")) {
	case "A", "A N":
		c.binary = false
	case "I", "L 8":
		c.binary = true
	default:
		c.writeln("504 Unsupported type. Supported types: A, A N, I, L 8.")
		return
	}
	c.writeln("200 TYPE set")
}

func (c *conn) stru(args []string) {
	if len(args) != 1 {
		c.writeln("501 Usage: STRU F")
		return
	}
	if args[0] != "F" {
		c.writeln("504 Only file structure is supported")
		return
	}
	c.writeln("200 STRU set")
}

func (c *conn) retr(args []string) {
	if len(args) != 1 {
		c.writeln("501 Usage: RETR filename")
		return
	}
	filename := args[0]
	file, err := os.Open(filename)
	if err != nil {
		c.log(logPairs{"cmd": "RETR", "err": err})
		c.writeln("550 File not found.")
		return
	}
	c.writeln("150 File ok. Sending.")
	conn, err := c.dataConn()
	if err != nil {
		c.writeln("425 Can't open data connection")
		return
	}
	defer conn.Close()
	if c.binary {
		_, err := io.Copy(conn, file)
		if err != nil {
			c.log(logPairs{"cmd": "RETR", "err": err})
			c.writeln("450 File unavailable.")
			return
		}
	} else {
		// Convert line endings LF -> CRLF.
		r := bufio.NewReader(file)
		w := bufio.NewWriter(conn)
		for {
			line, isPrefix, err := r.ReadLine()
			if err != nil {
				if err == io.EOF {
					break
				}
				c.log(logPairs{"cmd": "RETR", "err": err})
				c.writeln("450 File unavailable.")
				return
			}
			w.Write(line)
			if !isPrefix {
				w.Write([]byte("\r\n"))
			}
		}
		w.Flush()
	}
	c.writeln("226 Transfer complete.")
}

func (c *conn) stor(args []string) {
	if len(args) != 1 {
		c.writeln("501 Usage: STOR filename")
		return
	}
	filename := args[0]
	file, err := os.Create(filename)
	if err != nil {
		c.log(logPairs{"cmd": "STOR", "err": err})
		c.writeln("550 File can't be created.")
		return
	}
	c.writeln("150 Ok to send data.")
	conn, err := c.dataConn()
	if err != nil {
		c.writeln("425 Can't open data connection")
		return
	}
	defer conn.Close()
	_, err = io.Copy(file, conn)
	if err != nil {
		c.log(logPairs{"cmd": "RETR", "err": err})
		c.writeln("450 File unavailable.")
		return
	}
	c.writeln("226 Transfer complete.")
}

func (c *conn) run() {
	c.writeln("220 Ready.")
	s := bufio.NewScanner(c.rw)
	var cmd string
	var args []string
	for s.Scan() {
		if c.CmdErr() != nil {
			c.log(logPairs{"err": fmt.Errorf("command connection: %s", c.CmdErr())})
			return
		}
		fields := strings.Fields(s.Text())
		if len(fields) == 0 {
			continue
		}
		cmd = strings.ToUpper(fields[0])
		args = nil
		if len(fields) > 1 {
			args = fields[1:]
		}
		switch cmd {
		case "LIST":
			c.list(args)
		case "NOOP":
			c.writeln("200 Ready.")
		case "PASV":
			c.pasv(args)
		case "PORT":
			c.port(args)
		case "QUIT":
			c.writeln("221 Goodbye.")
			return
		case "RETR":
			c.retr(args)
		case "STOR":
			c.stor(args)
		case "STRU":
			c.stru(args)
		case "SYST":
			// DJB recommends always replying with this string, to be
			// consistent with other servers and avoid weird fallback modes in
			// some clients.
			c.writeln("215 UNIX Type: L8")
		case "TYPE":
			c.type_(args)
		case "USER":
			c.writeln("230 Login successful.")
		default:
			c.writeln(fmt.Sprintf("502 Command %q not implemented.", cmd))
		}
		// Cleanup PASV listeners if they go unused.
		if cmd != "PASV" && c.pasvListener != nil {
			c.pasvListener.Close()
			c.pasvListener = nil
		}
		c.prevCmd = cmd
	}
	if s.Err() != nil {
		c.log(logPairs{"err": fmt.Errorf("scanning commands: %s", s.Err())})
	}
}

func main() {
	var port int
	flag.IntVar(&port, "port", 8000, "listen port")

	ln, err := net.Listen("tcp4", fmt.Sprintf(":%d", port))
	if err != nil {
		log.Fatal("Opening main listener:", err)
	}
	for {
		c, err := ln.Accept()
		if err != nil {
			log.Print("Accepting new connection:", err)
		}
		go NewConn(c).run()
	}
}
```

## 8.3. 示例: 并发的Echo服务

`clock`服务器每一个连接都会起一个`goroutine`。在本节中我们会创建一个`echo`服务器，这个服务在每个连接中会有多个`goroutine`。大多数`echo`服务仅仅会返回他们读取到的内容，就像下面这个简单的`handleConn`函数所做的一样：

```go
func handleConn(c net.Conn) {
	io.Copy(c, c) // NOTE: ignoring errors
	c.Close()
}
```

一个更有意思的`echo`服务应该模拟一个实际的`echo`的“回响”，并且一开始要用大写`HELLO`来表示“声音很大”，之后经过一小段延迟返回一个有所缓和的`Hello`，然后一个全小写字母的`hello`表示声音渐渐变小直至消失，像下面这个版本的`handleConn`(译注：笑看作者脑洞大开)：

*gopl.io/ch8/reverb1*

```go
func echo(c net.Conn, shout string, delay time.Duration) {
	fmt.Fprintln(c, "\t", strings.ToUpper(shout))
	time.Sleep(delay)
	fmt.Fprintln(c, "\t", shout)
	time.Sleep(delay)
	fmt.Fprintln(c, "\t", strings.ToLower(shout))
}

func handleConn(c net.Conn) {
	input := bufio.NewScanner(c)
	for input.Scan() {
		echo(c, input.Text(), 1*time.Second)
	}
	// NOTE: ignoring potential errors from input.Err()
	c.Close()
}
```

我们需要升级我们的客户端程序，这样它就可以发送终端的输入到服务器，并把服务端的返回输出到终端上，这使我们有了使用并发的另一个好机会：

*gopl.io/ch8/netcat2*

```go
func main() {
	conn, err := net.Dial("tcp", "localhost:8000")
	if err != nil {
		log.Fatal(err)
	}
	defer conn.Close()
	go mustCopy(os.Stdout, conn)
	mustCopy(conn, os.Stdin)
}
```

当`main goroutine`从标准输入流中读取内容并将其发送给服务器时，另一个`goroutine`会读取并打印服务端的响应。当`main goroutine`碰到输入终止时，例如，用户在终端中按了`Control-D`(`^D`)，在windows上是`Control-Z`，这时程序就会被终止，尽管其它`goroutine`中还有进行中的任务。（在8.4.1中引入了`channels`后我们会明白如何让程序等待两边都结束。）

下面这个会话中，客户端的输入是左对齐的，服务端的响应会用缩进来区别显示。
客户端会向服务器“喊三次话”：

```go
$ go build gopl.io/ch8/reverb1
$ ./reverb1 &
$ go build gopl.io/ch8/netcat2
$ ./netcat2
Hello?
    HELLO?
    Hello?
    hello?
Is there anybody there?
    IS THERE ANYBODY THERE?
Yooo-hooo!
    Is there anybody there?
    is there anybody there?
    YOOO-HOOO!
    Yooo-hooo!
yooo-hooo!
^D
$ killall reverb1
```

注意客户端的第三次`shout`在前一个`shout`处理完成之前一直没有被处理，这貌似看起来不是特别“现实”。真实世界里的回响应该是会由三次`shout`的回声组合而成的。为了模拟真实世界的回响，我们需要更多的`goroutine`来做这件事情。这样我们就再一次地需要`go`这个关键词了，这次我们用它来调用`echo`：

*gopl.io/ch8/reverb2*

```go
func handleConn(c net.Conn) {
	input := bufio.NewScanner(c)
	for input.Scan() {
		go echo(c, input.Text(), 1*time.Second)
	}
	// NOTE: ignoring potential errors from input.Err()
	c.Close()
}
```

`go`后跟的函数的参数会在`go`语句自身执行时被求值；因此`input.Text()`会在`main goroutine`中被求值。
现在回响是并发并且会按时间来覆盖掉其它响应了：

```go
$ go build gopl.io/ch8/reverb2
$ ./reverb2 &
$ ./netcat2
Is there anybody there?
    IS THERE ANYBODY THERE?
Yooo-hooo!
    Is there anybody there?
    YOOO-HOOO!
    is there anybody there?
    Yooo-hooo!
    yooo-hooo!
^D
$ killall reverb2
```

让服务使用并发不只是处理多个客户端的请求，甚至在处理单个连接时也可能会用到，就像我们上面的两个`go`关键词的用法。然而在我们使用`go`关键词的同时，需要慎重地考虑`net.Conn`中的方法在并发地调用时是否安全，事实上对于大多数类型来说也确实不安全。我们会在下一章中详细地探讨并发安全性。
## 8.4. Channels

如果说`goroutine`是Go语言程序的并发体的话，那么`channels`则是它们之间的通信机制。一个`channel`是一个通信机制，它可以让一个`goroutine`通过它给另一个`goroutine`发送值信息。每个`channel`都有一个特殊的类型，也就是`channels`可发送数据的类型。一个可以发送`int`类型数据的`channel`一般写为`chan int`。

使用内置的`make`函数，我们可以创建一个`channel`：

```Go
ch := make(chan int) // ch has type 'chan int'
```

和`map`类似，`channel`也对应一个`make`创建的底层数据结构的引用。当我们复制一个`channel`或用于函数参数传递时，我们只是拷贝了一个`channel`引用，因此调用者和被调用者将引用同一个`channel`对象。和其它的引用类型一样，`channel`的零值也是`nil`。

两个相同类型的`channel`可以使用`==`运算符比较。如果两个`channel`引用的是相同的对象，那么比较的结果为真。一个`channel`也可以和`nil`进行比较。

一个`channel`有发送和接受两个主要操作，都是通信行为。一个发送语句将一个值从一个`goroutine`通过`channel`发送到另一个执行接收操作的`goroutine`。发送和接收两个操作都使用`<-`运算符。在发送语句中，`<-`运算符分割channel和要发送的值。在接收语句中，`<-`运算符写在channel对象之前。一个不使用接收结果的接收操作也是合法的。

```Go
ch <- x  // a send statement
x = <-ch // a receive expression in an assignment statement
<-ch     // a receive statement; result is discarded
```

`Channel`还支持`close`操作，用于关闭`channel`，随后对基于该`channel`的任何发送操作都将导致`panic`异常。对一个已经被`close`过的`channel`进行接收操作依然可以接受到之前已经成功发送的数据；如果`channel`中已经没有数据的话将产生一个零值的数据。

使用内置的`close`函数就可以关闭一个`channel`：

```Go
close(ch)
```

以最简单方式调用`make`函数创建的是一个无缓存的`channel`，但是我们也可以指定第二个整型参数，对应`channel`的容量。如果`channel`的容量大于零，那么该`channel`就是带缓存的`channel`。

```Go
ch = make(chan int)    // unbuffered channel
ch = make(chan int, 0) // unbuffered channel
ch = make(chan int, 3) // buffered channel with capacity 3
```

我们将先讨论无缓存的`channel`，然后在8.4.4节讨论带缓存的`channel`。

### 8.4.1. 不带缓存的Channels

一个基于无缓存`Channels`的发送操作将导致发送者`goroutine`阻塞，直到另一个`goroutine`在相同的`Channels`上执行接收操作，当发送的值通过`Channels`成功传输之后，两个`goroutine`可以继续执行后面的语句。反之，如果接收操作先发生，那么接收者`goroutine`也将阻塞，直到有另一个`goroutine`在相同的`Channels`上执行发送操作。

基于无缓存`Channels`的发送和接收操作将导致两个`goroutine`做一次同步操作。因为这个原因，无缓存`Channels`有时候也被称为同步`Channels`。当通过一个无缓存`Channels`发送数据时，接收者收到数据发生在再次唤醒唤醒发送者`goroutine`之前（译注：*happens before*，这是Go语言并发内存模型的一个关键术语！）。

在讨论并发编程时，当我们说x事件在y事件之前发生（*happens before*），我们并不是说x事件在时间上比y时间更早；我们要表达的意思是要保证在此之前的事件都已经完成了，例如在此之前的更新某些变量的操作已经完成，你可以放心依赖这些已完成的事件了。

当我们说`x`事件既不是在`y`事件之前发生也不是在`y`事件之后发生，我们就说`x`事件和`y`事件是并发的。这并不是意味着`x`事件和`y`事件就一定是同时发生的，我们只是不能确定这两个事件发生的先后顺序。在下一章中我们将看到，当两个`goroutine`并发访问了相同的变量时，我们有必要保证某些事件的执行顺序，以避免出现某些并发问题。

在8.3节的客户端程序，它在主`goroutine`中（译注：就是执行`main`函数的`goroutine`）将标准输入复制到`server`，因此当客户端程序关闭标准输入时，后台`goroutine`可能依然在工作。我们需要让主`goroutine`等待后台`goroutine`完成工作后再退出，我们使用了一个`channel`来同步两个`goroutine`：

*gopl.io/ch8/netcat3*

```Go
func main() {
	conn, err := net.Dial("tcp", "localhost:8000")
	if err != nil {
		log.Fatal(err)
	}
	done := make(chan struct{})
	go func() {
		io.Copy(os.Stdout, conn) // NOTE: ignoring errors
		log.Println("done")
		done <- struct{}{} // signal the main goroutine
	}()
	mustCopy(conn, os.Stdin)
	conn.Close()
	<-done // wait for background goroutine to finish
}
```

当用户关闭了标准输入，主`goroutine`中的`mustCopy`函数调用将返回，然后调用`conn.Close()`关闭读和写方向的网络连接。关闭网络连接中的写方向的连接将导致`server`程序收到一个文件（end-of-file）结束的信号。关闭网络连接中读方向的连接将导致后台`goroutine`的`io.Copy`函数调用返回一个“`read from closed connection`”（“从关闭的连接读”）类似的错误，因此我们临时移除了错误日志语句；在练习8.3将会提供一个更好的解决方案。（需要注意的是`go`语句调用了一个函数字面量，这是Go语言中启动`goroutine`常用的形式。）

在后台`goroutine`返回之前，它先打印一个日志信息，然后向`done`对应的`channel`发送一个值。主`goroutine`在退出前先等待从`done`对应的`channel`接收一个值。因此，总是可以在程序退出前正确输出“`done`”消息。

基于`channels`发送消息有两个重要方面。首先每个消息都有一个值，但是有时候通讯的事实和发生的时刻也同样重要。当我们更希望强调通讯发生的时刻时，我们将它称为**消息事件**。有些消息事件并不携带额外的信息，它仅仅是用作两个`goroutine`之间的同步，这时候我们可以用`struct{}`空结构体作为`channels`元素的类型，虽然也可以使用`bool`或`int`类型实现同样的功能，`done <- 1`语句也比`done <- struct{}{}`更短。

**练习 8.3：** 在`netcat3`例子中，`conn`虽然是一个`interface`类型的值，但是其底层真实类型是`*net.TCPConn`，代表一个TCP连接。一个TCP连接有读和写两个部分，可以使用`CloseRead`和`CloseWrite`方法分别关闭它们。修改`netcat3`的主`goroutine`代码，只关闭网络连接中写的部分，这样的话后台`goroutine`可以在标准输入被关闭后继续打印从`reverb1`服务器传回的数据。（要在`reverb2`服务器也完成同样的功能是比较困难的；参考**练习 8.4**。）

```go
// netcat.go
package main

import (
	"io"
	"log"
	"net"
	"os"
)

func main() {
	addr, err := net.ResolveTCPAddr("tcp", "localhost:8000")
	if err != nil {
		log.Fatal(err)
	}
	conn, err := net.DialTCP("tcp", nil, addr)
	if err != nil {
		log.Fatal(err)
	}
	done := make(chan struct{})
	go func() {
		io.Copy(os.Stdout, conn) // NOTE: ignoring errors
		log.Println("done")
		done <- struct{}{} // signal the main goroutine
	}()
	mustCopy(conn, os.Stdin)
	conn.CloseWrite()
	<-done // wait for background goroutine to finish
}

func mustCopy(dst io.Writer, src io.Reader) {
	if _, err := io.Copy(dst, src); err != nil {
		if err == io.EOF {
			return
		}
		log.Fatal(err)
	}
}
```

### 8.4.2. 串联的Channels（Pipeline）

`Channels`也可以用于将多个`goroutine`连接在一起，一个`Channel`的输出作为下一个`Channel`的输入。这种串联的`Channels`就是所谓的管道（`pipeline`）。下面的程序用两个`channels`将三个`goroutine`串联起来，如图8.1所示。

![Channels - 图1](images\ch8-01.png)

第一个`goroutine`是一个计数器，用于生成`0、1、2、……`形式的整数序列，然后通过`channel`将该整数序列发送给第二个`goroutine`；第二个`goroutine`是一个求平方的程序，对收到的每个整数求平方，然后将平方后的结果通过第二个`channel`发送给第三个`goroutine`；第三个`goroutine`是一个打印程序，打印收到的每个整数。为了保持例子清晰，我们有意选择了非常简单的函数，当然三个`goroutine`的计算很简单，在现实中确实没有必要为如此简单的运算构建三个`goroutine`。

*gopl.io/ch8/pipeline1*

```Go
func main() {
	naturals := make(chan int)
	squares := make(chan int)

	// Counter
	go func() {
		for x := 0; ; x++ {
			naturals <- x
		}
	}()

	// Squarer
	go func() {
		for {
			x := <-naturals
			squares <- x * x
		}
	}()

	// Printer (in main goroutine)
	for {
		fmt.Println(<-squares)
	}
}
```

如您所料，上面的程序将生成`0、1、4、9、……`形式的无穷数列。像这样的串联`Channels`的管道（Pipelines）可以用在需要长时间运行的服务中，每个长时间运行的`goroutine`可能会包含一个死循环，在不同`goroutine`的死循环内部使用串联的`Channels`来通信。但是，如果我们希望通过`Channels`只发送有限的数列该如何处理呢？

如果发送者知道，没有更多的值需要发送到`channel`的话，那么让接收者也能及时知道没有多余的值可接收将是有用的，因为接收者可以停止不必要的接收等待。这可以通过内置的`close`函数来关闭`channel`实现：

```Go
close(naturals)
```

当一个`channel`被关闭后，再向该`channel`发送数据将导致`panic`异常。当一个被关闭的`channel`中已经发送的数据都被成功接收后，后续的接收操作将不再阻塞，它们会立即返回一个零值。关闭上面例子中的`naturals`变量对应的`channel`并不能终止循环，它依然会收到一个永无休止的零值序列，然后将它们发送给打印者`goroutine`。

没有办法直接测试一个`channel`是否被关闭，但是接收操作有一个变体形式：它多接收一个结果，多接收的第二个结果是一个布尔值`ok`，`true`表示成功从`channels`接收到值，`false`表示`channels`已经被关闭并且里面没有值可接收。使用这个特性，我们可以修改`squarer`函数中的循环代码，当`naturals`对应的`channel`被关闭并没有值可接收时跳出循环，并且也关闭`squares`对应的`channel`.

```Go
// Squarer
go func() {
	for {
		x, ok := <-naturals
		if !ok {
			break // channel was closed and drained
		}
		squares <- x * x
	}
	close(squares)
}()
```

因为上面的语法是笨拙的，而且这种处理模式很常见，因此Go语言的`range`循环可直接在`channels`上面迭代。使用`range`循环是上面处理模式的简洁语法，它依次从`channel`接收数据，当`channel`被关闭并且没有值可接收时跳出循环。

在下面的改进中，我们的计数器`goroutine`只生成100个含数字的序列，然后关闭`naturals`对应的`channel`，这将导致计算平方数的`squarer`对应的`goroutine`可以正常终止循环并关闭`squares`对应的`channel`。（在一个更复杂的程序中，可以通过`defer`语句关闭对应的`channel`。）最后，主`goroutine`也可以正常终止循环并退出程序。

*gopl.io/ch8/pipeline2*

```Go
func main() {
	naturals := make(chan int)
	squares := make(chan int)

	// Counter
	go func() {
		for x := 0; x < 100; x++ {
			naturals <- x
		}
		close(naturals)
	}()

	// Squarer
	go func() {
		for x := range naturals {
			squares <- x * x
		}
		close(squares)
	}()

	// Printer (in main goroutine)
	for x := range squares {
		fmt.Println(x)
	}
}
```

其实你并不需要关闭每一个`channel`。只有当需要告诉接收者`goroutine`，所有的数据已经全部发送时才需要关闭`channel`。不管一个`channel`是否被关闭，当它没有被引用时将会被Go语言的垃圾自动回收器回收。（不要将关闭一个打开文件的操作和关闭一个`channel`操作混淆。对于每个打开的文件，都需要在不使用的时候调用对应的`Close`方法来关闭文件。）

试图重复关闭一个`channel`将导致`panic`异常，试图关闭一个`nil`值的`channel`也将导致`panic`异常。关闭一个`channels`还会触发一个广播机制，我们将在8.9节讨论。


### 8.4.3. 单方向的Channel

随着程序的增长，人们习惯于将大的函数拆分为小的函数。我们前面的例子中使用了三个`goroutine`，然后用两个`channels`来连接它们，它们都是`main`函数的局部变量。将三个`goroutine`拆分为以下三个函数是自然的想法：

```Go
func counter(out chan int)
func squarer(out, in chan int)
func printer(in chan int)
```

其中计算平方的`squarer`函数在两个串联`Channels`的中间，因此拥有两个`channel`类型的参数，一个用于输入一个用于输出。两个`channel`都拥有相同的类型，但是它们的使用方式相反：一个只用于接收，另一个只用于发送。参数的名字`in`和`out`已经明确表示了这个意图，但是并无法保证`squarer`函数向一个`in`参数对应的`channel`发送数据或者从一个`out`参数对应的`channel`接收数据。

这种场景是典型的。当一个`channel`作为一个函数参数时，它一般总是被专门用于只发送或者只接收。

为了表明这种意图并防止被滥用，Go语言的类型系统提供了单方向的`channel`类型，分别用于只发送或只接收的`channel`。类型`chan<- int`表示一个只发送`int`的`channel`，只能发送不能接收。相反，类型`<-chan int`表示一个只接收`int`的`channel`，只能接收不能发送。（箭头`<-`和关键字`chan`的相对位置表明了`channel`的方向。）这种限制将在编译期检测。

因为关闭操作只用于断言不再向`channel`发送新的数据，所以只有在发送者所在的`goroutine`才会调用`close`函数，因此对一个只接收的`channel`调用`close`将是一个编译错误。

这是改进的版本，这一次参数使用了单方向`channel`类型：

*gopl.io/ch8/pipeline3*

```Go
func counter(out chan<- int) {
	for x := 0; x < 100; x++ {
		out <- x
	}
	close(out)
}

func squarer(out chan<- int, in <-chan int) {
	for v := range in {
		out <- v * v
	}
	close(out)
}

func printer(in <-chan int) {
	for v := range in {
		fmt.Println(v)
	}
}

func main() {
	naturals := make(chan int)
	squares := make(chan int)
	go counter(naturals)
	go squarer(squares, naturals)
	printer(squares)
}
```

调用`counter(naturals)`时，`naturals`的类型将隐式地从`chan int`转换成`chan<- int`。调用`printer(squares)`也会导致相似的隐式转换，这一次是转换为`<-chan int`类型只接收型的`channel`。任何双向`channel`向单向`channel`变量的赋值操作都将导致该隐式转换。这里并没有反向转换的语法：也就是不能将一个类似`chan<- int`类型的单向型的`channel`转换为`chan int`类型的双向型的`channel`。


### 8.4.4. 带缓存的Channels

带缓存的`Channel`内部持有一个元素队列。队列的最大容量是在调用`make`函数创建`channel`时通过第二个参数指定的。下面的语句创建了一个可以持有三个字符串元素的带缓存`Channel`。图8.2是`ch`变量对应的`channel`的图形表示形式。

```Go
ch = make(chan string, 3)
```

![Channels - 图2](images\ch8-02.png)

向缓存`Channel`的发送操作就是向内部缓存队列的尾部插入元素，接收操作则是从队列的头部删除元素。如果内部缓存队列是满的，那么发送操作将阻塞直到因另一个`goroutine`执行接收操作而释放了新的队列空间。相反，如果`channel`是空的，接收操作将阻塞直到有另一个`goroutine`执行发送操作而向队列插入元素。

我们可以在无阻塞的情况下连续向新创建的channel发送三个值：

```Go
ch <- "A"
ch <- "B"
ch <- "C"
```

此刻，`channel`的内部缓存队列将是满的（图8.3），如果有第四个发送操作将发生阻塞。

![Channels - 图3](images\ch8-03.png)

如果我们接收一个值，

```Go
fmt.Println(<-ch) // "A"
```

那么`channel`的缓存队列将不是满的也不是空的（图8.4），因此对该`channel`执行的发送或接收操作都不会发生阻塞。通过这种方式，`channel`的缓存队列解耦了接收和发送的`goroutine`。

![Channels - 图4](images\ch8-04.png)

在某些特殊情况下，程序可能需要知道`channel`内部缓存的容量，可以用内置的`cap`函数获取：

```Go
fmt.Println(cap(ch)) // "3"
```

同样，对于内置的`len`函数，如果传入的是`channel`，那么将返回`channel`内部缓存队列中有效元素的个数。因为在并发程序中该信息会随着接收操作而失效，但是它对某些故障诊断和性能优化会有帮助。

```Go
fmt.Println(len(ch)) // "2"
```

在继续执行两次接收操作后channel内部的缓存队列将又成为空的，如果有第四个接收操作将发生阻塞：

```Go
fmt.Println(<-ch) // "B"
fmt.Println(<-ch) // "C"
```

在这个例子中，发送和接收操作都发生在同一个`goroutine`中，但是在真实的程序中它们一般由不同的`goroutine`执行。Go语言新手有时候会将一个带缓存的`channel`当作同一个`goroutine`中的队列使用，虽然语法看似简单，但实际上这是一个错误。`Channel`和`goroutine`的调度器机制是紧密相连的，如果没有其他`goroutine`从`channel`接收，发送者——或许是整个程序——将会面临永远阻塞的风险。如果你只是需要一个简单的队列，使用`slice`就可以了。

下面的例子展示了一个使用了带缓存`channel`的应用。它并发地向三个镜像站点发出请求，三个镜像站点分散在不同的地理位置。它们分别将收到的响应发送到带缓存`channel`，最后接收者只接收第一个收到的响应，也就是最快的那个响应。因此`mirroredQuery`函数可能在另外两个响应慢的镜像站点响应之前就返回了结果。（顺便说一下，多个`goroutines`并发地向同一个`channel`发送数据，或从同一个`channel`接收数据都是常见的用法。）

```Go
func mirroredQuery() string {
	responses := make(chan string, 3)
	go func() { responses <- request("asia.gopl.io") }()
	go func() { responses <- request("europe.gopl.io") }()
	go func() { responses <- request("americas.gopl.io") }()
	return <-responses // return the quickest response
}

func request(hostname string) (response string) { /* ... */ }
```

如果我们使用了无缓存的`channel`，那么两个慢的`goroutines`将会因为没有人接收而被永远卡住。这种情况，称为`goroutines`泄漏，这将是一个BUG。和垃圾变量不同，泄漏的`goroutines`并不会被自动回收，因此确保每个不再需要的`goroutine`能正常退出是重要的。

关于无缓存或带缓存`channels`之间的选择，或者是带缓存`channels`的容量大小的选择，都可能影响程序的正确性。无缓存`channel`更强地保证了每个发送操作与相应的同步接收操作；但是对于带缓存`channel`，这些操作是解耦的。同样，即使我们知道将要发送到一个`channel`的信息的数量上限，创建一个对应容量大小的带缓存`channel`也是不现实的，因为这要求在执行任何接收操作之前缓存所有已经发送的值。如果未能分配足够的缓存将导致程序死锁。

`Channel`的缓存也可能影响程序的性能。想象一家蛋糕店有三个厨师，一个烘焙，一个上糖衣，还有一个将每个蛋糕传递到它下一个厨师的生产线。在狭小的厨房空间环境，每个厨师在完成蛋糕后必须等待下一个厨师已经准备好接受它；这类似于在一个无缓存的`channel`上进行沟通。

如果在每个厨师之间有一个放置一个蛋糕的额外空间，那么每个厨师就可以将一个完成的蛋糕临时放在那里而马上进入下一个蛋糕的制作中；这类似于将`channel`的缓存队列的容量设置为`1`。只要每个厨师的平均工作效率相近，那么其中大部分的传输工作将是迅速的，个体之间细小的效率差异将在交接过程中弥补。如果厨师之间有更大的额外空间——也是就更大容量的缓存队列——将可以在不停止生产线的前提下消除更大的效率波动，例如一个厨师可以短暂地休息，然后再加快赶上进度而不影响其他人。

另一方面，如果生产线的前期阶段一直快于后续阶段，那么它们之间的缓存在大部分时间都将是满的。相反，如果后续阶段比前期阶段更快，那么它们之间的缓存在大部分时间都将是空的。对于这类场景，额外的缓存并没有带来任何好处。

生产线的隐喻对于理解`channels`和`goroutines`的工作机制是很有帮助的。例如，如果第二阶段是需要精心制作的复杂操作，一个厨师可能无法跟上第一个厨师的进度，或者是无法满足第三阶段厨师的需求。要解决这个问题，我们可以再雇佣另一个厨师来帮助完成第二阶段的工作，他执行相同的任务但是独立工作。这类似于基于相同的`channels`创建另一个独立的`goroutine`。

我们没有太多的空间展示全部细节，但是`gopl.io/ch8/cake`包模拟了这个蛋糕店，可以通过不同的参数调整。它还对上面提到的几种场景提供对应的基准测试（§11.4） 。


## 8.5. 并发的循环

本节中，我们会探索一些用来在并行时循环迭代的常见并发模型。我们会探究从全尺寸图片生成一些缩略图的问题。`gopl.io/ch8/thumbnail`包提供了`ImageFile`函数来帮我们拉伸图片。我们不会说明这个函数的实现，只需要从`gopl.io`下载它。

*gopl.io/ch8/thumbnail*

```go
package thumbnail

// ImageFile reads an image from infile and writes
// a thumbnail-size version of it in the same directory.
// It returns the generated file name, e.g., "foo.thumb.jpg".
func ImageFile(infile string) (string, error)
```

下面的程序会循环迭代一些图片文件名，并为每一张图片生成一个缩略图：

*gopl.io/ch8/thumbnail*

```go
// makeThumbnails makes thumbnails of the specified files.
func makeThumbnails(filenames []string) {
	for _, f := range filenames {
		if _, err := thumbnail.ImageFile(f); err != nil {
			log.Println(err)
		}
	}
}
```

显然我们处理文件的顺序无关紧要，因为每一个图片的拉伸操作和其它图片的处理操作都是彼此独立的。像这种子问题都是完全彼此独立的问题被叫做易并行问题（译注：embarrassingly parallel，直译的话更像是尴尬并行）。易并行问题是最容易被实现成并行的一类问题（废话），并且最能够享受到并发带来的好处，能够随着并行的规模线性地扩展。

下面让我们并行地执行这些操作，从而将文件`IO`的延迟隐藏掉，并用上多核`cpu`的计算能力来拉伸图像。我们的第一个并发程序只是使用了一个`go`关键字。这里我们先忽略掉错误，之后再进行处理。

```go
// NOTE: incorrect!
func makeThumbnails2(filenames []string) {
	for _, f := range filenames {
		go thumbnail.ImageFile(f) // NOTE: ignoring errors
	}
}
```

这个版本运行的实在有点太快，实际上，由于它比最早的版本使用的时间要短得多，即使当文件名的`slice`中只包含有一个元素。这就有点奇怪了，如果程序没有并发执行的话，那为什么一个并发的版本还是要快呢？答案其实是`makeThumbnails`在它还没有完成工作之前就已经返回了。它启动了所有的`goroutine`，每一个文件名对应一个，但没有等待它们一直到执行完毕。

没有什么直接的办法能够等待`goroutine`完成，但是我们可以改变`goroutine`里的代码让其能够将完成情况报告给外部的`goroutine`知晓，使用的方式是向一个共享的`channel`中发送事件。因为我们已经确切地知道有`len(filenames)`个内部`goroutine`，所以外部的`goroutine`只需要在返回之前对这些事件计数。

```go
// makeThumbnails3 makes thumbnails of the specified files in parallel.
func makeThumbnails3(filenames []string) {
	ch := make(chan struct{})
	for _, f := range filenames {
		go func(f string) {
			thumbnail.ImageFile(f) // NOTE: ignoring errors
			ch <- struct{}{}
		}(f)
	}
	// Wait for goroutines to complete.
	for range filenames {
		<-ch
	}
}
```

注意我们将f的值作为一个显式的变量传给了函数，而不是在循环的闭包中声明：

```go
for _, f := range filenames {
	go func() {
		thumbnail.ImageFile(f) // NOTE: incorrect!
		// ...
	}()
}
```

回忆一下之前在5.6.1节中，匿名函数中的循环变量快照问题。上面这个单独的变量`f`是被所有的匿名函数值所共享，且会被连续的循环迭代所更新的。当新的`goroutine`开始执行字面函数时，`for`循环可能已经更新了`f`并且开始了另一轮的迭代或者（更有可能的）已经结束了整个循环，所以当这些`goroutine`开始读取f的值时，它们所看到的值已经是`slice`的最后一个元素了。显式地添加这个参数，我们能够确保使用的`f`是当`go`语句执行时的“当前”那个`f`。

如果我们想要从每一个`worker goroutine`往主`goroutine`中返回值时该怎么办呢？当我们调用`thumbnail.ImageFile`创建文件失败的时候，它会返回一个错误。下一个版本的`makeThumbnails`会返回其在做拉伸操作时接收到的第一个错误：

```go
// makeThumbnails4 makes thumbnails for the specified files in parallel.
// It returns an error if any step failed.
func makeThumbnails4(filenames []string) error {
	errors := make(chan error)

	for _, f := range filenames {
		go func(f string) {
			_, err := thumbnail.ImageFile(f)
			errors <- err
		}(f)
	}

	for range filenames {
		if err := <-errors; err != nil {
			return err // NOTE: incorrect: goroutine leak!
		}
	}

	return nil
}
```

这个程序有一个微妙的bug。当它遇到第一个非`nil`的`error`时会直接将`error`返回到调用方，使得没有一个`goroutine`去排空`errors channel`。这样剩下的`worker goroutine`在向这个`channel`中发送值时，都会永远地阻塞下去，并且永远都不会退出。这种情况叫做`goroutine`泄露（§8.4.4），可能会导致整个程序卡住或者跑出`out of memory`的错误。

最简单的解决办法就是用一个具有合适大小的`buffered channel`，这样这些`worker goroutine`向`channel`中发送错误时就不会被阻塞。（一个可选的解决办法是创建一个另外的`goroutine`，当`main goroutine`返回第一个错误的同时去排空`channel`。）

下一个版本的`makeThumbnails`使用了一个`buffered channel`来返回生成的图片文件的名字，附带生成时的错误。

```go
// makeThumbnails5 makes thumbnails for the specified files in parallel.
// It returns the generated file names in an arbitrary order,
// or an error if any step failed.
func makeThumbnails5(filenames []string) (thumbfiles []string, err error) {
	type item struct {
		thumbfile string
		err       error
	}

	ch := make(chan item, len(filenames))
	for _, f := range filenames {
		go func(f string) {
			var it item
			it.thumbfile, it.err = thumbnail.ImageFile(f)
			ch <- it
		}(f)
	}

	for range filenames {
		it := <-ch
		if it.err != nil {
			return nil, it.err
		}
		thumbfiles = append(thumbfiles, it.thumbfile)
	}

	return thumbfiles, nil
}
```

我们最后一个版本的`makeThumbnails`返回了新文件们的大小总计数（bytes）。和前面的版本都不一样的一点是我们在这个版本里没有把文件名放在slice里，而是通过一个string的channel传过来，所以我们无法对循环的次数进行预测。

为了知道最后一个`goroutine`什么时候结束（最后一个结束并不一定是最后一个开始），我们需要一个递增的计数器，在每一个`goroutine`启动时加一，在`goroutine`退出时减一。这需要一种特殊的计数器，这个计数器需要在多个`goroutine`操作时做到安全并且提供在其减为零之前一直等待的一种方法。这种计数类型被称为`sync.WaitGroup`，下面的代码就用到了这种方法：

```go
// makeThumbnails6 makes thumbnails for each file received from the channel.
// It returns the number of bytes occupied by the files it creates.
func makeThumbnails6(filenames <-chan string) int64 {
	sizes := make(chan int64)
	var wg sync.WaitGroup // number of working goroutines
	for f := range filenames {
		wg.Add(1)
		// worker
		go func(f string) {
			defer wg.Done()
			thumb, err := thumbnail.ImageFile(f)
			if err != nil {
				log.Println(err)
				return
			}
			info, _ := os.Stat(thumb) // OK to ignore error
			sizes <- info.Size()
		}(f)
	}

	// closer
	go func() {
		wg.Wait()
		close(sizes)
	}()

	var total int64
	for size := range sizes {
		total += size
	}
	return total
}
```

注意`Add`和`Done`方法的不对称。`Add`是为计数器加一，必须在worker goroutine开始之前调用，而不是在`goroutine`中；否则的话我们没办法确定`Add`是在"closer" goroutine调用`Wait`之前被调用。并且`Add`还有一个参数，但`Done`却没有任何参数；其实它和`Add(-1)`是等价的。我们使用`defer`来确保计数器即使是在出错的情况下依然能够正确地被减掉。上面的程序代码结构是当我们使用并发循环，但又不知道迭代次数时很通常而且很地道的写法。

`sizes channel`携带了每一个文件的大小到main goroutine，在main goroutine中使用了range loop来计算总和。观察一下我们是怎样创建一个closer goroutine，并让其在所有worker goroutine们结束之后再关闭sizes channel的。两步操作：wait和close，必须是基于sizes的循环的并发。考虑一下另一种方案：如果等待操作被放在了main goroutine中，在循环之前，这样的话就永远都不会结束了，如果在循环之后，那么又变成了不可达的部分，因为没有任何东西去关闭这个channel，这个循环就永远都不会终止。

图8.5 表明了makethumbnails6函数中事件的序列。纵列表示goroutine。窄线段代表sleep，粗线段代表活动。斜线箭头代表用来同步两个goroutine的事件。时间向下流动。注意main goroutine是如何大部分的时间被唤醒执行其range循环，等待worker发送值或者closer来关闭channel的。

![并发的循环 - 图1](images\ch8-05.png)

**练习 8.4：** 修改reverb2服务器，在每一个连接中使用`sync.WaitGroup`来计数活跃的echo goroutine。当计数减为零时，关闭TCP连接的写入，像练习8.3中一样。验证一下你的修改版`netcat3`客户端会一直等待所有的并发“喊叫”完成，即使是在标准输入流已经关闭的情况下。

```go
// reverb.go
package main

import (
	"bufio"
	"fmt"
	"log"
	"net"
	"strings"
	"sync"
	"time"
)

func echo(c net.Conn, shout string, delay time.Duration, wg sync.WaitGroup) {
	fmt.Fprintln(c, "\t", strings.ToUpper(shout))
	time.Sleep(delay)
	fmt.Fprintln(c, "\t", shout)
	time.Sleep(delay)
	fmt.Fprintln(c, "\t", strings.ToLower(shout))
	wg.Done()
}

func handleConn(c net.Conn) {
	wg := sync.WaitGroup{}
	input := bufio.NewScanner(c)
	for input.Scan() {
		wg.Add(1)
		go echo(c, input.Text(), 1*time.Second, wg)
	}
	wg.Wait()
	// NOTE: ignoring potential errors from input.Err()
	c.Close()
}

func main() {
	l, err := net.Listen("tcp", "localhost:8000")
	if err != nil {
		log.Fatal(err)
	}
	for {
		conn, err := l.Accept()
		if err != nil {
			log.Print(err) // e.g., connection aborted
			continue
		}
		go handleConn(conn)
	}
}
```

**练习 8.5：** 使用一个已有的CPU绑定的顺序程序，比如在3.3节中我们写的`Mandelbrot`程序或者3.2节中的3-D `surface`计算程序，并将他们的主循环改为并发形式，使用`channel`来进行通信。在多核计算机上这个程序得到了多少速度上的改进？使用多少个`goroutine`是最合适的呢？

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
	"runtime"
	"sync"
	"time"
)

const (
	xmin, ymin, xmax, ymax = -2, -2, +2, +2
	width, height          = 1024, 1024
)

func main() {
	workers := runtime.GOMAXPROCS(-1)
	img := image.NewRGBA(image.Rect(0, 0, width, height))
	start := time.Now()
	wg := sync.WaitGroup{}
	rows := make(chan int, height)
	for row := 0; row < height; row++ {
		rows <- row
	}
	close(rows)
	for i := 0; i < workers; i++ {
		wg.Add(1)
		go func() {
			for py := range rows {
				y := float64(py)/height*(ymax-ymin) + ymin
				for px := 0; px < width; px++ {
					x := float64(px)/width*(xmax-xmin) + xmin
					// Image point (px, py) represents complex value z.
					z := complex(x, y)
					img.Set(px, py, newton(z))
				}
			}
			wg.Done()
		}()
	}
	wg.Wait()

	fmt.Println("rendered in:", time.Since(start))
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		png.Encode(w, img) // NOTE: ignoring errors
	})
	log.Fatal(http.ListenAndServe(":8080", nil))
}

// z^4 - 1 = 0
// 4z^3
//
// z' = z - f(x)/f'(x)
// z' = z - (z^4 - 1) / 4z^3
// z' = z - z/4 - z^3/4
// z' = (3z-z^3)/4
// OR z' -= (z-z^3)/4
func newton(z complex128) color.Color {
	iterations := 37
	for n := uint8(0); int(n) < iterations; n++ {
		z -= (z - 1/(z*z*z)) / 4
		if cmplx.Abs(cmplx.Pow(z, 4)-1) < 1e-6 {
			return color.Gray{255 - uint8(math.Log(float64(n))/math.Log(float64(iterations+0))*255)}
		}
	}
	return color.Black
}
```

## 8.6. 示例: 并发的Web爬虫

在5.6节中，我们做了一个简单的web爬虫，用`bfs`(广度优先)算法来抓取整个网站。在本节中，我们会让这个爬虫并行化，这样每一个彼此独立的抓取命令可以并行进行`IO`，最大化利用网络资源。`crawl`函数和`gopl.io/ch5/findlinks3`中的是一样的。

`gopl.io/ch8/crawl1`

```go
func crawl(url string) []string {
	fmt.Println(url)
	list, err := links.Extract(url)
	if err != nil {
		log.Print(err)
	}
	return list
}
```

主函数和5.6节中的`breadthFirst`(广度优先)类似。像之前一样，一个`worklist`是一个记录了需要处理的元素的队列，每一个元素都是一个需要抓取的URL列表，不过这一次我们用channel代替slice来做这个队列。每一个对crawl的调用都会在他们自己的goroutine中进行并且会把他们抓到的链接发送回worklist。

```go
func main() {
	worklist := make(chan []string)

	// Start with the command-line arguments.
	go func() { worklist <- os.Args[1:] }()

	// Crawl the web concurrently.
	seen := make(map[string]bool)
	for list := range worklist {
		for _, link := range list {
			if !seen[link] {
				seen[link] = true
				go func(link string) {
					worklist <- crawl(link)
				}(link)
			}
		}
	}
}
```

注意这里的crawl所在的goroutine会将link作为一个显式的参数传入，来避免“循环变量快照”的问题（在5.6.1中有讲解）。另外注意这里将命令行参数传入worklist也是在一个另外的goroutine中进行的，这是为了避免channel两端的main goroutine与crawler goroutine都尝试向对方发送内容，却没有一端接收内容时发生死锁。当然，这里我们也可以用buffered channel来解决问题，这里不再赘述。

现在爬虫可以高并发地运行起来，并且可以产生一大坨的URL了，不过还是会有俩问题。一个问题是在运行一段时间后可能会出现在log的错误信息里的：


```go
$ go build gopl.io/ch8/crawl1
$ ./crawl1 http://gopl.io/
http://gopl.io/
https://golang.org/help/
https://golang.org/doc/
https://golang.org/blog/
...
2015/07/15 18:22:12 Get ...: dial tcp: lookup blog.golang.org: no such host
2015/07/15 18:22:12 Get ...: dial tcp 23.21.222.120:443: socket: too many open files
...
```

最初的错误信息是一个让人莫名的DNS查找失败，即使这个域名是完全可靠的。而随后的错误信息揭示了原因：这个程序一次性创建了太多网络连接，超过了每一个进程的打开文件数限制，既而导致了在调用`net.Dial`像DNS查找失败这样的问题。

这个程序实在是太他妈并行了。无穷无尽地并行化并不是什么好事情，因为不管怎么说，你的系统总是会有一些个限制因素，比如CPU核心数会限制你的计算负载，比如你的硬盘转轴和磁头数限制了你的本地磁盘IO操作频率，比如你的网络带宽限制了你的下载速度上限，或者是你的一个web服务的服务容量上限等等。为了解决这个问题，我们可以限制并发程序所使用的资源来使之适应自己的运行环境。对于我们的例子来说，最简单的方法就是限制对`links.Extract`在同一时间最多不会有超过n次调用，这里的n一般小于文件描述符的上限值，比如20。这和一个夜店里限制客人数目是一个道理，只有当有客人离开时，才会允许新的客人进入店内。

我们可以用一个有容量限制的`buffered channel`来控制并发，这类似于操作系统里的计数信号量概念。从概念上讲，`channel`里的n个空槽代表n个可以处理内容的token（通行证），从channel里接收一个值会释放其中的一个token，并且生成一个新的空槽位。这样保证了在没有接收介入时最多有n个发送操作。（这里可能我们拿`channel`里填充的槽来做`token`更直观一些，不过还是这样吧。）由于channel里的元素类型并不重要，我们用一个零值的`struct{}`来作为其元素。

让我们重写crawl函数，将对`links.Extract`的调用操作用获取、释放token的操作包裹起来，来确保同一时间对其只有20个调用。信号量数量和其能操作的IO资源数量应保持接近。

<u><i>gopl.io/ch8/crawl2</i></u>
```go
// tokens is a counting semaphore used to
// enforce a limit of 20 concurrent requests.
var tokens = make(chan struct{}, 20)

func crawl(url string) []string {
	fmt.Println(url)
	tokens <- struct{}{} // acquire a token
	list, err := links.Extract(url)
	<-tokens // release the token
	if err != nil {
		log.Print(err)
	}
	return list
}
```

第二个问题是这个程序永远都不会终止，即使它已经爬到了所有初始链接衍生出的链接。（当然，除非你慎重地选择了合适的初始化URL或者已经实现了练习8.6中的深度限制，你应该还没有意识到这个问题。）为了使这个程序能够终止，我们需要在worklist为空或者没有crawl的goroutine在运行时退出主循环。

```go
func main() {
	worklist := make(chan []string)
	var n int // number of pending sends to worklist

	// Start with the command-line arguments.
	n++
	go func() { worklist <- os.Args[1:] }()

	// Crawl the web concurrently.
	seen := make(map[string]bool)

	for ; n > 0; n-- {
		list := <-worklist
		for _, link := range list {
			if !seen[link] {
				seen[link] = true
				n++
				go func(link string) {
					worklist <- crawl(link)
				}(link)
			}
		}
	}
}
```

这个版本中，计数器n对`worklist`的发送操作数量进行了限制。每一次我们发现有元素需要被发送到worklist时，我们都会对n进行`++`操作，在向worklist中发送初始的命令行参数之前，我们也进行过一次`++`操作。这里的操作`++`是在每启动一个`crawler`的`goroutine`之前。主循环会在`n`减为`0`时终止，这时候说明没活可干了。

现在这个并发爬虫会比5.6节中的深度优先搜索版快上20倍，而且不会出什么错，并且在其完成任务时也会正确地终止。

下面的程序是避免过度并发的另一种思路。这个版本使用了原来的crawl函数，但没有使用计数信号量，取而代之用了20个常驻的crawler goroutine，这样来保证最多20个HTTP请求在并发。

```go
func main() {
	worklist := make(chan []string)  // lists of URLs, may have duplicates
	unseenLinks := make(chan string) // de-duplicated URLs

	// Add command-line arguments to worklist.
	go func() { worklist <- os.Args[1:] }()

	// Create 20 crawler goroutines to fetch each unseen link.
	for i := 0; i < 20; i++ {
		go func() {
			for link := range unseenLinks {
				foundLinks := crawl(link)
				go func() { worklist <- foundLinks }()
			}
		}()
	}

	// The main goroutine de-duplicates worklist items
	// and sends the unseen ones to the crawlers.
	seen := make(map[string]bool)
	for list := range worklist {
		for _, link := range list {
			if !seen[link] {
				seen[link] = true
				unseenLinks <- link
			}
		}
	}
}
```

所有的爬虫goroutine现在都是被同一个channel - unseenLinks喂饱的了。主goroutine负责拆分它从worklist里拿到的元素，然后把没有抓过的经由unseenLinks channel发送给一个爬虫的goroutine。

seen这个map被限定在main goroutine中；也就是说这个map只能在main goroutine中进行访问。类似于其它的信息隐藏方式，这样的约束可以让我们从一定程度上保证程序的正确性。例如，内部变量不能够在函数外部被访问到；变量（§2.3.4）在没有发生变量逃逸（译注：局部变量被全局变量引用地址导致变量被分配在堆上）的情况下是无法在函数外部访问的；一个对象的封装字段无法被该对象的方法以外的方法访问到。在所有的情况下，信息隐藏都可以帮助我们约束我们的程序，使其不发生意料之外的情况。

crawl函数爬到的链接在一个专有的goroutine中被发送到worklist中来避免死锁。为了节省篇幅，这个例子的终止问题我们先不进行详细阐述了。

**练习 8.6：** 为并发爬虫增加深度限制。也就是说，如果用户设置了`depth=3`，那么只有从首页跳转三次以内能够跳到的页面才能被抓取到。

```go
// findlinks.go
package main

import (
	"flag"
	"fmt"
	"log"
	"sync"

	"gopl.io/ch5/links"
)

// tokens is a counting semaphore used to
// enforce a limit of 20 concurrent requests.
var tokens = make(chan struct{}, 20)
var maxDepth int
var seen = make(map[string]bool)
var seenLock = sync.Mutex{}

func crawl(url string, depth int, wg *sync.WaitGroup) {
	defer wg.Done()
	fmt.Println(depth, url)
	if depth >= maxDepth {
		return
	}
	tokens <- struct{}{} // acquire a token
	list, err := links.Extract(url)
	<-tokens // release the token
	if err != nil {
		log.Print(err)
	}
	for _, link := range list {
		seenLock.Lock()
		if seen[link] {
			seenLock.Unlock()
			continue
		}
		seen[link] = true
		seenLock.Unlock()
		wg.Add(1)
		go crawl(link, depth+1, wg)
	}
}

func main() {
	flag.IntVar(&maxDepth, "d", 3, "max crawl depth")
	flag.Parse()
	wg := &sync.WaitGroup{}
	for _, link := range flag.Args() {
		wg.Add(1)
		go crawl(link, 0, wg)
	}
	wg.Wait()
}
```

**练习 8.7：** 完成一个并发程序来创建一个线上网站的本地镜像，把该站点的所有可达的页面都抓取到本地硬盘。为了省事，我们这里可以只取出现在该域下的所有页面（比如`golang.org`开头，译注：外链的应该就不算了。）当然了，出现在页面里的链接你也需要进行一些处理，使其能够在你的镜像站点上进行跳转，而不是指向原始的链接。

```go
// mirror.go
package main

import (
	"bytes"
	"flag"
	"fmt"
	"io"
	"log"
	"net/http"
	"net/url"
	"os"
	"path/filepath"
	"strings"
	"sync"

	"golang.org/x/net/html"
)

// tokens is a counting semaphore used to
// enforce a limit of 20 concurrent requests.
var tokens = make(chan struct{}, 20)
var maxDepth int
var seen = make(map[string]bool)
var seenLock = sync.Mutex{}
var base *url.URL

func crawl(url string, depth int, wg *sync.WaitGroup) {
	defer wg.Done()

	tokens <- struct{}{} // acquire a token
	urls, err := visit(url)
	<-tokens //release token
	if err != nil {
		log.Printf("visit %s: %s", url, err)
	}

	if depth >= maxDepth {
		return
	}
	for _, link := range urls {
		seenLock.Lock()
		if seen[link] {
			seenLock.Unlock()
			continue
		}
		seen[link] = true
		seenLock.Unlock()
		wg.Add(1)
		go crawl(link, depth+1, wg)
	}
}

// Copied from gopl.io/ch5/outline2.
func forEachNode(n *html.Node, pre, post func(n *html.Node)) {
	if pre != nil {
		pre(n)
	}
	for c := n.FirstChild; c != nil; c = c.NextSibling {
		forEachNode(c, pre, post)
	}
	if post != nil {
		post(n)
	}
}

func linkNodes(n *html.Node) []*html.Node {
	var links []*html.Node
	visitNode := func(n *html.Node) {
		if n.Type == html.ElementNode && n.Data == "a" {
			links = append(links, n)
		}
	}
	forEachNode(n, visitNode, nil)
	return links
}

func linkURLs(linkNodes []*html.Node, base *url.URL) []string {
	var urls []string
	for _, n := range linkNodes {
		for _, a := range n.Attr {
			if a.Key != "href" {
				continue
			}
			link, err := base.Parse(a.Val)
			// ignore bad and non-local URLs
			if err != nil {
				log.Printf("skipping %q: %s", a.Val, err)
				continue
			}
			if link.Host != base.Host {
				//log.Printf("skipping %q: non-local host", a.Val)
				continue
			}
			urls = append(urls, link.String())
		}
	}
	return urls
}

// rewriteLocalLinks rewrites local links to be relative and links without
// extensions to point to index.html, eg /hi/there -> /hi/there/index.html.
func rewriteLocalLinks(linkNodes []*html.Node, base *url.URL) {
	for _, n := range linkNodes {
		for i, a := range n.Attr {
			if a.Key != "href" {
				continue
			}
			link, err := base.Parse(a.Val)
			if err != nil || link.Host != base.Host {
				continue // ignore bad and non-local URLs
			}
			// Clear fields so the url is formatted as /PATH?QUERY#FRAGMENT
			link.Scheme = ""
			link.Host = ""
			link.User = nil
			a.Val = link.String()
			n.Attr[i] = a
		}
	}
}

func visit(rawurl string) (urls []string, err error) {
	fmt.Println(rawurl)
	resp, err := http.Get(rawurl)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()
	if resp.StatusCode != http.StatusOK {
		resp.Body.Close()
		return nil, fmt.Errorf("GET %s: %s", rawurl, resp.Status)
	}

	u, err := base.Parse(rawurl)
	if err != nil {
		return nil, err
	}
	if base.Host != u.Host {
		log.Printf("not saving %s: non-local", rawurl)
		return nil, nil
	}

	var body io.Reader
	contentType := resp.Header["Content-Type"]
	if strings.Contains(strings.Join(contentType, ","), "text/html") {
		doc, err := html.Parse(resp.Body)
		resp.Body.Close()
		if err != nil {
			return nil, fmt.Errorf("parsing %s as HTML: %v", u, err)
		}
		nodes := linkNodes(doc)
		urls = linkURLs(nodes, u) // Extract links before they're rewritten.
		rewriteLocalLinks(nodes, u)
		b := &bytes.Buffer{}
		err = html.Render(b, doc)
		if err != nil {
			log.Printf("render %s: %s", u, err)
		}
		body = b
	}
	err = save(resp, body)
	return urls, err
}

// If resp.Body has already been consumed, `body` can be passed and will be
// read instead.
func save(resp *http.Response, body io.Reader) error {
	u := resp.Request.URL
	filename := filepath.Join(u.Host, u.Path)
	if filepath.Ext(u.Path) == "" {
		filename = filepath.Join(u.Host, u.Path, "index.html")
	}
	err := os.MkdirAll(filepath.Dir(filename), 0777)
	if err != nil {
		return err
	}
	fmt.Println("filename:", filename)
	file, err := os.Create(filename)
	if err != nil {
		return err
	}
	if body != nil {
		_, err = io.Copy(file, body)
	} else {
		_, err = io.Copy(file, resp.Body)
	}
	if err != nil {
		log.Print("save: ", err)
	}
	// Check for delayed write errors, as mentioned at the end of section 5.8.
	err = file.Close()
	if err != nil {
		log.Print("save: ", err)
	}
	return nil
}

func main() {
	flag.IntVar(&maxDepth, "d", 3, "max crawl depth")
	flag.Parse()
	wg := &sync.WaitGroup{}
	if len(flag.Args()) == 0 {
		fmt.Fprintln(os.Stderr, "usage: mirror URL ...")
		os.Exit(1)
	}
	u, err := url.Parse(flag.Arg(0))
	if err != nil {
		fmt.Fprintf(os.Stderr, "invalid url: %s\n", err)
	}
	base = u
	for _, link := range flag.Args() {
		wg.Add(1)
		go crawl(link, 1, wg)
	}
	wg.Wait()
}
```

```go
// mirror_test.go
package main

import (
	"golang.org/x/net/html"
	"net/url"
	"strings"
	"testing"
)

func TestRewriteLocalLinks(t *testing.T) {
	url_, err := url.Parse("http://www.rah.com/a/b/c")
	if err != nil {
		t.Fatal(err)
	}
	base = url_
	input := `<html><body><a href="http://www.rah.com/d/e">text</a></body></html>`
	doc, err := html.Parse(strings.NewReader(input))
	if err != nil {
		t.Fatal(err)
	}
	nodes := linkNodes(doc)
	rewriteLocalLinks(nodes, url_)
	want := "/d/e"
	got := nodes[0].Attr[0].Val
	if got != want {
		t.Fatalf("got %q, want %q", got, want)
	}
}
```

> **译注：**
> 拓展阅读 [Handling 1 Million Requests per Minute with Go](http://marcio.io/2015/07/handling-1-million-requests-per-minute-with-golang/)。

## 8.7. 基于select的多路复用

下面的程序会进行火箭发射的倒计时。`time.Tick`函数返回一个`channel`，程序会周期性地像一个节拍器一样向这个channel发送事件。每一个事件的值是一个时间戳，不过更有意思的是其传送方式。

<u><i>gopl.io/ch8/countdown1</i></u>

```go
func main() {
	fmt.Println("Commencing countdown.")
	tick := time.Tick(1 * time.Second)
	for countdown := 10; countdown > 0; countdown-- {
		fmt.Println(countdown)
		<-tick
	}
	launch()
}
```

现在我们让这个程序支持在倒计时中，用户按下return键时直接中断发射流程。首先，我们启动一个goroutine，这个goroutine会尝试从标准输入中读入一个单独的byte并且，如果成功了，会向名为abort的channel发送一个值。

<u><i>gopl.io/ch8/countdown2</i></u>
```go
abort := make(chan struct{})
go func() {
	os.Stdin.Read(make([]byte, 1)) // read a single byte
	abort <- struct{}{}
}()
```

现在每一次计数循环的迭代都需要等待两个channel中的其中一个返回事件了：当一切正常时的ticker channel（就像NASA jorgon的"nominal"，译注：这梗估计我们是不懂了）或者异常时返回的abort事件。我们无法做到从每一个channel中接收信息，如果我们这么做的话，如果第一个channel中没有事件发过来那么程序就会立刻被阻塞，这样我们就无法收到第二个channel中发过来的事件。这时候我们需要多路复用（multiplex）这些操作了，为了能够多路复用，我们使用了select语句。

```go
select {
case <-ch1:
	// ...
case x := <-ch2:
	// ...use x...
case ch3 <- y:
	// ...
default:
	// ...
}
```

上面是`select`语句的一般形式。和`switch`语句稍微有点相似，也会有几个`case`和最后的`default`选择分支。每一个`case`代表一个通信操作（在某个`channel`上进行发送或者接收），并且会包含一些语句组成的一个语句块。一个接收表达式可能只包含接收表达式自身（译注：不把接收到的值赋值给变量什么的），就像上面的第一个`case`，或者包含在一个简短的变量声明中，像第二个`case`里一样；第二种形式让你能够引用接收到的值。

`select`会等待`case`中有能够执行的`case`时去执行。当条件满足时，`select`才会去通信并执行`case`之后的语句；这时候其它通信是不会执行的。一个没有任何`case`的`select`语句写作`select{}`，会永远地等待下去。

让我们回到我们的火箭发射程序。`time.After`函数会立即返回一个`channel`，并起一个新的`goroutine`在经过特定的时间后向该`channel`发送一个独立的值。下面的`select`语句会一直等待直到两个事件中的一个到达，无论是`abort`事件或者一个10秒经过的事件。如果10秒经过了还没有`abort`事件进入，那么火箭就会发射。

```go
func main() {
	// ...create abort channel...

	fmt.Println("Commencing countdown.  Press return to abort.")
	select {
	case <-time.After(10 * time.Second):
		// Do nothing.
	case <-abort:
		fmt.Println("Launch aborted!")
		return
	}
	launch()
}
```


下面这个例子更微妙。`ch`这个`channel`的`buffer`大小是1，所以会交替的为空或为满，所以只有一个case可以进行下去，无论`i`是奇数或者偶数，它都会打印0 2 4 6 8。

```go
ch := make(chan int, 1)
for i := 0; i < 10; i++ {
	select {
	case x := <-ch:
		fmt.Println(x) // "0" "2" "4" "6" "8"
	case ch <- i:
	}
}
```

如果多个case同时就绪时，select会随机地选择一个执行，这样来保证每一个channel都有平等的被select的机会。增加前一个例子的buffer大小会使其输出变得不确定，因为当buffer既不为满也不为空时，select语句的执行情况就像是抛硬币的行为一样是随机的。

下面让我们的发射程序打印倒计时。这里的select语句会使每次循环迭代等待一秒来执行退出操作。

<u><i>gopl.io/ch8/countdown3</i></u>
```go
func main() {
	// ...create abort channel...

	fmt.Println("Commencing countdown.  Press return to abort.")
	tick := time.Tick(1 * time.Second)
	for countdown := 10; countdown > 0; countdown-- {
		fmt.Println(countdown)
		select {
		case <-tick:
			// Do nothing.
		case <-abort:
			fmt.Println("Launch aborted!")
			return
		}
	}
	launch()
}
```

`time.Tick`函数表现得好像它创建了一个在循环中调用`time.Sleep`的`goroutine`，每次被唤醒时发送一个事件。当countdown函数返回时，它会停止从tick中接收事件，但是ticker这个goroutine还依然存活，继续徒劳地尝试向channel中发送值，然而这时候已经没有其它的goroutine会从该channel中接收值了——这被称为goroutine泄露（§8.4.4）。

Tick函数挺方便，但是只有当程序整个生命周期都需要这个时间时我们使用它才比较合适。否则的话，我们应该使用下面的这种模式：

```go
ticker := time.NewTicker(1 * time.Second)
<-ticker.C    // receive from the ticker's channel
ticker.Stop() // cause the ticker's goroutine to terminate
```

有时候我们希望能够从channel中发送或者接收值，并避免因为发送或者接收导致的阻塞，尤其是当channel没有准备好写或者读时。select语句就可以实现这样的功能。select会有一个default来设置当其它的操作都不能够马上被处理时程序需要执行哪些逻辑。

下面的select语句会在abort channel中有值时，从其中接收值；无值时什么都不做。这是一个非阻塞的接收操作；反复地做这样的操作叫做“轮询channel”。

```go
select {
case <-abort:
	fmt.Printf("Launch aborted!\n")
	return
default:
	// do nothing
}
```

channel的零值是nil。也许会让你觉得比较奇怪，nil的channel有时候也是有一些用处的。因为对一个nil的channel发送和接收操作会永远阻塞，在select语句中操作nil的channel永远都不会被select到。

这使得我们可以用nil来激活或者禁用case，来达成处理其它输入或输出事件时超时和取消的逻辑。我们会在下一节中看到一个例子。

**练习 8.8：** 使用`select`来改造8.3节中的`echo`服务器，为其增加超时，这样服务器可以在客户端10秒中没有任何喊话时自动断开连接。

```go
package main

import (
	"bufio"
	"fmt"
	"io"
	"log"
	"net"
	"strings"
	"sync"
	"time"
)

func echo(c net.Conn, shout string, delay time.Duration, wg *sync.WaitGroup) {
	defer wg.Done()
	fmt.Fprintln(c, "\t", strings.ToUpper(shout))
	time.Sleep(delay)
	fmt.Fprintln(c, "\t", shout)
	time.Sleep(delay)
	fmt.Fprintln(c, "\t", strings.ToLower(shout))
}

func scan(r io.Reader, lines chan<- string) {
	s := bufio.NewScanner(r)
	for s.Scan() {
		lines <- s.Text()
	}
	// scan will most likely try to read from the connection after it's closed
	// by handleConn. I don't know how to avoid this. Go seems to shun async io
	// in favour of goroutines, so it probably isn't worth avoiding.
	if s.Err() != nil {
		log.Print("scan: ", s.Err())
	}
}

func handleConn(c net.Conn) {
	wg := &sync.WaitGroup{}
	defer func() {
		wg.Wait()
		c.Close()
	}()
	lines := make(chan string)
	go scan(c, lines)
	timeout := 2 * time.Second
	timer := time.NewTimer(2 * time.Second)
	for {
		select {
		case line := <-lines:
			timer.Reset(timeout)
			wg.Add(1)
			go echo(c, line, 1*time.Second, wg)
		case <-timer.C:
			return
		}
	}
}

func main() {
	l, err := net.Listen("tcp", "localhost:8000")
	if err != nil {
		log.Fatal(err)
	}
	for {
		conn, err := l.Accept()
		if err != nil {
			log.Print(err) // e.g., connection aborted
			continue
		}
		go handleConn(conn)
	}
}
```

## 8.8. 示例: 并发的目录遍历

在本小节中，我们会创建一个程序来生成指定目录的硬盘使用情况报告，这个程序和Unix里的du工具比较相似。大多数工作用下面这个walkDir函数来完成，这个函数使用dirents函数来枚举一个目录下的所有入口。

<u><i>gopl.io/ch8/du1</i></u>
```go
// walkDir recursively walks the file tree rooted at dir
// and sends the size of each found file on fileSizes.
func walkDir(dir string, fileSizes chan<- int64) {
	for _, entry := range dirents(dir) {
		if entry.IsDir() {
			subdir := filepath.Join(dir, entry.Name())
			walkDir(subdir, fileSizes)
		} else {
			fileSizes <- entry.Size()
		}
	}
}

// dirents returns the entries of directory dir.
func dirents(dir string) []os.FileInfo {
	entries, err := ioutil.ReadDir(dir)
	if err != nil {
		fmt.Fprintf(os.Stderr, "du1: %v\n", err)
		return nil
	}
	return entries
}
```

ioutil.ReadDir函数会返回一个os.FileInfo类型的slice，os.FileInfo类型也是os.Stat这个函数的返回值。对每一个子目录而言，walkDir会递归地调用其自身，同时也在递归里获取每一个文件的信息。walkDir函数会向fileSizes这个channel发送一条消息。这条消息包含了文件的字节大小。

下面的主函数，用了两个goroutine。后台的goroutine调用walkDir来遍历命令行给出的每一个路径并最终关闭fileSizes这个channel。主goroutine会对其从channel中接收到的文件大小进行累加，并输出其和。

```go
package main

import (
	"flag"
	"fmt"
	"io/ioutil"
	"os"
	"path/filepath"
)

func main() {
	// Determine the initial directories.
	flag.Parse()
	roots := flag.Args()
	if len(roots) == 0 {
		roots = []string{"."}
	}

	// Traverse the file tree.
	fileSizes := make(chan int64)
	go func() {
		for _, root := range roots {
			walkDir(root, fileSizes)
		}
		close(fileSizes)
	}()

	// Print the results.
	var nfiles, nbytes int64
	for size := range fileSizes {
		nfiles++
		nbytes += size
	}
	printDiskUsage(nfiles, nbytes)
}

func printDiskUsage(nfiles, nbytes int64) {
	fmt.Printf("%d files  %.1f GB\n", nfiles, float64(nbytes)/1e9)
}
```

这个程序会在打印其结果之前卡住很长时间。

```
$ go build gopl.io/ch8/du1
$ ./du1 $HOME /usr /bin /etc
213201 files  62.7 GB
```

如果在运行的时候能够让我们知道处理进度的话想必更好。但是，如果简单地把printDiskUsage函数调用移动到循环里会导致其打印出成百上千的输出。

下面这个du的变种会间歇打印内容，不过只有在调用时提供了-v的flag才会显示程序进度信息。在roots目录上循环的后台goroutine在这里保持不变。主goroutine现在使用了计时器来每500ms生成事件，然后用select语句来等待文件大小的消息来更新总大小数据，或者一个计时器的事件来打印当前的总大小数据。如果-v的flag在运行时没有传入的话，tick这个channel会保持为nil，这样在select里的case也就相当于被禁用了。

<u><i>gopl.io/ch8/du2</i></u>
```go
var verbose = flag.Bool("v", false, "show verbose progress messages")

func main() {
	// ...start background goroutine...

	// Print the results periodically.
	var tick <-chan time.Time
	if *verbose {
		tick = time.Tick(500 * time.Millisecond)
	}
	var nfiles, nbytes int64
loop:
	for {
		select {
		case size, ok := <-fileSizes:
			if !ok {
				break loop // fileSizes was closed
			}
			nfiles++
			nbytes += size
		case <-tick:
			printDiskUsage(nfiles, nbytes)
		}
	}
	printDiskUsage(nfiles, nbytes) // final totals
}
```

由于我们的程序不再使用range循环，第一个select的case必须显式地判断fileSizes的channel是不是已经被关闭了，这里可以用到channel接收的二值形式。如果channel已经被关闭了的话，程序会直接退出循环。这里的break语句用到了标签break，这样可以同时终结select和for两个循环；如果没有用标签就break的话只会退出内层的select循环，而外层的for循环会使之进入下一轮select循环。

现在程序会悠闲地为我们打印更新流：

```
$ go build gopl.io/ch8/du2
$ ./du2 -v $HOME /usr /bin /etc
28608 files  8.3 GB
54147 files  10.3 GB
93591 files  15.1 GB
127169 files  52.9 GB
175931 files  62.2 GB
213201 files  62.7 GB
```

然而这个程序还是会花上很长时间才会结束。完全可以并发调用walkDir，从而发挥磁盘系统的并行性能。下面这个第三个版本的du，会对每一个walkDir的调用创建一个新的goroutine。它使用sync.WaitGroup（§8.5）来对仍旧活跃的walkDir调用进行计数，另一个goroutine会在计数器减为零的时候将fileSizes这个channel关闭。

<u><i>gopl.io/ch8/du3</i></u>
```go
func main() {
	// ...determine roots...
	// Traverse each root of the file tree in parallel.
	fileSizes := make(chan int64)
	var n sync.WaitGroup
	for _, root := range roots {
		n.Add(1)
		go walkDir(root, &n, fileSizes)
	}
	go func() {
		n.Wait()
		close(fileSizes)
	}()
	// ...select loop...
}

func walkDir(dir string, n *sync.WaitGroup, fileSizes chan<- int64) {
	defer n.Done()
	for _, entry := range dirents(dir) {
		if entry.IsDir() {
			n.Add(1)
			subdir := filepath.Join(dir, entry.Name())
			go walkDir(subdir, n, fileSizes)
		} else {
			fileSizes <- entry.Size()
		}
	}
}
```

由于这个程序在高峰期会创建成百上千的goroutine，我们需要修改dirents函数，用计数信号量来阻止他同时打开太多的文件，就像我们在8.7节中的并发爬虫一样：

```go
// sema is a counting semaphore for limiting concurrency in dirents.
var sema = make(chan struct{}, 20)

// dirents returns the entries of directory dir.
func dirents(dir string) []os.FileInfo {
	sema <- struct{}{}        // acquire token
	defer func() { <-sema }() // release token
	// ...
```

这个版本比之前那个快了好几倍，尽管其具体效率还是和你的运行环境，机器配置相关。

**练习 8.9：** 编写一个du工具，每隔一段时间将root目录下的目录大小计算并显示出来。

```go
package main

import (
	"flag"
	"fmt"
	"io/ioutil"
	"os"
	"path/filepath"
	"sync"
	"time"
)

var vFlag = flag.Bool("v", false, "show verbose progress messages")

type SizeResponse struct {
	root int
	size int64
}

func main() {
	// ...determine roots...

	flag.Parse()

	// Determine the initial directories.
	roots := flag.Args()
	if len(roots) == 0 {
		roots = []string{"."}
	}

	// Traverse each root of the file tree in parallel.
	sizeResponses := make(chan SizeResponse)
	var n sync.WaitGroup
	for i, root := range roots {
		n.Add(1)
		go walkDir(root, &n, i, sizeResponses)
	}
	go func() {
		n.Wait()
		close(sizeResponses)
	}()

	// Print the results periodically.
	var tick <-chan time.Time
	if *vFlag {
		tick = time.Tick(500 * time.Millisecond)
	}
	nfiles := make([]int64, len(roots))
	nbytes := make([]int64, len(roots))
loop:
	for {
		select {
		case sr, ok := <-sizeResponses:
			if !ok {
				break loop // sizeResponses was closed
			}
			nfiles[sr.root]++
			nbytes[sr.root] += sr.size
		case <-tick:
			printDiskUsage(roots, nfiles, nbytes)
		}
	}

	printDiskUsage(roots, nfiles, nbytes) // final totals
	// ...select loop...
}

func printDiskUsage(roots []string, nfiles, nbytes []int64) {
	for i, r := range roots {
		fmt.Printf("%10d files  %.3f GB under %s\n", nfiles[i], float64(nbytes[i])/1e9, r)
	}
}

// walkDir recursively walks the file tree rooted at dir
// and sends the size of each found file on sizeResponses.
func walkDir(dir string, n *sync.WaitGroup, root int, sizeResponses chan<- SizeResponse) {
	defer n.Done()
	for _, entry := range dirents(dir) {
		if entry.IsDir() {
			n.Add(1)
			subdir := filepath.Join(dir, entry.Name())
			go walkDir(subdir, n, root, sizeResponses)
		} else {
			sizeResponses <- SizeResponse{root, entry.Size()}
		}
	}
}

// sema is a counting semaphore for limiting concurrency in dirents.
var sema = make(chan struct{}, 20)

// dirents returns the entries of directory dir.
func dirents(dir string) []os.FileInfo {
	sema <- struct{}{}        // acquire token
	defer func() { <-sema }() // release token
	// ...

	entries, err := ioutil.ReadDir(dir)
	if err != nil {
		fmt.Fprintf(os.Stderr, "du: %v\n", err)
		return nil
	}
	return entries
}
```

## 8.9. 并发的退出

有时候我们需要通知goroutine停止它正在干的事情，比如一个正在执行计算的web服务，然而它的客户端已经断开了和服务端的连接。

Go语言并没有提供在一个goroutine中终止另一个goroutine的方法，由于这样会导致goroutine之间的共享变量落在未定义的状态上。在8.7节中的rocket launch程序中，我们往名字叫abort的channel里发送了一个简单的值，在countdown的goroutine中会把这个值理解为自己的退出信号。但是如果我们想要退出两个或者任意多个goroutine怎么办呢？

一种可能的手段是向abort的channel里发送和goroutine数目一样多的事件来退出它们。如果这些goroutine中已经有一些自己退出了，那么会导致我们的channel里的事件数比goroutine还多，这样导致我们的发送直接被阻塞。另一方面，如果这些goroutine又生成了其它的goroutine，我们的channel里的数目又太少了，所以有些goroutine可能会无法接收到退出消息。一般情况下我们是很难知道在某一个时刻具体有多少个goroutine在运行着的。另外，当一个goroutine从abort channel中接收到一个值的时候，他会消费掉这个值，这样其它的goroutine就没法看到这条信息。为了能够达到我们退出goroutine的目的，我们需要更靠谱的策略，来通过一个channel把消息广播出去，这样goroutine们能够看到这条事件消息，并且在事件完成之后，可以知道这件事已经发生过了。

回忆一下我们关闭了一个channel并且被消费掉了所有已发送的值，操作channel之后的代码可以立即被执行，并且会产生零值。我们可以将这个机制扩展一下，来作为我们的广播机制：不要向channel发送值，而是用关闭一个channel来进行广播。

只要一些小修改，我们就可以把退出逻辑加入到前一节的du程序。首先，我们创建一个退出的channel，不需要向这个channel发送任何值，但其所在的闭包内要写明程序需要退出。我们同时还定义了一个工具函数，cancelled，这个函数在被调用的时候会轮询退出状态。

`gopl.io/ch8/du4`

```go
var done = make(chan struct{})

func cancelled() bool {
	select {
	case <-done:
		return true
	default:
		return false
	}
}
```

下面我们创建一个从标准输入流中读取内容的`goroutine`，这是一个比较典型的连接到终端的程序。每当有输入被读到（比如用户按了回车键），这个`goroutine`就会把取消消息通过关闭`done`的`channel`广播出去。

```go
// Cancel traversal when input is detected.
go func() {
	os.Stdin.Read(make([]byte, 1)) // read a single byte
	close(done)
}()
```

现在我们需要使我们的`goroutine`来对取消进行响应。在`main goroutine`中，我们添加了`select`的第三个`case`语句，尝试从`done channel`中接收内容。如果这个`case`被满足的话，在`select`到的时候即会返回，但在结束之前我们需要把`fileSizes channel`中的内容“排”空，在`channel`被关闭之前，舍弃掉所有值。这样可以保证对`walkDir`的调用不要被向`fileSizes`发送信息阻塞住，可以正确地完成。

```go
for {
	select {
	case <-done:
		// Drain fileSizes to allow existing goroutines to finish.
		for range fileSizes {
			// Do nothing.
		}
		return
	case size, ok := <-fileSizes:
		// ...
	}
}
```

walkDir这个goroutine一启动就会轮询取消状态，如果取消状态被设置的话会直接返回，并且不做额外的事情。这样我们将所有在取消事件之后创建的goroutine改变为无操作。

```go
func walkDir(dir string, n *sync.WaitGroup, fileSizes chan<- int64) {
	defer n.Done()
	if cancelled() {
		return
	}
	for _, entry := range dirents(dir) {
		// ...
	}
}
```

在`walkDir`函数的循环中我们对取消状态进行轮询可以带来明显的益处，可以避免在取消事件发生时还去创建goroutine。取消本身是有一些代价的；想要快速的响应需要对程序逻辑进行侵入式的修改。确保在取消发生之后不要有代价太大的操作可能会需要修改你代码里的很多地方，但是在一些重要的地方去检查取消事件也确实能带来很大的好处。

对这个程序的一个简单的性能分析可以揭示瓶颈在`dirents`函数中获取一个信号量。下面的select可以让这种操作可以被取消，并且可以将取消时的延迟从几百毫秒降低到几十毫秒。

```go
func dirents(dir string) []os.FileInfo {
	select {
	case sema <- struct{}{}: // acquire token
	case <-done:
		return nil // cancelled
	}
	defer func() { <-sema }() // release token
	// ...read directory...
}
```

现在当取消发生时，所有后台的goroutine都会迅速停止并且主函数会返回。当然，当主函数返回时，一个程序会退出，而我们又无法在主函数退出的时候确认其已经释放了所有的资源（译注：因为程序都退出了，你的代码都没法执行了）。这里有一个方便的窍门我们可以一用：取代掉直接从主函数返回，我们调用一个panic，然后runtime会把每一个goroutine的栈dump下来。如果main goroutine是唯一一个剩下的goroutine的话，他会清理掉自己的一切资源。但是如果还有其它的goroutine没有退出，他们可能没办法被正确地取消掉，也有可能被取消但是取消操作会很花时间；所以这里的一个调研还是很有必要的。我们用panic来获取到足够的信息来验证我们上面的判断，看看最终到底是什么样的情况。

**练习 8.10：** HTTP请求可能会因`http.Request`结构体中`Cancel channel`的关闭而取消。修改8.6节中的web `crawler`来支持取消`http`请求。（提示：`http.Get`并没有提供方便地定制一个请求的方法。你可以用`http.NewRequest`来取而代之，设置它的Cancel字段，然后用`http.DefaultClient.Do(req)`来进行这个http请求。）

```go
// mirror.go
package main

import (
	"bytes"
	"flag"
	"fmt"
	"io"
	"log"
	"net/http"
	"net/url"
	"os"
	"os/signal"
	"path/filepath"
	"strings"
	"sync"

	"golang.org/x/net/html"
)

// tokens is a counting semaphore used to
// enforce a limit of 20 concurrent requests.
var tokens = make(chan struct{}, 20)
var maxDepth int
var seen = make(map[string]bool)
var seenLock = sync.Mutex{}
var base *url.URL
var cancel = make(chan struct{})

func crawl(url string, depth int, wg *sync.WaitGroup) {
	defer wg.Done()

	tokens <- struct{}{} // acquire a token
	urls, err := visit(url)
	<-tokens //release token
	if err != nil {
		log.Printf("visit %s: %s", url, err)
	}

	if depth >= maxDepth {
		return
	}
	for _, link := range urls {
		seenLock.Lock()
		if seen[link] {
			seenLock.Unlock()
			continue
		}
		seen[link] = true
		seenLock.Unlock()
		wg.Add(1)
		go crawl(link, depth+1, wg)
	}
}

// Copied from gopl.io/ch5/outline2.
func forEachNode(n *html.Node, pre, post func(n *html.Node)) {
	if pre != nil {
		pre(n)
	}
	for c := n.FirstChild; c != nil; c = c.NextSibling {
		forEachNode(c, pre, post)
	}
	if post != nil {
		post(n)
	}
}

func linkNodes(n *html.Node) []*html.Node {
	var links []*html.Node
	visitNode := func(n *html.Node) {
		if n.Type == html.ElementNode && n.Data == "a" {
			links = append(links, n)
		}
	}
	forEachNode(n, visitNode, nil)
	return links
}

func linkURLs(linkNodes []*html.Node, base *url.URL) []string {
	var urls []string
	for _, n := range linkNodes {
		for _, a := range n.Attr {
			if a.Key != "href" {
				continue
			}
			link, err := base.Parse(a.Val)
			// ignore bad and non-local URLs
			if err != nil {
				log.Printf("skipping %q: %s", a.Val, err)
				continue
			}
			if link.Host != base.Host {
				//log.Printf("skipping %q: non-local host", a.Val)
				continue
			}
			urls = append(urls, link.String())
		}
	}
	return urls
}

// rewriteLocalLinks rewrites local links to be relative and links without
// extensions to point to index.html, eg /hi/there -> /hi/there/index.html.
func rewriteLocalLinks(linkNodes []*html.Node, base *url.URL) {
	for _, n := range linkNodes {
		for i, a := range n.Attr {
			if a.Key != "href" {
				continue
			}
			link, err := base.Parse(a.Val)
			if err != nil || link.Host != base.Host {
				continue // ignore bad and non-local URLs
			}
			// Clear fields so the url is formatted as /PATH?QUERY#FRAGMENT
			link.Scheme = ""
			link.Host = ""
			link.User = nil
			a.Val = link.String()
			n.Attr[i] = a
		}
	}
}

func visit(rawurl string) (urls []string, err error) {
	fmt.Println(rawurl)
	req, err := http.NewRequest("GET", rawurl, nil)
	req.Cancel = cancel
	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()
	if resp.StatusCode != http.StatusOK {
		resp.Body.Close()
		return nil, fmt.Errorf("GET %s: %s", rawurl, resp.Status)
	}

	u, err := base.Parse(rawurl)
	if err != nil {
		return nil, err
	}
	if base.Host != u.Host {
		log.Printf("not saving %s: non-local", rawurl)
		return nil, nil
	}

	var body io.Reader
	contentType := resp.Header["Content-Type"]
	if strings.Contains(strings.Join(contentType, ","), "text/html") {
		doc, err := html.Parse(resp.Body)
		resp.Body.Close()
		if err != nil {
			return nil, fmt.Errorf("parsing %s as HTML: %v", u, err)
		}
		nodes := linkNodes(doc)
		urls = linkURLs(nodes, u) // Extract links before they're rewritten.
		rewriteLocalLinks(nodes, u)
		b := &bytes.Buffer{}
		err = html.Render(b, doc)
		if err != nil {
			log.Printf("render %s: %s", u, err)
		}
		body = b
	}
	err = save(resp, body)
	return urls, err
}

// If resp.Body has already been consumed, `body` can be passed and will be
// read instead.
func save(resp *http.Response, body io.Reader) error {
	u := resp.Request.URL
	filename := filepath.Join(u.Host, u.Path)
	if filepath.Ext(u.Path) == "" {
		filename = filepath.Join(u.Host, u.Path, "index.html")
	}
	err := os.MkdirAll(filepath.Dir(filename), 0777)
	if err != nil {
		return err
	}
	fmt.Println("filename:", filename)
	file, err := os.Create(filename)
	if err != nil {
		return err
	}
	if body != nil {
		_, err = io.Copy(file, body)
	} else {
		_, err = io.Copy(file, resp.Body)
	}
	if err != nil {
		log.Print("save: ", err)
	}
	// Check for delayed write errors, as mentioned at the end of section 5.8.
	err = file.Close()
	if err != nil {
		log.Print("save: ", err)
	}
	return nil
}

func main() {
	flag.IntVar(&maxDepth, "d", 3, "max crawl depth")
	flag.Parse()
	wg := &sync.WaitGroup{}
	if len(flag.Args()) == 0 {
		fmt.Fprintln(os.Stderr, "usage: mirror URL ...")
		os.Exit(1)
	}
	u, err := url.Parse(flag.Arg(0))
	if err != nil {
		fmt.Fprintf(os.Stderr, "invalid url: %s\n", err)
	}
	base = u
	for _, link := range flag.Args() {
		wg.Add(1)
		go crawl(link, 1, wg)
	}

	done := make(chan struct{})
	go func() {
		wg.Wait()
		done <- struct{}{}
	}()
	interrupt := make(chan os.Signal, 1)
	signal.Notify(interrupt, os.Interrupt)
	select {
	case <-done:
		return
	case <-interrupt:
		close(cancel)
	}
}
```

**练习 8.11：** 紧接着8.4.4中的`mirroredQuery`流程，实现一个并发请求`url`的`fetch`的变种。当第一个请求返回时，直接取消其它的请求。

```go
// first.go
package main

import (
	"flag"
	"fmt"
	"log"
	"net/http"
	"strings"
	"sync"
)

func main() {
	flag.Parse()
	cancel := make(chan struct{})
	responses := make(chan *http.Response)
	wg := &sync.WaitGroup{}
	for _, url := range flag.Args() {
		wg.Add(1)
		go func(url string) {
			defer wg.Done()
			req, err := http.NewRequest("HEAD", url, nil)
			if err != nil {
				log.Printf("HEAD %s: %s", url, err)
				return
			}
			req.Cancel = cancel
			resp, err := http.DefaultClient.Do(req)
			if err != nil {
				log.Printf("HEAD %s: %s", url, err)
				return
			}
			responses <- resp
		}(url)
	}
	resp := <-responses
	defer resp.Body.Close()
	close(cancel) // Cancel incomplete requests.
	fmt.Println(resp.Request.URL)
	for name, vals := range resp.Header {
		fmt.Printf("%s: %s\n", name, strings.Join(vals, ","))
	}
	wg.Wait()
}
```

## 8.10. 示例: 聊天服务

我们用一个聊天服务器来终结本章节的内容，这个程序可以让一些用户通过服务器向其它所有用户广播文本消息。这个程序中有四种`goroutine`。`main`和`broadcaster`各自是一个`goroutine`实例，每一个客户端的连接都会有一个`handleConn`和`clientWriter`的`goroutine`。`broadcaster`是`select`用法的不错的样例，因为它需要处理三种不同类型的消息。

下面演示的main goroutine的工作，是`listen`和`accept`(译注：网络编程里的概念)从客户端过来的连接。对每一个连接，程序都会建立一个新的`handleConn`的`goroutine`，就像我们在本章开头的并发的`echo`服务器里所做的那样。

*gopl.io/ch8/chat*

```go
func main() {
	listener, err := net.Listen("tcp", "localhost:8000")
	if err != nil {
		log.Fatal(err)
	}
	go broadcaster()
	for {
		conn, err := listener.Accept()
		if err != nil {
			log.Print(err)
			continue
		}
		go handleConn(conn)
	}
}
```

然后是`broadcaster`的`goroutine`。他的内部变量`clients`会记录当前建立连接的客户端集合。其记录的内容是每一个客户端的消息发出`channel`的“资格”信息。

```go
type client chan<- string // an outgoing message channel

var (
	entering = make(chan client)
	leaving  = make(chan client)
	messages = make(chan string) // all incoming client messages
)

func broadcaster() {
	clients := make(map[client]bool) // all connected clients
	for {
		select {
		case msg := <-messages:
			// Broadcast incoming message to all
			// clients' outgoing message channels.
			for cli := range clients {
				cli <- msg
			}
		case cli := <-entering:
			clients[cli] = true

		case cli := <-leaving:
			delete(clients, cli)
			close(cli)
		}
	}
}
```

`broadcaster`监听来自全局的`entering`和`leaving`的`channel`来获知客户端的到来和离开事件。当其接收到其中的一个事件时，会更新`clients`集合，当该事件是离开行为时，它会关闭客户端的消息发送`channel`。`broadcaster`也会监听全局的消息`channel`，所有的客户端都会向这个`channel`中发送消息。当`broadcaster`接收到什么消息时，就会将其广播至所有连接到服务端的客户端。

现在让我们看看每一个客户端的`goroutine`。`handleConn`函数会为它的客户端创建一个消息发送`channel`并通过entering channel来通知客户端的到来。然后它会读取客户端发来的每一行文本，并通过全局的消息`channel`来将这些文本发送出去，并为每条消息带上发送者的前缀来标明消息身份。当客户端发送完毕后，`handleConn`会通过leaving这个`channel`来通知客户端的离开并关闭连接。

```go
func handleConn(conn net.Conn) {
	ch := make(chan string) // outgoing client messages
	go clientWriter(conn, ch)

	who := conn.RemoteAddr().String()
	ch <- "You are " + who
	messages <- who + " has arrived"
	entering <- ch

	input := bufio.NewScanner(conn)
	for input.Scan() {
		messages <- who + ": " + input.Text()
	}
	// NOTE: ignoring potential errors from input.Err()

	leaving <- ch
	messages <- who + " has left"
	conn.Close()
}

func clientWriter(conn net.Conn, ch <-chan string) {
	for msg := range ch {
		fmt.Fprintln(conn, msg) // NOTE: ignoring network errors
	}
}
```

另外，`handleConn`为每一个客户端创建了一个`clientWriter`的`goroutine`，用来接收向客户端发送消息的`channel`中的广播消息，并将它们写入到客户端的网络连接。客户端的读取循环会在`broadcaster`接收到`leaving`通知并关闭了`channel`后终止。

下面演示的是当服务器有两个活动的客户端连接，并且在两个窗口中运行的情况，使用`netcat`来聊天：

```go
$ go build gopl.io/ch8/chat
$ go build gopl.io/ch8/netcat3
$ ./chat &
$ ./netcat3
You are 127.0.0.1:64208               $ ./netcat3
127.0.0.1:64211 has arrived           You are 127.0.0.1:64211
Hi!
127.0.0.1:64208: Hi!                  127.0.0.1:64208: Hi!
                                      Hi yourself.
127.0.0.1:64211: Hi yourself.         127.0.0.1:64211: Hi yourself.
^C
                                      127.0.0.1:64208 has left
$ ./netcat3
You are 127.0.0.1:64216               127.0.0.1:64216 has arrived
                                      Welcome.
127.0.0.1:64211: Welcome.             127.0.0.1:64211: Welcome.
                                      ^C
127.0.0.1:64211 has left”
```

当与`n`个客户端保持聊天`session`时，这个程序会有`2n+2`个并发的`goroutine`，然而这个程序却并不需要显式的锁（§9.2）。`clients`这个`map`被限制在了一个独立的`goroutine`中，`broadcaster`，所以它不能被并发地访问。多个`goroutine`共享的变量只有这些`channel`和`net.Conn`的实例，两个东西都是并发安全的。我们会在下一章中更多地讲解约束，并发安全以及`goroutine`中共享变量的含义。

**练习 8.12：** 使`broadcaster`能够将`arrival`事件通知当前所有的客户端。这需要你在`clients`集合中，以及`entering`和`leaving`的`channel`中记录客户端的名字。



**练习 8.13：** 使聊天服务器能够断开空闲的客户端连接，比如最近五分钟之后没有发送任何消息的那些客户端。提示：可以在其它`goroutine`中调用`conn.Close()`来解除`Read`调用，就像`input.Scanner()`所做的那样。



**练习 8.14：** 修改聊天服务器的网络协议，这样每一个客户端就可以在`entering`时提供他们的名字。将消息前缀由之前的网络地址改为这个名字。



**练习 8.15：** 如果一个客户端没有及时地读取数据可能会导致所有的客户端被阻塞。修改`broadcaster`来跳过一条消息，而不是等待这个客户端一直到其准备好读写。或者为每一个客户端的消息发送`channel`建立缓冲区，这样大部分的消息便不会被丢掉；`broadcaster`应该用一个非阻塞的`send`向这个`channel`中发消息。