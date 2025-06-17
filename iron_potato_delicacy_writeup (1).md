# Iron Potato Delicacy — Hard‑Challenge Write‑Up

> **Category:** Crypto / Steganography   |   **Points:** 300 (Hard)

---

## 0  Scenario Overview

While searching the Iron Potato vault, our team recovered a single file, \`what_is_pizza.hex\`.\
The CTF brief gave two crucial hints:

1. Every Supreme‑Leader memo begins with the clear‑text prefix \`SECRET MEMO:\`.
2. His antique “tractor typewriter” rotates its **rotor one step for every key‑press**.

Our mission was to decrypt that file and extract the hidden directive (the flag) (the flag).

---

## 1  Recon & File Reconnaissance

| Check           | Result                                                                    |
| --------------- | ------------------------------------------------------------------------- |
| `file` command  | ASCII text, **hex dump** only (no binary footers)                         |
| Size            | 512 bytes                                                                 |
| Entropy         | Near‑uniform, but entirely printable — indicates **simple stream cipher** |
| Known‑plaintext | `SECRET MEMO:` crib at offset 0                                           |

---

## 2  Static Analysis

```bash
# Convert hex → raw bytes for inspection
xxd -r -p what_is_pizza.hex > pizza.bin
hexdump -C pizza.bin | head  # first 16 bytes
```

We observed that each ciphertext byte was shifted by an *incrementally growing* value. Plotting `(cipher − plain)` against the index produced a perfect line with slope +1.

---

## 3  Rotor Cipher Model

The behaviour matches a rotor starting at 0 and adding +1 each keystroke:

\(c_i = (p_i + i) \pmod{256} \qquad\Longrightarrow\qquad p_i = (c_i - i) \pmod{256}\)

No extra key material is involved; the keystream is fully deterministic.

---

## 4  One‑Shot Python Decryptor

```python
#!/usr/bin/env python3
"""Iron Potato rotor‑cipher solver"""
import pathlib

ct_hex = pathlib.Path("what_is_pizza.hex").read_text().strip()
ct     = bytes.fromhex(ct_hex)                    # hex → bytes
pt     = bytes((b - i) & 0xFF for i, b in enumerate(ct))

pathlib.Path("memo.txt").write_bytes(pt)         # save for later
print(pt.decode(errors="replace")[:120])         # sanity preview
```

Running the script produced a clean ASCII memo beginning with the expected prefix.

---

## 5  Flag Extraction

Scrolling to the signature block revealed:

```
... please deliver the Iron Potato delicacy immediately.
FLAG: C1{1R0NP0T4_LF5R}
—End of Memo—
```

The exact, case‑sensitive flag is:

```text
C1{1R0NP0T4_LF5R}
```

Submitting that string solved the challenge.

---

## 6  Lessons Learned

- **Low‑entropy keystreams are brittle** – a 12‑byte crib fully recovers the stream.
- **Known‑plaintext ≠ harmless** – predictable headers make encryption moot.
- **Minimal tooling wins** – we solved with `xxd`, `hexdump`, and 10 lines of Python.

