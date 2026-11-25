# Offensive Security Write-ups

Practice write-ups from [PentestGarage](https://pentestgarage.com) CTF challenges, covering web
exploitation, binary exploitation, reverse engineering and password cracking. Each write-up explains
the vulnerability, the exploitation steps and the fix.

> Flag values are removed. These write-ups explain the method, not the answers.

## Web security

| Challenge | Vulnerability | Technique |
|---|---|---|
| [seeql](seeql.md) | SQL injection | Boolean + UNION-based extraction, schema dumping |
| [ssti-machine-control](ssti-machine-control.md) | Server-side template injection | Jinja2 sandbox escape to RCE |
| [insecure_deserialization](insecure_deserialization.md) | Insecure deserialization | Python pickle RCE via `__reduce__` |
| [sensitive_data_exposure](sensitive_data_exposure.md) | Exposed `.git` directory | Repo reconstruction from leaked objects |
| [sensitivity](sensitivity.md) | Exposed `.env` file | Config file disclosure |
| [web-security_source](web-security_source.md) | Client-side authentication | Reading secrets from JavaScript source |

## Binary exploitation and reversing

| Challenge | Type | Technique |
|---|---|---|
| [echo](echo.md) | Pwn | Format-string bug to ret2libc |
| [ezpz](ezpz.md) | Reversing | Reversing a custom bytecode VM |

## Password cracking and forensics

| Challenge | Type | Technique |
|---|---|---|
| [ssh-password-cracking](ssh-password-cracking.md) | Password cracking | Cracking an encrypted OpenSSH key |
| [bruteforce](bruteforce.md) | Log forensics | Finding a successful brute force in access logs |

## Full reports

| Target | Type | Result |
|---|---|---|
| [DC-3 (VulnHub)](dc-3/) | Full penetration test report (PDF) | Root access; 20 findings |

## Note

All challenges were solved in authorized CTF lab environments on PentestGarage.
