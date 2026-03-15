# ClawPal Product Redesign Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Simplify ClawPal from 7 pages to 4+1 (Home/Recipes/History/Doctor + Settings), add sidebar layout, add Chat window with LLM tool-calling, and add Agent overview on Home page.

**Architecture:** Replace top-bar tabs with a left sidebar. Restructure Home to show status summary + Agents overview + recommended Recipes + recent history + a right-side Chat panel. Move Model Profile management to Settings. Merge Data cleanup into Doctor. Delete Channels and Data pages. Add new Rust commands for Agent detail listing and API key resolution. Add a frontend Chat component that calls LLM APIs via Tauri HTTP and renders tool-call results inline.

**Tech Stack:** Tauri 2.0 (Rust backend), React 18 + TypeScript (frontend), Vite, json5

---

## Task 1: Sidebar layout — replace top-bar with left sidebar

**Files:**
- Modify: `src/App.tsx`
- Modify: `src/styles.css`

**Step 1: Update styles.css — add sidebar layout, remove topbar styles**

Replace the entire `src/styles.css` with sidebar-based layout:

```css
:root {
  --bg: #0f1220;
  --panel: #171b2f;
  --text: #e6ebff;
  --accent: #6dd0ff;
  --sidebar-width: 200px;
}

* { box-sizing: border-box; }
body {
  margin: 0;
  font-family: ui-sans-serif, -apple-system, sans-serif;
  color: var(--text);
  background: linear-gradient(120deg, #0f1220, #151935);
}

.app-shell {
  display: flex;
  height: 100vh;
}

.sidebar {
  width: var(--sidebar-width);
  min-width: var(--sidebar-width);
  background: var(--panel);
  border-right: 1px solid #29325a;
  display: flex;
  flex-direction: column;
  padding: 16px 0;
}

.sidebar h1 {
  font-size: 18px;
  padding: 0 16px 12px;
  margin: 0;
  border-bottom: 1px solid #29325a;
}

.sidebar nav {
  display: flex;
  flex-direction: column;
  flex: 1;
  padding: 8px 0;
}

.sidebar nav button {
  text-align: left;
  border: none;
  background: transparent;
  color: var(--text);
  padding: 10px 16px;
  font-size: 14px;
  cursor: pointer;
  border-radius: 0;
}

.sidebar nav button:hover {
  background: rgba(109, 208, 255, 0.08);
}

.sidebar nav button.active {
  background: rgba(109, 208, 255, 0.12);
  color: var(--accent);
}

.sidebar .sidebar-divider {
  border-top: 1px solid #29325a;
  margin: 8px 0;
}

.content {
  flex: 1;
  overflow-y: auto;
  padding: 20px 24px;
}

button {
  border: 1px solid #2d3560;
  background: #1f2750;
  color: var(--text);
  padding: 8px 12px;
  border-radius: 8px;
  cursor: pointer;
}

.recipe-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(220px, 1fr)); gap: 12px; }
.recipe-card, .card, .history-item { background: var(--panel); padding: 12px; border-radius: 10px; border: 1px solid #29325a; }
.diff-viewer { background: #0b0f20; padding: 12px; border-radius: 8px; overflow: auto; max-height: 260px; }
.param-form label { display: block; margin: 10px 0; }
.param-form input, .param-form textarea { width: 100%; }
.status-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(180px, 1fr)); gap: 12px; }
```

**Step 2: Rewrite App.tsx — sidebar navigation with 4+1 routes**

Remove imports for `Models`, `Channels`, `Data`. Add `Settings` import. Change layout from top-bar to sidebar. Remove `models`, `channels`, `data` from Route type.

```tsx
import React, { useState } from "react";
import { Home } from "./pages/Home";
import { Recipes } from "./pages/Recipes";
import { Install } from "./pages/Install";
import { History } from "./pages/History";
import { Doctor } from "./pages/Doctor";
import { Settings } from "./pages/Settings";

type Route = "home" | "recipes" | "install" | "history" | "doctor" | "settings";

export function App() {
  const [route, setRoute] = useState<Route>("home");
  const [recipeId, setRecipeId] = useState<string | null>(null);
  const [recipeSource, setRecipeSource] = useState<string | undefined>(undefined);

  return (
    <div className="app-shell">
      <aside className="sidebar">
        <h1>ClawPal</h1>
        <nav>
          <button className={route === "home" ? "active" : ""} onClick={() => setRoute("home")}>Home</button>
          <button className={route === "recipes" ? "active" : ""} onClick={() => setRoute("recipes")}>Recipes</button>
          <button className={route === "history" ? "active" : ""} onClick={() => setRoute("history")}>History</button>
          <button className={route === "doctor" ? "active" : ""} onClick={() => setRoute("doctor")}>Doctor</button>
          <div className="sidebar-divider" />
          <button className={route === "settings" ? "active" : ""} onClick={() => setRoute("settings")}>Settings</button>
        </nav>
      </aside>
      <main className="content">
        {route === "home" && <Home />}
        {route === "recipes" && (
          <Recipes
            onInstall={(id, source) => {
              setRecipeId(id);
              setRecipeSource(source);
              setRoute("install");
            }}
          />
        )}
        {route === "install" && recipeId && (
          <Install
            recipeId={recipeId}
            recipeSource={recipeSource}
            onDone={() => setRoute("recipes")}
          />
        )}
        {route === "install" && !recipeId && <p>No recipe selected.</p>}
        {route === "history" && <History />}
        {route === "doctor" && <Doctor />}
        {route === "settings" && <Settings />}
        {route === "install" && (
          <button onClick={() => setRoute("recipes")} style={{ marginTop: 12 }}>
            ← Recipes
          </button>
        )}
      </main>
    </div>
  );
}
```

**Step 3: Verify build**

Run: `cd /Users/zhixian/Codes/clawpal && npm run build`

This will fail because `Settings` page doesn't exist yet. That's expected — we'll create it in Task 3.

**Step 4: Commit**

```bash
git add src/App.tsx src/styles.css
git commit -m "refactor: replace top-bar tabs with left sidebar layout"
```

---

## Task 2: Delete Channels and Data pages

**Files:**
- Delete: `src/pages/Channels.tsx`
- Delete: `src/pages/Data.tsx`

**Step 1: Delete files**

```bash
rm src/pages/Channels.tsx src/pages/Data.tsx
```

**Step 2: Commit**

```bash
git add -u src/pages/Channels.tsx src/pages/Data.tsx
git commit -m "chore: remove Channels and Data pages (merged into Doctor/Recipes)"
```

---

## Task 3: Create Settings page with Model Profile management

**Files:**
- Create: `src/pages/Settings.tsx`
- Modify: `src/lib/api.ts` — add `resolveApiKey` API call
- Modify: `src/lib/types.ts` — add `ResolvedApiKey` type

**Step 1: Add ResolvedApiKey type to types.ts**

Append to `src/lib/types.ts`:

```typescript
export interface ResolvedApiKey {
  profileId: string;
  maskedKey: string;
}
```

**Step 2: Add resolveApiKeys API call to api.ts**

Add to the `api` object in `src/lib/api.ts`:

```typescript
  resolveApiKeys: (): Promise<ResolvedApiKey[]> =>
    invoke("resolve_api_keys", {}),
```

**Step 3: Add resolve_api_keys Rust command**

In `src-tauri/src/commands.rs`, add a new command that reads the env var referenced by `authRef` in each model profile and returns masked keys:

```rust
#[derive(Debug, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct ResolvedApiKey {
    pub profile_id: String,
    pub masked_key: String,
}

#[tauri::command]
pub fn resolve_api_keys() -> Result<Vec<ResolvedApiKey>, String> {
    let paths = resolve_paths();
    let profiles = load_model_profiles(&paths);
    let mut out = Vec::new();
    for profile in &profiles {
        let key = std::env::var(&profile.auth_ref).unwrap_or_default();
        let masked = mask_api_key(&key);
        out.push(ResolvedApiKey {
            profile_id: profile.id.clone(),
            masked_key: masked,
        });
    }
    Ok(out)
}

fn mask_api_key(key: &str) -> String {
    let key = key.trim();
    if key.is_empty() {
        return "not set".to_string();
    }
    if key.len() <= 8 {
        return "***".to_string();
    }
    let prefix = &key[..4.min(key.len())];
    let suffix = &key[key.len().saturating_sub(4)..];
    format!("{prefix}...{suffix}")
}
```

Register `resolve_api_keys` in `src-tauri/src/lib.rs` invoke_handler.

**Step 4: Create Settings.tsx**

```tsx
import React, { useEffect, useMemo, useState } from "react";
import { api } from "../lib/api";
import type { ModelCatalogProvider, ModelProfile, ResolvedApiKey } from "../lib/types";

type ProfileForm = {
  id: string;
  name: string;
  provider: string;
  model: string;
  authRef: string;
  baseUrl: string;
  description: string;
  enabled: boolean;
};

function emptyProfile(): ProfileForm {
  return { id: "", name: "", provider: "", model: "", authRef: "", baseUrl: "", description: "", enabled: true };
}

export function Settings() {
  const [profiles, setProfiles] = useState<ModelProfile[]>([]);
  const [resolvedKeys, setResolvedKeys] = useState<ResolvedApiKey[]>([]);
  const [modelCatalog, setModelCatalog] = useState<ModelCatalogProvider[]>([]);
  const [profileForm, setProfileForm] = useState<ProfileForm>(emptyProfile());
  const [chatProfileId, setChatProfileId] = useState(() => localStorage.getItem("clawpal_chat_profile") || "");
  const [message, setMessage] = useState("");

  const refresh = () => {
    Promise.all([
      api.listModelProfiles(),
      api.listModelCatalog(),
      api.resolveApiKeys(),
    ])
      .then(([nextProfiles, nextCatalog, nextKeys]) => {
        setProfiles(nextProfiles);
        setModelCatalog(nextCatalog);
        setResolvedKeys(nextKeys);
      })
      .catch(() => setMessage("Failed to load settings data"));
  };

  useEffect(refresh, []);

  const keyByProfile = useMemo(() => {
    const map = new Map<string, string>();
    for (const item of resolvedKeys) map.set(item.profileId, item.maskedKey);
    return map;
  }, [resolvedKeys]);

  const modelCandidates = useMemo(() => {
    const found = modelCatalog.find((c) => c.provider === profileForm.provider);
    return found?.models || [];
  }, [modelCatalog, profileForm.provider]);

  const upsert = (event: React.FormEvent) => {
    event.preventDefault();
    if (!profileForm.name || !profileForm.provider || !profileForm.model || !profileForm.authRef) {
      setMessage("Fill required profile fields");
      return;
    }
    api.upsertModelProfile({
      id: profileForm.id || "",
      name: profileForm.name,
      provider: profileForm.provider,
      model: profileForm.model,
      authRef: profileForm.authRef,
      baseUrl: profileForm.baseUrl || undefined,
      description: profileForm.description || undefined,
      enabled: profileForm.enabled,
    })
      .then(() => { setMessage("Saved profile"); setProfileForm(emptyProfile()); refresh(); })
      .catch(() => setMessage("Save failed"));
  };

  return (
    <section>
      <h2>Settings</h2>

      <h3>Model Profiles</h3>
      <div style={{ display: "grid", gap: 12, gridTemplateColumns: "1fr 1fr", alignItems: "start" }}>
        <article className="card">
          <h4>{profileForm.id ? "Edit profile" : "Create profile"}</h4>
          <form onSubmit={upsert} className="param-form">
            <input placeholder="Name" value={profileForm.name} onChange={(e) => setProfileForm((p) => ({ ...p, name: e.target.value }))} />
            <input placeholder="Provider" value={profileForm.provider} onChange={(e) => setProfileForm((p) => ({ ...p, provider: e.target.value }))} list="provider-list" />
            <datalist id="provider-list">{modelCatalog.map((e) => <option key={e.provider} value={e.provider} />)}</datalist>
            <input placeholder="Model" value={profileForm.model} onChange={(e) => setProfileForm((p) => ({ ...p, model: e.target.value }))} list="model-list" />
            <datalist id="model-list">{modelCandidates.map((m) => <option key={m.id} value={m.id} label={m.name || m.id} />)}</datalist>
            <input placeholder="Auth env var (e.g. OPENAI_API_KEY)" value={profileForm.authRef} onChange={(e) => setProfileForm((p) => ({ ...p, authRef: e.target.value }))} />
            <input placeholder="Custom base URL (optional)" value={profileForm.baseUrl} onChange={(e) => setProfileForm((p) => ({ ...p, baseUrl: e.target.value }))} />
            <label><input type="checkbox" checked={profileForm.enabled} onChange={(e) => setProfileForm((p) => ({ ...p, enabled: e.target.checked }))} /> Enabled</label>
            <button type="submit">Save</button>
            {profileForm.id && (
              <button type="button" onClick={() => { api.deleteModelProfile(profileForm.id).then(() => { setProfileForm(emptyProfile()); refresh(); }).catch(() => setMessage("Delete failed")); }}>Delete</button>
            )}
          </form>
        </article>

        <article className="card">
          <h4>Profiles</h4>
          {profiles.length === 0 && <p>No model profiles yet.</p>}
          {profiles.map((p) => (
            <div key={p.id} style={{ border: "1px solid #2d3560", padding: 8, borderRadius: 8, marginBottom: 8 }}>
              <div style={{ display: "flex", justifyContent: "space-between" }}>
                <strong>{p.name}</strong>
                <span style={{ opacity: 0.8 }}>{p.enabled ? "enabled" : "disabled"}</span>
              </div>
              <div style={{ opacity: 0.85, marginTop: 4 }}>Provider: {p.provider}</div>
              <div style={{ opacity: 0.85 }}>Model: {p.model}</div>
              <div style={{ opacity: 0.85 }}>API Key: {keyByProfile.get(p.id) || "unknown"}</div>
              {p.baseUrl && <div style={{ opacity: 0.85 }}>Custom URL: {p.baseUrl}</div>}
              <div style={{ marginTop: 6, display: "flex", gap: 6 }}>
                <button type="button" onClick={() => setProfileForm({ id: p.id, name: p.name, provider: p.provider, model: p.model, authRef: p.authRef, baseUrl: p.baseUrl || "", description: p.description || "", enabled: p.enabled })}>Edit</button>
                <button type="button" onClick={() => { api.deleteModelProfile(p.id).then(() => { if (profileForm.id === p.id) setProfileForm(emptyProfile()); refresh(); }).catch(() => setMessage("Delete failed")); }}>Delete</button>
              </div>
            </div>
          ))}
        </article>
      </div>

      <h3 style={{ marginTop: 16 }}>Chat Model</h3>
      <article className="card">
        <label>
          Model Profile for Chat:
          <select value={chatProfileId} onChange={(e) => { setChatProfileId(e.target.value); localStorage.setItem("clawpal_chat_profile", e.target.value); }}>
            <option value="">Select a profile</option>
            {profiles.filter((p) => p.enabled).map((p) => (
              <option key={p.id} value={p.id}>{p.name} ({p.model})</option>
            ))}
          </select>
        </label>
      </article>

      <h3 style={{ marginTop: 16 }}>Paths</h3>
      <article className="card">
        <p>Paths are shown on the Home page status summary. Override with environment variables:</p>
        <code>CLAWPAL_OPENCLAW_DIR</code>, <code>CLAWPAL_DATA_DIR</code>
      </article>

      <p>{message}</p>
    </section>
  );
}
```

**Step 5: Verify build**

Run: `npm run build`

Expected: build passes (Settings is now imported, Models/Channels/Data are removed from App.tsx).

**Step 6: Commit**

```bash
git add src/pages/Settings.tsx src/lib/types.ts src/lib/api.ts src-tauri/src/commands.rs src-tauri/src/lib.rs
git commit -m "feat: add Settings page with Model Profile management and API key display"
```

---

## Task 4: Delete Models page

**Files:**
- Delete: `src/pages/Models.tsx`

**Step 1: Delete file**

```bash
rm src/pages/Models.tsx
```

**Step 2: Verify build**

Run: `npm run build`

Expected: passes — Models.tsx is no longer imported anywhere.

**Step 3: Commit**

```bash
git add -u src/pages/Models.tsx
git commit -m "chore: remove Models page (functionality moved to Settings)"
```

---

## Task 5: Add Agent overview Rust command

**Files:**
- Modify: `src-tauri/src/commands.rs` — add `list_agents_overview` command
- Modify: `src-tauri/src/lib.rs` — register command
- Modify: `src/lib/types.ts` — add `AgentOverview` type
- Modify: `src/lib/api.ts` — add `listAgentsOverview` call

**Step 1: Add AgentOverview struct and command in commands.rs**

```rust
#[derive(Debug, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct AgentOverview {
    pub id: String,
    pub model: Option<String>,
    pub channels: Vec<String>,
    pub online: bool,
}

#[tauri::command]
pub fn list_agents_overview() -> Result<Vec<AgentOverview>, String> {
    let paths = resolve_paths();
    let cfg = read_openclaw_config(&paths)?;
    let mut agents = Vec::new();

    if let Some(list) = cfg.pointer("/agents/list").and_then(Value::as_array) {
        let channel_nodes = collect_channel_nodes(&cfg);
        for agent in list {
            let id = agent.get("id").and_then(Value::as_str).unwrap_or("agent").to_string();
            let model = agent.get("model").and_then(read_model_value)
                .or_else(|| cfg.pointer("/agents/defaults/model").and_then(read_model_value))
                .or_else(|| cfg.pointer("/agents/default/model").and_then(read_model_value));
            let channels: Vec<String> = channel_nodes.iter()
                .filter(|ch| {
                    // A channel is associated with an agent if it references this agent
                    // or if it's in the default agent's scope
                    ch.path.contains(&id) || channel_nodes.len() <= 4
                })
                .map(|ch| ch.path.clone())
                .collect();
            let has_sessions = paths.base_dir.join("agents").join(&id).join("sessions").exists();
            agents.push(AgentOverview {
                id,
                model,
                channels,
                online: has_sessions,
            });
        }
    }
    Ok(agents)
}
```

Register `list_agents_overview` in `src-tauri/src/lib.rs`.

**Step 2: Add frontend types and API**

In `src/lib/types.ts`:

```typescript
export interface AgentOverview {
  id: string;
  model: string | null;
  channels: string[];
  online: boolean;
}
```

In `src/lib/api.ts`:

```typescript
  listAgentsOverview: (): Promise<AgentOverview[]> =>
    invoke("list_agents_overview", {}),
```

**Step 3: Verify build**

Run: `npm run build`

**Step 4: Commit**

```bash
git add src-tauri/src/commands.rs src-tauri/src/lib.rs src/lib/types.ts src/lib/api.ts
git commit -m "feat: add list_agents_overview command for Home page agent section"
```

---

## Task 6: Rewrite Home page — status summary + Agents + recommended Recipes + recent history

**Files:**
- Modify: `src/pages/Home.tsx`

**Step 1: Rewrite Home.tsx**

Replace the entire file. The new Home page has four sections (no Chat yet — that comes in Task 7):

```tsx
import React, { useEffect, useState } from "react";
import { api } from "../lib/api";
import type { AgentOverview, HistoryItem, Recipe, SystemStatus } from "../lib/types";

export function Home() {
  const [status, setStatus] = useState<SystemStatus | null>(null);
  const [agents, setAgents] = useState<AgentOverview[]>([]);
  const [recipes, setRecipes] = useState<Recipe[]>([]);
  const [history, setHistory] = useState<HistoryItem[]>([]);
  const [message, setMessage] = useState("");

  useEffect(() => {
    Promise.all([
      api.getSystemStatus(),
      api.listAgentsOverview(),
      api.listRecipes(),
      api.listHistory(5, 0),
    ])
      .then(([s, a, r, h]) => {
        setStatus(s);
        setAgents(a);
        setRecipes(r);
        setHistory(h.items);
      })
      .catch(() => setMessage("Failed to load home data"));
  }, []);

  return (
    <div className="home-layout">
      <div className="home-main">
        <h2>Home</h2>

        {/* Status Summary */}
        <div className="status-grid">
          <div className="card">
            Healthy: {status?.healthy ? "Yes" : "Unknown"}
          </div>
          <div className="card">
            OpenClaw: {status?.openclawVersion || "unknown"}
            {status?.openclawUpdate?.upgradeAvailable && (
              <div style={{ color: "var(--accent)", marginTop: 4 }}>
                Update available: {status.openclawUpdate.latestVersion}
              </div>
            )}
          </div>
          <div className="card">
            Model: {status?.models?.globalDefaultModel || "not set"}
          </div>
        </div>

        {/* Agents Overview */}
        <h3 style={{ marginTop: 16 }}>Agents</h3>
        <div className="status-grid">
          {agents.length === 0 && <p>No agents configured</p>}
          {agents.map((agent) => (
            <div className="card" key={agent.id}>
              <div style={{ display: "flex", justifyContent: "space-between" }}>
                <strong>{agent.id}</strong>
                <span>{agent.online ? "online" : "offline"}</span>
              </div>
              <div style={{ opacity: 0.85, marginTop: 4 }}>
                Model: {agent.model || "inherit"}
              </div>
              <div style={{ opacity: 0.75, marginTop: 2 }}>
                {agent.channels.length > 0
                  ? agent.channels.join(", ")
                  : "no channels"}
              </div>
            </div>
          ))}
        </div>

        {/* Recommended Recipes */}
        <h3 style={{ marginTop: 16 }}>Recommended Recipes</h3>
        <div className="recipe-grid">
          {recipes.slice(0, 4).map((recipe) => (
            <article className="recipe-card" key={recipe.id}>
              <h4 style={{ margin: "0 0 4px" }}>{recipe.name}</h4>
              <p style={{ opacity: 0.85, margin: 0 }}>{recipe.description}</p>
            </article>
          ))}
        </div>

        {/* Recent Activity */}
        <h3 style={{ marginTop: 16 }}>Recent Activity</h3>
        <div>
          {history.length === 0 && <p>No recent activity</p>}
          {history.map((item) => (
            <div key={item.id} style={{ opacity: 0.85, marginBottom: 4 }}>
              {item.createdAt} — {item.recipeId || "manual"} ({item.source})
            </div>
          ))}
        </div>

        <p>{message}</p>
      </div>
    </div>
  );
}
```

**Step 2: Add home-layout styles to styles.css**

Append to `src/styles.css`:

```css
.home-layout {
  display: flex;
  gap: 16px;
  height: 100%;
}

.home-main {
  flex: 1;
  overflow-y: auto;
}

.home-chat {
  width: 340px;
  min-width: 300px;
  display: flex;
  flex-direction: column;
  border-left: 1px solid #29325a;
  padding-left: 16px;
}
```

**Step 3: Verify build**

Run: `npm run build`

**Step 4: Commit**

```bash
git add src/pages/Home.tsx src/styles.css
git commit -m "feat: redesign Home page with status summary, agents, recipes, and recent activity"
```

---

## Task 7: Merge Data cleanup into Doctor page

**Files:**
- Modify: `src/pages/Doctor.tsx`

**Step 1: Extend Doctor.tsx with data cleanup section**

Add memory/session cleanup UI below the existing diagnostics section. Reuse the existing `api.listMemoryFiles`, `api.clearMemory`, `api.listSessionFiles`, `api.clearAllSessions`, `api.clearAgentSessions` calls.

```tsx
import React, { useEffect, useMemo, useReducer, useState } from "react";
import { api } from "../lib/api";
import { initialState, reducer } from "../lib/state";
import type { MemoryFile, SessionFile } from "../lib/types";

function formatBytes(bytes: number) {
  if (bytes <= 0) return "0 B";
  const units = ["B", "KB", "MB", "GB"];
  let index = 0;
  let value = bytes;
  while (value >= 1024 && index < units.length - 1) { value /= 1024; index += 1; }
  return `${value.toFixed(1)} ${units[index]}`;
}

export function Doctor() {
  const [state, dispatch] = useReducer(reducer, initialState);
  const [memoryFiles, setMemoryFiles] = useState<MemoryFile[]>([]);
  const [sessionFiles, setSessionFiles] = useState<SessionFile[]>([]);
  const [dataMessage, setDataMessage] = useState("");

  const hasReport = Boolean(state.doctor);
  const autoFixable = hasReport
    ? state.doctor!.issues.filter((i) => i.autoFixable).map((i) => i.id)
    : [];

  const refreshDoctor = () =>
    api.runDoctor()
      .then((report) => dispatch({ type: "setDoctor", doctor: report }))
      .catch(() => dispatch({ type: "setMessage", message: "Failed to run doctor" }));

  const refreshData = () =>
    Promise.all([api.listMemoryFiles(), api.listSessionFiles()])
      .then(([mem, sess]) => { setMemoryFiles(mem); setSessionFiles(sess); })
      .catch(() => setDataMessage("Failed to load data files"));

  useEffect(() => { refreshDoctor(); refreshData(); }, []);

  const agents = useMemo(
    () => Array.from(new Set(sessionFiles.map((f) => f.agent))).sort().filter(Boolean),
    [sessionFiles],
  );

  const sessionBytes = sessionFiles.reduce((sum, f) => sum + f.sizeBytes, 0);
  const memoryBytes = memoryFiles.reduce((sum, f) => sum + f.sizeBytes, 0);

  return (
    <section>
      <h2>Doctor</h2>

      {/* Diagnostics */}
      <h3>Config Diagnostics</h3>
      {state.doctor && (
        <div>
          <p>Health score: {state.doctor.score}</p>
          <ul>
            {state.doctor.issues.map((issue) => (
              <li key={issue.id}>
                {issue.severity.toUpperCase()} {issue.message}
                {issue.autoFixable && (
                  <button onClick={() => api.fixIssues([issue.id]).then(() => refreshDoctor()).catch(() => dispatch({ type: "setMessage", message: "Fix failed" }))}>
                    fix
                  </button>
                )}
              </li>
            ))}
          </ul>
          <button onClick={() => api.fixIssues(autoFixable).then(() => refreshDoctor()).catch(() => dispatch({ type: "setMessage", message: "Fix all failed" }))} disabled={!autoFixable.length}>
            Fix all auto issues
          </button>
          <button onClick={refreshDoctor} style={{ marginLeft: 8 }}>Refresh</button>
        </div>
      )}
      {!hasReport && <button onClick={refreshDoctor}>Run Doctor</button>}
      <p>{state.message}</p>

      {/* Data Cleanup */}
      <h3 style={{ marginTop: 20 }}>Data Cleanup</h3>
      <div className="status-grid">
        <article className="card">
          <h4>Memory</h4>
          <p>{memoryFiles.length} files ({formatBytes(memoryBytes)})</p>
          <button onClick={() => api.clearMemory().then((n) => { setDataMessage(`Cleared ${n} memory file(s)`); refreshData(); }).catch(() => setDataMessage("Clear failed"))}>
            Clear all memory
          </button>
        </article>
        <article className="card">
          <h4>Sessions</h4>
          <p>{sessionFiles.length} files ({formatBytes(sessionBytes)})</p>
          {agents.map((agent) => {
            const count = sessionFiles.filter((f) => f.agent === agent).length;
            const bytes = sessionFiles.filter((f) => f.agent === agent).reduce((s, f) => s + f.sizeBytes, 0);
            return (
              <div key={agent} style={{ marginBottom: 4, display: "flex", justifyContent: "space-between", alignItems: "center" }}>
                <span>{agent}: {count} files ({formatBytes(bytes)})</span>
                <button onClick={() => api.clearAgentSessions(agent).then((n) => { setDataMessage(`Cleared ${n} files for ${agent}`); refreshData(); }).catch(() => setDataMessage("Clear failed"))}>
                  Clear
                </button>
              </div>
            );
          })}
          <button onClick={() => api.clearAllSessions().then((n) => { setDataMessage(`Cleared ${n} session file(s)`); refreshData(); }).catch(() => setDataMessage("Clear all failed"))} style={{ marginTop: 8 }}>
            Clear all sessions
          </button>
        </article>
      </div>
      <p>{dataMessage}</p>
    </section>
  );
}
```

**Step 2: Verify build**

Run: `npm run build`

**Step 3: Commit**

```bash
git add src/pages/Doctor.tsx
git commit -m "feat: merge data cleanup (memory/sessions) into Doctor page"
```

---

## Task 8: Chat component — LLM integration with tool calling

This is the largest task. It creates a Chat panel that calls an LLM API using the user's Model Profile, with tool-calling support for reading config, previewing changes, and applying recipes.

**Files:**
- Create: `src/components/Chat.tsx`
- Create: `src/lib/chat.ts` — LLM API call logic and tool definitions
- Modify: `src/pages/Home.tsx` — add Chat panel to right side
- Modify: `src/lib/types.ts` — add chat-related types
- Modify: `src-tauri/src/commands.rs` — add `read_raw_config` command
- Modify: `src-tauri/src/lib.rs` — register command

**Step 1: Add read_raw_config Rust command**

In `src-tauri/src/commands.rs`:

```rust
#[tauri::command]
pub fn read_raw_config() -> Result<String, String> {
    let paths = resolve_paths();
    let cfg = read_openclaw_config(&paths)?;
    serde_json::to_string_pretty(&cfg).map_err(|e| e.to_string())
}
```

Register in `src-tauri/src/lib.rs`.

**Step 2: Add API call in api.ts**

```typescript
  readRawConfig: (): Promise<string> =>
    invoke("read_raw_config", {}),
```

**Step 3: Add chat types to types.ts**

```typescript
export interface ChatMessage {
  role: "user" | "assistant" | "system";
  content: string;
  toolCalls?: ChatToolCall[];
  pendingDiff?: string;
}

export interface ChatToolCall {
  name: string;
  args: Record<string, unknown>;
  result?: string;
}
```

**Step 4: Create src/lib/chat.ts — tool definitions and LLM call logic**

```typescript
import { api } from "./api";
import type { ModelProfile } from "./types";

export const SYSTEM_PROMPT = `You are ClawPal, an AI assistant that helps users configure OpenClaw.
You have tools to read the current config, preview changes, apply changes, list recipes, list agents, and run diagnostics.
When a user asks to change configuration:
1. Read the current config to understand what exists
2. Generate the appropriate config patch
3. Preview the change and show the diff to the user
4. Only apply after the user confirms

Always explain what you're about to do before doing it. Be concise.`;

export interface ToolDef {
  type: "function";
  function: {
    name: string;
    description: string;
    parameters: Record<string, unknown>;
  };
}

export const TOOLS: ToolDef[] = [
  {
    type: "function",
    function: {
      name: "read_config",
      description: "Read the current OpenClaw configuration file",
      parameters: { type: "object", properties: {}, required: [] },
    },
  },
  {
    type: "function",
    function: {
      name: "list_agents",
      description: "List all configured agents with their models and channels",
      parameters: { type: "object", properties: {}, required: [] },
    },
  },
  {
    type: "function",
    function: {
      name: "list_recipes",
      description: "List available configuration recipes",
      parameters: { type: "object", properties: {}, required: [] },
    },
  },
  {
    type: "function",
    function: {
      name: "preview_change",
      description: "Preview a configuration change by providing a recipe ID and parameters",
      parameters: {
        type: "object",
        properties: {
          recipe_id: { type: "string", description: "The recipe ID to preview" },
          params: { type: "object", description: "Parameters for the recipe" },
        },
        required: ["recipe_id", "params"],
      },
    },
  },
  {
    type: "function",
    function: {
      name: "run_doctor",
      description: "Run configuration diagnostics to check for issues",
      parameters: { type: "object", properties: {}, required: [] },
    },
  },
];

export async function executeToolCall(name: string, args: Record<string, unknown>): Promise<string> {
  switch (name) {
    case "read_config": {
      const raw = await api.readRawConfig();
      return raw;
    }
    case "list_agents": {
      const agents = await api.listAgentsOverview();
      return JSON.stringify(agents, null, 2);
    }
    case "list_recipes": {
      const recipes = await api.listRecipes();
      return JSON.stringify(recipes.map((r) => ({ id: r.id, name: r.name, description: r.description })), null, 2);
    }
    case "preview_change": {
      const recipeId = args.recipe_id as string;
      const params = args.params as Record<string, string>;
      const preview = await api.previewApply(recipeId, params);
      return JSON.stringify({ diff: preview.diff, warnings: preview.warnings, impactLevel: preview.impactLevel });
    }
    case "run_doctor": {
      const report = await api.runDoctor();
      return JSON.stringify(report, null, 2);
    }
    default:
      return `Unknown tool: ${name}`;
  }
}

export function buildApiUrl(profile: ModelProfile): string {
  const base = profile.baseUrl || getDefaultBaseUrl(profile.provider);
  return `${base.replace(/\/$/, "")}/chat/completions`;
}

function getDefaultBaseUrl(provider: string): string {
  switch (provider.toLowerCase()) {
    case "openai": return "https://api.openai.com/v1";
    case "anthropic": return "https://api.anthropic.com/v1";
    default: return "https://api.openai.com/v1";
  }
}
```

**Step 5: Create src/components/Chat.tsx**

```tsx
import React, { useCallback, useEffect, useRef, useState } from "react";
import { api } from "../lib/api";
import { DiffViewer } from "./DiffViewer";
import { buildApiUrl, executeToolCall, SYSTEM_PROMPT, TOOLS } from "../lib/chat";
import type { ModelProfile } from "../lib/types";

interface Message {
  role: "user" | "assistant";
  content: string;
  diff?: string;
}

export function Chat() {
  const [messages, setMessages] = useState<Message[]>([]);
  const [input, setInput] = useState("");
  const [loading, setLoading] = useState(false);
  const [profile, setProfile] = useState<ModelProfile | null>(null);
  const [apiKey, setApiKey] = useState("");
  const bottomRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const profileId = localStorage.getItem("clawpal_chat_profile");
    if (!profileId) return;
    api.listModelProfiles().then((profiles) => {
      const found = profiles.find((p) => p.id === profileId);
      if (found) {
        setProfile(found);
        // Resolve the API key from env on the Rust side
        api.resolveApiKeys().then((keys) => {
          const match = keys.find((k) => k.profileId === profileId);
          // We need the full key, not masked — we'll need a separate command
          // For now, read from env var name in the profile
        });
      }
    });
  }, []);

  useEffect(() => {
    bottomRef.current?.scrollIntoView({ behavior: "smooth" });
  }, [messages]);

  const send = useCallback(async () => {
    if (!input.trim() || loading) return;
    if (!profile) {
      setMessages((prev) => [...prev, { role: "assistant", content: "Please select a Chat model in Settings first." }]);
      return;
    }

    const userMsg: Message = { role: "user", content: input.trim() };
    setMessages((prev) => [...prev, userMsg]);
    setInput("");
    setLoading(true);

    try {
      // Build conversation for API
      const apiMessages = [
        { role: "system" as const, content: SYSTEM_PROMPT },
        ...messages.map((m) => ({ role: m.role, content: m.content })),
        { role: "user" as const, content: userMsg.content },
      ];

      const url = buildApiUrl(profile);
      const response = await fetch(url, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "Authorization": `Bearer ${apiKey}`,
        },
        body: JSON.stringify({
          model: profile.model,
          messages: apiMessages,
          tools: TOOLS,
        }),
      });

      if (!response.ok) {
        const text = await response.text();
        setMessages((prev) => [...prev, { role: "assistant", content: `API error: ${response.status} ${text}` }]);
        return;
      }

      const data = await response.json();
      const choice = data.choices?.[0];

      if (choice?.message?.tool_calls) {
        // Handle tool calls
        const results: string[] = [];
        for (const tc of choice.message.tool_calls) {
          const args = JSON.parse(tc.function.arguments || "{}");
          const result = await executeToolCall(tc.function.name, args);
          results.push(`[${tc.function.name}]: ${result}`);
        }
        setMessages((prev) => [...prev, {
          role: "assistant",
          content: choice.message.content || `Executed: ${choice.message.tool_calls.map((tc: { function: { name: string } }) => tc.function.name).join(", ")}`,
          diff: results.find((r) => r.includes('"diff"'))?.match(/"diff":\s*"([^"]+)"/)?.[1],
        }]);
      } else {
        setMessages((prev) => [...prev, {
          role: "assistant",
          content: choice?.message?.content || "No response",
        }]);
      }
    } catch (err) {
      setMessages((prev) => [...prev, { role: "assistant", content: `Error: ${err}` }]);
    } finally {
      setLoading(false);
    }
  }, [input, loading, profile, apiKey, messages]);

  if (!profile) {
    return (
      <div className="home-chat">
        <h3>Chat</h3>
        <p style={{ opacity: 0.7 }}>Select a Chat model in Settings to enable the AI assistant.</p>
      </div>
    );
  }

  return (
    <div className="home-chat">
      <h3>Chat</h3>
      <div style={{ flex: 1, overflowY: "auto", marginBottom: 8 }}>
        {messages.map((msg, i) => (
          <div key={i} style={{ marginBottom: 8, textAlign: msg.role === "user" ? "right" : "left" }}>
            <div style={{
              display: "inline-block",
              background: msg.role === "user" ? "#2d3560" : "var(--panel)",
              padding: "8px 12px",
              borderRadius: 8,
              maxWidth: "90%",
              textAlign: "left",
            }}>
              <div style={{ whiteSpace: "pre-wrap" }}>{msg.content}</div>
              {msg.diff && <DiffViewer value={msg.diff} />}
            </div>
          </div>
        ))}
        {loading && <div style={{ opacity: 0.6 }}>Thinking...</div>}
        <div ref={bottomRef} />
      </div>
      <div style={{ display: "flex", gap: 8 }}>
        <input
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyDown={(e) => { if (e.key === "Enter" && !e.shiftKey) { e.preventDefault(); send(); } }}
          placeholder="Describe what you want to configure..."
          style={{ flex: 1 }}
        />
        <button onClick={send} disabled={loading}>Send</button>
      </div>
    </div>
  );
}
```

**Step 6: Add Chat to Home page**

In `src/pages/Home.tsx`, import and render the Chat component in the right panel:

```tsx
import { Chat } from "../components/Chat";
```

And at the end of the return, after `</div>` (closing `home-main`), add:

```tsx
      <Chat />
```

**Step 7: Add resolve_full_api_key Rust command**

The Chat component needs the actual (unmasked) API key to call LLM APIs. Add a command that returns the raw env var value:

In `src-tauri/src/commands.rs`:

```rust
#[tauri::command]
pub fn resolve_full_api_key(profile_id: String) -> Result<String, String> {
    let paths = resolve_paths();
    let profiles = load_model_profiles(&paths);
    let profile = profiles.iter().find(|p| p.id == profile_id)
        .ok_or_else(|| "Profile not found".to_string())?;
    std::env::var(&profile.auth_ref)
        .map_err(|_| format!("Environment variable {} not set", profile.auth_ref))
}
```

Register in `src-tauri/src/lib.rs`.

In `src/lib/api.ts`:

```typescript
  resolveFullApiKey: (profileId: string): Promise<string> =>
    invoke("resolve_full_api_key", { profileId }),
```

Update Chat.tsx `useEffect` to call `api.resolveFullApiKey(profileId)` and `setApiKey(key)`.

**Step 8: Verify build**

Run: `npm run build`

**Step 9: Commit**

```bash
git add src/components/Chat.tsx src/lib/chat.ts src/pages/Home.tsx src/lib/api.ts src/lib/types.ts src-tauri/src/commands.rs src-tauri/src/lib.rs
git commit -m "feat: add Chat panel with LLM tool-calling on Home page"
```

---

## Task 9: Auto-extract Model Profiles on first launch

**Files:**
- Modify: `src/pages/Settings.tsx`
- Modify: `src/App.tsx`

**Step 1: Add first-launch auto-extract logic**

In `src/App.tsx`, add a `useEffect` that checks localStorage for a flag, and if not set, calls `api.extractModelProfilesFromConfig()` then sets the flag:

```tsx
useEffect(() => {
  if (!localStorage.getItem("clawpal_profiles_extracted")) {
    api.extractModelProfilesFromConfig()
      .then(() => localStorage.setItem("clawpal_profiles_extracted", "1"))
      .catch(() => {}); // Ignore if no config exists yet
  }
}, []);
```

Import `api` at the top of App.tsx.

**Step 2: Verify build**

Run: `npm run build`

**Step 3: Commit**

```bash
git add src/App.tsx
git commit -m "feat: auto-extract model profiles from config on first launch"
```

---

## Task 10: Clean up unused Rust commands from lib.rs

**Files:**
- Modify: `src-tauri/src/lib.rs` — remove commands no longer used by any frontend page

**Step 1: Review and remove unused command registrations**

The following commands are no longer called from the frontend (Channels page deleted, Models binding UI removed, Data page merged):
- `update_channel_config` — was used by Channels page (deleted)
- `delete_channel_node` — was used by Channels page (deleted)
- `set_global_model` — was used by Models page (deleted)
- `set_agent_model` — was used by Models page (deleted)
- `set_channel_model` — was used by Models page (deleted)
- `list_model_bindings` — was used by Models page (deleted)
- `list_channels` — was used by Channels page (deleted)
- `list_channels_minimal` — was used by Models page (deleted)

**Keep** these (still used):
- All Recipe/History/Doctor commands
- `list_model_profiles`, `upsert_model_profile`, `delete_model_profile`
- `list_model_catalog`, `extract_model_profiles_from_config`
- `list_memory_files`, `clear_memory`, `delete_memory_file` (Doctor)
- `list_session_files`, `clear_all_sessions`, `clear_agent_sessions`, `delete_session_file` (Doctor)
- `list_agent_ids` (still used internally)
- `resolve_api_keys`, `resolve_full_api_key`, `list_agents_overview`, `read_raw_config` (new)

Remove unused commands from lib.rs `generate_handler!` and remove their entries from the `use` import. Keep the Rust function implementations in commands.rs — they may be useful for Chat tool calling later.

Also clean up `src/lib/api.ts` — remove API calls that are no longer used:
- `updateChannelConfig`
- `deleteChannelNode`
- `setGlobalModel`
- `setAgentModel`
- `setChannelModel`
- `listModelBindings`
- `listChannels`
- `listChannelsMinimal`

And clean up `src/lib/types.ts` — remove types no longer used:
- `ChannelNode`
- `ModelBinding`

**Step 2: Verify build**

Run: `npm run build`

**Step 3: Commit**

```bash
git add src-tauri/src/lib.rs src/lib/api.ts src/lib/types.ts
git commit -m "chore: remove unused channel/model binding commands and types"
```

---

## Task 11: Final verification and cleanup

**Step 1: Full build check**

```bash
cd /Users/zhixian/Codes/clawpal && npm run build
```

**Step 2: Rust build check**

```bash
cd /Users/zhixian/Codes/clawpal/src-tauri && cargo build
```

**Step 3: Type check**

```bash
cd /Users/zhixian/Codes/clawpal && npm run typecheck
```

**Step 4: Verify file structure**

Confirm the following pages exist:
- `src/pages/Home.tsx` — redesigned
- `src/pages/Recipes.tsx` — unchanged
- `src/pages/Install.tsx` — unchanged
- `src/pages/History.tsx` — unchanged
- `src/pages/Doctor.tsx` — expanded with data cleanup
- `src/pages/Settings.tsx` — new

Confirm the following pages are deleted:
- `src/pages/Models.tsx`
- `src/pages/Channels.tsx`
- `src/pages/Data.tsx`

**Step 5: Commit any remaining fixes**

```bash
git add -A
git commit -m "chore: final cleanup after product redesign"
```
