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

* **298 DataTables**
* Tables organized into functional categories under `data/`
* `D_ItemsStatic.json` as the central hub for item definitions
* Component tables for individual item behaviours
* Crafting, talent, inventory, attachment, weapon, armour, creature, world and gameplay tables

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

* RowName references
* Defaults
* repeated usage
* naming
* cross-table relationships

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

| D_ItemsStatic field  | Target                 | Purpose                                            |
| -------------------- | ---------------------- | -------------------------------------------------- |
| `Meshable`           | `D_Meshable`           | Mesh/visual representation                         |
| `Itemable`           | `D_Itemable`           | Name, description, icon, weight, stack information |
| `Interactable`       | `D_Interactable`       | Interaction behaviour                              |
| `Highlightable`      | `D_Highlightable`      | Highlight/outline behaviour                        |
| `Focusable`          | `D_Focusable`          | Focus/ADS behaviour                                |
| `Actionable`         | `D_Actionable`         | Actions available to the item                      |
| `Usable`             | `D_Usable`             | Use/repair/consume behaviour                       |
| `Durable`            | `D_Durable`            | Durability                                         |
| `Decayable`          | `D_Decayable`          | Spoilage/decay                                     |
| `Equippable`         | `D_Equippable`         | Equipment rules                                    |
| `Consumable`         | `D_Consumable`         | Food/drink effects                                 |
| `Fillable`           | `D_Fillable`           | Liquid-container behaviour                         |
| `Resource`           | `D_Resource`           | Resource classification                            |
| `Rocketable`         | `D_Rocketable`         | Rocket-related eligibility                         |
| `Transmutable`       | `D_Transmutable`       | Incinerator/fuel conversion                        |
| `Slotable`           | `D_Slotable`           | World socket/slot behaviour                        |
| `ToolDamage`         | `D_ToolDamage`         | Tool/melee damage and mining-related values        |
| `RangedWeaponData`   | `D_RangedWeaponData`   | Ranged weapon behaviour                            |
| `FirearmData`        | `D_FirearmData`        | Firearm-specific behaviour                         |
| `Armour`             | `D_Armour`             | Armour-specific data                               |
| `Buildable`          | `D_Buildable`          | Building behaviour                                 |
| `Deployable`         | `D_Deployable`         | Deployable behaviour                               |
| `Generator`          | `D_Generator`          | Power-generation behaviour                         |
| `AmmoType`           | `D_AmmoTypes`          | Ammunition behaviour                               |
| `Ballistic`          | `D_Ballistic`          | Projectile behaviour                               |
| `InventoryContainer` | `D_InventoryContainer` | Internal inventory/attachment slots                |
| `Attachments`        | `D_IcarusAttachments`  | Attachment stat system                             |
| `Processing`         | `D_Processing`         | Crafting-station behaviour                         |
| `LivingItem`         | `D_LivingItem`         | Great Hunt legendary weapon system                 |
| `Audio`              | Audi000                |                                                    |

`D_Itemable` is one of the most important component tables.

It commonly controls item presentation and basic item properties, including things such as:

* Display name
* Description
* Icon
* Weight
* Stack behaviour
* Item classification information

### Important

Do not assume every property commonly associated with an "item" is stored in `D_Itemable`.

Check the row and its Defaults definition.

---

# 8. D_ItemTemplate

`D_ItemTemplate` is a thin layer connecting an item template to its `D_ItemsStatic` definition.0**-station behaviour:** `D_Processing`

Use these as the starting points when investigating most item and crafting modifications.
