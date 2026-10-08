# Multi-Threaded-Port-Scanner-Firewall-Evasion-Lab
Purple Team Methodology
# 🟣 Purple Team Lab: Multi-Threaded Port Scanner & Log Footprint Analysis

## 📌 Executive Summary
This project documents a Purple Team laboratory exercise designed to evaluate the behavioral footprint of a custom-built, high-speed TCP port scanner (Red Team) against a modern Kali Linux Virtual Machine (Blue Team). 

The primary objective was to establish a correlated feedback loop between offensive traffic execution and defensive system logging. By cross-referencing timestamps between both hosts, this lab visualizes the exact network handshake footprint left in system logs during automated scanning phases, while documenting remediation techniques against modern Linux routing architectures.

---

## 💻 Technical Environment Layout
*   **Attacking Host (Red Team):** Native Kali Linux Laptop
*   **Target Host (Blue Team):** Kali Linux VM (Bridged Network Mode)
*   **Primary Target Services:** 
    *   TCP Port 22 (OpenSSH Service)
    *   TCP Port 8080 (Python `http.server` daemon)

---

## 🔴 Red Team Operations (The Attack Phase)

### 1. Custom Weaponization
A custom script was engineered in Python utilizing the built-in `socket` and `concurrent.futures` libraries. Rather than relying on slow, sequential loops, the script implements an aggressive, concurrent worker pool.

*   **Capabilities:** Multi-threaded concurrency (`max_workers=100`), automated TCP handshake connection verification (`connect_ex`), dynamic banner-grabbing via payload injection (`GET / HTTP/1.1\r\n\r\n`), and automated socket disposal.

```python
def scan_port(port):
    try:
        with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as sock:
            sock.settimeout(0.6)
            result = sock.connect_ex((target_ip, port))
            if result == 0:
                sock.send(b"GET / HTTP/1.1\r\n\r\n")
                ready_data = sock.recv(1024)
                banner = ready_data.decode('utf-8', errors='ignore').strip()
                print(f"{port:<10} {'OPEN':<10} {banner}")
    except Exception:
        pass
```

### 2. Attack Execution Timeline
By distributing traffic over 100 concurrent threads, the script enumerates the target environment in seconds. 

<img width="577" height="480" alt="image" src="https://github.com/user-attachments/assets/54fac949-541b-4089-929a-9d5960311869" />

*   **Initial Scan (20:46:08):** The single-threaded iteration successfully captures the initial web playground on `Port 8080: OPEN`.
*   **Code Refactoring:** Dropped into `nano portscanner.py` to upgrade the script architecture into a multi-threaded, banner-grabbing layout.
*   **Upgraded Scan (20:57:21):** The new concurrent script executes, prompts for the target IP address, and successfully records `Port 22: OPEN`.

---

## 🔵 Blue Team Operations & Forensic Timeline

The core focus of Purple Teaming is validating whether an attack is visible to the defender. Checking the internal system journals on the Kali VM revealed a clear chronological log footprint matching the attack vector.

<img width="748" height="132" alt="image" src="https://github.com/user-attachments/assets/c9041a67-383c-431c-b789-2d0a92d24a00" />

### ⏱️ Correlated Timestamp Analysis
1.  **20:56:08** – The Blue Team initializes the OpenSSH server daemon via systemd (`Starting ssh.service... Server listening on 0.0.0.0 port 22`).
2.  **20:57:21** – **The Intrusion.** At the exact microsecond the Red Team script executes, the VM's kernel registers the scanning thread hitting port 22:  
    `Oct 07 20:57:21 Kali sshd[...]: banner exchange: Connection from [LAPTOP_IP] port ...`

---

## 🛠️ Defensive Troubleshooting & Hardening

### 1. Resolving Zsh Terminal Parse Errors
When attempting to build manual firewall filters via the modern `nftables` framework, standard commands can trigger syntax errors due to terminal shell variations:
```text
zsh: parse error near ']'
```
*   **Root Cause:** Modern Kali Linux utilizes the **Zsh** shell as default. Zsh interprets trailing brackets `]` and curly braces `{}` as specialized command grouping operators.
*   **Remediation:** Enclose the argument parameter syntax inside single quotes (`'`) to bypass shell expansion logic:
```bash
sudo nft add chain ip purple_team input_filter '{ type filter hook input priority 0 ; policy drop ; }'
```

### 2. Eliminating the Attack Surface
To complete the remediation lifecycle and permanently secure the device, the underlying service daemon was completely disabled to strip the host of its open attack surface:
```bash
sudo systemctl stop ssh
sudo systemctl disable ssh
```
*Verification via `ss -antpl | grep :22` confirmed that zero network listeners remained active on the interface.*

---

---

## 📖 Key Takeaways & Lessons Learned

### 🛠️ Technical Security Analysis (Cyber Jargon)
1. **Concurrency Defeats Reactive Logging:** High-speed multi-threaded scripts can complete an entire enumeration phase before standard log-parsing security tools can process and trigger a dynamic firewall rule.
2. **The Layer 2 Hypervisor Blind Spot:** Local subnet traffic communicating via virtual hypervisor bridging interfaces frequently bypasses traditional user-space firewall limits (`default allow incoming`), as the packets do not traverse the standard external routing stack.
3. **True Defense is Zero Trust:** Security must be established using a **Default Deny** posture at the lowest kernel level (`nftables`), forcing unauthorized automated traffic frames to drop silently without returning a TCP handshake response.

### 📢 The Executive Summary (Plain English)
If you explain this lab to someone who has never touched cybersecurity before, here is the real story of what happened:
1. **You Can't Hide in the Digital World:** Building a high-speed scanner is like having a robot knock on 9,000 doors in a house in under two seconds. Even if the robot runs away instantly, the house still logs a record that someone turned the doorknob. The moment our script touched Port 22, the computer permanently wrote down exactly when it happened.
2. **Standard Security Has Blind Spots:** We found out that standard firewalls are built to block strangers coming in from the outside internet, but they completely ignore "trusted neighbors" sitting on the exact same local Wi-Fi network. Because my laptop and the target machine were on the same network, our initial security rules just let the scanner slip right past.
3. **The "Locked Vault" Approach Wins:** To finally stop the attack, we had to switch from an "allow everything except known bad guys" mindset to a strict **"block absolutely everything by default"** mindset. By shutting down the open doors completely and setting the computer to ignore untrusted local traffic, the scanning script was left completely blind.

