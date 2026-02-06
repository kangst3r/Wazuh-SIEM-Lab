# 🛡️ Wazuh SIEM Lab (Ubuntu 22.04 + Windows Agent)

This repository documents my **end‑to‑end setup of a Wazuh SIEM lab**, including all debugging steps, common pitfalls, and fixes encountered during installation.

The goal of this project was to:

* Deploy a **Wazuh Manager + Dashboard** on an Ubuntu VM
* Enrol a **Windows machine as a Wazuh Agent**
* Understand real‑world issues such as OS compatibility, networking, agent registration, and version mismatches

This repo is written as **learning documentation**.

---

## 🧱 Architecture Overview

```
[ Windows PC ]        (Wazuh Agent 4.12.x)
      |
      |  TCP 1514 / 1515
      v
[ Ubuntu 22.04 VM ]   (Wazuh Manager + Indexer + Dashboard 4.12.x)
      |
      |  HTTPS 443
      v
[ Wazuh Dashboard (Browser) ]
```

* Hypervisor: VMware
* Networking: **VMnet8 (NAT)**
* Manager IP: `192.168.x.x` (lab environment)

---

## ⚙️ VM Setup (Manager)

### Ubuntu Version

* ✅ **Ubuntu 22.04.5 LTS (Jammy Jellyfish)**
* ❌ Ubuntu 24.04 LTS is **not supported** by Wazuh (dashboard fails)

### VM Resources

* **RAM:** 4 GB minimum (8 GB recommended)
* **CPU:** 2 cores
* **Disk:** 40 GB
* **Network:** NAT (VMnet8)

---

## 📦 Installing Wazuh Manager + Dashboard

```bash
curl -s https://packages.wazuh.com/4.12/wazuh-install.sh | sudo bash -a
```

Successful installation creates:

* `wazuh-manager`
* `wazuh-indexer`
* `wazuh-dashboard`

Dashboard access:

```
https://<manager-ip>
```

Self‑signed certificate warnings are expected.

---

## 🔍 Verifying Manager Services

```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-dashboard
sudo ss -tulpn | grep 443
```

---

## 🖥️ Windows Agent Setup

### Important Version Rule

> ⚠️ **Wazuh agent and manager versions must match**

* Manager: **4.12.x**
* Agent: **must also be 4.12.x**

A newer agent (e.g. 4.14.x) will be rejected with:

```
ERROR: Incompatible version for new agent
```

---

### Installing the Windows Agent

1. Download **Wazuh Agent 4.12.x (Windows)** from:

   * [https://packages.wazuh.com/4.12/windows/](https://packages.wazuh.com/4.12/windows/)
2. Run the `.msi` installer
3. During install:

   * Manager IP: `192.168.x.x`
   * Agent name: `WINDOWS-AGENT`

---

## 🔑 Agent Registration (Manager Side)

```bash
sudo /var/ossec/bin/manage_agents
```

Steps:

1. `A` – Add agent
2. Enter name + IP
3. `E` – Extract key
4. Copy authentication key

---

## 🔑 Agent Enrolment (Windows Side)

1. Open **Wazuh Agent Manager**
2. Paste authentication key
3. Save
4. Restart agent

### Restarting the Windows Agent (recommended)

```powershell
Restart-Service WazuhSvc
```

---

## 🔌 Connectivity Checks

### From Windows → Manager

```powershell
Test-NetConnection 192.168.x.x -Port 1514
```

Must return:

```
TcpTestSucceeded : True
```

### On Manager

```bash
sudo ss -tulpn | grep 1514
```

---

## 📡 Confirming Agent Connection

```bash
sudo /var/ossec/bin/agent_control -lc
```

Expected output:

```
ID: 003, WINDOWS-AGENT, Active
```

---

## 📊 Dashboard Validation

* Navigate to **Agents**
* Set time range to **Last 1 hour**
* Generate Windows activity (log in/out, file changes)

Events may take **1–5 minutes** to appear initially.

---

## 🧨 Common Issues & Fixes

### Dashboard not found

* Cause: Unsupported OS (Ubuntu 24.04)
* Fix: Reinstall on Ubuntu 22.04

### Agent registered but not active

* Cause: Agent service not running or blocked port
* Fix: Restart agent + check port 1514

### Incompatible version error

* Cause: Agent newer than manager
* Fix: Downgrade agent to match manager version

---

## 📁 Repo Structure (Recommended)

```
wazuh-siem-lab/
├── README.md
├── screenshots/
│   ├── install/
│   ├── dashboard/
│   ├── agent/
│   └── errors/
├── notes/
│   └── troubleshooting.md
└── diagrams/
    └── architecture.png
```

### Screenshot tips

* Use **descriptive filenames**
* Group by phase (install, agent, dashboard)
* Reference screenshots directly in README

Example:

```md
![Agent Active](screenshots/agent/agent-active.png)
```

---

## 🧠 Key Takeaways

* OS compatibility matters in security tooling
* Version mismatches are a real‑world failure mode
* Logs are the fastest source of truth
* SIEMs are resource‑intensive by design

---

## 📌 Next Improvements

* Add Linux agent
* Enable Windows Security Event Channel
* Create custom Wazuh rules
* Forward alerts to email / webhook

---

## ✨ Author Notes

This project intentionally documents **mistakes and fixes**, because that is how real infrastructure work happens.

If you are learning SIEMs or SOC tooling, you will encounter these issues too — this repo exists so you don’t have to rediscover them blindly.
