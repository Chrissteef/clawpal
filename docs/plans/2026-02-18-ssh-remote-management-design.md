# SSH Remote OpenClaw Management — Design

## Overview

Allow ClawPal to manage remote openclaw instances on VPS servers via SSH, providing the same full management capabilities as local (config editing, session management, agent/channel management, gateway restart, etc.).

## Key Decisions

| Decision | Choice |
|---|---|
| Management scope | Full mirror — same capabilities as local |
| SSH auth | Mixed — read `~/.ssh/config` first, also support manual config (host + key) |
| Execution method | SFTP for file operations + SSH exec for CLI commands |
| Multi-instance | Parallel — multiple remotes connected simultaneously |
| UI layout | Top tab bar (Local / Remote-A / Remote-B / +) |
| Connection lifecycle | On-demand — connect when switching to tab, auto-disconnect on idle (5min), auto-reconnect on next operation |
| Remote openclaw path | Default `~/.openclaw`, not configurable |

## Architecture

```
UI (React)
  ├── Instance Tab Bar
  └── Page Components (Home, Channels, Doctor, ...)
        │
        ▼
  API Layer (api.ts)
        │
        ▼
  Connection Layer
  ├── LocalBackend    — existing logic, direct Tauri commands
  └── RemoteBackend   — same interface, via SSH exec + SFTP
        │
        ▼
  Tauri Rust Backend
  ├── Local commands   — existing, unchanged
  └── SSH commands     — new: ssh_exec, sftp_read, sftp_write, etc.
```

- Rust layer provides SSH/SFTP low-level primitives.
- Remote business commands combine these primitives in Rust, returning the same types as local commands.
- Frontend uses a context-based API router — pages don't know if they're operating on local or remote.

## Remote Instance Configuration

```typescript
interface RemoteInstance {
  id: string;           // auto-generated unique ID
  label: string;        // display name, e.g. "Production VPS"
  host: string;         // hostname or SSH config alias
  port: number;         // default 22
  username: string;
  authMethod: "key" | "ssh_config";
  keyPath?: string;     // private key path (when authMethod is "key")
}
```

- Persisted in `.clawpal/remote-instances.json`.
- No passwords/passphrases stored — rely on ssh-agent or prompt user.
- If `host` matches a `~/.ssh/config` Host entry, inherit its settings.

## SSH Connection Management

**Rust crate:** `russh` + `russh-sftp` (pure Rust async SSH implementation, cross-platform).

**Connection pool:** Global `Mutex<HashMap<String, SshSession>>` in Rust, keyed by instance ID. Auto-reconnect on stale connections.

**Tab status indicators:**
- Gray dot: not connected
- Green dot: connected
- Red dot: connection error

## Rust SSH Primitives (Tauri Commands)

```rust
// Connection management
ssh_connect(instance_id) -> Result<bool>
ssh_disconnect(instance_id) -> Result<bool>
ssh_status(instance_id) -> Result<String>  // "connected" | "disconnected" | "error"

// SSH exec
ssh_exec(instance_id, command) -> Result<SshExecResult>
// SshExecResult { stdout, stderr, exit_code }

// SFTP file operations
sftp_read(instance_id, path) -> Result<String>
sftp_write(instance_id, path, content) -> Result<bool>
sftp_list(instance_id, path) -> Result<Vec<SftpEntry>>
sftp_stat(instance_id, path) -> Result<SftpStat>
sftp_remove(instance_id, path) -> Result<bool>
sftp_read_bytes(instance_id, path) -> Result<Vec<u8>>
```

## Remote Business Commands

Built on top of primitives, returning the same types as local commands. Named with `remote_` prefix.

**Config:**
- `remote_read_raw_config(id)` — sftp_read openclaw.json
- `remote_apply_config_patch(id, patch, params)` — sftp read → merge patch in memory → sftp write
- `remote_get_system_status(id)` — sftp read config + ssh_exec version check + gateway health probe

**Gateway:**
- `remote_restart_gateway(id)` — ssh_exec "openclaw gateway restart"
- `remote_check_gateway_health(id)` — ssh_exec curl or TCP probe via SSH

**Sessions:**
- `remote_analyze_sessions(id)` — sftp list + sftp read JSONL files, reuse classification logic
- `remote_preview_session(id, agent, session)` — sftp read single JSONL
- `remote_delete_sessions_by_ids(id, agent, ids)` — sftp remove files + update sessions.json

**Agents/Channels:**
- All via sftp read/write config, same logic as local.

## Frontend Architecture

**Instance Tab Bar:**

```
┌──────────┬──────────────┬──────────────┬─────┐
│ 🟢 Local │ 🟢 Prod VPS  │ ⚫ Staging   │  +  │
└──────────┴──────────────┴──────────────┴─────┘
```

- First tab is always Local (not removable).
- `+` button opens Add Remote Instance dialog.
- Right-click tab for edit/disconnect/remove.

**API context layer:**

```typescript
const [activeInstance, setActiveInstance] = useState<string>("local");

function getApi(instanceId: string) {
  if (instanceId === "local") return localApi;
  return createRemoteApi(instanceId);
}
```

`createRemoteApi(id)` returns the same interface as `localApi`, mapping each method to the corresponding `remote_*` Tauri command. Page components get the API from context — zero changes to page logic.

## Implementation Phases

### Phase 1 — Basic Connectivity (MVP)

- SSH connection management (connect/disconnect/status)
- SFTP read/write primitives
- SSH exec primitive
- Remote instance config CRUD (add/edit/delete, persist to JSON)
- Instance Tab Bar UI
- `remote_read_raw_config` + `remote_get_system_status` — Home page shows remote status

### Phase 2 — Core Management

- Remote config editing (apply_config_patch remote version)
- Remote gateway restart
- Remote Agent/Channel management
- API context layer — pages switch seamlessly

### Phase 3 — Full Parity

- Remote session analysis/preview/delete
- Remote Doctor diagnostics
- Remote Chat (via ssh_exec openclaw agent --message)
- Connection resilience: reconnect, timeout handling, error UX polish
