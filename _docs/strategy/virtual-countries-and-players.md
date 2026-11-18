# Virtual Countries and Player Slots

**Parent:** [../roadmap.md](../roadmap.md)  
**Related:** [ai-architecture.md](ai-architecture.md), [unit-metadata-and-cunit.md](unit-metadata-and-cunit.md)

## Core principle

**StarCraft player numbers are not authoritative identities for countries, armies, or wars.**

Player slots are temporary **execution resources**. Permanent simulation identity lives in epScript arrays.

epScript player constants are **0-based** (`P1 = 0` … `P12 = 11`) per [`_Reference/Constants Reference/TrgPlayer.md`](../../_Reference/Constants%20Reference/TrgPlayer.md).

---

## How the map uses slots today

| Slot | Current role (approx.) |
|------|-------------------------|
| P1–P5 | Human players (economy loops iterate 0–4) |
| P6–P7 | Available / lightly used |
| P8 | Pre-nation buildings, test armies, some FX owners |
| P9 | Staging owner during building refresh |
| P10–P11 | Unused capacity |
| P12 | Neutral — diplo props, sea-lane holders, not a normal AI actor |

Extended players **P9–P11** (and Neutral P12 where used) can own units and be manipulated via EUD/CUnit APIs. **Confirmed:** vision is byte-masked to P1–P8 only — extended players cannot receive vision for normal auto-acquire combat. **Also confirmed:** they can still damage P1–P8 units when given direct CUnit attack orders at combat range. Treat them as **order-driven ownership namespaces**, not autonomous vision-capable agents.

Alliance status is believed **not** limited the same way (P12 can be friend or foe); still verify damage under `SetAllianceStatus` when wiring fronts. See [priority-experiments.md](priority-experiments.md).

---

## Logical tables

### Countries

Permanent IDs. Suggested fields:

| Field | Purpose |
|-------|---------|
| `country_id` | Stable identity |
| `controller_kind` | Human / AI / unclaimed |
| `human_slot` | If human-controlled, which P1–P5 (optional) |
| `capital_province` | Capital duchy index |
| `military_strength` | Aggregate or cached strength |
| `active_war_count` | Quick filter |
| `flags` | e.g. destroyed, revolting |

A human “playing England” means their **country_id** is England. Units and duchies point at that country, not merely at `P1`.

### Provinces / duchies

Evolve beyond today’s SC-centric `duchyOwner` in [`constants.eps`](../../constants.eps):

| Field | Purpose |
|-------|---------|
| `province_id` / duchy index | Stable |
| `owner_country_id` | Legal owner |
| `occupier_country_id` | If occupied |
| `occupation_state` / timer | Siege/occupation progress |
| `strategic_value` | AI scoring |
| adjacency | From `adjecency.graphml` (export into arrays later) |

**Do not** infer province ownership solely from the StarCraft owner of units standing there.

Building `unit.owner` may still be set for UI color / engine convenience, but simulation truth is `owner_country_id`.

### Wars

Wars are entities, not a single pairwise hostility flag:

| Field | Purpose |
|-------|---------|
| `war_id` | Stable |
| `attacker_country_id` | Primary attacker |
| `defender_country_id` | Primary defender |
| `participants` bitmask / list | Additional joiners |
| `war_state` | Active / ending / white peace / conquest |
| `fronts` | Links to army assignments |

Multiple wars can run at once. See [wars-diplomacy-occupation.md](wars-diplomacy-occupation.md).

### Armies

| Field | Purpose |
|-------|---------|
| `army_id` | Stable |
| `country_id` | Owning country |
| `war_id` | Optional active war |
| `objective_province` | Current target |
| `execution_slot` | SC player currently owning the units |
| `allocation_generation` | Invalidate stale ops |
| `mission` / `army_state` | Idle, marching, fighting, occupying, … |
| strength caches | Unit counts / rough power |

### Player allocation table

| Field | Purpose |
|-------|---------|
| `player_slot` | P0–P11 index |
| `allocated` | Free / busy |
| `country_id` / `army_id` / `war_id` | Current binding |
| `allocation_generation` | Bump on every reassignment |

**Allocate slots to armies/fronts, not permanently to countries.** A country with two distant wars may need two slots, or may time-slice one slot if those armies need not fight each other simultaneously.

---

## Load-bearing constraint

Units that must **fight each other with native combat** cannot share the same StarCraft owner.

Therefore hostile fronts require distinct `execution_slot` values (among slots that experiments prove can deal damage). Friendly allied armies *may* share a slot if that simplifies bookkeeping — product choice after Phase 1.

---

## Dynamic allocation sketch

1. Strategic layer opens a war / assigns an objective.
2. Military layer requests an execution slot from the free pool (prefer conventional computer-capable slots for any actor that needs auto-acquire).
3. `GiveUnits` / `cgive` moves the army’s units onto that slot; alliances updated for the slot vs enemies.
4. Orders issued (`Order` Attack/Move/Patrol toward objective locations).
5. On war end or army idle, units return to a home slot or stay parked; allocation released; generation bumped.

Never assign ownership by writing only `CUnit.owner` (`0x04C`) — see [unit-metadata-and-cunit.md](unit-metadata-and-cunit.md).

---

## Slot budget (working hypothesis)

Subject to Phase 1 results:

- **P1–P5:** reserved for humans’ primary control / UI ownership of demesne where needed
- **P6–P8:** primary pool for AI army execution and “neutral hostile” CPU presence
- **P9–P11:** overflow ownership namespaces if combat experiments pass; otherwise logistics-only / non-combat holders
- **P12:** Neutral props only

Exact budget is an experiment outcome, not a moral law.

---

## Migration from current `duchyOwner`

Today `duchyOwner[i]` stores a StarCraft player (or `27` for empty) to support liberate-vs-occupy in [`duchies.eps`](../../duchies.eps).

Target:

1. Add `owner_country_id` / `occupier_country_id`.
2. Keep building player ownership synchronized for color and engine Bring/Give conditions during transition.
3. Teach siege conclude + levy spawn to read country IDs first.
4. Retire SC-player-as-country assumptions once armies carry metadata.
