# Reference

Every module, function, field and option. For explanations and examples, see the [Guide](../guide/index.md).

## Conventions

- Signatures use Luau type syntax. `?` marks optional parameters and nilable returns.
- `ctx:method(...)` and `bot:method(...)` are methods (called with `:`). `module.function(...)` are plain functions (called with `.`).
- Discord objects (users, members, channels, guilds, messages) are passed through as Discord sends them, with `snake_case` fields. See [Discord's documentation](https://docs.discord.com/developers/intro) for their shapes.
- Ids (snowflakes) are always **strings**.
- Functions that talk to Discord return `(result, nil)` on success and `(nil, err)` on failure. See [Rest errors](rest.md#errors).

## Modules

| Module | Reached as | |
| --- | --- | --- |
| [discord](module.md) | `require("@discord")` | The package entry: `new`, the definition helpers, the libraries |
| [Bot](bot.md) | `discord.new(...)` | Registration, routing, running |
| [Bot options](bot-options.md) | `discord.new(options)` | Every option |
| [Context](context.md) | `ctx` | Every field and method |
| [Definitions](definitions.md) | | Commands, options, subcommands, components, modals, events |
| [Message input](message-input.md) | `ctx:reply(input)` | What message functions accept |
| [ui](ui.md) | `discord.ui`, `ctx.ui` | Component builders, embeds, polls |
| [api](api.md) | `bot.api`, `ctx.api` | Endpoint wrappers |
| [Rest](rest.md) | `discord.Rest`, `bot.rest` | The HTTP client |
| [permissions](permissions.md) | `discord.permissions`, `ctx.permissions` | Permission flags and math |
| [fmt](format.md) | `discord.fmt`, `ctx.fmt` | Markdown, mentions, timestamps |
| [enums](enums.md) | `discord.enums`, `ctx.enums` | Discord constants |
| [intents](intents.md) | `discord.intents` | Gateway intents |
| [Cache](cache.md) | `bot.cache`, `ctx.cache` | The in-memory cache |
| [Gateway](gateway.md) | `bot.gateway` | The low-level gateway client |
| [Types](types.md) | `require("@discord")` types | Exported Luau types |

## Requiring modules directly

The library's modules can be required on their own, without creating a bot:

```luau
local ui = require("@discord/lib/components")
local fmt = require("@discord/lib/format")
local Permissions = require("@discord/lib/permissions")
local enums = require("@discord/lib/enums")
local Rest = require("@discord/lib/rest")
local makeApi = require("@discord/lib/api")
local messages = require("@discord/lib/messages")
local commands = require("@discord/lib/commands")
local Cache = require("@discord/lib/cache")
local gateway = require("@discord/lib/gateway/gateway")
local Intent = require("@discord/lib/gateway/intents")
```
