# arrow-kanban

[![Crates.io](https://img.shields.io/crates/v/arrow-kanban.svg)](https://crates.io/crates/arrow-kanban)
[![Documentation](https://docs.rs/arrow-kanban/badge.svg)](https://docs.rs/arrow-kanban)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Rust](https://img.shields.io/badge/rust-1.85%2B-orange.svg)](https://www.rust-lang.org)

Arrow-native kanban with SHACL shapes, graph-native PRs, dual boards, and
NATS-powered multi-agent collaboration.

Everything lives in Apache Arrow RecordBatches — no files, no databases, no
ORM. Items, runs, relations, proposals, and comments are columns in Arrow
tables persisted to Parquet.

## Quick Start

Add to your `Cargo.toml`:

```toml
[dependencies]
arrow-kanban = "0.14"
```

### Library usage

```rust
use arrow_kanban::crud::KanbanStore;
use arrow_kanban::item_type::ItemType;

// Create a store
let mut store = KanbanStore::new();

// Create an item
let id = store.create("Build the thing", ItemType::Expedition, None)?;

// Move through states
store.move_item(&id, "in_progress", None)?;
store.move_item(&id, "review", None)?;
store.move_item(&id, "done", Some("completed"))?;

// Query
let items = store.list_items(Some("in_progress"), None, None)?;
```

### CLI usage

```bash
# Install
cargo install arrow-kanban

# Local mode (Parquet files in .nusy-kanban/)
nk create expedition "Build the thing"
nk move EX-3001 in_progress
nk board
nk show EX-3001

# Server mode (NATS — single-writer, multi-agent)
nk --server nats://192.168.8.110:4222 board
```

## Dual Boards

Two boards for different workflows, each with dedicated item types:

### Development board

| Type | Prefix | Use case |
|------|--------|----------|
| **Expedition** | EX- | Multi-phase feature work |
| **Chore** | CH- | Routine maintenance |
| **Voyage** | VY- | 3+ related expeditions |
| **Hazard** | HZ- | Risk / tech debt |
| **Signal** | SG- | Observation worth tracking |
| **Feature** | FT- | User-facing capability |

Default lifecycle: `backlog → in_progress → review → done`

### Research board (HDD)

| Type | Prefix | Use case |
|------|--------|----------|
| **Paper** | PAPER- | Research publication |
| **Hypothesis** | H- | Testable claim |
| **Experiment** | EXPR- | One-shot validation run |
| **Measure** | M- | Ongoing metric |
| **Idea** | IDEA- | Captured thought |
| **Literature** | LIT- | External reference |

Research types have specialized lifecycles (e.g., hypotheses are `draft → active → retired`, never "complete").

## CLI Commands

| Command | Description |
|---------|-------------|
| `create <type> "title"` | Create item (auto-allocates ID) |
| `move <id> <status>` | Transition status (with WIP limit checks) |
| `update <id> --title/--priority/--assign/--tags/--body-file` | Update fields |
| `comment <id> "text"` | Add threaded comment |
| `show <id>` | Full item detail with body and comments |
| `list [--status X] [--board Y] [--type Z]` | Filter and list items |
| `board` | Column view of a board |
| `query "natural language"` | Semantic search across items |
| `stats` | Velocity, throughput, burndown |
| `roadmap` | Voyage-grouped dependency-ordered view |
| `critical-path` | Dependency chain analysis |
| `worklist [--agents "A,B,C"]` | Per-agent work assignments |
| `blocked` | Items with unresolved dependencies |
| `history` | Recently completed items |
| `export [--format html\|json\|md]` | Export board data |
| `validate` | SHACL conformance check |
| `rank` | Priority ranking |
| `migrate` | Import from markdown files |
| `pr <subcommand>` | Graph-native proposal lifecycle |
| `hdd <subcommand>` | Research board CRUD |

## Graph-Native PRs

Code review without GitHub. Proposals track safety gates and unresolved
comments in Arrow — `nk pr` is the only review surface.

```bash
nk pr create --title "EX-3001: Build the thing" --base main
nk pr list
nk pr view PROP-2001
nk pr diff PROP-2001
nk pr review PROP-2001 --approve --reviewer "Agent-B"
nk pr merge PROP-2001 --delete-branch
```

**Lifecycle:** `open → reviewing → approved → merged` (or `→ rejected → revised → reviewing`)

**Rules:** Author cannot self-approve. Unresolved comments block approval.

## NATS Server Mode

For multi-agent teams, run the companion
[arrow-kanban-server](https://github.com/hankh95/arrow-kanban-server) to get
single-writer semantics over NATS:

```bash
# All agents connect to the same server
alias nk='arrow-kanban --server nats://192.168.8.110:4222'
nk board          # reads from server
nk create ...     # writes go through server
```

Commands use NATS request-reply on `kanban.cmd.*` subjects. Mutations broadcast
durable events on `kanban.event.*` (JetStream).

## SHACL Shapes

Item types are defined as OWL classes with SHACL shape constraints:

- **Valid states** per type (e.g., hypotheses can't be "complete")
- **WIP limits** per status column
- **Body templates** generated from shape properties
- **Conformance validation** via `nk validate`

Shapes live in `ontology/shapes/` as Turtle files.

## Analytics

```bash
nk stats              # Board summary + velocity
nk roadmap            # Dependency-ordered campaign view
nk critical-path      # Longest dependency chain
nk worklist           # Agent workload distribution
nk blocked            # Items waiting on dependencies
nk history            # Recently completed with resolution
```

## Feature Flags

| Flag | Default | Description |
|------|---------|-------------|
| `client` | Yes | NATS client for `--server` mode |
| `build` | Yes | Graph-native build system integration |

```toml
# Minimal (no NATS, no build)
arrow-kanban = { version = "0.14", default-features = false }

# With NATS client
arrow-kanban = { version = "0.14", features = ["client"] }
```

## Architecture

```
arrow-kanban/
├── src/
│   ├── crud.rs          # Create/read/update/delete (Arrow RecordBatches)
│   ├── schema.rs        # Arrow table schemas (Items, Runs, Comments)
│   ├── state_machine.rs # Status transitions, WIP limits
│   ├── item_type.rs     # 12 item types across dual boards
│   ├── display.rs       # Terminal formatting (board, table, detail)
│   ├── export.rs        # HTML, JSON, Markdown, burndown SVG
│   ├── query.rs         # Natural language query + SPARQL
│   ├── templates.rs     # SHACL shape-driven template generation
│   ├── hooks.rs         # Config-driven event hooks
│   ├── comments.rs      # Threaded, resolvable comments
│   ├── stats.rs         # Velocity, burndown, agent throughput
│   ├── validate.rs      # SHACL conformance checking
│   ├── persist.rs       # Parquet persistence (WAL + atomic rename)
│   ├── client.rs        # NATS client for remote server mode
│   ├── relations.rs     # Item relationships (related, blocked-by, parent)
│   └── ... (31 modules, 400+ tests)
├── ontology/
│   ├── kanban.ttl       # OWL class hierarchy
│   └── shapes/          # SHACL shapes (dev + research)
└── tests/
```

## Persistence

Items are Arrow RecordBatches persisted to Parquet via WAL + atomic rename:

```
.nusy-kanban/
  items.parquet        # All kanban items
  runs.parquet         # Status transition history
  relations.parquet    # Item relationships
  proposals.parquet    # Graph-native proposals
  comments.parquet     # Review comments
```

Every mutation is durably written before the response is sent.

## License

MIT — Copyright (c) Hank Head / Congruent Systems PBC
