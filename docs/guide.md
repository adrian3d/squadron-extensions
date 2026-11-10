# Writing a Squadron extension

An extension adds things to Squadron without rebuilding it: buttons and menu entries, tools that agents can call, panels beside the conversation, colors. It can be a single `extension.json`, or carry logic written in Rust and compiled to WebAssembly.

- [Quick start](#quick-start)
- [extension.json](#extensionjson)
- [Logic in Rust](#logic-in-rust)
- [Panels](#panels)
- [Secrets and network](#secrets-and-network)
- [Permissions and approval](#permissions-and-approval)
- [Limits](#limits)
- [Test and debug](#test-and-debug)
- [Publish](#publish)

## Quick start

1. Install Rust and the WebAssembly target:

   ```sh
   rustup target add wasm32-wasip2
   ```

2. Copy [`examples/hello`](../examples/hello) to a folder of your own, and depend on the API from git instead of the local path:

   ```toml
   [dependencies]
   squadron_extension_api = { git = "https://github.com/Jgrasp/squadron-extensions" }
   ```

3. Build:

   ```sh
   cargo build --release --target wasm32-wasip2
   ```

4. In Squadron, open Settings › Extensions (also in the Squadron menu), click **Add an extension**, then **A folder on this computer**, and choose your folder.

Squadron reloads the extension within a second each time `extension.json` or the `.wasm` file changes. A local folder comes before an installed extension with the same id, so you can work on a published extension without uninstalling it.

## extension.json

```json
{
  "id": "hello",
  "version": "1.0.0",
  "api": 1,
  "name": { "en": "Hello", "fr": "Hello" },
  "description": { "en": "Example extension written in Rust" },
  "wasm": "target/wasm32-wasip2/release/hello.wasm",
  "contributes": {
    "items": [
      { "slot": "agent.tools", "id": "count", "label": { "en": "Count" }, "icon": "plus", "action": { "run": "count" } },
      { "slot": "agent.tools", "id": "panel", "label": { "en": "Hello panel" }, "icon": "grid", "action": { "panel.open": "main" } }
    ],
    "tools": [{ "name": "clicks", "description": "Tells how many times the user clicked the Count button." }],
    "panels": [{ "id": "main", "title": { "en": "Hello" }, "page": "panel/index.html" }],
    "secrets": [{ "id": "token", "label": { "en": "GitHub token" } }],
    "theme": { "accent": "#4fa3e0" }
  },
  "permissions": { "exec": ["git"], "network": ["api.github.com"] }
}
```

Every problem in the manifest shows under the extension in Settings › Extensions, and an extension with a problem adds nothing.

### Top level

| Field | Required | Rule |
| --- | --- | --- |
| `id` | yes | Lowercase letters, digits and dashes, 40 characters at most. It prefixes everything the extension adds, so it never changes once published. |
| `version` | yes | Like `1.2.3`. |
| `api` | yes | `1`. |
| `name` | yes | A text: `{ "en": "…", "fr": "…" }`. English is required; other languages are optional and fall back to English. |
| `description` | no | A text. |
| `wasm` | no | The path of the compiled component, inside the extension folder. Needed for `run` actions, tools without `command`, and anything in [Logic in Rust](#logic-in-rust). |
| `permissions` | no | See [Permissions and approval](#permissions-and-approval). |

### contributes.items

Buttons and menu entries. Squadron draws them in its own style: an extension chooses the words, the icon and the action, never the look.

| Field | Rule |
| --- | --- |
| `slot` | `agent.tools` (the toolbar of an agent), `agent.menu` (its ⋯ menu, before Archive) or `menu.squadron` (the Squadron menu). |
| `id` | Lowercase letters, digits and dashes, unique in the extension. The item's key is `<extension>.<id>`. |
| `label` | A text. |
| `hint` | Optional text, shown under the label in menus and as a tooltip on buttons. |
| `icon` | Optional, one of `editor`, `pull`, `terminal`, `diff`, `kanban`, `remote`, `subagents`, `artifact`, `branch`, `options`, `context`, `turns`, `tasks`, `persona`, `plus`, `copy`, `folder`, `archive`, `stop`, `pin`, `grid`, `jump`, `update`, `mcp`. |
| `action` | Exactly one of the actions below. |

| Action | What a click does |
| --- | --- |
| `{ "terminal.run": "lando logs -f" }` | Runs the command in the agent's terminal and shows it. One line. The user allows the command first. |
| `{ "prompt.insert": "Review this diff…" }` | Adds the text to the message the user is writing to the agent. |
| `{ "url.open": "https://…" }` | Opens the link in the browser. `https://` only. |
| `{ "squadron": "kanban" }` | Runs a Squadron action: `newAgent`, `jump`, `kanban`, `editor`, `terminal`, `diff`, `sidebar`, `find`, `interrupt`, `back`, `forward`, `search`, `projects`, `quit`. |
| `{ "run": "count" }` | Calls `run("count", agent)` in the extension's WebAssembly. Needs `wasm`. |
| `{ "panel.open": "main" }` | Opens or closes one of the extension's panels. |

`terminal.run`, `prompt.insert` and `panel.open` need an agent, so they cannot go in `menu.squadron`.

Users move and hide extension items like built-in ones, in Settings › Layout or by asking an agent.

### contributes.tools

Tools that every agent started by Squadron can call, through the `squadron-extensions` MCP server. An agent sees the tool `hello_clicks` for the tool `clicks` of the extension `hello`. Tools appear and disappear in running sessions without a restart.

| Field | Rule |
| --- | --- |
| `name` | Lowercase letters, digits and underscores, 40 characters at most. |
| `description` | Tells the agent when to use the tool. Required. |
| `command` | Optional. A shell command run in the agent's folder; the agent receives the last 30,000 characters of its output. Without `command`, Squadron calls `call_tool` in the WebAssembly. |
| `input` | Optional JSON schema of type `object` for the tool's arguments. The WebAssembly receives them as JSON. |

Claude Code asks the user before each call, as for any tool.

### contributes.panels

| Field | Rule |
| --- | --- |
| `id` | Lowercase letters, digits and dashes. |
| `title` | A text. |
| `page` | The path of an `.html` file inside the extension folder. |

See [Panels](#panels).

### contributes.secrets

| Field | Rule |
| --- | --- |
| `id` | Lowercase letters, digits and dashes. |
| `label` | A text. |
| `hint` | Optional text, for example where to create the token. |

See [Secrets and network](#secrets-and-network).

### contributes.theme

Squadron color variables and their values, for example `{ "accent": "#4fa3e0" }`. Values are colors only: `#rgb`, `#rrggbb`, `rgb()`, `rgba()`, `hsl()`, `hsla()`.

## Logic in Rust

The WebAssembly runs inside Squadron, one instance per extension, and keeps running while its panels are closed. Implement only what you need:

```rust
use squadron_extension_api::{Agent, Extension, host, squadron_extension};

struct Hello;

impl Extension for Hello {
    fn on_event(name: String, agent: Option<Agent>, data: String) {
        if name == "agent.done" {
            if let Some(agent) = agent {
                host::notify("Hello", &format!("{} finished its turn.", agent.name));
            }
        }
    }

    fn run(action: String, agent: Option<Agent>) -> Result<(), String> {
        match action.as_str() {
            "greet" => {
                let agent = agent.ok_or("No agent to greet.")?;
                host::insert_prompt(&agent.id, &format!("Hello {}!", agent.name));
                Ok(())
            }
            _ => Err(format!("Unknown action {action}.")),
        }
    }

    fn call_tool(name: String, input: String, agent: Option<Agent>) -> Result<String, String> {
        Err(format!("Unknown tool {name}."))
    }
}

squadron_extension!(Hello);
```

An `Agent` has `id`, `name`, `project` and `dir`, the folder the agent works in (its worktree, or its project). An error returned by `run` shows to the user; an error returned by `call_tool` goes back to the agent.

### Events

`on_event` receives:

| Event | Agent | `data` |
| --- | --- | --- |
| `started` | none | `{}`. Sent once the extension is loaded, also after a reload. |
| `agent.done` | yes | `{}`. The agent finished its turn. |
| `agent.waiting` | yes | `{ "waiting": … }`. The agent waits for the user. |
| `panel.message` | the agent of the panel | `{ "panel": "<id>", "message": … }`. A panel called `squadron.send`. |
| `timer` | none | `{}`. The delay given to `host::set_timer` is over. API 1.1. |

### Host functions

| Function | What it does |
| --- | --- |
| `host::log(message)` | Writes a line to Squadron's log. |
| `host::notify(title, body)` | Shows a system notification. |
| `host::insert_prompt(agent, text)` | Adds text to the message the user is writing to an agent. |
| `host::set_badge(item, badge)` | Shows a short badge on one of the extension's items, or removes it with `None`. |
| `host::set_label(item, label)` | Replaces an item's label, or restores the manifest's with `None`. |
| `host::state_get(key)`, `host::state_set(key, value)` | The extension's storage, kept across restarts and updates. 1 MB in total. |
| `host::exec(program, args, dir)` | Runs a program listed in `permissions.exec`, without a shell. `dir` defaults to the home folder. Returns `code`, `stdout` and `stderr`. |
| `host::fetch(request)` | Sends an HTTPS request to a host listed in `permissions.network`. See [Secrets and network](#secrets-and-network). |
| `host::panel_post(panel, message)` | Sends a JSON message to one of the extension's open panels. |
| `host::agents()` | The agents open in Squadron, archived ones aside. Needs `agents: ["read"]`. API 1.1. |
| `host::archive_agent(agent)` | Archives an agent, as the user would. Refused while it works, waits for the user, is pinned, remote or shown on screen: try again later. Needs `agents: ["archive"]`. API 1.1. |
| `host::set_timer(seconds)` | Sends the event `timer` after `seconds`, 10 at least, and cancels the one pending. A timer does not survive a reload or a restart: set it again on `started`. API 1.1. |

The WebAssembly has no access to files, the network or environment variables other than through these functions.

## Panels

A panel is an HTML page of the extension, shown beside the conversation like the terminal or the diff. The user chooses where it goes (beside, below, full screen) and Squadron remembers it.

The page runs in an isolated frame:

- it can use its own HTML, CSS, JavaScript, images and fonts from the extension folder;
- it cannot reach the network, Squadron's window, Tauri, pop-ups (`alert`, `confirm`) or `localStorage`;
- keep its state in the WebAssembly with `host::state_set`.

Squadron adds a small script at the top of every page, which gives `window.squadron`:

```js
squadron.on((message) => { /* a value sent by host::panel_post */ })
squadron.send({ type: 'ready' }) // the WebAssembly receives panel.message
```

A message sent before the page has loaded is lost. Let the page speak first: send `ready` when it loads, and answer it from `on_event`.

```rust
fn on_event(name: String, _agent: Option<Agent>, data: String) {
    if name == "panel.message" && data.contains(r#""type":"ready""#) {
        host::panel_post("main", r#"{"count":3}"#);
    }
}
```

### Theme

Squadron sets these CSS variables in the page, at load and each time its theme changes: `--bg-main`, `--surface-1`, `--surface-2`, `--surface-3`, `--border`, `--border-focus`, `--text`, `--text-strong`, `--text-muted`, `--text-dim`, `--ink`, `--accent`, `--accent-link`, `--ok-text`, `--err-text`, `--sans`, `--mono`. This list is part of the API and will not shrink.

For a page that looks like Squadron with no effort, add its stylesheet. It styles the background, text, links, headings, `button`, `button.primary`, inputs and `.muted`:

```html
<link rel="stylesheet" href="/_squadron/kit.css" />
```

## Secrets and network

Declare the secrets in `contributes.secrets`. The user fills them in the extension's details in Settings › Extensions, and Squadron keeps them in the system keyring. Neither the WebAssembly nor the panel can read them.

To use a secret in a request, write `{{id}}` in a header value. Squadron replaces it, and replaces the secret with `[secret]` if it comes back in the response:

```rust
use squadron_extension_api::Request;

let answer = host::fetch(&Request {
    method: "GET".into(),
    url: "https://api.github.com/user".into(),
    headers: vec![
        ("Authorization".into(), "Bearer {{token}}".into()),
        ("User-Agent".into(), "my-extension".into()),
    ],
    body: None,
})?;
```

`host::fetch`:

- only `https://`, only to a host listed in `permissions.network`;
- redirects to another host are not followed: you get the 3xx answer;
- `{{id}}` only works in header values, not in the URL or the body;
- an unset secret makes the request fail with a message asking the user to set it.

A tool with a `command` receives each secret that is set as an environment variable `SQUADRON_<ID>`, uppercase with dashes turned into underscores: `SQUADRON_TOKEN` for `token`, `SQUADRON_API_KEY` for `api-key`. `host::exec` and `terminal.run` never receive secrets.

## Permissions and approval

```json
"permissions": { "exec": ["git"], "network": ["api.github.com"], "agents": ["read", "archive"] }
```

- `exec`: program names that `host::exec` may run, with any arguments.
- `network`: exact host names that `host::fetch` may reach.
- `agents`: `read` lets `host::agents` list the open agents, `archive` lets `host::archive_agent` archive them. API 1.1.

When an extension arrives or changes, Squadron shows the user what it asks for: the programs of `exec`, the hosts of `network`, the access to `agents`, every `terminal.run` command and every tool `command`. Until the user allows them, its buttons ask first, its tools stay hidden from agents, `host::exec` and `host::fetch` fail. Any new or changed command or host asks again.

Allowed commands run with the user's rights. Ask only for what you need, and say why in your description.

## Limits

| Limit | Value |
| --- | --- |
| Computation per call | 5 billion units of fuel; the call stops beyond it. |
| Memory | 64 MB. |
| State | 1 MB. |
| `host::exec`, tool commands | 120 s; tool commands return the last 30,000 characters. |
| `host::fetch` | 30 s, 8 MB of response. |
| `host::set_timer` | 10 s at least. |
| Answer to a click or a tool | 150 s. |
| Manifest | 256 KB. |
| Published archive | 50 MB, 2,000 files. |

An extension that crashes or passes a limit is restarted with a fresh instance, keeping its state, and its error shows in Settings › Extensions.

An extension built against a newer API than the installed Squadron fails to load: one that calls a function marked API 1.1 needs a Squadron that serves it. An extension built against an older API keeps working. Keep the API version you build against in step with the Squadron versions you support.

## Test and debug

- Load the folder with **A folder on this computer**; each build reloads it.
- Errors in the manifest, a missing `.wasm` file and crashes show on the extension's card (badge **Error**) and in its details.
- `host::log` writes to Squadron's log.
- An item's action error (from `run`) shows in a dialog.
- To try a tool, ask an agent to call `<extension>_<tool>`; agents already running see it too.

## Publish

Extensions are published from their own public GitHub repository, listed in [`registry.json`](../registry.json):

1. Your extension needs, at the commit you publish: its `extension.json`, a `Cargo.lock` if it has Rust code (CI builds with `--locked`), and a license.
2. Open a pull request that adds one entry:

   ```json
   "my-extension": {
     "repository": "https://github.com/you/my-extension",
     "commit": "<full sha of the commit to publish>",
     "path": ".",
     "version": "1.0.0"
   }
   ```

   `path` is the folder of `extension.json` in the repository, and `version` must match it at that commit.
3. CI checks the entry and builds the extension from that commit. Once the pull request is reviewed and merged, CI publishes the archive as a release of this repository and adds it to `index.json`, with its sha256.

The archive holds `extension.json`, the `.wasm` file and every file of the folder that is not Rust source (`src/`, `Cargo.toml`, `Cargo.lock`, `build.rs`). Squadron installs exactly what CI built from the reviewed commit.

To publish an update, open a pull request that changes `commit` and raises `version`. Users get a notification and update in one click; a new command or host asks for their approval again.

One pull request adds or updates one extension.

### Your own source

Users can add other sources than the official one, in **Add an extension › Another source**. A source is a GitHub repository organized like this one: fork it, enable GitHub Actions in the fork, empty `registry.json` and `index.json` (`{ "api": 1, "extensions": [] }`), and publish your extensions the same way. Squadron reads `index.json` from the `main` branch and downloads archives only from the releases of that repository. Its extensions show as **Imported** and are not reviewed for Squadron; an extension updates only from the source it was installed from.
