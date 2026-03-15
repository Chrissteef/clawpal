# Openclaw-Driven Model/Channel Ops Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

## For Feature

**Goal:** Enable model catalog and model/profile management to use openclaw CLI + caching keyed to openclaw version, add upgrade-check visibility, sync current config models into profiles, and enrich discord channel metadata.

**Architecture:** Introduce a small openclaw-command layer in Tauri that executes `openclaw` with `--json` options, normalizes JSON outputs, writes cache under clawpal dir keyed by openclaw CLI version, and exposes UI actions for sync/inspect.

**Tech Stack:** Rust (`std::process` + `serde_json`), React/Tauri IPC, OpenClaw CLI commands.

---

### Task 1: OpenClaw command adapter + version-aware model catalog cache

**Files:**
- `src-tauri/src/commands.rs`
- `src-tauri/src/models.rs`
- `src-tauri/Cargo.toml` (if needed)

**Step 1: Implement command runner utilities**
- Add small wrappers for running CLI commands (`openclaw --version`, `openclaw models list --all --json`, with optional `--provider`, `openclaw channels list --json --no-usage`, `openclaw update status --json`, `openclaw channels resolve ... --json` when needed).
- Parse and normalize JSON outputs into internal structs.
- Capture stderr for diagnostic messages.

**Step 2: Add cache structs and paths**
- Add catalog cache file path e.g. `model-catalog-cache.json` under clawpal dir.
- Include fields: `cliVersion`, `updatedAt`, `providers`, `source`, `error`, `ttlMinutes`.

**Step 3: Add version-aware refresh policy**
- Read current `openclaw --version` and current cache version.
- If version changed, or cache stale (e.g. 12h), refresh by running `openclaw models list --all --json`.
- If refresh fails, fall back to config-based extraction so UI remains usable.

**Step 4: Wire `list_model_catalog` to new pipeline**
- Return CLI/ cache-normalized `ModelCatalogProvider[]`.
- Keep `model/probe` fallback from config providers in emergency path.

### Task 2: OpenClaw upgrade status endpoint

**Files:**
- `src-tauri/src/commands.rs`
- `src-tauri/src/lib.rs`
- `src/lib/types.ts`
- `src/lib/api.ts`
- `src/pages/Home.tsx`

**Step 1: Add API result model**
- Add `OpenclawUpdateCheck` to `types.ts` with `installKind/channel/source/version/availability` etc (at least `outdated`, `currentVersion`, `latestVersion`, `upgradeAvailable`, `updateLine`, `checkedAt`).

**Step 2: Add backend command**
- Add `check_openclaw_update` that runs `openclaw update status --json`.
- Return normalized result and errors non-fatal to UI.

**Step 3: Extend status payload**
- Optionally add `openclawUpdate` field to `SystemStatus`, fetched inside `get_system_status`.

**Step 4: Show on Home page**
- Add “OpenClaw update” card: current vs latest, and availability warning if newer exists.

### Task 3: Auto-extract existing model bindings to profiles

**Files:**
- `src-tauri/src/commands.rs`
- `src-tauri/src/lib.rs`
- `src/lib/types.ts`
- `src/lib/api.ts`
- `src/pages/Models.tsx`

**Step 1: Add extractor command**
- New command `extract_model_profiles_from_config(dryRun: bool)` that inspects `/agents/*model`, `channels.*.model`, and defaults for explicit model refs.
- For each unique model ref create/refresh profiles from current config, using auth mapping heuristics:
  - if model has `provider` prefix and `auth` map exists, pick first matching `auth.profiles` for that provider;
  - fallback to `default`.
- Return `created|updated|skipped` stats and preview list.

**Step 2: Frontend button action**
- Add button `Import current model config as profiles` in models page.
- Add optional checkbox / confirm to enable overwrite for detected matches.

**Step 3: Make bindings discover profile matches**
- Since extractor writes profiles with exact `provider/model`, channel/agent bindings immediately resolve to profile IDs.
- Keep `find_profile_by_model` logic unchanged (already matches by exact string).

### Task 4: Discord channel name enrichment via CLI resolve

**Files:**
- `src-tauri/src/commands.rs`
- `src-tauri/src/lib.rs`
- `src-tauri/src/types.ts` (if new shape)
- `src/pages/Channels.tsx`

**Step 1: Add channel-name resolver command**
- Parse Discord channel IDs from channel node paths (e.g. `channels.discord.guilds.<guildId>.channels.<channelId>`).
- For discovered IDs, call `openclaw channels resolve <id> --channel discord --kind group --json` and map id->name.
- Return best effort, with `displayName` and `nameStatus` (`resolved`/`missing`).

**Step 2: Cache small resolution results**
- Add small cache in clawpal dir keyed by `guildId:channelId` and openclaw version.
- Graceful fallback to raw ID when resolve fails.

**Step 3: UI display**
- Add `displayName` in channel row and keep path as stable key.

### Execution flow

**Use subagent-driven mode with review checkpoints** because this spans Rust + React + IPC.

