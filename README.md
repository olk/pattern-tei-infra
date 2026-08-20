# pattern-tei-infra

Shared TEI (Text Embeddings Inference) sidecar stack for the
[architecture-pattern-mcp](https://github.com/olk/architecture-pattern-mcp) and [agent-pattern-mcp](https://github.com/olk/agent-pattern-mcp) MCP servers.

## What this does

Runs a single pair of TEI containers — `pattern-tei-embed` (embedder) and
`pattern-tei-rerank` (reranker) — on a shared Docker network `pattern-tei-shared`.
Both MCP **systemd** stacks join this network and reach the sidecars by DNS
name, so only one copy of each model lives in memory.

**Dev** compose files (`docker/docker-compose.yml` in each repo) keep their
own TEI sidecars with unique container names — they do **not** depend on this
stack.

## Prerequisites

- Docker Engine 24+ with Compose plugin v2.20+ (`docker compose version`)
- TEI images built locally or pulled from a registry:
  ```bash
  # From either repo `architecture-pattern-mcp` or `agent-pattern-mcp`.
  make docker-build-tei
  # Produces: pattern-tei-embed:latest, pattern-tei-rerank:latest
  ```
  Or for registry-based deploys:
  ```bash
  TEI_IMAGE=olkowa/pattern-tei-embed:0.1.0 \
  TEI_RERANK_IMAGE=olkowa/pattern-tei-rerank:0.1.0 \
    docker compose -f docker-compose.yml up -d
  ```

## Start / stop

```bash
# Via systemd (recommended — auto-starts at boot, orders MCP stacks after it):
sudo systemctl enable --now pattern-tei-infra.service

# Via docker compose directly:
docker compose -f docker-compose.yml up -d
docker compose -f docker-compose.yml down
```

## Teardown rule

**Stop MCP stacks before stopping this stack.** Removing `tei-shared` while
MCP containers are attached breaks their TEI connectivity:

```bash
sudo systemctl stop architecture-pattern-mcp agent-pattern-mcp
sudo systemctl stop pattern-tei-infra
```

## Verification

```bash
docker network inspect tei-shared          # should list 2 TEI containers
docker exec architecture-pattern-mcp curl -fsS http://pattern-tei-embed:8080/health
docker exec architecture-pattern-mcp curl -fsS http://pattern-tei-rerank:8080/health
```

## Files

| File | Purpose |
|---|---|
| `docker-compose.yml` | TEI sidecar definitions + `tei-shared` network |
| `pattern-tei-infra.service` | systemd unit template (install to `/etc/systemd/system/`) |
