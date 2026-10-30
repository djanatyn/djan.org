+++
title = "parsing the final fantasy 1 NES world map"
date = 2026-10-26
draft = false

[taxonomies]
categories = ["lab notebook"]

[extra]
toc = false
code_copy = true
comment = false
+++

i've started playing final fantasy 1 several times over the years, but i've never completed the game. the death mechanics are brutal, i get bored grinding for levels, and i was too prideful to use the [penninsula of power](https://strategywiki.org/wiki/Final_Fantasy/The_Journey_Begins#Power_peninsula_(optional)).

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

there are a few magic constants in there: `NEW_GAME_VALUE = 0x1`, and `CURSOR_ADDRESS = 0x62`. `CURSOR_ADDRESS` is an offset into the NES memory that can be read:
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

while we can only see a small section of the overworld in normal gameplay, the entire map is quite expansive at 256x256 tiles:

{{ <figure src="/img/ff1_world_map_mikes_rpg_center_map.png" alt="final fantasy 1 overworld map" caption="world map from [Mike's RPG Center](https://mikesrpgcenter.com/ffantasy/maps/worldmap.html)" />}}

how do we navigate this world? what is it made out of? how do we get to corneria?

# browsing the archives

fortunately for me, final fantasy 1 has been researched by fans for decades. [the cutting room floor](https://tcrf.net/The_Cutting_Room_Floor) wiki has a section detailing the ROM map, describing the [world map data format](https://datacrystal.tcrf.net/wiki/Final_Fantasy/ROM_map#World_map_data) that we're dealing with here:

> World map data is stored in ROM Bank `0x01`, which is accessed by the CPU at address `0x8000`, and located at `0x4000` in the ROM image. (`0x4010` for iNES files). 

i'll be using [`rizin`](https://rizin.re/) to follow along with the wiki's description:

{{ <figure src="/img/ff1_world_map_pointer_table.png" alt="final fantasy 1 overworld map" caption="world map data as raw bytes" />}}

the [`final fantasy/world map data`](https://datacrystal.tcrf.net/wiki/Final_Fantasy/World_map_data) page describes the format of this data in more detail, so we can start to make sense of it:

> From `$4010` to `$4210` there are 256 little-endian 16-bit integer values representing addresses in the NES memory map (with the `$4010` ROM bank currently in place) that mark the left-hand-side beginnings of each 256-tile row of the map.

that means that this is a list of addresses. knowing that they're [little-endian](https://en.wikipedia.org/wiki/Endianness) 16-bit integers, we can start translating sequences of bytes like `0x00` and `0x82` into the address `$8200`.

we can [define some structs](https://book.rizin.re/src/analysis/types.html) with rizin to inspect these values within the ROM:
```c
// a pointer to a world map row
struct Location { int16_t address; }
// a collection of 256 pointers
struct WorldMapTable { Location address[256]; }
```

{{ <figure src="/img/ff1_world_map_rizin_structs.png" alt="defining structs in rizin" caption="translating bytes into pointers" /> }}

this is a [pointer table](https://www.nesdev.org/wiki/Pointer_table). the game doesn't need to load all 256 rows of the map into memory, only the rows that are nearby the player's current world map coordinate.

you might notice that while each row is 256 tiles, the first row is at `0x8200`, and the second row is at `0x8203`. how could 256 tiles fit into 3 bytes?

# banking

let's take a look at the bytes, starting with the first pointer, `$8200`. that pointer is a [CPU address](https://www.nesdev.org/wiki/CPU_memory_map), in the `$4020-$FFFF` range that is reserved for cartridge use:

> Since this bank will loaded into the NES memory map at readable address `$8000`, all memory pointers in the ROM must point to the row's ROM file address + `0x3FF0`. To translate a memory pointer back into a ROM file address, you subtract `0x3FF0` instead.

we need to do a bit of arithmetic to resolve this address, as it assumes the `0x4010` bank is in memory at `$8000`. there's several different banks available on the cartridge, and the [mapper](https://www.nesdev.org/wiki/Mapper) allows the NES to switch between each one:

> NES games come in cartridges, and inside of those cartridges are various circuits and hardware. Different games use different circuits and hardware, and the configuration and capabilities of such cartridges is commonly called their mapper. Mappers are designed to extend the system and bypass its limitations, such as by adding RAM to the cartridge or even extra sound channels. More commonly though, mappers are designed to allow games larger than 40K to be made.

there are lots of different mappers! this [particular cart](https://nescartdb.com/profile/view/154/final-fantasy) uses the [MMC1](https://www.nesdev.org/wiki/MMC1) mapper. that's why we need to subtract `0x3FF0`: to translate from the banked CPU address to the ROM address.

# decompressing an ocean

```c
// [0x00004210]> pc 3 @ 0x8200 - 0x3FF0
#define ARRAY_SIZE 3
const uint8_t array[ARRAY_SIZE] = {
  0x97, 0x00, 0xff
};
```

> The map data is not stored raw, but uses simple compression in a kind of run-length encoding.

> Row data is stored sequentially as such:
> - Byte values indicate a row's tiles from the left-edge, from left-to-right through the tiles, ending at the row's right edge.
> - A single byte ranging `0x00` to `0x7F` describes a single tile of a specific value. The next byte describes the tile to its right.
> - A byte ranging `0x80` to `0xFE` is the same as its value subtracted by `0x80` (or exclusive-or'ed by 0x80), but the immediately following byte indicates repetition of the tile value. A repetition byte value of `0x01` means no repetition, `0x02` means repeat once, `0x03` means repeat twice, 0x04 means repeat three times, and so on. `0x00` means to repeat 256 times, so that it can fill an entire row. A repetition value of `0xFF` is prohibited.
> - A byte value of `0xFF` after the row's final tile terminates the row data.

```
[0x00004210]> % 0x97 & 0b01111111
int32   23
uint32  23
hex     0x17
octal   027
unit    23
segment 0000:0017
string  "\x17"
fvalue  127.0
float   127.000000f
double  127.000000
binary  0b00010111
trits   0t212
```

{{ <figure src="/img/ff1_world_map_ocean.png" alt="an ocean" /> }}

# writing a parser with a finite-state machine

## interpreting tiles

## testing a row

# generating a graphic for the world map

# extracting world tile properties

# building a topological map

```rust
```
