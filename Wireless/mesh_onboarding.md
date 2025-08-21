
# Catalyst 9800 — Mesh AP Configuration (GUI Step‑by‑Step)

> **Scope:** Configure Access Points to operate in **Mesh** on Cisco Catalyst 9800 WLC using the **Web UI**. Covers both **Bridge (Mesh)** and **Flex+Bridge** deployments, with RAP/MAP roles, tags, and validation.

---

## 0) Terms & Roles
- **RAP (Root AP):** Wired uplink to the LAN/WLC; anchors the mesh.
- **MAP (Mesh AP):** Wireless child that forms backhaul to a RAP (or another MAP).
- **Bridge (Mesh):** Pure mesh; data is bridged across the AWPP backhaul.
- **Flex+Bridge:** Mesh backhaul **plus** Flex features (local switching/resiliency).
- **BGN (Bridge Group Name):** Logical mesh domain boundary; RAP/MAP must match.

---

## 1) Prerequisites
- **Country codes, time/NTP** set on the WLC.
- **Discoverability:** APs can reach the WLC (DHCP Option 43, DNS `CISCO-CAPWAP-CONTROLLER`, or static priming).
- **AAA (for EAP-FAST mesh auth):**
  - You will add **AP MACs** (Base Ethernet MAC, no separators) to **Device Authentication**.
  - Create **method lists** for **Authentication (dot1x)** and **Authorization (credential-download)**.
- **Switch ports:**
  - RAP uplink is typically a **trunk**; set **native VLAN** for AP mgmt and allow client/bridge VLANs as needed.
- **Licensing:** Appropriate AP count and features are licensed on the WLC.

---

## 2) Global Mesh & Security
**GUI:** `Configuration → Wireless → Mesh`  
1) **Global parameters:** Leave defaults initially (can tune backhaul band, scanning, etc.).  
2) **(Optional) Mesh PSK provisioning:** If using mesh PSK instead of EAP‑FAST, configure keys here later; otherwise skip.

**GUI (AAA):** `Configuration → Security → AAA`
1) **Device Authentication (MAC allowlist):** `AAA Advanced → Device Authentication → Add`
   - Enter **Base Ethernet MAC** of each RAP/MAP **without** `.` `:` `-`.
2) **Method lists:** `AAA → Method List`
   - **Authentication (dot1x):** Create a list (e.g., `Mesh_Authentication`) → use **local** or **RADIUS**.
   - **Authorization (credential-download):** Create a list (e.g., `Mesh_Authz`) → **local**.

> Tip: If you must let an AP in **Bridge** mode join **before** tagging, temporarily enable default lists (CLI):  
> `aaa authentication dot1x default local` and `aaa authorization credential-download default local`.

---

## 3) Create Mesh Profile
**GUI:** `Configuration → Wireless → Mesh → Profiles → + Add`
1) **General:**
   - **Name** (e.g., `MESH-CAMPUS`), **Description**.
   - **Bridge Group Name (BGN)** (e.g., `BGN-CAMPUS`). (All RAP/MAP in the domain must match.)
   - **Backhaul Band/Channel Width**: choose 5 GHz or 6 GHz as per design.
2) **Advanced:**
   - **Authentication Method** = your **Authentication (dot1x)** list.
   - **Authorization Method** = your **Authorization (credential-download)** list.
   - (Optional) **Background scanning**, **Convergence**, **Ethernet Bridging** (enable if you bridge wired devices on RAP/MAP).

**(Optional) Mesh PSKs:** If using **mesh PSK** model, add up to five provisioned keys. (Use PSK provisioning window during migration.)

---

## 4) AP Join Profile (binds Mesh Profile)
**GUI:** `Configuration → Tags & Profiles → AP Join → + Add`
1) **General tab:** Name (e.g., `APJ-MESH`), description.
2) **AP tab:** Set **Mesh Profile** = `MESH-CAMPUS`.
3) **EAP parameters:** **EAP Type** = *EAP-FAST*; **AP Authorization Type** = *CAPWAP DTLS*.
4) **Save & Apply**.

---

## 5) Site & Policy/RF Tags
1) **Policy Tag:** Map your WLANs → Policy Profiles as usual (central or local switching as required).  
   **GUI:** `Configuration → Tags & Profiles → Tags → Policy → + Add`
2) **RF Tag:** Reference your RF Profiles (2.4/5/6 GHz).  
   **GUI:** `Configuration → Tags & Profiles → Tags → RF → + Add`
3) **Site Tag:**  
   **GUI:** `Configuration → Tags & Profiles → Tags → Site → + Add`
   - **AP Join Profile** = `APJ-MESH`.
   - **Flex+Bridge?** Uncheck **Enable Local Site** to expose **Flex Profile** dropdown, then attach the site’s **Flex Profile**.
   - **Save & Apply**.

---

## 6) Stage APs and Convert to Mesh
**GUI:** `Configuration → Wireless → Access Points`
1) **Join APs in Local mode first** (recommended). Verify they are listed under **All Access Points**.
2) **Assign Tags:** Open AP → **Tags** tab → set **Policy Tag**, **RF Tag**, **Site Tag** (the one with APJ‑MESH).
3) **Convert Mode:** On the AP (General tab), set **AP Mode = Bridge** (Mesh) → **Update**. AP reboots and returns as **Bridge** (or **Flex+Bridge** if your Site Tag includes Flex).
4) **Assign Role:** After it returns, open the AP → **Mesh tab** → set **Role** to **RAP** (for the wired uplink AP) or **MAP** (for wireless children).

**Switch (RAP) example:**
```
interface GigabitEthernet1/0/10
 description RAP uplink
 switchport mode trunk
 switchport trunk native vlan <AP_MGMT_VLAN>
 switchport trunk allowed vlan <AP_MGMT_VLAN>,<CLIENT/BRIDGE_VLANs>
 spanning-tree portfast trunk
 power inline auto
```

---

## 7) Validate & Monitor
**GUI:**
- `Monitoring → Wireless → AP Statistics → (select AP) → Mesh tab` — view backhaul, parent, link stats.
- `Monitoring → Wireless → Mesh → AP` — see **mesh tree** and global mesh stats.
- `Monitoring → Wireless → Mesh → Convergence` — mesh convergence view.

**CLI (WLC):**
```
show ap config general | include Backhaul
show ap tag summary
show ap name <AP_NAME> mesh neighbors
show wireless tag site detail <SITE_TAG>
show wireless profile mesh summary
```

---

## 8) Troubleshooting
- **AP doesn’t join in Bridge:** Ensure AAA **Device Authentication** MAC is present (no separators). If AP starts in Bridge, use the **default method lists** temporarily so it can join and receive tags.
- **Auth errors:** Confirm **Authentication/Authorization** method list names match in **Mesh Profile**; verify **EAP‑FAST** settings.
- **No backhaul:** Check **BGN** match, **backhaul band/channel**, RF profile settings; verify RAP is wired and reachable.
- **Logs:** `Troubleshooting → Radioactive Trace` → add AP MAC, **Start**, then **Generate** to download join logs.
- **Flex+Bridge data path:** If using Flex, confirm **Flex Profile** is attached to the **Site Tag** and local VLAN/DHCP exist at the site.

---

## 9) Quick Build Checklist
- [ ] AAA Device Authentication (AP MACs added)  
- [ ] AAA Method Lists (dot1x + credential-download)  
- [ ] Mesh Global defaults set  
- [ ] Mesh Profile (BGN, backhaul, auth/authorization, options)  
- [ ] AP Join Profile (bind Mesh Profile; EAP‑FAST/CAPWAP DTLS)  
- [ ] Policy Tag / RF Tag created  
- [ ] Site Tag (APJ‑MESH; Flex Profile if Flex+Bridge)  
- [ ] APs tagged; RAP/MAP roles assigned; RAP switchport trunked  
- [ ] Validate in Monitoring (Mesh tree, Mesh tab); CLI checks

---

### Notes
- GUI labels can vary slightly by IOS‑XE release (17.3+). Use the nearest **Mesh**, **Tags & Profiles**, **AAA** menus if names differ.
