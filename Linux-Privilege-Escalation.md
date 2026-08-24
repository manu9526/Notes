# Linux Privilege Escalation — Enumeration Notes
 
## Enumeration
 
```bash
hostname
hostnamectl
uname -a
cat /proc/version
cat /etc/issue
ps -a
ps axjf
ps aux
env
sudo -l
id
id <user>          # eg: id kali
```
 
List users with home directories:
```bash
cat /etc/passwd | cut -d ":" -f 1 | grep home
```
 
### Network / Netstat
 
| Command | Purpose |
|---|---|
| `netstat -a` | All ports and established connections |
| `netstat -at` | List TCP or UDP protocol |
| `netstat -all` | Same as above |
| `netstat -l` | List ports in listening mode |
| `netstat -lt` | Only ports that are listening (TCP) |
| `netstat -ano` | Numeric output with PID/program info |
 
---
 
## Automated Enumeration Tools
 
- **LinPEAS**
- **LinEnum**
- **LES** (Linux Exploit Suggester)
- **Linux Smart Enumeration**
- **Linux Priv Checker**
---
 
## Kernel Exploit
 
```bash
hostname
hostnamectl
uname -a
cat /proc/version
```
 
---
 
## Priv-Esc: SUDO
 
Some applications allow spawning a root shell.
 
```bash
sudo -l   # check current sudo privileges
```
 
> Reference: [gtfobins.github.io](https://gtfobins.github.io)
 
---
 
## Priv-Esc: SUID
 
List files that have the SUID or SGID bit set:
 
```bash
find / -type f -perm -04000 -ls 2>/dev/null
find / -perm +6000 2>/dev/null | grep '/bin/'
```
 
---
 
## Priv-Esc: Capabilities
 
List enabled capabilities:
 
```bash
getcap -r / 2>/dev/null
```
 
> Reference: [GTFOBins](https://gtfobins.github.io)
 
---
 
## Priv-Esc: Cron Jobs
 
Find a cron job set by root:
 
```bash
cat /etc/crontab
```
 
---
 
## Priv-Esc: PATH
 
Identify writable directories:
 
```bash
find / -writable 2>/dev/null | cut -d "/" -f 2 | sort -u
find / -writable 2>/dev/null | grep usr | cut -d "/" -f 2,3 | sort -u
find / -writable 2>/dev/null | cut -d "/" -f 2,3 | grep -u proc | sort -u
```
 
Add `/tmp` to PATH:
 
```bash
export PATH=/tmp:$PATH
```
 
---
 
## Priv-Esc: NFS (Network File Sharing)
 
If `no_root_squash` is enabled on a writable NFS share, it can be exploited by creating an executable with the SUID bit set, which grants root-level privilege.
 
```bash
cat /etc/exports
```
 
List mountable shares:
```bash
showmount -e <target-IP>
```
 
Create a directory to mount the share:
```bash
mkdir /tmp/shares
```
 
Mount the share to the directory:
```bash
mount -o rw [IP]:[/share-name] /tmp/shares
# eg: mount -o rw 192.168.1.1:/backups /tmp/shares
```
