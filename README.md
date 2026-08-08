# Icarus Modding — DataTable Reference

This repository is a full export of *Icarus*'s (RocketWerkz) DataTable JSON files, used as
reference for **data-only modding**: creating new items, weapons, armour, recipes, and talent
unlocks by adding rows to existing tables and pointing them at existing meshes, animations,
audio, and icons — without adding any new 3D assets.

All 298 tables live under `data/`, organized into category folders (e.g. `data/Items/`,
`data/Traits/`, `data/Alterations/`). This structure is the same one used by the
[Jimk72/Icarus_Software](https://github.com/Jimk72/Icarus_Software) modding tool's own file
browser — the folder a table sits in **is** the `CurrentFile` category that tool expects when
you build a mod (`"CurrentFile": "Items-D_ItemsStatic.json"`, etc.). Section 10 has the full
category map.

Each table has the same shell:

```json
{
  "RowStruct": "/Script/Icarus.SomeStructType",
  "Defaults": { /* default values for every field a row can have */ },
  "Columns":  [ /* metadata about key columns, mostly irrelevant for modding */ ],
  "Rows": [
    { "Name": "UniqueRowName", "...": "..." },
    { "Name": "AnotherRow", "...": "..." }
  ]
}
```

Modding is almost entirely about **adding new entries to `Rows`** in the right combination of
tables, using the row's `Name` as the primary key other tables reference.

---

## 1. The core reference pattern

Tables reference each other with small pointer objects instead of embedding data. There are two
shapes you'll see everywhere:

```json
"SomeField": { "RowName": "TargetRowName" }
```
```json
"SomeField": { "RowName": "TargetRowName", "DataTableName": "D_SomeTable" }
```

- If `DataTableName` is present, it tells you exactly which table to look in.
- If it's omitted, the target table is implied by the field itself (e.g. `TalentTree` on a
  `D_Talents` row always points into `D_TalentTrees`, `Requirement` always points into
  `D_Talents`, etc.) — check that field's `Defaults` entry in the same file to confirm which
  table it defaults to.

The practical workflow is: **load every table you need into a Python dict keyed by `Name`,
then follow the `RowName` pointers by hand.**

```python
import json

def load(path):
    with open(path) as f:
        return {r["Name"]: r for r in json.load(f)["Rows"]}

items_static = load("data/Items/D_ItemsStatic.json")
itemable     = load("data/Traits/D_Itemable.json")
firearm      = load("data/Tools/D_FirearmData.json")
```

---

## 2. Anatomy of an item — the component pattern

Every item in Icarus is a `D_ItemsStatic` row acting as a hub, with each gameplay facet split
into its own component table. A row only carries the components it actually needs — a firearm
has a `FirearmData` pointer, a food item has a `Consumable` pointer, etc.

**Example — `Rifle_Hunting` in `data/Items/D_ItemsStatic.json`:**

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
  "Manual_Tags":       { "GameplayTags": [ { "TagName": "Item.Weapon.Firearm.Rifle" } ] },
  "Generated_Tags":    { "GameplayTags": [ /* auto-derived from which components exist */ ] }
}
```

Note the pointer's `RowName` does **not** have to match the parent item's `Name` — e.g.
`Itemable` points to `Item_Rifle_Hunting`, not `Rifle_Hunting`. This lets multiple items reuse
the same display/description/weight data.

### Component tables you'll wire into a `D_ItemsStatic` row

| Field on `D_ItemsStatic` row | Target table (folder)              | Purpose |
|---|---|---|
| `Meshable`        | `Traits/D_Meshable`        | Which 3D mesh/skeletal mesh to render (reuse an existing one — no new assets) |
| `Itemable`        | `Traits/D_Itemable`        | Display name, description, flavor text, weight, max stack, icon |
| `Interactable`    | `Traits/D_Interactable`    | World pickup interaction behavior |
| `Highlightable`   | `Traits/D_Highlightable`   | Outline/highlight visuals |
| `Focusable`       | `Traits/D_Focusable`       | Aim-down-sights / focus behavior (weapons) |
| `Actionable`      | `Traits/D_Actionable`      | What actions (fire, swing, use) the item exposes → links to `Tools/D_Actions` |
| `Usable`          | `Traits/D_Usable`          | Right-click/use behavior (repair, consume, etc.) |
| `Durable`         | `Traits/D_Durable`         | Max durability, degradation |
| `Decayable`       | (in `Traits/D_Decayable`)  | Spoilage/decay curve |
| `Equippable`      | `Traits/D_Equippable`      | Equip slot rules (for gear/armour) |
| `Consumable`      | `Traits/D_Consumable`      | Eat/drink effects |
| `Fillable`        | `Traits/D_Fillable`        | Liquid container behavior |
| `Resource`        | `Traits/D_Resource`        | Raw resource classification |
| `Rocketable`      | `Traits/D_Rocketable`      | Craftable-into-rocket eligibility |
| `Transmutable`    | `Traits/D_Transmutable`    | Incinerator fuel-conversion value (NOT weapon alterations — see §4 note) |
| `Slotable`        | `Traits/D_Slotable`        | World-socket item slotting (fuel/water tanks, oxygen refills, skinning) |
| `ToolDamage`      | `Tools/D_ToolDamage`       | Melee/tool damage values |
| `RangedWeaponData`| `Tools/D_RangedWeaponData` | Shared ranged-weapon stats (bows, firearms) |
| `FirearmData`     | `Tools/D_FirearmData`      | Firearm-specific ballistics, recoil, fire rate, animations |
| `Armour`          | `Traits/D_Armour`          | Armour piece stats & mesh |
| `Buildable`       | `Traits/D_Buildable`       | Placeable structure behavior |
| `Deployable`      | `Traits/D_Deployable`      | Placeable device behavior |
| `Generator`       | `Traits/D_Generator`       | Power generation stats |
| `AmmoType`        | `Tools/D_AmmoTypes`        | Ammo payload behavior |
| `Ballistic`       | `Traits/D_Ballistic`       | Physical projectile behavior |
| `InventoryContainer` | `Inventory/D_InventoryContainer` | Attachment/ammo sub-inventory |
| `Attachments`     | `Attachments/D_IcarusAttachments` | Links an *attachment item* to the `Alterations/D_Alterations` stat bonus it grants when equipped |
| `Processing`      | `Traits/D_Processing`      | Crafting-bench behavior (recipe queue, shelter requirement — see §7) |
| `LivingItem`       | `Traits/D_LivingItem`      | Marks item as a Great Hunt "Living Weapon" with its own upgrade-slot system (see §8) |
| `Audio`           | (varies, e.g. `Audio/D_FirearmAudioData`, `Audio/D_ItemAudioData`) | Sound cues |

`data/Items/D_ItemTemplate.json` is a thin second layer on top: each row is just
`{ "Name": "Rifle_Hunting", "ItemStaticData": { "RowName": "Rifle_Hunting" } }`. This is the
table **recipe outputs reference** (see below) — think of `D_ItemsStatic` as the "definition"
and `D_ItemTemplate` as the "craftable/spawnable instance handle."

---

## 3. Anatomy of a recipe

Recipes live in `data/Crafting/D_ProcessorRecipes.json` (crafting benches, furnaces, etc.) or
`data/Crafting/D_ExtractorRecipes.json` (auto-extractors). Example — `Stone_Pickaxe`:

```json
{
  "Name": "Stone_Pickaxe",
  "Requirement": { "RowName": "Stone_Pickaxe" },
  "RecipeSets": [ { "RowName": "Character", "DataTableName": "D_RecipeSets" } ],
  "Inputs": [
    { "Element": { "RowName": "Fiber", "DataTableName": "D_ItemsStatic" }, "Count": 10 },
    { "Element": { "RowName": "Stick", "DataTableName": "D_ItemsStatic" }, "Count": 4 },
    { "Element": { "RowName": "Stone", "DataTableName": "D_ItemsStatic" }, "Count": 6 }
  ],
  "Outputs": [
    { "Element": { "RowName": "Stone_Pickaxe", "DataTableName": "D_ItemTemplate" }, "Count": 1 }
  ],
  "Audio": { "RowName": "Default" }
}
```

Key points:
- **`Inputs.Element`** always points into **`D_ItemsStatic`** (raw material cost).
- **`Outputs.Element`** always points into **`D_ItemTemplate`** (the craftable result).
- **`Requirement.RowName`** points into **`data/Talents/D_Talents.json`** — this is the exact
  mechanism that gates a recipe behind a talent-tree unlock. No `Requirement` = always craftable
  once the bench/`RecipeSet` is available.
- **`RecipeSets`** ties the recipe to a crafting station — see `data/Crafting/D_RecipeSets.json`
  for the full list (`Character` = personal crafting menu, no bench needed; plus
  `Crafting_Bench`, `Machining_Bench`, `Alteration_Bench`, `Fabricator`, etc.).
- **Feature-level gating** (e.g. restricting a recipe to a DLC/update) is done via
  `Metadata.RequiredFeatureLevel.RowName`, pointing into `data/Development/D_FeatureLevels.json`:
  ```json
  "Metadata": { "RequiredFeatureLevel": { "RowName": "DangerousHorizons" } }
  ```

---

## 4. Anatomy of a talent unlock

`data/Talents/D_Talents.json` rows place a node on a talent tree UI and optionally gate it
behind other talents:

```json
{
  "Name": "Stone_Axe",
  "ExtraData":   { "RowName": "Item_Stone_Axe", "DataTableName": "D_Itemable" },
  "TalentTree":  { "RowName": "Blueprint_T1_Player" },
  "Position":    { "X": 500, "Y": 850 },
  "Size":        { "X": 184, "Y": 184 },
  "RequiredTalents": [
    { "RowName": "Stone_Tools_Rerout", "DataTableName": "D_Talents" }
  ],
  "bDefaultUnlocked": true,
  "DrawMethodOverride": "YThenX"
}
```

- **`TalentTree.RowName`** points into `D_TalentTrees.json`, which in turn has an `Archetype`
  pointer into `D_TalentArchetypes.json` (defines which talent *view*/UI panel it renders in —
  see `D_TalentViews.json`, `D_TalentModels.json`, `D_TalentModelViews.json`, all in the same
  `Talents/` folder).
- **`RequiredTalents`** is how prerequisite chains are built — a list of other `D_Talents` rows
  that must be unlocked first.
- **`ExtraData`** commonly points at the `D_Itemable` row for the icon/name shown on the talent
  node itself (separate from the recipe's own output icon).
- A recipe is linked to a talent purely through the recipe's `Requirement.RowName` matching a
  `D_Talents` row `Name` — there's no reverse pointer stored on the talent itself.

Relevant crafting-bench talent trees include `Blueprint_T1_Player` (personal crafting),
`Blueprint_T5_Manufacturer` (Manufacturer bench, tier 5), and similarly-named trees for each
bench (`Blueprint_T*_<Bench>`). Check `D_TalentTrees.json` for the full list.

> ⚠️ **Naming trap:** `Traits/D_Transmutable.json` sounds like it might be the "weapon
> alteration" system — it is **not**. It's the incinerator's fuel-conversion table (`Wood`,
> `Stick`, `Biofuel`, etc.). The actual weapon-attachment stat system is
> `Attachments/D_IcarusAttachments.json` → `Alterations/D_Alterations.json` (see §5).

---

## 5. Weapon attachments (the equippable-item stat system)

Standard equippable attachments (scopes, damage mods, accuracy mods, etc.) use a three-table
chain — this is **separate** from the Living Item system in §8:

```
D_ItemsStatic (attachment item)
  → Attachments.RowName → data/Attachments/D_IcarusAttachments.json row
      → GrantedAlteration.RowName → data/Alterations/D_Alterations.json row (the Stats{})
```

Example — `Rifle_Attachment_Accuracy_1`:

```json
// data/Items/D_ItemsStatic.json
{
  "Name": "Rifle_Attachment_Accuracy_1",
  "Meshable": { "RowName": "Mesh_Attachment_Ranged_Weapon" },
  "Itemable": { "RowName": "Item_Attachment_Rifle_Accuracy_1" },
  "Attachments": { "RowName": "Rifle_Accuracy_1" },
  "Manual_Tags": { "GameplayTags": [
    { "TagName": "Item.Attachment.Rank.1" },
    { "TagName": "Item.Attachment.Rifle" }
  ]}
}

// data/Attachments/D_IcarusAttachments.json
{ "Name": "Rifle_Accuracy_1", "GrantedAlteration": { "RowName": "Rifle_Accuracy_1" } }

// data/Alterations/D_Alterations.json
{
  "Name": "Rifle_Accuracy_1",
  "Stats": {
    "(Value=\"BaseRifleProjectileAccuracy_+%\")": 20,
    "(Value=\"BaseRifleCriticalDamage_+%\")": 15
  }
}
```

All standard rifle/pistol/bow/shotgun/crossbow attachments share **one generic mesh**
(`Mesh_Attachment_Ranged_Weapon`) — there's no need for a unique model per attachment, only a
unique icon. The `Item.Attachment.<WeaponClass>` tag (`Rifle`, `Pistol`, `Shotgun`, `Bow`,
`Crossbow`) is what makes an item fit into that weapon class's attachment slot (see §6).

Attachments are crafted at the `Alteration_Bench` / `Advanced_Alteration_Bench` recipe sets (or
`Character` if you want them craftable without a bench) — the bench name is *only* the crafting
station, it doesn't change how the item gets installed (still via the slot system in §6).

---

## 6. Weapon attachment **slots** (how many an item can hold)

A weapon's slot count is controlled by a separate two-table chain from the attachment item
itself:

```
D_ItemsStatic.InventoryContainer.RowName
    → data/Inventory/D_InventoryContainer.json row
        → InventoryInfo.RowName → data/Inventory/D_InventoryInfo.json row
            → StartingSlots (total slot count)
            → SlotOverrides (pins specific slot indices to a specific D_TagQueries filter,
                              e.g. index 0 = ammo-only)
            → SlotTemplate (fallback filter for any slot index not covered by SlotOverrides)
```

Example — `Rifle_Ammo_Attachment` (shared by every tier-1–3 rifle):
```json
// D_InventoryContainer.json
{ "Name": "Rifle_Ammo_Attachment", "InventoryInfo": { "RowName": "Rifle_Ammo_Attachment" }, "AttachmentSlot": 1 }

// D_InventoryInfo.json
{
  "Name": "Rifle_Ammo_Attachment",
  "SlotTemplate": { "RowName": "Any_Rifle_Attachment" },
  "StartingSlots": 2,
  "SlotOverrides": [ { "Query": { "RowName": "Any_Ammo", "DataTableName": "D_TagQueries" }, "Location": 0 } ]
}
```
2 slots = 1 ammo (index 0, overridden) + 1 generic attachment (index 1, falls back to
`Any_Rifle_Attachment`, which matches anything tagged `Item.Attachment.Rifle`).

⚠️ **This container is shared** across every weapon that references it by name (e.g. both
`Rifle_Hunting` and `Rifle_BoltAction` use `Rifle_Ammo_Attachment`). Editing it affects every
weapon using it. To change slots for **one** weapon only, clone both rows under a new name and
repoint that weapon's `InventoryContainer` at the clone — don't edit the shared vanilla row.

No vanilla weapon in the dataset exceeds `StartingSlots: 2`.

---

## 7. Crafting benches & the shelter requirement

`data/Traits/D_Processing.json` is what turns a deployable into a recipe-crafting station:

```json
{
  "Name": "Crafting_Bench",
  "DefaultRecipeSet": { "RowName": "Crafting_Bench" },
  "bRequiresShelter": true,
  "QueueSize": 5
}
```

`bRequiresShelter` is the real, data-driven shelter gate — 43+ benches use it (including
`Manufacturer` and `Polymerizer`). **Not every deployable has a `D_Processing` row** — e.g.
`Repair_Bench` doesn't, because it repairs existing items rather than running crafting recipes;
its interaction instead routes through Blueprint classes referenced in
`data/Deployables/D_DeployableSetup.json` (`DeployableBlueprint` field, e.g.
`BP_Repair_Bench.BP_Repair_Bench_C`) — behavior compiled into the Blueprint itself, outside the
reach of data-only JSON modding.

---

## 8. Living Items (Great Hunt legendary weapons)

A separate system from §5/§6, used only by the 9 named legendary weapons dropped from Great
Hunt bosses. Driven by three tables, all outside the standard item/attachment tables:

- **`data/Traits/D_LivingItem.json`** — one row per legendary weapon, with an `UpgradeSlots`
  array (each slot lists 2–3 alternative upgrade choices).
- **`data/LivingItems/D_LivingItemUpgrades.json`** — each upgrade choice, pointing via
  `AlterationToApply` into `Alterations/D_Alterations.json` for its actual stat bonus, plus a
  Biomass `UpgradeCost` and optional `ChallengeToUnlock`.
- **`data/LivingItems/D_LivingItemShopItems.json`** — the Bio Lab shop listing (purchase gating,
  background art, required account flags/DLC package).

| Living Item row | Weapon item (`D_ItemsStatic`) | Slot themes |
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

**Important:** a Living Item's mesh customizations per upgrade (e.g.
`SM_GUN_Sniper_LGD_Scope_C`) are socketed to that weapon's unique skeleton and won't transfer to
a standard weapon. The `Stats{}` on the linked `D_Alterations` row transfer fine on their own,
though — either by cloning the alteration into a new standard attachment (§5), or by copying the
`Stats` block straight into a target item's own `AdditionalStats` field (§2 example) for a
permanent, unconditional bonus with no crafting/slot involved.

One `D_Alterations` stat, `FirearmScopeType_Enum`, appears only on Legendary Sniper scope
variants — it's a real, generically-categorized stat (`D_Stats.json` confirms `Ranged_Weapon`
category) but untested outside the Living Item context; likely selects a scope-overlay UI. Worth
testing in isolation before relying on it elsewhere.

---

## 9. Minimal recipe for adding a new craftable item

1. **Component tables**: add rows to whichever of `D_Itemable`, `D_Meshable`, `D_Durable`,
   `D_Decayable`, `D_Actionable`, `D_Usable`, `D_FirearmData`/`D_Armour`/etc. your item needs —
   reusing existing meshes/animations/audio row names wherever possible.
2. **`D_ItemsStatic`**: add the hub row, pointing at all the component rows from step 1.
3. **`D_ItemTemplate`**: add a row with `ItemStaticData` pointing at the step-2 row — this is
   what recipes and reward tables actually output.
4. **`D_Talents`** (optional): add an unlock node, wired into a `TalentTree`, with
   `RequiredTalents` for any prerequisite.
5. **`D_ProcessorRecipes`**: add the recipe — `Inputs` from `D_ItemsStatic`, `Outputs` pointing
   to the step-3 `D_ItemTemplate` row, `Requirement` pointing to the step-4 talent (if any),
   `RecipeSets` pointing at the crafting bench (or `Character` for no-bench crafting), and
   optionally `Metadata.RequiredFeatureLevel`.

For an **equippable attachment item** specifically, also add steps from §5 (`D_IcarusAttachments`
+ `D_Alterations`) instead of/alongside a weapon-specific component.

---

## 10. Full `CurrentFile` category map (data/ folder → files)

This is the authoritative list — verified directly against the repo's folder structure, which
mirrors the Jimk72 Icarus_Software tool's own category system. When building a mod file for that
tool, the `CurrentFile` value is `"<Category>-<Filename>.json"` using the category (folder) name
below.

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

## 11. Practical tooling notes

- **Always verify a table's category against this repo's folder structure before guessing a
  `CurrentFile` value** — a wrong category silently fails to merge the new rows into the real
  table (confirmed root cause of a "missing stats" bug on a custom attachment mod: guessed
  `Traits-D_IcarusAttachments.json`, real category is `Attachments-`).
- File names sometimes appear with a numeric timestamp prefix when freshly exported/uploaded
  (e.g. `1785321879462_D_FirearmData.json`) — always `ls`/list the working directory before
  hardcoding a filename.
- A reliable pattern for cross-referencing: load every relevant table into a dict keyed by
  `Name`, then resolve `RowName` pointers by hand rather than trying to keep everything in one
  giant structure.
- Work in batches: pull all the tables relevant to one subsystem (e.g. firearms + ammo + valid
  ammo groups) together and cross-check consistency before writing new rows.
- Produce mod deliverables in both a human-readable Markdown summary and the raw JSON row
  diffs/additions, so changes are easy to review before merging into a working mod file.
- Standard JSON has **no comment syntax** — no `//`, `/* */`, `#`, or backslash tricks. Some
  tool UIs (including EXMod's "Clean up JSON") may tolerate stray text via their own
  preprocessor, but it's non-standard and risks silently corrupting a row if the tool's
  stripping logic has edge cases. Keep notes in a separate file instead.

---

*This README documents structural relationships inferred directly from the JSON in this
repository, cross-checked against the actual `data/` folder layout after a recent game update
that reorganized the export. If RocketWerkz changes a struct's fields in a future update,
re-verify against the `Defaults` block in the relevant file before relying on a field name here.*