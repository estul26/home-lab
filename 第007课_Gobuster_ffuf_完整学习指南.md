# 第七课：Gobuster / ffuf 完整学习指南

> **实验环境：你自己的 CyberLab**
>
> 本课程只用于你自己创建、拥有并控制的 Web 服务。  
> 不要扫描学校、公司、公共网站、随机公网 IP 或任何未经明确授权的系统。

---

# 1. 本课目标

学完本课后，你应该能够：

```text
理解 Content Discovery、Directory Discovery 和 Web Fuzzing
理解 Gobuster 与 ffuf 的区别
创建小型、有针对性的 Wordlist
使用 Gobuster dir 发现目录和文件
使用 Gobuster vhost 发现虚拟主机
使用 Gobuster fuzz 测试自定义 FUZZ 位置
使用 ffuf 的 FUZZ 关键字
使用 ffuf 发现目录、文件、VHost 和参数
理解 HTTP Status、Size、Words、Lines 和 Regex
使用 Matcher 和 Filter 去除误报
识别和处理 Soft 404 / Wildcard Response
限制线程、速率、延迟和总运行时间
保存 TXT、JSON、HTML、CSV 和 Markdown 结果
使用 tcpdump、Wireshark 和 Access Log 分析请求
在 Wazuh 中确认实验日志链路
完成一份 Web Content Discovery 实验报告
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
Kali 10.10.20.10
→ HTTP 8083
→ Ubuntu 10.10.20.20
```

---

# 2. 什么是 Content Discovery？

浏览器访问网站首页时，只能看到开发者公开链接的内容。

服务器上可能还有：

```text
/admin/
/api/
/assets/
/private/
/nested/
/health
/hidden.txt
/backup.zip
```

这些路径可能没有出现在首页链接中。

Content Discovery 的作用是：

```text
把 Wordlist 中的每个词放进 URL
→ 向服务器发送请求
→ 根据状态码和响应特征
→ 找出可能存在的目录、文件或端点
```

例如 Wordlist：

```text
admin
api
backup
private
```

工具会测试：

```text
http://10.10.20.20:8083/admin
http://10.10.20.20:8083/api
http://10.10.20.20:8083/backup
http://10.10.20.20:8083/private
```

---

# 3. 什么是 Fuzzing？

在本课中，Fuzzing 表示：

```text
把一组输入依次替换到指定位置
观察响应差异
```

ffuf 使用关键字：

```text
FUZZ
```

例如：

```bash
ffuf \
  -w words.txt \
  -u http://10.10.20.20:8083/FUZZ
```

工具会把 `FUZZ` 替换为 Wordlist 中的每一行。

`FUZZ` 也可以放在：

- URL Path
- Query Parameter 名称
- Query Parameter 值
- HTTP Header
- Host Header
- POST Body
- JSON Body

---

# 4. Gobuster 与 ffuf 的区别

| 项目 | Gobuster | ffuf |
|---|---|---|
| 核心设计 | Mode-based discovery | 任意位置 FUZZ |
| 目录发现 | `gobuster dir` | `FUZZ` 放进 URL |
| VHost | `gobuster vhost` | Fuzz `Host` Header |
| 参数 Fuzz | `gobuster fuzz` | URL/Body 中放 `FUZZ` |
| Matcher/Filter | 状态码、长度等 | 状态码、大小、字数、行数、Regex、时间 |
| 自动校准 | 较有限 | `-ac` |
| 多 Wordlist | 较简单 | 支持自定义关键字和多种模式 |
| Raw Request | 版本功能需看帮助 | `-request` |
| 输出格式 | 主要文本 | JSON、HTML、Markdown、CSV 等 |
| 初学优势 | 命令直观 | 过滤能力强、位置灵活 |

简单记忆：

```text
Gobuster：模式清楚、适合目录和 VHost
ffuf：FUZZ 灵活、过滤功能更强
```

---

# 5. 这类工具的局限

发现：

```text
/admin/ [200]
```

并不代表：

```text
已经绕过认证
已经获得管理员权限
存在漏洞
可以继续攻击
```

返回：

```text
403 Forbidden
```

也很重要，因为它说明：

```text
路径可能真实存在
但当前请求没有访问权限
```

这些工具只能提供：

```text
路径和响应线索
```

后续仍需使用：

- curl
- 浏览器
- Burp Suite
- 服务器日志
- 源代码
- 配置文件

进行验证。

---

# 6. 检查和安装工具

Kali：

```bash
which gobuster
which ffuf
```

查看版本：

```bash
gobuster version
```

```bash
ffuf -V
```

查看帮助：

```bash
gobuster -h
gobuster dir -h
gobuster vhost -h
gobuster fuzz -h
```

```bash
ffuf -h
```

如果没有安装：

```bash
sudo apt update
sudo apt install gobuster ffuf
```

工具版本更新后，部分参数可能变化。

始终先检查：

```bash
gobuster dir -h
ffuf -h
```

---

# 7. 创建实验目录

Kali：

```bash
mkdir -p ~/CyberLab/Discovery/{wordlists,results,captures,requests,notes,reports}
```

Ubuntu：

```bash
mkdir -p ~/CyberLab/Discovery/{server,logs}
```

检查：

```bash
find ~/CyberLab/Discovery -maxdepth 2 -type d
```

---

# 8. 创建安全的小型 Wordlist

## 目录 Wordlist

Kali：

```bash
cat > ~/CyberLab/Discovery/wordlists/dirs.txt <<'EOF'
admin
api
assets
backup
health
hidden
index
missing
nested
private
status
uploads
EOF
```

## 文件名 Wordlist

```bash
cat > ~/CyberLab/Discovery/wordlists/files.txt <<'EOF'
backup
config
data
hidden
index
notes
robots
status
EOF
```

## VHost Wordlist

```bash
cat > ~/CyberLab/Discovery/wordlists/vhosts.txt <<'EOF'
admin
api
dev
files
staging
test
EOF
```

## 参数名 Wordlist

```bash
cat > ~/CyberLab/Discovery/wordlists/params.txt <<'EOF'
debug
format
id
mode
page
role
test
view
EOF
```

## 参数值 Wordlist

```bash
cat > ~/CyberLab/Discovery/wordlists/values.txt <<'EOF'
1
2
42
99
admin
auditor
guest
json
status
user
xml
EOF
```

查看：

```bash
wc -l ~/CyberLab/Discovery/wordlists/*.txt
```

---

# 9. 为什么先使用小 Wordlist？

大型 Wordlist 可能包含：

```text
几千
几万
几十万
```

个词。

这会造成：

- 大量 HTTP 请求
- Access Log 快速增长
- 更长扫描时间
- 服务器负载增加
- PCAP 文件变大
- 误报增加
- Wazuh 事件增多

本课先使用：

```text
10–20 个词
低线程
低速率
```

先理解结果，再扩大范围。

---

# 10. Ubuntu 实验服务器

下面的服务器提供：

```text
真实存在的目录和文件
301 Redirect
403 Forbidden
普通 404
固定长度 Soft 404
VHost
GET 参数名测试
GET 参数值测试
POST 表单值测试
嵌套目录
Access Log
```

在 Ubuntu：

```bash
nano ~/CyberLab/Discovery/server/discovery_lab.py
```

粘贴：

```python
#!/usr/bin/env python3

from http.server import BaseHTTPRequestHandler, HTTPServer
from urllib.parse import parse_qs, urlparse


HOST = "10.10.20.20"
PORT = 8083

SOFT_404_BODY = b"CYBERLAB SOFT 404 RESPONSE\n"
NORMAL_404_BODY = b"CyberLab resource not found\n"


def html(title: str, body: str) -> bytes:
    return (
        "<!doctype html>"
        "<html><head>"
        f"<title>{title}</title>"
        "</head><body>"
        f"<h1>{title}</h1>"
        f"<p>{body}</p>"
        "</body></html>"
    ).encode("utf-8")


class DiscoveryHandler(BaseHTTPRequestHandler):
    server_version = "CyberLabDiscovery/1.0"

    def send_content(
        self,
        status: int,
        body: bytes,
        content_type: str = "text/plain; charset=utf-8",
        extra_headers: dict[str, str] | None = None,
    ) -> None:
        self.send_response(status)
        self.send_header("Content-Type", content_type)
        self.send_header("Content-Length", str(len(body)))
        self.send_header("X-CyberLab", "Gobuster-FFUF-Lab")

        if extra_headers:
            for name, value in extra_headers.items():
                self.send_header(name, value)

        self.end_headers()

        if self.command != "HEAD":
            self.wfile.write(body)

    def handle_vhost(self, host_header: str) -> bool:
        hostname = host_header.split(":", 1)[0].lower()

        if hostname == "admin.lab.local":
            self.send_content(
                200,
                html("Admin Virtual Host", "Restricted admin lab host."),
                "text/html; charset=utf-8",
            )
            return True

        if hostname == "api.lab.local":
            self.send_content(
                200,
                b'{"service":"cyberlab-api","version":"1"}\n',
                "application/json",
            )
            return True

        if hostname == "dev.lab.local":
            self.send_content(
                403,
                b"Development virtual host is restricted\n",
            )
            return True

        return False

    def do_GET(self) -> None:
        host_header = self.headers.get("Host", "")

        if self.handle_vhost(host_header):
            return

        parsed = urlparse(self.path)
        path = parsed.path
        query = parse_qs(parsed.query)

        if path == "/":
            self.send_content(
                200,
                html(
                    "CyberLab Discovery",
                    "Controlled Gobuster and ffuf training server.",
                ),
                "text/html; charset=utf-8",
            )
            return

        if path == "/admin":
            self.send_response(301)
            self.send_header("Location", "/admin/")
            self.send_header("Content-Length", "0")
            self.end_headers()
            return

        if path == "/admin/":
            self.send_content(
                200,
                html("Admin Area", "Authentication would be required here."),
                "text/html; charset=utf-8",
            )
            return

        if path == "/api":
            self.send_response(301)
            self.send_header("Location", "/api/")
            self.send_header("Content-Length", "0")
            self.end_headers()
            return

        if path == "/api/":
            self.send_content(
                200,
                b'{"api":"online","version":"v1"}\n',
                "application/json",
            )
            return

        if path == "/assets":
            self.send_response(301)
            self.send_header("Location", "/assets/")
            self.send_header("Content-Length", "0")
            self.end_headers()
            return

        if path == "/assets/":
            self.send_content(
                200,
                html("Assets", "Static resources."),
                "text/html; charset=utf-8",
            )
            return

        if path == "/private" or path == "/private/":
            self.send_content(403, b"Forbidden private area\n")
            return

        if path == "/health":
            self.send_content(
                200,
                b'{"status":"healthy"}\n',
                "application/json",
            )
            return

        if path == "/hidden.txt":
            self.send_content(
                200,
                b"CyberLab hidden text file\n",
            )
            return

        if path == "/backup.zip":
            self.send_content(
                200,
                b"CYBERLAB TEST ZIP PLACEHOLDER\n",
                "application/zip",
            )
            return

        if path == "/robots.txt":
            self.send_content(
                200,
                b"User-agent: *\nDisallow: /private/\n",
            )
            return

        if path == "/nested":
            self.send_response(301)
            self.send_header("Location", "/nested/")
            self.send_header("Content-Length", "0")
            self.end_headers()
            return

        if path == "/nested/":
            self.send_content(
                200,
                html("Nested Area", "A second level exists."),
                "text/html; charset=utf-8",
            )
            return

        if path == "/nested/secret":
            self.send_response(301)
            self.send_header("Location", "/nested/secret/")
            self.send_header("Content-Length", "0")
            self.end_headers()
            return

        if path == "/nested/secret/":
            self.send_content(
                200,
                html("Nested Secret", "Controlled recursion result."),
                "text/html; charset=utf-8",
            )
            return

        if path.startswith("/soft/"):
            self.send_content(200, SOFT_404_BODY)
            return

        if path == "/params":
            valid_names = {"debug", "format", "view"}
            supplied = set(query.keys())
            matched = sorted(valid_names.intersection(supplied))

            if matched:
                body = (
                    "Valid parameter names: "
                    + ",".join(matched)
                    + "\n"
                ).encode("utf-8")
                self.send_content(200, body)
            else:
                self.send_content(
                    200,
                    b"NO VALID PARAMETER NAME\n",
                )
            return

        if path == "/item":
            item_id = query.get("id", [""])[0]

            if item_id == "1":
                self.send_content(
                    200,
                    b'{"id":1,"name":"alpha"}\n',
                    "application/json",
                )
                return

            if item_id == "42":
                self.send_content(
                    200,
                    b'{"id":42,"name":"answer"}\n',
                    "application/json",
                )
                return

            self.send_content(
                404,
                b"ITEM NOT FOUND\n",
            )
            return

        self.send_content(404, NORMAL_404_BODY)

    def do_HEAD(self) -> None:
        self.do_GET()

    def do_POST(self) -> None:
        parsed = urlparse(self.path)

        if parsed.path != "/form":
            self.send_content(404, NORMAL_404_BODY)
            return

        content_length = int(self.headers.get("Content-Length", "0"))
        raw_body = self.rfile.read(content_length)
        form = parse_qs(raw_body.decode("utf-8", errors="replace"))
        role = form.get("role", [""])[0]

        if role in {"admin", "auditor", "user"}:
            self.send_content(
                200,
                f"VALID ROLE: {role}\n".encode("utf-8"),
            )
            return

        self.send_content(
            403,
            b"INVALID ROLE\n",
        )

    def log_message(self, format: str, *args: object) -> None:
        user_agent = self.headers.get("User-Agent", "-")
        host_header = self.headers.get("Host", "-")
        print(
            f'{self.client_address[0]} '
            f'[{self.log_date_time_string()}] '
            f'Host="{host_header}" '
            f'UA="{user_agent}" '
            f'{format % args}',
            flush=True,
        )


def main() -> None:
    server = HTTPServer((HOST, PORT), DiscoveryHandler)
    print(f"Discovery lab listening on http://{HOST}:{PORT}")
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
chmod +x ~/CyberLab/Discovery/server/discovery_lab.py
```

---

# 11. 启动服务器并保存日志

Ubuntu：

```bash
python3 ~/CyberLab/Discovery/server/discovery_lab.py \
  2>&1 |
tee ~/CyberLab/Discovery/logs/access.log
```

预期：

```text
Discovery lab listening on http://10.10.20.20:8083
```

保持终端运行。

第二个 Ubuntu 终端：

```bash
ss -lntp | grep ':8083'
```

---

# 12. Kali 验证实验服务

```bash
ping -c 3 10.10.20.20
```

```bash
nc -zvn -w 3 10.10.20.20 8083
```

```bash
curl -i http://10.10.20.20:8083/
```

```bash
sudo nmap -n -sV -p 8083 10.10.20.20
```

测试几个路径：

```bash
curl -i http://10.10.20.20:8083/admin
```

```bash
curl -i http://10.10.20.20:8083/private/
```

```bash
curl -i http://10.10.20.20:8083/not-real
```

---

# 13. HTTP Status 在发现过程中的意义

| 状态码 | 解释 |
|---:|---|
| 200 | 资源返回成功 |
| 204 | 成功但没有正文 |
| 301 | 永久重定向，常见于目录补 `/` |
| 302 | 临时重定向 |
| 307/308 | 保留请求方法的重定向 |
| 401 | 需要认证 |
| 403 | 资源可能存在，但拒绝访问 |
| 404 | 资源通常不存在 |
| 405 | 路径存在，但方法不允许 |
| 500 | 服务器错误，需要谨慎验证 |

重要：

```text
403 也是有价值的结果
301 可能说明一个目录存在
200 不一定代表真正资源
```

---

# 14. 第一部分：Gobuster

Gobuster 使用模式：

```bash
gobuster [mode] [options]
```

本课使用：

```text
dir
vhost
fuzz
```

不在本课运行：

```text
dns
s3
gcs
tftp
```

原因是这些模式不适合当前隔离实验目标。

---

# 15. Gobuster dir 基础扫描

Kali：

```bash
gobuster dir \
  -u http://10.10.20.20:8083/ \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt
```

可能发现：

```text
/admin
/api
/assets
/health
/nested
/private
```

具体输出可能包括：

```text
Status
Size
Redirect
```

---

# 16. Gobuster 结果解释

示例：

```text
/admin    (Status: 301)
/private  (Status: 403)
/health   (Status: 200)
```

解释：

```text
/admin：
服务器要求跳转到 /admin/

/private：
路径可能存在，但访问被拒绝

/health：
端点返回成功
```

不要只记录 200。

---

# 17. 显示响应长度

```bash
gobuster dir \
  -u http://10.10.20.20:8083/ \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -l
```

响应长度可帮助识别：

- 相同错误页面
- Soft 404
- 重复模板
- 不同资源

---

# 18. 发现文件扩展名

```bash
gobuster dir \
  -u http://10.10.20.20:8083/ \
  -w ~/CyberLab/Discovery/wordlists/files.txt \
  -x txt,zip,json,html
```

Gobuster 会尝试类似：

```text
backup
backup.txt
backup.zip
backup.json
backup.html
```

本实验应该发现：

```text
hidden.txt
backup.zip
robots.txt
```

注意：

```text
扩展名越多
请求数量越多
```

---

# 19. 低线程扫描

```bash
gobuster dir \
  -u http://10.10.20.20:8083/ \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -t 5
```

本课推荐：

```text
-t 5
```

不要为了追求速度使用很高线程。

---

# 20. 添加请求延迟

```bash
gobuster dir \
  -u http://10.10.20.20:8083/ \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -t 2 \
  --delay 200ms
```

不同版本支持的 Duration 格式可能包括：

```text
200ms
1s
```

以本机：

```bash
gobuster dir -h
```

为准。

---

# 21. 自定义 User-Agent 和 Header

User-Agent：

```bash
gobuster dir \
  -u http://10.10.20.20:8083/ \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -a 'CyberLab-Gobuster/1.0'
```

自定义 Header：

```bash
gobuster dir \
  -u http://10.10.20.20:8083/ \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -H 'X-Lab-Name: Gobuster'
```

Cookie：

```bash
gobuster dir \
  -u http://10.10.20.20:8083/ \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -c 'cyberlab=student'
```

---

# 22. 状态码选择

只显示指定状态码：

```bash
gobuster dir \
  -u http://10.10.20.20:8083/ \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -s 200,301,302,401,403
```

Gobuster 版本可能要求：

```text
不要同时使用互相冲突的 Status Include 和 Exclude
```

如果命令提示冲突，查看：

```bash
gobuster dir -h
```

并只选择一种策略。

---

# 23. 排除状态码

某些版本支持使用状态黑名单：

```bash
gobuster dir \
  -u http://10.10.20.20:8083/ \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -b 404
```

含义：

```text
不显示 404
```

使用本机帮助确认：

```bash
gobuster dir -h
```

---

# 24. 跟随 Redirect

```bash
gobuster dir \
  -u http://10.10.20.20:8083/ \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -r
```

注意：

```text
跟随 Redirect 可能改变最终状态码和响应长度
```

报告中记录：

- 原始路径
- 原始状态码
- Location
- 最终 URL

---

# 25. 保存 Gobuster 结果

```bash
gobuster dir \
  -u http://10.10.20.20:8083/ \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -t 5 \
  -o ~/CyberLab/Discovery/results/gobuster-dir.txt
```

Quiet 模式：

```bash
gobuster dir \
  -u http://10.10.20.20:8083/ \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -q \
  -o ~/CyberLab/Discovery/results/gobuster-quiet.txt
```

---

# 26. Gobuster 完整基础命令

```bash
gobuster dir \
  -u http://10.10.20.20:8083/ \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -t 5 \
  --delay 100ms \
  -l \
  -a 'CyberLab-Gobuster/1.0' \
  -o ~/CyberLab/Discovery/results/gobuster-baseline.txt
```

---

# 27. Gobuster 与 Soft 404

正常不存在路径：

```bash
curl -i http://10.10.20.20:8083/random-invalid
```

返回：

```text
404
```

Soft 404 路径：

```bash
curl -i http://10.10.20.20:8083/soft/random-invalid
```

返回：

```text
200 OK
CYBERLAB SOFT 404 RESPONSE
```

这是 Soft 404：

```text
资源不存在
但服务器返回 200
```

---

# 28. Gobuster Soft 404 预检查

尝试：

```bash
gobuster dir \
  -u http://10.10.20.20:8083/soft/ \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -l
```

Gobuster 可能报告：

```text
服务器对随机不存在路径返回匹配的状态码
可能存在 Wildcard / Soft 404
```

不要直接忽略错误。

先测量基线。

---

# 29. 测量 Soft 404 长度

```bash
curl -s \
  http://10.10.20.20:8083/soft/not-real |
wc -c
```

本实验的 Soft 404 Body 是固定长度。

记录结果，例如：

```text
28
```

实际数字以终端为准。

---

# 30. Gobuster 排除响应长度

查看本机帮助：

```bash
gobuster dir -h |
grep -i length
```

然后使用：

```bash
gobuster dir \
  -u http://10.10.20.20:8083/soft/ \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  --exclude-length <SOFT_404_LENGTH>
```

把：

```text
<SOFT_404_LENGTH>
```

替换成刚才测量的数字。

不要复制示例数字而不测量。

---

# 31. Gobuster VHost 准备

在 Kali 添加实验域名：

```bash
echo '10.10.20.20 lab.local' |
sudo tee -a /etc/hosts
```

验证：

```bash
getent hosts lab.local
```

```bash
curl -i http://lab.local:8083/
```

不要添加公共域名。

---

# 32. Gobuster VHost

```bash
gobuster vhost \
  -u http://lab.local:8083/ \
  -w ~/CyberLab/Discovery/wordlists/vhosts.txt \
  --append-domain \
  -t 5
```

它会尝试类似：

```text
admin.lab.local
api.lab.local
dev.lab.local
```

本实验服务器应该对这些 Host 返回不同响应：

```text
admin.lab.local → 200
api.lab.local   → 200
dev.lab.local   → 403
```

---

# 33. VHost 与 DNS Subdomain 的区别

| DNS Subdomain | Virtual Host |
|---|---|
| 依赖 DNS 解析 | 依赖 HTTP Host Header |
| DNS 需要有记录 | 可能没有 DNS 记录 |
| 指向 IP | Web Server 根据 Host 选择站点 |
| `gobuster dns` | `gobuster vhost` |

本实验使用：

```text
VHost
```

不运行公共 DNS 枚举。

---

# 34. Gobuster fuzz 模式

Gobuster 新版本支持在 URL、Header 或 Body 中使用：

```text
FUZZ
```

参数值实验：

```bash
gobuster fuzz \
  -u 'http://10.10.20.20:8083/item?id=FUZZ' \
  -w ~/CyberLab/Discovery/wordlists/values.txt
```

Header 实验：

```bash
gobuster fuzz \
  -u http://10.10.20.20:8083/ \
  -H 'Host: FUZZ.lab.local' \
  -w ~/CyberLab/Discovery/wordlists/vhosts.txt
```

本机具体参数和过滤能力以：

```bash
gobuster fuzz -h
```

为准。

---

# 35. Gobuster 常见问题

## 所有结果都是 200

可能是：

```text
Soft 404
统一错误页面
反向代理默认页面
```

解决：

- curl 测试随机路径
- 比较 Response Length
- 使用 `--exclude-length`
- 使用 ffuf 的 `-fs`、`-fw` 或 `-fr`

## 大量 403

403 不一定是假结果。

检查：

```bash
curl -i URL
```

比较 Body 和 Length。

## 没有结果

检查：

```bash
curl -i TARGET
```

```bash
gobuster dir -h
```

确认：

- URL 正确
- Wordlist 正确
- Status Filter 没有排除真实结果
- 服务器正在运行
- 路径大小写正确

---

# 36. 第二部分：ffuf

ffuf 的核心是：

```text
把 Wordlist 词替换到 FUZZ
```

基本语法：

```bash
ffuf \
  -w WORDLIST \
  -u URL_WITH_FUZZ
```

---

# 37. ffuf 基础目录发现

```bash
ffuf \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/FUZZ
```

默认结果通常显示：

```text
Status
Size
Words
Lines
Duration
```

这些字段是 ffuf 过滤误报的核心。

---

# 38. ffuf 彩色与 Verbose 输出

彩色：

```bash
ffuf \
  -c \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/FUZZ
```

Verbose：

```bash
ffuf \
  -v \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/FUZZ
```

Silent：

```bash
ffuf \
  -s \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/FUZZ
```

---

# 39. ffuf 文件扩展名

```bash
ffuf \
  -w ~/CyberLab/Discovery/wordlists/files.txt \
  -u http://10.10.20.20:8083/FUZZ \
  -e .txt,.zip,.json,.html
```

应该发现：

```text
hidden.txt
backup.zip
robots.txt
```

`-e` 会在每个词后测试扩展名。

---

# 40. ffuf Matcher 与 Filter

ffuf 内部逻辑：

```text
发送请求
→ Matcher 判断是否保留
→ Filter 判断是否排除
→ 显示剩余结果
```

Matcher：

```text
想保留什么
```

Filter：

```text
想隐藏什么
```

---

# 41. Status Code Matcher

只保留：

```text
200、301、302、401、403
```

命令：

```bash
ffuf \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/FUZZ \
  -mc 200,301,302,401,403
```

范围：

```bash
ffuf \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/FUZZ \
  -mc 200-299,301,302,401,403
```

匹配所有：

```bash
-mc all
```

---

# 42. Status Code Filter

排除 404：

```bash
ffuf \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/FUZZ \
  -mc all \
  -fc 404
```

多个值：

```bash
-fc 400,404
```

---

# 43. Response Size

`Size` 通常表示响应 Body 大小。

匹配大小：

```bash
-ms 123
```

排除大小：

```bash
-fs 123
```

多个值和范围：

```bash
-fs 20,30,100-120
```

实际数值必须先测量。

---

# 44. Words 和 Lines

匹配字数：

```bash
-mw 5
```

过滤字数：

```bash
-fw 5
```

匹配行数：

```bash
-ml 2
```

过滤行数：

```bash
-fl 2
```

当错误页面会反射路径，导致 Size 变化时：

```text
Words 或 Lines 可能比 Size 更稳定
```

---

# 45. Regex Matcher 与 Filter

保留包含指定文字：

```bash
ffuf \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/FUZZ \
  -mr 'CyberLab'
```

排除包含：

```bash
ffuf \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/FUZZ \
  -fr 'resource not found'
```

Regex 会针对：

```text
Response Headers + Body
```

进行匹配。

---

# 46. 自动校准

```bash
ffuf \
  -ac \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/FUZZ
```

`-ac` 会尝试自动识别常见错误响应特征并建立过滤。

自动校准很方便，但仍要人工验证：

```text
过滤了什么？
是否误过滤真实结果？
```

---

# 47. ffuf Soft 404 实验

基础扫描：

```bash
ffuf \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/soft/FUZZ \
  -mc all
```

你可能看到每个词都是：

```text
200
相同 Size
相同 Words
相同 Lines
```

这不是所有路径都真实存在。

这是 Soft 404。

---

# 48. 测量 Soft 404 基线

```bash
curl -s \
  http://10.10.20.20:8083/soft/random-one |
wc -c
```

再测试：

```bash
curl -s \
  http://10.10.20.20:8083/soft/random-two |
wc -c
```

如果长度相同，记录该 Size。

---

# 49. 使用 -fs 排除 Soft 404

```bash
ffuf \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/soft/FUZZ \
  -mc all \
  -fs <SOFT_404_SIZE>
```

把占位符替换为实际 Size。

也可以：

```bash
ffuf \
  -ac \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/soft/FUZZ
```

---

# 50. ffuf 递归

实验服务器有：

```text
/nested/
/nested/secret/
```

命令：

```bash
ffuf \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/FUZZ \
  -recursion \
  -recursion-depth 2 \
  -t 5
```

注意：

```text
递归要求 URL 的结尾是 FUZZ
```

递归会明显增加请求数量。

本课限制：

```text
Depth 2
小 Wordlist
低线程
```

---

# 51. 递归策略

默认策略：

```bash
-recursion-strategy default
```

通常根据 Redirect 发现目录。

Greedy：

```bash
-recursion-strategy greedy
```

会对更多匹配结果递归。

本课优先使用：

```text
default
```

不要在大型 Wordlist 上使用 Greedy。

---

# 52. ffuf VHost 发现

先测量默认 Host 响应：

```bash
curl -s \
  -H 'Host: invalid.lab.local' \
  http://10.10.20.20:8083/ |
wc -c
```

记录默认响应 Size。

然后：

```bash
ffuf \
  -w ~/CyberLab/Discovery/wordlists/vhosts.txt \
  -u http://10.10.20.20:8083/ \
  -H 'Host: FUZZ.lab.local' \
  -mc all \
  -fs <DEFAULT_VHOST_SIZE> \
  -t 5
```

应该发现：

```text
admin
api
dev
```

对应：

```text
admin.lab.local
api.lab.local
dev.lab.local
```

---

# 53. 为什么 VHost 需要 Size Filter？

很多服务器对不存在的 Host 返回：

```text
相同状态码
相同默认网页
```

例如所有无效 Host 都返回：

```text
200
默认首页
```

仅根据状态码会产生误报。

所以要比较：

- Size
- Words
- Lines
- Title
- 特定文字

---

# 54. ffuf GET 参数名发现

实验端点：

```text
/params?PARAMETER=test
```

有效参数名：

```text
debug
format
view
```

先测量无效参数响应：

```bash
curl -s \
  'http://10.10.20.20:8083/params?invalid=test' |
wc -c
```

然后：

```bash
ffuf \
  -w ~/CyberLab/Discovery/wordlists/params.txt \
  -u 'http://10.10.20.20:8083/params?FUZZ=test' \
  -mc all \
  -fs <INVALID_PARAM_SIZE> \
  -t 5
```

或使用 Regex：

```bash
ffuf \
  -w ~/CyberLab/Discovery/wordlists/params.txt \
  -u 'http://10.10.20.20:8083/params?FUZZ=test' \
  -mr 'Valid parameter names'
```

---

# 55. ffuf GET 参数值发现

端点：

```text
/item?id=VALUE
```

有效值：

```text
1
42
```

命令：

```bash
ffuf \
  -w ~/CyberLab/Discovery/wordlists/values.txt \
  -u 'http://10.10.20.20:8083/item?id=FUZZ' \
  -mc 200 \
  -t 5
```

这里只进行：

```text
测试资源 ID 的内容差异
```

不要把它理解为授权绕过。

---

# 56. ffuf POST 表单值发现

端点：

```text
POST /form
role=VALUE
```

有效实验值：

```text
admin
auditor
user
```

命令：

```bash
ffuf \
  -w ~/CyberLab/Discovery/wordlists/values.txt \
  -u http://10.10.20.20:8083/form \
  -X POST \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'role=FUZZ' \
  -mc 200 \
  -t 5
```

这是受控参数测试。

不是密码攻击。

---

# 57. ffuf Header Fuzzing

测试实验 VHost：

```bash
ffuf \
  -w ~/CyberLab/Discovery/wordlists/vhosts.txt \
  -u http://10.10.20.20:8083/ \
  -H 'Host: FUZZ.lab.local'
```

FUZZ 也可以放在自定义 Header：

```bash
ffuf \
  -w ~/CyberLab/Discovery/wordlists/values.txt \
  -u http://10.10.20.20:8083/ \
  -H 'X-Lab-Value: FUZZ'
```

当前服务器不会因 `X-Lab-Value` 改变响应，所以这是观察请求格式的实验。

---

# 58. 自定义 User-Agent 和 Cookie

User-Agent：

```bash
ffuf \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/FUZZ \
  -H 'User-Agent: CyberLab-ffuf/1.0'
```

Cookie：

```bash
ffuf \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/FUZZ \
  -b 'cyberlab=student; lesson=ffuf'
```

---

# 59. 限制 ffuf 线程

ffuf 默认线程可能较高。

本实验使用：

```bash
-t 5
```

示例：

```bash
ffuf \
  -t 5 \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/FUZZ
```

不要把线程设置得很高。

---

# 60. 限制请求速率

每秒最多 5 个请求：

```bash
ffuf \
  -rate 5 \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/FUZZ
```

这比只控制线程更直观。

本课推荐：

```text
-rate 5
```

---

# 61. 请求延迟

固定延迟：

```bash
ffuf \
  -p 0.2 \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/FUZZ
```

随机范围：

```bash
ffuf \
  -p 0.1-0.5 \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/FUZZ
```

本课使用延迟是为了：

```text
降低负载
方便观察日志
```

不是为了躲避检测。

---

# 62. 限制总运行时间

```bash
ffuf \
  -maxtime 60 \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/FUZZ
```

含义：

```text
整个任务最多运行 60 秒
```

递归单任务限制：

```bash
ffuf \
  -maxtime-job 30 \
  -recursion \
  -recursion-depth 2 \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/FUZZ
```

---

# 63. ffuf Timeout

```bash
ffuf \
  -timeout 5 \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/FUZZ
```

每个 HTTP 请求最多等待约 5 秒。

---

# 64. ffuf 完整低负载命令

```bash
ffuf \
  -c \
  -t 5 \
  -rate 5 \
  -timeout 5 \
  -maxtime 60 \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/FUZZ \
  -mc 200,301,302,401,403
```

---

# 65. ffuf 保存 JSON

```bash
ffuf \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/FUZZ \
  -o ~/CyberLab/Discovery/results/ffuf-dir.json \
  -of json
```

格式化查看：

```bash
python3 -m json.tool \
  ~/CyberLab/Discovery/results/ffuf-dir.json |
less
```

---

# 66. 保存 HTML

```bash
ffuf \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/FUZZ \
  -o ~/CyberLab/Discovery/results/ffuf-dir.html \
  -of html
```

在 Kali 浏览器打开：

```bash
xdg-open ~/CyberLab/Discovery/results/ffuf-dir.html
```

---

# 67. 保存 Markdown

```bash
ffuf \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/FUZZ \
  -o ~/CyberLab/Discovery/results/ffuf-dir.md \
  -of md
```

Markdown 很适合加入 Git 实验记录。

检查内容后再提交，避免泄露真实目标或敏感数据。

---

# 68. 保存 CSV

```bash
ffuf \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/FUZZ \
  -o ~/CyberLab/Discovery/results/ffuf-dir.csv \
  -of csv
```

---

# 69. 保存所有格式

```bash
ffuf \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/FUZZ \
  -o ~/CyberLab/Discovery/results/ffuf-all \
  -of all
```

`-of all` 会生成多种格式，文件数量较多。

---

# 70. Raw HTTP Request

创建请求：

```bash
cat > ~/CyberLab/Discovery/requests/raw-request.txt <<'EOF'
GET /FUZZ HTTP/1.1
Host: 10.10.20.20:8083
User-Agent: CyberLab-Raw-ffuf/1.0
Accept: */*
Connection: close

EOF
```

因为这是 HTTP：

```bash
ffuf \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -request ~/CyberLab/Discovery/requests/raw-request.txt \
  -request-proto http
```

Raw Request 适合以后与 Burp Suite 配合。

---

# 71. 多 Wordlist

ffuf 可以为 Wordlist 指定自定义关键字：

```bash
-w wordlist.txt:KEYWORD
```

示例：

```bash
ffuf \
  -w ~/CyberLab/Discovery/wordlists/vhosts.txt:HOST \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt:PATH \
  -u http://10.10.20.20:8083/PATH \
  -H 'Host: HOST.lab.local' \
  -mode clusterbomb \
  -t 3 \
  -rate 5
```

`clusterbomb` 会尝试组合。

由于请求数相乘，本课只使用小 Wordlist。

---

# 72. Pitchfork 模式

```bash
ffuf \
  -w ~/CyberLab/Discovery/wordlists/vhosts.txt:HOST \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt:PATH \
  -u http://10.10.20.20:8083/PATH \
  -H 'Host: HOST.lab.local' \
  -mode pitchfork \
  -t 3
```

Pitchfork：

```text
第一行与第一行配对
第二行与第二行配对
```

Clusterbomb：

```text
尝试所有组合
```

---

# 73. Matcher 组合

保留：

```text
Status 200
并且 Body 包含 Admin
```

命令：

```bash
ffuf \
  -w ~/CyberLab/Discovery/wordlists/vhosts.txt \
  -u http://10.10.20.20:8083/ \
  -H 'Host: FUZZ.lab.local' \
  -mc 200 \
  -mr 'Admin' \
  -mmode and
```

默认 Matcher 组合模式通常是：

```text
or
```

`and` 要求所有 Matcher 同时满足。

---

# 74. Filter 组合

过滤条件也可以使用：

```bash
-fmode and
```

使用多个 Filter 前，先理解：

```text
or：满足任何一个就过滤
and：全部满足才过滤
```

错误组合可能隐藏真实结果。

---

# 75. ffuf Auto Calibration 的检查

运行：

```bash
ffuf \
  -ac \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/FUZZ
```

然后人工抽查：

```bash
curl -i http://10.10.20.20:8083/admin
```

```bash
curl -i http://10.10.20.20:8083/random-invalid
```

不要完全依赖自动过滤。

---

# 76. Gobuster 与 ffuf 结果验证

任何有价值结果都用 curl 复查：

```bash
curl -i http://10.10.20.20:8083/admin
```

```bash
curl -i http://10.10.20.20:8083/private/
```

```bash
curl -i http://10.10.20.20:8083/hidden.txt
```

VHost：

```bash
curl -i \
  -H 'Host: admin.lab.local' \
  http://10.10.20.20:8083/
```

参数：

```bash
curl -i \
  'http://10.10.20.20:8083/item?id=42'
```

---

# 77. 使用 tcpdump 观察请求

Kali 终端一：

```bash
sudo tcpdump \
  -i eth0 \
  -nn \
  -A \
  'host 10.10.20.20 and tcp port 8083'
```

终端二运行低速 ffuf：

```bash
ffuf \
  -t 1 \
  -rate 2 \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/FUZZ
```

tcpdump 中可以看到：

```text
GET /admin HTTP/1.1
GET /api HTTP/1.1
GET /private HTTP/1.1
User-Agent
Host
Status
Location
```

---

# 78. 保存 Gobuster PCAP

```bash
sudo tcpdump \
  -i eth0 \
  -nn \
  -s 0 \
  -w ~/CyberLab/Discovery/captures/gobuster-dir.pcap \
  'host 10.10.20.20 and tcp port 8083'
```

另一个终端：

```bash
gobuster dir \
  -u http://10.10.20.20:8083/ \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -t 2 \
  --delay 200ms
```

停止抓包：

```text
Ctrl + C
```

---

# 79. 保存 ffuf PCAP

```bash
sudo tcpdump \
  -i eth0 \
  -nn \
  -s 0 \
  -w ~/CyberLab/Discovery/captures/ffuf-dir.pcap \
  'host 10.10.20.20 and tcp port 8083'
```

另一个终端：

```bash
ffuf \
  -t 2 \
  -rate 5 \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/FUZZ
```

---

# 80. Wireshark 分析

打开：

```bash
wireshark \
  ~/CyberLab/Discovery/captures/ffuf-dir.pcap
```

过滤：

```text
http.request
```

只看 GET：

```text
http.request.method == "GET"
```

目标：

```text
ip.addr == 10.10.20.20 &&
tcp.port == 8083
```

状态码：

```text
http.response.code == 200
```

```text
http.response.code == 301
```

```text
http.response.code == 403
```

---

# 81. Wireshark 统计请求

打开：

```text
Statistics
→ HTTP
→ Requests
```

还可以：

```text
Statistics
→ Conversations
```

比较：

| 项目 | Gobuster | ffuf |
|---|---:|---:|
| HTTP 请求数 |  |  |
| TCP 会话数 |  |  |
| 总数据包 |  |  |
| 扫描时间 |  |  |
| 发现结果 |  |  |

---

# 82. Ubuntu Access Log

服务器启动时已经保存：

```text
~/CyberLab/Discovery/logs/access.log
```

查看：

```bash
less ~/CyberLab/Discovery/logs/access.log
```

只看 Kali：

```bash
grep '10.10.20.10' \
  ~/CyberLab/Discovery/logs/access.log
```

只看 ffuf：

```bash
grep -i 'ffuf' \
  ~/CyberLab/Discovery/logs/access.log
```

只看 Gobuster：

```bash
grep -i 'gobuster' \
  ~/CyberLab/Discovery/logs/access.log
```

统计请求：

```bash
grep -c '"GET ' \
  ~/CyberLab/Discovery/logs/access.log
```

---

# 83. Wazuh 日志链路

Python Access Log 默认不一定被 Wazuh Agent 收集。

在 Ubuntu 实验前：

```bash
logger -t discovery-lab \
  "Starting authorized Gobuster and ffuf experiment from Kali 10.10.20.10"
```

运行实验后：

```bash
logger -t discovery-lab \
  "Completed authorized Web content discovery against TCP 8083"
```

Ubuntu 本地查看：

```bash
journalctl \
  -t discovery-lab \
  --since "15 minutes ago"
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
discovery-lab
```

---

# 84. 是否一定产生 Wazuh 告警？

不一定。

Gobuster 和 ffuf 产生 HTTP 请求。

Wazuh 要自动检测这种活动，通常需要：

- Web Access Log
- Wazuh 收集该日志
- 对请求频率或路径的规则
- WAF
- IDS/IPS
- Suricata
- Zeek

本课成功标准：

```text
服务器收到请求
Access Log 有记录
PCAP 有流量
Wazuh 能看到手工标记日志
```

---

# 85. Soft 404 识别流程

发现很多相同结果时：

```text
1. 随机生成一个不存在路径
2. 使用 curl 查看 Status
3. 测量 Size
4. 测量 Words
5. 测量 Lines
6. 检查是否反射输入
7. Gobuster 使用 exclude-length
8. ffuf 使用 -fs/-fw/-fl/-fr 或 -ac
9. 人工复查剩余结果
```

---

# 86. Wildcard 与 Soft 404

## Soft 404

```text
不存在的路径返回 200 和错误页面
```

## Wildcard Response

```text
很多任意输入都返回相同或近似响应
```

可能出现在：

- Web Path
- DNS
- VHost
- API Gateway
- Reverse Proxy

它们都会导致：

```text
大量假阳性
```

---

# 87. 403 的处理

发现 403：

```text
/private/ [403]
```

正确操作：

```text
记录路径
使用 curl 确认
比较响应 Header 和 Body
检查服务器端配置
```

不要尝试：

```text
未授权绕过
路径编码绕过
Header 绕过
访问控制规避
```

本课只学习识别和记录。

---

# 88. 301/302 的处理

发现 Redirect：

```text
/admin [301] → /admin/
```

检查：

```bash
curl -i http://10.10.20.20:8083/admin
```

跟随：

```bash
curl -i -L http://10.10.20.20:8083/admin
```

记录：

```text
初始状态
Location
最终状态
最终路径
```

---

# 89. Wordlist 选择原则

优先级：

```text
1. 自己根据目标业务创建小 Wordlist
2. 根据已知技术选择专用 Wordlist
3. 使用常见小型列表
4. 最后才考虑大型列表
```

例如发现 API 后，创建：

```text
v1
v2
health
status
users
items
```

比盲目跑几十万词更有效。

---

# 90. SecLists

Kali 可能已经安装部分 Wordlist。

检查：

```bash
ls /usr/share/seclists 2>/dev/null
```

如果没有：

```bash
sudo apt install seclists
```

常见 Web Content 路径可能位于：

```text
/usr/share/seclists/Discovery/Web-Content/
```

不要直接使用大型列表。

先查看行数：

```bash
wc -l WORDLIST
```

只在自己的实验服务器上使用。

---

# 91. 从大型 Wordlist 取前 50 行

```bash
head -n 50 \
  /usr/share/seclists/Discovery/Web-Content/common.txt \
  > ~/CyberLab/Discovery/wordlists/common-first-50.txt
```

检查：

```bash
wc -l \
  ~/CyberLab/Discovery/wordlists/common-first-50.txt
```

然后低速运行。

---

# 92. 扫描负载估算

Gobuster 扩展名：

```text
Wordlist 100 行
扩展名 4 个
原词 + 4 个扩展
约 500 个请求
```

ffuf 多 Wordlist Clusterbomb：

```text
Host List 10 行
Path List 100 行
约 1,000 个组合
```

递归会进一步增加。

运行前先估算请求数。

---

# 93. 中止扫描

Gobuster 和 ffuf 都可以使用：

```text
Ctrl + C
```

立即停止。

ffuf 还支持交互式控制台，但本课不依赖该功能。

看到以下情况应停止：

- CPU 或内存异常
- Access Log 快速爆增
- 大量错误
- 目标响应变慢
- 过滤明显错误
- 扫描范围不正确

---

# 94. 完整实验一：Gobuster 目录和文件

## 目录

```bash
gobuster dir \
  -u http://10.10.20.20:8083/ \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -t 5 \
  -l \
  -o ~/CyberLab/Discovery/results/lab1-gobuster-dirs.txt
```

## 文件

```bash
gobuster dir \
  -u http://10.10.20.20:8083/ \
  -w ~/CyberLab/Discovery/wordlists/files.txt \
  -x txt,zip,json,html \
  -t 5 \
  -l \
  -o ~/CyberLab/Discovery/results/lab1-gobuster-files.txt
```

## 验证

```bash
curl -i http://10.10.20.20:8083/hidden.txt
```

```bash
curl -i http://10.10.20.20:8083/backup.zip
```

---

# 95. 完整实验二：ffuf 目录和文件

目录：

```bash
ffuf \
  -c \
  -t 5 \
  -rate 5 \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/FUZZ \
  -mc 200,301,302,401,403 \
  -o ~/CyberLab/Discovery/results/lab2-ffuf-dirs.json \
  -of json
```

文件：

```bash
ffuf \
  -c \
  -t 5 \
  -rate 5 \
  -w ~/CyberLab/Discovery/wordlists/files.txt \
  -u http://10.10.20.20:8083/FUZZ \
  -e .txt,.zip,.json,.html \
  -mc 200,301,302,401,403 \
  -o ~/CyberLab/Discovery/results/lab2-ffuf-files.html \
  -of html
```

---

# 96. 完整实验三：VHost

Kali `/etc/hosts`：

```bash
echo '10.10.20.20 lab.local' |
sudo tee -a /etc/hosts
```

Gobuster：

```bash
gobuster vhost \
  -u http://lab.local:8083/ \
  -w ~/CyberLab/Discovery/wordlists/vhosts.txt \
  --append-domain \
  -t 5 \
  -o ~/CyberLab/Discovery/results/lab3-gobuster-vhost.txt
```

ffuf：

```bash
BASELINE_SIZE="$(
  curl -s \
    -H 'Host: invalid.lab.local' \
    http://10.10.20.20:8083/ |
  wc -c
)"

ffuf \
  -t 5 \
  -rate 5 \
  -w ~/CyberLab/Discovery/wordlists/vhosts.txt \
  -u http://10.10.20.20:8083/ \
  -H 'Host: FUZZ.lab.local' \
  -mc all \
  -fs "$BASELINE_SIZE" \
  -o ~/CyberLab/Discovery/results/lab3-ffuf-vhost.json \
  -of json
```

---

# 97. 完整实验四：GET 和 POST 参数

参数名：

```bash
INVALID_SIZE="$(
  curl -s \
    'http://10.10.20.20:8083/params?invalid=test' |
  wc -c
)"

ffuf \
  -t 5 \
  -w ~/CyberLab/Discovery/wordlists/params.txt \
  -u 'http://10.10.20.20:8083/params?FUZZ=test' \
  -mc all \
  -fs "$INVALID_SIZE"
```

GET 值：

```bash
ffuf \
  -t 5 \
  -w ~/CyberLab/Discovery/wordlists/values.txt \
  -u 'http://10.10.20.20:8083/item?id=FUZZ' \
  -mc 200
```

POST 值：

```bash
ffuf \
  -t 5 \
  -w ~/CyberLab/Discovery/wordlists/values.txt \
  -u http://10.10.20.20:8083/form \
  -X POST \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'role=FUZZ' \
  -mc 200
```

---

# 98. 完整实验五：Soft 404

测量：

```bash
SOFT_SIZE="$(
  curl -s \
    http://10.10.20.20:8083/soft/not-real |
  wc -c
)"

echo "$SOFT_SIZE"
```

未过滤：

```bash
ffuf \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/soft/FUZZ \
  -mc all
```

过滤：

```bash
ffuf \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/soft/FUZZ \
  -mc all \
  -fs "$SOFT_SIZE"
```

自动校准：

```bash
ffuf \
  -ac \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/soft/FUZZ
```

---

# 99. 完整实验六：网络与日志分析

Ubuntu 标记开始：

```bash
logger -t discovery-lab \
  "Starting controlled discovery traffic capture"
```

Kali 抓包：

```bash
sudo tcpdump \
  -i eth0 \
  -nn \
  -s 0 \
  -w ~/CyberLab/Discovery/captures/lab6-discovery.pcap \
  'host 10.10.20.20 and tcp port 8083'
```

另一个 Kali 终端：

```bash
ffuf \
  -t 2 \
  -rate 2 \
  -w ~/CyberLab/Discovery/wordlists/dirs.txt \
  -u http://10.10.20.20:8083/FUZZ
```

停止 tcpdump。

Ubuntu：

```bash
logger -t discovery-lab \
  "Completed controlled discovery traffic capture"
```

分析：

```bash
wireshark \
  ~/CyberLab/Discovery/captures/lab6-discovery.pcap
```

Wazuh 搜索：

```text
discovery-lab
```

---

# 100. 实验报告模板

## Lab Information

```text
Lab Name：Gobuster and ffuf Content Discovery
Date：
Student：
Scanner：Kali-CyberLab 10.10.20.10
Target：Ubuntu-Target 10.10.20.20
Service：http://10.10.20.20:8083/
Authorization：Personal CyberLab
```

## Objective

```text
使用 Gobuster 和 ffuf 发现目录、文件、VHost 和参数，
并使用 curl、tcpdump、Wireshark、Access Log 和 Wazuh 验证。
```

## Wordlists

| Wordlist | Lines | Purpose |
|---|---:|---|
| dirs.txt |  | Directory |
| files.txt |  | File |
| vhosts.txt |  | VHost |
| params.txt |  | Parameter names |
| values.txt |  | Parameter values |

## Commands

```bash
# Gobuster dir
# Gobuster vhost
# ffuf directory
# ffuf vhost
# ffuf parameter
```

## Findings

| Type | Value | Status | Size | Evidence | Confidence |
|---|---|---:|---:|---|---|
| Directory |  |  |  |  |  |
| File |  |  |  |  |  |
| VHost |  |  |  |  |  |
| Parameter |  |  |  |  |  |

## False Positive Analysis

```text
Soft 404 Status：
Soft 404 Size：
Soft 404 Words：
Filter used：
Results removed：
Manual verification：
```

## Traffic Statistics

| Tool | Requests | Duration | Packets | Bytes |
|---|---:|---:|---:|---:|
| Gobuster |  |  |  |  |
| ffuf |  |  |  |  |

## Evidence

```text
Gobuster Result：
ffuf JSON：
ffuf HTML：
curl Output：
Access Log：
PCAP：
Wireshark Screenshot：
Wazuh Event：
```

## Security Observations

```text
哪些路径不应该公开？
403 是否正确？
是否暴露备份文件？
是否存在没有链接的管理页面？
是否存在多余 VHost？
```

## Recommendations

```text
删除不需要的备份文件。
对管理页面实施认证和网络限制。
不要把敏感信息仅依赖“隐藏路径”保护。
配置正确的 404。
监控高频路径探测。
保持 Web Access Log。
```

## Lessons Learned

```text
Gobuster 与 ffuf 的区别：
最有效的过滤器：
Soft 404 的识别方法：
Wordlist 选择经验：
```

---

# 101. 常见错误与解决方法

## Gobuster/ffuf command not found

```bash
sudo apt update
sudo apt install gobuster ffuf
```

## Connection refused

```bash
nc -zvn -w 3 10.10.20.20 8083
```

Ubuntu：

```bash
ss -lntp | grep ':8083'
```

## 所有结果都是 200

检查 Soft 404：

```bash
curl -i http://10.10.20.20:8083/random-invalid
```

```bash
curl -i http://10.10.20.20:8083/soft/random-invalid
```

使用 Size/Words/Regex Filter。

## Gobuster 报 Wildcard

不要直接强制继续。

先：

```text
测量随机路径
确定错误响应特征
使用 exclude-length
```

## ffuf 没有结果

检查：

- `FUZZ` 是否存在
- Wordlist 路径
- `-mc` 是否过严
- `-fc/-fs/-fw/-fl/-fr` 是否过滤了真实结果
- URL 是否正确
- 大小写

## VHost 全部命中

测量无效 Host 的基线 Size，并使用：

```text
-fs
-fw
-fr
```

## 递归请求太多

停止：

```text
Ctrl + C
```

然后降低：

```text
-recursion-depth
Wordlist 大小
-t
-rate
```

## Output 文件无法创建

```bash
mkdir -p ~/CyberLab/Discovery/results
```

检查权限：

```bash
ls -ld ~/CyberLab/Discovery/results
```

---

# 102. 安全边界

本课只允许测试：

```text
http://10.10.20.20:8083/
Host: *.lab.local
```

不要测试：

- 学校或公司网站
- 公共 API
- 随机公网 IP
- 邻居设备
- 不属于你的云服务器
- 真实登录系统
- 大型互联网范围

不要进行：

- 密码爆破
- 认证绕过
- WAF 绕过
- 403 绕过
- 隐藏来源
- 规避检测
- 超高并发

---

# 103. 本课练习

1. 检查 Gobuster 和 ffuf 版本。
2. 查看两个工具的帮助。
3. 创建五个小型 Wordlist。
4. 启动 Ubuntu 8083 实验服务器。
5. 使用 Gobuster 发现目录。
6. 使用 Gobuster 发现带扩展名文件。
7. 保存 Gobuster 输出。
8. 使用 ffuf 发现目录。
9. 使用 ffuf 发现文件。
10. 比较 200、301 和 403。
11. 使用 curl 验证所有发现。
12. 测量正常 404。
13. 测量 Soft 404。
14. 使用 Gobuster 排除长度。
15. 使用 ffuf `-fs` 排除 Soft 404。
16. 使用 ffuf `-ac`。
17. 使用 Gobuster 发现 VHost。
18. 使用 ffuf 发现 VHost。
19. 使用 ffuf 发现 GET 参数名。
20. 使用 ffuf 发现 GET 参数值。
21. 使用 ffuf 测试 POST role 值。
22. 使用 ffuf Depth 2 递归。
23. 使用 `-t 5`。
24. 使用 `-rate 5`。
25. 使用 `-maxtime 60`。
26. 保存 JSON、HTML 和 Markdown 输出。
27. 创建并使用 Raw Request。
28. 使用 tcpdump 保存 PCAP。
29. 使用 Wireshark 统计请求。
30. 查看 Ubuntu Access Log。
31. 在 Wazuh 搜索 `discovery-lab`。
32. 完成实验报告。

---

# 104. 复习题

1. 什么是 Content Discovery？
2. Gobuster 和 ffuf 的主要区别是什么？
3. ffuf 的 `FUZZ` 有什么作用？
4. 200 是否一定代表真实资源？
5. 403 为什么也是有价值的结果？
6. 301 常见于什么情况？
7. 什么是 Soft 404？
8. 什么是 Wildcard Response？
9. Gobuster `dir` 模式做什么？
10. Gobuster `vhost` 模式做什么？
11. `-x` 在 Gobuster 中常用于什么？
12. `-l` 在 Gobuster 中做什么？
13. ffuf `-mc` 是什么？
14. ffuf `-fc` 是什么？
15. ffuf `-fs` 是什么？
16. ffuf `-fw` 是什么？
17. ffuf `-fl` 是什么？
18. ffuf `-mr` 与 `-fr` 有什么区别？
19. `-ac` 做什么？
20. `-e` 做什么？
21. `-recursion-depth` 为什么要限制？
22. `-t` 做什么？
23. `-rate` 做什么？
24. `-p` 做什么？
25. `-maxtime` 做什么？
26. VHost 与 DNS Subdomain 有什么区别？
27. 为什么 VHost 常需要 Size Filter？
28. 为什么先使用小 Wordlist？
29. curl 如何验证发现结果？
30. tcpdump/Wireshark 可以证明什么？
31. Wazuh 为什么不一定自动告警？
32. 为什么不能对未授权网站运行这些工具？

---

# 105. 参考答案

1. 使用 Wordlist 请求未链接的路径或端点。
2. Gobuster 按模式工作；ffuf 可在任意位置放 FUZZ 并有丰富过滤器。
3. 标记要被 Wordlist 输入替换的位置。
4. 不一定，可能是 Soft 404。
5. 可能说明资源存在但被访问控制保护。
6. 目录补斜杠或页面跳转。
7. 不存在资源却返回 200 的错误页面。
8. 任意输入都返回相同或近似响应。
9. 目录和文件发现。
10. 通过 Host Header 发现虚拟主机。
11. 添加文件扩展名。
12. 显示响应长度。
13. 按状态码保留结果。
14. 按状态码排除结果。
15. 按响应大小排除。
16. 按字数排除。
17. 按行数排除。
18. `-mr` 保留匹配 Regex，`-fr` 排除匹配 Regex。
19. 自动校准常见错误响应过滤。
20. 给 FUZZ 输入添加扩展名。
21. 防止请求数量快速增长。
22. 控制并发线程。
23. 控制每秒请求数。
24. 添加请求间隔。
25. 限制总运行时间。
26. DNS Subdomain 依赖 DNS；VHost 依赖 HTTP Host Header。
27. 无效 Host 可能都返回相同状态码。
28. 降低负载、日志和误报。
29. 查看实际 Status、Header、Body 和 Redirect。
30. 请求路径、数量、Header、时间和网络证据。
31. 需要 Web 日志收集和匹配规则。
32. 可能违法并影响他人服务。

---

# 106. Gobuster 命令速查

## 目录

```bash
gobuster dir \
  -u http://10.10.20.20:8083/ \
  -w dirs.txt
```

## 文件扩展

```bash
gobuster dir \
  -u http://10.10.20.20:8083/ \
  -w files.txt \
  -x txt,zip,json,html
```

## 低线程和延迟

```bash
gobuster dir \
  -u http://10.10.20.20:8083/ \
  -w dirs.txt \
  -t 5 \
  --delay 200ms
```

## 保存

```bash
gobuster dir \
  -u http://10.10.20.20:8083/ \
  -w dirs.txt \
  -o result.txt
```

## VHost

```bash
gobuster vhost \
  -u http://lab.local:8083/ \
  -w vhosts.txt \
  --append-domain
```

## Fuzz

```bash
gobuster fuzz \
  -u 'http://10.10.20.20:8083/item?id=FUZZ' \
  -w values.txt
```

---

# 107. ffuf 命令速查

## 目录

```bash
ffuf \
  -w dirs.txt \
  -u http://10.10.20.20:8083/FUZZ
```

## 扩展名

```bash
ffuf \
  -w files.txt \
  -u http://10.10.20.20:8083/FUZZ \
  -e .txt,.zip,.json
```

## Status Matcher

```bash
ffuf \
  -w dirs.txt \
  -u http://10.10.20.20:8083/FUZZ \
  -mc 200,301,302,401,403
```

## Soft 404 Filter

```bash
ffuf \
  -w dirs.txt \
  -u http://10.10.20.20:8083/soft/FUZZ \
  -mc all \
  -fs SIZE
```

## 自动校准

```bash
ffuf \
  -ac \
  -w dirs.txt \
  -u http://10.10.20.20:8083/FUZZ
```

## VHost

```bash
ffuf \
  -w vhosts.txt \
  -u http://10.10.20.20:8083/ \
  -H 'Host: FUZZ.lab.local' \
  -fs SIZE
```

## GET 参数

```bash
ffuf \
  -w values.txt \
  -u 'http://10.10.20.20:8083/item?id=FUZZ' \
  -mc 200
```

## POST 参数

```bash
ffuf \
  -w values.txt \
  -u http://10.10.20.20:8083/form \
  -X POST \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'role=FUZZ' \
  -mc 200
```

## 低负载

```bash
ffuf \
  -t 5 \
  -rate 5 \
  -maxtime 60 \
  -w dirs.txt \
  -u http://10.10.20.20:8083/FUZZ
```

## JSON 输出

```bash
ffuf \
  -w dirs.txt \
  -u http://10.10.20.20:8083/FUZZ \
  -o result.json \
  -of json
```

---

# 108. 核心记忆口诀

```text
Gobuster：
dir 找目录文件
vhost 找虚拟主机
fuzz 替换 FUZZ
-w 选 Wordlist
-u 选 URL
-x 加扩展
-t 控线程
-l 看长度
-o 存结果

ffuf：
FUZZ 标记位置
-mc 保留状态
-fc 排除状态
-fs 排除大小
-fw 排除字数
-fl 排除行数
-mr 保留 Regex
-fr 排除 Regex
-ac 自动校准
-rate 控速率
-maxtime 限时间
-of 选输出格式
```

---

# 109. 完成标准

完成本课后，你应该拥有：

```text
Ubuntu 8083 实验服务器
5 个小型 Wordlist
Gobuster 目录结果
Gobuster 文件结果
Gobuster VHost 结果
ffuf 目录结果
ffuf 文件结果
ffuf VHost 结果
ffuf GET 参数名结果
ffuf GET 参数值结果
ffuf POST 参数结果
Soft 404 对比
递归结果
Gobuster PCAP
ffuf PCAP
Wireshark 请求统计
Ubuntu Access Log
Wazuh discovery-lab 事件
JSON/HTML/Markdown 输出
完整实验报告
```

检查 Kali：

```bash
find ~/CyberLab/Discovery -type f
```

检查 Ubuntu：

```bash
find ~/CyberLab/Discovery -type f
```

---

# 110. 下一课

下一课建议：

```text
Nikto
```

学习内容：

- Web Server 基础安全检查
- Header 和危险文件线索
- 状态码
- 扫描插件
- 请求数量和日志
- 误报验证
- curl、Nmap 和 WhatWeb 对比
- tcpdump/Wireshark 流量分析
- Wazuh 观察

当前学习关系：

```text
Nmap：发现端口和服务
Netcat：测试连接
Wireshark：图形化流量分析
tcpdump：命令行抓包
curl：手动 HTTP 请求
WhatWeb：技术指纹识别
Gobuster/ffuf：目录、文件、VHost 和参数发现
```

---

# 111. 参考说明

本指南中的主要功能和参数依据：

```text
Gobuster 官方项目文档
ffuf 官方项目 README 与 Wiki
Kali Linux 官方工具页面
本机 gobuster/ffuf --help 输出
```

工具更新后，以本机帮助为准：

```bash
gobuster version
gobuster dir -h
gobuster vhost -h
gobuster fuzz -h
ffuf -V
ffuf -h
```
