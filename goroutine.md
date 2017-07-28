## goroutine

大家对于goroutine应该都很熟悉，goroutine是go的核心组成之一，即使用关键字go可以将一部分代码并发的执行。既然是并发那么就会将我们代码的复杂性增加，既然是并发那么当有多个goroutine时，它就会交给runtime来调度，这个顺序也是随机的(请记住这一点，后面我们会渗入理解这个随机性)，如果想要按照我们定义的顺序执行我们可以通过channel或者锁来实现同步。

### 应用将不会等待所有的goroutines完成
这对于初学者而言是个很常见的错误，看下面代码，猜猜输出是什么？
```
package main

import (
	"fmt"
)

func gorun(i int) {
	fmt.Println(i)
}

func main() {
	for i := 0; i < 10; i++ {
		go gorun(i)
	}
}
```
是的，将是不确定的输出，main函数才不会等待所有的goroutine结束才退出。
最常见的解决方法是使用“WaitGroup”变量。它将会让主goroutine等待所有的worker goroutine完成。或者你也可以通过channel给gorputine发送信号，让它们退出。
就像这样：
```
package main

import (
	"fmt"
	"sync"
)

var wg sync.WaitGroup

func gorun(i int) {
	fmt.Println(i)
	wg.Done()
}

func main() {
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go gorun(i)
	}
	wg.Wait()
}
```
### 如何使用多核并行任务？
Go既然是并发的执行，大家都理解并发和并行的区别。并发一般是被内核通过时间片或者中断来控制的，遇到io阻塞或者时间片用完的时会转移线程的使用权，而并行是多个CPU同时执行任务，即在同一时间有多个任务在调度。所以一个核的情况下不可能有并行的情况，因为同一时间只有一个任务在调度。 Go默认所有任务都运行在一个cpu核里，如果要在goroutine中使用多核，go就给我们暴露了一个接口来控制使用CPU的数量可以使用，runtime.GOMAXPROCS 函数修改，当参数小于 1 时使用默认值。
```
package main

import (
	"fmt"
	"runtime"
	"sync"
)

var wg sync.WaitGroup

func gorun(i int) {
	fmt.Println(i)
	wg.Done()
}

func main() {
	runtime.GOMAXPROCS(runtime.NumCPU())
	<!-- runtime.GOMAXPROCS(1） -->
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go gorun(i)
	}
	wg.Wait()
}
```
Ok,我们看到数字乱序的执行，随机输出，很好理解。然后现在我们把代码改为runtime.GOMAXPROCS(1）
输出是什么？
9 0 1 2 3 4 5 6 7 8
再运行一次，结果仍然不变。此时我们是不是可以下结论说在单核情况下是顺序执行的。但你还会疑惑，不应该顺序执行吗？我们会有疑问，为什么不是 
0 1 2 3 4 5 6 7 8 9
其实在这里我们还记不记得我们说过goroutine默认情况下调度是随机的？我们可以加入-race参数再看看。
顺序变成随机的了，这里其实并没有什么改变只是加入了race检测。
可以看到，即使用了 runtime.GOMAXPROCS(1)，也不能保证 goroutine 以确定的顺序执行。不用 -race 参数，且保证 GOMAXPROCS 为 1，在不同版本下，也可能有不同的输出。这个就像Go中的Map一样仍然是随机的。
如果想要 goroutine 以确定的顺序执行，就要用各种线程同步的机制，channel 也好，sync 包里的各种机制也好，总之不能依赖调度器当前的实现。
所以，在Go的世界里，我们也要坚持最基本的准则，这也是Go的陷阱之一，很多熟悉的人都会陷入。

### 优先调度
一个goroutine阻止其他goroutine运行。例如当你有一个不让调度器运行的for循环时。
```
package main

import (
	"fmt"
)
func main() {  
    done := false
    go func(){
        done = true
    }()
    for !done {
    }
    fmt.Println("done!")
}
```
for循环并不需要是空的。只要它包含了不会触发调度执行的代码，就会发生这种问题。调度器会在GC、“go”声明、阻塞channel操作、阻塞系统调用和lock操作后运行。它也会在非内联函数调用后执行。
我们就可以使用runtime.Gosched()用于让出CPU时间片，让当前线程让出 cpu 以让其它线程运行,它不会挂起当前线程，因此当前线程未来会继续执行。

### 其他
退出一个goroutine，使用runtime.Goexit()
查看当前gorputine数量，runtime.NumGoroutine()