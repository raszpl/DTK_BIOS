# DTK_BIOS
DTK BIOS reverse engineering.

[wikipedia](https://en.wikipedia.org/wiki/DTK_Computer#Foundation_and_expansion_(1981%E2%80%931989)) : 
"Unusual for a company of its stature, Datatech also developed its own BIOS for its IBM PC compatibles. Its first PC BIOS clone was developed in 1985; while second source of such BIOSes had already been developed by companies such as Phoenix Technologies in the United States, Datatech feared that they would be sued out of existence by IBM and so developed its own clean-room implementations in 1985. Although Datatech's fears were later assuaged, quality-assurance supervisor David Wang felt that the continued development of in-house BIOSes afforded the company technical expertise that could be applied to other aspects of their R&D lab, as was the case for the company's ASIC division."

In this repository I try to investigate and reverse engineer its idiosyncrasies.

# BIOS
Theretroweb has 34 examples of bespoke DTK bioses https://theretroweb.com/bios?page=1&itemsPerPage=40&biosManufacturerId=141.
I start with Symphony Labs SL82C460 based 386 [PEM-0036Y](https://theretroweb.com/motherboards/s/dtk-pem-0036y) board [DTK 386 ROM BIOS Version 4.27 DDY103-01](https://www.vogons.org/viewtopic.php?p=1254644#p1254644)

Quirks:
- Monkey patching and stupid code. For example [sub_F53BE](https://github.com/raszpl/DTK_BIOS/blob/f81fe9854597db5af95cd36128b1cbe49b79fe43/PEM-0036Y_DTK.lst#L11537) jumping to retn of sub_F5376 instead of using its own one two bytes below at seg000:53DF.
- Clever but useless optimizations.
- Uses Ram Refresh (FSB dependent) for timing.
- Multiple developers using different coding styles, sometimes even in same function. For example loading BDA segment from eprom vs from immediate. Loading AX vs loading AH and AL separately when calling Interrupt Services.
- Setup: Clunky slow window growing and shrinking animations. Especially shrinking animation is infuriating because it takes ~1 second irregardless of CPU speed.
- Setup: Esc dosnt exit sub menus, have to explicitly press enter on last item named Exit.
- Setup: No option to exit without saving.
- Setup: Undesirable defensive programming. Prints [sub_F57D6](https://github.com/raszpl/DTK_BIOS/blob/f81fe9854597db5af95cd36128b1cbe49b79fe43/PEM-0036Y_DTK.lst#L12185) guarded against CGA Snow by default (waiting for H_Sync) without first detecting if we are even using CGA card to begin with. As a side effect rate limits print speed to one character per H retrace.
- Setup/System Configuration: Menu navigation produces CGA Snow like glitches when moving cursor on _every graphic adapter_ :o Its an actual bug in cursor routine sprinkling random Underscores.
- Setup/System Configuration/Time&Date: Navigating horizontally requies pressing Up/Down while Left/Right changes settings. Can also press Enter to open special sub window for Time/Date that requires entering Hyphens and Colons! Why make user enter punctuation?!?
- Setup/System Configuration/Disk: Doesnt validate Custom IDE types (48, 49), allows setting with all 0 values. All 0 setting freezes bootup/Setup Preformat for 20-30 seconds.
- Setup/System Configuration/Disk: Doesnt support Auto configuration thru [IDE IDENTIFY DRIVE Command (0xEC)](https://www.os2museum.com/wp/identify-ancient-drive/).
- Setup/System Configuration/Primary Display Card: Pressing Enter/Left/Right opens sub Window. Only CGA card displays _two options_ and allows picking something using exclusively Number keys, cursor keys do nothing. Any other Graphic Adapter Type just displays useless 'VGA/EGA/MGA ARE AUTO-SET' message in the sub Window. 
- Setup/System Configuration/Coprocessor: Useless Menu item looking like user could actually set something here. Enter/Left/Right opens sub Window with "COPROCESSOR IS AUTO-SET" text only.
- Setup/Preformat: Engrish "type haven't set up" instead of "type not set up".

# status
dtk-pem-0036y ~50% dissasembled, ~20% labeled.
