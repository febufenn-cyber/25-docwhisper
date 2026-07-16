# CLAUDE.md

This repo is product #25 of Febin's 50-SaaS challenge. Nothing is built yet — `docs/` is the source of truth. Read `docs/LLD.md` then `docs/PLAN.md` and execute tasks in order with TDD.

Stack conventions + the shipped reference implementation live in the private repo `febufenn-cyber/50-saas` (`contract-reviewer/`) — same Worker+Supabase+provider patterns, adapted here to org/membership-scoped RLS and per-seat billing instead of solo-user credits.

The embeddings decision (Workers AI `bge`, not Claude — Claude has no embeddings endpoint), the no-self-hosted-RAG-framework decision, and the sync-with-a-page-cap ingestion decision in the LLD are verified — do not re-litigate without evidence.
