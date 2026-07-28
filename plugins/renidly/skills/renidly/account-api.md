# Renidly Account & Credits — Reference

Base: `https://renidly.com/api/panel` · Read your tier, per-minute rate limit, and credit balance directly with your key — no dashboard login or cookie.

## Two important differences from the data APIs

1. **Different auth header.** Account routes authenticate with **`X-AUTHAPI-Key`** (not `X-renidly-apikey`). It's the **same key value** — just a different header name on these routes.
   ```
   X-AUTHAPI-Key: <your-api-key>
   ```
2. **Trailing slash is required.** The authenticated routes end in `/k/` and must include the final slash. `…/credits/balance/k` (no slash) returns `Invalid session`; `…/credits/balance/k/` works.

Envelope is the shared shape (`success`, `message`, `errors`, `data`).

## Endpoints

| Method & path | Key? | Returns |
|---|---|---|
| `GET /api/panel/user/sub/tiers/` | **No key** (free, public) | The full tier ladder |
| `GET /api/panel/credits/routes/costs/` | **No key** (free, public) | Per-route credit costs (only routes ≠ 1 credit) |
| `GET /api/panel/credits/tier/k/` | `X-AUTHAPI-Key` | Current tier + rate limit + balance + neighbouring tiers |
| `GET /api/panel/credits/balance/k/` | `X-AUTHAPI-Key` | Just the current credit balance |
| `GET /api/panel/credits/balance/k/enterprise/` | Enterprise key | Balance for an Enterprise workspace |

### GET /api/panel/user/sub/tiers/  — public tier catalog (no key)
Lists every tier. Free, unauthenticated. Use to show pricing/limits or map a balance to a tier without a key.
```
curl https://renidly.com/api/panel/user/sub/tiers/
```
`data.results[]` each: `id`, `name`, `min_credits`, `max_credits` (null on the top tier), `limit_per_minute`, `credits_per_dollar`.

Current ladder (verified):

| Tier | Credits range | Req/min | Credits per $ |
|---|---|---|---|
| Testing | 0 – 99 | 7 | 100 |
| Hobby | 100 – 9,999 | 30 | 120 |
| Developer | 10,000 – 29,999 | 70 | 185 |
| Startup | 30,000 – 59,999 | 90 | 210 |
| Growth | 60,000 – 99,999 | 150 | 270 |
| Scale | 100,000 – 299,999 | 250 | 345 |
| Business | 300,000 – 599,999 | 350 | 355 |
| Ultra | 600,000 – 999,999 | 450 | 375 |
| Strategic Partner | 1,000,000+ | 550 | 400 |

### GET /api/panel/credits/routes/costs/  — per-route costs (no key)
Lists the credit cost of each endpoint. **Only routes whose cost differs from 1 are returned** — **any route not in the list costs 1 credit**, and `credits_cost: 0` means free. Costs can change, so fetch it live rather than hardcoding.
```
curl https://renidly.com/api/panel/credits/routes/costs/
```
`data`: `count`, and `routes[]` each with `route` (path), `title`, `description`, `credits_cost` (integer).

- To answer "how much does endpoint X cost?": fetch this list, match `route` to X's path — if present, the cost is `credits_cost` (0 = free); if absent, it's **1 credit**.
- **Base per-call cost only.** Some endpoints bill *per resolved item* on top of/instead of a base: batch submits and per-item endpoints show `0` here and charge as items resolve (see `data-api.md` / `email-api.md`), and `prospects` bills per email returned.
- Snapshot of notable costs (illustrative — verify live): `emails/verify` = 2, `emails/find` = 3, `emails/find/linkedin` = 5, `emails/reverse` = 5, `organization/enrich` = 2, `discover/opportunities` = 2; health probes, batch submits, and skills lookups = 0 (free).

### GET /api/panel/credits/tier/k/  — my tier + rate limit (authed)
Call this to know your **rate limit at runtime**.
```
curl https://renidly.com/api/panel/credits/tier/k/ -H "X-AUTHAPI-Key: $RENIDLY_API_KEY"
```
`data`:
- `balance` (number) — current credit balance.
- `credits_per_dollar` (int) — credits one USD buys.
- `current_tier` (object) — `name`, `min_credits`, `max_credits`, `limit_per_minute` (feed this to your rate limiter).
- `next_tier` (object | null) — same shape **plus** `credits_needed` (credits to reach it); `null` on the top tier.
- `previous_tier` (object | null) — same shape; `null` on the lowest tier.

### GET /api/panel/credits/balance/k/  — just my balance (authed)
Lightweight; use for low-balance alerts / "should I top up".
```
curl https://renidly.com/api/panel/credits/balance/k/ -H "X-AUTHAPI-Key: $RENIDLY_API_KEY"
```
`data`: `{ "balance": <number> }`.

### GET /api/panel/credits/balance/k/enterprise/  — Enterprise balance
Same as Balance but for an Enterprise workspace; authenticate with the **Enterprise** key (a Personal key returns `Invalid API key` here).
```
curl https://renidly.com/api/panel/credits/balance/k/enterprise/ -H "X-AUTHAPI-Key: $RENIDLY_ENTERPRISE_KEY"
```

## Answering "what's my balance / tier / rate limit"
1. Use the caller's Renidly API key (env var like `RENIDLY_API_KEY`, or ask for it). For balance-only, call `…/credits/balance/k/`; for tier + rate limit, call `…/credits/tier/k/`. Enterprise workspace → the `…/enterprise/` route with the Enterprise key.
2. Send it as `X-AUTHAPI-Key` and keep the trailing slash.
3. Report `data.balance`, `data.current_tier.name`, and `data.current_tier.limit_per_minute`; if `next_tier` isn't null, mention `next_tier.credits_needed` to reach the next tier.
4. No key available and only the tier catalog is needed → the public `…/user/sub/tiers/` route (no key).

> **Already using the SDK?** You don't need a separate balance call for the *current* balance — every SDK response carries it on `.meta`: `result.meta.remaining_balance` (Python) / `result.meta.remainingBalance` (Node) is the balance right after that call, and `result.meta.credit_consumed` / `creditConsumed` is what the call cost (SDK **≥ 0.2.0**). Use the dedicated `…/credits/balance/k/` endpoint (or `account.balance()`) when you need the balance *without* making another billable request. See `sdks.md`.

## Self-tuning rate limiter (the main use case)
`limit_per_minute` is **tier-based** and shifts as your balance crosses tier boundaries, so a hardcoded value drifts out of sync (over-throttling or hitting `429`). Read the live value from `…/credits/tier/k/` and refresh periodically.

```python
import time, threading, requests
API_KEY = "your_api_key_here"
BASE, HEADERS = "https://renidly.com/api/panel", {"X-AUTHAPI-Key": "your_api_key_here"}
_limit = {"per_minute": 7}  # safe default until first refresh

def refresh_rate_limit():
    r = requests.get(f"{BASE}/credits/tier/k/", headers=HEADERS, timeout=10)
    _limit["per_minute"] = r.json()["data"]["current_tier"]["limit_per_minute"]
    return _limit["per_minute"]

def keep_in_sync(interval=300):
    while True:
        try: refresh_rate_limit()
        except Exception: pass   # keep last known value on transient errors
        time.sleep(interval)

refresh_rate_limit()
threading.Thread(target=keep_in_sync, daemon=True).start()
# your limiter reads _limit["per_minute"] when allocating tokens
```

### Best practices
- **Cache, don't poll per request** — refresh every 1–5 min; tier only changes at a balance boundary.
- **Refresh on a `429`** — re-read `…/credits/tier/k/` (your tier likely dropped), back off, resume at the new limit.
- **Keep a safe default** so a failed refresh never runs unbounded.
- **Watch `next_tier.credits_needed`** for low-balance alerts and to know how far you are from a higher limit.
- **Target ~90% of `limit_per_minute`** to absorb clock skew across scaled instances.
