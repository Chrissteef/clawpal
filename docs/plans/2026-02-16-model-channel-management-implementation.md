# Model & Channel Management Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add practical in-app management for model auth profiles, model binding strategy, and basic channel configuration (dm/group/allowlist/mode) with safe write operations and Home status visibility.

**Architecture:** Keep existing Tauri command boundary as the source of truth. Rust keeps all file I/O + patch writes; UI only renders/edit values and submits commands. Configuration changes go through atomic write + snapshot, while model auth profile registry lives in ClawPal metadata directory for separation from OpenClaw config.

**Tech Stack:** Tauri 2, Rust (`serde`, `serde_json`), React 18, TypeScript.

---

### Task 1: Define shared data contract for model/channels

**Files:**
- `src/lib/types.ts`
- `src-tauri/src/commands.rs`

**Step 1: Write the structure in types + command models (code first pass)**

- Add frontend interfaces for:
  - `ModelProfile` (`id`, `name`, `provider`, `model`, `authRef`, `baseUrl`, `description`, `enabled`)
  - `ChannelNode` (`path`, `channelType`, `mode`, `allowlist`, `modelValue`, `hasModelField`)
  - `ModelBinding` (`scope`, `scopeId`, `modelProfileId`, `modelValue`)
  - API command input/output types.

**Step 2: Add corresponding Rust serializable structs in `commands.rs`**

- Add `#[derive(Serialize, Deserialize)]` structs that mirror above.

---

### Task 2: Backing storage + utilities

**Files:**
- `src-tauri/src/commands.rs`
- `src-tauri/src/models.rs`

**Step 1: Add metadata paths for profile storage**

- Use `OpenClawPaths` with existing `clawpal_dir` and introduce a `model_profiles.json` file path inside it.

**Step 2: Implement helper readers/writers**

- `load_model_profiles(paths) -> Vec<ModelProfile>`
- `save_model_profiles(paths, &Vec<ModelProfile>)`
- Path-safe setters for nested JSON fields inside `openclaw.json` using dot-path nodes.

---

### Task 3: Model profile CRUD commands

**Files:**
- `src-tauri/src/commands.rs`
- `src-tauri/src/lib.rs`
- `src/lib/api.ts`
- `src/lib/types.ts`

**Step 1: Add Tauri commands**

- `list_model_profiles() -> Vec<ModelProfile>`
- `upsert_model_profile(profile: ModelProfile) -> ModelProfile`
- `delete_model_profile(profile_id: String) -> bool`

**Step 2: Register commands in `invoke_handler`**

- Ensure frontend can call these commands via API wrapper.

---

### Task 4: Channel discovery and channel config commands

**Files:**
- `src-tauri/src/commands.rs`
- `src-tauri/src/lib.rs`
- `src/lib/api.ts`
- `src/lib/types.ts`

**Step 1: Discover editable channel nodes**

- Implement recursion under `/channels` for nodes that carry `type|mode|allowlist|model`.
- Return normalized `ChannelNode` list with stable paths.

**Step 2: Expose write commands**

- `list_channels() -> Vec<ChannelNode>`
- `update_channel_config(path: String, channel_type: Option<String>, mode: Option<String>, allowlist: Vec<String>, model: Option<String>) -> bool`
- `delete_channel_node(path: String) -> bool` (safe: remove node if exists)

**Step 3: Register commands**

- Add new invoke entries in `lib.rs`.

---

### Task 5: Model binding commands (global/agent/channel)

**Files:**
- `src-tauri/src/commands.rs`
- `src-tauri/src/lib.rs`
- `src/lib/api.ts`
- `src/lib/types.ts`

**Step 1: Implement read summary**

- Add helper to return effective model binding map from current config:
  - global default model
  - per-agent model overrides
  - per-channel model overrides

**Step 2: Implement commands**

- `set_global_model(profile_id: Option<String>) -> bool`
- `set_agent_model(agent_id: String, profile_id: Option<String>) -> bool`
- `set_channel_model(channel_path: String, profile_id: Option<String>) -> bool`

**Step 3: Snapshot safety**

- Use existing snapshot flow (`read_openclaw_config` + `add_snapshot`) before each write.

---

### Task 6: Frontend pages and routing

**Files:**
- `src/App.tsx`
- `src/pages/Models.tsx` (new)
- `src/pages/Channels.tsx` (new)
- `src/lib/api.ts`
- `src/lib/types.ts`

**Step 1: Wire routes for Models and Channels**

- Extend top nav and render new pages.

**Step 2: Models page**

- List model profiles, create/update/delete quickly.
- Add `profile id` selector.
- Add global / agent / channel assignment panel (simple controls first).

**Step 3: Channels page**

- List discovered channel nodes and allow update model/allowlist/mode.
- Add minimal inline editing (textarea for allowlist lines).

---

### Task 7: Surface in Home

**Files:**
- `src-tauri/src/commands.rs`
- `src/lib/types.ts`
- `src/pages/Home.tsx`

**Step 1: Expand system status with `model` binding counts**

- Keep current aggregates and make counts update from registry + bindings.

**Step 2: Display quick links/button actions**

- From Home open Models / Channels quick nav (simple text action).

---

### Task 8: Manual verification checklist

**Files:**
- `docs/mvp-checklist.md`

**Step 1: Add verification items for this feature set**

- Profile add/update/delete.
- Channel mode/allowlist/model update.
- Global/agent/channel model assignment.
- Status card updates after edits.

**Step 2: Execute manual run list**

- Build frontend and verify command invocation completes.

