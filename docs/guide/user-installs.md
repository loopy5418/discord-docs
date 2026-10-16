# User-installed apps

Discord applications can be installed in two ways:

- **Guild install**: a server admin adds the bot to a server, the usual way.
- **User install**: a user adds the app to their own account. Its commands then work in any server, DM or group DM they're in, even where the bot user isn't a member.

## Allowing user installs

1. In the [Developer Portal](https://discord.com/developers/applications), open **Installation**, tick **User Install**, and give it the `applications.commands` scope.
2. Mark the commands that should be available to user installs:

```luau
bot:command({
    name = "quote",
    description = "Quote a message",
    install = { "guild", "user" },                       -- installable both ways
    contexts = { "guild", "botDm", "privateChannel" },   -- usable everywhere
    run = function(ctx) ... end,
})
```

| Field | Values | Meaning |
| --- | --- | --- |
| `install` | `"guild"` (or `"server"`), `"user"` | How the app must be installed for the command to exist |
| `contexts` | `"guild"` (or `"server"`), `"botDm"` (or `"dm"`), `"privateChannel"` (or `"private"`, `"group"`) | Where it can be used |

`privateChannel` covers group DMs and DMs between other users, which only user-installed commands can reach. Names are matched loosely: `"bot_dm"`, `"BotDM"` and `"botDm"` are all accepted.

To apply settings to every command that doesn't set them, use the bot options:

```luau
discord.new({
    defaultInstall = { "guild", "user" },
    defaultContexts = { "guild", "botDm", "privateChannel" },
})
```

Commands that set neither (and have no defaults) leave the fields out, and Discord applies its own defaults.

## What the bot can do there

When a command runs through a user install in a server the bot isn't in, the bot can only answer the interaction. It can't act in the channel as itself.

| Works | Doesn't work |
| --- | --- |
| `ctx:reply`, `followUp`, `edit`, `defer`, `deleteReply` | `ctx:send` (posting as the bot) |
| `ctx:showModal`, `ctx:prompt` | `ctx:react`, `ctx:typing` |
| Components on the reply, with their handlers | Fetching the server, channel or members |
| | Channel and guild API calls |

`ctx.botPresent` tells you which case applies:

```luau
run = function(ctx, message)
    ctx:reply("Quoted!")
    if ctx.botPresent then
        ctx:react("💬")   -- only where the bot can see the channel
    end
end
```

| Field | Meaning |
| --- | --- |
| `ctx.isUserInstall` | Triggered through a user install only |
| `ctx.isGuildInstall` | The app is installed in this server |
| `ctx.botPresent` | The bot itself can act here: a server it is installed in, or its own DM |
| `ctx.where` | `"guild"`, `"botDm"` or `"privateChannel"` |
| `ctx.authorizingOwners` | Discord's raw `authorizing_integration_owners` |

`botPermissions` checks are skipped where `ctx.botPresent` is false, since the bot has no permissions there.

!!! note
    A server can stop external apps from posting publicly. In such servers, Discord shows user-installed responses as ephemeral, whatever you ask for.

## Example

The showcase's *Quote* message command works as a user install. See [Example bots](../examples/bundled.md#showcase).
