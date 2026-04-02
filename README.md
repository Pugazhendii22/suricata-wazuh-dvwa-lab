# 🛡️ Web Attack Detection Lab — Suricata + Wazuh + DVWA

![Suricata](https://img.shields.io/badge/Suricata-IDS-orange?style=flat-square&logo=suricata)
![Wazuh](https://img.shields.io/badge/Wazuh-4.7.5-blue?style=flat-square)
![DVWA](https://img.shields.io/badge/DVWA-Vulnerable%20App-red?style=flat-square)
![Platform](https://img.shields.io/badge/Platform-Ubuntu%20Linux-purple?style=flat-square&logo=ubuntu)
![Status](https://img.shields.io/badge/Status-Complete-brightgreen?style=flat-square)

> A hands-on SOC home lab that simulates real-world web attacks against a deliberately vulnerable application and detects them in real time using Suricata IDS integrated with Wazuh SIEM — all on bare-metal hardware.

---

## 📌 Overview

This lab replicates a realistic SOC monitoring scenario using entirely open-source tools. A vulnerable web application (DVWA) is deployed as the attack target. SQL injection and automated NIDS test traffic are launched from an attacker machine, intercepted by Suricata IDS running on the victim server, and forwarded as structured alerts to a Wazuh SIEM for correlation, triage, and MITRE ATT&CK mapping.

**No virtual machines. All bare-metal hardware on a local LAN.**

---

## 🧱 Architecture

```
┌──────────────────┐       HTTP Attack Traffic        ┌──────────────────────────────┐
│                  │ ─────────────────────────────►   │      Victim Server           │
│   Kali Linux     │                                   │                              │
│   (Attacker)     │   Tools: sqlmap, tmNIDS           │  ┌──────────────────────┐   │
│                  │                                   │  │  DVWA (Apache/PHP)   │   │
└──────────────────┘                                   │  └──────────────────────┘   │
                                                       │  ┌──────────────────────┐   │
                                                       │  │   Suricata IDS       │   │
                                                       │  │   49,300+ ET Rules   │   │
                                                       │  └──────────┬───────────┘   │
                                                       │             │ eve.json       │
                                                       │  ┌──────────▼───────────┐   │
                                                       │  │   Wazuh Agent        │   │
                                                       │  └──────────┬───────────┘   │
                                                       └─────────────┼───────────────┘
                                                                     │ TCP 1514
                                                                     ▼
                                                       ┌──────────────────────────────┐
                                                       │       Wazuh Server           │
                                                       │       (Laptop - Ubuntu)      │
                                                       │   Dashboard + Correlation    │
                                                       └──────────────────────────────┘
```

### Machine Roles

| Machine | Hostname | Role | OS | IP |
|---|---|---|---|---|
| Laptop | pugal-TravelLite | Wazuh Server v4.7.5 | Ubuntu | 192.168.0.x |
| Extra PC 1 | suricata-nids | Suricata IDS + Wazuh Agent | Ubuntu | 192.168.0.155 |
| Extra PC 2 | pugal1-H81 | Victim (DVWA) + Suricata + Wazuh Agent | Ubuntu | 192.168.0.203 |
| Kali SSD | kali | Attacker | Kali Linux | 192.168.0.x |

All machines on the same LAN (192.168.0.0/24).

---

## 🧰 Tools & Technologies

| Tool | Version | Purpose |
|---|---|---|
| [Suricata](https://suricata.io/) | 6.0.4 | Network IDS — real-time traffic inspection |
| [Wazuh](https://wazuh.com/) | 4.7.5 | SIEM — alert collection, correlation, MITRE mapping |
| [DVWA](https://github.com/digininja/DVWA) | Latest | Deliberately Vulnerable Web App — attack target |
| [Emerging Threats Rules](https://rules.emergingthreats.net/) | Open | 49,300+ Suricata detection signatures |
| [sqlmap](https://sqlmap.org/) | 1.9.8 | Automated SQL injection and database enumeration |
| [tmNIDS](https://github.com/3CORESec/testmynids.org) | Latest | NIDS detection tester — simulates 16 attack categories |
| Apache2 + MariaDB + PHP | — | DVWA web stack |

---

## ⚙️ Setup Guide

### Step 1 — Deploy DVWA (Victim Server)

Install the web stack:
```bash
sudo apt -y install apache2 mariadb-server php php-mysqli php-gd libapache2-mod-php git
sudo mysql_secure_installation
```

Set up the database:
```sql
sudo mysql -u root -p
create database dvwa;
create user 'dvwa'@'localhost' identified by 'password';
grant all privileges on dvwa.* to 'dvwa'@'localhost';
flush privileges;
exit;
```

Deploy DVWA:
```bash
sudo git clone https://github.com/digininja/DVWA.git /var/www/html/dvwa
sudo cp /var/www/html/dvwa/config/config.inc.php.dist /var/www/html/dvwa/config/config.inc.php
# Edit config: set db_user=dvwa, db_password=password, db_database=dvwa
sudo chmod -R 755 /var/www/html/dvwa
sudo chown -R www-data:www-data /var/www/html/dvwa
sudo a2enmod rewrite
sudo systemctl restart apache2
```

Access `http://<victim-ip>/dvwa/setup.php` → Create/Reset Database → Login: `admin / password` → Set security to **Low**.

---

### Step 2 — Install & Configure Suricata

```bash
sudo add-apt-repository ppa:oisf/suricata-stable -y
sudo apt install suricata -y
sudo suricata-update   # Downloads Emerging Threats Open rules
```

Update `/etc/suricata/suricata.yaml`:
```yaml
# Point to the correct rule path
default-rule-path: /var/lib/suricata/rules

rule-files:
  - suricata.rules
  - local.rules

# Set correct network interface
af-packet:
  - interface: enp2s0

# Enable structured JSON output for Wazuh
outputs:
  - eve-log:
      enabled: yes
      filename: eve.json
```

```bash
sudo systemctl restart suricata
sudo grep "rules loaded" /var/log/suricata/suricata.log
# ✅ Expected: 49300 rules successfully loaded
```

---

### Step 3 — Install & Configure Wazuh Agent

Install the version matching the server (v4.7.5):
```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg \
  --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
sudo chmod 644 /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" \
  | sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update
WAZUH_MANAGER="<wazuh-server-hostname>" sudo apt install wazuh-agent=4.7.5-1 -y
```

Configure `/var/ossec/etc/ossec.conf`:
```xml
<client>
  <server>
    <address>your-wazuh-server-hostname</address>
    <port>1514</port>
    <protocol>tcp</protocol>
  </server>
</client>

<!-- Forward Suricata alerts to Wazuh -->
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/suricata/eve.json</location>
</localfile>
```

Register and start the agent:
```bash
# On Wazuh Server — add agent and extract key
sudo /var/ossec/bin/manage_agents

# On agent machine — import the key
sudo /var/ossec/bin/manage_agents

sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
sudo grep "status" /var/ossec/var/run/wazuh-agentd.state
# ✅ Expected: status='connected'
```

---

## 🔥 Attack Scenarios & Detection Results

### Scenario 1 — NIDS Detection Testing with tmNIDS

tmNIDS is a framework by 3CORESec that generates traffic matching 16 categories of known attack signatures to validate NIDS detection capability.

**Run from the Suricata machine:**
```bash
curl -sSL https://raw.githubusercontent.com/3CORESec/testmynids.org/master/tmNIDS \
  -o /tmp/tmNIDS && chmod +x /tmp/tmNIDS && /tmp/tmNIDS
# Select: 16 - CHAOS! RUN ALL!
```

**Detections triggered in Wazuh:**

| Alert Description | Category |
|---|---|
| ET MALWARE W32/LetsGo.APT Sleep CnC Beacon | Malware C2 |
| ET MALWARE Covenant Framework HTTP Beacon | Malware C2 |
| ET MALWARE Possible Winnti TLS SNI Observed | Malware C2 |
| ET ADWARE_PUP yupsearch.com Spyware Install | Adware/PUP |
| ET ADWARE_PUP 180solutions (Zango) Spyware | Adware/PUP |
| ET INFO Pastebin-like Service Domain in DNS | C2 Exfiltration Indicator |
| ET INFO URL Shortener Domain in DNS Lookup | Policy Violation |
| ET GAMES Nintendo Wii / Second Life User-Agent | Policy Violation |

> **103 pages of alerts generated** from a single tmNIDS run.

📸 Screenshots: `tmnids-wazuh-alerts.png`, `tmnids-wazuh-alerts-2.png`

---

### Scenario 2 — SQL Injection with sqlmap

**Run from Kali against DVWA:**
```bash
sqlmap -u "http://192.168.0.203/dvwa/vulnerabilities/sqli/?id=1&Submit=Submit" \
  --cookie="PHPSESSID=<session_id>;security=low" \
  --dbs --batch
```

**sqlmap result:**
```
back-end DBMS: MySQL >= 5.0.12 (MariaDB fork)

available databases [2]:
[*] dvwa
[*] information_schema

Techniques used:
- Time-based blind (SLEEP)
- UNION query (2 columns)
```

**Wazuh detections:**

| Alert | Rule ID | Level | MITRE ATT&CK |
|---|---|---|---|
| A web attack returned code 200 (success) | 31106 | 6 | T1190 — Initial Access |
| Web server 500 error code (Internal Error) | 31122 | 5 | — |

> **85 pages of T1190 Initial Access alerts** triggered by the SQLi attack.

📸 Screenshot: `sqli-t1190-alerts.png`

---

## 🗺️ MITRE ATT&CK Coverage

| Technique | ID | Triggered By |
|---|---|---|
| Exploit Public-Facing Application | T1190 | sqlmap SQLi against DVWA |
| Command and Control | T1071 | Malware beacon traffic via tmNIDS |

---

## 🧩 Challenges & Troubleshooting

| Problem | Root Cause | Fix |
|---|---|---|
| Suricata only 3 rules loaded | Rule path mismatch after `suricata-update` | Changed `default-rule-path` to `/var/lib/suricata/rules` |
| Wazuh Agent won't connect | Agent version newer than server v4.7.5 | Pinned with `apt install wazuh-agent=4.7.5-1` |
| sqlmap redirected to login | PHPSESSID cookie expired | Re-logged into DVWA for fresh session cookie |
| Suricata not seeing Kali traffic | Suricata only sniffs its own NIC | Installed Suricata directly on victim machine |
| Windows Suricata permission errors | UAC blocking writes to Program Files | Switched to Ubuntu — far simpler |
| Wazuh GPG import fails | `curl` not installed on fresh Ubuntu | `sudo apt install curl -y` first |

---

## 📁 Repository Structure

```
suricata-wazuh-dvwa-lab/
├── README.md
└── screenshots/
    ├── tmnids-wazuh-alerts.png       # ET MALWARE & ADWARE detections
    ├── tmnids-wazuh-alerts-2.png     # ET MALWARE Winnti & C2 detections
    └── sqli-t1190-alerts.png         # T1190 Initial Access SQLi alerts
```

---

## 💡 Key Takeaways

- **Suricata + Emerging Threats rules** (49,300+ signatures) provides broad coverage across malware, C2 beacons, adware, policy violations, and web attacks out of the box.
- **Wazuh automatically maps detections to MITRE ATT&CK**, giving SOC analysts immediate adversary context without manual correlation.
- **eve.json is the bridge** — Suricata's structured JSON output makes integration with any SIEM straightforward and reliable.
- **Version pinning matters** — Wazuh agent and server must match; mismatches silently fail to connect.
- A **bare-metal home lab** can replicate enterprise-grade SOC visibility at zero cost using open-source tooling.

---

## 🔭 Future Work

- [ ] Brute force detection using Hydra against DVWA login
- [ ] Wazuh active response to auto-block attacker IP on detection
- [ ] XSS and Command Injection attack scenarios
- [ ] Port mirroring via managed switch for proper network-tap architecture
- [ ] Custom Wazuh rules for DVWA-specific attack patterns

---
