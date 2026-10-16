# Component handlers

When someone clicks a button or picks from a select menu, Discord sends an interaction that carries the component's **custom id**. discord routes it to a handler. There are three ways to provide one:

| Way | Best for |
| --- | --- |
| [Registered handler](#registered-handlers) (`bot:component`) | Components that must keep working across restarts, such as role menus or ticket buttons |
| [Inline handler](#inline-handlers) (`run = function`) | Short-lived UIs tied to one command run, such as counters, wizards or "show more" buttons |
| [Waiting](waiting.md) (`ctx:awaitComponent`, `ctx:collect`) | Linear flows inside a command, such as "click to confirm", then continue |

## Registered handlers

A registered handler matches custom ids by **name**. The name is the part before the first `:`. The rest is split on `:` into **parameters**:

```luau
-- the button
ui.button({ label = "Close ticket", customId = "ticket:close:" .. ticketId })

-- the handler: handles every custom id whose name is "ticket"
bot:component({
    customId = "ticket",
    run = function(ctx, action, ticketId)
        -- for "ticket:close:42": action = "close", ticketId = "42"
        if action == "close" then
            closeTicket(ticketId)
            ctx:update({ content = "Ticket closed.", components = {} })
        end
    end,
})
```

Putting data in the custom id lets the handler work after a restart, because everything it needs travels with the button. Custom ids can be up to 100 characters.

| Context field | `"ticket:close:42"` |
| --- | --- |
| `ctx.customId` | `"ticket:close:42"` |
| `ctx.name` | `"ticket"` |
| `ctx.params` | `{ "close", "42" }` |
| `run(ctx, ...)` | `run(ctx, "close", "42")` |

A handler can take several names:

```luau
bot:component({ customId = { "approve", "deny" }, run = function(ctx)
    ctx:update(if ctx.name == "approve" then "Approved" else "Denied")
end })
```

In a file, return `discord.component({ ... })`:

```luau title="components/ticket.luau"
local discord = require("@discord")

return discord.component({
    customId = "ticket",
    run = function(ctx, action, id) ... end,
})
```

### Matching with a pattern

Use `match` for ids that don't have the `name:params` shape. It is a [Lua pattern](https://www.lua.org/manual/5.1/manual.html#5.4.1) matched against the whole custom id, and its captures become the parameters:

```luau
bot:component({
    match = "^page_(%d+)_(%a+)$",
    run = function(ctx, page, direction)
        -- "page_3_next" -> page = "3", direction = "next"
    end,
})
```

Anchor patterns with `^` and `$` so that they only match what you intend. Handlers registered by `customId` are tried before patterns, and patterns are tried in the order they were registered.

### Select menus

A select menu's handler gets the same parameters as a button's. The picked values are in `ctx.values`:

```luau
bot:component({
    customId = "color",
    run = function(ctx)
        ctx:reply({ content = `You picked {table.concat(ctx.values, ", ")}`, ephemeral = true })
    end,
})
```

For user, role, mentionable and channel selects, `ctx:selected()` returns the objects instead of their ids.

### Deferring

Component handlers aren't deferred automatically. If one may take more than about 2 seconds, set `defer`:

| `defer` | Does | Then use |
| --- | --- | --- |
| `true` or `"update"` | `ctx:deferUpdate()`: acknowledges without changing the message | `ctx:update(...)` / `ctx:edit(...)` |
| `"reply"` | `ctx:defer()`: a new "thinking..." message | `ctx:reply(...)` |
| `"ephemeral"` | `ctx:defer({ ephemeral = true })` | `ctx:reply(...)` |

```luau
bot:component({
    customId = "roles",
    defer = "ephemeral",
    run = function(ctx, action)
        -- ... slow API calls ...
        ctx:reply("Roles updated.")    -- fills in the ephemeral deferred message
    end,
})
```

### Checks

Component handlers support the same checks as commands: `permissions`, `botPermissions`, `ownerOnly`, `guildOnly`, `dmOnly`, `checks` and `cooldown`. Discord doesn't enforce permissions on components, so discord checks `permissions` at runtime. Anyone who can see a message can click its buttons, so put checks on handlers that do something sensitive:

```luau
bot:component({
    customId = "mod",
    permissions = { "ModerateMembers" },
    cooldown = { seconds = 3, scope = "user" },
    run = function(ctx, action, userId) ... end,
})
```

`ephemeral = true` makes the handler's replies ephemeral by default.

### Restricting who can click

To limit a button to one user, put that user's id in the custom id:

```luau
ui.button({ label = "Delete", customId = `del:{ctx.user.id}:{messageId}` })

bot:component({
    customId = "del",
    run = function(ctx, ownerId, messageId)
        if ctx.user.id ~= ownerId then
            return ctx:error("This isn't your button.")
        end
        ...
    end,
})
```

## Inline handlers

Instead of a `customId`, you can give a button, select or modal a `run` function. discord then generates a random custom id and keeps the function in memory:

```luau
ctx:reply({
    content = "Hello",
    components = {
        ui.button({ label = "Wave back", run = function(click)
            click:reply(`{click:displayName()} waves 👋`)
        end }),
    },
})
```

- The function receives the component's context, and `ctx.values` as its second argument for selects. Modals get `ctx.fields`.
- It can use the command's local variables, which makes small stateful UIs easy. See the counter in [Components](components.md#updating-a-component-message).
- It is kept for **15 minutes** by default. `ttl` sets the lifetime in seconds, and `once = true` removes it after the first click:

    ```luau
    ui.button({ label = "Claim", once = true, ttl = 3600, run = function(c) ... end })
    ```

- Inline handlers live in memory. After a restart, or once the TTL has passed, a click gets the ephemeral reply *"This has expired."*
- They go through [middleware](middleware.md) and error handling. They do **not** go through checks or `defer` settings, which only registered handlers have. Do any checks you need inside the function.
- Anyone who can see the message can click it. If only the command's user should be able to, compare `click.user.id` with `ctx.user.id`.

The generated ids start with `~cb:`. You can create one yourself with `ui.callback(fn, ttl, once)`, which returns a custom id, and remove it with `ui.removeCallback(customId)`.

!!! note "Reserved ids"
    Custom ids starting with `~` are reserved for the ids discord generates (`~cb:`, `~confirm:`, `~page:`, `~modal:`). They aren't split into a name and parameters, and a click on an unknown `~` id gets the reply *"This has expired."* Don't start your own ids with `~`.

## How a click is routed

For each component interaction, discord checks these in order and stops at the first match:

1. **Waiting collectors**: an active `ctx:awaitComponent`, `ctx:collect`, `ctx:confirm` or `ctx:paginate` that matches.
2. **Inline handlers**: a `~cb:` id that hasn't expired.
3. **Registered handlers**: a handler whose `customId` equals the id's name or the whole custom id.
4. **Patterns**: the first registered `match` pattern that matches.

If nothing matches, the bot doesn't answer, and after 3 seconds the user sees *"This interaction failed"*. Set `debug = true` on the bot to log *no component handler for custom id ...* when this happens.

Modal submits are routed the same way, through modal handlers and modal collectors. See [Modals](modals.md).

## Reading the message

`ctx.message` is the message the component is on, so the handler can read it:

```luau
bot:component({
    customId = "roles",
    run = function(ctx)
        local select = ctx:findComponent(ctx.customId)   -- the select itself, with its options
        for _, option in select.options do
            print(option.label, option.value)
        end
    end,
})
```

This lets the message store state, such as which roles a role menu offers, so you don't have to save it anywhere.
