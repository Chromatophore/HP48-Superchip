# HP48-Superchip
This is a repo that includes a selection of original HP-48 binaries relating to Chip-8 - specifically the Chip-48 and Super-Chip platforms. They were obtained to allow Chip-8 games to be run on an original 1990s HP48S calculator for OctoJam III, a game jam where people are asked to create new Chip-8 games with Octo. Here are some explanations of some terms:

#### [Chip 8](https://en.wikipedia.org/wiki/CHIP-8):
An interpreted Assembly Language designed in the 1970s to allow for people to share games across a handful of home computer platforms.
#### HP 48:
A popular graphing calculator launched in the early 90s by HP that supported the creation of programs. Similar to the popular Ti81, but did not support Basic. As a result, (I believe) Andreas Gustafsson thought to write a Chip 8 interpreter for the HP48 so people could more easily write programs and games, and play those that had been already written before.
#### Chip-48:
The name of the Chip-8 interpreter for the HP48 that was written by Andreas Gustafsson. It implements only the original instruction specification without modifications.
#### Super-Chip or schip:
An expanded instruction set of the Chip 8 interpreted language with additions such as a higher resolution and scrolling. It was first implemented on the HP48, and was based on the Chip-48 interpreter.
#### [Octo](https://github.com/JohnEarnest/Octo):
A modern Javascript based Chip-8 interpreter which supports the Super-Chip expansions as well as makes some of its own. It aims to be accurate in the way it interprets the Chip-8 code such that programs designed with it would work when deployed to the original period correct hardware and interpreters.

# OK, back to what this repo is about:

A side intent to simply running Chip-8 programs was also to investigate the actual functionality of these original platforms, as many later interpreters implement what it is stated that Super-Chip does based on its specification, which was widely available, rather than what the binary would have actually done, the specifics of which would be limited to those with an HP48 with cable. As a result, many 'quirks' - behavior in the interpreter that diverges from the original Chip-8 interpreters written in the 1970s for home computers of the era - can be observed. Previously these quirks were identified based on programs written in the era, however these were limited to the examples that could be discovered and the ideas people had at the time - as a result, having a working device on hand and a team of inquisitive minds has yielded the discovery of several more quirks that were previously unknown.

Many of these quirks were quite unexpected and their individual origins were a question that piqued (in particular) my interest - were they deliberate changes? Mistakes? I decided to investigate each of them to try and ascertain & subsequently document their origin, and, if possible, create a new schip binary for the HP48 calculator that conforms closer to the behavior observed on the Cosmac VIP's interpreter, and of Octo, so that it is easier to make games that can run on the platform without having to work around some of the more awkward behaviors.

One of the most helpful information sources in this process has been an ["Introduction to Saturn Assembly Language"](http://members.ziggo.nl/kees.van.der.sanden/downloads/Saturn_tutorial.pdf) - it includes a wealth of information about the instructions and design of the Saturn processor - the processor that is in the HP48, as well as the op codes associated with them, which is what has allowed me to reverse engineer and modify the binaries.

# Behavior and Quirk Investigations
These will, once I have written them, link to investigations of each quirk of which I am aware.
#### [8XY6 & 8XYE](investigations/quirk_shift.md)
Bit shifts X register by 1, VIP: shifts Y by one and places in X, HP48-SC: ignores Y field, shifts X
#### FX55 & FX65
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