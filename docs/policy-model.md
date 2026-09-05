# Policy Model

This document defines the authorization semantics used by Hermes Gate.

It describes how requests are represented, how rules are matched, how temporary and persistent approvals work, how conflicting rules are resolved, and what each decision means.

The policy model is intentionally deterministic.

Hermes Gate must not require an LLM to decide whether an operation is allowed.

## Core decisions

Every evaluated request resolves to one of three decisions:

```text
ALLOW
ASK
DENY
```

### ALLOW

`ALLOW` means Hermes Gate does not object to the operation.

It does **not** bypass Hermes' own guardrails.

The operation continues through Hermes' normal safety and approval systems.

Conceptually:

```text
Hermes Gate ALLOW
        ↓
Hermes native guardrails
        ↓
tool execution
```

### ASK

`ASK` means the operation requires user authorization unless a matching active approval lease already exists.

The user may approve the operation with a specific lifetime and scope.

### DENY

`DENY` means the operation must not execute.

A deny is not an approval prompt.

The tool call is blocked and Hermes receives an explanation where possible.

## Requests

Hermes Gate evaluates normalized authorization requests.

A request should contain enough information to identify what Hermes is trying to do without pretending to fully predict arbitrary program behavior.

A generic request may contain:

```text
tool
operation
resource
arguments
cwd
session
task
raw request
```

Not every field is required for every request.

For example:

```text
tool = terminal
executable = git
argv = ["status"]
cwd = /home/imi/dev/hermes-gate
```

A more strongly classified request may become:

```text
operation = service.restart
resource = llama-swap.service
```

The original request must remain available for audit and approval display.

## Rule model

A policy rule consists conceptually of:

```text
id
match
decision
priority
reason
```

Example:

```yaml
- id: sudo-default
  match:
    tool: terminal
    executable: sudo
  decision: ask
  reason: Privileged commands require approval.
```

A rule does not need to match every field of a request.

Unspecified fields act as wildcards.

## Matchers

Initial matcher types should remain simple and deterministic.

Supported match dimensions may include:

```text
tool
operation
executable
argv
argv prefix
cwd
path
resource
MCP server
MCP tool
```

The policy engine should not claim to understand arbitrary script effects.

For example:

```text
git status
```

can be matched reliably.

However:

```text
python unknown_script.py
```

cannot generally be reduced to every filesystem, network, or privilege effect it may later produce.

When classification is uncertain, Hermes Gate should fall back to the information it can reliably identify.

## Exact and scoped matching

Approvals may refer either to an exact request or to a broader scope.

### Exact request

An exact approval matches one normalized operation.

Example:

```text
sudo systemctl restart llama-swap.service
```

It does not automatically authorize:

```text
sudo systemctl restart docker.service
```

### Scoped request

A scoped approval authorizes a meaningful class of operations.

Example:

```text
operation = service.manage
resource = llama-swap.service
```

This may authorize:

```text
status
restart
stop
start
```

for that service, depending on the scope definition.

Scoped approvals should prefer resource-aware authorization over broad command prefixes when reliable typed information is available.

## Approval lifetimes

Hermes Gate supports four approval lifetimes:

```text
ONCE
TASK
SESSION
PERSISTENT
```

These are represented as rules or leases with different expiration conditions.

### ONCE

Applies only to the current authorization request.

The same request made again requires another decision unless another rule or lease applies.

### TASK

Applies to the current logical user task.

The lease expires when the task completes or is abandoned.

Task-boundary detection may depend on Hermes integration and may be added after the initial implementation.

### SESSION

Applies until the current Hermes session ends or is reset.

A session approval must not survive into a new session.

### PERSISTENT

Creates or updates a persistent user policy rule.

Persistent approvals survive application restarts and future Hermes sessions.

Persistent policy must remain human-readable and inspectable.

## Approval outcomes

When the user is prompted for an `ASK` request, the UI may offer outcomes such as:

```text
Allow once
Allow exact request for task
Allow scoped request for task
Allow exact request for session
Allow scoped request for session
Always allow exact request
Always allow scoped request
Deny
```

The exact UI may expose fewer options initially.

Internally, these should reduce to the same rule and lease model rather than requiring unrelated mechanisms.

## Leases

A lease is a temporary authorization created from an approval.

A lease contains enough information to determine:

```text
decision
match scope
lifetime
session
task
creation time
source approval
```

Example:

```text
decision = ALLOW
operation = service.manage
resource = llama-swap.service
lifetime = SESSION
session = abc123
```

Leases should be auditable and revocable.

## Persistent policy

Persistent approvals must be stored separately from temporary session state.

Persistent rules should be:

```text
human-readable
deterministic
version-controlled if desired
auditable
editable without specialized tooling
```

Hermes Gate should avoid opaque authorization databases for the primary persistent policy.

A database may still be used for transient leases, audit state, or runtime metadata.

## Default behavior

If no Hermes Gate rule or lease matches a request, the default decision is:

```text
ALLOW
```

This preserves normal Hermes behavior.

Hermes Gate is an additional authorization layer, not a replacement for Hermes' own guardrails.

A future stricter policy profile may choose a different default, but the normal main-agent profile should default to allow.

## Rule precedence

Rule resolution must be deterministic.

The exact algorithm may evolve, but the initial model should follow these principles:

1. hard security denies override all user-created temporary approvals;
2. more specific matching rules override less specific rules of equal class;
3. explicit deny rules override ordinary allow rules at the same level;
4. active approval leases may satisfy `ASK` rules within their defined scope;
5. approval leases must not override hard security denies;
6. if no rule matches, use the configured default decision.

A useful conceptual ordering is:

```text
hard DENY
    ↓
explicit scoped DENY
    ↓
matching temporary/persistent ALLOW lease
    ↓
static ASK / ALLOW rules
    ↓
default policy
```

This ordering should be encoded explicitly in tests.

## Hard denies

Some rules may be classified as hard denies.

A hard deny exists to protect the Hermes Gate control plane or another invariant that the agent must not be able to override through ordinary approval.

Examples may include direct modification of:

```text
Hermes Gate protected policy
Hermes Gate broker configuration
hardening launcher configuration
```

Hard denies are different from normal user policy denies.

A hard deny cannot be converted into an allow lease from the normal approval UI.

Changing a hard deny requires changing the trusted policy outside the Hermes agent authority path.

## Specificity

Where multiple ordinary rules match, Hermes Gate should prefer the most specific rule.

Specificity may consider dimensions such as:

```text
exact tool
exact executable
exact resource
exact path
exact argv
prefix match
wildcard match
```

For example:

```text
ASK all sudo
```

combined with:

```text
ALLOW sudo systemctl status llama-swap.service
```

should allow only the more specific operation.

Specificity must be defined mechanically rather than guessed.

## Deny semantics

A normal `DENY` rule means the user has chosen not to permit that class of operation.

Example:

```yaml
- id: no-force-push
  match:
    operation: git.force_push
  decision: deny
```

A future approval flow may allow the user to modify the underlying persistent rule outside the blocked request.

The blocked operation itself should not silently override a deny.

## ASK semantics and leases

An `ASK` rule is satisfied if an active matching approval lease exists.

Example policy:

```text
sudo → ASK
```

Active lease:

```text
exact request:
sudo systemctl restart llama-swap.service

lifetime:
SESSION
```

Results:

```text
sudo systemctl restart llama-swap.service
    → ALLOW through lease

sudo reboot
    → ASK
```

A broader lease might instead authorize:

```text
operation = service.manage
resource = llama-swap.service
lifetime = SESSION
```

which permits multiple service-management commands against that same resource.

## Persistent rule amendments

When the user chooses an always-allow option, Hermes Gate should create a readable persistent rule.

Example original request:

```text
systemctl restart llama-swap.service
```

A weak persistent amendment would be:

```text
allow systemctl
```

This is too broad.

A preferred amendment is:

```text
allow service.manage
resource = llama-swap.service
```

where typed classification is reliable.

If typed classification is unavailable, Hermes Gate may fall back to an exact or command-prefix rule.

Persistent rule suggestions should prefer the narrowest useful scope.

## Native Hermes guardrails

Hermes Gate policy does not replace Hermes policy.

If Hermes Gate returns:

```text
ALLOW
```

Hermes may still:

```text
ask
block
checkpoint
rewrite
reject
```

according to Hermes' own behavior.

Hermes Gate must never interpret its own `ALLOW` as permission to force Hermes to execute something Hermes would normally block.

Conversely, Hermes Gate `DENY` prevents the request from reaching execution even if Hermes itself would have allowed it.

## Command normalization

Exact-command leases require a stable normalized representation.

Normalization may include:

```text
executable resolution where appropriate
argv tokenization
cwd normalization
path normalization
removal of irrelevant formatting differences
```

Normalization must not alter command meaning.

Raw command text must always remain available for audit and approval display.

Normalization should remain conservative.

## Path handling

Filesystem policy must operate on normalized paths.

The engine should account for:

```text
relative paths
absolute paths
~
.
..
symlinks where resolution is appropriate
case behavior on Windows-backed filesystems
WSL mount paths
```

A policy decision must not depend on naïve string-prefix comparison alone.

For example:

```text
/home/imi/docs
```

must not accidentally match:

```text
/home/imi/docs-backup
```

Path policy should use path-aware containment logic.

## Ambiguous requests

When Hermes Gate cannot confidently classify a request, it must not invent a more permissive interpretation.

Possible fallback behavior includes:

```text
use executable-level policy
use raw tool policy
ASK
defer to Hermes native guardrails
```

The configured policy determines the fallback.

Ambiguity itself is not automatically equivalent to danger.

The goal is conservative interpretation without turning normal use into constant approval prompts.

## Rule explanation

Every non-trivial decision should be explainable.

The engine should be able to report:

```text
final decision
matched rule
matched lease
specificity
reason
scope
```

Example:

```text
Decision: ALLOW
Source: session lease
Rule: sudo-default
Scope: service.manage
Resource: llama-swap.service
```

This information is useful both in approval UI and audit logs.

## Audit relationship

Policy evaluation should emit structured decision metadata.

At minimum:

```text
request identifier
session identifier
task identifier if available
matched rule
matched lease
decision
decision source
reason
```

The policy engine itself should not be responsible for storing the audit log.

It should return enough structured information for the audit subsystem to record the decision.

## Example policies

### Normal terminal command

Policy:

```text
terminal default → ALLOW
```

Request:

```text
git status
```

Result:

```text
ALLOW
```

Hermes native guardrails still run afterward.

### Privileged command

Policy:

```text
sudo → ASK
```

Request:

```text
sudo systemctl restart llama-swap.service
```

No matching lease:

```text
ASK
```

User selects:

```text
Allow exact request for session
```

Later identical request:

```text
ALLOW
```

Different sudo request:

```text
ASK
```

### Scoped service lease

Policy:

```text
service management → ASK
```

User grants:

```text
service.manage
resource = llama-swap.service
lifetime = SESSION
```

Requests:

```text
restart llama-swap.service → ALLOW
status llama-swap.service  → ALLOW
restart docker.service      → ASK
```

### Hard control-plane deny

Policy:

```text
write Hermes Gate protected policy → hard DENY
```

Request:

```text
write ~/.config/hermes-gate/protected-policy.yaml
```

Result:

```text
DENY
```

No normal approval option is offered.

### Persistent rule

Policy:

```text
git push → ASK
```

User approves:

```text
git push
repo = /home/imi/dev/hermes-gate
lifetime = PERSISTENT
```

Future matching pushes:

```text
ALLOW
```

Pushes from unrelated repositories continue to follow their own policy.

## Initial implementation constraints

The first implementation does not need every concept in this document.

M1 should support:

```text
ALLOW / ASK / DENY
static rules
exact tool matching
exact executable matching
argv prefix matching
path matching
one-time approval
session approval
persistent rules
deterministic precedence
explainable decisions
```

Task leases, typed resources, richer shell classification, and suggested semantic scopes may be added later.

The core semantics should remain compatible with this model as those features are introduced.

## Guiding principles

The policy model follows several principles:

1. authorization decisions are deterministic;
2. `ALLOW` never bypasses Hermes native guardrails;
3. the model cannot authorize itself;
4. persistent permissions remain inspectable;
5. temporary approval has explicit scope and lifetime;
6. broader convenience must not silently weaken hard security boundaries;
7. exact approvals should stay exact unless the user explicitly chooses a broader scope;
8. semantic scopes are preferred over broad executable prefixes when reliable;
9. uncertain parsing must not be presented as certainty;
10. ordinary safe behavior should remain low-friction.
