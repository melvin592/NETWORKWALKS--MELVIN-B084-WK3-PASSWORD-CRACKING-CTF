# Unlocking a Locked PDF with John the Ripper

> A lab guide for recovering the password of a **password-protected PDF** you own or are authorized to test, using **John the Ripper** and the wordlists provided in the Networkwalks lab.

---

## Table of Contents

1. [Overview](#overview)
2. [Objectives](#objectives)
3. [Lab Environment](#lab-environment)
4. [Prerequisites](#prerequisites)
5. [Step 1 — Inspect the PDF Encryption](#step-1--inspect-the-pdf-encryption)
6. [Step 2 — Extract the Hash with pdf2john](#step-2--extract-the-hash-with-pdf2john)
7. [Step 3 — Prepare the Wordlist](#step-3--prepare-the-wordlist)
8. [Step 4 — Run John the Ripper](#step-4--run-john-the-ripper)
9. [Step 5 — Show the Recovered Password](#step-5--show-the-recovered-password)
10. [Step 6 — Unlock the PDF](#step-6--unlock-the-pdf)
11. [Using the Networkwalks Built-in Wordlists](#using-the-networkwalks-built-in-wordlists)
12. [Alternative — Hashcat](#alternative--hashcat)
13. [Troubleshooting](#troubleshooting)
14. [Best Practices](#best-practices)
15. [Legal & Ethical Notice](#legal--ethical-notice)
16. [References](#references)

---

## Overview

A password-protected PDF encrypts its contents with a key derived from the user password. To recover the password, we:

1. Identify the encryption type.
2. Extract a crackable hash from the PDF.
3. Test candidate passwords from a wordlist (or generated masks) against that hash.
4. Use the recovered password to decrypt the file.

**John the Ripper (Jumbo)** includes `pdf2john`, which converts PDF encryption data into a hash format John can attack.

> ⚠️ Use this only on PDFs you created or have explicit permission to access. Recovering a password you do not own may be illegal.

---

## Objectives

- Inspect a PDF's encryption settings.
- Extract a John-compatible hash from a locked PDF.
- Run dictionary, rule-based, and mask attacks.
- Recover the password and decrypt the PDF.
- Document the process professionally.

---

## Lab Environment

| Component | Details |
|---|---|
| **OS** | Kali Linux (or any Linux with John Jumbo) |
| **Attacker tools** | John the Ripper Jumbo, pdf2john, qpdf |
| **Target** | `locked.pdf` (authorized lab file) |
| **Wordlists** | Networkwalks provided lists + `rockyou.txt` |
| **Output** | Decrypted `unlocked.pdf` |

---

## Prerequisites

### Install John the Ripper (Jumbo)

Kali ships with John, but the **Jumbo** build is required for PDF support:

```bash
sudo apt update
sudo apt install -y john pdf2john qpdf
```

Verify:

```bash
john --list=formats | grep -i pdf
```

You should see `pdf` in the output.

### Confirm pdf2john exists

```bash
which pdf2john
which pdf2john.pl
```

### Get a wordlist

Kali includes `rockyou.txt` (compressed):

```bash
sudo gunzip /usr/share/wordlists/rockyou.txt.gz
ls -lh /usr/share/wordlists/rockyou.txt
```

---

## Step 1 — Inspect the PDF Encryption

Check whether the PDF is encrypted and which algorithm it uses:

```bash
qpdf --show-encryption locked.pdf
```

Example output (abridged):

```text
R = 4
P = -3904
User password = 
Encrypted using AESv2
```

Also useful:

```bash
pdfinfo locked.pdf
exiftool locked.pdf | grep -i encrypt
```

**Why this matters:** PDF revisions (R2–R6) determine hash format and cracking speed. Modern AES-256 (R6) is much slower to attack than legacy RC4 (R2–R4).

---

## Step 2 — Extract the Hash with pdf2john

```bash
pdf2john locked.pdf > pdf.hash
```

If the command is not found, use the full path:

```bash
/usr/share/john/pdf2john.pl locked.pdf > pdf.hash
```

**Inspect the hash:**

```bash
cat pdf.hash
```

Typical format:

```text
locked.pdf:$pdf$4*2*40*-1*...
```

> John accepts the `filename:` prefix. For **Hashcat**, strip it:
>
> ```bash
> cut -d: -f2- pdf.hash > pdf.hashcat
> ```

---

## Step 3 — Prepare the Wordlist

### Use a single list

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt pdf.hash
```

### Combine the Networkwalks lists

If the lab provides multiple wordlists, merge them:

```bash
cat /path/to/networkwalks-list1.txt \
    /path/to/networkwalks-list2.txt \
    /usr/share/wordlists/rockyou.txt > combined.txt
```

Remove duplicates:

```bash
sort -u combined.txt -o combined.txt
```

### Locate the lab-provided lists

```bash
find / -iname "*networkwalks*" 2>/dev/null
find / -iname "*password*" -o -iname "*wordlist*" 2>/dev/null | head
```

---

## Step 4 — Run John the Ripper

### 4.1 Dictionary attack

```bash
john --wordlist=combined.txt pdf.hash
```

### 4.2 Dictionary + rules (recommended)

Rules mutate words (e.g., `password` → `P@ssw0rd123`):

```bash
john --wordlist=combined.txt --rules pdf.hash
```

### 4.3 Incremental (brute force)

```bash
john --incremental pdf.hash
```

> Slow for long passwords. Use only when the password is short or the search space is small.

### 4.4 Mask attack (known pattern)

If you know the password length/pattern (e.g., 4 lowercase letters):

```bash
john --mask='?l?l?l?l' pdf.hash
```

Common placeholders:

| Mask | Meaning |
|---|---|
| `?l` | lowercase letter |
| `?u` | uppercase letter |
| `?d` | digit |
| `?s` | symbol |
| `?a` | any printable |

### 4.5 Resume or check status

Press **Enter** while John runs to see progress. Resume later:

```bash
john --restore
```

---

## Step 5 — Show the Recovered Password

```bash
john --show pdf.hash
```

Example:

```text
locked.pdf:Secret123

1 password hash cracked, 0 left
```

The password appears after the colon.

---

## Step 6 — Unlock the PDF

### Option A — qpdf (recommended)

```bash
qpdf --password='Secret123' --decrypt locked.pdf unlocked.pdf
```

### Option B — pdftk

```bash
pdftk locked.pdf input_pw Secret123 output unlocked.pdf
```

### Option C — PDF viewer

Open the PDF in a viewer, enter the password, then **Print → Save as PDF**.

**Verify:**

```bash
pdfinfo unlocked.pdf
```

The output should no longer show `Encrypted: yes`.

---

## Using the Networkwalks Built-in Wordlists

The Networkwalks lab provides pre-built password lists. Typical usage:

| List | Suggested use |
|---|---|
| `common-passwords.txt` | First pass — fastest hits |
| `networkwalks.txt` | Lab-specific words and patterns |
| `names.txt` | Usernames, names, places |
| `rockyou.txt` | Large general-purpose list |

**Suggested order (fastest → slowest):**

```bash
# 1. Lab-specific lists
john --wordlist=/path/to/networkwalks.txt pdf.hash

# 2. Common passwords
john --wordlist=/path/to/common-passwords.txt pdf.hash

# 3. Rules on the small lists
john --wordlist=/path/to/networkwalks.txt --rules pdf.hash

# 4. Big list
john --wordlist=/usr/share/wordlists/rockyou.txt pdf.hash
```

> 💡 Start with small, targeted lists — they finish quickly and often succeed before the large list is needed.

---

## Alternative — Hashcat

For GPU-accelerated cracking:

```bash
# Strip the filename prefix first
cut -d: -f2- pdf.hash > pdf.hashcat

# PDF modes by revision:
#   10500 = PDF 1.4–1.6 (Acrobat 3–7)
#   10600 = PDF 1.7 Level 3 (Acrobat 9)
#   10700 = PDF 1.7 Level 8 (Acrobat 10–11)

hashcat -m 10500 pdf.hashcat /usr/share/wordlists/rockyou.txt
```

Show the result:

```bash
hashcat -m 10500 pdf.hashcat --show
```

> Check the mode with `hashcat --example-hashes | grep -A 5 PDF`.

---

## Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| `pdf2john: command not found` | Jumbo scripts not in PATH | Use `/usr/share/john/pdf2john.pl` |
| `No password hashes loaded` | Wrong hash format | Regenerate the hash; don't edit it |
| John says "0 guesses" | Empty wordlist | Verify the path and file contents |
| Very slow cracking | AES-256 / long password | Use rules/masks; try targeted lists first |
| `rockyou.txt` missing | Still compressed | `sudo gunzip /usr/share/wordlists/rockyou.txt.gz` |
| Password not found | Not in wordlist | Try rules, mask, or incremental mode |
| qpdf fails | Wrong password | Re-check `john --show` output |
| Hashcat "no hashes loaded" | Filename prefix included | Strip with `cut -d: -f2-` |

---

## Best Practices

1. **Start small** — targeted lists before `rockyou.txt`.
2. **Use rules** — cheap mutations dramatically improve hit rate.
3. **Know the pattern** — masks are far faster than blind brute force.
4. **Document commands and results** — reproducibility matters.
5. **Check encryption type first** — AES-256 changes expectations.
6. **Keep the original PDF** — always decrypt to a new file.
7. **Respect the law** — only crack files you own or are authorized to test.

---

## Legal & Ethical Notice

> This guide is for **authorized lab and educational use only**.
>
> - Only recover passwords from PDFs you created or have **explicit written permission** to access.
> - Do not use these techniques against third-party documents.
> - Follow applicable laws and organizational policy (e.g., CFAA, GDPR, computer-misuse statutes).

---

## References

- [John the Ripper (Jumbo)](https://www.openwall.com/john/)
- [pdf2john](https://github.com/openwall/john/blob/bleeding-jumbo/run/pdf2john.py)
- [qpdf Documentation](https://qpdf.readthedocs.io/)
- [Hashcat Example Hashes](https://hashcat.net/wiki/doku.php?id=example_hashes)
- [Kali Wordlists](https://www.kali.org/tools/wordlists/)
