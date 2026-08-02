# Icarus Modding — DataTable Reference

This repository is a full export of *Icarus*'s (RocketWerkz) DataTable JSON files, used as
reference for **data-only modding**: creating new items, weapons, armour, recipes, and talent
unlocks by adding rows to existing tables and pointing them at existing meshes, animations,
audio, and icons — without adding any new 3D assets.

The 298 tables here are Unreal Engine `DataTable` exports. Each file has the same shell:

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

items_static = load("D_ItemsStatic.json")
itemable     = load("D_Itemable.json")
firearm      = load("D_FirearmData.json")
```

---

## 2. Anatomy of an item — the component pattern

Every item in Icarus is a `D_ItemsStatic` row acting as a hub, with each gameplay facet split
into its own component table. A row only carries the components it actually needs — a firearm
has a `FirearmData` pointer, a food item has a `Consumable` pointer, etc.

**Example — `Rifle_Hunting` in `D_ItemsStatic.json`:**

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

| Field on `D_ItemsStatic` row | Target table                | Purpose |
|---|---|---|
| `Meshable`        | `D_Meshable`        | Which 3D mesh/skeletal mesh to render (reuse an existing one — no new assets) |
| `Itemable`        | `D_Itemable`        | Display name, description, flavor text, weight, max stack, icon |
| `Interactable`    | `D_Interactable`    | World pickup interaction behavior |
| `Highlightable`   | `D_Highlightable`   | Outline/highlight visuals |
| `Focusable`       | `D_Focusable`       | Aim-down-sights / focus behavior (weapons) |
| `Actionable`      | `D_Actionable`      | What actions (fire, swing, use) the item exposes → links to `D_Actions` |
| `Usable`          | `D_Usable`          | Right-click/use behavior (repair, consume, etc.) |
| `Durable`         | `D_Durable`         | Max durability, degradation |
| `Decayable`       | `D_Decayable`       | Spoilage/decay curve |
| `Equippable`      | `D_Equippable`      | Equip slot rules (for gear/armour) |
| `Consumable`      | `D_Consumable`      | Eat/drink effects |
| `Fillable`        | `D_Fillable`        | Liquid container behavior |
| `Resource`        | `D_Resource`        | Raw resource classification |
| `Rocketable`       | `D_Rocketable`      | Craftable-into-rocket eligibility |
| `Transmutable`    | `D_Transmutable`    | Alteration bench transform rules |
| `Slotable`        | `D_Slotable`        | Attachment slot behavior |
| `ToolDamage`      | `D_ToolDamage`      | Melee/tool damage values |
| `RangedWeaponData`| `D_RangedWeaponData`| Shared ranged-weapon stats (bows, firearms) |
| `FirearmData`     | `D_FirearmData`     | Firearm-specific ballistics, recoil, fire rate, animations |
| `Armour`          | `D_Armour`          | Armour piece stats & mesh |
| `Buildable`       | `D_Buildable`       | Placeable structure behavior |
| `Deployable`      | `D_Deployable`      | Placeable device behavior |
| `Generator`       | `D_Generator`       | Power generation stats |
| `AmmoType`        | `D_AmmoTypes`       | Ammo payload behavior |
| `Ballistic`       | `D_Ballistic`       | Physical projectile behavior |
| `InventoryContainer` | `D_InventoryContainer` | Attachment/ammo sub-inventory |
| `Audio`           | (varies, e.g. `D_FirearmAudioData`, `D_ItemAudioData`) | Sound cues |

`D_ItemTemplate.json` is a thin second layer on top: each row is just
`{ "Name": "Rifle_Hunting", "ItemStaticData": { "RowName": "Rifle_Hunting" } }`. This is the
table **recipe outputs reference** (see below) — think of `D_ItemsStatic` as the "definition"
and `D_ItemTemplate` as the "craftable/spawnable instance handle."

---

## 3. Anatomy of a recipe

Recipes live in `D_ProcessorRecipes.json` (crafting benches, furnaces, etc.) or
`D_ExtractorRecipes.json` (auto-extractors). Example — `Stone_Pickaxe`:

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
- **`Requirement.RowName`** points into **`D_Talents`** — this is the exact mechanism that gates
  a recipe behind a talent-tree unlock. No `Requirement` = always craftable once the bench/`RecipeSet` is available.
- **`RecipeSets`** ties the recipe to a crafting station (see `D_RecipeSets.json` for the full
  list — `Crafting_Bench`, `Machining_Bench`, `Material_Processor`, `Fabricator`, etc.).
- **Feature-level gating** (e.g. restricting a recipe to a DLC/update) is done via
  `Metadata.RequiredFeatureLevel.RowName`, pointing into `D_FeatureLevels.json`:
  ```json
  "Metadata": { "RequiredFeatureLevel": { "RowName": "DangerousHorizons" } }
  ```

---

## 4. Anatomy of a talent unlock

`D_Talents.json` rows place a node on a talent tree UI and optionally gate it behind other
talents:

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
  see `D_TalentViews.json`, `D_TalentModels.json`, `D_TalentModelViews.json`).
- **`RequiredTalents`** is how prerequisite chains are built — a list of other `D_Talents` rows
  that must be unlocked first.
- **`ExtraData`** commonly points at the `D_Itemable` row for the icon/name shown on the talent
  node itself (separate from the recipe's own output icon).
- A recipe is linked to a talent purely through the recipe's `Requirement.RowName` matching a
  `D_Talents` row `Name` — there's no reverse pointer stored on the talent itself.

Relevant crafting-bench talent trees include `Blueprint_T1_Player` (personal crafting),
`Blueprint_T5_Manufacturer` (Manufacturer bench, tier 5), and similarly-named trees for each
bench (`Blueprint_T*_<Bench>`). Check `D_TalentTrees.json` for the full list.

---

## 5. Weapons — firearms & ranged weapons

- **`D_FirearmData.json`** — per-weapon stats: accuracy, recoil, fire rate, reload time,
  animation montage paths, `ValidAmmoTypes` pointer.
- **`D_RangedWeaponData.json`** — stats shared by all ranged weapons (bows + firearms).
- **`D_ValidAmmoTypes.json`** — named ammo *groups* (e.g. `AllRifle`, `AllArrows`), each with an
  `AmmoTypes` list of `D_ItemsStatic` rows that count as valid ammo for that group.
- **`D_AmmoTypes.json`** — ammo behavior data (projectile count, damage, accuracy spread) that
  a *specific ammo item*'s `AmmoType` component points to.
- **`D_Ballistic.json`** — physical projectile behavior (gravity, payload, trail VFX, bounce/break
  behavior) that a specific ammo item's `Ballistic` component points to.
- **`D_FirearmScopeData.json`** — scope zoom/FOV data.
- **`D_ToolDamage.json`** / **`D_ToolTypes.json`** — melee/tool weapon damage and classification.

To add a new firearm caliber: add a `D_AmmoTypes` row (damage/projectile behavior), a
`D_Ballistic` row (physics/VFX — reuse an existing bullet/arrow behavior), a `D_ItemsStatic`
item row with `AmmoType`/`Ballistic`/`Itemable`/`Meshable` pointers (reusing an existing mesh),
then add that item's row name into the relevant `D_ValidAmmoTypes` group (or make a new group
and point a `D_FirearmData` row's `ValidAmmoTypes` at it).

---

## 6. Armour

- **`D_Armour.json`** — per-piece stats (`ArmourStats`), mesh, armour type slot
  (`ArmourType`: Helmet/Chest/Legs/Undersuit/etc.), and `ImplicitDefaultArmour` (companion
  pieces auto-equipped with this one, e.g. an undersuit's built-in straps/helmet).
- **`D_ArmourSets.json`** / **`D_ArmourSetBonus.json`** — set-bonus definitions when multiple
  pieces from the same set are worn together.
- **`D_Saddles.json`** — mount saddle equivalent of armour data.

An armour item's `D_ItemsStatic` row points to its `D_Armour` row via the `Armour` component,
same pattern as `FirearmData`.

---

## 7. Feature levels (gating content behind updates/DLC)

`D_FeatureLevels.json` defines named release/update tiers (`Core`, `FirstCohort`, `Styx`,
`Homestead`, `DangerousHorizons`, `GreatHunt_Elysium`, etc.). Any table can gate a row behind
one of these via a `Metadata.RequiredFeatureLevel` (or similarly-named) pointer — confirmed on
`D_ProcessorRecipes`, and also present on `D_Talents`, `D_TalentTrees`, `D_ItemsStatic`,
`D_ItemRewards`, `D_Deployable`, `D_Buildable`, and others (grep the table you're editing for
`FeatureLevel` to find the exact field name/location for that struct).

---

## 8. Minimal recipe for adding a new craftable item

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
   `RecipeSets` pointing at the crafting bench, and optionally `Metadata.RequiredFeatureLevel`.

This is exactly the pattern used for this project's `Titanium_Assault_Rifle` mod: a
`D_ItemsStatic`/`D_Itemable`/`D_FirearmData`/`D_ItemTemplate` item, craftable via a
`D_ProcessorRecipes` entry at the Manufacturer bench, gated behind `DangerousHorizons` and a
`D_Talents` node placed in `Blueprint_T5_Manufacturer` requiring `Polymerizer` as a prerequisite.

---

## 9. Full table index by domain

<details>
<summary><b>Items — Core Identity &amp; Components (20)</b></summary>

D_Decayable, D_Durable, D_Equippable, D_Floatable, D_Focusable, D_Highlightable, D_Hitable,
D_Interactable, D_ItemAnimations, D_ItemAttachment, D_ItemAttachments,
D_ItemClassificationsIcons, D_ItemRanks, D_ItemRewards, D_ItemTemplate, D_Itemable,
D_Meshable, D_Slotable, D_Transmutable, D_Usable
</details>

<details>
<summary><b>Items — Weapons: melee, tools, turrets (4)</b></summary>

D_NPCWeapon, D_ToolDamage, D_ToolTypes, D_Turret
</details>

<details>
<summary><b>Items — Ammo &amp; Ballistics (8)</b></summary>

D_AmmoTypes, D_Ballistic, D_FirearmAudioData, D_FirearmData, D_FirearmScopeData,
D_ProjectileTypes, D_RangedWeaponData, D_ValidAmmoTypes
</details>

<details>
<summary><b>Items — Armour (4)</b></summary>

D_Armour, D_ArmourSetBonus, D_ArmourSets, D_Saddles
</details>

<details>
<summary><b>Items — Buildables &amp; Deployables (11)</b></summary>

D_Buildable, D_BuildableAudioData, D_BuildingLookup, D_BuildingPieces, D_BuildingSkins,
D_BuildingStability, D_BuildingTypes, D_Deployable, D_DeployableSetup, D_DeployableTypes,
D_SettlementBuildings
</details>

<details>
<summary><b>Items — Resources, Farming, Food (19)</b></summary>

D_Consumable, D_CrudeOil, D_Farmable, D_FarmingGrowthStates, D_FarmingSeeds, D_Fillable,
D_FishData, D_FishSetup, D_FishSpawnConfig, D_FishSpawnZones, D_FoodTypes, D_Fuel,
D_IcarusResources, D_MetaResourceNodes, D_OptionalResourceFlows, D_OreDeposit, D_RefinedOil,
D_Resource, D_ResourceNodeAudioData
</details>

<details>
<summary><b>Crafting &amp; Recipes (7)</b></summary>

D_CraftingAudioData, D_CraftingModifications, D_CraftingTags, D_ExtractorRecipes,
D_Processing, D_ProcessorRecipes, D_RecipeSets
</details>

<details>
<summary><b>Talents &amp; Progression (8)</b></summary>

D_PlayerTalentModifiers, D_TalentArchetypes, D_TalentModelViews, D_TalentModels,
D_TalentRanks, D_TalentTrees, D_TalentViews, D_Talents
</details>

<details>
<summary><b>Stats, Afflictions &amp; Damage (20)</b></summary>

D_AfflictionChance, D_CharacterGrowth, D_CharacterPerks, D_CharacterStartingStats,
D_CustomGameStats, D_DamageTypeInfo, D_Experience, D_ExperienceEvents,
D_ItemWeightStatQueries, D_ItemsStatic, D_ModifierStateAudioData, D_ModifierStates,
D_MusicTrackStateGroups, D_OrchestrationStateFlags, D_ProspectStats, D_StatAfflictions,
D_StatCategories, D_StatGameplayTags, D_Statistics, D_Stats
</details>

<details>
<summary><b>AI &amp; Creatures (30)</b></summary>

D_AIAudioData, D_AICreatureType, D_AIDescriptors, D_AIEvents, D_AIGrowth, D_AIRelationships,
D_AISetup, D_AISpawnConfig, D_AISpawnRules, D_AISpawnZones, D_BestiaryData, D_BestiaryPoints,
D_BestiaryTraitTypes, D_BestiaryTraits, D_CreatureAudioThreatData, D_EpicCreatures,
D_GreatHuntCreatureInfo, D_GreatHunts, D_Horde, D_HordeWave, D_InventoryContainer,
D_ItemTraitMasks, D_Mounts, D_Paintings, D_SettlementNPCTraits, D_SettlementRaids,
D_TamedCreatureModifiers, D_Tames, D_TerrainZoneAudioData, D_Terrains
</details>

<details>
<summary><b>Quests, Missions &amp; Factions (20)</b></summary>

D_Accolades, D_Dialogue, D_DialoguePool, D_DialogueSpeaker, D_DynamicQuestRewardItems,
D_DynamicQuestRewards, D_DynamicQuests, D_FactionInfo, D_FactionMissions, D_Factions,
D_MissionNPC, D_MissionTypes, D_MusicQuestConditions, D_PlayerAccoladeCategories,
D_QuestEnemyModifiers, D_QuestEvents, D_QuestQueries, D_QuestVocalisationModifiers,
D_QuestWeatherModifiers, D_Quests
</details>

<details>
<summary><b>Audio (6)</b></summary>

D_BiomeAudioData, D_CriticalHitAreaAudioData, D_ItemAudioData, D_PlayerFootstepAudioData,
D_RiverAudioData, D_TreeAudioData
</details>

<details>
<summary><b>World, Biomes, Environment (9)</b></summary>

D_Atmospheres, D_Biomes, D_GroupedInstancedMapData, D_InstancedMapData, D_Outposts,
D_ProspectForecast, D_ProspectList, D_Surfaces, D_WeatherBiomeGroups
</details>

<details>
<summary><b>Settlements &amp; NPCs (7)</b></summary>

D_SettlementEventTypes, D_SettlementEvents, D_SettlementNPCClothing, D_SettlementNPCItems,
D_SettlementNPCRoles, D_SettlementNPCSkills, D_SettlementNPCTaskTypes
</details>

<details>
<summary><b>UI, Input, Misc/System (124)</b></summary>

D_AccountFlags, D_Actionable, D_Actions, D_AlterationModifiers, D_Alterations,
D_AssetReferences, D_AttachmentIcons, D_AutonomousSpawns, D_BagPriority, D_BlueprintUnlocks,
D_BreakableRockData, D_Challenges, D_CharacterCreationData, D_CharacterFlags,
D_CharacterTimeline, D_CharacterVoices, D_ChargedModifiers, D_CollectableNotes,
D_Combustible, D_ContextMenuGroupTypes, D_CriticalHitAreas, D_CriticalHitSetup,
D_CurrencyConversions, D_DLCPackageData, D_DirtMoundModifications, D_DropGroups,
D_DropShipActions, D_DropShipSequences, D_Energy, D_ErrorCodes, D_ExoticSpawn,
D_FLODDescriptions, D_FeatureLevels, D_FieldGuideCategories, D_FieldGuideMetaData,
D_FieldGuideRedirect, D_FieldGuideSets, D_Flammable, D_GOAPActions, D_GOAPGoals,
D_GOAPMotivations, D_GOAPProperties, D_GOAPSetup, D_GameplayConfig, D_Generator,
D_GeneticLineages, D_GeneticValues, D_GrantedAuras, D_GraphicsTierDescription,
D_GraphicsTierDescriptionMods, D_Hints, D_HuntingClueSetup, D_HuntingSetup,
D_IcarusAttachments, D_Interactions, D_Inventory, D_InventoryID, D_InventoryInfo,
D_KeyIcons, D_KeybindContexts, D_Keybindings, D_Keys, D_Languages, D_LevelSequences,
D_LivingItem, D_LivingItemShopItems, D_LivingItemUpgrades, D_LogCategories, D_MapIcons,
D_MapSearchArea, D_MetaCurrency, D_MusicLocationConditions, D_MusicTracks,
D_NationalFlags, D_OrchestrationEvents, D_Oxygen, D_PlayerIdentity,
D_PlayerTrackerCategories, D_PlayerTrackers, D_PrebuiltStructures,
D_PreviewCameraSettings, D_QuickMove, D_RCONCommand, D_RTXGIVolumes, D_RadialMenuData,
D_RadialOptions, D_RecoveryBeacons, D_RepGraphClassPolicies, D_RepGraphClassSettings,
D_Rocketable, D_Rulesets, D_ScalingRules, D_ScriptedEvents, D_SeedModifications,
D_SessionFlags, D_SortTypePriority, D_StaminaActionCosts, D_StasisBag,
D_SurvivalTriggers, D_TagQueries, D_Thermal, D_TimeOfDay, D_TimelineRanks, D_Uses,
D_ValidHitQueries, D_ValidHitTypes, D_ValidInteractQueries, D_VehicleSetups, D_Vehicular,
D_VocalisationSettings, D_Vocalisations, D_VoxelDistributionRegion, D_VoxelMaterialMap,
D_VoxelSetupData, D_Water, D_WaterSetup, D_WeatherActions, D_WeatherEvents,
D_WeatherPools, D_WeatherTierIcon, D_Weight, D_WorkshopItems, D_WorldBosses, D_WorldData
</details>

---

## 10. Practical tooling notes

- File names sometimes appear with a numeric timestamp prefix when freshly exported/uploaded
  (e.g. `1785321879462_D_FirearmData.json`) — always `ls`/list the working directory before
  hardcoding a filename.
- A reliable pattern for cross-referencing: load every relevant table into a dict keyed by
  `Name`, then resolve `RowName` pointers by hand rather than trying to keep everything in one
  giant structure.
- Work in batches: pull all the tables relevant to one subsystem (e.g. firearms + ammo + valid
  ammo groups) together and cross-check consistency before writing new rows.
- Produce mod deliverables in both a human-readable Markdown summary and the raw JSON row
  diffs/additions, so changes are easy to review before merging into a working `.pak`.

---

*This README documents structural relationships inferred directly from the JSON in this
repository as of the current upload. If RocketWerkz changes a struct's fields in a future game
update, re-verify against the `Defaults` block in the relevant file before relying on a field
name here.*
