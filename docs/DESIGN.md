# Design: Team Workspace AI

**Project:** `team-workspace-ai`  
**Parent system design:** [10 — Global Realtime Product Surface](../10-global-realtime-product-surface.md) · [04 — RAG / Document Retrieval](../04-rag-embedding-pipeline.md) · [07 — Agent Runtime with Hard Containment](../07-agent-runtime-containment.md)

## 1. What this POC demonstrates

Workspace ACL on threads; contractors without role are denied.

## 2. Architecture (POC)

```text
POST /workspaces/{wid}/threads → role check → reply
```

## 3. Patterns used (and why)

| Pattern | Why used | Where in code |
|---------|----------|---------------|
| Workspace RBAC | Shared AI must honor org permissions. | `acls` map. |
| Deny by default | Unknown users get none. | `role=none` → 403. |
| Auditable thread append | Admin visibility into workspace AI use. | In-memory thread list. |

## 4. Key endpoints

`GET /health`, `POST /workspaces/{wid}/threads`

## 5. Tradeoffs / POC limits

No SSO/SCIM — static ACL map.

## 6. How to run

See the **Run (self-contained POC)** section in [`../README.md`](../README.md).

This folder is self-contained and can be published as its own GitHub repository.

## 7. Design walkthrough video

> **Watch on YouTube:** [Team Workspace Ai — System Design #Shorts](https://youtu.be/lgwZGeB3lnw)
>
> Direct link: **https://youtu.be/lgwZGeB3lnw**

Also available in-repo:
- GIF preview: [`video/design-overview.gif`](./video/design-overview.gif)
- MP4 download: [`video/design-overview.mp4`](./video/design-overview.mp4)
- Narration script: [`video/narration.txt`](./video/narration.txt)

---

**Copyright (c) 2026 Debashis Bhattacharjee. All Rights Reserved.**  
Unauthorized copying or redistribution of this material is prohibited.  
GitHub: [Debashis2007](https://github.com/Debashis2007)

