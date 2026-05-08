# Claude Keypad — Phase 1 Implementation Plan (PC Service + Hook)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up the PC-side software that intercepts Claude Code's PreToolUse permission requests via the official hook system. End state: when Claude Code wants to run a tool, the hook fires, the service receives the request, prints it to console, returns "allow", and Claude proceeds — proving the entire integration path works before any hardware is involved. The hook also gracefully degrades to "ask" whenever the service is unreachable or any error occurs, so Claude Code never gets stuck.

**Architecture:** A single Cargo workspace under `pc/` with three crates:

- `common` — shared serde types (hook stdin schema, hook stdout schema, service HTTP request/response).
- `svc` — `claude-keypad-svc` binary running an axum HTTP server on `127.0.0.1:48420`. v1 always returns `allow`; later phases plug in the screen + keyboard.
- `hook` — `claude-keypad-hook` binary invoked by Claude Code per PreToolUse event. Reads JSON stdin, POSTs to svc, writes `hookSpecificOutput` JSON to stdout. Error paths always emit `permissionDecision: "ask"`.

**Tech Stack:** Rust 1.76+ stable, Cargo workspace, axum 0.7, reqwest 0.12 (blocking client), serde / serde_json, tracing + tracing-subscriber, tokio. Tests use cargo's built-in framework + `tower::ServiceExt` for in-process router testing. No DB, no external services, Windows-only target.

**Repo layout (target end state of Phase 1):**

```
slot_machine/
├── docs/superpowers/{specs,plans}/...
├── pc/
│   ├── Cargo.toml                  ← workspace root
│   ├── common/
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   ├── svc/
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── main.rs
│   │       ├── lib.rs
│   │       └── handlers.rs
│   ├── hook/
│   │   ├── Cargo.toml
│   │   └── src/
│   │       ├── main.rs
│   │       └── lib.rs
│   └── .gitignore
├── README.md                        ← high-level project README
└── pc/README.md                     ← Phase 1 build/install instructions
```

**Prerequisites checklist (do before Task 1):**

- [ ] Rust toolchain installed: `rustc --version` returns ≥ 1.76. Install via https://rustup.rs if missing.
- [ ] `~/.cargo/bin` on PATH (rustup default; verify with `cargo --version` from a fresh shell).
- [ ] Claude Code CLI works locally: `claude --version` from a shell.
- [ ] User-level Claude Code settings file exists at `%USERPROFILE%\.claude\settings.json` (create empty `{}` if absent — Task 14 will modify it).

---

## Task 1: Cargo workspace skeleton

**Files:**

- Create: `pc/Cargo.toml`
- Create: `pc/common/Cargo.toml`
- Create: `pc/common/src/lib.rs`
- Create: `pc/svc/Cargo.toml`
- Create: `pc/svc/src/main.rs`
- Create: `pc/svc/src/lib.rs`
- Create: `pc/hook/Cargo.toml`
- Create: `pc/hook/src/main.rs`
- Create: `pc/hook/src/lib.rs`
- Create: `pc/.gitignore`

- [ ] **Step 1: Create workspace `Cargo.toml`**

`pc/Cargo.toml`:

```toml
[workspace]
resolver = "2"
members = ["common", "svc", "hook"]

[workspace.package]
edition = "2021"
version = "0.1.0"
license = "MIT"
publish = false

[workspace.dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
anyhow = "1"
thiserror = "1"
```

- [ ] **Step 2: Create `common` crate**

`pc/common/Cargo.toml`:

```toml
[package]
name = "claude-keypad-common"
edition.workspace = true
version.workspace = true
license.workspace = true
publish.workspace = true

[dependencies]
serde = { workspace = true }
serde_json = { workspace = true }
```

`pc/common/src/lib.rs`:

```rust
//! Shared types for hook ↔ svc IPC and hook ↔ Claude Code IO.
```

- [ ] **Step 3: Create `svc` crate**

`pc/svc/Cargo.toml`:

```toml
[package]
name = "claude-keypad-svc"
edition.workspace = true
version.workspace = true
license.workspace = true
publish.workspace = true

[[bin]]
name = "claude-keypad-svc"
path = "src/main.rs"

[lib]
name = "claude_keypad_svc"
path = "src/lib.rs"

[dependencies]
claude-keypad-common = { path = "../common" }
serde = { workspace = true }
serde_json = { workspace = true }
tracing = { workspace = true }
tracing-subscriber = { workspace = true }
anyhow = { workspace = true }
axum = "0.7"
tokio = { version = "1", features = ["rt-multi-thread", "macros", "signal"] }

[dev-dependencies]
tower = { version = "0.5", features = ["util"] }
http-body-util = "0.1"
```

`pc/svc/src/main.rs`:

```rust
fn main() {
    println!("claude-keypad-svc placeholder");
}
```

`pc/svc/src/lib.rs`:

```rust
//! claude-keypad-svc library entry — handlers, app builder, state.
```

- [ ] **Step 4: Create `hook` crate**

`pc/hook/Cargo.toml`:

```toml
[package]
name = "claude-keypad-hook"
edition.workspace = true
version.workspace = true
license.workspace = true
publish.workspace = true

[[bin]]
name = "claude-keypad-hook"
path = "src/main.rs"

[lib]
name = "claude_keypad_hook"
path = "src/lib.rs"

[dependencies]
claude-keypad-common = { path = "../common" }
serde = { workspace = true }
serde_json = { workspace = true }
anyhow = { workspace = true }
reqwest = { version = "0.12", default-features = false, features = ["blocking", "json", "rustls-tls"] }

[dev-dependencies]
tokio = { version = "1", features = ["rt-multi-thread", "macros", "net", "signal"] }
axum = "0.7"
```

`pc/hook/src/main.rs`:

```rust
fn main() {
    println!("claude-keypad-hook placeholder");
}
```

`pc/hook/src/lib.rs`:

```rust
//! claude-keypad-hook library entry — testable run() function.
```

- [ ] **Step 5: Create `.gitignore`**

`pc/.gitignore`:

```
/target
**/*.rs.bk
Cargo.lock
```

> Note: we ignore `Cargo.lock` for libraries but commit it for binaries. This workspace contains binaries (svc, hook), so we should commit it. **Override** by removing `Cargo.lock` from `.gitignore` after the first build:

Final `pc/.gitignore`:

```
/target
**/*.rs.bk
```

- [ ] **Step 6: Verify the workspace builds**

Run from `pc/`:

```
cargo build
```

Expected: builds 3 crates with no warnings, two binaries placed under `pc/target/debug/`.

- [ ] **Step 7: Commit**

```
git add pc/
git commit -m "chore(pc): cargo workspace skeleton with common/svc/hook crates"
```

---

## Task 2: Common crate — hook stdin and stdout schemas

**Files:**

- Modify: `pc/common/src/lib.rs`

The hook binary reads a JSON object on stdin (the schema Claude Code sends per the spec's reference) and writes a JSON object on stdout. Both schemas live in `common` so svc and hook agree.

- [ ] **Step 1: Write the failing test for `HookInput` deserialization**

Append to `pc/common/src/lib.rs`:

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Deserialize)]
pub struct HookInput {
    pub session_id: String,
    pub transcript_path: String,
    pub cwd: String,
    pub hook_event_name: String,
    pub permission_mode: Option<String>,
    pub tool_name: Option<String>,
    pub tool_input: Option<serde_json::Value>,
}

#[cfg(test)]
mod tests {
    use super::*;

    const SAMPLE_PRE_TOOL_USE: &str = r#"{
        "session_id": "abc123",
        "transcript_path": "C:/Users/me/.claude/projects/x/transcript.jsonl",
        "cwd": "C:/Users/me/proj",
        "permission_mode": "default",
        "hook_event_name": "PreToolUse",
        "tool_name": "Bash",
        "tool_input": {"command": "npm install"}
    }"#;

    #[test]
    fn deserializes_pre_tool_use_input() {
        let input: HookInput = serde_json::from_str(SAMPLE_PRE_TOOL_USE).unwrap();
        assert_eq!(input.session_id, "abc123");
        assert_eq!(input.tool_name.as_deref(), Some("Bash"));
        assert_eq!(
            input.tool_input.unwrap()["command"].as_str(),
            Some("npm install")
        );
    }
}
```

- [ ] **Step 2: Run the test, confirm it passes**

```
cargo test -p claude-keypad-common
```

Expected: 1 passed.

- [ ] **Step 3: Add `PermissionDecision` enum and `HookOutput` struct, with failing test**

Append to `pc/common/src/lib.rs`:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "lowercase")]
pub enum PermissionDecision {
    Allow,
    Deny,
    Ask,
}

#[derive(Debug, Serialize)]
pub struct HookSpecificOutput {
    #[serde(rename = "hookEventName")]
    pub hook_event_name: String,
    #[serde(rename = "permissionDecision")]
    pub permission_decision: PermissionDecision,
    #[serde(rename = "permissionDecisionReason", skip_serializing_if = "Option::is_none")]
    pub permission_decision_reason: Option<String>,
}

#[derive(Debug, Serialize)]
pub struct HookOutput {
    #[serde(rename = "hookSpecificOutput")]
    pub hook_specific_output: HookSpecificOutput,
}

impl HookOutput {
    pub fn ask() -> Self {
        Self {
            hook_specific_output: HookSpecificOutput {
                hook_event_name: "PreToolUse".into(),
                permission_decision: PermissionDecision::Ask,
                permission_decision_reason: None,
            },
        }
    }

    pub fn allow(reason: Option<String>) -> Self {
        Self {
            hook_specific_output: HookSpecificOutput {
                hook_event_name: "PreToolUse".into(),
                permission_decision: PermissionDecision::Allow,
                permission_decision_reason: reason,
            },
        }
    }

    pub fn deny(reason: Option<String>) -> Self {
        Self {
            hook_specific_output: HookSpecificOutput {
                hook_event_name: "PreToolUse".into(),
                permission_decision: PermissionDecision::Deny,
                permission_decision_reason: reason,
            },
        }
    }
}

#[cfg(test)]
mod output_tests {
    use super::*;

    #[test]
    fn ask_serializes_to_expected_json() {
        let s = serde_json::to_string(&HookOutput::ask()).unwrap();
        assert_eq!(
            s,
            r#"{"hookSpecificOutput":{"hookEventName":"PreToolUse","permissionDecision":"ask"}}"#
        );
    }

    #[test]
    fn allow_with_reason_includes_reason() {
        let s = serde_json::to_string(&HookOutput::allow(Some("user pressed key 1".into()))).unwrap();
        assert!(s.contains(r#""permissionDecision":"allow""#));
        assert!(s.contains(r#""permissionDecisionReason":"user pressed key 1""#));
    }

    #[test]
    fn deny_serializes() {
        let s = serde_json::to_string(&HookOutput::deny(None)).unwrap();
        assert!(s.contains(r#""permissionDecision":"deny""#));
        assert!(!s.contains("permissionDecisionReason"));
    }
}
```

- [ ] **Step 4: Run all common tests**

```
cargo test -p claude-keypad-common
```

Expected: 4 passed (input deserialization + 3 output tests).

- [ ] **Step 5: Commit**

```
git add pc/common/
git commit -m "feat(common): hook input + output schema with serde tests"
```

---

## Task 3: Common crate — service HTTP API types

**Files:**

- Modify: `pc/common/src/lib.rs`

Hook ↔ svc speak a small JSON HTTP API. Define the request/response shapes here so both sides cannot drift.

- [ ] **Step 1: Add `ServicePermissionRequest` and `ServicePermissionResponse`**

Append to `pc/common/src/lib.rs`:

```rust
#[derive(Debug, Serialize, Deserialize)]
pub struct ServicePermissionRequest {
    pub session_id: String,
    pub cwd: String,
    pub tool_name: String,
    pub tool_input: serde_json::Value,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub permission_mode: Option<String>,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct ServicePermissionResponse {
    pub decision: PermissionDecision,
    #[serde(default)]
    pub remember: bool,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub reason: Option<String>,
}

#[derive(Debug, Serialize, Deserialize)]
pub struct ServiceHealth {
    pub status: String,
    pub version: String,
}
```

- [ ] **Step 2: Add a roundtrip test**

Append to `pc/common/src/lib.rs` (inside `#[cfg(test)]` block or as a new one):

```rust
#[cfg(test)]
mod api_tests {
    use super::*;

    #[test]
    fn permission_request_roundtrip() {
        let req = ServicePermissionRequest {
            session_id: "s".into(),
            cwd: "/tmp".into(),
            tool_name: "Bash".into(),
            tool_input: serde_json::json!({"command": "echo hi"}),
            permission_mode: Some("default".into()),
        };
        let s = serde_json::to_string(&req).unwrap();
        let back: ServicePermissionRequest = serde_json::from_str(&s).unwrap();
        assert_eq!(back.tool_name, "Bash");
    }

    #[test]
    fn permission_response_defaults_remember_false() {
        let s = r#"{"decision":"allow"}"#;
        let r: ServicePermissionResponse = serde_json::from_str(s).unwrap();
        assert!(!r.remember);
        assert_eq!(r.decision, PermissionDecision::Allow);
    }
}
```

- [ ] **Step 3: Run tests**

```
cargo test -p claude-keypad-common
```

Expected: 6 passed.

- [ ] **Step 4: Commit**

```
git add pc/common/
git commit -m "feat(common): service HTTP API types"
```

---

## Task 4: Service — app builder + `/health` endpoint with TDD

**Files:**

- Create: `pc/svc/src/handlers.rs`
- Modify: `pc/svc/src/lib.rs`

The svc binary's logic lives in the `lib.rs` (testable) and is called from `main.rs` (the binary entrypoint). We extract a `build_app()` function so tests can call the router with `tower::ServiceExt::oneshot()`.

- [ ] **Step 1: Write the failing test for `/health`**

Create `pc/svc/src/lib.rs`:

```rust
use axum::Router;
use std::sync::Arc;

pub mod handlers;

#[derive(Clone)]
pub struct AppState {
    pub version: String,
}

impl Default for AppState {
    fn default() -> Self {
        Self { version: env!("CARGO_PKG_VERSION").to_string() }
    }
}

pub fn build_app(state: AppState) -> Router {
    Router::new()
        .route("/health", axum::routing::get(handlers::health))
        .with_state(Arc::new(state))
}

#[cfg(test)]
mod tests {
    use super::*;
    use axum::body::Body;
    use axum::http::{Request, StatusCode};
    use http_body_util::BodyExt;
    use tower::ServiceExt;

    #[tokio::test]
    async fn health_returns_200_with_status_ok() {
        let app = build_app(AppState::default());
        let response = app
            .oneshot(Request::builder().uri("/health").body(Body::empty()).unwrap())
            .await
            .unwrap();
        assert_eq!(response.status(), StatusCode::OK);
        let body = response.into_body().collect().await.unwrap().to_bytes();
        let json: serde_json::Value = serde_json::from_slice(&body).unwrap();
        assert_eq!(json["status"], "ok");
        assert!(json["version"].as_str().unwrap().contains('.'));
    }
}
```

Create `pc/svc/src/handlers.rs`:

```rust
use axum::{extract::State, Json};
use claude_keypad_common::ServiceHealth;
use std::sync::Arc;

use crate::AppState;

pub async fn health(State(state): State<Arc<AppState>>) -> Json<ServiceHealth> {
    Json(ServiceHealth {
        status: "ok".into(),
        version: state.version.clone(),
    })
}
```

- [ ] **Step 2: Run the test**

```
cargo test -p claude-keypad-svc
```

Expected: 1 passed.

- [ ] **Step 3: Commit**

```
git add pc/svc/
git commit -m "feat(svc): /health endpoint and testable app builder"
```

---

## Task 5: Service — `/permission` POST endpoint (stub: always allow + log)

**Files:**

- Modify: `pc/svc/src/handlers.rs`
- Modify: `pc/svc/src/lib.rs`

Phase 1 stub: the endpoint always returns `decision: "allow"` and logs the inbound request. Later phases plug in screen + keyboard logic.

- [ ] **Step 1: Write the failing test**

Append to `pc/svc/src/lib.rs`'s `tests` module:

```rust
    #[tokio::test]
    async fn permission_endpoint_returns_allow_for_any_request() {
        let app = build_app(AppState::default());
        let req_body = serde_json::json!({
            "session_id": "s1",
            "cwd": "/tmp",
            "tool_name": "Bash",
            "tool_input": {"command": "echo hi"}
        });
        let response = app
            .oneshot(
                Request::builder()
                    .method("POST")
                    .uri("/permission")
                    .header("content-type", "application/json")
                    .body(Body::from(serde_json::to_vec(&req_body).unwrap()))
                    .unwrap(),
            )
            .await
            .unwrap();
        assert_eq!(response.status(), StatusCode::OK);
        let body = response.into_body().collect().await.unwrap().to_bytes();
        let json: serde_json::Value = serde_json::from_slice(&body).unwrap();
        assert_eq!(json["decision"], "allow");
        assert_eq!(json["remember"], false);
    }
```

- [ ] **Step 2: Run, observe failure**

```
cargo test -p claude-keypad-svc permission_endpoint_returns_allow_for_any_request
```

Expected: FAIL — 404 (no route).

- [ ] **Step 3: Add the handler and route**

Append to `pc/svc/src/handlers.rs`:

```rust
use claude_keypad_common::{
    PermissionDecision, ServicePermissionRequest, ServicePermissionResponse,
};
use tracing::info;

pub async fn permission(
    State(_state): State<Arc<AppState>>,
    Json(req): Json<ServicePermissionRequest>,
) -> Json<ServicePermissionResponse> {
    info!(
        session = %req.session_id,
        tool = %req.tool_name,
        cwd = %req.cwd,
        "permission request (stub: auto-allow)"
    );
    info!(input = %serde_json::to_string(&req.tool_input).unwrap_or_default(), "tool_input");
    Json(ServicePermissionResponse {
        decision: PermissionDecision::Allow,
        remember: false,
        reason: Some("phase1 auto-allow".into()),
    })
}
```

Modify `build_app` in `pc/svc/src/lib.rs`:

```rust
pub fn build_app(state: AppState) -> Router {
    Router::new()
        .route("/health", axum::routing::get(handlers::health))
        .route("/permission", axum::routing::post(handlers::permission))
        .with_state(Arc::new(state))
}
```

- [ ] **Step 4: Run the test, confirm pass**

```
cargo test -p claude-keypad-svc
```

Expected: 2 passed.

- [ ] **Step 5: Commit**

```
git add pc/svc/
git commit -m "feat(svc): /permission stub returns allow and logs request"
```

---

## Task 6: Service — `/session-start` and `/session-stop` log-only endpoints

**Files:**

- Modify: `pc/svc/src/handlers.rs`
- Modify: `pc/svc/src/lib.rs`

These are wired up early so Phase 4 (idle screen) doesn't need to retrofit them. v1 just logs.

- [ ] **Step 1: Write the failing test**

Append to `tests` module in `pc/svc/src/lib.rs`:

```rust
    #[tokio::test]
    async fn session_endpoints_return_204() {
        for path in ["/session-start", "/session-stop"] {
            let app = build_app(AppState::default());
            let response = app
                .oneshot(
                    Request::builder()
                        .method("POST")
                        .uri(path)
                        .header("content-type", "application/json")
                        .body(Body::from(b"{}".to_vec()))
                        .unwrap(),
                )
                .await
                .unwrap();
            assert_eq!(response.status(), StatusCode::NO_CONTENT, "path={}", path);
        }
    }
```

- [ ] **Step 2: Run, observe failure**

```
cargo test -p claude-keypad-svc session_endpoints_return_204
```

Expected: FAIL.

- [ ] **Step 3: Add handlers and routes**

Append to `pc/svc/src/handlers.rs`:

```rust
use axum::http::StatusCode;

pub async fn session_event(Json(body): Json<serde_json::Value>) -> StatusCode {
    info!(body = %body, "session event");
    StatusCode::NO_CONTENT
}
```

Modify `build_app`:

```rust
pub fn build_app(state: AppState) -> Router {
    Router::new()
        .route("/health", axum::routing::get(handlers::health))
        .route("/permission", axum::routing::post(handlers::permission))
        .route("/session-start", axum::routing::post(handlers::session_event))
        .route("/session-stop", axum::routing::post(handlers::session_event))
        .with_state(Arc::new(state))
}
```

- [ ] **Step 4: Run, confirm pass**

```
cargo test -p claude-keypad-svc
```

Expected: 3 passed.

- [ ] **Step 5: Commit**

```
git add pc/svc/
git commit -m "feat(svc): session-start/stop/event endpoints (log-only)"
```

---

## Task 7: Service — main entrypoint binds to 127.0.0.1:48420 with logging

**Files:**

- Modify: `pc/svc/src/main.rs`

- [ ] **Step 1: Replace `pc/svc/src/main.rs`**

```rust
use anyhow::Result;
use claude_keypad_svc::{build_app, AppState};
use tracing_subscriber::{fmt, EnvFilter};

const BIND_ADDR: &str = "127.0.0.1:48420";

#[tokio::main]
async fn main() -> Result<()> {
    fmt()
        .with_env_filter(
            EnvFilter::try_from_default_env().unwrap_or_else(|_| EnvFilter::new("info")),
        )
        .init();

    let app = build_app(AppState::default());
    let listener = tokio::net::TcpListener::bind(BIND_ADDR).await?;
    tracing::info!(addr = BIND_ADDR, "claude-keypad-svc listening");
    axum::serve(listener, app).await?;
    Ok(())
}
```

- [ ] **Step 2: Run the binary manually**

In one shell from `pc/`:

```
cargo run -p claude-keypad-svc
```

Expected: log line `claude-keypad-svc listening addr="127.0.0.1:48420"`.

In a second shell:

```
curl -s http://127.0.0.1:48420/health
```

Expected: `{"status":"ok","version":"0.1.0"}`.

```
curl -s -X POST http://127.0.0.1:48420/permission -H "content-type: application/json" -d "{\"session_id\":\"x\",\"cwd\":\"/tmp\",\"tool_name\":\"Bash\",\"tool_input\":{\"command\":\"ls\"}}"
```

Expected: `{"decision":"allow","remember":false,"reason":"phase1 auto-allow"}`.

Stop the service with Ctrl+C.

- [ ] **Step 3: Commit**

```
git add pc/svc/
git commit -m "feat(svc): bind to 127.0.0.1:48420 and log requests"
```

---

## Task 8: Hook — extract testable `run()` and parse stdin

**Files:**

- Modify: `pc/hook/src/lib.rs`
- Modify: `pc/hook/src/main.rs`

We design hook so the entire logic is in `lib.rs::run(stdin, stdout)` taking `impl Read` / `impl Write`. `main.rs` is a 5-line shim.

- [ ] **Step 1: Write the failing test for stdin parsing → ask path on malformed input**

Replace `pc/hook/src/lib.rs`:

```rust
use anyhow::Result;
use claude_keypad_common::{HookInput, HookOutput};
use std::io::{Read, Write};

const HOOK_EVENT_PRE_TOOL_USE: &str = "PreToolUse";

/// Top-level entrypoint. Always exits successfully (returns Ok) so Claude Code
/// never sees exit code 2 / the hook never blocks the workflow.
pub fn run(mut stdin: impl Read, mut stdout: impl Write) -> Result<()> {
    let mut buf = String::new();
    if stdin.read_to_string(&mut buf).is_err() {
        return write_output(&mut stdout, &HookOutput::ask());
    }

    let input: HookInput = match serde_json::from_str(&buf) {
        Ok(v) => v,
        Err(_) => return write_output(&mut stdout, &HookOutput::ask()),
    };

    // For non-PreToolUse events (Stop, SessionStart), we don't need a permission
    // decision — return ask anyway since Claude Code ignores the field for those.
    if input.hook_event_name != HOOK_EVENT_PRE_TOOL_USE {
        return write_output(&mut stdout, &HookOutput::ask());
    }

    // TODO Task 9-11: probe service, POST /permission, parse response.
    // For now, default to ask so the test passes.
    write_output(&mut stdout, &HookOutput::ask())
}

fn write_output(stdout: &mut impl Write, out: &HookOutput) -> Result<()> {
    let s = serde_json::to_string(out)?;
    stdout.write_all(s.as_bytes())?;
    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::io::Cursor;

    #[test]
    fn malformed_stdin_outputs_ask() {
        let stdin = Cursor::new(b"not json".to_vec());
        let mut stdout = Vec::new();
        run(stdin, &mut stdout).unwrap();
        let s = String::from_utf8(stdout).unwrap();
        assert!(s.contains(r#""permissionDecision":"ask""#));
    }

    #[test]
    fn non_pre_tool_use_event_outputs_ask() {
        let stdin = Cursor::new(
            br#"{"session_id":"s","transcript_path":"x","cwd":"/","hook_event_name":"Stop"}"#
                .to_vec(),
        );
        let mut stdout = Vec::new();
        run(stdin, &mut stdout).unwrap();
        let s = String::from_utf8(stdout).unwrap();
        assert!(s.contains(r#""permissionDecision":"ask""#));
    }
}
```

Replace `pc/hook/src/main.rs`:

```rust
use anyhow::Result;
use std::io::{stdin, stdout};

fn main() -> Result<()> {
    claude_keypad_hook::run(stdin().lock(), stdout().lock())
}
```

- [ ] **Step 2: Run the tests**

```
cargo test -p claude-keypad-hook
```

Expected: 2 passed.

- [ ] **Step 3: Commit**

```
git add pc/hook/
git commit -m "feat(hook): testable run() with malformed-input → ask fallback"
```

---

## Task 9: Hook — call svc `/health`, fall back to ask if unreachable

**Files:**

- Modify: `pc/hook/src/lib.rs`

Use `reqwest::blocking` for simplicity. 50 ms timeout: if the service is up, this is plenty; if it's not, we fail fast and let Claude Code show its native UI.

- [ ] **Step 1: Refactor `run` to take a `HookConfig` so tests can point at a stubbed server**

In `pc/hook/src/lib.rs`, **replace** the existing `run` function from Task 8 with the two functions below (`run` becomes a thin wrapper that delegates to `run_with_config`). Add the `HookConfig` struct and the `service_alive` helper. Keep `write_output` and the `HOOK_EVENT_PRE_TOOL_USE` constant from Task 8.

```rust
use std::time::Duration;

pub struct HookConfig {
    pub service_base_url: String,
    pub health_timeout: Duration,
    pub permission_timeout: Duration,
}

impl Default for HookConfig {
    fn default() -> Self {
        Self {
            service_base_url: "http://127.0.0.1:48420".into(),
            health_timeout: Duration::from_millis(50),
            permission_timeout: Duration::from_secs(310),
        }
    }
}

pub fn run(stdin: impl Read, stdout: impl Write) -> Result<()> {
    run_with_config(stdin, stdout, &HookConfig::default())
}

pub fn run_with_config(
    mut stdin: impl Read,
    mut stdout: impl Write,
    cfg: &HookConfig,
) -> Result<()> {
    // ... move existing logic here ...
    let mut buf = String::new();
    if stdin.read_to_string(&mut buf).is_err() {
        return write_output(&mut stdout, &HookOutput::ask());
    }

    let input: HookInput = match serde_json::from_str(&buf) {
        Ok(v) => v,
        Err(_) => return write_output(&mut stdout, &HookOutput::ask()),
    };

    if input.hook_event_name != HOOK_EVENT_PRE_TOOL_USE {
        return write_output(&mut stdout, &HookOutput::ask());
    }

    if !service_alive(cfg) {
        return write_output(&mut stdout, &HookOutput::ask());
    }

    // TODO Task 10: POST /permission.
    write_output(&mut stdout, &HookOutput::ask())
}

fn service_alive(cfg: &HookConfig) -> bool {
    let client = match reqwest::blocking::Client::builder()
        .timeout(cfg.health_timeout)
        .build()
    {
        Ok(c) => c,
        Err(_) => return false,
    };
    client
        .get(format!("{}/health", cfg.service_base_url))
        .send()
        .map(|r| r.status().is_success())
        .unwrap_or(false)
}
```

- [ ] **Step 2: Add tests using a fake axum service**

Append to `tests` module in `pc/hook/src/lib.rs`:

```rust
    use axum::{routing::get, Router};
    use std::net::TcpListener;
    use tokio::runtime::Runtime;

    fn spawn_test_service<F>(make_router: F) -> (String, std::thread::JoinHandle<()>)
    where
        F: FnOnce() -> Router + Send + 'static,
    {
        let listener = TcpListener::bind("127.0.0.1:0").unwrap();
        let addr = listener.local_addr().unwrap();
        let url = format!("http://{}", addr);
        let handle = std::thread::spawn(move || {
            let rt = Runtime::new().unwrap();
            rt.block_on(async move {
                let listener = tokio::net::TcpListener::from_std(listener).unwrap();
                let app = make_router();
                axum::serve(listener, app).await.unwrap();
            });
        });
        (url, handle)
    }

    #[test]
    fn service_unreachable_outputs_ask() {
        let cfg = HookConfig {
            service_base_url: "http://127.0.0.1:1".into(), // closed port
            health_timeout: Duration::from_millis(50),
            permission_timeout: Duration::from_secs(1),
        };
        let stdin = Cursor::new(
            br#"{"session_id":"s","transcript_path":"x","cwd":"/","hook_event_name":"PreToolUse","tool_name":"Bash","tool_input":{"command":"ls"}}"#
                .to_vec(),
        );
        let mut stdout = Vec::new();
        run_with_config(stdin, &mut stdout, &cfg).unwrap();
        assert!(String::from_utf8(stdout).unwrap().contains(r#""permissionDecision":"ask""#));
    }

    #[test]
    fn service_alive_but_no_permission_endpoint_yet_still_outputs_ask_phase1() {
        let (base, _h) = spawn_test_service(|| Router::new().route("/health", get(|| async { "ok" })));
        std::thread::sleep(Duration::from_millis(50));

        let cfg = HookConfig {
            service_base_url: base,
            health_timeout: Duration::from_millis(200),
            permission_timeout: Duration::from_secs(1),
        };
        let stdin = Cursor::new(
            br#"{"session_id":"s","transcript_path":"x","cwd":"/","hook_event_name":"PreToolUse","tool_name":"Bash","tool_input":{"command":"ls"}}"#
                .to_vec(),
        );
        let mut stdout = Vec::new();
        run_with_config(stdin, &mut stdout, &cfg).unwrap();
        // Currently still returns ask because Task 10 hasn't wired POST yet.
        assert!(String::from_utf8(stdout).unwrap().contains(r#""permissionDecision":"ask""#));
    }
```

- [ ] **Step 3: Run tests**

```
cargo test -p claude-keypad-hook
```

Expected: 4 passed.

- [ ] **Step 4: Commit**

```
git add pc/hook/
git commit -m "feat(hook): probe svc /health with 50ms timeout; ask on unreachable"
```

---

## Task 10: Hook — POST `/permission`, parse decision, output

**Files:**

- Modify: `pc/hook/src/lib.rs`

- [ ] **Step 1: Replace the TODO branch with the real POST**

In `pc/hook/src/lib.rs`, modify `run_with_config` body, replacing the TODO line and adding a helper:

```rust
    if !service_alive(cfg) {
        return write_output(&mut stdout, &HookOutput::ask());
    }

    let req = ServicePermissionRequest {
        session_id: input.session_id.clone(),
        cwd: input.cwd.clone(),
        tool_name: input.tool_name.clone().unwrap_or_default(),
        tool_input: input.tool_input.clone().unwrap_or(serde_json::Value::Null),
        permission_mode: input.permission_mode.clone(),
    };

    let decision = match query_decision(cfg, &req) {
        Ok(d) => d,
        Err(_) => return write_output(&mut stdout, &HookOutput::ask()),
    };

    let out = match decision.decision {
        PermissionDecision::Allow => HookOutput::allow(decision.reason),
        PermissionDecision::Deny => HookOutput::deny(decision.reason),
        PermissionDecision::Ask => HookOutput::ask(),
    };
    write_output(&mut stdout, &out)
}

fn query_decision(
    cfg: &HookConfig,
    req: &ServicePermissionRequest,
) -> Result<ServicePermissionResponse> {
    let client = reqwest::blocking::Client::builder()
        .timeout(cfg.permission_timeout)
        .build()?;
    let resp = client
        .post(format!("{}/permission", cfg.service_base_url))
        .json(req)
        .send()?;
    Ok(resp.error_for_status()?.json::<ServicePermissionResponse>()?)
}
```

Add the missing imports at the top:

```rust
use claude_keypad_common::{
    HookInput, HookOutput, PermissionDecision, ServicePermissionRequest, ServicePermissionResponse,
};
```

- [ ] **Step 2: Rename Task 9's `service_alive_but_no_permission_endpoint_yet_still_outputs_ask_phase1` to `permission_endpoint_error_outputs_ask`**

The test from Task 9 expected `ask` because the endpoint was missing. After Task 10's wiring, the same scenario (404 on `/permission`) should still produce `ask` — the test stays semantically correct, but the name no longer reflects intent.

**Delete** the old test and **replace** it with this one (same body, new name + clearer comment):

```rust
    #[test]
    fn permission_endpoint_error_outputs_ask() {
        let (base, _h) = spawn_test_service(|| {
            Router::new()
                .route("/health", get(|| async { "ok" }))
                // No /permission route → 404
        });
        std::thread::sleep(Duration::from_millis(50));

        let cfg = HookConfig {
            service_base_url: base,
            health_timeout: Duration::from_millis(200),
            permission_timeout: Duration::from_secs(1),
        };
        let stdin = Cursor::new(
            br#"{"session_id":"s","transcript_path":"x","cwd":"/","hook_event_name":"PreToolUse","tool_name":"Bash","tool_input":{"command":"ls"}}"#
                .to_vec(),
        );
        let mut stdout = Vec::new();
        run_with_config(stdin, &mut stdout, &cfg).unwrap();
        assert!(String::from_utf8(stdout).unwrap().contains(r#""permissionDecision":"ask""#));
    }
```

- [ ] **Step 3: Add a test where the service returns "allow"**

Append:

```rust
    use axum::routing::post;
    use axum::Json;

    #[test]
    fn permission_allow_response_outputs_allow() {
        let (base, _h) = spawn_test_service(|| {
            Router::new()
                .route("/health", get(|| async { "ok" }))
                .route(
                    "/permission",
                    post(|Json(_): Json<serde_json::Value>| async {
                        Json(serde_json::json!({
                            "decision": "allow",
                            "remember": false,
                            "reason": "test"
                        }))
                    }),
                )
        });
        std::thread::sleep(Duration::from_millis(50));

        let cfg = HookConfig {
            service_base_url: base,
            health_timeout: Duration::from_millis(200),
            permission_timeout: Duration::from_secs(2),
        };
        let stdin = Cursor::new(
            br#"{"session_id":"s","transcript_path":"x","cwd":"/","hook_event_name":"PreToolUse","tool_name":"Bash","tool_input":{"command":"ls"}}"#
                .to_vec(),
        );
        let mut stdout = Vec::new();
        run_with_config(stdin, &mut stdout, &cfg).unwrap();
        let out = String::from_utf8(stdout).unwrap();
        assert!(out.contains(r#""permissionDecision":"allow""#), "got: {}", out);
    }

    #[test]
    fn permission_deny_response_outputs_deny() {
        let (base, _h) = spawn_test_service(|| {
            Router::new()
                .route("/health", get(|| async { "ok" }))
                .route(
                    "/permission",
                    post(|Json(_): Json<serde_json::Value>| async {
                        Json(serde_json::json!({
                            "decision": "deny",
                            "remember": false,
                            "reason": "user pressed key 3"
                        }))
                    }),
                )
        });
        std::thread::sleep(Duration::from_millis(50));

        let cfg = HookConfig {
            service_base_url: base,
            health_timeout: Duration::from_millis(200),
            permission_timeout: Duration::from_secs(2),
        };
        let stdin = Cursor::new(
            br#"{"session_id":"s","transcript_path":"x","cwd":"/","hook_event_name":"PreToolUse","tool_name":"Bash","tool_input":{"command":"ls"}}"#
                .to_vec(),
        );
        let mut stdout = Vec::new();
        run_with_config(stdin, &mut stdout, &cfg).unwrap();
        let out = String::from_utf8(stdout).unwrap();
        assert!(out.contains(r#""permissionDecision":"deny""#), "got: {}", out);
    }
```

- [ ] **Step 4: Run all hook tests**

```
cargo test -p claude-keypad-hook
```

Expected: 6 passed.

- [ ] **Step 5: Commit**

```
git add pc/hook/
git commit -m "feat(hook): POST /permission and forward decision; ask on any error"
```

---

## Task 11: End-to-end smoke test (svc + hook in process)

**Files:**

- Create: `pc/hook/tests/end_to_end.rs`

This integration test spins up the **real** svc and runs the **real** hook logic against it, verifying the full path returns `allow` (matching svc Task 5's stub).

- [ ] **Step 1: Write the test**

`pc/hook/tests/end_to_end.rs`:

```rust
use claude_keypad_hook::{run_with_config, HookConfig};
use claude_keypad_svc::{build_app, AppState};
use std::io::Cursor;
use std::net::TcpListener;
use std::time::Duration;
use tokio::runtime::Runtime;

#[test]
fn full_flow_returns_allow() {
    let listener = TcpListener::bind("127.0.0.1:0").unwrap();
    let addr = listener.local_addr().unwrap();
    let url = format!("http://{}", addr);

    let _server = std::thread::spawn(move || {
        let rt = Runtime::new().unwrap();
        rt.block_on(async move {
            let listener = tokio::net::TcpListener::from_std(listener).unwrap();
            let app = build_app(AppState::default());
            axum::serve(listener, app).await.unwrap();
        });
    });
    std::thread::sleep(Duration::from_millis(150));

    let cfg = HookConfig {
        service_base_url: url,
        health_timeout: Duration::from_millis(500),
        permission_timeout: Duration::from_secs(5),
    };

    let stdin = Cursor::new(
        br#"{
        "session_id":"s",
        "transcript_path":"x",
        "cwd":"/",
        "hook_event_name":"PreToolUse",
        "tool_name":"Bash",
        "tool_input":{"command":"echo hi"}
    }"#
        .to_vec(),
    );
    let mut stdout = Vec::new();
    run_with_config(stdin, &mut stdout, &cfg).unwrap();
    let out = String::from_utf8(stdout).unwrap();
    assert!(out.contains(r#""permissionDecision":"allow""#), "got: {}", out);
}
```

Add `claude-keypad-svc` to hook's `[dev-dependencies]` so the test can use it. Modify `pc/hook/Cargo.toml`:

```toml
[dev-dependencies]
tokio = { version = "1", features = ["rt-multi-thread", "macros", "net", "signal"] }
axum = "0.7"
claude-keypad-svc = { path = "../svc" }
```

- [ ] **Step 2: Run the integration test**

```
cargo test -p claude-keypad-hook --test end_to_end
```

Expected: 1 passed.

- [ ] **Step 3: Commit**

```
git add pc/hook/
git commit -m "test(hook): end-to-end smoke test against real svc"
```

---

## Task 12: Build release binaries and install to PATH

**Files:** none (build artifacts only)

- [ ] **Step 1: Build release**

From `pc/`:

```
cargo build --release
```

Expected: `pc/target/release/claude-keypad-svc.exe` and `claude-keypad-hook.exe` produced. Build completes with no warnings.

- [ ] **Step 2: Install both binaries to `~/.cargo/bin`**

```
cargo install --path svc --force
cargo install --path hook --force
```

Expected: cargo copies both binaries to `%USERPROFILE%\.cargo\bin`. That directory is on PATH (rustup default).

- [ ] **Step 3: Verify availability from a fresh shell**

Open a new shell:

```
where claude-keypad-svc
where claude-keypad-hook
```

Expected: both resolve to `%USERPROFILE%\.cargo\bin\*.exe`.

- [ ] **Step 4: No commit needed (no source changes).**

---

## Task 13: Configure Claude Code to use the hook

**Files:**

- Modify: `%USERPROFILE%\.claude\settings.json`

We add a `PreToolUse` hook that runs `claude-keypad-hook`. We do NOT yet wire `Stop` / `SessionStart` (those are Phase 4).

- [ ] **Step 1: Inspect existing settings**

```
type %USERPROFILE%\.claude\settings.json
```

If the file does not exist or is empty, create it with `{}`.

- [ ] **Step 2: Add the hook entry**

Open the file in your editor. Merge in (creating the `hooks.PreToolUse` array if absent):

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": ".*",
        "hooks": [
          {
            "type": "command",
            "command": "claude-keypad-hook",
            "timeout": 320
          }
        ]
      }
    ]
  }
}
```

If the file already contains other top-level fields (model, theme, etc.), preserve them — only the `hooks` block is owned by us.

- [ ] **Step 3: Validate JSON**

```
type %USERPROFILE%\.claude\settings.json | python -c "import json,sys; json.load(sys.stdin); print('ok')"
```

Expected: `ok`. If you don't have python, paste the file content into any JSON validator.

- [ ] **Step 4: No commit needed (file is outside repo).**

---

## Task 14: Manual end-to-end verification with real Claude Code

**Files:** none

This is the Phase 1 acceptance test.

- [ ] **Step 1: Start the svc in a terminal**

```
claude-keypad-svc
```

Expected: log line `claude-keypad-svc listening addr="127.0.0.1:48420"`. Leave it running.

- [ ] **Step 2: In a second terminal, start Claude Code in any project**

```
claude
```

- [ ] **Step 3: Trigger a tool call**

In the Claude Code session, ask:

> "Run `echo hello-claude-keypad` in the shell."

Expected behavior:

- The svc terminal prints a log line containing `permission request (stub: auto-allow)` with `tool=Bash` and the command in `tool_input`.
- Claude Code does **not** show a permission prompt — it just runs the command.
- The command output `hello-claude-keypad` appears in the Claude Code session.

- [ ] **Step 4: Verify graceful fallback**

Stop the svc (Ctrl+C in its terminal). Ask Claude Code to run another command, e.g.:

> "Run `echo fallback-test` in the shell."

Expected behavior:

- Claude Code shows its **native** permission prompt (the original 1/2/3 menu).
- Choosing 1 (Allow once) executes the command normally.

This proves the graceful degradation path. Both halves of acceptance pass: hook works when service is up, falls back cleanly when service is down.

- [ ] **Step 5: No commit (verification only).**

---

## Task 15: Phase 1 README and final commit

**Files:**

- Create: `pc/README.md`
- Create or modify: `README.md` (project root)

- [ ] **Step 1: Write `pc/README.md`**

```markdown
# claude-keypad — PC Software (Phase 1)

Rust workspace with three crates:

- `common` — shared types (hook IO schema, service HTTP types).
- `svc` — `claude-keypad-svc`, the always-on background service. Phase 1 stub: auto-allows every PreToolUse request and logs it.
- `hook` — `claude-keypad-hook`, the Claude Code hook entrypoint. Talks to svc; falls back to "ask" on any error.

## Build

Requires Rust 1.76+ stable.

\`\`\`
cargo build --release
\`\`\`

## Install to PATH

\`\`\`
cargo install --path svc --force
cargo install --path hook --force
\`\`\`

Both binaries land in `%USERPROFILE%\.cargo\bin`, which is on PATH after `rustup` setup.

## Configure Claude Code

Add to `%USERPROFILE%\.claude\settings.json`:

\`\`\`json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": ".*",
        "hooks": [
          { "type": "command", "command": "claude-keypad-hook", "timeout": 320 }
        ]
      }
    ]
  }
}
\`\`\`

## Run

In one terminal:

\`\`\`
claude-keypad-svc
\`\`\`

Leave it running. Then start `claude` normally in another terminal — every tool call will be logged and auto-allowed.

If the service isn't running, Claude Code falls back to its native permission UI automatically.

## Tests

\`\`\`
cargo test
\`\`\`

Includes:

- Schema serde tests in `common`.
- In-process router tests in `svc` (`tower::ServiceExt::oneshot`).
- Hook unit tests with stubbed services.
- End-to-end smoke test in `hook/tests/end_to_end.rs` running real svc + real hook.

## Phase 1 scope

Only the Hook ↔ Service path is implemented. Screen and keyboard come in Phase 2 and 3 respectively. See `docs/superpowers/specs/2026-05-08-claude-keypad-design.md` for the full v1 design.
```

- [ ] **Step 2: Write project root `README.md`** (if not present)

```markdown
# Claude Keypad

A hardware companion for Claude Code: 4-key QMK keypad + 320×480 USB LCD + PC service.
The service intercepts Claude Code's PreToolUse permission requests via the official
hook API, displays the prompt on a dedicated screen, and lets you accept/deny with a
satisfying physical click — no terminal context-switch.

## Status

Phase 1 (PC service + hook integration): in progress. See `docs/superpowers/plans/`.

## Layout

- `docs/superpowers/specs/` — design specs.
- `docs/superpowers/plans/` — implementation plans.
- `pc/` — Rust workspace for the PC software.
- `firmware/` — ESP32-S3 firmware for the LCD module (Phase 2).
- `keyboard/` — QMK config for the keypad (Phase 3).
```

- [ ] **Step 3: Commit and push**

```
git add pc/README.md README.md
git commit -m "docs: project + pc readmes for phase 1"
git push
```

---

## Phase 1 Definition of Done

All boxes below must be checked before declaring Phase 1 complete and starting Phase 2:

- [ ] `cargo build --release` succeeds with zero warnings on Windows.
- [ ] `cargo test` passes all tests across all three crates (common, svc, hook).
- [ ] `claude-keypad-svc` logs a request when Claude Code calls a tool, and Claude proceeds without showing a native permission prompt.
- [ ] Stopping the service makes Claude Code fall back to the native prompt without errors or stalls.
- [ ] `pc/README.md` and project `README.md` exist and reflect actual usage.
- [ ] All Phase 1 commits are pushed to the GitHub remote.

After this, write `docs/superpowers/plans/<date>-claude-keypad-phase2-screen.md` for the screen firmware.
