# Rest

```luau
local rest = discord.Rest.new(token, options?)   -- or bot.rest / ctx.rest
```

The HTTP client for Discord's API (`https://discord.com/api/v10`), built on [httpx](https://github.com/loopy5418/httpx).

## `Rest.new`

```luau
Rest.new(token: string, options: { debug: boolean?, timeout: number? }?) -> Rest
```

| Option | Default | Description |
| --- | --- | --- |
| `debug` | `false` | Print every request and rate limit wait |
| `timeout` | `30` | Request timeout in seconds |

A leading `Bot ` in the token is removed.

## `rest:request`

```luau
rest:request(method: string, route: string, options: RequestOptions?) -> (any, DiscordError?)
```

`route` is the path after `/api/v10`, e.g. `"/channels/123/messages"`.

| Option | Type | Description |
| --- | --- | --- |
| `body` | `any` | Sent as JSON |
| `query` | `{ [string]: any }` | Query parameters |
| `files` | `{ File }` | Upload as `multipart/form-data`, with `body` as `payload_json` and `attachments` filled in (existing `body.attachments` are kept first) |
| `reason` | `string` | `X-Audit-Log-Reason` (URL-encoded) |
| `auth` | `boolean` | `false` sends no `Authorization` header |
| `headers` | `{ [string]: string }` | Extra headers |

Returns the decoded JSON, `true` for empty successful responses, or `nil` and an error.

## Shortcuts

```luau
rest:get(route, options?)
rest:post(route, body?, options?)
rest:patch(route, body?, options?)
rest:put(route, body?, options?)
rest:delete(route, options?)
```

## Rate limits and retries

- Requests are grouped into buckets by method and route, with ids folded together except the major parameters (channel, guild, webhook and interaction ids). Reactions share one bucket per channel.
- Each bucket processes one request at a time and waits when `X-RateLimit-Remaining` reaches 0, until `X-RateLimit-Reset-After`.
- On `429`, the client waits `retry_after` seconds (globally if the limit is global) and retries.
- Network errors and `5xx` responses are retried after 1, 2, 3 seconds.
- At most 4 attempts per request.

## Errors

Failures return a `DiscordError`:

| Field | Description |
| --- | --- |
| `status` | HTTP status (absent for network errors) |
| `code` | Discord's JSON error code |
| `message` | Discord's message, the response body, or the network error |
| `errors` | Detailed field errors |
| `method`, `route` | The request |

`tostring(err)` → `"PUT /guilds/1/bans/2: 403 (code 50013) Missing Permissions"`.

### `Rest.isError`

```luau
Rest.isError(value: any) -> boolean
```

## Other members

| Member | Description |
| --- | --- |
| `Rest.API` | `"https://discord.com/api/v10"` |
| `Rest.encodeComponent(s)` | Percent-encodes everything except unreserved characters |
| `rest.token` | The token |
| `rest.client` | The underlying httpx client |
