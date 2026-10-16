# Types

`lib/types.luau` declares Luau types for every definition field, bot option and context member, so the [Luau Language Server](https://github.com/JohnnyMorganz/luau-lsp) can autocomplete and check them.

## Exported from the package

```luau
local discord = require("@discord")

type Context = discord.Context
type CommandDefinition = discord.CommandDefinition
type ComponentDefinition = discord.ComponentDefinition
type ModalDefinition = discord.ModalDefinition
type EventDefinition = discord.EventDefinition
type BotOptions = discord.BotOptions
```

Annotate handlers and definitions to get completion:

```luau
local discord = require("@discord")

local ping: discord.CommandDefinition = {
    name = "ping",
    description = "Pong",
    run = function(ctx: discord.Context)
        ctx:reply("Pong!")
    end,
}

return ping
```

## All types in `types.luau`

`require("@discord/lib/types")` exports these types (the module's value is an empty table):

| Type | Describes |
| --- | --- |
| `MessageInput` | What `reply`/`send`/`edit` accept |
| `File` | A file to upload |
| `AllowedMentions` | Mention settings |
| `OptionType` | `"string" \| "integer" \| ...` |
| `Choice` | An option choice |
| `OptionDefinition` | A command option |
| `Cooldown` | A cooldown setting |
| `Check` | A check function |
| `CommandDefinition` | A command |
| `SubcommandDefinition` | A subcommand or group |
| `ComponentDefinition` | A component handler |
| `ModalDefinition` | A modal handler |
| `EventDefinition` | An event handler |
| `BotOptions` | `discord.new` options |
| `Context` | The handler context |

Discord objects (users, messages, ...) are typed as `any`. The definitions are documented in [Definitions](definitions.md), [Bot options](bot-options.md), [Message input](message-input.md) and [Context](context.md).

## Editor setup

`sn init --lsp` (or `sn init --lsp-only` in an existing project) writes the language server configuration. For `require("@discord")` to resolve in your editor, the alias must point at the installed package. If it doesn't resolve, add it to `.luaurc`:

```json title=".luaurc"
{
    "aliases": {
        "sonata": ".sonata/types/libs",
        "discord": ".sonata/deps/discord",
        "httpx": ".sonata/deps/httpx"
    }
}
```
