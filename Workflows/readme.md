# Cisco Workflows Beginner’s Guide
_Last updated: 2025-08-21_

---

## 1. Introduction

Cisco **Workflows** is a cloud‑based, low/no‑code automation platform embedded in the Cisco Meraki Dashboard (Early Access 2025). It lets you design, run, and govern automations across Cisco products (Meraki, Catalyst, SD‑WAN, ACI, ISE) and external systems (ServiceNow, Infoblox, REST APIs, SSH, Ansible).

This guide is designed for beginners: we’ll break down the interface, share best practices, and walk through a few **simple examples**.

---

## 2. Interface Breakdown

When you open **Automation → Workflows → New Workflow**, you’ll see the **Workflow Editor**. Here’s how it’s structured:

### a. Canvas
- The central area where you **drag and drop activities** (building blocks).
- Workflows flow left → right, starting with **Start** and ending with **End** nodes.

### b. Activities Panel
- Located on the left.
- Contains **Cisco activities** (e.g., Meraki → List Networks), **Generic adapters** (REST, SSH, PowerShell), and **Logic blocks** (If, For Each, Script, Approval, etc.).

### c. Variables Panel
- Define **Inputs** (provided when workflow runs), **Locals** (used internally), and **Secrets** (API keys, passwords).
- Inputs can be marked **required** and surfaced in the run wizard.

### d. Properties Panel
- When you click an activity, its **properties** appear here.
- Example: For a REST activity → URL, Method, Headers, Body.

### e. Toolbar
- **Save, Run, Debug, Publish** options.
- **Undo/Redo** and zoom controls for the canvas.

### f. Run Logs & History
- After running a workflow, logs appear at the bottom.
- You can inspect **inputs, outputs, errors, and approvals** for each activity.

---

## 3. Best Practices

### a. Design
- **Start simple**: one task, then expand.
- Use **Inputs** for anything that may change (e.g., network name, VLAN ID).
- Normalize naming: `camelCase` for variables.

### b. Security
- Store credentials in **Secrets/Targets** (never in plain text).
- Use **HMAC or token authentication** for webhooks.
- Restrict workflow editing with **RBAC**.

### c. Reliability
- Always include a **dry‑run mode**.
- Add **approval gates** for production‑impacting actions.
- Use **Try/Retry** on API calls that may rate‑limit.

### d. Reusability
- Break down large automations into **smaller sub‑workflows**.
- Document variables and outputs clearly.
- Version workflows; keep copies in GitHub.

### e. Observability
- Emit **structured logs** (`JSON.stringify`) instead of plain text.
- Include correlation IDs (e.g., change ticket #) in logs and outputs.

---

## 4. Simple Examples

### Example 1 — Hello World (Log a Message)
1. Create a new workflow → name it `Hello_World`.
2. Drag a **Log Activity** to the canvas.
3. Set Message: `"Hello Cisco Workflows!"`
4. Connect Start → Log → End.
5. Run it. Output shows the log entry.

---

### Example 2 — List Networks in a Meraki Org
1. Inputs: `orgId` (string).
2. Drag **Meraki → List Networks**.
   - Input: `organizationId = {orgId}`
3. Drag **Log Activity** → set message to:  
   ```
   Networks: {JSON.stringify(activity.listNetworks.output)}
   ```
4. Run workflow with a valid `orgId`.  
   You’ll see all networks printed.

---

### Example 3 — Webhook Triggered Logger
1. Inputs: `rawBody` (string).
2. Create a **Webhook Automation Rule** → map body → `rawBody`.
3. In workflow, add a **Script Activity**:  
   ```js
   try {
     payload = JSON.parse(rawBody);
     console.log("Received payload:", payload);
   } catch (e) {
     throw new Error("Invalid JSON");
   }
   ```
4. Add a **Log Activity**:  
   `Message: Webhook received for device ${payload.device}`

---

### Example 4 — Simple Switchport Config (Meraki)
1. Inputs: `serial` (string), `portId` (string), `vlanId` (integer).
2. Add a **REST → PUT** activity:
   - URL:  
     ```
     https://api.meraki.com/api/v1/devices/{serial}/switch/ports/{portId}
     ```
   - Headers:  
     ```
     Authorization: Bearer {merakiApiKey}
     Content-Type: application/json
     ```
   - Body:  
     ```json
     {
       "type": "access",
       "vlan": {vlanId}
     }
     ```
3. Run workflow → it updates the switchport VLAN.

---

## 5. Next Steps

- Explore the **Cisco Workflow Exchange** for prebuilt automations.
- Practice with **Meraki** APIs (SSIDs, VLANs, Alerts).
- Add **Approvals** and **Dry‑run** to your simple workflows.
- Try connecting to third‑party systems (e.g., ServiceNow, Slack).

---

## 6. Quick Reference

- **Activities** = building blocks (Meraki, REST, SSH, logic).  
- **Variables** = inputs, locals, secrets.  
- **Rules** = triggers (Webhook, Schedule, Manual).  
- **Best Practices** = security, dry‑run, approvals, logs.  

---

Happy automating 🚀
