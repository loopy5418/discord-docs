# Building a program

`sn build` turns a Sonata project into one executable that runs without Sonata installed. Bots built with discord work the same way as under `sn run`, except for one detail about folders.

```sh
sn build
./dist/my-bot
```

The output path is `build.output` in the manifest, or `dist/<name>` by default. See Sonata's [building guide](https://sonata.5418.org/docs/guide/building/) for every flag.

## Folders need to be listed as assets

A built program contains every compiled `.luau` file, but there is no folder on disk for the loader to walk. To load folders, the loader lists the program's **assets** instead. Add each folder you pass to `bot:load` to `build.assets`:

```luau title=".sonata/project.luau" hl_lines="9"
return {
    name = "my-bot",
    version = "0.1.0",
    entrypoint = "main.luau",
    dependencies = {
        { name = "discord", url = "https://github.com/loopy5418/discord" },
    },
    build = {
        assets = { "commands", "components", "events" },
    },
}
```

Without this, the program stops at startup with an error that names the folder:

```text
can't load "commands" in this build: folders are read from the program's assets, so add "commands" to build.assets in .sonata/project.luau
```

What does and doesn't need an asset entry:

| `bot:load(...)` | Needs `build.assets`? |
| --- | --- |
| `"commands"` (a folder) | Yes, the folder |
| `"commands/ping"` or `"commands/ping.luau"` (one file) | No, modules are compiled in |
| `"plugins/*.luau"` (a glob) | Yes, globs only match listed assets |
| a table or function | No |

!!! tip "Check before you build"
    Under `sn run`, `@sonata/pocket` only shows listed assets too. But discord's loader reads folders straight from disk under `sn run`, so a missing asset entry only shows up in the built program. Run the built program once before you ship it.

## Packages that load their own folders

By default the loader lists the **root project's** assets. A library package that calls `bot:load` on its own folders must pass a lister written in its own code, because `@sonata/pocket` answers for the package whose function calls it:

```luau
local pocket = require("@sonata/pocket")

local bot = discord.new({
    assets = function(folder)
        return pocket.list(folder)
    end,
})
```

Most bots are root projects and don't need this.

## Paths at runtime

In a built program, `sonata.project.root` is the folder that contains the executable. discord uses it for the default command cache file (`<root>/.commands.lock.json`), so the cache sits next to the program. Relative paths you open yourself, such as a `token` file, resolve against the working directory.

## Reducing size

If your manifest sets `build.libraries`, only the listed `@sonata/*` libraries are included. Sonata adds the libraries that discord and httpx declare, so you only need to list what your own code uses. See [Installation](installation.md#restricting-libraries).
