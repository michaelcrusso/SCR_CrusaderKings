# Engine Delegation and Attrition / Supply

**Parent:** [../roadmap.md](../roadmap.md)  
**Related:** [unit-metadata-and-cunit.md](unit-metadata-and-cunit.md), [wars-diplomacy-occupation.md](wars-diplomacy-occupation.md)

## Design principle

**Prefer handing arithmetic and continuous background work to the StarCraft engine.** epScript/triggers should **arm**, **disarm**, and **evaluate** game state — not re-implement clocks, regen, and drains in software every frame when a native field already ticks.

| Prefer | Avoid when possible |
|--------|---------------------|
| Engine timers / HP / orders / pathfinding | Virtual per-frame counters that duplicate the engine |
| Set a CUnit field once; let the game decrement | Full-map `EUDLoopCUnit` every frame to subtract `1` from a sidecar |
| Boundary + threshold interventions | Constant ±1 offset fights against native regen |
| Scoped `UnitGroup` of active levies | Scanning all ~1700 slots for every system every tick |

Strategic identity (country, war, army) still lives in epScript arrays. **Physics and countdown** should ride the engine where SCR allows.

Label offset-level claims as **EUD Book / community** when not named in `_Reference`.

---

## Rejected / weak supply proxies

| Idea | Why not (as primary) |
|------|----------------------|
| Cloak energy drain while “not invisible” | Drain is gated on cloak status; visibility hacks are fragile and hurt combat |
| Other continuous “drain spells” | No useful vanilla channeling mana drain besides cloak |
| Own spellcaster energy every frame | Works, but fights regen and needs energy-capable unit types; heavier than necessary |
| Broodling “energy” as spellcaster mana | It is **not** mana — see below |

---

## Preferred direction: `removeTimer` as supply fuse

### What Broodling “energy” really is

Broodlings (and hallucinations / similar temporary units) show a depleting bar that tracks **`removeTimer`**, not spellcaster energy. EMP/Feedback/`ModifyUnitEnergy` do not refill it. When the timer reaches **0**, the engine **removes** the unit regardless of HP.

Community / EUD Book:

- Field is a **u16** at CUnit offset **0x110** (book family around `0x59CDB8` for the table base + offset documentation).
- Not exposed as a named member in our `_Reference` CUnit excerpt — poke via EPD/offset; validate on Remastered.
- **Not** a `UnitPrototypeFlags` bit you enable globally. Behavior is “timer set → engine counts down → remove at 0.”
- UMS practice: the same timer can be applied to **non-Broodling** units for timed life (e.g. after create, write a frame count).

### Operating model (territory food / attrition)

| State | Action |
|-------|--------|
| **Friendly territory** | Set `removeTimer = 0` → no countdown (safe / supplied). Do **not** continuously `+=` every frame. |
| **Enter enemy / hostile occupation** | Arm timer to “full supply” in **frames** (u16, max 65535). |
| **While in enemy territory** | **Do nothing to the timer** — engine decrements each logic frame. |
| **Timer ≤ threshold** (e.g. 5–10, not only `== 1`) | Preempt removal: subtract HP (or apply starve damage), set timer back to a small refresh value; repeat until HP policy says allow death. |
| **Return to friendly** | Set `removeTimer = 0` again. |
| **Allow death** | Stop refreshing and let the timer hit 0, or kill explicitly. |

### Why a threshold, not exactly `1`

Frame ordering between epScript (`beforeTriggerExec` / `afterTriggerExec`) and the engine’s decrement/remove is easy to lose. Catching a **band** of low values is more reliable than a single-frame `== 1` window. **Prototype** which lifecycle hook wins on SCR before locking the threshold.

### UI caveat

Broodlings/hallucinations present `removeTimer` as an energy-like bar. Other levy types may get timed life **without** that bar unless unit choice or further UI work provides one. **Function first; visible bar second.**

### Cost profile

- Enemy drain: **engine** (free for us).
- Scripts: territory **transitions** + periodic **low-timer** scans on an **active levy group** — not full-map energy writes every frame.

---

## General engine-delegation checklist

When designing a new system, ask:

1. Does the engine already tick something we can arm/disarm (timer, order, HP burn, cooldown)?
2. Can triggers only run on **edges** (enter province, siege start, war declare) and **thresholds**?
3. If we must loop, can it be a maintained `UnitGroup` / army list at interval `N` frames?
4. Are we about to virtualize a clock that `removeTimer`, countdown timer, or order timers already provide?

If (1) is yes, prefer that path and document offsets / experiments under [priority-experiments.md](priority-experiments.md).

---

## Experiments to log

| ID | Question |
|----|----------|
| A1 | Writing `removeTimer` on current levy types (Zergling/Zealot/Firebat/Ghost) causes countdown + removal at 0? |
| A2 | `removeTimer = 0` fully disables removal for that unit? |
| A3 | Catching `<= N` in `beforeTriggerExec` vs `afterTriggerExec` reliably preempts death? |
| A4 | Does any levy type show a visible bar for this timer without DAT/UI work? |
| A5 | Territory edge detection cheap enough when combined with existing siege locations? |

Add pass/fail rows to the experiments log when tested.

---

## Relation to older random-death siege ideas

Random per-tick kill chance in enemy land is simpler but less transparent. `removeTimer` gives a **deterministic, engine-driven** supply fuse with optional HP starve phase — preferred if A1–A3 pass. Random attrition remains a fallback if timer preempt proves unreliable.
