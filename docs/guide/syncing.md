# Registering and syncing

Slash and context menu commands have to be **registered** with Discord before they show up in the client. discord does this for you: on the first `READY`, it sends any commands that changed since the last run. This is called *syncing*.

Prefix-only commands are never registered. They exist only inside your bot.

## How syncing works

1. Each command is converted to the JSON Discord expects.
2. Commands are grouped by **scope**: `global`, or a guild id (from `guilds = { ... }` or `devGuild`).
3. For each scope, discord hashes the JSON (SHA-256) and compares it with the hash saved last time.
4. If the hash changed, the scope is **bulk-overwritten**: every command in it is replaced in one request. The log says so:

    ```text
    [bot] synced 12 command(s) to global scope
    [bot] synced 3 command(s) to guild 123456789012345678
    ```

5. If nothing changed, nothing is sent. discord still fetches the scope's command list to learn the command ids, which it uses for [command mentions](slash-commands.md#mentioning-a-command).
6. A scope that had commands last time but has none now is cleared. If you remove `devGuild`, the leftover guild copies are deleted on the next start.

Hashes are saved in `.commands.lock.json` in the project root, keyed by application id. In a built program, the project root is the folder that holds the executable. Add the file to `.gitignore`. It only reflects what was synced from this machine.

## Options

| Bot option | Effect |
| --- | --- |
| `syncCommands = true` (default) | Sync changed scopes on the first `READY` |
| `syncCommands = "force"` | Overwrite every scope on every start |
| `syncCommands = false` | Never sync automatically |
| `commandCache = "path.json"` | Where to keep the hashes |
| `commandCache = false` | Don't keep hashes, so every start syncs every scope |
| `devGuild = "id"` | Put every command in this guild instead of its usual scope |

## Global commands vs. guild commands

| | Global | Guild |
| --- | --- | --- |
| Where | Every server with the bot, and DMs | One server |
| Updates | Can take a moment to show up. Restart the client (++ctrl+r++) if they don't. | Instant |
| Use for | Production | Development, or server-specific commands |

While developing, set `devGuild` to your test server:

```luau
local bot = discord.new({
    devGuild = process.env.DEV_GUILD, -- nil in production -> global
})
```

To register a command in particular servers only, give it `guilds`:

```luau
bot:command({ name = "staff", description = "Staff tools", guilds = { "111", "222" }, run = ... })
```

## Syncing by hand

Turn automatic syncing off and call `bot:syncCommands` yourself, for example from a deploy script:

```luau
local bot = discord.new({ syncCommands = false })
bot:load("commands")

bot:once("READY", function()
    bot:syncCommands({ force = true })
    bot:stop()
end)
bot:run()
```

`bot:syncCommands` needs the application id, which is known after `READY`. You can also pass `applicationId` to `discord.new`, which lets you sync without connecting to the gateway at all:

```luau
local bot = discord.new({ applicationId = "123456789012345678", syncCommands = false })
bot:load("commands")
bot:syncCommands({ force = true })
```

## Removing commands

```luau
bot:clearCommands()                       -- every global command
bot:clearCommands("123456789012345678")   -- every command in one guild
```

!!! warning
    `clearCommands` doesn't update `.commands.lock.json`. If you clear commands by hand, the next start may see unchanged hashes and skip the sync. Start once with `syncCommands = "force"`, or delete the lock file.

## Lower-level access

`api.commands` wraps the application command endpoints directly:

```luau
local list = ctx.api.commands.list()               -- global
local list = ctx.api.commands.list(guildId)        -- one guild
ctx.api.commands.create({ name = "x", description = "y", type = 1 }, guildId)
ctx.api.commands.edit(commandId, { description = "new" })
ctx.api.commands.delete(commandId)
ctx.api.commands.set(listOfCommandJSON, guildId)   -- bulk overwrite
ctx.api.commands.permissions(guildId)              -- per-guild permission overrides
```

`discord.commands` isn't exported, but `require("@discord/lib/commands").toJSON(definition)` turns a normalized definition into the JSON Discord expects. The normalized definitions are in `bot.commandList`.

## When commands don't show up

See [Troubleshooting](../help/troubleshooting.md#commands-dont-show-up).
