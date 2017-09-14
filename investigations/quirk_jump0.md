# BNNN (aka jump0)
Sets PC to address NNN + v0 - VIP: correctly jumps based on value in v0. SCHIP: also uses highest nibble of address to select register, instead of v0 (high nibble pulls double duty). Effectively, making it jumpN where target memory address is N##. Very awkward quirk.

## Initial notes:

While attempting to test octo games on the HP48, I discovered several were crashing outright. By putting in key pauses I was able to work out that it was crashing when running Jump0 commands.

## Investigation:

Writing a small test program to investigate yielded the answer: Instead of using v0, schip was using v2 as the offset variable. However, trying to make use of this in a larger program didn't work either. A heavy feeling fell on my heart and I realised what was happening: The small program was within the range 0x200-0x2ff, with the highest nibble being 2. Many commands read their register in from this nibble, and investigating the source code for Chip-48 showed the following:

```
ib:        ; parametric jump to NNN+V0
        call.3        varpd0
        call.3        nnnc
        clr.a        a
        move.b        @d0,a
        add.a        a,c
        move.a        c,r3        ; assign to pc
        retclrc
```

The culprit here is varpd0, this is the subroutine call that is used in most commands to read a register X in from #X##. This clearly shows two things: 1) that chip-48 through to Superchip 1.1 erroniously read this nibble for their jump tables, and 2) Nobody ever tested or used this command because all Chip-8 programs start at 0x200, meaning there is no normal situation where v0 is used.

Investigating the source suggests that there is a better method to call: var0pd0. It's possible this was the inteded function call, and it was typod, or it could have simply been an oversight. Var0pd0 is only used otherwise in save/load I and saving to userflags.

Therefore, if possible the 'fix' for this is to locate var0pd0 in schip, and call that instead of varpd0.

## Fixing it:

So, looking for the function calls from saving/loading I turned up something: One of schip 1.1's speed tweaks was to simplify the whole process and no longer call a var0pd0 subroutine, since it turns out, the pointer for v0 is very easy to obtain (it's simply at the beginning of the data area, r4). However, there is no room in our code area for adding a the extra instructions, so without a subroutine we could call, I was quite concerned that a more advanced strategy would be needed.

However, along with Varpd0, there are a selection of other commands that load in eg vF and the delay/buzzer registers, so, I looked through to locate the disassembly for these routines. And, this investigation was rewarded. Naturally, as soon as I located it, I checked to see if anything actually called it: the answer was, of course, 'no'. However, it is perfectly functional dead code, and does what is required (at 6 nibbles, 2 more than the small subroutine jump space). It is significantly slower to have the subroutine call but jump0 isn't exactly used much.

```
00351  C=0     A 				# This is varpd0
00353  C=DAT1  P
00357  C=C+C   A 				# this is cvarpd0
00359  A=R4
0035C  C=C+A   A
0035E  D0=C
00361  RTN
00363  A=R4 					# I think this is it! var0pd0!
00366  D0=A 					# Nothing calls it, so, it's a good candidate
00369  RTN
0036B  C=0     A 				# This is varFpd0
0036D  LC      F
00370  GOTO    00357
```

Going to BNNN's handler:

```
008D9  GOSUB   00351 		# Jump0, this is calling varpd0 # CAn we rewrite this?
008DD  GOSUB   00333		# nnnc
008E1  A=0     A 			# Set A to 0
008E3  A=DAT0  B 			# Read a byte of data from DAT0, which is the Vn pointer
008E6  C=C+A   A 			# Add this value to C
008E8  R3=C 				# Set r3 to C - R3 is the PC
008EB  RTNCC 				# Returns from the command
```

And adjusting the initial GOSUB to point to 00363 instead yielded perfect success and removed the quirk from SCHPC. The nibbles, located at 0x479 in the binary (there is an odd preceding nibble at this point), are:
```
3747A7
->
3768A7
```

GOSUB is 7xyz, where xyz is a relative subroutine call of value zyx, and if the highest bit is set then it's -2048 + the rest of value, apparently. Also note that it works from the address of the end of the 7xyz instruction/the next command's address, naturally.

So, the original value of 8DD + A74 = 2269d - 2048d + 628d = 849d = 0x351  
Hence, the new value is 768A, as 8DD + A86 = 2269d - 2048d + 646d = 867d = 0x363  