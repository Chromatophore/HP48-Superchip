# 16x16 Sprites & 0 Line Sprites

Superchip claims to add 16x16 sprites by using instruction DXYN with n = 0 (n specified # of lines). These only function as expected in higher resolution display mode. In low resolution, an 8x16 sprite (a normal sprite with 16 rows) is drawn.

## Initial notes:

One of the new features of superchip is the addition of a large, 16x16 sprite. It's selectable by using an aspect of the Chip8 Sprite command that serves no real purpose. The instruction, DXYN, takes an X and Y coordinate, and then a number of lines that can fit in a single nibble. Even though it is a valid option, there is really no reason to draw a sprite with 0 lines. Schip hijacks this and offers special handling for it, drawing a 16x16 sprite.

However, here is note about it in the release documentation:
```
In extended screen mode the resolution is 64 by 128 pixels. A
sprite of 16x16 size is available.
```

The information that I suspect this tries to relate, but was most likely missed/lost, is that: the 16x16 sprite handler is only available in the extended screen mode. It was right here in the release document, like so many other little things, but its significance was missed.

## Investigation:

As we'll see in some other quirks, there are separate sprite drawing handlers for when the system is in high resolution and low resolution modes. I give an explanation [here](https://github.com/Chromatophore/HP48-Superchip/blob/master/investigations/quirk_collide.md#investigation-colliding-row-count-in-vf) in another quirk, but the bottom line is that this separation makes sense when you realise that both modes share the same, high resolution display buffer.

Ultimately, the large sprite detection and rendering only exists in the sprite command that's called when in extended resolution mode.

However, the behavior in standard/low resolution mode is still not as expected. This is readily observable because we have several examples of Octojam games that attempt to use this large sprites in the low res mode.

The Cosmac's Chip8 interpreter's drawing instruction has it's structure laid out on [this awesome website](http://laurencescotford.co.uk/?p=304) by Laurence Scotford. In it you can see that at 008E->0091, it assesses the number of remaining rows before drawing anything, so the classic behavior is to draw 0 sprite rows

The low res routine starts at 00E81, it's called with the number of rows to draw, 0 to 15, stored in the C register into nibble 15, which is also accessible as the S register. Our test is at at the end of the routine, where we run this code:

```
00F8B  C=C-1   S 			# Decrease value in S field
00F8E  ?C=0    S 			# Test if it is 0
00F91  GOYES   00F97 		# Skip looping if it is 0
00F93  GOTO    00ECD 		# Go back up and draw another line
00F97  RTN  				# Return 
```

This provides our iterations to draw out all our bytes. Here, clearly, there is difference in control flow, so the way our loop unrolls means that we will wrap around from a 0 line input value to 15 before hitting the conditional, which then continues from there, giving us an 8x16 high sprite.

## Fixing it?

In this circumstance, the effect of using a 0 height sprites is sort of up for grabs - they do basically nothing on the cosmac vip's interpreter. Any functionality that can be injected into either of these instructions is pretty much worthwhile. Both behaviors are useful, but the difference is awkward. The processes of converting the low resolution behavior to draw a double wide sprite would be reasonably inconvenient - it changes both the initial function, and the loop in the function it calls, and spacing things out doesn't seem like a lot of fun to me right now.

for the time being, I'm going to be leaving this as is.
