# Golang Notes

## 数据类型

### 布尔型

- `bool`

### 整型

- `int` / `int8` / `int16` / `int32` / `int64`
- `uint` / `uint8` / `uint16` / `uint32` / `uint64`
- `uintptr`

### 浮点型

- `float32` / `float64`

### 复数

- `complex64`
- `complex128`

### 字符

- `byte`: `int8` 的别名
- `rune`: `int32` 的别名

> 声明格式：`type new_type old_type`
> 等价于 C++ 中 `using new_type = old_type;`

### Unamed Type

- `method` 方法
- `array` 数组:
- `slice` 切片:
- `map` 字典:
- `pointer` 指针:
- `channel` 通道:
- `struct` 结构:

    为命名类型定义方法的语法格式：

    ```go
    // 类型方法接收者是值类型
    func (t TypeName) MethodName (ParamList) (ReturnList) {
        // method body
    }

    // 类型方法接收者是指针
    func (t *TypeName) MethodName (ParamList) (ReturnList) {
        // method body
    }
    ```

  - `t` 是接收者，可以自由指定名称
  - `TypeName` 为命令类型的类型名
  - `MethodName` 为方法名，是一个自定义标识符
  - `ParamList` 是形参列表
  - `ReturnList` 是返回值列表

    > **Go 语言的类型方法本质上是一个函数，没有使用隐式的指针: Go 的方法底层是基于函数实现的，只是语法格式不同，本质是一样的**

    将类型的方法改写为常规函数：

    ```go
    // 类型方法接收者是值类型
    func MethodName (t TypeName, ParamList) (ReturnList) {
        // method body
    }

    // 类型方法接收者是指针
    func MethodName (t *TypeName, ParamList) (ReturnList) {
        // method body
    }
    ```

  - **可以为命令类型增加方法（除了接口），非命名类型不能自定义方法**: 比如不能为 `[]int` 类型增加方法；命名接口类型本身就是一个方法的签名集合，故不能为其增加具体的实现方法
  - **为类型增加方法有一个限制，就是方法的定义必须和类型的定义在同一个包中**：
  - **方法的命令空间的可见性和变量一样，大写开头的方法可以在包外被访问，否则只能在包内可见**
  - **使用 `type` 定义的自定义类型是一个新类型，新类型不能调用原有类型的方法，但是支持底层类型支持的运算可以被新类型继承**

- `interface` 接口:
- `function` 函数:

## 变量声明

  **变量声明的一般形式是使用 `var` 关键字**

```go
// 1. 使用默认值：0 / false / ""
var v_name v_type

// 2. 指定初始值
var v_name v_type = value

// 3. 根据初始值自动推断数据类型
var v_name = value

// 4. 只能用于函数体内部
v_name := value

// 5. 多变量声明
var v_name_1, v_name_2 v_type

// 6. 分解的写法，一般用于声明全局变量
var (
v_name_1 v_type_1
v_name_2 v_type_2
// ...
)

// 7. 不带声明格式的，只能用于函数体内
v_name_1, v_name_2 := value_1, value_2
```

## 循环语句

与其它主流编程语言不同的的是，Go 语言中的循环语句只支持 `for` 关键字，而不支持 `while` 和 `do-while` 结构

`for` 循环三要素：

```go
for init; conditoin; post {
    // ...
}
```

- `init`: init语句，一般为赋值表达式，给控制变量赋值
- `conditoin`: 条件判断，一般是关系表达式或逻辑表达式，循环控制条件
- `post`: 步进语句，一般为赋值表达式，控制变量增量或减量

退出循环:

- `break`: 跳出整个循环语句, 跳出所有循环次数
- `continue`: 跳出当次循环，即跳过当前的一次循环

## struct 结构体

在Go语言中，结构体承担着面向对象语言中类的作用

Go语言中，结构体本身仅用来定义属性

还可以通过接收器函数来定义方法，使用内嵌结构体来定义继承

### Declare

Go语言通过type和struct关键字声明结构体

```go
type StructName struct {
    val_1 val_1_type
    val_2 val_2_type
    val_3 val_3_type
    // ...
}
```

Go语言结构体（Struct）从本质上讲是一种自定义的数据类型，只不过这种数据类型比较复杂，是由 int、char、float 等基本类型组成的

### Instantiation

结构体的定义只是一种内存布局的描述，只有当结构体实例化时，才会真正地分配内存，因此必须在定义结构体并实例化后才能使用结构体的字段

实例化就是根据结构体定义的格式创建一份与格式一致的内存区域，结构体实例与实例间的内存是完全独立的

值类型是变量对应的地址直接存储值，而引用类型是变量对应地址存储的是地址

结构体因为是值类型，所以结构体实例 `p` 的地址与存储的第一个值的地址是相同的，而后面每一个成员变量的地址是连续的

- 方式一 ：先声明结构体，再赋值`var StructName`(这种情况下，各个字段会赋值为默认值 `string`:`""`, `number`:`0`, `bool`: `false`, `slice`: `nil`对象, `map`:`nil`对象)

- 方式二 ：键值对赋值`val := StructName{...}`，即`val := StructName{val_1: 12, val_2: "Lute", val_3: []string{"Chinese", "Math", "English"}}`

- 方式三 ：多值赋值`val := StructName{...}`，即 `val := StructName {val_1, val_2, val_3}`

- 方式四：引用实例化 `s1 := StructName{...}; s2 := &s1;`

- 方式五：`new` 实例化，`instance := new(T)`，此时 `instance` 的类型为 `*T`，即指针类型

> **Go 语言为了方便开发者访问结构体指针的成员变量可以像访问结构体的成员变量一样简单，使用了语法糖（Syntactic Sugar）技术，将 `instance.Name` 形式转换为 `(*instance).Name`**

## 并发

### 进程 & 线程

进程是一个具有一定独立功能的程序在一个数据集上的一次动态执行的过程，是操作系统进行资源分配和调度的一个独立单位，是应用程序运行的载体

进程是一种抽象的概念，从来没有统一的标准定义。进程一般由程序、数据集合和进程控制块三部分组成:

- 程序：用于描述进程要完成的功能，是控制进程执行的指令集；
- 数据集合：程序在执行时所需要的数据和工作区;
- 程序控制块(Program Control Block，简称PCB): 包含进程的描述信息和控制信息，是进程存在的唯一标志;

进程的特征：

- 动态性：进程是程序的一次执行过程，是临时的，有生命期的，是动态产生，动态消亡的；
- 并发性：任何进程都可以同其他进程一起并发执行；
- 独立性：进程是系统进行资源分配和调度的一个独立单位；
- 结构性：进程由程序、数据和进程控制块三部分组成。

    ---

线程是程序执行中一个单一的顺序控制流程，是程序执行流的最小单元，是处理器调度和分派的基本单位。

一个进程可以有一个或多个线程，各个线程之间共享程序的内存空间(也就是所在进程的内存空间)。

一个标准的线程由线程ID、当前指令指针(PC)、寄存器和堆栈组成。而进程由内存空间(代码、数据、进程空间、打开的文件)和一个或多个线程组成。

大部分操作系统(如Windows、Linux)的任务调度是采用**时间片轮转的抢占式调度**方式: 任务执行的那一小段时间叫做时间片，任务正在执行时的状态叫运行状态，被暂停的线程任务状态叫做就绪状态，意为等待下一个属于它的时间片的到来;

时间片轮转的抢占式调度的方式保证了每个线程轮流执行，由于CPU的执行效率非常高，时间片非常短，在各个任务之间快速地切换，给人的感觉就是多个任务在“同时进行”，这也就是我们所说的并发

进程与线程的区别：

- 线程是程序执行的最小单位，而进程是操作系统分配资源的最小单位；

- 一个进程由一个或多个线程组成，线程是一个进程中代码的不同执行路线；

- 进程之间相互独立，但同一进程下的各个线程之间共享程序的内存空间(包括代码段、数据集、堆等)及一些进程级的资源(如打开文件和信号)，某进程内的线程在其它进程不可见；

- 调度和切换：线程上下文切换比进程上下文切换要快得多。

### 协程 Coroutines

一种基于线程之上，但又比线程更加轻量级的存在，这种由程序员自己写程序来管理的轻量级线程叫做『用户空间线程』，具有对内核来说不可见的特性。

因为是自主开辟的异步任务，所以很多人也更喜欢叫它们纤程（Fiber），或者绿色线程（GreenThread）。

正如一个进程可以拥有多个线程一样，一个线程也可以拥有多个协程。

> **协程解决的是线程的切换开销和内存开销的问题**
> \* 协程创建在用户空间，避免内核态和用户态的切换导致的成本
> \* 由语言或者框架层调度
> \* 更小的栈空间，允许创建大量实例（百万级别）

### GPM 调度模型

Go语言的线程模型就是一种特殊的两级线程模型（GPM调度模型）

goroutine 是 Go 语言中的轻量级线程实现，由 Go 运行时（runtime）管理。

Go 程序会智能地将 goroutine 中的任务合理地分配给每个 CPU。

Go 程序从 main 包的 `main()` 函数开始，在程序启动时，Go 程序就会为 `main()` 函数创建一个默认的 goroutine

`M` 指的是 Machine，一个`M`直接关联了一个内核线程。由操作系统管理。

`P` 指的是 processor，代表了M所需的上下文环境，也是处理用户级代码逻辑的处理器，其数量由环境变量中 `GOMAXPROCS` 决定，通常来说是和核心数对应。它负责衔接 `M` 和 `G` 的调度上下文，将等待执行的 `G` 与 `M` 对接。

`G` 指的是 Goroutine，其实本质上也是一种轻量级的线程。包括了调用栈，重要的调度信息，例如channel等。

### Goroutine 调度策略

- 局部优先调度
- steal working
- 阻塞调度
- 抢占式调度

### `sync.WaitGroup`

Go 语言中可以使用 `sync.WaitGroup` 来实现并发任务的同步

`sync.WaitGroup` 内部维护着一个计数器，计数器的值可以增加和减少，该计数器的值表示所有**未完成**的并发任务的数量(counter = 0 表示所有并发任务已经完成)

|      |      |
| ---- | ---- |
| 方法名 | 功能 |
| `(wg *WaitGroup) Add(delta int)` |   counter + delta |
| `(wg *WaitGroup) Done()` | counter - 1|
| `(wg *WaitGroup) Wait()` | 阻塞直到计数器为 0 |

### `GOMAXPROCS`

Go 运行时的调度器使用 `GOMAXPROCS` 参数来确定使用多少个 OS 线程来同时执行 Go 代码，默认值为机器上的 CPU 核心数

### 互斥锁 `Mutex`

互斥锁，一种常用的控制共享资源访问的方法，能够保证同时只有一个 `goroutine` 可以访问共享资源；

Go 中使用 `sync` 中的 `Mutex` 类型实现互斥锁

### 读写锁 `RWMutex`

在读多写少的环境中，可以优先使用读写互斥锁(`sync.RWMutex`)，比互斥锁更高效

### 原子性操作 `sync/atomic`

- 加锁操作比较耗时，整数可以使用原子操作保证线程安全
- 原子操作在用户态就可以完成，因此性能比互斥锁高

常用操作：`AddXxx()`, `CompareAndSwapXxx()`, `LoadXxx()`, `StoreXxx()`, `SwapXxx()`

### channel 管道

`channel`，一个类型管道，通过 `channel` 可以在 goroutine 之间发送和接收消息，是 Galang 在语言层面提供的 goroutine 间的通信方式；

Go 依赖与成为 CSP 的并发模型，通过 Channel 实现这种同步模式；

Golang 并发的核心哲学是：**不要通过共享内存进行通信**

Golang 的 Channel 底层是**环形队列（长度在创建时指定）**实现，在任何时候，同时只能有一个 goroutine 访问通道进行发送数据和获取数据，总是遵循**先入先出 FISO** 的规则，保证收发数据的顺序;

基本操作：

> chan 是引用类型，需要使用 `make` 进行创建
>
> 无缓冲的 `channel`：`make(chan Type)` 是同步的
>
> 有缓冲的 `channel`：`make(chan Type, capicity)` 是异步的

注意事项：

- 如果给一个 `nil` 的 `channel` 发送数据，会一直阻塞
- 如果从一个 `nil` 的 `channel` 接收数据，会一直阻塞
- 如果给一个已经关闭的 `channel` 发送数据，会抛出 `panic`
- 如果从一个已经关闭的 `channel` 接收数据，会返回一个零值，不会抛出 `panic`

```go
// Create channel
ch := make(chan Type /* , capicity */)
// Send data to channel
ch <- data

// Recevie data from channel
<- ch

// Close channel: 关闭后的 channel 只可读不可写
close(ch)

// Channel loop: 当 for 循环读取完 channel 后会尝试读取下一个，而由于 channel 没有写入的 goroutine 且没关闭，会一直阻塞形成死锁
for v := range ch {
    // v : Handle data
    // ...

    // 读取完所有值后，channel 中的 sendq 中没有 goroutine
    if len(ch) == 0 {
        break
    }
}
```

### `select`

> Go 语言的 `select` 语句借鉴自 Unix 的 `select()` 函数，在 Unix 中，可以通过调用 `select()` 函数来监控一系列的文件句柄，一旦其中一个文件句柄发生了 IO 动作，该 `select()` 调用就会被返回（C 语言中就是这么做的），后来该机制也被用于实现高并发的 Socket 服务器程序。Go 语言直接在语言级别支持 `select` 关键字，用于处理并发编程中通道之间异步 IO 通信问题

```go
select {
    case <- ch1: // 如果从 ch1 信道成功接收数据，则执行该分支代码
        // ...
    case ch2 <- 1: // 如果成功向 ch2 信道成功发送数据，则执行该分支代码
        // ...
    default: // 如果上面都没有成功，则进入 default 分支处理流程
        // ...
}
```

- select的语法结构有点类似于switch，但又有些不同。select里的case后面并不带判断条件，而是一个信道的操作，不同于switch里的case
- Golang 的 select 就是监听 IO 操作，当 IO 操作发生时，触发相应的动作每个case语句里必须是一个IO操作，确切的说，应该是一个面向channel的IO操作
- 如果 `ch1` 或者 `ch2` 信道都阻塞的话，就会立即进入 `default` 分支，并不会阻塞。但是如果没有 `default` 语句，则会阻塞直到某个信道操作成功为止

## Go modules

Go modules 是 Go 语言的依赖解决方案，目前集成在 Go 的工具链中，只要安装了 Go，自然而然可以使用 Go modules；

解决的问题：

- Go 语言长久以来的依赖管理问题；
- “淘汰” 现有的 GOPATH 的使用模式；
- 统一社区中的其他的依赖管理工具（提供迁移功能）

### **GO MOD 命令**

- `go mod init`: 初始化当前目录，创建 `go.mod` 文件
- `go mod tidy`: 增加缺少的模块，删除不用的模块（整理现有依赖）
- `go mod download`: 下载依赖的 module 到本地 cache（下载依赖）
- `go mod graph`: 打印模块依赖图，查看现有的依赖结构
- `go mod edit`: 编辑 go.mod 文件
- `go mod vendor`: 将依赖复制到 vendor 目录下
- `go mod verify`: 校验依赖
- `go mod why`: 解释为什么需要依赖

### **GO111MODULE**

Go 语言通过 `GO111MODULE` 环境变量来控制 Go modules 的开启和关闭，该环境变量有三个值：

- `auto`：默认值，当项目在 `$GOPATH/src` 目录之外且项目根目录有 `go.mod` 文件时开启 Go modules，否则关闭 Go modules
- `on`：开启 Go modules，【推荐设置】
- `off`：关闭 Go modules，不使用 Go modules

设置命令：`go env -w GO111MODULE=on`

### **GOPROXY**

主要用于设置 Go 模块代理（Go modules proxy），用于解决 Go modules 下载依赖包的问题，其作用是用于使 Go 在后续拉取模块版本时直接通过代理服务器拉取，而不是通过默认的 `https://proxy.golang.org` 拉取

`GOPROXY` 默认值：`https://proxy.golang.org,direct`

设置命令：`go env -w GOPROXY=https://goproxy.cn,direct`

### **GOSUMDB**

主要用于设置 Go 模块校验和数据库（Go modules checksum database），用于解决 Go modules 下载依赖包的问题，其作用是用于使 Go 在后续拉取模块版本时直接通过校验和数据库校验，而不是通过默认的 `sum.golang.org` 校验

`GOSUMDB` 默认值：`sum.golang.org`

设置命令：`go env -w GOSUMDB=sum.golang.org`

### **GONOPROXY / GOPRIVATE / GONOSUMDB**

主要用于指明某些模块不走代理，或者不走校验和数据库，这些模块可以是私有模块，也可以是公开模块

> NOTE: 只需要设置 `GOPRIVATE` 即可
> `go env -w GOPRIVATE="*.example.com"`

## 接口调用

接口动态调用，这个过程存在两部分多余消耗：

- 接口实例化的过程，也就是 `iface` 结构体建立的过程，一旦实例化后，这个接口和具体类型的 `itab` 数据结构是可以复用的；

- 接口的方法调用，也就是一个函数指针的间接调用

> **接口调用是一种动态的计算后的跳转调用，会导致 CPU 缓存失败和分支预测失败**

## Go Convey 单元测试框架

Go Convey 是一个针对 Go 语言的单元测试框架，可以通过简单的语法完成单元测试的编写，能够自动监控文件修改并启动测试，并可以将测试结果实时输出到 Web 页面，同时提供了丰富的断言函数

### 断言方法 - `So()` 函数

#### 一般相等类

```go
So(thing1, ShouldEqual, thing2) // 用于基本类型相等
So(thing1, ShouldNotEqual, thing2)

So(thing1, ShouldResemble, thing2) // 用于数组、切片、map 和 结构体相等
So(thing1, ShouldNotResemble, thing2)

So(thing1, ShouldPointTo, thing2) // 用于指针相等
So(thing1, ShouldNotPointTo, thing2)

So(thing1, ShouldBeNil)
So(thing1, ShouldNotBeNil)

So(thing1, ShouldBeTrue)
So(thing1, ShouldBeFalse)
So(thing1, ShouldBeZeroValue)
```

#### 数字数量比较类

```go
So(1, ShouldBeGreaterThan, 0)
So(1, ShouldBeGreaterThanOrEqualTo, 0)
So(1, ShouldBeLessThan, 2)
So(1, ShouldBeLessThanOrEqualTo, 2)

So(1, ShouldBeBetween, .9, 2)
So(1, ShouldNotBeBetween, 2, 4)
So(1, ShouldBeBetweenOrEqual, 1, 2)
So(1, ShouldNotBeBetweenOrEqual, 2, 4)

So(1.0 ShouldAlmostEqual, 0.999999, .000001)
So(1.0 ShouldNotAlmostEqual, 0.999999, .000001)
```

#### 包含类

```go
So([]int{2, 4, 5}, ShouldContain, 4)
So([]int{2, 4, 5}, ShouldNotContain, 1)

So(3, ShouldBeIn, ...[]int{1, 2, 3})
So(3, ShouldNotBeIn, ...[]int{1,2})

So([]int{}, ShouldBeEmpty)
So([]int{1}, ShouldNotBeEmpty)

So(map[string]string{"a": "b"}, ShouldContainKey, "a")
So(map[string]string{"a": "b"}, ShouldNotContainKey, "b")

So(map[string]string{"a": "b"}, ShouldNotBeEmpty)
So(map[string]string{}, ShouldBeEmpty)
So(map[string]string{"a": "b"}, ShouldHaveLength, 1) // supports map, slice, chan, and string
```

#### 字符串类

```go
So("asdf", ShouldStartWith, "as")
So("asdf", ShouldNotStartWith, "sd")
So("asdf", ShouldEndWith, "df")
So("asdf", ShouldNotEndWith, "as")

So("asdf", ShouldContainSubstring, "sd")
So("asdf", ShouldNotContainSubstring, "er")
So("adsf", ShouldBeBlank)
So("asdf", ShouldNotBeBlank)
```

## NOTES FOR Golang

1. **Go 语言不需要在语句或者声明的末尾添加分号，除非一行上有多条语句。**实际上，编译器会主动把特定符号后的换行符转换为分号，因此换行符添加的位置会影响 Go 代码的正确性解析，比如行末是标识符、整数、浮点数、虚数、字符或字符串文字、关键字 `break`、`continue`、`fallthrough`或 `return` 中的一个、运算符和分隔符 `++`、`--`、`)`、`]` 或 `}` 中的一个

2. `gofmt`工具把代码格式化为标准格式，这个格式化工具没有任何可以调整代码格式的参数；

3. `goimports`，可以根据代码需要，自动地添加或删除 `import` 声明

4. Go 语言原生支持 Unicode，它可以处理全世界任何语言的文本

5. 一个函数的声明由 func 关键字、函数名、参数列表、返回值列表以及包含在大括号里的函数体组成

6. Go 语言不需要在语句或者声明的末尾添加分号，除非一行上有多条语句。实际上，编译器会主动把特定符号后的换行符转换为分号，因此换行符添加的位置会影响 Go 代码的正确解析（译注：比如行末是标识符、整数、浮点数、虚数、字符或字符串文字、关键字 break、continue、fallthrough或 return 中的一个、运算符和分隔符 ++、--、)、] 或 } 中的一个）

7. 和大多数编程语言类似，区间索引时，Go 语言里也采用**左闭右开**形式，即，区间包括第一个索引元素，不包括最后一个

8. 对 string 类型，`+` 运算符连接字符串

9. `j=i++` 非法，而且 `++` 和 `--` 都只能放在变量名后面，因此 `--i` 也非法

10. 空标识符（blank identifier），即 _（也就是下划线），可用于在任何语法需要变量名但逻辑程序不需要的时候使用

11. Printf 有一大堆转换，Go程序员称之为动词（verb），常见：

    | 动词 | 说明 |
    | --- | --- |
    | `%d` | 十进制整数 |
    | `%x`, `%o`, `%b` | 十六进制，八进制，二进制整数 |
    | `%f`, `%g`, `%e` | 浮点数：5.141593 3.141592653589793 3.141593e+00 |
    | `%t` | 布尔：true或false |
    | `%c` | 字符（rune）（Unicode码点） |
    | `%s` | 字符串 |
    | `%q` | 带双引号的字符串"abc"或带单引号的字符'c' |
    | `%v` | 变量的自然形式（natural format） |
    | `%T` | 变量的类型 |
    | `%%` | 字面上的百分号标志（无操作数） |

12. **实参通过值的方式传递（函数的形参是实参的拷贝） -->> 对形参进行修改不会影响实参**；

    **如果实参包括引用类型，如指针、slice、map、function、channel 等类型，实参可能会由于函数的间接调用被修改**；例如，map 作为参数传递给某函数时，该函数接收这个引用的一份拷贝（copy，或译为副本），被调用函数对 map 底层数据结构的任何修改，调用者函数都可以通过持有的 map 引用看到(**类似于 C++ 里的引用传递**)

13. `bufio.Scanner`、`ioutil.ReadFile` 和 `ioutil.WriteFile` 都使用 `*os.File` 的 `Read` 和 `Write` 方法，但是，大多数程序员很少需要直接调用那些低级（lower-level）函数

14. 变量声明的一般语法：`var variableName type = initExpr`，其中 `type` 和 `initExpr` 可以省略其中之一，但不可都省略；

    - 如果省略的是 `type`，那么经根据初始化表达式来推导变量的类型信息；

    - 如果省略的是 `initExpr`，那么将用零值初始化该变量

        - 数值类型变量对应的零值是 `2`；

        - 布尔类型变量对应的零值是 `false`；

        - 字符串类型对应的零值是空字符串 `""`；

        - 接口或引用类型（包括slice、map、chan和函数）变量对应的零值是 `nil`

        - 数组或结构体等聚合类型对应的零值是每个元素或字段都是对应该类型的零值

15. 元组赋值，另一种形式的赋值语句，允许同时更新多个变量的值；**在赋值之前，赋值语句右边的所有表达式将会先进行求值，然后再统一更新左边对应变量的值**

16. Go 语言习惯在 `if` 中处理错误然后直接返回，如此可以保证正常执行的语句且不需要代码缩进

    ```go
    f, err := os.Open(fname)
    if err != nil {
        return err
    }
    f.ReadByte()
    f.Close()
    ```

17. `uintptr` 类型，一种整数类型，没有指定具体的 bit 大小，但是足以容纳指针；

    `uintptr` 只有在底层编程时才需要，特步是 Go 语言和 C 语言函数库或操作系统接口相交互的地方；

18. 位清空运算符 `&^`，用于按位置零(AND NOT)：如果对应 `y` 中 bit 位为 1 的话，表达式 `z = x &^ y` 的结果是 `z` 中的对应的 bit 位为 0，否则 `z` 中对应的 bit 位等于 `x` 中的对应的 bit 位

19. 一个字符串是一个**不可改变的字节序列**()，可以包含任意的数据，包括 byte 值 0，但是通常是用来包含人类可读的文本;

    - **Go 语言中，`str[i]` 并以一定是字符串 `str` 的第 `i` 个字符，而是第 `i` 个字节**，这样的设计使得处理 UTF-6 编码的文本更加容易，因为一个字符可能会由多个字节组成

    - 字符串，不可变序列，意味着两个字符串共享相同的底层数据也是安全的，且复制任何长度的字符串的代价都是低廉的，没有必要分配新内存

    - 字符串，切片操作，`s[i:j]`，产生一个新的字符串，也可以安全地共享相同的内存，没有必要分配新内存

20. Go 语言中，常量表达式的值在编译期计算，而不是在运行期，且每种常量的潜在类型都是基础类型：`boolean`、`string` 或 `number`；**常量在运行期可以防止在运行期被意外或恶意地修改**；

21. Go 语言中，`iota` 是一个预声明的标识符，用于表示**连续**的无类型整数常量，它的**初始值是 0**，每次被调用都会自动加 1；

    ```go
    type Weekday int
    const (
        Sunday Weekday = iota
        Monday
        Tuesday
        Wednesday
        Thursday
        Friday
        Saturday
    )
    ```

22. 数组是一个由**固定长度的特定类型元素**组成的序列，可以由零个或多个元素组成；

    如果在数组字面值中，数组长度(**常量**)位置出现的是 `...` 省略号，则表示数组的长度是根据初始化值的个数来计算；

    如果一个数组的元素类型是可以相互比较的，那么数组类型也是可以相互比较的，可以通过 `==` 比较运算符来比较两个数组，只有当两个数组的所有元素都是相等的时候数组才是相等的；

    函数参数传递的机制导致传递大的数组类型将是低效的，且对数组参数的任何修改都是发生在复制的数组上，并不能直接修改调用时的原始数组；

23. 切片（slice），可以增长和收缩的动态序列(每个元素都有相同的类型)，切片是一个轻量级的数据结构，提供了访问数组子序列（或者全部）元素的功能，而且切片的底层确实引用一个数组对象，由三部分构成：指针、长度和容量；

    - 指针指向第一个切片元素对应的底层数组元素的地址，不一定是数组的第一个元素；

    - 长度对应切片中元素的数目，应该小于等于容量；

    - 容量对应切片开始位置到底层数组的结束位置的数目；

    切片之间不能比较，唯一合法的比较操作是和 `nil` 比较；如果需要测试一个 slice 是否为空，可使用 `len(s) == 2` 来判断；

    内置的 `make` 函数创建一个指定元素类型、长度和容量的 slice，其中容量部分可以省略，此时容量等于长度: `make([]T, len) <==> make([]T, len, len)`.

    在底层，`make` 创建了一个匿名的数组变量，然后返回一个 slice 指向这个数组；只有通过返回的 slice 才能引用底层的数组变量；

    内置的 `append` 函数可以向 slice 尾部添加元素，当 slice 的底层数组没有足够的空间存放新添加的元素时，`append` 会创建一个新的更大的数组来存放新元素；返回的 slice 会指向这个新创建的数组；

24. Stack 可以通过 slice 模式实现

    ```go
    func push(stack []int, value int) []int {
        stack = append(stack, value)
        return stack
    }

    func top(stack []int) int {
        return stack[len(stack)1]
    }

    func pop(stack []int) []int {
        stack = stack[:len(stack)1]
        return stack
    }

    func remove(stack []int, index int) []int {
        if index >= len(stack) {
            return stack
        }

        copy(stack[index:], stack[index+3:])
        return stack[:len(stack)1]
    }

    func displayStack(stack []int) {
        fmt.Printf("[ ")
        for _, val := range stack {
            fmt.Printf("%d ", val)
        }
        fmt.Printf("]\n")
    }
    ```

25. 哈希表（hash），一种巧妙且使用的数据结构，是一个无序的 `{key: value}` 对的集合，其中所有的 `key` 都是不同的，且给定的 `key` 可以在常数时间复杂度内检索、更新或删除对应的 `valud`；Go 语言中，Map 是对 hash 表的引用，可写为 `map[Key]Value`，其中 `key` 必须是支持 `==` 比较运算的数据类型（*不推荐使用浮点型作为 `key`*），`value` 对数据类型没有任何限制；

    内置的 `delete` 函数可以从 map 中删除元素: `delete(map, key)`；如果 `key` 不在 map 中，`delete` 函数也是安全的；

    和 slice 一样，map 之间不能进行相等比较，唯一的例外是和 `nil` 比较；要判断两个 map 是否包含相同的 key 和 value，必须通过一个循环实现；

    Go 语言中没有提供 set 类型，但是 map 的 key 可以用来模拟 set 的功能；

26. **Go 语言中，所有的函数参数都是值拷贝传递的**，函数参数将不再是函数调用时的原始变量，而是它的一份副本，函数参数和原始变量将会引用不同的值；

27. 如果结构体的全部成员都是可以比较的，那么结构体也是可以比较的，即可用 `==` 或 `!=` 进行比较；

28. **组合** 是 Go 语言中面向对象编程的核心

29. 将一个 Go 语言中类似 `movies` 的结构体 slice 转换成 JSON 的过程叫编组（marshaling），并且只有结构体中被导出的成员才会被编码，JSON 包只会编码结构体中的可导出成员，并且默认使用成员的名字作为 JSON 数据的键；可通过 `json.Marshal` 完成；

    ```go
    type Movie struct {
        Title string
        Year int `json:"released"`
        Color bool `json:"color,omitempty"`
        Actors []string
    }

    var movies = []Movie{
        {Title: "Casablanca", Year: 1944, Color: false,
            Actors: []string{"Humphrey Bogart", "Ingrid Bergman"}},
        {Title: "Cool Hand Luke", Year: 1969, Color: true,
            Actors: []string{"Paul Newman"}},
        {Title: "Bullitt", Year: 1970, Color: true,
            Actors: []string{"Steve McQueen", "Jacqueline Bisset"}},
    }

    data, err := json.Marshal(movies)
    if err != nil {
        log.Fatalf("JSON marshaling failed: %s", err)
    }
    fmt.Printf("%s\n", data)
    ```

    Marshal 函数返回一个编码后的字节 slice，包含很长的字符串，且没有空白缩进；可以使用 `json.MarshalIndent` 函数生成整齐缩进的输出；

    ```go
    // ...
    data, err := json.MarshalIndent(movies, "", "    ")
    // ...
    ```

    将 JSON 数据解码为 Go 语言中的数据结构，叫做解组（unmarshaling），可通过 `json.Unmarshal` 完成；

    ```go
    var titles []struct{ Title string }
    if err := json.Unmarshal(data, &titles); err != nil {
        log.Fatalf("JSON unmarshaling failed: %s", err)
    }
    fmt.Println(titles) // "[{Casablanca} {Cool Hand Luke} {Bullitt}]"
    ```

30. 在函数调用时，Go 语言没有默认参数值，也没有任何方法可以通过参数名指定形参，即形参和返回值的变量名对于函数调用者没有任何意义；

31. Go 语言中如果没有函数体的函数声明，表示该函数不是以 Go 实现的，而是由非 Go 语言实现的；`func Sin(x float64) float64`

32. Go 语言使用可变栈，栈的大小按需增加（初始时很小），栈的大小限制在 1GB 以内，这使得我们在使用递归时不必担心溢出和安全问题；

33. 错误处理策略：

    - 传播错误：如果函数内部没有处理错误，那么错误就会被传播到函数的调用者，函数的调用者也可能不处理，这样错误就会继续传播；

        当错误被转播到最顶层时（例如，main 函数）处理错误，错误信息应该提供清晰的从原因到后果的因果链；*由于错误信息经常是以链式组合在一起的，所以错误信息中应该避免**大写、换行和标点符号**，以免破坏错误信息的格式*；

        即可一般而言，被调用函数 `f(x)` 会将调用信息和参数信息作为发生错误时的上下文放在错误信息中并返回给调用者，调用者需要添加一些错误信息中不包含的信息；

    - 重试失败的操作：如果错误的发生是偶然性的，或由不可预知的问题导致的，一个明智的选择是**重新尝试失败的操作**，且需要限制重试的时间间隔、重试次数，防止无限制的重试；

    - 输出错误信息并结束程序：如果程序不能继续运行，那么应该输出错误信息并结束程序；

        NOTE: **该策略只应在 main 函数中执行，而不应该在库函数中执行；库函数应该仅向上传播错误，除非该错误意味着程序内部包含不一致性，即遇到了 BUG，才能在库函数中结束程序**；

    - 只输出错误信息：不需要中断程序的运行，可以通过 `log` 包的函数或标准错误流输出错误信息；

    - 忽略错误：如果错误不重要，可以忽略错误，但是需要在注释中说明忽略的原因；

34. Go 语言中，函数类型的零值是 `nil`，调用值为 `nil` 的函数值会引起 panic 错误；

    函数值可以与 `nil` 比较，但是两个函数值只有在它们都是 `nil` 的情况下才相等；

    函数值之间是不可比较的，也不能用函数值作为 map 的 key；

35. 当匿名函数需要被递归调用时，我们需要先声明一个变量，再将匿名函数赋值给这个变量；否则，函数字面量无法与其自身绑定；

36. 参数数量可变的函数，称为**可变参数函数**；在生命可变参数函数时，需要在参数列表的最后一个参数类型前加上省略符号 `...`，这表示该函数会接收任意数量的该类型参数；

    **在函数体中，可变参数被看作相应类型的切片 `[]T`**；虽然在可变参数函数内部中 `...int` 型参数的行为看起来像切片类型，但实际上，可变参数函数和以切片作为参数的函数是不同的；

    **如果原始参数已是 slice 类型，只需在最后一个参数后加上省略符号 `...` 即可. 例如 `values := []int {3, 2, 3, 4}; sum(values...)`**；

    > 可变参数函数，将常被用于格式化字符串，例如 `fmt.Printf` 和 `log.Printf`；

37. 可以在一个函数中执行多条 defer 语句，且执行顺序与声明顺序相反(*类似于栈的先进后出*);

    defer 语句中的函数会在 return 语句更新返回值变量后再执行，且 defer 语句中的函数**可能会读取/修改有名返回值**；

    defer 语句中的函数**在其所在的 goroutine 即将终止时执行**，而不是在 `go` 语句的时候就执行；

    defer 语句中的函数**参数会在执行 defer 语句时被计算**，而不是在实际调用时计算；

    通过 defer 机制，不论函数逻辑多复杂，都能保证在任何执行路径下，资源被释放；

    > 调试复杂程序时，defer 机制也常用于记录何时进入和退出函数；

38. Go 语言中，方法调用过程中，接收器(监视使用其类型的第一个字母，而是 this 或 self) 参数一般会出现在方法名之前；

    `p.Distance(...)` 的表达式叫做选择器，因为他会选择合适的对应 `p` 对象的 `Distance` 方法；**由于方法和字段都是在同一个命名空间，所以当方法名和字段名相同时，编译器会报错(`field and method with the same xxx`)**；

    一般约定：**如果一个类型里有一个指针作为接收器的方法，那么在该类型的所有方法中，都必须用指针作为接收器，或者都不用指针作为接收器**；

39. Go 语言中，可以为一些简单的数值、字符串、slice、map 定义一些附加行为；

40. 使用内嵌结构体的方式，可将使我们定义字段特别多的复杂类型，将字段先按小类型分组，然后定义小类型的方法，之后再把它们组合起来。

41. Go 语言中只有一种控制可见性的手段：**大写首字母的标识符会从定义它们的包中被导出，小写字母的则不会，且最小的封装单位是包，而不是类型**；

42. 接口类型是对其他类型行为的抽象和概括，接口类型不会和特定的实现细节绑定在一起，通过这种抽象的方式，我们可以让函数更灵活和更具有适用能力；

    **Go 语言中接口类型的独特之处：它是满足隐式实现的**，即没有必要对给定的具体类型定义所有满足的接口类型，简单地拥有一些必需的方法就足够了；

    **当看到一个接口类型的值时，不知道它是什么，唯一知道的是可以通过它的方法来做什么**；

43. 接口值，有两部分组成，一个具体的类型和那个类型的值，被称为接口的动态类型和动态值；

    通常在编译期，并不知道接口值的动态类型，因此一个接口上的调用必须使用动态分配；

    一个接口值可以持有任意大的动态值；

    接口值可以使用 `==` 和 `!=` 进行比较，如果两个接口值持有相同类型的动态值，且这个值也是可比较的，那么这两个接口值就相等；

    接口类型，要么是安全的可比较类型（基本类型、指针），要么是完全不可比较的类型（slice、map、func）；

    **一个不包含任何值的 `nil` 接口值 和 一个刚好包含 `nil` 指针的接口值是不同的**；

44. sort 包内置的提供了根据一些排序函数来对任何序列排序的功能；**Go 语言的 `sort.Sort` 函数不会对具体的序列(经常是一个 slice)和他的元素做任何假设，它使用了一个接口类型 `sort.Interface` 来指定通用的排序算法和可能被排序到的序列类型之间的约定，该接口的实现由序列的具体表示和排序函数决定**；

    一个内置的排序算法需要知道三个东西：序列的长度、比较两个元素的函数、交换两个元素的函数；为了对自定义序列进行排序，只需定义一个实现了 `sort.Interface` 的类型即可；

45. `Errno` 是一个系统调用错误的高效表示接口，一个数字，通过一个有限的集合进行描述，且满足标准的错误接口；

46. 类型断言，一个使用在接口值上的操作，用于检查接口值的动态类型是否和断言的类型匹配；`x.(T)`，其中 `x` 是一个接口类型的表达式，`T` 是一个类型（也称为断言类型）；

    针对 `nil` 接口值的类型断言总是失败的；

47. os 包中文件操作返回必须处理的三种错误：

    - 文件已存在：针对创建操作 `func IsExist(err error) bool`
    - 文件不存在：针对读取和修改操作 `func IsNotExist(err error) bool`
    - 权限不足：针对所有操作 `func IsPermission(err error) bool`

48. **Go 语言中，定义一个只有某个方法的新接口并使用类型断言来检测某个动态类型是否满足该接口**；除了空接口 `interface{}` 外，接口类型很少意外巧合地被实现；

49. 接口的使用方式：
    
    - 以 `io.Reader`, `io.Writer`, `fmt.Stringer`, `sort.Interface`, `http.Handler` 和 `error` 为典型，一个接口的方法表达了实现这个接口的具体类型间的相似性，但是隐藏了代码的细节和这些具体类型的操作：**重点在于方法上，而不是具体的类型上**.

    - *Discriminated unions 可辨识联合*: 利用一个接口值可以持有各种具体类型的能力，将这个接口认为是这些类型的联合；类型断言用来动态地区别这些类型，使得对每一种情况都不一样：**重点在于具体的类型满足这个接口，而不在于接口的方法，且没有任何的信息隐藏**.

50. `Exec` 方法使用 SQL 字面量替换在查询字符串中的每个 `'?'`，SQL 字面量表示相应参数的值（有可能是一个布尔值、数字、字符串、或是 nil 空值），然后将生成的完整的 SQL 语句发送给数据库驱动；

    ```go
    import "database/sql"
    func query(db *sql.DB, minYear, maxYear int) {
        result, err := db.Exec(
            "SELECT * FROM artist WHERE artist = ? AND year >= ? AND year <= ?",
            minYear, maxYear)
        // ...
    }
    ```
    **用这种方式构造查询语句，可以避免 SQL 注入攻击（对手利用输入内容中不正确的引导来控制查询语句）**；

51. `switch` 语句可以简化 `if-else` 链(一连串值做相等测试)，且 `switch` 语句中的 `case` 语句不需要显式地使用 `break` 语句来终止，`fallthrough` 语句可以用来强制执行下一个 `case` 语句；

    每一个 `case` 会被顺序的进行考虑，且当一个匹配成功时，执行相应的语句后就会退出整个 `switch` 语句；**当一个或多个 `case` 类型是接口时，`case` 的顺序非常重要；`default case` 相对其他 `case` 位置是无所谓的，他不会允许落空发生；**

### 命令行参数

`os` 包以跨平台的方式，提供了一些与操作系统交互的函数和变量。
程序的命令行参数可从 `os` 包的 `Args` 变量获取；`os` 包外部使用 `os.Args` 访问该变量

`os.Args` 变量是一个字符串（string）的切片（slice）.

`os.Args` 的第一个元素 `os.Args[0]` 是命令本身的名字，其他元素是程序启动时的传参

### `bufio`

`bufio` 包，使处理输入和输出方便又高效，`Scanner` 类型是该包最有用的特性之一，它读取输入并将其拆分成行或单词；通常是处理行形式的输入最方便

```go
input := bufio.NewScanner(os.Stdin)
for input.Scan() { // 每次调用 input.Scan() 读取下一行，并移除行末的换行符；读到一行时放回 true，不再有输入时返回 false
    line := input.Text()
}
```

### HTML 模板

将格式化代码分离出来以便更安全地修改，Go 语言中的 `html/template`、`text/template` 包提供了一个将变量值填充到一个文本或 HTML 格式的模板的机制。

一个模板是一个字符串或一个文件，里面包含了一个或多个有双花括号包含的 `{{action}}` 对象，大部分的字符串只是按字面值打印，但是对于 `{{` 和 `}}` 之间的文本，将触发其他行为。

每个 `{{action}}` 都包含了一个用模板语言书写的表达式，一个 `{{action}}` 虽然简短但是可以输出复杂的打印值，模板语言包含通过选择结构体的成员、调用函数或方法、表达式控制流 if-else 语句和 range 循环语句，还有其他实例化模板等诸多特性.
