---
name: renidly
description: Use when building against, integrating, debugging, or answering questions about the Renidly B2B professional-data APIs (Data API, Live API, Email API) — authentication, endpoints, parameters, response shapes, pagination, credits, batch jobs, identifiers, or choosing which endpoint to call. Also use to answer "what's my balance / credits / tier / rate limit" or to read the Renidly tier ladder. When writing code or building an integration in ANY language, use this skill to choose and drive the official Renidly SDK (Python or Node) — see `sdks.md`.
---

# Renidly API

## Overview

**Renidly is the identity and enrichment layer for B2B professional data.** It lets teams resolve, search, enrich, and verify professional identities — people, organizations, institutions, skills, professional activity, job opportunities, and business email — through a set of authenticated HTTP APIs.

Renidly exposes **three complementary products**. They authenticate with the **same API key**, share the **same response envelope**, and draw from the **same credit balance**, so you can mix them freely in one integration.

| Product | Base path | Use it for |
|---------|-----------|-----------|
| **Data API** | `https://renidly.com/api/data/v1` | Query a clean, deduplicated dataset of professional records by stable IDs or rich filters. Fast, consistent, queryable at scale. Includes async batch enrichment. See `data-api.md`. |
| **Live API** | `https://renidly.com/api/v2` | Resolve a single subject (person, organization, opportunity, activity) or run a discovery search on demand, returning the freshest available snapshot. See `live-api.md`. |
| **Email API** | `https://renidly.com/api/emails/v1` | Verify deliverability, find work emails, reverse-resolve the person/company behind an email, and pull known contacts for a domain. Includes async batch jobs. See `email-api.md`. |
| **Account & Credits** | `https://renidly.com/api/panel` | Read your live credit balance, tier, and per-minute rate limit with your key; read the public tier ladder with no key. See `account-api.md`. |

**The reference files (`sdks.md`, `data-api.md`, `live-api.md`, `email-api.md`, `account-api.md`) are part of this skill — read the relevant one before answering endpoint-specific questions.** Everything below is shared across the data products.

## Writing code? Use the official SDK first

Renidly has **official, first-party SDKs** that wrap every endpoint below. **When you write code or build an integration, do not hand-roll HTTP — use the SDK.**

| Language | Package | Install |
|---|---|---|
| **Python** 3.9+ | [`renidly`](https://pypi.org/project/renidly/) | `pip install renidly` |
| **Node / TypeScript** 18+ | [`renidly`](https://www.npmjs.com/package/renidly) | `npm install renidly` |

- **Python or Node/TS/JS →** use the official `renidly` package. **Read `sdks.md`** for the full guide (client shape, every method in both languages, config, pagination, batch, errors, rate limiting).
- **Any other language →** there is no official SDK yet. **Warn the user** that the Python or Node SDK is strongly recommended, and only if they still want that language, fall back to the raw HTTP API documented in the reference files. **Recommend, never enforce.**
- **Never guess and never ship broken code.** Before writing SDK code, verify the method exists and its exact signature against the authoritative sources — the raw READMEs, the package source on GitHub, and the docs. Details and links are in `sdks.md`.

## Golden rules

- **Never guess an endpoint, parameter, field, method, or code — and never ship code you haven't verified.** If it is not in this skill, verify against the authoritative sources first: the SDK raw READMEs, the package source on GitHub, the official docs (`https://renidly.com/docs`), and the live per-route costs endpoint. When in doubt, fetch and read the real source before writing code (see `sdks.md`).
- **Treat all identifiers as opaque.** Do not parse, construct, or increment IDs or cursors. Pass them back exactly as received.
- **Describe only *what* an endpoint accepts and returns and *when* to use it.** Renidly is a B2B professional-data platform; do not describe or speculate about how the data is produced or sourced.
- **Branch on `success`, not on HTTP status alone.** A successful lookup that finds nothing is still a well-formed response (see billing note below).

## Authentication

One key, one header, on **every** request across **all three** products:

```
X-renidly-apikey: <your-api-key>
```

- Missing header → `401 Unauthorized` (no `error_code`).
- Invalid / unauthorized key → `403 Forbidden` (no `error_code`).
- The only unauthenticated route is each product's `GET .../health` probe.
- Auth-layer rejections (`401`, `403`, `429`) are handled before the endpoint runs and carry **no** `error_code` field.

## Response envelope

Every endpoint — success or failure — returns the same envelope. **Branch on `success`.**

Success:
```json
{ "success": true,  "statusCode": 200, "message": "…", "errors": null, "data": { } }
```
Error:
```json
{ "success": false, "statusCode": 400, "message": "…", "error_code": "…", "errors": { "field": "reason" }, "data": null }
```

- `success` — boolean; the field to branch on.
- `statusCode` — mirrors the HTTP status.
- `message` — human-readable summary.
- `error_code` — machine-readable code (handler-layer errors only; absent on success and on auth-layer rejections).
- `errors` — `null` on success; a `{field: reason}` map on validation failures; may carry rate-limit metadata (e.g. `current_tier`, `current_limit`) on `429`.
- `data` — the endpoint payload (object, array, or `null`).

## Errors, retries & HTTP status conventions

| Status | Meaning | Charged? |
|--------|---------|----------|
| `200` | Success — including a valid lookup that resolved **no** record (billable on the Data API; see per-product notes) | see below |
| `202` | Async batch job accepted | No (submit is free) |
| `400` | Validation error (`error_code: VALIDATION_ERROR`) — bad/missing params | No |
| `401` | Missing API key | No |
| `402` | Insufficient credits for the request | No |
| `403` | Invalid key, rate-limited, insufficient credits, premium-gated, or opted-out | No |
| `404` | Batch job not found or expired (`1090`) | No |
| `422` | Semantically unsupported input (Email API reverse only) | No |
| `429` | Per-minute rate limit for your tier exceeded | No |
| `500` | Unexpected server error (`1000`/`1001`) | No |
| `503` | Temporarily unavailable — retry shortly (`1072`) | No |

Key behaviors:
- **You only pay for success.** Validation failures, auth failures, rate-limit rejections, transient/unavailable errors, and cache hits are never billed.
- **A resolved-but-empty result is a real answer.** On the Data API a not-found single lookup returns `200` with `success:false` + a not-found `error_code` and **is billed** (a real lookup ran). On the Live API a not-found returns `200` with a not-found message and is **not** billed.
- **Transient errors are retried server-side and never charged.** Do add client-side retry with backoff on `429`/`503`.

## Credits & rate limits

- Costs are **per endpoint** and per successful call; simple reference lookups are cheap, richer resolutions cost more. **Exact per-route costs: `GET https://renidly.com/api/panel/credits/routes/costs/` (public, no key).** It lists only routes whose cost ≠ 1 — **any route not listed costs 1 credit**, and `credits_cost: 0` means free. (Batch/per-item endpoints show 0 base and bill per resolved item.) See `account-api.md`.
- **Cache is transparent and free.** Identical repeat requests (same inputs) are served from a short-lived cache at no cost and near-zero latency; a miss runs the real resolution and is billed. You do not choose cached vs live — it's automatic per call.
- **Rate limits are per-minute and tier-based.** Exceeding them returns `429` with your tier/limit in `errors`. Upgrade tier to raise the ceiling.

### Reading balance, tier & rate limit (Account & Credits)

Account routes live under `https://renidly.com/api/panel` and — unlike the data APIs — authenticate with the **`X-AUTHAPI-Key`** header (same key value, different header name) and **require a trailing slash** on `/k/` routes. Full detail in `account-api.md`.

- **"What's my balance?"** → `GET /api/panel/credits/balance/k/` with `X-AUTHAPI-Key`. Read `data.balance`.
- **"What's my tier / rate limit?"** → `GET /api/panel/credits/tier/k/` with `X-AUTHAPI-Key`. Read `data.current_tier.name`, `data.current_tier.limit_per_minute`, `data.balance`, and `data.next_tier.credits_needed`.
- **Enterprise workspace balance** → `GET /api/panel/credits/balance/k/enterprise/` with the **Enterprise** key.
- **The tier ladder, no key needed** → `GET /api/panel/user/sub/tiers/` (free, public).

To actually answer a live balance/tier question, call the endpoint with the caller's API key (e.g. from `RENIDLY_API_KEY`), sent as `X-AUTHAPI-Key`, trailing slash kept.

## Identifiers

**Data API — opaque, prefixed, stable public IDs.** Never enumerable, never changing:

| Prefix | Subject |
|--------|---------|
| `prsn_…` | person / professional record |
| `org_…`  | organization / company |
| `inst_…` | educational institution |
| `skl_…`  | skill |

Prefer these over public handles/slugs (which can change). Institutions are additionally addressable by `normalized_name`; skills by their `skl_` id.

**Live API — subject-specific tokens** (all opaque strings; treat as black boxes):
- People: `entityId` (stable) or a public `handle` (convenience; resolve once to `entityId`).
- Organizations: numeric `id` (stable) or public `slug` (resolve via resolve-slug).
- Opportunities: `opportunityEntityId`. Activities: `entityId`. Geographies: `geoEntityId`.

**Email API — the email address itself is the key**, plus opaque `job_id` (UUID) for batch jobs and opaque cursors for paging.

## Pagination

Three models appear across the platform — always read the specific endpoint:

- **Page-based** (`page`, `limit`) — Data API institutions/skills search, alumni; Email `datePosted`-style lists; response carries `has_more`.
- **Cursor / keyset** (opaque `cursor` → `next_cursor`) — Data API people/company search & employees; Email prospects & batch tracking; Live activity feeds. Omit the cursor for the first page; pass `next_cursor` back for the next; empty/absent means the end.
- **Offset-based** (`start`, `count`) — most Live API list/search endpoints. `start` is a zero-based offset; some endpoints constrain it (e.g. Live activity reactions require `start` to be a multiple of 10).

## Batch / async jobs (Data API + Email API)

Large workloads run as background jobs. Same three-step shape everywhere:

1. **Submit** — `POST …/batch/enrich` (Data) or `POST …/verify/batch` · `…/find/batch` (Email) with a list (up to 1000 items). Returns `202` + `{ "job_id": "…" }` immediately. Balance is pre-checked; nothing is charged at submit.
2. **Track** — `GET` the same path with `job_id` (+ an `after` cursor). Returns `status` (`processing` / `completed` / `error`), progress counters (`total`, `resolved`, `errors`), a `next_cursor`, and a page of results keyed to your inputs.
3. **Collect** — page through results as they resolve (each item billed on definitive result; not-founds returned in a dedicated list). Jobs are pollable until they expire.

Prefer batch over single calls whenever the work is a list (CRM/database hydration, imports, backfills, scheduled refreshes) rather than one lookup a user is waiting on.

## Which endpoint? (quick decision guide)

- **I have a stable ID or want consistent, queryable records at scale** → Data API (`data-api.md`).
- **I need the freshest snapshot of one subject, its recent activity, related signals, or a discovery search** → Live API (`live-api.md`).
- **I have an email / a name+domain / a profile URL and care about deliverability or the person behind an address** → Email API (`email-api.md`).
- **I have a whole list to process** → the batch endpoints on the Data or Email API.
- **I want my balance, tier, or live rate limit** → Account & Credits (`account-api.md`); the tier ladder and per-route costs need no key.
- **I want to know what an endpoint costs** → `GET /api/panel/credits/routes/costs/` (public); absent from the list ⇒ 1 credit.
- **I just want to check my key works without spending credits** → the relevant product's `health` probe.

Full per-endpoint parameters, response fields, enums, and edge cases live in the three reference files. Read them before answering specifics — do not reconstruct endpoints from memory.
