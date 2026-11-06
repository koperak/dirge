# dirge playbook: main software-development scenarios

Distilled from the dirge docs (`README.md`, `docs/spec-workflow.md`, `docs/prompts.md`, `docs/agent-loop.md`, `docs/issues.md`, `docs/skills.md`, `docs/agents.md`, `docs/verification-discipline.md`, `docs/features.md`, `prompts/*.md`, `skills/*`).

## How to read the examples

There are two kinds of input, and the examples mark them differently:

- **Slash commands** (`/prompt plan`, `/spec`) are typed by you in the dirge input box.
- **Tool calls** (`spec(action: "propose", ...)`) are made by the agent. You normally don't type them. You ask in plain language, for example `Record the design decisions for add-dark-mode`, and the agent calls the tool. The call syntax is shown so you know what the agent does and how to check it with `/spec`.

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

Setup and launch:

```bash
export OPENROUTER_API_KEY="..."      # or another provider's key
dirge                                # interactive session (prompt: code)
dirge -p "Explain this project"      # one-shot
dirge -c                             # continue last session
dirge --session my-feature           # resume or create a session with a stable id
```

---

## 1. Requirement gathering

**Use case:** you have a vague idea, such as "add dark mode" or "users want export".

**Step 1. Clarify in design-only mode.** `brainstorm` asks one question at a time, proposes 2-3 approaches with trade-offs, and writes no code until you approve.

```text
/prompt brainstorm
I want users to be able to switch the app to a dark theme.
```

**Step 2. Check what already exists** so you don't duplicate requirements:

```text
spec(action: "specs")                          # list capabilities
spec(action: "specs", capability: "theming")   # read its requirements
```

**Step 3. Capture the agreement as a change.** Switch to `code` and ask the agent to record it:

```text
/prompt code
Create a spec change for the dark mode design we just agreed on.
```

The agent calls:

```text
spec(action: "propose", slug: "add-dark-mode",
     why: "users want a low-light option",
     what: "theme toggle + persisted preference")

spec(action: "add_delta", slug: "add-dark-mode",
     op: "added", capability: "theming",
     requirement: "User can toggle theme",
     text: "The system SHALL persist the chosen theme across sessions.",
     scenarios: [{name: "toggle",
                  when_then: "WHEN the user flips the toggle THEN the theme switches and is saved"}])
```

Rules:
- Use SHALL/MUST wording, and give every added or modified requirement at least one WHEN/THEN scenario.
- Other `op` values:

```text
# modified: send the FULL new text and scenarios, not a diff
spec(action: "add_delta", slug: "add-dark-mode", op: "modified",
     capability: "theming", requirement: "User can toggle theme",
     text: "The system SHALL persist the theme and follow the OS setting by default.",
     scenarios: [{name: "os-default", when_then: "WHEN no choice is saved THEN the OS theme is used"}])

# removed: say why and where users go instead
spec(action: "add_delta", slug: "add-dark-mode", op: "removed",
     capability: "theming", requirement: "Legacy high-contrast toggle",
     reason: "superseded by dark mode", migration: "use the theme toggle")

# renamed: name change only
spec(action: "add_delta", slug: "add-dark-mode", op: "renamed",
     capability: "theming", requirement: "User can toggle theme",
     rename_to: "User can switch theme")
```

**Step 4. Review as a human** (read-only):

```text
/spec                       # all changes with task progress
/spec add-dark-mode         # proposal, deltas, tasks
/spec specs theming         # living requirements + scenarios
```

**Step 5. File follow-ups** that should outlive the session:

```text
Create an issue "Respect prefers-color-scheme" with high priority.
/issues list
```

**Tip:** `/prompt ask` is for read-only Q&A about an existing codebase:

```text
/prompt ask
How does authentication work today?
```

---

## 2. Architecture definition

**Use case:** decide how to build the feature and record why.

**Step 1. Explore read-only.** `plan` and `ask` deny edit, write, bash, spec, etc., so the codebase can't change while you explore.

```text
/prompt plan
Plan how to add dark mode. Explore the current styling approach first.
```

The agent reads with `repo_overview`, `grep`, `glob`, and the semantic tools (`list_symbols`, `find_callers`, ...) and delivers a written plan in chat. It won't implement until you approve. Save the plan yourself if you want it in a file.

**Step 2. Compare approaches.** Ask for explicit trade-offs:

```text
Give me 3 approaches (CSS variables, a theme provider, per-component styles)
with trade-offs and your recommendation.
```

**Step 3. Record the decision in the change.** This is the field that, on archive, is folded into project memory together with `why`.

```text
spec(action: "set_field", slug: "add-dark-mode", field: "design",
     value: "Decision: CSS custom properties switched via a data-theme attribute on <html>.
             Why: no re-render cost; works with existing components.
             Rejected: per-component style props (touches ~120 files);
                       a JS theme provider (extra runtime, flash of wrong theme on load).
             Trade-off: IE11 is unsupported.")
```

Other editable fields are `title`, `why` and `what`:

```text
spec(action: "set_field", slug: "add-dark-mode", field: "what",
     value: "theme toggle + persisted preference + OS default")
```

The active change (why, what, design, deltas, tasks) is injected into the system prompt of every new or resumed session, so later sessions know the decision without being told.

**Step 4. Store cross-cutting conventions and pitfalls** in memory:

```text
Remember: all colors must come from CSS variables in theme.css, never hard-coded hex values.
/memory                  # inspect what is stored
/memory edit             # edit the whole store in $EDITOR
/memory review           # approve queued writes (when memory.confirm_writes is on)
```

**Step 5. Optional: a stronger model for design.** Create `.dirge/agents/architect.md`:

```markdown
---
model: anthropic/claude-opus
prompt: plan
---
You are a software architect. Prefer small, reversible decisions and state trade-offs.
```

Then switch to it:

```text
/agent architect
/agent off
```

The exact frontmatter keys are in `docs/agents.md`. Check them before relying on the sample above.

---

## 3. Iterative development

**Use case:** implement the agreed change in small, verified steps.

**Step 1. Break down the work and track it.**

```text
spec(action: "add_task", slug: "add-dark-mode", text: "Add theme CSS variables")
spec(action: "add_task", slug: "add-dark-mode", text: "Add toggle component")
spec(action: "add_task", slug: "add-dark-mode", text: "Persist choice in localStorage")

spec(action: "set_task", task_id: 1, status: "in_progress")   # pending | in_progress | done | blocked
spec(action: "set_task", task_id: 1, status: "done")
```

**Step 2. Pick the mode that fits the size of the task.**

Small or medium. `code` mode enforces TDD: understand → explore → failing test → minimal implementation → run → verify → review. It stops and asks if the task would take more than 30 minutes.

```text
/prompt code
Implement task 2 of add-dark-mode. Write a failing test first.
```

Large or risky. `/plan` runs four context-isolated phases: Explore (read-only) → Plan → Implement → Review (a write-disabled reviewer runs the code and returns DONE or NEEDS_FIX, up to `phased_workflow_max_review_cycles`, default 2).

```jsonc
// config.json: enable the workflow and the approve/edit/cancel gate
{
  "phased_workflow_enabled": true,
  "phased_workflow_plan_approval": true
}
```

```text
/plan implement add-dark-mode as specified in the active spec change
# at the gate: [a]pprove / [e]dit (give feedback, the plan is regenerated) / [c]ancel
```

Grinding to green:

```text
/loop make the full test suite pass
```

**Step 3. Keep a backlog and a checklist.**

```text
Add an issue "Dark mode: chart colors" (normal priority)     # persistent, shown every turn
/issues list
/issues 3
/issues search chart
```

`write_todo_list` is the ephemeral, in-session checklist the agent maintains on its own.

**Step 4. Isolate experiments with a worktree.**

```text
/worktree dark-mode-spike
... work ...
/wt-merge main
/wt-exit
```

**Step 5. Control cost and safety.**

```text
/mode restrictive            # standard | restrictive | accept | yolo
/allow add "cargo test"      # pre-approve a command for this session (check /allow list for the exact form)
/allow list
/why bash "rm -rf build"     # dry-run a permission decision and print the policy trace
/effort high                 # reasoning effort for the next turn
/model                       # show or switch model
/compress                    # force a context summarization pass
```

**Step 6. Recover.**

```text
/undo                        # last exchange
Esc Esc                      # rewind picker: restores files AND conversation
/tree                        # session tree
/fork                        # branch from the last user message
/retry                       # retry last prompt
```

**Step 7. Bugs and cleanup.**

```text
/prompt debug
The toggle resets after reload. Find the root cause before proposing a fix.

/prompt simplify
Simplify theme.ts without changing behavior.
```

---

## 4. Testing and verification

**Use case:** make "done" mean verified.

**Step 1. Set a project gate** so a green result must come from your real command. Matching is by program + subcommand, and flags and env assignments are ignored.

```jsonc
// config.json
{
  "verification_command": "cargo fmt --all --check && cargo clippy --all-targets -- -D warnings && cargo test"
}
```

Related gates (`claim_gate`, `source_gate`, `completeness_gate`) are in `docs/verification-discipline.md` and `docs/config.md`.

**Step 2. Test first, in `code` mode.**

```text
Add tests for the theme toggle: default theme, saved theme, and an unknown saved value.
Show the failing output before implementing.
```

Ask for boundary cases explicitly:

```text
For every threshold in the change, add a case at, just below, and just above it.
```

Check the real exit status, not a pipeline's:

```bash
cargo test | tail      # reports tail's status, hides a failing test
cargo test             # correct
cargo test || true     # always "passes": avoid
```

**Step 3. Review the diff.**

```text
/code-review                   # two-pass, severity-ranked; needs critic_provider
/prompt review                 # read-only review in chat: Approve / Needs Changes / Reject
Review the diff against main.
/prompt review-security        # finds exploitable vulnerabilities
```

**Step 4. Route review to another model.** Pick one of two ways:

```jsonc
// config.json: a role pointing at a provider alias
{
  "critic_provider": "cheap-reviewer",
  "providers": { "cheap-reviewer": { /* provider settings, see docs/config.md */ } }
}
```

```text
/agent reviewer               # a profile defined in .dirge/agents/reviewer.md
```

**Step 5. Autonomous run with a stop condition** (goal gate, judged by `critic_provider`):

```bash
dirge --goal "all tests pass and changes committed" -p "Implement add-dark-mode"
```

**Step 6. Before merging, ask for a verification report:**

```text
List what you verified (commands and results) and the residual risks.
```

LSP diagnostics appear inline after edits (`docs/lsp.md`). Command hooks can run linters automatically (`docs/hooks.md`).

---

## 5. Documentation

**Use case:** keep docs and knowledge current without extra effort.

**Step 1. Close the change and update the living specs.** Archive is refused while any task is open, and a change can't be archived twice.

```text
spec(action: "status", slug: "add-dark-mode")     # confirm every task is done
spec(action: "archive", slug: "add-dark-mode")    # folds deltas into living specs, writes why + design to memory
```

```text
/spec specs                  # list capabilities
/spec specs theming          # the updated requirements + scenarios
/memory                      # the rationale recorded from why + design
```

**Step 2. Update project docs alongside the code.**

```text
/prompt code
Update README.md and CHANGELOG.md for dark mode. Document the data-theme attribute.

/prompt review
Check that the docs match the diff and flag anything stale.
```

**Step 3. Capture reusable playbooks.**

```text
/learn ~/projects/acme-sdk, focus on auth + pagination
/learn https://docs.example.com/api/quickstart
/learn            # bare: distill the workflow from this conversation
```

The result is a `SKILL.md`, which must include a `## Verification` section with one command that proves it works. You can also write one by hand in `.dirge/skills/<name>/SKILL.md`:

```markdown
---
name: release-notes
description: Use when preparing release notes from merged PRs
---
# Release notes
1. List merged PRs since the last tag: `git log --merges <last-tag>..HEAD`.
2. Group by feature, fix, and docs.

## Verification
`grep -c '^## ' CHANGELOG.md` shows the new section was added.
```

Bundled skills are not installed automatically:

```bash
cp -r skills/systematic-debugging .dirge/skills/    # per project
cp -r skills/spec-driven-workflow ~/.dirge/skills/  # global
```

**Step 4. Custom prompts.**

```text
/prompt write-prompt
Create a prompt for writing release notes.
```

A prompt is a markdown file with optional frontmatter that restricts tools:

```markdown
---
deny_tools: [edit, write, apply_patch, bash, webfetch]
description: Read-only mode for answering questions
---
You are in Q&A mode. Answer from the code; never modify files.
```

**Step 5. Share dirge with other tools** by running it as an MCP server:

```bash
claude mcp add dirge -- dirge mcp
```

---

## End-to-end example

```text
/prompt brainstorm                      # clarify, choose approach, approve design
/prompt code
  spec propose / add_delta / set_field design / add_task
/spec add-dark-mode                     # human check
/plan implement add-dark-mode           # or work in code mode, task by task
/code-review                            # severity-ranked findings
/prompt review-security                 # optional
  spec set_task ... done ; spec archive add-dark-mode
/learn                                  # capture the workflow as a skill
```

## Caveats

- Everything the agent tracks (specs, issues, memory) lives in `.dirge/sessions/state.db`. Check whether your VCS ignores `.dirge/` if you don't want it shared.
- `/plan`, `/loop` and `/code-review` depend on config or build features (`phased_workflow_enabled`, the `loop` feature, `critic_provider`). Check `docs/config.md` before relying on them.
- Syntax in the `spec(...)` examples comes from `skills/spec-driven-workflow/SKILL.md`. The agent-profile frontmatter keys, the `/allow add` argument form and the config snippets are illustrative. Check them against `docs/agents.md`, `docs/permissions.md` and `docs/config.md`.
