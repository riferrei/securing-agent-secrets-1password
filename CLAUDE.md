# CLAUDE.md

Guidance for working in this repository. These rules are not incidental. They
are the security invariants this project is built around, so treat them as hard
constraints, not preferences.

## What this is

A small Go assistant for a customer support team. It answers business questions
about customers (tier, balance, account status, region), using a local model
(via Ollama) to interpret intent. The customer entity is
split across two hashes: `customer:*` (business fields) and `pii:*` (email,
phone, SSN, address). The secret being protected across the project is the Valkey
connection credential.

This is the `controlling-blast-radius` branch. The secret already lives in
1Password (the previous branch); now the agent's reach is scoped. It is issued a
least-privilege Valkey credential — a read-only ACL user restricted to
`~customer:*`, so it cannot read `pii:*` or write anything. 1Password holds both
the admin and the agent credential; the app is only ever handed the agent one.
Seeding, which needs writes, runs separately as a one-shot under the admin
credential.

## Architecture

```
UI (nginx)  ->  REST API (Go, :8000)  ->  Agent (Go)  ->  valkey-prod (as ACL user "agent")
                                             |
                                             +--> 1Password (agent credential, read-only)
                                             +--> Ollama (local LLM)
```

The agent connects to `valkey-prod` as the read-only `agent` ACL user
(`~customer:* +@read +@connection +info`). A one-shot `seed` service loads the data as the admin
user; the agent user cannot write. Both credentials live in one vault; the app
is only ever handed the read-only one.

- `frontend/` static UI served by nginx, which proxies `/api/` to the backend.
- `internal/httpapi` REST surface: `/api/health`, `/api/ready`, `/api/chat`.
- `internal/agent` the model-and-tools reasoning loop.
- `internal/valkeystore` typed access to customers, split across `customer:*` (business) and `pii:*` keys.
- `internal/llm` minimal Ollama chat client with tool calling.
- `internal/config` loads configuration and the `op://` connection references.
- `internal/vault` resolves secrets from 1Password via the service account.
- `cmd/server` and `cmd/seed` entrypoints.

## Security invariants (do not break)

1. **Never log or print a resolved secret.** Not in server logs, not in an error
   message, not in an API response, not in the UI.
2. **The model interprets intent. The Go code owns the Valkey call.** The model
   picks a customer id through the `get_customer` tool. It never generates Valkey
   commands. This is a safety property (no model-authored destructive commands)
   and a correctness one.
3. **Keep the credential out of the model's context.** Do not put the connection
   string in the system prompt, tool descriptions, or any message sent to the
   model. It is only used by the Go code to open the Valkey connection.
4. **No plaintext secret touches disk or version control.** The Valkey credential
   lives in 1Password; `.env` holds only an `op://` reference. The service
   account token is provided at runtime via `OP_SERVICE_ACCOUNT_TOKEN` and is
   never written down.
5. **Resolve, use, discard.** The credential is resolved from 1Password at
   startup, used to build the Valkey client, and never written anywhere else.
6. **The agent's identity stays scoped.** The backend connects as the read-only
   `agent` ACL user (`~customer:* +@read +@connection +info`). Keep it read-only:
   do not grant write or admin categories, widen the key pattern, or give the
   backend the admin credential. Seeding, which needs write access, runs
   as a separate one-shot under the admin credential.

## The series

Each step of the hardening journey is its own branch. `main` is the landing page
that lists them. Each branch is a complete, runnable version of the same agent.
Keep the diff between consecutive branches small: the diff is the story.

## Run commands

Export the service account token, then:

```bash
export OP_SERVICE_ACCOUNT_TOKEN=ops_...
op run --env-file=op.env -- docker compose up --build -d   # provisions the ACL, seeds, starts the stack
# UI at http://localhost:8080

docker compose exec valkey-prod valkey-cli --no-auth-warning --user agent -a "$(op read 'op://Agent Prod/valkey-agent/password')" HGETALL pii:0001   # NOPERM: agent denied PII
docker compose down
```

Prerequisites: Docker, and a 1Password Business account with a service account
that reads the `Agent Prod` vault. That vault holds two Server items: `valkey-prod`
(username `default`, admin password) and `valkey-agent` (username `agent`, the
read-only credential). Ollama and the model run as compose services. On Apple
Silicon the containerized Ollama is CPU-only; see the README for host Ollama.

Conversations have short-term memory: the server keeps per-session history so
follow-up questions resolve against earlier turns.
