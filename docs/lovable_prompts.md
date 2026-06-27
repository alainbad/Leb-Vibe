# Lovable Build Prompts

Copy-paste these prompts into Lovable in order. Each builds on the previous.

---

## Prompt 1 — Project Setup + Database

```
Create a full-stack app called TableLB — Lebanon's premier restaurant and lifestyle 
discovery platform. The tagline is "Discover Lebanon. One Table at a Time."

Enable Supabase. Then create the following database setup:

1. Enable extensions: uuid-ossp, pg_trgm, unaccent

2. Create these enums:
   - user_role: user, venue_staff, venue_owner, editor, admin, super_admin
   - venue_type: restaurant, cafe, bar, pub, beach_club, nightlife, lounge
   - venue_status: pending, active, suspended, closed
   - price_range: budget, moderate, expensive, luxury
   - reservation_status: pending, confirmed, cancelled, completed, no_show
   - review_status: pending, approved, rejected
   - offer_type: discount, fixed_price, free_item, package
   - event_status: upcoming, ongoing, cancelled, completed
   - subscription_tier: free, basic, premium, enterprise
   - day_of_week: monday through sunday

3. Create these tables with full column definitions per the schema spec:
   profiles, areas, amenities, venues, venue_images, venue_amenities_map,
   venue_operating_hours, venue_cuisine_types, venue_tags,
   menus, menu_sections, menu_items,
   reservation_slots, reservations,
   reviews, review_media,
   events, event_attendees,
   offers,
   collections, collection_venues,
   articles, article_tags, article_venues,
   saved_venues, featured_placements, ads, venue_staff,
   notifications, analytics_events, venue_daily_stats, subscriptions

4. Add RLS on all tables. Key policies:
   - venues: public read for active, owners read own, admin read all
   - reservations: users see own, venue owners see their venue's
   - reviews: public sees approved, users see own, venue sees theirs
   - saved_venues, notifications: users see only their own

5. Add triggers:
   - on_auth_user_created: auto-create profiles row on signup
   - update_venue_search_vector: maintain tsvector on venue name/description
   - update_venue_rating: recalculate avg_rating and review_count when reviews change
   - set_new_opening_flag: is_new_opening = true if created within 60 days
   - update_updated_at: maintain updated_at on all relevant tables

The app uses TypeScript, React, Tailwind CSS, and shadcn/ui.
```

---

## Prompt 2 — Authentication

```
Add authentication to TableLB:

- Email/password signup and login
- Google OAuth
- The on_auth_user_created trigger auto-creates the profiles row
- Auth context (useAuth hook) accessible throughout the app exposing: user, profile, role, loading, signIn, signUp, signOut
- Login page at /login with email/password form + Google OAuth button
- Register page at /register with full_name, email, password fields + Google OAuth button  
- Protected route wrapper that checks auth and redirects to /login if not authenticated
- Role-based route protection: dashboard requires venue_owner/venue_staff/admin, admin panel requires admin/super_admin
- Redirect after login to the page user was trying to access, or homepage
- "Add Your Venue" link in navbar only visible to logged-in users
```

---

## Prompt 3 — Homepage

```
Build the TableLB homepage at /:

Hero Section:
- Full-width (100vw, 70vh) hero with a dark-overlay background image
- Centered headline: "Discover Lebanon. One Table at a Time."
- Subheadline: "Restaurants, cafés, bars, beach clubs and nightlife"
- Smart search bar below headline: single text input, full-width (max 700px), 
  with a search icon and placeholder "Try \"rooftop bar\" or \"romantic dinner\"..."
- On submit, navigate to /search?q=<query>

Category Quick-Links (below hero):
- Horizontal scrollable row of category pills/icons:
  Restaurants | Cafés | Bars | Pubs | Beach Clubs | Nightlife | Events | Offers
- Each links to /explore?category=<type>

Homepage Sections (each has title + "See all" link + horizontal scroll card row):
1. Trending Now — venues ordered by (reservation_count * 2 + view_count) DESC, last 7 days
2. Editor's Picks — venues where editors_pick = true
3. Hidden Gems — venues where avg_rating >= 4 AND review_count < 30
4. New Openings — venues created within last 60 days, ordered by created_at DESC
5. Rooftops — venues with tag = 'rooftop'
6. Best Breakfast — venues where serves_breakfast = true, ordered by avg_rating
7. Summer Spots — venues with is_summer_spot = true

Venue Card component (reusable, used across all sections):
- Cover image (aspect ratio 4:3)
- Venue name (bold)
- Category badge
- Area name
- Star rating + review count
- Price range indicator ($, $$, $$$, $$$$)
- Save button (heart icon, top-right corner of image)
- Card links to /venue/<slug>
```

---

## Prompt 4 — Search & Explore

```
Build the search and explore experience:

Search page at /search:
- URL state: ?q=<query>&category=<type>&area=<id>&price=<range>&rating=<min>&amenities=<ids>
- Search bar at top, pre-filled from URL param
- Results count: "23 places found for \"sushi\""
- Filter sidebar (desktop) / filter sheet/drawer (mobile):
  - Category: tab buttons (All, Restaurants, Cafés, Bars, Pubs, Beach Clubs, Nightlife)
  - Area: dropdown/select populated from areas table
  - Price Range: checkbox group (Budget $, Moderate $$, Expensive $$$, Luxury $$$$)
  - Min Rating: star toggle (3+, 4+, 4.5+)
  - Amenities: checkbox list (Outdoor Seating, Sea View, Valet Parking, Rooftop, Live Music, Pet Friendly, etc.)
  - Open Now: toggle switch
- Sort: dropdown (Relevance, Highest Rated, Most Reviewed, Newest)
- Results grid: 3-col desktop, 2-col tablet, 1-col mobile, using VenueCard component
- Infinite scroll or Load More button
- All filters update URL params and re-query Supabase
- Full-text search via tsvector: search_vector @@ plainto_tsquery('english', query)
- Empty state with suggestions if no results

Explore page at /explore:
- Same as search but shows a category-specific hero and category-filtered results
- Category determined by URL param: /explore?category=restaurant
```

---

## Prompt 5 — Venue Profile Page

```
Build the venue profile page at /venue/[slug]:

Fetch venue by slug joining: area, venue_images, amenities, cuisines, tags, 
operating_hours, active menu with sections and items, upcoming events, active offers.

Page Layout:

1. Photo Gallery
   - Large cover image (full width, max 600px height)
   - Thumbnail row below showing all venue_images (clickable to open lightbox)
   
2. Venue Header
   - Name (h1, large)
   - Category badges (restaurant, cafe, etc.)
   - Cuisine types
   - Area → links to /explore?area=<id>
   - Price range ($ symbols)
   - Star rating (filled stars) + "(128 reviews)" link scrolls to reviews section
   - Open/Closed status badge (calculated from operating_hours + current time)
   
3. Action Bar (sticky on mobile)
   - Reserve button (primary, links to #reservation or opens reservation modal)
   - WhatsApp button (opens wa.me/<number>)
   - Maps button (opens google_maps_url)
   - Save button (heart toggle, calls saved_venues insert/delete)
   - Share button
   
4. About Section
   - Description paragraph
   - Story paragraph (if exists)
   
5. Facilities / Amenities
   - Icon grid of all venue amenities
   
6. Operating Hours
   - Table: day | hours | open/closed
   - Highlight today's row
   
7. Menu Section
   - If menu_pdf_url: show PDF download button
   - Otherwise: accordion by menu_section, items listed with name/description/price/dietary icons
   
8. Events Section (if upcoming events exist)
   - Horizontal scroll of event cards: cover, title, date, price/free badge
   
9. Offers Section (if active offers exist)
   - Card list of active offers with type badge
   
10. Reservation Section (id="reservation")
    - Date picker, time slot selector, party size selector
    - Slots fetched from check-availability edge function
    - "Reserve Now" button opens reservation form modal
    
11. Reviews Section (id="reviews")
    - Rating breakdown: overall + food/service/ambiance/value bars
    - Review cards: avatar, name, date, stars, title, body, photos, venue reply
    - Pagination (10 per page)
    - "Write a Review" button (auth required)
    
12. Similar Venues
    - 4-6 venue cards from same area + category
```

---

## Prompt 6 — Reservation System

```
Build the full reservation flow:

Reservation Modal/Page:
- Triggered from venue profile "Reserve" button
- Step 1 — Select: date picker, party size selector (1-20), time slot grid
  - Time slots fetched from edge function check-availability
  - Unavailable slots shown greyed out
- Step 2 — Details form:
  - Contact name, phone (+961 prefix), email
  - Occasion selector: Birthday, Anniversary, Business, Casual, Other
  - Indoor/Outdoor preference
  - Special requests textarea
- Step 3 — Confirmation page:
  - Show booking summary
  - Show confirmation code (auto-generated, 8-char alphanumeric)
  - "Add to Calendar" button
  - Links back to venue profile

My Reservations page at /reservations:
- List of user's reservations ordered by date desc
- Each shows: venue name+image, date, time, party size, status badge
- Status colors: pending=yellow, confirmed=green, cancelled=gray, completed=blue, no_show=red
- Cancel button on confirmed reservations (if > 2 hours away)
- Cancelled reservation shows cancellation reason

Confirmation code generation (Edge Function or Postgres):
- 8-char uppercase alphanumeric
- Unique constraint on reservations table
```

---

## Prompt 7 — Reviews

```
Build the review system:

Write Review Modal (triggered from venue profile):
- Auth-gated: if not logged in, redirect to login
- Check if user already reviewed this venue (show edit form if yes)
- Rating inputs: 5-star selectors for Food, Service, Ambiance, Value
  (overall auto-calculated as average)
- Title input (optional)
- Body textarea (required, min 50 chars)
- Visit date picker
- Photo upload (up to 5 images, stored in review-media bucket)
- Submit button → creates review with status='pending'
- Success state: "Your review has been submitted and is pending approval"

Review Card component:
- User avatar + username + date
- Star rating display
- Sub-ratings (food/service/ambiance/value) as small bars
- Review title (bold) + body
- Review photos (thumbnail grid, click to expand)
- Venue reply (if exists, shown in a highlighted box "Owner replied:")
- Flag button (for logged-in users)

My Reviews page at /reviews (in user account):
- List of user's reviews with venue name, date, status badge
- Edit button for pending reviews
```

---

## Prompt 8 — Restaurant Dashboard

```
Build the Restaurant Dashboard at /dashboard (requires venue_owner or venue_staff role):

Sidebar navigation:
- Overview
- Reservations
- Menu
- Gallery
- Events
- Offers
- Reviews
- Staff (venue_owner only)
- Settings (venue_owner only)

Overview page:
- Metric cards: Views (today/week/month), Reservations (this week), Avg Rating, Pending Reviews
- Simple line chart: views per day over last 30 days
- Quick actions: Add Event, Create Offer, View Pending Reservations

Reservations page:
- Date picker to select date
- Reservation list for selected date with columns: time, name, party size, occasion, status, actions
- Actions: Confirm (pending), Mark Complete, Mark No-Show
- Status filter tabs: All, Pending, Confirmed, Completed

Menu page:
- Section list with drag-to-reorder
- Add Section button
- Each section expandable: shows items list
- Add Item button per section
- Item form: name, description, price, dietary flags (vegetarian/vegan/halal/gluten-free/spicy), image upload
- Inline edit for existing items

Gallery page:
- Image grid with drag-to-reorder
- Upload new images (Supabase Storage: venue-images bucket)
- Set as Cover button per image
- Delete image button

Events page:
- List of venue events with status badges
- Create Event button → form: title, type, date/time, free/paid, capacity, description, cover image
- Edit and Delete actions

Offers page:
- List of offers with active/inactive toggle
- Create Offer button → form: title, type (discount/fixed/free/package), value, validity, days, promo code
- Edit and Delete

Reviews page:
- List of approved reviews for this venue
- Each shows: user, date, rating, body
- "Reply" button → textarea + submit (max one reply per review)
- Read-only once replied

Staff page:
- List of current staff with role badges
- "Invite Staff" button → enter email + select role (manager/staff/host)
- Remove staff button

Settings page:
- Venue info form: name, description, story, phone, whatsapp, email, website, socials
- Address + map picker
- Operating hours (per day: open/closed toggle, open time, close time)
- Reservation settings: accepts_reservations toggle, lead_time, window_days, cancellation_hours, auto_confirm toggle
- Meal service: serves_breakfast/brunch/lunch/dinner checkboxes
- Save Changes button
```

---

## Prompt 9 — Admin CMS

```
Build the Admin CMS at /admin (requires admin or super_admin role):

Sidebar navigation:
- Dashboard
- Venue Approval
- Users
- Reviews
- Collections
- Featured
- Magazine
- Ads
- Analytics

Admin Dashboard:
- Platform KPI cards: Total Active Venues, Total Users, Reservations This Month, Pending Approvals, Pending Reviews

Venue Approval page:
- Table of pending venues: name, owner email, area, type, submitted date, actions
- Click venue name to expand full venue details preview
- Approve button → sets status=active, sends email notification
- Reject button → opens modal to enter rejection reason, sets venue back to owner for editing

Users page:
- Searchable table: username, email, role, join date, is_banned
- Actions: View Profile, Change Role (dropdown), Ban/Unban
- Filters: by role, by banned status

Review Moderation page:
- Table of pending reviews: user, venue, rating, excerpt, submitted date
- Click to expand full review text
- Approve → status=approved (triggers avg_rating update via trigger)
- Reject → status=rejected

Collections page:
- List of all collections with is_active toggle
- Create Collection button → form: title, slug, description, cover image
- Edit collection → manage venues list with position drag-and-drop
  - Search and add venue, set position order
  - Remove venue from collection

Featured Placements page:
- List of active placements: venue name, type, zone, start, end
- Create Placement button → form: select venue, placement_type, placement_zone, start/end dates

Magazine page:
- List of articles with status badges (draft/published/archived)
- Create Article button → rich text editor (cover image, title, category, tags, content, SEO fields)
- Publish / Unpublish / Archive actions
- "Mentioned Venues" multi-select on article form

Ads page:
- List of ads with active toggle, impression/click counts
- Create Ad: title, image, destination URL, placement type, date range, target category
```

---

## Prompt 10 — Collections & Magazine Pages

```
Build the public-facing Collections and Magazine sections:

Collections index at /collections:
- Grid of collection cards: cover image, title, description, venue count
- Each links to /collections/<slug>

Collection detail at /collections/[slug]:
- Hero: cover image + title + description
- If sponsored: sponsor logo + "Sponsored by" label
- Venue grid: numbered position, VenueCard component

Magazine index at /magazine:
- Featured article hero (is_featured=true): full-width image, title, excerpt, author, read time
- Category filter tabs: All, Guide, Review, Feature, List, News
- Article card grid: cover image, category badge, title, excerpt, author avatar, date, read time
- Pagination

Article page at /magazine/[slug]:
- Hero image (full width)
- Category + read time header
- Title (h1)
- Author row: avatar, name, published date
- Rich HTML content rendered safely
- "Mentioned Venues" section at bottom: horizontal row of VenueCards
- Related articles (same category)
- Social share buttons
```

---

## Prompt 11 — User Account

```
Build the user account section:

Profile page at /profile:
- Avatar upload (Supabase Storage: user-avatars bucket)
- Edit form: full_name, username (unique check), bio, phone, language (EN/AR)
- Change email / Change password links (use Supabase updateUser)
- Preferred area select

Saved Venues at /saved:
- Grid of saved VenueCards
- Unsave button on each card
- Empty state: "No saved places yet. Start exploring!"

My Reservations at /reservations:
- Tabs: Upcoming | Past | Cancelled
- Each reservation: venue image+name, date, time, party_size, status badge, confirmation code
- Cancel button (upcoming confirmed reservations with > 2hr remaining)

My Reviews at /reviews:
- List: venue name+cover, star rating, review excerpt, status badge, visit date
- Edit button for pending reviews

Notifications at /notifications:
- List: icon by type, title, body, timestamp, unread dot
- Mark all as read button
- Click notification → navigate to relevant page (reservation, review, venue)
```

---

## Prompt 12 — Navigation & Layout

```
Build the app shell and navigation:

Main navbar (all pages):
- Left: TableLB logo
- Center: navigation links (Home, Explore, Restaurants, Cafes, Bars, Beach Clubs, Events, Offers, Collections, Magazine)
- Right: Search icon, notification bell (badge count), user avatar/menu, Login button (if guest)
- On mobile: hamburger menu with slide-in drawer
- Navbar transparent on homepage hero, solid white on scroll and all other pages

User dropdown menu (when logged in):
- My Profile
- Saved Places  
- My Reservations
- My Reviews
- Notifications
- Dashboard (if venue_owner or venue_staff)
- Admin Panel (if admin)
- Sign Out

Footer:
- Logo + tagline
- Links: About, Contact, Magazine, Collections, Add Your Venue, Privacy, Terms
- Social icons: Instagram, Facebook, TikTok
- © 2024 TableLB. All rights reserved.

Category pages (navbar links):
- /restaurants, /cafes, /bars, /pubs, /beach-clubs, /nightlife
- Each is just /explore?category=<type> with a custom hero headline

Breadcrumbs on venue profile and collection/article pages.
```
