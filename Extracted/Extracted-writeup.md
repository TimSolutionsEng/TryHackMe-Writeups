# TryHackMe: Extracted

## 1. Room Overview

The Extracted room focused on digital forensics and incident response (DFIR). The investigation involved suspicious network traffic that had not been ingested into the SIEM, requiring direct examination of a packet capture to identify what had happened.

The objective was to investigate the traffic, recover stolen data, identify the KeePass master password and retrieve the final flag.

## 2. Tools Used

- Kali Linux
- Wireshark / TShark
- TCPFlow
- Python 3
- John the Ripper
- KeePassXC
- KeePass password dumper

## 3. Initial Investigation

I started by examining the provided PCAPNG file using `capinfos` and TShark.

The capture contained more than 53,000 packets, with most of the traffic consisting of TCP data.

I identified three important communication ports:

- **1339:** PowerShell script transfer.
- **1337:** KeePass process memory dump transfer.
- **1338:** KeePass database transfer.

This gave me a direction for the investigation.

## 4. PowerShell Script Analysis

I exported the HTTP objects from the capture and recovered a suspicious PowerShell script.

The script revealed that the compromised workstation was checking for a running KeePass process.

If KeePass was running, it used ProcDump to create a memory dump. The script then XOR-encoded the dump with `0x41`, Base64-encoded the result and transmitted it over port 1337.

It also retrieved the KeePass database, XOR-encoded it with `0x42`, Base64-encoded it and transmitted it over port 1338.

This established that both the memory dump and the database had been exfiltrated.

## 5. Recovering the KeePass Database

I reconstructed TCP stream 2 and extracted its payload.

After removing the TCP-related hexadecimal representation, I decoded the Base64 data and reversed the XOR operation using `0x42`.

The recovered file was identified as a valid KeePass KDBX database.

I then used `keepass2john` to extract its password hash.

My initial attempt involved using the RockYou wordlist with John the Ripper, but it did not recover the password.

I also experimented with extracting strings from the memory evidence. This produced a huge amount of data without immediately revealing the answer.

## 6. The Difficult Part: Memory Dump Reconstruction

This was the most challenging stage of the investigation.

The memory dump was transferred over TCP stream 1. However, the capture contained missing sections.

When I attempted to decode the entire Base64 stream, the process failed because of missing data markers.

I initially struggled to understand how to reconstruct the dump without corrupting its contents.

After investigating the missing sections, I worked on recovering individual fragments, decoding them and reversing the XOR operation using `0x41`.

Eventually, I recovered a fragment with a valid Windows minidump header.

Using the `minidump` utility, I inspected the dump and confirmed that it contained KeePass process modules, including `KeePass.exe`.

This was an important breakthrough: I was no longer dealing with random binary data. I had recovered genuine KeePass process memory.

## 7. The Breakthrough: KeePass Password Recovery

After further investigation, I discovered CVE-2023-32784, a KeePass vulnerability that can leave fragments of a master password in process memory.

I obtained a Python implementation of the KeePass password dumper and ran it against the recovered memory fragment.

The tool returned:

`●NoWaYIcanF0rGetThis123`

The first character was not recovered by the tool.

Rather than guessing, I used John the Ripper with a mask attack to test possible characters in that position.

Command:

`john --mask='?aNoWaYIcanF0rGetThis123' ../keepass_hash.txt`

John successfully cracked the hash and identified the missing character.

The recovered master password was:

`?NoWaYIcanF0rGetThis123`

## 8. Retrieving the Flag

With the password recovered, I installed KeePassXC and opened the reconstructed database using:

`keepassxc-cli ls ../recovered.kdbx`

Among the database entries was one titled **You win!**

I inspected the entry using:

`keepassxc-cli show ../recovered.kdbx "You win!"`

The flag was stored in the Notes field.

## 9. Lessons Learned

This investigation taught me several important things:

- Network traffic can reveal much more than communication patterns; it can expose stolen files and credentials.
- TCP stream reconstruction is essential when investigating data exfiltration.
- Missing capture data can complicate forensic recovery, but individual fragments may still contain valuable evidence.
- PowerShell scripts should be investigated carefully because they can automate data collection and exfiltration.
- Process memory can contain sensitive information that is not available from ordinary network traffic.
- Vulnerability research can provide the missing link when traditional password recovery techniques fail.

Most importantly, I learned not to abandon an investigation simply because the first few approaches fail. Understanding the evidence and changing the approach made the difference.

---

**Room:** TryHackMe – Extracted  
**Focus:** Digital Forensics and Incident Response (DFIR)  
**Status:** Completed
