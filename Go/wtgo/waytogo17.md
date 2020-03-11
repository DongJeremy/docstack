# 第17章：Go 语言模式

## 17.1 逗号 ok 模式

在学习本书第二部分和第三部分时，我们经常在一个表达式返回2个参数时使用这种模式：`,ok`，第一个参数是一个值或者`nil`，第二个参数是`true`/`false`或者一个错误`error`。在一个需要赋值的`if`条件语句中，使用这种模式去检测第二个参数值会让代码显得优雅简洁。这种模式在go语言编码规范中非常重要。下面总结了所有使用这种模式的例子：

（1）在函数返回时检测错误（参考[第5.2小节](05.2.md)）:

```go
value, err := pack1.Func1(param1)

if err != nil {
    fmt.Printf("Error %s in pack1.Func1 with parameter %v", err.Error(), param1)
    return err
}

// 函数Func1没有错误:
Process(value)

e.g.: os.Open(file) strconv.Atoi(str)
```

这段代码中的函数将错误返回给它的调用者，当函数执行成功时，返回的错误是`nil`，所以使用这种写法：

```go
func SomeFunc() error {
    …
    if value, err := pack1.Func1(param1); err != nil {
        …
        return err
    }
    …
    return nil
}
```

这种模式也常用于通过`defer`使程序从`panic`中恢复执行（参考[第17.2（4）小节](17.2.md)）。

要实现简洁的错误检测代码，更好的方式是使用闭包，参考[第16.10.2小节](16.10.md)

（2）检测映射中是否存在一个键值（参考[第8.2小节](08.2.md)）：`key1`在映射`map1`中是否有值？

```go
if value, isPresent = map1[key1]; isPresent {
        Process(value)
}
// key1不存在
…
```

（3）检测一个接口类型变量`varI`是否包含了类型`T`：类型断言（参考[第11.3小节](11.3.md)）：

```go
if value, ok := varI.(T); ok {
    Process(value)
}
// 接口类型varI没有包含类型T
```

（4）检测一个通道`ch`是否关闭（参考[第14.3小节](14.3.md)）：

```go
    for input := range ch {
        Process(input)
    }
```

或者:

```go
    for {
        if input, open := <-ch; !open {
            break // 通道是关闭的
        }
        Process(input)
    }
```

## 17.2 defer 模式

使用 `defer` 可以确保资源不再需要时，都会被恰当地关闭或归还到“池子”中。更重要的一点是，它可以恢复 panic。

1. 关闭一个文件流：（见 [12.7节](12.7.md)）
```go
// 先打开一个文件 f
defer f.Close()
```

2. 解锁一个被锁定的资源（`mutex`）：（见 [9.3节](09.3.md)）
```go
mu.Lock()
defer mu.Unlock()
```

3. 关闭一个通道（如有必要）：
```
ch := make(chan float64)
defer close(ch)
```

也可以是两个通道：
```go
answerα, answerβ := make(chan int), make(chan int)
defer func() { close(answerα); close(answerβ) }()
```

4. 从 panic 恢复：（见 [13.3节](13.3.md)）
```go
defer func() {
	if err := recover(); err != nil {
		log.Printf("run time panic: %v", err)
	}
}()
```

5. 停止一个计时器：（见 [14.5节](14.5.md)）
```go
tick1 := time.NewTicker(updateInterval)
defer tick1.Stop()
```

6. 释放一个进程 p：（见 [13.6节](13.6.md)）
```go
p, err := os.StartProcess(…, …, …)
defer p.Release()
```

7. 停止 CPU 性能分析并立即写入：（见 [13.10节](13.10.md)）
```go
pprof.StartCPUProfile(f)
defer pprof.StopCPUProfile()
```

当然 `defer` 也可以在打印报表时避免忘记输出页脚。

## 17.3 可见性模式

我们在 [4.2.1节](04.2.md) 见过简单地使用可见性规则控制对类型成员的访问，他们可以是 Go 变量或函数。[10.2.1节](10.2.md) 展示了如何在单独的包中定义类型时，强制使用工厂函数。

## 17.4 运算符模式和接口

运算符是一元或二元函数，它返回一个新对象而不修改其参数，类似 C++ 中的 `+` 和 `*`，特殊的中缀运算符（`+`，`-`，`*` 等）可以被重载以支持类似数学运算的语法。但除了一些特殊情况，Go 语言并不支持运算符重载：为了克服该限制，运算符必须由函数来模拟。既然 Go 同时支持面向过程和面向对象编程，我们有两种选择：

### 17.4.1 函数作为运算符

运算符由包级别的函数实现，以操作一个或两个参数，并返回一个新对象。函数针对要操作的对象，在专门的包中实现。例如，假设要在包 `matrix` 中实现矩阵操作，就会包含 `Add()` 用于矩阵相加，`Mult()` 用于矩阵相乘，他们都会返回一个矩阵。这两个函数通过包名来调用，因此可以创造出如下形式的表达式：
```go
m := matrix.Add(m1, matrix.Mult(m2, m3))
```

如果我们想在这些运算中区分不同类型的矩阵（稀疏或稠密），由于没有函数重载，我们不得不给函数起不同的名称，例如：
```go
func addSparseToDense (a *sparseMatrix, b *denseMatrix) *denseMatrix
func addDenseToDense (a *denseMatrix, b *denseMatrix) *denseMatrix
func addSparseToSparse (a *sparseMatrix, b *sparseMatrix) *sparseMatrix
```

这可不怎么优雅，我们能选择的最佳方案是将它们隐藏起来，作为包的私有函数，并暴露单一的 `Add()` 函数作为公共 API。可以在嵌套的 `switch` 断言中测试类型，以便在任何支持的参数组合上执行操作：
```go
func Add(a Matrix, b Matrix) Matrix {
	switch a.(type) {
	case sparseMatrix:
		switch b.(type) {
		case sparseMatrix:
			return addSparseToSparse(a.(sparseMatrix), b.(sparseMatrix))
		case denseMatrix:
			return addSparseToDense(a.(sparseMatrix), b.(denseMatrix))
		…
		}
	default:
		// 不支持的参数
		…
	}
}
```

然而，更优雅和优选的方案是将运算符作为方法实现，标准库中到处都运用了这种做法。有关 Ryanne Dolan 实现的线性代数包的更详细信息，可以在 https://github.com/skelterjohn/go.matrix 找到。

### 17.4.2 方法作为运算符

根据接收者类型不同，可以区分不同的方法。因此我们可以为每种类型简单地定义 `Add` 方法，来代替使用多个函数名称：
```go
func (a *sparseMatrix) Add(b Matrix) Matrix
func (a *denseMatrix) Add(b Matrix) Matrix
```

每个方法都返回一个新对象，成为下一个方法调用的接收者，因此我们可以使用*链式调用*表达式：
```go
m := m1.Mult(m2).Add(m3)
```
比上一节面向过程的形式更简洁。

正确的实现同样可以基于类型，通过 `switch` 类型断言在运行时确定：
```go
func (a *sparseMatrix) Add(b Matrix) Matrix {
	switch b.(type) {
	case sparseMatrix:
		return addSparseToSparse(a.(sparseMatrix), b.(sparseMatrix))
	case denseMatrix:
		return addSparseToDense(a.(sparseMatrix), b.(denseMatrix))
	…
	default:
		// 不支持的参数
		…
	}
}
```

再次地，这比上一节嵌套的 `switch` 更简单。

### 17.4.3 使用接口

当在不同类型上执行相同的方法时，创建一个通用化的接口以实现多态的想法，就会自然产生。

例如定义一个代数 `Algebraic` 接口：
```go
type Algebraic interface {
	Add(b Algebraic) Algebraic
	Min(b Algebraic) Algebraic
	Mult(b Algebraic) Algebraic
	…
	Elements()
}
```

然后为我们的 `matrix` 类型定义 `Add()`，`Min()`，`Mult()`，……等方法。

每种实现上述 `Algebraic` 接口类型的方法都可以链式调用。每个方法实现都应基于参数类型，使用 `switch` 类型断言来提供优化过的实现。另外，应该为仅依赖于接口的方法，指定一个默认处理分支：
```go
func (a *denseMatrix) Add(b Algebraic) Algebraic {
	switch b.(type) {
	case sparseMatrix:
		return addDenseToSparse(a, b.(sparseMatrix))
	…
	default:
		for x in range b.Elements() …
	}
}
```

如果一个通用的功能无法仅使用接口方法来实现，你可能正在处理两个不怎么相似的类型，此时应该放弃这种运算符模式。例如，如果 `a` 是一个集合而 `b` 是一个矩阵，那么编写 `a.Add(b)` 没有意义。就集合和矩阵运算而言，很难实现一个通用的 `a.Add(b)` 方法。遇到这种情况，把包拆分成两个，然后提供单独的 `AlgebraicSet` 和 `AlgebraicMatrix` 接口。
