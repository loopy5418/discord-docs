# Gateway

```luau
local gw = bot.gateway
-- or, without a bot:
local gateway = require("@discord/lib/gateway/gateway")
local gw = gateway.new({ token = TOKEN, intents = { "Guilds" } })
```

The low-level gateway client: one WebSocket connection with heartbeats, resuming and reconnecting. The bot uses it internally; you rarely need more than `bot.gateway.latency` and `requestGuildMembers`.

## `gateway.new`

```luau
gateway.new(options: {
    token: string,                         -- with or without "Bot "
    intents: ({ string | number } | number)?,  -- default Intent.Default
    presence: table?,                      -- sent with identify
    debug: boolean?,                       -- print every payload (tokens redacted)
}) -> Gateway
```

The module also exports `gateway.intents` (the [intents](intents.md) module).

## Fields

| Field | Description |
| --- | --- |
| `latency` | Seconds between the last heartbeat and its ACK; `nil` before the first |
| `intents` | The intent mask |
| `user` | The bot user from `READY` |
| `sessionId`, `resumeUrl`, `seq` | Session state for resuming |
| `running` | Whether `run` is active |
| `ws` | The current httpx WebSocket, or `nil` |

## Methods

### `gw:on`

```luau
gw:on(event: string, handler: (data: any) -> ()) -> Gateway
gw:on("*", handler: (event: string, data: any) -> ())
```

Adds a dispatch handler (each call runs in its own task). Warns about events the intents won't deliver, and about message events without `MessageContent`. Note that on the gateway client, handlers receive only the data, not a context.

### `gw:run`

```luau
gw:run() -> (boolean, string?)
```

Connects and blocks. Returns `true` after `stop()`, or `false, reason` on a fatal close code:

| Code | Reason |
| --- | --- |
| 4004 | authentication failed (bad token); also returned when `GET /gateway/bot` answers 401 |
| 4010 | invalid shard |
| 4011 | sharding required |
| 4012 | invalid API version |
| 4013 | invalid intents |
| 4014 | disallowed intents (the privileged intents are named in the reason) |

### `gw:stop`

```luau
gw:stop()
```

Closes with code 1000 and makes `run` return.

### `gw:setPresence`

```luau
gw:setPresence(presence: { status: string?, activities: { table }?, afk: boolean?, since: number? }) -> (boolean, string?)
```

### `gw:requestGuildMembers`

```luau
gw:requestGuildMembers(guildId: string, options: {
    query: string?,         -- name prefix; "" with limit 0 for everyone (needs GuildMembers)
    limit: number?,         -- default 0
    user_ids: { string }?,
    presences: boolean?,
    nonce: string?,
}?) -> (boolean, string?)
```

Members arrive as `GUILD_MEMBERS_CHUNK` events.

### `gw:send`

```luau
gw:send(op: number, data: any) -> (boolean, string?)
```

Sends a raw payload. Returns `false, "not connected"` without a connection.

## Connection behaviour

- **Identify / resume**: the first connection identifies. After a drop, it resumes at `resume_gateway_url` with the session id and sequence, so missed events are replayed.
- **Heartbeats**: the first beat waits `interval × random(0..1)`, as Discord asks. If a beat isn't ACKed before the next one, the connection is treated as a zombie and reconnected (resuming).
- **Opcodes**: `HEARTBEAT` from Discord is answered at once; `RECONNECT` resumes; `INVALID_SESSION` waits 1–5 s, then resumes if Discord allows, else identifies anew.
- **Close codes**: fatal codes (above) stop `run`; 4007 and 4009 re-identify; anything else resumes.
- **Backoff**: repeated failures wait 2, 4, 8 ... up to 60 seconds; a connection that lived over a minute resets it.
- Messages up to 64 MiB are accepted (large `GUILD_CREATE`s).

The gateway URL comes from `GET /gateway/bot`, connecting with `?v=10&encoding=json`. There is no sharding support; one connection serves the bot.

## Codes

`require("@discord/lib/gateway/codes")` has `OP` (opcode names), `FATAL_CLOSE`, `NO_RESUME_CLOSE` and `RESUME_CLOSE` (4000, used when closing to resume).
