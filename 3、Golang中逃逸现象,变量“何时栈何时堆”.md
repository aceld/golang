[TOC]

## 3、Golang中逃逸现象, 变量“何时栈?何时堆?”

### 一、C/C++报错?Golang通过?

我们先看一段代码

```go
package main

func foo(arg_val int)(*int) {

    var foo_val int = 11;
    return &foo_val;
}

func main() {

    main_val := foo(666)

    println(*main_val)
}
```

编译运行

```bash
$ go run pro_1.go 
11
```

竟然没有报错!



了解C/C++的小伙伴应该知道,这种情况是一定不允许的,因为 外部函数使用了子函数的局部变量, 理论来说,子函数的`foo_val` 的声明周期早就销毁了才对,如下面的C/C++代码

```c
#include <stdio.h>

int *foo(int arg_val) {

    int foo_val = 11;

    return &foo_val;
}

int main()
{
    int *main_val = foo(666);

    printf("%d\n", *main_val);
}

```

编译

```bash
$ gcc pro_1.c 
pro_1.c: In function ‘foo’:
pro_1.c:7:12: warning: function returns address of local variable [-Wreturn-local-addr]
     return &foo_val;
            ^~~~~~~~

```

出了一个警告,不管他,再运行

```
$ ./a.out 
段错误 (核心已转储)
```

程序崩溃.



如上C/C++编译器明确给出了警告，foo把一个局部变量的地址返回了；反而高大上的go没有给出任何警告，难道是go编译器识别不出这个问题吗？



### 二、Golang编译器得逃逸分析

​		go语言编译器会自动决定把一个变量放在栈还是放在堆，编译器会做**逃逸分析(escape analysis)**，**当发现变量的作用域没有跑出函数范围，就可以在栈上，反之则必须分配在堆**。
 go语言声称这样可以释放程序员关于内存的使用限制，更多的让程序员关注于程序功能逻辑本身。



我们再看如下代码:

```go
package main

func foo(arg_val int) (*int) {

    var foo_val1 int = 11;
    var foo_val2 int = 12;
    var foo_val3 int = 13;
    var foo_val4 int = 14;
    var foo_val5 int = 15;


    //此处循环是防止go编译器将foo优化成inline(内联函数)
    //如果是内联函数，main调用foo将是原地展开，所以foo_val1-5相当于main作用域的变量
    //即使foo_val3发生逃逸，地址与其他也是连续的
    for i := 0; i < 5; i++ {
        println(&arg_val, &foo_val1, &foo_val2, &foo_val3, &foo_val4, &foo_val5)
    }

    //返回foo_val3给main函数
    return &foo_val3;
}


func main() {
    main_val := foo(666)

    println(*main_val, main_val)
}

```

我们运行一下

```bash
$ go run pro_2.go 
0xc000030758 0xc000030738 0xc000030730 0xc000082000 0xc000030728 0xc000030720
0xc000030758 0xc000030738 0xc000030730 0xc000082000 0xc000030728 0xc000030720
0xc000030758 0xc000030738 0xc000030730 0xc000082000 0xc000030728 0xc000030720
0xc000030758 0xc000030738 0xc000030730 0xc000082000 0xc000030728 0xc000030720
0xc000030758 0xc000030738 0xc000030730 0xc000082000 0xc000030728 0xc000030720
13 0xc000082000

```

我们能看到`foo_val3`是返回给main的局部变量, 其中他的地址应该是`0xc000082000`,很明显与其他的foo_val1、2、3、4不是连续的.

我们用`go tool compile`测试一下

```bash
$ go tool compile -m pro_2.go
pro_2.go:24:6: can inline main
pro_2.go:7:9: moved to heap: foo_val3
```

果然,在编译的时候, `foo_val3`具有被编译器判定为逃逸变量, 将`foo_val3`放在堆中开辟.



我们在用汇编证实一下:

```bash
$ go tool compile -S pro_2.go > pro_2.S
```

打开pro_2.S文件, 搜索`runtime.newobject`关键字

```go
 ...
 16     0x0021 00033 (pro_2.go:5)   PCDATA  $0, $0
 17     0x0021 00033 (pro_2.go:5)   PCDATA  $1, $0
 18     0x0021 00033 (pro_2.go:5)   MOVQ    $11, "".foo_val1+48(SP)
 19     0x002a 00042 (pro_2.go:6)   MOVQ    $12, "".foo_val2+40(SP)
 20     0x0033 00051 (pro_2.go:7)   PCDATA  $0, $1
 21     0x0033 00051 (pro_2.go:7)   LEAQ    type.int(SB), AX
 22     0x003a 00058 (pro_2.go:7)   PCDATA  $0, $0
 23     0x003a 00058 (pro_2.go:7)   MOVQ    AX, (SP)
 24     0x003e 00062 (pro_2.go:7)   CALL    runtime.newobject(SB)  //foo_val3是被new出来的
 25     0x0043 00067 (pro_2.go:7)   PCDATA  $0, $1
 26     0x0043 00067 (pro_2.go:7)   MOVQ    8(SP), AX
 27     0x0048 00072 (pro_2.go:7)   PCDATA  $1, $1
 28     0x0048 00072 (pro_2.go:7)   MOVQ    AX, "".&foo_val3+56(SP)
 29     0x004d 00077 (pro_2.go:7)   MOVQ    $13, (AX)
 30     0x0054 00084 (pro_2.go:8)   MOVQ    $14, "".foo_val4+32(SP)
 31     0x005d 00093 (pro_2.go:9)   MOVQ    $15, "".foo_val5+24(SP)
 32     0x0066 00102 (pro_2.go:9)   XORL    CX, CX
 33     0x0068 00104 (pro_2.go:15)  JMP 252
 ...
```

看出来, foo_val3是被runtime.newobject()在堆空间开辟的, 而不是像其他几个是基于地址偏移的开辟的栈空间.

### 三、new的变量在栈还是堆?

那么对于new出来的变量,是一定在heap中开辟的吗,我们来看看

```go
package main

func foo(arg_val int) (*int) {

    var foo_val1 * int = new(int);
    var foo_val2 * int = new(int);
    var foo_val3 * int = new(int);
    var foo_val4 * int = new(int);
    var foo_val5 * int = new(int);


    //此处循环是防止go编译器将foo优化成inline(内联函数)
    //如果是内联函数，main调用foo将是原地展开，所以foo_val1-5相当于main作用域的变量
    //即使foo_val3发生逃逸，地址与其他也是连续的
    for i := 0; i < 5; i++ {
        println(arg_val, foo_val1, foo_val2, foo_val3, foo_val4, foo_val5)
    }

    //返回foo_val3给main函数
    return foo_val3;
}


func main() {
    main_val := foo(666)

    println(*main_val, main_val)
}

```

我们将foo_val1-5全部用new的方式来开辟, 编译运行看结果

```bash
$ go run pro_3.go 
666 0xc000030728 0xc000030720 0xc00001a0e0 0xc000030738 0xc000030730
666 0xc000030728 0xc000030720 0xc00001a0e0 0xc000030738 0xc000030730
666 0xc000030728 0xc000030720 0xc00001a0e0 0xc000030738 0xc000030730
666 0xc000030728 0xc000030720 0xc00001a0e0 0xc000030738 0xc000030730
666 0xc000030728 0xc000030720 0xc00001a0e0 0xc000030738 0xc000030730
0 0xc00001a0e0
```

很明显, `foo_val3`的地址`0xc00001a0e0 `依然与其他的不是连续的. 依然具备逃逸行为.

### 四、结论

Golang中一个函数内局部变量，不管是不是动态new出来的，它会被分配在堆还是栈，是由编译器做逃逸分析之后做出的决定。



按理来说, 人家go的设计者明明就不希望开发者管这些,但是面试官就偏偏找这种问题问? 醉了也是.