# JanusMCD

## Overview

JanusMCD is a Minecraft server plugin (Spigot/Paper 1.16.5+) that bridges chat and operational events between Minecraft and Discord.

It provides:

- **Two-way chat relay** (Minecraft <-> Discord) with **Webhook Impersonation** support
- **Multi-channel sync** (one Minecraft server to multiple Discord channels, with optional cross-Discord relay)
- **Console relay + command execution** (optional)
- **Filtering / anti-spam** controls for Discord-originated messages
- **Server status embeds** (auto-updating online/offline + players + TPS)
- **Account linking + auth flows** (DM-based linking + login protections)
- **Advancement and death notifications**
- **Stealth Vanish** (Packet-level hiding via ProtocolLib for true invisibility + Silent Chests)
- **Proximity Voice Chat** (Location-based Discord voice channels with smart clustering)

## Requirements

- **Minecraft server**: Spigot/Paper 1.16.5 or newer
- **Java**: 17+
- **Discord**: Bot token + permissions to read/send messages in configured channels
- **ProtocolLib**: (Highly Recommended) Required for packet-level vanish (Silent Chests, Entity Hiding, Tab-Complete protection).

## Installation (Server Owners)

1. Drop the JanusMCD jar into `plugins/`.
2. Start the server once to generate configs under `plugins/JanusMCD/`.
3. Edit your live configs in `plugins/JanusMCD/*.yml` (the files in `src/main/resources/` are only defaults bundled in the jar).
4. Set `discord-token` and at least one entry in `channel-ids`.
5. Restart the server (or use `/janusmcd reload` for supported hot-reload items).

## How JanusMCD works (Mental Model)

### Startup sequence

- `JanusMCD#onEnable()` initializes:
  - `DebugManager`
  - `ConfigManager` (loads multiple YAMLs and can watch for file changes)
  - `DatabaseManager`
  - feature managers (account linking, auth, freeze, vanish, advancements, death messages, etc.)
  - `DiscordManager` which boots JDA asynchronously.

### `account-linking.yml`
*   `account-linking.enabled`
*   `account-linking.require-verification`
*   `account-linking.limits.max_links_per_discord`
*   `logging.webhook`

### `voice.yml`
*   `enabled`
*   `category-id` (Discord Category ID for voice channels)
*   `lobby-channel-id` (Main "waiting room" channel)
*   `proximity-radius` (Distance to hear others)


### Config loading and reloads

JanusMCD uses a multi-file config model:

- `config.yml` (main settings)
- `account-linking.yml`
- `advancements.yml`
- `deathmessages.yml`
- `debug.yml`
- `vanish.yml`

The plugin supports:

- **Manual reload** via `/janusmcd reload`
- **File watch reload** via `ConfigWatcher` (debounced ~1s)

Important: not every setting is safe to hot-reload; some features read values at startup, others re-check config every time they run.

### Discord <-> Minecraft relay

- Discord -> MC: messages in configured `channel-ids` are filtered (`filterMessage`) and broadcast to online players.
- MC -> Discord: chat messages are formatted and sent to all `channel-ids`.
- Optional: **cross-discord-sync** will re-post a message received in one Discord relay channel into the other relay channels.

### Console integration

- If enabled, messages in `console-channel-ids` can be treated as server console commands.
- The plugin can also relay console output/logs to Discord.
- Sensitive commands are masked before logging (hardcoded for safety).

### Status embeds

Status embeds are configured in `config.yml` as a list under `status-embeds`.

- The plugin posts/edits an embed message in each configured channel.
- The embed includes server uptime, player list (limited), and TPS (if available).
- **Message IDs are persisted into `data.yml`**, so the plugin can keep editing the same message without rewriting your `config.yml` comments.

## Commands and permissions

### `/janusmcd reload`

- **Permission**: `janusmcd.reload`
- Reloads config files and updates many in-memory settings.

### `/freeze <player>`

- **Permission**: `janusmcd.freeze`

### `/vanish`

- **Permission**: `janusmcd.vanish`
- See vanished players: `janusmcd.vanish.see`
- Bypass pickup restriction: `janusmcd.vanish.no-pickup.bypass`
- Bypass interact restriction: `janusmcd.vanish.interact`
- Other permissions:
    - `janusmcd.vanish.chat`: Allow chatting while vanished (if chat is otherwise blocked).
    - `janusmcd.vanish.reload`: Reload vanish configuration.
    - `janusmcd.vanish.other`: Toggle vanish for other players.

### `/voice`

> **Note**: There is no `/voice` command. Proximity Voice Chat works automatically by moving players in Discord based on their in-game location.

### Link Management (Slash Commands)

These commands are available as Discord Slash Commands (e.g., `/link status`).
> **Multi-Server Support**: These commands are registered on **all** Discord servers the bot is in, but require strictly enforced permissions.

*   **`/link status <user>`**
    *   **Permission**: `MANAGE_SERVER` (Admin Only)
    *   Checks the account link status for a Discord user.
*   **`/link remove <user> <uuid>`**
    *   **Permission**: `MANAGE_SERVER`
    *   Removes a specific Minecraft account link from a Discord user.
*   **`/link clear <user>`**
    *   **Permission**: `MANAGE_SERVER`
    *   Removes *all* Minecraft account links for a Discord user.

### Reporting

*   **`/report` (Slash Command)**
    *   **Permission**: `MANAGE_SERVER` (Admin Only)
    *   Generates a visual "Server Metrics Report" chart showing DAU (Daily Active Users), retention rates, average session length, and online player trends.



## Configuration reference

All configs live in `plugins/JanusMCD/`.

### `config.yml` (main)

#### Discord

- **`discord-token`** (string, required)
  - Discord bot token.
  - Use a placeholder in shared configs and never commit your real token.

- **`channel-ids`** (list mixed)
  - Supports a mixed list of simple **Strings** (Legacy Bot Message) and **Objects** (Advanced Webhook).
  - **String format**: `"123456789"`
  - **Object format**:
    ```yaml
    - id: "123456789"
      webhook-url: "https://discord.com/api/webhooks/..."
      mode: "DISCORD" # MINECRAFT, DISCORD, HYBRID_MINECRAFT, HYBRID_DISCORD
    ```
  - **Modes**:
    - `MINECRAFT`: Uses player's IGN and Skin.
    - `DISCORD`: Uses linked Discord user's Display Name and Avatar (fallback to MC if unlinked).
    - `HYBRID_MINECRAFT`: MC identity + footer `Linked as @DiscordUser`.
    - `HYBRID_DISCORD`: Discord identity + footer `IGN: PlayerName`.

- **`cross-discord-sync`** (bool, default `true`)
  - If `true`, a message received in one relay channel is re-posted to the other relay channels.

- **`console-channel-ids`** (list of strings)
  - Discord channels treated as “console control” channels.
  - Supports legacy single key `console-channel-id` (string) for backward compatibility.

- **`allow-console-commands`** (bool, default `true`)
  - If enabled, messages in console channels can be executed as server console commands.

#### Status embeds

`status-embeds` is a list of objects. Each entry supports:

- **`enabled`** (bool, default `true`)
- **`channel-id`** (string)
- **`message-id`** (string)
  - You can leave this empty; JanusMCD will create the embed and store the resulting ID into `data.yml`.
- **`server-name`** (string)
- **`update-interval`** (int seconds, default `60`)
- **`detailed-info`** (bool, default `true`)
- **`online-emoji`** (string, default `🟢`)
- **`offline-emoji`** (string, default `🔴`)

Example:

```yaml
status-embeds:
  - enabled: true
    channel-id: "123456789012345678"
    message-id: ""
    server-name: "Survival"
    update-interval: 60
    detailed-info: true
    online-emoji: "🟢"
    offline-emoji: "🔴"
```

#### Filtering

- **`filtering.enabled`** (bool, default `true`)
- **`filtering.banned-words`** (list of strings)
- **`filtering.banned-patterns`** (list of regex strings)
  - Regex patterns are compiled case-insensitive; invalid patterns are logged and ignored.
- **`filtering.action`** (string, default `block`)
  - Observed options: `block`, `replace`, `log`.
- **`filtering.replace-with`** (string, default `***`)
- **`filtering.block-urls.enabled`** (bool, default `true`)
- **`filtering.block-urls.action`** (string, default `replace`)
- **`filtering.block-urls.replace-with`** (string, default `[URL removed]`)

Example:

```yaml
filtering:
  enabled: true
  banned-words:
    - slur1
    - slur2
  banned-patterns:
    - "(?i)free\s+nitro"
  action: "block"
  replace-with: "***"
  block-urls:
    enabled: true
    action: "replace"
    replace-with: "[URL removed]"
```

#### Anti-spam

The default config includes an `anti-spam` section. Its runtime behavior depends on the implementation in the plugin (rate-limiting is present in the auth subsystem; chat anti-spam may be implemented elsewhere).

- **`anti-spam.enabled`** (bool)
- **`anti-spam.max-messages`** (int)
- **`anti-spam.time-period`** (int seconds)
- **`anti-spam.cooldown`** (int seconds)
- **`anti-spam.notify-user`** (bool)
- **`anti-spam.notify-message`** (string; supports `{cooldown}`)

#### Command logging

- **`command-logging.enabled`** (bool, default `true`)
- **`command-logging.channel-ids`** (list of strings)
  - Legacy: `command-logging.channel-id` (string).
- **`command-logging.embed-color`** (string hex, default `#3498db`)

If no command channels are configured, JanusMCD falls back to `console-channel-ids`.

#### Join/leave messages

- **`join-leave-messages.use-embeds`** (bool)
- **`join-leave-messages.channel-id`** (string; empty uses first `channel-ids`)
- **`join-leave-messages.colors.join`** (hex)
- **`join-leave-messages.colors.leave`** (hex)
- **`join-leave-messages.show-player-avatar`** (bool)
- **`join-leave-messages.avatar-url`** (string template)
  - Supports `{uuid}` and `{name}`.
- **`join-leave-messages.avatar-location`** (string, default `thumbnail`)
  - Controls where the avatar appears in the embed.
  - Options: `thumbnail`, `author`, `both`, `none`.

#### Server notifications

- **`server-notifications.channel-ids`** (list; empty means “all chat channels”)
  - Legacy: `server-notifications.channel-id`.
- **`server-notifications.startup.enabled`** (bool)
- **`server-notifications.startup.message`** (string)
- **`server-notifications.shutdown.enabled`** (bool)
- **`server-notifications.shutdown.message`** (string)

### `account-linking.yml`

This governs the DM-based account linking flow.

- **`account-linking.enabled`** (bool)
  - Master toggle. When disabled, enforcement steps are skipped.
- **`account-linking.require-verification`** (bool)
  - If `true`, linking requires a PIN/code verification.
- **`account-linking.limits.max_links_per_discord`** (int; enforced to 1..3)
  - The plugin clamps values to 1..3 and may rewrite the config to remove legacy keys.
- **`code-expiry-minutes`** (int)
  - Intended expiry for verification codes.
- **`cooldown-seconds`** (int)
- **`link-message`**, **`invalid-code-message`**, **`success-message`**, **`already-linked-message`** (strings)

Logging:

- **`logging.log-channel`** (legacy string channel id)
- **`logging.webhook`** (string URL)
- **`logging.log-auth-success`**, `log-auth-failures`, `log-link-created`, `log-link-removed`, `log-link-cleared` (bool)
- **`logging.messages.*`** (strings)
  - Supports `{player}`, `{uuid}`, `{discord_id}`, `{discord_tag}`.

Example:

```yaml
account-linking:
  enabled: true
  require-verification: true
  limits:
    max_links_per_discord: 1

code-expiry-minutes: 10
cooldown-seconds: 60

logging:
  webhook: "https://discord.com/api/webhooks/..."
  log-link-created: true
  messages:
    link-created: "🔗 **{player}** linked to <@{discord_id}>"
```

### `voice.yml`

Manages the Proximity Voice Chat (PVC) feature.

- **`enabled`** (bool)
- **`category-id`** (string, required)
  - ID of the Discord **Category** where temporary voice channels will be created.
  - Bot must have `Manage Channels`, `Move Members`, and `Mute Members` permissions here.
- **`lobby-channel-id`** (string, required)
  - ID of the static "Waiting Room" voice channel.
  - Players must be in this channel to be picked up by the proximity system.
- **`proximity-radius`** (int blocks, default `20`)
  - Distance within which players can hear each other.
- **`update-interval`** (int seconds, default `5`)
  - How frequently clustering checks run.
- **`mute-on-death`** (bool, default `true`)
  - If `true`, players are server-muted in Discord while dead in-game.
- **`channel-name-format`** (string, default `Voice Group %d`)
  - Template for temporary channel names.

**How it works:**
1. Players join the **Lobby Channel** in Discord (configured in `voice.yml`).
2. JanusMCD tracks their in-game location.
3. If players are within `proximity-radius` of each other, the bot **automatically** moves them into a temporary **Voice Group** channel.
4. When they move apart, they are returned to the Lobby or reassigned to new groups.

> **Note**: This feature is entirely automated. There are no in-game commands for players to run.

### `vanish.yml`

Now powered by **ProtocolLib** for deep stealth.

- **`enabled`** (bool)
- **`vanish-on-join`** (bool)

Suppression:
- **`suppression.ingame-messages`** (bool): Hides Join/Quit messages.
- **`suppression.discord-messages`** (bool): Hides Discord relay join/quit messages.
- **`suppression.advancements`** (bool): Silences advancement announcements.
- **`suppression.death-messages`** (bool): Hides death messages.

Interactions:
- **`interactions.prevent-item-pickup`** (bool)
- **`interactions.prevent-mob-targeting`** (bool)
- **`interactions.prevent-block-interaction`** (bool): Stops chest opening, button pressing, etc.
- **`interactions.prevent-damage`** (bool): God mode while vanished.
- **`interactions.prevent-physical-interactions`** (bool): Tripwires, pressure plates, crops.

**Silent Chests**:
When ProtocolLib is installed, opening chests/barrels/shulkers while vanished does NOT trigger opening animations or sounds for other players.

- **`action-bar-message`** (string)
  - Message shown to the vanished player (e.g., "&b&lVANISHED"). Set to empty to disable.

- **`enabled`** (bool)
- **`channel-id`** (string; empty uses first configured chat channel)

Filtering:

- **`included-namespaces`** (list)
- **`excluded-namespaces`** (list)
- **`excluded-advancements`** (list of full keys, e.g. `minecraft:story/root`)

Formatting:

- **`challenge-format`**, **`goal-format`**, **`task-format`** (strings)
  - Supports `{player}`, `{advancement}`, and (in code paths) `{description}`, `{progress}`.

Embed:

- **`embed.preset`**: `achievement` | `compact` | `detailed`
- **`embed.color`**: named (`BLUE`, `GREEN`, ...) or hex (`#00FF00`)
- **`embed.show_fields`** (bool)

### `deathmessages.yml`

- **`enabled`** (bool)
- **`channel-id`** (string; empty uses first chat channel)
- **`use-embed`** (bool)
- **`message-format`** (string; used when `use-embed: false`)
  - Supports `{player}`, `{message}`.

### `debug.yml`

Controls the plugin’s `DebugManager` verbosity.

- **`debug.enabled`** (bool)
- **`debug.level`** (string)
  - `SEVERE`, `WARNING`, `INFO`, `CONFIG`, `FINE`, `FINER`, `FINEST`
- **`debug.features.chat-event-debugging`** (bool)
- **`debug.features.auth`** (bool)
- **`debug.format.timestamp`**, **`debug.format.show-thread`**, **`debug.format.show-source`** (bool)
- **`modules.*.enabled`**, **`modules.*.level`** (per-module overrides)

### `vanish.yml`

- **`enabled`** (bool)
- **`vanish-on-join`** (bool)

Suppression:

- **`suppression.ingame-messages`** (bool)
- **`suppression.discord-messages`** (bool)
- **`suppression.advancements`** (bool)
- **`suppression.death-messages`** (bool)

Interactions:

- **`interactions.prevent-item-pickup`** (bool)
- **`interactions.prevent-mob-targeting`** (bool)
- **`interactions.prevent-block-interaction`** (bool)
- **`interactions.prevent-damage`** (bool)

- **`action-bar-message`** (string)
  - Set to empty string to disable the action bar message.

> **Note**: For the most secure vanish experience (preventing tab-completion leaks, server list snooping, etc.), ensure **ProtocolLib** is installed. JanusMCD will automatically detect and use it for packet-level hiding.

## Examples

### Minimal working config

```yaml
discord-token: "REPLACE_ME"

channel-ids:
  - "123456789012345678"

cross-discord-sync: true

console-channel-ids: []
allow-console-commands: false
```

### Multi-server / multi-channel relay

```yaml
channel-ids:
  - "111111111111111111" # Discord guild A
  - "222222222222222222" # Discord guild B

cross-discord-sync: true
```

### Advanced Webhook Configuration

```yaml
channel-ids:
  # Standard Bot Message Channel
  - "111111111111111111"

  # Webhook Channel (Impersonating Discord User)
  - id: "222222222222222222"
    webhook-url: "https://discord.com/api/webhooks/..."
    mode: "DISCORD"
```

> **Warning**: When using Webhook configuration, you **MUST** use the `id: "CHANNEL_ID"` format.
>
> **Invalid YAML**:
> ```yaml
> - "123456789"
>   webhook-url: "..." # ❌ Syntax Error: Cannot add keys to a string item
> ```
>
> **Valid YAML**:
> ```yaml
> - id: "123456789"
>   webhook-url: "..." # ✅ Correct: Item is an object
> ```

## Troubleshooting

### “Discord token not set or invalid”

- Verify `discord-token` is correct and not surrounded by extra whitespace.
- Ensure the bot is invited to the server and has permissions.

### Messages don’t relay

- Check that channel IDs are correct (numeric snowflakes).
- Confirm the bot can read/send messages in those channels.
- Temporarily raise `debug.level` to `FINE` in `debug.yml`.

### Status embed keeps recreating

- If a message is deleted, JanusMCD will detect “unknown message” and recreate it.
- Message IDs are stored in `data.yml`; deleting that file will also force recreation.

### Reload doesn’t apply to a setting

- Some settings are only read at startup.
- Prefer a full server restart after significant changes.

### Skin Resolution & Offline Support

**Q: What if I use `HYBRID` or `DISCORD` mode but the player isn't linked?**
A: The webhook will automatically fall back to **Minecraft** mode (showing the player's IGN and Skin).

**Q: My server is in Offline/Cracked mode. Will heads work?**
A: **Yes**, fully supported via **SkinsRestorer**.
- JanusMCD detects SkinsRestorer (v14 and v15+) automatically.
- It uses the player's *Skin Name* (set via `/skin`) to fetch the avatar, ensuring the webhook matches their visual appearance, not their Steve/Alex UUID.
- Default `avatar-url` templates (e.g., `mc-heads.net`) support this out of the box.

