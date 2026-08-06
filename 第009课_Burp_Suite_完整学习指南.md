# 第九课：Burp Suite 完整学习指南

> **实验环境：你自己的 CyberLab**
>
> 本课程只用于你自己创建、拥有并控制的 Web 服务。  
> 不要拦截、修改或测试学校、公司、公共网站、他人账号或任何未经明确授权的系统。

---

# 1. 本课目标

学完本课后，你应该能够：

```text
理解 Burp Suite 的作用
区分 Community Edition 与 Professional
启动 Burp 并检查 Proxy Listener
使用 Burp 自带浏览器
理解 Intercept、Forward、Drop 和 HTTP history
设置严格的 Target Scope
使用 Site map 和 Inspector
把请求发送到 Repeater 并重复修改
使用 Decoder 和 Comparer
进行极小规模 Intruder Demo
理解 Sequencer 的用途
配置外部 Firefox 和 Burp CA
让 curl 通过 Burp Proxy
保存请求、响应、PCAP 和实验报告
结合 tcpdump、Wireshark、Access Log 和 Wazuh
排查 Listener、Proxy 和证书问题
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
Burp Suite：Kali 10.10.20.10
实验 Web Server：Ubuntu 10.10.20.20:8086
默认 Burp Proxy Listener：127.0.0.1:8080
```

---

# 2. 什么是 Burp Suite？

Burp Suite 是一个集成式 Web 应用安全测试平台。

核心工作方式：

```text
浏览器
  ↓
Burp Proxy
  ↓
目标 Web Server
```

Burp 可以对 HTTP/HTTPS 请求进行：

- 捕获
- 查看
- 修改
- 转发
- 丢弃
- 保存
- 重复发送
- 比较
- 编码和解码

Community Edition 的重点是：

```text
手动观察请求
手动修改参数
手动重复验证
理解浏览器与服务器之间的通信
```

---

# 3. Burp 与前面工具的关系

```text
Nmap：发现端口和服务
curl：命令行发送 HTTP 请求
WhatWeb：识别 Web 技术
Gobuster/ffuf：发现内容和参数
Nikto：检查 Web Server 配置
Burp Suite：拦截、修改、重放和分析请求
tcpdump：命令行抓包
Wireshark：图形化流量分析
Wazuh：日志和安全事件分析
```

推荐工作流：

```text
Nmap
→ curl
→ WhatWeb
→ Gobuster/ffuf
→ Burp
→ Nikto
→ 日志和 PCAP
→ 修复并复测
```

---

# 4. Community 与 Professional

## Community Edition

官方定位：

```text
免费、核心手动测试工具包
```

主要包含：

- HTTP/HTTPS 和 WebSocket Proxy
- HTTP history
- Repeater
- Decoder
- Sequencer
- Comparer
- Intruder Demo

## Professional

在 Community 基础上增加：

- Web Vulnerability Scanner
- 完整 Intruder
- Project Files
- 自动 Crawl 和 Content Discovery
- Collaborator/OAST
- Professional 扩展和自动化功能

本课使用：

```text
Burp Suite Community Edition
```

因此不要寻找 Community 中不存在的：

```text
自动 Scanner
完整高速 Intruder
Professional Project Files
自动扫描报告
```

---

# 5. 主要工具

| 工具 | 作用 |
|---|---|
| Proxy | 拦截浏览器和服务器流量 |
| HTTP history | 查看经过 Proxy 的历史请求 |
| Target | 查看目标和 Site map |
| Scope | 限制授权测试范围 |
| Repeater | 修改并重复发送请求 |
| Inspector | 结构化查看参数、Header、Cookie |
| Decoder | 编码、解码和 Hash |
| Comparer | 比较两段数据 |
| Intruder | 自动替换 Payload 位置 |
| Sequencer | 分析 Token 随机性 |

---

# 6. 安全规则

Burp 可以修改真实请求。错误操作可能：

- 修改数据
- 重复提交表单
- 触发账号锁定
- 暴露 Cookie 和 Token
- 产生大量请求

因此：

```text
只测试自己的实验服务器
先设置 Scope
不要测试真实银行、邮箱、学校或政府系统
不要拦截真实账号密码和 Session Cookie
不要把 Burp CA 安装到不必要的个人设备
不要公开上传 Burp 请求历史和 Token
```

---

# 7. 检查和安装

Kali：

```bash
which burpsuite
```

```bash
apt policy burpsuite
```

启动：

```bash
burpsuite
```

如果没有安装：

```bash
sudo apt update
sudo apt install burpsuite
```

---

# 8. 第一次启动

常见启动过程：

```text
Community Edition
→ Temporary project
→ Use Burp defaults
→ Start Burp
```

Community 主要使用临时项目，因此本课证据要单独保存。

---

# 9. 创建实验目录

Kali：

```bash
mkdir -p ~/CyberLab/Burp/{requests,responses,captures,notes,reports,screenshots}
```

Ubuntu：

```bash
mkdir -p ~/CyberLab/Burp/{server,logs}
```

---

# 10. Ubuntu Burp 实验服务器

服务器提供：

```text
主页和链接
GET 参数
POST 表单
JSON 请求
Header 回显
Cookie
Redirect
不同状态码
Token 示例
```

它不会执行系统命令或修改敏感文件。

---

# 11. 创建服务器代码

Ubuntu：

```bash
nano ~/CyberLab/Burp/server/burp_lab.py
```

粘贴：

```python
#!/usr/bin/env python3

import html
import json
import secrets
from http.server import BaseHTTPRequestHandler, HTTPServer
from urllib.parse import parse_qs, urlparse

HOST = "10.10.20.20"
PORT = 8086


def page(title: str, body: str) -> bytes:
    return f"""<!doctype html>
<html lang="en">
<head><meta charset="utf-8"><title>{html.escape(title)}</title></head>
<body><h1>{html.escape(title)}</h1>{body}</body>
</html>
""".encode("utf-8")


class BurpLabHandler(BaseHTTPRequestHandler):
    server_version = "CyberLabBurp/1.0"

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
        self.send_header("X-CyberLab", "Burp-Suite-Lab")
        if headers:
            for name, value in headers.items():
                self.send_header(name, value)
        self.end_headers()
        if self.command != "HEAD":
            self.wfile.write(body)

    def request_details(self, body: bytes = b"") -> dict:
        parsed = urlparse(self.path)
        return {
            "method": self.command,
            "path": parsed.path,
            "query": parse_qs(parsed.query),
            "headers": dict(self.headers),
            "body": body.decode("utf-8", errors="replace"),
            "client_ip": self.client_address[0],
        }

    def do_GET(self) -> None:
        parsed = urlparse(self.path)
        path = parsed.path
        query = parse_qs(parsed.query)

        if path == "/":
            body = """
<ul>
  <li><a href="/item?id=1">Item 1</a></li>
  <li><a href="/item?id=2">Item 2</a></li>
  <li><a href="/headers">Headers</a></li>
  <li><a href="/cookie">Set Cookie</a></li>
  <li><a href="/redirect">Redirect</a></li>
  <li><a href="/status/403">403 Response</a></li>
  <li><a href="/token">Token Example</a></li>
</ul>
<form action="/form" method="post">
  <label>Name: <input name="name" value="student"></label>
  <label>Course: <input name="course" value="burp"></label>
  <button type="submit">Submit Form</button>
</form>
"""
            self.send_body(200, page("CyberLab Burp Suite", body))
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
                    json.dumps(items[item_id], indent=2).encode(),
                    "application/json",
                )
            else:
                self.send_body(
                    404,
                    b'{"error":"item not found"}\n',
                    "application/json",
                )
            return

        if path == "/headers":
            self.send_body(
                200,
                json.dumps(self.request_details(), indent=2).encode(),
                "application/json",
            )
            return

        if path == "/cookie":
            self.send_body(
                200,
                page("Cookie Created", "<p>A training cookie was set.</p>"),
                headers={
                    "Set-Cookie": (
                        "lab_session=student; Path=/; "
                        "HttpOnly; SameSite=Lax"
                    )
                },
            )
            return

        if path == "/redirect":
            self.send_response(302)
            self.send_header("Location", "/final")
            self.send_header("Content-Length", "0")
            self.end_headers()
            return

        if path == "/final":
            self.send_body(
                200,
                page("Redirect Complete", "<p>302 redirect completed.</p>"),
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
                f"CyberLab status {status}\n".encode(),
                "text/plain; charset=utf-8",
            )
            return

        if path == "/token":
            token = secrets.token_hex(16)
            self.send_body(
                200,
                page("Token Example", f"<p>Training token: {html.escape(token)}</p>"),
                headers={
                    "Set-Cookie": (
                        f"lab_token={token}; Path=/; "
                        "HttpOnly; SameSite=Lax"
                    )
                },
            )
            return

        self.send_body(
            404,
            b"CyberLab Burp resource not found\n",
            "text/plain; charset=utf-8",
        )

    def do_HEAD(self) -> None:
        self.do_GET()

    def do_POST(self) -> None:
        length = int(self.headers.get("Content-Length", "0"))
        body = self.rfile.read(length)

        if self.path == "/form":
            result = {
                "method": "POST",
                "content_type": self.headers.get("Content-Type"),
                "form": parse_qs(body.decode("utf-8", errors="replace")),
            }
            self.send_body(
                200,
                json.dumps(result, indent=2).encode(),
                "application/json",
            )
            return

        if self.path == "/json":
            try:
                parsed_json = json.loads(body.decode("utf-8"))
                result = {"received": parsed_json, "valid_json": True}
                status = 200
            except (json.JSONDecodeError, UnicodeDecodeError):
                result = {"error": "invalid JSON", "valid_json": False}
                status = 400
            self.send_body(
                status,
                json.dumps(result, indent=2).encode(),
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
        x_lab = self.headers.get("X-Lab-Request", "-")
        print(
            f'{self.client_address[0]} '
            f'[{self.log_date_time_string()}] '
            f'UA="{user_agent}" '
            f'X-Lab-Request="{x_lab}" '
            f'{format % args}',
            flush=True,
        )


def main() -> None:
    server = HTTPServer((HOST, PORT), BurpLabHandler)
    print(f"Burp lab listening on http://{HOST}:{PORT}")
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

赋予权限：

```bash
chmod +x ~/CyberLab/Burp/server/burp_lab.py
```

---

# 12. 启动服务器

Ubuntu：

```bash
python3 ~/CyberLab/Burp/server/burp_lab.py \
  2>&1 | \
tee ~/CyberLab/Burp/logs/access.log
```

预期：

```text
Burp lab listening on http://10.10.20.20:8086
```

第二个 Ubuntu 终端：

```bash
ss -lntp | grep ':8086'
```

---

# 13. Kali 直接验证

```bash
ping -c 3 10.10.20.20
```

```bash
nc -zvn -w 3 10.10.20.20 8086
```

```bash
curl -i http://10.10.20.20:8086/
```

```bash
sudo nmap -n -sV -p 8086 10.10.20.20
```

---

# 14. Proxy Listener

Burp 默认 Listener 通常是：

```text
127.0.0.1:8080
```

检查：

```text
Proxy
→ Proxy settings
→ Proxy listeners
```

应看到：

```text
Running
127.0.0.1:8080
```

Kali 命令：

```bash
ss -lntp | grep ':8080'
```

---

# 15. 为什么绑定 127.0.0.1？

```text
127.0.0.1
```

表示只有 Kali 本机程序可以连接。

本课不要改成：

```text
All interfaces
0.0.0.0
10.10.20.10
```

否则其他虚拟机可能使用你的 Proxy。

---

# 16. Listener 端口冲突

如果出现：

```text
Failed to start proxy service
```

检查：

```bash
sudo ss -lntp | grep ':8080'
```

可把 Listener 改为：

```text
127.0.0.1:8081
```

浏览器和 curl 也必须使用相同端口。

---

# 17. 使用 Burp 自带浏览器

进入：

```text
Proxy
→ Intercept
→ Open Browser
```

Burp Browser 已经：

- 配置 Proxy
- 信任 Burp CA
- 支持 HTTP 和 HTTPS
- 使用隔离浏览器环境

打开：

```text
http://10.10.20.20:8086/
```

使用内置浏览器时，不需要手工设置 Firefox Proxy 或安装 CA。

---

# 18. Intercept is on

当显示：

```text
Intercept is on
```

浏览器请求会停在 Burp 中。

操作：

```text
Forward：发送给服务器
Drop：丢弃请求
Intercept is off：不再暂停请求，但继续记录历史
```

---

# 19. 第一次拦截

1. 开启 `Intercept is on`。
2. Burp Browser 访问：

```text
http://10.10.20.20:8086/item?id=1
```

3. Burp 中应看到：

```http
GET /item?id=1 HTTP/1.1
Host: 10.10.20.20:8086
User-Agent: ...
```

4. 点击：

```text
Forward
```

---

# 20. 修改拦截请求

再次访问 `id=1`，在 Burp 中改成：

```text
id=42
```

点击 `Forward`。

浏览器应显示：

```json
{
  "id": 42,
  "name": "Lab Answer",
  "price": 42
}
```

这证明：

```text
Burp 可以修改浏览器真正发送给服务器的请求
```

---

# 21. Drop

访问：

```text
http://10.10.20.20:8086/headers
```

在 Burp 点击：

```text
Drop
```

结果：

```text
请求不会到达 Ubuntu
Access Log 不应出现该请求
```

---

# 22. Intercept is off

正常浏览时推荐：

```text
Intercept is off
```

请求不会被暂停，但仍会出现在：

```text
Proxy
→ HTTP history
```

---

# 23. HTTP history

进入：

```text
Proxy
→ HTTP history
```

这里可以看到所有经过 Burp Proxy 的请求，包括：

- Method
- Host
- URL
- Status
- Length
- MIME Type
- Extension
- Time
- Listener Port
- Edited 状态

点击一条记录后，可以查看：

```text
Request
Response
Raw
Pretty
Hex
Inspector
```

---

# 24. HTTP history 过滤

HTTP history 可能出现浏览器产生的额外请求。

使用 Filter：

```text
Show only in-scope items
Hide image/CSS/font requests
Filter by status code
Filter by MIME type
Search by text
```

本课只关注：

```text
10.10.20.20:8086
```

---

# 25. 设置 Target Scope

Scope 告诉 Burp：

```text
哪些 Host、Port 和 URL 是本次授权目标
```

进入：

```text
Target
→ Scope
```

添加：

```text
Protocol：http
Host：10.10.20.20
Port：8086
```

也可以在 HTTP history 中右键目标请求：

```text
Add to scope
```

Burp 可能询问是否停止记录 Out-of-scope 流量。

选择严格范围可以减少噪音。

---

# 26. 为什么 Scope 很重要？

Scope 可以帮助：

- 防止误测其他网站
- 过滤浏览器后台请求
- 让 Site map 更干净
- 避免把 Wazuh、Kali 或其他站点一起测试
- 明确实验授权边界

本课 Scope 只有：

```text
http://10.10.20.20:8086/
```

---

# 27. Target Site map

进入：

```text
Target
→ Site map
```

浏览实验网站后，可能看到：

```text
/
/item?id=1
/item?id=2
/headers
/cookie
/redirect
/final
/status/403
/token
/form
```

Site map 来自：

```text
Burp 观察到的请求和响应
```

Community 不提供 Professional 的自动 Crawl，因此需要通过浏览器、curl 或其他工具产生请求。

---

# 28. Site map 不等于服务器全部内容

Site map 只包含 Burp 已经观察或导入的内容。

没有出现的路径可能：

- 尚未访问
- 没有页面链接
- 需要其他参数
- 需要其他 Host
- 需要认证

可以把 Gobuster/ffuf 发现的路径再通过 Burp 访问。

---

# 29. Inspector

Inspector 通常位于请求编辑器右侧。

它可以结构化显示：

- Request attributes
- Query parameters
- Body parameters
- Cookies
- Request headers
- Response headers

例如请求：

```http
GET /item?id=1 HTTP/1.1
Host: 10.10.20.20:8086
```

Inspector 会把：

```text
id = 1
```

显示为 Query Parameter。

---

# 30. Inspector 修改参数

在 Repeater 或 Intercept 中：

1. 打开 Inspector。
2. 找到 `Query parameters`。
3. 把 `id` 从 `1` 改为 `2`。
4. 请求 Raw 文本会同步变化。
5. 发送请求。

Inspector 与 Raw 编辑器是同一个请求的不同视图。

---

# 31. Send to Repeater

在 HTTP history 中选择：

```text
GET /item?id=1
```

右键：

```text
Send to Repeater
```

常见快捷键：

```text
Ctrl + R
```

然后进入：

```text
Repeater
```

---

# 32. Repeater 的作用

Repeater 可以：

```text
修改请求
点击 Send
查看响应
再次修改
再次发送
比较结果
```

官方描述中，Repeater 用于重复发送修改后的 HTTP 或 WebSocket 消息。

它是手动 Web 测试中最常用的工具之一。

---

# 33. Repeater 实验：修改 ID

原始请求：

```http
GET /item?id=1 HTTP/1.1
Host: 10.10.20.20:8086
```

点击：

```text
Send
```

记录响应：

```text
Status：200
Body：Keyboard
```

把 ID 改为：

```text
2
```

再次发送。

再改为：

```text
42
```

再改为：

```text
99
```

比较：

| ID | Status | Name/Message |
|---:|---:|---|
| 1 | 200 | Keyboard |
| 2 | 200 | Mouse |
| 42 | 200 | Lab Answer |
| 99 | 404 | item not found |

---

# 34. Repeater History

Repeater 提供请求历史。

使用请求编辑器上方的前进/后退按钮，可以查看：

- 之前发送的请求
- 对应响应
- 修改差异

这适合恢复到某个已知测试状态。

---

# 35. Repeater 修改 Header

在请求中添加：

```http
X-Lab-Request: repeater-header-test
```

发送到：

```text
/headers
```

例如：

```http
GET /headers HTTP/1.1
Host: 10.10.20.20:8086
X-Lab-Request: repeater-header-test
Connection: close
```

响应 JSON 中应出现该 Header。

Ubuntu Access Log 也会记录：

```text
X-Lab-Request="repeater-header-test"
```

---

# 36. Repeater 修改 Cookie

先访问：

```text
/cookie
```

HTTP history 中找到后续请求，可能包含：

```http
Cookie: lab_session=student
```

发送到 Repeater，修改为：

```http
Cookie: lab_session=burp-test
```

再请求：

```text
/headers
```

响应会回显修改后的 Cookie。

注意：

```text
这只是自己的实验 Cookie
不要修改真实网站 Session Cookie
```

---

# 37. Repeater POST 表单

在浏览器主页提交表单。

HTTP history 中应看到：

```http
POST /form HTTP/1.1
Content-Type: application/x-www-form-urlencoded

name=student&course=burp
```

发送到 Repeater。

修改：

```text
name=analyst
course=burp-suite
```

点击 `Send`，检查服务器回显。

---

# 38. Repeater POST JSON

可以从 curl 通过 Burp 产生 JSON 请求，也可以在 Repeater 中手工创建：

```http
POST /json HTTP/1.1
Host: 10.10.20.20:8086
Content-Type: application/json
Connection: close
Content-Length: 54

{
  "student": "analyst",
  "lesson": "burp"
}
```

Burp 通常会在编辑后重新计算 Content-Length。

发送后应返回：

```json
{
  "received": {
    "student": "analyst",
    "lesson": "burp"
  },
  "valid_json": true
}
```

---

# 39. 无效 JSON

把 Body 改为：

```json
{
  "student": "analyst",
  "lesson": burp
}
```

由于 `burp` 没有双引号，这是无效 JSON。

预期：

```text
Status：400
valid_json：false
```

这可以帮助理解：

```text
语法错误
Content-Type
服务器验证
状态码
```

---

# 40. Pretty、Raw 和 Hex

消息编辑器常见视图：

## Pretty

格式化显示：

- JSON
- HTML
- XML
- Header

## Raw

显示原始 HTTP 文本。

## Hex

显示十六进制和原始字节。

初学建议：

```text
Pretty 看结构
Raw 学 HTTP
Hex 排查特殊字符
```

---

# 41. Decoder

进入：

```text
Decoder
```

Decoder 可以：

- URL Encode/Decode
- HTML Encode/Decode
- Base64 Encode/Decode
- ASCII Hex Encode/Decode
- Hash
- 多层转换

也可以在消息编辑器中选中文字，右键：

```text
Send to Decoder
```

---

# 42. URL Encoding 实验

原始文本：

```text
Cyber Lab & Burp
```

URL Encode 后可能类似：

```text
Cyber%20Lab%20%26%20Burp
```

URL Decode 可以恢复原文。

常见编码：

| 字符 | URL 编码 |
|---|---|
| 空格 | `%20` |
| `&` | `%26` |
| `=` | `%3D` |
| `/` | `%2F` |
| `?` | `%3F` |

---

# 43. Base64 实验

输入：

```text
CyberLab-Burp
```

Base64 Encode 后会得到一段可打印字符。

再执行 Base64 Decode，恢复原文。

重要：

```text
Base64 是编码
不是加密
```

不要把 Base64 当作密码保护。

---

# 44. HTML Encoding

输入：

```text
<student>&"burp"
```

HTML Encode 后可能包含：

```text
&lt;
&gt;
&amp;
&quot;
```

编码的目的通常是让数据安全地出现在特定语法环境中。

---

# 45. Hash

Decoder 可以计算常见 Hash，例如：

- MD5
- SHA-1
- SHA-256

Hash 是单向摘要，不是可逆加密。

对安全用途优先使用现代算法，例如 SHA-256；不要用 MD5 存储密码。

---

# 46. Comparer

Comparer 用于比较两段请求或响应。

可以：

```text
Words Compare
Bytes Compare
```

适合发现：

- 状态码变化
- Header 变化
- Cookie 变化
- JSON 字段变化
- 响应长度差异
- 很小的文本差异

---

# 47. Comparer 实验

在 Repeater 中分别发送：

```text
/item?id=1
/item?id=2
```

对第一个响应右键：

```text
Send to Comparer
```

对第二个响应执行相同操作。

进入：

```text
Comparer
```

选择两个项目，点击：

```text
Words
```

应该看到：

```text
id 1 ↔ id 2
Keyboard ↔ Mouse
25 ↔ 15
```

---

# 48. 比较 200 与 404

比较：

```text
/item?id=1
/item?id=99
```

记录：

- Status
- Body
- Content-Length
- JSON 字段
- Response Headers

Comparer 可以辅助判断响应是否真正不同。

---

# 49. Intruder 的用途

Intruder 会：

```text
使用同一个请求模板
在指定位置插入不同 Payload
重复发送请求
整理结果
```

Community Edition 提供 Intruder Demo，速度和功能受到限制。

本课只做：

```text
4 个数字 ID
自己的实验服务器
低请求量
```

不进行密码攻击或账号枚举。

---

# 50. Send to Intruder

在 Repeater 或 HTTP history 中选择：

```http
GET /item?id=1 HTTP/1.1
```

右键：

```text
Send to Intruder
```

进入：

```text
Intruder
```

---

# 51. Intruder Positions

在 Positions 中清除自动位置，再只选择数字：

```text
id=§1§
```

Attack Type 使用最简单的：

```text
Sniper
```

只保留一个 Payload Position。

---

# 52. Intruder Payloads

Payload List：

```text
1
2
42
99
```

开始 Attack。

比较：

| Payload | Status | Length | 结果 |
|---:|---:|---:|---|
| 1 | 200 |  | Keyboard |
| 2 | 200 |  | Mouse |
| 42 | 200 |  | Lab Answer |
| 99 | 404 |  | Not Found |

---

# 53. Intruder 安全边界

本课禁止：

- 密码爆破
- 登录尝试
- OTP 测试
- 账号枚举
- 高并发请求
- 大型 Payload List
- 未授权目标

Intruder 只用于理解：

```text
Payload Position
Payload List
Status
Length
结果差异
```

---

# 54. Sequencer

Sequencer 用于分析：

```text
Session Token
CSRF Token
Password Reset Token
其他需要不可预测的 Token
```

实验服务器的：

```text
/token
```

每次返回随机 Token。

Sequencer 通常需要大量样本才能进行统计分析。

本课只学习概念，不运行长时间大样本采集。

---

# 55. 为什么少量样本不够？

看到 5 个 Token 不重复，不代表 Token 安全。

随机性分析需要考虑：

- 样本数量
- 字符分布
- Bit 分布
- 可预测模式
- 生成算法
- 生命周期
- 绑定关系

Burp Sequencer 结果只是统计线索，不代替服务器端代码审计。

---

# 56. 外部 Firefox：什么时候需要？

优先使用：

```text
Burp 自带浏览器
```

外部 Firefox 适合：

- 使用自己的 Firefox Profile
- 测试特定浏览器设置
- 使用浏览器扩展
- Burp Browser 无法启动

使用外部浏览器需要两部分：

```text
1. 配置 Proxy
2. HTTPS 时安装 Burp CA
```

---

# 57. Firefox Proxy 配置

在 Kali Firefox：

```text
Settings
→ General
→ Network Settings
→ Settings
→ Manual proxy configuration
```

填写：

```text
HTTP Proxy：127.0.0.1
Port：8080
```

选择把该 Proxy 用于 HTTPS，或在 HTTPS Proxy 中填写相同地址和端口。

不要把 Proxy 设置成：

```text
10.10.20.20
```

Burp 运行在 Kali，本地 Listener 是：

```text
127.0.0.1:8080
```

---

# 58. 测试外部 Firefox Proxy

Burp 保持运行，把 Intercept 设为：

```text
Intercept is off
```

Firefox 访问：

```text
http://10.10.20.20:8086/
```

检查：

```text
Proxy
→ HTTP history
```

应该看到该请求。

如果看不到，说明 Firefox 没有通过 Burp。

---

# 59. Burp CA 的作用

访问 HTTPS 时：

```text
浏览器 ↔ Burp
Burp ↔ HTTPS Server
```

Burp 需要为浏览器动态生成目标证书。

浏览器必须信任：

```text
Burp CA
```

否则会显示证书警告。

使用 Burp 自带浏览器时，CA 已经配置好。

---

# 60. 下载 Burp CA

仅在已经配置为使用 Burp Proxy 的外部 Firefox 中访问：

```text
http://burpsuite
```

选择：

```text
CA Certificate
```

通常下载文件：

```text
cacert.der
```

不要从随机网站下载 Burp CA。

---

# 61. Firefox 导入 CA

Firefox：

```text
Settings
→ Privacy & Security
→ Certificates
→ View Certificates
→ Authorities
→ Import
```

选择：

```text
cacert.der
```

信任该 CA 用于识别网站。

完成后重新启动 Firefox。

---

# 62. CA 安全规则

Burp CA 允许 Burp 解密这个浏览器的 HTTPS 流量。

因此：

```text
只安装在 Kali 实验浏览器 Profile
不要安装到 Mac 主机的个人浏览器
不要分享 Burp CA 私钥
实验完成后可删除 CA
不要用该浏览器登录真实账号
```

从 Firefox Authorities 中删除 Burp CA，可以恢复正常信任状态。

---

# 63. TLS 拦截概念

HTTPS 通过 Burp 时存在两条连接：

```text
浏览器 → Burp：TLS Connection 1
Burp → Server：TLS Connection 2
```

Burp 在中间：

- 解密浏览器请求
- 显示和允许修改
- 再加密发给服务器
- 解密服务器响应
- 再加密发给浏览器

这是授权测试中的受控 TLS Interception。

---

# 64. curl 通过 Burp Proxy

确保 Burp Listener 正在运行，Intercept 先关闭。

Kali：

```bash
curl \
  -x http://127.0.0.1:8080 \
  -H 'X-Lab-Request: curl-through-burp' \
  http://10.10.20.20:8086/headers
```

然后在 Burp：

```text
Proxy
→ HTTP history
```

应该看到 curl 请求。

---

# 65. curl 与 Intercept

开启：

```text
Intercept is on
```

再运行 curl。

curl 会等待，因为请求停在 Burp 中。

在 Burp 点击：

```text
Forward
```

curl 才会收到响应。

如果点击：

```text
Drop
```

curl 会失败或收到连接被关闭的错误。

---

# 66. curl POST JSON 通过 Burp

```bash
curl \
  -x http://127.0.0.1:8080 \
  -H 'Content-Type: application/json' \
  -H 'X-Lab-Request: curl-json' \
  -d '{"student":"analyst","lesson":"burp"}' \
  http://10.10.20.20:8086/json
```

在 HTTP history 中：

```text
Send to Repeater
```

再修改 JSON 并发送。

---

# 67. 保存原始请求

在 HTTP history 或 Repeater 中选择请求。

可以：

- 右键保存项目，具体菜单名称随版本变化
- 复制 Raw Request 到文本文件
- 保存截图
- 保存对应响应

手工文件示例：

```bash
nano ~/CyberLab/Burp/requests/item-id-1.txt
```

内容：

```http
GET /item?id=1 HTTP/1.1
Host: 10.10.20.20:8086
Connection: close
```

---

# 68. 保存响应

```bash
nano ~/CyberLab/Burp/responses/item-id-1-response.txt
```

保存：

```text
Status Line
Response Headers
Response Body
测试时间
对应请求文件
```

注意删除真实 Cookie 或 Token 后再分享。

---

# 69. Proxy 流量与网卡

Burp 实验有两段流量：

```text
浏览器/curl → Burp Listener
127.0.0.1:随机端口 → 127.0.0.1:8080
```

```text
Burp → Ubuntu Server
10.10.20.10:随机端口 → 10.10.20.20:8086
```

所以：

```text
lo   可以看到 Browser/curl ↔ Burp
eth0 可以看到 Burp ↔ Ubuntu
```

---

# 70. 抓取 Burp 到 Ubuntu 的流量

Kali：

```bash
sudo tcpdump \
  -i eth0 \
  -nn \
  -s 0 \
  -w ~/CyberLab/Burp/captures/burp-to-ubuntu.pcap \
  'host 10.10.20.20 and tcp port 8086'
```

然后用 Burp Browser 或 Repeater 发送请求。

结束：

```text
Ctrl + C
```

---

# 71. 抓取浏览器到 Burp 的流量

Kali：

```bash
sudo tcpdump \
  -i lo \
  -nn \
  -s 0 \
  -w ~/CyberLab/Burp/captures/browser-to-burp.pcap \
  'tcp port 8080'
```

这段流量只发生在 Kali Loopback。

---

# 72. Wireshark 分析

打开 Burp 到 Ubuntu 的 PCAP：

```bash
wireshark \
  ~/CyberLab/Burp/captures/burp-to-ubuntu.pcap
```

过滤：

```text
http
```

```text
http.request
```

```text
tcp.port == 8086
```

查看：

- GET / POST
- URI
- Header
- Cookie
- JSON Body
- Status
- Redirect
- Content-Length

---

# 73. Burp 与 Wireshark 的区别

| Burp Suite | Wireshark |
|---|---|
| 理解和修改 HTTP | 分析所有网络协议 |
| 代理层工作 | 数据包层工作 |
| 方便 Repeater | 不能主动重放 HTTP |
| 自动处理 HTTP 结构 | 显示 Ethernet/IP/TCP/HTTP |
| 可以修改请求 | 主要观察和分析 |

组合：

```text
Burp 看应用请求
Wireshark 看网络证据
```

---

# 74. Ubuntu Access Log

查看：

```bash
less ~/CyberLab/Burp/logs/access.log
```

只看 Kali：

```bash
grep '10.10.20.10' \
  ~/CyberLab/Burp/logs/access.log
```

查自定义 Header：

```bash
grep 'repeater-header-test' \
  ~/CyberLab/Burp/logs/access.log
```

统计请求：

```bash
grep -c '"GET ' \
  ~/CyberLab/Burp/logs/access.log
```

---

# 75. Wazuh 日志链路

Ubuntu 实验前：

```bash
logger -t burp-lab \
  "Starting authorized Burp Suite lab from Kali 10.10.20.10"
```

实验后：

```bash
logger -t burp-lab \
  "Completed authorized Burp Suite lab against TCP 8086"
```

本地：

```bash
journalctl \
  -t burp-lab \
  --since '15 minutes ago'
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
burp-lab
```

---

# 76. Burp 是否一定触发 Wazuh 告警？

不一定。

Burp 本身只是 HTTP 客户端和 Proxy。

自动检测通常需要：

- Web Access Log
- Wazuh 收集该日志
- User-Agent 或行为规则
- WAF
- IDS/IPS
- Suricata
- Zeek

本课成功标准：

```text
Burp HTTP history 有请求
Ubuntu Access Log 有请求
PCAP 有流量
Wazuh 能看到 burp-lab 手工日志
```

---

# 77. 完整实验一：Proxy 和 Intercept

1. 启动 Ubuntu 8086 Server。
2. 启动 Burp。
3. 检查 `127.0.0.1:8080` Listener。
4. 打开 Burp Browser。
5. 开启 Intercept。
6. 访问：

```text
http://10.10.20.20:8086/item?id=1
```

7. 修改为 `id=42`。
8. Forward。
9. 再拦截一个请求并 Drop。
10. 检查 Access Log。

---

# 78. 完整实验二：History、Scope 和 Site map

1. 把 `10.10.20.20:8086` 加入 Scope。
2. 关闭 Intercept。
3. 浏览主页所有链接。
4. 提交一次表单。
5. 查看 HTTP history。
6. Filter 只显示 In-scope。
7. 查看 Target Site map。
8. 记录所有路径、Method 和 Status。

---

# 79. 完整实验三：Repeater

从 HTTP history 发送到 Repeater：

```text
/item?id=1
```

测试：

```text
1
2
42
99
```

再测试：

- 添加 `X-Lab-Request`
- 修改 Cookie
- 修改表单 Body
- 发送有效 JSON
- 发送无效 JSON

保存每个关键请求和响应。

---

# 80. 完整实验四：Decoder 和 Comparer

Decoder：

```text
URL Encode/Decode
Base64 Encode/Decode
HTML Encode/Decode
SHA-256
```

Comparer：

```text
比较 item 1 与 item 2
比较 item 1 与 item 99
```

记录：

- Status 差异
- Body 差异
- Length 差异
- Header 差异

---

# 81. 完整实验五：Intruder Demo

只使用：

```text
1
2
42
99
```

目标：

```text
/item?id=§1§
```

Attack Type：

```text
Sniper
```

分析：

- Status
- Length
- Response Body

不要使用账号、密码或大型 Payload List。

---

# 82. 完整实验六：curl、PCAP 和 Wazuh

Ubuntu：

```bash
logger -t burp-lab \
  "Starting Burp proxy traffic capture"
```

Kali 终端一：

```bash
sudo tcpdump \
  -i eth0 \
  -nn \
  -s 0 \
  -w ~/CyberLab/Burp/captures/full-lab.pcap \
  'host 10.10.20.20 and tcp port 8086'
```

Kali 终端二：

```bash
curl \
  -x http://127.0.0.1:8080 \
  -H 'X-Lab-Request: full-lab' \
  http://10.10.20.20:8086/headers
```

在 Burp Repeater 中再发送一次修改请求。

停止抓包，使用 Wireshark 分析。

Ubuntu：

```bash
logger -t burp-lab \
  "Completed Burp proxy traffic capture"
```

在 Wazuh 搜索：

```text
burp-lab
```

---

# 83. 常见错误：页面一直加载

原因通常是：

```text
Intercept is on
请求正在 Burp 中等待
```

解决：

```text
Forward
或
Intercept is off
```

---

# 84. HTTP history 没有请求

检查：

1. Burp Listener 是否 Running。
2. 是否使用 Burp Browser。
3. 外部 Firefox 是否配置 `127.0.0.1:8080`。
4. 是否访问正确目标。
5. 是否错误使用 Ubuntu IP 作为 Proxy。
6. HTTP history Filter 是否隐藏结果。

---

# 85. HTTPS 证书警告

使用 Burp Browser：

```text
通常不应需要手工 CA 设置
```

使用外部 Firefox：

```text
需要安装 Burp CA
```

还要检查：

- 浏览器是否真的使用 Burp Proxy
- 是否导入当前 Burp 实例的 CA
- 是否导入到了 Authorities
- 是否重启浏览器

---

# 86. Burp Browser 无法启动

可以：

1. 关闭并重新打开 Burp。
2. 检查 Kali 磁盘和内存。
3. 查看 Burp Event Log。
4. 使用外部 Firefox。
5. 确认 Java 和 Burp 软件包正常。

Kali：

```bash
burpsuite
```

从终端启动可以看到错误输出。

---

# 87. Target Connection Refused

检查 Ubuntu：

```bash
ss -lntp | grep ':8086'
```

Kali：

```bash
ping -c 3 10.10.20.20
```

```bash
nc -zvn -w 3 10.10.20.20 8086
```

```bash
curl -i http://10.10.20.20:8086/
```

如果直接 curl 都失败，问题不在 Burp。

---

# 88. 127.0.0.1 的常见误解

```text
127.0.0.1:8080
```

是 Kali 本机的 Burp Listener。

```text
10.10.20.20:8086
```

是 Ubuntu 的目标服务。

不要把二者混淆。

---

# 89. 请求噪音太多

解决：

- 设置 Scope
- HTTP history 只显示 In-scope
- 使用 Burp Browser 的独立环境
- 关闭其他页面
- 隐藏图片、字体和 CSS
- 清空无关历史后重新实验

---

# 90. Repeater 响应不符合预期

检查：

- Host 是否正确
- Port 是否正确
- Method 是否正确
- Content-Type 是否正确
- JSON 是否有效
- Body 参数是否 URL Encode
- Cookie 是否有重复项
- Content-Length 是否更新
- Server 是否仍在运行

Burp 通常会自动维护 Content-Length，但仍应查看 Raw Request。

---

# 91. 证据保护

Burp 文件可能包含：

- Session Cookie
- CSRF Token
- Authorization Header
- 用户名
- 表单内容
- API Token
- 内部 IP 和路径

公开分享前搜索：

```bash
grep -RniE \
  'authorization|cookie|password|token|session|secret' \
  ~/CyberLab/Burp
```

---

# 92. 实验报告模板

## Lab Information

```text
Lab Name：Burp Suite Manual HTTP Testing
Date：
Student：
Proxy Host：Kali 10.10.20.10
Proxy Listener：127.0.0.1:8080
Target：Ubuntu 10.10.20.20:8086
Edition：Community Edition
Authorization：Personal CyberLab
```

## Objective

```text
使用 Burp Proxy、HTTP history、Scope、Repeater、Decoder、
Comparer 和 Intruder Demo 分析受控 HTTP 请求。
```

## Environment

```text
Burp Version：
Kali Package Version：
Browser：Burp Browser / Firefox
External CA Installed：Yes / No
```

## Scope

```text
Protocol：http
Host：10.10.20.20
Port：8086
Excluded Targets：All others
```

## Requests Tested

| Request | Modification | Status | Result |
|---|---|---:|---|
| `/item?id=1` | `id=42` |  |  |
| `/headers` | Added Header |  |  |
| `/form` | Modified Body |  |  |
| `/json` | Invalid JSON |  |  |

## Tools Used

```text
Proxy：
HTTP history：
Target Scope：
Repeater：
Inspector：
Decoder：
Comparer：
Intruder Demo：
```

## Evidence

```text
Raw Requests：
Raw Responses：
Screenshots：
Ubuntu Access Log：
PCAP：
Wireshark Filters：
Wazuh Event：
```

## Analysis

```text
Intercept On 和 Off 的区别：
Forward 与 Drop 的区别：
Repeater 的用途：
Scope 的价值：
Burp 与 Wireshark 的区别：
```

## Security Notes

```text
是否包含 Cookie：
是否包含 Token：
是否需要脱敏：
CA 是否已删除：
```

## Lessons Learned

```text
最重要的 Burp 工具：
最容易发生的错误：
下一步需要练习的内容：
```

---

# 93. 本课练习

1. 检查 Burp 是否安装。
2. 启动 Temporary Project。
3. 检查 `127.0.0.1:8080` Listener。
4. 启动 Ubuntu 8086 Server。
5. 使用 Burp Browser 访问首页。
6. 开启 Intercept。
7. 修改 `id=1` 为 `id=42`。
8. Drop 一个请求。
9. 关闭 Intercept。
10. 查看 HTTP history。
11. 把目标加入 Scope。
12. 只显示 In-scope 流量。
13. 查看 Site map。
14. 发送请求到 Repeater。
15. 测试 ID 1、2、42、99。
16. 添加 `X-Lab-Request` Header。
17. 修改 Cookie。
18. 修改 POST 表单。
19. 发送有效 JSON。
20. 发送无效 JSON。
21. 使用 Inspector 修改参数。
22. 使用 Decoder URL Encode。
23. 使用 Decoder Base64 Encode/Decode。
24. 使用 Comparer 比较两个响应。
25. 使用 Intruder Demo 测试 4 个 ID。
26. 配置外部 Firefox Proxy。
27. 可选安装 Burp CA。
28. 让 curl 通过 Burp。
29. 保存 PCAP。
30. 用 Wireshark 查看 HTTP。
31. 查看 Ubuntu Access Log。
32. 在 Wazuh 搜索 `burp-lab`。
33. 完成实验报告。
34. 删除不再需要的外部浏览器 CA。

---

# 94. 复习题

1. Burp Proxy 的作用是什么？
2. Community 与 Professional 的主要区别是什么？
3. 默认 Proxy Listener 是什么？
4. 为什么 Listener 应绑定 127.0.0.1？
5. Burp Browser 有什么优点？
6. `Intercept is on` 会发生什么？
7. Forward 与 Drop 的区别是什么？
8. Intercept Off 后是否还有 HTTP history？
9. Scope 有什么作用？
10. Site map 来自哪里？
11. Repeater 的主要用途是什么？
12. Inspector 有什么用途？
13. Decoder 能处理哪些格式？
14. Base64 是加密吗？
15. Comparer 有什么用途？
16. Intruder Positions 是什么？
17. 为什么本课只使用 4 个 Payload？
18. Sequencer 用于什么？
19. 外部 Firefox 为什么需要 Burp CA？
20. Burp Browser 是否需要手工安装 CA？
21. `http://burpsuite` 有什么用途？
22. curl 如何通过 Burp？
23. `lo` 与 `eth0` 分别能看到哪段流量？
24. Burp 与 Wireshark 的区别是什么？
25. Burp 是否一定触发 Wazuh 告警？
26. 页面一直加载最常见原因是什么？
27. 为什么不要用 Burp Browser 登录真实账号？
28. 为什么 Burp 请求历史不能直接公开？
29. Community 是否包含自动 Scanner？
30. 为什么设置 Scope 是第一批重要步骤之一？

---

# 95. 参考答案

1. 拦截、查看和修改浏览器与服务器之间的 Web 流量。
2. Community 是核心手动工具；Professional 增加 Scanner、完整 Intruder、Project Files 和自动化功能。
3. `127.0.0.1:8080`。
4. 防止其他机器连接本地 Proxy。
5. 已配置 Proxy 和 CA，适合立即测试。
6. 请求会暂停在 Burp 中等待处理。
7. Forward 发送请求，Drop 丢弃请求。
8. 有，经过 Proxy 的流量仍会记录。
9. 限制本次授权测试的 Host、Port 和 URL。
10. 来自 Burp 观察到或导入的流量。
11. 修改并重复发送同一请求。
12. 结构化查看和修改参数、Header、Cookie 和 Body。
13. URL、HTML、Base64、Hex 和 Hash 等。
14. 不是，只是编码。
15. 比较两个请求或响应的差异。
16. 指定 Payload 被插入的位置。
17. 控制请求量，只理解基本工作方式。
18. 分析 Token 随机性。
19. 让 Firefox 信任 Burp 动态生成的 HTTPS 证书。
20. 通常不需要，Burp Browser 已预配置。
21. 通过 Burp Proxy 下载 Burp CA 和访问欢迎页。
22. 使用 `curl -x http://127.0.0.1:8080 URL`。
23. `lo` 看浏览器到 Burp；`eth0` 看 Burp 到 Ubuntu。
24. Burp 负责 HTTP 代理和修改；Wireshark 负责数据包分析。
25. 不一定，需要日志收集和检测规则。
26. Intercept 仍然开启且请求等待 Forward。
27. Burp 会看到密码、Cookie 和 Token。
28. 可能包含敏感认证和会话信息。
29. 不包含。
30. 防止误测其他目标并减少噪音。

---

# 96. 快捷操作速查

| 操作 | 常见位置/快捷键 |
|---|---|
| Open Browser | Proxy → Intercept |
| Intercept On/Off | Proxy → Intercept |
| Forward | Intercept 页面 |
| Drop | Intercept 页面 |
| HTTP history | Proxy → HTTP history |
| Add to scope | 右键目标请求 |
| Send to Repeater | 右键 / `Ctrl+R` |
| Send to Intruder | 右键菜单 |
| Send to Decoder | 选中文字后右键 |
| Send to Comparer | 右键请求或响应 |
| Send | Repeater |

快捷键可能因操作系统和 Burp 版本略有不同。

---

# 97. curl 命令速查

## 通过 Burp GET

```bash
curl \
  -x http://127.0.0.1:8080 \
  http://10.10.20.20:8086/
```

## 自定义 Header

```bash
curl \
  -x http://127.0.0.1:8080 \
  -H 'X-Lab-Request: curl-burp' \
  http://10.10.20.20:8086/headers
```

## POST JSON

```bash
curl \
  -x http://127.0.0.1:8080 \
  -H 'Content-Type: application/json' \
  -d '{"student":"analyst"}' \
  http://10.10.20.20:8086/json
```

---

# 98. 核心记忆口诀

```text
Proxy 拦截流量
History 保存历史
Scope 限制范围
Site map 整理目标
Inspector 看结构
Repeater 反复修改
Decoder 编码解码
Comparer 比较差异
Intruder 小量自动替换
Sequencer 看 Token 随机性

页面卡住先看 Intercept
HTTPS 外部浏览器要 CA
Burp Browser 已经预配置
127.0.0.1 是 Kali 本机
```

---

# 99. 完成标准

完成本课后，你应该拥有：

```text
Ubuntu 8086 Burp 实验服务器
Burp Proxy Listener 截图
Intercept 修改请求截图
HTTP history 截图
Target Scope 截图
Site map 截图
至少 4 个 Repeater 请求结果
Decoder 实验记录
Comparer 实验记录
Intruder Demo 结果
curl-through-Burp 请求
Browser-to-Burp PCAP
Burp-to-Ubuntu PCAP
Ubuntu Access Log
Wazuh burp-lab 事件
脱敏后的 Raw Request/Response
完整实验报告
```

检查 Kali：

```bash
find ~/CyberLab/Burp -type f
```

检查 Ubuntu：

```bash
find ~/CyberLab/Burp -type f
```

---

# 100. 下一课

下一课建议：

```text
OWASP ZAP
```

学习内容：

- ZAP Proxy
- ZAP Browser
- Sites Tree
- History
- Breakpoints
- Manual Request Editor
- Passive Scan
- Scope/Context
- Community 工具对比
- Burp 与 ZAP 的区别
- Access Log、PCAP 和 Wazuh 观察

当前学习关系：

```text
Nmap：发现服务
Netcat：测试连接
Wireshark：图形化流量分析
tcpdump：命令行抓包
curl：手动 HTTP 请求
WhatWeb：技术指纹
Gobuster/ffuf：内容发现
Nikto：Web Server 检查
Burp Suite：拦截、修改和重放 HTTP
```

---

# 101. 参考说明

本指南依据：

```text
PortSwigger Burp Suite 官方文档
PortSwigger Community Edition 官方功能说明
PortSwigger Proxy、Repeater、Decoder、Comparer 和 Scope 文档
Kali Linux 官方 Burp Suite 工具页面
```

Burp 界面和功能会更新，最终以本机版本和官方文档为准。
