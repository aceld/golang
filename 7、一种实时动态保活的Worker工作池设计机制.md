[TOC]
## 7、动态保活Worker工作池设计


### 一、我们如何知道一个Goroutine已经死亡？
实际上，Go语言并没有给我们暴露如何知道一个Goroutine是否存在的接口，如果要证明一个Go是否存在，可以在子Goroutine的业务中，定期写向一个keep live的Channel，然后主Goroutine来发现当前子Go的状态。Go语言在对于Go和Go之间没有像进程和线程一样有强烈的父子、兄弟等关系，每个Go实际上对于调度器都是一个独立的，平等的执行流程。

>PS: 如果你是监控子线程、子进程的死亡状态，就没有这么简单了，这里也要感谢go的调度器给我们提供的方便，我们既然用Go，就要基于Go的调度器来实现该模式。

那么，我们如何做到一个Goroutine已经死亡了呢？

#### 子Goroutine
可以通过给一个被监控的Goroutine添加一个`defer` ，然后`recover()` 捕获到当前Goroutine的异常状态，最后给主Goroutine发送一个死亡信号，通过`Channel`。

#### 主Goroutine
在`主Goroutine`上，从这个`Channel`读取内容，当读到内容时，就重启这个`子Goroutine`，当然`主Goroutine`需要记录`子Goroutine`的`ID`，这样也就可以针对性的启动了。


### 二、代码实现
我们这里以一个工作池的场景来对上述方式进行实现。

`WorkerManager`作为`主Goroutine`, `worker`作为子`Goroutine`

> WorkerManager

```go
type WorkerManager struct {
   //用来监控Worker是否已经死亡的缓冲Channel
   workerChan chan *worker
   // 一共要监控的worker数量
   nWorkers int
}

//创建一个WorkerManager对象
func NewWorkerManager(nworkers int) *WorkerManager {
   return &WorkerManager{
      nWorkers:nworkers,
      workerChan: make(chan *worker, nworkers),
   }
}

//启动worker池，并为每个Worker分配一个ID，让每个Worker进行工作
func (wm *WorkerManager)StartWorkerPool() {
   //开启一定数量的Worker
   for i := 0; i < wm.nWorkers; i++ {
      i := i
      wk := &worker{id: i}
      go wk.work(wm.workerChan)
   }

  //启动保活监控
   wm.KeepLiveWorkers()
}

//保活监控workers
func (wm *WorkerManager) KeepLiveWorkers() {
   //如果有worker已经死亡 workChan会得到具体死亡的worker然后 打出异常，然后重启
   for wk := range wm.workerChan {
      // log the error
      fmt.Printf("Worker %d stopped with err: [%v] \n", wk.id, wk.err)
      // reset err
      wk.err = nil
      // 当前这个wk已经死亡了，需要重新启动他的业务
      go wk.work(wm.workerChan)
   }
}
```


>worker
```go
type worker struct {
   id  int
   err error
}

func (wk *worker) work(workerChan chan<- *worker) (err error) {
   // 任何Goroutine只要异常退出或者正常退出 都会调用defer 函数，所以在defer中想WorkerManager的WorkChan发送通知
   defer func() {
      //捕获异常信息，防止panic直接退出
      if r := recover(); r != nil {
         if err, ok := r.(error); ok {
            wk.err = err
         } else {
            wk.err = fmt.Errorf("Panic happened with [%v]", r)
         }
      } else {
         wk.err = err
      }
 
     //通知 主 Goroutine，当前子Goroutine已经死亡
      workerChan <- wk
   }()

   // do something
   fmt.Println("Start Worker...ID = ", wk.id)

   // 每个worker睡眠一定时间之后，panic退出或者 Goexit()退出
   for i := 0; i < 5; i++ {
      time.Sleep(time.Second*1)
   }

   panic("worker panic..")
   //runtime.Goexit()

   return err
}
```


### 三、测试

>main
```go
func main() {
   wm := NewWorkerManager(10)

   wm.StartWorkerPool()
}
```

结果：

```bash
$ go run workmanager.go 
Start Worker...ID =  2
Start Worker...ID =  1
Start Worker...ID =  3
Start Worker...ID =  4
Start Worker...ID =  7
Start Worker...ID =  6
Start Worker...ID =  8
Start Worker...ID =  9
Start Worker...ID =  5
Start Worker...ID =  0
Worker 9 stopped with err: [Panic happened with [worker panic..]] 
Worker 1 stopped with err: [Panic happened with [worker panic..]] 
Worker 0 stopped with err: [Panic happened with [worker panic..]] 
Start Worker...ID =  9
Start Worker...ID =  1
Worker 2 stopped with err: [Panic happened with [worker panic..]] 
Worker 5 stopped with err: [Panic happened with [worker panic..]] 
Worker 4 stopped with err: [Panic happened with [worker panic..]] 
Start Worker...ID =  0
Start Worker...ID =  2
Start Worker...ID =  4
Start Worker...ID =  5
Worker 7 stopped with err: [Panic happened with [worker panic..]] 
Worker 8 stopped with err: [Panic happened with [worker panic..]] 
Worker 6 stopped with err: [Panic happened with [worker panic..]] 
Worker 3 stopped with err: [Panic happened with [worker panic..]] 
Start Worker...ID =  3
Start Worker...ID =  6
Start Worker...ID =  8
Start Worker...ID =  7
...
...
```

我们会发现，无论子Goroutine是因为 panic()异常退出，还是Goexit()退出，都会被主Goroutine监听到并且重启。主要我们就能够起到保活的功能了. 当然如果线程死亡？进程死亡？我们如何保证？ 大家不用担心，我们用Go开发实际上是基于Go的调度器来开发的，进程、线程级别的死亡，会导致调度器死亡，那么我们的全部基础框架都将会塌陷。那么就要看线程、进程如何保活啦，不在我们Go开发的范畴之内了。
