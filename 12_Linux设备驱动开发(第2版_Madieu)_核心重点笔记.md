# 《Linux 设备驱动开发》（原书第2版 · John Madieu · 内核 v5.10）核心重点笔记

> **书名**：Linux Device Driver Development, Second Edition  
> **作者**：John Madieu（法国嵌入式 Linux / 内核工程师，LABCSMART 创始人）  
> **出版社**：Packt Publishing，2022 年 3 月第 2 版（第 1 版 2017 年 10 月）  
> **内核基准**：Linux v5.10（同时在 x86 PC 与 NXP i.MX6 的 UDOO QUAD 上实测）  
> **定位**：第 1 版的**全面重构与现代化升级**。全书由 22 章精简重组为 **4 大板块、17 章**（基础 / 平台抽象与设备驱动 / 榨取硬件 / 嵌入式杂项子系统），新增设备树 Overlay、Regmap、独立 IRQ 框架、libgpiod、libiio、DMA 引擎等内容，并系统性地把「老写法 → 新写法」的迁移路径贯穿始终——**tasklet 废弃改用 threaded IRQ、`create_workqueue` → `alloc_workqueue`（cmwq）、整型 GPIO → 描述符 GPIO、sysfs GPIO → GPIO 字符设备 + libgpiod、I2C `probe` → `probe_new`**。

---

## 一、 构建体系与模块基础（第1–2章）

### 1. 源码、构建与版本模型
* **Kconfig + Kbuild 双层**：`obj-m` / `obj-y` / `obj-$(CONFIG_XXX)`、`ccflags-y`；产物 `Module.symvers`（符号 CRC，跨模块解析的前提）、`modules.order`、`modules.builtin`，以及由 `depmod` 生成的 `modules.alias` / `modules.dep` / `modules.symbols`。
* **make 目标**：`defconfig` / `menuconfig` / `xconfig` / `savedefconfig` / `oldconfig` / `olddefconfig` / `localmodconfig`；`make -jN`；`make modules_install`。
* **树外模块**：`make -C $KERNEL_SRC M=$(shell pwd) [modules|modules_install|clean]`；交叉编译须同时指定 `ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf-`；Yocto 环境用 `STAGING_KERNEL_DIR`。
* **目录**：`arch/ drivers/ fs/ kernel/ mm/ include/linux/ init/`。
* **版本史**：2.6 前奇偶号（奇=开发，偶=稳定）→ 2.6.x 语义化版本 → **2.6.39 直接跳到 3.0（2011，结束语义化）→ 3.20 跳到 4.0** → 此后任意 `X.Y` 编号。
* **分支模型**：mainline（Linus 的 master，每 2–3 个月一发）/ linux-stable（`<A.B>.y`，Greg Kroah-Hartman 维护）/ LTS；**补丁必须先进入 Linus 的树才会被 stable 接受**，未 upstream 的树外改动永远不会进 stable。

### 2. 模块骨架与元数据
* `module_init(fn)` / `module_exit(fn)`；`__init` / `__exit` 宏把代码放进 `.init.text` / `.exit.text` 段（init 段在初始化后被回收）。
* `MODULE_LICENSE("GPL")`（缺失会 taint 内核）、`MODULE_AUTHOR`、`MODULE_DESCRIPTION`、`MODULE_VERSION`、`MODULE_PARM_DESC`。
* **符号导出**：`EXPORT_SYMBOL`（任意模块可用）vs `EXPORT_SYMBOL_GPL`（仅 GPL 模块可用）。
* **模块参数**：`module_param(name, type, perm)`、`module_param_array`（类型 `bool/int/uint/charp/…`），暴露于 `/sys/module/<mod>/parameters/`。
* **设备表**：`MODULE_DEVICE_TABLE(of|i2c|spi|platform|usb, table)` → 生成 `modules.alias`，支撑 `modprobe` 自动加载。
* **错误指针**：`ERR_PTR` / `IS_ERR` / `PTR_ERR`；**日志**：`pr_fmt` + `pr_info/pr_err`、`dev_info/dev_err`，`/proc/sys/kernel/printk` 默认 `4 4 1 7`（console_loglevel / default_message_loglevel / minimum / boot）。

---

## 二、 内核核心辅助设施：锁、时间与下半部（第3章）

### 1. 锁、等待队列与延时
* **spinlock**：`spinlock_t`、`DEFINE_SPINLOCK`、`spin_lock_irqsave` / `spin_unlock_irqrestore`、`spin_lock_bh`——**不可睡眠**。
* **mutex**：`DEFINE_MUTEX`、`mutex_lock` / `mutex_lock_interruptible` / `mutex_lock_killable` / `mutex_trylock`——可睡眠，仅进程上下文。
* **等待队列**：`wait_queue_head_t`、`DECLARE_WAIT_QUEUE_HEAD`、`wait_event[_interruptible][_timeout]`、`wake_up[_interruptible][_all]`，配 `msecs_to_jiffies` / `usecs_to_jiffies`。
* **延时**：忙等 `udelay` / `ndelay` / `mdelay`；睡眠 `msleep` / `usleep_range`。
* **时间基准**：`jiffies`、`HZ`（x86 默认 250，ARM 常见 100/250）；clocksource / clockevent / tick device 三层抽象。

### 2. 定时器与工作延迟机制
* **标准定时器**：`struct timer_list` + `timer_setup()` / `mod_timer()` / `del_timer_sync()` / `timer_pending()` / `from_timer()`，回调运行于 **softirq 上下文**。
* **高精度定时器**：`struct hrtimer`、`ktime_t`、`hrtimer_init` / `hrtimer_start` / `hrtimer_forward_now`；需 `CONFIG_HIGH_RES_TIMERS`。
* **softirq**：`struct softirq_action`、`open_softirq`、`raise_softirq[_irqoff]`、`__do_softirq`、`ksoftirqd` 兜底；v5.10 共 **10 个向量**（HI / TIMER / NET_TX / NET_RX / BLOCK / BLOCK_IOPOLL / TASKLET / SCHED / HRTIMER / RCU），约束 `MAX_SOFTIRQ_RESTART = 10`、`MAX_SOFTIRQ_TIME = 2ms`。
* **tasklet（已废弃）**：书中明言 *"Tasklet API is deprecated, and you should consider using threaded IRQs instead"*；`struct tasklet_struct`（新增 `use_callback` 字段）、`DECLARE_TASKLET[_OLD|_DISABLED]`、`tasklet_setup` / `tasklet_schedule` / `tasklet_kill` / `tasklet_disable` / `tasklet_enable`。
* **workqueue + cmwq（现代方式）**：`struct work_struct`、`struct delayed_work`、`struct workqueue_struct`；`DECLARE_WORK`、`INIT_WORK`、`queue_work`、`schedule_work`、`schedule_delayed_work`、`system_wq`。
* **旧 → 新**：`create_workqueue(name)`（旧）→ **`alloc_workqueue(name, flags, max_active)`**（新，并发管理工作队列 cmwq）；`alloc_ordered_workqueue`；标志 `WQ_UNBOUND` / `WQ_MEM_RECLAIM` / `WQ_FREEZABLE` / `WQ_HIGHPRI` / `WQ_CPU_INTENSIVE`。
* **threaded IRQ 是 tasklet 的现代替代**：`request_threaded_irq(irq, handler, thread_fn, flags, name, dev)`。

---

## 三、 字符设备驱动（第4章）

### 1. 设备号、cdev 与注册链
* `dev_t` = **12 位 major + 20 位 minor**（`MINORBITS = 20`）；宏 `MAJOR` / `MINOR` / `MKDEV`。
* **分配/释放**：`alloc_chrdev_region(&dev, 0, count, name)`（动态取号，**推荐**）vs `register_chrdev_region`（静态指定）；`unregister_chrdev_region`。
* **注册**：`cdev_init` + `cdev_add` / `cdev_del`；`class_create`（= `class_register`）+ `device_create` 由 udev 生成 `/dev` 节点；`device_destroy` / `class_destroy`。
* 结构：`struct cdev`、`struct file_operations`、`struct inode`、`struct file`（`private_data`）、`struct class`。

### 2. 回调语义与易错点
* `file_operations` 回调：`open` / `release` / `read` / `write` / `llseek` / `mmap` / `ioctl` / `poll`。
* **`copy_to_user` / `copy_from_user` 返回「未拷贝的字节数」**，必须检查；**不可持锁调用**。
* **ioctl 命令宏**：`_IO` / `_IOR` / `_IOW` / `_IOWR`（PDF 提取常误作 `_IORW`）。
* **poll**：`poll_wait(file, &wq, wait)` 注册等待队列，返回 `POLLIN` / `POLLOUT` / `POLLERR`。
* **mmap**：`remap_pfn_range` / `io_remap_pfn_range`（PDF 误作 `io_remap_page_range`）、`struct vm_area_struct`（PDF 误作 `vma_area`）。
* **陷阱**：老的 `register_chrdev` 接口已废弃；**chrdev 没有 devm 变体**，必须手写严格逆序清理。

---

## 四、 设备树与设备/驱动/总线抽象（第5–6章）

### 1. 设备树语法、Overlay 与 OF API
* **文件类型**：`.dts` / `.dtsi`（include）/ `.dtb`（`dtc` 编译）/ `.dtbo`（overlay）。
* **语法要素**：节点与属性、`#address-cells` / `#size-cells` / `reg` / `ranges`、`compatible`、label（`&label`）、`aliases`、`phandle`、`chosen`。
* **运行时视图**：`/proc/device-tree`、`/sys/firmware/devicetree/base`。
* **Overlay**：`/plugin/;`；`fragment@N { target-path | target = <&label>; __overlay__ { ... }; };`；需 `__symbols__` / `__fixups__`，故 `dtc -@` 编译，且要求 **dtc ≥ 1.4.4 且 Linux ≥ 4.14**；内核开 `CONFIG_OF_OVERLAY`；经 configfs（`/sys/kernel/config/device-tree/overlays/`）动态加载。
* **OF API**：`of_find_node_by_name` / `_by_path` / `_by_compatible`、`of_get_child_by_name`、`of_property_read_u32` / `_u64` / `_string` / `_string_array` / `_bool`、`of_property_count_elems_of_size`、`of_get_named_gpio`、`of_parse_phandle`、`of_device_is_compatible`、`of_match_device`、`of_address_to_resource`、`of_iomap`、`of_irq_get`、`of_platform_populate`。
* **匹配**：`struct of_device_id` + `MODULE_DEVICE_TABLE(of, ...)`；`compatible` 匹配时**最具体者优先**。
* **陷阱**：`#address-cells` / `#size-cells` 必须与父节点一致；overlay 依赖符号导出。

### 2. 设备/驱动/总线三元抽象
* **`struct device`**：`parent`、`kobj`、`bus`、`driver`、`platform_data`、`driver_data`、`of_node`、`fwnode`、`devt`、`id`、`release` 回调（**必须提供**，否则 kobject 告警）。
* **`struct bus_type`**：`name`、`dev_name`、`match`、`probe`、`remove`、`shutdown`、`suspend` / `resume`、`dev_groups` / `drv_groups` / `bus_groups`、`pm`。
* **`struct device_driver`**：`name`、`bus`、`probe` / `remove`、`of_match_table`、`id_table`。
* **匹配机制**：设备与驱动注册后由 `bus->match()` 撮合，成功后调用 `driver->probe()`；生命周期由 kobject 引用计数管理（`device_register` / `device_unregister`、`driver_register` / `driver_unregister`）。
* **层次**：总线 → 设备 → 驱动；platform / i2c / spi / usb / pci 都只是 `bus_type` 的实例。
* **陷阱**：`probe` 失败必须回滚已申请资源。

---

## 五、 平台设备与驱动（第7章）

### 1. 结构、id 语义与匹配顺序
* `struct platform_device`：`name`、`id`、`dev`、`num_resources`、`resource`、`id_entry`、`mfd_cell`。
* `struct platform_driver`：`probe(struct platform_device *)`、`remove`、`shutdown`、`suspend` / `resume`、`driver`（含 `of_match_table`）、`id_table`、`bool prevent_deferred_probe`。
* **id 语义**：`-1` = `PLATFORM_DEVID_NONE` → 设备名 `"%s"`；`-2` = `PLATFORM_DEVID_AUTO` → `"%s.%d.auto"`；其它 → `"%s.%d"`。
* **`platform_match()` 匹配顺序**：`driver_override` → `of_driver_match_device`（DT）→ `acpi_driver_match_device` → `id_table` → `name` 字符串比较。
* `struct resource`：`start` / `end` / `flags` / `name` / `parent`；类型 `IORESOURCE_MEM` / `IORESOURCE_IO` / `IORESOURCE_IRQ`（PDF 误作 `ressource`）。

### 2. API 与典型流程
```
module_platform_driver() → platform_driver_register()
        → 总线 match（DT compatible / id_table / name）
        → probe(pdev)
             ├─ platform_get_resource() / platform_get_irq()
             ├─ devm_platform_ioremap_resource()   映射寄存器
             ├─ devm_request_irq()                 注册中断
             └─ 注册到对应子系统（input / IIO / gpio…）
        → remove() 逆序释放
```
* API：`platform_device_register` / `_unregister`、`platform_device_alloc`、`platform_device_add_resources`、`platform_get_resource` / `_byname`、`platform_get_irq`、`devm_platform_ioremap_resource`、`module_platform_driver()`。
* **`platform_driver_probe()`**：把 probe 放 `__init` 段，**禁止 deferred probe**——用于确定不会被延迟探测的设备。
* **陷阱**：需 `CONFIG_OF`；用了 `platform_driver_probe` 就无法再 deferred probe。

---

## 六、 I2C 与 SPI 客户端驱动（第8–9章）

### 1. I2C 驱动
* **结构**：`struct i2c_adapter`（控制器）+ `struct i2c_algorithm`（`master_xfer` 等）；`struct i2c_client`：`flags`、`addr`、`name[I2C_NAME_SIZE=20]`、`adapter`、`dev`、`irq`；`struct i2c_driver`：`class`、`probe(client, id)`、`remove`、`probe_new(client)`、`shutdown`、`driver`、`id_table`。
* `struct i2c_msg`：`addr`、`flags`（`I2C_M_RD 0x0001`、`I2C_M_TEN 0x0010`）、`len`、`buf`——**一个 msg = 一次 START..STOP**。
* **旧 → 新**：`probe_new` 是新的驱动模型接口，**将废弃旧 probe 方法**，以去掉其常被忽略的第二个参数 `id`。
* **API**：`i2c_register_driver` / `module_i2c_driver`；`i2c_transfer`（多 msg 组成 combined transfer）、`i2c_master_send` / `i2c_master_recv`、`i2c_smbus_read_byte_data` / `write_byte_data` / `read_word_data` / `read_i2c_block_data`；`i2c_get_clientdata` / `i2c_set_clientdata`（均有 devm_* 变体）。
* **流程**：控制器 `i2c_add_adapter` → 由 DT / board info 创建 client（`i2c_new_client_device`）→ `i2c_add_driver` → probe。
* **陷阱**：SMBus 命令式传输与 I2C 裸传输语义不同；DT 中设备节点用 `reg = <addr>` 给地址。

### 2. SPI 驱动
* **结构**：`struct spi_controller`（旧名 `spi_master`，`master` 字段仅兼容保留、将废弃）：`num_chipselect`、`min/max_speed_hz`、`setup`、`transfer`、`transfer_one`、`transfer_one_message`、`set_cs`、`set_cs_timing`、`can_dma`、`dma_tx` / `dma_rx`、`kworker`、`pump_messages`、`queue`、`cur_msg`。
* `struct spi_device`：`dev`、`controller`、`max_speed_hz`、`chip_select`、`bits_per_word`、`mode`、`irq`、`cs_gpio`（**LEGACY**）、`cs_gpiod`（新）、`word_delay`、`statistics`。
* `struct spi_transfer`：`tx_buf` / `rx_buf` / `len`、`tx_dma` / `rx_dma`、`tx_sg` / `rx_sg`、`cs_change:1`、`tx_nbits:3` / `rx_nbits:3`（`SPI_NBITS_SINGLE 0x01` / `DUAL 0x02` / `QUAD 0x04`）、`bits_per_word`、`delay_usecs` / `delay`、`speed_hz`、`effective_speed_hz`。
* `struct spi_message`：`transfers`、`spi`、`is_dma_mapped:1`、`complete`、`context`、`frame_length`、`actual_length`、`status`。
* **模式**：`SPI_MODE_0..3`（CPOL × CPHA 四种组合）。
* **API**：`spi_message_init`、`spi_message_add_tail`、`spi_sync` / `spi_async`、`spi_setup`、`spi_write` / `spi_read` / `spi_write_then_read`、`spi_register_driver` / `module_spi_driver`、`devm_spi_register_controller`。
* **流程**：`spi_alloc_master` / `spi_register_controller` → DT 生成 `spi_device` → `spi_register_driver` → probe → 构造 message/transfer → `spi_sync`。
* **陷阱**：`cs_gpio`（旧）→ `cs_gpiod`（新）；每个 transfer 可独立切换 bits/speed/CS；transfer 至少需 `tx_buf` 或 `rx_buf` 之一。

---

## 七、 内核内存分配与 I/O 内存（第10章）

### 1. 地址空间、内存区与分配器层级
* **分页**：`PAGE_SHIFT = 12`（4KB，x86/ARM；ARM64 可选 16KB→14、64KB→16）；32 位 x86 为 **3GB/1GB** 用户/内核分割 @ `0xC0000000`，内核低端 **896MB**（ZONE_NORMAL），之上为 ZONE_HIGHMEM。
* **内存区**：`ZONE_DMA` / `ZONE_DMA32` / `ZONE_NORMAL` / `ZONE_HIGHMEM` / `ZONE_MOVABLE`；`struct zone`、`struct page`。
* **分配器层次**：buddy（页级，按 2^order 合并）→ slab 家族（**SLUB 自 2.6.23 起为默认**）/ SLOB / SLAB（小对象缓存）。

### 2. API 与 I/O 内存映射
* `kmalloc` / `kzalloc` / `kcalloc`（**物理连续，单次上限约 4MB** 且随碎片下降）、`kfree`；`vmalloc` / `vzalloc` / `vfree`（虚拟连续、物理可不连续，仅用于大块）；`kmem_cache_create` / `kmem_cache_alloc` / `kmem_cache_free` / `kmem_cache_destroy`；`alloc_pages` / `__get_free_pages` / `get_zeroed_page` / `free_pages`。
* **GFP 标志**：`GFP_KERNEL`（可睡眠，进程上下文）、`GFP_ATOMIC`（不可睡眠，中断/持锁）、`GFP_DMA` / `GFP_DMA32`、修饰 `__GFP_ZERO` / `__GFP_HIGHMEM` / `__GFP_NOWARN`。
* **I/O 内存**：`request_mem_region` / `release_mem_region`（`/proc/iomem` 可见）、`ioremap` / `iounmap`、`devm_ioremap` / `devm_ioremap_resource`；访问用 `ioread8/16/32`、`iowrite8/16/32`（旧写法 `readl` / `writel`）；指针带 `__iomem` 注解（Sparse 检查）。
* **映射到用户空间**：`remap_pfn_range`。
* **陷阱**：`kmalloc` 上限约 4MB；`vmalloc` 内存不可用于 DMA、不可在原子上下文使用；中断中只能用 `GFP_ATOMIC`；必须检查 `ioremap` 返回值。

---

## 八、 DMA 支持（第11章）

### 1. 两类映射与同步
* **Coherent / Consistent（一致性）**：`dma_alloc_coherent(dev, size, &dma_handle, gfp)` / `dma_free_coherent` / `dma_alloc_noncoherent`；缓存一致性由硬件保证，适合**长期存在**的缓冲（描述符环），代价是 CPU 访问可能变慢。
* **Streaming（流式）**：映射 → 使用 → 解映射，用于一次性传输。
  * `enum dma_data_direction { DMA_BIDIRECTIONAL=0, DMA_TO_DEVICE=1, DMA_FROM_DEVICE=2, DMA_NONE=3 }`。
  * `dma_map_single` / `dma_unmap_single` + `dma_mapping_error`；`dma_map_sg` / `dma_unmap_sg_attrs`；所有权传递 `dma_sync_single_for_cpu` / `_for_device`、`dma_sync_sg_for_cpu` / `_for_device`。
* **scatterlist**：`struct scatterlist`、`sg_set_buf` / `sg_dma_address` / `sg_dma_len`。
* **掩码**：`dma_set_mask_and_coherent` / `dma_set_mask` / `dma_set_coherent_mask` + `DMA_BIT_MASK(n)`。
* **完成量**：`struct completion`、`init_completion`、`wait_for_completion[_timeout]`、`complete` / `complete_all`——DMA 完成回调的标准通知机制。

### 2. dmaengine（DMA 控制器驱动 API）
```
1. 申请通道  dma_request_channel() / dma_request_chan()（DT: dmas = <&dmac ch ...>）
2. 配置通道  dmaengine_slave_config(chan, &struct dma_slave_config{...})
3. 取描述符  dmaengine_prep_slave_single() / dmaengine_prep_dma_memcpy()
             / dmaengine_prep_dma_cyclic()          -> struct dma_async_tx_descriptor
4. 提交事务  dmaengine_submit(tx)                   -> dma_cookie_t
5. 发起传输  dma_async_issue_pending(chan)
      设备搬运完成 -> DMA 中断 -> tx->callback -> complete()
   查询状态  dmaengine_tx_status()
```
* **cyclic（环形）**用于串口等持续传输（书中 atmel_serial 示例，配合 `dmaengine_tx_status` 与 `dma_sync_sg_for_cpu/_for_device`）。
* **陷阱**：映射后 CPU **不可直接访问**（必须先 sync）；方向必须正确；忘记 unmap 会泄漏 DMA 映射。

---

## 九、 Regmap 寄存器映射抽象（第12章）

### 1. 配置结构与初始化
* **目的**：统一 I2C / SPI / MMIO 的寄存器读写，隐藏总线差异、缓存与加锁。
* `struct regmap_config` 关键字段：`name`、**`reg_bits`（必填）**、`reg_stride`、`pad_bits`、**`val_bits`（必填）**、回调 `writeable_reg` / `readable_reg` / `volatile_reg` / `precious_reg`、`disable_locking`、`lock` / `unlock` / `lock_arg`、`reg_read` / `reg_write`、`fast_io`、`max_register`、访问表 `wr_table` / `rd_table` / `volatile_table` / `precious_table`（`struct regmap_access_table { yes_ranges, no_ranges }`）、`reg_defaults` / `num_reg_defaults`、`enum regcache_type cache_type`、`reg_defaults_raw`、`read_flag_mask` / `write_flag_mask`、`use_single_rw`、`can_multi_write`、`reg_format_endian` / `val_format_endian`、`ranges` / `num_ranges`。
* **初始化**：`devm_regmap_init_i2c` / `_spi` / `_mmio` / `_init`（非 devm 版配 `regmap_exit`）。

### 2. 访问、缓存与陷阱
* **访问**：`regmap_read` / `regmap_write`、**`regmap_update_bits(map, reg, mask, val)`（读改写，最常用）**、`regmap_bulk_read` / `regmap_bulk_write`、`regmap_multi_reg_write`（`struct reg_sequence { reg, def, delay_us }`）。
* **缓存**：`cache_type` = `REGCACHE_NONE` / `RBTREE` / `FLAT` / `COMPRESSED`（5.10 另有 MAPLE）；挂起恢复用 `regcache_mark_dirty` + `regcache_sync`。
* **陷阱**：`reg_bits` / `val_bits` 必填；`volatile_reg` 用于寄存器值会自变的设备；`disable_locking` 需自管锁；`use_single_rw` 用于不支持 bulk 的总线（书中名即 `use_single_rw`，**非** `use_single_read/write`）。

---

## 十、 内核 IRQ 框架（第13章）

### 1. 四大结构与 IRQ domain
* `struct irq_chip`：`name`、`irq_enable` / `irq_disable`、`irq_ack`、`irq_mask` / `irq_unmask`、`irq_eoi`、`irq_set_affinity`、`irq_set_type`、`irq_set_wake`、`irq_bus_lock` / `irq_bus_sync_unlock`、`ipi_send_single` / `ipi_send_mask`。**注意：`handle_edge_irq` 的控制器必须定义 `irq_ack`（即使函数体为空）**。
* `struct irq_desc`：`irq_data`、`kstat_irqs`（`__percpu`）、`handle_irq`、`action`、`irqs_unhandled`、`lock`（`raw_spinlock_t`）、`percpu_enabled`、`threads_active`、`wait_for_threads`、`nr_actions`、`parent_irq`。
* `struct irqaction`：`handler`、`dev_id`、`next`、`thread_fn`、`thread`、`irq`、`flags`、`thread_flags`、`name`、`dir`。
* `struct irq_data`：`irq`（Linux 虚拟号）、`hwirq`（硬件号）、`chip`、`domain`、`chip_data`。
* **号映射**：`irq_domain_add_linear()` / `_tree()` / `_nomap()`、`irq_domain_add_simple`、`irq_create_mapping(domain, hwirq)`、`irq_find_mapping`；层级域需 `CONFIG_IRQ_DOMAIN_HIERARCHY`。

### 2. 处理流、注册与 ARM GIC 号段
* **处理流设置**：`irq_set_chip_and_handler(irq, chip, handle_edge_irq|handle_level_irq|handle_simple_irq)`；chained 级联用 `generic_handle_irq()`；chained vs nested 两种级联模型。
* **注册**：`request_irq` / `request_threaded_irq` / `request_any_context_irq` / `free_irq`；devm 变体 `devm_request_irq` / `devm_request_threaded_irq`。
* **flags**：`IRQF_SHARED 0x00000080`、`IRQF_ONESHOT 0x00002000`、`IRQF_NO_SUSPEND 0x00004000`、`IRQF_NO_THREAD 0x00010000`、`IRQF_TIMER = IRQF_NO_SUSPEND|IRQF_NO_THREAD`；触发类型 `IRQF_TRIGGER_RISING` / `FALLING` / `HIGH` / `LOW`。
* **返回值语义**：`IRQ_NONE`（非本设备）、`IRQ_HANDLED`（已处理）、`IRQ_WAKE_THREAD`（唤醒 `thread_fn`）。
* **ARM GIC 号段**：SGI 0–15、PPI 16–31、SPI ≥32；IPI 共 8 个：`IPI_WAKEUP` / `IPI_TIMER` / `IPI_RESCHEDULE` / `IPI_CALL_FUNC` / `IPI_CPU_STOP` / `IPI_IRQ_WORK` / `IPI_COMPLETION` / `NR_IPI`。
* **陷阱**：共享中断（`IRQF_SHARED`）需唯一 `dev_id` 且必须能判断是否本设备（否则返回 `IRQ_NONE`）；会睡眠或耗时的处理必须用 threaded IRQ（配 `IRQF_ONESHOT`）。

---

## 十一、 Linux 设备模型与 sysfs（第14章）

### 1. kobject 家族与属性宏
* `struct kobject`：`name`、`entry`、`parent`、`kset`、`ktype`、`sd`、`kref`。
* `struct kobj_type`：**`release`（必须）**、`sysfs_ops`、`default_attrs`；`struct sysfs_ops`：`show` / `store`（**show 的缓冲区仅 `PAGE_SIZE`**，须用 `scnprintf`）；`struct kset`：`list`、`list_lock`、`kobj`。
* `struct attribute`：`name`、`owner`、`mode`；`__ATTR(_name,_mode,_show,_store)` 宏；`DEVICE_ATTR` / `DEVICE_ATTR_RW` / `_RO` / `_WO`（生成 `dev_attr_<name>`）、`BUS_ATTR_*` / `DRIVER_ATTR_*` / `CLASS_ATTR_*`；大二进制属性用 `struct bin_attribute`；`struct attribute_group`（含 `is_visible` 回调，可条件暴露属性）。
* **API**：`kobject_init` / `kobject_create` / `kobject_add` / `kobject_init_and_add` / `kobject_get` / `kobject_put` / `kobject_uevent`；`kset_create_and_add` / `kset_unregister`；`sysfs_create_file` / `sysfs_remove_file` / `sysfs_create_bin_file` / `sysfs_create_group` / `sysfs_notify`。

### 2. 生命周期与 devres
* **流程**：`device_register` → `kobject_add` → sysfs 建目录与属性 → 用户读写触发 `show` / `store` → `device_unregister` → `kobject_put` → `ktype->release`。
* **devres（托管资源）**：`devm_kzalloc` / `devm_ioremap` / `devm_request_irq` / `devm_gpiochip_add_data`…绑定到 `struct device`，驱动卸载时**按申请逆序自动释放**——这是第 2 版通篇推荐的写法。
* **陷阱**：kobject 必须提供 `release`；show 缓冲区仅 `PAGE_SIZE`；属性 mode 用 `S_IRUGO` / `S_IWUSR`；`show`/`store` 中不做重活。

---

## 十二、 IIO 框架（第15章）

### 1. 结构与模式
* `struct iio_dev`：`modes`、`currentmode`、`dev`、`buffer`、`scan_bytes`、`available_scan_masks`、`active_scan_mask`、`scan_timestamp`、`trig`、`pollfunc`、`channels`、`num_channels`、`name`、`info`、`setup_ops`、`chrdev`。
* **模式**：`INDIO_DIRECT_MODE`（sysfs 单次读）、`INDIO_BUFFER_TRIGGERED`、`INDIO_BUFFER_SOFTWARE`、`INDIO_BUFFER_HARDWARE`、`INDIO_EVENT_TRIGGERED`、`INDIO_HARDWARE_TRIGGERED`、`INDIO_ALL_BUFFER_MODES` / `INDIO_ALL_TRIGGERED_MODES`。
* `struct iio_info`：`read_raw`、`write_raw`、`driver_module`。
* `struct iio_chan_spec`：`type`、`channel`、`channel2`、`address`、`scan_index`、`scan_type{sign, realbits, storagebits, shift, repeat, endianness}`、`info_mask_separate` / `_shared_by_type` / `_shared_by_dir` / `_shared_by_all`、`event_spec`、`ext_info`、`extend_name`、`datasheet_name`、`modified` / `indexed` / `output` / `differential`。
* `enum iio_chan_info_enum`：`IIO_CHAN_INFO_RAW=0`、`PROCESSED`、`SCALE`、`OFFSET`、`CALIBSCALE`、…、`SAMP_FREQ`、`FREQUENCY`、`PHASE`、`HARDWAREGAIN`、`HYSTERESIS`；sysfs 属性名格式 `{direction}_{type}{index}_{modifier}_{info_mask}`。

### 2. 触发缓冲与消费者
* **API**：`devm_iio_device_alloc` / `devm_iio_device_register` / `iio_device_unregister` / `iio_priv`；`iio_triggered_buffer_setup(dev, pollfunc, handler, setup_ops)` + `iio_triggered_buffer_cleanup()`；`iio_push_to_buffers_with_timestamp`、`iio_trigger_notify_done`、`iio_get_time_ns`、`iio_buffer_enabled`。
* **触发缓冲流程**：pollfunc 返回 `IRQ_WAKE_THREAD` → 下半部用 `for_each_set_bit(bit, active_scan_mask, masklength)` 逐通道读取 → `iio_push_to_buffers_with_timestamp()` → `iio_trigger_notify_done(trig)`；`setup_ops` = { `preenable`, `postenable`, `postdisable`, `predisable` }。
* **消费者（内核内）**：`devm_iio_channel_get` / `devm_iio_channel_get_all`；`iio_read_channel_processed` / `iio_read_channel_raw`；必须处理 `EPROBE_DEFER`。
* **用户空间**：sysfs（`in_voltage3_raw` × `in_voltage_scale` 得物理量）+ 字符设备 `/dev/iio:deviceX`；**libiio** 库与 CLI（`iio_info` / `iio_readdev` 等）。
* **陷阱**：触发缓冲需 `INDIO_BUFFER_TRIGGERED` 模式；`available_scan_masks` 决定可同时采样的通道组合。

---

## 十三、 Pin Controller 与 GPIO 子系统（第16章）

### 1. pinctrl 与两代 GPIO 接口
* **pinctrl**：`struct pinctrl_desc`：`name`、`pins`、`npins`、`pctlops`、`pmxops`、`confops`、`owner`；`struct pinctrl_pin_desc { number, name }`；控制器注册 `devm_pinctrl_register(pdev, desc, driver_data)`；消费者 `devm_pinctrl_get(dev)` → `pinctrl_lookup_state(p, "default")` → `pinctrl_select_state(p, state)`，或一步 `devm_pinctrl_get_select(dev, "default")`；DT 用 `pinctrl-0` / `pinctrl-names`。
* **描述符 GPIO（新，推荐）**：头文件 `<linux/gpio/consumer.h>`；`struct gpio_desc { chip, flags, label }`；`gpiod_get` / `gpiod_get_index` / `gpiod_get_optional` + devm_ 变体；`enum gpiod_flags { GPIOD_ASIS=0, GPIOD_IN, GPIOD_OUT_LOW, GPIOD_OUT_HIGH }`；`gpiod_get_value` / `gpiod_set_value[_cansleep]`、`gpiod_direction_input` / `_output`、`gpiod_put`、`gpiod_to_irq`；属性名后缀按 `gpio_suffixes[] = {"gpios","gpio"}` 依次尝试（`of_find_gpio()`）。
* **整型 GPIO（旧/遗留）**：头文件 `<linux/gpio.h>`；`gpio_is_valid`、`gpio_request` / `gpio_free`、`gpio_direction_input` / `_output`、`gpio_get_value` / `gpio_set_value`（+ `_cansleep`）、`gpio_set_debounce`、`gpio_cansleep`。
* **GPIO 控制器**：`struct gpio_chip`：`label`、`gpiodev`、`parent`、`owner`、`request` / `free` / `get_direction` / `direction_input` / `direction_output` / `get` / `get_multiple` / `set` / `set_multiple` / `set_config` / `to_irq`、`base`、`ngpio`、`names`、`can_sleep`、`bgpio_*`、`struct gpio_irq_chip irq`；注册 `gpiochip_add_data` / `devm_gpiochip_add_data`，移除 `gpiochip_remove`。
* **`struct gpio_irq_chip`（v5.x 新抽象）**：`chip`、`domain`、`domain_ops`、`handler`、`default_type`、`parent_handler`、`parent_handler_data` / `_array`、`num_parents`、`parents`、`map`、`threaded`、`per_parent_data`、`init_hw`、`init_valid_mask`、`valid_mask`、`first`、`irq_enable` / `disable` / `unmask` / `mask`；需 `CONFIG_IRQ_DOMAIN_HIERARCHY`。

### 2. 用户空间与 GPIO aggregator
* **sysfs GPIO 已废弃**——书中明言 *"The Kernel Linux GPIO user space sysfs is deprecated and has been discontinued"*。
* **新方案：GPIO 字符设备**（kernel v4.8 合入），每个控制器一个 `/dev/gpiochipN`；**libgpiod**（C/C++/Python）：`gpiod_chip_open_by_name` / `_lookup`、`gpiod_chip_get_line`、`struct gpiod_chip` / `gpiod_line`、`gpiod_line_release_bulk`、`gpiod_chip_close`、`GPIOD_LINE_EVENT_RISING_EDGE`；CLI 工具 `gpiodetect` / `gpioinfo` / `gpioget` / `gpioset` / `gpiofind` / `gpimon`。
* **GPIO aggregator（kernel 5.8 新增）**：`CONFIG_GPIO_AGGREGATOR=y`；sysfs 接口 `/sys/bus/platform/drivers/gpio-aggregator/` 的 `new_device` / `delete_device`，格式 `[<gpioA>] [<gpiochipB> <offsets>]`；DT 用 `compatible = "gpio-aggregator"` + `gpios`。
* **陷阱**：旧整型 GPIO 无法描述非全局编号且需自行管理，应改用描述符方式；`bgpio_*` 可简化通用 MMIO GPIO 控制器的实现。

---

## 十四、 输入子系统（第17章）

### 1. 结构与能力设置
* `struct input_dev`：`name`、`phys`、能力位图 `evbit` / `keybit` / `relbit` / `absbit` / `mscbit`、`repeat_key`、`rep[REP_CNT]`、`absinfo`、`key`、`open` / `close`、`users`、`dev`、`num_vals` / `max_vals` / `vals`、`devres_managed`。
* **分配/注册**：`input_allocate_device` / `devm_input_allocate_device`、`input_free_device`、`input_register_device`、`input_unregister_device`。
* **能力设置**：`set_bit(EV_KEY/EV_REL/EV_ABS, dev->evbit)`；`input_set_abs_params(dev, code, min, max, fuzz, flat)`；`input_set_capability`。

### 2. 上报、轮询与用户空间
* **上报**：`input_report_abs` / `input_report_key` / `input_report_rel(dev, code, value)` + **`input_sync(dev)` 结束一次事件包**；`EV_KEY` 的 value：0=释放、1=按下、2=自动重复。
* **轮询设备**（无中断的 GPIO 按键等）：`struct input_polled_dev`：`private`、`open` / `close` / `poll`、`poll_interval`（**默认 500ms**）、`poll_interval_max`、`poll_interval_min`（默认 0）、`input`；`devm_input_allocate_polled_device` / `input_allocate_polled_device` / `input_free_polled_device`。
* **用户空间**：`struct input_event { struct timeval time; __u16 type; __u16 code; __s32 value; }`；节点 `/dev/input/event<X>`；工具 `evtest`；用 `select()` / `poll()` + `read()` 循环读取。
* **流程**：`input_allocate_device` → 设 evbit / abs 参数 → `input_register_device` → 中断或轮询中 `input_report_*` + `input_sync` → 用户空间 read → `input_unregister_device` + `input_free_device`。
* **陷阱**：每次上报必须 `input_sync`，否则用户空间收不到完整报告。

---

## 十五、 第2版相对第1版的关键变化

| 主题 | 第1版（v4.13） | 第2版（v5.10） |
| :--- | :--- | :--- |
| 设备树 | 基础语法 + OF API | 新增 **Overlay**（fragment / `__overlay__` / configfs 加载，`dtc -@`） |
| GPIO 用户空间 | sysfs（`/sys/class/gpio`） | **sysfs 废弃**，改用 GPIO 字符设备（v4.8）+ **libgpiod**；新增 **GPIO aggregator**（5.8） |
| 中断下半部 | softirq / tasklet / workqueue | **tasklet 明确废弃**，推荐 threaded IRQ；tasklet 结构新增 `use_callback` |
| 工作队列 | `create_workqueue(name)` | **cmwq**：`alloc_workqueue(name, flags, max_active)` |
| I2C | `probe(client, id)` | **`probe_new(client)`**（去掉第二个参数） |
| IRQ | 分散于各章 | **独立成章**（`struct irq_desc` / `irq_data` / `irqaction` / domain） |
| GPIO 中断抽象 | 手工拼装 | **`struct gpio_irq_chip`**（层级中断域与 GPIO 一体化） |
| 用户空间工具 | 无 | **libiio**、`libgpiod` 及其 CLI 工具集 |
| 架构支持 | ARM 32 位为主 | 增加 **aarch64** 交叉工具链 |
| 章节组织 | 22 章 | **4 大板块 17 章**（第16章 pinctrl/GPIO 大幅扩充，第17章 input 相对精简） |

---

## 十六、 核心设计思想提炼

1. **设备与驱动分离，总线负责撮合**：设备只描述「有什么资源」（DT 节点 / `platform_device` + `struct resource`），驱动只描述「怎么用」，`bus_type->match()` 是二者唯一的相遇点；`probe()` 承担全部资源申请，`remove()` 严格逆序释放。
2. **硬件描述数据化**：设备树把板级硬编码（board file、`i2c_board_info`）转为可由 bootloader 传递的 `.dtb`；`compatible` 是匹配契约，**Overlay 让同一份内核镜像支持不同板级配置而无需重编译**。
3. **上下文决定 API 选择**：硬中断 / softirq 上下文不可睡眠，只能用 `GFP_ATOMIC`、非阻塞路径与 `spi_async()`；慢总线（I2C/SPI）操作必须退到 threaded IRQ 或 workqueue——这正是 **tasklet 被废弃、threaded IRQ 成为正统** 的根本原因。
4. **统一抽象屏蔽总线差异**：regmap 抹平 I2C/SPI/MMIO 的寄存器访问，gpiolib 抹平片上与扩展 GPIO，IRQ domain 抹平硬件号与 Linux 虚拟号，IIO / input / pinctrl 为同类外设提供统一 ABI——驱动只填 `ops` 结构体，框架负责其余一切。
5. **托管资源优先（devres）**：`devm_*` 把资源生命周期绑定到 `struct device`，使 `remove()` 与错误回滚路径不再需要手写释放代码，是第 2 版通篇推荐的主流写法。
6. **现代 API 的演进方向**：`module_*_driver()` 宏取代手写 init/exit、描述符 GPIO（`gpiod_*`）取代整型 GPIO、GPIO 字符设备 + libgpiod 取代 sysfs、`ioread*/iowrite*` 取代 `readl/writel`、`probe_new` 取代 `probe`、`alloc_workqueue` 取代 `create_workqueue`、`dmaengine` 取代厂商私有 DMA 接口。
7. **典型陷阱清单**：`struct device` 未实现 `release`；`show` 缓冲区超 `PAGE_SIZE`；共享中断未返回 `IRQ_NONE`；`handle_edge_irq` 控制器漏定义 `irq_ack`；映射后 CPU 直接访问 DMA 缓冲（漏 `dma_sync_*`）；`reg_bits`/`val_bits` 未填；忘记 `input_sync()`；用 `platform_driver_probe()` 后仍期待 deferred probe；`vmalloc` 内存交给 DMA。
