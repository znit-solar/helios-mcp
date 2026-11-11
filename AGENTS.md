# AGENTS.md

Guidance for AI agents (and people) working in a checkout of this
repository. `CLAUDE.md` imports this file for Claude Code.

## What this repository is

The client kit for the Helios MCP server: instructions, plugin manifests and
skills that connect AI assistants (Claude Code, Claude Desktop, Codex,
ChatGPT) to Helios, the management system of a Brazilian solar energy
company. The server lives at `https://helios.datastrata.app/api/mcp/mcp` and
authenticates each user with OAuth against their Helios account.

This repository is **published from the Helios source repository** on each
deploy. Changes made here directly are overwritten by the next publish;
make them at the source.

## Layout

| Path | Purpose |
|---|---|
| `README.md` | Setup guide for users, in Brazilian Portuguese |
| `TOOLS.md` | Tool catalog generated from the live server. Never edit by hand |
| `skills/<name>/SKILL.md` | Workflow skills shipped by both plugins |
| `.claude-plugin/plugin.json`, `.mcp.json` | Claude Code plugin manifest and its MCP server |
| `.claude-plugin/marketplace.json` | Claude Code marketplace listing this repository as the `helios` plugin |
| `plugin.json`, `mcp.json` | Portable Agent Plugins manifest and MCP server, read by Codex |
| `.agents/plugins/marketplace.json` | Codex marketplace listing the same plugin |
| `PUBLISHING.md` | What the publish job stamps into these files |
| `LICENSE` | MIT license for this repository |

Both plugins point at the same directory, so they share `skills/`. Keep the
server name `helios` and its URL identical in `.mcp.json` and `mcp.json`.

## Writing skills

- English, in the standard `SKILL.md` format: frontmatter `name` (the folder
  name, kebab-case) and `description`, which says what the skill does and
  when to use it, including the pt-BR phrases users are likely to type.
- Describe a workflow: the order of tool calls, what to search before
  writing and how to handle each error type. Refer to tools by name only.
  The server already sends every assistant the tool descriptions, parameter
  schemas and its own instructions, so don't copy parameter docs or format
  rules into a skill; they would drift from the server.
- Use only tool names listed in `TOOLS.md`.
- Every skill tells the agent to answer the user in Brazilian Portuguese and
  to confirm with the user before any tool that creates, updates or deletes
  data. Read-only workflows say so.
- Tell the agent to take uuids only from tool results, never to guess them,
  and to search before retrying a write that failed with `ServerError`.

## Public repository rules

This repository is public. Never add secrets, tokens, client IDs, company
or user uuids, real customer data (names, CPF/CNPJ, addresses, phones,
emails, installation numbers), hostnames other than the public MCP URL, or
documentation of Helios internals. Examples use obviously fictitious
values. The one deliberate exception is the support contact in the
README's `Ajuda` section, which the Helios administrator chose to publish.

## Checking a change

- JSON files parse and use tab indentation.
- Claude Code: `claude plugin validate .` validates the plugin and the
  marketplace; `claude --plugin-dir .` loads the plugin for a test session.
- After connecting, ask "Quem sou eu no Helios?": the assistant should call
  `get_current_user`.
