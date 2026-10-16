# Context

The first argument of every handler. See [The context](../guide/context.md) for an overview.

## Fields

### Libraries and state

| Field | Type | Description |
| --- | --- | --- |
| `bot` | Bot | The bot |
| `api` | api | [Endpoint wrappers](api.md) |
| `rest` | Rest | The [REST client](rest.md) |
| `cache` | Cache | The [cache](cache.md) |
| `ui` | | [Component builders](ui.md) |
| `fmt` | | [Formatting](format.md) |
| `enums` | | [Constants](enums.md) |
| `permissions` | | [Permissions](permissions.md) |
| `store` | table | `bot.store`, shared for the bot's lifetime |
| `state` | table | Empty table for this invocation |

### What happened

| Field | Type | Description |
| --- | --- | --- |
| `kind` | string | `"slash"`, `"user"`, `"message"`, `"prefix"`, `"component"`, `"modal"`, `"autocomplete"`, `"event"` |
| `isInteraction` | boolean | Created from an interaction |
| `isPrefix` | boolean | A prefix command |
| `createdAt` | number | Unix seconds when the context was created |

### Who and where

| Field | Type | Set for | Description |
| --- | --- | --- | --- |
| `user` / `author` | User | all (when known) | Who triggered it |
| `member` | Member? | servers | Interaction members include `permissions`; prefix members have `.user` attached |
| `guildId` | string? | | `nil` in DMs |
| `channelId` | string? | | |
| `channel` | table? | interactions | The partial channel sent with the interaction |
| `inGuild` / `inDM` | boolean | | |
| `where` | string? | interactions, prefix | `"guild"`, `"botDm"`, `"privateChannel"` |
| `locale` / `guildLocale` | string? | interactions | |
| `isUserInstall` | boolean | interactions, prefix | Authorized only by a user install |
| `isGuildInstall` | boolean | interactions, prefix | The app is installed in this server |
| `botPresent` | boolean | | Whether the bot can act here as itself |
| `authorizingOwners` | table? | interactions | Discord's `authorizing_integration_owners` |
| `appPermissions` | string? | interactions | The bot's permissions in the channel (a decimal string) |
| `entitlements` | `{ table }` | interactions | The user's entitlements |

### Interaction data

| Field | Description |
| --- | --- |
| `interaction` | The raw interaction |
| `id` / `token` | Interaction id and token |
| `data` | The interaction's `data` |
| `message` | Prefix: the command message. Components: the message the component is on. Modals from a component: that message. Events: the message for `MESSAGE_CREATE`/`MESSAGE_UPDATE`. |
| `content` | Prefix and message events: the message text |

### Commands

| Field | Set for | Description |
| --- | --- | --- |
| `command` | commands, autocomplete | The normalized command definition |
| `subcommandDef` | subcommands | The subcommand's definition |
| `commandName` | commands | The top-level name |
| `commandPath` | commands, components, modals | `"config prefix set"`; for registered components/modals the custom id or pattern |
| `group` / `subcommand` | subcommands | |
| `options` / `args` | slash, prefix, autocomplete | Values by option name |
| `rawArgs` | prefix | Words after the command (quotes removed) |
| `argText` | prefix | Text after the command name, as typed |
| `prefix` | prefix | The prefix used |
| `invokedWith` | prefix | The name or alias used |
| `target` | context menus | The user or message |
| `targetMember` | user commands in servers | The target's member, with `.user` |
| `targetId` | context menus | |
| `focused` | autocomplete | `{ name, value, type }` |

### Components and modals

| Field | Description |
| --- | --- |
| `customId` | The full custom id |
| `name` | The custom id before the first `:` (the whole id for `~` ids) |
| `params` | The parts after it, or the `match` captures |
| `values` | Select values (`{}` for buttons) |
| `componentType` | The component type number |
| `fields` | Modals: values by input custom id |
| `resolved` | Resolved users, members, roles, channels, attachments |

### Events

| Field | Description |
| --- | --- |
| `event` | The event name |
| `eventData` | The payload |

### Response state

| Field | Description |
| --- | --- |
| `responded` | The interaction was acknowledged (or a prefix reply was sent) |
| `deferred` | `defer` or `deferUpdate` was used |
| `replied` | A reply was sent (or filled in after a defer) |
| `ephemeral` | Whether the response is ephemeral |
| `ephemeralDefault` | The default for replies (from the definition's `ephemeral`) |
| `replyMessage` | The response message, once known |

## Responding

### `ctx:reply`

```luau
ctx:reply(input: MessageInput) -> (Message | true | nil, err?)
```

The first response; after `defer`, fills in the deferred message; afterwards, a follow-up. Prefix: a reply to the command message (`ephemeral` ignored), or a plain message if there is none. Raises in autocomplete contexts. See [Responding](../guide/responding.md).

### `ctx:followUp`

```luau
ctx:followUp(input: MessageInput) -> (Message?, err?)
```

A follow-up message (honours `ephemeral`). Before any response, same as `reply`. Prefix: same as `reply`.

### `ctx:defer`

```luau
ctx:defer(options: { ephemeral: boolean? }?) -> (any, err?)
```

Acknowledges with "thinking...". `ephemeral` defaults to `ephemeralDefault`. No-op if already responded. Prefix: shows the typing indicator.

### `ctx:deferUpdate`

```luau
ctx:deferUpdate() -> (any, err?)
```

Components and modals: acknowledges without changing the message. Raises outside interactions.

### `ctx:update`

```luau
ctx:update(input: MessageInput) -> (any, err?)
```

Components and modals opened from a component: edits the message the component is on (the first response, or an edit after it). Other kinds: same as `edit`.

### `ctx:edit` / `ctx:editReply`

```luau
ctx:edit(input: MessageInput) -> (Message?, err?)
```

Interactions: edits the original response; returns `nil, "nothing to edit yet: reply or defer first"` before one. Prefix: edits the bot's first reply, or replies if there is none.

### `ctx:deleteReply`

```luau
ctx:deleteReply() -> (any, err?)
```

Deletes the original response, or the bot's first prefix reply.

### `ctx:fetchReply`

```luau
ctx:fetchReply() -> (Message?, err?)
```

Fetches the original response (interactions), or returns the bot's first reply (prefix).

### `ctx:send`

```luau
ctx:send(input: MessageInput) -> (Message?, err?)
```

Posts a normal message in the channel through the bot user. Returns `nil, "this context has no channel"` without a channel.

### `ctx:showModal`

```luau
ctx:showModal(modal: table) -> (any, err?)
```

Opens a modal as the first response. `modal` is a `ui.modal(...)` result or a table `ui.modal` accepts. Raises outside interactions.

### `ctx:prompt`

```luau
ctx:prompt(modal: table, timeout: number?) -> Context?
```

Opens a modal and yields until the same user submits it; returns the submission's context, or `nil` after `timeout` seconds (default 300) or if the modal couldn't be shown.

### `ctx:respond`

```luau
ctx:respond(choices: { string | number | { name: string, value: any } }) -> (any, err?)
```

Autocomplete only (raises otherwise). Sends up to 25 choices; names are cut to 100 characters.

### `ctx:error` / `ctx:fail`

```luau
ctx:error(text: string) -> (any, err?)
```

Sends `❌ text` ephemerally with mentions disabled. On a public deferred response, deletes it and sends the error as an ephemeral follow-up. Autocomplete: responds with no choices.

## Messages

### `ctx:react`

```luau
ctx:react(emoji: string | table, messageId: string?) -> (any, err?)
```

Reacts to `messageId` in this channel, or by default: the targeted message (message commands), else `ctx.message`. Returns `nil, "no message to react to"` if there's none.

### `ctx:typing`

```luau
ctx:typing() -> (any, err?)
```

### `ctx:dm`

```luau
ctx:dm(input: MessageInput, userId: string?) -> (Message?, err?)
```

DMs the caller, or `userId`.

## Waiting

### `ctx:awaitComponent`

```luau
ctx:awaitComponent(options: {
    customId: (string | { string })?, customIds: { string }?,
    message: (string | Message)?,
    user: (string | { string } | false)?, users: { string }?, anyone: boolean?,
    filter: ((Context) -> boolean)?,
    timeout: number?,  -- default 60
}?) -> Context?
```

Yields until a matching component interaction arrives and returns its context, or `nil` on timeout. Default user: the caller. Without `customId` and `message`: components on `ctx.replyMessage`, or else in this channel.

### `ctx:collect`

```luau
ctx:collect(options: { ...awaitComponent options, max: number?, onEnd: ((reason: string) -> ())? },
            handler: (Context) -> ()) -> { stop: () -> (), count: () -> number }
```

Calls `handler` (in a new task) for each match. Ends after `timeout` idle seconds (default 120), `max` matches, or `stop()`; `onEnd` receives `"timeout"`, `"max"` or `"stopped"`. Doesn't yield.

### `ctx:awaitMessage`

```luau
ctx:awaitMessage(options: { timeout: number?, channelId: string?, user: (string | false)?, filter: ((Message) -> boolean)? }?) -> Message?
```

Yields until a `MESSAGE_CREATE` in the channel (default: this one) from the user (default: the caller; `false` for anyone) passing `filter`. Default timeout 60.

### `ctx:waitFor`

```luau
ctx:waitFor(event: string, filter: ((any) -> boolean)?, timeout: number?) -> any?
```

[`bot:waitFor`](bot.md#botwaitfor).

### `ctx:confirm`

```luau
ctx:confirm(question: MessageInput, options: { yes: string?, no: string?, timeout: number?, ephemeral: boolean?, user: (string | false)? }?) -> boolean?
```

Replies with the question plus Confirm/Cancel buttons; returns `true`, `false`, or `nil` on timeout (default 60) or if sending failed. Disables the buttons afterwards.

### `ctx:paginate`

```luau
ctx:paginate(pages: { MessageInput }, options: { timeout: number?, start: number?, ephemeral: boolean?, user: (string | false)?, wrap: boolean? }?)
```

Replies with page `start` (default 1) and ◀️ / counter / ▶️ buttons; yields until `timeout` idle seconds (default 120), then disables the buttons. `wrap` defaults to `true`. A single page gets no buttons and returns immediately.

## Permissions

| Method | Returns |
| --- | --- |
| `ctx:memberPermissions()` | The caller's permission mask in this channel. Interactions: from Discord. Prefix: computed from the cache/API. DMs: every permission. |
| `ctx:botPermissions()` | The bot's mask. Interactions: `app_permissions`. Otherwise computed; `0` before `READY`. |
| `ctx:hasPermission(perms)` | `ok: boolean, missing: { string }` |
| `ctx:botHasPermission(perms)` | `ok: boolean, missing: { string }` |
| `ctx:isOwner(userId?)` | Whether the caller (or `userId`) is an owner |

`perms` is a name, a list of names, or a mask. Threads use their parent channel's overwrites.

## Lookups

| Method | Returns |
| --- | --- |
| `ctx:fetchGuild()` | This server, or `nil` in DMs |
| `ctx:fetchChannel(channelId?)` | A channel (default: this one) |
| `ctx:fetchMember(userId?)` | A member with `.user` (default: the caller; uses `ctx.member` when it has roles) |
| `ctx:fetchUser(userId)` | A user; checks the message's mentions first |
| `ctx:fetchRole(idOrName)` | A role in this server by id or case-insensitive name |

## Conveniences

| Method | Returns |
| --- | --- |
| `ctx:getOption(name, default?)` | `ctx.options[name]`, or `default` |
| `ctx:field(customId, default?)` | `ctx.fields[customId]`, or `default` |
| `ctx:mention()` | `"<@id>"` for the caller |
| `ctx:displayName(user?, member?)` | Nickname, global name or username (default: the caller) |
| `ctx:latency()` | Gateway heartbeat round trip in seconds, or `nil` |
| `ctx:sleep(seconds?)` | `task.wait(seconds)` |
| `ctx:spawn(fn, ...)` | `task.spawn(fn, ...)` |
| `ctx:log(...)` | Prints `[commandPath or customId or event or kind] ...` |
| `ctx:commandMention(name)` | `</name:id>` for a synced slash command (the first word picks the command), or `"/name"` |
| `ctx:is(name)` | Whether `name` equals `ctx.name`, `ctx.customId` or `ctx.commandName` |
| `ctx:selected()` | Select values resolved to users (with `.member`), roles, channels or attachments |
| `ctx:uploads(customId)` | Attachments uploaded through a modal file upload |
| `ctx:findComponent(customId)` | A component in `ctx.message` by custom id |

## Constructors

Used internally; available for tests and custom transports.

| Function | Creates |
| --- | --- |
| `Context.fromInteraction(bot, interaction, transport?)` | An interaction context |
| `Context.fromMessage(bot, message)` | A prefix context |
| `Context.forEvent(bot, event, data)` | An event context |
| `Context.parseCustomId(customId)` | `name, params` |

`Context` is `require("@discord/lib/context")`.
