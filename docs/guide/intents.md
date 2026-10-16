# Intents

**Intents** tell Discord which groups of gateway events to send the bot. Asking only for what you need saves bandwidth, and some intents are *privileged*: they must be switched on in the Developer Portal before the bot can use them.

## Setting intents

```luau
local bot = discord.new({
    intents = { "Guilds", "GuildMessages", "GuildMembers", "MessageContent" },
})
```

`intents` accepts names, flag numbers, or one bitmask:

```luau
intents = { "Guilds", "GuildMessages" }
intents = { discord.intents.Flags.Guilds, discord.intents.Flags.GuildMessages }
intents = 513
```

An unknown name raises an error at startup.

### The default

When you don't pass `intents`, the bot uses:

- `Guilds`, `GuildMessages`, `DirectMessages`
- plus `MessageContent` when a `prefix` is set

Slash commands, context menus, components and modals arrive through `INTERACTION_CREATE`, which needs **no** intent. A bot that only uses interactions can run with just `{ "Guilds" }`, which also keeps the [cache](cache.md) of servers, channels and roles filled.

## Available intents

| Intent | Events | Privileged |
| --- | --- | --- |
| `Guilds` | servers, channels, threads, roles, stage instances | |
| `GuildMembers` | member join/update/leave | ✔ |
| `GuildModeration` | bans, audit log entries | |
| `GuildExpressions` | emojis, stickers, soundboard | |
| `GuildIntegrations` | integrations | |
| `GuildWebhooks` | webhook updates | |
| `GuildInvites` | invite create/delete | |
| `GuildVoiceStates` | voice states, voice channel effects | |
| `GuildPresences` | presence updates | ✔ |
| `GuildMessages` | messages in servers | |
| `GuildMessageReactions` | reactions in servers | |
| `GuildMessageTyping` | typing in servers | |
| `DirectMessages` | messages in DMs | |
| `DirectMessageReactions` | reactions in DMs | |
| `DirectMessageTyping` | typing in DMs | |
| `MessageContent` | the *content* of messages (see below) | ✔ |
| `GuildScheduledEvents` | scheduled events | |
| `AutoModerationConfiguration` | AutoMod rule changes | |
| `AutoModerationExecution` | AutoMod actions | |
| `GuildMessagePolls` | poll votes in servers | |
| `DirectMessagePolls` | poll votes in DMs | |

## Message Content

`MessageContent` doesn't deliver events of its own. Without it, messages arrive with an empty `content`, and empty `embeds`, `attachments`, `components` and `poll`, except for:

- messages in DMs with the bot
- messages that mention the bot
- the bot's own messages

[Prefix commands](prefix-commands.md) in servers need it, unless people trigger them with a mention.

## Privileged intents

`GuildMembers`, `GuildPresences` and `MessageContent` must be enabled on the **Bot** page of the [Developer Portal](https://discord.com/developers/applications) under *Privileged Gateway Intents*. Bots in 100 or more servers also need Discord's approval (verification).

Asking for a privileged intent that isn't enabled makes Discord close the connection with code **4014** (*disallowed intents*). To avoid that, discord checks the application before connecting:

- **By default**, privileged intents the portal hasn't enabled are **dropped** with a warning, and the bot connects without them:

    ```text
    [bot] warning: MessageContent isn't enabled for this application (Developer Portal > Bot > Privileged Gateway Intents); connecting without it
    ```

- **With `strictIntents = true`**, nothing is dropped. If the portal disagrees, the connection fails with 4014: `bot:run()` returns `false` and the reason, and `bot:start()` exits.

`bot:disallowedIntents()` returns the list of requested privileged intents that the portal doesn't allow, or `nil` if the application couldn't be fetched.

## Checking intents at runtime

`bot.intents` is the bitmask the bot connects with, after any were dropped:

```luau
local intents = discord.intents

if intents.has(bot.intents, "MessageContent") then
    print("can read message content")
end

print(table.concat(intents.names(bot.intents), ", "))
```

When a handler is registered for an event the current intents won't deliver, discord warns about it (see [Events](events.md#common-events)).

## The intents module

`discord.intents` has the flag values and helpers: `Flags`, `Default`, `NonPrivileged`, `EventRequires`, `resolve`, `has`, `add`, `remove`, `receives`, `names`, `privilegedIn`. See the [intents reference](../reference/intents.md).

!!! note
    `discord.intents` (the module) and `bot.intents` (a number) are different things.
