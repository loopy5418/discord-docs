# Loading from files

`bot:load` requires modules and registers whatever they return. It takes any number of targets:

```luau
bot:load("commands")                   -- every .luau file under ./commands, recursively
bot:load("commands/ping")              -- one file (the extension is optional)
bot:load("commands/ping.luau")
bot:load("plugins/*.luau")             -- a glob pattern
bot:load(require("./extra"))           -- an already-required module (a table or a function)
bot:load("commands", "components", "events")
```

It returns how many modules were loaded.

## Paths

Relative paths are resolved **from the file that calls `bot:load`**, like `require`. `main.luau` calling `bot:load("commands")` loads the `commands` folder next to `main.luau`, whatever the working directory is. Absolute paths work too.

## Folders

When loading a folder:

- Files ending in `.luau` or `.lua` are loaded, recursively.
- Files and folders whose names start with `_` or `.` are **skipped**. Use them for shared helpers (`_helpers.luau`, `_lib/`).
- A folder containing an `init.luau` is loaded as **one module** (the folder is required), and its other files aren't loaded on their own. This applies both to the folder you pass and to subfolders.
- Files are loaded in **sorted path order**, so `01-setup.luau` runs before `02-commands.luau`.

## Globs

Targets containing `*`, `?`, `[` or `{` are glob patterns, matched relative to the calling file. Matched folders are loaded like folders, and matched files whose names start with `_` or `.` are skipped. A pattern that matches nothing prints a warning.

## What a module can return

| Return value | Registered as |
| --- | --- |
| `discord.command({...})`, `discord.slash`, `discord.prefix`, `discord.userCommand`, `discord.messageCommand` | a command |
| `discord.component({...})` | a component handler |
| `discord.modal({...})` | a modal handler |
| `discord.event({...})` | an event handler |
| a table with `event` and `run` | an event handler |
| a table with `customId` or `match`, and `run` | a component handler (a modal handler with `modal = true`) |
| a table with `name`, and `run` or `subcommands` | a command |
| a table with `kind = "command"` / `"component"` / `"modal"` / `"event"` | that kind |
| a table with `commands`, `components`, `modals` and/or `events` lists | everything in the lists |
| a list of any of the above | each item |
| a function | a **plugin**, called as `fn(bot)` |

A module that returns anything else (a string, `nil`, an unrecognized table) raises an error naming the file.

Tagging with `discord.command(...)` and friends isn't required, but it makes intent explicit, and the `discord.prefix` / `discord.userCommand` / `discord.messageCommand` forms set `type` for you.

### Examples

```luau title="commands/ping.luau — one definition"
return {
    name = "ping",
    description = "Pong",
    run = function(ctx) ctx:reply("Pong!") end,
}
```

```luau title="commands/info.luau — several"
local discord = require("@discord")
return {
    discord.command({ name = "serverinfo", description = "About this server", run = ... }),
    discord.command({ name = "botinfo", description = "About the bot", run = ... }),
}
```

```luau title="features/tickets.luau — a feature in one file"
return {
    commands = { { name = "ticket", description = "Open a ticket", run = ... } },
    components = { { customId = "ticket", run = ... } },
    modals = { { customId = "ticket-form", run = ... } },
    events = { { event = "CHANNEL_DELETE", run = ... } },
}
```

```luau title="plugins/stats.luau — a plugin"
return function(bot)
    local count = 0
    bot:use(function(ctx, next)
        count += 1
        next()
    end)
    bot:command({
        name = "stats",
        description = "How busy the bot is",
        run = function(ctx) ctx:reply(`{count} handlers run since start`) end,
    })
end
```

## Registering without files

`bot:add(value)` registers anything a module can return, without requiring a file. `bot:load(table)` does the same and counts it:

```luau
bot:add({ name = "ping", description = "Pong", run = function(ctx) ctx:reply("Pong!") end })
bot:add(require("./features/tickets"))
```

## Errors while loading

Loading errors stop startup with the file's path in the message, for example:

```text
/home/me/my-bot/commands/ban.luau: ...slash command name "Ban" must be 1-32 lowercase characters without spaces
failed to load /home/me/my-bot/commands/broken.luau: ...: Expected 'end' ...
can't load "comands": no such file or folder (looked in /home/me/my-bot)
```

Two definitions with the same name raise *command "x" is defined twice (file1 and file2)*.

## Reloading

Modules are loaded with `require`, which caches them. Changes take effect when the bot restarts.

## In built programs

A program made with `sn build` has no folders to walk. The loader lists the program's **assets** instead, so every folder (and glob) you load must be listed in `build.assets`. Single files don't need it. See [Building a program](../getting-started/building.md#folders-need-to-be-listed-as-assets).
