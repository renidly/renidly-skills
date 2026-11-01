# Renidly Phone API — Reference

Base: `https://renidly.com/api/phones/v1` · Auth: `X-renidly-apikey` header · Envelope, errors, credits: see `SKILL.md`.

The Phone API returns a professional's phone numbers from their profile URL, and resolves the professional behind a phone number. Responses are **never cached**: every call runs live and is billed by its outcome. People who opted out are never returned. There is no `/health` route.

## Endpoint index

| Method & path | Purpose |
|---|---|
| `GET /find` | A person's phone numbers from their profile URL or username |
| `GET /reverse` | The people behind a phone number |

---

## GET /find
- **Params:** `url` (required) — a profile URL `https://www.linkedin.com/in/<username>` or the bare `<username>`. Anything else → `400` (`VALIDATION_ERROR`).
- **Returns:** `found` (bool), `person` (the profile, or `null` when not found — `full_name`, `first_name`, `last_name`, `headline`, `summary`, `title`, `linkedin_url`, `profile_picture`, `location`, `industry`, `follower_count`, `is_premium`, `is_open_to_work`, `experience[]`, `education[]`), and `phones[]` (empty when not found).
- **Each phone:** `phone` (E.164, e.g. `+16502530000`), `type` (`mobile` | `personal` | `direct`), `line_type` (`mobile`, `fixed_line`, `fixed_line_or_mobile`, `voip`, `personal_number`, or empty), `country` (ISO alpha-2), `is_shared` (true = also listed for several other people, e.g. a company switchboard).
- **Order:** personal lines before shared ones, mobiles first.
- **Billing:** numbers returned → the endpoint's credit cost. Profile doesn't exist, or it has no known numbers → **1 credit** (a real lookup ran). Malformed `url` (`400`), opted-out profile (`403`, `error_code` `OPTED_OUT`), or an error → free.

## GET /reverse
- **Params:** `phone` (required) — any common format: `+1 (650) 253-0000`, `650.253.0000`, `00 44 20 7946 0958`. Digits, spaces and `+ ( ) - .` only. `country` (optional, ISO alpha-2, default `US`) — used only for a number written without a country code (`+` or `00`; `011` too when `country` is `US`/`CA`), e.g. `020 7946 0958` + `country=GB`.
- **Rejected with `400`, free:** letters, extensions (`x123`), free text, numbers invalid in their country's numbering plan, toll-free / premium-rate / shared-cost / UAN / voicemail numbers, and an invalid `country`.
- **Returns:** `phone` (the input in E.164), `found` (bool), `candidates[]` — the people behind the number (same person shape as `/find`). Usually one; **up to 3** when the number is shared (e.g. a household line). A switchboard number listed for many people returns none.
- **Billing:** people returned → the endpoint's credit cost, **once per call** however many candidates. Number known but none of its people could be returned (e.g. their profiles no longer exist) → **1 credit**. Unknown number → **free**. Invalid input (`400`) or an error → free.

---

## Phone API notes & edge cases
- **Balance check:** a call is accepted only when the balance covers the endpoint's full credit cost (else `403` before any work). If the balance drops below the outcome's cost mid-call → `402` (`1080`), nothing charged, no data returned.
- **Data is not carrier-verified** — a number may be outdated. US/Canada numbers can't be told apart as mobile or landline, so their `line_type` is `fixed_line_or_mobile`.
- **Opt-out:** `/find` answers `403` (`OPTED_OUT`, free) for an opted-out profile; `/reverse` silently leaves opted-out people out of `candidates`.
- **Errors:** `503` (`1072`) temporarily unavailable — retry, free; `500` (`1000`) unexpected — free.
