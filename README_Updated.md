# Icarus Modding — DataTable Reference

A practical reference for **data-only Icarus modding** using the DataTable JSON exported by the
[Jimk72/Icarus_Software](https://github.com/Jimk72/Icarus_Software) tool. All 298 tables live
under `data/`, organized into category folders that match that tool's own `CurrentFile`
category system (`data/Items/`, `data/Traits/`, `data/Alterations/`, etc. — Section 12 has the
full map).

This document has two jobs:
1. A reference for the *verified structure* of the tables and how they connect.
2. A discipline for *investigating new questions* without re-deriving things from scratch or
   quietly turning a guess into a "fact."

The most important idea in this whole repo: **the item is rarely the complete system.**
`D_ItemsStatic` is a hub of pointers — the real answer to "what controls X" is almost always in
a different table than the one you started in. Follow the reference, don't guess the table.

---

## 0. Confidence levels

Every relationship documented below falls into one of three buckets. When adding new findings,
tag them the same way — don't let a plausible guess quietly become treated as fact.

- **VERIFIED** — confirmed either by direct JSON inspection *and* working in-game behavior, or
  by a clear, unambiguous structural pattern repeated across many rows (e.g. the item-component
  pointer pattern in §2).
- **INFERRED** — strongly supported by naming, `Defaults`, and cross-table consistency, but not
  independently confirmed in-game.
- **UNTESTED** — the JSON relationship exists, but the actual gameplay effect hasn't been
  confirmed yet.

Most of this document is VERIFIED against real in-game testing done over the course of several
mods (attachments, water systems, slot counts). Anything still open is flagged inline.

---

## 1. The core reference pattern *(VERIFIED)*

Each table has the same shell:

```json
{
  "RowStruct": "/Script/Icarus.SomeStructType",
  "Defaults": { /* default values for every field a row can have */ },
  "Columns":  [ /* metadata about key columns, mostly irrelevant for modding */ ],
  "Rows": [
    { "Name": "UniqueRowName", "...": "..." }
  ]
}
```

Tables reference each other with small pointer objects instead of embedding data:

```json
"SomeField": { "RowName": "TargetRowName" }
"SomeField": { "RowName": "TargetRowName", "DataTableName": "D_SomeTable" }
```

If `DataTableName` is present, it tells you exactly which table to look in. If it's omitted, the
target table is implied by the field itself — **check that field's entry in the same file's
`Defaults` block to confirm which table it defaults to. Never assume from the field name alone.**

```python
import json

def load(path):
    with open(path) as f:
        return {r["Name"]: r for r in json.load(f)["Rows"]}

items_static = load("data/Items/D_ItemsStatic.json")
itemable     = load("data/Traits/D_Itemable.json")
```

---

## 2. Anatomy of an item — the component pattern *(VERIFIED)*

Every item is a `D_ItemsStatic` row acting as a hub, with each gameplay facet split into its own
component table. A row only carries the components it actually needs.

**Example — `Rifle_Hunting`:**

```json
{
  "Name": "Rifle_Hunting",
  "Meshable":          { "RowName": "Mesh_Rifle_Hunting" },
  "Itemable":          { "RowName": "Item_Rifle_Hunting" },
  "Interactable":      { "RowName": "Item" },
  "Focusable":         { "RowName": "Focusable_Gun_Rifle" },
  "Highlightable":     { "RowName": "Generic" },
  "Actionable":        { "RowName": "Rifle_SingleShot" },
  "Usable":            { "RowName": "Repair" },
  "Durable":           { "RowName": "Rifle_Hunting" },
  "Decayable":         { "RowName": "Decay_General" },
  "InventoryContainer":{ "RowName": "Rifle_Ammo_Attachment" },
  "Audio":             { "RowName": "Rifle" },
  "FirearmData":       { "RowName": "Rifle_Hunting" },
  "AdditionalStats":   { "(Value=\"BaseCriticalDamage_+%\")": 100 },
  "Manual_Tags":       { "GameplayTags": [ { "TagName": "Item.Weapon.Firearm.Rifle" } ] }
}
```

The pointer's `RowName` does **not** have to match the parent item's `Name` — `Itemable` points
to `Item_Rifle_Hunting`, not `Rifle_Hunting`, so multiple items can reuse the same
display/description/weight data.

### Component tables

| Field | Target (folder/table) | Purpose |
|---|---|---|
| `Meshable` | `Traits/D_Meshable` | 3D mesh (reuse existing — no new assets in data-only modding) |
| `Itemable` | `Traits/D_Itemable` | Display name, description, icon, weight, stack size |
| `Interactable` | `Traits/D_Interactable` | World pickup interaction |
| `Highlightable` | `Traits/D_Highlightable` | Outline/highlight visuals |
| `Focusable` | `Traits/D_Focusable` | ADS/focus behavior |
| `Actionable` | `Traits/D_Actionable` | Actions exposed (fire, swing, use) → `Tools/D_Actions` |
| `Usable` | `Traits/D_Usable` | Right-click/use behavior |
| `Durable` | `Traits/D_Durable` | Max durability, degradation |
| `Decayable` | `Traits/D_Decayable` | Spoilage/decay curve |
| `Equippable` | `Traits/D_Equippable` | Equip slot rules |
| `Consumable` | `Traits/D_Consumable` | Eat/drink effects |
| `Fillable` | `Traits/D_Fillable` | Liquid-container capacity |
| `Resource` | `Traits/D_Resource` | Resource-network connection flags (§7) |
| `Slotable` | `Traits/D_Slotable` | World-socket item slotting (§8) |
| `ToolDamage` | `Tools/D_ToolDamage` | Melee/tool damage, mining efficiency/radius |
| `RangedWeaponData` | `Tools/D_RangedWeaponData` | Shared ranged-weapon stats |
| `FirearmData` | `Tools/D_FirearmData` | Firearm ballistics, recoil, fire rate |
| `Armour` | `Traits/D_Armour` | Armour stats & mesh |
| `Buildable` / `Deployable` | `Traits/D_Buildable` / `D_Deployable` | Placement behavior → `Deployables/D_DeployableSetup` |
| `Generator` | `Traits/D_Generator` | Resource generation/conversion rate (§7) |
| `AmmoType` | `Tools/D_AmmoTypes` | Ammo payload behavior |
| `Ballistic` | `Traits/D_Ballistic` | Projectile physics |
| `InventoryContainer` | `Inventory/D_InventoryContainer` | Attachment/ammo sub-inventory (§6) |
| `Attachments` | `Attachments/D_IcarusAttachments` | Links attachment item → stat bonus (§5) |
| `Processing` | `Traits/D_Processing` | Crafting-bench recipe queue, shelter requirement |
| `LivingItem` | `Traits/D_LivingItem` | Great Hunt legendary weapon upgrade system (§9) |
| `Audio` | varies (`Audio/D_FirearmAudioData`, etc.) | Sound cues |

`Items/D_ItemTemplate.json` is a thin second layer: `{ "Name": "X", "ItemStaticData": { "RowName": "X" } }`.
`D_ItemsStatic` = the item's definition. `D_ItemTemplate` = the handle recipes/rewards actually output.

> ⚠️ **`Traits/D_Transmutable.json` is NOT the weapon-alteration system**, despite the name — it's
> the incinerator's fuel-conversion table (`Wood`, `Stick`, `Biofuel`). The real stat-alteration
> system is `D_IcarusAttachments → D_Alterations` (§5).

---

## 3. Anatomy of a recipe *(VERIFIED)*

Recipes: `Crafting/D_ProcessorRecipes.json` (benches) or `Crafting/D_ExtractorRecipes.json` (auto-extractors).

```json
{
  "Name": "Stone_Pickaxe",
  "Requirement": { "RowName": "Stone_Pickaxe" },
  "RecipeSets": [ { "RowName": "Character", "DataTableName": "D_RecipeSets" } ],
  "Inputs": [
    { "Element": { "RowName": "Fiber", "DataTableName": "D_ItemsStatic" }, "Count": 10 }
  ],
  "Outputs": [
    { "Element": { "RowName": "Stone_Pickaxe", "DataTableName": "D_ItemTemplate" }, "Count": 1 }
  ]
}
```

- `Inputs.Element` → **`D_ItemsStatic`** (raw material cost).
- `Outputs.Element` → **`D_ItemTemplate`** (the craftable result) — not `D_ItemsStatic`.
- `Requirement.RowName` → **`Talents/D_Talents`** — talent gate. Omit for always-craftable.
- `RecipeSets` → **`Crafting/D_RecipeSets`** — which station(s); `Character` = no bench needed,
  and a recipe can list multiple sets (e.g. `Character` + a bench) to be available both ways.
- Feature-level gating: `"Metadata": { "RequiredFeatureLevel": { "RowName": "DangerousHorizons" } }`
  → `Development/D_FeatureLevels.json`.

---

## 4. Anatomy of a talent unlock *(VERIFIED)*

```json
{
  "Name": "Stone_Axe",
  "ExtraData":   { "RowName": "Item_Stone_Axe", "DataTableName": "D_Itemable" },
  "TalentTree":  { "RowName": "Blueprint_T1_Player" },
  "RequiredTalents": [ { "RowName": "Stone_Tools_Rerout", "DataTableName": "D_Talents" } ]
}
```

- `TalentTree.RowName` → `D_TalentTrees` → `Archetype` → `D_TalentArchetypes` (which UI panel).
- `RequiredTalents` builds prerequisite chains.
- A recipe links to a talent purely via `Requirement.RowName` matching a `D_Talents` row `Name`
  — **there's no reverse pointer on the talent itself.**

---

## 5. Weapon attachments (equippable-item stat system) *(VERIFIED)*

```
D_ItemsStatic (attachment item)
  → Attachments.RowName → Attachments/D_IcarusAttachments.json row
      → GrantedAlteration.RowName → Alterations/D_Alterations.json row (the Stats{})
```

```json
// Items/D_ItemsStatic.json
{
  "Name": "Rifle_Attachment_Accuracy_1",
  "Meshable": { "RowName": "Mesh_Attachment_Ranged_Weapon" },
  "Attachments": { "RowName": "Rifle_Accuracy_1" },
  "Manual_Tags": { "GameplayTags": [
    { "TagName": "Item.Attachment.Rank.1" }, { "TagName": "Item.Attachment.Rifle" }
  ]}
}
// Attachments/D_IcarusAttachments.json
{ "Name": "Rifle_Accuracy_1", "GrantedAlteration": { "RowName": "Rifle_Accuracy_1" } }
// Alterations/D_Alterations.json
{ "Name": "Rifle_Accuracy_1", "Stats": { "(Value=\"BaseRifleProjectileAccuracy_+%\")": 20 } }
```

All standard rifle/pistol/bow/shotgun/crossbow attachments share **one generic mesh**
(`Mesh_Attachment_Ranged_Weapon`) — no unique model needed, only a unique icon. The
`Item.Attachment.<WeaponClass>` tag (`Rifle`/`Pistol`/`Shotgun`/`Bow`/`Crossbow`) is what makes
an item fit that weapon class's slot (§6). Attachments craft at `Alteration_Bench`/
`Advanced_Alteration_Bench` (or `Character` for no-bench crafting) — the bench is only the
crafting station, unrelated to how the item gets installed.

> 🪤 **Real bug hit in production:** `D_IcarusAttachments.json` actually lives in the
> **`Attachments/`** category folder, not `Traits/` — a wrong guess here silently broke an entire
> attachment mod's stats (icon and craftability still worked, since those come from different
> tables). **Always verify a table's real category against the repo folder before setting
> `CurrentFile`** — this is exactly Mistake #3 below, and it's the most expensive one to get
> wrong because the failure is silent.

---

## 6. Weapon attachment **slots** (how many an item can hold) *(VERIFIED)*

```
D_ItemsStatic.InventoryContainer.RowName
    → Inventory/D_InventoryContainer.json row
        → InventoryInfo.RowName → Inventory/D_InventoryInfo.json row
            → StartingSlots (total slot count)
            → SlotOverrides (pins specific indices to a D_TagQueries filter, e.g. index 0 = ammo)
            → SlotTemplate (fallback filter for any index not covered by SlotOverrides)
```

```json
// D_InventoryContainer.json
{ "Name": "Rifle_Ammo_Attachment", "InventoryInfo": { "RowName": "Rifle_Ammo_Attachment" }, "AttachmentSlot": 1 }
// D_InventoryInfo.json
{
  "Name": "Rifle_Ammo_Attachment", "StartingSlots": 2,
  "SlotTemplate": { "RowName": "Any_Rifle_Attachment" },
  "SlotOverrides": [ { "Query": { "RowName": "Any_Ammo", "DataTableName": "D_TagQueries" }, "Location": 0 } ]
}
```

⚠️ **This container is shared** across every weapon referencing it by name (both `Rifle_Hunting`
and `Rifle_BoltAction` use `Rifle_Ammo_Attachment`). Editing it changes every weapon using it. To
change slots for **one** weapon only: clone both rows under a new name and repoint that weapon's
`InventoryContainer` at the clone — this is the general **shared-row safety pattern** (§13),
apply it any time you're about to edit a row you haven't confirmed is exclusive to your target.

No vanilla weapon in the dataset exceeds `StartingSlots: 2`.

---

## 7. The resource network — Water/Energy `Store` / `Produce` / `Consume` *(VERIFIED — new this pass)*

This system controls how liquids/power move between connected deployables via pipes/wires, and
it was the source of a lot of confusion until fully traced. It's driven by `Traits/D_Resource.json`
(the connection switch) + `Traits/D_Water.json` or `Traits/D_Energy.json` (the actual flow behavior).

```
D_ItemsStatic.Resource.RowName
    → Traits/D_Resource.json row
        → bHasWaterConnection: true/false   (default FALSE — must be explicitly set)
        → WaterFlow.RowName → Traits/D_Water.json row
            → FlowType: "Store" | "Produce" | (omitted = "Consume")
            → ResourceFlowRate (default 1000 if unset)
```

### The three flow types, confirmed by testing

| FlowType | Behavior | Examples |
|---|---|---|
| **`Store`** | Passive. Holds resource, can be drawn from by a `Consume` node, but **never actively pushes** to another node — including another `Store` node. Two `Store` nodes connected to each other transfer **nothing**, no matter how full one is. | `Rain_Reservoir`, `Rain_Reservoir_T3`, `Water_Barrel`, `Homestead_Water_Tank`, `Homestead_Well`, `Portable_Tank_Water` |
| **`Produce`** | Active, **unconditional** source — pushes `ResourceFlowRate` per second regardless of whether it actually has anything backing it. Not tied to a `Generator` component's accumulated amount at all. Vanilla only uses this on devices meant to sit in a real water body (see `WorldPlacementType` below) — using it on a ground-placed item creates free/infinite resource. | `Pump`, `Biofuel_Water_Pump` |
| **`Consume`** (default — field simply omitted) | Active puller. Requests up to `ResourceFlowRate` from anything connected that has supply (`Store` or `Produce`), respecting real available quantity. **This is the only flow type confirmed to correctly draw a finite amount from a `Store` node.** | `Water_Purifier_T2`, all `Water_Trough_*` variants |

**Practical rule confirmed by an in-game A/B test:** a `Store` reservoir successfully supplied a
`Consume` purifier's request, but the same reservoir connected to a `Store` barrel produced
nothing (`"insufficient Water Input"`) even with a real pipe connection. **`Store → Store` never
transfers; only `Consume` actively pulls, and only `Produce` actively (and unconditionally)
pushes.** If you want a passive tank to auto-fill from another passive tank, the *tank receiving
water* needs to become `Consume`-type — following the exact pattern every `Water_Trough_*` uses
— not the source becoming `Produce` (that creates free/unlimited resource instead of a real
finite transfer).

⚠️ Flipping a *supply* tank's own `FlowType` to `Consume` would also stop it from being able to
*supply* anything downstream of it that currently draws from it — check what else references a
row before changing its role, same shared-row caution as everywhere else.

### `WorldPlacementType` — why the vanilla Pump needs to be in water *(VERIFIED)*

```
D_ItemsStatic.Deployable.RowName → Deployables/D_DeployableSetup.json row
    → WorldPlacementType (default: "GroundPlacement")
```

`Biofuel_Water_Pump`'s `DeployableSetup` row explicitly sets `"WorldPlacementType": "WaterPlacement"`
— it can only be placed on/in a real body of water. This is *why* it's safe for that item to use
an unconditional `Produce` flow (its "storage" is the ocean/river, effectively infinite already).
Any custom item using `Produce` without this field defaults to normal ground placement — meaning
it becomes a genuinely free water source with no environmental cost, unlike the vanilla pump.

---

## 8. `D_Slotable` — filling portable containers (a third, separate mechanism) *(VERIFIED — new this pass)*

Distinct from both §5 (attachments) and §7 (the resource network). This is how a device refills
a portable item (canteen, waterskin) placed inside its own inventory, rather than moving bulk
resource through pipes.

```
D_ItemsStatic.Slotable.RowName → Traits/D_Slotable.json row
    → SocketStringID (e.g. "tank")
    → StringIDQueries → SlotQuery.RowName → Tags/D_TagQueries.json (e.g. "Water_Containers")
```

`Water_Purifier_T2` combines all three systems on one item: it's a `Consume` node on the water
network (pulls 200/sec — matches the in-game "Req: 200" readout exactly), uses a `Generator`
component (`TransmutableItems: [Charcoal]`) as filter fuel, and uses `Slotable: "WaterRefill"` to
actually top up a canteen/bottle sitting in its "tank" socket. The network-pulled water and the
portable-container refill are two different outputs of the same item, governed by two entirely
separate component systems — don't assume one implies the other.

---

## 9. Living Items (Great Hunt legendary weapons) *(VERIFIED)*

Separate from §5/§6. Used only by the 9 named legendary weapons from Great Hunt bosses.

- **`Traits/D_LivingItem.json`** — one row per weapon, `UpgradeSlots` array (2–3 choices each).
- **`LivingItems/D_LivingItemUpgrades.json`** — each choice → `AlterationToApply` →
  `Alterations/D_Alterations.json`, plus Biomass `UpgradeCost` and optional `ChallengeToUnlock`.
- **`LivingItems/D_LivingItemShopItems.json`** — the Bio Lab shop listing.

| Living Item row | Weapon item | Slot themes |
|---|---|---|
| `Legendary_Sniper` | `LegendaryWeapon_Sniper` | Barrel / Scope / Stock / Module |
| `Legendary_Bow` | `LegendaryWeapon_Bow` | (unlabeled, 4 slots) |
| `Black_Wolf` | `LegendaryWeapon_Black_Wolf_Revolver` | Barrel / Chamber / Trigger / Handle |
| `Scorpion` | `LegendaryWeapon_Scorpion_Rifle` | (unlabeled, 4 slots) |
| `Sandworm` | `LegendaryWeapon_Sandworm_Crossbow` | (unlabeled, 4 slots) |
| `Rock_Golem` | `LegendaryWeapon_GolemGauntlet` | (unlabeled, 4 slots) |
| `Ape` | `LegendaryWeapon_Ape_Club` | (unlabeled, 4 slots) |
| `Lava_Hunter` | `LegendaryWeapon_LavaHunter_FlameThrower` | (unlabeled, 4 slots) |
| `Sandwyrm_Chainsaw` | `LegendaryWeapon_Sandwyrm_Chainsaw` | (unlabeled, 4 slots) |

**Important:** a Living Item's per-upgrade mesh customization is socketed to that weapon's unique
skeleton and won't transfer to a standard weapon. The `Stats{}` on the linked `D_Alterations` row
transfers fine on its own — clone it into a standard attachment (§5), or copy `Stats` straight
into a target item's own `AdditionalStats` field (§2) for a permanent, unconditional bonus.

One stat, `FirearmScopeType_Enum`, appears only on Legendary Sniper scope variants — real,
generically-categorized (`D_Stats.json` confirms `Ranged_Weapon` category) but **UNTESTED**
outside the Living Item context; likely selects a scope-overlay UI. Test in isolation first.

---

## 10. Crafting benches & the shelter requirement *(VERIFIED)*

```json
// Traits/D_Processing.json
{ "Name": "Crafting_Bench", "DefaultRecipeSet": { "RowName": "Crafting_Bench" }, "bRequiresShelter": true, "QueueSize": 5 }
```

43+ benches use `bRequiresShelter` (including `Manufacturer`, `Polymerizer`). **Not every
deployable has a `D_Processing` row** — e.g. `Repair_Bench` doesn't, because it repairs existing
items rather than running recipes; its behavior instead routes through a Blueprint referenced in
`Deployables/D_DeployableSetup.json` (`DeployableBlueprint`, e.g. `BP_Repair_Bench.BP_Repair_Bench_C`)
— compiled into the Blueprint itself, outside data-only reach. **If a table has no row for your
target item, that's a signal to check for Blueprint-driven behavior before assuming the table is
wrong.**

---

## 11. Minimal recipe for adding a new craftable item

1. **Component tables** — add rows to whichever of `D_Itemable`, `D_Meshable`, `D_Durable`,
   `D_Decayable`, `D_Actionable`, `D_Usable`, `D_FirearmData`/`D_Armour`/etc. your item needs,
   reusing existing meshes/animations/audio wherever possible.
2. **`D_ItemsStatic`** — the hub row, pointing at all step-1 rows.
3. **`D_ItemTemplate`** — `ItemStaticData` → step-2 row. What recipes/rewards actually output.
4. **`D_Talents`** *(optional)* — unlock node, `TalentTree` + `RequiredTalents`.
5. **`D_ProcessorRecipes`** — `Inputs` from `D_ItemsStatic`, `Outputs` → step-3 row,
   `Requirement` → step-4 talent, `RecipeSets` (bench or `Character`), optional
   `Metadata.RequiredFeatureLevel`.

For an **equippable attachment**, also add §5's `D_IcarusAttachments` + `D_Alterations` pair.
For a **water/resource-connected deployable**, also add §7's `D_Resource` + `D_Water`/`D_Energy` pair.

---

## 12. Full `CurrentFile` category map (data/ folder → files)

Authoritative — verified directly against the repo's folder structure, which mirrors the
Jimk72 tool's own category system. `CurrentFile` = `"<Category>-<Filename>.json"`.

<details><summary><b>AI (27)</b></summary>

D_AICreatureType, D_AIDescriptors, D_AIEvents, D_AIGrowth, D_AIRelationships, D_AISetup,
D_AISpawnConfig, D_AISpawnRules, D_AISpawnZones, D_AutonomousSpawns, D_EpicCreatures,
D_GOAPActions, D_GOAPGoals, D_GOAPMotivations, D_GOAPProperties, D_GOAPSetup, D_GeneticLineages,
D_GeneticValues, D_GreatHuntCreatureInfo, D_HuntingClueSetup, D_HuntingSetup, D_MissionNPC,
D_Mounts, D_Saddles, D_TamedCreatureModifiers, D_Tames, D_WorldBosses
</details>

<details><summary><b>Accolades (1)</b></summary>

D_Accolades
</details>

<details><summary><b>Alterations (2)</b></summary>

D_AlterationModifiers, D_Alterations
</details>

<details><summary><b>Animation (2)</b></summary>

D_ItemAnimations, D_ItemAttachment
</details>

<details><summary><b>Armour (2)</b></summary>

D_ArmourSetBonus, D_ArmourSets
</details>

<details><summary><b>Assets (1)</b></summary>

D_AssetReferences
</details>

<details><summary><b>Attachments (3)</b></summary>

D_AttachmentIcons, D_IcarusAttachments, D_ItemAttachments
</details>

<details><summary><b>Audio (19, +MusicConditions subfolder)</b></summary>

D_AIAudioData, D_BiomeAudioData, D_BuildableAudioData, D_CharacterVoices, D_CraftingAudioData,
D_CreatureAudioThreatData, D_CriticalHitAreaAudioData, D_FirearmAudioData, D_ItemAudioData,
D_ModifierStateAudioData, D_MusicTrackStateGroups, D_MusicTracks, D_PlayerFootstepAudioData,
D_ResourceNodeAudioData, D_RiverAudioData, D_TerrainZoneAudioData, D_TreeAudioData,
D_VocalisationSettings, D_Vocalisations
</details>

<details><summary><b>Bestiary (4)</b></summary>

D_BestiaryData, D_BestiaryPoints, D_BestiaryTraitTypes, D_BestiaryTraits
</details>

<details><summary><b>Blueprints (1)</b></summary>

D_BlueprintUnlocks
</details>

<details><summary><b>Building (4)</b></summary>

D_BuildingLookup, D_BuildingPieces, D_BuildingSkins, D_BuildingStability
</details>

<details><summary><b>Challenges (1)</b></summary>

D_Challenges
</details>

<details><summary><b>Character (2)</b></summary>

D_CharacterCreationData, D_CharacterGrowth
</details>

<details><summary><b>Config (1)</b></summary>

D_GameplayConfig
</details>

<details><summary><b>Crafting (5)</b></summary>

D_CraftingModifications, D_CraftingTags, D_ExtractorRecipes, D_ProcessorRecipes, D_RecipeSets
</details>

<details><summary><b>CriticalHit (1)</b></summary>

D_CriticalHitSetup
</details>

<details><summary><b>Currency (2)</b></summary>

D_CurrencyConversions, D_MetaCurrency
</details>

<details><summary><b>DLC (1)</b></summary>

D_DLCPackageData
</details>

<details><summary><b>Damage (2)</b></summary>

D_CriticalHitAreas, D_DamageTypeInfo
</details>

<details><summary><b>Deployables (2)</b></summary>

D_DeployableSetup, D_Turret
</details>

<details><summary><b>Development (2)</b></summary>

D_FeatureLevels, D_LevelSequences
</details>

<details><summary><b>Dialogue (3)</b></summary>

D_Dialogue, D_DialoguePool, D_DialogueSpeaker
</details>

<details><summary><b>DropShip (2)</b></summary>

D_DropShipActions, D_DropShipSequences
</details>

<details><summary><b>Errors (1)</b></summary>

D_ErrorCodes
</details>

<details><summary><b>Events (1)</b></summary>

D_ScriptedEvents
</details>

<details><summary><b>Experience (1)</b></summary>

D_ExperienceEvents
</details>

<details><summary><b>FLOD (1)</b></summary>

D_FLODDescriptions
</details>

<details><summary><b>Factions (4)</b></summary>

D_FactionInfo, D_FactionMissions, D_Factions, D_MissionTypes
</details>

<details><summary><b>Farming (5)</b></summary>

D_DirtMoundModifications, D_Farmable, D_FarmingGrowthStates, D_FarmingSeeds, D_SeedModifications
</details>

<details><summary><b>FieldGuide (4)</b></summary>

D_FieldGuideCategories, D_FieldGuideMetaData, D_FieldGuideRedirect, D_FieldGuideSets
</details>

<details><summary><b>Fish (3)</b></summary>

D_FishData, D_FishSpawnConfig, D_FishSpawnZones
</details>

<details><summary><b>Flags (3)</b></summary>

D_AccountFlags, D_CharacterFlags, D_SessionFlags
</details>

<details><summary><b>GreatHunt (1)</b></summary>

D_GreatHunts
</details>

<details><summary><b>Hints (1)</b></summary>

D_Hints
</details>

<details><summary><b>Horde (2)</b></summary>

D_Horde, D_HordeWave
</details>

<details><summary><b>Input (4)</b></summary>

D_KeyIcons, D_KeybindContexts, D_Keybindings, D_Keys
</details>

<details><summary><b>InstancedMap (2)</b></summary>

D_GroupedInstancedMapData, D_InstancedMapData
</details>

<details><summary><b>Interactions (3)</b></summary>

D_Interactions, D_RadialMenuData, D_RadialOptions
</details>

<details><summary><b>Inventory (6)</b></summary>

D_BagPriority, D_InventoryContainer, D_InventoryID, D_InventoryInfo, D_ItemWeightStatQueries,
D_QuickMove
</details>

<details><summary><b>Items (6, +Types subfolder)</b></summary>

D_ItemClassificationsIcons, D_ItemRanks, D_ItemRewards, D_ItemTemplate, D_ItemTraitMasks,
D_ItemsStatic
</details>

<details><summary><b>LivingItems (2)</b></summary>

D_LivingItemShopItems, D_LivingItemUpgrades
</details>

<details><summary><b>Localization (1)</b></summary>

D_Languages
</details>

<details><summary><b>Logging (1)</b></summary>

D_LogCategories
</details>

<details><summary><b>MetaResource (1)</b></summary>

D_MetaResourceNodes
</details>

<details><summary><b>MetaWorkshop (1)</b></summary>

D_WorkshopItems
</details>

<details><summary><b>Modifiers (6)</b></summary>

D_AfflictionChance, D_ChargedModifiers, D_GrantedAuras, D_ModifierStates, D_StatAfflictions,
D_SurvivalTriggers
</details>

<details><summary><b>NationalFlags (1)</b></summary>

D_NationalFlags
</details>

<details><summary><b>Notes (1)</b></summary>

D_CollectableNotes
</details>

<details><summary><b>Online (3)</b></summary>

D_RCONCommand, D_RepGraphClassPolicies, D_RepGraphClassSettings
</details>

<details><summary><b>Orchestration (2)</b></summary>

D_OrchestrationEvents, D_OrchestrationStateFlags
</details>

<details><summary><b>Outpost (1)</b></summary>

D_Outposts
</details>

<details><summary><b>Paintings (1)</b></summary>

D_Paintings
</details>

<details><summary><b>Perks (1)</b></summary>

D_CharacterPerks
</details>

<details><summary><b>PlayerTracker (3)</b></summary>

D_PlayerAccoladeCategories, D_PlayerTrackerCategories, D_PlayerTrackers
</details>

<details><summary><b>Prebuilt (1)</b></summary>

D_PrebuiltStructures
</details>

<details><summary><b>Prospects (4)</b></summary>

D_Atmospheres, D_Biomes, D_ProspectList, D_Terrains
</details>

<details><summary><b>Quests (7, +Modifiers subfolder)</b></summary>

D_DynamicQuestRewardItems, D_DynamicQuestRewards, D_DynamicQuests, D_QuestEvents,
D_QuestQueries, D_Quests, D_StasisBag
</details>

<details><summary><b>RTXGI (1)</b></summary>

D_RTXGIVolumes
</details>

<details><summary><b>Resources (2)</b></summary>

D_IcarusResources, D_OptionalResourceFlows
</details>

<details><summary><b>Rulesets (1)</b></summary>

D_Rulesets
</details>

<details><summary><b>Scaling (1)</b></summary>

D_ScalingRules
</details>

<details><summary><b>Settings (2)</b></summary>

D_GraphicsTierDescription, D_GraphicsTierDescriptionMods
</details>

<details><summary><b>Settlement (10)</b></summary>

D_SettlementBuildings, D_SettlementEventTypes, D_SettlementEvents, D_SettlementNPCClothing,
D_SettlementNPCItems, D_SettlementNPCRoles, D_SettlementNPCSkills, D_SettlementNPCTaskTypes,
D_SettlementNPCTraits, D_SettlementRaids
</details>

<details><summary><b>Sorting (1)</b></summary>

D_SortTypePriority
</details>

<details><summary><b>Spawn (1)</b></summary>

D_ExoticSpawn
</details>

<details><summary><b>Statistics (1)</b></summary>

D_Statistics
</details>

<details><summary><b>Stats (5)</b></summary>

D_CharacterStartingStats, D_CustomGameStats, D_ProspectStats, D_StatCategories, D_Stats
</details>

<details><summary><b>Tags (2)</b></summary>

D_StatGameplayTags, D_TagQueries
</details>

<details><summary><b>Talents (8)</b></summary>

D_PlayerTalentModifiers, D_TalentArchetypes, D_TalentModelViews, D_TalentModels, D_TalentRanks,
D_TalentTrees, D_TalentViews, D_Talents
</details>

<details><summary><b>TimeOfDay (1)</b></summary>

D_TimeOfDay
</details>

<details><summary><b>Tools (10)</b></summary>

D_Actions, D_AmmoTypes, D_FirearmData, D_FirearmScopeData, D_NPCWeapon, D_RangedWeaponData,
D_StaminaActionCosts, D_ToolDamage, D_Uses, D_ValidAmmoTypes
</details>

<details><summary><b>Traits (38)</b></summary>

D_Actionable, D_Armour, D_Ballistic, D_Buildable, D_Combustible, D_Consumable, D_CrudeOil,
D_Decayable, D_Deployable, D_Durable, D_Energy, D_Equippable, D_Experience, D_Fillable,
D_Flammable, D_Floatable, D_Focusable, D_Fuel, D_Generator, D_Highlightable, D_Hitable,
D_Interactable, D_Inventory, D_Itemable, D_LivingItem, D_Meshable, D_Oxygen, D_Processing,
D_RefinedOil, D_Resource, D_Rocketable, D_Slotable, D_Thermal, D_Transmutable, D_Usable,
D_Vehicular, D_Water, D_Weight
</details>

<details><summary><b>UI (8)</b></summary>

D_CharacterTimeline, D_ContextMenuGroupTypes, D_MapIcons, D_MapSearchArea, D_PlayerIdentity,
D_PreviewCameraSettings, D_RecoveryBeacons, D_TimelineRanks
</details>

<details><summary><b>ValidHits (3)</b></summary>

D_ValidHitQueries, D_ValidHitTypes, D_ValidInteractQueries
</details>

<details><summary><b>Vehicles (1)</b></summary>

D_VehicleSetups
</details>

<details><summary><b>Weather (6)</b></summary>

D_ProspectForecast, D_WeatherActions, D_WeatherBiomeGroups, D_WeatherEvents, D_WeatherPools,
D_WeatherTierIcon
</details>

<details><summary><b>World (10)</b></summary>

D_BreakableRockData, D_DropGroups, D_FishSetup, D_OreDeposit, D_Surfaces,
D_VoxelDistributionRegion, D_VoxelMaterialMap, D_VoxelSetupData, D_WaterSetup, D_WorldData
</details>

---

## 13. Golden rules

1. **Never guess the controlling table — follow the reference.** Finding an item in
   `D_ItemsStatic` doesn't mean the value you want is stored there.
2. **Never modify a shared row without checking who else uses it.** Search the repo for the
   `RowName` first. If more than one item depends on it: clone → repoint → modify the clone.
3. **Never assume a number's unit or meaning from its name.** `Mining_Efficiency = 1.35` looks
   like "135%" but confirm in-game before documenting it as fact — same caution applies to any
   ratio/rate field.
4. **Never assume a DataTable can override compiled Blueprint behavior.** If a table has no row
   for your target (e.g. `Repair_Bench` missing from `D_Processing`), that's a strong signal the
   behavior is Blueprint-driven and out of data-only reach.
5. **Keep Standard Attachments (§5) and Living Items (§9) mentally separate** — similar-looking
   systems, not interchangeable, and stat data doesn't imply mesh/visual compatibility.
6. **Always verify a `CurrentFile` category against the repo's actual folder**, never guess by
   pattern-matching similar tables — see the `D_IcarusAttachments` example in §5.
7. **`D_ItemTemplate` for recipe outputs, `D_ItemsStatic` for recipe inputs** — this asymmetry is
   easy to get backwards.
8. **Record every real discovery as VERIFIED / INFERRED / UNTESTED** and write down the *exact*
   table/row/field, not just "it worked" — future-you needs the trail, not just the result.

## 14. Common mistakes (in order of how expensive they were to find)

- **Editing the wrong table** — the item hub rarely holds the value itself; follow the pointer.
- **Modifying a shared row** — silently changes every other item using it.
- **Guessing a `CurrentFile` category** — fails silently; the new rows just never merge into the
  real table, while everything else about the item (name, icon, craftability) looks fine.
- **Assuming a field's meaning from its name** — verify against `Defaults`, related systems, and
  in-game testing before trusting it.
- **Confusing Standard Attachments with Living Items** — different tables, different systems.
- **Assuming a `Produce`-type resource node respects its own stored quantity** — it doesn't; only
  `Consume` does. `Produce` is an unconditional tap (§7).
- **Assuming a DataTable controls Blueprint behavior** — some gameplay is compiled in, not data.

---

## 15. Practical tooling notes

- File names sometimes appear with a numeric timestamp prefix when freshly exported/uploaded
  (e.g. `1785321879462_D_FirearmData.json`) — always list the working directory before
  hardcoding a filename.
- Reliable cross-referencing pattern: load every relevant table into a dict keyed by `Name`, then
  resolve `RowName` pointers by hand rather than keeping everything in one giant structure.
- Work by subsystem (firearms, water network, talents, crafting) rather than trying to hold the
  whole database in mind at once.
- Produce mod deliverables as both a human-readable summary (what changed, why, which
  rows/tables, whether cloned) and the raw JSON row diff, so changes are reviewable before
  merging into a working mod file.
- Standard JSON has **no comment syntax** — no `//`, `/* */`, `#`, or backslash tricks. Some tool
  UIs (e.g. EXMod's "Clean up JSON") may tolerate stray text via their own preprocessor, but it's
  non-standard and risks silently corrupting a row. Keep notes in a separate file instead.

---

*This README documents structural relationships inferred and verified directly from the JSON in
this repository, cross-checked against the actual `data/` folder layout after a game update that
reorganized the export. If RocketWerkz changes a struct's fields in a future update, re-verify
against the `Defaults` block in the relevant file before relying on a field name here — and when
a new relationship is confirmed, add it here with its confidence level so the next investigation
starts from where this one ended.*