# 使用网络和 NFS 开发 Linux 驱动

**开发主机编译驱动，开发板通过 NFS 读取 `.ko` 并加载。** 更新模块后重新加载即可。修改内核镜像或设备树时，更新对应的启动文件再重启。

## 网络连接

网线直连时，两端使用同一网段里的不同 IP，网关和 DNS 留空。接同一台路由器时，也可以用 DHCP。

虚拟机既要上网又要连开发板时，可以配两张网卡。

| 虚拟网卡 | 用途 |
| --- | --- |
| NAT 网卡（如 VMware 的 VMnet8） | 通过宿主机的网络上网，通常用 DHCP 获取地址和默认路由 |
| 桥接网卡（如 VMware 的 VMnet0） | 桥接到接开发板的物理有线网卡。开发板用这张网卡的 IP 访问虚拟机里的 NFS 服务 |

以下用 `192.168.50.10/24` 表示主机，`192.168.50.20/24` 表示开发板。用 `ip link` 查看并替换实际网卡名。主机使用临时 IP 时，重启后要重新设置。

```bash
# 开发主机
sudo ip link set enp3s0 up                        # 启用连接开发板的网口
sudo ip addr replace 192.168.50.10/24 dev enp3s0   # 在这个网口设置主机的 IP 和子网前缀
```

## 首次配置 NFS 服务

以 Debian / Ubuntu 为例，在主机安装服务、建立共享目录和测试文件。[Ubuntu NFS 文档](https://ubuntu.com/server/docs/how-to/networking/install-nfs/)

```bash
sudo apt update                          
sudo apt install nfs-kernel-server rpcbind   

# 创建共享目录，开发用户可写，板端可进入目录并读取公开文件
sudo install -d -m 0755 -o "$(id -un)" -g "$(id -gn)" /srv/nfs/drivers
printf 'NFS OK\n' > /srv/nfs/drivers/hello.txt  # 准备挂载后要读取的测试文件
chmod 644 /srv/nfs/drivers/hello.txt           # 允许开发板读取测试文件
sudoedit /etc/exports                         # 编辑 NFS 的目录导出规则
```

在 `/etc/exports` 加入下列配置，将 `/srv/nfs/drivers` 共享给 `192.168.50.20`。`ro` 让板端只读，`root_squash` 将板端 root 映射为服务端匿名用户。

```text
/srv/nfs/drivers 192.168.50.20(ro,sync,no_subtree_check,root_squash)
```

```bash
sudo systemctl start rpcbind nfs-kernel-server  # 启动 NFS 和 RPC 服务
sudo exportfs -ra                               # 重新读取配置并应用导出规则
sudo exportfs -v                                # 查看当前导出的目录、客户端及选项
```

示例使用 NFSv3，板端内核和挂载工具都要支持它。Debian / Ubuntu 客户端可装 `nfs-common`。

## 开发板每次开机后

先确认开发主机的 IP 和 NFS 服务可用。板端手动执行的 `mount` 重启后会失效。需要自动挂载时，可在板端用 `/etc/fstab` 配置网络挂载，或在 BusyBox 启动脚本里等网口配好 IP 后再挂载。开机时已自动配置 IP 或完成挂载的，可以跳过对应的手动命令。

1. **在开发板配置 IP 并挂载共享。** 用 root 执行，`ping` 通后再挂载。精简系统可用 `ifconfig eth0 192.168.50.20 netmask 255.255.255.0 up` 配置并启用网口。

   ```sh
   ip link set eth0 up                          # 启用板端网口
   ip addr replace 192.168.50.20/24 dev eth0     # 设置板端 IP 和子网前缀
   ping -c 3 192.168.50.10                      # 向主机发送 3 次探测，验证网络连通

   mkdir -p /mnt/drivers                        # 创建板端挂载点
   # 通过 NFSv3/TCP 把主机的导出目录挂到板端，使用只读模式，无需远程文件锁
   mount -t nfs -o vers=3,proto=tcp,nolock,ro 192.168.50.10:/srv/nfs/drivers /mnt/drivers
   cat /mnt/drivers/hello.txt                   # 读到 NFS OK 说明板端能访问共享文件
   ```

2. **在主机编译并发布 `.ko`。** 共享目录里已有本次要测试的模块时，可以直接加载。需要编译时，在板端用 `uname -r` 查看内核版本、`uname -m` 查看架构，据此选择对应的内核构建和工具链。下面以 ARM64 为例，在 `demo.c` 同目录的 `Kbuild` 写 `obj-m := demo.o`，让内核构建系统生成 `demo.ko`。[Linux 外部模块构建文档](https://docs.kernel.org/kbuild/modules.html)

   ```bash
   KDIR="$HOME/sdk/kernel-build"                # 指定目标板的内核构建目录，改成实际路径
   # -C 进入内核构建目录，M 指向当前模块源码目录，ARCH 和 CROSS_COMPILE 指定目标架构和工具链
   make -C "$KDIR" M="$PWD" ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- modules
   install -m 0644 demo.ko /srv/nfs/drivers/     # 把新模块复制到 NFS 目录，并设为板端可读
   ```

3. **在开发板加载模块并测试。** 查看日志，再用测试程序验证设备功能。

   ```sh
   insmod /mnt/drivers/demo.ko                  # 按文件路径把模块加载到板端内核
   dmesg | tail -n 20                           # 查看最近 20 行内核日志，确认初始化或报错信息
   ```

本次开机期间继续改驱动时，沿用已挂载的目录。重新编译并复制 `.ko` 后，先退出占用设备的程序，再执行 `rmmod demo && insmod /mnt/drivers/demo.ko`。`rmmod` 卸载旧模块，`&&` 保证卸载成功后才加载新版。

## NFS 根文件系统

调试整套根文件系统时，可单独导出 `/srv/nfs/rootfs`。引导加载器从本地介质或通过 TFTP 加载内核和设备树，Linux 启动时把 NFS 目录挂为 `/`，开机后可以直接访问其中的文件。内核直接挂载需要内建 NFS 客户端、`CONFIG_ROOT_NFS`、`CONFIG_IP_PNP` 和网卡驱动。下面是直连时的启动参数示例。[Linux NFS 根文件系统文档](https://www.kernel.org/doc/html/v6.1/admin-guide/nfs/nfsroot.html)

```text
root=/dev/nfs rw nfsroot=192.168.50.10:/srv/nfs/rootfs,vers=3,proto=tcp ip=192.168.50.20:192.168.50.10::255.255.255.0::eth0:off
```

`root=/dev/nfs` 选择网络根文件系统，`rw` 请求读写挂载，`nfsroot=` 指定服务端、目录和协议。`ip=` 负责启动时配置网络，示例将板端 IP 设为 `192.168.50.20`，使用 `eth0`，留空网关和主机名，末尾的 `off` 表示使用指定的静态 IP。

## 常见问题

- **`ping` 不通时**，查网口灯、两端 IP 和掩码，以及桥接所选的物理网口。
- **NFS 挂载超时时**，查服务状态和 `/proc/fs/nfsd/versions`。用 NFSv3 时，可用 `rpcinfo -p localhost` 检查 RPC 服务，并核对防火墙。
- **挂载报 `access denied` 时**，用 `exportfs -v` 检查导出路径和板端 IP。
- **模块加载失败时**，先看 `dmesg`。`Invalid module format` 查内核构建和架构，`Unknown symbol` 查依赖和符号版本。
