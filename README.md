# DTK_BIOS
DTK BIOS reverse engineering.

[wikipedia](https://en.wikipedia.org/wiki/DTK_Computer#Foundation_and_expansion_(1981%E2%80%931989)) : 
"Unusual for a company of its stature, Datatech also developed its own BIOS for its IBM PC compatibles. Its first PC BIOS clone was developed in 1985; while second source of such BIOSes had already been developed by companies such as Phoenix Technologies in the United States, Datatech feared that they would be sued out of existence by IBM and so developed its own clean-room implementations in 1985. Although Datatech's fears were later assuaged, quality-assurance supervisor David Wang felt that the continued development of in-house BIOSes afforded the company technical expertise that could be applied to other aspects of their R&D lab, as was the case for the company's ASIC division."

In this repository I try to investigate and reverse engineer its idiosyncrasies.

# BIOS
Theretroweb has 34 examples of bespoke DTK bioses https://theretroweb.com/bios?page=1&itemsPerPage=40&biosManufacturerId=141.
I start with Symphony Labs SL82C460 based 386 [PEM-0036Y](https://theretroweb.com/motherboards/s/dtk-pem-0036y) board [DTK 386 ROM BIOS Version 4.27 DDY103-01](https://www.vogons.org/viewtopic.php?p=1254644#p1254644):

```
                          ROM SETUP PROGRAM VERSION 3.0
              (C) COPYRIGHT DATATECH ENTERPRISES CO., LTD 1990-1992.
                               ALL RIGHTS RESERVED.
    ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄
    ▌┌─────────────────────────────────────────────────────────────────────┐▐
    ▌│                                                                     │▐
    ▌│                     SET UP SYSTEM CONFIGURATION                     │▐
    ▌│                                                                     │▐
    ▌│    CURRENT DATE : [06-29-2026]         ┌──── HARD DISK INFO ────┐   │▐
    ▌│    CURRENT TIME : [ 09:24:11 ]         │            --C-- --D-- │   │▐
    ▌│    COPROCESSOR  : [   None   ]         │ TYPE     :     1    49 │   │▐
    ▌│    BASE      MEMORY : [  640 KB]       │ CYL      :   306     0 │   │▐
    ▌│    EXTENDED  MEMORY : [ 3072 KB]       │ HEAD     :     4     0 │   │▐
    ▌│    DISKETTE DRIVE A : [  NO   ]        │ SEC/TRK  :    17     0 │   │▐
    ▌│    DISKETTE DRIVE B : [  NO   ]        │ PRE-COMP :   128     0 │   │▐
    ▌│    FIXED DISK TYPE C : [ NO ]          │ LAND-ZONE:   305     0 │   │▐
    ▌│    FIXED DISK TYPE D : [ 47 ]          │ CAP.(MB) :    10     0 │   │▐
    ▌│    PRIMARY DISPLAY CARD : [ CGA/80 ]   └────────────────────────┘   │▐
    ▌│    BOOT SEQUENCE : [ A THEN C ]                                     │▐
    ▌│    INITIAL NUM LOCK : [ OFF ]                                       │▐
    ▌│    EXIT                                                             │▐
    ▌│    --------------------------------------------------------------   │▐
    ▌│                    ↑↓ :CHANGE ITEM   ◄─┘ :ACCEPT                    │▐
    ▌└─────────────────────────────────────────────────────────────────────┘▐
    ▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀▀
```

Quirks:
- Monkey patching and stupid code. For example [sub_F53BE](https://github.com/raszpl/DTK_BIOS/blob/f81fe9854597db5af95cd36128b1cbe49b79fe43/PEM-0036Y_DTK.lst#L11537) jumping to retn of sub_F5376 instead of using its own one two bytes below at seg000:53DF.
- Typos - randomly using 1 or 2 for Booleans.
- Some clever looking but useless attempts at optimization like reusing already loaded CMOS_0Bh_REGB_ALARM_INT_EN (20h) as IO_Port_PIC_Cmd_NON_SPECIFIC_EOI (also 20h) saving 0 bytes because 'mov al, bl' (88 D8) = 'mov al, 20h' (B0 20).
- Uses Ram Refresh (FSB dependent) for timing.
- Authored by multiple developers using different coding styles, sometimes even in same function! For example loading BDA segment from eprom vs immediate. Loading AX vs loading AH and AL separately when calling Interrupt Services.
- Setup: Every single Option is its own Function! Whole Setup looks constructed out of MACROS.
- Setup: Clunky slow window growing and shrinking animations. Especially shrinking animation is infuriating because it takes ~1 second irregardless of CPU speed.
- Setup: Little code reuse. Every window growing and shrinking animation size has its own dedicated Function instead of one animation routine and a table of sizes.
- Setup: Esc dosnt exit sub menus, have to explicitly press enter on last item named Exit.
- Setup: No option to exit without saving. Options are saved in CMOS the moment User changes them.
- Setup: Clumsy defensive programming. Prints [sub_F57D6](https://github.com/raszpl/DTK_BIOS/blob/f81fe9854597db5af95cd36128b1cbe49b79fe43/PEM-0036Y_DTK.lst#L12185) guarded against CGA Snow by default (waiting for H_Sync) without first detecting if we are even using CGA card to begin with. As a side effect rate limits print speed to one character per H retrace. But some internal SETUP print functions are sufficiently protected from CGA Snow, some parts still use standard int10h.
  - Counting up Ram during Boot produces CGA Snow, probably because standard int10h functions singular retrace wait is quite a way from store code.
  - Setup/System Configuration: Menu navigation still produces CGA Snow when moving cursor because it uses int10h_9_Write_character_and_attribute_at_cursor.
  - Setup/System Configuration: Example of stupid code in
    <details><summary>Cursor highlighting at seg000:C01B set_Attribute_in_vram</summary>
    <pre>
      mov     ax, di
      mov     si, ax
      mov     ah, bl
    color_change_loop:
      call    wait_H_Retrace_IRQ0_timer_int_Off
      mov     al, es:[si]
      mov     es:[di], ax
      add     di, 2
      add     si, 2
      loop    color_change_loop
    </pre>
      Why all this useless reading? it should look like this:
    <pre>
      inc     di              ; Advance di by 1 to point directly to the first Attribute byte
    color_change_loop:
      call    wait_H_Retrace_IRQ0_timer_int_Off
      mov     es:[di], bl   ; Overwrite just the Attribute byte
      add     di, 2
      loop    color_change_loop
    </pre>
    </details>  
- Setup/System Configuration/Time&Date: Navigating horizontally requies pressing Up/Down while Left/Right changes settings. Can also press Enter to open special sub window for Time/Date that requires entering Hyphens and Colons! Why make user enter punctuation?!?
- Setup/System Configuration/Disk: Doesnt validate Custom IDE types (48, 49), allows setting with all 0 values. All 0 setting freezes bootup/Setup Preformat for 20-30 seconds.
- Setup/System Configuration/Disk: Doesnt support Auto configuration thru [IDE IDENTIFY DRIVE Command (0xEC)](https://www.os2museum.com/wp/identify-ancient-drive/).
- Setup/System Configuration/Primary Display Card: Pressing Enter/Left/Right opens sub Window. Only CGA card displays _two options_ and allows picking something using exclusively Number keys, cursor keys do nothing. Any other Graphic Adapter Type just displays useless 'VGA/EGA/MGA ARE AUTO-SET' message in the sub Window. 
- Setup/System Configuration/Coprocessor: Useless Menu item looking like user could actually set something here. Enter/Left/Right opens sub Window with "COPROCESSOR IS AUTO-SET" text only.
- Setup/Preformat: Engrish "type haven't set up" instead of "type not set up".

# status
dtk-pem-0036y ~50% dissasembled, ~20% labeled.
