# Options

Options are the arguments of a command. Each is a table with at least a `name`:

```luau
options = {
    { name = "user", type = "user", description = "Who", required = true },
    { name = "days", type = "integer", description = "How many days", min = 0, max = 7, default = 0 },
    { name = "reason", type = "string", description = "Why", maxLength = 400 },
}
```

`run(ctx, args)` receives the values keyed by name: `args.user`, `args.days`, `args.reason`.

!!! warning "Required options come first"
    Discord requires every required option to come before the optional ones. discord raises an error at registration otherwise.

## Types

| `type` | Aliases | Value in `args` |
| --- | --- | --- |
| `"string"` (default) | `"str"`, `"text"` | a string |
| `"integer"` | `"int"` | a whole number |
| `"number"` | `"float"` | a number |
| `"boolean"` | `"bool"` | `true` / `false` |
| `"user"` | | a user object, with `.member` attached in servers |
| `"member"` | | the guild member, with `.user` attached |
| `"channel"` | | a channel object |
| `"role"` | | a role object |
| `"mentionable"` | | a user (with `.member`) or a role |
| `"attachment"` | `"file"` | an attachment object (`url`, `filename`, `size`, `content_type`, ...) |

You can also pass Discord's numeric option type (`type = 3`). The value then comes through unchanged.

`user` and `member` are the same option for Discord. They differ in what your code gets:

```luau
{ name = "target", type = "user" }    -- args.target.id, args.target.username, args.target.member?.nick
{ name = "target", type = "member" }  -- args.target.nick, args.target.roles, args.target.user.username
```

For `member`, the value is `nil` when the picked user isn't in the server.

## Fields

| Field | Applies to | Description |
| --- | --- | --- |
| `name` | all | 1–32 lowercase characters, no spaces |
| `description` | all | 1–100 characters (default `"No description"`) |
| `type` | all | See above |
| `required` | all | The user must fill it in |
| `default` | all | The value when the option is left out (not sent to Discord) |
| `choices` | string, integer, number | Fixed values to pick from (up to 25) |
| `min` / `max` | integer, number | Allowed range (also `minValue` / `maxValue`) |
| `minLength` / `maxLength` | string | Allowed length (0–6000) |
| `channelTypes` | channel | Which kinds of channel can be picked |
| `autocomplete` | string, integer, number | Suggestions as the user types (see [Autocomplete](autocomplete.md)) |
| `nameLocalizations` / `descriptionLocalizations` | all | Translations |
| `rest` | prefix commands | Take the rest of the message (see [Prefix commands](prefix-commands.md#taking-the-rest-of-the-text)) |
| `skip` | prefix commands | `false` stops an optional typed argument from being skipped |

Discord checks `required`, `choices`, `min`/`max`, `minLength`/`maxLength` and `channelTypes` for slash commands, so values that break them never reach your code. discord checks the same rules when it parses [prefix commands](prefix-commands.md).

## Choices

```luau
{
    name = "size",
    type = "string",
    description = "Pizza size",
    required = true,
    choices = { "small", "medium", "large" },        -- the shown name and the value are the same
}

{
    name = "speed",
    type = "integer",
    choices = {
        { name = "Slow", value = 1 },
        { name = "Fast", value = 10 },
        { name = "Ludicrous", value = 100, nameLocalizations = { fr = "Démentiel" } },
    },
}
```

`args.speed` is the `value` (`1`, `10` or `100`), not the name.

## Channel types

```luau
{ name = "where", type = "channel", channelTypes = { "text", "announcement" } }
```

Accepted names: `text`, `dm`, `voice`, `group`, `category`, `announcement` (or `news`), `announcementThread`, `thread` (or `publicThread`), `privateThread`, `stage`, `directory`, `forum`, `media`. Numbers from [`enums.ChannelType`](../reference/enums.md#channeltype) also work.

## Defaults

`default` fills in a value when the option is left out, for slash and prefix commands alike:

```luau
options = {
    { name = "amount", type = "integer", min = 1, max = 100, default = 10 },
},
run = function(ctx, args)
    print(args.amount) -- 10 if the user didn't give one
end,
```

`ctx:getOption(name, fallback)` does the same inside a handler.

## Attachments

```luau
bot:command({
    name = "resize",
    description = "Resize an image",
    options = {
        { name = "image", type = "attachment", description = "The image", required = true },
        { name = "width", type = "integer", min = 16, max = 4096 },
    },
    run = function(ctx, args)
        local file = args.image
        ctx:reply(`Got {file.filename} ({file.size} bytes, {file.content_type}) at {file.url}`)
    end,
})
```

Download the file with httpx (`require("@httpx").get(file.url)`), or any HTTP client. For [prefix commands](prefix-commands.md#attachments), attachment options are filled from the files attached to the message.

## Resolved objects

For `user`, `member`, `channel`, `role`, `mentionable` and `attachment` options, Discord sends the resolved objects along with the interaction. discord swaps the raw ids for those objects, so `args.user.username` works without an API call.

Resolved channels are *partial*: `id`, `name`, `type`, `permissions`, and a few more fields. Call `ctx:fetchChannel(args.channel.id)` for the full channel.
