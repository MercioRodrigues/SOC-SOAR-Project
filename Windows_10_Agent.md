# Windows 10 VM Agent

## Introduction
Welcome to this part of my SOC SOAR Project. In this part I am going to: 
</b>
- Deploy a Wazuh agent on a Windows 10 VM
- Set Wazuh to ingest Sysmon logs
- Create a custom rule for Mimikatz
- Run Mimikatz and check for alerts
- Automate workflow using Shuffle SOAR:
    
    - Enrich File hash IOC with Virustotal
    - Create an Alert in TheHive
    - Send an email to the SOC Analyst
 
![Captura de ecrã 2024-06-11 152926](https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/67793509-fe87-4aec-bd56-0f2dfc1d876e)

## Deploying an Agent to Wazuh
</br>
First I went to Wazuh dashboard. Remember to log in with the credentials that were given upon installation.
</br>
I clicked the add agent button and then I selected the Windows option.
</br>
On the server address, I need to put my Wazuh server's public IP.
</br>
<p align="center">
</br>
</br>
<img src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/6d7b31d6-b3b2-4efa-b6e2-98f2ecc41eca" height="60%" width="60%" alt="Wazuh Agent Deploy"/>
</br>
<img src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/6e9528cb-b36a-40dc-a3b6-4658baac2a20" height="55%" width="55%" alt="Wazuh Agent Deploy"/>
</p>
</br>
</br>

Next I go to my Windows machine PowerShell and copy and paste the commands including the start agent one.
</br>
</br>
![Captura de ecrã 2024-06-11 161444](https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/0931622b-770c-4f9f-b8e6-a9dcf878185c)
</br>
</br>
I can go to Wazuh dashboard and see the newly added agent.
</br>
</br>
![Captura de ecrã 2024-05-31 175339](https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/4b40c53c-fe71-4646-89e7-b64557398e35)

From now on we can click on security events and query for events 😀.
</br>
</br>
Logging in on our Windows VM, we can see that Wazuh is successfully gathering events from the Windows machine, like a successful login. 
</br>
</br>
![Captura de ecrã 2024-06-01 131636](https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/ad4edbb4-ed90-48dc-b1c5-b6ab2c19aba6)
</br>
</br>
So now let's have some fun. I will try to create a custom alert for Mimikatz. 
</br>
For those who don't know Mimikatz is a tool for extracting passwords, hashes, PINs, and Kerberos tickets from Windows memory.

Let's look into the Wazuh config file on the Windows machine. I want to create an alert that would be generated when the processes containing Mimikatz are found.
</br> 
</br>
**Notice that:** If you remember in my "pre-setup", I installed Sysmon on the machine for this purpose.
</br>
</br>
**And now I will configure Wazuh to ingest logs from Sysmon.** 
</br>
</br>
## Ingest Sysmon logs
</br>
Lets get over to the Wazuh config file located at `C:\Program Files (x86)\ossec-agent\`
</br>
</br>
<p align="center">
  Wazuh Agent folder
</br>
</br>
  <img src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/751ddd06-8233-4471-98e9-d3adae9114df" height="60%" width="60%" alt="Wazuh Agent Folder"/>
<p/>
  
  <u>**Make sure to open `ossec` conf file with Notepad and admin privileges.**</u> 
</br>
</br>
When I scroll down through the file, I'm looking for the `Log analysis` section.
But before that, I also need the channel name of Sysmon, we can get that from Sysmon Properties, one of the ways is from "Event Viewer".
</br>
</br>
<p align="center">
  Sysmon Properties
</br>
</br>
  <img src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/ef8d2a46-9e29-4aa7-af0c-4e0f6cc126cc" height="60%" width="60%" alt="Sysmon Properties"/>
</p>
</br>
</br>

With that information, I can now create a new entry under `Log analysis` inside the `ossec` file
</br>
Copy and paste the Sysmon channel name between location brackets.
</br>

![Captura de ecrã 2024-06-01 134728](https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/878f1234-862d-4ce5-97bf-6fb5b7ece871)

_For the sake of ingestion, I deleted some of the other `Log analysis` entries…_ 
</br>
</br>
**Any time we change the Wazuh config we should restart its service on Windows Services and that's what I did next.** 
</br>
</br>
<p align="center">
Wazuh Service
</br>
</br>
  <img src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/2b7efaa1-90dc-41d3-a28c-aa4188743834" height="60%" width="60%" alt="Restart Wazuh service"/>
</p>
</br>
</br>
Back to Wazuh dashboard, I test it in the events tab and query for Sysmon events.
</br>
</br>
  <img src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/6706cc3c-a617-4bb1-b0e4-23b5342e9ee8"/>
</br>
</br>
So, so far so good 🙂.
</br>
</br>

By default wazuh doesn't log everything so before running Mimicatz I will change this behaviour on the Wazuh manager server by changing its `ossec.conf` file.
</br>
**Source:** https://documentation.wazuh.com/current/user-manual/manager/wazuh-archives.html

`nano /var/ossec/etc/ossec.conf`
</br>
</br>
<img src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/28344559-fc9f-4037-8186-8a12065cc600"/>
</br>
I changed the “no” values to “yes” and saved it, and then restarted Wazuh manager service.

`systemctl restart wazuh-manager.service`
</br>
</br>
This changes forces Wazuh to begin archiving all the logs into a file named “archives”.
</br>
In order for Wazuh to ingest these files we need to change `filebeat.yml`, located at `/etc/filebeat/filebeat.yml` inside our Wazuh server machine.
</br>
Filebeat is used in conjunction with the Wazuh manager to send events and alerts to the Wazuh indexer.
</br>
</br>
![Captura de ecrã 2024-06-08 165134](https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/3f81ee52-4417-42e6-a611-14bcaa389884)
</br>
I changed archives enabled to “true”. And then restarted filebeat service.

`systemctl restart filebeat.service`
</br>
</br>
</br>
Going back to the Wazuh dashboard I went to index patterns that are inside Stack Management
</br>
</br>
<table>
  <tr>  
    <td><img src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/bb927c00-b987-4b5c-8b58-0e00a9e5652e"  height="60%" width="60%" alt="Wazuh Stack Management"/></td>
    <td><img src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/0904ee1c-0ea1-41a8-8a8c-98f1e7a55f31"  alt="Wazuh Stack Management"/></td>
  </tr>
</table>
</br>
I want to create an index pattern for the archives so that I can search for all the logs.
</br>
</br>
<table>
  <tr>  
    <td><img src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/ea063c69-483e-410a-b899-0c1bb90a3b48" alt=""wazuh-archives"/></td>
    <td><img src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/a6dc86b3-74ae-4159-b216-7ed68de84aa3" alt="wazuh-archives"/></td>
  </tr>
</table>

I named the new index pattern **“wazuh-archives-*”** and click next. I chose **“timestamp”** and clicked create. 
</br>
</br>
Next let's download mimikatz. For that I had to make an exclusion on Windows security for the Downloads folder otherwise it would have been impossible
<table>
  <tr>  
    <td><img src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/164a2903-7230-423d-a21f-dd683374f1a5" alt="Windows Security"/></td>
    <td><img src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/0971ef4d-d9e4-4ff4-9a0d-ee6c98f21bbb" alt="Windows Security"/></td>
  </tr>
</table>

![Captura de ecrã 2024-06-08 171402](https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/d7a1ca96-57ea-4ac4-8107-ad28d50b96c3)






  
