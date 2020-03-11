# 第九章　基于共享变量的并发

前一章我们介绍了一些使用`goroutine`和`channel`这样直接而自然的方式来实现并发的方法。然而这样做我们实际上回避了在写并发代码时必须处理的一些重要而且细微的问题。

在本章中，我们会细致地了解并发机制。尤其是在多`goroutine`之间的共享变量，并发问题的分析手段，以及解决这些问题的基本模式。最后我们会解释`goroutine`和操作系统线程之间的技术上的一些区别。
## 9.1. 竞争条件

在一个线性（就是说只有一个`goroutine`的）的程序中，程序的执行顺序只由程序的逻辑来决定。例如，我们有一段语句序列，第一个在第二个之前（废话），以此类推。在有两个或更多`goroutine`的程序中，每一个`goroutine`内的语句也是按照既定的顺序去执行的，但是一般情况下我们没法去知道分别位于两个`goroutine`的事件`x`和`y`的执行顺序，`x`是在`y`之前还是之后还是同时发生是没法判断的。当我们没有办法自信地确认一个事件是在另一个事件的前面或者后面发生的话，就说明`x`和`y`这两个事件是并发的。

考虑一下，一个函数在线性程序中可以正确地工作。如果在并发的情况下，这个函数依然可以正确地工作的话，那么我们就说这个函数是并发安全的，并发安全的函数不需要额外的同步工作。我们可以把这个概念概括为一个特定类型的一些方法和操作函数，对于某个类型来说，如果其所有可访问的方法和操作都是并发安全的话，那么该类型便是并发安全的。

在一个程序中有非并发安全的类型的情况下，我们依然可以使这个程序并发安全。确实，并发安全的类型是例外，而不是规则，所以只有当文档中明确地说明了其是并发安全的情况下，你才可以并发地去访问它。我们会避免并发访问大多数的类型，无论是将变量局限在单一的一个`goroutine`内，还是用互斥条件维持更高级别的不变性，都是为了这个目的。我们会在本章中说明这些术语。

相反，包级别的导出函数一般情况下都是并发安全的。由于`package`级的变量没法被限制在单一的`gorouine`，所以修改这些变量“必须”使用互斥条件。

一个函数在并发调用时没法工作的原因太多了，比如死锁（`deadlock`）、活锁（`livelock`）和饿死（`resource starvation`）。我们没有空去讨论所有的问题，这里我们只聚焦在竞争条件上。

竞争条件指的是程序在多个`goroutine`交叉执行操作时，没有给出正确的结果。竞争条件是很恶劣的一种场景，因为这种问题会一直潜伏在你的程序里，然后在非常少见的时候蹦出来，或许只是会在很大的负载时才会发生，又或许是会在使用了某一个编译器、某一种平台或者某一种架构的时候才会出现。这些使得竞争条件带来的问题非常难以复现而且难以分析诊断。

传统上经常用经济损失来为竞争条件做比喻，所以我们来看一个简单的银行账户程序。

```go
// Package bank implements a bank with only one account.
package bank
var balance int
func Deposit(amount int) { balance = balance + amount }
func Balance() int { return balance }
```

(当然我们也可以把`Deposit`存款函数写成`balance += amoun`t，这种形式也是等价的，不过长一些的形式解释起来更方便一些。)

对于这个简单的程序而言，我们一眼就能看出，以任意顺序调用函数`Deposit`和`Balance`都会得到正确的结果。也就是说，`Balance`函数会给出之前的所有存入的额度之和。然而，当我们并发地而不是顺序地调用这些函数的话，`Balance`就再也没办法保证结果正确了。考虑一下下面的两个`goroutine`，其代表了一个银行联合账户的两笔交易：

```go
// Alice:
go func() {
	bank.Deposit(200)                // A1
	fmt.Println("=", bank.Balance()) // A2
}()

// Bob:
go bank.Deposit(100)                 // B
```

Alice存了`$200`，然后检查她的余额，同时Bob存了`$100`。因为A1和A2是和B并发执行的，我们没法预测他们发生的先后顺序。直观地来看的话，我们会认为其执行顺序只有三种可能性：“`Alice先`”，“`Bob先`”以及“`Alice/Bob/Alice`”交错执行。下面的表格会展示经过每一步骤后`balance`变量的值。引号里的字符串表示余额单。

```go
Alice first        Bob first        Alice/Bob/Alice
          0                0                      0
  A1    200        B     100             A1     200
  A2 "= 200"       A1    300             B      300
  B     300        A2 "= 300"            A2  "= 300"
```

所有情况下最终的余额都是`$300`。唯一的变数是`Alice`的余额单是否包含了`Bob`交易，不过无论怎么着客户都不会在意。

但是事实是上面的直觉推断是错误的。第四种可能的结果是事实存在的，这种情况下Bob的存款会在Alice存款操作中间，在余额被读到（`balance + amount`）之后，在余额被更新之前（`balance = ...`），这样会导致Bob的交易丢失。而这是因为Alice的存款操作A1实际上是两个操作的一个序列，读取然后写；可以称之为A1r和A1w。下面是交叉时产生的问题：

```go
Data race
0
A1r      0     ... = balance + amount
B      100
A1w    200     balance = ...
A2  "= 200"
```

在A1r之后，`balance + amount`会被计算为200，所以这是A1w会写入的值，并不受其它存款操作的干预。最终的余额是`$200`。银行的账户上的资产比Bob实际的资产多了`$100`。（译注：因为丢失了Bob的存款操作，所以其实是说Bob的钱丢了。）

这个程序包含了一个特定的竞争条件，叫作数据竞争。无论任何时候，只要有两个goroutine并发访问同一变量，且至少其中的一个是写操作的时候就会发生数据竞争。

如果数据竞争的对象是一个比一个机器字（译注：32位机器上一个字=4个字节）更大的类型时，事情就变得更麻烦了，比如interface，string或者slice类型都是如此。下面的代码会并发地更新两个不同长度的slice：

```go
var x []int
go func() { x = make([]int, 10) }()
go func() { x = make([]int, 1000000) }()
x[999999] = 1 // NOTE: undefined behavior; memory corruption possible!
```

最后一个语句中的x的值是未定义的；其可能是nil，或者也可能是一个长度为10的slice，也可能是一个长度为`1,000,000`的slice。但是回忆一下slice的三个组成部分：指针（pointer）、长度（length）和容量（capacity）。如果指针是从第一个make调用来，而长度从第二个make来，x就变成了一个混合体，一个自称长度为1,000,000但实际上内部只有10个元素的slice。这样导致的结果是存储999,999元素的位置会碰撞一个遥远的内存位置，这种情况下难以对值进行预测，而且debug也会变成噩梦。这种语义雷区被称为未定义行为，对C程序员来说应该很熟悉；幸运的是在Go语言里造成的麻烦要比C里小得多。

尽管并发程序的概念让我们知道并发并不是简单的语句交叉执行。我们将会在9.4节中看到，数据竞争可能会有奇怪的结果。许多程序员，甚至一些非常聪明的人也还是会偶尔提出一些理由来允许数据竞争，比如：“互斥条件代价太高”，“这个逻辑只是用来做logging”，“我不介意丢失一些消息”等等。因为在他们的编译器或者平台上很少遇到问题，可能给了他们错误的信心。一个好的经验法则是根本就没有什么所谓的良性数据竞争。所以我们一定要避免数据竞争，那么在我们的程序中要如何做到呢？

我们来重复一下数据竞争的定义，因为实在太重要了：数据竞争会在两个以上的goroutine并发访问相同的变量且至少其中一个为写操作时发生。根据上述定义，有三种方式可以避免数据竞争：

第一种方法是不要去写变量。考虑一下下面的map，会被“懒”填充，也就是说在每个key被第一次请求到的时候才会去填值。如果Icon是被顺序调用的话，这个程序会工作很正常，但如果Icon被并发调用，那么对于这个map来说就会存在数据竞争。

```go
var icons = make(map[string]image.Image)
func loadIcon(name string) image.Image

// NOTE: not concurrency-safe!
func Icon(name string) image.Image {
	icon, ok := icons[name]
	if !ok {
		icon = loadIcon(name)
		icons[name] = icon
	}
	return icon
}
```

反之，如果我们在创建goroutine之前的初始化阶段，就初始化了map中的所有条目并且再也不去修改它们，那么任意数量的goroutine并发访问Icon都是安全的，因为每一个goroutine都只是去读取而已。

```go
var icons = map[string]image.Image{
	"spades.png":   loadIcon("spades.png"),
	"hearts.png":   loadIcon("hearts.png"),
	"diamonds.png": loadIcon("diamonds.png"),
	"clubs.png":    loadIcon("clubs.png"),
}

// Concurrency-safe.
func Icon(name string) image.Image { return icons[name] }
```

上面的例子里icons变量在包初始化阶段就已经被赋值了，包的初始化是在程序main函数开始执行之前就完成了的。只要初始化完成了，icons就再也不会被修改。数据结构如果从不被修改或是不变量则是并发安全的，无需进行同步。不过显然，如果update操作是必要的，我们就没法用这种方法，比如说银行账户。

第二种避免数据竞争的方法是，避免从多个goroutine访问变量。这也是前一章中大多数程序所采用的方法。例如前面的并发web爬虫（§8.6）的main goroutine是唯一一个能够访问seen map的goroutine，而聊天服务器（§8.10）中的broadcaster goroutine是唯一一个能够访问clients map的goroutine。这些变量都被限定在了一个单独的goroutine中。

由于其它的goroutine不能够直接访问变量，它们只能使用一个channel来发送请求给指定的goroutine来查询更新变量。这也就是Go的口头禅“不要使用共享数据来通信；使用通信来共享数据”。一个提供对一个指定的变量通过channel来请求的goroutine叫做这个变量的monitor（监控）goroutine。例如broadcaster goroutine会监控clients map的全部访问。

下面是一个重写了的银行的例子，这个例子中balance变量被限制在了monitor goroutine中，名为teller：

<u><i>gopl.io/ch9/bank1</i></u>
```go
// Package bank provides a concurrency-safe bank with one account.
package bank

var deposits = make(chan int) // send amount to deposit
var balances = make(chan int) // receive balance

func Deposit(amount int) { deposits <- amount }
func Balance() int       { return <-balances }

func teller() {
	var balance int // balance is confined to teller goroutine
	for {
		select {
		case amount := <-deposits:
			balance += amount
		case balances <- balance:
		}
	}
}

func init() {
	go teller() // start the monitor goroutine
}
```

即使当一个变量无法在其整个生命周期内被绑定到一个独立的goroutine，绑定依然是并发问题的一个解决方案。例如在一条流水线上的goroutine之间共享变量是很普遍的行为，在这两者间会通过channel来传输地址信息。如果流水线的每一个阶段都能够避免在将变量传送到下一阶段后再去访问它，那么对这个变量的所有访问就是线性的。其效果是变量会被绑定到流水线的一个阶段，传送完之后被绑定到下一个，以此类推。这种规则有时被称为串行绑定。

下面的例子中，Cakes会被严格地顺序访问，先是`baker gorouine`，然后是`icer gorouine`：

```go
type Cake struct{ state string }

func baker(cooked chan<- *Cake) {
	for {
		cake := new(Cake)
		cake.state = "cooked"
		cooked <- cake // baker never touches this cake again
	}
}

func icer(iced chan<- *Cake, cooked <-chan *Cake) {
	for cake := range cooked {
		cake.state = "iced"
		iced <- cake // icer never touches this cake again
	}
}
```

第三种避免数据竞争的方法是允许很多goroutine去访问变量，但是在同一个时刻最多只有一个goroutine在访问。这种方式被称为“互斥”，在下一节来讨论这个主题。

**练习 9.1：** 给`gopl.io/ch9/bank1`程序添加一个`Withdraw(amount int)`取款函数。其返回结果应该要表明事务是成功了还是因为没有足够资金失败了。这条消息会被发送给monitor的goroutine，且消息需要包含取款的额度和一个新的channel，这个新channel会被monitor goroutine来把`boolean`结果发回给Withdraw。

```go
// bank.go
package bank

type Withdrawal struct {
	amount  int
	success chan bool
}

var deposits = make(chan int) // send amount to deposit
var balances = make(chan int) // receive balance
var withdrawals = make(chan Withdrawal)

func Deposit(amount int) { deposits <- amount }
func Balance() int       { return <-balances }
func Withdraw(amount int) bool {
	ch := make(chan bool)
	withdrawals <- Withdrawal{amount, ch}
	return <-ch
}

func teller() {
	var balance int // balance is confined to teller goroutine
	for {
		select {
		case amount := <-deposits:
			balance += amount
		case w := <-withdrawals:
			if w.amount > balance {
				w.success <- false
				continue
			}
			balance -= w.amount
			w.success <- true
		case balances <- balance:
		}
	}
}

func init() {
	go teller() // start the monitor goroutine
}
```

```go
// bank_test.go
package bank

import (
	"fmt"
	"testing"
)

func TestBank(t *testing.T) {
	done := make(chan struct{})

	// Alice
	go func() {
		Deposit(200)
		Withdraw(200)
		fmt.Println("=", Balance())
		done <- struct{}{}
	}()

	// Bob
	go func() {
		Deposit(50)
		Withdraw(50)
		Deposit(100)
		done <- struct{}{}
	}()

	// Wait for both transactions.
	<-done
	<-done

	if got, want := Balance(), 100; got != want {
		t.Errorf("Balance = %d, want %d", got, want)
	}
}

func TestWithdrawal(t *testing.T) {
	b1 := Balance()
	ok := Withdraw(50)
	if !ok {
		t.Errorf("ok = false, want true. balance = %d", Balance())
	}
	expected := b1 - 50
	if b2 := Balance(); b2 != expected {
		t.Errorf("balance = %d, want %d", b2, expected)
	}
}

func TestWithdrawalFailsIfInsufficientFunds(t *testing.T) {
	b1 := Balance()
	ok := Withdraw(b1 + 1)
	b2 := Balance()
	if ok {
		t.Errorf("ok = true, want false. balance = %d", b2)
	}
	if b2 != b1 {
		t.Errorf("balance = %d, want %d", b2, b1)
	}
}
```

## 9.2. sync.Mutex互斥锁

在8.6节中，我们使用了一个buffered channel作为一个计数信号量，来保证最多只有20个goroutine会同时执行HTTP请求。同理，我们可以用一个容量只有1的channel来保证最多只有一个goroutine在同一时刻访问一个共享变量。一个只能为1和0的信号量叫做二元信号量（binary semaphore）。

<u><i>gopl.io/ch9/bank2</i></u>
```go
var (
	sema    = make(chan struct{}, 1) // a binary semaphore guarding balance
	balance int
)

func Deposit(amount int) {
	sema <- struct{}{} // acquire token
	balance = balance + amount
	<-sema // release token
}

func Balance() int {
	sema <- struct{}{} // acquire token
	b := balance
	<-sema // release token
	return b
}
```

这种互斥很实用，而且被sync包里的Mutex类型直接支持。它的Lock方法能够获取到token(这里叫锁)，并且Unlock方法会释放这个token：

<u><i>gopl.io/ch9/bank3</i></u>
```go
import "sync"

var (
	mu      sync.Mutex // guards balance
	balance int
)

func Deposit(amount int) {
	mu.Lock()
	balance = balance + amount
	mu.Unlock()
}

func Balance() int {
	mu.Lock()
	b := balance
	mu.Unlock()
	return b
}
```

每次一个goroutine访问bank变量时（这里只有balance余额变量），它都会调用mutex的Lock方法来获取一个互斥锁。如果其它的goroutine已经获得了这个锁的话，这个操作会被阻塞直到其它goroutine调用了Unlock使该锁变回可用状态。mutex会保护共享变量。惯例来说，被mutex所保护的变量是在mutex变量声明之后立刻声明的。如果你的做法和惯例不符，确保在文档里对你的做法进行说明。

在Lock和Unlock之间的代码段中的内容goroutine可以随便读取或者修改，这个代码段叫做临界区。锁的持有者在其他goroutine获取该锁之前需要调用Unlock。goroutine在结束后释放锁是必要的，无论以哪条路径通过函数都需要释放，即使是在错误路径中，也要记得释放。

上面的bank程序例证了一种通用的并发模式。一系列的导出函数封装了一个或多个变量，那么访问这些变量唯一的方式就是通过这些函数来做（或者方法，对于一个对象的变量来说）。每一个函数在一开始就获取互斥锁并在最后释放锁，从而保证共享变量不会被并发访问。这种函数、互斥锁和变量的编排叫作监控monitor（这种老式单词的monitor是受“monitor goroutine”的术语启发而来的。两种用法都是一个代理人保证变量被顺序访问）。

由于在存款和查询余额函数中的临界区代码这么短——只有一行，没有分支调用——在代码最后去调用Unlock就显得更为直截了当。在更复杂的临界区的应用中，尤其是必须要尽早处理错误并返回的情况下，就很难去（靠人）判断对Lock和Unlock的调用是在所有路径中都能够严格配对的了。Go语言里的defer简直就是这种情况下的救星：我们用defer来调用Unlock，临界区会隐式地延伸到函数作用域的最后，这样我们就从“总要记得在函数返回之后或者发生错误返回时要记得调用一次Unlock”这种状态中获得了解放。Go会自动帮我们完成这些事情。

```go
func Balance() int {
	mu.Lock()
	defer mu.Unlock()
	return balance
}
```

上面的例子里Unlock会在return语句读取完balance的值之后执行，所以Balance函数是并发安全的。这带来的另一点好处是，我们再也不需要一个本地变量b了。

此外，一个deferred Unlock即使在临界区发生panic时依然会执行，这对于用recover（§5.10）来恢复的程序来说是很重要的。defer调用只会比显式地调用Unlock成本高那么一点点，不过却在很大程度上保证了代码的整洁性。大多数情况下对于并发程序来说，代码的整洁性比过度的优化更重要。如果可能的话尽量使用defer来将临界区扩展到函数的结束。

考虑一下下面的Withdraw函数。成功的时候，它会正确地减掉余额并返回true。但如果银行记录资金对交易来说不足，那么取款就会恢复余额，并返回false。

```go
// NOTE: not atomic!
func Withdraw(amount int) bool {
	Deposit(-amount)
	if Balance() < 0 {
		Deposit(amount)
		return false // insufficient funds
	}
	return true
}
```

函数终于给出了正确的结果，但是还有一点讨厌的副作用。当过多的取款操作同时执行时，balance可能会瞬时被减到0以下。这可能会引起一个并发的取款被不合逻辑地拒绝。所以如果Bob尝试买一辆sports car时，Alice可能就没办法为她的早咖啡付款了。这里的问题是取款不是一个原子操作：它包含了三个步骤，每一步都需要去获取并释放互斥锁，但任何一次锁都不会锁上整个取款流程。

理想情况下，取款应该只在整个操作中获得一次互斥锁。下面这样的尝试是错误的：

```go
// NOTE: incorrect!
func Withdraw(amount int) bool {
	mu.Lock()
	defer mu.Unlock()
	Deposit(-amount)
	if Balance() < 0 {
		Deposit(amount)
		return false // insufficient funds
	}
	return true
}
```

上面这个例子中，Deposit会调用mu.Lock()第二次去获取互斥锁，但因为mutex已经锁上了，而无法被重入（译注：go里没有重入锁，关于重入锁的概念，请参考java）——也就是说没法对一个已经锁上的mutex来再次上锁——这会导致程序死锁，没法继续执行下去，Withdraw会永远阻塞下去。

关于Go的mutex不能重入这一点我们有很充分的理由。mutex的目的是确保共享变量在程序执行时的关键点上能够保证不变性。不变性的一层含义是“没有goroutine访问共享变量”，但实际上这里对于mutex保护的变量来说，不变性还包含更深层含义：当一个goroutine获得了一个互斥锁时，它能断定被互斥锁保护的变量正处于不变状态（译注：即没有其他代码块正在读写共享变量），在其获取并保持锁期间，可能会去更新共享变量，这样不变性只是短暂地被破坏，然而当其释放锁之后，锁必须保证共享变量重获不变性并且多个goroutine按顺序访问共享变量。尽管一个可以重入的mutex也可以保证没有其它的goroutine在访问共享变量，但它不具备不变性更深层含义。（译注：[更详细的解释](https://stackoverflow.com/questions/14670979/recursive-locking-in-go/14671462#14671462)，Russ Cox认为可重入锁是bug的温床，是一个失败的设计）

一个通用的解决方案是将一个函数分离为多个函数，比如我们把Deposit分离成两个：一个不导出的函数deposit，这个函数假设锁总是会被保持并去做实际的操作，另一个是导出的函数Deposit，这个函数会调用deposit，但在调用前会先去获取锁。同理我们可以将Withdraw也表示成这种形式：

```go
func Withdraw(amount int) bool {
	mu.Lock()
	defer mu.Unlock()
	deposit(-amount)
	if balance < 0 {
		deposit(amount)
		return false // insufficient funds
	}
	return true
}

func Deposit(amount int) {
	mu.Lock()
	defer mu.Unlock()
	deposit(amount)
}

func Balance() int {
	mu.Lock()
	defer mu.Unlock()
	return balance
}

// This function requires that the lock be held.
func deposit(amount int) { balance += amount }
```

当然，这里的存款deposit函数很小，实际上取款Withdraw函数不需要理会对它的调用，尽管如此，这里的表达还是表明了规则。

封装（§6.6），用限制一个程序中的意外交互的方式，可以使我们获得数据结构的不变性。因为某种原因，封装还帮我们获得了并发的不变性。当你使用mutex时，确保mutex和其保护的变量没有被导出（在go里也就是小写，且不要被大写字母开头的函数访问啦），无论这些变量是包级的变量还是一个struct的字段。
## 9.3. sync.RWMutex读写锁

在100刀的存款消失时不做记录多少还是会让我们有一些恐慌，Bob写了一个程序，每秒运行几百次来检查他的银行余额。他会在家，在工作中，甚至会在他的手机上来运行这个程序。银行注意到这些陡增的流量使得存款和取款有了延时，因为所有的余额查询请求是顺序执行的，这样会互斥地获得锁，并且会暂时阻止其它的goroutine运行。

由于Balance函数只需要读取变量的状态，所以我们同时让多个Balance调用并发运行事实上是安全的，只要在运行的时候没有存款或者取款操作就行。在这种场景下我们需要一种特殊类型的锁，其允许多个只读操作并行执行，但写操作会完全互斥。这种锁叫作“多读单写”锁（multiple readers, single writer lock），Go语言提供的这样的锁是sync.RWMutex：

```go
var mu sync.RWMutex
var balance int
func Balance() int {
	mu.RLock() // readers lock
	defer mu.RUnlock()
	return balance
}
```

Balance函数现在调用了RLock和RUnlock方法来获取和释放一个读取或者共享锁。Deposit函数没有变化，会调用mu.Lock和mu.Unlock方法来获取和释放一个写或互斥锁。

在这次修改后，Bob的余额查询请求就可以彼此并行地执行并且会很快地完成了。锁在更多的时间范围可用，并且存款请求也能够及时地被响应了。

RLock只能在临界区共享变量没有任何写入操作时可用。一般来说，我们不应该假设逻辑上的只读函数/方法也不会去更新某一些变量。比如一个方法功能是访问一个变量，但它也有可能会同时去给一个内部的计数器+1（译注：可能是记录这个方法的访问次数啥的），或者去更新缓存——使即时的调用能够更快。如果有疑惑的话，请使用互斥锁。

RWMutex只有当获得锁的大部分goroutine都是读操作，而锁在竞争条件下，也就是说，goroutine们必须等待才能获取到锁的时候，RWMutex才是最能带来好处的。RWMutex需要更复杂的内部记录，所以会让它比一般的无竞争锁的mutex慢一些。

## 9.4. 内存同步

你可能比较纠结为什么Balance方法需要用到互斥条件，无论是基于channel还是基于互斥量。毕竟和存款不一样，它只由一个简单的操作组成，所以不会碰到其它goroutine在其执行“期间”执行其它逻辑的风险。这里使用mutex有两方面考虑。第一Balance不会在其它操作比如Withdraw“中间”执行。第二（更重要的）是“同步”不仅仅是一堆goroutine执行顺序的问题，同样也会涉及到内存的问题。

在现代计算机中可能会有一堆处理器，每一个都会有其本地缓存（local cache）。为了效率，对内存的写入一般会在每一个处理器中缓冲，并在必要时一起flush到主存。这种情况下这些数据可能会以与当初goroutine写入顺序不同的顺序被提交到主存。像channel通信或者互斥量操作这样的原语会使处理器将其聚集的写入flush并commit，这样goroutine在某个时间点上的执行结果才能被其它处理器上运行的goroutine得到。

考虑一下下面代码片段的可能输出：

```go
var x, y int
go func() {
	x = 1 // A1
	fmt.Print("y:", y, " ") // A2
}()
go func() {
	y = 1                   // B1
	fmt.Print("x:", x, " ") // B2
}()
```

因为两个goroutine是并发执行，并且访问共享变量时也没有互斥，会有数据竞争，所以程序的运行结果没法预测的话也请不要惊讶。我们可能希望它能够打印出下面这四种结果中的一种，相当于几种不同的交错执行时的情况：

```
y:0 x:1
x:0 y:1
x:1 y:1
y:1 x:1
```

第四行可以被解释为执行顺序A1,B1,A2,B2或者B1,A1,A2,B2的执行结果。然而实际运行时还是有些情况让我们有点惊讶：

```
x:0 y:0
y:0 x:0
```

根据所使用的编译器，CPU，或者其它很多影响因子，这两种情况也是有可能发生的。那么这两种情况要怎么解释呢？

在一个独立的goroutine中，每一个语句的执行顺序是可以被保证的，也就是说goroutine内顺序是连贯的。但是在不使用channel且不使用mutex这样的显式同步操作时，我们就没法保证事件在不同的goroutine中看到的执行顺序是一致的了。尽管goroutine A中一定需要观察到x=1执行成功之后才会去读取y，但它没法确保自己观察得到goroutine B中对y的写入，所以A还可能会打印出y的一个旧版的值。

尽管去理解并发的一种尝试是去将其运行理解为不同goroutine语句的交错执行，但看看上面的例子，这已经不是现代的编译器和cpu的工作方式了。因为赋值和打印指向不同的变量，编译器可能会断定两条语句的顺序不会影响执行结果，并且会交换两个语句的执行顺序。如果两个goroutine在不同的CPU上执行，每一个核心有自己的缓存，这样一个goroutine的写入对于其它goroutine的Print，在主存同步之前就是不可见的了。

所有并发的问题都可以用一致的、简单的既定的模式来规避。所以可能的话，将变量限定在goroutine内部；如果是多个goroutine都需要访问的变量，使用互斥条件来访问。

## 9.5. sync.Once惰性初始化

如果初始化成本比较大的话，那么将初始化延迟到需要的时候再去做就是一个比较好的选择。如果在程序启动的时候就去做这类初始化的话，会增加程序的启动时间，并且因为执行的时候可能也并不需要这些变量，所以实际上有一些浪费。让我们来看在本章早一些时候的icons变量：

```go
var icons map[string]image.Image
```

这个版本的Icon用到了懒初始化（lazy initialization）。

```go
func loadIcons() {
	icons = map[string]image.Image{
		"spades.png":   loadIcon("spades.png"),
		"hearts.png":   loadIcon("hearts.png"),
		"diamonds.png": loadIcon("diamonds.png"),
		"clubs.png":	loadIcon("clubs.png"),
	}
}

// NOTE: not concurrency-safe!
func Icon(name string) image.Image {
	if icons == nil {
		loadIcons() // one-time initialization
	}
	return icons[name]
}
```

如果一个变量只被一个单独的goroutine所访问的话，我们可以使用上面的这种模板，但这种模板在Icon被并发调用时并不安全。就像前面银行的那个Deposit(存款)函数一样，Icon函数也是由多个步骤组成的：首先测试icons是否为空，然后load这些icons，之后将icons更新为一个非空的值。直觉会告诉我们最差的情况是loadIcons函数被多次访问会带来数据竞争。当第一个goroutine在忙着loading这些icons的时候，另一个goroutine进入了Icon函数，发现变量是nil，然后也会调用loadIcons函数。

不过这种直觉是错误的。（我们希望你从现在开始能够构建自己对并发的直觉，也就是说对并发的直觉总是不能被信任的！），回忆一下9.4节。因为缺少显式的同步，编译器和CPU是可以随意地去更改访问内存的指令顺序，以任意方式，只要保证每一个goroutine自己的执行顺序一致。其中一种可能loadIcons的语句重排是下面这样。它会在填写icons变量的值之前先用一个空map来初始化icons变量。

```go
func loadIcons() {
	icons = make(map[string]image.Image)
	icons["spades.png"] = loadIcon("spades.png")
	icons["hearts.png"] = loadIcon("hearts.png")
	icons["diamonds.png"] = loadIcon("diamonds.png")
	icons["clubs.png"] = loadIcon("clubs.png")
}
```

因此，一个goroutine在检查icons是非空时，也并不能就假设这个变量的初始化流程已经走完了（译注：可能只是塞了个空map，里面的值还没填完，也就是说填值的语句都没执行完呢）。

最简单且正确的保证所有goroutine能够观察到loadIcons效果的方式，是用一个mutex来同步检查。

```go
var mu sync.Mutex // guards icons
var icons map[string]image.Image

// Concurrency-safe.
func Icon(name string) image.Image {
	mu.Lock()
	defer mu.Unlock()
	if icons == nil {
		loadIcons()
	}
	return icons[name]
}
```

然而使用互斥访问icons的代价就是没有办法对该变量进行并发访问，即使变量已经被初始化完毕且再也不会进行变动。这里我们可以引入一个允许多读的锁：

```go
var mu sync.RWMutex // guards icons
var icons map[string]image.Image
// Concurrency-safe.
func Icon(name string) image.Image {
	mu.RLock()
	if icons != nil {
		icon := icons[name]
		mu.RUnlock()
		return icon
	}
	mu.RUnlock()

	// acquire an exclusive lock
	mu.Lock()
	if icons == nil { // NOTE: must recheck for nil
		loadIcons()
	}
	icon := icons[name]
	mu.Unlock()
	return icon
}
```


上面的代码有两个临界区。goroutine首先会获取一个读锁，查询map，然后释放锁。如果条目被找到了（一般情况下），那么会直接返回。如果没有找到，那goroutine会获取一个写锁。不释放共享锁的话，也没有任何办法来将一个共享锁升级为一个互斥锁，所以我们必须重新检查icons变量是否为nil，以防止在执行这一段代码的时候，icons变量已经被其它gorouine初始化过了。

上面的模板使我们的程序能够更好的并发，但是有一点太复杂且容易出错。幸运的是，sync包为我们提供了一个专门的方案来解决这种一次性初始化的问题：sync.Once。概念上来讲，一次性的初始化需要一个互斥量mutex和一个boolean变量来记录初始化是不是已经完成了；互斥量用来保护boolean变量和客户端数据结构。Do这个唯一的方法需要接收初始化函数作为其参数。让我们用sync.Once来简化前面的Icon函数吧：

```go
var loadIconsOnce sync.Once
var icons map[string]image.Image
// Concurrency-safe.
func Icon(name string) image.Image {
	loadIconsOnce.Do(loadIcons)
	return icons[name]
}
```

每一次对`Do(loadIcons)`的调用都会锁定`mutex`，并会检查`boolean`变量（译注：Go1.9中会先判断boolean变量是否为1(true)，只有不为1才锁定mutex，不再需要每次都锁定mutex）。在第一次调用时，boolean变量的值是false，Do会调用loadIcons并会将boolean变量设置为true。随后的调用什么都不会做，但是mutex同步会保证loadIcons对内存（这里其实就是指icons变量啦）产生的效果能够对所有goroutine可见。用这种方式来使用sync.Once的话，我们能够避免在变量被构建完成之前和其它goroutine共享该变量。

**练习 9.2：** 重写2.6.2节中的`PopCount`的例子，使用`sync.Once`，只在第一次需要用到的时候进行初始化。（虽然实际上，对`PopCount`这样很小且高度优化的函数进行同步可能代价没法接受。）

```go
// popcount_test.go
package popcount

import (
	"sync"
	"testing"
)

func PopCountShiftMask(x uint64) int {
	count := 0
	mask := uint64(1)
	for i := 0; i < 64; i++ {
		if x&mask > 0 {
			count++
		}
		mask <<= 1
	}
	return count
}

func PopCountShiftValue(x uint64) int {
	count := 0
	mask := uint64(1)
	for i := 0; i < 64; i++ {
		if x&mask > 0 {
			count++
		}
		x >>= 1
	}
	return count
}

func PopCountClearRightmost(x uint64) int {
	count := 0
	for x != 0 {
		x &= x - 1
		count++
	}
	return count
}

// pc[i] is the population count of i.
var loadTableOnce sync.Once
var pc [256]byte

func loadTable() {
	for i := range pc {
		pc[i] = pc[i/2] + byte(i&1)
	}
}

func Table() [256]byte {
	loadTableOnce.Do(func() {
		for i := range pc {
			pc[i] = pc[i/2] + byte(i&1)
		}
	})
	return pc
}

// PopCount returns the population count (number of set bits) of x.
func PopCountTable(x uint64) int {
	// Calling Table here makes PopCountTable 4 times slower, from 8ns -> 34ns
	// per call.
	pc := Table()
	return int(pc[byte(x>>(0*8))] +
		pc[byte(x>>(1*8))] +
		pc[byte(x>>(2*8))] +
		pc[byte(x>>(3*8))] +
		pc[byte(x>>(4*8))] +
		pc[byte(x>>(5*8))] +
		pc[byte(x>>(6*8))] +
		pc[byte(x>>(7*8))])
}

func bench(b *testing.B, f func(uint64) int) {
	for i := 0; i < b.N; i++ {
		f(uint64(i))
	}
}

func BenchmarkTable(b *testing.B) {
	bench(b, PopCountTable)
}

func BenchmarkShiftMask(b *testing.B) {
	bench(b, PopCountShiftMask)
}

func BenchmarkShiftValue(b *testing.B) {
	bench(b, PopCountShiftValue)
}

func BenchmarkClearRightmost(b *testing.B) {
	bench(b, PopCountClearRightmost)
}
```

## 9.6. 竞争条件检测

即使我们小心到不能再小心，但在并发程序中犯错还是太容易了。幸运的是，Go的runtime和工具链为我们装备了一个复杂但好用的动态分析工具，竞争检查器（the race detector）。

只要在go build，go run或者go test命令后面加上-race的flag，就会使编译器创建一个你的应用的“修改”版或者一个附带了能够记录所有运行期对共享变量访问工具的test，并且会记录下每一个读或者写共享变量的goroutine的身份信息。另外，修改版的程序会记录下所有的同步事件，比如go语句，channel操作，以及对`(*sync.Mutex).Lock`，`(*sync.WaitGroup).Wait`等等的调用。（完整的同步事件集合是在The Go Memory Model文档中有说明，该文档是和语言文档放在一起的。译注：https://golang.org/ref/mem ）

竞争检查器会检查这些事件，会寻找在哪一个goroutine中出现了这样的case，例如其读或者写了一个共享变量，这个共享变量是被另一个goroutine在没有进行干预同步操作便直接写入的。这种情况也就表明了是对一个共享变量的并发访问，即数据竞争。这个工具会打印一份报告，内容包含变量身份，读取和写入的goroutine中活跃的函数的调用栈。这些信息在定位问题时通常很有用。9.7节中会有一个竞争检查器的实战样例。

竞争检查器会报告所有的已经发生的数据竞争。然而，它只能检测到运行时的竞争条件；并不能证明之后不会发生数据竞争。所以为了使结果尽量正确，请保证你的测试并发地覆盖到了你的包。

由于需要额外的记录，因此构建时加了竞争检测的程序跑起来会慢一些，且需要更大的内存，即使是这样，这些代价对于很多生产环境的工作来说还是可以接受的。对于一些偶发的竞争条件来说，让竞争检查器来干活可以节省无数日夜的debugging。（译注：多少服务端C和C++程序员为此竞折腰。）
## 9.7. 示例: 并发的非阻塞缓存

本节中我们会做一个无阻塞的缓存，这种工具可以帮助我们来解决现实世界中并发程序出现但没有现成的库可以解决的问题。这个问题叫作缓存（`memoizing`）函数（译注：`Memoization`的定义： `memoization` 一词是Donald Michie 根据拉丁语memorandum杜撰的一个词。相应的动词、过去分词、ing形式有memoiz、memoized、memoizing），也就是说，我们需要缓存函数的返回结果，这样在对函数进行调用的时候，我们就只需要一次计算，之后只要返回计算的结果就可以了。我们的解决方案会是并发安全且会避免对整个缓存加锁而导致所有操作都去争一个锁的设计。

我们将使用下面的`httpGetBody`函数作为我们需要缓存的函数的一个样例。这个函数会去进行HTTP GET请求并且获取`http`响应`body`。对这个函数的调用本身开销是比较大的，所以我们尽量避免在不必要的时候反复调用。

```go
func httpGetBody(url string) (interface{}, error) {
	resp, err := http.Get(url)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()
	return ioutil.ReadAll(resp.Body)
}
```

最后一行稍微隐藏了一些细节。`ReadAll`会返回两个结果，一个`[]byte`数组和一个错误，不过这两个对象可以被赋值给`httpGetBody`的返回声明里的`interface{}`和`error`类型，所以我们也就可以这样返回结果并且不需要额外的工作了。我们在`httpGetBody`中选用这种返回类型是为了使其可以与缓存匹配。

下面是我们要设计的cache的第一个“草稿”：

<u><i>gopl.io/ch9/memo1</i></u>
```go
// Package memo provides a concurrency-unsafe
// memoization of a function of type Func.
package memo

// A Memo caches the results of calling a Func.
type Memo struct {
	f     Func
	cache map[string]result
}

// Func is the type of the function to memoize.
type Func func(key string) (interface{}, error)

type result struct {
	value interface{}
	err   error
}

func New(f Func) *Memo {
	return &Memo{f: f, cache: make(map[string]result)}
}

// NOTE: not concurrency-safe!
func (memo *Memo) Get(key string) (interface{}, error) {
	res, ok := memo.cache[key]
	if !ok {
		res.value, res.err = memo.f(key)
		memo.cache[key] = res
	}
	return res.value, res.err
}
```

Memo实例会记录需要缓存的函数f（类型为Func），以及缓存内容（里面是一个string到result映射的map）。每一个result都是简单的函数返回的值对儿——一个值和一个错误值。继续下去我们会展示一些Memo的变种，不过所有的例子都会遵循上面的这些方面。

下面是一个使用Memo的例子。对于流入的URL的每一个元素我们都会调用Get，并打印调用延时以及其返回的数据大小的log：

```go
m := memo.New(httpGetBody)
for url := range incomingURLs() {
	start := time.Now()
	value, err := m.Get(url)
	if err != nil {
		log.Print(err)
	}
	fmt.Printf("%s, %s, %d bytes\n",
	url, time.Since(start), len(value.([]byte)))
}
```

我们可以使用测试包（第11章的主题）来系统地鉴定缓存的效果。从下面的测试输出，我们可以看到URL流包含了一些重复的情况，尽管我们第一次对每一个URL的`(*Memo).Get`的调用都会花上几百毫秒，但第二次就只需要花1毫秒就可以返回完整的数据了。

```go
$ go test -v gopl.io/ch9/memo1
=== RUN   Test
https://golang.org, 175.026418ms, 7537 bytes
https://godoc.org, 172.686825ms, 6878 bytes
https://play.golang.org, 115.762377ms, 5767 bytes
http://gopl.io, 749.887242ms, 2856 bytes
https://golang.org, 721ns, 7537 bytes
https://godoc.org, 152ns, 6878 bytes
https://play.golang.org, 205ns, 5767 bytes
http://gopl.io, 326ns, 2856 bytes
--- PASS: Test (1.21s)
PASS
ok  gopl.io/ch9/memo1   1.257s
```

这个测试是顺序地去做所有的调用的。

由于这种彼此独立的HTTP请求可以很好地并发，我们可以把这个测试改成并发形式。可以使用sync.WaitGroup来等待所有的请求都完成之后再返回。

```go
m := memo.New(httpGetBody)
var n sync.WaitGroup
for url := range incomingURLs() {
	n.Add(1)
	go func(url string) {
		start := time.Now()
		value, err := m.Get(url)
		if err != nil {
			log.Print(err)
		}
		fmt.Printf("%s, %s, %d bytes\n",
		url, time.Since(start), len(value.([]byte)))
		n.Done()
	}(url)
}
n.Wait()
```


这次测试跑起来更快了，然而不幸的是貌似这个测试不是每次都能够正常工作。我们注意到有一些意料之外的cache miss（缓存未命中），或者命中了缓存但却返回了错误的值，或者甚至会直接崩溃。

但更糟糕的是，有时候这个程序还是能正确的运行（译：也就是最让人崩溃的偶发bug），所以我们甚至可能都不会意识到这个程序有bug。但是我们可以使用-race这个flag来运行程序，竞争检测器（§9.6）会打印像下面这样的报告：

```go
$ go test -run=TestConcurrent -race -v gopl.io/ch9/memo1
=== RUN   TestConcurrent
...
WARNING: DATA RACE
Write by goroutine 36:
  runtime.mapassign1()
      ~/go/src/runtime/hashmap.go:411 +0x0
  gopl.io/ch9/memo1.(*Memo).Get()
      ~/gobook2/src/gopl.io/ch9/memo1/memo.go:32 +0x205
  ...
Previous write by goroutine 35:
  runtime.mapassign1()
      ~/go/src/runtime/hashmap.go:411 +0x0
  gopl.io/ch9/memo1.(*Memo).Get()
      ~/gobook2/src/gopl.io/ch9/memo1/memo.go:32 +0x205
...
Found 1 data race(s)
FAIL    gopl.io/ch9/memo1   2.393s
```

`memo.go`的32行出现了两次，说明有两个goroutine在没有同步干预的情况下更新了cache map。这表明Get不是并发安全的，存在数据竞争。

```go
28  func (memo *Memo) Get(key string) (interface{}, error) {
29      res, ok := memo.cache(key)
30      if !ok {
31          res.value, res.err = memo.f(key)
32          memo.cache[key] = res
33      }
34      return res.value, res.err
35  }
```

最简单的使cache并发安全的方式是使用基于监控的同步。只要给Memo加上一个mutex，在Get的一开始获取互斥锁，return的时候释放锁，就可以让cache的操作发生在临界区内了：

<u><i>gopl.io/ch9/memo2</i></u>
```go
type Memo struct {
	f     Func
	mu    sync.Mutex // guards cache
	cache map[string]result
}

// Get is concurrency-safe.
func (memo *Memo) Get(key string) (value interface{}, err error) {
	memo.mu.Lock()
	res, ok := memo.cache[key]
    if !ok {
		res.value, res.err = memo.f(key)
		memo.cache[key] = res
	}
	memo.mu.Unlock()
	return res.value, res.err
}
```

测试依然并发进行，但这回竞争检查器“沉默”了。不幸的是对于Memo的这一点改变使我们完全丧失了并发的性能优点。每次对f的调用期间都会持有锁，Get将本来可以并行运行的I/O操作串行化了。我们本章的目的是完成一个无锁缓存，而不是现在这样的将所有请求串行化的函数的缓存。

下一个Get的实现，调用Get的goroutine会两次获取锁：查找阶段获取一次，如果查找没有返回任何内容，那么进入更新阶段会再次获取。在这两次获取锁的中间阶段，其它goroutine可以随意使用cache。

<u><i>gopl.io/ch9/memo3</i></u>
```go
func (memo *Memo) Get(key string) (value interface{}, err error) {
	memo.mu.Lock()
	res, ok := memo.cache[key]
	memo.mu.Unlock()
	if !ok {
		res.value, res.err = memo.f(key)

		// Between the two critical sections, several goroutines
		// may race to compute f(key) and update the map.
		memo.mu.Lock()
		memo.cache[key] = res
		memo.mu.Unlock()
	}
	return res.value, res.err
}
```

这些修改使性能再次得到了提升，但有一些URL被获取了两次。这种情况在两个以上的goroutine同一时刻调用Get来请求同样的URL时会发生。多个goroutine一起查询cache，发现没有值，然后一起调用f这个慢不拉叽的函数。在得到结果后，也都会去更新map。其中一个获得的结果会覆盖掉另一个的结果。

理想情况下是应该避免掉多余的工作的。而这种“避免”工作一般被称为duplicate suppression（重复抑制/避免）。下面版本的Memo每一个map元素都是指向一个条目的指针。每一个条目包含对函数f调用结果的内容缓存。与之前不同的是这次entry还包含了一个叫ready的channel。在条目的结果被设置之后，这个channel就会被关闭，以向其它goroutine广播（§8.9）去读取该条目内的结果是安全的了。

<u><i>gopl.io/ch9/memo4</i></u>
```go
type entry struct {
	res   result
	ready chan struct{} // closed when res is ready
}

func New(f Func) *Memo {
	return &Memo{f: f, cache: make(map[string]*entry)}
}

type Memo struct {
	f     Func
	mu    sync.Mutex // guards cache
	cache map[string]*entry
}

func (memo *Memo) Get(key string) (value interface{}, err error) {
	memo.mu.Lock()
	e := memo.cache[key]
	if e == nil {
		// This is the first request for this key.
		// This goroutine becomes responsible for computing
		// the value and broadcasting the ready condition.
		e = &entry{ready: make(chan struct{})}
		memo.cache[key] = e
		memo.mu.Unlock()

		e.res.value, e.res.err = memo.f(key)

		close(e.ready) // broadcast ready condition
	} else {
		// This is a repeat request for this key.
		memo.mu.Unlock()

		<-e.ready // wait for ready condition
	}
	return e.res.value, e.res.err
}
```

现在Get函数包括下面这些步骤了：获取互斥锁来保护共享变量cache map，查询map中是否存在指定条目，如果没有找到那么分配空间插入一个新条目，释放互斥锁。如果存在条目的话且其值没有写入完成（也就是有其它的goroutine在调用f这个慢函数）时，goroutine必须等待值ready之后才能读到条目的结果。而想知道是否ready的话，可以直接从ready channel中读取，由于这个读取操作在channel关闭之前一直是阻塞。

如果没有条目的话，需要向map中插入一个没有准备好的条目，当前正在调用的goroutine就需要负责调用慢函数、更新条目以及向其它所有goroutine广播条目已经ready可读的消息了。

条目中的`e.res.value`和`e.res.err`变量是在多个`goroutine`之间共享的。创建条目的`goroutine`同时也会设置条目的值，其它`goroutine`在收到"ready"的广播消息之后立刻会去读取条目的值。尽管会被多个`goroutine`同时访问，但却并不需要互斥锁。ready channel的关闭一定会发生在其它goroutine接收到广播事件之前，因此第一个goroutine对这些变量的写操作是一定发生在这些读操作之前的。不会发生数据竞争。

这样并发、不重复、无阻塞的cache就完成了。

上面这样`Memo`的实现使用了一个互斥量来保护多个`goroutine`调用Get时的共享map变量。不妨把这种设计和前面提到的把map变量限制在一个单独的monitor goroutine的方案做一些对比，后者在调用Get时需要发消息。

`Func`、`result`和`entry`的声明和之前保持一致：

```go
// Func is the type of the function to memoize.
type Func func(key string) (interface{}, error)

// A result is the result of calling a Func.
type result struct {
	value interface{}
	err   error
}

type entry struct {
	res   result
	ready chan struct{} // closed when res is ready
}
```

然而Memo类型现在包含了一个叫做`requests`的`channel`，Get的调用方用这个`channel`来和`monitor` `goroutine`来通信。`requests channel`中的元素类型是request。Get的调用方会把这个结构中的两组key都填充好，实际上用这两个变量来对函数进行缓存的。另一个叫response的channel会被拿来发送响应结果。这个channel只会传回一个单独的值。

<u><i>gopl.io/ch9/memo5</i></u>
```go
// A request is a message requesting that the Func be applied to key.
type request struct {
	key      string
	response chan<- result // the client wants a single result
}

type Memo struct{ requests chan request }
// New returns a memoization of f.  Clients must subsequently call Close.
func New(f Func) *Memo {
	memo := &Memo{requests: make(chan request)}
	go memo.server(f)
	return memo
}

func (memo *Memo) Get(key string) (interface{}, error) {
	response := make(chan result)
	memo.requests <- request{key, response}
	res := <-response
	return res.value, res.err
}

func (memo *Memo) Close() { close(memo.requests) }
```

上面的Get方法，会创建一个`response channel`，把它放进`request`结构中，然后发送给monitor goroutine，然后马上又会接收它。

cache变量被限制在了monitor goroutine ``(*Memo).server`中，下面会看到。monitor会在循环中一直读取请求，直到request channel被Close方法关闭。每一个请求都会去查询cache，如果没有找到条目的话，那么就会创建/插入一个新的条目。

```go
func (memo *Memo) server(f Func) {
	cache := make(map[string]*entry)
	for req := range memo.requests {
		e := cache[req.key]
		if e == nil {
			// This is the first request for this key.
			e = &entry{ready: make(chan struct{})}
			cache[req.key] = e
			go e.call(f, req.key) // call f(key)
		}
		go e.deliver(req.response)
	}
}

func (e *entry) call(f Func, key string) {
	// Evaluate the function.
	e.res.value, e.res.err = f(key)
	// Broadcast the ready condition.
	close(e.ready)
}

func (e *entry) deliver(response chan<- result) {
	// Wait for the ready condition.
	<-e.ready
	// Send the result to the client.
	response <- e.res
}
```

和基于互斥量的版本类似，第一个对某个key的请求需要负责去调用函数f并传入这个key，将结果存在条目里，并关闭ready channel来广播条目的ready消息。使用`(*entry).call`来完成上述工作。

紧接着对同一个key的请求会发现map中已经有了存在的条目，然后会等待结果变为ready，并将结果从response发送给客户端的`goroutine`。上述工作是用`(*entry).deliver`来完成的。对call和deliver方法的调用必须让它们在自己的`goroutine`中进行以确保monitor goroutines不会因此而被阻塞住而没法处理新的请求。

这个例子说明我们无论用上锁，还是通信来建立并发程序都是可行的。

上面的两种方案并不好说特定情境下哪种更好，不过了解他们还是有价值的。有时候从一种方式切换到另一种可以使你的代码更为简洁。（译注：不是说好的`golang`推崇通信并发么。）

**练习 9.3：** 扩展`Func`类型和`(*Memo).Get`方法，支持调用方提供一个可选的`done channel`，使其具备通过该`channel`来取消整个操作的能力（§8.9）。一个被取消了的`Func`的调用结果不应该被缓存。

```go
// memo.go
package memo

import (
	"fmt"
)

// Func is the type of the function to memoize.
type Func func(key string, done <-chan struct{}) (interface{}, error)

// A result is the result of calling a Func.
type result struct {
	value interface{}
	err   error
}

type entry struct {
	res   result
	ready chan struct{} // closed when res is ready
}

// A request is a message requesting that the Func be applied to key.
type request struct {
	key      string
	done     <-chan struct{}
	response chan<- result // the client wants a single result
}

type Memo struct {
	requests, cancels chan request
}

// New returns a memoization of f.  Clients must subsequently call Close.
func New(f Func) *Memo {
	memo := &Memo{make(chan request), make(chan request)}
	go memo.server(f)
	return memo
}

func (memo *Memo) Get(key string, done <-chan struct{}) (interface{}, error) {
	response := make(chan result)
	req := request{key, done, response}
	memo.requests <- req
	fmt.Println("get: waiting for response")
	res := <-response
	fmt.Println("get: checking if cancelled")
	select {
	case <-done:
		fmt.Println("get: queueing cancellation request")
		memo.cancels <- req
	default:
		// Not cancelled. Continue.
	}
	fmt.Println("get: return")
	return res.value, res.err
}

func (memo *Memo) Close() { close(memo.requests) }

func (memo *Memo) server(f Func) {
	cache := make(map[string]*entry)
Loop:
	for {
	Cancel:
		// Process all cancellations before requests.
		// After Get has returned a cancellation for some key, any subsequent
		// requests for that key should return the result of a new call to
		// Func. If select is allowed to choose randomly between processing
		// requests and cancellations it can't be predicted whether a request
		// will be cancelled by a previous cancellation or not without looking
		// at the cancels queue, which client obviously can't do.
		for {
			select {
			case req := <-memo.cancels:
				fmt.Println("server: deleting cancelled entry (early)")
				delete(cache, req.key)
			default:
				break Cancel
			}
		}
		// Wait for requests or cancellations, and break to process all
		// cancellations if there are any.
		select {
		case req := <-memo.cancels:
			fmt.Println("server: deleting cancelled entry")
			delete(cache, req.key)
			continue Loop
		case req := <-memo.requests:
			fmt.Println("server: request")
			e := cache[req.key]
			if e == nil {
				// This is the first request for this key.
				e = &entry{ready: make(chan struct{})}
				cache[req.key] = e
				go e.call(f, req.key, req.done) // call f(key)
			}
			go e.deliver(req.response)
		}
	}
}

func (e *entry) call(f Func, key string, done <-chan struct{}) {
	// Evaluate the function.
	e.res.value, e.res.err = f(key, done)
	fmt.Println("call: returned from f")
	// Broadcast the ready condition.
	close(e.ready)
}

func (e *entry) deliver(response chan<- result) {
	// Wait for the ready condition.
	<-e.ready
	// Send the result to the client.
	response <- e.res
}
```

```go
// memo_test.go
package memo

import (
	"fmt"
	"sync"
	"testing"
)

func TestCancel(t *testing.T) {
	cancelled := fmt.Errorf("cancelled")

	finish := make(chan string)
	f := func(key string, done <-chan struct{}) (interface{}, error) {
		if done == nil {
			res := <-finish
			return res, nil
		} else {
			<-done
			return nil, cancelled
		}
	}

	m := New(Func(f))
	key := "key"

	done := make(chan struct{})
	wg1 := &sync.WaitGroup{}
	wg1.Add(1)
	go func() {
		v, err := m.Get(key, done)
		wg1.Done()
		if v != nil || err != cancelled {
			t.Errorf("got %v, %v; want %v, %v", v, err, nil, cancelled)
		}
	}()
	close(done)
	// Wait for the cancellation to be processed before making the next request
	// for `key`.
	wg1.Wait()

	wg2 := &sync.WaitGroup{}
	wg2.Add(1)
	go func() {
		v, err := m.Get(key, nil)
		s := v.(string)
		if s != "ok" || err != nil {
			t.Errorf("got %v, %v; want %v, %v", s, err, "ok", nil)
		}
		wg2.Done()
	}()
	finish <- "ok"
	wg2.Wait()
}
```

## 9.8. Goroutines和线程

在上一章中我们说`goroutine`和操作系统的线程区别可以先忽略。尽管两者的区别实际上只是一个量的区别，但量变会引起质变的道理同样适用于`goroutine`和线程。现在正是我们来区分开两者的最佳时机。

### 9.8.1. 动态栈

每一个OS线程都有一个固定大小的内存块（一般会是2MB）来做栈，这个栈会用来存储当前正在被调用或挂起（指在调用其它函数时）的函数的内部变量。这个固定大小的栈同时很大又很小。因为`2MB`的栈对于一个小小的`goroutine`来说是很大的内存浪费，比如对于我们用到的，一个只是用来`WaitGroup`之后关闭`channel`的`goroutine`来说。而对于go程序来说，同时创建成百上千个`goroutine`是非常普遍的，如果每一个`goroutine`都需要这么大的栈的话，那这么多的`goroutine`就不太可能了。除去大小的问题之外，固定大小的栈对于更复杂或者更深层次的递归函数调用来说显然是不够的。修改固定的大小可以提升空间的利用率，允许创建更多的线程，并且可以允许更深的递归调用，不过这两者是没法同时兼备的。

相反，一个`goroutine`会以一个很小的栈开始其生命周期，一般只需要`2KB`。一个`goroutine`的栈，和操作系统线程一样，会保存其活跃或挂起的函数调用的本地变量，但是和OS线程不太一样的是，一个`goroutine`的栈大小并不是固定的；栈的大小会根据需要动态地伸缩。而`goroutine`的栈的最大值有1GB，比传统的固定大小的线程栈要大得多，尽管一般情况下，大多`goroutine`都不需要这么大的栈。

**练习 9.4:** 创建一个流水线程序，支持用`channel`连接任意数量的`goroutine`，在跑爆内存之前，可以创建多少流水线阶段？一个变量通过整个流水线需要用多久？（这个练习题翻译不是很确定）

```go
// pipe.go
package main

func pipeline(stages int) (in chan int, out chan int) {
	out = make(chan int)
	first := out
	for i := 0; i < stages; i++ {
		in = out
		out = make(chan int)
		go func(in chan int, out chan int) {
			for v := range in {
				out <- v
			}
			close(out)
		}(in, out)
	}
	return first, out
}
```

```go
// pipe_test.go
package main

import (
	"testing"
)

func BenchmarkPipeline(b *testing.B) {
	in, out := pipeline(100000)
	for i := 0; i < b.N; i++ {
		in <- 1
		<-out
	}
	close(in)
}
```

### 9.8.2. Goroutine调度

OS线程会被操作系统内核调度。每几毫秒，一个硬件计时器会中断处理器，这会调用一个叫作`scheduler`的内核函数。这个函数会挂起当前执行的线程并将它的寄存器内容保存到内存中，检查线程列表并决定下一次哪个线程可以被运行，并从内存中恢复该线程的寄存器信息，然后恢复执行该线程的现场并开始执行线程。因为操作系统线程是被内核所调度，所以从一个线程向另一个“移动”需要完整的上下文切换，也就是说，保存一个用户线程的状态到内存，恢复另一个线程的到寄存器，然后更新调度器的数据结构。这几步操作很慢，因为其局部性很差需要几次内存访问，并且会增加运行的`cpu`周期。

Go的运行时包含了其自己的调度器，这个调度器使用了一些技术手段，比如`m:n`调度，因为其会在`n`个操作系统线程上多工（调度）`m`个`goroutine`。Go调度器的工作和内核的调度是相似的，但是这个调度器只关注单独的Go程序中的`goroutine`（译注：按程序独立）。

和操作系统的线程调度不同的是，Go调度器并不是用一个硬件定时器，而是被Go语言“建筑”本身进行调度的。例如当一个`goroutine`调用了`time.Sleep`，或者被`channel`调用或者`mutex`操作阻塞时，调度器会使其进入休眠并开始执行另一个`goroutine`，直到时机到了再去唤醒第一个`goroutine`。因为这种调度方式不需要进入内核的上下文，所以重新调度一个`goroutine`比调度一个线程代价要低得多。

**练习 9.5: ** 写一个有两个`goroutine`的程序，两个`goroutine`会向两个无`buffer channel`反复地发送`ping-pong`消息。这样的程序每秒可以支持多少次通信？

```go
// pong.go
package main

import (
	"fmt"
	"os"
	"os/signal"
	"time"
)

func main() {
	q := make(chan int)
	var i int64
	start := time.Now()
	go func() {
		q <- 1
		for {
			i++
			q <- <-q
		}
	}()
	go func() {
		for {
			q <- <-q
		}
	}()

	c := make(chan os.Signal, 1)
	signal.Notify(c, os.Interrupt)
	<-c
	fmt.Println(float64(i)/float64(time.Since(start))*1e9, "round trips per second")
}
```

### 9.8.3. GOMAXPROCS

Go的调度器使用了一个叫做GOMAXPROCS的变量来决定会有多少个操作系统的线程同时执行Go的代码。其默认的值是运行机器上的CPU的核心数，所以在一个有8个核心的机器上时，调度器一次会在8个OS线程上去调度GO代码。（GOMAXPROCS是前面说的m:n调度中的n）。在休眠中的或者在通信中被阻塞的goroutine是不需要一个对应的线程来做调度的。在I/O中或系统调用中或调用非Go语言函数时，是需要一个对应的操作系统线程的，但是GOMAXPROCS并不需要将这几种情况计算在内。

你可以用GOMAXPROCS的环境变量来显式地控制这个参数，或者也可以在运行时用runtime.GOMAXPROCS函数来修改它。我们在下面的小程序中会看到GOMAXPROCS的效果，这个程序会无限打印0和1。


```go
for {
	go fmt.Print(0)
	fmt.Print(1)
}

$ GOMAXPROCS=1 go run hacker-cliché.go
111111111111111111110000000000000000000011111...

$ GOMAXPROCS=2 go run hacker-cliché.go
010101010101010101011001100101011010010100110...
```

在第一次执行时，最多同时只能有一个goroutine被执行。初始情况下只有main goroutine被执行，所以会打印很多1。过了一段时间后，GO调度器会将其置为休眠，并唤醒另一个goroutine，这时候就开始打印很多0了，在打印的时候，goroutine是被调度到操作系统线程上的。在第二次执行时，我们使用了两个操作系统线程，所以两个goroutine可以一起被执行，以同样的频率交替打印0和1。我们必须强调的是goroutine的调度是受很多因子影响的，而runtime也是在不断地发展演进的，所以这里的你实际得到的结果可能会因为版本的不同而与我们运行的结果有所不同。

**练习9.6:** 测试一下计算密集型的并发程序（练习8.5那样的）会被GOMAXPROCS怎样影响到。在你的电脑上最佳的值是多少？你的电脑CPU有多少个核心？

```

```



### 9.8.4. Goroutine没有ID号

在大多数支持多线程的操作系统和程序语言中，当前的线程都有一个独特的身份（id），并且这个身份信息可以以一个普通值的形式被很容易地获取到，典型的可以是一个integer或者指针值。这种情况下我们做一个抽象化的thread-local storage（线程本地存储，多线程编程中不希望其它线程访问的内容）就很容易，只需要以线程的id作为key的一个map就可以解决问题，每一个线程以其id就能从中获取到值，且和其它线程互不冲突。

goroutine没有可以被程序员获取到的身份（id）的概念。这一点是设计上故意而为之，由于thread-local storage总是会被滥用。比如说，一个web server是用一种支持tls的语言实现的，而非常普遍的是很多函数会去寻找HTTP请求的信息，这代表它们就是去其存储层（这个存储层有可能是tls）查找的。这就像是那些过分依赖全局变量的程序一样，会导致一种非健康的“距离外行为”，在这种行为下，一个函数的行为可能并不仅由自己的参数所决定，而是由其所运行在的线程所决定。因此，如果线程本身的身份会改变——比如一些worker线程之类的——那么函数的行为就会变得神秘莫测。

Go鼓励更为简单的模式，这种模式下参数（译注：外部显式参数和内部显式参数。tls 中的内容算是"外部"隐式参数）对函数的影响都是显式的。这样不仅使程序变得更易读，而且会让我们自由地向一些给定的函数分配子任务时不用担心其身份信息影响行为。

你现在应该已经明白了写一个Go程序所需要的所有语言特性信息。在后面两章节中，我们会回顾一些之前的实例和工具，支持我们写出更大规模的程序：如何将一个工程组织成一系列的包，如何获取，构建，测试，性能测试，剖析，写文档，并且将这些包分享出去。

