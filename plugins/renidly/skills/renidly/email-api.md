# Renidly Email API — Reference

Base: `https://renidly.com/api/emails/v1` · Auth: `X-renidly-apikey` header · Envelope, errors, credits: see `SKILL.md`.

The Email API verifies deliverability, discovers work emails, reverse-resolves the person and company behind a business email, and lists known contacts for a company domain. It includes async batch jobs for verify and find. Repeat calls with identical inputs are served from cache (free). Persons/addresses can opt out; opted-out lookups return `403` and are not charged.

## Endpoint index

| Method & path | Purpose |
|---|---|
| `GET /health` | Liveness probe (no key, free) |
| `GET /verify` | Check whether an address can receive mail |
| `GET /find` | Find a work email from name + company domain |
| `GET /find/linkedin` | Find a work email from a professional profile URL |
| `GET /reverse` | Resolve the person + company behind a business email |
| `GET /prospects` | Page known emails for a company domain |
| `POST /verify/batch` | Submit a bulk verify job (≤1000) |
| `GET /verify/batch` | Track a bulk verify job |
| `POST /find/batch` | Submit a bulk find job (≤1000) |
| `GET /find/batch` | Track a bulk find job |

---

## GET /verify
Check whether an email address can receive mail and flag risky conditions.
- **Params:** `email` (required, valid syntax).
- **Returns:** `email`, `deliverable` (bool), `reason`, `catch_all` (bool), `mx_hosts` (array).
- **`reason` enum:** `mailbox_accepts`, `mailbox_rejected`, `catch_all`, `risky`, `invalid_syntax`, `disposable`, `relay`, `no_mx`.
- **Billed** per address checked (any verdict). Cached.

## GET /find
Discover a person's work email from their name and company domain.
- **Params:** `first_name` (required), `last_name` (required), `domain` (required — bare hostname, e.g. `acme.com`; no scheme, no `@`, must contain a dot).
- **Returns:** `email` (empty if not found), `found` (bool), `catch_all` (bool), `confidence` (`high` | `low`).
- **Billed** per lookup (found or not). Cached.

## GET /find/linkedin
Identify a person and their current company from a professional profile URL, then find their work email at that company.
- **Params:** `url` (required — a professional profile URL or its bare public slug).
- **Returns:** `email`, `found`, `catch_all`, `confidence`, plus resolved `first_name`, `last_name`, `domain`, `company`, and the normalized profile `url`.
- **Billed** per lookup (found or not). Cached.

## GET /reverse
Resolve the person and company behind a **business** email address.
- **Params:** `email` (required). **Professional working mailboxes only** — public providers (gmail/outlook/…), role accounts (`info@`, `support@`, …), disposable, relay, and invalid addresses are rejected with `422` before any work runs.
- **Returns:** `email`, `found` (bool), `confidence` (`high` | `medium` | `low` | `none`), `person` (object or null — full name, headline, summary, title, profile url, picture, location, industry, follower count, `is_premium`, `is_open_to_work`, `experience[]`, `education[]`), and `current_company` (object or null — name, slug, website, profile url, logo, description, industry, size, staff count, follower count, location, specialities).
- **Billing:** full cost when a person **or** company is returned; partial (3 credits) when discovery ran but found nothing; free when the address is unsupported. Cached.

## GET /prospects
Page emails already known for a company domain.
- **Params:** `domain` (required, bare hostname). `kind` (required — `full` = all known emails | `verified_only` = deliverable only). `cursor` (opaque; omit for first page, pass `next_cursor` next).
- **Returns:** `domain`, `kind`, `count` (≤20), `next_cursor` (empty on last page), `prospects[]` (`email`, `first_name`, `last_name`).
- **Billing:** per email returned — 1 credit each for `full`, 2 credits each for `verified_only`; an empty page is free. Fails closed (`402`) if the balance can't cover the page. Cached per page.

---

## Batch jobs
Async, up to **1000** items per job; results pollable for **6 hours**. Submit pre-checks balance (rejects `402`, charges nothing) and returns `202` + `{ job_id }`. Items are processed in the background and billed per definitive result at the matching single-call cost; transient failures are retried and not charged. If the balance hits zero mid-job, the job stops with `status: error` and already-resolved results remain retrievable. Tracking is always live (never cached) and free.

### POST /verify/batch  ·  GET /verify/batch
- **Submit body:** `{ "emails": [ … ] }` — 1–1000 addresses; duplicates/blanks dropped.
- **Track params:** `job_id` (required), `after` (result cursor, start 0). Returns `status` (`processing`|`completed`|`error`), `error`, `total`, `resolved`, `errors`, `next_cursor`, and `results` — an `email → { deliverable, reason, catch_all }` map (≤100 per page). Page by passing `next_cursor` back as `after` until it reaches `resolved`.

### POST /find/batch  ·  GET /find/batch
- **Submit body:** `{ "queries": [ { "first_name", "last_name", "domain" }, … ] }` — 1–1000; each field required; duplicates/incomplete dropped.
- **Track params:** `job_id` (required), `after` (start 0). Returns the same job envelope; `results` is an array of `{ query, found, email }` (≤100 per page).

---

## Email API notes & edge cases
- **`domain` must be a bare hostname** (`acme.com`) — not `https://acme.com` or `user@acme.com`, and must contain a dot.
- **Reverse is business-email-only** — public/role/disposable/relay/invalid → `422` (free, no work done). Everything else returns `200`; branch on `found`/`confidence`.
- **Catch-all domains** are flagged `catch_all:true`; verify may return `risky` when a specific mailbox can't be confirmed, and find skips unresolvable catch-all domains rather than spend budget.
- **Opt-out:** find/find-by-url/reverse return `403` (not charged) for opted-out people or addresses.
- **Batch cursor** (`after`/`next_cursor`) is an integer offset into resolved results; poll until `next_cursor == resolved`. A verify job id polled on the find track endpoint (or vice-versa) returns `404`.
- **Wrong-endpoint / expired job** → `404` (`error_code` `1090`). Submit `402` when balance can't cover the unique item count (nothing charged).
