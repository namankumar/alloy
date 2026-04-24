# alloy

Claude Code for knowledge workers.

Claude Code is a CLI harness: it reads skill files, loads context, orchestrates AI agents, and writes output. It's powerful and locked to one machine, one person, no UI, no web access. Alloy is the same thing with a GUI and a web layer — built for knowledge workers instead of developers.

**Skills are first-class.** Reusable AI workflows stored as plain markdown. Load them, run them, share them with a teammate. The same concept as Claude Code's skill system, visible and editable in the UI.

**Context management.** The files your AI reads before working — current priorities, org dynamics, project state — surfaced, organized, and accessible. Not buried in a local folder on one machine.

**Agent orchestration.** Trigger AI workflows from the UI, not a terminal. The harness runs the skill; you see the output. Same model as Claude Code, no CLI required.

**BYOAI.** Your keys, your model. A Rust router in the Tauri backend handles all provider communication — OpenAI, Anthropic, Ollama, any OpenAI-compatible endpoint. Keys live in the OS keychain, never on disk unencrypted.

**Web-accessible.** The workspace runs in a browser. Access it remotely, share a skill, a context file, or the whole system with a colleague. Private by default. You decide what's visible and to whom.

**Local-first, self-hostable.** Everything is plain `.md` on disk. Run your own sync server (Docker) so files never touch Alloy's infrastructure. Or use Alloy Cloud for managed sync — opt-in, never the default.

This repo is a product spec. No code yet. See `PLAN.md` for the full build plan.

## Why not ChatGPT or Claude?

ChatGPT Canvas and Claude Projects are close in feel. The gap: their artifacts live in vendor databases, their memory is vendor-managed, and their "skills" are custom GPTs someone else controls. Alloy's files are `.md` on disk. Skills are files you write, version, and share. Memory is the workspace. Nothing lives on Alloy's servers unless you choose it.

**Alloy is for teams that want to own their AI workflow, not rent it.**

## Why file-first

We chose file-based output over a chat interface because knowledge work output needs to persist. Chat tools give you answers. Alloy gives you files. Every output is a markdown artifact you own and version-control. The constraint forced us toward local-first: if output lives in files, the sync layer becomes the hard problem.

## Sync tiers

| Tier | How it works | Real-time collab |
|---|---|---|
| Git | Commit to a repo, web reads from it | No |
| Self-hosted | Run the Alloy server on your infra | Yes |
| Alloy Cloud | Managed sync, opt-in | Yes |

## What lives in a workspace

| Layer | What it is | Claude Code equivalent |
|---|---|---|
| Skills | Reusable AI workflow definitions | `.claude/skills/` |
| Context | Files the AI reads before working | `context/`, `CLAUDE.md` |
| Output | Documents, strategies, notes produced | `docs/`, `cache/` |

## Stack

- **Tauri:** Rust backend, system WebView frontend
- **React + TypeScript:** frontend
- **TipTap:** block editor (ProseMirror-based)
- **SQLite FTS5:** full-text search index, built from file watcher, deletable/rebuildable
- **OS keychain:** API key storage

## BYOAI architecture

```
Frontend (React)
  → Tauri invoke("ai_stream", { prompt, context })
    → AI Router (Rust)
      → OpenAI / Anthropic / Ollama / Custom adapter
      → Stream tokens back via Tauri events
```

## Status

Spec phase. Looking for collaborators, particularly on the TipTap editor layer and the Rust AI router.
