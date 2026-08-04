# 第一课：Nmap 完整学习指南

> **实验环境：自己的 CyberLab**
>
> 本课程只用于你自己创建并拥有的虚拟机环境。  
> 不要扫描学校、公司、公共 Wi-Fi、随机公网 IP 或没有明确授权的系统。

---

# 1. 本课目标

学完本课后，你应该能够回答：

```text
目标主机是否在线？
目标开放了哪些端口？
端口对应什么服务？
服务版本是什么？
Windows 和 Linux 的扫描结果有什么区别？
扫描结果如何保存？
如何在 Wazuh 中寻找相关日志？
```

你将使用：

```text
攻击测试机：Kali-CyberLab
IP：10.10.20.10

Linux 靶机：Ubuntu-Target
IP：10.10.20.20

Windows 靶机：Windows-Target
IP：10.10.20.30

SIEM：Wazuh-SIEM
IP：10.10.20.40

实验网络：
10.10.20.0/24
```

---

# 2. 什么是 Nmap？

Nmap 全称：

```text
Network Mapper
```

它是一款网络发现和端口扫描工具。

常见用途：

- 发现在线主机
- 扫描开放端口
- 判断端口状态
- 识别网络服务
- 识别服务版本
- 尝试识别操作系统
- 运行安全检查脚本
- 保存扫描结果
- 比较系统加固前后的变化

Nmap 不只是“攻击工具”。

系统管理员也经常使用它检查：

- 哪些设备连接到网络
- 哪些端口不应该开放
- 防火墙规则是否生效
- 服务是否正常运行
- 修复后端口是否关闭

---

# 3. 什么是端口？

一台计算机可以同时运行多个网络服务。

不同服务通常监听不同端口。

| 端口 | 常见服务 | 用途 |
|---:|---|---|
| 22 | SSH | Linux 远程管理 |
| 53 | DNS | 域名解析 |
| 80 | HTTP | 普通网站 |
| 443 | HTTPS | 加密网站 |
| 445 | SMB | Windows 文件共享 |
| 3389 | RDP | Windows 远程桌面 |
| 1514 | Wazuh | Agent 日志通信 |
| 1515 | Wazuh | Agent 注册 |
| 55000 | Wazuh API | Wazuh API |
| 9200 | Wazuh Indexer | 索引服务 |

TCP 和 UDP 各自拥有：

```text
0–65535
```

共 65,536 个端口。

通常扫描范围是：

```text
1–65535
```

---

# 4. TCP 和 UDP

## 4.1 TCP

TCP 是面向连接的协议。

特点：

- 建立连接
- 确认数据到达
- 数据顺序可靠
- 常用于 SSH、HTTP、HTTPS、SMB、RDP

TCP 三次握手：

```text
Client → SYN
Server → SYN-ACK
Client → ACK
```

## 4.2 UDP

UDP 不需要建立完整连接。

特点：

- 速度快
- 没有连接确认
- 扫描通常更慢
- 常用于 DNS、DHCP、SNMP、NTP

---

# 5. Nmap 基本语法

基本格式：

```bash
nmap [选项] [目标]
```

示例：

```bash
nmap 10.10.20.20
```

多个目标：

```bash
nmap 10.10.20.20 10.10.20.30
```

扫描整个实验网段：

```bash
nmap 10.10.20.0/24
```

扫描一个范围：

```bash
nmap 10.10.20.20-40
```

---

# 6. 开始前检查

在 Kali 中打开终端。

确认 Kali IP：

```bash
ip -br address
```

预期：

```text
eth0    UP    10.10.20.10/24
```

确认路由：

```bash
ip route
```

预期：

```text
10.10.20.0/24 dev eth0
```

不应该有：

```text
default via ...
```

检查 Nmap：

```bash
nmap --version
```

如果没有安装：

```bash
sudo apt install nmap
```

---

# 7. 创建实验目录

在 Kali 中创建本课目录：

```bash
mkdir -p ~/CyberLab/Nmap/{scans,notes}
```

查看：

```bash
tree ~/CyberLab/Nmap
```

如果没有 `tree`：

```bash
sudo apt install tree
```

目录结构：

```text
CyberLab/
└── Nmap/
    ├── notes/
    └── scans/
```

---

# 8. 实验一：主机发现

## 8.1 扫描整个 CyberLab

```bash
sudo nmap -sn -n 10.10.20.0/24
```

参数解释：

| 参数 | 意思 |
|---|---|
| `sudo` | 允许 Nmap 使用低层网络功能 |
| `-sn` | 只发现主机，不扫描端口 |
| `-n` | 不进行 DNS 查询 |
| `10.10.20.0/24` | 扫描整个 CyberLab 网段 |

预期发现：

```text
10.10.20.10    Kali
10.10.20.20    Ubuntu
10.10.20.30    Windows
10.10.20.40    Wazuh
```

实际结果可能只显示：

```text
Nmap scan report for 10.10.20.20
Host is up
```

## 8.2 只查看在线主机

```bash
sudo nmap -sn -n 10.10.20.0/24 | grep "Nmap scan report"
```

## 8.3 使用 ARP 发现本地网络主机

因为四台虚拟机位于同一个二层网络，可以使用：

```bash
sudo nmap -sn -PR -n 10.10.20.0/24
```

参数：

```text
-PR = ARP Ping
```

在本地局域网中，ARP 主机发现通常很可靠。

---

# 9. 实验二：基础端口扫描

扫描 Ubuntu：

```bash
sudo nmap -n 10.10.20.20
```

默认情况下，Nmap 扫描常见的 1,000 个 TCP 端口。

预期至少看到：

```text
22/tcp open ssh
```

扫描 Windows：

```bash
sudo nmap -n 10.10.20.30
```

Windows 可能没有开放端口。

扫描 Wazuh：

```bash
sudo nmap -n 10.10.20.40
```

Wazuh 可能显示：

```text
443/tcp   open
1514/tcp  open
1515/tcp  open
55000/tcp open
9200/tcp  open
```

具体结果取决于服务配置和防火墙。

---

# 10. 理解端口状态

Nmap 常见端口状态：

| 状态 | 意思 |
|---|---|
| `open` | 有服务正在监听 |
| `closed` | 主机可访问，但没有服务监听 |
| `filtered` | 防火墙或网络设备阻止判断 |
| `unfiltered` | 端口可访问，但不能确定是否开放 |
| `open|filtered` | 无法区分开放还是过滤 |
| `closed|filtered` | 无法区分关闭还是过滤 |

## 10.1 Open

```text
22/tcp open ssh
```

表示 TCP 22 有 SSH 服务。

## 10.2 Closed

```text
80/tcp closed http
```

表示主机在线，但 TCP 80 没有服务监听。

## 10.3 Filtered

```text
445/tcp filtered microsoft-ds
```

表示防火墙可能丢弃了扫描数据包。

---

# 11. 实验三：扫描指定端口

扫描 SSH：

```bash
sudo nmap -n -p 22 10.10.20.20
```

扫描多个端口：

```bash
sudo nmap -n -p 22,80,443 10.10.20.20
```

扫描端口范围：

```bash
sudo nmap -n -p 1-1000 10.10.20.20
```

扫描 Wazuh 重要端口：

```bash
sudo nmap -n -p 443,1514,1515,55000,9200 10.10.20.40
```

扫描 Windows 常见端口：

```bash
sudo nmap -n -p 135,139,445,3389 10.10.20.30
```

---

# 12. 实验四：扫描所有 TCP 端口

扫描 Ubuntu 全部 TCP 端口：

```bash
sudo nmap -n -sS -p- 10.10.20.20
```

参数：

```text
-sS = TCP SYN Scan
-p- = 扫描 1–65535
```

扫描 Windows：

```bash
sudo nmap -n -Pn -sS -p- 10.10.20.30
```

为什么 Windows 使用 `-Pn`？

```text
Windows 防火墙可能不回应 Ping
-Pn 告诉 Nmap：假设目标在线，继续扫描
```

全端口扫描可能需要更长时间。

---

# 13. TCP SYN 扫描

命令：

```bash
sudo nmap -n -sS 10.10.20.20
```

SYN 扫描过程：

```text
Kali → SYN
目标 → SYN-ACK
Kali → RST
```

它不会完成完整的 TCP 连接。

优点：

- 速度较快
- 常用于基础端口扫描
- 可以判断开放、关闭或过滤

通常需要：

```text
sudo
```

---

# 14. TCP Connect 扫描

命令：

```bash
nmap -n -sT 10.10.20.20
```

它会建立完整 TCP 连接：

```text
SYN
SYN-ACK
ACK
```

特点：

- 不需要 root
- 更容易被应用日志记录
- 比 SYN 扫描更明显
- 适合比较日志差异

实验：

```bash
nmap -n -sT -p 22 10.10.20.20
```

然后在 Ubuntu 查看 SSH 日志：

```bash
sudo journalctl -u ssh --since "5 minutes ago"
```

注意：

端口连接不一定等于成功认证，因此日志内容可能不同。

---

# 15. 实验五：服务与版本识别

扫描 Ubuntu 服务版本：

```bash
sudo nmap -n -sV 10.10.20.20
```

预期：

```text
22/tcp open ssh OpenSSH 8.9p1 Ubuntu
```

扫描 Wazuh：

```bash
sudo nmap -n -sV 10.10.20.40
```

参数：

```text
-sV = Service Version Detection
```

Nmap 会发送探测请求，尝试判断：

- 服务名称
- 软件名称
- 软件版本
- 操作系统信息
- 产品信息

## 15.1 调整版本探测强度

较轻探测：

```bash
sudo nmap -n -sV --version-light 10.10.20.20
```

较强探测：

```bash
sudo nmap -n -sV --version-all 10.10.20.20
```

初学时优先使用：

```text
-sV
```

---

# 16. 实验六：操作系统识别

命令：

```bash
sudo nmap -n -O 10.10.20.20
```

参数：

```text
-O = OS Detection
```

扫描 Windows：

```bash
sudo nmap -n -Pn -O 10.10.20.30
```

更完整：

```bash
sudo nmap -n -sV -O 10.10.20.20
```

注意：

操作系统识别不是 100% 准确。

准确度受以下因素影响：

- 开放端口数量
- 关闭端口数量
- 防火墙
- 虚拟化环境
- 网络设备
- 响应特征不足

---

# 17. Aggressive Scan

命令：

```bash
sudo nmap -n -A 10.10.20.20
```

`-A` 通常包含：

```text
操作系统识别
服务版本识别
默认 NSE 脚本
Traceroute
```

等价概念：

```text
-sV + -O + -sC + --traceroute
```

它会产生更多流量，也更容易被日志或安全工具发现。

初学时只对单个实验目标使用。

不要对整个网段直接运行：

```bash
sudo nmap -A 10.10.20.0/24
```

---

# 18. Nmap Scripting Engine

NSE 全称：

```text
Nmap Scripting Engine
```

它允许 Nmap 运行脚本来收集更多信息。

## 18.1 默认脚本

```bash
sudo nmap -n -sC 10.10.20.20
```

参数：

```text
-sC = --script=default
```

和版本识别一起使用：

```bash
sudo nmap -n -sC -sV 10.10.20.20
```

## 18.2 查看脚本文件

```bash
ls /usr/share/nmap/scripts | head
```

搜索 SSH 脚本：

```bash
ls /usr/share/nmap/scripts | grep ssh
```

## 18.3 安全的信息收集脚本

SSH 算法：

```bash
sudo nmap -n -p 22 --script ssh2-enum-algos 10.10.20.20
```

SSH 主机密钥：

```bash
sudo nmap -n -p 22 --script ssh-hostkey 10.10.20.20
```

HTTP 标题：

```bash
sudo nmap -n -p 80,443 --script http-title 10.10.20.40
```

TLS 证书：

```bash
sudo nmap -n -p 443 --script ssl-cert 10.10.20.40
```

TLS 加密套件：

```bash
sudo nmap -n -p 443 --script ssl-enum-ciphers 10.10.20.40
```

注意：

只在自己的 CyberLab 中运行 NSE 脚本。

---

# 19. UDP 扫描

UDP 扫描通常比 TCP 慢。

扫描常见 UDP 端口：

```bash
sudo nmap -n -sU --top-ports 20 10.10.20.20
```

扫描特定 UDP 端口：

```bash
sudo nmap -n -sU -p 53,67,68,123,161 10.10.20.20
```

参数：

```text
-sU = UDP Scan
```

UDP 常见状态：

```text
open
closed
open|filtered
```

因为 UDP 没有握手，很多时候 Nmap 无法确定是开放还是过滤。

---

# 20. 扫描速度与时序

Nmap 时序模板：

| 模板 | 名称 | 用途 |
|---|---|---|
| `-T0` | Paranoid | 极慢 |
| `-T1` | Sneaky | 很慢 |
| `-T2` | Polite | 较慢 |
| `-T3` | Normal | 默认 |
| `-T4` | Aggressive | 实验室常用 |
| `-T5` | Insane | 很快，可能不准确 |

在 CyberLab 中可以使用：

```bash
sudo nmap -n -T4 -sV 10.10.20.20
```

不要认为越快越好。

过快可能导致：

- 丢包
- 扫描结果不准确
- 防火墙触发
- 服务压力增加
- Wazuh 告警增加

初学建议：

```text
默认 -T3
或实验室使用 -T4
```

---

# 21. 保存扫描结果

## 21.1 普通文本

```bash
sudo nmap -n -sV 10.10.20.20 \
  -oN ~/CyberLab/Nmap/scans/ubuntu-baseline.nmap
```

## 21.2 XML

```bash
sudo nmap -n -sV 10.10.20.20 \
  -oX ~/CyberLab/Nmap/scans/ubuntu-baseline.xml
```

## 21.3 Grepable

```bash
sudo nmap -n -sV 10.10.20.20 \
  -oG ~/CyberLab/Nmap/scans/ubuntu-baseline.gnmap
```

## 21.4 同时保存三种格式

```bash
sudo nmap -n -sV 10.10.20.20 \
  -oA ~/CyberLab/Nmap/scans/ubuntu-baseline
```

生成：

```text
ubuntu-baseline.nmap
ubuntu-baseline.xml
ubuntu-baseline.gnmap
```

`-oA` 最适合实验记录。

---

# 22. 推荐的基线扫描

## 22.1 Ubuntu 基线

```bash
sudo nmap -n -sS -sV -O 10.10.20.20 \
  -oA ~/CyberLab/Nmap/scans/ubuntu-baseline
```

## 22.2 Windows 基线

```bash
sudo nmap -n -Pn -sS -sV -O 10.10.20.30 \
  -oA ~/CyberLab/Nmap/scans/windows-baseline
```

## 22.3 Wazuh 基线

```bash
sudo nmap -n -sS -sV 10.10.20.40 \
  -oA ~/CyberLab/Nmap/scans/wazuh-baseline
```

## 22.4 全网主机发现

```bash
sudo nmap -sn -n 10.10.20.0/24 \
  -oA ~/CyberLab/Nmap/scans/cyberlab-host-discovery
```

---

# 23. 查看保存结果

普通文本：

```bash
cat ~/CyberLab/Nmap/scans/ubuntu-baseline.nmap
```

分页查看：

```bash
less ~/CyberLab/Nmap/scans/ubuntu-baseline.nmap
```

只查看开放端口：

```bash
grep "/tcp.*open" ~/CyberLab/Nmap/scans/ubuntu-baseline.nmap
```

查看 XML 文件：

```bash
head ~/CyberLab/Nmap/scans/ubuntu-baseline.xml
```

---

# 24. 比较修复前后的结果

假设修复前：

```bash
sudo nmap -n -sV 10.10.20.20 \
  -oN ~/CyberLab/Nmap/scans/ubuntu-before.nmap
```

在 Ubuntu 配置防火墙后：

```bash
sudo nmap -n -sV 10.10.20.20 \
  -oN ~/CyberLab/Nmap/scans/ubuntu-after.nmap
```

比较：

```bash
diff \
  ~/CyberLab/Nmap/scans/ubuntu-before.nmap \
  ~/CyberLab/Nmap/scans/ubuntu-after.nmap
```

也可以使用 Ndiff：

```bash
ndiff \
  ~/CyberLab/Nmap/scans/ubuntu-before.xml \
  ~/CyberLab/Nmap/scans/ubuntu-after.xml
```

---

# 25. 在 Wazuh 中观察扫描

## 25.1 需要理解的重要事实

Nmap 扫描属于网络活动。

Wazuh Agent 主要收集：

- 操作系统日志
- 应用日志
- Windows Event Logs
- 文件完整性变化
- 系统清单
- 安全配置
- 部分网络相关事件

因此：

```text
Nmap 扫描不一定自动产生明显 Wazuh 告警
```

是否能看到扫描，取决于目标是否记录了相关事件，例如：

- SSH 服务产生连接日志
- Windows Firewall 开启日志
- Linux 防火墙记录丢弃数据包
- IDS/IPS 记录扫描行为
- 应用服务记录连接
- 自定义 Wazuh 规则分析这些日志

## 25.2 搜索 Ubuntu 相关事件

在 Wazuh Dashboard：

```text
Threat Hunting
```

筛选：

```text
agent.name: ubuntu-target
```

时间范围：

```text
Last 15 minutes
```

如果使用了 SSH 版本探测，可以查看：

```bash
sudo journalctl -u ssh --since "15 minutes ago"
```

Ubuntu Agent 日志：

```bash
sudo tail -n 50 /var/ossec/logs/ossec.log
```

## 25.3 搜索 Windows 相关事件

在 Dashboard 筛选：

```text
agent.name: windows-target
```

Windows Defender Firewall 默认不一定记录所有丢弃连接。

可以在 Windows 中检查：

```powershell
Get-NetFirewallProfile |
Select-Object Name,Enabled,LogBlocked,LogAllowed
```

---

# 26. 第一个完整扫描实验

## 实验名称

```text
Nmap Baseline Assessment
```

## 目标

```text
发现 CyberLab 主机
扫描 Ubuntu、Windows 和 Wazuh
识别开放端口和服务
保存扫描结果
观察目标日志和 Wazuh
```

## 允许范围

```text
10.10.20.20
10.10.20.30
10.10.20.40
```

## 禁止范围

```text
Mac 主机
家庭路由器
家庭其他设备
公网 IP
学校或公司网络
```

## 步骤一：发现主机

```bash
sudo nmap -sn -n 10.10.20.0/24 \
  -oA ~/CyberLab/Nmap/scans/lab1-host-discovery
```

## 步骤二：扫描 Ubuntu

```bash
sudo nmap -n -sS -sV 10.10.20.20 \
  -oA ~/CyberLab/Nmap/scans/lab1-ubuntu
```

## 步骤三：扫描 Windows

```bash
sudo nmap -n -Pn -sS -sV 10.10.20.30 \
  -oA ~/CyberLab/Nmap/scans/lab1-windows
```

## 步骤四：扫描 Wazuh

```bash
sudo nmap -n -sS -sV \
  -p 443,1514,1515,55000,9200 \
  10.10.20.40 \
  -oA ~/CyberLab/Nmap/scans/lab1-wazuh
```

## 步骤五：记录结果

```bash
ls -lh ~/CyberLab/Nmap/scans
```

---

# 27. 第二个完整实验：SSH 枚举

目标：

```text
Ubuntu-Target
10.10.20.20
TCP 22
```

## 27.1 基础扫描

```bash
sudo nmap -n -p 22 10.10.20.20
```

## 27.2 服务版本

```bash
sudo nmap -n -sV -p 22 10.10.20.20
```

## 27.3 SSH 主机密钥

```bash
sudo nmap -n -p 22 \
  --script ssh-hostkey \
  10.10.20.20
```

## 27.4 SSH 算法

```bash
sudo nmap -n -p 22 \
  --script ssh2-enum-algos \
  10.10.20.20
```

## 27.5 保存结果

```bash
sudo nmap -n -sV -p 22 \
  --script ssh-hostkey,ssh2-enum-algos \
  10.10.20.20 \
  -oA ~/CyberLab/Nmap/scans/lab2-ubuntu-ssh
```

## 27.6 Ubuntu 查看日志

```bash
sudo journalctl -u ssh --since "10 minutes ago"
```

---

# 28. 第三个完整实验：Wazuh HTTPS 枚举

目标：

```text
Wazuh-SIEM
10.10.20.40
TCP 443
```

## 28.1 检查端口

```bash
sudo nmap -n -p 443 10.10.20.40
```

## 28.2 识别服务

```bash
sudo nmap -n -sV -p 443 10.10.20.40
```

## 28.3 查看证书

```bash
sudo nmap -n -p 443 \
  --script ssl-cert \
  10.10.20.40
```

## 28.4 查看 TLS 加密套件

```bash
sudo nmap -n -p 443 \
  --script ssl-enum-ciphers \
  10.10.20.40
```

## 28.5 查看网页标题

```bash
sudo nmap -n -p 443 \
  --script http-title \
  10.10.20.40
```

## 28.6 保存结果

```bash
sudo nmap -n -sV -p 443 \
  --script ssl-cert,ssl-enum-ciphers,http-title \
  10.10.20.40 \
  -oA ~/CyberLab/Nmap/scans/lab3-wazuh-https
```

---

# 29. 常见错误与解决方法

## 29.1 Host seems down

错误：

```text
Note: Host seems down
```

解决：

```bash
sudo nmap -Pn -n 10.10.20.30
```

原因：

```text
目标可能阻止 Ping
```

## 29.2 DNS Warning

如果看到 DNS 相关警告，使用：

```text
-n
```

你的 CyberLab 没有 DNS，这是正常设计。

## 29.3 Operation not permitted

使用：

```bash
sudo nmap ...
```

## 29.4 所有端口都是 filtered

可能原因：

- Windows Firewall
- Ubuntu UFW
- Wazuh Server 防火墙
- 网卡连接错误
- 目标不在 CyberLab
- 目标虚拟机已关机

先测试：

```bash
ping -c 3 10.10.20.20
```

再测试 ARP：

```bash
ip neigh
```

## 29.5 扫描非常慢

可能原因：

- 使用 `-p-`
- UDP 扫描
- 目标丢弃数据包
- 没有使用 `-n`
- 防火墙导致超时

可以在实验室尝试：

```bash
sudo nmap -n -T4 10.10.20.20
```

## 29.6 结果和预期不一致

检查目标服务：

Ubuntu：

```bash
sudo ss -lntup
```

Windows：

```powershell
Get-NetTCPConnection -State Listen
```

Wazuh：

```bash
sudo ss -lntup
```

---

# 30. Nmap 重要参数速查表

| 参数 | 含义 |
|---|---|
| `-sn` | 只进行主机发现 |
| `-n` | 禁止 DNS 解析 |
| `-Pn` | 假设主机在线 |
| `-sS` | TCP SYN 扫描 |
| `-sT` | TCP Connect 扫描 |
| `-sU` | UDP 扫描 |
| `-sV` | 服务版本识别 |
| `-O` | 操作系统识别 |
| `-A` | 综合扫描 |
| `-sC` | 默认 NSE 脚本 |
| `-p 22` | 扫描端口 22 |
| `-p 1-1000` | 扫描端口范围 |
| `-p-` | 扫描全部 TCP 端口 |
| `--top-ports 20` | 扫描最常见 20 个端口 |
| `-T4` | 较快时序模板 |
| `-oN` | 普通文本输出 |
| `-oX` | XML 输出 |
| `-oG` | Grepable 输出 |
| `-oA` | 同时保存多种格式 |
| `--script` | 运行 NSE 脚本 |
| `--reason` | 显示判断端口状态的原因 |
| `--open` | 只显示开放端口 |
| `-v` | 显示更多过程信息 |
| `-vv` | 显示更详细信息 |

---

# 31. 推荐命令组合

## 主机发现

```bash
sudo nmap -sn -n 10.10.20.0/24
```

## 快速基础扫描

```bash
sudo nmap -n 10.10.20.20
```

## 服务版本扫描

```bash
sudo nmap -n -sV 10.10.20.20
```

## Windows 扫描

```bash
sudo nmap -n -Pn -sV 10.10.20.30
```

## 全 TCP 端口扫描

```bash
sudo nmap -n -sS -p- 10.10.20.20
```

## 默认脚本加版本识别

```bash
sudo nmap -n -sC -sV 10.10.20.20
```

## 保存完整结果

```bash
sudo nmap -n -sC -sV 10.10.20.20 \
  -oA ~/CyberLab/Nmap/scans/ubuntu-detailed
```

## Wazuh 重要端口

```bash
sudo nmap -n -sV \
  -p 443,1514,1515,55000,9200 \
  10.10.20.40
```

---

# 32. 实验报告模板

## Lab Information

```text
Lab Name：Nmap Baseline Assessment
Date：
Student：
Attacker：Kali-CyberLab 10.10.20.10
Targets：
- Ubuntu-Target 10.10.20.20
- Windows-Target 10.10.20.30
- Wazuh-SIEM 10.10.20.40
Authorization：Personal CyberLab
```

## Objective

```text
发现在线主机，识别开放端口和服务，并保存基线扫描结果。
```

## Commands

```bash
# 把本次使用的命令粘贴到这里
```

## Findings

| Target | Port | State | Service | Version | Notes |
|---|---:|---|---|---|---|
| Ubuntu | 22 | Open | SSH |  |  |
| Windows |  |  |  |  |  |
| Wazuh | 443 | Open | HTTPS |  |  |

## Evidence

```text
扫描文件：
截图：
Wazuh 查询：
目标系统日志：
```

## Risk Analysis

```text
哪些端口是必要的？
哪些端口可能不需要？
哪些服务暴露范围过大？
防火墙是否生效？
```

## Recommendation

```text
关闭不需要的服务。
限制管理端口来源 IP。
启用并检查防火墙。
保持软件更新。
```

## Lessons Learned

```text
今天学到了什么？
哪个参数最重要？
扫描结果中有哪些疑问？
下一步要验证什么？
```

---

# 33. 本课练习

## 练习一

发现 CyberLab 中所有在线主机。

要求：

```text
使用 -sn
禁止 DNS 查询
保存结果
```

## 练习二

扫描 Ubuntu 的常见 TCP 端口。

回答：

```text
哪些端口开放？
什么服务正在运行？
服务版本是什么？
```

## 练习三

扫描 Windows。

回答：

```text
为什么需要 -Pn？
Windows 是否有开放端口？
如果没有开放端口，这是否代表 Windows 完全安全？
```

## 练习四

扫描 Wazuh 的以下端口：

```text
443
1514
1515
55000
9200
```

回答：

```text
哪些端口开放？
每个端口的功能是什么？
哪些端口应该只允许 CyberLab 访问？
```

## 练习五

比较：

```text
-sS
-sT
```

记录：

```text
扫描速度
扫描结果
目标日志变化
Wazuh 是否显示相关事件
```

## 练习六

使用：

```text
ssl-cert
ssl-enum-ciphers
```

检查 Wazuh Dashboard 的 HTTPS 服务。

---

# 34. 复习题

1. Nmap 全称是什么？
2. `-sn` 的作用是什么？
3. `-n` 为什么适合 CyberLab？
4. `-Pn` 什么时候使用？
5. `-sS` 和 `-sT` 有什么区别？
6. `-sV` 的作用是什么？
7. `-O` 是否一定准确？
8. `-p-` 扫描多少个端口？
9. `open` 和 `filtered` 有什么区别？
10. 为什么 UDP 扫描通常更慢？
11. `-oA` 会生成哪些文件？
12. Nmap 扫描为什么不一定在 Wazuh 中产生告警？
13. 如何比较修复前后的扫描结果？
14. 为什么不能扫描随机公网 IP？
15. Windows 不回应 Ping 时，应该添加哪个参数？

---

# 35. 参考答案

1. Network Mapper。
2. 只进行主机发现，不扫描端口。
3. CyberLab 没有 DNS，`-n` 可以避免无用的 DNS 查询。
4. 目标阻止 Ping，但你知道目标在线时。
5. `-sS` 不完成完整连接；`-sT` 完成完整 TCP 连接。
6. 识别服务和版本。
7. 不一定，防火墙和虚拟化会影响结果。
8. TCP 1–65535。
9. `open` 表示有服务监听；`filtered` 表示防火墙阻止判断。
10. UDP 没有连接握手，响应不明确，Nmap 需要等待超时。
11. `.nmap`、`.xml` 和 `.gnmap`。
12. Wazuh 依赖端点或网络设备产生可分析日志。
13. 使用 `diff` 或 `ndiff`。
14. 因为没有授权，可能违反法律和服务规则。
15. `-Pn`。

---

# 36. 完成标准

完成本课时，你应该已经拥有：

```text
CyberLab 主机发现结果
Ubuntu 基线扫描
Windows 基线扫描
Wazuh 基线扫描
SSH 枚举结果
Wazuh HTTPS 枚举结果
至少一份扫描实验报告
```

检查文件：

```bash
find ~/CyberLab/Nmap -type f
```

---

# 37. 核心记忆口诀

```text
-sn：找主机
-p：选端口
-sS：扫 TCP
-sU：扫 UDP
-sV：看版本
-O：猜系统
-sC：跑默认脚本
-Pn：跳过 Ping
-n：不要 DNS
-oA：保存全部格式
```

---

# 38. 下一课

下一课：

```text
Netcat
```

学习内容：

- 测试 TCP 端口
- 建立客户端和服务器连接
- 发送简单文本
- 传输测试数据
- 对比 Nmap 和 Netcat
- 在 Ubuntu 与 Wazuh 中观察连接日志

在进入 Netcat 前，先确保你能独立完成：

```text
主机发现
端口扫描
服务识别
保存结果
阅读 Nmap 输出
```

---

# 39. 安全边界

只允许扫描：

- 自己创建的虚拟机
- 自己拥有的设备
- 明确授权的系统
- 专业训练平台明确分配给你的目标

不要扫描：

- 学校网络
- 公司网络
- 公共 Wi-Fi
- 邻居网络
- 随机公网 IP
- 未经授权的网站
- 家庭中不属于实验范围的设备

本课程授权范围：

```text
10.10.20.20
10.10.20.30
10.10.20.40
```
