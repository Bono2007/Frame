# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

> **Note:** This file is named `AGENTS.md` to be AI-tool agnostic. A `CLAUDE.md` symlink is provided for Claude Code compatibility.

---

## Commands

```bash
npm run dev        # Watch-mode build + launch Electron (development)
npm start          # One-time build + launch Electron
npm run build      # Bundle renderer only (esbuild → dist/renderer.js)
npm run watch      # Watch renderer bundle only
npm run structure  # Regenerate STRUCTURE.json
npm run dist:mac   # Package as macOS .dmg (requires signing identity)
npm run dist:mac:unsigned  # Package without code signing
```

No test runner is configured. There are no lint scripts.

---

## Architecture

Frame is an **Electron desktop app** split into three processes:

### Main Process (`src/main/`)
Runs in Node.js. Manages PTYs (node-pty), filesystem, and all IPC handlers. Each feature has its own manager file (e.g. `tasksManager.js`, `githubManager.js`). Entry point: `src/main/index.js` — calls `setupAllIPC()` which delegates to each manager's `setupIPC(win)`.

### Renderer Process (`src/renderer/`)
Browser context, **bundled by esbuild** into `dist/renderer.js`. No `require()` available — use `window.ipcRenderer` (Electron's `contextBridge` is not used; `nodeIntegration: true`). Entry point: `src/renderer/index.js`. Each UI panel is its own module (`tasksPanel.js`, `pluginsPanel.js`, etc.).

### Shared (`src/shared/`)
`ipcChannels.js` is the **single source of truth** for all IPC channel names — always import from here, never hardcode strings.

### IPC Pattern
```
Renderer → Main:  ipcRenderer.send(IPC.CHANNEL, payload)
Main → Renderer:  win.webContents.send(IPC.CHANNEL, data)
Main → Renderer (reply): event.reply(IPC.CHANNEL, data)
```

### Key Conventions
- New features require both a main-side manager (`src/main/`) and a renderer-side panel (`src/renderer/`), plus IPC constants in `src/shared/ipcChannels.js`
- After adding/moving files, run `npm run structure` to update `STRUCTURE.json`
- STRUCTURE.json is updated automatically by the pre-commit hook

---

## Core Working Principle

**Only do what the user asks.** Do not go beyond the scope of the request.

- Implement exactly what the user requested — nothing more, nothing less.
- Do not change business logic, flow, or architecture unless the user explicitly asks for it.
- If a user asks for a design change, only change the design. Do not refactor, restructure, or modify functionality alongside it.
- If you have additional suggestions or improvements, **present them as suggestions** to the user. Never implement them without approval.
- The user's request must be completed first. Additional ideas come after, as proposals.

**Example:** If the user asks for a modal design change, only change the visual appearance. Do not add new IPC channels, modify event flows, or restructure code.

---

## 🧭 Project Navigation

**Read these files at the start of each session:**

1. **STRUCTURE.json** - Module map, which file is where
2. **PROJECT_NOTES.md** - Project vision, past decisions, session notes
3. **tasks.json** - Pending tasks

**Workflow:**
1. Read these files to understand the project and capture context
2. Identify relevant files based on the task
3. Update STRUCTURE.json after making changes (if new modules/files are added)

**Fast File Lookup:** When searching for files related to a feature or concept, run:
```bash
node scripts/find-module.js <keyword>
```
This searches STRUCTURE.json's intentIndex and returns the exact files you need. Use this **before** doing manual grep/glob searches. Examples:
- `node scripts/find-module.js github` → finds githubManager.js + githubPanel.js
- `node scripts/find-module.js terminal` → finds all terminal-related files
- `node scripts/find-module.js --list` → lists all features and their files

**Note:** This system doesn't prevent reading code - it just helps you know where to look.

---

## Task Management (tasks.json)

### Task Recognition Rules

**These ARE TASKS - add to tasks.json:**
- When the user requests a feature or change
- Decisions like "Let's do this", "Let's add this", "Improve this"
- Deferred work when we say "We'll do this later", "Let's leave it for now"
- Gaps or improvement opportunities discovered while coding
- Situations requiring bug fixes

**These are NOT TASKS:**
- Error messages and debugging sessions
- Questions, explanations, information exchange
- Temporary experiments and tests
- Work already completed and closed
- Instant fixes (like typo fixes)

### Task Creation Flow

1. Detect task patterns during conversation
2. Ask the user at an appropriate moment: "I identified these tasks from our conversation, should I add them to tasks.json?"
3. If the user approves, add to tasks.json

### Task Structure

```json
{
  "id": "unique-id",
  "title": "Short and clear title (max 60 characters)",
  "description": "AI's detailed explanation - what will be done, how it will be done, which files will be affected",
  "userRequest": "User's original request/prompt - copy exactly",
  "acceptanceCriteria": "When is this task considered complete? List of concrete criteria",
  "notes": "Important notes, decisions, alternatives that came up during discussion",
  "status": "pending | in_progress | completed",
  "priority": "high | medium | low",
  "category": "feature | fix | refactor | docs | test",
  "context": "Session date and context",
  "createdAt": "ISO date",
  "updatedAt": "ISO date",
  "completedAt": "ISO date | null"
}
```

### Task Content Rules

**title:** Short, action-oriented title
- ✅ "Add tasks button to terminal toolbar"
- ❌ "Tasks"

**description:** AI's detailed technical explanation
- What will be done (what)
- How it will be done (how) - brief technical approach
- Which files will be affected
- Minimum 2-3 sentences

**userRequest:** User's original words
- Copy the user's prompt/request exactly
- Important for preserving context
- In "User said: ..." format

**acceptanceCriteria:** Completion criteria
- Concrete, testable items
- "Task is complete when this happens" list

**notes:** Discussion notes (optional)
- Alternatives considered
- Important decisions and their reasons
- Dependencies marked as "we'll do this later"

### Task Status Updates

- When starting work on a task: `status: "in_progress"`
- When task is completed: `status: "completed"`, update `completedAt`
- After commit: Check and update the status of related tasks

---

## PROJECT_NOTES.md Rules

### When to Update?
- When an important architectural decision is made
- When a technology choice is made
- When an important problem is solved and the solution method is noteworthy
- When an approach is determined together with the user

### Format
Free format. Date + title is sufficient:
```markdown
### [2026-01-26] Topic title
Conversation/decision as is, with its context...
```

### Update Flow
- Update immediately after a decision is made
- You can add without asking the user (for important decisions)
- You can accumulate small decisions and add them in bulk

---

## 📝 Context Preservation (Automatic Note Taking)

Frame's core purpose is to prevent context loss. Therefore, capture important moments and ask the user.

### When to Ask?

Ask the user when one of the following situations occurs: **"Should I add this conversation to PROJECT_NOTES.md?"**

- When a task is successfully completed
- When an important architectural/technical decision is made
- When a bug is fixed and the solution method is noteworthy
- When "let's do this later" is said (in this case, also add to tasks.json)
- When a new pattern or best practice is discovered

### Completion Detection

Pay attention to these signals:
- User approval: "okay", "done", "it worked", "nice", "fixed", "yes"
- Moving from one topic to another
- User continuing after build/run succeeds

### How to Add?

1. **DON'T write a summary** - Add the conversation as is, with its context
2. **Add date** - In `### [YYYY-MM-DD] Title` format
3. **Add to Session Notes section** - At the end of PROJECT_NOTES.md

### When NOT to Ask

- For every small change (it becomes spam)
- Typo fixes, simple corrections
- If the user already said "no" or "not needed", don't ask again for the same topic in that session

### If User Says "No"

No problem, continue. The user can also say what they consider important themselves: "add this to notes"

---

## STRUCTURE.json Rules

**This file is the map of the codebase.**

### When to Update?
- When a new file/folder is created
- When a file/folder is deleted or moved
- When module dependencies change
- When an IPC channel is added or changed
- When an important architectural pattern is discovered (architectureNotes)

### Format
```json
{
  "modules": {
    "main/tasksManager": {
      "path": "src/main/tasksManager.js",
      "purpose": "Task CRUD operations",
      "exports": ["init", "loadTasks", "addTask"],
      "depends": ["fs", "path", "shared/ipcChannels"]
    }
  },
  "ipcChannels": {
    "LOAD_TASKS": {
      "direction": "renderer → main",
      "handler": "main/tasksManager.js"
    }
  },
  "architectureNotes": {
    "circularDependencies": {
      "issue": "Description",
      "solution": "Solution"
    }
  }
}
```

### Update Rules
- Pre-commit hook updates automatically (before commit)
- Manual: `npm run structure`
- If you added a new IPC channel, check the ipcChannels section

---

## QUICKSTART.md Rules

### When to Update?
- When installation steps change
- When new requirements are added
- When important commands change

---

## General Rules

1. **Language:** Write documentation in English (except code examples)
2. **Date Format:** ISO 8601 (YYYY-MM-DDTHH:mm:ssZ)
3. **After Commit:** Check tasks.json and STRUCTURE.json
4. **Session Start:** Review pending tasks in tasks.json

---

*This file was automatically created by Frame.*
*Creation date: 2026-01-24*
