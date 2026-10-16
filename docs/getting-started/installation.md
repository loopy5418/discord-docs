# Installation

## Install Sonata

discord runs on [Sonata](https://sonata.5418.org). Install the `sn` command if you don't have it:

=== "Linux / macOS"

    ```sh
    curl -fsSL https://sonata.5418.org/install.sh | bash
    ```

    Without cURL:

    ```sh
    wget -qO- https://sonata.5418.org/install.sh | bash
    ```

=== "Windows (PowerShell)"

    ```powershell
    irm https://sonata.5418.org/install.ps1 | iex
    ```

Check that it worked:

```sh
sn --version
```

See Sonata's [installation guide](https://sonata.5418.org/docs/guide/installation/) for other options, such as building from source.

## Create a project

```sh
sn init my-bot --lsp
cd my-bot
```

This creates the following files:

```text
my-bot/
├── main.luau            the entry script
└── .sonata/
    ├── project.luau     the manifest
    └── deps/            installed dependencies
```

`--lsp` also sets up the [Luau Language Server](https://github.com/JohnnyMorganz/luau-lsp), which gives you completion and type checking in your editor. It is optional, but discord ships types for every option and context field, and the language server uses them.

## Add discord

```sh
sn pkg add github.com/loopy5418/discord
```

This adds discord to the `dependencies` list in `.sonata/project.luau`, installs it and its dependency [httpx](https://github.com/loopy5418/httpx) into `.sonata/deps/`, and records the exact commits in `.sonata/lock.luau`.

To pin a branch, tag or commit, add `#<ref>`:

```sh
sn pkg add https://github.com/loopy5418/discord#main
```

Your manifest now contains something like this:

```luau title=".sonata/project.luau"
return {
    name = "my-bot",
    version = "0.1.0",
    entrypoint = "main.luau",
    dependencies = {
        { name = "discord", url = "https://github.com/loopy5418/discord" },
    },
}
```

Scripts load it with `require("@discord")`:

```luau title="main.luau"
local discord = require("@discord")
print(discord.fmt.bold("it works"))
```

```sh
sn run
```

!!! tip "Fetching dependencies on another machine"
    Sonata never downloads anything unless you ask. After cloning a project, install what the manifest lists with `sn pkg add --install`, or pass `--install` to `sn run` / `sn build`.

!!! tip "Version control"
    Commit `.sonata/project.luau` and `.sonata/lock.luau`, and add `.sonata/deps/` to `.gitignore`. Also ignore the files the bot writes or reads: `token` and `.commands.lock.json` (see [Registering and syncing](../guide/syncing.md)).

    ```gitignore title=".gitignore"
    .sonata/deps/
    dist/
    token
    .commands.lock.json
    ```

## Updating

```sh
sn pkg update discord
```

This fetches the latest commit of the ref in your manifest and updates the lock file.

## Restricting libraries

If your manifest sets `build.libraries`, Sonata allows the union of your list and each dependency's list. discord declares the libraries it needs (`task`, `json`, `fs`, `path`, `crypto`, `process`, `datetime`, `pocket`), and httpx declares its own (including `net`), so you only need to list what your own code uses.

## Next

[Set up a Discord application](discord-application.md) to get a token.
