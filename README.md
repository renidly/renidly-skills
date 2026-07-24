# Renidly Skills for Claude Code

A Claude Code plugin marketplace that ships the **`renidly`** skill — a complete,
code-verified reference for the Renidly B2B professional-data APIs (Data, Live,
Email, and Account & Credits): endpoints, parameters, response shapes, auth,
pagination, credits, and batch jobs.

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

- `SKILL.md` — overview, auth, shared conventions, decision guide
- `data-api.md` — Data API (`/api/data/v1`)
- `live-api.md` — Live API (`/api/v2`)
- `email-api.md` — Email API (`/api/emails/v1`)
- `account-api.md` — Account & Credits (`/api/panel`): balance, tier, rate limit, route costs
