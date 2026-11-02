Nmap scan 

nmap -sC -sV -A 10.129.227.227
Starting Nmap 7.95 ( https://nmap.org ) at 2026-11-02 05:45 EST
Nmap scan report for 10.129.227.227
Host is up (0.26s latency).
Not shown: 995 closed tcp ports (conn-refused)
PORT      STATE    SERVICE VERSION
22/tcp    open     ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 9e:1f:98:d7:c8:ba:61:db:f1:49:66:9d:70:17:02:e7 (RSA)
|   256 c2:1c:fe:11:52:e3:d7:e5:f7:59:18:6b:68:45:3f:62 (ECDSA)
|_  256 5f:6e:12:67:0a:66:e8:e2:b7:61:be:c4:14:3a:d3:8e (ED25519)
80/tcp    open     http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Is my Website up ?
146/tcp   filtered iso-tp0
1011/tcp  filtered unknown
12000/tcp filtered cce4x
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Added siteisup.htb to /etc/hosts

just seen a application where we can check weather the given site is up or down 

and i just ran a 

ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -u http://siteisup.htb -H "Host: FUZZ.siteisup.htb" -fs 138

amd found that dev is a sub domain for the siteisup.htb then added it as well to hosts

but getting forbidden on dev 

tried feroxbuster on the site and found .git "directory listing 

wget -m -np http://siteisup.htb/dev/.git/

Checking the content within that .git folder, we can see that there are some staged changes to detele some files from the last commit:

usin ls -la

We run the following commands to restore them:


git restore --staged .
git restore .

ls -la

  total 40
  drwxr-xr-x 3 kali kali 4096 Sep 10 13:48 .
  drwxr-xr-x 5 kali kali 4096 Sep 10 13:37 ..
  -rw-r--r-- 1 kali kali   59 Sep 10 13:48 admin.php
  -rw-r--r-- 1 kali kali  147 Sep 10 13:48 changelog.txt
  -rw-r--r-- 1 kali kali 3145 Sep 10 13:48 checker.php
  drwxr-xr-x 8 kali kali 4096 Sep 10 13:48 .git
  -rw-r--r-- 1 kali kali  117 Sep 10 13:48 .htaccess
  -rw-r--r-- 1 kali kali  273 Sep 10 13:48 index.php
  -rw-r--r-- 1 kali kali 5531 Sep 10 13:48 stylesheet.css


But what it’s more important, how we can access to the server, this is described in the .htaccess file:

    .htaccess

SetEnvIfNoCase Special-Dev "only4dev" Required-Header
Order Deny,Allow
Deny from All
Allow from env=Required-Header

So we need the following header to access the dev subdomain: Special-Dev: only4dev. That was why we could now get anything from the dev subdomain. With an easy curl request we can see that we can now accesss to the server:

now able to access the Dev.siteisup.htb






