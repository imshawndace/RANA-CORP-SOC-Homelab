# pfSense VLAN Configuration — RANACORP Homelab

## 1. Overview

Segmentation between RANACORP, BLUETEAM, and REDTEAM is implemented using
**Open vSwitch (OVS)** on Proxmox rather than a single 802.1Q trunk
configured inside pfSense. Each VLAN is realized as a separate tagged
port on an OVS bridge; pfSense and each VM's virtual NIC connects to that
bridge on the port matching its VLAN. From pfSense's perspective, each
VLAN simply appears as its own physical-feeling interface (net1, net2,
net3) — no VLAN tagging is configured inside pfSense itself.

| VLAN | Name | Subnet | pfSense interface |
|---|---|---|---|
| 10 | RANACORP | 10.0.1.0/24 | net1 (LAN) |
| 20 | BLUETEAM | 10.0.2.0/24 | net2 (OPT1) |
| 30 | REDTEAM | 10.0.3.0/24 | net3 (OPT2) |
| — | WAN | (redacted) | net0 (unchanged, connected to vmbr0) |

---

## 2. Installing Open vSwitch (Proxmox host)

From the Proxmox node's **Shell** (root, no `sudo` needed):

```bash
apt update
apt install openvswitch-switch
```

`apt update` first to ensure the latest package version is pulled rather
than whatever was current at image-build time.

> Note: OVS is only necessary here because the lab uses **three** VLANs.
> For a two-VLAN setup, Proxmox's built-in Linux bridge would be
> sufficient — OVS was chosen partly for the added hands-on experience
> with a separate virtual switching technology.

---

## 3. Creating the OVS Bridge and Ports

Under **Node → System → Network**:

1. **Create → OVS Bridge**
   - Accepts the default auto-assigned name (e.g. `vmbr1`) — Proxmox's
     existing default Linux bridge is `vmbr0`, so this becomes the next
     available bridge number
2. **Create → OVS IntPort**, once per VLAN:

| Port name | Bridge | VLAN tag |
|---|---|---|
| VLAN10 | vmbr1 | 10 |
| VLAN20 | vmbr1 | 20 |
| VLAN30 | vmbr1 | 30 |

Naming ports to match their VLAN number (rather than arbitrary labels)
keeps the tag, the port name, and the third octet of each VLAN's subnet
all consistent — VLAN 10 → `.1.x`, VLAN 20 → `.2.x`, VLAN 30 → `.3.x`.

**Apply Configuration** after creating the bridge and ports — this step
is easy to skip and causes the bridge to show as inactive later (see
Section 6).

---

## 4. Attaching pfSense to the VLANs

pfSense's virtual NICs are mapped one-to-one with the VLAN ports:

| pfSense NIC | Bridge | VLAN tag | Role |
|---|---|---|---|
| net0 | vmbr0 | — (untouched) | WAN, internet-facing |
| net1 | vmbr1 | 10 | RANACORP / LAN |
| net2 | vmbr1 | 20 | BLUETEAM / OPT1 |
| net3 | vmbr1 | 30 | REDTEAM / OPT2 |

**pfSense must be powered off** to change NIC bridge/VLAN assignments —
attempting this while running fails silently or errors out. Set each NIC
under the pfSense VM's **Hardware** tab, then power pfSense back on.

---

## 5. Attaching VMs to the Correct VLAN

Each VM's single NIC is set to bridge `vmbr1` with the VLAN tag matching
its intended network — VMs must also be **powered off** to change this.

| VM | Bridge | VLAN tag | Network |
|---|---|---|---|
| WIN10-1 / WIN11-1 | vmbr1 | 10 | RANACORP |
| Domain Controller | vmbr1 | 10 | RANACORP |
| Kali (attack platform) | vmbr1 | 30 | REDTEAM |
| Kali Purple | vmbr1 | 20 | BLUETEAM |

### Start order

pfSense must be fully up before any other VM is powered on, since it's
providing DHCP/routing for every other segment. Within RANACORP, the
domain controller should come up before the domain-joined workstations
once Active Directory is in place. After pfSense and the DC, the order
of remaining VMs doesn't matter.

---

## 6. Troubleshooting: Bridge Not Active

If pfSense or a VM fails to get network connectivity after this
configuration, check **Node → System → Network** — a newly created OVS
bridge can show as **not active** if "Apply Configuration" wasn't
clicked after creating the bridge/ports. Re-apply, then restart the
affected VM(s).

---

## 7. Verification

Confirm DHCP leasing and VLAN placement are correct **before** moving on
to firewall rules — get this baseline working first, since firewall
rules will otherwise mask genuine networking problems.

From each VM, check the assigned IP (`ipconfig` on Windows, `ip a` on
Linux) and confirm it falls in the expected VLAN's DHCP range
(`.11–.250` per subnet in this build):

| VM | Expected subnet |
|---|---|
| WIN10-1 / WIN11-1 / DC | 10.0.1.x |
| Kali (attack) | 10.0.3.x |
| Kali Purple | 10.0.2.x |

### Implicit deny demonstration

pfSense allows all outbound traffic by default only on the **LAN**
interface (net1 / RANACORP) — every other interface (OPT1/BLUETEAM,
OPT2/REDTEAM) is **implicit-deny** until explicit allow rules are
created. This was verified by:

- Pinging `8.8.8.8` from **Kali Purple (BLUETEAM/OPT1)** → fails, blocked
  by implicit deny
- Pinging `8.8.8.8` from a **RANACORP** host → succeeds, since LAN
  traffic is allowed by default

This default exists so the LAN interface can't accidentally lock out
admin access to pfSense itself before any rules are configured — every
other interface starts fully closed and has to be deliberately opened.

---

## 8. Inter-VLAN Firewall Policy

VLAN interfaces and OVS ports only provide connectivity — actual
segmentation is enforced by per-interface firewall rules layered on top
of the implicit-deny behavior above. Full rule-by-rule detail lives in
`Firewall-Rules/README.md`; summarized:

- **RANACORP → BLUETEAM:** allowed only on Wazuh/Velociraptor
  enrollment and log-shipping ports — all else blocked
- **RANACORP → REDTEAM / Home:** blocked
- **RANACORP → Internet:** allowed
- **BLUETEAM → Home / REDTEAM:** blocked; internet allowed
- **REDTEAM → BLUETEAM / Home:** blocked; internet allowed; attacks
  against RANACORP permitted but governed manually rather than via a
  standing allow-all rule
- **WAN:** only management access to internal UIs from the home/admin
  network allowed inbound; all else blocked

---

## 9. Notes / Lessons Learned

- OVS ports are tagged (access-port style) at the Proxmox/OVS layer —
  pfSense itself never sees or configures 802.1Q tags; it just has three
  "plain" interfaces
- Any VM whose bridge/VLAN assignment needs to change must be powered
  off first — this applies to pfSense as well as the VMs behind it
- A freshly created OVS bridge is not necessarily active until
  "Apply Configuration" is explicitly clicked — worth checking first
  when a VM has no connectivity right after this setup
- Verify basic IP addressing/DHCP across every VM *before* writing any
  firewall rules — isolates whether a later connectivity issue is
  network config or firewall policy
- The LAN interface's default-allow behavior is a deliberate pfSense
  safety net, not a gap — every other interface starts closed

---

## 10. Next Steps

- [ ] Build out explicit per-interface firewall rule sets (see
      `Firewall-Rules/README.md`)
- [ ] Configure DHCP static mappings for infrastructure hosts (DC,
      Wazuh manager)
- [ ] Re-run the implicit-deny ping test after firewall rules are in
      place to confirm intended behavior still holds

