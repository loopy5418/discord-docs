# permissions

```luau
local Permissions = discord.permissions   -- or ctx.permissions, or require("@discord/lib/permissions")
```

Discord's permission bitfield is wider than 32 bits and arrives as a decimal string, so `bit32` can't handle it. This module keeps masks as plain Luau numbers (exact up to 2^53) and works on them arithmetically.

Wherever a function takes *permissions*, it accepts a name (`"BanMembers"`), a list of names, a mask number, a decimal string (`"8"`), or a mix in a list. Names are matched forgivingly (`"ban_members"`, `"BAN_MEMBERS"`, `"banmembers"`); unknown names raise.

## Flags

`Permissions.Flags[name]` is the flag value (`2^bit`):

| Name | Bit | | Name | Bit |
| --- | --- | --- | --- | --- |
| `CreateInstantInvite` | 0 | | `ManageNicknames` | 27 |
| `KickMembers` | 1 | | `ManageRoles` | 28 |
| `BanMembers` | 2 | | `ManageWebhooks` | 29 |
| `Administrator` | 3 | | `ManageGuildExpressions` | 30 |
| `ManageChannels` | 4 | | `UseApplicationCommands` | 31 |
| `ManageGuild` | 5 | | `RequestToSpeak` | 32 |
| `AddReactions` | 6 | | `ManageEvents` | 33 |
| `ViewAuditLog` | 7 | | `ManageThreads` | 34 |
| `PrioritySpeaker` | 8 | | `CreatePublicThreads` | 35 |
| `Stream` | 9 | | `CreatePrivateThreads` | 36 |
| `ViewChannel` | 10 | | `UseExternalStickers` | 37 |
| `SendMessages` | 11 | | `SendMessagesInThreads` | 38 |
| `SendTTSMessages` | 12 | | `UseEmbeddedActivities` | 39 |
| `ManageMessages` | 13 | | `ModerateMembers` | 40 |
| `EmbedLinks` | 14 | | `ViewCreatorMonetizationAnalytics` | 41 |
| `AttachFiles` | 15 | | `UseSoundboard` | 42 |
| `ReadMessageHistory` | 16 | | `CreateGuildExpressions` | 43 |
| `MentionEveryone` | 17 | | `CreateEvents` | 44 |
| `UseExternalEmojis` | 18 | | `UseExternalSounds` | 45 |
| `ViewGuildInsights` | 19 | | `SendVoiceMessages` | 46 |
| `Connect` | 20 | | `SendPolls` | 49 |
| `Speak` | 21 | | `UseExternalApps` | 50 |
| `MuteMembers` | 22 | | `PinMessages` | 51 |
| `DeafenMembers` | 23 | | `BypassSlowmode` | 52 |
| `MoveMembers` | 24 | | | |
| `UseVAD` | 25 | | | |
| `ChangeNickname` | 26 | | | |

`Permissions.All` is every flag combined.

## Functions

### `Permissions.resolve`

```luau
Permissions.resolve(input: any) -> number
```

Turns names, lists, numbers or decimal strings into one mask. `nil` → `0`.

### `Permissions.has`

```luau
Permissions.has(mask: number | string, wanted: any, ignoreAdmin: boolean?) -> boolean
```

Whether `mask` has every permission in `wanted`. `Administrator` implies everything unless `ignoreAdmin`.

### `Permissions.missing`

```luau
Permissions.missing(mask: number | string, wanted: any) -> { string }
```

The names in `wanted` that `mask` lacks (sorted). Empty when `mask` has `Administrator`.

### `Permissions.add` / `Permissions.remove`

```luau
Permissions.add(a, b) -> number      -- a | b
Permissions.remove(a, b) -> number   -- a & ~b
```

### `Permissions.names`

```luau
Permissions.names(mask) -> { string }
```

Every permission in a mask, sorted.

### `Permissions.toString`

```luau
Permissions.toString(mask) -> string
```

The decimal string Discord expects (no exponent).

### `Permissions.label`

```luau
Permissions.label("ManageMessages") -> "Manage Messages"
```

### `Permissions.compute`

```luau
Permissions.compute(guild, member, userId: string, channel?) -> number
```

A member's permissions computed the way Discord does: the guild owner gets everything; otherwise `@everyone` plus the member's roles (Administrator → everything); then, with a channel, its overwrites for `@everyone`, the member's roles (denies before allows), and the member. `guild` needs `id`, `owner_id` and `roles`; `member` needs `roles`.

## Examples

```luau
local P = discord.permissions

local mask = P.resolve({ "SendMessages", "EmbedLinks" })
print(P.toString(mask))                         --> "18432"
print(P.has(mask, "SendMessages"))              --> true
print(table.concat(P.missing(mask, { "SendMessages", "BanMembers" }), ", ")) --> "BanMembers"

-- an invite link
local url = `https://discord.com/oauth2/authorize?client_id={appId}&scope=bot+applications.commands&permissions={P.toString(mask)}`
```
