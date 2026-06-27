# TableLB — Backend & Functional Plan

**Lebanon's Premier Lifestyle Discovery Platform**

## Structure

```
/docs
  database_schema.md       — Full PostgreSQL/Supabase schema
  api_reference.md         — REST API endpoints
  auth_and_roles.md        — Authentication & RBAC
  lovable_prompts.md       — Step-by-step Lovable build prompts
/supabase
  /migrations
    001_initial_schema.sql — Full executable SQL
BACKEND_PLAN.md            — Master backend plan
```

## Tech Stack

| Layer | Technology |
|-------|------------|
| Database | Supabase (PostgreSQL 15) |
| Auth | Supabase Auth (Email + Google + Apple) |
| Storage | Supabase Storage |
| Edge Functions | Supabase Edge Functions (Deno/TypeScript) |
| Full-text Search | PostgreSQL tsvector + pg_trgm |
| Semantic Search | pgvector (Phase 4) |
| Real-time | Supabase Realtime |
| Frontend | React + TypeScript + Tailwind + shadcn/ui |
| Payments | Stripe |
| Maps | Google Maps API |
| Email | Resend |
| WhatsApp/SMS | Twilio |
| AI | OpenAI (Phase 4) |

## Implementation Phases

| Phase | Scope |
|-------|-------|
| 1 — MVP | Venues, search, profiles, auth, saved venues, basic reviews |
| 2 — Business | Reservations, dashboard, admin CMS, events, offers, collections |
| 3 — Growth | Analytics, payments, featured placements, notifications |
| 4 — AI | Smart search, recommendations, planners, voice search |
