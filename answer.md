# Answer

### 1. chan是什么？

**Answer 1:**  

- chan 是 go 语言里面的一个关键字，是 channel 的简写，表示通道。如果说 goroutine 是 Go语言程序的并发体的话，那么 chan 就是 goroutine 用来通信和同步的重要组件。
- 一个 chan 是一个通信机制，它可以让一个 goroutine 通过它给另一个 goroutine 发送值信息。每个 chan 都有一个特殊的类型，也就是 chan 可发送数据的类型。



### 2. 读写已关闭的chan会发生什么事？原因是什么？

**Answer 2:**  

- 读已经关闭的 chan 能一直读到东西，但根据关闭前通道内是否有元素，读到的内容是不一样的。

  - 如果 chan 关闭前，buffer 内有元素还未读,会正确读到 chan 内的值，且返回的第二个 bool 值（是否读成功）为true。
  - 如果 chan 关闭前，buffer 内的元素已经被读完，chan 内无值，接下来所有接收的值都会非阻塞直接成功，返回通道元素类型的零值，但是第二个 bool 值一直为false。

- 向已经关闭的 chan 写数据会导致panic。

  

### 3. 有缓冲通道和无缓冲通道区别？

**Answer 3:**  

- 有缓冲通道和无缓冲通道的主要区别是无缓冲通道要求读写是同步的，否则会阻塞，而有缓冲通道是非同步的。
- 无缓冲通道的特性是要求同时由读写两端把持chan。如果只有读端，没有写端，则读阻塞；如果只有写端，没有读端，则写阻塞，示例如下：

```go
//创建无缓冲通道ch1
ch1 := make(chan int)
ch1 <- 1
//不仅仅是向 ch1 通道放 1 而是一直要有别的协程进行读操作：<-ch1 ，那么ch1<-1之后的代码才会继续执行下去，要不然就一直阻塞着
//没有读端，会阻塞
ch1 <- 2 


//创建有缓冲通道ch2
ch2 := make(chan int, 5)
ch2 <- 1
//缓冲没满，不会阻塞
ch2 <- 2

```



### 4. 以下代码会有什么问题？

```go
a := make(chan int)
a <- 1   //将数据写入channel
z := <-a //从channel中读取数据
```

**Answer 4:**  

```Plain Text
// 产生的问题：

运行时报错：fatal error: all goroutines are asleep - deadlock!

因为通过make(chan int)开辟的通道 a 是无缓冲通道，一次写入或读取就会阻塞，直到相反的操作发生，
此处因为处于一个goroutine中，a<-1写入操作阻塞，程序无法到达下一行的读取操作，
也无法获得其他的读取操作，所以整个程序阻塞，造成死锁。

可以另开一个goroutine,当前的goroutine同步读取来解决
```

正确写法：

```go
package main

import "fmt"

func main() {
	a := make(chan int)

	// 开启一个并发匿名函数
	go func() {
		a <- 1
	}()
	// 等待匿名goroutine
	z := <-a
	fmt.Println(z)
}
```



### 5. 以下代码会有什么问题？

```go
func multipleDeathLock() {
    chanInt := make(chan int)
    defer close(chanInt)
    go func() {
        res := <-chanInt
        fmt.Println(res)
    }()
    chanInt <- 1
    chanInt <- 1
}
```

**Answer 5:**  

```Plain Text
// 产生的问题：
运行时报错：fatal error: all goroutines are asleep - deadlock!

chanInt是无缓冲信道，需要同时有读写两端才不会发生阻塞
第一行的chanInt <- 1运行完成时，可能还没有等到子协程读取，
第二行的chanInt <- 1执行就会造成阻塞。
同时，第二行的chanInt <- 1写入时没有相应的读端，也会产生阻塞

```

正确写法：

```go
func multipleDeathLock() {
	chanInt := make(chan int)
	defer close(chanInt)
	go func() {
		res := <-chanInt
		fmt.Println(res)
		res = <-chanInt
		fmt.Println(res)
	}()
	chanInt <- 1
	chanInt <- 2
}

```



### 6. 以下代码会有什么问题？

```go
func multipleDeathLock2() {
    chanInt := make(chan int)
    defer close(chanInt)
    go func() {
        chanInt <- 1
        chanInt <- 2
    }()
    for {
        res, ok := <-chanInt
        if !ok {
            break
        }
        fmt.Println(res)
    }
}
```

**Answer 6:**  

```Plain Text
// 产生的问题：
运行时报错：fatal error: all goroutines are asleep - deadlock!

主协程的for{}在依次读取子协程传入的两个数据后，会继续读取，
此时chanInt内没有数据，相当于只有读端没有写端，会等待产生阻塞，因此需要保证读写次数一致
```

正确写法：

```go
func multipleDeathLock2() {
	chanInt := make(chan int)
	defer close(chanInt)
	// n是写入和读出的次数
	var n int = 2
	go func() {
		for i := 1; i <= n; i++ {
			chanInt <- i
		}
	}()

	for i := 1; i <= n; i++ {
		res, ok := <-chanInt
		if !ok {
			break
		}
		fmt.Println(res)
	}

}
```



### 7. goroutine是什么？

**Answer 7:**  

- goroutine 是 Go语言的并发执行体，是轻量级的线程实现，由 Go 运行时（runtime）管理，Go 运行时调度程序将 Goroutines 多路复用到线程上，当线程阻塞时，Go 运行时将阻塞的 Goroutines 移动到另一个可运行的内核线程，通过智能地将 goroutine 中的任务合理地分配给每个 CPU，实现了尽可能高的效率。
- Go语言使用go关键字启动一个goroutine 
- goroutine 有如下特性：
  - go的执行是非阻塞的，不会等待
  - go后面的函数的返回值会被忽略
  - 调度器不能保证多个goroutine 的执行次序
  - 没有父子goroutine 的概念，所有的goroutine 是平等的被调度和执行的
  - Go程序在执行时会单独为main函数创建一个goroutine ，遇到其他关键字时再去创建其他的goroutine 。
  - GO没有暴露goroutine id 给用户，所以不能在一个goroutine 里面显式的操作另一个goroutine ，不过可以使用内置包下的一些函数访问和设置相关信息。

### 8. goroutine什么时候会终止？

**Answer 8:**  

1. 当main函数return时，所有的goroutine立即终止，程序退出。
2. 接收到channel的结束通知：这是最主要的goroutine终止方式。goroutine虽然不能强制结束另外一个goroutine，但是它可以通过接受到channel通知后终止。
3. 接收到context的结束通知
4. 程序遇到panic
5. goroutine的程序计算执行结束



### 9. 以下代码会有什么问题？

```go
func goroutineLeak() {
    chanInt := make(chan int)
    defer func() {
        close(chanInt)
        time.Sleep(time.Second)
    }()
    go func() {
        for {
            res := <-chanInt
            fmt.Println(res)
        }
    }()
    chanInt <- 1
    chanInt <- 1
}
```

**Answer 9:**  

```Plain Text
// 产生的问题：
在输出两个1之后输出很多0，这是因为goroutineLeak()执行完之后，defer执行，关闭了chanInt()，
在关闭之后等待了1s，但是匿名函数中goroutine并没有关闭，而是一直在循环取值，
在读取完写入里面的两个1后，会读取到通道的类型零值，在程序停止前，一直输出0。
问题：goroutine会永远运行下去，如果以后再次使用又会出现新的泄漏！导致内存、cpu占用越来越多
解决方法：通过读取时返回的ok判断通道是否已关闭，如果已关闭，退出for{}，停止读取

```

正确写法：

```go
func goroutineLeak() {
	chanInt := make(chan int)
	defer func() {
		close(chanInt)
		time.Sleep(time.Second)
	}()
	go func() {
		for {
			res, ok := <-chanInt
			if !ok {
				break
			}
			fmt.Println(res)
		}
	}()
	chanInt <- 1
	chanInt <- 1
}
```



### 10. 以下代码会有什么问题？

```go
func goroutineLeakNoClosed() {
    chanInt := make(chan int)
    go func() {
        for {
            res := <-chanInt
            fmt.Println(res)
        }
    }()
}
```

**Answer 10:**  

```Plain Text
// 产生的问题：
运行描述：chanInt()在子协程读取，但没有写端，虽然在主协程结束后，子协程也会结束，
可能来不及报deadlock就结束了，因此没有报错。
在goroutineLeakNoClosed()结束前,通道chanInt没有关闭,释放资源

问题：
1、无任何输出的阻塞；
2、换成写入也是一样的
3、如果是有缓冲的通道，换成已满的通道写没有读；或者换成向空的通道读没有写也是同样的情况
4、除了阻塞，goroutine进入死循环也是泄露的原因
```

正确写法：

```go
func goroutineLeakNoClosed() {
	chanInt := make(chan int)
	defer close(chanInt)
	go func() {
		for {
			res := <-chanInt
			fmt.Println(res)
		}
	}()
}
```



### 11. 使用chan实现一个消息队列

**Answer 11:**  

```Plain Text
// 思路
1、使用无缓冲通道同步goruntine
2、带缓冲的chan作为消息队列
3、开启子协程写入队列
4、在主协程读队列
```

实现：

```go
package main

import (
	"fmt"
	"runtime"
)

func main() {
	//无缓冲通道，用来同步
	ch := make(chan struct{})
	//有缓存通道，用作消息队列
	var capacity int = 100
	ci := make(chan int, capacity)
	go func(i chan struct{}, j chan int) {
		for i := 0; i < 10; i++ {
			ci <- i
			fmt.Println("写入队列：", i)
		}
		close(ci)
		ch <- struct{}{}
	}(ch, ci)

	fmt.Println("num of goruntine:", runtime.NumGoroutine())
	<-ch
	fmt.Println("num of goruntine:", runtime.NumGoroutine())

	for v := range ci {
		fmt.Println("读取队列数据：", v)
	}
}
```



### 12. 实现一个goroutine协程池

**Answer 11:**  

主程序：

```go
// 主程序 goroutine_pool.go
// 实现协程池有利于协程的管理，方便建立和销毁
package pool

import (
	"github.com/google/uuid"
	"log"
	"os"
	"sync"
	"sync/atomic"
	"time"
)

type GResult struct {
	id  string
	res interface{}
}

type GTask struct {
	id   string
	args []interface{}
	task func(args ...interface{}) interface{}
}

type GPool struct {
	gch         chan byte
	gSumNum     int32
	gCurNum     int32
	totalTask   int32
	taskQueue   chan GTask
	resultQueue chan GResult
	gchans      []chan byte
	wg          sync.WaitGroup
}

const (
	//任务队列最大长度
	TASK_QUEUE_MAX_SIZE = 100000
	//结果队列最大长度
	RESULT_QUEUE_MAX_SIZE = 2
	//最大的协程个数
	MAX_GOROUTINE_NUM = 1024
	//最小的协程个数
	MIN_GOROUTINE_NUM = 5
	//锁间隙
	MONITOR_INTERVAL = 1
	//用来增加或减少协程的参数
	SCALE_CHECK_MAX = 5
)

var logger = log.New(os.Stdout, "debug: ", log.LstdFlags|log.Lshortfile)

func NewGPool(gSumNum int32) *GPool {
	taskQueue := make(chan GTask, TASK_QUEUE_MAX_SIZE)
	resultQueue := make(chan GResult, RESULT_QUEUE_MAX_SIZE)
	gch := make(chan byte)
	if gSumNum < MIN_GOROUTINE_NUM {
		gSumNum = MIN_GOROUTINE_NUM
	}
	return &GPool{gSumNum: gSumNum, taskQueue: taskQueue, resultQueue: resultQueue, gch: gch}
}

func (pool *GPool) AddTask(task func(...interface{}) interface{}, args ...interface{}) {
	id := uuid.New().String()
	//logger.Printf("add %v start\n", id)
	pool.taskQueue <- GTask{id: id, task: task, args: args}
	//logger.Printf("add %v stop\n", id)
}

// 控制pool里的协程数
func (pool *GPool) scale() {
	timer := time.NewTicker(MONITOR_INTERVAL * time.Second)
	var scale int16

	for {
		select {
		case <-timer.C:
			logger.Printf("gCurNum: %v\ttotal tasks: %v\n", pool.gCurNum, len(pool.taskQueue))
			switch {
			case len(pool.taskQueue) >= int(2*pool.gCurNum):
				if pool.gCurNum*2 <= MAX_GOROUTINE_NUM && scale >= SCALE_CHECK_MAX {
					pool.incr(pool.gCurNum)
					scale = 0
				} else if scale < SCALE_CHECK_MAX {
					scale++
				}

			case len(pool.taskQueue) <= int(pool.gCurNum/2):
				if pool.gCurNum/2 >= MIN_GOROUTINE_NUM && scale <= -1*SCALE_CHECK_MAX {
					logger.Printf("desc action - gNum: %v\n", pool.gCurNum)
					pool.desc(pool.gCurNum / 2)
					scale = 0
				} else if scale > -1*SCALE_CHECK_MAX {
					scale--
				}

			default:
				logger.Println("no scale!")
			}
		}
		logger.Printf("pool Goroutine num: %v\tscale: %v\n", pool.gCurNum, scale)
	}
}

// 减少pool里的协程数
func (pool *GPool) desc(num int32) {
	logger.Printf("desc info - num: %v\t gchansLen: %v\n", num, len(pool.gchans))
	if int(num) > len(pool.gchans) {
		return
	}
	for _, c := range pool.gchans[:len(pool.gchans)-int(num)] {
		c <- 1
	}
	pool.gchans = pool.gchans[len(pool.gchans)-int(num):]
}

// 增加pool里的协程数
func (pool *GPool) incr(num int32) {
	for i := 0; i < int(num); i++ {
		ch := make(chan byte)
		pool.gchans = append(pool.gchans, ch)
		pool.wg.Add(1)
		go func(mych <-chan byte) {
			atomic.AddInt32(&(pool.gCurNum), 1)
			//logger.Printf("the goroutine %v\n", mych)
			defer func() {
				logger.Println("closed")
				atomic.AddInt32(&pool.gCurNum, -1)
				pool.wg.Done()
			}()

		lable:
			for {
				select {
				case task, ok := <-pool.taskQueue:
					if ok {
						res := task.task(task.args...)
						pool.resultQueue <- GResult{id: task.id, res: res}
					} else {
						break lable
					}
				case cmd := <-mych:
					switch cmd {
					case 1:
						logger.Println("receive stop cmd!")
						break lable
					}
				}
			}
			logger.Println("goroutine end!")
		}(ch)
	}
}

// 协程池启动
func (pool *GPool) Start() {
	pool.incr(pool.gSumNum)
	go pool.scale()
}

// 协程池结束
func (pool *GPool) Stop() {
	close(pool.taskQueue)
	logger.Println("close taskQueue")
	pool.wg.Wait()
	close(pool.resultQueue)
	logger.Println("close resultQueue")
	<-pool.gch
}

// 获取任务处理结果
func (pool *GPool) GetResult(f func(res GResult)) {
	go func() {
		for r := range pool.resultQueue {
			f(r)
			atomic.AddInt32(&pool.totalTask, 1)
		}
		pool.gch <- 1
	}()
}

func (pool *GPool) getCurData() {
	go func() {
		for {
			time.Sleep(1 * time.Second)
			logger.Printf("gCurNum: %v, taskQueue length: %v, resultQueue length: %v\n", pool.gCurNum, len(pool.taskQueue), len(pool.resultQueue))
		}
	}()
}

```

测试代码：

```go
// 测试代码 goroutine_pool_test
package pool

import (
	"fmt"
	"reflect"
	"testing"
	"time"
)

func TestGPool(t *testing.T) {
	pool := NewGPool(2)
	pool.Start()
	pool.GetResult(func(r GResult) {
		t.Logf("Result: %v\n", r)
	})
	pool.getCurData()

	for i := 0; i < 2000; i++ {
		pool.AddTask(func(args ...interface{}) interface{} {
			fmt.Println("Task execute!")
			fmt.Println(reflect.TypeOf(args))
			fmt.Printf("%#v\n", args)
			time.Sleep(10 * time.Millisecond)
			return args
		}, i)
	}
	time.Sleep(30 * time.Second)
	for i := 0; i < 3000; i++ {
		pool.AddTask(func(args ...interface{}) interface{} {
			//fmt.Println("Task execute!")
			//fmt.Println(reflect.TypeOf(args))
			//fmt.Printf("%#v\n", args)
			//time.Sleep(100 * time.Millisecond)
			return args
		}, i)
	}

	pool.Stop()
	t.Logf("Processed total task number: %v\n", pool.totalTask)
	if pool.totalTask != 5000 {
		t.Error("Task Lost!")
	}
}
```



### 13. 画出go标准库net/http下Client的Do方法流程图

![Client的Do方法流程图](C:\Users\lhling\Desktop\goTest\Client的Do方法流程图.png)

### 14. 阐述你对context.Context的理解

tips：context.Context暴露的接口方法

**Answer 14:**  

context 是 Golang 从 1.7 版本引入的一个标准库。它使得一个request范围内所有goroutine运行时的取消可以得到有效的控制。当最上层的 goroutine 因为某些原因执行失败时，下层的 Goroutine 由于没有接收到这个信号所以会继续工作；但是当我们正确地使用 context.Context 时，就可以在下层及时停掉无用的工作以减少额外资源的消耗。Context 是 context 包对外暴露的接口，该接口定义了四个方法，其中包括：

- Deadline 方法需要返回当前 Context 被取消的时间，也就是完成工作的截止日期；
- Done 方法需要返回一个 Channel，这个 Channel 会在当前工作完成或者上下文被取消之后关闭，多次调用 Done 方法会返回同一个 Channel；
- Err 方法会返回当前 Context 结束的原因，它只会在 Done 返回的 Channel 被关闭时才会返回非空的值；
  - 如果当前 Context 被取消就会返回 Canceled 错误；
  - 如果当前 Context 超时就会返回 DeadlineExceeded 错误；
- Value 方法会从 Context 中返回键对应的值，对于同一个上下文来说，多次调用 Value 并传入相同的 Key 会返回相同的结果，这个功能可以用来传递请求特定的数据；

```go
type Context interface {
    // Done returns a channel that is closed when this Context is canceled
    // or times out.
    Done() <-chan struct{}

    // Err indicates why this context was canceled, after the Done channel
    // is closed.
    Err() error

    // Deadline returns the time when this Context will be canceled, if any.
    Deadline() (deadline time.Time, ok bool)

    // Value returns the value associated with key or nil if none.
    Value(key interface{}) interface{}
}
```

可以看到，Context 本质就是一个 interface，实现这四个方法的对象都可以理解为是一个 Context。

context 包提供了四种实现了 Context 接口的 struct，包括：

- emptyCtx：用来作为context树的根节点。我们常用的 context.Background() 与 context.TODO() 底层即为 emptyCtx，没有 Deadline，也获取不到 Value。日常使用的绝大部分 Context 都是在 emptyCtx 的基础上派生出来的。
- valueCtx ：valueCtx 在内嵌 Context 的同时，包含了一组键值对，皆为 interface{}，我们经常使用的 context.WithValue() 函数底层就是一个嵌入了参数 context 的 valueCtx。
- cancelCtx：取消的Context
- timerCtx：一个超时自动取消的Context，内部仍然使用cancelCtx实现取消，额外增一个定时器定时调用 cancel 函数实现该功能（WithTimeOut将当前时间+超时时间计算得到绝对时间后使用WithDeadLine实现）。只能嵌入 cancelCtx ，其他类型的 Context 不行。

对外暴露的四个函数签名：

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)

func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc)

func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)

func WithValue(parent Context, key, val interface{}) Context
```



### 15. 阐述你对reflect.TypeOf、reflect.ValueOf的理解

**Answer 15:**  

- reflect.TypeOf是reflect包提供的一个返回Type类型接口变量的函数，TypeOf能够通过接口抽象出来的方法动态获取输入参数接口中的值的类型，如果接口为空则返回nil。 reflect.TypeOf 接受任意的 interface{} 类型，并以 reflect.Type 形式返回其动态类型：

  ```go
  t := reflect.TypeOf(3)  // a reflect.Type
  fmt.Println(t.String()) // "int"
  fmt.Println(t)          // "int"
  ```

  

- reflect.ValueOf是reflect.Value struct下的函数，用来获取实例的值信息。函数 reflect.ValueOf 接受任意的 interface{} 类型，并返回一个装载着其动态值的 reflect.Value。

  ```go
  v := reflect.ValueOf(3) // a reflect.Value
  fmt.Println(v)          // "3"
  fmt.Printf("%v\n", v)   // "3"
  fmt.Println(v.String()) // NOTE: "<int Value>"
  ```

  

- reflect.ValueOf 的逆操作是 reflect.Value.Interface 方法。它返回一个 interface{} 类型，装载着与 reflect.Value 相同的具体值：

  ```go
  v := reflect.ValueOf(3) // a reflect.Value
  x := v.Interface()      // reflect.Value.Interface
  i := x.(int)            // an int
  fmt.Printf("%d\n", i)   // "3"
  ```

  

### 16. 请用代码实现一个基于channel的广播

```Plain Text
// 举例说明：有一个名为broadcast的chan和一组名为listeners的[]chan
var broadcast chan string
var listeners []chan string

// 1.当给broadcast发送一个string值时，所有的listener都能收到
// 2.listener支持安全地增删
// 3.程序支持接收signal安全退出
// 4.请自行组织程序逻辑结构，不限定实现方式
```

```go
// broadcast.go
package main

import (
	"fmt"
	"sync"
)

type BroadcastService struct {
	// 广播通道，这是消费者监听的通道
	broadcast chan string
	// 监听通道，转发给这些channel
	listeners []chan string
	// 添加消费者
	newRequests chan (chan string)
	// 移除消费者
	removeRequests chan (chan string)
}

// 创建一个广播服务
func NewBroadcastService() *BroadcastService {
	return &BroadcastService{
		broadcast:      make(chan string),
		listeners:      make([]chan string, 3),
		newRequests:    make(chan (chan string)),
		removeRequests: make(chan (chan string)),
	}
}

// 创建一个新消费者并返回一个监听通道
func (bs *BroadcastService) Listener() chan string {
	ch := make(chan string)
	bs.newRequests <- ch
	return ch
}

// 移除一个消费者
func (bs *BroadcastService) RemoveListener(ch chan string) {
	bs.removeRequests <- ch
}
func (bs *BroadcastService) addListener(ch chan string) {
	for i, v := range bs.listeners {
		if v == nil {
			bs.listeners[i] = ch
			return
		}
	}
	bs.listeners = append(bs.listeners, ch)
}
func (bs *BroadcastService) removeListener(ch chan string) {
	for i, v := range bs.listeners {
		if v == ch {
			bs.listeners[i] = nil
			// 一定要关闭! 否则监听它的groutine将会一直block
			close(ch)
			return
		}
	}
}

func (bs *BroadcastService) Run() chan string {
	go func() {
		for {
			// 处理新建消费者或者移除消费者
			select {
			case newCh := <-bs.newRequests:
				bs.addListener(newCh)
			case removeCh := <-bs.removeRequests:
				bs.removeListener(removeCh)
			case v, ok := <-bs.broadcast:
				// 如果广播通道关闭，则关闭掉所有的消费者通道
				if !ok {
					goto terminate
				}
				// 将值转发到所有的消费者channel
				for _, dstCh := range bs.listeners {
					if dstCh == nil {
						continue
					}
					dstCh <- v
				}
			}
		}
	terminate:
		//关闭所有的消费通道
		for _, dstCh := range bs.listeners {
			if dstCh == nil {
				continue
			}
			close(dstCh)

		}
	}()
	return bs.broadcast
}

func main() {
	bs := NewBroadcastService()
	chBroadcast := bs.Run()
	chA := bs.Listener()
	chB := bs.Listener()
	var wg sync.WaitGroup
	wg.Add(2)
	go func() {
		for v := range chA {
			fmt.Println("A", v)
		}
		wg.Done()
	}()
	go func() {
		for v := range chB {
			fmt.Println("B", v)
		}
		wg.Done()
	}()
	//广播消息
	var str string
	for i := 0; i < 3; i++ {
		str = fmt.Sprintf("消息：%d", i)
		chBroadcast <- str
	}
    //移除A监听者
	bs.RemoveListener(chA)
	for i := 3; i < 6; i++ {
		str = fmt.Sprintf("消息：%d", i)
		chBroadcast <- str
	}
    //关闭广播通道
	close(chBroadcast)
	wg.Wait()
}
```



### 17. 值类型和引用类型的区别，如何选择使用

**Answer 17:**  

- 值类型的变量直接存储数据，引用类型的变量存储的是数据的引用，数据存储在数据堆中，引用指向该数据所在位置。
- 值类型的实例数据分配在它所声明的地方，作为类成员，跟随所属实例存储，作为方法中的局部变量，在栈上存储。引用类型的引用变量在栈中分配，对象在堆中分配内存。
- 值类型能够直接返回数据，相比引用类型效率更高
- 在值类型和引用类型作为函数或方法的形参时，在go语言中，都属于值传递，值类型会传递数据值的副本，函数或方法对形参的修改不会影响值类型的实参；引用类型会传递引用变量的副本，该副本同样指向对象实例，因此对指针变量的修改会影响到对象数据。
- 值类型不支持多态，引用类型支持
- 如何选择：通常将用于底层数据存储的类型设计为值类型，将用于定义应用程序行为的类型设计为引用类型，如果对类型将来的应用情况不确定，应该使用引用类型。



### 18. 函数调用返回error，处理方式有哪些？如何获取error的栈信息

**Answer 18:**  

```go
//1. 使用原生的error
if err != nil {
    //...
    return err
}

//2. 提前定义好error
//把错误定义为一个变量，再判断
var (
    ErrInvalidParam = errors.New("invalid param")
)
//为防止包装后的导致误判，使用errors.Is()判断
if errors.Is(err, ErrInvalidParam) {
    //..
}

//3. 使用自定义的错误类型
//自定义一种错误类型：
type ErrInvalidParam struct {
    ParamName  string
    ParamValue string
}

func (e *ErrInvalidParam) Error() string {
    return fmt.Sprintf("invalid param: %+v, value: %+v", e.ParamName, e.ParamValue)
}
//使用类型断言机制或者类型选择机制，来对不同类型的错误进行处理：
e, ok := err.(*ErrInvalidParam)
if ok && e != nil {
	//...
}

//4.使用一些错误包


```



### 19. 系统panic有什么影响，如何解决和快速定位panic根因

**Answer 19:**  

- 影响：会使程序终止运行，系统宕机。

- 定位方法：
  - 调用栈：查看panic时的调用栈，根据打印的出错函数及文件行数，找到panic的位置，再详细处理。
  - 出错地址： 
    1. 获取内核模块基地址
    2. 计算地址偏移
    3. 生成模块可调试文件
    4. gdb调试



### 20. 写一段伪代码实现目录树中的文件排序功能，需要支持文件可拖动排序。

**Answer 20:**  

```js
//1.排序
function compare(property){
    return function(a,b){
        var value1 = a[property];
        var value2 = b[property];
        return value1 - value2;
}
//2、拖动，将其分为三种情况：同级目录 inner、不同级目录prev，不同级目录next
//pid表示父目录Id,id表示当前文件节点id,targetNode为拖拽到的目标文件下
//伪代码实现，其中的语法表达只是为了表达功能
func drop(targetNode,moveType,curNode，trees){
	if moveType == 'inner'{
        if targetNode.id == curNode.pid
        //在同一父目录下移动
        //排序,调用compare
        zNodes= newList.sort(compare('sort'));
        else {
            //不同父目录下移动,将当前节点添加为新的父目录的孩子节点
            targetNode.child.add(curNode)
            //将当前节点从原来父目录的孩子节点中去除
            trees[curNode.pid].child.delete(curNode)
            //将当前节点的父节点设为目标节点id
            curNode.pid = targetNode.id
            //排序
            zNodes= newList.sort(compare('sort'));
        }  
    } else {
        //将当前节点从原来父目录的孩子节点中去除
        trees[curNode.pid].child.delete(curNode)
        //将当前节点的父节点设为目标节点id
        curNode.pid = targetNode.id
        //排序
        zNodes= newList.sort(compare('sort'));
        level = targetNode.level + 1
    } 
}
```

