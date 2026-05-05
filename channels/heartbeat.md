# Deploy Sweep (Heartbeat)

The heartbeat channel fires every 2 minutes. There is no payload
to parse — your job is to detect new production deploys on the
watched Vercel project(s) since the last sweep, evaluate each
one, and post the result to Slack.

## What it does

1. Resolve watched projects per SOUL **Phase 1**: prefer
   `VERCEL_PROJECTS` env var (comma-separated project ids or
   names); otherwise list every project the OAuth grant exposes.
2. For each project, read MEMORY.md for `last_seen_deploy_id`
   and `last_seen_created_at`. List recent production deploys
   (`target: production`), sorted by `created` descending. Take
   any deploys with `created` newer than the watermark.
3. **First run for a project**: seed the watermark to the most
   recent existing prod deploy and stop for that project — the
   backlog is not alerted on.
4. Skip deploys still `BUILDING` or `QUEUED` — the next
   heartbeat will pick them up. Do not advance the watermark
   past them.
5. For each new `READY` prod deploy, run SOUL **Phase 3** (pull
   metadata + metrics) and **Phase 4** (diff vs the previous
   good deploy). If metrics aren't available yet (deploy too
   fresh), skip on this fire and re-evaluate on the next
   heartbeat.
6. Post per SOUL **Phase 5**: green tick if clean, rollback
   recommendation if regressed, build-failure note if the deploy
   ended in `ERROR`. One Slack post per deploy per channel.
7. Update MEMORY.md `last_seen_deploy_id` and
   `last_seen_created_at` for each project past the deploy you
   just evaluated.

## MEMORY.md state shape

The agent persists a small block in MEMORY.md to track what's
been processed. Shape (one entry per watched project):

```
## vercel-deployment-health

projects:
  prod-web:
    last_seen_deploy_id: dpl_XXXXXXXXXXXXXXXXXXXXXXXX
    last_seen_created_at: 2026-05-05T17:42:00.000Z
    last_good_deploy_id: dpl_YYYYYYYYYYYYYYYYYYYYYYYY
  api-gateway:
    last_seen_deploy_id: dpl_ZZZZZZZZZZZZZZZZZZZZZZZZ
    last_seen_created_at: 2026-05-05T17:38:00.000Z
    last_good_deploy_id: dpl_WWWWWWWWWWWWWWWWWWWWWWWW
```

Update this block in place each fire. Do not append a new block
per fire. `last_good_deploy_id` is the most recent `READY` prod
deploy that did not regress — used as the rollback target on the
next regression.

## Where to post

Per SOUL: every Slack channel the bot is a member of, one post
per deploy per channel. If the bot is in zero channels, DM the
workspace install user with the alert and a one-line invite hint.

## Skip conditions

Skip posting (and stay silent) if any of these are true:

- No new prod deploys for any watched project since the
  watermark. Do not post a "no new deploys" line.
- This is the first run for a project after deploy. Seed the
  watermark to the most recent existing prod deploy and stop —
  the backlog is not alerted on.
- A deploy is still `BUILDING` or `QUEUED`. Wait for the next
  heartbeat.
- A deploy is `READY` but metrics aren't available yet (too
  fresh — vercel-mcp returns no analytics window). Skip on this
  fire, do not advance the watermark, re-evaluate next heartbeat.
- The watched-projects resolver returns zero projects (no
  `VERCEL_PROJECTS` set and the OAuth grant exposes none). Write
  `projects: not_found` to MEMORY.md and stop. Do not post.

## Dedup guarantee

The single source of truth for "have I posted about this deploy
yet" is MEMORY.md `last_seen_deploy_id` per project. Always read
it before posting and always update it after posting. Never post
twice for the same deploy id.
