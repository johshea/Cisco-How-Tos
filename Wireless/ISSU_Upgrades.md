 Catalyst 9800 ISSU Upgrade — Step‑by‑Step (CLI)

**Last updated:** 2025-08-21

This guide walks you through an **In‑Service Software Upgrade (ISSU)** on a pair of Cisco Catalyst 9800 wireless controllers running in **HA SSO**. It focuses on the **CLI workflow** with optional AP image predownload to minimize client impact.

> **What ISSU does (at a glance)**  
> 1) Stage the new image on the active WLC and sync it to standby → 2) (Recommended) Pre‑download AP images → 3) Reload **standby** to the new version → 4) SSO switchover → 5) Reload former‑active → 6) Stagger AP reboots → 7) Commit.

---

## 0) Prerequisites & Pre‑Checks

- **HA SSO pair** (active/standby) and **INSTALL** mode on both.
- **ISSU‑supported path** between your current and target IOS‑XE releases (see Cisco’s ISSU matrix for 9800).  
- **Sufficient bootflash space** on the active WLC (use `install remove inactive` if needed).
- **No config changes** during the ISSU window.
- **Maintenance window** recommended; ensure adequate coverage so staggered AP reloads won’t impact service.

**Verify state & mode**
```console
show install summary
show version | i Installation mode
show boot
show redundancy
show chassis rmi
show issu state detail
```
Healthy baseline examples:
- `Installation mode is INSTALL`
- `BOOT variable = bootflash:packages.conf`
- Redundancy shows **Operating Redundancy Mode = sso** (active / standby-hot)
- `show install summary` shows a single **C (Activated & Committed)** image
- `show issu state detail` shows **No ISSU operation is in progress**

**Save & backup**
```console
write memory
show tech-support | redirect bootflash:tech_before_issu.txt
```

---

## 1) Stage the Target Image

Copy the target image (e.g., `C9800-CL-universalk9.17.15.02.SPA.bin`) to **active** WLC `bootflash:` via SCP/HTTPS/USB, then:
```console
verify /md5 bootflash:C9800-CL-universalk9.17.15.02.SPA.bin
install add file bootflash:C9800-CL-universalk9.17.15.02.SPA.bin
```
This expands packages and (for ISSU) syncs them to the standby automatically.

**Optional cleanup if space is tight:**
```console
install remove inactive
```

---

## 2) (Recommended) Pre‑download AP Images (faster AP cycles)

Clear prior stats (helps visibility), trigger predownload, and monitor:
```console
clear ap predownload statistics
ap image predownload
show ap image
show ap name <AP-NAME> image
```
**Per‑site (primary AP) predownload (17.13+ and Efficient Image Upgrade):**
```console
ap image predownload site-tag <SITE-TAG> start
show ap primary list
show ap image
```
**Optional (only APs with completed predownload):**
```console
ap image swap completed
```
> Tip: Predownload can be initiated well ahead of the window. The **swap** happens during activation (ISSU) or when you explicitly run `ap image swap`.*

---

## 3) Activate ISSU

Kick off the ISSU orchestration. You may set/extend the auto‑rollback timer (30–1200 minutes). Example uses 360 minutes:
```console
install activate issu auto-abort-timer 360
```
**What happens:**  
- Standby reloads to the new image.  
- SSO forms across mixed versions (V1 active / V2 standby).  
- Switchover occurs → V2 becomes active.  
- Former‑active reloads to V2.  
- APs are then upgraded in staggered waves.

**Monitor progress:**
```console
show chassis rmi
show redundancy
show issu state detail
show ap uptime
show logging | include UPGRADE|ISSU|redundancy
```
Wait until both controllers are healthy on **V2** and AP staggered upgrade is complete.

---

## 4) (Optional) Stop Auto‑Abort Timer

If you need more time for validation before commit:
```console
install auto-abort-timer stop
```

---

## 5) Commit (Make Persistent)

Once validation is complete, **commit** to clear rollback timer and make the image persistent across reloads:
```console
install commit
show install summary
```
Expected: the new image shows **C (Activated & Committed)**.

---

## 6) Post‑Checks

```console
show redundancy
show version | i Version|uptime
show ap summary
show wireless client summary
show wlan summary
show issu state detail
show install summary
```
**Backup after:**  
```console
show tech-support | redirect bootflash:tech_after_issu.txt
```

---

## Rollback & Abort Options

- **Abort before commit (ISSU rollback of controllers & AP path):**
```console
install abort issu
```
- **After commit:** use normal install rollback (this is **not** an ISSU rollback and implies downtime):
```console
show install rollback
install rollback to id <ID>
```

---

## Example “Single‑Shot” ISSU Command Block

> Run interactively and watch outputs; don’t paste blindly.

```console
! Pre-checks
show redundancy
show install summary
show issu state detail

! Stage image
verify /md5 bootflash:C9800-CL-universalk9.17.15.02.SPA.bin
install add file bootflash:C9800-CL-universalk9.17.15.02.SPA.bin

! AP predownload (recommended)
clear ap predownload statistics
ap image predownload
show ap image

! Activate ISSU (sets 6-hour auto-rollback window)
install activate issu auto-abort-timer 360

! Monitor until both WLCs are on V2 and AP upgrades complete
show redundancy
show chassis rmi
show ap uptime

! Commit when satisfied
install commit
show install summary
```

---

## Notes, Tips & Gotchas

- **Supported paths only:** ISSU is supported **within a major train** (e.g., 17.x→17.x) and **not** between trains (e.g., 17.x→18.x). Always validate with the current **ISSU matrix** and **release notes**.
- **Install mode only:** ISSU requires **INSTALL** mode (not BUNDLE). Ensure both WLCs boot from `packages.conf`.
- **No changes during ISSU:** Avoid configuration changes while ISSU runs.
- **APs over WAN:** Use **AP image predownload** (and where available, **Efficient Image Upgrade / site‑tag predownload**) to shorten the total window over high‑latency links.
- **Auto‑abort timer:** After activation, a rollback timer is active until you **commit**. You can `install auto-abort-timer stop` if you need more time.
- **Space cleanup:** `install remove inactive` can reclaim space before staging.
- **Troubleshooting:** Check `show issu state detail`, redundancy, AP image status, logs; if an issue occurs **before commit**, `install abort issu` reverts.

---

## References

- **Upgrade Catalyst 9800 WLC HA SSO Using ISSU (CLI workflow, prechecks, timers, abort/commit):**  
  https://www.cisco.com/c/en/us/support/docs/wireless/catalyst-9800-series-wireless-controllers/222405-upgrade-catalyst-9800-wlc-ha-sso-using-i.html

- **In‑Service Software Upgrade (ISSU) Tech Note & Matrix (Updated 2025):**  
  https://www.cisco.com/c/en/us/td/docs/wireless/controller/9800/tech-notes/b_issu_9800.pdf  
  Matrix link inside the PDF points to the latest **ISSU support table**.

- **AP Image Predownload (CLI & site‑tag variants / Efficient Image Upgrade):**  
  https://www.cisco.com/c/en/us/td/docs/wireless/controller/9800/17-14/config-guide/b_wl_17_14_cg/m_predwnld_image_ap_ewlc.html  
  https://www.cisco.com/c/en/us/td/docs/wireless/controller/9800/17-13/config-guide/b_wl_17_13_cg/m_eff_image_upgrade_ewlc.html

- **Monitoring ISSU:**  
  `show issu state detail` usage and states are documented across the configuration guides for your target release.

---

### Change Log (yours)
- [ ] Current → Target version(s):
- [ ] Window & success criteria:
- [ ] Validation checklist complete:
- [ ] Commit complete:


---

## ISSU via Web GUI (Cisco Catalyst 9800)

While the CLI is most common, the 9800 WLC also supports **ISSU through the web interface**.

### 1) Access the Upgrade Menu
- Log in to the **Cisco Catalyst 9800 Web UI** (HTTPS).  
- Navigate to:  
  **Administration → Software Management → ISSU**

### 2) Pre-Checks
- Confirm **HA SSO** state is healthy (Active / Standby-Hot).  
- Ensure both controllers are in **INSTALL mode**.  
- Verify sufficient **bootflash space**.

### 3) Upload and Stage Image
- In the ISSU page, choose **Add Image**.  
- Browse to the image file (`.bin`) or provide a remote server path (SCP/HTTP/FTP).  
- The controller will unpack and sync the image to the standby.

### 4) (Optional) Pre-download AP Images
- Go to **Wireless → Access Points → Predownload**.  
- Initiate predownload for all or per-site APs.  
- Monitor until status shows “Completed” before proceeding.

### 5) Activate ISSU
- In **Administration → Software Management → ISSU**, click **Activate ISSU**.  
- You can configure the **Auto Abort Timer** (default 30 minutes, adjustable up to 1200 minutes).  
- The system automatically:  
  1. Reloads the standby into the new image.  
  2. Performs an SSO switchover.  
  3. Reloads the former-active controller.  
  4. Orchestrates staggered AP upgrades.

### 6) Monitor
- The GUI provides live status on:  
  - Controller upgrade progress.  
  - Redundancy state.  
  - AP upgrade status.  
- You can also validate under:  
  **Monitoring → System → Redundancy / Access Points**.

### 7) Commit
- Once satisfied, return to:  
  **Administration → Software Management → ISSU**  
  and click **Commit**.  
- This makes the new image persistent and clears the rollback timer.

### 8) Rollback (Before Commit)
- If issues are found before committing, use **Abort ISSU** in the GUI to revert.

---



---

# Catalyst 9800 ISSU Upgrade — Step-by-Step (GUI)

_Added: 2025-08-21_

The following outlines the **WebUI** workflow for performing an ISSU on an HA SSO pair, mirroring the CLI steps in the first half of this guide. Exact menu names can vary slightly by release, but the flow is consistent.

## 0) Prerequisites & Health (GUI)
1. **Dashboard → Monitoring → Redundancy** (or **Monitoring → High Availability**): Verify **SSO** with one **Active** and one **Standby Hot**.
2. **Administration → Software Management → Install/Upgrade**: Confirm **Install Mode** (packages) and current version.
3. **Administration → Software Management → Image Repository**: Check available storage; remove inactive packages if space is low.
4. **Administration → Configuration → Save/Archive** (or **Save Configuration**): Save running configuration.

## 1) Stage the Target Image (GUI)
1. Go to **Administration → Software Management → Image Repository**.
2. Click **Add** (or **Upload**) and select the target `C9800-*.bin` image to place it into the repository on the **Active** WLC.
3. After upload completes, the controller automatically **extracts** the image into packages (equivalent to `install add file ...`) and syncs to the **Standby**.

## 2) (Recommended) AP Image Predownload (GUI)
**Global predownload:**
1. Navigate to **Configuration → Wireless → Access Points → Global Configuration**.
2. Find **Image Predownload** and click **Start** (or **Initiate**).
3. Monitor progress in **Monitoring → Wireless → AP Image** or via the **Access Points** table (columns for image state).

**Site-Tag / Efficient Image Upgrade (if supported by your release):**
1. Go to **Configuration → Tags & Profiles → Tags → Site**.
2. Select a **Site Tag**, then choose **Image Predownload** → **Start** (for that site). This lets you stage AP images per site/wave.

> Tip: You can initiate predownload well before the maintenance window. The actual **swap** happens during activation or when explicitly triggered.

## 3) Activate ISSU (GUI)
1. Open **Administration → Software Management → Install/Upgrade**.
2. Select the **staged version** in the package list.
3. Click **Activate ISSU** (or **Activate** → choose **ISSU** mode).
4. Set the **Auto-Abort Timer** (e.g., 360 minutes) and confirm.
   - Standby reloads to target, mixed-mode SSO forms, then an **SSO switchover** is performed.
   - The former-active reloads to the target version.
   - APs upgrade in **staggered waves** (leveraging predownload where applicable).

**Monitor during activation:**
- **Monitoring → Redundancy** (roles/state/uptime).
- **Monitoring → Software Management / Jobs** (task/log view).
- **Monitoring → Wireless → Access Points / Clients** (AP/client health).
- **Monitoring → Event Viewer / Syslog** for ISSU messages.

## 4) (Optional) Stop Auto-Abort Timer (GUI)
- If validation requires more time before commit, go to **Administration → Software Management → Install/Upgrade** and **Stop Auto-Abort Timer**.

## 5) Commit (GUI)
- Once satisfied with validation and AP/client health, return to **Administration → Software Management → Install/Upgrade** and click **Commit** to finalize and clear the rollback window.

## 6) Post-Checks (GUI)
- **Monitoring → Redundancy**: Active/Standby both on **target version** and **SSO** healthy.
- **Monitoring → Software Management → Image Repository / Packages**: Target image shows as **Activated & Committed**.
- **Monitoring → Wireless → Access Points / Clients / WLANs**: Normal counts and status.
- (Optional) **Administration → Configuration → Save/Archive** and capture a **tech-support** from **Troubleshooting**.

## Rollback / Abort (GUI)
- **Before Commit:** In **Administration → Software Management → Install/Upgrade**, choose **Abort ISSU** to revert.
- **After Commit:** Use normal **Rollback** to a prior install ID (this is not ISSU and may incur downtime).

### Notes (GUI)
- ISSU requires **INSTALL mode** and a **supported upgrade path** (within the same major train).
- Avoid config changes during ISSU.
- For APs over WAN, use **Predownload** and **Site-Tag** waves to minimize impact.


---

# Screenshot Callouts & Quick Checklist

## Screenshot Callouts (GUI Navigation)

These callouts describe where you would capture or reference screenshots in a runbook or change ticket.  
Replace each with an actual screenshot from your environment if documenting.

1. **Dashboard → Monitoring → Redundancy**
   - Screenshot: Show Active/Standby roles (SSO healthy).

2. **Administration → Software Management → Image Repository**
   - Screenshot: Target image uploaded and extracted.

3. **Administration → Software Management → Install/Upgrade**
   - Screenshot: List of available packages before activation.

4. **Configuration → Wireless → Access Points → Global Configuration**
   - Screenshot: Predownload controls (start/stop, progress).

5. **Configuration → Tags & Profiles → Site Tags**
   - Screenshot: Site-tag based image predownload (if supported).

6. **Administration → Software Management → Install/Upgrade (Activate ISSU Dialog)**
   - Screenshot: Activate ISSU window with Auto-Abort Timer selection.

7. **Monitoring → Software Management → Jobs / Logs**
   - Screenshot: Progress of activation steps (standby reload, switchover, AP waves).

8. **Administration → Software Management → Install/Upgrade (Commit)**
   - Screenshot: Commit button after validation.

9. **Monitoring → Wireless → Access Points**
   - Screenshot: APs on new code, normal status.

10. **Monitoring → Wireless → Clients / WLANs**
    - Screenshot: Client count recovered and WLANs operational.

---

## Quick ISSU Upgrade Checklist (Change Ticket Template)

**Pre-Upgrade**
- [ ] Verify redundancy (SSO active/standby healthy).
- [ ] Confirm INSTALL mode (`packages.conf`).
- [ ] Validate ISSU support (matrix/release notes).
- [ ] Save config and take backup (tech-support).
- [ ] Ensure sufficient space / remove inactive images.

**Stage Image**
- [ ] Upload new `.bin` image to repository.
- [ ] Verify extraction into packages.
- [ ] Sync confirmed to standby.

**AP Image Preparation**
- [ ] Start AP global predownload.
- [ ] (Optional) Site-tag predownload if using Efficient Image Upgrade.
- [ ] Verify APs staged with new image.

**ISSU Activation**
- [ ] Start ISSU from Install/Upgrade menu.
- [ ] Set Auto-Abort Timer (e.g., 360 min).
- [ ] Monitor standby reload → switchover → active reload.
- [ ] Track AP upgrade waves.

**Validation**
- [ ] Confirm both WLCs on target version.
- [ ] Verify SSO healthy.
- [ ] Check APs all joined and clients stable.
- [ ] Validate WLANs and services.

**Commit**
- [ ] Stop auto-abort timer (if extended validation needed).
- [ ] Commit once validation complete.
- [ ] Confirm image shows Activated & Committed.

**Post-Upgrade**
- [ ] Save config.
- [ ] Capture post-upgrade tech-support.
- [ ] Update documentation / inventory.

**Rollback (If Needed)**
- [ ] Before commit: Abort ISSU from Install/Upgrade.
- [ ] After commit: Rollback via install ID (downtime expected).


---

# Blank Screenshot Placeholders

Use these placeholders in the runbook to mark where screenshots should be inserted. Replace each box with an actual screenshot when documenting.

## Screenshot 1: Dashboard → Monitoring → Redundancy

```
[ Insert Screenshot Here ]
```

## Screenshot 2: Administration → Software Management → Image Repository

```
[ Insert Screenshot Here ]
```

## Screenshot 3: Administration → Software Management → Install/Upgrade (Pre-Activation)

```
[ Insert Screenshot Here ]
```

## Screenshot 4: Configuration → Wireless → Access Points → Global Configuration (Predownload)

```
[ Insert Screenshot Here ]
```

## Screenshot 5: Configuration → Tags & Profiles → Site Tags (Predownload per Site)

```
[ Insert Screenshot Here ]
```

## Screenshot 6: Administration → Software Management → Install/Upgrade (Activate ISSU Dialog)

```
[ Insert Screenshot Here ]
```

## Screenshot 7: Monitoring → Software Management → Jobs / Logs

```
[ Insert Screenshot Here ]
```

## Screenshot 8: Administration → Software Management → Install/Upgrade (Commit)

```
[ Insert Screenshot Here ]
```

## Screenshot 9: Monitoring → Wireless → Access Points

```
[ Insert Screenshot Here ]
```

## Screenshot 10: Monitoring → Wireless → Clients / WLANs

```
[ Insert Screenshot Here ]
```
