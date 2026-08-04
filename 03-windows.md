# UTM 安装 Windows 11 ARM64 并连接 CyberLab

## 1. 实验目标

在 Apple Silicon MacBook Pro 上，通过 UTM 创建一台 Windows 11 ARM64 虚拟机，并连接到隔离的 `CyberLab` 网络，作为 Windows 靶机。

最终配置：

```text
UTM 虚拟机名称：Windows-Target
Windows 主机名：windows-target
系统：Windows 11 Pro ARM64
虚拟化引擎：QEMU
CPU：6 Cores
RAM：12 GB
Storage：100 GB

Network Mode：Host Only
Host Network：CyberLab
IP：10.10.20.30/24
Default Gateway：无
DNS：无
Network Category：Private
```

当前 CyberLab：

```text
Kali-CyberLab    10.10.20.10
Ubuntu-Target    10.10.20.20
Windows-Target   10.10.20.30
```

---

## 2. 使用的安装镜像

使用：

```text
Win11_25H2_English_Arm64_v2.iso
```

Apple Silicon 应使用：

```text
ARM64
AArch64
```

不要使用：

```text
AMD64
x86_64
```

---

## 3. 创建 Windows 虚拟机

打开：

```text
UTM
→ 点击 +
→ Virtualize
→ Windows
```

设置：

```text
Use Apple Virtualization：不要勾选
Engine：QEMU
Boot from ISO：Win11_25H2_English_Arm64_v2.iso
Install Windows 10 or higher：勾选
Install drivers and SPICE tools：勾选
```

推荐资源：

```text
CPU：6 Cores
RAM：12 GB
Storage：100 GB
Architecture：ARM64 / aarch64
Name：Windows
```

Summary 页面勾选：

```text
Open VM Settings
```

然后点击 `Save`。

---

## 4. 连接 CyberLab

进入：

```text
Windows
→ Edit
→ Network
```

设置：

```text
Network Mode：Host Only
Host Network：CyberLab
Emulated Network Card：virtio-net-pci
MAC Address：保持自动生成
```

不要选择：

```text
Shared Network
Bridged Network
```

`CyberLab` 不提供 DHCP，因此 Windows 安装完成后需要手动设置固定 IP。

---

## 5. 进入 UEFI Shell 时手动启动安装程序

第一次启动时可能进入：

```text
UEFI Interactive Shell
Shell>
```

如果屏幕中显示 `FS0` 是 CDROM，输入：

```text
fs0:\efi\boot\bootaa64.efi
```

按 `Enter`。

如果出现：

```text
Press any key to boot from CD or DVD...
```

立即按一次空格键。

如果直接命令无法运行，依次输入：

```text
fs0:
cd EFI\BOOT
dir
BOOTAA64.EFI
```

---

## 6. Windows Setup

### Product Key

点击：

```text
I don’t have a product key
```

### Windows 版本

建议选择：

```text
Windows 11 Pro
```

适合练习：

```text
Local Security Policy
Group Policy
Remote Desktop
Local Users and Groups
Windows Defender Firewall
BitLocker
Event Viewer
```

### 安装类型

选择：

```text
Custom: Install Windows only
```

### 安装磁盘

选择：

```text
Disk 0 Unallocated Space
100.0 GB
```

直接点击 `Next`。不需要手动创建分区。

---

## 7. 安装重启注意事项

Windows 会自动重启多次。

第一次重启后：

```text
不要再按键从 CD/DVD 启动
```

让它自动从虚拟硬盘继续安装。

---

## 8. 创建本地账户

在：

```text
Who’s going to use this device?
```

输入：

```text
labuser
```

设置实验室密码并完成安全问题。

最终账户：

```text
Username：labuser
Password：自己设置的实验室密码
```

---

## 9. 检查 VirtIO 网卡

进入桌面后，以管理员身份打开 PowerShell：

```text
Start
→ 搜索 PowerShell
→ Run as administrator
```

运行：

```powershell
Get-NetAdapter
```

本次结果：

```text
Name：Ethernet
InterfaceDescription：Red Hat VirtIO Ethernet Adapter
ifIndex：3
Status：Up
```

说明 VirtIO 网卡驱动正常。

---

## 10. 安装 UTM Guest Tools

Guest Tools 用于：

```text
剪贴板复制粘贴
动态分辨率
SPICE 集成
其他虚拟机驱动
```

在 UTM 窗口点击光盘图标：

```text
Install Windows Guest Tools
```

Windows 中打开：

```text
This PC
→ CD Drive (D:) UTM Guest Tools
```

运行：

```text
utm-guest-tools-0.1.271
```

不同版本文件名可能不同。

安装步骤：

```text
双击安装程序
→ Yes
→ Next
→ Install
→ Finish
→ Restart Windows
```

---

## 11. 启用剪贴板共享

Windows 完全关机后：

```text
Windows
→ Edit
→ Sharing
→ Clipboard Sharing：开启
→ Save
```

剪贴板共享需要：

```text
UTM Clipboard Sharing 已开启
Windows Guest Tools 已安装
Windows 已重启
```

测试：

```text
Mac：Command + C
Windows：Ctrl + V
```

PowerShell 中也可以尝试鼠标右键粘贴。

---

## 12. 设置固定 IP

在管理员 PowerShell 中关闭 DHCP：

```powershell
Set-NetIPInterface `
  -InterfaceAlias "Ethernet" `
  -AddressFamily IPv4 `
  -Dhcp Disabled
```

如果存在 `169.254.x.x` 地址，删除它：

```powershell
Get-NetIPAddress `
  -InterfaceAlias "Ethernet" `
  -AddressFamily IPv4 |
Where-Object { $_.IPAddress -like "169.254.*" } |
Remove-NetIPAddress -Confirm:$false
```

设置固定 IP：

```powershell
New-NetIPAddress `
  -InterfaceAlias "Ethernet" `
  -IPAddress 10.10.20.30 `
  -PrefixLength 24
```

这里故意不设置：

```text
Default Gateway
DNS
```

---

## 13. 验证 IP

运行：

```powershell
ipconfig
```

正确结果：

```text
IPv4 Address：10.10.20.30
Subnet Mask：255.255.255.0
Default Gateway：空白
```

---

## 14. 修改网络类型为 Private

检查：

```powershell
Get-NetConnectionProfile
```

最初可能显示：

```text
Name：Unidentified network
InterfaceAlias：Ethernet
InterfaceIndex：3
NetworkCategory：Public
```

改为 Private：

```powershell
Set-NetConnectionProfile `
  -InterfaceIndex 3 `
  -NetworkCategory Private
```

也可以使用：

```powershell
Set-NetConnectionProfile `
  -InterfaceAlias "Ethernet" `
  -NetworkCategory Private
```

再次检查：

```powershell
Get-NetConnectionProfile
```

正确结果：

```text
NetworkCategory：Private
```

---

## 15. 修改计算机名称

```powershell
Rename-Computer -NewName "windows-target"
```

然后重启：

```powershell
Restart-Computer
```

重启后检查：

```powershell
hostname
```

正确结果：

```text
windows-target
```

---

## 16. 允许 CyberLab Ping Windows

先删除旧规则：

```powershell
Remove-NetFirewallRule `
  -DisplayName "Allow CyberLab ICMPv4" `
  -ErrorAction SilentlyContinue
```

创建新规则：

```powershell
New-NetFirewallRule `
  -DisplayName "Allow CyberLab ICMPv4" `
  -Direction Inbound `
  -Protocol ICMPv4 `
  -IcmpType 8 `
  -LocalAddress 10.10.20.30 `
  -RemoteAddress 10.10.20.0/24 `
  -Profile Any `
  -Action Allow
```

这条规则：

```text
只允许来自 CyberLab 10.10.20.0/24 的 Ping
不会关闭 Windows Defender Firewall
```

检查规则：

```powershell
Get-NetFirewallRule `
  -DisplayName "Allow CyberLab ICMPv4" |
Format-List DisplayName,Enabled,Profile,Direction,Action
```

预期：

```text
Enabled：True
Profile：Any
Direction：Inbound
Action：Allow
```

---

## 17. 从 Windows 测试其他虚拟机

确保 Kali 和 Ubuntu 已启动：

```powershell
ping 10.10.20.10
ping 10.10.20.20
```

---

## 18. 从 Kali 测试 Windows

```bash
ping -c 3 10.10.20.30
```

扫描：

```bash
sudo nmap -n -Pn -sV 10.10.20.30
```

参数：

```text
-n：不使用 DNS
-Pn：即使目标不回应 Ping，也继续扫描
-sV：识别服务版本
```

全新 Windows 没有开放 TCP 端口也可能是正常结果。

---

## 19. Kali 无法 Ping Windows 时的检查

### Windows IP

```powershell
ipconfig
```

确认：

```text
IPv4：10.10.20.30
Subnet Mask：255.255.255.0
Gateway：空白
```

### 网络类别

```powershell
Get-NetConnectionProfile
```

确认：

```text
NetworkCategory：Private
```

### 防火墙规则

```powershell
Get-NetFirewallRule `
  -DisplayName "Allow CyberLab ICMPv4"
```

### Kali 使用 ARP 发现

```bash
sudo nmap -n -sn -PR 10.10.20.30
```

如果显示：

```text
Host is up
```

说明 Windows 位于同一个 CyberLab，只是防火墙仍可能阻止 ICMP。

### UTM 网卡

关闭 Windows 后检查：

```text
Windows
→ Edit
→ Network
```

应为：

```text
Network Mode：Host Only
Host Network：CyberLab
Network Card：virtio-net-pci
```

---

## 20. 清除安装 ISO

Windows 稳定后正常关机：

```text
Start
→ Power
→ Shut down
```

进入：

```text
Windows
→ Edit
→ Drives
```

清除：

```text
Win11_25H2_English_Arm64_v2.iso
UTM Guest Tools ISO（可选）
```

点击：

```text
Clear
```

不要删除 VirtIO 系统硬盘。

---

## 21. 重命名 UTM 虚拟机

将 UTM 虚拟机名称改为：

```text
Windows-Target
```

区别：

```text
UTM VM Name：Windows-Target
Windows Hostname：windows-target
```

---

## 22. 创建干净备份

确认 Windows 已关闭：

```text
右键 Windows-Target
→ Clone…
```

命名：

```text
01-Windows-Clean
```

不要同时启动原机和克隆机，因为它们可能拥有相同的：

```text
IP 地址
MAC 地址
Hostname
```

---

## 23. 临时实验模式

UTM 右键菜单：

```text
Run without saving changes
```

推荐：

```text
普通学习：Run
临时实验：Run without saving changes
长期备份：Clone
```

---

## 24. 最终配置摘要

```text
UTM VM Name：Windows-Target
Windows Hostname：windows-target
Username：labuser

OS：Windows 11 Pro ARM64
Engine：QEMU
Architecture：ARM64 / aarch64
CPU：6 Cores
RAM：12 GB
Storage：100 GB

Network Mode：Host Only
Host Network：CyberLab
Network Card：virtio-net-pci
Adapter：Red Hat VirtIO Ethernet Adapter

IPv4：10.10.20.30
Subnet Mask：255.255.255.0
Default Gateway：无
DNS：无
Network Category：Private

Windows Defender Firewall：保持开启
ICMPv4：只允许 CyberLab 网段
UTM Guest Tools：已安装
Clipboard Sharing：已启用
```

---

## 25. CyberLab 拓扑

```text
                         CyberLab
                     10.10.20.0/24
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
Kali-CyberLab          Ubuntu-Target          Windows-Target
10.10.20.10            10.10.20.20           10.10.20.30
攻击测试机              Linux 靶机             Windows 靶机
```

---

## 26. 常用命令

### Windows

```powershell
ipconfig
Get-NetAdapter
Get-NetConnectionProfile
hostname
ping 10.10.20.10
ping 10.10.20.20
```

### Kali

```bash
ping -c 3 10.10.20.30
sudo nmap -n -Pn -sV 10.10.20.30
sudo nmap -n -sn -PR 10.10.20.30
```

---

## 27. 下一步

Windows 配置完成后，可以继续：

```text
1. 从 Kali 扫描 Windows
2. 查看 Windows Defender Firewall 日志
3. 查看 Windows Event Viewer
4. 启用测试服务并重新扫描
5. 安装 Wazuh Agent
6. 将 Windows 日志发送到 Wazuh-SIEM
```

下一台虚拟机建议：

```text
Wazuh-SIEM
IP：10.10.20.40/24
```

---

## 28. 核心注意事项

1. Apple Silicon 必须使用 Windows 11 ARM64 ISO。
2. 使用 QEMU，不勾选 Apple Virtualization。
3. Windows 网卡连接 `CyberLab` Host Only。
4. Windows 使用 `10.10.20.30/24`。
5. Gateway 和 DNS 留空。
6. UEFI Shell 中可运行 `bootaa64.efi`。
7. 安装重启后不要再次从 ISO 启动。
8. 安装 UTM Guest Tools 才能使用剪贴板共享。
9. 不要为了 Ping 直接关闭 Windows Firewall。
10. 只开放 CyberLab 所需的测试规则。
11. 不要同时启动 IP 相同的克隆虚拟机。
12. 只测试自己拥有或明确授权的系统。
