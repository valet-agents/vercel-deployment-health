# Vercel Deployment Health

Watches every production deploy for perf and error-rate regressions — posts a green tick if it's clean, or a rollback recommendation with the prior good deploy id if something's off.

## Prerequisites
- A [Vercel](https://vercel.com) account with the project(s) you want to watch — you'll authorize the Vercel MCP via OAuth
- A Slack workspace where you can install the agent's bot and invite it to one or more channels

<table>
  <tr>
    <td><strong>CHANNELS</strong></td>
    <td><code>slack</code> · <code>heartbeat</code> — every 2m</td>
  </tr>
  <tr>
    <td><strong>CONNECTORS</strong></td>
    <td><code>vercel-mcp</code></td>
  </tr>
  <tr>
    <td colspan="2" align="center">
      <br />
      <a href="https://valet.dev/deploy?from=github.com/valet-agents/vercel-deployment-health">
        <img src="https://raw.githubusercontent.com/valet-agents/vercel-deployment-health/main/.github/deploy-button.svg" alt="Deploy Agent →" height="40" />
      </a>
      <br /><br />
    </td>
  </tr>
</table>
