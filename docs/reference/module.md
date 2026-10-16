# discord (module)

```luau
local discord = require("@discord")
```

The package entry. It is the `Bot` class itself, so it has the constructor, the definition helpers and the libraries.

## Constructor

### `discord.new`

```luau
discord.new(options: BotOptions?) -> Bot
```

Creates a bot. Raises if no token is given and `DISCORD_TOKEN` isn't set. See [Bot options](bot-options.md) and [Bot](bot.md).

## Definition helpers

Called with `.` on the module, these **tag** a definition so [`bot:load`](../guide/loading.md) knows what a file returns, and return the same table. Called with `:` on a bot, they **register** it right away.

| Helper | Tags as | Sets |
| --- | --- | --- |
| `discord.command(def)` | command | |
| `discord.slash(def)` | command | |
| `discord.prefix(def)` | command | `type = "prefix"` (if unset) |
| `discord.userCommand(def)` | command | `type = "user"` (if unset) |
| `discord.messageCommand(def)` | command | `type = "message"` (if unset) |
| `discord.component(def)` | component handler | |
| `discord.modal(def)` | modal handler | |
| `discord.event(def)` | event handler | |

```luau
return discord.userCommand({ name = "Avatar", run = function(ctx, user) ... end })
```

Tagging sets a `__kind` field on the table.

## Libraries

| Field | Module | Reference |
| --- | --- | --- |
| `discord.ui` | Component builders | [ui](ui.md) |
| `discord.fmt` | Formatting helpers | [fmt](format.md) |
| `discord.enums` | Constants | [enums](enums.md) |
| `discord.permissions` | Permission flags and math | [permissions](permissions.md) |
| `discord.intents` | Gateway intents | [intents](intents.md) |
| `discord.messages` | Message input → JSON | [Message input](message-input.md#messagesbuild) |
| `discord.Rest` | The REST client class | [Rest](rest.md) |
| `discord.makeApi` | Builds an `api` object from a `Rest` client | [api](api.md) |

The same libraries (except `intents`, `messages`, `Rest` and `makeApi`) are on every context: `ctx.ui`, `ctx.fmt`, `ctx.enums`, `ctx.permissions`.

## Utilities

### `discord.eventName`

```luau
discord.eventName(name: string) -> string
```

Normalizes an event name: `"messageCreate"`, `"message_create"` and `"MESSAGE_CREATE"` all become `"MESSAGE_CREATE"`. `"*"` stays `"*"`.

## Types

```luau
local discord = require("@discord")

type Context = discord.Context
type CommandDefinition = discord.CommandDefinition
type ComponentDefinition = discord.ComponentDefinition
type ModalDefinition = discord.ModalDefinition
type EventDefinition = discord.EventDefinition
type BotOptions = discord.BotOptions
```

See [Types](types.md).

## Running the package directly

Running the discord repository itself (`sn run` in a clone, or its built program) starts the example launcher instead of returning the module. See [Example bots](../examples/bundled.md).
