# UTM 安装 Kali Linux ARM64 并连接 CyberLab

## 1. 实验目标

在配备 Apple Silicon 芯片的 MacBook Pro 上，通过 UTM 创建一台 Kali Linux ARM64 虚拟机，并将它连接到隔离的 `CyberLab` 网络。

最终配置：

```text
虚拟机名称：Kali-CyberLab
架构：ARM64 / aarch64
虚拟化软件：UTM
虚拟化引擎：QEMU
网络：CyberLab Host Only
固定 IP：10.10.20.10/24
默认网关：无
DNS：无
```

---

## 2. 使用的安装镜像

使用 ARM64 版本：

```text
kali-linux-2026.2-installer-arm64.iso
```

不要使用：

```text
amd64
x86_64
```

Apple Silicon 应使用：

```text
arm64
aarch64
```

---

## 3. 创建 CyberLab 隔离网络

打开 UTM：

```text
UTM
→ Settings
→ Network
→ Host Networks
→ Add
```

网络名称填写：

```text
CyberLab
```

UTM 会显示一个 UUID。

该 Host Network 不提供 DHCP，因此虚拟机需要手动配置固定 IP。

---

## 4. 创建 Kali 虚拟机

打开 UTM 主界面：

```text
点击 +
→ Virtualize
→ Linux
```

选择 Kali ISO：

```text
kali-linux-2026.2-installer-arm64.iso
```

推荐资源：

```text
Engine：QEMU
Use Virtualization：启用
Architecture：ARM64 (aarch64)
CPU：6 Cores
RAM：8 GB
Storage：64 GB
Operating System：Linux
```

虚拟机名称：

```text
Kali-CyberLab
```

在 Summary 页面勾选：

```text
Open VM Settings
```

然后点击：

```text
Save
```

---

## 5. 配置 CyberLab 网卡

进入虚拟机设置：

```text
Kali-CyberLab
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

UTM 会显示：

```text
No DHCP will be provided by UTM
```

这是正常提示。

不要为实验网卡选择：

```text
Shared Network
Bridged Network
```

---

## 6. 第一次启动与黑屏处理

启动 Kali 后，在 GRUB 菜单中可能看到：

```text
Install
Graphical install
Advanced options
```

在当前 UTM + Kali ARM64 环境中，`Graphical install` 可能出现黑屏。

如果黑屏超过几分钟，不需要继续等待。

### 6.1 强制关闭虚拟机

点击 UTM 窗口的电源按钮：

```text
Power
→ Stop / Force Stop
```

### 6.2 添加 Serial 设备

确认虚拟机已经完全关闭：

```text
Kali-CyberLab
→ Edit
→ New…
→ Serial
```

设置：

```text
Mode：Built-in Terminal
Target：Automatic Serial Device (max 4)
Theme：Default
```

保存设置。

### 6.3 使用文本安装模式

重新启动 Kali。

在 GRUB 菜单选择：

```text
Install
```

不要选择：

```text
Graphical install
```

安装过程会在 UTM 的 Terminal / Serial 窗口中显示。

---

## 7. 安装程序设置

### 7.1 语言和地区

推荐：

```text
Language：English
Location：Canada
Keyboard：American English
```

### 7.2 网络配置

因为 CyberLab 没有 DHCP，安装程序会显示：

```text
Network autoconfiguration failed
```

点击：

```text
Continue
```

然后选择：

```text
Do not configure the network at this time
```

不要反复选择 DHCP 重试。

### 7.3 主机名

填写：

```text
Hostname：kali
```

Domain name 留空。

### 7.4 用户账户

示例：

```text
Full name：Cyber Lab User
Username：kali
Password：设置自己的实验室密码
```

说明：

```text
Full name：显示名称，可以有空格
Username：登录用户名，建议小写且不要有空格
```

---

## 8. 磁盘分区

在分区方式页面选择：

```text
Guided – use entire disk
```

它只会使用 Kali 的虚拟硬盘，不会影响 MacBook 的真实硬盘。

选择虚拟硬盘：

```text
Virtual disk 1 (vda)
Virtio Block Device
约 68.7 GB
```

分区方案选择：

```text
All files in one partition
```

然后选择：

```text
Finish partitioning and write changes to disk
```

确认：

```text
Write the changes to disks? → Yes
```

不需要使用：

```text
Encrypted LVM
Manual partitioning
Separate /home partition
```

---

## 9. 软件选择

保留默认桌面：

```text
Xfce
```

建议保留：

```text
Kali default desktop
Default recommended tools
Top 10 tools（如果默认已选择）
```

不需要选择：

```text
GNOME
KDE Plasma
Large
Everything
```

选择完成后：

```text
Tab
→ Continue
→ Enter
```

等待系统安装完成。

---

## 10. 安装完成后的处理

当安装程序显示：

```text
Installation complete
```

不要让它再次从 ISO 启动。

### 10.1 清除 Kali 安装 ISO

关闭虚拟机或进入虚拟机设置：

```text
Kali-CyberLab
→ Edit
→ Drives
→ USB Drive
```

找到：

```text
kali-linux-2026.2-installer-arm64.iso
```

点击：

```text
Clear
```

注意：

```text
不要删除 VirtIO Drive
```

`VirtIO Drive` 是已经安装好 Kali 的虚拟硬盘。

### 10.2 删除 Serial 设备

进入：

```text
Edit
→ Serial
→ Remove
```

Serial 仅用于避免安装程序黑屏。

### 10.3 设置显示设备

进入：

```text
Edit
→ Display
```

选择：

```text
Emulated Display Card：virtio-gpu-pci
```

保存设置。

---

## 11. 第一次进入 Kali

启动：

```text
Kali-CyberLab
```

使用安装时创建的用户名和密码登录。

macOS 可能询问：

```text
“UTM” would like to access the Microphone
```

网络安全实验不需要麦克风，可以选择：

```text
Don’t Allow
```

---

## 12. 设置 Kali 固定 IP

打开 Kali 终端。

### 12.1 查看网卡和连接名称

```bash
nmcli device status
```

```bash
nmcli connection show
```

本次安装显示：

```text
网卡：eth0
连接名称：Wired connection 1
```

### 12.2 配置固定 IP

执行：

```bash
sudo nmcli connection modify "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses 10.10.20.10/24 \
  ipv4.gateway "" \
  ipv4.dns "" \
  ipv4.never-default yes \
  ipv6.method disabled
```

重新启用连接：

```bash
sudo nmcli connection down "Wired connection 1"
```

```bash
sudo nmcli connection up "Wired connection 1"
```

如果 `down` 提示连接没有激活，可以继续执行 `up`。

---

## 13. 验证网络配置

### 13.1 检查 IP

```bash
ip -br address
```

正确结果：

```text
eth0    UP    10.10.20.10/24
```

### 13.2 检查路由

```bash
ip route
```

正确结果类似：

```text
10.10.20.0/24 dev eth0 proto kernel scope link src 10.10.20.10 metric 100
```

不应该出现：

```text
default via ...
```

### 13.3 检查 NetworkManager

```bash
nmcli device status
```

正确状态：

```text
eth0    ethernet    connected    Wired connection 1
```

### 13.4 验证互联网隔离

```bash
ping -c 3 1.1.1.1
```

预期失败：

```text
Network is unreachable
```

这表示 Kali 没有默认网关，无法通过 CyberLab 网卡访问互联网。

---

## 14. 关闭 Kali

建议通过系统正常关机：

```bash
sudo poweroff
```

等待 UTM 显示：

```text
Stopped
```

不要在 Kali 正在运行时直接关闭 UTM。

---

## 15. 创建干净备份

当前 UTM 界面没有传统 Snapshot 按钮，可以通过克隆保存干净状态。

确保 Kali 已关机：

```text
右键 Kali-CyberLab
→ Clone…
```

名称填写：

```text
01-Kali-Clean
```

建议用途：

```text
Kali-CyberLab：平时学习使用
01-Kali-Clean：干净备份，不要日常启动
```

因为克隆机可能与原机使用相同 IP，所以不要同时启动两台。

---

## 16. 临时实验模式

UTM 右键菜单中：

```text
Run without saving changes
```

可以用于一次性实验。

在这种模式中：

```text
启动时读取当前虚拟硬盘
本次运行产生的磁盘修改不会永久保存
关闭后恢复到运行前状态
```

推荐：

```text
正常学习：Run
临时危险实验：Run without saving changes
长期备份：Clone
```

---

## 17. Kali 最终配置摘要

```text
虚拟机名称：Kali-CyberLab
Engine：QEMU
Virtualization：Enabled
Architecture：ARM64 (aarch64)
CPU：6 Cores
RAM：8 GB
Storage：64 GB
Desktop：Xfce

Network Mode：Host Only
Host Network：CyberLab
Network Card：virtio-net-pci

Interface：eth0
Connection：Wired connection 1
IP：10.10.20.10/24
Gateway：无
DNS：无
IPv6：Disabled
Internet：默认关闭
```

---

## 18. 下一台虚拟机

下一步创建 Ubuntu Server 靶机：

```text
名称：Ubuntu-Target
架构：ARM64
CPU：4 Cores
RAM：6 GB
Storage：50–64 GB
Network：CyberLab Host Only
IP：10.10.20.20/24
Gateway：无
DNS：无
```

Ubuntu 配置完成后，从 Kali 测试：

```bash
ping -c 3 10.10.20.20
```

主机发现：

```bash
sudo nmap -sn 10.10.20.0/24
```

服务扫描：

```bash
sudo nmap -sV 10.10.20.20
```

---

## 19. 核心注意事项

1. Apple Silicon 必须使用 ARM64 / AArch64 镜像。
2. CyberLab 使用 Host Only，不使用 Bridged。
3. CyberLab 不提供 DHCP，需要固定 IP。
4. Kali 安装程序黑屏时，添加 Serial 并使用 `Install`。
5. 安装完成后清除 ISO、删除 Serial、设置 `virtio-gpu-pci`。
6. CyberLab 网卡不设置网关和 DNS。
7. 不要同时启动 IP 相同的克隆虚拟机。
8. 只扫描和测试自己拥有或明确授权的目标。


<img width="1800" height="900" alt="image" src="https://github.com/user-attachments/assets/5e74b1c9-704c-449c-bbfa-fe2efb62743c" />

<img width="900" height="1030" alt="image" src="https://github.com/user-attachments/assets/3efab7a7-12a9-46a4-8bdc-cf3d1c191db0" />

<img width="1600" height="800" alt="image" src="https://github.com/user-attachments/assets/f8d87124-b375-4aca-b2a3-61e75105bc95" />

<img width="1600" height="1280" alt="image" src="https://github.com/user-attachments/assets/8b6fbf5e-a1c4-4295-a9cc-fbd59b06e8b8" />

<img width="1600" height="800" alt="image" src="https://github.com/user-attachments/assets/6044551b-93a5-4500-92d2-1202f409dd0f" />

<img width="1190" height="752" alt="image" src="https://github.com/user-attachments/assets/fb8fb1d4-da5a-4623-b407-8dcbf5bdec10" />

<img width="1190" height="752" alt="image" src="https://github.com/user-attachments/assets/46416ecc-7d24-4e77-b108-ee0e20eeed53" />

<img width="1190" height="752" alt="image" src="https://github.com/user-attachments/assets/095f56b9-0f60-4b45-90c2-f1b1d040fbe7" />

<img width="1190" height="752" alt="image" src="https://github.com/user-attachments/assets/763f3c56-ab32-42a6-b539-e4749a9985fa" />

<img width="1600" height="800" alt="image" src="https://github.com/user-attachments/assets/49745792-e870-47ff-ad2a-554cc62354f2" />

<img width="2048" height="1344" alt="image" src="https://github.com/user-attachments/assets/d7c8de76-4763-4f41-a9f2-2c7ab659688c" />

<img width="1268" height="569" alt="image" src="https://github.com/user-attachments/assets/f3f86300-e6df-4507-937f-9ec83571efe7" />







