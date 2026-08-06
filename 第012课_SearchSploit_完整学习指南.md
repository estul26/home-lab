# 第十二课：SearchSploit 完整学习指南

> **实验环境：你自己的 CyberLab**
>
> 本课只学习如何离线搜索、筛选和静态审查公开 Exploit / PoC。  
> 不执行未知 Exploit，不对学校、公司、公共网站、随机公网 IP 或任何未经明确授权的系统进行利用。

---

# 1. 本课目标

完成本课后，你应该能够：

```text
理解 Exploit-DB 与 SearchSploit 的关系
理解 Exploit、PoC、Advisory、DoS、Shellcode 和 Metasploit Module
安装、检查和更新本地 Exploit-DB 数据库
使用软件名称、版本、平台、漏洞类型和 CVE 搜索
理解 SearchSploit 默认 AND 搜索和版本范围匹配
使用 Title、Exact、Strict、Case 和 Exclude 缩小结果
阅读 Exploit Title、EDB-ID、Path、Platform 和 Type
使用 JSON 输出并通过 jq 处理
查看本地完整路径
把 Exploit 文件复制到独立研究目录
使用 PAGER 静态阅读代码
对未知 PoC 进行安全的静态审查
判断软件版本、平台、架构和漏洞条件是否匹配
使用 Nmap XML 与 SearchSploit 联动
理解“有 Exploit”不代表“目标可利用”
理解 SearchSploit 搜索本身为什么不会扫描目标
使用 Wazuh 标记研究活动并理解可见性限制
完成一份 Public Exploit Research 报告
```

CyberLab：

```text
Kali-CyberLab      10.10.20.10
Ubuntu-Target      10.10.20.20
Windows-Target     10.10.20.30
Wazuh-SIEM         10.10.20.40
Network            10.10.20.0/24
```

本课主要使用：

```text
Kali：SearchSploit、Nmap、静态代码审查
Ubuntu：提供授权服务版本信息和日志标记
Wazuh：查看手工实验标记
```

---

# 2. 什么是 Exploit-DB？

Exploit-DB 是公开 Exploit 和 Proof-of-Concept 的档案库。

它包含的内容可能包括：

- Exploit 源代码
- Proof-of-Concept
- Local Exploit
- Remote Exploit
- Web Application Exploit
- Denial of Service
- Shellcode
- 研究资料
- 与 CVE 对应的公开技术样例

重要：

```text
Exploit-DB 更接近 Exploit / PoC Repository
不是完整 Vulnerability Advisory 数据库
```

因此每条记录不一定包含：

- 完整风险评级
- 厂商修复说明
- 所有受影响版本
- 完整缓解措施
- 当前补丁状态
- 生产环境适用性

---

# 3. 什么是 SearchSploit？

SearchSploit 是 Exploit-DB 的命令行离线搜索工具。

它使用 Kali 本地数据库搜索：

```text
Exploit
Shellcode
可选 Papers
```

主要优点：

- 不需要访问 Exploit-DB 网站
- 可在隔离网络中使用
- 搜索速度快
- 可使用多个关键词
- 可按 CVE 搜索
- 可显示本地文件路径
- 可复制 PoC 到研究目录
- 可直接静态阅读代码
- 可处理 Nmap XML

核心理解：

```text
SearchSploit 搜索的是本地文件和 CSV 索引
它本身不是网络扫描器
也不会自动利用目标
```

---

# 4. SearchSploit 与在线 Exploit-DB

| SearchSploit | Exploit-DB 网站 |
|---|---|
| 本地离线搜索 | 在线搜索 |
| 使用本地数据库版本 | 使用网站当前数据库 |
| 适合隔离实验室 | 需要互联网 |
| 可直接访问本地代码 | 通过网页查看和下载 |
| 结果可能落后于网站 | 通常更及时 |
| 支持 `--nmap` | 更适合人工网页研究 |

本地数据库是否新鲜取决于：

```text
Kali 软件包更新时间
最近一次 apt 更新
最近一次 searchsploit -u
```

---

# 5. 与前面工具的关系

```text
Nmap：发现软件和服务版本
WhatWeb：识别 Web 技术
Nikto：发现服务器配置和版本线索
Burp/ZAP：分析 Web 请求和 Alert
SQLmap：确认 SQL Injection
SearchSploit：根据产品、版本和 CVE 搜索公开 PoC
Metasploit：结构化使用模块进行授权验证
Wazuh：观察目标或端点日志
```

推荐研究流程：

```text
发现产品和版本
→ 确认版本证据
→ 查找 CVE
→ SearchSploit 搜索
→ 静态审查 PoC
→ 判断前置条件
→ 建立安全靶场
→ 获得授权后再决定是否验证
```

本课只做到：

```text
静态审查和适用性判断
```

---

# 6. 术语区别

## Vulnerability

软件、配置或设计中的安全弱点。

## Exploit

利用某个弱点产生安全影响的代码或方法。

## Proof-of-Concept

证明问题存在的最小技术样例。

PoC 不一定：

- 稳定
- 完整
- 安全
- 适合所有版本
- 适合生产环境

## Advisory

描述漏洞、影响、版本和修复建议的安全公告。

## Shellcode

用于在目标环境中执行特定动作的机器代码片段。

## Metasploit Module

按照 Metasploit Framework 结构编写的模块。

---

# 7. Exploit 不等于 Payload

Exploit：

```text
利用漏洞获得某种能力
```

Payload：

```text
漏洞利用成功后执行的动作
```

例如：

```text
Exploit：触发内存损坏
Payload：打开测试会话或执行受控操作
```

SearchSploit 中有些代码只包含：

- Crash PoC
- Exploit Logic
- Shellcode Placeholder
- Hard-coded Payload
- Metasploit Module

必须阅读代码后才能判断。

---

# 8. Local、Remote、WebApps 和 DoS

## Local

攻击者通常已经能在目标系统执行本地代码。

可能目标：

- Privilege Escalation
- Kernel Vulnerability
- Local Service
- File Permission

## Remote

可通过网络服务触发。

仍可能要求：

- 认证
- 特定协议
- 特定配置
- 可达端口
- 特定架构

## WebApps

针对 CMS、Plugin、Web Framework 或 Web Application。

## DoS

目标主要是：

```text
使服务崩溃、冻结或不可用
```

本课程不执行 DoS PoC。

---

# 9. EDB-ID、CVE 和厂商编号

## EDB-ID

Exploit-DB 记录编号。

例如格式：

```text
EDB-ID: 12345
```

## CVE

公开漏洞标识符：

```text
CVE-YYYY-NNNN
```

## 厂商编号

可能包括：

- Microsoft Security Bulletin
- Cisco Advisory ID
- Red Hat Advisory
- Ubuntu USN
- GitHub Security Advisory

关系不一定是一对一：

```text
一个 CVE 可能有多个 EDB-ID
一个 Exploit 可能覆盖多个 CVE
部分 Exploit 可能没有 CVE
```

---

# 10. 检查和安装 SearchSploit

Kali：

```bash
which searchsploit
```

查看帮助：

```bash
searchsploit -h
```

检查 Exploit-DB 目录：

```bash
ls -ld /usr/share/exploitdb
```

查看主要目录：

```bash
find /usr/share/exploitdb \
  -maxdepth 2 \
  -type d |
head -n 20
```

如果没有安装：

```bash
sudo apt update
sudo apt install exploitdb
```

---

# 11. Kali 中的主要路径

标准 Kali 软件包通常使用：

```text
/usr/share/exploitdb/
```

主要内容：

```text
/usr/share/exploitdb/exploits/
/usr/share/exploitdb/shellcodes/
```

检查：

```bash
ls /usr/share/exploitdb
```

数据库索引通常是 CSV 文件。

查找：

```bash
find /usr/share/exploitdb \
  -maxdepth 2 \
  -type f \
  -name '*.csv'
```

---

# 12. 可选数据库包

可选安装：

```bash
sudo apt install exploitdb-papers
```

```bash
sudo apt install exploitdb-bin-sploits
```

注意：

```text
Papers 占用较多空间
Binary Exploits 可能包含未知二进制文件
```

本课不需要安装 Binary Exploits。

不要执行未知 Binary。

---

# 13. 更新本地数据库

```bash
searchsploit -u
```

在 Kali 软件包环境中，这会检查并安装可用的 Exploit-DB 更新。

也可以正常更新软件包：

```bash
sudo apt update
sudo apt install --only-upgrade exploitdb
```

你的 CyberLab 是隔离环境时：

```text
只临时给 Kali 提供互联网
Ubuntu、Windows Target 继续保持隔离
更新完成后恢复 Kali 的实验网络设置
```

不要给靶机长期增加 Shared Network。

---

# 14. 记录数据库状态

```bash
apt policy exploitdb
```

```bash
dpkg -s exploitdb |
grep -E '^(Package|Version|Status):'
```

记录日期：

```bash
date -Iseconds
```

保存：

```bash
mkdir -p ~/CyberLab/SearchSploit/notes
```

```bash
{
  date -Iseconds
  apt policy exploitdb
} > ~/CyberLab/SearchSploit/notes/database-version.txt
```

---

# 15. 创建实验目录

```bash
mkdir -p ~/CyberLab/SearchSploit/{searches,review,nmap,notes,reports,quarantine}
```

检查：

```bash
find ~/CyberLab/SearchSploit \
  -maxdepth 2 \
  -type d
```

目录用途：

| 目录 | 用途 |
|---|---|
| `searches` | 搜索输出 |
| `review` | 复制后静态阅读的 PoC |
| `nmap` | Nmap XML |
| `notes` | 数据库和分析笔记 |
| `reports` | 最终报告 |
| `quarantine` | 未审查的未知文件 |

---

# 16. 最基础搜索

```bash
searchsploit apache
```

多个关键词：

```bash
searchsploit apache linux remote
```

SearchSploit 默认使用：

```text
AND
```

因此：

```bash
searchsploit apache linux remote
```

要求结果匹配多个词，而不是任意一个词。

---

# 17. 搜索词越多，结果越少

宽泛搜索：

```bash
searchsploit apache
```

缩小：

```bash
searchsploit apache 2.4
```

进一步缩小：

```bash
searchsploit apache 2.4 linux remote
```

原则：

```text
先广后窄
```

结果为零时，减少关键词。

---

# 18. 不要使用缩写

不推荐：

```bash
searchsploit sqli
```

推荐：

```bash
searchsploit "SQL Injection"
```

不推荐：

```bash
searchsploit privesc
```

推荐：

```bash
searchsploit "Privilege Escalation"
```

原因：

```text
Exploit Title 可能使用完整术语
```

---

# 19. 软件名称和版本搜索

示例：

```bash
searchsploit openssh 8.9
```

```bash
searchsploit apache 2.4.49
```

```bash
searchsploit wordpress 5.0
```

```bash
searchsploit linux kernel 5.15
```

结果会随本地数据库版本变化。

不要因为没有结果就认为：

```text
软件没有漏洞
```

---

# 20. Platform 和 Type 搜索

Windows Remote：

```bash
searchsploit windows remote
```

Linux Local：

```bash
searchsploit linux local
```

Web Application SQL Injection：

```bash
searchsploit webapps "SQL Injection"
```

Denial of Service：

```bash
searchsploit apache dos
```

本课只查看 DoS 记录，不复制或运行。

---

# 21. 默认搜索范围

默认情况下，SearchSploit 会搜索：

```text
Exploit Title
文件 Path
```

因此搜索数字时可能产生 Path 相关误报。

例如：

```text
版本号中的数字
目录名中的数字
架构名中的数字
```

这时使用 Title Search。

---

# 22. 只搜索 Title

```bash
searchsploit -t apache 2.4.49
```

`-t`：

```text
只搜索 Exploit Title
不使用文件 Path 进行匹配
```

适合减少：

- Platform Path 误报
- 版本数字误报
- 文件名误报

---

# 23. Exact Search

```bash
searchsploit -e "WordPress 4.1"
```

`-e`：

```text
要求 Exploit Title 中按顺序进行精确匹配
并隐含 Title Search
```

精确搜索太严格时可能没有结果。

推荐流程：

```text
普通搜索
→ Title Search
→ Exact Search
```

---

# 24. Strict Version Search

```bash
searchsploit -s "Apache Struts 2.0.0"
```

`-s` 会关闭模糊版本范围匹配。

默认搜索可能把：

```text
1.1
```

匹配到：

```text
1.0 < 1.3
```

Strict 更适合确认具体版本字符串。

---

# 25. Case-sensitive Search

```bash
searchsploit -c Apache
```

默认不区分大小写。

`-c` 用于：

```text
区分 Apache 与 apache
```

一般情况下不需要。

---

# 26. 排除不需要的结果

排除 DoS：

```bash
searchsploit linux kernel 5.15 \
  --exclude='/dos/'
```

排除 PoC 和 DoS：

```bash
searchsploit linux kernel 5.15 \
  --exclude='(PoC)|/dos/'
```

多个排除词使用：

```text
|
```

不要把 Shell 的 Pipe 与引号内的分隔符混淆。

---

# 27. 按 CVE 搜索

```bash
searchsploit --cve 2021-41773
```

也可以尝试：

```bash
searchsploit --cve CVE-2021-41773
```

具体接受格式以本机帮助为准。

CVE 搜索可能返回：

- 一个 Exploit
- 多个 Exploit
- 无结果

无结果不代表 CVE 不存在。

---

# 28. 输出 EDB-ID

```bash
searchsploit --id apache 2.4.49
```

普通输出显示 Path。

`--id` 改为显示：

```text
EDB-ID
```

适合后续：

```text
-p
-x
-m
```

---

# 29. 显示完整标题

```bash
searchsploit -o apache 2.4.49
```

`-o` 允许 Title 超出默认列宽。

适合长标题。

---

# 30. Verbose

```bash
searchsploit -v apache 2.4.49
```

Verbose 会显示更多搜索信息。

在 Nmap XML 联动时：

```bash
searchsploit --nmap scan.xml -v
```

可能搜索更多组合。

---

# 31. 禁用颜色

```bash
searchsploit \
  --disable-colour \
  apache 2.4.49
```

适合保存纯文本：

```bash
searchsploit \
  --disable-colour \
  apache 2.4.49 \
  > ~/CyberLab/SearchSploit/searches/apache-2.4.49.txt
```

---

# 32. 显示在线 URL

```bash
searchsploit -w apache 2.4.49
```

`-w` 显示 Exploit-DB 网站 URL，而不是本地 Path。

这不会自动打开网页。

隔离 Kali 中，URL 可能无法访问。

---

# 33. JSON 输出

```bash
searchsploit -j apache 2.4.49
```

保存：

```bash
searchsploit \
  -j apache 2.4.49 \
  > ~/CyberLab/SearchSploit/searches/apache-2.4.49.json
```

格式化：

```bash
jq . \
  ~/CyberLab/SearchSploit/searches/apache-2.4.49.json
```

如果没有 jq：

```bash
sudo apt install jq
```

---

# 34. 使用 jq 提取结果

先查看结构：

```bash
jq 'keys' \
  ~/CyberLab/SearchSploit/searches/apache-2.4.49.json
```

常见 SearchSploit JSON 可能包含不同结果数组。

安全地查看：

```bash
jq . \
  ~/CyberLab/SearchSploit/searches/apache-2.4.49.json |
less
```

不同 SearchSploit 版本的字段结构可能变化。

---

# 35. 查看 EDB-ID 的本地路径

```bash
searchsploit -p <EDB-ID>
```

示例格式：

```bash
searchsploit -p 39446
```

它会显示本地完整路径。

某些桌面环境还可能尝试复制路径到剪贴板。

不要直接执行路径中的代码。

---

# 36. 使用 PAGER 阅读

```bash
searchsploit -x <EDB-ID>
```

`-x` 会使用 `$PAGER` 打开 PoC。

退出 `less`：

```text
q
```

搜索文字：

```text
/keyword
```

下一处：

```text
n
```

上一处：

```text
N
```

---

# 37. 设置安全的 PAGER

检查：

```bash
echo "$PAGER"
```

可设置：

```bash
export PAGER='less -R'
```

然后：

```bash
searchsploit -x <EDB-ID>
```

不要把 PAGER 设置成会自动执行文件的程序。

---

# 38. Mirror 到当前目录

进入独立目录：

```bash
cd ~/CyberLab/SearchSploit/review
```

复制：

```bash
searchsploit -m <EDB-ID>
```

`-m`：

```text
只复制 Exploit 文件
不执行它
```

立即检查：

```bash
ls -la
```

---

# 39. 不要直接运行 Mirrored 文件

不要这样做：

```bash
chmod +x exploit.py
./exploit.py
```

也不要：

```bash
python3 exploit.py
```

直到完成：

- 代码审查
- 适用性确认
- 依赖确认
- 安全靶场准备
- 快照
- 网络隔离
- 明确授权
- 回滚计划

本课程不执行 Exploit。

---

# 40. 隔离未知文件

先复制到：

```text
~/CyberLab/SearchSploit/quarantine/
```

示例：

```bash
cd ~/CyberLab/SearchSploit/quarantine
searchsploit -m <EDB-ID>
```

设置只读：

```bash
chmod a-w ./*
```

查看权限：

```bash
ls -l
```

只读不能保证文件安全，但可减少误修改。

---

# 41. 计算 Hash

```bash
sha256sum ./*
```

保存：

```bash
sha256sum ./* \
  > SHA256SUMS.txt
```

Hash 用途：

- 记录研究时的文件版本
- 证明文件是否变化
- 报告证据
- 对比修改前后版本

Hash 不能证明代码安全。

---

# 42. 检查文件类型

```bash
file ./*
```

常见类型：

- Python Script
- Perl Script
- Ruby Script
- C Source
- Text
- Shell Script
- ELF Binary
- PE Binary

如果是未知 Binary：

```text
不要执行
```

---

# 43. 第一轮静态查看

```bash
head -n 80 FILE
```

```bash
tail -n 80 FILE
```

```bash
less FILE
```

统计：

```bash
wc -l FILE
```

检查编码：

```bash
file -bi FILE
```

---

# 44. 搜索危险关键字

```bash
grep -nEi \
  'system\(|exec\(|popen\(|subprocess|os\.system|eval\(|shell=True|rm -rf|chmod|chown|curl|wget|socket|connect\(|bind\(|listen\(|/bin/sh|powershell|cmd\.exe' \
  FILE
```

这只是初筛。

没有命中不代表文件安全。

---

# 45. 搜索网络和目标参数

```bash
grep -nEi \
  'target|host|port|url|uri|callback|lhost|lport|rhost|rport|payload|shellcode' \
  FILE
```

记录：

- 默认目标
- 默认端口
- Callback 地址
- Payload 类型
- 可配置参数
- Hard-coded 地址

---

# 46. 搜索文件读写行为

```bash
grep -nEi \
  'open\(|write\(|unlink\(|remove\(|rename\(|/etc/|/tmp/|AppData|ProgramData|System32' \
  FILE
```

检查代码是否：

- 创建文件
- 覆盖文件
- 删除文件
- 修改配置
- 写系统目录
- 下载额外组件

---

# 47. 语言特定检查

## Python

```bash
grep -nE \
  'import |subprocess|os\.|socket|requests|urllib|eval|exec' \
  FILE
```

只做语法检查：

```bash
python3 -m py_compile FILE
```

注意：

```text
py_compile 只编译语法
通常不会运行主程序
但未知 Python Import 和构建行为仍需谨慎
```

初学阶段可跳过未知代码的任何解释器调用。

## Shell

```bash
bash -n FILE
```

`bash -n` 只检查语法，不执行命令。

## Perl

```bash
perl -c FILE
```

## Ruby

```bash
ruby -c FILE
```

即使是语法检查，也只应在隔离研究 VM 中进行。

---

# 48. C/C++ 源代码检查

不要立即编译。

先查看：

```bash
grep -nE \
  'system\(|exec|fork|socket|connect|bind|listen|strcpy|strcat|sprintf|memcpy|shellcode' \
  FILE
```

如果以后需要编译，应该：

- 使用独立克隆 VM
- 禁止 Shared Folder
- 保持网络隔离
- 检查 Makefile
- 检查编译命令
- 不以 root 编译
- 不在目标系统编译

本课不编译 Exploit。

---

# 49. 静态审查问题清单

阅读每个 PoC 时回答：

```text
作者是谁？
发布日期是什么？
对应 EDB-ID 和 CVE 是什么？
目标软件是什么？
受影响版本是什么？
目标 OS 是什么？
目标架构是什么？
Local 还是 Remote？
是否需要认证？
是否需要已有 Shell？
是否要求特定配置？
默认 IP 和 Port 是什么？
代码是否包含 Payload？
代码是否执行 Shell Command？
代码是否下载外部文件？
代码是否修改或删除文件？
代码是否可能导致 Crash？
是否有硬编码 Offset？
是否只针对特定 Build？
是否存在 Python 2 依赖？
是否引用已失效的库？
是否有明确 Usage？
是否有修复和 Patch 信息？
```

---

# 50. 版本匹配不是只看数字

必须同时检查：

- Product Name
- Edition
- Component
- Plugin
- Version
- Build
- Patch Level
- Operating System
- Architecture
- Configuration
- Authentication
- Network Exposure

例如：

```text
Apache HTTP Server
```

与：

```text
使用 Apache 的某个第三方 Web App
```

不是同一个目标。

---

# 51. Distribution Backport

Linux 发行版可能保留较旧版本字符串，同时回移安全补丁。

因此：

```text
Banner 看起来旧
不代表漏洞未修复
```

检查：

- 发行版安全公告
- 软件包 Revision
- Changelog
- Patch 状态
- 实际行为

不要只依赖：

```text
nmap -sV
Server Header
WhatWeb
```

---

# 52. Exploit 适用性矩阵

| 条件 | 目标证据 | PoC 要求 | Match |
|---|---|---|---|
| Product |  |  | Yes/No |
| Version |  |  | Yes/No/Unknown |
| OS |  |  | Yes/No |
| Architecture |  |  | Yes/No |
| Local/Remote |  |  | Yes/No |
| Authentication |  |  | Yes/No |
| Configuration |  |  | Yes/No/Unknown |
| Patch |  |  | Yes/No/Unknown |
| Network Reachability |  |  | Yes/No |
| Required Privilege |  |  | Yes/No |

只有大多数关键条件明确匹配，才值得建立专门靶场继续验证。

---

# 53. Exploit Title 不足以证明适用

Title 可能写：

```text
Product 1.0 - Remote Code Execution
```

代码注释可能进一步要求：

```text
Windows x86 only
Authentication required
Specific Plugin enabled
Debug Mode enabled
Hard-coded language pack
Specific Build
```

必须阅读：

```text
Title
Header Comments
Usage
Source Code
References
```

---

# 54. CVE 研究流程

```text
1. 从 Nmap/WhatWeb/Nikto 得到产品和版本
2. 查询厂商安全公告
3. 找到 CVE
4. SearchSploit --cve
5. 阅读 Exploit Header
6. 阅读漏洞前置条件
7. 确认 Patch 状态
8. 决定是否建立靶场
```

SearchSploit 不是厂商安全公告的替代品。

---

# 55. “No Results” 的正确解释

SearchSploit 无结果可能因为：

- 本地数据库过旧
- 搜索词过于严格
- 产品名称不同
- 版本写法不同
- 只有 Advisory，没有公开 PoC
- PoC 不在 Exploit-DB
- CVE 没有关联 EDB-ID
- 漏洞尚未公开
- 搜索使用了缩写

正确操作：

```text
减少关键词
去掉 Strict
只搜索产品名
搜索 CVE
更新数据库
查看厂商公告
```

---

# 56. “Many Results” 的处理

结果太多：

```text
加入版本
加入平台
加入 remote/local/webapps
使用 -t
使用 -s
使用 -e
使用 --exclude
```

示例：

```bash
searchsploit wordpress
```

缩小：

```bash
searchsploit -t wordpress 5.0
```

继续：

```bash
searchsploit -t wordpress 5.0 \
  --exclude='/dos/'
```


---

# 57. 保存搜索结果

纯文本：

```bash
searchsploit \
  --disable-colour \
  -t apache 2.4.49 \
  > ~/CyberLab/SearchSploit/searches/apache-title.txt
```

JSON：

```bash
searchsploit \
  -j apache 2.4.49 \
  > ~/CyberLab/SearchSploit/searches/apache.json
```

同时保留命令：

```bash
{
  echo '$ searchsploit -t apache 2.4.49'
  searchsploit \
    --disable-colour \
    -t apache 2.4.49
} > ~/CyberLab/SearchSploit/searches/apache-with-command.txt
```

---

# 58. 使用 tee

在终端查看并保存：

```bash
searchsploit \
  --disable-colour \
  apache 2.4.49 |
tee ~/CyberLab/SearchSploit/searches/apache-2.4.49.txt
```

记录时间：

```bash
{
  date -Iseconds
  searchsploit \
    --disable-colour \
    apache 2.4.49
} | tee ~/CyberLab/SearchSploit/searches/apache-dated.txt
```

---

# 59. 对比普通、Title 和 Strict

```bash
searchsploit \
  --disable-colour \
  apache 2.4.49 \
  > ~/CyberLab/SearchSploit/searches/apache-normal.txt
```

```bash
searchsploit \
  --disable-colour \
  -t apache 2.4.49 \
  > ~/CyberLab/SearchSploit/searches/apache-title.txt
```

```bash
searchsploit \
  --disable-colour \
  -s apache 2.4.49 \
  > ~/CyberLab/SearchSploit/searches/apache-strict.txt
```

统计：

```bash
for file in ~/CyberLab/SearchSploit/searches/apache-*.txt
do
  echo "===== $file ====="
  wc -l "$file"
done
```

说明：

```text
行数包含 Header 和 Footer
不完全等于结果数量
```

---

# 60. 使用 diff 比较过滤效果

```bash
diff -u \
  ~/CyberLab/SearchSploit/searches/apache-normal.txt \
  ~/CyberLab/SearchSploit/searches/apache-title.txt
```

再比较：

```bash
diff -u \
  ~/CyberLab/SearchSploit/searches/apache-title.txt \
  ~/CyberLab/SearchSploit/searches/apache-strict.txt
```

回答：

```text
哪些结果因为 Path 匹配而出现？
Strict 是否移除了版本范围结果？
Exact 是否过于严格？
```

---

# 61. 使用 grep 分析结果

只看 Remote：

```bash
grep -i 'remote' \
  ~/CyberLab/SearchSploit/searches/apache-normal.txt
```

排除 DoS：

```bash
grep -vi '/dos/' \
  ~/CyberLab/SearchSploit/searches/apache-normal.txt
```

查看 Python PoC：

```bash
grep -E '\.py$' \
  ~/CyberLab/SearchSploit/searches/apache-normal.txt
```

不要只根据文件扩展名判断安全性。

---

# 62. JSON 搜索工作流

```bash
searchsploit \
  -j wordpress 5.0 \
  > ~/CyberLab/SearchSploit/searches/wordpress-5.json
```

查看：

```bash
jq . \
  ~/CyberLab/SearchSploit/searches/wordpress-5.json |
less
```

搜索字符串：

```bash
jq -r '.. | strings' \
  ~/CyberLab/SearchSploit/searches/wordpress-5.json |
grep -iE 'remote|local|webapps|dos'
```

这种写法不依赖固定 JSON 字段名。

---

# 63. 使用 Path 找 EDB-ID

结果 Path 可能类似：

```text
multiple/webapps/12345.py
```

其中：

```text
12345
```

通常对应 EDB-ID。

但最稳妥的方法：

```bash
searchsploit --id SEARCH_TERMS
```

然后使用：

```bash
searchsploit -p <EDB-ID>
```

---

# 64. 复制后保留原始文件

进入研究目录：

```bash
mkdir -p ~/CyberLab/SearchSploit/review/case-01
cd ~/CyberLab/SearchSploit/review/case-01
```

复制：

```bash
searchsploit -m <EDB-ID>
```

计算原始 Hash：

```bash
sha256sum ./* \
  > original-SHA256SUMS.txt
```

如果需要做注释，先复制：

```bash
cp ORIGINAL_FILE review-copy.txt
```

不要修改原始文件。

---

# 65. 创建静态审查记录

```bash
cat > review-notes.md <<'EOF'
# SearchSploit Static Review

## Identification

- EDB-ID:
- CVE:
- Title:
- Author:
- Published:
- Local path:
- SHA-256:

## Target Requirements

- Product:
- Version:
- OS:
- Architecture:
- Local/Remote:
- Authentication:
- Required privilege:
- Required configuration:

## Code Behaviour

- Network connection:
- Files created:
- Files modified:
- Commands executed:
- External downloads:
- Payload:
- Crash/DoS risk:
- Hard-coded values:

## Applicability

- Product match:
- Version match:
- Platform match:
- Configuration match:
- Patch status:
- Overall confidence:

## Decision

- Static review only
- Build dedicated lab
- Reject as non-matching
- Needs more research
EOF
```

---

# 66. 可疑外部下载

PoC 代码可能引用：

```text
wget
curl
requests.get
urllib
Invoke-WebRequest
PowerShell DownloadString
GitHub Raw
Paste sites
Unknown IP
```

检查：

```bash
grep -nEi \
  'https?://|ftp://|wget|curl|DownloadString|Invoke-WebRequest|requests\.get|urllib' \
  FILE
```

不要自动下载链接内容。

链接可能：

- 已失效
- 被接管
- 内容已改变
- 包含恶意文件
- 与原 PoC 不一致

---

# 67. Hard-coded Target

检查：

```bash
grep -nE \
  '([0-9]{1,3}\.){3}[0-9]{1,3}' \
  FILE
```

代码可能包含：

- 作者测试 IP
- Callback IP
- 默认目标
- 示例 DNS
- 公网服务器

不要运行带有未知 Hard-coded 地址的代码。

---

# 68. Hard-coded Credentials

```bash
grep -nEi \
  'password|passwd|username|login|credential|secret|api[_-]?key|token' \
  FILE
```

可能是：

- 默认账号
- 测试凭据
- Exploit 前置条件
- 作者遗留秘密
- Payload 参数

不要尝试这些凭据访问真实系统。

---

# 69. 检查 Shebang

```bash
head -n 1 FILE
```

常见：

```text
#!/usr/bin/env python
#!/usr/bin/python2
#!/usr/bin/perl
#!/bin/bash
#!/usr/bin/ruby
```

如果是 Python 2：

```text
现代 Kali 可能无法直接运行
代码也可能存在过时依赖
```

本课只记录，不修改为 Python 3 并执行。

---

# 70. 检查 Usage 和参数

```bash
grep -nEi \
  'usage|example|argument|option|target|host|port' \
  FILE |
head -n 80
```

关注：

- 参数顺序
- 默认端口
- Required Flag
- Target Format
- Authentication
- Callback
- Payload

---

# 71. 检查破坏性行为

```bash
grep -nEi \
  'delete|drop table|truncate|format|shutdown|reboot|kill|crash|denial|overwrite|rm -rf|unlink' \
  FILE
```

如果 PoC 可能：

- 删除文件
- 删除数据库
- 停止服务
- 重启系统
- 导致 Kernel Panic
- 破坏数据

标记：

```text
Do not execute
```

---

# 72. 检查 Privilege 要求

```bash
grep -nEi \
  'root|administrator|sudo|system|kernel|privilege|setuid|uid=0' \
  FILE
```

Local Privilege Escalation 通常要求：

```text
攻击者已经有本地低权限访问
```

如果你只有网络端口可达：

```text
Local Exploit 不适用当前阶段
```

---

# 73. 检查架构

```bash
grep -nEi \
  'x86|x64|x86_64|amd64|arm|arm64|aarch64|mips|32-bit|64-bit' \
  FILE
```

你的实验环境主要是：

```text
Apple Silicon ARM64
Guest OS 多数为 ARM64
```

很多旧 x86/x64 Exploit：

```text
不能直接用于 ARM64
```

即使软件版本相同，架构也可能不匹配。

---

# 74. Apple Silicon CyberLab 的特殊限制

你的 MacBook 使用 Apple Silicon。

因此：

```text
Kali ARM64
Ubuntu ARM64
Windows ARM64
```

静态研究时必须检查：

- CPU Architecture
- Endianness
- Pointer Size
- Assembly
- Shellcode Architecture
- Binary Format
- Kernel Build
- Compatibility Layer

x86 Shellcode 不能直接在 ARM64 上执行。

本课不使用模拟器执行旧 Exploit。

---

# 75. 搜索 ARM64 相关结果

```bash
searchsploit arm64 local
```

```bash
searchsploit aarch64
```

```bash
searchsploit linux arm
```

结果较少是正常的。

不要因为标题包含 Linux 就假设支持 ARM64。

---

# 76. SearchSploit 与 Nmap XML

SearchSploit 可以读取 Nmap XML 中的服务和版本信息。

先运行 Nmap：

```bash
sudo nmap \
  -n \
  -sV \
  -p 22,8089 \
  10.10.20.20 \
  -oX ~/CyberLab/SearchSploit/nmap/ubuntu-services.xml
```

检查：

```bash
head -n 30 \
  ~/CyberLab/SearchSploit/nmap/ubuntu-services.xml
```

---

# 77. 使用 --nmap

```bash
searchsploit \
  --nmap ~/CyberLab/SearchSploit/nmap/ubuntu-services.xml
```

Verbose：

```bash
searchsploit \
  --nmap ~/CyberLab/SearchSploit/nmap/ubuntu-services.xml \
  -v
```

保存：

```bash
searchsploit \
  --disable-colour \
  --nmap ~/CyberLab/SearchSploit/nmap/ubuntu-services.xml \
  > ~/CyberLab/SearchSploit/searches/nmap-search.txt
```

---

# 78. Nmap XML 结果可能为空

你的实验服务可能被识别为：

```text
Python BaseHTTPServer
Unknown
OpenSSH with patched Ubuntu package
```

SearchSploit 可能：

```text
没有匹配
匹配过于宽泛
返回无关结果
```

这不表示命令失败。

重要的是学习：

```text
自动版本关联需要人工确认
```

---

# 79. Nmap Service Detection 的限制

Nmap `-sV` 依赖：

- Banner
- Probe Response
- Pattern Database
- 端口行为

可能出现：

- Banner 欺骗
- 版本缺失
- Product 识别错误
- Reverse Proxy
- Backported Patch
- 自定义服务

因此：

```text
Nmap → SearchSploit
只是研究起点
不是漏洞确认
```

---

# 80. Nmap 全部实验目标基线

只扫描自己的三个目标：

```bash
sudo nmap \
  -n \
  -sV \
  -p 22,80,443,8080-8090 \
  10.10.20.20 \
  10.10.20.30 \
  10.10.20.40 \
  -oX ~/CyberLab/SearchSploit/nmap/cyberlab-services.xml
```

注意：

- Windows 端口开放情况不同
- Wazuh 可能只开放部分端口
- 不要使用 `-p-` 作为本课默认
- 不要扫描 `10.10.20.0/24` 中未知设备

运行：

```bash
searchsploit \
  --nmap ~/CyberLab/SearchSploit/nmap/cyberlab-services.xml
```

---

# 81. 从 Nmap 手工提取产品

```bash
grep -oE \
  'product="[^"]+"|version="[^"]+"|extrainfo="[^"]+"' \
  ~/CyberLab/SearchSploit/nmap/ubuntu-services.xml
```

然后手工组合：

```text
Product + Major Version
Product + Exact Version
Product + Platform
```

进行 SearchSploit 搜索。

---

# 82. Case Study：Apache 2.4.49

这是一个离线研究案例。

搜索：

```bash
searchsploit apache 2.4.49
```

Title：

```bash
searchsploit -t apache 2.4.49
```

CVE：

```bash
searchsploit --cve 2021-41773
```

排除 DoS：

```bash
searchsploit apache 2.4.49 \
  --exclude='/dos/'
```

任务：

```text
只记录结果
不复制和执行
比较 Product、Version、Platform 和 Type
```

---

# 83. Case Study：OpenSSH

先查看 Ubuntu 服务：

```bash
sudo nmap \
  -n \
  -sV \
  -p 22 \
  10.10.20.20
```

根据实际输出搜索：

```bash
searchsploit openssh <VERSION>
```

再缩小：

```bash
searchsploit -t openssh <VERSION>
```

检查结果是：

- OpenSSH Server
- OpenSSH Client
- Portable
- User Enumeration
- Local
- Remote
- DoS

不要把所有 OpenSSH 结果都当成远程 RCE。

---

# 84. Case Study：Linux Kernel

Ubuntu 查看：

```bash
uname -a
```

```bash
uname -r
```

Kali 搜索：

```bash
searchsploit linux kernel <MAJOR.MINOR>
```

排除 DoS：

```bash
searchsploit linux kernel <MAJOR.MINOR> \
  --exclude='/dos/'
```

判断：

- ARM64 是否支持
- Local 还是 Remote
- 需要何种权限
- 精确 Kernel Build
- Ubuntu 是否回移补丁

本课不复制 Kernel Exploit。

---

# 85. Case Study：Web CMS

假设 WhatWeb 报告：

```text
WordPress 5.0
```

搜索：

```bash
searchsploit wordpress 5.0
```

只看 Title：

```bash
searchsploit -t wordpress 5.0
```

还要判断结果是：

- WordPress Core
- Theme
- Plugin
- WooCommerce Extension
- Authentication Required
- SQL Injection
- XSS
- File Upload
- DoS

Plugin Exploit 不适用于没有该 Plugin 的站点。

---

# 86. 无版本信息时的流程

如果只知道：

```text
Apache
WordPress
OpenSSH
```

不要直接把所有结果当发现。

正确做法：

```text
1. 使用 Nmap/WhatWeb 获取版本
2. 查看 Header 和页面源码
3. 查看软件包版本
4. 查看服务器端进程
5. 再搜索具体版本
```

没有版本时，SearchSploit 只适合：

```text
建立研究候选列表
```

---

# 87. SearchSploit 不访问目标

运行：

```bash
searchsploit apache 2.4.49
```

它只读取 Kali 本地数据库。

证明实验：

Kali 终端一：

```bash
sudo tcpdump \
  -i eth0 \
  -nn \
  'host 10.10.20.20 or host 10.10.20.30 or host 10.10.20.40'
```

终端二：

```bash
searchsploit apache 2.4.49
```

预期：

```text
SearchSploit 搜索期间没有访问目标的流量
```

---

# 88. 哪些操作可能产生网络流量？

```text
searchsploit SEARCH_TERMS
```

通常不访问目标或互联网。

可能产生网络流量的相关操作：

- `searchsploit -u`
- `apt update`
- 在线打开 Exploit-DB URL
- 手动下载外部文件
- 运行 PoC
- 使用 Nmap 生成 XML

注意：

```text
--nmap 读取现有 XML 时本身不重新扫描
```

---

# 89. Wazuh 可见性

SearchSploit 在 Kali 本地运行。

当前 Wazuh Agent 位于：

```text
Ubuntu Target
Windows Target
```

Kali 没有 Wazuh Agent，因此 Wazuh 通常看不到：

- SearchSploit 搜索词
- 本地 CSV 读取
- `-x` 阅读代码
- `-m` 在 Kali 创建文件

这是正常的。

---

# 90. Wazuh 手工实验标记

研究前，从 Kali：

```bash
ssh wazuh-test-user@10.10.20.20 \
  'logger -t searchsploit-lab "Starting authorized offline public exploit research on Kali 10.10.20.10"'
```

研究后：

```bash
ssh wazuh-test-user@10.10.20.20 \
  'logger -t searchsploit-lab "Completed authorized offline public exploit research"'
```

Ubuntu 检查：

```bash
journalctl \
  -t searchsploit-lab \
  --since "15 minutes ago"
```

Wazuh Dashboard 搜索：

```text
agent.name: ubuntu-target
```

然后：

```text
searchsploit-lab
```

---

# 91. Wazuh 观察结论

手工 Marker 只证明：

```text
实验时间和日志链路
```

它不证明 Wazuh 自动检测到 SearchSploit。

真正监控 Kali 研究行为，需要：

- Kali 安装 Wazuh Agent
- Auditd
- Process Monitoring
- File Integrity Monitoring
- Shell History Monitoring Policy
- EDR

本课不修改 Kali 监控架构。

---

# 92. 为什么不运行未知 PoC？

公开 PoC 可能：

- 有编程错误
- 破坏目标
- 包含恶意代码
- 连接作者服务器
- 写入后门
- 删除数据
- 使用硬编码 Payload
- 只适配旧环境
- 不稳定
- 导致 Kernel Panic
- 需要过时依赖
- 触发防病毒或 EDR

“公开”不等于“安全”。

---

# 93. 安全研究层级

## Level 1：搜索

```text
SearchSploit 关键词和 CVE
```

## Level 2：查看 Metadata

```text
Title、Path、Platform、Type、EDB-ID
```

## Level 3：静态阅读

```text
-x、less、grep、Hash
```

## Level 4：复制到研究目录

```text
-m，但不执行
```

## Level 5：专用靶场验证

需要：

- 明确授权
- 隔离克隆
- 快照
- 版本匹配
- 无敏感数据
- 回滚计划
- 监控

本课完成到：

```text
Level 4
```

---

# 94. 建立专用靶场前检查

```text
目标软件是否合法获得？
版本是否准确？
架构是否匹配？
漏洞是否会破坏数据？
是否有官方测试镜像？
是否有快照？
网络是否隔离？
是否关闭 Shared Folder？
是否禁止访问宿主机？
是否有回滚计划？
是否记录 Hash？
是否知道停止条件？
是否需要禁用 Wazuh 主动响应？
```

任何一项不清楚：

```text
不要运行 PoC
```

---

# 95. 文件权限和 Ownership

复制后：

```bash
ls -l
```

不要使用：

```bash
sudo chown root:root FILE
sudo chmod 777 FILE
```

研究文件应保持普通用户拥有：

```bash
chown "$USER":"$USER" FILE
chmod 600 FILE
```

这样可以减少误执行和其他用户读取。

---

# 96. 不在共享文件夹中研究

不要把未知 PoC 放在：

- macOS Shared Folder
- iCloud Drive
- OneDrive
- Google Drive 同步目录
- 自动备份目录
- 公共 Git Repository

原因：

- 文件可能同步到其他设备
- 安全软件可能触发
- 误提交
- 泄露研究内容
- 宿主机暴露

使用 Kali VM 内部：

```text
~/CyberLab/SearchSploit/
```

---

# 97. 不使用 root 阅读和复制

不要：

```bash
sudo searchsploit -x <EDB-ID>
```

不要：

```bash
sudo searchsploit -m <EDB-ID>
```

普通搜索和复制不需要 root。

最小权限原则：

```text
只在 Nmap SYN Scan 等必要操作使用 sudo
```

---

# 98. 不执行 Binary Exploit

如果文件类型是：

```text
ELF
PE32
Mach-O
Unknown Binary
```

本课只运行：

```bash
file FILE
sha256sum FILE
strings -a FILE | head
```

不要：

```bash
./FILE
wine FILE.exe
qemu-ARCH FILE
```

---

# 99. Strings 静态查看

只对副本：

```bash
strings -a FILE |
head -n 100
```

搜索：

```bash
strings -a FILE |
grep -Ei \
  'http|socket|cmd|powershell|/bin/sh|password|token'
```

`strings` 结果不完整，也不能证明 Binary 行为。

---

# 100. 不进行在线 Malware 上传

不要把未知 PoC、公司文件或客户样本上传到公共分析网站。

公共上传可能：

- 公开样本
- 泄露客户数据
- 触发第三方共享
- 违反授权协议

本课只使用本地静态工具。

---

# 101. SearchSploit 与 Metasploit

SearchSploit 结果中可能包含：

```text
(Metasploit)
```

或 `.rb` 文件。

这可能表示有 Metasploit Module 或 Metasploit 格式代码。

但：

```text
SearchSploit 的 EDB 记录
与
当前 Metasploit 安装中的模块
不一定完全同步
```

下一阶段应在 Metasploit 中单独：

```text
search
info
show options
```

而不是直接执行模块。

---

# 102. 查找 Metasploit 相关结果

```bash
searchsploit apache metasploit
```

```bash
searchsploit wordpress metasploit
```

结果只表示：

```text
记录标题或路径与 Metasploit 有关
```

不代表模块已安装或适用于目标。

---

# 103. Exploit 类型判断表

| Type | 当前操作 |
|---|---|
| Remote RCE | 只静态审查 |
| Local Privilege Escalation | 检查已有访问和架构要求 |
| SQL Injection | 与 SQLmap 和代码证据对比 |
| XSS | 与 Burp/ZAP Alert 对比 |
| File Upload | 只读前置条件 |
| Authentication Bypass | 不在真实系统验证 |
| DoS | 不执行 |
| Shellcode | 不执行 |
| Binary Exploit | 不执行 |
| Metasploit Module | 留到下一课分析 |

---

# 104. 结果可信度

## High Confidence Candidate

```text
Product、Version、Platform、Architecture、Configuration 和 Patch 状态均匹配
```

## Medium Confidence Candidate

```text
产品和版本匹配，但配置或 Patch 状态未知
```

## Low Confidence Candidate

```text
只有 Product 名称匹配
```

## Rejected

```text
版本、平台、架构或前置条件不匹配
```

注意：

```text
High Confidence Candidate
仍然不是 Confirmed Exploitable
```

---

# 105. 研究决定

每条结果使用以下决定之一：

```text
Reject：不匹配
Monitor：等待更多版本/补丁信息
Static Review：继续阅读代码
Build Lab：建立专用脆弱版本靶场
Vendor Verification：先查官方公告
Do Not Test：破坏性、DoS 或授权不足
```


---

# 106. 完整实验一：基础搜索与过滤

记录数据库版本：

```bash
{
  date -Iseconds
  apt policy exploitdb
} | tee \
  ~/CyberLab/SearchSploit/notes/lab1-database-version.txt
```

依次运行：

```bash
searchsploit apache
```

```bash
searchsploit apache 2.4
```

```bash
searchsploit -t apache 2.4
```

```bash
searchsploit -s apache 2.4.49
```

```bash
searchsploit apache 2.4.49 \
  --exclude='(PoC)|/dos/'
```

保存每个结果。

比较：

| Search | Results | Useful | False Positives |
|---|---:|---:|---:|
| Broad |  |  |  |
| Version |  |  |  |
| Title |  |  |  |
| Strict |  |  |  |
| Exclude |  |  |  |

---

# 107. 完整实验二：CVE 与 JSON

搜索 CVE：

```bash
searchsploit \
  --cve 2021-41773 \
  --disable-colour |
tee ~/CyberLab/SearchSploit/searches/cve-2021-41773.txt
```

JSON：

```bash
searchsploit \
  -j \
  --cve 2021-41773 \
  > ~/CyberLab/SearchSploit/searches/cve-2021-41773.json
```

查看：

```bash
jq . \
  ~/CyberLab/SearchSploit/searches/cve-2021-41773.json |
less
```

记录：

```text
EDB-ID：
Title：
Path：
Platform：
Type：
是否 DoS：
是否 Metasploit：
```

不复制、不执行。

---

# 108. 完整实验三：安全静态审查

选择一个：

```text
非 DoS
源代码格式
与你的目标无直接攻击关系
仅用于阅读
```

显示路径：

```bash
searchsploit -p <EDB-ID>
```

查看：

```bash
searchsploit -x <EDB-ID>
```

复制到隔离目录：

```bash
mkdir -p \
  ~/CyberLab/SearchSploit/review/lab3-<EDB-ID>
```

```bash
cd \
  ~/CyberLab/SearchSploit/review/lab3-<EDB-ID>
```

```bash
searchsploit -m <EDB-ID>
```

记录：

```bash
file ./*
sha256sum ./*
wc -l ./*
```

只用 `less` 和 `grep` 阅读。

---

# 109. 完整实验四：静态代码审查

对复制的文件运行：

```bash
grep -nEi \
  'usage|target|host|port|payload|shellcode|callback' \
  FILE
```

```bash
grep -nEi \
  'system\(|exec\(|popen\(|subprocess|os\.system|shell=True' \
  FILE
```

```bash
grep -nEi \
  'https?://|wget|curl|DownloadString|requests\.get' \
  FILE
```

```bash
grep -nEi \
  'rm -rf|unlink|delete|drop table|shutdown|reboot|crash|dos' \
  FILE
```

完成 `review-notes.md`。

---

# 110. 完整实验五：Nmap XML 联动

Nmap：

```bash
sudo nmap \
  -n \
  -sV \
  -p 22,8089 \
  10.10.20.20 \
  -oX ~/CyberLab/SearchSploit/nmap/lab5-ubuntu.xml
```

SearchSploit：

```bash
searchsploit \
  --disable-colour \
  --nmap ~/CyberLab/SearchSploit/nmap/lab5-ubuntu.xml \
  -v |
tee ~/CyberLab/SearchSploit/searches/lab5-nmap-results.txt
```

回答：

```text
Nmap 识别了哪些 Product？
识别了哪些 Version？
SearchSploit 返回多少候选？
哪些候选明显不匹配？
为什么自动关联不能作为漏洞证明？
```

---

# 111. 完整实验六：ARM64 适用性

Ubuntu：

```bash
uname -m
```

```bash
uname -r
```

Kali 搜索：

```bash
searchsploit \
  linux kernel "$(ssh wazuh-test-user@10.10.20.20 uname -r)"
```

宽泛 ARM：

```bash
searchsploit arm64 local
```

```bash
searchsploit aarch64
```

选择一个结果，只检查：

- Platform
- Architecture
- Kernel Range
- Local/Remote
- Required Privilege

不复制 Kernel Exploit。

---

# 112. 完整实验七：证明离线行为

终端一：

```bash
sudo tcpdump \
  -i eth0 \
  -nn \
  'host 10.10.20.20 or host 10.10.20.30 or host 10.10.20.40'
```

终端二：

```bash
searchsploit apache 2.4.49
```

```bash
searchsploit --cve 2021-41773
```

```bash
searchsploit -x <EDB-ID>
```

观察：

```text
这些本地操作是否访问 CyberLab Target？
```

然后运行一次 Nmap，比较主动网络流量。

---

# 113. 完整实验八：Wazuh Marker

开始：

```bash
ssh wazuh-test-user@10.10.20.20 \
  'logger -t searchsploit-lab "Starting SearchSploit lab 8"'
```

进行：

```text
基础搜索
CVE 搜索
静态阅读
```

结束：

```bash
ssh wazuh-test-user@10.10.20.20 \
  'logger -t searchsploit-lab "Completed SearchSploit lab 8"'
```

Wazuh：

```text
Threat Hunting
agent.name: ubuntu-target
searchsploit-lab
```

解释：

```text
Wazuh 看到的是手工 Marker
不是自动检测 SearchSploit
```

---

# 114. 完整实验九：产品适用性报告

从你的 Nmap XML 选择一个真实服务。

填写：

```text
Target：
Port：
Detected Product：
Detected Version：
Detection Source：
Search Terms：
Candidate EDB-ID：
Candidate Title：
Candidate Type：
Target Architecture：
PoC Architecture：
Authentication Requirement：
Configuration Requirement：
Patch Status：
Decision：
Confidence：
```

最终决定只能是：

```text
Reject
Needs More Research
Static Review
Build Dedicated Lab
```

本课不选择：

```text
Execute Against Target
```

---

# 115. Public Exploit Research 报告模板

## Lab Information

```text
Lab Name：SearchSploit Public Exploit Research
Date：
Student：
Research Host：Kali-CyberLab 10.10.20.10
Targets：
- Ubuntu 10.10.20.20
- Windows 10.10.20.30
- Wazuh 10.10.20.40
Authorization：Personal CyberLab
Exploit-DB Package Version：
SearchSploit Database Date：
```

## Objective

```text
根据授权服务版本和 CVE 搜索公开 Exploit/PoC，
进行静态代码审查和适用性判断，不执行 Exploit。
```

## Service Evidence

| Target | Port | Product | Version | Evidence |
|---|---:|---|---|---|
|  |  |  |  |  |

## Search Commands

```bash
# Product search
# Version search
# Title search
# Strict search
# CVE search
# Nmap XML search
```

## Search Results

| EDB-ID | Title | Path | Platform | Type | CVE |
|---:|---|---|---|---|---|
|  |  |  |  |  |  |

## Static Review

```text
File：
SHA-256：
Language：
Lines：
Usage：
Target Product：
Target Version：
Architecture：
Authentication：
Commands Executed：
Network Connections：
Files Modified：
External Download：
Destructive Behaviour：
```

## Applicability Matrix

| Condition | Target | PoC | Match |
|---|---|---|---|
| Product |  |  |  |
| Version |  |  |  |
| OS |  |  |  |
| Architecture |  |  |  |
| Local/Remote |  |  |  |
| Authentication |  |  |  |
| Configuration |  |  |  |
| Patch Status |  |  |  |

## Decision

```text
Candidate Status：
- Rejected
- Needs More Research
- Static Review Complete
- Dedicated Lab Recommended

Confidence：
Reason：
```

## Safety Analysis

```text
Crash Risk：
Data Modification Risk：
Privilege Requirement：
External Network Risk：
Rollback Requirement：
```

## Evidence Files

```text
Nmap XML：
Search TXT：
Search JSON：
Mirrored File：
SHA256SUMS：
Review Notes：
tcpdump Observation：
Wazuh Marker：
```

## Recommendations

```text
Verify vendor advisory.
Verify package patch status.
Do not rely only on banners.
Do not execute unreviewed code.
Use an isolated version-matched clone for later validation.
Keep exploit research files out of shared folders.
```

---

# 116. 常见错误与解决方法

## searchsploit command not found

```bash
sudo apt update
sudo apt install exploitdb
```

确认：

```bash
which searchsploit
```

## /usr/share/exploitdb 不存在

```bash
dpkg -L exploitdb |
head -n 30
```

重新安装：

```bash
sudo apt install --reinstall exploitdb
```

## 没有搜索结果

尝试：

```text
减少关键词
删除版本小数位
去掉 -s
去掉 -e
使用产品全名
搜索 CVE
更新数据库
```

## 结果太多

使用：

```text
-t
-s
-e
--exclude
Platform
Version
Local/Remote
```

## -p 找不到 EDB-ID

确认：

```bash
searchsploit --id SEARCH_TERMS
```

数据库可能已经变化，或输入的数字不是 EDB-ID。

## -x 没有打开

检查：

```bash
echo "$PAGER"
```

```bash
export PAGER='less -R'
```

再试：

```bash
searchsploit -x <EDB-ID>
```

## -m 没有复制到预期位置

`-m` 会复制到：

```text
当前工作目录
```

先：

```bash
pwd
```

再创建研究目录并 `cd`。

## JSON 无法解析

检查：

```bash
head -n 20 FILE.json
```

确认命令没有把普通文字混入 JSON。

重新：

```bash
searchsploit -j SEARCH_TERMS > result.json
```

## jq command not found

```bash
sudo apt install jq
```

## --nmap 没有结果

检查 XML：

```bash
xmllint \
  --noout \
  ~/CyberLab/SearchSploit/nmap/scan.xml
```

查看 Service：

```bash
grep '<service ' \
  ~/CyberLab/SearchSploit/nmap/scan.xml
```

Nmap 可能没有识别 Product 或 Version。

## searchsploit -u 失败

原因可能：

- Kali 没有互联网
- APT Source 问题
- DNS 问题
- 权限问题
- 软件包锁

检查：

```bash
sudo apt update
```

不要为了更新给 Target VM 增加互联网。

---

# 117. 为什么搜索结果会变化？

结果取决于：

- Exploit-DB 本地数据库版本
- Kali 软件包版本
- 新增或修订记录
- 搜索词
- 默认版本范围匹配
- Title 和 Path
- Strict / Exact
- Exclude
- Papers 配置

报告中必须记录：

```text
Date
Exploit-DB Package Version
Search Command
Search Terms
EDB-ID
SHA-256
```

---

# 118. Exploit-DB 记录的局限

公开 PoC 可能缺少：

- 完整测试环境
- 可靠错误处理
- Patch 信息
- 现代依赖
- ARM64 支持
- 稳定性
- 安全回滚
- 详细前置条件

记录可能多年未更新。

必须独立验证：

```text
厂商公告
CVE Record
发行版公告
软件包 Changelog
代码行为
```

---

# 119. 文件保留策略

建议结构：

```text
SearchSploit/
├── searches/
├── review/
├── nmap/
├── notes/
├── reports/
└── quarantine/
```

建议文件名：

```text
2026-08-06-apache-search.txt
2026-08-06-cve-2021-41773.json
EDB-12345-original.py
EDB-12345-SHA256SUMS.txt
EDB-12345-review.md
```

---

# 120. Git 管理建议

可以提交：

- 自己写的研究笔记
- 清理后的搜索摘要
- 不包含敏感信息的报告模板

不要公开提交：

- 未审查 PoC
- Binary Exploit
- Payload
- Shellcode
- 客户环境信息
- 内部 IP
- Cookie
- 凭据
- PCAP
- 未清理 Nmap XML

`.gitignore`：

```gitignore
quarantine/
review/*/original-*
*.pcap
*.har
*.bin
*.exe
*.elf
```

---

# 121. 本课练习

1. 检查 `searchsploit` 和 `exploitdb`。
2. 记录软件包版本和日期。
3. 创建 SearchSploit 实验目录。
4. 运行宽泛 Apache 搜索。
5. 加入版本号。
6. 使用 `-t`。
7. 使用 `-s`。
8. 使用 `-e`。
9. 使用 `--exclude`。
10. 使用 `--cve`。
11. 使用 `--id`。
12. 使用 `-o`。
13. 使用 `-v`。
14. 使用 `--disable-colour`。
15. 使用 `-w` 查看在线 URL 输出。
16. 保存 JSON。
17. 使用 jq 查看 JSON。
18. 使用 `-p` 查看路径。
19. 使用 `-x` 静态阅读。
20. 使用 `-m` 复制一个源代码 PoC。
21. 计算 SHA-256。
22. 使用 `file` 和 `wc`。
23. 搜索网络行为。
24. 搜索命令执行行为。
25. 搜索外部下载。
26. 搜索破坏性行为。
27. 填写静态审查笔记。
28. 生成 Ubuntu Nmap XML。
29. 使用 `--nmap`。
30. 人工排除不匹配候选。
31. 检查 ARM64 适用性。
32. 使用 tcpdump 证明离线搜索。
33. 在 Wazuh 创建开始和结束 Marker。
34. 填写适用性矩阵。
35. 完成 Public Exploit Research 报告。

---

# 122. 复习题

1. Exploit-DB 主要是什么类型的数据库？
2. SearchSploit 是否会自动扫描目标？
3. SearchSploit 为什么适合隔离网络？
4. EDB-ID 与 CVE 有什么区别？
5. Exploit 与 Payload 有什么区别？
6. PoC 是否等于稳定 Exploit？
7. Local Exploit 通常需要什么？
8. Remote Exploit 是否一定无需认证？
9. DoS PoC 为什么不应在本课执行？
10. SearchSploit 默认使用 AND 还是 OR？
11. 默认搜索哪些位置？
12. `-t` 做什么？
13. `-e` 做什么？
14. `-s` 做什么？
15. `-c` 做什么？
16. `--exclude` 做什么？
17. `--cve` 做什么？
18. `--id` 做什么？
19. `-p` 做什么？
20. `-x` 做什么？
21. `-m` 做什么？
22. `-j` 做什么？
23. `-w` 做什么？
24. `--nmap` 做什么？
25. `-u` 做什么？
26. 为什么不能直接运行 Mirrored 文件？
27. SHA-256 能证明代码安全吗？
28. 为什么检查外部 URL？
29. 为什么检查架构？
30. 为什么 x86 Exploit 不一定适合 ARM64？
31. 为什么旧版本字符串不一定代表未修复？
32. 为什么 Title 不足以判断适用性？
33. SearchSploit 没结果是否证明没有漏洞？
34. 很多结果时如何缩小？
35. Wazuh 为什么通常看不到 Kali 上的 SearchSploit？
36. `--nmap` 是否会重新扫描目标？
37. 为什么不在共享文件夹保存未知 PoC？
38. 为什么不以 root 研究 PoC？
39. 为什么公开 PoC 不一定安全？
40. 什么条件下才考虑建立专用靶场？

---

# 123. 参考答案

1. 公开 Exploit 和 Proof-of-Concept 档案库。
2. 不会，普通搜索只读取本地数据库。
3. 数据和工具都可保存在本地。
4. EDB-ID 是 Exploit-DB 记录编号；CVE 是漏洞标识符。
5. Exploit 利用漏洞；Payload 是利用后执行的动作。
6. 不是。
7. 已有本地低权限访问。
8. 不一定，可能需要账号或特定配置。
9. 它可能影响服务可用性。
10. AND。
11. Title 和本地文件 Path。
12. 只搜索 Title。
13. 按顺序精确匹配 Title。
14. 使用严格版本匹配，关闭模糊版本范围。
15. 区分大小写。
16. 移除包含指定词的结果。
17. 按 CVE 搜索。
18. 显示 EDB-ID。
19. 显示 Exploit 本地完整路径。
20. 使用 PAGER 静态查看文件。
21. 复制文件到当前目录。
22. 输出 JSON。
23. 显示 Exploit-DB 在线 URL。
24. 从已有 Nmap XML 中提取服务信息并搜索。
25. 检查和安装数据库更新。
26. 未知代码可能破坏系统或包含恶意行为。
27. 不能，只证明文件内容标识。
28. 代码可能连接、下载或泄露数据。
29. Exploit、Shellcode 和 Offset 可能只适合特定 CPU。
30. 指令集和二进制格式不同。
31. Linux 发行版可能回移安全补丁。
32. 详细前置条件常写在代码和注释中。
33. 不能。
34. 加版本、平台、类型，并使用 Title、Strict、Exact 和 Exclude。
35. Kali 当前没有 Wazuh Agent，行为也发生在本地。
36. 不会，它只读取 XML。
37. 避免同步到宿主机和其他设备。
38. 遵守最小权限并降低误操作影响。
39. 公开文件可能不稳定、恶意或已经被篡改。
40. 授权、版本匹配、隔离、快照和回滚计划都明确时。

---

# 124. 命令速查表

## 帮助

```bash
searchsploit -h
```

## 基础搜索

```bash
searchsploit apache 2.4
```

## Title

```bash
searchsploit -t apache 2.4
```

## Exact

```bash
searchsploit -e "WordPress 4.1"
```

## Strict

```bash
searchsploit -s "Apache Struts 2.0.0"
```

## Exclude

```bash
searchsploit linux kernel 5.15 \
  --exclude='(PoC)|/dos/'
```

## CVE

```bash
searchsploit --cve 2021-41773
```

## EDB-ID 输出

```bash
searchsploit --id apache 2.4.49
```

## JSON

```bash
searchsploit -j apache 2.4.49
```

## Path

```bash
searchsploit -p <EDB-ID>
```

## Examine

```bash
searchsploit -x <EDB-ID>
```

## Mirror

```bash
searchsploit -m <EDB-ID>
```

## Online URL

```bash
searchsploit -w apache 2.4.49
```

## Nmap XML

```bash
searchsploit --nmap scan.xml
```

## Update

```bash
searchsploit -u
```

---

# 125. 核心记忆口诀

```text
先确认版本
再搜索 Exploit

普通搜索：Title + Path
-t：只看 Title
-e：精确顺序
-s：严格版本
--exclude：去掉噪音
--cve：按 CVE
--id：显示 EDB-ID

-p：找路径
-x：只阅读
-m：只复制
-j：JSON
-w：网页地址
--nmap：读 XML
-u：更新数据库

有 PoC 不等于可利用
版本相同不等于补丁相同
平台相同不等于架构相同
公开代码不等于安全代码

不执行未知 PoC
不运行 DoS
不在共享目录保存
不使用 root
```

---

# 126. 完成标准

完成本课后，你应该拥有：

```text
Exploit-DB 版本记录
基础搜索结果
Title/Strict/Exact 对比
Exclude 结果
CVE 搜索
JSON 输出
一个静态阅读案例
Mirrored 文件 Hash
静态审查笔记
Ubuntu Nmap XML
SearchSploit --nmap 结果
ARM64 适用性分析
离线流量证明
Wazuh searchsploit-lab Marker
适用性矩阵
Public Exploit Research 报告
```

检查：

```bash
find ~/CyberLab/SearchSploit \
  -type f |
sort
```

---

# 127. 下一课

下一课建议：

```text
Metasploit Framework
```

下一课学习内容：

- Metasploit 架构
- `msfconsole`
- Module 类型
- `search`
- `info`
- `show options`
- `show missing`
- `set` 和 `unset`
- Workspace
- Database
- Auxiliary Scanner
- Exploit、Payload、Session 概念
- `check` 优先于 `run`
- 只使用专门的本地靶场
- Handler 与 Payload 网络流量
- Wazuh 检测和日志观察
- 不对当前生产式服务直接执行 Exploit

当前学习关系：

```text
Nmap：发现服务
Netcat：测试连接
Wireshark：图形化抓包
tcpdump：命令行抓包
curl：手动 HTTP 请求
WhatWeb：技术识别
Gobuster/ffuf：内容发现
Nikto：Web Server 检查
Burp Suite：手工代理分析
OWASP ZAP：代理、扫描和自动化
SQLmap：SQL Injection 检测
SearchSploit：离线公开 Exploit 研究
```

---

# 128. 参考说明

本指南依据：

```text
Exploit-DB 官方 SearchSploit Manual
Exploit-DB 官方 Repository
Kali Linux exploitdb 工具页面
本机 searchsploit -h 输出
```

当前官方功能包括：

```text
多关键词 AND 搜索
Title、Exact、Strict 和 Case 搜索
Exclude 和 CVE 搜索
JSON、Path、Online URL 和 EDB-ID 输出
Examine、Mirror、Update
Nmap XML 联动
```

工具和数据库会更新，最终以你的 Kali 输出为准：

```bash
searchsploit -h
apt policy exploitdb
```
