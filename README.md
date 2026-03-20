# arrow-kanban

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

**Arrow-native kanban with a nautical soul** — track work with graph-native PRs,
SHACL shapes, dual boards, and NATS-powered multi-agent collaboration.

## Status: Pre-release

This crate is extracted from the NuSy project and requires additional work
before it compiles standalone:

- [ ] Feature-gate PR workflow (depends on unpublished `nusy-graph-review`)
- [ ] Feature-gate CI runner (depends on unpublished `nusy-conductor`)
- [ ] Feature-gate embeddings (depends on `nusy-graph-query` — published on GitHub but not crates.io)
- [ ] Refactor `main.rs` for conditional compilation of feature-gated modules

Core engine (CRUD, state machine, schemas, display, export, templates, hooks,
comments, stats) compiles without these features.

## Architecture

```
arrow-kanban/
├── src/
│   ├── crud.rs          # Create/read/update/delete items (Arrow RecordBatches)
│   ├── schema.rs        # Arrow table schemas (Items, Runs, Comments, ExperimentRuns)
│   ├── state_machine.rs # Status transitions, WIP limits
│   ├── display.rs       # Terminal formatting
│   ├── export.rs        # HTML, JSON, Markdown export + burndown SVG
│   ├── query.rs         # Natural language query decomposition + SPARQL
│   ├── templates.rs     # SHACL shape-driven template generation
│   ├── hooks.rs         # Config-driven event hooks
│   ├── comments.rs      # Threaded, resolvable item comments
│   ├── stats.rs         # Velocity, burndown, agent throughput
│   ├── validate.rs      # SHACL conformance checking
│   ├── persist.rs       # Parquet persistence (WAL + atomic rename)
│   ├── client.rs        # NATS client for remote server mode
│   ├── mcp_server.rs    # MCP stdio server (7 tools for Claude/AI agents)
│   └── ... (31 modules total)
├── ontology/
│   ├── kanban.ttl       # OWL class hierarchy (12 item types)
│   └── shapes/          # SHACL shapes (dev + research + workflow)
├── skills/              # Claude Code workflow skills
└── tests/               # 400+ tests
```

## Features

- **12 item types** across dev + research dual boards
- **SHACL shapes** define valid states, WIP limits, and body templates
- **Graph-native PRs** with safety gates and cross-agent review
- **MCP server** for AI agent tool use (7 structured tools)
- **NATS server** for multi-agent single-writer coordination
- **Semantic search** with embedding providers (hash, Ollama, subprocess)
- **Analytics** — velocity, burndown, agent throughput, critical path

## Dependencies

| Crate | crates.io | Required |
|-------|-----------|----------|
| `arrow` 55 | Yes | Core |
| `parquet` 55 | Yes | Core |
| `arrow-graph-git` 0.1 | Yes | Persistence (feature: `persistence`) |
| `nusy-graph-query` | GitHub only | Embeddings (feature: `embeddings`) |
| `clap` 4 | Yes | CLI |
| `async-nats` 0.38 | Yes | NATS client (feature: `client`) |

## License

MIT
