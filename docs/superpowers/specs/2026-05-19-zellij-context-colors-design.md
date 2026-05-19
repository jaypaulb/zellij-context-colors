# Zellij Context Colors — Design

**Date:** 2026-05-19
**Status:** Draft (in brainstorming)

## Problem

Zellij panes look identical regardless of what they are doing. When several panes are open — some local, some SSH'd into remote hosts, some in different project directories — there is no quick visual cue for which is which. `karlbunch/zellij-colorful` solves this by assigning each pane a random color, but the assignment is per-pane and ephemeral: the same project directory gets a different color every session, and an SSH session to the same host looks different every time.

We want colors that mean something: **the same context always produces the same color**, across sessions and across panes.

## Concept

Each pane has a **context key** of the form `{host}/{cwd}`:

- `host` is the machine the foreground process is talking to.
  - Local panes: the local machine's hostname (e.g. `halbuntu`).
  - Panes whose foreground command is `ssh <target>`: the SSH target hostname (e.g. `hal`).
- `cwd` is the current working directory, with `~` for the user's home.

Examples:

| Pane state                          | Context key             |
| ----------------------------------- | ----------------------- |
| Local shell at `~/Projects/foo`     | `halbuntu/~/Projects/foo` |
| Local shell at `~`                  | `halbuntu/~`              |
| `ssh hal` then `cd ~/Projects/foo`  | `hal/~/Projects/foo`      |
| `ssh hal` (home dir, no cwd info)   | `hal/~`                   |

Each context key maps to one RGBA color. The color is applied as the pane's background tint.

Only the `ssh` command flips the host portion of the key. Every other foreground command (`vim`, `python`, `cly`, …) leaves the host alone, so running `cly` in `~/Projects/foo` keeps the same color as the bare shell in that directory.

## User experience

### First time

Open a pane. The plugin sees a context key it does not recognise, picks a color deterministically from a curated palette by hashing the key, and applies it. The new entry is written to the config file so the color is stable from now on.

### Re-open the same context

Open another pane that lands in the same `{host}/{cwd}`. The plugin looks up the existing entry in the config and applies the same color. Both panes look the same.

### Manual override

Run, from inside the pane you want to recolor:

```sh
zctx set '#AA3311FF'
```

This is a shell alias that sends a `CustomMessage` to the plugin via `zellij action send-plugin-message`. The plugin records the new color under the pane's current context key, writes it to the config, and applies it immediately. All other panes already on that context key update too.

### SSH'ing into a host

Local shell in pane 3 is at `~/Projects/foo` → color is `halbuntu/~/Projects/foo`.
You run `ssh hal`. Foreground command changes → plugin flips host to `hal` **and stashes the local cwd (`~/Projects/foo`)**, resetting the active cwd to `~` since the remote cwd is unknown. Key becomes `hal/~`. If the remote shell emits OSC 7, `CwdChanged` fires with the remote path and the key updates to e.g. `hal/~/work/proj`. When `ssh` exits, the plugin flips host back to `halbuntu` **and restores the stashed local cwd (`~/Projects/foo`)**. The original color returns.

### Other commands

Running `cly`, `vim`, `python`, `cargo build`, etc. fires `CommandChanged` but the argv[0] is not `ssh`, so the plugin ignores it. The color stays put.

## Architecture

A single headless Zellij plugin (Rust → WASM, no visible pane), loaded at session startup.

```
┌────────────────────────────────────────────────────────────┐
│                  zellij-context-colors                     │
│                                                            │
│   ┌────────────┐    ┌──────────────┐    ┌──────────────┐   │
│   │ Event loop │ -> │ Per-pane     │ -> │ Color        │   │
│   │            │    │ state map    │    │ resolver     │   │
│   └────────────┘    └──────────────┘    └──────────────┘   │
│         ▲                                       │          │
│         │                                       ▼          │
│   Zellij events                          ┌──────────────┐  │
│   (CwdChanged,                           │ Config file  │  │
│    CommandChanged,                       │ (TOML)       │  │
│    PaneUpdate,                           └──────────────┘  │
│    CustomMessage)                                          │
└────────────────────────────────────────────────────────────┘
```

### Components

**Event loop.** Subscribes to:

- `PaneUpdate` — to learn about new panes and initialise their state.
- `CwdChanged(pane_id, path)` — update the `cwd` part of that pane's key.
- `CommandChanged(pane_id, argv)` — if `argv[0] == "ssh"`:
  - Walk argv from index 1 onwards. Skip options (any arg starting with `-`) and their values when the option is known to take one (`-J -p -i -l -F -L -R -D -W -o -b -c -e -m -O -Q -S -B -E -I`). The first remaining non-option argv element is the target. Strip any `user@` prefix.
  - Stash the current local cwd in `local_cwd`, set `host` to the parsed target, set `cwd` to `~`.
  - If parsing fails (no target found), set `host` to `?`.

  If `argv[0] != "ssh"` and the pane is currently in SSH mode (i.e. ssh just exited), restore `host` to the local hostname and `cwd` to the stashed `local_cwd`. If the pane was never in SSH mode, leave host and cwd alone — other commands do not change the key.
- `CustomMessage` — receive manual-override commands from the `zctx` shell alias.
- `PaneClosed` — drop the pane from the state map.

**Per-pane state map.** `HashMap<pane_id, PaneContext>` where `PaneContext` holds the current `host`, `cwd`, and last-applied color. When either `host` or `cwd` changes the key is rebuilt and the color is resolved.

**Color resolver.** Given a context key:

1. Look it up in the in-memory copy of the config. If present, return that color.
2. Otherwise, hash the key string (stable hash, e.g. FNV-1a or SipHash with a fixed seed) and index into a curated palette of ~16 RGB colors that have been chosen to remain readable behind common terminal text. Append `FF` for the alpha byte (fully opaque, matching the user-facing examples).
3. Persist the auto-assigned entry to the config so it stays stable.

**Config persistence.** TOML at `~/.config/zellij/context-colors.toml`:

```toml
[colors]
"hal/~"              = "#123456FF"
"hal/~/Projects/foo" = "#AA3311FF"
"halbuntu/~"         = "#226644FF"
```

Read on plugin startup; written on every change (atomic write via temp file + rename). Unrecognised or unparseable entries are logged and skipped, not fatal.

### Applying colors

The plugin uses Zellij's `ChangeApplicationState` permission and the same pane-background-color API that `zellij-colorful` uses, called whenever a pane's color resolves to a different value than its last-applied one.

## Data model

```rust
struct ContextKey { host: String, cwd: String } // serialises as "host/cwd"

struct PaneContext {
    host: String,               // local hostname OR ssh target
    cwd:  String,               // "~" or absolute path with $HOME tilde-replaced
    in_ssh: bool,               // whether foreground command is ssh
    local_cwd: Option<String>,  // stashed local cwd while in_ssh; restored on ssh exit
    last_color: Option<Rgba>,
}

struct Config {
    colors: HashMap<String, Rgba>,  // key string -> color
}
```

Color is stored as 4-byte RGBA. Parsed from `#RRGGBBAA` strings; serialised the same way.

## Edge cases

- **`ssh` with no argument or just options.** Host becomes `?`, key becomes `?/~`, gets its own color. Better than crashing or guessing.
- **`ssh user@host`.** The `user@` prefix is stripped; only `host` is used as the key.
- **`ssh -J jumphost target`, `ssh -p 2222 host`, etc.** Parsed per the rule in the `CommandChanged` handler — options with values are skipped, first remaining bare arg is the target.
- **`mosh`, `tmux ssh`, wrapper scripts.** Out of scope for v1. Only literal `argv[0] == "ssh"` is treated as SSH.
- **`zctx set` issued while in SSH mode.** Override applies to the current `host/cwd` key (e.g. `hal/~/work`). Returning to local context shows the local color again, as expected.
- **Tab/window moves.** Pane IDs are assumed to be stable across moves; if they are not, the state map will lose track and the pane will re-resolve from scratch on its next event. To verify during implementation.
- **Config file missing.** Treat as empty; create on first write.
- **Config file unparseable.** Log error, treat as empty in memory, do not overwrite (so the user can fix it manually).
- **Same context key, multiple panes.** All panes with the same key get the same color. Manual override on one updates all of them.
- **Plugin restart.** State map is rebuilt from `PaneUpdate` on startup. Config is the source of truth for colors. Note: a pane that was in SSH mode when the plugin restarted will lose its stashed `local_cwd` — it will resolve based on the current SSH key, which is correct, and will repopulate `local_cwd` on the next ssh entry.

## Out of scope (for v1)

- Per-tab or per-tab-bar coloring (we are doing pane background only).
- Configurable command list (only `ssh` is special).
- Wildcard or regex entries in the config (only literal keys).
- Layered/blended colors (one color per key).
- Sync across machines (just a local file).
- A GUI/TUI for managing the config (TOML by hand or `zctx` is enough).

## Open questions for implementation

- Exact name and signature of the Zellij API call for setting pane background — needs verification against the current Zellij plugin SDK version we target.
- Whether `CustomMessage` to a headless plugin can be routed by plugin URL alone, or whether the alias needs to know the plugin's instance ID.
- How the plugin identifies the *originating pane* of a `CustomMessage` from `zellij action send-plugin-message`. Likely via an env var (`ZELLIJ_PANE_ID`) the shell alias can include in the message payload; needs confirmation.
- Palette selection: pick colors now or defer to implementation.

These are research items for the writing-plans phase, not blockers for the design.
