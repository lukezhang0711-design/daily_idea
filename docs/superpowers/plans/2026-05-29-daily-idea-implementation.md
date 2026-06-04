# 小想法（daily_idea）实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 做一个单人用的 mac 桌面应用「小想法」——常驻桌面的进度组件 + 工作区（计划/待办/想法/AI 洞察）+ 全局随手记 + 厂商无关的 AI 回顾。

**Architecture:** 一个 Tauri v2 应用，三个窗口（工作区主窗口 / 桌面悬浮组件 / 随手记小窗）。前端 React + TypeScript + Vite；数据用本地 SQLite（tauri-plugin-sql + migrations）；AI 调用放在 Rust 端（reqwest），key 存 mac 钥匙串（keyring crate），适配层厂商无关（OpenAI 兼容 + Anthropic）。纯本地、无服务器、无云同步。

**Tech Stack:** Tauri v2 (Rust), React 18 + TypeScript + Vite, tauri-plugin-sql (SQLite), tauri-plugin-global-shortcut, tauri-plugin-notification, tauri-plugin-autostart, `keyring` crate, `reqwest`, `tokio`. 测试：Rust 用 `cargo test`（数据/AI/业务逻辑纯函数），前端用 `vitest`（纯逻辑），UI 与原生窗口行为用手动验证清单。

**配套规格（必读）：**
- 产品方案：`docs/superpowers/specs/2026-05-29-daily-idea-product-design.md`（v3，权威）
- 技术方案：`docs/superpowers/specs/2026-05-29-daily-idea-tech-design.md`（v1）

---

## 阶段总览（路线图）

每个阶段交付一份"能跑、能测"的软件，按顺序做。

| 阶段 | 交付什么（做完能干嘛） | 为什么这个顺序 |
|---|---|---|
| **P1 骨架 + 桌面窗验证 + 第一个闭环** | 应用能启动；桌面悬浮组件能显示；全局快捷键记一条想法→桌面立刻看到 | 先把**最大未知（桌面悬浮窗）**验证掉，并立起数据底座，拿到一个真能用的纵切片 |
| **P2 数据层完整 + 计划/待办 CRUD** | 工作区能建/改/删计划与待办、设日期优先级、改状态、进度条、回收站 | 核心数据与业务逻辑（进度、软删除、移动）是后面一切的基础 |
| **P3 工作区界面成型** | 列表→详情全套、零散待办、搜索、标签 | 把数据层包成完整可用的工作区 |
| **P4 AI 层（厂商无关）** | 设置里配任意模型 key；想法转计划/待办（AI 起草+确认）；想法收集箱完善 | AI 依赖前面的数据；先做"按需触发"的 AI（转化），再做定时 |
| **P5 AI 洞察 + 对话 + 调度通知** | 定时/手动生成洞察卡片、深挖对话、结论回写；到期提醒、免打扰、午夜刷新、开机自启 | 定时与通知依赖完整数据与 AI |
| **P6 收尾** | 数据导出、首次引导、权限检测降级、空状态 | 打磨与兜底 |

**本文件把 P1 写到可直接执行的最细粒度；P2–P6 给出可执行的任务清单（文件/职责/接口/测试/验收）。** 每个阶段开工前，按需把该阶段展开成 P1 那样的细步骤。

---

## 文件结构（总体）

应用建在仓库根目录（与 `docs/` 同级）。

```
daily_idea/
├─ src/                         # 前端（React + TS）
│  ├─ main.tsx                  # 入口，按 URL/参数决定渲染哪个窗口
│  ├─ windows/
│  │  ├─ Workspace.tsx          # 工作区主窗口根组件
│  │  ├─ DesktopWidget.tsx      # 桌面悬浮组件根组件
│  │  └─ QuickCapture.tsx       # 随手记小窗根组件
│  ├─ features/
│  │  ├─ plans/                 # 计划：列表/详情/逻辑
│  │  ├─ todos/                 # 待办
│  │  ├─ ideas/                 # 想法收集箱
│  │  ├─ insights/              # AI 洞察 + 对话
│  │  ├─ search/                # 搜索
│  │  └─ settings/              # 设置
│  ├─ data/
│  │  ├─ db.ts                  # 前端调用数据命令的封装
│  │  └─ types.ts               # 共享 TS 类型（与 Rust 对齐）
│  └─ lib/                      # 纯逻辑（progress/sort/dueSoon 等，可单测）
├─ src-tauri/                   # Rust 后端
│  ├─ src/
│  │  ├─ main.rs                # Tauri 启动、注册命令、建窗口
│  │  ├─ db/                    # SQLite 连接、migrations、CRUD 命令
│  │  ├─ ai/                    # AI 适配层（trait + openai_compat + anthropic）
│  │  ├─ keychain.rs            # key 读写（keyring crate）
│  │  ├─ scheduler.rs           # 定时任务（洞察、到期、午夜刷新）
│  │  └─ windows.rs             # 桌面悬浮窗/随手记窗的创建与原生属性
│  ├─ migrations/               # SQL 迁移文件
│  ├─ Cargo.toml
│  └─ tauri.conf.json
└─ package.json
```

**边界原则**：每个 feature 文件夹自带它的 UI + 该 feature 的纯逻辑；跨 feature 的纯函数（排序、进度、到期判断）放 `src/lib/` 并单测；所有写数据库的动作只走 `src/data/db.ts` → Rust 命令，UI 不直接拼 SQL。

---

## P1 · 骨架 + 桌面窗验证 + 第一个闭环

**目标**：应用能启动；桌面悬浮组件常显、不抢焦点；按全局快捷键记一条想法，桌面组件立刻多一条。
**这一阶段同时是"桌面悬浮窗"这个最大未知的技术验证（spike）。**

### Task 1: 初始化 Tauri + React 工程

**Files:**
- Create: 整个 `daily_idea/` 脚手架（由命令生成）
- Modify: `src-tauri/tauri.conf.json`

- [ ] **Step 1: 用官方脚手架创建工程**

Run（在仓库根目录）:
```bash
npm create tauri-app@latest daily_idea -- --template react-ts --manager npm
cd daily_idea && npm install
```
Expected: 生成 `daily_idea/`，含 `src/`（React+TS）与 `src-tauri/`（Rust）。

- [ ] **Step 2: 跑通空壳**

Run:
```bash
cd daily_idea && npm run tauri dev
```
Expected: 弹出一个默认窗口，能看到模板页面。关掉。

- [ ] **Step 3: Commit**

```bash
git add daily_idea
git commit -m "chore: 初始化 Tauri + React 工程脚手架"
```

### Task 2: 桌面悬浮组件窗口验证（spike，最关键）

**目标**：建一个**透明、无边框、不抢焦点（点它不夺走当前 app 焦点）、跳过程序坞、常显在桌面、可拖动、内容可点击**的窗口。先用 Tauri 配置尝试；不行则下沉到原生 `NSPanel`。

**Files:**
- Modify: `src-tauri/tauri.conf.json`（声明第二个窗口）
- Create: `src-tauri/src/windows.rs`
- Create: `src/windows/DesktopWidget.tsx`

- [ ] **Step 1: 在配置里声明桌面组件窗口**

在 `tauri.conf.json` 的 `app.windows` 里加一个窗口（`label: "widget"`，`url: "index.html?window=widget"`，`transparent: true`，`decorations: false`，`alwaysOnTop: false`，`skipTaskbar: true`，`focus: false`，`width: 320`，`height: 480`，定位右上角）。

- [ ] **Step 2: 前端按参数渲染对应窗口**

`src/main.tsx` 读 `new URLSearchParams(location.search).get("window")`，为 `"widget"` 渲染 `<DesktopWidget/>`、`"capture"` 渲染 `<QuickCapture/>`、否则 `<Workspace/>`。`DesktopWidget.tsx` 先放一个写死的占位面板（半透明卡片 + 一行"占位"）。

- [ ] **Step 3: 验证悬浮窗行为（手动验证清单）**

Run: `npm run tauri dev`
逐项确认并记录结果（✅/❌）：
1. 窗口透明、无边框、显示在桌面右上角。
2. 点击窗口里的按钮，**当前正在用的其他 app 不丢焦点**（不会跳到本应用）。
3. 不出现在程序坞/任务切换里（skipTaskbar 生效）。
4. 切换 Space/全屏/Stage Manager 时表现可接受（至少不消失、不挡事）。
5. 能拖动定位。

- [ ] **Step 4: 若 Tauri 配置不达标，下沉原生 NSPanel**

若 Step 3 第 2 或 4 项不达标：在 `windows.rs` 用 `objc2`/`cocoa` 拿到该窗口的 `NSWindow`，设置为 `NSPanel` 风格、`collectionBehavior` 含 `.canJoinAllSpaces | .stationary`、`level` 设为 desktop 级、`styleMask` 加 `.nonactivatingPanel`。在 `main.rs` 的 `setup` 钩子里对 `label=="widget"` 的窗口应用。

代码骨架（`windows.rs`）:
```rust
#[cfg(target_os = "macos")]
pub fn make_desktop_panel(window: &tauri::WebviewWindow) {
    use objc2::msg_send;
    // 取 ns_window：window.ns_window()，转 NSWindow*；
    // 设 level、collectionBehavior(.canJoinAllSpaces|.stationary)、
    // styleMask 增加 NSWindowStyleMaskNonactivatingPanel、ignoresMouseEvents=false。
    // 具体指针操作按 objc2 当前 API 写。
}
```
然后重跑 Step 3，直到 1–5 全 ✅。

- [ ] **Step 5: 记录 spike 结论**

在本文件末尾"## Spike 记录"追加：最终用的是「Tauri 配置」还是「原生 NSPanel」、哪些行为达标/妥协。

- [ ] **Step 6: Commit**

```bash
git add daily_idea
git commit -m "feat: 桌面悬浮组件窗口（透明/不抢焦点/常显）验证通过"
```

### Task 3: 数据底座（SQLite + 想法表 + 写入/读取命令）

**Files:**
- Modify: `src-tauri/Cargo.toml`、`src-tauri/src/main.rs`
- Create: `src-tauri/migrations/0001_init.sql`
- Create: `src-tauri/src/db/mod.rs`
- Create: `src/data/types.ts`、`src/data/db.ts`

- [ ] **Step 1: 加 sql 插件 + 初始迁移**

`Cargo.toml` 加 `tauri-plugin-sql = { version = "2", features = ["sqlite"] }`。`main.rs` 注册插件并指向 migrations。`0001_init.sql` 先建 `ideas` 表：
```sql
CREATE TABLE ideas (
  id TEXT PRIMARY KEY,
  text TEXT NOT NULL,
  note TEXT DEFAULT '',
  status TEXT NOT NULL DEFAULT 'inbox',     -- inbox | converted | archived
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  deleted_at TEXT                            -- 非空=在回收站
);
```

- [ ] **Step 2: 写失败测试（纯逻辑：新建想法的字段默认值）**

`src/lib/idea.ts` 计划导出 `newIdea(text: string, now: string): Idea`。在 `src/lib/idea.test.ts`:
```ts
import { describe, it, expect } from "vitest";
import { newIdea } from "./idea";
it("新建想法默认进 inbox、无删除时间", () => {
  const i = newIdea("  喝水  ", "2026-05-29T10:00:00Z");
  expect(i.text).toBe("喝水");        // 去空白
  expect(i.status).toBe("inbox");
  expect(i.deletedAt).toBeNull();
  expect(i.id).toMatch(/.+/);
});
```

- [ ] **Step 3: 跑测试确认失败**

Run: `cd daily_idea && npx vitest run src/lib/idea.test.ts`
Expected: FAIL（`newIdea` 未定义）。

- [ ] **Step 4: 实现 `newIdea`**

`src/lib/idea.ts`:
```ts
import { Idea } from "../data/types";
export function newIdea(text: string, now: string): Idea {
  return { id: crypto.randomUUID(), text: text.trim(), note: "",
           status: "inbox", createdAt: now, updatedAt: now, deletedAt: null };
}
```
`src/data/types.ts` 定义 `Idea` 类型（字段同上）。

- [ ] **Step 5: 跑测试确认通过**

Run: `npx vitest run src/lib/idea.test.ts` → Expected: PASS。

- [ ] **Step 6: 前端数据命令封装**

`src/data/db.ts` 导出 `addIdea(i: Idea)`、`listIdeas(): Promise<Idea[]>`（只取 `deleted_at IS NULL`，按 `created_at` 倒序），内部用 tauri-plugin-sql 执行 SQL。

- [ ] **Step 7: Commit**

```bash
git add daily_idea
git commit -m "feat: 数据底座（SQLite + ideas 表 + 新建/读取想法）"
```

### Task 4: 全局快捷键 → 随手记小窗 → 写库

**Files:**
- Modify: `src-tauri/Cargo.toml`、`src-tauri/src/main.rs`、`tauri.conf.json`
- Create: `src/windows/QuickCapture.tsx`

- [ ] **Step 1: 注册全局快捷键**

`Cargo.toml` 加 `tauri-plugin-global-shortcut = "2"`。`main.rs` 注册默认 `CmdOrCtrl+Shift+Space`，按下时显示/聚焦 `capture` 窗口（无则建）。`capture` 窗口：无边框、居中、`width:520,height:120`、`transparent:true`。

- [ ] **Step 2: 随手记界面**

`QuickCapture.tsx`：一个输入框；回车 → 调 `addIdea(newIdea(value, new Date().toISOString()))` → 发一个 Tauri 事件 `idea-added` → 隐藏窗口并清空；Esc → 隐藏（**不清空，保留草稿**：草稿存 `localStorage`，下次打开回填）。失焦不关闭、不清空。

- [ ] **Step 3: 手动验证闭环**

Run: `npm run tauri dev`，在别的 app 里按 `⌘⇧Space` → 弹框 → 打字回车 → 确认 `ideas` 表新增一行（可临时在工作区窗口打印 `listIdeas()`）。

- [ ] **Step 4: Commit**

```bash
git add daily_idea
git commit -m "feat: 全局快捷键随手记 → 写入想法（失焦保留草稿）"
```

### Task 5: 桌面组件显示"最近想法" + 即时刷新

**Files:**
- Modify: `src/windows/DesktopWidget.tsx`、`src-tauri/src/main.rs`

- [ ] **Step 1: 组件读取并显示最近 3 条想法**

`DesktopWidget.tsx` 挂载时 `listIdeas()` 取前 3 条渲染（半透明卡片，标题"最近想法"）。

- [ ] **Step 2: 监听新增事件即时刷新**

监听 Tauri 事件 `idea-added`（由随手记发出，需在 Rust 端广播到所有窗口或用 `emit_to("widget", ...)`）→ 重新 `listIdeas()` 重画。

- [ ] **Step 3: 手动验证"记想法→桌面立刻看到"**

Run: `npm run tauri dev`；按快捷键记一条 → 桌面组件**立刻**多一条。这就是 P1 的纵切片闭环。

- [ ] **Step 4: Commit**

```bash
git add daily_idea
git commit -m "feat: 桌面组件显示最近想法并即时刷新（P1 闭环打通）"
```

**P1 验收**：① 应用启动出现桌面悬浮组件；② 任意 app 里 `⌘⇧Space` 记想法、桌面立刻刷新；③ spike 结论已记录；④ 全部 commit。

---

## P2 · 数据层完整 + 计划/待办 CRUD

**Files（新增/修改）：** `src-tauri/migrations/0002_plans_todos.sql`、`src-tauri/src/db/`、`src/data/types.ts`、`src/data/db.ts`、`src/lib/progress.ts`、`src/lib/sort.ts`、`src/lib/dueSoon.ts`、`src/features/plans/`、`src/features/todos/`

**建表（0002）：** 见技术方案第 3 节与产品方案第 5 节。关键列：
```sql
CREATE TABLE plans ( id TEXT PRIMARY KEY, name TEXT NOT NULL, priority TEXT NOT NULL DEFAULT 'med',
  due_date TEXT, note TEXT DEFAULT '', status TEXT NOT NULL DEFAULT 'active',  -- active|done|paused|archived
  created_at TEXT, updated_at TEXT, deleted_at TEXT );
CREATE TABLE todos ( id TEXT PRIMARY KEY, name TEXT NOT NULL, plan_id TEXT,    -- 空=零散
  status TEXT NOT NULL DEFAULT 'todo', -- todo|doing|done
  due_date TEXT, snooze_until TEXT, note TEXT DEFAULT '',
  created_at TEXT, updated_at TEXT, deleted_at TEXT );
```

**必做的纯逻辑 + 单测（TDD，每个都先写失败测试）：**
- `progress.ts` → `计算进度(todos)`：完成数 / 未删除总数；空（0 条）返回 `null`（界面据此显示"还没拆任务"、不画进度条）。测：3 条 1 完成 → 33%；0 条 → null；删除的不计分母。
- `sort.ts` → `统一排序(items)`：已过期 > 今天 > 3 天内 > 其余；再按优先级 high>med>low（零散待办按 med）；同档按截止早；再按 updated_at。测：构造混合用例断言顺序。
- `dueSoon.ts` → `是否快到期(due, now)`（3 天内）、`是否过期(due, now)`；按本机时区。测：今天、+2 天、+4 天、昨天。

**CRUD 命令（`db/`）：** plans/todos 的建、改、改状态、软删除（写 `deleted_at`）、恢复、移动 todo（改 `plan_id`，含设 null=移出）。删计划：默认连带把其 todos 也写 `deleted_at`；提供参数 `keep_todos: bool`，为 true 时只把 todos 的 `plan_id` 置 null（变零散）。

**界面：** 计划列表（带进度条，调 `计算进度`）、计划详情（改字段、加 todo、改状态、删除时弹确认含"保留 todo"选项）、todo 行勾选（三态切换）。

**P2 验收：** 能建/改/删计划与待办、设日期优先级、勾选改进度；删计划走回收站且可选保留 todo；`progress/sort/dueSoon` 单测全绿。

---

## P3 · 工作区界面成型（详情层级 + 零散待办 + 搜索 + 标签）

**Files：** `0003_tags.sql`、`src/features/search/`、`src/features/todos/Loose.tsx`、`src/features/*/Detail.tsx`、标签相关 `db.ts`

**建表（0003）：** `tags(id,name UNIQUE,created_at)`、`item_tags(item_type,item_id,tag_id)`。

**任务：**
- 工作区四栏（计划/收集箱/AI 洞察占位/设置占位）+ 顶栏"＋随手记"按钮 + 搜索框。
- "列表 → 点名字 → 详情"路由；计划详情里 todo 可点进 todo 详情；todo 详情可"移动到计划/移出"。
- 计划页加独立"零散待办"区（`plan_id IS NULL`）。
- 搜索：范围 = 标题+正文+备注+补充；含已归档（结果标注）；不含回收站；可按标签/状态筛选。纯过滤逻辑放 `src/lib/search.ts` 并单测。
- 标签：增/改名/合并/删（删标签只删 `item_tags` 关联，不删内容）。
- **桌面组件补全三区**：计划进度（归档/已完成不显示）+ 今日/快到期待办（过期标红置顶、`status='doing'` 高亮）+ 最近想法；每区 ≤3 条、多了折叠"还有 N 个 →"；今日待办支持**打勾完成 / 推迟**（推迟选 明天/本周末/下周/自定，写 `snooze_until`）。
- **深链**：桌面/列表点名字 = 打开工作区窗口并定位到该详情；关窗即回桌面。

**P3 验收：** 桌面/列表点名字进详情；零散待办有家；关键词+标签能搜到（含归档、不含回收站）；标签可改名/合并/删；桌面三区齐全、可打勾/推迟、过期标红、进行中高亮。

---

## P4 · AI 层（厂商无关）+ 想法转化

**Files：** `src-tauri/src/ai/{mod.rs,openai_compat.rs,anthropic.rs}`、`src-tauri/src/keychain.rs`、`src/features/settings/Ai.tsx`、`src/features/ideas/Convert.tsx`

**AI 适配 trait（Rust）：**
```rust
pub struct ChatMsg { pub role: String, pub content: String }
pub struct ChatOpts { pub max_tokens: u32, pub timeout_s: u64, pub json: bool }
#[async_trait::async_trait]
pub trait AiProvider {
    async fn chat(&self, msgs: Vec<ChatMsg>, opts: ChatOpts) -> Result<String, AiError>;
}
pub struct OpenAiCompat { pub base_url: String, pub model: String, pub api_key: String }
pub struct Anthropic   { pub base_url: String, pub model: String, pub api_key: String }
```
- 两个实现：`openai_compat`（POST `{base_url}/chat/completions`）、`anthropic`（POST `{base_url}/v1/messages`）。
- **失败处理（精简，按 codex 筛后的结论）**：一个超时（默认 30s）+ 失败重试 1 次 + 校验返回是否为预期结构；都不行则返回 `AiError`，上层显示"稍后再试"。**不做**每模型能力矩阵。
- **Key 安全**：`keychain.rs` 用 `keyring` crate 读写（service=`daily_idea`，account=配置 id）。配置（名称/base_url/model/active）存 settings 表；key 只进钥匙串。
- 设置界面：可添加多组"AI 配置"，选一个为当前。

**想法转化（按需 AI）：**
- 想法详情两个按钮："转成计划"、"转成一条 todo"。
- "转成计划"：调 AI 出结构化草稿（计划名 + todo 列表），前端展示**待确认**草稿（可改）→ 确认才写库；写库时记一条 `conversion_ops`（idea_id、生成的 target 类型/ id、`undo_deadline`）。
- **撤销（有边界，按 codex 采纳项）**：建 `conversion_ops` 表 + `ai_actions`。刚转完显示"撤销"；撤销前检查生成对象是否被改过（比对 updated_at/字段），若改过则提示"这些后续改动会一并撤销"再确认；超 `undo_deadline` 撤销入口消失。转化后想法置 `status='converted'` 且记 `converted_target_*`；断链规则见产品方案第 10 节。

**P4 验收：** 设置里填任意模型 key 能用；想法能转计划/待办（AI 草稿 + 我确认）并可撤销；撤销遇到后续改动会提示；key 不落明文库。

---

## P5 · AI 洞察 + 对话 + 调度通知

**Files：** `0004_insights.sql`、`src-tauri/src/scheduler.rs`、`src/features/insights/`、autostart/notification 插件接入

**建表（0004）：** `insights(id,type,content,status,created_at,archived_at)`、`conversations(id,insight_id,created_at)`、`messages(id,conversation_id,role,content,created_at)`。

**任务：**
- **洞察生成**：构造上下文（手动唤起时前端先问"扫全部/最近一周"；定时默认扫全部历史想法 + 进行中计划）→ 调 AI（要求结构化输出：若干 `{type,content}`）→ 写 `insights`（unread）。**上下文策略（采纳项，精简）**：不原样全塞——超量时对较旧内容做摘要后再投喂；给硬上限。
- **去重（少而精）**：新洞察与"未处理的旧洞察"比对，过近似的不出；无可串时返回空并提示。
- **卡片**：列表带状态（未读/已读/已采纳/已忽略）；"深挖"建 `conversation` 转对话；"采纳"复用 P4 转化流程；"没用"→ `dismissed` 并作为信号。旧的未动的超期 `archived_at` 归档，主列表只显示新鲜的。
- **对话**：messages 持久化，挂在来源卡片下；结论**经确认**回写到对应想法（沿用"AI 改数据先确认"）。
- **调度（codex 采纳项）**：`scheduler.rs` 用 tokio 定时；**明确"app 未运行"行为**：接 `tauri-plugin-autostart` 开机自启 + 常驻（菜单栏托盘），用户真退出则不提醒（可接受，写进设置说明）。到期检查每天定时跑一次；**午夜重算**"今天/过期"并刷新桌面组件；睡眠唤醒后做一次补算（仅状态重算，不补发错过的洞察）。
- **通知**：`tauri-plugin-notification`；遵守**免打扰时段**（默认 22:00–9:00）；快到期每天**汇总弹一次**。

**P5 验收：** 能定时/手动出洞察（体现串联/挖关联/指方向、去重、空时不硬凑）；卡片可深挖/采纳/标没用、旧的自动归档；对话持久、结论确认后回写；开机自启、午夜刷新、免打扰生效；错过的洞察不补跑。

---

## P6 · 收尾（导出 / 首次引导 / 权限降级 / 空状态）

**Files：** `src/features/settings/Export.tsx`、`src/features/onboarding/`、各空状态组件

**任务：**
- **数据导出（codex 采纳项）**：一键导出**一份完整、人能读的 JSON**，包含 plans/todos/ideas/insights/tags/conversations/messages + 删除状态 + schema 版本号（便于将来手动恢复）。**不做应用内导入/恢复**。
- **首次引导**：检测并引导授予权限（全局快捷键的辅助功能、通知）；跑通"记一条想法→桌面看到"。
- **权限降级**：启动检测快捷键/通知权限状态；快捷键冲突或未授权时，设置页给重新授权入口，并提示工具内"＋随手记"是兜底。
- **空状态**：无计划/想法/洞察时给友好引导文案；AI 失败"稍后再试"；内容太少"再多记一些"。
- **想法去重提示**：完全相同文本再记时温和提示，不阻断回车即走。

**P6 验收：** 能导出完整 JSON；首次引导能把人带到第一个闭环；权限缺失有清晰降级；各空状态/失败友好。

---

## 跨阶段约定

- **测试策略**：纯逻辑（progress/sort/dueSoon/search/转化校验/洞察去重）一律 TDD 先写失败测试（vitest / cargo test）；UI 与原生窗口行为用各阶段验收里的手动清单。
- **提交**：每个 Task 末尾 commit；小步频繁。
- **DRY/YAGNI**：所有写库走 `data/db.ts`→Rust 命令；不提前做"明确不做"清单里的东西（产品方案第 13 节、技术方案第 8 节）。
- **数据可靠（codex 采纳的便宜项）**：SQLite 开 WAL；应用启动时若检测到库损坏，提示并保留旧文件副本；导出文件自带 schema 版本号。
- **回收站清理**：软删除（`deleted_at` 非空）满约 30 天的记录，由应用启动时的清理任务真删。

---

## Spike 记录

（P1 Task 2 完成后在此追加：桌面悬浮窗最终用 Tauri 配置还是原生 NSPanel、各行为达标/妥协情况、对后续阶段的影响。）
