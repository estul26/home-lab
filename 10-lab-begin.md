## CyberLab 接下来学习路线

1. **创建稳定备份**
   分别克隆 Kali、Ubuntu、Windows 和 Wazuh，保留一个“全部正常连接”的恢复点。

2. **建立正常状态基线**
   记录四台机器的 IP、开放端口、运行服务、CPU、内存和已安装软件，保存第一次 Nmap 扫描结果。

3. **确认日志收集链路**
   在 Ubuntu 和 Windows 创建测试事件，确认能在 Wazuh 的 **Threat Hunting** 中搜索到。Wazuh Agent 可以收集 Linux 日志和 Windows Event Log，并发送给 Wazuh Server 分析。([Wazuh Documentation][1])

4. **学习 Wazuh Dashboard**
   熟悉以下页面：

   * Endpoints
   * Threat Hunting
   * Security Events
   * File Integrity Monitoring
   * Vulnerability Detection
   * Configuration Assessment
   * MITRE ATT&CK
     Wazuh Dashboard 本身就提供这些安全监控与调查视图。([Wazuh Documentation][2])

5. **文件完整性监控实验（FIM）**
   在 Ubuntu 的 `/etc` 和 Windows 的测试文件夹中创建、修改、删除文件，然后在 Wazuh 查看告警。FIM 可以检测文件创建、修改和删除。([Wazuh Documentation][3])

6. **登录与身份验证监控实验**
   从 Kali：

   * SSH 登录 Ubuntu
   * 多次输入错误密码
   * 成功登录
     然后在 Wazuh 中查找失败登录、成功登录和来源 IP。

7. **Windows 安全事件实验**
   在 Windows：

   * 创建本地用户
   * 修改用户组
   * 触发登录失败
   * 修改 Defender Firewall 规则
   * 创建计划任务
     然后在 Wazuh 中查看对应的 Windows Security、System 和 Application 事件。Wazuh 默认可以分析这些 Windows 事件通道。([Wazuh Documentation][4])

8. **漏洞检测实验**
   查看 Ubuntu 和 Windows 的：

   * 已安装软件
   * 系统清单
   * 检测到的 CVE
   * 严重程度
   * 建议修复方法
     Wazuh Vulnerability Detection 会结合 Agent 收集的软件清单识别可能存在漏洞的软件包。([Wazuh Documentation][5])

9. **安全配置评估实验（SCA）**
   检查：

   * SSH 配置
   * 密码策略
   * 文件权限
   * 防火墙
   * 不必要服务
   * Windows 安全设置
     SCA 用于发现错误配置和暴露面，并提供加固建议。([Wazuh Documentation][6])

10. **Kali 扫描与 Wazuh 关联分析**
    从 Kali 对 Ubuntu 和 Windows 执行：

    * Ping
    * Nmap 基础扫描
    * 全端口扫描
    * 服务版本识别
      同时观察目标系统和 Wazuh 是否产生相关日志。

11. **MITRE ATT&CK 映射**
    查看告警关联到哪些：

    * Tactic
    * Technique
    * Technique ID
      Wazuh 的 MITRE ATT&CK 页面可用于查看环境中检测到的攻击战术、技术和过程。([Wazuh Documentation][7])

12. **自定义规则和主动响应**
    最后再学习：

    * 自定义检测规则
    * 自定义解码器
    * 告警等级
    * 自动封禁来源 IP
    * Active Response
      这一步应放在日志分析和规则理解之后，避免误封自己的 Kali。

**推荐立即开始的顺序：**

```text
备份
→ 测试日志
→ 学习 Dashboard
→ FIM 实验
→ SSH 登录监控
→ Windows 事件监控
→ 漏洞与 SCA
→ Kali 扫描关联分析
→ MITRE ATT&CK
→ 自定义规则与响应
```

[1]: https://documentation.wazuh.com/current/user-manual/capabilities/log-data-collection/configuration.html?utm_source=chatgpt.com "Configuring log collection for different operating systems"
[2]: https://documentation.wazuh.com/current/user-manual/wazuh-dashboard/navigating-the-wazuh-dashboard.html?utm_source=chatgpt.com "Navigating the Wazuh dashboard - Wazuh dashboard · Wazuh documentation"
[3]: https://documentation.wazuh.com/current/user-manual/capabilities/file-integrity/index.html?utm_source=chatgpt.com "File integrity monitoring - Capabilities · Wazuh documentation"
[4]: https://documentation.wazuh.com/current/user-manual/manager/wazuh-manager.html?utm_source=chatgpt.com "Wazuh manager - Wazuh server · Wazuh documentation"
[5]: https://documentation.wazuh.com/current/user-manual/capabilities/vulnerability-detection/configuring-scans.html?utm_source=chatgpt.com "Configuration - Vulnerability detection · Wazuh documentation"
[6]: https://documentation.wazuh.com/current/user-manual/capabilities/sec-config-assessment/index.html?utm_source=chatgpt.com "Security Configuration Assessment - Capabilities · Wazuh documentation"
[7]: https://documentation.wazuh.com/current/getting-started/use-cases/threat-hunting.html?utm_source=chatgpt.com "Threat hunting - Use cases · Wazuh documentation"
