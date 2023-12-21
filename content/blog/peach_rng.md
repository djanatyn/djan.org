+++
title = "understanding melee peach item pull rng logic"
date = 2023-12-20
draft = false

[taxonomies]
categories = ["lab notebook"]
+++

# peach pulls mystery items

as a melee peach player, i spend a lot of my time pulling random items out of the ground.

{{ figure(src="/img/peach-turnip-pull.gif", alt="gif of peach pulling a turnip", caption="image from ssbwiki") }}

princess peach has a special move called ["Vegetable"](https://www.ssbwiki.com/Vegetable), activated by pressing down-special while grounded. if peach is able to complete her pull animation successfully, she retrieves an item from the ground - either a turnip, a bobomb, a beamsword, or a mr saturn (with turnips being the most common). it's a defining move for her character, and for many players is the default option to consider when in a safe position. although she loses access to a few options while holding a turnip (most notably grab), the possibility of throwing it at an opponent demands respect. it can open up new approaches, enable otherwise impossible combo routes, and empowers her to interact with her opponent from a distance.

over the 20+ year history of the game, the community has created resources to help players understand how often to expect particular outcomes. this image is what i see referenced most often:

{{ figure(src="/img/expected-frequency-chart.png", alt="table of peach item pull percentages", caption="credit: Magus420") }}

as shown above, not all pulls are equally likely. there are different kinds of turnips, and some turnips do more damage than others. some sequences of pulls are highly valuable, but extremely unlikely to occur - pulling two stitchfaces in a row (especially [when you know how to use them](https://www.youtube.com/watch?v=rrzFeWC5kmc)) is considered a special occassion. there are even advanced techniques like [knitting](https://smashboards.com/threads/postmodern-rng-tactics-knitting-panning-theory-discussion.414626/) that allow peach to continuously pull items faster than she could normally.

understanding how turnip rng works won't give you any competitive advantage in the game. but it is fun! so, let's continue.

# counting turnips

when playing melee online these days, it's common to record game records using the [SLP replay format](https://github.com/project-slippi/slippi-wiki/blob/master/SPEC.md). these replays can be processed programmatically to collect statistics on the occurrence of specific events ingame. 

previously, i had been very curious about whether [my actual replay files matched up to the expected values ](https://github.com/djanatyn/turnip-counter) predicted by the chart above. at the time, [hohav](https://github.com/hohav) had already put together a rust crate called [peppi](https://github.com/hohav/peppi) which could process these replays. it didn't take very long to put together a program which would iterate over a directory of replay files, identify any peach item pulls that matched my character, and record them to a simple sqlite database:
```sql
CREATE TABLE IF NOT EXISTS games (
    id INTEGER PRIMARY KEY NOT NULL,
    filename TEXT NOT NULL,
    start_time INTEGER,
    p1_name TEXT NOT NULL,
    p1_code TEXT NOT NULL,
    p2_name TEXT NOT NULL,
    p2_code TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS items (
    id INTEGER PRIMARY KEY NOT NULL,
    game_id INTEGER NOT NULL,
    item_id INTEGER NOT NULL,
    frame INTEGER NOT NULL,
    kind TEXT NOT NULL,
    FOREIGN KEY (game_id) REFERENCES games (id)
);
```

even with a small sample of 3,505 pulls and 418 games, my results were very close to expectations:
```sql
sqlite> SELECT COUNT(*) FROM items;
3505
sqlite> SELECT COUNT(*) FROM games;
418
sqlite> SELECT kind, CAST(COUNT(*) AS REAL) / (SELECT COUNT(*) FROM items) FROM items GROUP BY kind;
Beamsword      0.00114122681883024 -- ~0.11% actual vs 0.13% predicted
Bobomb         0.00228245363766049 -- ~0.23% actual vs 0.26% predicted
DotEyesTurnip  0.0145506419400856  -- ~1.46% actual vs 1.71% predicted
MrSaturn       0.00485021398002853 -- ~0.46% actual vs 0.39% predicted
NormalTurnip   0.888445078459344   -- ~88.84% actual vs 88.95% predicted (59.873 + 10.264 + 8.553 + 5.132 + 5.132)
StitchTurnip   0.0182596291012839  -- ~1.82% actual vs 1.71% predicted
WinkyTurnip    0.0704707560627675  -- ~7.04% actual vs 6.84% predicted
```

but i was still curious. i had a chart of frequencies to expect, and i saw that my actual games were close to these expectations, but i still didn't understand how the game logic determined which item comes out of the ground.

# a reddit post on melee rng

at some point, i encountered [a reddit post from september 2017 by twotwelvedegrees](https://www.reddit.com/r/SSBM/comments/71gn1d/the_basics_of_rng_in_melee/), titled "The Basics of RNG in Melee". not only did this post explain the basics of melee's [linear congruential generator](https://en.wikipedia.org/wiki/Linear_congruential_generator) implementation, it also broke down the particular RNG calls which were executed for peach's item pulls:

> All random events in the game are controlled by the output of two functions: get_random_int(max_val) and get_random_float()...On any turnip pull Peach starts with a call to get_random_int(128) which proceeds as follows: 

| Value | Result |
| --- | --- |
| 0 | Peach pulls an item |
| 1-127 | Peach pulls a turnip |

> On pulling an item, there is then a call to get_random_int(6) which proceeds as follows:

| Value | Result |
| --- | --- |
| 0-1 | Peach pulls a bomb |
| 2-4 | Peach pulls a Mr. Saturn |
| 5 | Peach pulls a beam sword |

> On pulling a turnip, there is then a call to get_random_int(58) which proceeds as follows:

| Value | Result |
| --- | --- |
| 0-34 | Peach pulls a regular turnip |
| 35-40 | Peach pulls an unamused turnip |
| 41-45 | Peach pulls a line eyes turnip |
| 46-48 | Peach pulls a circle eyes turnip |
| 49-51 | Peach pulls a super happy turnip |
| 52-55 | Peach pulls a winky turnip |
| 56 | Peach pulls a dot eyes |
| 57 | Peach pulls a stitch |

this was eye-opening for me. if i could analyze the code that was running while peach pulled an item, i could see (and provide a citation for) the conditional branches that were executed. i would no longer need to trust the chart - i could derive this information on my own. i was excited! and thankfully, much of the hard work had already been done for me.

# the ssbm datasheet

in the very first paragraph, twotwelvedegrees mentions a spreadsheet:

> Major props to u/dansalvato and this spreadsheet since it basically did the research for me.

this is the [SSBM Data Sheet (1.02)](https://docs.google.com/spreadsheets/d/1JX2w-r2fuvWuNgGb6D3Cs4wHQKLFegZe2jhbBuIhCG8/preview#gid=19). it contains function addresses for many of the subroutines the game engine executes during normal gameplay. this document represents an extraordinary amount of effort reverse-engineering the game, made available for any curious hackers like myself.

one of the sections under the "Function Addresses" tab is labeled "Random Number Generator". the `get_random_int` function mentioned in the reddit post is described in some detail here:

- there is a main RNG function at `0x80380580`,
- it's input is `r3`, which is the number of possible numbers to generate for the call,
- it's output is `r3`, which is the random number returned

if this is accurate, it means that every time peach attempts to pulls an item, we expect there's a call to `get_random_int(128)` at `0x80380580`. we should be able to load up our copy of melee, start the game, identify the function, and set some breakpoints to see it in action!

# enabling dolphin's debugger

i'm on nixos, so to load dolphin, i pull down a version from nixpkgs:
```sh
nix build nixpkgs#dolphin-emu -o dolphin-hacking ; : get dolphin package
./dolphin-hacking/bin/dolphin-emu                ; : execute dolphin
```

you're on your own for acquiring a copy of melee's disk image.

on my copy of dolphin, the debugging interface wasn't visible by default, so i had to enable it:

{{ figure(src="/img/enable-debugging-ui.png", alt="screenshot of 'Enable Debugging UI' checkbox", caption="Options > Configuration > Interface") }}

i also toggled the 'Code', 'Registers', and 'Memory' pane into view:

{{ figure(src="/img/dolphin-view-settings.png", alt="enabling 'Code', 'Registers', and 'Memory' view", caption="View Toggles") }}

i also moved some panes around:

{{ figure(src="/img/dolphin-ui.png", alt="screenshot of dolphin debugger ui", caption="ready for action") }}

i've got an adapter for my gamecube controller setup, and i have a few gecko codes enabled which change the default behavior of the game to make investigation a little easier:

- Unlock All Characters and Stages
- VS 1 player (set time to none)

the game loads as expected, and we're at the character select screen:

{{ figure(src="/img/dolphin-debugger-game-running.png", alt="screenshot of melee running with debug pane to the side", caption="debugging activated") }}

you might notice that all instructions are marked as `<unknown>` while the game is running. if we hit the pause button, we can see the instructions:

{{ figure(src="/img/dolphin-unpaused.png", alt="screenshot of instructions in debug pane", caption="instructions are now visible") }}

cool! we can see the address of each instruction, it's parameters, and there's even a callstack showing how we got here.

we already know where `get_random_int` is, so let's try searching for it.

# finding get_random_int

there's a "Search Address" text input available. typing in `0x80380580` and hitting enter will highlight the line for that address. we can right-click this line, select "Add Function" from the context menu, and we'll see a set of instructions highlighted:

{{ figure(src="/img/add_function.png", alt="instructions for get_random_int", caption="get_random_int assembly instructions") }}

the name `zz_80380580_` is not very helpful. we can change that by right-clicking the same line, selecting "Rename symbol" from the context menu, and giving this symbol a better name, `get_random_int`.

we can even select "Copy function" in the context menu to get a listing for all of the instructions:

```
get_random_int
80380580: lwz	r5, -0x570C (r13)
80380584: lis	r4, 0x0003
80380588: addi	r0, r4, 17405
8038058c: lwz	r4, 0 (r5)
80380590: mullw	r4, r4, r0
80380594: addis	r4, r4, 39
80380598: subi	r0, r4, 24893
8038059c: stw	r0, 0 (r5)
803805a0: lwz	r4, -0x570C (r13)
803805a4: lwz	r0, 0 (r4)
803805a8: rlwinm	r0, r0, 16, 16, 31 (ffff0000)
803805ac: mullw	r0, r3, r0
803805b0: srawi	r3, r0,16
803805b4: addze	r3, r3
803805b8: blr	
```

understanding what all of these instructions is outside the scope of this post. instead, let's ask dolphin to log every time this function is executed by adding a breakpoint!

# logging get_random_int calls

first, let's enable the breakpoint view (i forgot to turn it on earlier):

{{ figure(src="/img/dolphin-view-breakpoints.png", alt="enabling breakpoints", caption="oops we need those") }}

navigate to the breakpoint tab, click "New", and let's observe these calls as they execute:

{{ figure(src="/img/log-breakpoint.png", alt="breakpoint configuration", caption="breakpoint configuration") }}

- activate this breakpoint at `0x80380580` (our suspected `get_random_int` function),
- our condition `r3, 1` means to display the register `r3` in the log (which should be the range on numbers generated),
- don't stop execution of the game, just log it

it should look like this (it even includes our renamed symbol from earlier):

{{ figure(src="/img/breakpoint-success.png", alt="configured breakpoint", caption="configured breakpoint enabled") }}

in melee, attempting to place the character select token when it isn't hovering over a particular character will select a random character - the random button is a modern innovation.

{{ figure(src="/img/random-characters.gif", alt="randomly selecting characters", caption="og random button") }}

that random choice has to come from somewhere. in the log, we can actually see some `get_random_int` calls happening already:
```
Breakpoint condition returned: 1. Vars:  r3=25
```
`Vars: r3=25` sounds about right - there are 25 characters in the game!

{% alert() %}
if you don't see any logs for breakpoints, check your logging configuration ("View" > "Show Logging Configuration") and try enabling all log types.
{% end %}

# identifying where peach rng calls happen

let's get ingame and try pulling a turnip! hit down-b, hit start to pause the game, and let's see what happened.

{{ figure(src="/img/pull-turnip.png", alt="screenshot of daisy pulling turnip", caption="daisy is the best peach costume") }}

```
...
Breakpoint condition returned: 1. Vars:  r3=128
Breakpoint condition returned: 1. Vars:  r3=58
...
```

nice: 
- `get_random_int(128)` is determining whether we pull an item or a turnip, and
- `get_random_int(58)` is determining the face of the turnip

we're not logging the return value of the RNG call right now, but we've identified the `get_random_int` calls that are being executed when peach pulls an item. that's great progress!

what code is calling `get_random_int(128)`? let's modify our breakpoint with a different condition, and set it to break (not just log):

{{ figure(src="/img/modify-condition.png", alt="new breakpoint condition", caption="check for a specific argument") }}

after pulling another turnip, the game should pause, stopping at `get_random_int`. interestingly, even though peach may have entered the item pull animation, the animation does not show which item peach has pulled, since it hasn't decided yet. let's check out our callstack:

{{ figure(src="/img/link-register.png", alt="callstack screenshot", caption="the path that led us here") }}

`LR` represents the ["link register"](https://en.wikipedia.org/wiki/Link_register). whenever we call a subroutine with the `bl` instruction, we populate a special register with the address to return the program counter to once our subroutine finishes executing (usually with a `blr` instruction). if we click on the `LR = 8011d088` address, we can get some context on what's happening before this RNG call is executed, and what we do with the result.

this instruction is happening in the middle of the subroutine, so i scrolled up the previous `blr` instruction (denoting the end of the last subroutine), moved onto the next instruction (`0x8011d018`), and defined a new function as we did previously. i named it `turnip_rng_caller` - it may be used for other interactions, but we're fairly confident it's part of peach's item pull logic.

{% tip() %}
the address `0x8011d018` is included in the [smashboards community symbol map](https://smashboards.com/threads/smashboards-community-symbol-map.426763/), given the name `_$_wP_Peach_DownB_GenTurnip` - looks like we're on the right track.
{% end %}

# Peach_DownB_GenTurnip logic

let's take a closer look at the logic around `0x8011d088` in the body of the function:
```
...
8011d088: bl	->0x80380580
8011d08c: cmpwi	r3, 0
8011d090: bne-	 ->0x8011D0A0
8011d094: mr	r3, r29
8011d098: bl	->0x8011CE48
8011d09c: mr	r31, r3
8011d0a0: lwz	r4, 0x010C (r30)
8011d0a4: mr	r6, r31
8011d0a8: lfs	f1, 0x002C (r30)
8011d0ac: mr	r3, r29
8011d0b0: lwz	r4, 0x0008 (r4)
8011d0b4: lbz	r5, 0x0010 (r4)
8011d0b8: addi	r4, sp, 52
8011d0bc: bl	->0x802BD4AC
...
```

there's a few things happening here:
- we call `get_random_int(128)` (denoted by address `0x80380580`, with `r3` set to `128`),
- when we return to the subroutine, `r3` is populated with the random return value
- we check `r3` with the `cmpwi` instruction to see if the return value is 0,
- if *the return value is not 0*, we branch to `0x8011d0a0` using the `bne` instruction
- if *the return value is 0*, we continue execution.

this maps directly to the explanation we saw in "The Basics of RNG in Melee" previously, but this time we're doing it live.

we can try stepping through the `get_random_int` call until we get to the return `blr` instruction at `0x803805b8`. if we check the value of the `r3` register, we can see the random result returned:

{{ figure(src="/img/r3-return.png", alt="r3 is returning 0x20", caption="0x20 == 32") }}

`32 != 0`, so this is going to be a turnip. but now that we know how this logic works...what if we got a little mischevious?

# giving peach a buff

navigate to the `cmpwi` instruction at `0x8011d08c`:

{{ figure(src="/img/cmpwi.png", alt="cmpwi instruction", caption="our fate is in our own hands") }}

let's create a new breakpoint:

{{ figure(src="/img/cmpwi-conditions.png", alt="cmpwi breakpoint conditions", caption="this is not tournament legal") }}

instead of using the returned value from the `get_random_int(128)` call, we're just going to set register `r3` to 0 when evaluating this instruction.

let's disable our previous breakpoint for now so we can test this out:

{{ figure(src="/img/disable-old-breakpoint.png", alt="disabling get_random_int breakpoint", caption="otherwise, our game stops on every item pull") }}

hit start, and let's get pulling!

{{ figure(src="/img/buffed-peach.gif", alt="peach is only throwing items", caption="the dream") }}

# final destination, no items, beamswords only

i'm quite fond of beamswords, so let's see if we can make them a little more popular. if we refer to our charts from earlier, we expect that there will be a `get_random_int(6)` call. we can modify our `get_random_int` breakpoint condition accordingly:

{{ figure(src="/img/breakpoint-modify.png", alt="r3 == 6", caption="r3 == 6") }}

if you've followed along this far, hopefully you're getting the hang of the dolphin debugger. hit the `LR == 8011cf0c` address in the callstack, a familiar `bl ->0x80380580` instruction. let's navigate to the *next instruction* (`0x8011cf10`) and create a new breakpoint, setting `r3 = 5` and continuing execution:

{{ figure(src="/img/mr-saturn-breakpoint.png", alt="r3 = 5", caption="r3 = 5") }}

once again, let's disable our `get_rand_int` item RNG call:

{{ figure(src="/img/disable-item-rng.png", alt="disabling item rng breakpoint", caption="we're almost there") }}

and finally:

{{ figure(src="/img/sword-peach.gif", alt="peach only pulls beamswords now", caption="watch out marth") }}

# closing thoughts

nothing here is novel - my goal is only to share the fun things i've been learning about recently. 
i spend much of my time working towards understanding through experimentation, and i'm hoping this blog can serve as a lab notebook.

if you have any questions, comments, or corrections, feel free to send me an email at <djanatyn@gmail.com>. thanks for taking the time to read, take care!
