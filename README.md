# viam-agentic-kit

Portable instructions and packaging that help an existing AI assistant use the [Viam MCP server](https://docs.viam.com/reference/mcp/) well.

The MCP server is part of the Viam app and is served at `https://app.viam.com/mcp`. It exposes the Viam fleet API as MCP tools, and every tool runs with the signed-in user's own Viam permissions. This repository contains no server code and no credentials. It adds a skill that says how to use the tools for one task, the connection settings in the Agent Plugins format, and one-entry marketplace catalogs so each client can discover and install the kit.

## Contents

| Path | Format | Read by |
| --- | --- | --- |
| `plugin.json` | [Agent Plugins 1.0.0](https://agent-plugins.org/specification) manifest | Codex, ChatGPT, Cursor |
| `mcp.json` | Agent Plugins MCP configuration: `https://app.viam.com/mcp` over Streamable HTTP, no headers, no secrets | Codex, ChatGPT, Cursor; Claude Code through its catalog entry |
| `skills/obstacle-setup/SKILL.md` | [Agent Skills](https://agentskills.io/specification) | Every client |
| `agents/obstacle-planner.md` | [Claude Code subagent](https://code.claude.com/docs/en/sub-agents), `@agent-viam-agentic-kit:obstacle-planner` | Claude Code and Cowork (Cowork install untested, see Claude); Cursor documents agents only for its own plugin format, though its 3.24.12 loader also scans `agents/` |
| `.agents/plugins/marketplace.json` | Codex marketplace catalog | Codex CLI, ChatGPT desktop app, ChatGPT workspace import |
| `.claude-plugin/marketplace.json` | Claude Code marketplace catalog | Claude Code; also accepted by Codex and ChatGPT as a Claude-compatible fallback; claude.ai marketplace import untested (see Claude) |
| `.cursor-plugin/marketplace.json` | Cursor marketplace catalog | Cursor team marketplaces |

The plugin itself is the Agent Plugins layout at the repository root, one manifest, one MCP file, and one skill, plus one subagent in Claude Code's format. The three catalogs exist because the Agent Plugins specification leaves marketplaces out of scope and each client has its own file name. Codex and ChatGPT also accept the Claude Code catalog as a fallback; with both present, Codex reads `.agents/plugins/marketplace.json` (verified), and OpenAI does not document which one a ChatGPT workspace import uses. Each catalog names the marketplace `viam` and lists the single plugin `viam-agentic-kit` with the repository root as its source, so the result is the same either way. There are no per-client copies of the skill, no hooks, or scripts.

The skill, `obstacle-setup`, sets up motion-planning obstacles for an arm: the table, pick and place markers, walls and other boxes, and frame troubleshooting. It creates and changes machine configuration through the server's write tools, which save immediately with no undo, so it tells the assistant to confirm the machine and the numbers with the user first. Clients prompt before each write tool call because the server does not mark those tools read-only.

## Authentication

The server accepts only OAuth access tokens issued for the Viam MCP application, sent as a bearer token. Viam API keys and app.viam.com browser sessions are rejected.

Clients discover the flow on their own. An unauthenticated request gets a `401` whose `WWW-Authenticate` header points at `https://app.viam.com/.well-known/oauth-protected-resource/mcp`, and the authorization server metadata at `https://app.viam.com/.well-known/oauth-authorization-server/mcp` advertises dynamic client registration, PKCE with S256, and the `openid` and `offline_access` scopes. In practice, the first use opens a browser window to sign in to Viam and approve access, and the client refreshes the token afterwards.

Two things must be true for tool calls to succeed:

- The signed-in Viam account has access to the organizations, locations, and machines in question. The server grants nothing beyond that.
- The account has accepted Viam's current terms of service. Until it has, every tool returns an error that says to log in at https://app.viam.com and accept them.

## Marketplace

Each catalog is the smallest file its client accepts. The Codex catalog is a name and one entry. The Claude Code catalog adds the `owner` field Claude Code requires and points its entry at `mcp.json`, because Claude Code reads neither the root manifest nor the standard MCP file and otherwise would install the skill without the server. The Cursor catalog has the same shape as Claude Code's without that pointer. Claude Code and Codex install the plugin as `viam-agentic-kit@viam`; ChatGPT and Cursor pick it by name in their UI.

Register a catalog from a local checkout or from GitHub. The repository is `viamrobotics/viam-agentic-kit`; its visibility is internal, so a client needs GitHub access that can read it. The GitHub forms below read the default branch, which does not hold this kit until it is merged. Until then, name the branch where the client allows it: `--ref <branch>` for Codex, `#<branch>` for Claude Code, and the **Branch, tag, or commit** field of a ChatGPT workspace import.

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

Workspace: an admin opens **Admin > Plugins**, selects **Add**, then **Import marketplace**, and enters the repository URL `https://github.com/viamrobotics/viam-agentic-kit` in **Source**, with **Path** left empty and, until the kit is merged, this branch in **Branch, tag, or commit**. The import accepts `.agents/plugins/marketplace.json` or `.claude-plugin/marketplace.json`; OpenAI documents no precedence between them, and both list the same single plugin. The imported plugin is again Desktop only.

Web, tools only: open **Plugins**, select the plus button, then **Add custom MCP server**. Enter a name, the server URL `https://app.viam.com/mcp`, and choose **OAuth** for authentication. Create it as a plugin, install it, and mention it with `@` in a prompt. Adding a custom MCP server needs developer mode. OpenAI's Help Center documents it for Business and Enterprise/Edu workspaces on ChatGPT web, where an admin turns it on under **Workspace Settings > Permissions & Roles** and each eligible user then enables it in their own settings; Pro users can enable it themselves but can connect only MCP servers with read/fetch permissions, which this server, with its write tools, may not satisfy. See [Developer mode and MCP apps in ChatGPT](https://help.openai.com/en/articles/12584461-developer-mode-and-mcp-apps-in-chatgpt).

None of the ChatGPT flows was run here.

### Claude

Claude on the web and in the desktop app is not expected to install this plugin. Its **Upload plugin** route requires a Claude manifest at `.claude-plugin/plugin.json`, and Anthropic's directory lists a plugin without one for Claude Code only, not for claude.ai or Cowork; this kit keeps the plugin in the Agent Plugins layout only. Adding this repository under **Customize > Plugins > Add > Add marketplace** was not tried. Claude gets the same tools and the same skill in two steps instead:

Tools: on a Free, Pro, or Max plan, go to **Customize > Connectors**, click **Add custom connector**, and enter `https://app.viam.com/mcp` as the server URL. If the dialog asks, choose **Sign in now** for authentication and **Register automatically** for the OAuth client, then click **Add**. Sign in to Viam when prompted. On a Team or Enterprise plan, an Owner adds the connector under **Organization settings > Connectors** (**Add**, then **Custom**), after which each member opens **Customize > Connectors** and clicks **Connect**. Free accounts can add one custom connector.

Skill: zip the `skills/obstacle-setup` folder so that the folder is at the top level of the archive, go to **Customize > Skills**, click **+**, then **+ Create skill**, choose **Upload a skill**, and select the ZIP. Custom skills require code execution to be enabled under **Settings > Capabilities** (or by an Owner under **Organization settings > Plugins & skills** on Team and Enterprise). Neither Claude flow was run here.

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

Inside a session the skill is `/viam-agentic-kit:obstacle-setup` and `/mcp` lists `plugin:viam-agentic-kit:viam`. The browser sign-in was not run here. If your claude.ai account already has the Viam connector from the Claude section and you are signed in to Claude Code with that account rather than an API key, the connector is available in Claude Code as well, under its claude.ai name.

### Cursor

Put the checkout in Cursor's local plugin folder and reload:

```sh
git clone https://github.com/viamrobotics/viam-agentic-kit.git ~/.cursor/plugins/local/viam-agentic-kit
```

Then restart Cursor or run **Developer: Reload Window**. Cursor reads the root `plugin.json`, the `skills/` directory, and `mcp.json`. Open **Customize** in the sidebar to confirm the `obstacle-setup` skill and the `viam` MCP server are listed, and sign in to Viam from the server's entry before using the tools; Cursor supports OAuth for remote servers, but this sign-in was not run here. The skill is `/obstacle-setup` in Agent chat. On Teams and Enterprise plans, admins control local plugins with **Allow Local Plugin Imports** under **Dashboard > Settings > Security & Identity > Marketplace and Plugins**; it is off by default on Enterprise.

Distributing this kit through a Cursor marketplace needs a team marketplace, a Teams and Enterprise feature; Cursor's public marketplace lists only Cursor-reviewed official plugins. Cursor's docs say team marketplaces can distribute Agent Plugins, and the path is the Dashboard under **Plugins & MCPs**, where an admin uses **Import from Repo** with this repository's URL (GitHub, GitLab, Bitbucket, and Azure DevOps are accepted), after which members install from **Customize**. The kit ships `.cursor-plugin/marketplace.json` for that import, with one entry whose source is the repository root. Cursor's reference resolves an entry by looking for a `.cursor-plugin/plugin.json` in the entry's directory and otherwise falling back to folder-based discovery, but shows no example of an entry pointing at an Agent Plugins root manifest, so treat the import as untested; if it does not index the plugin, the local plugin folder above is the documented alternative. No Cursor flow was run here.

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
| Codex CLI 0.162.0 | `.agents/plugins/marketplace.json` | marketplace add from a local path and from GitHub (`--ref` to this branch), plugin install, skill listed as `viam-agentic-kit:obstacle-setup` in the rendered session prompt, `viam` listed by `codex mcp list`, OAuth discovery |
| ChatGPT | `.agents/plugins/marketplace.json` | not run; follows OpenAI's plugin management and custom MCP server documentation |
| Claude (claude.ai) | none; connector and skill upload instead | not run; follows Claude's connector and skills documentation |
| Claude Code 2.1.296 | `.claude-plugin/marketplace.json` | `claude plugin validate` passes, marketplace add from a local path and from GitHub (`#branch`), the one-step `--marketplace` install, plugin install, inventory lists the skill and the `viam` server, OAuth discovery for `plugin:viam-agentic-kit:viam` |
| Cursor | `.cursor-plugin/marketplace.json` | not run; the catalog follows Cursor's documented shape, and whether its entry resolves to a root Agent Plugins manifest is undocumented |

`plugin.json` and `mcp.json` validate against the Agent Plugins 1.0.0 JSON Schemas. `claude plugin validate` passes on the repository with one warning, that the catalog has no description, which is omitted to keep it thin. The skill passes the Agent Skills reference validator (`agentskills validate`). The endpoint's tool catalog was confirmed live through an authenticated claude.ai custom connector: `list_organizations` and `list_machines` returned data, and no Viam resource was changed.

## Keeping the kit in step with the server

Tool names, inputs, and outputs are defined by the server. The skill is adapted from the Viam app's assisted-setup obstacle reference, with its tool names mapped to the MCP server's `read_machine_config`, `add_machine_config_item`, and `update_machine_config_item`. When the server changes, update the skill; the full catalog is in the [MCP server reference](https://docs.viam.com/reference/mcp/). The three catalogs change only when the marketplace name, the plugin name, or the MCP file name changes.
