Dead Endpoint – TryHackMe Walkthrough

1. Introduction

Dead Endpoint was one of my practical digital forensics investigations on TryHackMe.

The room involved investigating a compromised environment by examining a disk image, analysing web server logs, identifying a suspicious executable and understanding how encrypted files could be recovered.

What made this investigation interesting was that it required combining several areas of cybersecurity. I could not rely on one tool or one source of evidence. I had to move between filesystem analysis, log investigation, binary analysis and cryptography to understand what had happened.

In this write-up, I document my investigation process, the commands I used, the challenges I encountered, how I understood the technical concepts and what I learned.

I have intentionally left out the room’s flags and direct answers so that someone else can use this as a learning guide without having the challenge spoiled.

2. Understanding the Investigation

Before starting the investigation, I needed to understand what I was looking for.

The main challenge was to investigate the provided evidence and establish how the suspicious activity was connected to the files and executable found in the system.

I divided the investigation into several stages:

1. Disk image examination.
2. Filesystem investigation.
3. Web server log analysis.
4. Suspicious executable identification.
5. Static analysis and reverse engineering.
6. Understanding the encryption process.
7. Recovering and validating encrypted evidence.

Breaking the investigation into stages made it easier to manage because each discovery provided information that helped with the next stage.

3. Disk Image Examination

The first thing I did was examine the provided disk image.

A disk image is a copy of a storage device. It can contain partitions, filesystems, directories, deleted files and other evidence.

I used mmls, which is part of The Sleuth Kit, to examine the partition layout.

mmls ~/Desktop/artifacts/disk.img

Why I used mmls

I initially needed to understand how the disk was organised.

A disk image does not necessarily begin directly with the filesystem I want to investigate. It may contain multiple partitions, each starting at a different sector.

mmls helped me identify the partition structure and the starting offsets.

Something I learned:

When using forensic tools to examine a disk image, the partition offset matters. If I use the wrong offset, I may end up examining the wrong part of the image or receive errors.

I therefore made sure I understood the partition layout before proceeding.

4. Filesystem Investigation

After identifying the partitions, I moved to filesystem analysis.

I used fls, another tool from The Sleuth Kit, to examine the contents of the filesystem.

My first command was:

fls -r -p -o <partition_offset> ~/Desktop/artifacts/disk.img > root.txt

This recursively listed filesystem entries and saved the results in a text file.

I also wanted to examine deleted entries separately, so I used:

fls -r -p -d -o <partition_offset> ~/Desktop/artifacts/disk.img > deleted.txt

Understanding the commands

* -r recursively examines directories.
* -p displays the full path of entries.
* -o specifies the partition’s starting offset.
* -d lists deleted entries.
* > redirects the output into a file.

What I was looking for

I wanted to understand the directory structure, identify unusual files and locate anything that could help explain the incident.

I learned that deleted files can still be valuable during an investigation. Deletion does not necessarily mean that all information about a file has disappeared.

I also learned that filesystem analysis requires patience. A large filesystem can contain thousands of entries, so saving the results and narrowing down the investigation is much more practical than trying to inspect everything manually.

Something that initially confused me

I had to understand the difference between a partition offset and an inode number.

They are not interchangeable:

* A partition offset helps identify where a filesystem begins within the disk image.
* An inode identifies a particular filesystem object.

Understanding this distinction became important when I later needed to extract files from the image.

5. Investigating the Web Server Logs

After examining the filesystem, I moved to the web server logs.

Logs are important in digital forensics because they provide a record of activity that occurred on a system.

I located the Nginx log directory:

/var/log/nginx/

The investigation involved examining the access and error logs.

I paid attention to:

* Request timestamps.
* Requested URLs and endpoints.
* HTTP request methods.
* Response status codes.
* Repeated or unusual requests.
* Relationships between different events.

Why I examined the logs

I wanted to understand how the web application had been accessed and whether there were signs of suspicious activity.

An access log can reveal what requests reached a server, while an error log can provide additional information about application or server problems.

I learned that a single log entry rarely tells the whole story.

For example, an unusual request might appear harmless when examined alone. However, when it occurs around the same time as suspicious file activity, it becomes more important.

This taught me to correlate evidence instead of drawing conclusions from isolated events.

What I learned

Log analysis is not just about finding unusual IP addresses or status codes. It is about reconstructing events and understanding how different activities relate to one another.

6. Identifying and Extracting the Suspicious Executable

During filesystem investigation, I identified an executable in an unusual hidden directory associated with the application.

I wanted to examine it without executing it.

I used icat to extract the file from the disk image using its inode number.

The general command structure was:

icat -o <partition_offset> disk.img <inode_number> > extracted_file

Why I used icat

When examining a forensic disk image, a file may not be directly accessible through the normal filesystem.

icat allows a file to be extracted using its inode number.

This was useful because I could obtain the executable for further examination without having to boot or modify the original evidence.

After extraction, I checked the file type:

file extracted_file

I then examined readable strings:

strings -a extracted_file

What these tools helped me understand

file helped identify the executable format and architecture.

strings helped reveal readable information embedded in the binary, including references to functions and libraries.

I discovered references to cryptographic functions, which suggested that encryption was an important part of the executable’s behaviour.

Important lesson: Finding a suspicious executable is only the beginning. I needed to understand what it did before drawing conclusions about its role in the incident.

7. Static Analysis and Understanding the Executable

At this stage, I wanted to understand how the executable worked.

I used objdump to disassemble it:

objdump -d extracted_file

Disassembly converts machine instructions into a more readable assembly representation.

I also used the information obtained from strings to guide my investigation towards interesting functions.

What I investigated

I paid attention to references associated with:

* OpenSSL cryptographic functions.
* Random number generation.
* System time retrieval.
* File timestamp modification.
* Memory cleansing.

These references helped me identify areas of the program that were worth investigating more closely.

What confused me

One of the difficult parts was understanding that seeing a cryptographic function in a binary does not automatically explain how the encryption works.

For example, finding a reference to an AES function tells me that the program may use AES, but it does not tell me:

* Where the encryption key comes from.
* How the nonce is generated.
* What data is encrypted.
* How the authentication tag is handled.
* Whether additional authenticated data is used.

I had to look beyond the function names and examine how the functions were called and how their arguments were prepared.

This was where disassembly became useful.

What I learned

Static analysis helps investigate a program without executing it.

I also learned that readable strings and disassembly complement each other. Strings provide clues, while disassembly helps explain the relationships between operations.

8. Understanding the Cryptographic Implementation

This was one of the most technically challenging parts of the room.

The executable contained references to AES-GCM, a mode of operation that provides authenticated encryption.

I needed to understand the different components involved before I could investigate the encrypted files.

AES-256-GCM

AES is a symmetric encryption algorithm. Symmetric encryption means the same secret key is used for encryption and decryption.

GCM stands for Galois/Counter Mode. It provides:

* Confidentiality: protecting the contents of the data.
* Integrity: helping detect unauthorised modifications.
* Authentication: verifying that the encrypted data and associated authenticated information are valid.

Understanding the key and nonce

One thing I had to be careful about was the difference between a key and a nonce.

The encryption key is the secret material used by the encryption algorithm.

The nonce is a value used during the encryption operation. It is not a replacement for the encryption key.

I also encountered random number generation through OpenSSL.

Initially, it was easy to assume that every random value generated by the program must be part of the encryption key. However, examining how the values were used helped me understand their different purposes.

Understanding pseudorandom number generation

I also investigated a custom pseudorandom number generation routine.

This required looking at the constants and operations used in the assembly instructions.

I learned that a pseudorandom generator produces a sequence of values using an initial state and a mathematical procedure.

A seed is the initial input that determines the generated sequence.

Understanding this concept was important because the executable’s key-generation process depended on a sequence of generated values.

Rather than simply recognising the algorithm’s name, I had to understand how its output was used by the program.

9. Examining the Encrypted File Structure

After investigating the executable, I moved to the encrypted files.

I wanted to understand how the encrypted data was organised.

I examined the binary structure rather than assuming that the entire file contained ciphertext.

The file contained a header followed by encrypted content.

I investigated the purpose of different sections, including:

* Magic bytes.
* Version information.
* Length information.
* Nonce.
* Authentication tag.
* Encrypted payload.

Why this mattered

Encrypted files often contain metadata that is necessary for decryption.

For authenticated encryption, the decryption process needs the appropriate nonce and authentication tag.

I also learned about Additional Authenticated Data (AAD).

AAD is information that is authenticated as part of an encryption operation but is not itself encrypted.

This helped me understand why the header could be important even though it was not part of the encrypted payload.

What initially confused me

I initially needed to understand where the actual encrypted content started.

It is easy to treat an encrypted file as one continuous block of ciphertext. However, the file may contain a structured header with information required to perform decryption.

Examining the file’s bytes and understanding the format helped me separate metadata from the encrypted content.

10. Reproducing the Decryption Process

Once I understood the executable’s encryption process and the encrypted file structure, I used Python to reproduce the necessary operations.

I used the Python cryptography library to work with AES-GCM.

My general approach was:

1. Examine the encrypted file.
2. Understand how the executable generated and used cryptographic material.
3. Reproduce the relevant generation logic.
4. Extract the required encryption metadata.
5. Perform authenticated decryption.
6. Validate the recovered output.

I learned that AES-GCM decryption is not successful merely because a program produces some bytes.

The authentication check must also succeed. If the key, nonce, authentication tag or authenticated data is incorrect, the decryption operation should fail.

Why Python was useful

Python allowed me to reproduce the relevant operations and automate the recovery process.

It was particularly useful for:

* Reading binary files.
* Working with byte offsets.
* Performing bitwise operations.
* Reproducing mathematical routines.
* Processing encrypted data.
* Validating recovered files.

This was a good example of how programming knowledge can support digital forensic investigations.

11. Validating the Recovered Evidence

After processing the encrypted files, I needed to establish whether the recovered output was valid.

I did not want to assume that a successful script execution automatically meant that the recovery was correct.

I checked the output using file identification and image validation techniques.

For an image, I considered:

* Whether the file had a valid image signature.
* Whether it could be opened.
* Whether the file structure was consistent with the expected format.

This reinforced an important forensic principle:

Always validate your findings before treating them as evidence.

A file being generated does not necessarily mean that it has been correctly recovered.

12. Challenges and Things That Confused Me

This room had several areas that required me to slow down and understand what was happening.

12.1 Understanding partition offsets

At first, it was important to distinguish the partition’s starting offset from other filesystem identifiers.

I learned that forensic tools need the correct filesystem location before they can interpret the contents properly.

12.2 Understanding inode numbers

I had to understand how files could be extracted directly from a disk image using inode numbers.

This helped me appreciate that filesystems maintain metadata that can be useful even when files are not accessed through a normal directory path.

12.3 Interpreting log entries

I learned not to treat every unusual HTTP request as proof of an attack.

The important part was understanding the context, comparing timestamps and looking for relationships between events.

12.4 Reading assembly instructions

This was one of the more demanding parts because assembly is less readable than a high-level programming language.

I had to connect function names, constants, registers and operations to understand what the executable was doing.

12.5 Understanding cryptography

The distinction between keys, nonces, seeds, authentication tags and AAD initially required careful attention.

I learned that these components have different responsibilities and that understanding the complete encryption process is necessary before attempting recovery.

13. What I Learned From This Room

This investigation helped me connect several technical areas that I had previously studied separately.

Digital Forensics

* How to examine disk images.
* How to identify partitions.
* How to investigate filesystem contents.
* How to extract files using inode numbers.

Linux and Command-Line Skills

* Using forensic command-line utilities.
* Redirecting command output into files.
* Working with binary files.
* Investigating hidden directories and executable files.

Network and Log Analysis

* Understanding web server access logs.
* Investigating HTTP requests.
* Correlating timestamps and system events.

Reverse Engineering

* Identifying executable formats.
* Using strings to find useful references.
* Using disassembly to investigate program behaviour.
* Understanding why static analysis is useful.

Cryptography

* Understanding AES-GCM.
* Distinguishing keys from nonces.
* Understanding pseudorandom number generation.
* Understanding authentication tags and AAD.
* Using Python to investigate encrypted data.

Software Engineering

* Breaking a complex problem into smaller stages.
* Using scripts to automate repetitive tasks.
* Validating program output.
* Understanding how low-level code and libraries work together.

14. Final Reflection

Dead Endpoint was a valuable exercise because it required more than knowing commands. I had to understand what each command was doing, why a particular piece of evidence mattered and how one discovery could guide the next stage of the investigation.

The most challenging part for me was moving from identifying cryptographic functions to understanding how the executable actually used them.

It also showed me how useful programming knowledge can be in cybersecurity. Python helped me automate parts of the investigation, while understanding lower-level operations made it possible to interpret the executable’s behaviour.

My biggest takeaway was that a good investigation should be evidence-driven. Instead of jumping to conclusions, I needed to examine the available evidence, understand its meaning, connect related findings and validate the final results.

This room strengthened my interest in digital forensics, reverse engineering and defensive security.

⸻

Platform: TryHackMe
Room: Dead Endpoint
Investigation Areas: Digital Forensics, Log Analysis, Static Analysis and Cryptography
Status: Completed

Challenge flags, direct answers, secret values and answer-specific details have intentionally been excluded.
