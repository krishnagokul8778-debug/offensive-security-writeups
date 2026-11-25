# sensitive_data_exposure — PentestGarage

- **Category:** Web Security
- **Difficulty:** Medium (200 pts)
- **Date solved:** 2026-08-10
- **Target:** `ctf-gokulpg-sensitive-data-exposure-c0c1ebd7.challenges.pentestgarage.com:80`

## Challenge
> Sensitive information should never be publicly accessible. Find exposed
> credentials, secrets, or configuration files and use them to recover the flag.

## Recon
Static "Under Construction" page (nginx, Last-Modified 2021). Probing common
sensitive paths found an **exposed `.git/` directory**:
- `/.git/` → 403 (listing off) but individual files reachable
- `/.git/HEAD` → 200 (`ref: refs/heads/master`)
- `/.git/config` → 200

## Reading history
```
curl .git/logs/HEAD
```
```
0000... b2b83664...  commit (initial): initial commit
b2b83664... 270c9d06...  commit: removed sensitive data leak
```
The second commit "removed sensitive data leak" — so the secret is still in the
**first commit** `b2b83664`. (git deletes are not retroactive; the blob remains.)

## Reconstructing the repo (no dir listing)
Fetched objects by hash into a local `.git` and walked the tree:
```
git init
# fetch .git/objects/<xx>/<rest> for each hash:
commit b2b83664  ->  tree a3cb0d83
tree   a3cb0d83  ->  blob acbf8768  flag.txt
                     blob 609f9f9b  index.html
git cat-file -p acbf8768...
```
(Equivalent to running `git-dumper`.)

## Flag
```
ctf{__redacted__}
```

## Takeaways / remediation
- Never deploy the `.git/` directory to a web root; block it at the server
  (`location ~ /\.git { deny all; }`) and deploy build artifacts only.
- Committing a secret then deleting it in a later commit does NOT remove it —
  it stays in history. Rotate the secret and rewrite history (filter-repo / BFG).
