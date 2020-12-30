[TOC]

## 六、WaitGroup

### (1) WaitGroup与goroutine的竞速问题

>  编译并运行如下代码会发生什么？

> test18.go

```go
package main

import (
	"sync"
	//"time"
)

const N = 10

var wg = &sync.WaitGroup{}

func main() {

	for i := 0; i < N; i++ {
		go func(i int) {
			wg.Add(1)
			println(i)
			defer wg.Done()
		}(i)
	}
	wg.Wait()

}
```

**结果**

```bash
结果不唯一，代码存在风险, 所有go未必都能执行到
```



这是使用WaitGroup经常犯下的错误！请各位同学多次运行就会发现输出都会不同甚至又出现报错的问题。 这是因为`go`执行太快了，导致`wg.Add(1)`还没有执行main函数就执行完毕了。 改为如下试试

```go
package main

import (
	"sync"
)

const N = 10

var wg = &sync.WaitGroup{}

func main() {

    for i:= 0; i< N; i++ {
        wg.Add(1)
        go func(i int) {
            println(i)
            defer wg.Done()
        }(i)
    }

    wg.Wait()
}

```