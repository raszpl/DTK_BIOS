# DTK_BIOS
DTK BIOS reverse engineering.

https://en.wikipedia.org/wiki/DTK_Computer#Foundation_and_expansion_(1981%E2%80%931989) 
"Unusual for a company of its stature, Datatech also developed its own BIOS for its IBM PC compatibles. Its first PC BIOS clone was developed in 1985; while second source of such BIOSes had already been developed by companies such as Phoenix Technologies in the United States, Datatech feared that they would be sued out of existence by IBM and so developed its own clean-room implementations in 1985. Although Datatech's fears were later assuaged, quality-assurance supervisor David Wang felt that the continued development of in-house BIOSes afforded the company technical expertise that could be applied to other aspects of their R&D lab, as was the case for the company's ASIC division."

In this repository I try to investigate and reverse engineer its idiosyncrasies.

# BIOS
Theretroweb has 34 examples of bespoke DTK bioses https://theretroweb.com/bios?page=1&itemsPerPage=40&biosManufacturerId=141.
I start with Symphony Labs SL82C460 based 386 [PEM-0036Y](https://theretroweb.com/motherboards/s/dtk-pem-0036y) board [DTK 386 ROM BIOS Version 4.27 DDY103-01](https://www.vogons.org/viewtopic.php?p=1254644#p1254644)

Quirks:
- Undesirable defensive programming. Prints guarded against CGA Snow by default (waiting for H_sync) without first detecting if we are even using CGA card to begin with.
- Lots of monkey patching and stupid code. For example [sub_F53BE](https://github.com/raszpl/DTK_BIOS/blob/f81fe9854597db5af95cd36128b1cbe49b79fe43/PEM-0036Y_DTK.lst#L11537) jumping to retn of sub_F5376 instead of using its own one two bytes below at seg000:53DF.

# status
dtk-pem-0036y ~50% dissasembled, ~20% labeled.
