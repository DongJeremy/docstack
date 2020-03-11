# 第14章：协程（goroutine）与通道（channel）

作为一门 21 世纪的语言，Go 原生支持应用之间的通信（网络，客户端和服务端，分布式计算，参见第 15 章）和程序的并发。程序可以在不同的处理器和计算机上同时执行不同的代码段。Go 语言为构建并发程序的基本代码块是 协程 (goroutine) 与通道 (channel)。他们需要语言，编译器，和runtime的支持。Go 语言提供的垃圾回收器对并发编程至关重要。

**不要通过共享内存来通信，而通过通信来共享内存。**

通信强制协作。


## 14.1 并发、并行和协程

### 14.1.1 什么是协程

一个应用程序是运行在机器上的一个进程；进程是一个运行在自己内存地址空间里的独立执行体。一个进程由一个或多个操作系统线程组成，这些线程其实是共享同一个内存地址空间的一起工作的执行体。几乎所有'正式'的程序都是多线程的，以便让用户或计算机不必等待，或者能够同时服务多个请求（如 Web 服务器），或增加性能和吞吐量（例如，通过对不同的数据集并行执行代码）。一个并发程序可以在一个处理器或者内核上使用多个线程来执行任务，但是只有同一个程序在某个时间点同时运行在多核或者多处理器上才是真正的并行。

并行是一种通过使用多处理器以提高速度的能力。所以并发程序可以是并行的，也可以不是。

公认的，使用多线程的应用难以做到准确，最主要的问题是内存中的数据共享，它们会被多线程以无法预知的方式进行操作，导致一些无法重现或者随机的结果（称作 `竞态`）。

**不要使用全局变量或者共享内存，它们会给你的代码在并发运算的时候带来危险。**

解决之道在于同步不同的线程，对数据加锁，这样同时就只有一个线程可以变更数据。在 Go 的标准库 `sync` 中有一些工具用来在低级别的代码中实现加锁；我们在第 [9.3](09.3.md) 节中讨论过这个问题。不过过去的软件开发经验告诉我们这会带来更高的复杂度，更容易使代码出错以及更低的性能，所以这个经典的方法明显不再适合现代多核/多处理器编程：`thread-per-connection` 模型不够有效。

Go 更倾向于其他的方式，在诸多比较合适的范式中，有个被称作 `Communicating Sequential Processes（顺序通信处理）`（CSP, C. Hoare 发明的）还有一个叫做 `message passing-model（消息传递）`（已经运用在了其他语言中，比如 Erlang）。

在 Go 中，应用程序并发处理的部分被称作 `goroutines（协程）`，它可以进行更有效的并发运算。在协程和操作系统线程之间并无一对一的关系：协程是根据一个或多个线程的可用性，映射（多路复用，执行于）在他们之上的；协程调度器在 Go 运行时很好的完成了这个工作。

协程工作在相同的地址空间中，所以共享内存的方式一定是同步的；这个可以使用 `sync` 包来实现（参见第 [9.3](09.3.md) 节），不过我们很不鼓励这样做：Go 使用 `channels` 来同步协程（可以参见第 [14.2](14.2.md) 节等章节）

当系统调用（比如等待 I/O）阻塞协程时，其他协程会继续在其他线程上工作。协程的设计隐藏了许多线程创建和管理方面的复杂工作。

协程是轻量的，比线程更轻。它们痕迹非常不明显（使用少量的内存和资源）：使用 4K 的栈内存就可以在堆中创建它们。因为创建非常廉价，必要的时候可以轻松创建并运行大量的协程（在同一个地址空间中 100,000 个连续的协程）。并且它们对栈进行了分割，从而动态的增加（或缩减）内存的使用；栈的管理是自动的，但不是由垃圾回收器管理的，而是在协程退出后自动释放。

协程可以运行在多个操作系统线程之间，也可以运行在线程之内，让你可以很小的内存占用就可以处理大量的任务。由于操作系统线程上的协程时间片，你可以使用少量的操作系统线程就能拥有任意多个提供服务的协程，而且 Go 运行时可以聪明的意识到哪些协程被阻塞了，暂时搁置它们并处理其他协程。

存在两种并发方式：确定性的（明确定义排序）和非确定性的（加锁/互斥从而未定义排序）。Go 的协程和通道理所当然的支持确定性的并发方式（例如通道具有一个 sender 和一个 receiver）。我们会在第 [14.7](14.7.md) 节中使用一个常见的算法问题（工人问题）来对比两种处理方式。

协程是通过使用关键字 `go` 调用（执行）一个函数或者方法来实现的（也可以是匿名或者 lambda 函数）。这样会在当前的计算过程中开始一个同时进行的函数，在相同的地址空间中并且分配了独立的栈，比如：`go sum(bigArray)`，在后台计算总和。

协程的栈会根据需要进行伸缩，不出现栈溢出；开发者不需要关心栈的大小。当协程结束的时候，它会静默退出：用来启动这个协程的函数不会得到任何的返回值。

任何 Go 程序都必须有的 `main()` 函数也可以看做是一个协程，尽管它并没有通过 `go` 来启动。协程可以在程序初始化的过程中运行（在 `init()` 函数中）。

在一个协程中，比如它需要进行非常密集的运算，你可以在运算循环中周期的使用 `runtime.Gosched()`：这会让出处理器，允许运行其他协程；它并不会使当前协程挂起，所以它会自动恢复执行。使用 `Gosched()` 可以使计算均匀分布，使通信不至于迟迟得不到响应。

### 14.1.2 并发和并行的差异

Go 的并发原语提供了良好的并发设计基础：表达程序结构以便表示独立地执行的动作；所以Go的重点不在于并行的首要位置：并发程序可能是并行的，也可能不是。并行是一种通过使用多处理器以提高速度的能力。但往往是，一个设计良好的并发程序在并行方面的表现也非常出色。

在当前的运行时（2012 年一月）实现中，Go 默认没有并行指令，只有一个独立的核心或处理器被专门用于 Go 程序，不论它启动了多少个协程；所以这些协程是并发运行的，但他们不是并行运行的：同一时间只有一个协程会处在运行状态。

这个情况在以后可能会发生改变，不过届时，为了使你的程序可以使用多个核心运行，这时协程就真正的是并行运行了，你必须使用 `GOMAXPROCS` 变量。

这会告诉运行时有多少个协程同时执行。

并且只有 gc 编译器真正实现了协程，适当的把协程映射到操作系统线程。使用 `gccgo` 编译器，会为每一个协程创建操作系统线程。

### 14.1.3 使用 GOMAXPROCS

在 gc 编译器下（6g 或者 8g）你必须设置 GOMAXPROCS 为一个大于默认值 1 的数值来允许运行时支持使用多于 1 个的操作系统线程，所有的协程都会共享同一个线程除非将 GOMAXPROCS 设置为一个大于 1 的数。当 GOMAXPROCS 大于 1 时，会有一个线程池管理许多的线程。通过 `gccgo` 编译器 GOMAXPROCS 有效的与运行中的协程数量相等。假设 n 是机器上处理器或者核心的数量。如果你设置环境变量 GOMAXPROCS>=n，或者执行 `runtime.GOMAXPROCS(n)`，接下来协程会被分割（分散）到 n 个处理器上。更多的处理器并不意味着性能的线性提升。有这样一个经验法则，对于 n 个核心的情况设置 GOMAXPROCS 为 n-1 以获得最佳性能，也同样需要遵守这条规则：协程的数量 > 1 + GOMAXPROCS > 1。

所以如果在某一时间只有一个协程在执行，不要设置 GOMAXPROCS！

还有一些通过实验观察到的现象：在一台 1 颗 CPU 的笔记本电脑上，增加 GOMAXPROCS 到 9 会带来性能提升。在一台 32 核的机器上，设置 GOMAXPROCS=8 会达到最好的性能，在测试环境中，更高的数值无法提升性能。如果设置一个很大的 GOMAXPROCS 只会带来轻微的性能下降；设置 GOMAXPROCS=100，使用 `top` 命令和 `H` 选项查看到只有 7 个活动的线程。

增加 GOMAXPROCS 的数值对程序进行并发计算是有好处的；

请看 [goroutine_select2.go](examples/chapter_14/goroutine_select2.go)

总结：GOMAXPROCS 等同于（并发的）线程数量，在一台核心数多于1个的机器上，会尽可能有等同于核心数的线程在并行运行。

### 14.1.4 如何用命令行指定使用的核心数量

使用 `flags` 包，如下：

```go
var numCores = flag.Int("n", 2, "number of CPU cores to use")
```
在 `main()` 中：
```go
flag.Parse()
runtime.GOMAXPROCS(*numCores)
```

协程可以通过调用`runtime.Goexit()`来停止，尽管这样做几乎没有必要。

示例 14.1-[goroutine1.go](examples/chapter_14/goroutine1.go) 介绍了概念：

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	fmt.Println("In main()")
	go longWait()
	go shortWait()
	fmt.Println("About to sleep in main()")
	// sleep works with a Duration in nanoseconds (ns) !
	time.Sleep(10 * 1e9)
	fmt.Println("At the end of main()")
}

func longWait() {
	fmt.Println("Beginning longWait()")
	time.Sleep(5 * 1e9) // sleep for 5 seconds
	fmt.Println("End of longWait()")
}

func shortWait() {
	fmt.Println("Beginning shortWait()")
	time.Sleep(2 * 1e9) // sleep for 2 seconds
	fmt.Println("End of shortWait()")
}
```

输出：

```go
In main()
About to sleep in main()
Beginning longWait()
Beginning shortWait()
End of shortWait()
End of longWait()
At the end of main() // after 10s
```

`main()`，`longWait()` 和 `shortWait()` 三个函数作为独立的处理单元按顺序启动，然后开始并行运行。每一个函数都在运行的开始和结束阶段输出了消息。为了模拟他们运算的时间消耗，我们使用了 `time` 包中的 `Sleep` 函数。`Sleep()` 可以按照指定的时间来暂停函数或协程的执行，这里使用了纳秒（ns，符号 1e9 表示 1 乘 10 的 9 次方，e=指数）。

他们按照我们期望的顺序打印出了消息，几乎都一样，可是我们明白这是模拟出来的，以并行的方式。我们让 `main()` 函数暂停 10 秒从而确定它会在另外两个协程之后结束。如果不这样（如果我们让 `main()` 函数停止 4 秒），`main()` 会提前结束，`longWait()` 则无法完成。如果我们不在 `main()` 中等待，协程会随着程序的结束而消亡。

当 `main()` 函数返回的时候，程序退出：它不会等待任何其他非 main 协程的结束。这就是为什么在服务器程序中，每一个请求都会启动一个协程来处理，`server()` 函数必须保持运行状态。通常使用一个无限循环来达到这样的目的。

另外，协程是独立的处理单元，一旦陆续启动一些协程，你无法确定他们是什么时候真正开始执行的。你的代码逻辑必须独立于协程调用的顺序。

为了对比使用一个线程，连续调用的情况，移除 go 关键字，重新运行程序。

现在输出：

```go
In main()
Beginning longWait()
End of longWait()
Beginning shortWait()
End of shortWait()
About to sleep in main()
At the end of main() // after 17 s
```

协程更有用的一个例子应该是在一个非常长的数组中查找一个元素。

将数组分割为若干个不重复的切片，然后给每一个切片启动一个协程进行查找计算。这样许多并行的协程可以用来进行查找任务，整体的查找时间会缩短（除以协程的数量）。

### 14.1.5 Go 协程（goroutines）和协程（coroutines）

> （译者注：标题中的“Go协程（goroutines）” 即是 14 章讲的协程指的是 Go 语言中的协程。而“协程（coroutines）”指的是其他语言中的协程概念，仅在本节出现。）

在其他语言中，比如 C#，Lua 或者 Python 都有协程的概念。这个名字表明它和 Go协程有些相似，不过有两点不同：

- Go 协程意味着并行（或者可以以并行的方式部署），协程一般来说不是这样的
- Go 协程通过通道来通信；协程通过让出和恢复操作来通信

Go 协程比协程更强大，也很容易从协程的逻辑复用到 Go 协程。

## 14.2 协程间的信道

### 14.2.1 概念

在第一个例子中，协程是独立执行的，他们之间没有通信。他们必须通信才会变得更有用：彼此之间发送和接收信息并且协调/同步他们的工作。协程可以使用共享变量来通信，但是很不提倡这样做，因为这种方式给所有的共享内存的多线程都带来了困难。

而 Go 有一种特殊的类型，*通道（channel）*，就像一个可以用于发送类型化数据的管道，由其负责协程之间的通信，从而避开所有由共享内存导致的陷阱；这种通过通道进行通信的方式保证了同步性。数据在通道中进行传递：*在任何给定时间，一个数据被设计为只有一个协程可以对其访问，所以不会发生数据竞争。* 数据的所有权（可以读写数据的能力）也因此被传递。

工厂的传送带是个很有用的例子。一个机器（生产者协程）在传送带上放置物品，另外一个机器（消费者协程）拿到物品并打包。

通道服务于通信的两个目的：值的交换，同步的，保证了两个计算（协程）任何时候都是可知状态。

![](images/14.2_fig14.1.png?raw=true)

通常使用这样的格式来声明通道：`var identifier chan datatype`

未初始化的通道的值是nil。

所以通道只能传输一种类型的数据，比如 `chan int` 或者 `chan string`，所有的类型都可以用于通道，空接口 `interface{}` 也可以。甚至可以（有时非常有用）创建通道的通道。

通道实际上是类型化消息的队列：使数据得以传输。它是先进先出（FIFO）的结构所以可以保证发送给他们的元素的顺序（有些人知道，通道可以比作 Unix shells 中的双向管道（two-way pipe））。通道也是引用类型，所以我们使用 `make()` 函数来给它分配内存。这里先声明了一个字符串通道 ch1，然后创建了它（实例化）：

```go
var ch1 chan string
ch1 = make(chan string)
```

当然可以更短： `ch1 := make(chan string)`。

这里我们构建一个int通道的通道： `chanOfChans := make(chan int)`。

或者函数通道：`funcChan := make(chan func())`（相关示例请看第 [14.17](14.17.md) 节）。

所以通道是第一类对象：可以存储在变量中，作为函数的参数传递，从函数返回以及通过通道发送它们自身。另外它们是类型化的，允许类型检查，比如尝试使用整数通道发送一个指针。

### 14.2.2 通信操作符 <-

这个操作符直观的标示了数据的传输：信息按照箭头的方向流动。

流向通道（发送）

`ch <- int1` 表示：用通道 ch 发送变量 int1（双目运算符，中缀 = 发送）

从通道流出（接收），三种方式：

`int2 = <- ch` 表示：变量 int2 从通道 ch（一元运算的前缀操作符，前缀 = 接收）接收数据（获取新值）；假设 int2 已经声明过了，如果没有的话可以写成：`int2 := <- ch`。

`<- ch` 可以单独调用获取通道的（下一个）值，当前值会被丢弃，但是可以用来验证，所以以下代码是合法的：

```go
if <- ch != 1000{
	...
}
```

同一个操作符 `<-` 既用于**发送**也用于**接收**，但Go会根据操作对象弄明白该干什么 。虽非强制要求，但为了可读性通道的命名通常以 `ch` 开头或者包含 `chan`。通道的发送和接收都是原子操作：它们总是互不干扰的完成的。下面的示例展示了通信操作符的使用。

示例 14.2-[goroutine2.go](examples/chapter_14/goroutine2.go)

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	ch := make(chan string)

	go sendData(ch)
	go getData(ch)

	time.Sleep(1e9)
}

func sendData(ch chan string) {
	ch <- "Washington"
	ch <- "Tripoli"
	ch <- "London"
	ch <- "Beijing"
	ch <- "Tokyo"
}

func getData(ch chan string) {
	var input string
	// time.Sleep(2e9)
	for {
		input = <-ch
		fmt.Printf("%s ", input)
	}
}
```

输出：

```
Washington Tripoli London Beijing tokyo
```

`main()` 函数中启动了两个协程：`sendData()` 通过通道 ch 发送了 5 个字符串，`getData()` 按顺序接收它们并打印出来。

如果 2 个协程需要通信，你必须给他们同一个通道作为参数才行。

尝试一下如果注释掉 `time.Sleep(1e9)` 会如何。

我们发现协程之间的同步非常重要：

- main() 等待了 1 秒让两个协程完成，如果不这样，sendData() 就没有机会输出。
- getData() 使用了无限循环：它随着 sendData() 的发送完成和 ch 变空也结束了。
- 如果我们移除一个或所有 `go` 关键字，程序无法运行，Go 运行时会抛出 panic：

```
---- Error run E:/Go/Goboek/code examples/chapter 14/goroutine2.exe with code Crashed ---- Program exited with code -2147483645: panic: all goroutines are asleep-deadlock!
```

为什么会这样？运行时（runtime）会检查所有的协程（像本例中只有一个）是否在等待着什么东西（可从某个通道读取或者写入某个通道），这意味着程序将无法继续执行。这是死锁（deadlock）的一种形式，而运行时（runtime）可以为我们检测到这种情况。

注意：不要使用打印状态来表明通道的发送和接收顺序：由于打印状态和通道实际发生读写的时间延迟会导致和真实发生的顺序不同。

练习 14.4：解释一下为什么如果在函数 `getData()` 的一开始插入 `time.Sleep(2e9)`，不会出现错误但也没有输出呢。

### 14.2.3 通道阻塞

默认情况下，通信是同步且无缓冲的：在有接受者接收数据之前，发送不会结束。可以想象一个无缓冲的通道在没有空间来保存数据的时候：必须要一个接收者准备好接收通道的数据然后发送者可以直接把数据发送给接收者。所以通道的发送/接收操作在对方准备好之前是阻塞的：

1）对于同一个通道，发送操作（协程或者函数中的），在接收者准备好之前是阻塞的：如果ch中的数据无人接收，就无法再给通道传入其他数据：新的输入无法在通道非空的情况下传入。所以发送操作会等待 ch 再次变为可用状态：就是通道值被接收时（可以传入变量）。

2）对于同一个通道，接收操作是阻塞的（协程或函数中的），直到发送者可用：如果通道中没有数据，接收者就阻塞了。

尽管这看上去是非常严格的约束，实际在大部分情况下工作的很不错。

程序 `channel_block.go` 验证了以上理论，一个协程在无限循环中给通道发送整数数据。不过因为没有接收者，只输出了一个数字 0。

示例 14.3-[channel_block.go](examples/chapter_14/channel_block.go)

```go
package main

import "fmt"

func main() {
	ch1 := make(chan int)
	go pump(ch1)       // pump hangs
	fmt.Println(<-ch1) // prints only 0
}

func pump(ch chan int) {
	for i := 0; ; i++ {
		ch <- i
	}
}
```

输出：

```
0
```

`pump()` 函数为通道提供数值，也被叫做生产者。

为通道解除阻塞定义了 `suck` 函数来在无限循环中读取通道，参见示例 14.4-[channel_block2.go](examples/chapter_14/channel_block2.go)：

```go
func suck(ch chan int) {
	for {
		fmt.Println(<-ch)
	}
}
```
在 `main()` 中使用协程开始它：

```go
go pump(ch1)
go suck(ch1)
time.Sleep(1e9)
```

给程序 1 秒的时间来运行：输出了上万个整数。

练习 14.1：[channel_block3.go](exercises/chapter_14/channel_block3.go)：写一个通道证明它的阻塞性，开启一个协程接收通道的数据，持续 15 秒，然后给通道放入一个值。在不同的阶段打印消息并观察输出。

### 14.2.4 通过一个（或多个）通道交换数据进行协程同步。

通信是一种同步形式：通过通道，两个协程在通信（协程会和）中某刻同步交换数据。无缓冲通道成为了多个协程同步的完美工具。

甚至可以在通道两端互相阻塞对方，形成了叫做死锁的状态。Go 运行时会检查并 panic，停止程序。死锁几乎完全是由糟糕的设计导致的。

无缓冲通道会被阻塞。设计无阻塞的程序可以避免这种情况，或者使用带缓冲的通道。

练习 14.2： [blocking.go](exercises/chapter_14/blocking.go)

解释为什么下边这个程序会导致 panic：所有的协程都休眠了 - 死锁！

```go
package main

import (
	"fmt"
)

func f1(in chan int) {
	fmt.Println(<-in)
}

func main() {
	out := make(chan int)
	out <- 2
	go f1(out)
}
```

### 14.2.5 同步通道-使用带缓冲的通道

一个无缓冲通道只能包含 1 个元素，有时显得很局限。我们给通道提供了一个缓存，可以在扩展的 `make` 命令中设置它的容量，如下：

```go
buf := 100
ch1 := make(chan string, buf)
```

buf 是通道可以同时容纳的元素（这里是 string）个数

在缓冲满载（缓冲被全部使用）之前，给一个带缓冲的通道发送数据是不会阻塞的，而从通道读取数据也不会阻塞，直到缓冲空了。

缓冲容量和类型无关，所以可以（尽管可能导致危险）给一些通道设置不同的容量，只要他们拥有同样的元素类型。内置的 `cap` 函数可以返回缓冲区的容量。

如果容量大于 0，通道就是异步的了：缓冲满载（发送）或变空（接收）之前通信不会阻塞，元素会按照发送的顺序被接收。如果容量是0或者未设置，通信仅在收发双方准备好的情况下才可以成功。

同步：`ch :=make(chan type, value)`

- value == 0 -> synchronous, unbuffered (阻塞）
- value > 0 -> asynchronous, buffered（非阻塞）取决于value元素

若使用通道的缓冲，你的程序会在“请求”激增的时候表现更好：更具弹性，专业术语叫：更具有伸缩性（scalable）。在设计算法时首先考虑使用无缓冲通道，只在不确定的情况下使用缓冲。

练习 14.3：[channel_buffer.go](exercises/chapter_14/channel_buffer.go)：给 [channel_block3.go](exercises/chapter_14/channel_block3.go) 的通道增加缓冲并观察输出有何不同。

### 14.2.6 协程中用通道输出结果

为了知道计算何时完成，可以通过信道回报。在例子 `go sum(bigArray)` 中，要这样写：

```go
ch := make(chan int)
go sum(bigArray, ch) // bigArray puts the calculated sum on ch
// .. do something else for a while
sum := <- ch // wait for, and retrieve the sum
```

也可以使用通道来达到同步的目的，这个很有效的用法在传统计算机中称为信号量（semaphore）。或者换个方式：通过通道发送信号告知处理已经完成（在协程中）。

在其他协程运行时让 main 程序无限阻塞的通常做法是在 `main` 函数的最后放置一个 `select {}`。

也可以使用通道让 `main` 程序等待协程完成，就是所谓的信号量模式，我们会在接下来的部分讨论。

### 14.2.7 信号量模式

下边的片段阐明：协程通过在通道 `ch` 中放置一个值来处理结束的信号。`main` 协程等待 `<-ch` 直到从中获取到值。

我们期望从这个通道中获取返回的结果，像这样：

```go
func compute(ch chan int){
	ch <- someComputation() // when it completes, signal on the channel.
}

func main(){
	ch := make(chan int) 	// allocate a channel.
	go compute(ch)		// start something in a goroutines
	doSomethingElseForAWhile()
	result := <- ch
}
```

这个信号也可以是其他的，不返回结果，比如下面这个协程中的匿名函数（lambda）协程：

```go
ch := make(chan int)
go func(){
	// doSomething
	ch <- 1 // Send a signal; value does not matter
}()
doSomethingElseForAWhile()
<- ch	// Wait for goroutine to finish; discard sent value.
```

或者等待两个协程完成，每一个都会对切片s的一部分进行排序，片段如下：

```go
done := make(chan bool)
// doSort is a lambda function, so a closure which knows the channel done:
doSort := func(s []int){
	sort(s)
	done <- true
}
i := pivot(s)
go doSort(s[:i])
go doSort(s[i:])
<-done
<-done
```

下边的代码，用完整的信号量模式对长度为N的 float64 切片进行了 N 个` doSomething()` 计算并同时完成，通道 sem 分配了相同的长度（且包含空接口类型的元素），待所有的计算都完成后，发送信号（通过放入值）。在循环中从通道 sem 不停的接收数据来等待所有的协程完成。

```go
type Empty interface {}
var empty Empty
...
data := make([]float64, N)
res := make([]float64, N)
sem := make(chan Empty, N)
...
for i, xi := range data {
	go func (i int, xi float64) {
		res[i] = doSomething(i, xi)
		sem <- empty
	} (i, xi)
}
// wait for goroutines to finish
for i := 0; i < N; i++ { <-sem }
```

注意上述代码中闭合函数的用法：`i`、`xi` 都是作为参数传入闭合函数的，这一做法使得每个协程（译者注：在其启动时）获得一份 `i` 和 `xi` 的单独拷贝，从而向闭合函数内部屏蔽了外层循环中的 `i` 和 `xi`变量；否则，for 循环的下一次迭代会更新所有协程中 `i` 和 `xi` 的值。另一方面，切片 `res` 没有传入闭合函数，因为协程不需要`res`的单独拷贝。切片 `res` 也在闭合函数中但并不是参数。


### 14.2.8 实现并行的 for 循环

在上一部分章节 [14.2.7](14.2.md#1427-信号量模式) 的代码片段中：for 循环的每一个迭代是并行完成的：

```go
for i, v := range data {
	go func (i int, v float64) {
		doSomething(i, v)
		...
	} (i, v)
}
```

在 for 循环中并行计算迭代可能带来很好的性能提升。不过所有的迭代都必须是独立完成的。有些语言比如 Fortress 或者其他并行框架以不同的结构实现了这种方式，在 Go 中用协程实现起来非常容易：

### 14.2.9 用带缓冲通道实现一个信号量

信号量是实现互斥锁（排外锁）常见的同步机制，限制对资源的访问，解决读写问题，比如没有实现信号量的 `sync` 的 Go 包，使用带缓冲的通道可以轻松实现：

- 带缓冲通道的容量和要同步的资源容量相同
- 通道的长度（当前存放的元素个数）与当前资源被使用的数量相同
- 容量减去通道的长度就是未处理的资源个数（标准信号量的整数值）

不用管通道中存放的是什么，只关注长度；因此我们创建了一个长度可变但容量为0（字节）的通道：

```go
type Empty interface {}
type semaphore chan Empty
```

将可用资源的数量N来初始化信号量 `semaphore`：`sem = make(semaphore, N)`

然后直接对信号量进行操作：

```go
// acquire n resources
func (s semaphore) P(n int) {
	e := new(Empty)
	for i := 0; i < n; i++ {
		s <- e
	}
}

// release n resources
func (s semaphore) V(n int) {
	for i:= 0; i < n; i++{
		<- s
	}
}
```

可以用来实现一个互斥的例子：

```go
/* mutexes */
func (s semaphore) Lock() {
	s.P(1)
}

func (s semaphore) Unlock(){
	s.V(1)
}

/* signal-wait */
func (s semaphore) Wait(n int) {
	s.P(n)
}

func (s semaphore) Signal() {
	s.V(1)
}
```

练习 14.5：[gosum.go](exercises/chapter_14/gosum.go)：用这种习惯用法写一个程序，开启一个协程来计算2个整数的和并等待计算结果并打印出来。

练习 14.6：[producer_consumer.go](exercises/chapter_14/producer_consumer.go)：用这种习惯用法写一个程序，有两个协程，第一个提供数字 0，10，20，...90 并将他们放入通道，第二个协程从通道中读取并打印。`main()` 等待两个协程完成后再结束。

习惯用法：通道工厂模式

编程中常见的另外一种模式如下：不将通道作为参数传递给协程，而用函数来生成一个通道并返回（工厂角色）；函数内有个匿名函数被协程调用。

在 [channel_block2.go](examples/chapter_14/channel_block2.go) 加入这种模式便有了示例 14.5-[channel_idiom.go](examples/chapter_14/channel_idiom.go)：

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	stream := pump()
	go suck(stream)
	time.Sleep(1e9)
}

func pump() chan int {
	ch := make(chan int)
	go func() {
		for i := 0; ; i++ {
			ch <- i
		}
	}()
	return ch
}

func suck(ch chan int) {
	for {
		fmt.Println(<-ch)
	}
}
```

### 14.2.10 给通道使用 for 循环

`for` 循环的 `range` 语句可以用在通道 `ch` 上，便可以从通道中获取值，像这样：

```go
for v := range ch {
	fmt.Printf("The value is %v\n", v)
}
```

它从指定通道中读取数据直到通道关闭，才继续执行下边的代码。很明显，另外一个协程必须写入 `ch`（不然代码就阻塞在 for 循环了），而且必须在写入完成后才关闭。`suck` 函数可以这样写，且在协程中调用这个动作，程序变成了这样：

示例 14.6-[channel_idiom2.go](examples/chapter_14/channel_idiom2.go)：

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	suck(pump())
	time.Sleep(1e9)
}

func pump() chan int {
	ch := make(chan int)
	go func() {
		for i := 0; ; i++ {
			ch <- i
		}
	}()
	return ch
}

func suck(ch chan int) {
	go func() {
		for v := range ch {
			fmt.Println(v)
		}
	}()
}
```

习惯用法：通道迭代模式

这个模式用到了后边14.6章示例 [producer_consumer.go](exercises/chapter_14/producer_consumer.go) 的生产者-消费者模式，通常，需要从包含了地址索引字段 items 的容器给通道填入元素。为容器的类型定义一个方法 `Iter()`，返回一个只读的通道（参见第 [14.2.11](14.2.md#14211-通道的方向) 节）items，如下：

```go
func (c *container) Iter () <- chan item {
	ch := make(chan item)
	go func () {
		for i:= 0; i < c.Len(); i++{	// or use a for-range loop
			ch <- c.items[i]
		}
	} ()
	return ch
}
```

在协程里，一个 for 循环迭代容器 c 中的元素（对于树或图的算法，这种简单的 for 循环可以替换为深度优先搜索）。

调用这个方法的代码可以这样迭代容器：

```go
for x := range container.Iter() { ... }
```

其运行在自己启动的协程中，所以上边的迭代用到了一个通道和两个协程（可能运行在不同的线程上）。 这样我们就有了一个典型的生产者-消费者模式。如果程序在协程给通道写完值之前结束，则这个协程不会被垃圾回收；这是设计使然。这种看起来并不符合预期的行为正是由通道这种线程安全的通信方式所导致的。如此一来，一个协程为了写入一个永远无人读取的通道而被挂起就成了一个bug，而并非你预想中的那样被悄悄回收掉（garbage-collected）了。

习惯用法：生产者消费者模式

假设你有 `Produce()` 函数来产生 `Consume` 函数需要的值。它们都可以运行在独立的协程中，生产者在通道中放入给消费者读取的值。整个处理过程可以替换为无限循环：

```go
for {
	Consume(Produce())
}
```

### 14.2.11 通道的方向

通道类型可以用注解来表示它只发送或者只接收：

```go
var send_only chan<- int 		// channel can only receive data
var recv_only <-chan int		// channel can only send data
```

只接收的通道（chan<- T）无法关闭，因为关闭通道是发送者用来表示不再给通道发送值了，所以对只接收通道是没有意义的。通道创建的时候都是双向的，但也可以分配有方向的通道变量，就像以下代码：

```go
var c = make(chan int) // bidirectional
go source(c)
go sink(c)

func source(ch chan<- int){
	for { ch <- 1 }
}

func sink(ch <-chan int) {
	for { <-ch }
}
```

习惯用法：管道和选择器模式

更具体的例子还有协程处理它从通道接收的数据并发送给输出通道：

```go
sendChan := make(chan int)
receiveChan := make(chan string)
go processChannel(sendChan, receiveChan)

func processChannel(in <-chan int, out chan<- string) {
	for inValue := range in {
		result := ... /// processing inValue
		out <- result
	}
}
```

通过使用方向注解来限制协程对通道的操作。

这里有一个来自 Go 指导的很赞的例子，打印了输出的素数，使用选择器（‘筛’）作为它的算法。每个 prime 都有一个选择器，如下图：

![](images/14.2_fig14.2.png?raw=true)

版本1：示例 14.7-[sieve1.go](examples/chapter_14/sieve1.go)

```go
// Copyright 2009 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.package main
package main

import "fmt"

// Send the sequence 2, 3, 4, ... to channel 'ch'.
func generate(ch chan int) {
	for i := 2; ; i++ {
		ch <- i // Send 'i' to channel 'ch'.
	}
}

// Copy the values from channel 'in' to channel 'out',
// removing those divisible by 'prime'.
func filter(in, out chan int, prime int) {
	for {
		i := <-in // Receive value of new variable 'i' from 'in'.
		if i%prime != 0 {
			out <- i // Send 'i' to channel 'out'.
		}
	}
}

// The prime sieve: Daisy-chain filter processes together.
func main() {
	ch := make(chan int) // Create a new channel.
	go generate(ch)      // Start generate() as a goroutine.
	for {
		prime := <-ch
		fmt.Print(prime, " ")
		ch1 := make(chan int)
		go filter(ch, ch1, prime)
		ch = ch1
	}
}
```

协程 `filter(in, out chan int, prime int)` 拷贝整数到输出通道，丢弃掉可以被 prime 整除的数字。然后每个 prime 又开启了一个新的协程，生成器和选择器并发请求。

输出：

```
2 3 5 7 11 13 17 19 23 29 31 37 41 43 47 53 59 61 67 71 73 79 83 89 97 101
103 107 109 113 127 131 137 139 149 151 157 163 167 173 179 181 191 193 197 199 211 223
227 229 233 239 241 251 257 263 269 271 277 281 283 293 307 311 313 317 331 337 347 349
353 359 367 373 379 383 389 397 401 409 419 421 431 433 439 443 449 457 461 463 467 479
487 491 499 503 509 521 523 541 547 557 563 569 571 577 587 593 599 601 607 613 617 619
631 641 643 647 653 659 661 673 677 683 691 701 709 719 727 733 739 743 751 757 761 769
773 787 797 809 811 821 823 827 829 839 853 857 859 863 877 881 883 887 907 911 919 929
937 941 947 953 967 971 977 983 991 997 1009 1013...
```

第二个版本引入了上边的习惯用法：函数 `sieve`、`generate` 和 `filter` 都是工厂；它们创建通道并返回，而且使用了协程的 lambda 函数。`main` 函数现在短小清晰：它调用 `sieve()` 返回了包含素数的通道，然后通过 `fmt.Println(<-primes)` 打印出来。

版本2：示例 14.8-[sieve2.go](examples/chapter_14/sieve2.go)

```go
// Copyright 2009 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.

package main

import (
	"fmt"
)

// Send the sequence 2, 3, 4, ... to returned channel
func generate() chan int {
	ch := make(chan int)
	go func() {
		for i := 2; ; i++ {
			ch <- i
		}
	}()
	return ch
}

// Filter out input values divisible by 'prime', send rest to returned channel
func filter(in chan int, prime int) chan int {
	out := make(chan int)
	go func() {
		for {
			if i := <-in; i%prime != 0 {
				out <- i
			}
		}
	}()
	return out
}

func sieve() chan int {
	out := make(chan int)
	go func() {
		ch := generate()
		for {
			prime := <-ch
			ch = filter(ch, prime)
			out <- prime
		}
	}()
	return out
}

func main() {
	primes := sieve()
	for {
		fmt.Println(<-primes)
	}
}
```

## 14.3 协程的同步：关闭通道-测试阻塞的通道

通道可以被显式的关闭；尽管它们和文件不同：不必每次都关闭。只有在当需要告诉接收者不会再提供新的值的时候，才需要关闭通道。只有发送者需要关闭通道，接收者永远不会需要。

继续看示例 [goroutine2.go](examples/chapter_14/goroutine2.go)（示例 14.2）：我们如何在通道的 `sendData()` 完成的时候发送一个信号，`getData()` 又如何检测到通道是否关闭或阻塞？

第一个可以通过函数 `close(ch)` 来完成：这个将通道标记为无法通过发送操作 `<-` 接受更多的值；给已经关闭的通道发送或者再次关闭都会导致运行时的 panic。在创建一个通道后使用 defer 语句是个不错的办法（类似这种情况）：

```go
ch := make(chan float64)
defer close(ch)
```

第二个问题可以使用逗号，ok 操作符：用来检测通道是否被关闭。

如何来检测可以收到没有被阻塞（或者通道没有被关闭）？

```go
v, ok := <-ch   // ok is true if v received value
```

通常和 if 语句一起使用：

```go
if v, ok := <-ch; ok {
  process(v)
}
```

或者在 for 循环中接收的时候，当关闭或者阻塞的时候使用 break：

```go
v, ok := <-ch
if !ok {
  break
}
process(v)
```

在示例程序 14.2 中使用这些可以改进为版本 goroutine3.go，输出相同。

实现非阻塞通道的读取，需要使用 select（参见第 [14.4](14.4.md) 节）。

示例 14.9-[goroutine3.go](examples/chapter_14/goroutine3.go)：

```go
package main

import "fmt"

func main() {
	ch := make(chan string)
	go sendData(ch)
	getData(ch)
}

func sendData(ch chan string) {
	ch <- "Washington"
	ch <- "Tripoli"
	ch <- "London"
	ch <- "Beijing"
	ch <- "Tokio"
	close(ch)
}

func getData(ch chan string) {
	for {
		input, open := <-ch
		if !open {
			break
		}
		fmt.Printf("%s ", input)
	}
}
```

改变了以下代码：

- 现在只有 `sendData()` 是协程，`getData()` 和 `main()` 在同一个线程中：

```go
go sendData(ch)
getData(ch)
```

- 在 `sendData()` 函数的最后，关闭了通道：

```go
func sendData(ch chan string) {
	ch <- "Washington"
	ch <- "Tripoli"
	ch <- "London"
	ch <- "Beijing"
	ch <- "Tokio"
	close(ch)
}
```

- 在 for 循环的 `getData()` 中，在每次接收通道的数据之前都使用 `if !open` 来检测：

```go
for {
		input, open := <-ch
		if !open {
			break
		}
		fmt.Printf("%s ", input)
	}
```

使用 for-range 语句来读取通道是更好的办法，因为这会自动检测通道是否关闭：

```go
for input := range ch {
  	process(input)
}
```

阻塞和生产者-消费者模式：

在第 14.2.10 节的通道迭代器中，两个协程经常是一个阻塞另外一个。如果程序工作在多核心的机器上，大部分时间只用到了一个处理器。可以通过使用带缓冲（缓冲空间大于 0）的通道来改善。比如，缓冲大小为 100，迭代器在阻塞之前，至少可以从容器获得 100 个元素。如果消费者协程在独立的内核运行，就有可能让协程不会出现阻塞。

由于容器中元素的数量通常是已知的，需要让通道有足够的容量放置所有的元素。这样，迭代器就不会阻塞（尽管消费者协程仍然可能阻塞）。然而，这实际上加倍了迭代容器所需要的内存使用量，所以通道的容量需要限制一下最大值。记录运行时间和性能测试可以帮助你找到最小的缓存容量带来最好的性能。

## 14.4 使用 select 切换协程

从不同的并发执行的协程中获取值可以通过关键字`select`来完成，它和`switch`控制语句非常相似（章节5.3）也被称作通信开关；它的行为像是“你准备好了吗”的轮询机制；`select`监听进入通道的数据，也可以是用通道发送值的时候。

```go
select {
case u:= <- ch1:
        ...
case v:= <- ch2:
        ...
        ...
default: // no value ready to be received
        ...
}
```

`default` 语句是可选的；fallthrough 行为，和普通的 switch 相似，是不允许的。在任何一个 case 中执行 `break` 或者 `return`，select 就结束了。

`select` 做的就是：选择处理列出的多个通信情况中的一个。

- 如果都阻塞了，会等待直到其中一个可以处理
- 如果多个可以处理，随机选择一个
- 如果没有通道操作可以处理并且写了 `default` 语句，它就会执行：`default` 永远是可运行的（这就是准备好了，可以执行）。

在 `select` 中使用发送操作并且有 `default` 可以确保发送不被阻塞！如果没有 `default`，select 就会一直阻塞。

`select` 语句实现了一种监听模式，通常用在（无限）循环中；在某种情况下，通过 `break` 语句使循环退出。

在程序 [goroutine_select.go](examples/chapter_14/goroutine_select.go) 中有 2 个通道 `ch1` 和 `ch2`，三个协程 `pump1()`、`pump2()` 和 `suck()`。这是一个典型的生产者消费者模式。在无限循环中，`ch1` 和 `ch2` 通过 `pump1()` 和 `pump2()` 填充整数；`suck()` 也是在无限循环中轮询输入的，通过 `select` 语句获取 `ch1` 和 `ch2` 的整数并输出。选择哪一个 case 取决于哪一个通道收到了信息。程序在 main 执行 1 秒后结束。

示例 14.10-[goroutine_select.go](examples/chapter_14/goroutine_select.go)：

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	ch1 := make(chan int)
	ch2 := make(chan int)

	go pump1(ch1)
	go pump2(ch2)
	go suck(ch1, ch2)

	time.Sleep(1e9)
}

func pump1(ch chan int) {
	for i := 0; ; i++ {
		ch <- i * 2
	}
}

func pump2(ch chan int) {
	for i := 0; ; i++ {
		ch <- i + 5
	}
}

func suck(ch1, ch2 chan int) {
	for {
		select {
		case v := <-ch1:
			fmt.Printf("Received on channel 1: %d\n", v)
		case v := <-ch2:
			fmt.Printf("Received on channel 2: %d\n", v)
		}
	}
}
```

输出：

```
Received on channel 2: 5
Received on channel 2: 6
Received on channel 1: 0
Received on channel 2: 7
Received on channel 2: 8
Received on channel 2: 9
Received on channel 2: 10
Received on channel 1: 2
Received on channel 2: 11
...
Received on channel 2: 47404
Received on channel 1: 94346
Received on channel 1: 94348
```

一秒内的输出非常惊人，如果我们给它计数（goroutine_select2.go），得到了 90000 个左右的数字。

### 练习：

练习 14.7：

- a）在练习 5.4 的 for_loop.go 中，有一个常见的 for 循环打印数字。在函数 `tel` 中实现一个 for 循环，用协程开始这个函数并在其中给通道发送数字。`main()` 线程从通道中获取并打印。不要使用 `time.Sleep()` 来同步：[goroutine_panic.go](exercises/chapter_14/goroutine_panic.go)
- b）也许你的方案有效，可能会引发运行时的 panic：`throw:all goroutines are asleep-deadlock!` 为什么会这样？你如何解决这个问题？[goroutine_close.go](exercises/chapter_14/goroutine_close.go)
- c）解决 a）的另外一种方式：使用一个额外的通道传递给协程，然后在结束的时候随便放点什么进去。`main()` 线程检查是否有数据发送给了这个通道，如果有就停止：[goroutine_select.go](exercises/chapter_14/goroutine_select.go)


练习 14.8：

从示例 [6.13 fibonacci.go](examples/chapter_6/fibonacci.go) 的斐波那契程序开始，制定解决方案，使斐波那契周期计算独立到协程中，并可以把结果发送给通道。

结束的时候关闭通道。`main()` 函数读取通道并打印结果：[goFibonacci.go](exercises/chapter_14/gofibonacci.go)

使用练习 [6.9 fibonacci2.go](exercises/chapter_6/fibonacci2.go) 中的算法写一个更短的 [gofibonacci2.go](exercises/chapter_14/gofibonacci2.go)

使用 `select` 语句来写，并让通道退出（[gofibonacci_select.go](exercises/chapter_14/gofibonacci_select.go)）

注意：当给结果计时并和 6.13 对比时，我们发现使用通道通信的性能开销有轻微削减；这个例子中的算法使用协程并非性能最好的选择；但是 [gofibonacci3](exercises/chapter_14/gofibonacci3.go) 方案使用了 2 个协程带来了 3 倍的提速。


练习 14.9：

做一个随机位生成器，程序可以提供无限的随机 0 或者 1 的序列：[random_bitgen.go](exercises/chapter_14/random_bitgen.go)

练习 14.10：[polar_to_cartesian.go](exercises/chapter_14/polar_to_cartesian.go)

（这是一种综合练习，使用到第 4、9、11 章和本章的内容。）写一个可交互的控制台程序，要求用户输入二位平面极坐标上的点（半径和角度（度））。计算对应的笛卡尔坐标系的点的 x 和 y 并输出。使用极坐标和笛卡尔坐标的结构体。

使用通道和协程：

- `channel1` 用来接收极坐标
- `channel2` 用来接收笛卡尔坐标

转换过程需要在协程中进行，从 channel1 中读取然后发送到 channel2。实际上做这种计算不提倡使用协程和通道，但是如果运算量很大很耗时，这种方案设计就非常合适了。

练习 14.11： [concurrent_pi.go](exercises/chapter_14/concurrent_pi.go) / [concurrent_pi2.go](exercises/chapter_14/concurrent_pi2.go)

使用以下序列在协程中计算 pi：开启一个协程来计算公式中的每一项并将结果放入通道，`main()` 函数收集并累加结果，打印出 pi 的近似值。

![](images/14.4_piseries.png?raw=true)

计算执行时间（参见第 [6.11](6.11.md) 节）

再次声明这只是为了一边练习协程的概念一边找点乐子。

如果你需要的话可使用 `math.pi` 中的 Pi；而且不使用协程会运算的更快。一个急速版本：使用 `GOMAXPROCS`，开启和 `GOMAXPROCS` 同样多个协程。

习惯用法：后台服务模式

服务通常是是用后台协程中的无限循环实现的，在循环中使用 `select` 获取并处理通道中的数据：

```go
// Backend goroutine.
func backend() {
	for {
		select {
		case cmd := <-ch1:
			// Handle ...
		case cmd := <-ch2:
			...
		case cmd := <-chStop:
			// stop server
		}
	}
}
```

在程序的其他地方给通道 `ch1`，`ch2` 发送数据，比如：通道 `stop` 用来清理结束服务程序。

另一种方式（但是不太灵活）就是（客户端）在 `chRequest` 上提交请求，后台协程循环这个通道，使用 `switch` 根据请求的行为来分别处理：

```go
func backend() {
	for req := range chRequest {
		switch req.Subjext() {
			case A1:  // Handle case ...
			case A2:  // Handle case ...
			default:
			  // Handle illegal request ..
			  // ...
		}
	}
}
```

## 14.5 通道、超时和计时器（Ticker）

`time` 包中有一些有趣的功能可以和通道组合使用。

其中就包含了 `time.Ticker` 结构体，这个对象以指定的时间间隔重复的向通道 C 发送时间值：

```go
type Ticker struct {
    C <-chan Time // the channel on which the ticks are delivered.
    // contains filtered or unexported fields
    ...
}
```

时间间隔的单位是 ns（纳秒，int64），在工厂函数 `time.NewTicker` 中以 `Duration` 类型的参数传入：`func NewTicker(dur) *Ticker`。

在协程周期性的执行一些事情（打印状态日志，输出，计算等等）的时候非常有用。

调用 `Stop()` 使计时器停止，在 `defer` 语句中使用。这些都很好的适应 `select` 语句:

```go
ticker := time.NewTicker(updateInterval)
defer ticker.Stop()
...
select {
case u:= <-ch1:
    ...
case v:= <-ch2:
    ...
case <-ticker.C:
    logState(status) // call some logging function logState
default: // no value ready to be received
    ...
}
```

`time.Tick()` 函数声明为 `Tick(d Duration) <-chan Time`，当你想返回一个通道而不必关闭它的时候这个函数非常有用：它以 d 为周期给返回的通道发送时间，d是纳秒数。如果需要像下边的代码一样，限制处理频率（函数 `client.Call()` 是一个 RPC 调用，这里暂不赘述（参见第 [15.9](15.9.md) 节）：

```go
import "time"

rate_per_sec := 10
var dur Duration = 1e9 / rate_per_sec
chRate := time.Tick(dur) // a tick every 1/10th of a second
for req := range requests {
    <- chRate // rate limit our Service.Method RPC calls
    go client.Call("Service.Method", req, ...)
}
```

这样只会按照指定频率处理请求：`chRate` 阻塞了更高的频率。每秒处理的频率可以根据机器负载（和/或）资源的情况而增加或减少。

问题 14.1：扩展上边的代码，思考如何承载周期请求数的暴增（提示：使用带缓冲通道和计时器对象）。

定时器（Timer）结构体看上去和计时器（Ticker）结构体的确很像（构造为 `NewTimer(d Duration)`），但是它只发送一次时间，在 `Dration d` 之后。

还有 `time.After(d)` 函数，声明如下：

```go
func After(d Duration) <-chan Time
```

在 `Duration d` 之后，当前时间被发到返回的通道；所以它和 `NewTimer(d).C` 是等价的；它类似 `Tick()`，但是 `After()` 只发送一次时间。下边有个很具体的示例，很好的阐明了 `select` 中 `default` 的作用：

示例 14.11：[timer_goroutine.go](examples/chapter_14/timer_goroutine.go)：

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	tick := time.Tick(1e8)
	boom := time.After(5e8)
	for {
		select {
		case <-tick:
			fmt.Println("tick.")
		case <-boom:
			fmt.Println("BOOM!")
			return
		default:
			fmt.Println("    .")
			time.Sleep(5e7)
		}
	}
}
```

输出：

```
    .
    .
tick.
    .
    .
tick.
    .
    .
tick.
    .
    .
tick.
    .
    .
tick.
BOOM!
```

习惯用法：简单超时模式

要从通道 `ch` 中接收数据，但是最多等待1秒。先创建一个信号通道，然后启动一个 `lambda` 协程，协程在给通道发送数据之前是休眠的：

```go
timeout := make(chan bool, 1)
go func() {
        time.Sleep(1e9) // one second
        timeout <- true
}()
```

然后使用 `select` 语句接收 `ch` 或者 `timeout` 的数据：如果 `ch` 在 1 秒内没有收到数据，就选择到了 `time` 分支并放弃了 `ch` 的读取。

```go
select {
    case <-ch:
        // a read from ch has occured
    case <-timeout:
        // the read from ch has timed out
        break
}
```

第二种形式：取消耗时很长的同步调用

也可以使用 `time.After()` 函数替换 `timeout-channel`。可以在 `select` 中通过 `time.After()` 发送的超时信号来停止协程的执行。以下代码，在 `timeoutNs` 纳秒后执行 `select` 的 `timeout` 分支后，执行`client.Call` 的协程也随之结束，不会给通道 `ch` 返回值：

```go
ch := make(chan error, 1)
go func() { ch <- client.Call("Service.Method", args, &reply) } ()
select {
case resp := <-ch
    // use resp and reply
case <-time.After(timeoutNs):
    // call timed out
    break
}
```

注意缓冲大小设置为 1 是必要的，可以避免协程死锁以及确保超时的通道可以被垃圾回收。此外，需要注意在有多个 `case` 符合条件时， `select` 对 `case` 的选择是伪随机的，如果上面的代码稍作修改如下，则 `select` 语句可能不会在定时器超时信号到来时立刻选中 `time.After(timeoutNs)` 对应的 `case`，因此协程可能不会严格按照定时器设置的时间结束。

```go
ch := make(chan int, 1)
go func() { for { ch <- 1 } } ()
L:
for {
    select {
    case <-ch:
        // do something
    case <-time.After(timeoutNs):
        // call timed out
        break L
    }
}
```

第三种形式：假设程序从多个复制的数据库同时读取。只需要一个答案，需要接收首先到达的答案，`Query` 函数获取数据库的连接切片并请求。并行请求每一个数据库并返回收到的第一个响应：

```go
func Query(conns []Conn, query string) Result {
    ch := make(chan Result, 1)
    for _, conn := range conns {
        go func(c Conn) {
            select {
            case ch <- c.DoQuery(query):
            default:
            }
        }(conn)
    }
    return <- ch
}
```

再次声明，结果通道 `ch` 必须是带缓冲的：以保证第一个发送进来的数据有地方可以存放，确保放入的首个数据总会成功，所以第一个到达的值会被获取而与执行的顺序无关。正在执行的协程可以总是可以使用 `runtime.Goexit()` 来停止。


在应用中缓存数据：

应用程序中用到了来自数据库（或者常见的数据存储）的数据时，经常会把数据缓存到内存中，因为从数据库中获取数据的操作代价很高；如果数据库中的值不发生变化就没有问题。但是如果值有变化，我们需要一个机制来周期性的从数据库重新读取这些值：缓存的值就不可用（过期）了，而且我们也不希望用户看到陈旧的数据。

## 14.6 协程和恢复（recover）

一个用到 `recover` 的程序（参见第 13.3 节）停掉了服务器内部一个失败的协程而不影响其他协程的工作。

```go
func server(workChan <-chan *Work) {
    for work := range workChan {
        go safelyDo(work)   // start the goroutine for that work
    }
}

func safelyDo(work *Work) {
    defer func() {
        if err := recover(); err != nil {
            log.Printf("Work failed with %s in %v", err, work)
        }
    }()
    do(work)
}
```

上边的代码，如果 `do(work)` 发生 panic，错误会被记录且协程会退出并释放，而其他协程不受影响。

因为 `recover` 总是返回 `nil`，除非直接在 `defer` 修饰的函数中调用，`defer` 修饰的代码可以调用那些自身可以使用 `panic` 和 `recover` 避免失败的库例程（库函数）。举例，`safelyDo()` 中 `defer` 修饰的函数可能在调用 `recover` 之前就调用了一个 `logging` 函数，`panicking` 状态不会影响 `logging` 代码的运行。因为加入了恢复模式，函数 `do`（以及它调用的任何东西）可以通过调用 `panic` 来摆脱不好的情况。但是恢复是在 `panicking` 的协程内部的：不能被另外一个协程恢复。

## 14.7 新旧模型对比：任务和worker

假设我们需要处理很多任务；一个worker处理一项任务。任务可以被定义为一个结构体（具体的细节在这里并不重要）：

```go
type Task struct {
    // some state
}
```

旧模式：使用共享内存进行同步

由各个任务组成的任务池共享内存；为了同步各个worker以及避免资源竞争，我们需要对任务池进行加锁保护：

```go
    type Pool struct {
        Mu      sync.Mutex
        Tasks   []*Task
    }
```
sync.Mutex([参见9.3](09.3.md)是互斥锁：它用来在代码中保护临界区资源：同一时间只有一个go协程（goroutine）可以进入该临界区。如果出现了同一时间多个go协程都进入了该临界区，则会产生竞争：Pool结构就不能保证被正确更新。在传统的模式中（经典的面向对象的语言中应用得比较多，比如C++,JAVA,C#)，worker代码可能这样写：

```go
func Worker(pool *Pool) {
    for {
        pool.Mu.Lock()
        // begin critical section:
        task := pool.Tasks[0]        // take the first task
        pool.Tasks = pool.Tasks[1:]  // update the pool of tasks
        // end critical section
        pool.Mu.Unlock()
        process(task)
    }
}
```

这些worker有许多都可以并发执行；他们可以在go协程中启动。一个worker先将pool锁定，从pool获取第一项任务，再解锁和处理任务。加锁保证了同一时间只有一个go协程可以进入到pool中：一项任务有且只能被赋予一个worker。如果不加锁，则工作协程可能会在`task:=pool.Tasks[0]`发生切换，导致`pool.Tasks=pool.Tasks[1:]`结果异常：一些worker获取不到任务，而一些任务可能被多个worker得到。加锁实现同步的方式在工作协程比较少时可以工作的很好，但是当工作协程数量很大，任务量也很多时，处理效率将会因为频繁的加锁/解锁开销而降低。当工作协程数增加到一个阈值时，程序效率会急剧下降，这就成为了瓶颈。

新模式：使用通道

使用通道进行同步：使用一个通道接受需要处理的任务，一个通道接受处理完成的任务（及其结果）。worker在协程中启动，其数量N应该根据任务数量进行调整。

主线程扮演着Master节点角色，可能写成如下形式：

```go
    func main() {
        pending, done := make(chan *Task), make(chan *Task)
        go sendWork(pending)       // put tasks with work on the channel
        for i := 0; i < N; i++ {   // start N goroutines to do work
            go Worker(pending, done)
        }
        consumeWork(done)          // continue with the processed tasks
    }
```

worker的逻辑比较简单：从pending通道拿任务，处理后将其放到done通道中：

```go
    func Worker(in, out chan *Task) {
        for {
            t := <-in
            process(t)
            out <- t
        }
    }
```

这里并不使用锁：从通道得到新任务的过程没有任何竞争。随着任务数量增加，worker数量也应该相应增加，同时性能并不会像第一种方式那样下降明显。在pending通道中存在一份任务的拷贝，第一个worker从pending通道中获得第一个任务并进行处理，这里并不存在竞争（对一个通道读数据和写数据的整个过程是原子性的：参见[14.2.2](14.2.md))。某一个任务会在哪一个worker中被执行是不可知的，反过来也是。worker数量的增多也会增加通信的开销，这会对性能有轻微的影响。

从这个简单的例子中可能很难看出第二种模式的优势，但含有复杂锁运用的程序不仅在编写上显得困难，也不容易编写正确，使用第二种模式的话，就无需考虑这么复杂的东西了。

因此，第二种模式对比第一种模式而言，不仅性能是一个主要优势，而且还有个更大的优势：代码显得更清晰、更优雅。一个更符合go语言习惯的worker写法：

**IDIOM: Use an in- and out-channel instead of locking**

```go
    func Worker(in, out chan *Task) {
        for {
            t := <-in
            process(t)
            out <- t
        }
    }
```

对于任何可以建模为Master-Worker范例的问题，一个类似于worker使用通道进行通信和交互、Master进行整体协调的方案都能完美解决。如果系统部署在多台机器上，各个机器上执行Worker协程，Master和Worker之间使用netchan或者RPC进行通信（参见15章）。

怎么选择是该使用锁还是通道？

通道是一个较新的概念，本节我们着重强调了在go协程里通道的使用，但这并不意味着经典的锁方法就不能使用。go语言让你可以根据实际问题进行选择：创建一个优雅、简单、可读性强、在大多数场景性能表现都能很好的方案。如果你的问题适合使用锁，也不要忌讳使用它。go语言注重实用，什么方式最能解决你的问题就用什么方式，而不是强迫你使用一种编码风格。下面列出一个普遍的经验法则：

* 使用锁的情景：
    - 访问共享数据结构中的缓存信息
    - 保存应用程序上下文和状态信息数据
  
* 使用通道的情景：
    - 与异步操作的结果进行交互
    - 分发任务
    - 传递数据所有权

当你发现你的锁使用规则变得很复杂时，可以反省使用通道会不会使问题变得简单些。

## 14.8 惰性生成器的实现

生成器是指当被调用时返回一个序列中下一个值的函数，例如：

```go
    generateInteger() => 0
    generateInteger() => 1
    generateInteger() => 2
    ....
```

生成器每次返回的是序列中下一个值而非整个序列；这种特性也称之为惰性求值：只在你需要时进行求值，同时保留相关变量资源（内存和cpu）：这是一项在需要时对表达式进行求值的技术。例如，生成一个无限数量的偶数序列：要产生这样一个序列并且在一个一个的使用可能会很困难，而且内存会溢出！但是一个含有通道和go协程的函数能轻易实现这个需求。

在14.12的例子中，我们实现了一个使用 int 型通道来实现的生成器。通道被命名为`yield`和`resume`，这些词经常在协程代码中使用。

示例 14.12 [lazy_evaluation.go](examples/chapter_14/lazy_evaluation.go)：

```go
package main

import (
    "fmt"
)

var resume chan int

func integers() chan int {
    yield := make(chan int)
    count := 0
    go func() {
        for {
            yield <- count
            count++
        }
    }()
    return yield
}

func generateInteger() int {
    return <-resume
}

func main() {
    resume = integers()
    fmt.Println(generateInteger())  //=> 0
    fmt.Println(generateInteger())  //=> 1
    fmt.Println(generateInteger())  //=> 2    
}
```

有一个细微的区别是从通道读取的值可能会是稍早前产生的，并不是在程序被调用时生成的。如果确实需要这样的行为，就得实现一个请求响应机制。当生成器生成数据的过程是计算密集型且各个结果的顺序并不重要时，那么就可以将生成器放入到go协程实现并行化。但是得小心，使用大量的go协程的开销可能会超过带来的性能增益。

这些原则可以概括为：通过巧妙地使用空接口、闭包和高阶函数，我们能实现一个通用的惰性生产器的工厂函数`BuildLazyEvaluator`（这个应该放在一个工具包中实现）。工厂函数需要一个函数和一个初始状态作为输入参数，返回一个无参、返回值是生成序列的函数。传入的函数需要计算出下一个返回值以及下一个状态参数。在工厂函数中，创建一个通道和无限循环的go协程。返回值被放到了该通道中，返回函数稍后被调用时从该通道中取得该返回值。每当取得一个值时，下一个值即被计算。在下面的例子中，定义了一个`evenFunc`函数，其是一个惰性生成函数：在main函数中，我们创建了前10个偶数，每个都是通过调用`even()`函数取得下一个值的。为此，我们需要在`BuildLazyIntEvaluator`函数中具体化我们的生成函数，然后我们能够基于此做出定义。

示例 14.13 [general_lazy_evalution1.go](examples/chapter_14/general_lazy_evalution1.go)：

```go
package main

import (
    "fmt"
)

type Any interface{}
type EvalFunc func(Any) (Any, Any)

func main() {
    evenFunc := func(state Any) (Any, Any) {
        os := state.(int)
        ns := os + 2
        return os, ns
    }
    
    even := BuildLazyIntEvaluator(evenFunc, 0)
    
    for i := 0; i < 10; i++ {
        fmt.Printf("%vth even: %v\n", i, even())
    }
}

func BuildLazyEvaluator(evalFunc EvalFunc, initState Any) func() Any {
    retValChan := make(chan Any)
    loopFunc := func() {
        var actState Any = initState
        var retVal Any
        for {
            retVal, actState = evalFunc(actState)
            retValChan <- retVal
        }
    }
    retFunc := func() Any {
        return <- retValChan
    }
    go loopFunc()
    return retFunc
}

func BuildLazyIntEvaluator(evalFunc EvalFunc, initState Any) func() int {
    ef := BuildLazyEvaluator(evalFunc, initState)
    return func() int {
        return ef().(int)
    }
}
```

输出：
```go
0th even: 0
1th even: 2
2th even: 4
3th even: 6
4th even: 8
5th even: 10
6th even: 12
7th even: 14
8th even: 16
9th even: 18
```

练习14.12：[general_lazy_evaluation2.go](exercises/chapter_14/general_lazy_evalution2.go)
通过使用14.12中工厂函数生成前10个斐波那契数

提示：因为斐波那契数增长很迅速，使用`uint64`类型。
注：这种计算通常被定义为递归函数，但是在没有尾递归的语言中，例如go语言，这可能会导致栈溢出，但随着go语言中堆栈可扩展的优化，这个问题就不那么严重。这里的诀窍是使用了惰性求值。gccgo编译器在某些情况下会实现尾递归。

## 14.9 实现 Futures 模式

所谓Futures就是指：有时候在你使用某一个值之前需要先对其进行计算。这种情况下，你就可以在另一个处理器上进行该值的计算，到使用时，该值就已经计算完毕了。

Futures模式通过闭包和通道可以很容易实现，类似于生成器，不同地方在于Futures需要返回一个值。

参考条目文献给出了一个很精彩的例子：假设我们有一个矩阵类型，我们需要计算两个矩阵A和B乘积的逆，首先我们通过函数`Inverse(M)`分别对其进行求逆运算，再将结果相乘。如下函数`InverseProduct()`实现了如上过程：

```go
func InverseProduct(a Matrix, b Matrix) {
    a_inv := Inverse(a)
    b_inv := Inverse(b)
    return Product(a_inv, b_inv)
}
```

在这个例子中，a和b的求逆矩阵需要先被计算。那么为什么在计算b的逆矩阵时，需要等待a的逆计算完成呢？显然不必要，这两个求逆运算其实可以并行执行的。换句话说，调用`Product`函数只需要等到`a_inv`和`b_inv`的计算完成。如下代码实现了并行计算方式：

```go
func InverseProduct(a Matrix, b Matrix) {
    a_inv_future := InverseFuture(a)   // start as a goroutine
    b_inv_future := InverseFuture(b)   // start as a goroutine
    a_inv := <-a_inv_future
    b_inv := <-b_inv_future
    return Product(a_inv, b_inv)
}
```

`InverseFuture`函数以`goroutine`的形式起了一个闭包，该闭包会将矩阵求逆结果放入到future通道中：

```go
func InverseFuture(a Matrix) chan Matrix {
    future := make(chan Matrix)
    go func() {
        future <- Inverse(a)
    }()
    return future
}
```

当开发一个计算密集型库时，使用Futures模式设计API接口是很有意义的。在你的包使用Futures模式，且能保持友好的API接口。此外，Futures可以通过一个异步的API暴露出来。这样你可以以最小的成本将包中的并行计算移到用户代码中。（参见参考文件18：[http://www.golangpatterns.info/concurrency/futures](http://www.golangpatterns.info/concurrency/futures)）


## 14.10 复用

### 14.10.1 典型的客户端/服务器（C/S）模式

客户端-服务器应用正是 goroutines 和 channels 的亮点所在。

客户端(Client)可以是运行在任意设备上的任意程序，它会按需发送请求(request)至服务器。服务器(Server)接收到这个请求后开始相应的工作，然后再将响应(response)返回给客户端。典型情况下一般是多个客户端（即多个请求）对应一个（或少量）服务器。例如我们日常使用的浏览器客户端，其功能就是向服务器请求网页。而Web服务器则会向浏览器响应网页数据。

使用Go的服务器通常会在协程中执行向客户端的响应，故而会对每一个客户端请求启动一个协程。一个常用的操作方法是客户端请求自身中包含一个通道，而服务器则向这个通道发送响应。

例如下面这个`Request`结构，其中内嵌了一个`replyc`通道。
```go
type Request struct {
    a, b      int    
    replyc    chan int // reply channel inside the Request
}
```
或者更通俗的：
```go
type Reply struct{...}
type Request struct{
    arg1, arg2, arg3 some_type
    replyc chan *Reply
}
```


接下来先使用简单的形式,服务器会为每一个请求启动一个协程并在其中执行`run()`函数，此举会将类型为`binOp`的`op`操作返回的int值发送到`replyc`通道。


```go
type binOp func(a, b int) int

func run(op binOp, req *Request) {
    req.replyc <- op(req.a, req.b)
}
```
`server`协程会无限循环以从`chan *Request`接收请求，并且为了避免被长时间操作所堵塞，它将为每一个请求启动一个协程来做具体的工作：

```go
func server(op binOp, service chan *Request) {
    for {
        req := <-service; // requests arrive here  
        // start goroutine for request:        
        go run(op, req);  // don’t wait for op to complete    
    }
}
```
`server`本身则是以协程的方式在`startServer`函数中启动：
```go
func startServer(op binOp) chan *Request {
    reqChan := make(chan *Request);
    go server(op, reqChan);
    return reqChan;
}
```
`startServer`则会在`main`协程中被调用。

在以下测试例子中，100个请求会被发送到服务器，只有它们全部被送达后我们才会按相反的顺序检查响应：
```go
func main() {
    adder := startServer(func(a, b int) int { return a + b })
    const N = 100
    var reqs [N]Request
    for i := 0; i < N; i++ {
        req := &reqs[i]
        req.a = i
        req.b = i + N
        req.replyc = make(chan int)
        adder <- req  // adder is a channel of requests
    }
    // checks:
    for i := N - 1; i >= 0; i-- {
        // doesn’t matter what order
        if <-reqs[i].replyc != N+2*i {
            fmt.Println(“fail at”, i)
        } else {
            fmt.Println(“Request “, i, “is ok!”)
        }
    }
    fmt.Println(“done”)
}
```
这些代码可以在[multiplex_server.go](examples/chapter_14/multiplex_server.go)找到

输出：

    Request 99 is ok!
    Request 98 is ok!
    ...
    Request 1 is ok!
    Request 0 is ok!
    done

 





这个程序仅启动了100个协程。然而即使执行100,000个协程我们也能在数秒内看到它完成。这说明了Go的协程是如何的轻量：如果我们启动相同数量的真实的线程，程序早就崩溃了。

示例： [14.14-multiplex_server.go](examples/chapter_14/multiplex_server.go)
```go
package main

import "fmt"

type Request struct {
	a, b   int
	replyc chan int // reply channel inside the Request
}

type binOp func(a, b int) int

func run(op binOp, req *Request) {
	req.replyc <- op(req.a, req.b)
}

func server(op binOp, service chan *Request) {
	for {
		req := <-service // requests arrive here
		// start goroutine for request:
		go run(op, req) // don't wait for op
	}
}

func startServer(op binOp) chan *Request {
	reqChan := make(chan *Request)
	go server(op, reqChan)
	return reqChan
}

func main() {
	adder := startServer(func(a, b int) int { return a + b })
	const N = 100
	var reqs [N]Request
	for i := 0; i < N; i++ {
		req := &reqs[i]
		req.a = i
		req.b = i + N
		req.replyc = make(chan int)
		adder <- req
	}
	// checks:
	for i := N - 1; i >= 0; i-- { // doesn't matter what order
		if <-reqs[i].replyc != N+2*i {
			fmt.Println("fail at", i)
		} else {
			fmt.Println("Request ", i, " is ok!")
		}
	}
	fmt.Println("done")
}

```
### 14.10.2 卸载（Teardown）：通过信号通道关闭服务器

在上一个版本中`server`在`main`函数返回后并没有完全关闭，而被强制结束了。为了改进这一点，我们可以提供一个退出通道给`server`：

```go
func startServer(op binOp) (service chan *Request, quit chan bool) {
    service = make(chan *Request)
    quit = make(chan bool)
    go server(op, service, quit)
    return service, quit
}
```

`server`函数现在则使用`select`在`service`通道和`quit`通道之间做出选择：

```go
func server(op binOp, service chan *request, quit chan bool) {
    for {
        select {
            case req := <-service:
                go run(op, req) 
            case <-quit:
                return   
        }
    }
}
```
当`quit`通道接收到一个`true`值时，`server`就会返回并结束。

在`main`函数中我们做出如下更改：

    adder, quit := startServer(func(a, b int) int { return a + b })

在`main`函数的结尾处我们放入这一行：`quit <- true`

完整的代码在 multiplex_server2.go,输出和上一个版本是一样的。

示例： [14.15-multiplex_server2.go](examples/chapter_14/multiplex_server2.go)
```go
package main

import "fmt"

type Request struct {
	a, b   int
	replyc chan int // reply channel inside the Request
}

type binOp func(a, b int) int

func run(op binOp, req *Request) {
	req.replyc <- op(req.a, req.b)
}

func server(op binOp, service chan *Request, quit chan bool) {
	for {
		select {
		case req := <-service:
			go run(op, req)
		case <-quit:
			return
		}
	}
}

func startServer(op binOp) (service chan *Request, quit chan bool) {
	service = make(chan *Request)
	quit = make(chan bool)
	go server(op, service, quit)
	return service, quit
}

func main() {
	adder, quit := startServer(func(a, b int) int { return a + b })
	const N = 100
	var reqs [N]Request
	for i := 0; i < N; i++ {
		req := &reqs[i]
		req.a = i
		req.b = i + N
		req.replyc = make(chan int)
		adder <- req
	}
	// checks:
	for i := N - 1; i >= 0; i-- { // doesn't matter what order
		if <-reqs[i].replyc != N+2*i {
			fmt.Println("fail at", i)
		} else {
			fmt.Println("Request ", i, " is ok!")
		}
	}
	quit <- true
	fmt.Println("done")
}
```
练习 [14.13 multiplex_server3.go](exercises/chapter_14/multiplex_server3.go):使用之前的例子，编写一个在`Request`结构上带有`String()`方法的版本，它能决定服务器如何输出；并使用以下两个请求来测试这个程序：
```go    
    req1 := &Request{3, 4, make(chan int)}
    req2 := &Request{150, 250, make(chan int)}
    ...
    // show the output
    fmt.Println(req1,"\n",req2)
```

## 14.11 限制同时处理的请求数

使用带缓冲区的通道很容易实现这一点（参见 [14.2.5](14.2.md#1425-%E5%90%8C%E6%AD%A5%E9%80%9A%E9%81%93-%E4%BD%BF%E7%94%A8%E5%B8%A6%E7%BC%93%E5%86%B2%E7%9A%84%E9%80%9A%E9%81%93)），其缓冲区容量就是同时处理请求的最大数量。程序[max_tasks.go](examples/chapter_14/max_tasks.go)虽然没有做什么有用的事但是却包含了这个技巧：超过`MAXREQS`的请求将不会被同时处理，因为当信号通道表示缓冲区已满时`handle`函数会阻塞且不再处理其他请求，直到某个请求从`sem`中被移除。`sem`就像一个信号量，这一专业术语用于在程序中表示特定条件的标志变量。

示例：[14.16-max_tasks.go](examples/chapter_14/max_tasks.go)
```go
package main

const MAXREQS = 50

var sem = make(chan int, MAXREQS)

type Request struct {
	a, b   int
	replyc chan int
}

func process(r *Request) {
	// do something
}

func handle(r *Request) {
	sem <- 1 // doesn't matter what we put in it
	process(r)
	<-sem // one empty place in the buffer: the next request can start
}

func server(service chan *Request) {
	for {
		request := <-service
		go handle(request)
	}
}

func main() {
	service := make(chan *Request)
	go server(service)
}
```
通过这种方式，应用程序可以通过使用缓冲通道（通道被用作信号量）使协程同步其对该资源的使用，从而充分利用有限的资源（如内存）。

## 14.12 链式协程

下面的演示程序 [chaining.go](examples/chapter_14/chaining.go) 再次展示了启动巨量的Go协程是多么容易。这些协程已全部在 main 函数中的 for
循环里启动。当循环完成之后，一个0被写入到最右边的通道里，于是100,000个协程开始执行，接着`1000000`这个结果会在1.5秒之内被打印出来。


这个程序同时也展示了如何通过`flag.Int`来解析命令行中的参数以指定协程数量，例如：`chaining -n=7000`会生成7000个协程。


[示例 14.17 - chaining.go](examples/chapter_14/chaining.go)
```go
package main

import (
	"flag"
	"fmt"
)

var ngoroutine = flag.Int("n", 100000, "how many goroutines")

func f(left, right chan int) { left <- 1 + <-right }

func main() {
	flag.Parse()
	leftmost := make(chan int)
	var left, right chan int = nil, leftmost
	for i := 0; i < *ngoroutine; i++ {
		left, right = right, make(chan int)
		go f(left, right)
	}
	right <- 0      // bang!
	x := <-leftmost // wait for completion
	fmt.Println(x)  // 100000, ongeveer 1,5 s
}
```
*译者注：原本认为leftmost的结果为1，认为只在最初做了一次赋值，实际结果为100000（无缓存信道具有同步阻塞的特性）*

*1.主线程的right <- 0，right不是最初循环的那个right，而是最终循环的right*

*2.for循环中最初的go f(left, right)因为没有发送者一直处于等待状态*

*3.当主线程的right <- 0执行时，类似于递归函数在最内层产生返回值一般*

## 14.13 在多核心上并行计算

假设我们有`NCPU`个CPU核心：`const  NCPU = 4 //对应一个四核处理器` 然后我们想把计算量分成`NCPU`个部分，每一个部分都和其他部分并行运行。

这可以通过以下代码所示的方式完成（我们且省略具体参数）

```go
func DoAll(){
    sem := make(chan int, NCPU) // Buffering optional but sensible
    for i := 0; i < NCPU; i++ {
        go DoPart(sem)
    }
    // Drain the channel sem, waiting for NCPU tasks to complete
    for i := 0; i < NCPU; i++ {
        <-sem // wait for one task to complete
    }
    // All done.
}

func DoPart(sem chan int) {
    // do the part of the computation
    sem <-1 // signal that this piece is done
}

func main() {
    runtime.GOMAXPROCS(NCPU) // runtime.GOMAXPROCS = NCPU
    DoAll()
}
    
```
- `DoAll()`函数创建了一个`sem`通道，每个并行计算都将在对其发送完成信号；在一个 for 循环中`NCPU`个协程被启动了，每个协程会承担`1/NCPU`的工作量。每一个`DoPart()`协程都会向`sem`通道发送完成信号。

- `DoAll()`会在 for 循环中等待`NCPU`个协程完成：`sem`通道就像一个信号量，这份代码展示了一个经典的信号量模式。（参见 [14.2.7](14.2.md#1427-%E4%BF%A1%E5%8F%B7%E9%87%8F%E6%A8%A1%E5%BC%8F)）

在以上运行模型中，您还需将`GOMAXPROCS`设置为`NCPU`（参见 [14.1.3](14.1.md#1413-%E4%BD%BF%E7%94%A8-gomaxprocs)）。

## 14.14 并行化大量数据的计算

假设我们需要处理一些数量巨大且互不相关的数据项，它们从一个`in`通道被传递进来，当我们处理完以后又要将它们放入另一个`out`通道，就像一个工厂流水线一样。处理每个数据项也可能包含许多步骤：Preprocess（预处理） / StepA（步骤A） / StepB（步骤B） / ... / PostProcess（后处理）

一个典型的用于解决按顺序执行每个步骤的顺序流水线算法可以写成下面这样：
```go 
func SerialProcessData(in <-chan *Data, out chan<- *Data) {
    for data := range in {
        tmpA := PreprocessData(data)
        tmpB := ProcessStepA(tmpA)
        tmpC := ProcessStepB(tmpB)
        out <- PostProcessData(tmpC)
    }
}
```

一次只执行一个步骤，并且按顺序处理每个项目：在第1个项目没有被`PostProcess`并放入`out`通道之前绝不会处理第2个项目。

如果你仔细想想，你很快就会发现这将会造成巨大的时间浪费。

一个更高效的计算方式是让每一个处理步骤作为一个协程独立工作。每一个步骤从上一步的输出通道中获得输入数据。这种方式仅有极少数时间会被浪费，而大部分时间所有的步骤都在一直执行中：
```go
func ParallelProcessData (in <-chan *Data, out chan<- *Data) {
    // make channels:
    preOut := make(chan *Data, 100)
    stepAOut := make(chan *Data, 100)
    stepBOut := make(chan *Data, 100)
    stepCOut := make(chan *Data, 100)
    // start parallel computations:
    go PreprocessData(in, preOut)
    go ProcessStepA(preOut,StepAOut)
    go ProcessStepB(StepAOut,StepBOut)
    go ProcessStepC(StepBOut,StepCOut)
    go PostProcessData(StepCOut,out)
}   
```
通道的缓冲区大小可以用来进一步优化整个过程。

## 14.15 漏桶算法

（译者注：翻译遵照原文，但是对于完全没听过这个算法的人来说比较晦涩，请配合代码片段理解）

考虑以下的客户端-服务器结构：客户端协程执行一个无限循环从某个源头（也许是网络）接收数据；数据读取到`Buffer`类型的缓冲区。为了避免分配过多的缓冲区以及释放缓冲区，它保留了一份空闲缓冲区列表，并且使用一个缓冲通道来表示这个列表：`var freeList = make(chan *Buffer,100)`

这个可重用的缓冲区队列（freeList）与服务器是共享的。 当接收数据时，客户端尝试从`freeList`获取缓冲区; 但如果此时通道为空，则会分配新的缓冲区。 一旦消息被加载后，它将被发送到服务器上的`serverChan`通道：

    var serverChan = make(chan *Buffer)

以下是客户端的算法代码：

```go
 func client() {
    for {
        var b *Buffer
        // Grab a buffer if available; allocate if not 
        select {
            case b = <-freeList:
                // Got one; nothing more to do
            default:
                // None free, so allocate a new one
                b = new(Buffer)
        }
        loadInto(b)         // Read next message from the network
        serverChan <- b     // Send to server
        
    }
 }
 
```
服务器的循环则接收每一条来自客户端的消息并处理它，之后尝试将缓冲返回给共享的空闲缓冲区：
```go
func server() {
    for {
        b := <-serverChan       // Wait for work.
        process(b)
        // Reuse buffer if there's room.
        select {
            case freeList <- b:
                // Reuse buffer if free slot on freeList; nothing more to do
            default:
                // Free list full, just carry on: the buffer is 'dropped'
        }
    }
}
```
但是这种方法在`freeList`通道已满的时候是行不通的，因为无法放入空闲`freeList`通道的缓冲区会被“丢到地上”由垃圾收集器回收（故名：漏桶算法）

## 14.16 对Go协程进行基准测试

在 [13.7 节](13.7.md) 我们提到了在Go语言中对你的函数进行基准测试。在此我们将其应用到一个用协程向通道写入整数再读出的实例中。这个函数将通过`testing.Benchmark`调用`N`次（例如：`N = 1,000,000`），`BenchMarkResult`有一个`String()`方法来输出其结果。`N`的值将由`gotest`来判断并取得一个足够大的数字，以获得合理的基准测试结果。当然同样的基准测试方法也适用于普通函数。

如果你想排除指定部分的代码或者更具体的指定要测试的部分，可以使用`testing.B.startTimer()`和`testing.B.stopTimer()`来开始或结束计时器。基准测试只有在所有的测试通过后才能运行！ 

示例：[14.18-benchmark_channels.go](examples/chapter_14/benchmark_channels.go)

```go
package main

import (
	"fmt"
	"testing"
)

func main() {
	fmt.Println(" sync", testing.Benchmark(BenchmarkChannelSync).String())
	fmt.Println("buffered", testing.Benchmark(BenchmarkChannelBuffered).String())
}

func BenchmarkChannelSync(b *testing.B) {
	ch := make(chan int)
	go func() {
		for i := 0; i < b.N; i++ {
			ch <- i
		}
		close(ch)
	}()
	for range ch {
	}
}

func BenchmarkChannelBuffered(b *testing.B) {
	ch := make(chan int, 128)
	go func() {
		for i := 0; i < b.N; i++ {
			ch <- i
		}
		close(ch)
	}()
	for range ch {
	}
}
```
输出：
```
  Output:Windows:  N       Time 1 op   Operations per sec
  sync      1000000  2443 ns/op  -->  409 332 / s
  buffered   1000000  4850 ns/op  -->  810 477 / s
  Linux:
```

## 14.17 使用通道并发访问对象

为了保护对象被并发访问修改，我们可以使用协程在后台顺序执行匿名函数来替代使用同步互斥锁。在下面的程序中我们有一个类型`Person`其中包含一个字段`chF`，这是一个用于存放匿名函数的通道。

这个结构在构造函数`NewPerson()`中初始化的同时会启动一个后台协程`backend()`。`backend()`方法会在一个无限循环中执行`chF`中放置的所有函数，有效的将它们序列化从而提供了安全的并发访问。更改和读取`salary`的方法会通过将一个匿名函数写入`chF`通道中，然后让`backend()`按顺序执行以达到其目的。需注意的是`Salary`方法创建的闭包函数是如何将`fChan`通道包含在其中的。

当然，这是一个简化的例子，它不应该被用在这种案例下。但是它却向我们展示了在更复杂的场景中该如何解决这种问题。

示例：[14.19-conc_access.go](examples/chapter_14/conc_access.go)
```go
package main

import (
	"fmt"
	"strconv"
)

type Person struct {
	Name   string
	salary float64
	chF    chan func()
}

func NewPerson(name string, salary float64) *Person {
	p := &Person{name, salary, make(chan func())}
	go p.backend()
	return p
}

func (p *Person) backend() {
	for f := range p.chF {
		f()
	}
}

// Set salary.
func (p *Person) SetSalary(sal float64) {
	p.chF <- func() { p.salary = sal }
}

// Retrieve salary.
func (p *Person) Salary() float64 {
	fChan := make(chan float64)
	p.chF <- func() { fChan <- p.salary }
	return <-fChan
}

func (p *Person) String() string {
	return "Person - name is: " + p.Name + " - salary is: " + strconv.FormatFloat(p.Salary(), 'f', 2, 64)
}

func main() {
	bs := NewPerson("Smith Bill", 2500.5)
	fmt.Println(bs)
	bs.SetSalary(4000.25)
	fmt.Println("Salary changed:")
	fmt.Println(bs)
}
```
输出：
```
Person - name is: Smith Bill - salary is: 2500.50
Salary changed:
Person - name is: Smith Bill - salary is: 4000.25
```