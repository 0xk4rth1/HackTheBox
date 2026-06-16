# HackTheBox - Facts

## Machine Information

| Field | Details |
|---|---|
| Platform | HackTheBox |
| Machine | Facts |
| OS | Linux |
| Difficulty | Easy |

---

## Description

The **Facts** machine focuses on exploiting vulnerabilities in the Camaleon CMS. Initial access is obtained by abusing **CVE-2025-2304 (Mass Assignment)** to escalate privileges to an administrator account. Administrative access exposes a **Local File Inclusion (LFI)** vulnerability that allows arbitrary file reads, including the user flag. Further enumeration reveals AWS S3 credentials stored in the application configuration. The S3 bucket contains an encrypted SSH private key, which is cracked offline to gain SSH access. Finally, privilege escalation is achieved through a misconfigured **sudo** rule allowing execution of **facter** with a custom Ruby module.

Main concepts learned:

- Mass Assignment vulnerabilities
- Local File Inclusion (LFI)
- AWS S3 enumeration
- SSH private key cracking with John the Ripper
- Privilege escalation via Facter custom modules

---

## Disclaimer

This walkthrough is for educational purposes only. Perform these techniques only in authorized environments such as HackTheBox labs.

# Enumeration

Three ports were identified:

- 22/tcp — SSH
- 80/tcp — HTTP
- 54321/tcp — AWS S3 endpoint

Initial enumeration included Nmap service discovery and web application reconnaissance. The web service hosted a Camaleon CMS instance with an accessible registration page.

# Service Enumeration

## HTTP Enumeration

Interesting findings:

- Camaleon CMS
- User registration enabled
- Administrative login panel
- Vulnerable password update endpoint
- Local File Inclusion endpoint:

`/admin/media/download_private_file`

## SSH Enumeration

SSH became useful after obtaining the encrypted private key from the S3 bucket.

# Vulnerability Analysis

## CVE-2025-2304 - Mass Assignment

### Description

The password update endpoint failed to properly filter user-controlled parameters. By supplying:

`password[role]=admin`

the application updated the account role during a profile update request.

### Impact

The vulnerability allowed privilege escalation from a normal user to an administrator.

## Local File Inclusion

### Description

The authenticated download endpoint accepted directory traversal sequences, allowing arbitrary file reads.

Example:

`http://facts.htb/admin/media/download_private_file?file=../../../../../../etc/passwd`

The same vulnerability was used to retrieve:

- `/home/william/user.txt`

and later application configuration containing AWS credentials.

# Initial Access

1. Register a normal account.
2. Exploit CVE-2025-2304 using the password update endpoint (Mass Assignment)
3. Obtain administrator access.
4. Abuse the LFI endpoint.
5. Read `/home/william/user.txt`.

User flag:

`REDACTED`

# AWS Enumeration

Filesystem configuration exposed:

- Access Key: `AKIA654DC625E1DC4740`
- Secret Key: `REDACTED`
- Bucket: `randomfacts`
- Region: `us-east-1`
- Endpoint: `http://facts.htb:54321`

Buckets discovered:

- internal
- randomfacts

The internal bucket contained:

- `.ssh/authorized_keys`
- `.ssh/id_ed25519`

The SSH private key was downloaded and cracked using John the Ripper.

Recovered passphrase:

`REDACTED`

The public key identified the owner as:

`trivia@facts.htb`

SSH access was then obtained as **trivia**.

# Privilege Escalation

## Enumeration

`sudo -l`

Output showed:

`(ALL) NOPASSWD: /usr/bin/facter`

## Exploitation

Facter supports loading custom Ruby facts.

A malicious Ruby module executed:

/tmp/exploit.rb:

`Facter.add('exploit') do
  setcode do
    Facter::Core::Execution.exec('cp /bin/bash /var/tmp/test; chmod 6777 /var/tmp/test')
    'Malicious fact has run'
  end
end`

Running Facter with:

`sudo /usr/bin/facter --custom-dir /tmp/ exploit`

created a SUID(root) shell.

Executing:

`/var/tmp/test -p`

provided a root shell.

# Root Access

`whoami`

Output:

`root`

# Root Flag

`cat /root/root.txt`

`REDACTED`

# Attack Chain Summary

Enumeration

↓

Register User

↓

Mass Assignment (CVE-2025-2304)

↓

Administrator Access

↓

Local File Inclusion

↓

Read AWS Credentials

↓

Enumerate S3 Bucket

↓

Recover SSH Private Key

↓

Crack SSH Passphrase

↓

SSH as trivia

↓

Abuse sudo Facter

↓

Root

# Tools Used

- Nmap
- Burp Suite
- Python
- AWS CLI
- John the Ripper
- ssh-keygen
- SSH
- Facter

# Lessons Learned

- Validate server-side authorization during profile updates.
- Prevent Mass Assignment by explicitly permitting fields.
- Sanitize file paths to eliminate directory traversal.
- Protect secrets stored within application configuration.
- Avoid exposing privileged cloud credentials.
- Review sudo permissions for custom module execution.
