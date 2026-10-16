# Project structure

discord doesn't require a layout. Small bots fit in one file. Larger bots usually keep one command per file and load folders with [`bot:load`](../guide/loading.md).

## A typical layout

```text
my-bot/
├── main.luau                  creates the bot, loads the folders, starts it
├── commands/
│   ├── ping.luau
│   ├── help.luau
│   ├── _helpers.luau          skipped by the loader (starts with _)
│   ├── moderation/
│   │   ├── ban.luau
│   │   └── purge.luau
│   └── fun/
│       └── counter.luau
├── components/
│   └── roles.luau             handlers for buttons and selects by custom id
├── modals/
│   └── report.luau
├── events/
│   ├── ready.luau
│   └── welcome.luau
└── .sonata/
    └── project.luau
```

```luau title="main.luau"
local discord = require("@discord")

local bot = discord.new({ prefix = "!" })

bot:load("commands", "components", "modals", "events")

bot:start()
```

`bot:load` walks each folder recursively. It requires every `.luau` file and registers whatever the file returns. Paths are relative to the file that calls `bot:load`, as with `require`.

## What a file returns

A command file returns its definition:

```luau title="commands/ping.luau"
return {
    name = "ping",
    description = "Check that the bot is alive",
    prefix = true,
    run = function(ctx)
        ctx:reply("Pong!")
    end,
}
```

The loader works out what a table is from its fields. A table with `name` and `run` is a command, one with `customId` or `match` is a component handler, and one with `event` is an event handler. To make it explicit, and get better type checking, wrap the table in a helper:

```luau title="components/roles.luau"
local discord = require("@discord")

return discord.component({
    customId = "roles",
    run = function(ctx, action)
        ctx:reply(`You picked {action}`)
    end,
})
```

```luau title="events/ready.luau"
return {
    event = "READY",
    once = true,
    run = function(ctx, data)
        print(`logged in as {data.user.username}`)
    end,
}
```

A file can also return several definitions, or a plugin function:

```luau title="commands/info.luau"
local discord = require("@discord")

return {
    discord.command({ name = "userinfo", description = "About a user", run = function(ctx) end }),
    discord.userCommand({ name = "User Info", run = function(ctx, user) end }),
}
```

```luau title="plugins/logging.luau"
return function(bot)
    bot:use(function(ctx, next)
        print(ctx.kind, ctx.commandPath)
        next()
    end)
end
```

[Loading from files](../guide/loading.md) lists every form a module can return.

## Shared code

The loader skips files and folders whose names start with `_` or `.`, so put shared code there and `require` it:

```luau title="commands/_helpers.luau"
local helpers = {}

function helpers.isStaff(ctx)
    return ctx:hasPermission("ManageMessages")
end

return helpers
```

```luau title="commands/moderation/purge.luau"
local helpers = require("../_helpers")
-- ...
```

A folder with an `init.luau` is loaded as one module, and its other files are not loaded on their own. Use this for a command that spans several files.

## Shared state

`ctx.store` is a table shared by every handler for as long as the bot runs. It is the same table as `bot.store`:

```luau
-- commands/counter.luau
run = function(ctx)
    ctx.store.count = (ctx.store.count or 0) + 1
    ctx:reply(`Called {ctx.store.count} times`)
end
```

It lives in memory and is lost on restart. Save anything that must persist to a file or a database yourself.

## Configuration

Keep tokens and IDs out of your code. Two common patterns:

=== "Environment variables"

    ```luau
    local process = require("@sonata/process")

    local bot = discord.new({
        token = process.env.DISCORD_TOKEN,
        devGuild = process.env.DEV_GUILD,
    })
    ```

=== "A config module"

    ```luau title="config.luau"
    local fs = require("@sonata/fs")
    local json = require("@sonata/json")

    -- config.json is not committed
    return json.decode(fs.readFile("config.json"))
    ```

    ```luau title="main.luau"
    local config = require("./config")
    local bot = discord.new({ token = config.token, owners = config.owners })
    ```

## Next

[Build the project](building.md) into one executable.
