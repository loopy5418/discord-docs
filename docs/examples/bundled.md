# Example bots

The [discord repository](https://github.com/loopy5418/discord) is also a runnable project: running it starts a launcher for its example bots.

```sh
git clone https://github.com/loopy5418/discord
cd discord
sn pkg add --install                     # fetch httpx

sn run                                   # list the examples
sn run -- whoami --token <token>         # check a token and the gateway connection
sn run -- ping --guild <server id>       # the smallest bot
sn run -- showcase                       # every feature, loaded from folders
sn run -- http --port 8080               # interactions over HTTP, no gateway
sn run -- selftest                       # offline tests, no token needed
```

They also work as a built program:

```sh
sn build
./dist/discord whoami
```

And one example can run on its own:

```sh
sn run --entry examples/ping/main.luau -- --token <token>
```

## Options

| Flag | Description |
| --- | --- |
| `--token <token>` | The bot token. Otherwise `DISCORD_TOKEN`, or a file named `token` in the working directory or the project root (for a built program: the executable's folder). |
| `--guild <id>` | Register the example's commands in that server only (instant updates). Also `DISCORD_GUILD`. |
| `--prefix <prefix>` | Prefix for text commands (default `!`) |
| `--port <port>` | Port for the `http` example (default 8080) |
| `--public-key <hex>` | Public key for the `http` example (default: fetched) |
| `--debug` | Print gateway payloads |
| `--debug-rest` | Print REST calls |

With `sn run`, put the arguments after `--`.

`ping`, `showcase` and `http` register slash commands on the application when they start. `whoami` only reads.

## whoami

A connection check. It:

1. Validates the token over REST and prints the bot's name and id.
2. Prints the application: public or private, owner, server count.
3. Shows which privileged intents are enabled in the portal.
4. Warns if an Interactions Endpoint URL is set (interactions won't come over the gateway).
5. Prints an invite link (with a sensible set of permissions) and a user-install link.
6. Shows the gateway URL, recommended shards and session start limits.
7. Connects to the gateway with only `Guilds`, waits for `READY`, and disconnects.

Run it first when a bot doesn't connect. It shows REST use without a bot (`discord.makeApi(discord.Rest.new(token))`) and `bot:run()` with `bot:stop()`.

## ping

The smallest useful bot (`examples/ping/main.luau`): one hybrid command, `/ping` and `!ping`, that reports the gateway latency, and a `READY` handler that sets the status.

```luau
local bot = discord.new({
    token = cfg.token,
    prefix = cfg.prefix,
    intents = { "Guilds", "GuildMessages", "DirectMessages", "MessageContent" },
    devGuild = cfg.guild,
})

bot:command({
    name = "ping",
    description = "Check that the bot is alive",
    prefix = true,
    run = function(ctx)
        local latency = ctx:latency()
        ctx:reply(`🏓 Pong! ({if latency then math.floor(latency * 1000) .. "ms" else "no heartbeat yet"})`)
    end,
})

bot:once("READY", function(ctx, data)
    ctx.bot:setStatus("online", "/ping", "listening")
end)

bot:start()
```

## showcase

Every feature, with commands, components and events in folders (`examples/showcase/`):

```text
showcase/
├── main.luau                     creates the bot, loads folders, adds /uptime and logging middleware
├── commands/
│   ├── _helpers.luau             skipped by the loader; shared code
│   ├── ping.luau                 hybrid command with an alias
│   ├── help.luau                 help from bot:helpEntries(), paginated, with autocomplete
│   ├── config.luau               subcommands and a subcommand group, prefix too
│   ├── feedback.luau             ctx:prompt with a label, a select and a text area
│   ├── roles.luau                posts a role picker (handled in components/roles.luau)
│   ├── fun/color.luau            autocomplete (list and function), embeds
│   ├── fun/counter.luau          components v2 with inline button handlers
│   ├── info/quote.luau           message context menu, works as a user install
│   ├── info/userinfo.luau        one file, two commands: /userinfo and User Info
│   ├── moderation/ban.luau       permissions, ctx:confirm, audit log reason
│   └── moderation/purge.luau     prefix-only, channel cooldown, bulk delete
├── components/
│   └── roles.luau                handles "roles:*" custom ids
└── events/
    ├── ready.luau                once, sets the status
    └── messages.luau             a list of events, camelCase names
```

Things worth reading in it:

- **`commands/config.luau`**: `/config show`, `/config prefix set <value>`, `/config prefix reset`, also as `!config prefix set ?`. Settings live in `ctx.store`.
- **`commands/moderation/ban.luau`**: `permissions` and `botPermissions`, `ctx:confirm(..., { ephemeral = true })`, then `ctx.api.guilds.ban` with `deleteMessageSeconds` and a reason, reporting failures with `followUp`.
- **`commands/roles.luau`** + **`components/roles.luau`**: the command posts a select with custom id `roles:pick`; the registered handler (`customId = "roles"`, `defer = "ephemeral"`) reads the offered roles back from the message with `ctx:findComponent`, compares with `ctx.member.roles`, and adds/removes roles. It works after restarts because no state is kept in memory.
- **`commands/fun/counter.luau`**: a render function rebuilding a container whose inline buttons capture a local `count`.
- **`commands/help.luau`**: groups `bot:helpEntries()` by category and shows one container per page with `ctx:paginate`; slash commands are shown as clickable mentions via `ctx:commandMention`.

The project's manifest lists the showcase folders in `build.assets`, so `./dist/discord showcase` works as a built program.

## http

Interactions over HTTP, with no gateway (`examples/http/main.luau`):

```luau
local bot = discord.new({
    token = cfg.token,
    gateway = false,
    http = { port = cfg.port, publicKey = cfg.publicKey },
    devGuild = cfg.guild,
})

bot:command({
    name = "hello",
    description = "Answered over HTTP",
    install = { "guild", "user" },
    contexts = { "guild", "botDm", "privateChannel" },
    run = function(ctx)
        ctx:reply({
            components = {
                ui.container({
                    ui.text(`Hello, {ctx:displayName()}! This came over HTTP.`),
                    ui.row(ui.button({ label = "Again", run = function(c)
                        c:update({ components = { ui.text("Still HTTP 👋") } })
                    end })),
                }),
            },
        })
    end,
})

bot:start()
```

Start it, expose the port over HTTPS (a tunnel works), and set the URL as the Interactions Endpoint URL in the Developer Portal. See [HTTP interactions](../guide/http-interactions.md).

## selftest

`tests/offline.luau` exercises the library without a network: it replaces `bot.rest.request` with a fake, loads the showcase, and feeds it hand-made interactions and messages. It covers registration JSON, slash commands, subcommand groups, autocomplete, inline and registered components, context menus, `prompt`, `paginate`, error handling, prefix parsing, permission checks, cooldowns, permissions math, message building and ed25519 verification.

```sh
sn run -- selftest
# 57 passed, 0 failed
```

It's also a good template for testing your own bot offline.
