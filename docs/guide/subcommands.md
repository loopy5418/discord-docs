# Subcommands

Subcommands group related actions under one command: `/tag create`, `/tag delete`, `/tag list`. Up to two levels are allowed: a command can have **groups**, and groups have subcommands, as in `/config prefix set`.

## Subcommands

```luau
bot:command({
    name = "tag",
    description = "Manage tags",
    subcommands = {
        {
            name = "create",
            description = "Create a tag",
            options = {
                { name = "name", type = "string", required = true },
                { name = "text", type = "string", required = true },
            },
            run = function(ctx, args)
                ctx.store.tags = ctx.store.tags or {}
                ctx.store.tags[args.name] = args.text
                ctx:reply(`Created tag **{args.name}**`)
            end,
        },
        {
            name = "delete",
            description = "Delete a tag",
            options = { { name = "name", type = "string", required = true } },
            run = function(ctx, args)
                ctx.store.tags[args.name] = nil
                ctx:reply("Deleted.")
            end,
        },
    },
})
```

A command with `subcommands` can't have `options` of its own. Put the options on each subcommand.

## Groups

A subcommand that has its own `subcommands` is a group:

```luau
bot:command({
    name = "config",
    description = "Server settings",
    guildOnly = true,
    permissions = { "ManageGuild" },
    subcommands = {
        { name = "show", description = "Show the settings", run = function(ctx) ... end },
        {
            name = "prefix",
            description = "The prefix for text commands",
            subcommands = {
                {
                    name = "set",
                    description = "Change the prefix",
                    options = { { name = "value", type = "string", required = true, maxLength = 5 } },
                    run = function(ctx, args) ... end,
                },
                { name = "reset", description = "Use the default prefix", run = function(ctx) ... end },
            },
        },
    },
})
```

This registers `/config show`, `/config prefix set <value>` and `/config prefix reset`. Groups can't contain further groups.

## Which subcommand ran

Inside the handler:

| Field | `/config prefix set` |
| --- | --- |
| `ctx.commandName` | `"config"` |
| `ctx.commandPath` | `"config prefix set"` |
| `ctx.group` | `"prefix"` |
| `ctx.subcommand` | `"set"` |
| `ctx.command` | the `config` definition |
| `ctx.subcommandDef` | the `set` definition |

## One handler for all subcommands

A subcommand without its own `run` uses the command's `run`. In that handler, branch on `ctx.subcommand`:

```luau
bot:command({
    name = "music",
    description = "Music controls",
    subcommands = {
        { name = "play", description = "Resume" },
        { name = "pause", description = "Pause" },
        { name = "skip", description = "Next track" },
    },
    run = function(ctx)
        player[ctx.subcommand]()
        ctx:reply(`Done: {ctx.subcommand}`)
    end,
})
```

## Per-subcommand settings

Subcommands can set these fields. Each overrides the command's value for that subcommand:

`permissions`, `botPermissions`, `ownerOnly`, `guildOnly`, `dmOnly`, `cooldown`, `checks`, `ephemeral`, `defer`, `autocomplete`.

```luau
subcommands = {
    { name = "list", description = "Everyone can list", run = ... },
    { name = "wipe", description = "Owners only", ownerOnly = true, run = ... },
}
```

!!! note
    Discord only registers `permissions` for the whole command, so it decides who *sees* the command. A subcommand's `permissions` is checked by discord at runtime for prefix invocations, but not for slash invocations. To restrict a subcommand for slash use, add a check:

    ```luau
    { name = "wipe", checks = { function(ctx)
        return ctx:hasPermission("Administrator"), "Admins only."
    end }, run = ... }
    ```

## As prefix commands

With `prefix = true`, subcommands work as prefix commands too: `!config prefix set ?`. Subcommands can have `aliases` for prefix use. If the user leaves out the subcommand, they get a usage line:

```text
Usage: `!config <show|prefix>`
```

See [Prefix commands](prefix-commands.md#subcommands).
