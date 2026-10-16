# The context

Every handler receives a **context**, `ctx`, as its first argument. This includes commands, components, modals, autocomplete and events. The context says what happened, who did it and where, and has methods for answering.

```luau
run = function(ctx, args)
    ctx:reply(`Hi {ctx.user.username}, you ran /{ctx.commandName} in {ctx.guildId or "a DM"}`)
end
```

Contexts for interactions and for messages share one interface. `ctx:reply` answers a slash command and a `!command` alike, so a single `run` function serves [hybrid commands](prefix-commands.md#hybrid-commands).

## Kinds

`ctx.kind` says what triggered the handler:

| `ctx.kind` | Triggered by | Handler |
| --- | --- | --- |
| `"slash"` | A slash command | `run(ctx, options)` |
| `"user"` | A user context menu command | `run(ctx, targetUser)` |
| `"message"` | A message context menu command | `run(ctx, targetMessage)` |
| `"prefix"` | A prefix command message | `run(ctx, args)` |
| `"component"` | A button or select | `run(ctx, ...params)` |
| `"modal"` | A modal submit | `run(ctx, fields)` |
| `"autocomplete"` | Typing in an autocomplete option | `autocomplete(ctx, value, optionName)` |
| `"event"` | A gateway event | `run(ctx, data)` |

`ctx.isInteraction` is `true` for every kind that comes from an interaction (everything except `prefix` and `event`). `ctx.isPrefix` is `true` for prefix commands.

## Who and where

| Field | Description |
| --- | --- |
| `ctx.user` / `ctx.author` | The user who triggered this |
| `ctx.member` | Their guild member, in servers. On interactions it includes `permissions`. |
| `ctx.guildId` | The server id, or `nil` in DMs |
| `ctx.channelId` | The channel id |
| `ctx.channel` | The partial channel Discord sends with interactions |
| `ctx.inGuild` / `ctx.inDM` | Whether this happened in a server or not |
| `ctx.where` | `"guild"`, `"botDm"` or `"privateChannel"` (group DMs and other users' DMs) |
| `ctx.locale` / `ctx.guildLocale` | The user's language and the server's (interactions) |
| `ctx.isUserInstall` / `ctx.isGuildInstall` | How the app was installed for this interaction (see [User-installed apps](user-installs.md)) |
| `ctx.botPresent` | Whether the bot itself is in this server or DM and can make normal API calls there |

```luau
run = function(ctx)
    if ctx.inDM then
        return ctx:reply("This is a DM.")
    end
    ctx:reply(`Server {ctx.guildId}, channel {ctx.channelId}`)
end
```

## What was sent

The fields set depend on the kind:

=== "Slash commands"

    | Field | Description |
    | --- | --- |
    | `ctx.options` / `ctx.args` | Option values by name (also the second argument of `run`) |
    | `ctx.commandName` | The top-level command name |
    | `ctx.commandPath` | The full path, e.g. `"config prefix set"` |
    | `ctx.group` / `ctx.subcommand` | The subcommand group and subcommand names |
    | `ctx.command` | The command's normalized definition |

=== "Prefix commands"

    | Field | Description |
    | --- | --- |
    | `ctx.message` | The message that ran the command |
    | `ctx.content` | Its text |
    | `ctx.prefix` | The prefix that matched (`"!"`, or `"<@id>"` for a mention) |
    | `ctx.invokedWith` | The name or alias the user typed |
    | `ctx.options` / `ctx.args` | Parsed arguments by option name |
    | `ctx.rawArgs` | The words after the command, quotes removed |
    | `ctx.argText` | Everything after the command name, as typed |

=== "Context menus"

    | Field | Description |
    | --- | --- |
    | `ctx.target` | The user or message that was right-clicked |
    | `ctx.targetMember` | The user's member object (user commands in servers) |
    | `ctx.targetId` | The target's id |

=== "Components"

    | Field | Description |
    | --- | --- |
    | `ctx.customId` | The full custom id, e.g. `"vote:yes"` |
    | `ctx.name` | The part before the first `:` (`"vote"`) |
    | `ctx.params` | The parts after it (`{ "yes" }`) |
    | `ctx.values` | Selected values (selects) |
    | `ctx.componentType` | The component type number |
    | `ctx.message` | The message the component is on |
    | `ctx.resolved` | Users, roles and channels picked in a select |

=== "Modals"

    | Field | Description |
    | --- | --- |
    | `ctx.customId`, `ctx.name`, `ctx.params` | As for components |
    | `ctx.fields` | Submitted values by input custom id |
    | `ctx.message` | The message, when the modal was opened from a component |

=== "Autocomplete"

    | Field | Description |
    | --- | --- |
    | `ctx.focused` | `{ name, value, type }` of the option being typed in |
    | `ctx.options` | The other options typed so far |

=== "Events"

    | Field | Description |
    | --- | --- |
    | `ctx.event` | The event name, e.g. `"MESSAGE_CREATE"` |
    | `ctx.eventData` | The event payload (also the second argument of `run`) |
    | `ctx.message`, `ctx.content` | For `MESSAGE_CREATE` and `MESSAGE_UPDATE` |
    | `ctx.user`, `ctx.member` | When the payload has them |

Handy accessors:

```luau
ctx:getOption("amount", 10)  -- an option, or 10 if it was left out
ctx:field("title", "")       -- a modal field, or ""
ctx:is("vote")               -- true if the custom id, its name, or the command name is "vote"
```

## Responding

The [Responding](responding.md) page covers these in detail:

| Method | Use |
| --- | --- |
| `ctx:reply(msg)` | Answer. Later calls send follow-ups. |
| `ctx:defer(opts)` | Acknowledge now and answer later (up to 15 minutes) |
| `ctx:followUp(msg)` | Another message after the first response |
| `ctx:edit(msg)` | Edit the response |
| `ctx:update(msg)` | Edit the message a component is on |
| `ctx:deferUpdate()` | Acknowledge a component without changing anything yet |
| `ctx:deleteReply()` | Delete the response |
| `ctx:fetchReply()` | Get the response message |
| `ctx:send(msg)` | A plain message in the channel |
| `ctx:error(text)` | An ephemeral `❌ text` |
| `ctx:showModal(modal)` | Open a modal |
| `ctx:prompt(modal)` | Open a modal and wait for it to be submitted |
| `ctx:respond(choices)` | Autocomplete suggestions |

And for messages:

| Method | Use |
| --- | --- |
| `ctx:react(emoji)` | React to the command message, the component's message or the targeted message |
| `ctx:typing()` | Show "Bot is typing..." |
| `ctx:dm(msg, userId?)` | DM the user (or someone else) |

## Waiting

These are covered in [Waiting for input](waiting.md):

| Method | Waits for |
| --- | --- |
| `ctx:awaitComponent(opts)` | One button or select press |
| `ctx:collect(opts, fn)` | Every press, until a timeout |
| `ctx:awaitMessage(opts)` | A message from the user |
| `ctx:waitFor(event, filter, timeout)` | Any gateway event |
| `ctx:confirm(question)` | A Confirm/Cancel answer |
| `ctx:paginate(pages)` | Pages with previous/next buttons |

## Permissions

```luau
local ok, missing = ctx:hasPermission({ "BanMembers", "KickMembers" })
if not ok then
    return ctx:error("You're missing: " .. table.concat(missing, ", "))
end

if not ctx:botHasPermission("ManageRoles") then ... end
if ctx:isOwner() then ... end

local mask = ctx:memberPermissions()   -- the caller's permissions in this channel
local mine = ctx:botPermissions()      -- the bot's
```

On interactions, Discord sends both sets of permissions, already computed. For prefix commands they are computed from the cached server, roles and channel overwrites. In DMs every permission counts as granted. See [Checks, permissions and cooldowns](checks.md).

## Lookups

These check the [cache](cache.md) first and fall back to the API:

```luau
local guild = ctx:fetchGuild()                -- this server
local channel = ctx:fetchChannel()            -- this channel, or ctx:fetchChannel(id)
local member = ctx:fetchMember()              -- the caller's member, or ctx:fetchMember(userId)
local user = ctx:fetchUser(id)                -- checks the message's mentions first
local role = ctx:fetchRole("Moderator")       -- by id or name (case-insensitive)
```

## Libraries on the context

The context carries the bot's libraries, so command files don't need to require them:

| Field | Module |
| --- | --- |
| `ctx.bot` | The [Bot](../reference/bot.md) |
| `ctx.api` | [Endpoint wrappers](../reference/api.md) |
| `ctx.rest` | The [REST client](../reference/rest.md) |
| `ctx.ui` | [Component builders](../reference/ui.md) |
| `ctx.fmt` | [Formatting helpers](../reference/format.md) |
| `ctx.enums` | [Constants](../reference/enums.md) |
| `ctx.permissions` | [Permission helpers](../reference/permissions.md) |
| `ctx.cache` | The [cache](../reference/cache.md) |
| `ctx.store` | Shared state for the bot's lifetime |
| `ctx.state` | Scratch space for this one invocation |

## Small helpers

```luau
ctx:mention()                   -- "<@id>" of the caller
ctx:displayName()               -- nickname > global name > username
ctx:displayName(user, member)   -- the same for someone else
ctx:latency()                   -- gateway heartbeat round trip in seconds, or nil
ctx:commandMention("config set") -- a clickable </config set:id>, or "/config set"
ctx:sleep(2)                    -- task.wait
ctx:spawn(fn, ...)              -- task.spawn
ctx:log("something", 42)        -- prints "[commandPath] something 42"
ctx:selected()                  -- select values resolved to user/role/channel objects
ctx:uploads("files")            -- attachments from a modal file upload
ctx:findComponent("vote:yes")   -- a component on the message, by custom id
```

The [Context reference](../reference/context.md) lists every field and method with its exact behaviour.
