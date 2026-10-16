# Definitions

The tables you register with `bot:command`, `bot:component`, `bot:modal` and `bot:event`, or return from files. Definitions may carry extra fields of your own; unknown fields are kept and ignored.

## CommandDefinition

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `name` | `string` | **required** | Slash: 1–32 lowercase characters, no spaces. Context menus: 1–32 characters. Prefix-only: anything. |
| `description` | `string` | `"No description"` | 1–100 characters (slash commands) |
| `type` | `"slash" \| "user" \| "message" \| "prefix"` | `"slash"` | Also accepts `"chat"`, `"chatInput"`, `1`, `2`, `3`, `"text"` |
| `prefix` | `boolean` | `false` | Slash commands: also a prefix command (a hybrid) |
| `slash` | `boolean` | `true` | `false` on a slash command makes it prefix-only |
| `aliases` | `{ string }` | `{}` | More prefix names |
| `options` | `{ OptionDefinition }` | `{}` | Not allowed with `subcommands` or on context menus |
| `subcommands` | `{ SubcommandDefinition }` | | Subcommands and groups |
| `run` | `function` | **required** without `subcommands` | `run(ctx, options)` (slash, prefix), `run(ctx, user)` (user), `run(ctx, message)` (message) |
| `autocomplete` | `function` | | Fallback autocomplete handler: `(ctx, value, optionName) -> choices?` |
| `permissions` | `{ string } \| string \| number` | | Default member permissions; checked at runtime for prefix |
| `botPermissions` | `{ string } \| string \| number` | | Checked before running (in servers where the bot is present) |
| `ownerOnly` | `boolean` | | |
| `guildOnly` | `boolean` | | Also sets `contexts = { "guild" }` when `contexts` is unset |
| `dmOnly` | `boolean` | | |
| `nsfw` | `boolean` | | Age-restricted; checked at runtime for prefix |
| `install` | `{ "guild" \| "user" }` | `defaultInstall` | Integration types (`"server"` = `"guild"`) |
| `contexts` | `{ "guild" \| "botDm" \| "privateChannel" }` | `defaultContexts` | Interaction contexts (`"server"`, `"dm"`, `"private"`, `"group"` also accepted) |
| `guilds` | `{ string }` | | Register only in these guilds |
| `cooldown` | `Cooldown` | | |
| `checks` | `{ Check }` | | |
| `ephemeral` | `boolean` | | Replies default to ephemeral |
| `defer` | `boolean \| "ephemeral"` | | Defer before running |
| `nameLocalizations` | `{ [locale]: string }` | | |
| `descriptionLocalizations` | `{ [locale]: string }` | | |
| `category` | `string` | `"General"` in help | For `bot:helpEntries` |
| `usage`, `examples` | | | Free-form metadata |
| `hidden` | `boolean` | | Leave out of `bot:helpEntries` |

## OptionDefinition

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `name` | `string` | **required** | Slash: 1–32 lowercase characters, no spaces |
| `description` | `string` | `"No description"` | 1–100 characters |
| `type` | `OptionType \| number` | `"string"` | See below |
| `required` | `boolean` | `false` | Required options must come first |
| `default` | `any` | | Value when left out |
| `choices` | `{ Choice }` | | Up to 25 |
| `min` / `max` | `number` | | Also `minValue` / `maxValue` |
| `minLength` / `maxLength` | `number` | | |
| `channelTypes` | `{ string \| number }` | | Names from [`enums.ChannelTypeAlias`](enums.md#channeltypealias) or numbers |
| `autocomplete` | `function \| { Choice } \| true` | | Handler, a list to filter, or `true` to use the command's handler |
| `rest` | `boolean` | last string option | Prefix: take the rest of the text |
| `skip` | `boolean` | `true` | Prefix: skip an optional typed option that doesn't parse |
| `nameLocalizations` / `descriptionLocalizations` | | | |

**OptionType**: `"string"` (`"str"`, `"text"`), `"integer"` (`"int"`), `"number"` (`"float"`), `"boolean"` (`"bool"`), `"user"`, `"member"`, `"channel"`, `"role"`, `"mentionable"`, `"attachment"` (`"file"`).

**Choice**: a string, a number, or `{ name: string, value: string | number, nameLocalizations: { [locale]: string }? }`.

## SubcommandDefinition

| Field | Description |
| --- | --- |
| `name`, `description` | As for commands |
| `options` | The subcommand's options |
| `subcommands` | Makes it a group (only at the first level) |
| `run` | Falls back to the command's `run` |
| `aliases` | Prefix aliases |
| `autocomplete` | Falls back to the command's |
| `permissions`, `botPermissions`, `ownerOnly`, `guildOnly`, `dmOnly`, `cooldown`, `checks`, `ephemeral`, `defer` | Override the command's |
| `nameLocalizations`, `descriptionLocalizations` | |

## Cooldown

```luau
type Cooldown = number | {
    seconds: number,
    scope: ("user" | "guild" | "channel" | "global")?,  -- default "user"
    message: string?,
}
```

A number means `{ seconds = n, scope = "user" }`.

## Check

```luau
type Check = (ctx: Context) -> (boolean | string | nil, string?)
```

`false` (optionally with a message) or a string fails; anything else passes.

## ComponentDefinition

| Field | Type | Description |
| --- | --- | --- |
| `customId` | `string \| { string }` | Name(s) to handle: the custom id before the first `:` (or the whole id) |
| `match` | `string` | A Lua pattern on the whole custom id; captures become parameters |
| `run` | `(ctx, ...params) -> ()` | **Required** |
| `defer` | `true \| "update" \| "reply" \| "ephemeral"` | Defer before running |
| `ephemeral` | `boolean` | Replies default to ephemeral |
| `permissions`, `botPermissions`, `ownerOnly`, `guildOnly`, `dmOnly`, `cooldown`, `checks` | | Checked before running |

`customId` or `match` is required. `name` is accepted in place of `customId`.

## ModalDefinition

Same as ComponentDefinition, with `run(ctx, fields)`. Untagged tables need `modal = true` to be loaded as modal handlers.

## EventDefinition

| Field | Type | Description |
| --- | --- | --- |
| `event` | `string` | **Required.** `"MESSAGE_CREATE"`, `"messageCreate"`, `"*"`, ... |
| `run` | `(ctx, data) -> ()` | **Required.** `(ctx, name, data)` for `"*"` |
| `once` | `boolean` | Run only once |

## Normalized commands

`bot:command` returns, and `bot.commandList` holds, a normalized copy with these extra fields:

| Field | Description |
| --- | --- |
| `kind` | `"slash"`, `"user"`, `"message"` or `"prefix"` |
| `slash` | Whether it is registered as a slash command |
| `isPrefix` | Whether it answers to a prefix |
| `options` | Normalized options, each with `typeName` (canonical type) and `discordType` (number) |
| `subcommandMap` / `subcommandOrder` | Subcommands by lowercase name (and alias) / in order |
| `source` | Where it came from (a file path), if loaded |

`require("@discord/lib/commands")` exposes `normalize`, `toJSON`, `resolveLeaf`, `resolveOptions`, `tokenize`, `parseArgs`, `usage`, `takesRest` and `OptionTypes`, which discord uses internally.
