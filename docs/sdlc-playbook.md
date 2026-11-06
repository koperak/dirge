# dirge playbook: main software-development scenarios

Distilled from the dirge docs (`README.md`, `docs/spec-workflow.md`, `docs/prompts.md`, `docs/agent-loop.md`, `docs/issues.md`, `docs/skills.md`, `docs/agents.md`, `docs/verification-discipline.md`, `docs/features.md`, `prompts/*.md`, `skills/*`). Nothing here is committed to the repo.

## 0. Building blocks

| Building block | What it is | Use |
|---|---|---|
| Prompts (`/prompt <name>`) | Persona + tool restrictions enforced by the permission layer | `brainstorm`, `plan`, `ask` (read-only); `code` (TDD, default); `review`, `review-security`, `debug`, `simplify` |
| `spec` tool + `/spec` | SQLite-backed living specs (capability → requirement → WHEN/THEN scenario), changes with deltas and tasks | Requirements and living documentation |
| `issue` tool + `/issues` | Persistent kanban, top open issues injected every turn | Backlog across sessions |
| `write_todo_list` | Ephemeral in-session checklist | Short-term steps |
| `memory` tool + `/memory` | Per-project and global facts and pitfalls, injected into the system prompt | Decisions, conventions, lessons |
| `/plan <task>` | Phased Explore → Plan → Implement → Review workflow (needs `phased_workflow_enabled`) | Complex tasks |
| `/loop [prompt]` | Iterative coding loop (needs the `loop` feature) | Grinding to green |
| `/agent <name>` | Profile = model + prompt + tool policy (`.dirge/agents/<name>.md`) | Cheap model for review, strong model for implementation |
| `/learn`, skills | Turn a workflow or source into a reusable `SKILL.md` | Team playbooks |
| `/mode`, `/allow`, `/why` | Security mode and permissions | Safe autonomy |
| `/worktree`, `/wt-merge`, `/wt-exit` | Git worktree per task | Parallel or isolated work |
| `Esc-Esc` | Rewind files and conversation | Undo a long run |

Setup: `export OPENROUTER_API_KEY=...` (or another provider), then `dirge` (interactive), `dirge -p "..."` (one-shot), `dirge -c` (continue), `dirge --session <id>` (stable id).

---

## 1. Requirement gathering

**Use case:** you have a vague idea, such as "add dark mode" or "users want export".

**Instructions**
1. `/prompt brainstorm`. This mode is design-only. It asks one question at a time, proposes 2-3 approaches with trade-offs, and writes no code until you approve.
2. Answer the clarifying questions: purpose, constraints, success criteria, scope in and out.
3. Switch to `/prompt code` (or use the bundled `spec-driven-workflow` skill). Capture the agreement as a change:
   - `spec propose slug=add-dark-mode why=… what=…`
   - `spec add_delta op=added capability=theming requirement="User can toggle theme" text="The system SHALL persist…" scenarios=[{name, when_then:"WHEN … THEN …"}]`
   - Use SHALL/MUST wording, and give every added or modified requirement at least one WHEN/THEN scenario.
   - Other ops: `modified` (full new text), `removed` (`reason` + `migration`), `renamed` (`rename_to`).
4. Before proposing, check `spec specs` / `spec specs capability=<name>` so you don't duplicate existing requirements.
5. Review with `/spec <slug>`. It is read-only for humans.
6. File follow-ups as issues: ask the agent to create them with the `issue` tool, then check `/issues`.

**Tip:** `/prompt ask` is for read-only Q&A about an existing codebase, for example "how does auth work today?".

---

## 2. Architecture definition

**Use case:** decide how to build the feature and record why.

**Instructions**
1. Explore read-only first. Either `/prompt plan` (edit, write, bash, spec, etc. are denied) or `/prompt ask`. Useful agent tools: `repo_overview`, `grep`, `glob`, semantic symbol/caller/callee tools, LSP.
2. In `plan` mode the agent works through discovery, then delivers a written plan in chat. It will not implement until you approve. Save the plan yourself.
3. For a design doc, use `brainstorm`, which writes `docs/design/YYYY-MM-DD-<feature>-design.md` after approval. Skip that step if you don't want files in the repo.
4. Record decisions in the change: `spec set_field slug=… field=design value="<decisions, trade-offs, rejected alternatives>"`. On archive, why + design are folded into project memory, so the rationale outlives the change.
5. Use `memory` (`/memory`, `/memory edit`) for cross-cutting conventions and pitfalls. Set `memory.confirm_writes` to review what gets stored (`/memory review`).
6. Optionally, use a stronger model for design with `/agent <architect>`, defined in `.dirge/agents/architect.md` (frontmatter model + prompt + tool policy).

---

## 3. Iterative development

**Use case:** implement the agreed change in small, verified steps.

**Instructions**
1. Break the work down: `spec add_task slug=… text=…`. Track status with `spec set_task task_id=N status=in_progress|done|blocked`. The active change is injected into every new or resumed session, so context survives restarts.
2. Pick the mode that fits the size of the task:
   - **Small or medium:** `/prompt code`. It enforces TDD: understand → explore → failing test → minimal implementation → run → verify (linters, types, full suite) → review. It stops and asks if the task would take more than 30 minutes.
   - **Large or risky:** `/plan <request>`. Four context-isolated phases: Explore (read-only) → Plan (optional approve/edit/cancel gate via `phased_workflow_plan_approval`) → Implement (streamed, normal permissions) → Review (write-disabled reviewer runs the code and returns DONE or NEEDS_FIX). It loops up to `phased_workflow_max_review_cycles` (default 2). Anything that isn't a parseable DONE counts as not done.
   - **Grinding to green:** `/loop <prompt>`.
3. Keep the backlog in `issue` (persistent) and the checklist in `write_todo_list` (ephemeral).
4. Isolate experiments: `/worktree <name>`, then `/wt-merge`, then `/wt-exit`.
5. Control cost and safety:
   - `/mode standard|restrictive|accept|yolo`.
   - `/allow add …` to pre-approve commands.
   - `/why <tool> <input>` to debug a permission decision.
   - `/effort`, `/model`, `/compress` to tune reasoning and context.
6. Recover: `/undo` (last exchange), `Esc-Esc` (rewind files and conversation), `/fork` or `/tree` (branch the session), `/retry`.
7. Bugs: `/prompt debug` finds the root cause before proposing a fix. The bundled `systematic-debugging` skill covers the same ground.
8. Cleanup: `/prompt simplify` refactors for clarity without behavior change.

---

## 4. Testing and verification

**Use case:** make "done" mean verified.

**Instructions**
1. Set a project gate in config so a green result must come from your real command: `verification_command` (for example `cargo fmt --all --check && cargo clippy --all-targets -- -D warnings && cargo test`). Matching is by program + subcommand. Check docs/verification-discipline.md for the related `claim_gate`, `source_gate` and `completeness_gate` settings.
2. In `code` mode, ask for tests first (red → green). Check the real exit status, not a pipeline's: `cmd | tail` and `cmd || true` hide failures. Ask for boundary cases at, below and above each threshold.
3. `/code-review` reviews the working-tree diff (two-pass, severity-ranked) and needs `critic_provider` configured. `/prompt review` gives a read-only review in chat with an Approve, Needs Changes or Reject outcome. `/prompt review-security` hunts exploitable vulnerabilities.
4. Route review to a cheaper or different model through `critic_provider` / `review_provider`, or through `/agent reviewer`.
5. Use `/plan` for an independent reviewer that actually runs the code.
6. Optional: a Claude-Code-compatible `hooks` config can run tests or linters automatically on events (docs/hooks.md). LSP diagnostics show up inline (docs/lsp.md).
7. Before merging, ask for a summary of what was verified and the residual risks.

---

## 5. Documentation

**Use case:** keep docs and knowledge current without extra effort.

**Instructions**
1. **Living specs.** When the work is done, close every task (archive is refused while any are open), then run `spec archive slug=…`. This folds the deltas into the living specs in a single transaction and writes the rationale to memory. Browse with `/spec specs` and `/spec specs <capability>`.
2. **Project docs.** In `code` mode, ask the agent to update README, docs or CHANGELOG alongside the code change. Use `/prompt review` to check the docs against the diff.
3. **Reusable playbooks.** `/learn <path|url|description>` (or bare `/learn` after a successful session) creates a `SKILL.md` with a mandatory `## Verification` section. Skills live in `.dirge/skills` or similar, and the discovery directories are in docs/skills.md. A salience system curates them over time.
4. **Knowledge base.** `/memory` shows project and global memory. Keep durable conventions and pitfalls there. Idle sessions trigger an automatic background review that extracts learnings.
5. **Custom prompts and profiles.** Write them with `/prompt write-prompt`. See docs/prompts.md and docs/agents.md for the file formats.
6. **Share with other tools.** `dirge mcp` exposes dirge as an MCP server, for example `claude mcp add dirge -- dirge mcp`.

---

## End-to-end example

```
/prompt brainstorm            # clarify, choose approach, approve design
/prompt code
  spec propose / add_delta / set_field design / add_task
/spec add-dark-mode           # human check
/plan implement add-dark-mode # or just work in code mode, task by task
/code-review                  # severity-ranked findings
/prompt review-security       # optional
  spec set_task … done ; spec archive add-dark-mode
/learn                        # capture the workflow as a skill
```

## Caveats

- Everything the agent tracks (specs, issues, memory) lives in `.dirge/sessions/state.db`. Check whether your VCS ignores `.dirge/` if you don't want it shared. Nothing in this playbook writes to the repo except what you ask the agent to change.
- `/plan`, `/loop` and `/code-review` depend on config or build features (`phased_workflow_enabled`, the `loop` feature, `critic_provider`). Verify against docs/config.md before relying on them.
- I derived this from the docs and did not run dirge. Check the exact option names against docs/config.md.
