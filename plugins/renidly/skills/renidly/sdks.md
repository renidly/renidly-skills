# Renidly SDKs — Python & Node (use these first)

Renidly ships **official, first-party SDKs** that wrap every endpoint in this skill. When you build an integration, **use the SDK — do not hand-roll HTTP calls.** The SDKs give you auth, retries, pagination, batch jobs, typed errors, and rate limiting for free, and they stay correct as the API evolves.

| Language | Package | Install | Import |
|---|---|---|---|
| **Python** (3.9+) | [`renidly`](https://pypi.org/project/renidly/) | `pip install renidly` | `from renidly import Renidly` |
| **Node / TypeScript** (18+) | [`renidly`](https://www.npmjs.com/package/renidly) | `npm install renidly` | `import { Renidly } from "renidly"` |

## Language policy (follow exactly)

1. **Building in Python or Node/TS/JS?** Use the official `renidly` package. Never write raw `requests`/`fetch`/`axios` calls against the API when the SDK covers it.
2. **Building in another language (Go, Ruby, PHP, Java, …)?** There is no official SDK yet. **Warn the user**: *"Renidly has official Python and Node SDKs; if you can use one of those, it's strongly recommended for reliability and support. Otherwise I'll call the HTTP API directly."* Then — only if they confirm — implement against the raw HTTP API using the reference files in this skill (`data-api.md`, `live-api.md`, `email-api.md`, `account-api.md`). **Never enforce** a language; just recommend.
3. **Same data constraints as the rest of this skill.** Renidly is a B2B professional-data platform. Describe only what methods accept and return — never how the data is produced or sourced.

## NEVER guess — verify against the real source

The SDKs are the source of truth. **Before you write code that calls an SDK method, confirm the method exists and its exact signature.** Do not reconstruct an API from memory, and **never ship code you have not verified.**

Authoritative sources, in order:

- **Raw READMEs** (fetch these directly):
  - Python → `https://raw.githubusercontent.com/renidly/renidly-python/main/README.md`
  - Node → `https://raw.githubusercontent.com/renidly/renidly-node/main/README.md`
- **Source code** (read the resource files to confirm a method name/params/return):
  - Python → `https://github.com/renidly/renidly-python/tree/main/src/renidly` (see `resources/`)
  - Node → `https://github.com/renidly/renidly-node/tree/main/src` (see `resources/`)
- **Official docs** → `https://renidly.com/docs`
- **Live per-route costs** → `GET https://renidly.com/api/panel/credits/routes/costs/` (public, no key)

**Verify-before-you-ship checklist:**
1. Fetch the raw README for the target language.
2. If the method or a parameter isn't 100% clear from the README, open the matching source file under `resources/` and read the signature and docstring/JSDoc.
3. Only then write code. If you still can't confirm something, say so and use `raw_request` / `rawRequest` (the escape hatch) or ask — **do not invent a method, parameter, or field.**

---

## Quickstart

**Python**
```python
from renidly import Renidly

renidly = Renidly("rnd-...")            # or Renidly() + RENIDLY_API_KEY env

person  = renidly.data.people.retrieve(handle="ryanroslansky")   # -> record or None
company = renidly.data.companies.retrieve(slug="stripe")
email   = renidly.emails.verify("sundar@google.com")
print(person.headline, company.name, email.deliverable)
```

**Node / TypeScript** — every call is `async`.
```ts
import { Renidly } from "renidly";

const renidly = new Renidly("rnd-...");  // or new Renidly() + RENIDLY_API_KEY env

const person  = await renidly.data.people.retrieve({ handle: "ryanroslansky" }); // -> record or null
const company = await renidly.data.companies.retrieve({ slug: "stripe" });
const email   = await renidly.emails.verify("sundar@google.com");
console.log(person.headline, company.name, email.deliverable);
```

Shape is identical in both: `renidly.<product>.<resource>.<action>(...)`. **Python methods are `snake_case`; Node methods are `camelCase`.** Data-API filter params keep the API's `snake_case` names in both languages; Live-API filter params keep the API's `camelCase` names in both.

## Authentication & configuration

The key is a positional arg, a config field, or the `RENIDLY_API_KEY` env var.

- **Python:** options live on a `RenidlyConfig` object passed as `config=`:
  ```python
  from renidly import Renidly, RenidlyConfig
  renidly = Renidly("rnd-...", config=RenidlyConfig(timeout=30, max_retries=3, auto_rate_limit=True))
  ```
- **Node:** options are a **plain object** (the 2nd arg). `RenidlyConfig` is a **TypeScript type only** — you don't instantiate it, you pass an object literal:
  ```ts
  const renidly = new Renidly("rnd-...", { timeout: 30_000, maxRetries: 3, autoRateLimit: true });
  ```

| Purpose | Python (`RenidlyConfig`) | Node (config object) | Default |
|---|---|---|---|
| Timeout | `timeout` (seconds) | `timeout` (**ms**) | 30s / 30000ms |
| Auto-retries | `max_retries` | `maxRetries` | 2 |
| Backoff base | `backoff_factor` (s) | `backoffFactor` (**ms**) | 0.5 / 500 |
| API host | `base_url` | `baseUrl` | `https://renidly.com` |
| Extra headers | `default_headers` | `defaultHeaders` | `{}` |
| Return `data` vs full envelope | `unwrap_data_obj` | `unwrapData` | `true` |
| Empty lookup → error vs null | `raise_on_not_found` | `throwOnNotFound` | `false` |
| Map failures to typed errors | `raise_on_api_error` | `throwOnApiError` | `true` |
| Client-side rate limiting | `auto_rate_limit` | `autoRateLimit` | `false` |
| Fixed per-minute limit (enterprise) | `rate_limit_per_minute` | `rateLimitPerMinute` | — |
| Target fraction of limit | `rate_limit_safety` | `rateLimitSafety` | 1.0 |

**Per-request override** (multi-tenant, custom timeout): pass `options` as the last argument to any method — `options={"api_key": "...", "timeout": 5}` (Python) / `{ apiKey: "...", timeout: 5000 }` (Node).

## Full method map (verified against source — confirm before use)

Namespaces: `renidly.data`, `renidly.live`, `renidly.emails`, `renidly.account`.

### `data` — clean, queryable records
| Method | Python | Node |
|---|---|---|
| Person by id/handle | `data.people.retrieve(id=…, handle=…)` | `data.people.retrieve({ id, handle })` |
| People search | `data.people.search(**filters)` | `data.people.search({ …filters })` |
| People bulk enrich | `data.people.enrich_batch(ids=…, handles=…, live=False)` | `data.people.enrichBatch({ ids, handles, live })` |
| Company by id/slug | `data.companies.retrieve(id=…, slug=…)` | `data.companies.retrieve({ id, slug })` |
| Company search | `data.companies.search(**filters)` | `data.companies.search({ …filters })` |
| Company employees | `data.companies.employees(slug, **filters)` | `data.companies.employees(slug, { …filters })` |
| Company bulk enrich | `data.companies.enrich_batch(ids=…, handles=…, live=False)` | `data.companies.enrichBatch({ ids, handles, live })` |
| Institution by name | `data.institutions.retrieve(normalized_name)` | `data.institutions.retrieve(normalizedName)` |
| Institution search | `data.institutions.search(name, page=…, limit=…)` | `data.institutions.search(name, { page, limit })` |
| Institution alumni | `data.institutions.alumni(normalized_name, **filters)` | `data.institutions.alumni(normalizedName, { …filters })` |
| Skill by id | `data.skills.retrieve(id)` | `data.skills.retrieve(id)` |
| Skill search | `data.skills.search(name, page=…, limit=…)` | `data.skills.search(name, { page, limit })` |
| Job changes search | `data.job_changes.search(**filters)` | `data.jobChanges.search({ …filters })` |

### `live` — freshest snapshot on demand
| Method | Python | Node |
|---|---|---|
| Person enrich | `live.people.enrich(entity_id=…, handle=…)` | `live.people.enrich({ entityId, handle })` |
| Resolve handle → entityId | `live.people.resolve_handle(handle)` | `live.people.resolveHandle(handle)` |
| Employment history | `live.people.employment_history(entity_id)` | `live.people.employmentHistory(entityId)` |
| Endorsements / lookalikes / interests | `live.people.endorsements|lookalikes|interests(entity_id)` | `live.people.endorsements|lookalikes|interests(entityId)` |
| Activity feed / details | `live.activities.feed(entity_id, …)` · `live.activities.retrieve(entity_id)` | `live.activities.feed(entityId, …)` · `live.activities.retrieve(entityId)` |
| Reactions / replies / replies_by_author | `live.activities.reactions|replies|replies_by_author(entity_id, …)` | `live.activities.reactions|replies|repliesByAuthor(entityId, …)` |
| Opportunity details | `live.opportunities.retrieve(opportunity_entity_id)` | `live.opportunities.retrieve(opportunityEntityId)` |
| Opportunity similar / related_views / hiring_team | `live.opportunities.similar|related_views|hiring_team(opportunity_entity_id)` | `live.opportunities.similar|relatedViews|hiringTeam(opportunityEntityId)` |
| Opportunities by person | `live.opportunities.by_person(person_entity_id, …)` | `live.opportunities.byPerson(personEntityId, …)` |
| Org enrich / headcount | `live.organizations.enrich(id)` · `live.organizations.headcount(id)` | `live.organizations.enrich(id)` · `live.organizations.headcount(id)` |
| Org similar / affiliated / activities | `live.organizations.similar|affiliated|activities(id, …)` | `live.organizations.similar|affiliated|activities(id, …)` |
| Resolve slug → id | `live.organizations.resolve_slug(slug)` | `live.organizations.resolveSlug(slug)` |
| Org opportunities | `live.organizations.opportunities(organization_entity_ids, …)` | `live.organizations.opportunities(organizationEntityIds, …)` |
| Discover people / organizations / opportunities | `live.discover.people|organizations|opportunities(**filters)` | `live.discover.people|organizations|opportunities({ …filters })` |

### `emails` — verify, find, resolve
| Method | Python | Node |
|---|---|---|
| Verify | `emails.verify(email)` | `emails.verify(email)` |
| Find | `emails.find(first_name=…, last_name=…, domain=…)` | `emails.find({ firstName, lastName, domain })` |
| Find by profile URL | `emails.find_by_url(url)` | `emails.findByUrl(url)` |
| Reverse | `emails.reverse(email)` | `emails.reverse(email)` |
| Prospects for a domain | `emails.prospects(domain, kind, cursor=…)` | `emails.prospects(domain, kind, { cursor })` |
| Bulk verify | `emails.verify_batch([…])` | `emails.verifyBatch([…])` |
| Bulk find | `emails.find_batch([{…}])` | `emails.findBatch([{…}])` |

### `account` — balance, tier, pricing
| Method | Python | Node |
|---|---|---|
| Balance | `account.balance()` | `account.balance()` |
| Tier & rate limit | `account.tier()` | `account.tier()` |
| Enterprise balance | `account.enterprise_balance()` | `account.enterpriseBalance()` |
| Public tier ladder | `account.tiers()` | `account.tiers()` |
| Per-route costs | `account.route_costs()` | `account.routeCosts()` |

> The exact filter parameters for every `search`/`discover`/`employees`/`alumni` method are typed in each SDK (`types/params`). **Read the raw README or the source for the full, current list — do not guess parameter names.**

## Pagination

Every `search`/list method is paginated and walks all pages lazily.

**Python** — the call returns a list you can index/iterate for the current page, plus `.auto_paging_iter()` to walk every page:
```python
page = renidly.data.people.search(title="cto", limit=25)
len(page); page[0]; page.has_more          # current page
for p in renidly.data.people.search(title="cto").auto_paging_iter():   # every page
    print(p.headline)
# Async client: `async for p in ....auto_paging_iter():`
```

**Node** — the call returns a **hybrid** that is *both* awaitable *and* async-iterable:
```ts
const page = await renidly.data.people.search({ title: "cto", limit: 25 }); // one page
page.length; page.data[0]; page.hasMore;
for await (const p of renidly.data.people.search({ title: "cto" })) {        // every page
  console.log(p.headline);                                                    // no extra await
}
// Explicit generator: (await renidly.data.people.search({title:"cto"})).autoPagingIter()
```

Each page is a separate billed request — read its cost/balance off `page.meta.credit_consumed` / `page.meta.remaining_balance` (Python) · `page.meta.creditConsumed` / `page.meta.remainingBalance` (Node). See [Response objects & credit metadata](#response-objects--credit-metadata).

## Batch / async jobs

Submit returns a job handle immediately; then wait (blocking) or stream. Up to 1000 items per job.

Methods: `data.people.enrich_batch|enrichBatch`, `data.companies.enrich_batch|enrichBatch`, `emails.verify_batch|verifyBatch`, `emails.find_batch|findBatch`.

**Python**
```python
job = renidly.data.people.enrich_batch(handles=["ryanroslansky", "williamhgates"], live=True)
result = job.wait(on_progress=lambda n: print("resolved", n))
print(result.status, result.resolved, "/", result.total, result.not_found)
for row in result.results:
    print(row.matched_input, "->", row.headline)
# or stream:  for row in renidly.emails.verify_batch([...]).stream(): ...
```

**Node**
```ts
const job = await renidly.data.people.enrichBatch({ handles: ["ryanroslansky", "williamhgates"], live: true });
const result = await job.wait({ onProgress: (n) => console.log("resolved", n) });
console.log(result.status, result.resolved, "/", result.total, result.notFound);
for (const row of result.results) console.log(row.matched_input, "->", row.headline);
// or stream:  for await (const row of (await renidly.emails.verifyBatch([...])).stream()) { ... }
```

Each resolved row carries `matched_input` linking it back to what you submitted. Not-found inputs come back in `not_found` (Python) / `notFound` (Node).

## Errors

Every failure raises (Python) / rejects (Node) with a specific subclass of `RenidlyError`, and the message includes the field-level detail.

| Error class (both) | When |
|---|---|
| `AuthenticationError` | missing / invalid key |
| `PermissionDeniedError` | key valid but not allowed here (premium-gated / opted-out) |
| `InvalidRequestError` | bad input — read `field_errors` / `fieldErrors` |
| `InsufficientCreditsError` | not enough credits |
| `NotFoundError` | batch job not found / expired |
| `RateLimitError` | per-minute limit hit — `tier`, `limit`, `retry_after`/`retryAfter` |
| `ServiceUnavailableError` | transient — retry shortly |
| `APIConnectionError` | network / timeout |

```python
from renidly import InvalidRequestError, RateLimitError, RenidlyError
try:
    renidly.emails.find(first_name="A", last_name="B", domain="bad")
except InvalidRequestError as e:
    print(e.message, e.field_errors)
except RenidlyError as e:
    print(e.status_code, e.error_code, e.message)
```
```ts
import { InvalidRequestError, RateLimitError, RenidlyError } from "renidly";
try {
  await renidly.emails.find({ firstName: "A", lastName: "B", domain: "bad" });
} catch (e) {
  if (e instanceof InvalidRequestError) console.log(e.serverMessage, e.fieldErrors);
  else if (e instanceof RenidlyError) console.log(e.status, e.errorCode, e.serverMessage);
}
```

**Not-found single lookups:** `retrieve(...)` returns `None`/`null` by default (not an error). Set `raise_on_not_found`/`throwOnNotFound` to raise instead.

## Response objects & credit metadata

Responses are dynamic and drill-able — access any field (nested included) directly; no schema classes needed. Prefer the raw envelope? Set `unwrap_data_obj:false` / `unwrapData:false`.

Every result also carries a **`.meta`** object with the HTTP + billing metadata for the call that produced it. It's kept off the response data, so `person.headline` is your data and `person.meta.credit_consumed` is billing info (SDK **≥ 0.2.0**):

| `.meta` field | Python | Node |
|---|---|---|
| **Credits charged for THIS request** | `meta.credit_consumed` | `meta.creditConsumed` |
| **Balance remaining after the charge** | `meta.remaining_balance` | `meta.remainingBalance` |
| HTTP status | `meta.status_code` | `meta.statusCode` |
| Request id | `meta.request_id` | `meta.requestId` |
| Raw response headers | `meta.headers` | `meta.headers` |
| Parsed body envelope | `meta.body` | `meta.body` |
| Raw response text | `meta.raw_body` | `meta.rawBody` |
| Underlying HTTP response object | `meta.raw_http` (`httpx.Response`) | `meta.rawHttp` (fetch `Response`) |

```python
p = renidly.data.people.retrieve(handle="ryanroslansky")
print(p.meta.credit_consumed, p.meta.remaining_balance)   # e.g. 1.0 19813.0
```
```ts
const p = await renidly.data.people.retrieve({ handle: "ryanroslansky" });
console.log(p.meta.creditConsumed, p.meta.remainingBalance); // e.g. 1 19813
```

- `.meta` is on **every** result — single objects, list pages, and each item in a page. In Node it's **non-enumerable** (won't show in `JSON.stringify`/`Object.keys`).
- `credit_consumed`/`remaining_balance` are `None`/`undefined` when a route isn't credit-billed (e.g. `account.*`) or wasn't charged (errors, **cached hits**, zero-result billing). Result-billed endpoints report the real dynamic amount — e.g. `emails.prospects(...)` returning 18 emails → `meta.credit_consumed == 18`. Cached responses are served free → `0`.
- During `auto_paging_iter()` / `for await`, each **page** is a separate billed request, so each item reflects **its own page's** `.meta`.
- `.last_response` / `.lastResponse` remains as a **deprecated alias** for `.meta`.

## Automatic rate limiting

Turn on `auto_rate_limit`/`autoRateLimit` and the SDK stays under your per-minute limit with a sliding 60-second window. Regular keys read the limit from your tier automatically; **enterprise keys require `rate_limit_per_minute`/`rateLimitPerMinute`.**

## Escape hatch

For an endpoint not yet surfaced as a method, call it directly instead of guessing:
- Python: `renidly.raw_request("GET", "/people/search", service="data", params={...})`
- Node: `await renidly.rawRequest("GET", "/people/search", { service: "data", params: {...} })`

---

**Reminder:** the tables above are a map, not a contract. The packages evolve — for the current, exact surface always fetch the **raw README** and read the **source** before writing code, and never ship a method or parameter you haven't confirmed.
