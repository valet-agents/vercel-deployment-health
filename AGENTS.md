This folder contains the source for a Skilled Agent originally built for the Valet runtime. Changes should follow the Skilled Agent open standard.

## Setup

### Connectors

- **vercel-mcp**: The Vercel MCP server, OAuth-authenticated. The agent uses it to list production deploys, pull deploy metadata (commit, author, branch, build duration), pull recent analytics and logs for error-rate / perf comparisons, and (only on explicit Slack confirmation) trigger rollbacks or redeploys. Add it from the catalog at the org level so other Vercel-powered agents can share it.

### Channels

- **slack** (slack): The agent's per-agent Slack bot. Listens for @mentions and replies in-thread, and posts deploy alerts (green tick or rollback recommendation) to whichever channels the bot has been invited to. Slack writes use the auto-injected outbound Slack connector.
- **heartbeat** (heartbeat): Fires once a day. Polls each watched Vercel project for new production deploys, evaluates each one against the previous good deploy, and posts the result. Declared inline in `valet.yaml`, so it's created automatically by the dashboard setup flow.

### Secrets

This agent uses the OAuth variant of the Vercel MCP, so no API token is needed at the org or agent level. The OAuth grant happens in the dashboard setup flow when you connect Vercel.

### External Setup

1. Authorize Vercel via OAuth in the dashboard setup flow. Make sure the OAuth grant covers the Vercel team(s) that own the project(s) you want to watch.
2. **Pick which projects to watch.** By default the agent watches every project the OAuth grant exposes. To narrow the scope, set the `VERCEL_PROJECTS` env var on the agent to a comma-separated list of project ids or names (e.g. `VERCEL_PROJECTS=prod-web,api-gateway`).
3. After deploy, invite the agent's Slack bot to whichever channel(s) you want deploy alerts in. The agent posts to every channel it's a member of — invite it to one focused channel (e.g. an engineering or ops channel), or several. If the bot has not been invited anywhere, the alert is sent as a DM to the workspace install user with a one-line nudge.
4. Invite the bot to any additional channels where teammates should be able to @mention it for ad-hoc Vercel questions (e.g. *"is staging green?"*, *"what's the error rate on production right now?"*).
5. The first heartbeat after deploy is the seed run — it records the most recent existing deploy as the watermark and posts nothing. The next prod deploy after that is the first one alerted on. To smoke-test sooner, @mention the bot in Slack with a question like *"anything off about the last release?"* — that exercises the Slack + Vercel path without waiting for a new deploy.

## Customizing

- **Change the heartbeat interval**: edit `every` on the `heartbeat` channel in `valet.yaml`, then redeploy. The default `24h` runs a daily deploy review; drop it to `1h` or `5m` if you deploy often and want regressions caught closer to release time.
- **Tune the regression thresholds**: open SOUL.md "Phase 4: Diff against the previous good deploy" and edit the multipliers and absolute-value floors. The defaults (error rate > 2x AND > 0.5%; p95 > 1.5x AND > 500ms) avoid noise on low-traffic projects but catch real regressions on busy ones. Lower the absolute-value floors if your projects have very low baseline error rates and you want tighter alerting.
- **Watch a different scope of projects**: set or update `VERCEL_PROJECTS` on the agent to a comma-separated list of project ids or names. Unset it to fall back to every project the OAuth grant exposes.
- **Control where deploy alerts post**: invite or remove the bot from channels in Slack — that's the only signal the agent uses. There is no channel name in the configuration.
