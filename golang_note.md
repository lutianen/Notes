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
- 方式无：`new` 实例化，`instance := new(T)`，此时 `instance` 的类型为 `*T`，即指针类型

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

    ---

大部分操作系统(如Windows、Linux)的任务调度是采用**时间片轮转的抢占式调度**方式: 任务执行的那一小段时间叫做时间片，任务正在执行时的状态叫运行状态，被暂停的线程任务状态叫做就绪状态，意为等待下一个属于它的时间片的到来;

时间片轮转的抢占式调度的方式保证了每个线程轮流执行，由于CPU的执行效率非常高，时间片非常短，在各个任务之间快速地切换，给人的感觉就是多个任务在“同时进行”，这也就是我们所说的并发

    ---

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
