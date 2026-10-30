# All-Pakistan TechBiz Security CTF: Write-ups

Write-ups for challenges from the **All-Pakistan TechBiz Security CTF** (30 October 2026, organised by TechBiz Security), covering OSINT, Web, Crypto, Forensics, Stego and Reversing.

Points listed are the points I earned (they decay as more teams solve a challenge).

| ID | Challenge | Category | Difficulty | Points earned | Flag |
|----|-----------|----------|------------|--------|------|
| C01 | [Hello, Recon](#c01--hello-recon) | OSINT | Warmup | 91 | `TBZ{r0b0ts_txt_is_not_a_security_control}` |
| C02 | [Layers](#c02--layers) | Crypto | Warmup | 88 | `TBZ{peel_it_back_one_layer_at_a_time}` |
| C03 | [View Source](#c03--view-source) | Web | Warmup | 84 | `TBZ{the_client_never_keeps_a_secret}` |
| C05 | [Loose Lips](#c05--loose-lips) | Forensics | Medium | 208 | `TBZ{plaintext_http_is_a_confession}` |
| C06 | [Hidden In Plain Sight](#c06--hidden-in-plain-sight) | Stego | Medium | 195 | `TBZ{one_bit_at_a_time_in_the_red}` |
| C07 | [Noise Floor](#c07--noise-floor) | Forensics | Medium | 198 | `TBZ{one_login_in_two_thousand_tries}` |
| C08 | [Amended](#c08--amended) | Forensics | Medium | 198 | `TBZ{rebasing_does_not_shred_anything}` |
| C09 | [Rolling Code](#c09--rolling-code) | Crypto | Hard | 400 | `TBZ{repeating_keys_repeat_their_mistakes}` |
| C10 | [Small e](#c10--small-e) | Crypto | Hard | 410 | `TBZ{textbook_rsa_is_not_encryption}` |
| C11 | [What Happened Here](#c11--what-happened-here) | Forensics | Hard | 380 | `TBZ{deleting_only_removes_the_pointer}` |
| C12 | [Checkpoint](#c12--checkpoint) | Reversing | Hard | 350 | `TBZ{a_little_machine_inside_the_machine}` |

**Tools used:** Python 3, Pillow, `base64`, `strings`, `git`, `awk`/`grep`, browser DevTools.

---

## C01 · Hello, Recon

**Category:** OSINT | **Points:** 91 | **Flag:** `TBZ{r0b0ts_txt_is_not_a_security_control}`

### Challenge
A target with the hint that someone "told the crawlers exactly what not to look at."

### Solution
`robots.txt` tells crawlers which paths *not* to visit, which makes it a free list of interesting paths for an attacker. Fetching `/robots.txt` on the target gave:

```
User-agent: *
Disallow: /admin/
Disallow: /backup/
Disallow: /s3cr3t-notes-7f3a/
Disallow: /tmp/

Sitemap: /sitemap.xml
```

`/admin/`, `/backup/` and `/tmp/` are generic decoys. The odd one out is `/s3cr3t-notes-7f3a/`. Opening it showed an "internal note: do not publish" page containing the flag.

### Takeaway
`robots.txt` is a politeness convention, not access control. Anything you list there is advertised to everyone, and sensitive paths need real authentication.

---

## C02 · Layers

**Category:** Crypto | **Points:** 88 | **Flag:** `TBZ{peel_it_back_one_layer_at_a_time}`

### Challenge
A pastebin dump with the hint *"Three wrappers, nothing clever. Work from the outside in."* No key, no cipher, just encodings.

### Solution
1. **Base64 decode** the blob. The result looks like hex, but contains letters `n`–`s`:
   ```
   54425n7o7065656p5s69745s...
   ```
2. Hex digits `a–f` are missing and `n–s` appear instead. That is **ROT13** applied to the hex letters (`a→n`, `b→o`, ... `f→s`). Reverse it by mapping `n–s` back to `a–f`.
3. **Hex decode** the result to ASCII.

```python
s = '54425n7o7065656p5s69745s...'   # output of step 1
t = s.translate(str.maketrans('nopqrs', 'abcdef'))
print(bytes.fromhex(t).decode())
```

### Takeaway
Encoding is not encryption. Chained encodings are peeled off one at a time, and a sanity check at each layer (does it look like hex? base64?) tells you what the next layer is.

---

## C03 · View Source

**Category:** Web | **Points:** 84 | **Flag:** `TBZ{the_client_never_keeps_a_secret}`

### Challenge
A status page with nothing on it. *"Reading the source is not quite enough. Follow what the code does with what you find."*

### Solution
The page source is just a heading and one script tag:

```html
<script src="app.bundle.js"></script>
```

The real content is in that bundle, `app.bundle.js`. It builds a "dev token" by interleaving two arrays of string fragments:

```js
var _f = ["TBZ{the_c", "er_keeps_"];
var _g = ["lient_nev", "a_secret}"];

function interleave(a, b) {
  var out = "";
  for (var i = 0; i < a.length; i++) { out += a[i] + (b[i] || ""); }
  return out;
}
window.__DEV_TOKEN__ = interleave(_f, _g);
```

`interleave` alternates the pieces, `_f[0] + _g[0] + _f[1] + _g[1]`:

```
"TBZ{the_c" + "lient_nev" + "er_keeps_" + "a_secret}"
= TBZ{the_client_never_keeps_a_secret}
```

You can also run it directly: open DevTools, go to the Console and type `window.__DEV_TOKEN__`.

### Takeaway
Anything shipped to the browser is readable by the user. A comment like "dev build only, stripped in production" is not protection, and secrets must never be placed in client-side code.

---

## C05 · Loose Lips

**Category:** Forensics | **Points:** 208 | **Flag:** `TBZ{plaintext_http_is_a_confession}`

### Challenge
A 1 KB packet capture from an internal network where "somebody signed in." Hint: *finding the credentials is step one, not the answer.*

### Solution
`strings` on the pcap shows the whole HTTP conversation in clear text:

```
POST /login HTTP/1.1
Host: intranet.local
username=s.rehman&password=Summer2026!&remember=1

HTTP/1.1 302 Found
Set-Cookie: sid=4f1a9c2e7b

GET /dashboard HTTP/1.1
Cookie: sid=4f1a9c2e7b

<!-- export token, rotate before go-live -->
<div id="export" data-token="VEJae3BsYWludGV4dF9odHRwX2lzX2FfY29uZmVzc2lvbn0="></div>
```

The credentials are a distraction. The real find is the `data-token` attribute in the dashboard response, which is **base64**:

```bash
echo 'VEJae3BsYWludGV4dF9odHRwX2lzX2FfY29uZmVzc2lvbn0=' | base64 -d
```

### Takeaway
Login over plain HTTP exposes credentials *and* session cookies. Anything sensitive in a response body (here, a token in an HTML attribute) is also readable by anyone on the wire.

---

## C06 · Hidden In Plain Sight

**Category:** Stego | **Points:** 195 | **Flag:** `TBZ{one_bit_at_a_time_in_the_red}`

### Challenge
An ordinary-looking PNG postcard (240×160). *"The image metadata tells you where to look."*

### Solution
The PNG has a `tEXt` Comment chunk that spells out the extraction method:

```
channel=R; order=row-major; terminator=NUL
```

So: take the **least significant bit of the Red channel**, pixel by pixel, row by row, pack the bits into bytes (MSB-first), and stop at a `0x00` byte.

```python
from PIL import Image

im = Image.open('postcard.png').convert('RGB')
bits = [p[0] & 1 for p in im.getdata()]

out = bytearray()
for i in range(0, len(bits) - 7, 8):
    v = int(''.join(map(str, bits[i:i+8])), 2)
    if v == 0:
        break
    out.append(v)
print(out.decode())
```

MSB-first gives the flag directly. LSB-first produces garbage, which confirms the bit order.

### Takeaway
Changing the LSB shifts a colour by at most 1/255, which is invisible to the eye. Always read image metadata first, since it often tells you exactly where to look.

---

## C07 · Noise Floor

**Category:** Forensics | **Points:** 198 | **Flag:** `TBZ{one_login_in_two_thousand_tries}`

### Challenge
An access log with ~2,000 login attempts overnight, and one of them worked. Find **which account** and **what was taken**.

### Solution
**1. Find the one success.** Filter `POST /login` and count status codes:

```bash
grep 'POST /login' access.log | awk '{print $9}' | sort | uniq -c
#  1789 401
#   211 200
#     1 302
```

The `401`s are failed logins, and the single `302` (redirect) is the successful one:

```
103.244.178.62 - - [25/Oct/2026:02:49:45 +0500] "POST /login?u=m.iqbal HTTP/1.1" 302 0 "-" "python-requests/2.32.3"
```

**2. Pivot on the attacker IP** to see what they did after logging in:

```bash
grep '103.244.178.62' access.log
```

```
03:12:51  GET /dashboard
03:12:54  GET /reports
03:12:55  GET /reports/q3-summary.pdf          (184 KB)
03:12:58  GET /export?format=csv&token=VEJae29uZV9sb2dpbl9pbl90d29fdGhvdXNhbmRfdHJpZXN9   (2 KB)
03:13:00  GET /logout
```

**3.** The `token` parameter is base64 and decodes to the flag.

### Findings summary
| | |
|---|---|
| Compromised account | `m.iqbal` |
| Attacker IP | `103.244.178.62` |
| Tooling | `python-requests/2.32.3` |
| Time of login | 02:49:45 (+0500) |
| Data taken | `q3-summary.pdf` and a CSV export |

### Takeaway
Don't count the failures. Look for the outlier status code, then pivot on the source IP. Note that the successful login came ~23 minutes before the data theft, so the attacker waited.

---

## C08 · Amended

**Category:** Forensics | **Points:** 198 | **Flag:** `TBZ{rebasing_does_not_shred_anything}`

### Challenge
A zipped git repo. Someone committed a secret, noticed, and "cleaned up the history." The README says so itself.

### Solution
`git log --all` shows only 4 clean commits and `git reflog` is empty, so the history looks tidy. But rewriting history only moves branch pointers; the old objects stay in `.git/objects` until garbage collection runs.

```bash
git fsck --lost-found --unreachable
# unreachable commit 5dad0e0...
# unreachable commit e841efb...
# unreachable blob   5ac45ff...
```

Inspect the dangling commit:

```bash
git show e841efb
```

```
Author: M. Iqbal
    Add local env file so the agent runs on the bench

+++ b/.env
+NG_API=https://api.northgate.test
+NG_ENV=production
+NG_DEPLOY_TOKEN=TBZ{rebasing_does_not_shred_anything}
```

`git cat-file -p 5ac45ff` prints the same `.env` blob directly.

### Takeaway
Rebasing or amending does **not** remove data. The secret survives until `git gc --prune=now`, and anyone who ever cloned or pulled the repo still has it. In a real incident the correct response is to **rotate the credential** (this one was `NG_ENV=production`), not just clean the history.

---

## C09 · Rolling Code

**Category:** Crypto | **Points:** 400 | **Flag:** `TBZ{repeating_keys_repeat_their_mistakes}`

### Challenge
One long intercepted radio message, base64-armoured. The "rolling code" scheme isn't rolling at all, and no key is given. *"The message is long enough to give it to you."*

### Solution
The "rolling code" is a **repeating-key XOR** cipher.

**1. Recover the key length** with normalized Hamming distance between adjacent blocks. The lowest score is at **11**, and 22 and 33 (multiples of 11) also score well, which confirms it.

**2. Recover each key byte.** Split the ciphertext into 11 columns (every 11th byte). Each column was XORed with a single key byte, so try all 256 values and keep the one that gives the most English-looking text (letters, spaces, no control characters).

```python
def score(bs):
    return (sum(c in b'etaoinshrdlu ETAOINSHRDLU' for c in bs)
            - 5 * sum(c < 32 or c > 126 for c in bs))

key = bytes(
    max(range(256), key=lambda kb: score(bytes(x ^ kb for x in d[i::k])))
    for i in range(k)
)
```

Recovered key: `n0rthg4te!1`

**3.** XORing with the key gives a readable internal memo from Northgate Security Operations, which even admits the scheme *"is not encryption... a short secret repeated over and over,"* and contains the flag.

### Takeaway
Repeating-key XOR is breakable with ciphertext alone: find the key length, then solve each column as single-byte XOR with frequency analysis. The longer the message, the easier it is.

---

## C10 · Small e

**Category:** Crypto | **Points:** 410 | **Flag:** `TBZ{textbook_rsa_is_not_encryption}`

### Challenge
A 2048-bit RSA modulus `n`, public exponent `e = 3`, and ciphertext `c`. *"The developer said the key was '2048-bit, so it's fine.'"*

### Solution
This is **textbook RSA with no padding and a tiny exponent**. The message `m` is short, so `m³ < n`. The modular reduction never happens, and:

```
c = m³ mod n  =  m³   (as plain integers)
```

So no factoring is needed. Just take the **integer cube root** of `c`:

```python
def icbrt(x):
    lo, hi = 0, 1 << (x.bit_length() // 3 + 2)
    while lo < hi:
        m = (lo + hi) // 2
        if m**3 < x: lo = m + 1
        else:        hi = m
    return lo

m = icbrt(c)
assert m**3 == c
print(m.to_bytes((m.bit_length() + 7) // 8, 'big'))
```

The assertion `m**3 == c` passing confirms the attack worked.

### Takeaway
Key size doesn't help when the scheme is broken. RSA without padding (OAEP) is insecure, and a small `e` with a small message makes it trivially reversible.

---

## C11 · What Happened Here

**Category:** Forensics | **Points:** 380 | **Flag:** `TBZ{deleting_only_removes_the_pointer}`

### Challenge
A 4 MiB raw USB image with three visible files. *"The one that mattered is not one of them."*

### Solution
`file` identifies a **FAT16** volume:

```
usb.img: DOS/MBR boot sector ... FAT (16 bit)
```

The root directory lists `README.TXT`, `ROSTER.CSV`, `NOTES.TXT`, plus a fourth entry whose first character is gone (`?ANDOVER.TXT`). That is a **deleted file**. In FAT, deleting only overwrites the first byte of the directory entry with `0xE5` and marks the clusters free; the data itself stays on disk.

Running `strings` on the raw image recovers it:

```
Incident handover - DO NOT CIRCULATE
-----------------------------------
Temp credential issued for the audit window:
TBZ{deleting_only_removes_the_pointer}
Revoke on Monday.
```

(Forensic tools like `fls`/`icat`, Autopsy, or PhotoRec would recover the same file with proper metadata.)

### Takeaway
Deleting a file does not erase it. Always examine raw disk images for deleted directory entries and unallocated space. Secure removal requires overwriting (wiping) the data.

---

## C12 · Checkpoint

**Category:** Reversing | **Points:** 350 | **Flag:** `TBZ{a_little_machine_inside_the_machine}`

### Challenge
A handset licensing page asking for an 8-character key. *"The answer is not in the page. Everything you need to work it out is."*

### Solution
The page only has an input and a button. All logic is in `check.js`, a small **virtual machine**:

- `PROG` (base64) holds **8 tiny programs**, one per key character. Each is a list of `(opcode, immediate)` pairs.
- Opcodes: `91` = XOR, `46` = ADD, `65` = SUB, `156` = ROL (rotate left), all on an 8-bit accumulator.
- Each character is run through its program, and the result must equal `TARGET[i]`.
- If the key is valid, `reveal(key)` seeds an xorshift PRNG from the key and decrypts `BLOB`, which prints the flag.

**Inverting it.** Each program maps one input byte to one output byte, so there are only 256 possibilities per position. Brute-force each position independently, with no need to reverse the VM by hand:

```python
cands = [[c for c in range(256) if run(c, progs[k]) == TARGET[k]]
         for k in range(8)]
# [[110], [48], [114], [116], [104], [103], [52], [116]]
```

Exactly one candidate per position, giving the key **`n0rthg4t`**.

Then replicate `reveal()` (djb2-style seed → xorshift32 keystream → XOR with `BLOB`) to decrypt the flag. Entering `n0rthg4t` on the page shows it too.

### Takeaway
Client-side validation can't be secret. A custom VM or obfuscation only slows analysis, and when the per-byte search space is tiny, a divide-and-conquer brute force beats manual reversing. The flag is also derived from the key, so you must find the real key rather than patch the check.

---

## Notes

- Flags use the format `TBZ{...}`.
- All challenges are from an intentionally vulnerable CTF environment.
