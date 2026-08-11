# [MITRE ID] - [Short Detection Title]

> Example filename: `T1003.001-lsass-credential-dumping.md`

| Field | Value |
|---|---|
| **Date** | YYYY-MM-DD |
| **Analyst** | Your Name |
| **MITRE ATT&CK** | Txxxx.xxx - Technique Name |
| **Tactic** | e.g. Credential Access |
| **Severity** | Low / Medium / High / Critical |
| **Status** | Detected / Missed / Tuned |

---

## 1. Scenario

Describe the red team action in plain terms — what was executed, from where,
and against what target.

> Example: From the REDTEAM (VLAN30) Kali host, executed Mimikatz against
> WIN10-1 to dump credentials from LSASS memory, simulating a post-exploitation
> credential harvesting step after initial foothold.

---

## 2. Attack Execution

Command(s) or tooling used to generate the activity. Redact anything
sensitive (IPs, hostnames tied to your real network).

```bash
# example
mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" exit
```

**Source host:** REDTEAM-KALI (10.0.3.x)
**Target host:** WIN10-1 (10.0.1.x)

---

## 3. Event Sources & Telemetry

| Source | Event ID / Type | What it shows |
|---|---|---|
| Sysmon | Event ID 10 (ProcessAccess) | lsass.exe accessed by mimikatz.exe |
| Sysmon | Event ID 1 (ProcessCreation) | mimikatz.exe execution, parent process |
| Windows Security Log | 4663 (if enabled) | Object access to lsass.exe |

Include the raw or lightly-trimmed log snippet:

```
GrantedAccess: 0x1010
SourceImage: C:\Users\...\mimikatz.exe
TargetImage: C:\Windows\system32\lsass.exe
```

---

## 4. Detection in Wazuh

- **Rule ID(s) fired:** e.g. `100002`
- **Rule description:** (as shown in Wazuh)
- **Alert level:** e.g. 12

**Screenshot:** `screenshots/[filename].png`

If no rule fired (a "missed" detection), note that here instead and explain
what you built afterward (see Section 6).

---

## 5. Analyst Notes

- What makes this alert meaningful vs. noise
- False-positive considerations (e.g. legitimate LSASS access by AV/EDR tools)
- What you'd check next in a real triage (parent process tree, network
  connections from the host, other endpoints showing similar activity)

---

## 6. Detection Engineering / Tuning

If you wrote or modified a Wazuh rule/decoder to catch or improve this
detection, document it here.

```xml
<!-- example custom rule -->
<rule id="100002" level="12">
  <if_sid>61603</if_sid>
  <field name="win.eventdata.targetImage">lsass.exe</field>
  <description>Possible LSASS credential dumping via process access</description>
  <mitre>
    <id>T1003.001</id>
  </mitre>
</rule>
```

---

## 7. Response / Remediation

What you would do next in a real environment (isolate host, kill process,
reset credentials, escalate, etc.) — written the way you'd hand it off to
IR.

---

## 8. References

- MITRE ATT&CK: https://attack.mitre.org/techniques/Txxxx/
- Any tool/blog references used
