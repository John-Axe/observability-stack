# observability-stack

The free, local, no-recurring-cost observability layer for the
[security-engineering portfolio ecosystem](../portfolio-ecosystem/ARCHITECTURE.md).
Every producer repo writes a `findings.jsonl` in the shape defined by
[`finding-schema`](../finding-schema); this stack tails all of them and
renders them in Grafana.

```
producer repo's findings.jsonl  -->  Promtail  -->  Loki  -->  Grafana
```

## Services

| Service | Port | Purpose |
|---|---|---|
| Grafana | `localhost:3000` (anonymous viewer access, no login) | Dashboards |
| Loki | internal only (`loki:3100`) | Log/finding storage, queried by Grafana |
| Promtail | internal only | Tails every sibling repo's `findings.jsonl` |
| Postgres | `localhost:5432` (`findings`/`findings`) | Reserved for the Phase 2 findings-api — not wired to anything yet |
| MinIO | `localhost:9000` (API), `localhost:9001` (console) — `minioadmin`/`minioadmin` | S3-compatible object storage, replaces AWS S3 for local dev |
| LocalStack | `localhost:4566` | AWS API emulation (IAM, Lambda, EventBridge, S3, DynamoDB, SQS, SNS, CloudWatch Logs) for `iam-privesc-mapper` / `aws-auto-remediation` local testing |

Nothing here costs money and nothing needs a real AWS account. `docker
compose down -v` removes every container and volume and leaves no trace.

## Run it

```bash
docker compose up -d
```

Then open `http://localhost:3000` — the "Security Findings" folder has a
pre-provisioned **Security Findings Overview** dashboard (severity
breakdown, findings-over-time by source, category pie chart, raw log
stream). No login needed; anonymous viewer access is enabled for local
convenience only — **do not** expose port 3000 beyond localhost.

## How a repo plugs in

Promtail mounts the parent `GitHub/` folder read-only and globs
`/repos/*/findings.jsonl` — i.e. every sibling repo's own repo-root
`findings.jsonl`, one level deep, nothing else. There is no per-repo
compose config to write. A producer just needs to:

1. Write findings as JSON-lines matching `../finding-schema/schema/finding.schema.json`, one object per line, appended to `findings.jsonl` in its own repo root.
2. Add `findings.jsonl` to its own `.gitignore` — it's local runtime output, not something to commit.
3. That's it. Within a few seconds of a new line landing, it's queryable in Grafana.

See `dlp-secret-pii-scanner`'s `--emit-findings` flag and
`iam-privesc-mapper`'s `--emit-findings` flag for the first two real
producers.

## Why this instead of a managed stack

Managed Grafana Cloud / hosted Loki / AWS Managed Grafana all have a
recurring cost floor, however small, and this portfolio's constraint is
zero recurring cost. A local docker-compose stack exercises the exact same
Grafana/LogQL skills an interviewer cares about, with `docker compose down
-v` as the entire teardown story instead of a support ticket.

## Known limitation of this session

Docker isn't available in the WSL environment this was built in (Docker
Desktop's WSL integration wasn't enabled), so this stack is
syntax-validated but not yet run end-to-end. Bring it up with `docker
compose up -d` on your end and confirm the dashboard renders — if the Loki
schema or Promtail pipeline stages need adjusting for the exact image
versions pinned here, that's the first thing to check.
