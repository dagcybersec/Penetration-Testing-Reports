# OpenAdmin Penetration Test Report

## Executive Summary

A black-box penetration test was conducted against the OpenAdmin Linux host hosted on Hack The Box.

The assessment resulted in full compromise of the target system.

Initial reconnaissance identified externally accessible SSH and HTTP services. Web enumeration subsequently revealed an OpenNetAdmin installation running version 18.1.1. The application was vulnerable to remote command execution, allowing operating system commands to be executed as the Apache service account `www-data`.

Post-exploitation enumeration identified cleartext database credentials stored within the OpenNetAdmin configuration. The exposed password had been reused by the local user `jimmy`, allowing SSH access to the host.

Access as `jimmy` exposed an internal-only web application listening on localhost. Review of the application source code revealed functionality that read and returned the SSH private key belonging to the user `joanna`.

The private key was encrypted; however, its passphrase was successfully recovered using an offline dictionary attack. The recovered key and passphrase allowed SSH authentication as `joanna`.

Finally, insecure sudo permissions allowed Joanna to execute Nano as root without supplying a password. Nano's command execution functionality was abused to obtain a root shell.

The assessment demonstrated that several application, credential-management and privilege-control weaknesses could be chained together to achieve complete compromise of the system.

---

## Assessment Scope

| Item | Details |
|---|---|
| Target | OpenAdmin |
| Platform | Hack The Box |
| Target IP | 10.129.54.123 |
| Hostname | openadmin.htb |
| Operating System | Ubuntu Linux |
| Assessment Type | Black-Box Penetration Test |
| Assessment Date | 06 September 2026 |

The objective of the assessment was to identify vulnerabilities and security weaknesses that could allow unauthorized access to the target and determine whether they could be chained to achieve full system compromise.

Testing was performed only against the designated Hack The Box target.

---

## Methodology

The assessment followed a structured penetration-testing methodology consisting of the following phases:

### Reconnaissance

Network and service enumeration was performed to identify exposed ports, services, software versions and web applications.

Tools and techniques included:

- Nmap
- HTTP service inspection
- Directory enumeration
- Manual web application review

### Vulnerability Identification

Identified applications and software versions were investigated for known security vulnerabilities.

Searchsploit and manual application analysis were used to identify potential attack vectors.

### Exploitation

Confirmed vulnerabilities were exploited in order to obtain an initial foothold on the target.

Remote command execution through OpenNetAdmin resulted in access as the `www-data` service account.

### Post-Exploitation Enumeration

Following initial compromise, the system was enumerated for:

- Local users
- Application configuration files
- Stored credentials
- Internal services
- Web application source code
- SSH material
- Sudo permissions
- Privilege-escalation opportunities

### Lateral Movement

Recovered credentials and authentication material were evaluated for their ability to provide access to additional local user accounts.

### Privilege Escalation

Local security configuration was reviewed to identify methods of obtaining administrative privileges.

Successful exploitation ultimately resulted in root-level access.

---

## Risk Rating

Findings were assigned severity ratings based on the potential impact of successful exploitation and the level of access provided.

| Severity | Description |
|---|---|
| Critical | Vulnerabilities that allow complete system compromise, unrestricted remote code execution or immediate root-level access. |
| High | Vulnerabilities that expose sensitive credentials, authentication material or provide significant unauthorized access. |
| Medium | Weaknesses that provide useful information or limited access but generally require additional conditions for major compromise. |
| Low | Minor security weaknesses with limited direct impact. |
| Informational | Observations that do not represent an immediate security vulnerability but may assist defensive improvement. |

---

## Findings Overview

| Finding ID | Finding | Severity |
|---|---|---|
| OA-01 | OpenNetAdmin 18.1.1 Remote Command Execution | Critical |
| OA-02 | Cleartext Application Credentials | High |
| OA-03 | Password Reuse Between Application and Local Account | High |
| OA-04 | SSH Private Key Disclosure Through Internal Web Application | High |
| OA-05 | Weak SSH Private Key Passphrase | High |
| OA-06 | Dangerous Sudo Configuration Allowing Root Command Execution | Critical |

---

## Compromise Walkthrough

### 1. Initial Reconnaissance

A TCP port scan was performed against the target:

```bash
nmap -sS -sC -sV -O -Pn -p1-10000 10.129.54.123
```

The scan identified two externally accessible services:

```text
22/tcp  open  ssh   OpenSSH 7.6p1 Ubuntu
80/tcp  open  http  Apache httpd 2.4.29
```

The HTTP service initially presented the default Apache2 Ubuntu landing page.

![Apache2 Ubuntu default page](evidence/01-apache-default-page.png)

---

### 2. Web Enumeration

Web content enumeration identified additional directories including:

```text
/artwork/
/music/
```

Further investigation revealed an OpenNetAdmin installation accessible at:

```text
http://openadmin.htb/ona/
```

The application reported:

```text
OpenNetAdmin v18.1.1
```

The identified software version was researched using:

```bash
searchsploit opennetadmin
```

A Remote Code Execution exploit affecting OpenNetAdmin 18.1.1 was identified:

```text
OpenNetAdmin 18.1.1 - Remote Code Execution
Exploit-DB: 47691
```

The vulnerability was associated with command injection and CVE-2019-25065.

---

### 3. Initial Access Through OpenNetAdmin

The identified exploit was executed against the OpenNetAdmin installation:

```bash
bash 47691.sh http://openadmin.htb/ona/
```

Remote command execution was verified using:

```bash
id
```

The command was executed as:

```text
www-data
```

![OpenNetAdmin 18.1.1 remote code execution](evidence/02-opennetadmin-rce.png)

This confirmed successful initial compromise of the web application.

A reverse shell was subsequently established to provide more convenient interactive access to the target as `www-data`.

---

### 4. Local Enumeration and Credential Discovery

Post-exploitation enumeration identified two local users of interest:


```text
jimmy
joanna
```

![Local enumeration with LinPEAS as www-data](evidence/03-local-enumeration-linpeas.png)


The OpenNetAdmin application configuration was reviewed.

The following file contained database connection credentials:

```text
/opt/ona/www/local/config/database_settings.inc.php
```

The configuration stored the database username and password in cleartext.


The password value has been intentionally excluded from this public report.

---

### 5. Password Reuse and SSH Access as Jimmy

The recovered password was tested against local system accounts.

SSH authentication as `jimmy` was successful:

```bash
ssh jimmy@10.129.54.123
```

Successful authentication demonstrated that the database password had been reused for the local Jimmy account.


The attack path had now progressed from:

```text
www-data → jimmy
```

---

### 6. Internal Web Application Discovery

Enumeration as Jimmy revealed an internal application directory:

```bash
ls -la /var/www/internal
```

The following files were present:

```text
index.php
logout.php
main.php
```

The application was associated with an Apache virtual host listening only on:

```text
127.0.0.1:52846
```

Reviewing the source code of `main.php` revealed:

```php
$output = shell_exec('cat /home/joanna/.ssh/id_rsa');
echo "<pre>$output</pre>";
```



This functionality directly read Joanna's SSH private key.

---

### 7. Joanna SSH Private Key Disclosure

Because the application was accessible only through localhost, it was queried from the compromised system:

```bash
curl http://127.0.0.1:52846/main.php
```

The HTTP response returned Joanna's encrypted RSA private key.

The returned key began with:

```text
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
```

![Internal web application disclosing Joanna's encrypted SSH private key](evidence/04-internal-app-key-disclosure.png)


The key was saved locally for further analysis.

---

### 8. Private Key Passphrase Recovery

The private key was encrypted and therefore required a passphrase before it could be used.

The key was converted into a John the Ripper compatible format:

```bash
ssh2john joanna_id_rsa > joanna.hash
```

An offline dictionary attack was then performed:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt joanna.hash
```

The passphrase protecting the key was successfully recovered.


![Joanna SSH private key passphrase recovery](evidence/05-joanna-passphrase-crack.png)

---

### 9. SSH Access as Joanna

The recovered private key was assigned appropriate filesystem permissions:


```bash
chmod 600 joanna_id_rsa
```

SSH authentication was then performed:

```bash
ssh -i joanna_id_rsa joanna@10.129.54.123
```

The recovered passphrase successfully unlocked the key.

Access was obtained as:

```text
joanna
```

The user proof file was accessible from Joanna's home directory.

![Successful SSH authentication as Joanna](evidence/06-joanna-ssh-access.png)

The user flag value has intentionally been excluded from this public report.

The lateral movement path was therefore:

```text
jimmy → joanna
```

---

### 10. Sudo Enumeration

Joanna's sudo permissions were enumerated:

```bash
sudo -l
```

The following rule was identified:

```text
(ALL) NOPASSWD: /bin/nano /opt/priv
```

![SSH access as Joanna and sudo privilege enumeration](evidence/07-joanna-access-and-sudo.png)

This allowed Joanna to launch Nano as root without providing a password.

---

### 11. Root Privilege Escalation

The permitted sudo command was executed:

```bash
sudo /bin/nano /opt/priv
```

Nano was therefore running with root privileges.

Nano's built-in command execution functionality was used to spawn a shell from within the privileged process.

Root-level access was verified using:

```bash
whoami
```

Result:

```text
root
```

![Root privilege escalation through sudo Nano abuse](evidence/08-root-privilege-escalation.png)

The target was fully compromised.

### Complete Attack Path

```text
OpenNetAdmin 18.1.1
        ↓
Remote Command Execution
        ↓
www-data
        ↓
Cleartext Database Credentials
        ↓
Password Reuse
        ↓
jimmy
        ↓
Internal Web Application
        ↓
Joanna SSH Private Key Disclosure
        ↓
Offline Passphrase Recovery
        ↓
joanna
        ↓
Insecure Sudo Nano Permission
        ↓
root
```

---

## Technical Findings

### OA-01 — OpenNetAdmin 18.1.1 Remote Command Execution

**Severity:** Critical

**Affected Asset:** `openadmin.htb:80` — OpenNetAdmin `/ona/`

#### Description

The target exposed OpenNetAdmin version 18.1.1 through the Apache HTTP service.

The installed version was vulnerable to command injection, allowing remote operating system commands to be executed through the application.

Public exploit code identified through Searchsploit successfully triggered the vulnerability.

#### Evidence

The vulnerable application was identified as:

```text
OpenNetAdmin v18.1.1
```

The following exploit was used:

```bash
bash 47691.sh http://openadmin.htb/ona/
```

Successful exploitation was verified using:

```bash
id
```

which demonstrated execution as:

```text
www-data
```

Evidence screenshot:

```text
evidence/02-opennetadmin-rce.png
```

#### Impact

An unauthenticated attacker with access to the vulnerable application could execute arbitrary commands on the underlying operating system.

This vulnerability provided the initial foothold used to compromise the remainder of the system.

#### Root Cause

An outdated and vulnerable version of OpenNetAdmin was deployed and exposed through the web server.

#### Recommendation

Upgrade OpenNetAdmin to a patched and supported release.

If the application is no longer required, remove it entirely.

Administrative applications should also be restricted to trusted management networks or authenticated users where operationally possible.

---

### OA-02 — Cleartext Application Credentials

**Severity:** High

**Affected Asset:** `/opt/ona/www/local/config/database_settings.inc.php`

#### Description

The OpenNetAdmin database configuration contained authentication credentials stored directly in cleartext.

The credentials became accessible following compromise of the web application.

#### Evidence

The following configuration file was identified:

```text
/opt/ona/www/local/config/database_settings.inc.php
```

The file contained database connection information including a username and password.


Sensitive credential values have intentionally been excluded from this report.

#### Impact

An attacker with local access to the application files could recover valid credentials.

These credentials could subsequently be tested against other accounts or services and increase the impact of the original compromise.

#### Root Cause

Sensitive authentication material was stored in plaintext within an application configuration file accessible from the compromised application environment.

#### Recommendation

Protect application secrets using a dedicated secrets-management solution where possible.

Configuration files containing authentication material should use restrictive filesystem permissions.

Credentials should also be rotated immediately following suspected application compromise.

---

### OA-03 — Password Reuse Between Application and Local Account

**Severity:** High

**Affected Asset:** Local account `jimmy` / SSH service on TCP 22

#### Description

The password recovered from the OpenNetAdmin database configuration was also valid for the local `jimmy` operating system account.

The credential was successfully used to authenticate to SSH.

#### Evidence

The recovered application password was tested against the Jimmy account:

```bash
ssh jimmy@10.129.54.123
```

Authentication succeeded.


#### Impact

Compromise of an application credential resulted directly in interactive operating system access.

This allowed an attacker to move from the restricted Apache `www-data` account to the local Jimmy account.

#### Root Cause

The same password was reused across separate security contexts.

#### Recommendation

Enforce unique credentials for:

- Database accounts
- Application accounts
- Local operating system accounts
- Administrative accounts

Password reuse should be prohibited through organisational credential-management policy.

---

### OA-04 — SSH Private Key Disclosure Through Internal Web Application

**Severity:** High

**Affected Asset:** Internal web application on `127.0.0.1:52846` / Joanna SSH credentials

#### Description

The internal web application contained code that directly read Joanna's SSH private key:

```php
$output = shell_exec('cat /home/joanna/.ssh/id_rsa');
echo "<pre>$output</pre>";
```

Accessing the application caused the private key contents to be returned in the HTTP response.

#### Evidence

The internal application was accessed with:

```bash
curl http://127.0.0.1:52846/main.php
```

The response contained:

```text
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
```

Evidence screenshot:

```text
evidence/04-internal-app-key-disclosure.png
```

#### Impact

An attacker who gained access to the internal application could retrieve sensitive authentication material belonging to Joanna.

Once the key passphrase was recovered, the attacker could impersonate Joanna and authenticate through SSH.

#### Root Cause

The internal web application had excessive access to sensitive user authentication material and explicitly exposed that material through application functionality.

#### Recommendation

Private SSH keys must never be accessible through web application functionality.

Applications should run under dedicated service accounts and follow the principle of least privilege.

Filesystem permissions should prevent application processes from reading user private keys.

---

### OA-05 — Weak SSH Private Key Passphrase

**Severity:** High

**Affected Asset:** Joanna SSH private key

#### Description

Joanna's SSH private key was encrypted with a passphrase.

However, the passphrase was sufficiently weak to be recovered using an offline dictionary attack with a commonly available password wordlist.

#### Evidence

The private key was converted for password cracking:

```bash
ssh2john joanna_id_rsa > joanna.hash
```

John the Ripper was then executed using:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt joanna.hash
```

The passphrase was successfully recovered.

Evidence screenshot:

```text
evidence/05-joanna-passphrase-crack.png
```

#### Impact

Possession of the encrypted private key allowed unrestricted offline password attempts without generating authentication events against the SSH service.

Recovery of the passphrase resulted in successful SSH access as Joanna.

#### Root Cause

The private key was protected with a predictable passphrase vulnerable to dictionary-based password cracking.

#### Recommendation

Use long, unique and high-entropy passphrases to protect SSH private keys.

Passphrases should not consist of common words, predictable phrases or passwords present in common password dictionaries.

---

### OA-06 — Dangerous Sudo Configuration Allowing Root Command Execution

**Severity:** Critical

**Affected Asset:** Local account `joanna` / `/bin/nano /opt/priv`

#### Description

Joanna was permitted to execute Nano as root without supplying a password:

```text
(ALL) NOPASSWD: /bin/nano /opt/priv
```

Although the rule appeared to restrict Joanna to editing a specific file, Nano is an interactive program capable of executing external commands.

The permitted command was therefore sufficient to obtain arbitrary command execution as root.

#### Evidence

The permission was identified using:

```bash
sudo -l
```

The allowed command was:

```text
(ALL) NOPASSWD: /bin/nano /opt/priv
```

Nano was launched with:

```bash
sudo /bin/nano /opt/priv
```

Command execution functionality within Nano was then used to spawn a shell.

Root privileges were verified with:

```bash
whoami
```

Result:

```text
root
```

Evidence screenshots:

```text
evidence/07-joanna-access-and-sudo.png
evidence/08-root-privilege-escalation.png
```

#### Impact

Compromise of Joanna's account provided an immediate path to root-level privileges.

An attacker obtaining root access would have unrestricted control over the host, including access to system configuration, sensitive files, credentials and running services.

#### Root Cause

An interactive text editor capable of executing external commands was granted unrestricted root privileges through sudo.

#### Recommendation

Do not permit interactive editors such as Nano or Vim to run through sudo unless unrestricted root access is intentionally required.

If privileged modification of a specific file is necessary, implement a narrowly scoped administrative mechanism that performs only the required action and does not provide arbitrary command execution.

---

## Remediation Summary

The following remediation actions are recommended in order of priority:

1. Upgrade or remove the vulnerable OpenNetAdmin 18.1.1 installation.
2. Remove the dangerous sudo rule permitting Joanna to execute Nano as root.
3. Rotate all credentials exposed through the OpenNetAdmin configuration.
4. Eliminate password reuse between application, database and operating system accounts.
5. Prevent internal web applications from accessing users' SSH private keys.
6. Replace weak SSH private key passphrases with strong, unique passphrases.
7. Apply restrictive permissions to files containing sensitive application credentials.
8. Apply least-privilege principles to application and service accounts.
9. Restrict administrative and internal services to authorized users and trusted network locations.
10. Regularly review sudo rules for applications capable of spawning shells or executing commands.

---

## Limitations

This assessment was performed against a single designated Hack The Box host in a controlled lab environment.

The engagement was conducted as a black-box assessment, with no credentials or internal system information provided before testing.

Testing focused on identifying a viable attack path from unauthenticated network access to full system compromise.

The assessment did not include:

- Source-code review of the complete OpenNetAdmin application
- Denial-of-service testing
- Persistence mechanisms
- Destructive testing
- Malware deployment
- Testing against systems outside the designated target

The findings therefore represent vulnerabilities and weaknesses identified during the assessment window and should not be interpreted as an exhaustive review of every possible security issue on the host.

---

## Conclusion

The OpenAdmin host was fully compromised through a chain of vulnerabilities and insecure security configurations.

Initial reconnaissance identified an exposed OpenNetAdmin 18.1.1 installation. Exploitation of a remote command-execution vulnerability provided an initial foothold as the `www-data` service account.

Post-exploitation enumeration then exposed cleartext application credentials. Password reuse allowed the recovered credential to provide SSH access as `jimmy`.

Further investigation revealed an internal web application capable of disclosing Joanna's encrypted SSH private key. The weak passphrase protecting the key was successfully recovered through an offline dictionary attack, enabling SSH access as `joanna`.

Finally, an unsafe sudo configuration allowed Joanna to execute Nano as root. Nano's command execution functionality was abused to obtain a root shell and complete control of the target.

The successful attack chain was:

```text
OpenNetAdmin RCE
        ↓
www-data
        ↓
Cleartext Credentials
        ↓
Password Reuse
        ↓
jimmy
        ↓
Internal Web Application
        ↓
SSH Private Key Disclosure
        ↓
Passphrase Recovery
        ↓
joanna
        ↓
Sudo Nano Abuse
        ↓
root
```

The assessment demonstrates how weaknesses across patch management, credential storage, password reuse, application permissions and sudo configuration can combine to turn an initial web application compromise into complete operating system compromise.
