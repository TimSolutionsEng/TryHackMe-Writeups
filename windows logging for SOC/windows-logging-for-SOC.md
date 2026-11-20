# TryHackMe: Windows Logging for SOC

## Overview

I completed the TryHackMe Windows Logging for SOC room, where I practised using Windows Event Viewer and Sysmon logs to investigate suspicious activity on a Windows machine.

The exercise helped me understand how Windows records authentication events, user management activities, file creation, network connections and PowerShell command history.

## 1. Windows Security Event IDs

I learned that Windows Security logs contain event IDs that help analysts investigate authentication and account-related activities.

Some of the important event IDs I studied were:

| Event ID | Meaning | Investigation purpose |
|---|---|---|
| 4624 | Successful logon | Identify successful authentication |
| 4625 | Failed logon | Investigate failed login attempts |
| 4720 | User account created | Identify unexpected account creation |
| 4732 | Member added to a security-enabled local group | Investigate privilege changes |

### RDP brute-force investigation

I investigated failed login attempts against the Windows machine.

I learned that:
- Event ID 4625 records failed logon attempts.
- Repeated failures from one IP address can indicate a brute-force attempt.
- Logon Type 10 identifies Remote Interactive logons, commonly associated with Remote Desktop Protocol (RDP).
- Event details can help identify the source IP, target account and authentication activity.

I identified the IP address responsible for the brute-force activity during the exercise.

## 2. Sysmon Event IDs

I also worked with Sysmon logs to investigate suspicious system activity.

The following event IDs were particularly useful:

| Event ID | Meaning | What I investigated |
|---|---|---|
| 1 | Process creation | Processes started on the system |
| 3 | Network connection | Connections initiated by processes |
| 11 | File creation | Files created by processes |
| 15 | FileCreateStreamHash | File stream creation and associated hashes |
| 22 | DNS query | Domain name queries made by processes |

### Malware investigation

During the exercise, I investigated a suspicious downloaded file and followed its activity.

My investigation involved:

1. Identifying the downloaded file using Sysmon Event ID 15.
2. Examining file creation activity and using the `TargetFilename` field to identify the persistence file.
3. Using Event ID 3 to investigate the malware's network connection.
4. Using Event ID 22 to investigate DNS queries and identify the domain associated with the malicious IP address.

This helped me understand how different event IDs can be correlated to reconstruct suspicious activity.

## 3. PowerShell History

I learned that PowerShell command history can provide useful evidence during an investigation.

The PSReadLine history file is commonly stored at:

`C:\Users\Administrator\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt`

This file records commands entered into interactive PowerShell sessions.

During the exercise, I:
- Located the Administrator's PowerShell history file.
- Examined the commands stored in the file.
- Identified the first command executed as `Get-ComputerInfo`.
- Learned to inspect file properties when investigating the history file's creation date.

### Limitations of PowerShell command history

PowerShell command history is useful, but it has limitations:

- It records commands rather than their output.
- It does not provide a complete record of script execution.
- It may not capture every PowerShell activity.
- It can survive system reboots, unless manually deleted.

This means command history should be correlated with other logs when investigating an incident.

## 4. Key Lessons Learned

Through this room, I improved my understanding of:

- Reading and filtering Windows Security logs.
- Understanding the purpose of Windows Event IDs.
- Investigating failed authentication attempts and RDP activity.
- Recognising suspicious account creation and group membership changes.
- Using Sysmon to investigate process, file, network and DNS activity.
- Correlating events to investigate possible malware activity.
- Locating and analysing PowerShell command history.
- Understanding why multiple log sources are important during incident investigations.

## 5. Practical Takeaway

One important lesson I took from this exercise is that a single event rarely tells the entire story.

For example, a failed login event may indicate an unsuccessful authentication attempt, but correlating multiple events can reveal a brute-force attack, a subsequent successful login and potentially unauthorised activity.

As I continue developing my blue-team and SOC analysis skills, I want to become better at connecting these events into a clear investigation timeline.

---

**Room:** TryHackMe – Windows Logging for SOC  
**Focus:** Windows Security Logs, Sysmon and PowerShell History  
**Learning Area:** Blue Team / Security Operations Centre (SOC)
