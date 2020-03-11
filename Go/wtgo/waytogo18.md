# 18 出于性能考虑的实用代码片段

# 18.1 字符串

（1）如何修改字符串中的一个字符：

```go
str:="hello"
c:=[]byte(str)
c[0]='c'
s2:= string(c) // s2 == "cello"
```

（2）如何获取字符串的子串：

```go
substr := str[n:m]
```

（3）如何使用`for`或者`for-range`遍历一个字符串：

```go
// gives only the bytes:
for i:=0; i < len(str); i++ {
… = str[i]
}
// gives the Unicode characters:
for ix, ch := range str {
…
}
```

（4）如何获取一个字符串的字节数：`len(str)`

 如何获取一个字符串的字符数：

 最快速：`utf8.RuneCountInString(str)` 

 `len([]rune(str))` 

（5）如何连接字符串：

 最快速：

`with a bytes.Buffer`（参考[章节7.2](07.2.md)）

`Strings.Join()`（参考[章节4.7](04.7.md)）

使用`+=`：

 ```go
 str1 := "Hello " 
 str2 := "World!"
 str1 += str2 //str1 == "Hello World!"
 ```

（6）如何解析命令行参数：使用`os`或者`flag`包

（参考[例12.4](examples/chapter_12/fileinput.go)）

# 18.2 数组和切片

创建：

`arr1 := new([len]type)`

`slice1 := make([]type, len)`

初始化：

`arr1 := [...]type{i1, i2, i3, i4, i5}`

`arrKeyValue := [len]type{i1: val1, i2: val2}`

`var slice1 []type = arr1[start:end]`

（1）如何截断数组或者切片的最后一个元素：

`line = line[:len(line)-1]`

（2）如何使用`for`或者`for-range`遍历一个数组（或者切片）：

```go
for i:=0; i < len(arr); i++ {
… = arr[i]
}
for ix, value := range arr {
…
}
```

（3）如何在一个二维数组或者切片`arr2Dim`中查找一个指定值`V`：

```go
found := false
Found: for row := range arr2Dim {
    for column := range arr2Dim[row] {
        if arr2Dim[row][column] == V{
            found = true
            break Found
        }
    }
}
```

# 18.3 映射

创建：    `map1 := make(map[keytype]valuetype)`

初始化：   `map1 := map[string]int{"one": 1, "two": 2}`

（1）如何使用`for`或者`for-range`遍历一个映射：

```go
for key, value := range map1 {
…
}
```

（2）如何在一个映射中检测键`key1`是否存在：

`val1, isPresent = map1[key1]`

返回值：键`key1`对应的值或者`0`, `true`或者`false`

（3）如何在映射中删除一个键：

`delete(map1, key1)`

# 18.4 结构体

创建：

```go
type struct1 struct {
    field1 type1
    field2 type2
    …
}
ms := new(struct1)
```

初始化：

```go
ms := &struct1{10, 15.5, "Chris"}
```

当结构体的命名以大写字母开头时，该结构体在包外可见。
通常情况下，为每个结构体定义一个构建函数，并推荐使用构建函数初始化结构体（参考[例10.2](examples/chapter_10/person.go)）：

```go    
ms := Newstruct1{10, 15.5, "Chris"}
func Newstruct1(n int, f float32, name string) *struct1 {
    return &struct1{n, f, name} 
}
```

# 18.5 接口

（1）如何检测一个值`v`是否实现了接口`Stringer`：

```go
if v, ok := v.(Stringer); ok {
    fmt.Printf("implements String(): %s\n", v.String())
}
```

（2）如何使用接口实现一个类型分类函数：
    
```go
func classifier(items ...interface{}) {
    for i, x := range items {
        switch x.(type) {
        case bool:
            fmt.Printf("param #%d is a bool\n", i)
        case float64:
            fmt.Printf("param #%d is a float64\n", i)
        case int, int64:
            fmt.Printf("param #%d is an int\n", i)
        case nil:
            fmt.Printf("param #%d is nil\n", i)
        case string:
            fmt.Printf("param #%d is a string\n", i)
        default:
            fmt.Printf("param #%d’s type is unknown\n", i)
        }
    }
}
```

# 18.6 函数

如何使用内建函数`recover`终止`panic`过程（参考[章节13.3](13.3.md)）：
    
```go
func protect(g func()) {
    defer func() {
        log.Println("done")
        // Println executes normally even if there is a panic
        if x := recover(); x != nil {
            log.Printf("run time panic: %v", x)
        }
    }()
    log.Println("start")
    g()
}
```

# 18.7 文件

（1）如何打开一个文件并读取：

```go    
file, err := os.Open("input.dat")
  if err != nil {
    fmt.Printf("An error occurred on opening the inputfile\n" +
      "Does the file exist?\n" +
      "Have you got acces to it?\n")
    return
  }
  defer file.Close()
  iReader := bufio.NewReader(file)
  for {
    str, err := iReader.ReadString('\n')
    if err != nil {
      return // error or EOF
    }
    fmt.Printf("The input was: %s", str)
  }
```

（2）如何通过切片读写文件：
    
```go
func cat(f *file.File) {
  const NBUF = 512
  var buf [NBUF]byte
  for {
    switch nr, er := f.Read(buf[:]); true {
    case nr < 0:
      fmt.Fprintf(os.Stderr, "cat: error reading from %s: %s\n",
        f.String(), er.String())
      os.Exit(1)
    case nr == 0: // EOF
      return
    case nr > 0:
      if nw, ew := file.Stdout.Write(buf[0:nr]); nw != nr {
        fmt.Fprintf(os.Stderr, "cat: error writing from %s: %s\n",
          f.String(), ew.String())
      }
    }
  }
}
```

# 18.8 协程（goroutine）与通道（channel）

出于性能考虑的建议：

实践经验表明，为了使并行运算获得高于串行运算的效率，在协程内部完成的工作量，必须远远高于协程的创建和相互来回通信的开销。

1 出于性能考虑建议使用带缓存的通道：

使用带缓存的通道可以很轻易成倍提高它的吞吐量，某些场景其性能可以提高至10倍甚至更多。通过调整通道的容量，甚至可以尝试着更进一步的优化其性能。

2 限制一个通道的数据数量并将它们封装成一个数组：

如果使用通道传递大量单独的数据，那么通道将变成性能瓶颈。然而，将数据块打包封装成数组，在接收端解压数据时，性能可以提高至10倍。

创建：`ch := make(chan type,buf)`

（1）如何使用`for`或者`for-range`遍历一个通道：

```go
for v := range ch {
    // do something with v
}
```

（2）如何检测一个通道`ch`是否关闭：

```go
//read channel until it closes or error-condition
for {
    if input, open := <-ch; !open {
        break
    }
    fmt.Printf("%s", input)
}
```

或者使用（1）自动检测。

（3）如何通过一个通道让主程序等待直到协程完成：

（信号量模式）：

```go
ch := make(chan int) // Allocate a channel.
// Start something in a goroutine; when it completes, signal on the channel.
go func() {
    // doSomething
    ch <- 1 // Send a signal; value does not matter.
}()
doSomethingElseForAWhile()
<-ch // Wait for goroutine to finish; discard sent value.
```

如果希望程序一直阻塞，在匿名函数中省略 `ch <- 1`即可。

（4）通道的工厂模板：以下函数是一个通道工厂，启动一个匿名函数作为协程以生产通道：

```go
func pump() chan int {
    ch := make(chan int)
    go func() {
        for i := 0; ; i++ {
            ch <- i
        }
    }()
    return ch
}
```

（5）通道迭代器模板：

（6）如何限制并发处理请求的数量：参考[章节14.11](14.11.md)

（7）如何在多核CPU上实现并行计算：参考[章节14.13](14.13.md)

（8）如何终止一个协程：`runtime.Goexit()`  

（9）简单的超时模板：

```go 

timeout := make(chan bool, 1)
go func() {
    time.Sleep(1e9) // one second  
    timeout <- true
}()
select {
    case <-ch:
    // a read from ch has occurred
    case <-timeout:
    // the read from ch has timed out
}
```

（10）如何使用输入通道和输出通道代替锁：

```go
func Worker(in, out chan *Task) {
    for {
        t := <-in
        process(t)
        out <- t
    }
}
```

（11）如何在同步调用运行时间过长时将之丢弃：参考[章节14.5](14.5.md) 第二个变体

（12）如何在通道中使用计时器和定时器：参考[章节14.5](14.5.md)

（13）典型的服务器后端模型：参考[章节14.4](14.4.md)

# 18.9 网络和网页应用

## 18.9.1 模板：

制作、解析并使模板生效：

```go
var strTempl = template.Must(template.New("TName").Parse(strTemplateHTML))
```

在网页应用中使用HTML过滤器过滤HTML特殊字符：

`{{html .}}` 或者通过一个字段 `FieldName {{ .FieldName |html }}`

使用缓存模板（参考[章节15.7](15.7.md)） 

# 18.10 其他

如何在程序出错时终止程序：

```go	
if err != nil {
   fmt.Printf("Program stopping with error %v", err)
   os.Exit(1)
}
```

或者：

```go
if err != nil { 
	panic("ERROR occurred: " + err.Error())
}
```

# 18.11 出于性能考虑的最佳实践和建议

（1）尽可能的使用`:=`去初始化声明一个变量（在函数内部）；

（2）尽可能的使用字符代替字符串；

（3）尽可能的使用切片代替数组；

（4）尽可能的使用数组和切片代替映射（详见参考文献15）；

（5）如果只想获取切片中某项值，不需要值的索引，尽可能的使用`for range`去遍历切片，这比必须查询切片中的每个元素要快一些；

（6）当数组元素是稀疏的（例如有很多`0`值或者空值`nil`），使用映射会降低内存消耗；

（7）初始化映射时指定其容量；

（8）当定义一个方法时，使用指针类型作为方法的接受者；

（9）在代码中使用常量或者标志提取常量的值；

（10）尽可能在需要分配大量内存时使用缓存；

（11）使用缓存模板（参考[章节15.7](15.7.md)）。