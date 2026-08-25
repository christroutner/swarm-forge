# Plan: Option B — `SWARMFORGE_DIR` (external machinery, per-project config)

> **Status**: pending implementation — working document to resume later.
> **Context**: came out of the trial run with `saas-prototype` (see the fork README).
> The goal is that projects do not carry SwarmForge code, only their configuration.

## 1. Goal

The project stops carrying SwarmForge **code** (scripts) and **shared rules**
(articles, default roles). It only keeps its **own configuration** (`swarmforge.conf` +
`project.prompt` + overrides). The fork (or any shared location via `SWARMFORGE_DIR`)
is the sole source of the machinery. With `SWARMFORGE_DIR` unset, **everything still works
as today** (full backward compatibility).

## 2. Design: where each thing lives

```
SWARMFORGE_DIR (base, e.g. ~/.local/share/swarmforge = fork clone)
├── swarm/ + swarmforge/scripts/          ← launcher, daemon, helpers, adapters
├── swarmforge/roles/*.prompt             ← default roles
└── swarmforge/constitution/articles/     ← engineering, handoffs, workflow (shared)

PROJECT
└── swarmforge/                            ← "thin": ONLY per-project material
    ├── swarmforge.conf                    ← project roles + models
    └── constitution/articles/project.prompt (+ local-*.prompt, role overrides if any)
```

**Merge rule**: the project wins by name — if the project has
`roles/cleaner.prompt`, that takes precedence; otherwise the base one is used.

## 3. Code changes (file by file)

| File | Change | Why |
|---|---|---|
| `swarmforge.bb` → `context` | Add `:base-dir` = `(or (System/getenv "SWARMFORGE_DIR") (fs/path working-dir "swarmforge"))`; keep `:swarm-forge-dir` = `working-dir/swarmforge` (project config) | Shared base vs. per-project config |
| `swarmforge.bb` → `parse-config` | `roles-dir` = per-role lookup: `project/roles/<role>.prompt` if it exists, else `base/roles/<role>.prompt` | Role overrides |
| `swarmforge.bb` → new `sync-shared-config!` | For each worktree **and for master**: copy from `base` the shared articles and default roles into the destination `swarmforge/` **only if missing** (project ones win); scripts as today | The agent reads `swarmforge/constitution.prompt` relative to its cwd — after sync, the merged view is there |
| `swarmforge.bb` → `prepare-workspace!` / setup | When syncing on master (project root), add derived files (shared articles) to `.gitignore` so the project's git stays clean | Shared articles are generated at startup, not committed |
| `write-agent-instruction-file!` | **No changes** | Relative paths still work because sync merges the view into each worktree |
| `check-helper-scripts!` | **No changes** (validates `script-dir`, which points at the base) | — |
| `handoffd.bb`, helpers, adapters | **No changes** | Already resolve the project via git (`roles.tsv`) |
| `swarm` wrapper | Small install/docs tweak: when installed globally, it runs the base `swarmforge.sh`; the download block remains for first-time setup | Global install |

**Estimated total**: ~40-60 new/modified lines in `swarmforge.bb` + docs. Nothing else.

## 4. User setup (once)

```bash
# 1. Install the machinery once
git clone https://github.com/pablo-io/swarm-forge ~/.local/share/swarmforge
ln -s ~/.local/share/swarmforge/swarm ~/.local/bin/swarm
export SWARMFORGE_DIR=~/.local/share/swarmforge   # (in your .bashrc)

# 2. In any project: create ONLY the config
mkdir -p swarmforge/constitution/articles
# swarmforge.conf + project.prompt (+ local-* / overrides if applicable)

# 3. Run
cd /my/project && swarm
```

## 5. Test plan

1. **`bb test`** — the existing suite (24 tests) must pass with no semantic changes.
2. **Mode B**: minimal project with only conf + `project.prompt` → launch with `SWARMFORGE_DIR`
   → verify: worktrees with the merged view (shared articles + roles + scripts),
   functional master, clean startup.
3. **Handoff smoke**: a real end-to-end handoff (like the `saas-prototype` run)
   under mode B.
4. **Backward compatibility**: `saas-prototype` (with full `swarmforge/`) without the env var →
   must keep working the same.
5. **Sync**: change a script in the fork (e.g. a fix) → it is reflected without copying
   anything into the project.

## 6. Optional migration of `saas-prototype` (after validating B)

- Thin its `swarmforge/`: delete `scripts/`, `roles/`, and shared articles; leave
  `swarmforge.conf` + `project.prompt` (with the design rules).
- Re-copy updated prompts from the fork (`architect.prompt` with the written-report
  rule) — now via the base, not manual copy.

## 7. Risks / open decisions

- **Shared articles synced onto master** will be gitignored (derived) — if someone wants
  to version them explicitly, they can commit them (sync does not overwrite existing ones).
- **`roles.tsv` and state** stay under the project's `.swarmforge/` (gitignored) — unchanged.
- Agent instructions remain relative to the worktree — key to the design
  (zero protocol changes).

## 8. Suggested implementation order

1. `context` + `parse-config` (base dir + role lookup)
2. `sync-shared-config!` + gitignore for derived files
3. `bb test`
4. Mode B trial with a minimal project
5. Doc in the fork README ("SWARMFORGE_DIR mode" section)
