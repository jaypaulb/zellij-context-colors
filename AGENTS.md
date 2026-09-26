# zellij-context-colors — Agent Context

**READ THIS FIRST.** Orientation for anyone (human or agent) reviewing or working in this repo.

## What this is

A [Zellij](https://zellij.dev) plugin (Rust → WASM, headless — no visible pane) that tints each pane's background based on its **context**: the pair `{host}/{cwd}`. SSH into a remote host and the color flips; `cd` and it flips again; running any other foreground command in the same directory keeps the color. Colors are **deterministic and persistent** (same context ⇒ same color across sessions), in contrast to the random-per-pane approach of `karlbunch/zellij-colorful` that inspired it.

## Repo state (important for reviewers; as of 2026-09 — update when `feat/v0.1.0-implementation` merges)

- **`main` @ HEAD is docs-only.** The tracked tree is just `README.md`, `.gitignore`, and two design docs under `docs/superpowers/`. There is **no Rust source on `main`.**
- **The implementation lives on branch `feat/v0.1.0-implementation`**, checked out in an *excluded* git worktree (`worktree-feat+v0.1.0-implementation/`, ignored via `.gitignore`'s `.claude/` / worktree conventions). Reviewing `main`'s HEAD means reviewing the design/spec, not code.
- `Cargo.lock`, `/target/`, `*.wasm` are gitignored — build artifacts are not tracked.

## Where things live

- **`README.md`** — user-facing overview, config format, how-it-works summary.
- **`docs/superpowers/specs/2026-05-19-zellij-context-colors-design.md`** — the authoritative design: concept, architecture, data model, edge cases, out-of-scope, open questions. Start here for intent.
- **`docs/superpowers/plans/2026-05-19-zellij-context-colors.md`** — the detailed implementation plan (~1500 lines).

## Core design invariants (from the spec)

- **Context key is `{host}/{cwd}`.** `host` = local hostname normally, or the SSH target when the foreground command is literally `ssh`. `cwd` uses `~` for `$HOME`.
- **Only `argv[0] == "ssh"` flips the host.** Every other foreground command (`vim`, `python`, `cargo`, `cly`, …) leaves host/cwd alone — the color must stay put.
- **On `ssh` entry:** stash local cwd, set host to parsed target (strip `user@`, skip options-with-values to find the target; `?` if none), reset cwd to `~`. **On `ssh` exit:** restore host to local hostname and restore the stashed local cwd.
- **Determinism + persistence.** Unknown keys get a color by stable-hashing the key into a curated ~16-color palette, then the entry is **written back** to `~/.config/zellij/context-colors.toml` so it is stable forever. Config is the source of truth for colors.
- **Colors are RGBA** parsed/serialized as `#RRGGBBAA` (trailing byte = alpha).
- **Config writes are atomic** (temp file + rename). Missing config ⇒ empty/create on first write. Unparseable config ⇒ log and treat as empty in memory, **do not overwrite** (let the user fix it).
- **Same key ⇒ same color across all panes**; a manual `zctx set '#AA3311FF'` override (a `CustomMessage` from a shell alias) updates every pane on that key.

## Out of scope for v1

Per-tab coloring, configurable command list (only `ssh` is special), wildcard/regex config keys, blended colors, cross-machine sync, GUI config editor, and non-`ssh` remote wrappers (`mosh`, `tmux ssh`).
