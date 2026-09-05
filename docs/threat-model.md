# Threat Model

This document defines the security assumptions, protected assets, threat actors, failure modes, and security boundaries relevant to Hermes Gate.

Hermes Gate is designed for a persistent main agent running on a user's normal local environment.

The main agent intentionally retains broad access to user files, development tools, network services, and host resources. The goal is therefore not perfect isolation.

The goal is to prevent ordinary agent authority from silently escalating into more sensitive authority without passing through an explicit authorization boundary.

## Security objective

The primary security objective is:

> A mistake, prompt injection, malicious dependency, or compromised tool running under Hermes should not be able to silently gain authority beyond what the Hermes process tree has already been granted.

Hermes Gate is also intended to reduce the impact of destructive but non-privileged mistakes through scoped approval and recovery mechanisms.

It does not attempt to make arbitrary local execution risk-free.

## Protected assets

Hermes Gate is concerned with protecting several classes of assets.

### User data

Examples:

```text
documents
projects
Obsidian vaults
media
configuration
local databases
personal files
```

The main agent may legitimately need to read and modify these.

They are therefore not universally isolated from Hermes.

Protection instead comes from:

```text
authorization rules
scoped approvals
destructive-operation classification
filesystem checkpoints
backups
```

### Security control plane

These assets must receive stronger protection because modifying them could allow the agent to weaken or bypass its own restrictions.

Examples:

```text
Hermes Gate policy
Hermes Gate persistent approval state
Hermes Gate broker configuration
Hermes Gate launcher/hardening configuration

Agent Vault configuration
credential-management configuration
agenix private material
SSH private material

backup control configuration
```

The long-term architecture should prevent the Hermes process tree from directly modifying these assets.

### Privileged host authority

Examples:

```text
root privileges
sudo
raw disk access
system service modification
kernel/system configuration
```

These capabilities should eventually have one controlled route through Hermes Gate.

### Container host authority

Access to Docker or another privileged container daemon can effectively grant host-level authority.

Examples:

```text
/var/run/docker.sock
podman control sockets
containerd control interfaces
```

These must not remain available as alternative routes around Hermes Gate policy.

### Windows authority

On WSL, Windows interoperability provides another route outside the Linux process boundary.

Examples:

```text
powershell.exe
cmd.exe
arbitrary Windows executables
WSL interop sockets
Windows-side control utilities
```

Raw interoperability must eventually be treated as a capability rather than an unrestricted escape path.

### Credentials

Examples:

```text
API keys
SSH keys
GitHub credentials
provider tokens
signing keys
```

Hermes Gate does not replace Agent Vault or other credential systems.

However, direct access to credentials can turn otherwise limited agent access into remote authority and therefore belongs in the overall threat model.

## Trust assumptions

Hermes Gate does not fully trust the main agent.

It also does not fully trust software launched by the main agent.

This includes:

```text
Hermes itself
the active language model
plugins
MCP servers
shell commands
Python scripts
npm/pip packages
build systems
repositories
downloaded binaries
web content
tool output
```

The trusted computing base should eventually be kept significantly smaller.

Likely trusted components include:

```text
Hermes Gate policy engine
Hermes Gate broker
Hermes Gate hardening launcher
kernel namespace enforcement
Agent Vault security boundary
user approval decisions
```

Hermes native guardrails are treated as an additional useful defense, but Hermes Gate must not assume they are sufficient by themselves.

## Threat actors and failure sources

Hermes Gate is designed around several different sources of harmful behavior.

### Agent mistake

The model intends to perform a legitimate operation but produces the wrong command or parameters.

Example:

```text
intended:
rm -rf ~/Downloads/temp/*

generated:
rm -rf ~/Downloads/*
```

This is considered a likely class of failure even if any individual event is rare.

### Parsing or integration bug

Hermes, a plugin, shell wrapper, or another component interprets an operation differently from what was intended.

Examples:

```text
incorrect escaping
unexpected glob expansion
argument splitting
path normalization bug
shell parsing error
wrong target resource
```

The security model must not rely entirely on perfect parsing.

### Prompt injection

Untrusted content influences the agent into requesting an action that benefits the attacker rather than the user.

Sources may include:

```text
webpages
repository files
README instructions
issues
emails
MCP responses
tool output
generated documents
```

The model being convinced to perform an operation must not itself constitute authorization.

### Malicious dependency

The user or agent intentionally runs a normal-looking package or tool whose code performs additional malicious actions.

Examples:

```text
npm install script
Python package setup hook
malicious binary
build script
repository tooling
```

Hermes Gate cannot statically predict all behavior of arbitrary code.

Sensitive authority must therefore be protected at execution boundaries as well as at the initial command decision.

### Compromised integration

A Hermes plugin, MCP server, external helper, or other integration may itself be compromised or buggy.

It may attempt actions that were not directly requested by the language model.

### Malicious repository

A repository may contain instructions, hooks, scripts, dependencies, or binaries designed to exploit an agent that works on it.

Task agents should normally use stronger disposable isolation.

The main-agent threat model assumes that the user may nevertheless occasionally ask Hermes to interact with untrusted repositories.

### Upstream Hermes defect

Hermes itself may contain:

```text
approval bugs
tool-routing bugs
command parsing bugs
unsafe defaults
plugin bugs
checkpoint bugs
unexpected execution paths
```

Hermes Gate should therefore not treat Hermes internals as the only enforcement boundary.

## Primary threat scenarios

### Sensitive command requested directly

Example:

```text
sudo systemctl disable agent-guard
```

Expected defense:

```text
Hermes Gate policy
    ↓
ASK or DENY
```

Hermes Gate must evaluate the operation before normal execution.

### Prompt injection requests privilege escalation

Example:

```text
web content
    ↓
agent believes instruction
    ↓
sudo ...
```

Expected defense:

```text
model intent is insufficient
    ↓
Hermes Gate authorization still required
```

The security boundary should not depend on the model recognizing the injection.

### Child process invokes sudo

Example:

```text
python script.py
    ↓
os.system("sudo ...")
```

A `pre_tool_call` hook alone cannot see the later subprocess call.

Long-term expected defense:

```text
no_new_privs
+
raw sudo unavailable
+
broker-controlled privilege route
```

This prevents a child process from bypassing the original policy decision.

### Child process modifies Hermes Gate policy

Example:

```text
npm package install hook
    ↓
writes ~/.config/hermes-gate/policy...
```

Expected long-term defense:

```text
Hermes Gate control-plane paths mounted read-only
```

The agent should not be able to grant itself new authority by directly editing policy.

### Docker host escape

Example:

```text
docker run -v /:/host ...
```

If Hermes can directly access a privileged Docker socket, filesystem protections in the Hermes namespace can be bypassed by asking the Docker daemon to operate outside that namespace.

Expected long-term defense:

```text
raw Docker socket hidden
+
Docker operations routed through Hermes Gate broker
```

Read-only operations may be automatically allowed while host-equivalent operations may require approval or be denied.

### Windows interoperability bypass

Example:

```text
powershell.exe ...
```

A Windows process may perform effects that Linux-side filesystem or privilege controls do not observe.

Expected long-term defense:

```text
raw WSL interop unavailable inside Hermes tree
+
Windows execution routed through controlled broker
```

Ordinary Windows inspection may remain low-friction.

Sensitive Windows mutation may require explicit authorization.

### Process namespace escape

A process may attempt to reach resources exposed through other local processes or namespaces.

Potential surfaces include:

```text
/proc/<pid>/root
ptrace
inherited file descriptors
local control sockets
/run
/run/user/<uid>
```

The Hermes process tree should eventually receive a separate PID namespace and a fresh `/proc` where necessary.

### Local RPC escape

A local daemon outside the Hermes enforcement boundary may expose authority through a Unix socket.

Examples include:

```text
Docker
credential agents
system services
desktop automation services
custom local daemons
```

Every authority-bearing local socket reachable by Hermes should be explicitly classified as:

```text
safe
removed
read-only where possible
brokered
```

### Destructive ordinary filesystem command

Example:

```text
rm -rf ~/Documents/wrong-directory
```

This may still be within the legitimate ordinary-user authority of the main agent.

Expected defenses:

```text
ASK rule for broad destructive operations
+
automatic filesystem checkpoint where practical
+
external backups
```

This is an accepted limitation of keeping the main agent useful.

### Remote irreversible side effect

Examples:

```text
git push
send email
delete cloud resource
modify remote API state
purchase
publish content
```

Filesystem checkpoints cannot undo these operations.

Expected defense:

```text
authorization policy
+
resource-aware approval
+
scoped lease where repeated use is legitimate
```

### Credential exfiltration

A malicious process may attempt to read or transmit credentials.

Expected defense may include:

```text
Agent Vault
restricted credential files
protected control-plane paths
network policy where appropriate
```

Hermes Gate should avoid relying on secret values being present directly in the agent environment.

## Authority escalation model

The main security distinction is between ordinary authority and sensitive authority.

Ordinary authority includes capabilities the main agent needs frequently:

```text
read normal user files
modify normal projects
run compilers
run Python
run Node
use Git locally
access network services
read system state
```

Sensitive authority may include:

```text
root
Docker host authority
Windows execution
security-control modification
credential access
irreversible remote mutations
```

Sensitive authority should have one auditable route.

The desired property is:

> A process cannot avoid an ASK or DENY decision merely by using another executable, local daemon, subsystem, or child process that reaches the same authority.

## Authorization failure modes

### False allow

A dangerous request incorrectly matches an `ALLOW` rule.

Mitigations:

```text
Hermes native guardrails remain active
narrow deterministic matching
explicit deny precedence
hard enforcement around privileged capabilities
audit trail
```

### False ask

A harmless operation prompts unnecessarily.

Impact:

```text
user friction
approval fatigue
```

Mitigations:

```text
scoped leases
session grants
task grants
persistent rules
better typed resource classification
```

Avoiding approval fatigue is a security concern because excessive prompts encourage indiscriminate approval.

### False deny

A legitimate operation is blocked.

Impact is primarily usability.

Hermes Gate should prefer clear diagnostics and easily inspectable rules so false denies can be corrected safely.

### Parser ambiguity

A command cannot be confidently classified.

Hermes Gate should not invent semantic certainty.

Possible behavior may include:

```text
fall back to executable-level policy
ASK
defer to Hermes native guards
```

The exact policy is configuration-dependent.

## Approval threats

### Approval fatigue

Too many prompts can cause the user to approve operations without reading them.

Mitigation:

```text
task-scoped leases
session-scoped leases
resource-aware rules
safe operations allowed by default
```

### Over-broad persistent rule

Example:

```text
always allow sudo systemctl
```

when the intended authorization was only:

```text
manage llama-swap.service
```

Persistent rule suggestions should prefer the narrowest useful resource scope.

### Misleading approval description

The displayed operation may not accurately represent what will execute.

Approval UI should show:

```text
original tool
original command/arguments
normalized interpretation
matched rule
suggested scope
working directory
relevant target resource
```

The raw request must remain inspectable.

### Stale lease

A session/task authorization remains active longer than intended.

Leases require explicit lifetime ownership and cleanup.

Session leases should terminate with the session.

Task leases should terminate when the relevant task boundary ends.

## Checkpoint limitations

Hermes filesystem checkpoints are useful for recovering from some local filesystem mistakes.

They are not a general security boundary.

Checkpoint recovery may fail to cover:

```text
paths outside checkpoint scope
very broad home-directory operations
excluded files
external storage
Windows-side mutations
remote effects
system-level changes
```

Checkpointing must therefore supplement rather than replace authorization.

## Sandbox and namespace limitations

The planned permissive namespace is intentionally not a full isolation environment.

Hermes retains broad machine access.

The namespace exists to remove alternate routes around selected security decisions.

The following remain important limitations:

* the sandbox shares the host kernel;
* kernel vulnerabilities may bypass namespace enforcement;
* broad user-level filesystem authority remains broad;
* network access may remain unrestricted;
* OS-level hardening must account for authority-bearing local sockets;
* Windows-side protection requires Windows-side enforcement where relevant.

Highly hostile workloads should use disposable Docker or VM isolation rather than the main-agent environment.

## Out of scope

Hermes Gate does not claim to protect against:

### Kernel compromise

A working local kernel exploit can bypass userspace authorization mechanisms.

### Hypervisor or Windows kernel compromise

WSL and Windows ultimately depend on lower-level host security.

### Physical attacker

A user with physical access to the unlocked system is outside this project's intended threat model.

### Fully compromised trusted broker

If Hermes Gate's trusted broker or hardening component itself is compromised, its guarantees may no longer hold.

The project should minimize this trusted code surface.

### Complete prevention of user-data loss

Because the main agent intentionally has ordinary write authority, it may still corrupt or delete ordinary user data.

Backups remain necessary.

### Perfect semantic analysis

Hermes Gate cannot determine the complete behavior of arbitrary code before execution.

Security must not depend on doing so.

## Security invariants

The mature system should aim to maintain these invariants:

1. Hermes cannot directly modify Hermes Gate's protected control plane.
2. Hermes cannot acquire root authority except through the configured privileged broker.
3. Hermes cannot reach host-equivalent Docker authority except through the configured broker.
4. Hermes cannot use raw WSL interoperability when Windows execution is configured as brokered.
5. A model decision never counts as user authorization by itself.
6. `ALLOW` from Hermes Gate never bypasses Hermes' native safety checks.
7. `DENY` cannot be overridden by a less-specific temporary allow.
8. Approval leases have explicit scope and lifetime.
9. Security decisions are deterministic and auditable.
10. Ambiguous command semantics are not treated as proven safe.
11. Filesystem rollback is not treated as protection for irreversible external side effects.
12. The main agent retains broad ordinary capabilities unless there is a concrete security reason to broker or restrict them.

## Open questions

The current design still requires empirical testing and decisions around:

```text
WSL interop masking
drvfs read-only mount behavior
Docker broker design
PID namespace requirements
authority-bearing sockets under /run and /run/user
approval UI integration with Hermes
task-boundary detection
checkpoint coverage outside project directories
Windows-side control-plane protection
```

These should remain explicit until tested rather than being treated as solved assumptions.

## Threat-model philosophy

The project does not assume that every dangerous outcome can be predicted from the original command.

Instead, Hermes Gate uses multiple layers:

```text
intent authorization
        +
scoped user approval
        +
native Hermes guardrails
        +
hard protection of sensitive authority
        +
recovery for ordinary filesystem mistakes
```

The central security principle is:

> A surprising command may still cause an ordinary user-level mistake, but it should not silently turn that mistake into root authority, container-host authority, Windows authority, credential compromise, or modification of the security system itself.
