# The bot

Everything starts with a `Bot`:

```luau
local discord = require("@discord")

local bot = discord.new({
    token = TOKEN,
    prefix = "!",
})
```

`discord.new` takes a table of [options](../reference/bot-options.md). Each one has a default, so the token is usually the only one you set. Here are the options you are most likely to change:

| Option | What it does |
| --- | --- |
| `token` | The bot token. Defaults to the `DISCORD_TOKEN` environment variable. |
| `prefix` | Prefix(es) for [prefix commands](prefix-commands.md): `"!"`, `{ "!", "?" }`, or a function. |
| `intents` | Which [gateway events](intents.md) to receive. |
| `devGuild` | Register every command in this server only, for instant updates while developing. |
| `owners` | User ids allowed to run `ownerOnly` commands. Defaults to the application's owner or team. |
| `allowedMentions` | Who messages may ping by default. Defaults to users only, without pinging the replied-to user. |

The full list is in [Bot options](../reference/bot-options.md).

## Registering things

A bot holds four kinds of definitions. Each can be registered directly on the bot, or loaded from files with [`bot:load`](loading.md):

```luau
bot:command({ name = "ping", description = "Pong", run = function(ctx) ctx:reply("Pong!") end })
bot:component({ customId = "vote", run = function(ctx, choice) ctx:update(`Voted {choice}`) end })
bot:modal({ customId = "report", run = function(ctx, fields) ctx:reply("Thanks!") end })
bot:on("GUILD_CREATE", function(ctx, guild) print("in", guild.name) end)
```

These helpers also come in shorthand forms that set the command type for you:

| Method | Registers |
| --- | --- |
| `bot:command(def)` / `bot:slash(def)` | A slash command (or a hybrid with `prefix = true`) |
| `bot:prefix(def)` | A prefix-only command (`type = "prefix"`) |
| `bot:userCommand(def)` | A user context menu command (`type = "user"`) |
| `bot:messageCommand(def)` | A message context menu command (`type = "message"`) |
| `bot:component(def)` | A component handler |
| `bot:modal(def)` | A modal handler |
| `bot:event(def)` | An event handler from a `{ event, run, once }` table |
| `bot:on(event, fn)` / `bot:once(event, fn)` | An event handler |

The same helpers exist on the module (`discord.command(def)`, `discord.component(def)`, ...). On the module they don't register anything: they tag the table so that [`bot:load`](loading.md) knows what it is.

Registering an invalid definition raises an error right away, so mistakes show up at startup. Examples are an uppercase slash command name, a missing `run`, a required option after an optional one, or two commands with the same name.

## Running

```luau
bot:start()
```

`bot:start()` connects to the gateway and blocks for as long as the bot runs. If Discord closes the connection for good (a bad token, or intents that aren't allowed), it exits the process with status 1.

To handle that case yourself, use `bot:run()`, which returns instead of exiting:

```luau
local ok, reason = bot:run()
if not ok then
    print("the bot stopped:", reason)
end
```

`bot:stop()` disconnects and makes `run` return `true`.

### What happens on startup

1. If the bot asks for privileged intents that the Developer Portal hasn't enabled, they are dropped with a warning (unless `strictIntents = true`). See [Intents](intents.md).
2. The bot connects to the gateway and identifies.
3. On the first `READY`:
    - `bot.user` and `bot.applicationId` are set.
    - The owners are loaded from the application (unless you passed `owners`).
    - Changed commands are [synced](syncing.md) (unless `syncCommands = false`).
    - `bot.ready` is set.
4. Events start arriving: `GUILD_CREATE` for each server, then messages, interactions, and so on.

If the connection drops, the gateway client resumes the session, and Discord replays the events missed in the meantime. If Discord makes the bot identify again instead, it sends another `READY`. The startup work above (owners, syncing, `bot.ready`) is not repeated, but handlers registered with `bot:on("READY", ...)` do run again. Use `bot:once` for one-time setup.

### Waiting until the bot is ready

`bot.ready` is a Sonata `task.event()`. It is set once the first `READY` has been handled, the owners are known and commands are synced:

```luau
local task = require("@sonata/task")

task.spawn(function()
    bot.ready:wait()
    print("commands are synced, ids:", bot.commandIds)
end)

bot:start()
```

`bot.ready:isSet()` tells you whether it already happened. The showcase example uses this to tell servers the bot joins apart from the servers it already was in at startup:

```luau
bot:on("GUILD_CREATE", function(ctx, guild)
    if ctx.bot.ready:isSet() then
        print(`joined {guild.name}`)
    end
end)
```

## Presence

Set the bot's status and activity once the gateway is connected, for example on `READY`:

```luau
bot:once("READY", function(ctx)
    ctx.bot:setStatus("online", "/help", "listening")  -- "Listening to /help"
end)
```

`bot:setStatus(status, text, type)`:

- `status`: `"online"`, `"idle"`, `"dnd"` or `"invisible"`.
- `text`: the activity text. Omit it to clear the activity.
- `type`: `"playing"`, `"streaming"`, `"listening"`, `"watching"`, `"competing"` or `"custom"` (the default, which shows only the text).

For anything else, send a raw presence with `bot:setPresence`:

```luau
bot:setPresence({
    status = "idle",
    activities = { { name = "the stars", type = 3 } }, -- Watching the stars
})
```

`setStatus` and `setPresence` need a live connection. Before the bot connects, they return `false, "not connected"`. To start with a presence, pass `presence` to `discord.new`, which sends it with the identify payload. Discord expects a complete presence object there:

```luau
local json = require("@sonata/json")

discord.new({
    presence = {
        status = "dnd",
        activities = { { name = "Custom Status", type = 4, state = "busy" } },
        afk = false,
        since = json.null,
    },
})
```

## Shared state

`bot.store` (also `ctx.store`) is an empty table shared by every handler. Use it for in-memory state such as settings, counters or caches:

```luau
bot:command({
    name = "count",
    description = "Count up",
    run = function(ctx)
        ctx.store.count = (ctx.store.count or 0) + 1
        ctx:reply(tostring(ctx.store.count))
    end,
})
```

`ctx.state` is a fresh table for each invocation. Middleware can fill it for the handler (see [Middleware](middleware.md)).

## Looking things up

The bot has lookups that check the [cache](cache.md) first and fall back to the API:

```luau
local user = bot:fetchUser("123")
local member = bot:fetchMember(guildId, "123")   -- with .user attached
local channel = bot:fetchChannel("456")
local guild = bot:fetchGuild(guildId)
```

Each one returns `nil` when the object can't be found. Contexts have shortcuts for these that default to the current server, channel and user. See [The context](context.md#lookups).

## Other useful fields

| Field | What it is |
| --- | --- |
| `bot.user` | The bot's user object (after `READY`) |
| `bot.applicationId` | The application id (after `READY`, or the `applicationId` option) |
| `bot.api` | [Endpoint wrappers](api.md) |
| `bot.rest` | The [REST client](../reference/rest.md) |
| `bot.cache` | The [cache](cache.md) |
| `bot.gateway` | The [gateway client](../reference/gateway.md). `bot.gateway.latency` is the heartbeat round trip in seconds. |
| `bot.intents` | The intent bitmask the bot connects with |
| `bot.startedAt` | Unix time `discord.new` was called |
| `bot.commandIds` | Slash command name → id, after syncing |
| `bot.commandList` | Every registered command, normalized, in order |

See the [Bot reference](../reference/bot.md) for the rest.

## Using the API without a bot

For scripts that only call the REST API, such as a one-off cleanup or a deploy step, skip the gateway entirely:

```luau
local discord = require("@discord")

local api = discord.makeApi(discord.Rest.new(TOKEN))
local me = api.users.me()
print(me.username)
api.channels.send("123456789012345678", "Deployed!")
```

Endpoints that need the application id (`api.commands`, `api.interactions`, `api.application.emojis`, ...) need a bot, because they read `bot.applicationId`. See [Calling the API](api.md).
