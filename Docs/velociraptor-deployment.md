# Velociraptor Deployment — RANACORP Homelab

## 1. Overview

Velociraptor is deployed as the DFIR/endpoint-hunting platform, running
on an Ubuntu Desktop VM inside **BLUETEAM (VLAN20)**, alongside Wazuh.
Clients (agents) are installed on the RANACORP domain endpoints so they
can be hunted against and have artifacts collected from them.

| Component | Detail |
|---|---|
| Server host | Ubuntu Desktop VM, VLAN20 (BLUETEAM) |
| Server static IP | `10.0.2.3` *(adjust to your actual lab IP)* |
| Frontend port | 8000 (client callback) |
| GUI port | 8889 |
| Clients deployed | WIN10-1, WIN11-1, Domain Controller |

---

## 2. Server VM Provisioning

Created as a new Proxmox VM:

| Setting | Value |
|---|---|
| Name | Velociraptor |
| OS/ISO | Ubuntu Desktop |
| Graphics | SPICE |
| Machine type | q35 |
| QEMU Agent | Enabled |
| Display memory | 128 MB |
| Disk | 100 GB (artifacts/collected data need headroom) |
| CPU | 2 cores |
| Memory | 4096 MB (4 GB) |
| Network | Bridge `vmbr1`, VLAN tag `20` (BLUETEAM) |
| Firewall | Disabled at the Proxmox NIC level (segmentation handled by pfSense) |

Ubuntu Desktop (not Server) was chosen deliberately for this build —
install proceeds with the standard interactive installer: language,
keyboard, wired connection, default app selection (no third-party apps
installed), hostname set to `velociraptor`, user account created, and
timezone set.

> Remember to eject the installer ISO from the VM's CD/DVD drive in
> Proxmox after install completes, before the reboot — otherwise the VM
> may attempt to boot from the installer again.

---

## 3. Setting a Static IP

By default the VM receives a DHCP lease in the BLUETEAM range (verified
via `ip a`) — confirms both connectivity and correct VLAN placement.
Since this is a server, that lease is then converted to a static
reservation in pfSense:

1. **pfSense → Status → DHCP Leases**
2. Find the Velociraptor VM's current lease → click the **+** to convert
   it to a static mapping
3. Assign a fixed IP outside of address conflicts with other
   infrastructure already reserved in that VLAN (e.g. `10.0.2.3`,
   avoiding `10.0.2.2` already used by the Wazuh manager)
4. **Save**, then **Apply Changes**

On the Ubuntu VM, the interface won't pick up the new static lease until
it's refreshed. `net-tools` isn't installed by default on current Ubuntu,
so install it first if using `ifconfig`:

```bash
sudo apt update
sudo apt install net-tools

sudo ifconfig <interface> down
sudo ifconfig <interface> up
```

Confirm the new static IP is active with `ifconfig` (or `ip a`) before
proceeding.

---

## 4. Installing the Velociraptor Server

Download the latest release binary for Linux from the official
Velociraptor GitHub releases page (version referenced at time of this
build: `0.7.5.6` — **check the releases page for the current version**,
since the exact download URL is version-specific):

```bash
wget <velociraptor-release-url-for-your-version>

chmod +x velociraptor-<version>-linux-amd64
```

Organize into a dedicated directory:

```bash
sudo mkdir /opt/velociraptor
sudo mv velociraptor-<version>-linux-amd64 /opt/velociraptor/
cd /opt/velociraptor
```

### Generate the server config (interactive)

```bash
./velociraptor-<version>-linux-amd64 config generate -i
```

Key prompts and answers used for this deployment:

| Prompt | Answer |
|---|---|
| Deployment type | Self-signed SSL |
| Platform | Linux |
| Datastore location | `/opt/velociraptor` |
| Write logs | Yes → default `logs` subdirectory |
| PKI certificate validity | 1 year (adjust for your rebuild cadence) |
| Use registry for client writeback | Yes |
| Public frontend domain/IP | Static server IP, e.g. `10.0.2.3` (not a public DNS name — internal-only lab) |
| DNS type | None (not internet-reachable, internal network only) |
| Experimental websocket | No |
| Frontend port | 8000 (default) |
| GUI port | 8889 (default) |
| Admin username | (chosen username) |
| Admin password | (set password) |
| Additional users | None (hit enter to skip) |
| Config output location | `/opt/velociraptor` |

> Keep note of whatever port you assign for the frontend (8000 by
> default) — if any other service is later installed on this same VM, it
> must avoid that port.

### Edit the generated config

The generated `server.config.yaml` binds the GUI to loopback
(`127.0.0.1`) by default — this needs to be changed to the server's
actual static IP so it's reachable from other machines:

```bash
nano server.config.yaml
```

Locate the `GUI` → `bind_address` field and change it from
`127.0.0.1` to the static IP (e.g. `10.0.2.3`). Save (`Ctrl+X`, `Y`,
`Enter`).

### Build and run the server package

```bash
sudo ./velociraptor-<version>-linux-amd64 --config server.config.yaml \
  config repack ...   # generates the Debian package per config

chmod 764 <generated-package>.deb
sudo dpkg -i <generated-package>.deb
```

Verify the service is running:

```bash
systemctl status velociraptor_server
```

Should report **active (running)**.

---

## 5. Accessing the GUI

From a browser on any host with network access to VLAN20 (e.g. a
RANACORP Windows machine, per firewall policy):

```
https://10.0.2.3:8889
```

Accept the self-signed certificate warning, then log in with the admin
username/password set during config generation.

---

## 6. Client (Agent) Deployment

Repeated per endpoint (WIN10-1, WIN11-1, Domain Controller).

### Get the client files
Two files are required, both retrieved from the Velociraptor GUI:

1. **Client config YAML** — GUI home page → **current orgs** → client
   config YAML → downloads to the endpoint's Downloads folder. Verify it
   references the correct server IP:port (e.g. `10.0.2.3:8000`) by
   opening it in a text editor.
2. **Velociraptor Windows binary (MSI)** — from the official
   Velociraptor downloads/releases page, Windows MSI build, matching
   the same version as the server.

### Install
1. Run the downloaded MSI as **administrator** — installs to
   `C:\Program Files\Velociraptor\`
2. The installed package ships with a placeholder/invalid
   `client.config.yaml` — **delete it**
3. Copy the real client config YAML (downloaded from the GUI) into that
   same folder and **rename it to exactly `client.config.yaml`** — the
   filename must match precisely for the service to pick it up
4. Confirm the client appears in the Velociraptor GUI (home →
   search/magnifying glass) shortly after

### Domain Controller — file-share delivery method

Opening a browser directly on a server isn't good practice, so the
client binary and config were instead staged via the existing AD file
share (the SMB share set up during AD configuration):

1. From a RANACORP workstation, copy both the Velociraptor MSI and the
   renamed `client.config.yaml` into the network share
2. From the domain controller, copy both files locally (e.g. into
   `Documents`) — execution from the network share itself isn't
   necessary, just used as the transfer method
3. Run the MSI as administrator
4. Replace the installed placeholder config with the real
   `client.config.yaml` under `C:\Program Files\Velociraptor\`
5. Confirm the DC appears as a third client in the Velociraptor GUI

### Service verification (any endpoint)
Windows **Services** console → confirm **Velociraptor** service is
present and **Running**.

---

## 7. Notes / Lessons Learned

- The client config filename must exactly match what the installed
  Velociraptor package expects (`client.config.yaml`) — a mismatched
  name means the agent won't call back even though installation
  "succeeded"
- The GUI's default `bind_address` (loopback) has to be manually
  corrected post-config-generation or the GUI won't be reachable from
  anywhere but the server itself
- Static IP reservation must happen *before* generating the server
  config, since the frontend/public IP is baked into the config at
  generation time
- Using the domain's existing SMB share as a file-transfer method to the
  DC avoided opening a browser directly on server — a reasonable
  operational habit worth carrying into real environments
- Velociraptor release binaries are version-specific in their download
  URL — always check the current release version rather than assuming
  the one referenced here is still current

---

## 8. Next Steps

- [ ] Run a first artifact collection / hunt against a client to confirm
      end-to-end functionality
- [ ] Document a sample DFIR walkthrough (collection → analysis) in
      `Detections/` or a dedicated write-up
- [ ] Consider integrating Velociraptor alerting/results with Wazuh for
      centralized visibility

