# viam-agentic-kit

Portable instructions and packaging that help an existing AI assistant use the [Viam MCP server](https://docs.viam.com/reference/mcp/) well.

The MCP server is part of the Viam app and is served at `https://app.viam.com/mcp`. It exposes the Viam fleet API as MCP tools, and every tool runs with the signed-in user's own Viam permissions. This repository contains no server code and no credentials. It adds a skill that says how to use the tools for one task, the connection settings in the formats plugin-aware clients load, and a one-entry marketplace catalog so each client can discover and install the kit.

## Contents

| Path | Format | Read by |
| --- | --- | --- |
| `plugin.json` | [Agent Plugins 1.0.0](https://agent-plugins.org/specification) manifest | Codex, ChatGPT, Cursor |
| `.claude-plugin/plugin.json` | Claude plugin manifest, same values | Claude Code and claude.ai, which do not read the root manifest |
| `mcp.json` | Agent Plugins MCP configuration: `https://app.viam.com/mcp` over Streamable HTTP, no headers, no secrets | Codex, ChatGPT, Cursor |
| `.mcp.json` | Claude MCP configuration, same server | Claude Code and claude.ai, which do not read `mcp.json` |
| `.claude-plugin/marketplace.json` | Marketplace catalog with one entry: the plugin at the repository root | Claude Code, claude.ai, Codex, ChatGPT |
| `skills/obstacle-setup/SKILL.md` | [Agent Skills](https://agentskills.io/specification) | Every client |

The two manifests and the two MCP files carry the same values because the clients read different file names. Change them together. The skill is the only instruction set; there are no per-client copies, subagents, hooks, or scripts.

The skill, `obstacle-setup`, sets up motion-planning obstacles for an arm: the table, pick and place markers, walls and other boxes, and frame troubleshooting. It creates and changes machine configuration through the server's write tools, which save immediately with no undo, so it tells the assistant to confirm the machine and the numbers with the user first. Clients prompt before each write tool call because the server does not mark those tools read-only.

## Authentication

The server accepts only OAuth access tokens issued for the Viam MCP application, sent as a bearer token. Viam API keys and app.viam.com browser sessions are rejected.

Clients discover the flow on their own. An unauthenticated request gets a `401` whose `WWW-Authenticate` header points at `https://app.viam.com/.well-known/oauth-protected-resource/mcp`, and the authorization server metadata at `https://app.viam.com/.well-known/oauth-authorization-server/mcp` advertises dynamic client registration, PKCE with S256, and the `openid` and `offline_access` scopes. In practice, the first use opens a browser window to sign in to Viam and approve access, and the client refreshes the token afterwards.

Two things must be true for tool calls to succeed:

- The signed-in Viam account has access to the organizations, locations, and machines in question. The server grants nothing beyond that.
- The account has accepted Viam's current terms of service. Until it has, every tool returns an error that says to log in at https://app.viam.com and accept them.

## Marketplace

The catalog is `.claude-plugin/marketplace.json`. It names the marketplace `viam` and lists one plugin, `viam-agentic-kit`, whose source is the repository root, so the plugin id is `viam-agentic-kit@viam` in every client that installs from it.

Claude Code and claude.ai read this file natively. Codex and the ChatGPT desktop app read it as a Claude-compatible catalog, and a ChatGPT workspace import accepts it, so one file serves four clients. Cursor reads only `.cursor-plugin/marketplace.json`, which this repository does not ship (see Cursor below).

Register the catalog from a local checkout or from GitHub. The repository is `viamrobotics/viam-agentic-kit`; its visibility is internal, so a client needs GitHub access that can read it.

## Install

The sections below assume a checkout at `./viam-agentic-kit` where a local path is used.

```sh
git clone https://github.com/viamrobotics/viam-agentic-kit.git
```

### Codex CLI

Register the catalog, install the plugin, and sign in to the `viam` server it declares:

```sh
codex plugin marketplace add ./viam-agentic-kit        # or: codex plugin marketplace add viamrobotics/viam-agentic-kit
codex plugin add viam-agentic-kit@viam
codex mcp login viam
```

`codex mcp list` then lists `viam` as enabled; its Auth column reads `Not logged in` until the browser sign-in completes, which was not run here. Codex prefixes a plugin's skills with the plugin name, so in a session `/skills` lists `viam-agentic-kit:obstacle-setup` and `/mcp` lists `viam`. Invoke the skill by mentioning it:

```sh
codex '$viam-agentic-kit:obstacle-setup Add a table obstacle to my machine "rover": 1000 by 700 by 25 mm, arm base centered.'
```

To remove the kit, run `codex plugin remove viam-agentic-kit@viam` and then `codex plugin marketplace remove viam`.

### ChatGPT

ChatGPT gets the plugin from a marketplace and the tools from the plugin's MCP server.

Desktop app: OpenAI documents the desktop app as the place to install and test a local plugin, from a marketplace registered with `codex plugin marketplace add` as in the Codex section. A plugin that declares an MCP server is marked **Desktop only**, so it does not load in ChatGPT on the web or on mobile.

Workspace: an admin opens **Admin > Plugins**, selects **Add**, then **Import marketplace**, and enters the repository URL `https://github.com/viamrobotics/viam-agentic-kit` with **Path** left empty. OpenAI's import accepts a Claude-compatible catalog at `.claude-plugin/marketplace.json`. The imported plugin is again Desktop only.

Web, tools only: open **Plugins**, select the plus button, then **Add custom MCP server**. Enter a name, the server URL `https://app.viam.com/mcp`, and choose **OAuth** for authentication. Create it as a plugin, install it, and mention it with `@` in a prompt. Adding a custom MCP server needs developer mode. OpenAI's Help Center documents it for Business and Enterprise/Edu workspaces on ChatGPT web, where an admin turns it on under **Workspace Settings > Permissions & Roles** and each eligible user then enables it in their own settings; Pro users can enable it themselves but can connect only MCP servers with read/fetch permissions, which this server, with its write tools, may not satisfy. See [Developer mode and MCP apps in ChatGPT](https://help.openai.com/en/articles/12584461-developer-mode-and-mcp-apps-in-chatgpt).

None of the ChatGPT flows was run here.

### Claude

Claude on the web and in the desktop app installs the plugin from the marketplace. Go to **Customize > Plugins**, select **Add**, then **Add marketplace**, and enter `viamrobotics/viam-agentic-kit`. Because the repository is not public, connect your GitHub account when the dialog asks and give the Claude GitHub App access to it. Then select `viam-agentic-kit` and add it. Open the plugin's **Connectors** tab and add, then connect, the `viam` connector, which signs you in to Viam. In a chat, type `/` and pick `viam-agentic-kit:obstacle-setup`, or describe the task and let Claude load it.

On a Team or Enterprise plan, an Owner can instead sync the repository for everyone under **Organization settings > Plugins & skills** with **Add**, then **Sync from GitHub**. Organization sync requires the marketplace repository to be private or internal on github.com, which this one is, and the Claude GitHub App installed on it. Members then find the plugin under **Customize > Plugins**, and an Owner adds the connector for the organization before members connect with their own accounts.

A plugin you add on claude.ai also reaches Claude Code as a synced plugin the next time you start a session signed in to the same account. Neither claude.ai flow was run here.

### Claude Code

Register the catalog, install the plugin, and sign in to the server the plugin registers:

```sh
claude plugin marketplace add ./viam-agentic-kit       # or: claude plugin marketplace add viamrobotics/viam-agentic-kit
claude plugin install viam-agentic-kit@viam
claude mcp login plugin:viam-agentic-kit:viam           # or /mcp inside a session
```

On Claude Code 2.1.292 or later, the first two commands collapse into one:

```sh
claude plugin install viam-agentic-kit --marketplace viamrobotics/viam-agentic-kit
```

Inside a session the skill is `/viam-agentic-kit:obstacle-setup` and `/mcp` lists `plugin:viam-agentic-kit:viam`. The browser sign-in was not run here. Starting Claude Code inside the checkout itself also offers the repository's project-level `.mcp.json` server, named `viam`, for approval; decline it when the plugin is installed, or use it instead of the plugin for a one-off session.

### Cursor

Put the checkout in Cursor's local plugin folder and reload:

```sh
git clone https://github.com/viamrobotics/viam-agentic-kit.git ~/.cursor/plugins/local/viam-agentic-kit
```

Then restart Cursor or run **Developer: Reload Window**. Cursor reads the root `plugin.json`, the `skills/` directory, and `mcp.json`. Open **Customize** in the sidebar to confirm the `obstacle-setup` skill and the `viam` MCP server are listed, and sign in to Viam from the server's entry before using the tools; Cursor supports OAuth for remote servers, but this sign-in was not run here. The skill is `/obstacle-setup` in Agent chat. On Teams and Enterprise plans, admins control local plugins with **Allow Local Plugin Imports** under **Dashboard > Settings > Security & Identity > Marketplace and Plugins**; it is off by default on Enterprise.

Marketplace discovery is not available in Cursor from this repository. Cursor's catalog format is `.cursor-plugin/marketplace.json`, and its documentation resolves each entry through a `.cursor-plugin/plugin.json` manifest, which an Agent Plugins layout does not have. Cursor's docs say team marketplaces can distribute Agent Plugins, but do not say how an entry points at one, so no Cursor catalog is shipped and no Cursor flow was run.

## Try it

Ask something the skill covers, on a machine with an arm you can reconfigure:

> Add a table obstacle to my machine "rover": the table is 1000 by 700 by 25 mm and the arm base is centered on it.

The assistant should call `list_machines` with the name and `read_machine_config` to check for an existing table, then `add_machine_config_item` to create the `table` component and `update_machine_config_item` to give it a frame, and finally `read_machine_config` again to confirm the frame. Both write calls change the machine's saved configuration at once; the client asks you to approve each one.

A connectivity check that changes nothing and needs no skill:

> Which Viam organizations can I access?

That is a single call to `list_organizations`.

## Compatibility

Verified on Linux with the files in this repository. "OAuth discovery" means the client fetched the OAuth metadata, registered itself, and printed the Viam sign-in URL; the browser sign-in itself was not run.

| Client | Catalog | Verified |
| --- | --- | --- |
| Codex CLI 0.162.0 | `.claude-plugin/marketplace.json` | marketplace add from a local path, plugin install, skill listed as `viam-agentic-kit:obstacle-setup` in the rendered session prompt, `viam` listed by `codex mcp list`, OAuth discovery |
| ChatGPT | same file, via import | not run; follows OpenAI's plugin management and custom MCP server documentation |
| Claude (claude.ai) | same file | not run; follows Claude's plugin and organization sync documentation |
| Claude Code 2.1.296 | same file | `claude plugin validate --strict` passes on the repository, marketplace add from a local path, plugin install, inventory lists the skill and the `viam` server, OAuth discovery for `plugin:viam-agentic-kit:viam` |
| Cursor | none | not run; marketplace discovery not supported by this layout |

`plugin.json` and `mcp.json` validate against the Agent Plugins 1.0.0 JSON Schemas, `claude plugin validate --strict` passes on the repository (it checks the Claude manifest, the catalog, `.mcp.json`, and the skill), and the skill passes the Agent Skills reference validator (`agentskills validate`). The endpoint's tool catalog was confirmed live through an authenticated claude.ai custom connector: `list_organizations` and `list_machines` returned data, and no Viam resource was changed.

## Keeping the kit in step with the server

Tool names, inputs, and outputs are defined by the server. The skill is adapted from the Viam app's assisted-setup obstacle reference, with its tool names mapped to the MCP server's `read_machine_config`, `add_machine_config_item`, and `update_machine_config_item`. When the server changes, update the skill; the full catalog is in the [MCP server reference](https://docs.viam.com/reference/mcp/).
