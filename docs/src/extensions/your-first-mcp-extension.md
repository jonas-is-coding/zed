---
title: "Tutorial: Your First MCP Server Extension"
description: "A step-by-step walkthrough of building, installing, and iterating on an MCP server extension for Zed."
---

# Tutorial: Your First MCP Server Extension {#your-first-mcp-extension}

This tutorial walks you through building a complete [MCP server extension](./mcp-extensions.md) end-to-end. By the end, you will have a working extension that exposes the official [`@modelcontextprotocol/server-everything`](https://github.com/modelcontextprotocol/servers/tree/main/src/everything) reference server inside Zed's [Agent Panel](../ai/agent-panel.md), and you will know how to iterate on it.

If you want a higher-level overview first, read [Developing Extensions](./developing-extensions.md) and [MCP Server Extensions](./mcp-extensions.md). This page is the hands-on companion to those.

## What you will build {#what-you-will-build}

A Zed extension named `my-first-mcp` that:

- Registers a context server with Zed.
- Tells Zed to install [`@modelcontextprotocol/server-everything`](https://www.npmjs.com/package/@modelcontextprotocol/server-everything) from npm using Zed's bundled Node.
- Launches the server on stdio so the Agent Panel can call its tools.

`server-everything` is Anthropic's reference MCP server. It exposes a small set of tools (`echo`, `add`, sample resources, and prompts) — enough to verify the wiring works without writing any MCP server code yourself.

## Prerequisites {#prerequisites}

You need these installed before starting:

1. **Zed** (any recent version).
2. **Rust via [rustup](https://www.rust-lang.org/tools/install)**. Rust installed via Homebrew or another package manager will not work — installing dev extensions requires rustup specifically.
3. **The `wasm32-wasip1` target**:

   ```sh
   rustup target add wasm32-wasip1
   ```

   Extensions are compiled to WebAssembly. Forgetting this step is the most common first-time error.

You do **not** need Node.js installed system-wide — Zed bundles its own Node binary that the extension API exposes via [`zed::node_binary_path()`](https://docs.rs/zed_extension_api/latest/zed_extension_api/fn.node_binary_path.html).

## The dev-install loop {#dev-install-loop}

While developing, you do not need to publish your extension to test it. Instead, run the {#action zed::InstallDevExtension} action from the command palette and select your extension's directory. Zed compiles it, loads it, and reports any errors in [`Zed.log`](../telemetry.md) ({#action zed::OpenLog}). After editing your code, re-run `zed: install dev extension` to recompile and reload — no Zed restart required.

For more verbose output, launch Zed from a terminal with the `--foreground` flag:

```sh
zed --foreground
```

`println!` and `dbg!` from your extension will appear there.

## Step 1: Create the project {#step-1-create-project}

```sh
mkdir my-first-mcp
cd my-first-mcp
git init
```

Extensions must be Git repositories. An empty repo is fine for local dev; you only need a remote when [publishing](./developing-extensions.md#publishing-your-extension).

## Step 2: `extension.toml` {#step-2-extension-toml}

Create `extension.toml` with the full manifest. Every field below is required for an MCP extension:

```toml
id             = "my-first-mcp"
name           = "My First MCP"
description    = "Walkthrough extension wrapping @modelcontextprotocol/server-everything."
version        = "0.0.1"
schema_version = 1
authors        = ["Your Name <you@example.com>"]
repository     = "https://github.com/your-username/my-first-mcp"

[context_servers.my-first-mcp]
```

A few field-by-field notes that catch first-timers:

- `id` must be unique across the [extensions registry](https://github.com/zed-industries/extensions). It cannot contain `zed` or `extension`. For local dev anything works.
- `schema_version = 1` is the current manifest schema. Setting it wrong produces a confusing parse error.
- `[context_servers.my-first-mcp]` is the registration. The key after the dot is the **context server id** that gets passed back to your Rust code. It does not need to match the extension `id`, but matching keeps things simple.

## Step 3: `Cargo.toml` {#step-3-cargo-toml}

Procedural extensions are Rust crates compiled to WebAssembly:

```toml
[package]
name    = "my-first-mcp"
version = "0.0.1"
edition = "2021"
publish = false

[lib]
crate-type = ["cdylib"]

[dependencies]
zed_extension_api = "0.7.0"
```

`crate-type = ["cdylib"]` is what produces the `.wasm` artifact. `publish = false` keeps `cargo publish` from accidentally pushing your extension crate to crates.io. Use the latest [`zed_extension_api`](https://crates.io/crates/zed_extension_api) version that is still listed as compatible with your target Zed in the [extension API README](https://github.com/zed-industries/zed/blob/main/crates/extension_api#compatible-zed-versions).

## Step 4: `src/lib.rs` {#step-4-lib-rs}

This is the whole extension:

```rust
use zed_extension_api::{
    self as zed, Command, ContextServerId, Project, Result,
};

const PACKAGE_NAME: &str = "@modelcontextprotocol/server-everything";
const SERVER_PATH: &str = "node_modules/@modelcontextprotocol/server-everything/dist/index.js";

struct MyFirstMcpExtension;

impl zed::Extension for MyFirstMcpExtension {
    fn new() -> Self {
        Self
    }

    fn context_server_command(
        &mut self,
        _context_server_id: &ContextServerId,
        _project: &Project,
    ) -> Result<Command> {
        let latest_version = zed::npm_package_latest_version(PACKAGE_NAME)?;
        let installed_version = zed::npm_package_installed_version(PACKAGE_NAME)?;
        if installed_version.as_deref() != Some(latest_version.as_ref()) {
            zed::npm_install_package(PACKAGE_NAME, &latest_version)?;
        }

        let node_path = zed::node_binary_path()?;
        let server_path = std::env::current_dir()
            .map_err(|e| e.to_string())?
            .join(SERVER_PATH)
            .to_string_lossy()
            .to_string();

        Ok(Command {
            command: node_path,
            args: vec![server_path],
            env: vec![],
        })
    }
}

zed::register_extension!(MyFirstMcpExtension);
```

What this does, line by line:

- `zed::npm_package_latest_version` and `zed::npm_install_package` are part of the extension API. Zed downloads and caches the package into a sandboxed `node_modules/` next to your extension. You do not call out to the user's `npm` or `npx`.
- `zed::node_binary_path()` returns the Node binary that ships with Zed. Using it means your extension works even if the user has no system Node installed.
- The returned `Command` is what Zed runs to start the MCP server. `args: vec![server_path]` is equivalent to `node node_modules/.../dist/index.js`. The server speaks MCP over stdio; Zed handles the rest.

You do **not** need a `[[capabilities]]` entry in `extension.toml` for this extension. The capability system is only for extensions that themselves call `zed::process::Command::new(...)` from inside the WASM module — for example, to validate a user-provided binary. The MCP server process launched via the returned `Command` is started by Zed itself and does not need a declared capability.

Your directory should now look like:

```text
my-first-mcp/
├── Cargo.toml
├── extension.toml
└── src/
    └── lib.rs
```

## Step 5: Install as a dev extension {#step-5-install-dev-extension}

In Zed, open the command palette and run **`zed: install dev extension`** ({#action zed::InstallDevExtension}). Select the `my-first-mcp` directory. Zed compiles your crate to WASM and loads it. The first compile takes longer because Cargo fetches dependencies; subsequent reloads are fast.

If anything goes wrong, open `Zed.log` ({#action zed::OpenLog}) — compile errors and runtime panics from the extension show up there.

## Step 6: Use it from the Agent Panel {#step-6-use-it}

Open the [Agent Panel](../ai/agent-panel.md) and start a new thread. The `my-first-mcp` server should appear in the list of available context servers. Ask the agent to use one of `server-everything`'s tools — for example:

> Use the `echo` tool to repeat the string "hello from my first extension".

The agent will call the tool through your extension and show the result. That round-trip — Zed → your WASM extension → Node + MCP server → tool result back to the agent — is the same one that every published MCP extension goes through.

## Step 7: Iterate {#step-7-iterate}

Try changing the extension and reloading:

1. Edit `src/lib.rs` — for example, add `eprintln!("starting server");` just before the `Ok(Command { … })` return.
2. Run `zed: install dev extension` again and re-select the directory. Zed recompiles and reloads.
3. Restart Zed with `zed --foreground` from a terminal. Your `eprintln!` output appears in the terminal when the extension starts the server.

This edit-install-observe loop is how all extensions are developed. Keep `Zed.log` open on the side; most non-obvious failures (missing fields, schema mismatches, server crashes) leave a trace there.

## Going further {#going-further}

- **Wrap a different MCP server.** Any [official Anthropic server](https://github.com/modelcontextprotocol/servers) with an npm package works with the same skeleton — change `PACKAGE_NAME` and `SERVER_PATH`. For Python or Go-based servers, you ship a [downloadable binary](https://docs.rs/zed_extension_api/latest/zed_extension_api/fn.download_file.html) and return a `Command` pointing at it instead.
- **Take user settings.** Most published extensions read configuration from the user's [Zed `settings.json`](../configuring-zed.md) via [`ContextServerSettings::for_project`](https://docs.rs/zed_extension_api/latest/zed_extension_api/settings/struct.ContextServerSettings.html), then forward values into `Command.env`. The [`postgres-context-server`](https://github.com/zed-extensions/postgres-context-server) extension is a complete reference.
- **Write your own MCP server.** See the [Model Context Protocol documentation](https://modelcontextprotocol.io) and SDKs.
- **Publish.** When your extension is ready, follow [Publishing your extension](./developing-extensions.md#publishing-your-extension).

## Troubleshooting {#troubleshooting}

**`error: target 'wasm32-wasip1' not found`**
Run `rustup target add wasm32-wasip1`. The compile target is required for every Zed extension.

**`failed to install dev extension` immediately on selection**
Almost always means Rust was not installed via rustup. Re-install via [rustup.rs](https://www.rust-lang.org/tools/install) and remove any `rust` Homebrew formula.

**The context server does not appear in the Agent Panel**
Check `Zed.log` ({#action zed::OpenLog}). Common causes are a missing or mistyped field in `extension.toml` (`id`, `schema_version`), a panic during `npm_install_package`, or the server process exiting immediately. Re-run with `zed --foreground` to see the server's stderr.

**Extension loads, but tool calls hang or fail**
The Node entry point path is wrong. Verify `SERVER_PATH` against the package's `bin` field in npm — for `@modelcontextprotocol/server-*` packages it is consistently `dist/index.js`, but third-party packages vary.

**Changes do not take effect after editing `lib.rs`**
Re-run `zed: install dev extension`. Editing the source alone is not enough; the WASM artifact must be rebuilt.
