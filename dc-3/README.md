# DC-3 — Penetration Test (VulnHub)

A full black-box penetration test of the **DC-3** VulnHub machine, from an unauthenticated start to
root, written up as a formal report.

**[Full report (PDF)](DC-3-Penetration-Test-Report.pdf)** — 19 pages: executive summary, CVSS-scored
findings, attack narrative and remediation.

- **Target:** DC-3 (VulnHub), host-only lab network
- **Type:** Black-box, unauthenticated
- **Result:** root access achieved; 20 findings catalogued (Critical to Informational)

## Attack path

| Phase | Action | Technique / CVE |
|---|---|---|
| 1. Recon | Port scan; only port 80 open | `nmap` |
| 2. Enumeration | Identified Joomla 3.7.0 on Apache (Ubuntu 16.04) | `whatweb`, version manifest |
| 3. SQL injection | Unauthenticated SQLi in `com_fields` dumped the admin password hash | CVE-2017-8917, `sqlmap` |
| 4. Credential cracking | Cracked the bcrypt hash offline in ~1 second | John the Ripper + rockyou |
| 5. Code execution | Planted a PHP reverse shell via the Joomla template editor → shell as `www-data` | Authenticated RCE |
| 6. Privilege escalation | Compiled and ran a kernel eBPF double-free exploit → root | CVE-2016-4557 |

## Key lessons

- Keep CMS software patched: a single unauthenticated SQL injection exposed the whole site.
- Weak passwords fall instantly to offline cracking; the admin hash broke in about a second.
- An outdated kernel turned a limited web-service account into full root.

## Scope

Tested only in my own isolated VirtualBox lab, against a VulnHub machine built for practice.
