# Autocomplete

Autocomplete shows suggestions while the user types an option. Unlike `choices`, the list can change, and the user can still type something that isn't in it.

## A static list

Pass a list, and discord filters it by what the user typed (a case-insensitive substring match):

```luau
local FRUITS = { "apple", "apricot", "banana", "blueberry", "cherry" }

bot:command({
    name = "fruit",
    description = "Pick a fruit",
    options = {
        { name = "name", type = "string", description = "A fruit", required = true, autocomplete = FRUITS },
    },
    run = function(ctx, args)
        ctx:reply(`You picked {args.name}`)
    end,
})
```

Typing `ap` suggests *apple* and *apricot*. List items can be strings, numbers, or `{ name = "...", value = ... }` tables, and the filter matches on `name`.

## A function

For dynamic suggestions, pass a function. It receives the context, the text typed so far and the option's name, and returns the choices:

```luau
{
    name = "tag",
    type = "string",
    description = "A tag",
    autocomplete = function(ctx, typed, optionName)
        local out = {}
        for name in ctx.store.tags or {} do
            if string.find(name, string.lower(typed or ""), 1, true) then
                table.insert(out, name)
            end
        end
        return out
    end,
}
```

The returned list can hold:

- strings or numbers, where the shown name and the value are the same
- `{ name = "Shown text", value = "sent value" }` tables

At most 25 choices are sent, and names are cut to 100 characters.

The function may yield, so it can make API calls. Keep in mind that Discord waits only **3 seconds** for suggestions.

### The other options

`ctx.options` holds the options the user has filled in so far, so suggestions can depend on them:

```luau
options = {
    { name = "category", type = "string", choices = { "fruit", "vegetable" }, required = true },
    {
        name = "item",
        type = "string",
        required = true,
        autocomplete = function(ctx, typed)
            local list = if ctx.options.category == "fruit" then FRUITS else VEGETABLES
            return list
        end,
    },
}
```

`ctx.focused` is `{ name, value, type }` for the option being typed in.

### Responding yourself

Instead of returning a list, you can call `ctx:respond(choices)`:

```luau
autocomplete = function(ctx, typed)
    ctx:respond({ { name = "First", value = "1" }, { name = "Second", value = "2" } })
end
```

If the function returns nothing (and didn't call `respond`), an empty list is sent. If it errors, the error goes to your [error handlers](middleware.md#error-handlers) and no suggestions are sent, so the user sees Discord's "loading options failed". Autocomplete handlers can't `reply`, `defer` or open modals.

## One handler for several options

A command-level `autocomplete` function handles every autocomplete option that doesn't have its own handler. Mark those options with `autocomplete = true`, which tells Discord the option has suggestions without giving it a handler of its own:

```luau
bot:command({
    name = "convert",
    description = "Convert units",
    options = {
        { name = "from", type = "string", required = true, autocomplete = true },
        { name = "to", type = "string", required = true, autocomplete = true },
    },
    autocomplete = function(ctx, typed, optionName)
        local units = if optionName == "from" then { "m", "km", "mi" } else { "m", "km", "mi", "ft" }
        return units
    end,
    run = function(ctx, args) ... end,
})
```

Subcommands can have their own `autocomplete` too. discord looks for a handler in this order: the option's own function or list, then the subcommand's `autocomplete`, then the command's.

!!! note
    Autocomplete suggestions are not validated. The user can ignore them and type anything, so check the value in `run`.

!!! note
    Autocomplete only exists for slash commands. In [prefix commands](prefix-commands.md) an autocomplete option is a normal argument.
