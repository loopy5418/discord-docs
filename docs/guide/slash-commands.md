# Slash commands

A slash command is a table with a `name`, a `description` and a `run` function:

```luau
bot:command({
    name = "roll",
    description = "Roll a die",
    options = {
        { name = "sides", type = "integer", description = "How many sides", min = 2, max = 100 },
    },
    run = function(ctx, args)
        local sides = args.sides or 6
        ctx:reply(`🎲 {math.random(1, sides)} (d{sides})`)
    end,
})
```

`run` receives the [context](context.md) and a table of option values keyed by name. The same table is `ctx.options` (and `ctx.args`). Options the user left out are `nil`, unless they have a `default`.

## Names and descriptions

Discord's rules for slash commands:

- **Names** are 1–32 characters, **lowercase**, with no spaces (`user-info` and `user_info` are fine). discord checks this when the command is registered and raises an error if it's broken.
- **Descriptions** are 1–100 characters. If you leave one out, discord uses `"No description"`.

The same rules apply to option and subcommand names.

## Where commands are registered

By default commands are **global**: they work in every server the bot is in, and in DMs with the bot. discord registers them on startup (see [Registering and syncing](syncing.md)).

| Field | Effect |
| --- | --- |
| `guilds = { "id1", "id2" }` | Register only in these servers |
| bot option `devGuild = "id"` | Register **every** command in this server instead (for development) |
| `guildOnly = true` | Usable only in servers, not in DMs (shorthand for `contexts = { "guild" }`) |
| `contexts = { ... }` | Where it can be used: `"guild"`, `"botDm"`, `"privateChannel"` |
| `install = { ... }` | Install types: `"guild"`, `"user"` (see [User-installed apps](user-installs.md)) |

## Who can use them

```luau
bot:command({
    name = "purge",
    description = "Delete messages",
    permissions = { "ManageMessages" },     -- default permissions to see and use it
    botPermissions = { "ManageMessages" },  -- the bot needs these too
    guildOnly = true,
    cooldown = 5,                           -- seconds per user
    run = function(ctx, args) ... end,
})
```

- `permissions` is registered with Discord as the command's *default member permissions*. Members without them don't see the command, and server admins can change who may use it under **Server Settings > Integrations**. discord also checks it at runtime for [prefix commands](prefix-commands.md).
- `botPermissions` is checked before `run`. If the bot lacks any of them, the user gets *"I need these permissions: ..."*.
- `nsfw = true` marks the command as age-restricted. Discord only shows it in age-restricted channels.
- `ownerOnly`, `dmOnly`, `checks` and `cooldown` are covered in [Checks, permissions and cooldowns](checks.md).

## Response defaults

| Field | Effect |
| --- | --- |
| `ephemeral = true` | Replies default to ephemeral |
| `defer = true` | Defer before `run` (shows "thinking...") |
| `defer = "ephemeral"` | Defer with an ephemeral response |

See [Responding](responding.md).

## Localization

Names, descriptions and choices can be translated. Discord shows the version for the user's client language:

```luau
bot:command({
    name = "hello",
    description = "Say hello",
    nameLocalizations = { fr = "bonjour", ["es-ES"] = "hola" },
    descriptionLocalizations = { fr = "Dire bonjour", ["es-ES"] = "Saludar" },
    options = {
        {
            name = "tone",
            description = "How",
            nameLocalizations = { fr = "ton" },
            descriptionLocalizations = { fr = "Comment" },
            choices = {
                { name = "Friendly", value = "friendly", nameLocalizations = { fr = "Amical" } },
            },
        },
    },
    run = function(ctx, args)
        -- ctx.locale is the user's language, e.g. "fr"
        ctx:reply(if ctx.locale == "fr" then "Bonjour !" else "Hello!")
    end,
})
```

Your code always sees the original `name` and the choice `value`, whatever language the user's client uses.

## Metadata for help commands

Definitions can carry any extra fields. discord ignores the ones it doesn't know, so you can use them for your own purposes. These are read by `bot:helpEntries()`:

| Field | Use |
| --- | --- |
| `category` | Grouping in help listings (default `"General"`) |
| `usage` | Your own usage text |
| `examples` | Example invocations |
| `hidden = true` | Leave the command out of `bot:helpEntries()` |

```luau
for _, entry in ctx.bot:helpEntries("!") do
    -- entry.name, entry.description, entry.category, entry.kind, entry.aliases,
    -- entry.usage ("!ban <user> [days] [reason...]"), entry.definition
end
```

The [showcase help command](../examples/recipes.md#a-help-command) builds a paginated help menu from this.

## Hybrid commands

Add `prefix = true` and the same command also answers to `!name`. See [Prefix commands](prefix-commands.md#hybrid-commands).

## Mentioning a command

After syncing, discord knows each slash command's id, so you can produce a clickable mention:

```luau
ctx:reply(`Try {ctx:commandMention("config set")}`) -- </config set:123...>, or "/config set" if unknown
```

## Validation errors

These problems raise an error when the command is registered, so they show up at startup:

- a missing `name`, or a missing `run` (on a command without subcommands)
- a slash command, option or subcommand name that isn't 1–32 lowercase characters without spaces
- a **required option after an optional one** (Discord requires required options first)
- an unknown option type or command `type`
- `options` on a command that also has `subcommands` (put them on the subcommands)
- options or subcommands on a context menu command
- two commands with the same name and type, or two prefix commands sharing a name or alias

## Next

- [Options](options.md): every option type and its fields.
- [Subcommands](subcommands.md): `/config show`, `/config prefix set`.
