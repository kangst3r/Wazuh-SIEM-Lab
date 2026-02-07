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
  * This was an error I encountered personally and had to downgrade from Ubuntu 24.04 to 22.04. Using an unsupported version of Ubuntu resulted in the inability to download the Wazuh Dashboard.
<img src="screenshots/failed to install dashboard.png" width = "800" >

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

<img src="screenshots/incompatible version.png" width = "800" >
After realising this, I could either downgrade my Agent or upgrade my Manager, I decided to downgrade my Agent as it was the most straightforward option for this case. Of course in reality in a professional setting I would upgrade to the latest supported version, so as to ensure the latest patches are installed.

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
You would now be able to view the Agent via the Wazuh Dashboard on the Manager's system.

---

## File Integrity Monitoring (FIM) on Windows

After successfully setting up my Agent and Manager systems for my Wazuh SIEM Lab home lab, I decided to dive deeper into what could potentially be done within Wazuh.
This section documents how I explored Wazuh's **File Integrity Monitoring (FIM)** and how I enabled it on the Windows Wazuh agent using **Syscheck**, allowing real-time detection of file creation, modification, and deletion events.

---

### Objective

During this project, I hoped to achieve the folowing:
- Monitor a specific Windows directory in **real time**
- Generate alerts in the Wazuh dashboard when files are created, modified, or deleted
- Validate end-to-end visibility from **agent → manager → dashboard**

---

## Agent Configuration (Windows)

### Step 1: Locate the agent configuration file

Open the following file on the Windows agent using Notepad **as Administrator**:
```bash
C:\Program Files (x86)\ossec-agent\ossec.conf
```

---

### Step 2: Enable real-time directory monitoring

Inside the `<syscheck>` block, add the following entry:

```bash
<directories realtime="yes">C:\Users\abc\Test</directories>
```

Replace `C:\Users\abc\Test` with the location of the folder you want to monitor on your Agent.
This enables **real-time monitoring** for the specified directory entered.

---

### Step 3: Save and restart the Windows agent

After saving the configuration, remember to restart the agent to apply changes:

```powershell
Restart-Service WazuhSvc
```

---

## Verifying File Integrity Monitoring

### Step 4: Confirm agent status

In the Wazuh Dashboard:
- Navigate to **Agents**
- Ensure the Windows agent status is **Active**

<img src="screenshots/agent active.png" width = "600" >

---

### Step 5: Trigger file system events

In the monitored directory, example case (`C:\Users\abc\Test`), do any of the following:
- Create a new file
- Modify an existing file
- Delete a file

These actions should immediately generate Syscheck events in the Wazuh Dashboard (Manager).

<img src="screenshots/syscheck event.png" width = "600" >

---

### Step 6: Validate alerts in the dashboard

In the Wazuh Dashboard:
- Navigate to **Integrity Monitoring**
- Confirm that alerts corresponding to file changes appear

<img src="screenshots/FIM alerts.png" width = "600" >
<img src="screenshots/FIM event details.png" width = "600" >

---

## Conclusion

Building this lab gave me hands-on experience with deploying and operating a real-world SIEM rather than just reading about one. I learned how OS compatibility, version alignment, networking, and agent–manager communication affect system reliability, and how to troubleshoot issues using logs instead of trial and error. Configuring features such as File Integrity Monitoring also helped me understand how SIEMs are actually used in practice. This experience helped me understand more about cybersecurity and SOC roles, where diagnosing broken pipelines, validating security controls, and understanding how alerts are generated are just as important as detecting threats themselves.
