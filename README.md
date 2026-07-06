# Docker Sandbox Kits

Community sandbox kits for Docker Sandboxes (`sbx`).

## Kits

### yugabyte-agent
Production-ready sandbox for building stateful AI agents backed by YugabyteDB.
Includes Anthropic SDK, LangGraph, and psycopg for distributed SQL.

```bash
sbx run --kit ./yugabyte-agent claude
```

### meko-agent
Sandbox for building multi-agent systems with Meko cross-agent memory.
Includes Anthropic SDK, LangGraph, and a Meko MCP client for memory handoff between agents.

```bash
sbx run --kit ./meko-agent claude
```

## Usage

Set credentials on your host before creating a sandbox:

```bash
# For yugabyte-agent
export ANTHROPIC_API_KEY=sk-ant-...
export YB_CONNECTION_STRING="postgresql://yugabyte:password@host:5433/yugabyte"

# For meko-agent
export ANTHROPIC_API_KEY=sk-ant-...
export MEKO_API_KEY=your-meko-key
export MEKO_DATAPACK_ID=your-datapack-uuid
```

## Author
Nasi Chaudhari — Docker Captain, Developer Engagement Manager at YugabyteDB
