# SOC-SOAR-Project

## Objective


The objective of this project was to set up and configure a Security Operations Center (SOC) using Wazuh and TheHive for monitoring and responding to security incidents. Using Shuffle, the project involved creating a scalable, automated security monitoring system capable of detecting and responding to threats.


### Skills Learned


- Setting up and configuring cloud-based security tools.
- Installing and managing Wazuh for threat detection.
- Installing and managing TheHive for case management.
- Configuring Sysmon on Windows machines and integrating it with Wazuh for enhanced logging and detection.
- Creating custom alerts and rules in Wazuh.
- Implementing a SOAR by automating security workflows with Shuffle.
- Integrating external APIs like VirusTotal for threat intelligence enrichment.
- Utilizing Linux command-line tools for system administration and configuration management.
- Configuration of data storage and indexing engines (Cassandra, Elasticsearch).
- Cyber threat intelligence and response automation

### Tools Used


- Cloud Service Provider: Used to deploy Ubuntu machines.
- Wazuh: Open-source security monitoring tool.
- TheHive: Security incident response platform.
- Sysmon: System Monitor for Windows.
- Cassandra: Data storage for TheHive.
- Elasticsearch: Data indexing engine for TheHive.
- Shuffle: Automation platform for security operations.
- VirusTotal: Online threat intelligence platform. 
- Windows 10 VM: Used as a Wazuh agent.
- Ubuntu: Used for Wazuh and TheHive servers as well as one of the Agents.

## Steps




### Network Diagram
<table>
  <tr>
    <td><img src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/1af47f1b-9b55-4bea-8a59-7e72aceb400b" width="1000"/></td>
    <td><img src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/c9345f60-5586-4249-a3a2-3631fca9c4ac" width="1300"/></td>
  </tr>
</table>


### Pre-Setup
I deployed 3 Ubuntu machines from a CSP (cloud service provider), for running Wazuh and TheHive in the cloud. The 3rd one will be used as a Wazuh agent. I put Wazuh and Thehive behind a firewall to allow inbound traffic from my personal public address on port 22 for management, and another rule to allow traffic from the Agents. I installed a Windows 10 VM to use as a Wazuh agent, and I installed Sysmon on that machine as well. 

### Installation and Configuration of Wazuh and TheHive in the Cloud 
#### Wazuh Installation
After deploying the machine and, before starting to do anything it is important to update our system.
```
apt-get update && apt-get upgrade -y
```

When finished let’s install Wazuh Dashboard. 

**Source:** https://documentation.wazuh.com/current/installation-guide/wazuh-dashboard/installation-assistant.html
```
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
```

When the installation was completed I took note of the username and password to access Wazuh Dashboard. 
To see all the Wazuh passwords I can go anytime to read the file `wazuh-passwords.txt` that is inside `wazuh-install-files.tar`

I run `sudo tar -xvf wazuh-install-files.tar`  to extract it and, `cat wazuh-passwords.txt` to display it.

Next, I used the public IP address of the Wazuh server to access the dashboard: `https://165.XXX.XXX.XXX/` and log in with the username and password that was shown after the installation.
<p align="center">
<br/>
Wazuh Dashboard Log-in Page
<br/>
<br/>  
<img src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/4d11084d-953a-463a-b617-cdad68b7c1e7" height="50%" width="50%" alt="Wazuh Dashboard"/>
<br/>
<br/>Screenshot of Wazuh Dashboard
<img src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/2a5aaed9-74d0-4f60-b81c-900c8ba5cb54" height="80%" width="80%" alt="Wazuh Dashboard"/>
</p>
<br/>

#### TheHive installation 
On the second Ubuntu server; just like the one used to install Wazuh; before starting to do anything it is important to update our system.

`apt-get update && apt-get upgrade -y`


There are some prerequisites needed to be installed before the actual TheHive installation.

Source: https://docs.strangebee.com/thehive/installation/step-by-step-installation-guide/#dependencies

**The essential dependecies of TheHive's include:**

- Java
- Cassandra for robust data storage.
- Elasticsearch serves as a powerful indexing engine

**Dependencies:**
<br/>
```
apt install wget gnupg apt-transport-https git ca-certificates ca-certificates-java curl  software-properties-common python3-pip lsb-release
```

**Java Installation:**
<br/>
```
wget -qO- https://apt.corretto.aws/corretto.key | sudo gpg --dearmor  -o /usr/share/keyrings/corretto.gpg
echo "deb [signed-by=/usr/share/keyrings/corretto.gpg] https://apt.corretto.aws stable main" |  sudo tee -a /etc/apt/sources.list.d/corretto.sources.list
sudo apt update
sudo apt install java-common java-11-amazon-corretto-jdk
echo JAVA_HOME="/usr/lib/jvm/java-11-amazon-corretto" | sudo tee -a /etc/environment 
export JAVA_HOME="/usr/lib/jvm/java-11-amazon-corretto"
```

**Cassandra Installation:**
<br/>
```
wget -qO -  https://downloads.apache.org/cassandra/KEYS | sudo gpg --dearmor  -o /usr/share/keyrings/cassandra-archive.gpg
echo "deb [signed-by=/usr/share/keyrings/cassandra-archive.gpg] https://debian.cassandra.apache.org 40x main" |  sudo tee -a /etc/apt/sources.list.d/cassandra.sources.list
sudo apt update
sudo apt install cassandra
```


**ElasticSearch Installation:**
<br/>
```
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch |  sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
sudo apt-get install apt-transport-https
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/7.x/apt stable main" |  sudo tee /etc/apt/sources.list.d/elastic-7.x.list
sudo apt update
sudo apt install elasticsearch
```

**TheHive Installation Itself:**
<br/>
```
wget -O- https://archives.strangebee.com/keys/strangebee.gpg | sudo gpg --dearmor -o /usr/share/keyrings/strangebee-archive-keyring.gpg
echo 'deb [signed-by=/usr/share/keyrings/strangebee-archive-keyring.gpg] https://deb.strangebee.com thehive-5.2 main' | sudo tee -a /etc/apt/sources.list.d/strangebee.list
sudo apt-get update
sudo apt-get install -y thehive
```
<br/>
<br/>

### Dependencies Configuration

#### Configure Cassandra<br/>
In order to configure Cassandra i have to edit `cassandra.yaml' file.
```
nano /etc/cassandra/cassandra.yaml
```

Changes: 	
- **Cluster name** - I named mine “SOAR project”.
- **Listen address** - I put my TheHive server public address.
- **Rpc address** -  I put my TheHive server public address.
- **Seeds** - TheHive_server_IP:7000

**Remove Existing Data Before Starting Service:**
I need to stop the service first. 
```
sudo systemctl stop cassandra.service
```

Since we installed Cassandra using their package we have to remove old files.
```
rm -rf /var/lib/cassandra/*
```

And finally, let's start and enable Cassandra so it can start automatically after boot. 
```
sudo systemctl start cassandra
sudo systemctl enable cassandra
``` 
We can also check if its running...
```
sudo systemctl status cassandra
``` 
![Captura de ecrã 2024-05-31 153305](https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/e1def0fe-2f1b-4cea-8fa0-c8c12500d3f4)
<br/>
<br/>
#### Configure Elasticsearch

Next we config another dependency, Elasticsearch used for managing data indices or in other words, querying data. 
```
nano /etc/elasticsearch/elasticsearch.yml
```
Changes: 
- Cluster name to anything we want.  
- Uncomment `node.name`
- Put our Thehive machine ip after  `network.host`
- By default the port is 9200 and I just uncomment it.
- Under “Discovery” I uncomment `cluster.initial_master_nodes` and remove “node 2” since we are going to use only one. 
<br/>

<img src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/42cc7a92-0862-4e01-bc52-d5baf5e5264b" height="50%" width="50%" alt="Cassandra yaml file"/>
<img src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/f581881e-b0a1-4616-8513-8721e2b21bc9" height="50%" width="50%" alt="Cassandra yaml file"/>




  
