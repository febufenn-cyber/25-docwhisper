# LLD — docwhisper

## Architecture (request flow)

```
Ingestion (sync, page/chunk-capped for v1):
Browser (magic-link auth)
   |  POST /api/documents (multipart PDF upload)
   v
Worker (src/handlers.ts)
   |-- enforce cap: <=30 pages / <=150 chunks (reject larger uploads with a clear error)
   |-- extractText(pdf) -> unpdf (reused from the contract-reviewer reference)
   |-- chunkText(text) -> ~500-token chunks, ~50-token overlap
   |-- for each chunk: embed(chunk) -> Workers AI `@cf/baai/bge-base-en-v1.5` (AI binding)
   |-- insert doc_chunks rows (content + embedding vector(768))
   |-- update documents.status = 'ready'
   v
Browser polls GET /api/documents/:id -> 'ready'

Query (sync, single request):
Browser
   |  POST /api/chat {question, document_ids?}
   v
Worker
   |-- embed(question) -> Workers AI (same bge model as ingestion — must match)
   |-- pgvector similarity search (top-k=5) scoped to org_id [+ document_ids filter]
   |-- buildAnswer(question, chunks) -> Anthropic claude-sonnet-4-6, json_schema
   |-- insert chat_messages row
   v
Browser renders {answer, citations:[{document, page}]}
```

## Data model (Supabase Postgres, `pgvector` extension, RLS on)

```sql
create extension if not exists vector;

create table public.orgs (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  created_at timestamptz not null default now()
);

create table public.memberships (
  org_id uuid not null references public.orgs(id) on delete cascade,
  user_id uuid not null references auth.users(id) on delete cascade,
  role text not null default 'member' check (role in ('owner','member')),
  primary key (org_id, user_id)
);
alter table public.memberships enable row level security;
create policy "read own memberships" on public.memberships for select
  using (auth.uid() = user_id);

create table public.documents (
  id uuid primary key default gen_random_uuid(),
  org_id uuid not null references public.orgs(id) on delete cascade,
  uploaded_by uuid not null references auth.users(id),
  filename text not null,
  storage_path text not null,
  status text not null default 'pending' check (status in ('pending','ready','failed')),
  page_count int, chunk_count int, error text,
  created_at timestamptz not null default now()
);
alter table public.documents enable row level security;
create policy "org members read documents" on public.documents for select
  using (org_id in (select org_id from public.memberships where user_id = auth.uid()));

create table public.doc_chunks (
  id uuid primary key default gen_random_uuid(),
  document_id uuid not null references public.documents(id) on delete cascade,
  org_id uuid not null,                    -- denormalized for a single-hop RLS check
  chunk_index int not null,
  page int,
  content text not null,
  embedding vector(768) not null,          -- bge-base-en-v1.5 dimension
  created_at timestamptz not null default now()
);
alter table public.doc_chunks enable row level security;
create policy "org members read chunks" on public.doc_chunks for select
  using (org_id in (select org_id from public.memberships where user_id = auth.uid()));
create index doc_chunks_embedding_idx on public.doc_chunks
  using hnsw (embedding vector_cosine_ops);

create table public.chat_messages (
  id uuid primary key default gen_random_uuid(),
  org_id uuid not null,
  user_id uuid not null references auth.users(id),
  question text not null,
  answer jsonb,
  model text, input_tokens int, output_tokens int,
  created_at timestamptz not null default now()
);
alter table public.chat_messages enable row level security;
create policy "org members read messages" on public.chat_messages for select
  using (org_id in (select org_id from public.memberships where user_id = auth.uid()));

insert into storage.buckets (id, name, public) values ('documents','documents',false)
on conflict (id) do nothing;
```

All writes go through the service-role key from the Worker; RLS above governs client-side reads only (dashboard listing, debugging via Supabase Studio).

## API routes

| Route | Method | Auth | Behavior | Failure modes |
|---|---|---|---|---|
| `/config.js` | GET | none | Public Supabase URL/anon key | — |
| `/api/documents` | POST | Supabase JWT (org member) | Ingest PDF: extract, chunk, embed, store (sync, capped size) | too large (413), extraction failure -> `failed` (500) |
| `/api/documents` | GET | Supabase JWT | List org's documents + status | — |
| `/api/documents/:id` | GET | Supabase JWT | Single document status/detail | not found (404) |
| `/api/chat` | POST | Supabase JWT | Embed question, retrieve, answer with citations | no documents ready (409), Claude failure (502) |
| `/api/me` | GET | Supabase JWT | Org + membership + seat info | — |

## LLM strategy

- **Embeddings**: Cloudflare Workers AI, `@cf/baai/bge-base-en-v1.5` (768-dim), used identically for chunk embedding at ingestion and question embedding at query time — mismatched models between the two sides would silently break retrieval, so this is a single constant referenced from both paths, not two separate configs.
- **Generation**: Anthropic **claude-sonnet-4-6**, `output_config: {format:{type:"json_schema", schema: ANSWER_SCHEMA}}` — same strict-JSON pattern as the reference. System prompt instructs: answer only from the provided chunks, cite `{document, page}` per claim, say "not covered in the uploaded documents" rather than guessing.
- Output schema (sketch): `{answer: string, citations: [{document: string, page: number}], covered: boolean}`.
- Cost estimate: embeddings via Workers AI are billed in Neurons, sub-paisa per chunk at this volume — effectively negligible. Generation: ~5 chunks x ~500 tokens + question ≈ 2.5k input, ~400 output tokens/query ≈ $0.02 (≈ Rs2/query) — comfortably inside a Rs499/seat/mo budget for normal usage.
- **Sync vs async — ingestion**: v1 caps documents at ~30 pages / ~150 chunks specifically so embedding all chunks fits inside Cloudflare's ~100s response budget (each Workers AI embed call is fast, but 150 sequential/batched calls plus PDF extraction adds up) — this keeps ingestion a single synchronous request with no callback plumbing. Larger documents need queue-based async ingestion (Cloudflare Queues), which is **out of scope for v1** — see below.
- **Sync — query**: a single embed + pgvector search + one Claude call is well within the response budget; no async needed here at all.

## Frontend pages

`index.html` (org signup/login), `documents.html` (upload, list, status), `chat.html` (ask a question, see cited answer with expandable source chunks), `team.html` (invite members, seat count), `tos.html` (disclaimer: answers are retrieval-augmented and can be wrong; always verify against the source document for anything consequential).

## Error handling + billing flow

This is **per-seat subscription billing, not per-operation credits** — a deliberate divergence from the reference's `spend_credit`/`refund_credit` RPC pattern, which doesn't fit a seat-based product. Access is gated by `orgs.active` (or membership existing at all) rather than a balance. Ingestion failure marks the document `failed` with a visible error and no billing implication (v1 is flat monthly per seat, Razorpay subscription post-KYC per the portfolio-wide plan). Query failure surfaces a retry-able error; no credit accounting involved.

## Integrations and launch gates

- **Workers AI binding** (`ai` in `wrangler.jsonc`) must be enabled on the account — no external OAuth or API partnership needed, it's a first-party Cloudflare binding.
- No other third-party integrations required for v1.
- Razorpay subscription checkout is gated on KYC completion (portfolio-wide constraint); manual UPI billing for early customers in the meantime.

## Security notes

- RLS is **org/membership-scoped**, not `user_id = auth.uid()` like the reference — every table carries `org_id` and policies join through `memberships`, since documents are team-shared, not solo-owned.
- Secrets via `wrangler secret put`: `ANTHROPIC_API_KEY`, `SUPABASE_SERVICE_ROLE_KEY`. Workers AI needs no secret, just the `ai` binding.
- Input validation: PDF-only MIME check, page/chunk cap enforced before any embedding calls (reject early, don't partially ingest then fail).
- Storage bucket `documents` is private; access only via signed URLs issued by the Worker to org members.

## Out of scope for v1

Async/queued ingestion for large documents (Cloudflare Queues), non-PDF formats (docx/html/markdown), per-document fine-grained sharing permissions (v1 is org-wide only), self-hosting any existing RAG framework (explicitly ruled out), reranking models, chat history search/export, multi-language embeddings beyond the base bge model's coverage.
