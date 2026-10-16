# Checks, permissions and cooldowns

Commands, component handlers and modal handlers can declare who may run them. discord checks these rules before the handler runs. If one fails, the user gets an ephemeral `❌` message explaining why, and the handler doesn't run.

```luau
bot:command({
    name = "ban",
    description = "Ban a member",
    guildOnly = true,
    permissions = { "BanMembers" },
    botPermissions = { "BanMembers" },
    cooldown = { seconds = 10, scope = "guild" },
    checks = {
        function(ctx)
            return ctx.channelId == MOD_CHANNEL, "Use this in the mod channel."
        end,
    },
    run = function(ctx, args) ... end,
})
```

## The rules, in order

| Field | Fails when | Message |
| --- | --- | --- |
| `ownerOnly = true` | the user isn't an [owner](#owners) | *Only the bot's owners can use this.* |
| `guildOnly = true` | used outside a server | *This only works in servers.* |
| `dmOnly = true` | used in a server | *This only works in DMs.* |
| `nsfw = true` | prefix command in a server channel that isn't age-restricted | *This only works in age-restricted channels.* |
| `permissions = { ... }` | the user lacks a permission (see below) | *You need these permissions: Ban Members* |
| `botPermissions = { ... }` | the bot lacks a permission in this channel | *I need these permissions: Manage Roles* |
| `checks = { fn, ... }` | a check fails | the check's message |
| `cooldown = ...` | the user is on cooldown | *Slow down! Try again in 5 seconds.* |

The first rule that fails stops the rest. Every rule except `nsfw` can also be set on a subcommand, where it overrides the command's value.

Checks run **before** [middleware](middleware.md), deferring and the handler.

## `permissions`

Who runs the command needs these permissions **in the channel** (role permissions plus channel overwrites; Administrator implies everything; the server owner has everything).

- For **slash and context menu commands**, `permissions` is registered as the command's *default member permissions*. **Discord** enforces it: members without them don't even see the command. Server admins can change this under *Server Settings > Integrations*, and discord respects their choice by not checking again.
- For **prefix commands, components and modals**, discord checks it at runtime.
- In DMs, `permissions` is ignored.

Permission names are the ones in [`permissions.Flags`](../reference/permissions.md#flags): `"BanMembers"`, `"ManageMessages"`, `"ModerateMembers"`, `"Administrator"` ... Matching is forgiving: `"ban_members"`, `"BAN_MEMBERS"` and `"banmembers"` all work. A single name, a list, or a bitmask number are all accepted.

## `botPermissions`

The bot's permissions in the channel. For interactions Discord sends them along (`app_permissions`). For prefix commands they're computed from the cached server, the bot's roles and the channel's overwrites (threads use their parent channel's overwrites).

Skipped in DMs, and where the bot isn't present ([user installs](user-installs.md)).

## Owners

`ownerOnly` handlers can only be used by the bot's owners. Unless you pass `owners = { "id", ... }` to `discord.new`, the owners are loaded on startup from the application: its owner, or every member of its team.

```luau
if ctx:isOwner() then ... end
if ctx.bot:isOwner(userId) then ... end
```

Owners also **skip cooldowns**.

## Custom checks

A check is a function that receives the context. It may yield (call the API, read a file, ...):

```luau
checks = {
    function(ctx)
        return ctx.store.premium[ctx.guildId] == true, "This server doesn't have premium."
    end,
    function(ctx)
        if isBlacklisted(ctx.user.id) then
            return "You're blocked from using this bot."  -- a string fails with that message
        end
        -- returning nothing (or true) passes
    end,
}
```

| Return | Result |
| --- | --- |
| `true`, or nothing | pass |
| `false` | fail with *You can't use this.* |
| `false, "message"` | fail with the message |
| `"message"` | fail with the message |

Checks run in order and stop at the first failure. To share checks, put them in a [helper file](../getting-started/project-structure.md#shared-code):

```luau title="commands/_checks.luau"
local checks = {}

function checks.inVoice(ctx)
    local state = ctx.store.voice and ctx.store.voice[ctx.user.id]
    return state ~= nil, "Join a voice channel first."
end

return checks
```

```luau
local checks = require("./_checks")
return { name = "play", checks = { checks.inVoice }, run = ... }
```

For checks that apply to *everything*, use [middleware](middleware.md) instead.

## Cooldowns

```luau
cooldown = 5                                                   -- 5 seconds per user
cooldown = { seconds = 30, scope = "guild" }                   -- once per 30 s per server
cooldown = { seconds = 60, scope = "channel", message = "One at a time in this channel!" }
cooldown = { seconds = 3600, scope = "global" }                -- once per hour for everyone
```

| `scope` | One cooldown per |
| --- | --- |
| `"user"` (default) | user |
| `"guild"` | server (per user in DMs) |
| `"channel"` | channel |
| `"global"` | the whole bot |

- The cooldown starts when the checks pass, before the handler runs, even if the handler then fails.
- Each subcommand has its own cooldown, because the key includes the full command path (`config prefix set`).
- The default message shows a relative timestamp, *Slow down! Try again in 5 seconds.* Set `message` to replace it.
- Cooldowns are kept in memory and reset on restart.
- Owners aren't affected.

## Checking by hand

The same tools are available inside handlers:

```luau
local ok, missing = ctx:hasPermission({ "KickMembers", "BanMembers" })
local ok, missing = ctx:botHasPermission("ManageRoles")
local mask = ctx:memberPermissions()
local mask = ctx:botPermissions()

local perms = ctx.permissions
perms.has(mask, "ManageMessages")
perms.names(mask)                    -- { "AddReactions", "SendMessages", ... }
perms.label("ManageMessages")        -- "Manage Messages"
```

Checking against another member, for example making sure a moderator outranks their target, is up to you. `ctx:fetchGuild().roles` has the role positions.

See the [permissions reference](../reference/permissions.md).
