+++
title = "parsing the final fantasy 1 NES world map (part 1)"
description = "how 3 bytes can form an ocean"
date = 2026-10-30
draft = false

[taxonomies]
categories = ["lab notebook"]

[extra]
toc = false
code_copy = true
comment = false
og_image = "/img/ff1_world_map_overworld.png"
+++

i've started playing final fantasy 1 several times over the years, but i've never completed the game. the death mechanics are brutal, i get bored grinding for levels, and i was too prideful to use the [peninsula of power](https://strategywiki.org/wiki/Final_Fantasy/The_Journey_Begins#Power_peninsula_(optional)).

these days, i'm approaching the game from a different angle: i've been using [tetanes](https://github.com/lukexor/tetanes) to write some rust code to [play the game](https://github.com/djanatyn/chaos-seeker) for me!

i've been implementing this by writing small handlers that read NES memory to interpret the game state. as an example, here's how i handle the main menu:

```rust
mod menu {
    use super::*;

    /// Address showing cursor position in menu.
    ///
    /// Cursor can point at CONTINUE, or NEW GAME.
    pub const CURSOR_ADDRESS: u16 = 0x62;
    /// Value of CURSOR_ADDRESS when pointing at NEW GAME.
    pub const NEW_GAME_VALUE: u8 = 0x1;

    fn cursor_new_game(state: &mut GameState) -> bool {
        state.read(CURSOR_ADDRESS) == NEW_GAME_VALUE
    }

    fn at_character_creation(state: &mut GameState) -> bool {
        state.read(char_create::SLOT_SELECTED_ADDRESS) == 0x0
    }

    /// Move cursor to new game and hit start.
    #[instrument]
    pub fn handler(state: &mut GameState) -> GameProgress {
        if at_character_creation(state) {
            tracing::warn!(state.frame, "reached char creation");
            return GameProgress::CharacterCreation;
        }

        if cursor_new_game(state) {
            // hit start for new game
            state.toggle(JoypadBtnState::START);
        } else {
            // move cursor down to new game
            state.toggle(JoypadBtnState::DOWN);
        }

        GameProgress::BootMenu
    }
}
```

this handler will check if we're at character creation. if not, it'll make progress toward that state by checking the cursor position:
- if the cursor is on the "new game" button, it'll press start.
- if the cursor is not on the new game button, it'll press down on the d-pad to move the cursor.

if executed each frame, eventually we'll reach the next phase of gameplay (character creation).

there are a few magic constants in there: `NEW_GAME_VALUE = 0x1`, and `CURSOR_ADDRESS = 0x62`. `CURSOR_ADDRESS` is an CPU address in the NES's internal RAM:
```rust
pub fn read(&mut self, addr: u16) -> u8 {
    self.control_deck.bus_mut().read(addr)
}
```

and `NEW_GAME_VALUE = 0x1` is the value it shows when the cursor is on the `NEW GAME` button:

{{ <figure src="/img/ff1_world_map_cursor.gif" alt="gif showing 0x62 changing from 0 to 1 in fceux RAM Watch"/> }}


i spend a lot of time in the [fceux](https://fceux.com/web/home.html) NES emulator's debugger to figure out exactly how things change in memory as the player interacts with the game. this has been very fun for me!

with that said, navigating two choices is fairly simple. final fantasy is about more than menus. immediately after we've defeated the character creation screen, before any towns or dungeons, we're placed into the overworld:

{{ <figure src="/img/ff1_world_map_overworld.png" alt="ingame overworld map in final fantasy 1 for the NES" />}}

while we can only see a small section of the overworld in normal gameplay, the entire map is quite expansive at 256x256 map tiles:

{{ <figure src="/img/ff1_world_map_mikes_rpg_center_map.png" alt="final fantasy 1 overworld map" caption="world map from [Mike's RPG Center](https://mikesrpgcenter.com/ffantasy/maps/worldmap.html)" />}}

how do we navigate this world? what is it made out of? how do we get to corneria?

# browsing the archives

fortunately for me, final fantasy 1 has been researched by fans for decades. [the cutting room floor](https://tcrf.net/The_Cutting_Room_Floor) wiki has a section detailing the ROM map, describing the [world map data format](https://datacrystal.tcrf.net/wiki/Final_Fantasy/ROM_map#World_map_data) that we're dealing with here:

> World map data is stored in ROM Bank `0x01`, which is accessed by the CPU at address `0x8000`, and located at `0x4000` in the ROM image. (`0x4010` for iNES files). 

i'll be using [`rizin`](https://rizin.re/) to follow along with the wiki's description:

{{ <figure src="/img/ff1_world_map_pointer_table.png" alt="final fantasy 1 overworld map" caption="[world map data as raw bytes](https://gist.github.com/djanatyn/96f20c9b261c4c4c550c698d4bc4f8bc#file-ff1_world_map_rizin_structs-md)" />}}

the [`final fantasy/world map data`](https://datacrystal.tcrf.net/wiki/Final_Fantasy/World_map_data) page describes the format of this data in more detail, so we can start to make sense of it:

> From `0x4010` to `0x4210` there are 256 little-endian 16-bit integer values representing addresses in the NES memory map (with the `0x4010` ROM bank currently in place) that mark the left-hand-side beginnings of each 256-tile row of the map.

that means that this is a list of addresses. knowing that they're [little-endian](https://en.wikipedia.org/wiki/Endianness) 16-bit integers, we can start translating sequences of bytes like `0x00` and `0x82` into the address `$8200`.

we can [define some structs](https://book.rizin.re/src/analysis/types.html) with rizin to inspect these values within the ROM:
```c
// a pointer to a world map row
struct Location { uint16_t address; }
// a collection of 256 pointers
struct WorldMapTable { Location address[256]; }
```

{{ <figure src="/img/ff1_world_map_rizin_structs.png" alt="defining structs in rizin" caption="[translating bytes into pointers](https://gist.github.com/djanatyn/96f20c9b261c4c4c550c698d4bc4f8bc#file-ff1_world_map_rizin_structs-md)" /> }}

this is a [pointer table](https://www.nesdev.org/wiki/Pointer_table). the game doesn't need to load all 256 rows of the map into memory, only the rows that are nearby the player's current world map coordinate.

you might notice that while each row is 256 map tiles, the first row is at `$8200`, and the second row is at `$8203`.

how could 256 map tiles fit into 3 bytes?

# banking, briefly

we can start answering our question by looking at those 3 bytes, at CPU address `$8200`. however, i must note that i'm making a distinction in my notation already: `0x8200` is a 16-bit integer, but `$8200` (with a `$`) is a **16-bit CPU address**.

the CPU address `$8200` does *not* correspond to offset `0x8200` in the ROM. 

addressable memory was limited to 64K on the [6502 microprocessor](https://en.wikipedia.org/wiki/MOS_Technology_6502) that the NES used. before interfacing with a cartridge, much of that memory is [already mapped](https://www.nesdev.org/wiki/CPU_memory_map) to core functionality for the console: drawing pictures, processing audio, performing I/O, and internal RAM. at the end of the memory map, the range `$4020-$FFFF` is reserved for cartridge use.

this range is not large enough to hold the entire final fantasy 1 ROM, even if we left out unnecessary headers:
```
$ du -h ff1.nes
260K	ff1.nes
```

to work around the restriction of limited addressable memory, while simultaneously allowing cartridges to extend the capabilities of the NES itself, the concept of [mappers](https://www.nesdev.org/wiki/Mapper) emerged:

> NES games come in cartridges, and inside of those cartridges are various circuits and hardware. Different games use different circuits and hardware, and the configuration and capabilities of such cartridges is commonly called their mapper. Mappers are designed to extend the system and bypass its limitations, such as by adding RAM to the cartridge or even extra sound channels. More commonly though, **mappers are designed to allow games larger than 40K to be made.**

final fantasy 1's PRG-ROM ("program" read-only memory) contains *16* different banks of *16K* each, for 256K of data. the mapper allows the cartridge to switch between banks dynamically during gameplay, mapping the appropriate bank into the 6502 CPU address space! in our case, there's a 16K bank that begins at file offset `0x4010`, which gets "mapped" into CPU address `$8000`.

there are lots of different mappers! this [particular cart](https://nescartdb.com/profile/view/154/final-fantasy) uses the [MMC1](https://www.nesdev.org/wiki/MMC1) mapper.

---

we're not running a NES emulator in rizin, so we'll need to do some translation from CPU-relative addresses to ROM-relative addresses:

> Since this bank will loaded into the NES memory map at readable address `$8000`, all memory pointers in the ROM must point to the row's ROM file address + `0x3FF0`. To translate a memory pointer back into a ROM file address, you subtract `0x3FF0` instead.

this value `0x3FF0` is just the difference between the CPU address and the bank ROM offset:
```
$ fend '0x8000 - 0x4010'
0x3ff0
```

# decompressing an ocean

alright, let's look at these bytes now that we know where they are:

```c
// [0x00004210]> pc 3 @ 0x8200 - 0x3FF0
#define ARRAY_SIZE 3
const uint8_t array[ARRAY_SIZE] = {
  0x97, 0x00, 0xff
};
```

this row needs to represent 256 map tiles. rather than storing 256 map tiles in sequence, we're using a simple lossless run-length encoding compression scheme:

> The map data is not stored raw, but uses simple compression in a kind of run-length encoding.

there's a brief explanation of the format in the ROM Map documentation:

> Each entry in this data area is either 1 byte or 2 bytes long. The most significant bit of the first byte is set if there is a second byte for this entry. The first byte (excluding the MSB flag) specifies the tile number (0-127). The second byte, if present, specifies the number of tiles that should be displayed. If the second byte is 2, then two tiles should be rendered. If the second byte is 0, then the tile should be rendered 256 times. A single byte entry with the value of 0xFF represents the end of the row.

MSB here stands for ["most significant bit"](https://en.wikipedia.org/wiki/Bit_numbering#Bit_significance_and_indexing).

immediately, we've learned a few things about the first byte:

- if the most significant bit is unset (`0b0XXX_XXXX`), it's a single tile, and
- if the most significant bit is set (`0b1XXX_XXXX`) , it's a repeated tile, and the next byte is its repetition count.

we can use a [bitmask](https://en.wikipedia.org/wiki/Mask_(computing)) to check that bit. `number & 0b1000_0000` will return 0 if the bit is unset:

```
> 0x97 in binary
10010111
> (0x97 and 0b1000_0000) != 0
true
```

it's not unset! because the bit is set, we know **this is a repeated tile**!

to figure out *which* tile is repeated, we can use the same bitmask with an [xor](https://en.wikipedia.org/wiki/Bitwise_operation#XOR) to drop the MSB:

```
> 0x97 xor 0b1000_0000
0x17
```

alternatively, it's also common to clear the MSB with an AND, which gives us the same result:

```
> 0x97 and 0b0111_1111
0x17
```

`0x17` is the tile for [Ocean](https://datacrystal.tcrf.net/wiki/Final_Fantasy/World_map_data#FF1_Overworld_Tile_Values).

{{ <figure src="/img/ff1_world_map_ocean_tile.png" alt="an ocean" /> }}

how much ocean?

> [..] If the second byte is 0, then the tile should be rendered **256 times**. A single byte entry with the value of 0xFF represents **the end of the row.**

`[0x97, 0x00, 0xFF]` is a row that's just 256 map tiles of ocean.

considering the alternative (`[0x17, 0x17, 0x17...]`), we saved a *lot* of bytes. without getting too clever, we'd need 256 bytes for the ocean tiles, and another byte to end the row.

instead, we only needed 3, saving ROM space and reducing ROM reads in exchange for some CPU work!

# to be continued

in our next post, we'll:
- build a finite state machine to parse these tiles,
- render PNG images based on the tile data we parse,
- learn to parse tile properties with bitmasks using 2008 romhacking notes, and
- generate a topological map that we'll use for path-finding!

see you later! i hope you have a great day :)
