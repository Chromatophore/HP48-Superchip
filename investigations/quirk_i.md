# FX55 & FX65

Saves/Loads registers up to X at I pointer - VIP: increases I, HP48-SC: I remains static

## Initial notes:

When you load/store n registers to I, on the cosmac VIP, your I pointer would incriment by n such that you could read through all of your memory if chunks of n by repeatedly loading. However, superchip roms exist that demonstrate that they expect I to remain static when reading, allowing you to read the same piece of memory over and over again.

## Investigation:

So, this is probably my favourite quirk that I've checked out. If you're familiar any with Chip-8 Emulation history, this quirk, along with the bit shift quirk, are the big two differences that come up when dealing with schip programs. One of the first things that was done when my calculator was able to run binaries was to run a selection of test programs that displayed that it was behaving the ways we expected, this quirk included, and it clearly was so we all patted ourselves on the back about how right we were. The I indeed remaining unchanged, and well, ultimately I'll show that this is what happens, but when I was originally investigating this, I got a bit of a surprise.

If you think about the way chip-8 is implimented on the HP48, you can realise why this quirk might arise - the I register is stored in r0, and has to be moved into the working registers and then into one of the d0 or d1 pointer register to be made use of. This means that you're not using the value directly, so when you might have to modify it to point at your next piece of data, you don't end up modifying the saved data in r0 at all. And I expected to see this bear out. Since we have source code for 2 of the binaries, this is where I started (this is from sc10):

```
; setup subroutine for fx55 or fx65 instruction
;
; In: d0 points to VX
; Out: d0 points to to V0, a points to VX, and d1 points to MI
copysetup:
	swap.a	c,d0		; copy d0 to c
	move.a	c,d0
	push.a	c			; save pointer to the last variable
	call.3	var0pd0		; point d0 to first variable (v0)
	move.a	r0,c		; get I
	call.3	virtophy		;;; virtophy takes a chip 8 address and converts to the calculator's memory
	move.a	c,d1		; point d1 to data at I
	pop.a	c
	move.a	c,a			; now a contains pointer to last var.
	ret

...
ifx55:	; save vars in memory
	call.3	copysetup
savelo:
	move.b	@d0,c		; read a byte from variable
	move.b	c,@d1		; store in MI
	swap.a	c,d0		; get d0 to c
	move.a	c,d0
	brge.a	c,a,doret1	; are we ready yet?
	add.a	2,d1
	add.a	2,d0
	move.a	r0,c		; get I value
	inc.x	c			; increment 3 low nibbles
	retcs				; if carry, we might overwrite #1000
	move.a	c,r0		; I changes permanently
	jump.3	savelo
doret1:
	retclrc
```

Now, the astute amonst you may have noticed something. That would be that saucy little comment: "I changes permanently". This subroutine definately updates I as it iterates over the write - which is very much at odds with expectations. This routine starts with d0 pointing at the initial register (v0) and d1 pointing at what I was pointing at but in calculator memory. The loop reads a byte from location d0, and saves it into d1, reads the base pointer out of d0 into C then saves it back into d0 (because we swapped it with the data we wanted to save). Then it checks if C >= A; you'll note that the last instruction of copysetup puts a copy of our end pointer into A - we do this because we can't check d0 vs d1 directly. Then, if this check fails, it knows it hasn't written the last byte of data yet, and it adds 2 (nibbles) to the pointers, moves the r0 value (which is I) to C, increases it with the X field (3 low nibbles) so it can crash the program out if this overflows 0xFFF, then stuffs the result back into r0.

The quirks test program we use, it turns out, has a fatal flaw. If you're any good at loop analysis you'll have noticed that, unlike the behavior on the VIP, while this code does change I, it will not increase it one the final time when it completes, as the loop will exit from the brge - skipping the add 2 and saving to I.

And so, a test program was written: http://johnearnest.github.io/Octo/index.html?gist=9a47e585a69283531c2eaeb60e16e914

By saving/loading 2 registers, it becomes possible to detect if the I register moves on 0 (reports 2), 1 (reports 3) or 2 (reports 4). Octo's existing quirk options report the two following results:
Octo's standard behavior, and I save/load quirks enabled behavior:  
![Octo Base](quirk_i_img/octo_noquirk.png) ![Octo Quirk](quirk_i_img/octo_quirk.png)
C48 & SC10:  
![SC10](quirk_i_img/sc10.jpg)  
SCHIP 1.1:  
![SCHIP](quirk_i_img/schip.jpg)  

So, although I thought maybe I was going to expose a big surprise with this quirk, as we can see, it turns out: Superchip 1.1 does in fact demonstrate the behavior inkeeping with the existing expectation, even if c48 and sc10 do not.

So, what is happening in schip?

```
0041C  A=R2 		# Setup Routine for Load/Save coommands. Loads physical address into A
0041F  C=R0 		# Loads I into C
00422  C=C+C   A 	# Bytes -> Nibbles
00424  C=C+A   A 	# C contains physical address where I points
00426  D1=C 		# Set D1 to physical address where I points
00429  A=R4 		# Set A to memory Start area? Turns out, v0-f are at the beginning of memory area
0042C  AD0EX 		# Swap A with D0?
0042F  RTN
...
00C51  LC      55 				# Stores!
00C55  ?C#A    B 				# Branch if not equal to save command
00C58  GOYES   00C86
00C5A  GOSUBL  0041C 			# Definately Save/Load FINDME
00C60  C=R0 					# Set C to I
00C63  D=C     A 				# Copy C to D
00C65  C=DAT0  B 				# Move byte from dat0 to C - dat0
00C68  DAT1=C  B 				# Store byte from C to dat1
00C6B  CD0EX 					# Swap?
00C6E  D0=C 					# Move?
00C71  ?C>=A   A 				# Check if C >= A, end function
00C74  GOYES   00C84 			# RTNCC
00C76  D1=D1+  2 				# Step pointer
00C79  D0=D0+  2 				# Step pointer
00C7C  D=D+1   X 				# Increase D rather than C (nb use of x field, first 3 nibbles)
00C7F  GONC    00C65 			# Go if no carry to loop, otherwise return
00C82  RTNSC
00C84  RTNCC
```

We can see that this has been modified, with the inclusion of the D register (not to be confused with D0/D1). It has a tighter loop, as using one of the other registers saves on a few operations. You can see though that it makes no attempt to save anything back to r0, so this will not update the I register. The information needed to do so is basically there however, just stored in D and off by 1. Additionally, Copysetup has been significantly neatend up and made faster - d0 when we enter into this function is pointing at the memory location of the last register we want to copy, so, by not modifying d0, when we swap the address of v0 (r4) into it via A, we actually end up with the physical address of the end register stored back in A, which is exactly what we want to use to terminate our loop.

I think then, it's hard to call the origin of this quirk. It seems like a mistake was made in c48 with the loop design, and that survived unchanged into schip. Then, it seems like it went out of the window entirely when schip received it's speedier rewrite.

# Fixing it

OK so this is a the first real test, I guess. If we move our D=D+1 before the GOYES, we should have the correct value stored in it, which seems like the best course of action? If we pointed the GOYES exit to the same point as the Load routine, we could run the same code for both, but, this would only give us 2 nibbles, and all the R storage registers need 3 nibbles, and only worth with A and C. There's really not enough room. However, the code following these routines are 2 not heavily utilised functions that are part of the sprite calls. We're quite near the end of the program, so, it's possible that I might be able to relocate these in entirity out to the end of the code base and use the space left behind for a variety of deeds;

```
00943  GOSUB   00D1B 					# Calls S_Regs  
```

An address of 943 gives me 00947 + 0x7FF yields 0x1146, which is a good ways past the 0x10EA the program currently ends at. Likewise an address of 009A7 yields 0x11AA. These routines are from 00D1B to 00D4E, for s_rregs, and 00D50 to 00D83 for r_rregs, so that's going to put me at 0x111D for the end of the first routine, which is still in 3 nibble range for the calls to the 2nd function. Seems like it should be a reasonable solution to get a big well of space to work with.

Then, I figure when I have the space I'll just copy d to c and save it into r0.

