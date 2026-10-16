# Message content

`ctx:reply`, `ctx:send`, `ctx:edit`, `ctx:update`, `ctx:followUp`, `ctx:dm` and the `api` message functions all take the same **message input**. It can be any of these:

```luau
ctx:reply("plain text")                                   -- a string (or a number)
ctx:reply(ui.container({ ui.text("# Hi") }))              -- a single component
ctx:reply(ui.embed({ title = "Hi", description = "..." })) -- a single embed (title and description)
ctx:reply({ content = "text", embeds = { ... }, ... })    -- an options table
```

The full list of fields is in [Message input](../reference/message-input.md). This page goes through them by task.

## Text

```luau
ctx:reply({ content = "**Bold**, *italic*, `code`" })
```

`ctx.fmt` has [helpers](../reference/format.md) for markdown, mentions and timestamps:

```luau
local fmt = ctx.fmt
ctx:reply(table.concat({
    fmt.heading("Server stats", 2),
    `Owner: {fmt.user(guild.owner_id)}`,
    `Created: {fmt.timestamp(fmt.snowflakeTime(guild.id), "R")}`,
    fmt.subtext("updated just now"),
}, "\n"))
```

When you echo text a user typed, escape it so it shows literally:

```luau
ctx:reply(`You said: {fmt.escape(args.text)}`)
```

## Embeds

```luau
local ui = ctx.ui

ctx:reply({
    embeds = {
        ui.embed({
            title = "Server info",
            url = "https://example.com",
            description = "A description with **markdown**",
            color = "#5865F2",                       -- hex string, number, or { r, g, b }
            author = { name = "Someone", icon = avatarUrl, url = "https://..." },
            thumbnail = iconUrl,
            image = bannerUrl,
            fields = {
                { name = "Members", value = "42", inline = true },
                { "Channels", "7", true },          -- shorthand: { name, value, inline }
            },
            footer = { text = "Footer text", icon = iconUrl },
            timestamp = true,                       -- now; or unix seconds, or an ISO string
        }),
    },
})
```

`embed = ui.embed(...)` is shorthand for a single embed. A message can have up to 10 embeds.

!!! note
    A bare embed passed straight to `reply` is only recognized when it has both `title` and `description`. Otherwise pass it in `embeds = { ... }`.

## Components

```luau
ctx:reply({
    content = "Pick one",
    components = {
        ui.button({ label = "Yes", customId = "poll:yes", style = "success" }),
        ui.button({ label = "No", customId = "poll:no", style = "danger" }),
    },
})
```

Buttons and selects left at the top level are wrapped in action rows for you. Up to 5 buttons share a row, and a select gets a row of its own.

Messages built from **components v2** (containers, text displays, sections, galleries, separators, files) get the `IsComponentsV2` flag automatically. Such messages **can't have `content` or `embeds`**. Put the text in `ui.text(...)` instead. discord raises an error if you mix them:

```text
messages with components v2 can't have content or embeds; put the text in ui.text(...) instead
```

See [Components](components.md).

## Files

Attach files with `files` (or `file` for one). Each file has a `name` and either `data` (the contents) or `path` (a file on disk):

```luau
ctx:reply({
    content = "Here's the report",
    files = {
        { name = "report.txt", data = "line 1\nline 2" },
        { name = "chart.png", path = "out/chart.png", description = "A chart" },
        { name = "secret.png", data = bytes, spoiler = true },
    },
})
```

| File field | Description |
| --- | --- |
| `name` | The file name Discord shows |
| `data` | The contents, as a string |
| `path` | Or a path to read the contents from |
| `contentType` | MIME type (default `application/octet-stream`) |
| `description` | Alt text |
| `spoiler` | Blur it (prefixes the name with `SPOILER_`) |

Refer to an uploaded file from an embed or component with `attachment://<name>`:

```luau
ctx:reply({
    embeds = { ui.embed({ title = "Chart", description = "Today", image = "attachment://chart.png" }) },
    files = { { name = "chart.png", path = "out/chart.png" } },
})

ctx:reply({
    components = { ui.container({ ui.text("Your file:"), ui.file("attachment://report.pdf") }) },
    files = { { name = "report.pdf", path = "report.pdf" } },
})
```

When **editing**:

- Without new `files`, existing attachments are kept unless you pass `attachments`. `attachments = {}` removes them all.
- With new `files`, the message keeps only the new files plus whatever you list in `attachments`. To keep an old attachment, include it: `attachments = { { id = oldAttachment.id } }`.

## Mentions

By default, messages from the bot can ping **users** but not roles, `@everyone` or `@here`, and replies don't ping the replied-to user. This comes from the bot's `allowedMentions` option:

```luau
discord.new({ allowedMentions = { parse = { "users" }, repliedUser = false } }) -- the default
```

Override it per message:

```luau
ctx:reply({ content = "@everyone hi", allowedMentions = { parse = { "everyone" } } })
ctx:reply({ content = "<@123> <@&456>", allowedMentions = { users = { "123" } } }) -- only pings 123
ctx:reply({ content = `{ctx.fmt.user(id)} won`, allowedMentions = false })          -- pings nobody
```

| `allowedMentions` | Pings |
| --- | --- |
| `false` | Nobody |
| `{ parse = { "users", "roles", "everyone" } }` | Every mention of those kinds in the text |
| `{ users = { ids }, roles = { ids } }` | Only those ids |
| `{ repliedUser = true }` | The author of the message being replied to |

## Replying to a message

Prefix replies always reference the command message. For other messages, use `reply`:

```luau
ctx.api.channels.send(channelId, { content = "Answering this", reply = messageId })
ctx.api.channels.send(channelId, { content = "Answering this", reply = message })   -- a message object
ctx.api.channels.send(channelId, {
    content = "Answering this",
    reply = { messageId = id, channelId = otherChannel, failIfNotExists = true },
})
```

## Silent, TTS and other flags

```luau
ctx:reply({ content = "No notification", silent = true })   -- @silent
ctx:reply({ content = "https://example.com", suppressEmbeds = true })
ctx:send({ content = "Read aloud", tts = true })
ctx:reply({ content = "raw flags", flags = 4 })            -- combined with the flags above
```

## Polls

```luau
ctx:reply({
    poll = ctx.ui.poll({
        question = "Best fruit?",
        answers = { "Apple", "Banana", { text = "Cherry", emoji = "🍒" } },
        duration = 24,   -- hours (default 24)
        multi = false,   -- allow several answers
    }),
})
```

End a poll early with `ctx.api.channels.endPoll(channelId, messageId)`, and list the voters for an answer with `ctx.api.channels.pollVoters(channelId, messageId, answerId)`.

## Stickers

```luau
ctx:send({ stickers = { "749054660769218631" } })
```

## Using `messages.build` directly

The conversion is done by `discord.messages.build(input, defaults, forEdit)`. It returns the JSON body and the files. It's useful when you call `ctx.rest` yourself:

```luau
local body, files = discord.messages.build({ content = "hi", files = { ... } })
ctx.rest:request("POST", `/channels/{id}/messages`, { body = body, files = files })
```
