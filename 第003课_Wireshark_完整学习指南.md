# 第三课：Wireshark 完整学习指南

> **实验范围：你自己的 CyberLab**
>
> 本课程只用于你拥有并控制的虚拟机。不要在学校、公司、公共 Wi‑Fi、邻居网络或未经授权的系统上抓包。

---

# 1. 本课目标

完成本课后，你应该能够：

```text
理解 Wireshark 的用途
选择正确的网卡
区分 Capture Filter 与 Display Filter
分析 ARP、ICMP、TCP、UDP、HTTP、SSH 和 TLS
识别 TCP 三次握手与连接关闭
观察 Nmap 与 Netcat 产生的流量
保存、打开与导出 PCAP/PCAPNG
使用 Endpoints、Conversations、Protocol Hierarchy 和 I/O Graph
结合 Wazuh 进行网络与端点日志关联分析
```

CyberLab：

```text
Kali-CyberLab      10.10.20.10
Ubuntu-Target      10.10.20.20
Windows-Target     10.10.20.30
Wazuh-SIEM         10.10.20.40
Network            10.10.20.0/24
```

---

# 2. 什么是 Wireshark？

Wireshark 是图形化网络协议分析工具，可以：

- 捕获网络数据包
- 查看源 IP、目标 IP、端口和协议
- 分析 TCP 握手、重传和连接关闭
- 查看 HTTP 明文请求与响应
- 查看 SSH/TLS 的握手和加密数据
- 保存 PCAP 文件用于故障排查与安全调查

简单记忆：

```text
Nmap：发现和扫描
Netcat：连接和传输
Wireshark：观察和分析
Wazuh：端点日志和安全告警
```

---

# 3. 网络协议分层

一个常见数据包可能包含：

```text
Frame
Ethernet II
IPv4
TCP 或 UDP
应用协议，例如 HTTP、SSH、TLS
```

| 层次 | 常见协议 |
|---|---|
| 应用层 | HTTP、HTTPS、SSH、DNS |
| 传输层 | TCP、UDP |
| 网络层 | IPv4、IPv6、ICMP |
| 数据链路层 | Ethernet、ARP |

---

# 4. Wireshark 与 tcpdump

| 功能 | Wireshark | tcpdump |
|---|---:|---:|
| 图形分析 | 强 | 无 |
| 服务器命令行抓包 | 不方便 | 强 |
| 保存 PCAP | 支持 | 支持 |
| 协议解码 | 很详细 | 基础 |
| 初学分析 | 很适合 | 需要命令经验 |

推荐流程：

```text
Ubuntu/Wazuh 用 tcpdump 抓包
→ 把 PCAP 复制到 Kali
→ 用 Wireshark 分析
```

---

# 5. UTM 虚拟网络的重要限制

Kali 通常可以看到：

```text
Kali 自己发送的流量
Kali 自己接收的流量
广播和部分组播
```

Kali 不一定能看到：

```text
Ubuntu ↔ Windows 的全部单播流量
Windows ↔ Wazuh 的全部单播流量
Ubuntu ↔ Wazuh 的全部单播流量
```

原因是虚拟交换机通常只把单播数据送到对应网卡。

因此：

```text
分析 Kali → Ubuntu：在 Kali 或 Ubuntu 抓包
分析 Windows → Wazuh：在 Windows 或 Wazuh 抓包
分析 Ubuntu → Wazuh：在 Ubuntu 或 Wazuh 抓包
```

---

# 6. 检查和启动 Wireshark

在 Kali：

```bash
which wireshark
wireshark --version
```

启动：

```bash
wireshark
```

如果没有安装：

```bash
sudo apt update
sudo apt install wireshark
```

检查用户组：

```bash
groups
```

如果没有 `wireshark`：

```bash
sudo usermod -aG wireshark "$USER"
```

然后注销并重新登录。

不建议：

```bash
sudo wireshark
```

---

# 7. 选择正确网卡

在 Kali：

```bash
ip -br address
```

你的 CyberLab 网卡通常是：

```text
eth0    UP    10.10.20.10/24
```

在 Wireshark 首页选择：

```text
eth0
```

不要误选：

```text
lo
未连接接口
其他不属于 CyberLab 的接口
```

---

# 8. Wireshark 主界面

Wireshark 有三个主要区域：

## Packet List

```text
No.
Time
Source
Destination
Protocol
Length
Info
```

## Packet Details

```text
Frame
Ethernet II
Internet Protocol
TCP/UDP
应用协议
```

## Packet Bytes

```text
十六进制
ASCII
原始数据
```

---

# 9. Capture Filter 与 Display Filter

## Capture Filter

在开始抓包前使用，决定哪些数据包会被保存。

示例：

```text
host 10.10.20.20
tcp port 22
net 10.10.20.0/24
```

## Display Filter

抓包后使用，只改变画面显示，不删除原始数据。

示例：

```text
ip.addr == 10.10.20.20
tcp.port == 22
http
```

| 项目 | Capture Filter | Display Filter |
|---|---|---|
| 使用时间 | 抓包前 | 抓包后 |
| 作用 | 决定抓什么 | 决定显示什么 |
| 语法 | BPF | Wireshark 语法 |
| 是否丢弃其他数据 | 是 | 否 |

记忆：

```text
Capture Filter：少抓
Display Filter：再筛
```

---

# 10. Capture Filter 速查

| 目的 | Filter |
|---|---|
| Ubuntu | `host 10.10.20.20` |
| Kali 与 Ubuntu | `host 10.10.20.10 and host 10.10.20.20` |
| 整个 CyberLab | `net 10.10.20.0/24` |
| TCP | `tcp` |
| UDP | `udp` |
| SSH | `tcp port 22` |
| Netcat 4444 | `tcp port 4444` |
| HTTP 8080 | `tcp port 8080` |
| ICMP | `icmp` |
| ARP | `arp` |

---

# 11. Display Filter 速查

| 目的 | Filter |
|---|---|
| Ubuntu 所有流量 | `ip.addr == 10.10.20.20` |
| Kali → Ubuntu | `ip.src == 10.10.20.10 && ip.dst == 10.10.20.20` |
| ARP | `arp` |
| ICMP | `icmp` |
| TCP | `tcp` |
| UDP | `udp` |
| SSH | `tcp.port == 22` |
| Netcat 4444 | `tcp.port == 4444` |
| HTTP | `http` |
| TLS | `tls` |
| SYN | `tcp.flags.syn == 1 && tcp.flags.ack == 0` |
| SYN-ACK | `tcp.flags.syn == 1 && tcp.flags.ack == 1` |
| RST | `tcp.flags.reset == 1` |
| FIN | `tcp.flags.fin == 1` |
| 重传 | `tcp.analysis.retransmission` |
| 单个 TCP 流 | `tcp.stream eq 0` |
| 大包 | `frame.len > 1000` |

---

# 12. 创建实验目录

在 Kali：

```bash
mkdir -p ~/CyberLab/Wireshark/{captures,exports,notes,reports}
```

推荐文件名：

```text
lab1-icmp-kali-ubuntu.pcapng
lab2-arp-kali-ubuntu.pcapng
lab3-netcat4444-kali-ubuntu.pcapng
lab4-nmap-syn-kali-ubuntu.pcapng
lab5-http8080-kali-ubuntu.pcapng
lab6-ssh-kali-ubuntu.pcapng
lab7-tls-kali-wazuh.pcapng
```

---

# 13. 实验一：ICMP Ping

## 开始抓包

在 Wireshark 双击：

```text
eth0
```

## 产生流量

在 Kali 第二个终端：

```bash
ping -c 4 10.10.20.20
```

## 停止抓包

点击红色停止按钮。

## 过滤

```text
icmp
```

你应该看到：

```text
Echo request
Echo reply
```

展开：

```text
Internet Protocol Version 4
Internet Control Message Protocol
```

请求：

```text
Source：10.10.20.10
Destination：10.10.20.20
Type：8 Echo request
```

回复：

```text
Source：10.10.20.20
Destination：10.10.20.10
Type：0 Echo reply
```

观察：

```text
Identifier
Sequence Number
TTL
Round-trip time
```

---

# 14. 实验二：ARP

ARP 用于把 IPv4 地址解析为 MAC 地址。

在 Kali：

```bash
sudo ip neigh flush all
```

开始抓包后执行：

```bash
ping -c 1 10.10.20.20
```

过滤：

```text
arp
```

常见结果：

```text
Who has 10.10.20.20? Tell 10.10.20.10
10.10.20.20 is at xx:xx:xx:xx:xx:xx
```

查看邻居表：

```bash
ip neigh
```

---

# 15. 实验三：TCP 三次握手与 Netcat 明文

## Ubuntu 监听

```bash
nc -lvn 4444
```

## Kali 开始抓包

选择 `eth0`。

## Kali 连接

```bash
nc -vn 10.10.20.20 4444
```

输入：

```text
WIRESHARK_NETCAT_TEST
```

Ubuntu 回复：

```text
REPLY_FROM_UBUNTU
```

然后按：

```text
Ctrl + C
```

## 过滤

```text
tcp.port == 4444
```

三次握手：

```text
Kali → Ubuntu：SYN
Ubuntu → Kali：SYN, ACK
Kali → Ubuntu：ACK
```

常用 TCP Flags：

| Flag | 含义 |
|---|---|
| SYN | 建立连接 |
| ACK | 确认 |
| PSH | 交付应用数据 |
| FIN | 正常关闭 |
| RST | 强制重置 |

---

# 16. Follow TCP Stream

选择一个 TCP 4444 数据包：

```text
右键
→ Follow
→ TCP Stream
```

你应该看到：

```text
WIRESHARK_NETCAT_TEST
REPLY_FROM_UBUNTU
```

这说明普通 Netcat 通信默认不加密。

可切换：

```text
ASCII
UTF-8
Hex Dump
Raw
```

---

# 17. TCP 连接关闭

过滤：

```text
tcp.flags.fin == 1
```

正常关闭可能看到：

```text
FIN, ACK
ACK
FIN, ACK
ACK
```

如果看到：

```text
RST
```

表示连接被重置。

---

# 18. 实验四：关闭端口

确保 Ubuntu 没有监听 TCP 65000。

Kali 开始抓包后：

```bash
nc -zvn -w 3 10.10.20.20 65000
```

过滤：

```text
tcp.port == 65000
```

常见结果：

```text
Kali → Ubuntu：SYN
Ubuntu → Kali：RST, ACK
```

解释：

```text
主机在线
端口关闭
```

开放、关闭、过滤的典型表现：

```text
Open：SYN → SYN-ACK
Closed：SYN → RST-ACK
Filtered：SYN 后无响应或收到 ICMP 不可达
```


---

# 19. 实验五：分析 Nmap SYN 扫描

Kali 开始抓包，然后执行：

```bash
sudo nmap -n -sS -p 20-30 10.10.20.20
```

过滤：

```text
ip.addr == 10.10.20.20 && tcp
```

只显示 Kali 发出的 SYN：

```text
ip.src == 10.10.20.10 &&
ip.dst == 10.10.20.20 &&
tcp.flags.syn == 1 &&
tcp.flags.ack == 0
```

开放端口通常表现为：

```text
SYN
SYN-ACK
RST
```

关闭端口通常表现为：

```text
SYN
RST-ACK
```

Nmap SYN Scan 不完成完整 TCP 连接。

端口扫描常见特征：

```text
同一个源 IP
短时间访问多个目标端口
大量 SYN
很多连接没有应用层数据
```

---

# 20. SYN Scan 与 Connect Scan 对比

## SYN Scan

```bash
sudo nmap -n -sS -p 22 10.10.20.20
```

流量：

```text
SYN
SYN-ACK
RST
```

## TCP Connect Scan

```bash
nmap -n -sT -p 22 10.10.20.20
```

流量：

```text
SYN
SYN-ACK
ACK
随后正常关闭或重置
```

分别抓包并比较：

```text
是否完成三次握手
目标日志是否不同
数据包数量是否不同
```

---

# 21. 实验六：HTTP 明文分析

## Ubuntu 启动临时 Web 服务

```bash
mkdir -p ~/CyberLab/Wireshark/web
```

```bash
echo "WIRESHARK_HTTP_TEST" \
  > ~/CyberLab/Wireshark/web/index.html
```

```bash
cd ~/CyberLab/Wireshark/web
python3 -m http.server 8080 --bind 10.10.20.20
```

## Kali 抓包并请求

```bash
curl http://10.10.20.20:8080/
```

过滤：

```text
http
```

只看请求：

```text
http.request
```

只看 GET：

```text
http.request.method == "GET"
```

只看响应：

```text
http.response
```

只看 HTTP 200：

```text
http.response.code == 200
```

你可能看到：

```text
GET / HTTP/1.1
Host: 10.10.20.20:8080
User-Agent: curl/...
```

响应可能包含：

```text
HTTP/1.0 200 OK
Server: SimpleHTTP/...
Content-Type: text/html
Content-Length: ...
WIRESHARK_HTTP_TEST
```

右键：

```text
Follow
→ HTTP Stream
```

或：

```text
Follow
→ TCP Stream
```

HTTP 默认不加密，因此 URL、Header 和正文可能直接可见。

停止 Ubuntu Web 服务：

```text
Ctrl + C
```

---

# 22. 实验七：SSH 加密流量

Kali 开始抓包，然后：

```bash
ssh labuser@10.10.20.20
```

登录后执行：

```bash
hostname
whoami
exit
```

过滤：

```text
tcp.port == 22
```

或：

```text
ssh
```

通常可以看到：

```text
源 IP 和目标 IP
TCP 22
TCP 三次握手
SSH Banner
数据包长度
连接时间
加密数据
```

通常看不到：

```text
登录密码
hostname 命令正文
whoami 输出
终端会话明文
```

原因：

```text
SSH 会加密应用数据
```

连接初期可能看到：

```text
SSH-2.0-OpenSSH_8.9p1 Ubuntu
```

Banner 之后的数据通常已加密。

---

# 23. 实验八：HTTPS / TLS

Kali 开始抓包，然后在浏览器访问：

```text
https://10.10.20.40
```

过滤：

```text
tls
```

或：

```text
tcp.port == 443
```

通常可以看到：

```text
Client Hello
Server Hello
Certificate
Cipher Suite
TLS Version
Encrypted Application Data
```

通常看不到：

```text
Wazuh 登录密码
Dashboard 页面正文
API 返回内容
Cookie 明文
```

常用过滤器：

```text
tls.handshake.type == 1
```

表示 Client Hello。

```text
tls.handshake.type == 2
```

表示 Server Hello。

```text
tls.handshake.type == 11
```

表示 Certificate。

展开：

```text
Transport Layer Security
```

查看：

```text
Version
Cipher Suites
Extensions
Supported Groups
Certificate
```

---

# 24. HTTP 与 HTTPS 对比

| 项目 | HTTP | HTTPS |
|---|---|---|
| 常见端口 | 80/8080 | 443 |
| 是否加密 | 否 | 是 |
| URL/正文可见 | 通常可见 | 通常不可见 |
| Wireshark 可见内容 | 请求和响应 | TLS 握手和密文 |
| 适合敏感数据 | 否 | 是 |

---

# 25. 实验九：UDP

Ubuntu 监听：

```bash
nc -u -lvn 5555
```

Kali 开始抓包后发送：

```bash
printf 'WIRESHARK_UDP_TEST\n' |
nc -u -vn -w 2 10.10.20.20 5555
```

过滤：

```text
udp.port == 5555
```

点击数据包，展开：

```text
User Datagram Protocol
Data
```

底部可能直接看到：

```text
WIRESHARK_UDP_TEST
```

TCP 与 UDP：

| TCP | UDP |
|---|---|
| 有握手 | 无握手 |
| 有确认 | 无确认 |
| 有重传机制 | 协议本身不保证重传 |
| 面向连接 | 无连接 |
| 可靠但开销较高 | 快但不保证送达 |

---

# 26. Wazuh Agent 通信抓包

Ubuntu Agent 通常连接：

```text
Ubuntu 10.10.20.20
→ Wazuh 10.10.20.40
→ TCP 1514/1515
```

Kali 不一定能看到这段单播流量。

在 Ubuntu 抓包：

```bash
sudo tcpdump \
  -i enp0s1 \
  -nn \
  host 10.10.20.40 \
  and '(tcp port 1514 or tcp port 1515)' \
  -w ~/wazuh-agent-traffic.pcap
```

等待约一分钟后按：

```text
Ctrl + C
```

从 Kali 复制：

```bash
scp labuser@10.10.20.20:~/wazuh-agent-traffic.pcap \
  ~/CyberLab/Wireshark/captures/
```

在 Wireshark 打开。

你通常能看到：

```text
源/目标 IP
TCP 1514 或 1515
连接时间
数据包大小
TCP 状态
加密或封装后的应用数据
```

---

# 27. Windows Agent 流量抓包

Windows Agent 通常连接：

```text
Windows 10.10.20.30
→ Wazuh 10.10.20.40
→ TCP 1514
```

在 Wazuh-SIEM 抓包：

```bash
sudo tcpdump \
  -i enp0s1 \
  -nn \
  host 10.10.20.30 \
  and tcp port 1514 \
  -w ~/windows-wazuh-agent.pcap
```

等待一段时间后按：

```text
Ctrl + C
```

从 Kali 复制：

```bash
scp labuser@10.10.20.40:~/windows-wazuh-agent.pcap \
  ~/CyberLab/Wireshark/captures/
```

---

# 28. 保存和打开抓包

保存：

```text
File
→ Save As
```

推荐：

```text
PCAPNG
```

PCAP 与 PCAPNG：

| 格式 | 特点 |
|---|---|
| PCAP | 传统、兼容性好 |
| PCAPNG | 可保存更多元数据和注释 |

打开：

```text
File
→ Open
```

或：

```bash
wireshark ~/CyberLab/Wireshark/captures/lab3-netcat4444.pcapng
```

---

# 29. 导出指定数据包

先应用过滤器：

```text
tcp.port == 4444
```

然后：

```text
File
→ Export Specified Packets
→ Displayed
```

保存为：

```text
netcat-only.pcapng
```

---

# 30. 标记和注释数据包

标记：

```text
Edit
→ Mark/Unmark Packet
```

常见快捷键：

```text
Ctrl + M
```

添加注释：

```text
右键数据包
→ Packet Comment
```

示例：

```text
First SYN from Kali to Ubuntu TCP 4444
```

PCAPNG 更适合保存注释。

---

# 31. Follow Stream

TCP：

```text
右键
→ Follow
→ TCP Stream
```

UDP：

```text
右键
→ Follow
→ UDP Stream
```

只显示某个 TCP 会话：

```text
tcp.stream eq 0
```

只显示某个 UDP 会话：

```text
udp.stream eq 0
```

---

# 32. Conversation Filter

右键一个数据包：

```text
Conversation Filter
→ TCP
```

Wireshark 会自动生成类似：

```text
ip.addr == 10.10.20.10 &&
ip.addr == 10.10.20.20 &&
tcp.port == 4444
```

也可以：

```text
Apply as Filter
Prepare as Filter
```

---

# 33. 逻辑运算符

| 运算符 | 含义 |
|---|---|
| `&&` | AND |
| `||` | OR |
| `!` | NOT |
| `==` | 等于 |
| `!=` | 不等于 |
| `contains` | 包含 |
| `matches` | 正则匹配 |

示例：

```text
ip.addr == 10.10.20.20 &&
tcp.port == 22
```

```text
tcp.port == 22 ||
tcp.port == 443
```

```text
tcp contains "WIRESHARK"
```

注意：加密协议正文不能通过 `contains` 直接搜索。

---

# 34. Statistics：Protocol Hierarchy

打开：

```text
Statistics
→ Protocol Hierarchy
```

可以看到：

```text
Ethernet
IPv4
TCP
UDP
HTTP
TLS
SSH
ICMP
ARP
```

以及：

```text
Packets
Bytes
Percentage
```

适合快速了解 PCAP 主要协议。

---

# 35. Statistics：Endpoints

打开：

```text
Statistics
→ Endpoints
```

IPv4 标签页可能显示：

```text
10.10.20.10
10.10.20.20
10.10.20.30
10.10.20.40
```

并统计：

```text
Packets
Bytes
Tx Packets
Rx Packets
```

Endpoints 表示：

```text
单个通信端点
```

---

# 36. Statistics：Conversations

打开：

```text
Statistics
→ Conversations
```

显示：

```text
Address A
Address B
Port A
Port B
Packets
Bytes
Duration
```

Conversations 表示：

```text
两个端点之间的会话
```

---

# 37. Statistics：I/O Graphs

打开：

```text
Statistics
→ I/O Graphs
```

可以查看：

```text
每秒数据包
每秒字节
流量峰值
扫描期间的突发流量
```

添加过滤器：

```text
tcp.flags.syn == 1 &&
tcp.flags.ack == 0
```

观察 Nmap SYN 扫描峰值。

---

# 38. Expert Information

打开：

```text
Analyze
→ Expert Information
```

常见项目：

```text
TCP Retransmission
Duplicate ACK
Previous segment not captured
RST
Zero Window
```

注意：

```text
Expert Information 不等于漏洞报告
```

它是协议分析提示。

---

# 39. TCP 分析过滤器

重传：

```text
tcp.analysis.retransmission
```

重复 ACK：

```text
tcp.analysis.duplicate_ack
```

乱序：

```text
tcp.analysis.out_of_order
```

Zero Window：

```text
tcp.analysis.zero_window
```

RST：

```text
tcp.flags.reset == 1
```

少量重传不一定表示攻击，可能是：

- 丢包
- 延迟
- 虚拟机负载
- 抓包缺失

---

# 40. 时间显示

打开：

```text
View
→ Time Display Format
```

常用格式：

```text
Seconds Since Beginning of Capture
Date and Time of Day
Seconds Since Previous Displayed Packet
```

分析延迟时可使用：

```text
Seconds Since Previous Displayed Packet
```


---

# 41. Wireshark 与 Wazuh 如何配合

| Wireshark | Wazuh |
|---|---|
| 分析网络数据包 | 分析端点日志与告警 |
| 查看 IP、端口、Flags | 查看用户、进程、文件和事件 |
| 保存 PCAP | 保存安全事件 |
| 观察协议内容 | 使用规则检测异常 |
| 适合网络取证 | 适合 SIEM 和端点监控 |

示例：

```text
Kali 扫描 Ubuntu
```

Wireshark 可以证明：

```text
源 IP：10.10.20.10
目标 IP：10.10.20.20
短时间访问多个端口
大量 SYN
```

Ubuntu/Wazuh 可能提供：

```text
SSH 日志
防火墙日志
应用日志
Wazuh 告警
事件时间
规则 ID
```

完整调查：

```text
PCAP 网络证据
+
目标主机日志
+
Wazuh 告警
```

---

# 42. 时间同步

关联 PCAP 与 Wazuh 时，系统时间应尽量一致。

Kali：

```bash
timedatectl
```

Ubuntu：

```bash
timedatectl
```

Wazuh：

```bash
timedatectl
```

Windows：

```powershell
Get-Date
```

如果时间不同，会难以把：

```text
Wireshark 数据包时间
```

和：

```text
Wazuh 事件时间
```

对应起来。

---

# 43. 完整实验一：ARP、ICMP 与 Netcat

## 实验信息

```text
Capture Host：Kali 10.10.20.10
Target：Ubuntu 10.10.20.20
Interface：eth0
```

## 步骤一：清除 ARP 缓存

```bash
sudo ip neigh flush all
```

## 步骤二：开始抓包

Wireshark 选择：

```text
eth0
```

## 步骤三：Ping

```bash
ping -c 3 10.10.20.20
```

## 步骤四：Ubuntu 监听

```bash
nc -lvn 4444
```

## 步骤五：Kali 连接

```bash
nc -vn 10.10.20.20 4444
```

发送：

```text
WIRESHARK_LAB_TEST
```

## 步骤六：分析

过滤：

```text
arp
```

```text
icmp
```

```text
tcp.port == 4444
```

找到：

```text
ARP Request
ARP Reply
ICMP Request
ICMP Reply
SYN
SYN-ACK
ACK
Netcat 明文
FIN 或 RST
```

保存：

```text
lab1-arp-icmp-netcat.pcapng
```

---

# 44. 完整实验二：Nmap 扫描分析

开始抓包后：

```bash
sudo nmap -n -sS -p 20-30 10.10.20.20
```

过滤：

```text
ip.src == 10.10.20.10 &&
ip.dst == 10.10.20.20 &&
tcp.flags.syn == 1 &&
tcp.flags.ack == 0
```

记录：

```text
扫描了多少端口？
哪个端口返回 SYN-ACK？
哪些端口返回 RST-ACK？
是否完成完整三次握手？
扫描持续多久？
```

保存：

```text
lab2-nmap-syn-scan.pcapng
```

---

# 45. 完整实验三：HTTP 与 SSH 对比

## HTTP

Ubuntu：

```bash
cd ~/CyberLab/Wireshark/web
python3 -m http.server 8080 --bind 10.10.20.20
```

Kali：

```bash
curl http://10.10.20.20:8080/
```

过滤：

```text
http
```

## SSH

Kali：

```bash
ssh labuser@10.10.20.20
```

过滤：

```text
tcp.port == 22
```

比较：

| 项目 | HTTP | SSH |
|---|---|---|
| 端口 | 8080 | 22 |
| 是否加密 | 否 | 是 |
| 正文是否可见 | 是 | 通常否 |
| 可分析内容 | 请求、响应、正文 | 握手、Banner、长度、时间 |

保存：

```text
lab3-http-vs-ssh.pcapng
```

---

# 46. 完整实验四：Wazuh TLS

开始抓包后访问：

```text
https://10.10.20.40
```

过滤：

```text
tls
```

记录：

```text
Client Hello 数据包号
Server Hello 数据包号
TLS 版本
Cipher Suite
Certificate
Encrypted Application Data
```

保存：

```text
lab4-wazuh-tls.pcapng
```

---

# 47. 完整实验五：网络与 Wazuh 关联

## 步骤一：记录开始时间

在 Kali：

```bash
date
```

## 步骤二：开始 Wireshark 抓包

## 步骤三：扫描 Ubuntu

```bash
sudo nmap -n -sT -p 22 10.10.20.20
```

## 步骤四：制造一次错误 SSH 登录

```bash
ssh wrong-user@10.10.20.20
```

输入一次错误密码，然后：

```text
Ctrl + C
```

## 步骤五：停止抓包

过滤：

```text
tcp.port == 22
```

## 步骤六：Wazuh 查询

在 Dashboard：

```text
Threat Hunting
```

时间范围：

```text
Last 15 minutes
```

筛选：

```text
agent.name: ubuntu-target
```

搜索：

```text
wrong-user
```

对比：

```text
Wireshark 中的源 IP
Wazuh 中的来源 IP
Wireshark 时间
Wazuh 事件时间
```

---

# 48. 实验报告模板

## Lab Information

```text
Lab Name：Wireshark Traffic Analysis
Date：
Student：
Capture Host：
Interface：
Target：
Network：CyberLab 10.10.20.0/24
Authorization：Personal CyberLab
```

## Objective

```text
捕获并分析 ARP、ICMP、TCP、Netcat、HTTP、SSH 和 TLS 流量。
```

## Capture Configuration

```text
Capture Filter：
Start Time：
End Time：
PCAP Filename：
```

## Display Filters

```text
arp
icmp
tcp.port == 4444
http
tcp.port == 22
tls
```

## Findings

| Protocol | Source | Destination | Observation |
|---|---|---|---|
| ARP |  |  |  |
| ICMP |  |  |  |
| TCP |  |  |  |
| HTTP |  |  |  |
| SSH |  |  |  |
| TLS |  |  |  |

## Important Packets

| Packet No. | Purpose | Notes |
|---:|---|---|
|  | SYN |  |
|  | SYN-ACK |  |
|  | ACK |  |
|  | HTTP GET |  |
|  | HTTP 200 |  |
|  | TLS Client Hello |  |

## Evidence

```text
PCAP：
Screenshot：
Follow TCP Stream：
Wazuh Event：
Target Log：
```

## Analysis

```text
TCP 三次握手是哪三个数据包？
Netcat 文本是否可见？
HTTP 正文是否可见？
SSH 正文是否可见？
TLS 中可以看到什么？
是否存在 RST 或重传？
```

## Lessons Learned

```text
Capture Filter 与 Display Filter 的区别：
Wireshark 与 Wazuh 的区别：
虚拟网络抓包的限制：
```

---

# 49. 常见错误与解决方法

## 没有任何数据包

确认：

```bash
ip -br address
```

选择：

```text
eth0
```

产生流量：

```bash
ping -c 3 10.10.20.20
```

## Permission denied

```bash
groups
sudo usermod -aG wireshark "$USER"
```

重新登录。

## Display Filter 变红

正确：

```text
ip.addr == 10.10.20.20
```

错误：

```text
ip.addr = 10.10.20.20
```

Wireshark 使用：

```text
==
```

## 看不到 HTTP

先用：

```text
tcp.port == 8080
```

必要时：

```text
右键
→ Decode As
→ HTTP
```

## 看不到 SSH 密码

这是正常的，因为 SSH 加密。

## 看不到 Windows ↔ Wazuh 流量

在 Wazuh 或 Windows 抓包，不要依赖 Kali 看到所有单播流量。

## PCAP 太大

- 使用 Capture Filter
- 限制抓包时间
- 只抓一个主机或端口
- 实验结束立即停止
- 使用 Ring Buffer（后续 tcpdump 课程学习）

## 显示 Checksum Error

虚拟网卡可能使用 Checksum Offloading。

抓包发生在校验和最终计算之前，因此不一定是真实错误。

---

# 50. 抓包安全与隐私

PCAP 可能包含：

- IP 和 MAC
- HTTP 内容
- Cookie
- 用户名
- 内部服务信息
- 文件内容
- 未加密凭据
- 系统主机名

因此：

```text
不要公开上传真实网络 PCAP
不要抓取他人通信
不要长期保存不必要的敏感数据
不要把 PCAP 放进公开 GitHub 仓库
```

本实验只使用无敏感信息的测试内容。

---

# 51. 本课练习

1. 捕获 Kali Ping Ubuntu，找到 ARP 和 ICMP。
2. 捕获 Netcat TCP 4444，找到三次握手。
3. 使用 Follow TCP Stream 查看测试文字。
4. 捕获关闭端口 65000，找到 RST-ACK。
5. 捕获 Nmap SYN Scan，统计 SYN 数量。
6. 捕获 HTTP 8080，找到 GET 和 200 OK。
7. 捕获 SSH，说明可见与不可见内容。
8. 捕获 Wazuh HTTPS，找到 Client Hello 和 Server Hello。
9. 使用 Protocol Hierarchy 记录主要协议。
10. 使用 Endpoints 找到最活跃 IP。
11. 使用 Conversations 找到 Netcat 会话。
12. 导出仅包含 TCP 4444 的新 PCAP。
13. 使用 Wazuh 关联一次错误 SSH 登录。
14. 写一份完整实验报告。

---

# 52. 复习题

1. Wireshark 的主要用途是什么？
2. Capture Filter 与 Display Filter 有何区别？
3. `host 10.10.20.20` 属于哪一种？
4. `ip.addr == 10.10.20.20` 属于哪一种？
5. TCP 三次握手是什么？
6. RST 表示什么？
7. ARP 的用途是什么？
8. ICMP Echo Request 的 Type 是多少？
9. ICMP Echo Reply 的 Type 是多少？
10. 为什么 Netcat 文本可见？
11. 为什么 SSH 密码不可见？
12. 为什么 HTTPS 正文不可见？
13. SYN Scan 对开放端口通常如何处理？
14. 关闭端口通常返回什么？
15. `tcp.stream eq 0` 的用途是什么？
16. Endpoints 与 Conversations 有何区别？
17. Expert Information 是否等于漏洞？
18. 为什么 Kali 看不到全部虚拟机单播流量？
19. PCAP 与 PCAPNG 有什么区别？
20. Wireshark 与 Wazuh 如何互补？

---

# 53. 参考答案

1. 捕获和分析网络数据包。
2. Capture Filter 决定抓什么；Display Filter 决定显示什么。
3. Capture Filter。
4. Display Filter。
5. SYN、SYN-ACK、ACK。
6. 强制重置连接。
7. 把本地 IPv4 地址解析为 MAC。
8. Type 8。
9. Type 0。
10. 普通 Netcat 默认不加密。
11. SSH 加密应用数据。
12. HTTPS 使用 TLS。
13. 收到 SYN-ACK 后发送 RST。
14. RST-ACK。
15. 只显示某个 TCP 会话。
16. Endpoints 是单个端点；Conversations 是端点之间的会话。
17. 不是，只是协议分析提示。
18. 虚拟交换机不会把所有单播发送给 Kali。
19. PCAPNG 能保存更多元数据和注释。
20. Wireshark 提供网络证据，Wazuh 提供端点日志和告警。

---

# 54. 核心记忆口诀

```text
先选对网卡
再开始抓包

Capture Filter：少抓
Display Filter：再筛

ARP 找 MAC
ICMP 测连通
TCP 看握手
UDP 无握手
HTTP 看明文
SSH 看加密
TLS 看握手和证书

Wireshark 看网络
Wazuh 看端点和告警
```

---

# 55. 完成标准

完成本课后，你应该拥有：

```text
ARP 抓包
ICMP 抓包
Netcat TCP 4444 抓包
Nmap SYN Scan 抓包
HTTP 8080 抓包
SSH 抓包
Wazuh TLS 抓包
一个 Follow TCP Stream 截图
一个 Protocol Hierarchy 截图
一个 Endpoints 截图
一个 Conversations 截图
一份实验报告
```

检查：

```bash
find ~/CyberLab/Wireshark -type f
```

---

# 56. 下一课

下一课建议：

```text
tcpdump
```

学习内容：

- 在 Ubuntu 和 Wazuh 命令行抓包
- 使用 BPF 过滤器
- 保存 PCAP
- 限制抓包数量和大小
- Ring Buffer
- 远程复制 PCAP
- 使用 Wireshark 分析 tcpdump 文件

前三课关系：

```text
Nmap：发现和扫描
Netcat：连接和传输
Wireshark：观察和分析
```

---

# 57. 安全边界

只允许分析：

```text
Kali-CyberLab      10.10.20.10
Ubuntu-Target      10.10.20.20
Windows-Target     10.10.20.30
Wazuh-SIEM         10.10.20.40
```

不要抓取：

- 学校或公司网络
- 公共 Wi‑Fi
- 邻居通信
- 家庭其他成员流量
- 未经授权的云服务器
- 随机公网目标

每次实验：

```text
确认授权范围
缩短抓包时间
避免真实敏感数据
实验结束后停止抓包
妥善保存或删除 PCAP
```
