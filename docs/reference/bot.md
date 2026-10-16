# Bot

```luau
local bot = discord.new(options)
```

Created by [`discord.new`](module.md#discordnew). Options are listed in [Bot options](bot-options.md).

## Fields

| Field | Type | Description |
| --- | --- | --- |
| `bot.options` | table | The options, with defaults filled in (without the token) |
| `bot.rest` | [Rest](rest.md) | The REST client |
| `bot.api` | [api](api.md) | Endpoint wrappers bound to this bot |
| `bot.cache` | [Cache](cache.md) | The gateway cache |
| `bot.gateway` | [Gateway](gateway.md) | The gateway client |
| `bot.intents` | number | The intent bitmask (after disallowed intents are dropped by `run`) |
| `bot.applicationId` | string? | The application id: from options, or learned on `READY` / `application()` |
| `bot.user` | table? | The bot's user, after `READY` |
| `bot.owners` | `{ [string]: boolean }` | Owner user ids |
| `bot.store` | table | Shared state (`ctx.store`) |
| `bot.ready` | `task.Event` | Set once the first `READY` is handled, owners loaded and commands synced |
| `bot.startedAt` | number | Unix seconds when the bot was created |
| `bot.commandList` | `{ table }` | Normalized command definitions, in registration order |
| `bot.commands` | `{ [string]: table }` | Application commands by `"slash:name"`, `"user:name"`, `"message:name"` (lowercase) |
| `bot.prefixCommands` | `{ [string]: table }` | Prefix commands by name and alias (lowercase with `caseInsensitive`) |
| `bot.commandIds` | `{ [string]: string }` | Slash command name → id, after syncing |
| `bot.components` / `bot.componentPatterns` | | Registered component handlers by custom id / with `match` |
| `bot.modals` / `bot.modalPatterns` | | Registered modal handlers |

## Registering

### `bot:command` / `bot:slash`

```luau
bot:command(def: CommandDefinition) -> table
```

Registers a command (slash, hybrid, prefix, user or message, depending on `type` and `prefix`). Returns the normalized definition. Raises on invalid definitions and duplicates. See [Definitions](definitions.md#commanddefinition).

### `bot:prefix`, `bot:userCommand`, `bot:messageCommand`

```luau
bot:prefix(def) -> table          -- type = "prefix"
bot:userCommand(def) -> table     -- type = "user"
bot:messageCommand(def) -> table  -- type = "message"
```

### `bot:addCommand`

```luau
bot:addCommand(def: CommandDefinition, source: string?) -> table
```

What `bot:command` calls. `source` is a label (usually a file path) shown in duplicate-definition errors.

### `bot:component` / `bot:addComponent`

```luau
bot:component(def: ComponentDefinition) -> table
bot:addComponent(def: ComponentDefinition, source: string?) -> table
```

Registers a component handler. Needs `run`, and `customId` or `match`. A numeric `cooldown` becomes `{ seconds = n, scope = "user" }`. Raises on duplicates.

### `bot:modal` / `bot:addModal`

```luau
bot:modal(def: ModalDefinition) -> table
bot:addModal(def: ModalDefinition, source: string?) -> table
```

Registers a modal handler.

### `bot:on`

```luau
bot:on(event: string, run: (ctx: Context, data: any) -> ()) -> Bot
```

Adds an event handler. The name is normalized with [`eventName`](module.md#discordeventname). For `"*"`, `run` receives `(ctx, eventName, data)`. Prints a warning if the bot's intents don't deliver the event.

### `bot:once`

```luau
bot:once(event: string, run: (ctx: Context, data: any) -> ()) -> Bot
```

Like `on`, removed after the first call.

### `bot:off`

```luau
bot:off(event: string, run: function?)
```

Removes `run` from the event, or every handler for it when `run` is `nil`.

### `bot:event` / `bot:addEvent`

```luau
bot:event(def: EventDefinition)
bot:addEvent(def: EventDefinition, source: string?)
```

Registers `{ event, run, once? }`. Raises if `event` isn't a string or `run` isn't a function.

### `bot:use`

```luau
bot:use(middleware: (ctx: Context, next: () -> ()) -> ()) -> Bot
```

Adds [middleware](../guide/middleware.md) around command, component, modal and autocomplete handlers.

### `bot:onError`

```luau
bot:onError(handler: (err: string, ctx: Context?) -> ()) -> Bot
```

Adds an [error handler](../guide/middleware.md#error-handlers). With at least one, the default printing is skipped.

### `bot:add`

```luau
bot:add(value: any, source: string?)
```

Registers anything a module can return: a tagged or untagged definition, a list, a `{ commands, components, modals, events }` table, or a plugin function (called as `value(bot)`). See [Loading](../guide/loading.md#what-a-module-can-return).

### `bot:load`

```luau
bot:load(...: string | table | function) -> number
```

Loads files, folders and globs (relative to the calling file) and registers what they return. Tables and functions are passed to `add`. Returns the number of modules loaded. Raises with the file path on failure. See [Loading from files](../guide/loading.md).

## Running

### `bot:run`

```luau
bot:run() -> (boolean, string?)
```

Starts the HTTP endpoint if `http` is set, drops disallowed privileged intents (unless `strictIntents`), connects to the gateway and blocks. Returns `true` after `bot:stop()`, or `false, reason` when Discord closes the connection for good (bad token, disallowed or invalid intents, ...).

With `gateway = false`, it fetches the bot user and application over REST, runs the startup work, and blocks on the HTTP server instead. Raises if there's no HTTP endpoint.

### `bot:start`

```luau
bot:start()
```

`run`, then `process.exit(1)` if it returned `false`.

### `bot:stop`

```luau
bot:stop()
```

Closes the gateway connection (code 1000) and the HTTP server, if any.

### `bot:listen`

```luau
bot:listen(options: { publicKey: string?, port: number?, host: string?, path: string?, responseTimeout: number? }) -> Server
```

Starts the [HTTP interactions](../guide/http-interactions.md) endpoint. Without `publicKey`, it is fetched from the application (raises if that fails). Returns the httpx server.

## Commands

### `bot:syncCommands`

```luau
bot:syncCommands(options: { force: boolean? }?)
```

Registers application commands with Discord, scope by scope, skipping scopes whose hash is unchanged (unless `force`). Learns command ids. Raises if the application id isn't known. Failures for a scope are logged, not raised. See [Registering and syncing](../guide/syncing.md).

### `bot:clearCommands`

```luau
bot:clearCommands(guildId: string?) -> (any, any)
```

Removes every command in a guild, or every global command.

### `bot:helpEntries`

```luau
bot:helpEntries(prefix: string?) -> { HelpEntry }
```

Every command without `hidden = true`, in registration order:

| Field | Description |
| --- | --- |
| `name` | The command name |
| `description` | Its description |
| `category` | `def.category`, or `"General"` |
| `kind` | `"slash"`, `"prefix"`, `"user"` or `"message"` |
| `aliases` | Prefix aliases |
| `usage` | `"!ban <user> [days] [reason...]"` for prefix commands and hybrids, `"/name ..."` for slash-only, `"(right click > Apps)"` for context menus |
| `definition` | The normalized definition |

`prefix` defaults to the bot's prefix when it's a single string, else `"/"`.

## Lookups

All of these check the cache first, then call the API and cache the result. They return `nil` if the object can't be found.

| Method | Returns |
| --- | --- |
| `bot:fetchUser(userId)` | a user |
| `bot:fetchGuild(guildId)` | a guild |
| `bot:fetchChannel(channelId)` | a channel |
| `bot:fetchMember(guildId, userId)` | a member, with `.user` attached |

### `bot:isOwner`

```luau
bot:isOwner(userId: string) -> boolean
```

### `bot:application`

```luau
bot:application(refresh: boolean?) -> (table?, err?)
```

The application object (`GET /applications/@me`), cached after the first call. Also sets `applicationId` if unknown.

### `bot:disallowedIntents`

```luau
bot:disallowedIntents() -> { string }?
```

Privileged intents the bot requests that the Developer Portal hasn't enabled. `nil` if the application can't be fetched.

## Presence

### `bot:setStatus`

```luau
bot:setStatus(status: string?, activity: string?, activityType: (string | number)?)
```

`status`: `"online"` (default), `"idle"`, `"dnd"`, `"invisible"`. `activityType`: a name from [`enums.ActivityType`](enums.md#activitytype) (case-insensitive) or a number, default `"custom"`. Custom activities show just the text. Without `activity`, clears activities. Returns `false, "not connected"` before the gateway is connected.

### `bot:setPresence`

```luau
bot:setPresence(presence: { status: string?, activities: { table }?, afk: boolean?, since: number? })
```

Sends a raw presence update. Missing fields are filled with defaults.

## Waiting

### `bot:waitFor`

```luau
bot:waitFor(event: string, filter: ((data: any) -> boolean)?, timeout: number?) -> any?
```

Yields until an event whose data passes `filter` arrives, and returns the data. `nil` after `timeout` seconds (default 60). A filter that errors counts as no match.

## Internals

Methods starting with `_` (`_dispatch`, `_handleInteraction`, `_handleMessage`, `_check`, `_invoke`, ...) are internal and may change. [How it works](../help/internals.md) describes what they do.
