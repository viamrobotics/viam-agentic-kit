---
name: viam-help
description: Check on a Viam machine through the Viam MCP server. Finds the machine by name, reports whether it is online, what it runs, and how it is configured, and points out anything that looks wrong. Use when the user asks about the state, health, or configuration of a Viam machine, robot, or part, or asks why a machine is not working. Read-only. Nothing here changes a machine, a config, or an account.
compatibility: Requires the Viam MCP server (https://app.viam.com/mcp) to be connected and authenticated in the client.
---

# Viam machine check-up

Inspect one Viam machine through the Viam MCP server and report what you find. Every step uses a tool the server marks read-only. Nothing here changes a machine, a config, or an account.

## When to use

- The user asks whether a machine is online, what it is running, or how it is configured.
- The user says a machine, component, or service is not working and wants a first look before changing anything.
- You need machine facts (OS, viam-server version, part names, configured resources) before another task.

Do not use it to change configuration or to operate hardware. The server has separate write tools for that, and the user must ask for those explicitly.

## Tools

The Viam MCP server exposes these tools. A client may show them with a prefix such as `viam.` or `mcp__viam__`. The name after the prefix is the same.

| Step | Tool | What it returns |
| --- | --- | --- |
| Find the machine | `list_machines` | Machine ids with location and organization. `name` keeps machines whose stored name contains it, ignoring case and separators. |
| Disambiguate | `list_organizations`, `list_organization_locations` | Organization and location ids to narrow `list_machines`. |
| Is it online? | `read_part_status` | `online`, `offline` with `last_online` and `offline_for`, or `awaiting_setup` (never connected). Lists the machine's other parts as `other_part_names`. |
| What does it run? | `read_machine_metadata` | What the part last reported about itself (OS, platform, host name, IP addresses, viam-server version), plus its location and organization. Lists the other parts as `other_parts`. |
| How is it configured? | `read_machine_config` | Every part's config as saved, plus the resolved contents of each imported fragment. |
| What is this model? | `get_registry_item_information` | Module and model documentation for an `api` and `model` pair taken from the config. |
| What do the docs say? | `read_viam_docs` | Passages from Viam's documentation for a concept or config question. |

## Steps

1. Find the machine. Call `list_machines` with `name` set to the machine name the user gave, as they spelled it: a name or part of one, never the whole sentence. One match: use its `id`. Several: ask the user which one, showing name, location, and organization. None, with `truncated` true: narrow with `organization_id` or `location_id` and try again. None at all: say so. Never guess an id.
2. Check status with `read_part_status`. `awaiting_setup` means viam-server has never connected to Viam, so there is no live state to inspect. `offline` means the same for now. Report `last_online` and `offline_for`.
3. Read `read_machine_metadata` for what the part runs.
4. Read `read_machine_config`. For each part, note the components, services, modules, and fragment imports in `config`. In `resolved_fragments`, `error` says why a fragment's contents are not included: one starting "resolved contents omitted" means the fragment resolved but was too large to inline, which is a note, not a problem; any other `error` means the import did not resolve. An entry with `disabled` true contributes nothing. A part-level `fragments_error` means its fragment list could not be read at all. An unresolved import, a disabled import, and a `fragments_error` are each a finding.
5. Only when the question is about a specific component or service: look up its `api` and `model` with `get_registry_item_information`, passing both as full triplets as the config spells them (a bare model name in a config means `rdk:builtin:<name>`), and check the configured attributes against the documentation. A "not found, or you do not have access" answer here means either that no registry module provides that pair or that the module is private to an organization you are not a member of; the server returns the same message for both. Report it as a model you could not look up, and continue.
6. On a multi-part machine, `read_machine_config` lists every part and marks the main one with `main_part`. `read_part_status` and `read_machine_metadata` read one part, the main part unless you pass `part_name`, and list the others as `other_part_names` and `other_parts`. Repeat them for another part only when the question is about it.

Rules:

- Ids are opaque. Use only an id a tool returned in this conversation.
- Keep calls to what the question needs. Do not read every part or look up every model.
- When a tool given an id says the resource is not found or not accessible, stop and ask. That is often a permission problem, not a missing machine.

## Missing information

- No machine name: call `list_machines` with no arguments. If the list is short, offer it. If it is long or `truncated`, ask for the name.
- A tool given an id answers "not found, or you do not have access": tell the user which id you used and that their Viam account may lack access to it. Do not retry with a guessed id.
- A tool says the access token is no longer valid, or that Viam's terms of service must be accepted: tell the user to re-authenticate the Viam MCP connection, or to log in at https://app.viam.com and accept the terms, then stop.
- No Viam tools are available at all: tell the user the Viam MCP server is not connected in this client and point them to the README of viam-agentic-kit.

## Report

Lead with the answer to the question that was asked. Then cover:

- Status: online, offline since when, or awaiting setup.
- Runs: OS and platform, and the viam-server version, as last reported.
- Configuration: how many components, services, modules, and fragments. List them by name and model when there are few, and summarize when there are many.
- Problems: a fragment import that did not resolve, a fragment list that could not be read, a part that has never connected, a configured model you could not look up in the registry, or a configured attribute that does not match the model's documentation. When nothing is wrong, say what you checked.
- Next step: one suggestion. Phrase it as a question when it would need a write tool or would move hardware.

Quote ids only when the user needs them, for example to open the machine at https://app.viam.com. Do not paste a whole config unless asked.
