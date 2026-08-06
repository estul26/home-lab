# 第十四课：Hydra 完整学习指南

> **实验环境：你自己的 CyberLab**
>
> 本课只用于你自己创建、拥有并控制的测试账号和认证服务。  
> 不要对学校、公司、邮箱、银行、政府网站、公共服务器、他人账号或任何未经明确授权的系统运行 Hydra。

---

# 1. 本课目标

完成本课后，你应该能够：

```text
理解 Hydra 的用途、能力和限制
区分在线认证测试与离线 Hash 恢复
理解 Password Guessing、Dictionary Attack、Password Spraying 和 Credential Stuffing
安装并检查 Kali 中的 Hydra
查看当前编译支持的协议模块
理解 -l、-L、-p、-P、-C
理解 -t、-T、-w、-W、-f、-F、-s、-S、-o 和 -b
使用一个专用测试账号和极小 Wordlist
对自己的 Ubuntu SSH 执行低速认证审计
对受控 HTTP Basic Authentication 执行低速认证审计
对受控 HTTP Form Login 执行低速认证审计
正确确定 Form Login 的失败或成功判断条件
使用 curl 和浏览器验证认证行为
控制并发、等待时间、重试和停止条件
理解账号锁定、Rate Limiting、MFA 和日志监控
保存 TXT 或 JSON 结果
使用 tcpdump、Wireshark、auth.log、Access Log 和 Wazuh 观察认证失败
清理测试账号、测试密码和实验服务
完成一份 Authorized Credential Audit 报告
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
Hydra：Kali 10.10.20.10
Ubuntu SSH：10.10.20.20:22
Ubuntu HTTP Lab：10.10.20.20:8092
```

本课只使用：

```text
一个专用测试账号
四个测试密码
一个目标
一个线程
发现正确密码后立即停止
```

---

# 2. 什么是 Hydra？

Hydra，也常称：

```text
THC Hydra
```

是一个并行化的网络登录测试工具。

它能够对许多网络认证协议尝试：

```text
Username
Password
Username:Password Pair
```

并判断认证是否成功。

常见支持协议包括：

- SSH
- FTP
- HTTP GET/HEAD/POST
- HTTP Form GET/POST
- HTTPS
- SMB
- RDP
- VNC
- Telnet
- LDAP
- IMAP
- POP3
- SMTP
- MySQL
- PostgreSQL
- MSSQL
- SNMP
- SOCKS5
- VMware Authentication
- XMPP

具体支持哪些模块取决于：

```text
Hydra 版本
Kali 软件包
编译时依赖
```

本机检查命令：

```bash
hydra -h
```

---

# 3. Hydra 的工作流程

```text
准备目标
→ 准备账号或账号列表
→ 准备密码或密码列表
→ 选择协议模块
→ 控制线程和等待时间
→ Hydra 逐个尝试
→ 模块判断成功或失败
→ 保存结果
```

例子：

```text
User：hydra-lab-user
Passwords：
  WrongPass-01!
  Summer2026!
  Password123!
  LabOnly-2026!
```

Hydra 会按配置尝试这些组合。

---

# 4. 在线测试与离线恢复

## Hydra：在线认证测试

```text
Kali
→ 网络连接
→ SSH / HTTP / SMB / RDP Server
→ 服务器验证密码
```

特点：

- 每次尝试访问真实服务
- 服务器可以记录失败
- 可能触发锁定
- 可能触发 Rate Limit
- 受网络延迟影响
- 容易被检测

## John / Hashcat：离线 Hash 恢复

```text
Hash File
→ 本地 CPU/GPU
→ 比较计算结果
```

特点：

- 不访问登录服务器
- 不触发服务账号锁定
- 速度通常远高于在线测试
- 需要合法获得 Hash

简单记忆：

```text
Hydra 打登录服务
John/Hashcat 解离线 Hash
```

---

# 5. 几种认证攻击概念

## Password Guessing

对一个账号尝试少量候选密码。

```text
user1 → pass1, pass2, pass3
```

## Dictionary Attack

使用 Wordlist 中的候选密码。

## Password Spraying

对多个账号尝试同一个或少量常见密码。

```text
user1 → Spring2026!
user2 → Spring2026!
user3 → Spring2026!
```

目的是降低单个账号快速锁定的概率。

本课不进行多账号 Spray。

## Credential Stuffing

使用从其他地方泄露的真实账号密码组合。

本课禁止：

```text
泄露数据
真实凭据
他人账号
```

## Brute Force

系统地生成大量组合。

Hydra 支持生成模式，但本课不使用大规模生成。

---

# 6. 为什么在线认证测试风险高？

可能造成：

- 账号锁定
- 服务负载增加
- 日志大量增长
- 用户无法登录
- 触发 MFA 请求
- 触发邮件或短信告警
- 被防火墙或 Fail2ban 封禁
- 违反法律、合同和政策
- 暴露测试密码
- 污染生产审计记录

本课采取：

```text
专用账号
极小 Wordlist
单线程
等待
发现后停止
隔离网络
完整日志观察
实验后清理
```

---

# 7. Hydra 不是什么？

Hydra 不是：

- 离线 Hash Cracker
- MFA Bypass 工具
- 自动权限提升工具
- 已经获得授权的证明
- 保证成功的工具
- 适合无限线程的工具
- 生产系统安全测试的默认首选

Hydra 找到正确密码只证明：

```text
当前候选凭据可以通过当前认证接口登录
```

它不自动证明：

- 账号权限很高
- 数据已经泄露
- MFA 不存在
- 其他协议也可登录
- 账号应被继续使用

---

# 8. 本课安全边界

允许：

```text
10.10.20.20:22
10.10.20.20:8092
账号 hydra-lab-user
HTTP 账号 hydraweb
四个自建测试密码
```

不允许：

```text
root
真实管理员账号
wazuh-test-user
Windows 真实账号
学校账号
邮箱账号
银行账号
公共 IP
10.10.20.0/24 批量扫描
大量用户名
rockyou.txt
自动生成全部字符组合
Credential Stuffing
账号锁定规避
MFA 疲劳攻击
代理轮换
隐藏来源
```

---

# 9. 检查和安装 Hydra

Kali：

```bash
which hydra
```

查看版本和帮助：

```bash
hydra -h
```

检查软件包：

```bash
apt policy hydra
```

安装：

```bash
sudo apt update
sudo apt install hydra
```

可选图形界面：

```bash
sudo apt install hydra-gtk
```

启动 GUI：

```bash
xhydra
```

本课主要使用命令行，因为：

```text
命令更容易记录
参数更清楚
报告更容易复现
```

---

# 10. 检查当前支持模块

```bash
hydra -h
```

输出末尾通常会列出支持服务。

查看模块专用帮助：

```bash
hydra -U ssh
```

```bash
hydra -U http-get
```

```bash
hydra -U http-post-form
```

如果模块未编译：

```text
不要假设 Kali 一定支持
```

重新检查软件包和依赖。

---

# 11. 创建实验目录

Kali：

```bash
mkdir -p ~/CyberLab/Hydra/{wordlists,results,captures,notes,reports}
```

Ubuntu：

```bash
mkdir -p ~/CyberLab/Hydra/{server,logs}
```

检查：

```bash
find ~/CyberLab/Hydra -maxdepth 2 -type d
```

---

# 12. 创建专用 SSH 测试账号

Ubuntu：

```bash
sudo adduser hydra-lab-user
```

按照提示设置专用实验密码：

```text
LabOnly-2026!
```

用户信息可以留空。

检查：

```bash
id hydra-lab-user
```

确认不是管理员：

```bash
groups hydra-lab-user
```

不应包含：

```text
sudo
adm
docker
lxd
```

---

# 13. 确认 SSH 密码认证

Ubuntu：

```bash
sudo sshd -T |
grep -E 'passwordauthentication|permitrootlogin|maxauthtries'
```

期望看到类似：

```text
passwordauthentication yes
permitrootlogin no
```

如果 `PasswordAuthentication` 是 `no`：

```text
Hydra SSH 密码实验无法进行
```

不要为真实服务器降低安全性。

只有在你的 Ubuntu Target 中，经过快照和授权后，才考虑创建独立配置。

---

# 14. 测试账号手工登录

Kali：

```bash
ssh hydra-lab-user@10.10.20.20
```

输入：

```text
LabOnly-2026!
```

登录成功后：

```bash
whoami
```

预期：

```text
hydra-lab-user
```

退出：

```bash
exit
```

必须先证明：

```text
服务正常
账号正常
正确密码正常
```

否则 Hydra 结果无法解释。

---

# 15. 创建小型 Username 文件

Kali：

```bash
cat > ~/CyberLab/Hydra/wordlists/users-one.txt <<'EOF'
hydra-lab-user
EOF
```

HTTP 用户：

```bash
cat > ~/CyberLab/Hydra/wordlists/http-users-one.txt <<'EOF'
hydraweb
EOF
```

检查：

```bash
cat ~/CyberLab/Hydra/wordlists/users-one.txt
```

---

# 16. 创建小型 Password Wordlist

Kali：

```bash
cat > ~/CyberLab/Hydra/wordlists/passwords-four.txt <<'EOF'
WrongPass-01!
Summer2026!
Password123!
LabOnly-2026!
EOF
```

检查行数：

```bash
wc -l \
  ~/CyberLab/Hydra/wordlists/passwords-four.txt
```

预期：

```text
4
```

正确密码放在最后，是为了观察：

```text
三次失败
一次成功
```

---

# 17. 保护 Wordlist 权限

```bash
chmod 600 \
  ~/CyberLab/Hydra/wordlists/*.txt
```

检查：

```bash
ls -l \
  ~/CyberLab/Hydra/wordlists
```

Wordlist 即使只包含测试密码，也不应公开提交。

---

# 18. 不使用 rockyou.txt

本课不使用：

```text
/usr/share/wordlists/rockyou.txt
```

原因：

- 数量巨大
- 请求过多
- 可能触发锁定和封禁
- 对理解参数没有帮助
- 增加日志和 PCAP
- 更容易误用到非实验目标

学习 Hydra 不等于运行最大 Wordlist。

---

# 19. Hydra 基本语法

常见两种形式。

URL 风格：

```bash
hydra OPTIONS service://server[:port][/option]
```

传统风格：

```bash
hydra OPTIONS server service
```

例子：

```bash
hydra \
  -l hydra-lab-user \
  -P passwords-four.txt \
  ssh://10.10.20.20
```

本课优先使用 URL 风格。

---

# 20. Username 参数

单个 Username：

```text
-l LOGIN
```

例子：

```bash
-l hydra-lab-user
```

Username 文件：

```text
-L FILE
```

例子：

```bash
-L users-one.txt
```

本课优先：

```text
-l
```

因为只有一个授权测试账号。

---

# 21. Password 参数

单个 Password：

```text
-p PASS
```

Password 文件：

```text
-P FILE
```

例子：

```bash
-P passwords-four.txt
```

本课使用：

```text
-P
```

来学习小型 Dictionary Attack。

---

# 22. Username:Password Pair 文件

```text
-C FILE
```

格式：

```text
username:password
```

创建：

```bash
cat > ~/CyberLab/Hydra/wordlists/test-pairs.txt <<'EOF'
hydra-lab-user:WrongPass-01!
hydra-lab-user:LabOnly-2026!
EOF
```

只允许：

```text
自建实验凭据
```

禁止把泄露数据放进 `-C` 文件。

---

# 23. -e 额外尝试

Hydra 支持：

```text
-e n
-e s
-e r
```

含义：

- `n`：空密码
- `s`：Password 与 Username 相同
- `r`：Username 反转作为 Password

本课不使用：

```text
-e nsr
```

原因：

- 增加额外尝试
- 可能触发锁定
- 这些模式不符合当前实验设计

---

# 24. 并发控制

每个目标的并行任务：

```text
-t TASKS
```

Hydra 默认值可能较高。

本课必须：

```bash
-t 1
```

理由：

- 请求顺序容易观察
- 不给服务器制造并发压力
- 降低锁定风险
- 便于对应日志时间
- 便于 Wireshark 分析

---

# 25. 多目标总并发

```text
-T TASKS
```

主要配合：

```text
-M server-list
```

本课不使用多目标列表。

不要对：

```text
10.10.20.0/24
```

执行认证测试。

---

# 26. 响应等待和连接间隔

```text
-w TIME
```

等待服务器响应的超时时间。

```text
-W TIME
```

每个线程两次连接之间的等待时间。

本课推荐：

```bash
-w 10
-W 2
```

含义：

```text
最多等待响应约 10 秒
每次连接之间等待约 2 秒
```

---

# 27. 找到后停止

```text
-f
```

找到一组有效凭据后停止当前目标。

多目标模式中的全局停止：

```text
-F
```

本课只有一个目标，使用：

```bash
-f
```

避免找到正确密码后继续尝试。

---

# 28. 自定义端口

```text
-s PORT
```

SSH 非标准端口例子：

```bash
-s 2222
```

HTTP Lab：

```bash
-s 8092
```

URL 风格也可直接写：

```text
http-get://10.10.20.20:8092/basic
```

---

# 29. TLS

```text
-S
```

表示使用 SSL/TLS 连接。

HTTP 模块也有：

```text
https-get
https-post-form
```

本课 8092 使用明文 HTTP，因此：

```text
不使用 -S
```

---

# 30. 输出文件

```text
-o FILE
```

例子：

```bash
-o ~/CyberLab/Hydra/results/ssh-result.txt
```

输出格式：

```text
-b text
-b json
-b jsonv1
```

例子：

```bash
-b json
```

不同版本支持情况以：

```bash
hydra -h
```

为准。

---

# 31. Verbose 输出

```text
-v
```

显示详细信息。

```text
-V
```

显示每次 Username/Password 尝试。

```text
-d
```

Debug 输出，可能非常多。

本课推荐：

```text
-vV
```

但报告中不要公开完整密码。

---

# 32. 恢复文件

Hydra 中断后可能保存：

```text
hydra.restore
```

恢复：

```bash
hydra -R
```

忽略现有 Restore：

```bash
hydra -I
```

本课只有四个密码，不需要恢复大型任务。

实验完成后检查：

```bash
find . -name 'hydra.restore'
```

不要恢复一个不再确认 Scope 的旧任务。


---

# 33. SSH 实验前日志标记

Ubuntu：

```bash
logger -t hydra-lab \
  "Starting authorized low-rate SSH credential audit from Kali 10.10.20.10"
```

检查：

```bash
journalctl \
  -t hydra-lab \
  --since "5 minutes ago"
```

---

# 34. 第一次 SSH Hydra 实验

Kali：

```bash
hydra \
  -l hydra-lab-user \
  -P ~/CyberLab/Hydra/wordlists/passwords-four.txt \
  -t 1 \
  -W 2 \
  -w 10 \
  -f \
  -vV \
  -o ~/CyberLab/Hydra/results/ssh-result.txt \
  -b text \
  ssh://10.10.20.20
```

预期：

```text
前三个密码失败
第四个密码成功
Hydra 找到后停止
```

结果可能显示类似：

```text
[22][ssh] host: 10.10.20.20
login: hydra-lab-user
password: LabOnly-2026!
```

具体格式随版本变化。

---

# 35. SSH 命令逐项解释

```bash
-l hydra-lab-user
```

只测试一个授权账号。

```bash
-P passwords-four.txt
```

使用四个测试密码。

```bash
-t 1
```

一次只建立一个认证任务。

```bash
-W 2
```

两次连接之间等待。

```bash
-w 10
```

服务器响应超时。

```bash
-f
```

发现有效凭据后停止。

```bash
-vV
```

显示任务和每次尝试。

```bash
-o ssh-result.txt
```

保存发现结果。

```text
ssh://10.10.20.20
```

使用 SSH 模块攻击明确目标。

---

# 36. 使用单个密码测试

只验证已知实验凭据：

```bash
hydra \
  -l hydra-lab-user \
  -p 'LabOnly-2026!' \
  -t 1 \
  -f \
  ssh://10.10.20.20
```

这不是 Dictionary Attack。

它相当于：

```text
自动验证一组已知凭据
```

---

# 37. 使用 Username 文件

```bash
hydra \
  -L ~/CyberLab/Hydra/wordlists/users-one.txt \
  -P ~/CyberLab/Hydra/wordlists/passwords-four.txt \
  -t 1 \
  -W 2 \
  -w 10 \
  -f \
  ssh://10.10.20.20
```

因为文件只有一行，它与 `-l` 行为接近。

本课不向文件加入其他账号。

---

# 38. 使用 Pair 文件

```bash
hydra \
  -C ~/CyberLab/Hydra/wordlists/test-pairs.txt \
  -t 1 \
  -W 2 \
  -w 10 \
  -f \
  ssh://10.10.20.20
```

`-C` 不应与：

```text
-l/-L
-p/-P
```

混合使用。

---

# 39. 验证 Hydra 结果

找到结果后，不要立即继续测试权限。

只进行一次手工验证：

```bash
ssh hydra-lab-user@10.10.20.20
```

输入找到的实验密码。

登录后：

```bash
whoami
id
```

退出：

```bash
exit
```

报告中记录：

```text
Credential validated
No privilege escalation attempted
```

---

# 40. 查看 SSH 认证日志

Ubuntu 22.04 常见：

```bash
sudo tail -n 100 /var/log/auth.log
```

如果文件不存在：

```bash
sudo journalctl \
  -u ssh \
  --since "20 minutes ago"
```

搜索账号：

```bash
sudo grep 'hydra-lab-user' \
  /var/log/auth.log |
tail -n 30
```

可能看到：

- Failed password
- Accepted password
- Source IP `10.10.20.10`
- Source Port
- sshd Process ID

---

# 41. 统计 SSH 失败与成功

Ubuntu：

```bash
sudo grep 'hydra-lab-user' \
  /var/log/auth.log |
grep -c 'Failed password'
```

成功：

```bash
sudo grep 'hydra-lab-user' \
  /var/log/auth.log |
grep -c 'Accepted password'
```

如果使用 Journal：

```bash
sudo journalctl \
  -u ssh \
  --since "20 minutes ago" |
grep 'hydra-lab-user'
```

---

# 42. SSH Wazuh 观察

Wazuh Dashboard：

```text
Threat Hunting
```

筛选：

```text
agent.name: ubuntu-target
```

搜索：

```text
hydra-lab-user
```

也可搜索：

```text
sshd
authentication failed
Failed password
Accepted password
10.10.20.10
```

Wazuh Rule ID 可能随版本和事件类型变化。

不要依赖单一固定 Rule ID。

---

# 43. 结束 SSH 实验标记

Ubuntu：

```bash
logger -t hydra-lab \
  "Completed authorized low-rate SSH credential audit"
```

Wazuh 搜索：

```text
hydra-lab
```

手工 Marker 用于确认：

```text
实验开始和结束时间
日志收集链路
```

---

# 44. SSH 测试可能触发的防御

目标可能配置：

- PAM Account Lockout
- Fail2ban
- SSH MaxAuthTries
- 防火墙限速
- SIEM Alert
- EDR
- IP Block
- MFA
- Password Authentication Disabled

检查锁定相关配置：

```bash
sudo grep -RniE \
  'pam_faillock|pam_tally|deny=' \
  /etc/pam.d \
  /etc/security \
  2>/dev/null
```

如果存在账号锁定：

```text
不要试图绕过
降低尝试次数
或只使用 HTTP 实验服务
```

---

# 45. 检查账号是否锁定

Ubuntu 可能支持：

```bash
sudo faillock \
  --user hydra-lab-user
```

并非所有系统都使用 `pam_faillock`。

不要在不了解配置时运行 Reset。

需要清理自己的实验账号锁定时，先查看：

```bash
faillock --help
```

并记录清理操作。

---

# 46. 创建 HTTP 认证实验服务器

Ubuntu：

```bash
nano ~/CyberLab/Hydra/server/hydra_http_lab.py
```

粘贴：

```python
#!/usr/bin/env python3

import base64
import html
from http.server import BaseHTTPRequestHandler, HTTPServer
from urllib.parse import parse_qs


HOST = "10.10.20.20"
PORT = 8092

VALID_USERNAME = "hydraweb"
VALID_PASSWORD = "LabOnly-2026!"


def page(title: str, body: str) -> bytes:
    return f"""<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>{html.escape(title)}</title>
</head>
<body>
  <h1>{html.escape(title)}</h1>
  {body}
</body>
</html>
""".encode("utf-8")


class HydraLabHandler(BaseHTTPRequestHandler):
    server_version = "CyberLabHydra/1.0"

    def send_body(
        self,
        status: int,
        body: bytes,
        content_type: str = "text/html; charset=utf-8",
        headers: dict[str, str] | None = None,
    ) -> None:
        self.send_response(status)
        self.send_header("Content-Type", content_type)
        self.send_header("Content-Length", str(len(body)))
        self.send_header("X-CyberLab", "Hydra-Authentication-Lab")

        if headers:
            for name, value in headers.items():
                self.send_header(name, value)

        self.end_headers()

        if self.command != "HEAD":
            self.wfile.write(body)

    def parse_basic_username(self) -> tuple[str, str]:
        authorization = self.headers.get("Authorization", "")

        if not authorization.startswith("Basic "):
            return "", ""

        encoded = authorization.removeprefix("Basic ").strip()

        try:
            decoded = base64.b64decode(
                encoded,
                validate=True,
            ).decode("utf-8")
        except Exception:
            return "", ""

        if ":" not in decoded:
            return "", ""

        return tuple(decoded.split(":", 1))

    def do_GET(self) -> None:
        if self.path == "/":
            self.send_body(
                200,
                page(
                    "CyberLab Hydra",
                    """
<ul>
  <li><a href="/basic">HTTP Basic Login</a></li>
  <li><a href="/login">HTTP Form Login</a></li>
  <li><a href="/health">Health</a></li>
</ul>
""",
                ),
            )
            return

        if self.path == "/health":
            self.send_body(
                200,
                b'{"status":"healthy"}\n',
                "application/json",
            )
            return

        if self.path == "/basic":
            username, password = self.parse_basic_username()

            if (
                username == VALID_USERNAME
                and password == VALID_PASSWORD
            ):
                self.send_body(
                    200,
                    page(
                        "Basic Authentication Success",
                        "<p>Welcome hydraweb</p>",
                    ),
                )
                return

            self.send_body(
                401,
                b"Basic authentication required\n",
                "text/plain; charset=utf-8",
                headers={
                    "WWW-Authenticate": (
                        'Basic realm="CyberLab Hydra"'
                    )
                },
            )
            return

        if self.path == "/login":
            self.send_body(
                200,
                page(
                    "Form Login",
                    """
<form action="/login" method="post">
  <label>Username:
    <input name="username">
  </label>
  <label>Password:
    <input name="password" type="password">
  </label>
  <button type="submit">Login</button>
</form>
""",
                ),
            )
            return

        self.send_body(
            404,
            b"CyberLab Hydra resource not found\n",
            "text/plain; charset=utf-8",
        )

    def do_HEAD(self) -> None:
        self.do_GET()

    def do_POST(self) -> None:
        if self.path != "/login":
            self.send_body(
                404,
                b"CyberLab POST resource not found\n",
                "text/plain; charset=utf-8",
            )
            return

        length = int(self.headers.get("Content-Length", "0"))
        raw_body = self.rfile.read(length)

        form = parse_qs(
            raw_body.decode("utf-8", errors="replace"),
            keep_blank_values=True,
        )

        username = form.get("username", [""])[0]
        password = form.get("password", [""])[0]

        if (
            username == VALID_USERNAME
            and password == VALID_PASSWORD
        ):
            self.send_body(
                200,
                page(
                    "Form Authentication Success",
                    "<p>Welcome hydraweb</p>",
                ),
            )
            return

        self.send_body(
            200,
            page(
                "Form Authentication Failed",
                "<p>Invalid credentials</p>",
            ),
        )

    def log_message(self, format: str, *args: object) -> None:
        basic_username, _ = self.parse_basic_username()
        user_agent = self.headers.get("User-Agent", "-")

        # Passwords are intentionally not logged.
        print(
            f'{self.client_address[0]} '
            f'[{self.log_date_time_string()}] '
            f'Method="{self.command}" '
            f'Path="{self.path}" '
            f'BasicUser="{basic_username or "-"}" '
            f'UA="{user_agent}" '
            f'{format % args}',
            flush=True,
        )


def main() -> None:
    server = HTTPServer(
        (HOST, PORT),
        HydraLabHandler,
    )

    print(f"Hydra HTTP lab listening on http://{HOST}:{PORT}")
    server.serve_forever()


if __name__ == "__main__":
    main()
```

保存并退出：

```text
Ctrl + O
Enter
Ctrl + X
```

权限：

```bash
chmod +x \
  ~/CyberLab/Hydra/server/hydra_http_lab.py
```

---

# 47. 启动 HTTP 实验服务

Ubuntu：

```bash
python3 \
  ~/CyberLab/Hydra/server/hydra_http_lab.py \
  2>&1 |
tee ~/CyberLab/Hydra/logs/http-access.log
```

预期：

```text
Hydra HTTP lab listening on http://10.10.20.20:8092
```

第二个 Ubuntu 终端：

```bash
ss -lntp | grep ':8092'
```

---

# 48. Kali 验证 HTTP 服务

```bash
nc -zvn -w 3 10.10.20.20 8092
```

```bash
curl -i \
  http://10.10.20.20:8092/
```

```bash
curl -i \
  http://10.10.20.20:8092/basic
```

预期：

```text
401 Unauthorized
WWW-Authenticate: Basic
```

---

# 49. 手工验证 HTTP Basic

错误密码：

```bash
curl -i \
  -u 'hydraweb:WrongPass-01!' \
  http://10.10.20.20:8092/basic
```

正确密码：

```bash
curl -i \
  -u 'hydraweb:LabOnly-2026!' \
  http://10.10.20.20:8092/basic
```

正确时应返回：

```text
200
Welcome hydraweb
```

---

# 50. HTTP Basic Hydra 实验

Kali：

```bash
hydra \
  -l hydraweb \
  -P ~/CyberLab/Hydra/wordlists/passwords-four.txt \
  -s 8092 \
  -t 1 \
  -W 2 \
  -w 10 \
  -f \
  -vV \
  -o ~/CyberLab/Hydra/results/http-basic-result.txt \
  -b text \
  10.10.20.20 \
  http-get \
  /basic
```

模块专用帮助：

```bash
hydra -U http-get
```

---

# 51. 为什么 HTTP Basic 容易判断？

失败：

```text
401 Unauthorized
```

成功：

```text
200 OK
```

Hydra 的 `http-get` 模块可以根据认证挑战和状态判断。

HTTP Basic 只把：

```text
username:password
```

进行 Base64 编码。

Base64 不是加密。

---

# 52. HTTP Basic 必须配合 HTTPS

在明文 HTTP 中：

```text
Authorization: Basic BASE64_VALUE
```

网络观察者可以解码凭据。

生产环境必须使用：

```text
HTTPS
```

即使使用 HTTPS，也应考虑：

- 强密码
- MFA
- Rate Limiting
- Lockout
- 日志和告警
- 限制管理接口来源

---

# 53. 手工验证 Form Login

打开登录页：

```bash
curl -i \
  http://10.10.20.20:8092/login
```

错误登录：

```bash
curl -i \
  -X POST \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'username=hydraweb' \
  --data-urlencode 'password=WrongPass-01!' \
  http://10.10.20.20:8092/login
```

响应中应包含：

```text
Invalid credentials
```

正确登录：

```bash
curl -i \
  -X POST \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'username=hydraweb' \
  --data-urlencode 'password=LabOnly-2026!' \
  http://10.10.20.20:8092/login
```

响应中应包含：

```text
Welcome hydraweb
```

---

# 54. HTTP Form 模块语法

```text
PATH:POST_DATA:CONDITION
```

本实验：

```text
/login
```

POST Body：

```text
username=^USER^&password=^PASS^
```

失败条件：

```text
F=Invalid credentials
```

完整 Module Option：

```text
/login:username=^USER^&password=^PASS^:F=Invalid credentials
```

Hydra 会把：

```text
^USER^
^PASS^
```

替换为候选值。

---

# 55. HTTP Form Hydra 实验

Kali：

```bash
hydra \
  -l hydraweb \
  -P ~/CyberLab/Hydra/wordlists/passwords-four.txt \
  -s 8092 \
  -t 1 \
  -W 2 \
  -w 10 \
  -f \
  -vV \
  -o ~/CyberLab/Hydra/results/http-form-result.txt \
  -b text \
  10.10.20.20 \
  http-post-form \
  '/login:username=^USER^&password=^PASS^:F=Invalid credentials'
```

模块帮助：

```bash
hydra -U http-post-form
```

---

# 56. 使用成功条件

也可以使用：

```text
S=Welcome hydraweb
```

命令：

```bash
hydra \
  -l hydraweb \
  -P ~/CyberLab/Hydra/wordlists/passwords-four.txt \
  -s 8092 \
  -t 1 \
  -W 2 \
  -w 10 \
  -f \
  -vV \
  10.10.20.20 \
  http-post-form \
  '/login:username=^USER^&password=^PASS^:S=Welcome hydraweb'
```

通常选择：

```text
稳定、唯一、不会出现在失败页面中的文字
```

---

# 57. F= 与 S= 的区别

## Failure Condition

```text
F=Invalid credentials
```

匹配时表示：

```text
本次登录失败
```

## Success Condition

```text
S=Welcome hydraweb
```

匹配时表示：

```text
本次登录成功
```

配置错误会导致：

- 所有密码都被判为成功
- 所有密码都被判为失败
- False Positive
- False Negative

必须先用 curl 或 Burp/ZAP 比较响应。

---

# 58. 如何确定 Form Condition？

步骤：

```text
1. 手工发送错误密码
2. 保存完整错误响应
3. 手工发送正确密码
4. 保存完整成功响应
5. 使用 diff 比较
6. 找出稳定文字、状态码、Header 或 Redirect
7. 设置 F= 或 S=
8. 只使用两个密码做预实验
```

保存响应：

```bash
curl -sS \
  -X POST \
  --data-urlencode 'username=hydraweb' \
  --data-urlencode 'password=WrongPass-01!' \
  http://10.10.20.20:8092/login \
  > ~/CyberLab/Hydra/notes/form-failure.html
```

```bash
curl -sS \
  -X POST \
  --data-urlencode 'username=hydraweb' \
  --data-urlencode 'password=LabOnly-2026!' \
  http://10.10.20.20:8092/login \
  > ~/CyberLab/Hydra/notes/form-success.html
```

比较：

```bash
diff -u \
  ~/CyberLab/Hydra/notes/form-failure.html \
  ~/CyberLab/Hydra/notes/form-success.html
```

---

# 59. Form Login 复杂情况

真实 Form 可能包含：

- CSRF Token
- Dynamic Hidden Field
- Session Cookie
- JavaScript Challenge
- Captcha
- MFA
- Redirect
- JSON Body
- Changing Error Message
- Rate Limit
- WAF
- SSO

Hydra 并不适合所有现代登录流程。

遇到复杂认证：

```text
不要尝试绕过
使用 Burp/ZAP 分析授权实验流程
与系统管理员设计安全测试
```

---

# 60. Colon 和特殊字符

`http-post-form` 使用冒号分隔：

```text
PATH:DATA:CONDITION
```

如果路径、Header 或条件中本身有冒号，可能需要转义：

```text
\:
```

具体语法查看：

```bash
hydra -U http-post-form
```

Shell 特殊字符应使用单引号保护：

```bash
'...^USER^...^PASS^...'
```

---

# 61. Cookie 和 Header

Form Login 可能需要 Cookie 或自定义 Header。

模块帮助可能支持：

```text
H=Header-Name: value
C=/cookie-path
```

具体能力随模块和版本变化。

先查看：

```bash
hydra -U http-post-form
```

不要从未知教程机械复制 Header 语法。

---

# 62. JSON Login

部分现代 API 使用：

```json
{
  "username": "user",
  "password": "pass"
}
```

Hydra HTTP Form 模块对复杂 JSON、动态 Token 和前端流程可能不理想。

本课不对 JSON Login 使用 Hydra。

后续授权测试可考虑：

- Burp Intruder
- ZAP Fuzzer
- 自建安全脚本
- API 测试工具

必须继续遵守低速和锁定控制。

---

# 63. 输出 JSON

SSH 示例：

```bash
hydra \
  -l hydra-lab-user \
  -P ~/CyberLab/Hydra/wordlists/passwords-four.txt \
  -t 1 \
  -W 2 \
  -w 10 \
  -f \
  -o ~/CyberLab/Hydra/results/ssh-result.json \
  -b json \
  ssh://10.10.20.20
```

查看：

```bash
python3 -m json.tool \
  ~/CyberLab/Hydra/results/ssh-result.json |
less
```

如果格式化失败：

```bash
head -n 30 \
  ~/CyberLab/Hydra/results/ssh-result.json
```

以当前 Hydra 输出为准。

---

# 64. 输出文件保护

```bash
chmod 600 \
  ~/CyberLab/Hydra/results/*
```

原因：

```text
结果文件包含有效测试凭据
```

不要：

- 公开上传
- 发到群聊
- 放入共享文件夹
- 保存真实密码
- 长期保留不需要的凭据


---

# 65. 认证尝试数量估算

基本组合：

```text
Attempts = User Count × Password Count
```

本课：

```text
1 User × 4 Passwords = 最多 4 次
```

使用 `-f` 后：

```text
找到正确密码立即停止
```

如果有：

```text
100 Users × 10,000 Passwords
```

就是：

```text
1,000,000 次
```

在线服务绝不能把这种数量当成普通实验。

---

# 66. 线程不是越高越好

高 `-t` 可能导致：

- SSH 连接被丢弃
- 服务器限速
- 账号快速锁定
- Hydra 误判
- 网络拥塞
- 日志难以关联
- Wazuh 大量事件
- 服务不可用

本课固定：

```text
-t 1
```

未来授权测试也应由：

```text
账号锁定策略
服务容量
测试窗口
管理员批准
```

决定并发。

---

# 67. 不使用自动密码生成

Hydra 支持：

```text
-x MIN:MAX:CHARSET
```

这会在运行时生成候选密码。

本课不使用。

原因：

```text
组合数量会快速爆炸
容易失控
不适合在线服务
```

查看帮助仅用于理解：

```bash
hydra -x -h
```

不要对实验目标运行大范围 `-x`。

---

# 68. 不使用多目标模式

Hydra 支持：

```text
-M FILE
```

用于多个 Server。

本课不使用：

```text
-M
-T
-F
```

原因：

- Scope 容易扩大
- 可能误测其他设备
- 日志和锁定风险增加
- 当前目标只有一个

---

# 69. Password Spraying 风险

Hydra 的用户循环选项和多用户文件可以形成类似 Password Spraying 的行为。

本课禁止：

```text
多个真实用户
相同常见密码
为了规避锁定而延长时间
跨多个服务测试同一密码
```

合法组织测试需要：

- 明确账号清单
- 书面授权
- 锁定阈值
- 业务窗口
- Help Desk 协调
- 测试源 IP
- 立即停止条件
- 用户通知计划

---

# 70. 使用 pw-inspector 检查测试列表

Hydra 软件包还可能提供：

```text
pw-inspector
```

查看：

```bash
pw-inspector -h
```

例如筛选：

- 最少 12 个字符
- 最多 64 个字符
- 至少包含 3 类字符
- 小写
- 大写
- 数字
- 可打印特殊字符

```bash
pw-inspector \
  -i ~/CyberLab/Hydra/wordlists/passwords-four.txt \
  -o ~/CyberLab/Hydra/wordlists/passwords-policy-filtered.txt \
  -m 12 \
  -M 64 \
  -c 3 \
  -l \
  -u \
  -n \
  -p
```

检查：

```bash
cat \
  ~/CyberLab/Hydra/wordlists/passwords-policy-filtered.txt
```

`pw-inspector` 只按字符规则过滤。

它不能判断：

- 密码是否已泄露
- 是否与用户名相关
- 是否容易被猜
- 是否重复使用
- 是否出现在组织禁止列表

---

# 71. tcpdump：SSH 实时观察

Kali 终端一：

```bash
sudo tcpdump \
  -i eth0 \
  -nn \
  'host 10.10.20.20 and tcp port 22'
```

终端二运行低速 Hydra SSH 实验。

你会看到：

- 多次 TCP Connection
- SSH Banner Exchange
- 加密后的认证流量
- Connection Close

你不会直接看到：

```text
SSH Password 明文
```

因为 SSH 会加密认证内容。

---

# 72. 保存 SSH PCAP

```bash
sudo tcpdump \
  -i eth0 \
  -nn \
  -s 0 \
  -w ~/CyberLab/Hydra/captures/hydra-ssh.pcap \
  'host 10.10.20.20 and tcp port 22'
```

另一个终端运行：

```bash
hydra \
  -l hydra-lab-user \
  -P ~/CyberLab/Hydra/wordlists/passwords-four.txt \
  -t 1 \
  -W 2 \
  -w 10 \
  -f \
  ssh://10.10.20.20
```

完成后：

```text
Ctrl + C
```

保护：

```bash
chmod 600 \
  ~/CyberLab/Hydra/captures/hydra-ssh.pcap
```

---

# 73. tcpdump：HTTP Basic

Kali：

```bash
sudo tcpdump \
  -i eth0 \
  -nn \
  -A \
  'host 10.10.20.20 and tcp port 8092'
```

另一个终端运行 HTTP Basic Hydra 实验。

明文 HTTP 中可能看到：

```text
Authorization: Basic ...
```

Base64 可以被解码，因此 PCAP 包含敏感凭据证据。

---

# 74. tcpdump：HTTP Form

保存 PCAP：

```bash
sudo tcpdump \
  -i eth0 \
  -nn \
  -s 0 \
  -w ~/CyberLab/Hydra/captures/hydra-http-form.pcap \
  'host 10.10.20.20 and tcp port 8092'
```

另一个终端运行：

```bash
hydra \
  -l hydraweb \
  -P ~/CyberLab/Hydra/wordlists/passwords-four.txt \
  -s 8092 \
  -t 1 \
  -W 2 \
  -w 10 \
  -f \
  10.10.20.20 \
  http-post-form \
  '/login:username=^USER^&password=^PASS^:F=Invalid credentials'
```

保护：

```bash
chmod 600 \
  ~/CyberLab/Hydra/captures/hydra-http-form.pcap
```

---

# 75. Wireshark：SSH

打开：

```bash
wireshark \
  ~/CyberLab/Hydra/captures/hydra-ssh.pcap
```

过滤：

```text
tcp.port == 22
```

或者：

```text
ssh
```

观察：

- TCP Streams
- SSH Protocol Banner
- 每次连接时间
- Connection Count
- Encrypted Packets
- FIN / RST

---

# 76. Wireshark：HTTP Basic

打开 HTTP PCAP 后过滤：

```text
tcp.port == 8092
```

```text
http
```

认证 Header：

```text
http.authorization
```

响应状态：

```text
http.response.code == 401
```

```text
http.response.code == 200
```

不要在截图中公开 Base64 Credential。

---

# 77. Wireshark：HTTP Form

过滤 POST：

```text
http.request.method == "POST"
```

登录路径：

```text
http.request.uri contains "/login"
```

失败页面：

```text
frame contains "Invalid credentials"
```

成功页面：

```text
frame contains "Welcome hydraweb"
```

在明文 HTTP 中，Form Body 中的密码可能直接可见。

---

# 78. Wireshark 统计

打开：

```text
Statistics
→ Conversations
```

以及：

```text
Statistics
→ HTTP
→ Requests
```

记录：

| 实验 | Connections | Attempts | Success | Duration |
|---|---:|---:|---:|---:|
| SSH |  |  |  |  |
| HTTP Basic |  |  |  |  |
| HTTP Form |  |  |  |  |

比较：

```text
SSH：认证内容加密
HTTP：认证内容可能明文可见
```

---

# 79. 查看 HTTP Access Log

Ubuntu：

```bash
less \
  ~/CyberLab/Hydra/logs/http-access.log
```

只看 Kali：

```bash
grep '10.10.20.10' \
  ~/CyberLab/Hydra/logs/http-access.log
```

只看 Basic：

```bash
grep 'Path="/basic"' \
  ~/CyberLab/Hydra/logs/http-access.log
```

只看 Form：

```bash
grep 'Path="/login"' \
  ~/CyberLab/Hydra/logs/http-access.log
```

统计请求：

```bash
grep -c '10.10.20.10' \
  ~/CyberLab/Hydra/logs/http-access.log
```

服务器故意不记录密码。

---

# 80. HTTP 日志与 Wazuh

自定义文件：

```text
~/CyberLab/Hydra/logs/http-access.log
```

默认不一定由 Wazuh Agent 收集。

所以本课使用：

```bash
logger -t hydra-lab \
  "Starting authorized HTTP authentication audit on port 8092"
```

完成：

```bash
logger -t hydra-lab \
  "Completed authorized HTTP authentication audit"
```

Wazuh 搜索：

```text
agent.name: ubuntu-target
hydra-lab
```

---

# 81. Hydra 是否一定触发 Wazuh 告警？

不一定。

## SSH

OpenSSH 认证失败通常进入：

```text
auth.log
或
systemd Journal
```

Wazuh 常能分析这些事件。

## 自定义 HTTP 服务

只有在以下条件满足时才可能自动分析：

- Wazuh 收集 Access Log
- Log 格式可解析
- 有对应 Decoder
- 有认证失败 Rule
- 事件频率达到规则条件

本课成功标准：

```text
SSH 本地日志存在
HTTP Access Log 存在
PCAP 存在
Wazuh 能看到 hydra-lab Marker
```

---

# 82. 防御：强密码

密码策略应考虑：

- 足够长度
- 避免常见密码
- 避免用户名和组织名称
- 避免季节加年份
- 避免默认密码
- 避免密码复用
- 使用 Password Manager
- 检查已泄露密码
- 对管理员使用更强要求

仅要求：

```text
大写 + 小写 + 数字 + 特殊字符
```

并不能保证密码不可猜。

---

# 83. 防御：MFA

MFA 可以降低仅凭密码登录的风险。

常见：

- FIDO2 / Security Key
- Passkey
- Authenticator App
- Push
- Smart Card
- Certificate

防止 MFA Fatigue：

- Number Matching
- 地理和设备风险判断
- 限制重复 Push
- 用户告警
- FIDO2 抗钓鱼认证

Hydra 找到密码不代表能通过 MFA。

---

# 84. 防御：禁用 SSH 密码认证

更安全的 SSH 配置可能使用：

```text
Public Key Authentication
PasswordAuthentication no
PermitRootLogin no
```

修改前：

- 保留当前 SSH Session
- 测试 Key Login
- 备份配置
- 使用 `sshd -t` 验证
- 准备 UTM Console 回滚

不要在不确认 Key 登录可用时关闭密码认证。

---

# 85. 防御：Rate Limiting

认证接口可以限制：

- 每个 IP 的请求速率
- 每个账号的失败速率
- 每个设备的尝试
- 登录失败后的延迟
- API Gateway 限速
- WAF 规则
- Fail2ban

防御不应只按 Source IP。

攻击者可能使用多个来源。

---

# 86. 防御：账号锁定

锁定策略需要平衡：

```text
防止密码猜测
与
防止攻击者恶意锁定用户
```

可考虑：

- 短期锁定
- 指数延迟
- 风险认证
- 管理员解锁
- 用户通知
- MFA
- 按账号和 IP 联合判断

不要使用容易造成大规模拒绝服务的策略。

---

# 87. 防御：统一错误消息

登录失败应避免暴露：

```text
Username does not exist
Password is wrong
Account disabled
MFA configured
```

可使用：

```text
Invalid username or password
```

但服务器日志仍应记录足够的内部原因。

---

# 88. 防御：监控和告警

监控：

- 单账号多次失败
- 单 IP 多账号失败
- 多 IP 对同一账号失败
- 成功前大量失败
- 新地理位置
- 非工作时间
- 不常见 User-Agent
- 禁用账号尝试
- 管理员账号失败
- MFA Push 激增

响应：

- 告警
- 限速
- 临时封禁
- 强制重置
- Session 撤销
- 用户联系
- 调查 Source IP

---

# 89. 完整实验一：SSH 小型审计

1. 创建 `hydra-lab-user`。
2. 设置实验密码。
3. 确认账号无 sudo 权限。
4. 手工 SSH 登录。
5. 创建四密码 Wordlist。
6. Ubuntu 写入开始 Marker。
7. Kali 抓 SSH PCAP。
8. Hydra 使用：
   - `-l`
   - `-P`
   - `-t 1`
   - `-W 2`
   - `-w 10`
   - `-f`
9. 验证结果。
10. 查看 auth.log。
11. Wazuh 搜索账号和 Marker。
12. 停止 PCAP。

---

# 90. 完整实验二：HTTP Basic

1. 启动 Ubuntu 8092 服务。
2. curl 测试无认证。
3. curl 测试错误密码。
4. curl 测试正确密码。
5. Hydra 使用 `http-get`。
6. 保存 TXT 结果。
7. 抓取 HTTP PCAP。
8. Wireshark 查看 401 和 200。
9. 解释 Base64 不等于加密。
10. 删除或保护结果。

---

# 91. 完整实验三：HTTP Form

1. curl 保存失败响应。
2. curl 保存成功响应。
3. 使用 `diff` 找到稳定文字。
4. Hydra 使用：
   - `^USER^`
   - `^PASS^`
   - `F=Invalid credentials`
5. 再用 `S=Welcome hydraweb` 比较。
6. 确认两种条件都只找到正确密码。
7. 检查 Access Log。
8. Wireshark 查看 POST Body。

---

# 92. 完整实验四：输出格式

Text：

```bash
hydra \
  -l hydraweb \
  -P ~/CyberLab/Hydra/wordlists/passwords-four.txt \
  -s 8092 \
  -t 1 \
  -f \
  -o ~/CyberLab/Hydra/results/form-text.txt \
  -b text \
  10.10.20.20 \
  http-post-form \
  '/login:username=^USER^&password=^PASS^:F=Invalid credentials'
```

JSON：

```bash
hydra \
  -l hydraweb \
  -P ~/CyberLab/Hydra/wordlists/passwords-four.txt \
  -s 8092 \
  -t 1 \
  -f \
  -o ~/CyberLab/Hydra/results/form-json.json \
  -b json \
  10.10.20.20 \
  http-post-form \
  '/login:username=^USER^&password=^PASS^:F=Invalid credentials'
```

保护：

```bash
chmod 600 \
  ~/CyberLab/Hydra/results/*
```

---

# 93. 完整实验五：防御复测

在 HTTP Lab 的复制版本中加入：

```text
每个 IP 每分钟最多少量失败
连续失败后增加延迟
记录失败计数
返回 429 Too Many Requests
```

端口改为：

```text
8093
```

再次使用相同四密码列表和单线程测试。

记录：

| 项目 | 8092 | 8093 |
|---|---|---|
| Attempts |  |  |
| Duration |  |  |
| 429 |  |  |
| Hydra Result |  |  |
| Access Log |  |  |

不要设计绕过 Rate Limit 的实验。

---

# 94. 实验报告模板

## Lab Information

```text
Lab Name：Hydra Authorized Credential Audit
Date：
Student：
Testing Host：Kali 10.10.20.10
Target：Ubuntu 10.10.20.20
Authorized Services：
- SSH 22
- HTTP 8092
Authorized Accounts：
- hydra-lab-user
- hydraweb
Hydra Version：
```

## Safety Controls

```text
Dedicated Accounts：
Wordlist Size：
Threads：
Connection Delay：
Timeout：
Stop on Success：
Lockout Policy Checked：
Snapshot：
Test Window：
```

## Commands

```bash
# SSH
# HTTP Basic
# HTTP Form
```

## Results

| Service | Account | Attempts | Valid Credential | Duration |
|---|---|---:|---|---:|
| SSH |  |  |  |  |
| HTTP Basic |  |  |  |  |
| HTTP Form |  |  |  |  |

## Form Detection

```text
Failure Marker：
Success Marker：
Status Codes：
Redirect：
False Positive Test：
```

## Log Evidence

```text
SSH auth.log：
HTTP Access Log：
Wazuh Event：
Source IP：
Failed Count：
Successful Count：
```

## Network Evidence

| PCAP | Protocol | Credentials Visible? | Reason |
|---|---|---|---|
| SSH | SSH | No | Encrypted |
| HTTP Basic | HTTP | Yes | Base64 over plaintext |
| HTTP Form | HTTP | Yes | Plain POST Body |

## Risk

```text
Password Strength：
MFA：
Rate Limiting：
Account Lockout：
SSH Password Authentication：
Monitoring：
```

## Recommendations

```text
Use strong unique passwords.
Enable MFA or passkeys where supported.
Use SSH keys and disable password authentication where appropriate.
Apply rate limiting and balanced lockout controls.
Use HTTPS for all Web authentication.
Monitor repeated failures and success-after-failure patterns.
Remove default and test accounts.
```

## Cleanup

```text
HTTP Server Stopped：
Test Account Locked/Deleted：
Password Files Deleted/Protected：
Result Files Redacted：
PCAP Protected：
Wazuh Marker Completed：
```

---

# 95. 常见错误：hydra command not found

```bash
sudo apt update
sudo apt install hydra
```

检查：

```bash
which hydra
hydra -h
```

---

# 96. SSH 模块不可用

查看：

```bash
hydra -h
```

确认支持列表中有：

```text
ssh
```

模块帮助：

```bash
hydra -U ssh
```

Kali 官方包通常包含 SSH 依赖，但实际以本机输出为准。

---

# 97. SSH Connection refused

Kali：

```bash
nc -zvn -w 3 10.10.20.20 22
```

Ubuntu：

```bash
sudo systemctl status ssh --no-pager
```

```bash
ss -lntp | grep ':22'
```

---

# 98. Password authentication disabled

Ubuntu：

```bash
sudo sshd -T |
grep passwordauthentication
```

如果：

```text
passwordauthentication no
```

Hydra 密码测试不会成功。

不要因此降低真实服务器安全性。

使用 HTTP 8092 Lab 完成本课即可。

---

# 99. Hydra 报所有 Form 密码都成功

原因通常是：

```text
失败条件错误
失败文字不存在
页面是动态的
Redirect 判断错误
```

重新：

1. curl 错误登录
2. curl 正确登录
3. 保存两个响应
4. `diff -u`
5. 选择稳定 `F=` 或 `S=`
6. 先用两个密码测试

---

# 100. Hydra 报所有 Form 密码都失败

检查：

- 正确密码是否在 Wordlist
- Username 是否正确
- Field 名称是否为 `username` 和 `password`
- Path 是否 `/login`
- Port 是否 8092
- 成功文字是否完全匹配
- Shell 引号
- Special Character
- HTTP 服务是否正在运行

手工 curl 正确登录必须先成功。

---

# 101. Form Module Syntax Error

查看：

```bash
hydra -U http-post-form
```

检查：

```text
PATH:DATA:CONDITION
^USER^
^PASS^
F=
S=
冒号转义
单引号
```

不要复制不同网站的 Form 字段。

---

# 102. Too many connection errors

降低：

```text
-t 1
-W 2
-w 10
```

检查服务：

```bash
nc -zvn -w 3 10.10.20.20 PORT
```

查看目标日志是否限速或封禁。

不要提高并发“解决”连接错误。

---

# 103. Hydra 等待 Restore

检查当前目录：

```bash
ls -l hydra.restore
```

只有确认旧任务 Scope 与当前完全一致时才使用：

```bash
hydra -R
```

要忽略旧 Restore：

```bash
hydra -I ...
```

更安全的方法：

```text
把旧 Restore 移到 notes 目录
重新输入经过审查的命令
```

---

# 104. 找到密码但手工登录失败

可能原因：

- Form 条件错误导致 False Positive
- 账号已经锁定
- 服务状态变化
- 密码包含不可见字符
- Wordlist 使用 CRLF
- 目标或端口错误
- 登录需要第二阶段
- Hydra Module 不适配该流程

检查 Wordlist：

```bash
cat -A \
  ~/CyberLab/Hydra/wordlists/passwords-four.txt
```

转换 CRLF：

```bash
sed -i 's/\r$//' \
  ~/CyberLab/Hydra/wordlists/passwords-four.txt
```

---

# 105. Password 中有特殊字符

使用文件通常最安全。

单个密码使用单引号：

```bash
-p 'LabOnly-2026!'
```

不要把真实密码直接放在：

- Shell History
- Screenshot
- Git
- Chat
- 报告正文

当前密码只是实验凭据，课程结束后删除账号。

---

# 106. 清理 HTTP 服务

在运行 Server 的 Ubuntu 终端：

```text
Ctrl + C
```

确认端口关闭：

```bash
ss -lntp | grep ':8092'
```

没有输出表示服务已停止。

---

# 107. 清理 SSH 测试账号

先锁定：

```bash
sudo usermod -L hydra-lab-user
```

确认没有登录进程：

```bash
pgrep -a -u hydra-lab-user
```

删除账号和 Home：

```bash
sudo userdel -r hydra-lab-user
```

确认：

```bash
id hydra-lab-user
```

预期：

```text
no such user
```

---

# 108. 清理凭据文件

先保留一份不含密码的报告。

然后删除：

```bash
rm -f \
  ~/CyberLab/Hydra/wordlists/passwords-four.txt \
  ~/CyberLab/Hydra/wordlists/test-pairs.txt \
  ~/CyberLab/Hydra/results/*.txt \
  ~/CyberLab/Hydra/results/*.json
```

在 VM、SSD 和 Copy-on-Write 文件系统中：

```text
普通删除不等于可靠物理擦除
```

保护敏感数据的主要方式是：

- 不使用真实凭据
- 全盘加密
- 最少保存
- 限制权限
- 实验后删除 VM Clone 或恢复 Snapshot

---

# 109. 清理 PCAP

PCAP 可能包含 HTTP 明文密码。

实验完成并记录统计后：

```bash
rm -f \
  ~/CyberLab/Hydra/captures/hydra-http-form.pcap
```

SSH PCAP 不含明文密码，但仍包含内部 IP 和时间。

按你的实验保留策略处理。

---

# 110. 最终 Wazuh Marker

Ubuntu：

```bash
logger -t hydra-lab \
  "Hydra lab cleanup completed; dedicated account and HTTP service removed"
```

Wazuh 搜索：

```text
hydra-lab
```

---

# 111. 本课练习

1. 检查 Hydra 版本和支持模块。
2. 创建实验目录。
3. 创建专用 SSH 用户。
4. 确认用户没有 sudo 权限。
5. 确认 SSH 密码认证状态。
6. 手工 SSH 登录。
7. 创建一个 Username 文件。
8. 创建四密码 Wordlist。
9. 设置文件权限为 600。
10. 计算尝试数量。
11. 运行单线程 SSH Hydra。
12. 使用 `-f`。
13. 使用 `-W 2`。
14. 使用 `-w 10`。
15. 保存 Text 结果。
16. 保存 JSON 结果。
17. 使用 `-C` Pair 文件。
18. 查看 auth.log。
19. Wazuh 搜索账号。
20. 创建 HTTP 8092 服务。
21. curl 测试 HTTP Basic。
22. Hydra 测试 HTTP Basic。
23. curl 测试 Form 失败。
24. curl 测试 Form 成功。
25. 使用 diff 比较。
26. Hydra 使用 `F=`。
27. Hydra 使用 `S=`。
28. 验证没有 False Positive。
29. 查看 HTTP Access Log。
30. 保存 SSH PCAP。
31. 保存 HTTP PCAP。
32. Wireshark 比较 SSH 和 HTTP。
33. 使用 `pw-inspector`。
34. 检查锁定配置。
35. 记录防御建议。
36. 停止 HTTP 服务。
37. 删除测试账号。
38. 删除或保护凭据结果。
39. 写入清理 Marker。
40. 完成实验报告。

---

# 112. 复习题

1. Hydra 的主要用途是什么？
2. Hydra 与 John/Hashcat 有何区别？
3. Password Guessing 是什么？
4. Password Spraying 是什么？
5. Credential Stuffing 是什么？
6. 为什么本课不使用真实凭据？
7. `-l` 做什么？
8. `-L` 做什么？
9. `-p` 做什么？
10. `-P` 做什么？
11. `-C` 做什么？
12. `-t` 做什么？
13. `-T` 做什么？
14. `-w` 做什么？
15. `-W` 做什么？
16. `-f` 与 `-F` 有何区别？
17. `-s` 做什么？
18. `-S` 做什么？
19. `-o` 做什么？
20. `-b` 做什么？
21. `-v` 与 `-V` 有何区别？
22. `-R` 做什么？
23. `-I` 做什么？
24. 为什么本课使用 `-t 1`？
25. 为什么不使用 rockyou.txt？
26. `^USER^` 和 `^PASS^` 是什么？
27. `F=` 做什么？
28. `S=` 做什么？
29. 为什么要先用 curl 测试 Form？
30. HTTP Basic 的 Base64 是加密吗？
31. SSH PCAP 中为什么看不到明文密码？
32. HTTP Form PCAP 为什么可能看到密码？
33. Hydra 是否可以绕过 MFA？
34. 为什么账号锁定策略需要平衡？
35. 为什么统一错误消息有帮助？
36. 为什么 User-Agent 不能作为唯一检测证据？
37. Wazuh 为什么不一定看到自定义 HTTP 失败？
38. 找到密码后为什么应停止？
39. 为什么实验后删除测试账号？
40. 为什么不能对未授权系统运行 Hydra？

---

# 113. 参考答案

1. 对网络认证服务进行授权的账号密码组合测试。
2. Hydra 在线访问服务；John/Hashcat 离线恢复 Hash。
3. 对账号尝试少量候选密码。
4. 对多个账号尝试相同或少量密码。
5. 使用泄露的真实凭据组合登录其他服务。
6. 避免泄露、误用和真实账号影响。
7. 指定一个 Username。
8. 从文件读取多个 Username。
9. 指定一个 Password。
10. 从文件读取多个 Password。
11. 读取 `username:password` Pair。
12. 每个目标的并行任务数。
13. 多目标时的总并行数。
14. 等待响应的超时。
15. 每个线程连接之间的等待时间。
16. `-f` 对单目标找到后停止；`-F` 对多目标全局停止。
17. 指定非默认端口。
18. 使用 SSL/TLS。
19. 保存找到的凭据结果。
20. 指定输出格式。
21. `-v` 显示详细过程；`-V` 显示每次尝试。
22. 恢复中断任务。
23. 忽略已有 Restore 文件。
24. 降低负载、锁定和误判风险。
25. 数量巨大且不适合基础在线实验。
26. Hydra Form Module 的 Username 和 Password 占位符。
27. 定义失败响应特征。
28. 定义成功响应特征。
29. 必须确定字段、路径和稳定的成功/失败差异。
30. 不是。
31. SSH 加密认证内容。
32. HTTP POST Body 是明文。
33. 不能，找到密码不等于完成第二因素。
34. 过严可能被攻击者利用来锁定用户。
35. 减少 Username Enumeration。
36. User-Agent 可以修改。
37. 需要收集该文件并配置 Decoder/Rule。
38. 避免多余尝试和账号影响。
39. 防止弱测试凭据被遗忘。
40. 可能违法并造成账号和服务影响。

---

# 114. 命令速查表

## SSH

```bash
hydra \
  -l hydra-lab-user \
  -P passwords-four.txt \
  -t 1 \
  -W 2 \
  -w 10 \
  -f \
  ssh://10.10.20.20
```

## 单个密码

```bash
hydra \
  -l hydra-lab-user \
  -p 'LabOnly-2026!' \
  -t 1 \
  -f \
  ssh://10.10.20.20
```

## Pair 文件

```bash
hydra \
  -C test-pairs.txt \
  -t 1 \
  -f \
  ssh://10.10.20.20
```

## HTTP Basic

```bash
hydra \
  -l hydraweb \
  -P passwords-four.txt \
  -s 8092 \
  -t 1 \
  -W 2 \
  -f \
  10.10.20.20 \
  http-get \
  /basic
```

## HTTP Form Failure Marker

```bash
hydra \
  -l hydraweb \
  -P passwords-four.txt \
  -s 8092 \
  -t 1 \
  -W 2 \
  -f \
  10.10.20.20 \
  http-post-form \
  '/login:username=^USER^&password=^PASS^:F=Invalid credentials'
```

## HTTP Form Success Marker

```bash
hydra \
  -l hydraweb \
  -P passwords-four.txt \
  -s 8092 \
  -t 1 \
  -W 2 \
  -f \
  10.10.20.20 \
  http-post-form \
  '/login:username=^USER^&password=^PASS^:S=Welcome hydraweb'
```

## Module Help

```bash
hydra -U ssh
hydra -U http-get
hydra -U http-post-form
```

## JSON Output

```bash
hydra \
  -l hydraweb \
  -P passwords-four.txt \
  -s 8092 \
  -t 1 \
  -f \
  -o result.json \
  -b json \
  10.10.20.20 \
  http-get \
  /basic
```

---

# 115. 核心记忆口诀

```text
Hydra 是在线认证测试
John/Hashcat 是离线 Hash 恢复

-l 一个用户
-L 用户列表
-p 一个密码
-P 密码列表
-C 用户密码 Pair

-t 控制并发
-W 控制连接间隔
-w 控制超时
-f 找到就停止
-s 指定端口
-S 使用 TLS
-o 保存结果
-b 选择格式

Form：
^USER^ 替换用户
^PASS^ 替换密码
F= 识别失败
S= 识别成功

先 curl
再 Hydra
先小列表
再看日志
找到就停
实验后清理
```

---

# 116. 完成标准

完成本课后，你应该拥有：

```text
Hydra 版本记录
支持模块记录
专用 SSH 测试账号
一个 Username Wordlist
四密码 Wordlist
SSH Hydra 结果
HTTP Basic 实验服务
HTTP Basic Hydra 结果
HTTP Form Hydra 结果
F= 和 S= 对比
Text 和 JSON 输出
SSH auth.log 证据
HTTP Access Log
SSH PCAP
HTTP PCAP
Wireshark 对比
Wazuh hydra-lab Marker
防御建议
账号和服务清理记录
完整 Authorized Credential Audit 报告
```

检查 Kali：

```bash
find ~/CyberLab/Hydra -type f |
sort
```

检查 Ubuntu：

```bash
find ~/CyberLab/Hydra -type f |
sort
```

确认账号已删除：

```bash
id hydra-lab-user
```

---

# 117. 下一课

下一课建议：

```text
John the Ripper
```

下一课学习内容：

- 在线认证与离线 Hash 的区别
- Hash、Salt 和 KDF
- John Format
- `john`、`unshadow` 和 `john --show`
- Wordlist Mode
- Single Crack Mode
- Incremental Mode 的风险
- Rule 基础
- Linux `/etc/shadow` 实验副本
- ZIP 和 Office 测试文件 Hash 提取
- Session、Restore 和 Pot File
- CPU 与 ARM64 性能
- Hash 安全存储
- Wazuh 文件访问和实验 Marker
- 密码修复和复测

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
SQLmap：SQL Injection
SearchSploit：公开 PoC 研究
Metasploit：模块化验证框架
Hydra：在线认证安全测试
```

---

# 118. 参考说明

本指南的功能和参数依据：

```text
THC Hydra 官方 GitHub Repository
Kali Linux 官方 Hydra 工具页面
本机 hydra -h
本机 hydra -U MODULE
pw-inspector -h
```

当前 Kali 官方工具页面列出的 Hydra 主要选项包括：

```text
-l/-L
-p/-P
-C
-e
-t/-T
-w/-W
-f/-F
-s
-S
-o
-b
-R
-I
```

Hydra 支持的认证协议和模块会随版本、依赖和编译设置变化。

最终以你的 Kali 输出为准：

```bash
hydra -h
hydra -U ssh
hydra -U http-get
hydra -U http-post-form
```
