# How it works

A tour of what happens between Discord and your handler, for when you're debugging, contributing, or curious. You don't need any of this to write a bot.

## The pieces

| File | Role |
| --- | --- |
| `lib/gateway/gateway.luau` | One WebSocket connection: identify, heartbeat, resume, reconnect |
| `lib/rest.luau` | HTTP requests with rate limit buckets and retries |
| `lib/api.luau` | Endpoint wrappers on top of `Rest` |
| `lib/init.luau` | `Bot`: registration, dispatch, routing, checks, syncing, running |
| `lib/context.luau` | `Context`: the object handlers get, and how responses are sent |
| `lib/commands.luau` | Definition validation, registration JSON, option resolution, prefix parsing |
| `lib/components.luau` | Builders and the inline handler registry |
| `lib/messages.luau` | Message input → JSON body + files |
| `lib/cache.luau` | Gateway-fed cache |
| `lib/permissions.luau` | 53-bit permission math |
| `lib/loader.luau` | File, folder, glob and asset loading |
| `lib/http.luau` | The HTTP interactions endpoint |

Everything network-related goes through [httpx](https://github.com/loopy5418/httpx): `httpx.client` for REST, `httpx.websocket.connect` for the gateway, `httpx.app` for the HTTP endpoint. Concurrency uses Sonata's `@sonata/task` (tasks, channels, mutexes, events).

## From the gateway to a handler

1. **Gateway**: a message arrives on the WebSocket, is decoded, and if it's a dispatch (`op 0`) the gateway client emits it to its `"*"` handler, in a new task.
2. **`Bot:_dispatch(event, data)`**:
    1. The cache applies the event.
    2. On `READY`: `bot.user` and the application id are recorded; the first time, owners are loaded and commands synced in the background, then `bot.ready` is set.
    3. **Waiters** (`bot:waitFor`) whose filter matches get the data through a channel.
    4. `INTERACTION_CREATE` → `_handleInteraction` and `MESSAGE_CREATE` → `_handleMessage`, each in a new task.
    5. Your `bot:on` handlers for the event, then `"*"` handlers, each in a new task with a fresh event context.

## Interactions

`_handleInteraction` builds a context with `Context.fromInteraction`, which sets `kind` from the interaction type and fills in user, member, guild, channel, install info and (for components/modals) the parsed custom id.

- **Application commands** are looked up in `bot.commands` by `"slash:name"` / `"user:name"` / `"message:name"`. For slash commands, `commands.resolveLeaf` walks subcommand groups to the leaf, and `commands.resolveOptions` turns raw options into values (swapping ids for resolved objects, applying `default`s).
- **Autocomplete** finds the focused option's handler (option → subcommand → command), runs it through middleware, and responds with the result.
- **Components and modals** go through `_routeComponent`: collectors → inline callbacks → registered handlers by name → patterns.

Everything that runs user code goes through `_runCommand` / `_routeComponent`:

1. `_check`: owner, guild/DM, NSFW, permissions, bot permissions, custom checks, cooldown. A failure → `ctx:error(message)`.
2. Defer if the definition says so; otherwise, for commands with `autoDefer`, schedule a defer at 2.3 s if nothing has answered.
3. `_invoke`: build the middleware chain, call the handler last, all inside `xpcall` with a traceback. Errors → `_handleError` → error handlers and the error message.

## Responding

The context tracks `responded`, `deferred`, `replied` and `deferredUpdate`, and picks the right Discord call:

- First response → `POST /interactions/{id}/{token}/callback` (type 4, 5, 6, 7, 8 or 9), with `?with_response=true` for messages so the created message comes back.
- After a defer → `PATCH /webhooks/{app}/{token}/messages/@original`.
- Later → `POST /webhooks/{app}/{token}` (follow-ups).

Over HTTP, the first response is handed to a `transport` instead, which sends it as the HTTP response body.

## Prefix commands

`_handleMessage` skips bots, webhooks, the bot itself and empty messages, finds the longest matching prefix (static, list, function, or mention), looks the first word up in `bot.prefixCommands`, walks subcommands word by word, then `commands.parseArgs` converts the remaining text using the leaf's options. Lookups for users/members/channels/roles go through the context (cache, then API). Parse errors reply with the error and `commands.usage(...)`.

## Inline handlers and collectors

- `ui.button({ run = fn })` calls `ui.callback(fn, ttl, once)`, which stores the function in a module-level table under a random `~cb:` id with an expiry. Expired entries are swept lazily.
- `ctx:awaitComponent`, `ctx:collect`, `ctx:prompt`, `ctx:confirm` and `ctx:paginate` register **collectors** on the bot (`_addCollector`) with a match function; the router offers each component/modal context to collectors first. `confirm`/`paginate` use `~confirm:`/`~page:` ids with a random nonce.

## Syncing

`syncCommands` turns each definition into JSON (`commands.toJSON`), groups by scope, hashes each scope's JSON with SHA-256, and `PUT`s (bulk-overwrites) only scopes whose hash differs from `.commands.lock.json`. Unchanged scopes are listed to learn command ids. Scopes that existed before but are now empty are overwritten with an empty list.

## Rate limits

Each `METHOD route-with-ids-folded major-id` key gets a bucket with a mutex, `remaining` and `resetAt`. A request locks its bucket, waits out the global limit and the bucket's reset if exhausted, sends, reads the `X-RateLimit-*` headers, and on 429 records the wait and retries. Different buckets don't block each other.

## The gateway connection

`Gateway:run` loops: fetch the URL (once), connect, and run `_connectOnce` until the socket closes. `_connectOnce` returns whether to resume, re-identify, stop, or give up (fatal close code). Heartbeats run in their own task per connection; a missing ACK closes with 4000 to resume. Failures back off exponentially up to 60 s.

## Testing

`tests/offline.luau` swaps `bot.rest.request` for a fake that records calls and returns canned responses, then calls `bot:_handleInteraction` and `bot:_handleMessage` with hand-built payloads. The same approach works for testing your own bot without a network.
