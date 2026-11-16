# TryHackMe – Classic Passwd

## Room Overview

**Room:** Classic Passwd  
**Platform:** TryHackMe  
**Category:** Reverse Engineering  
**Environment:** Kali Linux VM  
**Tools Used:** `file`, `strings`, `ltrace`, `objdump`, `GDB`

## 1. Introduction

The objective of this room was to investigate a Linux executable and understand how its authentication mechanism worked.

Rather than treating the executable as a black box, I approached it through static and dynamic analysis. I wanted to understand how the program handled user input, what it compared the input against, and how its execution eventually reached another function.

The exercise provided practical experience with reverse engineering an ELF executable without having access to its original source code.

## 2. Identifying the Executable

My first step was to identify the type of file provided by the challenge.

Command:

```bash
file Challenge_1609966715991\(1\).Challenge
```

The output identified the file as:

- ELF 64-bit executable.
- x86-64 architecture.
- Dynamically linked.
- Position Independent Executable (PIE).
- Not stripped.

### What I learned

An ELF file is a standard executable format used by Linux.

The fact that the executable was not stripped was particularly useful because it retained function names and other symbols that helped during disassembly and debugging.

PIE was also important. It meant that the executable could be loaded at different memory addresses during execution, something I encountered later when setting GDB breakpoints.

## 3. Searching for Interesting Strings

I used the `strings` command to inspect readable text embedded in the executable.

Command:

```bash
strings Challenge_1609966715991\(1\).Challenge
```

Among the output, I noticed several interesting strings:

- `Insert your username:`
- `Welcome`
- `Authentication Error`
- `strcmp`
- `strcpy`
- `THM{%d%d}`
- `vuln`
- `main`

### What I learned

The `strings` command can reveal useful clues about a program without executing it.

The presence of `strcmp` suggested that the program might compare user input against another string.

The format string `THM{%d%d}` was particularly interesting because it suggested that the program would eventually print two decimal integers.

However, finding these strings did not explain how the program reached that output. I needed to investigate its execution.

## 4. Investigating Program Behaviour with Ltrace

I initially discovered that `ltrace` was not installed in my Kali environment.

I installed it using:

```bash
sudo apt update
sudo apt install ltrace -y
```

I then executed:

```bash
ltrace ./Challenge_1609966715991\(1\).Challenge
```

After entering a test username, I observed a call to:

```text
strcmp("mrrobot", "AGB6js5d9dkG7")
```

The program subsequently displayed:

```text
Authentication Error
```

### What I learned

`ltrace` allows us to observe calls made by a program to library functions.

In this case, it exposed the arguments passed to `strcmp()`.

The function `strcmp()` compares two strings:

- It returns zero when the strings are equal.
- It returns a nonzero value when they differ.

This revealed that the program was comparing the supplied username against a hardcoded string.

It also showed that the authentication check was an important part of the program's execution flow.

## 5. Disassembling the Executable

To understand the internal program logic, I used `objdump`.

Command:

```bash
objdump -d -M intel ./Challenge_1609966715991\(1\).Challenge | less
```

The command disassembled the executable and displayed its instructions using Intel syntax.

I focused on the functions `main()`, `vuln()` and `gfl()`.

### Understanding vuln()

Inside `vuln()`, I identified instructions responsible for:

- Constructing the string used in the comparison.
- Displaying the username prompt.
- Reading user input.
- Calling `strcpy()`.
- Calling `strcmp()`.
- Displaying either a welcome message or an authentication error.

The important observation was that successful authentication allowed execution to continue, while unsuccessful authentication terminated the program.

### Understanding main()

The disassembly showed that `main()` called:

```text
vuln()
gfl()
```

This established the execution sequence.

The program first performed its authentication check. If that function returned successfully, execution continued into `gfl()`.

### Understanding gfl()

The `gfl()` function contained nested loops and comparisons.

I noticed two important comparison instructions:

```asm
cmpl $0x638a78,-0x4(%rbp)
```

and:

```asm
cmpl $0x2130,-0x8(%rbp)
```

These compared local variables against specific hexadecimal constants.

I also identified instructions that loaded values into registers before the program called `printf()`.

This suggested that the function was using loop conditions to determine the integers eventually passed to the output function.

## 6. Investigating the Format String

I used another strings command to locate the relevant strings and their offsets:

```bash
strings -a -t x Challenge_1609966715991\(1\).Challenge | grep -E 'THM|Authentication|Welcome'
```

The output showed:

```text
201f Welcome
2028 Authentication Error
203d THM{%d%d}
```

The format string was:

```text
THM{%d%d}
```

I learned that `%d` is a format specifier used by `printf()` to display a decimal integer.

Therefore, the program was preparing to print two integer values inside a string.

The remaining task was to understand where those values came from.

## 7. Debugging with GDB

I used GDB to examine the program while it was running.

Command:

```bash
gdb -q ./Challenge_1609966715991\(1\).Challenge
```

My first attempt was to set a breakpoint using the address obtained from the earlier disassembly:

```gdb
break *0x12c9
```

However, GDB returned:

```text
Cannot insert breakpoint 1.
Cannot access memory at address 0x12c9.
```

### Understanding the problem

The executable was a PIE binary.

This meant that its actual memory address during execution could differ from the address shown in the original disassembly.

Instead of relying on the original address, I used the function name:

```gdb
break gfl
```

I then started the program:

```gdb
run
```

After entering the previously observed comparison string, the program displayed:

```text
Welcome
```

GDB successfully stopped inside `gfl()`.

### Examining the instructions

I inspected the instructions around the current execution point:

```gdb
x/12i $rip
```

This displayed the next twelve assembly instructions.

I could now see the loop initialization, comparisons, increments and jumps.

I also identified the instructions responsible for preparing the arguments passed to `printf()`.

To inspect the actual runtime address of the call, I used:

```gdb
x/8i 0x5555555552bd
```

This revealed the call instruction at the relocated address:

```text
0x5555555552c9
```

I then placed a breakpoint at that address:

```gdb
break *0x5555555552c9
```

And continued execution:

```gdb
continue
```

GDB stopped immediately before the `printf()` call.

## 8. Examining the Registers

At this point, I needed to understand which values the program was about to print.

On the x86-64 Linux System V calling convention, the first integer or pointer arguments to a function are passed through registers.

For this particular `printf()` call:

- `RDI` contained the format string.
- `RSI` contained the first integer argument.
- `RDX` contained the second integer argument.

I inspected the registers using:

```gdb
p/d $esi
p/d $edx
```

I also inspected the format string:

```gdb
x/s $rdi
```

This confirmed that the format string was:

```text
THM{%d%d}
```

To verify the integer values against the hexadecimal constants identified during disassembly, I ran:

```gdb
p/x $esi
p/x $edx
```

The values matched the comparison constants in `gfl()`.

### What I learned

This was one of the most important parts of the investigation.

I was able to connect three different pieces of evidence:

1. The assembly showed the comparisons performed by the loops.
2. The register instructions showed how the values were prepared for `printf()`.
3. GDB allowed me to inspect those values at runtime.

This confirmed that the integers were generated through the loop logic rather than simply being read directly from user input.

## 9. Verifying the Program Output

After inspecting the registers, I allowed the program to continue:

```gdb
continue
```

The program executed the `printf()` instruction and displayed its output before terminating normally.

This confirmed my understanding of how the function constructed its output.

## 10. Challenges Encountered

### GDB could not insert the original breakpoint

Initially, I attempted to use an address taken directly from the disassembly.

This failed because the executable was position independent.

I resolved this by setting a breakpoint using the function name, allowing GDB to locate the function at its runtime address.

I then inspected the instructions and used the relocated address to set the breakpoint at the correct location.

### Understanding the nested loops

At first, the loop structure was not immediately obvious.

I had to examine the comparison instructions, conditional jumps and increment instructions to understand how execution progressed.

This helped me appreciate how high-level loops are represented in assembly language.

### Understanding registers

I also had to understand how function arguments are passed through registers on a 64-bit Linux system.

Learning the roles of `RDI`, `RSI` and `RDX` helped me understand how the program passed its format string and integer values to `printf()`.

## 11. Key Lessons Learned

Through this room, I gained practical experience with:

- Identifying Linux ELF executables using `file`.
- Extracting embedded strings using `strings`.
- Observing library function calls using `ltrace`.
- Disassembling executables using `objdump`.
- Understanding basic x86-64 assembly instructions.
- Recognising function calls and program execution flow.
- Understanding conditional jumps and nested loops.
- Debugging PIE executables with GDB.
- Setting breakpoints using function symbols and runtime addresses.
- Inspecting registers and understanding function arguments.
- Connecting static analysis with dynamic analysis.

## 12. Conclusion

Classic Passwd provided a practical introduction to reverse engineering.

The most valuable part of the exercise was learning not to rely on a single tool or technique.

`strings` provided initial clues, `ltrace` helped reveal the authentication comparison, `objdump` exposed the program's internal logic, and GDB allowed me to observe the program while it executed.

By combining these techniques, I was able to understand how the executable controlled authentication, entered another function, processed its loop conditions and prepared its final output.

The room also introduced me to a practical debugging challenge involving PIE executables and runtime memory addresses.

Overall, this exercise improved my understanding of Linux executables, assembly language, function calls and dynamic analysis.
