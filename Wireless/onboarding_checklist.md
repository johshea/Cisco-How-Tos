# Catalyst 9800 AP Onboarding — One‑Pager Checklist

## 1) Pre‑Checks
- [ ] **Time/NTP**: `Administration → Time`
- [ ] **Country Codes**: `Configuration → Wireless → Country`
- [ ] **Interfaces/VLANs**: `Configuration → Interfaces`
- [ ] **AAA Config** (if 802.1X): `Configuration → Security → AAA`
- [ ] **AP Discovery**: DHCP Option 43 / DNS `CISCO-CAPWAP-CONTROLLER` / Static

---

## 2) Build Order

### **Central‑Switched SSID**
1. **WLAN Profile** — `Tags & Profiles → WLANs → +Add`
2. **Policy Profile** — VLAN = central client VLAN; Central Switching **ON**
3. **Policy Tag** — Map WLAN → Policy Profile
4. **RF Profiles + RF Tag**
5. **(Optional) AP Join Profile**
6. **Site Tag** — No Flex Profile
7. **Assign Tags** — Policy / RF / Site

**Switchport**: Access or trunk carrying AP mgmt + WLC VLANs.

---

### **Flex (Local‑Switched) SSID**
1. **WLAN Profile** — `Tags & Profiles → WLANs → +Add`
2. **Policy Profile** — VLAN = branch client VLAN; Central Switching **OFF**
3. **Flex Profile** — Optional Local Auth / VLAN overrides
4. **Policy Tag** — Map WLAN → Policy Profile
5. **RF Profiles + RF Tag**
6. **Site Tag** — Attach Flex Profile
7. **Assign Tags** — Policy / RF / Site

**Switchport**: Trunk; Native VLAN = AP mgmt; allow branch client VLAN(s). DHCP local.

---

## 3) Tag Assignments
- **Policy Tag**: WLAN → Policy Profile mapping
- **RF Tag**: RF Profiles per band
- **Site Tag**: AP Join Profile; Flex Profile (if Flex)

**GUI path**: `Configuration → Wireless → Access Points → (AP) → Tags`

---

## 4) Validation (CLI)
```
show wlan summary
show wireless tag policy detail <policy_tag>
show wireless tag site detail <site_tag>
show ap tag summary
show wireless client summary
```

---

## 5) Troubleshooting Quick Hits
- **No SSID**: Wrong/missing Policy Tag mapping.
- **DHCP Failures**:
  - Central: VLAN not on WLC or missing helper.
  - Flex: Trunk missing VLAN / no local DHCP.
- **Auth Failures**: AAA config / RADIUS keys.
- **RF Issues**: Wrong RF Tag/profile.

---

### Quick GUI Map
- **WLANs**: Tags & Profiles → WLANs
- **Policy Profiles**: Tags & Profiles → Policy → Policy Profiles
- **Policy Tags**: Tags & Profiles → Tags → Policy
- **RF Profiles**: Tags & Profiles → RF Profiles
- **RF Tags**: Tags & Profiles → Tags → RF
- **Flex Profiles**: Tags & Profiles → Flex Profiles
- **Site Tags**: Tags & Profiles → Tags → Site
- **AP Join Profiles**: Tags & Profiles → AP Join
- **AP Inventory/Tagging**: Wireless → Access Points
