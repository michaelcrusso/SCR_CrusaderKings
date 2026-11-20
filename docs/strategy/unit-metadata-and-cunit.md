# Unit Metadata and CUnit Rules

**Parent:** [../roadmap.md](../roadmap.md)  
**Related:** [virtual-countries-and-players.md](virtual-countries-and-players.md), [priority-experiments.md](priority-experiments.md)

## Goal

Every military unit needs a **virtual identity** (`country_id`, `army_id`, …) independent of its StarCraft `owner`.

Engine ownership answers “who does the exe think controls this for combat/vision?”  
Metadata answers “which country/army is this for the simulation?”

---

## Prefer a sidecar table

Conceptually:

```text
UnitMetadata[1700]  // indexed by CUnit slot / parallel index scheme
  country_id
  army_id
  unit_role
  flags
  generation   // or other identity stamp
```

**Why sidecar first**

- Avoids repurposing engine fields that look “free.”
- Survives clearer reasoning when slots are reused.
- Matches epScript strengths (`EUDArray`, structured loops).

CUnit layout citation: [`_Reference/Built-in-Object-Types-(Extended).md`](../../_Reference/Built-in-Object-Types-(Extended).md) (`CUnit` members including `owner` at `0x04C`, player-unit links, `killCount` at `0x08F`, `unknown0x8C`, etc.).

### Field caution list

| Offset | Member (epScript) | Rule |
|--------|-------------------|------|
| `0x04C` | `owner` | Never repurpose; change only via safe give APIs |
| `0x068` / `0x06C` | `prevPlayerUnit` / `nextPlayerUnit` | Engine lists — broken by naive owner writes |
| `0x08C` | `unknown0x8C` | Documented as maybe leftover RGB; **do not rely on** without experiments |
| `0x08E` | `rankIncrease` | Engine-used |
| `0x08F` | `killCount` | Engine-used; currently overloaded for siege tagging — migrate off |
| `0x092` | `AIActionFlag` | Internally used |
| uniqueness / related | engine-managed | Investigate for slot-reuse validation (handoff mentions `0x0A5`; verify against Remastered + our wrappers before depending on it) |

---

## Slot reuse

CUnit slots are recycled when units die/despawn.

Required discipline:

1. Clear metadata when a unit is removed or confirmed dead.
2. Keep an **active unit set** (`UnitGroup` or parallel list) instead of trusting a full 1700 scan every frame for strategy ticks.
3. Before trusting metadata, verify the slot still holds the expected unit type / identity stamp.
4. Bump army `allocation_generation` when mass-reattaching units so delayed logic cannot apply stale orders.

[`UnitGroup`](../../_Reference/Built-in-Functions.md) is already used for siege Starports in [`duchies.eps`](../../duchies.eps) — reuse that pattern for armies.

---

## Ownership changes — safe paths only

### Preferred

- Classic action: `GiveUnits(...)` ([`_Reference/Built-in-Functions.md`](../../_Reference/Built-in-Functions.md))
- CUnit helpers: `cgive` / give, with the documented pattern of **queue units first** if iterating `EUDLoopPlayerCUnit` (giving mid-iteration corrupts the walk)

Current siege conclusion already prefers `GiveUnits` so player color and building fire offsets update correctly.

### Avoid

- Assigning only `unit.owner = newPlayer` / poking `0x04C` without list maintenance

Effects of ownership change on vision, supply display, and AI recognition remain **Phase 1 experiment** topics.

---

## Current technical debt: siege `killCount`

Siege Starports store the duchy index in `killCount` (`0x08F`) and read it on death in [`duchies.eps`](../../duchies.eps). Cancelled sieges write `255`.

This works for a small Britain slice. It is a poor long-term identity channel:

- `killCount` is a real engine field
- Byte-sized (duchy index ceiling concerns as content grows)
- Easy to confuse with actual kills if ever mixed with combat units

**Migration plan**

1. Introduce sidecar (or a dedicated siege-record array keyed by starport ptr/epd).
2. Point `concludeSiege` at that record.
3. Stop writing tactical identity into `killCount` for new systems; eventually stop for sieges too.

---

## Orders and combat

Strategic/military layers should issue ordinary orders where possible:

- `Order(unitType, player, fromLoc, Attack|Move|Patrol, toLoc)` ([Built-in-Functions](../../_Reference/Built-in-Functions.md))

Custom logic intervenes when:

- An army receives a new objective
- Occupation/siege thresholds need checking (already partly in `duchies.eps`)
- Allocation / alliance updates are required before combat can occur

Do not micro every frame if periodic re-orders suffice.

---

## Metadata lifecycle (checklist)

| Event | Action |
|-------|--------|
| Levy spawn | Set country/army metadata; set execution owner via Create/Give |
| Unit death / remove | Clear sidecar; update army counts |
| Slot reallocation | GiveUnits to new slot; bump generation; rewrite alliances |
| Siege gear create | Record duchy/war in sidecar (not killCount long-term) |
| Army disband | Clear metadata for all members; free allocation |

---

## Related docs

- [wars-diplomacy-occupation.md](wars-diplomacy-occupation.md)
- [priority-experiments.md](priority-experiments.md)
