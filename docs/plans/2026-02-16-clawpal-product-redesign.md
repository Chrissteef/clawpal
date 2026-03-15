# ClawPal 产品精简 & 重新定位

> 从"全功能配置管理后台"回归"AI 配置助手"

## 1. 问题

v0.2 新增了 Models、Channels、Data 三个管理页面后，产品从"场景驱动的配置助手"滑向了"OpenClaw 全功能管理后台"。功能杂糅，偏离了核心用户（新手）的需求。

## 2. 新产品定位

**ClawPal = AI 配置助手 + 精选 Recipe 库**

- 核心用户：不想碰 JSON 的新手
- 第一入口：Chat — 用自然语言描述需求，LLM 生成配置方案
- 第二入口：Recipe — 社区精选的常见场景，一键安装
- 安全保证：所有变更（Chat 或 Recipe）走同一安全链路：快照 → 预览 diff → 用户确认 → 应用

## 3. 精简前后对比

| 精简前 (7 页面) | 精简后 | 处理方式 |
|---|---|---|
| Home (状态仪表盘) | Home (状态 + Agents + 推荐 + Chat) | 重构 |
| Recipes | Recipes | 保留 |
| Install | Install (从 Recipe/Chat 进入) | 保留 |
| History | History | 保留 |
| Doctor | Doctor (+ 数据清理) | 扩展 |
| Models | Settings > Model Profiles | 降级，移除绑定功能 |
| Channels | 删除 | 由 Chat/Recipe 覆盖 |
| Data | 合并入 Doctor | 删除独立页面 |

## 4. 导航结构

左侧边栏，4 个主入口 + 1 个设置：

```
┌──────────┬──────────────────────────────────┐
│          │                                  │
│  Home    │                                  │
│          │                                  │
│  Recipes │         页面内容                  │
│          │                                  │
│  History │                                  │
│          │                                  │
│  Doctor  │                                  │
│          │                                  │
│  ─────── │                                  │
│  Settings│                                  │
│          │                                  │
└──────────┴──────────────────────────────────┘
```

Install 不在侧边栏出现，从 Recipe 卡片或 Chat 触发后进入。

## 5. Home 页面

左右分栏布局：左侧主内容区，右侧常驻 Chat 窗口。

### 5.1 左侧主区域（四段式）

**状态摘要** — 一行紧凑卡片：
- 配置健康状态（✅/❌）
- OpenClaw 版本 + 是否有更新
- 当前默认模型

**Agents 概览** — 每个 Agent 一行：
- Agent 名称 / ID
- 当前使用的模型
- 关联的 Channels（Discord#xxx、Telegram 等）
- 在线状态

**推荐 Recipes** — 3-4 张卡片，点击进入安装向导

**最近操作** — 最近 3-5 条历史记录

### 5.2 右侧 Chat 窗口

常驻面板，使用用户在 Settings 中配置的 Model Profile 调用 LLM。

**LLM 可调用的工具集：**

| 工具 | 作用 | 安全级别 |
|------|------|----------|
| `read_config` | 读取当前配置 | 无风险（只读） |
| `list_agents` | 列出 Agent 信息 | 无风险 |
| `list_recipes` | 搜索/推荐 Recipe | 无风险 |
| `preview_change` | 生成配置补丁并展示 diff | 无风险（不写入） |
| `apply_change` | 应用配置变更 | 需用户点确认 |
| `run_doctor` | 运行诊断 | 无风险 |
| `generate_recipe` | 根据描述生成新 Recipe | 无风险（只生成定义） |

**交互流程：**
```
用户: "我想让 agent-2 在 Telegram 群里只回复被 @ 的消息"
  ↓
LLM: 调用 read_config 了解当前配置
  ↓
LLM: 生成配置补丁，调用 preview_change
  ↓
Chat 内展示 diff 面板 + [确认应用] [取消] 按钮
  ↓
用户点确认 → 自动快照 → 写入配置
```

**关键约束：**
- 所有写操作必须先展示 diff，用户手动确认
- 使用用户在 Settings 中选定的 Chat 模型
- 未配置 Model Profile 时，Chat 区域提示引导去 Settings

### 5.3 布局示意

```
┌──────────┬──────────────────────────┬──────────────────┐
│          │                          │                  │
│  Sidebar │  状态摘要 (一行卡片)       │   Chat 窗口      │
│          │  ┌────┐ ┌────┐ ┌────┐   │                  │
│          │  │健康 │ │版本 │ │模型 │   │  💬 你想实现      │
│          │  │ ✅  │ │2.13│ │gpt4│   │   什么配置？       │
│          │  └────┘ └────┘ └────┘   │                  │
│          │                          │  ┌────────────┐  │
│          │  Agents                   │  │ 输入框      │  │
│          │  ┌──────────────────────┐ │  └────────────┘  │
│          │  │ agent-1    gpt-4o   │ │                  │
│          │  │ ✅ online  Discord#1│ │                  │
│          │  ├──────────────────────┤ │                  │
│          │  │ agent-2    claude   │ │                  │
│          │  │ ✅ online  Telegram │ │                  │
│          │  ├──────────────────────┤ │                  │
│          │  │ agent-3    gpt-4o   │ │                  │
│          │  │ ⚠ no channel        │ │                  │
│          │  └──────────────────────┘ │                  │
│          │                          │                  │
│          │  推荐 Recipes             │                  │
│          │  ┌─────┐ ┌─────┐ ┌─────┐│                  │
│          │  │人设  │ │模型  │ │性能 ││                  │
│          │  └─────┘ └─────┘ └─────┘│                  │
│          │                          │                  │
│          │  最近操作                  │                  │
│          │  • 2/15 应用了"Discord人设"│                  │
│          │  • 2/14 运行了 Doctor     │                  │
│          │                          │                  │
└──────────┴──────────────────────────┴──────────────────┘
```

## 6. Recipes 页面

保持现有设计：卡片式浏览，支持搜索/筛选/标签。

新增的 Recipe 类型（替代被删除的页面功能）：
- "切换默认模型" — 替代 Models 页面的全局绑定
- "为 Agent 设置专属模型" — 替代 Models 页面的 Agent 绑定
- "配置 Discord 频道白名单" — 替代 Channels 页面
- "配置 Telegram mention 规则" — 替代 Channels 页面

## 7. Doctor 页面（扩展）

两个区域：

### 7.1 配置诊断（原有）
- JSON 语法检查
- 必填字段验证
- 端口占用检测
- 文件权限检查
- 一键修复 + 变更原因展示

### 7.2 数据清理（从 Data 合并）
- Memory 文件统计 + 一键清理
- Session 文件统计 + 按 Agent 清理 / 全部清理
- 磁盘占用展示

```
┌─────────────────────────────────────────┐
│  Doctor                                 │
│                                         │
│  配置诊断                    [运行检查]   │
│  ┌─────────────────────────────────┐    │
│  │ ✅ JSON 语法正确                 │    │
│  │ ✅ 必填字段完整                  │    │
│  │ ❌ 端口 8080 被占用              │    │
│  │    → [一键修复] 切换到 8081      │    │
│  └─────────────────────────────────┘    │
│                                         │
│  数据清理                               │
│  ┌─────────────────────────────────┐    │
│  │ Memory: 6 files (2.3 MB)        │    │
│  │                   [清理全部]     │    │
│  │ Sessions: 23 files (15.1 MB)    │    │
│  │   agent-1: 12 files (8.2 MB)   │    │
│  │   agent-2: 11 files (6.9 MB)   │    │
│  │          [按 Agent 清理] [全部]  │    │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
```

## 8. Settings 页面

### 8.1 Model Profiles

首次启动自动从 OpenClaw 配置中提取，无需手动操作。

Profile 卡片展示字段：
- 名称
- 服务商（provider）
- 模型（model）
- API Key（脱敏：`sk-proj-abc...7xZ`）
- 自定义地址（仅设置时显示）

```
┌─────────────────────────────────────┐
│ GPT-4o                     ✅ 启用  │
│ 服务商: openai                      │
│ 模型:   gpt-4o                      │
│ API Key: sk-proj-abc...7xZ          │
│                      [编辑] [删除]  │
├─────────────────────────────────────┤
│ Claude                     ✅ 启用  │
│ 服务商: anthropic                   │
│ 模型:   claude-sonnet-4-5           │
│ API Key: sk-ant-k03...mNp           │
│ 自定义地址: https://my-proxy.com/v1 │
│                      [编辑] [删除]  │
└─────────────────────────────────────┘
│  [+ 新建 Profile]                   │
```

### 8.2 Chat 模型选择

下拉选择 Chat 窗口使用的 Model Profile。

### 8.3 路径配置

OpenClaw 目录和 ClawPal 数据目录的显示与自定义。

## 9. 删除清单

以下代码在实施时需要删除或重构：

| 文件 | 处理 |
|------|------|
| `src/pages/Channels.tsx` | 删除 |
| `src/pages/Data.tsx` | 删除，功能迁移到 Doctor |
| `src/pages/Models.tsx` | 删除，Profile 管理迁移到 Settings |
| `src-tauri/src/commands.rs` | 移除 Channel CRUD、Data 相关命令，保留 Model Profile 命令 |
| `src/App.tsx` | 移除 Channels/Data/Models 路由，改为侧边栏布局 |

## 10. 新增开发项

| 功能 | 优先级 | 说明 |
|------|--------|------|
| 侧边栏布局 | P0 | 替换当前顶部 Tab |
| Home 重构 | P0 | 四段式 + 右侧 Chat |
| Chat 窗口 | P0 | LLM 工具调用 + diff 确认流程 |
| Settings 页面 | P0 | Model Profile 管理 + Chat 模型选择 |
| Doctor 扩展 | P1 | 合并数据清理功能 |
| 新 Recipes | P1 | 模型切换、频道配置等替代 Recipe |
| Agents 概览 API | P1 | 首页 Agent 列表的后端支持 |
| 首次启动自动提取 | P2 | 自动从配置提取 Model Profiles |

---

*Created: 2026-02-16*
