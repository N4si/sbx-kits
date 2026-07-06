# yugabyte-agent

Docker Sandbox Kit for building stateful AI agents backed by YugabyteDB.

## Usage

```bash
sbx run --kit "git+https://github.com/docker/sbx-kits-contrib.git#dir=yugabyte-agent" claude
```

## Prerequisites

Set these on your host before creating the sandbox:

```bash
export ANTHROPIC_API_KEY=sk-ant-...
export YB_CONNECTION_STRING="postgresql://yugabyte:password@your-host:5433/yugabyte"
```

Your Anthropic API key is injected by the host-side proxy — it never enters the sandbox.

## What's Inside

- `agents/trend_analyzer.py` — financial trend analysis using Claude
- `agents/compliance_agent.py` — regulatory compliance audit (MAS, EU AI Act)
- `utils/db.py` — YugabyteDB connection helper
- `demo.py` — runs both agents and writes results to YugabyteDB

## Run the demo

```bash
# Runs automatically on startup, or manually inside the sandbox:
python demo.py
```
