# viam-agentic-kit

Portable instructions and packaging that help an existing AI assistant use the [Viam MCP server](https://docs.viam.com/reference/mcp/) well.

The MCP server is part of the Viam app and is served at `https://app.viam.com/mcp`. It exposes the Viam fleet API as MCP tools, and every tool runs with the signed-in user's own Viam permissions. This repository contains no server code and no credentials. It adds the two things an assistant needs besides the tools themselves: a skill that says how to use them for a common task, and the connection settings in a format that plugin-aware clients can load.

## Contents

| Path | Format | Purpose |
| --- | --- | --- |
| `plugin.json` | [Agent Plugins 1.0.0](https://agent-plugins.org/specification) manifest | Identifies the kit to clients that load plugins. |
| `mcp.json` | Agent Plugins MCP configuration | Points at `https://app.viam.com/mcp` over Streamable HTTP. No headers and no secrets: authentication is OAuth, handled by the client. |
| `skills/viam-help/SKILL.md` | [Agent Skills](https://agentskills.io/specification) | A read-only machine check-up: find a machine, report whether it is online, what it runs, how it is configured, and what looks wrong. |
| `.agents/plugins/marketplace.json` | Codex marketplace catalog | A one-entry catalog so `codex plugin` can install the kit from a local checkout, and so a ChatGPT workspace admin can import the repository as a marketplace. Other clients ignore it. |

The skill is the only instruction set. There are no per-client copies, subagents, hooks, or scripts.

## Authentication

The server accepts only OAuth access tokens issued for the Viam MCP application, sent as a bearer token. Viam API keys and app.viam.com browser sessions are rejected.

Clients discover the flow on their own. An unauthenticated request gets a `401` whose `WWW-Authenticate` header points at `https://app.viam.com/.well-known/oauth-protected-resource/mcp`, and the authorization server metadata at `https://app.viam.com/.well-known/oauth-authorization-server/mcp` advertises dynamic client registration, PKCE with S256, and the `openid` and `offline_access` scopes. In practice, the first use opens a browser window to sign in to Viam and approve access, and the client refreshes the token afterwards.

Two things must be true for tool calls to succeed:

- The signed-in Viam account has access to the organizations, locations, and machines in question. The server grants nothing beyond that.
- The account has accepted Viam's current terms of service. Until it has, every tool returns an error that says to log in at https://app.viam.com and accept them.

## Install

Clone the kit first. The sections below assume the checkout is at `./viam-agentic-kit`.

```sh
git clone https://github.com/viamrobotics/viam-agentic-kit.git
```

### Codex CLI

Register the checkout as a local marketplace, install the plugin from it, and sign in to the `viam` server the plugin declares:

```sh
codex plugin marketplace add ./viam-agentic-kit
codex plugin add viam-agentic-kit@viam
codex mcp login viam
```

`codex mcp list` then lists `viam` as enabled; its Auth column reads `Not logged in` until the browser sign-in completes, which was not run here. Codex prefixes a plugin's skills with the plugin name, so in a session `/skills` lists `viam-agentic-kit:viam-help` and `/mcp` lists `viam`. Invoke the skill by mentioning it:

```sh
codex '$viam-agentic-kit:viam-help Is my machine "rover" online, and what is configured on it?'
```

To check the connection without the skill:

```sh
codex exec 'Which Viam organizations can I access?'
```

To remove the kit, run `codex plugin remove viam-agentic-kit@viam` and then `codex plugin marketplace remove viam`.

### Claude

Claude on the web and in the desktop app gets the tools from a custom connector. On a Free, Pro, or Max plan, go to **Customize > Connectors**, click **Add custom connector**, and enter `https://app.viam.com/mcp` as the server URL. If the dialog asks, choose **Sign in now** for authentication and **Register automatically** for the OAuth client, then click **Add**. Sign in to Viam when prompted. On a Team or Enterprise plan, an Owner adds the connector under **Organization settings > Connectors** (**Add**, then **Custom**), after which each member opens **Customize > Connectors** and clicks **Connect**. Free accounts can add one custom connector.

The skill is uploaded separately. Zip the `skills/viam-help` folder so that the folder is at the top level of the archive, go to **Customize > Skills**, click **+**, then **+ Create skill**, choose **Upload a skill**, and select the ZIP. Custom skills require code execution to be enabled under **Settings > Capabilities** (or by an Owner under **Organization settings > Plugins & skills** on Team and Enterprise). The upload was not run here.

### Claude Code

Claude Code loads the skill from the checkout as a session plugin and reads MCP servers from its own configuration, so add the server once and sign in:

```sh
claude mcp add --transport http --scope user viam https://app.viam.com/mcp
claude mcp login viam
claude --plugin-dir ./viam-agentic-kit
```

The browser sign-in for `claude mcp login viam` was not run here. Inside the session the skill is `/viam-agentic-kit:viam-help`, where the prefix is the checkout's directory name, and `/mcp` shows `viam`. For example:

```
/viam-agentic-kit:viam-help Is my machine "rover" online, and what is configured on it?
```

Claude Code looks for a plugin's MCP servers in `.mcp.json`, not in the standard `mcp.json`, which is why the server is added with `claude mcp add` rather than through the plugin. If the Viam connector is already set up in your claude.ai account (see Claude above) and you are signed in to Claude Code with that account rather than an API key, the connector is available in Claude Code as well, under its claude.ai name, and the first two commands are unnecessary.

### Cursor

Put the checkout in Cursor's local plugin folder and reload:

```sh
git clone https://github.com/viamrobotics/viam-agentic-kit.git ~/.cursor/plugins/local/viam-agentic-kit
```

Then restart Cursor or run **Developer: Reload Window**. Cursor reads the root `plugin.json`, the `skills/` directory, and `mcp.json`. Open **Customize** in the sidebar to confirm the `viam-help` skill and the `viam` MCP server are listed, and sign in to Viam from the server's entry before using the tools; Cursor supports OAuth for remote servers, but this sign-in was not run here. The skill is `/viam-help` in Agent chat. On Teams and Enterprise plans, admins control local plugins with **Allow Local Plugin Imports** under **Dashboard > Settings > Security & Identity > Marketplace and Plugins**; it is off by default on Enterprise.

### ChatGPT

ChatGPT gets the tools from a custom MCP server and the skill from a plugin.

Tools: open **Plugins**, select the plus button, then **Add custom MCP server**. Enter a name, the server URL `https://app.viam.com/mcp`, and choose **OAuth** for authentication. Create it as a plugin, install it, and mention it with `@` in a prompt. Tools the server does not mark read-only require confirmation by default; the choice can be remembered for the rest of a conversation. Adding a custom MCP server needs developer mode. OpenAI's Help Center documents it for Business and Enterprise/Edu workspaces on ChatGPT web, where an admin turns it on under **Workspace Settings > Permissions & Roles** and each eligible user then enables it in their own settings; Pro users can enable it themselves but can connect only MCP servers with read/fetch permissions. Whether that admits this server, which also has write tools, was not tested. See [Developer mode and MCP apps in ChatGPT](https://help.openai.com/en/articles/12584461-developer-mode-and-mcp-apps-in-chatgpt).

Skill: on the web, ChatGPT loads skills only from installed plugins, and an imported plugin that declares MCP servers in `mcp.json` is marked **Desktop only**, so this kit's skill is not available on the web. The desktop app is the ChatGPT surface that can load it. A workspace admin imports the repository as a marketplace under **Admin > Plugins**, **Add**, **Import marketplace**, using the repository URL; the desktop app can also install a local plugin from a marketplace registered with `codex plugin marketplace add`, as in the Codex section. OpenAI also documents standalone skills as available in the desktop app, so copying `skills/viam-help` into `~/.agents/skills` may work there too, though whether such a skill appears in Chat rather than only in Codex is not stated. None of these was run here. On the web, only the tools are available, through the custom MCP server above.

## Try it

Ask something the skill covers:

> Is my machine "rover" online, and what is configured on it?

The assistant should call `list_machines` with the name, then `read_part_status`, `read_machine_metadata`, and `read_machine_config`, and reply with the status, what the part runs, a configuration summary, and any problems it noticed. Nothing in the skill changes a machine or a config.

A connectivity check that needs no machine and no skill:

> Which Viam organizations can I access?

That is a single call to `list_organizations`.

## Compatibility

Verified on Linux with the files in this repository. "OAuth discovery" means the client fetched the OAuth metadata, registered itself, and printed the Viam sign-in URL; the browser sign-in itself was not run.

| Client | Skill | MCP server | Verified |
| --- | --- | --- | --- |
| Codex CLI 0.162.0 | from the plugin | from `mcp.json` through the plugin | marketplace add, plugin install, skill listed as `viam-agentic-kit:viam-help` in the rendered session prompt, `viam` listed by `codex mcp list`, OAuth discovery |
| Claude (claude.ai) | ZIP upload under Customize > Skills | custom connector | connector signed in; `list_organizations` and `list_machines` returned data; skill upload not run |
| Claude Code 2.1.296 | `--plugin-dir` | `claude mcp add` | skill listed in a session, server added, OAuth discovery |
| Cursor | `skills/` through the local plugin | `mcp.json` through the local plugin | not run; follows Cursor's Agent Plugins documentation |
| ChatGPT | from a plugin or a standalone skill, desktop app only | custom MCP server | not run; follows OpenAI's custom MCP server, developer mode, and plugin management documentation |

`plugin.json` and `mcp.json` validate against the Agent Plugins 1.0.0 JSON Schemas, and the skill passes the Agent Skills reference validator (`agentskills validate`). `claude plugin validate` passes on this checkout, but it only parses the skill's frontmatter: Claude Code reads a manifest only from `.claude-plugin/plugin.json` and plugin MCP servers only from `.mcp.json`, so the result says nothing about `plugin.json` or `mcp.json`. The endpoint's tool catalog was confirmed live through an authenticated claude.ai custom connector: `list_organizations` and `list_machines` returned data, and no Viam resource was changed.

## Keeping the skill in step with the server

Tool names, inputs, and outputs are defined by the server. When it changes, update the tool table in `skills/viam-help/SKILL.md`. The full catalog is in the [MCP server reference](https://docs.viam.com/reference/mcp/).

The skill does not use `read_machine_logs`. It only reads, but the server does not mark it read-only because returning logs records billable data egress for the machine's organization, so it stays out of a read-only workflow.
