# Wars, Diplomacy, and Occupation

**Parent:** [../roadmap.md](../roadmap.md)  
**Related:** [ai-architecture.md](ai-architecture.md), [virtual-countries-and-players.md](virtual-countries-and-players.md)

## Intent

Model **wars as first-class entities**, map them onto StarCraft alliances only at the execution layer, and grow occupation from the siege system already in [`duchies.eps`](../../duchies.eps).

Product details (warscore formulas, peace UI, casus belli lists) are **not** bound to 2010s `_docs/*Plan*.txt` checklists. Those files are inspiration only.

---

## Wars as entities

A war is more than “these two players are enemies.”

Minimum fields (see countries doc):

- `war_id`
- Primary attacker / defender country IDs
- Participant set
- State (active, negotiating, ended)
- Associated fronts / armies

### Why entities

- Multiple simultaneous wars
- Third-party joiners
- Separate peace later without collapsing all hostility globally
- AI can score “which war am I prosecuting?” independently of SC foe lists

### Execution mapping

When an army is allocated to a slot for a war:

1. Set that slot Enemy toward slots executing hostile participants (`SetAllianceStatus` + `setcurpl`, both directions).
2. Ally toward friendly co-belligerent slots as needed.
3. On war end or reallocation, recompute alliances from remaining active wars — do not leave stale enmity.

Logical diplomacy lives in `diplomacy.eps`. SC alliance bytes are a **projection**, not the source of truth.

---

## Diplomacy module responsibilities

[`diplomacy.eps`](../../diplomacy.eps) is currently a stub. Target responsibilities:

| Concern | Notes |
|---------|--------|
| Relation matrix country↔country | Peace, alliance, war, truce — virtual |
| War create / join / end | Mutates war table |
| Human diplomatic actions | Later UI (demands, white peace, etc.) |
| AI diplomatic impulses | Phase 3+ scoring (“should I attack neighbor?”) |
| Slot alliance projection | Calls execution helpers when armies allocate |

Do not require humans to understand SC player indices for diplomacy — only countries.

---

## Attrition / supply in enemy territory

Prefer engine-delegated supply fuses (`removeTimer`) over virtual per-frame food clocks. See [engine-delegation-and-attrition.md](engine-delegation-and-attrition.md).

## Occupation and sieges — bridge from current code

### What already works

In [`duchies.eps`](../../duchies.eps):

- Detect ≥8 enemy men on a duchy siege location while defender is outnumbered
- Spawn siege gear timer building
- Cancel if attackers drop below threshold
- On timer death: `GiveUnits` duchy building; liberate vs occupy; HP% feedback; levy flags reset

This **is** the occupation mechanic for duchies, expressed in SC player terms.

### What to add for virtual countries

| Gap | Direction |
|-----|-----------|
| Occupier identity | Store `occupier_country_id`, not only SC owner |
| Liberate path | Compare sieging country to `owner_country_id` |
| Income while occupied | Economy should key off country ownership / occupation flags |
| Town/castle sieges | Still missing; optional later content |
| Warscore | Separate system; sieges feed it but do not replace wars |

Fortification levels should eventually drive siege duration (Phase 0 wiring gap).

### Occupation state machine (target)

```text
Peaceful → Contested → UnderSiege → Occupied → (Liberated | Annexed via peace)
```

Annexation (permanent owner change beyond wartime occupation) should be a **peace resolution** outcome, not an automatic side effect of every siege — exact rule is a product decision in Phase 3–4.

---

## Fronts and armies

A front is the pairing of:

- A war
- A geographic focus (province / region)
- One or more armies with execution slots

AI (Phase 3):

1. Score adjacent enemy provinces (`adjecency.graphml` → arrays)
2. Assign or create an army with mission Capture / Defend / Retake
3. Allocate slot if combat required against a hostile executor
4. `Order` forces toward objective; let existing siege checks run
5. On success, update occupation + warscore inputs; pick next objective

Humans issue the same underlying levy/siege verbs; AI uses the data model without a second physics.

---

## Warscore and peace (later product)

Not required to unblock architecture. When designed:

- Inputs: occupied provinces, battles (unit losses), ticking timers
- Outputs: AI accept/reject thresholds; human peace UI
- Keep war entities until peace fully resolves participant relations

Avoid encoding “three global war slots” from ancient plan notes unless that constraint proves useful.

---

## Module split reminder

| Module | Owns |
|--------|------|
| `diplomacy.eps` | Relations, wars, peace, alliance projection |
| `military.eps` | Armies, fronts, objective assignment, orders |
| `duchies.eps` | Province buildings, levy spawn, siege detection (until/unless moved) |
| `economy.eps` | Pays countries based on ownership/occupation rules |
| `events.eps` | Random wartime events (optional flavor) |
