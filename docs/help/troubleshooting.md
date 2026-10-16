# Troubleshooting

Start with the [`whoami` example](../examples/bundled.md#whoami) when a bot won't connect: it checks the token, the privileged intents, the interactions endpoint and the gateway in one go. Turn on `debug = true` (gateway payloads) and `debugRest = true` (REST calls) to see what's happening.

## The bot doesn't start

### `Bot.new: pass a token (or set DISCORD_TOKEN env variable)`

No `token` option and no `DISCORD_TOKEN` environment variable. Under `sn run`, environment variables come from your shell: `DISCORD_TOKEN=... sn run`.

### `stopped: authentication failed (bad token)`

Discord rejected the token (close code 4004, or 401 on `/gateway/bot`). Reset it on the **Bot** page of the Developer Portal and paste the new one. Check for stray quotes; whitespace at the end and a leading `Bot ` are removed for you.

### `disallowed intents (enable privileged intents in the developer portal)`

Close code 4014: the bot asked for a privileged intent the portal hasn't enabled. Normally discord drops such intents before connecting; you'll only see this with `strictIntents = true`, or if the application couldn't be fetched. Enable the intent under **Bot > Privileged Gateway Intents**, or stop requesting it.

### `warning: MessageContent isn't enabled for this application ...; connecting without it`

Not an error: the bot connected without the intent. Enable it in the portal if you need prefix commands in servers.

### `could not get the gateway url` / connection keeps retrying

Discord isn't reachable: network problems, a firewall, or a region where Discord is blocked. The client keeps retrying with a backoff (up to 60 s).

## Commands don't show up

1. **Look at the log.** On startup you should see `synced N command(s) to ...`. If you see `syncing commands (...) failed: ...`, the error says why (often an invalid field: see below).
2. **Global commands** can take a moment to appear in the client. Press ++ctrl+r++ to reload Discord. While developing, use `devGuild` for instant updates.
3. **Was the bot invited with `applications.commands`?** Without that scope, commands don't appear in the server. Re-invite with `scope=bot+applications.commands`.
4. **Nothing changed according to the hashes.** If commands were removed elsewhere (another machine, `clearCommands`, the portal), `.commands.lock.json` doesn't know. Start once with `syncCommands = "force"` or delete the file.
5. **`permissions`** hides the command from members without them. Check with an admin account, and under *Server Settings > Integrations*.
6. **`contexts` / `guildOnly`**: a command limited to servers doesn't show in DMs, and vice versa.
7. **Prefix-only commands** (`type = "prefix"`) are never registered; that's expected.

### `syncing commands (global) failed: PUT ...: 400 (code 50035) Invalid Form Body`

A definition breaks one of Discord's rules that discord doesn't check itself, such as a description over 100 characters, more than 25 options or choices, or invalid characters in a name. `err.errors` (printed after the message) points at the field: `{"0":{"options":{"2":{"description":...}}}}` means command 0, option 2.

## "The application did not respond" / "This interaction failed"

Discord waited 3 seconds without an answer.

- **A slow command**: with `autoDefer` on (the default) commands are deferred at ~2.3 s, so this usually means the bot is overloaded or the network is slow. Defer explicitly at the top (`defer = true`).
- **A slow component or modal handler**: these aren't auto-deferred. Add `defer = "update"` (or `"reply"` / `"ephemeral"`) to the definition.
- **No handler matched**: a button's custom id has no registered handler (typo, or the handler wasn't loaded). Set `debug = true` to log `no component handler for custom id "..."`.
- **The handler errored before answering**: discord tries to send the error message, but check the console for the traceback.
- **HTTP interactions**: the handler must answer within `responseTimeout` (2.8 s). See [HTTP interactions](../guide/http-interactions.md).
- **Another process is running the same bot** and answering (or failing) first. Stop the other one.

## "This has expired."

The button was made with an inline `run` handler (or by `confirm`/`paginate`), and the bot has restarted or the handler's TTL (15 minutes by default) has passed. Use a [registered handler](../guide/component-handlers.md#registered-handlers) for buttons that must keep working, or set `ttl`.

## Prefix commands don't respond

- **No Message Content intent**: without it, messages in servers have no text. Enable it in the portal and request it (discord requests it by default when `prefix` is set). Mentions (`@Bot ping`) and DMs work without it.
- **No prefix set**: with no `prefix` option, only `@Bot command` works (discord prints a note about this at startup).
- **The command isn't a prefix command**: slash commands need `prefix = true` (or `type = "prefix"`).
- **The bot can't see or send in the channel**: check its permissions.
- **The message came from a bot**: ignored unless `ignoreBots = false`.
- **`GuildMessages` / `DirectMessages` intents missing**: without them no messages arrive at all.

## Errors when sending

### `messages with components v2 can't have content or embeds`

You mixed `content`/`embeds` with v2 components (`ui.container`, `ui.text`, `ui.section`, ...). Put the text in `ui.text(...)`.

### `50013 Missing Permissions`

The bot lacks a permission in that channel, **or** the action targets someone whose highest role is above the bot's (kicking, banning, timing out, managing roles). Move the bot's role higher in *Server Settings > Roles*. Use `botPermissions` on commands to catch the simple cases with a clear message.

### `50001 Missing Access`

The bot can't see the channel at all.

### `50007 Cannot send messages to this user`

The user has DMs from server members turned off, or shares no server with the bot. Fall back to replying in the channel.

### `40060 Interaction has already been acknowledged`

Something answered the interaction twice. Usually two bot processes running at once, or a manual `rest` call answering an interaction discord also answered.

### `nothing to edit yet: reply or defer first`

`ctx:edit` on an interaction that hasn't been answered. Use `ctx:reply` first.

### `the application id isn't known yet`

`api.commands.*`, `api.interactions.*` or `syncCommands` was called before `READY`. Wait for `READY` (`bot.ready:wait()`), or pass `applicationId` to `discord.new`.

### `ctx:showModal() is only available for interactions`

Modals can't be opened from prefix commands or events. And within interactions, only as the first response, not after `defer` or from a modal submit.

## Loading problems

### `can't load "commands": no such file or folder (looked in ...)`

Paths are relative to the file that calls `bot:load`, not the working directory. Check the spelling and location.

### `can't load "commands" in this build: folders are read from the program's assets ...`

In a built program, add the folder to `build.assets` in `.sonata/project.luau`. See [Building a program](../getting-started/building.md).

### `don't know how to register the table from ...`

A file returned a table discord can't classify. Give commands a `name` and `run`, components a `customId` and `run`, events an `event` and `run`, or wrap it in `discord.command(...)` etc.

### `command "x" is defined twice (a.luau and b.luau)`

Two definitions share a name (and type). Rename one; remember aliases count for prefix commands.

### `required option "x" must come before optional ones`

Move required options to the front of `options`.

## Events don't fire

- The intent for the event is missing; discord warns at registration: `warning: "GUILD_MEMBER_ADD" needs one of these intents ...`.
- It's a privileged intent that was dropped at startup (check for the `isn't enabled for this application` warning).
- The event name is misspelled. Names are normalized (`messageCreate` → `MESSAGE_CREATE`), but `memberAdd` isn't an event; it's `GUILD_MEMBER_ADD`.
- With `gateway = false`, there are no events at all.

## Rate limits

`[rest] rate limited on ... retrying in Ns` means Discord asked the bot to slow down; the request is retried automatically. Seeing it often means the bot sends a lot to one channel or route in a short time: batch work (e.g. `bulkDelete` instead of many deletes), or space it out.
