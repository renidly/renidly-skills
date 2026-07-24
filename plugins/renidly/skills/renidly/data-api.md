# Renidly Data API — Reference

Base: `https://renidly.com/api/data/v1` · Auth: `X-renidly-apikey` header · Envelope, errors, credits: see `SKILL.md`.

The Data API serves a clean, deduplicated dataset of B2B professional records — people, organizations, institutions, skills, and job-change events — addressable by stable opaque IDs (`prsn_`, `org_`, `inst_`, `skl_`) or rich filters. List endpoints are cursor- or page-paginated. Single-record lookups that resolve nothing return `200` with `success:false` + a not-found `error_code` and are billed (a real lookup ran).

## Endpoint index

| Method & path | Purpose |
|---|---|
| `GET /health` | Liveness probe (no key, free) |
| `GET /people/profile` | One professional record by id or handle |
| `GET /people/search` | Filter professional records |
| `POST /people/batch/enrich` | Submit a bulk people-enrichment job |
| `GET /people/batch/enrich` | Track a people-enrichment job |
| `GET /companies/company` | One organization by id or slug |
| `GET /companies/search` | Filter organizations |
| `GET /companies/employees` | People at an organization |
| `POST /companies/batch/enrich` | Submit a bulk company-enrichment job |
| `GET /companies/batch/enrich` | Track a company-enrichment job |
| `GET /institutions/institution` | One institution by normalized name |
| `GET /institutions/search` | Search institutions by name |
| `GET /institutions/alumni` | Alumni / students of an institution |
| `GET /skills/skill` | One skill by `skl_` id |
| `GET /skills/search` | Search the skill catalog by name |
| `GET /job-changes/search` | Recent join / leave / title-change events |

---

## People

### GET /people/profile
One complete professional record.
- **Params (one required):** `id` (opaque `prsn_…`, preferred/stable) **or** `handle` (public profile handle; may change).
- **Returns:** full record — identifiers, name fields, headline, summary, current & historical positions (with organization id/name/slug/url), education, skills, certifications, languages, projects, publications, geo, follower & connection counts, flags (`is_creator`, `is_premium`, `is_open_to_work`, `is_hiring`), localized fields.
- **Not found:** `200`, `success:false`, `error_code` `1010` (billed).

### GET /people/search
Filter professional records; cursor-paginated. Combine filters with AND.
- **Text filters:** `first_name`, `last_name`, `headline`, `summary`, `geo_city`, `title` (min 3 chars each where free-text).
- **Keyword/exact:** `geo_country_code`, `primary_language`, `is_creator`, `is_premium`, `organization_slugs` (comma-sep), `institution_ids` (comma-sep `inst_`), `certifications`, `certification_authority`, `speaks_language`.
- **Skills:** `skills` (comma-sep normalized names), `skills_match` (`any` default | `all`), `skill_count_min`, `skill_count_max`.
- **Scope:** `current_only` (bool — restrict title/company matches to current positions).
- **Career/tenure:** `last_change_within_days`, `last_change_type` (`joined`|`left`|`title_change`), `tenure_min_years`, `tenure_max_years`, `company_count_min`, `company_count_max`, `current_company_count_min`, `is_boomerang`.
- **Education:** `education_level`, `degree`, `field_of_study`.
- **Paging:** `cursor` (opaque; omit for first page), `limit` (1–50, default 20). Response carries `has_more` + `next_cursor`.

### POST /people/batch/enrich  ·  GET /people/batch/enrich
Async bulk enrichment of up to 1000 people. See `SKILL.md` → “Batch / async jobs”.
- **Submit body:** a list of items (person ids and/or handles) plus optional `enrich_live` (bool) to force the freshest per-item resolution rather than the standard dataset read. Returns `202` + `job_id`.
- **Track params:** `job_id` (required), `after` (result cursor, start 0). Returns `status`, `total`, `resolved`, `errors`, `next_cursor`, a `not_found` list, and `profiles` keyed to the exact input you sent.

---

## Organizations (Companies)

### GET /companies/company
One organization record.
- **Params (one required):** `id` (opaque `org_…`, preferred) **or** `slug` (public slug; may change).
- **Returns:** firmographics — name, slug, url, logo, description/tagline, industry, employee count & range, HQ + locations, founded date, specialties, follower count, funding signals when available.
- **Not found:** `200`, `success:false`, `error_code` `1020` (billed).

### GET /companies/search
Filter organizations; cursor-paginated.
- **Params:** `name` **or** `website` required (min 3 chars). Plus `staff_count_min`/`staff_count_max`, `follower_count_min`/`follower_count_max`, `industries`, `industries_v2`, `hq_city`, `hq_country_code`, `founded`, `cursor`, `limit` (1–50, default 20).

### GET /companies/employees
People who work or worked at an organization; cursor-paginated. Filters combine with AND.
- **Params:** `slug` (required — the organization's public slug). `current_only` (bool). `title` (partial-match job-title filter, min 3 chars — e.g. `software engineer`; combine with `current_only=true` to target a current role). `geo_country_code`, `geo_city` (min 3). `start_year` (1900–current) + optional `start_month` (1–12) to match people who started in that period. `sort` (`newest` | `oldest` | `recently_left` — use `recently_left` with `current_only=false`). `cursor`, `limit` (1–50, default 20).

### POST /companies/batch/enrich  ·  GET /companies/batch/enrich
Async bulk enrichment of up to 1000 organizations. Same shape as people batch; Track returns results under `companies`, keyed to your inputs, plus a `not_found` list.

---

## Institutions

### GET /institutions/institution
One institution.
- **Params:** `normalized_name` (required — lowercase, hyphenated; discover via institutions search). Returns name, normalized name, url, opaque `inst_` id.

### GET /institutions/search
Partial-match search on institution name; page-paginated.
- **Params:** `name` (required, min 3). `page` (≥1, default 1), `limit` (1–50, default 20). Use it to discover an institution's `normalized_name` (for alumni) or its `inst_` id (for the people-search `institution_ids` filter).

### GET /institutions/alumni
Alumni and current students of an institution; page-paginated. Each result is a professional record plus the education record linking them to the institution.
- **Params:** `normalized_name` (required). `current_only` (bool). `degree`, `field_of_study`, `geo_city` (min 3 each). `geo_country_code`. Year ranges `start_year_min`/`start_year_max`/`end_year_min`/`end_year_max` (1900 … current+10; `max ≥ min`). `sort` (`newest` | `oldest` | `recently_graduated`). `page`, `limit`.

---

## Skills

### GET /skills/skill
One skill by opaque `skl_` id (from skills search).
- **Params:** `id` (required, `skl_…`). Returns `id`, `name`, `normalized_name`. Not found → `200`, `error_code` `1030` (billed).

### GET /skills/search
Partial-match search on skill name; page-paginated. Lightweight.
- **Params:** `name` (required, min 3). `page`, `limit` (1–50, default 20). Use it to find a skill's `skl_` id or its `normalized_name` for the people-search `skills` filter.

---

## Job changes

### GET /job-changes/search
Recent professional job-change events — people who joined, left, or changed titles at organizations; page-paginated.
- **Params:** `event_type` (`joined` | `left` | `title_change`). `organization_ids` (comma-sep `org_…`). `geo_city`, `geo_country_code`. `title` (partial match). `days_ago` (1–365 recency window; omit for none). `page`, `limit`.
- **Use for:** trigger-based prospecting, territory monitoring, relationship-change alerts.

---

## Data API notes & edge cases

- **Prefer stable IDs over handles/slugs.** Handles and slugs can change; `prsn_`/`org_`/`inst_`/`skl_` never do.
- **Single lookups are billable even when empty** (a real lookup ran) — this is why not-found is `200`, not `404`. Batch not-founds come back in the `not_found` list and are billed per definitive result.
- **`enrich_live` on batch** forces the freshest per-item resolution instead of the standard dataset read; expect it to cost like a live resolution and take a little longer per item.
- **Cursors and IDs are opaque** — pass them back verbatim; never build or mutate them.
- **`current_only`** narrows title/company matching to a person's current positions across the people-search, company-employees, and related surfaces.
