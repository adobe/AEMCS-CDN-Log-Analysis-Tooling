# Elasticsearch MCP for AEMCS CDN log analysis

[Model Context Protocol](https://modelcontextprotocol.io/) (MCP) is an open protocol that lets an AI
coding agent call tools on your behalf. The Elasticsearch MCP server exposes your indexed AEM as a
Cloud Service CDN logs as a set of read-only tools, so you can ask questions like *"which client IPs
came closest to the 40 requests/second rate limit yesterday?"* and have the agent build, run and
explain the Elasticsearch query for you.

**MCP is entirely optional.** Nothing in the existing ELK setup changes if you never enable it.
`docker compose up -d` in the [`ELK`](../) directory starts exactly the same three containers it
started before.

## Contents

- [Which deployment do I want?](#which-deployment-do-i-want)
- [Integrated deployment](#integrated-deployment)
- [Standalone deployment](#standalone-deployment)
- [Network security](#network-security)
- [Elasticsearch permissions](#elasticsearch-permissions)
- [Health and verification](#health-and-verification)
- [What works against Elasticsearch 8.10.1](#what-works-against-elasticsearch-8101)
- [Troubleshooting](#troubleshooting)

Once the server is running, see [CLIENTS.md](CLIENTS.md) to connect Claude Code, OpenAI Codex or
GitHub Copilot in VS Code, and for the CDN log fields you can query.

## Which deployment do I want?

| | Integrated | Standalone |
|---|---|---|
| Elasticsearch runs | on this machine, from [`ELK/compose.yaml`](../compose.yaml) | somewhere else (shared ELK host, VM, Elastic Cloud) |
| Compose file | [`../compose.yaml`](../compose.yaml), `mcp` profile | [`compose.yaml`](compose.yaml) in this directory |
| Credentials | none needed (bundled cluster has security disabled) | API key in a local `.env` |
| Start with | `docker compose --profile mcp up -d` | `docker compose up -d` |

Both put the MCP endpoint at `http://localhost:8080/mcp` on your workstation, so the AI client
configuration in [CLIENTS.md](CLIENTS.md) is identical either way.

### Integrated architecture

```text
AI client (Claude Code / Codex / Copilot)
   |
   | MCP over streamable HTTP -> 127.0.0.1:8080/mcp
   v
aem-cdn-logs-mcp  (container, "mcp" Compose profile)
   |
   | Elasticsearch API -> http://elasticsearch:9200   (Compose network, not localhost)
   v
elasticsearch
   ^
   |
logstash  <- ELK/logs/<env>/*.log
```

### Standalone architecture

```text
Claude / Codex / Copilot
          |
          | MCP over streamable HTTP
          v
   127.0.0.1:8080/mcp
          |
          v
aem-cdn-logs-mcp container (your workstation)
          |
          | HTTPS over VPN / private network / SSH tunnel
          v
   Remote Elasticsearch
```

## Integrated deployment

The MCP service lives in the existing [`ELK/compose.yaml`](../compose.yaml) behind the `mcp`
[Compose profile](https://docs.docker.com/compose/how-tos/profiles/). A service with a profile is
skipped unless that profile is requested, which is what keeps the change backward compatible.

```bash
cd ELK
docker compose --profile mcp up -d
```

The existing workflow is unaffected:

```bash
cd ELK
docker compose up -d
```

This starts `elasticsearch`, `logstash` and `kibana` only — no MCP container, no extra port, no
change in behaviour for anyone who does not opt in.

To stop just the MCP service and leave the stack running:

```bash
cd ELK
docker compose stop aem-cdn-logs-mcp
```

### Credentials

The bundled Elasticsearch runs with security disabled
([`elasticsearch/config/elasticsearch.yml`](../elasticsearch/config/elasticsearch.yml) sets
`xpack.security.enabled: false`), so the integrated deployment needs **no credentials at all**. The
service still reads `ES_API_KEY`, `ES_USERNAME`, `ES_PASSWORD` and `ES_SSL_SKIP_VERIFY` from the
environment, so if you turn security on you can supply them from an `ELK/.env` file — which is
git-ignored — without editing `compose.yaml`. Empty values are treated as "not set".

### Changing the port

Set `MCP_PORT` in `ELK/.env` (or in your shell) if `8080` is taken:

```dotenv
MCP_PORT=8090
```

Remember to update your AI client configuration to match.

## Standalone deployment

Use this when Elasticsearch runs on another machine and you only want the MCP server locally.

```bash
cd ELK/mcp
cp .env.example .env
# edit .env: set ES_URL and ES_API_KEY
docker compose up -d
```

`.env` is required — Compose reads it through `env_file`, and `ES_URL` has no useful default. It is
git-ignored; only [`.env.example`](.env.example) is committed, and it contains no secrets.

See [`.env.example`](.env.example) for every supported variable. The short version:

```dotenv
ES_URL=https://elk.example.com:9200
ES_API_KEY=<base64 API key>
```

API-key authentication is the recommended approach: keys are scoped to exactly the privileges you
grant them, can be revoked individually without touching a user account, and are what the
[permissions](#elasticsearch-permissions) section below builds. `ES_USERNAME` / `ES_PASSWORD` basic
auth is supported as a fallback when you cannot create API keys.

Leave `ES_SSL_SKIP_VERIFY=false`. Set it to `true` only against a development cluster with a
self-signed certificate — it disables certificate verification entirely, which means anything that
can intercept the connection can also read the credential you send over it.

## Network security

### The MCP port binds to loopback

Both Compose files publish the MCP port as:

```yaml
ports:
  - "127.0.0.1:8080:8080"
```

and deliberately **not** as:

```yaml
ports:
  - "8080:8080"
```

The second form binds `0.0.0.0`, which makes the port reachable from every network your machine is
attached to. That matters more here than for a typical dev service, because the MCP server has **no
authentication of its own**: it holds one Elasticsearch credential and will run any search, any
`_cat` call and any ES|QL query that credential permits, for anyone who can open a TCP connection to
it. On the standalone deployment that credential may reach a production cluster. Loopback binding
keeps the blast radius on your own machine, where the AI client also runs.

### Do not expose Elasticsearch :9200 to the Internet

Do not publish Elasticsearch's HTTP port to the public Internet — not for MCP, not for anything
else. An exposed `:9200` is one of the classic ways log data ends up leaked or wiped, and the
cluster bundled in this repository runs with `xpack.security.enabled: false`, meaning **anyone who
can reach it has unauthenticated read and write access to every index**.

For reference, [`ELK/compose.yaml`](../compose.yaml) publishes `9200` and `9300` on all interfaces.
That predates this change and is left as-is for backward compatibility, but treat it as a
development-only setting: run the stack on a workstation or an isolated host, not on anything with
a public IP. If you need to harden it, change those entries to `"127.0.0.1:9200:9200"` and
`"127.0.0.1:9300:9300"` — Kibana, Logstash and the MCP server all reach Elasticsearch over the
Compose network (`http://elasticsearch:9200`), not through the published port, so nothing in the
stack breaks.

### Reaching a remote Elasticsearch

In rough order of preference:

1. **Corporate / private network** — the MCP container reaches an internal hostname directly. Best
   when you already have one.
2. **VPN** — same thing, from outside the office. Connect the VPN first; the container uses the
   host's network stack for egress.
3. **SSH tunnel** — good when you have SSH access and nothing else. See below.
4. **HTTPS with proper authentication** — an Elasticsearch endpoint published over TLS behind a
   reverse proxy, with API-key authentication and IP allow-listing. Never plain HTTP, and never
   without authentication.

### SSH tunnel example

Forward the remote Elasticsearch port to your workstation:

```bash
ssh -N -L 9200:localhost:9200 user@elk-server
```

Leave that running, then point the MCP server at the local end of the tunnel:

```dotenv
ES_URL=http://localhost:9200
```

#### The `host.docker.internal` consideration

The tunnel listens on your **host's** loopback interface. Inside the MCP container, `localhost` is
the container's own loopback, where nothing is listening — so a naive `http://localhost:9200` would
fail.

The Elasticsearch MCP server handles this: when it runs in container mode (the official image sets
`CONTAINER_MODE=true`), it rewrites a `localhost` host in `ES_URL` to `host.docker.internal`, or
`host.containers.internal` under Podman, if that name resolves. On Docker Desktop for macOS and
Windows it already does. **On Linux it does not exist by default**, so
[`compose.yaml`](compose.yaml) adds it:

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

`host-gateway` is a Docker-provided alias for the host as seen from the container. The mapping is
harmless on Docker Desktop, where the name already resolves, so the same file works everywhere.

One catch: `ssh -L 9200:localhost:9200` binds the tunnel to `127.0.0.1` on your host, which the
container cannot reach even via `host.docker.internal` on some Docker Desktop configurations. If the
container cannot connect, bind the tunnel to all interfaces instead and rely on your host firewall:

```bash
ssh -N -L 0.0.0.0:9200:localhost:9200 user@elk-server
```

Alternatively, skip the rewrite entirely and name the gateway explicitly:

```dotenv
ES_URL=http://host.docker.internal:9200
```

## Elasticsearch permissions

This applies to the **standalone** deployment, or to an integrated deployment where you have enabled
security. The bundled cluster has security disabled and has no roles or API keys at all.

Give the MCP server a dedicated, read-only identity scoped to the CDN log indices. Do not use
`superuser`, and do not grant `manage` or `all`.

### Minimum privileges

The five tools the server exposes map onto these Elasticsearch APIs and privileges:

| MCP tool | Elasticsearch API | Required privileges |
|---|---|---|
| `list_indices` | `GET /_cat/indices` | cluster `monitor`, index `monitor` |
| `get_shards` | `GET /_cat/shards` | cluster `monitor`, index `monitor` |
| `get_mappings` | `GET /<index>/_mapping` | index `view_index_metadata` |
| `search` | `POST /<index>/_search` | index `read` |
| `esql` | `POST /_query` | index `read` |

The cluster-level `monitor` privilege is the one extra beyond pure index reads, and it is only there
because `_cat/indices` and `_cat/shards` are cluster APIs. It is read-only: it permits cluster health
and stats calls, and nothing that changes state. If you do not need index and shard listing, drop it
and the `monitor` index privilege — `search` and `esql` work without them, though the agent then has
to be told which index to query instead of discovering it.

Nothing in this set allows document writes, deletes, index deletion, mapping changes or cluster
configuration changes. `read`, `view_index_metadata` and `monitor` are all read-only privileges.

### Example role

Run in Kibana **Dev Tools**:

```json
PUT _security/role/aem_cdn_logs_mcp_reader
{
  "cluster": ["monitor"],
  "indices": [
    {
      "names": ["aem-cdn-logs*"],
      "privileges": ["read", "view_index_metadata", "monitor"]
    }
  ]
}
```

`aem-cdn-logs` is the index name this repository's Logstash pipeline writes to, see
[`logstash/pipeline/logstash-cdn-logs.config`](../logstash/pipeline/logstash-cdn-logs.config). The
trailing `*` covers rollover or date-suffixed variants if you have customised the pipeline. Narrow
the pattern further if your cluster holds other people's data.

### Example API key

```json
POST _security/api_key
{
  "name": "aem-cdn-logs-mcp",
  "role_descriptors": {
    "aem_cdn_logs_mcp_reader": {
      "cluster": ["monitor"],
      "indices": [
        {
          "names": ["aem-cdn-logs*"],
          "privileges": ["read", "view_index_metadata", "monitor"]
        }
      ]
    }
  },
  "expiration": "90d"
}
```

The response contains an `encoded` field. That base64 value is what goes into `ES_API_KEY`:

```dotenv
ES_API_KEY=<paste the "encoded" value here>
```

Use `encoded`, not `id` or `api_key` — the MCP server sends it verbatim as an
`Authorization: ApiKey <value>` header.

An API key can never exceed the privileges of the user who created it, so create it as a user that
already has limited rights rather than as an administrator. Set an `expiration` and rotate on
schedule; revoke with `DELETE _security/api_key` and the key `id`.

### Verifying the key is actually read-only

```bash
# Should succeed
curl -H "Authorization: ApiKey $ES_API_KEY" "https://elk.example.com:9200/aem-cdn-logs/_count"

# Should fail with 403 security_exception
curl -X DELETE -H "Authorization: ApiKey $ES_API_KEY" "https://elk.example.com:9200/aem-cdn-logs"
```

## Health and verification

Start the service, integrated or standalone:

```bash
docker compose --profile mcp up -d    # integrated, from ELK/
docker compose up -d                  # standalone, from ELK/mcp/
```

Check container state. The service reports `healthy` once it is listening:

```bash
docker compose ps
```

Follow the logs. A successful start logs the version and the bind address. The upstream server is in
maintenance mode and prints a deprecation notice on every start; it is expected and harmless:

```bash
docker compose logs -f aem-cdn-logs-mcp
```

```text
INFO elasticsearch_core_mcp_server: Elasticsearch MCP server, version 0.4.6
WARN elasticsearch_core_mcp_server: DEPRECATION NOTICE: This MCP server is deprecated ...
INFO elasticsearch_core_mcp_server: Starting http server at address 0.0.0.0:8080
```

Test the health endpoint from the host. It returns `Ready`:

```bash
curl http://localhost:8080/ping
```

The container healthcheck only verifies that the port is listening — the official image is a minimal
Wolfi base with no `curl` or `wget`, so it uses busybox `netstat`. `curl .../ping` from the host is
the real end-to-end check.

### Verify MCP can reach Elasticsearch

`/ping` says the MCP server is up; it says nothing about Elasticsearch, because the server connects
lazily on the first tool call. To check the path end to end, call a tool over the MCP endpoint:

```bash
curl -s -X POST http://localhost:8080/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call",
       "params":{"name":"list_indices","arguments":{"index_pattern":"aem-cdn-*"}}}'
```

A working setup returns your index and its document count:

```text
data: {"jsonrpc":"2.0","id":1,"result":{"content":[{"type":"text","text":"Found 1 indices:"},
{"type":"text","text":"[{\"index\":\"aem-cdn-logs\",\"status\":\"open\",\"docs.count\":300}]"}],"isError":false}}
```

If it returns an error instead, the message names the cause — connection refused, certificate
problem, or a `security_exception` from a credential without the right privileges.

The simplest check that does not involve MCP at all, for the integrated deployment:

```bash
docker compose exec aem-cdn-logs-mcp sh -c 'echo ok'   # container is alive
curl "http://localhost:9200/aem-cdn-logs/_count"         # Elasticsearch has data
```

## What works against Elasticsearch 8.10.1

This repository pins Elasticsearch, Logstash and Kibana to `8.10.1`, and this change does not
upgrade them. Against that version, using the `aem-cdn-logs` index produced by this repository's
Logstash pipeline:

| Tool | Status against ES 8.10.1 |
|---|---|
| `list_indices` | Works |
| `get_shards` | Works |
| `search` (query DSL, including aggregations) | Works — this is the tool the examples rely on |
| `get_mappings` | **Fails** — see below |
| `esql` | **Fails** — ES\|QL was introduced in Elasticsearch 8.11 |

Two limitations are worth knowing before you start:

**`esql` returns `405 Method Not Allowed`.** The `_query` endpoint does not exist before
Elasticsearch 8.11 ([ES|QL was released as a technical preview in
8.11](https://www.elastic.co/blog/whats-new-elasticsearch-platform-8-11-0) and went GA in 8.14). Ask
agents for Query DSL rather than ES|QL on this stack. Everything in [CLIENTS.md](CLIENTS.md) is
written against the `search` tool for that reason.

**`get_mappings` returns `error decoding response body`.** Logstash creates the object fields `event`
and `log` in the index, and MCP server 0.4.6 cannot deserialize mapping entries that have no
top-level `type` key, which is exactly what an object field looks like. This is a limitation of the
MCP server's response parsing, not of Elasticsearch — `GET aem-cdn-logs/_mapping` works fine in Dev
Tools. The practical workaround is that agents do not need it: [CLIENTS.md](CLIENTS.md) contains a
field reference you can paste into a prompt, and an agent can also infer fields from a single
`search` hit.

## Troubleshooting

**`docker compose up -d` does not start the MCP container.** That is the intended behaviour — the
service is behind the `mcp` profile. Use `docker compose --profile mcp up -d`.

**`docker compose up` in `ELK/mcp` fails with an env file error.** Copy the template first:
`cp .env.example .env`.

**Port 8080 already in use.** Set `MCP_PORT` to a free port in `.env` (standalone) or `ELK/.env`
(integrated), then update your AI client configuration to match.

**The agent reports the server has no tools, or cannot connect.** Check the URL includes the `/mcp`
path — `http://localhost:8080/mcp`, not `http://localhost:8080`. Then confirm
`curl http://localhost:8080/ping` returns `Ready`.

**Tool calls fail with a connection error.** For the integrated deployment, confirm Elasticsearch is
up (`docker compose ps`) and that `ES_URL` is `http://elasticsearch:9200` — the Compose service name,
not `localhost`. For the standalone deployment, confirm the tunnel or VPN is up and read the
[`host.docker.internal` notes](#the-hostdockerinternal-consideration).

**Tool calls fail with `security_exception`.** The credential lacks a privilege. Compare it against
the [minimum privileges](#minimum-privileges) table.

**`esql` fails with 405, or `get_mappings` fails with a decode error.** Expected on Elasticsearch
8.10.1, see [What works against Elasticsearch 8.10.1](#what-works-against-elasticsearch-8101).
