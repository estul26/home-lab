# 第十课：OWASP ZAP 完整学习指南

> **实验环境：你自己的 CyberLab**
>
> 本课程只用于你自己创建、拥有并控制的 Web 服务。  
> 不要对学校、公司、公共网站、随机公网 IP 或未经明确授权的系统执行 Spider、Fuzz 或 Active Scan。

---

# 1. 本课目标

完成本课后，你应该能够：

```text
理解 OWASP ZAP 的用途和核心架构
区分 Passive Scan 与 Active Scan
理解 Active Scan 为什么属于真实攻击行为
安装并启动 Kali 中的 ZAP
使用 ZAP 内置浏览器访问实验目标
设置 Context、Scope 和 Protected Mode
使用 History、Sites、Request、Response 和 Alerts
设置 Breakpoint 并修改 HTTP 请求
使用 Manual Request Editor / Requester 重放请求
使用 Traditional Spider 发现普通链接
使用 Client Spider 探索现代 Web 应用
理解 AJAX Spider 已不再是现代应用的首选
阅读 Passive Scan Alert
只对授权实验服务运行受控 Active Scan
配置低强度 Scan Policy
使用 Fuzzer 进行小规模参数替换
生成 HTML、JSON、Markdown、PDF 或 XML 报告
保存 Session 和实验记录
使用 Automation Framework 执行 Passive Baseline
结合 tcpdump、Wireshark、Access Log 和 Wazuh 观察 ZAP
修复问题并进行 Retest
```

你的 CyberLab：

```text
Kali-CyberLab      10.10.20.10
Ubuntu-Target      10.10.20.20
Windows-Target     10.10.20.30
Wazuh-SIEM         10.10.20.40
Network            10.10.20.0/24
```

本课目标：

```text
OWASP ZAP：Kali 10.10.20.10
实验 Web Server：Ubuntu 10.10.20.20:8087
ZAP 本地 Proxy：通常 127.0.0.1:8080
```

---

# 2. 什么是 OWASP ZAP？

OWASP ZAP，全名：

```text
Zed Attack Proxy
```

它是一个免费、开源、集成式 Web 应用安全测试工具。

ZAP 可以：

- 作为浏览器与服务器之间的代理
- 记录 HTTP 和 HTTPS
- 被动分析请求和响应
- 建立 Sites Tree
- 爬取页面和链接
- 修改和重发请求
- Fuzz 参数
- 主动测试潜在漏洞
- 管理 Scope 和认证上下文
- 生成多种格式报告
- 通过 API 或 YAML 自动化

简单理解：

```text
浏览器
   ↓
ZAP Proxy
   ↓
Web Server
```

---

# 3. ZAP 与 Burp Suite 的关系

ZAP 和 Burp Suite 都可以：

- 代理 HTTP/HTTPS
- 拦截和修改请求
- 查看 History
- 重放请求
- Fuzz 输入
- 管理目标范围
- 分析 Web 安全问题

主要区别：

| 项目 | ZAP | Burp Community |
|---|---|---|
| 价格 | 免费开源 | 免费但闭源 |
| Passive Scan | 完整支持 | 手动工具为主 |
| Active Scan | 免费支持 | 自动 Scanner 属于 Professional |
| Automation Framework | 内置 YAML 框架 | Community 自动化有限 |
| 报告 | 多模板、多格式 | Community 报告有限 |
| 初学自动扫描 | 较方便 | 更偏手工分析 |
| Repeater 类工具 | Requester / Manual Request | Repeater |
| Scope 安全模式 | Protected Mode | Target Scope |

记忆：

```text
Burp Community：手工请求分析非常强
ZAP：免费代理 + Passive Scan + Active Scan + Automation
```

---

# 4. ZAP 与前面课程的关系

```text
Nmap：发现 Web 端口
curl：手动发送 HTTP 请求
WhatWeb：识别技术指纹
Gobuster/ffuf：发现目录和参数
Nikto：检查 Web Server 配置
Burp Suite：拦截和手工修改请求
OWASP ZAP：代理、被动分析、爬虫、主动扫描和自动化
tcpdump/Wireshark：网络证据
Wazuh：服务器和端点日志
```

---

# 5. Passive Scan 与 Active Scan

## Passive Scan

Passive Scan 分析已经经过 ZAP 的 HTTP、HTTPS 或 WebSocket 消息。

它通常不会：

```text
为了测试漏洞而额外攻击目标
```

它可能发现：

- 缺失安全 Header
- Cookie 缺少安全属性
- 信息泄露
- 不安全缓存配置
- 私有 IP 泄露
- 可疑注释
- 混合内容
- 表单安全线索

## Active Scan

Active Scan 会：

```text
向目标发送已知攻击形式的请求
```

例如：

- 修改参数
- 发送特殊字符
- 测试注入线索
- 测试 XSS 反射
- 测试路径和方法
- 对多个输入点重复发送请求

因此：

```text
Active Scan 是真实攻击行为
只能对你拥有或明确获授权的目标运行
```

---

# 6. ZAP 的四种 Mode

ZAP 常见 Mode：

| Mode | 行为 |
|---|---|
| Safe | 不允许潜在危险操作 |
| Protected | 只允许对 Scope 内 URL 执行危险操作 |
| Standard | 不限制目标 |
| Attack | 新发现的 Scope 节点可能自动进行 Active Scan |

本课必须使用：

```text
Protected Mode
```

不要使用：

```text
Attack Mode
```

Protected Mode 的好处：

```text
即使误点 Active Scan
也只能作用于明确加入 Scope 的实验目标
```

---

# 7. Scope 与 Context

## Scope

Scope 是本次允许测试的 URL 集合。

默认：

```text
没有任何 URL 在 Scope 中
```

## Context

Context 是用于描述一个 Web 应用的配置容器。

它可以包含：

- Include Regex
- Exclude Regex
- Authentication
- Session Management
- Users
- Technology
- Scope
- Anti-CSRF 配置
- Data Driven Nodes

本课 Context：

```text
Name：CyberLab-ZAP
Include：http://10.10.20.20:8087.*
```

---

# 8. 检查和安装 ZAP

Kali：

```bash
which zaproxy
```

也可能存在：

```bash
which owasp-zap
```

查看版本：

```bash
zaproxy -version
```

查看帮助：

```bash
zaproxy -h
```

安装：

```bash
sudo apt update
sudo apt install zaproxy
```

启动：

```bash
zaproxy
```

Kali 当前软件包可能同时提供：

```text
zaproxy
owasp-zap
```

---

# 9. 检查 Java 和资源

ZAP 使用 Java。

检查：

```bash
java -version
```

查看内存：

```bash
free -h
```

你的 Kali 建议至少分配：

```text
RAM：4 GB
CPU：2 核
```

如果同时运行：

```text
Kali + Ubuntu + Wazuh + Windows
```

可暂时关闭不参与本课的 Windows Target。

---

# 10. 创建实验目录

Kali：

```bash
mkdir -p ~/CyberLab/ZAP/{sessions,reports,captures,automation,notes,screenshots}
```

Ubuntu：

```bash
mkdir -p ~/CyberLab/ZAP/{server,logs}
```

检查：

```bash
find ~/CyberLab/ZAP -maxdepth 2 -type d
```

---

# 11. Ubuntu 受控 ZAP 实验服务器

本实验服务器提供以下线索：

```text
缺失部分安全 Header
Cookie 缺少 Secure
反射型输入实验
CORS 通配符
Redirect
JSON API
表单
状态码
robots.txt
隐藏链接
可变参数
```

它不会：

```text
执行系统命令
读写敏感文件
连接外部数据库
提供真实账号绕过
破坏操作系统
```

---

# 12. 创建实验服务器

Ubuntu：

```bash
nano ~/CyberLab/ZAP/server/zap_lab.py
```

粘贴：

```python
#!/usr/bin/env python3

import html
import json
from http.server import BaseHTTPRequestHandler, HTTPServer
from urllib.parse import parse_qs, quote, urlparse


HOST = "10.10.20.20"
PORT = 8087


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


class ZapLabHandler(BaseHTTPRequestHandler):
    server_version = "CyberLabZAP/1.0"

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
        self.send_header("X-CyberLab", "OWASP-ZAP-Lab")

        if headers:
            for name, value in headers.items():
                self.send_header(name, value)

        self.end_headers()

        if self.command != "HEAD":
            self.wfile.write(body)

    def do_GET(self) -> None:
        parsed = urlparse(self.path)
        path = parsed.path
        query = parse_qs(parsed.query)

        if path == "/":
            body = """
<ul>
  <li><a href="/search?q=student">Search</a></li>
  <li><a href="/item?id=1">Item 1</a></li>
  <li><a href="/item?id=42">Item 42</a></li>
  <li><a href="/api/status">API Status</a></li>
  <li><a href="/cors">CORS Endpoint</a></li>
  <li><a href="/cookie">Cookie Example</a></li>
  <li><a href="/redirect?next=/final">Redirect</a></li>
  <li><a href="/robots.txt">robots.txt</a></li>
  <li><a href="/hidden/">Hidden Area</a></li>
</ul>

<form method="post" action="/form">
  <label>Name:
    <input name="name" value="student">
  </label>
  <label>Message:
    <input name="message" value="hello">
  </label>
  <button type="submit">Submit</button>
</form>
"""
            self.send_body(
                200,
                page("CyberLab OWASP ZAP", body),
                headers={
                    "Set-Cookie": "zap_session=training; Path=/"
                },
            )
            return

        if path == "/search":
            value = query.get("q", [""])[0]

            # Intentionally reflected without HTML escaping.
            body = (
                "<!doctype html><html><head>"
                "<title>Search</title></head><body>"
                "<h1>Search Result</h1>"
                f"<p>You searched for: {value}</p>"
                "</body></html>"
            ).encode("utf-8")

            self.send_body(200, body)
            return

        if path == "/item":
            item_id = query.get("id", [""])[0]
            items = {
                "1": {"id": 1, "name": "Keyboard", "price": 25},
                "2": {"id": 2, "name": "Mouse", "price": 15},
                "42": {"id": 42, "name": "Lab Answer", "price": 42},
            }

            if item_id in items:
                self.send_body(
                    200,
                    json.dumps(
                        items[item_id],
                        indent=2,
                    ).encode("utf-8"),
                    "application/json",
                )
            else:
                self.send_body(
                    404,
                    b'{"error":"item not found"}\n',
                    "application/json",
                )
            return

        if path == "/api/status":
            self.send_body(
                200,
                b'{"service":"zap-lab","status":"online"}\n',
                "application/json",
            )
            return

        if path == "/cors":
            origin = self.headers.get("Origin", "*")
            self.send_body(
                200,
                b'{"cors":"training endpoint"}\n',
                "application/json",
                headers={
                    "Access-Control-Allow-Origin": origin,
                    "Access-Control-Allow-Credentials": "true",
                },
            )
            return

        if path == "/cookie":
            self.send_body(
                200,
                page(
                    "Cookie Example",
                    "<p>Training cookie created.</p>",
                ),
                headers={
                    "Set-Cookie": "lab_cookie=value; Path=/"
                },
            )
            return

        if path == "/redirect":
            target = query.get("next", ["/final"])[0]
            self.send_response(302)
            self.send_header("Location", target)
            self.send_header("Content-Length", "0")
            self.end_headers()
            return

        if path == "/final":
            self.send_body(
                200,
                page(
                    "Final Page",
                    "<p>Redirect completed.</p>",
                ),
            )
            return

        if path == "/robots.txt":
            self.send_body(
                200,
                b"User-agent: *\nDisallow: /hidden/\n",
                "text/plain; charset=utf-8",
            )
            return

        if path in {"/hidden", "/hidden/"}:
            self.send_body(
                200,
                page(
                    "Hidden Area",
                    "<p>This path is for discovery practice.</p>",
                ),
            )
            return

        if path.startswith("/status/"):
            try:
                status = int(path.rsplit("/", 1)[1])
            except ValueError:
                status = 400

            if status < 100 or status > 599:
                status = 400

            self.send_body(
                status,
                f"CyberLab status {status}\n".encode("utf-8"),
                "text/plain; charset=utf-8",
            )
            return

        self.send_body(
            404,
            b"CyberLab ZAP resource not found\n",
            "text/plain; charset=utf-8",
        )

    def do_HEAD(self) -> None:
        self.do_GET()

    def do_POST(self) -> None:
        length = int(self.headers.get("Content-Length", "0"))
        raw_body = self.rfile.read(length)

        if self.path == "/form":
            form = parse_qs(
                raw_body.decode("utf-8", errors="replace")
            )
            self.send_body(
                200,
                json.dumps(
                    {
                        "received": form,
                        "message": "Training form accepted",
                    },
                    indent=2,
                ).encode("utf-8"),
                "application/json",
            )
            return

        if self.path == "/api/echo":
            try:
                data = json.loads(
                    raw_body.decode("utf-8", errors="strict")
                )
                status = 200
                response = {
                    "valid_json": True,
                    "received": data,
                }
            except (json.JSONDecodeError, UnicodeDecodeError):
                status = 400
                response = {
                    "valid_json": False,
                    "error": "invalid JSON",
                }

            self.send_body(
                status,
                json.dumps(
                    response,
                    indent=2,
                ).encode("utf-8"),
                "application/json",
            )
            return

        self.send_body(
            404,
            b"CyberLab POST path not found\n",
            "text/plain; charset=utf-8",
        )

    def log_message(self, format: str, *args: object) -> None:
        user_agent = self.headers.get("User-Agent", "-")
        x_zap = self.headers.get("X-ZAP-Initiator", "-")
        print(
            f'{self.client_address[0]} '
            f'[{self.log_date_time_string()}] '
            f'UA="{user_agent}" '
            f'X-ZAP-Initiator="{x_zap}" '
            f'{format % args}',
            flush=True,
        )


def main() -> None:
    server = HTTPServer((HOST, PORT), ZapLabHandler)
    print(f"ZAP lab listening on http://{HOST}:{PORT}")
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

赋予权限：

```bash
chmod +x ~/CyberLab/ZAP/server/zap_lab.py
```

---

# 13. 启动服务器并保存 Access Log

Ubuntu：

```bash
python3 ~/CyberLab/ZAP/server/zap_lab.py \
  2>&1 |
tee ~/CyberLab/ZAP/logs/access.log
```

预期：

```text
ZAP lab listening on http://10.10.20.20:8087
```

第二个 Ubuntu 终端：

```bash
ss -lntp | grep ':8087'
```

---

# 14. Kali 直接验证

```bash
ping -c 3 10.10.20.20
```

```bash
nc -zvn -w 3 10.10.20.20 8087
```

```bash
curl -i http://10.10.20.20:8087/
```

```bash
curl -i \
  'http://10.10.20.20:8087/search?q=student'
```

```bash
sudo nmap -n -sV -p 8087 10.10.20.20
```

---

# 15. 第一次启动 ZAP

Kali：

```bash
zaproxy
```

首次启动可能询问是否保存 Session。

学习阶段可选择：

```text
No, I do not want to persist this session
```

完成实验后再学习保存 Session。

---

# 16. 检查 Proxy Listener

进入：

```text
Tools
→ Options
→ Network
→ Local Servers / Proxies
```

不同版本 UI 文字可能略有变化。

常见默认：

```text
Address：localhost / 127.0.0.1
Port：8080
```

Kali 检查：

```bash
ss -lntp | grep ':8080'
```

不要绑定：

```text
0.0.0.0
All interfaces
10.10.20.10
```

除非你明确需要其他设备通过该 Proxy，并且理解暴露风险。

---

# 17. 使用 ZAP 内置浏览器

进入：

```text
Quick Start
→ Manual Explore
```

URL：

```text
http://10.10.20.20:8087/
```

选择浏览器后：

```text
Launch Browser
```

ZAP 启动的浏览器通常已经：

- 使用 ZAP Proxy
- 信任 ZAP 生成的 CA
- 可用于 HTTP 和 HTTPS
- 便于 Client Spider 和浏览器事件分析

---

# 18. Passive Scan 第一次实验

在 ZAP 浏览器中依次访问：

```text
/
 /search?q=student
 /item?id=1
 /cookie
 /cors
 /redirect?next=/final
 /robots.txt
 /hidden/
```

ZAP 会自动记录流量。

查看：

```text
History
Sites
Alerts
```

Passive Scan 在后台分析已经看到的响应。

---

# 19. History Tab

History 中常见字段：

```text
ID
Method
URL
Response Code
Response Length
RTT
Highest Alert
Tags
Note
```

点击一条请求，查看：

```text
Request
Response
```

推荐先筛选：

```text
Site = http://10.10.20.20:8087
```

---

# 20. Sites Tree

Sites Tree 会按层级显示：

```text
http://10.10.20.20:8087
├── search
├── item
├── api
│   └── status
├── cors
├── cookie
├── redirect
├── robots.txt
└── hidden
```

Sites Tree 表示 ZAP 已经知道的服务器端 URL 结构。

它不一定包含所有真实路径。

---

# 21. Alerts Tab

Passive Scan 可能报告：

- Cookie Without Secure Flag
- Cookie No HttpOnly Flag
- Missing Anti-clickjacking Header
- Content Security Policy Header Not Set
- CORS Misconfiguration
- Information Disclosure
- Cache-Control 问题
- Absence of Anti-CSRF Token
- Private IP Disclosure

实际结果取决于：

```text
ZAP 版本
已安装 Add-ons
Passive Scan Rules
访问过的页面
```

---

# 22. Alert 的 Risk 与 Confidence

ZAP Alert 通常包含：

```text
Risk
Confidence
Description
URL
Parameter
Attack
Evidence
Other Info
Solution
Reference
CWE
WASC
```

Risk 常见：

```text
High
Medium
Low
Informational
```

Confidence 常见：

```text
Low
Medium
High
Confirmed
False Positive
```

重要：

```text
Risk 不是最终业务风险
Confidence 不是绝对证明
```

---

# 23. 创建 Context 并加入 Scope

在 Sites Tree 右键：

```text
http://10.10.20.20:8087
```

选择类似：

```text
Include in Context
→ New Context
```

Context Name：

```text
CyberLab-ZAP
```

Include Regex 应类似：

```text
\Qhttp://10.10.20.20:8087\E.*
```

然后：

```text
Include in Scope
```

---

# 24. 切换 Protected Mode

工具栏中的 Mode 下拉菜单：

```text
Protected Mode
```

确认只有：

```text
http://10.10.20.20:8087
```

在 Scope 中。

不要把以下内容加入 Scope：

```text
Wazuh 10.10.20.40
随机网站
浏览器更新域名
搜索引擎
ZAP Marketplace
```

---

# 25. Scope 过滤

在 History 中使用：

```text
Show all URLs
或
Show URLs in Scope
```

在本课中推荐：

```text
Show URLs in Scope
```

这样可以减少浏览器后台请求造成的干扰。

---

# 26. Breakpoint：拦截请求

在 ZAP 工具栏启用：

```text
Set break on all requests and responses
```

或在 History 中对特定 URL 设置 Breakpoint。

浏览器访问：

```text
http://10.10.20.20:8087/item?id=1
```

ZAP 会暂停请求。

你可以把：

```text
id=1
```

改成：

```text
id=42
```

然后：

```text
Submit and continue
```

---

# 27. Request Break 与 Response Break

## Request Break

请求发送前暂停。

适合修改：

- Query Parameter
- Header
- Cookie
- Body
- HTTP Method
- Path

## Response Break

服务器响应返回浏览器前暂停。

适合观察或修改：

- Response Header
- Cookie
- Body
- Redirect Location

修改 Response 只影响当前浏览器看到的内容：

```text
不会修改服务器真实文件
```

---

# 28. 取消 Breakpoint

完成实验后关闭 Break。

否则浏览器会一直等待。

检查工具栏：

```text
Break button 不再启用
```

---

# 29. 使用 Requester / Manual Request Editor

在 History 中右键一条请求：

```text
Open/Resend with Request Editor
```

或：

```text
Open in Requester
```

不同 Add-on 和版本的名称可能不同。

你可以：

1. 修改 URL
2. 修改 Parameter
3. 添加 Header
4. 修改 Body
5. 点击 Send
6. 对比 Response

这相当于 Burp 的 Repeater。

---

# 30. Requester 实验

原始请求：

```http
GET /item?id=1 HTTP/1.1
Host: 10.10.20.20:8087
```

分别发送：

```text
id=1
id=2
id=42
id=999
```

记录：

| ID | Status | Length | Result |
|---:|---:|---:|---|
| 1 |  |  |  |
| 2 |  |  |  |
| 42 |  |  |  |
| 999 |  |  |  |

---

# 31. Header 实验

在 Requester 中请求：

```http
GET /cors HTTP/1.1
Host: 10.10.20.20:8087
Origin: https://training.invalid
```

查看响应：

```text
Access-Control-Allow-Origin
Access-Control-Allow-Credentials
```

本实验故意回显 Origin。

这只是学习 CORS 风险线索，不要用于未授权站点。


---

# 32. Traditional Spider

Traditional Spider 主要解析：

- HTML Link
- Form
- Redirect
- robots.txt
- 静态页面引用

在 Sites Tree 右键 Scope 内目标：

```text
Attack
→ Spider
```

确认：

```text
Starting Point：http://10.10.20.20:8087/
Context：CyberLab-ZAP
Recurse：Enabled
```

开始后查看：

```text
Spider Tab
```

---

# 33. Spider 的安全控制

本实验建议：

```text
Max Depth：3
Max Children：较低
Threads：1–2
Scope：只限 Context
```

不要使用大型无限爬取。

Spider 可能：

- 提交表单
- 跟随 Redirect
- 增加请求数量
- 访问所有发现链接

因此先确认实验服务器没有破坏性操作。

---

# 34. Client Spider

ZAP 的 Client Spider 用于探索现代 Web 应用。

它通过浏览器和 ZAP Browser Extension 访问 DOM，因此可以发现：

- JavaScript 生成的链接
- Fragment URL
- DOM 中存在但尚未访问的 URL
- 浏览器端 Storage
- 单页应用结构

启动方法可能包括：

```text
Tools
→ Client Spider
```

或：

```text
右键 Context
→ Attack
→ Client Spider
```

---

# 35. Client Spider 与 AJAX Spider

当前 ZAP 官方建议：

```text
现代 Web 应用优先使用 Client Spider
```

Client Spider 在已知情况下通常优于旧的 AJAX Spider。

比较：

| Traditional Spider | Client Spider | AJAX Spider |
|---|---|---|
| 解析普通 HTTP/HTML | 使用真实浏览器和 DOM | 旧式浏览器爬虫 |
| 速度较快 | 更适合现代应用 | 不再是现代应用首选 |
| 适合静态网站 | 适合 SPA 和动态 DOM | 仅在特定兼容场景使用 |
| 不执行完整前端逻辑 | 能观察浏览器事件 | 能执行 JS，但能力较旧 |

本课目标较简单：

```text
先用 Traditional Spider
再运行一次 Client Spider 比较
```

---

# 36. Client Map

安装 Client Side Integration Add-on 后，ZAP 可能显示：

```text
Client Map
```

Client Map 与 Sites Tree 不完全相同。

Sites Tree：

```text
服务器端 URL 结构
```

Client Map：

```text
浏览器和 DOM 观察到的客户端结构
```

现代 SPA 中，Client Map 可能包含：

- URL Fragment
- DOM 节点
- Local Storage
- Session Storage
- 尚未访问的 DOM URL

---

# 37. Passive Scan 等待

Spider 或 Client Spider 完成后，Passive Scan 可能仍在处理消息。

查看底部状态：

```text
Passive Scan Queue
```

报告前应等待：

```text
Queue = 0
```

否则部分 Alert 可能还没有生成。

---

# 38. Fuzzer 基础

在 History 或 Requester 中选择参数值：

```text
id=1
```

右键：

```text
Fuzz
```

选择 Payload：

```text
1
2
42
999
```

本课只使用小型 Payload 列表。

---

# 39. 小型 Fuzzer 实验

目标：

```text
/item?id=FUZZ
```

Payload：

```text
1
2
42
999
```

限制：

```text
Threads：1
Delay：适当
Payload 数量：4
```

记录：

| Payload | Status | Response Length | Observation |
|---|---:|---:|---|
| 1 |  |  |  |
| 2 |  |  |  |
| 42 |  |  |  |
| 999 |  |  |  |

---

# 40. Fuzzer 不等于密码爆破

本课 Fuzzer 仅用于：

```text
理解输入替换
比较响应
验证参数行为
```

不要用于：

- 真实密码猜测
- Token 爆破
- 登录尝试
- 账号枚举
- 认证绕过
- 大型 Payload

---

# 41. Encode / Decode / Hash

ZAP 提供：

```text
Tools
→ Encode / Decode / Hash
```

常见功能：

- URL Encode
- URL Decode
- Base64 Encode
- Base64 Decode
- HTML Encode
- HTML Decode
- Hex
- Hash

示例：

原文：

```text
Cyber Lab & ZAP
```

URL 编码：

```text
Cyber%20Lab%20%26%20ZAP
```

---

# 42. Manual Request Editor 实验

发送 JSON：

```http
POST /api/echo HTTP/1.1
Host: 10.10.20.20:8087
Content-Type: application/json

{"lesson":"zap","student":"cyberlab"}
```

预期：

```json
{
  "valid_json": true,
  "received": {
    "lesson": "zap",
    "student": "cyberlab"
  }
}
```

把 JSON 改成无效格式，观察：

```text
400
```

---

# 43. Active Scan 前检查清单

开始 Active Scan 前，必须全部满足：

```text
目标是自己的 Ubuntu 10.10.20.20
端口是 8087
Context 是 CyberLab-ZAP
目标已经 Include in Scope
ZAP 使用 Protected Mode
没有其他网站在 Scope
Windows 和 Wazuh 不在 Scope
Ubuntu 已做快照或脚本已备份
实验服务器没有真实数据
```

任何一项不满足：

```text
不要开始 Active Scan
```

---

# 44. Active Scan 是攻击

ZAP 官方明确指出：

```text
Active Scan 会使用已知攻击方式测试目标
```

它可能：

- 发送大量特殊参数
- 重复请求输入点
- 触发应用错误
- 修改服务器状态
- 影响性能
- 产生大量日志

本课只扫描：

```text
http://10.10.20.20:8087/
```

---

# 45. 创建低强度 Scan Policy

进入：

```text
Analyse
→ Scan Policy Manager
```

或 Active Scan 对话框中的 Policy 设置。

创建：

```text
Name：CyberLab-Low-Impact
```

建议：

```text
Attack Strength：Low
Alert Threshold：Medium
Threads：1
Delay：可适当增加
```

只启用与本实验相关的少量规则，例如：

```text
Cross Site Scripting
CORS
Server Header / Header checks
Parameter handling
```

不同 ZAP Add-on 的规则名称会变化。

---

# 46. 不启用的高风险规则

初学阶段避免：

- 高强度注入规则
- 大量文件包含测试
- 文件上传测试
- 远程命令相关规则
- 大规模 DOM XSS
- 可能导致资源耗尽的规则
- 不理解作用的 Alpha Rules

原则：

```text
先少量规则
理解请求
再逐步扩大
```

---

# 47. 启动受控 Active Scan

在 Sites Tree 右键：

```text
http://10.10.20.20:8087
```

选择：

```text
Attack
→ Active Scan
```

确认：

```text
Context：CyberLab-ZAP
Policy：CyberLab-Low-Impact
Scope：只限目标节点
Recurse：Enabled
User：None
```

开始扫描。

---

# 48. 观察 Active Scan

查看：

```text
Active Scan Tab
```

常见信息：

```text
Progress
Requests
Rule
Host
Elapsed Time
Alerts
```

如果：

- 请求太快
- Ubuntu 响应变慢
- CPU 异常
- Scope 错误
- 扫描其他目标

立即点击：

```text
Stop
```

---

# 49. 识别 ZAP 流量 Header

ZAP 提供脚本方式给流量添加类似：

```text
X-ZAP-Initiator
```

实验服务器会把该 Header 写入日志。

不是每条默认请求都一定包含这个 Header。

在 Access Log 中查看：

```bash
grep 'X-ZAP-Initiator' \
  ~/CyberLab/ZAP/logs/access.log
```

也可以自定义：

```text
X-Lab-Tool: OWASP-ZAP
```

帮助实验识别。

---

# 50. Active Scan 可能发现的内容

本实验可能产生：

- Reflected Cross Site Scripting 线索
- CORS 配置问题
- Cookie 安全属性问题
- 缺失安全 Header
- Open Redirect 线索
- Information Disclosure
- Parameter Manipulation
- Cache-Control 问题

具体结果取决于：

```text
ZAP 版本
Add-ons
Scan Policy
Attack Strength
Alert Threshold
```

---

# 51. Alert 人工验证

对每个 Alert：

1. 阅读 `Description`
2. 查看 `URL`
3. 查看 `Parameter`
4. 查看 `Attack`
5. 查看 `Evidence`
6. 查看原始 Request
7. 查看原始 Response
8. 使用 Requester 重发
9. 使用 curl 复查
10. 判断 True Positive 或 False Positive

---

# 52. 反射输入验证

ZAP 可能在：

```text
/search?q=
```

发现反射输入线索。

安全地使用普通标记验证：

```bash
curl -i \
  'http://10.10.20.20:8087/search?q=ZAP_TEST_MARKER'
```

查看响应中是否出现：

```text
ZAP_TEST_MARKER
```

反射存在不自动证明完整 XSS。

要判断 XSS，还要考虑：

- HTML Context
- Attribute Context
- JavaScript Context
- 编码
- CSP
- 浏览器解析

---

# 53. CORS 人工验证

```bash
curl -i \
  -H 'Origin: https://training.invalid' \
  http://10.10.20.20:8087/cors
```

检查：

```text
Access-Control-Allow-Origin
Access-Control-Allow-Credentials
```

若服务器反射任意 Origin 并允许 Credentials：

```text
可能构成危险 CORS 配置
```

实际风险取决于：

- 是否存在敏感认证 Cookie
- Cookie SameSite
- 端点是否返回敏感数据
- 浏览器凭据行为

---

# 54. Cookie 人工验证

```bash
curl -i \
  http://10.10.20.20:8087/cookie
```

检查：

```text
Secure
HttpOnly
SameSite
```

实验 Cookie 没有 `Secure`。

由于本课使用 HTTP：

```text
Secure Cookie 不会通过普通 HTTP 发送
```

真实生产应用应优先使用 HTTPS。

---

# 55. Redirect 人工验证

```bash
curl -i \
  'http://10.10.20.20:8087/redirect?next=/final'
```

再测试一个完整 URL：

```bash
curl -i \
  'http://10.10.20.20:8087/redirect?next=https://training.invalid/'
```

仅查看：

```text
Location Header
```

不要访问未知外部目标。

---

# 56. False Positive

Alert 可以在 ZAP 中标记为：

```text
False Positive
```

但标记前必须保存证据。

推荐记录：

```text
Alert：
Reason：
Manual Request：
Manual Response：
Server-side Evidence：
Conclusion：
```

不要因为不喜欢结果就直接标记 False Positive。

---

# 57. Retest

修复后，可使用 ZAP 的 Retest 功能或手动重发原请求。

流程：

```text
保存原始 Alert
→ 修复服务器
→ 重发相同请求
→ 比较 Response
→ 运行 Passive/Active Retest
→ 更新状态
```

---

# 58. Report Generation

进入：

```text
Report
→ Generate Report
```

常见设置：

```text
Title：CyberLab OWASP ZAP Report
Report Directory：~/CyberLab/ZAP/reports
Context：CyberLab-ZAP
Site：http://10.10.20.20:8087
Risk：High / Medium / Low / Informational
Confidence：按需选择
```

---

# 59. 报告模板

ZAP 当前 Report Generation Add-on 支持多种模板，例如：

- Modern HTML
- Risk and Confidence HTML
- Traditional HTML
- Traditional HTML with Requests and Responses
- Traditional JSON
- Traditional JSON with Requests and Responses
- SARIF JSON
- Traditional Markdown
- Traditional PDF
- Traditional XML
- Traditional XML with Requests and Responses

学习阶段推荐：

```text
HTML：便于阅读
Markdown：便于学习笔记
JSON：便于自动处理
```

---

# 60. 报告中的敏感信息

带 Requests and Responses 的报告可能包含：

- Cookie
- Authorization
- Token
- POST Body
- 内部 IP
- 测试 Payload
- 页面内容

不要公开上传：

```text
Traditional HTML Plus
Traditional JSON Plus
Traditional XML Plus
```

除非已经清理敏感信息。

---

# 61. 生成 Markdown 报告

选择模板：

```text
Traditional Markdown Report
```

保存：

```text
~/CyberLab/ZAP/reports/zap-cyberlab.md
```

打开：

```bash
less ~/CyberLab/ZAP/reports/zap-cyberlab.md
```

---

# 62. 保存 Session

进入：

```text
File
→ Persist Session
```

保存到：

```text
~/CyberLab/ZAP/sessions/cyberlab-zap
```

Session 可能包含：

- History
- Sites Tree
- Context
- Scope
- Alerts
- 请求和响应
- Cookie
- 认证数据

必须像敏感证据一样管理。

---

# 63. Add-ons

进入：

```text
Manage Add-ons
```

常见有用 Add-ons：

- Passive Scanner
- Active Scan Rules
- Client Side Integration
- Client Spider
- Report Generation
- Requester
- Fuzzer
- Selenium
- Retire.js
- OpenAPI Support
- Automation Framework

你的 CyberLab 通常没有互联网。

Add-on 更新可能需要临时给 Kali 或 ZAP-SIEM 单独提供联网能力。

不要为目标 VM 长期开启 Shared Network。

---

# 64. Safe Mode 学习

切换：

```text
Safe Mode
```

尝试对目标右键查看功能。

在 Safe Mode 中，ZAP 会禁止潜在危险操作，例如：

- Spider
- Active Scan
- Fuzz
- Break
- Resend

适合：

```text
只浏览和 Passive Scan
```

---

# 65. Protected Mode 学习

Protected Mode 中：

```text
Scope 内：可执行受控危险操作
Scope 外：禁止危险操作
```

这是本课和以后实验的推荐模式。

---

# 66. Standard 和 Attack Mode

## Standard

```text
不限制危险操作的目标
```

不推荐初学使用。

## Attack

```text
新发现的 Scope 节点可能自动 Active Scan
```

本课禁止使用。

---

# 67. Passive Baseline 工作流

最安全的 ZAP 工作流：

```text
1. Protected Mode
2. Context + Scope
3. Manual Explore
4. Traditional Spider
5. Client Spider
6. 等待 Passive Queue = 0
7. 阅读 Alerts
8. 人工验证
9. 生成报告
```

这个流程不主动发送漏洞攻击 Payload，除了爬虫本身产生的访问请求。

---

# 68. Automation Framework

ZAP Automation Framework 使用 YAML 定义：

- Environment
- Context
- URLs
- Spider
- Passive Scan Wait
- Active Scan
- Report
- Exit Status

命令行常见：

```bash
zaproxy -cmd -autorun plan.yaml
```

检查 Plan：

```bash
zaproxy -cmd -autocheck plan.yaml
```

生成最小模板：

```bash
zaproxy -cmd \
  -autogenmin ~/CyberLab/ZAP/automation/generated-min.yaml
```

---

# 69. 创建 Passive Automation Plan

Kali：

```bash
nano ~/CyberLab/ZAP/automation/passive-plan.yaml
```

粘贴：

```yaml
env:
  contexts:
    - name: CyberLab-ZAP
      urls:
        - http://10.10.20.20:8087
      includePaths:
        - http://10\.10\.20\.20:8087.*
      excludePaths: []
      authentication:
        method: manual
        parameters: {}
        verification:
          method: response
          loggedInRegex: ""
          loggedOutRegex: ""

jobs:
  - type: spider
    parameters:
      context: CyberLab-ZAP
      url: http://10.10.20.20:8087/
      maxDuration: 1
      maxDepth: 3

  - type: passiveScan-wait
    parameters:
      maxDuration: 2

  - type: report
    parameters:
      template: traditional-md
      reportDir: /home/kali/CyberLab/ZAP/reports
      reportFile: automation-passive-report.md
      reportTitle: CyberLab ZAP Passive Report
      reportDescription: Authorized passive ZAP laboratory scan.
      displayReport: false
    risks:
      - high
      - medium
      - low
      - info
    confidences:
      - high
      - medium
      - low
      - falsepositive
```

说明：

```text
这是 Passive Baseline
没有 activeScan Job
```

不同 ZAP/Add-on 版本的 YAML 参数可能变化。

先运行：

```bash
zaproxy -cmd \
  -autocheck ~/CyberLab/ZAP/automation/passive-plan.yaml
```

---

# 70. 执行 Passive Plan

```bash
zaproxy \
  -cmd \
  -silent \
  -autorun ~/CyberLab/ZAP/automation/passive-plan.yaml
```

检查报告：

```bash
ls -lh ~/CyberLab/ZAP/reports/
```

```bash
less \
  ~/CyberLab/ZAP/reports/automation-passive-report.md
```

---

# 71. Automation Exit Code

执行后：

```bash
echo $?
```

Automation Framework 与 `-cmd -autorun` 组合时，常见默认：

```text
0 = 没有错误或警告
1 = 一个或多个 Error
2 = 没有 Error，但有 Warning
```

具体结果也可由 `exitStatus` Job 控制。

---

# 72. 不在第一份 Plan 中加入 Active Scan

不要立即添加：

```yaml
- type: activeScan
```

先确保：

- Context 正确
- Include Regex 正确
- Spider 只访问实验目标
- Passive 报告正常
- Automation 没有访问其他 IP
- Access Log 可核对

Active Automation 留到掌握 GUI Active Scan 后。


---

# 73. curl 通过 ZAP Proxy

先确保 ZAP 正在运行，Proxy Listener：

```text
127.0.0.1:8080
```

Kali：

```bash
curl \
  -x http://127.0.0.1:8080 \
  -H 'X-Lab-Tool: curl-through-zap' \
  http://10.10.20.20:8087/headers
```

当前实验没有 `/headers`，会返回 404，但请求仍会出现在 ZAP History。

使用有效路径：

```bash
curl \
  -x http://127.0.0.1:8080 \
  -H 'X-Lab-Tool: curl-through-zap' \
  'http://10.10.20.20:8087/item?id=1'
```

---

# 74. curl 与 HTTPS Proxy

外部 HTTPS 目标经过 ZAP 时，curl 需要信任 ZAP CA。

实验中不要直接使用：

```bash
-k
```

作为长期解决方案。

可从 ZAP 导出公开 Root CA，然后：

```bash
curl \
  --cacert ~/CyberLab/ZAP/notes/zap-root-ca.pem \
  -x http://127.0.0.1:8080 \
  https://AUTHORIZED-TARGET/
```

本课的 Ubuntu 8087 使用 HTTP，因此不需要 CA。

---

# 75. 导出 ZAP Root CA

GUI 常见路径：

```text
Tools
→ Options
→ Network
→ Server Certificates
```

导出：

```text
ZAP Root CA Public Certificate
```

也可查看本机 CLI 是否支持：

```bash
zaproxy -h |
grep -E 'certpubdump|certfulldump'
```

只需浏览器信任时，应导出：

```text
Public Root CA
```

不要分享：

```text
包含私钥的完整 CA 文件
```

---

# 76. 外部 Firefox 配置

使用 ZAP 内置浏览器最简单。

需要外部 Firefox 时：

```text
HTTP Proxy：127.0.0.1
Port：8080
HTTPS Proxy：127.0.0.1
Port：8080
```

并导入 ZAP Root CA。

完成实验后：

```text
移除 Proxy
删除不再需要的 ZAP CA
```

不要把个人日常浏览器长期设置为 ZAP Proxy。

---

# 77. 两段网络连接

当浏览器和 ZAP 都在 Kali 上时：

```text
浏览器 → ZAP
```

使用：

```text
Loopback：127.0.0.1:8080
```

然后：

```text
ZAP → Ubuntu
```

使用：

```text
eth0
10.10.20.10 → 10.10.20.20:8087
```

所以抓包时要明确想观察哪一段。

---

# 78. 捕获浏览器到 ZAP

Kali：

```bash
sudo tcpdump \
  -i lo \
  -nn \
  -A \
  tcp port 8080
```

你可以看到浏览器与本地 ZAP Proxy 的 HTTP 消息。

注意：

```text
这是本机 Loopback 流量
```

---

# 79. 捕获 ZAP 到 Ubuntu

Kali：

```bash
sudo tcpdump \
  -i eth0 \
  -nn \
  -A \
  'host 10.10.20.20 and tcp port 8087'
```

这会显示 ZAP 实际发送给 Ubuntu 的请求。

---

# 80. 同时保存两个 PCAP

终端一：

```bash
sudo tcpdump \
  -i lo \
  -nn \
  -s 0 \
  -w ~/CyberLab/ZAP/captures/browser-to-zap.pcap \
  tcp port 8080
```

终端二：

```bash
sudo tcpdump \
  -i eth0 \
  -nn \
  -s 0 \
  -w ~/CyberLab/ZAP/captures/zap-to-ubuntu.pcap \
  'host 10.10.20.20 and tcp port 8087'
```

然后使用 ZAP Browser 浏览实验目标。

完成后分别按：

```text
Ctrl + C
```

---

# 81. Wireshark 分析

打开 ZAP 到 Ubuntu：

```bash
wireshark \
  ~/CyberLab/ZAP/captures/zap-to-ubuntu.pcap
```

过滤：

```text
http
```

只看请求：

```text
http.request
```

只看 Active Scan 期间的 User-Agent 或 Header：

```text
http.user_agent contains "ZAP"
```

```text
http.request.line contains "X-ZAP"
```

实际 User-Agent 和 Header 取决于配置。

---

# 82. Wireshark 统计

打开：

```text
Statistics
→ HTTP
→ Requests
```

以及：

```text
Statistics
→ Conversations
```

记录：

| 阶段 | HTTP Requests | TCP Conversations | Duration |
|---|---:|---:|---:|
| Manual Explore |  |  |  |
| Traditional Spider |  |  |  |
| Client Spider |  |  |  |
| Active Scan |  |  |  |

你应该看到：

```text
Active Scan 的请求量通常明显更多
```

---

# 83. Ubuntu Access Log

查看：

```bash
less ~/CyberLab/ZAP/logs/access.log
```

只看 Kali：

```bash
grep '10.10.20.10' \
  ~/CyberLab/ZAP/logs/access.log
```

统计请求：

```bash
grep -c '"GET ' \
  ~/CyberLab/ZAP/logs/access.log
```

统计方法：

```bash
grep -Eo '"(GET|HEAD|POST|OPTIONS|PUT|DELETE) ' \
  ~/CyberLab/ZAP/logs/access.log |
sort |
uniq -c
```

---

# 84. 区分 Manual、Spider 与 Active Scan

可以在不同阶段添加日志标记。

Ubuntu：

```bash
logger -t zap-lab \
  "Starting ZAP manual explore"
```

Manual 完成后：

```bash
logger -t zap-lab \
  "Completed ZAP manual explore"
```

Spider 前：

```bash
logger -t zap-lab \
  "Starting ZAP traditional spider"
```

Active Scan 前：

```bash
logger -t zap-lab \
  "Starting authorized ZAP active scan on 10.10.20.20:8087"
```

Active Scan 后：

```bash
logger -t zap-lab \
  "Completed authorized ZAP active scan"
```

---

# 85. Wazuh 日志链路

Ubuntu 本地查看：

```bash
journalctl \
  -t zap-lab \
  --since "30 minutes ago"
```

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
zap-lab
```

---

# 86. ZAP 是否一定触发 Wazuh 告警？

不一定。

Wazuh 要自动识别 ZAP 扫描，通常需要：

- Web Access Log
- Wazuh 收集该日志
- User-Agent 检测
- 请求速率规则
- 特殊 Payload 规则
- WAF
- IDS/IPS
- Suricata
- Zeek

本课成功标准：

```text
ZAP History 有请求
Ubuntu Access Log 有请求
PCAP 有流量
Wazuh 能看到 zap-lab 标记事件
```

---

# 87. Quick Start Automated Scan

ZAP Quick Start 中可能有：

```text
Automated Scan
```

输入 URL 后点击：

```text
Attack
```

它可能组合：

- Spider
- Passive Scan
- Active Scan

因为它会执行主动攻击测试：

```text
本课不把它作为第一次扫描方式
```

先掌握：

```text
Scope
Protected Mode
Manual Explore
Passive Scan
低强度 Active Scan
```

再使用 Automated Scan。

---

# 88. Quick CLI Scan

当前 Kali ZAP CLI 可能提供：

```text
-quickurl
-quickout
-quickprogress
```

查看：

```bash
zaproxy -h |
grep -E 'quickurl|quickout|quickprogress'
```

仅对实验目标：

```bash
zaproxy \
  -cmd \
  -quickurl http://10.10.20.20:8087/ \
  -quickout ~/CyberLab/ZAP/reports/quick-scan.html
```

Quick Scan 可能执行主动测试。

只能用于：

```text
10.10.20.20:8087
```

---

# 89. Reconnaissance CLI

某些当前版本支持：

```text
-zapit
```

查看：

```bash
zaproxy -h |
grep zapit
```

对实验目标：

```bash
zaproxy \
  -cmd \
  -zapit http://10.10.20.20:8087/
```

实际行为和输出以本机版本帮助为准。

---

# 90. Session 与报告备份

推荐文件结构：

```text
ZAP/
├── sessions/
├── reports/
├── captures/
├── automation/
├── notes/
└── screenshots/
```

文件命名：

```text
2026-08-06-zap-passive.md
2026-08-06-zap-active-low.html
2026-08-06-zap-to-ubuntu.pcap
2026-08-06-zap-session.session
```

---

# 91. 证据保存清单

每个 Alert 保存：

```text
Alert Name
Risk
Confidence
URL
Parameter
Attack
Evidence
Request
Response
Manual Verification
Server-side Evidence
Conclusion
Recommendation
Retest
```

---

# 92. Finding 验证模板

```text
Finding：
Tool：OWASP ZAP
Risk：
Confidence：

URL：
Parameter：
Attack：
Evidence：

Manual Request：
Manual Response：

Access Log Evidence：
PCAP Evidence：
Server-side Evidence：

Classification：
- True Positive
- False Positive
- Informational
- Needs More Context

Recommendation：
Retest Result：
```

---

# 93. 示例：Cookie Without Secure Flag

```text
Finding:
Cookie without Secure flag

URL:
http://10.10.20.20:8087/cookie

Evidence:
Set-Cookie: lab_cookie=value; Path=/

Manual Verification:
curl -i http://10.10.20.20:8087/cookie

Conclusion:
True Positive

Context:
The service uses HTTP. A Secure cookie requires HTTPS.

Recommendation:
Deploy HTTPS and add Secure, HttpOnly, and an appropriate SameSite value.
```

---

# 94. 示例：Missing Anti-clickjacking Header

```text
Finding:
Missing anti-clickjacking protection

Evidence:
No X-Frame-Options and no CSP frame-ancestors.

Verification:
curl -s -D - URL -o /dev/null

Conclusion:
Header absence is confirmed.

Risk:
Depends on whether sensitive actions can be framed and tricked through UI redressing.

Recommendation:
Use CSP frame-ancestors and/or X-Frame-Options as appropriate.
```

---

# 95. 示例：Reflected Input

```text
Finding:
User input is reflected into HTML.

URL:
/search?q=ZAP_TEST_MARKER

Evidence:
The response contains the marker without HTML escaping.

Conclusion:
Reflection is confirmed.
Full XSS exploitability depends on browser parsing context and controls.

Recommendation:
Apply context-aware output encoding and a restrictive CSP.
```

---

# 96. 修复练习

复制实验脚本：

```bash
cp \
  ~/CyberLab/ZAP/server/zap_lab.py \
  ~/CyberLab/ZAP/server/zap_lab_hardened.py
```

建议修复：

- 对 `/search` 使用 `html.escape`
- 添加 `Content-Security-Policy`
- 添加 `X-Content-Type-Options: nosniff`
- 添加 `Referrer-Policy`
- Cookie 使用适当属性
- CORS 使用明确可信 Origin
- 禁止任意 Redirect
- 限制可接受参数

将 Hardened 版本端口改为：

```text
8088
```

---

# 97. 修复后复测

原始服务：

```text
http://10.10.20.20:8087/
```

Hardened 服务：

```text
http://10.10.20.20:8088/
```

分别：

1. Manual Explore
2. Passive Scan
3. 低强度 Active Scan
4. 生成报告
5. 比较 Alert

记录：

| Alert | 8087 | 8088 | Result |
|---|---|---|---|
| Reflected Input/XSS |  |  |  |
| CORS |  |  |  |
| Cookie |  |  |  |
| Security Headers |  |  |  |
| Redirect |  |  |  |

---

# 98. 完整实验一：Manual Explore + Passive Scan

Ubuntu：

```bash
python3 ~/CyberLab/ZAP/server/zap_lab.py \
  2>&1 |
tee ~/CyberLab/ZAP/logs/access.log
```

Kali：

1. 启动 `zaproxy`
2. 切换 `Protected Mode`
3. 创建 `CyberLab-ZAP` Context
4. 把 `http://10.10.20.20:8087` 加入 Scope
5. Quick Start → Manual Explore
6. 访问所有首页链接
7. 等待 Passive Queue = 0
8. 查看 Alerts
9. 生成 Markdown 报告

---

# 99. 完整实验二：Breakpoint + Requester

1. Break `/item?id=1`
2. 改为 `id=42`
3. Forward
4. 从 History 打开 Requester
5. 测试 `1、2、42、999`
6. 添加自定义 Header：

```text
X-Lab-Request: ZAP-Requester
```

7. 保存结果表

---

# 100. 完整实验三：Spider 比较

依次运行：

```text
Traditional Spider
Client Spider
```

限制：

```text
Scope：CyberLab-ZAP
Depth：3
Threads：1–2
```

比较：

| 项目 | Traditional | Client |
|---|---:|---:|
| URLs |  |  |
| Requests |  |  |
| Duration |  |  |
| Client-only Nodes |  |  |

---

# 101. 完整实验四：受控 Active Scan

检查：

```text
Protected Mode
只有 8087 在 Scope
Low Attack Strength
Medium Alert Threshold
1 Thread
有限 Scan Rules
```

开始 Active Scan。

完成后：

- 等待 Passive Queue = 0
- 查看 Alerts
- 使用 curl 验证
- 生成 HTML 报告
- 保存 Session
- 停止实验服务器

---

# 102. 完整实验五：Automation Passive Baseline

1. 创建 `passive-plan.yaml`
2. 运行 `-autocheck`
3. 使用 `-cmd -silent -autorun`
4. 检查 Exit Code
5. 查看 Markdown 报告
6. 对比 GUI Passive Scan

---

# 103. 完整实验六：流量与 Wazuh

1. Ubuntu 写入 `zap-lab` 开始标记
2. Kali 同时抓 `lo:8080` 与 `eth0:8087`
3. ZAP 运行 Manual Explore 和低强度 Active Scan
4. 停止 PCAP
5. Ubuntu 写入完成标记
6. Wireshark 统计请求
7. 查看 Access Log
8. Wazuh 搜索 `zap-lab`

---

# 104. 实验报告模板

## Lab Information

```text
Lab Name：OWASP ZAP Web Application Assessment
Date：
Student：
Scanner：Kali-CyberLab 10.10.20.10
Target：Ubuntu-Target 10.10.20.20
URL：http://10.10.20.20:8087/
Authorization：Personal CyberLab
ZAP Version：
Installed Add-ons：
Mode：Protected
Context：CyberLab-ZAP
```

## Objective

```text
使用 ZAP Proxy、Passive Scan、Spider、Requester、Fuzzer 和受控 Active Scan
分析实验 Web 应用，并使用 curl、PCAP、Access Log 和 Wazuh 验证。
```

## Scope

```text
Included：
http://10.10.20.20:8087.*

Excluded：
All other URLs
```

## Test Stages

| Stage | Requests | Duration | Alerts |
|---|---:|---:|---:|
| Manual Explore |  |  |  |
| Traditional Spider |  |  |  |
| Client Spider |  |  |  |
| Active Scan |  |  |  |

## Findings

| ID | Alert | Risk | Confidence | Verification | Status |
|---:|---|---|---|---|---|
| 1 |  |  |  |  |  |
| 2 |  |  |  |  |  |

## Evidence

```text
ZAP Session：
HTML Report：
Markdown Report：
JSON Report：
Request/Response：
curl：
Access Log：
PCAP：
Wireshark Screenshot：
Wazuh Event：
```

## False Positive Analysis

```text
Alert：
Reason：
Manual Evidence：
Server Evidence：
Decision：
```

## Recommendations

```text
使用上下文相关的输出编码。
限制 CORS Origin。
使用 HTTPS 和安全 Cookie。
配置 CSP 和其他适当安全 Header。
限制 Redirect 目标。
对敏感操作实施认证、授权和 CSRF 防护。
```

## Retest

```text
Fix Applied：
Retest Date：
Alert Removed：
Alert Remaining：
New Alert：
```

---

# 105. 常见错误与解决方法

## zaproxy command not found

```bash
sudo apt update
sudo apt install zaproxy
```

## Java 错误

```bash
java -version
```

```bash
apt policy default-jre
```

必要时：

```bash
sudo apt install default-jre
```

## ZAP 无法启动

查看支持信息：

```bash
zaproxy -suppinfo
```

检查日志目录：

```bash
find ~/.ZAP* -name 'zap.log' -type f 2>/dev/null
```

## Listener 启动失败

```bash
sudo ss -lntp | grep ':8080'
```

更换为：

```text
127.0.0.1:8081
```

并同步修改浏览器 Proxy。

## 浏览器一直加载

检查：

- 是否设置 Break
- 请求是否停在 Break Tab
- 是否需要 Continue
- Proxy Listener 是否 Running
- Ubuntu 8087 是否在线

## History 没有请求

检查：

- 使用的是否是 ZAP Browser
- 外部浏览器 Proxy 是否正确
- History Filter 是否只显示 Scope
- URL 是否被 Global Exclude
- ZAP 是否处于 Safe Mode

## Active Scan 按钮不可用

可能原因：

- Safe Mode
- Protected Mode 下目标不在 Scope
- 没有 Active Scan Rules Add-on
- 选中的不是可扫描节点

## Client Spider 不存在

进入：

```text
Manage Add-ons
```

检查：

```text
Client Side Integration
Selenium
Browser Support
```

## 报告选项不存在

检查：

```text
Report Generation Add-on
```

## Automation YAML 报错

先：

```bash
zaproxy -cmd \
  -autocheck ~/CyberLab/ZAP/automation/passive-plan.yaml
```

再使用：

```bash
zaproxy -cmd \
  -autogenmin ~/CyberLab/ZAP/automation/template.yaml
```

对比当前版本生成的字段。

## Passive Alert 很少

原因可能：

- 没访问足够页面
- Passive Queue 未完成
- Add-ons 未安装
- History Filter 隐藏
- 目标响应很简单

## Active Scan 没发现 XSS

可能原因：

- 对应 Scan Rule 没安装或没启用
- Policy Threshold 过高
- Parameter 不在扫描输入点
- Active Scan 没递归到 `/search`
- ZAP 版本规则不同

使用普通 Marker 手工确认反射，不要盲目增加强度。

---

# 106. 性能控制

ZAP、浏览器和 Selenium 可能使用较多内存。

减少负载：

- 关闭 Windows Target
- Client Spider 只运行一次
- Active Scan 使用 1 Thread
- Attack Strength 设 Low
- Scope 只包含 8087
- 不使用 Attack Mode
- 完成后关闭浏览器
- 删除不需要的 Session

---

# 107. 敏感文件管理

以下文件可能包含敏感信息：

- Session
- Report Plus
- Request/Response
- PCAP
- Access Log
- CA 私钥
- Cookie
- Authorization Header
- Automation Context Credentials

不要公开上传。

检查：

```bash
grep -RniE \
  'authorization|cookie|token|password|session' \
  ~/CyberLab/ZAP
```

---

# 108. 本课练习

1. 检查 ZAP 版本和帮助。
2. 安装或启动 ZAP。
3. 创建实验目录。
4. 启动 Ubuntu 8087 Server。
5. 使用 curl 验证。
6. 检查 ZAP Listener。
7. 使用 Manual Explore。
8. 查看 History。
9. 查看 Sites Tree。
10. 查看 Passive Alerts。
11. 创建 Context。
12. 加入 Scope。
13. 切换 Protected Mode。
14. 设置 Request Break。
15. 修改 `id=1` 为 `id=42`。
16. 设置 Response Break。
17. 使用 Requester 重发四个 ID。
18. 使用 Encode/Decode。
19. 运行 Traditional Spider。
20. 运行 Client Spider。
21. 比较 Spider 结果。
22. 使用小型 Fuzzer。
23. 等待 Passive Queue = 0。
24. 创建 Low Impact Scan Policy。
25. 对 8087 运行受控 Active Scan。
26. 验证反射输入。
27. 验证 CORS。
28. 验证 Cookie。
29. 验证 Redirect。
30. 标记一个 False Positive 或确认 True Positive。
31. 生成 HTML 报告。
32. 生成 Markdown 报告。
33. 保存 Session。
34. 创建 Passive Automation Plan。
35. 使用 `-autocheck`。
36. 执行 `-autorun`。
37. 查看 Exit Code。
38. curl 通过 ZAP Proxy。
39. 捕获 Loopback PCAP。
40. 捕获 eth0 PCAP。
41. 使用 Wireshark 统计。
42. 查看 Ubuntu Access Log。
43. 在 Wazuh 搜索 `zap-lab`。
44. 创建 Hardened Server。
45. 修复后复测。
46. 完成实验报告。

---

# 109. 复习题

1. ZAP 的主要用途是什么？
2. Passive Scan 与 Active Scan 有什么区别？
3. 为什么 Active Scan 属于攻击？
4. Safe Mode 做什么？
5. Protected Mode 做什么？
6. 为什么本课不使用 Attack Mode？
7. Scope 是什么？
8. Context 是什么？
9. History 与 Sites Tree 有什么区别？
10. Breakpoint 有什么用途？
11. Requester 的用途是什么？
12. Traditional Spider 适合什么？
13. Client Spider 适合什么？
14. AJAX Spider 还是现代应用首选吗？
15. Passive Queue 为什么要等到 0？
16. Fuzzer 的用途是什么？
17. Scan Policy 控制什么？
18. Attack Strength 和 Alert Threshold 有何区别？
19. Alert Risk 与 Confidence 有何区别？
20. 为什么 Alert 必须人工验证？
21. ZAP 报告可以有哪些格式？
22. 为什么 Plus Report 更敏感？
23. Automation Framework 使用什么格式？
24. `-autorun` 做什么？
25. `-autocheck` 做什么？
26. `-silent` 做什么？
27. 为什么浏览器到 ZAP 使用 Loopback？
28. 为什么 ZAP 到 Ubuntu 使用 eth0？
29. Wazuh 为什么不一定自动告警？
30. 为什么 CA 私钥绝不能分享？
31. 为什么 Scope 只应包含 8087？
32. 修复后为什么要 Retest？
33. 为什么不能对未授权网站运行 Active Scan？

---

# 110. 参考答案

1. 代理、分析、爬取、主动扫描和自动化 Web 安全测试。
2. Passive 分析已有流量；Active 发送攻击形式请求。
3. 它使用已知攻击方式测试输入点。
4. 禁止潜在危险操作。
5. 只允许对 Scope 内目标执行危险操作。
6. 它会自动扫描新发现的 Scope 节点。
7. 被授权测试的 URL 集合。
8. 管理一个应用范围、认证、会话和用户的配置容器。
9. History 是消息列表；Sites Tree 是已知 URL 结构。
10. 在发送前或返回前暂停并修改消息。
11. 修改并重复发送请求。
12. 普通 HTML 和静态链接。
13. 现代 JavaScript 和 DOM 应用。
14. 不是，当前更推荐 Client Spider。
15. 确保所有 Passive 消息处理完成。
16. 用 Payload 替换输入并比较响应。
17. 控制 Active Scan 规则、强度、阈值和请求行为。
18. Strength 控制攻击强度；Threshold 控制报告敏感度。
19. Risk 表示潜在影响；Confidence 表示检测可信度。
20. 工具可能误报，也不了解业务上下文。
21. HTML、JSON、Markdown、PDF、XML、SARIF 等。
22. 可能包含完整 Request、Response、Cookie 和 Token。
23. YAML。
24. 执行 Automation Plan。
25. 检查 Plan 是否有效。
26. 防止 ZAP 发出非必要主动请求，例如更新检查。
27. 浏览器和 ZAP 都在 Kali 本机。
28. ZAP 要通过 Kali 的实验网卡访问 Ubuntu。
29. 需要 Web 日志收集和对应检测规则。
30. 拥有私钥的人可以伪造由该 CA 签发的证书。
31. 避免误扫 Wazuh、Windows 或其他服务。
32. 证明修复有效并检查回归。
33. 可能违法、修改数据或影响服务。

---

# 111. GUI 速查表

| 目标 | UI 路径 |
|---|---|
| Manual Explore | `Quick Start → Manual Explore` |
| History | `History` Tab |
| Sites Tree | `Sites` Tab |
| Alerts | `Alerts` Tab |
| Context | Sites 右键 → `Include in Context` |
| Scope | Sites 右键 → `Include in Scope` |
| Mode | 顶部 Mode 下拉 |
| Spider | Sites 右键 → `Attack → Spider` |
| Client Spider | `Tools / Attack → Client Spider` |
| Active Scan | Sites 右键 → `Attack → Active Scan` |
| Scan Policy | `Analyse → Scan Policy Manager` |
| Report | `Report → Generate Report` |
| Add-ons | `Manage Add-ons` |
| Options | `Tools → Options` |

---

# 112. CLI 速查表

## 版本

```bash
zaproxy -version
```

## 帮助

```bash
zaproxy -h
```

## 启动 GUI

```bash
zaproxy
```

## Silent 启动

```bash
zaproxy -silent
```

## 列出 Add-ons

```bash
zaproxy -cmd -addonlist
```

## 生成 Automation 模板

```bash
zaproxy -cmd \
  -autogenmin ~/CyberLab/ZAP/automation/template.yaml
```

## 检查 Plan

```bash
zaproxy -cmd \
  -autocheck ~/CyberLab/ZAP/automation/passive-plan.yaml
```

## 执行 Plan

```bash
zaproxy -cmd -silent \
  -autorun ~/CyberLab/ZAP/automation/passive-plan.yaml
```

## Quick Scan

```bash
zaproxy -cmd \
  -quickurl http://10.10.20.20:8087/ \
  -quickout ~/CyberLab/ZAP/reports/quick.html
```

---

# 113. 核心记忆口诀

```text
先 Scope
再 Protected Mode

Manual Explore 看流量
Passive Scan 不主动攻击
Spider 找链接
Client Spider 找现代 DOM
Requester 重发请求
Fuzzer 少量替换
Active Scan 是真实攻击

Alert 不是结论
curl 必须验证
Access Log 看服务器证据
PCAP 看网络证据
Wazuh 看端点日志

报告要清理
CA 私钥不分享
修复以后要 Retest
```

---

# 114. 完成标准

完成本课后，你应该拥有：

```text
Ubuntu 8087 ZAP 实验服务器
CyberLab-ZAP Context
正确 Scope
Protected Mode 截图
Manual Explore History
Sites Tree
Passive Alerts
Breakpoint 实验
Requester 实验
Traditional Spider 结果
Client Spider 结果
小型 Fuzzer 结果
低强度 Active Scan
HTML 报告
Markdown 报告
ZAP Session
Passive Automation Plan
Automation Report
Loopback PCAP
eth0 PCAP
Wireshark 统计
Ubuntu Access Log
Wazuh zap-lab 事件
Hardened Server
修复前后对比
完整实验报告
```

检查 Kali：

```bash
find ~/CyberLab/ZAP -type f
```

检查 Ubuntu：

```bash
find ~/CyberLab/ZAP -type f
```

---

# 115. 下一课

下一课建议：

```text
SQLmap
```

学习内容：

- SQL Injection 基础
- GET 和 POST 参数
- Boolean、Error、Time-based 检测概念
- DBMS Fingerprinting
- 安全等级与风险参数
- 只在本地 SQLite 实验应用测试
- Traffic File 和 HTTP Request File
- Burp/ZAP Request 导入
- 输出和 Session
- Access Log、PCAP 和 Wazuh 观察
- 修复参数化查询并复测

当前学习关系：

```text
Nmap：发现服务
Netcat：测试连接
Wireshark：图形化抓包
tcpdump：命令行抓包
curl：手动 HTTP 请求
WhatWeb：技术识别
Gobuster/ffuf：内容发现
Nikto：服务器安全检查
Burp Suite：手工代理分析
OWASP ZAP：代理、扫描和自动化
```

---

# 116. 参考说明

本指南依据当前：

```text
OWASP ZAP 官方 Desktop User Guide
OWASP ZAP Automation Framework 文档
OWASP ZAP Client Spider 文档
OWASP ZAP Report Generation 文档
Kali Linux zaproxy 工具页面
本机 zaproxy -h 输出
```

当前官方文档说明：

```text
Protected Mode 推荐用于限制危险操作到 Scope
Active Scan 会使用已知攻击方式测试目标
Client Spider 是现代 Web 应用的推荐爬虫
报告模板支持 HTML、JSON、Markdown、PDF、XML 和 SARIF 等格式
Automation Framework 使用 -autorun 执行 YAML Plan
```

工具更新后，以本机帮助和已安装 Add-ons 为准：

```bash
zaproxy -version
zaproxy -h
zaproxy -cmd -addonlist
```
