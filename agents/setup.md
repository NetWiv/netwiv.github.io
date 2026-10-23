# Set up NetWiv for a coding agent

You are a coding agent helping the user set up [NetWiv](https://netwiv.github.io), a .NET desktop
framework on the OS WebView. Follow these steps in order. The newest release is
**NetWiv 0.3.0** (https://github.com/kim0522ms/NetWiv/releases/tag/v0.3.0).

Rules:

- Ask the user before downloading files, installing tools or creating a project.
  Say what you will run and why.
- Do not change the user's system settings, commit, or edit files other than
  the agent registration described below.
- NetWiv packages are not on NuGet.org. They come from the release ZIP, which is
  used as a local package source.

## 1. Check the prerequisites

| Requirement | Check |
| --- | --- |
| .NET SDK 8 or later | `dotnet --version` |
| Node.js ^20.19 or 22.12+ (TypeScript starters) | `node --version` |
| OS WebView | Windows: WebView2. Linux: GTK3 and WebKitGTK 4.1. macOS 12+ on Apple Silicon: built in |

If something is missing or too old, you may install it, but only with the
user's approval:

1. List what is missing and the exact commands you would run, from the table
   below. Ask once for the whole list; install only what the user approves.
2. If a command needs `sudo` or an administrator prompt you cannot answer,
   give the command to the user to run instead.
3. After installing, open a new shell if needed so `PATH` picks up the tools,
   and rerun the checks. If the user declines, give the official install link
   and stop until they have installed it.

| OS | .NET SDK | Node.js | WebView |
| --- | --- | --- | --- |
| Windows | `winget install Microsoft.DotNet.SDK.8` | `winget install OpenJS.NodeJS.LTS` | `winget install Microsoft.EdgeWebView2Runtime` (usually present) |
| macOS | `brew install --cask dotnet-sdk` | `brew install node` | Built in |
| Ubuntu, Debian | `sudo apt-get install -y dotnet-sdk-8.0` | The distribution's Node.js is often too old; use the user's version manager (for example `nvm install 22`) or ask | `sudo apt-get install -y libgtk-3-0 libwebkit2gtk-4.1-0` |
| Fedora | `sudo dnf install -y dotnet-sdk-8.0` | `sudo dnf install -y nodejs`, if it meets the version | `sudo dnf install -y gtk3 webkit2gtk4.1` |

Use a package manager the user already has; do not install Homebrew, winget or
a Node version manager without asking separately. Prefer a version manager the
user already uses over the system Node.js.

NativeAOT publishing (`--aot`) needs more, so offer it only when the user wants
NativeAOT: the Visual Studio C++ build tools and Windows SDK on Windows, Xcode
Command Line Tools (`xcode-select --install`) on macOS, and `clang` with
`zlib1g-dev` (Debian, Ubuntu) or `zlib-devel` (Fedora) on Linux.

## 2. Find where the user is

- **A NetWiv project**: the folder has `.config/dotnet-tools.json` listing
  `netwiv.cli`. Go to step 5.
- **NetWiv CLI installed, no project**: `netwiv --version` works. Go to step 4.
- **Nothing yet**: go to step 3.

## 3. Install the CLI from the release bundle

Download both files from the release into a new folder:

- https://github.com/kim0522ms/NetWiv/releases/download/v0.3.0/NetWiv-0.3.0-packages.zip
- https://github.com/kim0522ms/NetWiv/releases/download/v0.3.0/SHA256SUMS

Verify the ZIP's SHA-256 against `SHA256SUMS` and stop if it does not match.
The ZIP has `packages/` at its root. Extract it to a lasting folder, such as
`~/.netwiv/feeds/0.3.0` (`%USERPROFILE%\.netwiv\feeds\0.3.0` on
Windows), not a temporary or Downloads folder: each new project records the
feed's absolute path in its `NuGet.Config`, and restoring breaks if the folder
is removed. Tell the user where it is. Then install the CLI with that
`packages` folder as the source.

```sh
dotnet tool install --global NetWiv.Cli --version 0.3.0 --add-source <extracted>/packages
```

Use `update` instead of `install` when an older CLI is present. On macOS and
Linux, make sure `~/.dotnet/tools` is on `PATH`.

## 4. Create a project

Ask the user for a name and a starter. `netwiv new --list` shows the six:
React, Vue, Svelte and Angular with TypeScript, Blazor WebAssembly and Blazor
Hybrid. React is the default.

```sh
netwiv new MyNetWivApp --template react --source <extracted>/packages
```

The new project already contains `.mcp.json`. Continue in that folder.

## 5. Connect the MCP server

`netwiv mcp` is a read-only MCP server that the project's own CLI runs over
stdio. It serves the guides for the project's NetWiv version and the project's
configuration, capabilities and commands. In the project folder:

```sh
dotnet tool restore
```

Then register the server for the agent you are, unless it is already registered:

- **Claude Code**: reads `.mcp.json` at the project root:

  ```json
  {
    "mcpServers": {
      "netwiv": {
        "command": "dotnet",
        "args": ["tool", "run", "netwiv", "--", "mcp"]
      }
    }
  }
  ```

- **Codex**: add to `~/.codex/config.toml`, and start Codex from the project folder:

  ```toml
  [mcp_servers.netwiv]
  command = "dotnet"
  args = ["tool", "run", "netwiv", "--", "mcp"]
  ```

- **Cursor**: the same JSON as `.mcp.json` in `.cursor/mcp.json`.
- **Other agents**: command `dotnet`, arguments `tool run netwiv -- mcp`,
  stdio transport, started from the project folder.

## 6. Claude Code or Codex: install the NetWiv skills

Skip this step unless you are Claude Code or Codex. The NetWiv skills from
[NetWiv/skills](https://github.com/NetWiv/skills) hold NetWiv workflows that do
not change between releases and read version-specific details from the MCP
server above, so one install covers every NetWiv version.

**Claude Code.** Check `claude plugin list` first; if `netwiv@netwiv` is
installed and enabled, go on. Otherwise ask the user two things together:

1. Install the skills? If they decline, go on.
2. Where? Explain the choice:
   - **For every project (user scope)**: available in all of the user's folders.
     Nothing is added to the project.
   - **For this project only (project scope)**: written to
     `.claude/settings.json` in the project. Committing that file offers the
     same skills to everyone who opens the project in Claude Code.

   Without a project yet, only user scope applies.

Then run, from the project folder for project scope, with the same scope for
both commands:

```sh
claude plugin marketplace add NetWiv/skills --scope <user|project>
claude plugin install netwiv@netwiv --scope <user|project>
```

**Codex.** Check `codex plugin list` first; if `netwiv@netwiv` is installed
and enabled, go on. Otherwise ask whether to install the skills, and say that
Codex installs plugins for all of the user's projects; it has no per-project
scope. If they agree:

```sh
codex plugin marketplace add NetWiv/skills
codex plugin add netwiv@netwiv
```

Report what was installed and where it was recorded.

## 7. Restart and verify

Most agents load new MCP servers and plugins only at startup. If yours does,
tell the user to restart it in the project folder and to approve the `netwiv`
server when asked; the restart also loads any skills installed in step 6.

Then call `netwiv_project_info`. Report the CLI version, the project's NetWiv
package version and whether they differ. Call `netwiv_docs_read` without
arguments to confirm the guides are available.

From then on, answer NetWiv questions with `netwiv_docs_search` and
`netwiv_docs_read` rather than from memory: they match the project's version.
After the user changes C# commands and builds, `netwiv_commands` shows the
generated frontend calls.
