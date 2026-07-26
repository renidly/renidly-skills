# Renidly Skills for Claude Code

A Claude Code plugin marketplace that ships the **`renidly`** skill — a complete,
code-verified reference for the Renidly B2B professional-data APIs (Data, Live,
Email, and Account & Credits): endpoints, parameters, response shapes, auth,
pagination, credits, and batch jobs.

It's **SDK-first**: when you ask any AI to build with Renidly, the skill steers it
to the **official SDKs** and shows how to use them correctly in both languages —
so you get working code, not guesses.

- **Python** — `pip install renidly` · [PyPI](https://pypi.org/project/renidly/) · [repo](https://github.com/renidly/renidly-python)
- **Node / TypeScript** — `npm install renidly` · [npm](https://www.npmjs.com/package/renidly) · [repo](https://github.com/renidly/renidly-node)

## Install

In Claude Code:

```
/plugin marketplace add <your-github-username>/renidly-skills
/plugin install renidly@renidly-skills
```

Then reload:

```
/reload-plugins
```

The skill auto-activates on Renidly questions, or invoke it manually with
`/renidly:renidly`.

## Update

```
/plugin marketplace update renidly-skills
/plugin update
```

## What's inside

- `SKILL.md` — overview, auth, shared conventions, SDK-first policy, decision guide
- `sdks.md` — the official Python & Node SDKs: install, config, every method in both languages, pagination, batch, errors, rate limiting, and how to verify against source
- `data-api.md` — Data API (`/api/data/v1`)
- `live-api.md` — Live API (`/api/v2`)
- `email-api.md` — Email API (`/api/emails/v1`)
- `account-api.md` — Account & Credits (`/api/panel`): balance, tier, rate limit, route costs
