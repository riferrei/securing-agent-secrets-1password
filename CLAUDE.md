# CLAUDE.md

Guidance for working in this repository. These rules are not incidental. They
are the security invariants this project is built around, so treat them as hard
constraints, not preferences.

## What this is

A small Go assistant for a customer support team. It answers business questions
about customers (tier, balance, account status, region), using a local model
(via Ollama) to interpret intent. Here every field, business
and PII (email, phone, SSN, address) alike, sits in one `customer:*` hash: the
naive baseline. Read and write both flow through the one Valkey connection, so the
agent's reach is broad, which is the blast radius a later branch scopes down. The
`controlling-blast-radius` branch refactors the entity into two hashes so a scoped
identity can be denied the PII. The secret being protected across the project is
the Valkey connection credential.

This is the `env-vars-as-source-truth` branch. The credential lives in a `.env`
file. It works, and that is the point: this is the setup developers believe is
safe. The weakness is operational, not in the code. The plaintext credential sits
on disk, one `.gitignore` slip from being committed. The next branch moves it
into 1Password.

## Architecture

```
UI (nginx)  ->  REST API (Go, :8000)  ->  Agent (Go)  ->  valkey-prod
                                             |
                                             +--> Ollama (local LLM)
```

The agent is naively wired to `valkey-prod` (the production database) with full
access. A later branch scopes it to a read-only, PII-blind Valkey identity.

- `frontend/` static UI served by nginx, which proxies `/api/` to the backend.
- `internal/httpapi` REST surface: `/api/health`, `/api/ready`, `/api/chat`.
- `internal/agent` the model-and-tools reasoning loop.
- `internal/valkeystore` typed access to customers, stored as a single `customer:*` hash (business fields and PII together).
- `internal/llm` minimal Ollama chat client with tool calling.
- `internal/config` loads configuration and the credential from the environment.
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
4. **No plaintext secret should ever be committed, except here, on purpose.**
   This branch commits `.env` deliberately to make the risk concrete: the
   anti-pattern the branch exists to show. From the next branch on, the credential
   never touches disk or version control.
5. **Resolve, use, discard.** The credential is read from the environment at
   startup and used to build the Valkey client. It is not written anywhere else.

## The series

Each step of the hardening journey is its own branch. `main` is the landing page
that lists them. Each branch is a complete, runnable version of the same agent.
Keep the diff between consecutive branches small: the diff is the story.

## Run commands

The committed `.env` has everything needed, so there is nothing to configure.

```bash
docker compose up --build -d   # build and start Valkey, Ollama, backend, frontend
# UI at http://localhost:8080 (the backend seeds sample customers on startup)

docker compose exec valkey-prod valkey-cli -a "$(grep -E '^VALKEY_PASSWORD=' .env | cut -d= -f2)" HGETALL customer:0001
docker compose down
```

Prerequisite: Docker. Ollama and the model run as compose services, so the
stack is self-contained. On Apple Silicon the containerized Ollama is CPU-only;
see the README for switching to host Ollama for GPU speed.

Conversations have short-term memory: the server keeps per-session history so
follow-up questions resolve against earlier turns.
