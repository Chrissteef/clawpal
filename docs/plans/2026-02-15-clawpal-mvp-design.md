# ClawPal MVP 设计文档（实现版）

日期：2026-02-15  
版本：MVP-1.0  
目标：用最小投入实现可用产品，覆盖 `design.md` 中 MVP 核心范围（安装向导、快照与回滚、配置诊断）。

## 1. 范围边界

### 1.1 本版实现范围（MVP）

- 安装向导
  - Recipe 列表（内置静态 Recipes）
  - 参数 schema 校验（必填、类型、pattern）
  - 预览变更（diff）
  - 应用配置（含自动备份）
- 版本控制与回滚
  - 每次写入前自动快照
  - 历史记录列表
  - 选中历史版本回滚（回滚前显示 diff）
  - 回滚过程二次确认
- 配置诊断（Doctor）
  - JSON 解析
  - 必填字段检查
  - 端口占用（仅基础检查）
  - 文件权限检查（读写）
  - 一键修复建议（语法修复、默认值补齐）

### 1.2 明确不做

- 官网、用户系统、在线提交与评论
- 复杂的配置兼容迁移（v1/v2）
- 深度路径白名单策略（保留可配置白名单雏形）
- 插件化/市场化的远程 Recipe 发布

## 2. 目标架构（Tauri 2 + React）

### 2.1 分层

- `src-tauri/src`
  - `main.rs`：初始化、窗口配置、命令注册
  - `commands.rs`：所有跨端调用入口
  - `recipe.rs`：Recipe 定义、参数校验、模板渲染
  - `config_io.rs`：配置读写（路径发现、json5 解析、备份文件读写）
  - `history.rs`：快照目录与元数据管理
  - `doctor.rs`：诊断与修复策略
- `src/`
  - `pages/`
    - `Home.tsx`：健康状态、版本、按钮入口
    - `Recipes.tsx`：卡片列表与搜索过滤
    - `Install.tsx`：安装向导
    - `History.tsx`：历史快照
    - `Doctor.tsx`：问题列表与修复动作
  - `components/`
    - `RecipeCard.tsx`
    - `ParamForm.tsx`
    - `DiffViewer.tsx`
    - `StatusPill.tsx`
  - `lib/`
    - `recipe_catalog.ts`：内置 recipes（内嵌 JSON 或 TS 常量）
    - `api.ts`：Tauri command 调用封装
    - `state.ts`：`Context + useReducer` 统一状态与副作用

### 2.2 目录与文件

```
~/.openclaw/openclaw.json                  # 当前配置
~/.openclaw/.clawpal/                       # 本地状态目录
~/.openclaw/.clawpal/history/
~/.openclaw/.clawpal/metadata.json          # 快照元信息（列表可直接读）
```

## 3. Recipe 与配置模型

### 3.1 Recipe 核心结构

```ts
interface Recipe {
  id: string;
  name: string;
  description: string;
  version: string;
  tags: string[];
  difficulty: 'easy' | 'normal' | 'advanced';
  params: RecipeParam[];
  patchTemplate: string; // JSON Merge Patch with {{param}}
  impact: {
    category: 'low' | 'medium' | 'high';
    summary: string;
  };
}

interface RecipeParam {
  id: string;
  label: string;
  type: 'string' | 'number' | 'boolean' | 'textarea';
  required: boolean;
  pattern?: string;
  minLength?: number;
  maxLength?: number;
  placeholder?: string;
}
```

### 3.2 预览与应用结果

```ts
interface PreviewResult {
  recipeId: string;
  diff: string;
  changes: ChangeItem[];
  overwritesExisting: boolean;
  canRollback: boolean; // true = 已生成快照
  impactLevel: 'low' | 'medium' | 'high';
  warnings: string[];
}

interface ChangeItem {
  path: string;
  op: 'add' | 'replace' | 'remove';
  risk: 'low' | 'medium' | 'high';
  reason?: string;
}
```

## 4. Tauri Command 设计

### 4.1 命令边界

- `get_system_status(): SystemStatus`
- `list_recipes(): Recipe[]`
- `preview_apply(recipe_id: string, params: Record<string, string>): PreviewResult`
- `apply_recipe(recipe_id: string, params: Record<string, string>): ApplyResult`
- `list_history(limit: number, offset: number): HistoryPage`
- `preview_rollback(snapshot_id: string): PreviewResult`
- `rollback(snapshot_id: string): ApplyResult`
- `run_doctor(): DoctorReport`
- `fix_issues(issue_ids: string[]): FixResult`
- `open_config_path(): string`

### 4.2 关键返回结构

```ts
interface ApplyResult {
  ok: boolean;
  snapshotId?: string;   // 回滚锚点
  configPath: string;
  backupPath?: string;
  warnings: string[];
  errors?: string[];
}

interface DoctorReport {
  ok: boolean;
  issues: DoctorIssue[];
  score: number; // 0-100 健康值
}

interface DoctorIssue {
  id: string;
  code: string;
  severity: 'error' | 'warn' | 'info';
  message: string;
  autoFixable: boolean;
  fixHint?: string;
}
```

## 5. 核心逻辑算法

### 5.1 参数校验

1. Recipe 找不到 -> 错误
2. 参数逐个校验：
   - 必填
   - 类型
   - pattern（正则）
   - 长度
3. 校验通过后进入渲染

### 5.2 参数渲染与差异生成

1. 用 `{{param_id}}` 进行文本替换
2. 解析 base 配置与 patch 模板（json5）
3. 进行深合并（merge patch）
4. Diff 生成：
   - 将当前配置与待应用配置序列化为 pretty JSON
   - 输出 unified diff 或关键节点差异

### 5.3 应用流程（Write path）

1. 调用 preview
2. 创建快照：
   - 将当前 `openclaw.json` 复制到 `.clawpal/history/<ts>_<slug>.json`
   - 更新 `metadata.json`（按时间倒序）
3. 原子写新配置：
   - 临时文件写入 -> rename 覆盖
4. 失败回滚：
   - 临时文件清理
   - 保留快照，但不上报成功

### 5.4 回滚流程

1. 读取目标快照
2. 计算与当前配置 diff
3. 确认后再执行快照（当前入历史）
4. 用目标快照替换当前配置

### 5.5 Doctor 与修复

- 语法错误：尝试修复尾逗号、未闭合引号等常见问题
- 关键字段缺失：按最小安全默认值补齐（仅在用户确认后）
- 端口占用：读取端口字段并做最小冲突提示（非阻塞 warning）
- 权限问题：展示“文件不可读/不可写+路径来源”建议

## 6. 安全与约束（MVP 简化）

- 禁止修改路径初版：
  - `gateway.auth.*`
  - `*.token`
  - `*.apiKey`
- `dangerous` 字段提示：
  - `gateway.port` 修改前增加二次确认
- Deep Link 与远程 Recipe 不支持（MVP 不接入官网）
- 所有写操作必须先写快照，`apply_result.snapshotId` 作为审计锚点

## 7. 兼容与平台策略

- 路径优先级：
  1. `~/.openclaw/.clawpal`（首次初始化）
  2. 现有配置路径检测（若 `.openclaw` 已存在则复用）
  3. WSL2 仅作“只读检测入口”（MVP 只读展示，不做深度自动映射）
- Windows/macOS/Linux 使用 Tauri 提供的跨平台路径 API

## 8. 内置 Recipe（MVP）

1. Discord 频道专属人设
2. Telegram 群组配置
3. 定时任务配置
4. 模型切换
5. 性能优化

每个 Recipe 均含：
- id/name/version/params/patchTemplate
- default 示例值与最小校验规则
- impact 元信息（影响等级）

## 9. 里程碑与验收

### Milestone A（第 1 阶段）
- 项目脚手架搭建
- Recipe 解析与参数校验
- 安装向导基础路径可跑通
- 最小 Diff 预览
- 验收：选 1 个 Recipe 完整完成 “填参数->预览->应用”

### Milestone B（第 2 阶段）
- 快照元数据与历史列表
- 回滚预览与回滚执行
- 验收：应用后 1 步回滚成功且 current 状态可恢复

### Milestone C（第 3 阶段）
- Doctor 基础扫描与修复入口
- 端口/权限/语法检查
- 验收：制造 1 处语法错误，修复后能读取启动

## 10. 关键风险与回退方案

- JSON5 风格保持未完全解决：MVP 采用“标准化写回”，后续引入更精细 AST 修改
- 端口占用检测可能误报：提示“仅警告”不阻断
- Recipe 深入语义冲突：通过 change list 显示冲突路径并要求确认
- 若配置文件损坏到不可修复：保留备份、提示手工恢复和重建路径

## 11. 交付标准

- 所有命令返回可序列化错误码，不出现裸异常弹窗
- 每次 apply 成功都生成可回滚快照（除极端写入错误）
- 历史列表支持 `时间/来源/描述` 查看
- Doctor 能给出至少一条可执行修复动作
- 无法修复时给出建议与重试按钮
