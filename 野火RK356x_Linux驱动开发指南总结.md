# 野火 RK356x Linux 驱动开发指南 · 核心重点总结与技术架构精要

> **文档出处**：[野火 RK356x Linux 驱动开发指南](https://doc.embedfire.com/linux/rk356x/driver/zh/latest/index.html)  
> **适用平台**：瑞芯微 Rockchip RK356x 系列（RK3568 / RK3566，四核 ARM Cortex-A55）  
> **参考硬件**：野火 LubanCat-RK 系列板卡（鲁班猫 1N、LubanCat-2 等）  
> **内核基准**：Linux 4.19.232 (LTS 稳定版) / Linux 6.1.99 (主线演进版)  
> **架构体系**：ARM64 (aarch64)

---

## 目录
- [一、 硬件与开发基准环境](#一-硬件与开发基准环境)
- [二、 Linux 驱动基础架构与设备模型](#二-linux-驱动基础架构与设备模型)
  - [1. 内核模块与交叉编译体系](#1-内核模块与交叉编译体系)
  - [2. 字符设备驱动演进（从传统到自动化）](#2-字符设备驱动演进从传统到自动化)
  - [3. Linux 设备驱动模型（Platform 平台总线）](#3-linux-设备驱动模型platform-平台总线)
  - [4. Linux 设备树（Device Tree）与 of_ API](#4-linux-设备树device-tree与-of_-api)
  - [5. 动态设备树插件（Device Tree Overlay, DTBO）](#5-动态设备树插件device-tree-overlay-dtbo)
  - [6. 内核并发与竞争防御体系](#6-内核并发与竞争防御体系)
  - [7. 内核定时器与延时机制](#7-内核定时器与延时机制)
- [三、 核心外设子系统与驱动框架](#三-核心外设子系统与驱动框架)
  - [1. Pinctrl 子系统与 GPIO 子系统现代化规范](#1-pinctrl-子系统与-gpio-子系统现代化规范)
  - [2. 中断子系统与底半部机制选择](#2-中断子系统与底半部机制选择)
  - [3. 阻塞、非阻塞 I/O 与异步通知（SIGIO）](#3-阻塞非阻塞-io-与异步通知sigio)
  - [4. Regmap 统一寄存器抽象层](#4-regmap-统一寄存器抽象层)
  - [5. I2C 与 SPI 总线驱动框架](#5-i2c-与-spi-总线驱动框架)
  - [6. 输入子系统与多点触摸（Multi-Touch Type B）驱动](#6-输入子系统与多点触摸multi-touch-type-b驱动)
  - [7. 其他基础外设框架（PWM、IIO/ADC、RTC、电源管理）](#7-其他基础外设框架pwmiioadcrtc电源管理)
- [四、 驱动进阶知识与高级系统架构](#四-驱动进阶知识与高级系统架构)
  - [1. DMA 机制、Cache 一致性与 IOMMU](#1-dma-机制cache-一致性与-iommu)
  - [2. DRM/KMS 现代显示框架（对比传统 Framebuffer）](#2-drmkms-现代显示框架对比传统-framebuffer)
  - [3. 网络子系统、NAPI 机制与 PHY/MDIO 链路](#3-网络子系统napi-机制与-phymdio-链路)
  - [4. SMP 对称多核架构与从核唤醒流程](#4-smp-对称多核架构与从核唤醒流程)
- [五、 内核驱动开发避坑指南与最佳实践](#五-内核驱动开发避坑指南与最佳实践)

---

## 一、 硬件与开发基准环境

### 1. 平台架构参数
* **处理器**：Rockchip RK3568 / RK3566，集成 4 核 64 位 ARM Cortex-A55 核心。
* **硬件单元**：集成 ARM GICv3 中断控制器、VOP2 显示处理器、USB 3.0/PCIE 3.0 控制器、双千兆以太网 GMAC 等。
* **板卡载体**：野火 LubanCat 系列，外设丰富（RGB LED、按键、I2C 传感器、MIPI-DSI/EDP/HDMI 屏幕、以太网接口等）。

### 2. 内核版本与分支
* **Linux 4.19.232**：对应官方 `stable-4.19-rk356x` 分支，配置文件为 `lubancat2_defconfig`。
* **Linux 6.1.99**：对应演进分支 `lbc-develop-6.1`，配置文件为 `lubancat_linux_rk356x_defconfig`。
* **确认方式**：在开发板终端执行 `uname -a` 查看运行的内核版本。

### 3. 交叉编译环境与 Makefile 规范
* **编译工具链**：`aarch64-linux-gnu-`。
* **驱动编译 Makefile 核心范式**：
  ```makefile
  KERNEL_DIR := /home/user/lubancat_kernel/
  ARCH := arm64
  CROSS_COMPILE := aarch64-linux-gnu-
  export ARCH CROSS_COMPILE

  obj-m := my_driver.o

  all:
  	$(MAKE) -C $(KERNEL_DIR) M=$(CURDIR) modules

  clean:
  	$(MAKE) -C $(KERNEL_DIR) M=$(CURDIR) clean
  ```
* **核心理念**：外部驱动模块无需在内核树内构建，借由内核目录顶层 Makefile 的 `M=$(CURDIR)` 机制，将当前目录构建为 `.ko` 目标，利用内核已生成的符号表进行链接。

---

## 二、 Linux 驱动基础架构与设备模型

### 1. 内核模块与交叉编译体系
* **生命周期宏**：
  * 入口声明：`module_init(xxx_init);`
  * 出口声明：`module_exit(xxx_exit);`
  * 证书声明：`MODULE_LICENSE("GPL");`（若不声明 GPL，无法使用部分内核专有符号）。
* **符号导出**：`EXPORT_SYMBOL(sym)` 与 `EXPORT_SYMBOL_GPL(sym)`，用于跨模块调用函数和变量。
* **传参机制**：`module_param(name, type, perm)`，允许在 `insmod xxx.ko key=value` 时向模块传递参数。

### 2. 字符设备驱动演进（从传统到自动化）
* **设备号机制**：
  * 主设备号（Major）：标识驱动类型；次设备号（Minor）：标识同一驱动下的具体硬件个体。
  * 动态申请：`alloc_chrdev_region(&dev_id, 0, count, "device_name");`
  * 释放设备号：`unregister_chrdev_region(dev_id, count);`
* **字符设备核心结构体**：
  * `struct cdev`：`cdev_init(&cdev, &fops)` 绑定 `file_operations`，再通过 `cdev_add(&cdev, dev_id, count)` 挂入系统。
* **设备节点自动创建（udev/mdev 机制）**：
  * 避免手动运行 `mknod /dev/xxx c major minor`。
  * 驱动通过 `class_create(THIS_MODULE, "my_class")` 创建类目录；
  * 调用 `device_create(cls, NULL, dev_id, NULL, "my_dev")` 在 `/dev` 自动生成设备节点。

### 3. Linux 设备驱动模型（Platform 平台总线）
* **软硬件解耦理念**：
  * **Device（设备）**：提供硬件信息（寄存器物理基址、中断号、引脚等）。
  * **Driver（驱动）**：提供软件操作逻辑与控制流程。
  * **Bus（总线）**：负责将 Driver 与 Device 匹配。对于不可枚举的总线（如挂载在 SoC 内部总线的外设），抽象出虚拟的 **Platform 总线**。
* **Platform 驱动核心结构体**：
  ```c
  static const struct of_device_id my_of_match[] = {
      { .compatible = "embedfire,rk356x-led" },
      { /* Sentinel */ }
  };
  MODULE_DEVICE_TABLE(of, my_of_match);

  static struct platform_driver my_platform_driver = {
      .probe = my_driver_probe,
      .remove = my_driver_remove,
      .driver = {
          .name = "rk356x_led",
          .of_match_table = my_of_match,
      },
  };
  module_platform_driver(my_platform_driver);
  ```
* **匹配机制**：内核启动时，平台总线会遍历设备树生成的平台设备，并与驱动中的 `of_match_table` 比对，一旦 `compatible` 字符串吻合，即自动调用驱动的 `probe()`。

### 4. Linux 设备树（Device Tree）与 of_ API
* **语法关键规范**：
  * 命名形式：`node-name@unit-address`（地址必须与 `reg` 的首基址严格对应）。
  * 寻址规则：`#address-cells`（基址占用的 32 位单元格数）与 `#size-cells`（长度占用的单元格数）决定子节点 `reg` 数组的组织格式。
  * 状态指示：`status = "okay";` 或 `"disabled";`。
  * 中断绑定：`interrupt-parent = <&gic>;` 声明中断控制器，`interrupts = <GIC_SPI 252 IRQ_TYPE_LEVEL_LOW>;` 声明具体向量号及触发模式。
* **驱动提取属性 API（of_ 函数族）**：
  * 节点查找：`of_find_node_by_path()`、`of_find_compatible_node()`。
  * 属性读取：`of_property_read_u32()`、`of_property_read_u32_array()`、`of_property_read_string()`、`of_property_read_bool()`。
  * 地址映射：`of_iomap(node, index)`，内部自动解析 `reg` 属性并执行 `ioremap` 映射为虚拟地址。

### 5. 动态设备树插件（Device Tree Overlay, DTBO）
* **核心优势**：无需重新编译和刷写整个主设备树（DTS），即可对特定外设节点进行独立打补丁、扩展或修改。
* **语法范式**：
  ```dts
  /dts-v1/;
  /plugin/;

  / {
      fragment@0 {
          target = <&gpio1>;
          __overlay__ {
              my_led {
                  compatible = "embedfire,rk356x-led";
                  gpios = <&gpio1 RK_PC7 GPIO_ACTIVE_LOW>;
                  status = "okay";
              };
          };
      };
  };
  ```
* **编译与加载全流程**：
  1. 将 `.dts` 源码使用 DTC 编译为二进制 `.dtbo` 插件文件；
  2. 复制到开发板 `/boot/dtb/overlay/` 目录；
  3. 修改 `/boot/uEnv/uEnv.txt`，在 `#overlay_start` 和 `#overlay_end` 之间添加该插件项；
  4. 系统重启后，U-Boot 自动在内存中将主 DTB 与各 DTBO 进行无缝合并，生成完整拓扑传递给内核。

### 6. 内核并发与竞争防御体系
| 锁/同步机制 | 是否允许休眠 | 适用上下文 | 特点与使用场景 |
| :--- | :--- | :--- | :--- |
| **原子操作 (`atomic_t`)** | 否 | 任意上下文 | 依赖处理器原子指令，用于引用计数或简单标志位翻转 |
| **自旋锁 (`spinlock_t`)** | **严禁休眠** | 进程 / 中断上下文 | 忙等待；在中断上下文中使用 `spin_lock_irqsave` / `spin_unlock_irqrestore` 保护共享资源并关中断 |
| **互斥锁 (`mutex`)** | **可以休眠** | 仅进程上下文 | 独占共享资源，访问时间较长的互斥区首选，无法获取时自动调度让出 CPU |
| **信号量 (`semaphore`)** | **可以休眠** | 仅进程上下文 | 具备资源计数值，允许多个任务并发访问限定数量的资源 |

### 7. 内核定时器与延时机制
* **核心结构体**：`struct timer_list`。
* **初始化与激活**：
  * 使用 `timer_setup(&my_timer, timer_callback, 0)` 绑定超时回调。
  * 调用 `mod_timer(&my_timer, jiffies + msecs_to_jiffies(100))` 设定触发时延并启动。
  * 销毁定时器：`del_timer_sync(&my_timer)`。
* **关键约束**：**定时器超时函数运行于软中断（原子上下文）中**，绝不可调用任何休眠或阻塞函数（如 `msleep`、`mutex_lock`、`copy_to_user` 等）。

---

## 三、 核心外设子系统与驱动框架

### 1. Pinctrl 子系统与 GPIO 子系统现代化规范
* **Pinctrl（引脚控制）**：
  * 负责引脚复用（Pinmux）与电气特性（Pinconf，上下拉、驱动强度）配置。
  * 瑞芯微平台格式：`rockchip,pins = <Bank, Pin, Func, Config>`（如 `<RK_GPIO0 RK_PC7 RK_FUNC_GPIO &pcfg_pull_none>`）。
* **GPIO 子系统新版描述符 API（推荐）**：
  * 全面替代旧版的全局整数编号（`gpio_request` / `gpio_set_value`）；
  * 设备树绑定：`led-gpios = <&gpio1 RK_PC7 GPIO_ACTIVE_LOW>;`
  * 驱动申请获取：
    ```c
    // 自动匹配 "-gpios" 后缀，生命周期交由设备托管（devres）
    struct gpio_desc *led = devm_gpiod_get(&pdev->dev, "led", GPIOD_OUT_HIGH);
    ```
  * 控制输出：`gpiod_set_value(led, 1);`（框架根据设备树的 `GPIO_ACTIVE_LOW` 自动做物理电平反相转换，业务代码保持逻辑一致）。

### 2. 中断子系统与底半部机制选择
* **中断核心 API**：
  * 获取中断号：`platform_get_irq(pdev, 0);`
  * 申请中断：`devm_request_irq(dev, irq, handler, IRQF_TRIGGER_FALLING, "key_irq", dev_id);`
* **底半部机制决策矩阵**：
  1. **软中断（softirq）**：运行在软中断上下文，性能最高，支持多核并发，不可休眠；专属于网络协议栈及内核核心驱动。
  2. **Tasklet**：基于软中断派生，同一 Tasklet 实例保证单核串行，无需考虑并发重入，不可休眠；适合耗时微秒级的简单逻辑。
  3. **工作队列（workqueue）**：运行在内核线程（`kworker`）的**进程上下文**中，**是底半部中唯一支持休眠与阻塞（`msleep`、I/O 等待）的机制**。
  4. **线程化中断（`request_threaded_irq`）**：顶半部快速应答确认并屏蔽硬件，底半部派发给内核专属线程；**慢速总线（I2C/SPI）外设驱动的标准范式**，避免了在硬中断中执行总线通信等待。

### 3. 阻塞、非阻塞 I/O 与异步通知（SIGIO）
* **阻塞 I/O**：
  * 驱动维护 `wait_queue_head_t wq`；
  * 读取无数据时进入睡眠：`wait_event_interruptible(wq, condition);`
  * 写入或中断触发数据就绪后唤醒：`wake_up_interruptible(&wq);`
* **非阻塞与多路复用轮询（Poll）**：
  * 驱动实现 `fops->poll`：
    ```c
    static __poll_t my_poll(struct file *filp, poll_table *wait) {
        __poll_t mask = 0;
        poll_wait(filp, &my_wait_queue, wait);
        if (data_ready) mask |= (EPOLLIN | EPOLLRDNORM);
        return mask;
    }
    ```
* **异步通知（Async Notification）**：
  * 驱动维护 `struct fasync_struct *async_queue;`
  * 实现 `fops->fasync`（内部调用 `fasync_helper`）；
  * 数据就绪时调用 `kill_fasync(&async_queue, SIGIO, POLL_IN)` 向用户态目标进程发送信号。

### 4. Regmap 统一寄存器抽象层
* **设计意图**：针对 I2C、SPI、MMIO 等不同总线外设，消除反复手写通信协议读写控制代码的重复劳动。
* **核心配置（`struct regmap_config`）**：
  * 定义寄存器地址位宽（`reg_bits`）与数据位宽（`val_bits`）。
  * 设定缓存类型：`REGCACHE_RBTREE`（红黑树缓存）或 `REGCACHE_FLAT`（扁平数组缓存）。
* **初始化与统一操作 API**：
  * 初始化：`devm_regmap_init_i2c(client, &config)` 或 `devm_regmap_init_spi(spi, &config)`。
  * 单值读写：`regmap_read(map, reg, &val)` / `regmap_write(map, reg, val)`。
  * 批量传输：`regmap_bulk_read()` / `regmap_bulk_write()`。
  * 原子掩码更新：`regmap_update_bits(map, reg, mask, val)`（完成读-改-写原子操作，不干扰其他比特位）。

### 5. I2C 与 SPI 总线驱动框架
* **三层架构分工**：
  * **控制器/适配器驱动（Adapter/Master）**：由 SoC 原厂实现（如 RK356x 原厂驱动生成 `/dev/i2c-X` 控制器）。
  * **总线核心层（Core）**：内核提供的通用通信协议抽象，实现消息打包与派发调度。
  * **设备驱动层（Device Driver）**：由外设开发者编写，实现 `i2c_driver` 或 `spi_driver`，在 `probe()` 中与 `i2c_client` / `spi_device` 绑定。

### 6. 输入子系统与多点触摸（Multi-Touch Type B）驱动
* **传统输入子系统**：
  * 核心结构体 `struct input_dev`；
  * 注册流程：`devm_input_allocate_device()` → `input_set_capability()` → `input_register_device()`；
  * 事件上报：`input_report_key()` / `input_report_abs()` → `input_sync()`（上报 `SYN_REPORT` 同步帧）。
* **多点触控（MT 协议 Type B / 槽位管理机制）**：
  * 传统 Type A 协议每帧上报所有触点，开销大且多指轨迹追踪易混乱；
  * **Type B 槽位（Slot）协议核心步骤**：
    1. 初始化分配槽位：`input_mt_init_slots(input, max_touches, INPUT_MT_DIRECT);`
    2. 遍历触点上报：
       ```c
       input_mt_slot(input, slot_id);
       input_mt_report_slot_state(input, MT_TOOL_FINGER, active);
       if (active) {
           touchscreen_report_pos(input, &props, x, y, true);
       }
       ```
    3. 同步帧：`input_mt_sync_frame(input);` 与 `input_sync(input);`
  * 内核多点触控核心层自动维护 `ABS_MT_TRACKING_ID`，大幅简化上层手势识别与多指轨迹计算。

### 7. 其他基础外设框架（PWM、IIO/ADC、RTC、电源管理）
* **PWM 子系统**：统一抽象 `struct pwm_device`，使用现代化原子更新接口 `pwm_apply_state(pwm, &state)` 调节周期、占空比、极性与使能。
* **IIO 工业 I/O 子系统**：专用于各类传感器与 ADC 数据采集，通过 `struct iio_chan_spec` 规范物理通道，支持单次 Sysfs 原始值采集与基于 DMA/Buffer 的连续高速流采集。
* **RTC 子系统**：驱动实现 `struct rtc_class_ops`（提供 `read_time`、`set_time`、`set_alarm` 等回调），提供系统关机走时与定时唤醒能力。
* **电源管理（PM Framework）**：支持运行时电源管理（Runtime PM）动态关闭闲置模块时钟，以及系统休眠/唤醒（Suspend / Resume）时序控制。

---

## 四、 驱动进阶知识与高级系统架构

### 1. DMA 机制、Cache 一致性与 IOMMU
* **CPU Cache 一致性矛盾**：DMA 控制器直接读写系统内存而绕过 CPU L1/L2 Cache，导致 CPU 读出旧的“脏数据”或 Cache 写回冲刷掉外设写入的新数据。
* **两类 DMA 内存映射模式**：
  * **一致性 DMA（Coherent DMA）**：
    * 分配函数：`dma_alloc_coherent(dev, size, &dma_handle, GFP_KERNEL)`；
    * 特性：硬件自动通过总线窥探维护一致性，物理地址与虚拟地址均连续；内存开销大，专用于控制指令、环形缓冲区描述符。
  * **流式 DMA（Streaming DMA）**：
    * 特性：基于普通动态内存按需映射，单次批量传输效率高；
    * **关键要求**：**必须在软件层面依据传输方向手动处理 Cache 刷新与失效**（`dma_sync_single_for_cpu()` / `dma_sync_single_for_device()`）；适用于高带宽网络报文、音频与视频流数据。
* **IOMMU 的核心作用**：
  * **地址重映射**：将外部设备可见的 IOVA（I/O 虚拟地址）转换为真实的物理内存地址，打破 32 位设备无法访问 64 位大内存的限制。
  * **碎片重整（Scatter-Gather DMA）**：将物理内存中零散的物理页，在 IOVA 空间映射为连续的虚拟区间，支持外设一次性搬运大块离散数据。
  * **DMA-BUF 显存共享**：提供统一的零拷贝缓冲区分发机制，实现摄像头采集（V4L2）→ 视频硬件编解码（VPU）→ 屏幕渲染（DRM）的全链路显存零拷贝流转。

### 2. DRM/KMS 现代显示框架（对比传统 Framebuffer）
* **传统 Framebuffer 的局限**：单一图层、无硬件显存管理、无标准化 VSYNC 同步，极易发生画面撕裂和闪烁。
* **DRM/KMS（Kernel Mode Setting）四大核心组件**：
  ```
  [ GEM Framebuffer 显存 ] 
            ↓
  [ Plane (硬件图层) ]  ---> 负责图层混合、缩放、裁剪、叠放
            ↓
  [ CRTC (显示控制器) ] ---> 对应 RK356x 内部的 VOP2，生成时序信号并掌控刷新率
            ↓
  [ Encoder (编码器) ]  ---> 信号格式转码（输出 MIPI-DSI / HDMI / eDP / LVDS 时序）
            ↓
  [ Connector (连接器) ] -> 物理输出端口，检测热插拔状态并读取屏幕 EDID
  ```
* **GEM（图形内存管理器）与原子提交（Atomic Commit）**：
  * GEM 规范显存的分配、虚实映射与跨进程流转（结合 PRIME / DMA-BUF）；
  * **原子提交机制**：上层显示服务将分辨率、所有图层位置、显存指针一次性打包并原子写入硬件，在垂直消隐期间瞬间生效，彻底消灭显示撕裂。

### 3. 网络子系统、NAPI 机制与 PHY/MDIO 链路
* **核心数据结构**：
  * `sk_buff`（套接字缓冲区）：多层协议头与数据载体，通过指针前后移动避免重复内存拷贝。
  * `net_device`：网络设备实体，绑定 `net_device_ops`（启动 `ndo_open`、发包 `ndo_start_xmit` 等）。
* **NAPI 中断轮询混合机制**：
  * 首个报文到达触发硬件中断；
  * 中断服务程序立即**关闭网卡硬件接收中断**，将 NAPI 实体加入软中断轮询列表；
  * 在软中断轮询函数中按设定配额（Budget）**批量接收报文**；
  * 接收完毕后重新开启硬件中断；**彻底解决高并发网络环境下的“中断风暴”问题**。
* **MAC 控制器与 PHY 芯片分工**：
  * MAC 控制器（RK356x 内部 GMAC）负责数据链路层打包与 DMA 传输；
  * 外部 PHY 芯片负责物理层信号收发；二者数据走高速接口（如 RGMII），控制与自协商状态检测走 **MDIO/MDC 总线**，内核通过 PHYLINK 框架监听链路状态并自动同步 MAC 运行速率。

### 4. SMP 对称多核架构与从核唤醒流程
* **架构特点**：RK3568 的 4 个 Cortex-A55 核心共享一致性全局内存与外设资源，统一由 Linux 调度器分配任务。
* **多核启动闭环（ARMv8 PSCI 规范）**：
  1. 系统复位后，主核（CPU0 / BP 引导处理器）率先启动并初始化内核基本环境；
  2. CPU0 解析设备树 `/cpus` 节点（包含各核心拓扑及 `enable-method = "psci"` 属性）；
  3. CPU0 依次发起 `cpu_up`，通过 PSCI 规范向安全世界固件（EL3 / TF-A）发送 SMC 汇编指令唤醒指定从核；
  4. 从核（AP）解除硬件挂起，跳转至内核入口 `secondary_entry` 进行自身环境初始化；
  5. 从核逐步推进内核 `cpuhp_state` 状态机，最终在 `__cpu_active_mask` 位图中置位，正式纳入全局调度器开始分担系统负载。

---

## 五、 内核驱动开发避坑指南与最佳实践

1. **全面拥抱设备资源管理托管（`devm_*` 系列 API）**
   * 优先使用 `devm_kzalloc`、`devm_gpiod_get`、`devm_request_irq`、`devm_regmap_init_*`；
   * 在驱动加载失败（`probe` 返回错误）或模块卸载（`remove`）时，内核会自动逆序回收对应资源，大幅减少由于人为疏忽导致的内存泄漏与悬挂中断。

2. **严格遵守内核上下文红线**
   * **原子上下文（硬中断、Tasklet、定时器回调）**：严禁调用任何可能导致睡眠或进程调度的函数（如 `msleep`、`mutex_lock`、`copy_to_user`、非 `GFP_ATOMIC` 的内存申请）。
   * 任何需要耗时等待总线响应、进行通信阻塞的操作，必须委托给**工作队列（Workqueue）**或**线程化中断（Threaded IRQ）**等进程上下文中运行。

3. **彻底抛弃硬编码与过时 API**
   * 严禁在驱动中使用硬编码的物理寄存器地址，必须全部下沉到**设备树（DTS/DTBO）**中描述；
   * GPIO 控制一律采用 `Pinctrl + gpiod_*` 描述符规范，不再使用易冲突的旧全局数字编号（`gpio_*`）；
   * 复杂总线外设寄存器访问一律采用 `Regmap`；图形显示输出全面拥抱 `DRM/KMS` 标准接口。
