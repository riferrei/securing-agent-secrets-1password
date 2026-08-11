# Controlling the blast radius

> A read-only, PII-blind identity: even a hijacked agent can't read SSNs or write.

Part of **[What Your Agent Doesn't Know Can't Hurt You][series]**, a hands-on
series on keeping secrets out of an AI agent's reach.

---

The earlier steps secured the *secret*. The agent's *reach* was untouched: it
connected with full access, so it could read every field of every customer, PII
included, and a prompt injection could talk it into doing exactly that. This step
shrinks the blast radius so that even a fully hijacked agent can do little harm.

The customer record splits into two keys: `customer:*` (business fields) and
`pii:*` (email, phone, SSN, address), and the agent is issued a **least-privilege
Valkey credential**: an ACL user scoped to `~customer:* +@read +@connection +info`.
It cannot read `pii:*` and cannot write anything. **Valkey refuses, not the agent.** Talk it into
enumerating everyone and it still comes up empty on SSNs; ask it to change a
record and the write is denied.

1Password's part: it holds *both* the admin credential and the read-only agent
credential, and the app is only ever handed the read-only one. Least privilege is
what's in the vault, not a promise in the code.

## How it works

![Architecture](docs/architecture.png)

The agent can't write and can't see PII, so it can't seed itself. Seeding is a
separate one-shot that resolves the *admin* credential and loads the data. Same
service account, two different Valkey identities.

## Prerequisites

- **Docker**
- The **[1Password CLI][op-cli]** (`op`) and a 1Password account with service
  accounts (Business or Teams)

## 1Password setup

One vault (`Agent Prod`) with **two Server items**: the admin identity that seeds,
and the read-only identity the agent runs as.

| Item | `host` | `port` | `username` | `password` |
|------|------------|--------|------------|------------|
| `valkey-prod` | `valkey-prod` | `6379` | `default` | admin password |
| `valkey-agent` | `valkey-prod` | `6379` | `agent` | any password you choose |

Plus a **service account** with read access to the vault, its token exported:

```bash
export OP_SERVICE_ACCOUNT_TOKEN=ops_...
```

## Run it

```bash
op run --env-file=op.env -- docker compose up --build -d
```

`op run` resolves both passwords into the Valkey ACL. The seed one-shot loads the
data as admin, then the app starts as the read-only `agent` user. Open
**http://localhost:8080**; stop with `docker compose down`.

## See the blast radius hold

Ask the agent a business question (*"What tier is customer 3, and what's their
balance?"*) and it answers fine. Ask for an SSN, or to change a record, and it
can't.

Prove it at the database. The PII is there, and the **admin** identity can read
it:

```bash
docker compose exec valkey-prod valkey-cli --no-auth-warning \
  -a "$(op read 'op://Agent Prod/valkey-prod/password')" HGETALL pii:0001
```

The **agent** identity cannot. Reads of PII and any write are refused:

```bash
pw="$(op read 'op://Agent Prod/valkey-agent/password')"
docker compose exec valkey-prod valkey-cli --no-auth-warning --user agent -a "$pw" HGETALL customer:0001  # ok
docker compose exec valkey-prod valkey-cli --no-auth-warning --user agent -a "$pw" HGETALL pii:0001        # NOPERM
docker compose exec valkey-prod valkey-cli --no-auth-warning --user agent -a "$pw" HSET customer:0001 x y  # NOPERM
```

The limit is enforced by the database, not by the model choosing to behave.

## If something does leak

Scoping bounds what a leaked credential *can* do; the vault bounds how long it
can do it. Everything here, the admin password, the agent password, and the
service account token that resolves them, is held in 1Password and disposable.
**Rotate** the service account token to cut over to a fresh one, or **revoke** it
to kill all access at once, both the seeding identity and the agent's, from a
single control point and with no code change or redeploy. Rotating either Valkey
password in the vault flows through on the next start the same way. A suspected
compromise is a click, not a scramble.

## What this doesn't solve

The scoping lives in *this app's* code and Valkey wiring. Point an agent at an
off-the-shelf MCP server (a tool you didn't write and can't add guardrails to),
and none of that discipline comes along for free. The next step brings 1Password
to exactly that case.

---

**← Previous: [A vault as the source of truth][prev] · Next: [Securing MCP servers][next] →**

[series]: https://github.com/riferrei/securing-agent-secrets-1password
[prev]: https://github.com/riferrei/securing-agent-secrets-1password/tree/vaults-as-source-truth
[next]: https://github.com/riferrei/securing-agent-secrets-1password/tree/securing-mcp-servers
[op-cli]: https://developer.1password.com/docs/cli/
