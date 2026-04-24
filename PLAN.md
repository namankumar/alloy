# Plan: Alloy — Claude Code for Knowledge Workers

## Context
Personal project. Alloy is an AI harness for knowledge work: the same model as Claude Code (skills, context, agent orchestration, file-based output), with a GUI and a web layer instead of a CLI.

Claude Code is powerful and locked to one machine, one person, no UI, no web access. Alloy solves that. The workspace holds skills (reusable AI workflows), context (what the AI reads before working), and output (what it produces). Alloy makes that system remotely accessible, shareable with colleagues, and operable from a browser — without changing how the underlying AI tools work.

**Core positioning: Alloy is for teams that want to own their AI workflow, not rent it.**

ChatGPT Canvas and Claude Projects are close in feel — persistent docs, AI iteration, some memory. The gap: their artifacts live in vendor databases, their "skills" are custom GPTs someone else controls, and their memory is vendor-managed. Alloy's files are `.md` on disk. Skills are files you write, version, and share. Memory is the workspace. Nothing lives on Alloy's servers unless you choose it.

Cursor is to VS Code what Alloy is to Obsidian. Obsidian is a text editor with AI bolted on as plugins and a read-only static site as its "web layer." Alloy is built around AI as the core interaction model with real multiplayer, real permissions, and a first-class web interface.

**Workspace layers:**

| Layer | What it is | Claude Code equivalent |
|---|---|---|
| Skills | Reusable AI workflow definitions | `.claude/skills/` |
| Context | Files the AI reads before working | `context/`, `CLAUDE.md` |
| Output | Documents, strategies, notes produced | `docs/`, `cache/` |

Lives at: `/projects/alloy/`

---

## Product Scope

### Primitives

**Workspace > Projects > Files.** Three-level hierarchy. Workspaces contain projects; projects contain files and folders. Switch between workspaces without closing the app. Each workspace has its own skills, context, and permissions.

**Multiplayer at the foundation.** Yjs CRDTs from day one — not bolted on in v3. Every doc is collaborative by default. Presence, cursors, real-time edits. Permissions per file and per folder: view, comment, edit, admin. Permissions are set by the workspace owner and propagate down unless overridden.

**Memory = the workspace.** No separate memory layer. The files in `context/` are what the AI reads. What the user writes is what the AI knows. Local by default; shared when permissions allow.

### Skills

Skills are the core unit of AI work. A skill is a markdown file: plain instructions, variables, and tool calls. Creating one is the same act as writing a doc.

- **Create:** write a `.md` file in `_skills/`, describe what it does, define inputs with `{{variables}}`
- **Save:** it's a file — committed to the workspace like any other doc
- **Execute:** select text, invoke a skill from the command palette or `/` slash menu, or run it on a schedule. Output streams inline or to a new doc.
- **Share:** skills are files — share them with a colleague the same way you share any doc. Permissions apply.
- **Out-of-box skills:** voice transcription, formatting cleanup, summarize, expand, rewrite, extract action items, weekly brief. Shipped in `_skills/defaults/`, editable and forkable.

### Editor

- Rich block editing via TipTap (ProseMirror-based)
- **Inline selections:** select any text, invoke a skill or AI command on the selection. Output replaces, appends, or opens in sidebar.
- Slash command palette: `/skill`, `/AI`, `/insert`, `/file`
- Code, math (KaTeX), Mermaid diagrams, wikilinks `[[]]`
- **Drag and drop:** images, files, URLs. Images stored in `_assets/` alongside the doc. Files linked by relative path.
- YAML frontmatter for metadata, tags, schema

### File Operations

CLI-like tools available from the command palette and keyboard shortcuts:

| Operation | Description |
|---|---|
| `find` | Search files by name, path, or frontmatter |
| `grep` | Full-text search across workspace (FTS5) |
| `mv` | Move/rename files and folders, updates all backlinks |
| `cp` | Duplicate a file or folder |
| `mkdir` | Create folder structure |

All file ops are undoable. Backlinks and wikilinks update automatically on `mv`.

### Tools & Integrations

Built-in tools available to skills and the AI router:

- **Web:** browse URLs, search the web, fetch page content
- **Google Drive:** read and write docs, sheets, slides
- **Analytics:** query Mixpanel, Amplitude, GA — surface data inline
- **Calendar:** read events, create holds
- **Linear / Monday:** read issues, update status, create tasks
- **Custom:** any OpenAI-compatible tool spec can be registered in workspace settings

Tools are declared in skill files and approved per-workspace. No tool runs without user-level permission.

### BYOAI

Rust AI router in the Tauri backend. Frontend never knows which model is active. Keys in OS keychain.

```
Frontend (React)
  → Tauri invoke("ai_stream", { prompt, context, tools })
    → AI Router (Rust)
      → OpenAI / Anthropic / Ollama / Custom adapter
      → Tool calls dispatched and results returned
      → Stream tokens back via Tauri events
```

Providers: OpenAI, Anthropic, Ollama, any OpenAI-compatible endpoint. Zero-friction default: local Ollama model, no key needed. BYOAI unlocks better models.

**Cut from v1:** database views, graph view, web clipper, mobile app.

---

## Local + Web Architecture

**The sync problem:** local app and web need to share state. Files are the source of truth. Three tiers:

| Tier | How it works | Who it's for |
|---|---|---|
| **Git sync** | Local app commits to a repo. Web reads from the same repo. No real-time collab, async only. | Solo users, async teams |
| **Self-hosted server** | User runs an Alloy server (Docker). Local and web connect to it. Files on their infra. Yjs runs through the server — real-time collab works. | Teams that want full ownership |
| **Alloy Cloud** | Alloy hosts the sync server. Opt-in. Files replicated to Alloy's servers. Managed, no ops required. | Teams that want convenience |

Default: self-hosted. Cloud is opt-in. Files never touch Alloy's servers unless you choose it.

**Local app** is the full-power version: all file ops, CLI tools, Claude Code integration, offline mode, OS keychain for keys. **Web** is the collaboration and access layer: editing, skill execution (via the server's AI router), sharing. Same files, same Yjs state — different surface.

```
Local App (Tauri)          Alloy Server (self-hosted or cloud)       Web App (browser)
─────────────────          ───────────────────────────────────       ─────────────────
File system (source) ───►  Yjs sync + AI router + permissions  ◄─── React frontend
AI router (local)          Files on server or S3                     No local file access
File ops (grep/mv/find)    WebSocket for real-time                   Full editor + skills
Offline capable            REST for async                            Requires connection
```

**Pricing model:**
- **Open source:** self-hosted, unlimited, free. No keys sent to Alloy.
- **Alloy Cloud:** managed sync + web access. Per-seat pricing. Files on Alloy's infra (opt-in).
- **Alloy Cloud Enterprise:** SSO, audit logs, data residency, SLA.

---

## Tech Stack

- **Runtime:** Tauri (Rust backend, system WebView frontend)
- **Frontend:** React + TypeScript
- **Editor:** TipTap (ProseMirror-based, block editing, extensible)
- **Multiplayer:** Yjs CRDTs (from v1)
- **Storage:** Plain `.md` files (source of truth) + SQLite index (cache, deletable/rebuildable)
- **Sync:** Git (v1 async), self-hosted Alloy server (v2 real-time), Alloy Cloud (v2 managed)
- **Build:** pnpm monorepo

---

---

## Branding & Design

**Name: Alloy.**
A blend of elements, stronger than any single part. You bring the raw materials — your thinking, your AI. Alloy shapes them into something durable. Metallic, fits the dark aesthetic. SEO-ownable: distinctive enough that "Alloy app" immediately surfaces the product with no competition. The metaphor holds across the whole product and across the AI age: human + machine, shaped into something neither could produce alone.

**Design intent:** Smooth and simple on the surface, capable underneath. The complexity is always one step away, never in your face. Users should feel invited the moment they open it, and never feel like they're fighting the tool to get a result.

**Emotional targets:** Invited. Trusting. Frictionless.

---

### Visual Language

**Typography — the primary design element.** No logo yet, so type carries the brand.
- Body: [Inter](https://rsms.me/inter/) or [Geist](https://vercel.com/font) — clean, legible at small sizes, feels modern without being trendy
- Editor: [iA Writer Quattro](https://ia.net/topics/a-typographic-christmas) or [Lora](https://fonts.google.com/specimen/Lora) — proportional serif for actual writing. Reading your own words in a good serif makes them feel worth reading.
- Mono: [Geist Mono](https://vercel.com/font) — code blocks, frontmatter, prompt schema

**Color — dark mode lead.**
First users are developers. Dark mode is the primary experience; light mode ships as an alternative.

- **Dark background:** `#141414` — deep neutral, not true black, not navy. Warm enough to feel intentional, dark enough to make content pop.
- **Dark surfaces:** `#1C1C1C` sidebar, `#222222` panels — subtle layering without heavy borders.
- **Text:** `#E8E6E1` — warm white, not pure white. Reads as light without eye strain against dark.
- **Light mode background:** `#F7F6F3` — stone white, cooler than Notion's cream. Distinct from day one.
- **Light mode text:** `#1A1A18` — near-black.

**Accent — gradient, not flat.**
A single gradient is the brand's only expressive moment. Used sparingly: active sidebar item indicator, selected text highlight, AI streaming cursor, primary buttons, focus rings.

- Direction: left to right, slight diagonal
- Candidate: indigo → violet (`#6366F1` → `#A855F7`) — feels modern, premium, not startup-blue
- Alternative: blue → cyan (`#3B82F6` → `#06B6D4`) — cleaner, more neutral, better for a tool
- The gradient never appears on text or large surfaces. Only on small interactive elements (2-4px indicator bars, button fills, underlines). Everywhere else is flat.

**AI visual treatment:**
AI-generated content gets a left border in the accent gradient color — thin (2px), not a full highlight. Subtle enough that it doesn't interrupt reading, clear enough that you always know what the model wrote vs. what you wrote.

**Spacing — generous.**
- Editor max-width: 680px centered. Wider than iA Writer, narrower than Notion's default. Optimized for reading and writing, not for fitting a sidebar + content + properties panel.
- Sidebar: collapsible, narrow by default (220px). Disappears when you're writing.
- Line height: 1.7 in the editor. Breathing room.

**Motion — minimal and purposeful.**
- No decorative animations. Transitions only where they communicate state (panel open/close, AI streaming, search results appearing).
- Duration: 120-180ms. Fast enough to feel native, slow enough to feel smooth.
- Easing: ease-out for entrances, ease-in for exits. Nothing bouncy.

---

### UI Principles

1. **The editor is the product.** Everything else (sidebar, AI panel, search) is support. Default state is full-width editor, nothing else visible.
2. **Progressive disclosure.** Slash commands reveal power without exposing it upfront. Databases, schema, views — only visible when you need them.
3. **AI is ambient, not modal.** Inline completions are ghosted text. The chat panel slides in from the side without covering the document. AI never hijacks the writing experience.
4. **Empty states earn trust.** A new workspace should feel like a fresh notebook, not a dashboard with no data. First-run experience shows one sample document and a clear prompt: "Open a folder to get started."
5. **Errors are human.** When something goes wrong (AI request fails, file can't be read), the message is plain English with a clear next step. No stack traces, no modal dialogs.

---

### Component Defaults

- **Component library:** [Radix UI](https://www.radix-ui.com/) primitives (accessible, unstyled) + [Tailwind CSS](https://tailwindcss.com/) for styling
- **Icons:** [Lucide](https://lucide.dev/) — consistent stroke weight, neutral style
- **Transitions:** [Framer Motion](https://www.framer.com/motion/) for layout animations only where needed
- **Fonts loaded locally** — no Google Fonts CDN calls, respects privacy posture

---

## File Format

Standard CommonMark + YAML frontmatter. Non-standard but universally supported extensions: `[[wikilinks]]`, mermaid fenced blocks, `$$` math. No proprietary syntax. Files open in any text editor.

**Database convention (v2):**
- Folder = database
- `_schema.yaml` at folder root defines columns + views
- Each .md file's frontmatter = a row

---

## Monorepo Structure

```
/projects/alloy/
├── apps/
│   └── desktop/
│       ├── src/                  # React frontend
│       │   ├── editor/           # TipTap + extensions
│       │   ├── sidebar/          # File tree, backlinks, outline
│       │   ├── ai/               # Chat sidebar, inline completion UI, prompt library UI
│       │   ├── search/           # Cmd+K search UI
│       │   └── settings/         # Provider config, appearance
│       └── src-tauri/            # Rust backend
│           ├── commands/         # Tauri command handlers
│           ├── indexer/          # File watcher + SQLite indexer
│           └── ai/               # AI router + provider adapters
├── packages/
│   ├── editor-core/              # TipTap extensions
│   ├── markdown-parser/          # remark-based MD ↔ ProseMirror
│   ├── schema/                   # Shared TS types
│   └── ui/                       # Radix + Tailwind components
└── tooling/
```

---

## Critical Files (build in this order)

1. `apps/desktop/src-tauri/src/indexer/mod.rs` — file watcher + SQLite indexer. Foundation for search, backlinks, everything.
2. `packages/markdown-parser/src/index.ts` — remark parser + ProseMirror serializer. Round-trip fidelity is critical (any loss corrupts user files).
3. `apps/desktop/src-tauri/src/ai/router.rs` — provider trait, streaming interface, keychain integration.
4. `packages/editor-core/src/extensions/` — TipTap custom nodes: slash commands, math, mermaid, wikilinks, frontmatter.
5. `apps/desktop/src/editor/Editor.tsx` — top-level editor component: TipTap init, save debounce, AI completion integration.

---

## Build Sequence

**v1 (months 1-4): The Harness**
Tauri app (macOS first). Multiplayer foundation (Yjs). Full editor (TipTap, drag-and-drop, inline selections). Skills system (`_skills/` folder, run from command palette). Context management UI. File ops (find, grep, mv). BYOAI router. Out-of-box skills. Permissions per doc. Success: user runs their full knowledge workflow — skills, context, output — entirely in Alloy.

**v2 (months 5-8): Web + Integrations**
Web interface: workspace accessible from any browser, no install required. Tool integrations (web browse, Google Drive, analytics, Linear, Calendar). Database views (table, kanban). Git-based sync. Success: a colleague opens a shared workspace in a browser and runs a skill without installing anything.

**v3 (months 9-14): Platform**
Plugin API for custom tools and skills. Mobile companion (read + light edit). Community skill library. Public workspace publishing.

---

## Notion Import (v1)

Wizard that takes a Notion export folder:
1. Walk directory tree, remap internal UUID links to slugs
2. Rename UUID image filenames
3. For DB exports: parse CSV → one .md file per row with YAML frontmatter + `_schema.yaml`
4. Copy into workspace

---

## Performance Targets

- App launch → ready: < 1 second
- File open (5000 words): < 100ms
- Search results: < 50ms
- Editor input latency: < 16ms

---

## Unprioritized Features

Things Notion has that are not in scope for v1-v3 but worth tracking. Not cut — just unscheduled.

- Inline database embeds (database view dropped inside a page, not just at folder level)
- Database relations & rollups (linking rows across two databases, computing aggregates)
- Database formulas (computed columns)
- Database charts (bar/pie/line charts over database data)
- Gallery and timeline database views (v2 has table and kanban only)
- Third-party integrations (Slack, GitHub, Jira, Google Drive, etc.)
- Community template gallery / marketplace
- Role-based permissions (workspace members, guests, view-only, page-level sharing)
- Audit logs (who viewed/edited what, when)
- Rich embed blocks (YouTube, Figma, Loom, Google Maps — ~50 Notion embed types)
- Public API (Notion's API lets teams build automations and integrations)
- Page analytics (view counts)
- Notifications (in-app and email)
- Connected databases (same database surfaced in multiple pages)
- Import from tools other than Notion (Confluence, Evernote, Bear, etc.)
- PDF export
- Full-featured mobile app (v3 companion is read + light edit only)

---

## Testing Strategy

### Libraries

| Layer | Tool | Why |
|---|---|---|
| Rust unit tests | built-in `#[cfg(test)]` + `tokio::test` for async | no extra dep needed |
| Rust mocking | [`mockall`](https://github.com/asomers/mockall) | mock provider traits in AI router tests |
| Frontend unit/component | [Vitest](https://vitest.dev/) + [Testing Library](https://testing-library.com/) | fast, Vite-native, no JSDOM weirdness |
| E2E / app flows | [WebdriverIO](https://webdriver.io/) with Tauri driver | official Tauri-recommended E2E approach |
| Performance benchmarks | [Criterion.rs](https://github.com/bheisler/criterion.rs) for Rust, custom `performance.now()` harness for frontend | statistical benchmarks, not one-shot timers |
| Test fixtures / corpus | Plain `.md` files in `tests/fixtures/` | version-controlled, human-readable edge case corpus |

### What to test per layer

**Markdown parser — highest risk.**
Round-trip fidelity: `parse(serialize(parse(md))) === parse(md)` for every node type. Edge case corpus in `packages/markdown-parser/tests/fixtures/`: nested lists, frontmatter with special chars, fenced code with backticks inside, math, mermaid, wikilinks, empty docs, 50k-word docs. Every bug found adds a fixture permanently.

**Rust backend — unit tests per module.**
- AI router: mock providers via `mockall`, assert streaming token normalization, keychain read/write, graceful fallback when provider unreachable
- Indexer: write files to temp dir, assert SQLite updated within 500ms, delete files, assert removal, FTS query results, backlink extraction accuracy
- File watcher: rapid successive writes (debounce), binary files ignored gracefully, concurrent writes don't corrupt index

**Frontend — focused, not exhaustive.**
Vitest + Testing Library for: slash command trigger logic, custom TipTap node rendering, wikilink autocomplete, prompt library CRUD, parameterized substitution (`{{selected_text}}` etc.), search result keyboard navigation, provider config save/load, invalid API key error state.

**E2E — core flows only (WebdriverIO).**
- Create workspace → create file → type content → file on disk has correct markdown
- `[[` → autocomplete → select page → wikilink in file
- `/AI summarize` → result has gradient left border
- Save prompt → reopen app → prompt in palette
- Notion import → pages with correct titles and frontmatter
- Cmd+K → results appear → click → correct file opens

**Performance benchmarks (Criterion.rs + CI tracking).**

| Metric | Target |
|---|---|
| File open, 5000 words | < 100ms |
| FTS search | < 50ms |
| Markdown parse + serialize round-trip | < 10ms |
| App launch to ready | < 1 second |

Benchmarks run in CI on every PR. Regression vs. baseline blocks merge.

**What to skip:** UI snapshot tests (brittle, low signal), 100% coverage targets, mocking the filesystem in Rust (use real temp dirs — catches more, faster to write).

---

## Verification

- `cargo tauri dev` launches from `apps/desktop/`
- Open a folder → file tree populates
- Create `.md` file → appears in tree, FTS indexed
- Type `[[` → wikilink autocomplete works
- Type `/` → slash command palette opens
- Configure AI provider in settings → inline completion triggers on pause
- Import Notion export → pages with correct links and frontmatter
