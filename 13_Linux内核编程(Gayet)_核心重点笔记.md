# 《Linux 内核编程》（Thierry Gayet · BPB Online 2025）核心重点笔记

> **书名**：Linux Kernel Programming — Developing kernel architecture and device drivers for character, block, USB, and network interfaces  
> **作者**：Thierry Gayet  
> **出版社**：BPB Online，2025 年  
> **内核基准**：Linux 6.2 / 6.5（示例实测于 6.2.0-39-generic、6.5.5）  
> **配套代码**：https://github.com/bpbpublications/Linux-Kernel-Programming  
> **定位**：一部**以「跑通第一个驱动」为目标的入门级实操教材**。全书 13 章自内核史与构建起步，逐章交付一个可编译、可加载、可观测的最小实例，覆盖**字符设备、块设备与 VFS、USB、网络栈、LSM 安全模块、内存与 DMA、通信接口（I2C/SPI/UART/PCI）、进程管理、内核调试**。理论深度浅于 Madieu 与 Robert Love，但**命令流水账与实验步骤最完整**，适合作为动手前的「操作手册」。

---

## 一、 内核史、构建与引导链（第1–2章）

### 1. 血统与版本演进
* **Unix 血统**：1969 年 Bell Labs（Ken Thompson / Dennis Ritchie）→ Multics 失败 → 1973 年用 C 重写 → AT&T 授权分裂出 BSD 与 System V 两大阵营 → 最终以 **POSIX** 标准化收束。
* **GNU 工程**（1983，Richard Stallman）：贡献 GPL、`gcc`/`glibc`/`bash`/coreutils，但其内核 **GNU Hurd** 微内核始终未成，留下「GNU 用户态 + 缺内核」的历史空位。
* **Linux**：1991 年 Torvalds 发布 0.01 → 1994 年 v1.0 → 采用 **GPLv2**；架构为**单体内核 + 可加载模块（LKM）**，而非微内核。
* **版本号里程碑**：2.6.x 长周期 → 3.x（2011）→ 4.x（2015）→ 5.x（2019）→ 6.x（2022 起）；开发模型为 kernel.org + Linus 的 merge window + 子系统 maintainer 的 git 补丁流（「集市 vs 大教堂」）。

### 2. Kconfig 与构建产物
* **子系统划分**：进程调度、内存管理、VFS、网络栈、IPC、设备驱动，统一入口为 **system call interface**。
* **Kconfig 条目类型**：`bool` / `tristate` / `int` / `string` / `hex`，依赖用 `depends on` / `select` / `default` / `help`；**三态** `=y`（编入 vmlinux）/ `=m`（编成 `.ko`）/ 未设置（不编译）。
* **make 目标**：`menuconfig` / `xconfig` / `gconfig` / `defconfig` / `oldconfig` / `olddefconfig` / `savedefconfig` / `localmodconfig` / `allyesconfig` / `allnoconfig`；产物 `.config`，`CONFIG_LOCALVERSION` 追加到 `uname -r` 后缀。
* **构建产物**：`vmlinux`（未压缩 ELF，带符号供 gdb/objdump）；`arch/x86/boot/bzImage` = `bootsect.o + setup.o + misc.o + piggy.o`（piggy 内为压缩的 vmlinux）；`System.map`（符号地址表）、`Module.symvers`（模块符号 CRC）。
* **安装链**：`make -j$(nproc)` → `make modules_install` → `make install`，`.ko` 落在 `/lib/modules/$(uname -r)/`。
* **配置现况查询**：`/proc/config.gz` 需 `CONFIG_IKCONFIG` + `CONFIG_IKCONFIG_PROC`，否则查 `/boot/config-$(uname -r)`；`uname -r` / `lsmod`。

### 3. 引导链与 initramfs
```
firmware(BIOS/UEFI) → bootloader(GRUB2) → kernel(自解压 setup → 解压 vmlinux → start_kernel)
                    → initrd/initramfs(cpio.gz) → init(PID 1: systemd / sysvinit)
```
* **GRUB2**：`/boot/grub2/grub.cfg`、`grub2-mkconfig -o ...`（Debian 侧 `update-grub`），条目含 `linux` / `initrd` 指令。
* **initramfs 生成**：`update-initramfs -u` / `dracut` / `mkinitcpio`；日志用 `dmesg`、`journalctl -k`。
* **QEMU 直启内核**：`qemu-system-x86_64 -kernel bzImage -initrd initrd -append "console=ttyS0" -nographic`。

---

## 二、 设备驱动入门与模块工程化（第3章）

### 1. 用户态 / 内核态边界
* 硬件特权级 ring 0（内核）/ ring 3（用户），MMU 隔离地址空间；数据跨界**必须** `copy_to_user()` / `copy_from_user()`，**严禁直接解引用用户指针**，指针加 `__user` 标注。
* **系统调用**：用户态经 `syscall` 指令陷入，`sys_call_table` 按编号分派，glibc 提供封装，`strace` 可观测；其他接口还有 `ioctl`、`/proc`、`/sys`(sysfs)、`netlink`、`mmap`、signals。

### 2. 模块骨架与工具链
* `module_init()` / `module_exit()`，段属性 `__init` / `__exit`；`MODULE_LICENSE("GPL")`、`MODULE_AUTHOR`、`MODULE_DESCRIPTION`、`MODULE_VERSION`。
* **模块参数**：`module_param(var, type, perm)`、`module_param_array`、`MODULE_PARM_DESC`；运行时见 `/sys/module/<name>/parameters/`。
* **工具分工**：`insmod`（直接插 `.ko`，**不解析依赖**）、`rmmod`、`modprobe`（读依赖并自动加载）、`depmod`（生成依赖表）、`modinfo`、`lsmod`。
* **索引文件**：`/lib/modules/$(uname -r)/` 下的 `modules.dep`、`modules.alias`、`modules.pcimap`、`modules.usbmap`。
* **开机加载**：`/etc/modules`、`/etc/modules-load.d/<m>.conf`；参数持久化于 `/etc/modprobe.d/<m>.conf` 内 `options <module> k=v`、`blacklist <m>`、`alias`。
* **编译命令**：`make -C /lib/modules/$(uname -r)/build M=$PWD modules`，Makefile 内 `obj-m += foo.o`；内核污染状态见 `/proc/sys/kernel/tainted`。

### 3. DKMS 与嵌入式构建系统
* **DKMS（内核升级后自动重编）**：源码放 `/usr/src/<pkg>-<ver>/`，含 `dkms.conf`（`PACKAGE_NAME` / `PACKAGE_VERSION` / `BUILT_MODULE_NAME[0]` / `DEST_MODULE_LOCATION[0]` / `AUTOINSTALL="yes"`）；子命令 `dkms add|build|install|uninstall|remove`、`dkms status`。
* **Secure Boot**：模块须签名（MOK），工具 `mokutil --import`、`sign-file` / `kmodsign`。
* **Buildroot**：`BR2_*` 配置符号，`make menuconfig` → `make`，产物在 `output/images/`（含 `zImage`/`rootfs`），包用 `*_defconfig`。
* **Yocto**：层 `meta-*`、`bblayers.conf`、`local.conf`、recipe `.bb`；`bitbake core-image-minimal`；内核模块以 `kernel-module-<name>` 加入 `IMAGE_INSTALL`；`devtool` 做开发迭代。

---

## 三、 Linux 设备模型与设备树（第4章）

### 1. 四大对象与 container_of
* **要解决的问题**：内核需要统一抽象表达「总线 / 设备 / 驱动 / 类」，才能支撑电源管理、热插拔、sysfs 呈现与模块自动加载（uevent → udev）。
* `struct kobject`：`name`、`entry`、`parent`、`kset`、`ktype`、`sd`(kernfs_node)、`kref`——**sysfs 的每个目录即一个 kobject**。
* `struct kobj_type`：`release`（必须）、`sysfs_ops`(`show`/`store`)、`default_attrs`；`struct kset` 是 kobject 的集合容器。
* `struct bus_type`：`name`、`match(dev, drv)`、`uevent`、`probe`、`remove`、`shutdown`。
* `struct device`：`parent`、`kobj`、`init_name`、`bus`、`driver`、`platform_data`、`driver_data`、`release`；`struct device_driver`：`name`、`bus`、`owner`、`probe`、`remove`、`of_match_table`、`acpi_match_table`。
* `struct class`：`name`、`class_attrs`，`class_create()` / `class_destroy()`，对应 `/sys/class/*`。
* **`container_of(ptr, type, member)`** 由内嵌成员反推外层对象，是设备模型「继承」语义的基石。
* **注册与遍历**：`bus_register`、`device_register`、`driver_register`；`bus_for_each_dev`、`bus_for_each_drv`。
* **sysfs 目录树**：`/sys/bus`、`/sys/class`、`/sys/devices`、`/sys/module`、`/sys/block`、`/sys/fs`、`/sys/kernel`、`/sys/power`、`/sys/firmware`。

### 2. procfs 与 5.6 破坏性变更
* `proc_create(name, mode, parent, proc_ops)`、`proc_mkdir`、`proc_remove`；私有数据 `PDE_DATA(inode)`；复杂输出用 `seq_file`。
* **内核 5.6 起 `struct proc_ops` 取代 `file_operations`**：字段为 `.proc_open` / `.proc_read` / `.proc_write` / `.proc_lseek` / `.proc_release` / `.proc_ioctl`——沿用旧 `.read`/`.write` 字段会**直接编译失败**。

### 3. 设备树与 Overlay
* 源 `.dts`（可 `#include` `.dtsi`）→ `dtc` 编译 → `.dtb`，由 bootloader 传给内核。
* **关键属性**：`compatible`（驱动匹配的字符串表）、`reg`（地址+长度）、`status`、`interrupt-parent` / `interrupts`、`#address-cells` / `#size-cells`、label `&node` 引用。
* **驱动侧 API**：`of_match_table`、`of_property_read_u32`、`of_get_property`、`for_each_child_of_node`。
* **Overlay**：`/plugin/;` + `fragment@N { target = <&label>; __overlay__ { ... }; }`，运行时经 ConfigFS 应用（`/sys/kernel/config/device-tree/overlays/`）。
* **U-Boot 交互**：`setenv fdtfile`、`fdt addr`、`bootz ${kernel_addr} ${initrd_addr} ${fdt_addr}`、`bootm`。

---

## 四、 字符设备驱动（第5章 · 全书最重实操章）

### 1. 设备号与两代注册路径
* **定位**：字符设备 = 字节流、顺序访问、无块缓冲，对应 `/dev/*`，`open/read/write/ioctl` 直达驱动。
* `dev_t`；`MKDEV(major, minor)` / `MAJOR()` / `MINOR()`，`iminor(inode)` 取次设备号。
* **老式一把梭**：`register_chrdev(major, name, &fops)`（major=0 动态分配）→ `unregister_chrdev`。
* **现代推荐**：`alloc_chrdev_region(&dev, 0, count, name)` / `register_chrdev_region` → `cdev_init` → `cdev_add` → … → `cdev_del` → `unregister_chrdev_region`。

### 2. 关键结构与易错回调语义
* `struct cdev`：`kobj`、`owner`、`ops`、`dev`、`count`。
* `struct file_operations`（本书涉及）：`.owner`、`.open`、`.read`、`.write`、`.release`、`.unlocked_ioctl`、`.llseek`、`.mmap`、`.poll`。
* `struct file`：`f_path`、`f_inode`、`f_pos`、`f_flags`、`f_mode`、`private_data`、`f_op`；`struct inode`：`i_mode`、`i_rdev`、`i_ino`、`i_cdev`。
* `ssize_t read(struct file*, char __user *buf, size_t count, loff_t *ppos)`：返回**实际拷贝字节数**，到 EOF 返回 0。
* **`copy_to_user`/`copy_from_user` 返回未拷贝的字节数**，`0` 才表示全部成功；非 0 应返回 `-EFAULT`。
* `write` 必须更新 `*ppos`；**只有同时实现 `llseek` 才能被 `cat` 正确读取**。
* `open`/`release` 第二参数分别是 `struct inode*` / `struct file*`，次设备号从 `inode` 取。

### 3. 自动建节点与 6.4 破坏性变更
* 路径：`class_create(...)` → `device_create(cls, NULL, MKDEV(major, 0), NULL, DEVICE_NAME)` → 内核发 uevent → **udev** 依据 `/sys` 属性建 `/dev/<name>`。
* **版本坑（书中显式给出）**：
```c
#if LINUX_VERSION_CODE >= KERNEL_VERSION(6, 4, 0)
    cls = class_create(DEVICE_NAME);              // 6.4 起去掉 THIS_MODULE
#else
    cls = class_create(THIS_MODULE, DEVICE_NAME);
#endif
```
* **销毁顺序**：`device_destroy` → `class_destroy` → `cdev_del` → `unregister_chrdev_region`（与创建**严格逆序**）。

### 4. ioctl 与实测流水线
* 命令码用 `_IO` / `_IOR` / `_IOW` / `_IOWR`（magic + nr + type + size 编码）；回调签名 `long (*unlocked_ioctl)(struct file *, unsigned int cmd, unsigned long arg)`。
* **实验流程（书中实测）**：① `make -C /lib/modules/6.2.0-39-generic/build M=$PWD modules` → ② `insmod ./chardrv.ko`；`lsmod | grep chardrv` → ③ `dmesg` 得 `I was assigned major number 235.` → ④ `mknod /dev/mydriver c 235 0` + `chmod 777` → ⑤ `cat /dev/mydriver` 得 `I already told you N times Hello world!` → ⑥ `rmmod chardrv`，并用 `udevadm monitor` 观察 uevent。
* **三类示例**：read-only（`read` + `open`/`release`）、read-write（+`write`）、ioctl（+`unlocked_ioctl`）。

---

## 五、 块设备驱动与虚拟文件系统（第6章）

### 1. I/O 栈与最小块驱动
* **I/O 栈自顶向下**：user → **VFS** → 具体文件系统 → **page cache** → **block layer** → **I/O scheduler** → 设备驱动 → 硬件。
* **关键结构**：`struct gendisk`（`major`、`first_minor`、`minors`、`disk_name[DISK_NAME_LEN]`、`fops`、`queue`、`part_tbl`）；`struct block_device_operations`（`owner`、`open`、`release`、`ioctl`，新式 `submit_bio`）；`struct request_queue`、`struct request`。
* **骨架**：
```c
block_major  = register_blkdev(0, "block");          // 0 = 动态主设备号
block_queue  = blk_init_queue(block_request, NULL);  // 老式 request_fn 模型
block_disk   = alloc_disk(1);                        // 1 个次设备
block_disk->major = block_major;
block_disk->first_minor = 0;
block_disk->fops  = &block_fops;
block_disk->queue = block_queue;
sprintf(block_disk->disk_name, "block");
set_capacity(block_disk, NUM_SECTORS);               // 单位：512B 扇区
add_disk(block_disk);
```
* **请求处理循环**：`req = blk_fetch_request(q)` → `blk_rq_pos(req)`（起始扇区）、`blk_rq_bytes(req)`、`rq_data_dir(req)`（READ/WRITE）→ 用 `kmap` / `vzalloc` 后台缓冲搬数据 → `__blk_end_request_cur(req, err)`。
* **拆除**：`del_gendisk` → `put_disk` → `blk_cleanup_queue` → `unregister_blkdev`。

### 2. I/O 调度器与观测
* 算法：`noop`（后并入 `none`）、`mq-deadline`、`CFQ`（5.0 移除）、`BFQ`、`Kyber`、`NONE`（blk-mq 直通）。
* 查看/切换：`cat /sys/block/sda/queue/scheduler` → `none [mq-deadline]`；`echo noop > /sys/block/sda/queue/scheduler`。
* `blk_init_queue` 属**单队列 legacy** 模型，现代内核走 **blk-mq**（`blk_mq_init_queue`）。
* 观测工具：`iostat -c -d -x -t -m /dev/sda 2`、`iotop`、`fio`、`dd`、`blktrace`、`lsblk`、`blockdev`。

### 3. VFS 四对象与 FUSE
* **四对象**：`super_block`（挂载实例）、`inode`（元数据）、`dentry`（目录项，有 dentry cache）、`file`（打开实例）；分派层级 `file_operations` → `inode_operations` → `super_operations` → `address_space_operations`。
* **文件系统注册**：`struct file_system_type { name, fs_flags, mount, kill_sb, owner }` + `register_filesystem` / `unregister_filesystem`；`mount()` / `umount()`；`/proc/filesystems`；page cache 载体 `struct address_space`（`read_folio`/`writepages`/`fsync`）。
* 常见 FS：ext2/3/4、XFS、Btrfs、FAT/exFAT、NTFS、tmpfs、proc、sysfs、squashfs、overlayfs。
* **FUSE（用户态文件系统）**：`#define FUSE_USE_VERSION 31`；`struct fuse_operations { .getattr, .read, ... }`；`fuse_main(argc, argv, &my_operations, NULL)`；编译 `gcc simplehellofs.c $(pkg-config fuse --cflags --libs) -o simplehellofs`；挂载 `./simplehellofs /mnt/fuse -f`，内核侧经 `/dev/fuse` 转发请求。

---

## 六、 USB 驱动与 libusb（第7章）

### 1. 体系结构与传输类型
* **拓扑**：Host Controller + Hub + Device 构成**分层星型**（≤7 层，单总线 ≤127 设备 = 7 位地址）。
* **主控器**：**UHCI**（USB1.1，Intel）、**OHCI**（USB1.1，非 Intel）、**EHCI**（USB2.0 高速）；速率 Low 1.5 / Full 12 / High 480 Mbit/s。
* **四种传输类型**：

| 类型 | 特性 | 典型用途 |
| :--- | :--- | :--- |
| Control | 双向、有保证，用于枚举 | 描述符读写、类命令 |
| Interrupt | 小包、周期性、有界延迟 | HID（键盘/鼠标） |
| Bulk | 大包、有 CRC 重传、**无时序保证** | 存储、打印机 |
| Isochronous | 保证带宽、**无重传** | 音视频 |

* **线上格式**：包结构 `SYNC + PID(8bit,含4位校验) + 地址(7bit) + 端点(4bit) + 帧号(11bit)/数据 + CRC-5(token)/CRC-16(data) + EOP`；编码用 **NRZI + bit stuffing**（连续 6 个 1 后插 1 个 0）；PID 分 token（OUT/IN/SOF/SETUP）、data（DATA0/DATA1/DATA2/MDATA）、handshake（ACK/NAK/STALL/NYET）、special（PRE/SPLIT/PING）。

### 2. 枚举七步流程
1. Hub 检测到 D+/D- 电平变化 → 报告主机；
2. 主机复位端口（≥10ms）；
3. `GET_DESCRIPTOR(Device, 8B)` 读 `bMaxPacketSize0`；
4. `SET_ADDRESS` 分配 1–127 地址；
5. 完整读 **Device Descriptor(18B)** → 读 **Configuration Descriptor**（含 Interface / Endpoint / String 子描述符）；
6. `SET_CONFIGURATION` 选定配置；
7. 内核为每个 `usb_interface` 匹配 `usb_driver`，调用 `probe()`。
* 描述符关键字段：`bcdUSB`、`idVendor`、`idProduct`、`bDeviceClass`、`bNumConfigurations`、`bNumInterfaces`、`bInterfaceClass`、`bEndpointAddress`、`bmAttributes`、`wMaxPacketSize`、`bInterval`。

### 3. 驱动侧结构与同步 API
* **结构**：`struct usb_device`（整设备）、`struct usb_interface`（一个功能）、`struct usb_host_interface`、`struct usb_endpoint_descriptor`；`struct usb_driver`：`name`、`probe(iface, id)`、`disconnect(iface)`、`suspend`/`resume`/`reset_resume`/`pre_reset`/`post_reset`、`id_table`。
* `struct usb_device_id`：`match_flags`、`idVendor`、`idProduct`、`bInterfaceClass`…；匹配宏 `USB_DEVICE(v,p)`、`USB_DEVICE_INFO(cls,sub,proto)`、`USB_INTERFACE_INFO`、`USB_DEVICE_AND_INTERFACE_INFO`。
* 注册：`module_usb_driver()` 或 `usb_register()` / `usb_deregister()`；`MODULE_DEVICE_TABLE(usb, id_table)` 生成 `modules.usbmap` 以支撑热插拔自动加载。
* **同步传输（本书采用的路径）**：`usb_control_msg(dev, pipe, request, requesttype, value, index, data, size, timeout)`（pipe 用 `usb_sndctrlpipe()` / `usb_rcvctrlpipe()`）、`usb_bulk_msg()`、`usb_interrupt_msg()`。
* **字符节点暴露**：`usb_register_dev(interface, &usb_class_driver)` / `usb_deregister_dev`；`struct usb_class_driver { name, fops, minor_base }`，节点在 `/dev/usb/<name>` 或 usbfs `/dev/bus/usb/BBB/DDD`。
* **Gadget（设备侧）**：UDC → `/sys/kernel/config/usb_gadget/`（ConfigFS）配置 `idVendor`/`idProduct`，挂 function（`f_acm`/`f_ecm`/`mass_storage`），最后写 `UDC` 文件名绑定控制器；OTG / DRD 负责角色切换。

### 4. libusb 用户态路径
* 流程：`libusb_init` → `libusb_get_device_list` → `libusb_get_device_descriptor` → `libusb_open` → `libusb_claim_interface` → `libusb_bulk_transfer` / `libusb_control_transfer` / `libusb_interrupt_transfer` → `libusb_release_interface` → `libusb_close` → `libusb_free_device_list` → `libusb_exit`。
* 用户态**无法 claim 已被内核驱动绑定的接口**（需先 `usb_detach_kernel_driver`）。
* 命令：`lsusb`、`lsusb -t`（拓扑树）、`lsusb -v`（描述符全量）、`usb-devices`；sysfs 在 `/sys/bus/usb/devices/`。
* ⚠ **本书第 7 章未涉及异步 URB 模型**（`struct urb`、`usb_alloc_urb`、`usb_submit_urb`、完成回调），而这恰是内核 USB 驱动的主流写法。

---

## 七、 网络驱动与 Netfilter（第8章）

### 1. 栈分层与 Netfilter
* **分层**：socket 层（BSD socket）→ 协议层（TCP/UDP/IP）→ **Netfilter** → 设备无关层（`dev_queue_xmit` / `netif_rx`）→ 设备驱动。
* **五个 hook**：`NF_INET_PRE_ROUTING`、`NF_INET_LOCAL_IN`、`NF_INET_FORWARD`、`NF_INET_LOCAL_OUT`、`NF_INET_POST_ROUTING`。
* **判决值**：`NF_ACCEPT`、`NF_DROP`、`NF_QUEUE`、`NF_STOLEN`（**Hook 接管包的归属，调用者不再处理**）、`NF_REPEAT`、`NF_STOP`。
* 注册：`struct nf_hook_ops` + `nf_register_net_hook` / `nf_unregister_net_hook`；用户态对应 `iptables` / `nftables`。

### 2. `sk_buff` 操作矩阵
* 队列：`struct sk_buff_head`（双向链表头 + `qlen`）；`skb_queue_head` / `skb_queue_tail` / `skb_dequeue`。
* 分配与克隆：`alloc_skb()`、`dev_alloc_skb()`、`skb_clone()`（共享数据、独立头）、`skb_copy()`（深拷贝）、`skb_copy_expand()`。
* **指针四操作**：`skb_reserve()`（预留头部空间，用于加协议头）、`skb_put()`（尾部扩容）、`skb_push()`（头部前移）、`skb_pull()`（剥离头部）；空间查询 `skb_headroom()` / `skb_tailroom()`。
* 释放：`kfree_skb()`、`dev_kfree_skb()`；共享信息 `skb_shinfo(skb)`（`frags[]`、`nr_frags`、GSO 相关）。
* 设备注册 API：`alloc_etherdev(sizeof_priv)` / `alloc_netdev`、`ether_setup()`、`register_netdev()`、`unregister_netdev()`、`free_netdev()`。
* ⚠ **本章名不副实**：`struct net_device` 字段、`net_device_ops`、`ndo_start_xmit`、NAPI、`napi_struct`、`ethtool_ops`、XDP 均**未出现**——本章重心在协议栈原理、Netfilter、sk_buff 操作与调优，而非「如何写一个网卡驱动」。

### 3. sysctl 调优与 DPDK
* **缓冲**：`net.core.rmem_max` / `wmem_max` / `rmem_default` / `wmem_default`；`net.ipv4.tcp_rmem` / `tcp_wmem`（三值 min/default/max）。
* **队列/并发**：`net.core.netdev_max_backlog`、`net.core.somaxconn`、`net.ipv4.tcp_max_syn_backlog`。
* **协议**：`net.ipv4.tcp_congestion_control`（cubic/reno/bbr）、`tcp_window_scaling`、`tcp_timestamps`、`tcp_sack`、`tcp_fin_timeout`、`tcp_tw_reuse`、`ip_local_port_range`；生效 `sysctl -w k=v`、`sysctl -p`、`/etc/sysctl.conf`。
* **工具**：`ethtool`、`ethtool -G eth0 rx 4096 tx 4096`（**环缓冲大小**）、`ethtool -S`（统计）、`ethtool -k`（offload）、`ip`、`ss`、`iperf3`、`tcpdump`、`nstat`、`sar -n DEV 1`。
* **零拷贝**：`sendfile()`、`splice()`、`mmap()`；进阶 `MSG_ZEROCOPY` / `SO_ZEROCOPY`。
* **DPDK（内核旁路）**：UIO/VFIO 把网卡映射到用户态，**poll-mode driver 轮询**取代中断，**hugepage** 减 TLB 缺失，多 lcore 独占；API `rte_eal_init`、`rte_eth_dev_configure`、`rte_eth_rx_burst` / `rte_eth_tx_burst`、`rte_pktmbuf_alloc` / `rte_pktmbuf_free`、`rte_mempool_create`、`RTE_LCORE_FOREACH_WORKER`。

---

## 八、 Linux 安全模块 LSM（第9章）

### 1. 框架定位与 hook 链
* **要解决的问题**：DAC（自主访问控制，uid/gid + rwx）无法表达「即便 root 也不许」的策略，故需 **MAC（强制访问控制）**。
* LSM 在内核各处埋 **hook 点**形成可插拔检查链；返回 0 = 允许，负 errno（`-EACCES` / `-EPERM`）= 拒绝。
* **两类 hook**：**restrictive**（只能进一步收紧 DAC，不能放宽）与 **authoritative**（可覆盖 DAC 判定）。
* `struct security_operations`：hook 函数指针容器，如 `inode_permission`、`inode_create`、`inode_unlink`、`file_permission`、`task_alloc`、`bprm_check_security`、`socket_*`。
* **对象上的安全字段**：`struct inode -> void *i_security`、`struct task_struct -> void *security`（`struct cred -> security`）、`struct file`、`struct super_block`、`struct sk_buff`。
* 调用包装为 `security_` 前缀内联函数（`security_inode_permission()` 等），未启用 LSM 时编译期退化为恒 0。

### 2. 主流实现与自定义 LSM
* **SELinux**（NSA，Flask 架构）：type enforcement、安全上下文 `user:role:type:level`、`enforcing`/`permissive`/`disabled` 三态。
* **AppArmor**：**路径**绑定 profile，比 SELinux 易用；另有 **Smack**、**TOMOYO**、**Yama**（`ptrace_scope` 限制 ptrace）。
* **自定义 LSM 骨架**：
```c
static struct security_hook_list my_lsm_hooks[] = {
    LSM_HOOK_INIT(inode_permission, my_lsm_inode_permission),
    LSM_HOOK_INIT(inode_create,     my_lsm_inode_create),
    LSM_HOOK_INIT(inode_unlink,     my_lsm_inode_unlink),
};
security_add_hooks(my_lsm_hooks, ARRAY_SIZE(my_lsm_hooks));
```
* hook 内可用 `MAY_WRITE` 等掩码、`current->pid`、`inode->i_ino` 做判定，返回 `-EACCES` 拒绝。
* **启用**：`CONFIG_SECURITY` + `CONFIG_SECURITY_<NAME>`；cmdline `lsm=` 控制顺序与启用；`/sys/kernel/security/lsm` 查看已装载；securityfs 挂载点 `/sys/kernel/security/`；栈叠（stacking）需较新内核。
* ⚠ 书中示例调用的 `security_task_is_allowed()` **并非真实内核 API**，仅为作者示意，不可照抄。

### 3. Capabilities 与凭证
* **能力模型**：把 root 特权切分为细粒度位——`CAP_CHOWN`、`CAP_DAC_OVERRIDE`、`CAP_DAC_READ_SEARCH`、`CAP_FOWNER`、`CAP_KILL`、`CAP_SETUID`/`CAP_SETGID`、`CAP_NET_BIND_SERVICE`、`CAP_NET_RAW`、`CAP_SYS_ADMIN`、`CAP_SYS_MODULE`、`CAP_SYS_RAWIO`、`CAP_SYS_PTRACE`、`CAP_SYS_BOOT`、`CAP_SYS_TIME`、`CAP_MKNOD`、`CAP_IPC_LOCK`、`CAP_LINUX_IMMUTABLE`。
* **操作**：`setcap cap_net_raw+ep ./prog`、`getcap`、`capsh --print`；`/proc/<pid>/status` 的 `CapPrm`/`CapEff`/`CapBnd`；syscall `capget` / `capset`。
* **凭证**：`uid/gid`（真实）、`euid/egid`（有效）、`suid/sgid`（保存）、`fsuid/fsgid`（文件系统判定用）、supplementary groups；`struct cred` + `prepare_creds()` / `commit_creds()`。

---

## 九、 内核内存与 DMA（第10章）

### 1. 地址空间、Zone 与分配器层级
* **三层地址**：用户虚拟地址 → 内核虚拟地址 → 物理地址；`virt_to_phys()` / `phys_to_virt()`（`__pa`/`__va`）、`virt_to_page()`、`page_address()`。
* **Zone**：`ZONE_DMA`（x86 上 <16 MB，给老 ISA 设备）、`ZONE_NORMAL`、`ZONE_HIGHMEM`（仅 32 位）。
* **分配器层级**：**buddy allocator** 按 2^order 页块分配/合并（解决外部碎片）→ **slab / SLUB / SLOB** 小对象缓存（减少内部碎片）→ **vmalloc** 虚拟连续、物理可不连续（大块、非 DMA）。

### 2. 分配 API 选型与 GFP 约束

| API | 物理连续 | 可睡眠 | 场景 |
| :--- | :---: | :---: | :--- |
| `kmalloc(size, gfp)` / `kzalloc` / `kcalloc` / `kmalloc_array` / `krealloc` | 是 | 取决于 gfp | 小对象、需 DMA |
| `vmalloc` / `vzalloc` / `vfree` | 否 | 是 | 大缓冲、不需 DMA |
| `alloc_pages(gfp, order)` / `__get_free_pages` / `free_pages` / `get_zeroed_page` | 是（2^order） | 取决于 gfp | 页级管理 |
| `dma_pool_create` / `dma_pool_alloc` / `dma_pool_free` | 是 | 否（atomic pool） | 设备描述符 |

* `kfree()` 与 `vfree()` **不可混用**；`kmalloc` 上限受 `KMALLOC_MAX_SIZE` 限制。
* **GFP 标志**：`GFP_KERNEL`（可睡眠，**只能进程上下文**）、`GFP_ATOMIC`（不可睡眠，**中断上下文 / 持自旋锁时**）、`GFP_USER`、`GFP_HIGHUSER`、`GFP_DMA`、`GFP_DMA32`、`GFP_NOIO`、`GFP_NOFS`、`GFP_KERNEL_ACCOUNT`；修饰位 `__GFP_ZERO`、`__GFP_NOWARN`、`__GFP_COMP`、`__GFP_RECLAIM`、`__GFP_IO`、`__GFP_FS`、`__GFP_HIGHMEM`。
* **典型坑**：在 `spin_lock` 临界区或 irq 上下文用 `GFP_KERNEL` → 触发 `scheduling while atomic` BUG。
* ⚠ 本书**未讲 slab 编程 API**（`kmem_cache_create/alloc/free/destroy`），只讲概念。

### 3. DMA 两类模型与块层 SG 集成
* **Coherent（一致性）DMA**：`dma_alloc_coherent(dev, size, &dma_handle, gfp)` / `dma_free_coherent()`；CPU 与设备**无需显式同步**即见同一数据，适合长期存在的描述符环；代价是可能走 uncached 内存，CPU 访问慢。
* **Streaming（流式）DMA**：一次传输一次性映射，需明确方向与同步。
  * 方向：`DMA_TO_DEVICE`、`DMA_FROM_DEVICE`、`DMA_BIDIRECTIONAL`、`DMA_NONE`。
  * **本书只讲 SG 变体**：`dma_map_sg(dev, sg, nents, dir)`（返回实际映射段数，**可能 < nents**，因合并）、`dma_unmap_sg()`、`dma_sync_sg_for_cpu()` / `dma_sync_sg_for_device()`。
  * ⚠ 单缓冲变体 `dma_map_single` / `dma_unmap_single` / `dma_sync_single_for_cpu|device` 以及 `remap_pfn_range` 均**未涉及**。
* **掩码**：`dma_set_mask(dev, mask)`（书中以裸 `0xFFFFFFFF` 表达，未用 `DMA_BIT_MASK(64)`）。
* **IOMMU / GART / bounce buffer**：IOMMU 提供设备侧地址转换（IOVA → PA），使**物理不连续**的页对设备表现为连续；GART 是 AMD 早期 IOMMU 实现；设备地址位宽不足时内核用 **bounce buffer**（`swiotlb`）在低端内存中转拷贝。
* **块层 SG 完整链路**：
```
blk_rq_map_sg(q, req, sglist)  →  dma_map_sg(dev, sg, nents, dir)
    →  for_each_sg(...) { sg_dma_address(sg); sg_dma_len(sg); }
    →  dma_unmap_sg(...)
```
`sg_dma_address()` / `sg_dma_len()` 取**设备可见地址与长度**（映射后与 `sg_phys` / `sg->length` 可能不同）。
* **调试观测**：`/proc/meminfo`、`/proc/slabinfo`、`slabtop`、`free`、`vmstat`；`CONFIG_SLUB_DEBUG`、`CONFIG_DEBUG_KMEMLEAK`。

---

## 十、 通信接口：I2C / SPI / UART / PCI（第11章）

### 1. I2C（两线、多主多从）
* **物理**：SDA（数据）+ SCL（时钟），开漏 + 上拉；每帧有 ACK/NACK，支持 clock stretching。
* **速率档**：Standard 100 kHz、Fast 400 kHz、Fast-mode Plus 1 MHz、High-speed 3.4 MHz、Ultra-fast 5 MHz；地址 **7 位**（0x08–0x77 可用），可扩展 10 位。
* **模型**：`struct i2c_adapter`（控制器）↔ `struct i2c_client`（从设备）↔ `struct i2c_driver`；`struct i2c_msg`（`addr`/`flags`/`len`/`buf`）。
* **注册**：`i2c_add_adapter` / `i2c_add_numbered_adapter` / `i2c_del_adapter`；`i2c_register_driver` / `i2c_add_driver`（更常用 `module_i2c_driver` 宏）。
* **传输**：`i2c_master_send(client, buf, count)`、`i2c_master_recv(client, buf, count)`；SMBus 便捷层 `i2c_smbus_read_byte_data` / `write_byte_data` / `read_word_data` / `write_word_data` / `read_i2c_block_data`。
* **实例化**：静态 `struct i2c_board_info` + `i2c_new_device()`；或 DT 中 `compatible = "vendor,chip"; reg = <0x50>;`；私有数据 `i2c_set_clientdata` / `i2c_get_clientdata`；用户态经 `i2c-dev` 的 `/dev/i2c-N`。

### 2. SPI（四线、全双工）
* **物理**：SCLK + MOSI + MISO + CS/SS，**全双工**；模式由 CPOL × CPHA 决定（mode 0/1/2/3），另有 MSB/LSB first、8/16 bit 字宽。
* **模型**：`struct spi_master`（新名 `spi_controller`）↔ `struct spi_device` ↔ `struct spi_driver`；消息用 `struct spi_message` 串起多个 `struct spi_transfer`。
* **注册**：控制器 `spi_alloc_master` / `spi_register_master` / `spi_unregister_master`（现代用 `devm_spi_alloc_master`）；驱动 `spi_register_driver`。
* **传输**：`spi_setup(spi)` 设定速率/模式；`spi_write()`、`spi_read()`、`spi_write_then_read(spi, txbuf, n_tx, rxbuf, n_rx)`；消息式 `spi_message_init()` → 多次 `spi_message_add_tail()` → `spi_sync()` / `spi_async()`。
* **`struct spi_transfer` 字段**：`tx_buf`、`rx_buf`、`len`、`speed_hz`、`bits_per_word`、`delay_usecs`、`cs_change`；统计在 `/sys/class/spi_master/spi0/statistics/`。
* **DT 属性**：`compatible`、`reg`、`spi-max-frequency`、`spi-cpol`、`spi-cpha`、`spi-cs-high`、`bits-per-word`。

### 3. 串口 / UART 与接收路径
* **模型**：`struct uart_driver` ↔ `struct uart_port`（一个物理口）↔ `struct uart_ops`，核心在 `serial_core.c`。
* **注册**：`uart_register_driver()` / `uart_unregister_driver()`、`uart_add_one_port()` / `uart_remove_one_port()`、`uart_suspend_port()` / `uart_resume_port()`。
* **`struct uart_ops` 关键回调**：`.startup`、`.shutdown`、`.set_termios`、`.type`、`.request_port`、`.release_port`、`.config_port`、`.verify_port`、`.start_tx`、`.stop_tx`、`.tx_empty`、`.set_mctrl`、`.get_mctrl`、`.stop_rx`、`.ioctl`、`.pm`。
* **接收路径（关键流程）**：
```
IRQ → uart_handle_sysrq_char(port, ch)   （Magic SysRq 旁路）
    → uart_insert_char(port, status, overrun, ch, flag)
    → 写入 uart_circ（tty_port 环形缓冲）
    → tty_flip_buffer_push(port)
    → line discipline（N_TTY：回显、行编辑、canonical 模式）
```
* **参数结构**：`struct termios` / `struct ktermios`；`tty_termios_baud_rate()`、`tty_set_termios()`、`tty_hangup()`。
* **经典 PC 端口数值**：**COM1 = `0x3F8`，IRQ 4**；`uartclk` = 1843200 Hz；分频 `divisor = uartclk / (16 × baud)`。
* **命令**：`stty -F /dev/ttyS0 115200 cs8 -cstopb -parenb`、`minicom`、`screen /dev/ttyUSB0 115200`、`picocom`、`socat`。
* ⚠ 本书**只到 `uart_driver`/`uart_port`/`uart_ops` 层**，未进 `tty_driver` / `tty_operations` / line discipline 的编程。

### 4. PCI / PCIe
* **配置空间**：PCI **256 字节**，PCIe 扩展至 **4096 字节**；寻址为 BDF（bus/device/function），单 domain 256 总线 × 32 设备 × 8 功能 = 65536。
* **BAR**（6 个）：I/O 空间 vs 内存空间，32/64 位内存 BAR，含 prefetchable 位；中断从 INTx（INTA–INTD）→ **MSI** → **MSI-X**。
* `struct pci_driver`：`node`、`name`、`id_table`、`probe(pdev, id)`、`remove(pdev)`、`suspend`/`resume`、`shutdown`、`sriov_configure`、`err_handler`、内嵌 `struct device_driver driver`。
* `struct pci_device_id`：`vendor`、`device`、`subvendor`、`subdevice`、`class`、`class_mask`、`driver_data`；匹配宏 `PCI_DEVICE(vend, dev)`、`PCI_DEVICE_CLASS(cls, mask)`、`PCI_ANY_ID`；`MODULE_DEVICE_TABLE(pci, id_table)`；注册 `pci_register_driver()`（或 `module_pci_driver`）。
* **使能与资源申请的完整顺序**：
```c
pci_enable_device(pdev);
bars = pci_select_bars(pdev, IORESOURCE_MEM | IORESOURCE_IO);   // 返回 BAR 位掩码
pci_request_selected_regions(pdev, bars, "mydriver");
base = pci_iomap(pdev, bar, 0);                                  // 映射后 ioread32/iowrite32
...
pci_iounmap(pdev, base);
pci_release_selected_regions(pdev, bars);
pci_disable_device(pdev);
```
* **访问宏**：`ioremap`/`iounmap`、`readl`/`writel`、`readb/writeb`、`readw/writew`、`readq/writeq`；端口 I/O `inb/outb/inw/outw/inl/outl`；屏障 `mb()` / `rmb()` / `wmb()`。
* **命令**：`lspci`、`lspci -vvv`、`lspci -t`、`lspci -nn`（带 ID）、`setpci`；`/proc/bus/pci/devices`、`/sys/bus/pci/devices/`。

---

## 十一、 进程管理与并发原语（第12章）

### 1. 进程状态与 CFS
* **九种状态**：Created（过渡态）、User running、Kernel running、Ready to run in memory、Asleep in memory、Ready to run (swapped)、Sleep (swapped)、Preempted（内核态返回用户态时被打断）、Zombie（已 exit，留退出码与统计供 parent 回收）。
* **CFS**（2.6.23 引入，取代 O(1) 调度器）：不用固定时间片，按权重比例分配 CPU；**vruntime** 越小优先级越高；用**红黑树**按 vruntime 组织，每次取**最左节点**运行；配 load balancing 做跨 CPU/core 均衡。

### 2. 调度策略与实时化
* **普通策略**：`SCHED_NORMAL`(=`SCHED_OTHER`, CFS)、`SCHED_BATCH`（吞吐优先、少抢占交互）、`SCHED_IDLE`（仅 CPU 空闲时跑）。
* **实时策略**：`SCHED_FIFO`（无时间片，跑至阻塞或让出，**可饿死低优先级**）、`SCHED_RR`（同优先级轮转）、`SCHED_DEADLINE`（按 runtime/deadline/period 调度，非优先级）。
* **数值**：RT 优先级 **1–99**（99 最高）；nice **-20 ~ 19**（越大越低）。
* **命令**：`nice -n 10 myprocess`、`renice`、`chrt -r -p 20 1234`、`chrt -f -p 99 $(pgrep app)`、`top`/`htop`/`atop`/`btop`/`glances`/`ps`/`pgrep`/`pidstat`。
* **PREEMPT_RT**：标准内核非完全可抢占、CFS 无时序保证；PREEMPT_RT 提供全内核抢占、**优先级继承**（防 priority inversion）、高精度定时器、中断线程化；发行版如 RHEL-RT、Ubuntu Studio（low-latency kernel）。测延迟 `cyclictest -t1 -p 80 -n -i 10000 -l 10000`。

### 3. kthread、workqueue 与同步原语
* **kthread**：`kthread_create(threadfn, data, namefmt, ...)` → `wake_up_process()`；便捷版 `kthread_run()`；停止 `kthread_stop(k)`，线程内用 `while (!kthread_should_stop())` 轮询；错误判定 `IS_ERR(kthread)` / `PTR_ERR(kthread)`。
* **workqueue**（运行于**进程上下文，允许 sleep/block**）：`struct work_struct`（工作项）、`struct workqueue_struct`（worker 池）；`DECLARE_WORK(name, fn)`、`INIT_WORK(&w, fn)`、`queue_work(wq, &w)`、`schedule_work(&w)`、`create_workqueue("name")`、`destroy_workqueue(wq)`。
* **同步原语**：
  * `spinlock_t`：`spin_lock_init` / `spin_lock` / `spin_unlock`，中断安全版 `spin_lock_irqsave(&l, flags)` / `spin_unlock_irqrestore`；
  * `struct mutex`：`mutex_init` / `mutex_lock` / `mutex_unlock`（争用则睡眠，**仅进程上下文**）；
  * `rwlock_t`：`read_lock`/`read_unlock`、`write_lock`/`write_unlock`（读多写少）；
  * **seqlock**：写侧 `write_seqlock`/`write_sequnlock`，读侧 `do { seq = read_seqbegin(&sl); ... } while (read_seqretry(&sl, seq));`
  * `struct completion`：`init_completion` / `wait_for_completion` / `complete`（中断与进程间同步）；
  * **RCU**：读侧 `rcu_read_lock()` / `rcu_read_unlock()`，写侧 `synchronize_rcu()`；
  * `struct semaphore`：`sema_init(&s, 1)` / `down` / `up`；**BKL（大内核锁）**已废弃并移除。
* ⚠ 本书未覆盖 `task_struct` 字段与状态常量、`fork`/`clone`/`execve`、`sched_setscheduler`、tasklet/softirq、`timer_list`/hrtimer、`delayed_work`/`cancel_work_sync`、`atomic_t`、lockdep。

---

## 十二、 内核与驱动调试（第13章）

### 1. 调试配置与毒化字节
* **基础**：`CONFIG_DEBUG_KERNEL`、`CONFIG_DEBUG_INFO`、`CONFIG_DEBUG_INFO_DWARF4`、`CONFIG_KALLSYMS`、`CONFIG_KALLSYMS_ALL`、`CONFIG_FRAME_POINTER`、`CONFIG_STACKTRACE`、`CONFIG_DEBUG_FS`。
* **内存**：`CONFIG_SLUB_DEBUG`、`CONFIG_DEBUG_PAGEALLOC`、`CONFIG_DEBUG_KMEMLEAK`、`CONFIG_DEBUG_LIST`、`CONFIG_DEBUG_OBJECTS`。
* **并发**：`CONFIG_DEBUG_SPINLOCK`、`CONFIG_DEBUG_MUTEXES`、`CONFIG_PROVE_LOCKING`、`CONFIG_LOCKDEP`、`CONFIG_DEBUG_ATOMIC_SLEEP`。
* **毒化字节**：释放后的 slab 对象被填 `0xA5` / `0x6B` 模式，用于识别 use-after-free。
* **功能开关**：`CONFIG_MAGIC_SYSRQ`、`CONFIG_DYNAMIC_DEBUG`、`CONFIG_KGDB`、`CONFIG_KGDB_SERIAL_CONSOLE`、`CONFIG_KGDB_KDB`、`CONFIG_KPROBES`、`CONFIG_KPROBE_EVENTS`、`CONFIG_FTRACE`、`CONFIG_FUNCTION_TRACER`。

### 2. Magic SysRq 与 printk
* **触发**：`Alt+SysRq+<key>`（键盘）或 `echo <key> > /proc/sysrq-trigger`（串口/远程）；总开关 `/proc/sys/kernel/sysrq`（位掩码）。
* **常用键**：`b` 立即重启、`c` 主动 crash（触发 kdump）、`e` SIGTERM 除 init 外全部、`i` SIGKILL 全部、`k` SAK（安全访问键）、`l` 所有 CPU 栈回溯、`m` 内存信息、`p` 寄存器、`q` 定时器、`s` sync、`t` 任务列表、`u` 重挂只读、`w` 阻塞任务、`z` ftrace 缓冲、`n` RT 任务。
* **printk 等级** 0–7：`KERN_EMERG`(0)、`KERN_ALERT`(1)、`KERN_CRIT`(2)、`KERN_ERR`(3)、`KERN_WARNING`(4)、`KERN_NOTICE`(5)、`KERN_INFO`(6)、`KERN_DEBUG`(7)；宏 `pr_err` / `pr_warn` / `pr_info` / `pr_debug`，设备版 `dev_err` / `dev_info` / `dev_dbg`。
* **控制台级别**：`/proc/sys/kernel/printk`（`console_loglevel default_msg_loglevel ...`）；`dmesg -n 8`。
* **读取**：`dmesg`、`dmesg -w`（跟随）、`dmesg -T`（人类时间）、`journalctl -k`、直接读 `/dev/kmsg`。

### 3. 动态调试与源码级调试
* **Dynamic Debug**（需 `CONFIG_DYNAMIC_DEBUG`）：控制文件 `/sys/kernel/debug/dynamic_debug/control`；`echo 'file mydrv.c line 42 +p' > control`；标志 `p`(print) `f`(function) `l`(line) `m`(module) `t`(thread id)；启动参数 `dyndbg="file mydrv.c +p"`。价值：`pr_debug()` 默认零开销，运行时按需开启，无需重编译。
* **kgdb / kdb**：启动参数 `kgdboc=ttyS0,115200`（可加 `kgdbwait` 让内核等待连接）；进入调试器 `echo g > /proc/sysrq-trigger`；主机侧 gdb `target remote /dev/ttyS0` → `add-symbol-file mydrv.ko <text_addr>` → `lx-symbols` → `continue`；kdb 命令 `help`、`lsmod`、`bt`、`ps`、`g`/`go`、`bp`、`md`、`rd`。
* **kprobe / kretprobe（无侵入探针）**：`struct kprobe { addr; symbol_name; pre_handler; post_handler; fault_handler; ... }`，`register_kprobe()` / `unregister_kprobe()`；`struct kretprobe` + `register_kretprobe()`，`maxactive` 控制并发实例上限；定位符号 `kallsyms_lookup_name("do_sys_open")`、`/proc/kallsyms`；亦可走 tracefs `kprobe_events` 写 `p:myprobe do_sys_open`。

### 4. OOPS / panic / kdump 与性能追踪
* **触发**：`BUG()`、`BUG_ON(cond)`、`WARN()`、`WARN_ON(cond)`（非致命，继续运行）、`panic()`。
* **OOPS 文本要素**：`RIP`、`Call Trace`、`Code:`、寄存器 `RAX..R15`、`CR2`（缺页地址）。
* **符号化**：`addr2line -e vmlinux -f -i <addr>`、`scripts/faddr2line vmlinux <func+off>`、`objdump -d vmlinux`、`gdb vmlinux`。
* **限制**：`oops_limit` 默认 **10000**（超出后 panic）、`panic_on_oops`、`panic_timeout`。
* **kdump**：`crashkernel=<size>@<addr>` 预留 crash 内核；`kexec -p /boot/vmlinuz --initrd=... --append="..."`；崩溃后经 `elfcorehdr=` 传内存镜像；用 `/proc/vmcore` + `crash` 工具 / `makedumpfile` 分析。
* **性能分析与追踪**：`perf stat`、`perf record -g` / `perf report`、`perf top`、`perf trace`；**ftrace**（tracefs `/sys/kernel/tracing/`）的 `available_tracers`、`current_tracer`（`nop` / `function` / `function_graph` / `sched_switch` / `irqsoff`）、`set_ftrace_filter`、`trace`、`trace_pipe`；前端 `trace-cmd record` / `trace-cmd report`、`KernelShark`；动态追踪 `systemtap`、`bpftrace`、`bcc`；静态工具 `nm`、`objdump`、`readelf`、`gdb`、`strace`、`ltrace`、`Valgrind`、`/proc/modules`、`/sys/module/<m>/sections/.text`。

---

## 十三、 本书定位与覆盖强度评注

1. **定位**：面向「第一次写驱动」的读者，每章一个可跑的最小示例 + 命令流水账；**理论深度浅**，实操步骤多，适合与 Madieu 两版配合使用（本书给「怎么敲」，Madieu 给「为什么」）。
2. **覆盖强度排序**（真实内容量）：第5章 > 第11章 > 第6章 ≈ 第13章 > 第9章 ≈ 第10章 > 第3章 ≈ 第4章 > 第7章 ≈ 第8章 > 第12章 > 第2章 > 第1章。
3. **第8章名不副实**：章名为 Network Drivers，实际几乎不写 `net_device` 驱动（无 `ndo_start_xmit`、无 NAPI、无 `net_device_ops`），重心在协议栈原理、Netfilter、`sk_buff` 操作 API、sysctl 调优与 DPDK。
4. **第7章只讲同步 USB**：URB 异步模型（内核 USB 驱动的主流写法）完全缺席，反而把 libusb 用户态路径讲得较全。
5. **第10章缺半边**：有 Zone / GFP / `alloc_pages` / SG-DMA / IOMMU，但缺 `kmem_cache_*`、`dma_map_single`、`remap_pfn_range`——即「slab 编程」与「单缓冲流式 DMA」两个最常用的 API 面。
6. **第11章缺 tty 层**：只到 `uart_driver` / `uart_port` / `uart_ops`，未进 `tty_driver` / `tty_operations` / line discipline 编程。
7. **书内瑕疵（引用时需标注「书中如此」）**：第12章 `SCHED_IDLE` 条目误用了 RT 策略的描述；第9章示例调用了不存在的 `security_task_is_allowed()`；第12章 `chrt` 命令原文把 `$(pgrep app)` 误排成 `$99`。
