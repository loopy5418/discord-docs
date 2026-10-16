# Responding

This page covers how to answer commands, components and modals: replies, deferring, follow-ups, edits and ephemeral messages.

## `ctx:reply`

For most handlers, `ctx:reply` is all you need:

```luau
ctx:reply("Hello!")
ctx:reply({ content = "Only you can see this", ephemeral = true })
ctx:reply({ embeds = { ctx.ui.embed({ title = "Hi", description = "An embed" }) } })
```

It accepts anything described in [Message content](messages.md): a string, an options table, a single embed or a single component.

Where the reply goes depends on what happened before it:

| Situation | `ctx:reply` does |
| --- | --- |
| Interaction, not answered yet | Sends the interaction response |
| Interaction, after `ctx:defer()` | Fills in the deferred "thinking..." message |
| Interaction, already answered | Sends a follow-up message |
| Prefix command | Replies to the command message (a Discord reply with a reference) |

So you can call `ctx:reply` more than once without tracking the state yourself:

```luau
ctx:reply("Step 1 done")   -- the response
ctx:reply("Step 2 done")   -- a follow-up
```

`ctx:reply` returns the message, or `nil` and an error:

```luau
local message, err = ctx:reply("Hi")
if message == nil then
    print("couldn't reply:", err)
end
```

For interaction responses the message comes back from Discord because discord asks for it (`with_response`). Over [HTTP interactions](http-interactions.md) the first response goes back in the HTTP reply instead, so `reply` returns `true`. Call `ctx:fetchReply()` when you need the message.

## The three-second rule

Discord gives a bot **3 seconds** to acknowledge an interaction. After that, the user sees *"The application did not respond"*. Once acknowledged, the interaction token stays valid for **15 minutes**, which is the window for follow-ups and edits.

When a command needs more than a moment, **defer** it:

```luau
run = function(ctx)
    ctx:defer()                  -- shows "Bot is thinking..."
    local data = slowLookup()     -- take your time (up to 15 minutes)
    ctx:reply(`Found {data}`)     -- replaces the "thinking..." message
end
```

There are three ways to defer:

- **Manually**, with `ctx:defer()` or `ctx:defer({ ephemeral = true })`.
- **In the definition**, with `defer = true` or `defer = "ephemeral"`. discord defers before `run` starts:

    ```luau
    bot:command({ name = "report", description = "Build a report", defer = true, run = function(ctx) ... end })
    ```

- **Automatically**. With `autoDefer` (on by default), a command that hasn't answered within about 2.3 seconds is deferred for you. A slow command still works even if you forgot to defer, at the cost of a moment of "thinking...". Turn it off with `autoDefer = false`.

!!! note
    `autoDefer` applies to slash and context menu commands. Component and modal handlers aren't auto-deferred. Give them `defer = "update"` (or `"reply"` / `"ephemeral"`) in their definition, or call `ctx:deferUpdate()` yourself. See [Component handlers](component-handlers.md#deferring).

For prefix commands, `ctx:defer()` shows the typing indicator.

## Ephemeral messages

Ephemeral messages are visible only to the user who triggered the interaction, and are labelled "Only you can see this". They exist only for interactions. For prefix commands, `ephemeral` is ignored.

```luau
ctx:reply({ content = "Secret", ephemeral = true })
```

To make every reply of a command ephemeral by default, set `ephemeral = true` on the definition. A single message can still override it:

```luau
bot:command({
    name = "settings",
    description = "Your settings",
    ephemeral = true,              -- replies default to ephemeral
    run = function(ctx)
        ctx:reply("Only you see this")
        ctx:reply({ content = "Everyone sees this", ephemeral = false })
    end,
})
```

!!! warning "A deferred response can't change visibility"
    `ctx:defer()` decides whether the eventual response is ephemeral. A later `ctx:reply({ ephemeral = ... })` that fills in the deferred message can't change that. Defer with the visibility you want: `ctx:defer({ ephemeral = true })`, or `defer = "ephemeral"` in the definition.

    `ctx:error()` handles this for you. On a public deferred response it deletes the "thinking..." message and sends the error as an ephemeral follow-up.

## Follow-ups

`ctx:followUp(msg)` sends another message after the first response. Each follow-up can have its own visibility:

```luau
ctx:reply("Working on it")
ctx:followUp({ content = "Here's a private detail", ephemeral = true })
```

If nothing has been sent yet, `followUp` sends the first response instead. For prefix commands it sends another reply.

## Editing

| Method | Edits |
| --- | --- |
| `ctx:edit(msg)` (alias `ctx:editReply`) | The response to this interaction. For prefix commands, the bot's first reply. |
| `ctx:update(msg)` | The message a component is on (components, and modals opened from a component) |

```luau
ctx:reply("Counting...")
for i = 3, 1, -1 do
    ctx:sleep(1)
    ctx:edit(tostring(i))
end
ctx:edit("Liftoff!")
```

`ctx:edit` needs something to edit. On an interaction that hasn't been answered, it returns `nil, "nothing to edit yet: reply or defer first"`. For prefix commands without a reply yet, it sends one.

Edits change only the fields you pass. To remove components, pass `components = {}`. To remove content, pass `content = ""`.

## Components: `update` and `deferUpdate`

A button or select press is also an interaction that must be answered within 3 seconds. Usually you answer by changing the message the component is on:

```luau
bot:component({
    customId = "like",
    run = function(ctx)
        ctx:update({ content = "Liked!", components = {} })
    end,
})
```

| Method | What the user sees |
| --- | --- |
| `ctx:update(msg)` | The message changes immediately |
| `ctx:deferUpdate()` | Nothing yet. Follow it with `ctx:update(msg)` or `ctx:edit(msg)` within 15 minutes. |
| `ctx:reply(msg)` | A new message (often ephemeral), leaving the original as it was |
| `ctx:defer()` | A new "thinking..." message, filled in by the next `reply` |

```luau
run = function(ctx)
    ctx:deferUpdate()
    local result = slowWork()
    ctx:update({ content = `Done: {result}` })
end
```

After `deferUpdate`, `ctx:reply` sends a follow-up rather than editing the message.

## Errors

`ctx:error(text)` (alias `ctx:fail`) sends an ephemeral `❌ text` with mentions disabled:

```luau
if not args.user then
    return ctx:error("Pick a user.")
end
```

It works at any point. Before a response it is the response, and after one it is a follow-up. For autocomplete it sends an empty list of choices. discord uses it for failed [checks](checks.md) and handler errors too.

## Plain channel messages

`ctx:send(msg)` posts a normal message to the channel. It is not a reply, and doesn't count as answering the interaction:

```luau
ctx:send("Announcement!")
ctx:reply({ content = "Posted.", ephemeral = true })
```

`send` goes through the bot user, so the bot needs access to the channel. For [user installs](user-installs.md) in servers the bot isn't in, use `reply` and `followUp`.

## Deleting and fetching

```luau
ctx:deleteReply()               -- deletes the response (or the bot's prefix reply)
local message = ctx:fetchReply() -- the response message (interactions), or the bot's first reply (prefix)
```

## Modals

A modal must be the **first** response to a command or component. You can't defer first, and you can't open a modal from a modal submit or a prefix command:

```luau
ctx:showModal(ctx.ui.modal({
    title = "Report",
    ctx.ui.textInput({ customId = "reason", label = "Reason", style = "paragraph" }),
}))
```

`ctx:prompt(modal)` opens a modal and waits for the submission. See [Modals](modals.md).

## DMs, reactions and typing

```luau
ctx:dm("Here's your receipt")              -- DMs the caller
ctx:dm("You were mentioned", someUserId)   -- or someone else
ctx:react("👍")                             -- reacts to the prefix message, the component's message,
                                            -- or the message a message command targets
ctx:react("<:pepe:123456789012345678>", messageId)  -- another message in this channel
ctx:typing()                                -- "Bot is typing..." for ~10 seconds
```

`ctx:dm` fails (returns `nil, err`) when the user has DMs from server members turned off.

## Summary

| Call | Not answered | Deferred (`defer`) | Deferred (`deferUpdate`) | Answered |
| --- | --- | --- | --- | --- |
| `reply` | response | fills in "thinking" | follow-up | follow-up |
| `followUp` | response | follow-up | follow-up | follow-up |
| `edit` | error | edits response | edits component message | edits response |
| `update` (components) | updates message | edits response | edits message | edits response |
| `defer` / `deferUpdate` | defers | no-op | no-op | no-op |
| `showModal` | opens modal | fails | fails | fails |
