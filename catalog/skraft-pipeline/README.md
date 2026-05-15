# skraft-pipeline

A six-workflow SDLC pipeline: **Discover → Discuss → Design → Distill → Deliver → Review**.  
Each role is a separate gh-aw workflow. They coordinate by **dispatching the next workflow** via
gh-aw's `dispatch-workflow` safe-output, passing typed inputs (issue number, story type,
iteration counter, optional PR number).

> **Status**: active — workflows are compiled and deployed in `.github/workflows/`.  
> VS Code interactive agents mirror the same pipeline in `.github/agents/`.

---

## See it in action

<!-- TODO: link to a playground PR once available -->
_Coming soon_ — a complete discover → discuss → design → distill → implement → review run on a
sample feature. The issue thread will show all six agents' comments, every workflow-run link, and
the reviewer's approve verdict.

---

## When to use this

You want **multiple specialized agents** (not one mega-prompt) collaborating on a task with full
SDLC traceability:

- Backlog items are triaged and prioritised before being planned
- Stories are refined with acceptance criteria before architecture is designed
- Implementation only starts after BDD scenarios and interface contracts are agreed
- Every output lives in `.skraft/sdlc/` — a git-backed audit trail
- Human override is possible at any stage

---

## When NOT to use this

- The task is a one-shot fix that doesn't need planning or design (use a single workflow instead).
- You only need code review without the full SDLC cycle.
- You need synchronous end-to-end in a single run (chain steps inside one workflow).

---

## The handoff model

```
   label: sdlc  (the only label a human adds to a fresh issue)
          │
          ▼
   ┌─────────────────────┐   dispatch (issue_number, story_type)
   │ backlog-discoverer  │──────────────────────────────────────────────────┐
   │                     │   detects story_type from type labels:           │
   │                     │   type/tech-debt|infra|refactoring → technical   │
   └─────────────────────┘   other → functional                            │
                                                                            ▼
   ┌──────────────────┐   dispatch (issue_number, story_type)
   │ backlog-planner  │──────────────────────────────────────────────────┐
   └──────────────────┘                                                  │
                                                                         ▼
   ┌─────────────────────┐   dispatch (issue_number, story_type)
   │ solution-architect  │────────────────────────────────────────────┐
   └─────────────────────┘                                            │
                                                                      ▼
   ┌──────────────────────┐   dispatch (issue_number, story_type, iteration=1)
   │ acceptance-designer  │──────────────────────────────────────────────────┐
   │                      │   functional → Gherkin + test-plan + impl-plan   │
   │                      │   technical  → impl-plan only (no .feature file) │
   └──────────────────────┘                                                  │
                                                                             ▼
   ┌──────────────────────┐   dispatch (issue_number, story_type, iteration, pr_number?)
   │  software-engineer   │────────────────────────────────────────────────────────────┐
   └──────────────────────┘                                                            │
                                                                                       ▼
   ┌──────────────────────────┐  approve  ► state:done    (human merges the PR)
   │ software-engineer-       │  block    ► state:blocked  (human resolves)
   │   reviewer               │  kickback ► dispatch software-engineer (
   └──────────────────────────┘            issue_number, story_type, pr_number, iteration+1)
```

`state:*` labels (`plan-needed`, `impl-needed`, `review-needed`, `done`, `blocked`) are
**cosmetic breadcrumbs for humans** — they let the GitHub UI show pipeline progress at a glance.
They do **not** drive control flow; `dispatch-workflow` safe-outputs do.

`story_type` (`functional` | `technical`) is detected once by `backlog-discoverer` and
**propagated unchanged** through every dispatch. It controls whether `acceptance-designer`
produces Gherkin scenarios (functional only).

---

## The comment contract

Agents communicate their work product via fenced HTML-comment blocks, which downstream agents
grep out of the issue body + comments. Never rely on prose ordering.

```markdown
<!-- skraft:discover iteration=1 -->
## Triage Report
...
<!-- /skraft:discover -->
```

| Section       | Written by                   | Read by                    |
|---------------|------------------------------|----------------------------|
| `discover`    | backlog-discoverer           | backlog-planner            |
| `discuss`     | backlog-planner              | solution-architect         |
| `design`      | solution-architect           | acceptance-designer        |
| `distill`     | acceptance-designer          | software-engineer          |
| `deliver`     | software-engineer            | software-engineer-reviewer |
| `review`      | software-engineer-reviewer   | *(human or kickback)*      |

Each section carries the `iteration` counter at the time it was produced. When any agent sees
`iteration > 3` as its input it flips the issue to `state:blocked` and stops.

---

## Files

| File | Trigger | Dispatches next |
|------|---------|-----------------|
| `backlog-discoverer.md` | `issues.labeled` with `sdlc` | `backlog-planner` (issue_number, **story_type**) |
| `backlog-planner.md` | `workflow_dispatch` (issue_number, story_type) | `solution-architect` (issue_number, **story_type**) |
| `solution-architect.md` | `workflow_dispatch` (issue_number, story_type) | `acceptance-designer` (issue_number, **story_type**, iteration=1) |
| `acceptance-designer.md` | `workflow_dispatch` (issue_number, story_type, iteration) | `software-engineer` (issue_number, **story_type**, iteration) |
| `software-engineer.md` | `workflow_dispatch` (issue_number, story_type, iteration, pr_number?) | `software-engineer-reviewer` (issue_number, **story_type**, pr_number, iteration) |
| `software-engineer-reviewer.md` | `workflow_dispatch` (pr_number, issue_number, story_type, iteration) | `software-engineer` on kickback (iteration+1), else nothing |

VS Code interactive agents with identical logic live in [`.github/agents/`](../../.github/agents/).

---

## Install

The workflows are already compiled and deployed in `.github/workflows/`. To install into a
**fresh target repo**, copy the six `.md` files and recompile:

```bash
for f in backlog-discoverer backlog-planner solution-architect \
          acceptance-designer software-engineer software-engineer-reviewer; do
  cp catalog/skraft-pipeline/$f.md .github/workflows/
  gh aw compile $f
done
```

Then create the labels (see Prerequisites below) and set the `COPILOT_GITHUB_TOKEN` secret:

```bash
gh aw secrets set COPILOT_GITHUB_TOKEN --value "<your-pat>"
```

<details>
<summary>Install from catalog via <code>gh aw add</code> (advanced)</summary>

```bash
gh aw add SebastienDegodez/agentic-project-demo/catalog/skraft-pipeline/backlog-discoverer.md@main
gh aw add SebastienDegodez/agentic-project-demo/catalog/skraft-pipeline/backlog-planner.md@main
gh aw add SebastienDegodez/agentic-project-demo/catalog/skraft-pipeline/solution-architect.md@main
gh aw add SebastienDegodez/agentic-project-demo/catalog/skraft-pipeline/acceptance-designer.md@main
gh aw add SebastienDegodez/agentic-project-demo/catalog/skraft-pipeline/software-engineer.md@main
gh aw add SebastienDegodez/agentic-project-demo/catalog/skraft-pipeline/software-engineer-reviewer.md@main
```
</details>

---

## Prerequisites in the target repo

- Repo Actions settings: **Read and write permissions** + **Allow GitHub Actions to create and
  approve pull requests**.
- Secret `COPILOT_GITHUB_TOKEN` — fine-grained PAT with `Account permissions → Copilot Requests: Read`.
- The following labels:

| Label | Color | Purpose |
|-------|-------|---------|
| `sdlc` | `#0052cc` | Kicks off the pipeline |
| `state:plan-needed` | `#e4e669` | Discoverer finished |
| `state:design-needed` | `#e4e669` | Planner finished |
| `state:distill-needed` | `#e4e669` | Architect finished |
| `state:impl-needed` | `#e4e669` | Acceptance designer finished |
| `state:review-needed` | `#e4e669` | Implementer finished |
| `state:done` | `#0e8a16` | Reviewer approved |
| `state:blocked` | `#b60205` | Pipeline halted — human needed |
| `type/feature` | `#0075ca` | Functional story (Gherkin produced) |
| `type/bug` | `#d73a4a` | Functional story (Gherkin produced) |
| `type/tech-debt` | `#e4e669` | Technical story (no Gherkin) |
| `type/infra` | `#e4e669` | Technical story (no Gherkin) |
| `type/refactoring` | `#e4e669` | Technical story (no Gherkin) |

```bash
gh label create sdlc --color 0052cc --description "Kicks off the skraft SDLC pipeline"
gh label create "state:plan-needed"    --color e4e669
gh label create "state:design-needed"  --color e4e669
gh label create "state:distill-needed" --color e4e669
gh label create "state:impl-needed"    --color e4e669
gh label create "state:review-needed"  --color e4e669
gh label create "state:done"           --color 0e8a16
gh label create "state:blocked"        --color b60205
gh label create "type/feature"         --color 0075ca
gh label create "type/bug"             --color d73a4a
gh label create "type/tech-debt"       --color e4e669
gh label create "type/infra"           --color e4e669
gh label create "type/refactoring"     --color e4e669
```

---

## Kicking off a task

1. Open an issue describing the feature or bug.
2. Add the single label **`sdlc`**.
3. Watch the thread. Each agent posts its work product as a comment; the software-engineer opens
   a draft PR that closes the issue when merged.
4. **Human override at any time**: add `state:blocked` to halt, edit a comment to steer the next
   agent, or manually `gh workflow run` a specific role to retry a stuck stage.
5. **Retrying a blocked task**: clear `state:blocked`, then re-add `sdlc`.

---

## Limits and gotchas

- **Concurrency**: each workflow uses `concurrency: group: skraft-issue-${issue_number}` so only
  one role runs at a time per issue.
- **Max iterations**: default 3 (reviewer kickback → implementer). The counter lives on the
  `iteration` input passed through the dispatch chain, bumped exclusively by the reviewer on
  kickback.
- **Input propagation**: every downstream workflow must fail loudly if required
  `workflow_dispatch` inputs are missing. Do not rely on label search as a fallback.
- **Cost**: a full pipeline run can easily spend 6× the tokens of a monolithic workflow. Set
  `timeout-minutes` conservatively and monitor the first few runs.
- **No auto-merge**: the reviewer approves but never merges. Humans merge.
- **Dispatch visibility**: each `dispatch-workflow` call shows up as a new run in the Actions
  tab, linked to the upstream run. Makes the chain visible.

---

## Agent roles quick-reference

| Agent | Phase | Skill(s) | Output |
|-------|-------|----------|--------|
| `backlog-discoverer` | DISCOVER | `issue-triage`, `github-search-protocol` | Triage report + sprint proposal + **story_type** |
| `backlog-planner` | DISCUSS | `issue-refinement`, `sprint-planning` | Refined stories + AC drafts |
| `solution-architect` | DESIGN | `architecture-patterns`, `architecture-decisions` | ADRs + event models + contracts |
| `acceptance-designer` | DISTILL | `bdd-methodology`, `test-design-mandates` | **functional**: Gherkin + test-plan + impl-plan / **technical**: impl-plan only |
| `software-engineer` | DELIVER | `outside-in-tdd`, `clean-architecture-testing` | Code + tests via Outside-In TDD |
| `software-engineer-reviewer` | DELIVER | `craft-discipline`, `mutation-testing` | Review verdict (approve / kickback / block) |
