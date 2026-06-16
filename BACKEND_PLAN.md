# Heat Record — Backend & Full-Product Plan

This document is a **plan only** — no backend code has been written yet. It turns
the current static dashboard (`index.html`, live on GitHub Pages) into the full
product you described:

- ML long-range forecasts (Prophet / LSTM)
- Subscriptions, alerts, notifications
- Admin panel, authentication, Postgres/Prisma
- Interactive district heat maps

Read it, then tell me what to build first.

---

## 1. Why a backend is needed (and why the current site can't do this)

The current site is a single static HTML file. The browser talks directly to
Open-Meteo. That's perfect for "show me live weather", but every feature above
needs something the browser **cannot** do on its own:

| Feature | Why it needs a server |
|---|---|
| ML forecasts | Models must be trained on years of data and run on a schedule, not in the visitor's browser |
| Alerts / notifications | Something must keep checking conditions and send email/push even when nobody's on the site |
| Accounts & subscriptions | Passwords, sessions, and billing must live on a trusted server, never client-side |
| Admin panel | Needs protected, authenticated server routes |
| Heat maps | Need pre-computed gridded data served as map tiles/GeoJSON |

So the architecture changes from *"static page → API"* to
*"web app → our API → database + ML jobs + notification service"*.

---

## 2. Proposed architecture

```
                ┌─────────────────────────────────────────────┐
                │  Browser (Next.js front-end, replaces        │
                │  today's single HTML file)                   │
                └───────────────┬─────────────────────────────┘
                                │ HTTPS / JSON
                ┌───────────────▼─────────────────────────────┐
                │  API layer (Next.js API routes / FastAPI)    │
                │  • auth  • subscriptions  • data endpoints   │
                └───┬───────────────┬───────────────┬──────────┘
                    │               │               │
          ┌─────────▼───┐   ┌───────▼──────┐  ┌─────▼─────────┐
          │ Postgres    │   │ Scheduled    │  │ Notification  │
          │ (Prisma)    │   │ jobs (cron)  │  │ service       │
          │ users,      │   │ • ingest     │  │ • email       │
          │ subs, areas,│   │ • train ML   │  │ • web push    │
          │ readings,   │   │ • check      │  │               │
          │ forecasts   │   │   alerts     │  │               │
          └─────────────┘   └──────┬───────┘  └───────────────┘
                                    │
                            ┌───────▼────────┐
                            │ ML service     │
                            │ Prophet / LSTM │
                            │ (Python)       │
                            └────────────────┘
```

### Recommended stack

| Layer | Recommendation | Why |
|---|---|---|
| Front-end | **Next.js (React) + TypeScript** | Same language end-to-end; great for the dashboard + admin |
| API | **Next.js API routes** (or **FastAPI** if you prefer Python near the ML) | Fewer moving parts |
| Database | **Postgres** + **Prisma ORM** | Exactly what you asked for; Prisma gives typed, migration-driven schema |
| Auth | **Auth.js (NextAuth)** or **Clerk** | Email/password + Google login, sessions, roles |
| ML | **Python**: Prophet first, LSTM (PyTorch/Keras) later | Prophet is fast to ship and strong for seasonal climate series |
| Jobs/cron | Host-native cron (Railway/Render) or **GitHub Actions** for ingestion | Keeps ingestion + retraining on a schedule |
| Notifications | **Resend/Postmark** (email) + **Web Push (VAPID)** | Free tiers exist for email; web push is free |
| Maps | **MapLibre GL** + tiles from a free provider; heat layer from our gridded data | Open-source, no per-map fee |

### Hosting & rough cost (monthly)

> The current static site stays **free** on GitHub Pages. These costs apply only
> to the new backend.

| Option | What you get | Approx cost |
|---|---|---|
| **Free-tier start** (Vercel hobby + Neon free Postgres + Resend free + GitHub Actions cron) | Enough to demo accounts, alerts, Prophet forecasts on a few areas | **$0**, with limits (sleepy services, small DB, capped emails) |
| **Production-lite** (Railway/Render starter + managed Postgres + Resend paid) | Always-on API, real cron, comfortable DB | **~$15–30** |
| **With heavier ML** (add a small always-on Python worker / GPU bursts for LSTM) | LSTM training + larger area grids | **~$40–80+** |

My honest recommendation: **start on the free tier** to prove each feature, then
move to production-lite once you're happy. LSTM/GPU only when Prophet isn't enough.

---

## 3. Data model (Postgres / Prisma sketch)

```
User        id, email, passwordHash, role(USER|ADMIN), createdAt
Area        id, city, district, name, lat, lon            // seeded from current list
Reading     id, areaId, observedAt, tempC, humidity, aqi, ...  // historical + live cache
Forecast    id, areaId, model(PROPHET|LSTM), targetDate, predTempC, lo, hi, generatedAt
Subscription id, userId, areaId, channel(EMAIL|PUSH), createdAt
AlertRule   id, userId, areaId, metric(TEMP|AQI|UV), op(GT|LT), threshold, active
AlertEvent  id, alertRuleId, firedAt, value, notified
PushDevice  id, userId, endpoint, keys(json)
```

Prisma migrations keep this versioned and reproducible.

---

## 4. ML approach (long-range forecasts)

1. **Ingest history**: pull ERA5 archive (the same Open-Meteo source the site uses
   today) per area into `Reading`.
2. **Phase A — Prophet** (ship first): one model per area per metric. Handles yearly
   seasonality + warming trend well; trains in seconds; gives confidence intervals.
3. **Phase B — LSTM** (later): sequence model for multi-variate, longer horizons;
   only worth it if Prophet's error is too high. Needs more compute.
4. **Schedule**: nightly job retrains/refreshes and writes to `Forecast`; the API
   just reads rows (fast, cheap).
5. **Honesty note**: "long-range" climate-style forecasts are *probabilistic
   trend/seasonal* outputs, not day-accurate weather weeks out — the UI should label
   them as such.

---

## 5. Alerts & notifications

- User creates an `AlertRule` (e.g. "Malir AQI > 200" or "Lahore high > 44°C").
- A cron job (every 15–30 min) checks live conditions against active rules.
- On trigger → write `AlertEvent` → send **email** (Resend) and/or **web push**.
- De-dupe so one heatwave doesn't spam; daily-digest option.

---

## 6. Admin panel

- Protected `/admin` (role = ADMIN via Auth.js).
- Manage areas (add UCs/coordinates), view users & subscriptions, see job health,
  trigger a manual retrain, inspect recent alert events.

---

## 7. Interactive district heat maps

- MapLibre GL base map of Lahore & Karachi divisions.
- Overlay a **choropleth/heat layer** from our gridded `Reading`/`Forecast` data
  (temperature or AQI) per district/tehsil.
- Time slider to scrub current → forecast.
- District boundaries from open GADM/OSM GeoJSON.

---

## 8. Suggested phased delivery

| Phase | Scope | Outcome |
|---|---|---|
| **0** ✅ done | Static live dashboard + full area coverage | Already deployed, free |
| **1** | Next.js app shell + Postgres/Prisma + ingestion of history into DB | Foundation; same UI, now data-backed |
| **2** | Auth + accounts + saved favourite areas | Users can log in |
| **3** | Prophet forecasts + a "long-range" tab | The ML feature, visible |
| **4** | Alert rules + email/push notifications | The alerts feature |
| **5** | Admin panel | Operations |
| **6** | Interactive heat maps | The map feature |
| **7** | LSTM (only if needed) + production hosting hardening | Scale/quality |

Each phase is independently shippable and testable.

---

## 9. What I need from you to start building

1. **Hosting/budget**: free-tier start, or go straight to production-lite (~$15–30/mo)?
2. **Auth**: email/password only, or also Google sign-in?
3. **Notifications**: email, web push, or both? (SMS/WhatsApp cost extra.)
4. **Priority order**: happy with the phase order above, or want (e.g.) alerts before ML?
5. **Repo layout**: keep this repo and add the app alongside `index.html`, or a new repo?

Answer these and I'll begin at Phase 1.
