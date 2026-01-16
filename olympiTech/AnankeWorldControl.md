# Ananke World Control

## Overview
Ananke WorldControl (AWC) is a lightweight server-control plugin for Paper/Spigot-style servers.

It lets you *allow or block* specific world mechanics at runtime:

- Environment rules (fire spread, weather, leaf decay, ice/snow behavior, etc.)
- Physics and fluids (block physics, gravity blocks, fluid mixing, waterlogging, bubble columns)
- Mob spawning and mob griefing
- Explosions (TNT, creeper, wither, bed/anchor)
- Player survival mechanics (fall damage, hunger loss, natural regen, void protection)
- Per-block restrictions (disable placing/breaking/interacting with specific blocks)
- Crafting restrictions (disable crafting specific items)
- Dimension controls (Nether/End access, Nether roof prevention)
- Optional custom-block restrictions (ItemsAdder/Oraxen/Nexo + PDC based)

AWC is designed so you can change most settings without editing huge lists by hand:

- It **auto-populates** `blocks.yml`, `mobs.yml`, `crafting.yml`, and `dimensions.yml`.
- It exposes a single admin command (`/awc`) for common toggles.

## Requirements

- **Minecraft server**: Bukkit/Paper/Purpur/Spigot
- **Minecraft version**: 1.21.x (`api-version: "1.21"`)
- **Permissions plugin**: optional but recommended for managing bypass/admin nodes

## Installation

1. Put the jar into `plugins/`.
2. Start/restart your server.
3. Configure:
   - `plugins/AnankeWorldControl/config.yml` (feature toggles + messages)
   - `plugins/AnankeWorldControl/blocks.yml` (block allow/deny list)
   - `plugins/AnankeWorldControl/mobs.yml` (mob allow/deny list)
   - `plugins/AnankeWorldControl/crafting.yml` (crafting allow/deny list)
   - `plugins/AnankeWorldControl/dimensions.yml` (dimension rules)
   - `plugins/AnankeWorldControl/custom-blocks.yml` (custom block allow/deny list)

## Permissions

Defined in `plugin.yml`:

- `ananke.worldcontrol.admin`
  - Required for `/awc ...`
  - Default: op

- `ananke.worldcontrol.bypass`
  - Bypasses *most* restrictions (block placement/break/interact, crafting prevention, dimension restrictions, etc.)
  - Default: op

## Command reference (`/awc`)

Command:

- `/awc <subcommand>`

All subcommands require `ananke.worldcontrol.admin`.

### `/awc reload`

Reloads:

- `config.yml`
- `blocks.yml`
- `mobs.yml`
- `crafting.yml`
- `custom-blocks.yml`
- `dimensions.yml`

### `/awc block <material> <true|false>`

Sets whether a vanilla block material is allowed in `blocks.yml`.

Examples:

- `/awc block TNT false`
- `/awc block minecraft:respawn_anchor false`

### `/awc mob <type> <true|false>`

Sets whether a mob can spawn in `mobs.yml`.

Examples:

- `/awc mob CREEPER false`
- `/awc mob PHANTOM false`

### `/awc craft <item> <true|false>`

Sets whether crafting/smelting an item is allowed in `crafting.yml`.

Examples:

- `/awc craft DIAMOND_SWORD false`
- `/awc craft GOLDEN_APPLE false`

### `/awc customblock list`

Lists known custom block IDs recorded so far.

### `/awc customblock set <id> <true|false>`

Sets a custom block ID allowed/denied in `custom-blocks.yml`.

### `/awc feature <name> <true|false>`

Toggles a feature in `config.yml`. Known features are hard-coded in `AwcCommand`.

Examples:

- `/awc feature fire-spread false`
- `/awc feature redstone false`

### `/awc dimension <nether|end|nether-roof> <true|false>`

Sets dimension defaults in `dimensions.yml`:

- `nether` => `defaults.nether-enabled`
- `end` => `defaults.end-enabled`
- `nether-roof` => `defaults.prevent-nether-roof`

## How it works (Architecture)

Main entrypoint:

- `com.cyb3rz3us.ananke.worldcontrol.AnankeWorldControlPlugin`

On enable:

- Saves default `config.yml`
- Loads/merges:
  - `dimensions.yml` (`DimensionConfig#loadAndMerge`)
  - `blocks.yml` (`BlockControl#loadAndMerge`)
  - `mobs.yml` (`MobControl#loadAndMerge`)
  - `crafting.yml` (`CraftControl#loadAndMerge`)
  - `custom-blocks.yml` (`CustomBlockControl#load`)
- Registers listeners that enforce restrictions
- Starts a repeating **time-lock task** (every second) if `environment.time-lock` is set to a valid tick value

Key design points:

- Feature toggles are read via `PluginSettings#isFeatureAllowed(key)`
  - Searches `environment.*`, `physics.*`, `mobs.*`, `players.*`, `interactions.*` (and legacy `features.*` fallback)
- Allow/deny lists are stored in separate YAMLs and automatically normalized to canonical names.

## Configuration

### `config.yml` (feature toggles + messages)

Location:

- `plugins/AnankeWorldControl/config.yml`

The main config is organized into categories. In general:

- `true` means **allowed**
- `false` means **blocked**

#### `environment.*` (shipped keys)

| Key | Default | What it controls | If set to `false` |
|---|---:|---|---|
| `environment.fire-spread` | `true` | Fire spreading to adjacent blocks (`BlockSpreadEvent` source = `FIRE`). | Prevents fire spread. |
| `environment.weather-change` | `true` | Rain/sun toggles (`WeatherChangeEvent`). | Cancels weather changes. |
| `environment.weather-thunder` | `true` | Thunder toggles (`ThunderChangeEvent`). | Cancels thunder changes. |
| `environment.grass-spread` | `true` | Grass spreading (`BlockSpreadEvent` source = `GRASS_BLOCK`). | Prevents grass spread. |
| `environment.mycelium-spread` | `true` | Mycelium spreading (`BlockSpreadEvent` source = `MYCELIUM`). | Prevents mycelium spread. |
| `environment.moss-spread` | `true` | Moss spreading (`BlockSpreadEvent` source = `MOSS_BLOCK`). | Prevents moss spread. |
| `environment.vine-spread` | `true` | Vine spreading (`BlockSpreadEvent` source = `VINE`). | Prevents vine spread. |
| `environment.leaf-decay` | `true` | Natural leaf decay (`LeavesDecayEvent`). | Cancels leaf decay. |
| `environment.ice-form` | `true` | Ice forming (`BlockFormEvent` new state = `ICE`). | Cancels ice formation. |
| `environment.ice-melt` | `true` | Ice melting (`BlockFadeEvent` for `ICE`/`FROSTED_ICE`). | Cancels melting. |
| `environment.snow-form` | `true` | Snow forming (`BlockFormEvent` new state = `SNOW`). | Cancels snow formation. |
| `environment.snow-melt` | `true` | Snow melting (`BlockFadeEvent` for `SNOW`). | Cancels melting. |
| `environment.cauldron-fill` | `true` | Cauldron natural fill (`CauldronLevelChangeEvent` reason = `NATURAL_FILL`). | Cancels natural cauldron filling. |
| `environment.farmland-moisture` | `true` | Farmland moisture changes (`MoistureChangeEvent`). | Cancels moisture changes (prevents drying/wetting updates). |
| `environment.lightning-fire` | `true` | Whether lightning is allowed to ignite blocks (`BlockIgniteEvent` cause = `LIGHTNING` in `PhysicsControlListener`). | Cancels lightning-based ignition. |
| `environment.time-lock` | `-1` | World time lock task (runs every second). Set to `-1` to disable. Set to `0-24000` to lock time to that tick value. | Not applicable (this is an integer). |

#### `physics.*` (shipped keys)

| Key | Default | What it controls | If set to `false` |
|---|---:|---|---|
| `physics.block-physics` | `true` | General block physics (`BlockPhysicsEvent` in `EnvironmentControlListener`). | Cancels many physics updates (e.g. blocks updating). |
| `physics.gravity-block-fall` | `true` | Gravity blocks (sand/gravel/anvils/concrete powder) (`EntityChangeBlockEvent` + physics checks). | Prevents falling blocks / gravity behavior in relevant handlers. |
| `physics.water-spread` | `true` | Water spreading (`BlockFromToEvent` in `FluidSpreadListener`). | Cancels water spread. |
| `physics.lava-spread` | `true` | Lava spreading (`BlockFromToEvent` in `FluidSpreadListener`). | Cancels lava spread. |
| `physics.infinite-water-sources` | `true` | Infinite water source formation (`BlockFormEvent` new state = `WATER` in `PhysicsControlListener`). | Cancels water source formation events that trigger this handler. |
| `physics.block-waterlogging` | `true` | Waterlogging behaviors (bucket into waterloggable blocks + flow into `Waterlogged` block data). | Cancels waterlogging attempts (may affect slabs/stairs). |
| `physics.fluid-mixing` | `true` | Water/lava mixing outcomes like obsidian/stone/cobble (`BlockFormEvent`). | Cancels those formation events. |
| `physics.bubble-columns` | `true` | Bubble column creation/updates (`BlockPhysicsEvent` changed type `BUBBLE_COLUMN`). | Cancels bubble column physics. |
| `physics.explosion-knockback` | `true` | Intended to control explosion knockback behavior (velocity handling in `PhysicsControlListener`). | When `false`, the plugin attempts to keep velocity stable during explosion damage. |
| `physics.block-rotation` | `true` | Rotation/axis/facing enforcement + debug stick blocking (`BlockRotationListener`). | Resets placed block data and blocks debug stick usage. |

#### `mobs.*` (shipped keys)

| Key | Default | What it controls | If set to `false` |
|---|---:|---|---|
| `mobs.phantom-spawn` | `true` | Phantom spawning (`CreatureSpawnEvent` extra check for `PHANTOM`). | Cancels phantom spawns. |
| `mobs.enderman-grief` | `true` | Endermen picking up/placing blocks (`EntityChangeBlockEvent`). | Cancels enderman grief. |
| `mobs.sheep-grow-wool` | `true` | Sheep wool regrow (`SheepRegrowWoolEvent`). | Cancels wool regrowth. |
| `mobs.villager-farming` | `true` | Villager farming actions (`EntityChangeBlockEvent` for `VILLAGER`). | Cancels villager block changes used by farming. |
| `mobs.ravager-grief` | `true` | Ravager block grief (`EntityChangeBlockEvent` for `RAVAGER`). | Cancels ravager block changes. |
| `mobs.tnt-explosions` | `true` | TNT and minecart explosions (`EntityExplodeEvent` + `BlockExplodeEvent`). | Cancels TNT explosions. |
| `mobs.creeper-explosions` | `true` | Creeper explosion block damage (`EntityExplodeEvent`). | Cancels creeper explosions or clears block damage depending on handler. |
| `mobs.wither-explosions` | `true` | Wither explosion block damage (`EntityExplodeEvent`). | Clears/cancels wither block damage depending on handler. |
| `mobs.ghast-fireball-damage` | `true` | Ghast fireball block damage (`EntityExplodeEvent` for `GHAST`). | Prevents ghast explosion block damage. |
| `mobs.wither-initial-explosion` | `true` | Intended for the wither spawn/initial blast behavior. | Behavior depends on listener logic; disabling generally reduces wither explosion effects. |
| `mobs.other-explosions` | `true` | Placeholder for miscellaneous explosion sources. | Disables/cancels other explosions where checked. |

#### `players.*` (shipped keys)

| Key | Default | What it controls | If set to `false` |
|---|---:|---|---|
| `players.fall-damage` | `true` | Fall damage (`EntityDamageEvent` cause `FALL`). | Cancels fall damage. |
| `players.hunger-loss` | `true` | Hunger decreasing (`FoodLevelChangeEvent`). | Cancels hunger loss. |
| `players.natural-regeneration` | `true` | Natural regen from being satiated (`EntityRegainHealthEvent` reason `SATIATED`). | Cancels natural regen. |
| `players.void-protection` | `true` | Void protection (`EntityDamageEvent` cause `VOID`). | When `false`, plugin teleports player to world spawn instead of taking void damage. |

#### `interactions.*` (shipped keys)

| Key | Default | What it controls | If set to `false` |
|---|---:|---|---|
| `interactions.redstone` | `true` | Redstone power (`BlockRedstoneEvent`). | Forces power to 0. |
| `interactions.flint-and-steel` | `true` | Flint & steel ignition (`BlockIgniteEvent` cause `FLINT_AND_STEEL`). | Cancels ignition attempts. |
| `interactions.chest-interaction` | `true` | Container interactions (basic detection in `PhysicsControlListener`). | Cancels right-click on containers and sends blocked message. |
| `interactions.prevent-piston-move-disabled-blocks` | `true` | Pistons moving disabled blocks (`BlockPistonExtendEvent`/`RetractEvent`). | When enabled, pistons are prevented from moving blocks that are disabled in `blocks.yml`. |
| `interactions.prevent-crafting-disabled-blocks` | `true` | Prevent crafting disabled blocks and also deny crafting for disabled items in `crafting.yml`. | When enabled, crafting results are blocked/cancelled. |
| `interactions.prevent-smelting-disabled-blocks` | `true` | Smelting restrictions (based on `blocks.yml` and `crafting.yml`). | Cancels furnace smelts to forbidden outputs. |
| `interactions.prevent-bucket-place-disabled-blocks` | `true` | Bucket empty placing forbidden blocks (e.g. WATER/LAVA). | Cancels bucket placements that would create disabled blocks. |
| `interactions.prevent-dispense-disabled-blocks` | `true` | Dispenser placing/dispensing forbidden blocks. | Cancels dispensing forbidden blocks/buckets. |

#### `messages.*` (shipped keys)

| Key | Default | Used for |
|---|---|---|
| `messages.prefix` | `&8[&bAWC&8]&r ` | Prefix for plugin messages. |
| `messages.blocked-block` | `&cYou cannot use that block here!` | Sent when a block is disabled by `blocks.yml`. |
| `messages.blocked-interact` | `&cInteraction with that block is disabled!` | Not used by all listeners, but reserved for interaction blocks. |
| `messages.blocked-feature` | `&cThat feature is currently disabled on this server.` | Generic “blocked” message for feature toggles. |

#### `custom-blocks.*` (shipped keys)

| Key | Default | What it controls |
|---|---:|---|
| `custom-blocks.enabled` | `true` | Enables custom block restriction system. |
| `custom-blocks.pdc-keys` | `[]` | List of PDC keys to read a string custom-block ID from the placed item. |

#### `dimensions` (shipped section)

The shipped `config.yml` contains a `dimensions:` section with a comment noting it is deprecated. Dimension controls are configured in `dimensions.yml`.

### `blocks.yml` (vanilla block allow/deny list)

Auto-generated and auto-updated.

Structure:

- `default-enabled` (boolean, default `true`)
  - The default value applied when AWC discovers new blocks (e.g. after Minecraft version updates).
- `blocks.<MATERIAL_NAME>`
  - Canonical `Material` name (e.g. `TNT`, `RESPAWN_ANCHOR`, `STONE`).

AWC migrates old keys automatically (lowercase / namespaced).

### `mobs.yml` (mob spawn allow/deny list)

Auto-generated and auto-updated.

Structure:

- `default-enabled` (boolean, default `true`)
- `mobs.<ENTITY_TYPE>`

Only living entities are included.

### `crafting.yml` (crafting/smelting allow/deny list)

This file is auto-populated by scanning all server recipes.

Structure:

- `default-enabled` (boolean, default `true`)
- `materials.<MATERIAL_NAME>`

Enforcement:

- `PrepareItemCraftEvent` prevents the output from appearing.
- `CraftItemEvent` cancels the craft.
- `FurnaceSmeltEvent` cancels smelting.

### `dimensions.yml` (dimension and nether-roof rules)

Structure:

- `defaults.nether-enabled` (default `true`)
- `defaults.end-enabled` (default `true`)
- `defaults.prevent-nether-roof` (default `true`)
- `defaults.nether-roof-min-y` (default `128`)
- `worlds.<worldName>.*` auto-populated for loaded worlds
  - `enabled`
  - `environment` (e.g. `NETHER`, `THE_END`)
  - For Nether worlds:
    - `prevent-nether-roof`
    - `nether-roof-min-y`

Enforcement includes:

- Cancelling portal/teleport attempts into disabled Nether/End worlds
- Kicking players out of Nether/End on join if the world is disabled
- Preventing movement above `nether-roof-min-y` if nether roof prevention is enabled

### `custom-blocks.yml` (custom block allow/deny list)

This file is created/maintained by AWC.

Structure:

- `default-enabled` (boolean, default `true`)
- `blocks.<customId>` => `true|false`

How IDs are detected:

- **ItemsAdder**: `dev.lone.itemsadder.api.CustomBlock` reflection hook
- **Oraxen**: `io.th0rgal.oraxen.api.OraxenBlocks` reflection hook
- **Nexo**: `com.nexomc.nexo.api.NexoBlocks` reflection hook
- **PDC scanning on placement**: reads string IDs from `custom-blocks.pdc-keys`

If no supported custom-block plugin is present and you don't configure PDC keys, custom blocks will effectively be ignored.

### `weapons.yml` (weapon restrictions)

Location: `plugins/AnankeWorldControl/weapons.yml`

This file controls weapon restrictions, including disabling specific weapons and limiting the number of weapons a player can have in their inventory.

#### Auto-population

The file is automatically populated with all weapon materials found in the game. On first run, it will be created with default values for each weapon type.

#### Format

```yaml
# Global defaults for all weapons
defaults:
  enabled: true  # Whether weapons are enabled by default
  limit: 0       # Maximum number of this weapon type a player can have (0 = unlimited)

# Per-weapon overrides
weapons:
  DIAMOND_SWORD:
    enabled: true   # Whether this weapon can be used
    limit: 1        # Maximum number of this weapon a player can have (0 = unlimited)
  
  BOW:
    enabled: true
    limit: 1
    
  # ... other weapons follow the same pattern
```

#### Enforcement

1. **Disabled Weapons**:
   - Players cannot attack with disabled weapons
   - Players cannot shoot projectiles from disabled ranged weapons (bows, crossbows, tridents)
   - Players cannot pick up or craft disabled weapons
   - Attempting to use a disabled weapon shows a warning message

2. **Weapon Limits**:
   - Players cannot pick up weapons that would exceed their limit
   - Players cannot craft weapons that would exceed their limit
   - Players cannot use shift-click or drag to move weapons into their inventory if it would exceed limits
   - A periodic sweep ensures limits are enforced even for admin-given items

3. **Strict Inventory Enforcement**:
   - Prevents bypassing limits through:
     - Shift-clicking from containers
     - Dragging items
     - Hotbar swapping
     - Any other inventory manipulation

#### Commands

- `/awc reload` - Reloads the weapons configuration

## Practical examples

### Disable TNT and creeper block damage

1. Disable block placement/use of TNT:

   - `/awc block TNT false`

2. Disable creeper explosions:

   - `/awc feature creeper-explosions false`

### Disable redstone

- `/awc feature redstone false`

This forces redstone power to stay at 0 in `BlockRedstoneEvent`.

### Lock time to day

In `config.yml`:

```yaml
environment:
  time-lock: 1000
```

This makes AWC set the time every second.

### Disable crafting golden apples

- `/awc craft GOLDEN_APPLE false`

### Disable Nether access but allow Overworld

- `/awc dimension nether false`

If players try to use Nether portals into a Nether world, they’ll be blocked.

## Troubleshooting

### “It changed back after restart”

- Make sure you’re editing the files in `plugins/AnankeWorldControl/` and not `src/main/resources/`.

### Feature toggle doesn’t seem to do anything

- Confirm the feature key matches what the plugin checks.
  - `/awc feature ...` only accepts a fixed set of known keys.
  - Some listeners also check keys that are *not* present in the default config; you can add them manually.

### Custom blocks aren’t being blocked

- Ensure `custom-blocks.enabled: true`.
- Ensure you have one of:
  - ItemsAdder/Oraxen/Nexo installed (AWC detects them automatically), or
  - `custom-blocks.pdc-keys` configured so AWC can read a string ID from item metadata.

### Nether roof prevention is too strict

- Adjust `defaults.nether-roof-min-y` or per-world `worlds.<name>.nether-roof-min-y` in `dimensions.yml`.
- Or disable it:
  - `/awc dimension nether-roof false`
