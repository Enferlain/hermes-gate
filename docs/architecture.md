# Architecture

Hermes Gate is a local authorization layer for running Hermes as a powerful main agent on a user's normal machine.

The goal is not to isolate Hermes from the environment it is supposed to manage. A main agent needs access to real projects, user files, development tools, network services, Windows interoperability, and other local resources to remain useful.

Instead, Hermes Gate adds a deterministic authorization layer around sensitive actions while preserving Hermes' existing guardrails.

The intended model is:

> Give the main agent normal capabilities, but require sensitive capabilities to pass through an explicit, auditable allow/ask/deny policy.

## Design goals

Hermes Gate should:

* preserve the capabilities expected from a main local agent;
* add deterministic `allow`, `ask`, and `deny` decisions;
* support one-time, task-scoped, session-scoped, and persistent approvals;
* distinguish between different kinds of authority such as read, write, execution, privilege escalation, and external side effects;
* preserve Hermes' native safety checks rather than replacing them;
* keep ordinary low-risk actions frictionless;
* avoid relying on an LLM as the final security decision-maker;
* make security-sensitive authority flow through a small number of auditable paths;
* remain useful even before the later OS-level hardening layers are implemented.

## Non-goals

Hermes Gate is not intended to:

* turn the main Hermes agent into a hermetic coding sandbox;
* prevent every possible user-level filesystem mistake;
* statically prove arbitrary shell scripts safe;
* replace Hermes;
* replace Agent Vault;
* provide malware-analysis isolation;
* replace containers or virtual machines for disposable task agents.

Separate coding or task agents can use stronger isolation such as Docker because they do not require the same machine-wide capabilities as the main agent.

## High-level architecture

The initial architecture is:

```text
Model
  │
  │ chooses tool call
  ▼
Hermes
  │
  │ pre_tool_call hook
  ▼
Hermes Gate
  │
  ├── ALLOW ────────┐
  │                 │
  ├── ASK            │
  │    │             │
  │    ▼             │
  │ approval/lease   │
  │    │             │
  │    └─────────────┤
  │                  │
  └── DENY           │
       │             │
       └── block     │
                     ▼
             Hermes native guardrails
                     │
                     ▼
                 tool executes
```

A Hermes Gate `ALLOW` decision does **not** mean "force execution."

It means:

> Hermes Gate has no objection to this operation.

Hermes' own approval system, dangerous-command detection, hard blocks, checkpointing, and other built-in protections remain active afterward.

This makes Hermes Gate an additional security layer rather than a replacement for Hermes security.

## Core components

### Hermes integration

The Hermes integration is intentionally thin.

Its main responsibility is to translate Hermes tool calls into Hermes Gate requests and pass the resulting decision back to Hermes.

The primary integration point is Hermes' `pre_tool_call` hook.

Conceptually:

```python
request = translate_hermes_tool_call(event)
decision = gate.evaluate(request)

if decision is DENY:
    block_tool_call()

if decision is ASK:
    request_user_approval()

if decision is ALLOW:
    defer_to_hermes()
```

Hermes-specific code should remain isolated from the core authorization engine so the policy system is not tightly coupled to one runtime.

### Policy engine

The policy engine evaluates requested operations against deterministic rules.

Every evaluation returns one of:

```text
ALLOW
ASK
DENY
```

The engine should begin with matchers that can be evaluated reliably, such as:

* tool name;
* command executable;
* argument prefix or exact arguments;
* current working directory;
* filesystem path;
* Hermes or MCP tool identifier;
* known resource identifier.

The engine should not pretend to understand arbitrary program behavior.

For example:

```text
git status
```

can be classified reliably.

A command such as:

```text
python arbitrary_script.py
```

cannot generally be statically reduced to all of its future effects.

Hermes Gate should handle uncertain semantics conservatively rather than attempting to build a complete shell or program analyzer.

### Approval and lease system

An `ASK` decision can be resolved into a scoped temporary or persistent authorization.

The intended approval lifetimes are:

```text
once
task
session
persistent
```

These are represented internally as rules with different lifetimes rather than entirely separate authorization mechanisms.

For example:

```text
ALLOW service-management
resource = llama-swap.service
scope = session
```

allows the agent to continue managing that service for the current session without prompting for every individual command.

The approval UX should eventually support choices such as:

```text
Allow once
Allow exact operation for task
Allow scope for task
Allow exact operation for session
Allow scope for session
Always allow scope
Deny
```

This model is inspired by Codex's distinction between one-time approval, session approval, and persistent execution-policy amendments, while allowing Hermes Gate to support richer resource-aware scopes.

### Rule precedence

Rules and leases must resolve deterministically.

The exact precedence model belongs in the policy specification, but the architecture requires that:

* explicit denies cannot be accidentally weakened by unrelated broad allows;
* temporary leases have clearly defined scope and lifetime;
* persistent rules are inspectable by the user;
* policy conflicts resolve predictably;
* the decision process can be explained in audit output.

The authorization engine must not depend on an LLM to resolve conflicting rules.

### Audit system

Every security-relevant decision should be auditable.

An audit event should be able to record:

```text
session
tool
requested operation
matched rule
decision
approval scope
lease used
reason
timestamp
```

The audit log is intended both for debugging policy and for understanding what authority Hermes actually exercised.

## Access and execution are separate authorities

Hermes Gate should avoid treating "access to a resource" as a single permission.

Different actions on the same resource can carry very different risk.

For example:

```text
READ   ~/Documents/**       allow
WRITE  ~/Documents/**       allow or ask
DELETE ~/Documents/**       ask

READ   service status       allow
RESTART normal service      session lease
MODIFY service definition   ask
```

Likewise, executing a program and reading its files are different authorities.

This distinction becomes increasingly important as Hermes Gate gains typed operation classifiers.

## Command classification

The command layer should provide useful structure without becoming the security boundary itself.

For straightforward commands it may extract:

```text
executable
argv
cwd
redirections
simple compound commands
```

Known commands can later be translated into typed operations.

For example:

```text
systemctl restart llama-swap.service
```

may become:

```text
operation = service.restart
resource = llama-swap.service
```

Similarly:

```text
docker logs searxng
```

may become:

```text
operation = container.logs
resource = searxng
```

Typed operations allow safer authorization than broad executable prefixes.

However, classification is advisory to the authorization system. It is not assumed to perfectly predict arbitrary program behavior.

## Failure model

Hermes Gate distinguishes between several classes of failure.

### Intended sensitive actions

Example:

```text
sudo systemctl restart ...
```

These are handled by policy and approval.

### Unexpected child-process behavior

Example:

```text
python script.py
```

unexpectedly attempts privilege escalation.

Policy parsing alone cannot reliably predict this.

Later hardening layers are intended to ensure that sensitive authority such as raw privilege escalation, Docker host control, or Windows execution has only one permitted route.

### Ordinary user-level filesystem mistakes

Example:

```text
rm -rf ~/Downloads/*
```

when a narrower path was intended.

The main agent cannot be prevented from making all ordinary user-level modifications without losing too much usefulness.

Broad or destructive operations may therefore be paired with approval rules and filesystem checkpoints.

Hermes already provides checkpoint infrastructure that Hermes Gate may integrate with.

### External side effects

Filesystem rollback cannot undo actions such as:

```text
git push
sent messages
remote API mutations
cloud resource deletion
payments
Windows registry changes
```

These operations require explicit authorization and cannot rely on checkpointing as a safety mechanism.

## Future hardening

The initial Hermes Gate plugin improves authorization UX and policy quality, but by itself it is not a complete security boundary.

A sufficiently capable child process could otherwise attempt to bypass the plugin through another authority path.

Later hardening will make selected sensitive capabilities non-bypassable while leaving the rest of the system largely unrestricted.

The intended hardening model is:

```text
Hermes
  │
  │ mostly normal environment
  ▼
permissive process boundary
  │
  ├── normal HOME and projects
  ├── normal network
  ├── normal filesystem access
  ├── /mnt/c and other required data
  │
  ├── Agent Guard control plane read-only
  ├── raw privilege escalation unavailable
  ├── raw Docker control unavailable
  └── raw WSL interop unavailable
```

The purpose of this boundary is not to determine what Hermes may work on.

Its purpose is to make Hermes Gate's policy assumptions true.

For example:

```text
Hermes Gate says:
Docker host authority must pass through the Docker broker.

Hard boundary guarantees:
/var/run/docker.sock cannot be accessed directly.
```

## Capability brokers

Some capabilities eventually need a single auditable execution route.

Likely brokered capabilities include:

```text
privilege escalation
Docker host control
Windows/WSL interop
other root-equivalent local control sockets
```

Normal operations should remain direct whenever possible.

Brokered capabilities should be kept to the minimum set required for security because unnecessary brokerage creates user friction.

The architectural principle is:

> Sensitive capabilities should have one auditable route rather than several equivalent bypass paths.

## Trust boundaries

The main agent and processes it launches are not assumed to be fully trusted.

Hermes Gate policy and the mechanisms enforcing Hermes Gate must therefore eventually be protected from modification by the Hermes process tree.

The trusted control plane includes at least:

```text
Hermes Gate policy
Hermes Gate persistent authorization state
Hermes Gate broker configuration
Agent Vault control data
security-sensitive credential configuration
hardening launcher configuration
```

The main agent may request changes to protected policy, but should not eventually be able to grant itself new authority by directly modifying the control plane.

## Relationship with Agent Vault

Agent Vault and Hermes Gate solve related but different problems.

Agent Vault controls access to credentials and provider authentication.

Hermes Gate controls whether the agent is authorized to perform an operation.

They may eventually compose so that Hermes Gate grants a capability and Agent Vault performs the corresponding authenticated operation without exposing the underlying secret to the agent.

Neither project should silently assume that the other provides its own authorization guarantees.

## Relationship with task-agent isolation

Hermes Gate is designed primarily for the persistent main agent.

Short-lived coding or task agents should generally use stronger isolation when practical.

For example:

```text
main Hermes agent
    → Hermes Gate
    → broad local capabilities

task coding agent
    → disposable Docker environment
    → repository + required tools
    → limited host access
```

These are complementary security models for different roles.

## Implementation stages

### M1 — deterministic authorization

Implement:

* Hermes `pre_tool_call` integration;
* request and decision models;
* static rule engine;
* `ALLOW`, `ASK`, and `DENY`;
* exact command and executable matching;
* path and tool matching;
* one-time approval;
* session approval;
* persistent rules;
* audit logging;
* preservation of Hermes native guardrails.

### M2 — richer authorization

Add:

* task-scoped leases;
* suggested approval scopes;
* typed operations and resources;
* improved shell parsing;
* richer path and tool policies;
* approval management and revocation.

### M3 — process hardening

Add:

* protected control-plane mounts;
* `no_new_privs`;
* namespace hardening;
* authority-surface auditing;
* protection against direct policy bypass.

### M4 — capability brokers

Broker sensitive capabilities such as:

* sudo/root operations;
* Docker host control;
* WSL/Windows execution;
* other privileged local services.

### M5 — recovery integration

Integrate destructive-operation classification with Hermes checkpointing and improve checkpoint coverage for main-agent filesystem operations.

## Guiding principle

Hermes Gate is not trying to remove the main agent's capabilities.

It is trying to make authority explicit.

Normal work should remain normal.

Sensitive authority should require a deterministic policy decision.

Repeated legitimate work should become frictionless through scoped leases.

And the agent should eventually be unable to bypass the same policy system that decides whether a sensitive operation is allowed.
