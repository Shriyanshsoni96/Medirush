# MediRush

Medicine delivery platform that connects **customers**, **local pharmacies** and **delivery riders**
through one backend. Built with the MERN stack (MongoDB, Express, React, Node).

```
MediRush/
├── backend/        Express 5 + Mongoose 9 REST API  (/api/v1, port 5000)
├── customer-app/   React 19 customer web app         (port 5173)
├── pharmacy-app/   React 19 pharmacy partner app     (port 5174)
├── rider-app/      React 19 rider app, mobile-first  (port 5175)
├── postman/        API collection (62 requests)
├── render.yaml     Render blueprint for the API
└── DEPLOYMENT.md   Atlas → Render → Vercel guide
```

## How an order flows

1. **Customer** searches the shared master catalog, builds a cart and checks out. The server re-prices the cart
   (`/orders/quote`) and blocks checkout unless **one** nearby pharmacy (approved, online, serves the pincode)
   has every item in stock.
2. **Dispatch** offers the order to the best-ranked eligible pharmacy for **7 minutes**. Decline, timeout,
   going offline or a stock change passes it to the next one; if none can, the order is cancelled and the
   customer is never charged.
3. **Pharmacy** accepts (stock deducted atomically, demo payment captured) → preparing → packed → ready.
4. **Rider** in that area accepts (one active delivery per rider) → at pharmacy → picked up → in transit →
   arrived → enters the customer's **4-digit code** (plus prescription / cash checks) → delivered (₹25 earned).
5. **Customer** tracks every step live and leaves a review.

Order, payment and delivery statuses are separate state machines; every change is a status-guarded write inside
a MongoDB transaction, so races (double accepts, accept vs. cancel, two riders) always have exactly one winner.

## Run everything locally

Requirements: Node 20+ (tested on 22). No MongoDB install needed for development.

```bash
# 1. API with an in-memory MongoDB replica set, freshly seeded with Pune demo data
cd backend && npm install && npm run dev:memory

# 2. Each app in its own terminal
cd customer-app && npm install && npm run dev    # http://localhost:5173
cd pharmacy-app && npm install && npm run dev    # http://localhost:5174
cd rider-app    && npm install && npm run dev    # http://localhost:5175
```

Demo logins (all share one generated password) are written to `backend/seed/SEED_ACCOUNTS.md` (git-ignored).
The in-memory database — and that password — reset every time the API restarts.

| App | Try with | Notes |
|---|---|---|
| Customer | aarav.mehta@example.com | Home pincode 411038 |
| Pharmacy | sanjeevani.kothrud@example.com | Approved, online, full stock |
| Pharmacy | baner.healthpoint@example.com | **Pending** → shows the approval screen |
| Rider | rahul.shinde@example.com | Approved, online, covers 411038/411004/411001 |
| Rider | vikas.pawar@example.com | **Suspended** |
| Admin (Postman) | admin@example.com | Approvals, interventions, catalog |

**Using MongoDB Atlas instead:** copy `backend/.env.example` to `backend/.env`, set `MONGODB_URI` and
`JWT_SECRET`, then `npm run seed` (wipes and loads demo data) and `npm run dev`.

## Tests

```bash
cd backend && npm test      # 120 tests, each file on its own in-memory replica set
```

Covers auth and approval gates, an authorization matrix (every role × every protected area, forged tokens,
operator injection, CORS), catalog and availability, billing and checkout blocking, the full order lifecycle,
sequential dispatch and expiry, race conditions, rider delivery, delivery-code lockout, failed deliveries and
returns, reviews, inventory, admin tools and the production scripts.

The three frontends were verified by building them and clicking through complete flows in a browser
(desktop and 375 px mobile); they have no automated UI tests yet.

## Configuration

All business values live in `backend/config/business.js` and are overridable by environment variables
(see `backend/.env.example`): delivery fee ₹25, free delivery above ₹499, 7-minute pharmacy offer window,
₹25 rider earning, 5 delivery-code attempts, 15-minute payment window, and more. Frontends read the public
ones from `GET /config/public`, so nothing is hard-coded in the UI.

## Scripts (backend)

| Command | Purpose | Safe in production |
|---|---|---|
| `npm run dev:memory` | In-memory DB + seed + API | — (dev only) |
| `npm run dev` / `npm start` | API with watch / API | ✅ |
| `npm run seed` | **Wipe** DB and load demo data | ❌ refuses when `NODE_ENV=production` |
| `npm run seed:catalog` | Upsert categories, medicines, templates (no deletes, no accounts) | ✅ |
| `npm run create:admin` | Create an admin from `ADMIN_*` env vars | ✅ |
| `npm test` | Test suite | — |

## API overview

Full list with request bodies: `postman/MediRush.postman_collection.json` (import into Postman, set `password`,
run the *Login* requests first — tokens are saved automatically).

| Area | Base path | Who |
|---|---|---|
| Health, config, catalog, templates, featured pharmacies | `/health`, `/config/public`, `/categories`, `/medicines`, `/templates`, `/pharmacies/featured` | public |
| Auth (separate login per app) | `/auth/{customer,pharmacy,rider,admin}/login`, `/auth/*/register`, `/auth/me` | — |
| Profile, addresses, frequently ordered | `/customers/me`, `/addresses` | customer |
| Quote, orders, demo payment, cancel, review, reorder | `/orders` | customer |
| Offers, order workflow, returns, inventory, catalog, sales | `/pharmacy/*` | approved pharmacy |
| Requests, delivery steps, complete/fail, earnings | `/rider/*` | approved rider |
| Notifications | `/notifications` | any signed-in user |
| Approvals, order/delivery interventions, catalog | `/admin/*` | admin |

## Deployment

See [DEPLOYMENT.md](DEPLOYMENT.md): MongoDB Atlas → API on Render (`render.yaml`) → three Vercel projects
(`vercel.json` in each app) → set `CORS_ORIGINS` → `seed:catalog` + `create:admin` → approve partners.

## Development notes

- Frontends use **Vite 7** (Rollup/esbuild). Vite 8's Rolldown native binary is blocked by Windows Application
  Control on the original dev machine.
- The three apps deliberately **copy** the small design system (UI kit, hooks, formatters) instead of sharing a
  package, so each deploys independently. Keep them in sync when changing shared components.
- Dev in-memory MongoDB is pinned to 7.0.24 (`MONGOMS_VERSION` to override) so dev and tests reuse one cached binary.

## Known limitations (v1) and next steps

- Payments are simulated (authorise on checkout, capture on pharmacy acceptance, void/refund on cancel).
- Prescription items are checked by the rider at the door; a pharmacist upload-and-verify step is the
  recommended upgrade.
- "Nearby" = pincode service areas; no maps or live GPS. Pharmacy documents already have a `location` field.
- Updates arrive by polling (5–20 s). Socket.IO can replace polling without changing the services.
- No admin web UI yet — admin work is done through the API/Postman.
- Deliveries wait in "Searching for rider" until a rider covering the pickup pincode is online; ensure rider
  coverage matches pharmacy service areas, or add an alert for long waits.
- Background jobs run in the API process; use an always-on host in production.
