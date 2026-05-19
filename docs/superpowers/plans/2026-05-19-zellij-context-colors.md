# zellij-context-colors Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a Zellij WASM plugin that tints each pane's background by a `{hostname}/{cwd}` context key, with deterministic auto-assignment, TOML persistence, and a `zctx` shell alias for manual override.

**Architecture:** Pure-Rust WASM plugin using `zellij-tile`. Plugin subscribes to `CwdChanged`, `CommandChanged`, `PaneUpdate`, `PaneClosed` events. Maintains a per-pane state machine that detects `ssh` entry/exit and stashes the local cwd accordingly. Colors are looked up in `~/.config/zellij/context-colors.toml`; unknown keys hash-map into a 16-color palette and are persisted back. Manual overrides flow through Zellij's `pipe` mechanism, with the originating `ZELLIJ_PANE_ID` carried in the message body since pipe messages don't include it natively.

**Tech Stack:** Rust 1.78+, `zellij-tile` (latest published version compatible with Zellij 0.40+), `serde`/`serde_derive`, `toml`, `siphasher`. Target: `wasm32-wasip1`.

---

## Reference Material

- Spec: `docs/superpowers/specs/2026-05-19-zellij-context-colors-design.md`
- Sister plugin (single-file reference): https://github.com/karlbunch/zellij-colorful
- Zellij plugin API: https://zellij.dev/documentation/plugin-api-commands
- Zellij pipes: https://zellij.dev/documentation/plugin-pipes

---

## File Structure

```
zellij-context-colors/
├── Cargo.toml                  # crate manifest, wasm target config
├── rust-toolchain.toml         # pin toolchain + wasm32-wasip1 target
├── src/
│   ├── lib.rs                  # ZellijPlugin impl, event wiring, register_plugin!
│   ├── color.rs                # Rgba type, parse "#RRGGBBAA", format
│   ├── context_key.rs          # ContextKey struct, "host/cwd" (de)serialization
│   ├── ssh_parser.rs           # parse_ssh_target(argv) -> Option<String>
│   ├── palette.rs              # PALETTE: [Rgba; 16], deterministic_color(key)
│   ├── config.rs               # Config: load/save TOML with atomic write
│   └── state.rs                # PaneContext, transitions (cwd/cmd/ssh enter/exit)
├── assets/
│   └── layout.kdl              # example layout loading the plugin at session start
├── docs/
│   └── install.md              # user-facing install steps + zctx alias
└── README.md                   # already exists
```

Each src module is a focused unit with its own `#[cfg(test)] mod tests` block. `lib.rs` is the only file that touches the Zellij runtime; everything else is pure Rust and unit-testable without a Zellij host.

---

## Task 1: Cargo project scaffold

**Files:**
- Create: `Cargo.toml`
- Create: `rust-toolchain.toml`
- Create: `src/lib.rs`

- [ ] **Step 1: Create `rust-toolchain.toml`**

```toml
[toolchain]
channel = "stable"
targets = ["wasm32-wasip1"]
```

- [ ] **Step 2: Create `Cargo.toml`**

```toml
[package]
name = "zellij-context-colors"
version = "0.1.0"
edition = "2021"
authors = ["jaypaulb"]
description = "Zellij plugin that tints panes by {hostname}/{cwd} context"
license = "MIT"
repository = "https://github.com/jaypaulb/zellij-context-colors"

[lib]
crate-type = ["cdylib"]

[dependencies]
zellij-tile = "0.41"
serde = { version = "1", features = ["derive"] }
toml = "0.8"
siphasher = "1"

[profile.release]
opt-level = "z"
lto = true
codegen-units = 1
strip = true
```

- [ ] **Step 3: Create skeleton `src/lib.rs`**

```rust
use zellij_tile::prelude::*;

#[derive(Default)]
struct State;

impl ZellijPlugin for State {
    fn load(&mut self, _configuration: std::collections::BTreeMap<String, String>) {}
    fn update(&mut self, _event: Event) -> bool { false }
    fn render(&mut self, _rows: usize, _cols: usize) {}
}

register_plugin!(State);
```

- [ ] **Step 4: Verify it builds to WASM**

```bash
cargo build --release --target wasm32-wasip1
```

Expected: produces `target/wasm32-wasip1/release/zellij_context_colors.wasm` with no errors. Warnings about unused imports are fine for now.

- [ ] **Step 5: Commit**

```bash
git add Cargo.toml rust-toolchain.toml src/lib.rs
git commit -m "Scaffold: Cargo crate with zellij-tile, builds to wasm32-wasip1"
```

---

## Task 2: Rgba color type with hex parsing

**Files:**
- Create: `src/color.rs`
- Modify: `src/lib.rs` (add `mod color;`)

The spec stores `#RRGGBBAA` (with alpha). Zellij's `set_pane_color` only takes 6-digit hex. We parse and store all 4 bytes, then format the RGB triplet for Zellij; alpha is preserved through the round-trip but unused in v1.

- [ ] **Step 1: Write failing test in `src/color.rs`**

```rust
//! 32-bit RGBA color with #RRGGBBAA parsing.

use serde::{Deserialize, Serialize, Deserializer, Serializer};

#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub struct Rgba { pub r: u8, pub g: u8, pub b: u8, pub a: u8 }

impl Rgba {
    pub fn parse(s: &str) -> Result<Self, String> {
        let s = s.trim_start_matches('#');
        if s.len() != 8 {
            return Err(format!("expected #RRGGBBAA (8 hex digits), got {} chars", s.len()));
        }
        let bytes = u32::from_str_radix(s, 16).map_err(|e| e.to_string())?;
        Ok(Self {
            r: ((bytes >> 24) & 0xFF) as u8,
            g: ((bytes >> 16) & 0xFF) as u8,
            b: ((bytes >> 8) & 0xFF) as u8,
            a: (bytes & 0xFF) as u8,
        })
    }

    pub fn format(self) -> String {
        format!("#{:02X}{:02X}{:02X}{:02X}", self.r, self.g, self.b, self.a)
    }

    /// 6-digit RGB hex for `set_pane_color` (alpha dropped).
    pub fn to_rgb_hex(self) -> String {
        format!("#{:02X}{:02X}{:02X}", self.r, self.g, self.b)
    }
}

impl Serialize for Rgba {
    fn serialize<S: Serializer>(&self, s: S) -> Result<S::Ok, S::Error> {
        s.serialize_str(&self.format())
    }
}

impl<'de> Deserialize<'de> for Rgba {
    fn deserialize<D: Deserializer<'de>>(d: D) -> Result<Self, D::Error> {
        let s = String::deserialize(d)?;
        Self::parse(&s).map_err(serde::de::Error::custom)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_full_rgba() {
        let c = Rgba::parse("#123456FF").unwrap();
        assert_eq!(c, Rgba { r: 0x12, g: 0x34, b: 0x56, a: 0xFF });
    }

    #[test]
    fn parses_without_hash_prefix() {
        let c = Rgba::parse("AA3311FF").unwrap();
        assert_eq!(c, Rgba { r: 0xAA, g: 0x33, b: 0x11, a: 0xFF });
    }

    #[test]
    fn rejects_short_input() {
        assert!(Rgba::parse("#FFF").is_err());
        assert!(Rgba::parse("#123456").is_err());
    }

    #[test]
    fn rejects_non_hex() {
        assert!(Rgba::parse("#GGGGGGGG").is_err());
    }

    #[test]
    fn formats_round_trip() {
        let c = Rgba { r: 0x12, g: 0x34, b: 0x56, a: 0xFF };
        assert_eq!(c.format(), "#123456FF");
        assert_eq!(Rgba::parse(&c.format()).unwrap(), c);
    }

    #[test]
    fn to_rgb_hex_drops_alpha() {
        let c = Rgba { r: 0x12, g: 0x34, b: 0x56, a: 0x80 };
        assert_eq!(c.to_rgb_hex(), "#123456");
    }
}
```

- [ ] **Step 2: Wire module into `src/lib.rs`**

Add at the top of `src/lib.rs`, above `use zellij_tile::prelude::*;`:

```rust
mod color;
```

- [ ] **Step 3: Run tests, expect PASS**

```bash
cargo test --lib color::
```

Expected: all 6 tests pass.

- [ ] **Step 4: Commit**

```bash
git add src/color.rs src/lib.rs
git commit -m "color: add Rgba type with #RRGGBBAA parse/format and serde impls"
```

---

## Task 3: ContextKey type

**Files:**
- Create: `src/context_key.rs`
- Modify: `src/lib.rs` (add `mod context_key;`)

- [ ] **Step 1: Write the module with tests**

```rust
//! Context key: "{host}/{cwd}" used to look up colors.

use std::fmt;

#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct ContextKey {
    pub host: String,
    pub cwd:  String,
}

impl ContextKey {
    pub fn new(host: impl Into<String>, cwd: impl Into<String>) -> Self {
        Self { host: host.into(), cwd: cwd.into() }
    }

    /// Normalise a path: tilde-replace HOME, drop trailing slashes (except "/" and "~").
    pub fn normalise_cwd(path: &str, home: &str) -> String {
        let p = if path == home {
            "~".to_string()
        } else if let Some(rest) = path.strip_prefix(&format!("{home}/")) {
            format!("~/{rest}")
        } else {
            path.to_string()
        };
        if p.len() > 1 && p.ends_with('/') {
            p.trim_end_matches('/').to_string()
        } else {
            p
        }
    }
}

impl fmt::Display for ContextKey {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}/{}", self.host, self.cwd)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn formats_host_slash_cwd() {
        let k = ContextKey::new("hal", "~/Projects/foo");
        assert_eq!(k.to_string(), "hal/~/Projects/foo");
    }

    #[test]
    fn normalises_home_to_tilde() {
        assert_eq!(ContextKey::normalise_cwd("/home/jaypaulb", "/home/jaypaulb"), "~");
    }

    #[test]
    fn normalises_subdir_of_home() {
        assert_eq!(
            ContextKey::normalise_cwd("/home/jaypaulb/Projects/foo", "/home/jaypaulb"),
            "~/Projects/foo"
        );
    }

    #[test]
    fn leaves_unrelated_paths_alone() {
        assert_eq!(ContextKey::normalise_cwd("/etc/hosts", "/home/jaypaulb"), "/etc/hosts");
    }

    #[test]
    fn drops_trailing_slash() {
        assert_eq!(
            ContextKey::normalise_cwd("/home/jaypaulb/Projects/foo/", "/home/jaypaulb"),
            "~/Projects/foo"
        );
    }

    #[test]
    fn preserves_root_slash() {
        assert_eq!(ContextKey::normalise_cwd("/", "/home/jaypaulb"), "/");
    }
}
```

- [ ] **Step 2: Wire module**

In `src/lib.rs`, alongside existing `mod color;` add:

```rust
mod context_key;
```

- [ ] **Step 3: Run tests**

```bash
cargo test --lib context_key::
```

Expected: 6 tests pass.

- [ ] **Step 4: Commit**

```bash
git add src/context_key.rs src/lib.rs
git commit -m "context_key: add ContextKey with cwd normalisation"
```

---

## Task 4: SSH target parser

**Files:**
- Create: `src/ssh_parser.rs`
- Modify: `src/lib.rs` (add `mod ssh_parser;`)

Per spec, the parser walks argv skipping options and their values, strips `user@`, returns `Some(host)` or `None`.

- [ ] **Step 1: Write the module with comprehensive tests**

```rust
//! Parse the target hostname out of an `ssh ...` argv.

/// OpenSSH options that take a value as the next argv element.
const SSH_OPTS_WITH_VALUE: &[&str] = &[
    "-J", "-p", "-i", "-l", "-F", "-L", "-R", "-D", "-W",
    "-o", "-b", "-c", "-e", "-m", "-O", "-Q", "-S", "-B", "-E", "-I",
];

/// Returns the target host of an `ssh` invocation, with any `user@` prefix stripped.
/// Returns `None` if no target argument can be found.
pub fn parse_ssh_target(argv: &[String]) -> Option<String> {
    if argv.first().map(String::as_str) != Some("ssh") {
        return None;
    }

    let mut i = 1;
    while i < argv.len() {
        let arg = &argv[i];
        if SSH_OPTS_WITH_VALUE.contains(&arg.as_str()) {
            i += 2;
            continue;
        }
        if arg.starts_with('-') {
            i += 1;
            continue;
        }
        let target = arg.split_once('@').map(|(_, h)| h).unwrap_or(arg);
        return Some(target.to_string());
    }
    None
}

#[cfg(test)]
mod tests {
    use super::*;

    fn argv(parts: &[&str]) -> Vec<String> {
        parts.iter().map(|s| s.to_string()).collect()
    }

    #[test]
    fn simple_host() {
        assert_eq!(parse_ssh_target(&argv(&["ssh", "hal"])).as_deref(), Some("hal"));
    }

    #[test]
    fn strips_user_prefix() {
        assert_eq!(parse_ssh_target(&argv(&["ssh", "jaypaul@hal"])).as_deref(), Some("hal"));
    }

    #[test]
    fn skips_port_option() {
        assert_eq!(parse_ssh_target(&argv(&["ssh", "-p", "2222", "hal"])).as_deref(), Some("hal"));
    }

    #[test]
    fn skips_jumphost_option() {
        assert_eq!(
            parse_ssh_target(&argv(&["ssh", "-J", "jump", "hal"])).as_deref(),
            Some("hal")
        );
    }

    #[test]
    fn skips_combined_options() {
        assert_eq!(
            parse_ssh_target(&argv(&["ssh", "-tt", "-p", "22", "-i", "key.pem", "jp@hal"])).as_deref(),
            Some("hal")
        );
    }

    #[test]
    fn no_target_returns_none() {
        assert_eq!(parse_ssh_target(&argv(&["ssh"])), None);
        assert_eq!(parse_ssh_target(&argv(&["ssh", "-V"])), None);
    }

    #[test]
    fn not_ssh_returns_none() {
        assert_eq!(parse_ssh_target(&argv(&["mosh", "hal"])), None);
        assert_eq!(parse_ssh_target(&argv(&["ssh-keygen", "-t", "ed25519"])), None);
    }

    #[test]
    fn ignores_extra_args_after_target() {
        // After we find the target, any remaining argv is the remote command.
        assert_eq!(
            parse_ssh_target(&argv(&["ssh", "hal", "ls", "-la"])).as_deref(),
            Some("hal")
        );
    }
}
```

- [ ] **Step 2: Wire module**

Add `mod ssh_parser;` to `src/lib.rs`.

- [ ] **Step 3: Run tests**

```bash
cargo test --lib ssh_parser::
```

Expected: 8 tests pass.

- [ ] **Step 4: Commit**

```bash
git add src/ssh_parser.rs src/lib.rs
git commit -m "ssh_parser: option-aware SSH target extraction with user@ stripping"
```

---

## Task 5: Palette + deterministic color

**Files:**
- Create: `src/palette.rs`
- Modify: `src/lib.rs` (add `mod palette;`)

Curated 16-color muted-dark palette. Each entry is a background color dark enough for terminal text to remain readable on top. Hash with `SipHasher13` (fixed seed) for stable, well-distributed assignment.

- [ ] **Step 1: Write the module with tests**

```rust
//! Deterministic colour assignment from a curated dark palette.

use crate::color::Rgba;
use siphasher::sip::SipHasher13;
use std::hash::{Hash, Hasher};

/// 16 muted dark backgrounds chosen for readability under terminal text.
/// Alpha is FF; consumers may override if/when Zellij gains alpha support.
pub const PALETTE: [Rgba; 16] = [
    Rgba { r: 0x2A, g: 0x1F, b: 0x3D, a: 0xFF }, // deep purple
    Rgba { r: 0x1F, g: 0x2A, b: 0x3D, a: 0xFF }, // deep blue
    Rgba { r: 0x1F, g: 0x3D, b: 0x2A, a: 0xFF }, // deep green
    Rgba { r: 0x3D, g: 0x2A, b: 0x1F, a: 0xFF }, // deep brown
    Rgba { r: 0x3D, g: 0x1F, b: 0x2A, a: 0xFF }, // deep maroon
    Rgba { r: 0x1F, g: 0x3D, b: 0x3D, a: 0xFF }, // deep teal
    Rgba { r: 0x3D, g: 0x3D, b: 0x1F, a: 0xFF }, // deep olive
    Rgba { r: 0x2F, g: 0x1F, b: 0x3D, a: 0xFF }, // purple-blue
    Rgba { r: 0x3D, g: 0x2F, b: 0x1F, a: 0xFF }, // rust
    Rgba { r: 0x1F, g: 0x3D, b: 0x2F, a: 0xFF }, // forest
    Rgba { r: 0x3D, g: 0x1F, b: 0x3D, a: 0xFF }, // plum
    Rgba { r: 0x1F, g: 0x2F, b: 0x3D, a: 0xFF }, // steel
    Rgba { r: 0x3D, g: 0x3D, b: 0x2F, a: 0xFF }, // mustard
    Rgba { r: 0x2F, g: 0x3D, b: 0x1F, a: 0xFF }, // moss
    Rgba { r: 0x3D, g: 0x2F, b: 0x3D, a: 0xFF }, // mauve
    Rgba { r: 0x2F, g: 0x1F, b: 0x2F, a: 0xFF }, // eggplant
];

const HASH_KEY_0: u64 = 0x5A5A_5A5A_5A5A_5A5A;
const HASH_KEY_1: u64 = 0xA5A5_A5A5_A5A5_A5A5;

/// Hash a context key string and pick a palette entry deterministically.
pub fn deterministic_color(key: &str) -> Rgba {
    let mut hasher = SipHasher13::new_with_keys(HASH_KEY_0, HASH_KEY_1);
    key.hash(&mut hasher);
    let idx = (hasher.finish() as usize) % PALETTE.len();
    PALETTE[idx]
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn palette_has_16_entries() {
        assert_eq!(PALETTE.len(), 16);
    }

    #[test]
    fn same_key_same_color() {
        let a = deterministic_color("hal/~/Projects/foo");
        let b = deterministic_color("hal/~/Projects/foo");
        assert_eq!(a, b);
    }

    #[test]
    fn different_keys_likely_different_colors() {
        // Not guaranteed, but for these specific keys the hash differs.
        let a = deterministic_color("hal/~");
        let b = deterministic_color("halbuntu/~");
        assert_ne!(a, b);
    }

    #[test]
    fn all_palette_entries_are_dark() {
        for c in PALETTE {
            let max = c.r.max(c.g).max(c.b);
            assert!(max <= 0x4F, "palette entry too bright: {:?}", c);
        }
    }
}
```

- [ ] **Step 2: Wire module**

Add `mod palette;` to `src/lib.rs`.

- [ ] **Step 3: Run tests**

```bash
cargo test --lib palette::
```

Expected: 4 tests pass.

- [ ] **Step 4: Commit**

```bash
git add src/palette.rs src/lib.rs
git commit -m "palette: 16 muted-dark backgrounds with SipHash13 deterministic assignment"
```

---

## Task 6: Config load/save with atomic write

**Files:**
- Create: `src/config.rs`
- Modify: `src/lib.rs` (add `mod config;`)

- [ ] **Step 1: Write the module with tests**

```rust
//! TOML config: ~/.config/zellij/context-colors.toml
//!
//! Format:
//!   [colors]
//!   "hal/~"              = "#123456FF"
//!   "halbuntu/~"         = "#226644FF"

use crate::color::Rgba;
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::path::{Path, PathBuf};

#[derive(Debug, Default, Clone, Serialize, Deserialize)]
pub struct Config {
    #[serde(default)]
    pub colors: HashMap<String, Rgba>,
}

impl Config {
    pub fn load_from(path: &Path) -> Result<Self, String> {
        match std::fs::read_to_string(path) {
            Ok(text) => toml::from_str::<Config>(&text).map_err(|e| e.to_string()),
            Err(e) if e.kind() == std::io::ErrorKind::NotFound => Ok(Config::default()),
            Err(e) => Err(e.to_string()),
        }
    }

    /// Atomic write: serialise to TOML, write to `<path>.tmp`, then rename over `<path>`.
    pub fn save_to(&self, path: &Path) -> Result<(), String> {
        if let Some(parent) = path.parent() {
            std::fs::create_dir_all(parent).map_err(|e| e.to_string())?;
        }
        let text = toml::to_string_pretty(self).map_err(|e| e.to_string())?;
        let tmp = tmp_path(path);
        std::fs::write(&tmp, text).map_err(|e| e.to_string())?;
        std::fs::rename(&tmp, path).map_err(|e| e.to_string())?;
        Ok(())
    }
}

fn tmp_path(path: &Path) -> PathBuf {
    let mut s = path.as_os_str().to_owned();
    s.push(".tmp");
    PathBuf::from(s)
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::io::Write;

    fn tempdir() -> std::path::PathBuf {
        let p = std::env::temp_dir().join(format!("zcc-test-{}", std::process::id()));
        let _ = std::fs::remove_dir_all(&p);
        std::fs::create_dir_all(&p).unwrap();
        p
    }

    #[test]
    fn missing_file_yields_empty_config() {
        let dir = tempdir();
        let path = dir.join("nope.toml");
        let cfg = Config::load_from(&path).unwrap();
        assert!(cfg.colors.is_empty());
    }

    #[test]
    fn round_trip_preserves_entries() {
        let dir = tempdir();
        let path = dir.join("ctx.toml");
        let mut cfg = Config::default();
        cfg.colors.insert("hal/~".into(), Rgba::parse("#123456FF").unwrap());
        cfg.colors.insert("halbuntu/~".into(), Rgba::parse("#226644FF").unwrap());
        cfg.save_to(&path).unwrap();
        let loaded = Config::load_from(&path).unwrap();
        assert_eq!(loaded.colors, cfg.colors);
    }

    #[test]
    fn unparseable_file_returns_err_without_clobbering() {
        let dir = tempdir();
        let path = dir.join("bad.toml");
        let mut f = std::fs::File::create(&path).unwrap();
        f.write_all(b"this is not valid toml = = =").unwrap();
        let err = Config::load_from(&path);
        assert!(err.is_err(), "expected parse error, got {:?}", err);
        // Caller is responsible for not overwriting; we just ensure load returns Err.
    }

    #[test]
    fn save_creates_parent_directories() {
        let dir = tempdir();
        let path = dir.join("nested/deep/ctx.toml");
        let mut cfg = Config::default();
        cfg.colors.insert("a/b".into(), Rgba::parse("#FFFFFFFF").unwrap());
        cfg.save_to(&path).unwrap();
        assert!(path.exists());
    }
}
```

- [ ] **Step 2: Wire module**

Add `mod config;` to `src/lib.rs`.

- [ ] **Step 3: Run tests**

```bash
cargo test --lib config::
```

Expected: 4 tests pass.

- [ ] **Step 4: Commit**

```bash
git add src/config.rs src/lib.rs
git commit -m "config: TOML load/save with atomic write and graceful-missing-file"
```

---

## Task 7: PaneContext state machine

**Files:**
- Create: `src/state.rs`
- Modify: `src/lib.rs` (add `mod state;`)

Encapsulates the cwd-stash/restore logic so it can be tested without the Zellij runtime.

- [ ] **Step 1: Write the module with tests**

```rust
//! Per-pane state and the transitions that derive a context key.

use crate::color::Rgba;
use crate::context_key::ContextKey;
use crate::ssh_parser::parse_ssh_target;

#[derive(Debug, Clone)]
pub struct PaneContext {
    pub host: String,
    pub cwd:  String,
    pub local_host: String,
    pub in_ssh: bool,
    pub local_cwd: Option<String>,
    pub last_color: Option<Rgba>,
}

impl PaneContext {
    pub fn new(local_host: String, cwd: String) -> Self {
        Self {
            host: local_host.clone(),
            cwd,
            local_host,
            in_ssh: false,
            local_cwd: None,
            last_color: None,
        }
    }

    pub fn key(&self) -> ContextKey {
        ContextKey::new(&self.host, &self.cwd)
    }

    /// CwdChanged event: update cwd. (Caller has already normalised it.)
    pub fn on_cwd_changed(&mut self, new_cwd: String) {
        self.cwd = new_cwd;
    }

    /// CommandChanged event: run the SSH-detection state machine.
    pub fn on_command_changed(&mut self, argv: &[String]) {
        match parse_ssh_target(argv) {
            Some(target) => {
                if !self.in_ssh {
                    self.local_cwd = Some(self.cwd.clone());
                }
                self.host = target;
                self.cwd = "~".to_string();
                self.in_ssh = true;
            }
            None if argv.first().map(String::as_str) == Some("ssh") => {
                // ssh with no parseable target: host=?, cwd=~
                if !self.in_ssh {
                    self.local_cwd = Some(self.cwd.clone());
                }
                self.host = "?".to_string();
                self.cwd = "~".to_string();
                self.in_ssh = true;
            }
            None => {
                if self.in_ssh {
                    self.host = self.local_host.clone();
                    self.cwd = self.local_cwd.take().unwrap_or_else(|| "~".to_string());
                    self.in_ssh = false;
                }
                // Otherwise: not ssh, not in ssh — leave key alone.
            }
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    fn argv(parts: &[&str]) -> Vec<String> {
        parts.iter().map(|s| s.to_string()).collect()
    }

    #[test]
    fn starts_with_local_host_and_given_cwd() {
        let ctx = PaneContext::new("halbuntu".into(), "~/Projects/foo".into());
        assert_eq!(ctx.key().to_string(), "halbuntu/~/Projects/foo");
        assert!(!ctx.in_ssh);
    }

    #[test]
    fn cwd_change_updates_key() {
        let mut ctx = PaneContext::new("halbuntu".into(), "~".into());
        ctx.on_cwd_changed("~/Projects/bar".into());
        assert_eq!(ctx.key().to_string(), "halbuntu/~/Projects/bar");
    }

    #[test]
    fn ssh_entry_stashes_local_cwd_and_resets() {
        let mut ctx = PaneContext::new("halbuntu".into(), "~/Projects/foo".into());
        ctx.on_command_changed(&argv(&["ssh", "hal"]));
        assert!(ctx.in_ssh);
        assert_eq!(ctx.key().to_string(), "hal/~");
        assert_eq!(ctx.local_cwd.as_deref(), Some("~/Projects/foo"));
    }

    #[test]
    fn ssh_exit_restores_local_cwd() {
        let mut ctx = PaneContext::new("halbuntu".into(), "~/Projects/foo".into());
        ctx.on_command_changed(&argv(&["ssh", "hal"]));
        ctx.on_command_changed(&argv(&["bash"]));
        assert!(!ctx.in_ssh);
        assert_eq!(ctx.key().to_string(), "halbuntu/~/Projects/foo");
        assert!(ctx.local_cwd.is_none());
    }

    #[test]
    fn cwd_changed_inside_ssh_updates_remote_cwd() {
        let mut ctx = PaneContext::new("halbuntu".into(), "~/Projects/foo".into());
        ctx.on_command_changed(&argv(&["ssh", "hal"]));
        ctx.on_cwd_changed("~/work/proj".into()); // OSC 7 from remote
        assert_eq!(ctx.key().to_string(), "hal/~/work/proj");
    }

    #[test]
    fn non_ssh_command_doesnt_change_key_when_local() {
        let mut ctx = PaneContext::new("halbuntu".into(), "~/Projects/foo".into());
        let before = ctx.key().to_string();
        ctx.on_command_changed(&argv(&["vim", "file.rs"]));
        ctx.on_command_changed(&argv(&["cly"]));
        ctx.on_command_changed(&argv(&["cargo", "build"]));
        assert_eq!(ctx.key().to_string(), before);
        assert!(!ctx.in_ssh);
    }

    #[test]
    fn ssh_with_no_target_uses_question_mark() {
        let mut ctx = PaneContext::new("halbuntu".into(), "~".into());
        ctx.on_command_changed(&argv(&["ssh"]));
        assert!(ctx.in_ssh);
        assert_eq!(ctx.key().to_string(), "?/~");
    }

    #[test]
    fn ssh_user_at_host_form() {
        let mut ctx = PaneContext::new("halbuntu".into(), "~".into());
        ctx.on_command_changed(&argv(&["ssh", "jp@hal"]));
        assert_eq!(ctx.key().to_string(), "hal/~");
    }
}
```

- [ ] **Step 2: Wire module**

Add `mod state;` to `src/lib.rs`.

- [ ] **Step 3: Run tests**

```bash
cargo test --lib state::
```

Expected: 8 tests pass.

- [ ] **Step 4: Commit**

```bash
git add src/state.rs src/lib.rs
git commit -m "state: PaneContext with SSH entry/exit + cwd stash/restore"
```

---

## Task 8: ZellijPlugin trait implementation

**Files:**
- Modify: `src/lib.rs` (replace skeleton with full implementation)

This wires the pure modules into the Zellij runtime. Cannot be unit-tested without a Zellij host; verified by build + manual smoke test in a Zellij session (covered in Tasks 11/12).

- [ ] **Step 1: Replace `src/lib.rs` with the full implementation**

```rust
mod color;
mod config;
mod context_key;
mod palette;
mod ssh_parser;
mod state;

use crate::color::Rgba;
use crate::config::Config;
use crate::context_key::ContextKey;
use crate::palette::deterministic_color;
use crate::state::PaneContext;

use std::collections::{BTreeMap, HashMap};
use std::path::PathBuf;
use zellij_tile::prelude::*;

const CONFIG_RELPATH: &str = ".config/zellij/context-colors.toml";

#[derive(Default)]
struct State {
    permissions_granted: bool,
    config: Config,
    config_path: PathBuf,
    panes: HashMap<u32, PaneContext>,
    local_host: String,
    home: String,
    permissions_pending: bool,
}

impl ZellijPlugin for State {
    fn load(&mut self, _configuration: BTreeMap<String, String>) {
        self.home = std::env::var("HOME").unwrap_or_else(|_| "/".to_string());
        self.local_host = std::env::var("HOSTNAME")
            .or_else(|_| std::env::var("HOST"))
            .unwrap_or_else(|_| "local".to_string());
        self.config_path = PathBuf::from(&self.home).join(CONFIG_RELPATH);
        self.config = Config::load_from(&self.config_path).unwrap_or_default();

        request_permission(&[
            PermissionType::ReadApplicationState,
            PermissionType::ChangeApplicationState,
        ]);
        subscribe(&[
            EventType::PermissionRequestResult,
            EventType::PaneUpdate,
            EventType::CwdChanged,
            EventType::CommandChanged,
            EventType::PaneClosed,
        ]);
        self.permissions_pending = true;
    }

    fn update(&mut self, event: Event) -> bool {
        match event {
            Event::PermissionRequestResult(PermissionStatus::Granted) => {
                self.permissions_granted = true;
                self.permissions_pending = false;
                set_selectable(false);
            }
            Event::PermissionRequestResult(PermissionStatus::Denied) => {
                self.permissions_pending = false;
                eprintln!("zellij-context-colors: permissions denied; plugin idle");
            }
            Event::PaneUpdate(manifest) if self.permissions_granted => {
                self.handle_pane_update(manifest);
            }
            Event::CwdChanged(pane_id, new_cwd) if self.permissions_granted => {
                let normalised = ContextKey::normalise_cwd(&new_cwd, &self.home);
                self.with_pane(pane_id, |ctx| ctx.on_cwd_changed(normalised));
                self.apply(pane_id);
            }
            Event::CommandChanged(pane_id, argv) if self.permissions_granted => {
                self.with_pane(pane_id, |ctx| ctx.on_command_changed(&argv));
                self.apply(pane_id);
            }
            Event::PaneClosed(pane_id) => {
                self.panes.remove(&pane_id);
            }
            _ => {}
        }
        false
    }

    fn pipe(&mut self, pipe_message: PipeMessage) -> bool {
        // Handled in Task 9; placeholder until then.
        let _ = pipe_message;
        false
    }

    fn render(&mut self, _rows: usize, _cols: usize) {}
}

impl State {
    fn handle_pane_update(&mut self, manifest: PaneManifest) {
        let local_host = self.local_host.clone();
        for (_, panes) in manifest.panes {
            for info in panes {
                if info.is_plugin { continue; }
                let pane_id = info.id;
                let lh = local_host.clone();
                self.panes.entry(pane_id).or_insert_with(|| {
                    PaneContext::new(lh, "~".to_string())
                });
                self.apply(pane_id);
            }
        }
    }

    fn ensure_pane(&mut self, pane_id: u32) {
        let local_host = self.local_host.clone();
        self.panes.entry(pane_id).or_insert_with(|| {
            PaneContext::new(local_host, "~".to_string())
        });
    }

    fn with_pane<F: FnOnce(&mut PaneContext)>(&mut self, pane_id: u32, f: F) {
        self.ensure_pane(pane_id);
        if let Some(ctx) = self.panes.get_mut(&pane_id) {
            f(ctx);
        }
    }

    fn apply(&mut self, pane_id: u32) {
        let Some(ctx) = self.panes.get_mut(&pane_id) else { return };
        let key = ctx.key().to_string();
        let color = match self.config.colors.get(&key) {
            Some(c) => *c,
            None => {
                let c = deterministic_color(&key);
                self.config.colors.insert(key.clone(), c);
                if let Err(e) = self.config.save_to(&self.config_path) {
                    eprintln!("zellij-context-colors: failed to save config: {e}");
                }
                c
            }
        };
        if ctx.last_color == Some(color) { return; }
        ctx.last_color = Some(color);
        set_pane_color(PaneId::Terminal(pane_id), None, Some(color.to_rgb_hex()));
    }
}

register_plugin!(State);
```

> **Note on Zellij API names**: This task assumes the published `zellij-tile` API exposes `set_pane_color(PaneId, Option<String>, Option<String>)`, `request_permission`, `subscribe`, `set_selectable`, `PaneId::Terminal(u32)`, `Event::CwdChanged(u32, String)`, `Event::CommandChanged(u32, Vec<String>)`, and `Event::PaneClosed(u32)`. If the actual signatures differ in the version pinned by `Cargo.toml`, adjust the calls and event-pattern bindings to match — the public docs at zellij.dev are the source of truth. Do **not** change the module structure or the semantics; only adapt the binding code.

- [ ] **Step 2: Build for WASM, expect clean compilation**

```bash
cargo build --release --target wasm32-wasip1
```

Expected: builds successfully. If the Zellij API signatures differ from those assumed, fix the call sites in `lib.rs` only — the pure modules and their tests are unaffected.

- [ ] **Step 3: Run all unit tests**

```bash
cargo test --lib
```

Expected: all tests from Tasks 2-7 still pass.

- [ ] **Step 4: Commit**

```bash
git add src/lib.rs
git commit -m "lib: ZellijPlugin impl wiring events, state map, and set_pane_color"
```

---

## Task 9: `pipe()` handler for manual override

**Files:**
- Modify: `src/lib.rs` (replace placeholder `pipe()` and add helper)

Wire format for the message body, sent by `zctx`:

```
set #RRGGBBAA pane=<ZELLIJ_PANE_ID>
```

The plugin parses it, looks up the pane's current context key, writes the override to config, and applies it. All other panes sharing that key re-resolve through the existing `apply()` flow on their next event — for an immediate visual update across panes, we iterate the state map and re-apply any whose key matches.

- [ ] **Step 1: Replace the placeholder `pipe()` and add helpers**

In `src/lib.rs`, replace the `pipe()` method body and add a helper after `apply`:

```rust
    fn pipe(&mut self, pipe_message: PipeMessage) -> bool {
        if !self.permissions_granted { return false; }
        let Some(payload) = pipe_message.payload else { return false; };
        let Some((color, pane_id)) = parse_zctx_payload(&payload) else {
            eprintln!("zellij-context-colors: malformed zctx payload: {payload:?}");
            return false;
        };
        self.ensure_pane(pane_id);
        let key = self.panes[&pane_id].key().to_string();
        self.config.colors.insert(key.clone(), color);
        if let Err(e) = self.config.save_to(&self.config_path) {
            eprintln!("zellij-context-colors: failed to save config: {e}");
        }
        // Re-apply to every pane currently on this key. apply() short-circuits
        // when the new color equals last_color, so this is a no-op for panes
        // that already display the override.
        let matching: Vec<u32> = self.panes.iter()
            .filter(|(_, c)| c.key().to_string() == key)
            .map(|(id, _)| *id).collect();
        for id in matching {
            self.apply(id);
        }
        false
    }
```

Add the parser function near the bottom of `src/lib.rs` (above `register_plugin!`):

```rust
fn parse_zctx_payload(payload: &str) -> Option<(Rgba, u32)> {
    // "set #RRGGBBAA pane=N"
    let mut parts = payload.split_whitespace();
    if parts.next()? != "set" { return None; }
    let color = Rgba::parse(parts.next()?).ok()?;
    let pane_kv = parts.next()?;
    let pane_id = pane_kv.strip_prefix("pane=")?.parse::<u32>().ok()?;
    Some((color, pane_id))
}

#[cfg(test)]
mod pipe_payload_tests {
    use super::*;

    #[test]
    fn parses_well_formed() {
        let (c, id) = parse_zctx_payload("set #AABBCCFF pane=7").unwrap();
        assert_eq!(c, Rgba::parse("#AABBCCFF").unwrap());
        assert_eq!(id, 7);
    }

    #[test]
    fn rejects_missing_pane() {
        assert!(parse_zctx_payload("set #AABBCCFF").is_none());
    }

    #[test]
    fn rejects_bad_color() {
        assert!(parse_zctx_payload("set FF pane=1").is_none());
    }

    #[test]
    fn rejects_bad_verb() {
        assert!(parse_zctx_payload("paint #AABBCCFF pane=1").is_none());
    }
}
```

- [ ] **Step 2: Run unit tests**

```bash
cargo test --lib
```

Expected: previous tests still pass, plus 4 new `pipe_payload_tests`.

- [ ] **Step 3: Build for WASM**

```bash
cargo build --release --target wasm32-wasip1
```

Expected: clean build.

- [ ] **Step 4: Commit**

```bash
git add src/lib.rs
git commit -m "pipe: zctx 'set #RRGGBBAA pane=N' override propagates across matching panes"
```

---

## Task 10: Example layout + install docs + zctx alias

**Files:**
- Create: `assets/layout.kdl`
- Create: `docs/install.md`

- [ ] **Step 1: Create `assets/layout.kdl`**

```kdl
// Example Zellij layout that starts zellij-context-colors as a headless plugin.
// Adapt the plugin_url to point at your installed .wasm file.
layout {
    default_tab_template {
        children
        pane size=1 borderless=true {
            plugin location="file:~/.config/zellij/plugins/zellij-context-colors.wasm"
        }
    }
}
```

- [ ] **Step 2: Create `docs/install.md`**

````markdown
# Installing zellij-context-colors

## Build

```sh
cargo build --release --target wasm32-wasip1
```

Output: `target/wasm32-wasip1/release/zellij_context_colors.wasm`.

## Install

```sh
mkdir -p ~/.config/zellij/plugins
cp target/wasm32-wasip1/release/zellij_context_colors.wasm \
   ~/.config/zellij/plugins/zellij-context-colors.wasm
```

## Load at session start

Copy `assets/layout.kdl` from this repo to your Zellij layouts directory, or merge its `default_tab_template` block into your existing layout.

```sh
mkdir -p ~/.config/zellij/layouts
cp assets/layout.kdl ~/.config/zellij/layouts/context-colors.kdl
```

Then start Zellij with the layout once to load the plugin:

```sh
zellij --layout context-colors
```

(Once active, the plugin runs as a background pane and persists for the session.)

## The `zctx` shell alias

Add to your `~/.zshrc` or `~/.bashrc`:

```sh
zctx() {
  if [ -z "$ZELLIJ_PANE_ID" ]; then
    echo "zctx: not inside a Zellij pane" >&2
    return 1
  fi
  if [ "$1" != "set" ] || [ -z "$2" ]; then
    echo "usage: zctx set #RRGGBBAA" >&2
    return 1
  fi
  zellij pipe \
    --plugin "file:$HOME/.config/zellij/plugins/zellij-context-colors.wasm" \
    -- "set $2 pane=$ZELLIJ_PANE_ID"
}
```

Then in any pane:

```sh
zctx set '#AA3311FF'
```

## Config file

`~/.config/zellij/context-colors.toml` is written automatically the first time a new context is seen. You can edit it by hand — entries look like:

```toml
[colors]
"hal/~"              = "#123456FF"
"hal/~/Projects/foo" = "#AA3311FF"
"halbuntu/~"         = "#226644FF"
```

The alpha byte is parsed and preserved but not currently used (Zellij's `set_pane_color` accepts only RGB).
````

- [ ] **Step 3: Commit**

```bash
git add assets/layout.kdl docs/install.md
git commit -m "docs: layout, install steps, and zctx shell alias"
```

---

## Task 11: Manual smoke test in Zellij

This is a verification step, not code. There are no unit tests that can exercise the Zellij runtime path; we confirm the plugin works end-to-end in a real session.

- [ ] **Step 1: Install per `docs/install.md`**

Build, copy the .wasm to `~/.config/zellij/plugins/`, copy the layout, and add the `zctx` alias to the active shell rc file. Source the rc file or open a new terminal.

- [ ] **Step 2: Open a fresh Zellij session**

```sh
zellij --layout context-colors
```

Expected: a normal Zellij session opens. The plugin pane is invisible (borderless, size=1). The current pane's background should pick up a color from the palette within ~1 second.

- [ ] **Step 3: Verify auto-assignment persistence**

```sh
cat ~/.config/zellij/context-colors.toml
```

Expected: a `[colors]` entry exists for `<your-hostname>/~` with an `#RRGGBBFF` value. Restart Zellij; the same color appears.

- [ ] **Step 4: Verify cwd-tracking**

```sh
cd ~/Projects/foo  # any directory
```

Expected: pane background changes to a different palette color. New entry appears in `context-colors.toml`.

- [ ] **Step 5: Verify SSH host flip**

```sh
ssh some-host    # any real host you can reach
```

Expected: pane background changes again. After `exit`, the original (pre-ssh) color returns.

- [ ] **Step 6: Verify non-SSH commands don't flip**

```sh
vim   # then :q
cargo build || true
```

Expected: no color change while these run.

- [ ] **Step 7: Verify manual override**

```sh
zctx set '#553355FF'
```

Expected: pane immediately changes to that color. Reload the config file — the new entry is present. Open a second pane and `cd` to the same directory — it gets the same color.

- [ ] **Step 8: Document any deviations**

If any of the steps fail or behave differently, capture the symptom in a new file `docs/known-issues.md` and commit it. Do **not** silently work around problems — surface them.

- [ ] **Step 9: Commit any docs updates**

```bash
git add -A
git commit -m "test: manual smoke test completed (see known-issues.md if present)"
```

---

## Task 12: Tag v0.1.0

- [ ] **Step 1: Update `CHANGELOG.md` (create if absent)**

```markdown
# Changelog

## v0.1.0 — 2026-05-19

Initial release.

- Per-pane background tinting by `{hostname}/{cwd}` context key.
- SSH detection via `CommandChanged` with option-aware target parsing.
- Local cwd stash/restore across SSH entry/exit.
- Deterministic 16-color palette assignment with SipHash13.
- TOML config persistence with atomic write.
- `zctx` shell alias for manual override, propagating across all panes sharing the key.
```

- [ ] **Step 2: Commit and tag**

```bash
git add CHANGELOG.md
git commit -m "chore: changelog for v0.1.0"
git tag -a v0.1.0 -m "v0.1.0 initial release"
git push --follow-tags
```

---

## Spec coverage check

| Spec requirement | Implemented in |
| --- | --- |
| Context key `{host}/{cwd}` | Task 3 (ContextKey), Task 7 (PaneContext::key) |
| SSH detection + option-aware target parsing | Task 4 (ssh_parser), Task 7 (state transition) |
| Strip `user@` prefix | Task 4 |
| Local-cwd stash on SSH entry / restore on exit | Task 7 |
| `?` fallback when SSH target unparseable | Task 7 |
| Only `ssh` flips host; other commands no-op | Task 7 |
| OSC 7 / `CwdChanged` updates remote cwd while in SSH | Task 7, Task 8 |
| Deterministic color from 16-entry palette | Task 5 |
| Persisted auto-assignment | Task 8 (`apply` writes on first miss) |
| TOML config at `~/.config/zellij/context-colors.toml` | Task 6, Task 8 |
| Atomic write (tmp + rename) | Task 6 |
| Unparseable config logged + not clobbered | Task 6 + Task 8 (`load_from` Err path) |
| Pane background applied via `set_pane_color` | Task 8 |
| Alpha parsed/preserved, RGB applied | Task 2 (`to_rgb_hex`) |
| `zctx set` manual override | Task 9, Task 10 |
| Multi-pane sync on override | Task 9 (matching-key re-apply) |
| Headless plugin (no visible UI) | Task 8 (`set_selectable(false)`), Task 10 (layout) |
| Plugin restart rebuilds state from `PaneUpdate` | Task 8 (`handle_pane_update`) |
| Drop pane on `PaneClosed` | Task 8 |
