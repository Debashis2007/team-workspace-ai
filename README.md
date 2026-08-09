# Use Case: Team / Workspace AI Product

**Design doc:** [docs/DESIGN.md](./docs/DESIGN.md) — architecture, patterns, and why.


**Parent system design:** [10 — Global Realtime Product Surface](../10-global-realtime-product-surface.md)  
**Also references:** [04 — RAG](../04-rag-embedding-pipeline.md), [07 — Agents](../07-agent-runtime-containment.md)

## Users & problem

Teams share AI threads in a workspace with roles, shared docs, and connectors. ACL on conversations and knowledge is the core risk.

## Requirements & SLOs

| Requirement | Target |
|-------------|--------|
| Authz | Workspace roles on threads/assets |
| Knowledge | Shared corpus with ACL ([04](../04-rag-embedding-pipeline.md)) |
| Admin | Audit, retention, SSO |
| Residency | Workspace region pin |

## Design (from parent)

```
Workspace → SSO/roles → shared threads
  → generation with workspace policy packs ([06](../06-safety-moderation-pipeline.md))
  → optional connectors via contained agents ([07](../07-agent-runtime-containment.md))
  → audit export for admins
```

## Specializations

| Concern | Workspace choice |
|---------|------------------|
| Sharing | Thread vs workspace visibility |
| Billing | Seat + usage hybrid |
| DLP | Block exfil via tools/emails |
| Isolation | Enterprise cell / private cloud options |

## Failure modes

- Private thread leaked via search → index ACL with thread perms.
- Contractor over-access → least-privilege roles; SCIM deprovision.
- Residency break via cross-region sync → home-region primary store.



## Run (self-contained POC)

This folder is a **standalone** project (safe to split into its own GitHub repo).

```bash
cd team-workspace-ai
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
PYTHONPATH=. python -m uvicorn app.main:app --reload --port 8000
```

```bash
curl -s http://127.0.0.1:8000/health | jq
```

curl -s -X POST http://127.0.0.1:8000/workspaces/w1/threads -H 'Content-Type: application/json' -d '{"user":"alice","role":"member","text":"summarize"}' | jq
