# tinyPool
Concurrency limiting goroutine pool. Limits the concurrency of task execution, not the number of tasks queued. submitting tasks can be sync/async mode, no matter how many tasks are queued.


License: MIT


## Installation
To install this package, you need to setup your Go workspace.  The simplest way to install the library is to run:
```
$ go get github.com/pandaknight2021/tinyPool
```

## Example
```go
package main

import (
	"fmt"
	"sync"
	"time"

	"github.com/pandaknight2021/tinyPool"
)

func demofn() {
	time.Sleep(10 * time.Millisecond)
	fmt.Println("demo")
}

func main() {
	var wg sync.WaitGroup
	p, _ := tinyPool.NewPool(10)
	defer p.Close()

	for j := 0; j < 10; j++ {
		wg.Add(1)
		p.Submit(func() {
			demofn()
			wg.Done()
		})
	}
	wg.Wait()
	fmt.Println("done")
}

```
## Benchmark

test env:  
cpu: i5 8th    memory:  16G  
poolsize: 5k   task: 1M (fast) / 100k(slow)
  
``` shell
ggoos: linux
goarch: amd64
pkg: github.com/pandaknight2021/tinyPool
cpu: Intel(R) Core(TM) i5-8250U CPU @ 1.60GHz
BenchmarkGoroutines_fast
BenchmarkGoroutines_fast-8   	       1	9527376700 ns/op	215811600 B/op	 1315699 allocs/op
BenchmarkTinyPool_fast
BenchmarkTinyPool_fast-8     	       1	6687470500 ns/op	16882824 B/op	 1010441 allocs/op
BenchmarkGoroutines_slow
BenchmarkGoroutines_slow-8   	       1	130869875900 ns/op	 7999920 B/op	   99999 allocs/op
BenchmarkTinyPool_slow
BenchmarkTinyPool_slow-8     	       1	139006410300 ns/op	 2439000 B/op	  109604 allocs/op
PASS
ok  	github.com/pandaknight2021/tinyPool	286.839s

```

## 📚 Relevant reference
-  [ants] (https://github.com/panjf2000/ants)
-  [The Case For A Go Worker Pool](https://brandur.org/go-worker-pool)
-  [workerpool](https://github.com/gammazero/workerpool)

## 后记
### 刷新了我对gorutine的性能认知

作为一个new gopher, 压抑不住造轮子的冲动, 决定干个简单的go pool, 按照我资深不入流的CPP经验, 实现方式无非: queue + goroutine pool, 说干就干,结果发现标准库没有现成的queue, 这也不是难事, 那就造个queue, 先用slice实现了一个加锁简易版mpsc,一测性能惨不忍睹,逐放弃. 那就看看其他人的实现, 发现有人实现list的无锁mpsc, 效率惊人. 但个人更喜欢基于ring buffer的mpsc, 于是徒手撸了一个,测试效果比list版高20%~30%, queue问题解决,那就开始实现pool吧,具体实现的方式 : submit(task) ->  queue.push(task) -> dipatcher.send(task) -> workers(goroutine), so easy. 三下五除二,搞定. benchmark一看: 竟然比无脑goroutine还慢3倍, mallocs内存也高的多, 各种排查,对比发现,问题出在mpsc分配内存次数过多(长度1M),那就改变实现方式: submit(task) -> channel -> worker(goroutine), 改完一测, 耗时优于无脑goroutine, 内存不到1/20.以前总听别人说Goroutine效率高,这次总算有了切身体会. 收工.





