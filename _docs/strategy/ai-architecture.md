# AI Architecture

**Parent:** [../roadmap.md](../roadmap.md)  
**Source proposals:** [handoff-virtual-countries-ai.md](handoff-virtual-countries-ai.md)

## Decision

**Use Approach B: an epScript strategic director.** StarCraft remains responsible for movement, pathfinding, weapon targeting, and combat resolution.

Do **not** make native ASC3 / `aiscript.bin` the authoritative strategic system for this map.

Optional later: limited native AI as a tactical helper on a small number of dedicated computer player slots — never as the country/war brain.

---

## Approach A — Native AI scripts

### What it offers

ASC3 can manage production goals, attack forces, towns/regions, and some combat evaluation. epScript can invoke scripts via `RunAIScript` / `RunAIScriptAt` ([`_Reference/Built-in-Functions.md`](../../_Reference/Built-in-Functions.md); script IDs in [`TrgAIScript.md`](../../_Reference/Constants%20Reference/TrgAIScript.md)).

### Why it fails as the primary brain here

- **One strategic AI per computer player** — one diplomatic identity, one vision state, one unit pool.
- Cannot represent **many hostile countries** that only temporarily share execution resources.
- UMS/campaign AI does not understand duchy/country/war abstractions.
- Attack-force commands want **specific unit types**, not “Men” / levy mixtures.
- Campaign targeting is region-driven and semi-randomized; it is not a clean “attack this province location” API.

### External research (not in `_Reference`)

Community work (BWMetaAI, SCR EUD emulator notes) suggests custom `aiscript.bin` buffers can be injected. That may be useful lab work later. It is **not documented in this repo’s `_Reference`** and must not gate the roadmap.

### Current map usage

[`initialization.eps`](../../initialization.eps) only runs shared-vision scripts (`+Vi5` / `+Vi6` / `+Vi7`) for humans — not strategic AI.

---

## Approach B — Trigger / epScript strategic director

### What the custom system owns

- Countries and capitals
- Diplomacy and wars
- Armies, fronts, objectives
- Province ownership and occupation policy
- Dynamic allocation of StarCraft player slots to executing armies

### What StarCraft still owns

- Pathfinding and collision
- Auto-acquire / weapon fire (when ownership + alliances + vision allow — **vision only for P1–P8**; extended owners need explicit CUnit attack orders)
- Combat damage and deaths
- Order execution after we issue `Order` / related actions

### Why this fits the map

The epScript rewrite already behaves like a proto–Approach B stack: explicit `EUDArray` duchy state, build-queue player verbs, siege loops, supply-hijacked economy. Empty [`military.eps`](../../military.eps), [`diplomacy.eps`](../../diplomacy.eps), and [`events.eps`](../../events.eps) are the intended homes for the director.

---

## Three-layer model

```mermaid
flowchart LR
  strategic[Strategic_simulation]
  military[Military_simulation]
  execution[StarCraft_execution]
  strategic --> military --> execution
```

| Layer | Responsibilities | Likely modules |
|-------|------------------|----------------|
| Strategic | Countries, diplomacy, wars, province ownership policy, objective scoring | `diplomacy.eps`, parts of `events.eps` |
| Military | Armies, unit metadata, assignments, occupation/siege orchestration | `military.eps`, existing `duchies.eps` siege/levy |
| Execution | Player slots, `GiveUnits` / `cgive`, `Order`, alliances, CUnit loops | thin helpers; called from military |

**Rule:** the strategic layer must not require knowing which StarCraft player currently owns a unit. It speaks in `country_id` / `army_id` / `province_id`. The military layer binds armies to execution slots. The execution layer performs engine actions.

---

## Hybrid allowance

Native AI may later:

- Drive **local tactical** behavior for a dedicated computer slot that exclusively represents one army or one “bandit” actor
- Use town/region tricks (`value_area`, invincible anchor buildings) as experiments

Native AI must **not**:

- Author country diplomacy or war membership
- Be the only way units receive strategic destinations
- Force production-centric economy assumptions onto levy gameplay

Virtualized native production / `wait_force` recognition is a distraction while Approach B is primary. See [priority-experiments.md](priority-experiments.md) for optional low-priority tests.

---

## Related docs

- [virtual-countries-and-players.md](virtual-countries-and-players.md) — identities vs slots
- [unit-metadata-and-cunit.md](unit-metadata-and-cunit.md) — unit bookkeeping
- [wars-diplomacy-occupation.md](wars-diplomacy-occupation.md) — wars and sieges
