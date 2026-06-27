# Authentication & Roles

## Auth Providers (Supabase Auth)

| Provider | Use Case |
|----------|----------|
| Email + Password | Primary signup |
| Google OAuth | Social login |
| Apple OAuth | Mobile app (Phase 4) |

---

## Sign Up Flow

1. User signs up via email or Google OAuth
2. Supabase creates `auth.users` record
3. DB trigger `on_auth_user_created` creates `profiles` row with `role = 'user'`
4. (Email) Confirmation email sent by Supabase Auth
5. After confirmation, user lands on homepage (or completes profile)

```sql
CREATE OR REPLACE FUNCTION handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO public.profiles (id, full_name, avatar_url)
  VALUES (
    NEW.id,
    NEW.raw_user_meta_data->>'full_name',
    NEW.raw_user_meta_data->>'avatar_url'
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION handle_new_user();
```

---

## Venue Owner Registration Flow

1. User has a `user` account
2. Clicks "Add Your Venue" → fills venue form
3. Venue created with `status = 'pending'`, `owner_id = auth.uid()`
4. Admin reviews and approves venue
5. On approval, user's role updated to `venue_owner`
6. Owner gets email: "Your venue is live!"
7. Owner gains access to `/dashboard`

---

## Venue Staff Invitation Flow

1. Venue owner goes to Dashboard → Staff
2. Enters staff member's email + role (manager/staff/host)
3. System creates `venue_staff` record + sends invitation email via Resend
4. Invitee registers (or logs in if existing user)
5. Their profile role is checked; if they manage any venue, they get `venue_staff` access scoped to that venue

---

## Role Definitions

### `user`
- Browse all active venues
- Save venues to favourites
- Create reservations
- Write reviews (one per venue)
- RSVP to events
- Access account pages: `/profile`, `/saved`, `/reservations`, `/reviews`, `/notifications`

### `venue_staff`
- All `user` permissions
- Access `/dashboard` scoped to their venue(s)
- View and manage reservations for their venue
- Cannot edit venue info or publish events/offers (manager role can)

### `venue_owner`
- All `user` permissions
- Full access to `/dashboard` for their venues
- CRUD: venue info, menu, gallery, events, offers
- Invite and manage staff
- View analytics
- Access subscription / billing

### `editor`
- All `user` permissions
- Access to `/admin/magazine` (create/publish articles)
- Access to `/admin/collections` (manage collections)
- Access to `/admin/events` (platform-wide event management)
- Cannot approve venues or manage users

### `admin`
- All `editor` permissions
- Full `/admin` panel access
- Approve/reject venues
- Moderate reviews
- Manage users (ban, change role)
- Manage featured placements and ads
- View platform analytics
- Cannot delete super_admin accounts

### `super_admin`
- Unrestricted access
- Can promote/demote admins
- Access to billing and platform config

---

## Permissions Matrix

| Action | guest | user | venue_staff | venue_owner | editor | admin |
|--------|:-----:|:----:|:-----------:|:-----------:|:------:|:-----:|
| Browse venues | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| View venue profiles | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Save venues | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Create reservation | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Write review | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ |
| RSVP to event | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Submit venue | ✗ | ✓ | ✗ | ✓ | ✗ | ✓ |
| Manage reservations | ✗ | ✗ | ✓ (own venue) | ✓ | ✗ | ✓ |
| Edit venue info | ✗ | ✗ | ✗ | ✓ | ✗ | ✓ |
| Manage menu/gallery | ✗ | ✗ | ✗ | ✓ | ✗ | ✓ |
| Publish events | ✗ | ✗ | ✗ | ✓ | ✓ | ✓ |
| Publish offers | ✗ | ✗ | ✗ | ✓ | ✗ | ✓ |
| View venue analytics | ✗ | ✗ | ✓ (own venue) | ✓ | ✗ | ✓ |
| Manage collections | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ |
| Publish articles | ✗ | ✗ | ✗ | ✗ | ✓ | ✓ |
| Approve venues | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |
| Moderate reviews | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |
| Manage users | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |
| Manage ads | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |
| Platform analytics | ✗ | ✗ | ✗ | ✗ | ✗ | ✓ |

---

## Frontend Route Guards

```typescript
// Route protection pattern for Lovable
const PROTECTED_ROUTES = {
  '/dashboard': ['venue_owner', 'venue_staff', 'admin', 'super_admin'],
  '/admin': ['admin', 'super_admin'],
  '/admin/magazine': ['editor', 'admin', 'super_admin'],
  '/admin/collections': ['editor', 'admin', 'super_admin'],
  '/profile': ['user', 'venue_staff', 'venue_owner', 'editor', 'admin', 'super_admin'],
  '/saved': ['user', 'venue_staff', 'venue_owner', 'editor', 'admin', 'super_admin'],
  '/reservations': ['user', 'venue_staff', 'venue_owner', 'editor', 'admin', 'super_admin'],
};
```

## JWT Custom Claims (Optional Enhancement)

For performance, embed role in JWT so frontend doesn't need to query `profiles` on every render:

```sql
-- Supabase Auth Hook: add role to JWT
CREATE OR REPLACE FUNCTION add_role_to_jwt(event jsonb)
RETURNS jsonb AS $$
DECLARE
  user_role text;
BEGIN
  SELECT role INTO user_role FROM profiles WHERE id = (event->>'user_id')::uuid;
  RETURN jsonb_set(event, '{claims,role}', to_jsonb(user_role));
END;
$$ LANGUAGE plpgsql;
```
(Set this as a custom auth hook in Supabase dashboard → Auth → Hooks)
