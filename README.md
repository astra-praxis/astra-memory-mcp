# Astra Memory

**User-owned, locally encrypted long-term memory for AI agents.** Ships as an
[MCP](https://modelcontextprotocol.io) stdio server — plug it into
[DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness), Claude Code,
Cursor, or any MCP-capable agent.

Astra Memory is not chat history and not a vector-database wrapper. It forms
memory about the user, the project and the work over long-term interaction —
preferences, facts, decisions, experiences, tasks — and **actively recalls the
right thing when the current task needs it**. Everything lives in an encrypted
SQLite database on your machine. No server. No cloud. No telemetry.

> 换模型、换应用，记忆体继续陪着你。Works with any model; memory stays yours.

## Highlights

- **Local-first & encrypted** — XChaCha20-Poly1305 payload encryption; the key
  never leaves your machine (see [docs/security.md](docs/security.md))
- **Recall, not retrieval** — formation / recall / lifecycle engine, not RAG:
  memories confirm, reinforce, decay, merge and supersede over time
- **Duplicate-free writes** — identical content is auto-merged by the engine
- **One binary, zero services** — stdio MCP server launched and stopped by your
  harness; no ports, no daemons, no accounts

## Install

Download the archive for your platform from
[Releases](https://github.com/astra-praxis/astra-memory-mcp/releases), unpack it,
and put `astra-memory-mcp` on your `PATH` (or note the absolute path).

| File | Platform |
|---|---|
| `astra-memory-mcp_<ver>_aarch64-apple-darwin.tar.gz` | macOS (Apple Silicon) |
| `astra-memory-mcp_<ver>_x86_64-apple-darwin.tar.gz` | macOS (Intel) |
| `astra-memory-mcp_<ver>_x86_64-unknown-linux-gnu.tar.gz` | Linux x86_64 |
| `astra-memory-mcp_<ver>_x86_64-pc-windows-msvc.zip` | Windows x64 |

## Tools

| Tool | Semantics |
|---|---|
| `astra_remember` | Explicit write ("remember that…"); duplicates auto-merge |
| `astra_recall` | Recall relevant memories for the current task context |
| `astra_forget` | Hard-delete one memory by id |
| `astra_get_memory` | Fetch one memory by id |

Formation internals are **never** exposed to the model — the observation
channel belongs to the host, not the agent.

## Connect to DeepSeek Harness

Save as `astra-memory.cordis.yml` (adjust the command path):

```yaml
- insert:
    - id: memory-astra
      name: '@deepseek-ai/dsh-mcp-client'
      config:
        serverName: astra-memory
        transport: stdio
        command: /usr/local/bin/astra-memory-mcp
        args: []
        env:
          ASTRA_MEMORY_DIR: /Users/you/.astra-memory
        cwd: !!js process.cwd()
```

```sh
dsh web --patch "$PWD/astra-memory.cordis.yml"
```

Other MCP clients: register the same stdio command with your client's MCP
configuration.

## Configuration

| Variable | Meaning | Default |
|---|---|---|
| `ASTRA_MEMORY_DIR` | Data directory (`memory-cme.sqlite` + `memory.key`) | `~/.astra-memory` |
| `ASTRA_MEMORY_KEY_HEX` | 64-char hex master key (takes precedence over the key file) | auto-generated, file mode 0600 |
| `ASTRA_MEMORY_PROJECT` | Default project scope; per-tool `project_id` wins | global |

## Verify (write → fresh session → recall)

1. Session A: "Remember that my validation drink is lapsang-\<suffix>." —
   the model should call `astra_remember`.
2. Open a **new** session (same host): "What is my validation drink? Check
   memory." — `astra_recall` should hit.
3. Same session: "Suggest one drink for the meeting using that preference."

## Known limitations (early preview)

- Without an embedding endpoint, recall relies on direct matching plus the
  memory relation graph; semantic generalization is on the roadmap
- `update` (edit / confirm-reinforce) is not exposed yet; `astra_forget` +
  `astra_remember` covers the same intent

## License

Proprietary — see [EULA.md](EULA.md). The memory database is yours; the
software is licensed, not sold. Data escape hatch: the storage format is
documented in [docs/protocol.md](docs/protocol.md) so your memory is never
hostage.
