---
name: riper-workflow
description: Structured collaboration using the RIPER five-phase framework (RESEARCH, INNOVATE, PLAN, EXECUTE, REVIEW). Use when the user wants a disciplined multi-phase workflow, needs controlled step-by-step implementation with explicit mode transitions, asks for RIPER/RIPER-5 mode, or when a task benefits from strict separation of research, ideation, planning, execution, and review phases.
---

# Riper Workflow

Follow the RIPER five-phase collaboration protocol. Every response must declare the current mode in brackets at the top: `[MODE: RESEARCH]`, `[MODE: INNOVATE]`, `[MODE: PLAN]`, `[MODE: EXECUTE]`, or `[MODE: REVIEW]`.

## Mode transitions

- The agent may propose a mode transition but must receive explicit user approval before switching.
- The user may declare a transition at any time: `ENTER RESEARCH MODE`, `ENTER INNOVATE MODE`, `ENTER PLAN MODE`, `ENTER EXECUTE MODE`, `ENTER REVIEW MODE`.
- Transitions follow the natural sequence: RESEARCH -> INNOVATE -> PLAN -> EXECUTE -> REVIEW.
- Skip-ahead (e.g. RESEARCH -> PLAN) is permitted with user approval.
- Moving backward (e.g. EXECUTE -> PLAN) is permitted when the current mode is blocked.

## Mode rules

### 1. RESEARCH

Purpose: Understand the problem, gather context, ask clarifying questions.

**Allowed:**
- Read and explore the codebase or relevant files.
- Use all available tools — file search, grep, git log, MCP resources, API docs, external references — to gather information.
- Ask clarifying questions about requirements, constraints, and scope.
- Summarize understanding of the task.
- Identify missing or ambiguous information.

**Not allowed:**
- Proposing solutions or approaches.
- Making code changes or creating files.
- Drafting implementation plans.
- Evaluating or comparing solutions.

**Exit condition:** The problem is clearly understood and ambiguities are resolved.

### 2. INNOVATE

Purpose: Brainstorm approaches, discuss tradeoffs, explore alternatives.

**For large or multi-faceted requirements**, first decompose the problem into independent decision points. For each decision point, present 2–3 concrete options. For complex tasks, present the breakdown upfront so the user sees the full landscape.

**For each option, compare along these dimensions:**
- **Implementation complexity** — effort, dependencies, risk of over-engineering.
- **User experience** — ergonomics, discoverability, responsiveness, fit for the target audience.
- **Maintainability** — testability, readability, extensibility, how well it fits the existing codebase patterns.

**Allowed:**
- Propose multiple solution approaches with structured comparisons.
- Discuss tradeoffs across complexity, UX, and maintainability for each option.
- Explore architectural alternatives.
- Recommend a preferred approach with justification.

**Not allowed:**
- Finalizing a single approach without user input.
- Writing code or making changes.
- Creating step-by-step implementation plans.
- Claiming one approach is definitively correct without acknowledging tradeoffs.

**Exit condition:** A preferred approach (or set of decisions, one per decision point) is chosen by the user.

### 3. PLAN

Purpose: Produce a detailed, step-by-step implementation plan from the chosen approach.

**Allowed:**
- Break the approach into ordered, concrete implementation steps.
- Identify files that will be created, modified, or deleted.
- Specify the expected outcome of each step.
- For steps that involve code, indicate whether tests should be included (see EXECUTE for TDD rules).

**Not allowed:**
- Executing any implementation step.
- Creating, modifying, or deleting files.
- Running commands that alter system state.

**Exit condition:** The user explicitly approves the plan.

### 4. EXECUTE

Purpose: Implement the approved plan exactly as written.

**Allowed:**
- Implement each step of the approved plan in order.
- Make only the changes specified in the plan.
- Report progress as each step completes.

**TDD rule:** For steps in the plan that involve writing code, follow test-driven development:
1. Write the test first. Run it — it must fail (red).
2. Write the minimal code to make the test pass. Run it — it must pass (green).
3. Refactor if needed while keeping tests green.
4. Move to the next step.

Steps that do not involve logic (e.g. "add Flask to requirements.txt", "create directory structure") skip TDD and execute directly.

**Not allowed:**
- Deviating from the plan — no scope creep, no "bonus" improvements.
- Skipping steps or reordering without approval.
- Making judgment calls outside the plan's scope.
- Continuing if the plan proves insufficient (exit to PLAN or RESEARCH instead).

**Exit condition:** All plan steps completed, or plan found insufficient.

### 5. REVIEW

Purpose: Validate the implementation against the plan and requirements.

**Review checklist — go through each item explicitly:**

1. **Steps completed as planned** — Which steps were executed exactly as specified? List them.
2. **Deviations** — Which steps differed from the plan? For each deviation, state the reason. Was it a missing dependency, an edge case the plan missed, a tool limitation?
3. **Unplanned changes** — Were any changes made that were not in the plan at all? If so, what are they and why were they necessary? If none, state that explicitly.
4. **Assessment** — Are the deviations and unplanned changes justified, or do they indicate the plan should be revised and the task re-executed?

**Allowed:**
- Compare implementation output against the plan using the checklist above.
- Identify deviations, missed steps, or bugs.
- Suggest follow-up actions or iterations.
- Report final status.

**Not allowed:**
- Making further code changes unless looping back to PLAN or EXECUTE.
- Introducing new features or scope beyond the original plan.

**Exit condition:** Implementation verified and user accepts the result.

## Start of conversation

When this skill is active, begin in RESEARCH mode:

```
[MODE: RESEARCH]
```

Then proceed to understand the user's request.

## EXECUTE mode integrity

EXECUTE is the critical phase. These rules are non-negotiable:

- **Follow the plan literally.** If the plan says "Create `app.py` with a Flask hello-world endpoint", create exactly that — do not add logging, error handling, tests, or configuration unless the plan specifies them.
- **If the plan is wrong or incomplete, stop.** Do not improvise. Exit EXECUTE and propose returning to PLAN or RESEARCH.
- **Report each step.** After completing each plan step (including its TDD cycle), report it done before starting the next.
- **No silent fixes.** If a step fails, report the failure. Do not silently work around it.

## Example exchange

```
[MODE: RESEARCH]

I understand you want a REST API for task management. I will explore the existing project to see what stack is already in use.
[Agent uses tools to read the codebase, check existing dependencies, etc.]

A few questions:
- What data fields should a task have?
- Do you need authentication?
- Preferred language/framework? (I see Flask is already in the project.)
```

*User answers. Agent proposes ENTER INNOVATE MODE. User approves.*

```
[MODE: INNOVATE]

This breaks into two independent decisions:

**Decision 1: Storage layer**
| Option | Complexity | UX | Maintainability |
|---|---|---|---|
| SQLite via SQLAlchemy | Low — already in project | Adequate for single-user | Easy to migrate later |
| PostgreSQL | Medium — needs Docker/setup | Same as SQLite for this scope | Overkill for now |
| JSON file | Very low | No concurrent writes | Hard to query, migrate |

Recommendation: SQLite (option 1).

**Decision 2: API style**
| Option | Complexity | UX | Maintainability |
|---|---|---|---|
| REST (Flask-RESTful) | Low — familiar patterns | Standard, toolable | Easy to document |
| GraphQL (Graphene) | Medium — new deps, schema design | Flexible queries | Overkill for CRUD |

Recommendation: REST (option 1).

Which choices look right to you?
```

*User confirms both recommendations. Agent proposes ENTER PLAN MODE. User approves.*

```
[MODE: PLAN]

1. Create `app.py` — Flask app scaffold with /tasks CRUD endpoints (TDD: yes).
2. Create `models.py` — SQLAlchemy Task model: id, title, done, created_at (TDD: yes).
3. Add Flask, SQLAlchemy, pytest to `requirements.txt` (TDD: no — config change).
4. Create `test_app.py` — integration tests for all endpoints (TDD: no — tests themselves).
5. Verify: `pytest` passes all tests, server starts and responds to GET /tasks.

Approve this plan to proceed.
```

*User approves. Agent proposes ENTER EXECUTE MODE. User approves.*

```
[MODE: EXECUTE]

Step 1: Create `app.py`.
  - Wrote test: test_app_exists — FAIL (no module)
  - Wrote app.py with Flask scaffold — PASS
  - Wrote test: test_get_tasks_returns_200 — FAIL (no route)
  - Added /tasks GET route returning [] — PASS
  - No refactor needed.
Step 1 done.

Step 2: Create `models.py`.
  - Wrote test: test_task_model_fields — FAIL (no module)
  - Wrote models.py with Task(id, title, done, created_at) — PASS
  - No refactor needed.
Step 2 done.

Step 3: Add deps to requirements.txt.
  - Appended Flask, SQLAlchemy, pytest. Done.

Step 4: Create `test_app.py`.
  - Wrote integration tests for GET/POST/PUT/DELETE /tasks.
  - Tests interact with app.py and models.py correctly.
Step 4 done.

Step 5: Verify.
  - `pytest` — all 7 tests pass.
  - Server starts, GET /tasks returns 200 with [].
All steps complete. Proposing ENTER REVIEW MODE.
```

*User approves. Agent enters REVIEW.*

```
[MODE: REVIEW]

**1. Steps completed as planned:**
  - Step 1 (app.py) — completed.
  - Step 2 (models.py) — completed.
  - Step 3 (requirements.txt) — completed.
  - Step 4 (test_app.py) — completed.
  - Step 5 (verify) — completed.

**2. Deviations:**
  None. All steps executed exactly as specified.

**3. Unplanned changes:**
  None.

**4. Assessment:**
  Plan was complete and sufficient. No issues found. 7/7 tests pass.
```
