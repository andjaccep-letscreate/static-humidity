# BINARY CONNECTION — Crossover Save Specification
**Metaling Through Time · Build & Production**
Schema version: 2 · Status: **REGISTRY FROZEN** · Audited 2026-09-10

Supersedes all earlier save drafts. Volume Two is the first emitter of this
format.

---

## 1. In-world framing

Binary Connection is Agency equipment, a function of the quartz pocket watch.
Never a settings menu.

- Menu entry: **BINARY CONNECTION**
- Retrieve: "Bind this run to your watch."
- Restore: "Restore a bound run."
- Success: "Connection established. Crew restored."
- Failure: "Connection incomplete. That signal is missing pieces."

No player-facing copy uses "save code", "import", "export", or "localStorage".

---

## 2. Architecture

| Layer | Role | Trusted |
|---|---|---|
| **The code** | Source of truth. The player's portable run file. | Yes |
| **localStorage** | Convenience cache so Continue works. | Never |

**The cache stores the code string itself, nothing else.** One format, one
decoder, one validator. There is no second "internal" save shape to drift.

Every storage access is wrapped in try/catch — in blocked browsers it *throws*,
it does not return null. The game is fully playable with storage permanently
unavailable.

**Why:** the game runs in a cross-origin iframe on itch.io, where browsers
partition or clear storage; Safari deletes unused site data after about a week
and we are mobile-first; itch.io and GitHub Pages are separate origins and can
never read each other's storage. The code lives with the player instead.

### Arriving without a code is the normal path
Volume Two is most players' first contact with the series.

- **New Game is the primary action on the title screen.** Continue appears only
  when volume progress exists.
- **Restore lives inside the menu, never on the title screen.** A prominent
  paste box would make every new player feel they are missing something.
- No prompt or explanatory text about codes before the ending.
- Binary Connection is introduced diegetically during play, then offered at the
  end.

### Volume progress is outside the contract
`mtt.vol2.progress` holds where the player is inside this game — zone integer,
in-zone state. It **embeds a copy of the current code string** so that Continue
is self-contained even if `mtt.crew` was cleared independently. It dies with the
game and is never read by any other game.

### Tamper resistance is deliberately out of scope
A player can hand-edit a code and grant themselves gear. The game is free, the
model is donations, and there are no leaderboards or competitive surface. Not one
byte is spent defending against this. The checksum exists to catch *accidental*
corruption, not cheating.

---

## 3. The registry — the spine of the format

One append-only table maps every fact to a fixed bit position.

### Allocation map

| Range | Purpose | Status |
|---|---|---|
| 0–3 | Founding crew | Assigned |
| 4–5 | **Retired, permanently unassigned** (see note) | Burned |
| 6–15 | **Recruited crew — one per game, ten games** | Partly assigned |
| 16–31 | vol1 gear and abilities | Reserved |
| 32–63 | vol2 gear | Partly assigned |
| 64–95 | vol2 abilities | Reserved |
| 96–159 | vol3 gear and abilities | Reserved |
| 160–223 | vol4 gear and abilities | Reserved |
| 224+ | Allocated per game on the same pattern | Reserved |

**Note on bits 4–5:** an earlier draft used these for `vol1.cleared` and
`vol2.cleared`. That left no room for vol3 onward. Game completion now lives in
the Endings section (Section 5), which scales to any number of games. Bits 4–5
are retired before anything ships and are never reused.

**Recruitment canon: exactly one character becomes playable per game.** The
other freed bosses return to their lives — they are not playable and need no
bits. Bits 6–15 cover ten games of recruits in one contiguous block.

**A recruit bit means "recruited," not "playable here."** Whether a recruited
character can be rostered in the current game is decided by that game's roster
table. The chemist's bit is set by Volume Two; she is playable from Volume Three.

**High ranges are free.** The flags section trims trailing zero bytes, so
reserved-but-unused bits add zero characters to a player's code. Generous
reservation costs nothing and is always the correct call.

### Assigned rows

```
BIT   ID                          KIND
---   -------------------------   ----------
  0   crew.gold                   crew
  1   crew.silver                 crew
  2   crew.titanium               crew
  3   crew.platinum               crew
  4–5  RETIRED — never assign

  6   vol2.rec.chemist            recruit   (playable from vol3)
  7–15  (one recruit per game, vol3 onward)

 16–31  RESERVED — vol1, assigned when vol1 is remade

 32   vol2.acc.titanium.chip      accessory
 33   vol2.acc.silver.cleat       accessory
 34   vol2.acc.platinum.glasses   accessory
 35   vol2.acc.gold.<tbd>         accessory
 36   vol2.acc.gold.<tbd>.mod     accessory   (modified variant, separate ID)
 40   vol2.wpn.gold.<tbd>         weapon
 41   vol2.wpn.silver.<tbd>       weapon
 42   vol2.wpn.titanium.<tbd>     weapon
 43   vol2.wpn.platinum.<tbd>     weapon

 64   vol2.abl.<owner>.<tbd>      ability
```

**Three rules, permanent:**

1. **Append only.** New items take the next free bit inside their reserved range.
2. **Never reorder, reuse, or delete.** A retired item's bit stays burned.
3. **Ranges are reserved per game** so two games in development cannot collide.

Rule 1 makes backward compatibility structural rather than implemented: an older
code is simply shorter, and absent trailing bits read as 0, meaning "not owned."

### The character bit set

```
CHARACTER BITS = {0, 1, 2, 3} ∪ {6, 7, 8, 9, 10, 11, 12, 13, 14, 15}
```

Fourteen positions, frozen. Bits 4–5 are excluded and never counted. Section 5
sizes the per-character sections from this set.

### Internal identifiers never follow marketing
`vol1`, `vol2`, `vol3` are permanent internal ordinals assigned at Phase 0.
Public titles no longer carry volume numbers and may change again. **A data field
must never depend on a store-page decision.**

A separate **display table** maps `vol1` → its current public title. Retitling
edits one line there. Player-facing strings are never built by concatenating a
registry ID. `canon-warden` will block any commit that does so or that alters an
existing registry row — **that hook does not exist yet; until it does, this
protection is manual review.**

---

## 4. What is stored

Facts only. Never numbers.

| Stored | Not stored |
|---|---|
| Crew roster and identity | HP, ATK, DEF, SPD |
| Recruited characters | Level, XP |
| Unlocked abilities | Gold, consumables |
| Collected gear and artifact IDs | Position, zone progress, battle state |
| Equipped slot assignments | Playtime, settings, audio volume |
| Weapon tier per character | |
| Game completion and ending tier | |

Stats are rebuilt on every load from the current game's level band table. This
is the locked charter rule — real progression is tools, not numbers — and it is
what lets us rebalance any game later without invalidating a single code.

**Completion is monotonic.** A replay can raise an ending tier; it can never
lower one or clear a completion. Re-finding owned gear sets an already-set bit,
which is naturally idempotent — no double-reward logic is needed.

### 4.1 EXCLUSION — the Agency pocket watch is not gear

Every detective carries an Agency-issued quartz pocket watch, personalized and
bound to them, permanent and unlosable. It is an **inherent character property**.

- It is **never** a weapon.
- It is **never** an accessory.
- It **never** occupies a slot.
- It is **never** in the registry and needs no bit, because possession is
  universal and unconditional — a bit whose value is always 1 stores nothing.

**Why this exclusion is written down:** each character has exactly two accessory
slots, and those slots are where the entire episode-exclusive collection hook
lives. Modelling the watch as an accessory would permanently consume half of
that space to represent something every character always has. The watch is
character definition, not inventory.

### 4.2 EXCLUSION — bonded equipment is a character property

The same exclusion covers anything permanently bonded to one character.

**The chemist's chemistry book** is her weapon, bonded to her, never dropped,
never traded, never found by anyone else. It is **not** a registry entry and does
**not** live in the vol2 gear range. It is part of her character definition,
present the moment her recruit bit is set.

The general rule: **if a thing is possessed unconditionally by whoever owns it,
it stores no information and gets no bit.**

**Three consequences to build against:**

1. A bonded weapon means that character's **weapon slot is fixed, not empty**.
   She cannot equip a different weapon. Her two accessory slots remain fully
   open, so the episode-exclusive collection hook is untouched for her.
2. She still needs a **weapon tier** for level-band sync, so the tier section
   covers her exactly like anyone else. Bonded means unchangeable, not absent.
3. **A bonded weapon slot always encodes as 0 in the Equipped section.** The
   game's roster table knows the slot is bonded; the code does not need to.

---

## 5. The code format

```
MTT<version>-<base64url payload, no padding>
```

- `MTT` — series prefix; a foreign code is rejected on sight
- `<version>` — all digits between `MTT` and the **first** dash. Currently `2`.
  Base64url may itself contain dashes, so the parser splits on the first one only.

### Payload sections, in order

| # | Section | Size | Contents |
|---|---|---|---|
| 1 | Flags | 1 length byte + N bytes | Registry bitfield, little-endian bit order, trailing zero bytes trimmed |
| 2 | Equipped | 3 indices per set character bit | Registry index per slot: weapon, acc1, acc2. 0 = empty or bonded |
| 3 | Tier | 1 byte per set character bit | Weapon tier 1–3 |
| 4 | Endings | 1 length byte + N bytes | Byte *i* = ending tier of game *i+1*. 0 = not cleared, 1–3 = cleared at that tier. Trailing zeros trimmed |
| 5 | Extension | everything remaining | Bytes appended by a newer schema version, preserved verbatim |
| 6 | Checksum | 3 bytes | FNV-1a 32-bit over sections 1–5, truncated to the low 24 bits, written most-significant byte first |

### Checksum byte order
The 24-bit checksum is written **most-significant byte first** (big-endian):
for a truncated value `0xbbbec9` the three bytes are `bb be c9`. FNV-1a uses
offset basis `0x811c9dc5` and prime `0x01000193`. Every fixture in the corpus
carries this order, so it is frozen with the corpus.

### Index encoding
An Equipped index is **one byte if the registry index is below 128, two bytes
otherwise** (standard base-128 varint, canonical minimal form). Volume Two's
gear sits at 32–43, so every current index is one byte. The two-byte form exists
so the format never hits a ceiling as later games push the registry past 255.

### Section sizing — by set bits, never by rostered characters
Equipped and Tier are sized by the **count of set bits in the character bit set**,
in ascending bit order. A code from a future game may carry a recruit this build
does not recognise. That recruit is not rostered but **still occupies its slot and
tier bytes**, which are preserved verbatim.

**Why:** if the decoder sized these sections by the characters it successfully
rosters, a code containing one unknown recruit would count one fewer character
than the emitter wrote, and every byte after that point would be read at the
wrong offset. The checksum would catch it, so nothing corrupts — but a perfectly
valid code would report "Connection incomplete." Counting set bits instead of
rostered characters is what prevents that silent forward-compatibility failure.

### Canonical encoding
The encoder is deterministic: same state, same bytes, every time. Minimal varints,
trailing-zero trimming applied exactly as stated, no padding. Every fixture in the
corpus is canonical.

### Expected length
A complete Volume Two run — full crew, chemist recruited, all nine exclusives,
everything equipped — is about **39 bytes, or roughly 55 characters** including
the prefix. An earlier draft estimated 30–45; that was optimistic. It grows by
roughly 5–8 characters per additional game with a fully equipped recruit.
Copy-paste only; never typed.

UI: large read-only field, **Copy button mandatory**, native share sheet where
cheap. Retrievable from the menu at any time, and presented at the ending.

---

## 6. Schema, field by field, with defaults

Initialize to these defaults **first**, then overwrite with code data. No field
is ever `undefined`.

| Field | Type | Default | Notes |
|---|---|---|---|
| `s` | int | `2` | From the prefix |
| `crew.gold.unlocked` | bool | `true` | Always present |
| `crew.silver.unlocked` | bool | `false` | |
| `crew.titanium.unlocked` | bool | `false` | |
| `crew.platinum.unlocked` | bool | `false` | |
| `crew.<recruit>.unlocked` | bool | `false` | One per recruit bit; "recruited," not "playable here" |
| `crew.*.abilities` | array | `[]` | Resolved from ability bits |
| `gear.owned` | array | `[]` | Positive-only; never stores "not found" |
| `gear.equipped.*.weapon` | id\|null | `null` | Always null for a bonded slot |
| `gear.equipped.*.acc` | array | `[null, null]` | Two accessory slots |
| `gear.tier.*` | int | `1` | Weapon tier 1–3 |
| `vols.<n>.tier` | int | `0` | 0 = not cleared, 1–3 = ending tier. Holds every index found in Endings, known games or not |
| `unknownBits` | array | `[]` | Registry bit positions set in the code that this build does not recognise |
| `unknownRecruits` | map | `{}` | Bit → `{ slots: [i, i, i], tier: t }` for set character bits this build cannot roster |
| `extension` | bytes | empty | Everything between Endings and the checksum, verbatim |

**Absence is the default.** No negative fact is ever stored. A player who never
found the Zone 1 chip has bit 32 at 0 — no handling required.

### The rebuild rule — how round-trip fidelity is guaranteed

**The encoder re-serialises from scratch every time. It never patches bytes.**

Every section can change length between one encode and the next: Flags grows
when the first gear is found, Equipped and Tier gain bytes *in the middle* when
a recruit bit is set, Endings grows when a game is cleared. Patching a retained
payload at fixed offsets would corrupt everything downstream the first time a
player picked up the Zone 1 chip.

So unknowns are preserved **as data, not as bytes at offsets**, and re-emitted at
whatever their correct offset is in the new payload:

- `unknownBits` are set in the rebuilt Flags at their original bit positions,
  extending the section if needed.
- `unknownRecruits` are emitted in ascending bit order alongside rostered
  characters, each contributing its three slot indices and tier byte exactly as
  read.
- `vols` entries this build does not know are written at their index in Endings.
- `extension` is appended verbatim after Endings.

Round-trip fidelity still holds: an unchanged state rebuilds to identical bytes
because the encoder is deterministic. It comes from rebuilding, not patching.

### Accessory dependency rule
Each accessory declares what it needs (`requires: "titanium.drone"`). If that is
absent in the current game, the item stays owned, stays visible in the
collection, and shows as **dormant** with a plain-English reason. Never an
error, never dropped, never silently inert.

---

## 7. Load / fallback / error handling

**Step 1 — Defaults.** Build the full structure from Section 6. The game is
playable before anything is read.

**Step 2 — Acquire.** Try `mtt.crew`, then `mtt.crew.bak`, both in try/catch.
Throw or absent means no cache — proceed silently to a normal title screen.

**Step 3 — Validate envelope.** Normalise first: trim leading and trailing
whitespace (interior whitespace is **not** removed — a valid code never contains
any, and a line-wrapped paste fails the checksum like any other corruption),
then strip a repeated `MTT<digits>-` prefix. Then: prefix must be `MTT`, version
must be digits, total length must be under **512 characters**, checksum must
match.

**Two different failure behaviours, deliberately:**
- A **pasted** code that fails → "Connection incomplete", and the existing crew
  is left exactly as it was.
- A **cached** code that fails → silent. Copy it to `mtt.crew.quarantine` (only
  if that key is empty — it is never overwritten and never multiplies), then
  continue with a clean crew. No message. Nothing about codes is shown before the
  ending.

**Step 4 — Decode.** Base64url → bytes. Read sections 1–4 by the rules in
Section 5. Everything between the end of Endings and the last 3 bytes is
`extension`. Set registry bits this build does not recognise go to
`unknownBits`; set character bits this build cannot roster go to
`unknownRecruits` with their slot indices and tier byte; every Endings index goes
to `vols`, known game or not.

**Step 5 — Version handling.** Version `2` loads directly. A **lower** version
runs the migration chain — none exists yet, but the plumbing ships now so the
first real migration is a one-function change. A **higher** version enters
forward mode: read sections 1–4 exactly as version 2 defines them (a newer version
may only ever *append* sections, never alter these), treat everything else as
Extension, and keep the original version number on write-back. **Never reject a
valid older code.**

**Step 6 — Resolve defensively.** `unknownBits` are reported as unrecognised
and never resolved. Equipped indices pointing at unowned or unknown items →
slot cleared. Weapon index in an accessory slot, or an item on the wrong
character → slot cleared. Non-zero index in a bonded slot → cleared. Tier outside
1–3 → clamped. Ending tier outside 0–3 → clamped. Nothing throws.

**Step 7 — Sync to band.** Stats from the band table plus weapon tier;
abilities and accessory effects attached; unmet dependencies rendered dormant.

**Step 8 — Write cache.** Copy the **previous** contents of `mtt.crew` into
`mtt.crew.bak`, then write the new code to `mtt.crew`. Both in try/catch. A
failed write is survivable: the game continues and mentions once, calmly, that
the player should bind their run before leaving.

---

## 8. Failure cases

Every case ends in a playable game. None may hang, crash, wipe, or show a raw
error.

| # | Case | Required outcome |
|---|---|---|
| 1 | No code, no cache (the common path) | Clean crew, silent, zero prompts |
| 2 | localStorage throws (Safari ITP, blocked cookies) | Clean crew, game normal |
| 3 | Quota exceeded on write | Continue, one calm notice |
| 4 | Cache present but empty string | Clean crew, silent |
| 5 | Cache present, not decodable | Quarantined, clean crew, silent |
| 6 | Quarantine already occupied, second bad cache | First quarantine untouched |
| 7 | Safari wiped storage after 7 days | Clean crew; pasted code restores fully |
| 8 | Player has only ever played this game | Fully normal throughout |
| 9 | Pasted code, wrong prefix | Rejected, current crew untouched |
| 10 | Pasted code truncated | Checksum fails, "incomplete" |
| 11 | Pasted code, one character altered | Checksum fails, "incomplete" |
| 12 | Trailing whitespace or newline | Trimmed, loads |
| 13 | Prefix pasted twice | Normalised, loads |
| 14 | Payload containing dashes | Split on first dash only; loads |
| 15 | Lower version code | Migration chain runs |
| 16 | Version 99 code with extra trailing section | Forward mode; Extension preserved |
| 17 | Registry bit set beyond this build's table | Preserved in `unknownBits`, inert |
| 18 | Equipped index → unowned item | Slot cleared |
| 19 | Equipped index → unknown item | Slot cleared, ownership bit preserved |
| 20 | Weapon index in accessory slot | Slot cleared |
| 21 | Item equipped to wrong character | Slot cleared |
| 22 | Non-zero index in a bonded slot | Slot cleared, bonded weapon retained |
| 23 | Two-byte index (registry ≥ 128) | Decodes correctly |
| 24 | Chip owned, Titanium has no drone kit | Dormant, explained, retained |
| 25 | Both base and modified Gold item owned | Both retained, one equippable |
| 26 | Tier byte out of range | Clamped to 1–3 |
| 27 | Ending tier byte out of range | Clamped to 0–3 |
| 28 | Code says this game already cleared | Replay allowed; tier never lowered |
| 29 | Absurdly long code (≥ 512 chars) | Rejected before decode |
| 30 | Cache and pasted code both present | Player chooses; never auto-merged |
| 31 | Flags length byte claims more bytes than exist | Checksum fails, rejected cleanly |
| 32 | Flags section is zero bytes long | Valid: nothing owned, Gold only |
| 33 | Endings section longer than games this build knows | Extra bytes preserved verbatim |
| 34 | **Round trip**: decode a canonical code, re-encode | **Byte-identical** |
| 35 | Unknown recruit bit set | Loads with no checksum error; slot and tier bytes preserved; round-trips byte-identical |
| 36 | Unknown recruit **and** three games cleared | Character count ignores Endings; offsets correct |
| 37 | Progress exists, crew cache missing | Continue works from the embedded code |
| 38 | **Growth with unknowns**: code has an unknown recruit + unknown registry bits + extension; player then acquires new gear and re-encodes | Every preserved unknown intact at its **new** offset; decoding the result yields the same unknowns plus the new gear |
| 39 | Same as 38, then the player also clears the game | Endings grows; unknowns still intact |

Case 34 protects every future game. Cases 38–39 are what prove the rebuild rule
works when sections change length. Round-trip fidelity is mandatory.

---

## 9. The regression harness

**`index.html?test=save`** — runs the whole matrix, prints a plain PASS/FAIL
list on screen. One URL, readable on a phone, no terminal.

**`index.html?decode=<code>`** — prints any code in plain English: crew,
recruits, abilities, gear, games cleared, equipped slots, and any unrecognised
bits. This is how a non-coder debugs a bitfield and how support answers a player
email.

### The corpus
Archive a real code from **every shipped build at several progress points** —
early, mid, complete, 100%. Committed to the repo and embedded in every future
game's test block, which ships the entire accumulated set.

**Verification means correctness, not absence of error.** Each fixture is stored
alongside its expected decoded result; the test asserts the data *matches*. A
code that loads into the wrong crew is a failure, not a pass.

**Corpus rule: append-only, forever.** Add fixtures; never edit or delete one.
If a fixture stops passing, the build does not ship.

`balance-verifier` will run `?test=save` alongside the 200-run combat simulation
before any merge and block on either failing. **That hook does not exist yet.
Until it does, Andres opens `?test=save` on his phone before every merge and
does not merge on any FAIL. Manual, but non-negotiable.**

---

## 10. What "frozen" means

The registry is frozen as of this document. Precisely:

**Frozen — never change:**
- The range allocation map in Section 3
- The character bit set {0–3, 6–15}
- Every row that has an assigned bit number
- The bit number of any ID already written down
- The section order, sizing rules, index encoding, and checksum byte order in
  Section 5
- The rule that a newer version may only append sections after Endings

**Not frozen — normal work:**
- Filling an unnamed ID inside an already-reserved range. Naming
  `vol2.acc.gold.<tbd>` once Zone 4 is designed is an **append**, not a schema
  change, and needs no review.
- Renaming an internal ID string that has never shipped in a build.
- Adding rows to the end of a reserved range.
- Everything in the display-name table, always.

This distinction exists so we neither freeze an incomplete design nor block the
freeze on unfinished design work.

---

## 11. Build order

1. ~~Review and freeze the registry~~ — **done, audited**
2. ~~Lock this spec into CANON.md~~ — on merge
3. **Branch: encoder, decoder, defaults, load pipeline, harness — no Volume Two
   content attached.** ← current
4. `?v=N` preview → phone playtest → sign-off → merge
5. Slot system, separate phase
