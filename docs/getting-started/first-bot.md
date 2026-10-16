# Your first bot

This page builds a small bot in one file, explaining each part along the way. You need a project with discord added ([Installation](installation.md)) and a bot token ([Setting up a Discord application](discord-application.md)).

## The smallest bot

```luau title="main.luau"
local discord = require("@discord")

local bot = discord.new({}) -- the token comes from DISCORD_TOKEN

bot:command({
    name = "ping",
    description = "Check that the bot is alive",
    run = function(ctx)
        ctx:reply("Pong!")
    end,
})

bot:start()
```

Run it:

```sh
DISCORD_TOKEN="your token" sn run
```

The output looks something like this:

```text
[gateway] intents: DirectMessages, GuildMessages, Guilds
[gateway] connecting to wss://gateway.discord.gg
[gateway] identifying
[gateway] ready as my-bot (123456789012345678), in 1 guild(s)
[bot] synced 1 command(s) to global scope
```

Type `/ping` in a server the bot is in.

!!! note "Global commands can take a moment"
    Commands registered globally can take a while to show up in the Discord client. Restarting the client (++ctrl+r++) refreshes the list. While developing, register them in one server instead, where they update instantly. The next section shows how.

Here is what each part does:

- `discord.new(options)` creates a [`Bot`](../reference/bot.md). The only option it needs is `token`. When `token` isn't given, it is read from the `DISCORD_TOKEN` environment variable, as here.
- `bot:command(definition)` registers a command. A definition is a plain table: `name`, `description`, and a `run` function. See [Slash commands](../guide/slash-commands.md).
- `ctx` is the [context](../guide/context.md): who ran the command, where, with which options, and the methods for answering. `ctx:reply(...)` sends the response.
- `bot:start()` connects to the gateway and blocks while the bot runs. On the first `READY` it registers (*syncs*) your commands with Discord.

## Developing in one server

Pass `devGuild` to register every command in one server. Updates there show up immediately:

```luau
local bot = discord.new({
    devGuild = "123456789012345678", -- right click the server > Copy Server ID
})
```

!!! tip
    **Copy Server ID** appears once you enable **Developer Mode** in Discord's settings, under *Advanced*.

When you remove `devGuild`, the commands move back to the global scope, and the copies in the server are cleared on the next start. See [Registering and syncing](../guide/syncing.md).

## Options

Commands can take options. They arrive as the second argument of `run`, keyed by name:

```luau
bot:command({
    name = "greet",
    description = "Say hello to someone",
    options = {
        { name = "user", type = "user", description = "Who to greet", required = true },
        { name = "times", type = "integer", description = "How many times", min = 1, max = 5 },
    },
    run = function(ctx, args)
        local line = `Hello, {ctx.fmt.user(args.user.id)}!`
        ctx:reply(string.rep(line .. "\n", args.times or 1))
    end,
})
```

`user` options resolve to a user object. `integer` options are checked against `min` and `max` by Discord. See [Options](../guide/options.md) for every type.

## A prefix command

Prefix commands are triggered by messages like `!ping`. Set a prefix and mark the command with `prefix = true`. The same definition then answers to both `/ping` and `!ping`:

```luau
local bot = discord.new({
    prefix = "!",
    intents = { "Guilds", "GuildMessages", "DirectMessages", "MessageContent" },
})

bot:command({
    name = "ping",
    description = "Check that the bot is alive",
    prefix = true,
    run = function(ctx)
        ctx:reply("Pong!")
    end,
})
```

Reading message text in servers needs the privileged **Message Content** intent, which must be enabled in the Developer Portal. Without it, prefix commands only work in DMs and when the message starts with the bot's mention (`@my-bot ping`). See [Prefix commands](../guide/prefix-commands.md).

!!! note
    When `prefix` is set and you don't pass `intents`, discord adds `MessageContent` for you.

## A button

[Components](../guide/components.md) are built with `discord.ui`, which is also available as `ctx.ui`. A button can carry its own handler:

```luau
local ui = discord.ui

bot:command({
    name = "clicker",
    description = "A button that counts",
    run = function(ctx)
        local clicks = 0
        ctx:reply({
            content = "Clicked 0 times",
            components = {
                ui.button({
                    label = "Click me",
                    run = function(click)
                        clicks += 1
                        click:update({ content = `Clicked {clicks} times` })
                    end,
                }),
            },
        })
    end,
})
```

`click` is a new context for the button press. `click:update(...)` edits the message the button is on. Handlers like this one are kept in memory for 15 minutes. For buttons that must keep working across restarts, register a [component handler](../guide/component-handlers.md) instead.

## Reacting to events

```luau
bot:once("READY", function(ctx, data)
    print(`logged in as {data.user.username}`)
    ctx.bot:setStatus("online", "/ping", "listening")
end)

bot:on("messageCreate", function(ctx, message)
    if message.content == "hello bot" then
        ctx:react("👋")
    end
end)
```

Event names can be written as Discord sends them (`MESSAGE_CREATE`) or in camelCase (`messageCreate`). See [Events](../guide/events.md).

## Handling errors

If a handler errors, discord prints the traceback and tells the user *"Something went wrong while running this."* To do something else, add an error handler:

```luau
bot:onError(function(err, ctx)
    print("handler failed:", err)
end)
```

See [Middleware and errors](../guide/middleware.md).

## The whole file

```luau title="main.luau"
local process = require("@sonata/process")
local discord = require("@discord")
local ui = discord.ui

local bot = discord.new({
    token = process.env.DISCORD_TOKEN,
    prefix = "!",
    devGuild = process.env.DEV_GUILD, -- nil registers globally
})

bot:command({
    name = "ping",
    description = "Check that the bot is alive",
    prefix = true,
    run = function(ctx)
        local latency = ctx:latency()
        ctx:reply(`Pong! {if latency then math.floor(latency * 1000) .. "ms" else ""}`)
    end,
})

bot:command({
    name = "clicker",
    description = "A button that counts",
    run = function(ctx)
        local clicks = 0
        ctx:reply({
            content = "Clicked 0 times",
            components = {
                ui.button({ label = "Click me", run = function(click)
                    clicks += 1
                    click:update({ content = `Clicked {clicks} times` })
                end }),
            },
        })
    end,
})

bot:once("READY", function(ctx, data)
    print(`logged in as {data.user.username}`)
end)

bot:start()
```

## Next

Once a bot has more than a few commands, [split it into folders](project-structure.md).
