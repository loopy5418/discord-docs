# ui

```luau
local ui = discord.ui      -- or ctx.ui, or require("@discord/lib/components")
```

Builders for components, modals, embeds and polls. Each returns a plain table in Discord's JSON shape. See [Components](../guide/components.md) and [Modals](../guide/modals.md).

Every component builder that takes a table also accepts a numeric `id` (an identifier Discord echoes back, unrelated to `customId`).

## Layout

### `ui.row`

```luau
ui.row(...: Component) -> ActionRow
ui.row({ Component }) -> ActionRow
```

An action row holding up to 5 buttons or one select.

### `ui.container`

```luau
ui.container({ accent: Color?, spoiler: boolean?, id: number?, ...children }) -> Container
```

A v2 container. Children are the array part (or `components`); strings become `ui.text`; loose buttons/selects are wrapped in rows. `accentColor` and `color` are aliases of `accent`.

### `ui.section`

```luau
ui.section({ ...texts, accessory: Thumbnail | Button, id: number? }) -> Section
```

One to three text displays (strings become `ui.text`) with an accessory.

### `ui.text`

```luau
ui.text(content: string) -> TextDisplay
ui.text({ content: string, id: number? }) -> TextDisplay
```

### `ui.thumbnail`

```luau
ui.thumbnail(url: string) -> Thumbnail
ui.thumbnail({ url: string, description: string?, spoiler: boolean?, id: number? }) -> Thumbnail
```

### `ui.gallery` / `ui.mediaGallery`

```luau
ui.gallery({ ...items, id: number? }) -> MediaGallery
-- items: url strings or { url, description?, spoiler? }; also accepts { items = { ... } }
```

### `ui.file`

```luau
ui.file(url: string) -> File
ui.file({ url: string, spoiler: boolean?, id: number? }) -> File
```

`url` must be `attachment://<name>` of a file uploaded with the message.

### `ui.separator`

```luau
ui.separator({ divider: boolean?, spacing: ("small" | "large" | number)?, id: number? }?) -> Separator
```

`divider` defaults to `true`.

## Interactive

### `ui.button`

```luau
ui.button({
    label: string?,
    style: ("primary" | "secondary" | "success" | "danger" | "link" | "premium" | number)?,
    customId: string?,     -- or custom_id
    run: ((ctx) -> ())?,   -- inline handler instead of customId
    ttl: number?,          -- inline handler lifetime in seconds (default 900)
    once: boolean?,        -- inline handler fires once
    url: string?,          -- link button
    skuId: string?,        -- premium button
    emoji: (string | table)?,
    disabled: boolean?,
    id: number?,
}) -> Button
```

Default style: `link` with `url`, `premium` with `skuId`, otherwise `primary`. Raises if a non-link, non-premium button has neither `customId` nor `run`, or a link button lacks `url`.

### `ui.link`

```luau
ui.link(label: string, url: string, emoji: (string | table)?) -> Button
```

### `ui.select` / `ui.stringSelect`

```luau
ui.select({
    customId: string? | run: function?, ttl: number?, once: boolean?,
    options: { string | SelectOption | { label, value?, description?, emoji?, default? } },
    placeholder: string?,
    min: number?, max: number?,         -- or minValues / maxValues
    disabled: boolean?,
    required: boolean?,                 -- in modals
    id: number?,
}) -> StringSelect
```

### `ui.option`

```luau
ui.option(label: string, value: string?, extra: { description: string?, emoji: any?, default: boolean? }?) -> SelectOption
```

`value` defaults to `label`.

### `ui.userSelect`, `ui.roleSelect`, `ui.mentionableSelect`, `ui.channelSelect`

```luau
ui.userSelect({ customId | run, placeholder?, min?, max?, disabled?, required?, defaults?: { string | { id, type } } })
ui.roleSelect({ ... })
ui.mentionableSelect({ ... })                -- default value type "user"
ui.channelSelect({ ..., channelTypes?: { string | number } })
```

`defaults` (or `defaultValues`) preselects ids. Channel types are names from [`enums.ChannelTypeAlias`](enums.md#channeltypealias) or numbers.

## Modals

### `ui.modal`

```luau
ui.modal({ title: string?, customId: string? | run: ((ctx, fields) -> ())?, ttl: number?, ...components }) -> Modal
```

`title` defaults to `"Form"`. Inputs placed directly are wrapped in a label (from their `label` or custom id). Without `customId` or `run`, a random `~modal:` id is generated (used by `ctx:prompt`). Inline `run` handlers fire once.

### `ui.label`

```luau
ui.label({ label: string, description: string?, component: Component, id: number? }) -> Label
```

### `ui.textInput`

```luau
ui.textInput({
    customId: string,
    style: ("short" | "paragraph" | number)?,  -- default "short"
    label: string?,                             -- used when wrapped automatically
    placeholder: string?, value: string?,
    minLength: number?, maxLength: number?,
    required: boolean?,
    id: number?,
}) -> TextInput
```

### `ui.fileUpload`

```luau
ui.fileUpload({ customId: string, min: number?, max: number?, required: boolean?, id: number? }) -> FileUpload
```

### `ui.radioGroup`

```luau
ui.radioGroup({ customId: string, options: { string | { label, value?, description?, default? } }, required: boolean?, id: number? })
```

### `ui.checkboxGroup`

```luau
ui.checkboxGroup({ customId: string, options: { ... }, min: number?, max: number?, required: boolean?, id: number? })
```

### `ui.checkbox`

```luau
ui.checkbox({ customId: string, default: boolean?, id: number? })
```

## Embeds and polls

### `ui.embed`

```luau
ui.embed({
    title: string?, description: string?, url: string?,
    color: Color?,
    timestamp: (true | number | string)?,           -- now, unix seconds, or ISO 8601
    author: (string | { name, url?, icon? })?,      -- icon also iconUrl / icon_url
    footer: (string | { text, icon? })?,
    image: (string | { url })?,
    thumbnail: (string | { url })?,
    fields: { { name, value, inline? } | { [1]: name, [2]: value, [3]: inline? } }?,
}) -> Embed
```

### `ui.poll`

```luau
ui.poll({
    question: string,
    answers: { string | { text: string, emoji: any? } },
    duration: number?,          -- hours, default 24
    multi: boolean?,            -- or allowMultiselect
}) -> Poll
```

## Inline handlers

### `ui.callback`

```luau
ui.callback(run: function, ttl: number?, once: boolean?) -> string
```

Stores `run` and returns a custom id (`~cb:...`) that triggers it. `ttl` defaults to 900 seconds. Expired entries are swept at most once a minute.

### `ui.findCallback`

```luau
ui.findCallback(customId: string) -> function?
```

The live handler for an id (removes it if `once`).

### `ui.removeCallback`

```luau
ui.removeCallback(customId: string)
```

## Helpers

| Function | Returns |
| --- | --- |
| `ui.color(value)` | A color number from `"#rrggbb"`, `"rrggbb"`, a number, or `{ r, g, b }`; `nil` for `nil` |
| `ui.emoji(value)` | A partial emoji from `"👍"`, `"<:name:id>"`, `"<a:name:id>"`, `"name:id"`, an id, or a table |
| `ui.isV2(components)` | Whether a top-level list contains v2 layout components |
| `ui.normalize(components)` | The list with loose buttons/selects wrapped in rows |
| `ui.disableAll(components)` | A deep copy with buttons (except links) and selects disabled |
| `ui.find(components, customId)` | The first component with that custom id, searching rows, containers, section accessories and labels |
