# All-Pakistan TechBiz Security CTF: Write-ups
 
Write-ups for the **All-Pakistan TechBiz Security CTF**, an online, nationwide event organised by TechBiz Security on **30 October 2026** (6 hours, 12 challenges).
 
**Result:** 12/12 flags solved, 2,807 points, rank **#12**.
 
> **Full write-ups:** [ctf-writeups.md](ctf-writeups.md)
 
## Challenges
 
| ID | Challenge | Category | Difficulty | Core idea |
|----|-----------|----------|------------|-----------|
| C01 | [Hello, Recon](ctf-writeups.md#c01--hello-recon) | OSINT | Warmup | `robots.txt` leaks a hidden path |
| C02 | [Layers](ctf-writeups.md#c02--layers) | Crypto | Warmup | Base64 → ROT13 → hex, layered encodings |
| C03 | [View Source](ctf-writeups.md#c03--view-source) | Web | Warmup | Secret rebuilt from JS fragments in the bundle |
| C04 | Pick A Number | Web | Medium | Broken access control (IDOR) on invoices |
| C05 | [Loose Lips](ctf-writeups.md#c05--loose-lips) | Forensics | Medium | Plaintext HTTP in a pcap |
| C06 | [Hidden In Plain Sight](ctf-writeups.md#c06--hidden-in-plain-sight) | Stego | Medium | LSB steganography in the red channel |
| C07 | [Noise Floor](ctf-writeups.md#c07--noise-floor) | Forensics | Medium | Finding the one successful login in a brute-force log |
| C08 | [Amended](ctf-writeups.md#c08--amended) | Forensics | Medium | Secret recovered from unreachable git objects |
| C09 | [Rolling Code](ctf-writeups.md#c09--rolling-code) | Crypto | Hard | Breaking repeating-key XOR |
| C10 | [Small e](ctf-writeups.md#c10--small-e) | Crypto | Hard | Textbook RSA with `e = 3`, cube-root attack |
| C11 | [What Happened Here](ctf-writeups.md#c11--what-happened-here) | Forensics | Hard | Recovering a deleted file from a FAT16 image |
| C12 | [Checkpoint](ctf-writeups.md#c12--checkpoint) | Reversing | Hard | Inverting a small VM-based licence check |
 
The C04 write-up is still to be added.
 
## Skills covered
 
- **Web / OSINT:** `robots.txt` recon, reading page source and JS bundles, access-control flaws
- **Crypto:** layered encodings, repeating-key XOR cryptanalysis, RSA with a small exponent
- **Forensics:** packet captures, web access logs, deleted-file recovery on FAT, git history forensics
- **Stego:** LSB extraction guided by image metadata
- **Reversing:** analysing client-side JavaScript and inverting a custom virtual machine
## Tools
 
Python 3, Pillow, `base64`, `strings`, `git`, `awk`/`grep`, browser DevTools.
 
## Notes
 
- All challenges target systems provided by the organisers for this event. Nothing here should be used against systems you do not own or have permission to test.
- Flags use the format `TBZ{...}`.
- These write-ups were published after the event ended.
## Author
 
Mah Noor
