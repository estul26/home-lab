# 第六课：WhatWeb 完整学习指南

> **实验环境：你自己的 CyberLab**
>
> 本课程只用于你自己创建、拥有并控制的 Web 服务。  
> 不要测试学校、公司、公共网站、随机公网 IP 或任何未经授权的系统。

---

# 1. 本课目标

学完本课后，你应该能够：

```text
理解 WhatWeb 的用途和限制
识别 Web Server、CMS、框架、JavaScript 库和 Header 线索
阅读状态码、标题、Cookie、Redirect 和插件结果
理解 Level 1、Level 3 和 Level 4
列出、搜索、查看和选择插件
控制 User-Agent、Header、Cookie、Redirect 和 Basic Auth
从文件读取多个实验目标
保存 Brief、Verbose、JSON、XML 和错误日志
使用线程、等待和超时参数降低扫描负载
使用 curl、Nmap、tcpdump、Wireshark 和 Wazuh 验证结果
理解误报、漏报和 Banner 欺骗
完成 Web 指纹识别实验报告
```

CyberLab：

```text
Kali-CyberLab      10.10.20.10
Ubuntu-Target      10.10.20.20
Windows-Target     10.10.20.30
Wazuh-SIEM         10.10.20.40
Network            10.10.20.0/24
```

---

# 2. 什么是 WhatWeb？

WhatWeb 是一个 Web 技术指纹识别工具。

它尝试回答：

```text
What is that website?
这个网站使用了什么技术？
```

它可能识别：

- Web Server
- CMS
- Web Framework
- Blog 平台
- JavaScript Library
- 前端框架
- Cookie
- HTTP Header
- 页面标题
- 重定向地址
- 版本号
- 邮箱地址
- 操作系统线索
- 错误页面线索

WhatWeb 使用插件匹配：

```text
HTTP Header
HTML 文字
正则表达式
HTML Tag
Cookie
URL 和路径
文件哈希
自定义 Ruby 逻辑
```

---

# 3. WhatWeb 不是什么？

WhatWeb 主要用于：

```text
Web Fingerprinting
Technology Identification
```

它不是完整漏洞扫描器。

WhatWeb 识别出：

```text
WordPress
Apache
PHP
jQuery
```

不代表：

```text
这些技术一定真实存在
版本一定准确
目标一定存在漏洞
目标一定可以被攻击
```

正确流程：

```text
WhatWeb 发现线索
→ curl 查看 Header 和 HTML
→ Nmap 确认端口和服务
→ 查看服务器进程和配置
→ 在授权范围内继续验证
```

---

# 4. 与前面工具的关系

```text
Nmap：发现端口和服务
Netcat：测试端口和原始连接
curl：手动发送 HTTP/HTTPS 请求
WhatWeb：自动识别 Web 技术
tcpdump：命令行抓取 WhatWeb 流量
Wireshark：详细分析请求和响应
Wazuh：分析服务器日志和安全事件
```

---

# 5. 检查和安装 WhatWeb

在 Kali：

```bash
which whatweb
```

查看版本：

```bash
whatweb --version
```

完整帮助：

```bash
whatweb -h
```

简短帮助：

```bash
whatweb --short-help
```

如果没有安装：

```bash
sudo apt update
sudo apt install whatweb
```

---

# 6. 创建实验目录

Kali：

```bash
mkdir -p ~/CyberLab/WhatWeb/{targets,results,captures,notes,reports}
```

Ubuntu：

```bash
mkdir -p ~/CyberLab/WhatWeb/{server,logs}
```

检查：

```bash
find ~/CyberLab/WhatWeb -maxdepth 2 -type d
```

---

# 7. 基本语法

```bash
whatweb [选项] URL
```

基础示例：

```bash
whatweb http://10.10.20.20:8080/
```

指定端口：

```bash
whatweb http://10.10.20.20:8082/
```

多个目标：

```bash
whatweb \
  http://10.10.20.20:8080/ \
  http://10.10.20.20:8082/
```

推荐明确写出：

```text
协议://IP:端口/路径
```

---

# 8. 启动简单 Web 实验服务

Ubuntu：

```bash
mkdir -p ~/CyberLab/WhatWeb/server/simple
```

创建网页：

```bash
cat > ~/CyberLab/WhatWeb/server/simple/index.html <<'EOF'
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>CyberLab Simple Website</title>
</head>
<body>
  <h1>WhatWeb Baseline Lab</h1>
  <p>Simple Python HTTP server.</p>
</body>
</html>
EOF
```

启动：

```bash
cd ~/CyberLab/WhatWeb/server/simple
python3 -m http.server 8080 --bind 10.10.20.20
```

保持终端运行。

---

# 9. 开始前验证

Kali：

```bash
ping -c 3 10.10.20.20
```

测试端口：

```bash
nc -zvn -w 3 10.10.20.20 8080
```

查看 Header：

```bash
curl -I http://10.10.20.20:8080/
```

Nmap：

```bash
sudo nmap -n -sV -p 8080 10.10.20.20
```

---

# 10. 实验一：基础识别

```bash
whatweb http://10.10.20.20:8080/
```

可能看到：

```text
http://10.10.20.20:8080/ [200 OK]
HTML5
HTTPServer[SimpleHTTP/... Python/...]
IP[10.10.20.20]
Title[CyberLab Simple Website]
```

实际结果取决于：

- WhatWeb 版本
- Python 版本
- 页面内容
- HTTP Header
- 本地插件版本

---

# 11. 阅读输出

常见格式：

```text
URL [HTTP Status] Plugin[Data], Plugin, Plugin[Version]
```

常见字段：

| 字段 | 含义 |
|---|---|
| `200 OK` | 请求成功 |
| `301/302` | 重定向 |
| `404` | 路径不存在 |
| `IP` | 目标 IP |
| `Title` | 页面标题 |
| `HTTPServer` | Server Header 线索 |
| `HTML5` | HTML5 Doctype |
| `Cookies` | Set-Cookie 线索 |
| `RedirectLocation` | 重定向目标 |
| `X-Powered-By` | 应用技术 Header |
| `UncommonHeaders` | 不常见 Header |
| `JQuery` | jQuery 指纹 |
| `WordPress` | WordPress 指纹 |

---

# 12. Verbose 输出

```bash
whatweb -v http://10.10.20.20:8080/
```

Verbose 可能包含：

- HTTP Status
- Title
- 检测摘要
- 插件描述
- 版本线索
- HTTP Headers
- 匹配来源

调试输出：

```bash
whatweb -vv http://10.10.20.20:8080/
```

初学推荐：

```text
先普通输出
再使用 -v 查看证据
```

---

# 13. 结果不是绝对事实

看到：

```text
Apache[2.4.58]
```

不一定证明后台真的运行 Apache。

原因：

- Server Header 可以修改
- CDN 或反向代理会隐藏后端
- HTML 模板可能包含旧线索
- 文件名可以伪造
- 插件可能误匹配
- 缓存页面可能过时

报告中写：

```text
Possible technology
Detected fingerprint
Observed indication
```

而不是直接写：

```text
Confirmed
```

---

# 14. Aggression Level

使用：

```bash
whatweb -a LEVEL URL
```

主要等级：

| 等级 | 名称 | 行为 |
|---:|---|---|
| 1 | Stealthy | 通常每个目标一个请求，并处理 Redirect |
| 3 | Aggressive | Level 1 插件匹配后发送额外请求 |
| 4 | Heavy | 多个插件尝试许多额外 URL |

注意：

```text
主要等级中没有 Level 2
```

---

# 15. Level 1：Stealthy

默认就是 Level 1：

```bash
whatweb http://10.10.20.20:8080/
```

明确指定：

```bash
whatweb -a 1 http://10.10.20.20:8080/
```

特点：

- 请求少
- 速度快
- 服务器负载低
- 适合第一次识别

默认从这里开始。

---

# 16. Level 3：Aggressive

```bash
whatweb -a 3 http://10.10.20.20:8080/
```

特点：

- 基础插件匹配后才进行额外测试
- 可能访问默认文件或路径
- 可能识别更多版本信息
- 服务器日志会增加

只在：

```text
自己的 CyberLab
明确授权的测试环境
```

中使用。

---

# 17. Level 4：Heavy

```bash
whatweb -a 4 http://10.10.20.20:8082/
```

特点：

- 请求很多
- 扫描更慢
- 服务器日志明显增加
- 可能产生不必要负载

本课程只允许对：

```text
你自己创建的 Ubuntu 8082 模拟服务
```

运行一次受控实验。

不要对：

```text
真实网站
Wazuh Dashboard
整个网段
公共服务
```

使用 Heavy。

---

# 18. 推荐升级流程

```text
curl -I 确认服务
→ WhatWeb -a 1
→ WhatWeb -v -a 1
→ 找出相关插件
→ 只选择相关插件运行 -a 3
→ 必要时在自己的模拟服务测试一次 -a 4
```

不要一开始就使用：

```text
-a 4
```

---

# 19. 创建模拟指纹服务器

此实验服务器故意提供：

```text
Apache 风格 Server Header
PHP 风格 X-Powered-By
WordPress Generator Meta
jQuery 路径
Bootstrap 路径
Cookie
Redirect
WordPress 风格路径
```

但它实际运行的是：

```text
Python
```

目的是证明：

```text
Web 指纹可以被伪造
WhatWeb 结果必须验证
```

---

# 20. Ubuntu 模拟指纹服务器代码

在 Ubuntu：

```bash
nano ~/CyberLab/WhatWeb/server/fingerprint_lab.py
```

粘贴：

```python
#!/usr/bin/env python3

from http.server import BaseHTTPRequestHandler, HTTPServer
from urllib.parse import urlparse


HOST = "10.10.20.20"
PORT = 8082


HTML = """<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="generator" content="WordPress 6.5">
  <title>CyberLab Technology Fingerprint Test</title>
  <link rel="stylesheet"
        href="/wp-content/themes/cyberlab/bootstrap.min.css">
  <script src="/wp-includes/js/jquery/jquery-3.7.1.min.js"></script>
</head>
<body>
  <h1>CyberLab WhatWeb Lab</h1>
  <p>Powered by CyberLab CMS</p>
  <a href="/wp-login.php">Lab Login</a>
</body>
</html>
""".encode("utf-8")


LOGIN_HTML = """<!doctype html>
<html>
<head>
  <meta name="generator" content="WordPress 6.5">
  <title>CyberLab Login</title>
</head>
<body>
  <form action="/wp-login.php" method="post">
    <input name="log">
    <input name="pwd" type="password">
  </form>
</body>
</html>
""".encode("utf-8")


class FingerprintHandler(BaseHTTPRequestHandler):
    server_version = "Apache/2.4.58"
    sys_version = "(Ubuntu)"

    def common_headers(self, body_length: int, content_type: str) -> None:
        self.send_header("Content-Type", content_type)
        self.send_header("Content-Length", str(body_length))
        self.send_header("X-Powered-By", "PHP/8.2.10")
        self.send_header("X-Frame-Options", "SAMEORIGIN")
        self.send_header("X-Content-Type-Options", "nosniff")
        self.send_header("X-CyberLab", "WhatWeb-Fingerprint-Lab")
        self.send_header(
            "Set-Cookie",
            "wordpress_test_cookie=WP%20Cookie; Path=/; HttpOnly",
        )

    def send_body(
        self,
        status: int,
        body: bytes,
        content_type: str = "text/html; charset=utf-8",
    ) -> None:
        self.send_response(status)
        self.common_headers(len(body), content_type)
        self.end_headers()

        if self.command != "HEAD":
            self.wfile.write(body)

    def do_GET(self) -> None:
        path = urlparse(self.path).path

        if path == "/":
            self.send_body(200, HTML)
            return

        if path == "/redirect":
            self.send_response(302)
            self.send_header("Location", "/")
            self.send_header("Content-Length", "0")
            self.end_headers()
            return

        if path == "/wp-login.php":
            self.send_body(200, LOGIN_HTML)
            return

        if path in {
            "/readme.html",
            "/license.txt",
            "/wp-admin/",
            "/wp-content/",
            "/wp-includes/",
        }:
            body = (
                f"CyberLab simulated WordPress path: {path}\n"
                "This is not a real WordPress installation.\n"
            ).encode("utf-8")
            self.send_body(200, body, "text/plain; charset=utf-8")
            return

        if path.endswith(".js"):
            body = b"/* jQuery v3.7.1 - CyberLab placeholder */\n"
            self.send_body(200, body, "application/javascript")
            return

        if path.endswith(".css"):
            body = b"/* Bootstrap CyberLab placeholder */\n"
            self.send_body(200, body, "text/css")
            return

        self.send_body(404, b"CyberLab 404\n", "text/plain; charset=utf-8")

    def do_HEAD(self) -> None:
        self.do_GET()

    def log_message(self, format: str, *args: object) -> None:
        print(
            f"{self.client_address[0]} "
            f"[{self.log_date_time_string()}] "
            f"{format % args}",
            flush=True,
        )


def main() -> None:
    server = HTTPServer((HOST, PORT), FingerprintHandler)
    print(f"Fingerprint lab listening on http://{HOST}:{PORT}")
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
chmod +x ~/CyberLab/WhatWeb/server/fingerprint_lab.py
```

---

# 21. 启动模拟服务器

```bash
python3 ~/CyberLab/WhatWeb/server/fingerprint_lab.py \
  2>&1 |
tee ~/CyberLab/WhatWeb/logs/fingerprint-access.log
```

预期：

```text
Fingerprint lab listening on http://10.10.20.20:8082
```

验证：

```bash
ss -lntp | grep ':8082'
```

Kali：

```bash
curl -I http://10.10.20.20:8082/
```

---

# 22. 实验二：识别模拟技术

```bash
whatweb http://10.10.20.20:8082/
```

可能识别到部分线索：

```text
Apache
Bootstrap
Cookies
HTML5
HTTPServer
JQuery
PHP
Title
WordPress
X-Frame-Options
```

实际结果取决于本地插件。

关键问题：

```text
服务器真的运行 Apache 吗？
真的运行 PHP 吗？
真的是 WordPress 吗？
```

答案：

```text
不是。
真正程序是 Python。
这些只是模拟指纹。
```

---

# 23. 使用 curl 验证线索

Header：

```bash
curl -I http://10.10.20.20:8082/
```

源码：

```bash
curl -s http://10.10.20.20:8082/ |
less
```

查 Generator：

```bash
curl -s http://10.10.20.20:8082/ |
grep -i generator
```

查 WordPress 风格路径：

```bash
curl -s http://10.10.20.20:8082/ |
grep -Eo '/wp-[^" ]+'
```

验证真实进程：

```bash
sudo ss -lntp | grep ':8082'
```

```bash
ps aux |
grep fingerprint_lab |
grep -v grep
```

---

# 24. 比较 Level 1 和 Level 3

先运行：

```bash
whatweb -a 1 http://10.10.20.20:8082/
```

记录 Ubuntu Access Log 中的请求数量。

然后：

```bash
whatweb -a 3 http://10.10.20.20:8082/
```

比较：

| 项目 | Level 1 | Level 3 |
|---|---|---|
| 请求数量 |  |  |
| 请求路径 |  |  |
| 识别结果 |  |  |
| 扫描时间 |  |  |

---

# 25. 受控 Level 4 实验

只运行一次：

```bash
whatweb \
  -a 4 \
  --max-threads 1 \
  --wait 0.2 \
  http://10.10.20.20:8082/
```

参数：

```text
--max-threads 1 = 一个线程
--wait 0.2       = 连接之间等待
```

观察 Ubuntu Access Log 中增加的路径。

---

# 26. User-Agent

```bash
whatweb \
  -U 'CyberLab-WhatWeb-Student/1.0' \
  http://10.10.20.20:8082/
```

长参数：

```bash
whatweb \
  --user-agent 'CyberLab-WhatWeb-Student/1.0' \
  http://10.10.20.20:8082/
```

User-Agent 可以被修改，因此：

```text
不能用 User-Agent 作为可靠身份验证
```

---

# 27. 自定义 Header

```bash
whatweb \
  -H 'X-Lab-Name:WhatWeb' \
  -H 'X-Student:CyberLab' \
  http://10.10.20.20:8082/
```

修改 User-Agent Header：

```bash
whatweb \
  -H 'User-Agent:CyberLab-Browser' \
  http://10.10.20.20:8082/
```

---

# 28. Cookie

手动提供：

```bash
whatweb \
  -c 'cyberlab=student; lesson=whatweb' \
  http://10.10.20.20:8082/
```

Cookie 文件：

```bash
touch ~/CyberLab/WhatWeb/targets/cookies.txt
```

```bash
whatweb \
  --cookie-jar ~/CyberLab/WhatWeb/targets/cookies.txt \
  http://10.10.20.20:8082/
```

如果本机选项名称不同，检查：

```bash
whatweb -h
```

---

# 29. Basic Authentication

语法：

```bash
whatweb \
  -u 'labuser:labpassword' \
  http://10.10.20.20:8082/
```

当前模拟服务器没有实现 Basic Auth。

安全注意：

```text
Basic Auth 必须配合 HTTPS
命令行密码可能进入 Shell History
只使用测试账号和测试密码
```

---

# 30. Redirect 控制

模拟路径：

```text
http://10.10.20.20:8082/redirect
```

默认：

```bash
whatweb http://10.10.20.20:8082/redirect
```

不跟随：

```bash
whatweb \
  --follow-redirect=never \
  http://10.10.20.20:8082/redirect
```

始终跟随：

```bash
whatweb \
  --follow-redirect=always \
  http://10.10.20.20:8082/redirect
```

可用值：

```text
never
http-only
meta-only
same-site
always
```

限制次数：

```bash
whatweb \
  --max-redirects=3 \
  http://10.10.20.20:8082/redirect
```

---

# 31. 插件是什么？

每个 WhatWeb 插件负责识别一种技术或线索。

示例：

```text
Apache
Bootstrap
Cookies
HTML5
HTTPServer
JQuery
PHP
Title
WordPress
X-Frame-Options
```

插件可能根据：

- Header
- HTML
- Cookie
- URL
- 文件哈希
- 特定文件和路径
- 正则表达式

进行匹配。

---

# 32. 列出插件

```bash
whatweb --list-plugins
```

简写：

```bash
whatweb -l
```

分页：

```bash
whatweb -l |
less
```

搜索列表：

```bash
whatweb -l |
grep -i wordpress
```

---

# 33. 搜索插件

```bash
whatweb --search-plugins=wordpress
```

其他例子：

```bash
whatweb --search-plugins=apache
```

```bash
whatweb --search-plugins=jquery
```

插件名称以你本机结果为准。

---

# 34. 查看插件详情

```bash
whatweb -I WordPress
```

或：

```bash
whatweb --info-plugins=WordPress
```

重点查看：

```text
插件根据什么证据识别？
Passive Match 是什么？
Aggressive Match 会请求哪些路径？
结果是否容易被伪造？
```

---

# 35. 只运行指定插件

先搜索插件名：

```bash
whatweb --search-plugins=wordpress
```

然后：

```bash
whatweb \
  -p WordPress \
  http://10.10.20.20:8082/
```

多个插件：

```bash
whatweb \
  -p WordPress,PHP,Apache,JQuery \
  http://10.10.20.20:8082/
```

指定插件的优点：

- 请求更少
- 输出更集中
- 更容易理解
- Aggressive 模式更可控

---

# 36. 指定插件运行 Level 3

```bash
whatweb \
  -a 3 \
  -p WordPress \
  -t 1 \
  http://10.10.20.20:8082/
```

推荐原则：

```text
知道要验证什么
→ 选择对应插件
→ 再使用 Level 3
```

---

# 37. WhatWeb Grep

只显示包含 WordPress 的结果：

```bash
whatweb \
  -g 'WordPress' \
  http://10.10.20.20:8082/
```

正则：

```bash
whatweb \
  -g '/WordPress|PHP|Apache/' \
  http://10.10.20.20:8082/
```

这里的 `-g` 是 WhatWeb 自己的结果筛选。

---

# 38. 多目标文件

创建：

```bash
cat > ~/CyberLab/WhatWeb/targets/lab-urls.txt <<'EOF'
http://10.10.20.20:8080/
http://10.10.20.20:8082/
http://10.10.20.20:8082/redirect
EOF
```

运行：

```bash
whatweb \
  -i ~/CyberLab/WhatWeb/targets/lab-urls.txt
```

详细输出：

```bash
whatweb \
  -v \
  -i ~/CyberLab/WhatWeb/targets/lab-urls.txt
```

---

# 39. 从管道读取目标

```bash
printf '%s\n' \
  'http://10.10.20.20:8080/' \
  'http://10.10.20.20:8082/' |
whatweb -i /dev/stdin
```

---

# 40. URL Prefix 和 Suffix

创建主机列表：

```bash
cat > ~/CyberLab/WhatWeb/targets/lab-hosts.txt <<'EOF'
10.10.20.20
EOF
```

运行：

```bash
whatweb \
  -i ~/CyberLab/WhatWeb/targets/lab-hosts.txt \
  --url-prefix 'http://' \
  --url-suffix ':8082/'
```

构造结果：

```text
http://10.10.20.20:8082/
```

---

# 41. 保存 Brief 日志

```bash
whatweb \
  --log-brief=~/CyberLab/WhatWeb/results/lab-brief.txt \
  http://10.10.20.20:8082/
```

查看：

```bash
cat ~/CyberLab/WhatWeb/results/lab-brief.txt
```

Brief 适合：

- 一行摘要
- 多目标比较
- 文本搜索

---

# 42. 保存 Verbose 日志

```bash
whatweb \
  --log-verbose=~/CyberLab/WhatWeb/results/lab-verbose.txt \
  http://10.10.20.20:8082/
```

查看：

```bash
less ~/CyberLab/WhatWeb/results/lab-verbose.txt
```

Verbose 适合：

- 写报告
- 查看 Header
- 查看插件证据
- 学习插件工作方式

---

# 43. 保存 JSON

```bash
whatweb \
  --log-json=~/CyberLab/WhatWeb/results/lab.json \
  http://10.10.20.20:8082/
```

查看：

```bash
head -n 30 ~/CyberLab/WhatWeb/results/lab.json
```

尝试格式化：

```bash
python3 -m json.tool \
  ~/CyberLab/WhatWeb/results/lab.json
```

如果无法解析，可能是本机版本输出结构不同，以原始文件为准。

---

# 44. 保存 Verbose JSON

```bash
whatweb \
  --log-json-verbose=~/CyberLab/WhatWeb/results/lab-verbose.json \
  http://10.10.20.20:8082/
```

该文件通常更大，包含更多插件信息。

---

# 45. 保存 XML

```bash
whatweb \
  --log-xml=~/CyberLab/WhatWeb/results/lab.xml \
  http://10.10.20.20:8082/
```

查看：

```bash
head -n 20 ~/CyberLab/WhatWeb/results/lab.xml
```

---

# 46. 保存错误日志

```bash
whatweb \
  --log-errors=~/CyberLab/WhatWeb/results/errors.log \
  http://10.10.20.20:65500/
```

查看：

```bash
cat ~/CyberLab/WhatWeb/results/errors.log
```

可能记录：

- Connection refused
- Timeout
- DNS 问题
- TLS 问题
- 插件异常

---

# 47. 同时保存多种格式

```bash
whatweb \
  --log-brief=~/CyberLab/WhatWeb/results/full-brief.txt \
  --log-verbose=~/CyberLab/WhatWeb/results/full-verbose.txt \
  --log-json=~/CyberLab/WhatWeb/results/full.json \
  --log-xml=~/CyberLab/WhatWeb/results/full.xml \
  --log-errors=~/CyberLab/WhatWeb/results/full-errors.log \
  http://10.10.20.20:8082/
```

---

# 48. Quiet、No Errors 和颜色

不在终端显示 Brief：

```bash
whatweb \
  -q \
  --log-verbose=~/CyberLab/WhatWeb/results/quiet.txt \
  http://10.10.20.20:8082/
```

隐藏错误：

```bash
whatweb \
  --no-errors \
  http://10.10.20.20:8082/
```

排错时不要使用 `--no-errors`。

禁止颜色：

```bash
whatweb \
  --colour=never \
  http://10.10.20.20:8082/
```

---

# 49. 线程控制

```bash
whatweb \
  --max-threads 1 \
  http://10.10.20.20:8082/
```

简写：

```bash
whatweb \
  -t 1 \
  http://10.10.20.20:8082/
```

本课程建议：

```text
单目标：1 个线程
少量 URL：1–3 个线程
```

---

# 50. Wait 和 Timeout

连接之间等待：

```bash
whatweb \
  -t 1 \
  --wait 1 \
  http://10.10.20.20:8082/
```

连接超时：

```bash
whatweb \
  --open-timeout 3 \
  http://10.10.20.20:8082/
```

读取超时：

```bash
whatweb \
  --read-timeout 5 \
  http://10.10.20.20:8082/
```

组合：

```bash
whatweb \
  -t 1 \
  --wait 0.5 \
  --open-timeout 3 \
  --read-timeout 5 \
  http://10.10.20.20:8082/
```

---

# 51. WhatWeb 与 Nmap

Nmap：

```bash
sudo nmap -n -sV -p 8082 10.10.20.20
```

WhatWeb：

```bash
whatweb http://10.10.20.20:8082/
```

| Nmap | WhatWeb |
|---|---|
| 判断端口状态 | 识别网页技术 |
| 服务版本探测 | Header/HTML/Cookie 指纹 |
| 可发现非 Web 服务 | 主要针对 Web |
| 输出端口和服务 | 输出 CMS、框架和页面线索 |

---

# 52. WhatWeb 与 curl

curl：

```bash
curl -i http://10.10.20.20:8082/
```

WhatWeb：

```bash
whatweb -v http://10.10.20.20:8082/
```

| curl | WhatWeb |
|---|---|
| 显示原始请求和响应 | 自动总结技术 |
| 精确控制请求 | 自动匹配插件 |
| 需要人工判断 | 快速发现线索 |
| 适合验证 | 适合识别 |

组合：

```text
WhatWeb 发现
→ curl 验证
```

---

# 53. tcpdump 实验

Kali 终端一：

```bash
sudo tcpdump \
  -i eth0 \
  -nn \
  -A \
  'host 10.10.20.20 and tcp port 8082'
```

终端二：

```bash
whatweb \
  -U 'CyberLab-WhatWeb/1.0' \
  -H 'X-Lab-Test:WhatWeb-tcpdump' \
  http://10.10.20.20:8082/
```

tcpdump 可能显示：

```text
GET / HTTP/1.1
Host: 10.10.20.20:8082
User-Agent: CyberLab-WhatWeb/1.0
X-Lab-Test: WhatWeb-tcpdump
```

---

# 54. 保存 Level 1 和 Level 3 PCAP

Level 1 抓包：

```bash
sudo tcpdump \
  -i eth0 \
  -nn \
  -s 0 \
  -w ~/CyberLab/WhatWeb/captures/whatweb-level1.pcap \
  'host 10.10.20.20 and tcp port 8082'
```

另一个终端：

```bash
whatweb -a 1 http://10.10.20.20:8082/
```

Level 3 抓包：

```bash
sudo tcpdump \
  -i eth0 \
  -nn \
  -s 0 \
  -w ~/CyberLab/WhatWeb/captures/whatweb-level3.pcap \
  'host 10.10.20.20 and tcp port 8082'
```

另一个终端：

```bash
whatweb -a 3 http://10.10.20.20:8082/
```

每次完成后按：

```text
Ctrl + C
```

停止 tcpdump。

---

# 55. Wireshark 分析

打开：

```bash
wireshark \
  ~/CyberLab/WhatWeb/captures/whatweb-level3.pcap
```

过滤：

```text
http
```

只看请求：

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
tcp.port == 8082
```

User-Agent：

```text
http.user_agent contains "WhatWeb"
```

---

# 56. 比较 Level 1 和 Level 3 流量

Wireshark：

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

| 项目 | Level 1 | Level 3 |
|---|---:|---:|
| HTTP 请求数 |  |  |
| 不同路径数 |  |  |
| TCP 会话数 |  |  |
| 总数据包数 |  |  |
| 总字节数 |  |  |

---

# 57. 查看 Ubuntu Access Log

查看：

```bash
less ~/CyberLab/WhatWeb/logs/fingerprint-access.log
```

只看 Kali：

```bash
grep '10.10.20.10' \
  ~/CyberLab/WhatWeb/logs/fingerprint-access.log
```

统计 GET：

```bash
grep -c '"GET ' \
  ~/CyberLab/WhatWeb/logs/fingerprint-access.log
```

Level 3 或 Level 4 可能请求：

```text
/wp-login.php
/readme.html
/license.txt
/wp-admin/
/wp-content/
```

实际路径取决于插件版本。

---

# 58. Wazuh 日志链路

Python Access Log 默认不一定被 Wazuh 收集。

在 Ubuntu 扫描前：

```bash
logger -t whatweb-lab \
  "Starting authorized WhatWeb scan from Kali 10.10.20.10"
```

Kali：

```bash
whatweb -a 1 http://10.10.20.20:8082/
```

Ubuntu 扫描后：

```bash
logger -t whatweb-lab \
  "Completed authorized WhatWeb scan against TCP 8082"
```

本地查看：

```bash
journalctl \
  -t whatweb-lab \
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
whatweb-lab
```

---

# 59. WhatWeb 是否一定触发 Wazuh 告警？

不一定。

WhatWeb 产生的是 HTTP 请求。

如果没有：

- Web Access Log
- Wazuh 日志收集配置
- 匹配规则
- WAF
- IDS/IPS

Wazuh 可能不会产生明显扫描告警。

本课目标是确认：

```text
网络请求存在
服务器日志存在
Wazuh 基础日志链路存在
```

不是要求一定产生高级攻击告警。

---

# 60. HTTPS 目标

可以尝试自己的 Wazuh Dashboard：

```bash
whatweb https://10.10.20.40/
```

实验室可能使用：

```text
自签名证书
不受信任 CA
证书名称与 IP 不匹配
```

如果 WhatWeb 返回 TLS 错误：

```text
记录错误
不要给 WhatWeb 使用不存在的 curl -k 参数
```

使用 curl 检查：

```bash
curl -vkI https://10.10.20.40/
```

使用 Nmap 检查证书：

```bash
sudo nmap -n \
  -p 443 \
  --script ssl-cert \
  10.10.20.40
```

---

# 61. HTTPS 抓包

Kali：

```bash
sudo tcpdump \
  -i eth0 \
  -nn \
  -s 0 \
  -w ~/CyberLab/WhatWeb/captures/whatweb-wazuh-tls.pcap \
  'host 10.10.20.40 and tcp port 443'
```

另一个终端：

```bash
whatweb https://10.10.20.40/
```

Wireshark 过滤：

```text
tls
```

可能看到：

- TCP 握手
- TLS Client Hello
- TLS Server Hello
- Certificate
- TLS Alert
- 加密数据

---

# 62. Burp Suite 代理预览

以后学习 Burp 时：

```bash
whatweb \
  --proxy 127.0.0.1:8080 \
  http://10.10.20.20:8082/
```

如果代理需要认证：

```bash
whatweb \
  --proxy 127.0.0.1:8080 \
  --proxy-user 'user:password' \
  http://10.10.20.20:8082/
```

本课不要求配置 Burp。

---

# 63. 常见误报来源

## Server Header

```text
Server: Apache/2.4.58
```

可以被自定义程序伪造。

## Generator Meta

```html
<meta name="generator" content="WordPress 6.5">
```

可以出现在普通静态页面。

## JavaScript 文件名

```text
jquery-3.7.1.min.js
```

不一定证明内容是真的 jQuery。

## 反向代理和 CDN

WhatWeb 可能只看到前端代理，无法看到真实后端。

## 旧缓存或模板

页面可能保留已移除技术的旧线索。

---

# 64. 常见漏报来源

- Header 被隐藏
- Generator Meta 被删除
- JavaScript 被打包和改名
- 页面需要登录
- 技术只存在于其他路径
- WAF 返回不同页面
- CDN 隐藏源站
- 插件没有对应规则
- JavaScript 动态加载内容
- 自定义框架

没有识别到：

```text
不代表技术不存在
```

---

# 65. 验证清单

当 WhatWeb 识别某项技术：

```text
1. 使用 -v 查看匹配来源
2. curl 查看 Header
3. curl 查看 HTML
4. 检查 Cookie
5. 检查 JavaScript/CSS 路径
6. 选择对应插件重复扫描
7. 对比 Nmap 服务探测
8. 查看监听进程
9. 查看服务器配置
10. 在报告中标注置信度
```

---

# 66. 置信度写法

示例：

```text
Detected: WordPress 6.5
Confidence: Low
Evidence: Generator meta and WordPress-style paths
Validation: Server process is Python
Conclusion: Simulated false positive
```

另一个示例：

```text
Detected: HTTP service on TCP 8082
Confidence: High
Evidence: Nmap, curl, WhatWeb, ss and Python process
```

---

# 67. 完整扫描流程

```text
1. 确认授权
2. 确认 IP 和端口
3. nc/Nmap 验证服务
4. curl 查看 Header
5. WhatWeb -a 1
6. WhatWeb -v
7. 搜索相关插件
8. 指定插件运行 -a 3
9. tcpdump/Wireshark 观察流量
10. 查看服务器 Access Log
11. 查看 Wazuh
12. 验证真实进程
13. 保存结果
14. 写报告
```

---

# 68. 完整实验一：简单 Python Server

Ubuntu：

```bash
cd ~/CyberLab/WhatWeb/server/simple
python3 -m http.server 8080 --bind 10.10.20.20
```

Kali：

```bash
curl -I http://10.10.20.20:8080/
```

```bash
whatweb http://10.10.20.20:8080/
```

```bash
whatweb -v http://10.10.20.20:8080/
```

保存：

```bash
whatweb \
  --log-brief=~/CyberLab/WhatWeb/results/simple-brief.txt \
  --log-verbose=~/CyberLab/WhatWeb/results/simple-verbose.txt \
  http://10.10.20.20:8080/
```

---

# 69. 完整实验二：模拟指纹和误报

Ubuntu：

```bash
python3 ~/CyberLab/WhatWeb/server/fingerprint_lab.py \
  2>&1 |
tee ~/CyberLab/WhatWeb/logs/fingerprint-access.log
```

Kali：

```bash
whatweb -v http://10.10.20.20:8082/
```

验证：

```bash
curl -i http://10.10.20.20:8082/
```

Ubuntu：

```bash
sudo ss -lntp | grep ':8082'
```

```bash
ps aux |
grep fingerprint_lab |
grep -v grep
```

回答：

```text
WhatWeb 识别了什么？
真实服务器是什么？
哪些指纹是模拟的？
```

---

# 70. 完整实验三：Aggression 比较

```bash
whatweb -a 1 http://10.10.20.20:8082/
```

```bash
whatweb -a 3 http://10.10.20.20:8082/
```

```bash
whatweb \
  -a 4 \
  -t 1 \
  --wait 0.2 \
  http://10.10.20.20:8082/
```

比较：

| 等级 | 请求数 | 路径数 | 时间 | 额外发现 |
|---:|---:|---:|---:|---|
| 1 |  |  |  |  |
| 3 |  |  |  |  |
| 4 |  |  |  |  |

---

# 71. 完整实验四：WordPress 插件

搜索：

```bash
whatweb --search-plugins=wordpress
```

详情：

```bash
whatweb -I WordPress
```

指定插件：

```bash
whatweb \
  -p WordPress \
  http://10.10.20.20:8082/
```

指定插件 Level 3：

```bash
whatweb \
  -a 3 \
  -p WordPress \
  -t 1 \
  http://10.10.20.20:8082/
```

---

# 72. 完整实验五：PCAP 分析

终端一：

```bash
sudo tcpdump \
  -i eth0 \
  -nn \
  -s 0 \
  -w ~/CyberLab/WhatWeb/captures/whatweb-wordpress.pcap \
  'host 10.10.20.20 and tcp port 8082'
```

终端二：

```bash
whatweb \
  -a 3 \
  -p WordPress \
  http://10.10.20.20:8082/
```

Wireshark：

```bash
wireshark \
  ~/CyberLab/WhatWeb/captures/whatweb-wordpress.pcap
```

过滤：

```text
http.request
```

查看：

```text
Statistics
→ HTTP
→ Requests
```

---

# 73. 完整实验六：多目标和多格式输出

创建：

```bash
cat > ~/CyberLab/WhatWeb/targets/final-lab.txt <<'EOF'
http://10.10.20.20:8080/
http://10.10.20.20:8082/
http://10.10.20.20:8082/redirect
EOF
```

运行：

```bash
whatweb \
  -a 1 \
  -t 1 \
  --colour=never \
  --log-brief=~/CyberLab/WhatWeb/results/final-brief.txt \
  --log-verbose=~/CyberLab/WhatWeb/results/final-verbose.txt \
  --log-json=~/CyberLab/WhatWeb/results/final.json \
  --log-xml=~/CyberLab/WhatWeb/results/final.xml \
  --log-errors=~/CyberLab/WhatWeb/results/final-errors.log \
  -i ~/CyberLab/WhatWeb/targets/final-lab.txt
```

检查：

```bash
ls -lh ~/CyberLab/WhatWeb/results
```

---

# 74. 实验报告模板

## Lab Information

```text
Lab Name：WhatWeb Technology Fingerprinting
Date：
Student：
Scanner：Kali-CyberLab 10.10.20.10
Target：Ubuntu-Target 10.10.20.20
URLs：
- http://10.10.20.20:8080/
- http://10.10.20.20:8082/
Authorization：Personal CyberLab
```

## Objective

```text
使用 WhatWeb 识别 Web 技术，并使用 curl、Nmap、
tcpdump、Wireshark 和服务器进程验证结果。
```

## Pre-Scan Commands

```bash
# Ping
# nc
# Nmap
# curl
```

## WhatWeb Commands

```bash
# Level 1
# Verbose
# Level 3
# Selected plugin
# Logging
```

## Results

| URL | Status | Detected Technology | Evidence | Confidence |
|---|---|---|---|---|
|  |  |  |  |  |

## Aggression Comparison

| Level | Requests | Paths | Duration | Findings |
|---:|---:|---:|---:|---|
| 1 |  |  |  |  |
| 3 |  |  |  |  |
| 4 |  |  |  |  |

## Validation

```text
curl Header：
HTML Source：
Nmap：
Server Process：
Listening Process：
Access Log：
PCAP：
Wazuh Event：
```

## False Positive Analysis

```text
WhatWeb detected：
Actual technology：
Why it matched：
How it was verified：
```

## Recommendations

```text
减少不必要的版本泄露。
移除不需要的 Generator Meta。
不要把 Header 隐藏当作唯一安全措施。
保持软件及时更新。
监控异常路径和高频请求。
```

## Lessons Learned

```text
WhatWeb 最有用的功能：
WhatWeb 最大限制：
为什么必须验证：
```

---

# 75. 常见错误和解决方法

## command not found

```bash
sudo apt update
sudo apt install whatweb
```

## Connection refused

Kali：

```bash
ping -c 3 10.10.20.20
```

```bash
nc -zvn -w 3 10.10.20.20 8082
```

Ubuntu：

```bash
ss -lntp | grep ':8082'
```

## Timeout

```bash
whatweb \
  --open-timeout 3 \
  --read-timeout 5 \
  http://10.10.20.20:8082/
```

## 输出很少

可能原因：

- 页面简单
- Header 很少
- 技术被隐藏
- 没有对应插件
- 内容由 JavaScript 动态加载

尝试：

```bash
whatweb -v URL
```

然后仅对实验目标：

```bash
whatweb -a 3 URL
```

## Level 3 和 Level 1 一样

可能原因：

- Level 1 没有匹配相关插件
- 插件没有 Aggressive Tests
- 插件版本不同
- 所有路径返回相同页面

查看：

```bash
whatweb -I PluginName
```

## 日志文件不存在

```bash
mkdir -p ~/CyberLab/WhatWeb/results
```

确保使用：

```text
--log-brief=FILE
```

## JSON 无法格式化

```bash
head -n 30 file.json
```

本机版本可能输出不同结构。

## HTTPS 证书错误

```bash
curl -vkI https://10.10.20.40/
```

不要给 WhatWeb 使用不存在的 `-k`。

## 结果和真实服务不同

```bash
curl -i URL
sudo ss -lntp
ps aux
```

验证实际技术。

---

# 76. 安全注意事项

WhatWeb 请求可能：

- 出现在 Access Log
- 触发 WAF 或 IDS
- 暴露源 IP
- 请求默认文件和路径
- 增加服务器负载

因此：

```text
只扫描授权目标
默认使用 Level 1
使用低线程
限制请求
保留实验记录
不要绕过检测
```

本课程不教授：

```text
IDS 规避
WAF 绕过
隐藏身份
未授权大规模扫描
```

---

# 77. 本课练习

1. 查看 WhatWeb 版本和帮助。
2. 识别 Python SimpleHTTPServer。
3. 使用 `-v` 查看证据。
4. 创建模拟指纹服务器。
5. 记录所有识别结果。
6. 使用 curl 验证 Header。
7. 使用 curl 验证 Generator Meta。
8. 使用 `ss` 和 `ps` 确认真实进程。
9. 比较 Level 1 和 Level 3。
10. 只在模拟服务运行一次 Level 4。
11. 搜索 WordPress 插件。
12. 查看 WordPress 插件详情。
13. 指定 WordPress 插件扫描。
14. 修改 User-Agent。
15. 添加自定义 Header。
16. 比较 Redirect 的 never 和 always。
17. 使用目标文件扫描三个 URL。
18. 保存 Brief 和 Verbose。
19. 保存 JSON 和 XML。
20. 保存错误日志。
21. 使用 `-t 1 --wait 1`。
22. tcpdump 查看请求。
23. 保存 Level 1 和 Level 3 PCAP。
24. Wireshark 比较请求数量。
25. 查看 Ubuntu Access Log。
26. 在 Wazuh 搜索 `whatweb-lab`。
27. 写一份误报分析。
28. 完成实验报告。

---

# 78. 复习题

1. WhatWeb 的主要用途是什么？
2. 它是不是完整漏洞扫描器？
3. Level 1 有什么特点？
4. Level 3 什么时候发送额外请求？
5. Level 4 为什么要谨慎？
6. 是否有主要 Level 2？
7. `-v` 有什么作用？
8. `-U` 有什么作用？
9. `-H` 有什么作用？
10. `-c` 有什么作用？
11. 如何控制 Redirect？
12. 如何列出插件？
13. 如何搜索插件？
14. 如何查看插件详情？
15. `-p` 有什么作用？
16. 为什么指定插件更好？
17. `-i` 有什么作用？
18. Brief 和 Verbose 有什么区别？
19. 如何保存 JSON？
20. 如何保存 XML？
21. `-t 1` 有什么作用？
22. `--wait 1` 有什么作用？
23. Server Header 为什么会导致误报？
24. 为什么没有识别到不代表技术不存在？
25. curl 如何验证 WhatWeb？
26. Nmap 如何验证 WhatWeb？
27. tcpdump/Wireshark 能观察什么？
28. Wazuh 为什么不一定自动告警？
29. 为什么不能对真实网站使用 Heavy？
30. 为什么报告需要置信度？

---

# 79. 参考答案

1. 识别 Web 技术指纹。
2. 不是，主要提供技术线索。
3. 请求少、速度快，通常每目标一个请求并处理 Redirect。
4. Level 1 插件匹配后。
5. 请求多、负载高、日志明显。
6. 主要等级中没有。
7. 显示详细插件信息、Header 和证据。
8. 修改 User-Agent。
9. 添加或替换 Header。
10. 提供 Cookie。
11. `--follow-redirect=WHEN`。
12. `whatweb -l`。
13. `whatweb --search-plugins=keyword`。
14. `whatweb -I PluginName`。
15. 选择指定插件。
16. 请求更少，结果更集中。
17. 从文件读取目标。
18. Brief 是摘要，Verbose 是详细报告。
19. `--log-json=FILE`。
20. `--log-xml=FILE`。
21. 只使用一个并发线程。
22. 连接之间等待。
23. Header 可以被伪造。
24. 技术可能被隐藏、动态加载或没有插件。
25. 查看原始 Header、HTML、Cookie 和 Redirect。
26. 验证端口和服务。
27. 请求数量、路径、Header、流量和响应。
28. 需要可收集的日志和检测规则。
29. 未授权且可能增加负载。
30. 指纹是线索，不是绝对证明。

---

# 80. 命令速查表

## 基础

```bash
whatweb http://10.10.20.20:8082/
```

## Verbose

```bash
whatweb -v http://10.10.20.20:8082/
```

## Level 1

```bash
whatweb -a 1 http://10.10.20.20:8082/
```

## Level 3

```bash
whatweb -a 3 http://10.10.20.20:8082/
```

## 受控 Level 4

```bash
whatweb \
  -a 4 \
  -t 1 \
  --wait 0.2 \
  http://10.10.20.20:8082/
```

## 插件列表

```bash
whatweb -l
```

## 搜索插件

```bash
whatweb --search-plugins=wordpress
```

## 插件详情

```bash
whatweb -I WordPress
```

## 指定插件

```bash
whatweb \
  -a 3 \
  -p WordPress \
  http://10.10.20.20:8082/
```

## User-Agent

```bash
whatweb \
  -U 'CyberLab-WhatWeb/1.0' \
  http://10.10.20.20:8082/
```

## Header

```bash
whatweb \
  -H 'X-Lab:WhatWeb' \
  http://10.10.20.20:8082/
```

## Redirect

```bash
whatweb \
  --follow-redirect=never \
  http://10.10.20.20:8082/redirect
```

## 多目标

```bash
whatweb \
  -i ~/CyberLab/WhatWeb/targets/lab-urls.txt
```

## Brief

```bash
whatweb \
  --log-brief=result.txt \
  http://10.10.20.20:8082/
```

## JSON

```bash
whatweb \
  --log-json=result.json \
  http://10.10.20.20:8082/
```

## 低负载

```bash
whatweb \
  -t 1 \
  --wait 1 \
  http://10.10.20.20:8082/
```

---

# 81. 核心记忆口诀

```text
Nmap 找端口
curl 看原始响应
WhatWeb 识别技术
tcpdump/Wireshark 看流量
Wazuh 看日志

-a 1：先轻量
-a 3：有线索再深入
-a 4：只限自己的模拟服务

-v：看证据
-l：列插件
-I：看插件详情
-p：选插件
-i：读目标文件
-t：控线程
--wait：降速度
--log-*：存证据
```

---

# 82. 完成标准

完成本课后，你应该拥有：

```text
SimpleHTTPServer 识别结果
模拟指纹服务器
Level 1 结果
Level 3 结果
一次受控 Level 4 结果
WordPress 插件详情
指定插件结果
Brief 日志
Verbose 日志
JSON 日志
XML 日志
错误日志
Level 1 PCAP
Level 3 PCAP
Access Log
Wazuh whatweb-lab 事件
误报分析
完整实验报告
```

检查 Kali：

```bash
find ~/CyberLab/WhatWeb -type f
```

检查 Ubuntu：

```bash
find ~/CyberLab/WhatWeb -type f
```

---

# 83. 下一课

下一课建议：

```text
Gobuster
```

学习内容：

- Web 目录和文件发现
- Wordlist
- 状态码
- 扩展名
- 并发和延迟
- 误报处理
- tcpdump/Wireshark 请求分析
- Ubuntu Access Log
- Wazuh 观察

当前学习关系：

```text
Nmap：发现端口和服务
Netcat：连接和传输
Wireshark：图形化分析
tcpdump：命令行抓包
curl：手动 HTTP/HTTPS 请求
WhatWeb：自动技术指纹识别
```

---

# 84. 安全边界

只允许识别：

```text
http://10.10.20.20:8080/
http://10.10.20.20:8082/
https://10.10.20.40/ 仅基础受控测试
```

不要识别：

- 学校或公司网站
- 公共服务
- 随机公网 IP
- 邻居设备
- 未经授权的 API
- 不属于你的云服务器
- 大型互联网地址范围

每次实验：

```text
确认授权
先使用 Level 1
限制线程
限制请求
保存日志
验证结果
标明置信度
实验结束后停止服务器
```

---

# 85. 参考说明

命令选项基于：

```text
WhatWeb 官方项目文档
Kali Linux 官方 WhatWeb 工具页面
本机 whatweb -h 输出
```

工具更新后，以本机输出为准：

```bash
whatweb --version
whatweb -h
```
