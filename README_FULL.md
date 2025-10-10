# LinkrNZ — Architecture, Flows & Wireframes


LinkrNZ — Architecture, Flows & Wireframes

Complete technical architecture, user flows, and wireframes for LinkrNZ — an escort marketing directory and booking/marketing platform targeted at New Zealand. This document assumes escorts operate as independent contractors; LinkrNZ provides marketing, profiles, search, messaging, payments and optional booking tools.

---

1. Summary / Goals

- Build a scalable, secure marketing directory and SaaS platform for escorts and clients.
- Primary actors: Escort (Provider), Client (Consumer), Admin / Moderation, Affiliate / Partner.
- Core features: public profiles, advanced search, image galleries, availability & bookings, messaging, payments/subscriptions, reviews (moderated), geolocation, analytics dashboard for escorts, admin moderation & compliance tools.

---

2. Recommended Tech Stack (opinionated)

- Backend: Node.js + TypeScript (NestJS or Express with TypeORM/Prisma)
- Database: PostgreSQL (relational), Redis for caching & queues
- File storage: DigitalOcean Spaces (S3 compatible) with signed URLs
- Search: ElasticSearch or MeiliSearch for fast text / geo search
- Payments: Stripe (if supported for this business model) or local NZ processors (Windcave/Realex/Paystation) — include PCI-compliance plans
- Auth: JWT + Refresh tokens, optional WebAuthn, 2FA (TOTP)
- Frontend: Next.js (React) — server-side rendering for SEO, static generation for public pages
- Mobile: React Native (Expo) or Flutter for cross-platform apps
- Messaging / Realtime: WebSockets via Socket.IO or Pusher; fallback via polling
- Queue / Background jobs: BullMQ with Redis
- Monitoring: Sentry, Prometheus/Grafana
- CI/CD / Hosting: DigitalOcean App Platform or Docker on DigitalOcean Droplets / Kubernetes (DO Kubernetes) + GitHub Actions
- Secrets: HashiCorp Vault or encrypted secrets via GitHub Actions

---

3. System Architecture (high-level)

1. Clients: Browser (Next.js), Mobile App (React Native) and third-party integrations.
2. CDN + Edge: Cloudflare in front of static assets and as WAF.
3. API Layer: RESTful/GraphQL API (Node/TS) behind authentication and rate-limiting.
4. Worker Layer: Background jobs for video/image processing, email sending, digest generation, payouts.
5. Data Layer: Postgres, Redis, Search index, Object Storage.
6. Admin Console: Admin-only UI to handle moderation queues, payouts, disputes, content takedowns.

Diagram (textual):

[Browser/Mobile] -> [CDN/WAF] -> [Next.js frontend / API Gateway] -> [API Servers]
                                       |-> [Auth Service]
                                       |-> [Search Service]
                                       |-> [Worker Queue (Redis)] -> [Workers]
                                       |-> [Postgres]
                                       |-> [Object Storage]

---

4. Backend: Services & Responsibilities

4.1 API Gateway / BFF
- Hosts Next.js server-side rendering and also acts as BFF for mobile apps.
- Handles SSR for public profile pages for SEO (Open Graph tags) and structured data.

4.2 Auth Service
- Signup flow for escorts requires identity verification (optional/required depending on business rules).
- OAuth for admin/staff, email link or password + TOTP for escorts/clients.
- Roles: guest, client, escort, admin, moderator, affiliate.
- Rate-limit and device management for sessions.

4.3 Profile Service
- CRUD for profiles, galleries, services offered, rates, availability blocks, amenities.
- Media upload: images/videos -> virus scan -> moderation queue -> store in S3-compatible storage.

4.4 Search & Discovery
- Index profiles with geolocation, tags, price, availability, language, ratings.
- Support faceted search and relevance tuning (boost verified escorts, paid promoted profiles).
- Use geo queries to return distance-sorted results.

4.5 Messaging & Notifications
- In-app messages stored encrypted-at-rest. Short-lived message history for compliance rules.
- Push notifications via FCM (Android) & APNs (iOS); fallback to email.
- Rate-limiting and profanity filter.

4.6 Bookings & Payments
- Booking object: client_id, escort_id, start, end, amount, status.
- Support PAY_NOW and PAY_ON_MEET patterns; hold funds in escrow for PAY_NOW until completion or cancellation.
- Stripe Connect or equivalent for payouts to escorts (managed accounts). Consider KYC, tax reporting, payout schedule.

4.7 Moderation & Compliance
- Auto-moderation (image nudity classifier, profanity detection) + human moderation queue.
- Takedown workflow, appeals, and automated enforcement for repeated violations.

4.8 Admin & Reporting
- Admin panels for content moderation, payments reconciliation, dispute resolution, user bans, analytics.

---

5. Data Model (core tables)

- users (id, role, email, password_hash, phone, verified, kyc_status, created_at)
- profiles (id, user_id, display_name, bio, services_json, tags, price_min, price_max, location_point, is_promoted)
- media (id, profile_id, url, type, status)
- bookings (id, client_id, profile_id, start, end, status, amount, currency)
- messages (id, conversation_id, sender_id, content_encrypted, read, created_at)
- reviews (id, profile_id, author_id, rating, content, moderated)
- payments (id, booking_id, gateway_id, status, amount, fee, payout_id)
- payouts (id, user_id, amount, currency, status, processed_at)
- search_index (external ES/Meili index)

---

6. Frontend (Web) — Key pages & components

- Homepage: hero search, featured/promoted profiles, categories
- Search results: filters (location radius, price, services, language, verification), map + list toggle
- Profile page: images, description, availability, services, CTA (message/book), reviews
- Booking flow: date/time picker, extras, confirmation, payment
- Messaging: conversation list, conversation view, attachments
- Escort dashboard: profile editor, calendar, bookings, earnings, analytics, promote profile
- Admin console: moderation queue, payments, user management

UI components: SearchBar, ProfileCard, Gallery, AvailabilityCalendar, FilterPanel, CheckoutModal, ChatWidget, NotificationsToast.

SEO: Use Next.js SSG for profile pages and dynamic meta tags + structured data (JSON-LD).

Accessibility: WCAG AA compliance for forms, images (alt text), keyboard nav and color contrast.

---

7. Mobile Apps — Feature parity & considerations

- Approach: React Native (single codebase) with native modules for payments and push notifications.
- Screens: Onboarding (escort verification), Explore/Search, Profile, Chat, Bookings, Wallet/Earnings, Settings.
- Native integrations: camera/photo picker for uploads, calendar sync, local notifications.
- Offline: cache recent messages & profile data for quick view; queue outgoing messages while offline.

Security: biometric unlock for sensitive sections, encrypted local storage (Keychain/Android Keystore).

---

8. User Flows (detailed)

8.1 Client: Discover → Book → Meet
1. Client lands on homepage or search.
2. Use filters and map to find a profile.
3. Open profile, view gallery, read bio and reviews.
4. Click Book or Message.
5. If Book: choose date/time, add extras, pay (escrow or full), booking created.
6. Escort accepts or auto-accepts (configurable). Confirmation notification sent.
7. After meeting, client can leave review. Platform releases funds to escort (if escrow) minus fees.

8.2 Escort: Signup → Verify → Publish
1. Escort signs up with email/phone.
2. Required identity verification step (ID upload, selfie for liveness check) — enters moderation queue.
3. Once verified, create profile, upload images (watermark/blur on pending), set availability and pricing.
4. Optionally pay for promoted listing.
5. Receive bookings, manage calendar, request payouts.

8.3 Messaging Flow
- When client messages escort, a conversation is created. Escort receives push notification.
- Sensitive content rules: images flagged by auto-moderation are held for review; sending images may be restricted until verification.

8.4 Admin Moderation Flow
- New or edited profiles/media flagged by auto-filters are pushed into moderation queue.
- Moderator reviews and either approves, requests changes, or rejects/blocks.
- Appeals flow: user submits evidence, admin re-evaluates.

---

9. Wireframes (text + simple visuals)

These are low-fidelity wireframes. Use them as starting point for a UI designer.

9.1 Homepage (desktop)

--------------------------------------------------
| Top Nav: Logo | Search | Become a Provider | Login |
--------------------------------------------------
| Hero: Large search bar (Location + What)         |
| Featured carousel: Promoted profiles             |
| Category tiles: New / Independent / Couples      |
| CTA: How it works | For providers               |
--------------------------------------------------
| Footer: About | Safety | Terms | Contact         |
--------------------------------------------------

9.2 Search Results (list + map)

--------------------------------------------------
| Left column: Filters (price, services, verified) |
| Middle column: Profile cards (image, name, snippet)|
| Right column: Map with pins                       |
--------------------------------------------------

9.3 Profile page

--------------------------------------------------
| Gallery (carousel) | Name, Verified badge         |
| Bio / Services      | Price range, Language         |
| Availability calendar| Book button (sticky)         |
| Reviews & Ratings   | Message button               |
--------------------------------------------------

9.4 Escort Dashboard (mobile)

[Header: Earnings | Notifications]
[Profile completeness meter]
[Buttons: Edit profile | Calendar | Bookings | Payouts | Promote]
[Recent bookings list]

---

10. Security & Privacy

- TLS everywhere, HSTS, secure cookies, SameSite, CSP.
- Store PII and KYC documents encrypted with strong encryption (AES-256) and separate keys.
- Comply with local NZ regulations on sex work advertising; ensure takedown & consent flows for minors and illegal content.
- Rate-limit endpoints and use account lockouts for suspicious behavior.
- Pen testing and code audits before launch.

---

11. Payment, Fees & Payouts

- Platform fee model: subscription tiers for escorts (free basic; plus/promoted tiers), commission per booking (e.g., 10-25%), or combination.
- Payout schedule: weekly or manual; support direct bank transfers or Stripe Connect payouts.
- Refund & dispute rules built into booking lifecycle (cancellation windows, no-shows).

---

12. Moderation & Legal Compliance

- KYC: verify escorts with government ID + liveness.
- Age verification mandatory (strict checks and automatic rejection if failed).
- Keep logs for disputes; only retain sensitive documents for minimum legally required period.
- Clear terms of service forbidding underage content, exploitation, or trafficking; integrate reporting tools.

---

13. Deployment & Operations

- Use GitHub Actions for CI: run tests, linters, build images, deploy to staging, then to production.
- Blue/Green or Canary deploy strategy to reduce downtime.
- DB backups daily + point-in-time recovery for Postgres.
- Autoscaling for API servers and workers.

---

14. Roadmap / MVP Feature Set

MVP (Phase 1)
- Public searchable profiles with galleries (manual moderation)
- Escort signup and KYC pipeline (basic)
- Client booking flow (pay now / manual payment placeholder)
- Messaging (in-app basic)
- Escort dashboard: manage profile, availability, bookings
- Admin moderation console

Phase 2
- Escrow payments, payouts, Stripe Connect
- Advanced search ranking, promoted listings
- Reviews with moderation
- Mobile apps (iOS/Android)

Phase 3
- AI-assisted moderation, recommendations, advertiser dashboard
- Affiliate & referral program, analytics suite

---

15. Developer Deliverables (what to build first)

1. Project scaffold: mono-repo with apps/frontend, apps/api, packages/ui, packages/db.
2. Authentication + user model + migrations.
3. Profile CRUD + media upload pipeline (S3 + moderation queue).
4. Search index integration & basic search UI.
5. Booking model & lightweight checkout flow (sandbox payments).
6. Admin moderation UI.

---

16. Appendix: API contract examples (sample endpoints)

- POST /api/v1/auth/signup — create account
- POST /api/v1/auth/login — returns access + refresh token
- GET /api/v1/profiles?lat=&lng=&q=&page= — search
- GET /api/v1/profiles/:id — profile detail
- POST /api/v1/profiles/:id/bookings — create booking
- POST /api/v1/uploads — media upload (returns signed URL)
- GET /api/v1/messages/:conversationId — fetch messages
- POST /api/v1/admin/moderation/:itemId/approve — moderator action

---

End of document.
