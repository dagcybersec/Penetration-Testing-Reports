# Penetration-Testing-Reports
Professional penetration testing reports from authorized lab environments, focused on methodology, exploitation, impact, and remediation.
# Reports

| # | Target | OS | Key Techniques | Report |
|---|---|---|---|---|
| 01 | Bashed | Linux | Web RCE, sudo abuse, writable root-executed script | [View Report](01-Bashed/report.md) |
| 02 | OpenAdmin | Linux | OpenNetAdmin RCE, credential reuse, SSH key disclosure, sudo Nano abuse | [View Report](02-OpenAdmin/report.md) |
| 03 | Tabby | Linux | Path Traversal / LFI → Tomcat Credentials → WAR Deployment → PwnKit → Root | [View Report](03-Tabby/report.md) |
| 04 | Admirer | Linux | Credential Disclosure → FTP Backup → Adminer File Read → Credential Reuse → PYTHONPATH Hijacking → Root | [View Report](04-Admirer/report.md) |
| 05 | Bastion | Windows | Guest SMB Access → Windows Backup Exposure → Registry Hive Extraction → Credential Recovery → SSH Access → mRemoteNG Credential Decryption → Administrator | [View Report](05-Bastion/report.md) |
| 06 | Return | Windows | LDAP Configuration → Credential Capture → WinRM → Server Operators → VMTools Service Abuse → LocalSystem | [View Report](06-Return/report.md) |
