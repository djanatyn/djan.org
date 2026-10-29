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

i've been implementing this by writing small handlers that read NES memory to interpret the game state. for some context, here's how i handle the main menu:

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

there are a few magic constants in there: `NEW_GAME_VALUE = 0x1`, and `CURSOR_ADDRESS = 0x62`. `CURSOR_ADDRESS` is an offset into the NES memory that can be read:
```rust
pub fn read(&mut self, addr: u16) -> u8 {
    self.control_deck.bus_mut().read(addr)
}
```

and `NEW_GAME_VALUE = 0x1` is the value it shows when the cursor is on the `NEW GAME` button:

{{ <figure src="/img/ff1_world_map_cursor.gif" alt="gif showing 0x62 changing from 0 to 1 in fceux RAM Watch"/> }}

i spend a lot of time in the [fceux](https://fceux.com/web/home.html) NES emulator's debugger to figure out exactly how things change in memory as the player interacts with the game. this has been very fun for me!

navigating two choices is fairly simple, but final fantasy is about more than menus. immediately after we've defeated the character creation screen, before any towns or dungeons, we're placed into the overworld:

{{ <figure src="/img/ff1_world_map_overworld.png" alt="ingame overworld map in final fantasy 1 for the NES" />}}

while we can only see a small section of the overworld in normal gameplay, the entire map is quite expansive at 256x256 tiles:

{{ <figure src="/img/ff1_world_map_mikes_rpg_center_map.png" alt="final fantasy 1 overworld map" caption="world map from [Mike's RPG Center](https://mikesrpgcenter.com/ffantasy/maps/worldmap.html)" />}}

how do we navigate this world? what is it made out of? how do we get to corneria?

# browsing the archives

fortunately for me, final fantasy 1 has been researched by fans for decades. [the cutting room floor](https://tcrf.net/The_Cutting_Room_Floor) wiki has a section detailing the ROM map, describing the [world map data format](https://datacrystal.tcrf.net/wiki/Final_Fantasy/ROM_map#World_map_data) that we're dealing with here:

> World map data is stored in ROM Bank `0x01`, which is accessed by the CPU at address `0x8000`, and located at `0x4000` in the ROM image. (`0x4010` for iNES files). 

i'll be using [`rizin`](https://rizin.re/) to follow along with the wiki's description:

{{ <figure src="/img/ff1_world_map_pointer_table.png" alt="final fantasy 1 overworld map" caption="world map data as raw bytes" />}}

it's still unclear what this data represents, but the [`final fantasy/world map data`](https://datacrystal.tcrf.net/wiki/Final_Fantasy/World_map_data) page goes into more detail on these row pointers:

> From `$4010` to `$4210` there are 256 little-endian 16-bit integer values representing addresses in the NES memory map (with the `$4010` ROM bank currently in place) that mark the left-hand-side beginnings of each 256-tile row of the map.

knowing that they're [little-endian](https://en.wikipedia.org/wiki/Endianness) 16-bit integers, we can start translating bytes like `0x00` and `0x82` into the address `$8200`!


we can [define a struct](https://book.rizin.re/src/analysis/types.html) with rizin to take a look at these values:

```c
struct Location { int16_t address; }
struct WorldMapTable { Location address[256]; }
```

{{ <figure src="/img/ff1_world_map_rizin_structs.png" alt="defining structs in rizin" caption="translating bytes into pointers" /> }}

this is a [pointer table](https://www.nesdev.org/wiki/Pointer_table). this is intuitive: the game doesn't need to load all 256 rows of the map into memory, only the rows that are nearby the player's current world map coordinate.

you might notice that while each row is 256 tiles, the first row is at `0x8200`, and the second row is at `0x8203`. how could 256 tiles fit into 3 bytes?

> The map data is not stored raw, but uses simple compression in a kind of run-length encoding.

# run-length encoding

## saving bytes

## bank size

# writing a parser with a finite-state machine

## interpreting tiles

## testing a row

# generating a graphic for the world map

# extracting world tile properties

# building a topological map

```rust
```
