# Prefix commands

Prefix commands are triggered by ordinary messages, such as `!ban @someone spamming`. discord parses the arguments with the same option definitions slash commands use, so one definition can serve both.

## Setting a prefix

```luau
local bot = discord.new({
    prefix = "!",
})
```

`prefix` can be:

| Value | Example |
| --- | --- |
| a string | `"!"` |
| a list | `{ "!", "?", "bot " }` |
| a function | `function(message, bot) return "!" end`, returning a string or a list |

The function runs for every message, may yield, and gets the raw message object, so you can have **per-server prefixes**:

```luau
local bot = discord.new({
    prefix = function(message, bot)
        local settings = bot.store.prefixes or {}
        return settings[message.guild_id] or "!"
    end,
})
```

If the function errors, the error is printed and the message is treated as having no prefix.

The longest matching prefix wins, so `"!!"` takes priority over `"!"`. Spaces after the prefix are allowed (`! ping`).

### Mentions as a prefix

With `mentionPrefix` (on by default), mentioning the bot also works as a prefix: `@MyBot ping`. This also works without the Message Content intent, because Discord always delivers the content of messages that mention the bot.

### Case

With `caseInsensitive` (on by default), both the prefix and the command name ignore case: `!PING`, `!Ping` and `!ping` are the same command.

## Defining prefix commands

There are three ways, depending on whether the command should also be a slash command:

=== "Hybrid (slash + prefix)"

    ```luau
    bot:command({
        name = "ping",
        description = "Pong",
        prefix = true,          -- also answers to !ping
        aliases = { "latency" },
        run = function(ctx) ctx:reply("Pong!") end,
    })
    ```

=== "Prefix only"

    ```luau
    bot:command({
        name = "purge",
        type = "prefix",        -- or bot:prefix({ ... }), or slash = false
        description = "Delete messages",
        options = { { name = "amount", type = "integer", min = 2, max = 100, required = true } },
        run = function(ctx, args) ... end,
    })
    ```

=== "Slash only"

    ```luau
    bot:command({ name = "settings", description = "...", run = function(ctx) ... end })
    ```

Prefix-only commands aren't registered with Discord. Their names don't have to follow the slash rules, so `!Purge` or `!8ball` work.

`aliases` adds more names for prefix use (`!latency`). Aliases don't affect the slash command.

## Hybrid commands

A hybrid runs the same `run` for `/ping` and `!ping`. Most context methods behave the same in both, with these differences:

| | Slash | Prefix |
| --- | --- | --- |
| `ctx:reply` | interaction response | a reply to the command message |
| `ephemeral` | works | ignored |
| `ctx:defer()` | "thinking..." | typing indicator |
| `ctx:showModal` / `ctx:prompt` | work | not available (raise an error) |
| `ctx.member.permissions` | sent by Discord | not set; [computed](checks.md) from the cache when needed |
| `user` option | user with `.member` | a user; `.member` only when Discord included it with the mention (use `type = "member"` to always get the member) |
| `ctx.message` | `nil` | the command message |

Check `ctx.isPrefix` when a hybrid needs to behave differently:

```luau
run = function(ctx, args)
    if ctx.isPrefix then
        ctx:react("✅")
    end
    ctx:reply("Done")
end
```

## Arguments

Arguments are split on whitespace. Text in quotes counts as one argument, using `"double"`, `'single'`, `“curly”` or `‘curly’` quotes:

```text
!tag create "hello world" This is the tag's text
         └─ name = "hello world"   └─ text = "This is the tag's text"
```

Each word is converted according to its option's type:

| Type | Accepts | Errors with |
| --- | --- | --- |
| `string` | any word; checked against `choices`, `minLength`, `maxLength` | `must be at least N characters`, `must be one of: ...` |
| `integer` | `42` | `must be a whole number`, `must be at least N` |
| `number` | `4.2` | `must be a number` |
| `boolean` | `true yes y on 1 enable enabled` / `false no n off 0 disable disabled` | `must be yes or no` |
| `user` | `<@id>`, `<@!id>`, or a raw id | `must be a user mention or id`, `isn't a user I can find` |
| `member` | the same, looked up in this server | `isn't a member of this server` |
| `channel` | `<#id>` or an id, in this server; checked against `channelTypes` | `isn't a channel I can find`, `is the wrong kind of channel` |
| `role` | `<@&id>`, an id, or a role **name** (case-insensitive) | `isn't a role I can find` |
| `mentionable` | a user or role mention, or an id | `must be a user or role` |
| `attachment` | files attached to the message, in order | `` `name` is required: attach a file `` |

`choices` match by name or value, case-insensitively. For `{ name = "Fast", value = 10 }` both `fast` and `10` work, and the command gets `10`.

Users, members, channels and roles are looked up in the [cache](cache.md) first and fetched from the API when needed.

### Taking the rest of the text

The **last string option** takes the rest of the message, spaces included:

```luau
options = {
    { name = "user", type = "user", required = true },
    { name = "reason", type = "string" },   -- the last string option
}
-- !ban @someone being rude in chat  ->  reason = "being rude in chat"
```

This happens automatically when the string option is last, or only attachment options come after it. Control it with `rest`:

- `rest = true` makes any option take the rest of the text.
- `rest = false` stops the last string option from doing it, so it takes one word.

The rest is taken verbatim, quotes included, unless it's a single quoted argument (`!say "hi there"` gives `hi there`).

### Optional typed arguments are skipped

If an **optional** non-string argument doesn't match its type, it is skipped and the word moves on to the next option:

```luau
options = {
    { name = "user", type = "user", required = true },
    { name = "days", type = "integer", min = 0, max = 7 },
    { name = "reason", type = "string" },
}
-- !ban @x 3 spam      -> days = 3, reason = "spam"
-- !ban @x spam        -> days = nil, reason = "spam"   ("spam" isn't an integer, so days is skipped)
-- !ban @x 30 spam     -> days = nil, reason = "30 spam" (30 is out of range, so days is skipped)
```

Set `skip = false` on an option to report an error instead.

### Errors and usage

When parsing fails, the bot replies with the problem and a usage line generated from the options:

```text
❌ `amount` must be a whole number
Usage: `!purge <amount>`
```

In the usage line, `<required>` and `[optional]` mark the options, `name...` marks one that takes the rest of the text, and `name (attachment)` marks a file. When the command was triggered by a mention, the usage line shows `@BotName ` as the prefix.

### Raw arguments

| Field | `!say  hello   "big world"` |
| --- | --- |
| `ctx.argText` | `hello   "big world"` (as typed) |
| `ctx.rawArgs` | `{ "hello", "big world" }` |
| `ctx.content` | the whole message |
| `ctx.invokedWith` | `say` (or the alias used) |
| `ctx.prefix` | `!` |

For complete control, give a command no options and parse `ctx.argText` yourself.

### Attachments

Attachment options take the message's attachments in order, so they don't use up words:

```luau
options = {
    { name = "image", type = "attachment", required = true },
    { name = "caption", type = "string" },
}
-- "!post a nice view" with a file attached  ->  image = <attachment>, caption = "a nice view"
```

## Subcommands

Subcommands and groups work with prefixes too, and subcommands can have `aliases`:

```text
!config show
!config prefix set ?
```

If a command with subcommands is called without one and has no `run` of its own, the bot replies with its usage:

```text
Usage: `!config <show|prefix>`
```

## The Message Content intent

Discord only sends the text of messages to bots with the privileged **Message Content** intent. Without it, `message.content` is empty except in:

- DMs with the bot
- messages that mention the bot
- the bot's own messages

So in servers, prefix commands need Message Content enabled in the Developer Portal (**Bot > Privileged Gateway Intents**) **and** requested by the bot. When `prefix` is set and you don't pass `intents`, discord requests `MessageContent` for you. If the portal hasn't enabled it, it is dropped with a warning and prefix commands keep working in DMs and with mentions.

## What is ignored

These messages never trigger prefix commands:

- messages from bots (unless `ignoreBots = false`), webhooks, and the bot itself
- messages without content
- every message, when the bot has no prefix commands registered

## Unknown commands

When a message starts with a prefix but names no known command, discord emits a custom `UNKNOWN_COMMAND` event:

```luau
bot:on("UNKNOWN_COMMAND", function(ctx, data)
    -- data.message, data.name (what was typed), data.prefix
    ctx.api.channels.react(data.message.channel_id, data.message.id, "❓")
end)
```

## Checks

`permissions`, `botPermissions`, `guildOnly`, `dmOnly`, `ownerOnly`, `nsfw`, `checks` and `cooldown` are all checked for prefix commands. For prefix commands, `permissions` is checked by discord, because Discord doesn't enforce it outside slash commands. See [Checks, permissions and cooldowns](checks.md).
