# epoll 编程

epoll 是 Linux 提供的高效 I/O 多路复用接口。它围绕一个 epoll 实例工作：程序把需要监视的文件描述符加入关注列表，内核在事件就绪时把对应信息返回给 `epoll_wait()`。

# epoll 的三个核心接口

## epoll_create1：创建实例

```c
int epoll_create1(int flags);
```

成功返回 epoll 实例的文件描述符，失败返回 `-1`。通常使用：

```c
int epfd = epoll_create1(EPOLL_CLOEXEC);
```

`EPOLL_CLOEXEC` 会设置 close-on-exec，避免程序调用 `exec` 后把 epoll fd 泄漏给新程序。旧接口 `epoll_create(size)` 的 `size` 在现代内核中已不再决定容量，只要求大于 0，新代码优先使用 `epoll_create1`。

## epoll_ctl：维护关注列表

```c
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```

`op` 决定操作类型：

- `EPOLL_CTL_ADD`：添加新的 fd 和事件。
- `EPOLL_CTL_MOD`：修改已注册 fd 的事件。
- `EPOLL_CTL_DEL`：移除 fd；此时 `event` 可以传 `NULL`。

常见的添加方式：

```c
struct epoll_event ev = {0};
ev.events = EPOLLIN | EPOLLRDHUP;
ev.data.fd = client_fd;

if (epoll_ctl(epfd, EPOLL_CTL_ADD, client_fd, &ev) == -1) {
    perror("epoll_ctl");
}
```

## epoll_wait：等待事件

```c
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

- `events`：由程序提供的数组，用于接收就绪事件。
- `maxevents`：数组最多能容纳多少项，必须大于 0。
- `timeout=-1`：一直等待；`0`：立即返回；正数：最多等待相应毫秒数。
- 返回正数表示就绪事件个数，返回 `0` 表示超时，返回 `-1` 表示失败。

若被信号中断，`epoll_wait` 可能返回 `-1` 且 `errno == EINTR`，事件循环通常重新等待或检查退出标志。

# struct epoll_event

```c
typedef union epoll_data {
    void     *ptr;
    int       fd;
    uint32_t  u32;
    uint64_t  u64;
} epoll_data_t;

struct epoll_event {
    uint32_t     events;
    epoll_data_t data;
};
```

`events` 是事件位掩码，`data` 是注册时交给内核、事件发生后原样返回的用户数据。简单程序可以保存 `data.fd`；复杂服务器常保存连接对象指针 `data.ptr`，把缓冲区、协议状态和超时信息放在连接对象中。

联合体中的成员共享同一块内存，设置 `data.ptr` 后不能再指望 `data.fd` 同时有效。

# 常用事件

以下事件即为`struct epoll_event`的`events`的值，即感兴趣的事件。

- `EPOLLIN`：可以读取；对监听套接字表示可以接受连接。
- `EPOLLOUT`：可以写入。套接字大多数时间都可写，不应无条件长期关注。
- `EPOLLRDHUP`：TCP 对端关闭写方向，便于发现半关闭。
- `EPOLLERR`：描述符出现错误；即使未显式注册也可能返回。
- `EPOLLHUP`：描述符发生挂断；即使未显式注册也可能返回。
- `EPOLLET`：使用边沿触发模式。
- `EPOLLONESHOT`：事件返回一次后暂时禁用，重新 `MOD` 后才继续监听，常用于避免多个工作线程同时处理同一连接。

发生 `EPOLLERR`、`EPOLLHUP` 时不能只忽略事件继续循环，应检查套接字错误并清理连接。对于同时包含 `EPOLLIN` 的事件，还要考虑缓冲区中可能仍有未读取数据。

# LT 模式的基本事件循环

下面展示核心结构，省略了创建监听套接字时的 `socket`、`bind` 和 `listen`：

```c
#include <errno.h>
#include <stdio.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <unistd.h>

#define MAX_EVENTS 64

int run_loop(int listen_fd)
{
    int epfd = epoll_create1(EPOLL_CLOEXEC);
    if (epfd == -1) {
        return -1;
    }

    struct epoll_event ev = {
        .events = EPOLLIN,
        .data.fd = listen_fd
    };
    if (epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ev) == -1) {
        close(epfd);
        return -1;
    }

    struct epoll_event ready[MAX_EVENTS];

    for (;;) {
        int n = epoll_wait(epfd, ready, MAX_EVENTS, -1);
        if (n == -1) {
            if (errno == EINTR) {
                continue;
            }
            break;
        }

        for (int i = 0; i < n; ++i) {
            int fd = ready[i].data.fd;

            if (fd == listen_fd) {
                int client_fd = accept(listen_fd, NULL, NULL);
                if (client_fd == -1) {
                    continue;
                }

                struct epoll_event client_ev = {
                    .events = EPOLLIN | EPOLLRDHUP,
                    .data.fd = client_fd
                };
                if (epoll_ctl(epfd, EPOLL_CTL_ADD,
                              client_fd, &client_ev) == -1) {
                    close(client_fd);
                }
                continue;
            }

            if ((ready[i].events & (EPOLLERR | EPOLLHUP)) &&
                !(ready[i].events & EPOLLIN)) {
                epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
                close(fd);
                continue;
            }

            char buf[1024];
            ssize_t count = recv(fd, buf, sizeof(buf), 0);
            if (count > 0) {
                /* 处理 buf[0..count-1]，不能假设它正好是一条完整消息 */
            } else if (count == 0) {
                epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
                close(fd);
            } else if (errno != EINTR) {
                epoll_ctl(epfd, EPOLL_CTL_DEL, fd, NULL);
                close(fd);
            }
        }
    }

    close(epfd);
    return -1;
}
```

这是便于理解的 LT 示例，不是完整生产服务器。它一次只 `accept` 一个连接、一次只 `recv` 一次；LT 模式下若仍有连接或数据，epoll 会再次通知。实际程序还要处理应用层消息边界、发送缓冲、超时和优雅退出。

# ET 模式必须处理到 EAGAIN

边沿触发只在状态变化时通知。监听套接字和连接套接字都应设置为非阻塞，并在收到事件后把当前工作处理干净。

设置非阻塞：

```c
#include <fcntl.h>

int set_nonblocking(int fd)
{
    int flags = fcntl(fd, F_GETFL, 0);
    if (flags == -1) {
        return -1;
    }
    return fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}
```

处理可读连接：

```c
for (;;) {
    ssize_t n = recv(fd, buf, sizeof(buf), 0);

    if (n > 0) {
        /* 处理本次读到的数据 */
    } else if (n == 0) {
        /* 对端关闭连接 */
        close_client(fd);
        break;
    } else if (errno == EINTR) {
        continue;
    } else if (errno == EAGAIN || errno == EWOULDBLOCK) {
        break;  // 当前数据已经读完，回到 epoll_wait
    } else {
        close_client(fd);
        break;
    }
}
```

监听套接字在 ET 模式下也要循环 `accept`，直到返回 `EAGAIN`。否则同时到达的连接可能留在已完成连接队列中，却因为没有新的状态变化而暂时得不到再次通知。Linux 中也可以使用 `accept4(..., SOCK_NONBLOCK | SOCK_CLOEXEC)` 直接创建非阻塞且 close-on-exec 的连接 fd。

# 正确管理写事件

非阻塞 `send()` 可能只发送部分数据，也可能因发送缓冲区已满返回 `EAGAIN`。程序需要保存未发送部分：

1. 立即尝试发送。
2. 若只发送一部分或遇到 `EAGAIN`，保存剩余数据，并通过 `EPOLL_CTL_MOD` 增加 `EPOLLOUT`。
3. 收到可写事件后继续发送。
4. 缓冲区清空后移除 `EPOLLOUT`，保留需要的读事件。

如果始终注册 `EPOLLOUT`，空闲套接字也会不断被报告为可写，造成事件循环高频唤醒和 CPU 空转。

# 连接关闭与资源清理

关闭连接时应统一完成：

- 从 epoll 关注列表删除 fd；
- 关闭 fd；
- 释放接收、发送缓冲区和连接状态；
- 取消与连接关联的定时器。

Linux 在最后一个引用关闭时会自动把文件描述移出 epoll，但显式 `EPOLL_CTL_DEL` 能让资源生命周期更清楚。还要注意 fd 是可复用的整数编号，不能让旧事件继续引用已经释放的连接对象。

# 常见错误

- 把“收到一次 `EPOLLIN`”理解为“收到一条完整消息”。TCP 是字节流，需要应用层协议处理粘包和拆包。
- ET 模式只读一次，没有循环到 `EAGAIN`，导致剩余数据长时间得不到处理。
- 对阻塞 fd 使用 ET，循环中的 `recv` 或 `accept` 最终阻塞整个事件线程。
- 永久监听 `EPOLLOUT`，造成 CPU 占用过高。
- 忽略 `recv==0`、`EPOLLRDHUP`、`EPOLLERR` 和 `EPOLLHUP`，连接资源无法及时释放。
- 事件处理函数执行耗时业务，拖慢所有连接。

# 小结

- `epoll_create1` 创建实例，`epoll_ctl` 管理关注列表，`epoll_wait` 返回就绪事件。
- `epoll_event.data` 用于保存 fd 或连接对象指针，联合体成员不能同时使用。
- LT 容易使用；ET 必须结合非阻塞 I/O，并对 `accept`、`recv`、`send` 处理到 `EAGAIN`。
- 写事件只应在存在待发送数据时注册，连接关闭要统一注销并释放状态。
