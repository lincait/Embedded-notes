# I/O 内存映射和用户数据拷贝

教材第四篇第 41 章用 I.MX6U-ALPHA 板上的 LED 说明寄存器怎么访问。应用调用 `write()` 后，驱动先把用户缓冲区里的命令拷入内核，再按命令修改 GPIO 寄存器。

## I/O 内存映射

芯片手册给出的 `GPIO1_DR` 地址是物理地址 `0x0209c000`。Linux 驱动需要先为这 4 字节的寄存器建立地址映射，再用映射后的地址访问它。以下接口见 `<linux/io.h>`。

### ioremap

```c
void __iomem *ioremap(phys_addr, size)
```

把指定的外设物理地址映射到内核地址空间。教材 ARM 内核中 `ioremap` 是宏，上面写的是调用形式。

- `phys_addr` 是起始物理地址。`size` 是要映射的字节数。
- 成功返回一个 `void __iomem *`，失败返回 `NULL`。例如用 `void __iomem *dr = ioremap(0x0209c000, 4)` 取得 `GPIO1_DR` 的访问地址。
- 后续用 `readl(dr)` 时，CPU 将该虚拟地址转换到物理地址 `0x0209c000`，读到 GPIO1 的数据寄存器。一般在初始化时映射一次，之后反复使用同一个 `dr`。

### iounmap

```c
void iounmap(volatile void __iomem *addr);
```

解除已经建立的 I/O 地址映射。`addr` 是 `ioremap()` 成功返回的地址，例如 `iounmap(dr)`。这个函数无返回值，通常在驱动退出或初始化失败时调用。

## 读写寄存器

按芯片手册规定的宽度选择接口。`b`、`w`、`l` 分别对应 8、16、32 位。教材 ARM 内核中这些接口由宏实现，下方按参数和返回类型写成原型，便于对照用法。

### readb

```c
u8 readb(const volatile void __iomem *addr);
```

从映射后的地址 `addr` 读取一个 8 位寄存器，返回读出的 `u8` 值。`readw`和`readl`同理。

### writeb

```c
void writeb(u8 value, volatile void __iomem *addr);
```

将 8 位值 `value` 写到 I/O 地址 `addr` 所指的寄存器，无返回值。`writew`和`writel`同理。

## GPIO1_DR 为什么要先读再写

ALPHA 板的 LED 接 `GPIO1_IO03`，低电平点亮。要让下面的写寄存器操作控制 LED，先打开 GPIO1 控制器的时钟，再把引脚选为 GPIO 功能，并把 GPIO1 的第 3 路设为输出。此时改 `GPIO1_DR` 的第 3 位，就能改这一路的输出电平。

### BIT

```c
BIT(nr)
```

产生只有一位为 1 的掩码。`nr` 是位号，从最低位的 0 开始。例如 `BIT(3)` 得到 `0x08`，只有第 3 位为 1。

### 读改写第 3 位

```c
u32 value = readl(dr);
value &= ~BIT(3);
writel(value, dr);
```

- GPIO1 的多路输出共用一个数据寄存器，所以先读出所有位，再只改控制 LED 的那一位。例如读到 `0x0a` 时，第 3 位和第 1 位都是 1。`value &= ~BIT(3)` 得到 `0x02`，只清掉第 3 位。写回后 LED 点亮，第 1 路仍保持高电平。熄灭时用 `value |= BIT(3)` 把第 3 位设为 1。
- 如果两段代码同时读到旧值，各自改一位再写回，后写的一方可能覆盖先写的一方。需要让所有修改这个寄存器的代码按同一套规则串行执行，比如使用同一把锁。有写 1 清零等特殊规则的寄存器要按手册要求写入。
- 这段写法对应教材的直接寄存器实验。由 GPIO 控制器驱动管理的寄存器应通过[GPIO 子系统接口](05_用GPIO控制LED.md)操作。

## 用户空间和内核空间的数据拷贝

教材 ARM 内核的接口见 `<asm/uaccess.h>`。两个函数都是**目标地址在前，源地址在后**。`__user` 标出用户空间指针。拷贝可能需要等用户页面调入物理内存，应在应用触发的读写回调等允许等待的地方调用。

### copy_from_user

```c
unsigned long copy_from_user(void *to, const void __user *from,
                             unsigned long n);
```

把应用的数据拷到内核，常在字符设备的 `.write` 中使用。内核通过这个接口处理读取用户地址时可能发生的访问异常。

- `to` 指向内核缓冲区，`from` 指向应用的数据，`n` 为要拷贝的字节数。例如 `copy_from_user(&cmd, buf, 1)` 从应用的 `buf` 取一个命令字节放进 `cmd`。传入的长度应在内核缓冲区容量之内。
- 返回**没拷到的字节数**。请求拷贝 4 字节却返回 1，表示还有 1 字节没拷到。返回 0 表示全部拷完。

### copy_to_user

```c
unsigned long copy_to_user(void __user *to, const void *from,
                           unsigned long n);
```

把内核中的数据交给应用，常在字符设备的 `.read` 中使用。

- `to` 指向应用的接收缓冲区，`from` 指向内核的数据，`n` 为要拷贝的字节数。例如 `copy_to_user(buf, &state, 1)` 将内核里的一字节状态 `state` 写给应用。传入的长度要同时符合内核已有的数据量和应用请求的长度。
- 返回没拷到的字节数，返回 0 表示全部拷完。例如完整拷贝 1 字节时，`copy_to_user()` 返回 0，`.read` 给应用返回实际读到的字节数 1。

## 一个字节怎样变成亮灭命令

1. 约定二进制字节 `1` 开灯、`0` 关灯。应用用 `unsigned char cmd = 1;`，再调用 `write(fd, &cmd, sizeof(cmd))`。字符串 `"1"` 的第一个字节是字符 `'1'`，数值为 49。
2. `.write` 先检查 `count`。传 0 字节时直接返回 0。传入 2 字节或更多时返回 `-EINVAL`，因为这里一次只接受 1 字节命令。
3. 拷贝这个字节时，如果 `copy_from_user()` 返回非 0，就给应用返回 `-EFAULT`。拷贝成功后还要检查值，收到了 `0` 或 `1` 才去操作 LED，其余值返回 `-EINVAL`。
4. 验证通过后再读改写 `GPIO1_DR`。命令值 `1` 对应清零第 3 位，因为 LED 低电平有效。处理成功后，`.write` 返回已处理的字节数 `1`。

参考 [Linux 内核 I/O 内存访问文档](https://docs.kernel.org/driver-api/device-io.html)和[用户空间访问函数说明](https://docs.kernel.org/kernel-hacking/hacking.html#copy-to-user-copy-from-user-get-user-put-user)。
