# Sysmon Deployment — RANACORP Homelab

## 1. Overview

Sysmon (System Monitor) is installed on RANACORP (VLAN10) Windows endpoints
to generate high-fidelity process, network, and file-system telemetry that
the default Windows Event Log doesn't provide. This is the log source that
Wazuh consumes for meaningful detection — without it, Wazuh alerting is
limited to basic Windows Security events.

| Component | Detail |
|---|---|
| Tool | Sysmon (Sysinternals) |
| Config source | SwiftOnSecurity / Sysmon-Modular (GitHub) |
| Deployed on | WIN10-1, WIN11-1, DC-1 |
| Verified via | Event Viewer → Applications and Services Logs → Microsoft → Windows → Sysmon → Operational |

---

## 2. Downloads

Two files are required:

1. **Sysmon** — from [Microsoft Sysinternals](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) (zipped archive containing
   `Sysmon.exe`, `Sysmon64.exe`, `Sysmon64a.exe`)
2. **A Sysmon configuration file** — from the [Sysmon-Modular project](https://github.com/olafhartong/sysmon-modular) on
   GitHub. Used one of the preconfigured config files (the default
   community config) rather than writing rules from scratch.

The config is plain text — copy its contents into Notepad and save as an
XML file, e.g. `sysmonconfig.xml`. It must be saved with a `.xml`
extension for Sysmon to load it correctly.

For lab convenience both files were kept in the `Downloads` folder of the
target endpoint. *(In a production environment these would be staged in a
proper application directory, not left in a user's Downloads folder —
noted here as a lab-only shortcut.)*

---

## 3. Installation

Extract the Sysmon zip archive first — this produces three executables for
different architectures. **Sysmon64.exe** was used (64-bit).

Open an elevated Command Prompt (**Run as administrator**). Since the
endpoint is domain-joined, this requires signing in with the domain
administrator account created during AD setup.

Navigate to the folder containing both files:

```cmd
cd C:\Users\<domain-user>\Downloads
dir
```

Confirm `sysmon64.exe` and the config XML (e.g. `sysmonconfig.xml`) are
both present, then install:

```cmd
sysmon64.exe -accepteula -i sysmonconfig.xml
```

- `-accepteula` — silently accepts the license agreement (skips the GUI
  prompt)
- `-i <config.xml>` — installs Sysmon as a service using the specified
  configuration file

Output should confirm Sysmon is installed and the service is running.

---

## 4. Verification

1. Open **Event Viewer** as administrator (domain admin credentials
   required)
2. Navigate to:
   `Applications and Services Logs → Microsoft → Windows → Sysmon → Operational`
3. Confirm events are actively being logged (event count increasing on
   refresh)

At this point Sysmon is generating telemetry locally but is **not yet**
being shipped to Wazuh — that requires configuring the Wazuh agent to
monitor the Sysmon Operational log channel (see
`wazuh-sysmon-integration.md`).

---

## 5. Notes / Lessons Learned

- The config file must be `.xml` — pasting the raw config text into
  Notepad and saving without the correct extension will cause the install
  to fail silently or be ignored
- Domain-joined machines require domain admin credentials to run
  elevated installs/Event Viewer — local admin isn't sufficient
- Using a maintained community config (Sysmon-Modular / SwiftOnSecurity)
  instead of a blank/default config is what makes the logs useful —
  out-of-the-box Sysmon with no config logs very little
- Repeat this process per endpoint (WIN10-1, WIN11-1, DC-1) — Sysmon config
  is not centrally pushed in this lab (a GPO-based rollout would be the
  production equivalent, noted as a future improvement)

---

## 6. Next Steps

- [ ] Configure Wazuh agent `ossec.conf` on each endpoint to monitor the
      `Microsoft-Windows-Sysmon/Operational` channel
- [ ] Confirm Sysmon events appear in the Wazuh dashboard
- [ ] Validate decoders/rules are correctly parsing Sysmon event IDs
      (1, 3, 7, 10, 11, 13, etc.)
- [ ] Document sample detections in `Detections/` using Sysmon-sourced
      alerts
