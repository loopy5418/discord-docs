---
title: Home
hide:
  - navigation
---

# discord

**discord** is a Discord bot framework for [Sonata](https://sonata.5418.org), the Luau runtime. It covers slash commands, prefix commands, buttons, modals and gateway events, and you write the bot as plain Luau tables and functions.

```luau
local discord = require("@discord")

local bot = discord.new({ token = TOKEN, prefix = "!" })

bot:command({
    name = "ping",
    description = "Pong!",
    prefix = true, -- answers to /ping and !ping
    run = function(ctx)
        ctx:reply("Pong!")
    end,
})

bot:load("commands", "components", "events")
bot:start()
```

It is built on Sonata's standard library and [httpx](https://github.com/loopy5418/httpx), a wrapper around `@sonata/net`. It has no other dependencies.

## What it does

<div class="grid cards" markdown>

-   **Commands of every kind**

    ---

    Slash commands, prefix commands, hybrids that answer to both, user and message context menus, subcommands and groups, autocomplete, and localizations. Commands that change are synced with Discord when the bot starts.

    [:octicons-arrow-right-24: Slash commands](guide/slash-commands.md)

-   **One context for everything**

    ---

    Every handler gets a `ctx` with the same methods: `reply`, `defer`, `edit`, `followUp`, `dm`, `react`, `confirm`, `paginate`, `prompt`, permission checks and lookups. It works the same way whether the handler was triggered by an interaction or a message.

    [:octicons-arrow-right-24: The context](guide/context.md)

-   **Components v2 and modals**

    ---

    Builders for containers, sections, galleries, buttons, selects, modals, labels, file uploads, radio groups and checkboxes. You can attach a handler straight to a button with `run = function(ctx) ... end`.

    [:octicons-arrow-right-24: Components](guide/components.md)

-   **Checks built in**

    ---

    Member and bot permissions, owner-only, guild-only, NSFW, custom checks and cooldowns, with readable error replies. Middleware runs around every handler.

    [:octicons-arrow-right-24: Checks](guide/checks.md)

-   **Organize by folders**

    ---

    `bot:load("commands")` loads every file in a folder. It also works in programs made with `sn build`, which read folders from the program's assets.

    [:octicons-arrow-right-24: Loading from files](guide/loading.md)

-   **Gateway or HTTP**

    ---

    The bot connects to the gateway with resuming, heartbeats and zombie-connection detection. It can also receive interactions on an ed25519-verified HTTP endpoint, with or without the gateway.

    [:octicons-arrow-right-24: HTTP interactions](guide/http-interactions.md)

-   **A REST client that handles rate limits**

    ---

    Rate limits are tracked per bucket and globally, and requests are retried. Files upload as multipart, and audit log reasons go in a header. `ctx.api` wraps the common endpoints, and `api.request` covers the rest.

    [:octicons-arrow-right-24: Calling the API](guide/api.md)

-   **Typed**

    ---

    `types.luau` declares every definition field, bot option and context member, so the Luau language server can complete them as you type.

    [:octicons-arrow-right-24: Types](reference/types.md)

</div>

## Where to start

- **New to Sonata or Discord bots?** Start with [Installation](getting-started/installation.md), then [Setting up a Discord application](getting-started/discord-application.md) and [Your first bot](getting-started/first-bot.md).
- **Know what you want to build?** The [Guide](guide/index.md) explains each feature with examples.
- **Looking up a function or a field?** The [Reference](reference/index.md) lists every module.
- **Want working code?** See the [example bots](examples/bundled.md) and the [recipes](examples/recipes.md).
- **Something isn't working?** See [Troubleshooting](help/troubleshooting.md).

## Requirements

- [Sonata](https://sonata.5418.org) 1.0 or newer (the `sn` command).
- A Discord application with a bot user. [Setting up a Discord application](getting-started/discord-application.md) shows how to create one.
- The bot uses Discord API **v10**.
