# Enterprise-Endpoint-Monitoring-File-Integrity-Lab-with-Wazuh-SIEM-XDR
Configured Wazuh Manager on Linux and installed Wazuh Agent on Windows 11 Pro Workstation using VMs

Markdown
# Enterprise Endpoint Monitoring & File Integrity Lab with Wazuh SIEM/XDR

## Project Overview
This project demonstrates the deployment of a centralized Security Information and Event Management (SIEM) and Extended Detection and Response (XDR) architecture. The objective of the lab was to establish real-time security telemetry ingestion, centralize event logs, and configure host-based File Integrity Monitoring (FIM). 

The implementation involved provisioning a **Wazuh Manager** on a Linux instance, navigating the cloud management console to generate a deployment profile, troubleshooting network/DNS isolation issues on a **Windows 11 Pro workstation**, and verifying end-to-end alert telemetry on the live dashboard.

---

## Architecture & Environment
* **SIEM/XDR Server (Wazuh Manager):** Linux (Ubuntu 22.04 LTS / Debian)
* **Monitored Endpoint (Wazuh Agent):** Windows 11 Pro Workstation (Virtual Machine)
* **Network Layout:** Multi-host lab network requiring direct DNS resolution over port 53 and telemetry routing over ports 1514 (Agent communication) and 1515 (Enrollment).

---

## Technical Deployment Steps

### Phase 1: Initializing the Wazuh Manager (Linux)
The deployment began by installing and starting the core SIEM infrastructure components (Indexer, Server, and Dashboard) on the central Linux machine.

**Run the automated all-in-one installation assistant:**
```bash
curl -sO [https://packages.wazuh.com/4.x/wazuh-install.sh](https://packages.wazuh.com/4.x/wazuh-install.sh)
sudo bash wazuh-install.sh -a
Verify that the core engine is up and listening:
```

```bash
sudo systemctl status wazuh-manager
```


<img width="656" height="228" alt="wazuh_status" src="https://github.com/user-attachments/assets/6fa23fe1-a933-49dd-be24-9f190c6af083" />

<img width="978" height="731" alt="status_running" src="https://github.com/user-attachments/assets/3446a08b-32fd-45de-b222-228a2a4677cb" />


---

### Phase 2: Generating the Agent Deployment Profile
Before configuring the endpoint, the agent profile was registered through the central management application to obtain the necessary installation arguments.

1. Opened a local web browser and navigated to the Wazuh Dashboard web interface (https://<YOUR_LINUX_MANAGER_IP>).

2. Logged in using the administrator credentials.

3. Navigated to Management -> Agents -> Deploy new agent.

4. Selected Windows as the operating system, specified the Linux Manager's static IP address, and generated the customized PowerShell deployment script block.

<img width="701" height="160" alt="browser_log-in" src="https://github.com/user-attachments/assets/e8dda666-ebcd-41a5-ba8e-c1c8dc4eddb1" />

<img width="834" height="656" alt="log-in_page" src="https://github.com/user-attachments/assets/b0818f51-1487-425d-add6-740e4ee10193" />

<img width="1007" height="108" alt="add_agent" src="https://github.com/user-attachments/assets/d61644cc-4fc0-4915-a28f-8e95bca6e37b" />

<img width="1002" height="552" alt="deploying" src="https://github.com/user-attachments/assets/0ca8664b-945d-4fe0-b086-183189dda78d" />

<img width="976" height="538" alt="deploying2" src="https://github.com/user-attachments/assets/b0d33cbb-93d7-4c9b-9629-2b6d6db0244e" />

<img width="976" height="588" alt="deploying_group" src="https://github.com/user-attachments/assets/143e0457-12a7-48a5-a025-23b0d977c82b" />

<img width="953" height="551" alt="deploying_PS_script" src="https://github.com/user-attachments/assets/75a65f7f-51c8-4d01-8539-68f81e12bf3b" />

---

### Phase 3: Endpoint DNS Troubleshooting & Agent Installation
When transitioning to the Windows 11 Pro workstation to run the deployment script, the automated download command failed due to internal lab network limitations.



#### 1. Troubleshooting the DNS Resolution Failure

The standard Invoke-WebRequest installation command failed initially, throwing a stream of red InvalidOperation exceptions in the terminal because the lab's pre-configured DNS server could not resolve external web hosts (packages.wazuh.com).

To circumvent this, the workstation's network adapter settings were manually updated directly inside an Administrative PowerShell session to point to Google's Public DNS (8.8.8.8):

Identify the active network interface alias

```PowerShell



Get-NetAdapter
```
Set the primary DNS server to bypass the broken lab resolver

```PowerShell



Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses ("8.8.8.8")
```
(Note: Replace "Ethernet" with the specific interface alias returned by your system).

<img width="1003" height="173" alt="invoke_not_resolved" src="https://github.com/user-attachments/assets/823f5dc6-8fdd-4a25-905e-17001c11f6bc" />

<img width="998" height="323" alt="troubleshooting" src="https://github.com/user-attachments/assets/8e08b9d8-dabb-4445-8020-491b674e2278" />

<img width="1013" height="321" alt="DNS_change_package_installed_started_service" src="https://github.com/user-attachments/assets/785134f8-0121-4ff1-8804-7fe5578a1f83" />



---

#### 2. Running the Agent Script

With public name resolution successfully restored, the PowerShell script executed cleanly to download the MSI package, dynamically configure the manager IP address, and start the system service.



```PowerShell



Invoke-WebRequest -Uri [https://packages.wazuh.com/4.x/windows/wazuh-agent-4.x.x-1.msi](https://packages.wazuh.com/4.x/windows/wazuh-agent-4.x.x-1.msi) -OutFile wazuh-agent.msi

msiexec /i wazuh-agent.msi /q WAZUH_MANAGER="<YOUR_LINUX_MANAGER_IP>"

net start wazuh-agent
```

#### 3. Verifying Endpoint Connection Status

To ensure telemetry was flowing, the connection status was verified locally on the endpoint:



```PowerShell



Get-Service -Name "wazuh-agent"
```

<img width="1013" height="321" alt="DNS_change_package_installed_started_service" src="https://github.com/user-attachments/assets/a761e17a-cd46-4bbc-9d90-a48bba76d4c1" />

---


### Phase 4: File Integrity Monitoring (FIM) Configuration

To showcase host-based intrusion detection capabilities, the agent was configured to track file modifications in real time.



#### 1. Opened the local Windows configuration file (C:\Program Files (x86)\ossec-agent\ossec.conf) using an administrative text editor. 


#### 2. Navigated to the <syscheck> configuration block and appended a new directory monitoring definition. The realtime="yes" and report_changes="yes" attributes were specified to ensure instant alert delivery and deep content diff tracking:

*Alternatively, you may use the GUI to open the Wazuh Agent Manager and make the configuration changes by inserting the line of code shown in the image below. Be sure to save the file before closing.*

<img width="645" height="319" alt="manage_agent_on_windows" src="https://github.com/user-attachments/assets/6061f9be-f120-45ea-9d62-94f5952d664c" />

<img width="332" height="293" alt="Agent_Manager" src="https://github.com/user-attachments/assets/31e48138-e811-4416-95f0-65325f0b4267" />




```XML



<syscheck>

<frequency>300</frequency>

<directories realtime="yes" report_changes="yes">C:\Users\Public\Documents\SensitiveData</directories></syscheck>
```
Restarted the endpoint agent via PowerShell to load the new policy:



```PowerShell



Restart-Service -Name "wazuh-agent"
```


<img width="1021" height="682" alt="FIM_insert" src="https://github.com/user-attachments/assets/caf98bde-a286-41ff-9777-42eb0f832efa" />


---

### Phase 5: Security Event Verification & FIM Logs

To validate end-to-end functionality, a lifecycle file integrity test was performed directly on the Windows 11 workstation.



#### 1. The Test: A text file was created inside C:\Users\Public\Documents\SensitiveData, modified with sample text, and then deleted.

#### 2. The Result: Navigating back to the Wazuh Dashboard -> Modules -> File Integrity Monitoring -> Events page confirmed that the system intercepted and visualized the actions seamlessly.

---

The web console successfully logged the entire sequence across multiple rule levels:


- Rule 554: File added to the system (Creation event)

- Rule 550: Integrity checksum changed (Content modification event)

- Rule 553: File deleted from the system (Destruction event)

<img width="1001" height="600" alt="wazuh_total_agents" src="https://github.com/user-attachments/assets/6e8bb933-be6a-4b6b-a84b-0db80863e959" />

<img width="1017" height="602" alt="Integrity_check_module" src="https://github.com/user-attachments/assets/a43b267e-e9f2-4644-96d8-9a50a376ddf1" />

<img width="1017" height="605" alt="Integrity_check_module_events" src="https://github.com/user-attachments/assets/8daafae3-e519-4d57-bd04-7652231c6265" />

<img width="1023" height="602" alt="Tracked_changes" src="https://github.com/user-attachments/assets/b25b61c9-7703-4ea3-b84a-2016d3349c34" />

---

## Core Security Capabilities Demonstrated

- DNS & Network Remediation: Diagnosing internal network blocks and utilizing administrative scripting interfaces to alter baseline IP stacks.

- SIEM Ingestion: Registering standalone operating system endpoints into a unified central management console.

- File Auditing: Implementing strict configuration parameters to audit runtime directory updates and preserve forensic chains of evidence.
