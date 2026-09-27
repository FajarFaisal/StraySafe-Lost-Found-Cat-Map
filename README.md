# 🐾 StraySafe — Lost & Found Cat Map

Built by **Fajar Faisal** for **#hackthekitty 2026 — World Cat Domination Day**.

A community map where people report lost, found, or sighted cats with a photo and a pin. Nearby users filter by status and radius, get email alerts for new reports near them, and an on-device AI suggests likely lost↔found matches by visual similarity.

## Tech stack

| Layer | Choice | Why |
|---|---|---|
| Frontend | React 19 + Vite + Tailwind v4 | Fast dev loop, hot reload, tiny prod bundle |
| Map | Leaflet + OpenStreetMap tiles | Free, no API key/billing setup needed mid-hackathon |
| Backend | Supabase (Postgres + PostGIS + Storage + Auth) | Real database + photo storage + geo queries, live in ~10 min |
| AI/ML | TensorFlow.js + MobileNet (runs in the browser) | Zero server cost, zero inference latency, works offline once cached |
| Analytics | Google Analytics 4 + a lightweight in-app dashboard (Supabase-backed) | GA4 for external reporting, in-app dashboard so judges see live usage stats without leaving the app |

The app **works with zero configuration** — without a `.env.local` it runs entirely on mock data so you always have something to demo, then gets real data the moment Supabase env vars are added.

## Quick start

```bash
npm install
npm run dev          # demo mode, no setup needed
```

### Connect the real backend (Supabase)

1. Create a free project at [supabase.com](https://supabase.com).
2. Open **SQL Editor → New query**, paste the contents of `supabase/schema.sql`, run it. This creates the `posts`, `alert_subscriptions`, and `analytics_events` tables, PostGIS-powered nearby search, pgvector-powered AI matching, and the `cat-photos` storage bucket — including its public-read policy.
3. Copy `.env.example` to `.env.local` and fill in `VITE_SUPABASE_URL` and `VITE_SUPABASE_ANON_KEY` from **Project Settings → API**.
4. Restart `npm run dev`. The "demo mode" badge in the header disappears once it's reading from real data.

### Connect Google Analytics 4 (optional)

1. Create a GA4 property → Admin → Data Streams → Web → copy the **Measurement ID** (`G-XXXXXXX`).
2. Add it to `.env.local` as `VITE_GA_MEASUREMENT_ID`.
3. Events tracked: `post_created`, `filter_used`, `alert_signup`, `match_viewed`, `map_pin_clicked`, `cat_detection_rejected`. These also log to the in-app **Activity** dashboard (top right) regardless of whether GA4 is configured, so you always have something to show judges live.

### Email alerts (now built)

New reports automatically email anyone subscribed nearby, via two Supabase Edge Functions + Resend (free tier, no domain verification needed for a hackathon demo):

- `supabase/functions/notify-nearby` -- runs on every new `posts` row, finds matching subscribers (`nearby_subscribers()` in `schema.sql`), sends each one an email
- `supabase/functions/unsubscribe` -- one-click unsubscribe link target included in every email

**Setup (~10 minutes):**

1. Install the Supabase CLI if you don't have it: `npm install -g supabase`
2. Log in and link the project: `supabase login` then `supabase link --project-ref <your-project-ref>` (find the ref in your project URL or Settings -> General)
3. Get a free [Resend](https://resend.com) API key (no credit card, no domain setup needed -- the function sends from Resend's shared `onboarding@resend.dev` address, which works for a hackathon demo without DNS configuration)
4. Set the function secrets:
   ```bash
   supabase secrets set RESEND_API_KEY=re_xxxxxxxx
   supabase secrets set WEBHOOK_SECRET=$(openssl rand -hex 16)
   supabase secrets set SITE_URL=https://your-deployed-app.vercel.app
   ```
   (Save the `WEBHOOK_SECRET` value somewhere -- you'll paste it into the Dashboard in step 6.)
5. Deploy both functions:
   ```bash
   supabase functions deploy notify-nearby --no-verify-jwt
   supabase functions deploy unsubscribe --no-verify-jwt
   ```
6. In the Supabase Dashboard -> **Database -> Webhooks -> Create a new webhook**:
   - Table: `posts`
   - Events: `Insert`
   - Type: `HTTP Request` -> your deployed `notify-nearby` function URL (shown after deploy, looks like `https://<ref>.supabase.co/functions/v1/notify-nearby`)
   - HTTP Headers: add `x-webhook-secret` = the same value you set in step 4 (this is what stops randoms from calling your function directly)

That's it -- every new report now fires the webhook, which emails everyone nearby who signed up, with a one-click unsubscribe link baked in.

## How the AI matching works

- **Cat-detection gate** (`src/lib/catAI.js`): on photo upload, MobileNet classifies the image client-side; if its top guesses don't include a cat-like ImageNet class, the user gets a gentle warning (not a hard block -- false negatives shouldn't lock anyone out during a real emergency).
- **Color tagging** (`src/lib/colorUtils.js`): a fast canvas pixel-bucket heuristic suggests tags like "orange" / "black-and-white", editable by the user. No ML needed, loads instantly.
- **Visual matching**: MobileNet's penultimate layer gives a 1280-dim embedding per photo, stored in Postgres via `pgvector`. The `find_matches` SQL function (in `schema.sql`) does a cosine-distance nearest-neighbor search across *opposite-status* posts (lost vs found/sighted) and surfaces them as "possible matches" with a similarity %, drawn on the map as an animated dashed "scent trail" between the two pins.
- This is genuinely free to run at hackathon scale -- no GPU server, no per-call API cost, runs in the visitor's own browser.

## Project structure

```
src/
  components/
    CatMap.jsx              map, paw markers, scent-trail lines
    FilterBar.jsx            status chips + radius slider
    ReportModal.jsx           3-step report flow (photo -> details -> location)
    PostDetail.jsx              full report + AI match list
    AlertModal.jsx                email alert signup
    AnalyticsDashboard.jsx          in-app usage stats
  lib/
    api.js                  data layer (Supabase calls + mock-data fallback)
    catAI.js                 TensorFlow.js cat-detection + embeddings (lazy-loaded)
    colorUtils.js              lightweight color-tag heuristic
    analytics.js                 GA4 + Supabase event tracking
    statusConfig.js                shared status colors/labels
    supabaseClient.js                Supabase singleton
supabase/
  schema.sql                full DB schema, run once in Supabase SQL Editor
  functions/
    notify-nearby/            Edge Function: emails nearby subscribers on new reports
    unsubscribe/                Edge Function: one-click unsubscribe link target
```

## Design notes

Palette and type are deliberately tied to the subject rather than a generic template: a tabby-stripe caramel base, with status colors chosen for what they signal (rust for "lost" = urgent, sage for "found" = safe, butter-yellow for "sighted" = alert). Fraunces (display serif) + Space Grotesk (UI) + JetBrains Mono (coordinates/timestamps, like text engraved on a collar tag). The map itself is the hero -- there's no marketing landing page, you land straight on the live map, which is the right call for a utility tool people open mid-emergency. The signature visual element is the animated dashed "scent trail" line connecting AI-matched lost/found pins.

Mobile-first throughout: bottom sheets instead of modals on small screens, a single floating action button, native range-slider accent color, no hover-only interactions, and the AI model is lazy-loaded so it doesn't tax mobile data/CPU until someone actually uploads a photo.

## 16-day roadmap

**Days 1-2 -- Foundation (mostly done for you in this scaffold)**
Confirm Supabase project + schema is live, get one real device-tested report flow working end to end (photo upload -> DB row -> shows on map).

**Days 3-5 -- Core map experience**
Real geolocation permission flow, marker clustering if report density gets high, "use my location" + manual pin-drop UX polish, loading/empty/error states for the map.

**Days 6-8 -- AI matching**
Wire `ai_embedding` writes on every post (already implemented in `ReportModal.jsx`), test `find_matches` with real photos of the same cat from different angles, tune the cosine-similarity threshold, add a "this isn't a match" feedback button to improve trust.

**Days 9-10 -- Alerts (now built, see "Email alerts" above)**
Test the deployed `notify-nearby` and `unsubscribe` functions end to end with a real Resend key; consider adding SMS via Twilio as a stretch goal if email open rates seem low during testing.

**Days 11-12 -- Analytics**
Finish GA4 event coverage, add a "time to resolution" stat (lost -> resolved) to the in-app dashboard since that's the metric judges will care about most for a "positive impact" theme.

**Days 13-14 -- Polish & accessibility**
Cross-device testing (iOS Safari, Android Chrome, iPad, desktop), keyboard navigation through the report flow, screen-reader labels on map markers, contrast check on status colors.

**Day 15 -- Demo prep**
Seed the database with a realistic local dataset (your actual neighborhood), write the 2-minute demo script, record a backup video in case of live-demo wifi issues.

**Day 16 -- Buffer + submission**
Deploy to Vercel/Netlify, write the submission writeup, rest.

## Demo screenshots
**Home page**
<img width="1426" height="840" alt="image" src="https://github.com/user-attachments/assets/f5111ae7-41bf-4412-84f4-18c121916bc0" />

**Reporting cat**
<img width="2244" height="1138" alt="image" src="https://github.com/user-attachments/assets/e06ab070-0c02-44cd-917e-c09581001557" />

<img width="798" height="854" alt="image" src="https://github.com/user-attachments/assets/f7c5a4d0-5252-48d2-9328-a94a39983521" />

<img width="2212" height="918" alt="image" src="https://github.com/user-attachments/assets/fd5a9a6a-62db-4e07-948f-ff6c88446f51" />

<img width="1944" height="788" alt="image" src="https://github.com/user-attachments/assets/4f88c71f-43f1-42c6-a541-d6300724ed3e" />

<img width="1256" height="786" alt="image" src="https://github.com/user-attachments/assets/79a02d96-aaf1-433c-bdcd-2e93a76f9d72" />


**Cat is marked found**
<img width="1204" height="1032" alt="image" src="https://github.com/user-attachments/assets/de05121b-269e-402e-b3ab-0fb0dc657410" />

<img width="1242" height="1040" alt="image" src="https://github.com/user-attachments/assets/857f2c10-b683-4377-87c5-1df6e037cfaa" />


**Live Analytics Dashboard**
<img width="1096" height="906" alt="image" src="https://github.com/user-attachments/assets/2656ff69-a070-4ad7-8402-28850cfa8408" />

## Deploying

Works on Vercel or Netlify with zero config -- connect the GitHub repo, set the same env vars from `.env.local` in the host's dashboard, done. Both have generous free tiers, fine for a hackathon project.
