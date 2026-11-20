# EUD Editor → TrigScript patch export

Convention for consuming DatEdit / Firegraft edits from the **eudeditor-linux** fork without euddraft.

## Drop path

Export from EUD Editor 2 (**Tools → Export for TrigScript…**) into:

```text
SCR_CrusaderKings/trigscript/eud-patches/
```

Expected files:

| File | Role |
| --- | --- |
| `eud-patches.json` | Structured memory ops + metadata |
| `RequireData.bin` | Present when Firegraft requirements are enabled; raw opcode blob (also expanded as dword ops in the JSON) |

Absolute path on this machine:

`/home/michael/maps/SCR_CrusaderKings/trigscript/eud-patches/`

## JSON schema (Phase 1)

Top-level fields:

- `schemaVersion` — currently `1`
- `starVersion` — offset table used (`Remastered` / `1.16.1`)
- `buffer.requireBase` / `buffer.buttonBase` — EUD-writable staging (Vanilla + reserve)
- `ops[]` — memory operations

Each op:

```json
{
  "address": 6702328,
  "addressHex": "0x6644F8",
  "modifier": "Add",
  "value": 127,
  "valueHex": "0x7F",
  "meta": {
    "source": "datEdit",
    "table": "units",
    "field": "Graphics",
    "objectId": 7,
    "size": 1,
    "delta": 127
  }
}
```

`meta.source` values: `datEdit`, `buttonPayload`, `buttonPointer`, `buttonCount`, `firegraftStatus`, `requirePayload`, `requirePointerTable`, `requireBaseRedirect`.

Modifiers match classic EUD / TrigScript: `SetTo`, `Add`, `Subtract`.

## TrigScript consumption

Prefer lift order when codegen improves:

1. `stats(...)` for named unit/weapon/upgrade/tech fields `meta` can identify
2. `memory.*` for named non-table values
3. `setMemory(address, modifier, value)` for everything else (Phase 1 default)

Minimal apply-all pattern (illustrative):

```ts
// Generated or hand-written from eud-patches.json — map-local Remastered EUD, not a scmjs mod.
program(() => {
  // once at start: apply each op
  setMemory(0x6644FC, Add, 0x7F000000); // example packed Graphics delta
});
```

Do **not** ship these table patches as a `scmjs mod` pack. CK requires map-local EUD inside the `.scx`.

## Producer

- Repo: `/home/michael/maps/eudeditor-linux/`
- Module: `EUD Editor/Module/TrigScriptExportModule.vb`
- With `euddraftuse : False` in the `.e2p`, map-insert opens the TrigScript export folder picker instead of euddraft.

## Wine

Phase 1 still runs the WinForms EE2 binary under Wine (or Windows). Native UI is Phase 2.
