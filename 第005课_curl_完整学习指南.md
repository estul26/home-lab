# 第五课：curl 完整学习指南

> **实验环境：你自己的 CyberLab**
>
> 本课程只用于你自己创建并拥有的虚拟机和 Web 服务。  
> 不要测试学校、公司、公共网站、随机公网 IP 或任何未经授权的系统。

---

# 1. 本课目标

学完本课后，你应该能够：

```text
理解 curl 的用途
发送 HTTP GET、HEAD、POST、PUT 和 DELETE 请求
查看请求头、响应头、状态码和响应正文
发送表单、JSON 和自定义 Header
处理重定向、Cookie、认证和超时
下载文件并验证哈希
测试 HTTPS 和 TLS 证书
理解 -k 的风险
使用 --resolve 测试虚拟主机
把 curl 请求与 tcpdump、Wireshark 和 Wazuh 关联
阅读常见 curl 错误
保存请求结果并编写实验报告
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
→ HTTP 8080
→ Ubuntu 10.10.20.20
```

---

# 2. 什么是 curl？

`curl` 是一个命令行数据传输工具。

它经常用于：

- 测试网站和 Web 服务
- 调试 HTTP 请求
- 调用 API
- 查看响应头
- 下载文件
- 上传表单和文件
- 测试重定向
- 测试 Cookie
- 检查 TLS 证书
- 测量响应时间
- 自动化脚本

`curl` 的名字常被理解为：

```text
Client URL
```

简单记忆：

```text
浏览器适合人工浏览
curl 适合精确测试和自动化
```

---

# 3. curl 与其他工具的关系

```text
Nmap：发现端口和服务
Netcat：建立原始 TCP/UDP 连接
Wireshark：图形化分析数据包
tcpdump：命令行抓包
curl：发送应用层 HTTP/HTTPS 请求
Wazuh：分析端点日志和安全告警
```

例如：

```text
Nmap 发现 8080 开放
→ curl 请求网页
→ tcpdump/Wireshark 查看 HTTP 流量
→ Wazuh 查看服务器日志
```

---

# 4. 检查 curl

在 Kali：

```bash
which curl
```

查看版本：

```bash
curl --version
```

查看帮助：

```bash
curl --help
```

完整帮助：

```bash
man curl
```

如果没有安装：

```bash
sudo apt update
sudo apt install curl
```

---

# 5. curl 基本语法

```bash
curl [选项] URL
```

示例：

```bash
curl http://10.10.20.20:8080/
```

URL 结构：

```text
scheme://host:port/path?query
```

示例：

```text
http://10.10.20.20:8080/status/200?name=student
```

解释：

```text
Scheme：HTTP
Host：10.10.20.20
Port：8080
Path：/status/200
Query：name=student
```

---

# 6. HTTP 请求和响应

一个 HTTP 请求通常包含：

```text
Request Line
Headers
空行
Body（可选）
```

例如：

```http
GET / HTTP/1.1
Host: 10.10.20.20:8080
User-Agent: curl/...
Accept: */*
```

响应通常包含：

```text
Status Line
Headers
空行
Body
```

例如：

```http
HTTP/1.1 200 OK
Content-Type: text/plain
Content-Length: 22

CyberLab curl server
```

---

# 7. 常见 HTTP 方法

| 方法 | 主要用途 |
|---|---|
| GET | 读取资源 |
| HEAD | 只读取响应头 |
| POST | 创建或提交数据 |
| PUT | 替换或更新资源 |
| PATCH | 部分更新 |
| DELETE | 删除资源 |
| OPTIONS | 查询支持的方法 |

注意：

```text
HTTP 方法本身不等于权限
服务器仍然必须进行认证和授权
```

---

# 8. 常见 HTTP 状态码

## 2xx：成功

| 状态码 | 含义 |
|---:|---|
| 200 | OK |
| 201 | Created |
| 204 | No Content |

## 3xx：重定向

| 状态码 | 含义 |
|---:|---|
| 301 | Permanent Redirect |
| 302 | Temporary Redirect |
| 304 | Not Modified |

## 4xx：客户端问题

| 状态码 | 含义 |
|---:|---|
| 400 | Bad Request |
| 401 | Unauthorized |
| 403 | Forbidden |
| 404 | Not Found |
| 405 | Method Not Allowed |
| 429 | Too Many Requests |

## 5xx：服务器问题

| 状态码 | 含义 |
|---:|---|
| 500 | Internal Server Error |
| 502 | Bad Gateway |
| 503 | Service Unavailable |
| 504 | Gateway Timeout |

记忆：

```text
2xx 成功
3xx 跳转
4xx 请求方问题
5xx 服务器问题
```

---

# 9. 创建实验目录

在 Kali：

```bash
mkdir -p ~/CyberLab/curl/{downloads,requests,responses,notes,reports}
```

在 Ubuntu：

```bash
mkdir -p ~/CyberLab/curl/server
```

---

# 10. 在 Ubuntu 创建本地 HTTP 实验服务器

为了练习 GET、POST、Header、Cookie 和状态码，我们创建一个只在 CyberLab 使用的小型 Python Server。

在 Ubuntu：

```bash
nano ~/CyberLab/curl/server/http_lab_server.py
```

粘贴以下内容：

```python
#!/usr/bin/env python3

import json
import time
from http.server import BaseHTTPRequestHandler, HTTPServer
from urllib.parse import parse_qs, urlparse


HOST = "10.10.20.20"
PORT = 8080


class LabHandler(BaseHTTPRequestHandler):
    server_version = "CyberLabHTTP/1.0"

    def send_json(self, status: int, data: dict, headers: dict | None = None) -> None:
        body = json.dumps(data, indent=2).encode("utf-8")
        self.send_response(status)
        self.send_header("Content-Type", "application/json; charset=utf-8")
        self.send_header("Content-Length", str(len(body)))

        if headers:
            for name, value in headers.items():
                self.send_header(name, value)

        self.end_headers()

        if self.command != "HEAD":
            self.wfile.write(body)

    def read_body(self) -> bytes:
        length = int(self.headers.get("Content-Length", "0"))
        return self.rfile.read(length) if length > 0 else b""

    def request_details(self, body: bytes = b"") -> dict:
        parsed = urlparse(self.path)

        return {
            "method": self.command,
            "path": parsed.path,
            "query": parse_qs(parsed.query),
            "client_ip": self.client_address[0],
            "headers": dict(self.headers),
            "body": body.decode("utf-8", errors="replace"),
            "body_length": len(body),
        }

    def do_GET(self) -> None:
        parsed = urlparse(self.path)

        if parsed.path == "/":
            self.send_json(
                200,
                {
                    "message": "CyberLab curl server",
                    "available_paths": [
                        "/",
                        "/headers",
                        "/redirect",
                        "/cookie",
                        "/slow",
                        "/status/<code>",
                    ],
                },
            )
            return

        if parsed.path == "/headers":
            self.send_json(200, self.request_details())
            return

        if parsed.path == "/redirect":
            self.send_response(302)
            self.send_header("Location", "/")
            self.send_header("Content-Length", "0")
            self.end_headers()
            return

        if parsed.path == "/cookie":
            self.send_json(
                200,
                {"message": "Cookie created"},
                headers={"Set-Cookie": "cyberlab=student; Path=/; HttpOnly"},
            )
            return

        if parsed.path == "/slow":
            time.sleep(3)
            self.send_json(200, {"message": "Response delayed for 3 seconds"})
            return

        if parsed.path.startswith("/status/"):
            try:
                status = int(parsed.path.split("/")[-1])
            except ValueError:
                self.send_json(400, {"error": "Invalid status code"})
                return

            if status < 100 or status > 599:
                self.send_json(400, {"error": "Status must be between 100 and 599"})
                return

            self.send_json(status, {"requested_status": status})
            return

        self.send_json(404, {"error": "Path not found", "path": parsed.path})

    def do_HEAD(self) -> None:
        self.send_json(200, {"message": "HEAD response"})

    def do_POST(self) -> None:
        body = self.read_body()
        self.send_json(200, self.request_details(body))

    def do_PUT(self) -> None:
        body = self.read_body()
        self.send_json(200, self.request_details(body))

    def do_PATCH(self) -> None:
        body = self.read_body()
        self.send_json(200, self.request_details(body))

    def do_DELETE(self) -> None:
        self.send_json(200, self.request_details())

    def do_OPTIONS(self) -> None:
        self.send_response(204)
        self.send_header("Allow", "GET, HEAD, POST, PUT, PATCH, DELETE, OPTIONS")
        self.send_header("Content-Length", "0")
        self.end_headers()

    def log_message(self, format: str, *args: object) -> None:
        print(
            f"{self.client_address[0]} "
            f"[{self.log_date_time_string()}] "
            f"{format % args}",
            flush=True,
        )


def main() -> None:
    server = HTTPServer((HOST, PORT), LabHandler)
    print(f"CyberLab HTTP server listening on http://{HOST}:{PORT}")
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

赋予执行权限：

```bash
chmod +x ~/CyberLab/curl/server/http_lab_server.py
```

启动：

```bash
python3 ~/CyberLab/curl/server/http_lab_server.py
```

预期：

```text
CyberLab HTTP server listening on http://10.10.20.20:8080
```

保持此终端运行。

---

# 11. 检查服务器端口

在 Ubuntu 的第二个终端：

```bash
ss -lntp | grep ':8080'
```

在 Kali：

```bash
nc -zvn -w 3 10.10.20.20 8080
```

也可以：

```bash
sudo nmap -n -p 8080 -sV 10.10.20.20
```

---

# 12. 实验一：基础 GET

在 Kali：

```bash
curl http://10.10.20.20:8080/
```

预期返回 JSON：

```json
{
  "message": "CyberLab curl server",
  "available_paths": [
    "/",
    "/headers",
    "/redirect",
    "/cookie",
    "/slow",
    "/status/<code>"
  ]
}
```

`curl` 默认：

```text
把响应正文输出到终端
```

---

# 13. 显示响应头

使用：

```bash
curl -i http://10.10.20.20:8080/
```

`-i` 表示：

```text
Include response headers
```

你会看到：

```text
HTTP/1.0 200 OK
Server: CyberLabHTTP/1.0 Python/...
Content-Type: application/json; charset=utf-8
Content-Length: ...
```

然后是响应正文。

---

# 14. 只查看响应头

```bash
curl -I http://10.10.20.20:8080/
```

`-I` 会发送：

```text
HEAD 请求
```

也可以明确指定：

```bash
curl -X HEAD -i http://10.10.20.20:8080/
```

通常推荐：

```text
-I
```

---

# 15. Verbose 模式

```bash
curl -v http://10.10.20.20:8080/
```

`-v` 可以看到：

```text
连接目标
请求行
请求 Header
响应状态
响应 Header
```

输出中的符号：

```text
> 表示 curl 发出的内容
< 表示服务器返回的内容
* 表示连接过程信息
```

示例：

```text
> GET / HTTP/1.1
> Host: 10.10.20.20:8080
> User-Agent: curl/...
>
< HTTP/1.0 200 OK
< Content-Type: application/json
```

---

# 16. Silent 与错误显示

安静模式：

```bash
curl -s http://10.10.20.20:8080/
```

`-s` 会隐藏：

- 进度条
- 一些错误信息

更推荐脚本中使用：

```bash
curl -sS http://10.10.20.20:8080/
```

解释：

```text
-s = Silent
-S = 即使 Silent，也显示错误
```

---

# 17. 保存响应正文

保存到指定文件：

```bash
curl \
  -o ~/CyberLab/curl/responses/home.json \
  http://10.10.20.20:8080/
```

检查：

```bash
cat ~/CyberLab/curl/responses/home.json
```

`-o`：

```text
自己指定文件名
```

---

# 18. 使用远程文件名

```bash
curl -O http://10.10.20.20:8080/example.txt
```

`-O` 会尝试使用 URL 最后的名字：

```text
example.txt
```

注意：

当前实验 Server 没有 `/example.txt`，所以会返回 404 JSON。

区别：

```text
-o filename = 自己指定名称
-O          = 使用 URL 中的名称
```

---

# 19. 查看状态码

只输出状态码：

```bash
curl -s \
  -o /dev/null \
  -w '%{http_code}\n' \
  http://10.10.20.20:8080/
```

预期：

```text
200
```

解释：

```text
-s          安静
-o /dev/null 丢弃正文
-w          自定义输出格式
```

---

# 20. 测试不同状态码

```bash
curl -i http://10.10.20.20:8080/status/200
```

```bash
curl -i http://10.10.20.20:8080/status/404
```

```bash
curl -i http://10.10.20.20:8080/status/500
```

只输出状态码：

```bash
for code in 200 201 301 400 401 403 404 500 503; do
  curl -s -o /dev/null \
    -w "$code -> %{http_code}\n" \
    "http://10.10.20.20:8080/status/$code"
done
```

---

# 21. curl 失败状态处理

默认情况下，HTTP 404 不一定让 curl 返回失败。

例如：

```bash
curl http://10.10.20.20:8080/status/404
echo $?
```

curl 可能仍返回：

```text
0
```

使用：

```bash
curl --fail http://10.10.20.20:8080/status/404
```

或者推荐：

```bash
curl --fail-with-body \
  http://10.10.20.20:8080/status/404
```

`--fail-with-body`：

```text
HTTP 4xx/5xx 返回非零退出码
同时保留响应正文
```

检查退出码：

```bash
echo $?
```

---

# 22. 实验二：Query String

```bash
curl \
  'http://10.10.20.20:8080/headers?name=student&lesson=curl'
```

查看：

```json
"query": {
  "name": [
    "student"
  ],
  "lesson": [
    "curl"
  ]
}
```

URL 中包含 `&` 时，建议使用引号：

```bash
curl 'http://10.10.20.20:8080/headers?a=1&b=2'
```

否则 Shell 可能把 `&` 解释为后台运行。

---

# 23. URL 编码

使用：

```bash
curl -G \
  --data-urlencode 'name=Cyber Lab Student' \
  --data-urlencode 'note=Nmap & curl' \
  http://10.10.20.20:8080/headers
```

参数：

```text
-G               把数据放进 Query String
--data-urlencode 自动进行 URL 编码
```

空格可能被编码为：

```text
%20
或
+
```

`&` 会被安全编码。

---

# 24. 自定义 Header

```bash
curl \
  -H 'X-Lab-Name: CyberLab' \
  -H 'X-Student: learner' \
  http://10.10.20.20:8080/headers
```

服务器会返回看到的 Header。

常见 Header：

```text
Content-Type
Accept
Authorization
User-Agent
Cookie
Host
```

---

# 25. 修改 User-Agent

```bash
curl \
  -A 'CyberLab-curl-client/1.0' \
  http://10.10.20.20:8080/headers
```

等价：

```bash
curl \
  -H 'User-Agent: CyberLab-curl-client/1.0' \
  http://10.10.20.20:8080/headers
```

注意：

```text
User-Agent 可以伪装
不能作为可靠身份验证依据
```

---

# 26. 设置 Accept Header

请求 JSON：

```bash
curl \
  -H 'Accept: application/json' \
  http://10.10.20.20:8080/
```

常见值：

```text
application/json
text/html
text/plain
*/*
```

---

# 27. 实验三：POST 表单数据

```bash
curl -i \
  -X POST \
  -d 'username=student&course=curl' \
  http://10.10.20.20:8080/echo
```

使用 `-d` 时，curl 默认发送：

```text
Content-Type: application/x-www-form-urlencoded
```

实际上，只要使用 `-d`，curl 会自动使用 POST，所以可以简写：

```bash
curl -i \
  -d 'username=student&course=curl' \
  http://10.10.20.20:8080/echo
```

---

# 28. 多个 -d 参数

```bash
curl \
  -d 'username=student' \
  -d 'course=curl' \
  -d 'level=beginner' \
  http://10.10.20.20:8080/echo
```

curl 会组合成：

```text
username=student&course=curl&level=beginner
```

---

# 29. URL 编码表单值

```bash
curl \
  --data-urlencode 'name=Cyber Lab Student' \
  --data-urlencode 'message=Hello & welcome' \
  http://10.10.20.20:8080/echo
```

适合包含：

- 空格
- `&`
- `=`
- 特殊字符

---

# 30. 实验四：POST JSON

```bash
curl -i \
  -H 'Content-Type: application/json' \
  -d '{
    "username": "student",
    "course": "curl",
    "target": "ubuntu"
  }' \
  http://10.10.20.20:8080/echo
```

更适合脚本的单行写法：

```bash
curl -sS \
  -H 'Content-Type: application/json' \
  -d '{"username":"student","course":"curl"}' \
  http://10.10.20.20:8080/echo
```

---

# 31. 从文件发送 JSON

在 Kali：

```bash
cat > ~/CyberLab/curl/requests/student.json <<'EOF'
{
  "username": "student",
  "course": "curl",
  "lab": "CyberLab"
}
EOF
```

发送：

```bash
curl -i \
  -H 'Content-Type: application/json' \
  --data-binary @~/CyberLab/curl/requests/student.json \
  http://10.10.20.20:8080/echo
```

解释：

```text
@file = 从文件读取
--data-binary = 尽量原样发送
```

---

# 32. -d 与 --data-binary

| 参数 | 特点 |
|---|---|
| `-d` | 适合表单或普通数据，可能处理换行 |
| `--data-binary` | 尽量原样发送文件内容 |
| `--data-urlencode` | 自动 URL 编码 |
| `-F` | multipart/form-data |

---

# 33. 实验五：PUT、PATCH 和 DELETE

PUT：

```bash
curl -i \
  -X PUT \
  -H 'Content-Type: application/json' \
  -d '{"status":"updated"}' \
  http://10.10.20.20:8080/resource/1
```

PATCH：

```bash
curl -i \
  -X PATCH \
  -H 'Content-Type: application/json' \
  -d '{"status":"partial-update"}' \
  http://10.10.20.20:8080/resource/1
```

DELETE：

```bash
curl -i \
  -X DELETE \
  http://10.10.20.20:8080/resource/1
```

注意：

```text
实验 Server 只回显请求
不会真正修改或删除系统文件
```

---

# 34. OPTIONS 请求

```bash
curl -i \
  -X OPTIONS \
  http://10.10.20.20:8080/
```

预期看到：

```text
Allow: GET, HEAD, POST, PUT, PATCH, DELETE, OPTIONS
```

OPTIONS 常用于了解服务器声明支持的方法。

---

# 35. 实验六：重定向

不跟随重定向：

```bash
curl -i http://10.10.20.20:8080/redirect
```

你会看到：

```text
HTTP/1.0 302 Found
Location: /
```

跟随重定向：

```bash
curl -i -L http://10.10.20.20:8080/redirect
```

`-L`：

```text
Follow redirect
```

查看重定向次数：

```bash
curl -sS -L \
  -o /dev/null \
  -w 'Redirects: %{num_redirects}\nFinal: %{url_effective}\n' \
  http://10.10.20.20:8080/redirect
```

---

# 36. 限制重定向次数

```bash
curl -L \
  --max-redirs 3 \
  http://10.10.20.20:8080/redirect
```

这样可以避免错误配置造成无限重定向。

---

# 37. 实验七：Cookie

服务器设置 Cookie：

```bash
curl -i http://10.10.20.20:8080/cookie
```

响应头：

```text
Set-Cookie: cyberlab=student; Path=/; HttpOnly
```

保存 Cookie：

```bash
curl -c ~/CyberLab/curl/requests/cookies.txt \
  http://10.10.20.20:8080/cookie
```

查看：

```bash
cat ~/CyberLab/curl/requests/cookies.txt
```

发送 Cookie：

```bash
curl -b ~/CyberLab/curl/requests/cookies.txt \
  http://10.10.20.20:8080/headers
```

参数：

```text
-c = 保存 Cookie
-b = 发送 Cookie
```

---

# 38. 手动发送 Cookie

```bash
curl \
  -H 'Cookie: cyberlab=student' \
  http://10.10.20.20:8080/headers
```

或者：

```bash
curl \
  -b 'cyberlab=student' \
  http://10.10.20.20:8080/headers
```

---

# 39. 实验八：文件上传形式

创建测试文件：

```bash
echo 'CyberLab curl upload test' \
  > ~/CyberLab/curl/requests/upload.txt
```

发送 multipart/form-data：

```bash
curl -i \
  -F 'description=CyberLab test file' \
  -F 'file=@'"$HOME"'/CyberLab/curl/requests/upload.txt' \
  http://10.10.20.20:8080/upload
```

服务器会回显：

- `Content-Type`
- Body 长度
- multipart 数据

注意：

```text
实验 Server 不会保存上传文件
只用于观察请求格式
```

---

# 40. 上传原始文件内容

```bash
curl -i \
  -X PUT \
  -H 'Content-Type: text/plain' \
  --data-binary @~/CyberLab/curl/requests/upload.txt \
  http://10.10.20.20:8080/files/upload.txt
```

同样只会回显，不会真正写入服务器文件。

---

# 41. 认证基础

HTTP Basic Authentication 使用：

```bash
curl -u 'labuser:labpassword' \
  http://10.10.20.20:8080/headers
```

服务器会看到：

```text
Authorization: Basic ...
```

重要：

```text
Basic Auth 只是 Base64 编码
不是加密
必须配合 HTTPS
```

不要在真实环境命令行中直接写真实密码，因为可能进入：

- Shell history
- 进程列表
- 屏幕录制
- 日志

更安全的交互方式：

```bash
curl -u labuser \
  http://10.10.20.20:8080/headers
```

curl 会提示输入密码。

---

# 42. Bearer Token

API 常用：

```bash
curl \
  -H 'Authorization: Bearer TEST_TOKEN_ONLY' \
  http://10.10.20.20:8080/headers
```

本课只使用：

```text
TEST_TOKEN_ONLY
```

不要把真实 Token：

- 放进公开 Markdown
- 提交到 GitHub
- 保存在截图
- 写进 Shell 脚本

---

# 43. 实验九：超时

连接超时：

```bash
curl \
  --connect-timeout 2 \
  http://10.10.20.20:8080/
```

总时间限制：

```bash
curl \
  --max-time 2 \
  http://10.10.20.20:8080/slow
```

因为 `/slow` 会延迟约 3 秒，所以 `--max-time 2` 可能返回：

```text
curl: (28) Operation timed out
```

同时使用：

```bash
curl \
  --connect-timeout 2 \
  --max-time 5 \
  http://10.10.20.20:8080/slow
```

---

# 44. 重试

```bash
curl \
  --retry 3 \
  --retry-delay 1 \
  --connect-timeout 2 \
  http://10.10.20.20:8080/
```

参数：

```text
--retry 3       最多重试
--retry-delay 1 每次间隔 1 秒
```

不要对会产生副作用的 POST/DELETE 请求盲目重试。

---

# 45. 查看性能时间

```bash
curl -sS \
  -o /dev/null \
  -w '
HTTP Code: %{http_code}
Remote IP: %{remote_ip}
Remote Port: %{remote_port}
DNS: %{time_namelookup}s
Connect: %{time_connect}s
First Byte: %{time_starttransfer}s
Total: %{time_total}s
Downloaded: %{size_download} bytes
' \
  http://10.10.20.20:8080/
```

重要指标：

| 指标 | 含义 |
|---|---|
| `time_namelookup` | DNS 时间 |
| `time_connect` | TCP 建连时间 |
| `time_appconnect` | TLS 握手时间 |
| `time_starttransfer` | 首字节时间 |
| `time_total` | 总时间 |

因为使用 IP：

```text
DNS 时间通常很小
```

---

# 46. 保存格式模板

创建：

```bash
cat > ~/CyberLab/curl/requests/timing-format.txt <<'EOF'
HTTP Code: %{http_code}
Remote IP: %{remote_ip}
Remote Port: %{remote_port}
Connect: %{time_connect}s
First Byte: %{time_starttransfer}s
Total: %{time_total}s
Downloaded: %{size_download} bytes
EOF
```

使用：

```bash
curl -sS \
  -o /dev/null \
  -w @~/CyberLab/curl/requests/timing-format.txt \
  http://10.10.20.20:8080/
```

---

# 47. HTTPS 测试 Wazuh Dashboard

只查看 Wazuh Dashboard 响应头：

```bash
curl -I https://10.10.20.40/
```

由于实验室证书可能是自签名，可能出现：

```text
curl: (60) SSL certificate problem
```

在自己的隔离实验室中临时测试：

```bash
curl -k -I https://10.10.20.40/
```

`-k` 等价：

```text
--insecure
```

含义：

```text
不验证服务器证书
```

---

# 48. -k 的风险

不要在真实环境习惯性使用：

```bash
-k
```

因为它会跳过：

- 证书信任检查
- 主机名检查
- 部分中间人攻击防护

实验室中可以临时使用：

```bash
curl -k https://10.10.20.40/
```

真实环境优先：

```text
安装正确 CA
使用 --cacert
修复证书
使用匹配的域名
```

---

# 49. 查看 TLS 连接细节

```bash
curl -vkI https://10.10.20.40/
```

你可能看到：

```text
TLS version
Cipher
Certificate subject
Certificate issuer
Server response
```

注意：

```text
-v 输出中可能包含 Header
不要在有真实 Token/Cookie 时随意分享
```

---

# 50. 使用自定义 CA

如果有 CA 证书：

```bash
curl \
  --cacert ~/CyberLab/curl/requests/cyberlab-ca.crt \
  https://wazuh.lab/
```

这是比 `-k` 更安全的做法。

---

# 51. 使用 --resolve

`--resolve` 可以临时指定：

```text
域名
端口
IP
```

示例：

```bash
curl \
  --resolve lab.local:8080:10.10.20.20 \
  http://lab.local:8080/headers
```

curl 会：

```text
连接 10.10.20.20:8080
但发送 Host: lab.local:8080
```

这适合测试：

- 虚拟主机
- DNS 尚未配置
- 新网站切换
- TLS/SNI

不会修改系统 `/etc/hosts`。

---

# 52. 手动设置 Host Header

```bash
curl \
  -H 'Host: lab.local' \
  http://10.10.20.20:8080/headers
```

区别：

```text
-H Host 只修改 HTTP Header
--resolve 同时控制域名解析和 Host
```

HTTPS 测试通常更适合：

```text
--resolve
```

因为 TLS 还涉及 SNI 和证书域名。

---

# 53. HTTP 版本

强制 HTTP/1.1：

```bash
curl --http1.1 \
  http://10.10.20.20:8080/
```

尝试 HTTP/2：

```bash
curl --http2 \
  https://10.10.20.40/
```

服务器是否支持，取决于配置。

实验 Python Server 通常使用：

```text
HTTP/1.0 或 HTTP/1.1
```

---

# 54. 下载文件实验

Ubuntu 创建文件：

```bash
mkdir -p ~/CyberLab/curl/download
```

```bash
dd if=/dev/zero \
  of=~/CyberLab/curl/download/test-1mb.bin \
  bs=1M count=1
```

计算哈希：

```bash
sha256sum ~/CyberLab/curl/download/test-1mb.bin
```

启动临时下载服务器：

```bash
cd ~/CyberLab/curl/download
python3 -m http.server 8081 --bind 10.10.20.20
```

Kali 下载：

```bash
curl \
  -o ~/CyberLab/curl/downloads/test-1mb.bin \
  http://10.10.20.20:8081/test-1mb.bin
```

验证：

```bash
sha256sum ~/CyberLab/curl/downloads/test-1mb.bin
```

两边 SHA-256 应相同。

---

# 55. 继续中断的下载

如果服务器支持 Range：

```bash
curl -C - \
  -o ~/CyberLab/curl/downloads/test-1mb.bin \
  http://10.10.20.20:8081/test-1mb.bin
```

参数：

```text
-C - = 从现有文件位置继续
```

服务器是否支持续传取决于：

```text
HTTP Range Requests
```

---

# 56. 限制下载速度

```bash
curl \
  --limit-rate 100K \
  -o ~/CyberLab/curl/downloads/test-limited.bin \
  http://10.10.20.20:8081/test-1mb.bin
```

适合：

- 测试慢速网络
- 避免占满带宽
- 观察下载时间

---

# 57. 输出响应头到文件

```bash
curl \
  -D ~/CyberLab/curl/responses/headers.txt \
  -o ~/CyberLab/curl/responses/body.json \
  http://10.10.20.20:8080/
```

参数：

```text
-D = 保存响应头
-o = 保存响应正文
```

查看：

```bash
cat ~/CyberLab/curl/responses/headers.txt
```

```bash
cat ~/CyberLab/curl/responses/body.json
```

---

# 58. 保存 Verbose 日志

```bash
curl -v \
  http://10.10.20.20:8080/ \
  -o ~/CyberLab/curl/responses/body.json \
  2> ~/CyberLab/curl/responses/verbose.log
```

原因：

```text
curl -v 信息输出到 stderr
正文输出到 stdout
```

`2>` 表示：

```text
重定向 stderr
```

---

# 59. 查看 curl 退出码

运行：

```bash
curl --fail-with-body \
  http://10.10.20.20:8080/status/404
```

查看：

```bash
echo $?
```

退出码：

```text
0 = 命令成功
非 0 = 出现错误
```

脚本示例：

```bash
if curl -sS --fail \
  http://10.10.20.20:8080/ \
  -o /dev/null; then
  echo "Service is healthy"
else
  echo "Service check failed"
fi
```

---

# 60. 常见 curl 错误

| 错误码 | 常见含义 |
|---:|---|
| 6 | 无法解析主机名 |
| 7 | 无法连接目标 |
| 22 | 使用 `--fail` 时收到 HTTP 4xx/5xx |
| 23 | 写入本地文件失败 |
| 28 | 超时 |
| 35 | TLS/SSL 连接错误 |
| 52 | 服务器没有返回数据 |
| 56 | 接收数据失败 |
| 60 | 证书验证失败 |

查看最近命令退出码：

```bash
echo $?
```

---

# 61. 错误 6：Could not resolve host

示例：

```text
curl: (6) Could not resolve host
```

原因：

- 域名拼写错误
- DNS 不可用
- CyberLab 没有 DNS

解决：

```text
使用 IP
使用 --resolve
检查 /etc/hosts
检查 DNS 设置
```

---

# 62. 错误 7：Failed to connect

```text
curl: (7) Failed to connect
```

检查：

```bash
ping -c 3 10.10.20.20
```

```bash
nc -zvn -w 3 10.10.20.20 8080
```

```bash
sudo nmap -n -p 8080 10.10.20.20
```

Ubuntu：

```bash
ss -lntp | grep ':8080'
```

---

# 63. 错误 28：Timeout

```text
curl: (28) Operation timed out
```

可能原因：

- 服务响应慢
- 网络不通
- 防火墙丢弃
- `--max-time` 太短

调试：

```bash
curl -v \
  --connect-timeout 3 \
  --max-time 10 \
  http://10.10.20.20:8080/
```

---

# 64. 错误 60：Certificate problem

```text
curl: (60) SSL certificate problem
```

可能原因：

- 自签名证书
- CA 不受信任
- 证书过期
- 主机名不匹配

实验室临时查看：

```bash
curl -vkI https://10.10.20.40/
```

长期正确处理：

```text
安装 CA
使用 --cacert
修复证书
使用正确域名
```

---

# 65. 与 tcpdump 联动

在 Kali 终端一：

```bash
sudo tcpdump -i eth0 -nn -A tcp port 8080
```

终端二：

```bash
curl \
  -H 'X-Lab-Test: CURL_TCPDUMP' \
  http://10.10.20.20:8080/headers
```

tcpdump 中可能看到：

```text
GET /headers HTTP/1.1
Host: 10.10.20.20:8080
User-Agent: curl/...
X-Lab-Test: CURL_TCPDUMP
```

说明：

```text
HTTP 是明文
```

---

# 66. 保存 curl PCAP

```bash
sudo tcpdump -i eth0 -nn -s 0 \
  -w ~/CyberLab/curl/requests/curl-http8080.pcap \
  'host 10.10.20.20 and tcp port 8080'
```

另一个终端发送请求：

```bash
curl \
  -H 'X-Lab-Test: CURL_PCAP' \
  -d '{"lesson":"curl"}' \
  -H 'Content-Type: application/json' \
  http://10.10.20.20:8080/echo
```

停止抓包：

```text
Ctrl + C
```

使用 Wireshark：

```bash
wireshark \
  ~/CyberLab/curl/requests/curl-http8080.pcap
```

过滤：

```text
http
```

或：

```text
tcp.port == 8080
```

---

# 67. Wireshark 中分析 curl

可以找到：

```text
TCP 三次握手
HTTP 请求方法
URI
Host
User-Agent
Content-Type
Content-Length
POST Body
HTTP 状态码
响应 Header
响应正文
TCP 连接关闭
```

右键：

```text
Follow
→ HTTP Stream
```

---

# 68. HTTPS 抓包对比

Kali 抓 Wazuh HTTPS：

```bash
sudo tcpdump -i eth0 -nn -s 0 \
  -w ~/CyberLab/curl/requests/curl-wazuh-https.pcap \
  'host 10.10.20.40 and tcp port 443'
```

另一个终端：

```bash
curl -kI https://10.10.20.40/
```

Wireshark 过滤：

```text
tls
```

你通常可以看到：

```text
Client Hello
Server Hello
Certificate
Encrypted Application Data
```

但不能直接看到 HTTPS 正文。

---

# 69. 与 Wazuh 日志联动

普通 curl 请求不一定自动产生 Wazuh 告警。

是否可见取决于：

- Web Server 是否记录访问日志
- Wazuh Agent 是否收集该日志
- 是否存在匹配规则
- 请求是否触发异常行为

为了确认日志链路，在 Ubuntu 请求前后记录：

```bash
logger -t curl-lab \
  "Starting authorized curl HTTP experiment"
```

实验完成后：

```bash
logger -t curl-lab \
  "Completed authorized curl request from Kali 10.10.20.10"
```

查看本地日志：

```bash
journalctl -t curl-lab --since "15 minutes ago"
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
curl-lab
```

---

# 70. 让 Python Server 日志写入文件

启动时：

```bash
python3 ~/CyberLab/curl/server/http_lab_server.py \
  2>&1 |
tee ~/CyberLab/curl/server/access.log
```

这样终端显示的同时，也保存到：

```text
~/CyberLab/curl/server/access.log
```

注意：

```text
Wazuh 默认不一定收集这个自定义文件
需要以后配置 localfile
```

本课先保存本地证据。

---

# 71. Burp Suite 代理预览

以后学习 Burp Suite 时，可以把 curl 请求发送到代理：

```bash
curl \
  -x http://127.0.0.1:8080 \
  http://10.10.20.20:8080/
```

参数：

```text
-x = Proxy
```

HTTPS 经过 Burp 时，通常还需要处理 Burp CA 证书。

不要在本课真实使用未配置的代理环境。

---

# 72. 环境变量代理

查看：

```bash
env | grep -i proxy
```

常见：

```text
HTTP_PROXY
HTTPS_PROXY
NO_PROXY
```

临时绕过代理：

```bash
curl --noproxy '*' \
  http://10.10.20.20:8080/
```

只让 CyberLab 不走代理：

```bash
export NO_PROXY='10.10.20.0/24,localhost,127.0.0.1'
```

---

# 73. curl 配置文件

curl 可以读取：

```text
~/.curlrc
```

不要把以下内容放入公开配置：

- 密码
- Token
- Cookie
- 私钥
- 敏感 Header

查看是否存在：

```bash
ls -la ~/.curlrc
```

调试时，如果怀疑配置影响命令：

```bash
curl -q http://10.10.20.20:8080/
```

`-q` 放在最前面时，可避免读取默认 curl 配置。

---

# 74. Shell 引号的重要性

推荐：

```bash
curl \
  -H 'Content-Type: application/json' \
  -d '{"name":"student"}' \
  http://10.10.20.20:8080/echo
```

常见问题：

- JSON 内部使用双引号
- 外层使用单引号
- URL 包含 `&` 时要加引号
- 不要让 Shell 展开特殊字符

---

# 75. 安全的脚本示例

创建：

```bash
nano ~/CyberLab/curl/requests/health-check.sh
```

内容：

```bash
#!/usr/bin/env bash

set -u

URL="http://10.10.20.20:8080/"
TIMEOUT_SECONDS=5

if curl \
  --silent \
  --show-error \
  --fail \
  --max-time "$TIMEOUT_SECONDS" \
  --output /dev/null \
  "$URL"; then
  echo "UP: $URL"
  exit 0
else
  code=$?
  echo "DOWN: $URL (curl exit code: $code)"
  exit "$code"
fi
```

赋权：

```bash
chmod +x ~/CyberLab/curl/requests/health-check.sh
```

运行：

```bash
~/CyberLab/curl/requests/health-check.sh
```

---

# 76. 不要把秘密写进脚本

错误示例：

```bash
TOKEN='REAL_SECRET_TOKEN'
PASSWORD='REAL_PASSWORD'
```

因为可能被：

- Git 记录
- 备份
- 日志
- 其他用户读取
- 截图泄露

更安全方式包括：

```text
交互输入
环境变量
专用 Secret Manager
权限受控配置文件
```

本课只使用测试值。

---

# 77. 完整实验一：GET、Header 和状态码

## Ubuntu

启动：

```bash
python3 ~/CyberLab/curl/server/http_lab_server.py
```

## Kali

GET：

```bash
curl http://10.10.20.20:8080/
```

Header：

```bash
curl -i http://10.10.20.20:8080/
```

Verbose：

```bash
curl -v http://10.10.20.20:8080/
```

状态码：

```bash
curl -s -o /dev/null \
  -w '%{http_code}\n' \
  http://10.10.20.20:8080/status/404
```

保存证据：

```bash
curl -D ~/CyberLab/curl/responses/lab1-headers.txt \
  -o ~/CyberLab/curl/responses/lab1-body.json \
  http://10.10.20.20:8080/
```

---

# 78. 完整实验二：POST JSON

创建请求文件：

```bash
cat > ~/CyberLab/curl/requests/lab2.json <<'EOF'
{
  "student": "CyberLab learner",
  "lesson": "curl",
  "source_ip": "10.10.20.10",
  "target_ip": "10.10.20.20"
}
EOF
```

发送：

```bash
curl -sS \
  -H 'Content-Type: application/json' \
  --data-binary @~/CyberLab/curl/requests/lab2.json \
  http://10.10.20.20:8080/echo \
  -o ~/CyberLab/curl/responses/lab2-response.json
```

查看：

```bash
cat ~/CyberLab/curl/responses/lab2-response.json
```

同时使用 tcpdump 保存 PCAP。

---

# 79. 完整实验三：Cookie 和重定向

Cookie：

```bash
curl -c ~/CyberLab/curl/requests/lab3-cookies.txt \
  http://10.10.20.20:8080/cookie
```

发送 Cookie：

```bash
curl -b ~/CyberLab/curl/requests/lab3-cookies.txt \
  http://10.10.20.20:8080/headers
```

重定向：

```bash
curl -i http://10.10.20.20:8080/redirect
```

跟随：

```bash
curl -i -L http://10.10.20.20:8080/redirect
```

记录：

```text
原始状态码
Location Header
最终 URL
重定向次数
```

---

# 80. 完整实验四：HTTP 与 HTTPS 对比

HTTP：

```bash
curl -v http://10.10.20.20:8080/
```

HTTPS：

```bash
curl -vkI https://10.10.20.40/
```

同时分别抓包。

比较：

| 项目 | HTTP | HTTPS |
|---|---|---|
| 目标 | Ubuntu 8080 | Wazuh 443 |
| 请求正文 | 可能可见 | 加密 |
| Header | 可能可见 | TLS 后加密 |
| 证书 | 无 | 有 |
| Wireshark | HTTP | TLS |

---

# 81. 完整实验五：下载与哈希

Ubuntu：

```bash
cd ~/CyberLab/curl/download
python3 -m http.server 8081 --bind 10.10.20.20
```

Kali：

```bash
curl \
  -o ~/CyberLab/curl/downloads/test-1mb.bin \
  http://10.10.20.20:8081/test-1mb.bin
```

比较：

```bash
sha256sum ~/CyberLab/curl/downloads/test-1mb.bin
```

与 Ubuntu 原始文件哈希。

---

# 82. 实验报告模板

## Lab Information

```text
Lab Name：curl HTTP Client Testing
Date：
Student：
Client：Kali-CyberLab 10.10.20.10
Server：Ubuntu-Target 10.10.20.20
Ports：8080 / 8081
Authorization：Personal CyberLab
```

## Objective

```text
使用 curl 测试 HTTP 方法、Header、状态码、JSON、Cookie、重定向和文件下载。
```

## Server Command

```bash
# 粘贴 Ubuntu Server 命令
```

## Client Commands

```bash
# 粘贴 curl 命令
```

## Results

| Test | Expected | Actual | Status |
|---|---|---|---|
| GET / | 200 |  |  |
| HEAD / | 200 |  |  |
| /status/404 | 404 |  |  |
| POST JSON | Body echoed |  |  |
| Redirect | 302 then 200 |  |  |
| Cookie | Saved and returned |  |  |
| Download hash | Match |  |  |
| HTTPS certificate | Self-signed behavior |  |  |

## Evidence

```text
Request file：
Response Header：
Response Body：
Verbose log：
PCAP：
Wireshark screenshot：
Wazuh event：
SHA-256：
```

## Analysis

```text
GET 与 HEAD 有什么区别？
-i 与 -I 有什么区别？
HTTP 404 为什么默认可能仍返回 curl exit code 0？
-k 做了什么？
HTTP 与 HTTPS 抓包有什么区别？
```

## Lessons Learned

```text
最常用的 curl 参数：
最常见的错误：
最重要的 TLS 安全规则：
```

---

# 83. 本课练习

1. 使用 GET 请求 `/`。
2. 使用 `-i` 查看响应头和正文。
3. 使用 `-I` 发送 HEAD。
4. 使用 `-v` 查看请求和响应过程。
5. 分别测试 200、404 和 500。
6. 使用 `--fail-with-body` 处理 404。
7. 发送包含两个参数的 Query String。
8. 使用 `--data-urlencode` 发送带空格的数据。
9. 添加自定义 `X-Lab-Name` Header。
10. 修改 User-Agent。
11. 提交 URL-encoded 表单。
12. 提交 JSON。
13. 从文件发送 JSON。
14. 发送 PUT、PATCH 和 DELETE。
15. 使用 OPTIONS 查看 Allow。
16. 测试 302 重定向并用 `-L` 跟随。
17. 保存并发送 Cookie。
18. 上传一个测试文本文件。
19. 测试 `/slow` 和 `--max-time`。
20. 输出连接和总时间。
21. 使用 `--resolve` 测试 `lab.local`。
22. 检查 Wazuh HTTPS 证书错误。
23. 使用 `-k` 临时测试并解释风险。
24. 下载 1 MB 文件并验证 SHA-256。
25. 使用 tcpdump 捕获 POST JSON。
26. 用 Wireshark Follow HTTP Stream。
27. 在 Wazuh 中找到 `curl-lab` 测试日志。
28. 写一份实验报告。

---

# 84. 复习题

1. curl 主要用于什么？
2. `-i` 与 `-I` 有什么区别？
3. `-v` 输出中的 `>` 和 `<` 表示什么？
4. `-sS` 为什么常用于脚本？
5. `-o` 与 `-O` 有什么区别？
6. 如何只输出 HTTP 状态码？
7. `--fail-with-body` 的作用是什么？
8. `-H` 的作用是什么？
9. 使用 `-d` 时默认是什么 HTTP 方法？
10. 如何发送 JSON？
11. `--data-binary @file` 有什么作用？
12. `-L` 的作用是什么？
13. `-c` 与 `-b` 分别做什么？
14. `-F` 用于什么？
15. `--connect-timeout` 与 `--max-time` 有什么区别？
16. `-k` 做了什么？
17. 为什么真实环境不应该随意使用 `-k`？
18. `--resolve` 有什么用途？
19. 为什么 HTTP 内容可以在抓包中看到？
20. 为什么 HTTPS 正文通常看不到？
21. curl 错误 7 通常表示什么？
22. curl 错误 28 通常表示什么？
23. curl 错误 60 通常表示什么？
24. 如何检查最近命令的退出码？
25. curl、tcpdump、Wireshark 和 Wazuh 如何配合？

---

# 85. 参考答案

1. 命令行传输数据和测试 HTTP/HTTPS/API。
2. `-i` 显示响应头加正文；`-I` 发送 HEAD，只查看响应头。
3. `>` 是请求；`<` 是响应。
4. 隐藏进度但保留错误。
5. `-o` 自定义文件名；`-O` 使用远程名称。
6. `-s -o /dev/null -w '%{http_code}\n'`。
7. HTTP 4xx/5xx 返回失败退出码，同时保留正文。
8. 添加或修改 Header。
9. POST。
10. 设置 `Content-Type: application/json` 并使用 `-d`。
11. 从文件尽量原样发送数据。
12. 跟随重定向。
13. `-c` 保存 Cookie；`-b` 发送 Cookie。
14. multipart/form-data 表单或文件上传。
15. 前者限制建连时间；后者限制整个操作时间。
16. 跳过 TLS 证书验证。
17. 会失去证书信任和主机名验证保护。
18. 临时把域名和端口映射到指定 IP。
19. HTTP 默认不加密。
20. HTTPS 使用 TLS 加密。
21. 无法连接目标。
22. 操作超时。
23. TLS 证书验证失败。
24. `echo $?`。
25. curl 发送请求，tcpdump 抓包，Wireshark 分析，Wazuh 查看端点日志和告警。

---

# 86. 命令速查表

## GET

```bash
curl http://10.10.20.20:8080/
```

## 响应头加正文

```bash
curl -i http://10.10.20.20:8080/
```

## HEAD

```bash
curl -I http://10.10.20.20:8080/
```

## Verbose

```bash
curl -v http://10.10.20.20:8080/
```

## 状态码

```bash
curl -s -o /dev/null \
  -w '%{http_code}\n' \
  http://10.10.20.20:8080/
```

## 自定义 Header

```bash
curl -H 'X-Lab: curl' \
  http://10.10.20.20:8080/headers
```

## POST 表单

```bash
curl -d 'name=student' \
  http://10.10.20.20:8080/echo
```

## POST JSON

```bash
curl \
  -H 'Content-Type: application/json' \
  -d '{"name":"student"}' \
  http://10.10.20.20:8080/echo
```

## 跟随重定向

```bash
curl -L http://10.10.20.20:8080/redirect
```

## Cookie

```bash
curl -c cookies.txt \
  http://10.10.20.20:8080/cookie

curl -b cookies.txt \
  http://10.10.20.20:8080/headers
```

## 超时

```bash
curl \
  --connect-timeout 2 \
  --max-time 5 \
  http://10.10.20.20:8080/
```

## HTTPS 实验室检查

```bash
curl -vkI https://10.10.20.40/
```

## 临时域名解析

```bash
curl \
  --resolve lab.local:8080:10.10.20.20 \
  http://lab.local:8080/
```

## 下载

```bash
curl -o output.bin \
  http://10.10.20.20:8081/test-1mb.bin
```

---

# 87. 核心记忆口诀

```text
curl URL：发送请求
-i：Header + Body
-I：只看 Header
-v：看详细过程
-sS：安静但显示错误
-o：保存为指定文件
-w：输出状态和时间
-H：加 Header
-d：发数据
-F：发 multipart
-L：跟随跳转
-c：存 Cookie
-b：发 Cookie
-k：跳过证书验证，只限实验
--resolve：临时指定域名和 IP
```

---

# 88. 完成标准

完成本课后，你应该拥有：

```text
Ubuntu HTTP 实验 Server
GET 和 HEAD 测试
状态码测试
POST 表单测试
POST JSON 测试
PUT/PATCH/DELETE 测试
Redirect 测试
Cookie 文件
文件上传请求
超时测试
性能时间输出
Wazuh HTTPS/TLS 测试
文件下载和 SHA-256
HTTP PCAP
HTTPS PCAP
Wireshark HTTP Stream
Wazuh curl-lab 日志
一份完整实验报告
```

检查 Kali 文件：

```bash
find ~/CyberLab/curl -type f
```

检查 Ubuntu 文件：

```bash
find ~/CyberLab/curl -type f
```

---

# 89. 下一课

下一课建议：

```text
WhatWeb
```

学习内容：

- 识别 Web 服务器和技术栈
- 分析 HTTP Header
- 识别 CMS、框架和插件线索
- 比较 curl、Nmap 和 WhatWeb
- 使用 Wireshark/tcpdump 查看探测流量
- 在 Wazuh 中观察 Web 日志

当前工具关系：

```text
Nmap：发现服务
Netcat：连接端口
Wireshark：图形化分析
tcpdump：命令行抓包
curl：发送 HTTP/HTTPS 请求
```

---

# 90. 安全边界

只允许测试：

```text
Ubuntu-Target      10.10.20.20
Windows-Target     10.10.20.30
Wazuh-SIEM         10.10.20.40
```

不要测试：

- 学校或公司网站
- 公共服务
- 随机公网 IP
- 邻居设备
- 未经授权的 API
- 不属于你的云服务器

每次实验：

```text
确认目标属于自己
使用测试数据
不要暴露真实密码或 Token
限制请求频率
保存必要证据
实验结束后停止测试 Server
删除不需要的 Cookie 和临时文件
```
