# Technical Hand-off: Virtual Countries and AI Architecture in StarCraft: Brood War / Remastered

> **Archive note:** Source handoff from a prior agent research pass. Evaluated against this repo’s epScript `_Reference` and current `.eps` code in 2026-11. Prefer the living docs under `_docs/roadmap.md` and `_docs/strategy/` for decisions; this file preserves the original proposals.

## 1. Project overview

The project is a custom StarCraft: Brood War / StarCraft Remastered UMS map implementing a grand-strategy-style simulation.

The map will support approximately 5–6 human players and use 2–3 conventional computer players, supplemented by extended player slots, to represent potentially dozens of virtual countries.

The central gameplay mechanics include:

- A large map divided into provinces.
- Countries represented as collections of provinces.
- Provinces changing ownership through military occupation.
- Occupation requiring a sufficient military presence for a specified duration.
- Multiple simultaneous wars involving multiple countries.
- Armies belonging to virtual countries rather than necessarily corresponding to individual StarCraft players.
- Military units spawned through triggers rather than conventional economic production.

Buildings are generally invincible and may be capturable. Conventional resource gathering and base construction are not central gameplay mechanics.

The primary technical challenge is representing many independent countries, armies, and simultaneous conflicts using StarCraft's limited player architecture.

## 2. Two possible AI architectures

### Approach A: Native AI scripts

StarCraft's native AI scripting language (ASC3) offers substantial functionality:

- Unit production and force requirements.
- Attack-force assembly.
- Defensive regional management.
- Town management.
- Pathfinding-aware strategic target selection.
- Combat-strength evaluation.
- Campaign and melee-specific behaviors.

It is possible to replace the native AI script buffer in Remastered using EUD triggers.

Important finding:

The Remastered EUD emulator supports both `AIScripts` and `AIScripts2`. The AI script pointer itself cannot be changed, but the buffer it references can be overwritten.

The emulator allocates an additional `0x6000` bytes for custom AI scripts.

The Staredit.net discussion "Facts about the SCR EUD emulator" documents this behavior.

More importantly, Jncraton demonstrated successful embedding of a custom `aiscript.bin` into EUD triggers in December 2017.

Reference project:
BWMetaAI — GitHub, by Jncraton.

EUD implementation:
`https://github.com/jncraton/BWMetaAI/blob/main/docs/eud.trg`

Relevant addresses in the historical implementation:
- Death table: `0x0058A364`
- Current Player: `0x006509B0`
- AIScript pointer: `0x0068C104`

Recommended AI editor: PyAI, particularly Neiv's compatible branch of PyMS.

### Limitations of native AI

Native AI is fundamentally player-centric.

A conventional computer player has:
- One strategic AI.
- One diplomatic identity.
- One vision state.
- One collection of units.
- One set of towns and defensive regions.

It cannot independently represent multiple hostile countries sharing the same StarCraft owner.

Furthermore, UMS maps force campaign-style AI behavior. Campaign AI selects targets and grouping regions differently from melee AI.

Campaign attack target selection is semi-randomized among eligible pathfinding regions. It does not offer a straightforward command for attacking an arbitrary user-defined map location.

The native AI therefore cannot directly understand the map's province and country abstractions.

### Approach B: Trigger-driven strategic AI

The alternative is to implement a relatively small strategic AI using EUD triggers, with StarCraft handling the actual tactical simulation.

The custom system would manage:
- Countries.
- Diplomacy.
- Wars.
- Military fronts.
- Armies.
- Province ownership.
- Occupation.
- Strategic objectives.
- Dynamic player allocation.

StarCraft would continue handling:
- Unit movement.
- Pathfinding.
- Weapon targeting.
- Combat resolution.
- Special abilities.
- Collision and terrain interactions.

This approach appears increasingly attractive because the map needs many simultaneous independent countries and conflicts.

It avoids trying to make the native AI understand concepts it was never designed to represent.

**Current architectural preference: a trigger-driven strategic director, potentially supplemented by native AI for selected conventional computer players.**

## 3. Player architecture and extended players

StarCraft provides eight conventional player slots.

Players 1–8 can participate normally and run AI scripts.

Players 9–11 are extended players:
- They can own units.
- They can be manipulated through EUDs.
- They do not have normal vision, including vision of their own units.
- They cannot run ordinary triggers.
- They cannot run conventional AI scripts.

Player 12 is the special Neutral player, not an ordinary extended computer player.

Extended-player units can be manipulated using CUnit pointers and low-level commands, but their ability to autonomously acquire targets and engage in combat is restricted by their unusual vision and ownership behavior.

An important distinction:

Being able to issue an order to an extended-player unit is not equivalent to giving its owner an autonomous AI.

Extended players should initially be treated as additional ownership namespaces rather than additional AI agents.

## 4. New finding: Alliance relationships include extended players

The EUD Book documents the Player Alliances table.

Address:
`0x0058D634`

Structure:
- 12 rows.
- 12 columns.
- One byte per relationship.
- Row represents the source player's perspective.
- Column represents the target player.

Values:
- `0`: Enemy
- `1`: Ally
- `2`: Allied Victory

Address formula:

`0x0058D634 + 12 * (source_player - 1) + (target_player - 1)`

Examples:

Player 1's relationship toward Player 9:
`0x0058D63C`

Player 9's relationship toward Player 1:
`0x0058D694`

The relationship matrix is directional. Both entries should be modified when reciprocal hostility or alliance is desired.

This provides a promising mechanism for dynamically changing the diplomatic relationships between conventional players and extended players.

However, alliance status and vision are separate systems.

The shared-vision table is documented at:
`0x0057F1EC`

Extended players have unusual vision handling. Changing their alliance entries does not establish that they can autonomously acquire targets.

Required experiments:
1. Set P1 → P9 to Enemy.
2. Set P9 → P1 to Enemy.
3. Observe whether P1 automatically acquires and attacks P9 units.
4. Test whether P9 units can acquire and attack P1 units.
5. Experiment with shared-vision entries.
6. Test actual weapon damage, not merely alliance indicators or displayed relationships.

This may be a critical component of the virtual-country architecture.

## 5. Proposed virtual-country data model

The central architectural principle is:

**StarCraft player numbers must not be authoritative identities for countries, armies, or wars.**

Player slots should instead be temporary execution resources.

Recommended logical tables:

### Countries

Each virtual country has a permanent numeric ID.

Suggested fields:
- `country_id`
- `controller_slot`
- `capital_province`
- `military_strength`
- `diplomacy_state`
- `active_armies`

### Provinces

Each province has a permanent ID and an independent owner.

Suggested fields:
- `province_id`
- `owner_country_id`
- `occupation_state`
- `occupation_timer`
- `strategic_value`
- `adjacent_provinces`

Province ownership should never be inferred from the StarCraft player currently owning the occupying units.

### Wars

Wars should be independent entities rather than simple hostile flags between countries.

Suggested fields:
- `war_id`
- `attacker_id`
- `defender_id`
- `participants`
- `war_state`
- `associated_fronts`

This permits multiple participants and multiple simultaneous conflicts.

### Armies

A country may have several armies, each assigned to a different strategic objective or front.

Suggested fields:
- `army_id`
- `country_id`
- `war_id`
- `objective_province`
- `execution_slot`
- `mission`
- `army_state`
- `unit_count`
- `ground_strength`
- `air_strength`

Possible states:
- Idle
- Marching
- Fighting
- Occupying
- Defending
- Reinforcing
- Retreating

### Player allocation

Maintain a separate allocation table for the actual StarCraft player slots.

Suggested fields:
- `player_slot`
- `allocated`
- `war_id`
- `country_id`
- `army_id`
- `allocation_generation`

The generation counter prevents stale references or delayed triggers from treating a reassigned slot as its previous assignment.

Player slots should be allocated to military formations or fronts, not permanently to countries.

## 6. Dynamic player allocation

Example:

Initially:
- Player 7 represents France's army on the German front.
- Player 8 represents Germany's army.
- Player 9 represents an Austrian expeditionary force.

Later, another war begins.

The system can release and reassign available slots, provided all associated units are transferred or otherwise handled correctly.

A possible arrangement:

| Slot | Temporary assignment |
|---|---|
| P7 | French army, German front |
| P8 | German army, French front |
| P9 | Austrian expeditionary force |
| P10 | Russian expeditionary force |
| P11 | Spanish expeditionary force |

The virtual country identities remain unchanged.

Only the execution assignments change.

Important caveat:

Changing a unit's CUnit owner field directly may not correctly update the engine's player-linked unit lists and other accounting structures.

The CUnit structure contains:
- Owner/player ID at `0x04C`.
- Previous/next unit pointers.
- Previous/next player-unit pointers.

Ownership reassignment should use a normal engine-supported ownership-changing mechanism wherever possible.

The effects of changing ownership on AI recognition, vision, alliances, supply, and unit lists must be tested.

## 7. Unit metadata and CUnit manipulation

Every unit needs a virtual identity independent of its actual StarCraft owner.

Proposed metadata:

- `country_id`
- `army_id`
- `unit_role`
- `rank`
- `flags`
- `generation`

This allows a trigger to determine the virtual nationality and army assignment of any unit.

### Sidecar metadata table

The preferred initial design is an external metadata table indexed by CUnit slot.

Conceptually:

`UnitMetadata[1700]`

Each entry contains:
- Country ID.
- Army ID.
- Unit role.
- Flags.
- Generation/identity information.

The standard CUnit structure is commonly documented as `0x150` bytes.

A sidecar table avoids modifying internal game fields that may be used by the engine.

### Potential CUnit fields

The following fields have been identified as interesting:

| Offset | Field | Assessment |
|---|---|---|
| `0x04C` | Player ID | Actively used; do not repurpose |
| `0x08C` | Apparently unused 16-bit field | Potential candidate |
| `0x08E` | Rank increase | Actively used |
| `0x08F` | Kill count | Actively used |
| `0x092` | AI action flag | Internally used |
| `0x0A5` | Uniqueness identifier | Engine-managed |

A field marked unused by BWAPI or an EUD reference is not necessarily safe to repurpose in every Remastered build.

The `0x08C` field may be worth investigating, but the sidecar design is preferable initially.

### CUnit slot reuse

CUnit indices are not permanent identities.

When a unit dies, its slot may eventually be reused.

Therefore:
- Invalidate metadata when units are removed.
- Maintain an active-unit list.
- Verify unit identity before trusting a metadata entry.
- Investigate the engine's uniqueness identifier at `0x0A5` as part of this verification.

A separate `ActiveUnits[]` list can avoid scanning the entire CUnit table every strategic update.

## 8. Strategic AI design

The proposed trigger-driven AI should be a strategic director, not a complete replacement for StarCraft's tactical AI.

Suggested components:

### A. National state

Each country tracks:
- Owned provinces.
- Available military strength.
- Diplomacy.
- Active wars.
- Active armies.
- Strategic priorities.

### B. Objective selection

Periodically evaluate possible objectives:
- Capture adjacent enemy provinces.
- Defend threatened provinces.
- Retake lost territory.
- Reinforce an allied country.
- Withdraw from untenable positions.

A simple scoring model can consider:
- Distance.
- Province value.
- Enemy strength.
- Friendly strength.
- Adjacency.
- Diplomatic relationships.
- Existing military commitments.

### C. Force management

Assign units to armies and objectives.

Use ordinary StarCraft orders wherever possible.

The custom system should intervene primarily when an army receives a new objective or when an important tactical condition occurs.

### D. Province occupation

When sufficient military units enter a province:
- Begin occupation timer.
- Check whether the required force remains present.
- Pause or reset the timer if occupation conditions fail.
- Transfer ownership when the timer completes.
- Update national state.
- Release the army for a subsequent objective.

## 9. Native AI research findings worth retaining

Although the trigger-driven approach is currently preferred, several native AI features remain useful.

### Towns and regions

Important commands:
- `start_town`
- `start_areatown`
- `value_area`
- `guard_all`
- `capt_expand`

The AI maintains internal pathfinding regions with states:

| State | Meaning |
|---|---|
| 0 | Inactive/unclassified |
| 1 | Continuing attack |
| 2 | Initial attack target |
| 3 | Enemy detected |
| 4 | Potential defensive territory |
| 5 | Town/defended territory |
| 6 | Actively defended |
| 7 | Chasing enemy |
| 8 | Preparing attack |
| 9 | Preparing attack after unreachable target |

`value_area` sets the pathfinding region containing the center of a map location to state 5.

If the AI does not control a building in that region, it may only defend the region once.

This suggests an experiment involving AI-owned invincible buildings as territorial anchors.

Potential configurations:
1. `value_area` alone.
2. `value_area` plus an AI-owned invincible building.
3. Separate `start_areatown` initialization for each province.

The building's placement determines its relationship to the pathfinding region; its footprint does not define the region.

### Attack force commands

- `attack_add`: Adds a unit type to the attack force.
- `attack_prepare`: Begins assembling an attack force.
- `attack_do`: Waits for the force to assemble and launches the attack.
- `wait_finishattack`: Waits for attack-related regional activity to finish.
- `attack_clear`: Clears the attack force and can abort an attack during its grouping phase.
- `eval_harass`: Evaluates friendly versus enemy strength near a target region.

`attack_add` and similar commands generally require specific unit types. There is no general-purpose equivalent of the trigger editor's `Men` or `Any Units` selector.

`wait_force` and `wait_train` are relevant to force production.

### Virtualized production

A promising experiment is to give AI players zero-cost, zero-build-time production infrastructure.

The AI can issue normal training requests while triggers handle or redirect the resulting units.

Preferred initial experiment:
- Let the game complete training normally.
- Verify that the AI recognizes the resulting units.
- Then investigate queue interception and trigger-based spawning.

The key question is whether trigger-created units satisfy `wait_force` and the native AI's regional force requirements.

### Strategic target limitations

Native campaign AI does not accept arbitrary map-location attack targets.

`set_gencmd` and `make_patrol` offer some location-specific behavior, but they are not equivalent to a general strategic objective API.

This is another reason to retain strategic objective selection in the custom trigger system.

## 10. Recommended architecture

The current preferred design is a three-layer system:

| Layer | Responsibilities |
|---|---|
| Strategic simulation | Countries, diplomacy, wars, provinces, objectives |
| Military simulation | Armies, unit metadata, assignments, occupation |
| StarCraft execution | Player ownership, CUnit pointers, movement, combat |

The strategic layer should never need to know which StarCraft player owns a unit.

The military layer translates strategic decisions into unit assignments.

The execution layer determines which actual player slot owns the units and how they are manipulated.

Native AI can potentially be retained for selected conventional computer players, but it should not be the authoritative strategic system.

## 11. Highest-priority experiments

Before committing to the architecture, investigate these questions:

1. Can the alliance table at `0x0058D634` be modified during gameplay to make P1–P8 hostile or allied toward P9–P11?
2. Does changing the alliance table actually alter weapon targeting and damage?
3. Can shared vision be configured to make extended-player units acquire targets?
4. Can extended-player units fight normally when given explicit CUnit-level orders?
5. Can units be transferred between conventional player slots without corrupting linked lists or other engine state?
6. Can two virtual countries sharing the same StarCraft owner reliably fight each other?
7. Does the game maintain sufficient distinction between virtual factions when their units share an owner?
8. Can CUnit sidecar metadata be maintained reliably through unit creation, death, removal, and slot reuse?
9. Can native AI recognize trigger-created units as satisfying its force requirements?
10. Can an AI-owned dummy building establish persistent defensive behavior in an artificial province?

The first four questions are particularly important because they determine how much of the military simulation can use actual StarCraft ownership and combat mechanics.

## 12. Primary technical references

- Staredit.net — Facts about the SCR EUD emulator.
- Staredit.net — Using EUD for custom AI.
- Jncraton — BWMetaAI, including its historical EUD map implementation.
- PyMS — AI Basics and `aiscript.bin` documentation.
- FraudsClub — SCAI Command Guide.
- Staredit.net — Defining AI commands and functions.
- Bob Fitch interview — Original StarCraft AI design and regional behavior.
- EUD Book — Player Alliances and CUnit memory layouts.
- BWAPI — CUnit structure definitions.
- SCMJS documentation — EUD, Players and Groups, and game memory tables.

---

## Final assessment

The most promising direction is a custom virtual-country simulation with dynamic player allocation and CUnit-level unit bookkeeping.

The native AI is valuable, but its player-centric architecture conflicts with the map's requirement for many independent countries and simultaneous conflicts.

A trigger-driven strategic director avoids that limitation while retaining StarCraft's native tactical simulation.

The most important unresolved technical issue is whether extended-player ownership and dynamically modified alliance relationships can provide enough of the native combat behavior to support the desired architecture.

The system should be designed so that country identity, army identity, war participation, and unit ownership remain independent concepts.
