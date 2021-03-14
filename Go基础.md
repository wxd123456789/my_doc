# 基础

## 关键字

面列举了 Go 代码中会使用到的 25 个关键字或保留字：

| break    | default     | func   | interface | select |
| -------- | ----------- | ------ | --------- | ------ |
| case     | defer       | go     | map       | struct |
| chan     | else        | goto   | package   | switch |
| const    | fallthrough | if     | range     | type   |
| continue | for         | import | return    | var    |

除了以上介绍的这些关键字，Go 语言还有 36 个预定义标识符：

| append | bool    | byte    | cap     | close  | complex | complex64 | complex128 | uint16 |
| ------ | ------- | ------- | ------- | ------ | ------- | --------- | ---------- | ------ |
| copy   | false   | float32 | float64 | imag   | int     | int8      | int16      | uint32 |
| int32  | int64   | iota    | len     | make   | new     | nil       | panic      | uint64 |
| print  | println | real    | recover | string | true    | uint      | uint8      | uintpt |

**fallthrough**

使用 fallthrough 会强制执行后面的 case 语句，fallthrough 不会判断下一条 case 的表达式结果是否为 true。

```go
switch {
case false:
   fmt.Println("1、case 条件语句为 false")
   fallthrough
case true:
   fmt.Println("2、case 条件语句为 true")
   fallthrough
case false:
   fmt.Println("3、case 条件语句为 false")
   fallthrough
case true:
   fmt.Println("4、case 条件语句为 true")
case false:
   fmt.Println("5、case 条件语句为 false")
   fallthrough
default:
   fmt.Println("6、默认 case")
}
/*2、case 条件语句为 true
3、case 条件语句为 false
4、case 条件语句为 true*/ 
```

**defer**

延迟（defer）语句，你可以在函数中添加多个defer语句。当函数执行到最后时，这些defer语句会按照逆序执行，最后该函数返回。特别是当你在进行一些打开资源的操作时，遇到错误需要提前返回，在返回前你需要关闭相应的资源，不然很容易造成资源泄露等问题。在`defer`后指定的函数会在函数退出前调用。

类似析构函数，常用关闭文件句柄、锁。

```go
func ReadWrite() bool {
    file.Open("file")
    defer file.Close()
    if failureX {
        return false
    }
    if failureY {
        return false
    }
    return true
}

//defer是采用后进先出模式 栈，所以如下代码会输出4 3 2 1 0
for i := 0; i < 5; i++ {
    defer fmt.Printf("%d ", i)
}
```

**make与new操作**

`make`用于内建类型（`map`、`slice` 和`channel`）的内存分配。`new`用于各种类型的内存分配。

。。。

**import**

`GOROOT`环境变量指定目录下去加载模块。

```go
 import(
     . "fmt"
 )
 //这个点操作的含义就是这个包导入之后在你调用这个包的函数时，你可以省略前缀的包名，也就是前面你调用的fmt.Println("hello world")可以省略的写成Println("hello world")

 import(
     f "fmt"
 )
//把包命名成另一个我们用起来容易记忆的名字

 import (
     "database/sql"
     _ "github.com/ziutek/mymysql/godrv"
 )
//_操作其实是引入该包，而不直接使用包里面的函数，而是调用了该包里面的init函数
```

## 数据结构

**分类**

| 序号 | 类型和描述                                                   |
| ---- | ------------------------------------------------------------ |
| 1    | **布尔型** 布尔型的值只可以是常量 true 或者 false。一个简单的例子：var b bool = true。 |
| 2    | **数字类型** 整型 int 和浮点型 float32、float64，Go 语言支持整型和浮点型数字，并且支持复数，其中位的运算采用补码。byte |
| 3    | **字符串类型:** 字符串就是一串固定长度的字符连接起来的字符序列。Go 的字符串是由单个字节连接起来的。Go 语言的字符串的字节使用 UTF-8 编码标识 Unicode 文本。 |
| 4    | **派生类型:** 包括：(a) 指针类型（Pointer）(b) 数组类型(c) 结构化类型(struct)(d) Channel 类型(e) 函数类型(f) 切片类型(g) 接口类型（interface）(h) Map 类型 |

**字节byte**

```go
_s := "ABC"
b := []byte(_s)
fmt.Println(b) //[65 66 67]

```

**数字类型**

| 序号 | 类型和描述                                                   |
| ---- | ------------------------------------------------------------ |
| 1    | **uint8** 无符号 8 位整型 (0 到 255)                         |
| 2    | **uint16** 无符号 16 位整型 (0 到 65535)                     |
| 3    | **uint32** 无符号 32 位整型 (0 到 4294967295)                |
| 4    | **uint64** 无符号 64 位整型 (0 到 18446744073709551615)      |
| 5    | **int8** 有符号 8 位整型 (-128 到 127)                       |
| 6    | **int16** 有符号 16 位整型 (-32768 到 32767)                 |
| 7    | **int32** 有符号 32 位整型 (-2147483648 到 2147483647)       |
| 8    | **int64** 有符号 64 位整型 (-9223372036854775808 到 9223372036854775807) |

**浮点型**

| 序号 | 类型和描述                        |
| ---- | --------------------------------- |
| 1    | **float32** IEEE-754 32位浮点型数 |
| 2    | **float64** IEEE-754 64位浮点型数 |
| 3    | **complex64** 32 位实数和虚数     |
| 4    | **complex128** 64 位实数和虚数    |

**字符串**

```go
fmt.Println(`aaa"123"`)// aaa"123"
//字符串处理

```

**切片**

Go 数组的长度不可改变，在特定场景中这样的集合就不太适用，Go 中提供了一种灵活，功能强悍的内置类型切片("动态数组")，与数组相比切片的长度是不固定的，可以追加元素，在追加时可能使切片的容量增大。 类似于java中的ArrayList，动态库容机制。。。

```go
sliceA := []int{1, 2}
p(sliceA)
sliceB := make([]int, 2, 10)
p(sliceB) //[[0 0]]
sliceB = append(sliceB, 3, 4, 5)
sliceC := make([]int, len(sliceB), 2*cap(sliceB))
copy(sliceC, sliceB)
```

**map**

Map 是一种无序的键值对的集合。Map 最重要的一点是通过 key 来快速检索数据，key 类似于索引，指向数据的值。 

**内置的运算符：**

- 算术运算符
- 关系运算符
- 逻辑运算符
- 位运算符
- 赋值运算符
- 其他运算符

## 函数式

**函数传值和传引用**

传指针的好处

- 传指针使得多个函数能操作同一个对象。
- 传指针比较轻量级 (8bytes),只是传内存地址，我们可以用指针传递体积大的结构体。如果用参数值传递的话, 在每次copy上面就会花费相对较多的系统开销（内存和时间）。所以当你要传递大的结构体的时候，用指针是一个明智的选择。
- Go语言中`channel`，`slice`，`map`这三种类型的实现机制类似指针，所以可以直接传递，而不用取地址后传递指针。（注：若函数需改变`slice`的长度，则仍需要取地址传递指针）

**函数作为值、类型**

在Go中函数也是一种变量，我们可以通过`type`来定义它，它的类型就是所有拥有相同的参数，相同的返回值的一种类型

```
type typeName func(input1 inputType1 , input2 inputType2 [, ...]) (result1 resultType1 [, ...])
```

函数作为类型到底有什么好处呢？那就是可以把这个类型的函数当做值来传递，请看下面的例子

```go
package main

import "fmt"

type testInt func(int) bool // 声明了一个函数类型

func isOdd(integer int) bool {
    if integer%2 == 0 {
        return false
    }
    return true
}

func isEven(integer int) bool {
    if integer%2 == 0 {
        return true
    }
    return false
}

// f testInt
func filter(slice []int, f testInt) []int {
    var result []int
    for _, value := range slice {
        if f(value) {
            result = append(result, value)
        }
    }
    return result
}

func main(){
    slice := []int {1, 2, 3, 4, 5, 7}
    fmt.Println("slice = ", slice)
    odd := filter(slice, isOdd)    // 函数当做值来传递了
    fmt.Println("Odd elements of slice are: ", odd)
    even := filter(slice, isEven)  // 函数当做值来传递了
    fmt.Println("Even elements of slice are: ", even)
}
```

**main函数和init函数**

Go里面有两个保留的函数：`init`函数（能够应用于所有的`package`）和`main`函数（只能应用于`package main`）。这两个函数在定义时不能有任何的参数和返回值。虽然一个`package`里面可以写任意多个`init`函数，但这无论是对于可读性还是以后的可维护性来说，我们都强烈建议用户在一个`package`中每个文件只写一个`init`函数。

Go程序会自动调用`init()`和`main()`，所以你不需要在任何地方调用这两个函数。每个`package`中的`init`函数都是可选的，但`package main`就必须包含一个`main`函数。

程序的初始化和执行都起始于`main`包。如果`main`包还导入了其它的包，那么就会在编译时将它们依次导入。有时一个包会被多个包同时导入，那么它只会被导入一次（例如很多包可能都会用到`fmt`包，但它只会被导入一次，因为没有必要导入多次）。当一个包被导入时，如果该包还导入了其它的包，那么会先将其它包导入进来，然后再对这些包中的包级常量和变量进行初始化，接着执行`init`函数（如果有的话），依次类推。等所有被导入的包都加载完毕了，就会开始对`main`包中的包级常量和变量进行初始化，然后执行`main`包中的`init`函数（如果存在的话），最后执行`main`函数。下图详细地解释了整个执行过程：

![img](https://astaxie.gitbooks.io/build-web-application-with-golang/content/zh/images/2.3.init.png?raw=true)

## 面向对象

**公有和私有**

- 大写字母开头的变量是可导出的，也就是其它包可以读取的，是公有变量；小写字母开头的就是不可导出的，是私有变量。
- 大写字母开头的函数也是一样，相当于`class`中的带`public`关键词的公有函数；小写字母开头的就是有`private`关键词的私有函数。

**匿名字段**

当匿名字段是一个struct的时候，那么这个struct所拥有的全部字段都被隐式地引入了当前定义的这个struct。 **继承匿名字段类的字段、方法。**

```go
type Skills []string

type Human struct {
    name string
    age int
    weight int
}

type Student struct {
    Human  // 匿名字段，struct 继承Human的所有字段
    Skills // 匿名字段，自定义的类型string slice
    int    // 内置类型作为匿名字段
    speciality string
}

func main() {
    // 初始化学生Jane
    jane := Student{Human:Human{"Jane", 35, 100}, speciality:"Biology"}
}
```

```go
package main

import "fmt"

const(
    WHITE = iota
    BLACK
    BLUE
    RED
    YELLOW
)

type Color byte

type Box struct {
    width, height, depth float64
    color Color
}

type BoxList []Box //a slice of boxes

func (b Box) Volume() float64 {
    return b.width * b.height * b.depth
}

func (b *Box) SetColor(c Color) {
    b.color = c
}

func (bl BoxList) BiggestColor() Color {
    v := 0.00
    k := Color(WHITE)
    for _, b := range bl {
        if bv := b.Volume(); bv > v {
            v = bv
            k = b.color
        }
    }
    return k
}

func (bl BoxList) PaintItBlack() {
    for i, _ := range bl {
        bl[i].SetColor(BLACK)
    }
}

func (c Color) String() string {
    strings := []string {"WHITE", "BLACK", "BLUE", "RED", "YELLOW"}
    return strings[c]
}

func main() {
    boxes := BoxList {
        Box{4, 4, 4, RED},
        Box{10, 10, 1, YELLOW},
        Box{1, 1, 20, BLACK},
        Box{10, 10, 1, BLUE},
        Box{10, 30, 1, WHITE},
        Box{20, 20, 20, YELLOW},
    }

    fmt.Printf("We have %d boxes in our set\n", len(boxes))
    fmt.Println("The volume of the first one is", boxes[0].Volume(), "cm³")
    fmt.Println("The color of the last one is",boxes[len(boxes)-1].color.String())
    fmt.Println("The biggest one is", boxes.BiggestColor().String())

    fmt.Println("Let's paint them all black")
    boxes.PaintItBlack()
    fmt.Println("The color of the second one is", boxes[1].color.String())

    fmt.Println("Obviously, now, the biggest one is", boxes.BiggestColor().String())
}
```

**指针作为receiver**

现在让我们回过头来看看SetColor这个method，它的receiver是一个指向Box的指针，是的，你可以使用*Box。想想为啥要使用指针而不是Box本身呢？

我们定义SetColor的真正目的是想改变这个Box的颜色，如果不传Box的指针，那么SetColor接受的其实是Box的一个copy，也就是说method内对于颜色值的修改，其实只作用于Box的copy，而不是真正的Box。所以我们需要传入指针。

这里可以把receiver当作method的第一个参数来看，然后结合前面函数讲解的传值和传引用就不难理解

这里你也许会问了那SetColor函数里面应该这样定义`*b.Color=c`,而不是`b.Color=c`,因为我们需要读取到指针相应的值。

你是对的，其实Go里面这两种方式都是正确的，当你用指针去访问相应的字段时(虽然指针没有任何的字段)，Go知道你要通过指针去获取这个值，看到了吧，Go的设计是不是越来越吸引你了。

也许细心的读者会问这样的问题，PaintItBlack里面调用SetColor的时候是不是应该写成`(&bl[i]).SetColor(BLACK)`，因为SetColor的receiver是*Box，而不是Box。

你又说对的，这两种方式都可以，因为Go知道receiver是指针，他自动帮你转了。

也就是说：

> 如果一个method的receiver是*T,你可以在一个T类型的实例变量V上面调用这个method，而不需要&V去调用这个method

类似的

> 如果一个method的receiver是T，你可以在一个*T类型的变量P上面调用这个method，而不需要* P去调用这个method

所以，你不用担心你是调用的指针的method还是不是指针的method，Go知道你要做的一切，这对于有多年C/C++编程经验的同学来说，真是解决了一个很大的痛苦。



**接口**

interface是一组method签名的组合，我们通过interface来定义对象的一组行为。 一个对象可以实现任意多个interface 。任意的类型都实现了空interface(我们这样定义：interface{}， 类似于c中的void *，java中的Object)，也就是包含0个method的interface。 

```go
package main

import "fmt"

type Human struct {
    name string
    age int
    phone string
}

type Student struct {
    Human //匿名字段
    school string
    loan float32
}

type Employee struct {
    Human //匿名字段
    company string
    money float32
}

//Human实现SayHi方法
func (h Human) SayHi() {
    fmt.Printf("Hi, I am %s you can call me on %s\n", h.name, h.phone)
}

//Human实现Sing方法
func (h Human) Sing(lyrics string) {
    fmt.Println("La la la la...", lyrics)
}

//Employee重载Human的SayHi方法
func (e Employee) SayHi() {
    fmt.Printf("Hi, I am %s, I work at %s. Call me on %s\n", e.name,
        e.company, e.phone)
    }

// Interface Men被Human,Student和Employee实现
// 因为这三个类型都实现了这两个方法
type Men interface {
    SayHi()
    Sing(lyrics string)
}

func main() {
    mike := Student{Human{"Mike", 25, "222-222-XXX"}, "MIT", 0.00}
    paul := Student{Human{"Paul", 26, "111-222-XXX"}, "Harvard", 100}
    sam := Employee{Human{"Sam", 36, "444-222-XXX"}, "Golang Inc.", 1000}
    tom := Employee{Human{"Tom", 37, "222-444-XXX"}, "Things Ltd.", 5000}

    //定义Men类型的变量i
    var i Men

    //i能存储Student
    i = mike
    fmt.Println("This is Mike, a Student:")
    i.SayHi()
    i.Sing("November rain")

    //i也能存储Employee
    i = tom
    fmt.Println("This is tom, an Employee:")
    i.SayHi()
    i.Sing("Born to be wild")

    //定义了slice Men
    fmt.Println("Let's use a slice of Men and see what happens")
    x := make([]Men, 3)
    //这三个都是不同类型的元素，但是他们实现了interface同一个接口
    x[0], x[1], x[2] = paul, sam, mike

    for _, value := range x{
        value.SayHi()
    }
    
    ///////////////////////////////////
    // 定义a为空接口
	var a interface{}
	var i int = 5
	s := "Hello world"
	// a可以存储任意类型的数值
	a = i
	a = s
}
```

嵌入interface：如果一个interface1作为interface2的一个嵌入字段，那么interface2隐式的包含了interface1里面的method，接口的继承。 



**反射**

反射就是能检查程序在运行时的状态。我们一般用到的包是reflect包。 

## 并发

goroutine是Go并行设计的核心。**goroutine说到底其实就是协程**，但是它比线程更小，十几个goroutine可能体现在底层就是五六个线程，Go语言内部帮你实现了这些goroutine之间的内存共享。执行goroutine只需极少的栈内存(大概是4~5KB)，当然会根据相应的数据伸缩。也正因为如此，可同时运行成千上万个并发任务。goroutine比thread更易用、更高效、更轻便。 

**chan**

通道（channel）是用来传递数据的一个数据结构。通道可用于两个 goroutine 之间通过传递一个指定类型的值来同步运行和通讯。操作符 `<-` 用于指定通道的方向，发送或接收。如果未指定方向，则为双向通道。

默认情况下，channel接收和发送数据都是阻塞的，除非另一端已经准备好，这样就使得Goroutines同步变的更加的简单，而不需要显式的lock。所谓阻塞，也就是如果读取（value := <-ch）它将会被阻塞，直到有数据接收。其次，任何发送（ch<-5）将会被阻塞，直到数据被读出。无缓冲channel是在多个goroutine之间同步很棒的工具。 

缓存通道：

允许指定channel的缓冲大小，很简单，就是channel可以存储多少元素。ch:= make(chan bool, 4)，创建了可以存储4个元素的bool 型channel。在这个channel 中，前4个元素可以无阻塞的写入。当写入第5个元素时，代码将会阻塞，直到其他goroutine从channel 中读取一些元素，腾出空间。 

**select**

适应于存在多个channel的时候。select可以监听channel上的数据流动。select默认是阻塞的，只有当监听的channel中有发送或接收可以进行时才会运行，当多个channel都准备好的时候，select是随机的选择一个执行的。

。。。



## 常用库

**内置库**

```go
encoding/json  Unmarshal json.Marshal(s) //json标准包来编解码JSON数据
encoding/xml //xml解析
regexp //正则 \\
os //文件
strconv //字符串转换
strings //字符串操作
reflect // 反射

//http
net/http
net/rpc
```

```go
func testJson() {
   var s Serverslice
   str := `{"servers":[{"serverName":"Shanghai_VPN","serverIP":"127.0.0.1"},{"serverName":"Beijing_VPN","serverIP":"127.0.0.2"}]}`
   json.Unmarshal([]byte(str), &s)
   fmt.Println(s)
}

func makeFile() {
   os.Mkdir("tmptest", 0777)
   fileName := "tmp.txt"
   _f, err := os.Create(fileName)
   if err != nil {
      fmt.Println(_f, err)
      return
   }
   defer _f.Close()
   for i := 0; i < 10; i++ {
      _f.WriteString("test\n")
      _f.Write([]byte("test\n"))
   }
}

func readFile() {
   fileName := "tmp.txt"
   _f, err := os.Open(fileName)
   if err != nil {
      fmt.Println(_f, err)
      return
   }
   defer _f.Close()
   buf := make([]byte, 1024)
   for {
      n, _ := _f.Read(buf)
      if n == 0 {
         break
      }
      os.Stdout.Write(buf[:n])
   }
}

func testStrOp() {
   // str operation
   fmt.Println(strings.Contains("seafood", "foo"))
   //...

   // *** to str
   str := make([]byte, 0, 100) //byte slice
   str = strconv.AppendInt(str, 4567, 10)
   str = strconv.AppendBool(str, false)
   str = strconv.AppendQuote(str, "abcdefg")
   str = strconv.AppendQuoteRune(str, '单')
   fmt.Println(string(str))

   // str to ***
   a, err := strconv.ParseBool("false")
   checkError(err)
   b, err := strconv.ParseFloat("123.23", 64)
   checkError(err)
   fmt.Println(a, b)
}
func checkError(e error) {
   if e != nil {
      fmt.Println(e)
   }
}
```



**第三方库**

```go
mysql驱动
orm db
simplejson


```



# 原理

## 数据结构

切片

slice源码



map

map源码



## 并发

channel 源码

协程池：不需要？？池化要解决的问题一个是频繁创建的开销，另一个是在等待时占用的资源。goroutine 和普通线程相比，创建和调度都不需要进入内核，也就是创建的开销已经解决了。同时相比系统线程，内存占用也是轻量的。所以池化技术要解决的问题goroutine 都不存在 ？？





