# Bot options

```luau
discord.new({
    token = TOKEN,
    -- ...
})
```

Every option is optional, except that a token must be given here or in `DISCORD_TOKEN`.

## Connection

| Option | Type | Default | Description |
| --- | --- | --- | --- |
| `token` | `string` | `DISCORD_TOKEN` env | The bot token. Trailing whitespace and a leading `Bot ` are removed. |
| `intents` | `{ string \| number } \| number` | see below | [Intents](../guide/intents.md): names, flags or a mask |
| `strictIntents` | `boolean` | `false` | Keep privileged intents the portal hasn't enabled (and fail with 4014) instead of dropping them |
| `presence` | `table` | | Presence sent with identify (a full presence object) |
| `gateway` | `boolean` | `true` | `false`: no gateway, HTTP interactions only (needs `http`) |
| `http` | `{ publicKey?, port?, host?, path? }` | | Start an [HTTP interactions](../guide/http-interactions.md) endpoint |
| `applicationId` | `string` | learned on `READY` | Needed before `READY` for `api.commands` / `syncCommands` |

Default `intents`: `Guilds`, `GuildMessages`, `DirectMessages`, plus `MessageContent` when `prefix` is set.

## Prefix commands

| Option | Type | Default | Description |
| --- | --- | --- | --- |
| `prefix` | `string \| { string } \| (message, bot) -> (string \| { string })?` | none | Prefix(es). The function may yield. |
| `mentionPrefix` | `boolean` | `true` | `@Bot command` works as a prefix |
| `caseInsensitive` | `boolean` | `true` | Prefixes and command names ignore case |
| `ignoreBots` | `boolean` | `true` | Ignore messages from bots |

## Commands

| Option | Type | Default | Description |
| --- | --- | --- | --- |
| `devGuild` | `string` | | Register **every** command in this guild instead of its usual scope |
| `syncCommands` | `boolean \| "force"` | `true` | Sync changed commands on the first `READY`; `"force"` always overwrites |
| `commandCache` | `string \| false` | `<project root>/.commands.lock.json` | Where sync hashes are kept; `false` disables the cache (always sync) |
| `defaultInstall` | `{ "guild" \| "user" }` | | `install` for commands that don't set it |
| `defaultContexts` | `{ "guild" \| "botDm" \| "privateChannel" }` | | `contexts` for commands that don't set it |
| `owners` | `{ string }` | the application's owner / team members | Users allowed to run `ownerOnly` handlers and exempt from cooldowns |

## Responses

| Option | Type | Default | Description |
| --- | --- | --- | --- |
| `allowedMentions` | `AllowedMentions` | `{ parse = { "users" }, repliedUser = false }` | Default for every message the bot sends |
| `autoDefer` | `boolean` | `true` | Defer commands that haven't responded after ~2.3 s |
| `errorMessage` | `string \| false` | `"Something went wrong while running this."` | Shown (ephemerally) when a handler errors; `false` to hide |

## Other

| Option | Type | Default | Description |
| --- | --- | --- | --- |
| `cache` | `boolean` | `true` | Fill the [cache](../guide/cache.md) from gateway events |
| `assets` | `(folder: string) -> { string }` | the root project's assets | Lists asset files for `bot:load` in [built programs](../getting-started/building.md#packages-that-load-their-own-folders) |
| `debug` | `boolean` | `false` | Print every gateway payload (tokens redacted), and log unhandled custom ids and load counts |
| `debugRest` | `boolean` | `false` | Print every REST request and rate limit wait |

## Example

```luau
local process = require("@sonata/process")

local bot = discord.new({
    token = process.env.DISCORD_TOKEN,
    prefix = { "!", "?" },
    intents = { "Guilds", "GuildMessages", "GuildMembers", "MessageContent" },
    devGuild = process.env.DEV_GUILD,
    owners = { "123456789012345678" },
    defaultInstall = { "guild" },
    allowedMentions = { parse = {} },   -- ping nobody by default
    errorMessage = "Oops! That broke. The developers have been told.",
    debugRest = process.env.DEBUG ~= nil,
})
```
