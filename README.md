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

Writing a chart needs a RegiaBI trial or paid account. Running a chart you've already generated needs nothing.
[Start a free trial](https://bizintelligencechampions.com/regiabi/developers?loc=ghmcp).

Give the server your account's credentials in one of two ways:

- **When the plugin asks** (Claude Code prompts for them on install, and keeps the key in your system's secure
  credential store), or as environment variables on the server: `BIC_LICENSEE` and `BIC_LICENSE_KEY`, plus
  `BIC_SECRET_KEY` if your account uses one.
- **A credentials file**, so no secret ever sits in a project's committed config. Put this in
  `~/.bic/credentials.json` (or point `BIC_CREDENTIALS_FILE` at another path):

  ```json
  { "licensee": "...", "licenseKey": "..." }
  ```

Leave the plugin's fields blank if you use the file. Values set on the server win over the file.

## Use

Ask your agent for the chart you want, in plain words, in the project you want it in. For example:

> *Assess the shape of ./sales.csv, tell me which charts fit it, then generate the best one into ./charts.*

The agent profiles the file, asks the server which chart types fit, and writes the chart's code and data into
`./charts`, ready to import. The skill this plugin installs walks the agent through the rest: wiring
cross-filtering between charts, sizing and theming are the host library's job, so the agent doesn't write them.

**Building a Microsoft Fabric App?** Scaffold it from Rayfin's data app template, then ask your agent which charts
suit your semantic model. `suggest_charts` answers from the model's own numbers.

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
