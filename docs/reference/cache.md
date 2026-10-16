# Cache

```luau
local cache = bot.cache   -- or ctx.cache
```

The in-memory cache filled from gateway events. See [The cache](../guide/cache.md).

## Fields

| Field | Type | Description |
| --- | --- | --- |
| `guilds` | `{ [id]: Guild }` | Guild objects without `channels`, `threads`, `members`, `presences`, `voice_states`; `roles` is a list |
| `channels` | `{ [id]: Channel }` | Guild channels and threads (with `guild_id` set), and channels fetched through lookups |
| `users` | `{ [id]: User }` | Users seen anywhere |
| `members` | `{ [guildId]: { [userId]: Member } }` | Members without `user` and `permissions` |
| `enabled` | `boolean` | `false` when the bot was created with `cache = false`; events are then ignored |

## Reading

| Method | Returns |
| --- | --- |
| `cache:guild(id)` | `Guild?` |
| `cache:channel(id)` | `Channel?` |
| `cache:user(id)` | `User?` |
| `cache:member(guildId, userId)` | `Member?` |
| `cache:role(guildId, roleId)` | `Role?` from the cached guild's roles |

## Writing

| Method | Description |
| --- | --- |
| `cache:putUser(user)` | Adds a user, or merges fields into the cached one |
| `cache:putMember(guildId, member, userId?)` | Stores a member (its `user` goes to `users`; `permissions` is dropped). `userId` is needed when `member.user` is absent. |
| `cache:putChannel(channel, guildId?)` | Stores a channel, setting `guild_id` if given and missing |
| `cache:handle(event, data)` | Applies a gateway dispatch (called by the bot for every event) |
| `cache:clear()` | Empties every table |

## Events handled

`GUILD_CREATE`, `GUILD_UPDATE`, `GUILD_DELETE`, `GUILD_ROLE_CREATE`, `GUILD_ROLE_UPDATE`, `GUILD_ROLE_DELETE`, `CHANNEL_CREATE`, `CHANNEL_UPDATE`, `CHANNEL_DELETE`, `THREAD_CREATE`, `THREAD_UPDATE`, `THREAD_DELETE`, `THREAD_LIST_SYNC`, `GUILD_MEMBER_ADD`, `GUILD_MEMBER_UPDATE`, `GUILD_MEMBER_REMOVE`, `GUILD_MEMBERS_CHUNK`, `MESSAGE_CREATE` (author, member, mentions), `USER_UPDATE`, `INTERACTION_CREATE` (user, member).

`GUILD_CREATE` / `GUILD_DELETE` for an *unavailable* guild (an outage) is ignored, so the cached data is kept.

## Standalone

```luau
local Cache = require("@discord/lib/cache")
local cache = Cache.new()
gateway:on("*", function(event, data) cache:handle(event, data) end)
```
