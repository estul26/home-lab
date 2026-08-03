# UTM 安装 Ubuntu Server ARM64 并连接 CyberLab

## 1. 实验目标

在 Apple Silicon MacBook Pro 上，通过 UTM 创建一台 Ubuntu Server ARM64 虚拟机，并将它连接到隔离的 `CyberLab` 网络，作为 Kali Linux 的 Linux 靶机。

最终配置：

```text
虚拟机名称：Ubuntu-Target
架构：ARM64 / aarch64
虚拟化软件：UTM
虚拟化引擎：QEMU
网络：CyberLab Host Only
固定 IP：10.10.20.20/24
默认网关：无
DNS：无
SSH：启用
```

实验网络关系：

```text
Kali-CyberLab       10.10.20.10
        │
        │ CyberLab 10.10.20.0/24
        │
Ubuntu-Target       10.10.20.20
```

---

## 2. 使用的安装镜像

使用 ARM64 版本：

```text
ubuntu-22.04.5-live-server-arm64.iso
```

Apple Silicon 应使用：

```text
arm64
aarch64
```

不要使用：

```text
amd64
x86_64
```

---

## 3. 创建 Ubuntu 虚拟机

打开 UTM 主界面：

```text
点击 +
→ Virtualize
→ Linux
```

在 Linux 创建页面：

```text
Use Apple Virtualization：不要勾选
Boot Image Type：Boot from ISO image
```

选择 ISO：

```text
ubuntu-22.04.5-live-server-arm64.iso
```

推荐资源：

```text
Engine：QEMU
Use Virtualization：启用
Architecture：ARM64 (aarch64)
CPU：4 Cores
RAM：6 GB
Storage：64 GB
Operating System：Linux
```

虚拟机名称：

```text
Ubuntu-Target
```

在 Summary 页面勾选：

```text
Open VM Settings
```

然后点击 `Save`。
<img width="900" height="1030" alt="iScreen Shoter - UTM - 260803105700" src="https://github.com/user-attachments/assets/a09e3212-5c4c-4c71-882d-f6d478bac68a" />

---

## 4. 连接 CyberLab 隔离网络

进入虚拟机设置：

```text
Ubuntu-Target
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

`CyberLab` 不提供 DHCP，因此 Ubuntu 必须手动配置固定 IP。
<img width="1600" height="800" alt="image" src="https://github.com/user-attachments/assets/38b5ff0e-fa40-49fd-a06a-e787cddb42bf" />

---

## 5. 启动 Ubuntu Server 安装程序

启动：

```text
Ubuntu-Target
```

选择：

```text
Try or Install Ubuntu Server
```

推荐设置：

```text
Language：English
Keyboard：English (US)
Installation type：Ubuntu Server
```

Ubuntu Server 默认没有图形桌面，安装完成后会进入命令行界面。
<img width="2560" height="1680" alt="image" src="https://github.com/user-attachments/assets/53184f99-dbc8-4680-9f9a-dfe08711089f" />

---

## 6. 配置固定 IP

安装程序进入：

```text
Network configuration
```

因为 `CyberLab` 没有 DHCP，网卡会显示：

```text
autoconfiguration failed
```

本次网卡名称为：

```text
enp0s1
```

选择 `enp0s1`，按 `Enter`，选择：

```text
Edit IPv4
```

把 IPv4 Method 改为：

```text
Manual
```

填写：

```text
Subnet：10.10.20.0/24
Address：10.10.20.20
Gateway：留空
Name Servers：留空
Search domains：留空
```

选择 `Save`。

保存后，网络页面应显示：

```text
enp0s1
static 10.10.20.20/24
```

因为没有网关和 DNS，底部可能仍显示：

```text
Continue without network
```
<img width="2560" height="1680" alt="image" src="https://github.com/user-attachments/assets/dde02053-107e-45b9-9a04-0a6788726621" />

这是正常的。选择它继续安装。

---

## 7. 存储配置

进入：

```text
Guided storage configuration
```

保持：

```text
(X) Use an entire disk
```

确认磁盘为：

```text
64.000 GB local disk
```

取消：

```text
[ ] Set up this disk as an LVM group
```

保持不选：

```text
[ ] Encrypt the LVM group with LUKS
```

选择：

```text
Done
```
<img width="2560" height="1680" alt="image" src="https://github.com/user-attachments/assets/d9893840-c288-4efe-8dbe-dac79a684da7" />

下一页再次选择 `Done`。出现确认警告时选择：

```text
Continue
```

这里只会格式化 `Ubuntu-Target` 的虚拟硬盘，不会影响 MacBook 或 Kali。

---

## 8. 创建用户账户

进入：

```text
Profile configuration
```

填写：

```text
Your name：Cyber Lab User
Your server's name：ubuntu-target
Pick a username：labuser
Choose a password：设置自己的实验室密码
Confirm your password：再次输入相同密码
```

说明：

```text
Your name：显示名称
Server name：主机名
Username：以后登录和 SSH 使用的用户名
```

主机名建议使用小写字母和连字符，不要包含空格。

填写完成后选择：

```text
Done
```
<img width="2560" height="1680" alt="image" src="https://github.com/user-attachments/assets/a8c6d59d-e768-4656-861d-71ef9a2be657" />


---

## 9. 跳过 Ubuntu Pro

进入：

```text
Upgrade to Ubuntu Pro
```

保持：

```text
(X) Skip Ubuntu Pro setup for now
```

然后选择：

```text
Continue
```

当前隔离实验室不需要 Ubuntu Pro。

---

## 10. 安装 OpenSSH Server

进入：

```text
SSH configuration
```

选择：

```text
[X] Install OpenSSH server
[X] Allow password authentication over SSH
[ ] Import SSH key
```

然后选择：

```text
Done
```

<img width="2560" height="1680" alt="image" src="https://github.com/user-attachments/assets/294cf857-dfe3-488c-926e-709ca4286aee" />

<img width="2560" height="1680" alt="image" src="https://github.com/user-attachments/assets/8beea289-b02c-48d6-b324-5de7d0401d12" />


安装完成后，可以从 Kali 通过 SSH 连接：

```bash
ssh labuser@10.10.20.20
```

第一次连接时输入：

```text
yes
```

然后输入 `labuser` 的密码。

---

## 11. 等待安装完成

安装程序会执行：

```text
安装基础系统
安装 Linux 内核
配置 GRUB
写入网络配置
安装 OpenSSH Server
创建用户账户
```

完成后会显示：

```text
Installation complete
```

底部会出现：

```text
Reboot Now
```
<img width="1600" height="1280" alt="image" src="https://github.com/user-attachments/assets/f81769ed-5e22-4b96-9b2c-f5c87c872fbd" />
<img width="1600" height="1280" alt="image" src="https://github.com/user-attachments/assets/9eee15e6-e553-4319-8030-25ceacb2ad56" />
<img width="2560" height="1680" alt="image" src="https://github.com/user-attachments/assets/c1314583-4454-494e-8e8b-f46e603f7b3d" />




---

## 12. 移除 Ubuntu 安装 ISO

在重启前，建议先卸载安装 ISO。

可以使用 UTM 工具栏的光盘图标弹出 ISO，或者关闭虚拟机后进入：

```text
Ubuntu-Target
→ Edit
→ Drives
→ 选择装载 Ubuntu ISO 的驱动器
```

找到：

```text
ubuntu-22.04.5-live-server-arm64.iso
```

点击：

```text
Clear
```

不要删除：

```text
VirtIO Drive
```

`VirtIO Drive` 是已经安装好 Ubuntu 的系统虚拟硬盘。

---

## 13. 第一次启动 Ubuntu

移除 ISO 后启动：

```text
Ubuntu-Target
```

正常情况下会出现：

```text
ubuntu-target login:
```

输入：

```text
Username：labuser
Password：安装时设置的密码
```

输入密码时屏幕不会显示字符，这是 Linux 的正常行为。

---

## 14. 验证 Ubuntu 网络配置

登录后运行：

```bash
ip -br address
```

正确结果应包含：

```text
enp0s1    UP    10.10.20.20/24
```

检查路由：

```bash
ip route
```

正确结果类似：

```text
10.10.20.0/24 dev enp0s1 proto kernel scope link src 10.10.20.20
```

不应该出现：

```text
default via ...
```

---

## 15. 验证 SSH 服务

检查 SSH：

```bash
systemctl status ssh
```

应该看到：

```text
active (running)
```

也可以运行：

```bash
sudo systemctl is-active ssh
```

正确结果：

```text
active
```

查看 TCP 22：

```bash
sudo ss -tulpn | grep :22
``\
---

## 16. 从 Ubuntu 测试 Kali

确保 Kali 已启动，然后在 Ubuntu 中执行：

```bash
ping -c 3 10.10.20.10
```
<img width="2560" height="1680" alt="image" src="https://github.com/user-attachments/assets/21b2f299-46e7-4ef2-82bb-1b6acddc8dbc" />

---

## 17. 从 Kali 测试 Ubuntu

在 Kali 中执行：

```bash
ping -c 3 10.10.20.20
```

扫描 Ubuntu：

```bash
sudo nmap -sV 10.10.20.20
```

预期至少发现：

```text
22/tcp open ssh
```

SSH 登录：

```bash
ssh labuser@10.10.20.20
```

退出 SSH：

```bash
exit
```
<img width="2560" height="1680" alt="image" src="https://github.com/user-attachments/assets/ef33465b-4d6e-4e10-b040-33b31c49e164" />

---

## 18. 验证互联网隔离

在 Ubuntu 中运行：

```bash
ping -c 3 1.1.1.1
```

正常情况下应失败，例如：

```text
Network is unreachable
```

这表示 Ubuntu 可以与 `CyberLab` 内的虚拟机通信，但不能访问互联网或家庭局域网。

---

## 19. 正常关闭 Ubuntu

使用：

```bash
sudo poweroff
```

等待 UTM 显示：

```text
Stopped
```

---

## 20. 创建干净备份

确保 Ubuntu 已关闭：

```text
右键 Ubuntu-Target
→ Clone…
```

名称填写：

```text
01-Ubuntu-Clean
```

用途：

```text
Ubuntu-Target：日常实验
01-Ubuntu-Clean：干净备份
```

不要同时启动原机和克隆机，因为它们可能拥有相同的 IP、MAC 地址和主机名。

---

## 21. 临时实验模式

右键 Ubuntu 虚拟机，可以选择：

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

## 22. Ubuntu 最终配置摘要

```text
虚拟机名称：Ubuntu-Target
Engine：QEMU
Virtualization：Enabled
Architecture：ARM64 (aarch64)
CPU：4 Cores
RAM：6 GB
Storage：64 GB
Operating System：Ubuntu Server 22.04.5 ARM64

Network Mode：Host Only
Host Network：CyberLab
Network Card：virtio-net-pci

Interface：enp0s1
IP：10.10.20.20/24
Subnet：10.10.20.0/24
Gateway：无
DNS：无
Internet：默认关闭

Hostname：ubuntu-target
Username：labuser
SSH Server：Enabled
SSH Port：22
Password Authentication：Enabled
```

---

## 23. 当前 CyberLab 设备

```text
CyberLab：10.10.20.0/24

Kali-CyberLab：
10.10.20.10

Ubuntu-Target：
10.10.20.20
```

常用测试命令：

```bash
# Kali 测试 Ubuntu
ping -c 3 10.10.20.20

# Ubuntu 测试 Kali
ping -c 3 10.10.20.10

# Kali 扫描 Ubuntu
sudo nmap -sV 10.10.20.20

# Kali SSH 登录 Ubuntu
ssh labuser@10.10.20.20
```

---

## 24. 第一个推荐实验

```text
1. 从 Kali Ping Ubuntu
2. 使用 Nmap 扫描 Ubuntu
3. 确认 TCP 22 开放
4. 使用 SSH 登录 Ubuntu
5. 在 Ubuntu 查看 SSH 登录日志
6. 配置 UFW 防火墙
7. 从 Kali 重新扫描
8. 比较修复前后的结果
```

Ubuntu 查看 SSH 日志：

```bash
sudo journalctl -u ssh
```

或者：

```bash
sudo tail -f /var/log/auth.log
```

查看防火墙：

```bash
sudo ufw status verbose
```

---

## 25. 核心注意事项

1. Apple Silicon 必须使用 ARM64 / AArch64 ISO。
2. 使用 QEMU，不勾选 Apple Virtualization。
3. 网卡连接 `CyberLab` Host Only。
4. CyberLab 不提供 DHCP，需要手动设置固定 IP。
5. Ubuntu 使用 `10.10.20.20/24`。
6. Gateway、DNS 和 Search domains 留空。
7. 安装 OpenSSH Server，方便 Kali 远程登录。
8. 安装完成后必须清除 ISO。
9. 不要删除 VirtIO Drive。
10. 不要同时启动 IP 相同的克隆虚拟机。
11. 只扫描和测试自己拥有或明确授权的目标。
