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



[Insert Screenshot: Wazuh Dashboard "Deploy New Agent" wizard showing the generated PowerShell command line string]

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



[Insert Screenshot: Windows PowerShell showing the red InvalidOperation error text, followed by the execution of the Set-DnsClientServerAddress command]

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
[Insert Screenshot: Windows PowerShell showing the installer downloading without errors, the service successfully starting, and its active status state]

### Phase 4: File Integrity Monitoring (FIM) Configuration

To showcase host-based intrusion detection capabilities, the agent was configured to track file modifications in real time.



#### 1. Opened the local Windows configuration file (C:\Program Files (x86)\ossec-agent\ossec.conf) using an administrative text editor.

#### 2. Navigated to the <syscheck> configuration block and appended a new directory monitoring definition. The realtime="yes" and report_changes="yes" attributes were specified to ensure instant alert delivery and deep content diff tracking:



```XML



<syscheck>

<frequency>300</frequency>

<directories realtime="yes" report_changes="yes">C:\Users\Public\Documents\SensitiveData</directories></syscheck>
```
Restarted the endpoint agent via PowerShell to load the new policy:



```PowerShell



Restart-Service -Name "wazuh-agent"
```
[Insert Screenshot: The Windows ossec.conf file open in a text editor, highlighting the newly added XML block]

### Phase 5: Security Event Verification & FIM Logs

To validate end-to-end functionality, a lifecycle file integrity test was performed directly on the Windows 11 workstation.



#### 1. The Test: A text file was created inside C:\Users\Public\Documents\SensitiveData, modified with sample text, and then deleted.

#### 2. The Result: Navigating back to the Wazuh Dashboard -> Modules -> File Integrity Monitoring -> Events page confirmed that the system intercepted and visualized the actions seamlessly.

The web console successfully logged the entire sequence across multiple rule levels:



- Rule 554: File added to the system (Creation event)

- Rule 550: Integrity checksum changed (Content modification event)

- Rule 553: File deleted from the system (Destruction event)

[Insert Screenshot: The main Wazuh Dashboard Events page displaying the precise FIM logs, showcasing the generated Rule IDs and paths mapped from the Windows endpoint]

## Core Security Capabilities Demonstrated

- DNS & Network Remediation: Diagnosing internal network blocks and utilizing administrative scripting interfaces to alter baseline IP stacks.

- SIEM Ingestion: Registering standalone operating system endpoints into a unified central management console.

- File Auditing: Implementing strict configuration parameters to audit runtime directory updates and preserve forensic chains of evidence.
