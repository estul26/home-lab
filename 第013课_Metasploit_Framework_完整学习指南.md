# 第十三课：Metasploit Framework 完整学习指南

> **实验环境：你自己的 CyberLab**
>
> 本课重点是 Metasploit Framework 的安全基础：模块搜索、数据库、Workspace、辅助扫描、结果管理和静态分析。  
> **不对当前 Ubuntu、Windows 或 Wazuh 系统执行 Exploit，不生成恶意 Payload，不建立未授权 Session。**

---

# 1. 本课目标

完成本课后，你应该能够：

```text
理解 Metasploit Framework 的架构和用途
区分 Framework、Pro、msfconsole、msfdb 和模块
理解 Exploit、Auxiliary、Payload、Post、Encoder 和 NOP
在 Kali 中安装、启动和更新 Metasploit Framework
初始化并检查 PostgreSQL 数据库
使用 Workspace 隔离 CyberLab 数据
导入 Nmap XML 或使用 db_nmap
查看 Hosts、Services、Vulns、Notes、Loot 和 Creds
使用 search、info、use、show options 和 show missing
理解 RHOSTS、RPORT、LHOST、LPORT、TARGETURI、SSL 和 VHOST
使用 set、setg、unset、unsetg 和 save
使用安全的 Auxiliary Scanner 检查 HTTP 和 SSH
把模块作为前台任务或后台 Job 运行
理解 check、run、exploit 和 sessions 的区别
理解 Payload、Handler、Shell 和 Meterpreter 的概念
理解 Exploit Ranking 和 Target Compatibility
认识 Apple Silicon ARM64 的兼容性限制
创建并运行安全的 Resource Script
创建一个只读取 HTTP Header 的自定义 Auxiliary Module
使用 tcpdump、Wireshark、Access Log 和 Wazuh 观察扫描
完成一份 Metasploit Framework Laboratory Report
```

CyberLab：

```text
Kali-CyberLab      10.10.20.10
Ubuntu-Target      10.10.20.20
Windows-Target     10.10.20.30
Wazuh-SIEM         10.10.20.40
Network            10.10.20.0/24
```

本课主要通信：

```text
Metasploit：Kali 10.10.20.10
Ubuntu SSH：10.10.20.20:22
Ubuntu HTTP Lab：10.10.20.20:8091
Wazuh：10.10.20.40
```

---

# 2. 什么是 Metasploit Framework？

Metasploit Framework 是一个开源安全研究和渗透测试平台。

它把许多安全任务组织成统一模块，例如：

- 服务扫描
- 版本识别
- 漏洞检测
- Exploit 验证
- Payload 管理
- Session 管理
- Post-Exploitation
- 数据库记录
- 自动化脚本
- 自定义模块开发

核心结构：

```text
msfconsole
    ↓
选择 Module
    ↓
配置 Options
    ↓
check / run / exploit
    ↓
Database / Job / Session / Result
```

---

# 3. Framework 与 Metasploit Pro

## Metasploit Framework

```text
免费
开源
主要通过 msfconsole 使用
适合研究、实验和手动工作流
```

## Metasploit Pro

```text
商业产品
提供 Web UI
自动化工作流
团队协作
报告和项目管理
更多商业功能
```

本课使用：

```text
Metasploit Framework
msfconsole
```

不要把 Pro 的网页功能和 Framework 命令混在一起。

---

# 4. Metasploit 与前面工具的关系

```text
Nmap：发现端口和服务
Netcat：手工连接与 Banner
WhatWeb：识别 Web 技术
Gobuster/ffuf：内容发现
Nikto：Web Server 检查
Burp/ZAP：代理和手工请求分析
SQLmap：SQL Injection 自动检测
SearchSploit：离线公开 PoC 研究
Metasploit：统一管理模块、扫描、验证、Payload 和 Session
```

推荐顺序：

```text
发现
→ 验证版本
→ 研究 CVE
→ SearchSploit / Metasploit 搜索
→ 阅读 Module 信息
→ check
→ 只在专用脆弱靶场决定是否验证
```

---

# 5. 安全边界

本课允许：

```text
搜索和阅读模块
初始化本地数据库
创建 Workspace
导入自己的 Nmap XML
扫描 10.10.20.20 的 22 和 8091
读取 HTTP 标题、Header 和 SSH Banner
运行自己编写的只读 Auxiliary Module
观察流量和日志
```

本课不允许：

```text
对当前 Windows、Ubuntu 或 Wazuh 运行 Exploit
使用 Credential Brute Force
生成可执行 Payload
创建 Reverse Shell
启动 Meterpreter Session
执行 Post-Exploitation
Dump Password Hash
持久化
权限提升
横向移动
关闭安全软件
DoS Module
自动 Exploit Resource Script
```

---

# 6. 为什么当前目标不直接用于 Exploit？

你的 Ubuntu、Windows 和 Wazuh 是正常实验基础设施，不是一次性脆弱靶机。

直接执行 Exploit 可能：

- 破坏系统
- 使 Wazuh 停止工作
- 损坏 Agent
- 修改配置
- 造成数据丢失
- 让后续课程无法继续
- 产生难以回滚的状态

正确方式：

```text
当前机器：扫描、日志和防御观察
专用克隆靶机：未来进行版本匹配的 Exploit 验证
```

---

# 7. 模块类型

Metasploit 的主要模块类型：

| 类型 | 用途 |
|---|---|
| `exploit` | 利用特定漏洞 |
| `auxiliary` | 扫描、枚举、协议操作和其他非 Payload 任务 |
| `payload` | Exploit 成功后运行的代码或命令 |
| `post` | 已经取得 Session 后执行的后续模块 |
| `encoder` | 对 Payload 数据进行编码变换 |
| `nop` | 生成 No Operation 指令 |
| `evasion` | 某些平台的规避型可执行构建模块 |

本课主要使用：

```text
auxiliary
```

---

# 8. Auxiliary 不一定无风险

Auxiliary Module 可能包括：

- 安全扫描器
- 版本识别
- 协议枚举
- 登录测试
- Fuzzer
- DoS
- 数据收集

所以：

```text
Auxiliary
≠
自动安全
```

运行前必须阅读：

```text
info
Description
Actions
Options
References
Notes
```

本课只使用只读 Scanner。

---

# 9. Exploit 与 Payload

## Exploit

利用漏洞进入目标代码执行流程。

## Payload

Exploit 成功后执行的动作。

关系：

```text
Exploit 负责打开入口
Payload 负责执行后续功能
```

有些 Exploit：

- 不需要单独 Payload
- 只造成 Crash
- 只执行一个命令
- 只能配合特定平台 Payload

---

# 10. Shell 与 Meterpreter

## Command Shell

通常是目标系统命令解释器。

例如：

```text
/bin/sh
cmd.exe
PowerShell
```

## Meterpreter

Metasploit 的高级 Session 类型，可通过扩展提供更多能力。

本课只学习概念：

```text
不建立 Shell
不建立 Meterpreter
```

---

# 11. Handler

Handler 是等待 Payload 回连或接收连接的组件。

常见模块：

```text
exploit/multi/handler
```

Handler 本身不利用漏洞。

但它通常与 Payload 配合，因此本课只查看：

```text
info
show options
show payloads
```

不启动监听，不生成 Payload。

---

# 12. Exploit Ranking

Metasploit Exploit Module 常见 Ranking：

```text
Manual
Low
Average
Normal
Good
Great
Excellent
```

Ranking 主要反映模块的可靠性和对目标的潜在破坏风险。

重要：

```text
Excellent 不代表绝对安全
Low 不代表一定失败
Ranking 不能替代版本和配置验证
```

---

# 13. ARM64 的重要限制

你的 CyberLab 运行在 Apple Silicon。

主要 Guest：

```text
Kali ARM64
Ubuntu ARM64
Windows ARM64
```

许多传统 Exploit 和 Payload 面向：

```text
x86
x64
```

运行模块前检查：

- Platform
- Architecture
- Target
- Payload Compatibility
- Operating System Build
- Service Version
- Module Notes

命令：

```text
info
show targets
show payloads
```

不能因为目标是 Windows 或 Linux，就假设模块支持 ARM64。

---

# 14. 检查和安装 Metasploit

Kali：

```bash
which msfconsole
```

查看版本：

```bash
msfconsole --version
```

查看软件包：

```bash
apt policy metasploit-framework
```

安装：

```bash
sudo apt update
sudo apt install metasploit-framework
```

Kali 当前软件包名称：

```text
metasploit-framework
```

---

# 15. 更新 Metasploit

在 Kali 有临时互联网时：

```bash
sudo apt update
sudo apt install --only-upgrade metasploit-framework
```

查看版本：

```bash
msfconsole --version
```

不要为了更新给 Ubuntu、Windows 或 Wazuh Target 长期开启 Shared Network。

---

# 16. 创建实验目录

Kali：

```bash
mkdir -p ~/CyberLab/Metasploit/{nmap,results,resources,modules,captures,notes,reports}
```

Ubuntu：

```bash
mkdir -p ~/CyberLab/Metasploit/{server,logs}
```

检查：

```bash
find ~/CyberLab/Metasploit -maxdepth 2 -type d
```

---

# 17. Ubuntu 安全 HTTP 实验服务

本实验服务只提供：

```text
HTTP Title
Server Header
自定义 Header
robots.txt
health
admin 页面
404
Access Log
```

它不是脆弱应用，不应被 Exploit。

---

# 18. 创建 Ubuntu HTTP Server

Ubuntu：

```bash
nano ~/CyberLab/Metasploit/server/metasploit_lab.py
```

粘贴：

```python
#!/usr/bin/env python3

import json
from http.server import BaseHTTPRequestHandler, HTTPServer
from urllib.parse import urlparse


HOST = "10.10.20.20"
PORT = 8091


def html_page(title: str, body: str) -> bytes:
    return f"""<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>{title}</title>
</head>
<body>
  <h1>{title}</h1>
  <p>{body}</p>
</body>
</html>
""".encode("utf-8")


class MetasploitLabHandler(BaseHTTPRequestHandler):
    server_version = "CyberLabHTTP/1.0"
    sys_version = "(Ubuntu-ARM64)"

    def send_content(
        self,
        status: int,
        body: bytes,
        content_type: str = "text/html; charset=utf-8",
    ) -> None:
        self.send_response(status)
        self.send_header("Content-Type", content_type)
        self.send_header("Content-Length", str(len(body)))
        self.send_header("X-CyberLab", "Metasploit-Auxiliary-Lab")
        self.send_header("X-Lab-Owner", "Authorized-Student")
        self.end_headers()

        if self.command != "HEAD":
            self.wfile.write(body)

    def do_GET(self) -> None:
        path = urlparse(self.path).path

        if path == "/":
            self.send_content(
                200,
                html_page(
                    "CyberLab Metasploit",
                    "Safe auxiliary-scanner training service.",
                ),
            )
            return

        if path == "/health":
            self.send_content(
                200,
                json.dumps(
                    {
                        "status": "healthy",
                        "service": "metasploit-lab",
                    },
                    indent=2,
                ).encode("utf-8"),
                "application/json",
            )
            return

        if path == "/robots.txt":
            self.send_content(
                200,
                b"User-agent: *\nDisallow: /admin/\n",
                "text/plain; charset=utf-8",
            )
            return

        if path in {"/admin", "/admin/"}:
            self.send_content(
                403,
                b"CyberLab admin area is forbidden\n",
                "text/plain; charset=utf-8",
            )
            return

        self.send_content(
            404,
            b"CyberLab Metasploit resource not found\n",
            "text/plain; charset=utf-8",
        )

    def do_HEAD(self) -> None:
        self.do_GET()

    def log_message(self, format: str, *args: object) -> None:
        user_agent = self.headers.get("User-Agent", "-")

        print(
            f'{self.client_address[0]} '
            f'[{self.log_date_time_string()}] '
            f'UA="{user_agent}" '
            f'{format % args}',
            flush=True,
        )


def main() -> None:
    server = HTTPServer(
        (HOST, PORT),
        MetasploitLabHandler,
    )

    print(f"Metasploit lab listening on http://{HOST}:{PORT}")
    server.serve_forever()


if __name__ == "__main__":
    main()
```

保存：

```text
Ctrl + O
Enter
Ctrl + X
```

权限：

```bash
chmod +x ~/CyberLab/Metasploit/server/metasploit_lab.py
```

---

# 19. 启动 HTTP Server

Ubuntu：

```bash
python3 \
  ~/CyberLab/Metasploit/server/metasploit_lab.py \
  2>&1 |
tee ~/CyberLab/Metasploit/logs/access.log
```

预期：

```text
Metasploit lab listening on http://10.10.20.20:8091
```

第二个 Ubuntu 终端：

```bash
ss -lntp | grep ':8091'
```

---

# 20. Kali 预检查

```bash
ping -c 3 10.10.20.20
```

```bash
nc -zvn -w 3 10.10.20.20 22
```

```bash
nc -zvn -w 3 10.10.20.20 8091
```

```bash
curl -i http://10.10.20.20:8091/
```

```bash
sudo nmap -n -sV -p 22,8091 10.10.20.20
```

---

# 21. 初始化 Metasploit 数据库

Kali 终端：

```bash
sudo msfdb init
```

部分安装也可由普通用户运行：

```bash
msfdb init
```

查看帮助：

```bash
msfdb --help
```

检查：

```bash
msfdb status
```

不要使用：

```bash
msfdb reinit
```

除非你明确要删除并重新创建已有数据库。

---

# 22. 数据库的作用

Metasploit Framework 使用 PostgreSQL 保存：

- Hosts
- Services
- Vulnerabilities
- Notes
- Loot
- Credentials
- Exploit Results
- Workspace 数据

没有数据库时，模块仍可能运行，但你会失去集中数据管理能力。

---

# 23. 启动 msfconsole

```bash
msfconsole
```

安静启动：

```bash
msfconsole -q
```

查看帮助：

```text
help
```

退出：

```text
exit
```

不要使用 `Ctrl + Z` 作为正常退出方式。

---

# 24. 检查数据库连接

在 msfconsole：

```text
db_status
```

预期类似：

```text
Connected to msf
```

如果未连接：

```text
[*] postgresql selected, no connection
```

先退出并检查：

```bash
msfdb status
```

---

# 25. Workspace

列出：

```text
workspace
```

创建：

```text
workspace -a CyberLab-Metasploit
```

切换：

```text
workspace CyberLab-Metasploit
```

当前 Workspace 前通常会有：

```text
*
```

Workspace 用于隔离不同实验数据。

---

# 26. Workspace 命名建议

```text
CyberLab-Metasploit
Web-Lab
Windows-Lab
Ubuntu-Lab
Course-YYYY-MM-DD
```

不要把：

```text
个人实验
学校授权测试
客户项目
```

混在同一个 Workspace。

---

# 27. 删除 Workspace

先确认当前 Workspace：

```text
workspace
```

删除：

```text
workspace -d OLD-WORKSPACE
```

删除可能移除其中保存的数据。

本课不要删除：

```text
CyberLab-Metasploit
```

---

# 28. 使用 db_nmap

在 msfconsole：

```text
db_nmap -n -sV -p 22,8091 10.10.20.20
```

它会：

```text
调用 Nmap
显示结果
把 Host 和 Service 保存到当前 Workspace
```

只扫描明确端口，不扫描整个网段。

---

# 29. 查看数据库 Host

```text
hosts
```

只看地址：

```text
hosts -c address
```

查看帮助：

```text
hosts -h
```

导出 CSV：

```text
hosts -o /home/kali/CyberLab/Metasploit/results/hosts.csv
```

---

# 30. 查看 Services

```text
services
```

筛选 10.10.20.20：

```text
services -R 10.10.20.20
```

如果当前版本参数不同：

```text
services -h
```

你应该看到：

```text
22/tcp
8091/tcp
```

---

# 31. Vulns、Notes、Loot 和 Creds

查看漏洞记录：

```text
vulns
```

查看 Notes：

```text
notes
```

查看 Loot：

```text
loot
```

查看 Credential 数据：

```text
creds
```

本课预期：

```text
Vulns 可能为空
Loot 为空
Creds 为空
```

这很正常。

---

# 32. 外部 Nmap XML

Kali 普通终端：

```bash
sudo nmap \
  -n \
  -sV \
  -p 22,8091 \
  10.10.20.20 \
  -oA ~/CyberLab/Metasploit/nmap/ubuntu-baseline
```

生成：

```text
ubuntu-baseline.nmap
ubuntu-baseline.xml
ubuntu-baseline.gnmap
```

---

# 33. 导入 Nmap XML

msfconsole：

```text
db_import /home/kali/CyberLab/Metasploit/nmap/ubuntu-baseline.xml
```

再次查看：

```text
hosts
services
```

不要重复导入大量相同扫描，避免数据混乱。

---

# 34. db_nmap 与 db_import

| `db_nmap` | `db_import` |
|---|---|
| 从 msfconsole 直接运行 Nmap | 导入已有扫描文件 |
| 一步完成扫描和保存 | 可保留外部 Nmap 工作流 |
| 适合快速实验 | 适合证据归档 |
| 命令记录在 Metasploit | XML 可以独立审查 |

本课建议两种都练习一次。

---

# 35. 搜索模块

基础：

```text
search http
```

查看搜索帮助：

```text
search -h
```

按类型：

```text
search type:auxiliary http
```

按名称：

```text
search type:auxiliary name:http_version
```

按平台：

```text
search type:exploit platform:windows
```

按 CVE：

```text
search cve:2021-41773
```

---

# 36. 常见搜索操作符

常见搜索条件：

```text
name:
path:
platform:
type:
app:
author:
cve:
bid:
osdvb:
```

当前版本可能支持更多条件。

始终查看：

```text
search -h
```

---

# 37. 搜索结果列

结果通常包含：

```text
#
Name
Disclosure Date
Rank
Check
Description
```

其中：

- `Name`：模块完整路径
- `Rank`：可靠性等级
- `Check`：是否实现 Check Method
- `Description`：模块用途

不要只看 Description 就运行。

---

# 38. 使用模块编号

搜索后可能显示：

```text
0 auxiliary/scanner/http/http_version
```

可以：

```text
use 0
```

更推荐记录完整路径：

```text
use auxiliary/scanner/http/http_version
```

编号只对当前搜索结果有效。

---

# 39. 模块 info

不加载模块：

```text
info auxiliary/scanner/http/http_version
```

加载后：

```text
info
```

重点查看：

- Name
- Module
- Platform
- Architecture
- Rank
- Description
- Authors
- Options
- Actions
- References
- Notes
- Reliability
- Stability
- Side Effects

---

# 40. use 和 back

加载：

```text
use auxiliary/scanner/http/http_version
```

Prompt 会改变。

退出模块：

```text
back
```

回到前一个模块：

```text
previous
```

如果当前版本不支持 `previous`，使用完整 `use` 路径重新加载。

---

# 41. show options

```text
show options
```

通常显示：

```text
Name
Current Setting
Required
Description
```

Required 为：

```text
yes
```

但没有值时，模块不能正常运行。

---

# 42. show missing

```text
show missing
```

用于显示尚未配置的必需选项。

如果当前版本或模块不支持：

```text
show options
```

并人工检查 Required 列。

---

# 43. show advanced

```text
show advanced
```

Advanced Options 可能影响：

- Timeout
- Proxy
- HTTP Header
- Redirect
- Scanner 行为
- Debug

不要修改不了解的 Advanced Option。

---

# 44. 常见选项

| 选项 | 含义 |
|---|---|
| `RHOSTS` | 一个或多个远程目标 |
| `RHOST` | 单个远程目标，视模块而定 |
| `RPORT` | 远程端口 |
| `LHOST` | 本地监听或回连地址 |
| `LPORT` | 本地监听端口 |
| `TARGETURI` | Web 应用根路径 |
| `SSL` | 是否使用 TLS |
| `VHOST` | HTTP Host Header |
| `THREADS` | Scanner 并发数 |
| `SESSION` | Post Module 使用的 Session |
| `TARGET` | Exploit Target 配置 |

本课只使用：

```text
RHOSTS
RPORT
TARGETURI
SSL
THREADS
```

---

# 45. set

```text
set RHOSTS 10.10.20.20
```

```text
set RPORT 8091
```

```text
set THREADS 1
```

查看单个值：

```text
get RHOSTS
```

清除：

```text
unset RHOSTS
```

---

# 46. setg 和 unsetg

Global Option：

```text
setg RHOSTS 10.10.20.20
```

```text
setg THREADS 1
```

查看：

```text
getg RHOSTS
```

清除：

```text
unsetg RHOSTS
```

危险点：

```text
Global RHOSTS 可能在切换模块后继续存在
```

每次运行前都检查：

```text
show options
```

---

# 47. 保存设置

在 msfconsole：

```text
save
```

这可以保存某些全局配置到 Metasploit 配置文件。

不要长期保存：

- 客户 IP
- Credential
- Payload 地址
- 不再使用的 Global Option

课程结束时：

```text
unsetg RHOSTS
unsetg RPORT
unsetg LHOST
unsetg LPORT
```

---

# 48. run 与 exploit

Auxiliary 通常使用：

```text
run
```

Exploit Module 常见：

```text
run
```

或：

```text
exploit
```

现代 msfconsole 中二者在 Exploit Module 中通常作用相近。

本课：

```text
只对 Auxiliary Scanner 使用 run
```

---

# 49. check

部分 Exploit Module 实现：

```text
check
```

它用于判断目标是否：

- Vulnerable
- Appears Vulnerable
- Safe
- Detected
- Unknown

重要：

```text
不是所有模块都有 check
check 也可能发送特殊测试请求
check 结果不是绝对证明
```

本课只阅读支持 Check 的模块，不对当前系统运行漏洞 Check。

---

# 50. check 优先原则

未来专用靶场工作流：

```text
info
→ show targets
→ show options
→ show missing
→ check
→ 人工验证
→ 快照
→ 才决定是否 run/exploit
```

不要：

```text
搜索到模块后直接 exploit
```


---

# 51. 安全实验一：HTTP Version Scanner

msfconsole：

```text
use auxiliary/scanner/http/http_version
```

配置：

```text
set RHOSTS 10.10.20.20
set RPORT 8091
set SSL false
set THREADS 1
```

检查：

```text
show options
show missing
```

运行：

```text
run
```

可能识别：

```text
CyberLabHTTP/1.0
Ubuntu-ARM64
HTTP Response
```

实际输出取决于当前 Metasploit Module 版本。

---

# 52. 使用 run -j 作为后台 Job

同一模块：

```text
run -j
```

查看：

```text
jobs
```

详细：

```text
jobs -l
```

停止某个 Job：

```text
jobs -k JOB_ID
```

停止全部 Job：

```text
jobs -K
```

本实验扫描很快，Job 可能在你查看前已经结束。

---

# 53. HTTP Title Scanner

搜索：

```text
search type:auxiliary name:title http
```

加载当前安装显示的准确模块。

常见路径：

```text
use auxiliary/scanner/http/title
```

配置：

```text
set RHOSTS 10.10.20.20
set RPORT 8091
set TARGETURI /
set SSL false
set THREADS 1
```

运行：

```text
run
```

预期标题：

```text
CyberLab Metasploit
```

---

# 54. TARGETURI

扫描首页：

```text
set TARGETURI /
```

扫描 health：

```text
set TARGETURI /health
```

扫描 robots：

```text
set TARGETURI /robots.txt
```

每次修改后：

```text
show options
run
```

TARGETURI 通常表示：

```text
目标 Web 应用或资源路径
```

---

# 55. SSL

HTTP 8091：

```text
set SSL false
```

HTTPS 服务：

```text
set SSL true
```

不要因为端口是 443 就完全假设：

```text
一定是 HTTPS
```

也不要因为是非标准端口就假设：

```text
一定是 HTTP
```

先用：

```bash
curl -vkI
nmap -sV
```

验证。

---

# 56. VHOST

如果一个 IP 上有多个虚拟站点：

```text
set VHOST app.lab.local
```

这通常影响：

```http
Host: app.lab.local
```

你的 8091 实验服务不要求 VHOST。

运行结束后：

```text
unset VHOST
```

---

# 57. HTTP Header 模块

搜索：

```text
search type:auxiliary http header
```

阅读候选模块：

```text
info MODULE_PATH
```

不同版本可能提供不同 HTTP Header Scanner。

选择前确认：

- 只读取响应
- 不进行 Fuzz
- 不发送破坏性 Method
- 支持当前 RPORT 和 TARGETURI

如果不确定，跳过并使用本课的自定义只读模块。

---

# 58. 安全实验二：SSH Version Scanner

搜索：

```text
search type:auxiliary name:ssh_version
```

常见模块：

```text
use auxiliary/scanner/ssh/ssh_version
```

配置：

```text
set RHOSTS 10.10.20.20
set RPORT 22
set THREADS 1
```

运行：

```text
run
```

该模块主要识别：

- SSH Banner
- Server Version
- 部分支持的加密算法线索
- 协议能力

它不需要登录账号。

---

# 59. SSH Version 结果不是漏洞证明

例如结果显示：

```text
OpenSSH 8.x
```

仍需考虑：

- Ubuntu 软件包 Revision
- Backported Security Patch
- Server Configuration
- Enabled Algorithms
- Authentication Policy
- CVE 条件
- Architecture

正确写法：

```text
Detected SSH version indication
Requires package and patch verification
```

---

# 60. 不使用 SSH Login Scanner

Metasploit 包含 Credential Login Scanner。

本课不使用：

```text
ssh_login
smb_login
ftp_login
http_login
```

原因：

- 可能触发账号锁定
- 可能形成密码攻击
- 会污染 Credential 数据库
- 不属于当前基础课程

Hydra 课程会单独讲解受控认证测试。

---

# 61. 安全实验三：TCP Port Scanner

搜索：

```text
search type:auxiliary name:portscan
```

常见模块：

```text
use auxiliary/scanner/portscan/tcp
```

查看：

```text
info
show options
```

只配置：

```text
set RHOSTS 10.10.20.20
set PORTS 22,8091
set THREADS 1
```

运行：

```text
run
```

不要配置：

```text
1-65535
10.10.20.0/24
```

本课已经知道目标和端口。

---

# 62. Metasploit Scanner 与 Nmap

| Metasploit Auxiliary | Nmap |
|---|---|
| 与模块工作流集成 | 专业端口和服务扫描器 |
| 结果可进入 Framework 数据库 | 扫描能力和指纹通常更全面 |
| 可按协议使用专用模块 | NSE 生态丰富 |
| 适合后续模块验证 | 适合最初发现和基线 |

推荐：

```text
Nmap 负责发现
Metasploit Auxiliary 负责协议级补充
```

---

# 63. 模块输出保存

msfconsole 中启用 Spool：

```text
spool /home/kali/CyberLab/Metasploit/results/msfconsole-session.log
```

执行实验。

停止 Spool：

```text
spool off
```

检查：

```bash
less ~/CyberLab/Metasploit/results/msfconsole-session.log
```

Spool 可能包含敏感信息。

---

# 64. hosts 导出

msfconsole：

```text
hosts -o /home/kali/CyberLab/Metasploit/results/hosts.csv
```

检查：

```bash
column -s, -t \
  ~/CyberLab/Metasploit/results/hosts.csv |
less
```

不同版本 CSV 格式可能不同。

---

# 65. services 导出

查看帮助：

```text
services -h
```

如果当前版本支持 `-o`：

```text
services -o /home/kali/CyberLab/Metasploit/results/services.csv
```

如果不支持，使用 Spool：

```text
spool /home/kali/CyberLab/Metasploit/results/services.txt
services
spool off
```

---

# 66. 数据库重复数据

重复运行：

```text
db_nmap
db_import
scanner module
```

可能更新或重复部分记录。

每个实验使用清晰 Workspace：

```text
CyberLab-Metasploit
```

重大实验前可创建：

```text
workspace -a CyberLab-Metasploit-Retest
```

不要依赖删除数据来整理所有实验。

---

# 67. Jobs

Job 是后台执行的模块任务。

常用：

```text
jobs
jobs -l
jobs -k ID
jobs -K
```

适合：

- 长时间 Scanner
- Handler
- 某些 Server Module
- 同时执行多个授权任务

本课最多运行：

```text
一个安全 Scanner Job
```

---

# 68. Session

Session 是成功连接到目标后的交互通道。

可能类型：

- Command Shell
- Meterpreter
- PowerShell
- SSH Session
- VNC
- 其他模块 Session

查看：

```text
sessions
```

详细：

```text
sessions -l
```

本课预期：

```text
No active sessions
```

---

# 69. Session 命令概念

未来专用靶场可能使用：

```text
sessions -i SESSION_ID
background
sessions -k SESSION_ID
```

含义：

- `-i`：进入 Session
- `background`：返回 msfconsole
- `-k`：关闭 Session

本课不创建 Session，因此不要执行交互操作。

---

# 70. Session 与 Job 的区别

| Job | Session |
|---|---|
| 后台运行的任务 | 与目标建立的通道 |
| Scanner、Handler 等 | Shell、Meterpreter 等 |
| `jobs` 管理 | `sessions` 管理 |
| Job 结束不代表 Session 结束 | Session 可由 Job 创建 |

例如：

```text
Handler Job 可以继续运行
并等待新的 Session
```

---

# 71. Payload 类型概念

Metasploit Payload 可大致分为：

```text
Singles
Stagers
Stages
```

## Single

完整功能放在一个 Payload 中。

## Stager

建立初始小型通信通道。

## Stage

通过 Stager 后续传输较大功能。

名称中常见：

```text
shell_reverse_tcp
shell/reverse_tcp
```

是否为 Staged Payload 取决于路径和平台。

---

# 72. Reverse 与 Bind

## Reverse

目标主动连接回测试机：

```text
Target → Kali
```

通常需要：

```text
LHOST
LPORT
```

## Bind

目标监听端口，测试机连接目标：

```text
Kali → Target
```

通常需要：

```text
RHOST
RPORT
```

本课不启动任何 Reverse 或 Bind Payload。

---

# 73. LHOST 不是 RHOSTS

```text
RHOSTS = 目标
LHOST  = 测试机用于接收连接的地址
```

你的 Kali CyberLab 地址：

```text
10.10.20.10
```

但不要在当前课程中保存为 Global LHOST。

多网卡 Kali 可能还有其他地址，错误 LHOST 会让连接失败或走错网络。

---

# 74. 查看兼容 Payload

加载一个 Exploit Module 后：

```text
show payloads
```

这只显示与当前 Exploit 兼容的 Payload。

不要直接使用：

```text
show payloads
→ 选第一个
```

还必须检查：

- Target OS
- Architecture
- Network Direction
- Required Privilege
- Size
- Bad Characters
- Session Type
- Egress Control

---

# 75. 查看 Exploit Targets

```text
show targets
```

常见 Target：

- Automatic
- Windows x86
- Windows x64
- Linux x64
- Specific Version
- Specific Language
- Specific Build

设置：

```text
set TARGET NUMBER
```

本课只阅读，不设置和运行 Exploit Target。

---

# 76. Handler 静态查看

```text
use exploit/multi/handler
```

查看：

```text
info
show options
show payloads
```

然后：

```text
back
```

不要：

```text
set PAYLOAD ...
run
exploit -j
```

本课不需要 Listener。

---

# 77. Exploit Module 安全研究流程

SearchSploit 找到 CVE 后：

```text
search cve:CVE-YEAR-NUMBER
```

对结果：

```text
info MODULE_PATH
```

记录：

- Disclosure Date
- Rank
- Check Support
- Platform
- Architecture
- Targets
- References
- Required Options
- Payloads
- Side Effects
- Reliability Notes
- Stability Notes

不要加载后直接运行。

---

# 78. 只读 CVE Module Case Study

例如研究一个公开 CVE：

```text
search cve:2021-41773
```

选择结果后：

```text
info MODULE_PATH
```

只回答：

```text
Module Type 是什么？
Rank 是什么？
是否支持 check？
目标软件和版本是什么？
需要哪些设置？
是否会执行 Payload？
有哪些 Side Effects？
是否适合 ARM64？
```

不设置当前 Ubuntu 为 RHOSTS。

---

# 79. Check Code 含义

支持 Check 的模块可能返回：

```text
Vulnerable
Appears
Detected
Safe
Unknown
Unsupported
```

这些状态通常基于：

- Banner
- Version
- 特定请求
- 行为差异
- 文件或路径
- 协议响应

`Appears` 通常不等于：

```text
Confirmed Exploitable
```

---

# 80. Side Effects 和 Notes

现代模块信息可能显示：

```text
Notes
Stability
Reliability
Side Effects
```

常见风险概念：

- Crash Service
- Crash OS
- IOC in Logs
- Artifact on Disk
- Account Lockout
- Service Restart
- Configuration Change

如果 Module 可能产生副作用：

```text
不要在当前基础设施运行
```

---

# 81. Resource Script

Resource Script 是一组顺序执行的 msfconsole 命令。

扩展名通常：

```text
.rc
```

用途：

- 重复扫描
- 设置 Workspace
- 配置 Module
- 保存一致实验流程
- 避免手工输入错误

---

# 82. 创建安全 Resource Script

Kali 普通终端：

```bash
cat > ~/CyberLab/Metasploit/resources/http-title.rc <<'EOF'
workspace CyberLab-Metasploit
use auxiliary/scanner/http/title
set RHOSTS 10.10.20.20
set RPORT 8091
set TARGETURI /
set SSL false
set THREADS 1
show options
run
back
hosts
services
EOF
```

检查：

```bash
cat ~/CyberLab/Metasploit/resources/http-title.rc
```

---

# 83. 执行 Resource Script

在 msfconsole：

```text
resource /home/kali/CyberLab/Metasploit/resources/http-title.rc
```

或启动时：

```bash
msfconsole \
  -q \
  -r ~/CyberLab/Metasploit/resources/http-title.rc
```

Resource Script 会顺序执行所有命令。

运行前必须逐行审查。

---

# 84. 不使用自动 Exploit Script

不要运行不了解的：

```text
autoexploit.rc
autoroute scripts
自动 Payload Handler
网上下载的 Resource Script
```

Resource Script 可以执行几乎所有 msfconsole 操作。

公开 `.rc` 文件不等于安全。

---

# 85. makerc

msfconsole 可以把本次输入历史保存为 Resource Script：

```text
makerc /home/kali/CyberLab/Metasploit/resources/recorded.rc
```

保存后检查：

```bash
less ~/CyberLab/Metasploit/resources/recorded.rc
```

它可能包含：

- IP
- Module Path
- Credential
- Payload 配置
- 误输入命令

公开前必须清理。

---

# 86. 自定义 Module 目录

用户自定义模块通常放在：

```text
~/.msf4/modules/
```

创建：

```bash
mkdir -p \
  ~/.msf4/modules/auxiliary/scanner/http
```

本课创建一个安全模块：

```text
只向授权 HTTP 服务发送 GET
读取 Status、Server 和 X-CyberLab Header
```

---

# 87. 创建自定义 HTTP Header Module

Kali：

```bash
nano \
  ~/.msf4/modules/auxiliary/scanner/http/cyberlab_header_check.rb
```

粘贴：

```ruby
class MetasploitModule < Msf::Auxiliary
  include Msf::Exploit::Remote::HttpClient
  include Msf::Auxiliary::Scanner

  def initialize(info = {})
    super(
      update_info(
        info,
        'Name' => 'CyberLab Safe HTTP Header Checker',
        'Description' => %q{
          Sends one authorized HTTP GET request per host and displays
          the response status, Server header, and X-CyberLab header.
          This module does not exploit a vulnerability.
        },
        'Author' => ['CyberLab Student'],
        'License' => MSF_LICENSE
      )
    )

    register_options(
      [
        Opt::RPORT(8091),
        OptString.new(
          'TARGETURI',
          [true, 'HTTP path to request', '/']
        )
      ]
    )
  end

  def run_host(ip)
    response = send_request_cgi(
      'method' => 'GET',
      'uri' => normalize_uri(target_uri.path)
    )

    unless response
      print_error("#{ip}:#{rport} - No HTTP response")
      return
    end

    server = response.headers['Server'] || 'not provided'
    cyberlab = response.headers['X-CyberLab'] || 'not provided'

    print_good(
      "#{ip}:#{rport} - Status=#{response.code} " \
      "Server=#{server.inspect} " \
      "X-CyberLab=#{cyberlab.inspect}"
    )
  end
end
```

保存并退出。

---

# 88. 加载自定义 Module

启动 msfconsole，或在现有 Console：

```text
reload_all
```

搜索：

```text
search name:cyberlab_header_check
```

加载：

```text
use auxiliary/scanner/http/cyberlab_header_check
```

查看：

```text
info
show options
```

---

# 89. 运行自定义安全 Module

配置：

```text
set RHOSTS 10.10.20.20
set RPORT 8091
set TARGETURI /
set SSL false
set THREADS 1
```

运行：

```text
run
```

预期：

```text
Status=200
Server="CyberLabHTTP/1.0 (Ubuntu-ARM64)"
X-CyberLab="Metasploit-Auxiliary-Lab"
```

实际 Server Header 格式由 Python 版本决定。

---

# 90. 自定义 Module 排错

搜索不到：

```text
reload_all
```

查看路径：

```bash
find ~/.msf4/modules -type f
```

Ruby 语法检查：

```bash
ruby -c \
  ~/.msf4/modules/auxiliary/scanner/http/cyberlab_header_check.rb
```

查看 Metasploit 日志：

```bash
find ~/.msf4 -maxdepth 2 -type f -name '*.log'
```

不要用 root 创建用户 Module，否则可能产生权限问题。

---

# 91. 自定义 Module 安全检查

确认代码中没有：

```text
system()
exec()
File.write
File.delete
download
payload
shell
session
```

本模块只调用：

```text
send_request_cgi
print_good
print_error
```

每个目标只发送一个 GET 请求。

---

# 92. 自定义 Module 的学习价值

你可以看到 Metasploit Module 的基础结构：

```text
MetasploitModule Class
Mixin
Metadata
Options
run_host
HTTP Request
Output
```

以后可以在防御课程中开发：

- Header Compliance Checker
- Health Endpoint Checker
- Banner Inventory Scanner
- Security Header Scanner

不要直接从扫描器发展成未授权 Exploit。


---

# 93. tcpdump 实时观察 Auxiliary Scanner

Kali 终端一：

```bash
sudo tcpdump \
  -i eth0 \
  -nn \
  -A \
  'host 10.10.20.20 and (tcp port 22 or tcp port 8091)'
```

Kali 终端二：

```bash
msfconsole -q
```

在 msfconsole 运行：

```text
use auxiliary/scanner/http/http_version
set RHOSTS 10.10.20.20
set RPORT 8091
set SSL false
set THREADS 1
run
```

tcpdump 中可能看到：

- TCP Handshake
- HTTP Request
- User-Agent
- Host Header
- Server Header
- X-CyberLab
- HTTP Status

---

# 94. 保存 Metasploit HTTP PCAP

Kali：

```bash
sudo tcpdump \
  -i eth0 \
  -nn \
  -s 0 \
  -w ~/CyberLab/Metasploit/captures/metasploit-http.pcap \
  'host 10.10.20.20 and tcp port 8091'
```

另一个终端运行 HTTP Version、Title 和自定义 Header Module。

结束后：

```text
Ctrl + C
```

---

# 95. 保存 SSH Version PCAP

```bash
sudo tcpdump \
  -i eth0 \
  -nn \
  -s 0 \
  -w ~/CyberLab/Metasploit/captures/metasploit-ssh-version.pcap \
  'host 10.10.20.20 and tcp port 22'
```

另一个终端运行：

```text
use auxiliary/scanner/ssh/ssh_version
set RHOSTS 10.10.20.20
set RPORT 22
set THREADS 1
run
```

停止 tcpdump。

---

# 96. Wireshark 分析 HTTP

打开：

```bash
wireshark \
  ~/CyberLab/Metasploit/captures/metasploit-http.pcap
```

过滤：

```text
http
```

只看请求：

```text
http.request
```

目标：

```text
ip.addr == 10.10.20.20 &&
tcp.port == 8091
```

自定义 Header：

```text
http.response.line contains "X-CyberLab"
```

---

# 97. Wireshark 分析 SSH

打开：

```bash
wireshark \
  ~/CyberLab/Metasploit/captures/metasploit-ssh-version.pcap
```

过滤：

```text
ssh
```

或者：

```text
tcp.port == 22
```

SSH 初始 Banner 在加密会话建立前通常可见。

后续认证和命令内容会被加密。

---

# 98. 比较 Nmap 与 Metasploit 流量

分别抓取：

```text
Nmap -sV
Metasploit http_version
Metasploit ssh_version
```

比较：

| 项目 | Nmap `-sV` | HTTP Version | SSH Version |
|---|---:|---:|---:|
| Requests |  |  |  |
| Ports |  |  |  |
| Protocol-specific |  |  |  |
| Duration |  |  |  |
| Detection |  |  |  |

Metasploit 专用 Scanner 通常只聚焦一个协议。

---

# 99. Ubuntu Access Log

查看：

```bash
less ~/CyberLab/Metasploit/logs/access.log
```

只看 Kali：

```bash
grep '10.10.20.10' \
  ~/CyberLab/Metasploit/logs/access.log
```

统计 GET：

```bash
grep -c '"GET ' \
  ~/CyberLab/Metasploit/logs/access.log
```

查看 User-Agent：

```bash
grep -o 'UA="[^"]*"' \
  ~/CyberLab/Metasploit/logs/access.log |
sort |
uniq -c
```

不同 Scanner Module 可能使用不同 User-Agent。

---

# 100. SSH Server Log

Ubuntu：

```bash
sudo journalctl \
  -u ssh \
  --since "15 minutes ago"
```

只进行 Version Scan 时，SSH 日志是否记录连接取决于：

- OpenSSH 配置
- LogLevel
- 连接阶段
- systemd 日志行为

没有明显日志不代表没有网络连接。

PCAP 可证明扫描流量。

---

# 101. Wazuh 实验标记

扫描前在 Ubuntu：

```bash
logger -t metasploit-lab \
  "Starting authorized Metasploit auxiliary scanning from Kali 10.10.20.10"
```

扫描后：

```bash
logger -t metasploit-lab \
  "Completed authorized Metasploit auxiliary scanning against ports 22 and 8091"
```

本地检查：

```bash
journalctl \
  -t metasploit-lab \
  --since "20 minutes ago"
```

---

# 102. Wazuh Dashboard

进入：

```text
Threat Hunting
```

筛选：

```text
agent.name: ubuntu-target
```

搜索：

```text
metasploit-lab
```

你应该看到开始和完成 Marker。

---

# 103. Metasploit 是否一定触发 Wazuh 告警？

不一定。

Metasploit Framework 包含大量模块，不存在单一固定网络行为。

自动检测取决于：

- 使用哪个 Module
- 请求内容
- User-Agent
- 服务日志
- Wazuh 是否收集相关日志
- Decoder 和 Rule
- IDS/IPS
- Suricata
- Zeek
- EDR

本课成功标准：

```text
Metasploit 扫描结果存在
PCAP 证明流量
Access Log 记录 HTTP 请求
Wazuh 看到 metasploit-lab Marker
```

---

# 104. 为什么 User-Agent 不是可靠检测？

User-Agent 可以：

- 被模块修改
- 被代理修改
- 被自定义 Module 设置
- 与浏览器相同
- 完全省略

可靠检测应结合：

- Request Pattern
- Path
- Frequency
- Payload Signature
- Source IP
- Server Error
- Process Activity
- Network IDS
- Endpoint Telemetry

---

# 105. Future Detection Lab

后续可以让 Wazuh 收集：

```text
~/CyberLab/Metasploit/logs/access.log
```

并建立规则检测：

- 高频 404
- Metasploit User-Agent
- 特定 Module Pattern
- 扫描多个端口
- SQLi / XSS Payload
- Exploit IOC

这属于：

```text
Wazuh Custom Decoder and Rule
```

本课不修改生产式 Wazuh 配置。

---

# 106. 数据库备份意识

Metasploit 数据库中可能保存：

- 内部 IP
- Service Banner
- Vulnerability
- Credential
- Loot
- Session 结果

不要把 PostgreSQL 数据目录或 `.msf4` 公开上传。

Workspace 是逻辑隔离，不是加密。

---

# 107. 清理 Global Options

课程结束前：

```text
unsetg RHOSTS
unsetg RHOST
unsetg RPORT
unsetg LHOST
unsetg LPORT
unsetg THREADS
```

查看：

```text
getg RHOSTS
```

避免下一次 Module 使用旧目标。

---

# 108. 停止 Jobs 和 Sessions

检查：

```text
jobs
sessions
```

停止所有不需要 Job：

```text
jobs -K
```

如果未来实验产生 Session：

```text
sessions -K
```

本课应没有 Session。

---

# 109. 退出和停止数据库

退出：

```text
exit
```

查看数据库：

```bash
msfdb status
```

如果你需要节省资源，可使用本机 `msfdb --help` 查看停止命令。

常见安装可能支持：

```bash
msfdb stop
```

不要删除数据库。

---

# 110. 完整实验一：Framework 和 Database

1. 检查 Metasploit 版本。
2. 创建实验目录。
3. 初始化数据库。
4. 启动 `msfconsole -q`。
5. 运行：

```text
db_status
workspace
workspace -a CyberLab-Metasploit
workspace CyberLab-Metasploit
```

6. 保存版本和数据库状态。

Kali：

```bash
{
  date -Iseconds
  msfconsole --version
  msfdb status
} > ~/CyberLab/Metasploit/notes/framework-status.txt
```

---

# 111. 完整实验二：Nmap Database

msfconsole：

```text
workspace CyberLab-Metasploit
db_nmap -n -sV -p 22,8091 10.10.20.20
hosts
services
vulns
loot
creds
```

导出：

```text
hosts -o /home/kali/CyberLab/Metasploit/results/hosts.csv
```

记录：

```text
数据库连接
Workspace
Host
开放端口
服务版本
Vulns 是否为空
Loot 是否为空
Creds 是否为空
```

---

# 112. 完整实验三：外部 Nmap Import

Kali：

```bash
sudo nmap \
  -n \
  -sV \
  -p 22,8091 \
  10.10.20.20 \
  -oA ~/CyberLab/Metasploit/nmap/lab3-ubuntu
```

msfconsole：

```text
workspace -a CyberLab-Metasploit-Import
workspace CyberLab-Metasploit-Import
db_import /home/kali/CyberLab/Metasploit/nmap/lab3-ubuntu.xml
hosts
services
```

比较：

```text
db_nmap Workspace
db_import Workspace
```

---

# 113. 完整实验四：HTTP Auxiliary

依次运行：

```text
auxiliary/scanner/http/http_version
auxiliary/scanner/http/title
auxiliary/scanner/http/cyberlab_header_check
```

每个 Module：

```text
info
show options
show missing
set RHOSTS 10.10.20.20
set RPORT 8091
set SSL false
set THREADS 1
run
back
```

保存 Spool 和 PCAP。

---

# 114. 完整实验五：SSH Version

```text
use auxiliary/scanner/ssh/ssh_version
info
show options
set RHOSTS 10.10.20.20
set RPORT 22
set THREADS 1
run
back
```

对比：

```bash
nmap -sV -p 22
```

和：

```bash
ssh -V
```

注意：

```text
ssh -V 显示 Kali Client 版本
不是 Ubuntu Server 版本
```

---

# 115. 完整实验六：Job

选择安全 HTTP Title Scanner：

```text
use auxiliary/scanner/http/title
set RHOSTS 10.10.20.20
set RPORT 8091
set TARGETURI /
set THREADS 1
run -j
jobs
jobs -l
```

如果 Job 已结束，记录：

```text
Scanner completed before jobs listing
```

不要为了制造长 Job 扫描大型范围。

---

# 116. 完整实验七：Resource Script

创建：

```text
http-title.rc
```

审查：

```bash
nl -ba \
  ~/CyberLab/Metasploit/resources/http-title.rc
```

运行：

```bash
msfconsole \
  -q \
  -r ~/CyberLab/Metasploit/resources/http-title.rc
```

保存输出。

---

# 117. 完整实验八：Module Research

选择一个公开 CVE，仅阅读：

```text
search cve:2021-41773
info MODULE_PATH
```

填写：

```text
Module Name：
Type：
Rank：
Check：
Platform：
Architecture：
Targets：
Required Options：
Compatible Payloads：
Side Effects：
References：
Current CyberLab Match：
Decision：
```

最终 Decision：

```text
Do Not Run on Current Infrastructure
```

---

# 118. 完整实验九：Traffic 与 Wazuh

1. Ubuntu 写入 `metasploit-lab` 开始 Marker。
2. Kali 抓 HTTP 和 SSH PCAP。
3. 运行三个安全 Auxiliary Scanner。
4. 停止抓包。
5. Ubuntu 写入完成 Marker。
6. 查看 Access Log。
7. Wireshark 统计。
8. Wazuh 搜索 `metasploit-lab`。
9. 记录 Wazuh 没有自动告警也可能是正常结果。

---

# 119. 实验报告模板

## Lab Information

```text
Lab Name：Metasploit Framework Auxiliary and Database Lab
Date：
Student：
Metasploit Host：Kali-CyberLab 10.10.20.10
Target：Ubuntu-Target 10.10.20.20
Allowed Ports：22, 8091
Workspace：CyberLab-Metasploit
Authorization：Personal CyberLab
Metasploit Version：
Database Status：
```

## Objective

```text
学习 Metasploit Framework 的模块、数据库、Workspace、
安全 Auxiliary Scanner、Resource Script 和网络可见性，
不执行 Exploit 或 Payload。
```

## Scope

```text
Included：
10.10.20.20:22
10.10.20.20:8091

Excluded：
10.10.20.30
10.10.20.40
All public networks
All Exploit execution
All credential attacks
```

## Database

```text
db_status：
Workspace：
Nmap XML：
Imported Hosts：
Imported Services：
Vulns：
Loot：
Creds：
```

## Module Results

| Module | Target | Port | Result | Evidence |
|---|---|---:|---|---|
| http_version |  |  |  |  |
| title |  |  |  |  |
| ssh_version |  |  |  |  |
| custom header |  |  |  |  |

## Module Review

```text
CVE：
Module：
Rank：
Check：
Platform：
Architecture：
Targets：
Side Effects：
Decision：
```

## Traffic Evidence

| Capture | Packets | Requests | Duration |
|---|---:|---:|---:|
| HTTP |  |  |  |
| SSH |  |  |  |

## Wazuh

```text
Start Marker：
End Marker：
Automatic Alert：
Explanation：
```

## Security Conclusions

```text
Version detection is not vulnerability confirmation.
Auxiliary does not always mean safe.
Workspace is logical data separation.
ARM64 compatibility must be verified.
check should precede exploit in a dedicated vulnerable lab.
Current infrastructure should not be exploited.
```

## Evidence Files

```text
msfconsole Spool：
Hosts CSV：
Services Output：
Nmap XML：
HTTP PCAP：
SSH PCAP：
Access Log：
Wazuh Marker：
Custom Module：
Resource Script：
```

---

# 120. 常见错误：msfconsole not found

```bash
sudo apt update
sudo apt install metasploit-framework
```

检查：

```bash
which msfconsole
apt policy metasploit-framework
```

---

# 121. Database not connected

退出 msfconsole：

```text
exit
```

检查：

```bash
msfdb status
```

初始化：

```bash
sudo msfdb init
```

重新启动：

```bash
msfconsole -q
```

检查：

```text
db_status
```

---

# 122. PostgreSQL 错误

检查：

```bash
systemctl status postgresql --no-pager
```

```bash
msfdb status
```

不要马上运行：

```bash
msfdb reinit
```

因为它可能删除旧数据库。

先保存错误输出并检查权限和磁盘空间。

---

# 123. Module not found

```text
search MODULE_KEYWORD
reload_all
```

检查版本：

```bash
msfconsole --version
```

更新软件包：

```bash
sudo apt update
sudo apt install --only-upgrade metasploit-framework
```

模块路径可能随版本改变。

---

# 124. Custom Module not found

检查：

```bash
find ~/.msf4/modules \
  -type f \
  -name 'cyberlab_header_check.rb'
```

Ruby 语法：

```bash
ruby -c \
  ~/.msf4/modules/auxiliary/scanner/http/cyberlab_header_check.rb
```

msfconsole：

```text
reload_all
search cyberlab
```

不要把模块放到错误用户的：

```text
/root/.msf4
```

---

# 125. No route to host

检查：

```bash
ip addr
```

```bash
ip route
```

```bash
ping -c 3 10.10.20.20
```

```bash
nc -zvn -w 3 10.10.20.20 8091
```

确认 UTM CyberLab 网卡正常。

---

# 126. Connection refused

Ubuntu：

```bash
ss -lntp | grep ':8091'
```

启动：

```bash
python3 \
  ~/CyberLab/Metasploit/server/metasploit_lab.py
```

检查防火墙：

```bash
sudo ufw status
```

---

# 127. Required option missing

```text
show options
show missing
```

设置：

```text
set OPTION VALUE
```

常见遗漏：

```text
RHOSTS
RPORT
TARGETURI
SESSION
```

本课不要设置 SESSION。

---

# 128. Wrong target from setg

查看：

```text
getg RHOSTS
getg RPORT
getg LHOST
```

清理：

```text
unsetg RHOSTS
unsetg RPORT
unsetg LHOST
unsetg LPORT
```

重新：

```text
show options
```

---

# 129. Scanner 太快

降低：

```text
set THREADS 1
```

如果 Module 有 Timeout 或 Delay Advanced Option：

```text
show advanced
```

不要设置高线程扫描整个网段。

---

# 130. HTTP Scanner 无结果

先验证：

```bash
curl -i http://10.10.20.20:8091/
```

msfconsole：

```text
set RPORT 8091
set SSL false
set TARGETURI /
show options
run
```

检查 Module 是否适合 HTTP。

---

# 131. SSH Version 无结果

```bash
nc -nv 10.10.20.20 22
```

```bash
sudo nmap -n -sV -p 22 10.10.20.20
```

Ubuntu：

```bash
sudo systemctl status ssh --no-pager
```

---

# 132. Resource Script 运行了错误命令

立即：

```text
jobs -K
sessions -K
```

退出：

```text
exit
```

审查脚本：

```bash
nl -ba SCRIPT.rc
```

不要运行从互联网复制但未逐行阅读的 Resource Script。

---

# 133. 本课练习

1. 检查 Metasploit 版本。
2. 安装或更新软件包。
3. 创建实验目录。
4. 启动 Ubuntu 8091 服务。
5. 使用 curl 和 Nmap 验证。
6. 初始化 msfdb。
7. 运行 `db_status`。
8. 创建 Workspace。
9. 使用 `db_nmap`。
10. 查看 `hosts`。
11. 查看 `services`。
12. 查看 `vulns`。
13. 查看 `loot`。
14. 查看 `creds`。
15. 导出 Hosts CSV。
16. 创建外部 Nmap XML。
17. 使用 `db_import`。
18. 使用 `search -h`。
19. 按 Type 搜索。
20. 按 CVE 搜索。
21. 使用 `info`。
22. 使用 `use` 和 `back`。
23. 使用 `show options`。
24. 使用 `show missing`。
25. 使用 `show advanced`。
26. 使用 `set` 和 `unset`。
27. 使用 `setg` 并清除。
28. 运行 HTTP Version Scanner。
29. 运行 HTTP Title Scanner。
30. 运行 SSH Version Scanner。
31. 运行有限 TCP Port Scanner。
32. 使用 `run -j`。
33. 查看 `jobs`。
34. 查看 `sessions`。
35. 静态查看 `multi/handler`。
36. 创建 Resource Script。
37. 运行 Resource Script。
38. 使用 `makerc`。
39. 创建自定义 Auxiliary Module。
40. 使用 `reload_all`。
41. 运行自定义 Header Checker。
42. 保存 Spool。
43. 保存 HTTP PCAP。
44. 保存 SSH PCAP。
45. 使用 Wireshark 分析。
46. 查看 Ubuntu Access Log。
47. 在 Wazuh 搜索 Marker。
48. 静态阅读一个 CVE Module。
49. 填写 Module Research 表。
50. 完成实验报告。

---

# 134. 复习题

1. Metasploit Framework 是什么？
2. Framework 与 Pro 有什么区别？
3. msfconsole 的作用是什么？
4. msfdb 的作用是什么？
5. Workspace 有什么作用？
6. Auxiliary 是否一定安全？
7. Exploit 与 Payload 有什么区别？
8. Post Module 什么时候使用？
9. Job 与 Session 有什么区别？
10. `db_nmap` 做什么？
11. `db_import` 做什么？
12. `hosts` 做什么？
13. `services` 做什么？
14. `vulns` 做什么？
15. `loot` 做什么？
16. `creds` 做什么？
17. `search` 做什么？
18. `info` 做什么？
19. `use` 做什么？
20. `back` 做什么？
21. `show options` 做什么？
22. `show missing` 做什么？
23. `show advanced` 做什么？
24. `RHOSTS` 是什么？
25. `RPORT` 是什么？
26. `LHOST` 是什么？
27. `TARGETURI` 是什么？
28. `set` 与 `setg` 有什么区别？
29. 为什么 Global Option 有风险？
30. `run` 与 `exploit` 有什么关系？
31. `check` 是否等于已确认可利用？
32. 为什么要查看 `show targets`？
33. 为什么要查看 `show payloads`？
34. Exploit Ranking 表示什么？
35. 为什么 ARM64 很重要？
36. Handler 是什么？
37. Reverse 与 Bind 有什么区别？
38. Resource Script 是什么？
39. 为什么要审查 Resource Script？
40. `makerc` 做什么？
41. 自定义 Module 放在哪里？
42. `reload_all` 做什么？
43. 为什么 Wazuh 不一定自动告警？
44. 为什么当前 Ubuntu 和 Windows 不应被 Exploit？
45. 为什么必须在课程结束时清理 Global Options？

---

# 135. 参考答案

1. 模块化的开源安全研究和渗透测试平台。
2. Framework 免费开源并以 Console 为主；Pro 是商业平台。
3. 提供统一命令行界面。
4. 初始化和管理 Framework 的 PostgreSQL 数据库。
5. 逻辑隔离不同实验的数据。
6. 不一定，Auxiliary 也可能包含 Fuzzer、Login 或 DoS。
7. Exploit 利用漏洞；Payload 在成功后执行功能。
8. 已经获得 Session 后。
9. Job 是后台任务；Session 是与目标建立的通道。
10. 运行 Nmap 并把结果存入数据库。
11. 导入已有扫描文件。
12. 查看数据库中的 Host。
13. 查看服务和端口。
14. 查看漏洞记录。
15. 查看收集的文件和数据。
16. 查看 Credential 记录。
17. 搜索模块。
18. 阅读模块详细信息。
19. 加载模块。
20. 离开当前模块。
21. 显示模块选项。
22. 显示缺失的必需选项。
23. 显示高级配置。
24. 远程目标地址集合。
25. 远程服务端口。
26. 本地监听或回连地址。
27. Web 目标路径。
28. `set` 只影响当前模块；`setg` 是全局设置。
29. 切换模块后可能继续指向旧目标。
30. Auxiliary 常用 run；Exploit Module 可使用 run/exploit。
31. 不等于，可能只是版本或行为线索。
32. 确认适配的系统、版本和架构。
33. 只选择与 Exploit 和 Target 兼容的 Payload。
34. 模块可靠性和潜在破坏风险等级。
35. 很多模块和 Payload 只支持 x86/x64。
36. 等待或接收 Payload 连接的组件。
37. Reverse 由目标回连；Bind 由目标监听。
38. 顺序执行 msfconsole 命令的脚本。
39. 它可能自动运行扫描、Exploit 或 Payload。
40. 把 Console 命令历史保存为 Resource Script。
41. `~/.msf4/modules/`。
42. 重新加载模块。
43. 需要对应服务日志、规则和网络或端点遥测。
44. 它们是实验基础设施，不是一次性脆弱靶机。
45. 避免后续 Module 误用旧 IP 和端口。

---

# 136. 命令速查表

## 启动

```bash
msfconsole -q
```

## 数据库

```bash
sudo msfdb init
msfdb status
```

```text
db_status
```

## Workspace

```text
workspace
workspace -a CyberLab-Metasploit
workspace CyberLab-Metasploit
```

## Nmap

```text
db_nmap -n -sV -p 22,8091 10.10.20.20
```

```text
db_import /path/scan.xml
```

## 数据

```text
hosts
services
vulns
notes
loot
creds
```

## 搜索

```text
search -h
search type:auxiliary name:http_version
search cve:2021-41773
```

## 模块

```text
info MODULE
use MODULE
show options
show missing
show advanced
back
```

## 设置

```text
set RHOSTS 10.10.20.20
set RPORT 8091
set THREADS 1
unset RHOSTS
```

## 全局设置

```text
setg RHOSTS 10.10.20.20
getg RHOSTS
unsetg RHOSTS
```

## 运行

```text
run
run -j
jobs
jobs -l
jobs -k ID
```

## Session

```text
sessions
sessions -l
```

## 输出

```text
spool /path/msf.log
spool off
```

## Resource Script

```text
resource /path/lab.rc
makerc /path/recorded.rc
```

## Custom Module

```text
reload_all
search cyberlab
```

---

# 137. 核心记忆口诀

```text
先 Nmap
再 Database
再 Module

search：找模块
info：读模块
use：加载模块
show options：看配置
show missing：找缺项
set：当前模块
setg：全局设置
run：执行 Auxiliary
check：先检查
exploit：只限专用靶场

RHOSTS 是目标
RPORT 是目标端口
LHOST 是本机地址
TARGETURI 是 Web 路径

Job 是后台任务
Session 是目标通道
Workspace 隔离数据
Spool 保存证据

Auxiliary 不一定安全
Rank 不代表无风险
ARM64 必须检查兼容性
当前基础设施不执行 Exploit
```

---

# 138. 完成标准

完成本课后，你应该拥有：

```text
Metasploit 版本记录
正常 PostgreSQL Database
CyberLab-Metasploit Workspace
db_nmap 结果
外部 Nmap XML
db_import 结果
Hosts CSV
Services 记录
HTTP Version 结果
HTTP Title 结果
SSH Version 结果
有限 Portscan 结果
一个 Job 实验
零 Session 的确认
Handler 静态笔记
一个 CVE Module 研究表
安全 Resource Script
Custom HTTP Header Module
msfconsole Spool
HTTP PCAP
SSH PCAP
Wireshark 统计
Ubuntu Access Log
Wazuh metasploit-lab Marker
完整实验报告
```

检查 Kali：

```bash
find ~/CyberLab/Metasploit \
  -type f |
sort
```

检查自定义 Module：

```bash
find ~/.msf4/modules \
  -type f |
sort
```

---

# 139. 下一课

下一课建议：

```text
Hydra
```

学习内容：

- 认证测试与密码攻击的区别
- 小型测试账号和 Wordlist
- SSH、HTTP Basic 和 Form Login
- `-l`、`-L`、`-p`、`-P`
- `-s`、`-t`、`-W`、`-f`
- 成功和失败字符串
- 账号锁定风险
- Rate Limiting
- Access Log 和 auth.log
- Wazuh Authentication Failure
- 安全密码策略
- MFA
- 修复和复测

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
OWASP ZAP：扫描和自动化
SQLmap：SQL Injection 检测
SearchSploit：公开 PoC 研究
Metasploit：模块、数据库和验证框架
```

---

# 140. 参考说明

本指南依据：

```text
Rapid7 Metasploit Framework 官方文档
Rapid7 Modules 和 Manual Exploitation 文档
Rapid7 Database、Workspace 和 Resource Script 文档
Metasploit Framework 官方 GitHub
Kali Linux metasploit-framework 工具页面
本机 msfconsole help 和 search -h 输出
```

当前官方资料说明：

```text
Metasploit Framework 使用模块执行扫描、利用和后续任务
PostgreSQL Database 可保存 Host、Service、Loot 和 Exploit Result
Workspace 用于逻辑隔离数据
Resource Script 可通过 resource 或 msfconsole -r 执行
Kali 使用 metasploit-framework 软件包提供 Framework
```

工具更新后，以你的 Kali 输出为准：

```bash
msfconsole --version
msfconsole -q
```

在 Console：

```text
help
search -h
info MODULE
show options
```
