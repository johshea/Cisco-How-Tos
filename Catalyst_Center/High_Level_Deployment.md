# Cisco Catalyst Center Deployment – Step‑by‑Step Guide

> This guide covers planning, installing, and bootstrapping **Cisco Catalyst Center** on either physical appliance (2nd/3rd‑gen) or **virtual appliance on VMware ESXi**. Use it as a practical runbook for greenfield installs or lab deployments.

---

## 0) Quick Checklist (print me)
- [ ] Decide **deployment model**: Physical appliance (2nd/3rd‑gen) or **Virtual Appliance on ESXi** (single node or 3‑node).
- [ ] Size & **reserve resources** (VA: 32 vCPU, 256 GB RAM, 3 TB SSD minimum), and ensure datastore capacity for snapshots/backups.
- [ ] Prepare **IP plan** (Enterprise/data, Management/OOB, optional Internet‑access interface), **subnets**, **gateway**, **DNS**, **NTP**, **SMTP**.
- [ ] Create **DNS A/PTR** records and confirm forward/reverse resolution.
- [ ] Open **firewall** paths to managed devices, ISE/AAA, DNAC updates, NTP, DNS, SMTP, Syslog/NetFlow/Telemetry, Smart Licensing.
- [ ] Prepare **Smart Account/Virtual Account** and license entitlements.
- [ ] Get **device credentials** ready: CLI (SSH), SNMPv2/v3, HTTP(S) for controllers/APIs.
- [ ] Plan **cluster** (single vs 3‑node) and **DR** strategy (main/recovery/witness).
- [ ] Download the correct **OVA/ISO** or rack the appliance. Verify version compatibility with your device fleet.
- [ ] Decide **air‑gapped** or internet‑connected; stage offline content if needed.
- [ ] Schedule a **maintenance window** for the initial build (60–120 minutes typical for VA import + bootstrap).

---

## 1) Plan & Prerequisites
1. **Choose the form factor**
   - **Hardware appliance:** 2nd‑ or 3rd‑generation (rack, power, cabling).
   - **Virtual Appliance (VA) on VMware ESXi:** Single node (lab) or 3‑node (production).  
2. **Capacity** (VA minimums)
   - 32 vCPU (64 GHz reservation), 256 GB RAM (reserved), 3 TB SSD, ~180 MB/s IO, 2–2.5k IOPS @ <5 ms.
3. **Networking**
   - Interfaces: **Enterprise/Data**, **Management/OOB**, optional **Internet‑access**. Trunk/access per design.
   - **IP addressing**, VLANs, gateways, DNS/NTP time zone, reachable SMTP.
   - **DNS records** for FQDN; ensure forward + reverse lookup.
4. **Security & compliance**
   - Optional **FIPS mode**, password policy, SSO/IdP plan, certificates (internal or public).
5. **Licensing**
   - Smart Account + Virtual Account, entitlements for Catalyst Center features.
6. **Integrations (optional)**
   - Cisco ISE/pxGrid, controllers (WLC), Prime migration, Syslog/NetFlow/Telemetry receivers.
7. **Change control**
   - Maintenance window, rollback plan, snapshots (if VA), and backup target ready.

---

## 2) Deploy the Platform

### A. Physical Appliance (2nd/3rd‑Gen)
1. **Rack & cable** power and data/management interfaces per the hardware guide.
2. **Power on** and connect to the console (or KVM) to access the setup wizard.
3. **Run the initial wizard** to set:
   - Admin credentials
   - Interface mode (IPv4/IPv6), IP addresses, mask, gateway
   - DNS, NTP, time zone
   - Optional **FIPS mode**
4. **Save and apply**; the platform will configure underlying services and reboot as needed.

### B. Virtual Appliance on VMware ESXi
1. **Import OVA** into vCenter/ESXi. Choose thick provisioning and reserve CPU/RAM per sizing.
2. **Attach networks** for Enterprise/Data, Management/OOB, and optional Internet‑access port groups.
3. **Power on** the VM; open the VM console to complete the **interactive setup**:
   - Set admin credentials.
   - Select addressing mode (IPv4/IPv6), configure interfaces.
   - Configure DNS/NTP/time zone, optionally enable **FIPS**.
4. (Optional) If using **CC VA Launcher**, follow its interactive workflow to pre‑seed inputs.
5. **Wait for services** to initialize. Note the HTTPS URL (FQDN) for the first login.

> **Air‑gapped?** Pre‑stage software images/patches on reachable storage and use the offline/PreLoad flows during upgrades.

---

## 3) First Login & Quick Start
1. Browse to **https://\<FQDN\>** and sign in (change default password immediately if prompted).
2. Complete the **Quick Start** tasks:
   - Create a permanent **admin** user/password.
   - Accept EULA and set telemetry preferences.
   - Configure **Smart Licensing** (or plan offline license flow if air‑gapped).
   - Opt in to **assurance** metrics collection.
3. Run **System Updates** to ensure you are on the desired release/patch level before onboarding devices.

---

## 4) (Optional) Form a 3‑Node Cluster
> Recommended for production. Perform on identical builds and versions.
1. Prepare **three nodes** (appliances or VMs) with unique IPs/hostnames, synchronized NTP/DNS.
2. From the **Cluster Setup** workflow, add secondary nodes to the primary.
3. Validate cluster health, data replication status, and that services report **Healthy/Green**.
4. If required, configure **Disaster Recovery** (main, recovery, witness) and test controlled failover.

---

## 5) Post‑Install Hardening
1. Replace any **default certificates** with CA‑signed certs (internal or public).
2. Enable **SSO** (SAML/OIDC) and enforce MFA via your IdP.
3. Define **role‑based access** (RBAC) profiles.
4. Configure **Secure Syslog**, SNMPv3, and limit management access with ACLs as needed.
5. Audit **time sync** (NTP), DNS resolution, and backup settings.

---

## 6) Baseline Design in the UI
1. **Design → Sites**: Create Areas, Buildings, and Floors.
2. **Design → Network Settings**:
   - **AAA/ISE**: RADIUS/TACACS+, pxGrid (if used).
   - **DHCP/DNS/NTP/Syslog/NetFlow**: add servers and policies.
   - **IP Address Pools**: Global and per‑site pools; map to sites.
   - **Wireless**: Controllers, SSIDs/policies (if managing WLCs).
3. **Credentials**: Define CLI (SSH), SNMPv2/v3, HTTP(S), NETCONF as needed.
4. **Image Repository**: Upload **golden images** for platforms; create image compliance policies.
5. **Telemetry**: Ensure assurance data is enabled; set SNMP traps/streaming as per design.

---

## 7) Device Onboarding
1. **Discovery** (Devices → Discovery):
   - Select method: **CDP/LLDP**, IP range, seed device, or PnP.
   - Apply **credential profiles** and discovery protocols.
2. **Inventory**: Claim devices, assign to **sites**, and verify reachability.
3. **Provision**:
   - Apply **Day‑N** templates/policies.
   - Push **golden images** via Image Update workflow.
4. **Assurance**:
   - Confirm device health, client onboarding, and application telemetry dashboards.

---

## 8) Backups, DR & Maintenance
1. **Backups**: Configure remote repository, schedule regular backups, and **test restore** procedures.
2. **Disaster Recovery** (optional): Set up **main/recovery/witness** sites and validate failover.
3. **Upgrade Strategy**:
   - Preload upgrade files (air‑gapped) or use online repositories.
   - Follow release notes; upgrade outside business hours; snapshot VA beforehand.
4. **Monitoring**: Use system health, cluster status, and assurance KPIs for ongoing ops.

---

## 9) Validation – Ready for Production
- **DNS/NTP/SMTP** tests pass; time and FQDN resolve correctly.
- **Licensing** is in the **Authorized** state (or planned offline method).
- **Image compliance** policies defined.
- **Discovery** finds expected devices; **Provision** pushes succeed.
- **Assurance** shows green posture; baseline reports exported.

---

## 10) Common Pitfalls & Tips
- **Under‑reservation** on ESXi causes instability—**reserve** CPU/RAM per minimums.
- **DNS reverse lookup** missing → discovery failures and certificate issues.
- **NTP skew** breaks clustering and SSO—monitor and alert on drift.
- **Certificates**: Use a proper FQDN/CN and SubjectAltName; update trust chains on devices if needed.
- **Air‑gapped**: Stage upgrades and images in advance to avoid long change windows.
- **ISE integration**: Ensure pxGrid certificates and user mapping before enabling features that depend on it.

---

## Appendix A – Minimums for ESXi Virtual Appliance (reference)
- **CPU**: 32 vCPU (reserve ~64 GHz)
- **Memory**: 256 GB (fully reserved)
- **Storage**: 3 TB SSD
- **IO**: ~180 MB/s; 2k–2.5k IOPS; <5 ms latency

> Always consult the current release’s Deployment Guide for exact specs/changes.

---
