# CLAUDE.md

Guidance for working in this branch.

## What this is

The `securing-mcp-servers` branch of *What Your Agent Doesn't Know Can't Hurt
You*. It is **different in kind from the other branches: there is no application
here.** The others build and harden a Go agent; this one shows the other path,
where the agent is an off-the-shelf MCP host and the tool layer is a server you
did not write.

The whole content is `README.md` and `mcp/claude_desktop_config.json`: how the
AWS Labs Valkey MCP server (`awslabs.valkey-mcp-server`, run via `uvx`) is secured
with 1Password. Its Valkey credential is the same read-only, PII-blind `agent`
credential from the `controlling-blast-radius` branch, resolved by `op run` from
an `op://` reference and never written in plaintext.

The lesson here is defense in depth. The server ships a `--readonly` flag that
disables its write and admin tools, so the branch layers two independent controls:
`--readonly` at the tool and the read-only `agent` ACL at the database. Either
alone blocks a write; keep both. The password env var the server reads is
`VALKEY_PWD` (not `VALKEY_PASSWORD`).

## Do not

- Do not add application code here, or treat this as a runnable version of the
  agent. If you need the running app (and the Valkey it talks to), check out
  `controlling-blast-radius`.
- Do not put a resolved secret in the config or anywhere on disk. The config
  holds `op://` references only.

## The series

Each step of the hardening journey is its own branch; `main` lists them. This is
the final one.
