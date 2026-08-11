# Wazuh Deployment — RANACORP Homelab

## 1. Overview

Wazuh serves as the SIEM/log aggregation platform for the lab, deployed on
the **BLUETEAM (VLAN20)** segment (Kali Purple host). Windows endpoints in
**RANACORP (VLAN10)** — the domain-joined workstations and the domain
controller — run the Wazuh agent and forward logs to this central manager,
mirroring how an MSSP/SOC would monitor a client network from a separate
management segment.

| Component | Value |
|---|---|
| Wazuh Manager host | Kali Purple (VLAN20 — BLUETEAM) |
| Manager IP | `10.0.2.2` |
| Dashboard URL | `https://10.0.2.2` |
| Agents deployed | WIN10-1, WIN11-1, DC-1 |

---

## 2. Manager Installation (Kali Purple)

Performed on the Kali Purple VM in VLAN20.

```bash
sudo apt update
sudo apt-get install curl -y
```

Then I run the official Wazuh install script (single-command all-in-one
installer, pulled from the [Wazuh install docs](https://documentation.wazuh.com/current/quickstart.html) at time of deployment).

On completion, the installer prints the **dashboard admin credentials**
(username `admin` + generated password) — *(must save these immediately, they
are not shown again by default)*

Dashboard is reachable at:

```
https://<wazuh-manager-ip>
```

The browser will warn about an untrusted certificate since Wazuh installs
with a self-signed cert by default — accept/continue past this for lab use.

---

## 3. Agent Deployment — Windows Workstations

Repeated for each domain-joined Windows endpoint (WIN10-1, WIN11-1).

1. In the Wazuh dashboard: **Agents → Add agent**
2. Select OS: **Windows**
3. Enter the **Wazuh server address** (manager IP, e.g. `10.0.2.2`)
4. Set an agent name that identifies the host (e.g. `win10-1`, `win11-1`)
   — **agent names are case-sensitive**, keep naming consistent across
   all endpoints
5. Assign to the default group (or a custom group if configured)
6. Copy the generated PowerShell install command

On the target Windows machine:

```powershell
# Run PowerShell as Administrator, paste the generated install command
# (downloads and installs the Wazuh agent MSI)

net start wazuhsvc
```

Back in the dashboard, the new agent shows as **Never connected** /
**Pending** until it completes its first callback to the manager —
typically within a minute or two. Refreshed until status showed **Active**.

---

## 4. Agent Deployment — Domain Controller (Windows Server)

Same general flow, with one difference: Windows Server environments
typically restrict direct PowerShell web requests, so the installer
package must be downloaded first rather than run inline.

1. **Agents → Add agent** → OS: **Windows** → version: **Windows Server**
2. Server address: manager IP (e.g. `10.0.2.2`)
3. Agent name: identify as the DC (e.g. `ranacorp-dc`)
4. Copy the **download link** (not the inline script) shown for Server
5. On the DC, open a browser and download the installer package to
   `Downloads` prior running the generated PowerShell install command.
6. From an elevated PowerShell/cmd prompt:

```powershell
cd $env:USERPROFILE\Downloads
dir   # confirm the installer package is present
# run the PowerShell install command (the inline script)

net start wazuhsvc
```

7. Confirm the agent transitions from **Pending → Active** in the
   dashboard.

---

## 5. Verification

- **Agents → Overview** in the Wazuh dashboard should show all deployed
  endpoints as **Active**
- Confirm each agent's IP matches its expected VLAN10 address
- At this stage, agents are enrolled but are only shipping default
  Windows Event Log data — Sysmon log forwarding is configured separately
  (see `sysmon-deployment.md`)

  
### Wazuh Dashboard (Agents List)
 
![Wazuh Dashboard (Agents List)](../Screenshots/Wazuh-dashboard.png)

---

## 6. Notes / Lessons Learned

- Save the dashboard admin password immediately after install — it's only
  displayed once
- Keep agent naming consistent (case-sensitive) across all endpoints to
  avoid confusion when correlating alerts back to a host
- Server OS agents require the download-then-run method instead of the
  inline PowerShell one-liner used for workstations
- Callback delay (Pending → Active) is normal and typically resolves
  within 1-2 minutes — no action needed, just refresh
- May need to check the Firewall rules on agents if there's connectivity issue.  

---

## 7. Next Steps

- [ ] Configure Sysmon on Windows endpoints and forward Sysmon channel
      logs to Wazuh (see `sysmon-deployment.md`)
- [ ] Tune/verify default Wazuh ruleset is generating expected alerts
- [ ] Document custom rules in `Detections/`
