# 第二课：Netcat 完整学习指南

> **实验环境：你自己的 CyberLab**
>
> 本课程只用于你自己创建并拥有的虚拟机。  
> 不要连接、监听、测试或传输数据到未经授权的系统。

---

# 1. 本课目标

学完本课后，你应该能够：

```text
理解 Netcat 的基本作用
建立 TCP 监听器和客户端连接
使用 Netcat 测试端口
发送和接收文本
安全地在实验室传输测试文件
发送原始 HTTP 请求
测试 UDP 通信
理解重定向符号和管道
使用 ss、tcpdump 和 Wazuh 观察连接
理解 Netcat 与 Nmap 的区别
排查常见连接问题
```

本课主要使用：

```text
Kali-CyberLab
IP：10.10.20.10
角色：客户端 / 测试机

Ubuntu-Target
IP：10.10.20.20
角色：监听服务器 / Linux 靶机

Windows-Target
IP：10.10.20.30
角色：可选测试目标

Wazuh-SIEM
IP：10.10.20.40
角色：日志与安全监控
```

实验网络：

```text
CyberLab：10.10.20.0/24
```

---

# 2. 什么是 Netcat？

Netcat 是一个简单的网络通信工具。

常见命令名称：

```text
nc
netcat
ncat
```

本课主要使用：

```text
nc
```

Netcat 经常被称为：

```text
网络工具中的瑞士军刀
```

它可以：

- 建立 TCP 连接
- 建立 UDP 连接
- 监听端口
- 发送文本
- 接收文本
- 测试端口是否开放
- 读取服务 Banner
- 发送原始协议请求
- 在隔离实验室传输测试文件
- 帮助排查网络和防火墙问题

Netcat 本身不是漏洞扫描器。

它更像一个通用的：

```text
网络客户端
+
网络监听器
+
数据输入输出工具
```

---

# 3. Netcat、Ncat 和 Socat 的区别

| 工具 | 说明 |
|---|---|
| Netcat / `nc` | 简单、轻量、适合基础 TCP/UDP 实验 |
| Ncat / `ncat` | Nmap 项目提供，支持更多功能，例如 TLS |
| Socat | 更强大的双向数据转发工具，语法更复杂 |

本课只学习：

```text
Netcat / nc
```

以后可以单独学习：

```text
Ncat
Socat
```

---

# 4. 检查 Netcat 是否安装

在 Kali 中：

```bash
which nc
```

查看帮助：

```bash
nc -h
```

查看实际程序：

```bash
readlink -f "$(which nc)"
```

在 Ubuntu 中：

```bash
which nc
```

```bash
nc -h
```

如果没有安装，可以在临时联网状态下执行：

```bash
sudo apt update
sudo apt install netcat-openbsd
```

注意：

不同 Netcat 版本的参数可能略有差异。

本课命令以 Kali 和 Ubuntu 常见的：

```text
OpenBSD Netcat
```

为主。

---

# 5. Netcat 基本语法

作为客户端连接目标：

```bash
nc [选项] 目标IP 目标端口
```

示例：

```bash
nc 10.10.20.20 4444
```

作为服务器监听：

```bash
nc -l [选项] 端口
```

示例：

```bash
nc -lvn 4444
```

---

# 6. 常用参数

| 参数 | 含义 |
|---|---|
| `-l` | Listen，监听模式 |
| `-v` | Verbose，显示详细信息 |
| `-n` | 不进行 DNS 解析 |
| `-z` | Zero-I/O，只测试端口，不发送正常数据 |
| `-u` | 使用 UDP |
| `-w 3` | 超时 3 秒 |
| `-k` | 保持监听，接受多个连接 |
| `-p` | 指定本地源端口；不同版本支持方式可能不同 |
| `-4` | 只使用 IPv4 |
| `-6` | 只使用 IPv6 |

最常见组合：

```text
-lvn
```

含义：

```text
-l：监听
-v：显示详细信息
-n：不解析 DNS
```

---

# 7. 开始前检查

确认四台虚拟机使用正确 IP。

## Kali

```bash
ip -br address
```

预期：

```text
eth0    UP    10.10.20.10/24
```

## Ubuntu

```bash
ip -br address
```

预期：

```text
enp0s1    UP    10.10.20.20/24
```

## 基础 Ping

在 Kali：

```bash
ping -c 3 10.10.20.20
```

预期：

```text
3 packets transmitted
3 received
0% packet loss
```

如果 Ping 不通，先不要进行 Netcat 实验。

---

# 8. 创建本课目录

在 Kali：

```bash
mkdir -p ~/CyberLab/Netcat/{files,captures,notes}
```

在 Ubuntu：

```bash
mkdir -p ~/CyberLab/Netcat/{received,logs}
```

检查：

```bash
find ~/CyberLab/Netcat -maxdepth 2 -type d
```

---

# 9. 实验一：建立简单 TCP 聊天连接

## 实验目标

建立：

```text
Kali
→ TCP 4444
→ Ubuntu
```

Ubuntu 作为监听器，Kali 作为客户端。

---

## 9.1 在 Ubuntu 启动监听器

在 Ubuntu 终端执行：

```bash
nc -lvn 4444
```

你可能看到：

```text
Listening on 0.0.0.0 4444
```

此时终端会停在监听状态，这是正常的。

不要关闭这个终端。

---

## 9.2 在 Kali 连接 Ubuntu

打开 Kali 终端：

```bash
nc -vn 10.10.20.20 4444
```

成功时可能显示：

```text
Connection to 10.10.20.20 4444 port [tcp/*] succeeded!
```

---

## 9.3 发送文字

在 Kali 输入：

```text
Hello from Kali
```

按 Enter。

Ubuntu 应该立即看到：

```text
Hello from Kali
```

在 Ubuntu 输入：

```text
Hello from Ubuntu
```

Kali 也会看到。

这说明连接是双向的：

```text
Kali ↔ Ubuntu
```

---

## 9.4 结束连接

在任意一端按：

```text
Ctrl + C
```

连接会关闭。

---

# 10. 理解刚才发生了什么

Ubuntu 执行：

```bash
nc -lvn 4444
```

意思是：

```text
在 TCP 4444 端口等待连接
```

Kali 执行：

```bash
nc -vn 10.10.20.20 4444
```

意思是：

```text
连接到 Ubuntu 的 TCP 4444
```

建立连接后，键盘输入会通过 TCP 发送到另一端。

Netcat 不理解文字的含义。

它只是负责：

```text
读取输入
→ 通过网络发送
→ 在另一端输出
```

---

# 11. 使用 ss 检查监听端口

保持 Ubuntu 的监听器运行。

打开 Ubuntu 第二个终端或通过 SSH 登录：

```bash
ssh labuser@10.10.20.20
```

检查监听端口：

```bash
ss -lnt
```

只查看 TCP 4444：

```bash
ss -lnt | grep ':4444'
```

可能看到：

```text
LISTEN 0 1 0.0.0.0:4444 0.0.0.0:*
```

参数：

```text
-l = listening
-n = 显示数字端口
-t = TCP
```

查看相关进程：

```bash
sudo ss -lntp | grep ':4444'
```

---

# 12. 从 Kali 使用 Nmap 验证监听端口

Ubuntu 保持监听：

```bash
nc -lvn 4444
```

在 Kali：

```bash
sudo nmap -n -p 4444 10.10.20.20
```

预期：

```text
4444/tcp open
```

关闭 Ubuntu Netcat 后再次扫描：

```bash
sudo nmap -n -p 4444 10.10.20.20
```

预期可能变成：

```text
4444/tcp closed
```

这个实验证明：

```text
监听程序启动
→ 端口 open

监听程序停止
→ 端口 closed
```

---

# 13. 实验二：使用 Netcat 测试开放端口

Netcat 的 `-z` 模式可以测试端口。

## 13.1 测试 Ubuntu SSH

在 Kali：

```bash
nc -zvn -w 3 10.10.20.20 22
```

预期：

```text
Connection to 10.10.20.20 22 port [tcp/*] succeeded!
```

参数：

```text
-z：只测试连接
-v：显示结果
-n：不进行 DNS 查询
-w 3：最多等待 3 秒
```

---

## 13.2 测试关闭端口

```bash
nc -zvn -w 3 10.10.20.20 65000
```

可能显示：

```text
Connection refused
```

含义：

```text
目标在线
但端口没有服务监听
```

如果显示超时，可能表示：

```text
防火墙丢弃了数据包
```

---

## 13.3 测试 Wazuh 端口

```bash
nc -zvn -w 3 10.10.20.40 443
```

```bash
nc -zvn -w 3 10.10.20.40 1514
```

```bash
nc -zvn -w 3 10.10.20.40 1515
```

```bash
nc -zvn -w 3 10.10.20.40 55000
```

你可以记录哪些端口成功。

---

# 14. 使用循环测试多个端口

Netcat 不像 Nmap 那样适合大规模扫描。

可以用 Bash 循环检查少量端口：

```bash
for port in 22 80 443 1514 1515; do
  nc -zvn -w 2 10.10.20.20 "$port"
done
```

测试 Wazuh：

```bash
for port in 443 1514 1515 55000 9200; do
  nc -zvn -w 2 10.10.20.40 "$port"
done
```

只测试自己明确拥有的实验系统。

大范围扫描仍然优先使用：

```text
Nmap
```

---

# 15. Netcat 与 Nmap 的区别

| 功能 | Netcat | Nmap |
|---|---:|---:|
| 测试一个端口 | 很方便 | 可以 |
| 大范围扫描 | 不推荐 | 推荐 |
| 服务版本识别 | 很有限 | 强 |
| 操作系统识别 | 不支持 | 支持 |
| NSE 脚本 | 不支持 | 支持 |
| 发送自定义文本 | 支持 | 不适合 |
| 建立监听器 | 支持 | 不用于此目的 |
| 文件流传输 | 支持 | 不支持 |
| TCP/UDP 调试 | 很适合 | 适合扫描 |

简单记忆：

```text
Nmap：发现和扫描
Netcat：连接和传输
```

---

# 16. 实验三：读取 SSH Banner

SSH 服务通常会在连接后发送 Banner。

在 Kali：

```bash
nc -nv -w 3 10.10.20.20 22
```

可能看到：

```text
SSH-2.0-OpenSSH_8.9p1 Ubuntu-3ubuntu0.x
```

按：

```text
Ctrl + C
```

结束连接。

Banner 可以帮助识别：

- 服务类型
- 软件名称
- 协议版本
- 软件版本

注意：

Banner 不一定完全准确。

管理员可以修改或隐藏 Banner。

---

# 17. 实验四：使用 Netcat 发送 HTTP 请求

## 17.1 在 Ubuntu 启动临时 Web 服务

Ubuntu 通常有 Python 3。

执行：

```bash
mkdir -p ~/CyberLab/Netcat/web
```

创建网页：

```bash
echo "CyberLab Netcat HTTP Test" \
  > ~/CyberLab/Netcat/web/index.html
```

进入目录：

```bash
cd ~/CyberLab/Netcat/web
```

启动临时 HTTP Server：

```bash
python3 -m http.server 8080 --bind 10.10.20.20
```

保持此终端运行。

---

## 17.2 Kali 测试端口

```bash
nc -zvn -w 3 10.10.20.20 8080
```

预期成功。

---

## 17.3 发送原始 HTTP GET 请求

在 Kali：

```bash
printf 'GET / HTTP/1.0\r\nHost: 10.10.20.20\r\n\r\n' |
nc -nv 10.10.20.20 8080
```

你应该看到类似：

```text
HTTP/1.0 200 OK
Server: SimpleHTTP/...
Content-type: text/html
Content-Length: ...
```

最后看到：

```text
CyberLab Netcat HTTP Test
```

---

## 17.4 发送 HEAD 请求

```bash
printf 'HEAD / HTTP/1.0\r\nHost: 10.10.20.20\r\n\r\n' |
nc -nv 10.10.20.20 8080
```

HEAD 请求通常只返回响应头，不返回网页正文。

---

## 17.5 停止临时 Web 服务

回到 Ubuntu 的 Python Server 终端：

```text
Ctrl + C
```

---

# 18. 理解 HTTP 请求格式

刚才发送的是：

```text
GET / HTTP/1.0
Host: 10.10.20.20
```

HTTP 请求由以下部分组成：

```text
请求方法
请求路径
HTTP 版本
请求头
空行
```

`\r\n` 表示 HTTP 使用的换行。

两个连续的：

```text
\r\n\r\n
```

表示：

```text
请求头结束
```

---

# 19. 实验五：传输一个测试文本文件

> Netcat 传输不加密。
>
> 只在隔离 CyberLab 中传输无敏感信息的测试文件。

## 19.1 在 Kali 创建测试文件

```bash
echo "CyberLab Netcat file transfer test" \
  > ~/CyberLab/Netcat/files/message.txt
```

查看：

```bash
cat ~/CyberLab/Netcat/files/message.txt
```

生成哈希：

```bash
sha256sum ~/CyberLab/Netcat/files/message.txt
```

记录哈希值。

---

## 19.2 Ubuntu 准备接收

在 Ubuntu：

```bash
nc -lvn 4445 \
  > ~/CyberLab/Netcat/received/message.txt
```

终端会等待连接。

---

## 19.3 Kali 发送文件

在 Kali：

```bash
nc -nv 10.10.20.20 4445 \
  < ~/CyberLab/Netcat/files/message.txt
```

等待连接结束。

如果发送后没有自动返回，可以按：

```text
Ctrl + C
```

---

## 19.4 Ubuntu 验证文件

```bash
cat ~/CyberLab/Netcat/received/message.txt
```

计算哈希：

```bash
sha256sum ~/CyberLab/Netcat/received/message.txt
```

如果两边 SHA-256 相同，说明文件内容一致。

---

# 20. 理解输入输出重定向

## 20.1 `>` 写入文件

```bash
nc -lvn 4445 > received.txt
```

含义：

```text
Netcat 收到的数据
→ 写入 received.txt
```

## 20.2 `<` 从文件读取

```bash
nc 10.10.20.20 4445 < message.txt
```

含义：

```text
读取 message.txt
→ 发送给 Netcat
```

## 20.3 `|` 管道

```bash
printf 'hello\n' | nc 10.10.20.20 4444
```

含义：

```text
printf 的输出
→ 作为 Netcat 的输入
```

---

# 21. 实验六：使用管道发送系统信息

在 Kali：

```bash
uname -a |
nc -nv 10.10.20.20 4446
```

在执行前，Ubuntu 必须先监听：

```bash
nc -lvn 4446 \
  > ~/CyberLab/Netcat/received/kali-uname.txt
```

Ubuntu 查看：

```bash
cat ~/CyberLab/Netcat/received/kali-uname.txt
```

这里只发送普通系统信息，不发送密码或敏感数据。

---

# 22. 实验七：UDP 通信

TCP 和 UDP 使用方式不同。

## 22.1 Ubuntu 监听 UDP 5555

```bash
nc -u -lvn 5555
```

参数：

```text
-u = UDP
-l = Listen
```

---

## 22.2 Kali 发送 UDP 文本

```bash
printf 'Hello over UDP\n' |
nc -u -vn -w 2 10.10.20.20 5555
```

Ubuntu 应看到：

```text
Hello over UDP
```

---

## 22.3 UDP 的特点

UDP 没有完整连接握手。

即使 Kali 没有显示明确成功，也不代表数据一定没有到达。

所以 UDP 测试通常需要：

- 在接收端直接观察
- 使用 tcpdump 抓包
- 使用 Wireshark 分析
- 检查应用日志

---

# 23. 实验八：保持监听多个连接

某些 Netcat 版本支持：

```text
-k
```

Ubuntu：

```bash
nc -lvnk 4447
```

然后 Kali 可以重复连接：

```bash
nc -nv 10.10.20.20 4447
```

结束一次连接后，再重新连接。

如果你的 Netcat 版本不支持该组合，使用循环：

```bash
while true; do
  nc -lvn 4447
done
```

按：

```text
Ctrl + C
```

停止循环。

---

# 24. 使用 tcpdump 观察 Netcat

在 Ubuntu 启动监听器：

```bash
nc -lvn 4444
```

在 Ubuntu 的第二个终端抓包：

```bash
sudo tcpdump -i enp0s1 -nn tcp port 4444
```

然后在 Kali 连接：

```bash
nc -nv 10.10.20.20 4444
```

tcpdump 会显示类似：

```text
10.10.20.10.xxxxx > 10.10.20.20.4444
10.10.20.20.4444 > 10.10.20.10.xxxxx
```

停止抓包：

```text
Ctrl + C
```

---

# 25. 保存 PCAP 抓包

在 Ubuntu：

```bash
sudo tcpdump \
  -i enp0s1 \
  -nn \
  tcp port 4444 \
  -w ~/CyberLab/Netcat/logs/netcat-4444.pcap
```

然后进行一次 Netcat 连接。

完成后按：

```text
Ctrl + C
```

查看文件：

```bash
ls -lh ~/CyberLab/Netcat/logs/netcat-4444.pcap
```

以后可以用 Wireshark 打开该文件。

---

# 26. 在 Wazuh 中观察 Netcat 实验

## 26.1 重要事实

普通 Netcat TCP 连接不一定自动产生 Wazuh 告警。

Wazuh Agent 主要收集：

- 系统日志
- 应用日志
- 身份验证日志
- Windows Event Logs
- 文件完整性事件
- 安全配置和系统清单

如果操作系统没有记录连接，Wazuh 就可能没有可分析的日志。

---

## 26.2 创建明确的实验日志

在 Ubuntu 启动监听器前执行：

```bash
logger -t netcat-lab \
  "Starting authorized Netcat listener on TCP 4444"
```

完成连接后执行：

```bash
logger -t netcat-lab \
  "Completed authorized Netcat connection from Kali 10.10.20.10"
```

查看本地日志：

```bash
journalctl -t netcat-lab --since "10 minutes ago"
```

---

## 26.3 在 Wazuh Dashboard 搜索

打开：

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
netcat-lab
```

或者：

```text
Starting authorized Netcat listener
```

成功时应看到来自：

```text
ubuntu-target
```

的日志事件。

这证明：

```text
Ubuntu logger
→ Wazuh Agent
→ Wazuh Manager
→ Indexer
→ Dashboard
```

---

# 27. 更准确地检测 Netcat 的方法

要自动检测 Netcat 执行，可以以后学习：

```text
auditd
Linux Audit Rules
Wazuh Auditd Integration
Sysmon for Linux
Process monitoring
Network IDS
Suricata
Zeek
```

例如，Auditd 可以记录：

```text
谁执行了 /usr/bin/nc
什么时候执行
使用了什么参数
由哪个用户执行
```

这属于以后课程：

```text
Wazuh Process Monitoring
```

本课先理解：

```text
网络流量存在
不等于系统一定有日志
```

---

# 28. Windows 中测试 Ubuntu Netcat 端口

Windows 不需要安装 Netcat，也可以测试端口。

在 Windows PowerShell：

```powershell
Test-NetConnection 10.10.20.20 -Port 4444
```

Ubuntu 必须先运行：

```bash
nc -lvn 4444
```

成功时：

```text
TcpTestSucceeded : True
```

关闭 Ubuntu 的监听器后再次测试：

```powershell
Test-NetConnection 10.10.20.20 -Port 4444
```

可能显示：

```text
TcpTestSucceeded : False
```

---

# 29. 使用 Netcat 检查 Windows 端口

在 Kali：

```bash
nc -zvn -w 3 10.10.20.30 445
```

```bash
nc -zvn -w 3 10.10.20.30 3389
```

如果没有启用 SMB 或 RDP，可能显示：

```text
Connection refused
```

或者超时。

这不代表 Windows 离线。

可以先测试：

```bash
ping -c 3 10.10.20.30
```

或使用：

```bash
sudo nmap -n -Pn -p 445,3389 10.10.20.30
```

---

# 30. Netcat 连接结果解释

## 30.1 Succeeded

```text
succeeded
```

表示：

```text
TCP 连接成功
目标端口有服务监听
防火墙允许连接
```

## 30.2 Connection refused

```text
Connection refused
```

表示：

```text
目标主机在线
目标端口没有服务监听
或者系统主动拒绝连接
```

## 30.3 Timed out

```text
timed out
```

可能表示：

```text
防火墙丢弃数据包
网络不通
目标关闭
路由错误
```

## 30.4 No route to host

```text
No route to host
```

表示：

```text
本机没有到目标网络的有效路由
或者防火墙返回不可达消息
```

---

# 31. 常见错误与解决方法

## 31.1 Address already in use

错误：

```text
Address already in use
```

原因：

```text
端口已经被其他进程占用
```

检查：

```bash
sudo ss -lntp | grep ':4444'
```

换一个端口：

```bash
nc -lvn 4448
```

---

## 31.2 Connection refused

检查监听器是否启动：

```bash
sudo ss -lntp | grep ':4444'
```

检查目标 IP：

```bash
ip -br address
```

检查防火墙：

```bash
sudo ufw status verbose
```

---

## 31.3 Connection timed out

检查：

```bash
ping -c 3 10.10.20.20
```

检查 UTM：

```text
Network Mode：Host Only
Host Network：CyberLab
```

检查目标防火墙：

```bash
sudo ufw status verbose
```

---

## 31.4 文件传输后命令不结束

Netcat 不一定知道文件什么时候传完。

可以：

```text
等待几秒
然后按 Ctrl + C
```

也可以使用支持关闭连接参数的 Ncat。

正式文件传输应优先使用：

```text
SCP
SFTP
HTTPS
```

---

## 31.5 两边都在等待

如果两边都执行监听：

```bash
nc -lvn 4444
```

就没有客户端发起连接。

正确关系：

```text
一端 Listen
另一端 Connect
```

---

## 31.6 使用错误 IP

确认：

```text
Kali：10.10.20.10
Ubuntu：10.10.20.20
Windows：10.10.20.30
Wazuh：10.10.20.40
```

---

# 32. 安全注意事项

Netcat 传输的数据默认：

```text
没有加密
没有身份验证
没有完整性保护
```

因此不要通过 Netcat 发送：

- 密码
- 私钥
- 真实个人资料
- 银行资料
- 身份文件
- 真实公司数据
- 敏感日志
- 生产环境配置

本课只发送：

```text
普通测试文字
无敏感信息的测试文件
实验室系统信息
```

---

# 33. 为什么本课不使用 Netcat Shell

Netcat 经常被攻击者用于建立远程 Shell。

但在基础阶段，先学习：

```text
TCP 连接
监听器
端口测试
数据流
HTTP 请求
文件传输
日志和抓包
```

以后学习远程 Shell 时，需要先掌握：

- 明确授权范围
- Linux 权限
- Windows 权限
- 防火墙
- 进程日志
- Wazuh 告警
- 快照恢复
- 安全清理

本课不进行：

```text
持久化
绕过安全软件
权限提升
真实系统远程控制
```

---

# 34. 第一个完整 Netcat 实验

## 实验名称

```text
Netcat TCP Connectivity and Monitoring
```

## 授权范围

```text
Kali：10.10.20.10
Ubuntu：10.10.20.20
TCP：4444
```

## 实验步骤

### Ubuntu：记录实验开始

```bash
logger -t netcat-lab \
  "Starting authorized TCP 4444 lab"
```

### Ubuntu：启动监听

```bash
nc -lvn 4444
```

### Kali：测试端口

```bash
nc -zvn -w 3 10.10.20.20 4444
```

### Kali：建立连接

```bash
nc -vn 10.10.20.20 4444
```

### Kali：发送

```text
NETCAT_LAB_TEST_FROM_KALI
```

### Ubuntu：回复

```text
NETCAT_LAB_TEST_FROM_UBUNTU
```

### 结束

```text
Ctrl + C
```

### Ubuntu：记录实验完成

```bash
logger -t netcat-lab \
  "Completed authorized TCP 4444 lab"
```

### Wazuh 搜索

```text
agent.name: ubuntu-target
```

搜索：

```text
netcat-lab
```

---

# 35. 第二个完整实验：端口状态变化

## 步骤一：监听前扫描

Kali：

```bash
sudo nmap -n -p 4444 10.10.20.20
```

记录状态。

## 步骤二：Ubuntu 启动监听

```bash
nc -lvn 4444
```

## 步骤三：再次扫描

```bash
sudo nmap -n -p 4444 10.10.20.20
```

记录状态。

## 步骤四：停止监听

```text
Ctrl + C
```

## 步骤五：再次扫描

```bash
sudo nmap -n -p 4444 10.10.20.20
```

比较：

```text
监听前
监听中
监听后
```

---

# 36. 第三个完整实验：文件传输与哈希验证

## Kali 创建文件

```bash
date > ~/CyberLab/Netcat/files/lab-report.txt
```

```bash
hostname >> ~/CyberLab/Netcat/files/lab-report.txt
```

```bash
ip -br address >> ~/CyberLab/Netcat/files/lab-report.txt
```

计算哈希：

```bash
sha256sum ~/CyberLab/Netcat/files/lab-report.txt
```

## Ubuntu 接收

```bash
nc -lvn 4445 \
  > ~/CyberLab/Netcat/received/lab-report.txt
```

## Kali 发送

```bash
nc -nv 10.10.20.20 4445 \
  < ~/CyberLab/Netcat/files/lab-report.txt
```

## Ubuntu 验证

```bash
cat ~/CyberLab/Netcat/received/lab-report.txt
```

```bash
sha256sum ~/CyberLab/Netcat/received/lab-report.txt
```

两边哈希应该相同。

---

# 37. 第四个完整实验：HTTP 手动请求

## Ubuntu

```bash
mkdir -p ~/CyberLab/Netcat/web
```

```bash
echo "Netcat HTTP Lab" \
  > ~/CyberLab/Netcat/web/index.html
```

```bash
cd ~/CyberLab/Netcat/web
python3 -m http.server 8080 --bind 10.10.20.20
```

## Kali

```bash
printf 'GET / HTTP/1.0\r\nHost: 10.10.20.20\r\n\r\n' |
nc -nv 10.10.20.20 8080
```

记录：

```text
HTTP 状态码
Server Header
Content-Type
Content-Length
网页正文
```

---

# 38. 实验报告模板

## Lab Information

```text
Lab Name：Netcat TCP Connectivity
Date：
Student：
Client：Kali-CyberLab 10.10.20.10
Server：Ubuntu-Target 10.10.20.20
Port：4444/TCP
Authorization：Personal CyberLab
```

## Objective

```text
使用 Netcat 建立 TCP 连接，测试端口，发送文本，并观察日志。
```

## Commands

```bash
# Ubuntu listener

# Kali client

# Verification commands
```

## Results

| Test | Expected | Actual | Status |
|---|---|---|---|
| Ping Ubuntu | Reply |  |  |
| TCP 4444 before listener | Closed |  |  |
| TCP 4444 during listener | Open |  |  |
| Text from Kali | Received |  |  |
| Text from Ubuntu | Received |  |  |
| Wazuh log | Visible |  |  |

## Evidence

```text
Kali screenshot：
Ubuntu screenshot：
Nmap result：
tcpdump capture：
Wazuh event：
```

## Findings

```text
监听器启动后端口状态发生什么变化？
防火墙是否允许连接？
Wazuh 是否自动检测到网络连接？
```

## Lessons Learned

```text
Netcat 和 Nmap 的区别是什么？
什么是监听器？
什么是客户端？
什么是输入输出重定向？
为什么 Netcat 文件传输不适合敏感数据？
```

---

# 39. 本课练习

## 练习一

在 Ubuntu 监听：

```text
TCP 5000
```

从 Kali 连接并发送：

```text
My first Netcat lab
```

---

## 练习二

使用 Netcat 测试：

```text
Ubuntu TCP 22
Windows TCP 445
Wazuh TCP 443
Wazuh TCP 1514
```

记录成功或失败。

---

## 练习三

传输一个名为：

```text
cyberlab-note.txt
```

的测试文件。

使用 SHA-256 验证内容一致。

---

## 练习四

在 Ubuntu 启动 Python HTTP Server：

```text
TCP 8081
```

使用 Netcat 手动发送 HTTP GET。

---

## 练习五

使用 UDP 发送：

```text
NETCAT_UDP_TEST
```

并使用 tcpdump 证明数据包到达。

---

## 练习六

比较以下工具测试 TCP 22 的结果：

```bash
nc -zvn -w 3 10.10.20.20 22
```

```bash
sudo nmap -n -p 22 10.10.20.20
```

```powershell
Test-NetConnection 10.10.20.20 -Port 22
```

---

# 40. 复习题

1. Netcat 常用命令是什么？
2. `-l` 表示什么？
3. `-v` 表示什么？
4. `-n` 为什么适合 CyberLab？
5. `-z` 的作用是什么？
6. `-u` 的作用是什么？
7. `-w 3` 表示什么？
8. 为什么一端必须监听，另一端必须连接？
9. `>` 和 `<` 分别有什么作用？
10. `|` 管道有什么作用？
11. `Connection refused` 通常表示什么？
12. `Timed out` 可能表示什么？
13. Netcat 和 Nmap 的主要区别是什么？
14. Netcat 文件传输是否加密？
15. 普通 Netcat 连接为什么不一定出现在 Wazuh？
16. 如何查看 Ubuntu 的监听端口？
17. 如何停止 Netcat？
18. TCP 与 UDP 的主要区别是什么？
19. 为什么要验证 SHA-256？
20. 为什么只允许在自己的 CyberLab 使用？

---

# 41. 参考答案

1. `nc`。
2. 进入监听模式。
3. 显示详细信息。
4. CyberLab 没有 DNS，避免不必要的解析和等待。
5. 只测试端口，不进行普通数据通信。
6. 使用 UDP。
7. 最多等待 3 秒。
8. 两端都监听时，没有一方主动发起连接。
9. `>` 把输出写入文件；`<` 从文件读取输入。
10. 把前一个命令的输出交给后一个命令。
11. 主机在线，但端口没有服务监听或主动拒绝。
12. 防火墙丢弃、目标离线、路由问题或网络不通。
13. Nmap 负责发现和扫描；Netcat 负责连接和传输。
14. 默认不加密。
15. 操作系统可能没有记录原始网络连接。
16. `ss -lntp`。
17. `Ctrl + C`。
18. TCP 有连接和可靠传输；UDP 无完整握手。
19. 确认传输前后文件内容一致。
20. 避免未经授权访问和法律风险。

---

# 42. 常用命令速查表

## TCP 监听

```bash
nc -lvn 4444
```

## TCP 连接

```bash
nc -vn 10.10.20.20 4444
```

## 测试端口

```bash
nc -zvn -w 3 10.10.20.20 22
```

## UDP 监听

```bash
nc -u -lvn 5555
```

## UDP 发送

```bash
printf 'hello\n' |
nc -u -vn -w 2 10.10.20.20 5555
```

## 接收文件

```bash
nc -lvn 4445 > received.txt
```

## 发送文件

```bash
nc -nv 10.10.20.20 4445 < file.txt
```

## SSH Banner

```bash
nc -nv -w 3 10.10.20.20 22
```

## HTTP GET

```bash
printf 'GET / HTTP/1.0\r\nHost: 10.10.20.20\r\n\r\n' |
nc -nv 10.10.20.20 8080
```

## 查看监听端口

```bash
sudo ss -lntp
```

## 抓取 Netcat 流量

```bash
sudo tcpdump -i enp0s1 -nn tcp port 4444
```

---

# 43. 核心记忆口诀

```text
-l：Listen，监听
-v：Verbose，看详细信息
-n：No DNS，不解析名字
-z：Zero-I/O，只测试端口
-u：UDP
-w：Wait，设置超时

一边监听
一边连接

Nmap 找端口
Netcat 连端口
```

---

# 44. 完成标准

完成本课后，你应该拥有：

```text
一次 TCP 双向聊天记录
一次端口状态变化实验
一次测试文件传输
两份 SHA-256 结果
一次 UDP 通信
一次 HTTP 原始请求
一个 tcpdump PCAP 文件
至少两条 Wazuh netcat-lab 测试日志
一份实验报告
```

检查 Kali 文件：

```bash
find ~/CyberLab/Netcat -type f
```

检查 Ubuntu 文件：

```bash
find ~/CyberLab/Netcat -type f
```

---

# 45. 下一课

下一课建议：

```text
Wireshark
```

学习内容：

- 选择正确网卡
- 捕获 TCP 三次握手
- 观察 Netcat 明文内容
- 分析 TCP Flags
- 使用 Display Filters
- 观察 HTTP 请求
- 观察 UDP 数据
- 保存和打开 PCAP
- 对比 Nmap 与 Netcat 流量

完成 Netcat 后，你已经掌握：

```text
Nmap：发现和扫描
Netcat：连接和传输
```

下一步用 Wireshark观察这些数据包在网络中实际是什么样子。

---

# 46. 安全边界

本课程只允许连接：

```text
Ubuntu-Target：10.10.20.20
Windows-Target：10.10.20.30
Wazuh-SIEM：10.10.20.40
```

不要连接或监听：

- 学校网络
- 公司网络
- 公共 Wi-Fi
- 邻居设备
- 随机公网 IP
- 未经授权的网站
- 非自己的云服务器
- 家庭中不属于实验范围的设备

所有实验应满足：

```text
目标属于自己
实验有明确目的
不传输敏感数据
实验结束后停止监听器
保存必要证据
恢复安全配置
```
