# Connecting AI clients to the Elasticsearch MCP server

This guide covers connecting Claude Code, OpenAI Codex and GitHub Copilot in VS Code to the
Elasticsearch MCP server, and the AEM as a Cloud Service CDN log fields you can then query.

Start the server first — see [README.md](README.md). Both deployments expose the same endpoint:

```text
http://localhost:8080/mcp
```

Confirm it is up before configuring a client:

```bash
curl http://localhost:8080/ping
```

## Contents

- [Claude Code](#claude-code)
- [OpenAI Codex](#openai-codex)
- [GitHub Copilot / VS Code](#github-copilot--vs-code)
- [stdio instead of HTTP](#stdio-instead-of-http)
- [The CDN log fields you can query](#the-cdn-log-fields-you-can-query)
- [What to ask it](#what-to-ask-it)
- [Do not trust a generated query blindly](#do-not-trust-a-generated-query-blindly)

> Each client uses a slightly different schema. Claude Code uses `mcpServers`, VS Code uses
> `servers`, Codex uses TOML. Copying one client's config into another does not work — use the right
> block for each.

## Claude Code

### Project configuration

A `.mcp.json` file in the repository root configures MCP servers for everyone working in that
repository, and is meant to be checked into version control:

```json
{
  "mcpServers": {
    "aem-cdn-logs": {
      "type": "http",
      "url": "http://localhost:8080/mcp"
    }
  }
}
```

The `"type"` field is required for any entry that has a `url`. Claude Code reads an entry without a
`type` as a stdio server and skips it. `"streamable-http"` is accepted as an alias for `"http"`.

This repository does **not** ship a `.mcp.json`, on purpose: it would prompt every contributor to
approve a server they may not be running. Create it yourself if you want it, or use one of the
non-project scopes below.

### User and local configuration

Claude Code has three scopes:

| Scope | Applies to | Stored in |
|---|---|---|
| `local` (default) | the current project, just you | `~/.claude.json` |
| `project` | the current project, shared with the team | `.mcp.json` in the project root |
| `user` | all your projects, just you | `~/.claude.json` |

Use `user` scope if you analyse CDN logs from several checkouts, and `local` if you just want it
here without committing anything.

### CLI

The recommended way to add it:

```bash
claude mcp add --transport http aem-cdn-logs http://localhost:8080/mcp
```

That writes `local` scope. To pick a different one:

```bash
# shared with the team, writes .mcp.json
claude mcp add --transport http aem-cdn-logs --scope project http://localhost:8080/mcp

# available in all your projects
claude mcp add --transport http aem-cdn-logs --scope user http://localhost:8080/mcp
```

Manage and inspect:

```bash
claude mcp list
claude mcp get aem-cdn-logs
claude mcp remove aem-cdn-logs
```

Inside a session, `/mcp` shows connection status and lets you reconnect.

## OpenAI Codex

Codex configures MCP servers in TOML. The global file is `~/.codex/config.toml`, and it is shared by
the Codex CLI, the IDE extension and the ChatGPT desktop app — configure it once and all three see
the server.

```toml
[mcp_servers.aem-cdn-logs]
url = "http://localhost:8080/mcp"
```

For a streamable HTTP server the fields are `url` (required), plus the optional
`bearer_token_env_var`, `http_headers` and `auth`. None of the optional ones are needed here: the
MCP server is on loopback and takes no authentication of its own.

Note that an inline `bearer_token` is **not** supported — Codex requires `bearer_token_env_var` and
reads the value from the environment, so tokens never land in the config file. That matters if you
later point the client at an authenticated endpoint.

### CLI

```bash
codex mcp add aem-cdn-logs --url http://localhost:8080/mcp
codex mcp list
```

`codex mcp add` writes to the global `~/.codex/config.toml`; it has no scope flag.

### Project scope

Codex also reads `.codex/config.toml` from the repository, for trusted projects only. Same syntax:

```toml
# .codex/config.toml
[mcp_servers.aem-cdn-logs]
url = "http://localhost:8080/mcp"
```

Use the global file unless you specifically want the server scoped to this checkout.

## GitHub Copilot / VS Code

VS Code uses `.vscode/mcp.json`, with a top-level **`servers`** key — not `mcpServers`:

```json
{
  "servers": {
    "aem-cdn-logs": {
      "type": "http",
      "url": "http://localhost:8080/mcp"
    }
  }
}
```

Workspace configuration lives in `.vscode/mcp.json` and can be committed to share with the team. For
a configuration that applies across all your workspaces, run **MCP: Open User Configuration** from
the Command Palette and add the same `servers` block there.

The file also supports an `inputs` array for values you do not want to hardcode, referenced as
`${input:some-id}`. You do not need it for this setup, but you would for an authenticated endpoint:

```json
{
  "inputs": [
    {
      "id": "es-api-key",
      "type": "promptString",
      "description": "Elasticsearch API key",
      "password": true
    }
  ],
  "servers": {
    "aem-cdn-logs": {
      "type": "http",
      "url": "https://mcp.example.com/mcp",
      "headers": { "Authorization": "ApiKey ${input:es-api-key}" }
    }
  }
}
```

### Commands

From the Command Palette (`Ctrl/Cmd+Shift+P`):

- **MCP: Add Server** — guided flow that writes the configuration for you
- **MCP: List Servers** — view configured servers; start, stop and restart them from here
- **MCP: Show Output** — server logs, the first place to look when a connection fails
- **MCP: Browse Resources** — browse resources exposed by a server
- **MCP: Reset Cached Tools** — refresh the tool list after the server changes
- **MCP: Open User Configuration** — edit the global, cross-workspace configuration

Then open Copilot Chat in **Agent** mode and check the tools picker for the `aem-cdn-logs` tools.

## stdio instead of HTTP

Some clients work better with stdio, where the client launches the MCP process itself. The official
image supports it with the `stdio` subcommand instead of `http`, and no port is involved.

```text
AI client
    |
    | stdio
    v
docker run -i --rm docker.elastic.co/mcp/elasticsearch:0.4.6 stdio
    |
    v
Elasticsearch
```

### Against the integrated ELK stack

The container needs to be on the Compose network to resolve the `elasticsearch` service name. The
network is named after the Compose project, which defaults to the directory name — `elk_elastic` for
this repository:

```bash
docker network ls | grep elastic     # confirm the name
```

Claude Code, `~/.claude.json` or `.mcp.json`:

```json
{
  "mcpServers": {
    "aem-cdn-logs": {
      "command": "docker",
      "args": [
        "run", "-i", "--rm",
        "--network", "elk_elastic",
        "-e", "ES_URL=http://elasticsearch:9200",
        "docker.elastic.co/mcp/elasticsearch:0.4.6",
        "stdio"
      ]
    }
  }
}
```

No credentials appear anywhere because the bundled cluster has security disabled.

### Against a remote cluster

Do not paste the API key into the client configuration file. Pass the variable names through with
bare `-e` flags and let Docker inherit the values from the environment the client was started in, so
the secret stays in one place — your shell profile, or the `.env` you already have:

```json
{
  "mcpServers": {
    "aem-cdn-logs": {
      "command": "docker",
      "args": [
        "run", "-i", "--rm",
        "--env-file", "/absolute/path/to/AEMCS-CDN-Log-Analysis-Tooling/ELK/mcp/.env",
        "docker.elastic.co/mcp/elasticsearch:0.4.6",
        "stdio"
      ]
    }
  }
}
```

`--env-file` reuses the same git-ignored `.env` the standalone Compose deployment uses — one
credential, one file, no duplication.

The Codex equivalent, which forwards named variables from your shell instead:

```toml
[mcp_servers.aem-cdn-logs]
command = "docker"
args = ["run", "-i", "--rm", "-e", "ES_URL", "-e", "ES_API_KEY",
        "docker.elastic.co/mcp/elasticsearch:0.4.6", "stdio"]
env_vars = ["ES_URL", "ES_API_KEY"]
```

### Which to use

**HTTP**

```text
Multiple AI clients
       |
       +----> shared localhost MCP server
```

- The server starts and stops independently of any client.
- Several clients — Claude Code, Codex and VS Code at once — share one instance.
- Configuration lives in one place: Compose and `.env`.
- Container lifecycle is managed the way the rest of the stack is: `docker compose ps`,
  `docker compose logs`, restart policies, healthchecks.

**stdio**

```text
AI client
    |
    +----> starts MCP process/container
```

- Simpler: no port, nothing listening, nothing to bind to loopback.
- The client owns the lifecycle — the server exists only while the client runs.
- Each client spawns its own container, each needing its own credential wiring.

**HTTP is the recommended approach for this repository.** The MCP server belongs to the same ELK
stack you already start and stop with Compose, and the integrated deployment gets it for free with a
profile flag. Debugging is also markedly easier: you can `curl /ping`, tail
`docker compose logs -f aem-cdn-logs-mcp`, and call a tool with `curl` to see whether a failure is
in the client or in Elasticsearch — none of which you can do when the process only exists inside an
AI client.

Reach for stdio when a client does not support streamable HTTP, or when you want no listening socket
at all.

## The CDN log fields you can query

The Logstash pipeline writes to the index **`aem-cdn-logs`**, with **`timestamp`** as the time field
— the same index pattern the bundled dashboards use.

Fields are mapped dynamically, so string fields are `text` with a `.keyword` subfield. **Aggregations
and sorting need the `.keyword` form** (`cli_ip.keyword`), while full-text matching uses the bare
field (`url`). Getting this wrong is the single most common reason an agent's query returns nothing.

| Field | Type | What it holds |
|---|---|---|
| `timestamp` | date | Request time, in UTC |
| `@timestamp` | date | Same value, normalised by the Logstash `date` filter |
| `cli_ip` | text + keyword | Client IP address |
| `cli_country` | text + keyword | Client country code |
| `req_ua` | text + keyword | User-Agent |
| `host` | text + keyword | Requested host |
| `url` | text + keyword | Requested path |
| `method` | text + keyword | HTTP method |
| `status` | long | HTTP response status |
| `cache` | text + keyword | CDN cache state: `HIT`, `MISS`, `PASS`, `SYNTH` |
| `res_ctype` | text + keyword | Response content type |
| `res_age` | long | Age of the cached response, seconds |
| `ttfb` | float | Time to first byte, seconds |
| `pop` | text + keyword | CDN point of presence |
| `rid` | text + keyword | Request ID |
| `rules` | text + keyword | Raw traffic-filter rule match string |
| `waf_action` | text + keyword | Action parsed out of `rules`, e.g. `log`, `block` |
| `waf_flags`, `waf_flags_items` | text + keyword | WAF flags; `_items` is the split, per-flag array |
| `waf_match`, `waf_match_items` | text + keyword | Matched rules; `_items` is the split array |
| `aem_env_name` | text + keyword | Environment, from the `logs/<env>/` subfolder name |
| `cli_ip_pop_key` | text + keyword | `<pop>_<cli_ip>`, for per-POP per-IP rate analysis |
| `custom_req_ref` | text + keyword | Referer, only when the CDN is configured to log it |
| `custom_bot_name` | text + keyword | Bot name, only when the CDN is configured to log it |

`custom_req_ref` and `custom_bot_name` require the `logProperty` CDN configuration described in the
[ELK README](../README.md); without it they are absent from the index.

Two things worth telling the agent up front:

- **Timestamps are UTC.** Cloud Manager exports UTC, and Elasticsearch stores UTC. Kibana renders in
  your local timezone, so a number from an MCP query and the same number read off a dashboard can
  disagree by your UTC offset.
- **On Elasticsearch 8.10.1 the `esql` and `get_mappings` tools do not work** — see
  [README.md](README.md#what-works-against-elasticsearch-8101). Ask for Query DSL instead.

## What to ask it

A few starting points. You ask in plain language; the agent builds the Elasticsearch query itself.

```text
Show the top 20 client IP addresses by request count in the last 24 hours.
```

```text
Which URLs returned the most 5xx responses yesterday?
```

```text
What is the cache hit ratio, and which URLs MISS most often?
```

```text
Group requests by User-Agent and tell me which clients look automated.
```

Tell the agent this once at the start of a session, or it will guess wrong:

```text
The index is aem-cdn-logs, the time field is timestamp and it is UTC. String fields have a
.keyword subfield - use cli_ip.keyword, url.keyword and so on for aggregations. Use the search
tool with Query DSL; the esql tool does not work on this cluster.
```

Adjust time ranges to match your data. `now-24h` is relative to the present, while downloaded logs
are usually historical, so an absolute range such as
`"gte": "2026-09-10T00:00:00Z", "lte": "2026-09-11T00:00:00Z"` is often what you want.

## Do not trust a generated query blindly

An AI-generated Elasticsearch query is a draft, not an answer. It can be syntactically valid, return
plausible numbers, and still measure the wrong thing — and nothing in the output tells you that
happened. This matters most when the result feeds a security or rate-limiting decision, because a
quietly wrong query produces a confidently wrong conclusion.

Make the agent show its work:

```text
Before executing the query, explain the ES|QL or Query DSL you intend to use.
```

```text
Show the query used to calculate this result.
```

```text
What would make this number wrong? Which assumptions did you make that I should check?
```

Then check:

- **Time range** — does it cover the data you actually loaded? Relative ranges like `now-24h` return
  nothing against logs exported last week.
- **Timezone** — `timestamp` is UTC. Kibana shows local time. A "daily peak" can land in the wrong
  day.
- **Field names** — `cli_ip` for matching, `cli_ip.keyword` for aggregating. Aggregating on a `text`
  field either errors or, worse, aggregates on analysed tokens.
- **Aggregations** — is a `terms` agg ordered by the metric you care about, or just by count? Terms
  aggregations are approximate: check `doc_count_error_upper_bound` and `sum_other_doc_count` before
  calling something a "top" result.
- **Cardinality** — `cardinality` is an approximation, and `terms` `size` truncates. A "top 20" over
  thousands of IPs can silently miss the one that matters.
- **Percentiles** — approximate by design, and meaningless over very few buckets. Ask what the
  denominator was.
- **Filters and excluded traffic** — the bundled dashboards exclude `/systemready` and `cache=SYNTH`.
  If your query does not, its numbers will not match the dashboards.
- **CDN semantics** — `PASS` is not `MISS`; rate limits apply per POP, not globally; a `HIT` at the
  CDN never reached the origin. A query that is correct as Elasticsearch can still be wrong as CDN
  analysis.
- **Sampling and completeness** — under heavy load, not every request reaches the logs Cloud Manager
  exports. Absence of evidence is not evidence of absence.

The cheapest check is to reproduce one number a different way — in Kibana, on a dashboard, or with a
narrow `search` over a single hour — before acting on the whole analysis.
