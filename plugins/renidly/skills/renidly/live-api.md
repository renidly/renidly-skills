# Renidly Live API — Reference

Base: `https://renidly.com/api/v2` · Auth: `X-renidly-apikey` header · Envelope, errors, credits: see `SKILL.md`.

The Live API resolves a single B2B professional subject — a person, organization, job opportunity, or professional activity — or runs a discovery search, returning the freshest available snapshot on demand. Every endpoint is `GET`. Successful results carry `message: "Data retrieved successfully"` (or a per-endpoint message). A resolved-but-empty result returns `200` with a not-found message and is **not** billed. Identifiers are opaque strings — treat them as black boxes.

## Identifiers
- **Person:** `entityId` (stable, preferred) or a public `handle` (convenience — resolve once, then reuse the `entityId`).
- **Organization:** numeric `id` (stable) or public `slug` (resolve via resolve-slug).
- **Opportunity:** `opportunityEntityId`. **Activity:** `entityId`. **Geography:** `geoEntityId`.
- Resolve handle/slug → id **once**, cache the id, reuse it for all subsequent calls.

## Endpoint index

| Group | Method & path | Purpose |
|---|---|---|
| Person | `GET /person/enrich` | Full profile for one professional |
| Person | `GET /person/resolve-handle` | Public handle → stable `entityId` |
| Person | `GET /person/employment-history` | Complete work history |
| Person | `GET /person/endorsements` | Recommendations written for the person |
| Person | `GET /person/lookalikes` | Similar professional profiles |
| Person | `GET /person/interests` | Entities the person follows |
| Activity | `GET /activity/feed` | A person's recent posts / activity stream |
| Activity | `GET /activity/details` | Full content + comments of one post |
| Activity | `GET /activity/reactions` | People who reacted to a post |
| Activity | `GET /activity/replies` | Comments/replies on a post |
| Activity | `GET /activity/replies/by-author` | Comments authored by a person |
| Opportunity | `GET /opportunity/details` | Full job-posting details |
| Opportunity | `GET /opportunity/similar` | Similar job postings |
| Opportunity | `GET /opportunity/related-views` | "People also viewed" postings |
| Opportunity | `GET /opportunity/hiring-team` | Hiring-team members for a posting |
| Opportunity | `GET /opportunity/by-person` | Postings created by a person |
| Organization | `GET /organization/enrich` | Full company profile |
| Organization | `GET /organization/headcount` | Employee-count distribution |
| Organization | `GET /organization/similar` | Similar companies / peers |
| Organization | `GET /organization/affiliated` | Affiliated / subsidiary pages |
| Organization | `GET /organization/resolve-slug` | Public slug → numeric `id` |
| Organization | `GET /organization/activities` | A company's recent posts |
| Organization | `GET /organization/opportunities` | Open postings by organization(s) |
| Discover | `GET /discover/people` | Search professionals by keyword + filters |
| Discover | `GET /discover/organizations` | Search companies by keyword + filters |
| Discover | `GET /discover/opportunities` | Search job postings by keyword + filters |

---

## Person
- **`GET /person/enrich`** — `entityId` **or** `handle` (at least one; `entityId` wins if both). Returns full profile: names, headline, follower/connection counts, flags (`isTopVoice`, `premium`, `influencer`, `openToWork`, `isHiring`), industry, images, and a structured `location`.
- **`GET /person/resolve-handle`** — `handle` (required). Returns the stable `entityId`. One-time resolution; reuse the id.
- **`GET /person/employment-history`** — `entityId` (required). Returns full work history: organization name/id/url/logo, title, description, location, parsed start/end, skills, and parallel-position groupings.
- **`GET /person/endorsements`** — `entityId` (required). Recommendations written for the person, each with author details and text. Use for trust signals / warm-intro paths.
- **`GET /person/lookalikes`** — `entityId` (required). Similar profiles — expand a shortlist from one ideal example.
- **`GET /person/interests`** — `entityId` (required). Entities the person follows (companies, groups, people, newsletters) with type + link. Use for personalization and audience insight.

## Activity
- **`GET /activity/feed`** — `entityId` (required, person). `cursor` (opaque, preferred) or `start` (offset). Returns `activities[]` (id, url, text, author, timestamps, engagement counts, media, repost content) + `nextCursor`.
- **`GET /activity/details`** — `entityId` (required, activity). Full post plus top-level comments and threaded replies.
- **`GET /activity/reactions`** — `entityId` (required, activity). `start` (offset, **must be a multiple of 10**; other values → `400`). Returns reactor profiles + reaction type, `totalReactions`, page info.
- **`GET /activity/replies`** — `entityId` (required, activity). `sortBy` (`relevance` default | `date_posted` = newest first). `count` (default 10). `start` (offset). Returns threaded comments.
- **`GET /activity/replies/by-author`** — `entityId` (required, person). `cursor` or `start`. Comments authored by that person across posts + `nextCursor`.

## Opportunity
- **`GET /opportunity/details`** — `opportunityEntityId` (required). Full posting: title, description, state, timestamps, functions, apply url, employment status, industries, plus nested organization and location.
- **`GET /opportunity/similar`** — `opportunityEntityId` (required). Similar postings (title, organization, location, salary range, posted date) + `total`.
- **`GET /opportunity/related-views`** — `opportunityEntityId` (required). "People also viewed" postings (behavioral relatedness) + `total`.
- **`GET /opportunity/hiring-team`** — `opportunityEntityId` (required). Hiring-team member profiles for the posting.
- **`GET /opportunity/by-person`** — `personEntityId` (required). `count` (1–25, default 10), `start`. Postings created by that person (e.g. a recruiter/founder's open roles).

## Organization
- **`GET /organization/enrich`** — `id` (required, numeric). Full company profile: identifiers, tagline/description, type, images, staff count + range, HQ + all locations, industries, specialties, website, founded, follower count, funding data when available, plus related/affiliated/parent pages.
- **`GET /organization/headcount`** — `id` (required). Employee-count total + distribution buckets (by department, seniority, location …). Use for sizing and structure signals.
- **`GET /organization/similar`** — `id` (required). Similar companies / peers (id, name, industry, followers, url) + count.
- **`GET /organization/affiliated`** — `id` (required). Affiliated/subsidiary/showcase pages + count. Maps the corporate family.
- **`GET /organization/resolve-slug`** — `slug` (required). Public slug → numeric `id`. One-time resolution; reuse the id.
- **`GET /organization/activities`** — `id` (required). `start` (offset). A company's recent posts (same shape as activity feed).
- **`GET /organization/opportunities`** — `organizationEntityIds` (required, comma-separated org ids). `start` (offset; page size fixed at 50). Open postings across those organizations + `total`, `hasMore`. Monitor hiring across target accounts.

## Discover (search)
Multi-value filters are **comma-separated**; values within one parameter are OR'd, different parameters are AND'd. All are offset-paginated (`start` 0–999, `count` 0–50).

- **`GET /discover/people`** — `keyword` (free text) and/or structured filters: `firstName`, `lastName`, `title`, `currentCompany` (org ids), `pastCompany` (org ids — alumni sourcing), `school` (institution ids), `industry` (ids), `geoEntityId` (location ids), `profileLanguage`, `serviceCategory`. `start`, `count` (default 20). Returns matching people + `total`, `hasMore`. Use for talent sourcing and account-based outreach.
- **`GET /discover/organizations`** — `keyword` (required). Filters: `headcountRange` (`1-10`, `11-50`, `51-200`, `201-500`, `501-1000`, `1001-5000`, `5001-10000`, `10001+`), `industry` (ids), `geoEntityId` (HQ location ids), `hasJobs` (`true`/`false`). `start`, `count` (default 25). Returns companies + `total`, `hasMore`.
- **`GET /discover/opportunities`** — `keyword` and/or rich filters: `sortBy` (`relevance` | `date_posted`), `datePosted` (`24h`|`1week`|`1month`), `experience` (`internship`,`entry_level`,`associate`,`mid_senior`,`director`,`executive`), `jobTypes` (`full_time`,`part_time`,`contract`,`temporary`,`internship`,`volunteer`,`other`), `workplaceTypes` (`onsite`,`remote`,`hybrid`), `salary` (min bucket `20k`…`100k`), `companies` (org ids), `industries`, `locations` (geo ids), `functions`, `titles`, `benefits`, `commitments`, `easyApply`, `verifiedJob`, `under10Applicants`, `fairChance` (each `true`/`false` where boolean). `start`, `count` (default 25). Returns postings + `total`.

---

## Live API notes & edge cases
- **Resolve once, reuse.** Convert a handle/slug to its stable id via `person/resolve-handle` or `organization/resolve-slug`, cache it, and pass the id to every other call. Handles/slugs can change; ids don't.
- **`start` on reactions must be a multiple of 10** (0, 10, 20…). Other list endpoints take any `start ≥ 0`, `count ≤ 50`.
- **Comma-separated filters** = OR within a parameter, AND across parameters (discovery endpoints).
- **`sortBy=date_posted`** returns newest first.
- **Not-found is `200` and free** (e.g. an unknown handle, a post with no reactions/comments) — branch on `success` and the message. Transient errors are retried and never charged.
- **Cursor vs offset:** `activity/feed` and `activity/replies/by-author` use opaque cursors (`nextCursor`); everything else uses `start`/`count`.
