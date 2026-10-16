# 《TCP/IP 详解 · 卷2：实现》（Wright & Stevens）核心重点笔记

> **书名**：TCP/IP Illustrated, Volume 2: The Implementation  
> **作者**：Gary R. Wright、W. Richard Stevens  
> **定位**：网络协议栈内核级工程实现的“旷世巨著”。全书通过解析约 15,000 行 4.4BSD-Lite 核心网络源代码，完整揭秘了操作系统内核中网络协议栈的底层数据结构、内存组织与算法流转。

---

## 一、 协议栈内存骨架：mbuf（存储器缓存）

### 1. `struct mbuf` 的精妙架构
* **设计意图**：网络报文在内核各层（链路层、IP 层、传输层、Socket 层）流转时，必须频繁添加、剥离协议首部，且报文长度极不规则。mbuf 专为零内存拷贝设计。
* **物理规格与变体（标准大小 128 字节）**：
  1. **小数据 mbuf**：内部保留 108 字节直接存放数据；
  2. **分组首部 mbuf（包含 `pkthdr`）**：头部带有整包长度与接收网卡接口指针，留有 100 字节数据区；
  3. **簇 mbuf（Cluster mbuf）**：当数据超过 208 字节时，mbuf 本身只作为控制头，其 `m_ext` 结构体指向外部 2048 字节（`MCLBYTES`）的独立页面缓冲区。
* **数据结构图解**：
  ```
  [ struct m_hdr ] -> m_next (同一数据报的下一个 mbuf 片段)
                   -> m_nextpkt (队列中下一个独立数据报的首个 mbuf)
                   -> m_data (指向数据实际起始位置的动态滑动指针)
                   -> m_len (当前 mbuf 中有效数据长度)
                   -> m_flags (M_PKTHDR, M_EXT 等)
  ```
* **核心工具函数**：
  * `m_get()` / `m_gethdr()`：从空闲链表中分配一个 mbuf 结构；
  * `m_free()` / `m_freem()`：释放单个 mbuf 或整条链表；
  * `m_pullup(m, len)`：将链表中分散在前几个 mbuf 的协议头部数据重组整理，确保首部连续存放在首个 mbuf 中，便于直接指针强转读取协议头字段。

---

## 二、 接口层与路由体系

### 1. 网络接口抽象 `struct ifnet` 与 `struct ifaddr`
* **`ifnet`（网卡硬件抽象）**：记录接口名称、MTU、硬件标志（`IFF_UP`、`IFF_BROADCAST`、`IFF_SIMPLEX`）、输出函数指针（`if_output`）、启动函数指针（`if_start`）及收发统计。
* **`ifaddr`（网络地址抽象）**：每个网卡可以绑定多个不同协议簇的地址（如同时具备 IPv4 与 IPv6），通过单链表挂载在 `ifnet` 下。
* **`sockaddr` 家族**：
  * `sockaddr_in`（IPv4 地址结构）；
  * `sockaddr_dl`（链路层物理地址结构，存储 MAC 与网卡索引）。

### 2. 路由表（Radix 树实现）
* 4.4BSD-Lite 采用 **Patricia 树（Radix 树变体）** 组织内核路由表；
* **最长前缀匹配（LPM）**：通过逐位二分比对键值掩码，快速检索最优路由条目 `struct rtentry`，并获取下一跳网关地址及出口网卡指针。

### 3. ARP 实现机制
* ARP 缓存表挂载在路由项的链路层地址中；
* 报文外发时若未命中 ARP 缓存，内核自动调用 `arprequest()` 发送广播 ARP 请求，并将待发报文挂入 `llinfo_arp` 的悬挂队列；
* 收到 ARP 应答后，在软中断 `arpintr()` 中刷新缓存，并批量发空悬挂的数据报。

---

## 三、 网络层实现：IP 协议引擎

### 1. IP 接收处理 `ip_input()`
* 硬件中断触发网卡接收，网卡驱动将数据组装为 mbuf 链表，放入 `ipintrq` 队列并触发软中断；
* **`ip_input()` 主流程**：
  1. 校验首部长度与 `m_pullup` 连续性校验；
  2. 计算并校验 IP 首部校验和（`ip_sum`）；
  3. 检查目的 IP 是否为本机单播/广播/组播，非本机则在使能 IP 转发时交由 `ip_forward()`；
  4. 解析 IP 选项（严格源路由、松散源路由、记录路由、时间戳）；
  5. 若遇到分片报文（`MF=1` 或 `Offset>0`），进入 `ip_reass()` 进行分片重组。

### 2. IP 分片重组算法 `ip_reass()`
* 内核维护全局哈希表 `ipq`（按源 IP、目的 IP、协议号、ID 进行哈希）；
* 将相同 IP 标识的片段插入链表，按照偏移量排序；
* 启动重组定时器（默认 30 秒超时），超时未集齐则丢弃全部已收分片并返回 ICMP 超时报文；
* 当检测到连续分片且首尾完整拼接时，组装为一个完整的数据报传递给上层传输层。

### 3. IP 发送处理 `ip_output()`
* 负责路由查找（命中则返回对应 `ifnet` 与下一跳 `sockaddr`）；
* 处理 IP 选项扩展；
* 检查报文总长是否超过出口网卡 MTU，超过则执行物理切片生成分片 mbuf 链表；
* 填充 IP 校验和并调用对应驱动的 `if_output` 接口。

---

## 四、 套接字层（Socket Layer）与协议控制块（PCB）

### 1. 套接字数据结构 `struct socket`
* 负责联结用户空间进程与内核网络协议；
* **发送/接收缓冲区 `struct sockbuf`**：
  * 维护报文链表指针 `sb_mb`、缓冲区配额（`sb_hiwat`）与当前已用字节（`sb_cc`）；
  * 具备睡眠/唤醒等待标志（`SB_WAIT`），为 `select` / `poll` 阻塞与非阻塞提供底层支持；
* **系统调用桥梁**：
  * `socreate()`、`sobind()`、`solisten()`、`soaccept()`；
  * `sosend()` 将用户态数据拷贝入 mbuf 并交由协议层输出；
  * `soreceive()` 从接收缓冲区提取 mbuf 拷贝回用户态空间。

### 2. 协议控制块（Internet PCB）
* `struct inpcb`：记录四元组信息（源 IP、源端口、目的 IP、目的端口）；
* **查找机制 `in_pcblookup()`**：当底层收到一个报文时，在全局 PCB 链表中执行哈希与匹配，优先寻找精确匹配四元组的已连接 Socket；未命中则寻找匹配本地 IP 和端口的监听 Socket。

---

## 五、 传输层核心实现：TCP 协议栈引擎

### 1. TCP 控制块 `struct tcpcb`
* 每个活动的 TCP 连接独占一个 `tcpcb`，直接与 `inpcb` 互相引用；
* 维护全量连接状态：连接状态 `t_state`、定时器数组 `t_timer`、发送序列号指针（`snd_una`, `snd_nxt`, `snd_max`, `snd_wnd`）、接收序列号指针（`rcv_nxt`, `rcv_wnd`）、拥塞窗口 `snd_cwnd`、慢启动阈值 `snd_ssthresh` 等。

### 2. 双定时器轮询机制
BSD 内核通过两个全局心跳定时器驱动整个 TCP 状态推进：
1. **快定时器 `tcp_fasttimo()`（每 200ms 执行一次）**：
   * 专用于延时确认（Delayed ACK）处理；
   * 若设置了 `TF_DELACK` 标志，则立即发送一个纯 ACK 报文。
2. **慢定时器 `tcp_slowtimo()`（每 500ms 执行一次）**：
   * 递减连接的 4 类定时器计数：
     * **重传定时器（`TCPT_REXMT`）**：超时则触发快速指数退避与报文重传；
     * **坚持定时器（`TCPT_PERSIST`）**：在对端通告零窗口时定期发送探测包；
     * **保活定时器（`TCPT_KEEP`）**：长连接心跳探测；
     * **2MSL 定时器（`TCPT_2MSL`）**：驱动 TIME_WAIT 状态结束并释放 `tcpcb` 资源。

### 3. TCP 输入引擎 `tcp_input()`
* **头部预测（Header Prediction，极速快路径）**：
  * 针对最常见的纯单向数据流或纯 ACK 场景设计；
  * 仅做几条关键寄存器比对（序列号等于 `rcv_nxt`，无异常标志），直接将数据入队并触发 `sbwakeup()`，绕过庞大完整的状态机，获得极限吞吐。
* **慢路径处理（完整状态机处理）**：
  * 检查 SYN / FIN / RST 标志，推进 11 种状态流转；
  * 解析选项（MSS、Window Scale、Timestamp）；
  * 处理失序数据报：失序数据被挂入 `t_segq` 重组队列，待空缺序列号集齐后合并呈递。

### 4. TCP 输出引擎 `tcp_output()`
* 依据当前拥塞窗口 `snd_cwnd` 与对端通告窗口 `snd_wnd` 计算实际允许外发的字节数；
* 构建 TCP 协议头，计算滑动窗口通告，写入 SYN/FIN/ACK 等标志位；
* 触发 IP 层 `ip_output()`。

---

## 六、 架构精髓与工程启示

1. **零拷贝设计哲学**：mbuf 的链表式组织与指针动态偏移，使得跨层流转无需任何物理内存拷贝；
2. **分层清晰与面向对象多态**：通过 `protosw` 结构体注册各协议操作集（`pr_input`, `pr_output`, `pr_usrreq`），使得添加新协议无需改动内核框架；
3. **软中断解耦**：硬件中断严格限制在驱动层快速拉取网卡 DMA 描述符，耗时的协议解析全量移交给内核软中断队列异步执行。
