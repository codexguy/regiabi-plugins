# RegiaBI charts: an MCP server and agent plugin

**Chart code that fits your data first try, as source you own: you pay to author it, never to run it.**

This repository packages the RegiaBI chart MCP server (`com.regiabi/chart-mcp`, published on npm as
[`@bicharts/chart-mcp`](https://www.npmjs.com/package/@bicharts/chart-mcp)) and its agent skill as a one-command
plugin for Claude Code, GitHub Copilot CLI, Codex and Gemini CLI. It holds the plugin manifests, the pinned server
command and the skill. The server itself is the npm package.

## What it does

Give your coding agent a table (a CSV, a dataframe, or a query result from a semantic model) and it gets back
working, interactive chart code that fits that data:

- **It profiles your data locally.** Row counts, cardinalities, measures versus dimensions, dates, places. No data
  leaves your machine for this step.
- **It ranks about 90 chart types by what the data can honestly show,** using a server-side eligibility engine
  rather than a guess, so a map is only offered when there are places to map and a sankey only when there are flows.
- **It returns verified source you own:** D3 for JavaScript and TypeScript projects, Plotly or matplotlib for
  Python. The code has already passed render checks. It needs no license, no account and no connection to us to
  run, deploy or ship, for any number of viewers.

The tools:

| Tool | What it's for |
| --- | --- |
| `assess_data_shape` | Profiles a table locally. Free, and needs no account. |
| `list_eligible_charts` | The authoritative list of chart types that fit this data, ranked for your project's language. |
| `suggest_charts` | For a Microsoft Fabric App over a semantic model: finds the model's natural views and what fits each, in one call. |
| `generate_chart` | Writes the chart: code, data and wiring for [`@bicharts/chart-host`](https://github.com/codexguy/bicharts) (open source), into your project. |

## Install

### Claude Code

```text
/plugin marketplace add codexguy/regiabi-plugins
/plugin install regiabi-charts@regiabi
```

### GitHub Copilot CLI

```text
copilot plugin marketplace add codexguy/regiabi-plugins
copilot plugin install regiabi-charts@regiabi
```

### Codex

```text
codex plugin marketplace add codexguy/regiabi-plugins
```

Then install RegiaBI charts from the Plugins Directory.

### Gemini CLI

```text
gemini extensions install https://github.com/codexguy/regiabi-plugins
```

### Any other MCP client

VS Code, Cursor, Claude Desktop and any other MCP client can run the server directly:

```json
{
  "mcpServers": {
    "bic-chart": {
      "command": "npx",
      "args": ["-y", "@bicharts/chart-mcp"]
    }
  }
}
```

## Configure

Nothing, usually. Writing a chart needs a RegiaBI account; running a chart you've already generated needs nothing.

**Sign in when it asks.** The first time your agent calls a tool that needs an account, the server opens your
browser at bizintelligencechampions.com. Sign in, or create a free account (it comes with a free trial), and click
**Allow**. The call your agent made then finishes by itself. The server keeps the sign-in in `~/.bic/oauth.json`
and renews it on its own, and you can sign a device out at any time from your account page.

**Or use a license key** (a machine with no browser, CI, a container). Leave sign-in alone and give the server
your account's credentials instead, in one of two ways:

- **When the plugin asks** (Claude Code prompts on install and keeps the key in your system's secure credential
  store), or as environment variables on the server: `BIC_LICENSEE` and `BIC_LICENSE_KEY`, plus `BIC_SECRET_KEY`
  if your account uses one.
- **A credentials file**, so no secret ever sits in a project's committed config. Put this in
  `~/.bic/credentials.json` (or point `BIC_CREDENTIALS_FILE` at another path):

  ```json
  { "licensee": "...", "licenseKey": "..." }
  ```

A key, when one is set, always wins over a sign-in. Leave the plugin's fields blank to sign in instead.

## Use

Ask your agent for the chart you want, in plain words, in the project you want it in. For example:

> *Assess the shape of ./sales.csv, tell me which charts fit it, then generate the best one into ./charts.*

The agent profiles the file, asks the server which chart types fit, and writes the chart's code and data into
`./charts`, ready to import. The skill this plugin installs walks the agent through the rest: wiring
cross-filtering between charts, sizing and theming are the host library's job, so the agent doesn't write them.

**Building a Microsoft Fabric App?** Scaffold it from Rayfin's data app template, then ask your agent which charts
suit your semantic model. `suggest_charts` answers from the model's own numbers.

## What it runs, sends and fetches

- **Signing in** opens your browser at `bizintelligencechampions.com` once; the server then keeps an access token and a
  refresh token in `~/.bic/oauth.json` (readable only by you) and sends the access token with each call to the service.
- **It runs** one local process: `npx -y @bicharts/chart-mcp@<version>`, the exact version the plugin pins. npm
  downloads that package from the public npm registry the first time.
- **`assess_data_shape` sends nothing.** It profiles your data on your machine.
- **`list_eligible_charts`, `suggest_charts` and `generate_chart` call the RegiaBI service** at
  `bizintelligencechampions.com` with your account's credentials and a profile of your table: column names, types,
  counts and summary statistics. By default no rows are sent; a `privacy_level` argument can send less (names
  only) or more (obfuscated sample rows). `suggest_charts` queries your semantic model through your Fabric App
  project's own `fabric-app-data` CLI, signed in as you, and sends the service the same kind of profile.
- **It writes** the generated chart code and its data into the folder you ask for, and nowhere else.
- **Telemetry:** one row when `generate_chart` hands code over (the call's duration, the row count, the language
  and the server's build) and one when a generation fails after the service answered. No data, and no usage
  counter: the service counts calls from the requests it receives. Set `BIC_MCP_TELEMETRY=0` in the server's
  environment to turn both rows off.
- **Signing in sends this computer's name** (its hostname), so the sign-in shows up on your account page and you can
  sign that machine out on its own.
- **`check_app` asks the npm registry** for the latest versions of the packages your app uses (`check_npm: false`
  skips it), and **`preview.html` loads d3 and mermaid from cdn.jsdelivr.net** when you open it (it's only written
  when you pass `preview_html: true`).

## Learn more

- [RegiaBI for developers](https://bizintelligencechampions.com/regiabi/developers?loc=ghmcp): docs, examples and
  live demos
- [`@bicharts/chart-mcp` on npm](https://www.npmjs.com/package/@bicharts/chart-mcp): every tool's parameters and
  results
- [`@bicharts/chart-host`](https://github.com/codexguy/bicharts): the open-source library that runs the charts
- [The official MCP Registry listing](https://registry.modelcontextprotocol.io/v0/servers/com.regiabi%2Fchart-mcp/versions/latest)

## Support and terms

- [Support](https://bizintelligencechampions.com/support)
- The server and the service it calls are covered by the
  [RegiaBI Programmatic Access Terms](https://bizintelligencechampions.com/terms-regiabi) and the
  [privacy policy](https://bizintelligencechampions.com/privacy). The server's own license ships in its npm
  package.
- The files in this repository (the plugin manifests, the README and the skill) are under the MIT license in
  [LICENSE](./LICENSE). They contain no server source.
