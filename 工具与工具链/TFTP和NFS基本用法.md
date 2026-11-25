# TFTP 和 NFS 基本用法

- **TFTP** 用于启动前调试设备树。U-Boot 把新版 `.dtb` 下载到开发板内存，再交给 Linux。
- **NFS** 用于 Linux 启动后调试驱动。Ubuntu 共享编译好的 `.ko`，开发板挂载后直接加载。

以下以正点原子 ALPHA i.MX6ULL 的 eMMC 系统为例。Ubuntu 用 `192.168.50.250/24`，开发板用 `192.168.50.251/24`。如果 Ubuntu 跑在虚拟机里，把连接开发板的虚拟网卡桥接到插网线的物理网卡。

## TFTP 临时启动新设备树

### 1. 确定板上使用的 DTB

在开发板串口打断自动启动，进入 U-Boot，执行 `printenv bootcmd loadfdt fdt_file`，也可以从正常启动日志里看实际加载的`.dtb`文件名。

本例的 4.3 寸 800×480 屏用 `imx6ull-14x14-emmc-4.3-800x480-c.dtb`。因此想要在不改变原来配置的基础上增改设备树，就要去这个`.dtb`对应的`.dts`文件里面改。

### 2. 在 Ubuntu 修改、编译并复制到 TFTP 目录

在 **Linux 内核源码根目录**编辑 `arch/arm/boot/dts/imx6ull-14x14-emmc-4.3-800x480-c.dts`。沿用当前源码配置和交叉工具链。下面以 `arm-linux-gnueabihf-` 为例，只编译这份 DTB。编译成功后再复制到 TFTP 服务目录。

```bash
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- imx6ull-14x14-emmc-4.3-800x480-c.dtb
sudo install -m 0644 arch/arm/boot/dts/imx6ull-14x14-emmc-4.3-800x480-c.dtb /tftpboot/
```

这里假设 `tftpd-hpa` 已启动，服务目录为 `/tftpboot`。实际目录看 Ubuntu 的 `/etc/default/tftpd-hpa` 中的 `TFTP_DIRECTORY`。

### 3. 在 U-Boot 加载并启动

先看 `printenv ipaddr serverip`，确认板端地址是 `192.168.50.251`，服务器地址是 Ubuntu 的 `192.168.50.250`。下面从 eMMC 第 1 分区读取现有 `zImage`，再从 Ubuntu 下载新 DTB。两条加载命令都成功后再执行 `bootz`。

```bash
fatload mmc 1:1 80800000 zImage
tftp 83000000 imx6ull-14x14-emmc-4.3-800x480-c.dtb
setenv bootargs 'console=ttymxc0,115200 root=/dev/mmcblk1p2 rootwait rw'
bootz 80800000 - 83000000
```

DTB 从 Ubuntu 的 `/tftpboot/` 通过网线传到板端内存 `0x83000000`。`bootz` 将它交给 Linux，根文件系统仍按 `bootargs` 从 eMMC 第 2 分区挂载。新 DTB 只用于本次启动，下次自动开机仍会按原配置读取 eMMC 上的 DTB。

### 4. 确认生效

- TFTP 显示 `done` 和 `Bytes transferred` 表示传输完成。遇到 `PHY ... TIMEOUT`，先检查网线是否插牢、链路灯是否亮，再在 U-Boot 执行 `ping 192.168.50.250`。能 ping 通但下载失败时，检查 Ubuntu 的 TFTP 服务、目录和文件名。
- 进入板端 Linux 后，执行 `ls /proc/device-tree/alphaled`，再检查本次修改的属性，确认内核读到了新设备树。

## NFS 加载驱动模块

**Ubuntu 主机**安装 `nfs-kernel-server` 和 `rpcbind`，创建 `/srv/nfs/drivers`，在 `/etc/exports` 加一行。 [Ubuntu NFS 文档](https://ubuntu.com/server/docs/how-to/networking/install-nfs/)

```bash
/srv/nfs/drivers 192.168.50.251(ro,sync,no_subtree_check,root_squash)
```

启动服务后执行 `sudo exportfs -ra`。用与板上内核匹配的构建目录编译模块，再执行 `sudo install -m 0644 demo.ko /srv/nfs/drivers/`。

**开发板进入 Linux 后**，确认 Linux 网口已配置好 `192.168.50.251/24`，再以 root 挂载并加载驱动。

```sh
mkdir -p /mnt/drivers
mount -t nfs -o vers=3,proto=tcp,nolock,ro 192.168.50.250:/srv/nfs/drivers /mnt/drivers
insmod /mnt/drivers/demo.ko
dmesg | tail -n 20
```

主机复制新版 `.ko` 后，板端退出占用设备的程序，再执行 `rmmod demo && insmod /mnt/drivers/demo.ko`。手动挂载在板端重启后要重新执行。
