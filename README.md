# Wazuh SIEM Lab (Ubuntu 22.04 + Windows Agent)

This repository documents my end‑to‑end setup of a Wazuh SIEM lab, including all debugging steps, common pitfalls, and fixes encountered during installation.

The goal of this project was to:

* Deploy a **Wazuh Manager + Dashboard** on an Ubuntu VM
* Enrol a **Windows machine as a Wazuh Agent**
* Understand potential real‑world issues I could run into when deploying endpoint monitoring systems.
This repo is written as learning documentation.

---

## Architecture Overview

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
* Networking: VMnet8 (NAT)
* Manager IP: `192.168.x.x` (lab environment)

---

## VM Setup (Manager)

### Ubuntu Version

* **Ubuntu 22.04.5 LTS (Jammy Jellyfish)**
* Ubuntu 24.04 LTS is **not supported** by Wazuh (dashboard fails)
  * This was an error I encountered personally and had to downgrade from Ubuntu 24.04 to 22.04
  * attach screenshot

### VM Resources

* **RAM:** 8 GB recommended
* **CPU:** 2 cores
* **Disk:** 40 GB
* **Network:** NAT (VMnet8)

---

## Installing Wazuh Manager + Dashboard

```bash
curl -s https://packages.wazuh.com/4.12/wazuh-install.sh | sudo bash -a
```
This command fetches the Wazuh 4.12 installer and executes it with administrative privileges to install and configure the Wazuh manager, indexer (OpenSearch), and web dashboard.

After a successful installation, it should create:

`wazuh-manager` <br>
`wazuh-indexer` <br>
`wazuh-dashboard` <br>

To access your Wazuh Dashboard, open Firefox on the Manager device and enter this into the URL box:

```
https://<manager-ip> 
```
192.168.x.x for my case.

Expect self‑signed certificate warnings. Just press advanced options and click proceed anyways.

---

## Verifying Manager Services

Execute the following commands on your Wazuh manager device:
```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-dashboard
sudo ss -tulpn | grep 443
```
You should get the following output respectively:

`Loaded: loaded (/lib/systemd/system/wazuh-manager.service; enabled)
Active: active (running)` <br>
`
Loaded: loaded (/lib/systemd/system/wazuh-dashboard.service; enabled)
Active: active (running)
` <br>
`
LISTEN 0      511        0.0.0.0:443      0.0.0.0:*    users:(("node",pid=1234,fd=20))
` <br>

This indicates that after installation, the Wazuh manager and dashboard services were verified to be running successfully. The dashboard was also confirmed to be listening on port 443, indicating that the web interface was accessible over HTTPS.

---

## Windows Agent Setup

### Important Version Rule

**Wazuh agent and manager versions must match !!!**

* Manager: **4.12.x**
* Agent: **must also be 4.12.x**

This was another issue that took me awhile to resolve, despite being connected to the Manager's IP, I simply could not see my Agent's device on my Wazuh Dashboard.
I decided to check the manager logs live with this command executed into the Manager device:
```bash
sudo tail -f /var/ossec/logs/ossec.log
```
and the following log entry filled the terminal
```
ERROR: Incompatible version for new agent
```
*insert screenshot
After realising this, I could either downgrade my Agent or upgrade my Manager, I decided to downgrade my Agent as it was the most straightforward option for this case. Of course in reality in a professional setting i would upgrade so as to ensure the latest patches are installed for everything.

---

### Installing the Windows Agent

1. Download **Wazuh Agent 4.12.x (Windows)** from:

   * [https://documentation.wazuh.com/4.12/installation-guide/wazuh-agent/wazuh-agent-package-windows.html](https://documentation.wazuh.com/4.12/installation-guide/wazuh-agent/wazuh-agent-package-windows.html)
2. Run the `.msi` installer
3. During install:

   * Manager IP: `192.168.x.x`
   * Agent name: `WINDOWS-AGENT`

---

## Agent Registration (Manager Side)

Run the following command into the terminal to manager your agents:
```bash
sudo /var/ossec/bin/manage_agents
```

After which follow the steps:

1. `A` – To add agent
2. Enter name and IP
3. `E` – To extract key
4. Copy authentication key

---

## Agent Enrolment (Windows Side)

1. Open **Wazuh Agent Manager**
2. Paste authentication key
3. Save
4. Restart agent

Restart the agent with the following command:
```powershell
Restart-Service WazuhSvc
```

---

## Connectivity Checks
After following these steps, the Agent should be successfully connected to the Manager, execute the following commands to check if there is a successful connection.

### From Windows → Manager

```powershell
Test-NetConnection 192.168.x.x -Port 1514
```

Must return:

`
TcpTestSucceeded : True
`<br>

### On Manager

Execute the following command:
```bash
sudo ss -tulpn | grep 1514
```
Must return: <br>
`
LISTEN 0      511        0.0.0.0:443      0.0.0.0:*    users:(("node",pid=1234,fd=20))
` <br>

---

## Confirming Agent Connection

```bash
sudo /var/ossec/bin/agent_control -lc
```

Expected output:

`
ID: 003, WINDOWS-AGENT, Active
`<br>

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
