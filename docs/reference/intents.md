# intents

```luau
local Intent = discord.intents   -- or require("@discord/lib/gateway/intents")
```

Gateway intent flags and helpers. See [Intents](../guide/intents.md) for which to use. The module is frozen.

## Names

`Guilds`, `GuildMembers`*, `GuildModeration`, `GuildExpressions`, `GuildIntegrations`, `GuildWebhooks`, `GuildInvites`, `GuildVoiceStates`, `GuildPresences`*, `GuildMessages`, `GuildMessageReactions`, `GuildMessageTyping`, `DirectMessages`, `DirectMessageReactions`, `DirectMessageTyping`, `MessageContent`*, `GuildScheduledEvents`, `AutoModerationConfiguration`, `AutoModerationExecution`, `GuildMessagePolls`, `DirectMessagePolls`

\* privileged

## Values

| Member | Description |
| --- | --- |
| `Intent.Flags[name]` | The flag value (`Intent.Flags.Guilds == 1`) |
| `Intent.Default` | `Guilds` + `GuildMessages` (the gateway client's default; the bot's own default is in [Bot options](bot-options.md#connection)) |
| `Intent.NonPrivileged` | Every non-privileged intent |
| `Intent.EventRequires[event]` | The intents that deliver an event (any one is enough); missing for events no intent controls |

## Functions

### `Intent.resolve`

```luau
Intent.resolve(input: { string | number } | number | nil) -> number
```

A mask from names and/or flags. `nil` → `Intent.Default`. Raises on unknown names.

### `Intent.has` / `Intent.add` / `Intent.remove`

```luau
Intent.has(mask: number, name: string) -> boolean
Intent.add(mask: number, name: string) -> number
Intent.remove(mask: number, name: string) -> number
```

### `Intent.receives`

```luau
Intent.receives(mask: number, event: string) -> (boolean, { string }?)
```

Whether `mask` receives `event`, and the intents that would deliver it. Events no intent controls (`READY`, `INTERACTION_CREATE`, ...) always return `true`.

### `Intent.names`

```luau
Intent.names(mask: number) -> { string }
```

The names in a mask, sorted.

### `Intent.privilegedIn`

```luau
Intent.privilegedIn(mask: number) -> { string }
```

The privileged intents in a mask.
