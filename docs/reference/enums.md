# enums

```luau
local enums = discord.enums   -- or ctx.enums, or require("@discord/lib/enums")
```

Discord constants for API v10. Builders and options usually accept the names as strings too (`style = "primary"`), resolved by [`enums.resolve`](#enumsresolve). All tables are frozen.

## InteractionType

`Ping = 1`, `ApplicationCommand = 2`, `MessageComponent = 3`, `Autocomplete = 4`, `ModalSubmit = 5`

## CallbackType

`Pong = 1`, `ChannelMessageWithSource = 4`, `DeferredChannelMessageWithSource = 5`, `DeferredUpdateMessage = 6`, `UpdateMessage = 7`, `AutocompleteResult = 8`, `Modal = 9`, `LaunchActivity = 12`

## CommandType

`ChatInput = 1` (slash), `User = 2`, `Message = 3`, `PrimaryEntryPoint = 4`

## OptionType

`SubCommand = 1`, `SubCommandGroup = 2`, `String = 3`, `Integer = 4`, `Boolean = 5`, `User = 6`, `Channel = 7`, `Role = 8`, `Mentionable = 9`, `Number = 10`, `Attachment = 11`

## IntegrationType

`GuildInstall = 0`, `UserInstall = 1`

## InteractionContext

`Guild = 0`, `BotDM = 1`, `PrivateChannel = 2`

## ComponentType

| Name | Value | | Name | Value |
| --- | --- | --- | --- | --- |
| `ActionRow` | 1 | | `TextDisplay` | 10 |
| `Button` | 2 | | `Thumbnail` | 11 |
| `StringSelect` | 3 | | `MediaGallery` | 12 |
| `TextInput` | 4 | | `File` | 13 |
| `UserSelect` | 5 | | `Separator` | 14 |
| `RoleSelect` | 6 | | `Container` | 17 |
| `MentionableSelect` | 7 | | `Label` | 18 |
| `ChannelSelect` | 8 | | `FileUpload` | 19 |
| `Section` | 9 | | `RadioGroup` | 21 |
| | | | `CheckboxGroup` | 22 |
| | | | `Checkbox` | 23 |

## ButtonStyle

`Primary = 1`, `Secondary = 2`, `Success = 3`, `Danger = 4`, `Link = 5`, `Premium = 6`

## TextInputStyle

`Short = 1`, `Paragraph = 2`

## SeparatorSpacing

`Small = 1`, `Large = 2`

## MessageFlags

| Name | Value |
| --- | --- |
| `Crossposted` | 1 |
| `IsCrosspost` | 2 |
| `SuppressEmbeds` | 4 |
| `SourceMessageDeleted` | 8 |
| `Urgent` | 16 |
| `HasThread` | 32 |
| `Ephemeral` | 64 |
| `Loading` | 128 |
| `FailedToMentionSomeRolesInThread` | 256 |
| `SuppressNotifications` | 4096 |
| `IsVoiceMessage` | 8192 |
| `HasSnapshot` | 16384 |
| `IsComponentsV2` | 32768 |

## ChannelType

| Name | Value |
| --- | --- |
| `GuildText` | 0 |
| `DM` | 1 |
| `GuildVoice` | 2 |
| `GroupDM` | 3 |
| `GuildCategory` | 4 |
| `GuildAnnouncement` | 5 |
| `AnnouncementThread` | 10 |
| `PublicThread` | 11 |
| `PrivateThread` | 12 |
| `GuildStageVoice` | 13 |
| `GuildDirectory` | 14 |
| `GuildForum` | 15 |
| `GuildMedia` | 16 |

## ChannelTypeAlias

The short names accepted by `channelTypes` (options and channel selects):

`text = 0`, `dm = 1`, `voice = 2`, `group = 3`, `category = 4`, `announcement = 5`, `news = 5`, `announcementThread = 10`, `thread = 11`, `publicThread = 11`, `privateThread = 12`, `stage = 13`, `directory = 14`, `forum = 15`, `media = 16`

These are matched exactly (case-sensitive).

## ActivityType

`Playing = 0`, `Streaming = 1`, `Listening = 2`, `Watching = 3`, `Custom = 4`, `Competing = 5`

## MessageType

`Default = 0`, `RecipientAdd = 1`, `RecipientRemove = 2`, `Call = 3`, `ChannelNameChange = 4`, `ChannelIconChange = 5`, `ChannelPinnedMessage = 6`, `UserJoin = 7`, `GuildBoost = 8`, `ThreadCreated = 18`, `Reply = 19`, `ChatInputCommand = 20`, `ThreadStarterMessage = 21`, `ContextMenuCommand = 23`, `AutoModerationAction = 24`, `PollResult = 46`

## TimestampStyle

`ShortTime = "t"`, `LongTime = "T"`, `ShortDate = "d"`, `LongDate = "D"`, `ShortDateTime = "f"`, `LongDateTime = "F"`, `Relative = "R"`

## `enums.resolve`

```luau
enums.resolve(group: table, value: any, what: string?) -> any
```

Looks `value` up in `group` by name, case-insensitively (`"primary"` → `1`). Numbers and `nil` pass through. Unknown names raise `unknown <what> "<value>"`.

```luau
enums.resolve(enums.ButtonStyle, "danger")   --> 4
```
