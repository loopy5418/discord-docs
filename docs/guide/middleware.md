# Middleware and errors

## Middleware

Middleware is a function that runs **around** every command, component, modal and autocomplete handler. It receives the context and a `next` function, which runs the rest of the chain and finally the handler:

```luau
bot:use(function(ctx, next)
    local started = os.clock()
    next()
    ctx:log(`took {math.floor((os.clock() - started) * 1000)}ms`)
end)
```

- Middleware runs in the order it was added, and each one decides when to call `next()`.
- **Not calling `next()`** stops the handler from running. This makes middleware useful for global blocks:

    ```luau
    bot:use(function(ctx, next)
        if BANNED[ctx.user.id] then
            return ctx:error("You're banned from using this bot.")
        end
        next()
    end)
    ```

- Middleware runs **after** [checks](checks.md) pass and after any `defer`, right before the handler.
- It also wraps [inline handlers](component-handlers.md#inline-handlers).
- It does **not** run for gateway [events](events.md).
- `bot:use` returns the bot, so calls can be chained.

### Passing data to the handler

`ctx.state` is a fresh table for each invocation, so middleware can prepare things for the handler:

```luau
bot:use(function(ctx, next)
    if ctx.guildId then
        ctx.state.settings = loadSettings(ctx.guildId)
    end
    next()
end)

-- in a command
run = function(ctx)
    local settings = ctx.state.settings
end
```

### Telling handlers apart

```luau
bot:use(function(ctx, next)
    if ctx.kind == "slash" or ctx.kind == "prefix" then
        print(`[{ctx.kind}] {ctx.user.username}: {ctx.commandPath}`)
    elseif ctx.kind == "component" then
        print(`[button] {ctx.customId}`)
    end
    next()
end)
```

`ctx.commandPath` is `"config prefix set"` for commands, the handler's custom id (or pattern) for registered components and modals, and `"inline component"` / `"inline modal"` for inline handlers.

## Errors

Every handler runs protected. When one errors:

1. The error and its traceback are passed to your [error handlers](#error-handlers). Without any, it is printed:

    ```text
    [bot] error in ban:
    commands/moderation/ban.luau:23: attempt to index nil with 'id'
    stack traceback: ...
    ```

2. If the handler hasn't replied yet, the user gets an ephemeral *"❌ Something went wrong while running this."* (Not for events and autocomplete, which have no one to answer.)

The bot keeps running either way.

### Error handlers

```luau
bot:onError(function(err, ctx)
    -- err: the error message with its traceback
    -- ctx: the context, or nil if the error happened while routing
    local where = if ctx then (ctx.commandPath or ctx.event or ctx.kind) else "?"
    print(`[error] {where}: {err}`)

    -- for example, report it to a log channel
    if ctx then
        ctx.api.channels.send(LOG_CHANNEL, {
            content = `Error in \`{where}\` by {ctx.fmt.user(ctx.user and ctx.user.id or "0")}`,
            files = { { name = "error.txt", data = err } },
            allowedMentions = false,
        })
    end
end)
```

- Several error handlers can be added; they all run.
- Adding one replaces the default printing.
- An error inside an error handler is printed and ignored.

### The message users see

| Bot option | Effect |
| --- | --- |
| `errorMessage = "..."` | Replace the text |
| `errorMessage = false` | Don't tell the user anything |

The message is only sent if the handler hadn't replied. If it had deferred publicly, the "thinking..." message is replaced by the ephemeral error.

### Expected failures

Calls to the API don't raise: they return `nil, err` (see [Calling the API](api.md#errors)). Handle the failures you expect and tell the user something specific:

```luau
local ok, err = ctx.api.guilds.ban(ctx.guildId, user.id, { reason = reason })
if not ok then
    return ctx:error(`Couldn't ban: {err.message or tostring(err)}`)
end
```

For things that should never happen, `error(...)` and let the error handler deal with it.

### Errors outside handlers

Errors in tasks you start yourself (`task.spawn`, `task.delay`, `ctx:spawn`) aren't caught by discord. Sonata reports them without stopping the program. Wrap the work in `pcall` if you need to handle them, or install a global handler with Sonata's `task.onError`.
