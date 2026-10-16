# fmt

```luau
local fmt = discord.fmt   -- or ctx.fmt, or require("@discord/lib/format")
```

Helpers for Discord markdown, mentions, timestamps and CDN URLs. All return strings unless noted.

## Mentions

| Function | Result |
| --- | --- |
| `fmt.user(idOrUser)` | `<@id>` |
| `fmt.channel(idOrChannel)` | `<#id>` |
| `fmt.role(idOrRole)` | `<@&id>` |
| `fmt.command(name, id)` | `</name:id>`, a clickable command (`name` may include subcommands: `"config set"`) |
| `fmt.emoji(name, id, animated?)` | `<:name:id>` or `<a:name:id>` |

The mention functions accept an id or an object with `.id`.

## Timestamps

```luau
fmt.timestamp(t: number?, style: string?) -> string
```

`<t:seconds>` or `<t:seconds:style>`, shown in each user's timezone. `t` is unix seconds (default now).

| Style | Shows |
| --- | --- |
| `"t"` | 16:20 |
| `"T"` | 16:20:30 |
| `"d"` | 20/04/2021 |
| `"D"` | 20 April 2021 |
| `"f"` (default) | 20 April 2021 16:20 |
| `"F"` | Tuesday, 20 April 2021 16:20 |
| `"R"` | 2 months ago / in 5 minutes |

`enums.TimestampStyle` names these (`Relative = "R"`, ...).

## Markdown

| Function | Result |
| --- | --- |
| `fmt.bold(s)` | `**s**` |
| `fmt.italic(s)` | `*s*` |
| `fmt.underline(s)` | `__s__` |
| `fmt.strike(s)` | `~~s~~` |
| `fmt.spoiler(s)` | `\|\|s\|\|` |
| `fmt.code(s)` | `` `s` `` |
| `fmt.codeblock(s, language?)` | a fenced code block |
| `fmt.quote(s)` | `> ` on every line |
| `fmt.blockquote(s)` | `>>> s` |
| `fmt.heading(s, level?)` | `# s` (level 1–3) |
| `fmt.subtext(s)` | `-# s`, small grey text |
| `fmt.link(text, url)` | `[text](url)` |
| `fmt.list(items, ordered?)` | `- a` lines, or `1. a` |

## Escaping

| Function | Description |
| --- | --- |
| `fmt.escape(s)` | Escapes markdown characters so text shows literally |
| `fmt.escapeMentions(s)` | Breaks `@everyone` / `@here` with a zero-width space |
| `fmt.truncate(s, max)` | Cuts to `max` characters, ending with `...` |

## Users and servers

| Function | Description |
| --- | --- |
| `fmt.displayName(user, member?)` | Server nickname, then global name, then username |
| `fmt.avatar(user, size?)` | Avatar URL (`.gif` for animated), or the default avatar |
| `fmt.guildIcon(guild, size?)` | Icon URL, or `nil` |

`size` is a power of two between 16 and 4096.

## Time

| Function | Description |
| --- | --- |
| `fmt.snowflakeTime(id)` | Unix seconds an id was created at |
| `fmt.duration(seconds)` | `"1d 2h 3m 4s"` (`"0s"` for 0) |
| `fmt.parseDuration(text)` | Seconds from `"10m"`, `"1h30m"`, `"2d"`, `"1w"` (units `s m h d w`), or a plain number; `nil` if unreadable |

```luau
local seconds = fmt.parseDuration(args.duration)
if seconds == nil then
    return ctx:error("Use a duration like 10m or 1h30m.")
end
ctx.api.guilds.timeout(ctx.guildId, args.user.id, seconds)
ctx:reply(`Timed out until {fmt.timestamp(os.time() + seconds, "F")} ({fmt.duration(seconds)}).`)
```
