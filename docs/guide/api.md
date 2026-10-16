# Calling the API

Beyond replying, most bots need to call Discord's REST API: ban someone, create a role, edit a channel, read message history. discord gives you two layers:

- **`ctx.api`** (also `bot.api`): functions for common endpoints, grouped by resource, which take friendly arguments.
- **`ctx.rest`** (also `bot.rest`): the raw REST client, for any endpoint.

Both handle authentication, rate limits, retries, file uploads and audit log reasons.

## `ctx.api`

```luau
local api = ctx.api

api.channels.send(channelId, "Hello")                          -- any message input
api.channels.messages(channelId, { limit = 50 })
api.channels.bulkDelete(channelId, ids, "cleanup")
api.guilds.ban(guildId, userId, { deleteMessageSeconds = 86400, reason = "spam" })
api.guilds.timeout(guildId, userId, 600, "calm down")          -- 10 minutes
api.guilds.addRole(guildId, userId, roleId, "self-assign")
api.guilds.createRole(guildId, { name = "Muted", color = "#888888", permissions = {} })
api.users.dm(userId, { content = "Hi!" })
```

The groups are `api.channels`, `api.guilds`, `api.users`, `api.commands`, `api.interactions`, `api.webhooks`, `api.application`, `api.stickers`, `api.invites`, plus `api.gateway()` and `api.request(...)`. Every function is listed in the [api reference](../reference/api.md).

Functions that send or edit messages accept the same input as `ctx:reply` (see [Message content](messages.md)), including `files`.

Where the API wants a permission bitfield or a color, the wrappers accept names and hex strings:

```luau
api.guilds.createRole(guildId, {
    name = "Helpers",
    permissions = { "ManageMessages", "MuteMembers" },
    color = "#57F287",
    hoist = true,
})

api.channels.setPermission(channelId, roleId, "role", { "ViewChannel" }, { "SendMessages" }, "read-only")
```

## Return values

Every API function returns two values:

| Outcome | Returns |
| --- | --- |
| Success with a body | the decoded JSON (a table) |
| Success without a body (204) | `true` |
| Failure | `nil`, and an error |

They **don't raise** on HTTP errors, so check the result:

```luau
local channel, err = ctx.api.guilds.createChannel(ctx.guildId, { name = "logs", type = 0 })
if channel == nil then
    return ctx:error(`Couldn't create the channel: {err.message or tostring(err)}`)
end
```

The same applies to `ctx:reply`, `ctx:send` and the other context methods.

## Errors

The error is a table with these fields:

| Field | Example |
| --- | --- |
| `status` | `403` (absent for network errors) |
| `code` | `50013`, Discord's [JSON error code](https://docs.discord.com/developers/topics/opcodes-and-status-codes#json) |
| `message` | `"Missing Permissions"` |
| `errors` | Discord's detailed field errors, for 400s |
| `method`, `route` | `"PUT"`, `"/guilds/1/bans/2"` |

`tostring(err)` gives a one-line summary:

```text
PUT /guilds/1/bans/2: 403 (code 50013) Missing Permissions
POST /channels/1/messages: 400 (code 50035) Invalid Form Body {"components":{"0":{...}}}
```

`discord.Rest.isError(value)` tells an error apart from other values.

Common codes:

| Code | Meaning |
| --- | --- |
| `10003` / `10008` / `10013` | Unknown channel / message / user |
| `50001` | Missing access (the bot can't see the channel) |
| `50007` | Can't DM this user |
| `50013` | Missing permissions (often: the target's role is above the bot's) |
| `50035` | Invalid form body: see `err.errors` |
| `40060` | Interaction already acknowledged |

## Audit log reasons

Functions that change things take an optional `reason`, shown in the server's audit log:

```luau
api.guilds.kick(guildId, userId, `kicked by {ctx.user.username}: spamming`)
api.channels.deleteMessage(channelId, messageId, "rule 3")
```

## Rate limits

You don't have to do anything about rate limits:

- Each route bucket tracks Discord's `X-RateLimit-*` headers, and requests in the same bucket wait their turn.
- On a `429`, the client waits for `retry_after` and retries. Global limits pause every request.
- Network errors and `5xx` responses are retried up to 4 attempts, with a short backoff.

Requests to the same bucket (for example, messages in the same channel) are sent **one at a time**. Requests to different buckets run in parallel.

`debugRest = true` on the bot prints every request and every wait.

## Endpoints without a wrapper

Use `api.request` or `ctx.rest` for anything else. The route is the path after `/api/v10`:

```luau
-- GET with query parameters
local members = ctx.api.request("GET", `/guilds/{guildId}/members`, { query = { limit = 1000 } })

-- the REST client's shortcuts
local thread = ctx.rest:post(`/channels/{channelId}/threads`, { name = "Discussion", type = 11 }, { reason = "auto" })
local event = ctx.rest:get(`/guilds/{guildId}/scheduled-events/{eventId}`)
ctx.rest:patch(`/guilds/{guildId}/members/{userId}`, { nick = "New nick" })
ctx.rest:put(`/guilds/{guildId}/members/{userId}/roles/{roleId}`, nil, { reason = "verified" })
ctx.rest:delete(`/channels/{channelId}`)
```

`rest:request(method, route, options)` options:

| Option | Description |
| --- | --- |
| `body` | Encoded as JSON |
| `query` | Query string parameters |
| `files` | Files to upload; the body is sent as `payload_json` in a multipart request |
| `reason` | Audit log reason |
| `auth` | `false` to send no `Authorization` header (webhook and interaction endpoints) |
| `headers` | Extra headers |

!!! tip "Empty objects vs. arrays"
    Sonata's JSON encoder can't tell an empty table `{}` apart from an empty array. When an endpoint needs one or the other explicitly, use `json.object()` / `json.array()` from `@sonata/json`.

## Webhooks

```luau
local id, token = ctx.api.webhooks.parseUrl("https://discord.com/api/webhooks/123/abc")
ctx.api.webhooks.execute(id, token, {
    content = "Deployed!",
    username = "CI",
    avatarUrl = "https://example.com/ci.png",
    embeds = { ctx.ui.embed({ title = "v1.2.0", description = "All green", color = "#57F287" }) },
})
```

`execute` also takes `threadId`, `threadName`, and components (including v2) for webhooks owned by the application. Pass `wait = false` as the fourth argument to skip getting the message back.

## Without a bot

For scripts, make an API object from a REST client:

```luau
local discord = require("@discord")
local api = discord.makeApi(discord.Rest.new(TOKEN, { timeout = 10 }))

local guilds = api.users.guilds()
for _, g in guilds do
    print(g.name)
end
```

Functions that need the application id (`api.commands.*`, `api.interactions.*`, `api.application.emojis` and friends) raise *the application id isn't known yet* without a bot.
