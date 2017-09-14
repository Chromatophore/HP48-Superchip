# Messing with the SCHIP Binary

So, it turns out I couldn't instantly recall quite a lot of the things I needed to know when returning to this project after a year. Shocking I know! As a result, I'm going to write down a lot of the important things that I figured out the first time and needed.

## Using the disassembly

I used dosbox to run [Decode 1.3](http://www.hpcalc.org/details/3780) to disassemble the calculator binaries (which you can get from the calculator if you need to, still using KERMIT, using Server mode on the HP48 and GET commands in Kermit). This program generates files that look like this:

```
0006B  D0=C
0006E  LC      7055B
00075  LA      38
0007C  GOSUB   001DD
00080  LC      70551
00087  LA      08
```

The 5 digit number at the start of the line is the nibble index of the command. This is important because the HP48 uses variable length opcodes and parameters, so there is no fixed addressing index/spacing. If you want to look at this byte code in the binary via a hex editor, the address values won't line up because they a) are nibble based (4 bits) rather than the byte based (8 bits) you'll see in the hex editor, and b) don't account for the HP48's binary header, which is 0xD bytes long.

The conversion process is however therefore quite simple: **Byte Address = (Nibble Address / 2) + 0xD**

### Finding commands (an example)

Given the above, if we take the nibble address, 0x00080, this would become 0x04D when converted. We also know from the addresses that we are looking at 7 nibbles worth of data, because the next command, LA 08, has the address of 0x00087. Looking now at the binary in question, going to this address and reading forwards 4 bytes, we find these bytes: 
```
43 51 05 87
```
Being vaguely familiar with the way the HP48 stores numbers, the way this command is going to work is to say, the LC instruction, N digits - 1, digit data in order of least significant to most significant. However you need to consider that while both the hex editor and the calculator are little endian, the hex editor displays the binary as little endian *bytes*, and so the nibbles with an individual byte are displayed in (for our purposes anyway) the wrong order. Consider the following bit stream separated in these two different ways:

```
0000111100001111
->
00001111 00001111 (as bytes) becomes
F0 F0
vs
0000 1111 0000 1111 (as nibbles) becomes
0 F 0 F
```
Now that we know this, let's read the data, reading out the least significant nibble first, and the most significant nibble second. The op code for LC is simply 3, so, this is how that maps in this case:
```
3 4 70551
A B CDEFG
<->
BA FG DE #C
43 51 05 87
```
If I had an editor that let me simply read nibbles in the correct order, well, I think that would make this way easier to visualise. The value, 8, paired with the 7 in the last byte, is the first value of the opcode for LA, which is 8082. If you wanted to change the value of this LC call, you would obviously need to make sure to preserve this 8 value when you rewrote the byte. If you needed to load a larger or smaller number, you'd need to adjust the nibble I marked as B.

### Things to bear in mind about DECODE
So, despite being my reliable go to, DECODE has at least a couple of problems. Sometimes it completely misses out commands, in particular increment calls seem to often be absent - keep an eye out for instructions taking up more address space than it seems like they should. Also, it has an off by one error for GOLONG addresses, because it calculates based on an incorrect assumption about the opcode + parameter length:
```
00B18  GOLONG  00675
vs
00674  ?ABIT=0 4
```
In this case, there simply isn't a 0x00675 to go to, it has miscalculated the ending address, I think because it believes this command is a nibble longer than it actually is.

Additionally, data blocks (like the fonts at the end of the binary) leave it very perplexed and provide dodgy output, which can be bad for us since instruction alignment is so important, but this problem usually resolves itself with only a few garbage results.

### The Header

The HP48 binary header needs to be intact and correct in order for a binary to run. If it isn't, it will simply dump onto the stack as a string and not execute. If you have an existing binary that works, you're fine to make any changes you want to it as there is no CRC or anything like that involved, however if you need to change the length, adding extra code to the end, you'll need to fix this header. Here are a few reference headers:

```
SC10
48504850 34382D4A CC2DF00801
SCHIP:
48504850 34382D45 CC2DE00E01
SCHPC:
48504850 34382D45 CC2D301601
Structure:
AAAAAAAA AAAAAABB CCCCDCDDDD
As ascii:
HPHP48-?????
```

The meat of this can be found in pieces across the internet, from I guess RPLMAN.doc to a variety of manuals and reference docs, but, for our purposes, I have compiled a simple 'fixitfixitfixit' description. The header will always begin with HPHP48-, then a 2 nibble ROM version in B maybe. The use of this I'm unsure about but note that SC10 has a different value from SCHIP. After that there will be the 5 nibble prologue in the C nibbles, which is 02DCC - the 'DOCODE' prologue. The important section for us though are the D nibbles. The value in D is a 5 nibble length of the binary. This value is in least significant order, and in our reference binaries reads:
```
0108F (SC10)
010EE (SCHIP)
01163 (SCHPC)
```

Hex fiend reports the size for our binaries as 0x852, 0x882 and 0x8BC bytes respectively, which when doubled, work out as 0x10A4, 0x1104 and 0x1178. So why the misalignment? The length excludes the A, B and C nibbles of the header, but includes itself. As a result, because of the otherwise fixed header, you can take your file size, double it, and subtract the nibbles of the earlier prologue, which total 21, or 0x15 nibbles. Additionally, if your program doesn't actually use the last nibble of your last byte, you can subtract that, too, although it might be safer to just include it/you might have to, since PCs are byte based systems. Also you may have no idea if your code is actually using it, in the case of SC10, the most significant nibble of the last byte is a 0, but it's a component of the last piece of data that is there, and is used.

At any rate, the function is: **Header size = File size * 2 - 15 - (last nibble unused)**

Example:
```
0x852 * 2 - 0x15 - 0 = 0x108F (SC10)
0x882 * 2 - 0x15 - 1 = 0x10EE (SCHIP)
0x8BC * 2 - 0x15 - 0 = 0x1163 (SCHPC)
```

## Things to know about certain commands
### GOSUB, GOTO, GOLONG, etc
In order to save binary space and make more efficient use of nibble by nibble ram reading speeds, there are several different ways to branch the calculators PC. GOTO, GOLONG, and GOVLNG are unconditional jumps, and GOSUB, GOSUBL, and GOSBVL. These, in order, use 3 signed relative, 4 signed relative, and 5 immediate nibbles to specify the resulting address. Despite the fact these commands are very similar, they have a weird variance in how they work.

The relative nibble values for GOTO and GOLONG count from the nibble address that the relative nibbles start. The relative nibble values for GOSUB and GOSUBL count from the very end of the instruction. So, for example:
```
000E5  GOTO    00724
6B3E16
6 XYZ thus 6E36 (1 nibble + 3 data = 4 total)
63E + E5 = 00723 (1 nibble off, data excluded)

00B18  GOLONG  00675 (actually 00674, decompiler error)
C8 5AFB
8C WXYZ thus 8CA5BF (2 nibble + 4 data = 6 total)
B18 + 7B5A - 8000 = 00672 (2 nibbles off, data excluded)

002A4  GOSUB   00374
C70C
7 XYZ thus 7CC0 (1 nibble + 3 data = 4 total)
2A4 + 0CC = 00370 (4 nibbles off, data included)

00A97  GOSUBL  0022C
81FE784F
8E WXYZ thus 8E F87F (2 nibble + 4 data = 6 total)
A97 + 778F - 8000 = 226 (6 nibbles off, data included)
```
I have no idea why but it is clearly demonstratable.

