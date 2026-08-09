# Design: Team Workspace AI

**Project:** `team-workspace-ai`  
**Parent system design:** `10-global-realtime-product-surface.md / 04 / 07`

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

Narrated with **ElevenLabs Debpro voice** and Debpro still image (via [GitaProject](/Users/deb/Development/GenAI/GitaProject)):

- Video: [`video/design-overview.mp4`](./video/design-overview.mp4)
- Script: [`video/narration.txt`](./video/narration.txt)

