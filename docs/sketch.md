# Initial sketch

Yeah. I’d make this a **standalone repo**, not a Hermes fork and not merely a loose Hermes plugin. The Hermes plugin is just one adapter into a more general permission system.

Something like **`agent-guard`**:

```text
agent-guard/
├── README.md
├── LICENSE
├── pyproject.toml
├── uv.lock
│
├── docs/
│   ├── architecture.md
│   ├── threat-model.md
│   ├── policy-model.md
│   ├── hermes-integration.md
│   └── hardening.md
│
├── config/
│   ├── default.rules.yaml
│   └── examples/
│       ├── development.rules.yaml
│       ├── main-agent.rules.yaml
│       └── strict.rules.yaml
│
├── src/
│   └── agent_guard/
│       ├── __init__.py
│       ├── cli.py
│       │
│       ├── policy/
│       │   ├── models.py
│       │   ├── engine.py
│       │   ├── matcher.py
│       │   ├── loader.py
│       │   └── suggestions.py
│       │
│       ├── approvals/
│       │   ├── models.py
│       │   ├── store.py
│       │   ├── leases.py
│       │   └── resolver.py
│       │
│       ├── commands/
│       │   ├── parser.py
│       │   ├── shell.py
│       │   ├── powershell.py
│       │   └── effects.py
│       │
│       ├── tools/
│       │   ├── filesystem.py
│       │   ├── terminal.py
│       │   ├── mcp.py
│       │   └── generic.py
│       │
│       ├── service/
│       │   ├── server.py
│       │   ├── protocol.py
│       │   └── sessions.py
│       │
│       ├── audit/
│       │   ├── logger.py
│       │   └── events.py
│       │
│       └── integrations/
│           └── hermes/
│               ├── plugin.py
│               ├── hooks.py
│               └── session.py
│
├── ui/
│   └── ...
│
├── tests/
│   ├── policy/
│   ├── approvals/
│   ├── commands/
│   ├── hermes/
│   └── fixtures/
│
└── scripts/
    ├── install-hermes-plugin.py
    └── doctor.py
```

The important conceptual split is:

```text
Hermes adapter        ← Hermes-specific glue
        ↓
Agent Guard core      ← actual product
        ↓
Policy / leases / audit
```

So if Hermes dies or you eventually move to some other PA agent, **the permission engine survives**.

## The heart of the repo

I think there are really only four important subsystems initially.

### `policy/`

This answers:

> Given this requested operation, what is the default policy?

For v1:

```yaml
rules:

  - id: normal-file-read
    match:
      operation: filesystem.read
      path: "**"
    decision: allow

  - id: normal-file-write
    match:
      operation: filesystem.write
      path: "/home/imi/**"
    decision: allow

  - id: secrets
    match:
      operation:
        - filesystem.read
        - filesystem.write
      path:
        - "/run/agenix/**"
        - "/home/imi/.ssh/**"
    decision: ask

  - id: sudo
    match:
      operation: process.execute
      executable: sudo
    decision: ask

  - id: guard-self-modification
    match:
      operation: filesystem.write
      path: "~/.config/agent-guard/**"
    decision: deny
```

And the result is always:

```python
ALLOW
ASK
DENY
```

Nothing fuzzy.

---

## `approvals/`

This is the **Codex-inspired bit**.

A policy says:

```text
sudo → ASK
```

but the approval store might already contain:

```text
allow sudo systemctl restart llama-swap
for current session
```

So evaluation becomes:

```text
requested operation
       ↓
persistent policy
       ↓
ephemeral leases
       ↓
final decision
```

Lease types:

```text
ONCE
TASK
SESSION
PERSISTENT
```

Maybe ultimately:

```text
ALLOW_ONCE
ALLOW_EXACT_FOR_TASK
ALLOW_SCOPE_FOR_TASK
ALLOW_EXACT_FOR_SESSION
ALLOW_SCOPE_FOR_SESSION
ALWAYS_ALLOW_SCOPE
DENY
```

But internally those don't need to be separate magic decision types. They're just rules with different lifetimes.

For example:

```json
{
  "decision": "allow",
  "operation": "service.manage",
  "resource": "llama-swap.service",
  "lifetime": "session",
  "session_id": "abc123"
}
```

That makes the implementation much cleaner.

---

# `commands/` is intentionally NOT the whole project

This is where we avoid making the same mistake Hermes makes.

V1 only needs to reliably extract obvious things:

```bash
git status
sudo systemctl restart llama-swap
docker logs searxng
rm -rf ~/Downloads/temp
```

into:

```text
executable
argv
cwd
redirections
simple compound commands
```

Something like:

```python
Command(
    executable="systemctl",
    argv=["restart", "llama-swap"],
    elevated=True,
)
```

Then typed recognizers can optionally turn that into:

```text
operation = service.restart
resource = llama-swap.service
```

But if we get:

```bash
bash -c 'weird $(dynamic nonsense) && whatever'
```

we do **not** pretend we've statically solved shell semantics.

It stays:

```text
process.execute
executable = bash
script = ...
```

and policy deals with it conservatively.

That's basically the lesson we should steal from Codex.

---

# Hermes integration should be tiny

Ideally:

```python
async def pre_tool_call(event):
    request = hermes_event_to_guard_request(event)

    decision = await guard.check(request)

    if decision == DENY:
        return {
            "action": "block",
            "message": decision.reason,
        }

    if decision == ASK:
        decision = await guard.request_approval(request)

        if decision.denied:
            return {
                "action": "block",
                "message": "Denied by Agent Guard",
            }

    # Important:
    # Do NOT return Hermes "force approve".
    # Let Hermes' own guardrails run afterward.
    return None
```

That's the entire philosophy.

Agent Guard ALLOW means:

> **I have no objection.**

Not:

> Disable Hermes security.

So:

```text
Agent Guard
     ↓ ALLOW
Hermes native guards
     ↓
execute
```

Exactly what you said you want.

---

# I think `guardd` is worth having even in v1

Rather than putting everything inside the Hermes Python process:

```text
Hermes
   │
   │ Unix socket
   ▼
agent-guardd
```

The daemon owns:

```text
policy
leases
session state
approval requests
audit trail
```

Hermes plugin becomes basically a client.

That buys us several things immediately.

If Hermes crashes, session grants can survive if desired.

The approval UI doesn't have to live inside Hermes.

Other agents can eventually use the same guard.

And when we later harden the system, **guardd is the natural trusted component**.

I'd expose a tiny local API like:

```text
check(request)
request_approval(request)
session_start(id)
session_end(id)
list_leases()
revoke_lease(id)
```

Unix socket only, no internet-facing server.

---

# CLI

I'd absolutely give it a proper CLI from day one.

```bash
agent-guard status

agent-guard rules list

agent-guard rules check \
    --command "sudo systemctl restart llama-swap"

agent-guard leases list

agent-guard leases revoke abc123

agent-guard audit tail

agent-guard doctor
```

`doctor` will become especially useful later:

```text
Hermes plugin                    OK
guardd                           OK
policy loaded                    OK
Hermes native approvals          ON
checkpoint support               ON
hardening namespace              NOT INSTALLED
Docker broker                    NOT INSTALLED
WSL interop broker               NOT INSTALLED
```

That would make this considerably less mysterious to maintain.

---

# Approval UI

For the first implementation, I would keep it extremely boring.

```text
┌ Agent Guard ────────────────────────────────────┐
│                                                │
│ Hermes wants to execute:                       │
│                                                │
│   sudo systemctl restart llama-swap.service    │
│                                                │
│ Policy: privileged-service                     │
│ Resource: llama-swap.service                   │
│                                                │
│ [Allow once]                                   │
│ [Allow for task]                               │
│ [Allow for session]                            │
│ [Always allow this scope]                      │
│                                                │
│ [Deny]                                         │
└────────────────────────────────────────────────┘
```

Later we can make it clever.

The first one just needs to be **reliable and not irritating**.

---

# Then I'd reserve directories for later functionality

Even if we don't implement these immediately:

```text
src/agent_guard/hardening/
    namespace.py
    no_new_privs.py
    control_plane.py

src/agent_guard/brokers/
    sudo.py
    docker.py
    wsl.py

src/agent_guard/checkpoints/
    hermes.py
    planner.py
```

This corresponds to the architecture we've already settled on:

```text
                   Agent Guard
                       │
           ┌───────────┼────────────┐
           │           │            │
        policy       leases       audit
           │
       Hermes hook
           │
           ▼
       normal execution

────────── later hardening ──────────

       permissive namespace
             │
      ┌──────┼────────┐
      │      │        │
     sudo  Docker    WSL
      │      │        │
      └── brokered ───┘
```

So we don't paint ourselves into a corner.

---

## Repo scope I would explicitly state in the README

Something like:

> Agent Guard is a local authorization layer for autonomous agents. It provides deterministic allow/ask/deny policies, scoped approvals, session leases, and auditable capability escalation while preserving the host capabilities required by a main personal agent.
>
> Agent Guard complements rather than replaces an agent runtime's native safety checks.

And explicitly:

```text
Goals
✓ normal main-agent usability
✓ deterministic permission policy
✓ one-shot / task / session / permanent grants
✓ separate read/write/execute authority
✓ retain native Hermes protections
✓ eventually make privileged authority non-bypassable

Non-goals
✗ hermetic task sandbox
✗ malware analysis
✗ proving arbitrary shell scripts safe
✗ replacing Hermes
✗ replacing Agent Vault
```

That makes it really clear what the project actually **is**.

### V1 milestone

I'd keep the first milestone almost brutally small:

```text
M1 — Codex-style permissions for Hermes

[ ] Hermes pre_tool_call plugin
[ ] guardd
[ ] YAML policy loader
[ ] ALLOW / ASK / DENY
[ ] exact command matching
[ ] executable + argv-prefix matching
[ ] tool matching
[ ] path matching
[ ] Allow once
[ ] Allow session
[ ] Always allow
[ ] Deny
[ ] audit log
[ ] Hermes native guards remain enabled
```

Then **M2** gets task scopes and typed capabilities.

**M3** gets NNP/bwrap/control-plane protection.

**M4** gets sudo/Docker/WSL brokers.

That gives us a repo where the first useful version is probably a few thousand lines at most, rather than accidentally starting a new operating system. 😭

And crucially, **M1 alone already fixes the thing that's annoying you about Hermes today.**

---

Yep, I can see it. `hermes-gate` is basically the repo we just sketched, but it’s still at the **skeleton stage**.

Right now the root already has the right broad shape: `config/`, `docs/`, `scripts/`, `src/`, `tests/`, `ui/`, plus `pyproject.toml`, and the README currently just says it’s a WIP Hermes plugin for running Hermes as a main agent in the local environment.

The commit history also makes it clear this was just initialized today: `Initial commit` → `local init` → `skeleton`. So there isn’t really an existing architecture to fight yet; this is the perfect point to lock it down.

I’d keep this repo and make it the thing we discussed, probably with this structure inside `src/`:

```text
src/hermes_gate/
├── __init__.py
├── cli.py
│
├── policy/
│   ├── models.py
│   ├── engine.py
│   ├── matcher.py
│   └── loader.py
│
├── approvals/
│   ├── models.py
│   ├── leases.py
│   ├── store.py
│   └── resolver.py
│
├── integrations/
│   └── hermes/
│       ├── plugin.py
│       ├── hooks.py
│       └── adapter.py
│
├── commands/
│   ├── parser.py
│   ├── shell.py
│   └── effects.py
│
├── service/
│   ├── server.py
│   ├── protocol.py
│   └── sessions.py
│
└── audit/
    ├── logger.py
    └── events.py
```

And I’d make the immediate repo mission very narrow:

```text
Hermes tool call
    ↓
pre_tool_call
    ↓
Hermes Gate
    ↓
ALLOW / ASK / DENY
    ↓
once / session / persistent scope
    ↓
Hermes native guardrails still run
```

The first files worth actually implementing are probably:

```text
pyproject.toml
src/hermes_gate/policy/models.py
src/hermes_gate/policy/engine.py
src/hermes_gate/approvals/leases.py
src/hermes_gate/integrations/hermes/plugin.py
tests/policy/test_engine.py
```

I would **not** touch bwrap, Docker brokers, sudo brokers, WSL interop, or fancy UI yet. Get the Codex-style permission lifecycle working first.

Also, `hermes-gate` is a better name than `agent-guard` if this is intentionally Hermes-first. We can still keep the internals generic enough that another runtime adapter could be added later.

If you want, next I can inspect the current skeleton commit itself and tell you exactly what I’d change/add before we start coding.
