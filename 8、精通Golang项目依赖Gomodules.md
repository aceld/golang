[TOC]

## 8、精通Golang项目依赖Go modules

### 一、什么是Go Modules?

Go modules 是 Go 语言的依赖解决方案，发布于 Go1.11，成长于 Go1.12，丰富于 Go1.13，正式于 Go1.14 推荐在生产上使用。



Go moudles 目前集成在 Go 的工具链中，只要安装了 Go，自然而然也就可以使用 Go moudles 了，而 Go modules 的出现也解决了在 Go1.11 前的几个常见争议问题：

1. Go 语言长久以来的依赖管理问题。
2. “淘汰”现有的 GOPATH 的使用模式。
3. 统一社区中的其它的依赖管理工具（提供迁移功能）。



### 二、GOPATH的工作模式

Go Modoules的目的之一就是淘汰GOPATH,  那么GOPATH是个什么?

为什么在 Go1.11 前就使用 GOPATH，而 Go1.11 后就开始逐步建议使用 Go modules，不再推荐 GOPATH 的模式了呢？

#### (1) Wait is GOPATH?

```bash
$ go env

GOPATH="/home/itheima/go"
...

```



我们输入`go env`命令行后可以查看到 GOPATH 变量的结果，我们进入到该目录下进行查看，如下：

```bash
go
├── bin
├── pkg
└── src
    ├── github.com
    ├── golang.org
    ├── google.golang.org
    ├── gopkg.in
    ....
```



GOPATH目录下一共包含了三个子目录，分别是：

- bin：存储所编译生成的二进制文件。
- pkg：存储预编译的目标文件，以加快程序的后续编译速度。
- src：存储所有`.go`文件或源代码。在编写 Go 应用程序，程序包和库时，一般会以`$GOPATH/src/github.com/foo/bar`的路径进行存放。

因此在使用 GOPATH 模式下，我们需要将应用代码存放在固定的`$GOPATH/src`目录下，并且如果执行`go get`来拉取外部依赖会自动下载并安装到`$GOPATH`目录下。



#### (2) GOPATH模式的弊端

在 GOPATH 的 `$GOPATH/src` 下进行 `.go` 文件或源代码的存储，我们可以称其为 GOPATH 的模式，这个模式拥有一些弊端.

* **A. 无版本控制概念.** 在执行`go get`的时候，你无法传达任何的版本信息的期望，也就是说你也无法知道自己当前更新的是哪一个版本，也无法通过指定来拉取自己所期望的具体版本。

- **B.无法同步一致第三方版本号.** 在运行 Go 应用程序的时候，你无法保证其它人与你所期望依赖的第三方库是相同的版本，也就是说在项目依赖库的管理上，你无法保证所有人的依赖版本都一致。
- **C.无法指定当前项目引用的第三方版本号. ** 你没办法处理 v1、v2、v3 等等不同版本的引用问题，因为 GOPATH 模式下的导入路径都是一样的，都是`github.com/foo/bar`。



### 三、Go Modules模式

我们接下来用Go Modules的方式创建一个项目, 建议为了与GOPATH分开,不要将项目创建在`GOPATH/src`下.

#### (1) go mod命令

| 命令            | 作用                             |
| :-------------- | :------------------------------- |
| go mod init     | 生成 go.mod 文件                 |
| go mod download | 下载 go.mod 文件中指明的所有依赖 |
| go mod tidy     | 整理现有的依赖                   |
| go mod graph    | 查看现有的依赖结构               |
| go mod edit     | 编辑 go.mod 文件                 |
| go mod vendor   | 导出项目所有的依赖到vendor目录   |
| go mod verify   | 校验一个模块是否被篡改过         |
| go mod why      | 查看为什么需要依赖某模块         |

#### (2) go mod环境变量

可以通过 `go env` 命令来进行查看

```bash
$ go env
GO111MODULE="auto"
GOPROXY="https://proxy.golang.org,direct"
GONOPROXY=""
GOSUMDB="sum.golang.org"
GONOSUMDB=""
GOPRIVATE=""
...
```



##### GO111MODULE

Go语言提供了 `GO111MODULE `这个环境变量来作为 Go modules 的开关，其允许设置以下参数：

- auto：只要项目包含了 go.mod 文件的话启用 Go modules，目前在 Go1.11 至 Go1.14 中仍然是默认值。
- on：启用 Go modules，推荐设置，将会是未来版本中的默认值。
- off：禁用 Go modules，不推荐设置。

可以通过来设置

```bash
$ go env -w GO111MODULE=on
```



##### GOPROXY

这个环境变量主要是用于设置 Go 模块代理（Go module proxy）,其作用是用于使 Go 在后续拉取模块版本时直接通过镜像站点来快速拉取。

GOPROXY 的默认值是：`https://proxy.golang.org,direct` 

`proxy.golang.org`国内访问不了,需要设置国内的代理.

* 阿里云

  https://mirrors.aliyun.com/goproxy/

* 七牛云

  https://goproxy.cn,direct



如:

 ```bash
$ go env -w GOPROXY=https://goproxy.cn,direct
 ```

GOPROXY 的值是一个以英文逗号 “,” 分割的 Go 模块代理列表，允许设置多个模块代理，假设你不想使用，也可以将其设置为 “off” ，这将会禁止 Go 在后续操作中使用任何 Go 模块代理。

如:

```bash
$ go env -w GOPROXY=https://goproxy.cn,https://mirrors.aliyun.com/goproxy/,direct
```



> direct

而在刚刚设置的值中，我们可以发现值列表中有 “direct” 标识，它又有什么作用呢？

实际上 “direct” 是一个特殊指示符，用于指示 Go 回源到模块版本的源地址去抓取（比如 GitHub 等），场景如下：当值列表中上一个 Go 模块代理返回 404 或 410 错误时，Go 自动尝试列表中的下一个，遇见 “direct” 时回源，也就是回到源地址去抓取，而遇见 EOF 时终止并抛出类似 “invalid version: unknown revision...” 的错误。



##### GOSUMDB

它的值是一个 Go checksum database，用于在拉取模块版本时（无论是从源站拉取还是通过 Go module proxy 拉取）保证拉取到的模块版本数据未经过篡改，若发现不一致，也就是可能存在篡改，将会立即中止。

GOSUMDB 的默认值为：`sum.golang.org`，在国内也是无法访问的，但是 GOSUMDB 可以被 Go 模块代理所代理（详见：Proxying a Checksum Database）。

因此我们可以通过设置 GOPROXY 来解决，而先前我们所设置的模块代理 `goproxy.cn` 就能支持代理 `sum.golang.org`，所以这一个问题在设置 GOPROXY 后，你可以不需要过度关心。

另外若对 GOSUMDB 的值有自定义需求，其支持如下格式：

- 格式 1：`<SUMDB_NAME>+<PUBLIC_KEY>`。
- 格式 2：`<SUMDB_NAME>+<PUBLIC_KEY> <SUMDB_URL>`。

也可以将其设置为“off”，也就是禁止 Go 在后续操作中校验模块版本。



##### GONOPROXY/GONOSUMDB/GOPRIVATE



这三个环境变量都是用在当前项目依赖了私有模块，例如像是你公司的私有 git 仓库，又或是 github 中的私有库，都是属于私有模块，都是要进行设置的，否则会拉取失败。

更细致来讲，就是依赖了由 GOPROXY 指定的 Go 模块代理或由 GOSUMDB 指定 Go checksum database 都无法访问到的模块时的场景。

而一般**建议直接设置 GOPRIVATE，它的值将作为 GONOPROXY 和 GONOSUMDB 的默认值，所以建议的最佳姿势是直接使用 GOPRIVATE**。

并且它们的值都是一个以英文逗号 “,” 分割的模块路径前缀，也就是可以设置多个，例如：

```bash
$ go env -w GOPRIVATE="git.example.com,github.com/eddycjy/mquote"
```

设置后，前缀为 git.xxx.com 和 github.com/eddycjy/mquote 的模块都会被认为是私有模块。

如果不想每次都重新设置，我们也可以利用通配符，例如：

```bash
$ go env -w GOPRIVATE="*.example.com"
```

这样子设置的话，所有模块路径为 example.com 的子域名（例如：git.example.com）都将不经过 Go module proxy 和 Go checksum database，**需要注意的是不包括 example.com 本身**。



### 四、使用Go Modules初始化项目

#### (1) 开启Go Modules

```bash
 $ go env -w GO111MODULE=on
```

又或是可以通过直接设置系统环境变量（写入对应的~/.bash_profile 文件亦可）来实现这个目的：

```bash
$ export GO111MODULE=on
```



#### (2) 初始化项目

创建项目目录

```bash
$ mkdir -p $HOME/aceld/modules_test
$ cd $HOME/aceld/modules_test
```

执行Go modules 初始化

```bash
$ go mod init github.com/aceld/modules_test
go: creating new go.mod: module github.com/aceld/modules_test
```

​	在执行 `go mod init` 命令时，我们指定了模块导入路径为 `github.com/aceld/modules_test`。接下来我们在该项目根目录下创建 `main.go` 文件，如下：

```go
package main

import (
    "fmt"
    "github.com/aceld/zinx/znet"
    "github.com/aceld/zinx/ziface"
)

//ping test 自定义路由
type PingRouter struct {
    znet.BaseRouter
}

//Ping Handle
func (this *PingRouter) Handle(request ziface.IRequest) {
    //先读取客户端的数据
    fmt.Println("recv from client : msgId=", request.GetMsgID(), 
              ", data=", string(request.GetData()))

    //再回写ping...ping...ping
    err := request.GetConnection().SendBuffMsg(0, []byte("ping...ping...ping"))
    if err != nil {
      fmt.Println(err)
    }
}

func main() {
    //1 创建一个server句柄
    s := znet.NewServer()

    //2 配置路由
    s.AddRouter(0, &PingRouter{})

    //3 开启服务
    s.Serve()
}

```

OK, 我们先不要关注代码本身,我们看当前的main.go也就是我们的`aceld/modules_test`项目,是依赖一个叫`github.com/aceld/zinx`库的. `znet`和`ziface`只是`zinx`的两个模块.

接下来我们在`$HOME/aceld/modules_test`,本项目的根目录执行 

```bash
$ go get github.com/aceld/zinx/znet

go: downloading github.com/aceld/zinx v0.0.0-20200221135252-8a8954e75100
go: found github.com/aceld/zinx/znet in github.com/aceld/zinx v0.0.0-20200221135252-8a8954e75100

```

我们会看到 我们的`go.mod`被修改,同时多了一个`go.sum`文件.



#### (3) 查看go.mod文件

> aceld/modules_test/go.mod

```go
module github.com/aceld/modules_test

go 1.14

require github.com/aceld/zinx v0.0.0-20200221135252-8a8954e75100 // indirect
```

我们来简单看一下这里面的关键字

`module`: 用于定义当前项目的模块路径

`go`:标识当前Go版本.即初始化版本

`require`: 当前项目依赖的一个特定的必须版本

`// indirect`: 示该模块为间接依赖，也就是在当前应用程序中的 import 语句中，并没有发现这个模块的明确引用，有可能是你先手动 `go get` 拉取下来的，也有可能是你所依赖的模块所依赖的.我们的代码很明显是依赖的`"github.com/aceld/zinx/znet"`和`"github.com/aceld/zinx/ziface"`,所以就间接的依赖了`github.com/aceld/zinx`



#### (4) 查看go.sum文件

在第一次拉取模块依赖后，会发现多出了一个 go.sum 文件，其详细罗列了当前项目直接或间接依赖的所有模块版本，并写明了那些模块版本的 SHA-256 哈希值以备 Go 在今后的操作中保证项目所依赖的那些模块版本不会被篡改。

```bash
github.com/aceld/zinx v0.0.0-20200221135252-8a8954e75100 h1:Ez5iM6cKGMtqvIJ8nvR9h74Ln8FvFDgfb7bJIbrKv54=
github.com/aceld/zinx v0.0.0-20200221135252-8a8954e75100/go.mod h1:bMiERrPdR8FzpBOo86nhWWmeHJ1cCaqVvWKCGcDVJ5M=
github.com/golang/protobuf v1.3.3/go.mod h1:vzj43D7+SQXF/4pzW/hwtAqwc6iTitCiVSaWz5lYuqw=
```

我们可以看到一个模块路径可能有如下两种：

 h1:hash情况

```bash
github.com/aceld/zinx v0.0.0-20200221135252-8a8954e75100 h1:Ez5iM6cKGMtqvIJ8nvR9h74Ln8FvFDgfb7bJIbrKv54=
```

go.mod hash情况

```bash
github.com/aceld/zinx v0.0.0-20200221135252-8a8954e75100/go.mod h1:bMiERrPdR8FzpBOo86nhWWmeHJ1cCaqVvWKCGcDVJ5M=
github.com/golang/protobuf v1.3.3/go.mod h1:vzj43D7+SQXF/4pzW/hwtAqwc6iTitCiVSaWz5lYuqw=
```

h1 hash 是 Go modules 将目标模块版本的 zip 文件开包后，针对所有包内文件依次进行 hash，然后再把它们的 hash 结果按照固定格式和算法组成总的 hash 值。

而 h1 hash 和 go.mod hash 两者，要不就是同时存在，要不就是只存在 go.mod hash。那什么情况下会不存在 h1 hash 呢，就是当 Go 认为肯定用不到某个模块版本的时候就会省略它的 h1 hash，就会出现不存在 h1 hash，只存在 go.mod hash 的情况。



### 五、修改模块的版本依赖关系

​		为了作尝试,假定我们现在都zinx版本作了升级, 由`zinx v0.0.0-20200221135252-8a8954e75100` 升级到 `zinx v0.0.0-20200306023939-bc416543ae24` (注意zinx是一个没有打版本tag打第三方库,如果有的版本号是有tag的,那么可以直接对应v后面的版本号即可)

​		那么,我们是怎么知道zinx做了升级呢, 我们又是如何知道的最新的`zinx`版本号是多少呢?

​		先回到`$HOME/aceld/modules_test`,本项目的根目录执行 

```bash
$ go get github.com/aceld/zinx/znet
go: downloading github.com/aceld/zinx v0.0.0-20200306023939-bc416543ae24
go: found github.com/aceld/zinx/znet in github.com/aceld/zinx v0.0.0-20200306023939-bc416543ae24
go: github.com/aceld/zinx upgrade => v0.0.0-20200306023939-bc416543ae24
```

这样我们,下载了最新的zinx, 版本是`v0.0.0-20200306023939-bc416543ae24`

​		然后,我么看一下go.mod

```go
module github.com/aceld/modules_test

go 1.14

require github.com/aceld/zinx v0.0.0-20200306023939-bc416543ae24 // indirect
```



我们会看到,当我们执行`go get` 的时候, 会自动的将本地将当前项目的`require`更新了.变成了最新的依赖.



好了, 现在我们就要做另外一件事,就是,我们想用一个旧版本的zinx. 来修改当前`zinx`模块的依赖版本号.

目前我们在`$GOPATH/pkg/mod/github.com/aceld`下,已经有了两个版本的zinx库

```bash
/go/pkg/mod/github.com/aceld$ ls
zinx@v0.0.0-20200221135252-8a8954e75100
zinx@v0.0.0-20200306023939-bc416543ae24
```

​		目前,我们`/aceld/modules_test`依赖的是`zinx@v0.0.0-20200306023939-bc416543ae24` 这个是最新版, 我们要改成之前的版本`zinx@v0.0.0-20200306023939-bc416543ae24`.

​		回到`/aceld/modules_test`项目目录下,执行

```bash
$ go mod edit -replace=zinx@v0.0.0-20200306023939-bc416543ae24=zinx@v0.0.0-20200221135252-8a8954e75100
```

​	然后我们打开go.mod查看一下

```go
module github.com/aceld/modules_test

go 1.14

require github.com/aceld/zinx v0.0.0-20200306023939-bc416543ae24 // indirect

replace zinx v0.0.0-20200306023939-bc416543ae24 => zinx v0.0.0-20200221135252-8a8954e75100
```



​	这里出现了`replace`关键字.用于将一个模块版本替换为另外一个模块版本。

