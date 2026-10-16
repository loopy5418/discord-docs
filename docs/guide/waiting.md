# Waiting for input

Handlers run in their own Sonata task, so a handler can stop and wait for the user without blocking the rest of the bot. This makes multi-step flows read top to bottom:

```luau
run = function(ctx)
    if not ctx:confirm("Delete all your data?") then
        return ctx:followUp({ content = "Cancelled.", ephemeral = true })
    end
    deleteData(ctx.user.id)
    ctx:followUp("Deleted.")
end
```

Every wait has a timeout and returns `nil` once it runs out. Interaction tokens last 15 minutes, so keep waits within that if you want to answer afterwards.

## `ctx:confirm`

Asks a yes/no question with two buttons, and returns `true`, `false`, or `nil` if nobody answered in time. The buttons are disabled once it returns.

```luau
local ok = ctx:confirm(`Ban **{user.username}**?`, {
    yes = "Ban",           -- button labels (default "Confirm" / "Cancel")
    no = "Keep",
    timeout = 30,          -- seconds (default 60)
    ephemeral = true,
})

if ok == nil then
    return ctx:followUp({ content = "Timed out.", ephemeral = true })
elseif not ok then
    return ctx:followUp({ content = "Cancelled.", ephemeral = true })
end
```

The question can be a string or a full message (`{ content = ..., embeds = ... }`); the buttons are added to its components. Only the user who ran the command can answer, unless you pass `user = false` (anyone) or `user = otherId`.

`confirm` sends the question with `ctx:reply`, so after it returns, send further messages with `ctx:followUp` (or `ctx:reply`, which follows up automatically).

## `ctx:paginate`

Shows a list of pages with ◀️ / ▶️ buttons and a page counter:

```luau
local pages = {}
for i = 1, #items, 10 do
    local slice = table.move(items, i, math.min(i + 9, #items), 1, {})
    table.insert(pages, { embeds = { ctx.ui.embed({ title = "Items", description = table.concat(slice, "\n") }) } })
end

ctx:paginate(pages, { timeout = 120 })
```

A page can be anything `ctx:reply` accepts:

- a string
- a message table (`{ content, embeds, components }`), with the navigation row appended to its components
- a single embed (a table with a `description`)
- a single component, such as a `ui.container(...)`

| Option | Default | Description |
| --- | --- | --- |
| `timeout` | `120` | Seconds without a click before the buttons are disabled |
| `start` | `1` | The first page shown |
| `wrap` | `true` | Going past the last page returns to the first, and the other way round |
| `ephemeral` | the command's default | Send the pages ephemerally |
| `user` | the caller | Who may turn pages (`false` for anyone) |

`paginate` returns only once the menu times out. If there is nothing more to do, call it last.

## `ctx:awaitComponent`

Waits for one click or select on a message and returns its context, or `nil` on timeout:

```luau
ctx:reply({
    content = "Pick a side",
    components = {
        ui.button({ label = "Red", customId = "side:red", style = "danger" }),
        ui.button({ label = "Blue", customId = "side:blue" }),
    },
})

local click = ctx:awaitComponent({ timeout = 30 })
if click == nil then
    return ctx:edit({ content = "Too slow!", components = {} })
end
click:update({ content = `You chose {click.params[1]}`, components = {} })
```

The returned context is a component interaction, so answer it with `update`, `reply` or `deferUpdate`.

| Option | Description |
| --- | --- |
| `customId` | A custom id or a list of them to accept |
| `message` | A message (or its id) the component must be on |
| `user` | A user id; `false` accepts anyone. Default: the user who ran the command. |
| `users` | A list of user ids |
| `anyone` | `true` accepts anyone |
| `filter` | `function(componentCtx) -> boolean` |
| `timeout` | Seconds (default 60) |

Without `customId` or `message`, it accepts components on the **bot's reply** to this context (`ctx.replyMessage`), or anywhere in the channel if there is no reply yet.

Collectors are checked **before** registered handlers, so a waiting `awaitComponent` takes the click even when a handler with the same custom id exists.

## `ctx:collect`

Calls a function for every matching click until the collector stops. It doesn't block: it returns a handle immediately.

```luau
local votes = { yes = 0, no = 0 }
ctx:reply({
    content = "Vote!",
    components = {
        ui.button({ label = "Yes", customId = "vote:yes", style = "success" }),
        ui.button({ label = "No", customId = "vote:no", style = "danger" }),
    },
})

local collector = ctx:collect({
    anyone = true,
    timeout = 60,       -- stop after 60 idle seconds (default 120)
    max = 100,          -- or after 100 clicks
    onEnd = function(reason) -- "timeout", "max" or "stopped"
        ctx:edit({ content = `Yes: {votes.yes}, No: {votes.no}`, components = {} })
    end,
}, function(click)
    votes[click.params[1]] += 1
    click:reply({ content = "Counted!", ephemeral = true })
end)

-- later: collector.stop(), collector.count()
```

`collect` takes the same options as `awaitComponent`, plus `max` and `onEnd`. The timeout is an **idle** timeout: each click restarts it.

## `ctx:awaitMessage`

Waits for a message from the user in the same channel:

```luau
ctx:reply("What's your favourite colour? (30 seconds)")
local message = ctx:awaitMessage({ timeout = 30 })
if message == nil then
    return ctx:followUp("No answer.")
end
ctx:followUp(`{message.content} is a nice colour.`)
```

| Option | Default | Description |
| --- | --- | --- |
| `timeout` | `60` | Seconds |
| `channelId` | this channel | Where to listen |
| `user` | the caller | A user id, or `false` for anyone |
| `filter` | | `function(message) -> boolean` |

This needs the `GuildMessages` (or `DirectMessages`) intent, and `MessageContent` to read the text in servers. See [Intents](intents.md).

## `ctx:waitFor` / `bot:waitFor`

Waits for any gateway event whose payload passes a filter:

```luau
local message = ctx:reply("React with ✅ to continue")
local reaction = ctx:waitFor("MESSAGE_REACTION_ADD", function(d)
    return d.message_id == message.id and d.user_id == ctx.user.id and d.emoji.name == "✅"
end, 30)
```

It returns the event data, or `nil` after `timeout` seconds (default 60). Waiters see an event before regular event handlers run.

## `ctx:prompt`

Opens a modal and waits for it to be submitted. See [Modals](modals.md#opening-a-modal-and-waiting-for-it).

## Defaults at a glance

| Method | Default timeout | Returns |
| --- | --- | --- |
| `confirm` | 60 s | `true` / `false` / `nil` |
| `paginate` | 120 s idle | nothing (returns when done) |
| `awaitComponent` | 60 s | component context / `nil` |
| `collect` | 120 s idle | `{ stop, count }` immediately |
| `awaitMessage` | 60 s | message / `nil` |
| `waitFor` | 60 s | event data / `nil` |
| `prompt` | 300 s | modal context / `nil` |
