# 第四课：tcpdump 完整学习指南

> **实验环境：你自己的 CyberLab**
>
> 本课程只用于你自己创建并拥有的虚拟机网络。  
> 不要在学校、公司、公共 Wi‑Fi、邻居网络或未经授权的系统上抓包。

---

# 1. 本课目标

学完本课后，你应该能够：

```text
理解 tcpdump 的用途
选择正确的网络接口
实时查看网络流量
使用 BPF Capture Filter
捕获 ARP、ICMP、TCP、UDP、HTTP、SSH 和 Wazuh 流量
识别 TCP SYN、SYN-ACK、FIN 和 RST
保存 PCAP 文件
限制抓包数量、时间和文件大小
使用 Ring Buffer 循环保存
读取已有 PCAP
把 Ubuntu/Wazuh 上的 PCAP 复制到 Kali
使用 Wireshark 分析 tcpdump 抓到的文件
结合 Wazuh 进行网络与端点日志关联分析
排查常见抓包问题
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

# 2. 什么是 tcpdump？

`tcpdump` 是一个命令行网络抓包工具。

它可以：

- 捕获网络数据包
- 实时查看流量
- 按 IP、端口和协议过滤
- 保存 PCAP 文件
- 读取已有 PCAP
- 在没有图形桌面的服务器上抓包
- 帮助排查连接、防火墙和服务问题
- 为 Wireshark 提供原始抓包文件

简单记忆：

```text
Wireshark：图形化详细分析
tcpdump：命令行快速抓包
```

---

# 3. 为什么要学习 tcpdump？

Ubuntu Server 和 Wazuh-SIEM 没有图形桌面。

当你需要分析：

```text
Ubuntu → Wazuh
Windows → Wazuh
Kali → Ubuntu
```

可以直接在相关服务器上使用 `tcpdump`。

典型流程：

```text
在 Ubuntu/Wazuh 上 tcpdump 抓包
→ 保存为 PCAP
→ 使用 SCP 复制到 Kali
→ 用 Wireshark 深入分析
```

---

# 4. 检查 tcpdump

在 Kali、Ubuntu 或 Wazuh：

```bash
which tcpdump
```

查看版本：

```bash
tcpdump --version
```

查看帮助：

```bash
tcpdump --help
```

如果没有安装：

```bash
sudo apt update
sudo apt install tcpdump
```

---

# 5. 为什么通常需要 sudo？

抓取网络接口上的原始数据包通常需要较高权限。

所以常用：

```bash
sudo tcpdump
```

如果不使用 `sudo`，可能看到：

```text
You don't have permission to perform this capture
```

---

# 6. tcpdump 基本语法

```bash
sudo tcpdump [选项] [过滤器]
```

示例：

```bash
sudo tcpdump -i eth0
```

含义：

```text
在 eth0 上实时捕获数据包
```

只捕获 Ubuntu 流量：

```bash
sudo tcpdump -i eth0 host 10.10.20.20
```

只捕获 TCP 22：

```bash
sudo tcpdump -i eth0 tcp port 22
```

---

# 7. 识别正确网卡

## Kali

```bash
ip -br address
```

通常：

```text
eth0    UP    10.10.20.10/24
```

## Ubuntu

```bash
ip -br address
```

通常：

```text
enp0s1    UP    10.10.20.20/24
```

## Wazuh

```bash
ip -br address
```

通常：

```text
enp0s1    UP    10.10.20.40/24
enp0s2    UP    192.168.64.x/24
```

其中：

```text
enp0s1 = CyberLab
enp0s2 = Shared Network / Internet
```

抓 CyberLab 流量时选择：

```text
enp0s1
```

---

# 8. 列出所有抓包接口

```bash
sudo tcpdump -D
```

可能看到：

```text
1.eth0
2.any
3.lo
```

或者：

```text
1.enp0s1
2.enp0s2
3.any
4.lo
```

---

# 9. 第一次抓包

在 Kali：

```bash
sudo tcpdump -i eth0
```

然后在另一个终端：

```bash
ping -c 3 10.10.20.20
```

停止抓包：

```text
Ctrl + C
```

tcpdump 最后会显示：

```text
packets captured
packets received by filter
packets dropped by kernel
```

---

# 10. 常用选项

| 参数 | 含义 |
|---|---|
| `-i eth0` | 指定接口 |
| `-D` | 列出接口 |
| `-n` | 不解析主机名 |
| `-nn` | 不解析主机名和端口名 |
| `-v` | 更多详细信息 |
| `-vv` | 更详细 |
| `-vvv` | 最详细 |
| `-c 10` | 捕获 10 个包后停止 |
| `-w file.pcap` | 保存 PCAP |
| `-r file.pcap` | 读取 PCAP |
| `-A` | 以 ASCII 显示数据 |
| `-X` | 十六进制和 ASCII |
| `-XX` | 包含链路层头部的 Hex + ASCII |
| `-s 0` | 捕获完整数据包 |
| `-q` | 精简输出 |
| `-tttt` | 显示完整日期时间 |
| `-e` | 显示 Ethernet Header |
| `-l` | 行缓冲，便于管道输出 |

推荐基础组合：

```bash
sudo tcpdump -i eth0 -nn
```

---

# 11. 为什么常用 -nn？

默认情况下，tcpdump 可能把：

```text
10.10.20.20
```

解析成主机名，把：

```text
22
```

显示成：

```text
ssh
```

使用：

```bash
-nn
```

可以避免 DNS 和服务名解析。

优点：

- 输出更快
- 结果更稳定
- 端口号更清楚
- CyberLab 没有 DNS，不会等待解析

---

# 12. BPF Capture Filter

tcpdump 使用：

```text
Berkeley Packet Filter
BPF
```

常见过滤对象：

```text
host
src
dst
net
port
portrange
tcp
udp
icmp
arp
```

逻辑运算：

```text
and
or
not
```

也可以使用：

```text
&&
||
!
```

为了可读性，初学建议使用：

```text
and
or
not
```

---

# 13. 按主机过滤

捕获与 Ubuntu 有关的流量：

```bash
sudo tcpdump -i eth0 -nn host 10.10.20.20
```

只捕获来自 Kali：

```bash
sudo tcpdump -i eth0 -nn src host 10.10.20.10
```

只捕获发送到 Ubuntu：

```bash
sudo tcpdump -i eth0 -nn dst host 10.10.20.20
```

Kali 与 Ubuntu 之间：

```bash
sudo tcpdump -i eth0 -nn \
  host 10.10.20.10 and host 10.10.20.20
```

---

# 14. 按网络过滤

整个 CyberLab：

```bash
sudo tcpdump -i eth0 -nn net 10.10.20.0/24
```

来自 CyberLab：

```bash
sudo tcpdump -i eth0 -nn src net 10.10.20.0/24
```

发送到 CyberLab：

```bash
sudo tcpdump -i eth0 -nn dst net 10.10.20.0/24
```

---

# 15. 按协议过滤

TCP：

```bash
sudo tcpdump -i eth0 -nn tcp
```

UDP：

```bash
sudo tcpdump -i eth0 -nn udp
```

ICMP：

```bash
sudo tcpdump -i eth0 -nn icmp
```

ARP：

```bash
sudo tcpdump -i eth0 -nn arp
```

IPv6：

```bash
sudo tcpdump -i eth0 -nn ip6
```

---

# 16. 按端口过滤

TCP 22：

```bash
sudo tcpdump -i eth0 -nn tcp port 22
```

TCP 4444：

```bash
sudo tcpdump -i eth0 -nn tcp port 4444
```

源端口 22：

```bash
sudo tcpdump -i eth0 -nn tcp src port 22
```

目标端口 22：

```bash
sudo tcpdump -i eth0 -nn tcp dst port 22
```

端口范围：

```bash
sudo tcpdump -i eth0 -nn portrange 20-30
```

---

# 17. 多条件过滤

Kali 到 Ubuntu 的 SSH：

```bash
sudo tcpdump -i eth0 -nn \
  src host 10.10.20.10 and \
  dst host 10.10.20.20 and \
  tcp dst port 22
```

Ubuntu 与 Wazuh 的 Agent 流量：

```bash
sudo tcpdump -i enp0s1 -nn \
  host 10.10.20.20 and \
  host 10.10.20.40 and \
  tcp port 1514
```

Wazuh 1514 或 1515：

```bash
sudo tcpdump -i enp0s1 -nn \
  host 10.10.20.40 and \
  '(tcp port 1514 or tcp port 1515)'
```

注意：

```text
包含 or 的复杂过滤器建议使用单引号或括号
```

例如：

```bash
sudo tcpdump -i enp0s1 -nn \
  'tcp port 1514 or tcp port 1515'
```

---

# 18. 创建实验目录

在 Kali：

```bash
mkdir -p ~/CyberLab/tcpdump/{captures,notes,reports}
```

在 Ubuntu：

```bash
mkdir -p ~/CyberLab/tcpdump/captures
```

在 Wazuh：

```bash
mkdir -p ~/CyberLab/tcpdump/captures
```

---

# 19. 实验一：捕获 ICMP Ping

在 Kali：

```bash
sudo tcpdump -i eth0 -nn icmp
```

另一个 Kali 终端：

```bash
ping -c 4 10.10.20.20
```

你可能看到：

```text
10.10.20.10 > 10.10.20.20: ICMP echo request
10.10.20.20 > 10.10.20.10: ICMP echo reply
```

停止：

```text
Ctrl + C
```

---

# 20. 限制捕获数量

只捕获 8 个 ICMP 包：

```bash
sudo tcpdump -i eth0 -nn -c 8 icmp
```

然后执行：

```bash
ping -c 4 10.10.20.20
```

4 次 Ping 通常产生：

```text
4 个 request
4 个 reply
共 8 个数据包
```

tcpdump 会自动停止。

---

# 21. 实验二：捕获 ARP

清除 Kali 邻居缓存：

```bash
sudo ip neigh flush all
```

开始抓 ARP：

```bash
sudo tcpdump -i eth0 -nn -e arp
```

另一个终端：

```bash
ping -c 1 10.10.20.20
```

你可能看到：

```text
ARP, Request who-has 10.10.20.20 tell 10.10.20.10
ARP, Reply 10.10.20.20 is-at xx:xx:xx:xx:xx:xx
```

`-e` 会显示 Ethernet/MAC 信息。

---

# 22. 实验三：Netcat TCP 4444

Ubuntu：

```bash
nc -lvn 4444
```

Kali 抓包：

```bash
sudo tcpdump -i eth0 -nn -A tcp port 4444
```

Kali 第二个终端：

```bash
nc -vn 10.10.20.20 4444
```

输入：

```text
TCPDUMP_NETCAT_TEST
```

Ubuntu 回复：

```text
REPLY_FROM_UBUNTU
```

因为使用了：

```text
-A
```

tcpdump 可能直接显示明文。

停止连接和抓包：

```text
Ctrl + C
```

---

# 23. -A、-X 和 -XX

ASCII：

```bash
sudo tcpdump -i eth0 -nn -A tcp port 4444
```

Hex + ASCII：

```bash
sudo tcpdump -i eth0 -nn -X tcp port 4444
```

包含 Ethernet Header：

```bash
sudo tcpdump -i eth0 -nn -XX tcp port 4444
```

说明：

```text
-A 适合查明文
-X 适合同时看十六进制和 ASCII
-XX 还包含链路层头部
```

---

# 24. 实验四：TCP 三次握手

Ubuntu 保持监听：

```bash
nc -lvn 4444
```

Kali：

```bash
sudo tcpdump -i eth0 -nn \
  'tcp port 4444'
```

然后连接：

```bash
nc -vn 10.10.20.20 4444
```

你会看到类似：

```text
Flags [S]
Flags [S.]
Flags [.]
```

解释：

| 输出 | 含义 |
|---|---|
| `[S]` | SYN |
| `[S.]` | SYN-ACK |
| `[.]` | ACK |
| `[P.]` | PSH-ACK |
| `[F.]` | FIN-ACK |
| `[R]` / `[R.]` | RST |

---

# 25. 只捕获 SYN

只捕获所有带 SYN 的 TCP 包：

```bash
sudo tcpdump -i eth0 -nn \
  'tcp[tcpflags] & tcp-syn != 0'
```

只捕获初始 SYN，不包括 SYN-ACK：

```bash
sudo tcpdump -i eth0 -nn \
  'tcp[tcpflags] & (tcp-syn|tcp-ack) == tcp-syn'
```

只捕获 SYN-ACK：

```bash
sudo tcpdump -i eth0 -nn \
  'tcp[tcpflags] & (tcp-syn|tcp-ack) == (tcp-syn|tcp-ack)'
```

---

# 26. 只捕获 RST 或 FIN

RST：

```bash
sudo tcpdump -i eth0 -nn \
  'tcp[tcpflags] & tcp-rst != 0'
```

FIN：

```bash
sudo tcpdump -i eth0 -nn \
  'tcp[tcpflags] & tcp-fin != 0'
```

---

# 27. 实验五：关闭端口

确保 Ubuntu 没有监听 TCP 65000。

Kali 抓包：

```bash
sudo tcpdump -i eth0 -nn tcp port 65000
```

另一个终端：

```bash
nc -zvn -w 3 10.10.20.20 65000
```

典型流量：

```text
Kali → SYN
Ubuntu → RST-ACK
```

表示：

```text
主机在线
端口关闭
```

---

# 28. 实验六：Nmap SYN Scan

Kali 开始抓包：

```bash
sudo tcpdump -i eth0 -nn \
  'src host 10.10.20.10 and dst host 10.10.20.20 and tcp'
```

另一个终端：

```bash
sudo nmap -n -sS -p 20-30 10.10.20.20
```

观察：

```text
短时间多个目标端口
大量 SYN
开放端口返回 SYN-ACK
关闭端口返回 RST-ACK
Nmap 对开放端口发送 RST
```

---

# 29. 实验七：HTTP 明文

Ubuntu：

```bash
mkdir -p ~/CyberLab/tcpdump/web
```

```bash
echo "TCPDUMP_HTTP_TEST" \
  > ~/CyberLab/tcpdump/web/index.html
```

```bash
cd ~/CyberLab/tcpdump/web
python3 -m http.server 8080 --bind 10.10.20.20
```

Kali 抓包：

```bash
sudo tcpdump -i eth0 -nn -A tcp port 8080
```

另一个终端：

```bash
curl http://10.10.20.20:8080/
```

你可能看到：

```text
GET / HTTP/1.1
Host: 10.10.20.20:8080
User-Agent: curl/...
HTTP/1.0 200 OK
TCPDUMP_HTTP_TEST
```

停止 Python Server：

```text
Ctrl + C
```

---

# 30. 实验八：SSH 加密流量

Kali 抓包：

```bash
sudo tcpdump -i eth0 -nn -A tcp port 22
```

另一个终端：

```bash
ssh labuser@10.10.20.20
```

登录后：

```bash
hostname
whoami
exit
```

通常能看到：

```text
IP
端口
TCP Flags
SSH Banner
数据包大小
加密后的数据
```

通常看不到：

```text
密码
命令正文
命令输出明文
```

因为 SSH 加密应用数据。

---

# 31. 实验九：UDP

Ubuntu：

```bash
nc -u -lvn 5555
```

Kali 抓包：

```bash
sudo tcpdump -i eth0 -nn -A udp port 5555
```

另一个终端：

```bash
printf 'TCPDUMP_UDP_TEST\n' |
nc -u -vn -w 2 10.10.20.20 5555
```

tcpdump 可能直接显示：

```text
TCPDUMP_UDP_TEST
```

UDP 没有 TCP 三次握手。

---

# 32. 保存 PCAP

在 Kali：

```bash
sudo tcpdump -i eth0 -nn -s 0 \
  -w ~/CyberLab/tcpdump/captures/kali-ubuntu.pcap \
  host 10.10.20.20
```

说明：

```text
-w = 写入文件
-s 0 = 捕获完整数据包
```

注意：

使用 `-w` 时，终端不会逐包显示详细内容。

停止：

```text
Ctrl + C
```

检查：

```bash
ls -lh ~/CyberLab/tcpdump/captures/kali-ubuntu.pcap
```

---

# 33. 读取 PCAP

```bash
tcpdump -nn \
  -r ~/CyberLab/tcpdump/captures/kali-ubuntu.pcap
```

只读取 ICMP：

```bash
tcpdump -nn \
  -r ~/CyberLab/tcpdump/captures/kali-ubuntu.pcap \
  icmp
```

只读取 TCP 22：

```bash
tcpdump -nn \
  -r ~/CyberLab/tcpdump/captures/kali-ubuntu.pcap \
  tcp port 22
```

查看 ASCII：

```bash
tcpdump -nn -A \
  -r ~/CyberLab/tcpdump/captures/kali-ubuntu.pcap
```

---

# 34. 用 Wireshark 打开 tcpdump 文件

在 Kali：

```bash
wireshark ~/CyberLab/tcpdump/captures/kali-ubuntu.pcap
```

然后使用 Wireshark Display Filter：

```text
icmp
tcp.port == 22
tcp.port == 4444
http
udp.port == 5555
```

这体现：

```text
tcpdump 负责抓
Wireshark 负责详细分析
```

---

# 35. 完整时间格式

实时抓包：

```bash
sudo tcpdump -i eth0 -nn -tttt icmp
```

输出类似：

```text
2026-08-05 11:20:15.123456
```

这有利于和 Wazuh 事件时间对应。

---

# 36. 把输出保存为文本

```bash
sudo tcpdump -i eth0 -nn -tttt icmp \
  | tee ~/CyberLab/tcpdump/notes/icmp-live.txt
```

注意：

```text
这是文本输出
不是 PCAP
```

如果需要原始数据，应使用：

```text
-w file.pcap
```

---

# 37. 同时查看和保存

tcpdump 本身在使用 `-w` 时不会正常逐包显示。

可以用两个终端：

终端一保存：

```bash
sudo tcpdump -i eth0 -nn -s 0 \
  -w ~/CyberLab/tcpdump/captures/session.pcap \
  host 10.10.20.20
```

终端二实时查看：

```bash
sudo tcpdump -i eth0 -nn \
  host 10.10.20.20
```

---

# 38. 限制抓包时间

使用 `timeout`：

```bash
sudo timeout 30 \
  tcpdump -i eth0 -nn -s 0 \
  -w ~/CyberLab/tcpdump/captures/30-seconds.pcap
```

含义：

```text
抓包 30 秒后自动停止
```

---

# 39. 限制数据包数量

```bash
sudo tcpdump -i eth0 -nn -c 100 \
  -w ~/CyberLab/tcpdump/captures/100-packets.pcap
```

捕获 100 个包后自动停止。

---

# 40. 按文件大小轮转

```bash
sudo tcpdump -i eth0 -nn -s 0 \
  -C 10 \
  -W 5 \
  -w ~/CyberLab/tcpdump/captures/ring.pcap
```

含义：

```text
-C 10 = 每个文件约 10 MB
-W 5  = 最多保留 5 个文件
```

可能生成：

```text
ring.pcap0
ring.pcap1
ring.pcap2
ring.pcap3
ring.pcap4
```

达到上限后会覆盖旧文件。

这叫：

```text
Ring Buffer
```

---

# 41. 按时间轮转

每 60 秒创建一个文件：

```bash
sudo tcpdump -i eth0 -nn -s 0 \
  -G 60 \
  -w '~/CyberLab/tcpdump/captures/capture-%Y%m%d-%H%M%S.pcap'
```

注意：

```text
如果不配合其他限制，可能不断创建文件
```

实验中可搭配：

```bash
sudo timeout 300 \
  tcpdump -i eth0 -nn -s 0 \
  -G 60 \
  -w '~/CyberLab/tcpdump/captures/capture-%Y%m%d-%H%M%S.pcap'
```

这样运行 5 分钟，每分钟一个文件。

---

# 42. Snapshot Length

默认可能只捕获部分数据。

完整捕获：

```bash
-s 0
```

例如：

```bash
sudo tcpdump -i eth0 -nn -s 0 \
  -w full-packets.pcap
```

只需要头部时可以限制：

```bash
-s 96
```

但初学实验建议：

```text
-s 0
```

---

# 43. 抓取 Wazuh Agent 流量

在 Wazuh-SIEM：

```bash
sudo tcpdump -i enp0s1 -nn -s 0 \
  -w ~/CyberLab/tcpdump/captures/agents.pcap \
  'tcp port 1514 or tcp port 1515'
```

等待约 1–2 分钟后停止。

读取：

```bash
tcpdump -nn \
  -r ~/CyberLab/tcpdump/captures/agents.pcap
```

你应该看到：

```text
10.10.20.20 ↔ 10.10.20.40
10.10.20.30 ↔ 10.10.20.40
```

---

# 44. 只抓 Ubuntu Agent

在 Wazuh：

```bash
sudo tcpdump -i enp0s1 -nn -s 0 \
  -w ~/CyberLab/tcpdump/captures/ubuntu-agent.pcap \
  'host 10.10.20.20 and (tcp port 1514 or tcp port 1515)'
```

---

# 45. 只抓 Windows Agent

在 Wazuh：

```bash
sudo tcpdump -i enp0s1 -nn -s 0 \
  -w ~/CyberLab/tcpdump/captures/windows-agent.pcap \
  'host 10.10.20.30 and (tcp port 1514 or tcp port 1515)'
```

---

# 46. 从服务器复制 PCAP 到 Kali

从 Kali 复制 Ubuntu PCAP：

```bash
scp labuser@10.10.20.20:~/CyberLab/tcpdump/captures/ubuntu-agent.pcap \
  ~/CyberLab/tcpdump/captures/
```

复制 Wazuh PCAP：

```bash
scp labuser@10.10.20.40:~/CyberLab/tcpdump/captures/windows-agent.pcap \
  ~/CyberLab/tcpdump/captures/
```

然后：

```bash
wireshark ~/CyberLab/tcpdump/captures/windows-agent.pcap
```

---

# 47. 在远程服务器抓包

在 Kali 通过 SSH 登录 Ubuntu：

```bash
ssh labuser@10.10.20.20
```

然后运行：

```bash
sudo tcpdump -i enp0s1 -nn -c 20 icmp
```

这种方式适合 Ubuntu Server 没有图形界面时使用。

---

# 48. 远程实时传输 PCAP 到 Kali

高级但实用的安全实验方式：

在 Kali：

```bash
ssh labuser@10.10.20.20 \
  "sudo tcpdump -i enp0s1 -U -w - host 10.10.20.10" \
  > ~/CyberLab/tcpdump/captures/remote-ubuntu.pcap
```

说明：

```text
-w - = 把 PCAP 写到标准输出
-U   = 数据包缓冲后尽快写出
SSH 负责加密传输
Kali 把输出保存为 PCAP
```

完成后按：

```text
Ctrl + C
```

注意：

- 只在自己的 CyberLab 使用
- 需要远程 sudo 权限
- 不要长时间运行
- 确保磁盘空间足够

---

# 49. 管道与 grep

只查看包含 `Flags` 的 TCP 输出：

```bash
sudo tcpdump -i eth0 -nn tcp \
  | grep --line-buffered 'Flags'
```

记录并查看：

```bash
sudo tcpdump -i eth0 -nn -l tcp port 4444 \
  | tee ~/CyberLab/tcpdump/notes/netcat-live.txt
```

`-l` 让 tcpdump 逐行输出，便于管道处理。

---

# 50. tcpdump 与 Wazuh 关联

场景：

```text
Kali 对 Ubuntu 扫描
```

tcpdump 可以证明：

```text
来源 IP
目标 IP
目标端口
TCP Flags
扫描时间
包数量
```

Wazuh 可能提供：

```text
Ubuntu 日志
SSH 登录失败
防火墙日志
规则 ID
告警等级
MITRE ATT&CK 映射
```

完整证据链：

```text
tcpdump PCAP
+
Wireshark 分析
+
Ubuntu/Windows 本地日志
+
Wazuh 告警
```

---

# 51. 实验：错误 SSH 登录关联

## Kali：开始抓包

```bash
sudo tcpdump -i eth0 -nn -tttt -s 0 \
  -w ~/CyberLab/tcpdump/captures/ssh-failed-login.pcap \
  'host 10.10.20.20 and tcp port 22'
```

## Kali 第二个终端

```bash
ssh wrong-user@10.10.20.20
```

输入一次错误密码，然后：

```text
Ctrl + C
```

停止抓包。

## Wireshark

```bash
wireshark \
  ~/CyberLab/tcpdump/captures/ssh-failed-login.pcap
```

过滤：

```text
tcp.port == 22
```

## Wazuh

进入：

```text
Threat Hunting
```

过滤：

```text
agent.name: ubuntu-target
```

搜索：

```text
wrong-user
```

对比：

```text
源 IP
目标 IP
事件时间
连接持续时间
Wazuh 规则
```

---

# 52. 实验：Nmap 扫描关联

Kali 抓包：

```bash
sudo tcpdump -i eth0 -nn -tttt -s 0 \
  -w ~/CyberLab/tcpdump/captures/nmap-ubuntu.pcap \
  'host 10.10.20.20 and tcp'
```

另一个终端：

```bash
sudo nmap -n -sS -p 1-100 10.10.20.20
```

停止抓包。

查看：

```bash
tcpdump -nn \
  -r ~/CyberLab/tcpdump/captures/nmap-ubuntu.pcap
```

使用 Wireshark：

```text
tcp.flags.syn == 1 &&
tcp.flags.ack == 0
```

---

# 53. 完整实验一：协议基础

## 目标

```text
捕获 ARP、ICMP 和 TCP 三次握手
```

## 命令

ARP：

```bash
sudo tcpdump -i eth0 -nn -e -c 2 arp
```

ICMP：

```bash
sudo tcpdump -i eth0 -nn -c 8 icmp
```

TCP 4444：

```bash
sudo tcpdump -i eth0 -nn tcp port 4444
```

## 记录

```text
ARP Request
ARP Reply
ICMP Request
ICMP Reply
SYN
SYN-ACK
ACK
```

---

# 54. 完整实验二：明文与加密对比

## Netcat 明文

```bash
sudo tcpdump -i eth0 -nn -A tcp port 4444
```

## HTTP 明文

```bash
sudo tcpdump -i eth0 -nn -A tcp port 8080
```

## SSH 加密

```bash
sudo tcpdump -i eth0 -nn -A tcp port 22
```

比较：

| 协议 | 是否可见正文 |
|---|---|
| Netcat | 通常可见 |
| HTTP | 通常可见 |
| SSH | 通常不可见 |

---

# 55. 完整实验三：Wazuh Agent 通信

在 Wazuh：

```bash
sudo timeout 120 \
  tcpdump -i enp0s1 -nn -s 0 \
  -w ~/CyberLab/tcpdump/captures/wazuh-agents-120s.pcap \
  'tcp port 1514 or tcp port 1515'
```

检查：

```bash
ls -lh ~/CyberLab/tcpdump/captures/wazuh-agents-120s.pcap
```

读取：

```bash
tcpdump -nn \
  -r ~/CyberLab/tcpdump/captures/wazuh-agents-120s.pcap
```

复制到 Kali 后用 Wireshark分析：

```text
Endpoints
Conversations
TCP Stream
Packet Length
Connection Duration
```

---

# 56. 实验报告模板

## Lab Information

```text
Lab Name：tcpdump Network Capture
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
使用 tcpdump 捕获并分析 CyberLab 网络流量。
```

## Capture Command

```bash
# 粘贴完整命令
```

## Filter

```text
Capture Filter：
Reason：
```

## Results

| Protocol | Source | Destination | Observation |
|---|---|---|---|
| ARP |  |  |  |
| ICMP |  |  |  |
| TCP |  |  |  |
| UDP |  |  |  |
| HTTP |  |  |  |
| SSH |  |  |  |

## Evidence

```text
PCAP 文件：
文本日志：
Wireshark 截图：
Wazuh 事件：
目标系统日志：
```

## Analysis

```text
捕获了多少数据包？
是否出现丢包？
是否看到 SYN/SYN-ACK/ACK？
是否看到 RST？
哪些协议是明文？
哪些协议是加密的？
```

## Lessons Learned

```text
为什么选择这个接口？
为什么使用 -nn？
为什么需要 -s 0？
为什么限制抓包时间或大小？
tcpdump 与 Wireshark 如何配合？
```

---

# 57. 常见错误与解决方法

## 找不到接口

```bash
sudo tcpdump -D
ip -br address
```

## Permission denied

使用：

```bash
sudo tcpdump ...
```

## 抓不到流量

确认：

- 选对接口
- 目标虚拟机已开机
- 使用正确 IP
- 实际产生了流量
- 过滤器没有过度限制

## 过滤器语法错误

复杂表达式使用引号：

```bash
sudo tcpdump -i enp0s1 -nn \
  'tcp port 1514 or tcp port 1515'
```

## PCAP 文件为空

检查：

```bash
ls -lh file.pcap
```

确认抓包期间确实产生匹配流量。

## 文件权限是 root

因为使用 `sudo`，文件可能属于 root。

修复：

```bash
sudo chown "$USER":"$USER" file.pcap
```

目录示例：

```bash
sudo chown -R "$USER":"$USER" \
  ~/CyberLab/tcpdump/captures
```

## PCAP 太大

使用：

```text
-c
timeout
-C
-W
-G
更严格的 BPF filter
```

## 显示 checksum 错误

虚拟网卡可能使用 Checksum Offloading。

这不一定是真实网络错误。

---

# 58. 抓包性能与丢包

抓包结束后会显示：

```text
packets captured
packets received by filter
packets dropped by kernel
```

如果 `dropped by kernel` 很高，可能原因：

- 流量太大
- 磁盘写入慢
- 虚拟机资源不足
- 过滤器太宽
- 抓包程序处理不过来

解决：

- 缩小过滤范围
- 使用 `-w` 直接写 PCAP
- 限制时间
- 提高 VM 资源
- 使用 Ring Buffer

---

# 59. 抓包安全与隐私

PCAP 可能包含：

- IP 和 MAC
- HTTP 内容
- 用户名
- Cookie
- 文件内容
- 内部服务信息
- 未加密凭据
- 系统主机名

因此：

```text
不要公开上传真实网络 PCAP
不要抓取他人流量
不要把 PCAP 放进公开 GitHub
不要长时间无过滤抓包
实验结束后停止
妥善保存或删除文件
```

---

# 60. 本课练习

1. 在 Kali 捕获 4 次 Ping 的 8 个 ICMP 包。
2. 清空 ARP 缓存并捕获 ARP Request/Reply。
3. 捕获 Netcat TCP 4444 并显示 ASCII。
4. 捕获 TCP 三次握手。
5. 捕获关闭端口的 RST-ACK。
6. 捕获 Nmap SYN Scan 20–30。
7. 捕获 HTTP 8080 明文请求。
8. 捕获 SSH 22 并说明为什么看不到密码。
9. 捕获 UDP 5555。
10. 保存一个完整 PCAP。
11. 从 PCAP 中只读取 TCP 22。
12. 使用 `timeout` 自动抓包 30 秒。
13. 使用 `-C` 和 `-W` 创建 Ring Buffer。
14. 在 Wazuh 抓取 Ubuntu Agent 的 1514 流量。
15. 把 Wazuh PCAP 复制到 Kali 并用 Wireshark 打开。
16. 关联一次错误 SSH 登录与 Wazuh 告警。
17. 写一份完整实验报告。

---

# 61. 复习题

1. tcpdump 的主要用途是什么？
2. `-i` 的作用是什么？
3. `-nn` 为什么常用？
4. `-c 10` 是什么意思？
5. `-w` 与 `-r` 的区别是什么？
6. `-A` 与 `-X` 的区别是什么？
7. `-s 0` 的作用是什么？
8. 如何只捕获 Ubuntu 的流量？
9. 如何只捕获 TCP 22？
10. 如何捕获 1514 或 1515？
11. TCP `[S]` 表示什么？
12. `[S.]` 表示什么？
13. `[R.]` 表示什么？
14. 为什么 SSH 内容通常不可读？
15. 为什么 HTTP 内容通常可读？
16. Ring Buffer 有什么用途？
17. 如何限制抓包 30 秒？
18. 如何把 Wazuh PCAP 复制到 Kali？
19. tcpdump 与 Wireshark 如何配合？
20. 为什么不能抓取未经授权的网络？

---

# 62. 参考答案

1. 命令行捕获和查看网络数据包。
2. 指定抓包接口。
3. 避免 DNS 和端口名解析，提高速度和清晰度。
4. 捕获 10 个包后停止。
5. `-w` 保存 PCAP，`-r` 读取 PCAP。
6. `-A` 显示 ASCII，`-X` 显示 Hex 和 ASCII。
7. 捕获完整数据包。
8. `host 10.10.20.20`。
9. `tcp port 22`。
10. `'tcp port 1514 or tcp port 1515'`。
11. SYN。
12. SYN-ACK。
13. RST-ACK。
14. SSH 加密应用数据。
15. HTTP 默认不加密。
16. 限制文件数量和磁盘占用，循环覆盖旧文件。
17. 使用 `timeout 30 tcpdump ...`。
18. 使用 `scp`。
19. tcpdump 抓包，Wireshark 详细分析。
20. 保护隐私并遵守法律和授权范围。

---

# 63. 命令速查表

## 查看接口

```bash
sudo tcpdump -D
```

## 基础抓包

```bash
sudo tcpdump -i eth0 -nn
```

## Ubuntu 流量

```bash
sudo tcpdump -i eth0 -nn host 10.10.20.20
```

## ICMP

```bash
sudo tcpdump -i eth0 -nn icmp
```

## ARP

```bash
sudo tcpdump -i eth0 -nn -e arp
```

## TCP 22

```bash
sudo tcpdump -i eth0 -nn tcp port 22
```

## TCP 4444 明文

```bash
sudo tcpdump -i eth0 -nn -A tcp port 4444
```

## 保存 PCAP

```bash
sudo tcpdump -i eth0 -nn -s 0 \
  -w capture.pcap
```

## 读取 PCAP

```bash
tcpdump -nn -r capture.pcap
```

## 30 秒抓包

```bash
sudo timeout 30 tcpdump -i eth0 -nn -s 0 \
  -w 30s.pcap
```

## Ring Buffer

```bash
sudo tcpdump -i eth0 -nn -s 0 \
  -C 10 -W 5 \
  -w ring.pcap
```

## Wazuh Agent

```bash
sudo tcpdump -i enp0s1 -nn \
  'tcp port 1514 or tcp port 1515'
```

---

# 64. 核心记忆口诀

```text
-i：选接口
-nn：不解析
-c：限制包数
-w：写 PCAP
-r：读 PCAP
-A：看 ASCII
-X：看 Hex + ASCII
-s 0：抓完整
-C：限制单文件大小
-W：限制文件数量
-G：按时间轮转

tcpdump 抓包
Wireshark 分析
Wazuh 看日志和告警
```

---

# 65. 完成标准

完成本课后，你应该拥有：

```text
ICMP 抓包
ARP 抓包
Netcat TCP 4444 抓包
Nmap SYN Scan 抓包
HTTP 8080 抓包
SSH 22 抓包
UDP 5555 抓包
Ubuntu/Wazuh Agent PCAP
一个 Ring Buffer 实验
一份 tcpdump 实验报告
至少一个用 Wireshark 打开的 tcpdump PCAP
```

检查：

```bash
find ~/CyberLab/tcpdump -type f
```

---

# 66. 下一课

下一课建议：

```text
curl
```

学习内容：

- HTTP 请求与响应
- GET、HEAD、POST
- Header
- Status Code
- Cookie
- JSON API
- TLS 证书
- 重定向
- 文件下载
- 与 Burp Suite 和 Wireshark 联动

目前学习关系：

```text
Nmap：发现和扫描
Netcat：连接和传输
Wireshark：图形化分析
tcpdump：命令行抓包
```

---

# 67. 安全边界

只允许抓取：

```text
Kali-CyberLab      10.10.20.10
Ubuntu-Target      10.10.20.20
Windows-Target     10.10.20.30
Wazuh-SIEM         10.10.20.40
```

不要抓取：

- 学校网络
- 公司网络
- 公共 Wi‑Fi
- 邻居通信
- 家庭其他成员流量
- 未经授权的云服务器
- 随机互联网目标

每次实验：

```text
确认授权范围
选择正确接口
使用必要过滤器
限制抓包时间和大小
避免真实敏感数据
实验结束后停止抓包
妥善保存或删除 PCAP
```
