# 07 — Agent Runtime with Hard Containment

**Prompt:** Design an agent platform that can use tools (code, browser, files, APIs) while bounding blast radius when the model is wrong or adversarially manipulated.

**Rank:** Top 10 (#07)

## Use cases

| Use case | Who | Why this design matters |
|----------|-----|-------------------------|
| Code interpreter / data analysis | Chat with Python tools | Micro-VM, CPU/disk quotas, no host secrets |
| Web-browsing research agent | Search + summarize agents | Egress allowlists; treat page text as untrusted |
| SaaS workflow automation | CRM, email, tickets | OAuth-scoped tools; human confirm for side effects |
| DevOps / SRE assistant | Cloud ops agents | Hard deny on prod-destructive actions without approval |
| Computer-use / UI agents | Desktop or browser control | Strong sandbox + session kill switches |

---

## 1. Clarify requirements

### Functional
- Plan / act loop: model proposes tool calls → runtime executes → observe → repeat.
- Tools: code interpreter, web browser, connectors, retrieval, outbound HTTP.
- User approvals for sensitive actions (optional per policy).
- Session memory + artifacts (files, plots).

### Non-functional
| Concern | Target |
|---------|--------|
| Containment | Hard boundaries independent of model text |
| Latency | Interactive first token still reasonable; tools add seconds |
| Isolation | Per-session sandbox; no cross-tenant filesystem |
| Audit | Full tool trace for debugging & abuse |

### Unacceptable failures
- Model talks its way into production credentials
- Cross-tenant file read
- Unbounded egress (SSRF to cloud metadata)
- Fork bomb / crypto miner in code tool
- Silent privilege escalation via prompt injection

---

## 2. Core principle

> **Never trust the model as a security boundary.**  
> Permissions live in the runtime. The model may *request*; the runtime *authorizes*.

This is the principal-level thesis to lead with.

---

## 3. High-level architecture

```
User → Agent Orchestrator (state machine)
          → Model (proposes structured tool calls)
          → Policy Engine (allow / deny / ask user)
          → Tool Brokers
                → Code: micro-VM / gVisor / Firecracker
                → Browser: isolated profile + network policy
                → Connectors: OAuth-scoped tokens (user-held)
                → HTTP: allowlisted egress proxy
          → Observation normalizer → back to model
          → Audit log + artifact store
```

---

## 4. Deep dive: sandbox & egress

### Code interpreter
| Control | Why |
|---------|-----|
| Micro-VM per session | Strong isolation vs containers alone |
| CPU / mem / time / disk quotas | DoS & cost |
| No host mounts | Prevent escape to secrets |
| Outbound via proxy only | Block metadata IP `169.254.169.254`, internal CIDRs |
| Fresh VM or golden image reset | State bleed |

### Browser tool
- Ephemeral profile; cleared cookies between sessions (or strict session scope).
- Network allowlist / deny internal; strip dangerous downloads into quarantine.
- Treat page content as **untrusted input** (prompt injection vector).

### Connector tokens
- Store tokens in a secret service; inject short-lived, scoped credentials per call.
- Model never sees refresh tokens.
- User-visible permission grants; revoke path.

### Egress proxy policy example
- Deny link-local & RFC1918 by default.
- Allow only approved domains for fetch tool.
- Size limits; response truncation before model context.

---

## 5. Policy engine

```text
tool_call(name, args, session) ->
  if not in tool_allowlist(role): DENY
  if args violate schema: DENY
  if risk(score) high: ASK_USER or DENY
  if rate_limit exceeded: DENY
  else ALLOW with rewritten args (e.g. injected credentials)
```

Risk features: tool type, destination, data classification, user tier, prior violations.

---

## 6. Prompt injection & untrusted observations

- Label tool outputs as untrusted in the conversation schema.
- Separate “instructions” channels from “data” channels where possible.
- Critical actions (email send, $ transfer, delete) require **out-of-band user confirmation**, not model judgment.
- RAG/browser text cannot elevate privileges.

---

## 7. Orchestration patterns

| Pattern | Use |
|---------|-----|
| Single-shot tool | Simple assistants |
| ReAct loop with max steps | General agents |
| DAG planner | Deterministic workflows |
| Human-in-the-loop gates | Enterprise |

Always: `max_steps`, `max_wall_time`, `max_tool_cost` circuit breakers.

---

## 8. Scale 10× / 100× / 1000×

| Scale | Breakage | Fix |
|-------|----------|-----|
| 10× sessions | VM density / cold start | Pool warm VMs; snapshot boot |
| 100× tool QPS | Proxy & broker bottlenecks | Shard brokers; regional cells |
| 1000× | Image sprawl, CVE patching | Immutable base images; automated fleet patch |

---

## 9. Observability & abuse

- Trace every tool call: latency, bytes, deny reasons.
- Detect crypto mining patterns, spam sends, credential stuffing via tools.
- Kill switch: disable tool class globally in minutes.

---

## 10. Multi-year bet

**Bet:** Invest in a **shared containment substrate** (micro-VM + egress policy + capability tokens) used by all agent products—rather than each team wrapping Docker “carefully.” Pair it with a policy engine that makes user confirmation mandatory for high-impact side effects.

**Why:** Agent capability will grow faster than model reliability; containment must be capability-agnostic.

---

## 11. 60-second summary

Authorize tools in a policy engine, execute in hard sandboxes with deny-by-default egress, never let model text equal permission, treat tool outputs as untrusted, and force human confirmation for high-blast-radius actions—bounded by step/time/cost kill switches.
