# PLAN — docwhisper

TDD, one task per focused session. Each task lists files touched, the interfaces it produces, the test to write first, and done-criteria.

## T1 — Supabase schema

Files: `supabase/migrations/0001_init.sql`
Interfaces: `pgvector` extension; tables `orgs`, `memberships`, `documents`, `doc_chunks`, `chat_messages`; storage bucket `documents`.
Test: no vitest — verify via `supabase db push` + two orgs/two users; confirm org A's member cannot read org B's `documents`/`doc_chunks`/`chat_messages`.
Done: migration applies cleanly; `hnsw` index created; RLS confirmed across two orgs.

## T2 — PDF extraction + chunking

Files: `src/ingest.ts`
Interfaces: `extractText(pdfBytes): Promise<{pages: string[]}>` (via `unpdf`, reused pattern from the reference's `src/pdf.ts`); `chunkText(pages: string[]): Chunk[]` (~500 tokens, ~50 overlap, page-tagged).
Test first: `test/ingest.test.ts` — fixture PDF -> asserts page count, chunk count within cap, chunk boundaries respect the overlap setting.
Done: rejects (throws a typed error) documents over the page/chunk cap before any embedding call is made.

## T3 — Embeddings client

Files: `src/embed.ts`
Interfaces: `embedText(text: string, ai: Ai): Promise<number[]>` (Workers AI `@cf/baai/bge-base-en-v1.5`), `EMBED_MODEL` constant used by both ingestion and query paths.
Test first: `test/embed.test.ts` — mocked `Ai` binding, asserts correct model string passed, asserts 768-length vector returned.
Done: single shared constant/function used by both T4 (ingestion) and T6 (query) — a test asserts both import the same `EMBED_MODEL`.

## T4 — Ingestion handler

Files: `src/handlers.ts` (`handleUploadDocument`), `src/supa.ts` (document/chunk writes)
Interfaces: `handleUploadDocument(req, env): Promise<Response>`.
Test first: `test/ingest-handler.test.ts` — mocked Supabase writes + mocked `embedText`, asserts a document row is created `pending` then flipped to `ready` with correct `chunk_count`, and `failed` with an error on extraction failure.
Done: end-to-end mocked flow covers the cap-rejection (413), extraction-failure, and success paths from the LLD's API table.

## T5 — Retrieval + answer generation

Files: `src/retrieve.ts`, `src/llm.ts`
Interfaces: `retrieveChunks(orgId, questionEmbedding, opts): Promise<ChunkMatch[]>` (pgvector query via PostgREST RPC or a `select ... order by embedding <=> $1 limit k`), `buildAnswer(question, chunks): Promise<{answer, citations, covered, model, inputTokens, outputTokens}>`.
Test first: `test/retrieve.test.ts` (mocked PostgREST response), `test/llm.test.ts` (mocked Anthropic response asserting `json_schema` output_config and that the system prompt forbids answering outside the provided chunks).
Done: `covered:false` case tested (question has no matching chunks -> model told to say so, not guess).

## T6 — Chat handler + router + worker entry

Files: `src/handlers.ts` (`handleChat`, `handleListDocuments`, `handleGetDocument`, `handleMe`), `src/router.ts`, `src/worker.ts`
Interfaces: wires T3-T5 together behind `POST /api/chat`.
Test first: `test/handlers.test.ts`, `test/router.test.ts` — covers no-ready-documents (409), Claude failure (502), org-scoped auth on every route.
Done: every failure path in the LLD's API table has a test.

## T7 — Frontend

Files: `public/index.html`, `public/documents.html`, `public/chat.html`, `public/team.html`, `public/tos.html`, `public/app.js`, `public/styles.css`
Test: manual QA checklist (upload a small PDF, wait for `ready`, ask a question in-scope and one out-of-scope, confirm citations link to the right page, confirm ToS disclaimer visible).
Done: chat UI clearly distinguishes "answered from your docs" vs. "not covered."

## T8 — Deploy + live smoke + launch checklist

Files: `wrangler.jsonc` (with `ai` binding), `scripts/smoke.ts`
Test: `scripts/smoke.ts` runs against the deployed Worker end-to-end (upload a known test PDF, poll to `ready`, ask a question with a known answer, assert the citation matches) — this is the ship gate.
Done: `wrangler deploy` succeeds; smoke passes; Workers AI binding confirmed live; secrets set (`ANTHROPIC_API_KEY`, `SUPABASE_SERVICE_ROLE_KEY`); pricing page + disclaimer live.

## Notes for whoever picks this up

Seat-count billing (Razorpay subscription tied to `memberships` count) is intentionally not a numbered task — it's a manual UPI process for the first customers per the portfolio-wide plan, and only needs automating once there's real subscription volume to justify it. Don't build subscription-seat-sync ahead of that need.
