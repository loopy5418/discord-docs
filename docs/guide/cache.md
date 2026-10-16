# The cache

The bot keeps what it sees on the gateway in memory, so lookups like "this server's roles" or "this channel's overwrites" don't always need an API call. The cache is `bot.cache`, also `ctx.cache`.

## What is cached

| Data | Filled from | Removed on |
| --- | --- | --- |
| `cache.guilds[id]` | `GUILD_CREATE`, `GUILD_UPDATE` | `GUILD_DELETE` (when the bot leaves) |
| `guild.roles` (a list) | `GUILD_CREATE`, `GUILD_ROLE_CREATE` / `_UPDATE` | `GUILD_ROLE_DELETE` |
| `cache.channels[id]` | `GUILD_CREATE` (channels and threads), `CHANNEL_*`, `THREAD_*`, `THREAD_LIST_SYNC` | `CHANNEL_DELETE`, `THREAD_DELETE`, the guild leaving |
| `cache.users[id]` | message authors and mentions, interaction users, members, `USER_UPDATE` | never |
| `cache.members[guildId][userId]` | `GUILD_CREATE` members, `GUILD_MEMBER_ADD` / `_UPDATE`, `GUILD_MEMBERS_CHUNK`, message and interaction members | `GUILD_MEMBER_REMOVE`, the guild leaving |

The `bot:fetch*` / `ctx:fetch*` lookups add what they fetch from the API, too.

What isn't cached: messages, presences, voice states, emojis and stickers (beyond what the guild object carries), and members of large servers that haven't been seen yet.

!!! note "Shapes"
    Cached guilds are the gateway's guild objects without their `channels`, `threads`, `members`, `presences` and `voice_states` (channels and members live in their own tables). Cached members have no `user` field (it's in `cache.users`) and no `permissions` (interaction members carry computed permissions that would go stale).

## Reading the cache

```luau
local guild = ctx.cache:guild(guildId)           -- or ctx.cache.guilds[guildId]
local channel = ctx.cache:channel(channelId)
local user = ctx.cache:user(userId)
local member = ctx.cache:member(guildId, userId)
local role = ctx.cache:role(guildId, roleId)

for id, guild in ctx.cache.guilds do
    print(guild.name, guild.member_count)
end
```

These return `nil` for anything not cached. Usually you want the lookups, which fall back to the API:

```luau
ctx:fetchGuild()            bot:fetchGuild(id)
ctx:fetchChannel(id?)       bot:fetchChannel(id)
ctx:fetchMember(userId?)    bot:fetchMember(guildId, userId)   -- with .user attached
ctx:fetchUser(id)           bot:fetchUser(id)
ctx:fetchRole(idOrName)
```

## Keeping members complete

Discord only sends every member on startup for small servers. To fill the cache for a large server, request the members (needs the `GuildMembers` intent):

```luau
bot:on("GUILD_CREATE", function(ctx, guild)
    ctx.bot.gateway:requestGuildMembers(guild.id, { query = "", limit = 0 })
end)
```

With `GuildMembers`, member joins, updates and leaves keep the cache current afterwards.

## Turning it off

```luau
discord.new({ cache = false })
```

Gateway events then no longer fill the cache, so permission checks for prefix commands and lookups go to the API. Users and members fetched by `bot:fetchUser` / `bot:fetchMember` are still remembered. Interactions are unaffected, since Discord sends the permissions with them.

`bot.cache:clear()` empties every table.

## Memory

Users and members accumulate as the bot sees them and are only dropped when a member leaves or the bot leaves a server. For bots in very many servers, consider `cache = false` and doing your own caching.

See the [Cache reference](../reference/cache.md).
