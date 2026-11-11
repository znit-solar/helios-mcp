# Publishing

How the publish job turns `client-kit/` in the Helios source repository into
the public repository `znit-solar/helios-mcp`. Read by the person or CI job
that publishes; users never need it.

The `publish-client-kit` CI job does all of this after each successful
production deploy, using `scripts/publish-client-kit.ts` in the source
repository; change that script when this procedure changes. It skips the
publish when nothing but the stamps would differ.

## What to copy

Copy the whole `client-kit/` directory to the root of the public
repository, including the dot directories (`.claude-plugin/`, `.agents/`)
and `.mcp.json`. Copy nothing from outside `client-kit/`.

There is no need to regenerate `TOOLS.md` first: the Helios test suite
fails while it is stale, so the committed file always matches the server
being deployed.

## What to stamp

The source keeps placeholders. Replace them in the copy, never in the
source:

| File | JSON path | Placeholder | Stamp with |
|---|---|---|---|
| `.claude-plugin/plugin.json` | `version` | `0.0.0-unpublished` | The release version |
| `plugin.json` | `version` | `0.0.0-unpublished` | The same release version |
| `.claude-plugin/marketplace.json` | `plugins[0].metadata.sourceCommit` | `unpublished` | The full Helios commit SHA being deployed |

- The version must change on every publish. Claude Code keeps an installed
  plugin on the version in its `plugin.json` until that string changes, so
  users of an unchanged version never receive new skills. Any increasing
  string works; the version is not checked against semver. A suggestion is
  `<MCP_SERVER_VERSION>+<short SHA>`, e.g. `1.0.0+a1b2c3d`, where the
  server version is the one printed at the top of `TOOLS.md`.
- Use the same version in both manifests, so Claude Code and Codex users
  see the same release.
- `metadata` is a free-form object that Claude Code ignores; the commit
  there is only for people tracing a release back to its source.
- Stamp by editing the JSON (e.g. `jq`), not with a text substitution that
  could match elsewhere, and keep tab indentation.

After stamping, none of the three files may still contain `unpublished`.

## Checks before pushing

- Every `.json` file parses.
- If the Claude Code CLI is available: `claude plugin validate .` passes.
- The copy contains no secrets, tokens, uuids, customer data or hostnames
  other than `helios.datastrata.app` (see `AGENTS.md`).

The public repository's history is the publish history: one commit per
publish, whose message names the version and the source commit.
