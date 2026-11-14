# Installation Guide

This guide builds the lab described in the [main README](../README.md) from scratch. It covers four virtual machines on VMware Workstation's host-only network:

| VM | Address | Purpose |
|---|---|---|
| Wazuh-Server | 192.168.56.10 | SIEM manager, indexer, dashboard |
| OpenCanary | 192.168.56.20 | Honeypot with a Wazuh agent |
| Win11-Endpoint | 192.168.56.40 | Windows endpoint with a Wazuh agent |
| Kali | 192.168.56.130 | Attacker (optional until Stage 5) |

Your Windows host is `192.168.56.1` on the same network.

## Contents

1. [Before you start](#1-before-you-start)
2. [Network: host-only adapter](#2-network-host-only-adapter)
3. [Wazuh server](#3-wazuh-server)
4. [OpenCanary honeypot](#4-opencanary-honeypot)
5. [Wazuh agent on OpenCanary](#5-wazuh-agent-on-opencanary)
6. [Detection rules](#6-detection-rules)
7. [Windows 11 endpoint](#7-windows-11-endpoint)
8. [Kali attacker](#8-kali-attacker)
9. [Verification checklist](#9-verification-checklist)
10. [Optional: archive logging](#10-optional-archive-logging)
11. [Shutting down](#11-shutting-down)

---

## 1. Before you start

**Host requirements:** Windows 11, 16 GB RAM, 4 or more cores, 100 to 250 GB free disk, VMware Workstation 17 or later.

**Files to download:**
- `wazuh-4.14.10.ova` from documentation.wazuh.com (Deployment, OVA)
- Ubuntu Server 24.04 LTS ISO from ubuntu.com
- Windows 11 multi-edition x64 ISO from microsoft.com (Software Download)
- The Wazuh Windows agent MSI `wazuh-agent-4.14.10-1.msi` from the Wazuh Windows agent page
- A Kali Linux VM image from kali.org

**Verify the Wazuh OVA** before importing. In PowerShell in the Downloads folder:

```powershell
Get-ChildItem .\wazuh-*.ova | Select-Object Name, Length
(Get-FileHash .\wazuh-*.ova -Algorithm SHA512).Hash
```

Compare the hash with the SHA512 published on the Wazuh OVA page. They must match exactly. Delete the file and download it again if they don't.

**Memory budget:** Wazuh 8 GB, OpenCanary 2 GB, Win11-Endpoint 4 GB, Kali 2 GB while attacking. Run only what the current step needs, and close heavy host applications while the lab runs.

---

## 2. Network: host-only adapter

1. In VMware, go to **Edit → Virtual Network Editor → Change Settings**.
2. Select **VMnet1 (Host-only)**.
3. Set **Subnet IP** to `192.168.56.0` and **Subnet mask** to `255.255.255.0`.
4. Keep **Use local DHCP service** enabled, click **Apply**, then **OK**.

Every lab VM uses **Network Adapter → Host-only** in its settings, except during the short internet phases described below.

**Internet phases.** Host-only has no route to the internet, so package downloads need NAT temporarily. The pattern is the same each time:

1. Shut the VM down.
2. Set **Network Adapter** to **NAT**, power on, and install.
3. Shut down, set the adapter back to **Host-only**, and power on.
4. Restore the static address for that VM.

Never leave the honeypot on NAT. It must stay isolated.

---

## 3. Wazuh server

### 3.1 Import the OVA

1. In VMware, **File → Open**, select `wazuh-4.14.10.ova`.
2. Name the VM `Wazuh-Server`, choose a storage path with at least 60 GB free, and click **Import**.

### 3.2 Hardware

1. Open **Edit virtual machine settings**.
2. Set **Memory** to 8192 MB and **Processors** to 2 (1 processor, 2 cores).
3. Set **Network Adapter** to **Host-only**.

Four vCPUs produced a "CPU has been disabled by the guest operating system" fault on this host. If that message appears, reduce the CPU count and use **Upgrade this virtual machine** to refresh the hardware version.

### 3.3 First login and address

1. Power on and wait 3 to 5 minutes for the services to start.
2. Log in on the console with the account shown on the login banner. Change its password with `passwd`.
3. Check the address:

   ```bash
   ip -4 addr show
   ```

### 3.4 Static address

The OVA uses systemd-networkd. Edit `/etc/systemd/network/20-eth0.network`:

```ini
[Match]
Name=eth0

[Network]
Address=192.168.56.10/24
```

Apply it:

```bash
sudo systemctl restart systemd-networkd
ip -4 addr show eth0
```

`eth0` must show `192.168.56.10/24`.

### 3.5 Check the services

```bash
sudo /var/ossec/bin/wazuh-control status
sudo systemctl status wazuh-indexer wazuh-dashboard --no-pager
```

The core daemons (`analysisd`, `remoted`, `db`, `authd`, `apid`, `modulesd`, `monitord`, `logcollector`, `syscheckd`, `execd`) should show `is running...`. If `wazuh-manager` shows `failed` in systemd while its daemons run, restart it:

```bash
sudo systemctl restart wazuh-manager
```

### 3.6 Dashboard

From the Windows host, open `https://192.168.56.10` and accept the self-signed certificate warning. Log in with the admin account from the Wazuh documentation.

### 3.7 Set the clock to UTC

Both Linux VMs run on a clock that doesn't sync on this network, so set it manually. Get the UTC time on Windows:

```powershell
(Get-Date).ToUniversalTime().ToString("yyyy-MM-dd HH:mm:ss")
```

Then on the Wazuh server:

```bash
sudo timedatectl set-ntp no
sudo timedatectl set-time "YYYY-MM-DD HH:MM:SS"
timedatectl
```

Repeat this after every reboot.

---

## 4. OpenCanary honeypot

### 4.1 Create the VM

1. In VMware, **File → New Virtual Machine → Custom**, select the Ubuntu Server ISO, and set the guest OS to Ubuntu 64-bit.
2. Name it `OpenCanary`.
3. Use **1 processor**, **2048 MB** memory, and a **20 GB** disk.
4. Set **Network Adapter** to **NAT** for the install, then to **Host-only** afterward.
5. During the Ubuntu installer, enable **Install OpenSSH server** and create your user account.

### 4.2 Static address

Ubuntu uses netplan. Edit `/etc/netplan/50-cloud-init.yaml`:

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: no
      addresses:
        - 192.168.56.20/24
```

Use spaces, not tabs. Stop cloud-init from rewriting the file, then apply:

```bash
sudo bash -c 'echo "network: {config: disabled}" > /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg'
sudo netplan apply
ip -4 addr show ens33
ping -c 3 192.168.56.10
```

The interface may be named something other than `ens33`. Check with `ip a` and match the name in the file.

### 4.3 Temporary internet for packages

Switch the adapter to **NAT**, power on, then change the netplan file to add DHCP for the install:

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: yes
      addresses:
        - 192.168.56.20/24
```

```bash
sudo netplan apply
ping -c 3 archive.ubuntu.com
```

If the ping succeeds, continue. When the installs finish, shut down, set the adapter back to **Host-only**, restore the static-only netplan from section 4.2, and power on.

### 4.4 Install OpenCanary

Run these one at a time, typing them rather than pasting:

```bash
sudo apt update
sudo apt install -y python3-dev python3-venv libssl-dev libpcap-dev libffi-dev build-essential
python3 -m venv ~/opencanary-venv
source ~/opencanary-venv/bin/activate
pip install opencanary
opencanaryd --copyconfig
deactivate
```

If `apt` returns `404` errors, run `sudo apt update` again and retry. The default config is written to `/etc/opencanaryd/opencanary.conf`.

### 4.5 Configure the services

```bash
sudo nano /etc/opencanaryd/opencanary.conf
```

Set these values, keeping the JSON commas and quotes intact:

- `"ssh.enabled"`: `true`, and `"ssh.port"`: `2222`. The real OpenSSH server stays on port 22.
- `"ftp.enabled"`: `true`, with the default port 21.
- `"http.enabled"`: `true`, with the default port 80.
- Every other service stays `false`.
- The logger writes to `/var/tmp/opencanary.log`.

Save with **Ctrl+O**, Enter, then **Ctrl+X**. Validate the JSON:

```bash
python3 -m json.tool /etc/opencanaryd/opencanary.conf > /dev/null && echo "JSON OK"
```

### 4.6 Start the honeypot

```bash
sudo ~/opencanary-venv/bin/opencanaryd --start
sudo ss -tlnp | grep -E ':(21|80|2222) '
```

Ports 21, 80, and 2222 should appear, owned by `twistd`. If the venv path differs, find it with `ls -d ~/opencanary-*` and use that name.

---

## 5. Wazuh agent on OpenCanary

### 5.1 Install the agent

Switch to NAT for the repository download, then:

```bash
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --dearmor -o /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
sudo apt update
apt-cache madison wazuh-agent | grep 4.14
sudo WAZUH_MANAGER="192.168.56.10" WAZUH_AGENT_NAME="opencanary" apt install -y wazuh-agent=4.14.10-1
```

### 5.2 Point the agent at the honeypot log

Open the agent configuration:

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add this block just before `</ossec_config>`:

```xml
  <localfile>
    <log_format>json</log_format>
    <location>/var/tmp/opencanary.log</location>
  </localfile>
```

Confirm it saved:

```bash
sudo grep -A2 "opencanary.log" /var/ossec/etc/ossec.conf
```

### 5.3 Start the agent

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now wazuh-agent
sudo systemctl status wazuh-agent --no-pager
```

The status must read `active (running)`. Then switch OpenCanary back to **Host-only** and restore the static netplan.

### 5.4 Confirm enrollment

On the Wazuh server:

```bash
sudo /var/ossec/bin/agent_control -l
```

The `opencanary` agent should appear as `Active`.

---

## 6. Detection rules

On the Wazuh server, edit `/var/ossec/etc/rules/local_rules.xml` and enter this content:

```xml
<group name="opencanary,honeypot,">
  <rule id="100100" level="3">
    <decoded_as>json</decoded_as>
    <field name="node_id">opencanary</field>
    <description>OpenCanary honeypot event.</description>
  </rule>

  <rule id="100101" level="6">
    <if_sid>100100</if_sid>
    <field name="logtype">4000</field>
    <description>OpenCanary: connection to a honeypot service.</description>
  </rule>

  <rule id="100103" level="8">
    <if_sid>100100</if_sid>
    <field name="logtype">3001</field>
    <description>OpenCanary: login attempt against a honeypot service.</description>
    <mitre>
      <id>T1110</id>
    </mitre>
  </rule>

  <rule id="100102" level="10" frequency="5" timeframe="120">
    <if_matched_sid>100100</if_matched_sid>
    <same_field>src_host</same_field>
    <description>OpenCanary: 5 or more honeypot events from one source in 2 minutes, possible brute force or scanning.</description>
    <mitre>
      <id>T1110</id>
    </mitre>
  </rule>
</group>
```

Field names are bare (`node_id`, `logtype`, `src_host`), because that is how the decoder produces them. The `data.` prefix appears only in stored alerts.

Validate and apply:

```bash
sudo /var/ossec/bin/wazuh-analysisd -t
sudo systemctl restart wazuh-manager
sudo /var/ossec/bin/wazuh-control status
```

The validation step should print only an INFO line about the rule tree threshold. Any `ERROR` or `CRITICAL` line names the file and line to fix.

---

## 7. Windows 11 endpoint

### 7.1 Create the VM

1. In VMware, **File → New Virtual Machine → Custom**, select the Windows 11 ISO, and let VMware detect Windows 11 x64.
2. Name it `Win11-Endpoint`.
3. Use **2 processors**, **4096 MB** memory, and a **60 GB** disk stored as a single file.
4. Set **Network Adapter** to **NAT** for setup, then **Host-only** afterward.
5. Under **Customize Hardware**, confirm the firmware is **UEFI** and a **Trusted Platform Module** device is present. Windows 11 requires TPM 2.0.

### 7.2 Windows setup

If setup asks you to connect to a network and shows "Something went wrong" with an error code, the virtual adapter has no working driver. Because the VM is on host-only, skip the network step:

1. Press **Shift + F10** to open Command Prompt.
2. Type `OOBE\BYPASSNRO` with the letter O, not zero, and press Enter.
3. The VM reboots. Choose **I don't have internet**, then **Continue with limited setup**.
4. Create a local account.

If that command isn't recognised, try `start ms-cxh:localonly`.

### 7.3 Tools and address

1. Install **VMware Tools** from the VM menu, then restart the VM.
2. Set a static address under **Settings → Network & internet → Ethernet → IP assignment → Edit**:
   - IP assignment: **Manual**
   - IPv4: `192.168.56.40`
   - Prefix length: `24`
   - Gateway and DNS: blank
3. Check the manager is reachable:

   ```powershell
   ping 192.168.56.10
   ```

### 7.4 Get the agent installer

The VM needs internet to download the agent. Two routes work:

**Route A: temporary NAT (used for this lab).** Shut down the VM, set **Network Adapter** to **NAT**, power on, and open **Administrator PowerShell**:

```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.10-1.msi -OutFile "$env:TEMP\wazuh-agent.msi"
```

Then shut down, set the adapter back to **Host-only**, and power on.

**Route B: download on the host.** Download the MSI on your Windows host into Downloads, then drag it onto the VM window. Inside the VM it appears in `Downloads`, and you install from there.

### 7.5 Install the agent

In **Administrator PowerShell** inside the VM, install from the path where the file is:

```powershell
msiexec.exe /i "$env:TEMP\wazuh-agent.msi" /q WAZUH_MANAGER="192.168.56.10" WAZUH_REGISTRATION_SERVER="192.168.56.10" WAZUH_AGENT_NAME="win11-endpoint"
NET START WazuhSvc
Get-Service WazuhSvc
```

`Get-Service` must show `Running`. If the service is `Stopped`, start it with `NET START WazuhSvc` and check the log:

```powershell
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" -Tail 20
```

Confirm the manager address was recorded:

```powershell
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.conf" | Select-String "address"
```

### 7.6 Enable logon auditing

Windows records logon events only when the audit policy is on. Run these in Administrator PowerShell:

```powershell
auditpol /set /subcategory:"Logon" /success:enable /failure:enable
auditpol /set /subcategory:"Credential Validation" /success:enable /failure:enable
auditpol /set /subcategory:"Account Lockout" /success:enable /failure:enable
auditpol /get /subcategory:"Logon"
```

`auditpol /get` must show `Success and Failure`. These commands don't need a network, so you can run them at any point.

### 7.7 Confirm enrollment

Restart the service so it reconnects on the host-only network:

```powershell
Restart-Service WazuhSvc
```

On the Wazuh server:

```bash
sudo /var/ossec/bin/agent_control -l
```

`win11-endpoint` should appear as `Active` with address `192.168.56.40`. If it shows a `192.168.176.x` address, the VM is still on NAT.

---

## 8. Kali attacker

Kali is only needed for Stage 5. Build it the same way as the other VMs, then:

1. Set **Network Adapter** to **Host-only**. Keep NAT off during attacks.
2. Confirm the address:

   ```bash
   ip -4 addr show
   ```

   It must be on `192.168.56.x`. The lab used `192.168.56.130`.

3. Confirm the honeypot is reachable:

   ```bash
   curl -I http://192.168.56.20
   ```

4. Run the Nmap scans only against the honeypot:

   ```bash
   nmap -sT -p 21,80,2222 192.168.56.20
   nmap -sT -p 1-1000 192.168.56.20
   ```

Do not scan the Wazuh server, the Windows host, or any address outside `192.168.56.0/24`.

For browser login tests, use Firefox or Burp Suite to submit the form at `http://192.168.56.20/index.html`. Use made-up credentials.

---

## 9. Verification checklist

Work through this list after the build. Each check should pass before you move on.

| Check | Where | Command or action | Expected |
|---|---|---|---|
| Wazuh address | Wazuh | `ip -4 addr show eth0` | `192.168.56.10/24` |
| Wazuh daemons | Wazuh | `sudo /var/ossec/bin/wazuh-control status` | Core daemons `is running...` |
| Dashboard | Windows host | Open `https://192.168.56.10` | Login page loads |
| Honeypot address | OpenCanary | `ip -4 addr show ens33` | `192.168.56.20/24`, no NAT address |
| Honeypot ports | OpenCanary | `sudo ss -tlnp` | 21, 80, 2222 owned by `twistd` |
| OpenCanary agent | Wazuh | `sudo /var/ossec/bin/agent_control -l` | `opencanary` is `Active` |
| Windows agent | Wazuh | `sudo /var/ossec/bin/agent_control -l` | `win11-endpoint` is `Active` at `192.168.56.40` |
| Windows service | Windows | `Get-Service WazuhSvc` | `Running` |
| Logon auditing | Windows | `auditpol /get /subcategory:"Logon"` | `Success and Failure` |
| Rules load | Wazuh | `sudo /var/ossec/bin/wazuh-analysisd -t` | No `ERROR` or `CRITICAL` lines |
| End-to-end | Windows host | `Test-NetConnection 192.168.56.20 -Port 2222` | Level 6 alert `100101` in `wazuh-alerts-*` |

To run the end-to-end test, send six connections to the honeypot from Windows PowerShell:

```powershell
1..6 | ForEach-Object { Test-NetConnection 192.168.56.20 -Port 2222 -InformationLevel Quiet }
```

Then search `wazuh-alerts-*` in Discover for `rule.id: 100101 or rule.id: 100102`.

---

## 10. Optional: archive logging

The alert index stores only events that match a rule. To keep every raw event for review, enable archive logging on the Wazuh server:

1. Add `<logall_json>yes</logall_json>` inside `<global>` in `/var/ossec/etc/ossec.conf`.
2. In `/etc/filebeat/filebeat.yml`, set `archives: enabled: true` under the `wazuh` module.
3. Restart the services:

   ```bash
   sudo systemctl restart wazuh-manager
   sudo systemctl restart filebeat
   ```

Archives appear in `wazuh-archives-*`, which you can create as an index pattern under **Dashboard management → Index patterns**, with `timestamp` as the time field.

Archive logging uses significant disk space. Turn it off when you no longer need raw events.

---

## 11. Shutting down

Shut the VMs down in this order, so the agents disconnect cleanly:

1. Win11-Endpoint: shut down from Windows.
2. OpenCanary: `sudo shutdown -h now`
3. Wazuh-Server: `sudo shutdown -h now`

Clocks reset on every boot. Repeat section 3.7 after each start, and check the verification list before running tests again.
