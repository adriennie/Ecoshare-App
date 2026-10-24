# EcoShare

EcoShare is a neighbourhood sharing app. People give away surplus food and household items, either for free or for community credits.

The main journey is: sign up → share (with AI help) → discover (ranked feed) → request → chat → hand over with a pickup code → settle.

| Part | Stack | Folder |
|---|---|---|
| Mobile and web app | Expo SDK 53, Expo Router, TypeScript | `frontend/` |
| API | Node 20+, Express 5, zod, pino | `backend/` |
| Database, auth, storage, realtime | Supabase (Postgres, RLS, pgvector) | `supabase/` |
| AI / ML | Python 3.11, FastAPI, Hugging Face Inference, scikit-learn, MLflow | `ml-service/` |

## Architecture

```
Expo app ──(Supabase Auth session)──► Supabase Auth
   │  reads via RLS + Realtime (messages, orders, notifications)
   │  uploads photos to storage/<uid>/...
   ▼
backend (Express) ── verifies the JWT on every request ──► Postgres RPCs (business rules, transactions)
   │                                                     ▲
   └── ml-service (FastAPI, holds HF_API_TOKEN) ─────────┘  (backend logs every prediction to ml_predictions)
          └── Hugging Face Inference Providers (VLM, toxicity, NSFW, embeddings)
```

**Identity.** Supabase Auth is the only authority. The backend never takes `user_id` from a request body; every route gets it from the verified access token. Client-supplied identity fields are rejected, because the schemas are strict.

**Rules live in the database.** Requesting, accepting, the pickup code, settlement, transfers and expiry are `SECURITY DEFINER` Postgres functions (`supabase/migrations/*_functions.sql`). They lock rows, run each operation in one transaction, and read the caller from `auth.uid()`.

**Default-deny RLS.** Clients can read only what policies allow. No client can write credits, orders, listings or messages directly.

### Exchange lifecycle

```
requested ─► accepted ─► ready_for_pickup ─► completed
    │            │               │               │
    ▼            ▼               ▼               ▼
declined    cancelled/expired  cancelled/expired/disputed  disputed
```

1. **Request.** The owner cannot request their own item, and a requester can have only one open request per item.
2. **Accept.** The listing row is locked and the item is reserved. A unique index allows only one reservation per listing. Price and fee are copied onto the order. The requester's credits are moved into a hold. All other requests are declined.
3. **Ready for pickup.** The requester taps "Show pickup code". This issues a cryptographically random 6-digit code. Only its bcrypt hash is stored, it expires after 30 minutes, it allows 5 attempts, and clients cannot read the hash column.
4. **Complete.** The owner enters the code. `complete_exchange` verifies it and settles in a single transaction: it releases the hold, writes balanced ledger entries (buyer −total, seller +price, platform +fee), and writes a unique `settlements` row. The call takes an **idempotency key**, so retries never charge twice.
5. A background job expires stale requests and reservations, which releases holds, and archives expired food.

### Credits

- Credits are whole integers. The ledger (`credit_ledger`) is append-only and double-entry, and triggers block updates and deletes even for privileged roles.
- `wallets.balance` is maintained by a trigger and has `CHECK (balance >= 0)`.
- Holds are tracked separately and satisfy `held <= balance`.
- The `wallet_reconciliation` view must always be empty; the job raises an alert log if it is not.
- The welcome bonus is **100 credits**, set in `app_config.signup_bonus_credits`. It is a product decision, and a unique index allows one per user.
- The fee is `ceil(2% of price)`.
- Credits have no cash value.

## AI features (Hugging Face)

All model calls go through `ml-service`, the only component that holds `HF_API_TOKEN`. Each call has a timeout and a fallback, so the app keeps working when AI is down.

**Free mode (no Inference credits).** With `requirements-local.txt` installed, ml-service runs open Hugging Face models on the machine: CLIP (`openai/clip-vit-base-patch32`) chooses the category from the photo, and BLIP (`Salesforce/blip-image-captioning-large`) writes the caption that becomes the title. toxic-bert, the NSFW classifier and MiniLM embeddings also run locally.
- `HF_MODE=auto` (default): uses the hosted API first and switches to local for an hour after a 402/401.
- `HF_MODE=local`: never calls the hosted API.
- `HF_MODE=api`: hosted only.
- Download the weights once (about 3 GB) with `python -m app.local_models --download`.
- On an Apple-silicon Mac a photo takes about 1 s after warm-up, and categories were 14/14 correct on the demo photos.

| Feature | Models (configurable in `ml-service/app/config.py`) | Fallback |
|---|---|---|
| **Smart listing.** A photo becomes a draft title, description, category, condition, allergens, dietary tags, storage tip, shelf life and weight. The user reviews every field. | `Qwen/Qwen2.5-VL-7B-Instruct` via HF Inference Providers, with the versioned prompt `prompts/listing_suggest.v1.yaml` | BLIP caption + `facebook/bart-large-mnli` zero-shot category |
| **Recommendations.** The "For you" feed and "Similar nearby". | `sentence-transformers/all-MiniLM-L6-v2` embeddings in **pgvector**; in-house gradient-boosted ranker | Heuristic ranker (similarity, distance, freshness, popularity, urgency) |
| **Trust & safety.** Moderation of listings and messages, plus food-safety rules. | `unitary/toxic-bert`, `Falconsai/nsfw_image_detection`, plus deterministic rules (prohibited items, expired food, unsafe food age) | Rules only |
| **Reliability.** P(exchange completes), shown to owners as a badge and never used to block anyone. | In-house calibrated logistic regression | Beta prior from the user's own history |
| **Impact.** kg CO₂e avoided and kg of waste diverted per completed exchange. | Versioned emission-factor table (`impact_factors.v1.json`) plus weight parsing | Job retries |

Moderation thresholds:
- **Message toxicity ≥ 0.93:** the message is blocked.
- **≥ 0.80:** the message is flagged. It is hidden for the recipient behind "tap to view" and a report is filed.
- **Listings ≥ 0.80:** the listing is set to `pending_review` and enters the admin moderation queue.

## MLOps

| Stage | Where |
|---|---|
| Event logging (impressions with served features, views, saves, dismisses; server-side requests and completions) | `interactions` table, `POST /api/events`, `backend/src/routes/feed.js` |
| Prediction logging (model, version, prompt version, latency, fallback, input hash) | `ml_predictions` table, `backend/src/services/predictions.js` |
| Feature contract shared by training and serving (prevents skew) | `ml-service/app/features.py` |
| Feature snapshots (versioned parquet + manifest with data hash) | `pipelines/build_features.py` |
| Training with experiment tracking and model registry | `pipelines/train_ranker.py`, `train_risk.py` → MLflow (`@challenger` alias) |
| Promotion gate (challenger must beat both the champion and the non-ML baseline on an out-of-time holdout) | `pipelines/evaluate.py` → `@production` alias, exported to `models/production/` and hot-reloaded via `/admin/reload` |
| Prompt and model evals (offline in CI; live with HF) | `evals/run_evals.py`, `evals/thresholds.yaml` |
| Drift monitoring (PSI per feature; fails the job on alert) | `pipelines/drift_report.py`, `.github/workflows/mlops.yml` (weekly) |
| Batch scoring (reliability badges) | `pipelines/score_users.py` |
| Serving metrics | `GET /metrics` on ml-service (Prometheus: latency, errors, fallback rate, score distribution) |

Until real interaction data exists, the heuristic ranker and the prior serve traffic. `make pipeline-synthetic` exercises the whole loop on synthetic data.

## Getting started

### 0. Security first (one-time)

Earlier commits contained a Supabase **service-role key**, which bypasses RLS. In the Supabase dashboard (Project settings → API), **roll the service-role key and the JWT secret**, then store the new key only in `backend/.env`. Removing it from the code does not undo the exposure: the old key stays in git history until it is rotated.

### 1. Database

Hosted project:

```bash
npx supabase link --project-ref <your-ref>
npx supabase db push
```

Or local, with Docker:

```bash
npx supabase start
npx supabase db reset
```

`db reset` applies the migrations and `seed.sql`. The seed creates demo users ana@ecoshare.test and ravi@ecoshare.test, both with the password `ecoshare-demo`.

Then, in Supabase Auth settings:
- Enable email confirmations.
- Add the redirect URLs `ecoshare://**` and `exp://**`.
- Optionally enable the Google provider.

The previous ad-hoc tables (`users`, `itemdata`, `credit_stack`, …) are **not** migrated automatically, so start from a fresh project or export the old data first.

### Demo data (dev/staging only)

```bash
ml-service/.venv/bin/python scripts/seed_demo.py
```

This creates 12 demo neighbours in Gurugram (`<name>@demo.ecoshare.test`, for example `diya@demo.ecoshare.test`, password `ecoshare-demo`), about 100 listings with photos, and exchange history driven through the real RPCs. Run it again with `--more` to add a fresh batch, for example after food listings expire.

### 2. ml-service

```bash
cd ml-service
cp .env.example .env
make install
make test
make serve
# optional, free local models: VIRTUAL_ENV=.venv uv pip install -r requirements-local.txt
```

Put your HF token in `.env`. `make serve` starts the API on http://localhost:8000.

### 3. Backend

```bash
cd backend
cp .env.example .env
npm install
npm test
npm run dev
```

Fill in your Supabase URL and keys plus `ML_SERVICE_TOKEN` in `.env`. `npm run dev` starts the API on http://localhost:3000.

### 4. App

```bash
cd frontend
cp .env.example .env
npm install
npx expo start
```

`EXPO_PUBLIC_API_URL` must point at the backend. On a phone, use your computer's LAN IP.

Or run the API and ML together with `docker compose up --build`. Add `--profile mlops` for the MLflow UI on :5001.

## Tests

| Command | What it proves |
|---|---|
| `cd backend && npm test` | Real migrations on embedded Postgres (PGlite). RLS isolation: a user cannot edit another user's listing, read someone else's chat, mint credits or self-promote to admin. Reservations, pickup-code expiry and lockout, atomic settlement, idempotent retries, holds, transfers. Plus HTTP tests: auth required, identity taken only from the token, validation, moderation, ML fallbacks. |
| `cd ml-service && make test && make eval` | HF client fallbacks, the feature contract, the train → gate → export → serve round trip, PSI drift detection, prompt-parsing evals. |
| `cd frontend && npm run typecheck && npm test` | Types, the API client, and listing validation. |

CI (`.github/workflows/ci.yml`) runs all of these, the synthetic MLOps loop, an Expo web build, and a gitleaks secret scan.

One limitation: PGlite is single-connection. Row-lock behaviour under true concurrency is enforced by `FOR UPDATE` and the unique indexes, but it is only exercised for real against Postgres (`supabase start`).

## API overview

All routes are under `/api` and return errors as `{ error: { code, message } }`.

| Area | Endpoints |
|---|---|
| Profile | `GET/PATCH/DELETE /me` |
| Listings | `GET /categories`, `GET /listings` (search, filters, pagination), `GET /listings/:id`, `POST /listings`, `PATCH/DELETE /listings/:id`, `POST/DELETE /listings/:id/save`, `GET /saved`, `GET /listings/mine` |
| Feed | `GET /feed` |
| Exchanges | `GET /orders`, `GET /orders/:id`, `POST /orders`, `POST /orders/:id/{respond,ready,pickup-code,complete,cancel,dispute}` |
| Chat | `GET/POST /chats`, `GET /chats/:id`, `GET/POST /chats/:id/messages`, `POST /chats/:id/read` |
| Credits | `GET /credits/wallet`, `GET /credits/ledger`, `POST /credits/transfer` |
| Other | `GET /notifications`, `POST /notifications/:id/read`, `POST /reports`, `POST/DELETE /blocks`, `POST /events`, `POST /ai/listing-suggest`, `GET /impact/{me,community}` |
| Admin | `GET /admin/reports`, `POST /admin/reports/:id/resolve`, `POST /admin/orders/:id/resolve-dispute` |
