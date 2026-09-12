# Lead Sniper — High-Value Stargazer Tracker

An automated n8n workflow that monitors a GitHub repository's stargazers, identifies high-value leads based on influence signals, enriches their profile data, generates an AI-written sales pitch, and posts a formatted alert to Slack — all without manual intervention.

Built as **Assignment 1** for the AI Automation Intern role at Yellow.ai.

---

## What it does

Every 15 minutes, the workflow:

1. Polls a target GitHub repository for new stargazers since the last check.
2. Enriches each new stargazer with their full public profile via the GitHub Users API.
3. Filters for **high-value leads** using the rule:
   ```
   followers > 100  OR  public_repos > 50
   ```
4. For qualifying leads, generates a one-sentence AI sales pitch (via Claude) referencing their bio and company.
5. Posts a formatted alert — name, GitHub profile, stats, bio, and the AI pitch — to a Slack channel.

Non-qualifying stargazers are skipped without any output.

---

## Workflow structure

| Node | Purpose |
|---|---|
| Schedule Trigger (Every 15 min) | Starts each poll cycle |
| Load Last Checked Timestamp | Reads the persisted cursor from workflow static data |
| GitHub: Get Stargazers | Fetches stargazers with `starred_at` timestamps |
| Parse Stargazers + Rate Headers | Extracts rate-limit headers and flattens the star list |
| Filter: Only New Stars | Keeps only stars newer than the last-checked cursor |
| Loop Over New Stargazers | Iterates new stargazers one at a time |
| Throttle (1s between calls) | Paces requests to respect GitHub's secondary rate limit |
| GitHub: Enrich User Profile | Fetches full profile data, with retry + backoff on failure |
| Flatten Profile Data | Extracts the fields needed for scoring and the pitch |
| IF: High-Value Lead? | Applies the qualification rule |
| AI: Generate Sales Pitch | Calls Claude (`claude-sonnet-4-6`) for a 1-sentence pitch |
| Format Slack Message | Builds a Slack Block Kit payload |
| Slack: Post Lead Alert | Posts the alert to Slack via webhook |
| Skip (Low Value) | No-op branch for leads that don't qualify |
| Save Last Checked Timestamp | Persists the newest `starred_at` seen this run |

---

## Rate-limit strategy

GitHub enforces both a **primary quota** (5,000 req/hour, authenticated) and a **secondary burst limit** (anti-abuse protection independent of remaining quota). This workflow handles both:

- **Fixed throttle** — a 1-second delay before every enrichment call keeps request pacing under the secondary burst threshold.
- **Header observability** — `x-ratelimit-remaining` and `x-ratelimit-reset` are extracted from every response and attached to each item, so quota consumption is visible in execution logs.
- **Retry with backoff** — enrichment calls retry up to 3 times with a 2-second delay on transient network errors, without retrying 404s (which can never succeed).
- **Cursor-based deduplication** — a persisted `lastCheckedTimestamp` ensures only genuinely new stargazers are ever enriched, minimizing call volume in the first place.
- **Compliant requests** — a `User-Agent` header (required by GitHub) and a pinned `X-GitHub-Api-Version` are set on all API calls.

**Known limitation:** rate-limit headers are currently tracked for observability only; the throttle delay is fixed rather than adaptive. A natural next step would be branching the delay length off `rateLimitRemaining` when it drops below a threshold.

---

## Setup

1. Import `Lead_Sniper_Workflow.json` into n8n.
2. Create the following credentials in n8n's Credential Store (do **not** hardcode these anywhere):
   - **GitHub API Token** — Header Auth credential, used by the two GitHub HTTP Request nodes.
   - **Anthropic API Key** — used by the `AI: Generate Sales Pitch` node.
3. Set the following environment variable in your n8n instance:
   - `SLACK_WEBHOOK_URL` — your Slack incoming webhook URL.
4. Update the target repository (`owner/repo`) in the `GitHub: Get Stargazers` node.
5. Activate the workflow.

---

## Tech stack

- [n8n](https://n8n.io) — workflow orchestration
- GitHub REST API — stargazer + user data
- Claude (Anthropic API) — sales pitch generation
- Slack Incoming Webhooks — alert delivery

---

## Security note

This repository's `workflow.json` contains **no hardcoded credentials**. GitHub authentication uses n8n's credential store; the Slack webhook is referenced via an environment variable (`{{ $env.SLACK_WEBHOOK_URL }}`). Anyone importing this workflow must supply their own credentials as described in Setup.
