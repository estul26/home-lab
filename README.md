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

# 6. 在 UTM 创建 CyberLab 网络

在 UTM 中打开：

```text
UTM
→ Settings
→ Network
→ Host Networks
→ Add
```

创建网络：

```text
Name：CyberLab
```

四台虚拟机必须连接到同一个 `CyberLab` 网络。

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