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

i've started playing final fantasy 1 several times over the years, but i've never completed the game. the death mechanics are brutal, i get bored grinding for levels, and i was too prideful to use the penninsula of power.

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

as much fun as menu-ing is, it's just the start. navigating two choices is fairly simple, but final fantasy is about more than menus. it's got a whole world to explore!

immediately after we've defeated the character creation screen, before any towns or dungeons, we're placed into the overworld:

{{ <figure src="/img/ff1_world_map_overworld.png" alt="ingame overworld map in final fantasy 1 for the NES" />}}

while we can only see a small section of the overworld in normal gameplay, the entire map is quite expansive at 256x256 tiles:

{{ <figure src="/img/ff1_world_map_mikes_rpg_center_map.png" alt="final fantasy 1 overworld map" caption="world map from [Mike's RPG Center](https://mikesrpgcenter.com/ffantasy/maps/worldmap.html)" />}}

how do we navigate this world? what is it made out of? how do we get to corneria?

# browsing the archives

any time i have a question about the internal workings of an older console game, i look to the romhackers.

final fantasy 1 was released in 1987, so there's been several decades of fans researching the code for the game itself. there are several decompilations

# run-length encoding

## saving bytes

## bank size

# writing a parser with a finite-state machine

## interpreting tiles

## testing a row

# generating a graphic for the world map

# extracting world tile properties

# building a topological map

# 

```rust
```
