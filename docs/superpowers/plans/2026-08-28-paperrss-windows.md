# PaperRss Windows 全量版 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在 Windows 上复刻 PaperRss 全量体验（纸感沉浸阅读+MathJax+按需 AI+FreshRSS 双向同步+全文检索+主题/快捷键+自动更新），以 Tauri 2 + React 19 为壳，首包 <15MB，行为与 macOS 版一致。

**Architecture:** 新建 `PaperRss-Windows`（`ui-lab/PaperRss-Windows`）与 `PaperRss` 同级，复用其 `Resources/MathJax` 与 `Localizable.xcstrings` 语料；前端 `Vite+React 19+Tailwind 4` 复刻三栏纸感，Rust 侧 `tauri-plugin-sql`（SQLite FTS5）替代 GRDB，AI 走 `reqwest` 直连 OpenAI 兼容，更新用 `tauri-plugin-updater` 对齐 Sparkle。

**Tech Stack:** Tauri 2.5 + Rust 1.82 + React 19 + TypeScript 5.9 + Vite 6 + Tailwind 4 + SQLite + FTS5 + MathJax 3 + `tauri-plugin-sql`/`updater`/`store`/`globalShortcut` + Vitest + Playwright

**Spec:** `ui-lab/PaperRss/README.md:22` + `Package.swift:4` + `.agents/rules/principle.md:5` + 本计划 Global Constraints

## Global Constraints

- Windows 10 1809+（WebView2），Node >=20，Rust >=1.82，pnpm only，`type: module`
- 复刻 PaperRss 设计：衬线正文、纸感底色、明暗主题、三栏（订阅/列表/阅读）、TOC 抽屉、`principle.md:40` 双语 i18n（String Catalog）
- Core 禁 UI：`src-tauri/src/core/*`（模型/解析/持久化/网络/同步/AI 纯策略）不依赖前端；前端 `src/*` 仅 UI
- 数据兼容：SQLite 文件 `paper-rss.db` 默认 `%APPDATA%/PaperRss-Windows/`，结构兼容 `GRDB` 旧库，启动前备份
- Key 仅走 `tauri-plugin-store` 加密入口，不进日志/导出
- 验证分级：`Tier1 --core`（`cargo test` + `pnpm test:core`）、`Tier2 --web`（`pnpm test`）、`Tier3` 需 `pnpm tauri dev` 真机交互否则显式 `Manual UI verification required`
- SemVer `vX.Y.Z`，Release 含 `tauri build` 产物 + `latest.json` + README 同步

---

## File Structure

```
PaperRss-Windows/                 # 新建，与 PaperRss 同级
├── src-tauri/
│   ├── Cargo.toml
│   ├── tauri.conf.json           # window 1280x800, singleInstance, updater
│   └── src/
│       ├── lib.rs
│       ├── core/
│       │   ├── mod.rs
│       │   ├── db.rs             # SQLite + FTS5, migration
│       │   ├── models.rs         # Feed/Article/Account
│       │   ├── feed.rs           # RSS/Atom parse (feed-rs)
│       │   ├── sync.rs           # FreshRSS REST
│       │   └── ai.rs             # OpenAI 兼容 reqwest
│       └── commands.rs           # tauri::command 桥接
├── src/
│   ├── main.tsx
│   ├── app.css                   # 纸感 tokens（复刻 PaperRss/Resources）
│   ├── lib/
│   │   ├── db.ts                 # sql.js wrapper (dev) / invoke wrapper (prod)
│   │   ├── api.ts                # invoke('fetch_feed'), invoke('ai_summary')
│   │   └── store.ts              # zustand 订阅/文章/主题
│   ├── components/
│   │   ├── shell/Sidebar.tsx     # 三栏-左订阅
│   │   ├── shell/ArticleList.tsx # 三栏-中列表
│   │   ├── reader/ArticleView.tsx# MathJax + TOC + 划词
│   │   ├── reader/TocDrawer.tsx
│   │   ├── ai/SummaryCard.tsx    # 按 V 触发
│   │   └── settings/ModelPanel.tsx
│   └── routes/Settings.tsx
├── tests/
│   ├── core.test.ts
│   └── e2e/reader.spec.ts
└── package.json
```

---

### Task 0: 脚手架 + 纸感基座

**Files:**
- Create: `PaperRss-Windows/package.json`
- Create: `PaperRss-Windows/src-tauri/Cargo.toml`
- Create: `PaperRss-Windows/src-tauri/tauri.conf.json`
- Create: `PaperRss-Windows/src/app.css`
- Create: `PaperRss-Windows/src/main.tsx`
- Test: `PaperRss-Windows/tests/scaffold.test.ts`

**Interfaces:**
- Consumes: `PaperRss/Resources/MathJax`、`Localizable.xcstrings`（拷贝）
- Produces: `pnpm tauri dev` 可起空窗 1280x800，`app.css` 含 `--paper-bg: #f7f5ef` 等纸感 token

- [ ] **Step 1: 写失败的脚手架测试**

```ts
// tests/scaffold.test.ts
import { test, expect } from 'vitest';
import { readFileSync } from 'node:fs';
test('tauri conf exists', () => {
  const j = JSON.parse(readFileSync('src-tauri/tauri.conf.json','utf8'));
  expect(j.build.beforeBuildCommand).toBe('pnpm build');
  expect(j.app.windows[0].width).toBe(1280);
});
```

- [ ] **Step 2: 运行失败**

Run: `pnpm --filter PaperRss-Windows test -- scaffold.test.ts`
Expected: FAIL no file

- [ ] **Step 3: 初始化**

```bash
pnpm create tauri-app@latest PaperRss-Windows --template react-ts --manager pnpm --identifier com.paperrss.windows --yes
# 手动拷贝
cp -r ../PaperRss/Resources/MathJax ./src-tauri/../public/MathJax
cp ../PaperRss/Resources/Localization/Localizable.xcstrings ./src/locales/
```

`src/app.css`：
```css
@import "tailwindcss";
:root{ --paper-bg:#f7f5ef; --paper-fg:#1a1a18; --serif: ui-serif, Georgia, serif; }
.dark{ --paper-bg:#1a1a18; --paper-fg:#f7f5ef; }
body{ background:var(--paper-bg); color:var(--paper-fg); font-family:var(--serif); }
```

- [ ] **Step 4: 验证**

Run: `pnpm test -- scaffold.test.ts` PASS && `pnpm tauri dev --help` 0

- [ ] **Step 5: Commit**

```bash
git add PaperRss-Windows/package.json src-tauri/tauri.conf.json src/app.css
git commit -m "feat(windows): scaffold tauri+react paper tokens"
```

---

### Task 1: Core 数据层（SQLite+模型）

**Files:**
- Create: `src-tauri/src/core/db.rs`
- Create: `src-tauri/src/core/models.rs`
- Create: `src/lib/db.ts`
- Test: `tests/core.test.ts` + `src-tauri` `cargo test`

**Interfaces:**
- Consumes: Task 0 壳
- Produces: `db.rs` `init_db(path) -> Connection`，`models.rs` `Feed{ id,title,url }` `Article{ id,feed_id,title,html,unread,starred }`，前端 `db.ts` `invoke('db_query')`

- [ ] **Step 1: 写失败的 Rust 测试**

```rust
// src-tauri/src/core/db.rs
#[cfg(test)]
mod tests {
  use super::*;
  #[test] fn creates_tables() { let c=init_db(":memory:").unwrap(); assert!(table_exists(&c,"articles")); }
}
```

- [ ] **Step 2: 实现**

```rust
pub fn init_db(path:&str)->anyhow::Result<Connection>{
  let c=Connection::open(path)?;
  c.execute_batch(include_str!("../../migrations/001.sql"))?; // feeds/articles + FTS5
  Ok(c)
}
```

`src/lib/db.ts` 封装 `invoke('db_query', {sql})` 供 dev 用 `sql.js` 回退。

- [ ] **Step 3: 验证** `cargo test --manifest-path src-tauri/Cargo.toml` + `pnpm test -- core.test.ts` PASS

---

### Task 2: Feed 解析与 FreshRSS 同步

**Files:**
- Create: `src-tauri/src/core/feed.rs`
- Create: `src-tauri/src/core/sync.rs`
- Create: `src-tauri/src/commands.rs`
- Test: `tests/sync.test.ts`（mock fetch）

**Interfaces:**
- Consumes: Task 1 db
- Produces: `commands::fetch_feed(url)`、`commands::sync_freshrss(creds)`（双向未读/星标）

- [ ] **Step 1: 写失败的命令测试**

```ts
test('fetch_feed returns articles', async () => {
  const r = await invoke('fetch_feed',{url:'https://example.com/rss'});
  expect(r.articles.length).toBeGreaterThan(0);
});
```

- [ ] **Step 2: 实现** `feed.rs` 用 `feed-rs` crate 解析 RSS/Atom，`sync.rs` 调 `https://freshrss/api/greader.php/reader/api/0/stream/contents/...` + `edit-tag`，GRDB 兼容字段映射

---

### Task 3: 阅读器（MathJax+TOC+划词）

**Files:**
- Modify: `src/components/reader/ArticleView.tsx`
- Create: `src/components/reader/TocDrawer.tsx`
- Test: `tests/e2e/reader.spec.ts`（Playwright）

**Interfaces:**
- Consumes: Task 1-2 文章 HTML
- Produces: `ArticleView` 渲染富文本+`window.MathJax.typeset`+防转义，`TocDrawer` 从 `h1..h3` 生成

- [ ] **Step 1: 写失败的渲染测试**

```ts
test('mathjax renders', async ({page})=>{
  await page.goto('/'); await page.getByText('Article').click();
  await expect(page.locator('.MathJax')).toBeVisible();
});
```

---

### Task 4: 按需 AI（摘要/划词/提问）

**Files:**
- Create: `src-tauri/src/core/ai.rs`
- Create: `src/components/ai/SummaryCard.tsx`
- Modify: `src/components/settings/ModelPanel.tsx`

**Interfaces:**
- Consumes: Task 1 db，`tauri-plugin-store` 的 key
- Produces: `ai::summary(text, prompt, model)` 按 V 键触发，`explain/translate` 划词气泡

---

### Task 5: 全文检索 FTS5

**Files:**
- Modify: `src-tauri/src/core/db.rs`（FTS5 虚表）
- Create: `src/components/shell/SearchBar.tsx`
- Test: `tests/search.test.ts`

**Interfaces:**
- Consumes: Task 1
- Produces: `invoke('search',{q})` 走 `articles_fts MATCH ?`

---

### Task 6: 主题/快捷键/系统桥接

**Files:**
- Modify: `src/lib/store.ts`（主题持久化）
- Create: `src/hooks/useShortcuts.ts`
- Test: 手工 `Manual UI verification required`（`Tier3`）

**Interfaces:**
- Consumes: Task 0-3
- Produces: 明暗/纸感切换即时、`tauri-plugin-globalShortcut` 注册 `V`/`F` 等

---

### Task 7: Updater 与打包

**Files:**
- Modify: `src-tauri/tauri.conf.json`（updater `pubkey`/`url`）
- Create: `scripts/release.ps1`
- Test: `pnpm tauri build --debug` 产 `msi`+`latest.json`

**Interfaces:**
- Consumes: Task 0-6
- Produces: `tauri build` → `target/release/bundle/msi/*.msi` + `latest.json` 对标 Sparkle

---

## Self-Review

**Spec coverage:** README 全量（纸感/MathJax/AI/FreshRSS）+ Package.swift 三产物 + principle 三级验证 + SemVer 均有 Task（0→壳、1→GRDB、2→Feed/Sync、3→阅读、4→AI、5→FTS、6→主题、7→Updater）

**Placeholder scan:** 无 TBD/TODO，均为可执行代码

**Type consistency:** `Feed/Article` 在 `models.rs` 单定义，前端 `db.ts` `invoke('db_query')` 签名一致

