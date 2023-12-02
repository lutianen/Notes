# Interview Notes

## TW2 - 中视场彩色相机地检系统

### PL 端接口

  **AD 模块**，SPI（Serial Peripheral Interface）接口，采集 AD 芯片原始值

  **GPIO 模块**，输出开机指令、关机指令、镜头加热等数字量

  **串口模块**，RS422 接口，通过全双工RS-485收发器芯片扩展2路RS-422接口，其中一路RS-422用于与相机的通信，另一路作为预留和测试使用；使用 TI 公司的 RS485 扩展

  **LVDS 模块**，实现相机 LVDS 数据采集、数组组织

### PS 端驱动

  **以太网驱动**，略

  **串口驱动**，将上位机通过 TCP数据包发来的 RS422消息 通过 UART控制器 发出

  **DMA 驱动**，设置DMA通道，待PL将LVDS图像数据通过DMA写入PS内存后，组成TCP数据包发到上位机

  **AXI 总线驱动**，略

### 协议冗余

  因为采用的是 TCP / IP 协议，TCP协议会保证数据的可靠性，没有必要做数据的校验；

  > 解释：由于本项目是地面检测系统，目的是测量协议的正确性，因此需要做检验等等，导致协议 + TCP/IP 看起来十分冗余；

### 图像数据暴涨，导致 PC 端磁盘占满，如何解决？

- 每次启动时，进行数据库自检，清除已失效图像数据，防止数据库存在无效内容；

- 存储图像数据前，进行磁盘大小读取（`std::filesystem::space_info diskSpace = std::filesystem::space("path")`），及时覆盖陈旧数据；

```c++
// 可参考： Information about free space on a disk
struct space_info {
    uintmax_t capacity;
    uintmax_t free;
    uintmax_t available;
};
```

## Lute 项目

### I/O 复用

  **I/O 复用，指的是应用程序通过 I/O 复用函数向内核注册一组事件，内核通过 I/O 复用函数把其中就绪的事件通知给应用程序。**

  Linux 中 常用的 I/O 复用函数：select、poll、epoll

  > **I/O 复用函数本身是阻塞的，他们能提供程序高效的原因是：他们具有同时监听多个 I/O 事件的能力**

#### 事件就绪分类

- 读就绪事件
  - 在 socket 内核中，接受缓冲区中的字节数大于或等于低水位标记 `SO_RCVLOWAT`，此时 `recv` 或 `read` 函数可以无阻塞地读该文件描述符，并且返回值大于 0
  - TCP 连接的对端关闭连接，此时本端调用 `recv` 或 `read` 函数对 socket 进行读操作，将会返回 `0`
  - 在监听 socket  上有新连接请求
  - 在 socket 上有未处理的错误
- 写就绪事件
  - 在 socket 内核中，发送缓冲区中的可用字节数（发送缓冲区的空闲位置的大小）大于或等于低水位标记 `SO_SNDLOWAT` 时，可以无阻塞地写，并且返回值大于 0
  - socket 的写操作被关闭（调用 `close` 或 `shutdown` 函数）时，对一个写操作被关闭的 socket 进行写操作，会触发 `SIGPIPE` 信号
  - socket 使用非阻塞 connect 连接成功或失败时
- 异常就绪事件

#### select 系统调用

  select 函数用于检测在一组 socket 中是否有事件就绪。

  函数原型：`int select (int nfds, fd_set* readfds, sd_set* writefds, fd_set* exceptfds, struct timeval* timeout)`

  > `nfds`：这个参数需要设置为**所有需要 select 函数检测事件的 fd 中的`最大值 + 1`** (因为文件描述符是从 0 开始计数的)

  缺点:

  1. 每次调用 `select` 函数，都需要把 fd 集合从 用户态 复制到 内核态，这个开销在 fd 较多时会很大，同时每次调用 `select` 函数都需要在内核中遍历传递进来的所有fd

  2. 单个进程能够监视的文件描述符的数量存在最大限制，在 Linux 上一般为 1024， 可以通过先修改 宏定义 然后重新编译内核来调整，非常麻烦且效率低下

  3. `select` 函数在每次调用之前都要对传入的参数进行重新设定

  4. 在 Linux 上，select 函数的实现原理是：底层使用 poll 函数

select 使用实例

```c++
#include <arpa/inet.h>
#include <fcntl.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <unistd.h>

#include <cassert>
#include <cerrno>
#include <cstdio>
#include <cstdlib>
#include <cstring>
#include <iostream>

int main(int argc, char* argv[]) {
    if (argc < 2) {
        std::cout << "Usage: " << ::basename(argv[0])
                  << " ip_address port_number" << std::endl;
        return 1;
    }

    const char* ip = argv[1];
    int port = ::atoi(argv[2]);

    int rc = 0;
    struct sockaddr_in addr;
    ::bzero(&addr, sizeof(addr));
    addr.sin_family = AF_INET;
    ::inet_pton(AF_INET, ip, &addr.sin_addr);
    addr.sin_port = ::htons(port);

    int listenfd = ::socket(AF_INET, SOCK_STREAM, 0);
    assert(listenfd >= 0);
    rc = ::bind(listenfd, (const struct sockaddr*)&addr, sizeof(addr));
    assert(rc != -1);
    rc = ::listen(listenfd, 5);
    assert(rc != -1);

    struct sockaddr_in client_addr;
    socklen_t client_addr_len = sizeof(client_addr);
    int connfd = ::accept(listenfd, (struct sockaddr*)&client_addr, &client_addr_len);
    if (connfd < 0) {
        std::cout << "errno is: " << errno << std::endl;
        ::close(listenfd);
    }

    char buf[1024];
    fd_set read_fds;
    fd_set exception_fds;
    FD_ZERO(&read_fds);
    FD_ZERO(&exception_fds);

    while (1) {
        std::memset(buf, '\0', sizeof(buf));
        /* 每次调用 select 前都需要重新在 read_fds 和 exception_fds 中设置文件描述符 connfd，
           因为事件发生后，文件描述符集合将被内核修改 */
        FD_SET(connfd, &read_fds);
        FD_SET(connfd, &exception_fds);

        rc = ::select(connfd + 1, &read_fds, nullptr, &exception_fds, nullptr);
        if (rc < 0) {
            std::cout << "select failure\n";
            break;
        }

        // 可读事件，采用普通的 recv 函数读取数据
        if (FD_ISSET(connfd, &read_fds)) {
            rc = ::recv(connfd, buf, sizeof(buf) - 1, 0);
            if (rc <= 0) break;

            std::cout << "get " << rc << " bytes of nomal data: " << buf << std::endl;
        }
        // 异常事件，采用带 MSG_OOB 标志的 recv 函数读取带外数据
        else if (FD_ISSET(connfd, &exception_fds)) {
            rc = ::recv(connfd, buf, sizeof(buf) - 1, MSG_OOB);
            if (rc <= 0) break;

            std::cout << "get " << rc << " bytes of oob data: " << buf << std::endl;
        }
    }

    ::close(connfd);
    ::close(listenfd);
    return 0;
}
```

---

#### poll 系统调用

- 函数原型：`int poll (struct pollfd* fds, nfds_t nfds, int timeout);`

- poll 使用基本流程

  <img src="https://raw.githubusercontent.com/lutianen/PicBed/main/202307041650094.png" alt="image-20230704165032045" style="zoom: 80%;" />

- 如果使用 epoll 模型，可以改用 `edge trigger`，但是有个问题：如果漏掉了 `accept(2)`，程序再也不会受到新连接。

  解决方案：**准备一个空闲的文件描述符**

     1. 先关闭这个空闲文件，获得一个文件描述符；
     1. 然后 `accept()` 得到 socket 连接的文件描述符；
     1. 随后立即 `close(2)` ，如此就优雅地断开了与客户端的连接；
     1. 最后重新打开空闲文件，把“坑”填上，以备再次出现这种情况时使用。

**poll 和 epoll 在程序上处理就绪事件的差异如下：**

```cpp
/* 处理 POLL */
int rc = poll(fds, MAX_EVENT_NUMBER, -1);
// 遍历所有已注册文件描述符并找到其中的就绪者
for (int i = 0; i < MAX_EVENT_NUMBER; ++i) {
    if (fds[i].revents & POLLIN) { // 判断第 i 个文件描述符是否就绪
        int sockfd = fds[i].fd;
        // handle sockfd
    }
}

/* 处理 EPOLL */
int rc = epoll_wait(epollfd, events, MAX_EVENT_NUMBER, -1);
for (int i = 0; i < rc; ++i) {
    int sockfd = events[i].data.fd;
    // handle sockfd, sockfd 肯定就绪，直接处理，无需向 poll 一样判断
}
```

#### epoll 系统调用

<img src="https://cdn.jsdelivr.net/gh/lutianen/PicBed@master/202309122334645.webp" alt="epoll" style="zoom: 67%;" />

- epoll 在内核里使用**==红黑树==来跟踪进程所有待检测的文件描述字**，把需要监控的 socket 通过 `epoll_ctl()` 函数加入内核中的红黑树里，红黑树是个高效的数据结构，增删改一般时间复杂度是 `O(logn)`
- epoll 使用**事件驱动**的机制，内核里**维护了一个==链表==来记录就绪事件**，当某个 socket 有事件发生时，通过**回调函数**内核会将其加入到这个就绪事件列表中，当用户调用 `epoll_wait()` 函数时，只会返回有事件发生的文件描述符的个数，不需要像 select/poll 那样轮询扫描整个 socket 集合，大大提高了检测的效率

```c
/**
 * @copyright Copyright (c) 2023
 * FOR STUDY AND RESEARCH SUPPORT ONLY
 *
 * @file epoll_unit.cc
 * @brief 同时处理 TCP 和 UDP 请求的回射服务器
 * @author https://github.com/lutianen
 */

#include <arpa/inet.h>  // sockaddr_in
#include <fcntl.h>
#include <sys/epoll.h>  // epoll_create epoll_ctl epoll_wait
#include <unistd.h>     // close send recv

#include <cassert>
#include <cstring>
#include <iostream>

#define kMAX_EVENT_NUMBER 1024
#define kTCP_BUFFER_SIZE 512
#define kUDP_BUFFER_SIZE 1024

int setNonBlocking(int fd) {
    int old_opt = ::fcntl(fd, F_GETFL);
    int new_opt = old_opt | O_NONBLOCK;
    ::fcntl(fd, F_SETFL, new_opt);
    return old_opt;
}

void addfd(int epollfd, int fd) {
    epoll_event event;
    event.data.fd = fd;
    event.events = EPOLLIN | EPOLLET;
    ::epoll_ctl(epollfd, EPOLL_CTL_ADD, fd, &event);
    setNonBlocking(fd);
}

int main(int argc, char* argv[]) {
    if (argc <= 2) {
        std::cout << "Usage: " << ::basename(argv[0]) << " ip_address port_number\n";
        return 1;
    }

    const char* ip = argv[1];
    int port = std::atoi(argv[2]);

    int rc = 0;
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(port);
    ::inet_pton(AF_INET, ip, &addr.sin_addr);
    // Create TCP socket and bind it to port
    int listenfd = ::socket(AF_INET, SOCK_STREAM, 0);
    assert(listenfd != -1);

    rc = ::bind(listenfd, (struct sockaddr*)&addr, sizeof(addr));
    assert(rc != -1);

    rc = ::listen(listenfd, 5);
    assert(rc != -1);

    // Create UDP socket and bind it port
    ::bzero(&addr, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_port = htons(port);
    ::inet_pton(AF_INET, ip, &addr.sin_addr);
    int udpfd = ::socket(AF_INET, SOCK_DGRAM, 0);
    assert(udpfd >= 0);
    rc = ::bind(udpfd, (struct sockaddr*)&addr, sizeof(addr));
    assert(rc != -1);

    epoll_event events[kMAX_EVENT_NUMBER];
    int epollfd = ::epoll_create(5);
    assert(epollfd != -1);

    // Register TCP socket and UDP socket
    addfd(epollfd, listenfd);
    addfd(epollfd, udpfd);

    while (true) {
        int cnt = ::epoll_wait(epollfd, events, kMAX_EVENT_NUMBER, -1);
        if (cnt < 0) {
            std::cerr << "Epoll failure\n";
            break;
        }

        for (int i = 0; i < cnt; ++i) {
            int sockfd = events[i].data.fd;

            if (sockfd == listenfd) {
                struct sockaddr_in client_addr;
                socklen_t client_addr_len = sizeof(client_addr);
                int connfd = ::accept(listenfd, (struct sockaddr*)&client_addr,
                                      &client_addr_len);
                addfd(epollfd, connfd);
            } else if (sockfd == udpfd) {
                char buf[kUDP_BUFFER_SIZE];
                std::memset(buf, 0, kUDP_BUFFER_SIZE);
                struct sockaddr_in client_addr;
                socklen_t client_addr_len = sizeof(client_addr_len);

                rc = ::recvfrom(sockfd, buf, kUDP_BUFFER_SIZE - 1, 0,
                                (struct sockaddr*)&client_addr,
                                &client_addr_len);
                if (rc > 0) {
                    std::cout << "UDP: " << buf << std::endl;
                    rc = ::sendto(sockfd, buf, kUDP_BUFFER_SIZE - 1, 0,
                                  (struct sockaddr*)&client_addr,
                                  client_addr_len);
                }
            } else if (events[i].events & EPOLLIN) {
                char buf[kTCP_BUFFER_SIZE];
                while (true) {
                    std::memset(buf, 0, kTCP_BUFFER_SIZE);
                    rc = ::recv(sockfd, buf, kTCP_BUFFER_SIZE - 1, 0);
                    if (rc < 0) {
                        if ((errno == EAGAIN) || (errno == EWOULDBLOCK)) break;
                        ::close(sockfd);
                        break;
                    } else if (rc == 0)
                        ::close(sockfd);
                    else {
                        std::cout << "TCP: " << buf << std::endl;
                        ::send(sockfd, buf, kTCP_BUFFER_SIZE, 0);
                    }
                }
            } else {
                std::cout << "Something else happend \n";
            }
        }
    }

    ::close(listenfd);
    return 0;
}
```

- 两种触发模式

  - **Level trigger**：服务器端不断地从 epoll_wait 中苏醒，直到内核缓冲区数据被 read 函数读完才结束
  - **Edge trigger**：服务器端只会从 epoll_wait 中苏醒一次
- 事件宏

  - `EPOLLIN` 表示对应的文件描述符**可读（包括对端 socket 正常关闭）**
  - `EPOLLOUT` 表示对应的文件描述符**可写**
  - `EPOLLPRI` 表示对应的文件描述符**有紧急的数据可读（带外数据）**
  - `EPOLLERR` 表示对应的文件描述符**发生错误**
  - `EPOLLHUP` 表示对应的文件描述符**被挂断**
  - `EPOLLET` 将 EPOLL 设为**边缘触发模式**（默认电平触发）
  - `EPOLLONESHOT` **只监听一次事件**，当监听完这次事件之后，如果还需要继续监听这个 socket 的话，需要再次把这个 socket 加入到内核中的事件注册表中

#### select / poll / epoll 对比

|        | 原理                                                         | 一个进程所能打开的最大连接数                                 | FD 剧增后带来的 IO 效率问题                                  | 消息传递方式                                                 |
| ------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| select | 本质上是通过设置或检查 fd 标志位的数据结构来进行下一步处理；<br />缺点：1. 单个进程可监视的 fd 的数量被限制.<br />            2. 需要维护一个用来存放大量 fd 的数据结构，使得用户空间和内核空间在传递该结构时复制开销大<br />            3. 对socket 进行扫描采用的是线性扫描 | 单个进程所能打开的最大连接数由 `FD_SETSIZE` 宏定义，大小为 32 个整数的大小（32 bit 机器上就是：32 *32，64 bit 机器上就是：32* 64）；<br />可以修改，当需要重新编译内核，性能或许受到影响 | 每次调用 select，都会对连接进行线性遍历，因此随着FD的增加，存在“线性下降性能问题” | 内核需要将消息传递到用户空间，需要内核拷贝动作               |
| poll   | 本质上和 select 没有区别，都是将用户传入的数组拷贝到内核空间，然后查询每个 fd 的设备状态；<br />特点：1. 没有最大连接数的限制（基于链表来存储，当时会导致 fd 链表整体复制于用户空间和内核空间）<br />            2. LT（level trigger），如果报告了 fd 当没被处理，会再次报告该 fd | 没有最大连接数的限制，因为它是基于链表来存储的               | 每次调用 poll ，都会对链表进行线性遍历，同样存在“线性下降性能问题” | 和 seleect 一样，都需要内核拷贝，且需要链表整体拷贝，性能浪费 |
| epoll  | 采用 `mmap` 和事件通知的方式进行 fd 监听；<br />特点：1. 采用 `mmap` 减少复制开销.<br />           2. "事件" 通知方式，通过 `epoll_ctl` 注册该 fd，一旦该 fd 就绪，内核就会采用类似于 callback 的回调机制来激活该 fd, `epoll_wait` 便可以收到通知 | 存在连接数上限，但是很大，1G 内存的机器上大约可以打开 10W 左右 | epoll 内核中实现是根据每个 fd 上的 callback 函数来实现的，只有活跃的 socket 才会主动调用 callback ，所有在活跃连接较少的情况下，不存在线性下降的性能问题；但是所有 socket 都很活跃的情况下，可能存在性能问题 | epoll 通过内核和用户空间共享一块内存实现，不存在内核拷贝动作 |

### Proactor 和 Reactor 的区别？

#### Reactor 模式

- 要求主线程 （I0 处理单元）只负责监听 `listenfd` 文件描述符上是否有事件发生，有的话立即将该事件通知工作线程（逻辑处理单元）；除此之外，主线程不做任何其他实质性的工作
- 读写数据、接受新的连接、以及处理客户端的请求均在工作线程完成
- **Reactor 模式，关键步骤：事件注册、事件循环、事件分发、事件处理**
- Ractor 模式的工作流程：
  1. 主线程向 epoll 内核事件表中注册 socket 上的读就绪事件；
  2. 主线程调用 `epoll_wait` 等待 socket 上有数据可读
  3. 当 socket 上有数据可读时，`epoll_wait` 通知主线程，然后主线程将可读事件放入请求队列；
  4. 睡眠在请求队列上的某个工作线程被唤醒，并从 socket 读取数据、处理客户请求，然后向 epoll 内核事件表中注册 socket 上的写就绪事件
  5. 主线程调用 `epoll_wait` 等待 socket 可写；
  6. 当 socket 可写时，`epoll_wait` 通知主线程，然后主线程将 socket 可写事件放入请求队列
  7. 睡眠在请求队列上的某个工作线程被唤醒，并向 socket 上写入服务器处理客户请求的结果

<img src="https://raw.githubusercontent.com/lutianen/PicBed/main/202307051538079.svg" alt="Reactor" style="zoom: 150%;" />

> 工作线程从请求队列中取出事件后，将根据事件的类型来决定如何处理它：
>
> - 对于可读事件，执行读数据和处理请求操作
> - 对于可写事件，执行写数据的操作
>
> **因此，在该类 Reactor 模式中，没必要区分所谓的 “ 读工作线程 ” 和 “ 写工作线程”**

#### Proactor 模式

- 要求将所有 IO 操作都交给主线程和内核来处理，工作线程仅仅负责业务逻辑

### 同步、异步、并发模型

| IO 模型    | 读写操作和阻塞阶段                                           |
| ---------- | :----------------------------------------------------------- |
| 阻塞 IO    | 程序阻塞于读写函数                                           |
| IO 复用    | 程序阻塞于 IO 复用系统调用，但可同时监听多个 IO 事件；对 IO 本身的读写操作是非阻塞的 |
| SIGIO 信号 | 信号触发读写就绪事件，用户程序执行读写操作；程序本身没有阻塞阶段 |
| 异步 IO    | 内核执行读写操作并触发读写完成事件；程序没有阻塞阶段         |

> **主要用于区分内核向应用程序通知的是何种 IO 事件（就绪事件 or 完成事件），以及由谁来完成 IO 读写（应用程序 or 内核）**

#### IO模型中的同步

- **同步** IO 模型，指的是应用程序发起 IO 操作后，必须等待 IO 操作完成后才能继续执行后续的操作，即 IO 操作的结果需要立即返回给应用程序；在此期间，应用程序处于阻塞状态，无法做其他操作。
- 优点：编程模型简单
- 缺点：效率较低（应用程序的执行速度被 IO 操作所限制）

> **对于操作系统内核来说，同步 IO 操作是指在内核处理 IO 请求时需要等待**

#### IO 模型中的异步

- **异步** IO 模型，指的是应用程序发起 IO 操作后，无须等待 IO 操作完成，可以立即进行后续的操作；在此期间，操作系统负责把 IO 操作的结果返回给应用程序；
- 优点：可以充分利用系统资源，提高 IO 操作的效率
- 缺点：编程模型相对复杂

> **对于操作系统内核来说，异步 IO 操作指的是，在内核处理 IO 请求时无需等待，立即返回**

#### 并发模式

> **并发模式，指的是 I/O 处理单元和多个逻辑单元之间协调完成任务的方法**

##### 半同步半异步模式

**half-sync / half-async**

- 此时的 ”同步“ 和 ”异步“ 不用于 I/O 模型中的 ”同步“ 、”异步“ 的概念。

- **并发模式中，“同步“ 指的是程序完全按照代码序列的顺序执行；” 异步 ” 指的是程序的执行需要有系统事件（常见的系统事件：中断、信号等）驱动**

- 按照同步方式运行的程序成为**同步线程**，按照异步方式运行的线程成为**异步线程**；

  显然，异步线程的执行效率高，实时性强，是很多嵌入式程序采用的模型；但编写异步方式执行的程序相对复杂，难于调试和扩展，而且不适合于大量的并发；

  同步线程，虽然执行效率相对较低，实时性较差，但逻辑简单。

  **对于服务器这种既要求较好的实时性，又要求能同时处理多个客户请求的应用程序，就应该同时使用同步线程和异步线程的方式实现，即半同步 / 半异步模式**

- 同步线程：用于处理客户逻辑，**请求队列将通知某个工作在同步模式的工作线程来读取并处理该请求对象**

- 异步线程：用于处理 I/O 事件，**异步线程监听到客户请求后，就将其封装成请求对象并插入到请求队列中**

<img src="https://raw.githubusercontent.com/lutianen/PicBed/main/202307051542079.svg" style="zoom:150%;" />

###### 半同步 / 半反应堆模式

**half-sync / half-reactive**

- 半同步 / 半反应堆模式，是 ”半同步/半异步“ 模式的一种变体

- 采用的事件处理模式：Reactor 模式，要求工作线程自己从 socket 上读取客户请求和往 socket 上写入服务器应答

- 异步线程：只有一个，且有主线程来充当，负责监听所用 socket 上的事件

  - 如果**监听 socket** 上有**可读事件**发生，即新连接到来，主线程就接受之以获得新的连接 socket ，然后往 epoll 内核事件表中注册该 socket 上的读写事件
  - 如果**连接 socket** 上有**读写事件**发生，即有新的客户请求到来或有数据要发送到客户端，主线程就将该 **连接 socket** 插入到请求队列

- 缺点：

  - 主线程和工作线程共享请求队列；

    主线程往请求队列中添加任务，或者工作线程从请求队列中取出任务，都需要对请求队列进行加锁，浪费 CPU 时间

  - 每个工作线程在同一事件只能处理一个客户请求；

    如果客户数量较多，而工作线程较少，则请求队列中将堆积很多任务对象，客户端的响应速度将越来越慢

  <img src="https://raw.githubusercontent.com/lutianen/PicBed/main/202307051626747.svg" style="zoom:150%;" />

###### 高效的半同步 / 半异步模式

- **主线程只管理监听 socket** ，**连接 socket 由工作线程来管理**；
- **每个线程（主线程 和 工作线程）都维持自己的事件循环，各自独立监听不同的事件**；
- 在该高效的半同步 / 半异步模式中，**每个线程都工作在异步模式**，因此它并非严格意义上的半同步 / 半异步模式；

<img src="https://raw.githubusercontent.com/lutianen/PicBed/main/202307051635410.svg" style="zoom:150%;" />

> 当有新的连接到来时，主线程接受并将返回的连接 socket 派发给某个工作线程此后该新的 socket 上的任何 I/O 操作都有被选中的工作线程来处理，直到客户关闭连接。
>
> 派发方式：最简单就是主线程往管道里写数据，工作线程检测到管道上有数据可读时，就分析是否是一个新的客户连接请求到来；

##### 领导者 / 追随者模式

领导者 / 追随者模式，是多个工作线程轮流获得事件源集合，轮流监听、分发并处理事件的一种模式。

- 在任意时间，程序都仅有一个领导者线程，它负责监听 I/O 事件；其他线程都是追随者，且休眠在线程池中等待成为新的领导者；
- 当前领导者如果检测到 I/O 事件，首先要从线程池中推选出新的领导者线程，然后处理 I/O 事件；此时，新的领导者等待新的 I/O 事件，而原来的领导者则处理 I/O 事件，二者形成并发；
- 优点：由于领导者线程自己监听 I/O 事件并处理客户请求，因此领导者 / 追随者模式不需要在线程之间传递任何额外的数据，也不需要在线程之间同步对请求队列的访问；
- 缺点：仅支持一个事件源集合，无法让每个工作工作线程独立地管理多个客户连接；

### 压力测试

#### Webbench

**Webbench**，是 Linux 系统上的 web 性能压力测试工具，最多可以模拟 **3 万**个并发连接数来测试服务器压力。

- 原理：**父进程 fork 多个子进程，每个子进程都循环做 web 访问测试，子进程将访问结果通过管道告知父进程，父进程做最终结果统计。**
- 优点：
  - 部署简单，适用于小型网站压力测试（最多模拟 3 万并发）
  - 具有静态页面测试能力，也支持动态页面（ASP，PHP，JAVA ，CGI）测试能力
  - 支持对含有 SSL 的安全网站（如电子商务网站）进行动态或静态性能测试
- 缺点：
  - 不适用于中大型网站测试
  - 采用多进程实现并发，而非多线程；长时间其会占用大量内存和 CPU，一般长时间的压力测试不推荐使用 webbench

### 常见面试题以及核心特性解析

1. **Buffer 仿 Netty ChannelBuffer，数据的读写通过 buffer 进行，用户不需要调用 `read(2) / write(2)`，只需要处理受到的数据和准备好要发送的数据。**

2. Channel 是 selectable IO channel ，负责注册和响应 IO 事件，不拥有 file descriptor. Channel 是 Acceptor、Connector、EventLoop、TimerQueue、TcpConnection 的成员，生命周期由后者控制。

3. TCP 网络编程最本质的是处理 ==**三个半事件**==：

   1. ==**连接的建立**==：包括服务端接受（`accept`）新连接和客户端成功发起（`connect`）连接。

      TCP 连接一旦建立，客户端和服务端是平等的，可以各自收发数据。

   2. ==**连接的断开**==：包括主动断开（`close` 或 `shutdown`）和被动断开（`read(2)` 返回 `0`）

   3. ==**消息的到达，即文件描述符可读**==。最为重要的一个事件

      对它的处理方式决定了网络编程的风格（阻塞还是非阻塞，如何处理分包，应用层的缓冲如何设计等等）。

   4. ==**消息发送完毕**==，只能算半个事件。

      对于低流量的服务，可以不用关心这个事件；另外，“发送完毕”指的是将数据写入操作系统的缓冲区，将由 TCP 协议栈负责数据的发送和重传，不代表对方已经受到数据。

4. 在非阻塞网络编程中，为什么要使用应用层发送缓冲区?

   > 1. 假如要发送 40 k 数据，但操作系统 TCP 发送缓冲区只有 25k 剩余空间，此时剩下的 15k 数据怎么办？
   >
   >    - 如果等待 OS 缓冲区可用，会阻塞当前线程（因为不知道对方什么时候收到并读取数据）
   >
   >    **因此，网络库应该把剩下的 15k 数据缓存起来，放到这个 TCP 连接的应用层发送缓冲区中，等 socket 可写的时候立刻发送数据，如此“发送”操作不会阻塞**
   >
   > 2. 如果在此期间应用程序又要发送 50k 数据，此时发送缓冲区中尚有 5k 数据，应用层该如何处理？
   >
   >    **网络库应该将这 50k 数据追加到发送缓冲区的末尾，而不是立刻尝试 write ，因为这样有可能打乱数据的顺序**

5. 在非阻塞网络编程中，为什么需要使用应用层接收缓冲区？

   > 假如一次读到的数据不够一个完整的数据包，那么这些已经读到的数据是不是应该先暂存在某个地方，等剩余的数据收到之后再一并处理？

6. 在非阻塞网络编程中，如何设计并使用缓冲区？

   > 一方面希望减少系统调用，一次读的数据越多越划算，那么似乎应该使用一个大的缓冲区。
   >
   > 另一个方面希望系统减少内存占用，那么似乎应该使用一个小的缓冲区。
   >
   > 默认：`kInitialSize = 1024 Byte`
   >
   > 解决方案：readv + 栈上空间
   >
   > - `readv`用于从指定文件描述符 sockfd 中读取数据，支持一次读取多个非连续缓冲区（散布读取）
   >
   > - ```cpp
   >   ssize_t Buffer::readFd(int fd, int* savedErrno) {
   >       // saved an ioctl()/FIONREAD call to tell how much to read
   >       char extrabuf[65536];
   >                                                                   
   >       // Prepare iovecs
   >       struct iovec vec[2];
   >       const size_t writeable = writableBytes();
   >       vec[0].iov_base = begin() + writerIndex_;
   >       vec[0].iov_len = writeable;
   >                                                                   
   >       vec[1].iov_base = extrabuf;
   >       vec[1].iov_len = sizeof(extrabuf);
   >                                                                   
   >       /// When there is enough space in this buff, don't read into extrabuf
   >       /// When extrabuf is used, we read 128k-1 bytes at most.
   >       const int iovcnt = (writeable < sizeof(extrabuf)) ? 2 : 1;
   >       const ssize_t n = sockets::readv(fd, vec, iovcnt);
   >       if (n < 0) {
   >           *savedErrno = errno;
   >       } else if (static_cast<size_t>(n) <= writeable) {
   >           writerIndex_ += static_cast<size_t>(n);
   >       } else {
   >           writerIndex_ = buffer_.size();
   >           append(extrabuf, static_cast<size_t>(n) - writeable);
   >       }
   >                                                                   
   >       return n;
   >   }
   >   ```

7. 如果使用发送缓冲区，万一接收方处理缓慢，数据会不会一直堆积在发送方，造成内存暴涨？如何做应用层的流量控制？

8. 如何设计并实现定时器？使之与网络IO共用一个线程，避免锁的使用？

9. Logger 如何实现类型安全？

   > 通过 logStream 类，将 `运算符 <<` 重载实现

10. Logger 有没有实现 **崩溃安全**？

    > Logger 采用 **双缓冲机制 + 前后异步线程**，当应用程序发生崩溃时，主线程会等待后台线程将缓冲区中日志信息写入文件结束后才会退出，保证了崩溃安全。
    >
    > **只有当前缓冲区为空，才会结束当前线程，即确保当前缓冲区日志信息成功落盘。**

11. 什么是惊群效应？

    惊群效应（Thundering herd），是指多进程（多线程）在同时阻塞等待同一个事件的时候（休眠状态），如果等待的事件发生，那么就会唤醒等待的所有进程（线程），但最终只能有一个进程（线程）获得控制权，对该事件进行处理，而其他进程（线程）获取控制权失败，只能重新进入休眠状态，这种现象和性能浪费就叫做惊群效应。

12. 生产者-消费者模型可能的优化点？

     - **如果生产者和消费者的速度差不多，则可以将队列改成环形队列，以节省内存空间。**
     - 为了追求效率，也可以将队列无锁化

13. 如何将 socket 文件描述符设置为 非阻塞模式？

     - 使用 `fcntl` 函数或 `ioctl` 函数给创建的 socket 增加 `O_NONBLOCK` 标志来将 soeckt 设置为非阻塞模式

       ```cpp
       int oldFlag = fcntl(sockfd, F_GETFL, 0);
       int newFlag = oldFlag | O_NONBLOCK;
       fcntl(sockfd, F_SETFL, newFlag);
       ```

     - 可以在创建 socket 时设置为 非阻塞 模式

       ```cpp
       # int socket(int domain, int type, int protocl);
       int sockfd = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK, IPPROTO_TCP);
       ```

     - 在 Linux 上利用 `accept` 函数返回的代表与客户端通信的 socket 也提供了一个扩展函数 `accept4`，直接将 `accept` 函数返回的 socket 设置为非阻塞的

       ```cpp
       # int accept(int sockfd, struct sockaddr* addr, socklen_t* addrlen);
       # int accept(int sockfd, struct sockaddr* addr, socklen_t* addrlen, int flags);
       
       socklen_t addrlen = sizeof(clientaddr);
       int clientfd = accept4(listenfd, &clientaddr, &addrlen, SOCK_NONBLOCK);
       ```

14. **`send` 和 `recv` 函数在阻塞模式和非阻塞模式下的表现？**

     `send` 函数在本质上并不是向网络上发送数据，而是将应用层发送缓冲区的数据拷贝到内核缓冲区中，至于数据什么时候从网卡缓冲区中真正发送到网络中，要根据 TCP/IP 协议栈的行为来确定。

     如果 socket 设置了 `TCP_NODELAY` 选项（禁用 nagel 算法），存放到内核缓冲区的数据就会被立即发送出去；反之将会缓冲数据。

     `recv` 函数本质上并不是从网络中收取数据，而是将内核缓冲区中的数据拷贝到应用程序的缓冲区中。在拷贝完成后，内核会将这部分数据从内核缓冲区中移除。

     > ==假设应用程序不断调用 `send` 函数，则数据会不断拷贝到对应的内核缓冲区中，那么在对端的内核缓冲区被填满后，本端的内核缓冲区也会被填满，此时本端继续调用 `send` 函数会发生什么？==
     >
     > - 当 socket 处于阻塞模式时，继续调用 `send` / `recv` 函数，程序会阻塞在调用处
     > - 当 socekt 处于非阻塞模式时，继续调用 `send` / `recv` 函数，并不会被阻塞，而是立即出错并返回 -1，且错误码`EWOULDBLOCK` 或 `EAGAIN` （两个值相同）

     **内核缓冲区有专门的名字，即 TCP 窗口**

    | send / recv 返回值 n | 返回值的含义                                                 |
    | :------------------: | :----------------------------------------------------------- |
    |        大于 0        | 成功发送（send）或接收（recv） 的字节数<br />根据已发送的字节数确定是不是期望字节数（在一个循环里根据偏移量发送数据，保证所有数据都发送成功） |
    |          0           | 对端关闭连接<br />发现相对端关闭连接，本端也关闭连接即可     |
    |    小于 0 （-1）     | 出错、被信号中断（EINTR）、对端 TCP 窗口太小导致发送失败或当前网卡缓冲区已无数据可接收（EWOULDBLOCK / EAGAIN） |

    | send / recv 返回值 n 和错误码 errno           | send 函数                                              | recv 函数                |
    | --------------------------------------------- | ------------------------------------------------------ | ------------------------ |
    | n = -1, errno = EWOULDBLOCK / EAGAIN          | TCP 窗口太小，数据暂时发送不出去（应缓存未发送的数据） | 当前内核缓冲区无可读数据 |
    | n = -1, errno = EINTR                         | 被信号中断，需要重新尝试发送                           | 被信号中断，需重试       |
    | n = -1, errno = others(这里指的是不是以上3种) | 出错                                                   | 出错                     |

15. 如果发送 0 字节数据，会发生什么？

     ==**`send` 函数发送 0 字节数据，此时 `send` 函数返回 `0`，但操作系统的协议栈并不会把这些数据发送出去**==

     ==**`recv` 函数只有在对端关闭连接时才会返回 `0`（对端 `send` 0 字节数据，本端的 `recv` 函数不会收到 0 字节数据）**==

16. 连接时顺便接收第 1 组数据，如何做到？

     Linux 提供了 `TCP_DEFER_ACCEPT` 的 socket 选项，设置该选项后，只有连接建立成功且收到第 1 组对端数据时，`accept` 才会返回。

17. 在 Linux 中调用一些 socket 函数（`connect`， `send`， `recv`， `epoll_wait` 等），如果返回 -1 ，一定是调用失败吗？

     在这些 socket 函数调用时，除了函数调用出错返回 -1 ，当 **被信号中断** 时也会返回 -1 ，此时我们可以通过错误码 `errno` 判断是否为 `EINTR`来确定；

     如果被信号中断，那么我们需要再次调用这些函数进行重试。

18. 在服务器端网络编程中，如何处理 `SIGPIPE` 信号？

     然而由于TCP是全双工通行，通信的一方无法获知对端的 socket 是调用 `close` 还是 `shutdown`关闭接收通道、发送通道或者同时关闭收发通道；

     当对端关闭连接，本端继续向对端发送数据，根据 TCP 协议规定，本端会收到对端的一个 RST 报文应答，若此时继续向对端发送数据，系统就会产生 `SIGPIPE`  信号给本端进程，而系统对 `SIGPIPE` 信号的默认处理行为是让本端进程退出；

    OS 的这种默认行为对于开发程序造成一定的影响，尤其是服务程序，因为后端服务程序需要同时对多个客户端进行服务，不能因为与某个客户端的连接出了问题，从而导致整个后端服务程序进程退出，不能继续为其他客户端服务；

    为了避免出现这种情况，一般解决方案：**捕获 SIGPIPE 信号并对其进行处理或忽略该信号** `signal (SIGPIPE, SIG_IGN)`

## 操作系统

### 操作系统这门课主要讲什么东西

操作系统（Operating System，简称 **OS**）是**计算机系统中最基础、最核心的软件之一，它负责管理计算机的硬件和软件资源，提供用户接口和服务**，是计算机系统中不可或缺的一部分：

1. **进程管理**：讲解进程的概念、进程调度算法、进程同步与通信、进程死锁等内容
2. **内存管理**：讲解内存的组织方式、分配与释放算法、虚拟内存、内存保护等内容
3. **文件系统**：讲解文件的组织方式、文件系统的层次结构、文件的操作和保护、文件系统的实现和管理等内容
4. **输入输出管理**：讲解输入输出设备的种类、输入输出的实现、设备驱动程序、中断处理等内容
5. **网络管理**：讲解计算机网络的基本概念、网络协议、网络拓扑结构、网络安全等内容
6. **操作系统安全**：讲解操作系统的安全性问题、常见的攻击方式、安全策略和防御措施等内容

### 现代操作系统设计目标

**现代 CPU + 操作系统，其设计主要目标是为了完美高效的实现一个多任务系统，多任务系统的三个核心特征：==权限分级==、==数据隔离==、==任务切换==**

x86_64 架构为例：

- 权限分级：**通过 CPU 的多模式机制和分段机制实现**
- 数据隔离：**通过分页机制实现**
- 任务切换：**通过中断机制和任务机制实现**

### 什么是进程、线程？两者有什么区别？

#### **进程**

  在操作系统中，进程使用 ==进程控制块PCB(Process Control Block)== 数据结构 `task_struct` 来描述，PCB **是进程存在的唯一标识**。

- 进程是指在系统中正在运行的一个应用程序，程序一旦运行就是进程；

- 进程可以认为是程序执行的一个实例，进程是系统进行资源分配的最小单位，且每个进程拥有独立的地址空间；

- 一个进程无法直接访问到另一个进程的变量和数据结构，如果希望一个进程去访问另一个进程的资源，需要使用进程间的通信；

- 进程间通信（必须经过内核）方式：

  - 管道：==存在于文件系统， `prw-r--r--  1 lux  lux   0  7月 13 20:35 mypipe`==，文件类型为 ==p== ，`int mkfifo(const char* path, __mode_t mode);`
  - 匿名管道：==存在于内存中，不存在于文件系统中==，`int pipe(int fd[2])`，一般是父进程写`fd[1]`，子进程读 `fd[[0]`
  - 消息队列：==保存在内核中的消息链表==，==通信不及时、数据块大小有限制==，==存在内核态与用户态之间的数据拷贝开销==
  - 共享内存：==存在于内存中==，==通信多个进程开不同/相同的辟虚拟内存，并映射到相同的物理内存==，==不存在数据拷贝开销==，`int shmget(key_t key, size_t size, int flag);` `void *shmat(int shm_id, const void *addr, int flag);`
  - 信号：==异常情况下的工作模式，唯一的异步通信机制==，`int kill(pid_t pid, int sig);`发送信号，`sighandler_t signal(int signum, sighandler_t handler);`
  - 信号量：==一个整型的计数器，对可用资源的数量进行计数，主要用于实现进程间的互斥与同步，而不是用于缓存进程间通信的数据==，==P操作，信号量减 1==，==V 操作，信号量加 1==
  - Socket：==跨网络与不同主机上的进程之间通信 或 本地进程间通信==，`int socket(int domain, int type, intprotocol)`

- 进程调度算法：先来线服务调度算法、短作业优先调度算法、最短剩余作业优先调度算法、最高响应比优先调度算法、最高优先级优先调度算法、时间片轮转算法（公平调度，$20 - 50 ms$）、多级反馈队列调度算法($最高优先级 + 时间片轮转$)；

    <img src="https://cdn.jsdelivr.net/gh/lutianen/PicBed@master/202310041250185.webp" alt="进程状态切换" style="zoom:67%;" />

#### **线程**

> **用户线程**，是基于用户态的线程管理库来实现的，==**线程控制块 TCB (Thread Control Block)**== 也是在库里实现， 操作系统只能看到整个进程的 PCB；即，用户级线程的模型属于 ==多对一== 的关系，多个用户线程对应一个内核线程；

> **内核线程**，是由操作系统管理的，对应的 TCB 存储在操作系统里，且线程的创建、销毁、调度都有操作系统完成；

> **轻量级线程 LWP(Light-weight process)**，是由内核支持的用户线程，一个进程可以有一个或多个 LWP，每个 LWP 是跟内核线程一对一映射的，即 LWP 都是由一个内核线程支持，而且 LWP 是由内核管理并像普通进程一样被调度。
>
> **在大多数系统中，LWP 和 普通进程的区别在于，LWP 只有一个最小的执行上下文和调度程序所需的统计信息。**

- 线程是进程的一个实体，是进程的一条执行路径；
- 线程是比进程更小的独立运行的基本单位，因此也被称为轻量级进程；

**==一个程序至少有一个进程，一个进程至少有一个线程==**

#### **区别**

- 进程是资源（包括内存、打开的文件等）分配的单位，线程是 CPU 调度的单位；
- 进程拥有一个完整的资源平台，而线程只独享必不可少的资源，如寄存器和栈；
- **同一进程的线程共享本进程的地址空间，而进程之间则是独立的地址空间**；
- **同一进程内的线程共享本地的资源，但是进程之间的资源是独立的**；
- **一个进程崩溃后，在保护模式下不会对其他进程产生影响，但是==一个线程崩溃整个进程崩溃，即多进程比多线程健壮==**；
- 进程切换，消耗的资源大，线程同样具有就绪、阻塞、执行三种基本状态，同样具有状态之间的转换关系；
- 多进程、多线程都可以并发执行，线程能减少并发执行的时间和空间开销；
- 每个独立的进程有一个程序入口、程序出口；线程不能独立运行，必须依存于应用程序中，有应用程序提供多个线程执行控制；

### 系统调度

<img src="https://raw.githubusercontent.com/lutianen/PicBed/main/202307052133106.png" alt="image-20230705213318903" style="zoom: 50%;" />

**调度本质 = CPU 上下文（执行环境） + 调度载体 + 执行代码**

- ==**CPU 上下文（Context），简单来说就是程序（进程/中断）运行时所需要的寄存器的最小集合，比如通用寄存器、程序计数器PC、CR3 页目录基址寄存器**== ，**进程的上下文切换不仅包含了虚拟内存、栈、全局变量等用户空间的资源，还包括了内核堆栈、寄存器等内核空间的资源。**
  - 中断上下文
  - 进程上下文
  - 系统调用上下文
  - 线程上下文
  - 协程上下文

- 调度载体：进程、线程、协程、硬中断、软中断

- 执行代码：用户代码、内核代码、中断处理程序（ISR）、系统调用处理程序

### 介绍一下 `fork`， `wait`， `exec` 函数

父进程产生子进程，使用 `fork` 拷贝出来一个父进程的副本，此时只拷贝了父进程的页表，两个进程都读同一块内存。

当有进程写的时候，采用**写时拷贝**机制（COW）分配内存，`exec`函数可以加载一个 elf 文件去替换父进程，从此父进程和子进程就可以运行不同的程序了。

`fork` 时父进程返回子进程的 pid，子进程返回 0；调用了 `wait` 的父进程将会发生阻塞，直到有子进程状态改变，执行成功返回 0，错误返回 -1。

`exec` 执行成功则子进程从新的程序开始运行，无返回值，执行失败返回 -1。

### 孤儿进程、僵尸进程的区别？如何避免？

**孤儿进程 Orphan Process**：==父进程先于子进程而退出，此时未结束的子进程会被 init(pid = 1) 进程接收，成为孤儿进程==；

**僵尸进程 Zombie Process**：==子进程先于父进程退出，此时父进程还未结束且并没有调用 `wait` / `waitpid` 函数获取子进程的状态信息，则子进程残留的状态信息（`task_struct` 结构和少量资源信息）会变成僵尸进程==；

**避免措施**：

- 父进程回收子进程：父进程应该在子进程终止后通过 `wait` / `waitpid` 系统调用来回收子进程的资源和状态，可以避免子进程成为僵尸进程；
- 设置 `SIGCHLD` 信号 handler ：父进程可以设置 `SIGCHLD` 信号的 handler ，在 handler 中调用 `wait` / `waitpid`，当子进程终止时，会发送 `SIGCHLD` 信号给父进程；
- 使用 双重 fork：父进程可以使用双重 `fork` 来创建子进程；首先，父进程创建一个子进程，然后子进程再创建一个孙进程，然后，子进程立即终止，使孙进程成为孤儿进程，被 init 进程接管，便可确保孙进程不会成为僵尸进程；

### 什么是守护进程？如何实现？

**守护进程 Daemon**：指的是在后台运行，没有控制终端与之相连的进程。它独立于控制终端，通常周期性地执行某种任务。

  守护进程脱离于终端的目的：为了避免进程在执行过程中的信息在任何终端上显示并且进程也不会被任何终端所产生的终端信息所打断。

**特点**

- 最重要的特点：==后台运行==
- 守护进程鼻部与其运行前的环境隔离开：这些坏境包括 ==未关闭的文件描述符==、==控制终端==、==会话和进程组==、==工作目录==、==文件 umask== 等，通常是从执行它的父进程（例如 shell）中继承下来的；
- 特殊的启动方式：
  - 在 Linux 系统启动时从启动脚本 `/etc/rc.d` 中启动
  - 由作业规划进程 `crond` 启动
  - 由用户终端 `shell` 执行

**实现**

1. 在父进程调用 `fork`并 `exit`退出
2. 在子进程中调用 `setsid` 函数创建==新的会话==
3. 在子进程中调用 `chdir` 函数，==让根目录 `/` 成为进程的工作目录==
4. 在子进程中调用 `umask` 函数，==设置进程的 `umask` 为 0==
5. 在子进程关闭任何不需要的文件描述符

### 介绍一下 Linux 中 `umask`

`umask` （用户文件创建屏蔽），是一个用于控制文件权限的重要命令，在 Linux 和类 Unix 操作系统中非常常见；通常在用户的 shell 配置文件中（`.bashrc`, `.bash_porfile`, etc）

==`umask`决定了新创建文件的默认权限，通过**屏蔽掉特定权限位**，可以限制文件的默认权限，以提高系统的安全性；==

### 某个线程崩溃，会导致进程退出吗？

一般来说，每个线程都是独立执行的单位，都有自己的上下文堆栈，一个线程崩溃不会对其他线程造成影响。

但是，在通常情况下，一个线程崩溃也会导致整个进程退出，因为 Linux 操作系统产生一个硬件异常，然后被操作系统捕获转化成 `SIGSEGV` 信号（ Segment Default 错误信号），而操作系统对这个信号的默认处理就是结束进程，这样整个进程都被销毁。

### 进程和作业的区别？

**进程**

  进程 Process ，是程序的一次动态执行，属于动态概念；

  一个进程可以执行多个程序，一个程序也可以有多个进程执行；

  程序可以作为一种软件资源长期保留，而进程是程序的一次执行；

  进程是一个独立的云新单位；

**作业**

  ==**作业 Job ，是一组相关的进程或命令的集合**==；通常，作业是由一个或多个命令组成的命令行任务；当在终端运行一个命令时，该命令将被视为一个作业。

**区别**

- 创建方式：进程是通过运行一个程序来创建的，作业是通过在终端中输入一个或多个命令来创建的；
- 关联性：进程是一个独立的实体，可以在后台运行，与终端无关；作业是与终端关联的，它们可以在前台或后台运行；
- 管理方式：进程有操作系统管理，包括资源分配、调度和终止等；作业由终端管理，可以使用命令来控制作业的运行和状态；
- 控制方式：进程可以通过信号来控制，例如发送 `SIGKILL` 信号来终止进程；作业可以使用终端命令来控制，例如使用 `Ctrl + C` 来中断作业的执行；
- 状态管理：进程有不同的状态，如运行、就绪、阻塞等；作业也有不同的状态，如运行、停止、后台等；

### 协程 Coroutine 是什么？

<img src="https://raw.githubusercontent.com/lutianen/PicBed/main/202307140942964.png" alt="image-20230714094225863" style="zoom: 50%;" />

实现协程的关键点：**==在于如何保存、恢复和切换上下文==**，协程切换只涉及基本的CPU上下文切换（CPU寄存器）

Coroutine 就是**函数**，只不过是可以是 suspend 和 resume 的函数。

**所有的协程共用的都是一个栈，即系统栈，也就也不必我们自行去给协程分配栈，因为是函数调用，我们当然也不必去显示的保存寄存器的值；**

**分类**

  有栈 (stackful) 协程：实现类似于内核态线程的实现，不同协程的切换还是要切换对应的栈上下文，只是不用陷入内核，例如 goroutine、libco

  无栈 (stackless) 协程：无栈协程的上下文都会放到公共内存中，在协程切换时使用状态机来切换，而不用切换对应的上下文（都已经在堆中），相比有栈协程更轻量，例如 C++20、Rus、JavaScript；**==本质就是一个状态机（state machine），即同一协程协程的切换本质不过是指令指针寄存器的改变==**

**特点**

- 一个线程可以有多个协程；协程不是被操作系统内核管理，而是完全由程序控制；
- 协程的开销远远小于线程；协程拥有自己的寄存器上下文和栈，在进行协程调度时，将寄存器上下文和栈保存到其他地方，在切换回来时恢复先前保存的寄存器上下文和栈；
- 每个协程表示一个执行单元，有自己的本地数据，与其他协程共享全局数据和其他资源；
- 跨平台、跨体系架构、无需线程上下文切换的开销、方便切换控制流，简化编程模型；
- 协程又称 ”微线程“，协程的完成主要靠 `yeild` 关键字；
- 协程的执行效率极高，和多线程相比，线程数量越多，协程的性能优势越明显；
- 不需要多线程的锁机制

### 页和段的区别？

**页 Page ，是==内存管理的最小单位==，用于将物理内存划分为固定大小的快，通常为 ==4KB==；操作系统将内存划分为连续的页，并使用页表来映射虚拟地址到物理地址**

**进程的虚拟地址空间被划分为多个页，每个页都有对应的物理地址**

**段 Segment ，是一种逻辑上的内存划分方式，用于将进程的虚拟地址划分为不同的部分，常见的段包括 ==代码段==、==数据段==、==堆段==、==栈段==等；每个段都有各自的权限和属性，并可以根据需要动态调整；段的大小是可变的（可根据需要进行扩展或缩小）**

**区别**

- 单位大小：页是内存管理的最小单位，通常是 4KB；段是逻辑上的内存划分，大小可变；
- 内存映射：页是通过页表将虚拟地址映射到物理地址；段是通过段描述符来定义和管理虚拟地址空间的不同部分；
- 权限和属性：页通常具有相同的权限和属性，例如读、写、执行等；段可以具有不同的权限和属性，例如代码段具有可执行权限，数据段具有读写权限等；
- 动态调整：页的大小是固定的，无法动态调整；段的大小可以根据需要进行动态调整，例如堆段的内存分配、释放；
- 应用场景：页主要用于虚拟内存管理，用于实现内存分页和页面置换等；段主要用于进程的地址空间划分，用于将不同的数据和代码分开管理；

### CPU Cache 的数据结构和读取过程

**CPU Cache 是由很多个 Cache Line 组成的，CPU Line 是 CPU 从内存读取数据的基本单位，而 CPU Line 是由各种标志（Tag）+ 数据块（Data Block）组成（在内存中的数据，内存块（Block）），读取的时候我们要拿到数据所在内存块的地址；**

CPU Cache 的数据是从内存中读取过来的，它是以一小块一小块读取数据的，而不是按照单个数组元素来读取数据的，在 CPU Cache 中的，这样一小块一小块的数据，称为 **Cache Line**；

事实上，CPU 读取数据的时候，无论数据是否存放到 Cache 中，CPU 都是先访问 Cache，只有当 Cache 中找不到数据时，才会去访问内存，并把内存中的数据读入到 Cache 中，CPU 再从 CPU Cache 读取数据；

**地址映射**

- 直接映射 Cache：Direct Mapped Cache

  - 把内存块的地址始终「映射」在一个 CPU Cache Line（缓存块） 的地址，至于映射关系实现方式，则是使用「取模运算」，取模运算的结果就是内存块地址对应的 CPU Cache Line（缓存块） 的地址
  - **只适合大容量 Cache 采用**

- 全相连 Cache：Fully Associative Cache

  ![07163541_oEp6](https://cdn.jsdelivr.net/gh/lutianen/PicBed@master/07163541_oEp6.gif)

  - **主存中任何一块都可以映射到Cache中的任何一块位置上**，Cache的利用率高，块冲突概率低**，只要淘汰Cache中的某一块，即可调入主存的任一块**。
  - **由于Cache比较电路的设计和实现比较困难，这种方式只适合于小容量Cache采用。**

- 组相连 Cache：Set Associative Cache

  - 直接映射和全相联映射的折中方案，**主存和Cache都分组，主存中一个组内的块数与Cache中的分组数相同，组间采用直接映射，组内采用全相联映射**
  - **适度兼顾二者的优点，尽量避免二者的缺点，得到普遍采用**

### 内存访问示意图

**TLB采用组相联、页表采用两级页表、cache采用组相联、cache仅考虑L1 d-cache，不考虑L1 i-cache、L2 cache和L3 cache、未考虑页表缺页、简化了cache未命中情况**

- **==快表位于 MMU 内部，页表位于内存中==**

<img src="https://raw.githubusercontent.com/lutianen/PicBed/master/07163541_gEDQ.png" alt="07163541_gEDQ" style="zoom:;" />

### 计算机中，32 bit 和 64 bit 在虚拟地址空间、物理地址空间有什么区别？

64 bit 有两大优点：

- 可以进行更大范围的整数运算
- 可以支持更大的内存

**==32位计算机的虚拟地址空间大小为 $2^{32}$（4GB），即4GB的寻址空间。这个空间被分为用户空间和内核空间==**

==**64位计算机的虚拟地址空间大小不是 $2^{32}$，也不是 $2^{64}$，而一般是 $2^{48}$**==，因为程序并不需要 $2^{64}$ 那么大的寻址空间，而且过大的寻址空间会造成资源的浪费；因此 64 为 Linux 一般使用 48bit 表示虚拟地址空间，40bit 标识物理地址空间。

### 递归锁？

**读写锁**

  从广义上来讲也可以认为是一种共享版的互斥锁；

  如果对一个临界区大部分是读操作，而只有少量的写操作，读写锁在一定程度上能够降低线程互斥产生的代价。

**互斥锁**

  **递归锁 Recursive Mutex**，又称 **可重入锁（Reentrant Mutex）**，即 ==同一线程可以多次获取同一个递归锁，不会产生死锁==；因为可重入锁维护了一个获取计数和一个所有者线程，只有获取计数为 0 时，锁才会被释放；当同一个线程重复获取该锁时，获取计数加 1， 线程释放时，获取计数减 1.

  **非递归锁 Non-recursive Mutex**，又称 **不可重入锁（Non-Reentrant Mutex）**，即 ==同一线程多次获取同一个非递归锁，会导致死锁==

### 用户态与内核态的区别？

用户态（User Mode）和内核态（Kernel Mode）是操作系统中的两种不同的执行模式，它们在访问计算机硬件和执行特权指令方面有着显著的区别：

1. **权限级别**：
   - 用户态：在用户态下运行的程序只能执行受限的指令，不能直接访问操作系统的关键资源和硬件设备。用户态的程序没有特权，不能执行一些危险的操作。
   - 内核态：内核态下运行的操作系统内核具有最高的特权级别，可以访问计算机的所有资源，执行特权指令，管理硬件设备，并控制系统的运行。内核态的代码运行在操作系统的上下文中。
2. **资源访问**：
   - 用户态：用户态的程序只能通过系统调用（System Call）请求内核来访问资源，例如文件操作、网络通信等。这些请求由操作系统核准后才能执行。
   - 内核态：内核态的代码可以直接访问所有系统资源，包括物理内存、硬件设备、中断处理等，而无需经过系统调用。
3. **执行特权指令**：
   - 用户态：用户态下的程序不能执行特权指令，例如修改页表、启用/禁用中断等。
   - 内核态：内核态下的代码可以执行特权指令，以控制底层硬件和操作系统的关键功能。
4. **安全性**：
   - 用户态：用户态下的程序受到较强的隔离和保护，不能直接破坏操作系统的稳定性和安全性。
   - 内核态：内核态下的代码拥有更大的权限，因此必须谨慎编写，以确保系统的稳定和安全。
5. **切换开销**：
   - 用户态到内核态的切换需要进行上下文切换，这会引入一定的性能开销。这通常是通过中断或系统调用触发的。
   - 内核态到用户态的切换也需要上下文切换，但通常比内核态到用户态的切换开销小。

### 用户态到内核态的转换原理？

**==用户态的程序，能访问的内存空间和对象收到限制，其所占有的 CPU 可被抢占==，==内核态的程序，能访问所有的内存空间和对象，且所占有的 CPU 不允许被抢占==**

**系统调用**

  ==系统调用==，是用户态程序主动要求切换到内核态的一种方式，而 ==系统调用机制的核心是使用操作系统为用户特别开放的一个中断来实现==，例如 Linux x86 架构中 `int 80h` 软中断；

**异常**

  ==异常==，当 CPU 执行用户态的程序时，发生了某些事先不可知的异常，此时会触发有当前运行进程切换到处理磁异常的内核相关程序中，也就转到了内核态，例如 ==缺页异常==；

**外围设备硬件中断**

  ==硬件中断==，当外围设备完成用户请求的操作后，回想 CPU 发出相应的中断信号，此时 CPU 会去执行中断处理程序，如果先前执行的用户态程序，那么这个转换过程也就发生了用户态到内核态的切换。例如 ==硬盘读写操作完成==

### 当用户程序执行系统调用时，OS 发生了什么？

**系统调用号**

  ==系统调用号，是一个唯一标识系统调用的数字 `/usr/include/asm/unistd.h`，不同的系统有不同的系统调用号，系统调用号是在编译操作系统内核时被确定的==

**系统调用表**

  ==系统调用表，是一个在操作系统内核中的数据结构 `sys_call_table`，将系统调用号映射到相应的系统调用服务例程；它是在操作系统启动时被初始化的，且通常是静态的，不会在运行时改变==

**系统调用 OS 流程**

1. ==用户程序准备系统调用参数==
2. ==用户程序出发软件中断==：用户程序执行一个特殊的指令（x86 架构下的 ==`int 0x80` 或 `syscall`==），这个指令会触发一个软件中断，并将执行权交给操作系统
3. ==内核接管控制权==：CPU 相应中断，将执行模式从用户模式切换到内核模式，然后开始执行内核中的中断处理程序
4. ==内核检查系统调用号和参数==：中断处理程序会从一个特定的地方（如一个寄存器）==获取系统调用号==，然后根据这个号码找到对应的系统调用服务例程，同时也会获取并检查系统调用的参数
5. ==内核执行系统调用==：内核开始执行对应的系统调用服务例程（完成实际的工作，比如读写文件、创建进程等）
6. ==内核准备返回值==：系统调用执行完毕后，它的返回值会被放在一个特定的地方，如一个寄存器
7. ==内核返回用户程序==：操作系统将用执行模式从内核模式切换到用户模式，然后跳回用户程序的下一条指令，继续执行用户程序
8. ==用户程序获取返回值==：用户程序从一个特定的地方（如一个寄存器）获取系统调用的返回值

### 函数调用底层实现

**x86_64 通用寄存器**

> rax：函数返回值存放的寄存器；
>
> rsp：堆栈指针寄存器，指向栈顶位置，堆栈的 pop 和 push 操作就是通过改变 rsp 的值实现的；
>
> rbp：栈帧指针，用于标识当前栈帧的起始位置；
>
> rdi、rsi、rdx、rcx、r8、r9：用于存储函数调用时的6个参数（多于6个的参数，依然还是通过入栈实现传递）；
>
> rbx、rbp、r12、r13、r14、r15：由被调用者保护，在函数返回时要恢复这些寄存器中原本的值；

![image-20230705215337015](https://raw.githubusercontent.com/lutianen/PicBed/main/202307132351364.png)

- 为了效率，尽量使用少于 6 个参数的函数；
- 传递比较大的参数，尽量使用指针，因为寄存器只有 64 bit；

**函数调用**

> 在子函数调用时，需要切换上下文使得当前调用栈进入到一个新的执行中：
>
> - 父函数将调用参数从后向前压栈：有函数调用者完成；
> - 将返回地址压栈保存： call 指令完成；
> - 挑战到子函数起始地址执行：call 指令完成；
> - 子函数将父函数栈帧起始地址（%rpb）压栈：由函数被调用者完成；
> - 将 %rbp 的值设置为当前 %rsp 的值，即将 %rbp 指向子函数栈帧的起始地址：由函数被调用者完成（上下文中的 Callee 逻辑），完成函数上下文的切换；

- **保存返回地址和保存上一栈帧的 %rbp 都是为了函数返回时，恢复父函数的栈帧结构（保存函数调用上下文）；**
- **父函数中进行参数压栈时，顺序是从后向前进行的（调用栈空间都是从大地址向小地址延伸，这一点刚好和堆空间相反）**

### 函数调用和系统调用的区别？

- **函数调用通常在用户态执行，系统调用需要切换到内核态执行；**
- **函数调用通常通过直接调用函数名来执行，系统调用需要通过触发软中断（x86 架构下的 `int 0x80`）来执行；**
- **由于需要从用户态切换到内核态，系统调用的开销通常比函数调用大；**
- **函数调用的错误通常通过返回值或异常来处理，系统调用的错误通过设置全局变量（`errno`）来处理；**
- **函数调用（由编程语言定义）是可移植的，系统调用（由操作系统定义）通常不可移植；**

|                       函数调用                       |                     系统调用                      |
| :--------------------------------------------------: | :-----------------------------------------------: |
| 在所有的 ANSI C 编译器版本中，C 库函数是 ==相同== 的 |       各个操作系统的系统调用是 ==不同== 的        |
|          调用 ==函数库== 里的一段程序或函数          |             调用 ==系统内核== 的服务              |
|                 与 ==用户程序== 联系                 |             ==操作系统== 的一个入口点             |
|                 ==用户地址空间执行==                 |               ==内核地址空间执行==                |
|              运行时间属于 ==用户时间==               |             运行时间属于 ==系统时间==             |
|           属于 ==过程调用，调用开销较小==            | 需要在 ==用户态和内核态进行上下文切换，开销较大== |
|         在 C 函数库 libc 中大约有 300 个函数         |          在 UNIX 中大约有 90 个系统调用           |

### 虚拟内存？虚拟地址空间？

<img src="https://raw.githubusercontent.com/lutianen/PicBed/master/memory-location1.png" alt="memory-location1" style="zoom: 80%;" /><img src="https://raw.githubusercontent.com/lutianen/PicBed/master/memory-location3.png" alt="memory-location3" style="zoom: 67%;" />

**虚拟内存**

  虚拟内存是一种内存管理技术，会使程序自己认为自己拥有一块很大且连续的内存，然而这个程序在内存中不是连续的，并且有些还会在磁盘上，在需要时进行数据交换；

  **优点**

​    可以弥补物理内存大小的不足；一定程度上提高反应速度；减少对物理内存的读取从而保护内存，延长内存使用寿命；

  **缺点**

​    占用一定的物理磁盘空间；加大了对磁盘的读写；设置不当会影响整机稳定性和速度；

**虚拟地址空间**

  虚拟地址空间，是对于单一进程的概念，使得这个进程看到的将是地址从 0x00000000 开始的整个内存空间，并且好像自己在独占主存；从低地址到高地址：代码段、数据段、堆段、内存映射区、栈段、内核空间；

### 什么是“活锁”？

“**活锁**”，指的是==多个线程使用 `trylock` 系列的函数时，由于互相谦让，导致即使在某段时间内锁资源可用，也可能导致需要锁资源的线程获取不到锁==。

实际编码时，应尽量避免让过多的舷侧和国内使用 `trylock` 请求锁，以免出现活锁，造成资源浪费。

### 线程安全？如何实现？

==线程安全，指的是在多线程环境中，多个线程同时访问一个共享资源时不会出现意外的结果==，即 ==在并发环境下，线程安全的程序不会出现数据竞争、死锁、活锁等问题==

**实现方式**

- 加锁：利用互斥量来对不安全对象进行加锁，实现线程执行的串行化，从而保证多线同时操作对象的安全性

- 原子操作：使用原子操作来保证共享资源的不可分割，任何时刻只有一个线程能够进行操作；

- 无锁编程：使用无锁编程技术，比如 CAS（Compare and Swap）、SC、FAI、TAS等，实现并发情况下的数据同步

      ==CAS (Compare and Swap)==：CAS是一种原子操作，它首先比较一个内存地址的值与一个给定的值，如果相等，就用一个新的值替换这个内存地址的值。CAS在多线程编程中经常用来实现乐观锁。例如，在Java中，AtomicInteger类就是使用CAS实现的。
        
      ==SC (Swap and Compare)==：SC是一种原子操作，它首先将一个内存地址的值替换成一个给定的值，然后比较这个内存地址的原值与另一个给定的值。如果相等，就说明替换成功，否则就需要再次尝试。SC操作通常用来实现锁的同步。
        
      ==FAI (Fetch and Increment)==：FAI是一种原子操作，它首先读取一个内存地址的值，然后将这个值加1，并将加1后的值写回到内存地址中。FAI操作常用于实现计数器。
        
      ==TAS (Test and Set)==：TAS是一种原子操作，它首先读取一个内存地址的值，如果这个值为0，则将这个内存地址的值设置为1，并返回0，否则返回1。TAS操作通常用于实现自旋锁，即当一个线程尝试获取锁时，如果锁已经被其他线程占用，它会不断地尝试获取锁，直到获取成功。

- 线程本地存储：使用线程本地存储（Thread Local Storage ，TLS），每个线程都有自己独立的一份数据副本，避免了线程间的数据竞争

- 避免共享：尽量避免使用共享资源，通过分解任务、拆分数据结构等方式，将共享资源转化为局部变量，从而避免了线程间的竞争

    ```c++
    #include <atomic>
    #include <iostream>
    #include <memory>
    #include <thread>
    #include <vector>
    
    const int THREAD_NUM = 4;
    std::atomic<int> counter;
    
    void initCounter(std::atomic<int>& cnt) { cnt = 0; }
    
    void incrementCounter(std::atomic<int>& cnt) {
        int expected, desired;
        do {
            expected = cnt;
            desired = expected + 1;
        } while (!cnt.compare_exchange_weak(expected, desired));
    }
    
    int main() {
        std::vector<std::unique_ptr<std::thread>> threads;
        initCounter(counter);
    
        for (size_t i = 0; i < THREAD_NUM; ++i) {
            std::unique_ptr<std::thread> pt(new std::thread([&]() {
                for (size_t i = 0; i < 100; ++i) {
                    incrementCounter(counter);
                }
                return nullptr;
            }));
            threads.push_back(std::move(pt));
        }
    
        for (auto& pt : threads) {
            pt->join();
        }
    
        std::cout << "counter = " << counter << std::endl;
        return 0;
    }
    ```

### 非线程安全函数为什么导致结果和预期不同？

可能的情况：**非线程安全的函数内部使用了一个全局变量或者内部的静态变量，导致前一次的调用结果被后一次的调用结果覆盖了**。

解决方案：**在函数内部改用线程局部存储技术代替原来使用静态变量或者全局变量**

### Linux 设备中 “字符设备” 和 “块设备” 有什么主要区别？请分别举例说明

字符设备：

- 字符设备，是==**能像字节流（类似文件）一样被访问的设备，由字符设备驱动程序来实现这种特性**==。

- 通常至少实现 `open` , `close` , `read` 和 `write` 系统调用。

- 常见字符设备：字符终端、串口、鼠标、键盘、摄像头、声卡、显卡等

- 字符设备信息查看：

    `lsmod` 可以查看模块的依赖关系

    `modprobe` 在加载模块时会加载其他依赖模块

    `/proc/interrupt` 该文件可以查看当前使用的中断号

块设备：

- 也是通过 `/dev` 目录下的文件系统节点来访问。块设备上能够容纳文件系统，例如 U盘，SD卡，磁盘等

**==字符设备和块设备的区别仅仅在于内核内部管理数据的方式，即内核和驱动程序之间的软件接口，而这些不同点对用户来说是透明的==**。

### 编程中有一个概念叫互相踩内存，“踩内存”是什么概念？

“踩内存”，指的是 **访问了不应该访问的内存地址**。

踩内存就是访问了不应该访问的内存地址，比如过说数组或内存越界啊，指针未初始化啊或使用的时候已经被释放掉啊等都会造成踩内存的现象。

### 什么是Linux内核，有什么作用？组成部分？

系统的核心程序，运行程序和管理磁盘、打印机等硬件设备

一个完整的 Linux 内核由 5 部分组成：内存管理、进程管理、进程间通信、虚拟文件系统、网络接口；

**内存管理**：

　　==内存管理主要完成的是如何合理有效地管理整个系统的物理内存，同时快速响应内核各个子系统对内存分配的请求。==

　　Linux内存管理==支持虚拟内存==，而多余出的这部分内存就是通过磁盘申请得到的，平时系统只把当前运行的程序块保留在内存中，其他程序块则保留在磁盘中。在内存紧缺时，内存管理负责在磁盘和内存间交换程序块。

**进程管理** ：

　　==进程管理主要控制系统进程对CPU的访问==。当需要某个进程运行时，由进程调度器根据基于优先级的调度算法启动新的进程。

**进程间通信** ：

　　==进程间通信主要用于控制不同进程之间在用户空间的同步、数据共享和交换==。由于不用的用户进程拥有不同的进程空间，因此==进程间的通信要借助于内核的中转来实现==。

　　一般情况下，当一个进程等待硬件操作完成时，会被挂起。当硬件操作完成，进程被恢复执行，而协调这个过程的就是进程间的通信机制。

**虚拟文件系统**：

　　==Linux内核中的虚拟文件系统用一个通用的文件模型表示了各种不同的文件系统，这个文件模型屏蔽了很多具体文件系统的差异，使Linux内核支持很多不同的文件系统==。

　　这个文件系统可以分为逻辑文件系统和设备驱动程序：逻辑文件系统指Linux所支持的文件系统，例如ext2、ext3和fat等；设备驱动程序指为每一种硬件控制器所编写的设备驱动程序模块。

**网络接口**：

　　==网络接口提供了对各种网络标准的实现和各种网络硬件的支持==。网络接口一般分为网络协议和网络驱动程序。网络协议部分负责实现每一种可能的网络传输协议。

　　网络设备驱动程序则主要负责与硬件设备进行通信，每一种可能的网络硬件设备都有相应的设备驱动程序。

### Linux系统的目录 /usr、/home、/bin、/dev、/var、/etc中主要存放什么文件？

`/usr` linux系统中占用硬盘空间最大的目录（**Unix System Resources**）。用户的很多应用程序和文件都存放在这个目录下

`/home` 存放用户的主目录

`/bin` 二进制可执行命令

`/dev` 包含了所有 Linux 系统中使用的外部设备

`/var` 存放着不断在扩充的东西

`/etc` 存放系统管理时要用到的各种配置文件和子目录

### 文件系统

<img src="https://raw.githubusercontent.com/lutianen/PicBed/main/202307241532645.png" alt="img" style="zoom:50%;" />      <img src="https://raw.githubusercontent.com/lutianen/PicBed/main/202307241533913.png" alt="img" style="zoom: 67%;" />

![img](https://raw.githubusercontent.com/lutianen/PicBed/main/202307241533902.webp)              <img src="https://raw.githubusercontent.com/lutianen/PicBed/main/202307241533454.png" alt="早期 Unix 文件系统" style="zoom: 33%;" />

**==虚拟文件系统==**

  文件系统种类众多（EXT2/3/4，XFS，NFS等），而操作系统希望==对用户提供统一的接口==，于是在用户层和文件系统层之间引入了**==虚拟文件系统 VFS（Virtual File System）==**；

  VFS 定义了一组所有文件系统都支持的数据结构和标准接口；

### 请介绍一下 Linux 中的文件设备类型？

**当使用 `ls -al` 时，第一个字符如何显示**

- 普通文件：符号`-`，使用 `rm` 删除，包括文本文件、二进制文件等
- 目录文件：符号`d`，使用 `rmdir` 删除（空文件夹）或 `rm -r` 删除，用于存储其他文件和目录的信息
- 字符设备文件：符号`c`，提供对字符流的访问的设备，例如键盘、鼠标、串口等
- 块设备文件：符号`b`，提供对数据以固定大小的块进行访问的设备，例如硬盘、固态硬盘等
- 套接字文件：符号`s`，用于进程间通信的一种方式
- 符号链接文件：符号`l`，指向另一个文件或目录的链接
- 具名管道文件：符号`p`，用于进程间通信的一种方式

### 延迟队列的底层实现？

延迟队列，一种特殊的队列，它可以在一定的时间延迟后才将元素从队列中取出。延迟队列，通常用于定时任务调度等功能。

一种常见的底层实现方式：有序链表存储元素，元素包含两个字段：`元素值` 和 `过期时间`。

**有序链表按照元素的过期时间从小到大排序，即越早过期的元素越靠近链表头部。在插入元素时，需要将元素插入到合适的位置，以保持链表的有序性。**

**延迟队列中的元素通常是被一个守护线程（Daemon Thread）或者定时器线程唤醒的。每个线程启动时会创建一个定时器线程，该线程会不断从有序链表的头部取出过期元素，则将其加入到一个任务队列中，由任务执行线程依次执行。**

### **简述LINUX系统从上电开始到系统起来的主要流程？**

1. ==**上电自检（Power-On Self Test，POST）**==：这是计算机系统上电后首先进行的自检过程，由BIOS（基本输入输出系统）完成，用以检查硬件设备的基本功能是否正常。
2. ==**启动BIOS**==：BIOS负责初始化硬件设备，设置系统参数等工作，然后从设定的启动设备中加载引导程序。
3. ==**启动引导程序（Bootloader）**==：引导程序通常存放在硬盘的MBR（主引导记录）或EFI分区中，如GRUB、LILO等。引导程序的主要工作是加载内核。
4. ==**加载内核**==：引导程序找到内核映像文件，将其加载到内存中，并传递一些参数。
5. ==**内核初始化**==：内核接管系统后，会进行一系列的初始化工作，包括初始化内存管理、设备驱动、进程调度等。
6. ==**启动init进程**==：内核初始化完成后，会启动PID为1的init进程，这是Linux系统中的第一个进程。
7. ==**运行init脚本**==：init进程依据/etc/inittab文件或者systemd的配置，运行相应的启动脚本，这些脚本会启动各种系统服务。
8. ==**用户登录**==：启动脚本执行完毕后，系统进入运行级别，显示登录提示，用户可以输入用户名和密码登录系统。

### 判断大小端的两种方法

```cpp
bool isBigEndian() {
    uint32_t a = 0x12345678;
    auto* pa = &a;
    if (*(reinterpret_cast<char*>(pa)) == 0x12) {
        // std::cout << "Little Endian" << std::endl;
        return false;
    } else if (*(reinterpret_cast<char*>(pa)) == 0x78) {
        // std::cout << "Big Endian" << std::endl;
        return true;
    } else
        exit(-1);
}

bool isLittleEndian() {
    union T {
        int i;
        char c;
    } dat;
    dat.i = 0x1234;
    if (dat.c == 0x12) {
        // std::cout << "Little Endian" << std::endl;
        return true;
    } else if (dat.c == 0x34) {
        // std::cout << "Big Endian" << std::endl;
        return false;
    } else
        exit(-1);
}
```

### Helloworld 程序从开始到打印到屏幕上的全过程？

1. 用户告诉操作系统执行 Helloworld 程序（键盘输入等）
2. 操作系统：找到 Helloworld 程序的相关信息，并检查其类型是否为可执行文件；通过程序头部信息，确定代码和数据在可执行程序文件中的位置，并计算出对应的磁盘块地址；
3. 操作系统：创建一个新进程，将 Helloworld 可执行文件映射到该进程结构中，表示由该进程执行 Helloworld 程序；
4. 操作系统：为 Helloworld 程序设置 CPU 上下文环境，并跳到程序开始处
5. 执行 Helloworld 程序的第一条指令，发生缺页异常
6. 操作系统：分配一页物理内存，并将代码从磁盘读入内存，然后继续执行 Helloworld 程序；
7. Helloworld 程序执行 puts函数（系统调用），在显示器上写一个字符；
8. 操作系统：找到要将字符串送往的显示设备，通常是有一个进程控制，因此操作系统将要写的字符串送给该进程
9. 操作系统：控制设备的进程告诉设备的窗口系统，它要显示该字符串，窗口系统确定这是一个合法的操作，然后将字符串转换成像素，并写入设备的存储映像区
10. 视频硬件将像素转换成显示器可接收和一组控制数据信号
11. 显示器解释写好，激发液晶屏

## C++

### `static` 的用法

1. `static` 修饰局部变量：使其变为静态存储方式（静态数据区），且这个局部变量在函数执行完毕后不会释放，而是继续保留在内存中；
2. `static` 修饰全局变量：使其只在本文件内部有效，而其他文件不可连接或引用该变量；
3. `static` 修饰函数：对函数的链接方式产生影响，使得函数只在本文件内部有效，对其他文件不可见；又叫静态函数，不用担心与其他文件的同名函数产生干扰；

### `const` 的用法

1. `const` 修饰常量：定义时就初始化，以后无法修改；

    ```c++
    const int a; // a 为常量
    int const a; // 等价于 const int a
    const int* a; // const 修饰 int*，即 a 的数组不可变
    int* const a; // const 修饰 a, 即指针 a 的指向不可变
    const int* const a;
    ```

2. `const` 修饰形参：`void func(const int a) {}` ，该形参在函数内不可修改；

3. `const` 修饰类成员函数：`void C::func(void) const {}`:

    - 该函数对成员变量只能进行读操作，即 `const` 类成员函数不能修改成员变量数值；
    - **C++ 不允许 `const` 对象调用没有以 `const` 修饰的成员函数**，否则编译报错；**因此对于大部分成员函数需要考虑是否需要提供 `const` 和 非 `const` 修饰的两个版本，且他们的行为也不一定一致；

### 区分变量存储在哪个段？

```c++
const int THREAD_NUM = 4; // 常量，数据段（.data段）
int THREAD_NUM_TEST = 4; // 全局变量，数据段（.data段）

int main() {
    std::cout << &THREAD_NUM << std::endl;
    std::cout << &THREAD_NUM_TEST << std::endl;
    int num = 42;      // 局部变量，栈段
    std::cout << &num << std::endl;
    const int num1 = 42;    // 局部常量，栈段
    std::cout << &num1 << std::endl;
    static const int num2 = 42;   // 静态局部变量，数据段（.data段）
    std::cout << &num2 << std::endl;
}
```

### `sizeof` 与 `strlen`？

```c++
int a[] = {1, 2, 3, 4, 5};
std::cout << sizeof(a) << std::endl;   // 20 = 4 * 5
std::cout << sizeof(*a) << std::endl;  // 4
char* p = "Lute-Ziwi";
std::cout << sizeof(p) << std::endl;                             // 8
std::cout << sizeof(*p) << std::endl;                            // 1
std::cout << sizeof(*(reinterpret_cast<int*>(p))) << std::endl;  // 4
std::cout << std::strlen(p) << std::endl;                        // 9

char str[] = "Lute";
std::cout << sizeof(str) << std::endl;  // 5
std::cout << std::strlen(str) << std::endl; // 4

std::cout << sizeof(1.2 + 2) << std::endl;   // 8
std::cout << sizeof(1.2f + 2) << std::endl;  // 4

int i = 1;
std::cout << sizeof(i++) << sizeof(++i) << std::endl;  // 4 4
std::cout << i << std::endl;                      // 1

struct T {
    int a;     // Offset = 0, Size = 4
    float f;   // Offset = 4, Size = 4
    double d;  // Offset = 8, Size = 8
    char c;    // Offset = 16, Size = 1
    char c_;   // Offset = 17, Size = 1, Padding = 6
};
std::cout << sizeof(T) << std::endl;  // 24

// x64
union U {
    long i;  // Size = 8, Padding = 16
    int k[5]; // Size = 20, Padding = 4
    char c;  // Size = 1, Padding = 23
};
std::cout << sizeof(U) << std::endl; // 24
```

### 哪个运算符类型必须是整数

**==位运算符(Bitwise Operator), 只能操作整型数据，不能操作浮点型数据==**

取余运算符 `%`、位运算符 `&` / `|` / `^` / `~`

### `strcat`、`strncat`、`strcmp`、`strcpy` 哪些函数会导致内存溢出?

导致内存溢出的函数：`strcat`, `strncat`, `strcpy`

- `strcat`：将一个字符串追加到另一个字符串的末尾，如果目标字符串的内存空间不足，就会导致内存溢出；
- `strncat`：与strcat类似，但是它会指定最大追加的字符数，避免完全依赖目标字符串的大小。然而，如果目标字符串的空间仍然不足以容纳源字符串，仍然可能导致内存溢出。
- `strcpy`：将一个字符串复制到另一个字符串，如果目标字符串的空间不足以容纳源字符串，就会导致内存溢出；

### 说一下你理解的 C++ 中的四种智能指针

为什么要使用智能指针：**智能指针其作用是管理一个指针,避免咋们程序员申请的空间在函数结束时忘记释放,造成内存泄漏这种情况滴发生。**

使用智能指针可以很大程度上的避免这个问题, 因为智能指针就是一个类, 当超出了类的作用域是, 类会自动调用析构函数, 析构函数会自动释放资源。

智能指针的作用原理就是在函数结束时自动释放内存空间, 不需要手动释放内存空间。

**常用接口**

```cpp
// 用来获取原生指针
T* get();
// 
T& operator*();
T* operator->();
T& operator=(const T& val);
// 将封装在内部的指针置为 nullptr, 但并不会破坏指针所指向的内容，函数返回的是内部指针置空之前的内容
T* release();
// 直接释封装在内部指针所指向的内存，并将内部指针初始化为 other
void reset(T* other = nullptr);
```

![](https://cdn.jsdelivr.net/gh/lutianen/PicBed@master/202309251715171.svg)

```cpp
template <class T>
class SharedPtr{
public:
    SharedPtr() : cnt_(new size_t(0)), ptr_(nullptr) {}
    SharedPtr(T* ptr) : cnt_(new size_t(1)), ptr_(ptr) {}
    ~SharedPtr() {
        --(*cnt_);
        if(0 == *cnt_) {
            delete cnt_;
            delete ptr_;
            cnt_ = nullptr;
            ptr_ = nullptr;
        }
    }

    SharedPtr(const SharedPtr& other) {
        cnt_ = other.cnt_;
        ptr_ = other.ptr_;
        ++(*cnt_);
    }
    SharedPtr& operator=(const SharedPtr& other) {
        if (ptr_ != other.ptr_) {
            if(ptr_ && --(*cnt_) == 0) {
                delete ptr_;
                delete cnt_;
            }
            
            ptr_ = other.ptr_;
            cnt_ = other.ptr_;
            if(ptr_) ++(*cnt_);
        }
        return *this;
    }
    SharedPtr(SharedPtr&& other) : cnt_(other.cnt_), ptr_(other.ptr_) {
        ++(*cnt_);
        other.cnt_ = nullptr;
        other.ptr_ = nullptr;
    }
    SharedPtr& operator=(SharedPtr&& other) {  
        if (this != &other) {
            delete ptr_;
            delete cnt_;
            cnt_ = other.cnt_;
            ptr_ = other.ptr_;
            other.cnt_ = nullptr;
            other.ptr_ = nullptr;
        }
        return *this;
    }

    T& operator*() { return *ptr_; }
    T* operator->() { return ptr_; }
    operator bool() { return ptr_ == nullptr; }
    T* get() { return ptr_; }

    size_t use_count() { return *cnt_; }
    bool unique() { return *cnt_ == 1; }
    void swap(SharedPtr& other) { std::swap(*this, other); }

private:
    size_t* cnt_;
    T* ptr_;
};
```

**auto_ptr**

- 采用所有权模式
- `p2 = p1;` p2 会剥夺 p1 的所有权，如再次访问 p1 将会报错
- 缺点：存在潜在的内存崩溃问题

**unique_ptr**

- 采用独占式模式，保证同一时间只有一个智能指针可以指向该对象

**shared_ptr**

- 实现共享拥有概念，多个智能指针可以指向相同的对象，该对象和相关资源会在 ”最后一个引用被销毁时“ 释放
- 可以通过 `use_count()` 来查看资源的拥有者个数
- 可以通过 `new`来构造，也可以通过传入 `unique_ptr` , `weak_ptr` 来构造
- 当调用 `release` 时，当前指针会释放资源所有权，引用计数减一；当引用计数为 0 时，释放资源

> **使用 `shared_ptr` 的注意事项(陷阱)**
>
> 1. 不要用同一个原始指针初始化多个 `shared_ptr`
>
>       ```cpp
>       // 错误示例
>       int *ptr = new int;
>       shared_ptr<int> p1(ptr);
>       shared_ptr<int> p2(ptr);  // logic error
>       ```
>
> 2. 不要在函数实参中创建 `shared_ptr`
>
>       `function (shared_ptr<int>(new int), g()); // 存在缺陷`
>
>       原因：C++ 的函数参数的计算顺序在不同的编译器不同的调用约定可能是不一样的，一般是从右到左，但也可能是从左到右，所以可能的过程是先 `new int` 然后调用 `g()`；如果恰好 `g()` 发生异常，而 `shared_ptr<int>` 还没创建，则 `int` 内存泄漏；
>
>       正确做法是：**先创建智能指针`shared_ptr<int> p(new int()); f(p, g());`**
>
> 3. 通过 `shared_from_this()` 返回 `this` 指针
>
>       不要将 `this指针` 作为 `shared_ptr` 返回出来，因为 `this 指针` 本质上是一个裸指针，如此可能导致重复析构；
>
>       ```cpp
>       struct A {
>         shared_ptr<A> getSelf() { return shared_ptr<A>(this); } // Don't do this !!!
>       };
>       int main () {
>         shared_ptr<A> sp1(new A);
>         shared_ptr<A> sp2 = sp1->getSelf();
>         return 0;
>       }
>       /*
>       由于用同一个指针（this）构造了两个指针指针 sp1 和 sp2，而他们之间是没有任何关系的，在离开作用域之后 this 将会被构造的两个智能指针各自析构，导致重复析构的错误；
>       */
>       ```
>
>       正确做法：让目标类通过派生 `std::shared_from_this<T>` 类，然后使用基类的成员函数 `shared_from_this()` 来返回 `this` 的 `shared_ptr`
>
>       ```cpp
>       class A : public std::enable_shared_from_this<A> {
>         std::shared_ptr<A> getSelf() { return shared_from_this(); }
>       };
>       
>       std::shared_ptr<A> spy(new A);
>       std::shared_ptr<A> p = spy->getSelf();
>       ```
>
> 4. 要避免循环引用
>
>       智能指针最大的一个陷阱：循环引用，会导致内存泄漏

**weak_ptr**

- 一种不控制对象生命周期的智能指针，指向一个 `shared_ptr` 管理的对象
- 只提供对管理对象的一个访问手段：设计目的是为配合 `shared_ptr` 而引入的一种智能指针，来协助 `shared_ptr` 工作。**解决 `shared_ptr` 相互引用时的死锁问题**
- 只可以从一个`shared_ptr` 或另一个 `weak_ptr` 对象构造
- 它的构造、析构不会导致引用计数的增加、减少

```cpp
char* s = "12345678";
std::cout << *s++ << std::endl;    // 1
std::cout << *(s++) << std::endl;  // 2
std::cout << *++s << std::endl;    // 4
std::cout << ++*s << std::endl;    // SIGSEGV(Address boundary error)
```

### shared_ptr发生死锁怎么办？

`shared_ptr`相互引用产生死锁：两个 `shared_ptr` 相互引用，那么这两个 `shared_ptr` 指针的引用计数永远不会降为 0 ，资源永远不会释放。

解决方案：`weak_ptr` 是对对象的一种弱引用，它不会增加对象的 `use_count`，`weak_ptr` 和 `shared_ptr` 可以互相转化：`shared_ptr` 可以直接赋值给 `weak_ptr` , `weak_ptr`  也可以通过调用 `lock` 函数来获得 `shared_ptr`.

### C++ 中的指针参数传递和引用参数传递

**指针参数传递本质上是值传递，只不过传递的是一个地址值。**

引用参数传递过程中，存放的调用者放进来的**实参变量的地址**。被调用函数对形参（本体）的任何操作都被处理成**间接寻址**，即通过栈中存放的地址访问主调用函数的实参变量。

**从编译的角度来讲**，程序在编译时分别将指针和引用添加到符号表上，**符号表中记录的变量名以及变量所对应地址，且生成后就不会再改**。==指针变量在符号表上对应的地址值为指针变量的地址值==，而==引用在符号表上对应的地址值为引用对象的地址值（与实参名字不同，地址相同）==。

### 简单说一下函数指针

定义：函数指针是指向函数的指针变量。该变量存储的是函数地址；

用途：调用函数、函数的参数，比如回调函数

### ++ 的前置和后置

`operator++(int)` 这个重载是后置自增 `p++`，不带任何参数的`operator++()`个重载是前置自增；

- `++p` 会返回自增后的值 `p + 1`，一个==左值引用==；
- `p++` 返回自增前的值 `p`，因此需要先保存旧的迭代器，然后自增，再返回旧迭代器，导致比较低效；

### C 和 C++ 的区别

**C 的 struct 更像是一个数据结构的实现体，而 C++ 的 class 更像是一个对象的实现体**

- 语法和关键字
- 重载和虚函数
- 类方面：C 的 struct 和 C++ 的 struct / class
- 模板、STL 标准库

### C++ 中是怎么定义常量的？常量存放在内存的哪个位置？

- 局部常量：存放在栈区
- 全局常量：编译期一般不分配内存，存放在符号表中以提高访问效率
- 字面值常量：比如字符串，存放在常量区

> **对于 `float a[3][4]` 编译器实际上会把他变成一维数组 `float a[3*4]`，然后把 `a[i][j]` 翻译为 `a[i * 4 + j]`，==且在作为参数传递时，退化为指针==**

### 介绍 C++ 所有的构造函数

类的对象被创建时, 编译系统为对象分配内存空间, 并自动调用构造函数, 由构造函数完成成员的初始化工作。

**构造函数的作用: 初始化对象的数据成员。**

**无参数构造函数**: 即默认构造函数,如果没有明确写出无参数构造函数,编译器会自动生成默认的无参数构造函数,函数为空,什么也不做,如果不想使用自动生成的无参构造函数,必需要自己显示写出一个无参构造函数。

**一般构造函数**:也称重载构造函数,一般构造函数可以有各种参数形式,一个类可以有多个一般构造函数,前提是参数的个数或者类型不同,创建对象时根据传入参数不同调用不同的构造函数。

**类型转换构造函数**:根据一个指定类型的对象创建一个本类的对象,也可以算是一般构造函数的一种,这里提出来,是想说有的时候不允许默认转换的话,要记得将其声明为 `explict` 的,来阻止一些隐式转换的发生。

**拷⻉构造函数**:拷⻉构造函数的函数参数为对象本身的引用,用于根据一个已存在的对象复制出一个新的该类的对象,一般在函数中会将已存在的对象的数据成员的值一一复制到新创建的对象中。**如果没有显示的写拷⻉构造函数,则系统会默认创建一个拷⻉构造函数,但当类中有指针成员时,最好不要使用编译器提供的默认的拷⻉构造函数,最好自己定义并且在函数中执行深拷⻉。**

**赋值构造函数**:**注意,这个类似拷⻉构造函数,将=右边的本类对象的值复制给=左边的对象**,它不属于构造函数,=左右两边的对象必需已经被创建。如果没有显示的写赋值运算符的重载,系统也会生成默认的赋值运算符,做一些基本的拷⻉工作。

```cpp
// 调用赋值构造函数
A a1, A a2; a1 = a2;

// 调用拷贝构造，因为 a2 并未存在
A a1; A a2 = a1;
```

### C++ 中哪些字符不能重载？

1. `::`（作用域解析运算符）
2. `.`（成员访问运算符）
3. `.*`（成员指针访问运算符）
4. `?:`（条件运算符）
5. `sizeof`（大小运算符）
6. `typeid`（类型信息运算符，在 RTTI 中使用）
7. `static_cast`（静态类型转换）
8. `dynamic_cast`（动态类型转换）
9. `const_cast`（常量类型转换）
10. `reinterpret_cast`（重新解释类型转换）
11. `new`（动态内存分配运算符）
12. `delete`（动态内存释放运算符）
13. `new[]`（动态内存分配数组运算符）
14. `delete[]`（动态内存释放数组运算符）

> 赋值运算符 `=`，地址运算符 `&`，下标运算符 `[]`，函数调用运算符 `()` 和类成员访问运算符 `->` 可以被重载，但如果你没有为它们提供自定义的版本，编译器会为你的类自动生成默认版本

### C++ 中多线程的理解？

线程同步：多个线程之间可能会共享数据，为了避免数据竞争和不确定行为，需要进行线程同步。常见的同步机制包括互斥量（`std::mutex`）、条件变量（`std::condition_variable`）、原子操作（`std::atomic`）等。

**有锁编程（Lock-based Programming），也称为传统的并发编程方式**

- `std::mutex` 是一种互斥量，用于保护共享数据的访问。通过使用互斥量，可以确保一次只有一个线程可以访问被保护的代码块或数据。
- 条件变量：`std::condition_variable` 是一种线程间的同步机制，用于实现线程之间的等待和通知机制。它允许一个线程等待某个条件满足，而另一个线程在条件满足时通知等待的线程继续执行。

**无锁编程（Lock-Free Programming）**

  ==无锁编程（Lock-free Programming）是一种并发编程的技术，旨在避免使用传统的互斥锁（mutex）和同步原语，以提高并发性和性能，并减少线程间的竞争和串行化==；

  无锁编程通常适用于对==性能要求较高、并发访问较频繁的场景==，对于简单的并发问题，使用传统的互斥锁可能更为简单和可靠

- 原子操作：`std::atomic` 提供了操作共享数据的原子操作，避免了数据竞争和并发访问的问题。原子操作可以确保对数据的读取和修改是不可分割的，不会被其他线程中断。
- 自旋锁（Spinlock）：自旋锁是一种无锁同步机制，它使用原子操作来实现线程之间的互斥。当线程无法获取锁时，它会以自旋的方式不断尝试获取锁，而不是阻塞等待。C++ 中可以使用 `std::atomic_flag` 来实现自旋锁。
- 无锁数据结构：无锁编程还涉及设计和实现无锁数据结构，如无锁队列、无锁栈和无锁哈希表等。这些数据结构使用原子操作和其他技术来确保并发访问的正确性和一致性。

### 生产者-消费者

```cpp
template <typename T>
class MTQueue {
    public:
    void push(T val) {
        std::unique_lock<std::mutex> lock(mtx_);
        vec_.push_back(std::move(val));
        cv_.notify_one();
    }

    void pushMany(std::initializer_list<T> vals) {
        std::unique_lock<std::mutex> lock(mtx_);
        std::copy(std::move_iterator<decltype(vals.begin())>(vals.begin()),
                  std::move_iterator<decltype(vals.end())>(vals.end()),
                  std::back_insert_iterator<std::vector<T>>(vec_));
        cv_.notify_all();
    }

    T pop() {
        std::unique_lock<std::mutex> lock(mtx_);
        cv_.wait(lock, [this]() { return !vec_.empty(); });
        T temp = std::move(vec_.back());
        vec_.pop_back();
        return temp;
    }

    std::pair<T, std::mutex> popHold() {
        std::unique_lock<std::mutex> lock(mtx_);
        cv_.wait(lock, [this]() { return !vec_.empty(); });
        T ret = std::move(vec_.back());
        vec_.pop_back();
        return std::pair<T, std::mutex>(std::move(ret), std::move(lock));
    }

    private:
    std::condition_variable cv_;
    std::mutex mtx_;
    std::vector<T> vec_;
};

int main() {
    MTQueue<int> foods;
    std::thread t1([&]() {
        for (int i = 0; i < 2; ++i) {
            auto food = foods.pop();
            std::cout << food << std::endl;
        }
    });

    std::thread t2([&]() {
        for (int i = 0; i < 2; ++i) {
            auto food = foods.pop();
            std::cout << food << std::endl;
        }
    });

    foods.push(12);
    foods.push(13);
    foods.pushMany({14, 15});

    t1.join();
    t2.join();
    return 0;
}
```

### `n` 个线程交替打印当前线程的 ID (0 ~ n-1)，重复打印 `m` 次

> `std::thread` 的构造函数会对传递的参数进行拷贝，而不是直接使用引用：`std::ref(a)`
>
> ```cpp
> #include <thread>
> 
> int main() {
>     int a = 1;
>     std::thread t1(
>         [](int& a) {
>             std::cout << a << std::endl;
>             a++;
>             std::cout << a << std::endl;
>         },
>         std::ref(a));
> 
>     t1.join();
>     std::cout << a << std::endl;
> }
> ```

```cpp
#include <atomic>
#include <iostream>
#include <thread>
#include <vector>

using namespace std;
atomic_int g_cnt{0};
atomic_int g_m{0};
vector<thread> g_threads;

int main() {
    int n = 4;
    int m = 2;
    g_m.store(m);

    for (int i = 0; i < n; ++i)
        g_threads.push_back(thread(
            [](int threadId) {
                while (g_m.load() > 0) {
                    if (g_cnt.load() == threadId) {
                        cout << threadId << endl;
                        g_cnt.fetch_add(1);
                        if (g_cnt.load() == 4) {
                            g_m.fetch_sub(1);
                            g_cnt = 0;
                        }
                    }
                }
            },
            i));

    for (auto& t : g_threads) t.join();
    return 0;
}
```

```c++
#include <atomic>
#include <iostream>
#include <memory>
#include <thread>
#include <vector>

std::atomic_int g_thread_cnt(0);
std::atomic_int g_repeat_cnt;
std::vector<std::unique_ptr<std::thread>> g_thread_pool;

int main() {
  int thread_num = 4;
  g_repeat_cnt.store(5);

  for (int i = 0; i < thread_num; ++i) {
    std::unique_ptr<std::thread> thr =
        std::make_unique<std::thread>(std::thread(
            [&](int thread_id) {
              while (g_repeat_cnt.load() > 0) {
                if (thread_id == g_thread_cnt.load()) {
                  std::cout << thread_id << std::endl;
                  g_thread_cnt.fetch_add(1);

                  if (g_thread_cnt.load() == thread_num) {
                    g_repeat_cnt.fetch_sub(1);
                    std::cout << ">>" << g_repeat_cnt << "<<" << std::endl;
                    g_thread_cnt.store(0);
                  }
                }
              }
            },
            i));
    g_thread_pool.push_back(std::move(thr));
  }

  for (auto &thr : g_thread_pool) thr->join();

  return 0;
}
```

### 类的默认函数有哪些？

默认会生成 4 个函数（3 个构造函数 + 1 个析构函数）

1. 默认构造函数
2. 拷贝构造函数
3. 移动构造函数
4. 析构函数

### 多态的实现

多态，一般是指 **继承 + 虚函数实现的多态**。

对于 **重载** 来说，实际上是 **编译期为函数生成符号表时的不同规则，只是一种语言特性**，也可以算作多态的一种，成为静态多态（编译期绑定，即在编译期就决定了调用哪个函数，根据参数列表来决定）

**动态多态，指的是子类继承父类的虚函数来实现，在运行期决定调用哪个函数；实现一般采用虚表指针 + 虚函数表**

### 析构函数一般写成虚函数的原因？构造函数呢？

为了防止内存泄漏。

一个基类的指针指向一个派生类对象，在使用完毕准备销毁时，如果基类的析构函数没有定义为虚函数，那么编译期根据指针指向的类型就会认为当前对象的类型是基类，**调用基类的析构函数，不会调用派生类的析构函数**，导致派生类的自身所占内存无法析构，内存泄漏。

如果基类的析构函数声明为虚函数，那么会先调用派生类的析构函数、再调用基类的析构函数；

构造函数一般不定义为虚函数

- 当我们要创建一个对象时，需要知道对象的完整信息，而虚函数调用只需要知道“部分”信息，即只需要知道函数接口，而不需要知道对象的具体类型，二者矛盾。
- 从编译器实现虚函数进行多态的方式来看，虚函数的调用是通过实例化之后对象的虚表指针来找对虚函数的地址进行调用，如果构造函数是虚的，那么虚表指针不存在，无法找到对应的虚函数表进行调用，违反了先实例化后调用的准则；

### 深拷贝和浅拷贝的区别？

当出现类的等号赋值时，会调用拷贝构造 / 拷贝赋值函数，在未显式定义拷贝构造函数的情况下，系统会调用默认的拷贝构造函数（浅拷贝），它能完成成员的一一赋值，因此 **当数据成员中不存在指针时，浅拷贝可行**

当数据成员中存在指针时，如果采用浅拷贝，则两个对象中的两个指针指向同一个地址，导致析构时野指针问题；必须采用深拷贝。

**深拷贝，会在堆内存中另外申请内存空间来存储数据，从而解决野指针问题**

### 什么情况下会调用拷⻉构造函数(三种情况)

1. **函数值传递**：一个对象以值传递的方式传入函数体,需要拷⻉构造函数创建一个临时对象压入到栈空间中
2. **函数值返回**：一个对象以值传递的方式从函数返回,需要执行拷⻉构造函数创建一个临时对象作为返回值
3. **值对象初始化**：一个对象需要通过另外一个对象进行初始化

### 为什么拷⻉构造函数必需时引用传递,不能是值传递?

为了防止递归调用，导致栈溢出，程序崩溃。

因为值传递会生成临时对象，调用拷贝构造函数，无限递归调用

### 构造函数析构函数可否抛出异常？

C++ 只会析构已完成的对象，对象只有在其析构函数执行完毕才算是完全析构。

在析构函数中发生异常，控制权转出析构函数之外，如果没有在当地进行捕捉，那么析构函数执行不完全，造成内存泄漏。

### 类如何实现只能静态分配？只能动态分配？

- 静态分配：把 `new` 和 `delete` 运算符私有化
- 动态分配：把**构造函数**、**析构函数** 设为**protected**，并通过子类来动态创建

### C++ 模板是什么？底层怎么实现的？

编译器并不是把函数模板处理成能够处理任意类的函数，而是将函数模板通过具体类型产生不同的函数；

编译器会对函数模板进行两次编译：

- 在模板定义的地方对模板代码进行编译
- 在模板实例化的地方，对模板参数替换后的代码进行编译

原因：**函数模板要被实例化后才能成为真正的函数，在使用函数模板的源文件中包含函数模板的头文件，如果该头文件只有声明，没有定义，那么编译器无法实例化该模板，最终导致链接错误**

### 请写出在 `main` 函数之前运行的函数

- 使用 GCC 扩展

  `__attribute__((constructor))` - 标记这个函数在 `main` 函数之前执行

  `__attribute__((destructor))` - 标记这个函数在程序结束之前执行（`main` 结束之后，或调用了 `exit` 函数后）

  ```cpp
  __attribute__((constructor)) void f() {
      std::cout << __LINE__ << ": " << __FUNCTION__ << std::endl;
  }
  ```

- 使用全局 static 变量的初始化，该过程在程序初始阶段，先于 `main` 函数执行

  ```cpp
  int before() {
      std::cout << __LINE__ << ": " << __FUNCTION__ << std::endl;
      return 0;
  }
  static int i = before();
  ```

- 利用 lambda 表达式

  ```cpp
  int a = []() {
      std::cout << __LINE__ << ": " << __FUNCTION__ << std::endl;
      return 0;
  } ();
  ```

### 分别写出 bool、int、float、指针类型的变量 a 与 “零” 的比较语句

```cpp
// bool
if(a) return true;
if (!a) return false;

// int
if (a == 0) return true;
if (a != 0) return false;

// float
const float EPSILON = 0.000001;
if (a <= EPSILION && a >= -EPSILON) return 0;

// pointer
if (a == NULL) return true;
if (a != NULL) return false;
```

### C++ 中的左值和右值

**C++ 11 通过引入右值引用来优化性能**

- **==通过移动语义来避免多余的拷贝问题==**
- **==通过移动语义将临时生成的左值的资源无代价地转移到另一个对象中去==**
- **==通过完美转发解决不能按照参数实际类型来转发的问题==**

**左值**

> 左值：可以取地址、具名、可以被赋值的表达式；准确来说，==左值是表达式（不一定是赋值表达式）后依然存在的持久对象；==
>
> `++i = 10;`

**右值**

> 右值：不能取地址、没有名字；==表达式结束后就不再存在的临时对象==；无法获取地址的对象：常量值、函数返回值、Lambda 表达式等
>
> 纯右值（Prue Rvalue）：纯粹的右值，指的是临时变量和不跟对象关联的字面量值，`10、true`；**==非引用返回的临时变量、运算表达式产生的临时变量、原始字面量、Lambda 表达式==**
>
> **==字面量除了*字符串*以外，均为纯右值==** <<-->> **==字符串字面量是一个左值，类型为 `const char` 数组==**
>
> 将亡值（eXpiring Value）：C++ 11 新增的跟右值引用相关的表达式，这样的表达式通常是要被移动的对象
>
> - 右值引用 `T&&` 的函数返回值
> - `std::move` 的返回值
> - 转换为 `T&&` 的类型转换函数的返回值
>
> **右值引用通常不能绑定到任何的左值；==想要绑定一个左值到右值引用，通常需要使用 `std::move` 将左值强制转换为右值==**
>
> NOTE
>
> - 通过右值引用的声明，右值“重获新生”，其生命周期与右值引用类型变量的生命周期一样长，只要该变量还活着，那么该右值临时变量将会一直活下去；
> - 右值引用独立于左值和右值，即==右值引用类型的变量可能是左值，也可能是右值==
> - `T&& t` 在发生自动类型推导时， 它是左值还是右值取决于它的初始化
>
> ``` cpp
> // "01234" 是左值，且类型为 const char [6]
> const char(&left)[6] = "01234";
> // const char(&&right)[6] = "01234"; // "01234" 是左值，不可被 右值引用
> 
> /* decltype(expr) 在 expr 为左值且非无括号包裹的 id 表达式与类成员表达式时，会返回左值引用 */
> static_assert(std::is_same<decltype("01234"), const char(&)[6]>::value, "");
> static_assert(std::is_same<decltype("01234"), decltype(left)>::value, "");
> ```

### C++ 中的 **move 语义**

move 语义，就是从右值中直接拿数据过来初始化或修改左值，而不需要重新构造左值后再析构右值

```cpp
class Person{
public:
    Person(Person&& rhs) {...}
    ...
};
```

### 下面的两段程序中，循环能否执行？为什么？

```cpp
// A: 
unsigned short i; unsigned short index = 0; for(i = 0; i <index-1; i++){printf(“a\n”); }
// B:
unsigned short i; unsigned long index = 0; for(i = 0; i <index-1; i++){printf(“b\n”); } 
```

A 不会执行，当执行到语句 `i<index-1` 时，由于类型不匹配，右边的`index`和`1`相减时会发生隐式类型转换 ，即`index`将被转换成有符号整型 ，转换之后的`index`还是`0`，因此程序片段A中的`index-1`的结果就是 `-1` ，此时判断`i<index-1`，即 `0<-1`，显然不成立，立即退出循环；

B 会执行，`index`是unsigned long型，当执行到语句`i<index-1` 时，由于类型不匹配，右边的index和1相减时也会发生由低精度类型向高精度方向的隐式类型转换 ，即1将被转换成无符号长整型 ，因此程序片段B中的`index-1`的过程用十六进制数表示实际上就是`0x00000-0x0001=0xffff`，此时再把左边的` i `隐式转换成无符号长整型之后判断 `i<index-1`，即 `0<0xffff`，显然成立

### 手写实现字符串函数：`strcat`, `strcpy`, `strncpy`, `memset`, `memcpy`

```c++
char* strcpy(char* dst, const char* src) {
    assert(dst != nullptr); // 检查输入参数
    assert(src != nullptr);
    char* cache = dst;  // 记录 dst 地址，便于返回
    while(*src != '\0') *(dst++) = *(src++);
    *dst = '\0'; // 手动添加末尾 '\0'
    return cache;
}

// 把 src 所指向的字符串追加到 dest 所指向的字符串的结尾
char* strcat(char* dest, const char* src) {
    assert(dst != nullptr); // 检查输入参数
    assert(src != nullptr);
    char* cache = dst;  // 记录 dst 地址，便于返回
    while(*dest != '\0') dest++;
    while(*src != '\0') *(dest++) = *(src++);
    *dest = '\0'; // 手动添加末尾 '\0'
    return cache;
}

// return s1 > s2 ? 1 : s1 < s2 ? -1 : 0;
int strcmp(const char* s1, const char* s2) {
    assert(s1 != nullptr);
    assert(s2 != nullptr);
    while(*s1 != '\0' && *s2 != '\0') {
        if (*s1 > *s2) return 1;
        else if (*s1 < *s2) return -1;
        else {
            ++s1; ++s2;
        }
    }
    
    if (*s1 > *s2) return 1;
    else if (*s1 < *s2) return -1;
    else return 0;
}

char* strstr(char* str1, char* str2) {
    assert(str1 != nullptr);
    assert(str2 != nullptr);
    if(*str2 == '\0') return nullptr; // str2 为空，直接返回 nullptr
    char* s = str1;
    while(*s != '\0') {
        char* s1 = s;
        char* s2 = str2;
        while(*s1 != '\0' && *s2 != '\0' && *s1 == *s2) s1++, s2++;
        if (*s2 == '\0') return str2;
        if (*s2 != '\0' && *s1 == '\0') return nullptr;
        s++;
    }
    return nullptr;
}

void* memcpy(void* dest, void* src, size_t len) {
    assert(dst != nullptr); // 检查输入参数
    assert(src != nullptr);
    void* ret = dest;
    for (size_t i = 0; i < len; ++i) {
        *reinterpret_cast<char*>(dest) = *reinterpret_cast<char*>(src);
        dest = reinterpret_cast<char*>(dest)++;
        src = reinterpret_cast<char*>(src)++;
    }
    return ret;
}

void* memset(void* src, char val, size_t len) {
    assert(ptr != nullptr);
    unsigned char* p = reinterpret_cast<unsigned char*>(ptr);
    for (size_t i = 0; i < len; ++i) 
        *(p + i) = val;
    return ptr;
}
```

## STL

### STL  介绍

STL 一共提供六大组件，包括 **容器**， **算法**， **迭代器**， **仿函数**， **适配器** 和 **配置器**。

容器通过配置器取得数据存储空间，算法通过迭代器存取容器内容，仿函数可以协助算法完成不同的策略变化，适配器可以应用于容器、仿函数、迭代器。

**容器**：各种数据结构，如 vector，list ，deque，set ，map 等，用来存放数据，采用类模板实现；

**算法**：各种常用的算法，如 sort（插入、快排、堆排），search（二分查找）等，采用函数模板实现；

**迭代器**：一种将 `operator*`, `operator->`, `operator++`, `operator--` 等指针相关操作赋予重载的类模板，所有 STL 容器都有自己的迭代器；

|                | 运算符重载                                               | 容器                       |
| -------------- | -------------------------------------------------------- | -------------------------- |
| 输入迭代器     | `*`, `!=`, `==`, `++`                                    | `istream_iterator`         |
| 输出迭代器     | `*`, `!=`, `==`, `++`                                    | `back_insert_iterator`     |
| 前向迭代器     | `*`, `!=`, `==`, `++`                                    | `forward_list`             |
| 双向迭代器     | `*`, `!=`, `==`, `++`, `--`                              | `set`, `map`, `list`       |
| 随机访问迭代器 | `*`, `!=`, `==`, `++`, `--`， `+`, `-`, `+=`, `-=`, `[]` | `vector`, `array`, `deque` |

 **包含关系：前向迭代器＞双向迭代器＞随机访问迭代器**

**仿函数**：一种重载了 `operator()`  的类或类模板；

**适配器**：一种用来修饰容器或仿函数或迭代器接口的东西；

**配置器**：负责空间配置与管理，一个实现了动态分配空间配置、空间管理、空间释放的类模板；

### vector 容器

#### `operator[]`

**`[]` 除了可以读取元素，还可以写入，因为返回的是==元素的引用`T&`==**

- `T& operator[] (size_t i) noexcept;`
- `const T& operator[] (size_t i) const noexcept;`
- `[]` 运算符在索引超出数组大小时，并不会直接报错（为了性能考虑）

#### `at()`

**`[]` 除了可以读取元素，还可以写入，因为返回的是==元素的引用`T&`==**

- `T& at(size_t i);`
- `const T& at(size_t i) const;`

- `at`会检测索引 `i`是否越界，如果他发现索引 `i >= a.size()`则会抛出异常` std::out_of_range `让程序提前终止
- `at` 需要额外检测下标是否越界，虽然更安全方便调试，但和` [] `相比有一定性能损失

#### `clear()`

可以清空该数组，也就相当于把长度设为零，变成空数组

- vector 之后把 `size()` 标记为 `0`，并调用其成员的析构函数
- **不会实际释放内存（`free`)**

#### vector 扩容，`resize`和`reserve`的区别？

**resize**

  可以改变 vector 的大小，即如果新的大小比旧的大小更大，那么会在末尾==添加默认构造函数创建的元素==，否则会==删除尾部元素==

  ==resize 到更大尺寸会导致 data 失效==

  扩容逻辑：`max(n, capacity * 2)`

**reserve**

  只是修改 vector 的 capacity ，不改变 size；

  reserve 只是预留空间，不会改变 vector 的大小，不会创建对象

- 若 n > capacity() ，则扩容。==**分配新存储，并将原空间元素拷贝到新空间内**==
- 若 n < capacity() ，缩容？，NO，==**该方法不做任何事**==

#### `data()` 获取首地址指针

返回指向数组中首个元素的指针，也就是等价于 `&a[0]`，`int* data() noexcept;` / `const int* data() const noexcept;`

**`data()` 指针是对 vector 的一种引用，实际对象生命周期仍由 vector 类本身管理**

#### `shrink_to_fit（）` 释放多余容量

`shrink_to_fit` 释放掉多余的容量，只保留刚好为 size() 大小的容量；**会重新分配一段更小内存，他同样是会把元素移动到新内存中的，因此迭代器和指针也会失效**

### `set` 容器

- set 的迭代器对象也重载了 `++` 为红黑树的遍历（中序遍历，即总是按照 Key 从小到大的顺序）

- `std::next` 等价于 `+`：自动判断迭代器是否支持 + 运算，如果不支持，会改为比较低效的调用 n 次 ++

  `std::advance` 等价于 `+=`：就地自增作为引用传入的迭代器，他同样会判断是否支持 += 来决定要采用哪一种实现

  `advance` 就地修改迭代器，没有返回值；`next` 修改迭代器后返回，不会改变原迭代器

### `map` 容器

#### `[]` 和 `at`

二者都是返回引用；

当键值不存在时，**`[]` 默认创建，`at()` 则抛出异常**

> **C++ 读取 map 中的元素，推荐使用 `at()` ，这与大多数语言中的`[]`行为一致（抛异常）**

当写入 map 元素时，`[]` 创建在先，`=` 赋值在后；`at()` 报错在先，`=` 赋值在后（写入失败）；

> **C++ 写入 map 中的元素时，推荐使用 `[]`，这与大多数语言的 `[]` 行为一致**

#### 元素类型

- `map<K, V>::value_type` -->> `pair<const K, V>`
- `map<K, V>::key_value` -->> `K`
- `map<K, V>::mapped_type` -->> `V`

### 说一说 STL 迭代器删除元素

对于序列化容器 vector, deque 来说，使用 `erase(iter)` 后，后面的每个元素的迭代器都会失效，到后面的每个元素都会向前移动一个位置，返回下一个有效的迭代器；

对于关联容器 map, set 来说，使用 `erase(iter)`后，当前元素的迭代器失效，由于采用红黑树，删除当前元素，不会影响到下一个元素的迭代器，所以在调用 `erase` 之前，记录下一个元素的迭代器即可；

对于 list 来说，采用了不连续分配的内存，并且使用了 `erase(iter)` 会返回下一个有效的迭代器；

### STL中的迭代器 iter ，`++iter` 和 `iter++` 有什么区别？

**函数原型**

  前置 ++，即 `++iter`，`T& opreator++();`

  后置 ++，即 `iter++`，`T operator++(int);`

**返回值**

  `++iter`，返回一个引用，不会产生临时对象，效率较高；

  `iter++`，返回临时对象，会产生临时对象，导致效率降低；

### STL 中的 map容器，`[]`和 `find`函数有什么区别？

**`[]`**

  将 `key` 作为下标去执行查找，并返回对应的值；

  如果没有找到，则会将 `key` 和 `val` 插入到 map 中

**`find`函数**

  使用 `key` 执行查找，如果找到则返回相应的迭代器，否则返回尾迭代器；

## GDB

### **GDB 启动调试**

```bash
gdb [--args] main [argv]

# 方法一
gdb --args main 5

# 方法二
gdb main
(gdb) run 5

# 方法三
gdb main
(gdb) set args 5
(gdb) run
```

### **断点 `break`**

- 对 `Foo` 命名空间中的 `foo` 函数：`(gdb) b Foo::foo`
- 匿名空间中的`bar` 函数：`(gdb) b (anonymous namespace)::bar`
- 在程序地址上打断点：`b *address`
- 文件行号：`b file:line_num`

- 保存、加载断点：`save breakpoints file-name-to-save`, `souce file-name-to-save`
- **临时断点**：`tbreak` , `tb`
- **条件断点**：`break ... if cond`,eg`b 10 if i == 101`
- 查看断点信息：`info break` / `info breakpoint`
- 删除断点：`clear location`  / `delete`，参数 location 通常为某一行代码的行号或者某个具体的函数名。当 location 参数为某个函数的函数名时，表示删除位于该函数入口处的所有断点

### 函数

- 列出函数的名字(列出可执行文件的所有函数名称)：`info functions`

- 跳入、跳出函数：

  `step` (`s`)：跳入函数

  `finsh` ：跳出函数，函数完整执行完后返回

  `return`：**直接返回，会跳过当前函数后面的语句直接返回**，返回值可以自定义，紧跟在`return`命令后面即可

- 直接执行函数：`call func()` / `print func()`

- 查看函数堆栈帧信息：`i frame`

### 监视变量 `watch`

- `watch`命令用于监视变量是否被改变（**本质是硬件断点**）

  **只有当被监控变量（表达式）的值发生改变，程序才会停止运行**

  `watch var_name`

- 监视变量读操作 `rwatch`：只要程序中出现**读取**目标变量（表达式）的值的操作，程序就会停止运行

- 监视变量写操作 `awatch`：只要程序中出现**读取**目标变量（表达式）的值或者**修改**值的操作，程序就会停止运行

### TUI

- `tui enable` / `tui disable`

****

### 如何使用 gdb 排查多线程中哪个函数出现了死锁？

**==查看程序的线程信息、堆栈信息和锁信息，并确定死锁出现的位置==**

1. 暂停程序：如果程序出现了死锁，可以使用`ctrl + c`命令暂停程序的执行
2. 查看线程信息：`info threads`
3. 切换线程：`thread threadID`
4. 查看堆栈信息：`bt`

## Valgrind

Valgrind 是一个用于检测和调试内存错误的工 具，它的主要功能是在程序运行时进行内存分配和释放的跟踪，以及检测内存泄漏、越界访问和其他常见的内存错误。Valgrind 并不直接影响代码的优化级别，因为它是在程序运行时进行检测和分析的。

**==valgrind 默认使用 memcheck 去检查内存问题==**：`valgrind --tool=memcheck args-valgrind program args-program`

==编译器的优化级别可能会影响 Valgrind 的工作效果==：当编译器对代码进行优化时，它可能会进行一些代码转换，例如删除未使用的变量、内联函数或循环展开。这些优化操作可能会导致 Valgrind 的检测结果不准确或不完整。

**==在使用 Valgrind 进行调试时，建议使用较低的编译优化级别，例如 `-O0`，以便生成更接近原始源代码的可执行文件。这样做可以提高 Valgrind 的检测能力，因为它可以更准确地跟踪和分析代码的执行==**

> **==在调试和分析阶段使用较低的优化级别进行 Valgrind 检测，然后在发布和生产阶段使用更高的优化级别以获得更好的性能可能是一个常见的做法==**

```bash
valgrind --log-file=valgrind.log \
   --tool=memcheck \
   --leak-check=full \
   --show-leak-kinds=all \
   ./your_app arg1 arg2
```

- `--log-file` 报告文件名。如果没有指定，输出到stderr
- `--tool=memcheck` 指定Valgrind使用的工具。Valgrind是一个工具集，包括Memcheck、Cachegrind、Callgrind等多个工具。memcheck是缺省项
- `--leak-check` 指定如何报告内存泄漏，可选值：`no` 不报告；`summary` 显示简要信息，有多少个内存泄漏，`summary`是缺省值；`yes` 和 `full` 显示每个泄漏的内存在哪里分配；
- `show-leak-kinds` 指定显示内存泄漏的类型的组合

**可检测问题**

- 未释放内存的使用
- 对释放后内存的读/写
- 对已分配内存块尾部的读/写
- 内存泄露
- 不匹配的使用 `malloc`/`new`/`new[]` 和 `free`/`delete`/`delete[]`
- 重复释放内存

## GIT

略

## 十大排序算法

|  排序算法  | 平均时间复杂度  |    最好情况     |    最坏情况     | 空间复杂度  |         排序方式         | 稳定性 |
| :--------: | :-------------: | :-------------: | :-------------: | :---------: | :----------------------: | :----: |
| *冒泡排序* |    $O(n^2)$     |     $O(n)$      |    $O(n^2)$     |   $O(1)$    | In-place（**原地排序**） |  稳定  |
| *选择排序* |    $O(n^2)$     |    $O(n^2)$     |    $O(n^2)$     |   $O(1)$    |         In-place         | 不稳定 |
| *插入排序* |    $O(n^2)$     |     $O(n)$      |    $O(n^2)$     |   $O(1)$    |         In-place         |  稳定  |
|  希尔排序  |   $O(n logn)$   |  $O(n log^2n)$  |  $O(n log^2n)$  |   $O(1)$    |         In-place         | 不稳定 |
| *归并排序* |   $O(n logn)$   |   $O(n logn)$   |   $O(n logn)$   |   $O(n)$    |        Out-place         |  稳定  |
| *快速排序* |   $O(n logn)$   |   $O(n logn)$   |    $O(n^2)$     | $O(n logn)$ |         In-place         | 不稳定 |
|  *堆排序*  |   $O(n logn)$   |   $O(n logn)$   |   $O(n logn)$   |   $O(1)$    |         In-place         | 不稳定 |
|  计数排序  |   $O(n + k)$    |   $O(n + k)$    |   $O(n + k)$    |   $O(k)$    |        Out-place         |  稳定  |
|   桶排序   |   $O(n + k)$    |   $O(n + k)$    |    $O(n^2)$     | $O(n + k)$  |        Out-place         |  稳定  |
|  基数排序  | $O(n \times k)$ | $O(n \times k)$ | $O(n \times k)$ | $O(n + k)$  |        Out-place         |  稳定  |

### 冒泡排序

**算法描述**

1. 比较相邻的元素：如果第一个比第二个大，两者交换
2. 对每一个对相邻元素做同样的工作，从开始第一对到结尾的最后一对，直至在最后的元素应该是最大的数
3. 针对所有元素重复以上步骤，除了最后一个
4. 重复步骤 1 ～ 3 ，直到排序完成

**算法实现**

```cpp
void bubbleSort(std::vector<int>& nums) {
    if (nums.empty()) return;

    for (int i = 0; i < nums.size() - 1; ++i) {
        for (int j = 0; j < nums.size() - i - 1; ++j) {
            if(nums[j] > nums[j + 1]) 
                std::swap(nums[j + 1], nums[j]);
        }
    } 
}
```

**时间复杂度、空间复杂度、排序方式、是否稳定**

- 时间复杂度：$O(n^2)$，无论怎么优化，元素交换的次数是一个固定值，是原始数据的逆序度
- 空间复杂度：$O(1)$，不需要借助额外的空间
- 排序方式：In-place
- 是否稳定：稳定

### 插入排序

**算法描述：**

  分为已排序区间和未排序区间，初始已排序区间只有一个元素（即数组中第一个元素），遍历未排序的每一个元素，在已排序数组中找到合适位置插入并保证数据一直有序；

**算法实现**

```cpp
void insertSort(std::vector<int>& nums) {
    if (nums.empty()) return;

    for (int i = 0; i < nums.size(); ++i) {  // 已排序数组
        for (int j = i; j > 0; --j)  // 从后向前
            if (nums[j] < nums[j - 1])
                std::swap(nums[j], nums[j - 1]);
    }
}
```

**时间复杂度、空间复杂度、排序方式、是否稳定**

- 时间复杂度：$O(n^2)$，无论怎么优化，元素交换的次数是一个固定值，是原始数据的逆序度
- 空间复杂度：$O(1)$，不需要借助额外的空间
- 排序方式：In-place
- 是否稳定：稳定

### 选择排序

**算法描述**

  分为已排序区间和未排序区间；每次会从未排序区间中找到最小的元素，将其放入到已排序区间的末尾；

**算法实现**

```cpp
void selectSort(std::vector<int>& nums) {
    if (nums.empty()) return;

    size_t idx = 0;
    // The sorted [0, 1, ..., i-1]
    // i is the position to be put
    for (int i = 0; i < nums.size(); ++i) {
        idx = i;
        // Find the index of min element.
        for (int j = i; j < nums.size(); ++j) {
            if (nums[j] < nums[idx])
                idx = j;
        }
        // Put min element to the end [0, 1, ..., i]
        std::swap(nums[idx], nums[i]);
    }
}
```

**时间复杂度、空间复杂度、排序方式、是否稳定**

- 时间复杂度：$O(n^2)$
- 空间复杂度：$O(1)$，不需要借助额外的空间
- 排序方式：In-place
- 是否稳定：不稳定，因为每次都要找剩余未排序元素中的最小值，并和前面的元素交换位置，破坏了稳定性

### 快速排序

**算法描述**

  先找到一个轴（pivot）元素，在原来的元素里根据这个轴元素划分，比这个轴小的元素排前面，比这个轴元素大的元素排后面；两部分数据依次递归排序，直至最终有序。

**算法实现**

```cpp
void quickSort(std::vector<int>& nums) {
    if (nums.empty()) return;
    
    std::function<void(int, int)> f = [&] (int l, int r) {
        if (l + 1 >= r) return;
        
        int first = l, last = r - 1, pivot = nums[first];
        while(first < last) {
            // From right to left
            while(first < last && nums[last] >= pivot) --last;
            nums[first] = nums[last]; // 因为 nums[first] 被保存在了 pivot 中，可以覆盖
            // From left to right
            while(first < last && nums[first] <= pivot) ++first;
            nums[last] = nums[first]; // nums[last] 被移动到 nums[first]，可覆盖
        }
        // 更新 pivot
        nums[first] = pivot;
        // 以 first 作为中间值划分区间
        f(l, first);
        f(first + 1, r);
    };

    f(0, nums.size());
}
```

### 归并排序

**算法描述**

  归并排序，是一个稳定的排序算法，时间复杂度为 $O(n longn)$ ，但不是原地排序，需要借助额外空间，空间复杂度 $O(n)$；

### 堆排序

**算法描述**

  利用堆这种数据结构所设计的一种排序算法。**堆是一个近似完全二叉树的结构，并同时满足堆的性质：子节点的值总是小于（或大于）它的父节点。**堆排序可以利用上一次的排序结果，不像其他一般的排序算法一样，每次都要进行 n - 1 次的比较，复杂度为 $O(n logn)$。

建堆：将数组原地建成一个堆，不借助额外空间，采用从上往下的堆化（对于完全二叉树来说，下标是 $n/2 + 1$ 到 n 的节点都是子节点，不需要堆化）

排序：“删除堆顶元素”：当堆顶元素移除之后，把下标为 n 的元素放到堆顶，然后在通过堆化的方法，将剩下的 n - 1 个元素重新构建成堆，堆化完成以后，再取堆顶的元素并放到 n - 1 的位置，直至最后堆中只剩下标 1 的一个元素

**优点**

- $O(n logn)$
- 原地排序

缺点：

- 相比于快速排序，堆排序数据访问的方式没有快速排序友好
- 数据交换的次数要多于快速排序

**算法实现**

```

```

### 桶排序

**算法描述**

  将数组分到有限数量的桶里，每个桶在个别排序（可以使用别的排序算法，也可以使用递归方式继续使用桶排序进行）

**算法实现**

```cpp

```

### 计数排序

**算法描述**

  用待排序的数作为计数数组的下标，统计每个数字的个数，然后依次输出即可得到有序序列。

**特点**

- 时间复杂度：$O(n)$，前提是，待排序的数要满足一定的范围内的整数，而且计数排序需要较多的辅助空间；

- 只用用在数据范围不太大的情景中，如果数据范围 k 要比排序的数据 n 大很多，就不适合使用计数排序
- 计数排序只能发给非负整数排序；如果要排序的类型是其他类型的话，需要将其在不改变相对大小的情况下进行变换，使其满足前提条件（**一定范围内的非负整数**）

**算法实现**

```cpp
```

### 基数排序

**算法描述**

  基数排序对要排序的数据有要求，需要可以分割出独立的 ”位“ 来比较，而且位之间有递进的关系（如果 a 数据的高位比 b 数据大，那么剩下的低位就不用比较了）；另外，每一位的数据范围不能太大，要可以用线性排序算法来排序，否则，基数排序的时间复杂度就无法做到 $O(n)$.

**基数排序，相当于通过循环进行了多次桶排序**

**算法实现**

```cpp
```

### 希尔排序

**算法描述**

  通过将比较的全部元素分为几个区域来提升**插入排序**的性能；这样可以让每一个元素可以一次性地朝最终位置前进一大步，然后该算法再取越来越小的步长进行排序，算法的最后一步就是普通的**插入排序**；直到间隔减小到1，此时数组已经完全排序。

**算法实现**

```cpp
void shellSort(std::vector<int>& nums) {
    if (nums.empty()) return;

    int n = nums.size();
    // gap is initialized to n/2, then is divided by 2 until it reaches 1
    for (int gap = n/2; gap > 0; gap /= 2) {
        // Do a gapped insertion sort for this gap size.
        // The first gap elements a[0..gap-1] are already in gapped order
        // keep adding one more element until the entire array is gap sorted
        for (int i = gap; i < n; i += 1) {
            // add a[i] to the elements that have been gap sorted
            // save a[i] in temp and make a hole at position i
            int temp = nums[i];
            // shift earlier gap-sorted elements up until the correct location for a[i] is found
            int j;           
            for (j = i; j >= gap && nums[j - gap] > temp; j -= gap)
                nums[j] = nums[j - gap];
            // put temp (the original a[i]) in its correct location
            nums[j] = temp;
        }
    }
}
```

## TCP / UDP 高频面试题

![](https://raw.githubusercontent.com/lutianen/PicBed/main/202307121824510.jpg)

### IP 首部格式

<img src="https://raw.githubusercontent.com/lutianen/PicBed/main/202308232230758.webp" alt="IP 首部格式"  />

### TCP 和 UDP 的特点与区别

<img src="https://raw.githubusercontent.com/lutianen/PicBed/master/12-TCP%E7%BC%96%E7%A8%8B%E6%A8%A1%E5%9E%8B.webp" alt="12-TCP编程模型" style="zoom: 50%;" />                    <img src="https://raw.githubusercontent.com/lutianen/PicBed/master/13-UDP%E7%BC%96%E7%A8%8B%E6%A8%A1%E5%9E%8B.webp" alt="13-UDP编程模型" style="zoom: 67%;" />

**用户数据报协议 UDP (User Datagram Protocol)**

  无连接的，尽最大可能交付，没有拥塞控制，面向报文（对于应用程序传下来的报文不合并也不拆分，只是添加 UDP 首部），支持一对一、一对多、多对一、多对多的交互通信；

  UDP通信时也可以用connect的，特定情况下效率会更高：UDP 的 `connect` 不是建立连接，而是绑定 IP 和 port，也就是建立（UDP 套接字 ==$目的地址 + 端口$==）之间的映射关系

**传输控制协议 TCP（Transmission Control Protocol）**

  面向连接的、提供可靠交付，有流量控制、拥塞控制，提供全双工通信，面向字节流（把应用层传下来的报文看作字节流，把字节流组织成大小不等的数据块），每一条 TCP 连接只能是点对点（一对一）的

### TCP、UDP 首部格式

<img src="https://raw.githubusercontent.com/lutianen/PicBed/main/202307132352724.png" style="zoom: 67%;" />

- **序列号**：在建立连接时由计算机生成的随机数作为其初始值，通过 SYN 包传给接收端主机，每发送一次数据，就「累加」一次该「数据字节数」的大小。**用来解决网络包乱序问题。**
- **确认应答号**：指下一次「期望」收到的数据的序列号，发送端收到这个确认应答以后可以认为在这个序号以前的数据都已经被正常接收。**用来解决丢包的问题。**
- **控制位：**
  - URG：标志紧急指针是否有效；
  - *ACK*：该位为 `1` 时，「确认应答」的字段变为有效，TCP 规定除了最初建立连接时的 `SYN` 包之外该位必须设置为 `1` 。
  - PSH：提示接收端立即从缓冲独走数据；
  - *RST*：该位为 `1` 时，表示 TCP 连接中出现异常必须强制断开连接。
  - *SYN*：该位为 `1` 时，表示希望建立连接，并在其「序列号」的字段进行序列号初始值的设定。
  - *FIN*：该位为 `1` 时，表示今后不会再有数据发送，希望断开连接。当通信结束希望断开连接时，通信双方的主机之间就可以相互交换 `FIN` 位为 1 的 TCP 段。

- **窗口大小**：接受窗口，用于告知对端（发送方）本端的缓冲还能接收多少字节数据，用于解决流量控制；
- **校验和**：接收端用 CRC 校验整个报文段有无损坏

---

<img src="https://raw.githubusercontent.com/lutianen/PicBed/main/202307132353917.png" alt="UDP 头部格式" style="zoom:67%;" />

**UDP 首部字段只有 8 个字节，包括源端口、目的端口、长度、校验和。**12 字节的伪首部为了计算检验和临时添加的。

*12 字节的伪首部，并不是 UDP 数据报的一部分，在数据报文中并不存在，而是在计算和验证 UDP校验和时使用。 包括 源 IP 地址、目的IP地址、协议号、UDP 长度，每一字段都是 4 字节*

### TCP 三次握手和四次挥手

**TCP 连接，由一个四元组 $\{源 IP，源 port, 目的 IP，目的 port\}$ 构成；**

**一个完整的 TCP 连接是双向和对称的，数据可以在两个方向上平等的流动，给上层提供一个全双工服务。**

**一旦建立了一个连接，这个连接的一个方向上的每个 TCP 报文段都包含了相反方向上的报文段的一个 ACK.**

**==ACK 确认报文不会发生重传，当 ACK 丢失，就由对方重传对应的报文==**

#### TCP三次握手

<img src="https://raw.githubusercontent.com/lutianen/PicBed/main/202307132353249.png" alt="TCP三次握手.drawio" style="zoom: 33%;" />

**第三次握手是可以携带数据的，前两次握手不可以携带数据**

##### 为什么需要握手三次？

1. **握手只需要确认双方通信的初始化序列号、确认号，保证通信不会乱序，防止失效的历史连接请求达到服务器，让服务器错误打开连接。**

2. 换个角度，客户端和服务端通信前要进行连接，三次握手的作用就是确保双方的接收、发送的能力是正常的。

    第一次握手：客户端发送网络包，服务端收到。

      服务端可以得出结论：**客户端的发送能力、服务端的接收能力正常**

    > 如果发生丢包，则客户端重发 SYN 报文，一般重传 5 或 6 次（`tcp_syn_retries` 控制），重传时间逐次翻倍，首次重传根据 OS 不同而不同（1 s, 3 s）；

    第二次握手：服务端发送网络包，客户端收到。

      客户端可以得出结论：**服务端的发送能力、接受能力正常，客户端发送能力、接收能力正常**

    > 如果发送丢包，客户端因未收到确认报文而触发超时重传机制（重传 SYN 报文），服务端因未收到 ACK 确认报文而触发超时重传机制（重传 SYN-ACK 报文），重传次数由 `tcp_synack_retries` 参数控制，一般为 5 或 6；

    第三次握手：客户端发包，服务端收到。

      服务端可以得出结论：**服务端的发送能力、接受能力正常，客户端发送能力、接收能力正常**

    > 如果发生丢包，服务端因未收到 SYN-ACK 报文的确认报文而触发超时重传（重传 SYN-ACK 报文），重传次数由 `tcp_synack_retries` 参数控制，一般为 5 或 6；

      **然而，如果没有第三次握手，服务端就不知道客户端的接收能力以及服务端的发送能力是否正常；同样，完全没有必要进行第四次握手，因为双方的发送、接收能力都已经确保正常**

#### TCP 四次挥手

<img src="https://raw.githubusercontent.com/lutianen/PicBed/main/202307132353556.png" alt="TCP 挥手" style="zoom: 50%;" />

**每个方向都需要一个 FIN 报文和对应的 ACK 报文**，因此成为四次挥手

**只有主动关闭连接的一方，才有 ==TIME_WAIT== 状态（持续时长 $2MSL$）**

##### 为什么挥手需要四次？

1. TCP 连接是双向传输的对等模式，即双方都可以同时发送或接收数据；

    当有一方要关闭连接时，会发送指令告知对方，对方会回复一个对应的 ACK，此时一个方向的连接关闭，但是另一个方向的连接仍然可以继续传输数据，因此需要双方分别发送 FIN 报文，且都会接收到对应的 ACK 报文。

    > ==第一次挥手报文丢失：==
    >
    >   主动关闭方调用 `close` 或 `shutdown` 函数后，回向被动关闭方发送 FIN 报文，并进入 ==FIN_WAIT_1== 状态，正常情况收到 ACK 后进入==FIN_WAIT_2== 状态；
    >
    >   如果 FIN 报文丢失，主动关闭方因未收到 ACK 而触发超时重传机制（重发 FIN 报文），重传次数由 `tcp_orphan_retries` 参数控制；当客户端重传次数超过 `tcp_orphan_retries` 后，就不再重发 FIN 报文，而是等待一段时间（时间为上一次超时时间的2倍），如果还是没能收到 ACK，那么直接进入 ==CLOSE== 状态。
    >
    >==第二次挥手报文丢失：==
    >
    >   当被动关闭方收到主动关闭方的第一次挥手报文后，就会回复 ACK 报文，并进入 ==CLOSE_WAIT== 状态；而由于 ACK 报文丢失当不重传，主动关闭方的行为和第一次挥手报文丢失表现一致；
    >
    >   **当主动关闭方收到 ACK 报文后，就会进入 ==FIN_WAIT_2== 状态，等待被动关闭方发送 FIN 报文**；
    >
    >   对于 `close` 函数关闭的连接，由于无法在发送和接收数据，导致 ==FIN_WAIT_2== 状态不可以持续太久，由 `tcp_fin_timeout` 参数控制，默认 60 秒，超时后主动关闭方的连接就会直接关闭。
    >
    >   对于 `shutdown` 函数关闭的连接，由于指定了只关闭发送方向，而接收方向并没有关闭（主动关闭方还可以接收数据），主动关闭方会一直处于 ==FIN_WAIT_2== 状态，直至收到第三次挥手报文。
    >
    > ==第三次挥手报文丢失：==
    >
    >  当被动关闭方收到主动关闭方的 FIN 报文后，内核会自动回复 ACK 报文，同时进入 ==CLOSE_WAIT== 状态（等待应用进程调用 `close` / `shutdown` 函数关闭连接，必须由进程主动调用来触发 FIN 报文发送）；
    >
    >   如果第三次挥手报文丢失，被动关闭方因未收到 ACK 报文而出发超时重传机制，重传机制仍由 `tcp_orphan_retries` 参数控制，即行为和第一次报文丢失时主动关闭方的行为一致；
    >
    > ==第四次挥手报文丢失：==
    >
    >   当主动关闭方收到被动关闭方发送的 FIN 报文后，就会回复 ACK 报文并进入 ==TIME_WAIT== 状态（持续 $2MSL$ 后进入 ==CLOSE== 状态）；
    >
    >  如果第四次挥手报文（ACK 报文）丢失，被动关闭方因未收到 ACK 报文而触发超时重传机制，行为表现和第三次挥手丢失一致；

2. **在特定的情况下，四次挥手可以变成三次挥手**

##### TIME_WAIT

==MSL==（Maximum Segment Lifetime），报文最大生存时间，是任何网络报文在网络上生存的最长时间，超过这个时间的报文将被丢弃。

**==MSL== 的单位是时间（一般为 30 s），==TTL== 的单位是路由跳数（一般为 64），==MSL== 应该要大于等于 ==TTL== 消耗为 0 的时间，以确保报文的自然消亡。**

主动关闭方在收到被动关闭方的 FIN 报文后进入 ==TIME_WAIT== 状态，此时不是直接进入 ==CLOSED== 状态，而是等待一个时间计时器设置的时间 $2MSL$ ，理由：

- 确保最有一个确认报文能够到达：**至少允许报文丢失一次**
- 为了让本连接持续时间内所产生的所有报文都从网络中消失，使得下一个新的连接不会出现旧的连接请求报文；

 **为什么不是 4 MSL 或 8 MSL？**

  可以想象一个丢包率达到百分之一的糟糕网络，连续两次丢包的概率只有万分之一，这个概率实在是太小了，忽略它比解决它更具性价比。

### 请解释一下 RTO 、RTT 和 超时重传？

**超时重传**

  发送端发送报文后若干长时间未收到确认的报文，则需要重传该报文。可能的情况：

- 发送的数据没有到达对端；
- 对端收到数据，但 ACK 报文在回传的过程中丢失；
- 接收端拒绝或丢弃数据；

**RTO**

  Retransmission Timeout，**从上一次发送数据，因为长期没有收到 ACK 响应，到下一次重发的时间间隔**，就是重传间隔。

  **==通常情况下，每次重传 RTO 是前一次的两倍，计量单位是 RTT，1 RTT / 2RTT / 4RTT ...==，==重传次数到达上限后停止重传==**

  **首次计算 RTO，其中 $R1$ 为第一次测量的 RTT：**
$$
\begin{aligned}
SRTT &= R1 \\
DevRTT &= R1 / 2 \\
RTO &= \mu * SRTT + \p * DevRTT \\
 &= \mu * R1 + \p * (R1 / 2)
\end{aligned}
$$
  **后续计算 RTO，其中 $R2$ 为最新测量的 RTT**：
$$
\begin{aligned}
SRTT &= SRTT + \alpha(RTT - SRTT) \\
  &= R1 + \alpha*(R2 - R1) \\
DevRTT &= (1 - \beta)*DevRTT + \beta*|RTT - SRTT| \\
       &= (1 - \beta) * (R1/2) + \beta * |R2 - R1| \\
RTO &= \mu*SRTT + \p * DevRTT
\end{aligned}
$$
  其中 $ SRTT $ 是计算平滑的 $RTT$，$ DevRTT$ 是计算平滑的 RTT 与最新 RTT 的差距；

  **在 Linux 中，$\alpha = 0.125, \beta = 0.25, \mu = 4$**.

**RTT**

  Round-Trip Time，**数据从发送到接收到对方响应之间的时间间隔，即一个数据报在网络中的一个往返用时，具体时长不稳定**

### 如何区分流量控制和拥塞控制？

流量控制属于通信双方协商  $<-->$  拥塞控制设计通信链路全局

流量控制需要通信双方各自维护一个发送窗口、接收窗口，任意一方的接收窗口大小自身决定，发送窗口大小由对方响应的 TCP 报文首部中的窗口值决定 $<-->$ 拥塞窗口的大小变化由试探性发送一定数量的探查网络状况后自适应调整

**$实际发送窗口 = min \{流控发送窗口, 拥塞窗口\}$**

### TCP 流量控制

==流量控制是为了控制发送方发送速率，保证接收方来得及接收==

接收方发送的确认报文中的==窗口字段==可以用来控制发送方的发送窗口大小，从而影响发送方的发送速率。

将窗口字段设置为 0 ，则发送方不能发送数据，称为**零窗口**；

解决方法：

  当发送端收到**零窗口**的通知后，它将开始**定期发送窗口探测数据段**。

  这个数据段通常**只包含一个字节的数据，并且它的序列号是上一次成功发送的数据的序列号**，因此**接收端不会实际接收这个数据**。**接收端在收到窗口探测数据段后，将返回一个确认，该确认包含当前的窗口大小**；接收端可以定期检查接收端的窗口大小。

  **窗口探测的次数一般为 3 次，每次大约 30-60 秒（不同的实现可能会不一样）。如果 3 次过后接收窗口还是 0 的话，有的 TCP 实现就会发 `RST` 报文来中断连接。**

### TCP 拥塞控制

目的：**避免 ==发送方== 的数据填满整个网络**

控制手段：**==拥塞窗口== cwnd， 发送方维护的一个状态变量，会根据网络的拥塞程度动态变换**

<img src="https://raw.githubusercontent.com/lutianen/PicBed/main/202307131730496.png" style="zoom:;" /> <img src="https://raw.githubusercontent.com/lutianen/PicBed/main/202307132353137.png" alt="拥塞发生-快速重传.drawio" style="zoom: 67%;" />

**慢启动**： 拥塞窗口大小指数增加

**拥塞避免**：拥塞窗口大小线性增加

**快重传**：在发送方，如果收到三个重复确认，那么可以知道下一个报文段丢失，此时执行快重传，立即重传下一个报文段。例如收到三个 M2，则 M3 丢失，立即重传 M3。

**快恢复**: 执行快恢复，令 $ssthresh = cwnd / 2 ，cwnd = ssthresh$，注意到此时直接进入拥塞避免

#### 常见的控制算法

##### TCP Reno 算法

**TCP Reno 算法通过超时重传（RTO）和重复确认（dupack）来判断丢包，然后采用慢启动、拥塞避免、快重传和快恢复四个阶段来调整用塞窗口的大小。**

##### TCP Vegas 算法

**TCP Vegas 算法不是通过丢包来判断拥塞，而是通过数据包的延迟来判断。**

**主要利用RTT（round trip time）来调整拥塞窗口的大小，能够在拥塞即将发生时就避免拥塞，而不是等到拥塞已经发生后再减小发送速率。**

##### TCP New Reno 算法

**New Reno 算法是对 Reno 算法中快速恢复阶段的重传进行改善的一种改进算法。**

Reno 算法的问题：如果一次拥塞中出现多个丢包，Reno 算法会误以为发生了多次拥塞，从而重复减小拥塞窗口导致发送速率下降，即一直重复这个过程，导致拥塞窗口一直减小。

解决方法：**New Reno 算法中的快速恢复中，一旦出现三次重复确认，会记下出现重复确认时未确认的数据包的最大序列号，然后重发重复确认的数据包；如果有多个数据包丢失，则继续重发丢失的数据包，直到最大序列号的数据包被确认才退出快速恢复阶段。**

##### TCP Westwood 和 Westwood+ 算法

**Westwood 算法改良自 New Reno 算法，不同于以往其他拥塞控制算法使用丢失来测量，该算法通过对确认包测量来确定一个 ”合适的发送速度” ，并以此调整拥塞窗口和慢启动阈值。**

**Westwood+ 算法使用更长的带宽估计间隔和优化的滤波器来修正 Wesstwood 对 ACK 压缩场景对带宽估计过高的问题（例如，路由器经常缓存数据使得 RTT 延长）。**

##### TCP PRR 算法

**PRR（Proportional Rate Reduction）算法，旨在恢复期间提高发送数据的准确性。**

该算法确保恢复后的拥塞窗口大小尽可能接近慢启动阈值。

##### TCP BIC 算法

**BIC（Binary Increase Congestion control）算法旨在优化高速高延迟网络（即根据 RFC 1072 所定义的 “长肥网络” （long fat network, LFN））的拥塞控制，采用二分搜索算法尝试找到能长时间保持拥塞窗口最大值的值。**

##### TCP CUBIC 算法

**CUBIC 是比 BIC 更温和和系统化的分支版本，采用三次函数代替二分算法作为其拥塞窗口算法，并且使用函数拐点作为拥塞窗口的设置值**

**CUBIC 中最关键的点在于它的窗口增长函数仅仅取决于连续的两次拥塞时间的时间间隔值，从而窗口增长完全独立于网络的时延 RTT。**

##### TCP BRR 算法

BRR（Bottlenck Bandwidth and Round-trip propagation time）算法：以往大部分拥塞算法是基于丢包来作为降低传输速率的信号，而 BRR 则基于模型主动探测。该算法**使用网络最近出站数据分组当时的最大带宽和往返时间来建立网络的显式模型**。

**该算法认为随着网络接口控制器逐渐进入千兆速度时，分组丢失不应该被认为是识别拥塞的主要决定因素，所以基于模型的拥塞控制算法能有更高的吞吐量和更低的延迟，可以用 BRR 来替代其他流行的拥塞算法**，比如 CUBIC.

特点：

- 解决了两个问题：
    1. 在有一定丢包率的网络链路上充分利用带宽，非常适合高延迟、高带宽的网络链路
    2. 降低网络链路上的 buffer 占用率，从而降低延迟，非常适合慢速接入网络的用户
- TCP 拥塞算法是数据的发送端决定发送窗口，因此在哪边部署，就对哪边发出的数据有效。
    1. 如果是下载，就应该在服务器部署
    2. 如果是上传，就应该在客户端部署
- 既然不容易区分拥塞丢包和错误丢包，TCP BRR 就不考虑丢包。
- 既然灌满水管的方式容易造成缓冲区膨胀，TCP BRR 就分别估计带宽和延迟，而不是直接估计水管的容积。

### TCP 短连接和长连接的区别？

**短连接**

  Client 向 Server 发送请求，Server 回应 Client ，一次读写完成，此时双方任何一个都可以发起 `close` 操作，不过一般都是 Client 先发起 `close` 操作；

  短连接一般只会在 Client / Server 间传递一次读写操作；

  优点：管理起来简单，建立存在的连接都是有效连接，无需额外的控制手段；

**长连接**

  Client 和 Server 完成一次读写后，它们之间的连接不会主动关闭，后续的读写操作继续使用这个连接；

  由于在长连接的应用场景下，Client 一般不会主动关闭它们之间的连接，随着 Client 连接越来越多，Server 压力会逐渐增大，因此 Server 端需要采取一些策略（比如，关闭一些长事件没有读写事件发生的连接），避免某些恶意连接导致 Server 服务受损；

### TCP 粘包、拆包、以及解决方案

#### 为什么常说 TCP 有粘包、拆包问题，而不提 UDP 呢？

**UDP**

  UDP 是基于报文发送的，且 UDP 首部采用 16 bit 来指示 UDP 数据报文的长度，因此应用层可以很好的将不同的数据报文区分开，从而避免粘包、拆包问题。

**TCP**

  基于字节流的，虽然应用层和 TCP 传输层之间的数据交互是大小不一的数据块，但 TCP 并没有把这些数据块区分边界，仅仅是一连串没有结构的字节流；

  另外，TCP 的首部字段并没有表示数据长度的字段；

#### 什么是粘包、拆包？

**粘包、拆包**

  基于字节流的TCP数据，就是一连串的二进制数据，这些数据可能是多个较小的 TCP 报文拼接而成，也可能是一个大的 TCP 报文的一部分（TCP 进行了切片），且在这些二进制数据之间不存在边界，因此接收端没办法区分不同的TCP数据报文，即粘包、拆包问题；

  发生原因：

- 要发送的数据大于 TCP 发送缓冲区剩余空间大小，将会发生拆包；
- 带发送的数据大于 MSS（最大报文长度），TCP 在传输前将进行拆包（1500 - 20 - 20）；
- 要发送的数据小于 TCP 发送缓冲区的大小，TCP 将多次写入缓冲区的数据一次发送出去，发生粘包；
- 接收端的应用层没有及时读取接收缓冲区中的数据，发生粘包；

#### 解决方案

由于 TCP 基于字节流，无法理解上层的业务数据，因此在底层无法保证数据包不被拆分或重组，只能在上层的应用协议栈设计来解决。

根据业界的主流协议的解决方案如下：

- **消息定长**：发送端将每个数据包封装成固定长度（不足可补 0 ）
- **设置消息边界**：服务端从网络流中按消息边界分离出消息内容
- **将消息分为消息头和消息体**：消息头中包含消息总长度（或消息体长度）的字段

## MySQL 数据库

### 增删改查

#### **插入数据 `INSERT`**

```mysql
 INSERT INTO TableName 
     [columnName1, columnName2, ...]
 VALUES(
     <columnValue1>, <columnValue2>, ...
 );
```

---

#### **删除数据 `DELETE`**

```mysql
DELETE FROM <TableName>
WHERE <Condition>;
```

- **==`WHERE` 子句省略时，删除所有行==**

---

#### **更新数据 `UPDATE`**

```mysql
UPDATE <TableName>
SET <column1> = <value1>,
 <column2> = <value2>, ...
WHERE <Condition>;
```

- **==更新多个列时，只需要使用一个 `SET` 命令，每个`column = value` 对之间使用`,`分隔（最后一列后不用`,`)==**
- **==可以使用子查询==**

---

#### **检索数据 `SELECT`**

```mysql
SELECT [DISTINCT]
 <column1, column2, ...>
FROM
 <Table1 [[AS]<aliasName>], table2, ...>
WHERE
 <Condition1 [AND / OR / NOT] Condition2 ...>
GROUP BY
 <Column1, Column2, ...>
HAVING
 <Column1, Column2, ...>
ORDER BY
 <Column1 [ASC / DESC], Column2 [ASC / DESC], ...>
LIMIT
 <A> OFFSET <B>;
```

- **==`DISTINCT` 指示数据库只返回不同的值，且必须放在所有列名之前，作用于所有的列==，不仅仅是跟在其后的那一列**

- **==`LIMIT`，指定返回的行数，`OFFSET`指定从哪开始计算==**；

    MySQL、SQLite 中 `LIMIT 4 OFFSET 3` 等价于 `LIMIT 3, 4`

- **==`ORDER BY`，根据一个或多个列的名字进行排序（默认升序`ASC`，降序`DESC`）==**

- **==`WHERE`，指定筛选条件：==**

  - `=`/ `<>` / `!=` / `<` / `<=` / `!<` / `>` / `>=` / `!>` / `BETWEEN ... AND ...` / `IS NULL`

  - `AND` / `OR` / `IN` / `NOT`

  - `LIKE`

        `%` 表示==任何字符出现任意次数==
          
        `_` 表示==任意单个字符==

- **==`GROUP BY`，把数据进行逻辑分组，以便能对每个组进行聚集计算==**

- **==`HAVING`，过滤分组==**，类似于 `WHERE`：

    ==`WHERE` 过滤行，`HAVING` 过滤分组==

    ==`HAVING`支持所有 `WHERE` 操作==

    ==`WHERE`在分组前进行过滤，`HAVING`在分组后进行过滤== ==>> `WHERE` 过滤掉（排除掉）的行不包含在分组内

    ==`HAVING` 应与 `GROUP BY` 结合使用==

### 表格操作

#### 表结构的查看

**==查看指定表结构：`DESC <TableName>;`==**

**==查看指定表的建表语句：`SHOW CREATE TABLE <TableName>`==**

#### 表结构的创建

```mysql
CREATE TABLE <TableName> (
 ... [COMMENTS],
    ... [COMMENTS],
    ...
) [COMMENTS];
```

#### 表结构的更新

**==`ALTER TABLE`==**

- **添加字段：`ALTER TABLE <TableName> ADD <ColumnName> <dataType>(LENGTH) [<COMMENT> 'comments'] [CONSTRAINT];`**

    ```mysql
    ALTER TABLE Temp 
     ADD nickname 
     VARCHAR(20) 
     COMMENT '昵称';
    ```

- **修改字段数据类型：`ALTER TABLE <TableName> MODIFY <ColumnName> <newDataType>(LENGTH);`**

    **修改字段的数据类型和字段名字：`ALTER TABLE <TableName> CHANGE <OldName> <NewName> <dataType>(LENGTH) [COMMENT 'comments'][CONSTRAINT];`**

    ```mysql
    ALTER TABLE Temp 
     CHANGE nickName userName
     VARCHAR(32)
     COMMENT '昵称';
    ```

- **删除字段：`ALTER TABLE <TableName> DROP <columnName>;`**

#### 表结构的删除

**==永久删除该表：`DROP TABLE [IF EXISTS] <TableName>;`==**

**==删除表格内容保留表结构：`TRUNCATE [TABLE] <TableName>;`==**

**DELETE 是逐行一条一条删除记录的；**

**TRUNCATE 则是直接删除原来的表，再重新创建一个一模一样的新表，而不是逐行删除表中的数据，执行数据比 DELETE 快。**

#### 表结构的重命名

**需要指定旧表名和新表名**

==`ALTER TABLE oldTable RENAME [TO | AS] newTable;`==

==`RENAME TABLE oldTable TO newTable [, oldTable TO newTable [, ...]];`==

### 数据类型

|    类型     |    大小    |                 有符号（SIGNED）范围                  |                 无符号（UNSIGNED）范围                  |         描述         |
| :---------: | :--------: | :---------------------------------------------------: | :-----------------------------------------------------: | :------------------: |
|   TINYINT   | **1** Byte |                     $(-128, 127)$                     |                       （0, 255）                        |        小整数        |
|  SMALLINT   | **2** Byte |                   $(-32768, 32767)$                   |                       (0, 65535)                        |       大整数值       |
|  MEDIUMINT  | **3** Byte |                 $(-8388608, 8388607)$                 |                      (0, 16777215)                      |       大整数值       |
| INT/INTEGER | **4** Byte |                $(-2^{32}, 2^{32} - 1)$                |                     (0, 4294967295)                     |       大整数值       |
|   BIGINT    | **8** Byte |                $(-2^{63}, 2^{63} - 1)$                |                      (0, 2^64 - 1)                      |      极大整数值      |
|    FLOAT    | **4** Byte |       $(-3.402823466E+38, 3.402823466351E+38)$        |         0 和 (1.175494351E-38, 3.402823466E+38)         |    单精度浮点数值    |
|   DOUBLE    | **8** Byte | $(-1.7976931348623157E+308, 1.7976931348623157E+308)$ | 0 和 (2.2250738585072014E-308, 1.7976931348623157E+308) |    双精度浮点数值    |
|   DECIMAL   |            |            依赖于M（精度）和D（标度）的值             |             依赖于M（精度）和D（标度）的值              | 小数值（精确定点数） |

|    类型    | 大小 （Bytes） |                       描述                        |
| :--------: | :------------: | :-----------------------------------------------: |
|    CHAR    |    0 - 255     | ==定长字符串==（需指定长度），性能较VARCHAR更高些 |
|  VARCHAR   |   0 - 65535    |           ==变长字符串==（需指定长度）            |
|  TINYBLOB  |    0 - 255     |         ==不超过 255 个字符的二进制数据==         |
|  TINYTEXT  |    0 - 255     |                 ==短文本字符串==                  |
|    BLOB    |   0 - 65535    |            ==二进制==形式的长文本数据             |
|    TEXT    |   0 - 65535    |                    长文本数据                     |
| MEDIUMBLOB |  0 - 16777215  |         ==二进制==形式的中等长度文本数据          |
| MEDIUMTEXT |  0 - 16777215  |               ==中等长度文本数据==                |
|  LONGBLOB  | 0 - 4294967295 |           ==二进制==形式的极大文本数据            |
|  LONGTEXT  | 0 - 4294967295 |                   极大文本数据                    |

| 类型      | 大小 | 范围                                       | 格式                | 描述                     |
| --------- | ---- | ------------------------------------------ | ------------------- | ------------------------ |
| DATE      | 3    | 1000-01-01 至 9999-12-31                   | YYYY-MM-DD          | 日期值                   |
| TIME      | 3    | -838:59:59 至 838:59:59                    | HH:MM:SS            | 时间值或持续时间         |
| YEAR      | 1    | 1901 至 2155                               | YYYY                | 年份值                   |
| DATETIME  | 8    | 1000-01-01 00:00:00 至 9999-12-31 23:59:59 | YYYY-MM-DD HH:MM:SS | 混合日期和时间值         |
| TIMESTAMP | 4    | 1970-01-01 00:00:01 至 2038-01-19 03:14:07 | YYYY-MM-DD HH:MM:SS | 混合日期和时间值，时间戳 |

### 关系型数据三大范式

**第一范式**

  1NF ，==数据库表中的字段都是单一属性的，不可再分==；==每一个属性都是原子项，不可分割==；如果数据库设计不满足第一范式，就不称为关系型数据库；

**第二范式**

  2NF，==数据库表中，取保非主键列完全依赖于主键列==；**即每个非主键列都完全依赖于主键 ，而不是部分依赖**

  如果一个表中某一字段 A 的值是由另一个字段或一组字段 B 的值来确定的，就成为 A 依赖于 B；

**第三范式**

  3NF，==取保非主键列之间不存在传递依赖==；即每个非主键列都只依赖于主键列，而不依赖于其他非主键列；  

### 数据库的索引类型？作用？

**==索引是对数据库表中一个或多个列的值进行排序的结构==**

**优点**

- 大大加快数据的检索速度
- 创建唯一性索引，保证数据库表中每一行数据的唯一性
- 加速表和表之间的连接
- 在使用分组和排序子句进行数据检索时，可以显著减少查询时分组和排序的时间

**缺点**

- 索引需要占用数据表以外的物理存储空间
- 创建索引和维护索引需要花费一定的时间
- 当对表进行更新操作时，索引需要被重建，降低了数据的维护速度

**类型**

- ==唯一索引 UNIQUE==：此索引的每一个索引值只对应唯一的数据记录；

    对于单列唯一性索引，这保证单列不包含重复的值；

    对于多列唯一性索引，保证多个值的组合不重复；

- ==主键索引 Primary Key==：数据库表经常有一列或列组合，其值唯一标识表中的每一行，该列称为表的主键。

    **主键索引是唯一所应的特定类型，要求主键中的每个值都唯一；**

    **当在查询中使用主键索引时，**

**实现方式**

- B+ 树、哈希表、位图索引

### 聚簇索引 和 非聚簇索引的区别？

**聚簇索引**

  聚簇索引表示表中存储的数据按照索引的顺序存储，检索效率比非聚簇索引高，但对数据更新影响较大；

  聚簇索引一个表只能有一个；

  聚簇索引存储是物理上连续的；

**非聚簇索引**

  非聚簇索引表示数据存储在一个地方，索引存储在另一个地方，索引带有指向数据的存储位置，索引检索效率比聚簇索引低，但对数据更新影响较小；

  非聚簇索引一个表可以存在多个；

  非聚簇索引是逻辑上连续，物理存储不连续；

### MySQL 中索引的常见类型有哪些？

**哈希表索引模型**

- 键 - 值（key - value）存储数据的结构
- 优点：查询速度快、记录增加速度快
- 缺点：key 的存储不是采用有序存储，区间查询速度慢（需要多次调用 Hash 函数）
- 适用场景：只需要等值查询（没有区间查询），例如 Memcached、NoSQL

**有序数组索引模型**

- 等值查询和范围查询场景下的性能都很优秀
- 只适用于==静态存储==引擎，因为更新数据的成本太高

**N 叉搜索树索引模型**

- N 值的大小取决于数据块的大小，InnoDB 的一个整数字段索引，N 差不多为 1200（当树高为 4 时，可以存储 $N^3$ 个数值，将近 17 亿）；考虑到树根的数据块总是在内存中，一个 10 亿行的表上的一个整数字段的索引，查询一个值最多只需要 3 次访盘；
- 主键长度越小，普通索引的叶子节点就越小，普通索引占用的空间也就越小

### 为什么 MySQL 采用 B+ 树，而不是 B树、二叉树？

==B 树，不再限制一个节点就只能有 2 个子节点，而是允许 M 个子节点 (M>2)==，从而降低树的高度；==B 树的每一个节点最多可以包括 M 个子节点，M 称为 B 树的阶==，所以 B 树就是一个**多叉树**；

<img src="https://raw.githubusercontent.com/lutianen/PicBed/main/202308051116233.png" alt="图片" style="zoom:60%;" />        <img src="https://cdn.xiaolincoding.com//mysql/other/dd076212a7637b9032c97a615c39dcd7.png" alt="图片" style="zoom: 80%;" />

B+ 树与 B 树的差异：

- 叶子节点（最底部的节点）才会存放实际数据（索引+记录），非叶子节点只会存放索引；
- 所有索引都会在叶子节点出现，叶子节点之间构成一个有序链表；
- 非叶子节点的索引也会同时存在在子节点中，并且是在子节点中所有索引的最大（或最小）。
- 非叶子节点中有多少个子节点，就有多少个索引；

**==B树不管叶子节点还非叶子节点，都会保存数据，这样导致了非叶子节点中能保存的指针数量就变少==**

**==B树每个节点都存储数据，而B+树只有叶子节点才存储数据，所以在查询相同数据量的情况下，B树的IO会更频繁==**

**==B+ 树叶子节点之间用链表连接了起来，有利于范围查询，而 B 树要实现范围查询，因此只能通过树的遍历来完成范围查询，涉及多个节点的磁盘 I/O 操作，范围查询效率不如 B+ 树==**

### `(A, B, C)` 联合索引生效问题？

**==MySQL 从左到右的是由索引中的字段，一个查询可以只使用索引中的一部分，但是必须包含最左侧部分（即A）==**

实例：`select * from myTest where a=3 and b>7 and c=3;` ==b范围值，断点，阻塞了c的索引==；a用到了，b也用到了，c没有用到，这个地方b是范围值，也算断点，只不过自身用到了索引

`select * from myTest where b=3 and c=4;` ==联合索引必须按照顺序使用，并且需要全部使用==，因为a索引没有使用，所以这里 b、c 都没有用上索引效果

- 最左前缀原则

  - 最左侧字段不存在
  - 中间字段跳过

- 范围查询 (**在业务允许的情况下，尽可能使用类似于 `>=` 或 `<=` 这类的范围查询，而避免使用 `>` 或 `<`**)

  - 联合索引中，出现范围查询（`>`, `<`），范围查询右侧的列索引失效
  - 当范围查询使用`>=` 或 `<=` 时，范围查询没有导致索引失效

- 索引列运算

  - 不要在索引列上进行运算操作，否则将索引失效

        ```mysql
        select * from tb_user where phone='12345678901';
        select * from tb_user where substring(phone, 10, 2) = '15';
        ```

- 字符串不加引号

  - 字符串类型字段使用时，不加引号，索引将失效；隐式类型转换；

- 模糊查询

  - 如果仅仅是尾部模糊查询，索引不会失效；
  - 如果头部模糊匹配，索引失效；

- or 连接条件

  - 用 or 分割开的条件，如果 or 前的条件中列有索引，而后面的列中没有索引，那么设计的索引都不会被用到；

        **当 or 连接的条件，左右两侧字段都有索引时，索引才会生效**

- 数据分布影响

  - 如果 MySQL 评估使用索引比全表更慢，则不使用索引

### 什么是 覆盖索引、索引下推？

**覆盖索引**

  ==覆盖索引，指的是查询语句的执行只使用了索引树，而不需要访问数据表==，因为索引树中已经包含了查询所需要的数据；

  ==通过查询非聚簇索引，即可获取结果==，**可以减少树的搜索次数，显著提升查询性能**，是一个常用的性能优化手段；

  实例：`SELECT ID FROM T WHERE k between 3 and 5;`只需要查询 ID，而 ID 在 k 索引树上，可直接提供查询结果，不需要回表操作；

**索引下推**

  索引下推（Index Predicate Pushdown）是一种优化技术，用于提高数据库查询性能。

  它在执行查询时，尽可能地将查询条件下推到索引层级进行处理，以减少查询的数据量和计算量。

  **索引条件下推优化可以减少存储引擎查询基础表的次数，也可以减少MySQL服务器从存储引擎接收数据的次数**

  索引下推在**非主键索引**上的优化，可以有效减少回表的次数，大大提升了查询的效率

  <img src="https://raw.githubusercontent.com/lutianen/PicBed/main/202308051108213.png" alt="image-20230805110847156" style="zoom:67%;" />

### 并行事务会引发什么问题？如何解决？

**==脏读（dirty read）==**：一个事务读取到另一个事务未提交但修改过的数据；

**==不可重复读（non-repeatable read）==**：在同一个事务中，多次读取同一个行数据，前后读取到的数据不同；

**==幻读（phantom read）==**：在一个事务中，多次查询符合某个条件的记录数量时，前后出现记录数量不一致的情况；

**解决方案**：设置不同的事务隔离级别

- ==读未提交 Read uncommitted==

    一个事务还没有提交，该事务所做的改变就能被其他事务看到；

    实现原理：**直接返回改行记录的最新数据，没有视图的概念**

    并发问题：脏读、不可重复读、幻读

- ==读提交 Read committed==

    一个事务提交之后，该事务所做的改变才会被其他事务看到；

    实现原理：**在每个 SQL 语句开始执行时，创建一个视图**

    并发问题：不可重复读、幻读

- ==可重复读 Repeatable read==

    一个事务执行过程中看到的数据，总是跟这个事务在启动是看到的数据一致；在该隔离级别下，未提交变更对其他事务是不可见的；

    实现原理：**在事务启动时创建一个视图，整个事务存在期间都用这个视图**

    并发问题：幻读

- ==串行化 Serializable==

    对于同一行记录，“写” 操作会加 “写锁”，“读” 操作会加 “读锁”；当出现读写锁冲突时，后访问的事务需要等待前一个事务提交后，才能继续执行；

    实现原理：**对数据库加锁，避免并行访问**

    并发问题：无

**==可以修改 / 配置事务隔离级别==**，针对 SESSION、GLOBAL，除非修改 `my.conf` 配置文件，否则重新连接时，依然是原来的事务隔离级别；

### InnoDB 通过何种技术保证事务的 ACID 四大特性？

**原子性 Atomicity**：Undo log

- 确保要么所有操作都执行成功，要么都不执行

**持久性 Durability**：Redo log

- 将事务日志记录到持久性存储设备上来保证持久性。即使在数据库发生崩溃或故障时，InnoDB可以从事务日志中恢复数据，确保已提交的事务对数据库的影响是持久的

**隔离性 Isolation**：MVCC

- 决定了事务在并发执行时的可见性，确保不同事务之间的数据访问互不干扰

**一致性 Consistency**：原子性 + 持久性 + 隔离性

- InnoDB通过在执行事务之前和之后检查数据的完整性来维护一致性

### Undo log

**==Undo log，用于回滚事务的日志，保证事务的原子性==**

### Redo log

**==用于实现事务的持久化，InnoDB 引擎独有==**，保证即使数据库发生重启，之前已提交的记录也不会丢失（Crash-Safe）；属于==物理日志==，记录的是“在某个数据页上做了什么修改”

**实现原理：** WAL （Writing Ahead Logging） + Buffer Pool（*当从数据库读取数据时，会先从Buffer Pool中读取数据，如果Buffer Pool中没有，则从磁盘读取后放入到Buffer Pool中。 当向数据库写入数据时，会先写入到Buffer Pool中，Buffer Pool中更新的数据会定期刷新到磁盘中（此过程称为刷脏）*）

  先写日志再刷盘；当有一条记录更新时，InnoDB 引擎会先把记录写到redo log中并更新内存（这时记录的更新就算完成了）；然后，InnoDB 引擎在适当的时候，将这个操作记录更新到磁盘中；

**InnoDB 的 redo log 大小固定但可配置**，一般一组为 4 个文件，每个文件大小为 1 GB ；采用==环形文件链==的结构进行组织，即从头开始写，写到末尾就又回到开头，循环写；

### bin log

**==归档日志，不具有 Crash-Safe 的能力==**；

bin log 是 MySQL 的 Server 层实现的，所有的引擎层都可以使用；

属于==逻辑日志==，记录的是语句的原始逻辑；

采用==追加写==的方式，但不会覆盖之前的 bin log（新建一个 bin log）；

### MVCC 是如何实现的？

==MVCC，多版本并发控制，是乐观锁的一种实现方式，可以做到读不加锁、读写不冲突，解决读写冲突的无锁并发控制。==通过「版本链」来控制并发事务访问同一个记录时的行为就叫 MVCC（多版本并发控制）；

<img src="https://raw.githubusercontent.com/lutianen/PicBed/main/202308051028750.png" alt="image-20230805102807669" style="zoom:80%;" />

**实现原理：**

  主要依赖记录中的 3 个隐式字段、undo log、Read View实现

- 记录事务 ID 的 `DB_TRX_ID`，大小为 6bit
- 记录上一个版本数据记录的回滚指针，大小 7 bit
- 隐含的自增 ID：在没有设置主键的情况下自动以 `DB_ROW_ID` 产生一个簇拥索引，大小 6bit

  **undo log 用于回滚事务的日志**

  **Read View 用于读取数据的视图**

### 讲一讲 MySQL 中的锁？

#### **锁的粒度划分**

**全局锁**

- `flush tables with read lock`，FWTRL，加全局锁 $<-->$ `unlock tables` 释放全局锁
- 锁整个数据库
- 场景：全库逻辑备份

**表级锁**

- 范围：对整张表进行加锁；
- 开销小、加锁快、并发度低
- 分类
  - 表锁
  - 元数据锁：**==自动给当前数据库表加锁==**，防止表结构被修改；当进行数据的 CURD 操作、对表的结构进行修改时自动加 MDL 锁；当事务提交时自动释放；
  - 意向锁
  - AUTO-INC 锁

**行级锁**

- MySQL 中粒度最细、开销最大的锁，只针对当前操作的行进行加锁
- 能够大大减少冲突的发生，并发度高
- InnoDB 引擎支持行级锁，MyISAM 不支持
- 分类
  - Record Lock，记录锁
  - Gap Lock，间隙锁
  - Next-Key Lock，临键锁：**==记录锁 + 间隙锁==**，锁定一个范围且锁定自身；
- ==两阶段锁==：在 InnoDB 引擎中，行锁是在需要时才加，但不是不需要就立即释放，而是需要等到事务结束才释放；==把最可能造成锁冲突、最可能影响并发度的锁尽量行后放==

#### 锁的方式划分

**共享锁 S锁**

  满足读读共享、读写互质

**排他锁 X锁**

  满足读写互质、写写互斥

#### 思想的方式划分

**悲观锁**

**乐观锁**

### MySQL 支持的存储引擎有哪些？

**InnoDB**

- 5.5.5 版本之后默认引擎
- 支持事务、行级锁、外键等特性
- 适用于高并发、高可靠性的应用场景
- 支持的索引类型：B+ Tree 索引、Full-Text 索引
- 支持在线热备份：采用**快照**实现

**MyISAM**

- 不支持事务、不支持行级锁
- 对于读密集型应用，查询效率更高
- 支持的索引类型：B+ Tree 索引、Full-Text 索引

**Memory**

- 数据存储在内存中，查询效率极高
- 不具有持久性，重启后数据消失
- 支持的索引类型：B+ Tree 索引、Hash 索引

### MySQL 主从复制是怎么实现的？

MySQL 主从复制是指**将数据从一个 MySQL 数据库服务器主节点复制到一个或多个从节点**的过程。可以提高数据库的可用性和性能，同时也可以实现数据备份和数据恢复。

<img src="https://raw.githubusercontent.com/lutianen/PicBed/main/202308050951200.png" alt="mysql 主从复制– Linux 技术杂谈" style="zoom:80%;" />

实现：

1. 当主机点进行 insert、update、delete 操作时，会顺序写入 bin log 中
2. 从主节点连接到从节点，主节点会创建一个 binlog dump 线程，将 binlog 的内容发送到从节点，==主节点会为自己的每一个从节点创建一个log dump 线程==
3. 从节点启动后，创建一个 I/O 线程，读取主节点传过来的 binlog 内容，并写入 relay log
4. 从节点还会创建一个 SQL 线程，从 relay log 里读取内容，从 Exec_Master_Log_Pos 位置开始执行读取的更新事件，将更新的内容写入到从节点的数据库

### MySQL 读写分离如何实现？如何区分读操作还是写操作？

利用“mysql-proxy”实现读写分离；

“mysql-proxy”是一个mysql官方提供用于实现读写分离的软件，也叫中间件，可以让主数据库处理写操作，而从数据库处理查询的操作，数据库的一致性则通过主从复制来实现。

mysql-proxy 能实现读写语句的区分主要依靠内部的一个lua脚本（能实现读写语句的判断）

****

## Redis 数据库

**==”NoSQL“ is “Not Only SQL”（不仅仅是SQL）==**

**==Redis （Remote Dictionary Server），远程字典服务==**

Redis is an in-memory **data structure store** used as a **database**, **cache**, **message broker,** and streaming engine.

### Redis 为什么是单线程？

> 官方表示：**Redis 是基于内存操作的，CPU不是Redis的性能瓶颈，Redis的瓶颈是根据机器的内存和网络带宽**，既然可以用单线程实现，就没必要使用多线程了。

> Redis 的提供数据为 100000+ (10W+) 的QPS，非常快

### Redis 为什么单线程还这么快？

**多线程（CPU上下文切换）不一定比单线程效率高！！！Redis 是将所有的数据全部存放在内存中的，所有说单线程去操作效率就是最高的，多线程（CPU上下文切换，是一个耗时操作）；**

**对于内存系统来说，如果没有上下文切换效率就是最高的！多次读写都是在一个CPU上完成的，在内存情况下就是最佳方案！**

### 五大基本数据类型

**Redis 数据库里面的每个键值对（key-value pair）都是由对象（object）组成的，其中：**

- **数据库键 key 总是一个字符串对象（string object）**
- **数据库键的值 value 则可以是字符串对象(string object)、列表对象(list object)、哈希对象(hash object)、集合对象(set object)、有序集合对象(sorted set object)中的一种**

**==String（字符串类型）==**

  value 除了可以是字符串，还可以是数字

  **使用场景**：==计数==（微博数、粉丝数），==缓存==

**==List （链表类型）==**

  在Redis 中，可以把 List 搞成队列、栈、阻塞队列

  **list 的 key 的底层实现就是一个链表，链表中的每一个节点都保存了一个整数值。**

 **Redis 的链表实现的特性**：

- 双向：链表节点都有 prev 和 next 指针，获取某个节点的前继节点和后继节点的复杂度都是O(1)
- 无环：表头节点的 prev 指针和表尾的 next 指针都指向 NULL，对链表的方位以 NULL为终点
- 带表头指针和表尾指针：通过 list 结构的 head 指针和 tail 指针
- 带链表长度计数器：list 结构包含 len 属性
- 多态：链表节点使用 `void*` 指针来保存节点值，并且可以通过 list 结构的 `dup`、`free`、`match sane`属性为节点值设置类型特定函数，因此链表可以用于保存各种不同类型的值；

 **使用场景**：==列表==（微博的关注列表、粉丝列表、消息列表）

**==Hash (哈希)==**

  本质和 String 类型没有太大区别，还是一个简单的 key-value！

  **hash 更加适合对象的存储，String 更适合字符串存储**

  **使用场景**：存储用户信息、商品信息

**==Set (集合)==**

  Set 中的元素不能重复且无序

  **使用场景**：共同关注、共同粉丝、公共喜好等

**==Zset(有序集合)==**

  Redis中，zset（有序集合）是一种特殊的数据类型，既可以像普通的集合（set）一样存储多个元素，还可以给每个元素赋一个分数（score），并根据分数对元素进行排序。在 set 中，**元素必须是唯一的，但分数可以重复**；

  **特点**

- 元素按照分数**从小到大**排序，可以通过指定范围获取分数在某个区间内的元素；
- **元素是唯一的**，每个元素都有一个唯一的标识符，称为成员（member）；
- 元素的分数可以是浮点数，用于区分元素之间的顺序；分数越小的元素越靠前；
- zset 中的元素可以动态添加、删除、修改分数，可以使用类似于集合（set）的命令来操作；
- zset 支持交集、并集等操作，可以对多个 zset 进行操作，得到一个新的 zset；

 **使用场景**：排行榜、弹幕消息等

### 三种特殊数据类型

**==geospatial 地理位置==**

  一种数据类型，可以存储和搜索坐标信息；适用于在给定的半径或矩形范围内查找附近的点；

  Redis 提供了 **GEO命令**，实际上**基于有序集合数据类型**，**通过 geohash 算法将维度和经度编码到有序集合的分数中来实现 geospatial 索引**；

**==HyperLogLog==**

  基数，不重复的元素，可以接收误差；

  Redis中 HyperLogLog 是用来做**基数统计**的，优点是在**输入元素的数量或体积非常非常大时，计算基数所需的空间总是固定的、并且是很小的**。

  Redis 中，**每个 HyperLogLog 键只需要花费 12 KB 内存，就可以计算 2^64 个不同元素的基数**；

  HyperLogLog 只会根据输入元素来计算基数，而不会存储输入元素本身，所以不能像集合那样，返回输入的各个元素

  需要有一定的容错，错误率为 0.81

**==Bitmap==**

  Redis 中 bitmap，基于string 类型进行的位操作，可以把一个字符串当作一个位向量来处理，也可以对一个或多个字符串进行位运算；

### RDB 持久化

RDB（Redis DataBase），==**Redis 会单独创建一个（fork）一个子进程来进行持久化，会先将数据写入到一个临时文件中，待持久化过程结束，然后用这个临时文件替换上次的持久化文件；整个过程中，主进程不进行任何 IO 操作，确保了极高的性能**==

生成的 RDB 文件`（dump.rdb，可修改）`是一个未经压缩的二进制文件，通过该文件可以还原生成 RDB 文件时的数据库状态；

**比 AOF 方式更高效，但 AOF 文件的更新频率更高**

<img src="https://raw.githubusercontent.com/lutianen/PicBed/main/202308042152612.png" alt="image-20230804215220451" style="zoom: 67%;" />

**优点**

- 适合大规模的数据恢复
- 对数据完整性不敏感

**缺点**

- ==RDB 最后一次持久化的数据可能丢失==
- `fork` 子进程时，会占用一定的内存空间

**触发机制**

- conf 配置文件中的条件满足时，自动触发 RDB 持久化机制；
- 执行 `FLUSH ALL` 命令，触发 RDB 持久化机制；
- 退出 Redis 服务器；

**载入 rdb 文件**

- **只有在 AOF 持久化功能处于关闭状态时，服务器才会使用 RDB 文件来还原数据库状态**

- **载入 RDB 文件期间，服务器会一直处于阻塞状态，直到载入工作完成**

### AOF 持久化

AOF（Append Of File），**==通过保存 Redis 服务器所执行的写命令来记录数据库状态 （读操作不记录）==**

被写入 AOF 文件的所有命令都是以 Redis 的命令请求协议格式保存的，纯文本格式；

默认无限追加至 aof 末尾，**默认不使用 AOF （`appendonly no`）**，默认文件名：`appendonly.aof`

<img src="https://raw.githubusercontent.com/lutianen/PicBed/main/202308042153906.png" alt="image-20230804215339776" style="zoom: 67%;" />

**优点**

- 每次修改都会进行持久化，文件完整性更好；
- 默认每秒同步一次，可能丢失一秒的数据；

**缺点**

- 相对于数据文件来说，AOF 远远大于 rdb，修复速度比 RDB 慢；
- AOF 运行效率比 RDB 慢，故默认 RDB 持久化；

**实现步骤**

1. 命令追加：append
2. 文件写入
3. 文件同步：sync

**AOF 重写**

 为了==解决 AOF 文件体积膨胀==的问题，Redis 提供了 ==AOF 文件重写(rewrite)== 功能：通过该功能，==Redis 服务器可以创建一个新的 AOF 文件来代替现有的 AOF 文件，新旧两个 AOF 文件所保存的数据库状态相同，但新 AOF 文件不会包含任何浪费空间的冗余命令==，因此新 AOF 文件体积通常比旧 AOF 文件小得多

**实际上，AOF 文件重写不需要对现有的 AOF 文件进行任何读取、分析或写入操作，而是通过读取服务器当前的数据库状态来实现**

实现原理：**采用子进程，首先从数据库中读取键现有的值，然后用一条命令去记录键值对，代替之前记录这个键值对的多条命令**

### **Redis 缓存穿透和雪崩**

#### 缓存穿透

用户想要查询一个数据，发现 ==Redis 内存数据库不存在，也就是缓存没有命中==，于是向持久层数据库查询，发现也没有，查询失败。

当用户很多的时候，缓存都没有命中（典型场景：秒杀），于是都去请求持久层数据库，这是对持久层数据库造成很大压力，相当于出现了缓存穿透。

**解决方案**

- 布隆过滤器

    布隆过滤器，是一种数据结构，对所有可能查询的参数以 hash 形式存储，在控制层先进性校验，不符合则舍弃，从而避免了对底层存储系统的查询压力；

- 缓存空对象

    当存储层不命中后，即使返回的空对象也将其缓存起来，同时设置一个过期时间，之后访问这个数据将从缓存中获取，保护了后端资源。

    **缺点**

  - 如果空值能够被缓存起来，这就意味着缓存需要更多的空间存储更多的键，因为这当中可能会有很多空值的键；
  - 即使对控制设置了过期时间，还是会存在缓存层和存储层的数据会有一段时间窗口的不一致，这对于需要保持一致性的业务会有影响；

#### 缓存击穿

缓存击穿，是==指一个 key 非常热点，在不停抗着大并发，大并发集中对这一个点进行访问，当这个 key 存在失效的瞬间，持续的大并发就会穿破缓存，直接请求数据库，就像在屏幕上凿开了一个洞==。

当某个 key 在过期的瞬间，有大量的请求并发访问，这类数据一般是热点数据，==由于缓存过期，会同时访问数据库来查询最新数据，并写回缓存，会导致数据库数据压力过大==。

**解决方案**

- 设置热点数据永不不过期

    从缓存层来看，不设置过期时间，就不会出现热点 key 过期后产生的问题；

- 互斥锁

    分布式锁：使用分布式锁，保证对于每个 key 同时只有一个线程去查询后端服务，其他线程没有获得分布式锁的权限，因此只需要等待即可。**将高并发的压力转移到了分布式锁**；

#### 缓存雪崩

缓存雪崩，==指某一个时间段，缓存集中失效==。比如 Redis 宕机！

**产生原因**：断电、断网；

**解决方案**

- Redis 高可用

    搭建集群

- 限流降级

    在缓存失效后，通过加锁或队列来控制读数据库写缓存的线程数量；比如对某个 key 只允许一个线程查询数据和写缓存，其他线程等待；

- 数据预热

    ==在正式部署之前，先把可能的数据预先访问一遍，这样部分可能大量访问的数据就会被加载到缓存中==。在即将发生大并发访问前手动触发加载缓存不同的 key，设置不同的过期时间，让缓存失效的时间点尽量均匀；

## 设计模式

> 设计模式，主要是针对面向对象语言提出的一种设计思想，主要是提高代码可复用性，抵御变化，尽量将变化所带来的影响降到最低

### 设计原则

- ==单一职责原则 SRP（Single Responsibility Principle）==：就一个类而言，应该仅有一个引起它变化的原因；
- ==开放封闭原则 OCP（Open Close Principle）==：软件实体可以扩展，但是不可修改，即面对需求，对程序的改动可以通过增加代码来实现，但是不能改动现有的代码；
- ==里氏代换原则==：一个软件实体如果使用的是一个基类，那么一定适用于派生类。即在软件中，把基类替换成派生类，程序的行为没有变化；
- ==依赖倒转原则 DIP（Dependence Inversion Principle）==：抽象不依赖细节、细节应该依赖抽象，即面向接口编程，而不是针对实现编程；
- ==迪米特原则 LSP（Liskov Substitution Principle）==：如果两个类不直接通信，那么这两个类就不应该发生直接的相互作用。如果一类需要调用另一个类的某个方法的话，可以通过第三个类来转发这个调用；
- ==接口隔离原则 ISP（Interface Segregation Principle）==：每个接口中不存在派生类用不到却必须实现的方法，如果不然，就要将接口拆分，使用多个隔离的接口；

### 单例模式

单例模式，是一种创建型设计模式，让你能够保证一个类只有一个实例，并提供一个访问该实例的全局节点。

单例模式禁止通过除特殊构建方法以外的任何方式来创建自身类的对象。 该方法可以创建一个新对象， 但如果该对象已经被创建， 则返回已有的对象。

#### 适用场景

1. **如果程序中的某个类对于所有客户端只有一个可用的实例**，可以使用单例模式
2. **如果你需要更加严格地控制全局变量**，可以使用单例模式

#### 示例代码

**==饿汉式==**：在单例类定义的时候就进行实例化，==线程安全==；

在最开始的时候静态对象就已经创建完成,设计方法是类中包含一个静态成员指针,该指针指向该类的一个对象, 提供一个公有的静态成员方法,返回该对象指针,为了使得对象唯一,构造函数设为私有。

```cpp
class SingletonInstance {
public:
    static SingletonInstance* getInstance() {
        // 局部静态变量
        static SingletonInstance ins;
        return ins;
    }
    
    ~SingletonInstance() = default;
    
private:
    SingletonInstance() { std::cout << "SingletonInstance() Hungry " << std::endl; }
    SingletonInstance(const SingletonInstance&) {};
    SingletonInstance& operator=(const SingletonInstance& rhs) { return *this; };
};
```

**==懒汉式==**：==线程安全需要加锁==

尽可能晚的创建这个对象的实例，即在类第一次被引用的时候就将自己初始化，*C++ 很多地方都有类似的思想，比如 COW、晚绑定等*

```c++
/**
 * The Singleton class defines the `GetInstance` method that serves as an
 * alternative to constructor and lets clients access the same instance of this
 * class over and over.
 */
class Singleton
{
    /**
     * The Singleton's constructor/destructor should always be private to
     * prevent direct construction/desctruction calls with the `new`/`delete`
     * operator.
     */
private:
    static Singleton * pinstance_;
    static std::mutex mutex_;

protected:
    Singleton(const std::string value): value_(value)
    {
    }
    ~Singleton() {}
    std::string value_;

public:
    /**
     * Singletons should not be cloneable.
     */
    Singleton(Singleton &other) = delete;
    /**
     * Singletons should not be assignable.
     */
    void operator=(const Singleton &) = delete;
    /**
     * This is the static method that controls the access to the singleton
     * instance. On the first run, it creates a singleton object and places it
     * into the static field. On subsequent runs, it returns the client existing
     * object stored in the static field.
     */
    static Singleton *GetInstance(const std::string& value);
    /**
     * Finally, any singleton should define some business logic, which can be
     * executed on its instance.
     */
    void SomeBusinessLogic()
    {
        // ...
    }
    
    std::string value() const{
        return value_;
    } 
};

/**
 * Static methods should be defined outside the class.
 */
Singleton* Singleton::pinstance_{nullptr};
std::mutex Singleton::mutex_;

/**
 * The first time we call GetInstance we will lock the storage location
 *      and then we make sure again that the variable is null and then we
 *      set the value. RU:
 */
Singleton *Singleton::GetInstance(const std::string& value)
{
    std::lock_guard<std::mutex> lock(mutex_);
    if (pinstance_ == nullptr)
    {
        pinstance_ = new Singleton(value);
    }
    return pinstance_;
}

void ThreadFoo(){
    // Following code emulates slow initialization.
    std::this_thread::sleep_for(std::chrono::milliseconds(1000));
    Singleton* singleton = Singleton::GetInstance("FOO");
    std::cout << singleton->value() << "\n";
}

void ThreadBar(){
    // Following code emulates slow initialization.
    std::this_thread::sleep_for(std::chrono::milliseconds(1000));
    Singleton* singleton = Singleton::GetInstance("BAR");
    std::cout << singleton->value() << "\n";
}

int main()
{   
    std::cout <<"If you see the same value, then singleton was reused (yay!\n" <<
                "If you see different values, then 2 singletons were created (booo!!)\n\n" <<
                "RESULT:\n";   
    std::thread t1(ThreadFoo);
    std::thread t2(ThreadBar);
    t1.join();
    t2.join();
    
    return 0;
}
```

---

### 模板方法

亦称：Template Method

**模板方法模式，是一种行为设计模式，它在超类中定义了一个算法的框架，允许子类在不修改结构的情况下重写算法的特定步骤**，即父类定义算法的框架，而将一些步骤延迟到子类去实现，使得子类可以复用骨架，并附加特性；

以开发框架为例，框架开发人员把框架调用流程定好，而将某些具体的步骤作为虚函数留给子类去重写；

#### 适用场景

1. **当你只希望客户端扩展某个特定算法步骤，而不是整个算法或其结构时，可使用模板方法模式**。
2. 当多个类的算法除一些细微不同之外几乎完全一样时， 你可使用该模式。 但其后果就是， 只要算法发生变化， 你就可能需要修改所有的类。

**使用示例：** 模板方法模式在 C++ 框架中很常见。 开发者通常使用它来向框架用户提供通过继承实现的、 对标准功能进行扩展的简单方式。

**识别方法：** 模板方法可以通过行为方法来识别， 该方法已有一个在基类中定义的 “默认” 行为。

#### 示例代码

```c++
/**
 * The Abstract Class defines a template method that contains a skeleton of some
 * algorithm, composed of calls to (usually) abstract primitive operations.
 *
 * Concrete subclasses should implement these operations, but leave the template
 * method itself intact.
 */
class AbstractClass {
  /**
   * The template method defines the skeleton of an algorithm.
   */
 public:
  void TemplateMethod() const {
    this->BaseOperation1();
    this->RequiredOperations1();
    this->BaseOperation2();
    this->Hook1();
    this->RequiredOperation2();
    this->BaseOperation3();
    this->Hook2();
  }
  /**
   * These operations already have implementations.
   */
 protected:
  void BaseOperation1() const {
    std::cout << "AbstractClass says: I am doing the bulk of the work\n";
  }
  void BaseOperation2() const {
    std::cout << "AbstractClass says: But I let subclasses override some operations\n";
  }
  void BaseOperation3() const {
    std::cout << "AbstractClass says: But I am doing the bulk of the work anyway\n";
  }
  /**
   * These operations have to be implemented in subclasses.
   */
  virtual void RequiredOperations1() const = 0;
  virtual void RequiredOperation2() const = 0;
  /**
   * These are "hooks." Subclasses may override them, but it's not mandatory
   * since the hooks already have default (but empty) implementation. Hooks
   * provide additional extension points in some crucial places of the
   * algorithm.
   */
  virtual void Hook1() const {}
  virtual void Hook2() const {}
};
/**
 * Concrete classes have to implement all abstract operations of the base class.
 * They can also override some operations with a default implementation.
 */
class ConcreteClass1 : public AbstractClass {
 protected:
  void RequiredOperations1() const override {
    std::cout << "ConcreteClass1 says: Implemented Operation1\n";
  }
  void RequiredOperation2() const override {
    std::cout << "ConcreteClass1 says: Implemented Operation2\n";
  }
};
/**
 * Usually, concrete classes override only a fraction of base class' operations.
 */
class ConcreteClass2 : public AbstractClass {
 protected:
  void RequiredOperations1() const override {
    std::cout << "ConcreteClass2 says: Implemented Operation1\n";
  }
  void RequiredOperation2() const override {
    std::cout << "ConcreteClass2 says: Implemented Operation2\n";
  }
  void Hook1() const override {
    std::cout << "ConcreteClass2 says: Overridden Hook1\n";
  }
};
/**
 * The client code calls the template method to execute the algorithm. Client
 * code does not have to know the concrete class of an object it works with, as
 * long as it works with objects through the interface of their base class.
 */
void ClientCode(AbstractClass *class_) {
  // ...
  class_->TemplateMethod();
  // ...
}

int main() {
  std::cout << "Same client code can work with different subclasses:\n";
  ConcreteClass1 *concreteClass1 = new ConcreteClass1;
  ClientCode(concreteClass1);
  std::cout << "\n";

  std::cout << "Same client code can work with different subclasses:\n";
  ConcreteClass2 *concreteClass2 = new ConcreteClass2;
  ClientCode(concreteClass2);

  delete concreteClass1;
  delete concreteClass2;
  return 0;
}
```

---

### 策略模式

**策略模式，是一种行为设计模式，它能让你定义一系列算法，并将每种算法分别放入独立的类中，以使算法的对象能够互相替换**，一般为了解决多个 `if-else` 带来的复杂性，在多个算法相似的情况下，通过策略模式可以减少 `if-else` 带来的复杂性和难以维护性，一般在项目中发现发现多个 `if-else` 并且预感将来还会在此增加 `if-else` 分支，就可以考虑使用策略模式；

#### 适用场景

1. **当你想使用对象中各种不同的算法变体**， 并希望能在运行时切换算法时，可使用策略模式
2. **当你有许多仅在执行某些行为时略有不同的相似类时**，可使用策略模式
3. 如果算法在上下文的逻辑中不是特别重要， 使用该模式能将类的业务逻辑与其算法实现细节隔离开来。
4. 当类中使用了复杂条件运算符以在同一算法的不同变体中切换时，可使用该模式

#### 实例代码

```c++
/**
 * The Strategy interface declares operations common to all supported versions
 * of some algorithm.
 *
 * The Context uses this interface to call the algorithm defined by Concrete
 * Strategies.
 */
class Strategy {
public:
    virtual ~Strategy() = default;
    virtual std::string doAlgorithm(std::string_view data) const = 0;
};

/**
 * The Context defines the interface of interest to clients.
 */
class Context {
    /**
     * @var Strategy The Context maintains a reference to one of the Strategy
     * objects. The Context does not know the concrete class of a strategy. It
     * should work with all strategies via the Strategy interface.
     */
private:
    std::unique_ptr<Strategy> strategy_;
    /**
     * Usually, the Context accepts a strategy through the constructor, but also
     * provides a setter to change it at runtime.
     */
public:
    explicit Context(std::unique_ptr<Strategy> &&strategy = {}) : strategy_(std::move(strategy)) {}

    /**
     * Usually, the Context allows replacing a Strategy object at runtime.
     */
    void set_strategy(std::unique_ptr<Strategy> &&strategy) { strategy_ = std::move(strategy); }

    /**
     * The Context delegates some work to the Strategy object instead of
     * implementing +multiple versions of the algorithm on its own.
     */
    void doSomeBusinessLogic() const {
        if (strategy_) {
            std::cout << "Context: Sorting data using the strategy (not sure how it'll do it)\n";
            std::string result = strategy_->doAlgorithm("aecbd");
            std::cout << result << "\n";
        } else {
            std::cout << "Context: Strategy isn't set\n";
        }
    }
};

/**
 * Concrete Strategies implement the algorithm while following the base Strategy
 * interface. The interface makes them interchangeable in the Context.
 */
class ConcreteStrategyA : public Strategy {
public:
    std::string doAlgorithm(std::string_view data) const override {
        std::string result(data);
        std::sort(std::begin(result), std::end(result));

        return result;
    }
};
class ConcreteStrategyB : public Strategy {
    std::string doAlgorithm(std::string_view data) const override {
        std::string result(data);
        std::sort(std::begin(result), std::end(result), std::greater<>());

        return result;
    }
};

/**
 * The client code picks a concrete strategy and passes it to the context. The
 * client should be aware of the differences between strategies in order to make
 * the right choice.
 */
void clientCode() {
    Context context(std::make_unique<ConcreteStrategyA>());
    std::cout << "Client: Strategy is set to normal sorting.\n";
    context.doSomeBusinessLogic();
    std::cout << "\n";
    std::cout << "Client: Strategy is set to reverse sorting.\n";
    context.set_strategy(std::make_unique<ConcreteStrategyB>());
    context.doSomeBusinessLogic();
}

int main() {
    clientCode();
    return 0;
}
```

### 工厂方法

亦称：虚拟构造函数、Virtual Constructor、Factory Method

**==工厂方法模式，是一种创建型设计模式，其在父类中提供一个创建对象的方法，允许子类决定实例化对象的类型。==**

工厂方法产生的对象通常称为“产品”。

**仅当这些产品具有共同的基类或者接口时， 子类才能返回不同类型的产品， 同时基类中的工厂方法还应将其返回类型声明为这一共有接口**

#### 适用场景

1. **当你在编写代码的过程中**，**如果无法预知对象确切类别及其依赖关系时**，**可使用工厂方法**。
2. **如果你希望用户能扩展你软件库或框架的内部组件**，**可使用工厂方法**。
3. **如果你希望复用现有对象来节省系统资源**，**而不是每次都重新创建对象**，**可使用工厂方法**。

#### 示例代码

```c++
/**
 * The Product interface declares the operations that all concrete products must
 * implement.
 */

class Product {
 public:
  virtual ~Product() {}
  virtual std::string Operation() const = 0;
};

/**
 * Concrete Products provide various implementations of the Product interface.
 */
class ConcreteProduct1 : public Product {
 public:
  std::string Operation() const override {
    return "{Result of the ConcreteProduct1}";
  }
};
class ConcreteProduct2 : public Product {
 public:
  std::string Operation() const override {
    return "{Result of the ConcreteProduct2}";
  }
};

/**
 * The Creator class declares the factory method that is supposed to return an
 * object of a Product class. The Creator's subclasses usually provide the
 * implementation of this method.
 */

class Creator {
  /**
   * Note that the Creator may also provide some default implementation of the
   * factory method.
   */
 public:
  virtual ~Creator(){};
  virtual Product* FactoryMethod() const = 0;
  /**
   * Also note that, despite its name, the Creator's primary responsibility is
   * not creating products. Usually, it contains some core business logic that
   * relies on Product objects, returned by the factory method. Subclasses can
   * indirectly change that business logic by overriding the factory method and
   * returning a different type of product from it.
   */

  std::string SomeOperation() const {
    // Call the factory method to create a Product object.
    Product* product = this->FactoryMethod();
    // Now, use the product.
    std::string result = "Creator: The same creator's code has just worked with " + product->Operation();
    delete product;
    return result;
  }
};

/**
 * Concrete Creators override the factory method in order to change the
 * resulting product's type.
 */
class ConcreteCreator1 : public Creator {
  /**
   * Note that the signature of the method still uses the abstract product type,
   * even though the concrete product is actually returned from the method. This
   * way the Creator can stay independent of concrete product classes.
   */
 public:
  Product* FactoryMethod() const override {
    return new ConcreteProduct1();
  }
};

class ConcreteCreator2 : public Creator {
 public:
  Product* FactoryMethod() const override {
    return new ConcreteProduct2();
  }
};

/**
 * The client code works with an instance of a concrete creator, albeit through
 * its base interface. As long as the client keeps working with the creator via
 * the base interface, you can pass it any creator's subclass.
 */
void ClientCode(const Creator& creator) {
  // ...
  std::cout << "Client: I'm not aware of the creator's class, but it still works.\n"
            << creator.SomeOperation() << std::endl;
  // ...
}

/**
 * The Application picks a creator's type depending on the configuration or
 * environment.
 */

int main() {
  std::cout << "App: Launched with the ConcreteCreator1.\n";
  Creator* creator = new ConcreteCreator1();
  ClientCode(*creator);
  std::cout << std::endl;
  std::cout << "App: Launched with the ConcreteCreator2.\n";
  Creator* creator2 = new ConcreteCreator2();
  ClientCode(*creator2);

  delete creator;
  delete creator2;
  return 0;
}
```

### 观察者模式

亦称：事件订阅者、监听者、Event-Subscribe、Listener、Observer

**观察者模式，是一种行为设计模式，允许你定义一种订阅机制，可在对象事件发生时通知多个 "观察" 该对象的其他对象。**

拥有一些值得关注的状态的对象通常被称为*目标*， 由于它要将自身的状态改变通知给其他对象， 我们也将其称为*发布者* （publisher）。 所有希望关注发布者状态变化的其他对象被称为*订阅者* （subscribers）。

实际上， 该机制包括:
  1） 一个用于存储订阅者对象引用的列表成员变量；
  2） 几个用于添加或删除该列表中订阅者的公有方法。

所有订阅者都必须实现同样的接口， 发布者仅通过该接口与订阅者交互。 接口中必须声明通知方法及其参数， 这样发布者在发出通知时还能传递一些上下文数据。

#### 适用场景

1. **当一个对象状态的改变需要改变其他对象，或实际对象是事先未知的或动态变化的时，可使用观察者模式**。
2. **当应用中的一些对象必须观察其他对象时，可使用该模式。但仅能在有限时间内或特定情况下使用**。

#### 示例代码

```c++
/**
 * Observer Design Pattern
 *
 * Intent: Lets you define a subscription mechanism to notify multiple objects
 * about any events that happen to the object they're observing.
 *
 * Note that there's a lot of different terms with similar meaning associated
 * with this pattern. Just remember that the Subject is also called the
 * Publisher and the Observer is often called the Subscriber and vice versa.
 * Also the verbs "observe", "listen" or "track" usually mean the same thing.
 */

#include <iostream>
#include <list>
#include <string>

class IObserver {
 public:
  virtual ~IObserver(){};
  virtual void Update(const std::string &message_from_subject) = 0;
};

class ISubject {
 public:
  virtual ~ISubject(){};
  virtual void Attach(IObserver *observer) = 0;
  virtual void Detach(IObserver *observer) = 0;
  virtual void Notify() = 0;
};

/**
 * The Subject owns some important state and notifies observers when the state
 * changes.
 */

class Subject : public ISubject {
 public:
  virtual ~Subject() {
    std::cout << "Goodbye, I was the Subject.\n";
  }

  /**
   * The subscription management methods.
   */
  void Attach(IObserver *observer) override {
    list_observer_.push_back(observer);
  }
  void Detach(IObserver *observer) override {
    list_observer_.remove(observer);
  }
  void Notify() override {
    std::list<IObserver *>::iterator iterator = list_observer_.begin();
    HowManyObserver();
    while (iterator != list_observer_.end()) {
      (*iterator)->Update(message_);
      ++iterator;
    }
  }

  void CreateMessage(std::string message = "Empty") {
    this->message_ = message;
    Notify();
  }
  void HowManyObserver() {
    std::cout << "There are " << list_observer_.size() << " observers in the list.\n";
  }

  /**
   * Usually, the subscription logic is only a fraction of what a Subject can
   * really do. Subjects commonly hold some important business logic, that
   * triggers a notification method whenever something important is about to
   * happen (or after it).
   */
  void SomeBusinessLogic() {
    this->message_ = "change message message";
    Notify();
    std::cout << "I'm about to do some thing important\n";
  }

 private:
  std::list<IObserver *> list_observer_;
  std::string message_;
};

class Observer : public IObserver {
 public:
  Observer(Subject &subject) : subject_(subject) {
    this->subject_.Attach(this);
    std::cout << "Hi, I'm the Observer \"" << ++Observer::static_number_ << "\".\n";
    this->number_ = Observer::static_number_;
  }
  virtual ~Observer() {
    std::cout << "Goodbye, I was the Observer \"" << this->number_ << "\".\n";
  }

  void Update(const std::string &message_from_subject) override {
    message_from_subject_ = message_from_subject;
    PrintInfo();
  }
  void RemoveMeFromTheList() {
    subject_.Detach(this);
    std::cout << "Observer \"" << number_ << "\" removed from the list.\n";
  }
  void PrintInfo() {
    std::cout << "Observer \"" << this->number_ << "\": a new message is available --> " << this->message_from_subject_ << "\n";
  }

 private:
  std::string message_from_subject_;
  Subject &subject_;
  static int static_number_;
  int number_;
};

int Observer::static_number_ = 0;

void ClientCode() {
  Subject *subject = new Subject;
  Observer *observer1 = new Observer(*subject);
  Observer *observer2 = new Observer(*subject);
  Observer *observer3 = new Observer(*subject);
  Observer *observer4;
  Observer *observer5;

  subject->CreateMessage("Hello World! :D");
  observer3->RemoveMeFromTheList();

  subject->CreateMessage("The weather is hot today! :p");
  observer4 = new Observer(*subject);

  observer2->RemoveMeFromTheList();
  observer5 = new Observer(*subject);

  subject->CreateMessage("My new car is great! ;)");
  observer5->RemoveMeFromTheList();

  observer4->RemoveMeFromTheList();
  observer1->RemoveMeFromTheList();

  delete observer5;
  delete observer4;
  delete observer3;
  delete observer2;
  delete observer1;
  delete subject;
}

int main() {
  ClientCode();
  return 0;
}
```

### 装饰器模式

亦称： 装饰者模式、装饰器模式、Wrapper、Decorator

**装饰模式，一种结构型设计模式，允许通过将对象放入包含行为的特殊封装对象中来为原对象绑定新的行为**，即动态地给一个对象添加一些额外的职责，就增加功能来说，装饰模式比生成子类更为灵活，不一定非要疯狂使用继承方式.

#### 适用场景

- 希望在无需修改修改代码的情况下即可使用对象，且希望在运行时为对象新增额外的行为，可以使用装饰器
- 如果使用继承扩展对象行为的方案难以实现或者根本行不通，则可以使用装饰器

#### 识别方法

**通过当前类或对象为参数的创建方法或构造函数来识别**

#### 代码实例

```cpp
/**
 * The base Component interface define operations that can be altered by decorators.
 */
class Component {
public:
    virtual ~Component() {}
    virtual std::string Opreator() const = 0;
};

/**
 * Concrete Components provide default implementations of the operations.
 * There might be several variations of these classes.
 */
class ConcreteComponent : public Component {
public:
    std::string Operation() const override {
        return "ConcreteComponent";
    }
};

/**
 * The base Decorator class of follows the same interface as the other components.
 * The primary prupose of this class is to define the warpping interface for all
 * concrete decorators. The default implementation of the wrapping code might include 
 * a field for storing a wrapped component and the means to initialize it.
 */
class Decorator : public Component {
public:
    Decorator(Component* component) : component_ (component) {}
  
   /* The Decorator delegates all work to the wrapped component. */
   std::string Operation() const override {
      return this->component_->Operation();
    }

private:
    Component *component_;
};

/**
 * Concrete Decorators call the wrapped object and alter its result in some way.
 */
class ConcreteDecoratorA : public Decorator {
  /**
   * Decorators may call parent implementation of the operation, instead of
   * calling the wrapped object directly. This approach simplifies extension of
   * decorator classes.
   */
 public:
  ConcreteDecoratorA(Component* component) : Decorator(component) {
  }
  std::string Operation() const override {
    return "ConcreteDecoratorA(" + Decorator::Operation() + ")";
  }
};

/**
 * Decorators can execute their behavior either before or after the call to a
 * wrapped object.
 */
class ConcreteDecoratorB : public Decorator {
 public:
  ConcreteDecoratorB(Component* component) : Decorator(component) {
  }

  std::string Operation() const override {
    return "ConcreteDecoratorB(" + Decorator::Operation() + ")";
  }
};

/**
 * The client code works with all objects using the Component interface. This
 * way it can stay independent of the concrete classes of components it works
 * with.
 */
void ClientCode(Component* component) {
  // ...
  std::cout << "RESULT: " << component->Operation();
  // ...
}

int main() {
  /**
   * This way the client code can support both simple components...
   */
  Component* simple = new ConcreteComponent;
  std::cout << "Client: I've got a simple component:\n";
  ClientCode(simple);
  std::cout << "\n\n";
  /**
   * ...as well as decorated ones.
   *
   * Note how decorators can wrap not only simple components but the other
   * decorators as well.
   */
  Component* decorator1 = new ConcreteDecoratorA(simple);
  Component* decorator2 = new ConcreteDecoratorB(decorator1);
  std::cout << "Client: Now I've got a decorated component:\n";
  ClientCode(decorator2);
  std::cout << "\n";

  delete simple;
  delete decorator1;
  delete decorator2;

  return 0;
}
```

### 迭代器模式

亦称：Iterator

**==迭代器模式，是一种行为设计模式，让你能在不暴露集合底层表现形式（列表、栈和树等）的情况下，遍历集合中的所有元素。==**

**使用示例：** 该模式在 C++ 代码中很常见。 许多框架和程序库都使用它来提供遍历其集合的标准方式。

**识别方法：** 迭代器可以通过导航方法 （例如 `next`和 `previous`等） 来轻松识别。 使用迭代器的客户端代码可能没有其所遍历的集合的直接访问权限。

#### 适用场景

1. **当集合背后为复杂的数据结构，且你希望对客户端隐藏其复杂性时（出于使用便利性或安全性的考虑），可以使用迭代器模式**。
2. **使用该模式可以减少程序中重复的遍历代码**
3. **如果你希望代码能够遍历不同的甚至是无法预知的数据结构， 可以使用迭代器模式**

#### 与其他模式的关系

- 可以使用 **迭代器模式** 来遍历 **组合模式树**
- 同时使用**工厂方法** 和 **迭代器** 来让当前子类集合返回不同类型的迭代器，并使得迭代器与集合相匹配

#### 示例代码

```cpp
/**
 * Iterator Design Pattern
 *
 * Intent: Lets you traverse elements of a collection without exposing its
 * underlying representation (list, stack, tree, etc.).
 */

#include <iostream>
#include <string>
#include <vector>

/**
 * C++ has its own implementation of iterator that works with a different
 * generics containers defined by the standard library.
 */

template <typename T, typename U>
class Iterator {
 public:
  typedef typename std::vector<T>::iterator iter_type;
  Iterator(U *p_data, bool reverse = false) : m_p_data_(p_data) {
    m_it_ = m_p_data_->m_data_.begin();
  }

  void First() {
    m_it_ = m_p_data_->m_data_.begin();
  }

  void Next() {
    m_it_++;
  }

  bool IsDone() {
    return (m_it_ == m_p_data_->m_data_.end());
  }

  iter_type Current() {
    return m_it_;
  }

 private:
  U *m_p_data_;
  iter_type m_it_;
};

/**
 * Generic Collections/Containers provides one or several methods for retrieving
 * fresh iterator instances, compatible with the collection class.
 */

template <class T>
class Container {
  friend class Iterator<T, Container>;

 public:
  void Add(T a) {
    m_data_.push_back(a);
  }

  Iterator<T, Container> *CreateIterator() {
    return new Iterator<T, Container>(this);
  }

 private:
  std::vector<T> m_data_;
};

class Data {
 public:
  Data(int a = 0) : m_data_(a) {}

  void set_data(int a) {
    m_data_ = a;
  }

  int data() {
    return m_data_;
  }

 private:
  int m_data_;
};

/**
 * The client code may or may not know about the Concrete Iterator or Collection
 * classes, for this implementation the container is generic so you can used
 * with an int or with a custom class.
 */
void ClientCode() {
  std::cout << "________________Iterator with int______________________________________" << std::endl;
  Container<int> cont;

  for (int i = 0; i < 10; i++) {
    cont.Add(i);
  }

  Iterator<int, Container<int>> *it = cont.CreateIterator();
  for (it->First(); !it->IsDone(); it->Next()) {
    std::cout << *it->Current() << std::endl;
  }

  Container<Data> cont2;
  Data a(100), b(1000), c(10000);
  cont2.Add(a);
  cont2.Add(b);
  cont2.Add(c);

  std::cout << "________________Iterator with custom Class______________________________" << std::endl;
  Iterator<Data, Container<Data>> *it2 = cont2.CreateIterator();
  for (it2->First(); !it2->IsDone(); it2->Next()) {
    std::cout << it2->Current()->data() << std::endl;
  }
  delete it;
  delete it2;
}

int main() {
  ClientCode();
  return 0;
}
```

---

### 组合模式

亦称：对象树、Object Tree、Composite

**==组合模式，是一种结构型设计模式，可以使用它将对象组合成树状结构，并且能像适用独立对象一样使用他们==**

**使用实例：** 组合模式在 C++ 代码中很常见， 常用于表示与图形打交道的用户界面组件或代码的层次结构。

**识别方法：** 组合可以通过将同一抽象或接口类型的实例放入树状结构的行为方法来轻松识别。

#### 适用场景

1. **如果你需要实现树状对象结构，可以使用组合模式**
2. **如果你希望客户端代码以相同方式处理简单和复杂元素，可以使用该模式**

#### 示例代码

```cpp
#include <algorithm>
#include <iostream>
#include <list>
#include <string>
/**
 * The base Component class declares common operations for both simple and
 * complex objects of a composition.
 */
class Component {
  /**
   * @var Component
   */
 protected:
  Component *parent_;
  /**
   * Optionally, the base Component can declare an interface for setting and
   * accessing a parent of the component in a tree structure. It can also
   * provide some default implementation for these methods.
   */
 public:
  virtual ~Component() {}
  void SetParent(Component *parent) {
    this->parent_ = parent;
  }
  Component *GetParent() const {
    return this->parent_;
  }
  /**
   * In some cases, it would be beneficial to define the child-management
   * operations right in the base Component class. This way, you won't need to
   * expose any concrete component classes to the client code, even during the
   * object tree assembly. The downside is that these methods will be empty for
   * the leaf-level components.
   */
  virtual void Add(Component *component) {}
  virtual void Remove(Component *component) {}
  /**
   * You can provide a method that lets the client code figure out whether a
   * component can bear children.
   */
  virtual bool IsComposite() const {
    return false;
  }
  /**
   * The base Component may implement some default behavior or leave it to
   * concrete classes (by declaring the method containing the behavior as
   * "abstract").
   */
  virtual std::string Operation() const = 0;
};
/**
 * The Leaf class represents the end objects of a composition. A leaf can't have
 * any children.
 *
 * Usually, it's the Leaf objects that do the actual work, whereas Composite
 * objects only delegate to their sub-components.
 */
class Leaf : public Component {
 public:
  std::string Operation() const override {
    return "Leaf";
  }
};
/**
 * The Composite class represents the complex components that may have children.
 * Usually, the Composite objects delegate the actual work to their children and
 * then "sum-up" the result.
 */
class Composite : public Component {
  /**
   * @var \SplObjectStorage
   */
 protected:
  std::list<Component *> children_;

 public:
  /**
   * A composite object can add or remove other components (both simple or
   * complex) to or from its child list.
   */
  void Add(Component *component) override {
    this->children_.push_back(component);
    component->SetParent(this);
  }
  /**
   * Have in mind that this method removes the pointer to the list but doesn't
   * frees the
   *     memory, you should do it manually or better use smart pointers.
   */
  void Remove(Component *component) override {
    children_.remove(component);
    component->SetParent(nullptr);
  }
  bool IsComposite() const override {
    return true;
  }
  /**
   * The Composite executes its primary logic in a particular way. It traverses
   * recursively through all its children, collecting and summing their results.
   * Since the composite's children pass these calls to their children and so
   * forth, the whole object tree is traversed as a result.
   */
  std::string Operation() const override {
    std::string result;
    for (const Component *c : children_) {
      if (c == children_.back()) {
        result += c->Operation();
      } else {
        result += c->Operation() + "+";
      }
    }
    return "Branch(" + result + ")";
  }
};
/**
 * The client code works with all of the components via the base interface.
 */
void ClientCode(Component *component) {
  // ...
  std::cout << "RESULT: " << component->Operation();
  // ...
}

/**
 * Thanks to the fact that the child-management operations are declared in the
 * base Component class, the client code can work with any component, simple or
 * complex, without depending on their concrete classes.
 */
void ClientCode2(Component *component1, Component *component2) {
  // ...
  if (component1->IsComposite()) {
    component1->Add(component2);
  }
  std::cout << "RESULT: " << component1->Operation();
  // ...
}

/**
 * This way the client code can support the simple leaf components...
 */

int main() {
  Component *simple = new Leaf;
  std::cout << "Client: I've got a simple component:\n";
  ClientCode(simple);
  std::cout << "\n\n";
  /**
   * ...as well as the complex composites.
   */

  Component *tree = new Composite;
  Component *branch1 = new Composite;

  Component *leaf_1 = new Leaf;
  Component *leaf_2 = new Leaf;
  Component *leaf_3 = new Leaf;
  branch1->Add(leaf_1);
  branch1->Add(leaf_2);
  Component *branch2 = new Composite;
  branch2->Add(leaf_3);
  tree->Add(branch1);
  tree->Add(branch2);
  std::cout << "Client: Now I've got a composite tree:\n";
  ClientCode(tree);
  std::cout << "\n\n";

  std::cout << "Client: I don't need to check the components classes even when managing the tree:\n";
  ClientCode2(tree, simple);
  std::cout << "\n";

  delete simple;
  delete tree;
  delete branch1;
  delete branch2;
  delete leaf_1;
  delete leaf_2;
  delete leaf_3;

  return 0;
}
```

---

### 享元模式

亦称：缓存、Cache、Flyweight

享元模式，是一种结构型设计模式，它摒弃了在每个对象种保存所有数据的方式，通过共享多个对象所共有的相同状态，让你能在有限的内存容量种载入更多对象。

**使用示例：** 享元模式只有一个目的： 将内存消耗最小化。 如果你的程序没有遇到内存容量不足的问题， 则可以暂时忽略该模式。

**识别方法：** 享元可以通过构建方法来识别， 它会返回缓存对象而不是创建新的对象。

#### 适用场景

 **仅在程序必须支持大量对象且没有足够的内存容量时使用享元模式**

在下列情况中最有效：

- 程序需要生成数量巨大的相似对象
- 这将耗尽目标设备的所有内存
- 对象中包含可抽取且能在多个对象间共享的重复状态。

#### 与其他模式的关系

- 如果你能将对象的所有共享状态简化为一个享元对象， 那么**享元**就和**单例模式**类似了； 但这两个模式有两个根本性的不同。

  1. **只会有一个单例实体， 但是*享元*类可以有多个实体， 各实体的内在状态也可以不同。**
  2. ***单例*对象可以是可变的。 享元对象是不可变的。**

#### 示例代码

```cpp
/**
 * Flyweight Design Pattern
 *
 * Intent: Lets you fit more objects into the available amount of RAM by sharing
 * common parts of state between multiple objects, instead of keeping all of the
 * data in each object.
 */

struct SharedState
{
    std::string brand_;
    std::string model_;
    std::string color_;

    SharedState(const std::string &brand, const std::string &model, const std::string &color)
        : brand_(brand), model_(model), color_(color)
    {
    }

    friend std::ostream &operator<<(std::ostream &os, const SharedState &ss)
    {
        return os << "[ " << ss.brand_ << " , " << ss.model_ << " , " << ss.color_ << " ]";
    }
};

struct UniqueState
{
    std::string owner_;
    std::string plates_;

    UniqueState(const std::string &owner, const std::string &plates)
        : owner_(owner), plates_(plates)
    {
    }

    friend std::ostream &operator<<(std::ostream &os, const UniqueState &us)
    {
        return os << "[ " << us.owner_ << " , " << us.plates_ << " ]";
    }
};

/**
 * The Flyweight stores a common portion of the state (also called intrinsic
 * state) that belongs to multiple real business entities. The Flyweight accepts
 * the rest of the state (extrinsic state, unique for each entity) via its
 * method parameters.
 */
class Flyweight
{
private:
    SharedState *shared_state_;

public:
    Flyweight(const SharedState *shared_state) : shared_state_(new SharedState(*shared_state))
    {
    }
    Flyweight(const Flyweight &other) : shared_state_(new SharedState(*other.shared_state_))
    {
    }
    ~Flyweight()
    {
        delete shared_state_;
    }
    SharedState *shared_state() const
    {
        return shared_state_;
    }
    void Operation(const UniqueState &unique_state) const
    {
        std::cout << "Flyweight: Displaying shared (" << *shared_state_ << ") and unique (" << unique_state << ") state.\n";
    }
};
/**
 * The Flyweight Factory creates and manages the Flyweight objects. It ensures
 * that flyweights are shared correctly. When the client requests a flyweight,
 * the factory either returns an existing instance or creates a new one, if it
 * doesn't exist yet.
 */
class FlyweightFactory
{
    /**
     * @var Flyweight[]
     */
private:
    std::unordered_map<std::string, Flyweight> flyweights_;
    /**
     * Returns a Flyweight's string hash for a given state.
     */
    std::string GetKey(const SharedState &ss) const
    {
        return ss.brand_ + "_" + ss.model_ + "_" + ss.color_;
    }

public:
    FlyweightFactory(std::initializer_list<SharedState> share_states)
    {
        for (const SharedState &ss : share_states)
        {
            this->flyweights_.insert(std::make_pair<std::string, Flyweight>(this->GetKey(ss), Flyweight(&ss)));
        }
    }

    /**
     * Returns an existing Flyweight with a given state or creates a new one.
     */
    Flyweight GetFlyweight(const SharedState &shared_state)
    {
        std::string key = this->GetKey(shared_state);
        if (this->flyweights_.find(key) == this->flyweights_.end())
        {
            std::cout << "FlyweightFactory: Can't find a flyweight, creating new one.\n";
            this->flyweights_.insert(std::make_pair(key, Flyweight(&shared_state)));
        }
        else
        {
            std::cout << "FlyweightFactory: Reusing existing flyweight.\n";
        }
        return this->flyweights_.at(key);
    }
    void ListFlyweights() const
    {
        size_t count = this->flyweights_.size();
        std::cout << "\nFlyweightFactory: I have " << count << " flyweights:\n";
        for (std::pair<std::string, Flyweight> pair : this->flyweights_)
        {
            std::cout << pair.first << "\n";
        }
    }
};

// ...
void AddCarToPoliceDatabase(
    FlyweightFactory &ff, const std::string &plates, const std::string &owner,
    const std::string &brand, const std::string &model, const std::string &color)
{
    std::cout << "\nClient: Adding a car to database.\n";
    const Flyweight &flyweight = ff.GetFlyweight({brand, model, color});
    // The client code either stores or calculates extrinsic state and passes it
    // to the flyweight's methods.
    flyweight.Operation({owner, plates});
}

/**
 * The client code usually creates a bunch of pre-populated flyweights in the
 * initialization stage of the application.
 */

int main()
{
    FlyweightFactory *factory = new FlyweightFactory({{"Chevrolet", "Camaro2018", "pink"}, {"Mercedes Benz", "C300", "black"}, {"Mercedes Benz", "C500", "red"}, {"BMW", "M5", "red"}, {"BMW", "X6", "white"}});
    factory->ListFlyweights();

    AddCarToPoliceDatabase(*factory,
                            "CL234IR",
                            "James Doe",
                            "BMW",
                            "M5",
                            "red");

    AddCarToPoliceDatabase(*factory,
                            "CL234IR",
                            "James Doe",
                            "BMW",
                            "X1",
                            "red");
    factory->ListFlyweights();
    delete factory;

    return 0;
}
```

---

## CMake

### CMakeLists.txt 模板

```cmake
cmake_minimum_required(VERSION 3.20)

set(CMAKE_CXX_STANDARD 11) # C++ 标准
set(CMAKE_CXX_STANDARD_REQUIRED ON) # OFF(默认) 则 CMake 检测到编译器不支持 C++17 时不报错，而是默默调低到 C++14 给你用；为 ON 则发现不支持报错，更安全
set(CMAKE_CXX_EXTENSIONS OFF) # ON(默认) 表示启用 GCC 特有的一些扩展功能；OFF 则关闭 GCC 的扩展功能，只使用标准的 C++(兼容 MSVC)
set(CMAKE_CXX_COMPILE_FEATURES OFF)

set(PROJECT_NAME cppTemplate)
project(${PROJECT_NAME} VERSION 0.1.0)

if (PROJECT_BINARY_DIR STREQUAL PROJECT_SOURCE_DIR)
    message(WARNING "This binary directory of CMake cannot be the same as source directory!")
endif()

# Release: -O3 -DNDEBUG
# Debug: -O0 -g
# MinSizeRel: -Os -DNDEBUG 
# RelWithDebInfo: -O2 -g -DNDEBUG
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release")
endif()

if(WIN32)
    add_definitions(-DNOMINMAX -D_USE_MATH_DEFINES)
endif()

if (NOT MSVC) 
    find_program(CCACHE_PROGRAM ccache)
    if (CCACHE_PROGRAM)
        message(STATUS "Found CCache: ${CCACHE_PROGRAM}")
        set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE ${CCACHE_PROGRAM})
        set_property(GLOBAL PROPERTY RULE_LAUNCH_LINK ${CCACHE_PROGRAM})
    endif()
endif()

# src
file(GLOB_RECURSE srcs CONFIGURE_DEPENDS src/*.cc src/*.cpp inc/*.h)
# add_library(${PROJECT_NAME} SHARED ${srcs})
add_executable(${PROJECT_NAME} ${srcs})

find_package(TBB REQUIRED)
target_link_libraries(${PROJECT_NAME} PUBLIC pthread
                        PUBLIC TBB::tbb)
target_include_directories(${PROJECT_NAME} PUBLIC inc)

# sub
# add_subdirectory()

# test
if (CMAKE_BUILD_TYPE STREQUAL "Debug")
    message(STATUS "Enable test")
    add_subdirectory(test)
endif()
```

### [对象库 ](https://www.scivision.dev/cmake-object-libraries/)

对象库类似于静态库，但不生成 .a 文件，只由 CMake 记住该库生成了哪些对象文件，对象库是 CMake 自创的，绕开了编译器和操作系统的各种繁琐规则，保证了跨平台统一性。

在自己的项目中，推荐全部用对象库(OBJECT)替代静态库(STATIC)避免跨平台的麻烦。对象库仅仅作为组织代码的方式，而实际生成的可执行文件只有一个，减轻了部署的困难。

```cmake
add_library(Lute OBJECT Lute.cc)
add_executable(main main.cc)
target_link_libraries(main PUBLIC Lute)
```

- **静态库的麻烦：GCC 编译器自作聪明，会自动剔除没有引用符号的那些对象**
- **对象库可以绕开编译器的不统一：保证不会自动剔除没引用到的对象文件**
- **动态库虽然可以避免剔除没引用的对象文件，但引入了运行时链接的麻烦**

### 坑点：动态库无法链接静态库

```cmake
add_library(otherlib STATIC otherlib.cc)

add_library(Lute SHARED Lute.cc)
target_link_library(Lute PUBLIC otherlib)

add_executable(main main.cc)
target_link_library(main PUBLIC Lute)
```

> 解决：让静态库编译时也生成位置无关的代码(PIC)，这样才能装在动态库里
>
> `set(CMAKE_POSITION_INDEPENDENT_CODE ON)`
>
> `set_property(TARGET otherlib PROPERTY POSITION_INDEPENDENT_CODE ON)`

### CCache 编译加速缓存

用法：

- 把 `gcc -c main.cpp -o main` 换成 `ccache gcc -c main.cpp -o main` 即可

- 在 CMake 中可以这样来启用 ccache（就是给每个编译和链接命令前面加上 ccache）

  ```cmake
  if (NOT MSVC) 
      find_program(CCACHE_PROGRAM ccache)
      if (CCACHE_PROGRAM)
          message(STATUS "Found CCache: ${CCACHE_PROGRAM}")
          set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE ${CCACHE_PROGRAM})
          set_property(GLOBAL PROPERTY RULE_LAUNCH_LINK ${CCACHE_PROGRAM})
      endif()
  endif()
  ```

**不过好像不支持 MSVC 的样子**

## 网络通信故障排查常用命令

#### `ifconfig` 命令

`ifconfig`  命令，是查看当前系统的网卡和 IP 地址信息的常用命令

- 安装：`yay -S net-tools`

- 启用 / 禁用某个网卡：

    `ifconfig Name up / down`

- 经某个 IP 地址绑定到某个网卡上，或将某个地址从某个网卡上解绑

    ```bash
    ifconfig Name add IPAddr
    ifconfig Name del IPAddr
    ```

#### `ping` 命令

`ping` 命令，一般用于检测本机到目标主机的网络是否通畅。

- 使用：`ping ipAddr`
- 实现：发送 ICMP 数据包

#### `netstat` 命令

`netstat` 命令，常用于查看网络连接状态

常用选项：

- `-a` 表示显示所有选项，不使用该选项时， `netstat` 默认不显示 LISTEN 相关选项
- `-t` 表示仅显示 TCP 相关选项
- `-u` 仅显示 UDP 相关选项
- `-n` 不显示别名，将能显示数字的全部转换为数字
- `-l` 仅列出处于监听（LISTEN）状态的服务
- `-p` 显示建立相关连接的程序名
- `-r` 显示路由信息、路由表

#### `lsof` 命令

**==lsof，list opened file descriptor，列出已经打开的文件描述符==**，该命令是 Linux 的扩展命令；

#### `nc` 命令

**==nc，netcat==**，常用于模拟一个服务器程序被其他客户端连接，或者模拟一个客户端连接其他服务器，连接之后就可以进行数据收发；

- **默认使用 TCP 协议，加上 `-u` 选项使用 UDP 协议**

- **默认使用 `\n` 作为每条消息的结尾；使用 `-C` 选项，则使用 `\r\n` 作为消息结束的标志**

- `nc` 发送、接受文件

    ```bash
    # 接收文件
    nc -l IP PORT > file
    
    # Send File
    nc IP PORT < file
    ```

#### `curl` 命令

**==`curl` 在 Linux 上可以模拟发送 HTTP 请求==**

- 基础用法：`curl http://www.baidu.com`， 默认行为是**将目标页面的内容输出到 shell 窗口**

- 可以使用 `-X` 选项显示执行请求采用的 GET 或 POST 请求（不指定时，默认采用 GET 方式）

    ```bash
    curl -X GET http://www.baidu.com
    curl -G http://www.baidu.com
    
    curl -X POST -d 'somepostdata' 'http://www.baidu.com'
    ```

- `-d` / `--data` 选项，指定 POST 的数据内容

- `-H` / `--head` 选项，指定 HTTP 的头部信息

    ```bash
    curl -H "Content-Type: application/json" http://example.com
    ```

- `-O`选项将下载文件并将其保存为与远程服务器上的文件名相同的文件，而`-o`选项将下载文件并将其保存为指定的本地文件名

    ```bash
    curl -O http://example.com/file.txt
    curl -o localfile.txt http://example.com/file.txt
    ```

#### `tcpdump` 命令

**==tcpdump==，是 Linux 提供的一个非常强大的抓包工具**

- `-i` 选项，指定要捕获的目标网卡名；如果需要抓取所有网卡上的包，则使用 `any` 关键字

    ```bash
    sudo tcpdump -i eno1
    sudo tcpdump -i any
    ```

- `-X` 选项，以 ASCII 和十六进制形式输出捕获的数据包内容，减去链路层的包头信息；

- `-XX` 选项，以 ASCII 和十六进制形式输出捕获的数据包内容，包括链路层的包头信息；

- `-n` ，不要将 IP 地址显示成别名；

- `-nn`，不要将 IP 地址、端口显示成别名；

- 支持各种数据包过滤的表达式：

    ```bash
    tcpdump -i any 'port 5883'
    tcpdump -i any 'tcp port 5883'
    tcpdump -i any 'tcp src port 5883'
    tcpdump -i any 'tcp src port 5883 or udp port 5883'
    tcpdump -i any 'src host 127.0.0.1 and tcp src port 5883' -XX -nn -vv
    ```
