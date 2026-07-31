# CLAUDE.md

Working notes for this repo. Base is `pokeemerald` (detached HEAD at tag `Release3.4`,
commit `d271db57c`) carrying the **tx_randomizer_and_challenges** fork — a randomizer /
Nuzlocke / challenge-modes hack. Fork code is tagged with a trailing
`//tx_randomizer_and_challenges` comment; match that convention in new code.

## Building

`arm-none-eabi-*` is **not** on the default PATH. Every build needs:

```bash
export DEVKITPRO=/opt/devkitpro
export DEVKITARM=/opt/devkitpro/devkitARM
export PATH="/opt/devkitpro/devkitARM/bin:$PATH"
```

Then:

```bash
make tidymodern && make modern -j8    # produces pokeemerald_modern.gba
```

- **Always `MODERN=1`.** `make` (MODERN=0, agbcc) **does not build this fork** — C89
  declaration-after-statement errors in `src/battle_controller_player.c` (`TypeEffectiveness`)
  and `-Werror` ambiguous-else in `src/battle_factory.c`. Don't waste time there.
- **Prefer `tidymodern` over `clean`.** `tidymodern` removes only
  `pokeemerald_modern.{gba,elf,map}` and `build/modern`, keeping compiled tools and
  generated graphics. `clean` forces a much longer rebuild.
- **Distrust incremental builds.** A previously-committed incremental ROM differed from a
  clean rebuild of identical source across 16.87 MB of a 16.94 MB image. Clean builds *are*
  byte-reproducible (verified: two consecutive clean builds → identical SHA-1), so when a
  ROM misbehaves inexplicably, rebuild clean before debugging anything.
- Full clean build takes roughly 4 minutes at `-j8`.

## Toolchain constraints

The installed toolchain is **devkitARM GCC 16.1.0** — far newer than anything this fork was
tested against. Known consequences:

- **Cannot build as `-std=gnu17`.** GCC 16's own `stddef.h:465` fails with
  `'nullptr' undeclared`; its headers require C23. The Makefile sets no `-std` for
  `MODERN=1`, so the project compiles as **`gnu23`** whether you want that or not.
- Modern `CFLAGS` also lack `-fno-strict-aliasing`, while the codebase type-puns heavily
  (`GetMonData`/`SetMonData`, `struct Pokemon` ↔ `BoxPokemon`). Suspect miscompilation before
  suspecting logic when behaviour is inexplicable.
- Extra flags can be tested without editing the Makefile — `CFLAGS` uses `override +=`, so
  `make modern CFLAGS="-fno-strict-aliasing"` appends correctly.
- `RegisterRamReset(RESET_ALL)` is **skipped** on `MODERN=1` ([src/main.c:91](src/main.c#L91)),
  so RAM is never cleared at boot. Uninitialized reads that "work" on older compilers can
  surface as garbage pointers here.
- If crashes persist and no logic bug is found, **install an older devkitARM (GCC 13/14)**.
  That is the highest-value remedy, not further source changes.

## Memory budget — nearly exhausted

```
EWRAM:  261128 / 262144 B   99.61%   (~1016 B free)
IWRAM:   26288 /  32768 B   80.22%
ROM:  19970132 / 32 MB      59.52%
```

- **Adding static `EWRAM_DATA` will likely fail to link.** Put new buffers on the heap via
  `Alloc`/`AllocZeroed` instead (this is why the trainer-card text buffer went into the
  heap-allocated `sData`).
- `SaveBlock1` has ~420 bytes of sector headroom (`STATIC_ASSERT` in
  [src/save.c:79](src/save.c#L79) enforces it).
- IWRAM statics end at `0x030066B0`; the user stack starts at `0x03007F00`, leaving ~6.2 KB
  of stack. Largest fixed stack frame in the binary is 408 bytes, so this is adequate but
  not roomy.

## Debug logging is compiled in

`NDEBUG` is never defined by the Makefile, so `LOG_HANDLER = LOG_HANDLER_MGBA_PRINT`
([include/config.h:34](include/config.h#L34)) is active and `MgbaPrintf` calls throughout the
fork are live. `include/config.h:31` warns these "should not be used in a productive
environment" and may crash non-mGBA emulators or hardware. Target mGBA when testing.

## Uncommitted changes

13 files, all currently unstaged. Three groups:

### 1. Pre-existing bug fixes (unrelated to any feature)

- **Type-effectiveness table truncation.** `gTypeEffectiveness_GenVI` was declared `[366]`
  but initialized with **369** elements, so GCC silently dropped the final
  `TYPE_ENDTABLE, TYPE_ENDTABLE, TYPE_MUL_NO_EFFECT` sentinel row. ~20 loops of the form
  `while (... != TYPE_ENDTABLE)` then ran off the end of the array, feeding arbitrary ROM
  bytes into `sTypeEffectivenessList[]` (a 19-byte array, unchecked). This is the **default**
  table (`TX_MODE_TYPE_EFFECTIVENESS 0` = Gen 6), so it ran in every battle. Fixed to `[369]`
  in [src/battle_main.c:541](src/battle_main.c#L541) and
  [include/battle_main.h:97](include/battle_main.h#L97).
- **Starter index read before bounds check.** `GetStarterPokemon` indexed
  `sStarterMon[chosenStarterId]` *before* its guard, and the guard used `>` instead of `>=`,
  so id `STARTER_MON_COUNT` read one `u16` past the 3-entry array. A garbage species flows
  into `CreateMonPicSprite_Affine` → `gMonFrontPicTable[species]` → decompress from a bogus
  pointer. Fixed in [src/starter_choose.c:361](src/starter_choose.c#L361).
- **Uninitialized variable used as an argument.** `PickRandomStarter` passed `species` to
  `RandomSeededModulo` before assigning it. Benign in practice (`RandomSeededModulo`
  overwrites `value` with `Random()` exactly when `tx_Random_Chaos` is set,
  [src/random.c:61](src/random.c#L61)) but UB in shipped code. Initialized to 0 in
  [src/pokemon.c:12030](src/pokemon.c#L12030).
- **Bard music duplicate index** (pre-existing user edit): two `[MOVE_POISON_JAB]` entries
  meant `[MOVE_HEAL_PULSE]` (368) was left zero-filled instead of the `0xff`
  `NULL_BARD_SOUND` sentinel. Second entry re-keyed in `src/data/bard_music/moves.h`.

### 2. Nuzlocke cemetery withdrawal (pre-existing user edit, extended)

`TX_NUZLOCKE_CEMETERY_WITHDRAW` lets fainted ("cemetery") Pokémon leave the PC before the
player is Champion; `SetPlacedMonData` clears `MON_DATA_NUZLOCKE_RIBBON` on the way out.

### 3. Feature: Nuzlocke revive counter

Withdrawing a fainted Pokémon from the PC costs one revive; the count shows on the trainer
card. Config: `TX_NUZLOCKE_REVIVES` (default 1) in
[include/tx_randomizer_and_challenges.h](include/tx_randomizer_and_challenges.h).

**Stored as revives *spent*, not remaining** — `tx_Nuzlocke_RevivesUsed:4` in
[include/global.h:1185](include/global.h#L1185). This is deliberate: a save written before the
field existed reads 0, which correctly means "full allowance unspent". Storing *remaining*
required the new-game defaults block to have run, so pre-existing saves showed 0 revives.
**Don't invert this back.**

Arithmetic is centralised in two helpers
([src/tx_randomizer_and_challenges.c:101](src/tx_randomizer_and_challenges.c#L101)) so the
display, gate and spend site cannot drift:

- `GetNuzlockeRevivesLeft()` — clamps at 0, so lowering `TX_NUZLOCKE_REVIVES` below an
  already-spent count can't underflow.
- `SpendNuzlockeRevive()` — stops at 15 rather than wrapping the 4-bit field back to a full
  allowance.

Wiring:

| Concern | Location | Note |
|---|---|---|
| Gate | `IsMonNuzlockeLocked`, [pokemon_storage_system.c:2550](src/pokemon_storage_system.c#L2550) | Champion status still unlocks unconditionally |
| Spend | `SetPlacedMonData`, [pokemon_storage_system.c:6549](src/pokemon_storage_system.c#L6549) | Guarded on `sMovingMonOrigBoxId != TOTAL_BOXES_COUNT` so reshuffling a fainted mon *within* the party doesn't spend a charge |
| Display | `PrintNuzlockeRevivesOnCard`, [trainer_card.c:1374](src/trainer_card.c#L1374) | Card **back**, row 5 |
| New-game init | [tx_rac_menu.c:1495](src/tx_rac_menu.c#L1495) | Sets used = 0 |

Display gotchas worth preserving:

- Row 5 is free only because `PrintBattleFacilityStringOnCard` is commented out in
  `PrintAllOnCardBack`. It is also the **last** row window 1 can fit (height 18 tiles =
  144 px; row 5 spans y 113–129). Row 6 would be clipped.
- Gated on `!sData->isLink` — the count lives in `SaveBlock1`, not `struct TrainerCard`, so
  without this a linked player's card would show the local player's revives.
- Gated on `tx_Challenges_Nuzlocke || tx_Nuzlocke_EasyMode`, not `IsNuzlockeActive()`. Mini
  mode also sends fainted mons to the cemetery
  ([tx_randomizer_and_challenges.c:291](src/tx_randomizer_and_challenges.c#L291)) so it spends
  revives too; keying off the settings keeps the count visible after becoming Champion.
- **Both nuzlocke modes default to off** (`TX_NUZLOCKE_NUZLOCKE 0`, `TX_NUZLOCKE_MINI_MODE 0`).
  A fresh save that skipped the challenge menu correctly shows no counter.

Note on `CB2_InitTxRandomizerChallengesMenu` case 6: that defaults block runs **every time**
the challenge menu opens, resetting all `tx_*` settings. In normal play it is only reachable
from the new-game Birch speech; the other entry point,
`Task_ChooseChallenge_NoNewGame`, exists solely for `data/scripts/debug.inc:52`. Don't put
anything there that must survive mid-run.

## Verification status

**No GBA emulator is installed in this environment**, so nothing below has been verified at
runtime by Claude. All build-level claims (compiles clean, no warnings, size asserts pass,
reproducible SHA) *are* verified.

- Confirmed by the user in-game: the revive row renders on the card back and the nuzlocke
  gate passes.
- **Not yet confirmed:** that spending a revive works — withdraw a fainted Pokémon from the
  PC, expect the count to drop to 0 and a second cemetery withdrawal to be refused.
- **Unresolved:** a crash reported at startup and at starter selection, which mGBA reported as
  a jump to invalid address `0x15741574`. That word appears nowhere in the ROM, so the bad
  pointer comes from RAM. The two starter-path fixes above are plausible causes but
  **unconfirmed**. Several theories were disproven with evidence: stale build, `SaveBlock1`
  growth, the type-table sentinel (battle-only), memory-region overflow, and stack overflow.
- To pinpoint a future crash: mGBA **Tools → Open debugger console**, `r` for registers. **LR**
  names the calling function; resolve it against `pokeemerald_modern.map`. `PC` alone only
  gives the invalid target.
