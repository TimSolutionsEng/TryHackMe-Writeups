TryHackMe: The Waiting Room

Investigation Write-up

1. Introduction

The Waiting Room is a digital forensics investigation involving a suspicious email and a series of artifacts that can be used to understand a potential security incident.

The investigation focuses on examining email evidence, identifying suspicious attachments, and understanding how different forensic artifacts can be used to reconstruct an incident.

My objective was to approach the room as a digital forensic investigator: begin with the available evidence, examine it carefully, identify anything suspicious, and determine what further investigation would be necessary.

This write-up documents my investigation process, the observations I managed to make, the challenges I encountered, and the lessons I gained.

2. Understanding the Investigation Environment

I accessed the TryHackMe investigation environment through its browser-based lab machine.

The investigation artifacts were available in the Desktop artifacts directory.

The files I identified included:

* vendor.eml – an email file containing the suspicious message.
* Sysmon-incident.evtx – a Windows event log containing system activity.
* incident.pcap – a network capture that could be examined for suspicious communications.
* tools/ – a directory containing investigation utilities.

I first wanted to understand what each artifact could tell me before attempting to draw conclusions.

I recognised that these files represented different sources of forensic evidence:

* Email evidence could help establish how an incident started.
* System event logs could reveal processes and other activity on the affected machine.
* Network traffic could help identify communications between the machine and external systems.

Combining these sources would make it possible to build a more complete incident timeline.

3. Beginning with Email Analysis

I decided to begin with the email because suspicious emails are often an important starting point when investigating possible phishing or malware incidents.

The supplied email was stored in .eml format.

Instead of opening it as an ordinary text document, I used Python’s email module to parse it.

The purpose of this approach was to inspect the message structure and identify its headers, body, and attachments.

The following Python code was used:

import email
from email import policy
with open("vendor.eml", "rb") as f:
    msg = email.message_from_binary_file(
        f,
        policy=policy.default
    )
print("FROM:", msg["From"])
print("TO:", msg["To"])
print("SUBJECT:", msg["Subject"])
for part in msg.walk():
    if part.get_content_disposition() == "attachment":
        print("FILENAME:", part.get_filename())
        print("CONTENT TYPE:", part.get_content_type())
        payload = part.get_payload(decode=True)
        if payload:
            print("SIZE:", len(payload))

4. Examining the Email Attachment

During the email examination, I identified an attachment with the following properties:

Property	Observation
Filename	Remittance
Content type	application/onenote
Size	22,096 bytes

This was an important observation because the attachment was a OneNote document.

OneNote files can contain more than ordinary notes. Depending on their contents, they may contain embedded objects, links, scripts, or other elements that require careful examination.

I therefore treated the attachment as potentially suspicious rather than assuming it was a harmless business document.

At this stage, I had identified the attachment and inspected its metadata, but I had not completed its extraction or internal examination.

5. What I Intended to Investigate Next

After identifying the attachment, the next stage would have been to examine its contents and correlate any suspicious activity with the other supplied evidence.

I identified three important areas for further investigation.

A. OneNote attachment examination

The attachment would need to be extracted and examined to determine:

* Whether it contained embedded files or objects.
* Whether any scripts or executable content were present.
* Whether there were suspicious links or references.
* Whether its contents could explain subsequent system activity.

B. Windows event log examination

The Sysmon-incident.evtx file could be examined to identify:

* Process creation events.
* Parent and child process relationships.
* Suspicious command execution.
* File creation and modification activity.
* Network connections initiated by processes.

Sysmon is particularly useful because it provides detailed information about activities occurring on Windows systems.

C. Network traffic examination

The incident.pcap file could be opened using Wireshark or examined with command-line network analysis tools.

The investigation would focus on:

* Connections to unfamiliar hosts.
* HTTP requests and responses.
* Unexpected downloads or uploads.
* Communication patterns that might correlate with suspicious processes.

The most important part would be correlating the timestamps and observations from these different sources.

6. Challenges Encountered

One of the main challenges I encountered was the instability of the browser-based TryHackMe lab machine.

Although I could access the investigation files and begin examining the email, the environment repeatedly failed during the investigation.

This prevented me from completing the attachment extraction and the remaining forensic analysis.

I learned that having the correct tools and investigation plan is not always enough. The investigation environment itself must also be reliable.

I also learned the importance of separating confirmed observations from assumptions. Identifying a suspicious attachment does not automatically prove what it contains or what actions it performed.

7. What I Learned

This investigation helped me understand several important digital forensics concepts.

1. Email files contain useful forensic evidence.

An .eml file is not simply a message. It contains structured information, including headers, message content, and potentially multiple attachments.

2. Attachment metadata is an important starting point.

The filename, content type, and size can provide useful clues, but they are not sufficient to establish whether a file is malicious.

3. OneNote files deserve careful examination.

Documents that appear ordinary can contain embedded content that may be relevant to a security investigation.

4. Different forensic artifacts complement one another.

Email evidence, Windows event logs, and network captures provide different perspectives of an incident.

A conclusion supported by multiple independent sources is generally stronger than one based on a single artifact.

5. Investigation conclusions must be evidence-based.

It is important to distinguish between what was directly observed, what is suspected, and what still needs verification.

8. My Investigation Status

At the time of writing, I had successfully:

* Accessed the TryHackMe lab environment.
* Located the supplied investigation artifacts.
* Identified and parsed the email file using Python.
* Identified the OneNote attachment named Remittance.
* Examined its content type and size.

The remaining attachment, Windows event log, and network traffic investigations were not completed hands-on because of problems with the lab environment.

I therefore do not present those stages as completed findings.

9. Final Reflection

The Waiting Room introduced me to an investigation approach that starts with seemingly ordinary evidence and gradually builds an understanding of a possible security incident.

One of my biggest takeaways was the importance of connecting evidence from different sources rather than analysing each artifact in isolation.

Although I could not complete every stage of the room, the initial email examination gave me practical experience with Python-based email parsing and identifying suspicious attachments.

I also learned an important lesson about forensic reporting: a good investigation report should clearly explain what was examined, what was discovered, what could not be established, and what should happen next.

This investigation remains a learning exercise that I intend to revisit when the lab environment is working reliably.
