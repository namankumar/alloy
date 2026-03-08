# Plan: Alloy — Markdown-Native Notion Replacement

## Context
Personal project. A local-first, markdown-native knowledge workspace that replaces Notion for individuals. Core bets: plain .md files on disk (zero lock-in), fast Tauri-based desktop app, and AI powered by the user's own keys (OpenAI, Anthropic, Ollama, etc.) instead of a subscription.

Lives at: `/projects/alloy/`

---

## Product Scope (v1)

Replace Notion's core personal workflows:

| Notion Feature | Slate Approach |
|---|---|
| Pages + hierarchy | Nested .md files in folders — filesystem IS the workspace |
| Rich block editing | TipTap with custom nodes |
| Slash commands | `/` command palette |
| Code / math / mermaid | Syntax-highlighted fenced blocks via CodeMirror + KaTeX + Mermaid.js |
| Images + embeds | Local file embeds via TipTap image node |
| Templates | .md template files in `_templates/` folder |
| Full-text search | SQLite FTS5 index built from file watcher |
| Tags + backlinks | YAML frontmatter tags + `[[wikilinks]]` with SQLite backlink table |
| **Databases** | Defer to v2 (YAML frontmatter + `_schema.yaml` convention) |

**Cut from v1:** database views, sync, graph view, real-time collab, web clipper.

---

## Tech Stack

- **Runtime:** Tauri (Rust backend, system WebView frontend — fast startup, small bundle)
- **Frontend:** React + TypeScript
- **Editor:** TipTap (ProseMirror-based, block editing, extensible)
- **Storage:** Plain .md files (source of truth) + SQLite index (cache only — deletable and rebuildable)
- **Sync (v1):** None — user uses iCloud/Dropbox/git on their own
- **Build:** pnpm monorepo

---

## BYOAI Architecture

Rust AI router in the Tauri backend. Frontend never knows which provider is active. Keys stored in OS keychain (never on disk unencrypted).

```
Frontend (React)
  → Tauri invoke("ai_stream", { prompt, context })
    → AI Router (Rust)
      → OpenAI adapter / Anthropic adapter / Ollama adapter / Custom OpenAI-compat
      → Stream tokens back via Tauri events
```

**Provider config:** JSON in OS keychain. Supports multiple providers, one active default.

**AI features (v1):**
- Inline ghost text completions (Tab to accept, 800ms debounce)
- `/AI` slash commands: summarize, expand, rewrite, bullet list
- Chat sidebar with current doc as context
- **Prompt Library** (personal + team-shared — see section below)

**Zero-friction default:** Ship with Ollama-compatible local model pre-configured — works offline, no key needed. BYOAI unlocks better models.

---

## Prompt Library

A shared team prompt database built into the AI layer.

**Saving a prompt:**
- Any AI interaction (slash command, chat, inline completion) has a one-click "Save prompt" action
- On save: give it a title, optional tags, choose to include output or prompt-only, mark as personal or team-shared

**Storage:**
- Personal prompts: `~/.alloy/prompts.json` (local, never synced)
- Team prompts: `_prompts/` folder inside the workspace — plain `.json` files, synced with the rest of the workspace via git/iCloud/Dropbox like any other file

**Using prompts:**
- From the `/AI` slash command palette: browse and search saved prompts instead of writing from scratch
- Prompts are parameterized — `{{selected_text}}`, `{{doc_title}}`, `{{date}}` get substituted at run time
- Anyone can fork a team prompt, edit it, and re-share the improved version

**Prompt evolution model:**
Prompts improve through contributions over time, not just individual saves.

- **Versioning:** Every edit to a team prompt creates a new version (stored as a version history array in the JSON). You can see who changed what and roll back.
- **Upvotes / usefulness signal:** After running a prompt, a lightweight "Was this useful?" thumbs up/down is shown. The score is stored on the prompt and surfaces in search ranking — better prompts float to the top.
- **Suggested edits:** Anyone can propose an edit to a team prompt without overwriting it. The edit sits as a `suggested_edit` on the prompt until the original author (or a designated prompt maintainer) approves it and merges it as a new version.
- **Usage count:** Tracked locally. Prompts used more frequently rank higher in the palette.
- **Community library (v2+):** An optional public prompt registry — teams can publish prompts to a shared Slate community index, browse prompts from other teams/users, and import them into their own workspace. All still BYOAI — the prompts are just text, no data leaves without explicit action.

**Team prompt schema (`_prompts/<slug>.json`):**
```json
{
  "id": "summarize-meeting-notes",
  "title": "Summarize meeting notes",
  "prompt": "Summarize the following meeting notes into: key decisions, action items, and open questions.\n\n{{selected_text}}",
  "include_output": false,
  "tags": ["meetings", "summarization"],
  "author": "naman",
  "created": "2026-03-07",
  "upvotes": 12,
  "usage_count": 47,
  "versions": [
    { "version": 1, "prompt": "...", "author": "naman", "date": "2026-03-07" },
    { "version": 2, "prompt": "...", "author": "alaa", "date": "2026-03-14" }
  ],
  "suggested_edits": []
}
```

**Why this matters:** Teams accumulate institutional AI knowledge — the prompts that actually work for their domain. Right now that lives scattered across Slack and Notion docs, undiscoverable and never improved. Slate makes it a first-class, versioned, community-improvable artifact. No equivalent anywhere.

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

**v1 (months 1-4): The Editor**
Tauri app (macOS first), full TipTap editing, file tree, backlinks, FTS search, BYOAI (inline + slash + chat), Prompt Library (personal + team-shared via `_prompts/` folder), Notion import wizard, templates. Success: user can migrate personal Notion and not go back.

**v2 (months 5-8): Databases + Sync**
Database views (table, kanban), `_schema.yaml` convention, git-based sync, semantic search/RAG, graph view, Windows/Linux parity.

**v3 (months 9-14): Collaboration**
Yjs CRDTs, comments, publish to web (static site export), plugin API, mobile companion.

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
