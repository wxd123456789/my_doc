# 数据结构

## chan

channel是Golang在语言层面提供的goroutine间的通信方式，比Unix管道更易用也更轻便。channel主要用于进程内各goroutine间通信，如果需要跨进程通信，建议使用分布式系统的方法来解决。

本章从源码角度分析channel的实现机制，实际上这部分源码非常简单易读。

**chan数据结构：**

```go
type hchan struct {
    qcount   uint           // 当前队列中剩余元素个数
    dataqsiz uint           // 环形队列长度，即可以存放的元素个数
    buf      unsafe.Pointer // 环形队列指针
    elemsize uint16         // 每个元素的大小
    closed   uint32            // 标识关闭状态
    elemtype *_type         // 元素类型
    sendx    uint           // 队列下标，指示元素写入时存放到队列中的位置
    recvx    uint           // 队列下标，指示元素从队列的该位置读出
    recvq    waitq          // 等待读消息的goroutine队列
    sendq    waitq          // 等待写消息的goroutine队列
    lock mutex              // 互斥锁，chan不允许并发读写
}
```

从数据结构可以看出channel由队列、类型信息、goroutine等待队列组成，下面分别说明其原理。

**环形队列**

chan内部实现了一个环形队列作为其缓冲区，队列的长度是创建chan时指定的。

下图展示了一个可缓存6个元素的channel示意图：

![null](http://wen.topgoer.com/uploads/gozhuanjia/images/m_f1b42d200c5d94d02eeacef7c99aa81b_r.png)

- dataqsiz指示了队列长度为6，即可缓存6个元素；
- buf指向队列的内存，队列中还剩余两个元素；
- qcount表示队列中还有两个元素；
- sendx指示后续写入的数据存储的位置，取值[0, 6)；
- recvx指示从该位置读取数据, 取值[0, 6)；

**等待队列**

从channel读数据，如果channel缓冲区为空或者没有缓冲区，当前goroutine会被阻塞。
向channel写数据，如果channel缓冲区已满或者没有缓冲区，当前goroutine会被阻塞。

被阻塞的goroutine将会挂在channel的等待队列中：

- 因读阻塞的goroutine会被向channel写入数据的goroutine唤醒；
- 因写阻塞的goroutine会被从channel读数据的goroutine唤醒；

下图展示了一个没有缓冲区的channel，有几个goroutine阻塞等待读数据：

![null](http://wen.topgoer.com/uploads/gozhuanjia/images/m_f48c37e012c38de53aeb532c993b6d2d_r.png)

注意，一般情况下recvq和sendq至少有一个为空。只有一个例外，那就是同一个goroutine使用select语句向channel一边写数据，一边读数据。

**创建channel**

创建channel的过程实际上是初始化hchan结构。其中类型信息和缓冲区长度由make语句传入，buf的大小则与元素大小和缓冲区长度共同决定。

创建channel的伪代码如下所示：

```
func makechan(t *chantype, size int) *hchan {
    var c *hchan
    c = new(hchan)
    c.buf = malloc(元素类型大小*size)
    c.elemsize = 元素类型大小
    c.elemtype = 元素类型
    c.dataqsiz = size

    return c
}
```

**向channel写数据**

向一个channel中写数据简单过程如下：

1. 如果等待接收队列recvq不为空，说明缓冲区中没有数据或者没有缓冲区，此时直接从recvq取出G,并把数据写入，最后把该G唤醒，结束发送过程；
2. 如果缓冲区中有空余位置，将数据写入缓冲区，结束发送过程；
3. 如果缓冲区中没有空余位置，将待发送数据写入G，将当前G加入sendq，进入睡眠，等待被读goroutine唤醒；

简单流程图如下：

![null](http://wen.topgoer.com/uploads/gozhuanjia/images/m_b235ef1f2c6ac1b5d63ec5660da97bd2_r.png)

**从channel读数据**

从一个channel读数据简单过程如下：

1. 如果等待发送队列sendq不为空，且没有缓冲区，直接从sendq中取出G，把G中数据读出，最后把G唤醒，结束读取过程；
2. 如果等待发送队列sendq不为空，此时说明缓冲区已满，从缓冲区中首部读出数据，把G中数据写入缓冲区尾部，把G唤醒，结束读取过程；
3. 如果缓冲区中有数据，则从缓冲区取出数据，结束读取过程；
4. 将当前goroutine加入recvq，进入睡眠，等待被写goroutine唤醒；

简单流程图如下：

![null](http://wen.topgoer.com/uploads/gozhuanjia/images/m_933ca9af4c3ec1db0b94b8b4ec208d4b_r.png)

**关闭channel**

关闭channel时会把recvq中的G全部唤醒，本该写入G的数据位置为nil。把sendq中的G全部唤醒，但这些G会panic。

除此之外，panic出现的常见场景还有：

1. 关闭值为nil的channel
2. 关闭已经被关闭的channel
3. 向已经关闭的channel写数据

## slice

> 参考：
>
> http://www.topgoer.com/go%E5%9F%BA%E7%A1%80/Slice%E5%BA%95%E5%B1%82%E5%AE%9E%E7%8E%B0.html
>
> http://wen.topgoer.com/docs/gozhuanjia/gozhuanjiaslice

## map

其底层存储方式为数组 。。。



# 网络编程

## Socket

socket图解：

![socket图解](http://www.topgoer.com/static/6.1/3.png) 

。。。

## REST

。。。

## RPC

。。。

## WebSocket

WebSocket是一种在单个TCP连接上进行全双工通信的协议，使得客户端和服务器之间的数据交换变得更加简单，允许服务端主动向客户端推送数据，浏览器和服务器只需要完成一次握手，两者之间就直接可以创建持久性的连接，并进行双向数据传输。需要安装第三方包： github.com/gorilla/websocket。

> 应用：http://www.topgoer.com/%E7%BD%91%E7%BB%9C%E7%BC%96%E7%A8%8B/WebSocket%E7%BC%96%E7%A8%8B.html 多人在线聊天

# 并发编程

## goroutine

goroutine是Go并行设计的核心。**goroutine说到底其实就是协程**，但是它比线程更小，十几个goroutine可能体现在底层就是五六个线程，Go语言内部帮你实现了这些goroutine之间的内存共享。执行goroutine只需极少的栈内存(大概是4~5KB)，当然会根据相应的数据伸缩。也正因为如此，可同时运行成千上万个并发任务。goroutine比thread更易用、更高效、更轻便。 

```
协程：独立的栈空间，共享堆空间，调度由用户自己控制，本质上有点类似于用户级线程，这些用户级线程的调度也是自己实现的。
线程：一个线程上可以跑多个协程，协程是轻量级的线程。

多线程程序在一个核的cpu上运行，就是并发。
多线程程序在多个核的cpu上运行，就是并行。

goroutine 只是由官方实现的超级"线程池"。
每个实力4~5KB的栈内存占用和由于实现机制而大幅减少的创建和销毁开销是go高并发的根本原因。goroutine 奉行通过通信来共享内存，而不是共享内存来通信。
```

```go
//这一次的执行结果只打印了main goroutine done!，并没有打印Hello Goroutine。
//在程序启动时，Go程序就会为main()函数创建一个默认的goroutine。当main()函数返回的时候该goroutine就结束了，所有在main()函数中启动的goroutine会一同结束。即如果主协程退出了，其他任务不执行了
func hello() {
   fmt.Println("Hello Goroutine!")
}
func main() {
   go hello()
   fmt.Println("main goroutine done!")
}
```

OS线程（操作系统线程）一般都有固定的栈内存（通常为2MB）。然而一个goroutine的栈在其生命周期开始时只有很小的栈（典型情况下2KB），goroutine的栈不是固定的，他可以按需增大和缩小，goroutine的栈大小限制可以达到1GB，虽然极少会用到这个大。所以在Go语言中一次创建十万左右的goroutine也是可以的。

## chan

通道（channel）是用来传递数据的一个数据结构。通道可用于两个 goroutine 之间通过传递一个指定类型的值来同步运行和通讯。操作符 `<-` 用于指定通道的方向，发送或接收。如果未指定方向，则为双向通道。

默认情况下，channel接收和发送数据都是阻塞的，除非另一端已经准备好，这样就使得Goroutines同步变的更加的简单，而不需要显式的lock。所谓阻塞，也就是如果读取（value := <-ch）它将会被阻塞，直到有数据接收。其次，任何发送（ch<-5）将会被阻塞，直到数据被读出。无缓冲channel是在多个goroutine之间同步很棒的工具。 

缓存通道：

允许指定channel的缓冲大小，很简单，就是channel可以存储多少元素。ch:= make(chan bool, 4)，创建了可以存储4个元素的bool 型channel。在这个channel 中，前4个元素可以无阻塞的写入。当写入第5个元素时，代码将会阻塞，直到其他goroutine从channel 中读取一些元素，腾出空间。 

已关闭的channel也是可读的 。

## select

与chan通道的发送和接收配合使用。select是Golang在语言层面提供的多路IO复用的机制，其可以检测多个channel是否ready(即是否可读或可写)，使用起来非常方便 。

适应于存在多个channel的时候。select可以监听channel上的数据流动。select默认是阻塞的，只有当监听的channel中有发送或接收可以进行时才会运行，当多个channel都准备好的时候，select是随机的选择一个执行的。select 语句类似于 switch 语句，但是select会随机执行一个可运行的case。如果没有case可运行，它将阻塞，直到有case可运行。select 是Go中的一个控制结构，类似于用于通信的switch语句。每个case必须是一个通信操作IO操作，要么是发送要么是接收。 select 随机执行一个可运行的case。如果没有case可运行，它将阻塞，直到有case可运行。一个默认的子句应该总是可运行的。

```go
    每个case都必须是一个通信
    所有channel表达式都会被求值
    所有被发送的表达式都会被求值
    如果任意某个通信可以进行，它就执行；其他被忽略。
    如果有多个case都可以运行，Select会随机公平地选出一个执行。其他不会执行。
    否则：
    如果有default子句，则执行该语句。
    如果没有default字句，select将阻塞，直到某个通信可以运行；Go不会重新对channel或值进行求值。
    //
    select { //不停的在这里检测
    case <-chanl : //检测有没有数据可以读
    //如果chanl成功读取到数据，则进行该case处理语句
    case chan2 <- 1 : //检测有没有可以写
    //如果成功向chan2写入数据，则进行该case处理语句
    //假如没有default，那么在以上两个条件都不成立的情况下，就会在此阻塞
    default:
    //如果以上都没有符合条件，那么则进行default处理流程
    }
```

## sync

### Mutex

> 引用：http://wen.topgoer.com/docs/gozhuanjia/gozhuanjiamutex

互斥锁是并发程序中对共享资源进行访问控制的主要手段 。互斥锁的数据结构：

```
type Mutex struct {
    state int32
    sema  uint32
}
```

- Mutex.state表示互斥锁的状态，比如是否被锁定等。
- Mutex.sema表示信号量，协程阻塞等待该信号量，解锁的协程释放信号量从而唤醒等待信号量的协程。

我们看到Mutex.state是32位的整型变量，内部实现时把该变量分成四份，用于记录Mutex的四种状态。

下图展示Mutex的内存布局：

![null](http://wen.topgoer.com/uploads/gozhuanjia/images/m_45a91868c2c9d5dc2617e9fda0e46049_r.png)

- Locked: 表示该Mutex是否已被锁定，0：没有锁定 1：已被锁定。
- Waiter: 表示阻塞等待锁的协程个数，协程解锁时根据此值来判断是否需要释放信号量。
- Woken: 表示是否有协程已被唤醒，0：没有协程唤醒 1：已有协程唤醒，正在加锁过程中。
- Starving：表示该Mutex是否处于饥饿状态，0：没有饥饿 1：饥饿状态，说明有协程阻塞了超过1ms。

协程之间抢锁实际上是抢给Locked赋值的权利，能给Locked域置1，就说明抢锁成功。抢不到的话就阻塞等待Mutex.sema信号量，一旦持有锁的协程解锁，等待的协程会依次被唤醒。

**加解锁过程：**

简单加锁

假定当前只有一个协程在加锁，没有其他协程干扰，那么过程如下图所示：

![null](http://wen.topgoer.com/uploads/gozhuanjia/images/m_4b43555a5440890c948680aab58e982f_r.png)

加锁过程会去判断Locked标志位是否为0，如果是0则把Locked位置1，代表加锁成功。从上图可见，加锁成功后，只是Locked位置1，其他状态位没发生变化。

加锁被阻塞

假定加锁时，锁已被其他协程占用了，此时加锁过程如下图所示：

![null](http://wen.topgoer.com/uploads/gozhuanjia/images/m_7009a47d6e8acb7e7b9c421ad1fece22_r.png)

从上图可看到，当协程B对一个已被占用的锁再次加锁时，Waiter计数器增加了1，此时协程B将被阻塞，直到Locked值变为0后才会被唤醒。

简单解锁

假定解锁时，没有其他协程阻塞，此时解锁过程如下图所示：

![null](http://wen.topgoer.com/uploads/gozhuanjia/images/m_6f4885510e5659f0615f17e6a5b89d2f_r.png)

由于没有其他协程阻塞等待加锁，所以此时解锁时只需要把Locked位置为0即可，不需要释放信号量。

解锁并唤醒协程

假定解锁时，有1个或多个协程阻塞，此时解锁过程如下图所示：

![null](http://wen.topgoer.com/uploads/gozhuanjia/images/m_f45d385c3f7bacc272878bbfd6d48182_r.png)

协程A解锁过程分为两个步骤，一是把Locked位置0，二是查看到Waiter>0，所以释放一个信号量，唤醒一个阻塞的协程，被唤醒的协程B把Locked位置1，于是协程B获得锁。

**自旋过程：**

加锁时，如果当前Locked位为1，说明该锁当前由其他协程持有，尝试加锁的协程并不是马上转入阻塞，而是会持续的探测Locked位是否变为0，这个过程即为自旋过程。自旋时间很短，但如果在自旋过程中发现锁已被释放，那么协程可以立即获取锁。此时即便有协程被唤醒也无法获取锁，只能再次阻塞。自旋的好处是，当加锁失败时不必立即转入阻塞，有一定机会获取到锁，这样可以避免协程的切换。

**自旋对应于CPU的”PAUSE”指令，CPU对该指令什么都不做，相当于CPU空转**，对程序而言相当于sleep了一小段时间，时间非常短，当前实现是30个时钟周期。自旋过程中会持续探测Locked是否变为0，连续两次探测间隔就是执行这些PAUSE指令，它不同于sleep，不需要将协程转为睡眠状态。

自旋条件：

加锁时程序会自动判断是否可以自旋，无限制的自旋将会给CPU带来巨大压力，所以判断是否可以自旋就很重要了。自旋必须满足以下所有条件：

- 自旋次数要足够小，通常为4，即自旋最多4次
- CPU核数要大于1，否则自旋没有意义，因为此时不可能有其他协程释放锁
- 协程调度机制中的Process数量要大于1，比如使用GOMAXPROCS()将处理器设置为1就不能启用自旋
- 协程调度机制中的可运行队列必须为空，否则会延迟协程调度

可见，自旋的条件是很苛刻的，总而言之就是不忙的时候才会启用自旋。

自旋的优势和问题：

自旋的优势是更充分的利用CPU，尽量避免协程切换。因为当前申请加锁的协程拥有CPU，如果经过短时间的自旋可以获得锁，当前协程可以继续运行，不必进入阻塞状态。

如果自旋过程中获得锁，那么之前被阻塞的协程将无法获得锁，如果加锁的协程特别多，每次都通过自旋获得锁，那么之前被阻塞的进程将很难获得锁，从而进入饥饿状态。

为了避免协程长时间无法获取锁，自1.8版本以来增加了一个状态，即Mutex的Starving状态。这个状态下不会自旋，一旦有协程释放锁，那么一定会唤醒一个协程并成功加锁。

**Mutex模式：**

前面分析加锁和解锁过程中只关注了Waiter和Locked位的变化，现在我们看一下Starving位的作用。每个Mutex都有两个模式，称为Normal和Starving。下面分别说明这两个模式。

normal模式：默认情况下，Mutex的模式为normal。该模式下，协程如果加锁不成功不会立即转入阻塞排队，而是判断是否满足自旋的条件，如果满足则会启动自旋过程，尝试抢锁。

starvation模式：自旋过程中能抢到锁，一定意味着同一时刻有协程释放了锁，我们知道释放锁时如果发现有阻塞等待的协程，还会释放一个信号量来唤醒一个等待协程，被唤醒的协程得到CPU后开始运行，此时发现锁已被抢占了，自己只好再次阻塞，不过阻塞前会判断自上次阻塞到本次阻塞经过了多长时间，如果超过1ms的话，会将Mutex标记为”饥饿”模式，然后再阻塞。处于饥饿模式下，不会启动自旋过程，也即一旦有协程释放了锁，那么一定会唤醒协程，被唤醒的协程将会成功获取锁，同时也会把等待计数减1。

**Woken状态：**

Woken状态用于加锁和解锁过程的通信，举个例子，同一时刻，两个协程一个在加锁，一个在解锁，在加锁的协程可能在自旋过程中，此时把Woken标记为1，用于通知解锁协程不必释放信号量了，好比在说：你只管解锁好了，不必释放信号量，我马上就拿到锁了。

使用defer避免死锁，加锁后立即使用defer对其解锁，可以有效的避免死锁。加锁和解锁最好出现在同一个层次的代码块中，比如同一个函数。重复解锁会引起panic，应避免这种操作的可能性。

### RWMutex

前面我们聊了互斥锁Mutex，所谓读写锁RWMutex，完整的表述应该是读写互斥锁，可以说是Mutex的一个改进版，在某些场景下可以发挥更加灵活的控制能力，比如：读取数据频率远远大于写数据频率的场景。

例如，程序中写操作少而读操作多，简单的说，如果执行过程是1次写然后N次读的话，使用Mutex，这个过程将是串行的，因为即便N次读操作互相之间并不影响，但也都需要持有Mutex后才可以操作。如果使用读写锁，多个读操作可以同时持有锁，并发能力将大大提升。

实现读写锁需要解决如下几个问题：

1. 写锁需要阻塞写锁：一个协程拥有写锁时，其他协程写锁定需要阻塞
2. 写锁需要阻塞读锁：一个协程拥有写锁时，其他协程读锁定需要阻塞
3. 读锁需要阻塞写锁：一个协程拥有读锁时，其他协程写锁定需要阻塞
4. 读锁不能阻塞读锁：一个协程拥有读锁时，其他协程也可以拥有读锁

**读写锁数据结构**

```go
type RWMutex struct {
    w           Mutex  //用于控制多个写锁，获得写锁首先要获取该锁，如果有一个写锁在进行，那么再到来的写锁将会阻塞于此
    writerSem   uint32 //写阻塞等待的信号量，最后一个读者释放锁时会释放信号量
    readerSem   uint32 //读阻塞的协程等待的信号量，持有写锁的协程释放锁后会释放信号量
    readerCount int32  //记录读者个数
    readerWait  int32  //记录写阻塞时读者个数
}
```

由以上数据结构可见，读写锁内部仍有一个互斥锁，用于将两个写操作隔离开来，其他的几个都用于隔离读操作和写操作。

### WaitGroup

其数据结构：

```
type WaitGroup struct {
    state1 [3]uint32
}
```

state1是个长度为3的数组，其中包含了state和一个信号量，而state实际上是两个计数器：

- counter： 当前还未执行结束的goroutine计数器
- waiter count: 等待goroutine-group结束的goroutine数量，即有多少个等候者
- semaphore: 信号量

信号量是Unix系统提供的一种保护共享资源的机制，用于防止多个线程同时访问某个资源。可简单理解为信号量为一个数值：

- 当信号量>0时，表示资源可用，获取信号量时系统自动将信号量减1；
- 当信号量==0时，表示资源暂不可用，获取信号量时，当前线程会进入睡眠，当信号量为正时被唤醒；

考虑到字节是否对齐，三者出现的位置不同，为简单起见，依照字节已对齐情况下，三者在内存中的位置如下所示：

![null](http://wen.topgoer.com/uploads/gozhuanjia/images/m_b68f98a52c940a8c94a7c39f1f56a901_r.png)

WaitGroup对外提供三个接口：

- Add(delta int): 将delta值加到counter中
- Wait()： waiter递增1，并阻塞等待信号量semaphore
- Done()： counter递减1，按照waiter数值释放相应次数信号量

```go
var wg sync.WaitGroup

func hello(i int) {
    defer wg.Done() // goroutine结束就登记-1
    fmt.Println("Hello Goroutine!", i)
}
func main() {
    for i := 0; i < 10; i++ {
        wg.Add(1) // 启动一个goroutine就登记+1
        go hello(i)
    }
    wg.Wait() // 等待所有登记的goroutine都结束
}
```

### Context

> 引用：https://www.jianshu.com/p/6def5063c1eb

Golang context是Golang应用开发常用的并发控制技术，它与WaitGroup最大的不同点是context对于派生goroutine有更强的控制力，它可以控制多级的goroutine。context翻译成中文是”上下文”，即它可以控制一组呈树状结构的goroutine，每个goroutine拥有相同的上下文。context用于控制goroutine的生命周期。当一个计算任务被goroutine承接了之后，由于某种原因（超时，或者强制退出）我们希望中止这个goroutine的计算任务，那么就用得到这个Context了。 

context实际上只定义了接口，凡是实现该接口的类都可称为是一种context，官方包中实现了几个常用的context，分别可用于不同的场景。 Context仅仅是一个接口定义，根据实现的不同，可以衍生出不同的context类型：

- cancelCtx实现了Context接口，通过WithCancel()创建cancelCtx实例；
- timerCtx实现了Context接口，通过WithDeadline()和WithTimeout()创建timerCtx实例；
- valueCtx实现了Context接口，通过WithValue()创建valueCtx实例；

应用场景：

- 请求的取消，比如一个协程失败强制退出另外几个协程；
- 请求的超时
- 在http请求中传递上下文

### Once

单例模式。。。

### Pool

sync Pool是用来保存和复用临时对象，以减少内存分配，降低CG压力。，里面的对象不是固定的。sync.Pool可以安全被多个线程同时使用，保证线程安全。sync.Pool中保存的任何项都可能随时不做通知的释放掉，所以不适合用于像socket长连接或数据库连接池。sync.Pool主要用途是增加临时对象的重用率，减少GC负担。

```go
func (s *Student) String() string {
   return s.name
}

func testPool() {
   studentPool := sync.Pool{
      New: func() interface{} {
         return &Student{"abc"}
      }}

   for i:=0;i<100000;i++{
      stud := studentPool.Get().(*Student)
      fmt.Printf("%p %v\n", stud, stud)
   }
}
//对比
type C1 struct {
	B1 [10000000]int
}

func usePool() {
	pool := sync.Pool{New:
	func() interface{} {
		return new(C1)
	}}
	startTime := time.Now()
	for i := 0; i < 10000; i++ {
		c := pool.Get().(*C1)
		c.B1[0] = 1
		pool.Put(c)//需要加上
	}
	fmt.Println("Used time : ", time.Since(startTime))
}

func standard() {
	startTime := time.Now()
	for i := 0; i < 10000; i++ {
		var c C1
		c.B1[0] = 1
	}
	fmt.Println("Used time : ", time.Since(startTime))
}
//standard Used time :  2m36.8892607s
//usePool Used time :  70.8105ms
```

## Atomic

原子包。。。

# 调度模型

> https://blog.csdn.net/chushoufengli/article/details/114940228
>
> https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/

Go 语言的调度器通过使用与 CPU 数量相等的线程减少线程频繁切换的内存开销，同时在每一个线程上执行额外开销更低的 Goroutine 来降低操作系统和硬件的负载。 运行时 G-M-P 模型中引入的处理器 P 是线程和 Goroutine 的中间层 ，P的任务队列。

## GMP模型

![img](https://img-blog.csdnimg.cn/20210317174330298.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NodXNob3VmZW5nbGk=,size_16,color_FFFFFF,t_70)

- G — 表示 Goroutine，它是一个待执行的任务；
- M — 表示操作系统的线程，它由操作系统的调度器调度和管理；
- P — 表示处理器，它可以被看做运行在线程上的本地调度器；

**Goroutine**

Goroutine 可能处于以下 9 种状态：

| 状态          | 描述                                                         |
| ------------- | ------------------------------------------------------------ |
| `_Gidle`      | 刚刚被分配并且还没有被初始化                                 |
| `_Grunnable`  | 没有执行代码，没有栈的所有权，存储在运行队列中               |
| `_Grunning`   | 可以执行代码，拥有栈的所有权，被赋予了内核线程 M 和处理器 P  |
| `_Gsyscall`   | 正在执行系统调用，拥有栈的所有权，没有执行用户代码，被赋予了内核线程 M 但是不在运行队列上 |
| `_Gwaiting`   | 由于运行时而被阻塞，没有执行用户代码并且不在运行队列上，但是可能存在于 Channel 的等待队列上 |
| `_Gdead`      | 没有被使用，没有执行代码，可能有分配的栈                     |
| `_Gcopystack` | 栈正在被拷贝，没有执行代码，不在运行队列上                   |
| `_Gpreempted` | 由于抢占而被阻塞，没有执行用户代码并且不在运行队列上，等待唤醒 |
| `_Gscan`      | GC 正在扫描栈空间，没有执行代码，可以与其他状态同时存在      |

可以将这些不同的状态聚合成三种：等待中、可运行、运行中，运行期间会在这三种状态来回切换：

- 等待中：Goroutine 正在等待某些条件满足，例如：系统调用结束等，包括 `_Gwaiting`、`_Gsyscall` 和 `_Gpreempted` 几个状态；
- 可运行：Goroutine 已经准备就绪，可以在线程运行，如果当前程序中有非常多的 Goroutine，每个 Goroutine 就可能会等待更多的时间，即 `_Grunnable`；
- 运行中：Goroutine 正在某个线程上运行，即 `_Grunning`；

**M**

Go 语言并发模型中的 M 是操作系统线程。调度器最多可以创建 10000 个线程，但是其中大多数的线程都不会执行用户代码（可能陷入系统调用），最多只会有 GOMAXPROCS 个活跃线程能够正常运行。 

在默认情况下，一个四核机器会创建四个活跃的操作系统线程，每一个线程都对应一个运行时中的 runtime.m 结构体。 Go 的默认设置，也就是线程数等于 CPU 数，默认的设置不会频繁触发操作系统的线程调度和上下文切换，所有的调度都会发生在用户态，由 Go 语言调度器触发，能够减少很多额外开销。 

。。。

**P处理器**

调度器中的处理器 P 是线程和 Goroutine 的中间层，它能提供线程需要的上下文环境，也会负责调度线程上的等待队列，通过处理器 P 的调度，每一个内核线程都能够执行多个 Goroutine，它能在 Goroutine 进行一些 I/O 操作时及时让出计算资源，提高线程的利用率。因为调度器在启动时就会创建 GOMAXPROCS 个处理器，所以 Go 语言程序的处理器数量一定会等于 GOMAXPROCS，这些处理器会绑定到不同的内核线程上。

runtime.p 是处理器的运行时表示，作为调度器的内部实现，它包含的字段也非常多，其中包括与性能追踪、垃圾回收和计时器相关的字段，主要关注处理器中的线程和运行队列：

```go
type p struct {
	m           muintptr
	runqhead uint32
	runqtail uint32
	runq     [256]guintptr//处理器持有的运行队列
	runnext guintptr
	...
}
```

 处理器的状态如下：

| 状态        | 描述                                                         |
| ----------- | ------------------------------------------------------------ |
| `_Pidle`    | 处理器没有运行用户代码或者调度器，被空闲队列或者改变其状态的结构持有，运行队列为空 |
| `_Prunning` | 被线程 M 持有，并且正在执行用户代码或者调度器                |
| `_Psyscall` | 没有执行用户代码，当前线程陷入系统调用                       |
| `_Pgcstop`  | 被线程 M 持有，当前处理器由于垃圾回收被停止                  |
| `_Pdead`    | 当前处理器已经不被使用                                       |

通过分析处理器 P 的状态，我们能够对处理器的工作过程有一些简单理解，例如处理器在执行用户代码时会处于 `_Prunning` 状态，在当前线程执行 I/O 操作时会陷入 `_Psyscall` 状态。

## 调度原理

1、调度器初始化

在调度器初始函数执行的过程中会将 maxmcount 设置成 10000，这也就是一个 Go 语言程序能够创建的最大线程数，虽然最多可以创建 10000 个线程，但是可以同时运行的线程还是由 GOMAXPROCS 变量控制。我们从环境变量 GOMAXPROCS 获取了程序能够同时运行的最大处理器数之后就会调用 runtime.procresize 更新程序中处理器的数量，在这时整个程序不会执行任何用户 Goroutine，调度器也会进入锁定状态，runtime.procresize 的执行过程如下：

1. 如果全局变量 allp 切片中的处理器数量少于期望数量，会对切片进行扩容；
2. 使用 new 创建新的处理器结构体并调用 runtime.p.init 初始化刚刚扩容的处理器；
3. 通过指针将线程 m0 和处理器 allp[0] 绑定到一起；
4. 调用 runtime.p.destroy 释放不再使用的处理器结构；
5. 通过截断改变全局变量 allp 的长度保证与期望处理器数量相等；
6. 将除 allp[0] 之外的处理器 P 全部设置成 _Pidle 并加入到全局的空闲队列中；

2、创建goroutine

获取 Goroutine 结构体的三种方法

1. 当处理器的 Goroutine 列表为空时，会将调度器持有的空闲 Goroutine 转移到当前处理器上，直到 gFree 列表中的 Goroutine 数量达到 32；
2. 当处理器的 Goroutine 数量充足时，会从列表头部返回一个新的 Goroutine；
3. 当调度器的 gFree 和处理器的 gFree 列表都不存在结构体时，运行时会调用 runtime.malg 初始化新的 runtime.g 结构

将 Goroutine 放到运行队列上，这既可能是全局的运行队列，也可能是处理器本地的运行队列。Go 语言有两个运行队列，其中一个是处理器本地的运行队列，另一个是调度器持有的全局运行队列，只有在本地运行队列没有剩余空间时才会使用全局队列。 

3、调度循环

调度器启动之后，Go 语言运行时会调用 runtime.mstart 以及 runtime.mstart1，前者会初始化 g0 的 stackguard0 和 stackguard1 字段，后者会初始化线程并调用 runtime.schedule 进入调度循环。获取可运行的 Goroutine：

1. 从本地运行队列、全局运行队列中查找；
2. 从网络轮询器中查找是否有 Goroutine 等待运行；
3. 通过 runtime.runqsteal 尝试从其他随机的处理器中窃取待运行的 Goroutine，该函数还可能窃取处理器的计时器；

Go 语言中的运行时调度循环会从 runtime.schedule 开始，最终又回到 runtime.schedule，我们可以认为调度循环永远都不会返回。 

4、触发调度

运行时触发调度的几个路径：

- 主动挂起 — runtime.gopark -> runtime.park_m 

  该函数会将当前 Goroutine 暂停，被暂停的任务不会放回运行队列 

- 系统调用 — [`runtime.exitsyscall`](https://draveness.me/golang/tree/runtime.exitsyscall) -> [`runtime.exitsyscall0`](https://draveness.me/golang/tree/runtime.exitsyscall0)

- 协作式调度 — runtime.Gosched -> runtime.gosched_m -> runtime.goschedImpl

  runtime.Gosched 函数会主动让出处理器，允许其他 Goroutine 运行。该函数无法挂起 Goroutine，调度器会在可能会将当前 Goroutine 调度到其他线程上 

- 系统监控 — runtime.sysmon -> runtime.retake -> runtime.preemptone

5、线程管理



**调度器的策略**

![img](https://img-blog.csdnimg.cn/20210317174858449.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NodXNob3VmZW5nbGk=,size_16,color_FFFFFF,t_70)

- 全局队列（Global Queue）：存放等待运行的 G。
- P 的本地队列：同全局队列类似，存放的也是等待运行的 G，存的数量有限，不超过 256 个。新建 G’时，G’优先加入到 P 的本地队列，如果队列满了，则会把本地队列中一半的 G 移动到全局队列。
- P 列表：所有的 P 都在程序启动时创建，并保存在数组中，最多有 GOMAXPROCS(可配置) 个。
- M：线程想运行任务就得获取 P，从 P 的本地队列获取 G，P 队列为空时，M 也会尝试从全局队列拿一批 G 放到 P 的本地队列，或从其他 P 的本地队列偷一半放到自己 P 的本地队列。M 运行 G，G 执行之后，M 会从 P 获取下一个 G，不断重复下去。

**1. 复用线程**：避免频繁的创建、销毁线程，而是对线程的复用。

1）work stealing 机制。当本线程无可运行的 G 时，尝试从其他线程绑定的 P 偷取 G，而不是销毁线程。

2）hand off 机制。当本线程因为 G 进行系统调用阻塞时，线程释放绑定的 P，把 P 转移给其他空闲的线程执行。

**2. 利用并行**：GOMAXPROCS 设置 P 的数量，最多有 GOMAXPROCS 个线程分布在多个 CPU 上同时运行。GOMAXPROCS 也限制了并发的程度，比如 GOMAXPROCS = 核数/2，则最多利用了一半的 CPU 核进行并行。

**3. 抢占**：在 coroutine 中要等待一个协程主动让出 CPU 才执行下一个协程，在 Go 中，一个 goroutine 最多占用 CPU 10ms，防止其他 goroutine 被饿死，这就是 goroutine 不同于 coroutine 的一个地方。

**4. 全局 G 队列**：在新的调度器中依然有全局 G 队列，但功能已经被弱化了，当 M 执行 work stealing 从其他 P 偷不到 G 时，它可以从全局 G 队列获取 G。

**调度流程**

一个go func ()的调度流程如下：

![img](https://img-blog.csdnimg.cn/20210317175548606.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2NodXNob3VmZW5nbGk=,size_16,color_FFFFFF,t_70)

 1、我们通过 go func () 来创建一个 goroutine；

 2、有两个存储 G 的队列，一个是局部调度器 P 的本地队列、一个是全局 G 队列。新创建的 G 会先保存在 P 的本地队列中，如果 P 的本地队列已经满了就会保存在全局的队列中；

 3、G 只能运行在 M 中，一个 M 必须持有一个 P，M 与 P 是 1：1 的关系。M 会从 P 的本地队列弹出一个可执行状态的 G 来执行，如果 P 的本地队列为空，就会想其他的 MP 组合偷取一个可执行的 G 来执行；

 4、一个 M 调度 G 执行的过程是一个循环机制；

 5、当 M 执行某一个 G 时候如果发生了 syscall 或则其余阻塞操作，M 会阻塞，如果当前有一些 G 在执行，runtime 会把这个线程 M 从 P 中摘除 (detach)，然后再创建一个新的内核线程 (如果有空闲的线程可用就复用空闲线程) 来服务于这个 P；

 6、当 M 系统调用结束时候，这个 G 会尝试获取一个空闲的 P 执行，并放入到这个 P 的本地队列。如果获取不到 P，那么这个线程 M 变成休眠状态， 加入到空闲线程中，然后这个 G 会被放入全局队列中。

# 网络轮询器

大部分的服务都是 I/O 密集型的，应用程序会花费大量时间等待 I/O 操作的完成。网络轮询器是 Go 语言运行时用来处理 I/O 操作的关键组件，它使用了操作系统提供的 I/O 多路复用机制增强程序的并发处理能力。 

**IO模型**

1、阻塞 I/O 

阻塞 I/O 是最常见的 I/O 模型，在默认情况下，当我们通过 read 或者 write 等系统调用读写文件或者网络时，应用程序会被阻塞。当我们执行 read 系统调用时，应用程序会从用户态陷入内核态，内核会检查文件描述符是否可读；当文件描述符中存在数据时，操作系统内核会将准备好的数据拷贝给应用程序并交回控制权。

2、非阻塞 I/O

当进程把一个文件描述符设置成非阻塞时，执行 read 和 write 等 I/O 操作会立刻返回。当我们将文件描述符修改成非阻塞后，读写文件会经历以下流程：第一次从文件描述符中读取数据会触发系统调用并返回 EAGAIN 错误，EAGAIN 意味着该文件描述符还在等待缓冲区中的数据；随后，应用程序会不断轮询调用 read 直到它的返回值大于 0，这时应用程序就可以对读取操作系统缓冲区中的数据并进行操作。进程使用非阻塞的 I/O 操作时，可以在等待过程中执行其他任务，提高 CPU 的利用率。

3、I/O 多路复用 

I/O 多路复用被用来处理同一个事件循环中的多个 I/O 事件。I/O 多路复用需要使用特定的系统调用，最常见的系统调用是 select，该函数可以同时监听最多 1024 个文件描述符例如socket的可读或者可写状态。除了标准的 select 之外，操作系统中还提供了一个比较相似的 poll 函数，它使用链表存储文件描述符，摆脱了 1024 的数量上限。多路复用函数会阻塞的监听一组文件描述符，当文件描述符的状态转变为可读或者可写时，select 会返回可读或者可写事件的个数，应用程序可以在输入的文件描述符中查找哪些可读或者可写，然后执行相应的操作。I/O 多路复用模型是效率较高的 I/O 模型，它可以同时阻塞监听了一组文件描述符的状态。很多高性能的服务和应用程序都会使用这一模型来处理 I/O 操作，例如：Redis 和 Nginx 等。

**网络轮询器模型**

Go 语言在网络轮询器中使用 I/O 多路复用模型处理 I/O 操作，但是他没有选择最常见的系统调用 select2。虽然 select 也可以提供 I/O 多路复用的能力，但是使用它有比较多的限制：

- 监听能力有限 — 最多只能监听 1024 个文件描述符；
- 内存拷贝开销大 — 需要维护一个较大的数据结构存储文件描述符，该结构需要拷贝到内核中；
- 时间复杂度 O(n)O(n) — 返回准备就绪的事件个数后，需要遍历所有的文件描述符；

为了提高 I/O 多路复用的性能，不同的操作系统也都实现了自己的 I/O 多路复用函数，例如：epoll、kqueue 和 evport 等。Go 语言为了提高在不同操作系统上的 I/O 操作性能，使用平台特定的函数实现了多个版本的网络轮询模块。这些模块在不同平台上实现了相同的功能，构成了一个常见的树形结构。编译器在编译 Go 语言程序时，会根据目标平台选择树中特定的分支进行编译。如果目标平台是 Linux，那么就会根据文件中的 // +build linux 编译指令选择 src/runtime/netpoll_epoll.go 并使用 epoll 函数处理用户的 I/O 操作。 

**总结**

网络轮询器实际上是对 I/O 多路复用技术的封装。

运行时的调度器和系统调用都会通过 runtime.netpoll 与网络轮询器交换消息，获取待执行的 Goroutine 列表，并将待执行的 Goroutine 加入运行队列等待处理。所有的文件 I/O、网络 I/O 和计时器都是由网络轮询器管理的，它是 Go 语言运行时重要的组件。

# 系统监控

Go 语言的系统监控也起到了很重要的作用，它在内部启动了一个不会中止的循环，在循环的内部会轮询网络、抢占长期运行或者处于系统调用的 Goroutine 以及触发垃圾回收，通过这些行为，它能够让系统的运行状态变得更健康。 

运行时通过系统监控来触发线程的抢占、网络的轮询和垃圾回收，保证 Go 语言运行时的可用性。系统监控能够很好地解决尾延迟的问题，减少调度器调度 Goroutine 的饥饿问题并保证计时器在尽可能准确的时间触发。 

。。。

# 内存管理

## 内存分配

Golang中也实现了内存分配器，原理与tcmalloc类似，简单的说就是维护一块大的全局内存，每个线程(Golang中为P)维护一块小的私有内存，私有内存不足再从全局申请。

Golang程序启动时会向系统申请的内存如下图所示：

![null](http://wen.topgoer.com/uploads/gozhuanjia/images/m_8aa79b32715f0ebe6e9618edd9d98383_r.png)

预申请的内存划分为**spans、bitmap、arena三部分。其中arena即为所谓的堆区，应用中需要的内存从这里分配。其中spans和bitmap是为了管理arena区而存在的。**

arena的大小为512G，为了方便管理把arena区域划分成一个个的page，每个page为8KB,一共有512GB/8KB个页；

spans区域存放span的指针，每个指针对应一个或多个page，所以span区域的大小为(512GB/8KB)*指针大小8byte = 512M

bitmap区域大小也是通过arena计算出来，不过主要用于GC。

以申请size为n的内存为例，分配步骤如下：

1. 获取当前线程的私有缓存mcache
2. 根据size计算出适合的class的ID
3. 从mcache的alloc[class]链表中查询可用的span
4. 如果mcache没有可用的span则从mcentral申请一个新的span加入mcache中
5. 如果mcentral中也没有可用的span则从mheap中申请一个新的span加入mcentral
6. 从该span中获取到空闲对象地址并返回

Golang内存分配是个相当复杂的过程，其中还掺杂了GC的处理，这里仅仅对其关键数据结构进行了说明，了解其原理而又不至于深陷实现细节。

1. Golang程序启动时申请一大块内存，并划分成spans、bitmap、arena区域
2. arena区域按页划分成一个个小块
3. span管理一个或多个页
4. mcentral管理多个span供线程申请使用
5. mcache作为线程私有资源，资源来源于mcentral

**内存管理组件**

Go 语言的内存分配器包含内存管理单元、线程缓存、中心缓存和页堆几个重要组件 。这几种最重要组件对应的数据结构 runtime.mspan、runtime.mcache、runtime.mcentral 和 runtime.mheap。

![go-memory-layout](https://img.draveness.me/2020-02-29-15829868066479-go-memory-layout.png) 

所有的 Go 语言程序都会在启动时初始化如上图所示的内存布局，每一个处理器都会分配一个线程缓存 runtime.mcache 用于处理微对象和小对象的分配，它们会持有内存管理单元 runtime.mspan。

每个类型的内存管理单元都会管理特定大小的对象，当内存管理单元中不存在空闲对象时，它们会从 runtime.mheap 持有的 134 个中心缓存 runtime.mcentral 中获取新的内存单元，中心缓存属于全局的堆结构体 runtime.mheap，它会从操作系统中申请内存。

**内存分配**

堆上所有的对象都会通过调用 runtime.newobject 函数分配内存，该函数会调用 runtime.mallocgc 分配指定大小的内存空间，这也是用户程序向堆上申请内存空间的必经函数。 runtime.mallocgc 会根据对象的大小执行不同的分配逻辑，运行时根据对象大小将它们分成微对象、小对象和大对象，这里会根据大小选择不同的分配逻辑。三种对象：

- 微对象 `(0, 16B)` — 先使用微型分配器，再依次尝试线程缓存、中心缓存和堆分配内存；
- 小对象 `[16B, 32KB]` — 依次尝试使用线程缓存、中心缓存和堆分配内存；
- 大对象 `(32KB, +∞)` — 直接在堆上分配内存；

1、微对象

Go 语言运行时将小于 16 字节的对象划分为微对象，它会使用线程缓存上的微分配器提高微对象分配的性能，我们主要使用它来分配较小的字符串以及逃逸的临时变量。微分配器可以将多个较小的内存分配请求合入同一个内存块中，只有当内存块中的所有对象都需要被回收时，整片内存才可能被回收。微分配器管理的对象不可以是指针类型，管理多个对象的内存块大小 maxTinySize 是可以调整的，在默认情况下，内存块的大小为 16 字节。maxTinySize 的值越大，组合多个对象的可能性就越高，内存浪费也就越严重；maxTinySize 越小，内存浪费就会越少，不过无论如何调整，8 的倍数都是一个很好的选择。

2、小对象 

小对象是指大小为 16 字节到 32,768 字节的对象以及所有小于 16 字节的指针类型的对象，小对象的分配可以被分成以下的三个步骤：

1. 确定分配对象的大小以及跨度类 runtime.spanClass；
2. 从线程缓存、中心缓存或者堆中获取内存管理单元并从内存管理单元找到空闲的内存空间；
3. 调用 runtime.memclrNoHeapPointers 清空空闲内存中的所有数据；

3、 大对象 

运行时对于大于 32KB 的大对象会单独处理，我们不会从线程缓存或者中心缓存中获取内存管理单元，而是直接调用 runtime.mcache.allocLarge 分配大片内存。runtime.mcache.allocLarge 会计算分配该对象所需要的页数，它按照 8KB 的倍数在堆上申请内存申请内存时会创建一个跨度类为 0 的 runtime.spanClass 并调用 runtime.mheap.alloc 分配一个管理对应内存的管理单元。

## GC

**标记清除**

最常见的垃圾回收算法有标记清除(Mark-Sweep) 和引用计数(Reference Count)，Go 语言采用的是标记清除算法。并在此基础上使用了三色标记法和写屏障技术、增量和并发，提高了效率。

标记清除收集器是跟踪式垃圾收集器，其执行过程可以分成标记（Mark）和清除（Sweep）两个阶段：

- 标记阶段 — 从根对象出发查找并标记堆中所有存活的对象；
- 清除阶段 — 遍历堆中的全部对象，回收未被标记的垃圾对象并将回收的内存加入空闲链表。

**三色抽象**

为了解决原始标记清除算法带来的长时间 STW，多数现代的追踪式垃圾收集器都会实现三色标记算法的变种以缩短 STW 的时间。三色标记算法将程序中的对象分成白色、黑色和灰色三类：

- 白色对象 — 潜在的垃圾，其内存可能会被垃圾收集器回收；
- 黑色对象 — 活跃的对象，包括不存在任何引用外部指针的对象以及从根对象可达的对象；
- 灰色对象 — 活跃的对象，因为存在指向白色对象的外部指针，垃圾收集器会扫描这些对象的子对象；

![tri-color-objects](https://img.draveness.me/2020-03-16-15843705141808-tri-color-objects.png)

三色标记垃圾收集器的工作原理步骤：

1. 从灰色对象的集合中选择一个灰色对象并将其标记成黑色；
2. 将黑色对象指向的所有对象都标记成灰色，保证该对象和被该对象引用的对象都不会被回收；
3. 重复上述两个步骤直到对象图中不存在灰色对象；

当三色的标记清除的标记阶段结束之后，应用程序的堆中就不存在任何的灰色对象，我们只能看到黑色的存活对象以及白色的垃圾对象，垃圾收集器可以回收这些白色的垃圾，使用三色标记垃圾收集器执行标记后的堆内存，堆中只有对象 D 为待回收的垃圾：

**屏障技术**

想要并发或者增量地标记对象还是需要使用屏障技术。内存屏障技术是一种屏障指令，它可以让 CPU 或者编译器在执行内存相关操作时遵循特定的约束，目前多数的现代处理器都会乱序执行指令以最大化性能，但是该技术能够保证内存操作的顺序性，在内存屏障前执行的操作一定会先于内存屏障后执行的操作。想要在并发或者增量的标记算法中保证正确性，我们需要达成以下两种三色不变性（Tri-color invariant）中的一种：

- 强三色不变性 — 黑色对象不会指向白色对象，只会指向灰色对象或者黑色对象；
- 弱三色不变性 — 黑色对象指向的白色对象必须包含一条从灰色对象经由多个白色对象的可达路径；

**增量和并发**

增量垃圾收集 — 增量地标记和清除垃圾，降低应用程序暂停的最长时间；并发垃圾收集 — 利用多核的计算资源，在用户程序执行时并发标记和清除垃圾；

因为增量和并发两种方式都可以与用户程序交替运行，所以我们需要**使用屏障技术**保证垃圾收集的正确性；与此同时，应用程序也不能等到内存溢出时触发垃圾收集，因为当内存不足时，应用程序已经无法分配内存，这与直接暂停程序没有什么区别，增量和并发的垃圾收集需要提前触发并在内存不足前完成整个循环，避免程序的长时间暂停。

增量式的垃圾收集需要与三色标记法一起使用，为了保证垃圾收集的正确性，我们需要在垃圾收集开始前打开写屏障，这样用户程序修改内存都会先经过写屏障的处理，保证了堆内存中对象关系的强三色不变性或者弱三色不变性。虽然增量式的垃圾收集能够减少最大的程序暂停时间，但是增量式收集也会增加一次 GC 循环的总时间，在垃圾收集期间，因为写屏障的影响用户程序也需要承担额外的计算开销，所以增量式的垃圾收集也不是只带来好处的，但是总体来说还是利大于弊 。

并发（Concurrent）的垃圾收集不仅能够减少程序的最长暂停时间，还能减少整个垃圾收集阶段的时间，通过开启读写屏障、**利用多核优势与用户程序并行执行**，并发垃圾收集器确实能够减少垃圾收集对应用程序的影响。

**GC的阶段**

1、清理终止阶段；

- 暂停程序，所有的处理器在这时会进入安全点（Safe point）；
- 如果当前垃圾收集循环是强制触发的，我们还需要处理还未被清理的内存管理单元；

2、标记阶段；

- 将状态切换至 _GCmark、开启写屏障、用户程序协助（Mutator Assiste）并将根对象入队；
- 恢复执行程序，标记进程和用于协助的用户程序会开始并发标记内存中的对象，写屏障会将被覆盖的指针和新指针都标记成灰色，而所有新创建的对象都会被直接标记成黑色；
- 开始扫描根对象，包括所有 Goroutine 的栈、全局对象以及不在堆中的运行时数据结构，扫描 Goroutine 栈期间会暂停当前处理器；
- 依次处理灰色队列中的对象，将对象标记成黑色并将它们指向的对象标记成灰色；
- 使用分布式的终止算法检查剩余的工作，发现标记阶段完成后进入标记终止阶段；

3、标记终止阶段；

- 暂停程序、将状态切换至 _GCmarktermination 并关闭辅助标记的用户程序；
- 清理处理器上的线程缓存；

4、清理阶段；

- 将状态切换至 _GCoff 开始清理阶段，初始化清理状态并关闭写屏障；
- 恢复用户程序，所有新创建的对象会标记成白色；
- 后台并发清理所有的内存管理单元，当 Goroutine 申请新的内存管理单元时就会触发清理；

**GC的触发**

1、后台运行的系统监控器

运行时会在应用程序启动时在后台开启一个用于强制触发垃圾收集的 Goroutine，该 Goroutine 的职责非常简单 — 调用 [`runtime.gcStart`](https://draveness.me/golang/tree/runtime.gcStart) 尝试启动新一轮的垃圾收集。

为了减少对计算资源的占用，该 Goroutine 会在循环中调用 runtime.goparkunlock 主动陷入休眠等待其他 Goroutine 的唤醒，runtime.forcegchelper 在大多数时间都是陷入休眠的，但是它会被系统监控器 runtime.sysmon 在满足垃圾收集条件时唤醒。

系统监控在每个循环中都会主动构建一个 runtime.gcTrigger 并检查垃圾收集的触发条件是否满足，如果满足条件，系统监控会将 runtime.forcegc 状态中持有的 Goroutine 加入全局队列等待调度器的调度。 

2、手动强制触发垃圾收集 

用户程序会通过 runtime.GC 函数在程序运行期间主动通知运行时执行，该方法在调用时会阻塞调用方直到当前垃圾收集循环完成，在垃圾收集期间也可能会通过 STW 暂停整个程序。手动触发垃圾收集的过程不是特别常见，一般只会在运行时的测试代码中才会出现。

3、申请内存

最后一个可能会触发垃圾收集的就是 runtime.mallocgc 了，运行时会将堆上的对象按大小分成微对象、小对象和大对象三类，这三类对象的创建都可能会触发新的垃圾收集循环。

**内存清理**

垃圾收集的清理中包含对象回收器（Reclaimer）和内存单元回收器，这两种回收器使用不同的算法清理堆内存：

- 对象回收器在内存管理单元中查找并释放未被标记的对象，但是如果 runtime.mspan 中的所有对象都没有被标记，整个单元就会被直接回收，该过程会被 runtime.mcentral.cacheSpan 或者 runtime.sweepone 异步触发；
- 内存单元回收器会在内存中查找所有的对象都未被标记的 runtime.mspan，该过程会被 runtime.mheap.reclaim 触发；

## 栈内存

应用程序的内存一般会分成堆区和栈区，程序在运行期间可以主动从堆区申请内存空间，这些内存由内存分配器分配并由垃圾收集器负责回收 。栈区的内存一般由编译器自动分配和释放，其中存储着函数的入参以及局部变量，这些参数会随着函数的创建而创建，函数的返回而消亡，一般不会在程序中长期存在，这种线性的内存分配策略有着极高地效率，但是工程师也往往不能控制栈内存的分配，这部分工作基本都是由编译器完成的。 

**逃逸分析**

所谓逃逸分析（Escape analysis）是指由编译器决定内存分配的位置，不需要程序员指定。函数中申请一个新的对象

- **如果分配在栈中，则函数执行结束可自动将内存回收；**
- **如果分配在堆中，则函数执行结束可交给GC（垃圾回收）处理；**

有了逃逸分析，返回函数局部变量将变得可能，除此之外，逃逸分析还跟闭包息息相关，了解哪些场景下对象会逃逸至关重要。

逃逸策略：

每当函数中申请新的对象，编译器会根据该对象是否被函数外部引用来决定是否逃逸：

1. 如果函数外部没有引用，则优先放到栈中；
2. 如果函数外部存在引用，则必定放到堆中；

注意，对于函数外部没有引用的对象，也有可能放到堆中，比如内存过大超过栈的存储能力。

逃逸场景：函数返回局部变量的指针（安全的）、栈不足、动态参数、闭包引用对象。

## 内存泄露

内存泄露场景：



如何排除定位：

pprof

## 内存对齐

参考：https://blog.csdn.net/Bobdragery/article/details/107791523

。。。

# 源码解读

## http

**客户端**

当我们调用 net/http.Client.Get 发出 HTTP 时，会按照如下的步骤执行：

1. 调用 net/http.NewRequest 根据方法名、URL 和请求体构建请求；
2. 调用 net/http.Transport.RoundTrip 开启 HTTP 事务、获取连接并发送请求；
3. 在 HTTP 持久连接的 net/http.persistConn.readLoop 方法中等待响应；

数据结构

HTTP 的客户端中包含几个比较重要的结构体，它们分别是 net/http.Client、net/http.Transport 和 net/http.persistConn：

- net/http.Client 是 HTTP 客户端，它的默认值是使用 net/http.DefaultTransport 的 HTTP 客户端；
- net/http.Transport 是 net/http.RoundTripper 接口的实现，它的主要作用就是支持 HTTP/HTTPS 请求和 HTTP 代理；
- net/http.persistConn 封装了一个 TCP 的持久连接，是我们与远程交换消息的句柄（Handle）；

客户端 net/http.Client 是级别较高的抽象，它提供了 HTTP 的一些细节，包括 Cookies 和重定向；而 net/http.Transport 会处理 HTTP/HTTPS 协议的底层实现细节，其中会包含连接重用、构建请求以及发送请求等功能。

1、构建请求

net/http.Request 表示 HTTP 服务接收到的请求或者 HTTP 客户端发出的请求，其中包含 HTTP 请求的方法、URL、协议版本、协议头以及请求体等字段，除了这些字段之外，它还会持有一个指向 HTTP 响应的引用：

```go
type Request struct {
	Method string
	URL *url.URL

	Proto      string // "HTTP/1.0"
	ProtoMajor int    // 1
	ProtoMinor int    // 0

	Header Header
	Body io.ReadCloser

	...
	Response *Response
}
```

net/http.NewRequest 是标准库提供的用于创建请求的方法，这个方法会校验 HTTP 请求的字段并根据输入的参数拼装成新的请求结构体。请求拼装的过程比较简单，它会检查并校验输入的方法、URL 以及负载，然而初始化了新的 net/http.Request 结构，处理负载body的过程稍微有一些复杂，我们会根据负载的类型不同，使用不同的方法将它们包装成 io.ReadCloser 类型。

2、建立连接

当我们使用标准库构建了 HTTP 请求之后，会开启 HTTP 事务发送 HTTP 请求并等待远程的响应，经过一连串的调用，我们最终来到了标准库实现底层 HTTP 协议的结构体 — net/http.Transport。net/http.Transport 实现了 net/http.RoundTripper 接口，也是整个请求过程中最重要并且最复杂的结构体，该结构体会在 net/http.Transport.roundTrip 中发送 HTTP 请求并等待响应，我们可以将该函数的执行过程分成两个部分：

- 根据 URL 的协议查找并执行自定义的 net/http.RoundTripper 实现；
- 从连接池中获取或者初始化新的持久连接并调用连接的 net/http.persistConn.roundTrip 发出请求；

在默认情况下，我们都会使用 net/http.persistConn 持久连接处理 HTTP 请求，该方法会先获取用于发送请求的连接，随后调用 net/http.persistConn.roundTrip：

```go
func (t *Transport) roundTrip(req *Request) (*Response, error) {
	...
	for {
		select {
		case <-ctx.Done():
			return nil, ctx.Err()
		default:
		}

		treq := &transportRequest{Request: req, trace: trace}
		cm, err := t.connectMethodForRequest(treq)
		if err != nil {
			return nil, err
		}

		pconn, err := t.getConn(treq, cm)
		if err != nil {
			return nil, err
		}

		resp, err := pconn.roundTrip(treq)
		if err == nil {
			return resp, nil
		}
	}
}
```

net/http.Transport.getConn 是获取连接的方法，该方法会通过两种方法获取用于发送请求的连接：1、调用 net/http.Transport.queueForIdleConn 在队列中等待闲置的连接；2、调用 net/http.Transport.queueForDial 在队列中等待建立新的连接。连接是一种相对比较昂贵的资源，如果在每次发出 HTTP 请求之前都建立新的连接，可能会消耗比较多的时间，带来较大的额外开销，通过连接池对资源进行分配和复用可以有效地提高 HTTP 请求的整体性能，多数的网络库客户端都会采取类似的策略来复用资源。

```go
func (t *Transport) getConn(treq *transportRequest, cm connectMethod) (pc *persistConn, err error) {
	req := treq.Request
	ctx := req.Context()

	w := &wantConn{
		cm:         cm,
		key:        cm.key(),
		ctx:        ctx,
		ready:      make(chan struct{}, 1),
	}

	if delivered := t.queueForIdleConn(w); delivered {
		return w.pc, nil
	}

	t.queueForDial(w)
	select {
	case <-w.ready:
		...
		return w.pc, w.err
	...
	}
}
```

当我们调用 net/http.Transport.queueForDial 尝试与远程建立连接时，标准库会在内部启动新的 Goroutine 执行 net/http.Transport.dialConnFor 用于建连，从最终调用的 net/http.Transport.dialConn 中我们能找到 TCP 连接和 net 库的身影。在创建新的 TCP 连接后，我们还会在后台为当前的连接创建两个 Goroutine，分别从 TCP 连接中读取数据或者向 TCP 连接写入数据

```go
func (t *Transport) dialConn(ctx context.Context, cm connectMethod) (pconn *persistConn, err error) {
	pconn = &persistConn{
		t:             t,
		cacheKey:      cm.key(),
		reqch:         make(chan requestAndChan, 1),
		writech:       make(chan writeRequest, 1),
		closech:       make(chan struct{}),
		writeErrCh:    make(chan error, 1),
		writeLoopDone: make(chan struct{}),
	}

	conn, err := t.dial(ctx, "tcp", cm.addr())
	if err != nil {
		return nil, err
	}
	pconn.conn = conn

	pconn.br = bufio.NewReaderSize(pconn, t.readBufferSize())
	pconn.bw = bufio.NewWriterSize(persistConnWriter{pconn}, t.writeBufferSize())

	go pconn.readLoop() //tcp读数据
	go pconn.writeLoop()//tcp写数据
	return pconn, nil
}
```



3、发送请求等待响应

持久的 TCP 连接会实现 net/http.persistConn.roundTrip 处理写入 HTTP 请求并在 select 语句中等待响应的返回：

```go
func (pc *persistConn) roundTrip(req *transportRequest) (resp *Response, err error) {
	writeErrCh := make(chan error, 1)
	pc.writech <- writeRequest{req, writeErrCh, continueCh}

	resc := make(chan responseAndError)
	pc.reqch <- requestAndChan{
		req:        req.Request,
		ch:         resc,
	}

	for {
		select {
		case re := <-resc:
			if re.err != nil {
				return nil, pc.mapRoundTripError(req, startBytesWritten, re.err)
			}
			return re.res, nil
		...
		}
	}
}
```

持久连接中的读循环 net/http.persistConn.readLoop 会负责从 TCP 连接中读取数据并将数据发送会 HTTP 请求的调用方，真正负责解析 HTTP 协议的还是 net/http.ReadResponse，读循环readLoop 中会调用pc.readResponse(rc, trace)。

```go
func ReadResponse(r *bufio.Reader, req *Request) (*Response, error) {
	tp := textproto.NewReader(r)
	resp := &Response{
		Request: req,
	}

	line, _ := tp.ReadLine()
	if i := strings.IndexByte(line, ' '); i == -1 {
		return nil, badStringError("malformed HTTP response", line)
	} else {
		resp.Proto = line[:i]
		resp.Status = strings.TrimLeft(line[i+1:], " ")
	}

	statusCode := resp.Status
	if i := strings.IndexByte(resp.Status, ' '); i != -1 {
		statusCode = resp.Status[:i]
	}
	resp.StatusCode, err = strconv.Atoi(statusCode)

	resp.ProtoMajor, resp.ProtoMinor, _ = ParseHTTPVersion(resp.Proto)

	mimeHeader, _ := tp.ReadMIMEHeader()
	resp.Header = Header(mimeHeader)

	readTransfer(resp, r)
	return resp, nil
}
```

每个 HTTP 请求都由另一个 Goroutine 中的 net/http.persistConn.writeLoop 循环写入的，这两个 Goroutine 独立执行并通过 Channel 进行通信。net/http.Request.write 会根据 net/http.Request 结构中的字段按照 HTTP 协议组成 TCP 数据段：

```go
func (pc *persistConn) writeLoop() {
	defer close(pc.writeLoopDone)
	for {
		select {
		case wr := <-pc.writech:
			startBytesWritten := pc.nwrite
			wr.req.Request.write(pc.bw, pc.isProxy, wr.req.extra, pc.waitForContinue(wr.continueCh))
			...
		case <-pc.closech:
			return
		}
	}
}
```

当我们调用 net/http.Request.write 向请求中写入数据时，实际上直接写入了 net/http.persistConnWriter 中的 TCP 连接中，TCP 协议栈会负责将 HTTP 请求中的内容发送到目标服务器上：

```go
type persistConnWriter struct {
	pc *persistConn
}

func (w persistConnWriter) Write(p []byte) (n int, err error) {
	n, err = w.pc.conn.Write(p)
	w.pc.nwrite += int64(n)
	return
}
```



**服务端**

简单应用：

```go
func handler(w http.ResponseWriter, r *http.Request) {
   fmt.Fprintf(w, "Hi there, I love %s!", r.URL.Path[1:])
}

func main() {
   http.HandleFunc("/", handler)
   log.Fatal(http.ListenAndServe(":8080", nil))
}
```

1、注册处理器

当我们直接调用 net/http.HandleFunc 注册处理器时，标准库会使用默认的 HTTP 服务器 net/http.DefaultServeMux 处理请求，该方法会直接调用 net/http.ServeMux.HandleFunc，上述方法会将处理器转换成 net/http.Handler 接口类型调用 net/http.ServeMux.Handle 注册处理器：

```go
func (mux *ServeMux) Handle(pattern string, handler Handler) {
	if _, exist := mux.m[pattern]; exist {
		panic("http: multiple registrations for " + pattern)
	}

	e := muxEntry{h: handler, pattern: pattern}
	mux.m[pattern] = e
	if pattern[len(pattern)-1] == '/' {
		mux.es = appendSorted(mux.es, e)
	}

	if pattern[0] != '/' {
		mux.hosts = true
	}
}
```

路由和对应的处理器会被组成 net/http.DefaultServeMux，该结构会持有一个 net/http.muxEntry 哈希，其中存储了从 URL 到处理器的映射关系，HTTP 服务器在处理请求时就会使用该哈希map查找处理器。

2、监听处理请求

net/http.Server.Serve 会在循环中监听外部的 TCP 连接并为每个连接调用 net/http.Server.newConn 创建新的 net/http.conn，它是 HTTP 连接的服务端表示：

```go
func (srv *Server) Serve(l net.Listener) error {
	l = &onceCloseListener{Listener: l}
	defer l.Close()

	baseCtx := context.Background()
	ctx := context.WithValue(baseCtx, ServerContextKey, srv)
	for {
		rw, err := l.Accept()
		if err != nil {
			select {
			case <-srv.getDoneChan():
				return ErrServerClosed
			default:
			}
			...
			return err
		}
		connCtx := ctx
		c := srv.newConn(rw)
		c.setState(c.rwc, StateNew) // before Serve can return
		go c.serve(connCtx)
	}
}
```

创建了服务端的连接之后，标准库中的实现会为每个 HTTP 请求创建单独的 Goroutine 并在其中调用 net/http.Conn.serve 方法，如果当前 HTTP 服务接收到了海量的请求，会在内部创建大量的 Goroutine，这可能会使整个服务质量明显降低无法处理请求。

```go
func (c *conn) serve(ctx context.Context) {
	c.remoteAddr = c.rwc.RemoteAddr().String()

	ctx = context.WithValue(ctx, LocalAddrContextKey, c.rwc.LocalAddr())
	ctx, cancelCtx := context.WithCancel(ctx)
	c.cancelCtx = cancelCtx
	defer cancelCtx()

	c.r = &connReader{conn: c}
	c.bufr = newBufioReader(c.r)
	c.bufw = newBufioWriterSize(checkConnErrorWriter{c}, 4<<10)

	for {
		w, _ := c.readRequest(ctx)
		serverHandler{c.server}.ServeHTTP(w, w.req)
		w.finishRequest()
		...
	}
}
```

上述代码片段是我们简化后的连接处理过程，其中包含读取 HTTP 请求、调用 Handler 处理 HTTP 请求以及调用完成该请求。读取 HTTP 请求会调用 net/http.Conn.readRequest，该方法会从连接中获取 HTTP 请求并构建一个实现了 net/http.ResponseWriter 接口的变量 net/http.response，向该结构体写入的数据都会被转发到它持有的缓冲区中：

```go
func (w *response) write(lenData int, dataB []byte, dataS string) (n int, err error) {
	...
	w.written += int64(lenData)
	if w.contentLength != -1 && w.written > w.contentLength {
		return 0, ErrContentLength
	}
	if dataB != nil {
		return w.w.Write(dataB)
	} else {
		return w.w.WriteString(dataS)
	}
}
```

解析了 HTTP 请求并初始化 net/http.ResponseWriter 之后，我们就可以调用 net/http.serverHandler.ServeHTTP 查找处理器来处理 HTTP 请求了：

```go
type serverHandler struct {
	srv *Server
}

func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
	handler := sh.srv.Handler
	if handler == nil {
		handler = DefaultServeMux
	}
	if req.RequestURI == "*" && req.Method == "OPTIONS" {
		handler = globalOptionsHandler{}
	}
	handler.ServeHTTP(rw, req)
}
```

如果当前的 HTTP 服务器中不包含任何处理器，我们会使用默认的 net/http.DefaultServeMux 处理外部的 HTTP 请求。net/http.ServeMux 是一个 HTTP 请求的多路复用器，它可以接收外部的 HTTP 请求、根据请求的 URL 匹配并调用最合适的处理器：

```go
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
	h, _ := mux.Handler(r)
	h.ServeHTTP(w, r)
}
```

经过一系列的函数调用，上述过程最终会调用 HTTP 服务器的 net/http.ServerMux.match，该方法会遍历前面注册过的路由表并根据特定规则进行匹配：

```go
func (mux *ServeMux) match(path string) (h Handler, pattern string) {
	v, ok := mux.m[path]
	if ok {
		return v.h, v.pattern
	}

	for _, e := range mux.es {
		if strings.HasPrefix(path, e.pattern) {
			return e.h, e.pattern
		}
	}
	return nil, ""
}
```

如果请求的路径和路由中的表项匹配成功，我们会调用表项中对应的处理器，处理器中包含的业务逻辑会通过 net/http.ResponseWriter 构建 HTTP 请求对应的响应并通过 TCP 连接发送回客户端。

## sql

。。。

