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
