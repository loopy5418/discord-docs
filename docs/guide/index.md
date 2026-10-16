# Guide

The guide explains each part of discord by topic, with examples. For exact signatures and every field, see the [Reference](../reference/index.md).

## Basics

- [The bot](bot.md): creating a bot, its lifecycle, presence, and shared state.
- [The context](context.md): the `ctx` every handler receives.
- [Responding](responding.md): `reply`, `defer`, `followUp`, `edit`, `update`, ephemeral messages and the 3-second rule.
- [Message content](messages.md): text, embeds, files, mentions, replies, polls and flags.

## Commands

- [Slash commands](slash-commands.md): defining and registering commands.
- [Options](options.md): option types, choices, limits, defaults.
- [Autocomplete](autocomplete.md): suggestions as the user types.
- [Subcommands](subcommands.md): `/config prefix set` style commands.
- [Prefix commands](prefix-commands.md): `!ping`, hybrids, argument parsing.
- [Context menu commands](context-menus.md): right click > Apps.
- [Registering and syncing](syncing.md): how commands reach Discord.
- [User-installed apps](user-installs.md): commands that work wherever the user is.

## Interactivity

- [Components](components.md): buttons, selects, and components v2 layouts.
- [Component handlers](component-handlers.md): custom ids, registered handlers, inline handlers.
- [Modals](modals.md): forms with text inputs, selects, file uploads and checkboxes.
- [Waiting for input](waiting.md): `awaitComponent`, `collect`, `awaitMessage`, `confirm`, `paginate`.

## Running a bot

- [Events](events.md): gateway events and waiting for them.
- [Intents](intents.md): which events Discord sends, and privileged intents.
- [Checks, permissions and cooldowns](checks.md).
- [Middleware and errors](middleware.md).
- [Loading from files](loading.md).
- [The cache](cache.md).
- [Calling the API](api.md): `ctx.api`, raw REST, and errors.
- [HTTP interactions](http-interactions.md): receiving interactions without the gateway.
