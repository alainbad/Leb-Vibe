# API Reference

All standard data access uses Supabase's auto-generated REST API (PostgREST).  
Custom business logic uses Supabase Edge Functions.

**Base URL:** `https://<project-ref>.supabase.co`  
**REST:** `/rest/v1/<table>`  
**Edge Functions:** `/functions/v1/<function-name>`  
**Auth header:** `Authorization: Bearer <supabase-jwt>`

---

## Venues

### List venues (with filters)
```
GET /rest/v1/venues
```

Query params:
```
status=eq.active
venue_type=cs.{restaurant}           -- contains
area_id=eq.<uuid>
price_range=in.(budget,moderate)
avg_rating=gte.4
is_featured=eq.true
editors_pick=eq.true
is_new_opening=eq.true
hidden_gem=eq.true
order=avg_rating.desc,review_count.desc
limit=20&offset=0
select=id,name,slug,cover_image_url,avg_rating,review_count,price_range,area_id,venue_type,is_featured
```

### Get full venue profile by slug
```
GET /rest/v1/venues?slug=eq.<slug>&select=*,
  area:areas(name_en,slug),
  images:venue_images(url,caption,is_cover,sort_order),
  amenities:venue_amenities_map(amenity:amenities(name_en,icon,slug)),
  cuisines:venue_cuisine_types(cuisine),
  tags:venue_tags(tag),
  hours:venue_operating_hours(*),
  menus(*,sections:menu_sections(*,items:menu_items(*))),
  events(*),
  offers(*),
  staff_count:venue_staff(count)
```

### Trending venues
```
GET /rest/v1/venues?status=eq.active
  &select=id,name,slug,cover_image_url,avg_rating,venue_type,area_id
  &order=reservation_count.desc,view_count.desc
  &limit=12
```

### Homepage sections (examples)
```
-- Editor's Picks
GET /rest/v1/venues?editors_pick=eq.true&status=eq.active&limit=8

-- Hidden Gems
GET /rest/v1/venues?hidden_gem=eq.true&status=eq.active&avg_rating=gte.4&limit=8

-- New Openings
GET /rest/v1/venues?is_new_opening=eq.true&status=eq.active&order=created_at.desc&limit=8

-- Rooftops
GET /rest/v1/venues?status=eq.active&select=...&venue_tags=like.*rooftop*
(use JOIN filter via amenity slug)
```

### Submit new venue (owner registration)
```
POST /rest/v1/venues
Content-Type: application/json

{
  "name": "Al Falamanki",
  "venue_type": ["restaurant", "cafe"],
  "description": "...",
  "area_id": "<uuid>",
  "address": "Clemenceau, Beirut",
  "phone": "+9611234567",
  "whatsapp": "+9611234567",
  "price_range": "moderate"
}
-- status defaults to 'pending', owner_id set to auth.uid() via RLS
```

### Update venue (owner)
```
PATCH /rest/v1/venues?id=eq.<uuid>
{ "description": "...", "instagram": "@handle", ... }
```

---

## Search

### Smart search (Edge Function)
```
POST /functions/v1/smart-search
Content-Type: application/json

{
  "query": "romantic sea view restaurant beirut",
  "filters": {
    "category": null,
    "area_id": null,
    "price_range": null,
    "amenities": [],
    "open_now": false,
    "min_rating": null
  },
  "sort": "relevance",
  "page": 1,
  "limit": 20
}
```

Response:
```json
{
  "venues": [...],
  "total": 23,
  "page": 1,
  "parsed_filters": {
    "detected_category": "restaurant",
    "detected_tags": ["romantic", "sea_view"],
    "detected_area": "beirut"
  }
}
```

### Full-text search (direct PostgREST)
```
GET /rest/v1/venues?status=eq.active
  &search_vector=fts.romantic%20dinner
  &order=avg_rating.desc
  &limit=20
```

---

## Reservations

### Check slot availability (Edge Function)
```
POST /functions/v1/check-availability

{
  "venue_id": "<uuid>",
  "date": "2024-08-15",
  "party_size": 4
}
```

Response:
```json
{
  "available_slots": ["18:00", "18:30", "19:00", "20:30"],
  "fully_booked": ["19:30", "20:00"],
  "venue_closed": false
}
```

**Logic inside Edge Function:**
1. Get venue operating hours for that day
2. Get active reservation_slots for that day
3. Count confirmed reservations per slot for that date
4. Return slots where `confirmed_count + party_size <= max_covers`

### Create reservation
```
POST /rest/v1/reservations

{
  "venue_id": "<uuid>",
  "reservation_date": "2024-08-15",
  "reservation_time": "19:00",
  "party_size": 4,
  "contact_name": "John Doe",
  "contact_phone": "+961XXXXXXXX",
  "contact_email": "john@example.com",
  "special_requests": "Birthday celebration",
  "indoor_outdoor": "indoor",
  "occasion": "birthday"
}
-- confirmation_code auto-generated, status = 'pending'
```

### User's reservations
```
GET /rest/v1/reservations?user_id=eq.<uid>
  &select=*,venue:venues(name,slug,cover_image_url,area:areas(name_en))
  &order=reservation_date.desc
```

### Venue's reservations (dashboard)
```
GET /rest/v1/reservations?venue_id=eq.<vid>
  &reservation_date=eq.2024-08-15
  &status=in.(pending,confirmed)
  &order=reservation_time.asc
```

### Update reservation status (venue)
```
PATCH /rest/v1/reservations?id=eq.<uuid>
{ "status": "confirmed", "confirmed_at": "now()" }
```

### Cancel reservation (user)
```
PATCH /rest/v1/reservations?id=eq.<uuid>
{ "status": "cancelled", "cancelled_at": "now()", "cancellation_reason": "Change of plans" }
```

---

## Reviews

### Get venue reviews
```
GET /rest/v1/reviews?venue_id=eq.<vid>&status=eq.approved
  &select=*,user:profiles(username,avatar_url),media:review_media(url)
  &order=created_at.desc
  &limit=20&offset=0
```

### Rating breakdown for a venue
```
GET /rest/v1/reviews?venue_id=eq.<vid>&status=eq.approved
  &select=rating_overall,rating_food,rating_service,rating_ambiance,rating_value
```
(Calculate breakdown client-side or via Edge Function)

### Create review
```
POST /rest/v1/reviews

{
  "venue_id": "<uuid>",
  "reservation_id": "<uuid>",
  "rating_overall": 4.5,
  "rating_food": 5.0,
  "rating_service": 4.0,
  "rating_ambiance": 4.5,
  "rating_value": 4.0,
  "title": "Amazing experience!",
  "body": "The food was incredible...",
  "visit_date": "2024-08-10"
}
```

### Venue reply to review
```
PATCH /rest/v1/reviews?id=eq.<uuid>
{ "venue_reply": "Thank you so much!", "venue_reply_at": "now()" }
```

### Admin approve/reject review
```
PATCH /rest/v1/reviews?id=eq.<uuid>
{ "status": "approved", "approved_by": "<admin_uid>", "approved_at": "now()" }
```

---

## Events

### Upcoming events (platform-wide)
```
GET /rest/v1/events?status=eq.upcoming
  &start_datetime=gte.<now>
  &select=*,venue:venues(name,slug,cover_image_url,area:areas(name_en))
  &order=start_datetime.asc
  &limit=20
```

### Events for a specific venue
```
GET /rest/v1/events?venue_id=eq.<vid>&status=eq.upcoming
  &order=start_datetime.asc
```

### Create event (venue owner)
```
POST /rest/v1/events

{
  "venue_id": "<uuid>",
  "title": "Live Jazz Night",
  "description": "...",
  "event_type": "live_music",
  "start_datetime": "2024-08-15T20:00:00+03:00",
  "end_datetime": "2024-08-15T23:30:00+03:00",
  "is_free": true,
  "cover_image_url": "..."
}
```

### RSVP to event
```
POST /rest/v1/event_attendees
{ "event_id": "<uuid>", "status": "going" }
```

---

## Offers

### Active offers for a venue
```
GET /rest/v1/offers?venue_id=eq.<vid>&is_active=eq.true
  &valid_until=gte.<today>
  &order=created_at.desc
```

### All platform-wide active offers
```
GET /rest/v1/offers?is_active=eq.true&valid_until=gte.<today>
  &select=*,venue:venues(name,slug,cover_image_url)
  &order=created_at.desc
  &limit=20
```

### Create offer (venue owner)
```
POST /rest/v1/offers

{
  "venue_id": "<uuid>",
  "title": "20% off Tuesdays",
  "offer_type": "discount",
  "discount_percent": 20,
  "days_of_week": ["tuesday"],
  "valid_from": "2024-08-01",
  "valid_until": "2024-12-31"
}
```

---

## Collections

### All active collections
```
GET /rest/v1/collections?is_active=eq.true
  &select=*
  &order=sort_order.asc
```

### Collection with venues
```
GET /rest/v1/collections?slug=eq.<slug>
  &select=*,
  venues:collection_venues(
    position,
    venue:venues(id,name,slug,cover_image_url,avg_rating,review_count,price_range,venue_type,area:areas(name_en))
  )
```

### Add venue to collection (admin/editor)
```
POST /rest/v1/collection_venues
{ "collection_id": "<uuid>", "venue_id": "<uuid>", "position": 1 }
```

---

## Magazine

### List published articles
```
GET /rest/v1/articles?status=eq.published
  &select=id,title,slug,excerpt,cover_image_url,category,published_at,read_time_min,author:profiles(username,avatar_url)
  &order=published_at.desc
  &limit=12
```

### Get single article
```
GET /rest/v1/articles?slug=eq.<slug>&status=eq.published
  &select=*,
  author:profiles(username,avatar_url,bio),
  tags:article_tags(tag),
  mentioned_venues:article_venues(venue:venues(id,name,slug,cover_image_url,avg_rating))
```

---

## User Account

### Get own profile
```
GET /rest/v1/profiles?id=eq.<uid>&select=*
```

### Update profile
```
PATCH /rest/v1/profiles?id=eq.<uid>
{ "full_name": "...", "bio": "...", "preferred_area": "..." }
```

### Saved venues
```
-- List
GET /rest/v1/saved_venues?user_id=eq.<uid>
  &select=saved_at,venue:venues(id,name,slug,cover_image_url,avg_rating,price_range)
  &order=saved_at.desc

-- Save
POST /rest/v1/saved_venues
{ "venue_id": "<uuid>" }

-- Unsave
DELETE /rest/v1/saved_venues?user_id=eq.<uid>&venue_id=eq.<uuid>
```

### Notifications
```
-- List
GET /rest/v1/notifications?user_id=eq.<uid>
  &order=created_at.desc
  &limit=30

-- Mark all read
PATCH /rest/v1/notifications?user_id=eq.<uid>&is_read=eq.false
{ "is_read": true }

-- Unread count
GET /rest/v1/notifications?user_id=eq.<uid>&is_read=eq.false&select=count
```

---

## Analytics

### Track an event (client-side)
```
POST /rest/v1/analytics_events

{
  "venue_id": "<uuid>",
  "event_type": "view",
  "session_id": "abc123",
  "metadata": { "source": "search", "query": "sushi beirut" }
}
```

**event_type values:** `view`, `reservation_click`, `whatsapp_click`, `maps_click`, `menu_click`, `offer_click`, `phone_click`, `instagram_click`

### Dashboard analytics (Edge Function)
```
GET /functions/v1/venue-analytics?venue_id=<uuid>&period=30d
```

Response:
```json
{
  "summary": {
    "total_views": 1250,
    "total_reservations": 47,
    "reservation_completion_rate": 0.87,
    "avg_rating": 4.3,
    "review_count": 28,
    "whatsapp_clicks": 89,
    "maps_clicks": 103,
    "menu_clicks": 215
  },
  "views_by_day": [
    { "date": "2024-07-28", "count": 42 }
  ],
  "reservations_by_day": [
    { "date": "2024-07-28", "count": 3 }
  ],
  "top_sources": [
    { "source": "search", "count": 430 },
    { "source": "collection", "count": 210 }
  ]
}
```

---

## Admin Endpoints

All require `role IN ('admin', 'super_admin')` enforced by RLS.

### Pending venue approvals
```
GET /rest/v1/venues?status=eq.pending
  &select=*,owner:profiles(username,email,full_name)
  &order=created_at.asc
```

### Approve venue
```
PATCH /rest/v1/venues?id=eq.<uuid>
{
  "status": "active",
  "approved_at": "now()",
  "approved_by": "<admin_uid>"
}
```

### Reject venue
```
PATCH /rest/v1/venues?id=eq.<uuid>
{
  "status": "pending",
  "rejection_reason": "Please add more photos and a complete menu."
}
```

### Pending review moderation queue
```
GET /rest/v1/reviews?status=eq.pending
  &select=*,user:profiles(username,avatar_url),venue:venues(name,slug)
  &order=created_at.asc
```

### User management
```
-- List users
GET /rest/v1/profiles?select=*&order=created_at.desc&limit=50

-- Ban user
PATCH /rest/v1/profiles?id=eq.<uuid>
{ "is_banned": true, "ban_reason": "Spam reviews" }

-- Promote to editor
PATCH /rest/v1/profiles?id=eq.<uuid>
{ "role": "editor" }
```

### Platform-wide analytics (Edge Function)
```
GET /functions/v1/admin-analytics?period=30d
```

Response includes: total venues (active/pending), new users, total reservations, total reviews, top venues by views, top venues by reservations.

---

## Edge Function Specs

### `check-availability`
- **Method:** POST
- **Auth:** Optional (rate-limit guests)
- **Logic:** Query reservation_slots for venue+day, query confirmed reservations for venue+date, return available slots

### `smart-search`
- **Method:** POST
- **Auth:** Optional
- **Logic:** Parse query text → extract category/area/tags/price → build PostgREST filter → run full-text + filter query → rank results

### `on-reservation-created`
- **Trigger:** Supabase DB webhook on `reservations` INSERT
- **Logic:** Send email via Resend + WhatsApp via Twilio to contact_phone. If venue.auto_confirm = true, immediately update status to 'confirmed'.

### `reservation-reminder`
- **Trigger:** Cron every 15 minutes
- **Logic:** Query reservations WHERE status='confirmed' AND reminder_sent=false AND reservation_datetime BETWEEN now() AND now()+2hours. Send reminder. Set reminder_sent=true.

### `venue-analytics`
- **Method:** GET
- **Auth:** Required (venue owner or admin)
- **Logic:** Aggregate analytics_events + reservations + reviews for venue, return structured metrics

### `process-payment`
- **Method:** POST
- **Auth:** Required
- **Logic:** Create Stripe customer if needed, create subscription or payment intent, return client_secret for frontend Stripe.js
