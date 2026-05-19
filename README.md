# zellij-context-colors

A [Zellij](https://zellij.dev) plugin that tints each pane's background based on its **context** — the combination of `{hostname}/{cwd}`. SSH into a remote host and the color flips; change directory and it flips again; run any other command in the same directory and the color stays.

Inspired by [karlbunch/zellij-colorful](https://github.com/karlbunch/zellij-colorful), but deterministic and persistent rather than random per pane.

## Status

Design phase. See [`docs/superpowers/specs/`](docs/superpowers/specs/) for the design document.

## How it works (in brief)

- The plugin subscribes to Zellij's `CwdChanged` and `CommandChanged` events.
- For each pane it tracks a context key: `{host}/{cwd}`.
  - Locally: host is the machine's hostname.
  - Inside `ssh <target>`: host is the SSH target.
  - Any other foreground command (vim, cly, python…) does not change the host.
- The context key is looked up in `~/.config/zellij/context-colors.toml`. New keys get a deterministic color from a curated palette and are written back to the config so they stay stable across restarts.
- A shell alias lets you override the color for the current context interactively.

## Config

`~/.config/zellij/context-colors.toml`:

```toml
[colors]
"hal/~"              = "#123456FF"
"hal/~/Projects/foo" = "#AA3311FF"
"halbuntu/~"         = "#226644FF"
```

The trailing two hex digits are the alpha channel.
