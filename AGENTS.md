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
- Describe a workflow: the order of tool calls, how to find the record
  before writing and how to handle each error type. Refer to tools by name
  only. Every read goes through `describe_analytics_schema`, whose overview
  has verified lookup examples, and `run_analytics_query`, which runs them:
  name the lookup a step needs (a client by CPF/CNPJ, what a number belongs
  to, a contact by phone or e-mail, an invoice with its boletos) rather
  than writing its SQL.
- The server sends every client the tool descriptions, parameter schemas
  and its own instructions, so don't copy parameter docs or format rules
  into a skill; they would drift from the server. Some clients don't show
  the instructions to their model, which is why the server repeats in each
  tool's description the rules that tool depends on.
- Use only tool names listed in `TOOLS.md`.
- Every skill tells the agent to answer the user in Brazilian Portuguese.
  Read-only workflows say so. Every skill that writes has a numbered step
  whose title starts with **Resolve**: find the record, decide its kind
  from what the user wants to change, and ask one question when two kinds
  still fit. A later step's title starts with **Confirm**: name the record
  kind, its identifier, each change from old to new and its scope, and
  wait for the user's yes before any tool that creates, updates or deletes
  data.
- Tell the agent to take uuids only from tool results, never to guess them,
  and to look the record up again before retrying a write that failed with
  `ServerError`: right after a write, only a lookup result whose freshness
  is `current` counts. When lookups themselves fail with `ServerError`, the
  agent says they are temporarily unavailable, points the user to the
  Helios web app and writes nothing it couldn't look up.

## Public repository rules

This repository is public. Never add secrets, tokens, client IDs, company
or user uuids, real customer data (names, CPF/CNPJ, addresses, phones,
emails, installation numbers), hostnames other than the public MCP URL, or
documentation of Helios internals. Examples use obviously fictitious
values, the same as the server's own: Maria Exemplo, CPF 123.456.789-09,
UC 3.012.345.678-90, phone +55 (31) 99876-5432. The one deliberate
exception is the support contact in the README's `Ajuda` section, which
the Helios administrator chose to publish.

## Checking a change

- JSON files parse and use tab indentation.
- Claude Code: `claude plugin validate .` validates the plugin and the
  marketplace; `claude --plugin-dir .` loads the plugin for a test session.
- After connecting, ask "Quem sou eu no Helios?": the assistant should call
  `describe_analytics_schema`.
- Then check the words users mix up: "Cancela a cobrança de FEV/2026 da
  Maria Exemplo" should get a question (boleto or invoice) and no cancel;
  "De quem é a instalação …?" with a plant's number should name the plant;
  "Muda o endereço da instalação …" should change the UC, after a yes.
- The Helios source repository's tests check every skill: the tools it
  names exist, a skill that writes has its Resolve and Confirm steps, and
  no real-looking CPF, CNPJ, phone, CEP or installation number appears.
