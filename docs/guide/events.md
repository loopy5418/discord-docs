# Events

The gateway sends the bot **events** as things happen: a message is posted, a member joins, a role changes. Each handler gets a [context](context.md) (with `ctx.kind == "event"`) and the event's data.

```luau
bot:on("GUILD_MEMBER_ADD", function(ctx, member)
    ctx.api.channels.send(WELCOME_CHANNEL, `Welcome, {ctx.fmt.user(member.user.id)}!`)
end)
```

## Registering handlers

```luau
bot:on(event, function(ctx, data) end)     -- every time
bot:once(event, function(ctx, data) end)   -- the next time only
bot:off(event, handler)                    -- remove one handler
bot:off(event)                             -- remove every handler for the event
```

`on` and `once` return the bot, so calls can be chained.

In files, return an event definition:

```luau title="events/ready.luau"
return {
    event = "READY",
    once = true,
    run = function(ctx, data)
        print(`logged in as {data.user.username} in {#data.guilds} server(s)`)
    end,
}
```

`discord.event({ ... })` tags the table explicitly. `bot:event({ ... })` registers one directly.

### Event names

Names can be written the way Discord sends them, or in camelCase or snake_case. All of these are the same event:

```luau
bot:on("MESSAGE_CREATE", fn)
bot:on("messageCreate", fn)
bot:on("message_create", fn)
```

### Every event

`"*"` receives every event. Its handlers get the event name as an extra argument:

```luau
bot:on("*", function(ctx, name, data)
    print("event", name)
end)
```

## How handlers run

- Each handler runs in its **own task**, so it can yield (make API calls, `ctx:sleep`, wait for things) without holding up other events.
- An error in a handler goes to [`bot:onError`](middleware.md#error-handlers) with the event context. The bot doesn't reply to anyone.
- Event handlers **don't** go through middleware or checks. Those are for commands, components and modals.
- `once` handlers are removed before they run, so they fire exactly once even if two events arrive together.

For each event, things happen in this order:

1. The [cache](cache.md) is updated.
2. `READY` only: the bot's own startup work.
3. [Waiters](waiting.md#ctxwaitfor-botwaitfor) (`bot:waitFor`) get the event.
4. `INTERACTION_CREATE` and `MESSAGE_CREATE` are routed to commands and handlers.
5. Your handlers for this event run, then the `"*"` handlers.

## The event context

| Field | Description |
| --- | --- |
| `ctx.event` | The event name, e.g. `"MESSAGE_CREATE"` |
| `ctx.eventData` | The payload (the same as the handler's second argument) |
| `ctx.guildId`, `ctx.channelId` | From the payload's `guild_id` / `channel_id`, when present |
| `ctx.message`, `ctx.content` | For `MESSAGE_CREATE` and `MESSAGE_UPDATE` |
| `ctx.user` | The message author, or the payload's `user` |
| `ctx.member` | The payload's `member`, when present |

The usual helpers work too. `ctx:reply` and `ctx:react` answer the message for `MESSAGE_CREATE`, and `ctx:send` posts to the event's channel:

```luau
bot:on("messageCreate", function(ctx, message)
    if message.author.bot then
        return
    end
    if string.find(string.lower(message.content), "good bot") then
        ctx:react("❤️")
    end
end)
```

!!! warning "Don't answer yourself"
    `MESSAGE_CREATE` fires for the bot's own messages too. Check `message.author.bot`, or compare with `ctx.bot.user.id`, before replying, or the bot may end up replying to itself in a loop.

## Common events

The warning column shows what discord prints when you register a handler for an event your [intents](intents.md) won't deliver:

```text
[bot] warning: "GUILD_MEMBER_ADD" needs one of these intents, which the bot doesn't have: GuildMembers
```

| Event | Data | Intent |
| --- | --- | --- |
| `READY` | `user`, `guilds` (unavailable stubs), `session_id`, `application` | none |
| `RESUMED` | — | none |
| `GUILD_CREATE` | the full server, with `channels`, `threads`, `roles`, `members` | `Guilds` |
| `GUILD_UPDATE` / `GUILD_DELETE` | the server / `{ id, unavailable }` | `Guilds` |
| `GUILD_ROLE_CREATE` / `_UPDATE` / `_DELETE` | `guild_id`, `role` / `role_id` | `Guilds` |
| `CHANNEL_CREATE` / `_UPDATE` / `_DELETE` | the channel | `Guilds` |
| `THREAD_CREATE` / `_UPDATE` / `_DELETE` | the thread | `Guilds` |
| `GUILD_MEMBER_ADD` / `_UPDATE` | the member, with `guild_id` and `user` | `GuildMembers` |
| `GUILD_MEMBER_REMOVE` | `guild_id`, `user` | `GuildMembers` |
| `GUILD_BAN_ADD` / `_REMOVE` | `guild_id`, `user` | `GuildModeration` |
| `MESSAGE_CREATE` / `_UPDATE` | the message | `GuildMessages` or `DirectMessages` |
| `MESSAGE_DELETE` | `id`, `channel_id`, `guild_id` | `GuildMessages` or `DirectMessages` |
| `MESSAGE_DELETE_BULK` | `ids`, `channel_id`, `guild_id` | `GuildMessages` |
| `MESSAGE_REACTION_ADD` / `_REMOVE` | `user_id`, `message_id`, `channel_id`, `emoji`, `member` | `GuildMessageReactions` or `DirectMessageReactions` |
| `TYPING_START` | `user_id`, `channel_id`, `timestamp` | `GuildMessageTyping` or `DirectMessageTyping` |
| `PRESENCE_UPDATE` | `user`, `status`, `activities` | `GuildPresences` |
| `VOICE_STATE_UPDATE` | the voice state | `GuildVoiceStates` |
| `INVITE_CREATE` / `_DELETE` | the invite | `GuildInvites` |
| `MESSAGE_POLL_VOTE_ADD` / `_REMOVE` | `user_id`, `message_id`, `answer_id` | `GuildMessagePolls` or `DirectMessagePolls` |
| `INTERACTION_CREATE` | the raw interaction (already routed to your commands) | none |

Discord's [gateway events documentation](https://docs.discord.com/developers/events/gateway-events) describes every event and its payload.

### Telling startup apart from joins

On startup, Discord sends a `GUILD_CREATE` for every server the bot is in. To react only to servers the bot *joins*, check `bot.ready`, which is set after startup:

```luau
bot:on("GUILD_CREATE", function(ctx, guild)
    if ctx.bot.ready:isSet() then
        print(`joined {guild.name} ({guild.member_count} members)`)
    end
end)
```

## Events from discord itself

| Event | When | Data |
| --- | --- | --- |
| `UNKNOWN_COMMAND` | A message starts with a prefix but names no command | `{ message, name, prefix }` |

`UNKNOWN_COMMAND` is not delivered to `"*"` handlers.

## Waiting for an event

`bot:waitFor` (also `ctx:waitFor`) blocks until a matching event arrives:

```luau
local data = bot:waitFor("GUILD_MEMBER_ADD", function(d)
    return d.guild_id == guildId and d.user.id == userId
end, 300) -- nil after 5 minutes
```

## Requesting members

Large servers don't send their full member list on startup. Ask the gateway for it, and the members arrive as `GUILD_MEMBERS_CHUNK` events, which also fill the cache:

```luau
bot.gateway:requestGuildMembers(guildId, { query = "", limit = 0 })    -- everyone (needs GuildMembers)
bot.gateway:requestGuildMembers(guildId, { user_ids = { "1", "2" } })  -- specific users
bot.gateway:requestGuildMembers(guildId, { query = "ab", limit = 10 }) -- names starting with "ab"
```

For a one-off lookup, `ctx.api.guilds.member(guildId, userId)` or `ctx.api.guilds.searchMembers(guildId, "ab")` over REST is often simpler.
