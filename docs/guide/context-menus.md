# Context menu commands

Context menu commands appear when you right-click (or long-press) a **user** or a **message** and open **Apps**. They take no options, and the handler receives the user or message that was clicked.

## User commands

```luau
bot:userCommand({
    name = "User Info",
    run = function(ctx, user)
        local member = ctx.targetMember  -- the member, when used in a server
        ctx:reply({
            content = `{user.username} joined {member and member.joined_at or "?"}`,
            ephemeral = true,
        })
    end,
})
```

## Message commands

```luau
bot:messageCommand({
    name = "Quote",
    run = function(ctx, message)
        if message.content == "" then
            return ctx:error("That message has no text.")
        end
        ctx:reply(`> {message.content}\n— {ctx.fmt.user(message.author.id)}`)
    end,
})
```

`bot:userCommand(def)` and `bot:messageCommand(def)` are shorthand for `bot:command(def)` with `type = "user"` or `type = "message"`. In files, return `discord.userCommand({...})` / `discord.messageCommand({...})`, or a table with `type = "user"`.

## Names

Unlike slash commands, context menu names can have capitals and spaces (1–32 characters). A context menu command can share its name with a slash command, so `/userinfo` and *User Info* can live side by side, as can `/quote` and *Quote*.

## Fields on the context

| Field | User command | Message command |
| --- | --- | --- |
| second argument of `run` | the user | the message |
| `ctx.target` | the user | the message |
| `ctx.targetMember` | the member (in servers), with `.user` | — |
| `ctx.targetId` | the user id | the message id |

`ctx:react(emoji)` in a message command reacts to the **targeted** message.

## Allowed fields

Context menu commands support `permissions`, `botPermissions`, `guildOnly`, `ownerOnly`, `install`, `contexts`, `guilds`, `nsfw`, `cooldown`, `checks`, `ephemeral`, `defer` and `nameLocalizations`. Adding `options` or `subcommands` raises an error. A `description` is accepted but not sent, since Discord doesn't show one, and `prefix` / `aliases` don't apply.

## Sharing code with a slash command

One file can return both commands:

```luau title="commands/userinfo.luau"
local discord = require("@discord")

local function card(ctx, user, member)
    return { content = `**{ctx:displayName(user, member)}** ({user.id})`, allowedMentions = false }
end

return {
    discord.command({
        name = "userinfo",
        description = "About a user",
        options = { { name = "user", type = "user" } },
        run = function(ctx, args)
            local user = args.user or ctx.user
            ctx:reply(card(ctx, user, user.member))
        end,
    }),
    discord.userCommand({
        name = "User Info",
        ephemeral = true,
        run = function(ctx, user)
            ctx:reply(card(ctx, user, ctx.targetMember))
        end,
    }),
}
```

## Limits

Discord allows far fewer context menu commands than slash commands per application. Check the current limits in [Discord's documentation](https://docs.discord.com/developers/interactions/application-commands). If a sync fails with a limit error, the log shows Discord's message.
