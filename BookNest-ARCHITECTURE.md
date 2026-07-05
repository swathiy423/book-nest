# BookNest — Architecture & Delivery Plan

BookNest is a large product (library + reading rooms + clubs + marketplace + social + admin). This doc scopes it into buildable phases and locks the technical decisions so future work stays consistent.

## 1. Scope reality check

The full brief is roughly 8 product surfaces, each comparable to a standalone app:

| Surface | Comparable to |
|---|---|
| Personal library + analytics | Goodreads core |
| Virtual reading rooms + presence | A lightweight Discord voice-channel clone |
| Book clubs | Discord server + polls |
| Exchange marketplace | A local Craigslist/Depop vertical |
| Social graph + feed | Twitter-lite |
| Admin | An internal ops tool |

Realistic path: ship a vertical slice per surface, wire them to a shared schema, and let features grow module-by-module rather than trying to hand-build all of them at once.

## 2. Stack decisions

- **Frontend:** React 18 + Vite, React Router, TanStack Query for server state, Zustand for local UI state, Tailwind CSS for styling, Framer Motion for the animation layer.
- **Backend:** Node.js + Express, REST API (GraphQL is overkill for this shape of data).
- **Database:** PostgreSQL via Prisma ORM. Prisma schema is the single source of truth for models.
- **Realtime:** Socket.IO for reading-room presence, club chat, marketplace messaging, notifications.
- **Auth:** JWT access token (short-lived) + httpOnly refresh cookie. Google OAuth via `passport-google-oauth20`. TOTP (e.g. `otplib`) for 2FA.
- **File storage:** S3-compatible bucket for book covers / exchange photos, signed upload URLs.
- **Search:** Postgres full-text search to start (tsvector columns); swap for Meilisearch/Typesense if catalog size demands it later.
- **Background jobs:** BullMQ + Redis for reminders, digest emails, badge calculation.

## 3. Data model (core entities)

```
User { id, email, passwordHash, name, avatarUrl, bio, genres[], goalYearlyBooks, trustScore, createdAt }
Book { id, title, author, coverUrl, description, genres[], pageCount, isbn }
UserBook { userId, bookId, status(enum: reading|completed|want|favorite|wishlist|dropped), progressPage, rating, review, startedAt, finishedAt }
ReadingSession { id, userId, bookId, roomTheme, durationSec, startedAt }
Club { id, name, description, isPrivate, bannerUrl, ownerId }
ClubMember { clubId, userId, role(enum: owner|mod|member) }
ClubPoll { id, clubId, question, options[], closesAt }
Listing { id, ownerId, bookId, condition, photos[], status(enum: available|pending|exchanged), preferredGenres[] }
ExchangeRequest { id, listingId, requesterId, status, scheduledAt }
Notification { id, userId, type, payload(json), readAt }
Follow { followerId, followingId }
```

Each of these becomes a Prisma model 1:1 — see `scaffold/backend/prisma/schema.prisma`.

## 4. API surface (v1)

```
POST   /auth/signup            /auth/login            /auth/refresh        /auth/2fa/verify
GET    /me                     PATCH /me
GET    /books?query=&genre=    GET /books/:id
GET    /library                POST /library/:bookId   PATCH /library/:bookId
GET    /clubs                  POST /clubs             GET /clubs/:id      POST /clubs/:id/join
POST   /clubs/:id/polls        POST /clubs/:id/messages
GET    /marketplace/listings   POST /marketplace/listings   POST /marketplace/listings/:id/request
GET    /notifications          PATCH /notifications/:id/read
GET    /search?q=
```

Realtime channels (Socket.IO namespaces): `/rooms/:roomId`, `/clubs/:clubId`, `/dm/:threadId`, `/notifications/:userId`.

## 5. Frontend route map

```
/                       marketing landing (public)
/login /signup /onboarding
/app                    dashboard
/app/library            personal library
/app/library/:bookId    book detail + notes
/app/rooms              room picker
/app/rooms/:roomId      active reading room (timer, ambience, presence)
/app/clubs              club directory
/app/clubs/:clubId      club page (feed, polls, events)
/app/marketplace        listings grid + map
/app/marketplace/:id    listing detail + messaging
/app/profile/:userId    public profile
/app/notifications
/admin/*                admin dashboard (role-gated)
```

## 6. Design system

See `prototypes/` for the visual language: a "reading lamp at midnight" theme — deep ink backgrounds, warm brass accent, a serif display face for headings, and a bookshelf-spine motif used as the site's signature structural element (section dividers, loading states, progress bars). Full token list lives at the top of each prototype's `<style>` block.

## 7. Phased delivery order

1. Auth + onboarding + dashboard shell
2. Personal library + book detail + analytics
3. Reading rooms (solo first, then "Read Together" presence)
4. Book clubs (feed + polls before full events/calendar)
5. Marketplace (listing + request flow before real-time negotiation/map)
6. Social layer (follow, profile, feed)
7. Notifications center + global search (cut across everything above)
8. Admin dashboard

## 8. What's delivered alongside this doc

- `scaffold/` — runnable-shape Express + Prisma backend and Vite + React frontend skeleton (routes, schema, auth middleware stubbed in). Needs `npm install` in an environment with network access.
- `prototypes/landing.html` — the marketing landing page, fully static/interactive, no build step needed.
- `prototypes/dashboard.html` — the logged-in dashboard, same approach.
- `prototypes/reading-room.html` — deep-dive on the Virtual Reading Room feature (ambient themes, Pomodoro timer, focus mode).

Open each `.html` file directly in a browser to preview; no server required.
