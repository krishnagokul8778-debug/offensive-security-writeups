# PentestGarage — Bruteforce

- **Category:** Forensics (Easy, 100 pts)
- **Date:** 2026-08-28
- **Supplied file:** `Access.log` (web server access log, 38 lines)

## Objective
Analyze the access log, identify malicious activity, distinguish successful vs failed authentication attempts, and recover the flag.

## Log overview
Custom access-log format: `[timestamp] IP - - "METHOD /path HTTP/1.1" STATUS "User-Agent"`. Single burst on 2024-11-13 12:15:32–12:15:50, four source IPs, endpoints `/login` (POST), `/admin` (GET), `/config` (GET). The 38 lines are the same 19 events duplicated (lines 20–38 repeat 1–19).

## Analysis

Per-IP status breakdown (deduplicated counts halve these):

| IP | 200 | 401 | 403 | 404 | Behavior |
|---|---|---|---|---|---|
| 49.47.192.51 | 6 | 4 | 4 | – | Failed `/login` attempts, `/admin` probe (403), then **successful logins (200)** |
| 85.210.67.189 | – | 6 | 2 | – | Rapid failed `/login` attempts, rotating User-Agents |
| 193.45.78.92 | – | 4 | 2 | 2 | Failed logins + `/admin` and `/config` probes |
| 163.245.89.121 | – | 6 | 2 | – | Failed logins + `/admin` probe |

Indicators of compromise:
- All four IPs hammer `/login` within seconds while rotating User-Agents (Windows/Mac/Android/iPhone) — classic distributed brute force.
- **Only `49.47.192.51` ever receives `200` on `POST /login`** (12:15:35, 12:15:44, 12:15:50), interleaved with its own 401/403 failures → the brute force succeeded from this IP.
- Confirming clue: diffing the duplicated halves of the log, the only altered line is the `49.47.192.51` 200-response line (a space removed after the IP), deliberately spotlighting that event.

Sanity checks: no non-ASCII/hidden bytes, no trailing whitespace, no embedded flag/base64 strings — the "hidden clue" is the successful-auth IOC itself.

## Result
Attacker IP that successfully brute-forced authentication: **`49.47.192.51`**

- **Flag:** `ctf{__redacted__}` *(solved; value removed for public repo)* (attacker IP; platform answer is the successful brute-force source)
