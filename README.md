**English** | [中文](README.zh-CN.md)

# Cybersecurity Home Lab — UTM Network Build & Playbook

A self-directed security home lab built with UTM on a MacBook Pro (Apple M5). It runs an isolated attack/defense network — a Kali attacker, Ubuntu Server and Windows 11 targets, and a Wazuh SIEM — and documents the full build, the connectivity tests, and a working methodology for scanning, detection and reporting.

This document records how the lab network is configured and how it is used for practice.

## 1. Lab goals

Use a MacBook Pro M5 and UTM to create an isolated cybersecurity lab for practising:

- Nmap network scanning
- Host discovery
- TCP/UDP port scanning
- Service and version enumeration
- Web vulnerability testing
- SSH logins and log analysis
- Windows security log analysis
- Firewall configuration
- Wazuh SIEM log collection
- Wireshark traffic analysis
- Hardening and re-scanning
- Incident response

---

## 2. Host configuration

| Item | Configuration |
|---|---|
| Host | MacBook Pro |
| Chip | Apple M5 |
| Memory | 48 GB |
| Virtualization software | UTM |
| VM architecture | ARM64 |
| Primary network mode | Host Network / isolated network |
| Lab network name | CyberLab |

---

## 3. Network topology

```text
                         MacBook Pro M5
                               │
                    UTM Host Network
                         CyberLab
                     10.10.20.0/24
                               │
       ┌───────────────────────┼───────────────────────┐
       │                       │                       │
 Kali Linux              Ubuntu Server           Windows 11
 Attack machine          Linux target            Windows target
 10.10.20.10             10.10.20.20             10.10.20.30
                                                       │
                                                       │
                                               Wazuh Server
                                               Logging & SIEM
                                               10.10.20.40
```

All VMs connect to the same isolated network named `CyberLab`.

---

## 4. IP address plan

| VM | Role | Static IP | Netmask | Gateway | DNS |
|---|---|---|---|---|---|
| Kali Linux | Attacker | `10.10.20.10` | `/24` | none | none |
| Ubuntu Server | Linux target | `10.10.20.20` | `/24` | none | none |
| Windows 11 | Windows target | `10.10.20.30` | `/24` | none | none |
| Wazuh Server | SIEM log server | `10.10.20.40` | `/24` | none | none |

### Network information

```text
Network name:    CyberLab
Network address: 10.10.20.0
CIDR:            10.10.20.0/24
Subnet mask:     255.255.255.0
Broadcast:       10.10.20.255
Default gateway: none
DNS:             none
DHCP:            none
```

### Why no gateway?

With no default gateway, the VMs can only talk to devices on the same lab subnet.

Through the lab interface they **cannot** reach:

- the home LAN
- the router
- other household devices
- the internet

This reduces the risk of a vulnerable target machine affecting the real network.

---

# 5. UTM network modes

## 5.1 CyberLab isolated interface

The main interface for attack and defense practice.

```text
Network Mode: Host Network / macOS Host Only
Host Network: CyberLab
IP:           manual / static
Gateway:      leave blank
DNS:          leave blank
```

The CyberLab interface is used for:

- Kali scanning Ubuntu
- Kali scanning Windows
- Kali scanning Wazuh
- Ubuntu sending logs to Wazuh
- Windows sending logs to Wazuh
- Wireshark capturing lab traffic
- Firewall rule testing
- Web vulnerability testing

## 5.2 Shared Network interface

Shared Network is a temporary internet interface.

```text
Network Mode: macOS Shared Network
IP settings:  DHCP / Automatic
Purpose:      downloading software and system updates
```

When Shared Network is used, UTM provides NAT internet access through the Mac.

## 5.3 Bridged Network

Do **not** connect deliberately vulnerable targets to Bridged Network.

Bridged Network places a VM directly on the home LAN, where it looks like a real household device.

Not recommended for:

- OWASP Juice Shop
- WebGoat
- Metasploitable
- servers with intentionally weak passwords
- unpatched Windows
- malware analysis
- exploitation practice

---

# 6. Creating the CyberLab isolated network in UTM

Before installing the VMs, create a custom Host Network named `CyberLab`. Kali, Ubuntu, Windows and Wazuh then all connect to it.

## 6.1 Create the Host Network

Open UTM and go to:

```text
UTM
→ Settings…
→ Network
→ Host Networks
→ click +
```

Set the network name:

```text
Name: CyberLab
```

Save the settings.

> Usually you only need to create the network name here; you don't need to enter `10.10.20.0/24`. The static IP addresses are set manually in each VM's operating system.

## 6.2 What CyberLab is for

`CyberLab` lets the lab VMs communicate with each other:

```text
Kali Linux
Ubuntu Server
Windows 11
Wazuh Server
```

They must all connect to the same:

```text
Host Network: CyberLab
```

The lab interface does not set:

```text
Default Gateway: leave blank
DNS:             leave blank
```

This way the VMs can communicate within `10.10.20.0/24`, but cannot reach the internet or home LAN through the lab interface.

## 6.3 Backend notes when creating VMs

When creating an ARM64 VM, choose:

```text
Create a New Virtual Machine
→ Virtualize
```

Do not choose the slower:

```text
Emulate
```

If your current UTM version cannot see the custom `CyberLab` Host Network under the Apple Virtualization backend, switch to the QEMU virtualization backend.

## 6.4 Connecting a VM to CyberLab

After a VM is created, fully shut it down, then open:

```text
Select the VM
→ Edit
→ Devices
→ Network
```

Set:

```text
Network Mode:          Host Network / macOS Host Only
Host Network:          CyberLab
Emulated Network Card: virtio-net-pci
```

Apply the same settings to all of:

- Kali Linux
- Ubuntu Server
- Windows 11
- Wazuh Server

Each VM must use a different MAC address.

## 6.5 CyberLab static IP plan

| VM | Static IP | Netmask | Gateway | DNS |
|---|---|---|---|---|
| Kali Linux | `10.10.20.10/24` | `255.255.255.0` | blank | blank |
| Ubuntu Server | `10.10.20.20/24` | `255.255.255.0` | blank | blank |
| Windows 11 | `10.10.20.30/24` | `255.255.255.0` | blank | blank |
| Wazuh Server | `10.10.20.40/24` | `255.255.255.0` | blank | blank |

## 6.6 Temporary internet during install

When installing an OS or downloading updates, add a temporary second interface:

```text
Network Mode: macOS Shared Network
IP:           DHCP / Automatic
Purpose:      temporary internet access
```

When updates are finished:

```text
1. Fully shut down the VM
2. Disable or remove the Shared Network interface
3. Keep only the CyberLab lab interface
4. Then start scanning or vulnerability testing
```

Do not connect deliberately vulnerable targets to:

```text
Bridged Network
```

## 6.7 Post-creation checklist

- [ ] `CyberLab` Host Network created in UTM
- [ ] All lab VMs set to the same `CyberLab`
- [ ] Each VM uses a different MAC address
- [ ] Each VM uses a different static IP
- [ ] CyberLab interface has no default gateway
- [ ] CyberLab interface has no DNS
- [ ] Shared Network disabled during vulnerability practice
- [ ] Bridged Network not enabled

---


# 7. Adding the lab interface to a VM

For each VM:

```text
1. Fully shut down the VM
2. Select the VM in UTM
3. Click Edit
4. Open Devices
5. Add or select Network
6. Set Network Mode to Host Network or macOS Host Only
7. Set Host Network to CyberLab
8. Save the configuration
```

Recommended virtual NIC model:

```text
virtio-net-pci
```

Each VM must have a different MAC address.

Do not copy the same MAC address onto multiple VMs.

---

# 8. Kali Linux network configuration

## 8.1 Recommended resources

```text
CPU:  6 cores
RAM:  8 GB
Disk: 60 GB
IP:   10.10.20.10/24
```

## 8.2 Check the interface name

```bash
ip -br link
```

```bash
nmcli device status
```

The interface name may be:

```text
eth0
ens5
enp0s1
```

## 8.3 GUI setup

Open:

```text
Settings
→ Network
→ Wired
→ IPv4
→ Manual
```

Fill in:

```text
Address: 10.10.20.10
Netmask: 255.255.255.0
Gateway: leave blank
DNS:     leave blank
```

After saving, disconnect and reconnect the interface.

## 8.4 Setup with nmcli

Show connection names:

```bash
nmcli connection show
```

Assume the connection name is:

```text
Wired connection 1
```

Run:

```bash
sudo nmcli connection modify "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses 10.10.20.10/24 \
  ipv4.gateway "" \
  ipv4.dns "" \
  ipv4.never-default yes
```

Reconnect:

```bash
sudo nmcli connection down "Wired connection 1"
sudo nmcli connection up "Wired connection 1"
```

Check:

```bash
ip -br address
```

```bash
ip route
```

Expected result:

```text
10.10.20.10/24
```

The lab interface should have no default route.

---

# 9. Ubuntu Server network configuration

## 9.1 Recommended resources

```text
CPU:  4 cores
RAM:  6 GB
Disk: 50 GB
IP:   10.10.20.20/24
```

## 9.2 Check the interface name

```bash
ip -br link
```

Assume the lab interface name is:

```text
enp0s1
```

## 9.3 Edit the Netplan configuration

```bash
ls /etc/netplan
```

Create or edit the configuration:

```bash
sudo nano /etc/netplan/01-cyberlab.yaml
```

Fill in:

```yaml
network:
  version: 2
  ethernets:
    enp0s1:
      dhcp4: false
      addresses:
        - 10.10.20.20/24
```

Note: replace `enp0s1` with the actual interface name.

Test the configuration:

```bash
sudo netplan try
```

Apply the configuration:

```bash
sudo netplan apply
```

Check:

```bash
ip -br address
```

```bash
ip route
```

Expected result:

```text
10.10.20.20/24
```

---

# 10. Windows 11 network configuration

## 10.1 Recommended resources

```text
CPU:  6 cores
RAM:  12 GB
Disk: 100 GB
IP:   10.10.20.30/24
```

## 10.2 Open network adapters

Press:

```text
Win + R
```

Enter:

```text
ncpa.cpl
```

Select the lab interface:

```text
Right-click the adapter
→ Properties
→ Internet Protocol Version 4
→ Properties
```

Fill in:

```text
IP address:      10.10.20.30
Subnet mask:     255.255.255.0
Default gateway: leave blank

Preferred DNS:   leave blank
Alternate DNS:   leave blank
```

## 10.3 Rename the adapter

Rename the lab interface to:

```text
LAB
```

## 10.4 Set the network to Private

In an administrator PowerShell:

```powershell
Set-NetConnectionProfile `
  -InterfaceAlias "LAB" `
  -NetworkCategory Private
```

Show the IP address:

```powershell
ipconfig
```

Or:

```powershell
Get-NetIPAddress -AddressFamily IPv4
```

## 10.5 Allow the lab network to ping Windows

Windows Firewall may block ICMP by default.

In an administrator PowerShell:

```powershell
New-NetFirewallRule `
  -DisplayName "Allow CyberLab ICMPv4" `
  -Direction Inbound `
  -Protocol ICMPv4 `
  -IcmpType 8 `
  -RemoteAddress 10.10.20.0/24 `
  -Action Allow
```

This rule only allows the CyberLab subnet to ping Windows.

---

# 11. Wazuh Server network configuration

## 11.1 Recommended resources

```text
CPU:  4 cores
RAM:  8 GB
Disk: 80 GB
IP:   10.10.20.40/24
```

If Wazuh is installed on Ubuntu Server, configure it with Netplan.

Check the interface:

```bash
ip -br link
```

Edit:

```bash
sudo nano /etc/netplan/01-cyberlab.yaml
```

Fill in:

```yaml
network:
  version: 2
  ethernets:
    enp0s1:
      dhcp4: false
      addresses:
        - 10.10.20.40/24
```

Apply:

```bash
sudo netplan try
```

```bash
sudo netplan apply
```

Confirm:

```bash
ip -br address
```

Expected:

```text
10.10.20.40/24
```

When installing the Wazuh agent, use this Manager address:

```text
10.10.20.40
```

---

# 12. Connectivity tests

## 12.1 Test other VMs from Kali

```bash
ping -c 3 10.10.20.20
```

```bash
ping -c 3 10.10.20.30
```

```bash
ping -c 3 10.10.20.40
```

## 12.2 Host discovery

```bash
sudo nmap -sn 10.10.20.0/24
```

Expected discovery:

```text
10.10.20.10    Kali Linux
10.10.20.20    Ubuntu Server
10.10.20.30    Windows 11
10.10.20.40    Wazuh Server
```

## 12.3 Test Ubuntu SSH from Windows

PowerShell:

```powershell
Test-NetConnection 10.10.20.20 -Port 22
```

If Ubuntu SSH is working, it should show:

```text
TcpTestSucceeded : True
```

## 12.4 Test Ubuntu SSH from Kali

```bash
ssh labuser@10.10.20.20
```

## 12.5 Test web services

If Ubuntu runs Juice Shop:

```text
http://10.10.20.20:3000
```

If Ubuntu runs WebGoat:

```text
http://10.10.20.20:8080/WebGoat
```

If the Wazuh Dashboard is up:

```text
https://10.10.20.40
```

---

# 13. Verify the lab network is isolated

With only the CyberLab interface attached, test from Kali or Ubuntu:

```bash
ping -c 3 1.1.1.1
```

This should fail.

Then test:

```bash
curl https://example.com
```

This should also fail.

View the routing table:

```bash
ip route
```

Normally you should only see:

```text
10.10.20.0/24 dev <lab interface>
```

You should not see:

```text
default via ...
```

This confirms the lab interface has no default gateway.

---

# 14. Temporary internet access

When you need to download tools or update the system, add a temporary second interface to the VM.

## Interface 1: CyberLab

```text
Purpose: attack/defense practice
IP:      static
Gateway: none
DNS:     none
```

## Interface 2: Shared Network

```text
Purpose: downloads and updates
IP:      DHCP
Gateway: automatic
DNS:     automatic
```

## Update workflow

```text
1. Stop vulnerable services
2. Shut down the VM
3. Add or enable the Shared Network interface
4. Start the VM
5. Update the system
6. Download the software you need
7. Shut down the VM
8. Disable or remove the Shared Network interface
9. Return to isolated lab mode
```

Ubuntu updates:

```bash
sudo apt update
sudo apt upgrade
```

Kali updates:

```bash
sudo apt update
sudo apt full-upgrade
```

---

# 15. Pre-attack checklist

- [ ] Kali connected to CyberLab
- [ ] Ubuntu connected to CyberLab
- [ ] Windows connected to CyberLab
- [ ] Wazuh connected to CyberLab
- [ ] All machines use different IPs
- [ ] All machines use different MAC addresses
- [ ] Lab interface has no default gateway
- [ ] Lab interface has no DNS
- [ ] Shared Network disabled
- [ ] Bridged Network not enabled
- [ ] Targets cannot reach the internet
- [ ] VM snapshots created
- [ ] Confirmed the scan targets belong to your own lab

---

# 16. Recommended snapshot names

## Kali Linux

```text
01-Kali-Clean
02-Kali-Tools-Installed
03-Kali-Before-Lab
```

## Ubuntu Server

```text
01-Ubuntu-Clean
02-Ubuntu-SSH-Apache
03-Ubuntu-JuiceShop
04-Ubuntu-Before-Attack
```

## Windows 11

```text
01-Windows-Clean
02-Windows-Lab-Configured
03-Windows-Before-Attack
```

## Wazuh

```text
01-Wazuh-Clean
02-Wazuh-Installed
03-Wazuh-Agents-Connected
```

---

# 17. First scan exercise

## 17.1 Check the Kali network

```bash
ip -br address
```

```bash
ip route
```

## 17.2 Discover lab devices

```bash
sudo nmap -sn 10.10.20.0/24
```

## 17.3 Scan Ubuntu's common ports

```bash
nmap 10.10.20.20
```

## 17.4 Service version scan

```bash
sudo nmap -sV 10.10.20.20
```

## 17.5 Scan all TCP ports

```bash
sudo nmap -sS -p- 10.10.20.20
```

## 17.6 Scan Windows

```bash
sudo nmap -sV 10.10.20.30
```

## 17.7 Scan the Wazuh Server

```bash
sudo nmap -sV 10.10.20.40
```

## 17.8 Save scan results

```bash
sudo nmap -sV 10.10.20.20 -oA ubuntu-initial-scan
```

This produces:

```text
ubuntu-initial-scan.nmap
ubuntu-initial-scan.gnmap
ubuntu-initial-scan.xml
```

---

# 18. Wireshark filters

View the lab subnet:

```text
net 10.10.20.0/24
```

View Ubuntu traffic:

```text
ip.addr == 10.10.20.20
```

View Windows traffic:

```text
ip.addr == 10.10.20.30
```

View TCP SYN:

```text
tcp.flags.syn == 1
```

View SSH:

```text
tcp.port == 22
```

View HTTP:

```text
http
```

View ICMP:

```text
icmp
```

View ARP:

```text
arp
```

---

# 19. Safety rules

Only test:

- VMs you created yourself
- devices you own
- deliberately vulnerable targets
- systems you are explicitly authorized to test
- professional cybersecurity training platforms

Do not scan:

- school networks
- company networks
- public Wi-Fi
- a neighbour's network
- random public IPs
- unauthorized websites
- cloud servers that are not yours
- household devices outside the lab scope

Do not run against real networks:

- password brute-forcing
- exploitation
- ARP spoofing
- malware
- denial-of-service testing
- heavy automated scanning
- unauthorized web testing

---

# 20. Lab log template

## Lab Information

```text
Lab Name:
Date:
Student:
Target:
Attacker:
Network:
Authorization:
```

## Objective

```text
What is the goal of this lab?
```

## Scope

```text
IPs allowed to test:
Devices not to test:
Tools allowed:
```

## Network Diagram

```text
Kali 10.10.20.10
        │
        │ CyberLab
        │
Ubuntu 10.10.20.20
```

## Commands

```bash
# Record the commands you used here
```

## Findings

| Finding | Evidence | Risk | Recommendation |
|---|---|---|---|
| Example: SSH open to the whole subnet | TCP 22 open | Medium | Restrict source IPs |

## Screenshots

```text
Add screenshot file names or paths.
```

## Remediation

```text
Record what fixes you applied.
```

## Verification

```text
Re-scan after remediation and record the results.
```

## Lessons Learned

```text
What did you learn in this lab?
Which steps went wrong?
What should you improve next time?
```

---

# 21. Final configuration summary

```text
CyberLab: 10.10.20.0/24

Kali Linux:
10.10.20.10

Ubuntu Server:
10.10.20.20

Windows 11:
10.10.20.30

Wazuh Server:
10.10.20.40

Subnet Mask:
255.255.255.0

Gateway:
none

DNS:
none

Internet:
off by default

Temporary Internet:
UTM Shared Network

Do not use:
Bridged Network
```

---

# 22. Core principles

1. All four VMs must connect to the same `CyberLab` Host Network.
2. Each machine uses a different static IP.
3. The CyberLab interface has no gateway and no DNS.
4. Disable Shared Network during vulnerability practice.
5. Never connect a vulnerable target with Bridged Network.
6. Take a snapshot before attacking.
7. Only scan and attack targets you own or are explicitly authorized to test.
8. For every lab, record the scan, evidence, remediation and verification results.

---




# How penetration testers actually work

Real penetration testing doesn't start by "attacking." The standard process is usually:

> **Get written authorization → confirm scope → reconnaissance → scanning → enumeration → verify vulnerabilities → control impact → write the report → assist with remediation → re-test**

In real work, communication, documentation and report writing are usually as important as using the tools.

---

## 1. Confirm authorization and scope

Before testing, the client must explicitly agree.

Typically you confirm:

- which IPs, domains and servers may be tested
- which systems must not be tested
- the test dates and times
- whether password testing is allowed
- whether exploitation is allowed
- whether social engineering is allowed
- whether production environments may be tested
- who to contact if something breaks
- under what conditions you must stop immediately

Key terms:

| Term | Meaning |
|---|---|
| Scope | What may be tested |
| Rules of Engagement | How testing must be conducted |
| Authorization | Permission to test |

Without authorization, even just scanning ports can create legal problems.

---

## 2. Reconnaissance

Testers understand the target first, rather than attacking immediately.

### 2.1 Passive reconnaissance

Avoid touching the target system directly where possible — for example:

- the company website
- DNS records
- public IP ranges
- public employee information
- technical job postings
- public code repositories
- leaked email addresses
- the cloud services or technologies in use

For example, a job posting might reveal the following tech environment:

```text
Windows Server
Active Directory
Microsoft 365
AWS
Fortinet Firewall
```

This information helps the tester understand the enterprise environment.

### 2.2 Active reconnaissance

Communicate directly with the authorized target — for example:

- pinging hosts
- port scanning
- DNS queries
- probing web servers
- identifying service versions

---

## 3. Host and port scanning

The tester needs to build an asset inventory:

```text
Which hosts are online?
Which ports are open?
Which services are running?
What are the service versions?
Are there unnecessary services?
```

Common tools:

- Nmap
- Masscan
- Nessus
- OpenVAS / Greenbone
- Nuclei

In the isolated lab, discover hosts first:

```bash
nmap -sn 10.10.20.0/24
```

Then identify the target services:

```bash
nmap -sV 10.10.20.20
```

The result might look like:

```text
22/tcp    open    ssh
80/tcp    open    http
3000/tcp  open    web application
```

Scan results only indicate "a possible problem"; they do not confirm a vulnerability.

---

## 4. Enumeration

Enumeration goes deeper than a plain scan.

Testers try to learn:

- web directories and APIs
- username formats
- SMB shares
- domains and subdomains
- DNS information
- SSL/TLS certificates
- software versions
- login pages
- authentication methods
- file and folder permissions
- Active Directory information

For example, after finding TCP 22 open, you still need to confirm:

- whether password login is allowed
- whether direct root login is allowed
- whether an old version is in use
- whether it is open to all networks
- whether failed logins are logged
- whether account lockout or rate limiting is configured

---

## 5. Vulnerability analysis

Testers connect scan results to real risk.

Main questions:

- do the software versions have known vulnerabilities
- is anything misconfigured
- do default accounts exist
- are weak passwords in use
- are security updates missing
- are file permissions too broad
- are insecure services exposed
- is sensitive data leaking
- is multi-factor authentication missing

Common vulnerability types:

- weak passwords
- default credentials
- unpatched software
- wrong permissions
- SQL Injection
- Cross-Site Scripting
- Broken Access Control
- insecure file upload
- information disclosure
- misconfigured cloud permissions
- unnecessary open ports

---

## 6. Vulnerability verification

Automated scanners can produce false positives, so pentesters verify manually.

For example, a scanner reports:

> The server may use a weak TLS configuration.

The tester needs to confirm further:

- which TLS versions are actually supported
- whether weak cipher suites are enabled
- whether it can really be exploited
- what impact a successful exploit could cause

The goal of verification is not to break the system, but to gather enough evidence to prove the risk.

Common evidence includes:

- screenshots
- HTTP requests and responses
- system logs
- affected ports
- software versions
- test files that could be read
- privileges obtained by a test account

---

## 7. Limited exploitation

Testers only exploit when the authorized scope allows it.

The goal is usually to answer:

> How much impact can this vulnerability actually cause?

Example:

```text
Normal user
→ accesses data that isn't theirs
→ privilege escalation
→ gains administrator rights
```

Another example:

```text
Web vulnerability
→ reads the test database
→ proves sensitive data could leak
```

Formal testing usually follows:

- use test accounts
- read only the minimum data needed to prove the risk
- do not modify real business data
- do not run denial-of-service attacks
- do not keep long-term access
- report high-risk issues promptly
- strictly follow the rules of engagement

---

## 8. Privilege escalation

After obtaining a normal account, the tester checks whether privileges can be escalated.

Common checks:

- can a normal Linux user become root
- can a normal Windows user become Administrator
- are service account privileges too broad
- is sudo misconfigured
- are scheduled tasks insecure
- can system files be modified by normal users
- are cleartext passwords stored
- are service configs or path permissions wrong

---

## 9. Lateral movement

In enterprise tests, after landing on one machine the tester may try to reach other authorized systems.

Example:

```text
Employee laptop
→ file server
→ application server
→ domain controller
```

Main checks:

- are passwords reused
- is the network properly segmented
- can normal users reach the server subnet
- are management protocols exposed
- are admin credentials left on ordinary machines
- can the firewall block lateral access

In a home lab you can simulate the basic flow:

```text
Kali
→ Ubuntu target
→ check whether Ubuntu can reach Windows
→ Wazuh observes the logs and alerts
```

When starting out, master scanning, logs and firewalls first, before moving on to complex lateral-movement labs.

---

## 10. Check detection capability

Modern penetration testing doesn't only ask:

> Can we get into the system?

It also asks:

> Can the security team detect it?

Things to observe:

- does Windows Event Viewer record the login events
- do Linux logs record the SSH failures
- does Wazuh raise an alert
- does the firewall log the connections
- does an IDS/IPS detect the scan
- does the security team respond to the abnormal activity

Home lab example:

```text
Kali launches an Nmap scan
        ↓
Ubuntu logs the network or login activity
        ↓
Wazuh collects the logs
        ↓
Check whether an alert is generated
```

---

## 11. Clean up the test environment

After testing, restore the environment, including:

- delete test accounts
- delete uploaded test files
- delete test keys
- close temporarily opened ports
- restore modified configs
- stop test services
- remove scheduled tasks created during testing
- tell the client what changes were made during the test

This cleanup isn't to hide the attack — it's to avoid leaving the client with new security risks.

---

## 12. Writing the penetration test report

The report is usually the most important deliverable.

### 12.1 Executive Summary

For management, in plain language, explaining:

- the overall risk
- the most serious issues
- the possible business impact
- what should be fixed first

### 12.2 Technical Findings

For IT and security staff.

| Item | Content |
|---|---|
| Finding | Vulnerability name |
| Severity | Critical / High / Medium / Low |
| Affected Asset | The affected IP, domain or system |
| Evidence | Screenshots, logs, requests and responses |
| Impact | What an attacker could do |
| Reproduction | How to reproduce it in an authorized environment |
| Remediation | How to fix it |
| References | CVE, vendor docs or security standards |

Example finding:

```text
Issue:
SSH is open to the entire lab subnet.

Evidence:
TCP 22 on 10.10.20.20 is open to all lab hosts.

Impact:
Any device that reaches the lab subnet can attempt an SSH login.

Recommendation:
Only allow the management host 10.10.20.10 to reach TCP 22,
and disable root password login.
```

---

## 13. Re-test after remediation

After the client finishes remediation, the tester checks again.

First scan:

```text
22/tcp open ssh
80/tcp open http
23/tcp open telnet
```

After remediation:

```text
22/tcp filtered
80/tcp open
23/tcp closed
```

Re-testing confirms:

- the vulnerability really is fixed
- no new problems were introduced
- the firewall rules took effect
- the risk has been reduced

---

# The difference between vulnerability scanning and penetration testing

| Type | Main work |
|---|---|
| Vulnerability Scan | Use tools to automatically find possible vulnerabilities |
| Vulnerability Assessment | Analyze, confirm and prioritize vulnerabilities |
| Penetration Test | Verify, within the authorized scope, whether vulnerabilities can be exploited |
| Red Team | Simulate a real attacker, testing people, processes and technology |
| Security Audit | Check whether systems comply with policies, standards or regulations |

A scanner finding 100 issues does not mean a penetration test is done.

The tester still needs to judge:

- which are false positives
- which can really be exploited
- which cause real business impact
- in what order they should be fixed

---

# A day in the life of a penetration tester

A typical workday might be:

```text
09:00  Confirm the test scope with the client
09:30  Review assets and scan results
10:30  Manually verify high-risk issues
12:00  Organize evidence and lab notes
13:00  Test web logins and access control
15:00  Review logs and detection alerts
16:00  Report urgent vulnerabilities
16:30  Write technical findings and remediation advice
```

A pentester does not spend eight hours running attack tools.

---

# A complete practice workflow for the home lab

Each lab can follow this order:

```text
1. Create a snapshot
2. Confirm the test scope
3. Host discovery
4. Port scanning
5. Service enumeration
6. Manual verification
7. Review Windows/Linux logs
8. Review Wazuh alerts
9. Configure the firewall or remediate the vulnerability
10. Re-scan
11. Write a one-page report
12. Restore the snapshot
```

---

# First recommended project: Ubuntu security assessment

## Project goal

Perform a complete, controlled security assessment of the Ubuntu target `10.10.20.20`.

## What to check

- which ports are open
- which services are running
- whether SSH is secure
- which port Juice Shop uses
- whether UFW is enabled
- whether failed logins leave logs
- whether Wazuh detects the scan
- whether the number of open ports drops after remediation

## Suggested workflow

### 1. Confirm the authorized scope

```text
Attacker:    10.10.20.10
Target:      10.10.20.20
Network:     CyberLab 10.10.20.0/24
Allowed:     scanning, service enumeration, log review, firewall testing
Not Allowed: DoS, destroying data, attacking the real network
```

### 2. Host discovery

```bash
sudo nmap -sn 10.10.20.0/24
```

### 3. Basic port scan

```bash
nmap 10.10.20.20
```

### 4. Service version identification

```bash
sudo nmap -sV 10.10.20.20
```

### 5. Full port scan

```bash
sudo nmap -sS -p- 10.10.20.20
```

### 6. Check the target logs

Ubuntu:

```bash
sudo journalctl -u ssh
```

```bash
sudo tail -f /var/log/auth.log
```

### 7. Review the firewall

```bash
sudo ufw status verbose
```

### 8. Remediate

Example:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 10.10.20.10 to any port 22 proto tcp
sudo ufw enable
```

### 9. Re-scan

```bash
sudo nmap -sV -p- 10.10.20.20
```

### 10. Write the report

Record:

- the scan results
- the issues found
- the risk levels
- the evidence
- the remediation actions
- the post-remediation verification
- what you learned in this lab

---

# The core professional mindset

A real penetration tester is not just someone who can "get into a system."

They must:

> **Have authorization, control risk, protect data, preserve evidence, explain business impact, and propose actionable remediation.**

Attack tools are only part of the job.

The truly valuable skills are:

- judgment
- verification
- documentation
- communication
- reporting
- remediation advice
- re-testing
