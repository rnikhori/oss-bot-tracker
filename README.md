# oss-bot-tracker

State for a daily cloud routine that works through `queue.json` — 95 open bugs across 10
Go repos from remote-first engineering shops (Grafana Labs, SUSE/Rancher, Traefik Labs,
Mattermost, Elastic, Cloudflare, InfluxData, Ory, Fly.io, DigitalOcean), originally
assembled as "The Remote Ten" triage board.

## Files

- `queue.json` — the full issue pool. Each entry has `status`: `pending` | `in_progress` |
  `pr_opened` | `skipped` | `not_applicable` | `failed`. `priority_rank` (1–10, else
  `null`) marks the critical shortlist — those go first, then the rest in list order.
- `progress_log.md` — one entry per day the bot runs, appended (never overwritten).

## Daily protocol (what the bot does each run)

1. Pull latest `queue.json` from this repo.
2. Pick the first entry where `status == "pending"`, ordered by `priority_rank` (nulls
   last), then by list position.
3. Set that entry's `status` to `in_progress`, commit, push — so a second run started
   before this one finishes won't double-pick it.
4. Clone the target repo, read its contribution docs / CLAUDE.md-equivalent, understand
   the subsystem the issue touches.
5. Triage: confirm the issue is still real against current code. If not (already fixed,
   stale, or genuinely not reproducible), set `status: "not_applicable"`, write why in
   `notes`, commit, stop for the day.
6. Implement the minimal fix. Add/update unit tests if the repo has a test framework.
   Run whatever integration tests are reasonably scoped and fast enough to run locally.
7. Spawn a review sub-agent on a heavier model (opus) to adversarially check the diff
   before finalizing. Address real findings; don't force a fix past a legitimate
   objection — if the reviewer finds the approach is wrong, iterate or mark `failed`
   with why, rather than opening a weak PR.
8. Attempt e2e testing only where it's realistically runnable without heavy infra
   (local docker-compose, a `make test-e2e` that doesn't need cloud credentials, etc.) —
   skip and say why otherwise, never fake it.
9. Fork, push, open the PR. Set `status: "pr_opened"`, fill in `pr_url`.
10. Append one entry to `progress_log.md` either way (success, not_applicable, or
    failed) and push.

One issue per day. If a day's issue can't be resolved within a single run, mark it
`failed` with notes and move to the next `pending` entry the following day — don't block
the whole queue on one hard issue.
