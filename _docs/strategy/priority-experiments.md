# Priority Experiments

**Parent:** [../roadmap.md](../roadmap.md)  
**Related:** [ai-architecture.md](ai-architecture.md), [virtual-countries-and-players.md](virtual-countries-and-players.md), [unit-metadata-and-cunit.md](unit-metadata-and-cunit.md)

## Purpose

Before locking Phase 2 allocation design, answer what StarCraft combat and ownership will actually do. Prefer epScript APIs documented in `_Reference`. Label raw-address work as optional fallback.

Record pass/fail in this file or a short sibling log when results exist.

---

## Ground rules

- Use `SetAllianceStatus` + `setcurpl` first ([Built-in-Functions](../../_Reference/Built-in-Functions.md)); raw `0x58D634` only if the action API cannot reach extended players.
- epScript players are **0-based** (`P1 = 0`).
- Built-in shared-vision AIScripts `+Vi0`…`+Vi7` / `-Vi0`…`-Vi7` cover **P1–P8 only** ([TrgAIScript](../../_Reference/Constants%20Reference/TrgAIScript.md)).
- Transfer units with `GiveUnits` or queued `cgive`, not raw `owner` writes.
- Judge success by **actual HP damage / kills**, not only alliance UI text.

---

## Tier 1 — Must run (blocks architecture)

### E1 — Conventional mutual enmity (control)

**Setup:** P1 and P8 each own melee units in the same location.  
**Actions:** Reciprocal `SetAllianceStatus(..., Enemy)`.  
**Pass:** Both sides auto-acquire and deal damage.  
**Unlocks:** Baseline confidence in alliance → combat pipeline (already partially used in init).

### E2 — Human vs extended owner (P9)

**Setup:** P1 units vs units `GiveUnits`’d or created for P9.  
**Actions:** Reciprocal Enemy between P1 and P9 via `SetAllianceStatus` if accepted; else document failure and try masked `SetMemory` on alliance table (**external address knowledge**).  
**Pass A:** P1 auto-attacks P9 and damages.  
**Pass B:** P9 auto-attacks P1 and damages.  
**Fail implications:** Extended slots may still hold units but need explicit `Order` micro, or cannot be combat executors.

### E3 — Extended vision

**Setup:** Same as E2.  
**Actions:** Try shared-vision AIScripts (expect P1–P8 only); optionally probe shared-vision table `0x57F1EC` (**external**).  
**Pass:** P9 units acquire targets without per-frame `Order`.  
**Fail:** Combat for extended owners is order-driven only — still usable, higher AI cost.

### E4 — Explicit orders on extended units

**Setup:** P9 owns units; may lack vision.  
**Actions:** `Order(..., Attack, targetLoc)` (and/or CUnit order helpers if used).  
**Pass:** Units path and deal damage on contact.  
**Unlocks:** Extended slots as execution namespaces even without auto-acquire.

### E5 — Safe reallocation between conventional slots

**Setup:** Army on P7; transfer to P8 mid-game with `GiveUnits` / queued `cgive`.  
**Pass:** Units survive, lists iterate correctly (`EUDLoopPlayerCUnit`), combat still works with updated alliances.  
**Fail:** Blocks dynamic fronts; would force static slot-per-country designs.

### E6 — Same-owner “virtual enemies”

**Setup:** Two groups tagged with different metadata `country_id`, **same** `owner` player, ordered to fight.  
**Pass (expected: FAIL for native combat):** No friendly fire / no mutual auto-attack.  
**Purpose:** Confirm allocation is mandatory for hostile fronts. A “pass” where they somehow fight would be surprising and would relax slot pressure.

---

## Tier 2 — Strongly recommended before Phase 2 polish

### E7 — Sidecar metadata through death and slot reuse

**Setup:** Spawn units; write sidecar; kill/remove; respawn into recycled slots.  
**Pass:** No stale country/army IDs applied to new units; generation checks work.  
**Unlocks:** Confidence to migrate off siege `killCount` tagging.

### E8 — Alliance projection under two wars

**Setup:** Slot A fights Slot B in war 1; Slot A also co-belligerent with Slot C. End war 1; ensure A↔B enmity clears without breaking A↔C.  
**Pass:** Alliance projection recomputes cleanly from war table.  
**Unlocks:** Multi-war diplomacy without sticky foes.

### E9 — Human demesne owner vs army execution slot

**Setup:** Human’s duchy buildings stay on human slot colors; field army temporarily on P7.  
**Pass:** Siege/occupation and levy rules still resolve using country metadata.  
**Unlocks:** Splitting UI ownership from army execution.

---

## Tier 3 — Optional / low priority (Approach A leftovers)

### E10 — Native AI recognizes trigger-created units

Only if experimenting with hybrid tactical AI on a dedicated computer slot.  
**Pass:** `wait_force`-style behavior counts trigger spawns.  
**Note:** Does not change Approach B preference.

### E11 — Invincible building + `value_area` / areatown anchors

Regional defensive bias for a dedicated computer player.  
**Pass:** Persistent defense of a province-sized region.  
**Note:** Convenience for hybrid setups; not required for Order-driven armies.

### E12 — aiscript.bin buffer injection

External research path (BWMetaAI / SCR emulator notes). **Not in `_Reference`.**  
Treat as curiosity unless a concrete tactical payoff appears after Approach B works.

---

## Suggested epScript toolkit for harnesses

| Need | API |
|------|-----|
| Alliances | `setcurpl` + `SetAllianceStatus` |
| Spawns | `CreateUnit` / `CreateUnitWithProperties` |
| Transfer | `GiveUnits`, `cunit.cgive` + queue |
| Orders | `Order` |
| Iteration | `EUDLoopCUnit`, `EUDLoopPlayerCUnit`, `EUDLoopNewCUnit` |
| Vision scripts | `RunAIScript("+ViN")` / `"-ViN"` for N=0..7 |
| Debug | `simpleprint`, HP checks via CUnit members |

Keep experiment harnesses out of production modules when possible (temporary hooks in `initialization.eps` or a dedicated `#if`-style section you remove later).

---

## Results log

| ID | Date | Result | Notes |
|----|------|--------|-------|
| E1 | | | Control case; expected pass from existing P1↔P8 enmity usage |
| E2 | 2026-11 (Michael) | **Partial** | Extended owners can fight P1–P8 when given **direct CUnit attack orders** at combat range. Auto-acquire for extended players blocked by vision limits (see E3). Alliance vs extended still assumed workable (P12 enmity/friendship precedent); confirm with damage tests if needed. |
| E3 | 2026-11 (Michael) | **FAIL** | Player vision is **byte-masked to P1–P8**; bits beyond Player 8 are ignored. Extended players **cannot** be given vision for normal auto-combat. Do not plan on `+ViN` / shared-vision table extending past P8. |
| E4 | 2026-11 (Michael) | **PASS** | Units owned by extended players (reported for P9–P12) can be ordered via CUnit commands to attack P1–P8 units within combat distance. Sufficient for order-driven combat executors. |
| E5 | | | |
| E6 | | | Still expected FAIL for native mutual combat; confirm when convenient |
| E7 | | | |
| E8 | | | |
| E9 | | | |
| E10 | | | |
| E11 | | | |
| E12 | | | |

### Architecture implication (from E2–E4)

Extended slots are viable as **order-driven combat namespaces**, not as vision-capable auto-acquire agents. Strategic/military AI that uses P9–P11 (or Neutral where applicable) must issue explicit attack/move orders and cannot rely on engine acquisition. Prefer conventional P1–P8 slots when auto-acquire and lower micro cost matter; use extended slots for overflow fronts that the director is willing to micromanage.
