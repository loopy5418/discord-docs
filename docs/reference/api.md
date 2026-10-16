# api

```luau
local api = ctx.api                -- or bot.api
local api = discord.makeApi(rest, bot?)
```

Endpoint wrappers grouped by resource. Every function returns `(result, nil)` or `(nil, err)`; see [Calling the API](../guide/api.md). `reason` parameters set the audit log reason. Message parameters accept any [message input](message-input.md).

`discord.makeApi(rest, bot?)` builds the object. With a `bot`, messages use its `allowedMentions` default and application-scoped functions use its `applicationId`; without one, those functions raise.

## `api.channels`

| Function | Endpoint |
| --- | --- |
| `get(channelId)` | `GET /channels/{id}` |
| `edit(channelId, fields, reason?)` | `PATCH /channels/{id}`: `name`, `topic`, `nsfw`, `rate_limit_per_user`, `parent_id`, `position`, `permission_overwrites`, ... |
| `delete(channelId, reason?)` | `DELETE /channels/{id}` |
| `send(channelId, message)` | Create a message |
| `message(channelId, messageId)` | Get a message |
| `messages(channelId, options?)` | List messages; `limit` (1–100), `before`, `after`, `around` |
| `editMessage(channelId, messageId, message)` | Edit a message |
| `deleteMessage(channelId, messageId, reason?)` | Delete a message |
| `bulkDelete(channelId, messageIds, reason?)` | Delete 2–100 messages younger than two weeks |
| `crosspost(channelId, messageId)` | Publish an announcement |
| `react(channelId, messageId, emoji)` | Add the bot's reaction |
| `unreact(channelId, messageId, emoji, userId?)` | Remove the bot's (or a user's) reaction |
| `reactions(channelId, messageId, emoji, options?)` | Users who reacted; `limit`, `after` |
| `clearReactions(channelId, messageId, emoji?)` | Remove all reactions, or all of one emoji |
| `typing(channelId)` | Typing indicator |
| `pins(channelId, options?)` | Pinned messages |
| `pin(channelId, messageId, reason?)` / `unpin(...)` | Pin / unpin |
| `setPermission(channelId, targetId, kind, allow, deny, reason?)` | Overwrite for a `"role"` or `"member"`; `allow`/`deny` are names, lists or masks |
| `deletePermission(channelId, targetId, reason?)` | Remove an overwrite |
| `invites(channelId)` | List invites |
| `createInvite(channelId, options?, reason?)` | `max_age`, `max_uses`, `temporary`, `unique` |
| `webhooks(channelId)` | List webhooks |
| `createWebhook(channelId, name, avatar?, reason?)` | `avatar` is a data URI |
| `threadFromMessage(channelId, messageId, name, options?, reason?)` | `auto_archive_duration`, `rate_limit_per_user` |
| `createThread(channelId, name, options?, reason?)` | `type` (11 public, default; 12 private), `invitable`, `auto_archive_duration`; in forum/media channels pass `message` (the first post, any message input) |
| `joinThread(threadId)` / `leaveThread(threadId)` | |
| `addThreadMember(threadId, userId)` / `removeThreadMember(threadId, userId)` | |
| `archivedThreads(channelId, private?, options?)` | Public or private archived threads |
| `endPoll(channelId, messageId)` | End a poll now |
| `pollVoters(channelId, messageId, answerId, options?)` | Users who voted for an answer |

Emoji arguments accept `"👍"`, `"<:name:id>"`, `"name:id"` or an emoji table.

## `api.guilds`

| Function | Endpoint |
| --- | --- |
| `get(guildId, withCounts?)` | The guild (`approximate_member_count` etc. with `withCounts`) |
| `edit(guildId, fields, reason?)` | Modify the guild |
| `leave(guildId)` | The bot leaves |
| `channels(guildId)` | List channels |
| `createChannel(guildId, fields, reason?)` | `name` (required), `type`, `topic`, `parent_id`, `permission_overwrites`, ... |
| `activeThreads(guildId)` | Active threads |
| `member(guildId, userId)` | One member |
| `members(guildId, options?)` | List members (needs `GuildMembers`); `limit` (1–1000), `after` |
| `searchMembers(guildId, query, limit?)` | Members whose name starts with `query` (default limit 10) |
| `editMember(guildId, userId, fields, reason?)` | `nick`, `roles`, `mute`, `deaf`, `channel_id`, `communication_disabled_until`, `flags` |
| `setNickname(guildId, nick?, reason?)` | The bot's nickname (`nil` resets) |
| `timeout(guildId, userId, seconds?, reason?)` | Time out for `seconds` (max 28 days); `0`/`nil` removes |
| `kick(guildId, userId, reason?)` | |
| `ban(guildId, userId, options?)` | `options`: `deleteMessageSeconds` (0–604800), `reason` |
| `unban(guildId, userId, reason?)` | |
| `bans(guildId, options?)` | List bans |
| `getBan(guildId, userId)` | One ban |
| `roles(guildId)` | List roles |
| `createRole(guildId, fields?, reason?)` | `name`, `permissions` (names or mask), `color` (any [color](ui.md#helpers)), `hoist`, `mentionable`, `icon`, `unicode_emoji` |
| `editRole(guildId, roleId, fields, reason?)` | Same fields |
| `deleteRole(guildId, roleId, reason?)` | |
| `addRole(guildId, userId, roleId, reason?)` / `removeRole(...)` | |
| `auditLog(guildId, options?)` | `user_id`, `action_type`, `before`, `after`, `limit` |
| `invites(guildId)` | |
| `webhooks(guildId)` | |
| `emojis(guildId)` | |
| `createEmoji(guildId, name, image, roles?, reason?)` | `image` is a data URI |
| `deleteEmoji(guildId, emojiId, reason?)` | |
| `scheduledEvents(guildId)` | With user counts |
| `autoModRules(guildId)` | |

## `api.users`

| Function | Endpoint |
| --- | --- |
| `get(userId)` | A user |
| `me()` | The bot user |
| `editMe(fields)` | `username`, `avatar` (data URI), `banner` |
| `guilds(options?)` | Servers the bot is in |
| `dmChannel(userId)` | The DM channel (cached per user) |
| `dm(userId, message)` | Open a DM and send a message |

## `api.commands`

Application commands. `guildId` selects a guild's commands; omit it for global ones. Need the application id.

| Function | Description |
| --- | --- |
| `list(guildId?)` | With localizations |
| `get(commandId, guildId?)` | |
| `create(command, guildId?)` | Raw command JSON |
| `edit(commandId, command, guildId?)` | |
| `delete(commandId, guildId?)` | |
| `set(list, guildId?)` | Bulk overwrite |
| `permissions(guildId, commandId?)` | Per-guild permission overrides |

## `api.interactions`

Used by the context; useful with a stored interaction token.

| Function | Description |
| --- | --- |
| `callback(id, token, body, files?, withResponse?)` | The initial response (raw callback body) |
| `getOriginal(token)` | The original response |
| `editOriginal(token, message)` | |
| `deleteOriginal(token)` | |
| `followUp(token, message, ephemeral?)` | |
| `editFollowUp(token, messageId, message)` | |
| `deleteFollowUp(token, messageId)` | |

## `api.webhooks`

| Function | Description |
| --- | --- |
| `get(webhookId, token?)` | With a token, no bot auth is needed |
| `edit(webhookId, fields, reason?)` | |
| `delete(webhookId, reason?)` | |
| `execute(webhookId, token, message, wait?)` | Message input plus `username`, `avatarUrl`, `threadId`, `threadName`; returns the message unless `wait == false` |
| `parseUrl(url)` | `id, token` from a webhook URL |

## `api.application`

| Function | Description |
| --- | --- |
| `me()` | The application |
| `edit(fields)` | `description`, `interactions_endpoint_url`, `install_params`, `integration_types_config`, ... |
| `emojis()` | Application emojis (usable by the bot everywhere) |
| `createEmoji(name, image)` / `deleteEmoji(emojiId)` | |
| `entitlements(options?)` | `user_id`, `guild_id`, `sku_ids`, `exclude_ended` |
| `consumeEntitlement(entitlementId)` | |
| `skus()` | |

## Other

| Function | Description |
| --- | --- |
| `api.stickers.get(stickerId)` | |
| `api.stickers.guild(guildId)` | A guild's stickers |
| `api.invites.get(code)` | With counts |
| `api.invites.delete(code, reason?)` | |
| `api.gateway()` | `GET /gateway/bot`: url, recommended shards, session start limits |
| `api.request(method, route, options?)` | Any endpoint; see [`rest:request`](rest.md#restrequest) |
