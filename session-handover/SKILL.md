---
name: session-handover
description: Generate a `handover.md` file at the project root that summarizes the current agent session, so a new session can continue the work without re-reading the old session's history. Use this skill whenever the user wants to switch to a new session, hand over or hand off a session, create a handover/handoff document, or summarize the current session for continuation. Make sure to trigger when the user says things like "生成 handover", "交接会话", "切换会话", "session handover", "handoff", "switch session", "summarize this session", "I'm running out of context", or otherwise signals that the current session is ending and a new one will pick up the work — even if they don't explicitly say "handover".
---

# Session Handover

Generate a `handover.md` file at the project root that summarizes the current agent session, so a new session can continue the work without re-reading the old session's history.

## Workflow

1. **Detect language**: Determine the primary natural language the user has been using in this session (Chinese or English). Generate the `handover.md` content in that same language. All section headers, descriptions, and content should follow this language.

2. **Review session history**: Look back through the entire current session conversation. Extract and organize information into the 7 sections defined below.

3. **Fill the 7 core sections** (use `template.md` as the structural reference; adapt headings to the chosen language):

   1. **Task Summary (任务概述)** — The user's original goal and what the overall task is about. The current overall phase (e.g., research done, implementing, awaiting verification).
   2. **Done (已完成的工作)** — Steps already completed, files created/modified, and key decisions made with their rationale (why A over B).
   3. **Current State (当前状态)** — The precise point where work was interrupted. Include the exact entry point (e.g., "just finished function X in `foo.ts`, haven't wired up Y yet"). Mention any uncommitted changes if known.
   4. **TODO / Next Steps (待办事项)** — An ordered list of next steps. Mark priority and blocking conditions for each item.
   5. **Key Context (关键上下文)** — Important constraints, user preferences, implicit rules, pitfalls encountered and their solutions (to prevent the new session from repeating mistakes). Include file paths, line numbers, and function references where relevant.
   6. **Verification (验证方式)** — How to confirm the task is complete (lint/test commands, expected behavior).
   7. **Open Questions (开放问题)** — Points that need user clarification but haven't been resolved yet.

4. **Handle existing file**: Before writing, check if `handover.md` already exists at the project root.
   - If it exists, ask the user whether to: overwrite, back up (rename the old one to `handover-{n}.md`), or cancel.
   - If it doesn't exist, write directly.

5. **Write the file**: Output to `handover.md` in the project root (current working directory root). Do not include a "new session prompt" section, git snapshot, or slim mode — keep it focused on the 7 sections only.

6. **Report**: After writing, briefly tell the user the file path and that the new session can read it to continue.

## Notes

- The skill generates content **automatically** from the current session history — do not ask the user to fill in fields.
- Do **not** add a "new session recovery prompt" at the end.
- Do **not** run git commands or include git status/diff snapshots.
- Do **not** offer a slim/compact mode.
- Keep the document concise but complete; avoid dumping raw conversation logs.
- Reference `template.md` for the structural skeleton.
