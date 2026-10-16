# 《操作系统导论》（OSTEP · Arpaci-Dusseau 夫妇）核心重点笔记

> **书名**：Operating Systems: Three Easy Pieces (OSTEP)  
> **作者**：Remzi H. Arpaci-Dusseau、Andrea C. Arpaci-Dusseau（威斯康星大学麦迪逊分校教授）  
> **定位**：现代计算机操作系统教学的巅峰标杆之作。全书围绕**“虚拟化（Virtualization）”、“并发（Concurrency）”和“持久性（Persistence）”**三大核心概念，以风趣深刻的语言和真实可运行的 C 语言工程代码，抽丝剥茧地揭示了现代 OS 的底层精髓。

---

## 一、 第一要素：虚拟化（Virtualization）

### 1. CPU 虚拟化与受限直接执行（Limited Direct Execution, LDE）
* **核心难题**：操作系统如何既能利用 CPU 的硬件速度直接运行程序，又能牢牢掌控对系统的绝对控制权？
* **LDE 机制运行全景**：
  * **特权级划分**：用户模式（User Mode，受限指令集）与内核模式（Kernel Mode，全权限特权指令）；
  * **陷阱与陷阱表（Trap & Trap Table）**：操作系统启动时初始化硬件陷阱表；用户程序通过发起 `trap` 指令触发系统调用，硬件自动将用户寄存器压入内核栈，并跳转到内核预设的处理程序；处理完成后通过 `return-from-trap` 特权指令恢复用户态；
  * **时钟中断（Timer Interrupt）**：解决恶意死循环夺回控制权的关键。时钟芯片每隔数毫秒产生一次硬件中断，强制将 CPU 控制权交还给 OS 调度器。

### 2. 进程调度黄金算法（从 FIFO 到 MLFQ）
* **调度度量**：周转时间（$T_{\text{周转}} = T_{\text{完成}} - T_{\text{到达}}$）与响应时间（$T_{\text{响应}} = T_{\text{首次运行}} - T_{\text{到达}}$）。
* **多级反馈队列（MLFQ，Multi-Level Feedback Queue）的五大铁律**：
  * **规则 1**：如果 Priority(A) > Priority(B)，A 运行（B 不运行）；
  * **规则 2**：如果 Priority(A) == Priority(B)，A 和 B 轮转（Round-Robin）运行；
  * **规则 3**：工作进入系统时，放在最高优先级；
  * **规则 4（防欺骗）**：一旦工作用完了其在某一层的时间配额（无论中间经历了多少次主动让出），其优先级都会降低一级；
  * **规则 5（防饥饿）**：经过一段时间 $S$，将系统中所有工作重新推入最高优先级队列（Priority Boost）。

### 3. 内存虚拟化（Paging, TLB 与 Swapping）
* **分页机制（Paging）**：
  * 虚拟地址划分为虚拟页号（VPN）与页内偏移（Offset），通过页表映射到物理页帧号（PFN）；
  * **转换后备缓冲器（TLB，快表）**：芯片上的地址转换硬件缓存，使虚拟地址转换绝大多数在 1 个时钟周期内命中（TLB Hit）；在上下文切换时，通过地址空间标识符（ASID）区分不同进程的 TLB 缓存项。
* **多级页表（Multi-level Page Table）**：
  * 解决单级线性页表占用海量内存的问题；
  * 引入页目录（Page Directory），只有实际映射使用的虚存区域才分配二级页表，未分配区域的二级页表无需常驻物理内存，节约 90% 以上页表开销。
* **时钟置换算法（Clock Algorithm）**：
  * LRU（最近最少使用）的高效工程近似；
  * 每个物理页维护一个访问引用位（Use bit）；发生缺页时，时钟指针循环遍历页面，遇到引用位为 1 则清 0，遇到引用位为 0 则直接将其驱逐换出。

---

## 二、 第二要素：并发（Concurrency）

### 1. 锁与硬件原语演进
* **软件层面的互斥困境**：纯软件无法在没有原子指令的情况下安全实现互斥锁；
* **关键硬件指令**：
  * **测试并设置（Test-and-Set / TAS）**：原子读取旧值并写入新值（自旋锁 Spinlock 的基础）；
  * **比较并交换（Compare-and-Swap / CAS）**：`CAS(ptr, expected, new)`，无锁编程基石；
  * **获取并增加（Fetch-and-Add）**：票据锁（Ticket Lock），天然保障排队先来后到的公平性。
* **两阶段锁（Two-Phase Lock / Linux Futex）**：
  * 第一阶段短时间自旋尝试获取锁（若持有者很快释放，避免线程休眠开销）；
  * 第二阶段若依然未获取，调用系统调用（如 `futex`）让出 CPU 进入内核等待队列休眠，直到被唤醒。

### 2. 条件变量与生产者-消费者规范
* **两大死锁与竞态避坑铁律**：
  1. **互斥锁与条件变量必须配对**：调用 `pthread_cond_wait()` 时必须持有锁，该函数原子性地释放锁并挂起线程；被唤醒后自动重新抢夺锁；
  2. **必须使用 `while` 循环而非 `if`**（Mesa 语义要求）：
     ```c
     pthread_mutex_lock(&mutex);
     while (count == MAX) {           // 绝不能写成 if (count == MAX)
         pthread_cond_wait(&empty, &mutex);
     }
     put(item);
     pthread_cond_signal(&fill);
     pthread_mutex_unlock(&mutex);
     ```

### 3. 死锁发生的四大必要条件（科夫曼条件）与破除之道
1. **互斥（Mutual Exclusion）**：资源不能共享；
2. **持有并等待（Hold and Wait）**：持有已分配资源并等待新资源（*破除：原子化一次性申请所有锁*）；
3. **非抢占（No Preemption）**：资源不能被强制剥夺（*破除：使用 `pthread_mutex_trylock`，失败则释放已持有锁*）；
4. **循环等待（Circular Wait）**：存在闭环依赖链（*破除：**全局严格规定加锁次序（Lock Ordering）**，最通用且工程可行的死锁预防策略*）。

---

## 三、 第三要素：持久性（Persistence）

### 1. 磁盘与 RAID 架构
* **机械硬盘耗时分析**：寻道时间（Seek Time，机械臂移动）+ 旋转延迟（Rotational Delay）远大于电子数据传输时间，因此**顺序 I/O 吞吐比随机 I/O 高 1~2 个数量级**。
* **RAID 核心等级对比**：
  * **RAID 0（条带化）**：容量与吞吐为 $N$，无任何冗余，损坏单盘全量数据丢失；
  * **RAID 1（镜像）**：容量为 $N/2$，高容错，读性能翻倍，写性能受限；
  * **RAID 5（分布式奇偶校验）**：容许任意一块单盘损坏，通过将校验块均摊分布在各盘（Rotated Parity），打破了 RAID 4 单一校验盘的集中写入瓶颈。

### 2. 文件系统崩溃一致性（Crash Consistency）与日志（Journaling）
* **崩溃一致性问题**：往文件写入新块需要执行三次写操作：更新 Inode、更新数据块位图（Bitmap）、写入实际数据块。若在写操作间隙系统突然断电，会导致文件系统元数据严重错乱。
* **预写式日志（WAL / Write-Ahead Logging）执行时序**：
  1. **日志写入（Journal Write）**：将事务内容（TxBegin、待修改元数据/数据）写入磁盘预留的连续日志区；
  2. **事务提交（TxCommit）**：向日志区写入 TxEnd 标记（**原子落盘，成功标志事务生效**）；
  3. **检查点刷盘（Checkpointing）**：将更新正式写入文件系统真正的物理 Inode 和数据块中；
  4. **释放日志**：清空已完成的日志空间。
* **崩溃恢复逻辑**：开机后仅需扫描日志区，未标记 Commit 的事务直接丢弃，已标记 Commit 的事务重新执行检查点（Redo），恢复仅需数秒，无需遍历全盘执行低效的 `fsck`。

### 3. 写时复制（COW）与日志结构文件系统（LFS）
* **彻底消除就地覆写（Overwrite）**：所有对文件和元数据的修改，永远追加写入新的空白块；
* **现代文件系统（ZFS、Btrfs）的根基**：天然具备瞬时快照（Snapshot）与回滚能力，从物理层面根除了断电元数据半死不活的可能。
