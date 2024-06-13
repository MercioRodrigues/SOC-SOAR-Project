# Ubuntu Server agent

The objective of this part of the project is: 

- To install and deploy a Wazuh agent on a Ubuntu machine.
- Push Wazuh alerts level 5 or higher to Shuffle, brute-force login attempts are going to be detected.
- Enrich source IP address IOC using Virustotal API.
- Send alerts to Thehive so cases can be created. 
- Automate responsive action, sending an email to the analyst with a hyperlink to perform a Firewall drop action, to block the malicious IP. 

![Captura de ecrã 2024-06-13 154024](https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/01942c43-2e68-456b-b095-91f1f1807a6b)

</b>
</b>

## Install and deploy Ubuntu agent
</b>
Before starting to do anything it is important to update our system.

`apt-get update && apt-get upgrade -y`
