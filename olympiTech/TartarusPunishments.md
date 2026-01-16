# Tartarus Punishments

## Overview

Tartarus Punishments is a lightweight punishment plugin for Bukkit/Paper/Purpur/Spigot servers.

It focuses on a simple workflow:

- You define **reason presets** in `config.yml` (duration, punishment type, staff permission level, inventory clearing)
- Staff run commands like `/punish`, `/forgive`, `/prison`, `/screenshare`
- The plugin stores punishment state in a local SQLite database (`plugins/TartarusPunishments/bans.db`)
- Players are blocked/kicked at the right times (pre-login ban checks, mute checks, prison freeze)

Supported punishment types in code:

- **BAN** (UUID ban + optional IP ban)
- **MUTE** (blocks chat + command usage; only used if you define a reason preset with `punishmentType: mute`)
- **PRISON** (teleport + freeze + return location)

There is also a **Screenshare** feature that freezes a player and automatically bans them if they log out during the screenshare.

## Requirements

- **Minecraft server**: Bukkit/Paper/Purpur/Spigot
- **Minecraft version**: 1.21.x (plugin `api-version: 1.21`)
- **Java**: 21+ (as per project description)

## Installation

1. Place the jar in your server `plugins/` folder.
2. Start/restart the server.
3. Edit `plugins/TartarusPunishments/config.yml` to match your rules.
4. Set the prison location in-game (recommended):

   - `/setprison`

## How it works (Mental Model)

### Startup

On enable (`tartarus.punishments.Punishments#onEnable`):

- Loads and potentially auto-migrates config defaults (`ConfigManager#loadConfig`)
- Initializes `DatabaseManager` (SQLite + schema migrations)
- Initializes:
  - `FreezeManager` (prison/screen-share freeze mechanics)
  - `PlayerJoinListener` (ban checks + applying prison state on join)
  - `ScreenshareManager` (screenshare state tracking + reject ban)
  - `MuteListener` (chat/command blocking)
- Registers commands and listeners

### Where punishment data lives

- **Persistent state**: SQLite file at `plugins/TartarusPunishments/bans.db`
  - Primary tables:
    - `bans` (stores all punishment types: ban/mute/prison)
    - `ip_bans` (optional IP bans)
    - `last_seen_ip` (tracks last IP for offline bans)

### Enforcing bans (pre-login)

`PlayerJoinListener#onPlayerPreLogin` runs during `AsyncPlayerPreLoginEvent`:

- Records last seen IP (`last_seen_ip`)
- If UUID is banned:
  - denies login with a formatted multi-line ban message
- Else if IP is banned:
  - denies login with the ban message

This prevents the player from joining at all.

### Enforcing mutes

`MuteListener` blocks:

- Chat (`AsyncPlayerChatEvent`)
- Commands (`PlayerCommandPreprocessEvent`)

If the player has an active `MUTE` record in the database.

### Prison (freeze + teleport + return)

- Prison is stored as a `PRISON` punishment in the database.
- If the player is online when imprisoned:
  - the plugin stores their current location as the **return location**
  - teleports them to the configured prison location
  - freezes them via `FreezeManager`

When the prison expires:

- The DB marks prison records as `pending_release` when they expire
- On next join (or periodic cleanup), the plugin returns the player to their stored return location.

### Screenshare

Screenshare is not stored in the database; it is tracked in memory and via scoreboard tags:

- Starting a screenshare adds scoreboard tag: `tartarus_screenshare_active`
- Player is frozen via `FreezeManager`
- If the player logs out while screensharing:
  - `ScreenshareManager` applies the configured `screensharereject` punishment

## Commands and permissions

Commands are defined in `plugin.yml`.

### `/punish <player> <reason>`

- **Aliases**: `/ban`, `/banplayer`
- **Base permission**: `tartarus.punishments.punish`
- Applies the reason preset (ban/mute/prison) from `config.yml`.
- Uses async player lookup + async DB checks.
- Also broadcasts a message to the server.

Important: each reason preset can specify its own permission level (admin/moderator/helper/custom), and the command enforces that.

### `/forgive <player>`

- **Aliases**: `/unban`, `/pardon`
- **Permission**: `tartarus.punishments.forgive`
- Clears the player’s punishment record.
- If the player is in prison and online, returns them to origin and unfreezes.
- If the player is in a screenshare, it also clears the screenshare state.

### `/banlist`

- **Aliases**: `/bans`
- **Permission**: `tartarus.punishments.banlist`
- Lists all banned players.
- Each entry is clickable to run `/forgive <player>`.

### `/prison <player> <duration>`

- **Permission**: `tartarus.punishments.prison`
- Imprisons a player for a custom duration.
- Stores return location (if the player is online).

### `/setprison`

- **Permission**: `tartarus.punishments.setprison`
- Sets the prison teleport location to your current position.

### `/screenshare <player>`

- **Permission**: `tartarus.punishments.screenshare`
- Player must be online.
- Applies a screenshare freeze.
- If they log out during the screenshare, the `screensharereject` punishment is applied.

### `/checkpunishments <player>`

- **Aliases**: `/checkp`, `/pcheck`, `/history`, `/phistory`
- **Permission**: `tartarus.punishments.check`
- Displays a comprehensive breakdown of:
    - **Global Strikes**: Current count / Threshold.
    - **Per-Reason History**: How many times they've violated a specific rule.
    - **Scaling Status**: Shows their current "Step" in Fractal mode or progress in Linear mode.
- **Punishment Scaling**:
  - The plugin automatically tracks punishment history and can escalate penalties based on your configuration (e.g., Strike 1 -> Mute, Strike 2 -> Ban).

## Punishment Scaling

Tartarus Punishments supports three powerful scaling modes to automate discipline:

### 1. Global Linear Scaling
*   **Disabled by default**.
*   Configured in `config.yml` under `punishment-scaling.global-linear`.
*   A "Three Strikes" style system.
*   **How it works**: Every time a player receives **any** punishment (warn, mute, prison, etc.), their global strike count increases. If they hit the `threshold`, the configured `action` (e.g., Ban) is applied for the `duration` (e.g., Permanent).

### 2. Linear Mode (Per Reason)
*   Configured inside a specific `ban-reason`.
*   **Example**: "Spamming".
*   If `mode: linear`, the plugin counts how many times the player has been punished for *this specific reason*.
*   If they hit the `linear-threshold`, the `linear-action` (e.g., Ban) is applied instead of the default punishment.

### 3. Fractal Mode (Per Reason)
*   The most advanced mode.
*   Configured inside a specific `ban-reason` by setting `mode: fractal`.
*   You define a list of `steps`.
*   **Example**:
    *   Step 1: 1 hour Mute
    *   Step 2: 12 hour Mute
    *   Step 3: 7 day Ban
    *   Step 4: Permanent Ban
*   The plugin automatically checks the player's history for this reason and applies the next step in the ladder.

## Integrations

### JanusMCD
Tartarus Punishments automatically detects if **JanusMCD** is installed.
- It registers a **Ban Checker** with JanusMCD.
- If you use JanusMCD's "Linked Accounts" feature, banning one account via Tartarus will automatically prevent all other linked accounts from joining the server (if JanusMCD is configured to enforce linked bans).

## Configuration reference (`config.yml`)

Config file path on a live server:

- `plugins/TartarusPunishments/config.yml`

### `ban-reasons`

This is the core of the plugin.

Each key under `ban-reasons` defines a preset reason you can use with `/punish <player> <key>`.

Each reason supports:

- **`duration`** (string)
  - Parsed by `DurationParser`.
  - Supports:
    - `s`, `m`, `h`, `d`, `w` (seconds/minutes/hours/days/weeks)
    - Chained values like `1h30m`
    - Special value: `perma` => permanent

- **`punishmentType`** (string)
  - `ban`, `mute`, `prison` (case-insensitive)

- **`inventoryClear`** (bool)
  - Only meaningful for bans.
  - If true, clears the player’s inventory when banned.
  - The plugin tracks inventory clear state in the DB to avoid re-clearing.

- **`reason`** (string)
  - Display reason.

- **`customReason`** (string)
  - If set, this becomes the displayed reason (`getMessageReason`).

- **`permission`** (string)
  - Controls who can use this specific reason.
  - Mapped via `ConfigManager.PunishmentReason#getFullPermissionNode()`:
    - `admin` => `tartarus.punishments.admin`
    - `moderator` => `tartarus.punishments.moderator`
    - `helper` => `tartarus.punishments.helper`
    - any other non-empty value is treated as a direct permission node
  - If empty/missing, the plugin logs a warning and defaults to `tartarus.punishments.punish`.

Example reason:

```yaml
ban-reasons:
  spam:
    duration: 12h
    customReason: ""
    permission: helper
    # Scaling Options (Optional)
    mode: "default" # "default", "linear", or "fractal"
    # Example Fractal Config:
    # mode: "fractal"
    # steps:
    #   - [mute, 1h]
    #   - [mute, 12h]
    #   - [ban, 7d]

```

### `ban-message`

This controls the kick/deny screen the user sees when banned.

#### `ban-message.lines`

List of lines. Each entry can be either:

- **Map format**:
  - `text`: string
  - `color` (or `colour`): named color (`red`, `gray`, etc.) or legacy code (`&c`, `§c`)
- **Plain string format**:
  - treated as legacy string and supports multiple color codes in the same line

Supported placeholders:

- `{player}`
- `{reason}`
- `{duration}`
- `{banner}`

#### `ban-message.inventory-cleared-line`

An optional line appended if `inventoryClear: true` for that ban reason.

Example:

```yaml
ban-message:
  lines:
    - text: "You are banned from this server!"
      color: red
    - "&7Reason: &f{reason}"
    - "&7Duration: &f{duration}"
    - "&7Banned by: &f{banner}"
  inventory-cleared-line:
    text: "Your inventory has been cleared."
    color: red
```

### `screenshare`

Controls screenshare behavior.

- **`screenshare.timeout-minutes`** (int)
  - Minimum enforced is 1.
- **`screenshare.message`** (string)
  - Sent to the player when screenshare starts.
  - Supports `{minutes}` placeholder.

#### `screenshare.reject`

Defines the special punishment applied when a player logs out during a screenshare.

Keys:

- `duration`
- `punishmentType`
- `inventoryClear`
- `reason`
- `customReason`
- `permission`

These are automatically loaded into a synthetic ban-reason key called:

- `screensharereject`

So you should not create `ban-reasons.screensharereject` anymore (the config manager migrates old configs).

Example:

```yaml
screenshare:
  timeout-minutes: 5
  message: "You have {minutes} minutes to join VC and do a screenshare check with an admin"
  reject:
    duration: perma
    punishmentType: ban
    inventoryClear: true
    reason: User refused to screenshare
    customReason: ""
    permission: admin
```

### `prison.location`

This is written automatically by `/setprison`.

- `prison.location.world`
- `prison.location.x`
- `prison.location.y`
- `prison.location.z`
- `prison.location.yaw`
- `prison.location.pitch`

If this is not set, imprisoning will still store the prison record, but players might not be teleported anywhere.

## Examples

### Add a mute reason

The default `config.yml` may not include any mute presets. To use mutes, add your own preset like the following.

```yaml
ban-reasons:
  chatspam:
    duration: 30m
    punishmentType: mute
    inventoryClear: false
    reason: Chat spam
    customReason: ""
    permission: helper
```

Staff can then run:

- `/punish SomePlayer chatspam`

### Add a prison reason

```yaml
ban-reasons:
  prison10m:
    duration: 10m
    punishmentType: prison
    inventoryClear: false
    reason: Temporary prison
    customReason: ""
    permission: moderator
```

## Troubleshooting

### Player isn’t being teleported when imprisoned

- Run `/setprison` to set `prison.location.*`.
- Ensure the configured world exists and is loaded.

### A reason key says “Invalid reason”

- The reason key is the YAML key under `ban-reasons`.
- Example: `ban-reasons.spam` -> use `/punish <player> spam`.

### “Player not found”

- The plugin uses async offline lookup. Ensure the spelling is correct.
- If the player never joined, some server implementations may not have an offline profile.

### IP bans seem too aggressive

- The plugin applies an IP ban automatically for BAN punishments when it can resolve an IP.
- It will use the online IP if the player is online, otherwise it uses `last_seen_ip`.

### Prison expires but player stays frozen

- Prison expiry is processed by periodic cleanup and by join/online checks.
- If it’s stuck, use `/forgive <player>` to force-clear the punishment and unfreeze.

