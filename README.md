# Wazuh SIEM Home Lab

A self-hosted Wazuh SIEM environment built from scratch on four virtual machines, with simulated attacks against a deliberately vulnerable web app, custom detection engineering, and a Cowrie SSH honeypot for capturing attacker behavior directly.

Full write-up (architecture, every issue hit and how it was diagnosed, all detection rules, and full screenshots) is in [`docs/Wazuh_SIEM_Lab_Report.pdf`](docs/Wazuh_SIEM_Lab_Report.pdf).

## Architecture

![Network topology](diagrams/network-topology.svg)

| VM | Role | IP |
|---|---|---|
| Wazuh Manager | Indexer + Manager + Dashboard (SIEM core) | 192.168.64.10 |
| Kali Linux | Attacker host | 192.168.64.7 |
| Victim | Target running DVWA (Apache/MariaDB) | 192.168.64.12 |
| Honeypot | Cowrie SSH deception host | 192.168.64.14 |

## Attacks simulated

| Attack | Target | Detection | MITRE ATT&CK |
|---|---|---|---|
| Brute-force login | DVWA custom login form | Custom rule (no built-in coverage existed) | T1110 — Brute Force |
| SQL injection (UNION-based) | DVWA SQLi module | Built-in Wazuh ruleset, zero config | T1190 — Exploit Public-Facing Application |
| SSH login + post-exploitation commands | Cowrie honeypot | Custom rule set built for Cowrie's JSON log | T1078, T1059, T1595 |
| Malware download capture | Cowrie honeypot | Custom rule, file hashed automatically by Cowrie | T1105 — Ingress Tool Transfer |

## Key challenges solved

- **Multi-service credential sprawl** — the indexer, manager, dashboard, and Filebeat each maintain independent credential stores; diagnosing which layer was actually failing required checking each one directly via its own API rather than trusting symptoms from the browser.
- **Silent log-collection failure** — the Wazuh agent reported "Analyzing file" for Apache's access log with no error, while actually being blocked by a file permission issue; the fix was invisible unless the full alert pipeline was tested end to end, not just the collector's own log.
- **Rule-tree evaluation gap** — a custom detection rule never fired despite correct match logic, because it had no `<if_sid>` parent; Wazuh only evaluates the child tree of whichever root rule already claimed an event. Root-caused with `wazuh-logtest -v`.
- **VM cloning pitfalls** — cloning the Victim VM to create the honeypot silently duplicated its MAC address (causing an IP collision) and its Wazuh agent identity (`client.keys`), requiring both to be regenerated before the new VM could be trusted as a distinct host.
- **Systemd socket activation overriding sshd_config** — moving SSH off port 22 to make room for Cowrie required a `ssh.socket` override, not just an `sshd_config` edit, since Ubuntu's socket activation controls the actual bind independent of the daemon's own config.
- **Building detection for a brand-new log source** — Cowrie's JSON events had no existing Wazuh rule coverage; real field names were pulled directly from raw log lines rather than guessed, and a base/child rule tree was validated with `wazuh-logtest` before being deployed live.

## Skills demonstrated

- SIEM deployment and administration (Wazuh indexer, manager, dashboard, agents)
- Linux system administration (systemd, LVM/disk management, service hardening, user/permission management)
- Custom detection rule authoring and validation (Wazuh rule XML, `wazuh-logtest`)
- MITRE ATT&CK mapping
- Web application attack simulation (brute force, SQL injection) with Hydra/curl and manual exploitation
- Honeypot deployment and attacker behavior capture (Cowrie)
- Root-cause troubleshooting across a multi-service, multi-VM stack

## Screenshots

| | |
|---|---|
| ![Dashboard overview](screenshots/dashboard-overview.png) Wazuh Threat Hunting dashboard | ![Brute-force detection](screenshots/dvwa-brute-force-detection.png) Custom rule catching DVWA brute force |
| ![SQLi validation](screenshots/sqli-logtest-validation.png) `wazuh-logtest` validating built-in SQLi rule | ![SQLi detection](screenshots/dvwa-sqli-detection.png) SQL injection alerts in the dashboard |
| ![Honeypot session](screenshots/honeypot-attacker-session.png) Attacker session inside the Cowrie fake shell | ![Honeypot detection chain](screenshots/honeypot-detection-chain.png) Full connect → login → command alert chain |

## Repo contents

```
docs/         Full report (PDF + Word), including every issue hit and fixed
rules/        Custom Wazuh detection rules (local_rules.xml)
diagrams/     Network topology diagram
screenshots/  Highlight screenshots (full set is in the report)
```

## Notes

This lab was built entirely on local virtual machines (UTM on macOS) with no cloud resources or public exposure. All IPs shown are private (192.168.64.0/24) and only reachable within the host machine's local network.
