# docwhisper

Chat-with-your-docs for small teams: upload PDFs, get cited answers pulled from your own policy and process documents.

**Status: planned — not yet built (50-SaaS challenge #25)**

## The problem

SMB teams have their real operating knowledge scattered across policy PDFs, SOPs, and process docs that nobody reads until they need one specific answer — and search-by-Ctrl+F across a shared drive doesn't cite where the answer came from.

## Target buyer

SMB teams (5-50 people) with a shared library of policy/process documents — HR, ops, compliance-adjacent, not engineering docs.

## Pricing hypothesis

Rs499/seat/mo.

## Stack summary

Cloudflare Worker (TypeScript) + Supabase Postgres with `pgvector` for retrieval, Workers AI for embeddings (`bge` model family), and a synchronous Claude call for the cited answer. Org/team-scoped RLS, not solo-user — documents are shared within an org, matching the per-seat pricing model.

## How to continue this build

Nothing is implemented yet. Read `docs/LLD.md` for the architecture and data model, then `docs/PLAN.md` for the ordered TDD task list, then follow `CLAUDE.md` for how this repo relates to the shipped reference implementation.

## Risks / Constraints

- **The Anthropic/Claude API has no embeddings endpoint.** Embeddings must come from Cloudflare Workers AI (`@cf/baai/bge-*` models) — do not attempt to use Claude for the embedding step.
- **Do not self-host Onyx** (or similar) as the RAG engine — its `ee/` directories are proprietary/source-available, not open source, and would poison the license story of this repo.
- The verified path is a **thin RAG-from-scratch build**: chunk, embed via Workers AI, store in `pgvector`, retrieve top-k, answer with Claude citing chunks. No off-the-shelf RAG framework.
- v1 caps document size (page/chunk count) so ingestion stays synchronous and inside Cloudflare's ~100s response cap; larger async ingestion is a v2 item, not built here.
- Cited answers can still be wrong — this is retrieval-augmented, not verified-correct, and that disclaimer must ship in the product.
