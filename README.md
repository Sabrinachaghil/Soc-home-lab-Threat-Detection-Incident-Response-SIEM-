# SOC Home Lab — Enterprise Threat Detection & Incident Response

## 🎯 Objective
Build a home SOC lab using **Wazuh** (open-source SIEM) to monitor, detect, and analyze simulated attacks in an isolated environment — demonstrating both detection capability and the real-world impact of monitoring coverage gaps.

> ⚠️ This project was conducted entirely in an **isolated local lab environment** (VMware, host-only network). No real-world or production systems were targeted.

## 🧪 Lab Architecture

| Component | Role | Monitoring |
|---|---|---|
| Ubuntu Server 26.04 | Wazuh Manager + Indexer + Dashboard (SIEM) | N/A (the SIEM itself) |
| Kali Linux | Attacker machine | ✅ Wazuh Agent installed |
| Metasploitable2 | Victim/Target (intentionally vulnerable) | ❌ No agent (legacy OS, unsupported) |

**Why this setup matters:** having one monitored host (Kali) and one unmonitored host (Metasploitable2) makes it possible to directly compare detection outcomes for the same type of attack — a realistic scenario, since real networks rarely have 100% agent coverage.

## 🛠️ Tools Used
- **Wazuh 4.9.2** — SIEM (log collection, analysis, alerting, FIM, MITRE ATT&CK mapping)
- **Kali Linux** — nmap, nikto, sqlmap, OpenSSH
- **Metasploitable2** — intentionally vulnerable target (DVWA, outdated services)

## 📋 Setup Summary

### 1. Wazuh Manager Installation (Ubuntu)
Installed via the official all-in-one installer (`wazuh-install.sh -a -i -o`). Required a 50GB disk (initial 9.8GB was insufficient and caused repeated install failures) and 4GB RAM.

### 2. Wazuh Agent Installation (Kali)
Installed and registered the agent, pointing it to the Manager's IP. Required matching the agent version (4.9.2) to the Manager version — a newer agent (4.14.10) was initially installed by mistake and rejected by the Manager with `Agent version must be lower or equal to manager version`.

## 🔍 Attack Simulations & Detection Results

| # | Attack | Target | Monitored? | Wazuh Alert? | Evidence |
|---|---|---|---|---|---|
| 1 | SSH brute-force (failed logins) | Kali (localhost) | ✅ | ✅ Yes | Rule 2502 (level 10) — *"User missed the password more than one time"* |
| 2 | File Integrity Monitoring test (`touch`/`tee` on `/etc/`) | Kali (local) | ✅ | ✅ Yes | Rule 5402 (level 3) — *"Successful sudo to ROOT executed"*, MITRE T1548.003 |
| 3 | Nmap scan (`-A -T4`) | Metasploitable2 | ❌ | ❌ No | 0 change in alert count |
| 4 | Nikto web scan | Metasploitable2 | ❌ | ❌ No | 0 change in alert count |
| 5 | SQL injection (sqlmap, DB enumeration) | Metasploitable2 (DVWA) | ❌ | ❌ No | 0 change in alert count |
| *(bonus)* | Nmap + Nikto (run incidentally) | Ubuntu/Wazuh server | ✅ (self) | ✅ Yes | Alert count jumped from 24 → 385 total events; MITRE categories: Valid Accounts, Password Guessing, Brute Force, SSH |

### Detection examples (monitored host — Kali)
```
Rule: 2502 (level 10) -> 'syslog: User missed the password more than one time'
sshd-session: PAM 2 more authentication failures; rhost=::1

Rule: 5402 (level 3) -> 'Successful sudo to ROOT executed.'
COMMAND=/usr/bin/touch /etc/wazuh-test-file.txt
```

### Non-detection (unmonitored host — Metasploitable2)
Running the same reconnaissance and exploitation tools (nmap, nikto, sqlmap) against Metasploitable2 produced **zero alerts** and **zero change** in the Wazuh dashboard's event counters, despite nmap and nikto returning rich vulnerability data (outdated Apache/PHP, anonymous FTP, Telnet, HTTP TRACE/XST) and sqlmap successfully extracting 7 database names.

## 🧠 Key Finding: The Monitoring Coverage Gap

This is the central finding of the project:

> **Detection depends entirely on monitoring coverage, not on attack sophistication.**

The exact same tools (nmap, nikto) produced full detections with rich MITRE ATT&CK classification when run against a monitored host (Ubuntu), and zero detections when run against an unmonitored host (Metasploitable2) — even though the attacks against Metasploitable2 were arguably more impactful (successful SQLi data extraction vs. simple reconnaissance).

This mirrors a real-world SOC problem: an attacker who finds an unmonitored asset on the network is effectively invisible to the SIEM, regardless of how "loud" or advanced their techniques are.

### Recommendation (as it would appear in a real IR report)
- Deploy a Wazuh agent (or equivalent HIDS) on every reachable host, prioritizing legacy/EOL systems that are often excluded due to compatibility issues — these are frequently the weakest links.
- Where agent deployment is not possible (e.g., unsupported legacy OS), compensate with network-based monitoring (NIDS, e.g., Suricata) positioned to see traffic to/from the unmonitored host.
- Periodically audit agent coverage against the full asset inventory to catch monitoring gaps before an attacker does.

## 🧩 Challenges & Troubleshooting

Real issues encountered during setup, kept here to show the debugging process rather than just the final working state:

| Issue | Cause | Fix |
|---|---|---|
| VM disk corrupted (`EXT4-fs error`, filesystem forced read-only) | VM files stored inside a OneDrive-synced folder, causing I/O conflicts during disk resize | Rebuilt the VM outside of any cloud-sync folder, on a dedicated `D:\` path |
| `wazuh-manager` installed but `/var/ossec` missing files (`No such file or directory`) | Initial VM disk was only 9.8GB — not enough for the indexer + manager + dashboard stack | Rebuilt the VM with a 50GB disk from the start (Wazuh's recommended minimum) |
| Agent registration failed: `Agent version must be lower or equal to manager version` | `apt install wazuh-agent` pulled the latest version (4.14.10) while the Manager was 4.9.2 | Removed the agent, reinstalled the exact matching `.deb` package (4.9.2) |
| sqlmap/nmap/nikto against Metasploitable2 produced zero Wazuh alerts | No Wazuh agent deployed on Metasploitable2 (unsupported legacy OS) | Documented as the project's key finding rather than "fixed" — see Key Finding section below |

## ⚖️ Disclaimer
This project is for educational purposes only, performed in a personal, isolated lab environment on applications and systems designed to be vulnerable (Metasploitable2, DVWA). Do not run these techniques against systems you do not own or have explicit authorization to test.
