# TableLB — Master Backend Plan

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                    Client Layer                      │
│         React + TypeScript + Tailwind CSS            │
│              (Lovable-generated UI)                  │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│                   API Layer                          │
│     Supabase Auto-generated REST API (PostgREST)     │
│     + Supabase Edge Functions (custom logic)         │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│                  Data Layer                          │
│         PostgreSQL 15 (via Supabase)                 │
│         Row Level Security (RLS)                     │
│         Full-text Search (tsvector + pg_trgm)        │
│         pgvector (AI embeddings — Phase 4)           │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│              External Services                       │
│   Stripe | Google Maps | Twilio | Resend | OpenAI   │
└─────────────────────────────────────────────────────┘
```

## 2. Supabase Project Setup

### Extensions to Enable
```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";
CREATE EXTENSION IF NOT EXISTS "unaccent";
CREATE EXTENSION IF NOT EXISTS "vector"; -- pgvector for Phase 4
```

### Storage Buckets

| Bucket | Contents | Access |
|--------|----------|--------|
| `venue-images` | Cover + gallery photos | Public |
| `menu-images` | Menu item photos | Public |
| `user-avatars` | Profile pictures | Public |
| `article-images` | Magazine content | Public |
| `review-media` | Review photos | Public |
| `event-media` | Event banners | Public |

---

## 3. User Roles

| Role | Description |
|------|-------------|
| `guest` | Unauthenticated — browse only |
| `user` | Registered — reviews, reservations, saves |
| `venue_staff` | Manage reservations for their venue |
| `venue_owner` | Full access to their venue dashboard |
| `editor` | Manage Magazine, Collections, Events |
| `admin` | Full platform management |
| `super_admin` | Unrestricted access |

---

## 4. Core Modules

### Module 1 — Venue Management

**Entities:** venues, venue_images, venue_amenities_map, venue_operating_hours, venue_branches, venue_cuisine_types, venue_tags

**Business Rules:**
- Venues start `pending` and require admin approval before going `active`
- Branches are separate venue records linked by `parent_venue_id`
- Operating hours support split shifts (e.g., 12:00–16:00 and 19:00–02:00)
- `is_new_opening` auto-set true for venues created in last 60 days (trigger)
- `featured_until` date controls homepage featured placement

**Venue Statuses:** `pending` → `active` | `suspended` | `closed`

---

### Module 2 — Search & Discovery

**Search Stack:**
1. `venues.search_vector` — tsvector combining name (A), description (B), address (C)
2. `pg_trgm` — trigram similarity for typo tolerance
3. Filters: category, area, price_range, amenities, open_now, rating, cuisine
4. Sort: relevance, rating, most reviewed, newest

**Natural Language Mapping (Edge Function):**

| Query | Parsed Filters |
|-------|---------------|
| "Sea view restaurant" | category=restaurant, amenity=sea_view |
| "Live music tonight" | event_type=live_music, event_date=today |
| "Romantic dinner" | tag=romantic OR collection=date_night |
| "Outdoor seating" | amenity=outdoor_seating |
| "Cheap burgers" | price_range=budget, search=burger |
| "Best sushi" | cuisine=japanese, sort=rating |

**Homepage Discovery Sections:**

| Section | Query Logic |
|---------|-------------|
| Trending | ORDER BY (reservation_count * 2 + view_count) DESC, last 7 days |
| Editor's Picks | editors_pick = true |
| Hidden Gems | avg_rating >= 4.0 AND review_count < 30 |
| New Openings | created_at > NOW() - INTERVAL '60 days' |
| Rooftops | amenity = rooftop |
| Summer Spots | amenity IN (beach, pool) |
| Romantic | tag = romantic |
| Best Breakfast | serves_breakfast = true ORDER BY avg_rating DESC |

---

### Module 3 — Reservations

**Flow:**
1. User selects date, time slot, party size
2. Edge Function checks availability (slot capacity minus existing confirmed bookings)
3. User submits contact details + preferences
4. System creates reservation `pending`
5. Auto-confirm (if venue setting on) OR venue manually confirms
6. Confirmation sent via email + WhatsApp
7. Reminder sent 2 hours before (cron edge function)

**Rules:**
- Slots are 30-minute intervals
- Default booking window: 30 days ahead
- Cancellation cutoff: configurable per venue (default 2 hours)
- No-shows tracked; flagged after 2 occurrences
- Commission tracked per reservation

**Statuses:** `pending` → `confirmed` | `cancelled`; `confirmed` → `completed` | `no_show`

---

### Module 4 — Reviews

**Rules:**
- Only authenticated users can post reviews
- One review per user per venue (DB unique constraint)
- 4 sub-ratings: Food, Service, Ambiance, Value (each 1–5)
- Overall rating auto-calculated as average of sub-ratings
- Up to 5 photos per review
- New reviews enter `pending` status (admin can set auto-approve per venue tier)
- Venues can post one reply per review
- Users can flag reviews; admins resolve flags
- `avg_rating` and `review_count` on venues auto-updated by DB trigger

**Moderation:** `pending` → `approved` | `rejected`

---

### Module 5 — Events

**Event Types:** live_music, dj_night, themed_dinner, sports_screening, ladies_night, trivia, comedy, brunch, other

**Fields:** title, description, venue, start/end datetime, type, ticket_price, capacity, RSVP count, cover image, recurring flag + RRULE

**RSVP vs Ticketed:** Free events use in-app RSVP. Paid events link to external ticket URL.

---

### Module 6 — Offers

**Types:** `discount` (%), `fixed_price` (set menu), `free_item`, `package`

**Fields:** title, description, terms, valid_from/until, days_of_week, promo_code, max_uses, use_count, cover_image

**Display:** Offer badge on venue card when active offer exists. Offer detail on venue profile.

---

### Module 7 — Collections

Admin-curated lists. Each collection has an ordered list of venues.

**Default Collections:** Top 50 Restaurants, Hidden Gems, Best Pizza, Best Sushi, Best Breakfast, Rooftops, Date Night, Family Friendly, Luxury Dining, Late Night, Live Music

**Sponsored collections** have sponsor name + logo shown at top.

---

### Module 8 — Magazine

**Article Types:** guide, review, feature, list, news

**Fields:** title, slug, excerpt, content (rich HTML), cover_image, author, category, tags, SEO meta, published_at, read_time_min

**Statuses:** `draft` → `published` | `archived`

**Venue Links:** Articles can reference venues (article_venues table), shown as "Mentioned in" section on venue profile.

---

### Module 9 — Restaurant Dashboard

| Section | Key Operations |
|---------|----------------|
| Overview | Metrics: views, reservations, rating, pending reviews |
| Reservations | Calendar + list view, confirm/cancel, party details |
| Menu Builder | CRUD sections + items, dietary flags, pricing |
| Offers | CRUD, activate/deactivate, track usage |
| Events | CRUD, RSVP management |
| Reviews | Read + reply (one reply per review) |
| Gallery | Upload, set cover, drag-to-reorder |
| Staff | Invite by email, set role (manager/staff/host), remove |
| Branches | Create/manage branch venues |
| Settings | Venue info, hours, reservation settings, subscription |

**Analytics Metrics:**
- Profile views (daily/weekly/monthly sparklines)
- Reservation count + completion rate
- Review count + avg rating trend
- Click-throughs: WhatsApp, Maps, Menu, Phone
- Top performing offers

---

### Module 10 — Admin CMS

| Section | Functionality |
|---------|---------------|
| Dashboard | Platform-wide KPIs |
| Venue Approval | Review pending venues, approve/reject with feedback |
| User Management | View, ban, promote users |
| Review Moderation | Pending review queue, approve/reject |
| Collections | CRUD + venue ordering |
| Featured Listings | Assign featured slots (time-bounded) |
| Ads | Create, schedule, target banners |
| Magazine | Full article CMS |
| Reports | Flag resolution |
| Analytics | Platform analytics |
| Support | Basic ticket management |

---

### Module 11 — AI Features (Phase 4)

| Feature | Implementation |
|---------|----------------|
| Personalized recs | User history → pgvector similarity |
| Smart search NLP | Edge Function: text → structured filters |
| Weekend planner | Multi-venue itinerary by preferences |
| Date planner | Dinner + activity romantic flow |
| Group planner | Handle diverse preferences + split bill |
| Travel assistant | Tourist mode: top picks by district |
| Voice search | Web Speech API → text → smart-search endpoint |

---

## 5. Revenue Implementation

| Stream | Implementation |
|--------|----------------|
| Premium listings | `venues.subscription_tier` (free/basic/premium/enterprise) |
| Monthly subscriptions | Stripe Subscriptions linked to venue |
| Featured placement | `featured_placements` table with date range |
| Reservation commissions | `commission_rate` per venue, tracked per reservation |
| Sponsored collections | `collections.is_sponsored` + sponsor metadata |
| Banner advertising | `ads` table with targeting + impression/click tracking |

---

## 6. Edge Functions

| Function | Trigger | Purpose |
|----------|---------|--------|
| `check-availability` | POST | Return available time slots for date + party size |
| `smart-search` | POST | NLP query → structured filters + ranked results |
| `on-reservation-created` | DB webhook | Send email + WhatsApp confirmation |
| `on-reservation-confirmed` | DB webhook | Notify user, schedule reminder |
| `reservation-reminder` | Cron every 15min | Send 2hr-before reminders |
| `on-review-approved` | DB webhook | Notify venue + user |
| `venue-approval-notify` | DB webhook | Email venue owner on approval/rejection |
| `process-payment` | POST | Stripe integration for subscriptions |
| `venue-analytics` | GET | Aggregate analytics for dashboard |
| `generate-recommendations` | GET | AI-powered personalized venue recs |
| `send-notification` | Internal | Generic notification dispatcher |

---

## 7. Real-time Subscriptions (Supabase Realtime)

- New reservation received → venue dashboard live update
- Reservation status changed → user notification
- Review approved → venue dashboard badge update
- New review posted → venue owner notification

---

## 8. Implementation Phases

### Phase 1 — MVP (Core Platform)
1. DB schema + RLS policies + triggers
2. Auth (Email + Google)
3. Venue listings + search + filters
4. Venue detail page (full profile)
5. Homepage sections
6. User accounts + saved venues
7. Basic reviews (auto-approve for MVP)

### Phase 2 — Business Features
1. Reservation system + availability check
2. Restaurant dashboard (reservations, menu, gallery)
3. Admin CMS (approval, moderation)
4. Events module
5. Offers module
6. Collections (admin-managed)
7. Magazine

### Phase 3 — Growth
1. Full analytics dashboard
2. Stripe subscriptions + featured placements
3. Email + WhatsApp notifications
4. Review moderation workflow
5. Banner ads system
6. Venue staff management

### Phase 4 — AI & Scale
1. Smart search (NLP Edge Function)
2. pgvector embeddings for venues
3. Personalized recommendations
4. Weekend / date / group planner
5. Voice search
6. Mobile app (React Native)
