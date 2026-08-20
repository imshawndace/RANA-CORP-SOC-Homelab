# Active Directory Setup — RANACORP Homelab

## 1. Overview

This covers standing up the domain controller for the RANACORP domain:
installing AD DS and Certificate Services, promoting the server to a DC,
creating the initial OU/user/group structure, setting a baseline GPO, and
domain-joining the Windows workstations.

| Component | Detail |
|---|---|
| Domain name | `ranacorp.local` *(adjust to your actual domain)* |
| DC role | Windows Server — AD DS + AD CS (Enterprise Root CA) |
| Domain-joined hosts | WIN10-1, WIN11-1 |
| VLAN | 10 (RANACORP) |

---

## 2. Installing AD DS

Performed on the Windows Server VM via **Server Manager**.

1. **Manage → Add Roles and Features**
2. Installation type: **Role-based or feature-based installation**
3. Select the target server (the local DC)
4. Server role: **Active Directory Domain Services**
   — accept the prompt to add required features
5. Proceed through the wizard defaults → **Install**

> Note: bumping the VM's RAM to 8 GB before this step speeds up
> installation noticeably; 4 GB works but is slower.

Once installation completes, select **Promote this server to a domain
controller**:

1. **Add a new forest** → domain name: `ranacorp.local`
2. Set the Directory Services Restore Mode (DSRM) password
3. Accept the auto-generated NetBIOS name
4. Keep default paths for database/log/SYSVOL
5. Review options → **Install**
6. Server restarts to complete promotion

> A warning about no static IP being configured is expected at this
> stage — addressed separately in network configuration, not during AD
> install itself.

---

## 3. Installing AD Certificate Services (Enterprise Root CA)

Same **Add Roles and Features** flow, run after AD DS promotion:

1. Select **Active Directory Certificate Services** → add required
   features
2. On the role services screen, ensure **Certification Authority** is
   checked
3. Complete the wizard → **Install**
4. After install, select **Configure Active Directory Certificate
   Services**:
   - Credentials: default (Administrator)
   - Role: **Certification Authority**
   - Setup type: **Enterprise CA**
   - CA type: **Root CA**
   - Private key: **Create a new private key**
   - Cryptography: **SHA256**
   - Common name: defaults to the domain (e.g. `ranacorp-CA`)
   - Validity period: 5 years (sufficient for lab lifespan)
   - Certificate database: default locations
5. **Configure** → confirm "Configuration succeeded"

---

## 4. Organizational Units, Users, and Groups

Via **Tools → Active Directory Users and Computers**.

### OU structure
- By default, Windows lists built-in users and groups together under
  **Users**. Created a dedicated **Groups** OU and moved all built-in
  groups into it, leaving **Users** containing only actual user accounts
  — purely for readability/organization.

### Users created

| User | Purpose | Notes |
|---|---|---|
| Domain user 1 | Standard user account | Password never expires (lab setting) |
| Domain user 2 | Standard user account | Different password than user 1 |
| SQL service account | Service account, intentionally weak credential | Deliberately created as an attackable target for later exercises |

> **Lab-only insecure practice, documented intentionally:** the SQL
> service account was given an easily-crackable password (meets minimum
> complexity but low entropy — upper+lowercase word, sequential digits,
> one symbol), and that password was also written into the account's
> **Description** field in plaintext. This mirrors a real
> misconfiguration pattern seen in the wild and is set up deliberately as
> a target for later credential-access exercises — not a mistake to
> replicate in a real environment.

Fastest method for creating similarly-privileged accounts: right-click an
existing user → **Copy**, which carries over matching attributes.

### File share
Created via **File and Storage Services → Shares → New Share (SMB
Quick)**, with default caching/permissions settings — used later as a
target share for access/enumeration exercises.

### Service Principal Name (SPN)
Registered an SPN for the SQL service account from an elevated command
prompt:

```cmd
setspn -q */ranacorp.local
```

Confirms the SPN is associated with the domain and discoverable —
relevant later for Kerberoasting-style exercises.

---

## 5. Baseline Group Policy

Via **Group Policy Management** (run as administrator):

1. Right-click the domain (`ranacorp.local`) → **Create a GPO in this
   domain, and link it here**
2. Name: `Disable Windows Defender`
3. Edit the GPO:
   `Computer Configuration → Policies → Administrative Templates →
   Windows Components → Microsoft Defender Antivirus`
4. **Turn off Microsoft Defender Antivirus** → set to **Enabled**
   (enabling this policy is what disables Defender — the double
   negative here is easy to misread)
5. Right-click the GPO → **Enforced**

> This GPO intentionally weakens endpoint defenses domain-wide to keep
> red team exercises unobstructed by default AV detection/quarantine —
> a deliberate lab design choice, not a real-world recommendation.

---

## 6. Domain Joining Workstations

Repeated per workstation (WIN10-1, WIN11-1):

### DNS configuration
1. Network & Internet settings → Ethernet adapter → DNS server
   assignment → **Edit**
2. Change from **Automatic** to **Manual**, enable IPv4
3. Set **Preferred DNS** to the domain controller's IP (e.g.
   `10.0.1.2`)
4. Save

This points the workstation's DNS resolution at the DC, which is
required for domain discovery to work.

### Joining the domain
1. Search → **Access work or school** → **Connect**
2. **Join this device to a local Active Directory domain**
3. Enter domain name: `ranacorp.local`
4. Authenticate with a **domain administrator** account (a standard
   domain user does not have permission to join machines to the domain)
5. Set account type on the device → **Administrator**
6. Restart to complete the join

### Verification
- On the DC: **Active Directory Users and Computers → Computers** —
  confirm the new workstation object appears
- Log in on the workstation using a domain user account (format:
  `DOMAIN\username` or select the domain from the login screen) to
  confirm domain authentication works end-to-end

---

## 7. Notes / Lessons Learned

- RAM allocation directly affects AD DS install/promotion speed — worth
  bumping temporarily during initial setup if available
- Only a domain admin account (not a regular domain user) can join new
  machines to the domain — a common point of confusion during first
  domain-join attempts
- The "Turn off Microsoft Defender Antivirus" policy name is
  counter-intuitive: **Enabling** the policy is what disables Defender
- Static DNS pointing at the DC must be set on each workstation before
  the domain-join step will succeed — domain discovery relies on it
- The intentionally weak SQL service account is a deliberate detection
  target, not an oversight — worth a comment in AD itself (or here) so
  it isn't "fixed" by mistake later in the project

---

## 8. Next Steps

- [ ] Deploy Wazuh agents to the DC and workstations (see
      `Wazuh-deployment.md`)
- [ ] Install Sysmon on domain-joined endpoints (see `sysmon-setup.md`)
- [ ] Document AD-focused attack scenarios (Kerberoasting against the
      SPN'd service account, password spraying, etc.) in `Detections/`
      once telemetry pipeline is confirmed working

