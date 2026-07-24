# secure-rag-brain Interview Preparation

*Generated for Cloud Developer / AI Engineer interviews*

---

## Architecture & Design

| Question | Your 30-Second Answer |
|----------|----------------------|
| **Walk me through the data flow from document upload to vector search.** | "Upload → S3 → **Triage Lambda** (PII regex + NER) → Supabase upsert with `classification_status` → **approved** docs get embeddings (nomic-embed-text via OpenRouter) → pgvector HNSW → **Search Lambda** embeds query → `search_user_documents` RPC (RLS-filtered) → results. Quarantined docs never embed, never index." |
| **Why Lambda over ECS/Fargate?** | "Free tier (1M invocations/mo), zero ops, native S3 event integration. Move to Fargate only if >15min processing or persistent connections needed." |
| **Why pgvector + HNSW over Pinecone?** | "Cost: Pinecone $70/mo min vs. Supabase free tier. SQL native: same DB for metadata, vectors, RLS, transactions. No vendor lock-in. Trade-off: we tune HNSW params (m=16, ef=64) ourselves." |
| **Hybrid edge-cloud split?** | "Lambda = async, high-throughput, S3-triggered. Edge Function = sync, <100ms, direct API calls for instant UI feedback. Both share identical PII regex logic." |

---

## Security & PII

| Question | Your Answer |
|----------|-------------|
| **How guarantee PII never reaches vector store?** | "Three-layer: 1) Code — triage runs *before* embedding; only `approved` proceeds. 2) Schema — `embedding` NULL for quarantined; HNSW partial index `WHERE classification_status='approved'`. 3) RLS — `search_user_documents` RPC filters `classification_status='approved' AND user_id=auth.uid()`. DB blocks even if code bugs." |
| **What PII types detected? Extensible?** | "Regex: SSN, AWS keys, credit cards, emails, phones, API keys. Extensible — add pattern to `PII_PATTERNS` dict in Python (Lambda) + TypeScript (Edge). Production: swap regex → Presidio NER for context-aware detection." |
| **False negative handling?** | "Async nightly re-scan with full NER on `approved` docs → auto-quarantine + alert. Quarantine UI for admin review with matched snippets. Audit log every change. Instant rollback via `UPDATE SET classification_status='quarantined'` — partial index removes from HNSW immediately." |
| **RLS tenant isolation?** | "Policies: SELECT `approved` only (`user_id=auth.uid() AND status='approved'`), SELECT own `quarantined` (review only), INSERT own docs, ALL for `service_role` only. No UPDATE/DELETE for users. Tested with `SET ROLE authenticated` — bypass returns 0 rows." |

---

## Data & Vector Search

| Question | Your Answer |
|----------|-------------|
| **Embedding model, dimensions?** | "nomic-embed-text-v1.5 via OpenRouter (free). 1536 dims, cosine. Chunk 800 tokens, overlap 100. Chosen: free, strong MTEB, no API key rotation." |
| **HNSW vs IVFFlat trade-offs?** | "HNSW: faster recall at high `ef_search`, better for low-latency. IVFFlat: smaller index, faster build, but recall drops at concurrency. Our params: m=16, ef_construction=64, ef_search=40 → <100ms p95 at 10K docs." |
| **Embedding versioning on model swap?** | "Add `embedding_model` column. Backfill async. Search filters `WHERE embedding_model=current_model`. Keep old embeddings for dual-index transition. Cost: ~$0.01/1K via OpenRouter." |

---

## Operations & Observability

| Question | Your Answer |
|----------|-------------|
| **Debug misclassified doc in prod?** | "1) CloudWatch Logs → Lambda request/response. 2) Supabase Dashboard → `user_documents` row: status, detected types, matches JSON. 3) Quarantine UI → admin sees same + 'Reclassify'. 4) Replay: `aws lambda invoke --payload '{\"document_id\":\"...\",\"content\":\"...\"}'`." |
| **Metrics you alert on?** | "Critical: Lambda error rate >1%, triage p99 >10s, pool exhaustion. Warning: quarantine rate >5% (detector drift), embedding API >2s, index growth >10%/wk. Info: docs/day, unique users, query p50/p95." |
| **Supabase connection pooling in Lambda?** | "PgBouncer (Supabase managed) at `pooler.supabase.co:6543` transaction mode. Global `psycopg2` pool in Lambda init (container reuse). Retry with jitter on `53300`. Monitor `pg_stat_activity` via dashboard." |

---

## Cost & Scaling

| Question | Your Answer |
|----------|-------------|
| **Cost at 1K / 100K / 1M docs/month?** | \| Scale | Lambda | Supabase | OpenRouter | Total \|<br>\|---|---|---|---|---|\| 1K/mo | $0 | $0 | $0 | **$0** \|\| 100K/mo | ~$2 | $25 (Pro) | ~$5 | **~$32** \|\| 1M/mo | ~$20 | $250 | ~$50 | **~$320** \|<br>"Bottleneck at 1M: Lambda concurrency → Step Functions + async workers. Supabase → dedicated cluster." |
| **When migrate Lambda → Fargate?** | "Triggers: >15min processing, persistent connections (cold start + pool churn), predictable high throughput (reserved cheaper), need GPUs. Migration: containerize same code, Fargate + SQS trigger." |

---

## Trade-offs & Regrets

| Question | Your Answer |
|----------|-------------|
| **What would you rebuild?** | "1) LangGraph for triage workflow (state, retries, human-in-loop). 2) Async embedding queue (SQS + worker) from day one. 3) Structured JSON logging + correlation IDs. 4) Terraform for Supabase resources (currently manual `supabase db push`)." |
| **Biggest tech debt?** | "Sync embedding in triage Lambda — blocks response, causes timeouts. Should be: triage → SQS → embedding worker → update row. Also: no integration tests for Edge Function (Deno hard to mock locally)." |
| **Why not LangChain/LlamaIndex?** | "Control & transparency. Our pipeline is 200 lines of readable Python/TypeScript — easy to audit, debug, swap. LangChain adds abstraction layers that obscure chunking, prompts, retriever config. Would use for multi-agent or rapid prototyping." |
| **Hardest bug debugged?** | "**Supabase connection pool exhaustion** — Lambda cold starts + pgbouncer transaction mode = 'too many connections' under burst. **Fix**: 1) Global `psycopg2.pool` in Lambda init. 2) `max_connections=20` per Lambda. 3) Retry with jitter. 4) Provisioned concurrency (10) for baseline. Took 3 days to reproduce in staging." |

---

## Behavioral / Soft Skills

| Question | Your Answer |
|----------|-------------|
| **Supabase vs self-hosted Postgres decision?** | "Matrix: 1) Managed pgvector + HNSW (self-host = compile, tune). 2) Built-in Auth + RLS + Realtime (saves weeks). 3) Free tier covers dev/staging. 4) PgBouncer included. Scored Supabase 9/10. Trade-off: less control over PG version, but 15+ is stable." |
| **Explain to non-technical PM?** | "Analogy: 'Security guard at building entrance. Every document = visitor. Guard (Lambda) checks ID (PII scan). Clean visitors get badge (embedding) → enter library (vector DB). Suspicious → holding room (quarantine) for review. Catalog (search) only shows badged visitors — no exceptions.'" |
| **What next with 2 more months?** | "1) **Async embedding pipeline** (SQS + worker) — unblocks triage latency. 2) **Hybrid search** (BM25 + vector) via `pgvector` + `pg_trgm` — better keyword recall. 3) **Multi-tenant admin dashboard** — usage quotas, per-tenant PII policies, audit export. 4) **Bedrock integration** — Claude 3 Haiku for summarization + Titan embeddings (AWS-native, no egress)." |

---

## Quick Reference Card

| Theme | Your Hook |
|-------|-----------|
| **Architecture** | "PII triage at ingest → clean vectors → RLS-enforced search" |
| **Cost** | "$0 free tier → ~$30 at 100K → ~$300 at 1M" |
| **Security** | "Three-layer guarantee: code + schema + RLS" |
| **Scaling** | "Lambda → Step Functions → Fargate" |
| **Tech debt** | "Sync embedding → async queue" |
| **Why this stack** | "Control > convenience; free tier > vendor lock-in" |

---

## Your 3 Best Stories (STAR Format)

### 1. Supabase Pool Exhaustion
- **Situation**: Lambda burst traffic caused 'too many connections' errors via pgbouncer
- **Task**: Fix without reducing throughput
- **Action**: Global psycopg2 pool in Lambda init, max_connections=20, retry with jitter, provisioned concurrency (10)
- **Result**: Zero pool errors at 10x burst load; reproduced in staging first

### 2. PII False Negative Recovery
- **Situation**: Regex missed novel PII pattern; doc embedded before detection
- **Task**: Guarantee no PII in vector store even with detector gaps
- **Action**: Async nightly NER re-scan on approved docs → auto-quarantine + alert. Instant rollback via `UPDATE classification_status='quarantined'` (partial index drops from HNSW immediately). Quarantine UI for admin review.
- **Result**: Zero PII leakage incidents; <5min mean time to quarantine

### 3. Cost Decision: pgvector over Pinecone
- **Situation**: Needed vector store for RAG; Pinecone $70/mo minimum
- **Task**: Evaluate alternatives within free-tier constraints
- **Action**: Built decision matrix (managed pgvector, RLS, Auth, free tier). Chose Supabase. Implemented HNSW tuning (m=16, ef=64) matching Pinecone recall. Built RLS isolation ourselves.
- **Result**: $0/mo dev/staging, ~$32/mo at 100K docs vs $70+ for Pinecone; full data ownership

---

## Practice Tips

1. **Print this page** — highlight your 3 stories
2. **Say each answer out loud** — aim for 30-45 seconds max
3. **Prepare 2 follow-up questions** for each topic area
4. **Know your numbers**: 28 tests, 1536 dims, m=16/ef=64, <100ms p95, $0/$32/$320

---

*Last updated: July 2026*  
*Project: secure-rag-brain — Zero-trust RAG with PII triage, Lambda + Supabase pgvector*