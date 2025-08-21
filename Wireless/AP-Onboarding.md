
# Catalyst 9800 AP Onboarding — End‑to‑End Guide (GUI + CLI)

> **Scope:** Step‑by‑step onboarding of Access Points (APs) to a Cisco Catalyst 9800 WLC, including **what to build, in what order**, and **exact GUI navigation** for both **Central‑Switched** and **Flex (Local‑Switched)** SSIDs.  
> **UI note:** Menu names can vary slightly by IOS‑XE release (approx. 17.6 – 17.15). If a label isn’t exact, use the closest match under **Configuration → Tags & Profiles** or **Configuration → Security/Interfaces/Wireless**.

---

## 1) Mental Model (What maps to what)

- **WLAN Profile** → Defines the SSID (name, auth/security).
- **Policy Profile** → Defines client behavior (VLAN, ACLs, QoS, session timers, central vs local switching, DHCP required, web‑auth, etc.).
- **Policy Tag** → Maps **WLAN(s)** to **Policy Profile(s)** and pushes that bundle to APs.
- **RF Profiles** → Radio settings per band (2.4/5/6 GHz: channels, power, data rates, RRM).
- **RF Tag** → Binds RF profiles to APs.
- **AP Join Profile** → AP‑level join and timers (optional).
- **Flex Profile** → Required for Flex; controls local switching/auth, optional VLAN/ACL overrides.
- **Site Tag** → Assigns AP Join Profile and (if Flex) the Flex Profile to APs.

**Every AP has three tags:** **Policy Tag**, **RF Tag**, **Site Tag**.

---

## 2) Prerequisites (one‑time or per‑site)

### 2.1 Controller basics
**GUI path:** `Administration → Time` and `Configuration → Wireless → Country`  
1) **Time/NTP**: Add NTP servers and set time/zone.  
2) **Country codes**: Add all regulatory domains where APs will operate; **Save & Apply**.  
3) **Certificates/trust** (if doing 802.1X/guest portals): ensure RADIUS & portal certs are trusted.

### 2.2 Interfaces, VLAN reachability, routing
- Ensure the WLC can reach your **management**, **RADIUS**, **DHCP**, and **upstream gateways**.  
- If **Central‑Switched**, the **client VLAN** must be reachable **at or via the WLC** (SVI/sub‑interface or routed path).  
- If **Flex (Local‑Switched)**, the **AP’s access switchport is a trunk** that carries **local client VLANs**; DHCP exists locally at the site.

> **GUI hint (9800):** L2/L3 lives under `Configuration → Interfaces`. Create VLAN interfaces or sub‑interfaces as required by your design. Bind physical ports to trunks as needed.

### 2.3 AAA (if 802.1X)
**GUI path:** `Configuration → Security → AAA`  
- **RADIUS Servers:** Add each server (IP, key).  
- **Server Groups:** Optional grouping.  
- **Method Lists:** Create **dot1x** method list and reference your RADIUS group/servers.

### 2.4 AP discovery (DHCP Option 43 or DNS)
- DHCP Option 43 or DNS `CISCO-CAPWAP-CONTROLLER` → points APs to the WLC.  
- Alternatively, statically prime APs (console/CLI) with the controller address.

---

## 3) Build Order (always do in this sequence)

1) **WLAN Profile(s)** — SSID + security.  
2) **Policy Profile(s)** — VLAN, central vs local switching, auth options, ACL/QoS.  
3) **Policy Tag** — Map WLAN → Policy Profile.  
4) **RF Profile(s)** and **RF Tag**.  
5) **(Optional) AP Join Profile**.  
6) **Site Tag** — For Flex, attach the **Flex Profile** here.  
7) **Assign Tags to APs** — Policy/RF/Site.

---

## 4) Central‑Switched SSID — Step‑by‑Step (GUI)

> **Use when:** You want client data to tunnel to the WLC. VLAN lives centrally.

### A) Create WLAN (SSID)
**GUI path:** `Configuration → Tags & Profiles → WLANs → + Add`  
1) **Name**: e.g., `CORP-CENTRAL`  
2) **WLAN ID**: unique (e.g., 20)  
3) **SSID**: visible name (e.g., `Corp`)  
4) **Security**:  
   - PSK: `Security → Layer2 → WPA2/WPA3` → set **PSK**.  
   - 802.1X: `Security → AAA Policy` → select your **dot1x Method List**.  
5) **Save & Apply**.

### B) Create Policy Profile (Central Switching)
**GUI path:** `Configuration → Tags & Profiles → Policy → Policy Profiles → + Add`  
1) **Name**: `PP-CENTRAL-CORP`  
2) **VLAN**: set the **client VLAN ID** used at the WLC/central site.  
3) **Central Switching**: **Enabled** (default).  
4) **(Optional) DHCP Required**, **Session Timeout**, **ACLs**, **QoS**.  
5) **Save & Apply**.

### C) Map WLAN → Policy (Policy Tag)
**GUI path:** `Configuration → Tags & Profiles → Tags → Policy → + Add`  
1) **Name**: `PT-CORP-CENTRAL`  
2) **WLAN‑to‑Policy**: Add `CORP-CENTRAL` → `PP-CENTRAL-CORP`.  
3) **Save & Apply**.

### D) RF Profiles + RF Tag
**GUI path:**  
- RF Profiles: `Configuration → Tags & Profiles → RF Profiles → + Add` (per band: 2.4/5/6)  
- RF Tag: `Configuration → Tags & Profiles → Tags → RF → + Add`  
1) Create or tune RF profiles: channels, power, data rates, 6 GHz PSC.  
2) Create **RF Tag** (e.g., `RT-DEFAULT`) referencing the RF profiles.  
3) **Save & Apply**.

### E) (Optional) AP Join Profile
**GUI path:** `Configuration → Tags & Profiles → AP Join → + Add`  
- Configure timers, syslogs, etc. Name: `APJ-DEFAULT`. **Save & Apply**.

### F) Site Tag (No Flex for central)
**GUI path:** `Configuration → Tags & Profiles → Tags → Site → + Add`  
1) **Name**: `ST-CENTRAL`  
2) **AP Join Profile**: `APJ-DEFAULT` (or `default-ap-profile`).  
3) **Flex Profile**: **None** (leave empty).  
4) **Save & Apply**.

### G) Assign Tags to APs
**GUI path:** `Configuration → Wireless → Access Points → (click AP) → Tags`  
- **Policy Tag**: `PT-CORP-CENTRAL`  
- **RF Tag**: `RT-DEFAULT`  
- **Site Tag**: `ST-CENTRAL`  
**Apply** and repeat for all relevant APs. (Bulk select is available.)

### H) Validate (GUI + CLI)
**GUI**:  
- `Monitoring → Wireless → Clients` → verify clients on **Corp** SSID.  
- `Configuration → Tags & Profiles → Tags → Policy → (PT-CORP-CENTRAL)` → verify mapping.  
- `Configuration → Wireless → Access Points` → add **Tags** columns to the table to confirm.  

**CLI checks:**
```
show wlan summary
show wireless tag policy detail PT-CORP-CENTRAL
show ap tag summary
show wireless client summary
```

---

## 5) Flex (Local‑Switched) SSID — Step‑by‑Step (GUI)

> **Use when:** You want client data to break out locally at the branch. AP trunks client VLANs; DHCP is local.

### A) Create WLAN (SSID)
**GUI path:** `Configuration → Tags & Profiles → WLANs → + Add`  
- **Name/ID/SSID**: e.g., `BRANCH-FLEX` / 30 / `Branch`  
- **Security**: PSK or 802.1X (as in Central). **Save & Apply**.

### B) Create Policy Profile (Local Switching)
**GUI path:** `Configuration → Tags & Profiles → Policy → Policy Profiles → + Add`  
1) **Name**: `PP-FLEX-BRANCH`  
2) **Central Switching**: **Disable** (toggle OFF).  
3) **VLAN**: **branch client VLAN ID** that the AP will tag on its **local trunk**.  
4) **Auth Model**:  
   - **Central Auth (common)**: keep **Central Authentication = ON** so 802.1X goes to HQ RADIUS, but **data** is local.  
   - **Local Auth (optional)**: set **Central Authentication = OFF** and use **Flex Local Auth** (see Flex Profile).  
5) Optional: DHCP required, ACLs, QoS. **Save & Apply**.

### C) Create Flex Profile
**GUI path:** `Configuration → Tags & Profiles → Flex Profiles → + Add`  
1) **Name**: `FX-BRANCH-A`  
2) **Local Authentication** (optional): enable and select **Local RADIUS Server Group** if doing local auth.  
3) **(Optional) VLAN/ACL Overrides**: define per‑site overrides if you use them.  
4) **Save & Apply**.

### D) Map WLAN → Policy (Policy Tag)
**GUI path:** `Configuration → Tags & Profiles → Tags → Policy → + Add`  
- **Name**: `PT-BRANCH-FLEX`  
- Add mapping: `BRANCH-FLEX` → `PP-FLEX-BRANCH`  
- **Save & Apply**.

### E) RF Profiles + RF Tag
(Reuse the same RF tag as central, or create a site‑specific one.)  
**GUI path:** as above in Central. Create **RT-BRANCH** if needed. **Save & Apply**.

### F) Site Tag (attach the Flex Profile)
**GUI path:** `Configuration → Tags & Profiles → Tags → Site → + Add`  
1) **Name**: `ST-BRANCH-A`  
2) **AP Join Profile**: `APJ-DEFAULT` (or as desired).  
3) **Flex Profile**: **FX-BRANCH-A** (attach it).  
4) **Save & Apply**.

### G) Switchport to AP (Access‑layer switch)
**Catalyst switch CLI (example):**
```
interface GigabitEthernet1/0/24
 description AP to Branch-A
 switchport
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan <AP_MGMT_VLAN>
 switchport trunk allowed vlan <AP_MGMT_VLAN>,<BRANCH_CLIENT_VLANS>
 spanning-tree portfast trunk
 power inline auto
```

> Ensure **local DHCP** exists for each **BRANCH_CLIENT_VLAN**. The AP management VLAN should reach the WLC.

### H) Assign Tags to APs
**GUI path:** `Configuration → Wireless → Access Points → (click AP) → Tags`  
- **Policy Tag**: `PT-BRANCH-FLEX`  
- **RF Tag**: `RT-BRANCH` (or `RT-DEFAULT`)  
- **Site Tag**: `ST-BRANCH-A`  
**Apply**.

### I) Validate (GUI + CLI)
**GUI:**  
- `Monitoring → Wireless → Clients` → client on **Branch** SSID gets **local** site IP.  
- `Configuration → Wireless → Access Points` → confirm tags.  

**CLI checks:**
```
show ap tag summary
show wireless tag site detail ST-BRANCH-A
show wireless client summary
```

---

## 6) Common Policy Knobs (quick reference)

**Policy Profile**: VLAN, Central Switching, Central Auth, DHCP Required, Session Timeout, Pre/Post‑auth ACLs, QoS (Platinum/Gold/Silver/Bronze), Device Classification, mDNS/Bonjour, Web‑auth portal.  
**RF Profile**: DCA channels, Power, Min/Max Tx power, Data‑rate sets, 6 GHz PSC, RRM thresholds, Client Load Balancing.  
**Flex Profile**: Local Auth enable + RADIUS group, optional VLAN/ACL overrides.

---

## 7) Minimal CLI Skeletons (ready to paste)

> **Tip:** Use CLI to bootstrap, then refine in GUI.

**Create WLAN (PSK example)**
```
conf t
wlan CORP-CENTRAL 20 Corp
  no shutdown
  security wpa psk set-key ascii 0 <your_psk>
exit
```

**Policy Profile — Central Switching**
```
wireless profile policy PP-CENTRAL-CORP
  vlan <CENTRAL_VLAN>
  dhcp required
  central association
  central authentication
  central switching
exit
```

**Policy Profile — Flex Local Switching**
```
wireless profile policy PP-FLEX-BRANCH
  no central switching
  vlan <BRANCH_VLAN>
  central authentication           ! keep for Central-Auth/Local-Switching
  ! no central authentication      ! (uncomment to disable for Local-Auth)
exit
```

**Flex + Site Tag**
```
wireless profile flex FX-BRANCH-A
  local-auth radius server-group <LOCAL_RADIUS_GROUP>   ! if doing Local-Auth
exit

wireless tag site ST-BRANCH-A
  flex-profile FX-BRANCH-A
  ap-profile APJ-DEFAULT
exit
```

**Policy Tag**
```
wireless tag policy PT-BRANCH-FLEX
  wlan BRANCH-FLEX policy PP-FLEX-BRANCH
exit
```

**Assign Tags to AP**
```
ap name <AP_NAME> policy-tag PT-BRANCH-FLEX
ap name <AP_NAME> site-tag   ST-BRANCH-A
ap name <AP_NAME> rf-tag     RT-BRANCH
```

**Checks**
```
show wlan summary
show wireless tag policy detail PT-BRANCH-FLEX
show wireless tag site detail ST-BRANCH-A
show ap tag summary
show wireless client summary
```

---

## 8) Troubleshooting Quick Hits

- **AP joined but no SSID**: Verify **Policy Tag** mapping includes your WLAN→Policy pair and that the AP has the correct Policy Tag.  
- **Clients fail DHCP**:  
  - Central: verify **client VLAN** exists/reachable at WLC; helper addresses configured.  
  - Flex: verify **AP switchport trunk allows client VLANs** and **site DHCP** is present.  
- **802.1X failures**: Check `Security → AAA → Accounting/Statistics` and WLC syslog; validate method list and shared keys.  
- **RF issues**: Confirm correct **RF Tag** and per‑band RF Profiles are applied to the AP.  
- **Wrong tags on AP**: In `Configuration → Wireless → Access Points`, edit the AP and correct **Policy/RF/Site tags**; use bulk edit for many APs.

---

## 9) Build Checklist (copy/paste)

- [ ] NTP/time, country codes, interfaces/routing ready  
- [ ] AAA servers + method lists (if 802.1X)  
- [ ] **WLAN** created (SSID/security)  
- [ ] **Policy Profile** created (VLAN, central vs local, auth)  
- [ ] **Policy Tag** mapping done (WLAN → Policy)  
- [ ] **RF Profiles** & **RF Tag** created  
- [ ] **(Optional) AP Join Profile** created  
- [ ] **Site Tag** created (Flex profile attached for local switching)  
- [ ] **AP switchport trunk** (Flex only)  
- [ ] **Assign Tags to APs** (Policy/RF/Site)  
- [ ] **Validate** (GUI + CLI)

---

### Appendix: GUI Quick Links (by area)

- **WLANs**: `Configuration → Tags & Profiles → WLANs`  
- **Policy Profiles**: `Configuration → Tags & Profiles → Policy → Policy Profiles`  
- **Policy Tags**: `Configuration → Tags & Profiles → Tags → Policy`  
- **RF Profiles**: `Configuration → Tags & Profiles → RF Profiles`  
- **RF Tags**: `Configuration → Tags & Profiles → Tags → RF`  
- **Flex Profiles**: `Configuration → Tags & Profiles → Flex Profiles`  
- **Site Tags**: `Configuration → Tags & Profiles → Tags → Site`  
- **AP Join Profiles**: `Configuration → Tags & Profiles → AP Join`  
- **AP Inventory/Tag Assignment**: `Configuration → Wireless → Access Points`  
- **AAA**: `Configuration → Security → AAA`  
- **Time/NTP**: `Administration → Time`  
- **Country Codes**: `Configuration → Wireless → Country`  
- **Interfaces**: `Configuration → Interfaces`
