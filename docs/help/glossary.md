# Glossary

Action row
:   A component that holds up to 5 buttons or one select menu.

Application
:   What you create in the Developer Portal. It owns the bot user, the commands, the public key and the install settings.

Autocomplete
:   Suggestions Discord shows while a user types a slash command option, provided by the bot.

Bucket
:   A group of API routes that share a rate limit.

Cache
:   discord's in-memory copy of servers, channels, roles, users and members seen on the gateway.

Components v2
:   Layout components (containers, text displays, sections, galleries, separators, files) that replace `content` and `embeds`. Messages using them carry the `IsComponentsV2` flag.

Context (`ctx`)
:   The object every handler receives, describing what happened and how to respond.

Context menu command
:   A command shown under *Apps* when right-clicking a user or message.

Custom id
:   A string (up to 100 characters) attached to a button, select or modal, sent back when it's used. discord splits it at `:` into a name and parameters.

Defer
:   Acknowledging an interaction now and answering later (within 15 minutes).

Ephemeral
:   A message only the interacting user can see.

Gateway
:   Discord's WebSocket connection, through which events arrive.

Guild
:   Discord's API name for a server.

Hybrid command
:   A command that works both as a slash command and a prefix command (`prefix = true`).

Inline handler
:   A `run` function given directly to a button, select or modal, kept in memory for a while.

Intent
:   A flag that subscribes the bot to a group of gateway events. Three are privileged.

Interaction
:   A user action Discord sends to the bot: a command, a component click, a modal submit, or an autocomplete request. It must be answered within 3 seconds.

Middleware
:   A function wrapping every command, component, modal and autocomplete handler.

Modal
:   A pop-up form.

Prefix command
:   A command triggered by a normal message starting with a prefix, like `!ping`.

Privileged intent
:   `GuildMembers`, `GuildPresences` or `MessageContent`; must be enabled in the Developer Portal.

Scope (commands)
:   Where a command is registered: globally, or in one guild.

Snowflake
:   Discord's 64-bit ids, always handled as strings. They encode a creation time (`fmt.snowflakeTime`).

Sync
:   Registering the bot's application commands with Discord.

User install
:   An application installed on a user's account rather than a server, so its commands work wherever the user is.
