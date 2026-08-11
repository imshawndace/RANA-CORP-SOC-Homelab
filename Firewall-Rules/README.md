# pfSense Firewall Rules

Sanitized summary of the rule sets configured on pfSense, one section per
interface/VLAN. These mirror what's shown in `../screenshots/`, transcribed
here as text so the policy is readable without opening an image, and so real
addresses stay out of the repo entirely.

Default posture across the lab: **deny by default, explicit allow for what's needed.**

---

## Floating / WAN

| Source | Destination | Port | Action | Description |
|---|---|---|---|---|
| Reserved (bogon) | * | * | Deny | Block bogon networks |
| Home network | Blue Team SIEM (Wazuh UI) | 443 | Allow | Access Wazuh UI |
| Home network | Blue Team DFIR (Velociraptor UI) | 8889 | Allow | Access Velociraptor UI |
| Home network | pfSense WAN address | 443 | Allow | Access pfSense UI |
| * | * | * | Deny | Block all other WAN traffic |

Only the home/admin network can reach management UIs, and only over HTTPS/
the specific management ports above. Everything else inbound on WAN is dropped.

---

## VLAN10 — RANACORP (10.0.1.0/24)

| Source | Destination | Port | Protocol | Action | Description |
|---|---|---|---|---|---|
| * | RANACORP address | 80, 443 | TCP | Allow | Anti-lockout rule (local mgmt access) |
| RANACORP subnet | BLUETEAM subnet | 1515 | TCP | Allow | Wazuh agent → manager (enrollment) |
| RANACORP subnet | BLUETEAM subnet | 8000 | TCP | Allow | Velociraptor agent → manager (enrollment) |
| RANACORP subnet | BLUETEAM subnet | 1514 | TCP/UDP | Allow | Wazuh agent → manager (event log) |
| RANACORP subnet | BLUETEAM subnet | 8889 | TCP | Allow | Velociraptor agent → manager (event log) |
| RANACORP subnet | REDTEAM subnet | * | Any | Deny | Block VLAN10 → Red Team |
| RANACORP subnet | BLUETEAM subnet | * | Any | Deny | Block VLAN10 → Blue Team (catch-all, below the monitoring allows) |
| RANACORP subnet | Home network | * | Any | Deny | Block VLAN10 → HOME |
| RANACORP subnet | * | * | Any | Allow | Allow VLAN10 → internet |

RANACORP endpoints can only reach BLUETEAM on the specific ports their
monitoring agents need (Wazuh, Velociraptor). Everything else toward Blue
Team, Red Team, or the home network is blocked. Internet egress is allowed
so patching/updates behave like a real corporate network.

---

## VLAN20 — BLUETEAM (10.0.2.0/24)

| Source | Destination | Port | Action | Description |
|---|---|---|---|---|
| BLUETEAM subnet | Home network | * | Deny | Block BLUETEAM → home network |
| BLUETEAM subnet | REDTEAM subnet | * | Deny | Block BLUETEAM → Red Team |
| BLUETEAM subnet | * | * | Allow | Allow BLUETEAM → internet |

Blue Team infrastructure (Wazuh, Velociraptor) is outbound-internet-only —
it can't initiate connections into the home network or the Red Team range.
Inbound access to it is governed entirely by the RANACORP rules above and
the WAN management rules.

---

## VLAN30 — REDTEAM (10.0.3.0/24)

| Source | Destination | Port | Action | Description |
|---|---|---|---|---|
| REDTEAM subnet | Home network | * | Deny | Block REDTEAM → home network |
| REDTEAM subnet | BLUETEAM subnet | * | Deny | Block REDTEAM → Blue Team |
| REDTEAM subnet | * | * | Allow | Allow REDTEAM → internet |

Red Team is fully isolated from both the home network and Blue Team
infrastructure. Attacks against RANACORP (VLAN10) are exercised deliberately
and are not covered by a standing allow rule — they're run manually as part
of specific exercises, then observed for detections in Wazuh/Velociraptor.

---

## Why this design

This is a simplified version of a real segmented enterprise network:

- **Least privilege** — each VLAN can only reach what it explicitly needs (e.g. agents can talk to the SIEM on defined ports, nothing else).
- **Monitoring path is one-way in** — RANACORP endpoints push logs/telemetry into BLUETEAM; BLUETEAM never initiates connections back into RANACORP, REDTEAM, or home.
- **Blast radius containment** — REDTEAM and RANACORP can't directly reach BLUETEAM or the home network, so a compromised or misbehaving VM in either segment can't pivot into the monitoring stack or the management network.
- **Deny-by-default** — every VLAN ends with an explicit deny before any broad allow, so nothing is implicitly reachable.
