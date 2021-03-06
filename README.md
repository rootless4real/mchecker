Description:
====
The purpose of mchecker is to analyze a binary stream of data and to provide information on the data in the context of executability. This will eventually include a high-level yes/no desicion (executable or not). Executablity is decided by many hueristics (listed below). 

Analysis is done raw; without the use of dissasembly libraries. This is done to maintain that we cannot assume that the data will be aligned for us. Becuase of the previous non-assumption, we can analyze data in a packet fragment.

Hueristics:
====
Of the heuristics that are finished, a sample output is provided for actual code, random data, and a document. In the case of all of these examples, the binary file 'cli' from Linux Mint 17 x64 was used for code (file size about 17MB), a 17MB file called 'random' is used for random data, generated by: dd if=/dev/urandom of=random bs=1024 count=17000, and the document is the 17MB Intel Users Manual Volumes 1,2, & 3 (it's a PDF).

###XOR reg, reg (Finished):###
If we wanted to set a register equal to 0 (to initilize it or just have the value be 0), we may think to use the operation (MOV EAX, 0), assuming our register is EAX. Although this makes sense, this is a 5 byte-ish opeartion. We can acheive the same effect in 2 bytes with (XOR EAX, EAX); XORing any value with itself produces the result of 0. Compilers know this. Due to this, and that programs often set registers to the value of zero, especially with loops and ECX, we may expect to see a high amount of these XORing a register with itself. Given that this is just 2 bytes we are looking for, false positives are possible. There are two ways to identify false positives:

Compare with expected amount of false positives (Dumb-Luck Analysis):
With several variations of opcodes to xor a register with itself, it is expected that by dumb-luck, we should see a false-positive 1 in every 2,000 bytes (2,114 bytes). Becuase of this, look for signifigantly more than this ratio

Entropy:
If we are getting dumb-luck false positives, it's expected to have a even distribution between the registers being xored with themselves (EAX, EBX, and ECX xored with themselves an equal amount of time). Even distribution is a sign that this is less likely to be code (even if there is a higher than dumb-luck ratio).

Example of Code:

![XOR Code](/images/xor1.png)

Example of PDF:

![XOR Non-Code](/images/xor2.png)

Example of Random:

![XOR Random](/images/xor3.png)

###One-Byte Histogram:###
There are statistically more common opcodes. MOV, PUSH, CALL, POP, CMP, and NOP account for more than half opcodes used. Regarding out-of-context byte analysis (not dissasembled) we also expect to see a lot of NULLs as well (usually as part of the operands, especially considering leading zero padding for large unsigned integers). Considering that this is not analysis on dissasembled code, the reliability isn't as high as other analysis, but this still provides value. 

For the Linux (Mint 17) examples; all files from $PATH (lots of ELF executables) were analyzed together. For the Windows examples, a search for "Programs" was performed on the whole filesystem and used. For Windows and Linux, these were fresh installs. This was done to give a stronger baseline of what should be expected from this output (and how code and non-code should compare).

Example of Linux 32-bit:

![Histogram](/images/byte4.png)

Example of Linux 64-bit:

![Histogram](/images/byte5.png)

Example of Windows7 32-bit:

![Histogram](/images/byte6.png)

Example of Windows7 64-bit:

![Histogram](/images/byte7.png)

Example of Code:

![Histogram](/images/byte1.png)

Example of PDF:

![Histogram](/images/byte2.png)

Example of Random:
![Histogram](/images/byte3.png)

###Triple MOV (Working, but unfinished):###
Not only is the MOV instruction very common, but we tend to see them grouped in chunks. This makes sense though, for example: We want to call a function and the function expects to see arguments on the stack (not uncommon), we usually must MOV these values to some registers before PUSHing them to the stack (this may be an argument for looking for consequtive PUSH's as well). In the case of this hueristic, we look for 3 MOV's in a row. Currently, this is the least reliable hueristic. It is also not very accurate; it currently doesn't take into account proper addressing and operand lengths of each variation of the MOV opcodes (it is currently a fuzzy guess). If reliability doesn't increase with an improvment on the addressing/operand sizing, then this hueristic will be removed.

Example of Code:

![3MOVs](/images/movs1.png)

Example of PDF:

![3MOVs](/images/movs2.png)

Example of Random:

![3MOVs](/images/movs3.png)

###CALL/RET Balance (Finished):###
CALL is an operation to essentially call a subroutine, when the code of the subroutine is done, the RET opcode is issued and control is returned to after the CALL was made. If we think about this, Every time we CALL a subroutine, we "mostly" expect a corresponding RET; we should see about as many CALLs as we see RETs. For whatever reason, we may see more CALLs than RETs for executable code, but to see more RETs than CALLs, this should be seen as unusual (non-code)

Example of Code:

![CALLRET](/images/callret1.png)

Example of PDF:

![CALLRET](/images/callret2.png)

Example of Random:

![CALLRET](/images/callret3.png)

###POP->RET (Finished):###
This is being seen as unreliable, as it is likely compiler dependent. However, I have noticed that it is at least somewhat common for RETs to be often preceded by POPs...So I count them.

Example of Code:

![POPRET](/images/popret1.png)

Example of PDF:

![POPRET](/images/popret2.png)

Example of Random:

![POPRET](/images/popret3.png)

###(CMP | TEST) -> (Jcc | MOVcc) (Not Started):###
This hueristic will take me some time (due to many variations), however, this is likely one of the most reliable (consistent and not compiler dependent) hueristics. It is common for a program to need to do conditional jumps (and sometimes conditional MOVs occur). Before doing a conditional operation, what do we need to do? We need to use an operation that somehow sets important EFLAGS bits (most commonly CMP and TEST).
