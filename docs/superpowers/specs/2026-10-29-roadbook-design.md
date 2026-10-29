# Roadbook — Design

*Date: 2026-10-29. Status: design for a proof of concept (POC); not yet scheduled to build.*

## 1. Intent

**What we heard**
- A small group of Porsche owners wants to do group drives.
- Two problems with Apple Maps / Google Maps today:
  - **A. Building the route** — hard to force the apps onto the specific (twisty) roads you want instead of the fastest path.
  - **B. Getting the route to everyone** — there's no good way to share a curated route so every driver ends up on the same roads.
- On drive day, each driver navigates with **the app they already use** (Apple Maps or Google Maps, including CarPlay / Android Auto). The group communicates by **RF radios**, so live tracking and in-car coordination are out of scope.
- **Anyone in the group can create a route.** Only its creator can edit it. Others may propose competing routes the same way.
- Sharing is **a link pasted into the group text**.
- Planning must work **equally well on phone and desktop**.
- Start with the simplest POC that can later grow into sign-in (creators have accounts) or a club space (shared library of drives).

**Assumptions** (correct these if wrong)
- The group is small (roughly 5–30 people) and US-based.
- Members use a mix of iPhone and Android; we don't know which nav app is dominant.
- View links are "anyone with the link can view" — no privacy beyond an unguessable URL.
- Waze is not supported (its links carry a single destination and no waypoints).

**Success criteria for the POC**
- The group drives a planned route with everyone on the same roads.
- Nobody needs help opening their leg links.

## 2. Scope

**In the POC**
- Create and edit a route on phone or desktop: name, description, optional date/time, typed stops with notes, and shaping points.
- Route stats: total and per-leg miles and minutes.
- A shareable drive page with one-tap Apple Maps / Google Maps links per leg.
- A rich link preview card in group texts.
- Edit access via a secret edit link (no accounts).

**Not in the POC**
- Accounts / sign-in, club groups (designed for, built later — see §9).
- Comments, reactions, RSVPs, voting on proposals (the group text covers discussion).
- Live location, turn-by-turn navigation inside the app, offline mode.
- Waze, GPX export, edit history, simultaneous editing.

## 3. Alternatives considered

| Option | Why not (for now) |
|---|---|
| **Everyone uses MapQuest**, which supports multi-stop routes (reportedly up to 26) and route sharing | Requires everyone to adopt an unfamiliar app; Android Auto support is new and experimental (Sept 2026). Hands-on trial of route building in the MapQuest mobile app was not promising — the same pain as problem A. |
| **Plan on Apple's routing (MapKit JS)** | Would make previews exact for Apple Maps users, but requires a $99/yr Apple Developer account, and the group's dominant nav app is unknown. |
| **Plan on Mapbox / MapQuest / other routing** | Previews wouldn't match *either* nav app the group drives with. |
| **No server: whole route encoded in the URL** | Every edit produces a new link, so the group text fills with stale versions; no place to add accounts later. |
| **Hosted backend called from the browser (Firebase / Supabase)** | Enforcing the edit key safely still needs server functions; little saved over a small server. |

**Chosen:** a small mobile-first web app with a database, planning on **Google Maps Platform** so previews match Google Maps exactly and Apple Maps closely.

## 4. Screens

All screens are designed for phones first and widen for desktop.

### 4.1 Home — `/`
- **"Plan a new drive"** button.
- **"My drives"**: drives created on this device, found via edit keys stored in the browser.

### 4.2 Planner — `/r/{id}/edit`
- Full-screen Google map. Controls in a swipe-up bottom sheet on phones; a side panel on desktop.
- **Add a stop:** search by name (Places Autocomplete) or long-press the map; choose a type — Meet, Regroup, Fuel, Food, Photo, Finish — and add an optional note.
- **Shape the road:**
  - Phone: tap the route line to drop a shaping point, then drag the point onto the desired road.
  - Desktop: drag the route line directly.
  - Tap a shaping point to delete it. Shaping points have large touch targets.
- Name, description, optional date/time.
- Live stats: total and per-leg miles and minutes.
- Waypoint-limit warning per leg (see §6.2).
- **Autosave.** **Share** opens the native share sheet (Web Share API) with the view link; on desktop it copies the link.
- Right after creating a drive, prompt: **"Save your edit link"**, with a share button so the creator can text or email it to themselves.

### 4.3 Drive page — `/r/{id}`
- Header: name, date/time, total miles and time, description.
- Map of the full route.
- Leg list, for example: **"Leg 2 · Regroup → Alice's Coffee · 34 mi · 52 min"** with **[Apple Maps] [Google Maps]** buttons. The first tap remembers the viewer's choice (in the browser); after that each leg shows one large button for that app, with a small link to switch.
- Stop notes shown inline.
- A link preview card (Open Graph tags plus a Static Maps image of the route) so the pasted link looks right in iMessage, WhatsApp and SMS.
- An **Edit** button if this device holds the edit key.
- A small note for Apple Maps users on iOS older than 18.4 (see §6.4).

## 5. Data

One record per drive, saved as a whole.

**Drive**

| Field | Type | Notes |
|---|---|---|
| `id` | text, 8 random chars | Used in the view link. Unguessable; anyone holding the link can view. |
| `edit_key_hash` | text | Hash of a random 128-bit edit key. The plain key only exists in the edit link and in the creator's browser storage. |
| `name` | text | Required. |
| `description` | text | Optional. |
| `drive_at` | timestamp | Optional. |
| `points` | JSON array | Ordered points (below). |
| `computed` | JSON, nullable | Cached route geometry (encoded polyline) and per-leg distance/duration from the last successful routing call. |
| `computed_stale` | boolean | True when the latest points were saved but routing failed. |
| `created_at`, `updated_at` | timestamp | |

**Point**

| Field | Applies to | Notes |
|---|---|---|
| `id` | all | Stable id for UI edits. |
| `kind` | all | `stop` or `shape`. |
| `lat`, `lng` | all | |
| `stop_type` | stop | `meet`, `regroup`, `fuel`, `food`, `photo`, `finish`. |
| `label` | stop | E.g. "Alice's Coffee". |
| `note` | stop | Optional. |
| `place_id` | stop | Optional Google place ID, so nav apps show the real business name. |

**Rules**
- The first and last points must be stops.
- **Legs are derived, not stored:** cut the point list at every stop. A leg is: previous stop → its shaping points → next stop.
- Last save wins. There is a single editor per drive.

**Edit access** is one function, `canEdit(drive, request)`, which currently checks the request's edit key against `edit_key_hash`. All write endpoints call it.

**Edit link format:** `/r/{id}/edit#k={key}`. The key sits in the URL fragment, which browsers never send to servers, so it stays out of server logs and `Referer` headers. The page reads it, stores it in browser storage, and sends it in a request header on each save.

## 6. Building the nav links

### 6.1 One leg, one link
- The destination is the leg's ending stop; its shaping points become waypoints, in order.
- **No origin in the link.** The nav app starts from the driver's current location (they're at the previous stop), which lets navigation start immediately.
- **Leg 0, "Get to the meet":** a link to the meet point with no waypoints.

**Google Maps**
```
https://www.google.com/maps/dir/?api=1
  &destination={lat},{lng}[&destination_place_id={id}]
  &waypoints={lat},{lng}|{lat},{lng}|...
  &travelmode=driving
  &dir_action=navigate
```

**Apple Maps** (iOS 18.4+ unified Maps URLs)
```
https://maps.apple.com/directions
  ?destination={lat},{lng}
  &waypoint={lat},{lng}&waypoint={lat},{lng}...
  &mode=driving
```

Links are built **only from `points`**, never from `computed`, so they work even if routing is down.

### 6.2 Waypoint limits
- Each app has its own limit in config: `GOOGLE_MAX_WAYPOINTS = 3` (Google's documented mobile-browser limit), `APPLE_MAX_WAYPOINTS = 9` (no documented limit; conservative start). Adjust both after the field test (§8.2).
- **While planning:** if a leg's shaping points exceed either limit, warn the creator — *"This leg needs 5 shaping points; Google links carry 3. Add a stop (maybe a photo spot), or it will be split."*
- **If the creator leaves it:** the drive page splits that leg for the affected app into sub-legs (**2a / 2b**), breaking at a shaping point. Sub-leg 2a's destination is that shaping point.

### 6.3 Keeping drivers on the planned roads
- The planner draws the route with Google's Routes API using the **same waypoints** as the links, so the preview matches what Google Maps users get.
- Apple Maps users get a close match, not a guaranteed one, since Apple's routing can choose differently between points.
- Guidance shown in the planner: place shaping points well into the road you want, not right at a junction.
- Shaping points act as stops in the nav apps (a driver may hear "arriving at waypoint"). Placing them mid-road keeps that low-key. Behavior is verified in the field test.

### 6.4 Known limit
Apple Maps waypoints require iOS 18.4+. Older iOS ignores them and routes directly to the destination. The drive page shows: *"Update iOS, or use the Google Maps button."*

## 7. Tech and hosting

- **App:** Next.js (React, TypeScript), one project for pages and server routes. Maps via `@vis.gl/react-google-maps`.
- **Database:** Postgres on Neon (free tier), with a lightweight typed query layer.
- **Hosting:** Vercel (free tier); deploys on push.
- **Home-screen install:** web app manifest for "Add to Home Screen". No offline mode.

**Google Maps Platform**

| API | Use |
|---|---|
| Maps JavaScript API | Maps on the planner and drive page |
| Places API (Autocomplete) | Searching for stops |
| Routes API | Route line and stats — called while editing (debounced) and on save; result cached in `computed` |
| Static Maps API | Link preview image |

**Key and cost safety**
- Browser key restricted by HTTP referrer to our domain; server key kept server-side and restricted to the APIs it needs.
- Billing budget alert plus per-API daily quota caps.
- A Google billing account (card on file) is required even though expected usage is inside the free allowances.

## 8. Failure cases and testing

### 8.1 Failure cases

| Situation | Behavior |
|---|---|
| Routes API fails while editing | Points save anyway; `computed_stale = true`; planner shows *"Route didn't update — tap to retry"* and keeps the last good line. The drive page still offers working links. |
| A point can't be reached by road | The affected leg is highlighted with a message pointing to the bad point. |
| Lost edit link | Not recoverable in the POC; mitigated by the "Save your edit link" prompt. Fixed properly by sign-in later. |
| Missing or wrong edit key on `/edit` | Show the read-only drive page. |
| Unknown or deleted drive id | Friendly "Drive not found" page. |
| Apple Maps on iOS < 18.4 | Note on the drive page (§6.4). |
| Creator edits from two devices at once | Last save wins. |

### 8.2 Testing
- **Unit tests** (pure functions): deriving legs from points, splitting legs by waypoint limit, building Google and Apple links, `canEdit`.
- **Server tests:** create a drive; save with the correct key, a wrong key and no key.
- **Field test** — plan a real ~2-hour drive and open every leg on:
  - iPhone + Apple Maps
  - iPhone + Google Maps
  - Android + Google Maps

  Ideally through CarPlay / Android Auto. Record:
  1. Did each app follow the planned roads?
  2. How many waypoints does the Google Maps *app* actually accept (vs. the documented 3 for mobile browsers)?
  3. What happens when passing a shaping point?
  4. Does the link preview render in iMessage, WhatsApp and SMS?
- **Prototype first:** shaping the route on a phone (§4.2) is the riskiest interaction and should be tried on real phones before the rest is built out.
- **Desktop nudge — decided by the prototype:**
  - If phone shaping feels good: no nudge; phone and desktop planning are equal.
  - If it feels clunky: phones still allow everything, but the planner shows a dismissible tip — *"Shaping roads is easier on a bigger screen"* — with a **Send edit link to myself** button (share sheet). The edit link opens on a laptop and picks up the key from the URL (§5), so moving between devices needs no account.
  - Adding and editing stops, notes and details stays fully supported on phones either way.

## 9. Growth path

- **Sign-in for creators (later):** add a `users` table and a nullable `owner_id` on drives. A signed-in user opening a valid edit link claims the drive. `canEdit` also accepts "signed-in owner". Existing links keep working.
- **Club groups (later):** add `groups` and membership; drives may belong to a group, giving the club a shared library of past and upcoming drives.

## 10. References
- [Google Maps URLs](https://developers.google.com/maps/documentation/urls/get-started) — up to 3 waypoints on mobile browsers, 9 elsewhere; 2,048-character URL limit.
- [Apple: Adopting unified Maps URLs](https://developer.apple.com/documentation/mapkit/unified-map-urls.md) — `/directions` with repeated `waypoint` parameters, iOS 18.4+.
- MapQuest Android Auto status: [9to5Google](https://9to5google.com/2026/09/02/mapquest-app-top-charts-android-auto/), [MobileSyrup](https://mobilesyrup.com/2026/09/04/mapquest-expands-to-android-auto-amid-rise-in-popularity/).
