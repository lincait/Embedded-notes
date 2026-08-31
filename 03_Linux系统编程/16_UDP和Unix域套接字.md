# UDP 编程与 Unix 域套接字

# UDP 协议与 TCP 的对比

UDP（用户数据报协议）是无连接的传输层协议。它和 TCP 的核心差异可以这样理解：TCP 像挂号信——有回执、按顺序、丢了会补发；UDP 像平邮明信片——寄出去就完事，不保证到达，也不保证顺序。

| 对比项 | TCP | UDP |
| --- | --- | --- |
| 连接 | 面向连接，需三次握手 | 无连接，直接发 |
| 可靠性 | 可靠，有确认重传 | 不可靠，可能丢包乱序 |
| 数据单位 | 字节流（无边界） | 数据报（有边界，一条一条） |
| 传输效率 | 较低（握手、确认开销大） | 较高（无额外开销） |
| 适用场景 | 文件传输、网页、需要准确数据 | 视频会议、实时游戏、广播 |

UDP 报文段格式比 TCP 简单得多：只有源端口、目标端口、长度、校验和加数据，没有序号、确认号、控制位这些字段——所以它做不到排序和确认，也正因为如此它"轻"。

# UDP 通信流程

UDP 没有握手步骤，所以编程流程比 TCP 简单：

- 服务端 `bind` 绑定地址后，**不需要 `listen`**，**不需要 `accept`**，直接收发消息。
- 客户端**不需要 `connect`**，直接向对方地址发消息。
- 由于没有握手，两端都无法确认对方是否收到消息——这就是 UDP 不可靠的体现。

UDP 收发数据不用 `send`/`recv`（这两个一般用于 TCP 连接），而用 `sendto`/`recvfrom`。它们比 `send`/`recv` 多了一对地址参数，因为无连接状态下，每次发送都要指定目标地址，每次接收都要知道消息从哪来：

```c
ssize_t sendto(int sockfd, const void *buf, size_t len, int flags,
               const struct sockaddr *dest_addr, socklen_t addrlen);
ssize_t recvfrom(int sockfd, void *buf, size_t len, int flags,
                 struct sockaddr *src_addr, socklen_t *addrlen);
```

- `sendto`：`dest_addr` 是目标地址，每次发送都可以发给不同的主机。
- `recvfrom`：`src_addr` 返回消息来源的地址，不关心就填 NULL；`addrlen` 调用前填缓冲区大小，调用后被更新为实际地址大小。

其余参数（sockfd、buf、len、flags）与 `send`/`recv` 相同。

# UDP 编程流程

**服务端：**

1. `socket(AF_INET, SOCK_DGRAM, 0)` 创建数据报套接字。
2. 填写 `sockaddr_in`（地址族、端口），`bind()` 绑定地址。
3. 直接 `recvfrom` 接收数据、`sendto` 发送数据（`sendto` 的目标地址取自 `recvfrom` 返回的 `src_addr`）。
4. 结束 `close()`。

**客户端：**

1. `socket(AF_INET, SOCK_DGRAM, 0)` 创建套接字。
2. 直接 `sendto` 向服务端地址发数据，`recvfrom` 收数据。
3. 结束 `close()`。

对比 TCP 可见，UDP 的收发函数每次都要带着对方的地址走，代价是编程简单、延迟低。

# Unix Domain Socket

## 简介

Socket 最初是为网络服务设计的，后来发展成一种进程间通信方式：**Unix Domain Socket（Unix 域套接字）**。它允许同一台主机上的进程高效通信，数据**不经过网络协议栈**，直接在操作系统内部传递，因此延迟低、性能高。

它通过文件系统中的一个特殊文件（套接字文件）来标识通信端点，进程通过这个文件完成双向数据传输，常用于进程间通信和客户端-服务器模型。

## 地址结构体

Unix 域套接字用 `struct sockaddr_un` 表示地址，地址就是一个文件系统路径名：

```c
struct sockaddr_un {
    sa_family_t sun_family;   // 填 AF_UNIX
    char        sun_path[108];// 文件路径名
};
```

要点：

- `sun_family` 必须填 `AF_UNIX`。
- `sun_path` 是套接字文件路径，路径名要以空字符结尾，总长度不能超过 `sun_path` 的大小。
- 传入 `addrlen` 时填 `sizeof(struct sockaddr_un)`。
- 套接字文件遵循所在目录的权限：进程必须对创建目录有写和搜索权限才能创建；连接流套接字还需要对它有写权限。
- 用完要 `unlink` 删除套接字文件，否则文件会残留。

## 与网络套接字的区别

| 对比项 | AF_INET 网络套接字 | AF_UNIX 域套接字 |
| --- | --- | --- |
| 通信范围 | 不同主机之间 | 同一台主机 |
| 地址形式 | IP 地址 + 端口 | 文件系统路径名 |
| 数据路径 | 经过网络协议栈 | 直接内核内部传递 |
| 效率 | 较低 | 较高 |

## 编程流程

Unix 域套接字的编程流程和 TCP 几乎一样，只是地址结构体和创建时的通信域不同：

- 服务端：`socket(AF_UNIX, SOCK_STREAM, 0)` → 填写 `sockaddr_un` → `bind` → `listen` → `accept` → `recv`/`send` → 结束 `unlink` 套接字文件。
- 客户端：`socket(AF_UNIX, SOCK_STREAM, 0)` → 填写对方 `sockaddr_un` → `connect` → `recv`/`send` → 结束 `close`。

如果不需要双向且确定的一对一通信，还有更轻量的 `socketpair` 可以直接创建一对已连接的匿名套接字，常用于父子进程之间。

# 小结

- UDP 无连接、不可靠、有消息边界，用 `sendto`/`recvfrom` 收发，服务端绑定后即可直接通信，无需 listen/accept。
- Unix 域套接字是本机进程间通信的高效方式，地址是文件路径，流程与 TCP 编程一致，只是通信域换成 `AF_UNIX`、地址结构体换成 `sockaddr_un`。
