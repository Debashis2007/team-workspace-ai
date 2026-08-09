# 04 — RAG / Document Retrieval with Live Embeddings

**Prompt:** Design a document retrieval system with embedding updates for enterprise search / Assistants-style RAG at high freshness and high QPS.

**Rank:** Top 10 (#04)

## Use cases

| Use case | Who | Why this design matters |
|----------|-----|-------------------------|
| Enterprise knowledge assistant | Company wiki, Drive, Notion | ACL-aware retrieval; freshness on doc edits |
| Customer support copilot | Helpdesk + ticket history | Hybrid search + citations; low P99 retrieve |
| Legal / compliance Q&A | Contract and policy corpora | Exact citations, strict tenancy, audit trail |
| Personalized consumer memory | User files / notes | Per-user indexes; privacy isolation |
| Product documentation chat | Public + private docs | Live embed updates when docs publish |

---

## 1. Clarify requirements

### Functional
- Ingest docs (uploads, connectors: Drive, Notion, web).
- Chunk, embed, index; retrieve top-k for grounded generation.
- Near-real-time updates: create / update / delete reflected in search.
- Multi-tenant isolation; ACL-aware retrieval (critical for enterprise).
- Citations back to source spans.

### Non-functional
| Metric | Target (example) |
|--------|------------------|
| Query P99 | ≤ 200 ms retrieval (before LLM) |
| Freshness | ≤ 30–60 s for metadata; ≤ few min for re-embed |
| Recall@k | Product-defined; measure with eval harness |
| Tenancy | Hard isolation; zero cross-tenant hits |
| Scale | Billions of chunks; tens of thousands of tenants |

### Scale axes
- Docs ingested/day, chunk count, query QPS, embedding dim, update rate.

### Unacceptable failures
- Serving another tenant’s document
- Stale ACL allowing access after revoke
- Embedding/index skew (doc in DB but not vector index)
- Unbounded re-embed storms on model upgrade

---

## 2. High-level architecture

```
Connectors / Upload → Ingest API → Parse & ACL extract
                    → Chunker → Embedding Workers
                    → Vector Index + Lexical Index + Doc Meta Store
                    → Query Service (hybrid retrieve + rerank + ACL filter)
                    → LLM Orchestrator (grounded generation)
```

### Stores
| Store | Role |
|-------|------|
| Object store | Raw bytes |
| Meta DB | Doc ID, version, ACL, checksum, status |
| Lexical index | BM25 / keyword |
| Vector index | ANN (HNSW, IVF, DiskANN-class) |
| Feature log | Retrieved IDs for eval / debugging |

---

## 3. Ingest & chunking

1. Normalize → extract text (PDF/HTML) → language detect.
2. Chunk with overlap; store `doc_id`, `chunk_id`, `version`, offsets for citations.
3. Embed with versioned embedding model id (`emb_v3`).
4. Write meta first (`status=indexing`), then indexes, then `status=ready`.
5. Deletes: tombstone in meta; async remove from indexes (idempotent).

**Consistency model:** eventual between meta and indexes is OK if query path **filters by meta ACL + ready version**. Never return a vector hit without meta authorization.

---

## 4. Deep dive: live updates & embedding model upgrades

### Per-doc updates
- Content hash change → rechunk changed regions if possible, else full re-embed.
- Use generation numbers: queries only accept `chunk.version == meta.current_version`.

### Embedding model migration
- Dual-write or dual-index: build `index_emb_v4` in background.
- Query with shadow traffic; compare recall; flip read pointer.
- Rate-limit backfill to protect online cluster.

### Freshness vs cost
> “Re-embedding an entire tenant corpus on every tiny edit is unaffordable. Hash-based change detection + chunk-level invalidation is the default; full re-embed only on embedding model change or corruption repair.”

---

## 5. Query path (hybrid + ACL)

```
Query → embed query → ANN top-N + BM25 top-N
     → merge → ACL filter (mandatory)
     → cross-encoder rerank top-m
     → return chunks + citations
```

### ACL
- Prefer **filter early** with tenant_id + doc ACL bitsets / posting lists.
- For sensitive enterprise: **per-tenant indexes** or highly partitioned indexes beat giant shared indexes with brittle filters.
- Cache query embeddings carefully; **never cache across users** without ACL keying.

### Rerank
- Cross-encoder improves precision; budget 20–50 ms.
- Skip rerank under load for free tier (degrade gracefully).

---

## 6. Multi-tenancy isolation options

| Design | Pros | Cons |
|--------|------|------|
| Shared index + ACL filter | Cheap | Higher leak risk if bugs |
| Partition by tenant | Stronger isolation | Hot tenants; many small indexes |
| Dedicated index for large tenants | Perf + isolation | Ops complexity |

**Principal recommendation:** hybrid—shared pools for SMB; dedicated partitions for large enterprise; always ACL-filter as defense in depth.

---

## 7. Scale 10× / 100× / 1000×

| Scale | Breakage | Fix |
|-------|----------|-----|
| 10× QPS | Query embed + ANN CPU/GPU | Cache popular query embeds; replica indexes |
| 100× docs | Memory for HNSW | Disk-based ANN; hierarchical IVF; tier cold tenants |
| 1000× tenants | Control plane / small-index overhead | Tenant packing; cell-based index service |

---

## 8. Eval & quality loop

- Golden sets per domain; track Recall@k, nDCG, citation accuracy, refusal on missing evidence.
- Online: thumbs-down → sample retrieval traces.
- Detect embedding drift when switching models.

---

## 9. Safety & privacy

- Respect ACLs on every hop; log access for audit.
- Don’t send another user’s chunks into the prompt.
- PII: optional redaction before embed for some products.
- Prompt injection via retrieved docs: treat retrieved text as **untrusted**; constrain tool use; cite and sandbox.

---

## 10. Multi-year bet

**Bet:** **Versioned embedding indexes + ACL-first query planner** as the core abstraction. Separate “document truth” (meta store) from “retrieval acceleration” (ANN). Invest in migration tooling for embedding upgrades—these become yearly, not once.

---

## 11. 60-second summary

Ingest with versioned chunks, dual lexical+vector indexes, and mandatory ACL filtering on the query path. Optimize for update locality and embedding migrations without downtime; isolate large tenants; evaluate retrieval quality continuously so RAG doesn’t silently rot.
