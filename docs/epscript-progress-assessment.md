# epScript Progress Assessment

**Date:** 2026-11-18  
**Scope:** Compare the current epScript codebase against pre-epScript design documentation.

**Assumption:** The epScript systems described below have been playtested and work well in-game.

**Cleanup (same date):** Removed `_Triggers/` and `_Deprecated/`. Slimmed former `_Documentation/` into `_docs/` (images + `.txt` only); deleted TrigEdit tooling (Excel death-count tables, Notepad++ UDL, empty stubs).

**Forward plan:** See living [roadmap.md](roadmap.md) and strategic docs under [strategy/](strategy/) (AI architecture, virtual countries, experiments). Prefer those over historical `*Plan*.txt` files.

---

## Summary

The epScript rewrite has successfully rebuilt the **core playable loop** of the old TrigEdit map — duchy state, levies, sieges, income/debt/manpower — with far less code and a cleaner architecture. Most of the broader CK-like ambition in the old design docs (warscore, diplomacy, crusades, vassals, full map content) is still unimplemented.

Rough progress:

| Lens | Estimate |
|------|----------|
| Core Britain vertical slice (own → upgrade → levy → siege → occupy → economy) | ~65–75% |
| Full historical design vision from old docs | ~20–30% |
| Architectural replacement of the old mega-trigger systems for ported features | Effectively complete |

**Bottom line:** The risky technical migration is largely done. Remaining work is product/design direction and content, not proving that epScript can carry these systems.

---

## Codebase inventory

Loaded by `CK2.edd` (underscore-prefixed dirs are ignored by euddraft):

| Module | Lines (approx.) | Role |
|--------|-----------------|------|
| `constants.eps` | ~317 | Duchy IDs, arrays, supply helpers, init data, string IDs |
| `duchies.eps` | ~644 | Levies, upgrades via build queue, sieges, UI string hooks |
| `economy.eps` | ~141 | Income/expense/debt cycles, manpower regen |
| `initialization.eps` | ~169 | Nation giveaways (England/Scotland), building refresh, test setup |
| `military.eps` | ~16 | Empty shell |
| `diplomacy.eps` | ~19 | Empty shell |
| `events.eps` | ~24 | Empty shell |
| `leftovers.eps` | ~21 | `buildReset` helper (not in `CK2.edd` load list) |

**Total live design logic:** roughly **1,300 lines**, concentrated in four modules. Empty shells are intentional homes for future systems.

---

## What epScript has reimplemented

These map directly to systems described in `README.md` and the old plan checklists, but with a different (better) encoding:

### Duchy state model

- Per-duchy `EUDArray`s for income, militia hall, barracks, stables, fortifications, occupation, levy flags, siege flag.
- Placeholder arrays already exist for loyalty, rebellion, and demesne (unused).
- Unit-pointer ↔ duchy-index registration replaces fragile Unit Index / HP-decimal schemes.
- Capacity stubbed to `MAX_DUCHIES = 154`; only Britain + Normandy (~15) are initialized with data.

### Military / holdings loop (`duchies.eps`)

- **Local / global levy raise**, demobilize, per-duchy enable/disable.
- Manpower cost checks against the supply-hijacked manpower pool.
- Spawn composition from militia/barracks levels (stables level stored but not yet used in spawn).
- **Mass-to-duchy** rally target via build-queue command.
- **Sieges:** ≥8 enemy men, defender outnumbered → siege gear timer → conclude with occupy or liberate; retreat cancels siege.
- Upgrade commands via Cybernetics Core build queue (income, fortifications, militia, barracks, stables).

### Economy (`economy.eps`)

- Timed income/expense netting.
- Debt accrual when minerals cannot cover deficit; debt repayment before surplus minerals.
- Manpower regeneration on a separate timer.

### Initialization (`initialization.eps`)

- England / Scotland territory grant + manpower setup (auto-selected for testing).
- Preplaced building refresh / doodad enable.
- Sea-lane placeholder swap.

### UX / presentation

- Partial dynamic duchy name strings and TBL overlays for manpower / income / expenses / loyalty label.

---

## Architectural wins vs the TrigEdit era

Old approach (documented in README and legacy plans):

- HP decimal places as upgrade encoding.
- Death-counter arithmetic for income and many per-duchy trigger farms.
- Mobilization staged through many sequential triggers and locations.
- Tens of thousands of lines of trigger text for systems now measured in hundreds of lines of epScript.

New approach:

- **Explicit arrays** for game state.
- **Build-queue “commands”** on duchy buildings as the player UI verb set.
- **General loops** over registered duchies for sieges and global levies.
- Content expansion is mostly data + map locations, not copy-pasted trigger graphs.

This is why the rewrite is not merely smaller — it is a different (and more maintainable) design.

---

## Gaps vs historical design documents

Sources now under `_docs/` (historical; not binding roadmap):

- `README.md` — premise, planned features, system descriptions
- `_docs/Crusader Kings plan.txt` — prioritized backlog + geography lists
- `_docs/CK2 Plan.txt` — older implementation checklist / design notes
- `_docs/CK issues.txt`, `_docs/1066 Missions.txt` — bugs / incomplete mission sketches
- `_docs/duchycoordinates.txt` — siege location bounds still mirrored in `duchies.eps`

### Still missing (from old “planned features” / high-priority lists)

- Warscore and war resolution (white peace / conquest / demand buttons)
- Town and castle sieges (only duchy-level sieges exist)
- Diplomacy (war declaration UI, alliances, marriages, pacts)
- Crusades / jihads and religious-war bonuses
- Mercenaries
- Debt combat penalties, desertions, attrition
- Vassal opinion, crown laws, revolts (arrays reserved, logic absent)
- Multi-start-date / large nation select UI
- Major siege random events
- Full European duchy footprint from old geography lists

### Partial / wiring gaps inside “done” systems

- `duchyIncome[]` upgrades increment the array, but payout currently reads player supply income — not clearly aggregating per-duchy income.
- Fortification level does not clearly drive siege timer/HP yet (`startSiege` uses a fixed low HP%).
- Stables upgrades do not yet affect levy composition.
- Nation selection is test-hardcoded, not a real menu.
- `military.eps` / `diplomacy.eps` / `events.eps` are placeholders; war/siege logic currently lives under `duchies.eps`.

### Note on old goals

Many unchecked items in `_docs/Crusader Kings plan.txt` and `_docs/CK2 Plan.txt` reflect **2010s TrigEdit constraints and CK2-port priorities**. They are useful as inspiration, not as a binding roadmap. Direction may change freely; this assessment treats them as historical intent, not current commitments.

---

## Progress opinion

You are past the hard part of the migration:

1. **Proven:** Core map spine works under epScript with dramatically less complexity.
2. **Not proven / not built:** The broader grand-strategy layer (wars as a system, diplomacy, religion, vassals, content scale).
3. **Next useful work** follows [roadmap.md](roadmap.md): stabilize core wiring (Phase 0), run player/combat experiments (Phase 1), then introduce virtual country/war/army tables and a strategic director in `military.eps` / `diplomacy.eps` — not another rewrite of duchy/economy fundamentals.

---

## Legacy cleanup status

| Path | Status |
|------|--------|
| `_Triggers/` | **Removed** — classic TrigEdit library (~14k lines). Recover via git history if needed. |
| `_Deprecated/` | **Removed** — older TrigEdit archive / tooling. Recover via git history if needed. |
| `_Documentation/` | **Removed** — useful images + `.txt` moved into `_docs/`; Excel death-count tables, `SC UMS.xml`, empty `build_queue_table.md` deleted. |
| `_docs/` | **Current home** for assessments, retained historical `.txt`, coordinates, and reference art |
| `_Reference/` | **Kept** — epScript language/API reference |
| `README.md` | Update eventually — still good *fantasy* overview; system-encoding sections describe the old world |

### `_docs/` contents after cleanup

- `epscript-progress-assessment.md` (this file)
- `roadmap.md` — living development roadmap
- `strategy/` — AI / virtual-country architecture, experiments, archived handoff
- Historical text: `Crusader Kings plan.txt`, `CK2 Plan.txt`, `CK issues.txt`, `1066 Missions.txt`, `Replace Table.txt`, `duchycoordinates.txt`
- Reference art: `CK.bmp`, `CK.jpg`, `Europe 1066.jpg`, `CK3 1066 Start Updated.png`, `medieval-map-of-europe-in-1444.jpg`, `6ju1q1aj6ee11.png`

`duchycoordinates.txt` remains the most operationally relevant retained file (siege L/T/R/B for duchies 0–14, mirrored in code). `Replace Table.txt` is TrigEdit-era naming archaeology only.

---

## Related paths

```
README.md
_docs/roadmap.md
_docs/strategy/
_docs/epscript-progress-assessment.md
constants.eps
duchies.eps
economy.eps
initialization.eps
military.eps
diplomacy.eps
events.eps
CK2.edd
_Reference/
```
