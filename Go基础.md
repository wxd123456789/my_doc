# 关键字

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

使用 fallthrough 会强制执行后面的 case 语句，fallthrough 不会判断下一条 case 的表达式结果是否为 true。应该很少用到吧，

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

延迟（defer）语句 延时调用，你可以在函数中添加多个defer语句。当函数执行到最后时，这些defer语句会按照逆序执行，最后该函数返回。特别是当你在进行一些打开资源的操作时，遇到错误需要提前返回，在返回前你需要关闭相应的资源，不然很容易造成资源泄露等问题。在`defer`后指定的函数会在函数退出前调用。

```
    1. 关闭文件句柄
    2. 锁资源释放
    3. 数据库连接释放
```

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

**defer规则：**

1、延迟函数的参数在defer语句出现时就已经确定下来了

```go
func a() {
    i := 0
    defer fmt.Println(i)
    i++
    return
}
```

defer语句中的fmt.Println()参数i值在defer出现时就已经确定下来，实际上是拷贝了一份。后面对变量i的修改不会影响fmt.Println()函数的执行，仍然打印”0”。注意：对于指针类型参数，规则仍然适用，只不过延迟函数的参数是一个地址值，这种情况下，defer后面的语句对变量的修改可能会影响延迟函数。

2、延迟函数执行按后进先出顺序执行，即先出现的defer最后执行

定义defer类似于入栈操作，执行defer类似于出栈操作。设计defer的初衷是简化函数返回时资源清理的动作，资源往往有依赖顺序，比如先申请A资源，再根据A资源申请B资源，根据B资源申请C资源，即申请顺序是:A–>B–>C，释放时往往又要反向进行。这就是把defer设计成LIFO的原因。

每申请到一个用完需要释放的资源时，立即定义一个defer来释放资源是个很好的习惯。

3、延迟函数可能操作主函数的具名返回值

> defer延时调用修饰函数的坑：http://www.topgoer.com/%E5%87%BD%E6%95%B0/%E5%BB%B6%E8%BF%9F%E8%B0%83%E7%94%A8defer.html
>
> 。。。

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

# 数据结构

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

**chan**

对于无缓冲的 channel，发送方将阻塞该信道，直到接收方从该信道接收到数据为止，而接收方也将阻塞该信道，直到发送方将数据发送到该信道中为止。

对于有缓存的 channel，发送方在没有空插槽（缓冲区使用完）的情况下阻塞，而接收方在信道为空的情况下阻塞。



# 运算符

- 算术运算符
- 关系运算符
- 逻辑运算符
- 位运算符
- 赋值运算符
- 其他运算符

# 函数式

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

**main()和init()**

Go里面有两个保留的函数：`init`函数（能够应用于所有的`package`）和`main`函数（只能应用于`package main`）。这两个函数在定义时不能有任何的参数和返回值。虽然一个`package`里面可以写任意多个`init`函数，但这无论是对于可读性还是以后的可维护性来说，我们都强烈建议用户在一个`package`中每个文件只写一个`init`函数。

Go程序会自动调用`init()`和`main()`，所以你不需要在任何地方调用这两个函数。每个`package`中的`init`函数都是可选的，但`package main`就必须包含一个`main`函数。

程序的初始化和执行都起始于`main`包。如果`main`包还导入了其它的包，那么就会在编译时将它们依次导入。有时一个包会被多个包同时导入，那么它只会被导入一次（例如很多包可能都会用到`fmt`包，但它只会被导入一次，因为没有必要导入多次）。当一个包被导入时，如果该包还导入了其它的包，那么会先将其它包导入进来，然后再对这些包中的包级常量和变量进行初始化，接着执行`init`函数（如果有的话），依次类推。等所有被导入的包都加载完毕了，就会开始对`main`包中的包级常量和变量进行初始化，然后执行`main`包中的`init`函数（如果存在的话），最后执行`main`函数。下图详细地解释了整个执行过程：

![img](https://astaxie.gitbooks.io/build-web-application-with-golang/content/zh/images/2.3.init.png?raw=true)

`init()` 函数是 Go 程序初始化的一部分。Go 程序初始化先于 main 函数，由 runtime 初始化每个导入的包，初始化顺序不是按照从上到下的导入顺序，而是按照解析的依赖关系，没有依赖的包最先初始化。每个包首先初始化包作用域的常量和变量（常量优先于变量），然后执行包的 `init()` 函数。同一个包，甚至是同一个源文件可以有多个 `init()` 函数。`init()` 函数没有入参和返回值，不能被其他函数调用，同一个包内多个 `init()` 函数的执行顺序不作保证。

# 异常错误

**1、异常**

Golang 没有结构化异常，使用 panic 抛出错误，recover 捕获错误。异常的使用场景简单描述：Go中可以抛出一个panic的异常，然后在defer中通过recover捕获这个异常，然后正常处理。

panic：

```
    1、内置函数
    2、假如函数F中书写了panic语句，会终止其后要执行的代码，在panic所在函数F内如果存在要执行的defer函数列表，按照defer的逆序执行
    3、返回函数F的调用者G，在G中，调用函数F语句之后的代码不会执行，假如函数G中存在要执行的defer函数列表，按照defer的逆序执行
    4、直到goroutine整个退出，并报告错误
```

recover：

```
    1、内置函数
    2、用来控制一个goroutine的panicking行为，捕获panic，从而影响应用的行为
    3、一般的调用建议
        a). 在defer函数中，通过recover来终止一个goroutine的panicking过程，从而恢复正常代码的执行
        b). 可以获取通过panic传递的error
```

注意:

```
    1.利用recover处理panic指令，defer 必须放在 panic 之前定义，另外 recover 只有在 defer 调用的函数中才有效。否则当panic时，recover无法捕获到panic，无法防止panic扩散。
    2.recover 处理异常后，逻辑并不会恢复到 panic 那个点去，函数跑到 defer 之后的那个点。
    3.多个 defer 会形成 defer 栈，后定义的 defer 语句会被最先调用。
```

```go
package main

import "fmt"

func main() {
	test()
}

func catchError() {
	if err := recover(); err != nil {
		fmt.Println(err.(string))
	}
}

func test() {
	defer catchError()
	panic("panic error!")
	fmt.Println("end")
}
//panic error!
```

**2、错误**

导致关键流程出现不可修复性错误的使用 panic，其他使用 error。 

返回异常：

```go
package main

import (
   "errors"
   "fmt"
)

func getCircleArea(radius float32) (area float32, err error) {
   if radius < 0 {
      // 构建个异常对象
      err = errors.New("半径不能为负")
      return
   }
   area = 3.14 * radius * radius
   return
}

func main() {
   area, err := getCircleArea(-5)
   if err != nil {
      fmt.Println(err)
   } else {
      fmt.Println(area)
   }
}
```

自定义异常：

```go
package main

import (
    "fmt"
    "os"
    "time"
)

type PathError struct {
    path       string
    op         string
    createTime string
    message    string
}

func (p *PathError) Error() string {
    return fmt.Sprintf("path=%s \nop=%s \ncreateTime=%s \nmessage=%s", p.path,
        p.op, p.createTime, p.message)
}

func Open(filename string) error {

    file, err := os.Open(filename)
    if err != nil {
        return &PathError{
            path:       filename,
            op:         "read",
            message:    err.Error(),
            createTime: fmt.Sprintf("%v", time.Now()),
        }
    }

    defer file.Close()
    return nil
}

func main() {
    err := Open("/Users/5lmh/Desktop/go/src/test.txt")
    switch v := err.(type) {
    case *PathError:
        fmt.Println("get path error,", v)
    default:
    }

}
```

# 面向对象

**结构体struct**

结构体标签：

结构体标签由一个或多个键值对组成。键与值使用冒号分隔，值用双引号括起来。键值对之间使用一个空格分隔。 注意事项： 为结构体编写Tag时，必须严格遵守键值对的规则。结构体标签的解析代码的容错能力很差，一旦格式写错，编译和运行时都不会提示任何错误，通过反射也无法正确取值。例如不要在key和value之间添加空格。

Tag是结构体的元信息，可以在运行的时候通过反射的机制读取出来。Tag在结构体字段的后方定义，由一对反引号包裹起来，具体的格式如下：

```
`key1:"value1" key2:"value2"`
```

**公有和私有**

- 大写字母开头的变量是可导出的，也就是其它包可以读取的，是公有变量；小写字母开头的就是不可导出的，是私有变量。
- 大写字母开头的函数也是一样，相当于`class`中的带`public`关键词的公有函数；小写字母开头的就是有`private`关键词的私有函数。

**方法和接受者**

Go语言中的方法（Method）是一种作用于特定类型变量的函数。这种特定类型变量叫做接收者（Receiver）。接收者的概念就类似于其他语言中的this或者 self。 

指针类型的接收者由一个结构体的指针组成，由于指针的特性，调用方法时修改接收者指针的任意成员变量，在方法结束后，修改都是有效的。这种方式就十分接近于其他语言中面向对象中的this或者self。 

当方法作用于值类型接收者时，Go语言会在代码运行时将接收者的值复制一份。在值类型接收者的方法中可以获取接收者的成员值，但修改操作只是针对副本，无法修改接收者变量本身。 

什么时候应该使用指针类型接收者：

```
    1.需要修改接收者中的值
    2.接收者是拷贝代价比较大的大对象
    3.保证一致性，如果有某个方法使用了指针接收者，那么其他的方法也应该使用指针接收者。
```

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

所以，你不用担心你是调用的指针的method还是不是指针的method，Go知道你要做的一切。

- 一个T类型的值可以调用为`*T`类型声明的方法，但是仅当此T的值是可寻址(addressable) 的情况下。编译器在调用指针属主方法前，会自动取此T值的地址。因为不是任何T值都是可寻址的，所以并非任何T值都能够调用为类型`*T`声明的方法。哪些值是不可寻址的呢，包括字符串中的字节；map 对象中的元素（slice 对象中的元素是可寻址的，slice的底层是数组）；常量；包级别的函数等。
- 反过来，一个`*T`类型的值可以调用为类型T声明的方法，这是因为解引用指针总是合法的。事实上，你可以认为对于每一个为类型 T 声明的方法，编译器都会为类型`*T`自动隐式声明一个同名和同签名的方法。

举一个例子，定义类型 T，并为类型 `*T` 声明一个方法 `hello()`，变量 t1 可以调用该方法，但是常量 t2 调用该方法时，会产生编译错误。

```go
type T string

func (t *T) hello() {
	fmt.Println("hello")
}

func main() {
	var t1 T = "ABC"
	t1.hello() // hello
	const t2 T = "ABC"
	t2.hello() // error: cannot call pointer method on t
}
```

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

**类型断言**

空接口可以存储任意类型的值，获取空接口的类型和值。一个接口的值（简称接口值）是由一个具体类型和具体类型的值两部分组成的。这两部分分别称为接口的动态类型和动态值。 

```go
func testTypeAssert(){
   var x interface{}
   s := "123"
   x = s
   v, ok := x.(string)
   if ok{
      fmt.Printf("T: %T, V: %s\n", v, v)
   }
   justifyType(123)
}

func justifyType(x interface{}) {
   switch v := x.(type) {
   case string:
      fmt.Printf("x is a string，value is %v\n", v)
   case int:
      fmt.Printf("x is a int is %v\n", v)
   case bool:
      fmt.Printf("x is a bool is %v\n", v)
   default:
      fmt.Println("unsupport type！")
   }
}
```

**空struct**

使用空结构体 struct{} 可以节省内存，一般作为占位符使用，表明这里并不需要一个值。

```go
fmt.Println(unsafe.Sizeof(struct{}{})) // 0
// Set 比如使用 map 表示集合时，只关注 key，value 可以使用 struct{} 作为占位符。如果使用其他类型作为占位符，例如 int，bool，不仅浪费了内存，而且容易引起歧义。
type Set map[string]struct{}

func testSet() {
   set := make(Set)
   keys := []string{"A", "A", "b", "A"}
   for _, v := range keys {
      set[v] = struct{}{}
   }
   fmt.Println(len(set))
   if _, ok := set["A"]; ok {
      fmt.Printf("Key A exist")
   }
}
//使用信道(channel)控制并发时，我们只是需要一个信号，但并不需要传递值，这个时候，也可以使用 struct{} 代替。
func main() {
	ch := make(chan struct{}, 1)
	go func() {
		<-ch
		// do something
	}()
	ch <- struct{}{}
	// ...
}

//声明只包含方法的结构体。
type Lamp struct{}

func (l Lamp) On() {
        println("On")

}
func (l Lamp) Off() {
        println("Off")
}
```

**反射**

> 参考：https://blog.csdn.net/u012291393/article/details/78378386

值类型查看类型及值，修改

struct查看类型及字段，调用函数，修改字段值

匿名字段继承

# 常用库

## **内置库**

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

**fmt**

```go
//Print系列函数会将内容输出到系统的标准输出，区别在于Print函数直接输出内容，Printf函数支持格式化输出字符串，Println函数会在输出内容的结尾添加一个换行符。
fmt.Print("在终端打印该信息。")
name := "枯藤"
fmt.Printf("我是：%s\n", name)
fmt.Println("在终端打印单独一行显示")
//Sprint系列函数会把传入的数据生成并返回一个字符串。
s2 := fmt.Sprintf("name:%s,age:%d", name, 12)
fmt.Printf(s2)
//Errorf函数根据format参数生成格式化字符串并返回一个包含该字符串的错误。
err := fmt.Errorf("这是一个错误")
_ = err
// %v 和 %+v 都可以用来打印 struct 的值，区别在于 %v 仅打印各个字段的值，%+v 还会打印各个字段的名称。
// %s %d %p %T....

```

**IO**

```go
//////////////////////////////////
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
//////////////////////////////////bufio包实现了带缓冲区的读写，是对文件读写的封装 bufio缓冲写数据
func wr() {
    // w写 r读 x执行   w  2   r  4   x  1
    file, err := os.OpenFile("./xxx.txt", os.O_CREATE|os.O_WRONLY, 0666)
    if err != nil {
        return
    }
    defer file.Close()
    // 获取writer对象
    writer := bufio.NewWriter(file)
    for i := 0; i < 10; i++ {
        writer.WriteString("hello\n")
    }
    // 刷新缓冲区，强制写出
    writer.Flush()
}

func re() {
    file, err := os.Open("./xxx.txt")
    if err != nil {
        return
    }
    defer file.Close()
    reader := bufio.NewReader(file)
    for {
        line, _, err := reader.ReadLine()
        if err == io.EOF {
            break
        }
        if err != nil {
            return
        }
        fmt.Println(string(line))
    }

}
//////////////////////////////////ioutil 工具包写文件 工具包读取文件
func wr2() {
   err := ioutil.WriteFile("./yyy.txt", []byte("www.5lmh.com"), 0666)
   if err != nil {
      fmt.Println(err)
      return
   }
}

func re2() {
   content, err := ioutil.ReadFile("./yyy.txt")
   if err != nil {
      fmt.Println(err)
      return
   }
   fmt.Println(string(content))
}
```

**time**

```go
now := time.Now()
fmt.Printf("current time:%v\n", now)
year := now.Year()     //年
month := now.Month()   //月
day := now.Day()       //日
hour := now.Hour()     //小时
minute := now.Minute() //分钟
second := now.Second() //秒
fmt.Printf("%d-%02d-%02d %02d:%02d:%02d\n", year, month, day, hour, minute, second)
timestamp1 := now.Unix()     //时间戳
timestamp2 := now.UnixNano() //纳秒时间戳
fmt.Printf("current timestamp1:%v\n", timestamp1)
fmt.Printf("current timestamp2:%v\n", timestamp2)
later := now.Add(time.Hour)
fmt.Println(later)
ticker := time.Tick(time.Second)
for i := range ticker {
   fmt.Println(i)//每秒都会执行的任务
}
```

**strings**

...

**strconv**

```go
//strconv包实现了基本数据类型与其字符串表示的转换
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

**Log**

```go
func init() {
   logFile, err := os.OpenFile("./xx.log", os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0644)
   if err != nil {
      fmt.Println("open log file failed, err:", err)
      return
   }
   log.SetOutput(logFile)
   log.SetFlags(log.Llongfile | log.Lmicroseconds | log.Ldate)
   log.Println("这是自定义的logger记录的日志。")
}
// Go内置的log库功能有限，例如无法满足记录不同级别日志的情况，我们在实际的项目中根据自己的需要选择使用第三方的日志库，如logrus、zap等。
```

**net/http  重点！**

```go
var BASEURL = "http://127.0.0.1:8080"

func httpServer() {
   http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
      fmt.Fprintln(w, "hello world")
   })
   err := http.ListenAndServe(":8080", nil)
   if err != nil {
      fmt.Printf("http server failed, err:%v\n", err)
      return
   }
}

func httpGetReq() {
   apiUrl := fmt.Sprintf("%s/get", BASEURL)
   // URL param
   data := url.Values{}
   data.Set("name", "枯藤")
   data.Set("age", "18")
   u, err := url.ParseRequestURI(apiUrl)
   if err != nil {
      fmt.Printf("parse url requestUrl failed,err:%v\n", err)
   }
   u.RawQuery = data.Encode() // URL encode
   fmt.Println(u.String())
   resp, err := http.Get(u.String())
   if err != nil {
      fmt.Println("post failed, err:%v\n", err)
      return
   }
   defer resp.Body.Close()
   b, err := ioutil.ReadAll(resp.Body)
   if err != nil {
      fmt.Println("get resp failed,err:%v\n", err)
      return
   }
   fmt.Println(string(b))
}

func httpPostReq() {
   url := fmt.Sprintf("%s/post", BASEURL)
   contentType := "application/json"
   data := `{"name":"枯藤","age":18}`
   resp, err := http.Post(url, contentType, strings.NewReader(data))
   if err != nil {
      fmt.Println("post failed, err:%v\n", err)
      return
   }
   defer resp.Body.Close()
   b, err := ioutil.ReadAll(resp.Body)
   if err != nil {
      fmt.Println("get resp failed,err:%v\n", err)
      return
   }
   fmt.Println(string(b))
}
```

**json && xml**

```go
type Person struct {
    Name  string
    Hobby string
}
func main() {
    p := Person{"5lmh.com", "女"}
    // 编码json
    b, err := json.Marshal(p)
    if err != nil {
        fmt.Println("json err ", err)
    }
    fmt.Println(string(b))
}
```

```go
type Person struct {
    Age       int    `json:"age,string"`
    Name      string `json:"name"`
    Niubility bool   `json:"niubility"`
}

func main() {
    b := []byte(`{"age":"18","name":"5lmh.com","marry":false}`)
    var p Person
    err := json.Unmarshal(b, &p)
    if err != nil {
        fmt.Println(err)
    }
    fmt.Println(p)
}
```

```go
func main() {
    // 假数据
    // int和float64都当float64
    b := []byte(`{"age":1.3,"name":"5lmh.com","marry":false}`)

    // 声明接口
    var i interface{}
    err := json.Unmarshal(b, &i)
    if err != nil {
        fmt.Println(err)
    }
    // 自动转到map
    fmt.Println(i)
    // 可以判断类型
    m := i.(map[string]interface{})
    for k, v := range m {
        switch vv := v.(type) {
        case float64:
            fmt.Println(k, "是float64类型", vv)
        case string:
            fmt.Println(k, "是string类型", vv)
        default:
            fmt.Println("其他")
        }
    }
}
```

**Flag**

```go
//Go语言内置的flag包实现了命令行参数的解析，flag包使得开发命令行工具更为简单。
//os.Args是一个[]string
if len(os.Args) > 0 {
   for index, arg := range os.Args {
      fmt.Printf("args[%d]=%v\n", index, arg)
   }
}
//定义命令行参数方式1
var name string
var age int
var married bool
var delay time.Duration
flag.StringVar(&name, "name", "张三", "姓名")
flag.IntVar(&age, "age", 18, "年龄")
flag.BoolVar(&married, "married", false, "婚否")
flag.DurationVar(&delay, "d", 0, "延迟的时间间隔")

//解析命令行参数
flag.Parse()
fmt.Println(name, age, married, delay)
//返回命令行参数后的其他参数
fmt.Println(flag.Args())
//返回命令行参数后的其他参数个数
fmt.Println(flag.NArg())
//返回使用的命令行参数个数
fmt.Println(flag.NFlag())
// $ ./flag_demo -name pprof --age 28 -married=false -d=1h30m
```

**reflect**

。。。

## **第三方库**

```go
mysql驱动
orm db
simplejson
...

```


