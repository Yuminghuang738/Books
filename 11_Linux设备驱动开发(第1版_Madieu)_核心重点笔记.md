# 《Linux 设备驱动开发》（原书第1版 · John Madieu · 内核 v4.13）核心重点笔记

> **书名**：Linux Device Drivers Development  
> **作者**：John Madieu（法国嵌入式 Linux / 内核工程师，LABCSMART 创始人）  
> **出版社**：Packt Publishing，2017 年 10 月第 1 版  
> **内核基准**：Linux v4.13（源码同时在 x86 PC 与 NXP i.MX6 的 UDOO QUAD 上实测）  
> **测试器件**：MCP23016（I2C GPIO 控制器）、24LC512（I2C EEPROM）  
> **定位**：面向嵌入式工程师的**驱动开发全景手册**。全书以 22 章的篇幅自底向上贯通「内核模块基础 → 字符/平台设备 → 设备树 → 总线子系统（I2C/SPI/Regmap）→ 内存与 DMA → 设备模型 → 外设框架（GPIO/IRQ/Input/RTC/PWM/Regulator/Framebuffer/NIC）」，是《Linux 内核设计与实现》之后衔接**真实驱动编写**的关键一环。

---

## 一、 内核开发环境与内核模块（第1–2章）

### 1. 构建、模块骨架与 kbuild
* 目录 `arch/`/`drivers/`/`include/`/`kernel/`/`mm/`/`fs/`/`net/`/`Documentation/`；主机构建 `make x86_64_defconfig` → `make menuconfig` → `make -j16`；ARM 交叉编译 `export ARCH=arm`、`export CROSS_COMPILE=arm-linux-gnueabihf-`，`make imx_v6_v7_defconfig`（i.MX6）、`make zImage -j16`、`make dtbs`、`make modules`。
* 镜像 `vmlinux`（ELF 未压缩）→ `Image`/`zImage`/`uImage`，模块 `.ko`，`dtc` 把 `.dts` 编成 `.dtb`；内核无 libc、无浮点（需 `kernel_fpu_begin/end`）、栈极小（`THREAD_SIZE`，32 位常 8KB）、无内存保护；`module_init()` / `module_exit()`；`MODULE_LICENSE("GPL")`（缺失会污染内核且部分符号不可用）、`MODULE_AUTHOR`、`MODULE_ALIAS`；`__init` 段代码初始化后被释放、`__exit` 仅卸载时用（`__devinit`/`__devexit` 自 v3.8 移除）；参数 `module_param(name, type, perm)`（类型 `int/long/charp/bool/invbool`）导出到 `/sys/module/<mod>/parameters/`；符号导出 `EXPORT_SYMBOL()` / `EXPORT_SYMBOL_GPL()`。
* `MODULE_DEVICE_TABLE(of, id_table)` 让 `depmod` 生成自动加载别名；返回 `int`：0 成功、负 errno（`-ENOMEM`/`-EINVAL`/`-EIO`/`-ENODEV`），指针错误用 `ERR_PTR()` / `IS_ERR()` / `PTR_ERR()`；日志等级 `KERN_EMERG(0)` … `KERN_DEBUG(7)`，现代首选 `pr_info()` / `dev_err(dev, ...)`。
* kbuild：`obj-m += foo.o`、`obj-$(CONFIG_X) += foo.o`、多文件模块 `foo-objs := a.o b.o`；外部模块 `KERNELDIR ?= /lib/modules/$(shell uname -r)/build`，编译 `make -C $(KERNELDIR) M=$(PWD) modules`；工具 `insmod` / `modprobe`（解析依赖）/ `rmmod` / `lsmod` / `depmod` / `modinfo`，配置 `/etc/modprobe.d/*.conf` 与 `modules.dep`。

---

## 二、 内核设施与辅助函数（第3章）

### 1. 容器宏、链表、定时器与同步
* `container_of(ptr, type, member)` 由成员指针反推结构体首地址，是内核「面向对象」的基础；`struct list_head { *next, *prev; }` 是双向环形链表（头节点为哑节点），用 `LIST_HEAD()` / `INIT_LIST_HEAD()` / `list_add_tail()` / `list_del()`，遍历 `list_for_each_entry(pos, head, member)` 与可安全删除的 `list_for_each_entry_safe(pos, n, head, member)`。
* 等待队列 `wait_event_interruptible(wq, cond)` / `wait_event_interruptible_timeout()` / `wake_up_interruptible()`（需自行处理虚假唤醒）；`struct timer_list` + `setup_timer()` / `mod_timer()` / `del_timer_sync()`，回调运行于 softirq 上下文不可睡眠；`jiffies` 与 `HZ`（常见 100/250/1000），`msecs_to_jiffies()`、`time_after/time_before` 防回绕；高精度定时器 `struct hrtimer` + `hrtimer_start(&t, ktime, HRTIMER_MODE_REL|ABS)`。
* 延时：原子上下文用 `udelay()`/`mdelay()`，可睡眠上下文首选 `usleep_range(min, max)`；互斥锁 `struct mutex` / `DEFINE_MUTEX()` / `mutex_lock_interruptible()`（可睡眠，中断上下文禁用）；自旋锁 `spinlock_t` / `spin_lock_irqsave(&lock, flags)`（持锁不可睡眠）；另有 `rwlock_t`、`seqlock_t`、RCU（`rcu_read_lock()` / `synchronize_rcu()` / `call_rcu()`）与原子操作 `atomic_t`、`test_and_set_bit()`。

### 2. 下半部与中断请求
* 演进路径：**softirq**（`open_softirq()`，10 种类型，多 CPU 可并发，`ksoftirqd` 兜底）→ **tasklet**（`DECLARE_TASKLET()` / `tasklet_schedule()` / `tasklet_kill()`，同一 tasklet 串行但不可睡眠）→ **workqueue**（`INIT_WORK()` / `schedule_work()` / `alloc_workqueue()` / `struct delayed_work`，进程上下文**可睡眠**，首选）。
* `request_irq(irq, handler, flags, name, dev)` 与 `request_threaded_irq(irq, primary_handler, thread_fn, flags, name, dev)`（primary 返回 `IRQ_WAKE_THREAD` 唤醒线程）；`irqreturn_t` 取 `IRQ_HANDLED` / `IRQ_NONE` / `IRQ_WAKE_THREAD`；flags 含 `IRQF_SHARED`（共享需各 handler 一致、`dev_id` 唯一非 NULL）、`IRQF_ONESHOT`、`IRQF_TRIGGER_RISING/FALLING/HIGH/LOW`；上下文判断 `in_interrupt()` / `in_irq()` / `in_softirq()` / `in_atomic()`。

---

## 三、 字符设备驱动（第4章）

### 1. 设备号、cdev 与 file_operations
* `dev_t` 32 位 = 高 12 位主设备号 + 低 20 位次设备号（`MINORBITS = 20`），故 major 最多 4096、minor 最多 1,048,576；宏 `MAJOR()` / `MINOR()` / `MKDEV(ma, mi)`；`alloc_chrdev_region(dev_t *dev, unsigned baseminor, unsigned count, const char *name)` 动态取号（推荐），`register_chrdev_region()` 用于已知号，`unregister_chrdev_region()` 归还，`/proc/devices` 查看；老接口 `register_chrdev()` 已不推荐。
* `struct cdev { struct kobject kobj; struct module *owner; const struct file_operations *ops; struct list_head list; dev_t dev; unsigned int count; }`；4 步注册 = `alloc_chrdev_region()` → `cdev_init(&cdev, &fops)` → `cdev_add(&cdev, dev, count)` → `class_create()` + `device_create()` 由 udev 自动生成 `/dev` 节点（取代手工 `mknod`），卸载 `cdev_del()`。
* `struct file_operations`：`struct module *owner`（填 `THIS_MODULE` 防使用中卸载）、`ssize_t (*read)(struct file *, char __user *, size_t, loff_t *)`、`ssize_t (*write)(struct file *, const char __user *, size_t, loff_t *)`、`loff_t (*llseek)`、`long (*unlocked_ioctl)(struct file *, unsigned int cmd, unsigned long arg)`、`int (*open)(struct inode *, struct file *)`、`int (*release)`、`unsigned int (*poll)(struct file *, struct poll_table_struct *)`、`int (*mmap)(struct file *, struct vm_area_struct *)`。
* `struct inode` 代表磁盘文件对象（`i_rdev`、`i_cdev`），`struct file` 代表打开实例（`f_flags`、`f_pos`、`f_op`、`private_data`），私有数据用 `container_of(inode->i_cdev, struct my_dev, cdev)` 反推。
* `copy_from_user(void *to, const void __user *from, unsigned long n)` / `copy_to_user()` 返回**未拷贝字节数**（0 为成功），配 `get_user()` / `put_user()` / `access_ok()`；`__user` 是 Sparse 注解，`make C=1` 检查，内核绝不能直接解引用用户指针。
* ioctl 命令号由 `_IO(type, nr)` / `_IOR` / `_IOW` / `_IOWR` 构造，编码 = direction(2 位) + size(14 位) + type(8 位) + nr(8 位)，用 `_IOC_DIR/_IOC_TYPE/_IOC_NR/_IOC_SIZE` 解析，32/64 位兼容需 `compat_ioctl`；`poll` 实现中用 `poll_wait(file, &wait_q, wait)` 注册等待队列并返回 `POLLIN|POLLRDNORM` / `POLLOUT`。

---

## 四、 平台设备与设备树（第5–6章）

### 1. 平台设备模型与 probe 流程
* `struct platform_device { name; id; struct device dev; num_resources; struct resource *resource; }`；`struct resource { start; end; flags; name; parent; }`，类型宏 `IORESOURCE_MEM/IO/IRQ/DMA/BUS/REG`；`struct platform_driver`：`int (*probe)(struct platform_device *)`、`int (*remove)(struct platform_device *)`、`void (*shutdown)()`、`struct device_driver driver`（`.name` / `.of_match_table` / `.acpi_match_table` / `.pm`）、`const struct platform_device_id *id_table`；注册 `platform_driver_register()` 或宏 `module_platform_driver(plat_drv)`。
* 取资源 `platform_get_resource()`、`platform_get_irq()`、`devm_ioremap_resource()`（一步完成 `request_mem_region` + `ioremap`）、`platform_set_drvdata()`；匹配优先级（`platform_match()`）：OF 匹配 `of_driver_match_device()` → ACPI 匹配 → `id_table` 匹配（填 `driver_data`）→ 名字 `strcmp`。
```
platform_driver_register()  → 驱动挂入 platform_bus_type 驱动链表
device_add() → bus_probe_device() → platform_match(dev, drv)
        │ 匹配成功
        ▼
really_probe() → dev->driver = drv → drv->probe(pdev)
        ├─ platform_get_resource() / platform_get_irq()   取资源
        ├─ devm_ioremap_resource()                        映射寄存器
        ├─ devm_request_irq()                             注册中断
        └─ platform_set_drvdata(pdev, priv)               存私有数据
```

### 2. 设备树语法与 OF API
* 类型 `.dts`（源）/ `.dtsi`（可包含头）/ `.dtb`（二进制 blob）/ `dtc`（编译器）；节点语法 `label: node-name@unit-address { prop = value; };`；`compatible = "vendor,device"` 是驱动匹配唯一依据；`reg = <addr size>` 中地址/长度各占几个 cell 由 `#address-cells` / `#size-cells` 决定（常 1/1 或 2/1）；布尔属性写作 `prop;`。
* 常见属性 `status = "okay"/"disabled"`、`interrupt-parent`、`interrupts`、`interrupt-controller`、`#interrupt-cells`、`clocks`/`clock-names`、`dmas`/`dma-names`、`gpios`、`pinctrl-0`/`pinctrl-names`、`<name>-supply`、`#gpio-cells`、`#pwm-cells`；`simple-bus` 表示子节点会被枚举成 `platform_device`。
* `struct of_device_id { char name[32]; char type[32]; char compatible[128]; const void *data; }` + `MODULE_DEVICE_TABLE(of, ...)` + `of_match_device()`；读属性 `of_property_read_u32(np, "prop", &val)`、`of_property_read_u32_array()`、`of_property_read_string()`、`of_property_read_bool()`；查找 `of_find_compatible_node()`、`for_each_child_of_node()`；地址 `of_address_to_resource()`、`of_iomap()`；枚举 `of_platform_populate()`；`of_node_put()` 引用计数不可漏。

---

## 五、 I2C 与 SPI 客户端驱动（第7–8章）

### 1. I2C 客户端驱动
* 两线（SCL/SDA）总线，主控制器（adapter）由 SoC 提供；`struct i2c_driver`：`int (*probe)(struct i2c_client *, const struct i2c_device_id *)`、`int (*remove)(struct i2c_client *)`、`struct device_driver driver`（`.name` / `.of_match_table`）、`const struct i2c_device_id *id_table`、`int (*detect)()`；注册 `i2c_add_driver()` 或宏 `module_i2c_driver()`；`struct i2c_client` 含 `unsigned short addr`（7 位地址）、`struct i2c_adapter *adapter`、`struct device dev`、`int irq`，私有数据 `i2c_set_clientdata()` / `i2c_get_clientdata()`。
* **注意**内核 4.10 前即使只用 OF 匹配也必须提供 `i2c_device_id` 数组（`probe` 第二参数非空），4.10 起允许 NULL `id_table`；传输 `i2c_transfer(struct i2c_adapter *adap, struct i2c_msg *msgs, int num)`，`struct i2c_msg { __u16 addr; __u16 flags; __u16 len; __u8 *buf; }`，flags 含 `I2C_M_RD` / `I2C_M_TEN` / `I2C_M_NOSTART`；简化 `i2c_master_send()` / `i2c_master_recv()`；SMBus `i2c_smbus_read_byte_data()` / `i2c_smbus_write_word_data()` / `i2c_smbus_read_i2c_block_data()`；能力检查 `i2c_check_functionality(adapter, I2C_FUNC_I2C | I2C_FUNC_SMBUS_BYTE_DATA)`，不满足返回 `-EIO`。
* DT：客户端节点挂在 I2C 控制器下，`reg = <0xNN>` 为从机地址，`clock-frequency` 写在控制器节点。

### 2. SPI 设备驱动
* 四线（SCLK/MOSI/MISO/CS）全双工同步串行总线，无地址、靠片选；`SPI_CPHA = 0x01`（相位）+ `SPI_CPOL = 0x02`（极性）组合出 `SPI_MODE_0` … `SPI_MODE_3`，另有 `SPI_CS_HIGH`、`SPI_LSB_FIRST`、`SPI_3WIRE`；`struct spi_device` 含 `struct spi_master *master`、`u32 max_speed_hz`、`u8 chip_select`、`u8 bits_per_word`、`u8 mode`、`int irq`；`struct spi_driver` 的 `probe(struct spi_device *)` + `module_spi_driver()`；`spi_set/get_drvdata()`、`spi_setup(spi)` 应用速率/模式/位宽。
* `struct spi_transfer { const void *tx_buf; void *rx_buf; unsigned len; u32 speed_hz; u8 bits_per_word; u16 delay_usecs; u32 cs_change; }`；流程 `spi_message_init()` → `spi_message_add_tail(&xfer, &msg)` → `spi_sync(spi, &msg)`（阻塞可睡眠）或 `spi_async()`（原子上下文）；最常用封装 `spi_write_then_read(spi, txbuf, n_tx, rxbuf, n_rx)`，半双工读时 `tx_buf = NULL`。
* DT：`reg = <cs_index>`、`spi-max-frequency`、`spi-cpol`、`spi-cpha`、`spi-cs-high`；用户空间 `/dev/spidevB.C`，ioctl `SPI_IOC_MESSAGE(N)` / `SPI_IOC_RD_MODE` / `SPI_IOC_RD_MAX_SPEED_HZ`，参数 `struct spi_ioc_transfer`。

---

## 六、 Regmap 与 IIO 子系统（第9–10章）

### 1. Regmap 寄存器映射抽象
* 目的：统一 I2C/SPI/MMIO 的寄存器读写，消除各总线重复代码（内核 3.1 引入）；`struct regmap_config` 关键字段 `int reg_bits`（寄存器地址位宽）、`int val_bits`（值位宽）、`int max_register`、`bool (*writeable_reg)()` / `(*readable_reg)()` / `(*volatile_reg)()`、`enum regcache_type cache_type`、`const struct reg_default *reg_defaults`、`bool fast_io`（禁用锁，仅限无并发场景）。
* 构造 `regmap_init_i2c(client, &config)` / `regmap_init_spi()` / `regmap_init_mmio(dev, base, &config)`，推荐 `devm_regmap_init_i2c/spi/mmio()`，销毁 `regmap_exit()`；访问 `regmap_read(map, reg, &val)`、`regmap_write()`、`regmap_update_bits(map, reg, mask, val)`（读改写，最常用）、`regmap_bulk_read/write()`、`regmap_multi_reg_write(map, struct reg_sequence *regs, int num)`；缓存类型 `REGCACHE_NONE/RBTREE/COMPRESSED/FLAT`，挂起恢复用 `regcache_mark_dirty()` + `regcache_sync()`，debugfs 见 `/sys/kernel/debug/regmap/<dev>/`。

### 2. IIO 工业 I/O 子系统
* 面向 ADC/DAC/传感器（加速度计、陀螺仪、光/温度）的统一框架，支持 sysfs 一次性读取与高速缓冲采集；`struct iio_dev` 含 `modes`、`const struct iio_chan_spec *channels`、`num_channels`、`const struct iio_info *info`、`scan_bytes`、`active_scan_mask`、`scan_timestamp`、`struct iio_trigger *trig`、`pollfunc`、`setup_ops`。
* `struct iio_info`：`int (*read_raw)(struct iio_dev *, const struct iio_chan_spec const *, int *val, int *val2, long mask)` 与 `write_raw`；`struct iio_chan_spec` 含 `enum iio_chan_type type`、`int channel`、`int scan_index`、`info_mask_separate`、`scan_type`（`sign`/`realbits`/`storagebits`/`shift`/`endianness`）；注册 `devm_iio_device_alloc(&dev, sizeof(*priv))` → `iio_priv()` → 填 `modes`/`channels`/`info`/`name` → `devm_iio_device_register()`；模式 `INDIO_DIRECT_MODE` / `INDIO_BUFFER_TRIGGERED`，`read_raw` 返回 `IIO_VAL_INT` / `IIO_VAL_INT_PLUS_MICRO`，`mask` 用 `IIO_CHAN_INFO_RAW` / `_SCALE` / `_OFFSET`。
* 触发缓冲 `iio_triggered_buffer_setup()` → pollfunc 按 `for_each_set_bit(bit, active_scan_mask, masklength)` 逐通道读取 → `iio_push_to_buffers_with_timestamp()` → `iio_trigger_notify_done()`；缓冲使能时 `read_raw` 应返回 `-EBUSY`；用户空间 `/sys/bus/iio/devices/iio:deviceX/` 下 `in_voltage3_raw` × `in_voltage_scale` 得物理量，`scan_elements/in_voltage3_type` 形如 `le:s12/16>>4`（小端、有符号 12 位真值、16 位存储、右移 4），缓冲读取走 `/dev/iio:deviceX`。

---

## 七、 内核内存管理与 DMA（第11–12章）

### 1. 地址空间、分配器与 MMIO
* `PAGE_SIZE = 4096`、`PAGE_SHIFT = 12`；`struct page` 是物理页描述符，配 `page_to_pfn()` / `pfn_to_page()` / `page_address()`；`__pa(x)` / `__va(x)` 仅对直接映射区有效；内存区 `ZONE_DMA`（<16MB，老 ISA DMA）/ `ZONE_NORMAL` / `ZONE_HIGHMEM`（32 位系统高于约 896MB 区域）。
* 页分配 `alloc_pages(gfp_t gfp_mask, unsigned int order)` / `__get_free_pages()` / `get_zeroed_page()`，`get_order(size)` 换算阶数（最大 order 通常 10，即 4MB）；GFP 标志 `GFP_KERNEL`（可睡眠）/ `GFP_ATOMIC`（不可睡眠）/ `__GFP_ZERO`；伙伴系统按 2 的幂合并空闲页；slab 三级结构 cache → slab → 对象，接口 `kmem_cache_create()` / `kmem_cache_alloc()` / `kmem_cache_free()`，实现有 SLOB/SLAB/SLUB（现代默认）。
* `kmalloc(size_t size, gfp_t flags)` / `kzalloc()` / `kcalloc()` / `kfree()` 分配物理连续内存，单次上限约 4MB；`vmalloc()` / `vzalloc()` / `vfree()` 虚拟连续而物理不连续，仅用于大块内存（`/proc/vmallocinfo`）；进程地址空间 `struct mm_struct`（`pgd`、`mmap`、`mmap_base`、`mmap_sem`）与 `struct vm_area_struct`，`find_vma(mm, addr)` 查找，缺页由 `do_page_fault()` 处理、写时复制触发 `do_wp_page()`。
* `request_mem_region(start, len, name)` / `release_mem_region()`（`/proc/iomem` 可见）；`ioremap(phys_addr, size)` 返回 `void __iomem *`，推荐 `devm_ioremap_resource()`，`iounmap()` 释放；访问 `ioread8/16/32()`、`iowrite8/16/32()`、`memcpy_fromio()` / `memcpy_toio()`（取代旧的 `readl/writel`），`__iomem` 为 Sparse 注解；mmap 到用户空间用 `int remap_pfn_range(struct vm_area_struct *vma, unsigned long addr, unsigned long pfn, unsigned long size, pgprot_t prot)`，映射前设 `vma->vm_flags |= VM_IO | VM_DONTEXPAND | VM_DONTDUMP;` 并用 `pgprot_noncached()` 关缓存。
* devres 托管资源 `devm_kzalloc()` / `devm_request_irq()` / `devm_clk_get()` / `devm_regulator_get()` 随设备解绑自动释放，是驱动推荐写法。

### 2. DMA 直接内存访问
* **一致性映射**（缓存一致性由硬件保证，适合长期描述符环）：`void *dma_alloc_coherent(struct device *dev, size_t size, dma_addr_t *dma_handle, gfp_t flag)` / `dma_free_coherent()` / `dmam_alloc_coherent()`，ARM 另有 `dma_alloc_writecombine()`。
* **流式映射**（对已有缓冲区临时建映射，用完必须 unmap/sync）：`dma_map_single(dev, ptr, size, direction)` → `dma_addr_t`、`dma_unmap_single()`；`enum dma_data_direction { DMA_BIDIRECTIONAL = 0, DMA_TO_DEVICE = 1, DMA_FROM_DEVICE = 2, DMA_NONE = 3 }`；`dma_sync_single_for_cpu()` / `dma_sync_single_for_device()` 传递所有权；散射聚集 `struct scatterlist` + `sg_init_table()` / `dma_map_sg(dev, sg, nents, dir)`（返回实际段数）；缓冲区不可位于 vmalloc 或 HIGHMEM，且需 cache line 对齐；完成通知用 `struct completion` + `wait_for_completion()` / `complete()`。
```
1. 申请通道  dma_cap_zero(&mask); dma_cap_set(DMA_SLAVE|DMA_MEMCPY, mask);
             dma_request_channel(&mask, dma_filter_fn, fn_param)
             DT 方式: dma_request_slave_channel(dev, "rx")
2. 配置通道  dmaengine_slave_config(chan, &dma_slave_config{direction,
                 src_addr, dst_addr, src_addr_width, src_maxburst, ...})
3. 取描述符  dmaengine_prep_slave_sg() / dmaengine_prep_dma_cyclic()
             tx->callback = dma_callback; flags |= DMA_PREP_INTERRUPT
4. 提交事务  dmaengine_submit(tx) -> dma_cookie_t
5. 发起传输  dma_async_issue_pending(chan)
      设备搬运完成 -> DMA 完成中断 -> callback -> complete(&done)
```
* 枚举 `enum dma_transaction_type { DMA_MEMCPY, DMA_XOR, DMA_SLAVE, DMA_CYCLIC, ... }`；`enum dma_transfer_direction { DMA_MEM_TO_MEM, DMA_MEM_TO_DEV, DMA_DEV_TO_MEM, DMA_DEV_TO_DEV }`；DT 绑定 `dmas = <&sdma 3 4 1>; dma-names = "rx";`，控制器用 `#dma-cells` 声明 cell 数。

---

## 八、 Linux 设备模型 LDM（第13章）

### 1. 三层抽象与层次结构
* `struct bus_type`：`const char *name`、`int (*match)(struct device *dev, struct device_driver *drv)`、`int (*probe)(struct device *)`、`int (*remove)()`、`void (*shutdown)()`、`const struct attribute_group **bus_groups/dev_groups/drv_groups`；`bus_register()` / `bus_for_each_dev()` / `bus_for_each_drv()`。
* `struct device_driver`：`name`、`bus`、`owner`、`const struct of_device_id *of_match_table`、`const struct acpi_device_id *acpi_match_table`、`probe` / `remove` / `shutdown` / `suspend/resume`、`groups`、`struct dev_pm_ops *pm`；`struct device`：`parent`、`struct kobject kobj`、`bus`、`driver`、`void *platform_data`、`void *driver_data`、`struct device_node *of_node`、`struct class *class`、`void (*release)(struct device *)`——**`release()` 回调必需**，否则内核告警。
```
/sys
├── bus/      bus_type  -> device/ 与 driver/ 子目录（匹配在此发生）
├── class/    class     -> 设备分类，配合 device_create() 生成 /dev 节点
├── devices/  device    -> 按总线与物理拓扑组织的设备树
├── dev/                -> 按 dev_t 索引的字符/块设备
└── module/             -> 已加载模块及其参数

  device 注册 ─┐
               ├─► bus_type->match() ─► 匹配成功 ─► driver->probe()
  driver 注册 ─┘        │
                  kobject + attribute 导出为 sysfs 属性
```

### 2. kobject、sysfs 与 class
* `struct kobject { const char *name; struct list_head entry; struct kobject *parent; struct kset *kset; struct kobj_type *ktype; struct kref kref; }`，接口 `kobject_init_and_add()` / `kobject_create_and_add()` / `kobject_get()` / `kobject_put()`；`struct kobj_type { void (*release)(struct kobject *); const struct sysfs_ops *sysfs_ops; struct attribute **default_attrs; }`；`struct sysfs_ops` 提供 `show()` / `store()`；`struct kset` 用 `kset_create_and_add()` 管理同类 kobject 集合。
* 属性 `struct attribute { const char *name; umode_t mode; }` + `sysfs_create_file()` / `sysfs_remove_file()`；`struct attribute_group` 配 `ATTRIBUTE_GROUPS()` 批量注册；层级专用宏 `DEVICE_ATTR(_name, _mode, _show, _store)`（生成 `dev_attr_<name>`，另有 `DEVICE_ATTR_RW/RO/WO`）、`BUS_ATTR()`、`DRIVER_ATTR()`、`CLASS_ATTR()`；`class_create(owner, name)` + `device_create(cls, parent, devt, drvdata, fmt, ...)` 自动创建 `/dev` 节点。

---

## 九、 GPIO、引脚控制与中断子系统（第14–16章）

### 1. Pin control 与 GPIO
* DT 中 `pinctrl-names = "default", "sleep"` 对应 `pinctrl-0` / `pinctrl-1`；引脚组定义如 i.MX `&iomuxc { pinctrl_xxx: xxxgrp { fsl,pins = <MX6QDL_PAD_... 0x...>; }; };`（`MX6QDL_PAD_*` 宏来自 `imx6qdl-pinfunc.h`）；API `pinctrl_get()` / `pinctrl_lookup_state(p, "default")` / `pinctrl_select_state()` / `devm_pinctrl_get_select_default()`，常量 `PINCTRL_STATE_DEFAULT/IDLE/SLEEP`，默认状态通常由内核自动应用。
* 描述符接口（新，推荐）：`struct gpio_desc`、`devm_gpiod_get(dev, "name", flags)`、`devm_gpiod_get_optional()`；`enum gpiod_flags { GPIOD_ASIS = 0, GPIOD_IN, GPIOD_OUT_LOW, GPIOD_OUT_HIGH, ... }`；DT 中依次尝试 `<name>-gpios`、`<name>-gpio`；`gpiod_direction_input()` / `gpiod_direction_output(desc, val)`、`gpiod_get_value()` / `gpiod_set_value()`、`gpiod_set_debounce(desc, usec)`、`gpiod_to_irq(desc)` → 传给 `request_threaded_irq()`；可能睡眠的设备（I2C 扩展）必须用 `gpiod_get/set_value_cansleep()`；传统整数接口 `gpio_request()` / `gpio_free()` / `gpio_to_irq()` / `of_get_named_gpio(np, "prop", index)` 逐步弃用。
* GPIO 控制器 `struct gpio_chip`：`label`、`dev`、`owner`、`of_node`、回调 `request` / `free` / `direction_output(chip, offset, int value)` / `get` / `set` / `set_multiple` / `set_debounce` / `to_irq`、编号 `int base`（负值表示动态分配）、`u16 ngpio`、标志 `bool can_sleep`；注册 `devm_gpiochip_add_data()`；DT 需 `gpio-controller;` + `#gpio-cells = <2>;`，支持中断再加 `interrupt-controller;` + `#interrupt-cells`（案例 MCP23016，16 个 GPIO，`can_sleep = true`）。

### 2. 高级中断管理
* `struct irq_chip { name; irq_mask/unmask/ack/eoi/set_type/set_wake/set_affinity; }`；`struct irq_domain { struct device_node *of_node; const struct irq_domain_ops *ops; irq_hw_number_t hwirq_max; void *host_data; }`，`struct irq_domain_ops { int (*map)(domain, virq, hwirq); void (*unmap)(); int (*xlate)(...); }`；建域与映射 `irq_domain_add_linear(node, size, ops, host_data)`、`irq_create_mapping(domain, hwirq)`、`irq_find_mapping(domain, hwirq)`；DT 解析辅助 `irq_domain_xlate_twocell()`；设置 `irq_set_chip_and_handler(virq, chip, handle_edge_irq|handle_level_irq)`。
* 分发路径 = 硬件 IRQ → CPU 异常向量 → `handle_arch_irq`（ARM 通常 `gic_handle_irq`）→ `handle_domain_irq()` / `generic_handle_irq()` → `struct irq_desc` → action 链；**链式（chained）**：父控制器 handler 直接 `generic_handle_irq()` 调子控制器，仍在硬中断上下文，子 handler 必须快且不可睡眠；**嵌套（nested）**：父 handler 调 `handle_nested_irq(virq)`，子设备 handler 在线程化上下文执行**可睡眠**，专用于 I2C/SPI 扩展 GPIO；配 `irq_set_chained_handler_and_data()`、`irq_set_nested_thread(irq, 1)`。
* `request_any_context_irq()` 自动选择硬中断或嵌套线程化（返回 `IRQC_IS_HARDIRQ` / `IRQC_IS_NESTED`）；gpiolib irqchip 封装 `gpiochip_irqchip_add()`、`gpiochip_set_chained_irqchip()`，v4.10+ 增加 `gpiochip_irqchip_add_nested()`、`gpiochip_set_nested_irqchip()`。

---

## 十、 输入、RTC、PWM、稳压器、帧缓冲与网卡（第17–22章）

### 1. Input、RTC、PWM 与 Regulator
* `struct input_dev`：`name`、`phys`、`uniq`、`struct input_id id`（`bustype/vendor/product/version`）、能力位图 `evbit[]` / `keybit[]` / `relbit[]` / `absbit[]`、`int (*open)()` / `void (*close)()`、`unsigned int users`；注册 `devm_input_allocate_device()` → `set_bit(EV_KEY, dev->evbit)` + `set_bit(KEY_POWER, dev->keybit)` → 绝对轴 `input_set_abs_params(dev, ABS_X, min, max, fuzz, flat)` → `input_register_device()`；上报 `input_report_key()` / `input_report_rel()` / `input_report_abs()`，**一组事件后必须调用 `input_sync(dev)`**，否则用户空间收不到完整报告；事件类型 `EV_KEY` / `EV_REL` / `EV_ABS` / `EV_MSC` / `EV_LED` / `EV_REP`；轮询设备 `input_allocate_polled_device()` + 填 `poll` / `poll_interval`（默认 500ms）+ `input_register_polled_device()`；用户空间 `/dev/input/event<X>` 读 `struct input_event { struct timeval time; __u16 type; __u16 code; __s32 value; }`。
* RTC：`struct rtc_time`（`tm_year` 为自 1900 起的年数、`tm_mon` 从 0 起），`rtc_valid_tm()` 校验、`bcd2bin()` / `bin2bcd()` 处理 BCD 寄存器；`struct rtc_class_ops` 含 `read_time` / `set_time` / `read_alarm` / `set_alarm` / `alarm_irq_enable`，注册 `devm_rtc_device_register()`；闹钟 `struct rtc_wkalrm { unsigned char enabled; unsigned char pending; struct rtc_time time; }`；中断上报 `rtc_update_irq(rtc, num, events)`，事件位必须含 `RTC_IRQF = 0x80`，另有 `RTC_PF = 0x40`（周期）/ `RTC_AF = 0x20`（闹钟）/ `RTC_UF = 0x10`（更新）；sysfs `/sys/class/rtc/rtc<id>/` 有 `date` / `time` / `wakealarm`，开机同步用 `CONFIG_RTC_HCTOSYS` + `CONFIG_RTC_HCTOSYS_DEVICE="rtc0"`。
* PWM：`Ton` / `Toff` / `Period = Ton + Toff` / `Duty cycle = Ton / Period`；`struct pwm_chip { dev; ops; int base; unsigned int npwm; of_xlate; of_pwm_n_cells; bool can_sleep; }`，`struct pwm_ops` 含 `request` / `free` / `int (*config)(chip, pwm, int duty_ns, int period_ns)` / `set_polarity` / `enable` / `disable` / `get_state`（v4.7+）；注册 `devm_pwmchip_add()`（被占用时 `pwmchip_remove()` 返回 `-EBUSY`）；消费者 `devm_pwm_get(dev, con_id)` + `pwm_config()` / `pwm_enable()` / `pwm_disable()`，v4.7+ 统一为 `pwm_apply_state(pwm, &state)` 配 `struct pwm_state { u64 period, duty_cycle; enum pwm_polarity polarity; bool enabled; }`；DT `pwms = <&pwm1 0 5000000 0>; pwm-names = "backlight";`，控制器 `#pwm-cells = <3>`。
* Regulator 生产者：`struct regulator_desc` + `struct regulator_ops { list_voltage/set_voltage_sel/get_voltage_sel/enable/disable/is_enabled/set_mode }` + `struct regulation_constraints`（`min_uV` / `max_uV` / `always_on` / `valid_ops_mask`），注册 `devm_regulator_register()`，DT 数据用 `of_regulator_match()` 提取（`.name` 必须与 DT 节点 label 完全一致）；消费者 `devm_regulator_get(dev, id)`（`id` 必须匹配 DT 的 `<name>-supply`）、`regulator_enable()` / `regulator_disable()` / `regulator_is_enabled()`、`regulator_set_voltage(reg, min_uV, max_uV)` / `regulator_get_voltage()`、`regulator_set_optimum_mode(reg, load_uA)`（案例 ISL6271A：1 个可变 Core Buck + 2 个固定 LDO）。

### 2. Framebuffer 与网卡驱动
* 帧缓冲四结构（`include/linux/fb.h`）：`struct fb_var_screeninfo`（可变：`xres`/`yres`/`xres_virtual`/`bits_per_pixel`/`pixclock`/`left_margin`/`red/green/blue` 各含 `length`/`offset`）、`struct fb_fix_screeninfo`（固定：`smem_start`/`smem_len`/`line_length`/`visual`）、`struct fb_cmap`、`struct fb_info`（`var`/`fix`/`cmap`/`fbops`/`screen_base`/`screen_size`/`par`）。
* 分配注册 `framebuffer_alloc(size, dev)`（size 为追加在 `fb_info` 之后的私有区，经 `info->par` 访问）/ `framebuffer_release()`，`register_framebuffer()` / `unregister_framebuffer()`（先 unregister 再 release）；`struct fb_ops` 含 `fb_open` / `fb_release` / `fb_read` / `fb_write` / `fb_check_var`（只校验调整用户参数，**不得修改 par**）/ `fb_set_par`（把 `info->var` 写进硬件）/ `fb_setcolreg` / `fb_blank` / `fb_pan_display` / `fb_fillrect` / `fb_copyarea` / `fb_imageblit` / `fb_mmap`；无硬件加速时用软件回退 `cfb_fillrect()` / `cfb_copyarea()` / `cfb_imageblit()`；注意 `fb_ops` 是硬件抽象层、`file_operations` 才是系统调用层（核心实现在 `drivers/video/fbdev/core/fbmem.c` 并转发到 `fb_ops`）；用户空间 `ioctl(FBIOGET_VSCREENINFO, &var)` + `ioctl(FBIOGET_FSCREENINFO, &fix)` → `mmap()` 显存，像素偏移 `pos = (x + xoffset) * (bpp/8) + (y + yoffset) * line_length`。
* 网卡不走 `/dev`，而经 `struct net_device` 接入网络栈（头文件 `<linux/skbuff.h>` / `<linux/netdevice.h>` / `<linux/etherdevice.h>` / `<linux/ethtool.h>`）；`struct sk_buff` 含 `head`/`data`/`tail`/`end`/`len`/`data_len`/`struct net_device *dev`/`atomic_t users`，分配三步 `netdev_alloc_skb(dev, length)` → `skb_reserve(skb, NET_IP_ALIGN)` → `skb_put(skb, len)`（`NET_IP_ALIGN = 2`）。
* `struct net_device` 含 `char name[IFNAMSIZ]`、`int irq`、`struct net_device_stats stats`、`const struct net_device_ops *netdev_ops`、`const struct ethtool_ops *ethtool_ops`、`unsigned char *dev_addr`、`struct phy_device *phydev`；分配注册 `alloc_etherdev(sizeof_priv)` + `netdev_priv(dev)` + `register_netdev()`，释放顺序 `unregister_netdev()` → `free_irq()` → `free_netdev()`。
* `struct net_device_ops`：`ndo_open` / `ndo_stop`、`netdev_tx_t (*ndo_start_xmit)(struct sk_buff *skb, struct net_device *dev)`（**唯一必需**，成功返回 `NETDEV_TX_OK`，忙则 `NETDEV_TX_BUSY`——此时 SKB 仍属网络队列层，不得修改或释放）、`ndo_set_rx_mode`、`ndo_change_mtu`、`ndo_tx_timeout`、`ndo_get_stats`；收包路径 = 收包中断 → `netdev_alloc_skb(ndev, len + NET_IP_ALIGN)` → `skb_reserve()` → 数据拷入 `skb_put()` 返回指针 → `skb->protocol = eth_type_trans(skb, ndev)` → 更新 `stats.rx_packets/rx_bytes` → `netif_rx_ni(skb)`（原子上下文用 `netif_rx()`）；发包时 `ndo_start_xmit` 中 `netif_stop_queue(dev)` → 写设备 FIFO → 触发发送，完成中断里累加 `stats.tx_packets` 后 `netif_wake_queue(dev)`；SPI 等慢总线必须把收发交给 workqueue（`spi_sync()` 会睡眠），MMIO 设备可直接在硬中断中完成；NAPI = 中断 + 高流量轮询的混合模式，硬件支持时应采用。

---

## 核心设计思想提炼

1. **设备与驱动分离，总线负责撮合**：设备描述「有什么资源」（`platform_device` + `struct resource` / DT 节点），驱动描述「怎么用」（`platform_driver` + `probe()`），`bus_type->match()` 是二者相遇的唯一入口；`probe()` 承担全部资源申请，`remove()` 严格逆序释放。
2. **硬件描述数据化**：设备树把板级硬编码（板文件、`i2c_board_info`、`spi_board_info`）转成可被 bootloader 传递的 `.dtb` 数据，`compatible` 成为匹配契约，`of_platform_populate()` 让内核自动枚举设备。
3. **上下文决定 API 选择**：硬中断/softirq/tasklet 上下文不可睡眠，只能用 `GFP_ATOMIC`、非阻塞路径与 `spi_async()`；慢总线（I2C/SPI）操作必须退到 workqueue、threaded IRQ 或嵌套中断——这正是「链式 vs 嵌套中断」「工作队列」存在的根本原因。
4. **统一抽象屏蔽总线差异**：regmap 抹平 I2C/SPI/MMIO 的寄存器访问差异，gpiolib 抹平片上与扩展 GPIO 的差异，IIO/Input/RTC/PWM/Regulator/Framebuffer 为同类外设提供统一用户空间 ABI——驱动只填 `ops` 结构体，框架负责其余一切。
5. **托管资源优先（devres）**：`devm_*` 系列把资源生命周期绑定到 `struct device`，使 `remove()` 与错误回滚路径不再需要手写释放代码，是内核 4.x 驱动的主流写法。
6. **现代 API 的演进方向**：`module_*_driver()` 宏取代手写 init/exit、描述符 GPIO（`gpiod_*`）取代整数 GPIO、`ioread*/iowrite*` 取代 `readl/writel`、`alloc_chrdev_region()` + cdev 取代 `register_chrdev()`、OF 匹配取代名字匹配。
7. **典型陷阱清单**：忘记 `input_sync()` 导致事件不送达；未提供 `device.release()`；`ndo_start_xmit` 返回 `NETDEV_TX_BUSY` 时误操作 SKB；`regulator_get()` 的 id 与 DT `<name>-supply` 不匹配；`of_regulator_match()` 的 `.name` 与 DT label 不一致；`fb_check_var()` 中误改 `par`；4.10 前 I2C 驱动漏写 `id_table`。
