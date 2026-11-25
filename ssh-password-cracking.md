# PentestGarage — SSH Password Cracking

- **Category:** Steganography / Password Cracking (Medium, 200 pts) — 136 solves
- **Date:** 2026-08-24
- **Target:** `ctf-gokulpg-ssh-password-cracking-8d8f1ff2.challenges.pentestgarage.com:31378` (internal `172.31.x`, via VPN)
- **User:** `ctf`

## Objective
Recover the SSH password/passphrase from the supplied files, log in, and retrieve the flag.

## Supplied files
`ssh_password_cracking.zip` contained:
- `id_ecdsa` — an **encrypted** OpenSSH private key (`aes256-cbc` + `bcrypt` KDF, 16 rounds)
- `username.txt` — `ctf`

The challenge is **not** a login brute force. It's a **passphrase crack** of the encrypted private key ("Brute force alone won't help—choose the right approach"). The key must be decrypted with a wordlist, then used for key-based auth.

## Approach

Normally: `ssh2john id_ecdsa > hash && john --wordlist=rockyou.txt hash`.

No `john`/`hashcat` were installed, so I cracked it in **pure Python** using the `bcrypt` (bcrypt_pbkdf) and `cryptography` (AES-CBC) packages. Logic:
1. Base64-decode the key, parse the `openssh-key-v1` header → cipher, KDF, salt, rounds.
2. For each candidate: `key||iv = bcrypt_kdf(passphrase, salt, 48, rounds)`, AES-256-CBC decrypt the private section.
3. Valid passphrase ⇔ the two leading 4-byte check integers of the decrypted blob are equal.

Wordlist: `rockyou.txt` (SecLists).

```python
import base64, struct, bcrypt
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
# parse header -> salt, rounds, enc ; then per candidate:
keyiv = bcrypt.kdf(pw, salt, 48, rounds, ignore_few_rounds=True)
out = Cipher(algorithms.AES(keyiv[:32]), modes.CBC(keyiv[32:48])).decryptor().update(enc)
if out[:4] == out[4:8]:  # matching check bytes => correct passphrase
    print(pw)
```

## Result

- **Passphrase:** `blink182`  (found at candidate #164 in rockyou.txt)

Strip the passphrase and log in:
```bash
cp id_ecdsa key_plain
ssh-keygen -p -f key_plain -P 'blink182' -N ''
chmod 600 key_plain
ssh -i key_plain -p 31378 ctf@ctf-gokulpg-ssh-password-cracking-8d8f1ff2.challenges.pentestgarage.com
```

## Flag
_Pending — target instance was not listening at solve time (needs Start/Reboot from the panel). Recovered credentials confirmed; log in and `cat` the flag once the box is live._

## Notes / lessons
- Encrypted OpenSSH keys are crackable with `ssh2john` + rockyou; the check-bytes trick lets you validate a passphrase offline with no `john` needed.
- The "Steganography" tag is a red herring — no stego, just an encrypted key.
