# 🛡️ DSH Safety Rails

> **Confirm before you act.** A drop-in guardrail that forces AI coding agents to explain and wait for approval before touching any file — with a dual-layer design the model cannot bypass.

[![DeepSeek Harness](https://img.shields.io/badge/works%20with-DeepSeek%20Harness%20%7C%20AGENTS.md%20tools-blue)](https://github.com/deepseek-ai)

---

## The Problem

When using AI coding tools (DSH, Claude Code, Cursor, etc.), a common and expensive failure mode:

> **The agent starts editing files before the requirements are clear.**
> When the direction turns out to be wrong, you burn tokens and time rolling it back — and risk leaving the codebase in a half-mangled state.

This repository provides a **configuration template** that makes the agent **explain first, confirm second, act last**. It targets DeepSeek Harness (DSH), but the core approach — enforcement rules plus a permission gate — applies to any coding agent that supports system prompts and user-level instruction files.

---

## The Core Principle

> **A user's request is not a user's approval.**
> "Please create file X" means *propose the plan and confirm it* — not *go ahead and do it*.

The root cause is an interpretation gap: the agent reads "the user asked me to do X" as "the user authorized me to do X directly." Requirements are usually fuzzy, so the core rule is:

> **A user's *request to perform an action* is itself not consent.** Even when the user says "create file X" or "modify Y", the agent must first present the concrete change plan and wait for confirmation. Only an explicit waiver in the same message ("no need to confirm", "just do it") allows immediate execution.

---

## Two-Layer Protection

| Layer | Mechanism | Effect |
|---|---|---|
| **Soft ①** | `AGENTS.md` user rules, injected into every session | The model reads the rules and follows them voluntarily |
| **Soft ②** | Custom agent preset (rules embedded in the system-prompt persona) | Rules survive even when the `AGENTS.md` path is not used |
| **Hard** | Permission preset `workspace-write` | **The system pops an approval dialog on every file write** — the model cannot bypass it |

Why the hard layer matters: prompt-level rules depend on model compliance. In practice, models (e.g., deepseek-v4-flash) have been observed ignoring the rule and writing files directly when given an explicit user instruction. **System-level approval is the only reliable guarantee.**

---

## What's Inside

| File | Destination | Purpose |
|---|---|---|
| `templates/AGENTS.md` | `~/.dsh/AGENTS.md` | Global rules (primary carrier, live-loaded) |
| `templates/standard-rules-agent.cordis.yml` | `~/.dsh/.agent-presets/standard-rules/agent.cordis.yml` | Custom preset with rules embedded |
| `templates/cordis.patch.yml` | `~/.dsh/profiles/web/cordis.patch.yml` | Points the default preset at `standard-rules` |
| `templates/settings.yaml` | `~/.dsh/settings.yaml` | Permission preset `workspace-write` |

---

## The Rules (Full Text)

```markdown
## File & Code Change Confirmation

1. Before any file creation, modification, or deletion (code, config, scripts, docs),
   explain: which file, what exactly changes, and why. Then stop and wait for the
   user's explicit approval before executing.
2. A user's "request to perform an action" is itself NOT consent. Even if the user
   directly says "create file X" or "modify Y", first present the concrete change plan
   and wait for confirmation. Only an explicit waiver in the same message
   ("no need to confirm", "just do it") allows immediate execution.
3. Batch confirmation: list the complete change inventory and confirm once for
   multi-file tasks. Any change outside the confirmed inventory requires re-confirmation.
4. Read-only exemption: no confirmation needed for viewing/searching/analyzing;
   for runtime-produced files (logs, caches, session data); and for temporary
   intermediate artifacts created to complete an already-approved task.
5. When intent is unclear about a change, its impact, or how to revert, ask a
   clarifying question instead of guessing and executing.
6. High-risk operations (deleting files, overwriting existing configs, bulk edits)
   must additionally state the impact scope and whether they are reversible.
7. These rules apply to all sessions; an explicit in-session waiver by the user
   takes precedence.
```

### Design Notes

- **Rule 2 is the heart** — it closes the "request ≠ consent" ambiguity, the biggest source of agents acting prematurely
- **Rule 3** — batch confirmation keeps large tasks from being interrupted per file
- **Rule 4** — read-only exemption keeps daily usage friction-free
- **Rule 5** — ask instead of guess when requirements are unclear
- **Rule 7** — an explicit waiver keeps you efficient when you just want it done

---

## Quick Start (DeepSeek Harness)

### 1. Copy the rules (minimum viable setup)

```bash
cp templates/AGENTS.md ~/.dsh/AGENTS.md
```

Loaded automatically for every new session (and live-reloaded in many cases).

### 2. (Recommended) Enable the system approval gate

Add to `~/.dsh/settings.yaml`:

```yaml
permission:
  defaultPreset: workspace-write
```

Now every file write triggers a **system approval dialog** — execution requires your approval.

### 3. (Optional) Preset-layer rules (defense in depth)

Web sessions mount an agent preset (default: `standard`), and the preset's persona **shadows** the global persona. Rules written only to `AGENTS.md` or the global persona may silently not apply. Copy the official `standard` preset and embed the rules in its persona:

```bash
cp -r <DSH_INSTALL>/config/agent-presets/standard ~/.dsh/.agent-presets/standard-rules
# replace its agent.cordis.yml with templates/standard-rules-agent.cordis.yml
# then append to ~/.dsh/profiles/web/cordis.patch.yml:
#   - id: agent-presets
#     config:
#       default: standard-rules
```

### Verify

Open a new session and send:

> Create a test.txt in the current directory with content "hello world"

- ✅ **Working**: the agent stops, states *which file, what content, why*, and waits for approval
- ❌ **Not working**: it creates the file directly — check that `AGENTS.md` loaded and the preset resolves to `standard-rules`

---

## Portability

| Tool | How to adapt |
|---|---|
| **DeepSeek Harness** | Use this repo's templates (AGENTS.md + preset + permission) |
| Claude Code / Cursor / others | Put the rule text in their project rule files (`CLAUDE.md`, `.cursorrules`, ...) and use their own permission mechanism |
| Custom coding agents | Add the rules to the system prompt; add a "confirm before write" gate at the tool-execution layer |

The core does not depend on DSH-specific features: **rule text + a file-permission gate** is a generic pattern.

---

## Security Notes

- **Never commit** `~/.dsh/.credentials.yaml` (contains API keys) to any repository
- Do not commit `sessions/` (conversation history) or `storages/` (personal data)
- This repo's `.gitignore` excludes them by default
- Before sharing your `settings.yaml`, remove personal routing config such as `agent-default-model`

---

## Known Limitations

- Soft layers depend on model compliance — behavior varies across models and contexts. **The hard layer (permission approval) is the reliable guarantee.**
- With `workspace-write`, writing outside the workspace directory is rejected (or gated by approval); adjust the permission preset as needed.
- Persona shadowing by agent presets is DSH-specific behavior; other tools do not have this issue.

## License

MIT
