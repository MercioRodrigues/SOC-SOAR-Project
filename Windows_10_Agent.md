# Windows 10 VM Agent

## Introduction
Welcome to the Window Agent part of my SOC SOAR Project. In this part I am going to: 
</b>
- Deploy a Wazuh agent on a Windows 10 VM
- Set Wazuh to ingest Sysmon logs
- Create a custom rule for Mimikatz
- Run Mimikatz with a different name and check for alerts
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

</br>
</br>

Next, I went to PowerShell and ran mimikatz:
</br>
</br>

![Captura de ecrã 2024-06-01 140218](https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/71fd2110-e8e2-4a66-9a62-f66ff5bca473)
</br>
</br>
Let's go to Wazuh Discover mode select wazuh-archive and query for mimikatz.
</br>
</br>

![Captura de ecrã 2024-06-01 143818](https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/52149abc-6dac-4412-8572-4abe670f4117)
</br>
</br>
And there it is!
</br>
</br>

## Creating a Custom Rule
What I want to do next is to start creating our alert.
I see 2 events, I am interested in event **ID:1** because this will show process creations. If I open it and scroll down I can see a field called, `data.win.eventdata.originalFileName.` I am going to use this field to create our rule, **an attacker could easily change the name of the file to trick us**, so like this even if that happens Wazuh can see it.  
</br>
</br>
Wazuh stores some built-in rules. In order to go there from the dashboard, we click on Wazuh symbol on top and choose management and then rules, and then we click on manage rules.
</br>
</br>
<table>
  <tr>  
    <td><img src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/70862a6c-0849-4f92-862d-df9d4e4e88b2" alt="Wazuh Rules"/></td>
    <td><img src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/ce0ed86f-f1e9-4a1d-afa1-fe4b7016bc4a" alt="Wazuh Rules"/></td>
  </tr>
</table>
</br>
</br>

I search for Sysmon and since I am interested in event **id:1**, I click on the eye button to open it. Next I can see a bunch of built-in rules specifically created for event id:1. 
</br>
</br>
<p align="center">
    Sysmon Rules
    </br>
    <img src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/4a990d7d-ca69-4fc7-9e78-dc433658d82c" height="60%" width="60%" alt="Sysmon Rules"/>
</br>
</br>
    Sysmon id=1 Rules
    </br>
    <img src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/d817a0eb-e83d-40dc-8744-beac773460a0" height="50%" width="50%" alt="Sysmon id1 Rules"/>
</p>
</br>
</br>
I copy one of these rules and then go back and click on custom rules.
</br>

![Captura de ecrã 2024-06-08 202123](https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/bf8ae640-84f6-4cc4-8263-af3c383126a6)
</br>
</br>
**So lets create a custom rule!**
</br>
</br>
There is a rule file called **local_rules.xml**. I pressed edit and I pasted the rule that I copied before right under the rule that I already see, so we can edit it. **Careful with indentation**. 
</br>
I changed the id of the rule to **100002** because custom rules always start from 100000.
</br>
For fun, I put the severity level to **15** which is the maximum hehe, the higher the more important it is. 
</br>
I deleted the option `no_full_log`. I changed the field name to what we need: `win.eventdata.originalFileName`, and also changed the name of the exe to be detected to **mimikatz**, plus some more changes including description and putting the MITRE technique id **T1003** for credential dumping, that is what mimikatz does.  
</br>
</br>
It should look like this:
</br>

![Captura de ecrã 2024-06-01 150210](https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/853d78ec-32a0-4b77-816e-50794b671c0b)
</br>
</br>
After saving I click the restart button because every time we change a rule we need to restart the manager.
</br>
</br>
**To test this rule I am going to change `mimikatz` name to `svchost`, which is a well-known Windows process, to try to trick Wazuh.**
</br>
</br>
I go back to Powershell and run this time the "disguised" mimikatz, named `svchost.exe`.

![Captura de ecrã 2024-06-01 152110](https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/e461b5cc-284f-450d-810e-3bb03b6a6a2c)
![Captura de ecrã 2024-06-01 152009](https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/c596b78f-a083-454f-ac53-433012de132f)
And as we can see Wazuh triggered an alert,  even though I changed the filename. 
</br>
</br>
We can see more details by opening the alert and checking what I just configured.
<img src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/e9e308d7-df95-4362-b360-228c4073ce95" height="80%" width="80%" alt="Mimikatz alert"/>
</br>
</br>
During an investigation, I can use the hash that is showing to double-check on the Virustotal website. 
</br>

![Captura de ecrã 2024-06-08 203558](https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/4f545d9e-3001-4d66-aea3-26c23ecda667)
</br>
</br>
I used the MD5 hash and I confirm in fact that it is mimikatz.
</br>
</br>
![Captura de ecrã 2024-06-01 153242](https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/df5df136-b3b1-4889-b860-003a6f5ea5c7)
</br>
</br>
**So far so good. Next I am going to take care of the automation using Shuffle. And I want to make this Virustotal investigation step automatic and send an email to the analyst.** 
</br>
</br>
## Make it SOAR

Let's take a break for me to explain my workflow for this part of the project. 
</br>
</br>
Mimikatz alert is sent to shuffle   -»  Shuffle receives the alert and extracts the sha256 from the file 
-» Check the reputation score with VirusTotal    -»   Send the details to TheHive to create an alert 
-» Send an email to the SOC analyst 
</br>
</br>
I am going to use Shuffle and Thehive to accomplish this. 
**I will set up TheHive because I want to expose myself to a case management system.**
</br>
</br>
### Shuffle
After creating an account on the Shuffle website, I create a new workflow.
</br>
The page will look like this:
<p align="center">
Shuffle Workflow
</p>
<table>
  <tr>  
    </br>
     <td><img src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/bebb294c-ea94-4719-8692-9d2dc693478b"  alt="Shuffle Workflow"/></td>
     <td><img src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/d927c4f3-6c87-4465-b52a-bd4be46167d5" alt="Shuffle Workflow"/></td>
  </tr> 
</table>    
Here we can add our "triggers" and “apps”.
I select and drag a webhook inside our working area. 
Selecting it will bring a window where I can name it and see the webhook URL. I copy it because I will need this later for the Wazuh Manager.
</br>
</br>

Reference: _According to Shuffle documentation; “ Webhooks are the real-time handler for Shuffle data. Webhooks were initially implemented to handle data from Office365 connectors and TheHive, but have turned into a generic trigger, taking any kind of HTTP data, as we saw the need for it.”_
</br>
</br>

<div style="display: flex; align-items: center;">
  <img align="left" width="200" src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/893c8db8-7136-4f27-ac43-1a89f6e34a58"/>
  <div style="margin-left: 10px;">Before going to Wazuh I need to change the call on “change me” by clicking on it. I deleted Hello World and added an Execution argument autocompletion. Like the image on the left.</div>
  <br/>
  <br/>
  <div style="margin-left: 10px;">Next I go to my Wazuh manager server CLI and I need to instruct Wazuh to connect to shuffle. I can do this by adding what is called an "integration tag" into the ossec config file.</div>
  <br/>
  <pre style="background-color: rgba(0, 0, 0, 0.3); padding: 10px; border-radius: 5px; color: #ddd;">nano /var/ossec/etc/ossec.conf</pre>
</div>
<br/>
<br/>
<br/>
<br/>
<br/>
<br/>
<br/>
I used Chatgpt to help me with this, and I changed and copied the following into `ossec.conf` file:
<br/>
<br/>

```
<integration>
  <name>shuffle</name>
  <hook_url>Webhook_URL</hook_url>
  <alert_level>5</alert_level> 
  <alert_format>json</alert_format>
</integration>
```
<br/>
<br/>

The `<hook _url>` is the webhook url that I copied from Shuffle.
On the alert level if we want to receive all alert levels we put 0 instead of 5. 
In my case and for the purpose of this project and testing, I'm interested in receiving only  when it triggers the rule that I created in Wazuh, this rule if you remember have an ID of `100002`

<br/>

So I will replace the alert level line with `<rule_id>100002</rule_id>` instead.
<br/>
**Attention: be careful with indentation!**
<br/>
<br/>

![Captura de ecrã 2024-06-02 154633](https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/8d2ce367-7792-439c-9889-5da250ec9924)
After saving I restart wazuh-manager service.

`systemctl restart wazuh-manager.service`
<br/>
<br/>
Now let's go to my Windows VM to generate an alert and check if it's working on shuffle. 
I press the play button on my workflow, I execute mimikatz, and then I click on the “show executions” button, and I check if the alert is getting into Shuffle. 

![Captura de ecrã 2024-06-02 181828](https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/8efe7251-4e4e-44ee-8781-f75fd2dc792a)
<br/>
<br/>
If everything was configured correctly, the alert should appear like the image above. 
<br/>
<br/>
I can open the alert and check all the information from the alert. 
<br/>
How cool is that!?
<br/>
<br/>
<p align="center">
 Alert data inside Shuffle
</br>
</br>
  <img src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/020640db-785e-464b-8116-6d167b8527a8" height="60%" width="60%" alt="Wazuh Agent Folder"/>
<p/>
</br>
</br>

**My next step is to make Shuffle use Virustotal to check the reputation of mimikatz using its sha256.**
</br>
</br>

## Enriching Hash IOC
</br>
</br>

To accomplish this first I click on **“Change me”** and then under Find action I choose the option **“Regex capture group”**. For the input data I select **“Execution Argument”** and then **"hashes"**.
</br>
</br>
For the regex, I had to use Chatgpt one more time, so it could create for me a regex to parse sha256 values. It should look like the image below. 	








  
