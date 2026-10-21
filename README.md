# Arnaud & Batista Auto

A typed refactor of the supplied dealership application. The public site, customer forms, company portal and administration dashboard share one validated business model.

## Try it

Open `deliverables/index.html` in a modern browser. This is a **local demonstration** with sample vehicles. Enter the demo administration or company account using the explicitly labelled buttons. Records stay in that browser; this file does not contact the dealership or authenticate a real employee. External random photos and fictional customer requests have been removed.

For the development version:

```sh
# Node 24 LTS; pnpm 11.25.0
pnpm install --frozen-lockfile
pnpm dev:demo
```

The demo starts at the URL printed by Vite. IndexedDB must be available for saves. Storage failures are surfaced, never reported as successful writes. Existing `abauto_v3` data remains untouched. See [migration](docs/MIGRATION.md).

## What the application does

- Browse, filter and sort published stock; inspect vehicle specifications, photos, scores and an adjustable financing estimate.
- Submit a vehicle for assessment through five steps; optionally request a Lisbon appointment or a home inspection.
- Request a visit and track a request with its reference **and a private code**.
- Let authorized partner companies reserve business vehicles at their assigned discount.
- Manage inventory, inspections, offers, purchases, reservations and sales; convert a purchased submission into inventory once.
- Export listing text and photos as a ZIP for manual marketplace publication.

The original inspection rule remains: **61–120 points → retail; 0–60 → companies**. Partner prices round to the nearest €10. Sale-submission mileage uses 5,000 km bands up to 300,000+.

## Project layout

| Location         | Responsibility                                                                               |
| ---------------- | -------------------------------------------------------------------------------------------- |
| `src/domain`     | Runtime schemas, state transitions, prices, access checks, slot availability and idempotency |
| `src/features`   | Catalog, sale form, tracking, partners and admin React screens                               |
| `src/components` | Accessible form controls, dialogs, calendar, image editor and status components              |
| `src/data`       | HTTP adapter, explicit demo adapter, IndexedDB transactions, outbox and migration            |
| `src/app`        | Routing, translations, preferences, business contact settings and PWA controls               |
| `src/sw.ts`      | Workbox precache, public-stock cache, background delivery and push events                    |
| `server`         | Fastify API, OIDC login, PostgreSQL/Drizzle storage, audit records and Web Push              |
| `migrations`     | Database schema; no sample production accounts or records                                    |
| `tests`          | Domain, HTTP, storage, migration, UI and browser workflows                                   |

## Run the full application

1. Provide PostgreSQL and a confidential OIDC application. Register the exact callback `https://YOUR-HOST/auth/callback`.
2. Copy `.env.example` to `.env` and enter your actual origin, database connection and identity-provider values. Production requires HTTPS. The server refuses to start with missing configuration; it never falls back to demo access.
3. Run `pnpm db:migrate`.
4. Create an account JSON file and provision it with `pnpm account:set /path/account.json`. `subject` must be the verified OIDC `sub` from your configured issuer. The role is read from PostgreSQL on each request, never from browser storage or a customer-supplied claim. Example structure:

```json
{
  "subject": "THE_VERIFIED_PROVIDER_SUBJECT",
  "principal": {
    "id": "employee-001",
    "name": "Your administrator",
    "role": "admin",
    "discount": 0,
    "client": {
      "name": "Your administrator",
      "phone": "+351910000000",
      "email": "",
      "address": ""
    }
  }
}
```

Use `role: "partner"` and that company's actual discount/contact information for a company account. Partner IDs must be stable and unique.

5. Run `pnpm build`, then `pnpm start`. Serve port 3000 behind your HTTPS reverse proxy. Keep API, authentication and assets on the same origin. `pnpm dev` starts Vite and the API together after configuration/build; use a localhost `APP_ORIGIN` with `NODE_ENV=development` for a provider that permits a localhost callback.
6. Add real inventory through the admin dashboard, or import a validated backup into an empty database: `pnpm db:import /path/backup.json`.

Web Push is optional: configure all three `VAPID_*` values to expose the admin notification opt-in. Generic notifications contain no customer details. HTTPS plus a supported browser are required. Push and background delivery have not been verified against a live provider in this environment.

## Offline behavior

The hosted build includes a manifest, icons and a service worker. App assets are precached. Only the public `/api/cars` response is cached; authentication, customer tracking, admin records and private requests are never HTTP-cached. Previously fetched stock can be viewed offline with a stale-data notice.

Customers may explicitly save a **sale submission** in the device outbox. It is visibly pending, not a confirmed appointment. Delivery uses a stable idempotency key and rechecks availability on the server. Background Sync, the browser's online event and the visible Send now action share the same delivery code. Transient failures remain pending; conflicts require review; unsent entries older than 24 hours stop retrying. After delivery, the outbox retains the receipt and removes the customer's form data. The user can remove an unsent entry.

Reservations, visits and admin changes require an immediate server response. They are not silently queued. Update prompts let the user finish their work before reloading. The single HTML preview cannot install a service worker: use the complete hosted build for installation, push and background delivery.

## Quality commands

```sh
pnpm lint
pnpm format:check
pnpm typecheck
pnpm test
pnpm build
pnpm build:demo
node scripts/check-standalone.mjs
pnpm exec playwright install chromium
pnpm e2e
pnpm lighthouse
pnpm audit --prod --audit-level high
```

CI runs lint/format → both strict type checks → unit/integration tests → production/demo builds → Lighthouse → Playwright → production dependency audit. Browser reports are retained. Husky installs in a Git checkout; commits use Conventional Commit messages.

Lighthouse 12 in this toolchain has performance, accessibility, best-practices and SEO categories; it does not expose the former PWA score. Installation assets and offline behavior therefore have explicit Playwright checks instead of an invented PWA-100 result.

See [refactor review](docs/REFACTOR_REVIEW.md), [architecture decisions](docs/ADR-001.md), [migration](docs/MIGRATION.md) and [validation results](docs/VALIDATION.md). These distinguish implemented code from services and browser checks that still require verification before launch.
