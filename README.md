# Honeypot-Based SOC Monitoring & Detection Lab

A local Security Operations Center (SOC) lab built on VMware Workstation. A Wazuh 4.14.10 SIEM collects telemetry from an OpenCanary honeypot and a Windows 11 endpoint. Custom detection rules turn honeypot activity into correlated alerts, and the lab was tested with scripted attacks, a browser-based login attack, and Nmap scans from a Kali Linux attacker VM.

The lab runs entirely on an isolated VMware host-only network. Nothing is exposed to the internet or the home network.

---

## Index

| Section | What it covers |
|---|---|
| [Overview](#overview) | Goals and what the lab demonstrates |
| [Project Stages](#project-stages) | The nine stages and how they map to the work |
| [Architecture](#architecture) | Virtual machines, addresses, and data flow |
| [Attack Simulation (Stage 5)](#attack-simulation-stage-5) | The attacks run against the lab and what each one produced |
| [Detection Rules (Stage 6)](#detection-rules-stage-6) | The four custom rules and their logic |
| [Investigation (Stage 7)](#investigation-stage-7) | Timeline, findings, and recommended fixes |
| [Dashboards (Stage 8)](#dashboards-stage-8) | Saved searches and how to use them |
| [Validation (Stage 9)](#validation-stage-9) | Test results, false-positive review, and requirement coverage |
| [Requirements](#requirements) | Host specifications and downloads |
| [Installation](#installation) | Full build guide (see `docs/INSTALLATION.md`) |
| [Maintenance After Reboot](#maintenance-after-reboot) | Steps to run each time the lab starts |
| [Troubleshooting](#troubleshooting) | Errors encountered during the build and their fixes |
| [Limitations](#limitations) | Scope boundaries of the lab |
| [Safety](#safety) | Authorised-use statement |
| [Repository Layout](#repository-layout) | File structure of this repository |

---

## Overview

This project builds a working SOC that detects attacks against a honeypot and a monitored Windows endpoint. It demonstrates:

- **Honeypot-based detection.** Fake FTP, HTTP, and SSH services record every interaction, including credentials submitted to a fake login page.
- **Centralised SIEM monitoring.** A Wazuh manager receives events from two agents, decodes them, and stores alerts.
- **Detection engineering.** Four custom rules, including a correlation rule that escalates repeated activity from one source to level 10 and maps it to MITRE ATT&CK T1110 (Brute Force).
- **Endpoint monitoring.** A Windows 11 agent with logon auditing enabled, reporting authentication events.
- **Attack analysis.** A recorded attack run was traced from the first honeypot hit to the correlation alert, with source attribution, timing, and recommended fixes.

---

## Project Stages

1. Environment setup: VMware Workstation, isolated host-only network (VMnet1, 192.168.56.0/24)
2. Wazuh deployment from the official OVA
3. OpenCanary honeypot deployment with FTP, HTTP, and SSH decoy services
4. Endpoint integration: Windows 11 agent and Windows Security auditing
5. Attack simulation: scripted connections, browser login attempts, and Nmap scans
6. Detection engineering: custom Wazuh rules 100100 to 100103
7. SOC investigation: timeline reconstruction and findings
8. Dashboards: saved searches for sources, endpoints, and rule chains
9. Validation: test set, false-positive review, and requirement coverage

---

## Architecture

| Virtual machine | Address | Operating system | Role |
|---|---|---|---|
| Wazuh-Server | 192.168.56.10 | Wazuh OVA (4.14.10) | SIEM manager, indexer, dashboard |
| OpenCanary | 192.168.56.20 | Ubuntu Server 24.04.5 LTS | Honeypot: FTP (21), HTTP (80), SSH decoy (2222), Wazuh agent 001 |
| Win11-Endpoint | 192.168.56.40 | Windows 11 Pro | Monitored endpoint, Wazuh agent 002 (`win11-endpoint`) |
| Kali | 192.168.56.130 | Kali Linux | Attacker VM: Burp Suite, Nmap |
| Windows host | 192.168.56.1 | Windows 11 | Test-traffic source, VMnet1 host adapter |

### Data flow

```
Attacker (Kali / host)
        │  connections and logins
        ▼
OpenCanary honeypot ──► /var/tmp/opencanary.log (JSON)
        │
        ▼
Wazuh agent 001 ──────► Wazuh manager (192.168.56.10:1514)
                              │
Win11-Endpoint agent 002 ─────┤  Windows Security events
                              ▼
                      Decoder (json, windows)
                              │
                              ▼
                      Rules 100100 → 100101, 100102, 100103
                              │
                              ▼
                      wazuh-alerts-*  (Discover, dashboards)
                      wazuh-archives-* (raw events, optional)
```

All virtual machines sit on VMware's host-only network. The honeypot is never bridged to the home network or exposed to the internet.

---

## Attack Simulation (Stage 5)

The lab was exercised with four kinds of activity. Each one targets the honeypot or the Windows endpoint on the isolated network, and each produced recorded evidence.

### Attack 1: Scripted connection bursts (Windows host)

Repeated connections to the honeypot's fake services from the Windows host, run from PowerShell.

- **Target:** OpenCanary at 192.168.56.20, ports 2222 and 80
- **Method:** six connections to the SSH decoy, then ten connections to the HTTP decoy in rapid succession
- **Source:** 192.168.56.1
- **Result:** each connection produced a level 6 alert from rule 100101. The HTTP burst passed the five-event threshold and produced a level 10 alert from rule 100102, mapped to T1110.

### Attack 2: Browser login attempts (Burp Suite, Kali)

Login submissions to the fake DiskStation page served on port 80 at `/index.html`, sent from Burp Suite.

- **Target:** `http://192.168.56.20/index.html`
- **Method:** POST requests with username and password fields, repeated several times
- **Source:** 192.168.56.130 (Kali)
- **Result:** each submission was logged as `logtype` 3001 with the username, password, user agent, and source address. Rule 100103 alerted on each one at level 8, and repeated submissions contributed to the level 10 correlation.

### Attack 3: Port scan of the decoy services (Nmap, Kali)

A TCP connect scan of the honeypot's three decoy ports.

- **Target:** 192.168.56.20, ports 21, 80, 2222
- **Command:** `nmap -sT -p 21,80,2222 192.168.56.20`
- **Source:** 192.168.56.130
- **Result:** all three ports reported open. Each connection was logged, and rule 100101 alerted per port.

### Attack 4: Full-range scan (Nmap, Kali)

A broader TCP scan to see what an attacker learns beyond the decoys.

- **Target:** 192.168.56.20, ports 1 to 1000
- **Command:** `nmap -sT -p 1-1000 192.168.56.20`
- **Result:** ports 21, 22, 80, and 2222 were open. OpenCanary logged only the three decoy services. Port 22 is the real OpenSSH server, so probes against it produced no honeypot event. This gap is recorded under Limitations.

### Endpoint authentication test (Windows 11)

Deliberate logons on the Windows endpoint, including a failed attempt and a successful one, to test Windows authentication logging.

- **Target:** Win11-Endpoint, agent 002
- **Method:** a wrong password at the lock screen, then a correct one
- **Result:** the Windows Security events were forwarded to the manager under agent `win11-endpoint`, with logon auditing set to Success and Failure for the Logon, Credential Validation, and Account Lockout subcategories.

### Summary of attack activity

| Attack | Tool | Source | Target | Rules triggered | Highest level |
|---|---|---|---|---|---|
| Connection bursts | PowerShell | 192.168.56.1 | 2222, 80 | 100101, 100102 | 10 |
| Login attempts | Burp Suite | 192.168.56.130 | 80 (`/index.html`) | 100103, 100102 | 10 |
| Decoy port scan | Nmap | 192.168.56.130 | 21, 80, 2222 | 100101 | 6 |
| Full-range scan | Nmap | 192.168.56.130 | 1 to 1000 | 100101 (decoys only) | 6 |
| Endpoint logons | Windows | Local | Win11-Endpoint | Windows Security events | Per Windows rules |

---

## Detection Rules (Stage 6)

Rules are stored in `/var/ossec/etc/rules/local_rules.xml` on the Wazuh server.

| Rule ID | Level | Matches | MITRE | Purpose |
|---|---|---|---|---|
| 100100 | 3 | Any event decoded as JSON with `node_id` = `opencanary` | | Base rule that identifies honeypot events |
| 100101 | 6 | Service connection (`logtype` 4000) | | Records each connection to a decoy service |
| 100103 | 8 | Login attempt (`logtype` 3001) | T1110 | Records credentials submitted to a fake login page |
| 100102 | 10 | Five or more honeypot events from one `src_host` in 120 seconds | T1110 | Correlates repeated activity from one source |

### Design decisions

- **Matching uses bare field names.** Rules test `node_id`, `logtype`, and `src_host` as the decoder produces them. The `data.` prefix appears only in stored alerts.
- **Rule 100102 builds on the base rule, not on 100101.** This means it counts both connections and login attempts. A source that mixes the two reaches the threshold, which is the behaviour a brute-force or scanning pattern needs.
- **The base rule is the only one that depends on the JSON decoder.** The other rules inherit that dependency, which keeps unrelated logs from matching.
- **Levels follow the risk.** A connection is informational (6), a credential submission is more serious (8), and a burst from one source is the highest (10).

---

## Investigation (Stage 7)

### Scope

The investigation follows one attack run: a sequence of connections and login attempts from the Kali attacker (192.168.56.130) to the honeypot at 192.168.56.20. The run is traced through the alerts in `wazuh-alerts-*` and the raw events in `wazuh-archives-*`.

Times below are UTC, the clock on both VMs. Discover displays them in IST (UTC+05:30). Alert timestamps trail the honeypot's `local_time` by a few seconds, which is normal ingestion delay.

### Timeline

| # | Time (UTC) | Source | Port | Event | Rule | Level |
|---|---|---|---|---|---|---|
| 1 | 09:38:32 | 192.168.56.130 | 2222 | Connection to the SSH decoy | 100101 | 6 |
| 2 | 09:42:48 | 192.168.56.130 | 2222 | Connection to the SSH decoy | 100101 | 6 |
| 3 | 09:45:40 | 192.168.56.130 | 2222 | Connection to the SSH decoy | 100101 | 6 |
| 4 | 09:50:41 | 192.168.56.130 | 2222 | Connection to the SSH decoy | 100101 | 6 |
| 5 | 09:55:26 | 192.168.56.130 | 2222 | Fifth event within two minutes, threshold crossed | 100102 | 10 |

### Attack pattern

- **Target service:** the fake SSH service on port 2222. The rule fires on the honeypot's own event, not on the real OpenSSH service.
- **Cadence:** five events over about seventeen minutes, in separate bursts. The correlation rule fires on the fifth event because the window is measured per source.
- **Source:** a single address, 192.168.56.130, confirmed as the Kali VM.

### Related activity in the same window

- **Wazuh server (agent 000):** sudo and auditd events from the manager's own administration. These are not part of the honeypot activity.
- **OpenCanary host checks:** the agent's scheduled `df -P` and `last -n 20` checks. These produce alerts unrelated to the attack.

### Findings

1. **Detection works end to end.** A connection to a decoy is decoded, enriched with its source address, stored, and matched by the rule chain. Every event produced a level 6 alert, and the correlation rule fired on the fifth event from the same source.

2. **Source attribution is accurate.** Early Kali traffic reached the honeypot through NAT and was recorded under the Windows host's address, 192.168.56.1. After Kali was moved to the host-only network, its traffic was recorded under 192.168.56.130. Attribution is only reliable when every attacker sits on the isolated network.

3. **Credentials are captured in plain text.** The fake login logs `PASSWORD` values unencrypted. This is what makes credential guessing visible, but it also makes the honeypot log sensitive.

4. **Scans reveal only the decoys.** A full-range scan found four open ports on the honeypot host. OpenCanary logs three of them. Port 22 is the real SSH server and is outside the honeypot's logging.

5. **Clock consistency is required.** Honeypot timestamps, alert timestamps, and the Discover display agree only after both VMs are set to UTC. Before that correction, the Wazuh clock was labelled UTC but showed local time, which would have misordered the timeline.

### Recommended fixes

1. **Replace default credentials.** Change the Wazuh OS and dashboard passwords before any shared use.
2. **Run OpenCanary and the agent without root.** Use `--uid` and `--gid` for `opencanaryd`, and review the agent configuration for the same change.
3. **Protect the honeypot log.** Restrict `/var/tmp/opencanary.log` to owner-only read (`chmod 600`) and rotate it, since it holds captured passwords.
4. **Separate the correlation rule by service.** Score repeated connections to one service separately from repeated logins, so the alert says what kind of activity it saw.
5. **Monitor real SSH separately.** Forward `/var/log/auth.log` from OpenCanary to Wazuh and write a brute-force rule for the real service, which the honeypot cannot see.
6. **Keep the honeypot isolated.** Host-only networking and no bridged adapter stop the decoy from becoming a pivot point into the home network.

---

## Dashboards (Stage 8)

The dashboard stage uses saved Discover searches on `wazuh-alerts-*`. Each search answers one question an analyst would ask.

### Saved search 1: Activity from the Kali attacker

```
agent.name: opencanary and data.src_host: 192.168.56.130
```

Use this to see every honeypot event from the attacker VM and the rule that matched each one. Recommended columns: `rule.id`, `rule.level`, `data.dst_port`, `data.logtype`.

### Saved search 2: Windows endpoint activity

```
agent.name: win11-endpoint
```

Shows every event from the Windows 11 endpoint, including the logon events produced by the auditing configured in the installation guide.

### Saved search 3: Correlation alerts

```
rule.id: 100101 or rule.id: 100102 or rule.id: 100103
```

Shows the full honeypot rule chain in time order. Use it to confirm that a run produced the expected mix of level 6, level 8, and level 10 alerts.

### Saved search 4: Authentication failures

```
agent.name: win11-endpoint and rule.groups: authentication_failed
```

Isolates failed Windows logons for review.

### How the searches fit together

- Start with search 3 to see what fired.
- Use search 1 to attribute the activity to a source.
- Use search 2 for endpoint context, and search 4 to focus on failed authentication.

---

## Validation (Stage 9)

### Test set

| Test | Command or action | Target | Expected result |
|---|---|---|---|
| T1 | Six connections, `Test-NetConnection 192.168.56.20 -Port 2222` | SSH decoy | Rule 100101 on each connection |
| T2 | Ten connections, `Test-NetConnection 192.168.56.20 -Port 80` | HTTP decoy | Rule 100101 each, then 100102 past the threshold |
| T3 | Sweep of ports 21, 22, 80, 2222 | Decoy services | Rule 100101 for 21, 80, 2222; no event for 22 |
| T4 | Login submissions from Burp Suite | `/index.html` | Rule 100103 per submission, 100102 on a burst |
| T5 | `nmap -sT -p 21,80,2222 192.168.56.20` | Decoy services | Rule 100101 per open port |
| T6 | `nmap -sT -p 1-1000 192.168.56.20` | Full range | Decoy events only; no event for port 22 |
| T7 | Failed and successful logon on the endpoint | Win11-Endpoint | Windows Security events under `win11-endpoint` |

### Results

- **T1 and T2:** each connection produced a level 6 alert. The HTTP burst crossed the five-event threshold and produced a level 10 alert mapped to T1110.
- **T3 and T5:** alerts appeared for the decoy ports, with none for port 22 in the sweep, which confirms the decoy logging boundary.
- **T4:** each submission was recorded with username, password, and user agent, and matched rule 100103.
- **T6:** the full-range scan produced events only for the decoy ports, matching the expected result.
- **T7:** the endpoint reported logon events under its own agent name.

### False-positive review

- **Wazuh server self-monitoring.** Sudo and auditd events from `wazuh-server` appear in the alert index. They are routine and are excluded from honeypot views by filtering on `agent.name`.
- **OpenCanary host checks.** The scheduled `df -P` and `last -n 20` checks generate alerts unrelated to the honeypot. Rule-ID filters keep them out of the review views.
- **Deliberate bursts.** Rule 100102 fires on the test bursts, which is the intended behaviour. In a monitored production setting, a legitimate polling tool hitting a decoy often would trigger the same rule, so known monitoring sources should be excluded.

### Requirement coverage

| Requirement | Covered by |
|---|---|
| Centralised security log collection | Wazuh manager with the OpenCanary and Windows agents |
| Honeypot activity monitoring | Rules 100100 to 100103 |
| Windows authentication monitoring | `win11-endpoint` agent with Logon auditing |
| Suspicious authentication detection | Rule 100103 (credential submissions) and 100102 (repeated activity) |
| SSH brute-force detection | Rule 100102 on the SSH decoy |
| Port scanning detection | Rules 100101 and 100102 on the decoy services |
| Custom Wazuh detection rules | Rules 100100 to 100103 |
| Alert investigation | Investigation (Stage 7) |
| Security dashboards | Dashboards (Stage 8) |

---

## Requirements

**Host**
- Windows 11, 16 GB RAM, 4 or more CPU cores
- VMware Workstation 17 or later (tested with 26.0.0)
- 100 to 250 GB free disk space

**Downloads**
- Wazuh OVA `wazuh-4.14.10.ova` from the official Wazuh documentation
- Ubuntu Server 24.04 LTS ISO from ubuntu.com
- Windows 11 multi-edition x64 ISO from microsoft.com
- Kali Linux VM image for the attacker

**Memory budget**

| Virtual machine | RAM | vCPU |
|---|---|---|
| Wazuh-Server | 8 GB | 2 |
| OpenCanary | 2 GB | 1 |
| Win11-Endpoint | 4 GB | 2 |
| Kali (during attacks) | 2 GB | 2 |

The full build, with each step and its commands, is in **[docs/INSTALLATION.md](docs/INSTALLATION.md)**.

---

## Installation

The step-by-step build guide is in **[docs/INSTALLATION.md](docs/INSTALLATION.md)**. It covers:

- Verifying and importing the Wazuh OVA
- Building the OpenCanary, Win11-Endpoint, and Kali virtual machines
- Configuring the host-only network and static addresses
- Installing and enrolling the Wazuh agents
- Writing the detection rules and running the test set

---

## Maintenance After Reboot

The manually set clocks reset on reboot. After each boot, set the clock to the current UTC time on both Linux VMs:

```bash
sudo timedatectl set-ntp no
sudo timedatectl set-time "YYYY-MM-DD HH:MM:SS"
```

Get the UTC time on Windows with:

```powershell
(Get-Date).ToUniversalTime().ToString("yyyy-MM-dd HH:mm:ss")
```

Then check each component:

- **Wazuh:** `sudo /var/ossec/bin/wazuh-control status`
- **Honeypot:** `sudo ss -tlnp | grep -E ':(21|80|2222) '`
- **Agents:** `sudo /var/ossec/bin/agent_control -l`

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| "CPU has been disabled by the guest operating system" | vCPU count or hardware version | Reduce vCPUs to 2 and upgrade the VM hardware |
| `wazuh-manager` shows `failed` while its daemons run | systemd start timed out | Restart the service, confirm with `wazuh-control status` |
| Rules don't match | Field names include the `data.` prefix | Use bare decoded names: `node_id`, `logtype`, `src_host` |
| `wazuh-logtest` exits with a core dump | Tool instability on this build | Validate against live alerts instead |
| Honeypot events missing from archives | Archive logging or Filebeat archive module disabled | Enable both (see the installation guide) |
| Package install returns 404 | Stale package index | Run `sudo apt update` and retry |
| Kali logs show `192.168.56.1` | Kali is on NAT | Set its adapter to Host-only |
| Windows VM can't reach the internet during setup | Host-only has no route | Use NAT temporarily, or `OOBE\BYPASSNRO` during Windows setup |
| Agent is `Active` but shows a NAT address | Endpoint still on NAT | Switch to Host-only and restart `WazuhSvc` |
| Discover times differ between fields | Clock or display timezone mismatch | Set both VMs to UTC and check `dateFormat:tz` in Advanced settings |

---

## Limitations

- **Default Wazuh credentials** are kept for this isolated lab. Change them for any shared or long-running environment.
- **OpenCanary and the Wazuh agent run as root**, which is the default configuration. A dedicated account is the recommended change.
- **Two attacker sources.** The lab models the Windows host and the Kali VM. It does not model multiple independent attackers.
- **Manual clocks.** Both Linux VMs reset their time on reboot, so timestamps are accurate only after the post-boot step runs.
- **Honeypot scope.** The honeypot covers its three decoy services. The real SSH service on port 22 is outside its logging.
- **Linux authentication logs** on the honeypot host are not forwarded, so rules cover the decoy services only.
- **Archive logging** (`logall_json`) increases disk use and is enabled only for raw-event review.
- **Dashboard scope.** Dashboards are saved Discover searches rather than a custom multi-panel dashboard.
- **Single-node deployment.** Wazuh runs as one node with no clustering or redundancy.

---

## Safety

This lab is for authorised, local testing only. The honeypot and every attacker VM stay on the host-only network and are not exposed to the internet, the home network, or any sensitive system. Scans and login tests target only `192.168.56.0/24`. All credentials used in testing are invented for the lab and are not real accounts.

---

## Repository Layout

```
.
├── README.md
├── docs/
│   └── INSTALLATION.md
└── configs/
    ├── local_rules.xml
    ├── opencanary.conf
    ├── 20-eth0.network
    ├── 50-cloud-init.yaml
    └── win11-auditpol.ps1
```

Before committing, remove any password fields from `opencanary.conf` and do not commit default Wazuh credentials or any real account details.
