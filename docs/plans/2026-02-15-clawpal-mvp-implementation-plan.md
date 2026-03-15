# ClawPal MVP Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Deliver a working Tauri MVP for ClawPal that implements install wizard, snapshot/rollback, and doctor repair flows with data-safe write operations.

**Architecture:** A thin React UI orchestrates a Rust-backed command API in Tauri; all configuration reads/writes, preview/rollback logic, and diagnosis checks are implemented in Rust for deterministic behavior and simpler cross-platform filesystem handling.

**Tech Stack:** Tauri 2, Rust, React 18, TypeScript, json5, monaco-editor (diff), Vitest, Cargo test

---

### Task 1: 初始化项目骨架与基础命令骨架

**Files:**
- Create: `package.json`
- Create: `src-tauri/Cargo.toml`
- Create: `src-tauri/src/main.rs`
- Create: `src-tauri/src/commands.rs`
- Create: `src-tauri/src/lib.rs`
- Create: `src-tauri/src/commands_tests.rs`
- Create: `src/main.tsx`
- Create: `src/App.tsx`
- Create: `src-tauri/tauri.conf.json`

**Step 1: Write the failing test**

Create a Rust compile smoke test for command registration entry.

```rust
// src-tauri/src/commands_tests.rs
#[test]
fn test_register_commands_compiles() {
    assert!(true);
}
```

**Step 2: Run it to verify it fails**

Run: `cd /Users/zhixian/Codes/clawpal/src-tauri && cargo test --test commands_tests -- --nocapture`
Expected: FAIL with missing crate/modules (project尚未初始化).

**Step 3: Write the minimal implementation**

- Initialize a valid Tauri app with `invoke_handler` placeholder.
- Register placeholder command stubs returning `Ok(())`.

**Step 4: Run test to verify it passes**

Run: `cd /Users/zhixian/Codes/clawpal/src-tauri && cargo test --test commands_tests -- --nocapture`
Expected: PASS.

**Step 5: Commit**

```bash
cd /Users/zhixian/Codes/clawpal
git add package.json src-tauri/Cargo.toml src-tauri/src/main.rs src-tauri/src/lib.rs src-tauri/src/commands.rs src-tauri/src/commands_tests.rs src-tauri/tauri.conf.json src/main.tsx src/App.tsx
git commit -m "chore: scaffold Tauri + React MVP project skeleton"
```

### Task 2: 建立类型与路径模型

**Files:**
- Create: `src-tauri/src/models.rs`
- Create: `src-tauri/src/config_io.rs`
- Modify: `src-tauri/src/lib.rs`
- Create: `src-tauri/src/models_tests.rs`

**Step 1: Write the failing test**

```rust
#[test]
fn test_openclaw_paths_are_resolved() {
    let paths = resolve_openclaw_paths();
    assert!(paths.config_path.to_string_lossy().contains(".openclaw"));
    assert!(paths.history_dir.ends_with(".clawpal/history"));
}
```

**Step 2: Run it to verify it fails**

Run: `cargo test models_tests::test_openclaw_paths_are_resolved`
Expected: FAIL (path resolver未实现 / test compile fail).

**Step 3: Write the minimal implementation**

- Define `OpenClawPaths { config_path, history_dir, metadata_path }`.
- Implement path resolution from home directory via `dirs` crate and create dirs if missing (non-destructive).

**Step 4: Run test to verify it passes**

Run: `cargo test models_tests::test_openclaw_paths_are_resolved`
Expected: PASS.

**Step 5: Commit**

```bash
git add src-tauri/src/models.rs src-tauri/src/config_io.rs src-tauri/src/models_tests.rs src-tauri/src/lib.rs
git commit -m "feat: define core path model for openclaw config and history"
```

### Task 3: 实现 Recipe 类型与参数校验

**Files:**
- Create: `src-tauri/src/recipe.rs`
- Modify: `src-tauri/src/commands.rs`
- Create: `src/lib/recipe_catalog.ts`
- Create: `src-tauri/src/recipe_tests.rs`

**Step 1: Write the failing test**

```rust
#[test]
fn test_validate_recipe_params_missing_required() {
    let recipe = sample_recipe();
    let params = serde_json::json!({});
    assert!(validate_recipe_params(&recipe, &params).is_err());
}
```

**Step 2: Run it to verify it fails**

Run: `cargo test recipe_tests::test_validate_recipe_params_missing_required`
Expected: FAIL (validate function未实现).

**Step 3: Write the minimal implementation**

- Implement `Recipe`, `RecipeParam`, `RecipeEngine::validate`.
- Parse params from static TS catalog entry and pass through Tauri command `list_recipes`.

**Step 4: Run test to verify it passes**

Run: `cargo test recipe_tests::test_validate_recipe_params_missing_required`
Expected: PASS.

**Step 5: Commit**

```bash
git add src-tauri/src/recipe.rs src-tauri/src/recipe_tests.rs src-tauri/src/commands.rs src/lib/recipe_catalog.ts
git commit -m "feat: define recipe schema and parameter validation"
```

### Task 4: 实现预览与应用流程（含备份）

**Files:**
- Modify: `src-tauri/src/recipe.rs`
- Modify: `src-tauri/src/config_io.rs`
- Create: `src-tauri/src/history.rs`
- Modify: `src-tauri/src/commands.rs`
- Create: `src-tauri/src/recipe_flow_tests.rs`
- Modify: `src/lib/api.ts`
- Create: `src/lib/types.ts`

**Step 1: Write the failing test**

```rust
#[test]
fn test_apply_writes_backup_before_modify() {
    // Arrange temp openclaw path & in-memory json
    // Assert backup exists after apply with different content hash
}
```

**Step 2: Run it to verify it fails**

Run: `cargo test recipe_flow_tests::test_apply_writes_backup_before_modify`
Expected: FAIL (apply + history not implemented).

**Step 3: Write the minimal implementation**

- Implement preview by rendering template + merge patch to candidate JSON.
- On apply: read current config, create snapshot copy, write temporary file then atomically replace target.
- Emit `ApplyResult` with `snapshot_id` and backup path.

**Step 4: Run test to verify it passes**

Run: `cargo test recipe_flow_tests::test_apply_writes_backup_before_modify`
Expected: PASS.

**Step 5: Commit**

```bash
git add src-tauri/src/recipe.rs src-tauri/src/config_io.rs src-tauri/src/history.rs src-tauri/src/commands.rs src-tauri/src/recipe_flow_tests.rs src/lib/api.ts src/lib/types.ts
git commit -m "feat: implement recipe preview/apply with backup snapshot"
```

### Task 5: 实现历史列表和回滚命令

**Files:**
- Modify: `src-tauri/src/history.rs`
- Modify: `src-tauri/src/commands.rs`
- Create: `src-tauri/src/history_tests.rs`
- Create: `src/pages/History.tsx`
- Modify: `src/App.tsx`

**Step 1: Write the failing test**

```rust
#[test]
fn test_history_snapshot_roundtrip_rollback() {
    // create fake metadata + two snapshots
    // restore earlier snapshot and verify current differs then matches target
}
```

**Step 2: Run it to verify it fails**

Run: `cargo test history_tests::test_history_snapshot_roundtrip_rollback`
Expected: FAIL (history index and rollback 未实现).

**Step 3: Write the minimal implementation**

- Implement metadata read/write format with monotonically sorted IDs.
- Implement `list_history`, `preview_rollback`, and `rollback`.
- Add history UI with “查看 diff / 回滚” actions.

**Step 4: Run test to verify it passes**

Run: `cargo test history_tests::test_history_snapshot_roundtrip_rollback`
Expected: PASS.

**Step 5: Commit**

```bash
git add src-tauri/src/history.rs src-tauri/src/commands.rs src-tauri/src/history_tests.rs src/pages/History.tsx src/App.tsx
git commit -m "feat: add snapshot history list and rollback flow"
```

### Task 6: 实现 Doctor 检查与修复

**Files:**
- Create: `src-tauri/src/doctor.rs`
- Modify: `src-tauri/src/commands.rs`
- Create: `src-tauri/src/doctor_tests.rs`
- Create: `src/pages/Doctor.tsx`
- Modify: `src/lib/api.ts`

**Step 1: Write the failing test**

```rust
#[test]
fn test_doctor_catches_invalid_json_syntax() {
    // Provide broken json5 content
    // assert issue with code = json.syntax
}
```

**Step 2: Run it to verify it fails**

Run: `cargo test doctor_tests::test_doctor_catches_invalid_json_syntax`
Expected: FAIL (doctor module未实现).

**Step 3: Write the minimal implementation**

- Check parse validity, mandatory top-level fields, permission/ownership basics.
- Implement one-shot fix functions for trailing comma + missing required field default.
- Return actionable `DoctorReport` from command.

**Step 4: Run test to verify it passes**

Run: `cargo test doctor_tests::test_doctor_catches_invalid_json_syntax`
Expected: PASS.

**Step 5: Commit**

```bash
git add src-tauri/src/doctor.rs src-tauri/src/commands.rs src-tauri/src/doctor_tests.rs src/pages/Doctor.tsx src/lib/api.ts
git commit -m "feat: add doctor diagnostics and safe auto-fix hooks"
```

### Task 7: 前端安装向导与 Diff 页面

**Files:**
- Modify: `src/pages/Recipes.tsx`
- Create: `src/pages/Install.tsx`
- Create: `src/components/RecipeCard.tsx`
- Create: `src/components/ParamForm.tsx`
- Create: `src/components/DiffViewer.tsx`
- Modify: `src/lib/api.ts`
- Create: `src/pages/Home.tsx`

**Step 1: Write the failing test**

```ts
// src/__tests__/InstallFlow.spec.ts
test('install flow shows preview and apply action', async () => {
  render(<Install />);
  // fill form -> click preview -> ensure diff rendered -> apply enabled
});
```

**Step 2: Run it to verify it fails**

Run: `npx vitest run src/__tests__/InstallFlow.spec.ts`
Expected: FAIL (页面未实现).

**Step 3: Write the minimal implementation**

- Wire form to Recipe params from catalog and invoke `preview_apply`.
- Add DiffViewer with highlight.
- Add confirm/apply button and success toasts.

**Step 4: Run test to verify it passes**

Run: `npx vitest run src/__tests__/InstallFlow.spec.ts`
Expected: PASS.

**Step 5: Commit**

```bash
git add src/pages/Recipes.tsx src/pages/Install.tsx src/components/RecipeCard.tsx src/components/ParamForm.tsx src/components/DiffViewer.tsx src/lib/api.ts src/pages/Home.tsx src/__tests__/InstallFlow.spec.ts
git commit -m "feat: implement recipe install wizard with preview+apply"
```

### Task 8: 主页面路由与状态整合

**Files:**
- Modify: `src/App.tsx`
- Modify: `src/lib/state.ts`
- Modify: `src/pages/Home.tsx`
- Modify: `src/pages/History.tsx`
- Create: `src/lib/state_tests.ts`

**Step 1: Write the failing test**

```ts
test('state loads recipes and history on boot', async () => {
  // mock tauri commands, render App, assert key tiles are loaded
});
```

**Step 2: Run it to verify it fails**

Run: `npx vitest run src/lib/state_tests.ts`
Expected: FAIL (state orchestration未实现).

**Step 3: Write the minimal implementation**

- Implement shared reducer/events for system status, recipes, history, doctor report.
- Add tabs/routes for Home/Recipes/History/Doctor.

**Step 4: Run test to verify it passes**

Run: `npx vitest run src/lib/state_tests.ts`
Expected: PASS.

**Step 5: Commit**

```bash
git add src/App.tsx src/lib/state.ts src/lib/state_tests.ts src/pages/Home.tsx src/pages/History.tsx
git commit -m "feat: wire app-wide state and navigation"
```

### Task 9: 端到端验收与发布脚手架

**Files:**
- Modify: `README.md`
- Modify: `package.json`
- Create: `scripts/release.sh`
- Create: `docs/mvp-checklist.md`
- Modify: `src-tauri/tauri.conf.json`

**Step 1: Write the failing test**

```bash
./scripts/release.sh --dry-run
```

Expected: FAIL (脚本不存在).

**Step 2: Run it to verify it fails**

Run: `bash scripts/release.sh --dry-run`
Expected: FAIL / script not found.

**Step 3: Write the minimal implementation**

- Add build scripts and basic build verification matrix (macOS/Windows/Linux placeholders).
- Add manual QA checklist to validate install/rollback/doctor on clean and broken states.
- Update README for usage.

**Step 4: Run test to verify it passes**

Run: `bash scripts/release.sh --dry-run`
Expected: PASS (dry-run prints validated commands without publishing).

**Step 5: Commit**

```bash
git add README.md package.json scripts/release.sh docs/mvp-checklist.md src-tauri/tauri.conf.json
git commit -m "chore: add MVP runbook and release scripts"
```

---

## 执行方式

Plan complete and saved to `docs/plans/2026-02-15-clawpal-mvp-implementation-plan.md`. Two execution options:

1. Subagent-Driven (this session) - 我在当前会话按任务分派并在每步进行复核
2. Parallel Session (separate) - 开新会话并使用 executing-plans 执行，有里程碑检查点

Which approach?
