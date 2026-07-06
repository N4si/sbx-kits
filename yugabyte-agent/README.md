# yugabyte-agent

A Docker Sandbox Kit for building stateful AI agents backed by YugabyteDB.

## What it is

This kit provisions an isolated sandbox environment with everything needed to build and run AI agents that persist state to a YugabyteDB database. The sandbox includes the Anthropic SDK for calling Claude, LangGraph for orchestrating multi-step agent workflows, and psycopg3 for connecting to any PostgreSQL-compatible YugabyteDB instance.

Two working demo agents are included out of the box:

- **trend_analyzer** — calls Claude to analyze financial market trends for a given symbol and timeframe, then writes the findings to a `agent_results` table in YugabyteDB
- **compliance_agent** — audits an entity against a regulatory framework (MAS, EU AI Act, etc.) and persists the audit report to the same table

## Why you need it

Building AI agents that actually work in production requires two things most demos skip: durable state and reproducible environments.

Without durable state, every agent run starts from scratch. There is no memory of prior analysis, no audit trail, no ability to replay a decision. YugabyteDB solves this — it is PostgreSQL-compatible, horizontally scalable, and supports JSONB for storing unstructured agent outputs alongside structured metadata.

Without a reproducible environment, every developer on a team sets up their own version of the stack differently. This kit packages the entire setup — dependencies, network policy, credential injection, file structure — into a single declarative artifact. One command gives any developer the same environment.

## Prerequisites

- Docker account with sbx CLI installed
- Anthropic API key (get one at console.anthropic.com)
- YugabyteDB instance — either YugabyteDB Managed (cloud.yugabyte.com) or self-hosted

## Setup

**Step 1: Install sbx**
```bash
# macOS
brew install docker/tap/sbx

# Windows
winget install Docker.sbx
```

**Step 2: Log in**
```bash
sbx login
```

**Step 3: Register your Anthropic API key**
```bash
sbx secret set-custom -g \
  --host api.anthropic.com \
  --env ANTHROPIC_API_KEY \
  --placeholder "sk-ant-proxy-managed" \
  --value YOUR_ANTHROPIC_KEY
```

The key stays on your host. The proxy injects it on outbound requests to api.anthropic.com. The sandbox never sees the real value.

**Step 4: Register your YugabyteDB connection string**
```bash
sbx secret set-custom -g \
  --host your-cluster-host.aws.yugabyte.cloud \
  --env YB_CONNECTION_STRING \
  --placeholder "postgresql://admin:password@your-cluster-host.aws.yugabyte.cloud:5433/yugabyte?sslmode=require" \
  --value "postgresql://admin:password@your-cluster-host.aws.yugabyte.cloud:5433/yugabyte?sslmode=require"
```

Also add your cluster host to the spec's allowedDomains if using a custom endpoint (see Network Policy below).

**Step 5: Run the kit**
```bash
sbx run --kit "git+https://github.com/N4si/sbx-kits.git#dir=yugabyte-agent" yugabyte-agent
```

## Running the demo

Once inside the sandbox:

```bash
cd /home/agent/workspace
python3 demo.py
```

Expected output:
```
==================================================
  YugabyteDB + Claude SDK Demo
==================================================

Step 1: Trend Analysis
----------------------------------------
  Analyzing AAPL over 30d...
  Analysis complete

Step 2: Compliance Audit
----------------------------------------
  Auditing Demo Corp against MAS...
  Audit complete

Step 3: Persist to YugabyteDB
----------------------------------------
  Connected to YugabyteDB
  Results saved to agent_results table

==================================================
  Demo complete!
==================================================
```

Verify the data landed in YugabyteDB:
```sql
SELECT id, agent, created_at FROM agent_results;
```

## File structure inside the sandbox

```
/home/agent/workspace/
├── agents/
│   ├── trend_analyzer.py      # Calls Claude, returns market trend analysis
│   └── compliance_agent.py    # Calls Claude, returns regulatory audit report
├── utils/
│   └── db.py                  # YugabyteDB connection helper (psycopg3)
└── demo.py                    # Runs both agents and persists results to DB
```

## Writing your own agent

```python
# agents/my_agent.py
import anthropic
import sys
sys.path.insert(0, '/home/agent/workspace')
from utils.db import db

client = anthropic.Anthropic()

def run(query):
    message = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        messages=[{"role": "user", "content": query}]
    )
    result = message.content[0].text

    # Persist to YugabyteDB
    db.execute(
        "INSERT INTO agent_results (agent, result) VALUES (%s, %s)",
        ("my_agent", result)
    )
    return result

if __name__ == "__main__":
    print(run("Summarize the key risks in the current macroeconomic environment"))
```

Run it:
```bash
python3 agents/my_agent.py
```

## Querying YugabyteDB directly

```python
from utils.db import db

# Read
rows = db.execute(
    "SELECT * FROM agent_results WHERE agent = %s ORDER BY created_at DESC LIMIT 5",
    ("trend_analyzer",)
)

# Write
db.execute(
    "INSERT INTO events (type, payload) VALUES (%s, %s)",
    ("analysis_complete", '{"symbol": "AAPL", "signal": "bullish"}')
)
```

## How credential injection works

Neither the Anthropic API key nor the YugabyteDB password enters the sandbox VM. The sbx proxy sits between the sandbox and the internet. When the sandbox makes a request to `api.anthropic.com`, the proxy intercepts it and injects the real API key as a header before forwarding. The sandbox sees only the placeholder value `sk-ant-proxy-managed`.

The same applies to YugabyteDB: the connection string placeholder is set as an environment variable inside the sandbox, and the proxy rewrites outbound connections to the DB host with the real credentials.

## Network policy

The sandbox runs under a deny-all baseline. Only these domains are reachable:

| Domain | Purpose |
|---|---|
| api.anthropic.com | Claude API |
| cloud.yugabyte.com | YugabyteDB Managed console |
| docs.yugabyte.com | YugabyteDB documentation |
| *.aws.yugabyte.cloud | YugabyteDB Managed cluster endpoints |
| pypi.org, files.pythonhosted.org, pythonhosted.org | pip package installs |
| github.com, raw.githubusercontent.com | Source installs |
| archive.ubuntu.com, security.ubuntu.com, ports.ubuntu.com | apt packages |
| download.docker.com | Docker apt repository |

If your YugabyteDB cluster is on a custom domain, add it to `caps.network.allow` in `spec.yaml` and run `sbx kit validate yugabyte-agent/` before re-running.

## Troubleshooting

**DNS resolution fails for YugabyteDB host**
Add the exact cluster hostname (not just the wildcard) to `caps.network.allow` in `spec.yaml`. Wildcards handle HTTP traffic but the exact hostname is needed for DNS resolution inside the sandbox.

**Install fails with network error**
Run `sbx policy log <sandbox-name>` to see which domain was blocked. Add it to `caps.network.allow`.

**Anthropic returns 401**
The placeholder value `sk-ant-proxy-managed` must match exactly what you set with `--placeholder` in `sbx secret set-custom`. Check with `sbx secret ls`.

**YugabyteDB connection refused**
The demo skips DB writes gracefully if the connection fails. Check that `YB_CONNECTION_STRING` is set correctly via `echo $YB_CONNECTION_STRING` inside the sandbox. The cluster host must be in the network allowlist.

**sbx command not found**
Install: `brew install docker/tap/sbx` (macOS) or `winget install Docker.sbx` (Windows). Minimum version: 0.28.0.