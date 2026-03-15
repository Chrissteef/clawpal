# Doctor Agent 2x2 Matrix Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Decouple agent source from execution target so ClawPal can diagnose both local and remote machines from any gateway.

**Architecture:** Add `execute_remote_command()` parallel to existing `execute_local_command()`, route via a `target` parameter on `doctor_approve_invoke`. Add unified sensitive path blacklist. Frontend auto-infers target from active instance tab with manual override.

**Tech Stack:** Rust (Tauri, tokio, serde_json), TypeScript (React, Tauri event API), SSH via existing `SshConnectionPool`

---

### Task 1: Add sensitive path blacklist to Rust backend

Adds a hard-block security layer that prevents reading or writing sensitive paths (`.ssh/`, `.gnupg/`, etc.) regardless of whether execution is local or remote. This check runs before any other path validation.

**Files:**
- Modify: `src-tauri/src/doctor_commands.rs:144-199` (insert before `allowed_read_dirs`)

**Step 1: Add the blacklist constant and validation function**

Insert at line 144 (before `allowed_read_dirs`):

```rust
/// Sensitive paths that are ALWAYS blocked for both read and write.
/// Checked after tilde expansion, before any other path validation.
const SENSITIVE_PATH_PATTERNS: &[&str] = &[
    "/.ssh/",
    "/.ssh",
    "/.gnupg/",
    "/.gnupg",
    "/.aws/",
    "/.aws",
    "/.config/gcloud/",
    "/.azure/",
    "/.kube/config",
    "/.docker/config.json",
    "/.netrc",
    "/.env",
    "/.bash_history",
    "/.zsh_history",
    "/etc/shadow",
    "/etc/sudoers",
];

fn validate_not_sensitive(path: &str) -> Result<(), String> {
    let expanded = shellexpand::tilde(path).to_string();
    for pattern in SENSITIVE_PATH_PATTERNS {
        if expanded.contains(pattern) {
            return Err(format!(
                "Access to {path} is blocked — matches sensitive path pattern: {pattern}"
            ));
        }
    }
    Ok(())
}
```

**Step 2: Wire blacklist into `validate_read_path`**

In `validate_read_path` (currently line 162), add as the first line of the function body:

```rust
validate_not_sensitive(path)?;
```

**Step 3: Wire blacklist into `validate_write_path`**

In `validate_write_path` (currently line 180), add as the first line of the function body:

```rust
validate_not_sensitive(path)?;
```

**Step 4: Wire blacklist into `execute_local_command`**

In the `read_file` arm (line 275-283), the blacklist is already enforced via `validate_read_path`. Same for `write_file` via `validate_write_path`. For `list_files`, also already covered. No additional changes needed in `execute_local_command` itself.

**Step 5: Verify it compiles**

Run: `cd /Users/zhixian/Codes/clawpal/src-tauri && cargo check 2>&1 | tail -5`
Expected: no errors

**Step 6: Commit**

```bash
git add src-tauri/src/doctor_commands.rs
git commit -m "security: add sensitive path blacklist for doctor agent"
```

---

### Task 2: Add `execute_remote_command` function

Creates a parallel execution path that routes doctor agent tool calls to a remote SSH host instead of the local filesystem. Uses `SshConnectionPool` methods (`exec`, `sftp_read`, `sftp_write`, `sftp_list`).

**Files:**
- Modify: `src-tauri/src/doctor_commands.rs` (add new function after `execute_local_command`)

**Step 1: Add the function**

Insert after `execute_local_command` (after current line 377):

```rust
/// Execute a command on a remote SSH host on behalf of the doctor agent.
async fn execute_remote_command(
    pool: &SshConnectionPool,
    host_id: &str,
    command: &str,
    args: &Value,
) -> Result<Value, String> {
    match command {
        "read_file" => {
            let path = args.get("path").and_then(|v| v.as_str())
                .ok_or("read_file: missing 'path' argument")?;
            validate_not_sensitive(path)?;
            let content = pool.sftp_read(host_id, path).await?;
            Ok(json!({"content": content}))
        }
        "list_files" => {
            let path = args.get("path").and_then(|v| v.as_str())
                .ok_or("list_files: missing 'path' argument")?;
            validate_not_sensitive(path)?;
            let entries = pool.sftp_list(host_id, path).await?;
            Ok(json!({"entries": entries.iter().map(|e| json!({
                "name": e.name,
                "isDir": e.is_dir,
                "size": e.size,
            })).collect::<Vec<_>>()}))
        }
        "read_config" => {
            // Try standard config locations on remote
            let result = pool.exec_login(host_id, "openclaw config-path 2>/dev/null || echo ~/.config/openclaw/openclaw.json").await?;
            let config_path = result.stdout.trim().to_string();
            let content = pool.sftp_read(host_id, &config_path).await
                .unwrap_or_else(|_| "(unable to read remote config)".into());
            Ok(json!({"content": content, "path": config_path}))
        }
        "system_info" => {
            let version_result = pool.exec_login(host_id, "openclaw --version 2>/dev/null || echo unknown").await?;
            let uname_result = pool.exec(host_id, "uname -a").await?;
            let hostname_result = pool.exec(host_id, "hostname").await?;
            Ok(json!({
                "platform": uname_result.stdout.split_whitespace().next().unwrap_or("unknown").to_lowercase(),
                "arch": uname_result.stdout.split_whitespace().nth(12).unwrap_or("unknown"),
                "openclawVersion": version_result.stdout.trim(),
                "hostname": hostname_result.stdout.trim(),
                "remote": true,
            }))
        }
        "validate_config" => {
            let result = pool.exec_login(host_id, "openclaw doctor --json 2>/dev/null").await?;
            if result.exit_code != 0 {
                return Ok(json!({
                    "ok": false,
                    "error": format!("openclaw doctor failed: {}", result.stderr.trim()),
                    "raw": result.stdout,
                }));
            }
            // Parse the JSON output from openclaw doctor
            let parsed: Value = serde_json::from_str(&result.stdout)
                .unwrap_or_else(|_| json!({"raw": result.stdout.trim()}));
            Ok(parsed)
        }
        "write_file" => {
            let path = args.get("path").and_then(|v| v.as_str())
                .ok_or("write_file: missing 'path' argument")?;
            let content = args.get("content").and_then(|v| v.as_str())
                .ok_or("write_file: missing 'content' argument")?;
            validate_not_sensitive(path)?;
            // Check for symlink on remote before writing
            let stat_result = pool.exec(host_id, &format!("test -L {} && echo SYMLINK || echo OK", shell_quote_for_remote(path))).await?;
            if stat_result.stdout.trim() == "SYMLINK" {
                return Err(format!("write_file: refusing to write through symlink at {path}"));
            }
            pool.sftp_write(host_id, path, content).await?;
            Ok(json!({"ok": true}))
        }
        "run_command" => {
            let cmd = args.get("command").and_then(|v| v.as_str())
                .ok_or("run_command: missing 'command' argument")?;
            validate_command(cmd)?;
            let result = pool.exec(host_id, cmd).await?;
            Ok(json!({
                "stdout": truncate_output(result.stdout.as_bytes()),
                "stderr": truncate_output(result.stderr.as_bytes()),
                "exitCode": result.exit_code,
            }))
        }
        _ => Err(format!("Unknown command: {command}")),
    }
}

/// Shell-quote a path for remote commands (prevents injection in the symlink check).
fn shell_quote_for_remote(s: &str) -> String {
    format!("'{}'", s.replace('\'', "'\\''"))
}
```

**Step 2: Verify it compiles**

Run: `cd /Users/zhixian/Codes/clawpal/src-tauri && cargo check 2>&1 | tail -5`
Expected: no errors (the function is not called yet, but should compile)

**Step 3: Commit**

```bash
git add src-tauri/src/doctor_commands.rs
git commit -m "feat: add execute_remote_command for SSH-based doctor diagnosis"
```

---

### Task 3: Add `collect_doctor_context_remote` Tauri command

Collects diagnostic context from a remote SSH host in a single call, returning the same JSON shape as `collect_doctor_context` so the gateway sees an identical context payload regardless of target.

**Files:**
- Modify: `src-tauri/src/doctor_commands.rs` (add new Tauri command after `collect_doctor_context`)
- Modify: `src-tauri/src/lib.rs:220` (register the new command)

**Step 1: Add the Tauri command**

Insert after `collect_doctor_context` (after current line 125):

```rust
#[tauri::command]
pub async fn collect_doctor_context_remote(
    pool: State<'_, SshConnectionPool>,
    host_id: String,
) -> Result<String, String> {
    // Collect openclaw version
    let version_result = pool.exec_login(&host_id, "openclaw --version 2>/dev/null || echo unknown").await?;
    let version = version_result.stdout.trim().to_string();

    // Collect config path and content
    let config_path_result = pool.exec_login(&host_id, "openclaw config-path 2>/dev/null || echo ~/.config/openclaw/openclaw.json").await?;
    let config_path = config_path_result.stdout.trim().to_string();
    let config_content = pool.sftp_read(&host_id, &config_path).await
        .unwrap_or_else(|_| "(unable to read remote config)".into());

    // Run doctor on remote
    let doctor_result = pool.exec_login(&host_id, "openclaw doctor --json 2>/dev/null").await?;
    let doctor_report: Value = serde_json::from_str(&doctor_result.stdout)
        .unwrap_or_else(|_| json!({
            "ok": false,
            "error": "Failed to parse doctor output",
            "raw": doctor_result.stdout.trim(),
        }));

    // Collect recent error log
    let error_log_result = pool.exec(&host_id, "tail -100 ~/.config/openclaw/error.log 2>/dev/null || echo ''").await?;
    let error_log = error_log_result.stdout;

    // System info
    let uname_result = pool.exec(&host_id, "uname -s").await?;
    let arch_result = pool.exec(&host_id, "uname -m").await?;

    let context = json!({
        "openclawVersion": version,
        "configPath": config_path,
        "configContent": config_content,
        "doctorReport": doctor_report,
        "errorLog": error_log,
        "platform": uname_result.stdout.trim().to_lowercase(),
        "arch": arch_result.stdout.trim(),
        "remote": true,
        "hostId": host_id,
    });

    serde_json::to_string(&context).map_err(|e| format!("Failed to serialize context: {e}"))
}
```

**Step 2: Register the command in lib.rs**

In `src-tauri/src/lib.rs`, find the `generate_handler!` invocation (around line 214-223). Add `collect_doctor_context_remote` after `collect_doctor_context`:

Change:
```
            collect_doctor_context,
            doctor_ssh_forward,
            doctor_ssh_forward_close,
```
To:
```
            collect_doctor_context,
            collect_doctor_context_remote,
```

This also removes the SSH forward stubs (per design).

**Step 3: Remove SSH forward stubs from doctor_commands.rs**

Delete the `doctor_ssh_forward` and `doctor_ssh_forward_close` functions (current lines 127-142).

**Step 4: Verify it compiles**

Run: `cd /Users/zhixian/Codes/clawpal/src-tauri && cargo check 2>&1 | tail -5`
Expected: no errors

**Step 5: Commit**

```bash
git add src-tauri/src/doctor_commands.rs src-tauri/src/lib.rs
git commit -m "feat: add collect_doctor_context_remote, remove SSH forward stubs"
```

---

### Task 4: Modify `doctor_approve_invoke` to accept target parameter

The key routing change: the `target` parameter determines whether a tool call executes locally or on a remote SSH host.

**Files:**
- Modify: `src-tauri/src/doctor_commands.rs:51-75` (change function signature and dispatch logic)

**Step 1: Update the function signature and dispatch**

Replace the current `doctor_approve_invoke` function (lines 51-75) with:

```rust
#[tauri::command]
pub async fn doctor_approve_invoke(
    client: State<'_, NodeClient>,
    pool: State<'_, SshConnectionPool>,
    app: AppHandle,
    invoke_id: String,
    target: String,
) -> Result<Value, String> {
    let invoke = client.take_invoke(&invoke_id).await
        .ok_or_else(|| format!("No pending invoke with id: {invoke_id}"))?;

    let command = invoke.get("command").and_then(|v| v.as_str()).unwrap_or("");
    let args = invoke.get("args").cloned().unwrap_or(Value::Null);

    // Route to local or remote execution
    let result = if target == "local" {
        execute_local_command(command, &args).await?
    } else {
        execute_remote_command(&pool, &target, command, &args).await?
    };

    // Send result back to gateway
    client.send_response(&invoke_id, result.clone()).await?;

    let _ = app.emit("doctor:invoke-result", json!({
        "id": invoke_id,
        "result": result,
    }));

    Ok(result)
}
```

**Step 2: Verify it compiles**

Run: `cd /Users/zhixian/Codes/clawpal/src-tauri && cargo check 2>&1 | tail -5`
Expected: no errors

**Step 3: Commit**

```bash
git add src-tauri/src/doctor_commands.rs
git commit -m "feat: route doctor_approve_invoke to local or remote based on target"
```

---

### Task 5: Update frontend API layer

Updates API signatures: `doctorApproveInvoke` gains a `target` parameter, adds `collectDoctorContextRemote`, removes SSH forward methods.

**Files:**
- Modify: `src/lib/api.ts:268-286`
- Modify: `src/lib/use-api.ts:201-210`

**Step 1: Update api.ts**

Replace the Doctor Agent section (lines 268-286) with:

```typescript
  // Doctor Agent
  doctorConnect: (url: string): Promise<void> =>
    invoke("doctor_connect", { url }),
  doctorDisconnect: (): Promise<void> =>
    invoke("doctor_disconnect"),
  doctorStartDiagnosis: (context: string): Promise<void> =>
    invoke("doctor_start_diagnosis", { context }),
  doctorSendMessage: (message: string): Promise<void> =>
    invoke("doctor_send_message", { message }),
  doctorApproveInvoke: (invokeId: string, target: string): Promise<Record<string, unknown>> =>
    invoke("doctor_approve_invoke", { invokeId, target }),
  doctorRejectInvoke: (invokeId: string, reason: string): Promise<void> =>
    invoke("doctor_reject_invoke", { invokeId, reason }),
  collectDoctorContext: (): Promise<string> =>
    invoke("collect_doctor_context"),
  collectDoctorContextRemote: (hostId: string): Promise<string> =>
    invoke("collect_doctor_context_remote", { hostId }),
```

**Step 2: Update use-api.ts**

Replace the Doctor Agent section (lines 201-210) with:

```typescript
      // Doctor Agent (local-only, no remote dispatch)
      doctorConnect: api.doctorConnect,
      doctorDisconnect: api.doctorDisconnect,
      doctorStartDiagnosis: api.doctorStartDiagnosis,
      doctorSendMessage: api.doctorSendMessage,
      doctorApproveInvoke: api.doctorApproveInvoke,
      doctorRejectInvoke: api.doctorRejectInvoke,
      collectDoctorContext: api.collectDoctorContext,
      collectDoctorContextRemote: api.collectDoctorContextRemote,
```

**Step 3: Verify TypeScript compiles**

Run: `cd /Users/zhixian/Codes/clawpal && npx tsc --noEmit 2>&1 | tail -10`
Expected: errors in use-doctor-agent.ts and Doctor.tsx (callers not updated yet — that's OK, we fix those next)

**Step 4: Commit**

```bash
git add src/lib/api.ts src/lib/use-api.ts
git commit -m "feat: update doctor API signatures for target parameter"
```

---

### Task 6: Update `useDoctorAgent` hook with target state and approval patterns

Adds `target` state, `approvedPatterns` set for read auto-approval caching, and passes `target` through to the API.

**Files:**
- Modify: `src/lib/use-doctor-agent.ts` (multiple sections)

**Step 1: Add new state variables**

After line 16 (`const [error, setError] = ...`), add:

```typescript
  const [target, setTarget] = useState("local");
  const [approvedPatterns, setApprovedPatterns] = useState<Set<string>>(new Set());
```

**Step 2: Update the `doctor:invoke` handler to use pattern-based approval**

Replace the current invoke handler (lines 57-69) with:

```typescript
      listen<DoctorInvoke>("doctor:invoke", (e) => {
        const invoke = e.payload;
        setPendingInvokes((prev) => new Map(prev).set(invoke.id, invoke));
        setMessages((prev) => [
          ...prev,
          { id: nextMsgId(), role: "tool-call", content: invoke.command, invoke, status: "pending" },
        ]);

        // Auto-approve read commands if pattern already approved
        if (invoke.type === "read") {
          const pattern = extractApprovalPattern(invoke);
          setApprovedPatterns((prev) => {
            if (prev.has(pattern)) {
              autoApprove(invoke.id);
              return prev;
            }
            // First time: show in chat, wait for user click — don't auto-approve
            return prev;
          });
        }
      }),
```

**Step 3: Add the pattern extraction helper**

Before the `useDoctorAgent` function (above line 11), add:

```typescript
function extractApprovalPattern(invoke: DoctorInvoke): string {
  const path = (invoke.args?.path as string) ?? "";
  // Use directory prefix as the pattern key
  const prefix = path.includes("/") ? path.substring(0, path.lastIndexOf("/") + 1) : path;
  return `${invoke.command}:${prefix}`;
}
```

**Step 4: Update `autoApprove` to record patterns and pass target**

Replace the `autoApprove` callback (lines 96-109) with:

```typescript
  const autoApprove = useCallback(async (invokeId: string) => {
    try {
      await api.doctorApproveInvoke(invokeId, target);
      setMessages((prev) =>
        prev.map((m) => {
          if (m.invoke?.id === invokeId && m.role === "tool-call") {
            // Record the pattern for future auto-approval
            const pattern = extractApprovalPattern(m.invoke);
            setApprovedPatterns((prev) => new Set(prev).add(pattern));
            return { ...m, status: "auto" as const };
          }
          return m;
        })
      );
    } catch (err) {
      setError(`Auto-approve failed: ${err}`);
    }
  }, [target]);
```

**Step 5: Update `approveInvoke` to pass target and record patterns**

Replace the `approveInvoke` callback (lines 154-167) with:

```typescript
  const approveInvoke = useCallback(async (invokeId: string) => {
    setMessages((prev) =>
      prev.map((m) => {
        if (m.invoke?.id === invokeId && m.role === "tool-call") {
          if (m.invoke) {
            const pattern = extractApprovalPattern(m.invoke);
            setApprovedPatterns((p) => new Set(p).add(pattern));
          }
          return { ...m, status: "approved" as const };
        }
        return m;
      })
    );
    try {
      await api.doctorApproveInvoke(invokeId, target);
    } catch (err) {
      setError(`Approve failed: ${err}`);
    }
  }, [target]);
```

**Step 6: Update `reset` to clear approval patterns**

Replace the `reset` callback (lines 189-195) with:

```typescript
  const reset = useCallback(() => {
    setMessages([]);
    setPendingInvokes(new Map());
    setLoading(false);
    setError(null);
    setApprovedPatterns(new Set());
    streamingRef.current = "";
  }, []);
```

**Step 7: Add target and setTarget to the return value**

In the return object (line 197), add:

```typescript
    target,
    setTarget,
    approvedPatterns,
```

**Step 8: Verify TypeScript compiles (may still have Doctor.tsx errors)**

Run: `cd /Users/zhixian/Codes/clawpal && npx tsc --noEmit 2>&1 | tail -10`

**Step 9: Commit**

```bash
git add src/lib/use-doctor-agent.ts
git commit -m "feat: add target state and approval pattern tracking to doctor hook"
```

---

### Task 7: Update Doctor.tsx — target selector and context collection

Adds target auto-inference from active instance tab, a "Change target" override, and routes context collection to local or remote based on target.

**Files:**
- Modify: `src/pages/Doctor.tsx:38-79` (state, start/stop handlers)
- Modify: `src/pages/Doctor.tsx:331-380` (agent section UI)

**Step 1: Update state and auto-infer target**

Replace lines 38-47 (agent source state + instance reset effect) with:

```typescript
  // Agent source state
  const [agentSource, setAgentSource] = useState<AgentSource>("hosted");
  const [diagnosing, setDiagnosing] = useState(false);

  // Auto-infer target from active instance tab
  useEffect(() => {
    if (isRemote && isConnected) {
      doctor.setTarget(instanceId);
    } else {
      doctor.setTarget("local");
    }
  }, [instanceId, isRemote, isConnected, doctor.setTarget]);

  // Reset doctor agent when switching instances
  useEffect(() => {
    doctor.reset();
    doctor.disconnect();
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, [instanceId]);
```

**Step 2: Update `handleStartDiagnosis` to use target**

Replace lines 49-71 with:

```typescript
  const handleStartDiagnosis = async () => {
    setDiagnosing(true);
    try {
      // Determine URL based on source
      let url: string;
      if (agentSource === "local") {
        url = "ws://localhost:18789";
      } else {
        url = "wss://doctor.openclaw.ai";
      }

      await doctor.connect(url);

      // Collect context based on target (not source)
      const context = doctor.target === "local"
        ? await api.collectDoctorContext()
        : await api.collectDoctorContextRemote(doctor.target);

      await doctor.startDiagnosis(context);
    } catch (err) {
      doctor.reset();
    } finally {
      setDiagnosing(false);
    }
  };
```

**Step 3: Simplify `handleStopDiagnosis`**

Replace lines 73-79 with:

```typescript
  const handleStopDiagnosis = async () => {
    await doctor.disconnect();
    doctor.reset();
  };
```

**Step 4: Update the agent section UI to show target info**

Find the Doctor Agent Section card (around line 331). After the `<CardTitle>` and before the source selector, add a target display:

```tsx
          {/* Target display */}
          <div className="flex items-center gap-2 mb-3 text-sm">
            <span className="text-muted-foreground">{t("doctor.target")}:</span>
            <span className="font-medium">
              {doctor.target === "local" ? t("doctor.localMachine") : doctor.target}
            </span>
          </div>
```

**Step 5: Remove SSH source option from radio buttons (deferred)**

In the agent source radio group, remove the SSH radio button entirely (the disabled one at lines 351-363). Keep only Local Gateway and Hosted Service.

**Step 6: Pass target to DoctorChat's approve handler**

Find where `DoctorChat` is rendered (search for `<DoctorChat`). Ensure `onApproveInvoke` passes through correctly — since the hook now handles target internally, no change needed in DoctorChat props. But verify that the `onApproveInvoke` prop calls `doctor.approveInvoke(invokeId)` which already uses `target` internally.

**Step 7: Verify TypeScript compiles**

Run: `cd /Users/zhixian/Codes/clawpal && npx tsc --noEmit 2>&1 | tail -10`
Expected: no errors (or only unrelated ones)

**Step 8: Commit**

```bash
git add src/pages/Doctor.tsx
git commit -m "feat: add target auto-inference and remote context collection to Doctor page"
```

---

### Task 8: Add i18n translation keys

Adds new translation keys for target display and approval UX.

**Files:**
- Modify: `src/locales/en.json`
- Modify: `src/locales/zh.json`

**Step 1: Add English translations**

After the `"doctor.comingSoon"` line (line 199 in en.json), add:

```json
  "doctor.target": "Target",
  "doctor.localMachine": "Local machine",
  "doctor.changeTarget": "Change",
  "doctor.allowRead": "Allow",
  "doctor.firstTimeApproval": "First access — click Allow to auto-approve future reads to this path",
```

**Step 2: Add Chinese translations**

After the corresponding `"doctor.comingSoon"` line in zh.json, add:

```json
  "doctor.target": "诊断目标",
  "doctor.localMachine": "本地",
  "doctor.changeTarget": "更改",
  "doctor.allowRead": "允许",
  "doctor.firstTimeApproval": "首次访问 — 点击允许后将自动执行该路径下的后续读取",
```

**Step 3: Commit**

```bash
git add src/locales/en.json src/locales/zh.json
git commit -m "i18n: add doctor target and approval translation keys"
```

---

### Task 9: Update DoctorChat for first-time read approval UX

Currently all reads auto-execute silently. Now first-time reads show an "Allow" button; once approved, future reads to the same path prefix auto-execute.

**Files:**
- Modify: `src/components/DoctorChat.tsx:136-173` (tool-call rendering)

**Step 1: Update DoctorChatProps to include approvedPatterns**

At the top of the file, update the `DoctorChatProps` interface (line 10-18):

Add after `onRejectInvoke`:

```typescript
  approvedPatterns?: Set<string>;
```

**Step 2: Update the tool-call card for pending reads**

In the `MessageBubble` component, find the `isPending` check (line 138). The current logic only shows Execute/Skip for writes. Update to show "Allow" for first-time reads:

Replace lines 138-147 with:

```typescript
    const isPendingWrite = message.status === "pending" && inv.type === "write";
    const isPendingRead = message.status === "pending" && inv.type === "read";
    const statusBadge = message.status === "auto"
      ? <Badge variant="outline" className="text-xs">{t("doctor.autoExecuted")}</Badge>
      : message.status === "approved"
        ? <Badge variant="secondary" className="text-xs">{t("doctor.execute")}</Badge>
        : message.status === "rejected"
          ? <Badge variant="destructive" className="text-xs">{t("doctor.rejected")}</Badge>
          : isPendingRead
            ? <Badge variant="outline" className="text-xs">{t("doctor.firstTimeApproval")}</Badge>
            : message.status === "pending" && inv.type === "write"
              ? <Badge variant="secondary" className="text-xs">{t("doctor.awaitingApproval")}</Badge>
              : null;
```

Update the buttons section (lines 154-163) to also show an "Allow" button for pending reads:

```tsx
            {isPendingWrite && (
              <>
                <Button size="sm" variant="default" onClick={() => onApprove(inv.id)}>
                  {t("doctor.execute")}
                </Button>
                <Button size="sm" variant="outline" onClick={() => onReject(inv.id)}>
                  {t("doctor.skip")}
                </Button>
              </>
            )}
            {isPendingRead && (
              <Button size="sm" variant="outline" onClick={() => onApprove(inv.id)}>
                {t("doctor.allowRead")}
              </Button>
            )}
```

**Step 3: Verify TypeScript compiles**

Run: `cd /Users/zhixian/Codes/clawpal && npx tsc --noEmit 2>&1 | tail -10`
Expected: no errors

**Step 4: Commit**

```bash
git add src/components/DoctorChat.tsx
git commit -m "feat: add first-time read approval UX in DoctorChat"
```

---

### Task 10: Full build verification

Ensures everything compiles cleanly: Rust backend, TypeScript frontend, and production Vite build.

**Files:** None (verification only)

**Step 1: Rust check**

Run: `cd /Users/zhixian/Codes/clawpal/src-tauri && cargo check 2>&1 | tail -10`
Expected: no errors, no warnings related to doctor code

**Step 2: TypeScript check**

Run: `cd /Users/zhixian/Codes/clawpal && npx tsc --noEmit 2>&1 | tail -10`
Expected: no errors

**Step 3: Vite production build**

Run: `cd /Users/zhixian/Codes/clawpal && npx vite build 2>&1 | tail -10`
Expected: build succeeds

**Step 4: Fix any issues found, commit fixes**

If any errors are found, fix them and create a new commit:

```bash
git add -A
git commit -m "fix: resolve build issues from doctor 2x2 matrix implementation"
```
