# Ubuntu Server agent

The objective of this part of the project is: 

- To install and deploy a Wazuh agent on a Ubuntu machine.
- Push Wazuh alerts level 5 or higher to Shuffle, brute-force login attempts are going to be detected.
- Enrich source IP address IOC using Virustotal API.
- Send alerts to Thehive so cases can be created. 
- Automate responsive action, sending an email to the analyst with a hyperlink to perform a Firewall drop action, to block the malicious IP. 
</br>

![Captura de ecrã 2024-06-13 154024](https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/01942c43-2e68-456b-b095-91f1f1807a6b)

</br>
</br>

## Install and deploy Ubuntu agent
</br>
Before starting to do anything it is important to update our system.

`apt-get update && apt-get upgrade -y`

</br>
</br>

After updating and upgrading the Ubuntu machine I go to the Wazuh web interface, click on the drop-down arrow, and go to **“Agents”**. There I click **“Deploy new agent”**.
</br>
</br>
On the next screen, I fill in all the necessary options like Linux **“DEB amd64”**. Wazuh Manager public **IP address**, and the **name** of the new Agent, in my case I called it  **“User02”**. 



<p align="center">
  Deploying New Agent
</br>
</br>
  <img src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/eb1ab850-2280-4b29-8275-61979e7c254c" height="60%" width="60%" alt="Wazuh Deploy Agent"/>
<p/>
</br>
</br>
Commands will appear to copy and paste on the Ubuntu agent.
</br>
</br>

**Double-check Wazuh:** After the installation is completed, let’s check in the Wazuh `ossec.conf` file that the client-server IP is correct. 
`nano /var/ossec/etc/ossec.conf`
</br>
</br>
In the `<client>` section of the configuration file, check if the IP address of the Wazuh manager is correct:

```
<client>
  <server-ip>WAZUH_MANAGER_IP</server-ip>
</client>
```
</br>

> $\color{blue}{Note}$
> 
> After any change made to the `ossec.conf` it is mandatory to restart `wazuh-manager.service`
> 
> `systemctl restart wazuh-manager.service`

</br>
</br>
Just a few minutes later we already have brute force login attempts on our machine and Wazuh does a good job detecting it. 

![Captura de ecrã 2024-06-03 215606](https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/77f91a47-55ab-4f4e-b582-35b0f2a2dd51)
</br>
</br>
> $\color{blue}{Note}$
>
> I created an inbound firewall rule to allow all traffic on port 22 for this machine, so I could have telemetry being generated instantly. I was expecting all these login attempts to happen.
</br>
</br>
Next, instead of integrating a rule on the Wazuh manager configuration file like I did for mimikatz in the [Windows VM Agent](https://github.com/MercioRodrigues/SOC-SOAR-Project/blob/main/Windows_10_Agent.md) part of the project, this time I am going to configure Wazuh to send all alerts level 5 or higher to Shuffle. 
</br>
</br>
</br>
## Let's Shuffle
</br>

Let's go to Shuffle and create a new Workflow, I add a **Webhook** and copy its hook URL. 
</br>
To test if Wazuh is sending alerts to Shuffle, I connect the webhook to the **“change me”** node.
</br>

<div style="display: flex; align-items: center;">
  <img align="left" width="200" src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/23b8012b-cfa2-493a-86c1-5379f52d93d6"/>
  <div style="margin-left: 10px;">On the <b>“Change Me”</b> node I select <b>“Repeat back to me”</b> and on the call I add <b>“Execution Argument”</b>. 
  </br>
  </br>
  I save the workflow and press start.
</div>
</br>
</br>

To push alerts to shuffle I add the following lines to the manager configuration file located at `/var/ossec/etc/ossec.conf`. 
```
<integration>
   <name>shuffle</name>
   <hook_url>http://WEBHOOK.url </hook_url>
   <level>5</level>
   <alert_format>json</alert_format>
 </integration>
```

I insert the webhook URL that was copied before and for the purpose and objectives of this lab the level of alerts that I want is level 5 or above. 
</br>
</br>
I save and restart wazuh-manager service:
</br>
`sudo systemctl restart wazuh-manager.service`
</br>
</br>
<div style="display: flex; align-items: center;">
  <img align="left" width="200" src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/45300bf6-c681-45af-91d2-6fa59b91325d"/>
  </br>
  </br>
  </br>
  </br>
  <div style="margin-left: 10px;">Soon I start seeing “Runs” which means that Wazuh is sending alerts to Shuffle.
</div>
</br>
</br>
</br>
</br>
</br>
If I open one of the results details, I can see all the data related to the alert.   
</br>
</br>
<p align="center">
   Alert Data
    </br>
    </br>
    <img src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/56cca428-0e20-44a2-93a7-874ad1d8c95a" height="60%" width="60%" alt="Alert Data"/>
</p>
</br>
</br>
Next, I will set up the workflow, to enrich the source IP address IOC, connect to TheHive, and configure an automated responsive action. The objective is to block any unrecognized IP addresses that attempt to connect to our Ubuntu machine via SSH. The analyst will receive an email with a responsive action to take in the form of a link. 
</br>
</br>
Since we know already that the webhook is gathering alerts from Wazuh we need to add an HTTP application.  
</br>
</br>
<p align="center">
   Workflow
    </br>
    </br>
    <img src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/35cd2ed1-d97d-4a69-b9c9-e00611b11c71" height="60%" width="60%" alt="Workflow"/>
</p>


**Editing HTTP node:** 
- **Name:** I named it Get-API
- **Action:** Curl
- **Statement:** `curl -u USER:PASSWORD -k -X GET "https://WAZUH-IP:55000/security/user/authenticate?raw_true`
</br>
The USER and PASSWORD are the Wazuh API username and Password:</br>

`nano wazuh-install-files/wazuh-passwords.txt`   «—---- Where all Wazuh credentials are stored
</br>
I Replaced WAZUH-IP with my Wazuh manager IP.
</br>
</br>
By re-running the workflow I can see the API token that was gathered.
</br>
</br>
**curl** — a heavily used command line tool to make network requests. Shuffle has support for CURL parsing, meaning we can use this command directly to make our action, and
use the CURL command to log in. 
The Wazuh API will provide a JWT token upon success.
</br>
</br>

**Reference Source:** Wazuh RESTful API authentication Documentation. It can be found on the Wazuh website or inside shuffle within the Wazuh app documentation. 
</br>
</br>

> $\color{red}{Important}$
> 
> I allowed traffic inbound on port 55000 in my firewall that is protecting the Wazuh server.

</br>
</br>
</br>
</br>

### Virustotal
Next, I will add Virustotal to extract the source IP and use intelligence to gather more information about the IP. 
</br>
</br>
![Captura de ecrã 2024-06-07 173136](https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/2ad4a945-ee82-481d-82ba-82de9638dda6)

</br>
</br>
Setting up Virustotal, I start by authenticating. I took my Virustotal API key from the Virustotal account that I have.
</br>
</br>

I continue by selecting what action I want, which in this case is **“Get IP address report”** 
</br>
</br>

**Ip:** I select **“Execution argument” -> “Scrip”** which generates the output: `$exec.all_fields.data.srcip`

I rerun the workflow to check if everything is well so far. If so we should be able to see a **Success** Status for Virustotal and all the data as well as the stand-out srcip address. 

## Responsive action
In the next step we are going to configure our responsive action, but before that let's add the Wazuh app inside our workflow and connect it to Virustotal for now for testing purposes. 
</br>
</br>
<p align="center">
   Workflow
    </br>
    </br>
    <img src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/e68bd176-9f1f-4df6-a623-512a18db085a" height="30%" width="30%" alt="Workflow"/>
</p>
</br>
</br>
I will configure it and see if everything is working properly with my API token.
</br>
</br>
<div style="display: flex; align-items: center;">
  <img align="left" width="200" src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/afd115a3-2e40-40a9-8dd1-4dcf786f044c"/>
  <div style="margin-left: 10px;"></br></br><b>Apikey:</b> Autocomplete option with “HTTP”. In my case, I had named it “Get-API” before.  
  </br>
  </br>
  <b>URL:</b> https://wazuh_manager_IP:55000 —- I placed my public Wazuh manager IP address.
  </br>
  </br> 
  <b>Agents list:</b> Every Wazuh agent has an id associated to it, for example, in this lab my Ubuntu machine has an id of "User02". I autocomplete this field with the agent id execution argument.
    </br> 
    </br>
      <div style="display: flex; align-items: center;">
        <img align="left" height="18%" width="18%" src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/5a4f7c21-1d33-4904-8dd5-ad75f1b9e16a"/>  
        </br>
        </br>
        <code>$exec.all_fields.agent.id</code>
</div>
</br>
</br>
</br>
</br>
</br>
</br>
</br>

Before filling the **Command** parameter I need to go to Wazuh manager and edit the `ossec.conf` file and setup an active response. 
</br>
</br>
Inside the config file below the `<Active Response>` tag we can see several commands tags, what these do is in case a condition is met, do a specific command (action).
</br>
</br>
I am interested in the **firewall-drop** command. This command will essentially modify the IP tables of our Ubuntu machine, resulting in dropping the traffic from the IP. 
</br>
</br>
I need to add below the last command an active response tag with the command that we want to run. 
```
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <level>5</level>
  <timeout>no</timeout>
</active-response>
```
</br>
Save and restart `Wazuh-manager.service`
</br>
</br>
Let's go back to the Wazuh node in Shuffle and now we can change the following parameters:

- **Command:** `firewall-drop0`  The 0 is because we chose no timeout. For active responses regarding APIs, the command name appends the timeout value.  

- **Alert:** `{"data":{"srcip":"$exec.all_fields.data.srcip"}}`  _Ref: After running some tests and checking the active response logs, I saw that what comes after the alert parameter was something like: 
`“alerts”: {"data":{"srcip":"xxx.xxx.xxx.xxx"}}`. So I just copied it and inside the alert field, and where you see **“x”** I added an  **“Execution Argument”** that is the **“srcip”**._

</br>
</br>

### User Input
The next step in configuring my responsive action is to insert an **“User input”** inside our workflow between Virustotal and Wazuh, that way we send an email to the **Soc analyst** with information and links to Block the IP address, or not if desired. 
</br>
</br>
<p align="center">
   Workflow
    </br>
    </br>
    <img src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/18800784-563b-4f34-b195-ce39263cddca" height="60%" width="60%" alt="Workflow"/>
</p>
</br>
</br>

After selecting the **“User input”** node, I chose the email option, filled in the email address desired, and edited the **information** parameter sent to the Analyst.
</br>
</br>
In my particular case, I wanted for the purpose of this test to send via email the alert text and Virustotal reputation check:

```
Detected: "$exec.text Reputation:
Malicious: $virustotal_v3_1_copy.body.data.attributes.last_analysis_stats.malicious
Suspicious: $virustotal_v3_1_copy.body.data.attributes.last_analysis_stats.suspicious
Undetected: $virustotal_v3_1_copy.body.data.attributes.last_analysis_stats.undetected
harmless: $virustotal_v3_1_copy.body.data.attributes.last_analysis_stats.harmless"
Would you like to block this Srcip? $exec.all_fields.data.srcip
```
</br>
</br>
After rerunning the workflow, a sample of what the security analyst receives looks like this:

![Captura de ecrã 2024-06-07 205844](https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/4c17cb94-f499-4929-966e-90f6eb805205)

<img src="https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/5f52005e-c85e-4f99-a5a5-545ce3f0ffaa" height="30%" width="30%" alt="Workflow"/>

Awesome! 🙂 And The link works as well! As we can see inside the agent **“iptables”**.

![Captura de ecrã 2024-06-07 210420](https://github.com/MercioRodrigues/SOC-SOAR-Project/assets/172152200/cef4c45a-53c4-4d8a-8d63-af5db3337a2e)
</br>
</br>
## TheHive

The final stage of my SOAR automation project involves sending alerts to TheHive so Analysts can open and manage cases. 

If you follow the first part on how to add and set up TheHive inside Shuffle you already know how to proceed, you can go back to that [part](https://github.com/MercioRodrigues/SOC-SOAR-Project/blob/main/Windows_10_Agent.md#L496) if you need a reminder. 

This time I decided to send more information just like I did for the email to send to the Analyst.
Inside the **Summary** parameter:
