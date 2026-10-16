# Components

Components are the interactive and layout parts of a message: buttons, select menus, and the **components v2** layout blocks such as containers, text, images and separators. They are built with `discord.ui`, which is also available as `ctx.ui`:

```luau
local ui = discord.ui
```

Every builder returns a plain table in the JSON shape Discord expects, so you can inspect or change them before sending.

This page covers building components. [Component handlers](component-handlers.md) covers what happens when someone clicks them.

## Buttons

```luau
ctx:reply({
    content = "Do you agree?",
    components = {
        ui.button({ label = "Yes", style = "success", customId = "agree:yes" }),
        ui.button({ label = "No", style = "danger", customId = "agree:no" }),
        ui.button({ label = "Docs", url = "https://example.com" }),          -- a link button
    },
})
```

| Field | Description |
| --- | --- |
| `label` | The text on the button (up to 80 characters) |
| `style` | `"primary"` (default), `"secondary"`, `"success"`, `"danger"`, `"link"`, `"premium"` |
| `customId` | Identifies the button to your [handler](component-handlers.md) (up to 100 characters) |
| `run` | An inline handler, used instead of `customId` (see [inline handlers](component-handlers.md#inline-handlers)) |
| `url` | Makes a link button. No handler; it opens the URL. |
| `skuId` | Makes a premium (purchase) button |
| `emoji` | `"👍"`, `"<:name:id>"`, `"<a:name:id>"`, `"name:id"`, an id, or an emoji table |
| `disabled` | Greyed out |
| `ttl`, `once` | For inline handlers: lifetime in seconds, and whether it fires only once |

The style is chosen for you when you leave it out: `url` makes a link button, `skuId` a premium button, and anything else is primary.

`ui.link(label, url, emoji?)` is shorthand for a link button.

## Select menus

```luau
ctx:reply({
    content = "Pick your languages",
    components = {
        ui.select({
            customId = "langs",
            placeholder = "Choose...",
            min = 1,
            max = 3,
            options = {
                "Luau",                                                      -- label = value
                ui.option("Rust", "rust", { emoji = "🦀", description = "Fast" }),
                { label = "Go", value = "go", default = true },
            },
        }),
    },
})
```

The handler reads the picked values from `ctx.values` (`{ "rust", "go" }`).

Selects that pick Discord objects instead of fixed options:

```luau
ui.userSelect({ customId = "who", max = 5 })
ui.roleSelect({ customId = "roles", defaults = { roleId1, roleId2 } })      -- preselected
ui.mentionableSelect({ customId = "target" })                               -- users and roles
ui.channelSelect({ customId = "where", channelTypes = { "text", "forum" } })
```

`ctx:selected()` in the handler turns the picked ids into user, role or channel objects. Picked users get `.member` attached in servers.

| Field | Description |
| --- | --- |
| `customId` / `run` | As for buttons |
| `placeholder` | Text shown before anything is picked |
| `min` / `max` | How many values can be picked (also `minValues` / `maxValues`) |
| `options` | String selects only: strings, `ui.option(...)`, or `{ label, value, description, emoji, default }` |
| `defaults` | User/role/mentionable/channel selects: preselected ids, or `{ id, type = "user" \| "role" \| "channel" }` |
| `channelTypes` | Channel selects: which kinds of channel can be picked |
| `disabled` | Greyed out |
| `required` | In modals: whether a value must be picked |

`ui.stringSelect` is an alias for `ui.select`.

## Action rows

Buttons and selects live in **action rows**. A row holds up to 5 buttons, or a single select. You can build rows yourself:

```luau
components = {
    ui.row(ui.button({ ... }), ui.button({ ... })),
    ui.row(ui.select({ ... })),
}
-- or ui.row({ button1, button2 })
```

Or leave buttons and selects loose, and discord packs them into rows: consecutive buttons share a row (up to 5), and each select gets its own. A message can have up to 5 action rows.

## Components v2

Components v2 lay out a whole message with components instead of `content` and `embeds`. discord detects them and sets the `IsComponentsV2` flag for you.

```luau
ctx:reply({
    components = {
        ui.container({
            accent = "#5865F2",
            ui.text("# Server status"),
            ui.section({
                "All systems operational.",
                ui.text(ctx.fmt.subtext("checked just now")),
                accessory = ui.thumbnail(iconUrl),
            }),
            ui.separator({ spacing = "large" }),
            ui.gallery({ screenshot1, screenshot2 }),
            ui.row(
                ui.button({ label = "Refresh", customId = "status:refresh" }),
                ui.link("Status page", "https://status.example.com")
            ),
        }),
    },
})
```

!!! warning
    A components v2 message **can't have `content` or `embeds`**. Put text in `ui.text(...)`. discord raises an error if you mix them. Once a message is sent as v2, it stays v2.

### `ui.container`

A box with an optional accent color bar. Its children are listed in the table's array part:

```luau
ui.container({
    accent = "#ED4245",        -- or accentColor / color: hex string, number or { r, g, b }
    spoiler = false,
    "A string becomes a text display",
    ui.text("So does this"),
    ui.button({ ... }),        -- loose buttons are wrapped in a row
})
```

### `ui.text`

A block of markdown text:

```luau
ui.text("## A heading\nSome **bold** text and a [link](https://example.com).")
```

### `ui.section`

Up to three text blocks with an **accessory** on the right, either a thumbnail or a button:

```luau
ui.section({
    "**Title**",
    "Description text",
    accessory = ui.thumbnail(avatarUrl),
})

ui.section({
    "Enable notifications?",
    accessory = ui.button({ label = "Enable", customId = "notify:on" }),
})
```

Strings in the array part become `ui.text` blocks.

### `ui.thumbnail`

```luau
ui.thumbnail(url)
ui.thumbnail({ url = url, description = "alt text", spoiler = true })
```

### `ui.gallery`

One to ten images or videos in a grid. `ui.mediaGallery` is an alias:

```luau
ui.gallery({ url1, url2 })
ui.gallery({
    { url = url1, description = "A cat" },
    { url = "attachment://dog.png", spoiler = true },
})
```

### `ui.file`

Shows an uploaded file. The URL must be an `attachment://` reference to a file in `files`:

```luau
ctx:reply({
    components = { ui.file("attachment://notes.txt") },
    files = { { name = "notes.txt", data = notes } },
})
```

### `ui.separator`

Vertical space, with or without a line:

```luau
ui.separator()                                    -- a line with small spacing
ui.separator({ divider = false, spacing = "large" })
```

### Component ids

Every builder accepts a numeric `id`. Discord echoes it back, and it is unrelated to `customId`. It's useful for finding a component in a message you received:

```luau
ui.text({ content = "Count: 0", id = 1 })
```

## Embeds and polls

`ui.embed` and `ui.poll` are also in `ui`. See [Message content](messages.md#embeds).

## Helpers

```luau
ui.color("#5865F2")             --> 5793266  (also "5865F2", 0x5865F2, { 88, 101, 242 })
ui.emoji("<a:wave:123>")        --> { id = "123", name = "wave", animated = true }
ui.disableAll(components)       --> a copy with every button and select disabled (link buttons stay)
ui.find(components, "vote:yes") --> the component with that custom id, anywhere in the tree
ui.isV2(components)             --> whether the list needs the IsComponentsV2 flag
ui.normalize(components)        --> loose buttons/selects wrapped in action rows
```

Disable components once they're no longer useful:

```luau
ctx:update({ components = ui.disableAll(ctx.message.components) })
```

## Updating a component message

From a component handler, `ctx:update(...)` replaces the message the component is on. Since a v2 message is entirely components, rebuild and send the whole layout. A render function keeps this simple:

```luau
bot:command({
    name = "counter",
    description = "A counter",
    run = function(ctx)
        local count = 0
        local function render()
            return {
                components = {
                    ui.container({
                        ui.text(`# {count}`),
                        ui.row(
                            ui.button({ label = "-1", style = "danger", run = function(c) count -= 1; c:update(render()) end }),
                            ui.button({ label = "+1", style = "success", run = function(c) count += 1; c:update(render()) end })
                        ),
                    }),
                },
            }
        end
        ctx:reply(render())
    end,
})
```

## Limits

Discord's main limits, which discord doesn't check for you:

- 5 action rows per message, and 5 buttons per row
- 25 options per select
- custom ids up to 100 characters, button labels up to 80
- components v2 messages: up to 40 components in total, and up to 4000 characters of text across all text displays

If you break one, the reply fails, and the error says which field is wrong (see [Calling the API](api.md#errors)).
