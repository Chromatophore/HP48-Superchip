# 8XY6 & 8XYE
Bit shifts X register by 1, VIP: shifts Y by one and places in X, HP48-SC: ignores Y field, shifts X

## Initial notes:

On some original hardware, possibly the cosmac VIP, my understanding is that the ALU was basically memory mapped such that the last nibble of this instruction was pushed to it pretty much whatever happened, and the instruction set's inclusion of 0-7 & E are simply the documented as being the useful operations.

## Investigation:

Regardless, by using quirks.8o, we were able to observe that the rotation quirk is present in Chip48. If we investigate the source code for it, we can see the following:

i8xy6:		; VX := VX >> 1; carry in VF  
		call.3		alusetup  
		move.b		c,a  
		srb.b		c  
		move.2		c,@d0		; store result  
		move.b		a,c		; use low bit of original byte for carry  
		jump.3		lsbcarry  
...  
i8xye:		; VX := VX << 1; carry in VF  
		call.3		alusetup  
		add.a		c,c  
		move.2		c,@d0  
		jump.3		savecarry  

The alusetup routine loads the values of register X and Y into the HP48's c and a registers respectively. The Human readable version of these routines is thus:

VX := VX >> 1, carry in vF  
	Load vX -> C, vY -> A  
	Move byte value of C to A (to keep it safe)  
	Shift Right Byte in C   
	Write 2 nibbles of C to @d0, the memory address of vX  
	Move byte in A into C  
	3 nibble offset jump to lsbcarry  

VX := VX << 1; carry in VF  
	Load vX -> C, vY -> A  
	Add 5 nibbles of C to C (shame as a shift left for our purposes)  
	Write 2 nibbles of C to @d0, the memory address of vX  
	3 nibble offset jump to savecarry  

Obviously, these completely ignore the value of vY, so it's pretty clear to see why vY is not used. This behavior can be traced to Chip-48 and these routines were not changed for Superchip 1.0.

In Superchip 1.1, some unrelated elements were changed - ALU Setup routine was changed about a bit, but, these two shift functions have exactly the same code. You might also wonder, how is the carry value actually set?

lsbcarry and savecarry point at pretty much the same code: the difference is that lsb carry shifts the nibbles of C to the right twice before running the savecarry code - savecarry takes the lowest bit of C and stores it into vF. The Chip-48 & Schip1.0 code looks like this:

savecarry:  
		srn.a		c			; extract the carry byte  
		srn.a		c  
lsbcarry:  
		move.p2		#01,a		; use low bit only  
		and.b		a,c				  
		push.a		c  
		call.3		varfpd0		; get a pointer to VF  
		pop.a		c  
		move.b		c,@d0		; store the carry byte  
		retclrc  

Hopefully this is straight forward enough to not need an explanation beyond the existing comments. However, this routine is used every time the carry flag might need to be set with the ALU related ops, and, one of the ops in here, specifically move.p2 #01,a which loads a value of 1 into a, is quite a large instruction and a somewhat slow operation if the value is large, and also running varfpd0 blows out the A and C register so we need to work with the stack, which costs 9 cycles a piece. For superchip, 1.1 this routine was rewritten to try and eek out some more speed for these frequently called routines. First, here is the disassembly for the above code:

0047C  CSR     A 					# savecarry   
0047E  CSR     A 					# Shifting C right by whole nibbles twice  
00480  LA      01 					# set bit 1 of A to true  
00487  C=C&A   B 					# And look at only that bit of C  
0048B  RSTK=C 						# Push it  
0048D  GOSUB   00415 				# Get vF pointer from varfpd0  
00491  C=RSTK 						# Pop C  
00493  DAT0=C  B 					# Write carry bit as a byte to vF  
00496  RTNCC 						# Return, clearing carry bit  

And, here is the disassembly for the Schip1.1 code:

003C7  CSR     A 					# savecarry.   
003C9  CSR     A 					# Shifting C right by whole nibbles twice  
003CB  A=0     A 					# Blow out A. # NB: Why is this going from 003CB to F? A=0 should be 2 nibbles  
003CF  A=A&C   B 					# This seems ineffective? And A with C? A is 0?  
003D3  C=R4 						# Set C to v0 address...  
003D6  D0=C 						# Stuff that in d0...  
003D9  D0=D0+  15 					# Add 30 to d0, which is going to point us at vF's byte  
003DC  D0=D0+  15  
003DF  DAT0=A  B 					# Save the value of A in there. How the hell does this even work?  

\# What byte code is actually here?  
\# ...F6F6D0E40E6611C...  
\# F6 = CSR  
\# F6 = CSR  
\# D0 = A=0  
\# E4 = A=A+1 						# <<< Who could have guessed...  
\# 0E66 = 0Ea6 = A=A&C B  
\# Yeah OK, so, that's cool, A=A+1 is literally just missing completely here. What kind of dissassembly is this?  

003E2  RTNCC 						# Return, clearing carry bit  

## Fixing it

In order to fix this quirk, we can refer back to the source code, for the basic part of the operations. The main thing we need to make sure of is that C ends up containing the carry information.

i8xy6:		; VX := VX >> 1; carry in VF  
		call.3		alusetup  
		move.b		c,a		  
		srb.b		c  
		move.2		c,@d0		; store result  
		move.b		a,c		; use low bit of original byte for carry  
		jump.3		lsbcarry  
  
Shift right seems like we could fix it in place. If we change move.b c,a to move.b a,c, we'll replace the value in c with that in a (so we replace vX with vY) and keep the value in a that we later &= with 0x01 to set the carry flag.

So looking this up in the Saturn guide, A=C (b) is AE8 and C=A (b) is AEA. Changing byte value 0x8A at 0x458 to 0x88 shoul change this function to use Y in right shifts. However, there's already a C=A a few lines below, and it reads that C=A (b) is AE6, so, I have some concerns! Changing the byte to 0x86 at 0x458 yields C=A in the dissassembly so that is what we're going with.

Shift left however is a more complicated matter:

i8xye:		; VX := VX << 1; carry in VF  
		call.3		alusetup  
		add.a		c,c  
		move.2		c,@d0  
		jump.3		savecarry  

While this could be a simple fix - we could just use A instead of C, when we execute savecarry, we'll have the unchanged value of C still, which isn't going to do us much good. As a result, we need to get out of here so we can move the code around without worrying about disrupting all the existing method calls. In theory this might be fine, because all these jump.3 and call.3 are all relative jumps.

Anyway here's the plan:
Load the address of our custom code into C
Push the address of our custom code to the stack
Jump to alusetup (instead of call) - it still has a return, so it'll pop the address we pushed off the stack and then run the code at our new area.

Is there room for that in 14 nibbles?

Load address into C:  
34#####  
Push C  
06  
# Jump alusetup  
6###  

That's 13 nibbles, so, that'll fit just fine. Is it faster or slower than just calling another subroutine instead?
LC with n=5, 3 + 1.5 * 5 = 10.5  
RSTK=C = 9 cycles  
GOTO = 14 cycles  
GOSUB is 15  
RTN is 11 cycles  

We would be having 11 + 9 + 14 = 34 for the above vs:

gosub alusetup = 15  
gosub customshift = 15 + 11 for the return = 26  
goto savecarry (our other routine would need this too) = 0  

so that would total 41, which means the other solution is faster, so, let's write that I guess.

And I guess the code that we'd want to run when we get there:

add.a a,a  
move.2 a,@d0  
move.b a,c  
jump.3 savecarry  

Let's get this done now even though it will probably be able to be stuffed into the space a different function will blow out will replace. Shift Left is located at 008C0, ALU Setup is located at 00392. The binary ends at 010EA, which is byte 0x882 (the file literally ends there). So let's give this a go - loading 6 nibbles into C just because that way we won't upset the decompiler, value of 0010EA...

008C0 -> 0x46D aligned, existing code:

7ECA C6 15C1 6CFA  
35AE0100 06 67CA  
ala:  
53EA10006076AC  

And our code we'll want to execute at address alpha would be:  
C4 1581 AE6 6???  

But because we're at the end of the binary, it's too far to jump with the 3 nibbles, so we'll have to use golong, which will be 8Cwxyz - this only costs 3 more cycles so it's not that much of a cost, and maybe we can rectify it later. SaveCarry is at 003C7 so we need to jump back to that from it looks like 10f3 + 7? 4346 -> 967
  
So now we're saying:  
C4 1581 AE6 8C1D2F  

So our dissassembly is now looking like:

008C0  LC      0010EA 			<- Address of our custom code  
008C8  RSTK=C 					<- wedging it into the return stack  
008CA  GOTO    00392 			<- ALU Setup being called with a goto insted of gosub so return will our C  
...  
010EA  A=A+A   A 				<- Double A  
010EC  DAT0=A  2 				<- Store it in Dat0  
010F0  C=A     B 				<- Copy it to C... hmm. no  
010F3  GOLONG  003C7  
010F9  @  

Made a bit of an error here! Copying A to C as a byte isn't going to work since we need the 3rd nibble.
Should have done this instead:

C4 1581 D6 8C2D2F  
->  
010EA  A=A+A   A  
010EC  DAT0=A  2  
010F0  C=A     A  
010F2  GOLONG  003C7  
010F8  @  

Looking good! Let's give it a burl! Test program: http://johnearnest.github.io/Octo/index.html?gist=ac35eb1db0d44ece9169df169140df63

OK So that's great: Crashes immidietly. Reason: vF might be insane garbage? Setting vF to 0 in advance fixes the problem. So yeah that's interesting. New Program: http://johnearnest.github.io/Octo/index.html?gist=f86495d81f2cbfe82018b8e3a453a384

OCTO results:  
1 1  
1 0  
4 1  
4 0  

SCHIP results:  
2 1  
2 0  
2 1  
2 0  

SCHPC results:  
1 1  
1 0  
1 0  
1 0  

Well, shift right is fine, but shift left is not working at all. Is the strategy working correctly at all or is it falling through and not running the code? v0 <<= v1 with v0 as 81 v1 as 82 strongly implies nothing has happened. Perhaps a valid strategy is to change the code to do what it normally did at this point. Having done this, it appears significantly like the code isn't running at all. I'm also feeling like I should probably include the higher byte except: I'd need to use a shift to observer it - I guess I could use BCD.

Let's go for maybe a more reliable method: A big ol jump out to where we want to be. Do the normal ALU load subroutine, then just a big jump out to where we're at

008C0 -> 0x46D aligned, existing code:  
7ECA C6 15C1 6CFA  
7ECA 8C4280 C6 C6  
ala:  
e7acc824082020  

This change yields  
1 1  
1 0  
4 0  
4 0  

Which is closer. I've tried changing the code to as follows:

010EA  C=A     A  
010EC  C=C+C   A  
010EE  DAT0=C  2  
010F2  GOLONG  003C7  

Which really really should work, but, still yields 4 0 4 0. Perhaps being able to relocate and not use GOLONG will make a difference but for now this seems like a dead end.