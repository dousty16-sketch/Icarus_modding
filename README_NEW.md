# ICARUS Modding — Master DataTable Reference

A practical reference for **data-only ICARUS modding** using the exported DataTables contained in this repository.

This repository is intended to serve two purposes:

1. A reference for human ICARUS mod development.
2. A structured knowledge base for tracing ICARUS DataTable relationships when designing and debugging mods.

The goal is not simply to list the game's JSON files. The important information is **how the tables connect to one another**.

---

# 1. Dataset Information

This repository contains an export of ICARUS DataTable JSON files used for data-only modding.

The current repository contains:

- **298 DataTables**
- Tables organized into functional categories under `data/`
- `D_ItemsStatic.json` as the central hub for item definitions
- Component tables for individual item behaviours
- Crafting, talent, inventory, attachment, weapon, armour, creature, world and gameplay tables

The repository structure also corresponds to the `CurrentFile` naming convention used by ICARUS modding tools such as Icarus Mod Manager / Icarus Software.

### Important

This repository represents a particular ICARUS data snapshot.

Do **not** assume that a relationship or field documented here will remain unchanged after a game update.

When ICARUS receives a major update:

1. Check the affected JSON table.
2. Check its `Defaults` block.
3. Check whether the referenced row still exists.
4. Check whether the field type or structure changed.
5. Re-test important behaviour in-game.

---

# 2. Documentation Confidence Levels

Every discovery should be classified using one of these levels.

### VERIFIED

The relationship is directly visible in the JSON structure and/or has been confirmed through working mod behaviour.

### INFERRED

The relationship is strongly supported by:

- RowName references
- Defaults
- repeated usage
- naming
- cross-table relationships

but has not yet been independently confirmed in-game.

### UNTESTED

The JSON relationship exists, but its actual gameplay effect has not yet been confirmed.

### IMPORTANT RULE

Do not turn an inferred or untested assumption into a documented fact without testing it.

---

# 3. Core ICARUS DataTable Architecture

ICARUS DataTables generally use this structure:

```text
Table
├── RowStruct
├── Defaults
├── Columns
└── Rows[]
```

Individual rows are identified by:

```json
{
    "Name": "Example_Row"
}
```

Other tables normally reference that row using:

```json
{
    "RowName": "Example_Row"
}
```

or:

```json
{
    "RowName": "Example_Row",
    "DataTableName": "D_ExampleTable"
}
```

### Explicit table reference

When `DataTableName` is present, it identifies the target table directly.

### Implicit table reference

When `DataTableName` is omitted, the field's struct/default definition determines the expected target table.

Never assume an omitted `DataTableName` merely from the field name.

Check the `Defaults` definition and existing rows.

---

# 4. The Most Important Rule: Follow the References

Do not treat a JSON table as an isolated file.

When investigating something, follow the chain.

For example:

```text
Item
 ↓
D_ItemsStatic
 ↓
Component reference
 ↓
Component table
 ↓
Additional reference
 ↓
Another DataTable
```

The correct table to modify is often **not** the table where the item was originally found.

---

# 5. The Item Hub — D_ItemsStatic

`D_ItemsStatic` is the central item-definition table.

A typical item looks conceptually like:

```text
D_ItemsStatic
│
├── Meshable
├── Itemable
├── Interactable
├── Focusable
├── Highlightable
├── Actionable
├── Usable
├── Durable
├── Decayable
├── Equippable
├── Consumable
├── InventoryContainer
├── ToolDamage
├── RangedWeaponData
├── FirearmData
├── Armour
├── Attachments
├── Processing
├── LivingItem
└── other components
```

An item only contains the components required by that item.

For example, a firearm can have:

```text
D_ItemsStatic
 ├── FirearmData
 ├── RangedWeaponData
 ├── ToolDamage
 ├── InventoryContainer
 └── Actionable
```

while food may instead use:

```text
D_ItemsStatic
 ├── Itemable
 ├── Consumable
 ├── Decayable
 └── Weight
```

---

# 6. D_ItemsStatic Component Reference

| D_ItemsStatic field | Target | Purpose |
|---|---|---|
| `Meshable` | `D_Meshable` | Mesh/visual representation |
| `Itemable` | `D_Itemable` | Name, description, icon, weight, stack information |
| `Interactable` | `D_Interactable` | Interaction behaviour |
| `Highlightable` | `D_Highlightable` | Highlight/outline behaviour |
| `Focusable` | `D_Focusable` | Focus/ADS behaviour |
| `Actionable` | `D_Actionable` | Actions available to the item |
| `Usable` | `D_Usable` | Use/repair/consume behaviour |
| `Durable` | `D_Durable` | Durability |
| `Decayable` | `D_Decayable` | Spoilage/decay |
| `Equippable` | `D_Equippable` | Equipment rules |
| `Consumable` | `D_Consumable` | Food/drink effects |
| `Fillable` | `D_Fillable` | Liquid-container behaviour |
| `Resource` | `D_Resource` | Resource classification |
| `Rocketable` | `D_Rocketable` | Rocket-related eligibility |
| `Transmutable` | `D_Transmutable` | Incinerator/fuel conversion |
| `Slotable` | `D_Slotable` | World socket/slot behaviour |
| `ToolDamage` | `D_ToolDamage` | Tool/melee damage and mining-related values |
| `RangedWeaponData` | `D_RangedWeaponData` | Ranged weapon behaviour |
| `FirearmData` | `D_FirearmData` | Firearm-specific behaviour |
| `Armour` | `D_Armour` | Armour-specific data |
| `Buildable` | `D_Buildable` | Building behaviour |
| `Deployable` | `D_Deployable` | Deployable behaviour |
| `Generator` | `D_Generator` | Power-generation behaviour |
| `AmmoType` | `D_AmmoTypes` | Ammunition behaviour |
| `Ballistic` | `D_Ballistic` | Projectile behaviour |
| `InventoryContainer` | `D_InventoryContainer` | Internal inventory/attachment slots |
| `Attachments` | `D_IcarusAttachments` | Attachment stat system |
| `Processing` | `D_Processing` | Crafting-station behaviour |
| `LivingItem` | `D_LivingItem` | Great Hunt legendary weapon system |
| `Audio` | **Needs verification** | Audio configuration |

These relationships should be verified against the current JSON data before being treated as permanent documentation.

---

# 7. D_Itemable

`D_Itemable` is one of the most important component tables.

It commonly controls item presentation and basic item properties, including things such as:

- Display name
- Description
- Icon
- Weight
- Stack behaviour
- Item classification information

### Important

Do not assume every property commonly associated with an "item" is stored in `D_Itemable`.

Check the row and its `Defaults` definition.

---

# 8. D_ItemTemplate

`D_ItemTemplate` is a thin layer connecting an item template to its `D_ItemsStatic` definition.

Typical relationship:

```text
D_ItemTemplate
    │
    └── ItemStaticData
             │
             └── D_ItemsStatic
```

Example:

```json
{
    "Name": "Stone_Pickaxe",
    "ItemStaticData": {
        "RowName": "Stone_Pickaxe"
    }
}
```

### Important distinction

Think of:

```text
D_ItemsStatic
```

as the item's definition.

Think of:

```text
D_ItemTemplate
```

as the item handle used by systems such as recipes and rewards.

---

# 9. Recipe Architecture

Crafting recipes are primarily found in:

```text
Crafting/D_ProcessorRecipes.json
```

Automatic extraction recipes are found in:

```text
Crafting/D_ExtractorRecipes.json
```

A normal processor recipe follows this pattern:

```text
D_ProcessorRecipes
│
├── Requirement
│      ↓
│   D_Talents
│
├── RecipeSets
│      ↓
│   D_RecipeSets
│
├── Inputs
│      ↓
│   D_ItemsStatic
│
└── Outputs
       ↓
    D_ItemTemplate
```

---

# 10. D_ProcessorRecipes

A recipe can contain:

- `Requirement`
- `RecipeSets`
- `Inputs`
- `Outputs`

The key cross-references are:

`Inputs.Element`

→ `D_ItemsStatic`

`Outputs.Element`

→ `D_ItemTemplate`

`Requirement.RowName`

→ `D_Talents`

`RecipeSets`

→ `D_RecipeSets`

### Example

```json
{
    "Name": "Stone_Pickaxe",
    "Requirement": {
        "RowName": "Stone_Pickaxe"
    },
    "RecipeSets": [
        {
            "RowName": "Character",
            "DataTableName": "D_RecipeSets"
        }
    ]
}
```

---

# 11. Making an Item Character-Craftable

To make a recipe available directly to the player:

```text
D_ProcessorRecipes
    ↓
RecipeSets
    ↓
Character
```

Example:

```json
"RecipeSets": [
    {
        "RowName": "Character",
        "DataTableName": "D_RecipeSets"
    }
]
```

`Character` is a RecipeSet. It is not the same thing as the talent system.

Talent requirements are controlled separately through:

```text
Requirement
```

---

# 12. Crafting Stations

`D_RecipeSets` determines which crafting context a recipe belongs to.

Examples may include:

```text
Character
Crafting_Bench
Carpentry_Bench
Carpentry_Bench_T4
Fabricator
Machining_Bench
Manufacturer
Alteration_Bench
Advanced_Alteration_Bench
```

The complete list must always be checked against the current `D_RecipeSets.json`.

---

# 13. Recipe Talent Requirements

Recipe gating uses:

```text
D_ProcessorRecipes.Requirement
        ↓
D_Talents
```

For example:

```json
"Requirement": {
    "RowName": "Stone_Pickaxe"
}
```

The corresponding `D_Talents` row controls the talent-side unlock.

### Important

The recipe points to the talent. Do not assume the talent contains a reverse pointer to the recipe.

---

# 14. Talent Architecture

Relevant talent tables include:

```text
D_Talents
D_TalentTrees
D_TalentArchetypes
D_TalentViews
D_TalentModels
D_TalentModelViews
D_TalentRanks
D_PlayerTalentModifiers
```

Typical relationship:

```text
D_Talents
│
├── TalentTree
│       ↓
│   D_TalentTrees
│
├── RequiredTalents
│       ↓
│   D_Talents
│
└── other talent data
```

`RequiredTalents` creates prerequisite chains.

---

# 15. Weapon and Tool Damage

Tools and weapons commonly connect through:

```text
D_ItemsStatic
      ↓
ToolDamage
      ↓
D_ToolDamage
```

Relevant values can include:

```text
Melee_Damage
DamageVariationPercentage
Mining_Radius
Mining_Efficiency
```

### Important status

Do not automatically interpret a numeric field as a percentage or multiplier from its name alone.

For example:

```text
Mining_Efficiency = 1.350000023841858
```

may behave like approximately 135%, but this must be confirmed through gameplay testing before being documented as a fact.

Do not assume:

```text
2.0 = 200%
```

until verified in-game.

---

# 16. Weapon Architecture

A weapon may combine several systems:

```text
D_ItemsStatic
│
├── ToolDamage
├── RangedWeaponData
├── FirearmData
├── Actionable
├── Focusable
├── InventoryContainer
├── Durable
├── Decayable
└── Audio
```

### Firearms

Typical chain:

```text
D_ItemsStatic
    ↓
FirearmData
    ↓
D_FirearmData
```

Additional firearm systems may include:

```text
D_RangedWeaponData
D_FirearmScopeData
D_AmmoTypes
D_ValidAmmoTypes
D_Actions
D_Uses
```

Do not modify a firearm based on one table alone. Investigate the complete weapon chain.

---

# 17. Attachments

Standard weapon attachments use:

```text
D_ItemsStatic
    ↓
Attachments
    ↓
D_IcarusAttachments
    ↓
GrantedAlteration
    ↓
D_Alterations
```

The actual stat modification generally lives in:

```text
D_Alterations.Stats
```

---

# 18. Attachment Tags

Attachment compatibility is strongly tied to gameplay tags.

Examples may include:

```text
Item.Attachment.Rank.1
Item.Attachment.Rifle
Item.Attachment.Pistol
Item.Attachment.Shotgun
Item.Attachment.Bow
Item.Attachment.Crossbow
```

A weapon slot can use a tag query to determine what can be inserted.

Therefore investigate:

```text
Attachment item
    ↓
Manual_Tags
    ↓
D_TagQueries
    ↓
Weapon slot
```

together.

---

# 19. Attachment Slots

Attachment slots are separate from the attachment item itself.

Typical chain:

```text
D_ItemsStatic
    ↓
InventoryContainer
    ↓
D_InventoryContainer
    ↓
InventoryInfo
    ↓
D_InventoryInfo
```

Then inspect:

```text
StartingSlots
SlotOverrides
SlotTemplate
```

When investigating why an attachment does or does not fit, check:

```text
Attachment tags
↓
D_TagQueries
↓
SlotTemplate / SlotOverrides
```

---

# 20. Shared Rows — Critical Modding Rule

A row can be referenced by multiple items.

For example, a shared inventory row may be used by several rifles.

Changing a shared row can therefore change every item that references it.

### Safe modification pattern

```text
Clone shared row
        ↓
Give clone a unique Name
        ↓
Clone dependent rows if required
        ↓
Repoint target item
        ↓
Modify clone
```

Never modify a shared vanilla row without checking its references first.

---

# 21. Living Items

Living Items are a separate system from standard attachments.

Relevant tables include:

```text
D_LivingItem
D_LivingItemUpgrades
D_LivingItemShopItems
D_Alterations
```

Typical architecture:

```text
D_ItemsStatic
    ↓
LivingItem
    ↓
D_LivingItem
    ↓
UpgradeSlots
    ↓
D_LivingItemUpgrades
    ↓
AlterationToApply
    ↓
D_Alterations
```

The Living Item system is used for Great Hunt legendary weapons.

---

# 22. Standard Attachments vs Living Items

These systems must not be confused.

### Standard attachment

```text
D_ItemsStatic
 ↓
D_IcarusAttachments
 ↓
D_Alterations
```

### Living Item

```text
D_LivingItem
 ↓
D_LivingItemUpgrades
 ↓
D_Alterations
```

Stat effects may be reusable, but visual and skeletal customisation should not automatically be assumed to transfer between systems.

---

# 23. D_Transmutable Warning

`D_Transmutable` is **not** the standard weapon alteration system.

Despite its name, it is associated with conversion/incinerator/fuel behaviour.

For weapon attachment stat modifications, investigate:

```text
D_IcarusAttachments
    ↓
D_Alterations
```

instead.

---

# 24. D_Processing

`D_Processing` controls recipe-processing behaviour for crafting stations.

Potential fields include:

```text
DefaultRecipeSet
bRequiresShelter
QueueSize
```

Example:

```json
{
    "Name": "Crafting_Bench",
    "DefaultRecipeSet": {
        "RowName": "Crafting_Bench"
    },
    "bRequiresShelter": true,
    "QueueSize": 5
}
```

### Important distinction

Not every deployable is a recipe-processing station.

Some stations use Blueprint-driven behaviour instead.

Therefore DataTable behaviour and Blueprint behaviour must be treated as separate systems.

---

# 25. Decay and Spoilage

Decay-related item architecture:

```text
D_ItemsStatic
    ↓
Decayable
    ↓
D_Decayable
```

Example research values include rows such as:

```json
{
    "Name": "Decay_Ice",
    "DecayTime": 300,
    "SpoilTime": 2700
}
```

Do not assume the units or gameplay interpretation of a decay field solely from its name.

Record confirmed values separately from assumptions.

---

# 26. Durability

Durability architecture:

```text
D_ItemsStatic
    ↓
Durable
    ↓
D_Durable
```

When modifying durability:

1. Find the item in `D_ItemsStatic`.
2. Follow `Durable.RowName`.
3. Find the corresponding `D_Durable` row.
4. Check whether that row is shared.
5. Clone it if only one item should change.
6. Test the resulting behaviour.

---

# 27. Consumables

Typical architecture:

```text
D_ItemsStatic
    ↓
Consumable
    ↓
D_Consumable
```

Food and drink may also interact with:

```text
D_ModifierStates
D_StatAfflictions
D_GrantedAuras
D_SurvivalTriggers
D_Stats
```

Do not assume every effect is stored directly inside `D_Consumable`.

Follow every `RowName` reference.

---

# 28. Armour

Armour can involve:

```text
D_ItemsStatic
    ↓
Armour
    ↓
D_Armour
```

Armour sets may additionally use:

```text
D_ArmourSets
D_ArmourSetBonus
```

When modifying armour, determine whether the desired change belongs to an individual armour item or an armour set bonus.

---

# 29. Building and Deployables

Relevant tables include:

```text
D_BuildingLookup
D_BuildingPieces
D_BuildingSkins
D_BuildingStability

D_DeployableSetup
D_Turret

D_Buildable
D_Deployable
D_Processing
```

A placeable object can therefore require several layers:

```text
Item
 ↓
Deployable / Buildable
 ↓
DeployableSetup
 ↓
Blueprint
```

If a desired behaviour is implemented inside a Blueprint, changing DataTables alone may not be sufficient.

---

# 30. Resources and Resource Nodes

Relevant tables include:

```text
D_IcarusResources
D_OptionalResourceFlows

D_BreakableRockData
D_OreDeposit
D_Surfaces
D_VoxelDistributionRegion
D_VoxelMaterialMap
D_VoxelSetupData
D_WaterSetup

D_MetaResourceNodes
```

When investigating mining/resource behaviour, determine whether the desired change concerns:

```text
tool behaviour
```

or:

```text
resource-node behaviour
```

These are different systems.

---

# 31. Farming

Relevant tables include:

```text
D_DirtMoundModifications
D_Farmable
D_FarmingGrowthStates
D_FarmingSeeds
D_SeedModifications
```

Potential architecture:

```text
Seed
 ↓
D_FarmingSeeds
 ↓
D_FarmingGrowthStates
 ↓
D_Farmable
```

For farming changes, always inspect the complete chain rather than assuming `D_Farmable` contains everything.

---

# 32. Creatures and AI

Creature-related tables are distributed across:

```text
AI/
Bestiary/
GreatHunt/
Spawn/
World/
Traits/
Modifiers/
```

Important categories include:

```text
D_AICreatureType
D_AISetup
D_AISpawnConfig
D_AISpawnRules
D_AISpawnZones
D_Tames
D_Mounts
D_Saddles
D_WorldBosses
D_BestiaryData
D_GreatHuntCreatureInfo
```

Creature modding should be investigated as a subsystem rather than treating one table as the "creature file."

---

# 33. Quests

Quest data is distributed across:

```text
Quests/
```

including:

```text
D_DynamicQuestRewardItems
D_DynamicQuestRewards
D_DynamicQuests
D_QuestEvents
D_QuestQueries
D_Quests
D_StasisBag
```

Quest changes should follow references between these tables.

A quest row may not contain all behaviour directly.

---

# 34. Weather, World and Prospects

Relevant systems include:

```text
Prospects/
    D_Atmospheres
    D_Biomes
    D_ProspectList
    D_Terrains

Weather/
    D_ProspectForecast
    D_WeatherActions
    D_WeatherBiomeGroups
    D_WeatherEvents
    D_WeatherPools
    D_WeatherTierIcon

World/
    D_WorldData
    D_WaterSetup
    D_Surfaces
    D_VoxelSetupData
```

World changes require particular care because multiple systems may contribute to the final result.

---

# 35. Modding Decision Tree

When starting a mod, first identify what type of change is required.

### Change an existing item

```text
Find item
 ↓
D_ItemsStatic
 ↓
Follow component pointer
 ↓
Check target row
 ↓
Check whether target row is shared
 ↓
Modify or clone
```

### Create a new item

```text
Create component rows
 ↓
Create D_ItemsStatic row
 ↓
Create D_ItemTemplate row
 ↓
Optional D_Talents row
 ↓
D_ProcessorRecipes
```

### Create a new craftable item

```text
D_ItemsStatic
 ↓
D_ItemTemplate
 ↓
D_ProcessorRecipes
 ↓
D_RecipeSets
 ↓
Optional D_Talents
```

### Create a new attachment

```text
D_ItemsStatic
 ↓
D_IcarusAttachments
 ↓
D_Alterations
 ↓
Tags
 ↓
D_TagQueries
```

### Change weapon attachment slots

```text
D_ItemsStatic
 ↓
D_InventoryContainer
 ↓
D_InventoryInfo
 ↓
StartingSlots
SlotOverrides
SlotTemplate
 ↓
D_TagQueries
```

---

# 36. Modify vs Clone vs Create

Every modding task should first be classified as one of three types.

## Modify Existing

Use when the change should affect the original row and all users of that row.

## Clone Existing

Use when only one item or system should receive the change.

This is generally the safest approach for shared component rows.

## Create New

Use when the mod introduces a genuinely new item, recipe, attachment, talent, etc.

---

# 37. Shared Row Investigation

Before changing any existing row:

### Ask:

> "Who else references this row?"

Recommended process:

```text
Target item
 ↓
Referenced row
 ↓
Search repository for RowName
 ↓
List every parent using it
 ↓
Determine whether it is safe to modify
```

If not:

```text
Clone
 ↓
Repoint
 ↓
Modify
```

---

# 38. "What Controls What?" Quick Reference

| Desired change | Primary starting point |
|---|---|
| Item name | `D_Itemable` |
| Item description | `D_Itemable` |
| Item icon | `D_Itemable` |
| Item weight | `D_Itemable` / `D_Weight` |
| Stack size | `D_Itemable` |
| Durability | `D_Durable` |
| Decay/spoilage | `D_Decayable` |
| Melee damage | `D_ToolDamage` |
| Mining efficiency | `D_ToolDamage` |
| Mining radius | `D_ToolDamage` |
| Fire rate | `D_FirearmData` |
| Firearm behaviour | `D_FirearmData` |
| Ranged weapon behaviour | `D_RangedWeaponData` |
| Ammo compatibility | `D_AmmoTypes` / `D_ValidAmmoTypes` / tags |
| Attachment stats | `D_Alterations` |
| Attachment link | `D_IcarusAttachments` |
| Attachment compatibility | `D_TagQueries` + Gameplay Tags |
| Number of attachment slots | `D_InventoryInfo.StartingSlots` |
| Special attachment slot | `D_InventoryInfo.SlotOverrides` |
| Recipe ingredients | `D_ProcessorRecipes.Inputs` |
| Recipe output | `D_ProcessorRecipes.Outputs` |
| Crafting station | `D_ProcessorRecipes.RecipeSets` |
| Character crafting | `RecipeSets → Character` |
| Recipe talent | `D_ProcessorRecipes.Requirement → D_Talents` |
| Talent prerequisites | `D_Talents.RequiredTalents` |
| Talent tree | `D_Talents.TalentTree` |
| Crafting bench queue | `D_Processing.QueueSize` |
| Shelter requirement | `D_Processing.bRequiresShelter` |
| Armour stats | `D_Armour` |
| Armour set bonus | `D_ArmourSetBonus` |
| Food/drink effects | `D_Consumable` + referenced modifier systems |
| Resource nodes | `World/` + `MetaResource/` |
| Creature AI | `AI/` |
| Creature spawn | `AI/` + `Spawn/` |
| Quest data | `Quests/` |
| Weather | `Weather/` |
| Prospect/world configuration | `Prospects/` + `World/` |

This table is a starting point, not a substitute for tracing the actual row references.

---

# 39. Common Modding Mistakes

## Mistake 1 — Editing the wrong table

Finding an item in `D_ItemsStatic` does not mean the desired value is stored there.

Follow the pointer.

## Mistake 2 — Modifying a shared row

Always check whether another item references the same component.

## Mistake 3 — Guessing a CurrentFile category

The category is part of the mod file identity.

Use the repository's actual folder/category.

## Mistake 4 — Assuming a field's meaning

A field called `Efficiency` does not automatically mean percentage or multiplier.

Verify using:

- Defaults
- existing rows
- related systems
- in-game testing

## Mistake 5 — Confusing standard attachments with Living Items

They use different systems.

## Mistake 6 — Assuming a DataTable controls Blueprint behaviour

Some gameplay is compiled into Blueprints.

Data-only JSON cannot necessarily modify Blueprint logic.

---

# 40. Standard Mod Investigation Procedure

When asked to modify something:

### Step 1

Find the item/system.

### Step 2

Find its primary DataTable.

### Step 3

Follow every relevant `RowName`.

### Step 4

Identify the actual controlling table.

### Step 5

Inspect `Defaults`.

### Step 6

Check other rows using the same referenced row.

### Step 7

Determine whether the row should be:

```text
modified
cloned
or newly created
```

### Step 8

Create the smallest possible modification.

### Step 9

Test in-game.

### Step 10

Record the result as:

```text
VERIFIED
INFERRED
or UNTESTED
```

---

# 41. Mod Investigation Standard

For every future modding task, document:

```text
Target:
Original item/system:

Primary table:

Referenced tables:

Relevant rows:

Field being changed:

Original value:

New value:

Shared row:
Yes / No

Clone required:
Yes / No

Confidence:
VERIFIED / INFERRED / UNTESTED

In-game result:

Notes:
```

This makes future modding much easier and prevents rediscovering the same information.

---

# 42. Mod Output Standard

When creating a mod, produce two forms of information.

### Human-readable summary

Explain:

```text
What changed
Why it changed
Which tables changed
Which rows changed
Whether rows were cloned
```

### Raw mod data

Provide the actual JSON changes separately.

This allows the mod to be reviewed before being merged into the final mod.

---

# 43. CurrentFile Category Map

The repository's `data/` structure is the source of truth for `CurrentFile` categories.

The current repository contains the following major categories:

```text
AI
Accolades
Alterations
Animation
Armour
Assets
Attachments
Audio
Bestiary
Blueprints
Building
Challenges
Character
Config
Crafting
CriticalHit
Currency
DLC
Damage
Deployables
Development
Dialogue
DropShip
Errors
Events
Experience
FLOD
Factions
Farming
FieldGuide
Fish
Flags
GreatHunt
Hints
Horde
Input
InstancedMap
Interactions
Inventory
Items
LivingItems
Localization
Logging
MetaResource
MetaWorkshop
Modifiers
NationalFlags
Notes
Online
Orchestration
Outpost
Paintings
Perks
PlayerTracker
Prebuilt
Prospects
Quests
RTXGI
Resources
Rulesets
Scaling
Settings
Settlement
Sorting
Spawn
Statistics
Stats
Tags
Talents
TimeOfDay
Tools
Traits
UI
ValidHits
Vehicles
Weather
World
```

The category map should be updated if the repository structure changes.

---

# 44. Important Tables by Category

## Crafting

```text
D_CraftingModifications
D_CraftingTags
D_ExtractorRecipes
D_ProcessorRecipes
D_RecipeSets
```

## Items

```text
D_ItemClassificationsIcons
D_ItemRanks
D_ItemRewards
D_ItemTemplate
D_ItemTraitMasks
D_ItemsStatic
```

## Inventory

```text
D_BagPriority
D_InventoryContainer
D_InventoryID
D_InventoryInfo
D_ItemWeightStatQueries
D_QuickMove
```

## Attachments

```text
D_AttachmentIcons
D_IcarusAttachments
D_ItemAttachments
```

## Alterations

```text
D_AlterationModifiers
D_Alterations
```

## Tags

```text
D_StatGameplayTags
D_TagQueries
```

## Talents

```text
D_PlayerTalentModifiers
D_TalentArchetypes
D_TalentModelViews
D_TalentModels
D_TalentRanks
D_TalentTrees
D_TalentViews
D_Talents
```

## Tools

```text
D_Actions
D_AmmoTypes
D_FirearmData
D_FirearmScopeData
D_NPCWeapon
D_RangedWeaponData
D_StaminaActionCosts
D_ToolDamage
D_Uses
D_ValidAmmoTypes
```

## Traits

```text
D_Actionable
D_Armour
D_Ballistic
D_Buildable
D_Combustible
D_Consumable
D_CrudeOil
D_Decayable
D_Deployable
D_Durable
D_Energy
D_Equippable
D_Experience
D_Fillable
D_Flammable
D_Floatable
D_Focusable
D_Generator
D_Highlightable
D_Hitable
D_Interactable
D_Inventory
D_Itemable
D_LivingItem
D_Meshable
D_Oxygen
D_Processing
D_RefinedOil
D_Resource
D_Rocketable
D_Slotable
D_Thermal
D_Transmutable
D_Usable
D_Vehicular
D_Water
D_Weight
```

---

# 45. Important Cross-Reference Chains

## Basic Item

```text
D_ItemsStatic
    ↓
D_Itemable
```

## Item visual

```text
D_ItemsStatic
    ↓
D_Meshable
```

## Durability

```text
D_ItemsStatic
    ↓
D_Durable
```

## Decay

```text
D_ItemsStatic
    ↓
D_Decayable
```

## Tool

```text
D_ItemsStatic
    ↓
D_ToolDamage
```

## Firearm

```text
D_ItemsStatic
    ↓
D_FirearmData
```

## Craftable item

```text
D_ItemsStatic
    ↓
D_ItemTemplate
    ↓
D_ProcessorRecipes
```

## Recipe ingredients

```text
D_ProcessorRecipes.Inputs
    ↓
D_ItemsStatic
```

## Recipe output

```text
D_ProcessorRecipes.Outputs
    ↓
D_ItemTemplate
    ↓
D_ItemsStatic
```

## Recipe talent

```text
D_ProcessorRecipes.Requirement
    ↓
D_Talents
```

## Talent prerequisites

```text
D_Talents.RequiredTalents
    ↓
D_Talents
```

## Standard attachment

```text
D_ItemsStatic
    ↓
D_IcarusAttachments
    ↓
D_Alterations
```

## Attachment compatibility

```text
D_ItemsStatic.Manual_Tags
    ↓
Gameplay Tags
    ↓
D_TagQueries
    ↓
D_InventoryInfo
```

## Weapon attachment slots

```text
D_ItemsStatic
    ↓
D_InventoryContainer
    ↓
D_InventoryInfo
    ↓
StartingSlots
SlotOverrides
SlotTemplate
```

## Living weapon

```text
D_ItemsStatic
    ↓
D_LivingItem
    ↓
D_LivingItemUpgrades
    ↓
D_Alterations
```

---

# 46. Repository Maintenance Rules

When adding new research to this repository:

### Always record the exact table

Example:

```text
Crafting/D_ProcessorRecipes.json
```

### Record the exact row

Example:

```text
Stone_Pickaxe
```

### Record the exact field

Example:

```text
RecipeSets
```

### Record the relationship

Example:

```text
RecipeSets.RowName
→ D_RecipeSets
```

### Record confidence

```text
VERIFIED
INFERRED
UNTESTED
```

### Record game version/export date when possible

This prevents old findings from being treated as current after a major ICARUS update.

---

# 47. JSON Safety

Standard JSON does not support comments.

Do not place:

```text
//
/*
#
```

inside production JSON unless the specific modding tool explicitly preprocesses it.

Keep research notes in:

```text
README.md
```

or separate documentation files.

Keep production JSON valid JSON.

---

# 48. Practical Cross-Reference Method

When analysing a subsystem, load the relevant tables into dictionaries keyed by `Name`.

Conceptually:

```python
def load(path):
    with open(path) as f:
        return {
            row["Name"]: row
            for row in json.load(f)["Rows"]
        }
```

Then:

```text
Find item
 ↓
Read pointer
 ↓
Find target row
 ↓
Read next pointer
 ↓
Repeat
```

Do not try to understand the entire ICARUS database at once.

Work by subsystem.

Examples:

```text
Firearms
Items
Tools
Inventory
Attachments
Talents
Crafting
```

---

# 49. Recommended Research Workflow

For a new modding question:

```text
1. Identify the desired gameplay change.

2. Find the item/system.

3. Find its D_ItemsStatic or primary table row.

4. Follow every relevant RowName reference.

5. Inspect Defaults.

6. Find all other references to the target row.

7. Determine whether the row is shared.

8. Determine whether the desired value is direct,
   a multiplier, percentage, enum, tag, or reference.

9. Modify/clone/create the appropriate row.

10. Build the mod.

11. Test in-game.

12. Record the result.

13. Update this README when a relationship is confirmed.
```

---

# 50. Golden Rules

### Rule 1

**Never guess the controlling table. Follow the reference.**

### Rule 2

**Never modify a shared row without checking its users.**

### Rule 3

**Never assume a number's unit or meaning from its name.**

### Rule 4

**Never assume a DataTable can change Blueprint behaviour.**

### Rule 5

**Separate standard attachments from Living Items.**

### Rule 6

**Use the repository's folder/category when creating `CurrentFile`.**

### Rule 7

**Use `D_ItemTemplate` for recipe outputs unless the specific table proves otherwise.**

### Rule 8

**Use `D_ItemsStatic` for item inputs unless the specific table proves otherwise.**

### Rule 9

**Record discoveries as VERIFIED, INFERRED, or UNTESTED.**

### Rule 10

**When a mod works, document the exact relationship that made it work.**

---

# 51. Known High-Value Modding Chains

These are the chains that should be checked first for common modding requests.

### "Make a tool stronger"

```text
D_ItemsStatic
 ↓
ToolDamage
 ↓
D_ToolDamage
```

### "Make a tool mine faster"

```text
D_ItemsStatic
 ↓
ToolDamage
 ↓
D_ToolDamage
 ↓
Mining_Efficiency
```

### "Increase mining area"

```text
D_ItemsStatic
 ↓
ToolDamage
 ↓
D_ToolDamage
 ↓
Mining_Radius
```

### "Make an item last longer"

```text
D_ItemsStatic
 ↓
Durable
 ↓
D_Durable
```

### "Stop/reduce spoilage"

```text
D_ItemsStatic
 ↓
Decayable
 ↓
D_Decayable
```

### "Make an item craftable"

```text
D_ItemsStatic
 ↓
D_ItemTemplate
 ↓
D_ProcessorRecipes
```

### "Make it character craftable"

```text
D_ProcessorRecipes
 ↓
RecipeSets
 ↓
Character
```

### "Add a talent requirement"

```text
D_ProcessorRecipes
 ↓
Requirement
 ↓
D_Talents
```

### "Give a weapon more attachment slots"

```text
D_ItemsStatic
 ↓
InventoryContainer
 ↓
D_InventoryContainer
 ↓
D_InventoryInfo
 ↓
StartingSlots
```

### "Make a new weapon attachment"

```text
D_ItemsStatic
 ↓
D_IcarusAttachments
 ↓
D_Alterations
 ↓
GameplayTags
 ↓
D_TagQueries
```

---

# 52. Future Documentation Standard

Whenever a new ICARUS modding discovery is confirmed, add it using this format:

```text
## [System] — [Feature]

Status:
VERIFIED / INFERRED / UNTESTED

Primary Table:
Category/File

Primary Row:
RowName

Reference Chain:

Table A
 ↓
Field
 ↓
Table B
 ↓
Field
 ↓
Table C

What it controls:

[Explanation]

Test:

[What was changed]

Result:

[Observed result]

Notes:

[Anything else important]
```

This creates a searchable history of actual modding discoveries rather than relying on memory.

---

# 53. Final Principle

The most important concept in ICARUS data-only modding is:

> **The item is rarely the complete system.**

An item usually acts as a hub.

```text
D_ItemsStatic
      │
      ├── Itemable
      ├── Meshable
      ├── Durable
      ├── Decayable
      ├── ToolDamage
      ├── FirearmData
      ├── InventoryContainer
      ├── Attachments
      ├── Consumable
      ├── Armour
      └── other components
```

The correct mod is therefore usually created by:

```text
Find the item
       ↓
Follow the component
       ↓
Find the controlling row
       ↓
Follow any secondary reference
       ↓
Check whether the row is shared
       ↓
Modify / clone / create
       ↓
Test
       ↓
Document
```

That process should be used for every future ICARUS modding investigation.

---

# 54. Repository Status

This repository is a living research reference.

The JSON data provides the structural source.

The README documents the relationships discovered from that data.

In-game testing provides the final confirmation of behaviour.

When those three disagree:

```text
JSON structure
    ↓
Investigate
    ↓
Defaults
    ↓
Cross-reference
    ↓
In-game test
    ↓
Document confirmed result
```

Never overwrite a known fact with an assumption.

---

## Primary Reference Tables

**Primary item hub:** `D_ItemsStatic`

**Primary crafting table:** `D_ProcessorRecipes`

**Primary item template table:** `D_ItemTemplate`

**Primary talent table:** `D_Talents`

**Primary attachment chain:** `D_IcarusAttachments → D_Alterations`

**Primary attachment-slot chain:** `D_InventoryContainer → D_InventoryInfo`

**Primary weapon/tool stat table:** `D_ToolDamage`

**Primary firearm table:** `D_FirearmData`

**Primary tag query table:** `D_TagQueries`

**Primary crafting-station behaviour:** `D_Processing`

Use these as starting points when investigating most item and crafting modifications.
