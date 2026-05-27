# AGENTS.md — True-Unleveled-Skyrim

## What this is

A **C# (.NET 8) Synthesis patcher** for Skyrim Special Edition. It is invoked by the [Synthesis](https://github.com/Mutagen-Modding/Synthesis) modding framework against a live Skyrim load order and produces a generated plugin file (`TUS.esp`). It is **not a standalone executable** — running it outside Synthesis is unsupported.

Single-project solution: `TrueUnleveledSkyrim.sln` → `TrueUnleveledSkyrim/TrueUnleveledSkyrim.csproj`.

---

## Build commands

```sh
dotnet restore TrueUnleveledSkyrim.sln
dotnet build TrueUnleveledSkyrim.sln
dotnet build TrueUnleveledSkyrim.sln -c Release
```

There is no CI, no test project, no lint script, and no task runner. `dotnet build` is the only verification step available.

---

## Nullable strictness — build will fail if violated

`Directory.Build.props` sets:
```xml
<Nullable>enable</Nullable>
<WarningsAsErrors>nullable</WarningsAsErrors>
```

**All nullable warnings are errors.** The codebase uses `null!` suppression and null-forgiving operators where Synthesis guarantees initialization. New code must follow the same pattern or resolve nullability properly — the build will not succeed otherwise.

---

## Patch execution order (order matters)

In `Program.cs`, steps run in this order and depend on each other:

1. `TUSConstants.GetPaths(state)` — resolves JSON file paths from Synthesis's `ExtraSettingsDataPath`
2. `LeveledItemsPatcher.PatchLVLI` — generates `_TUS_Weak` / `_TUS_Strong` leveled list variants; **must run before NPCs and Outfits**
3. `OutfitsPatcher.PatchOutfits` — references the `_TUS_*` variants from step 2
4. `NPCsPatcher.PatchNPCs` — uses items/outfits; internally uses `Patcher.LinkCache` (static) for newly created classes, not `state.LinkCache` (dynamic), because dynamic won't contain those classes yet
5. `ZonesPatcher.PatchZones`
6. `ItemsPatcher.PatchItems`

Within `NPCsPatcher`, per-NPC processing order is also fixed:
`SetStaticLevel → RebalanceClassValues → ChangeEquipment → RelevelNPCSkills → DistributeNPCPerks → SetFollowerScaling`

`DisableExtraDamagePerks` runs globally **after** the NPC loop.

---

## Runtime JSON config (`Data/` directory)

All JSON files in `TrueUnleveledSkyrim/Data/` are read at runtime via Newtonsoft.Json. Synthesis supplies this path as `ExtraSettingsDataPath`. Key files:

| File | Purpose |
|---|---|
| `zoneTypesByEDID.json` / `zoneTypesByEDIDMLU.json` | Zone levels by EditorID (default / Morrowloot Ultimate) |
| `zoneTypesByKeyword.json` / `zoneTypesByKeywordMLU.json` | Zone levels by location keyword |
| `NPCsByEDID.json` / `NPCsByFaction.json` | Manual NPC level overrides |
| `excludedNPCs.json` / `excludedLVLI.json` / `excludedPerks.json` | Exclusion lists |
| `raceLevelModifiers.json` | Additive/multiplicative level modifiers by race |
| `artifactKeys.json` | EditorID substrings identifying artifact leveled lists |
| `customFollowers.json` | Custom follower NPC entries |
| `settings.json` | Default Synthesis GUI settings (shipped with patcher) |

**All key matching is case-insensitive and uses partial EditorID substring matching**, not full FormIDs.

---

## Generated records (runtime, not on disk)

The patcher creates new records in `TUS.esp` at runtime — there is no codegen step:
- New leveled item lists: EditorID postfix `_TUS_Weak`, `_TUS_Strong`
- New outfit variants: same postfixes
- New NPC classes: EditorID prefix `TUSClass` + NPC EditorID

Do not look for these in source; they exist only in the output plugin.

---

## Key domain conventions

- **Winning overrides**: all patchers iterate `state.LoadOrder.PriorityOrder.*.WinningOverrides()` to get the final form of each record.
- **Template flags**: NPCs with `TemplateFlag.Stats` + non-null template skip stat/class changes; `TemplateFlag.SpellList` + non-null template skip perk changes.
- **Class rebuild exclusions**: unique NPCs whose class EditorID contains any of `{ "smith", "alchem", "enchant", "vendor", "apothec" }` are skipped.
- **Vanilla cache**: A separate link cache is built from only `Skyrim.esm`, `Dawnguard.esm`, `Dragonborn.esm` for stripping vanilla perks (`RemoveVanillaPerks`).
- **Follower detection**: checks `PotentialFollowerFaction`, `PotentialHireling` faction, and `customFollowers.json`.
- **Item tier levels** (for `MaxItemLevel` setting): 1=Iron, 2=Steel, 6=Orcish, 12=Dwarven, 19=Elven, 27=Glass, 36=Ebony, 46=Daedric. Default `MaxItemLevel` is 27.
- Perks with `"NoASIS"` in their EditorID are conventionally excluded via `excludedPerks.json`.

---

## Project layout

```
TrueUnleveledSkyrim/
├── Program.cs              ← Entry point, patch step order
├── Config/
│   ├── ConfigTypes.cs      ← JSON-mapped data types
│   ├── Constants.cs        ← File path constants for Data/ JSON files
│   ├── JsonHelper.cs       ← Newtonsoft.Json loader helpers
│   └── Settings.cs         ← Synthesis GUI settings (TUSConfig)
├── Data/                   ← Bundled JSON config (runtime input)
└── Patch/
    ├── Items.cs            ← Equipment stat rebalancer (Morrowloot stats)
    ├── LeveledItems.cs     ← Leveled list unleveler
    ├── NPCs.cs             ← NPC unleveler, class rebuilder, perk distributor
    ├── Outfits.cs          ← Outfit weak/strong variant generator
    └── Zones.cs            ← Encounter zone unleveler
```

---

## EditorConfig / line endings

`.editorconfig` enforces `crlf` line endings and `utf-8` encoding for all files. `CS4014` (unawaited task) is an error; `CS1998` (async without await) is suppressed.
