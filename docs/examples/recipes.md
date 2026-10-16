# Recipes

Short, complete solutions to common tasks. Each assumes `local discord = require("@discord")`, `local ui = discord.ui` and a `bot`.

## Welcome messages

Needs the privileged **Server Members** intent.

```luau
local bot = discord.new({
    intents = { "Guilds", "GuildMembers" },
})

local WELCOME_CHANNELS = { ["111111111111111111"] = "222222222222222222" } -- guild -> channel

bot:on("GUILD_MEMBER_ADD", function(ctx, member)
    local channelId = WELCOME_CHANNELS[member.guild_id]
    if channelId == nil then
        return
    end
    local user = member.user
    ctx.api.channels.send(channelId, {
        components = {
            ui.container({
                accent = "#57F287",
                ui.section({
                    `## Welcome, {ctx.fmt.displayName(user)}!`,
                    `{ctx.fmt.user(user.id)} just joined. Say hi!`,
                    accessory = ui.thumbnail(ctx.fmt.avatar(user, 256)),
                }),
            }),
        },
    })
end)
```

## Timeouts with readable durations

```luau
bot:command({
    name = "timeout",
    description = "Time out a member",
    guildOnly = true,
    permissions = { "ModerateMembers" },
    botPermissions = { "ModerateMembers" },
    prefix = true,
    options = {
        { name = "member", type = "member", description = "Who", required = true },
        { name = "duration", type = "string", description = "e.g. 10m, 1h30m, 2d", required = true, rest = false },
        { name = "reason", type = "string", description = "Why" },
    },
    run = function(ctx, args)
        local fmt = ctx.fmt
        local seconds = fmt.parseDuration(args.duration)
        if seconds == nil or seconds <= 0 or seconds > 28 * 86400 then
            return ctx:error("Use a duration between 1s and 28d, like `10m` or `1h30m`.")
        end
        local ok, err = ctx.api.guilds.timeout(ctx.guildId, args.member.user.id, seconds,
            `{ctx.user.username}: {args.reason or "no reason"}`)
        if not ok then
            return ctx:error(`Couldn't time out: {err.message or tostring(err)}`)
        end
        ctx:reply(`🔇 {fmt.user(args.member.user.id)} is timed out for {fmt.duration(seconds)} `
            .. `(until {fmt.timestamp(os.time() + seconds, "f")}).`)
    end,
})
```

`rest = false` on `duration` keeps it to one word for `!timeout @x 10m being loud`; `reason` (the last string option) takes the rest.

## Button roles that survive restarts

Everything the handler needs is in the custom id, so nothing is kept in memory.

```luau
bot:command({
    name = "rolebutton",
    description = "Post a button that toggles a role",
    guildOnly = true,
    permissions = { "ManageRoles" },
    botPermissions = { "ManageRoles" },
    options = {
        { name = "role", type = "role", description = "The role", required = true },
        { name = "label", type = "string", description = "Button text" },
    },
    run = function(ctx, args)
        ctx:send({
            components = {
                ui.button({ label = args.label or args.role.name, customId = `togglerole:{args.role.id}` }),
            },
        })
        ctx:reply({ content = "Posted.", ephemeral = true })
    end,
})

bot:component({
    customId = "togglerole",
    defer = "ephemeral",
    botPermissions = { "ManageRoles" },
    run = function(ctx, roleId)
        local has = table.find(ctx.member.roles, roleId) ~= nil
        local ok, err
        if has then
            ok, err = ctx.api.guilds.removeRole(ctx.guildId, ctx.user.id, roleId, "role button")
        else
            ok, err = ctx.api.guilds.addRole(ctx.guildId, ctx.user.id, roleId, "role button")
        end
        if not ok then
            return ctx:reply(`Couldn't change your roles: {err.message or tostring(err)}`)
        end
        ctx:reply(if has then `Removed {ctx.fmt.role(roleId)}.` else `Added {ctx.fmt.role(roleId)}.`)
    end,
})
```

## Support tickets: button → form → private thread

```luau
bot:command({
    name = "ticketpanel",
    description = "Post the ticket panel",
    guildOnly = true,
    permissions = { "ManageGuild" },
    run = function(ctx)
        ctx:send({
            components = {
                ui.container({
                    ui.text("## Need help?\nOpen a ticket and staff will get back to you."),
                    ui.button({ label = "Open ticket", emoji = "🎫", customId = "ticket:open" }),
                }),
            },
        })
        ctx:reply({ content = "Panel posted.", ephemeral = true })
    end,
})

-- the button opens a form (a modal must be the first response, so no defer here)
bot:component({
    customId = "ticket",
    run = function(ctx, action)
        if action ~= "open" then
            return
        end
        ctx:showModal(ui.modal({
            title = "Open a ticket",
            customId = "ticket-form",
            ui.textInput({ customId = "subject", label = "Subject", maxLength = 80 }),
            ui.textInput({ customId = "details", label = "Details", style = "paragraph", maxLength = 1500 }),
        }))
    end,
})

-- the form creates a private thread and adds the user
bot:modal({
    customId = "ticket-form",
    defer = "ephemeral",
    botPermissions = { "CreatePrivateThreads", "SendMessagesInThreads" },
    run = function(ctx, fields)
        local thread, err = ctx.api.channels.createThread(ctx.channelId, `ticket-{ctx.user.username}`, {
            type = 12,              -- private thread
            invitable = false,
            auto_archive_duration = 10080,
        }, `ticket by {ctx.user.username}`)
        if thread == nil then
            return ctx:reply(`Couldn't open a ticket: {err.message or tostring(err)}`)
        end
        ctx.api.channels.addThreadMember(thread.id, ctx.user.id)
        ctx.api.channels.send(thread.id, {
            content = `**{fields.subject}**\n{fields.details}\n\n{ctx:mention()}, staff will be with you soon.`,
            allowedMentions = { users = { ctx.user.id } },
        })
        ctx:reply(`Your ticket: {ctx.fmt.channel(thread.id)}`)
    end,
})
```

## A help command

Built from `bot:helpEntries()`, one page per category:

```luau
bot:command({
    name = "help",
    description = "List the commands",
    prefix = true,
    run = function(ctx)
        local byCategory, order = {}, {}
        for _, entry in ctx.bot:helpEntries(ctx.prefix or "!") do
            if byCategory[entry.category] == nil then
                byCategory[entry.category] = {}
                table.insert(order, entry.category)
            end
            local name = if entry.kind == "slash" then ctx:commandMention(entry.name) else `\`{entry.usage}\``
            table.insert(byCategory[entry.category], `{name} — {entry.description}`)
        end
        table.sort(order)

        local pages = {}
        for _, category in order do
            table.insert(pages, ui.container({
                accent = "#5865F2",
                ui.text(`## {category}`),
                ui.text(table.concat(byCategory[category], "\n")),
            }))
        end
        ctx:paginate(pages, { timeout = 90 })
    end,
})
```

Give commands a `category` to group them, and `hidden = true` to leave them out.

## Reminders

```luau
local task = require("@sonata/task")

bot:command({
    name = "remind",
    description = "Remind you of something",
    prefix = true,
    options = {
        { name = "in", type = "string", description = "When, e.g. 10m", required = true, rest = false },
        { name = "what", type = "string", description = "What to remind you of", required = true },
    },
    run = function(ctx, args)
        local seconds = ctx.fmt.parseDuration(args["in"])
        if seconds == nil or seconds <= 0 then
            return ctx:error("Use a duration like `10m` or `2h`.")
        end
        local channelId, userId = ctx.channelId, ctx.user.id
        task.delay(seconds, function()
            local ok = ctx.api.users.dm(userId, `⏰ Reminder: {args.what}`)
            if not ok then
                ctx.api.channels.send(channelId, {
                    content = `⏰ {ctx.fmt.user(userId)}: {args.what}`,
                    allowedMentions = { users = { userId } },
                })
            end
        end)
        ctx:reply({ content = `Okay, {ctx.fmt.timestamp(os.time() + seconds, "R")}.`, ephemeral = true })
    end,
})
```

Reminders live in memory and are lost on restart; save them to a file (next recipe) to keep them.

## Saving state to a file

`ctx.store` is in memory. A small helper can persist part of it as JSON:

```luau title="_storage.luau"
local fs = require("@sonata/fs")
local json = require("@sonata/json")

local FILE = "data.json"
local storage = {}

function storage.load(): { [string]: any }
    local ok, data = pcall(function()
        return json.decode(fs.readFile(FILE))
    end)
    return if ok and type(data) == "table" then data else {}
end

function storage.save(data: { [string]: any })
    fs.writeFile(FILE, json.encode(data, { indent = 2 }))
end

return storage
```

```luau title="main.luau"
local storage = require("./_storage")

local bot = discord.new({
    prefix = function(message, bot)
        local prefixes = bot.store.data.prefixes
        return message.guild_id and prefixes[message.guild_id] or "!"
    end,
})
bot.store.data = storage.load()
bot.store.data.prefixes = bot.store.data.prefixes or {}

bot:command({
    name = "setprefix",
    description = "Change this server's prefix",
    prefix = true,
    guildOnly = true,
    permissions = { "ManageGuild" },
    options = { { name = "prefix", type = "string", required = true, maxLength = 5, rest = false } },
    run = function(ctx, args)
        ctx.store.data.prefixes[ctx.guildId] = args.prefix
        storage.save(ctx.store.data)
        ctx:reply(`Prefix set to \`{args.prefix}\`.`)
    end,
})
```

For anything bigger, use a real database.

## Logging commands to a channel

```luau
local LOG_CHANNEL = "333333333333333333"

bot:use(function(ctx, next)
    next()
    if ctx.kind == "slash" or ctx.kind == "prefix" then
        ctx.api.channels.send(LOG_CHANNEL, {
            content = `{ctx.fmt.user(ctx.user.id)} ran \`{ctx.commandPath}\` in {if ctx.guildId then ctx.fmt.channel(ctx.channelId) else "DMs"}`,
            allowedMentions = false,
        })
    end
end)

bot:onError(function(err, ctx)
    print(err)
    bot.api.channels.send(LOG_CHANNEL, {
        content = `⚠️ error in \`{if ctx then ctx.commandPath or ctx.kind else "?"}\``,
        files = { { name = "traceback.txt", data = err } },
    })
end)
```

## An embed with a generated file

```luau
bot:command({
    name = "export",
    description = "Export the server's roles as CSV",
    guildOnly = true,
    defer = true,
    run = function(ctx)
        local guild = ctx:fetchGuild()
        local lines = { "id,name,position" }
        for _, role in guild.roles do
            table.insert(lines, `{role.id},"{role.name}",{role.position}`)
        end
        ctx:reply({
            embeds = { ui.embed({ title = "Roles", description = `{#guild.roles} roles`, color = "#5865F2" }) },
            files = { { name = "roles.csv", data = table.concat(lines, "\n"), contentType = "text/csv" } },
        })
    end,
})
```

## A deploy notifier (no bot)

A script that posts to a channel with only the REST API:

```luau title="notify.luau"
local process = require("@sonata/process")
local discord = require("@discord")

local api = discord.makeApi(discord.Rest.new(process.env.DISCORD_TOKEN))
local ok, err = api.channels.send(process.env.CHANNEL_ID, {
    embeds = { discord.ui.embed({
        title = "Deployed",
        description = args[1] or "a new version",
        color = "#57F287",
        timestamp = true,
    }) },
})
if not ok then
    print("failed:", err)
    process.exit(1)
end
```

```sh
sn run --entry notify.luau -- "v1.4.2"
```

Or, with a webhook URL and no bot token at all:

```luau
local api = discord.makeApi(discord.Rest.new("unused"))
local id, token = api.webhooks.parseUrl(process.env.WEBHOOK_URL)
api.webhooks.execute(id, token, { content = "Deployed!", username = "CI" })
```

## Asking for confirmation in a prefix command

`ctx:confirm` and `ctx:paginate` work for prefix commands too; the buttons are added to a normal reply:

```luau
bot:prefix({
    name = "nuke",
    description = "Delete the last 100 messages",
    permissions = { "ManageMessages" },
    botPermissions = { "ManageMessages" },
    run = function(ctx)
        if not ctx:confirm("Delete the last 100 messages here?", { timeout = 20 }) then
            return ctx:reply("Cancelled.")
        end
        local list = ctx.api.channels.messages(ctx.channelId, { limit = 100 }) or {}
        local ids = {}
        for _, m in list do
            table.insert(ids, m.id)
        end
        if #ids >= 2 then
            ctx.api.channels.bulkDelete(ctx.channelId, ids, `nuke by {ctx.user.username}`)
        end
    end,
})
```
