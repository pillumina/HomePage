---
title: "Channels Concurrency Work-Around"
date: 2020-12-22T11:22:18+08:00
hero: /images/posts/golang_banner.jpg
menu:
  sidebar:
    name: Channels Concurrency
    identifier: channels-concurrency
    parent: golang odyssey
    weight: 10
draft: false
---

  记录了一些channels常见的场景，以及自己的一些感受：

- 使用通道进行异步和并发编程是简单和惬意的；
- 通道同步技术比被很多其它语言采用的其它同步方案（比如[角色模型](https://en.wikipedia.org/wiki/Actor_model)和[async/await模式](https://en.wikipedia.org/wiki/Async/await)）有着更多的应用场景和更多的使用变种。

  通道作为同步手段，并非在任何情况下都是最佳的同步技术，本文也会补充原子操作和sync包内其他的技术作为参考。



### 将通道用做future/promise

很多其它流行语言支持future/promise来实现异步（并发）编程。 Future/promise常常用在请求/回应场合。

#### 返回单向接收通道做为函数返回结果

在下面这个例子中，`sumSquares`函数调用的两个实参请求并发进行。 每个通道读取操作将阻塞到请求返回结果为止。 两个实参总共需要大约3秒钟（而不是6秒钟）准备完毕（以较慢的一个为准）。

```go
package main

import (
	"time"
	"math/rand"
	"fmt"
)

func longTimeRequest() <-chan int32 {
	r := make(chan int32)

	go func() {
		time.Sleep(time.Second * 3) // 模拟一个工作负载
		r <- rand.Int31n(100)
	}()

	return r
}

func sumSquares(a, b int32) int32 {
	return a*a + b*b
}

func main() {
	rand.Seed(time.Now().UnixNano())

	a, b := longTimeRequest(), longTimeRequest()
	fmt.Println(sumSquares(<-a, <-b))
}
```



#### 将单向发送通道类型用做函数实参

和上例一样，在下面这个例子中，`sumSquares`函数调用的两个实参的请求也是并发进行的。 和上例不同的是`longTimeRequest`函数接收一个单向发送通道类型参数而不是返回一个单向接收通道结果。

```go
package main

import (
	"time"
	"math/rand"
	"fmt"
)

func longTimeRequest(r chan<- int32)  {
	time.Sleep(time.Second * 3) // 模拟一个工作负载
	r <- rand.Int31n(100)
}

func sumSquares(a, b int32) int32 {
	return a*a + b*b
}

func main() {
	rand.Seed(time.Now().UnixNano())

	ra, rb := make(chan int32), make(chan int32)
	go longTimeRequest(ra)
	go longTimeRequest(rb)

	fmt.Println(sumSquares(<-ra, <-rb))
}
```

对于上面这个特定的例子，我们可以只使用一个通道来接收回应结果，因为两个参数的作用是对等的。

```go
...

	results := make(chan int32, 2) // 缓冲与否不重要
	go longTimeRequest(results)
	go longTimeRequest(results)

	fmt.Println(sumSquares(<-results, <-results))
}
```

这可以看作是后面将要提到的数据聚合的一个应用。

#### 采用最快回应

本用例可以看作是上例中只使用一个通道变种的增强。

有时候，一份数据可能同时从多个数据源获取。这些数据源将返回相同的数据。 因为各种因素，这些数据源的回应速度参差不一，甚至某个特定数据源的多次回应速度之间也可能相差很大。 同时从多个数据源获取一份相同的数据可以有效保障低延迟。我们只需采用最快的回应并舍弃其它较慢回应。

注意：如果有*N*个数据源，为了防止被舍弃的回应对应的协程永久阻塞，则传输数据用的通道必须为一个容量至少为*N-1*的缓冲通道。

```go
package main

import (
	"fmt"
	"time"
	"math/rand"
)

func source(c chan<- int32) {
	ra, rb := rand.Int31(), rand.Intn(3) + 1
	// 睡眠1秒/2秒/3秒
	time.Sleep(time.Duration(rb) * time.Second)
	c <- ra
}

func main() {
	rand.Seed(time.Now().UnixNano())

	startTime := time.Now()
	c := make(chan int32, 5) // 必须用一个缓冲通道
	for i := 0; i < cap(c); i++ {
		go source(c)
	}
	rnd := <- c // 只有第一个回应被使用了
	fmt.Println(time.Since(startTime))
	fmt.Println(rnd)
}
```

“采用最快回应”用例还有一些其它实现方式，本文后面将会谈及。

#### 更多“请求/回应”用例变种

做为函数参数和返回结果使用的通道可以是缓冲的，从而使得请求协程不需阻塞到它所发送的数据被接收为止。

有时，一个请求可能并不保证返回一份有效的数据。对于这种情形，我们可以使用一个形如`struct{v T; err error}`的结构体类型或者一个空接口类型做为通道的元素类型以用来区分回应的值是否有效。

有时，一个请求可能需要比预期更长的用时才能回应，甚至永远都得不到回应。 我们可以使用本文后面将要介绍的超时机制来应对这样的情况。

有时，回应方可能会不断地返回一系列值，这也同时属于后面将要介绍的数据流的一个用例。



### 使用通道实现通知

通知可以被看作是特殊的请求/回应用例。在一个通知用例中，我们并不关心回应的值，我们只关心回应是否已发生。 所以我们常常使用空结构体类型`struct{}`来做为通道的元素类型，因为空结构体类型的尺寸为零，能够节省一些内存（虽然常常很少量）。

#### 向一个通道发送一个值来实现单对单通知

我们已知道，如果一个通道中无值可接收，则此通道上的下一个接收操作将阻塞到另一个协程发送一个值到此通道为止。 所以一个协程可以向此通道发送一个值来通知另一个等待着从此通道接收数据的协程。

在下面这个例子中，通道`done`被用来做为一个信号通道来实现单对单通知。

```go
package main

import (
	"crypto/rand"
	"fmt"
	"os"
	"sort"
)

func main() {
	values := make([]byte, 32 * 1024 * 1024)
	if _, err := rand.Read(values); err != nil {
		fmt.Println(err)
		os.Exit(1)
	}

	done := make(chan struct{}) // 也可以是缓冲的

	// 排序协程
	go func() {
		sort.Slice(values, func(i, j int) bool {
			return values[i] < values[j]
		})
		done <- struct{}{} // 通知排序已完成
	}()

	// 并发地做一些其它事情...

	<- done // 等待通知
	fmt.Println(values[0], values[len(values)-1])
}
```

#### 从一个通道接收一个值来实现单对单通知

如果一个通道的数据缓冲队列已满（非缓冲的通道的数据缓冲队列总是满的）但它的发送协程队列为空，则向此通道发送一个值将阻塞，直到另外一个协程从此通道接收一个值为止。 所以我们可以通过从一个通道接收数据来实现单对单通知。一般我们使用非缓冲通道来实现这样的通知。

**这种通知方式不如上例中介绍的方式使用得广泛，基本很少用**。

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	done := make(chan struct{})
		// 此信号通道也可以缓冲为1。如果这样，则在下面
		// 这个协程创建之前，我们必须向其中写入一个值。

	go func() {
		fmt.Print("Hello")
		// 模拟一个工作负载。
		time.Sleep(time.Second * 2)

		// 使用一个接收操作来通知主协程。
		<- done
	}()

	done <- struct{}{} // 阻塞在此，等待通知
	fmt.Println(" world!")
}
```

另一个事实是，上面的两种单对单通知方式其实并没有本质的区别。 它们都可以被概括为较快者等待较慢者发出通知。



#### 多对单和单对多通知

略微扩展一下上面两个用例，我们可以很轻松地实现多对单和单对多通知。

```go
package main

import "log"
import "time"

type T = struct{}

func worker(id int, ready <-chan T, done chan<- T) {
	<-ready // 阻塞在此，等待通知
	log.Print("Worker#", id, "开始工作")
	// 模拟一个工作负载。
	time.Sleep(time.Second * time.Duration(id+1))
	log.Print("Worker#", id, "工作完成")
	done <- T{} // 通知主协程（N-to-1）
}

func main() {
	log.SetFlags(0)

	ready, done := make(chan T), make(chan T)
	go worker(0, ready, done)
	go worker(1, ready, done)
	go worker(2, ready, done)

	// 模拟一个初始化过程
	time.Sleep(time.Second * 3 / 2)
	// 单对多通知
	ready <- T{}; ready <- T{}; ready <- T{}
	// 等待被多对单通知
	<-done; <-done; <-done
}
```

  这种写法是比较少见的，因为not clean enough，一般用`sync.WaitGroup`实现多对单的通知，使用关闭一个通道方式实现单对多。



#### 通过关闭一个通道来实现群发通知（单对多模式优化）

  关闭一个通道进行对多通知更简单。用到的特性是`能够从一个已经关闭的通道接受到无穷多的值`。

  我们可以把上一个例子中的三个数据发送操作`ready <- struct{}{}`替换为一个通道关闭操作`close(ready)`来达到同样的单对多通知效果。

```go
...
	close(ready) // 群发通知Let's go!
...
```

  其实，单对单通知一般也是用关闭通道的方式，这也是实践中用到最多的通知实现方式。`context`库中用这种特性实现了传达`操作取消`消息，后续会介绍具体的cases。



#### 定时通知（timer）



