# 第八课：Nikto 完整学习指南

> **实验环境：你自己的 CyberLab**
>
> 本课程只用于你自己创建、拥有并控制的 Web 服务。  
> 不要扫描学校、公司、公共网站、随机公网 IP 或任何未经明确授权的系统。

---

# 1. 本课目标

学完本课后，你应该能够：

```text
理解 Nikto 的用途、优点和限制
区分信息性结果、错误配置和潜在安全问题
扫描指定主机、端口、URL 和根路径
理解 Nikto 为什么会产生大量明显请求
使用安全的 Tuning 分类缩小扫描范围
列出并选择插件
控制 Redirect、Cookie、Header、User-Agent 和认证
限制请求超时、扫描时间和请求间隔
保存 TXT、HTML、JSON、XML 和 CSV 报告
保存阳性响应并进行人工验证
理解 404、Soft 404 和统一错误页面
使用 curl、Nmap、WhatWeb、Gobuster 和 ffuf 验证结果
使用 tcpdump、Wireshark、Access Log 和 Wazuh 观察扫描
完成一份 Web Server Security Assessment 报告
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
→ HTTP 8084
→ Ubuntu 10.10.20.20
```

---

# 2. 什么是 Nikto？

Nikto 是一个命令行 Web Server Security Scanner。

它会检查：

- Web Server 和组件版本线索
- 默认文件和程序
- 可能危险的文件
- 管理界面
- CGI 目录
- HTTP 方法
- 信息泄露
- 缺失的安全 Header
- 错误配置
- 常见路径和服务器特定问题
- `robots.txt`
- Cookie
- 认证要求
- Server Banner

Nikto 不是自动漏洞利用工具。

更准确地说：

```text
Nikto = 自动化 Web Server 检查清单
      + 已知路径和配置测试
      + 人工验证线索
```

---

# 3. Nikto 的重要特点

Nikto 通常：

- 请求数量较多
- 很容易出现在 Access Log
- 容易被 IDS、IPS、WAF 或 SIEM 观察
- 不追求隐蔽
- 会报告安全问题，也会报告信息性项目
- 可能产生误报
- 结果必须人工验证

记忆：

```text
Nikto 很明显
Nikto 给线索
Nikto 结果必须验证
```

---

# 4. 与前面工具的关系

```text
Nmap：发现 Web 端口和服务
curl：手动构造和验证 HTTP 请求
WhatWeb：识别 Web 技术指纹
Gobuster/ffuf：发现目录、文件、VHost 和参数
Nikto：检查 Web Server 错误配置和危险项目
tcpdump：命令行抓取扫描流量
Wireshark：详细分析请求和响应
Wazuh：查看服务器日志和安全事件
```

推荐顺序：

```text
Nmap
→ curl
→ WhatWeb
→ Gobuster/ffuf
→ Nikto
→ 人工验证
→ 修复
→ 重新扫描
```

---

# 5. Nikto 不是什么？

Nikto 不是：

- 完整的 Web 应用渗透测试
- 自动漏洞利用框架
- 密码破解工具
- 认证绕过证明工具
- SQL 注入专用工具
- XSS 专用工具
- 最终风险评级依据

Nikto 报告某个项目，不代表：

```text
漏洞一定存在
漏洞一定可以利用
影响一定严重
软件版本一定准确
```

---

# 6. 检查和安装

在 Kali：

```bash
which nikto
```

查看版本：

```bash
nikto -Version
```

查看帮助：

```bash
nikto -h
```

如果没有安装：

```bash
sudo apt update
sudo apt install nikto
```

查看软件包版本：

```bash
apt policy nikto
```

工具会更新，最终以本机输出为准：

```bash
nikto -Version
nikto -h
```

---

# 7. 基本语法

```bash
nikto -h TARGET
```

扫描完整 URL：

```bash
nikto -h http://10.10.20.20:8084/
```

分开指定主机和端口：

```bash
nikto \
  -h 10.10.20.20 \
  -p 8084
```

URL 别名：

```bash
nikto -url http://10.10.20.20:8084/
```

本课推荐直接使用完整 URL。

---

# 8. 创建实验目录

Kali：

```bash
mkdir -p ~/CyberLab/Nikto/{results,captures,saved-responses,notes,reports}
```

Ubuntu：

```bash
mkdir -p ~/CyberLab/Nikto/{server,logs}
```

检查：

```bash
find ~/CyberLab/Nikto -maxdepth 2 -type d
```

---

# 9. Ubuntu 受控实验服务器

下面创建一个故意暴露安全线索的服务器。

它会模拟：

```text
过时 Apache Banner
PHP X-Powered-By Header
缺失安全 Header
robots.txt
phpinfo.php
server-status
admin 页面
backup.zip
test 目录
cgi-bin
.git/config
TRACE 方法
宽松 OPTIONS 方法列表
固定 404
```

**重要：**

```text
实际程序是 Python
Apache/PHP Banner 是模拟的
Nikto 版本判断必须验证
```

---

# 10. 创建实验服务器代码

Ubuntu：

```bash
nano ~/CyberLab/Nikto/server/nikto_lab.py
```

粘贴：

```python
#!/usr/bin/env python3

from http.server import BaseHTTPRequestHandler, HTTPServer
from urllib.parse import urlparse


HOST = "10.10.20.20"
PORT = 8084
NOT_FOUND = b"CyberLab Nikto resource not found\n"


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


class NiktoLabHandler(BaseHTTPRequestHandler):
    server_version = "Apache/2.2.22"
    sys_version = "(Ubuntu)"

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
        self.send_header("X-Powered-By", "PHP/5.6.40")
        self.send_header("X-CyberLab", "Nikto-Training")
        self.send_header(
            "Set-Cookie",
            "lab_session=training-only; Path=/",
        )

        if extra_headers:
            for name, value in extra_headers.items():
                self.send_header(name, value)

        self.end_headers()

        if self.command != "HEAD":
            self.wfile.write(body)

    def do_GET(self) -> None:
        path = urlparse(self.path).path

        if path == "/":
            self.send_content(
                200,
                html(
                    "CyberLab Nikto Server",
                    "Controlled Web server security scanning lab.",
                ),
                "text/html; charset=utf-8",
            )
            return

        if path == "/robots.txt":
            self.send_content(
                200,
                (
                    b"User-agent: *\n"
                    b"Disallow: /admin/\n"
                    b"Disallow: /backup/\n"
                    b"Disallow: /test/\n"
                ),
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
                html(
                    "Administration Console",
                    "Training-only administrative page.",
                ),
                "text/html; charset=utf-8",
            )
            return

        if path == "/phpinfo.php":
            self.send_content(
                200,
                html(
                    "phpinfo()",
                    "PHP Version 5.6.40 - simulated output.",
                ),
                "text/html; charset=utf-8",
            )
            return

        if path == "/server-status":
            self.send_content(
                200,
                b"Apache Server Status for CyberLab\n",
            )
            return

        if path == "/backup.zip":
            self.send_content(
                200,
                b"CYBERLAB BACKUP PLACEHOLDER\n",
                "application/zip",
            )
            return

        if path in {"/test", "/test/"}:
            self.send_content(
                200,
                html(
                    "Test Directory",
                    "Temporary test content should not be public.",
                ),
                "text/html; charset=utf-8",
            )
            return

        if path in {"/cgi-bin", "/cgi-bin/"}:
            self.send_content(
                200,
                b"CyberLab CGI directory placeholder\n",
            )
            return

        if path == "/.git/config":
            self.send_content(
                200,
                (
                    b"[core]\n"
                    b"repositoryformatversion = 0\n"
                    b"# CyberLab simulated Git config\n"
                ),
            )
            return

        if path == "/README":
            self.send_content(
                200,
                b"CyberLab Nikto training README\n",
            )
            return

        if path == "/login":
            self.send_content(
                401,
                b"Authentication required\n",
                extra_headers={
                    "WWW-Authenticate": 'Basic realm="CyberLab"',
                },
            )
            return

        self.send_content(404, NOT_FOUND)

    def do_HEAD(self) -> None:
        self.do_GET()

    def do_OPTIONS(self) -> None:
        self.send_response(204)
        self.send_header(
            "Allow",
            "GET, HEAD, POST, OPTIONS, TRACE, PUT, DELETE",
        )
        self.send_header("Content-Length", "0")
        self.end_headers()

    def do_TRACE(self) -> None:
        request_line = (
            f"{self.command} {self.path} {self.request_version}\r\n"
        )
        header_lines = "".join(
            f"{name}: {value}\r\n"
            for name, value in self.headers.items()
        )
        body = (request_line + header_lines + "\r\n").encode(
            "utf-8",
            errors="replace",
        )
        self.send_content(200, body, "message/http")

    def do_POST(self) -> None:
        self.send_content(200, b"CyberLab POST received\n")

    def do_PUT(self) -> None:
        self.send_content(
            403,
            b"PUT is listed for testing but writes are disabled\n",
        )

    def do_DELETE(self) -> None:
        self.send_content(
            403,
            b"DELETE is listed for testing but deletion is disabled\n",
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
    server = HTTPServer((HOST, PORT), NiktoLabHandler)
    print(f"Nikto lab listening on http://{HOST}:{PORT}")
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
chmod +x ~/CyberLab/Nikto/server/nikto_lab.py
```

---

# 11. 启动服务器并保存日志

```bash
python3 ~/CyberLab/Nikto/server/nikto_lab.py \
  2>&1 |
tee ~/CyberLab/Nikto/logs/access.log
```

预期：

```text
Nikto lab listening on http://10.10.20.20:8084
```

第二个 Ubuntu 终端：

```bash
ss -lntp | grep ':8084'
```

---

# 12. Kali 扫描前验证

```bash
ping -c 3 10.10.20.20
```

```bash
nc -zvn -w 3 10.10.20.20 8084
```

```bash
curl -i http://10.10.20.20:8084/
```

```bash
curl -i http://10.10.20.20:8084/robots.txt
```

```bash
curl -i -X OPTIONS http://10.10.20.20:8084/
```

```bash
sudo nmap -n -sV -p 8084 10.10.20.20
```

```bash
whatweb http://10.10.20.20:8084/
```

---

# 13. 第一次基础扫描

```bash
nikto \
  -h http://10.10.20.20:8084/
```

Nikto 通常输出：

- 目标 IP
- 主机名
- 端口
- 开始时间
- Server Header
- 发现项目
- 请求数量
- 错误数量
- 结束时间

---

# 14. 可能看到的实验结果

实际结果由本机 Nikto 版本和数据库决定，可能包括：

```text
Server Banner
Outdated Apache indication
X-Powered-By
Missing X-Frame-Options
Missing X-Content-Type-Options
Cookie attributes
Allowed HTTP Methods
TRACE method
robots.txt entries
/admin/
/phpinfo.php
/server-status
/backup.zip
/.git/config
/test/
/cgi-bin/
Authentication-required URL
```

每个版本不一定报告完全相同的项目。

---

# 15. 结果分类

## 信息性

```text
Server Banner
X-Powered-By
Cookie
页面标题
HTTP 方法
```

## 错误配置

```text
缺失安全 Header
TRACE Enabled
公开 server-status
公开测试文件
```

## 潜在风险

```text
公开备份
公开 phpinfo
公开 Git 配置
暴露管理界面
```

## 需要人工确认

几乎所有发现都需要：

```text
curl
浏览器
服务器配置
真实进程
业务上下文
```

---

# 16. Display 输出控制

进度：

```bash
nikto \
  -Display P \
  -h http://10.10.20.20:8084/
```

Verbose：

```bash
nikto \
  -Display V \
  -h http://10.10.20.20:8084/
```

Redirect：

```bash
nikto \
  -Display 1 \
  -h http://10.10.20.20:8084/
```

Cookie：

```bash
nikto \
  -Display 2 \
  -h http://10.10.20.20:8084/
```

认证 URL：

```bash
nikto \
  -Display 4 \
  -h http://10.10.20.20:8084/
```

组合：

```bash
nikto \
  -Display 124P \
  -h http://10.10.20.20:8084/
```

---

# 17. 非交互和更新检查

非交互：

```bash
nikto \
  -nointeractive \
  -h http://10.10.20.20:8084/
```

禁止启动时更新检查：

```bash
nikto \
  -nocheck \
  -h http://10.10.20.20:8084/
```

你的 CyberLab 没有互联网时推荐使用：

```text
-nocheck
-nointeractive
```

---

# 18. Timeout 和 Max Time

单请求超时：

```bash
nikto \
  -timeout 5 \
  -h http://10.10.20.20:8084/
```

每台主机总时间：

```bash
nikto \
  -maxtime 5m \
  -h http://10.10.20.20:8084/
```

常见格式：

```text
60s
5m
1h
```

---

# 19. 请求之间暂停

```bash
nikto \
  -Pause 1 \
  -Tuning 123be \
  -h http://10.10.20.20:8084/
```

每个测试之间暂停约 1 秒。

用途：

```text
降低实验负载
方便观察日志和 PCAP
```

不用于隐藏扫描。

---

# 20. User-Agent、Header 和 Cookie

自定义 User-Agent：

```bash
nikto \
  -useragent 'CyberLab-Nikto/1.0' \
  -h http://10.10.20.20:8084/
```

添加 Header：

```bash
nikto \
  -Add-header 'X-Lab-Name: Nikto' \
  -Add-header 'X-Student: CyberLab' \
  -h http://10.10.20.20:8084/
```

显示 Cookie：

```bash
nikto \
  -Display 2 \
  -h http://10.10.20.20:8084/
```

禁止使用响应 Cookie：

```bash
nikto \
  -nocookies \
  -h http://10.10.20.20:8084/
```

---

# 21. Basic Authentication

```bash
nikto \
  -id 'labuser:labpassword' \
  -h http://10.10.20.20:8084/
```

也支持：

```text
id:pass:realm
```

当前 `/login` 只返回认证挑战，不实现完整登录。

安全规则：

```text
Basic Auth 应配合 HTTPS
只使用测试账号
命令行密码可能进入 Shell History
```

---

# 22. Port、Root、VHost 和 Redirect

指定端口：

```bash
nikto \
  -h 10.10.20.20 \
  -p 8084
```

指定根路径：

```bash
nikto \
  -root /app \
  -h http://10.10.20.20:8084/
```

指定 Host Header：

```bash
nikto \
  -vhost lab.local \
  -h 10.10.20.20 \
  -p 8084
```

跟随 3xx：

```bash
nikto \
  -followredirects \
  -h http://10.10.20.20:8084/
```

---

# 23. HTTPS

强制 SSL：

```bash
nikto \
  -ssl \
  -h 10.10.20.40 \
  -p 443
```

或：

```bash
nikto \
  -h https://10.10.20.40/
```

Wazuh 可能使用自签名证书。TLS 问题优先使用：

```bash
curl -vkI https://10.10.20.40/
```

```bash
sudo nmap -n \
  -p 443 \
  --script ssl-cert \
  10.10.20.40
```

---

# 24. Tuning 类别

| Code | 类别 |
|---|---|
| `1` | Interesting File / Seen in Logs |
| `2` | Misconfiguration / Default File |
| `3` | Information Disclosure |
| `4` | Injection (XSS/Script/HTML) |
| `5` | Remote File Retrieval — Web Root 内 |
| `6` | Denial of Service |
| `7` | Remote File Retrieval — Server Wide |
| `8` | Command Execution / Remote Shell |
| `9` | SQL Injection |
| `0` | File Upload |
| `a` | Authentication Bypass |
| `b` | Software Identification |
| `c` | Remote Source Inclusion |
| `d` | Web Service |
| `e` | Administrative Console |
| `x` | Reverse Tuning |

---

# 25. 本课程使用的安全 Tuning

本课程优先：

```text
1  Interesting Files
2  Misconfiguration
3  Information Disclosure
b  Software Identification
e  Administrative Console
```

命令：

```bash
nikto \
  -Tuning 123be \
  -h http://10.10.20.20:8084/
```

---

# 26. 本课程不运行的类别

不运行或不单独练习：

```text
6  Denial of Service
8  Command Execution / Remote Shell
9  SQL Injection
0  File Upload
a  Authentication Bypass
c  Remote Source Inclusion
```

本课重点是：

```text
配置检查
信息验证
日志观察
修复复测
```

---

# 27. CGI 和插件

不扫描 CGI：

```bash
nikto \
  -Cgidirs none \
  -h http://10.10.20.20:8084/
```

指定 CGI：

```bash
nikto \
  -Cgidirs '/cgi-bin/' \
  -h http://10.10.20.20:8084/
```

列出插件：

```bash
nikto -list-plugins
```

选择插件：

```bash
nikto \
  -Plugins PLUGIN_NAME \
  -h http://10.10.20.20:8084/
```

插件名称随版本变化，先复制本机列表中的准确名称。

---

# 28. 版本和数据库检查

```bash
nikto -Version
```

记录：

```text
Nikto 版本
插件版本
数据库版本
扫描日期
```

数据库检查：

```bash
nikto -dbcheck
```

---

# 29. 404 和 Soft 404

正常随机路径：

```bash
curl -i \
  http://10.10.20.20:8084/random-invalid-path
```

应该返回：

```text
404
CyberLab Nikto resource not found
```

不要随意使用：

```bash
-no404
```

否则可能增加假阳性。

---

# 30. 自定义错误响应

忽略特定负面状态码：

```bash
nikto \
  -404code '301,302' \
  -h http://10.10.20.20:8084/
```

指定错误响应字符串：

```bash
nikto \
  -404string 'resource not found' \
  -h http://10.10.20.20:8084/
```

使用前先用 curl 测试多个随机路径，确认行为稳定。

---

# 31. Soft 404 处理流程

```text
1. 请求两个随机路径
2. 比较 Status
3. 比较 Body
4. 比较 Length
5. 找出稳定错误字符串
6. 使用 -404string
7. 重新扫描
8. 人工复查
```


---

# 32. 保存 TXT 报告

```bash
nikto \
  -h http://10.10.20.20:8084/ \
  -o ~/CyberLab/Nikto/results/nikto-report.txt \
  -Format txt
```

---

# 33. 保存 HTML 报告

```bash
nikto \
  -h http://10.10.20.20:8084/ \
  -o ~/CyberLab/Nikto/results/nikto-report.html \
  -Format htm
```

打开：

```bash
xdg-open \
  ~/CyberLab/Nikto/results/nikto-report.html
```

---

# 34. 保存 JSON、XML 和 CSV

JSON：

```bash
nikto \
  -h http://10.10.20.20:8084/ \
  -o ~/CyberLab/Nikto/results/nikto-report.json \
  -Format json
```

查看：

```bash
head -n 40 \
  ~/CyberLab/Nikto/results/nikto-report.json
```

XML：

```bash
nikto \
  -h http://10.10.20.20:8084/ \
  -o ~/CyberLab/Nikto/results/nikto-report.xml \
  -Format xml
```

CSV：

```bash
nikto \
  -h http://10.10.20.20:8084/ \
  -o ~/CyberLab/Nikto/results/nikto-report.csv \
  -Format csv
```

---

# 35. 保存阳性响应

```bash
nikto \
  -Save ~/CyberLab/Nikto/saved-responses \
  -h http://10.10.20.20:8084/
```

检查：

```bash
find ~/CyberLab/Nikto/saved-responses \
  -type f
```

保存内容可能包含：

- Header
- Cookie
- 页面正文
- 内部路径
- 敏感信息

不要上传到公开 GitHub。

---

# 36. 推荐基线命令

```bash
nikto \
  -nocheck \
  -nointeractive \
  -Display 124P \
  -timeout 5 \
  -maxtime 5m \
  -Tuning 123be \
  -useragent 'CyberLab-Nikto/1.0' \
  -Add-header 'X-Lab-Name: Nikto' \
  -h http://10.10.20.20:8084/ \
  -o ~/CyberLab/Nikto/results/baseline.txt \
  -Format txt
```

---

# 37. 本课不使用 Evasion

Nikto 支持请求编码和 anti-IDS 形式。

本课程不使用：

```text
-evasion
```

原因：

- 本课不是规避检测
- 目标是观察正常扫描流量
- 保持日志和 PCAP 易于理解
- 未授权规避检测不合适

---

# 38. 本课不使用 Mutation

Nikto 支持 `-mutate` 猜测：

- 用户目录
- 密码文件名
- 子域名
- 目录名

本课程不运行。

目录和文件发现已经使用受控的：

```text
Gobuster / ffuf
```

完成。

---

# 39. 使用 curl 验证 Header

```bash
curl -i \
  http://10.10.20.20:8084/
```

检查：

```text
Server
X-Powered-By
X-Frame-Options
Content-Security-Policy
X-Content-Type-Options
Set-Cookie
```

---

# 40. 验证 robots.txt

```bash
curl -i \
  http://10.10.20.20:8084/robots.txt
```

重要：

```text
robots.txt 不是访问控制
```

它只告诉搜索引擎爬虫哪些路径不应抓取。

---

# 41. 验证管理页面

```bash
curl -i \
  http://10.10.20.20:8084/admin
```

跟随 Redirect：

```bash
curl -i -L \
  http://10.10.20.20:8084/admin
```

---

# 42. 验证 phpinfo 和 server-status

```bash
curl -i \
  http://10.10.20.20:8084/phpinfo.php
```

```bash
curl -i \
  http://10.10.20.20:8084/server-status
```

真实环境中，这些页面可能泄露：

- 软件版本
- 路径
- 环境变量
- 模块
- Worker 状态
- 客户端 IP
- 请求信息

当前内容是模拟的。

---

# 43. 验证备份和 Git 配置

只查看备份 Header：

```bash
curl -I \
  http://10.10.20.20:8084/backup.zip
```

验证 Git 配置：

```bash
curl -i \
  http://10.10.20.20:8084/.git/config
```

真实 `.git` 暴露可能泄露：

- Repository URL
- Branch
- 配置
- 源代码恢复线索

---

# 44. 验证 HTTP 方法

OPTIONS：

```bash
curl -i \
  -X OPTIONS \
  http://10.10.20.20:8084/
```

TRACE：

```bash
curl -i \
  -X TRACE \
  -H 'X-Lab-Trace: Nikto' \
  http://10.10.20.20:8084/
```

PUT：

```bash
curl -i \
  -X PUT \
  -d 'training-data' \
  http://10.10.20.20:8084/test-file
```

DELETE：

```bash
curl -i \
  -X DELETE \
  http://10.10.20.20:8084/test-file
```

本实验：

```text
PUT    → 403
DELETE → 403
```

所以：

```text
OPTIONS 列出某方法
≠
该方法真正可执行
```

---

# 45. 版本判断必须验证

Nikto 可能报告：

```text
Apache/2.2.22
PHP/5.6.40
```

Ubuntu 验证：

```bash
sudo ss -lntp | grep ':8084'
```

```bash
ps aux |
grep nikto_lab |
grep -v grep
```

真实程序是：

```text
Python
```

正确结论：

```text
Server Header 声称 Apache/2.2.22
后端进程验证为 Python
Apache 版本识别属于模拟误报
```

---

# 46. 验证缺失安全 Header

```bash
curl -s -D - \
  http://10.10.20.20:8084/ \
  -o /dev/null
```

检查：

```text
Content-Security-Policy
X-Frame-Options
X-Content-Type-Options
Strict-Transport-Security
Referrer-Policy
Permissions-Policy
```

注意：

- HSTS 只适用于 HTTPS
- CSP 需要按应用设计
- Header 缺失不自动等于高危
- 需要结合业务和浏览器行为判断

---

# 47. 验证 Cookie

```bash
curl -i \
  http://10.10.20.20:8084/
```

观察：

```text
Set-Cookie: lab_session=training-only; Path=/
```

缺少：

```text
Secure
HttpOnly
SameSite
```

判断时考虑：

- 是否使用 HTTPS
- Cookie 是否包含敏感会话
- JavaScript 是否需要读取
- SameSite 业务需求

---

# 48. 与 WhatWeb 比较

```bash
whatweb -v \
  http://10.10.20.20:8084/
```

| WhatWeb | Nikto |
|---|---|
| 技术指纹 | 安全检查和已知路径 |
| 请求通常较少 | 请求通常较多 |
| 发现“使用什么” | 发现“可能有什么问题” |
| Header/HTML 识别 | 配置、方法、默认文件 |

---

# 49. 与 Gobuster / ffuf 比较

Gobuster/ffuf：

```text
使用 Wordlist 发现内容
```

Nikto：

```text
使用内置数据库和插件检查安全问题
```

| Gobuster/ffuf | Nikto |
|---|---|
| Wordlist 驱动 | 数据库和插件驱动 |
| 强大路径发现 | 安全发现描述 |
| 可自定义输入 | 内置检查类别 |
| 适合枚举内容 | 适合 Web Server 审计 |

---

# 50. tcpdump 实时观察

Kali 终端一：

```bash
sudo tcpdump \
  -i eth0 \
  -nn \
  -A \
  'host 10.10.20.20 and tcp port 8084'
```

Kali 终端二：

```bash
nikto \
  -nocheck \
  -Tuning 123b \
  -useragent 'CyberLab-Nikto-PCAP/1.0' \
  -h http://10.10.20.20:8084/
```

可以看到：

- GET
- HEAD
- OPTIONS
- 不同路径
- User-Agent
- Status
- Server Header
- Cookie

---

# 51. 保存 Nikto PCAP

```bash
sudo tcpdump \
  -i eth0 \
  -nn \
  -s 0 \
  -w ~/CyberLab/Nikto/captures/nikto-baseline.pcap \
  'host 10.10.20.20 and tcp port 8084'
```

另一个终端：

```bash
nikto \
  -nocheck \
  -nointeractive \
  -Tuning 123be \
  -useragent 'CyberLab-Nikto/1.0' \
  -h http://10.10.20.20:8084/
```

结束后停止 tcpdump：

```text
Ctrl + C
```

---

# 52. Wireshark 分析

```bash
wireshark \
  ~/CyberLab/Nikto/captures/nikto-baseline.pcap
```

所有请求：

```text
http.request
```

Nikto User-Agent：

```text
http.user_agent contains "Nikto"
```

目标：

```text
ip.addr == 10.10.20.20 &&
tcp.port == 8084
```

状态码：

```text
http.response.code == 200
```

```text
http.response.code == 301
```

```text
http.response.code == 401
```

```text
http.response.code == 404
```

---

# 53. Wireshark 统计

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

- HTTP 请求数
- TCP 会话数
- 不同 URI
- 200、301、401、404 数量
- 总数据包
- 总字节
- 扫描时间

---

# 54. Ubuntu Access Log

查看：

```bash
less ~/CyberLab/Nikto/logs/access.log
```

只看 Kali：

```bash
grep '10.10.20.10' \
  ~/CyberLab/Nikto/logs/access.log
```

只看 Nikto：

```bash
grep -i 'nikto' \
  ~/CyberLab/Nikto/logs/access.log
```

统计 GET：

```bash
grep -c '"GET ' \
  ~/CyberLab/Nikto/logs/access.log
```

统计方法：

```bash
grep -Eo '"(GET|HEAD|OPTIONS|TRACE|POST|PUT|DELETE) ' \
  ~/CyberLab/Nikto/logs/access.log |
sort |
uniq -c
```

---

# 55. Wazuh 日志链路

Ubuntu 扫描前：

```bash
logger -t nikto-lab \
  "Starting authorized Nikto scan from Kali 10.10.20.10"
```

扫描后：

```bash
logger -t nikto-lab \
  "Completed authorized Nikto scan against TCP 8084"
```

本地：

```bash
journalctl \
  -t nikto-lab \
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
nikto-lab
```

---

# 56. Nikto 是否一定触发 Wazuh 告警？

不一定。

自动检测 Nikto 通常需要：

- Web Access Log
- Wazuh Agent 收集日志
- User-Agent 或请求行为规则
- WAF
- IDS/IPS
- Suricata
- Zeek

本课成功标准：

```text
Access Log 有请求
PCAP 有流量
手工 nikto-lab 日志进入 Wazuh
```

---

# 57. 发现项验证模板

```text
Finding：
Nikto Evidence：
Manual Request：
Manual Response：
Server-side Evidence：
True Positive / False Positive：
Risk：
Recommendation：
Retest：
```

---

# 58. 示例：Server Banner

```text
Finding:
Server Header discloses Apache/2.2.22 (Ubuntu)

Nikto Evidence:
Nikto reported the Server header.

Manual Verification:
curl -I http://10.10.20.20:8084/

Server-side Evidence:
ss and ps show Python running nikto_lab.py.

Conclusion:
Header disclosure is true.
Actual Apache version identification is a simulated false positive.

Confidence:
High for Header disclosure.
Low for actual Apache version.
```

---

# 59. 示例：phpinfo.php

```text
Finding:
Public phpinfo-like page

Manual Verification:
curl -i http://10.10.20.20:8084/phpinfo.php

Conclusion:
The URL is publicly accessible.
The content is simulated.

Recommendation:
Remove diagnostic pages from production.
```

---

# 60. 示例：TRACE

```text
Finding:
HTTP TRACE is enabled

Verification:
curl -i -X TRACE \
  -H 'X-Lab-Trace: Nikto' \
  http://10.10.20.20:8084/

Recommendation:
Disable TRACE unless explicitly required.
```

---

# 61. 风险等级不要只靠工具

Nikto 不知道：

- 页面是否对公网开放
- 数据是否敏感
- 是否有反向代理
- 用户角色
- 网络分段
- 真实软件版本
- 补偿控制
- 业务影响

风险评估需要：

```text
Evidence
Likelihood
Impact
Exposure
Business Context
Existing Controls
```

---

# 62. 修复优先级示例

## 高优先级

- 真实敏感备份公开
- 真实 `.git` 暴露
- 管理页面无认证
- 危险写入方法实际可用
- 明确可利用的漏洞

## 中优先级

- phpinfo 公开
- server-status 公开
- TRACE 无必要启用
- 管理页面暴露范围过大

## 低或信息性

- Server Header
- X-Powered-By
- 非敏感 robots.txt
- 部分 Header 缺失

最终等级依赖环境。

---

# 63. 修复前后对比

修复前：

```bash
nikto \
  -nocheck \
  -nointeractive \
  -Tuning 123be \
  -h http://10.10.20.20:8084/ \
  -o ~/CyberLab/Nikto/results/before.txt \
  -Format txt
```

修复后：

```bash
nikto \
  -nocheck \
  -nointeractive \
  -Tuning 123be \
  -h http://10.10.20.20:8085/ \
  -o ~/CyberLab/Nikto/results/after.txt \
  -Format txt
```

比较：

```bash
diff -u \
  ~/CyberLab/Nikto/results/before.txt \
  ~/CyberLab/Nikto/results/after.txt
```


---

# 64. 安全修复练习

复制脚本：

```bash
cp \
  ~/CyberLab/Nikto/server/nikto_lab.py \
  ~/CyberLab/Nikto/server/nikto_lab_hardened.py
```

在 Hardened 版本中：

```text
端口改为 8085
移除 Apache/PHP 模拟 Banner
添加 X-Content-Type-Options
添加 X-Frame-Options 或 CSP
改善 Cookie 属性
禁用 TRACE
减少 OPTIONS Allow
删除 phpinfo.php
删除 server-status
删除 backup.zip
删除 .git/config
保护 /admin/
```

修改前先克隆 Ubuntu VM 或备份脚本。

---

# 65. 完整实验一：基础扫描

Ubuntu：

```bash
python3 ~/CyberLab/Nikto/server/nikto_lab.py \
  2>&1 |
tee ~/CyberLab/Nikto/logs/access.log
```

Kali：

```bash
nikto \
  -nocheck \
  -nointeractive \
  -Display 124P \
  -h http://10.10.20.20:8084/
```

记录：

```text
请求数量
错误数量
发现数量
扫描时间
```

---

# 66. 完整实验二：安全 Tuning

```bash
nikto \
  -nocheck \
  -nointeractive \
  -Tuning 123be \
  -timeout 5 \
  -maxtime 5m \
  -h http://10.10.20.20:8084/ \
  -o ~/CyberLab/Nikto/results/lab2-tuned.txt \
  -Format txt
```

比较：

| 项目 | 完整扫描 | `123be` |
|---|---:|---:|
| 请求数 |  |  |
| 时间 |  |  |
| 发现数 |  |  |
| 主要差异 |  |  |

---

# 67. 完整实验三：多格式报告

TXT：

```bash
nikto \
  -nocheck \
  -Tuning 123be \
  -h http://10.10.20.20:8084/ \
  -o ~/CyberLab/Nikto/results/lab3.txt \
  -Format txt
```

HTML：

```bash
nikto \
  -nocheck \
  -Tuning 123be \
  -h http://10.10.20.20:8084/ \
  -o ~/CyberLab/Nikto/results/lab3.html \
  -Format htm
```

JSON：

```bash
nikto \
  -nocheck \
  -Tuning 123be \
  -h http://10.10.20.20:8084/ \
  -o ~/CyberLab/Nikto/results/lab3.json \
  -Format json
```

XML：

```bash
nikto \
  -nocheck \
  -Tuning 123be \
  -h http://10.10.20.20:8084/ \
  -o ~/CyberLab/Nikto/results/lab3.xml \
  -Format xml
```

---

# 68. 完整实验四：手工验证

验证以下项目：

```bash
curl -i http://10.10.20.20:8084/robots.txt
```

```bash
curl -i http://10.10.20.20:8084/phpinfo.php
```

```bash
curl -i http://10.10.20.20:8084/server-status
```

```bash
curl -I http://10.10.20.20:8084/backup.zip
```

```bash
curl -i http://10.10.20.20:8084/.git/config
```

```bash
curl -i -X OPTIONS http://10.10.20.20:8084/
```

```bash
curl -i -X TRACE http://10.10.20.20:8084/
```

每项标记：

```text
TP   True Positive
FP   False Positive
INFO Informational
NMC  Needs More Context
```

---

# 69. 完整实验五：流量与日志

Ubuntu：

```bash
logger -t nikto-lab \
  "Starting Nikto traffic capture lab"
```

Kali 终端一：

```bash
sudo tcpdump \
  -i eth0 \
  -nn \
  -s 0 \
  -w ~/CyberLab/Nikto/captures/lab5-nikto.pcap \
  'host 10.10.20.20 and tcp port 8084'
```

Kali 终端二：

```bash
nikto \
  -nocheck \
  -nointeractive \
  -Tuning 123be \
  -useragent 'CyberLab-Nikto-Lab5/1.0' \
  -h http://10.10.20.20:8084/
```

扫描完成后停止 tcpdump。

Ubuntu：

```bash
logger -t nikto-lab \
  "Completed Nikto traffic capture lab"
```

Wireshark：

```bash
wireshark \
  ~/CyberLab/Nikto/captures/lab5-nikto.pcap
```

Wazuh 搜索：

```text
nikto-lab
```

---

# 70. 完整实验六：修复与复测

Hardened Server 使用端口：

```text
8085
```

扫描原始服务：

```bash
nikto \
  -nocheck \
  -Tuning 123be \
  -h http://10.10.20.20:8084/ \
  -o ~/CyberLab/Nikto/results/vulnerable.txt \
  -Format txt
```

扫描 Hardened 服务：

```bash
nikto \
  -nocheck \
  -Tuning 123be \
  -h http://10.10.20.20:8085/ \
  -o ~/CyberLab/Nikto/results/hardened.txt \
  -Format txt
```

比较：

```bash
diff -u \
  ~/CyberLab/Nikto/results/vulnerable.txt \
  ~/CyberLab/Nikto/results/hardened.txt
```

回答：

```text
哪些发现消失？
哪些仍然存在？
是否产生新问题？
是否需要手工复测？
```

---

# 71. 实验报告模板

## Lab Information

```text
Lab Name：Nikto Web Server Security Assessment
Date：
Student：
Scanner：Kali-CyberLab 10.10.20.10
Target：Ubuntu-Target 10.10.20.20
Service：http://10.10.20.20:8084/
Authorization：Personal CyberLab
Nikto Version：
Database Version：
```

## Objective

```text
使用 Nikto 检查 Web Server 配置、默认文件、信息泄露和管理界面，
并使用 curl、服务器证据、PCAP 和 Wazuh 验证结果。
```

## Pre-Scan Verification

```bash
# Ping
# nc
# Nmap
# curl
# WhatWeb
```

## Scan Configuration

```text
Host：
Port：
Tuning：
Plugins：
Timeout：
Max Time：
Pause：
User-Agent：
Output Format：
```

## Commands

```bash
# Baseline
# Tuned
# Reports
# PCAP
```

## Findings

| ID | Finding | Nikto Evidence | Manual Evidence | Result | Risk |
|---:|---|---|---|---|---|
| 1 |  |  |  |  |  |
| 2 |  |  |  |  |  |

## Validation Status

```text
TP   True Positive
FP   False Positive
INFO Informational
NMC  Needs More Context
```

## Traffic Statistics

| Metric | Value |
|---|---:|
| HTTP Requests |  |
| TCP Conversations |  |
| 200 Responses |  |
| 301 Responses |  |
| 401 Responses |  |
| 404 Responses |  |
| Duration |  |

## Evidence

```text
Nikto TXT：
Nikto HTML：
Nikto JSON：
Saved Responses：
curl Output：
Server Process：
Access Log：
PCAP：
Wireshark：
Wazuh Event：
```

## Recommendations

```text
移除公开诊断和备份文件。
禁用不需要的 HTTP 方法。
保护管理界面。
减少不必要的版本泄露。
配置适当的安全 Header。
改善 Cookie 属性。
维护 Web Access Log 和检测规则。
```

## Retest

```text
Fix Applied：
Retest Date：
Finding Removed：
Finding Remaining：
New Issue：
```

## Lessons Learned

```text
Nikto 的最大优点：
Nikto 的最大限制：
最重要的验证方法：
```

---

# 72. 常见错误与解决方法

## command not found

```bash
sudo apt update
sudo apt install nikto
```

## Connection refused

```bash
nc -zvn -w 3 10.10.20.20 8084
```

Ubuntu：

```bash
ss -lntp | grep ':8084'
```

## Timeout

```bash
nikto \
  -timeout 10 \
  -h http://10.10.20.20:8084/
```

检查：

- UTM 网卡
- 防火墙
- 服务进程
- URL
- 端口

## 扫描时间太长

使用：

```text
-Tuning 123be
-maxtime 5m
-Cgidirs none
```

## 假阳性很多

检查：

- 404 行为
- Soft 404
- Redirect
- Banner 欺骗
- 统一错误页面

使用：

```text
-404string
-404code
curl 人工验证
```

## 报告文件为空

检查：

```bash
ls -lh ~/CyberLab/Nikto/results
```

确认：

- 目录存在
- 有写入权限
- `-o` 正确
- `-Format` 正确
- 扫描没有过早中止

## TLS 错误

```bash
curl -vkI https://TARGET/
```

确认：

- TLS 版本
- 证书
- 端口
- 主机名
- Nikto 版本

## 数据库错误

```bash
nikto -dbcheck
```

---

# 73. 为什么结果会变化？

Nikto 的：

- 扫描数据库
- 插件
- 版本规则
- Header 检查
- 输出格式

会更新。

所以报告必须记录：

```text
Nikto Version
Database Version
Scan Date
Command
Target
```

---

# 74. 不应公开的文件

不要公开上传：

- 未清理的 HTML 报告
- JSON/XML 报告
- Saved Responses
- Access Log
- PCAP
- Cookie
- 认证信息
- 真实域名和内部路径

上传 GitHub 前检查：

```bash
grep -RniE \
  'password|token|cookie|authorization|session' \
  ~/CyberLab/Nikto
```

---

# 75. 安全边界

只允许扫描：

```text
http://10.10.20.20:8084/
可选 Hardened：
http://10.10.20.20:8085/
```

不要扫描：

- 学校或公司网站
- 公共服务
- 随机公网 IP
- 邻居设备
- 未经授权的 API
- 不属于你的云服务器
- 大型互联网范围

不要使用：

```text
-evasion
DoS Tuning
Mutation 暴力猜测
规避 IDS/WAF
隐藏来源
```

---

# 76. 本课练习

1. 检查 Nikto 版本和帮助。
2. 创建实验目录。
3. 启动 Ubuntu 8084 实验服务器。
4. 使用 curl 验证首页。
5. 使用 Nmap 验证端口。
6. 使用 WhatWeb 查看指纹。
7. 运行基础扫描。
8. 使用 `-Display 124P`。
9. 使用 `-nointeractive`。
10. 使用 `-nocheck`。
11. 使用 `-timeout 5`。
12. 使用 `-maxtime 5m`。
13. 使用 `-Tuning 123be`。
14. 修改 User-Agent。
15. 添加 Header。
16. 显示 Cookie。
17. 列出插件。
18. 运行 `-dbcheck`。
19. 保存 TXT。
20. 保存 HTML。
21. 保存 JSON。
22. 保存 XML。
23. 保存阳性响应。
24. 验证 `robots.txt`。
25. 验证 `/admin/`。
26. 验证 `/phpinfo.php`。
27. 验证 `/server-status`。
28. 验证 `/backup.zip`。
29. 验证 `/.git/config`。
30. 验证 OPTIONS。
31. 验证 TRACE。
32. 验证 PUT/DELETE。
33. 使用 `ss` 和 `ps` 证明 Banner 误报。
34. 保存 Nikto PCAP。
35. 使用 Wireshark 统计请求。
36. 查看 Ubuntu Access Log。
37. 在 Wazuh 搜索 `nikto-lab`。
38. 对发现标记 TP/FP/INFO/NMC。
39. 创建 Hardened Server。
40. 修复后重新扫描并比较。

---

# 77. 复习题

1. Nikto 的主要用途是什么？
2. Nikto 是隐蔽工具吗？
3. Nikto 结果一定是真实漏洞吗？
4. `-h` 做什么？
5. `-p` 做什么？
6. `-Display P` 做什么？
7. `-nointeractive` 做什么？
8. `-nocheck` 做什么？
9. `-timeout` 与 `-maxtime` 有何区别？
10. `-Pause` 做什么？
11. `-Tuning 123be` 包含哪些类别？
12. 为什么不使用 Tuning 6？
13. 为什么不使用 `-evasion`？
14. `-Cgidirs none` 做什么？
15. 如何列出插件？
16. `-Plugins` 做什么？
17. `-dbcheck` 做什么？
18. `-404string` 有什么用途？
19. 为什么不应随意使用 `-no404`？
20. `-o` 与 `-Format` 有什么区别？
21. `-Save` 做什么？
22. 为什么 Saved Responses 可能敏感？
23. Server Header 为什么可能误报？
24. OPTIONS 列出 PUT 是否证明 PUT 可用？
25. 如何验证 TRACE？
26. robots.txt 是访问控制吗？
27. 为什么公开 phpinfo 有风险？
28. 为什么 `.git/config` 暴露有风险？
29. Nikto 与 WhatWeb 有何区别？
30. Nikto 与 Gobuster/ffuf 有何区别？
31. tcpdump/Wireshark 能证明什么？
32. Wazuh 为什么不一定自动告警？
33. 为什么需要修复后复测？
34. 为什么报告要记录版本和日期？
35. 为什么不能扫描未授权网站？

---

# 78. 参考答案

1. 自动检查 Web Server 错误配置、危险文件和已知问题线索。
2. 不是，它通常很明显。
3. 不一定，必须人工验证。
4. 指定目标。
5. 指定端口。
6. 显示进度。
7. 禁用交互功能。
8. 禁止启动时更新检查。
9. 前者限制单请求，后者限制每台主机总时间。
10. 在测试之间暂停。
11. Interesting Files、Misconfiguration、Information Disclosure、Software Identification、Administrative Console。
12. 它属于 Denial of Service。
13. 本课不学习规避检测。
14. 不扫描 CGI 目录。
15. `nikto -list-plugins`。
16. 选择要运行的插件。
17. 检查数据库和关键文件语法。
18. 把匹配字符串作为负面错误响应。
19. 可能增加假阳性。
20. `-o` 指定文件，`-Format` 指定格式。
21. 保存阳性响应。
22. 可能包含 Cookie、Header、页面和内部路径。
23. Header 可以被模拟、代理或管理员修改。
24. 不证明，必须实际请求。
25. `curl -X TRACE`。
26. 不是。
27. 可能泄露版本、路径和配置。
28. 可能泄露仓库和源代码信息。
29. WhatWeb 识别技术；Nikto 检查配置和危险项目。
30. Gobuster/ffuf 由 Wordlist 驱动；Nikto 由数据库和插件驱动。
31. 扫描请求、方法、路径、响应、时间和数量。
32. 需要收集 Web 日志和对应规则。
33. 证明修复有效并发现回归。
34. 数据库和结果会变化。
35. 可能违法并影响他人服务。

---

# 79. 命令速查表

## 基础

```bash
nikto \
  -h http://10.10.20.20:8084/
```

## 安全 Tuning

```bash
nikto \
  -Tuning 123be \
  -h http://10.10.20.20:8084/
```

## 进度

```bash
nikto \
  -Display 124P \
  -h http://10.10.20.20:8084/
```

## 时间限制

```bash
nikto \
  -timeout 5 \
  -maxtime 5m \
  -h http://10.10.20.20:8084/
```

## User-Agent

```bash
nikto \
  -useragent 'CyberLab-Nikto/1.0' \
  -h http://10.10.20.20:8084/
```

## Header

```bash
nikto \
  -Add-header 'X-Lab: Nikto' \
  -h http://10.10.20.20:8084/
```

## 禁用 CGI

```bash
nikto \
  -Cgidirs none \
  -h http://10.10.20.20:8084/
```

## 插件列表

```bash
nikto -list-plugins
```

## 数据库检查

```bash
nikto -dbcheck
```

## TXT 报告

```bash
nikto \
  -h http://10.10.20.20:8084/ \
  -o report.txt \
  -Format txt
```

## HTML 报告

```bash
nikto \
  -h http://10.10.20.20:8084/ \
  -o report.html \
  -Format htm
```

## JSON 报告

```bash
nikto \
  -h http://10.10.20.20:8084/ \
  -o report.json \
  -Format json
```

## 保存响应

```bash
nikto \
  -Save saved-responses \
  -h http://10.10.20.20:8084/
```

---

# 80. 核心记忆口诀

```text
Nikto 看 Web Server 安全问题
不是自动利用工具

-h：目标
-p：端口
-Tuning：选类别
-Display：看输出
-timeout：单请求时间
-maxtime：总扫描时间
-Pause：降低速度
-Plugins：选插件
-Cgidirs：CGI 范围
-o：输出文件
-Format：报告格式
-Save：保存阳性响应

发现不是结论
curl 必须验证
服务器证据最重要
修复以后再扫描
```

---

# 81. 完成标准

完成本课后，你应该拥有：

```text
Ubuntu 8084 Nikto 实验服务器
基础扫描结果
安全 Tuning 结果
TXT、HTML、JSON、XML 报告
Saved Responses
curl 人工验证记录
Banner 误报分析
HTTP 方法验证
Nikto PCAP
Wireshark 请求统计
Ubuntu Access Log
Wazuh nikto-lab 事件
TP/FP/INFO/NMC 结果表
Hardened Server
修复前后对比报告
完整实验报告
```

检查 Kali：

```bash
find ~/CyberLab/Nikto -type f
```

检查 Ubuntu：

```bash
find ~/CyberLab/Nikto -type f
```

---

# 82. 下一课

下一课建议：

```text
Burp Suite
```

学习内容：

- Proxy 和浏览器配置
- Intercept
- HTTP History
- Repeater
- Decoder
- Comparer
- Scope
- 手动修改实验请求
- 与 curl、ffuf 和 WhatWeb 联动
- TLS CA 证书
- Access Log、PCAP 和 Wazuh 观察

当前学习关系：

```text
Nmap：发现端口和服务
Netcat：测试连接
Wireshark：图形化流量分析
tcpdump：命令行抓包
curl：手动 HTTP 请求
WhatWeb：技术指纹识别
Gobuster/ffuf：内容发现
Nikto：Web Server 安全检查
```

---

# 83. 参考说明

本指南依据：

```text
Nikto 官方 GitHub 项目
Nikto 官方 Wiki
Kali Linux 官方 Nikto 工具页面
本机 nikto -h 输出
```

官方 Wiki 当前标注 Nikto 2.6.x 文档；Kali 软件包版本可能稍有差异。  
所有命令最终以你的 Kali 输出为准：

```bash
nikto -Version
nikto -h
nikto -list-plugins
```
