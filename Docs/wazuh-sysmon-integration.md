# Wazuh–Sysmon Integration — RANACORP Homelab

## 1. Overview

Sysmon generates rich endpoint telemetry locally, but it isn't shipped to
Wazuh automatically — the Wazuh agent has to be told to monitor the Sysmon
Operational event log channel. This doc covers that integration: agent
config, decoder/rule verification, and confirming Sysmon-sourced alerts
reach the dashboard.

Prerequisite: Sysmon installed and logging locally (see
`sysmon-setup.md`), Wazuh agent enrolled and Active (see
`Wazuh-deployment.md`).

| Component | Detail |
|---|---|
| Log channel | `Microsoft-Windows-Sysmon/Operational` |
| Agent config file | `C:\Program Files (x86)\ossec-agent\ossec.conf` |
| Decoders | Built-in Wazuh Sysmon decoders (`windows_sysmon`) |
| Ruleset | Built-in `0800-sysmon_id_*.xml` rule files |

---

## 2. Configure the Agent to Monitor Sysmon

On each Windows endpoint (WIN10-1, WIN11-1, DC-1), edit the agent's
`ossec.conf` and add a `<localfile>` block pointing at the Sysmon channel:

```xml
<ossec_config>
  <localfile>
    <location>Microsoft-Windows-Sysmon/Operational</location>
    <log_format>eventchannel</log_format>
  </localfile>
</ossec_config>
```

- `log_format` must be `eventchannel` (not `eventlog`) — this is the
  format used for the newer Windows Event Log channels, which is what
  Sysmon writes to
- This block can be added via the agent's local `ossec.conf` directly, or
  centrally via a **Wazuh agent group configuration** (`agent.conf`)
  pushed from the manager — the group approach is preferable once
  managing more than a couple of endpoints, since it avoids editing each
  machine individually

Restart the Wazuh agent service after editing:

```powershell
net stop wazuhsvc
net start wazuhsvc
```

---

## 3. Verify Sysmon Events Are Arriving

1. In the Wazuh dashboard, go to **Discover** (or **Security events**)
2. Filter by `agent.name` for the target host
3. Filter/search for `data.win.system.providerName: Microsoft-Windows-Sysmon`
4. Confirm events are populating with Sysmon-specific fields (e.g.
   `data.win.eventdata.image`, `data.win.eventdata.targetImage`)

If no Sysmon events appear:
- Confirm the `<localfile>` block was added correctly and the agent
  service was restarted
- Confirm Sysmon itself is still actively logging locally (Event Viewer
  check from `sysmon-setup.md`)
- Check the agent's `ossec.log` for config parsing errors

---

## 4. Decoders & Ruleset

Wazuh ships with built-in decoding and rules for Sysmon out of the box —
no custom decoder is required for standard Sysmon event IDs. Relevant
built-in rule groups:

| Event ID | Description | Rule group |
|---|---|---|
| 1 | Process creation | `sysmon_process-anomalies`, `sysmon_event1` |
| 3 | Network connection | `sysmon_event3` |
| 7 | Image/DLL load | `sysmon_event7` |
| 10 | Process access (e.g. LSASS access) | `sysmon_event10` |
| 11 | File create | `sysmon_event11` |
| 13 | Registry value set (persistence) | `sysmon_event13` |

Confirm these are firing by checking **Alerts** in the dashboard filtered
to the relevant `rule.groups`. Baseline "noisy" alerts (e.g. routine
process creation) are expected — this is the point at which tuning
becomes relevant (see Section 5).

---

## 5. Tuning

Out-of-the-box Sysmon + Wazuh rules will generate a lot of low-value
alerts, especially for process creation. Two options going forward:

- **Sysmon config tuning** — the community config (Sysmon-Modular /
  SwiftOnSecurity) already filters a lot of noise at the source; further
  exclusions can be added for known-noisy, non-suspicious lab activity
- **Wazuh rule tuning** — write custom rules (higher severity, more
  specific conditions) for the behaviors actually being tested in red
  team exercises, rather than relying only on default rule levels

Any custom rules written should be documented in `Firewall-Rules/` (if
network-related) or alongside the relevant write-up in `Detections/` —
not buried only in `ossec.conf`, so the reasoning is visible in the repo.

---

## 6. Notes / Lessons Learned

- `log_format` is the detail most likely to be gotten wrong —
  `eventchannel` vs `eventlog` looks similar but only one works for
  Sysmon's channel
- Editing `ossec.conf` per-agent works for a small lab but doesn't scale
  — a Wazuh agent group config is the more realistic/production-like
  approach and worth switching to once more endpoints are added
- Confirming telemetry end-to-end (Sysmon logging locally → agent
  shipping → Wazuh decoding → rule firing) is worth checking as four
  separate steps when troubleshooting, since a failure at any one stage
  looks the same from the dashboard ("no alerts")

---

## 7. Next Steps

- [ ] Push Sysmon monitoring config via Wazuh agent group instead of
      per-host `ossec.conf` edits
- [ ] Generate red team activity (Section 4 event IDs) and confirm
      end-to-end detection
- [ ] Write up first Sysmon-sourced detection in `Detections/`
