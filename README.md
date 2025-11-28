# Linux 内核中数据包的生命周期

原文：https://www.0xkato.xyz/life-of-a-packet-in-the-linux-kernel/

## Linux 中数据包的生命周期
从 `write()` 到 `recv()` 的实用指南

当你运行 `curl http://example.com`，终端里就显示出了一些 HTML，但实际发生了什么？Linux 通过一系列定义明确的小步骤来传递你的字节：选择路径、学习邻居的 MAC 地址、将数据包排队、让网卡发送它，然后在另一端反向执行这些操作。

这篇文章试图尽可能简单地解释这个过程。如果你使用过 Linux、运行过 `curl` 或用过 `ip addr` 命令，你就有资格阅读这篇文章。不需要深入的内核背景知识。

注意：当我在这篇文章中说"内核"时，我实际上指的是"Linux 内核及其网络栈"——运行在内核中并移动数据包的部分。

## 我们将涵盖的内容

这是我们将要讲解的简化路径：

```
你的应用
 ↓ write()/send()
TCP（分段你的字节）
 ↓
IP（选择发送目的地）
 ↓
Neighbor/ARP（查找下一跳 MAC）
 ↓
qdisc（队列管理、节奏控制）
 ↓
驱动/网卡（DMA 到硬件）
 ↓
线缆 / Wi‑Fi / 光纤
 ↓
网卡/驱动（另一台主机）
 ↓
IP（检查，决定是否给我们）
 ↓
TCP（重组、确认）
 ↓
服务器应用
```

## 第一部分 - 发送：从 `write()` 到线缆

### 步骤 1：你的应用将字节交给内核

你在 TCP 套接字上调用 `send()` 或 `write()`。内核接受你的缓冲区并排队准备发送。

- TCP 将大缓冲区分解为适合路径的段。每一方在 TCP 握手期间通告一个 MSS，发送方将其段大小限制为对方通告的 MSS，并受当前路径 MTU 和任何 IP/TCP 选项（例如时间戳）的进一步约束。
- 它为每个段标记序列号，以便另一端可以按顺序重组。

**小贴士 - 套接字**
套接字只是你程序的通信端点。对于 TCP，内核维护每个套接字的状态：序列号、拥塞窗口、定时器。

**小贴士 - TCP 握手**
在任何 `write()` 到达对端之前，TCP 会进行快速的三步设置：
1) 客户端 -> 服务器：带选项的 SYN（MSS、允许 SACK、窗口缩放、时间戳、ECN）。
2) 服务器 -> 客户端：带其选项的 SYN‑ACK。
3) 客户端 -> 服务器：ACK。双方就初始序列号和选项达成一致；状态变为 ESTABLISHED。
TLS 注意事项：对于 HTTPS，TLS 握手在 TCP 建立后运行。

**试试看**
在下载活动时运行 `ss -tni`。你会看到 TCP 发送和接收队列大小在数据在线路上传输和被应用消费时波动。

### 步骤 2：内核决定发送到哪里（路由）

内核查看目标 IP 并选择最佳匹配路由。在典型主机上，这归结为一个简单的问题：这个 IP 是在我的本地网络上，还是我将它交给网关？

- 如果地址在直连网络上，它从该接口出去。
- 否则，它去往你的默认网关（通常是你的路由器）。

**试试看**

```bash
ip route get 192.0.2.10
```

这会打印接口、下一跳（如果有）以及内核将使用的源 IP。

**小贴士 - 策略路由**
内核可以使用 `ip rule` 查询多个路由表（例如按源地址或标记选择路由）。大多数笔记本电脑和服务器使用主表。

### 步骤 3：内核学习下一跳 MAC（neighbor/ARP）

IP 路由选择下一跳。要实际发送以太网帧，内核需要该跳的 MAC 地址。

- 如果内核已经知道它（在邻居/ARP 缓存中），很好。
- 如果不知道，它会发送广播 ARP 请求："谁有 10.0.0.1？告诉我你的 MAC。" 回复会被缓存。

**试试看**

```bash
ip neigh show
```

你会看到类似 `10.0.0.1 lladdr 00:11:22:33:44:55 REACHABLE` 的条目。

**小贴士 - ARP vs NDP**
IPv4 使用 ARP（广播）。IPv6 使用邻居发现（组播）。相同的想法：为网络上的 IP 找到链路层地址。

### 步骤 4：数据包等待轮次（qdisc）

在网卡发送任何东西之前，数据包进入队列规则（qdisc）。你可以把这看作一个小等候队列加上一个交通警察，内核可以：

- 平滑突发，这样你就不会淹没链路并造成缓冲膨胀（大队列 -> 高延迟），
- 在不同流之间公平地共享带宽，以及
- 如果你配置了，执行整形/速率限制规则。

**试试看**

```bash
tc qdisc show dev eth0
tc -s qdisc show dev eth0 # 相同，但带计数器/统计信息
```

将 `eth0` 替换为你的实际接口名称（例如 `enp3s0`，`wlp2s0`）。

**小贴士 - MTU vs MSS**
MTU 是你的链路将承载的最大 L2 负载（典型以太网是 1500 字节）。
MSS 是段内最大的 TCP 负载，在 IP + TCP 头和选项之后。
在 TCP 握手期间，每一方通告它可以接收的 MSS，发送方不会发送大于对方通告 MSS 的段，并且还会遵守路径 MTU（PMTU）。
在 IPv4 的常见无选项情况下，MSS ≈ MTU − 40 字节。选项会进一步减少 MSS。

### 步骤 5：驱动和网卡完成繁重工作

内核的网络驱动将你的数据包交给网卡（NIC），并将其放入网卡读取的小发送队列中。然后网卡：

- 直接从 RAM 拉取字节（使用 DMA），并将它们转换为链路上的比特流，铜缆上的微小电压变化，光纤上的光脉冲，或者如果你使用 Wi-Fi 的话是无线电波。

这就是实际的"上线"时刻：内存中的数据变成网络上的信号。

**试试看**

```bash
ip -s link show dev eth0
ethtool -S eth0 # 网卡统计信息
ethtool -k eth0 # 启用的卸载功能
```

将 `eth0` 替换为你的实际接口名称。

**小贴士 - 卸载功能**
TSO/GSO：让网卡或协议栈将大缓冲区拆分为 MTU 大小的帧。
校验和卸载：在发送时，网卡在内核将数据包交给它之后、发送之前填充 IP/TCP 校验和，在接收时，网卡可以验证校验和并告诉内核结果。
GRO（接收时）：将许多小数据包合并成更大的块以节省 CPU。

**小贴士 - DMA**
直接内存访问（DMA）让网卡通过总线（例如 PCIe）直接在 RAM 中读/写你的数据，而无需 CPU 复制字节。这就是网卡如何高效地从发送环（和放置接收帧）拉取帧的方式。

### 步骤 6：在线缆上

在以太网上，网卡发送这样的帧：

```
[ 目标 MAC | 源 MAC | EtherType (IPv4) | IP 头 | TCP 头 | 负载 | FCS ]
```

交换机关心以太网头：它们查看目标 MAC 地址并从正确的端口转发帧。

路由器查看 IP 头，递减 TTL / 跳数限制，并（对于 IPv4）在将数据包转发到下一跳之前更新头校验和。

一跳一跳，每个交换机和路由器重复这个过程，直到路由器最终有一个直接到目标网络的路由，并将数据包传递到服务器的本地网络上。

**小贴士 - 帧 vs 数据包**
数据包是 IP 层单元（IP 头 + TCP/UDP + 负载）。
帧是该数据包在特定链路（例如以太网）上如何携带的，带有源/目标 MAC 和校验和。

## 第二部分 - 接收：从线缆回到你的应用

### 步骤 7：网卡将数据交给内核（NAPI）

在服务器上，网卡将传入的帧写入接收环（内存中的小队列）。然后 Linux 内核使用 NAPI 高效地拉取它们：它获得一个快速中断，然后切换到轮询以一次处理一批数据包。

**小贴士 - NAPI**
如果每个数据包都触发完整的中断，繁忙的网卡可能会压垮 CPU。NAPI 的技巧是：

- 触发一次中断，
- 暂时切换到轮询以排空一批数据包，
- 然后重新启用中断。

更少的中断，更好的吞吐量。

### 步骤 8：IP 检查数据包并决定做什么

内核验证 IP 头（版本、校验和、TTL 等），然后问："这个数据包是给我的吗？"

- 如果目标 IP 与服务器的某个地址匹配，它是本地的并向上移动到协议栈。
- 如果不是，并且启用了 IP 转发，内核可能会转发它，这就是 Linux 机器作为路由器的行为方式。
- 否则，数据包被丢弃。

如果你使用防火墙，这是 PREROUTING 和 INPUT（nftables/iptables）等钩子可以在数据包传递到本地套接字之前过滤、记录或 DNAT 流量的地方。SNAT/MASQUERADE 发生在 POSTROUTING 中。DNAT 也可以在 OUTPUT 中对本地生成的数据包发生。

**试试看**

```bash
sudo nft list ruleset
# 或者，使用 iptables：
sudo iptables -L -n -v
sudo iptables -t nat -L -n -v
```

### 步骤 9：TCP 重组、确认并唤醒应用

TCP 协议栈按顺序放置段，检查缺失的片段，并发送 ACK。当有数据准备好时，它唤醒在 `recv()` 中等待的进程。

**试试看**

```bash
ss -tni 'sport = :80 or dport = :80'
```

观察接收队列（Recv-Q）随着应用读取而增长和收缩。

## 简短实用说明

### 回环是特殊的（且快速）

发往 `127.0.0.1` 的数据包永远不会触及物理网卡。路由仍然发生，但所有东西都留在纯软件的 `lo` 接口的内存中。

### 桥接 vs 路由（同一台机器，不同角色）

如果机器是网桥（例如使用 `br0`），它在第 2 层转发帧，不改变 TTL。如果它在路由，它在第 3 层转发，TTL 减少一跳。

### NAT 发夹（为什么内部客户端访问外部 IP）

从同一 LAN 通过路由器的公共 IP 访问服务需要"发夹 NAT"。如果在这种情况下连接重置，检查 `PREROUTING` 和 `POSTROUTING` NAT 规则。

### IPv6

将 ARP 换成 NDP。否则，路径是相同的：

```bash
ip -6 route
ip -6 neigh
```

### UDP 是不同的（有意为之）

UDP 不做排序、重传或拥塞控制。发送路径使用 udp_sendmsg，接收路径传递整个数据报。你的应用处理丢失。

### 亲自看看（10 个快速命令）

```bash
# 1) 内核会将数据包发送到哪里？
ip route get 192.0.2.10

# 2) 存在哪些路由和规则？
ip route; ip rule

# 3) 谁是我的下一跳？
ip neigh show

# 4) 我的防火墙/NAT 在做什么？
sudo nft list ruleset
# 或者：
sudo iptables -L -n -v
sudo iptables -t nat -L -n -v

# 5) 哪些套接字是活动的？
ss -tni

# 6) 线缆上有什么（根据需要替换 eth0/host）？
sudo tcpdump -ni eth0 -e -vvv 'host 192.0.2.10 and tcp port 80'

# 7) 我的队列健康吗？
tc -s qdisc show dev eth0

# 8) 我的网卡正常吗？
ip -s link show dev eth0
ethtool -S eth0

# 9) 计数器是否暗示有问题？
nstat -a | grep -E 'InErrors|OutErrors|InNoRoutes|InOctets|OutOctets'
# （如果你明确想要将计数器归零，使用 `-z` 而不是 `-a`。）

# 10) 路径 MTU 安全吗？
tracepath 192.0.2.10 # 通过 ICMP 发现 PMTU：IPv4 "需要分片"（类型 3，代码 4）/ IPv6 "数据包太大"（类型 2）
```

### ARP/邻居问题

ip neigh 显示 FAILED 或状态不断翻转 -> L2 可达性、VLAN 标记或交换机过滤问题。

### MTU / PMTU 黑洞

小 ping 工作，大传输停滞 -> MTU 不匹配或 ICMP 被阻止。
允许 PMTU 信号通过你的防火墙（IPv4：ICMP 类型 3 代码 4 "需要分片"，IPv6：ICMPv6 类型 2 "数据包太大"）或修复 MTU。

### 反向路径过滤器咬人

非对称路由 + rp_filter=1 丢弃返回流量。使用 rp_filter=2（宽松）或使路由对称。

### NAT 意外

SNAT/MASQUERADE 错误地重写源，所以回复无处可去。检查 NAT 规则和 conntrack -L。

### 积压/接受压力

负载下新连接重置 -> 增加应用积压和 net.core.somaxconn，确保应用及时 accept()。

### 突发导致的缓冲膨胀

大队列，大延迟尖峰 -> 为 qdisc 选择 fq_codel（或 fq），并在你的应用中启用节奏控制（如果可用）。

## 内核调用路径（如果你感兴趣）

### 发送（典型 TCP 路径）

```
tcp_sendmsg
 -> tcp_push_pending_frames
 -> __tcp_transmit_skb
 -> ip_queue_xmit
 -> ip_local_out / ip_output
 -> ip_finish_output
 -> neigh_output
 -> dev_queue_xmit
 -> qdisc / sch_direct_xmit
 -> ndo_start_xmit（驱动）
```

### 接收（典型 IPv4 TCP 路径）

```
napi_gro_receive / netif_receive_skb
 -> __netif_receive_skb_core
 -> ip_rcv
 -> ip_rcv_finish
 -> ip_local_deliver
 -> ip_local_deliver_finish
 -> tcp_v4_rcv
 -> tcp_v4_do_rcv
 -> tcp_data_queue（唤醒读取器）
```

### 一个小清单放在手边

- **Socket（套接字）** - 你程序的网络 I/O 句柄。
- **MTU / MSS** - 最大链路负载 / 最大 TCP 负载。
- **ARP / NDP** - 查找链路层地址（IPv4 / IPv6）。
- **qdisc** - 每设备队列策略（公平性、整形）。
- **NAPI** - 高效接收：中断，然后轮询一批。
- **TSO/GSO/GRO** - 卸载以拆分/合并数据包并节省 CPU。
- **Conntrack** - 内核的流表（由 NAT 和过滤使用）。
- **PREROUTING/INPUT/OUTPUT/POSTROUTING** - 防火墙钩子点。
- **DMA（直接内存访问）** - 硬件读/写 RAM 无需 CPU 复制，网卡将此用于 TX/RX 环。
- **TTL / 跳数限制** - 每个路由器递减的每包计数器（IPv4 中的 TTL，IPv6 中的跳数限制）。当它达到零时，数据包被丢弃。
- **FCS（帧校验序列）** - 以太网帧末尾的链路层 CRC，用于检测线路上的位错误。



