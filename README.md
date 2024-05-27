# Microsoft Sentinel SIEM Honeypot

![image](https://miro.medium.com/v2/resize:fit:720/format:webp/1*qcAT8sslc064Vwz0sk65wg.png)

## Introduction

In this write-up, I configured Sentinel in Microsoft Azure as a Security Information and Event Management (SIEM) system. I connected it to a live virtual machine (VM) that acted as a honeypot. Real-time attacks, including RDP Brute Force, originating from different parts of the world were observed. I utilized a custom PowerShell script to identify the attacker’s geolocation information, and I could visualize it on the Microsoft Sentinel Geo map.

Microsoft Azure is a cloud computing platform that provides a range of cloud services. Microsoft Sentinel, on the other hand, is a cloud-native solution for SIEM and Security Orchestration, Automation, and Response (SOAR) that is built on Azure. Azure is used for building, deploying, and managing applications and services. Sentinel provides centralized security monitoring and analysis for detecting and responding to cybersecurity threats. It integrates with other Microsoft security solutions and utilizes AI and machine learning for advanced threat detection.

## Setting up a VM in Azure

The very first part of this process is to create a virtual machine in Microsoft Azure that will be exposed to the internet as our honeypot and allow different people from different countries to attack it. Once they discover it, they’ll try to log into it.

The name of the resource group is “Honeypotlab,” and the name of the VM is “Honeypot-vm.” I configured the image to Windows 10.

![image](https://miro.medium.com/v2/resize:fit:720/format:webp/1*9cYToPUz_rTq0_9Dj-uWAw.png)

## Modifying Firewall Rules to Accept All Traffic

I navigated to the Networking tab and set up the new firewall rules to allow everything to be open to the public internet. The purpose is to make this virtual machine highly visible and easily discoverable through any means possible. This way, whether attackers use TCP pings, SYN scans, or ICMP pings, they can quickly detect it without disrupting the traffic. Once discovered, attackers will commence their attacks.

![image](https://miro.medium.com/v2/resize:fit:720/format:webp/1*GZz_5bfSv4ivY75aAznwCQ.png)

I put a ***** for the destination port, representing anything. I also changed the **Priority** to 100. I named this rule **MENACE_ANY_IN.**

![image](https://miro.medium.com/v2/resize:fit:640/format:webp/1*pntMacrcAPeIXBnVTvrCAw.png)

So, I removed the default inbound rule and created my own inbound rule that allows everything into the VM.

![image](https://miro.medium.com/v2/resize:fit:640/format:webp/1*ZfnKT4XxeGVvrrBZk6wUeQ.png)

## Creating a Log Analytics Workspace

In this section, I created a Log Analytics workspace named “log-honeypot” to ingest logs from the VM, such as the Windows Event Logs. I made a custom log containing geographic information to discover the attackers’ origin.

![image](https://miro.medium.com/v2/resize:fit:720/format:webp/1*iqf26OtFxwDw63NDVnyi8w.png)

> - Connected Log Analytics to my VM (no picture)

## Setting up Sentinel
I set up Microsoft Sentinel, the SIEM used to visualize the attack data, and then I picked the log analytics workspace I wanted to connect to, “log-honeypot.”

![image](https://miro.medium.com/v2/resize:fit:1100/format:webp/1*IWDoY-PtsubaKsSc06u0Xg.png)

## Connected to my VM Through Remote Desktop

I logged into the “Honeypot-vm” through Remote Desktop.

![image](https://miro.medium.com/v2/resize:fit:640/format:webp/1*qIUS02gHYBzjgyyU_07ErA.png)

I failed the first login attempt.

![image](https://miro.medium.com/v2/resize:fit:640/format:webp/1*aS1ywk82cWPk7to2YsEjQg.png)

Once logged in, I navigated to the Start menu → Event Viewer.

Here in Event Viewer, we can see Audit Failure Event ID 4625, which I focused on throughout this project. Audit Failure and Event ID 4625 is a security event for an account failing to log on.

![image](https://miro.medium.com/v2/resize:fit:720/format:webp/1*POfSDDUNR9NKeByUtpLBxQ.png)

I tried to log on again with an invalid username, johnfail10.

![image](https://miro.medium.com/v2/resize:fit:640/format:webp/1*XxdbMs9P3Rsg-sOmF3ucNg.png)

Then I expanded the Event Properties to investigate further, and we can see the Account Name, johnfail0, which is the username I tried to log in with. We can also see the Failure Reason as an “Unknown username or bad password.” I tried to log in from the JBTECH workstation, and Source Network Address is my IP address from where I tried to log into the VM and failed.

![image](https://miro.medium.com/v2/resize:fit:640/format:webp/1*RG1AKXorlNHI_u9i4DgDbA.png)

This Event Viewer can be used after people discover me on the internet. All their IP addresses will appear here. So next, programmatically with PowerShell, I took the IP address shown above and then used an IP Geolocation API to create a custom log, sent that log to the Log Analytics workspace in Azure, and used Sentinel SIEM to read the country, longitude, and latitude to plot out the different attackers.

## Running a PowerShell Script to Retrieve Geo Data from Attackers

This custom PowerShell gets all the Event Log - Security Log, grabs all the failed login events, and creates a new log file. I pasted in my API key and ran the script.

The purple output appears when a failed log is on in the Event Viewer. It looks through the Event Viewer for 4625 failed logins, extracts them, and sends them to the API address to get the country and Geo data. Then it creates a failed log-on log file called “failed_rdp.”

The last line is a real failed log-on which is me, johnfail0, from the previous log-on attempt. However, everything above is the sample data I will use to train the Log Analytics workspace on what to look for in the log file.

![image](https://miro.medium.com/v2/resize:fit:640/format:webp/1*cwCp2hAFZJpJpseRpkxswQ.png)

Using the invalid usernames johnfail2 and johnfail4, I created three more log-on failure events, which popped up in purple and sent to the “failed_rdp” log file.

![image](https://miro.medium.com/v2/resize:fit:574/format:webp/1*HrAdnND8jZG82d9S-t2aFA.png)

As mentioned before, I used the sample data from the “failed_rdp” log file to create a custom log inside the Log Analytics workspace that will allow me to train it to bring in that kind of custom log with the Geo data.

My query will exclude the sample data and everything with the sample host to prevent it from appearing when the honeypot SIEM is fully functional.

![image](https://miro.medium.com/v2/resize:fit:640/format:webp/1*8Rc1l-gbRURTTK5bJB4zIQ.png)

I ran the query below to check Security Event for the Windows event logs, and it returned all the failed RDP logs.

```bash
SecurityEvent 
| where EventID == 4625
````
![image](https://miro.medium.com/v2/resize:fit:640/format:webp/1*LCRC_JFmYXyJ_drXL-GxXA.png)

I ran the **Failed_RDP_WITH_GEO_CL** query and it returned all of the sample data and the final attempts where I failed to log in.

```bash
FAILED_RDP_WITH_GEO_CL
```

![image](https://miro.medium.com/v2/resize:fit:720/format:webp/1*gxk-PJBeLvSxw7S2nH5FAA.png)

## Creating Custom Fields
I took the raw data and extracted specific fields to create longitude, latitude, country, timestamp, and other fields.
![image](https://miro.medium.com/v2/resize:fit:1100/format:webp/1*VfZdNJgKfbI_gJ6uEkI-Vw.png)

## Testing the Extracted Fields from Raw Custom Log Data

I tested the extracted fields from raw custom log data by returning to the VM and intentionally failing to log in. Then I ran the “Failed_RDP” query to ensure the logs were coming in and getting parsed out correctly.

![image](https://miro.medium.com/v2/resize:fit:640/format:webp/1*9UVfqwV7n-oc6af060Ox1A.png)

## Setting up the map with Longitude, Longitude, and Country

I configured the map settings to set up the Geo map in the workbook. I named the map “Filed RDP World Map.”

![image](https://miro.medium.com/v2/resize:fit:640/format:webp/1*VKQWEecu4Cr6Bu8GQpqWTQ.png)

After setting up the world map, I waited 30 minutes to an hour to see if other people had discovered the VM.

At the bottom of the map, you can see where I failed to log into the VM four times from the United States and the IP address. The log got synced to the Log Analytics workspace, ran the query inside Sentinel, and we will see incremental increases displayed on the map every time failed log-on attempts into the honeypot VM.

![image](https://miro.medium.com/v2/resize:fit:1100/format:webp/1*-pibYCiQIoAVGIx65u2Bvw.png)

## Philippines Begins Attacking

After checking my PowerShell script, we can see that people in the Philippines have discovered my VM and are trying to brute-force it.

As time progresses, you’ll see more and more remote desktop login attempts from individuals trying to log on to the honeypot VM with different usernames from different locations worldwide.

![image](https://miro.medium.com/v2/resize:fit:640/format:webp/1*5wo_JzVjiLE9X4sHfymeKg.png)

