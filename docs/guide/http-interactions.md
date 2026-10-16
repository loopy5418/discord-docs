# HTTP interactions

Normally, interactions (slash commands, buttons, modals) arrive over the gateway connection. Discord can instead **POST** each interaction to a public URL you choose. discord supports this:

- **HTTP only**: no gateway connection at all. Good for serverless-style deployments and bots that only use interactions.
- **Both**: the gateway for events and prefix commands, and HTTP for interactions.

## HTTP only

```luau
local discord = require("@discord")

local bot = discord.new({
    gateway = false,
    http = { port = 8080 },     -- publicKey defaults to the application's
})

bot:command({
    name = "hello",
    description = "Answered over HTTP",
    run = function(ctx)
        ctx:reply(`Hello, {ctx:displayName()}!`)
    end,
})

bot:start()
```

On start it prints the endpoint's address (`[http] interactions endpoint on ...:8080/interactions`) and syncs the commands. Then point Discord at it:

1. Make the endpoint reachable from the internet over **HTTPS**: a reverse proxy, a tunnel, or a host with TLS.
2. In the [Developer Portal](https://discord.com/developers/applications), open **General Information** and set **Interactions Endpoint URL** to `https://your.host/interactions`.
3. Save. Discord sends a test `PING` and some invalid requests to check the endpoint. **The bot must be running** when you save.

!!! warning
    While an Interactions Endpoint URL is set, Discord sends interactions **only** there, not over the gateway. Remove the URL to go back to gateway interactions.

## Options

`http = { ... }` in the bot options, or `bot:listen({ ... })`:

| Option | Default | Description |
| --- | --- | --- |
| `publicKey` | fetched from the application | The **Public Key** from General Information (hex) |
| `port` | `8080` | Port to listen on |
| `host` | all interfaces | Address to bind |
| `path` | `"/interactions"` | URL path |
| `responseTimeout` | `2.8` | Seconds to wait for the handler's first response (`bot:listen` only) |

`bot:listen(options)` starts the endpoint on its own, before `bot:run()`. It returns the httpx server (`server.url`, `server:close()`).

## Both gateway and HTTP

```luau
local bot = discord.new({
    prefix = "!",
    http = { port = 8080 },  -- the gateway stays on
})
```

Prefix commands and events come over the gateway, interactions over HTTP.

## How it works

- Each request's **ed25519 signature** is checked against the public key. Invalid requests get `401`, which is required by Discord.
- `PING` requests are answered with `PONG`.
- Other interactions go through the same routing as gateway interactions, so commands, components, modals, checks and middleware all work unchanged.
- The **first response** (`reply`, `defer`, `update`, `showModal`, autocomplete results) is sent back in the HTTP response body. Everything after it (follow-ups, edits) goes through the REST API as usual.
- A first response that has **files** can't go in the HTTP body. discord defers in the HTTP response, then edits the original message with the files.
- If no handler answers within `responseTimeout` (2.8 s), the request gets `500` and the user sees *"This interaction failed"*. With `autoDefer` (on by default) commands are deferred at about 2.3 s, so this mostly matters for **component and modal handlers**: give slow ones `defer = "update"` or call `ctx:deferUpdate()` early.

## What doesn't exist without the gateway

With `gateway = false`:

- no gateway events (`bot:on(...)` handlers never fire, `bot:waitFor` / `ctx:awaitMessage` time out)
- no prefix commands
- no presence (`setStatus`)
- no cache filled from events, so lookups use the API
- `ctx:latency()` is `nil`

On startup the bot fetches its user and application over REST instead of waiting for `READY`, then syncs commands.

## Example

The repository's `http` example: `sn run -- http --token <token> --port 8080`. See [Example bots](../examples/bundled.md#http).
