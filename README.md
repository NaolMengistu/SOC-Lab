# SOC Analyst Home Lab

## Table of Contents
1. [Overview](#overview)
2. [Objectives](#objectives)
3. [Lab Architecture](#lab-architecture)
4. [Tools & Technologies](#tools--technologies)
5. [Setup Guide (Step by Step)](#setup-guide-step-by-step)
6. [Simulating & Analyzing Attacks](#simulating--analyzing-attacks)
7. [Troubleshooting Common Issues](#troubleshooting-common-issues)
8. [Future Improvements](#future-improvements)
9. [Conclusion & Disclaimer](#conclusion--disclaimer)

---

![SPL login](https://github.com/user-attachments/assets/b03dfe88-2b2f-46fe-bad6-5669919a339d)

# SOC Analyst Home Lab

## **Overview**
This repository documents my journey in setting up a SOC (Security Operations Center) home lab using Splunk, VirtualBox, and multiple virtual machines to simulate real-world cybersecurity scenarios.

---

## **Objectives**
- Gain practical experience with Splunk for log monitoring and analysis.
- Configure a realistic SOC lab environment using VirtualBox and physical machines.
- Set up Windows and Linux machines for log collection and forwarding.
- Create real-world event scenarios for investigation and detection.

---

## **Lab Architecture**

### Network Topology

```
    +---------------------+          +-----------------------+          +-------------------+
    | Windows Server 22 vm|          | Kali Linux (VM)       |          | Windows 10 (VM)   |
    | 192.168.0.150       |<-------->| 192.168.0.130         |<-------->| 192.168.0.120     |
    +---------------------+          +-----------------------+          +-------------------+
                                                ^
                                                |
                                                v
                                 +--------------------------------+
                                 |  Windows 11 (Splunk Enterprise)|
                                 |  192.168.0.196 (Receiver)      |
                                 +--------------------------------+
                                                ^
                                                |
                                                |
                                                v
                                      +------------------+
                                      | Windows 10       |
                                      | 192.168.0.131    |
                                      +------------------+
                   (All forward logs to the Windows 11 machine running Splunk)

```


| Machine | Role | IP Address | Network Mode |
| --- | --- | --- | --- |
| **Windows 11** | Splunk Enterprise (Log Receiver), physical computer | 192.168.0.196 | - |
| **Windows Server 22 VM** | Log Source | 192.168.0.150 | Bridged |
| **Windows 10 VM** | Log Source | 192.168.0.120 | Bridged |
| **Kali Linux VM** | Log Source | 192.168.0.130 | Bridged |
| **Windows 10** | Log Source, physical computer | 192.168.0.131 | - |


All machines are forwarding logs to the Windows 11 machine running Splunk Enterprise.

## Tools & Technologies

- **Splunk Enterprise** (Log collection & analysis)
- **Universal Forwarder** (For log forwarding from endpoints to Splunk)
- **VirtualBox** (Virtualization platform)

*(Optional) Add other tools like Wireshark if you use them.*

---

## Setup Guide (Step by Step)

### 1. Install & Configure Virtual Machines

1. **Install VirtualBox**: Download and install VirtualBox on your host machine.
2. **Create VMs**: Create virtual machines for:
    - Windows Server 2022
    - Windows 10 (one or more VMs)
    - Kali Linux
3. **Network Mode**:
    - Set each VM to **Bridged Adapter** so that they get IPs on the same subnet (192.168.0.x).
4. **Assign Static IP Addresses**:
    - Windows Server 22: 192.168.0.150
    - Windows 10 VM(s): 192.168.0.120
    - Kali Linux: 192.168.0.130
    - Windows 10: 192.168.0.131

![Win-10-Net-Setting](https://github.com/user-attachments/assets/9e46f33a-d15d-42ec-b4f9-ba493f48073b)


### 2. Install & Configure Splunk on Windows 11 (Main Machine)

1. **Download Splunk Enterprise** from [Splunk's official website](https://www.splunk.com/).
2. **Run Installer**: Follow the wizard to install Splunk. Choose default settings or customize if needed.
3. **Initial Login**: Once installed, navigate to `http://localhost:8000`. Log in with the credentials you set.
4. **Enable Receiving on Port 9997**:
    - Go to **Settings > Forwarding and Receiving** > **Configure Receiving** > **New Receiving Port**.
    - Add custom or **9997** if it’s not already there.
5. **Local Data Inputs** (Optional):
    - Navigate to **Settings > Data Inputs** and configure **Local Event Log Collection** or other local logs you want to ingest.


### 3. Install & Configure Universal Forwarder on Endpoint Machines

### A. Windows Setup (Server 22, Windows 10 VMs, Windows 10 Laptop)

1. **Download Splunk Universal Forwarder** from Splunk’s site.
2. **Install** using the wizard or command line.
3. Use the ip address of the log receiver machine as a deployment server and receiving indexer.
4. **Open Command Prompt as Administrator** in the forwarder’s `bin` folder.
5. **Configure Forwarding**: If it was configured previously
    
    ```powershell
    splunk list forward-server
    splunk remove forward-server <the ip of current receiver>:<the port number>
    splunk add forward-server 192.168.0.196:9997
    splunk enable boot-start
    splunk restart
    
    ```
    
6. **Add Monitors** (e.g., Windows Event Logs):
    
    ```powershell
    splunk add monitor WinEventLog://Security
    splunk add monitor WinEventLog://System
    
    ```
    

### B. Kali Linux Setup

1. **Download & Install** the Splunk Universal Forwarder `.deb` (for Debian-based distros) or `.tgz`.
2. **Navigate** to `/opt/splunkforwarder/bin/`.
3. **Configure Forwarding**: If it was configured previously
    
    ```bash
    sudo ./splunk list forward-server
    sudo ./splunk remove forward-server <the ip of current receiver>:<the port number>
    sudo ./splunk add forward-server 192.168.0.196:9997 -auth admin:changeme
    sudo ./splunk enable boot-start
    sudo ./splunk restart
    
    ```
    
4. **Add Monitor** (e.g., `/var/log/syslog`):
    
    ```bash
    sudo ./splunk add monitor /var/log/syslog -index main -sourcetype linux_syslog
    
    ```
    

### 4. Verify Log Flow in Splunk (Windows 11)

1. **Open Splunk Web** on Windows 11: `http://192.168.0.196:8000`.
2. **Check Forwarder Management**:
    - **Settings > Forwarder Management** to see if forwarders are connecting.
3. **Run a Broad Search**:
    
    ```
    index=* | stats count by host, sourcetype
    
    ```
    
    - Confirm you see events from each machine.
4. **Troubleshoot if No Logs**:
    - See **Troubleshooting** below.

### 5. Simulating & Analyzing Attacks

- **Generate an event**:
    - e.g., multiple failed Windows logins to simulate brute force.
- **Use SPL Queries** in Splunk to detect suspicious events.

**Example**: Brute Force Attack Detection

```
index=windows sourcetype="WinEventLog:Security" EventCode=4625
| bin _time span=1m
| stats count by _time, src_ip, user
| where count > 5

```

### Creating an Alert for Multiple Failed Logins

1. In **Splunk Web**, go to the **Search & Reporting** app.
2. Enter a search similar to:
    
    ```
    index=windows sourcetype="WinEventLog:Security" EventCode=4625
    | bin _time span=1m
    | stats count by _time, user, src_ip
    | where count > 5
    ```
    
3. Run the search to verify results (i.e., returns events where a user fails login more than 5 times in a 1-minute window).
4. Click **Save As** > **Alert**.
5. Provide a **Title** (e.g., "Brute Force Alert"), a **Description**, and set the **Alert Type** to **Scheduled**.
6. In **Trigger Conditions**, choose something like:
    - **Run every**: 5 minutes
    - **Trigger alert when**: Number of Results > 0 (meaning if your search returns any events, it triggers).
7. Under **Alert Actions**, select how you want to be notified (Email, Slack, etc.) and click **Save**.
8. Splunk will now check every 5 minutes; if it sees any set of events that match the brute force pattern (more than 5 failures in 1 minute), it will fire the alert.

You can refine the time window or threshold as needed for your environment.

- **Generate Attack Traffic**:
    - e.g., multiple failed Windows logins to simulate brute force.
- **Use SPL Queries** in Splunk to detect suspicious events.

**Example**: Brute Force Attack Detection

```
index=windows sourcetype="WinEventLog:Security" EventCode=4625
| bin _time span=1m
| stats count by _time, src_ip, user
| where count > 5
```


## Troubleshooting Common Issues

### Checklist

- [ ]  Machines on the same subnet (192.168.0.x)
- [ ]  Ping works between forwarders and indexer
- [ ]  Splunk listening on port 9997
- [ ]  Firewalls allow inbound/outbound on port 9997
- [ ]  Forwarder service is running (`splunk status`)

### 1. Forwarding Not Working

- **Check Splunk Listening**:
    - In Splunk Web, **Settings > Forwarding and Receiving** > **Receiving** should list **9997**.
    - On Windows 11, run `netstat -ano | findstr 9997` to confirm.
- **Verify Forwarder**:
    - On the endpoint, go to the forwarder bin directory and run `splunk list forward-server`.
    - If it shows "Configured but inactive," check firewall and connectivity.
- **Firewall**:
    - On Windows, `New-NetFirewallRule -DisplayName "Splunk 9997" -Direction Inbound -Protocol TCP -LocalPort 9997 -Action Allow`.

### 2. No Logs Showing in Splunk

- **Data Inputs**:
    - Confirm you’ve set up the correct event logs or file monitors.
- **Check Internal Logs**:
    - `index=_internal` can reveal forwarder or indexer errors.
- **Restart Splunk** if changes were made but not applied.

### 3. Connectivity Issues

- **Bridged Adapter**:
    - Ensure each VM is in bridged mode.
- **IP Assignment**:
    - Confirm unique IPs in the `192.168.0.x` range.
- **Test Ping**:
    - `ping 192.168.0.196` from each endpoint.

### 4. Authentication / Credential Problems

- If you forget the Splunk admin password:
    - Remove `passwd` file or use `user-seed.conf` method to reset.

### 5. Universal Forwarder “splunk not found” Error

- Use the correct installation path, e.g. `C:\Program Files\SplunkUniversalForwarder\bin` or `/opt/splunkforwarder/bin`.
- Add the path to your system `PATH` variable or use the full path.

---

## Future Improvements

- Integrate **Suricata** for network traffic analysis.
- Add SIEM correlation rules and dashboards.
- Explore **MITRE ATT&CK** use cases in Splunk.

## Conclusion & Disclaimer

This SOC Analyst home lab provides a strong foundation for hands-on experience in security operations, threat detection, and incident response using Splunk. By following the detailed setup and troubleshooting steps, you can build a fully functional environment that mimics real-world security monitoring tasks.

**Disclaimer**: All attack simulations and activities described here must be performed in a controlled, isolated lab environment. Unauthorized testing or scans on networks without explicit permission is illegal and unethical. Please use these instructions responsibly.
