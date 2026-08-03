# UTM 网络安全 Home Lab 网络配置记录

## 1. 实验室目标

使用 MacBook Pro M5 和 UTM 创建一个隔离的网络安全实验室，用于练习：

- Nmap 网络扫描
- 主机发现
- TCP/UDP 端口扫描
- 服务与版本枚举
- Web 漏洞测试
- SSH 登录与日志分析
- Windows 安全日志分析
- 防火墙配置
- Wazuh SIEM 日志收集
- Wireshark 流量分析
- 安全加固与重新扫描
- 事件响应

---

## 2. 主机配置

| 项目 | 配置 |
|---|---|
| 主机 | MacBook Pro |
| 芯片 | Apple M5 |
| 内存 | 48GB |
| 虚拟化软件 | UTM |
| 虚拟机架构 | ARM64 |
| 主要网络模式 | Host Network / 隔离网络 |
| 实验网络名称 | CyberLab |

---

## 3. 网络拓扑

```text
                         MacBook Pro M5
                               │
                    UTM Host Network
                         CyberLab
                     10.10.20.0/24
                               │
       ┌───────────────────────┼───────────────────────┐
       │                       │                       │
 Kali Linux              Ubuntu Server           Windows 11
 攻击测试机               Linux 靶机              Windows 靶机
 10.10.20.10             10.10.20.20             10.10.20.30
                                                       │
                                                       │
                                               Wazuh Server
                                               日志与 SIEM
                                               10.10.20.40
```

所有虚拟机连接到同一个名为 `CyberLab` 的隔离网络。

---

## 4. IP 地址规划

| 虚拟机 | 用途 | 固定 IP | 子网掩码 | 网关 | DNS |
|---|---|---|---|---|---|
| Kali Linux | 攻击机 | `10.10.20.10` | `/24` | 无 | 无 |
| Ubuntu Server | Linux 靶机 | `10.10.20.20` | `/24` | 无 | 无 |
| Windows 11 | Windows 靶机 | `10.10.20.30` | `/24` | 无 | 无 |
| Wazuh Server | SIEM 日志服务器 | `10.10.20.40` | `/24` | 无 | 无 |

### 网络信息

```text
网络名称：CyberLab
网络地址：10.10.20.0
CIDR：10.10.20.0/24
子网掩码：255.255.255.0
广播地址：10.10.20.255
默认网关：无
DNS：无
DHCP：无
```

### 为什么不设置网关？

没有默认网关时，虚拟机只能与同一实验网段中的设备通信。

它们不能通过实验网卡访问：

- 家庭局域网
- 路由器
- 其他家庭设备
- 互联网

这样可以降低漏洞靶机影响真实网络的风险。

---

# 5. UTM 网络模式说明

## 5.1 CyberLab 隔离网卡

主要用于攻击和防御实验。

```text
Network Mode：Host Network / macOS Host Only
Host Network：CyberLab
IP：手动固定
Gateway：留空
DNS：留空
```

CyberLab 网卡用于：

- Kali 扫描 Ubuntu
- Kali 扫描 Windows
- Kali 扫描 Wazuh
- Ubuntu 向 Wazuh 发送日志
- Windows 向 Wazuh 发送日志
- Wireshark 抓取实验流量
- 防火墙规则测试
- Web 漏洞测试

## 5.2 Shared Network 网卡

Shared Network 是临时上网网卡。

```text
Network Mode：macOS Shared Network
IP 设置：DHCP / Automatic
用途：下载软件和系统更新
```

使用 Shared Network 时，UTM 会通过 Mac 提供 NAT 互联网连接。

## 5.3 Bridged Network

不要把故意存在漏洞的靶机连接到 Bridged Network。

Bridged Network 会让虚拟机直接进入家庭局域网，看起来像一台真实的家庭设备。

不建议用于：

- OWASP Juice Shop
- WebGoat
- Metasploitable
- 故意使用弱密码的服务器
- 未打补丁的 Windows
- 恶意软件分析
- 漏洞利用实验

---

# 6. 在 UTM 创建 CyberLab 隔离网络

在安装虚拟机之前，建议先创建一个名为 `CyberLab` 的自定义 Host Network。之后，Kali、Ubuntu、Windows 和 Wazuh 都连接到这个网络。

## 6.1 创建 Host Network

打开 UTM，然后进入：

```text
UTM
→ Settings…
→ Network
→ Host Networks
→ 点击 +
```

填写网络名称：

```text
Name：CyberLab
```

保存设置。

> 在这里通常只需要创建网络名称，不需要填写 `10.10.20.0/24`。固定 IP 地址会在每台虚拟机的操作系统中手动设置。

## 6.2 CyberLab 的网络作用

`CyberLab` 用于让实验室中的虚拟机互相通信：

```text
Kali Linux
Ubuntu Server
Windows 11
Wazuh Server
```

它们必须全部连接到同一个：

```text
Host Network：CyberLab
```

实验网卡不设置：

```text
Default Gateway：留空
DNS：留空
```

这样，虚拟机可以在 `10.10.20.0/24` 网段内通信，但不能通过这张实验网卡访问互联网或家庭局域网。

## 6.3 创建虚拟机时的后端注意事项

创建 ARM64 虚拟机时选择：

```text
Create a New Virtual Machine
→ Virtualize
```

不要选择速度较慢的：

```text
Emulate
```

如果当前 UTM 版本在 Apple Virtualization 后端中看不到自定义 `CyberLab` Host Network，请改用 QEMU 虚拟化后端。

## 6.4 将虚拟机连接到 CyberLab

虚拟机创建完成后，先完全关闭虚拟机，然后打开：

```text
选择虚拟机
→ Edit
→ Devices
→ Network
```

设置：

```text
Network Mode：Host Network / macOS Host Only
Host Network：CyberLab
Emulated Network Card：virtio-net-pci
```

对下面所有虚拟机执行相同设置：

- Kali Linux
- Ubuntu Server
- Windows 11
- Wazuh Server

每台虚拟机必须使用不同的 MAC 地址。

## 6.5 CyberLab 固定 IP 规划

| 虚拟机 | 固定 IP | 子网掩码 | 网关 | DNS |
|---|---|---|---|---|
| Kali Linux | `10.10.20.10/24` | `255.255.255.0` | 留空 | 留空 |
| Ubuntu Server | `10.10.20.20/24` | `255.255.255.0` | 留空 | 留空 |
| Windows 11 | `10.10.20.30/24` | `255.255.255.0` | 留空 | 留空 |
| Wazuh Server | `10.10.20.40/24` | `255.255.255.0` | 留空 | 留空 |

## 6.6 安装阶段临时上网

安装系统或下载更新时，可以临时添加第二张网卡：

```text
Network Mode：macOS Shared Network
IP：DHCP / Automatic
用途：临时访问互联网
```

更新完成后：

```text
1. 完全关闭虚拟机
2. 禁用或删除 Shared Network 网卡
3. 只保留 CyberLab 实验网卡
4. 再开始扫描或漏洞测试
```

不要把故意存在漏洞的靶机连接到：

```text
Bridged Network
```

## 6.7 创建完成检查

- [ ] UTM 中已经创建 `CyberLab` Host Network
- [ ] 所有实验虚拟机都选择同一个 `CyberLab`
- [ ] 每台虚拟机使用不同的 MAC 地址
- [ ] 每台虚拟机使用不同的固定 IP
- [ ] CyberLab 网卡没有默认网关
- [ ] CyberLab 网卡没有 DNS
- [ ] 漏洞实验期间 Shared Network 已关闭
- [ ] Bridged Network 未启用

---


# 7. 给虚拟机添加实验网卡

对每台虚拟机执行：

```text
1. 完全关闭虚拟机
2. 在 UTM 中选择虚拟机
3. 点击 Edit
4. 打开 Devices
5. 添加或选择 Network
6. Network Mode 选择 Host Network 或 macOS Host Only
7. Host Network 选择 CyberLab
8. 保存配置
```

建议使用的虚拟网卡型号：

```text
virtio-net-pci
```

每台虚拟机必须拥有不同的 MAC 地址。

不要复制相同的 MAC 地址到多台虚拟机。

---

# 8. Kali Linux 网络配置

## 8.1 推荐资源配置

```text
CPU：6 cores
RAM：8GB
Disk：60GB
IP：10.10.20.10/24
```

## 8.2 查看网卡名称

```bash
ip -br link
```

```bash
nmcli device status
```

网卡名称可能是：

```text
eth0
ens5
enp0s1
```

## 8.3 图形界面设置

打开：

```text
Settings
→ Network
→ Wired
→ IPv4
→ Manual
```

填写：

```text
Address：10.10.20.10
Netmask：255.255.255.0
Gateway：留空
DNS：留空
```

保存后，断开并重新连接网卡。

## 8.4 使用 nmcli 设置

查看连接名称：

```bash
nmcli connection show
```

假设连接名称是：

```text
Wired connection 1
```

执行：

```bash
sudo nmcli connection modify "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses 10.10.20.10/24 \
  ipv4.gateway "" \
  ipv4.dns "" \
  ipv4.never-default yes
```

重新连接：

```bash
sudo nmcli connection down "Wired connection 1"
sudo nmcli connection up "Wired connection 1"
```

检查：

```bash
ip -br address
```

```bash
ip route
```

预期结果：

```text
10.10.20.10/24
```

实验网卡不应该出现默认路由。

---

# 9. Ubuntu Server 网络配置

## 9.1 推荐资源配置

```text
CPU：4 cores
RAM：6GB
Disk：50GB
IP：10.10.20.20/24
```

## 9.2 查看网卡名称

```bash
ip -br link
```

假设实验网卡名称为：

```text
enp0s1
```

## 9.3 查看 Netplan 配置

```bash
ls /etc/netplan
```

创建或编辑配置：

```bash
sudo nano /etc/netplan/01-cyberlab.yaml
```

填写：

```yaml
network:
  version: 2
  ethernets:
    enp0s1:
      dhcp4: false
      addresses:
        - 10.10.20.20/24
```

注意：必须将 `enp0s1` 替换成实际网卡名称。

测试配置：

```bash
sudo netplan try
```

应用配置：

```bash
sudo netplan apply
```

检查：

```bash
ip -br address
```

```bash
ip route
```

预期结果：

```text
10.10.20.20/24
```

---

# 10. Windows 11 网络配置

## 10.1 推荐资源配置

```text
CPU：6 cores
RAM：12GB
Disk：100GB
IP：10.10.20.30/24
```

## 10.2 打开网络适配器

按：

```text
Win + R
```

输入：

```text
ncpa.cpl
```

选择实验网卡：

```text
右键网卡
→ Properties
→ Internet Protocol Version 4
→ Properties
```

填写：

```text
IP address：10.10.20.30
Subnet mask：255.255.255.0
Default gateway：留空

Preferred DNS：留空
Alternate DNS：留空
```

## 10.3 重命名网卡

建议将实验网卡重命名为：

```text
LAB
```

## 10.4 将网络设置为 Private

管理员 PowerShell：

```powershell
Set-NetConnectionProfile `
  -InterfaceAlias "LAB" `
  -NetworkCategory Private
```

查看 IP 地址：

```powershell
ipconfig
```

或者：

```powershell
Get-NetIPAddress -AddressFamily IPv4
```

## 10.5 允许实验网络 Ping Windows

Windows 防火墙可能默认阻止 ICMP。

管理员 PowerShell：

```powershell
New-NetFirewallRule `
  -DisplayName "Allow CyberLab ICMPv4" `
  -Direction Inbound `
  -Protocol ICMPv4 `
  -IcmpType 8 `
  -RemoteAddress 10.10.20.0/24 `
  -Action Allow
```

这个规则只允许 CyberLab 网段 Ping Windows。

---

# 11. Wazuh Server 网络配置

## 11.1 推荐资源配置

```text
CPU：4 cores
RAM：8GB
Disk：80GB
IP：10.10.20.40/24
```

如果 Wazuh 安装在 Ubuntu Server 上，使用 Netplan 配置。

查看网卡：

```bash
ip -br link
```

编辑：

```bash
sudo nano /etc/netplan/01-cyberlab.yaml
```

填写：

```yaml
network:
  version: 2
  ethernets:
    enp0s1:
      dhcp4: false
      addresses:
        - 10.10.20.40/24
```

应用：

```bash
sudo netplan try
```

```bash
sudo netplan apply
```

确认：

```bash
ip -br address
```

预期：

```text
10.10.20.40/24
```

安装 Wazuh Agent 时，Manager 地址使用：

```text
10.10.20.40
```

---

# 12. 网络连通性测试

## 12.1 从 Kali 测试其他虚拟机

```bash
ping -c 3 10.10.20.20
```

```bash
ping -c 3 10.10.20.30
```

```bash
ping -c 3 10.10.20.40
```

## 12.2 主机发现

```bash
sudo nmap -sn 10.10.20.0/24
```

预期发现：

```text
10.10.20.10    Kali Linux
10.10.20.20    Ubuntu Server
10.10.20.30    Windows 11
10.10.20.40    Wazuh Server
```

## 12.3 Windows 测试 Ubuntu SSH

PowerShell：

```powershell
Test-NetConnection 10.10.20.20 -Port 22
```

如果 Ubuntu SSH 正常，应该显示：

```text
TcpTestSucceeded : True
```

## 12.4 Kali 测试 Ubuntu SSH

```bash
ssh labuser@10.10.20.20
```

## 12.5 测试 Web 服务

如果 Ubuntu 运行 Juice Shop：

```text
http://10.10.20.20:3000
```

如果 Ubuntu 运行 WebGoat：

```text
http://10.10.20.20:8080/WebGoat
```

如果 Wazuh Dashboard 已启动：

```text
https://10.10.20.40
```

---

# 13. 验证实验网络是否隔离

只有 CyberLab 网卡时，在 Kali 或 Ubuntu 中测试：

```bash
ping -c 3 1.1.1.1
```

应该失败。

再测试：

```bash
curl https://example.com
```

也应该失败。

查看路由表：

```bash
ip route
```

正常情况下只应看到：

```text
10.10.20.0/24 dev <实验网卡>
```

不应该看到：

```text
default via ...
```

这表示实验网卡没有默认网关。

---

# 14. 临时连接互联网的方法

需要下载工具或更新系统时，为虚拟机添加第二张网卡。

## 网卡一：CyberLab

```text
用途：攻防实验
IP：固定
Gateway：无
DNS：无
```

## 网卡二：Shared Network

```text
用途：下载和更新
IP：DHCP
Gateway：自动
DNS：自动
```

## 更新模式

```text
1. 关闭漏洞服务
2. 关闭虚拟机
3. 添加或启用 Shared Network 网卡
4. 启动虚拟机
5. 更新系统
6. 下载需要的软件
7. 关闭虚拟机
8. 禁用或删除 Shared Network 网卡
9. 重新进入隔离实验模式
```

Ubuntu 更新：

```bash
sudo apt update
sudo apt upgrade
```

Kali 更新：

```bash
sudo apt update
sudo apt full-upgrade
```

---

# 15. 攻击实验前检查

- [ ] Kali 连接 CyberLab
- [ ] Ubuntu 连接 CyberLab
- [ ] Windows 连接 CyberLab
- [ ] Wazuh 连接 CyberLab
- [ ] 所有机器使用不同 IP
- [ ] 所有机器使用不同 MAC 地址
- [ ] 实验网卡没有默认网关
- [ ] 实验网卡没有 DNS
- [ ] Shared Network 已关闭
- [ ] Bridged Network 未启用
- [ ] 靶机无法访问互联网
- [ ] 已创建虚拟机快照
- [ ] 已确认扫描目标属于自己的实验环境

---

# 16. 推荐快照名称

## Kali Linux

```text
01-Kali-Clean
02-Kali-Tools-Installed
03-Kali-Before-Lab
```

## Ubuntu Server

```text
01-Ubuntu-Clean
02-Ubuntu-SSH-Apache
03-Ubuntu-JuiceShop
04-Ubuntu-Before-Attack
```

## Windows 11

```text
01-Windows-Clean
02-Windows-Lab-Configured
03-Windows-Before-Attack
```

## Wazuh

```text
01-Wazuh-Clean
02-Wazuh-Installed
03-Wazuh-Agents-Connected
```

---

# 17. 第一次扫描实验

## 17.1 查看 Kali 网络

```bash
ip -br address
```

```bash
ip route
```

## 17.2 发现实验室设备

```bash
sudo nmap -sn 10.10.20.0/24
```

## 17.3 扫描 Ubuntu 常见端口

```bash
nmap 10.10.20.20
```

## 17.4 服务版本扫描

```bash
sudo nmap -sV 10.10.20.20
```

## 17.5 扫描全部 TCP 端口

```bash
sudo nmap -sS -p- 10.10.20.20
```

## 17.6 扫描 Windows

```bash
sudo nmap -sV 10.10.20.30
```

## 17.7 扫描 Wazuh Server

```bash
sudo nmap -sV 10.10.20.40
```

## 17.8 保存扫描结果

```bash
sudo nmap -sV 10.10.20.20 -oA ubuntu-initial-scan
```

这会生成：

```text
ubuntu-initial-scan.nmap
ubuntu-initial-scan.gnmap
ubuntu-initial-scan.xml
```

---

# 18. Wireshark 过滤器

查看实验网段：

```text
net 10.10.20.0/24
```

查看 Ubuntu 流量：

```text
ip.addr == 10.10.20.20
```

查看 Windows 流量：

```text
ip.addr == 10.10.20.30
```

查看 TCP SYN：

```text
tcp.flags.syn == 1
```

查看 SSH：

```text
tcp.port == 22
```

查看 HTTP：

```text
http
```

查看 ICMP：

```text
icmp
```

查看 ARP：

```text
arp
```

---

# 19. 安全规则

只允许测试：

- 自己创建的虚拟机
- 自己拥有的设备
- 故意存在漏洞的靶机
- 明确授权的系统
- 专业网络安全训练平台

不要扫描：

- 学校网络
- 公司网络
- 公共 Wi-Fi
- 邻居网络
- 随机公网 IP
- 未授权网站
- 非自己的云服务器
- 家庭中不属于实验范围的设备

不要在真实网络中运行：

- 密码爆破
- 漏洞利用
- ARP 欺骗
- 恶意软件
- 拒绝服务测试
- 大量自动扫描
- 未授权 Web 测试

---

# 20. 实验记录模板

## Lab Information

```text
Lab Name：
Date：
Student：
Target：
Attacker：
Network：
Authorization：
```

## Objective

```text
本次实验的目标是什么？
```

## Scope

```text
允许测试的 IP：
禁止测试的设备：
允许使用的工具：
```

## Network Diagram

```text
Kali 10.10.20.10
        │
        │ CyberLab
        │
Ubuntu 10.10.20.20
```

## Commands

```bash
# 在这里记录使用的命令
```

## Findings

| Finding | Evidence | Risk | Recommendation |
|---|---|---|---|
| 示例：SSH 对整个网段开放 | TCP 22 Open | Medium | 限制来源 IP |

## Screenshots

```text
添加截图文件名称或路径。
```

## Remediation

```text
记录进行了哪些修复。
```

## Verification

```text
修复后重新扫描并记录结果。
```

## Lessons Learned

```text
本次实验学到了什么？
哪些步骤出现问题？
下一次应该如何改进？
```

---

# 21. 最终配置摘要

```text
CyberLab：10.10.20.0/24

Kali Linux：
10.10.20.10

Ubuntu Server：
10.10.20.20

Windows 11：
10.10.20.30

Wazuh Server：
10.10.20.40

Subnet Mask：
255.255.255.0

Gateway：
无

DNS：
无

Internet：
默认关闭

Temporary Internet：
UTM Shared Network

禁止使用：
Bridged Network
```

---

# 22. 核心原则

1. 四台虚拟机必须连接同一个 `CyberLab` Host Network。
2. 每台机器使用不同的固定 IP。
3. CyberLab 网卡不设置网关和 DNS。
4. 漏洞实验期间关闭 Shared Network。
5. 不使用 Bridged Network 连接漏洞靶机。
6. 攻击前创建快照。
7. 只扫描和攻击自己拥有或明确授权的目标。
8. 每次实验都记录扫描、证据、修复和验证结果。




# 渗透测试人员通常是怎么工作的？

正规的渗透测试并不是一上来就“攻击”。标准流程通常是：

> **获得书面授权 → 确认范围 → 信息收集 → 扫描 → 枚举 → 验证漏洞 → 控制影响 → 写报告 → 协助修复 → 重新测试**

真正工作中，沟通、记录和写报告通常与使用工具同样重要。

---

## 1. 确认授权和测试范围

测试前，客户必须明确同意。

通常需要确认：

- 哪些 IP、域名和服务器允许测试
- 哪些系统不能测试
- 测试日期和时间
- 是否允许进行密码测试
- 是否允许进行漏洞利用
- 是否允许社会工程
- 是否允许测试生产环境
- 出现故障时联系谁
- 什么情况下必须立即停止

关键术语：

| 术语 | 含义 |
|---|---|
| Scope | 测试范围 |
| Rules of Engagement | 交战规则 |
| Authorization | 测试授权 |

没有授权时，即使只是扫描端口，也可能产生法律问题。

---

## 2. 信息收集（Reconnaissance）

测试人员先了解目标，而不是马上攻击。

### 2.1 被动信息收集

尽量不直接接触目标系统，例如：

- 公司网站
- DNS 记录
- 公开 IP 范围
- 员工公开信息
- 技术招聘广告
- 公开代码仓库
- 泄露的电子邮件地址
- 使用的云服务或技术

例如，招聘广告可能暴露以下技术环境：

```text
Windows Server
Active Directory
Microsoft 365
AWS
Fortinet Firewall
```

这些信息可以帮助测试人员理解企业环境。

### 2.2 主动信息收集

主动与授权目标通信，例如：

- Ping 主机
- 端口扫描
- DNS 查询
- Web 服务器探测
- 服务版本识别

---

## 3. 主机和端口扫描

测试人员需要建立资产清单：

```text
哪些主机在线？
哪些端口开放？
哪些服务正在运行？
服务版本是什么？
是否存在不必要的服务？
```

常见工具：

- Nmap
- Masscan
- Nessus
- OpenVAS / Greenbone
- Nuclei

在隔离实验室中，可以先发现主机：

```bash
nmap -sn 10.10.20.0/24
```

然后识别目标服务：

```bash
nmap -sV 10.10.20.20
```

结果可能类似：

```text
22/tcp    open    ssh
80/tcp    open    http
3000/tcp  open    web application
```

扫描结果只代表“可能存在问题”，不等于已经确认漏洞。

---

## 4. 枚举（Enumeration）

枚举比普通扫描更深入。

测试人员会尝试了解：

- Web 目录和 API
- 用户名格式
- SMB 共享
- 域名和子域名
- DNS 信息
- SSL/TLS 证书
- 软件版本
- 登录页面
- 身份验证方式
- 文件和文件夹权限
- Active Directory 信息

例如发现 TCP 22 开放后，还需要确认：

- 是否允许密码登录
- 是否允许 root 直接登录
- 是否使用旧版本
- 是否对所有网络开放
- 登录失败是否有日志
- 是否配置账户锁定或速率限制

---

## 5. 漏洞分析

测试人员会把扫描结果与实际风险联系起来。

主要问题包括：

- 软件版本是否存在已知漏洞
- 配置是否错误
- 默认账户是否存在
- 是否使用弱密码
- 是否缺少安全更新
- 文件权限是否过宽
- 是否暴露不安全的服务
- 是否泄露敏感数据
- 是否缺少多因素认证

常见漏洞类型：

- 弱密码
- 默认凭据
- 未修补软件
- 错误权限
- SQL Injection
- Cross-Site Scripting
- Broken Access Control
- 不安全文件上传
- 信息泄露
- 错误的云权限
- 不必要的开放端口

---

## 6. 漏洞验证

自动扫描器可能产生误报，所以渗透测试人员需要手动验证。

例如扫描器报告：

> 服务器可能使用弱 TLS 配置。

测试人员需要进一步确认：

- 实际支持哪些 TLS 版本
- 是否启用了弱加密套件
- 是否真的可以被利用
- 成功利用后可能造成什么影响

验证的目标不是破坏系统，而是获取足够证据证明风险。

常见证据包括：

- 截图
- HTTP 请求和响应
- 系统日志
- 受影响端口
- 软件版本
- 可以读取到的测试文件
- 测试账户获得的权限

---

## 7. 有限制地利用漏洞

只有授权范围允许时，测试人员才会进行漏洞利用。

目的通常是回答：

> 这个漏洞到底可以造成多大影响？

示例：

```text
普通用户
→ 访问不属于自己的数据
→ 权限提升
→ 获得管理员权限
```

另一个示例：

```text
Web 漏洞
→ 读取测试数据库
→ 证明敏感信息可能泄露
```

正式测试中通常遵循：

- 使用测试账户
- 只读取证明风险所需的最少数据
- 不修改真实业务数据
- 不进行拒绝服务攻击
- 不长期保留访问权限
- 发现高危问题后及时报告
- 严格遵守交战规则

---

## 8. 权限提升（Privilege Escalation）

获得普通账户后，测试人员会检查是否可以提升权限。

常见检查内容：

- 普通 Linux 用户能否成为 root
- 普通 Windows 用户能否成为 Administrator
- 服务账户权限是否过大
- sudo 配置是否错误
- 计划任务是否不安全
- 系统文件是否可被普通用户修改
- 是否保存明文密码
- 服务配置或路径权限是否错误

---

## 9. 横向移动（Lateral Movement）

在企业测试中，测试人员进入一台机器后，可能尝试访问其他授权系统。

示例：

```text
员工电脑
→ 文件服务器
→ 应用服务器
→ 域控制器
```

主要检查：

- 密码是否重复使用
- 网络是否正确分段
- 普通用户是否能访问服务器网段
- 管理协议是否暴露
- 管理员凭据是否留在普通电脑
- 防火墙是否可以阻止横向访问

在家庭实验室中可以模拟基础流程：

```text
Kali
→ Ubuntu 靶机
→ 检查 Ubuntu 是否能够访问 Windows
→ Wazuh 观察日志和告警
```

开始学习时，应该先掌握扫描、日志和防火墙，再进入复杂的横向移动实验。

---

## 10. 检查检测能力

现代渗透测试不只问：

> 能不能进入系统？

还会问：

> 安全部门能不能发现？

需要观察：

- Windows Event Viewer 是否记录登录事件
- Linux 日志是否记录 SSH 失败
- Wazuh 是否触发告警
- 防火墙是否记录连接
- IDS/IPS 是否检测到扫描
- 安全团队是否响应异常活动

Home Lab 示例：

```text
Kali 发起 Nmap 扫描
        ↓
Ubuntu 记录网络或登录活动
        ↓
Wazuh 收集日志
        ↓
查看是否产生告警
```

---

## 11. 清理测试环境

测试结束后，需要恢复环境，包括：

- 删除测试账户
- 删除上传的测试文件
- 删除测试密钥
- 关闭临时开放的端口
- 恢复修改的配置
- 停止测试服务
- 清除测试创建的计划任务
- 向客户说明测试期间做过的修改

这里的清理不是为了隐藏攻击，而是避免给客户留下新的安全风险。

---

## 12. 编写渗透测试报告

报告通常是最重要的交付物。

### 12.1 Executive Summary

面向管理人员，使用容易理解的语言说明：

- 整体风险
- 最严重的问题
- 对业务可能造成的影响
- 应优先修复什么

### 12.2 Technical Findings

面向 IT 和安全人员。

| 项目 | 内容 |
|---|---|
| Finding | 漏洞名称 |
| Severity | Critical / High / Medium / Low |
| Affected Asset | 受影响的 IP、域名或系统 |
| Evidence | 截图、日志、请求和响应 |
| Impact | 攻击者可以做什么 |
| Reproduction | 如何在授权环境中重现 |
| Remediation | 如何修复 |
| References | CVE、厂商文档或安全标准 |

示例发现：

```text
问题：
SSH 允许整个实验网段访问。

证据：
10.10.20.20 的 TCP 22 端口对所有实验主机开放。

影响：
任何进入实验网段的设备都可以尝试登录 SSH。

建议：
只允许管理机 10.10.20.10 访问 TCP 22，
并禁用 root 密码登录。
```

---

## 13. 修复后重新测试

客户完成修复后，测试人员会再次检查。

第一次扫描：

```text
22/tcp open ssh
80/tcp open http
23/tcp open telnet
```

修复后：

```text
22/tcp filtered
80/tcp open
23/tcp closed
```

重新测试用于确认：

- 漏洞确实已经修复
- 没有引入新的问题
- 防火墙规则已经生效
- 风险已经降低

---

# 漏洞扫描和渗透测试的区别

| 类型 | 主要工作 |
|---|---|
| Vulnerability Scan | 使用工具自动查找可能存在的漏洞 |
| Vulnerability Assessment | 分析、确认并排列漏洞优先级 |
| Penetration Test | 在授权范围内验证漏洞是否可以利用 |
| Red Team | 模拟真实攻击者，测试人员、流程和技术 |
| Security Audit | 检查系统是否符合政策、标准或法规 |

扫描工具发现 100 个问题，并不代表已经完成渗透测试。

渗透测试人员还需要判断：

- 哪些是误报
- 哪些可以真正利用
- 哪些会造成实际业务影响
- 应该按照什么顺序修复

---

# 渗透测试人员的一天

一个典型工作日可能是：

```text
09:00  与客户确认测试范围
09:30  检查资产和扫描结果
10:30  手动验证高风险问题
12:00  整理证据和实验记录
13:00  测试 Web 登录和权限控制
15:00  查看日志和检测告警
16:00  报告紧急漏洞
16:30  编写技术发现和修复建议
```

渗透测试人员并不是八小时都在运行攻击工具。

---

# 适合 Home Lab 的完整练习流程

每次实验可以按照以下顺序：

```text
1. 创建快照
2. 确认测试范围
3. 主机发现
4. 端口扫描
5. 服务枚举
6. 手动验证
7. 查看 Windows/Linux 日志
8. 查看 Wazuh 告警
9. 配置防火墙或修复漏洞
10. 重新扫描
11. 编写一页报告
12. 恢复快照
```

---

# 第一个推荐项目：Ubuntu 安全评估

## 项目目标

对 Ubuntu 靶机 `10.10.20.20` 进行一次完整、可控的安全评估。

## 检查内容

- 哪些端口开放
- 运行了哪些服务
- SSH 是否安全
- Juice Shop 使用哪个端口
- UFW 是否开启
- 登录失败是否留下日志
- Wazuh 是否检测到扫描
- 修复后开放端口是否减少

## 建议流程

### 1. 确认授权范围

```text
Attacker：10.10.20.10
Target：10.10.20.20
Network：CyberLab 10.10.20.0/24
Allowed：扫描、服务枚举、日志检查、防火墙测试
Not Allowed：DoS、破坏数据、攻击真实网络
```

### 2. 主机发现

```bash
sudo nmap -sn 10.10.20.0/24
```

### 3. 基础端口扫描

```bash
nmap 10.10.20.20
```

### 4. 服务版本识别

```bash
sudo nmap -sV 10.10.20.20
```

### 5. 全端口扫描

```bash
sudo nmap -sS -p- 10.10.20.20
```

### 6. 检查目标日志

Ubuntu：

```bash
sudo journalctl -u ssh
```

```bash
sudo tail -f /var/log/auth.log
```

### 7. 查看防火墙

```bash
sudo ufw status verbose
```

### 8. 修复问题

示例：

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 10.10.20.10 to any port 22 proto tcp
sudo ufw enable
```

### 9. 重新扫描

```bash
sudo nmap -sV -p- 10.10.20.20
```

### 10. 编写报告

记录：

- 扫描结果
- 发现的问题
- 风险等级
- 证据
- 修复操作
- 修复后的验证结果
- 本次实验学到的内容

---

# 职业核心思维

真正的渗透测试人员不是只会“进入系统”的人。

他们必须做到：

> **有授权、控制风险、保护数据、保存证据、解释业务影响、提出可执行的修复方法。**

攻击工具只是工作的一部分。

真正有价值的能力是：

- 判断
- 验证
- 记录
- 沟通
- 报告
- 修复建议
- 重新测试
