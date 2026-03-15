# Recipe Engine Redesign

## Goal

Redesign the recipe system from "config patch templates" to "composable atomic operations with a step-by-step wizard UI".

## Core Concept

**Recipe = an ordered sequence of atomic operations (steps), with a unified parameter form and wizard-style execution.**

A recipe contains:
- **Metadata**: id, name, description, version, tags, difficulty
- **Params**: unified parameter pool, user fills one form upfront
- **Steps**: ordered list of operations, each referencing a registered action type and pulling values from the params pool via `{{param_id}}` substitution

## Architecture

**Frontend orchestration (Plan B)**: the front end executes steps sequentially by calling existing Tauri commands. No new backend execution engine needed. Existing API calls (`createAgent`, `setupAgentIdentity`, `assignChannelAgent`, `applyRecipe`, etc.) serve as the atomic operations.

Rationale: all atomic operations already exist as Tauri commands; transaction/rollback is acceptable at current scale (user can discard config changes); migration to backend execution is straightforward later if needed.

## Recipe JSON Format

```json
{
  "recipes": [
    {
      "id": "dedicated-channel-agent",
      "name": "Create dedicated Agent for Channel",
      "description": "Create an independent agent, set its identity, bind it to a Discord channel, and configure persona",
      "version": "1.0.0",
      "tags": ["discord", "agent", "persona"],
      "difficulty": "easy",
      "params": [
        { "id": "agent_id", "label": "Agent ID", "type": "string", "required": true, "placeholder": "e.g. my-bot" },
        { "id": "guild_id", "label": "Guild", "type": "discord_guild", "required": true },
        { "id": "channel_id", "label": "Channel", "type": "discord_channel", "required": true },
        { "id": "name", "label": "Display Name", "type": "string", "required": true, "placeholder": "e.g. MyBot" },
        { "id": "emoji", "label": "Emoji", "type": "string", "required": false, "placeholder": "e.g. \ud83e\udd16" },
        { "id": "persona", "label": "Persona", "type": "textarea", "required": true, "placeholder": "You are..." }
      ],
      "steps": [
        {
          "action": "create_agent",
          "label": "Create independent agent",
          "args": { "agentId": "{{agent_id}}", "independent": true }
        },
        {
          "action": "setup_identity",
          "label": "Set agent identity",
          "args": { "agentId": "{{agent_id}}", "name": "{{name}}", "emoji": "{{emoji}}" }
        },
        {
          "action": "bind_channel",
          "label": "Bind channel to agent",
          "args": { "channelType": "discord", "peerId": "{{channel_id}}", "agentId": "{{agent_id}}" }
        },
        {
          "action": "config_patch",
          "label": "Set channel persona",
          "args": {
            "patchTemplate": "{\"channels\":{\"discord\":{\"guilds\":{\"{{guild_id}}\":{\"channels\":{\"{{channel_id}}\":{\"systemPrompt\":\"{{persona}}\"}}}}}}}"
          }
        }
      ]
    }
  ]
}
```

## Action Registry

A frontend TypeScript map. Each action type registers:

1. **execute(args)** — calls the corresponding API
2. **describe(args)** — generates a human-readable description for the wizard confirmation step

| Action Type | API Call | Description Example |
|---|---|---|
| `create_agent` | `api.createAgent(agentId, modelProfileId, independent)` | "Create independent agent `my-bot`" |
| `setup_identity` | `api.setupAgentIdentity(agentId, name, emoji)` | "Set identity: MyBot \ud83e\udd16" |
| `bind_channel` | `api.assignChannelAgent(channelType, peerId, agentId)` | "Bind Discord channel \u2192 agent `my-bot`" |
| `config_patch` | `api.applyRecipe(...)` (reuses existing merge patch engine) | Uses step label from recipe JSON |
| `set_global_model` | `api.setGlobalModel(profileId)` | "Set default model to openai/gpt-4o" |

New action types are added by inserting one entry in this map — no backend changes needed.

## Wizard Execution UX (Cook Page)

### Phase 1: Fill Parameters
- Single form with all params (same as current ParamForm, with all existing param types supported)
- Button: "Next"

### Phase 2: Confirm Steps
- Vertical step list showing each operation with its human-readable description (from `describe()`)
- Each step shows a pending icon (\u25cb)
- Buttons: "Execute" / "Back"

### Phase 3: Execute
- Steps execute sequentially, status updates in real-time:
  - \u25cb pending \u2192 \u25c9 running \u2192 \u2713 done / \u2717 failed
- On failure: stop, show error, offer "Retry" or "Skip"
- No automatic rollback (user can use sidebar Discard if needed)

### Phase 4: Complete
- Summary: N succeeded, M skipped
- If any `config_patch` steps ran: "Use Apply Changes in sidebar to activate"
- Button: "Back to Recipes"

## Built-in Recipes

| Recipe | Steps | Target User |
|---|---|---|
| **Create dedicated Agent for Channel** | create_agent \u2192 setup_identity \u2192 bind_channel \u2192 config_patch(persona) | Core multi-step recipe |
| **Channel Persona** | config_patch(systemPrompt) | Single-step, for quickly setting persona on existing channel |

## External Recipe Loading

- Recipes tab keeps the existing "Load from URL/file path" input
- External recipes use the same format and the same registered action types
- Future: Tauri deep-link plugin (`clawpal://recipe?url=...`) for one-click loading from websites. Deferred to post-launch.

## Cleanup (Old Format Removal)

Remove from codebase:
- `patchTemplate` field on Recipe type (Rust + TS)
- `action` field on Recipe type (TS only)
- `impactCategory` field
- `src/lib/recipe_catalog.ts` (dead code)
- Existing `handleCustomAction` logic in Cook.tsx
- `build_candidate_config` usage in `preview_apply` / `apply_recipe` for standalone recipe apply (keep the merge patch engine itself for `config_patch` action)

## Files to Modify

**Backend (Rust):**
- `src-tauri/recipes.json` — new format with steps
- `src-tauri/src/recipe.rs` — update `Recipe` struct (add steps, remove patchTemplate), update `load_recipes`/`validate`

**Frontend (TypeScript):**
- `src/lib/types.ts` — update `Recipe`, `RecipeParam` types, add `RecipeStep` type
- `src/lib/api.ts` — possibly add helper for config_patch step execution
- `src/lib/actions.ts` — NEW: action registry (execute + describe for each action type)
- `src/pages/Cook.tsx` — rewrite as wizard (4 phases)
- `src/components/ParamForm.tsx` — minor: button text from prop (already done)
- `src/pages/Recipes.tsx` — minor: update if Recipe type fields changed
- `src/components/RecipeCard.tsx` — minor: remove impactCategory display if removed
- Delete `src/lib/recipe_catalog.ts`
