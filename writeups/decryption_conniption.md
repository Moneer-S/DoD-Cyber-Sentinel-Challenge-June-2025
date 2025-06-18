# Decryption Conniption — Forensics Challenge Write‑Up

> **Category:** Forensics   |   **Points:** 300 (Hard)  
> **Status:** ❌ **NOT SOLVED DURING CTF** - Post-competition analysis

---

## 0  Scenario Overview

**Note:** I did not solve this challenge during the CTF competition. This write-up documents my failed attempts during the event and the correct solution I learned afterward.

HR unknowingly hired a North‑Torbian Python dev who may have siphoned our proprietary code.  I was given the following:

- **`memory.raw`** – a 4 GB Windows RAM image (captured after the incident)
- **`evidence.pcapng`** – full network capture of the workstation
- MD5 for the original evidence ZIP (**`f2d271efcf526…`**) for integrity.

The brief said an **`SSLKEYLOGFILE`** env‑var was deployed, so TLS traffic *should* be decryptable.  A flag was hidden somewhere inside the stolen source‑code archive.

*Write‑up adapted with insights from Chris Haller (Challenge dev) and [**his Medium write-up**](https://m4lwhere.medium.com/decryption-conniption-2025-correlation-one-ctf-e864f274ebb9).*

> **Goal:** prove the exfiltration and recover the flag from the code.

## 1  Initial Recon & Correct Approach (retrospective)

| Step                  | Tool / Command                                                              | Expected Result                                               |
| --------------------- | --------------------------------------------------------------------------- | ------------------------------------------------------------- |
| 1 Extract TLS secrets | `strings -a -n12 memory.raw \| grep -E "^CLIENT_RANDOM" > keys.log`         | 26 CLIENT\_RANDOM lines                                       |
| 2 Decrypt PCAP        | `tshark -r evidence.pcapng -o tls.keylog_file:keys.log -w decrypted.pcapng` | Plain‑text HTTP/2 now visible                                 |
| 3 Locate large POST   | Wireshark filter: `http2.data.data && http.request.method == "POST"`        | One 630 kB stream (ID 7) with `content-type: application/zip` |
| 4 Export the object   | *File → Export Objects → HTTP* (save as `source.zip`)                       | Valid ZIP containing stolen code                              |
| 5 Read flag           | `grep -R "C1{" source/`                                                     | `C1{the_ir0n_p0tat0_checks_r3ferences}`                       |

That five‑step path solves the challenge in \~10 minutes.  I reached step 2 quickly—but then went off‑track.

## 2  What *I* Tried During the CTF

| Attempt                                                                                                                      | Rationale                                                                                          | Outcome                                          |
| ---------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- | ------------------------------------------------ |
| **Volatility3 dumpfiles / pagecache**                                                                                        | Maybe the attacker left the ZIP in RAM.                                                            | No matching files recovered.                     |
| **Foremost / Binwalk carve of** `memory.raw`                                                                                 | Brute‑force for ZIP headers.                                                                       | Found dozens of corrupt fragments; none rebuilt. |
| **Tshark** `--export-objects http`                                                                                           | Auto‑extract HTTP bodies.  Returned *empty* directory (because the traffic still looked like TLS). | Assumed no HTTP payload existed → mistake!       |
| **Hunted VNC clipboard (port 5900)**                                                                                         | Saw VNC packets; searched `vnc.client_cut_text` / `vnc.server_cut_text`.                           | No flag. VNC was just remote desktop, not exfil. |
| **Tcp stream brute‑grep** (`follow,tcp,ascii,#`)                                                                             | Maybe the flag was in clear ASCII.                                                                 | Searched all 40 streams—nothing.                 |
| **Hex‑encoded / Base‑64 flag in RAM**                                                                                        | Last‑minute hail‑mary: decode every `C1[0‑9A‑F]+`.                                                 | Lots of junk; no valid flag.                     |

I burned \~2 hours on side paths instead of re‑examining **why the HTTP export was empty.**

---

## 3  Where I Went Wrong

1. **TLS 1.2 vs 1.3 assumption** — I thought the traffic might be TLS 1.3 (which ignores `SSLKEYLOGFILE`), so I didn't trust the decrypted PCAP enough to look for HTTP.
2. **Blind faith in** `--export-objects` — The first decryption attempt still showed "Encrypted Alert" packets; `export-objects` produced nothing, reinforcing my bias that "there's no HTTP".
3. **Rabbit‑holes** — Once I latched onto VNC and corrupt ZIP fragments, sunk‑cost bias kept me digging instead of stepping back.

A single glance at **content‑type headers** in Wireshark would have revealed the 630 kB ZIP immediately.

---

## 4  Correct Solution in 90 Seconds (after the fact)

```bash
# 1  Extract secrets
strings -a -n12 memory.raw | grep -E "^CLIENT_RANDOM" > keys.log

# 2  Decrypt capture
editcap --inject-secrets tls,keys.log evidence.pcapng decrypted.pcapng

# 3  Identify biggest HTTP object
capinfos -cd decrypted.pcapng | sort -k3 -nr | head   # stream 7

# 4  Export that stream
mkdir loot &&
  tshark -r decrypted.pcapng --export-objects "http,loot" -2 -R "tcp.stream==7"

# 5  Unzip & grep
unzip -q loot/*.zip -d loot/src
grep -R "C1{" loot/src
```

Output:

```
loot/src/stealer_client.py:FLAG = "C1{the_ir0n_p0tat0_checks_r3ferences}"
```

Submit **`C1{the_ir0n_p0tat0_checks_r3ferences}`**.

---

## 5  Impact on the Scoreboard

**This challenge directly cost me a podium finish.** At **16:18** I was sitting in **2nd place** (2,675 pts), but by the **18:00** end of the CTF I'd dropped to **4th place**.  I never solved this challenge during the competition. Had I captured it (it was one of two remaining flags—alongside a 200 pt Forensics challenge), I would have vaulted to **2nd overall**, just 200 points shy of first's perfect score.  Final standings were:

| Place | User       | Score |
| ----- | ---------- | ----- |
| 1     | cameron    | 3175  |
| 2     | Tyler W-23 | 2875  |
| 3     | David R-81 | 2875  |
| 4     | Moneer     | 2675  |

## 6  Key Takeaways

- **Always verify decryption truly succeeded** — a single missing `CLIENT_RANDOM` line kept HTTP data opaque my first pass.
- **Export‑objects is blunt; inspect headers manually** with Wireshark's Follow‑Stream if zero objects export.
- **Side‑channels are plan B.** Clipboard, DNS, or hex encodings rarely trump plain exfil files.
- **Step back & re‑triage** when many rabbit holes produce nothing—time is the scarcest resource during live CTF.

---

### Flag

```text
C1{the_ir0n_p0tat0_checks_r3ferences}
```

