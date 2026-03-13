# Seeds - Design Overview

Git-native issue tracker for AI agent workflows. Part of the os-eco (overstory/mulch) ecosystem.

## Entry Points

- **CLI binary:** `sd` (via `src/index.ts`) — Commander-based command router
- **Programmatic:** Overstory calls `sd` via `Bun.spawn(["sd", ...])` with `--json` flag

## Data Flow

```
User/Agent ──> sd <command> ──> src/index.ts (router)
                                    │
                                    ▼
                              src/commands/*.ts
                                    │
                        ┌───────────┼───────────┐
                        ▼           ▼           ▼
                   src/store.ts  src/config.ts  src/id.ts
                        │           │
                   ┌────┴────┐      ▼
                   ▼         ▼   config.yaml
              issues.jsonl  templates.jsonl
              (lock → read → mutate → atomic write → unlock)
                        │
                        ▼
                   src/output.ts ──> stdout (JSON or human-readable)
```

## Key Abstractions

| Module | Responsibility |
|--------|---------------|
| `store.ts` | JSONL read/write with advisory locking and atomic writes |
| `types.ts` | Issue, Template, Config interfaces and constants |
| `id.ts` | Collision-checked `{project}-{4hex}` ID generation |
| `config.ts` | Load/save `.seeds/config.yaml` |
| `yaml.ts` | Minimal flat key-value YAML parser (~50 LOC) |
| `output.ts` | Dual-mode output (JSON via `--json`, or ANSI human-readable) |
| `markers.ts` | Marker-delimited section helpers for `sd onboard` |

## Dependencies

- **Runtime:** chalk (colors), commander (CLI parsing)
- **Bun built-ins:** `Bun.file`, `Bun.write`, `node:fs` (locks), `node:crypto` (IDs)
- **Dev:** @types/bun, typescript, @biomejs/biome

## Concurrency Model

Designed for multi-agent concurrent access (multiple Claude agents in worktrees):

1. **Advisory file locks** — `O_CREAT | O_EXCL` on `.seeds/*.lock`, 30s stale threshold
2. **Atomic writes** — Write to temp file, rename over target (POSIX atomic)
3. **Dedup on read** — After `merge=union` git merges, last occurrence of duplicate IDs wins

## Exit Points

- **stdout** — Command results (JSON or human-readable)
- **stderr** — Errors
- **Exit codes** — 0 success, 1 error
- **File system** — `.seeds/` directory mutations
- **Git** — `sd sync` stages and commits `.seeds/` changes
