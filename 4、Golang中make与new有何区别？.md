[TOC]

## 4、Golang中make与new有何区别？

### 一、**前言**

本文主要给大家介绍了Go语言中函数`new`与`make`的使用和区别，关于Go语言中`new`和`make`是内建的两个函数，主要用来创建分配类型内存。在我们定义生成变量的时候，可能会觉得有点迷惑，其实他们的规则很简单，下面我们就通过一些示例说明他们的区别和使用。

### 二、变量的声明

```go
var i int
var s string
```

​		变量的声明我们可以通过var关键字，然后就可以在程序中使用。当我们不指定变量的默认值时，这些变量的默认值是他们的零值，比如int类型的零值是0,string类型的零值是`""`，引用类型的零值是`nil`。

对于例子中的两种类型的声明，我们可以直接使用，对其进行赋值输出。但是如果我们换成指针类型呢？

> test1.go

```go
package main

import (
 "fmt"
)

func main() {
   var i *int
   *i=10
   fmt.Println(*i)
}
```



```bash
$ go run test1.go 
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x0 pc=0x4849df]

goroutine 1 [running]:
main.main()
	/home/itheima/go/src/golang_deeper/make_new/t
```

从这个提示中可以看出，对于引用类型的变量，我们不光要声明它，还要为它分配内容空间，否则我们的值放在哪里去呢？这就是上面错误提示的原因。

对于值类型的声明不需要，是因为已经默认帮我们分配好了。

要分配内存，就引出来今天的`new`和`make`。



### 三、new

对于上面的问题我们如何解决呢？既然我们知道了没有为其分配内存，那么我们使用new分配一个吧。

```go
func main() {
  
   var i *int
   i=new(int)
   *i=10
   fmt.Println(*i)
  
}
```

现在再运行程序，完美PASS，打印10。现在让我们看下new这个内置的函数。

```go
// The new built-in function allocates memory. The first argument is a type,
// not a value, and the value returned is a pointer to a newly
// allocated zero value of that type.
func new(Type) *Type
```

​		它只接受一个参数，这个参数是一个类型，分配好内存后，返回一个指向该类型内存地址的指针。同时请注意它同时把分配的内存置为零，也就是类型的零值。

我们的例子中，如果没有`*i=10`，那么打印的就是0。这里体现不出来new函数这种内存置为零的好处，我们再看一个例子。

> test2.go

```go
package main

import (
    "fmt"
    "sync"
)

type user struct {
    lock sync.Mutex
    name string
    age int
}

func main() {

    u := new(user) //默认给u分配到内存全部为0

    u.lock.Lock()  //可以直接使用，因为lock为0,是开锁状态
    u.name = "张三"
    u.lock.Unlock()

    fmt.Println(u)
}
```

运行

```bash
$ go run test2.go 
&{{0 0} 张三 0}
```



示例中的user类型中的lock字段我不用初始化，直接可以拿来用，不会有无效内存引用异常，因为它已经被零值了。

这就是new，它返回的永远是类型的指针，指向分配类型的内存地址。



### 四、make

make也是用于内存分配的，但是和new不同。

它只用于

* chan
* map
* slice

的内存创建，而且它返回的类型就是这三个类型本身，而不是他们的指针类型，因为这三种类型就是引用类型，所以就没有必要返回他们的指针了。

注意，因为这三种类型是引用类型，所以必须得初始化，但是不是置为零值，这个和new是不一样的。

```go
func make(t Type, size ...IntegerType) Type
```

从函数声明中可以看到，返回的还是该类型。



### 五、make与new的异同



相同

* 堆空间分配



不同

make: 只用于slice、map以及channel的初始化， 无可替代

new: 用于类型内存分配(初始化值为0)， 不常用



> new不常用
>
> 所以有new这个内置函数，可以给我们分配一块内存让我们使用，但是现实的编码中，它是不常用的。我们通常都是采用短语句声明以及结构体的字面量达到我们的目的，比如：
>
> ```go
> i : =0
> u := user{}
> ```



> make 无可替代
>
> 我们在使用slice、map以及channel的时候，还是要使用make进行初始化，然后才才可以对他们进行操作。




