# Modals

A modal is a pop-up form with text inputs, select menus, file uploads, radio groups and checkboxes. It can only be opened as the **first response** to a slash command, a context menu command or a component interaction.

## Opening a modal and waiting for it

`ctx:prompt` opens a modal, waits for the user to submit it, and returns the context of the submission:

```luau
bot:command({
    name = "feedback",
    description = "Send feedback",
    run = function(ctx)
        local submitted = ctx:prompt(ctx.ui.modal({
            title = "Feedback",
            ctx.ui.textInput({ customId = "subject", label = "Subject", maxLength = 100 }),
            ctx.ui.textInput({ customId = "body", label = "Details", style = "paragraph" }),
        }))

        if submitted == nil then
            return -- the user closed the modal, or 5 minutes passed
        end

        submitted:reply({
            content = `Thanks! Subject: {submitted.fields.subject}`,
            ephemeral = true,
        })
    end,
})
```

- The returned context belongs to the **modal submit** interaction. Reply on it (`submitted:reply(...)`), not on the original `ctx`, which was already used up when the modal opened.
- `ctx:prompt(modal, timeout)` waits `timeout` seconds (default 300) and returns `nil` if nothing is submitted. Discord doesn't tell the bot when a modal is closed, so a closed modal also ends in `nil`.
- Only submissions from the user who opened the modal are accepted.

## Modal handlers

For a modal that is opened from many places, or that must keep working across restarts, register a handler by custom id. This works the same way as [component handlers](component-handlers.md):

```luau
bot:command({
    name = "report",
    description = "Report a user",
    options = { { name = "user", type = "user", required = true } },
    run = function(ctx, args)
        ctx:showModal(ctx.ui.modal({
            title = "Report",
            customId = "report:" .. args.user.id,
            ctx.ui.textInput({ customId = "reason", label = "What happened?", style = "paragraph" }),
        }))
    end,
})

bot:modal({
    customId = "report",
    run = function(ctx, fields)
        local userId = ctx.params[1]
        ctx:reply({ content = `Reported <@{userId}>: {fields.reason}`, ephemeral = true })
    end,
})
```

`run(ctx, fields)` receives the submitted values. The parameters from the custom id are in `ctx.params`. Like component handlers, modal handlers support `match`, `defer`, `ephemeral` and `checks`.

In a file, return `discord.modal({ ... })`, or a plain table with `customId`, `run` and `modal = true`.

## Inline modal handlers

As with buttons, a modal can carry its own `run` function:

```luau
ctx:showModal(ctx.ui.modal({
    title = "Rename",
    ctx.ui.textInput({ customId = "name", label = "New name", value = currentName }),
    run = function(m, fields)
        m:reply(`Renamed to {fields.name}`)
    end,
}))
```

An inline modal handler runs once. It expires after 15 minutes unless you set `ttl`.

## Inputs

Every input needs a `customId`. The submitted value is stored under that key in `fields` (also `ctx.fields`).

### Labels

Each input is shown with a label and an optional description. To set both, wrap the input in `ui.label`:

```luau
ui.label({
    label = "Your feedback",
    description = "Be as detailed as you like",
    component = ui.textInput({ customId = "text", style = "paragraph" }),
})
```

An input placed directly in `ui.modal` is wrapped for you, using its `label` field (or its custom id) as the label:

```luau
ui.modal({
    title = "Quick",
    ui.textInput({ customId = "name", label = "Name" }),  -- same as ui.label({ label = "Name", ... })
})
```

### Text input

```luau
ui.textInput({
    customId = "bio",
    style = "paragraph",      -- "short" (one line, the default) or "paragraph"
    placeholder = "Tell us about yourself",
    value = "prefilled text",
    minLength = 10,
    maxLength = 1000,
    required = false,         -- inputs are required unless you set this
})
```

Value: a string.

### Selects

String, user, role, mentionable and channel selects all work in modals. Wrap them in a label:

```luau
ui.label({
    label = "Topic",
    component = ui.select({ customId = "topic", options = { "Bug", "Idea", "Other" }, required = true }),
})
```

Value: a **list** of the picked values, so the first one is `fields.topic[1]`.

### File upload

```luau
ui.label({
    label = "Screenshots",
    component = ui.fileUpload({ customId = "shots", min = 1, max = 3, required = true }),
})
```

Value: a list of attachment ids. `ctx:uploads("shots")` returns the attachment objects instead (`url`, `filename`, `size`, ...):

```luau
for _, file in submitted:uploads("shots") do
    print(file.filename, file.url)
end
```

### Radio group

The user picks exactly one option:

```luau
ui.label({
    label = "Priority",
    component = ui.radioGroup({
        customId = "priority",
        options = { "Low", { label = "High", value = "high", description = "Pages someone" } },
        required = true,
    }),
})
```

Value: the picked value.

### Checkbox group

The user can tick any number of options:

```luau
ui.label({
    label = "Notify me about",
    component = ui.checkboxGroup({ customId = "notify", options = { "Releases", "Outages", "News" }, min = 0, max = 3 }),
})
```

Value: a list of the ticked values.

### Checkbox

```luau
ui.label({ label = "I agree to the rules", component = ui.checkbox({ customId = "agree", default = false }) })
```

Value: `true` or `false`.

### Text in a modal

To add explanatory text between inputs, use `ui.text(...)`:

```luau
ui.modal({
    title = "Apply",
    ui.text("Answer honestly. Applications are reviewed weekly."),
    ui.textInput({ customId = "why", label = "Why do you want to join?", style = "paragraph" }),
})
```

## Reading values

```luau
fields.subject                 -- by custom id
ctx:field("subject", "none")   -- with a fallback
ctx.fields                     -- the same table as fields
```

| Input | Value |
| --- | --- |
| text input | a string |
| select (any kind) | a list of values |
| file upload | a list of attachment ids (use `ctx:uploads(id)` for the attachments) |
| radio group | the picked value |
| checkbox group | a list of values |
| checkbox | a boolean |

## Answering a submission

A modal submit is an interaction, so like any other it must be answered within 3 seconds:

- `ctx:reply(...)`, `ctx:defer()` and `ctx:followUp(...)` work as usual.
- If the modal was opened from a **component**, `ctx.message` is the message that component is on, and `ctx:update(...)` edits it.
- A modal submit can't open another modal.

## Limits

- The title can be up to 45 characters. If you leave it out, discord uses `"Form"`.
- A modal can have up to 5 top-level components (labels or text blocks).
- Text inputs accept up to 4000 characters.
