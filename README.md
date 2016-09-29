# HP48-Superchip
This is a repo that includes a selection of original HP-48 calculator binaries relating to Chip-8 interpreters - specifically the Chip-48 and Super-Chip platforms. They were obtained to allow Chip-8 games to be run on an original 1990s HP48S calculator for OctoJam III, a game jam where people are asked to create new Chip-8 games with Octo, a modern and very accessible interpreter.

Via analysis of the specifications for these interpreters, as well as contemporary programs which ran with them, it has been previously observed that there are some variations in how certain instructions are implemented - these variations will be referred to as "Quirks", or similar. After having actually obtained an original calculator and investigating the binaries further, several more previously unknown quirks have been observed. The goal of this repo is to investigate the origin of these quirks, and offer some explanation as to what might have caused them.

I would like to clarify that, these interpreters were written by hobbyists for fun, and shared freely on the internet via newsgroups. As far as I'm aware, they were not significantly widely used or analyzed, and they likely did not have access to original implementations to verify their interpreters behaved in the same way. That these interpreters exist at all is likely responsible for Chip-8 being remembered at all, so, I'm thankful to them for that.

Here are some explanations of some terms:

#### [Chip 8](https://en.wikipedia.org/wiki/CHIP-8):
An interpreted Assembly Language designed in the 1970s to allow for people to share games across a handful of home computer platforms. It is popular for programmers to write a Chip-8 interpreter in their language of choice, to become familiar with how simple microprocessors work, as there are a good selection of programs available for it.
#### HP 48:
A popular graphing calculator launched in the early 90s by HP. It supported the creation of programs via assembly language for it's Saturn microprocessor. Similar in a lot of ways to the popular Ti81, it did however not support an accessible language for general programming, such as Basic. As a result, (I believe/am guessing) Andreas Gustafsson thought to write a Chip 8 interpreter for the HP48 so people could more easily write programs and games, and play those that had been already written before.
#### Chip-48:
The name of the Chip-8 interpreter for the HP48 that was written by Andreas Gustafsson. It implements only the original instruction specification without (deliberate?) modifications. The source code for version 2.25 of the interpreter was found in newsgroup archives and is [included as part of this Repo](c48_source.txt).
#### Super-Chip or schip:
An expanded instruction set of the Chip 8 interpreted language with additions such as a higher resolution and scrolling. It was first implemented on the HP48, and was based on the Chip-48 interpreter. I'm aware of a [version 1.0](sc10_origin.txt) and a [version 1.1](SCHIP_origin.txt) - the Superchip spec and behavior that is most publicly apparent are for version 1.1. The source code for [version 1.0](sc10_source.txt) only has been found, and offers some insight, but version 1.1 was modified for speed and involved a lot of rewriting.
#### [Octo](https://github.com/JohnEarnest/Octo):
A modern Javascript based Chip-8 interpreter which supports the Super-Chip expansions as well as makes some of its own. It aims to be accurate in the way it interprets the Chip-8 code such that programs designed with it would work when deployed to the original period correct hardware and interpreters. It offers several "Quirk" flags that allow it to behave correctly as different platforms.

# OK, back to what this repo is about:

Aside from the goal of providing a nice extra feature for Octojam III, where compatible games would be shown running on 1990s hardware, I hoped to investigate the actual functionality of these original platforms in detail. This is because many later & current interpreters implement the written Super-Chip instruction specification, which was widely available, rather than what the program as written may have actually done - the specifics of which would be limited to those with an HP48 and the interest to verify. As a result, many 'quirks' - behavior in the interpreter that diverges from the original Chip-8 interpreters written in the 1970s for home computers of the era - can be observed. Previously these quirks were identified based on programs written in the era, however these were limited to the examples that could be discovered and the ideas people had at the time - as a result, having a working device on hand and a team of inquisitive minds has yielded the discovery of several more quirks that were previously unknown.

Many of these additional quirks were quite unexpected and their individual origins were a question that piqued (in particular) my interest - were they deliberate changes? Mistakes? Typos? I decided to investigate each that I became aware of, trying and ascertain & subsequently document their origin, and, if possible, create a new schip binary for the HP48 that conforms closer to the behavior observed on the Cosmac VIP's interpreter, and thus of Octo, so that it is easier to take a program from Octo that can run on the HP48 without having to work around some of the more awkward behaviors.

# Behavior and Quirk Investigations
These will, once I have written them, link to investigations of each quirk of which I am aware.
#### [8XY6 & 8XYE](investigations/quirk_shift.md)
Bit shifts X register by 1, VIP: shifts Y by one and places in X, HP48-SC: ignores Y field, shifts X
#### [FX55 & FX65](investigations/quirk_i.md)
Saves/Loads registers up to X at I pointer - VIP: increases I, HP48-SC: I remains static
#### [BNNN](investigations/quirk_jump0.md)
Sets PC to address NNN + v0 - VIP: correctly jumps based on v0 HP48-SC: reads highest nibble of address to select register to apply to address (high nibble pulls double duty)
#### Memory Limit
Address space of Chip-8 programs is limited to 0xFFF, 4 kibibytes, but, by spec, less 0x200 bytes as 0x000 to 0x1FF are 'reserved'. VIP: 0xEA0 and above apparently also reserved for system use. HP48-SC: No such utilisation of memory, but, maximum file length results in crash.
#### 16x16 Sprites
Superchip claims to add 16x16 sprites by using instruction DXYN with n = 0 (n specified # of lines). These only function in higher resolution display mode.
#### Swapping Display Modes
Superchip has two different display modes, 64x32 and 128x64. When swapped between, the display buffer is not cleared. Pixels are modified based on being XORed in 1x2 vertical columns, so odd patterns can be created.
#### Collision Enumeration
An interesting and apparently often unnoticed change to the Super Chip spec is the following:
All drawing is done in XOR mode. If this causes one or more pixels to be erased, VF is <> 00, other-wise 00.
In extended screen mode (aka hires), SCHIP 1.1 will report the number of rows that include a pixel that XORs with the existing data, so the 'correct' way to detect collisions is Vf <> 0 rather than Vf == 1.
#### Collision with the bottom of the screen
Sprites that are drawn such that they contain data that runs off of the bottom of the screen will set Vf based on the number of lines that run off of the screen, as if they are colliding.
#### Large Font
Superchip includes a larger font. This font does not include hex characters A through F - the spec actually states this but I'm not sure anyone realised.
#### Platform Speed
The HP48 calculator is much faster than the Cosmac VIP, but, there is still no solid understanding of how much faster it is for most instructions for the purposes of designing compelling programs with Octo. A [modified version of cmark77](https://johnearnest.github.io/Octo/index.html?gist=0b340c02d2c41c164fd6849a377dd235), a chip 8 graphical benchmark tool written by taqueso on the Something Awful forums was used and yielded scores of 0.80 kOPs in standard/lores and 1.3 kOps in extended/hires. However graphical ops are significantly more costly than other ops on period hardware versus Octo (where they are basically free) and as a result a raw computational cycles/second speed assessment still has not been completed.

# Thanks

None of this would have been possible without [hpcalc.org](http://www.hpcalc.org/), who have archived large amounts of information and software for a selection of HP calculators. One of the most helpful single information sources in this process has been an ["Introduction to Saturn Assembly Language"](http://members.ziggo.nl/kees.van.der.sanden/downloads/Saturn_tutorial.pdf) by Gilbert Fernandes and Eric Rechlin - it includes a wealth of information about the instructions and design of the Saturn processor - the processor that is in the HP48 - as well as the op codes associated with them, which is what has allowed me to reverse engineer and modify the binaries. I have noted some mistakes in it but otherwise it has been absolutely invaluable. I have tried many programs and failed to get them to be functionally useful to me in short order - the one program I rely heavily on is [Decode 1.3](http://www.hpcalc.org/details/3780), a DOS based disassembler, which runs quite happily in DOSBOX, though I have discovered problems with this, too.

I'd also like to thank [John Earnest](https://github.com/JohnEarnest) for making Octo and inspiring me to take a look at this, and also the rest of the [AwfulJams team](http://www.awfuljams.com/) for their tireless? dedication to this cause.

Finally, I'd like to thank the person whom owned my HP48 calculator previously, and kept it safely and in working condition in Portugal for what I can only assume has been around 25 years.