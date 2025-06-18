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
| 1 Extract TLS secrets | `strings -n16 -el memory.raw \| grep -E "CLIENT_RANDOM" > keylog.log`       | ~1,800 lines of TLS keys (UTF-16 format crucial)             |
| 2 Decrypt PCAP        | Add keylog.log to Wireshark Preferences → Protocols → TLS                   | TLS streams now decryptable                                   |
| 3 Find VNC password   | `tshark -Y 'vnc.client_message_type == 4 && vnc.key_down == 1' -T fields -e vnc.key` | Password: `th3_ir0n_p0tat0_guid3s_us<3`                       |
| 4 Locate POST upload  | Wireshark filter: `http.request.method == POST`                             | One POST to `upload.gofile.io` (packet 8181)                 |
| 5 Extract 7z archive  | Save packet 8181, carve 7z from HTTP body (remove webform headers)          | Password-protected 7z archive                                |
| 6 Unlock archive     | Open 7z with password from VNC keystrokes                                   | Unlocked archive reveals Python source                       |
| 7 Read flag           | Open extracted Python file                                                  | `C1{the_ir0n_p0tat0_checks_r3ferences}`                       |

That seven‑step path solves the challenge in \~15 minutes.  I reached step 2 quickly, but then went off‑track.

## 2  What *I* Tried During the CTF

| Attempt                                                                                                                      | Rationale                                                                                          | Outcome                                          |
| ---------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- | ------------------------------------------------ |
| **Volatility3 dumpfiles / pagecache**                                                                                        | Maybe the attacker left the ZIP in RAM.                                                            | No matching files recovered.                     |
| **Foremost / Binwalk carve of** `memory.raw`                                                                                 | Brute‑force for ZIP headers.                                                                       | Found dozens of corrupt fragments; none rebuilt. |
| **Tshark** `--export-objects http`                                                                                           | Auto‑extract HTTP bodies.  Returned *empty* directory (because the traffic still looked like TLS). | Assumed no HTTP payload existed → mistake!       |
| **Hunted VNC clipboard (port 5900)**                                                                                         | Saw VNC packets; searched `vnc.client_cut_text` / `vnc.server_cut_text`.                           | Found password but missed its significance for archive. |
| **Tcp stream brute‑grep** (`follow,tcp,ascii,#`)                                                                             | Maybe the flag was in clear ASCII.                                                                 | Searched all 40 streams—nothing.                 |
| **Hex‑encoded / Base‑64 flag in RAM**                                                                                        | Last‑minute hail‑mary: decode every `C1[0‑9A‑F]+`.                                                 | Lots of junk; no valid flag.                     |

I burned \~2 hours on side paths instead of re‑examining **why the HTTP export was empty.**

---

## 3  Where I Went Wrong

1. **TLS 1.2 vs 1.3 assumption** — I thought the traffic might be TLS 1.3 (which ignores `SSLKEYLOGFILE`), so I didn't trust the decrypted PCAP enough to look for HTTP.
2. **Blind faith in** `--export-objects` — The first decryption attempt still showed "Encrypted Alert" packets; `export-objects` produced nothing, reinforcing my bias that "there's no HTTP".
3. **UTF-16 encoding oversight** — I used `strings -a -n12` but Windows stores strings as UTF-16. The correct command `strings -n16 -el` would have extracted ~1,800 TLS key lines instead of just 26.
4. **Missing the connection** — I found the VNC password (`th3_ir0n_p0tat0_guid3s_us<3`) in clipboard traffic but didn't realize it was meant for unlocking the exported archive.
5. **Rabbit‑holes** — Once I latched onto VNC and corrupt ZIP fragments, sunk‑cost bias kept me digging instead of stepping back.

The critical error was using wrong `strings` parameters for Windows UTF-16 data, which prevented proper TLS decryption. With correct extraction, the HTTP POST to `upload.gofile.io` and VNC password connection would have been obvious.

---

## 4  Impact on the Scoreboard

**This challenge directly cost me a podium finish.** At **16:18** I was sitting in **2nd place** (2,675 pts), but by the **18:00** end of the CTF I'd dropped to **4th place**. Had I captured it (it was one of two remaining flags—alongside a 200 pt Forensics challenge), I would have vaulted to **2nd overall**, just 200 points shy of first's perfect score.  Final standings were:

| Place | User       | Score |
| ----- | ---------- | ----- |
| 1     | cameron    | 3175  |
| 2     | Tyler W-23 | 2875  |
| 3     | David R-81 | 2875  |
| 4     | Moneer     | 2675  |

## 5  Key Takeaways

- **Always verify decryption truly succeeded** — a single missing `CLIENT_RANDOM` line kept HTTP data opaque my first pass.
- **Export‑objects is blunt; inspect headers manually** with Wireshark's Follow‑Stream if zero objects export.
- **Side‑channels are plan B.** Clipboard, DNS, or hex encodings rarely trump plain exfil files.
- **Step back & re‑triage** when many rabbit holes produce nothing—time is the scarcest resource during live CTF.

---

### Flag

```text
C1{the_ir0n_p0tat0_checks_r3ferences}
```

