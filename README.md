# Depths & Spires — Add-on Modules

Add content without ever editing `depths-and-spires.html`.

## Folder layout (for GitHub Pages)

```
/depths-and-spires.html
/mods/manifest.json
/mods/example-relics.json
/mods/your-new-pack.json
```

## Adding a pack

1. Drop your `.json` file into `mods/`.
2. Add its filename to `mods/manifest.json`:

```json
{ "modules": ["example-relics.json", "your-new-pack.json"] }
```

That's it. Reload the page and the content is live.

> **Why the manifest?** Static hosts (GitHub Pages included) don't expose
> directory listings, so a page can't discover files on its own. The manifest is
> the one line you update — never the game file.

## No server? Import at runtime

Opening the HTML directly from disk (`file://`) blocks folder loading. Use
**Bestiary & Armory → Modules** to import a `.json` file or paste JSON. Handy
on mobile. Runtime imports last for that session.

## Module format

Top level: `name`, `version`, `author`, plus any of `items`, `enemies`, `traps`.
Reusing an existing `id` **overrides** that entry — how you rebalance core content.

### Items

| Field | Notes |
|---|---|
| `id` | required, unique |
| `name` | required |
| `type` | `weapon` / `consumable` / `artifact` / `trinket` |
| `icon` | emoji shown on cards |
| `cls` | `warrior` / `archer` / `thief`, or omit for any class |
| `uses` | number, or `-1` for unlimited (equipped) |
| `rarity` | loot weight; higher = more common (default 1) |
| `usableIn` | `["combat"]`, `["explore"]`, or both |
| `desc` | flavour text |
| `effects` | applied when the player uses it |
| `onHit` | applied automatically when the holder lands a hit |

### Effect kinds

```jsonc
{ "kind": "heal",     "amount": "2d4+2" }
{ "kind": "damage",   "amount": "2d6", "target": "enemy", "free": true }
{ "kind": "buff",     "stat": "atkMod", "amount": 2, "duration": 3 }
{ "kind": "ap",       "amount": 2 }
{ "kind": "roll",     "mode": "advantage", "duration": 2 }
{ "kind": "status",   "status": "bleed", "target": "enemy", "duration": 2 }
{ "kind": "cure",     "status": "all" }
{ "kind": "deckPeek", "count": 4 }
{ "kind": "teleport", "mode": "start" }
```

- `stat` for `buff`: `atkMod` (strength), `def`, `maxHp`, `apMax`, `dmgMult`
- `status`: `bleed`, `stun`, `freeze`, `poison`
- `duration` in rounds; omit for permanent
- `chance` (0–1) on any effect makes it probabilistic
- `free: true` means using it doesn't give the enemy a free swing
- `amount` accepts dice (`"3d4+3"`) or a flat number

### Enemies

`id`, `name`, `hp`, `def`, `dmg`, `icon`, `flavor`, optional `special: "poison"`,
optional `boss: true`.

### Traps

`id`, `name`, `dmg`, `dc`, `icon`, `flavor`.

## Validation

Bad modules are rejected with a specific reason (shown in the Modules tab)
rather than silently breaking the game. Errors are listed per file.


---

# Multiplayer (peer-to-peer)

**Title screen → Play with Friends.**

- **Host a Room** → you get a 5-character code. Share it. You run the game.
- **Join a Room** → enter the code and your name.
- Up to 4 seats. Pick your class in the lobby. Host picks the objective and hits Begin.

## How it works

WebRTC data channels via **PeerJS**, using its free public broker. The broker only
introduces peers — after that, gameplay traffic goes browser-to-browser. Nothing to
deploy, no accounts, works on GitHub Pages.

Architecture is **host-authoritative**: the host runs every rule and owns the only
real game state. Clients send *intents* ("explore east"); the host validates, applies,
and broadcasts a full snapshot. Desync is structurally impossible because there is
only ever one simulation. The trade is one round trip per action — fine for a
turn-based game.

Only the seat whose turn it is can act. Everyone else sees a read-only board and a
"Waiting for …" banner; their controls are hidden and shared prompts are inert.

## Requirements & limits

- **Must be served over HTTPS** (or localhost). WebRTC won't run from `file://`.
  GitHub Pages is HTTPS, so that's covered.
- **If the host leaves, the run ends.** There's no host migration — clients see a
  "Disconnected" banner. Saving/reconnecting isn't implemented.
- Late joiners are caught up automatically with the current snapshot.
- Some restrictive networks (strict corporate NAT/firewall) block direct WebRTC.
  PeerJS's free broker provides no TURN relay, so those cases will fail to connect.
- The public broker is best-effort and rate-limited. For anything serious, run your
  own PeerServer and point `new Peer()` at it.


---

# Session notes: seats, downs, host handover, clock

## Naming
The host enters a name on the lobby screen before creating a room (it used to be
hard-coded "Host"). Clients enter theirs when joining.

## Going down, and getting back up
0 HP no longer removes a character. They go **down**: a skull token stays on the
tile where they fell, they take no turns, and they keep their gear. Any ally who
reaches that tile gets a **✚ Revive** button — one action, back up with 1d4+2 HP,
acting again next round. The run only ends when the **whole party** is down at
once.

## Host handover
If the host drops, clients don't freeze. Every client already holds a full
snapshot, so one can take over:

- The successor is **deterministic** — the connected seat with the lowest number.
  Every client computes the same answer, which matters because clients have no
  connections to each other and can't negotiate.
- The successor opens a new peer id at the next **generation**
  (`dnspires-ABCDE-g1`) and continues from its snapshot. Everyone else
  reconnects to that id, retrying while it comes up.
- The **room code the players see never changes** — only the internal generation.
- The departed host's seat is held, not deleted.

## Rejoining
A dropped player's seat is **held open** mid-run. Reconnecting with the same name
reclaims that seat, character, inventory and position. Seats are matched by name
because peer ids change on every reconnect.

If it's a disconnected player's turn, the turn passes on automatically so the
table isn't stuck waiting.

## Turn clock and pause
Optional limit (off / 30s / 1min / 2min), set in the lobby (multiplayer) or on the
setup screen (single player). Running out **skips the turn**; a tile drawn but not
placed goes back to the deck rather than being lost.

The **host owns the clock** and publishes an absolute deadline — clients render
the countdown from that, so no one's timer drifts. **Anyone can pause**, including
off-turn; pausing freezes the clock and blocks all actions until resumed.

## Known limits
- Host handover has been tested for successor election and state carry-over, but
  **not against real WebRTC** — the sandbox has no network. Expect to shake
  something out on first live use.
- If *every* player disconnects at once, the run is gone. There's no save file.
- Rejoin matches on name, so two players sharing a name in one room will collide.


---

# Session notes: split parties, save files, icon paths

## Levels are per-character now
Taking a stairway moves **only the character who took it**. Everyone else stays
where they are.

- Every floor persists in `STATE.floors`. `STATE.grid` is just the working copy of
  whichever floor is on screen, so the rules code reads it unchanged.
- The board follows whoever is acting — end a turn and it switches to that
  character's floor automatically.
- The **entrance tile of a new floor is a stairway back**, so floors stay linked in
  both directions.
- Characters only appear on the floor they're actually standing on. The Level
  readout in the top bar shows a **⑂** when the party is split.
- Reviving requires being on the same **floor and tile**.
- The 🗺 browser lists every known floor and who is on it. Anything other than the
  floor you're playing is read-only.

## Save & continue
Two independent routes, because some webviews and private-browsing modes block
local storage entirely:

- **Autosave** to this device at the end of every round. The title screen then
  offers **Continue — Level N**.
- **Download save / Load a save file** (JSON) — always works, and is how you move a
  run between devices. Roughly 2-3 KB.

Both are behind the **💾** button in the top bar. In a network game only the host
can save or load, since the host holds the authoritative state.

A save carries everything: every floor, per-character levels and positions,
inventories, trophies, buffs, downed state, the log, and the tile deck.

## Icon paths
The HTML now references **external** icon files rather than inlined base64:

```html
<link rel="icon" href="icon-192x192.png">
<link rel="apple-touch-icon" href="icon-192x192.png">
```

So `icon-192x192.png` must sit **next to the HTML**. `manifest.webmanifest` has been
updated to match. Files provided: `icon-192x192.png` (repo root),
`icons/icon-512x512.png`, `icons/icon-maskable-512.png`.


---

# Module Studio (in-game authoring)

**Title screen → 🛠 Module Studio**, or the **🛠** button in the top bar mid-run.

Build content with forms instead of hand-written JSON:

- **Items** — name, icon, type, class lock, uses, rarity, where it's usable, description.
- **Effect builder** — pick an effect from a dropdown and only the fields that effect
  actually uses appear. Separate lists for *when used* and *when the holder lands a
  hit*, each with an optional chance.
- **Enemies / Traps** — HP, defence, damage dice, DC, flavour, poison, boss flag.
- **Live preview** rendered with the same card component the Codex uses, so what you
  see is what players will see.
- **Validation as you type** — bad dice (`banana`), missing names, duplicate ids and
  effect-less items are all flagged before export.

## Testing without leaving the game

- **▶ Test in game** registers the draft into the running session — it can appear as
  loot and shows up in the Codex.
- **Give to active character** puts an item straight into the current character's pack.
- **Fight it now** stages an encounter with the enemy you're editing.

The Studio is reachable mid-run, so you can tweak a value and re-test in seconds.

## Export

**Download .json**, **Copy JSON** (falls back to a selectable box where clipboard
access is blocked), or **Import…** to load an existing module back in for editing.
The output is exactly the module format the loader already reads — the Studio runs
its output through the same `validateModule` check on the way out.

Drafts autosave to the device, so a half-built pack survives a reload.

## One schema, one source of truth

The effect list in the Studio is generated from the same schema the engine uses.
Adding a new effect kind to the engine makes it authorable in the Studio
automatically — the two can't drift apart.

---

# Fix: joining after the game has started

A player who missed the start — or whose first connection attempt failed — could
never get in. The host was refusing every join once a run existed.

Now it's **drop-in co-op**: a late joiner is added to the party as a new character
and placed at the entrance of whichever floor most of the party is on, with a full
turn of actions. Rejoining a held seat still reclaims the original character rather
than making a duplicate, and a full room (4) is still refused.

Two related robustness fixes:

- **Slow handshakes no longer get skipped.** Room discovery gave each candidate id
  3.5 s and then moved on, which on mobile could step past a room that really was
  there. It now allows 9 s and retries an id before advancing.
- **"Network error: peer-unavailable" is no longer shown during normal probing.** That
  message is an expected outcome while checking for a newer host generation; it's now
  swallowed, and only a genuine "no room found" is reported.
- **Host id collisions self-heal.** Re-hosting quickly could hit a broker-held id and
  leave a dead room on screen; the host now issues a fresh code and says so.


---

# Custom tile themes

**Codex → Tiles** to pick one. **Studio → Tile themes** to make one.

The shipped art is three tones — white floor, grey walls, black outline/bevels. A
theme **remaps those tones** rather than replacing the drawing: the base art is
decoded once into pixel data, then recoloured on a canvas. Every doorway, bevel and
diagonal stays exactly where it was drawn, so a theme can't break tile connections.

Five themes ship built in (Quarried Stone, Mossy Crypt, Ember Forge, Frostbound
Spire, Void Sanctum). Your choice is **local and persists** — it's presentation, not
game state, so it isn't synced to other players and each person can pick their own.

## Authoring a theme

Four colour pickers (floor, wall, wall shadow, outline) with a live 6-tile preview.
Two optional extras:

- **Floor texture** — any image, tiled across floor pixels only.
- **Custom tile art** — override any of the six tiles outright with your own image,
  for anyone who does want to draw their own.

**Apply this theme** switches the live board to it immediately.

## In a module

```jsonc
"themes": [{
  "id": "bloodstone_vault",
  "name": "Bloodstone Vault",
  "floor": "#e8cfc0",
  "wall": "#7a2530",
  "wallShade": "#2a0d11",
  "outline": "#12060a",
  "floorTexture": "data:image/png;base64,…",     // optional
  "tiles": { "Fourway": "data:image/png;base64,…" }  // optional
}]
```

Colours are validated as hex on import.

---

# Realtime mode

Choose **Realtime** on the setup screen, or **Play style → Realtime** in the lobby.
Turn-based play is untouched — both modes coexist and the turn path was re-tested
for regressions.

## What changes

- **No turn order.** Everyone acts whenever their own **cooldown** has elapsed
  (move 1.2s, explore 1.6s, attack 1.8s). The action pool and turn clock are gone;
  the character sheet shows a cooldown instead of actions.
- **Enemies live on the board.** An encounter creates a battle **anchored to its
  tile**, drawn with an icon and a shared HP bar. No modal, so you keep seeing the
  dungeon.
- **Anyone can join a fight.** Walk onto the tile and Attack appears. Damage goes to
  one shared HP pool, and **everyone who fought gets the trophy**.
- **The enemy fights back on its own clock** (~3.2s), picking a random target from
  whoever is standing on its tile — so leaving a fight is a real option.
- Buffs expire on a wall clock instead of on turns. Reviving still needs the same
  floor and tile. Pause still works and freezes everything.

## Honest limits

- Realtime is a **v1**. It's been tested for cooldown gating, two players sharing a
  fight, enemy retaliation, kill resolution, shared trophies and pause — but not for
  balance. Cooldown and enemy tempo numbers (`COOLDOWN`, `ENEMY_TEMPO_MS`) are one
  edit each if the pacing feels wrong.
- Tile *placement* is still one player at a time — whoever drew the tile resolves it.
- **Not tested over real WebRTC.** The host stays authoritative and intents are
  seat-checked as before, but realtime means far more frequent messages than
  turn-based play. Expect to tune sync frequency once you try it on two devices.


---

# Sound

**🔊** in the top bar, or on the title screen.

Every cue is **synthesised with WebAudio at runtime** — no audio files, so the game
stays a single hostable page and a sound can be retuned by editing numbers instead
of re-recording. 22 cues: dice clatter, hit/crit/miss, taking damage, enemy death,
tile placement and rotation, footsteps, stairways, loot, traps, going down, revival,
joining a fight, victory, defeat, UI clicks.

There's also an optional **dungeon ambience** — a slow detuned drone with occasional
drips, on its own volume bus.

Settings (master / effects / ambience volume, mute, ambience toggle) persist to the
device, with buttons to audition cues. Browsers block audio until the player
interacts, so the context opens on the first gesture and everything before that is a
silent no-op.

## Module sounds

A module can ship real samples that override any built-in cue by id:

```jsonc
"sounds": [
  { "id": "hit",  "src": "data:audio/mpeg;base64,…" },
  { "id": "loot", "src": "data:audio/wav;base64,…" }
]
```

Custom samples take priority over the synthesised version. Cue ids are the keys of
`SFX` — `dice`, `hit`, `crit`, `miss`, `hurt`, `enemy_die`, `tile_place`,
`tile_rotate`, `step`, `door`, `stairs`, `loot`, `trap`, `down`, `revive`, `battle`,
`join`, `victory`, `defeat`, `ui_click`, `ui_back`, `error`.

---

# Blend masks (two textures on one tile)

A theme's floor can now be **two layers stencilled by a black & white mask**:
**white keeps texture A, black swaps to texture B**, and greys crossfade between
them. Verified pixel-by-pixel against the mask's own values.

Seven masks are built in and generated procedurally (no image files): `cracks`,
`rubble`, `checker`, `wear`, `stripes`, `speckle`, plus `none`. Or upload your own
mask image — any greyscale picture works, and the Studio shows it next to the tile
preview so you can see exactly what's being stencilled.

Both layers can be a flat colour or a tiled image, mixed freely — texture over
colour, colour over texture, or two textures.

```jsonc
"themes": [{
  "id": "cracked_vault",
  "name": "Cracked Vault",
  "floor": "#d9d6bd",          // layer A colour (if no texture A)
  "floorB": "#8d8a72",         // layer B colour (if no texture B)
  "floorTexture":  "data:image/png;base64,…",   // optional layer A image
  "floorTextureB": "data:image/png;base64,…",   // optional layer B image
  "mask": "cracks",            // built-in id, or a data URI of your own mask
  "wall": "#5f6f52", "wallShade": "#1e2a1c", "outline": "#0a0f0a"
}]
```

Masks tile at the same scale as the textures, and the recolour still runs off the
shipped geometry — so no mask can move a doorway or break tile connections.


---

# Progression, survivability and helping each other

## Characters level up

XP comes from kills (scaled to the enemy — bosses are worth 4x), placing tiles,
surviving traps, reviving, and helping allies. Levels arrive at 20 / 55 / 105 / 170
XP and so on.

Each level gives **+HP (and heals you by that amount**, so levelling is a real second
wind), **+1 to-hit every 2 levels**, **+1 defence every 3 levels**, and **+1 action at
level 5**. The character sheet shows `Lv N`, a gold XP bar, and live ⚔/🛡 values.

> Note on wording: the dungeon floor and character advancement are separate things.
> The top bar now says **Floor**, and characters have **Lv** — in the data they're
> `player.level` (floor) and `player.charLevel`.

## Dying less

Base HP went up a lot: Warrior 22→30, Archer 15→22, Thief 13→20, with the Archer
and Thief also getting a defence point. Two new ways to stay alive:

- **🩹 Rest** — one action, heals `1d4 + your level`. Not available mid-fight. This
  is the big one: you're no longer dependent on finding potions.
- **Healing draughts** are now twice as common and carry 2 uses.

Enemies were then rescaled to keep fights meaningful. This was **tuned by
simulation, not guesswork** — 400 duels per configuration:

| enemy HP | Warrior | Archer | Thief |
|---|---|---|---|
| ×1.0 (untuned) | 98% win, 82% HP left | 83%, 71% | 89%, 73% |
| **×1.6 (chosen)** | **93%, 74%** | **75%, 55%** | **79%, 61%** |
| ×2.4 | 81%, 61% | 40%, 37% | 50%, 48% |

`ENEMY_HP_SCALE` is one constant if you want it harder or softer. Enemies also gain
+2 HP per floor, but character growth outpaces it — an Archer at Lv4 on floor 3 wins
85% where a Lv1 wins 75%.

## Trading and healing each other

Stand on the same tile and a **🤝 Share** button appears:

- **Give** any item — it transfers to their pack.
- **Use on them** — spend a potion, elixir or buff item on an ally instead of
  yourself. Only offered for supportive items; a Scroll of Fireball has no
  "use on ally" button.
- **✚ Bind their wounds** — heals `1d4 + your level` with no item at all.

Each costs one action (a cooldown in realtime), works in both turn-based and
realtime, and is host-validated the same way movement is. Helping earns XP, so
playing medic isn't a waste of a turn.
