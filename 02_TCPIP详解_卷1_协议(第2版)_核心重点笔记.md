# 《TCP/IP 详解 · 卷1：协议》（原书第2版）核心重点笔记

> **书名**：TCP/IP Illustrated, Volume 1: The Protocols (2nd Edition)  
> **作者**：Kevin R. Fall（IETF 资深成员、贝尔实验室/卡内基梅隆大学研究员）、W. Richard Stevens（网络技术巨擘）  
> **定位**：计算机网络协议层面的“圣经”，第2版彻底重写并引入了现代 IPv6、MPLS、高级 TCP 拥塞控制（CUBIC、SACK）与网络安全体系。

---

## 一、 网络互联体系与分层模型

### 1. 分层架构与核心原则
* **四层体系结构**：
  * **应用层（Application）**：HTTP、DNS、SSH、SMTP 等；
  * **传输层（Transport）**：TCP（面向连接、可靠字节流）、UDP（无连接、尽力而为数据报）；
  * **网络层（Network）**：IPv4、IPv6、ICMP、IPsec、路由协议，提供端到端寻址与分组转发；
  * **链路层（Link / Host-to-Network）**：以太网、Wi-Fi（802.11），处理相邻节点间的帧传输与介质访问控制。
* **端到端原则（End-to-End Argument）**：
  * 网络核心（路由器/交换机）只负责快速、尽力而为的数据包转发；
  * 状态维护、可靠性保障、流量与拥塞控制由网络边缘的端系统（主机）实现。

---

## 二、 网络层协议（IPv4、IPv6、ARP 与 ICMP）

### 1. IPv4 与 IPv6 报文头对比
* **IPv4 首部（20 字节固定 + 可变选项）**：
  * 关键字段：`TTL`（生存时间，防路由环路）、`Protocol`（上层协议标识，如 6-TCP、17-UDP）、`Identification`、`Flags`（DF 不分片、MF 更多分片）、`Fragment Offset`。
  * **分片与重组风险**：分片在源端或中间路由器产生，但**重组仅在最终目的端完成**。分片中任一包丢失会导致整报文重传，现代网络强烈推荐使用 **PMTUD（路径 MTU 发现）** 结合 `DF=1` 避免网络层分片。
* **IPv6 演进（40 字节定长基本首部）**：
  * 移除首部校验和（下沉由链路层与传输层承担，加速转发）；
  * 移除中间路由器分片能力（仅源端可分片，必须强制使用 PMTUD）；
  * 引入扩展首部链表机制（Hop-by-Hop、Routing、Fragment、Security/ESP 等）；
  * 地址空间扩展至 128 位，原生支持组播（Multicast）并废除广播（Broadcast）。

### 2. 地址解析（ARP 与 IPv6 NDP）
* **ARP（IPv4）**：
  * 广播询问（`Who has IP?`），单播应答（`I am MAC`）；
  * **免费 ARP（Gratuitous ARP）**：用于检测 IP 地址冲突（ACD），以及在网卡更换或 VRRP 主备切换时刷新相邻设备的 ARP 缓存表。
* **邻居发现协议 NDP（IPv6）**：
  * 彻底废除广播，基于 ICMPv6 组播报文（邻居请求 NS、邻居通告 NA）；
  * 支持 **SLAAC（无状态地址自动配置）**：主机结合路由器通告（RA）的前缀与自身 MAC 派生（EUI-64）直接生成全球唯一单播地址。

### 3. ICMP 与 ICMPv6
* **差错报文类型**：目的不可达（网络/主机/端口不可达）、超时（TTL 减至 0，`traceroute` 的实现基础）、参数问题。
* **查询报文**：回显请求/应答（Echo Request / Reply，`ping` 命令的基础）。
* **安全红线**：针对 ICMP 差错报文本身、广播/组播地址报文，绝不产生新的 ICMP 差错报文，严防网络风暴。

---

## 三、 传输层核心：TCP 协议机制与状态机

### 1. TCP 核心特征
* 面向连接、点对点、全双工、基于**连续无边界的可靠字节流**；
* 传输保障：校验和、序列号（Sequence Number）、确认应答（ACK）、超时重传、流量控制（滑动窗口）、拥塞控制。

### 2. 连接生命周期管理（三次握手与四次挥手）

```
      Client                                   Server
        |                                        |
        | -------- 1. SYN (seq=x) -------------> |  (Listen -> SYN_RCVD)
(SYN_SENT) |                                     |
        | <------- 2. SYN+ACK (seq=y, ack=x+1) - |
(ESTABLISHED) |                                  |
        | -------- 3. ACK (ack=y+1) -----------> |  (ESTABLISHED)
        |                                        |
        |             数据全双工传输             |
        |                                        |
        | -------- 1. FIN (seq=u) -------------> |  (CLOSE_WAIT)
(FIN_WAIT_1) |                                   |
        | <------- 2. ACK (ack=u+1) ------------ |
(FIN_WAIT_2) |                                   |
        | <------- 3. FIN (seq=w, ack=u+1) ----- |  (LAST_ACK)
        | -------- 4. ACK (ack=w+1) -----------> |  (CLOSED)
(TIME_WAIT)                                      |
  [等待 2MSL 后彻底关闭]
```

* **三次握手本质**：双方各自协商并同步初始序列号（ISN），并交换 MSS、窗口缩放（Window Scale）、SACK 允许等选项。
* **TIME_WAIT 状态核心价值（维持 2MSL，通常 1~2 分钟）**：
  1. **防止前一个连接的迟到重复数据段**干扰后续新建的相同四元组连接（让网络中残留报文全部自然消亡）；
  2. **确保可靠终止全双工连接**：若 Client 最后一个 ACK 丢失，Server 会超时重发 FIN，处于 TIME_WAIT 的 Client 才能重新应答 ACK，避免 Server 收到新建连接报文返回 RST。

---

## 四、 TCP 高级控制与传输算法

### 1. 超时与重传时间估计（RTT & RTO）
* **Jacobson 算法（平滑 RTT 与方差跟踪）**：
  * $\text{SRTT} \leftarrow (1 - \alpha) \times \text{SRTT} + \alpha \times \text{RTT}_{\text{sample}}$
  * $\text{RTTVAR} \leftarrow (1 - \beta) \times \text{RTTVAR} + \beta \times |\text{SRTT} - \text{RTT}_{\text{sample}}|$
  * $\text{RTO} = \text{SRTT} + 4 \times \text{RTTVAR}$
* **Karn 算法**：
  * **在重传期间，绝不对重传数据包的 RTT 进行采样**（消除二义性）；
  * 发生重传后，RTO 强制采用指数退避（Exponential Backoff，翻倍增加，直至 64 秒上限）。

### 2. 滑动窗口与流量控制
* **接收端窗口通告（`rwnd`）**：防止发送方过快灌入数据淹没接收方缓冲区。
* **糊涂窗口综合征（Silly Window Syndrome, SWS）**：
  * 现象：接收方释放极小空间（如几个字节）立即通告，发送方立即发送极小报文，报头开销巨大造成网络严重浪费；
  * **接收方解决方案（Clark 算法）**：除非空闲缓冲区达到 MSS 的一半或接收缓冲区总容量的一半，否则强制通告 `rwnd = 0`；
  * **发送方解决方案（Nagle 算法）**：若存在未确认的在途数据（In-flight），则只有攒满一个 MSS 数据或收到前序 ACK 时才允许发送小包。
* **经典性能死锁（Nagle 与 Delayed ACK 的冲突）**：
  * 接收方启用延时确认（Delayed ACK，等待 40ms~200ms 合并 ACK）；
  * 发送方启用 Nagle 算法（等待前序 ACK 才发新小包）；
  * 导致微小 RPC 交互产生高达 40~200ms 的无端延迟；高并发低时延网络编程需通过套接字选项 `TCP_NODELAY` 关闭 Nagle。

### 3. 拥塞控制经典四大演化（Tahoe → Reno → NewReno → CUBIC）
* **拥塞窗口（`cwnd`）与发送上限**：$\text{W} = \min(\text{cwnd}, \text{rwnd})$。
* **慢启动（Slow Start）**：`cwnd` 从 1~10 个 SMSS 开始，每收到一个 ACK，`cwnd` 增加 1 个 SMSS（指数级爆发增长），直至触达慢启动阈值 `ssthresh`。
* **拥塞避免（Congestion Avoidance）**：进入线性增长阶段，每个 RTT 时间内 `cwnd` 仅增加 1 个 MSS（加法增大 AIMD）。
* **快速重传与快速恢复（Fast Retransmit & Fast Recovery）**：
  * 收到 **3 个重复确认（Duplicate ACK）** 时判定发生个别丢包（而非网络彻底瘫痪）；
  * 立即重传丢失报文，无需等待漫长的 RTO 超时；
  * `ssthresh = cwnd / 2`，`cwnd = ssthresh + 3`，维持在拥塞避免状态（乘法减小）。
* **现代主力算法（CUBIC）**：
  * 基于三次函数的增长曲线，将拥塞控制从 RTT 强绑定中解耦，在长肥网络（BDP 巨大网络）下具备极高的带宽利用率与稳定性。

---

## 五、 现代网络协议扩展与安全

1. **TCP 核心扩展选项**：
   * **Window Scale（窗口缩放，RFC 1323）**：将 16 位窗口字段左移至多 14 位，支持高达 1GB 的超大滑动窗口；
   * **Timestamps（时间戳，PAWS 算法）**：解决在超高速千兆/万兆网络中 32 位序列号快速环绕（Wrapped Sequence）带来的旧报文混淆问题；
   * **SACK（选择性确认）**：通告非连续接收的块区间，允许发送方仅针对性重传真正丢失的分段。
2. **端到端安全传输**：
   * **IPsec**：网络层透明加密与认证（AH 认证头、ESP 封装安全载荷、IKE 密钥交换）；
   * **TLS**：传输层之上的安全握手与记录协议，实现身份认证、对称密文通信与完整性校验。
