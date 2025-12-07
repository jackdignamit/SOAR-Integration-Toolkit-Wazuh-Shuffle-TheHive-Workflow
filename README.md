# SOC-Automation-with-Wazuh-Cortex-and-Shuffle
*Completed: November 1, 2025*

**description**

Tutorial by [MyDFIR](https://www.youtube.com/@MyDFIR).  
All implementation, exploration, and documentation performed independently as part of my cybersecurity learning journey.

- - - 

# Project Overview
description

## Tech Stack
| Tool & Technology | Purpose | Links |
|------------------|-----------------|-----------------|
| Wazuh       | SIEM/EDR platform for log collection, monitoring, and alerting | [https://wazuh.com/](https://wazuh.com/) |
| TheHive             | Incident response platform for case tracking | [https://strangebee.com/thehive/](https://strangebee.com/thehive/) |
| Shuffle             | SOAR platform for automated workflows  | [https://shuffler.io/](https://shuffler.io/) |
| Sysmon            | Endpoint telemtry for Windows  | [https://shuffler.io/](https://shuffler.io/) |
| Mimikatz           | Open-source credential-extracting tool used to simulate malicious activity | [https://github.com/ParrotSec/mimikatz](https://github.com/ParrotSec/mimikatz) |
| Windows 10 VM        | Endpoint environment to run Mimikatz and test Wazuh EDR detection | [https://www.vultr.com/](https://www.vultr.com/) |

- - -

# Architecture Diagram

<img width="1510" height="847" alt="Screenshot 2025-12-06 124107" src="https://github.com/user-attachments/assets/b7873087-d643-477c-a858-b45039c126df" />

1. Collect endpoint events of malicious behavior:
   
2. Wazuh Manager triggers alerts:
   
3. Shuffle receives Wazuh Alerts & sends responsive actions:

4. 

- - - 
# 🔢 Step-by-Step Walkthrough 🔢
## 1️⃣ Setup Virtualbox VM with Sysmon
To start, create a virtual machine, either using your own VM or a cloud-based service such as [Vultr](https://www.vultr.com/).  
For my lab, I used a [Virtualbox](https://www.virtualbox.org/) Windows 11 virtual machine.

**1.** To install a VirtualBox Windows 11 virtual machine, install the latest version from here: [https://www.virtualbox.org/](https://www.virtualbox.org/).  

**2.** Download a Windows 11 **.iso** file from [https://www.microsoft.com/en-us/software-download/windows11](https://www.microsoft.com/en-us/software-download/windows11).  

**3.** Add the **.iso** file to Virtualbox by navigating to New at the top of the screen, adding your .iso image, setting the version to Windows 11, and using all default settings.  
   - I recommend **8192 MB of base memory**, **2 processors**, and **80 GB** of hard disk storage space but it all depends on your setup.  
**4.** Startup your VM and follow the Microsoft setup. When it asks for product key, say you don't have one, and use Windows 11 Pro.  

**5.** Now let's setup Sysmon which can be installed from here: [https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)

*Sysmon is a free tool created by Microsoft that provides detailed information about system activities on Windows by capturing and logging events. For this particular project, it will be used to flag our credential stealer called Mimikatz.*

**6.** From there, we will need a config to tell Sysmon what to ignore and what to log. Let's use **sysmonconfig.xml** from here: [https://raw.githubusercontent.com/olafhartong/sysmon-modular/refs/heads/master/sysmonconfig.xml](https://raw.githubusercontent.com/olafhartong/sysmon-modular/refs/heads/master/sysmonconfig.xml)
   - I saved it in my downloaded Sysmon directory.
  
**7.** In an **Administrative PowerShell** window, use ```cd``` to change directories to your Sysmon folder. Its likely located in your `downloads` folder. Then, run it using `\.Sysmon64.exe -i sysmonconfig.xml`.

**You now have a functional VirtualBox virtual machine with Sysmon installed!**  
I recommend taking a snapshot of your virtual machine in this state so you can revert back to it if need be.

- - - 

## 2️⃣ Setup TheHive and Wazuh Virtual Machines
Let's use a cloud provider called [Vultr](https://www.vultr.com/) to create virtual machines to download Wazuh and TheHive.

**1.** Create an account on Vultr to create a cloud-based virtual machine. You don't need to use Vultr but it is what I will be using for this lab.  

**2.** Create two virtual machines for TheHive and Wazuh. I recommend naming them appropiately.

<img width="2078" height="397" alt="Screenshot 2025-11-01 125428" src="https://github.com/user-attachments/assets/1472914f-b043-4a83-b6d7-3246ef8927ac" />

**The virtual machine settings I used for each VM are:**
- Cloud Compute (Shared CPU)
- Ubuntu 24.04 x64
- Disabled Auto backups & IPv6 address (only need IPv4)
- **Wazuh VM:** 165 GB SSD, 4 vCPU, 8 GB Memory, 4 TB bandwidth ($40/month)
- **TheHive VM:** 320 GB SSD, 6 vCPUs, 16 GB Memory, 5 TB bandwidth ($80/month)

- - - 

## 3️⃣ Configure Wazuh
Wazuh is an open-source security platform that provides threat detection, monitoring, and incident response all in one platform.

1. Secure Shell connect into the Wazuh virtual machine using its public IP address listed on the Vultr dashboard.
   - `ssh root@[Wazuh public ip]`
2. To install Wazuh, download and run the Wazuh installation assistent. The latest version can be located here: [https://documentation.wazuh.com/current/quickstart.html](https://documentation.wazuh.com/current/quickstart.html)
   - This process may take a couple of minutes.
   
<img width="1463" height="792" alt="Screenshot 2025-12-07 125015" src="https://github.com/user-attachments/assets/f4036a4c-b8ca-45d7-a817-acafd1a4af0b" />
<img width="1481" height="176" alt="Screenshot 2025-11-01 125706" src="https://github.com/user-attachments/assets/0837eb85-bc97-4c89-9ec6-e9acae0c08e1" />

3. Once Wazuh has finished installing, **copy the provided username and password** needed to access the Wazuh dashboard. I recommend writing these down in a notepad file in case you forget.
   
4. We must first permit inbound traffic on TCP port 443 on the Wazuh server. Use the command `ufw allow 443` in the SSH session to enable this firewall rule.

5. Open up a web browser and navigate to the Wazuh dashboard using its public IP address. For example: `https://172.168.53.146`. Sign in using your obtained login information.
   - If accessing the dashboard fails, make sure that the server is up. You can check the server status using `systemctl status wazuh-manager.service`.

<img width="2483" height="1278" alt="Screenshot 2025-11-01 130713" src="https://github.com/user-attachments/assets/4a62cade-971b-42f4-9536-a168c59603d5" />

- - - 

## 4️⃣ Configure TheHive
TheHive is an open-source **Security Orchestration, Automation, and Response (SOAR)** platform designed to help security teams collaborate, investigate incidents, and manage cases efficiently.

1. Secure Shell connect into TheHive virtual machine using its public IP address listed on the Vultr dashboard.
   - `ssh root@[TheHive public ip]`
  
2. Open TheHive's website and follow the step-by-step instructions for setting up TheHive: [https://docs.strangebee.com/thehive/installation/installation-guide-linux-standalone-server/](https://docs.strangebee.com/thehive/installation/installation-guide-linux-standalone-server/)
- Install Java Virtual Machine

<img width="1366" height="1110" alt="Screenshot 2025-12-07 124900" src="https://github.com/user-attachments/assets/655473f2-bd3d-40ec-93c4-2d18aa9b0eda" />

3. 

- - - 

## 5️⃣ Install Mimikatz on your Virtualbox VM

- - - 

## 6️⃣ Create a custom detection rule on Wazuh

- - - 

## 7️⃣ Create a SOAR Playbook using Shuffle

- - - 

## 8️⃣ 

- - -

# Key Skills Demonstrated

- - - 

# Conclusion
