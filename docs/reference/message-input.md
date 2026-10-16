# Message input

What `ctx:reply`, `ctx:followUp`, `ctx:edit`, `ctx:update`, `ctx:send`, `ctx:dm`, `api.channels.send`, `api.channels.editMessage`, `api.users.dm`, `api.interactions.*` and `api.webhooks.execute` accept.

## Forms

| Input | Treated as |
| --- | --- |
| a string or number | `{ content = tostring(input) }` |
| a component (a table with a numeric `type`) | `{ components = { input } }` |
| an embed (a table with `title` and `description`, and no `content` / `components`) | `{ embeds = { input } }` |
| any other table | an options table (below) |

## Options table

| Field | Type | Description |
| --- | --- | --- |
| `content` | `string` | Text (up to 2000 characters) |
| `embeds` | `{ Embed }` | Up to 10, see [`ui.embed`](ui.md#uiembed) |
| `embed` | `Embed` | One embed (replaces `embeds`) |
| `components` | `{ Component }` | Loose buttons/selects are wrapped in rows; v2 components set `IsComponentsV2` |
| `files` | `{ File }` | Uploads (see below) |
| `file` | `File` | One upload (replaces `files`) |
| `ephemeral` | `boolean` | Interactions only; ignored for edits |
| `silent` | `boolean` | No push/desktop notification (`SuppressNotifications`) |
| `suppressEmbeds` | `boolean` | Hide link embeds |
| `tts` | `boolean` | Text-to-speech (not for edits) |
| `poll` | `Poll` | See [`ui.poll`](ui.md#uipoll) |
| `stickers` | `{ string }` | Sticker ids |
| `allowedMentions` | `AllowedMentions \| false` | Overrides the bot default; `false` pings nobody |
| `reply` | `string \| Message \| { messageId, channelId?, failIfNotExists? }` | Reply to a message (not for edits). Also `messageReference`. |
| `flags` | `number` | Raw flags, combined with the ones above |
| `v2` | `boolean` | Force `IsComponentsV2` |
| `attachments` | `{ table }` | Edits: attachments to keep (`{ id = ... }`) |
| `nonce`, `enforceNonce` | | Deduplication |

For `api.webhooks.execute`, also: `username`, `avatarUrl`, `threadId`, `threadName`.

Messages with `IsComponentsV2` can't have `content` or `embeds`: building one raises *messages with components v2 can't have content or embeds*.

## File

| Field | Type | Description |
| --- | --- | --- |
| `name` | `string` | **Required.** The file name; reference it as `attachment://name` |
| `data` | `string` | The contents |
| `path` | `string` | Or a path to read |
| `contentType` | `string` | Default `application/octet-stream` |
| `description` | `string` | Alt text |
| `spoiler` | `boolean` | Prefixes the name with `SPOILER_` |

One of `data` or `path` is required.

## AllowedMentions

| Field | Type | Description |
| --- | --- | --- |
| `parse` | `{ "users" \| "roles" \| "everyone" }` | Kinds of mention to ping |
| `users` | `{ string }` | Only these users (don't combine with `parse = { "users" }`) |
| `roles` | `{ string }` | Only these roles |
| `repliedUser` | `boolean` | Ping the author of the replied message |

Without `parse`, `users` and `roles`, nothing is pinged. The bot's default is `{ parse = { "users" }, repliedUser = false }`.

## `messages.build`

```luau
discord.messages.build(input: MessageInput, defaults: { ephemeral: boolean?, allowedMentions: any? }?, forEdit: boolean?)
    -> (body: table, files: { File }?)
```

Converts input to Discord's JSON body and the files to upload. `defaults` fills in `ephemeral` and `allowedMentions` when the input leaves them out. `forEdit` leaves out what edits can't change (`tts`, `nonce`, `reply`, the ephemeral flag). Raises on unsupported input and on v2 messages with content.

## `messages.allowedMentions`

```luau
discord.messages.allowedMentions(value: AllowedMentions | false | nil) -> table?
```

The `allowed_mentions` JSON for a value.
