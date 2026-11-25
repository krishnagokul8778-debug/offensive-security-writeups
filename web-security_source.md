# PentestGarage — Web Security: "Source"

**Category:** Web Security
**Difficulty:** Beginner
**Target:** `ctf-gokulpg-source-68652a0c.challenges.pentestgarage.com` (port 80, resolves to `172.31.31.148` on the VPN lab)
**Flag:** `ctf{__redacted__}` *(solved; value removed for public repo)*
**Lesson:** Client-side authentication is no authentication — anything the browser runs, the user can read.

---

## 0. Setup

1. Connect the lab VPN (OpenVPN GUI → Connect `Gokul_PG`). You should get IP `10.8.0.110`.
2. Confirm the target answers on port 80. ICMP is filtered, so `ping` fails — test with TCP instead:
   ```bash
   curl -s -I "http://ctf-gokulpg-source-68652a0c.challenges.pentestgarage.com/"
   ```
   > The bare IP returns `404` because the load balancer routes by hostname (virtual hosting). Always use the full hostname.

---

## 1. Recon — look at the page

```bash
host="ctf-gokulpg-source-68652a0c.challenges.pentestgarage.com"
curl -s "http://$host/"
```

The page (`<title>Source`) is a Bootstrap login form. Two things stand out in the HTML:

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/crypto-js/3.1.2/rollups/aes.js"></script>
<script src="js/main.js"></script>
...
<button ... onclick="authenticate()">Login</button>
```

- It loads the **CryptoJS AES** library.
- The Login button calls `authenticate()`, defined in `js/main.js`.

**Key insight:** when login logic runs in client-side JavaScript, the secret is already sitting in your browser. Read the source.

---

## 2. Read the JavaScript source

```bash
curl -s "http://$host/js/main.js"
```

```js
const enc_flag = "U2FsdGVkX19j3hQ8OYn5Xz/+lFTrFxQ8rftsde1ruJdTKlIubC5Geoz+qpd+UxHCDBtKZF8kI9eb3nwV4UIwNA=="

function decrypt(message, key) {
    return CryptoJS.AES.decrypt(message, key).toString(CryptoJS.enc.Utf8);
}
function get_flag(password) { return decrypt(enc_flag, password) }

function authenticate() {
    if (username === "admin" && password === "38a7744f5523335db845ff1976bf4747") {
        alert('your flag is: ' + get_flag(password))
    } else {
        alert("Invalid Password, Try Harder!")
    }
}
```

Everything we need is here:
- Username: `admin`
- Password: `38a7744f5523335db845ff1976bf4747` (hardcoded — the raw string *is* the secret; nothing to crack)
- The flag is `enc_flag` **decrypted using the password as the key**.

---

## 3. Recover the flag

**Option A — just use the form.** Enter `admin` / `38a7744f5523335db845ff1976bf4747` and click Login. The alert prints the flag.

**Option B — browser console.** Open DevTools on the page and run:
```js
get_flag("38a7744f5523335db845ff1976bf4747")
```

**Option C — decrypt from the shell (no browser).**
The base64 starts with `U2FsdGVkX19`, which decodes to the ASCII header `Salted__`. That's the OpenSSL "salted" format CryptoJS uses by default: AES-256-CBC with an MD5 key-derivation function. So OpenSSL can decrypt it directly:

```bash
echo "U2FsdGVkX19j3hQ8OYn5Xz/+lFTrFxQ8rftsde1ruJdTKlIubC5Geoz+qpd+UxHCDBtKZF8kI9eb3nwV4UIwNA==" \
  | openssl enc -aes-256-cbc -d -a -md md5 -pass pass:38a7744f5523335db845ff1976bf4747
```

Output:
```
ctf{__redacted__}
```

---

## 4. Takeaways

- **View source first.** HTML comments, linked JS, and files like `robots.txt` routinely leak secrets and hidden paths.
- **Never trust the client.** Credential checks and "encryption keys" shipped to the browser are visible to anyone. Real auth must happen server-side.
- **Recognize `Salted__` / `U2FsdGVkX19`.** It signals OpenSSL-compatible AES — decryptable with `openssl enc` once you know the passphrase.
- **A hash isn't always a puzzle.** Here the MD5-looking string was compared as a literal, so it *was* the password — no cracking required.
