---
engine: copilot
description: |
  Software-engineer-reviewer agent for the skraft SDLC pipeline.
  Triggered by workflow_dispatch from software-engineer. Audits the PR
  across 4 lenses, renders a structured verdict, and either approves
  or kicks back to software-engineer with iteration+1.

on:
  workflow_dispatch:
    inputs:
      pr_number:
        description: The PR to review.
        required: true
        type: string
      issue_number:
        description: The originating issue.
        required: true
        type: string
      story_type:
        description: functional or technical (propagated from discoverer).
        required: false
        type: string
        default: "functional"
      iteration:
        description: Current iteration in the impl→review loop (1-indexed).
        required: false
        type: string
        default: "1"

concurrency:
  group: skraft-issue-${{ github.event.inputs.issue_number }}
  cancel-in-progress: false

timeout-minutes: 20

permissions: read-all

network:
  allowed:
    - defaults

imports:
  - .github/agents/software-engineer-reviewer.agent.md

tools:
  github:
    toolsets: [default]

safe-outputs:
  threat-detection: false
  add-comment:
    max: 3
    target: "*"
  create-pull-request-review-comment:
    max: 10
  submit-pull-request-review:
    max: 1
    target: ${{ github.event.inputs.pr_number }}
    allowed-events: [APPROVE, COMMENT, REQUEST_CHANGES]
    supersede-older-reviews: true
  add-labels:
    allowed: [state:done, state:impl-needed, state:blocked]
    max: 2
    target: "*"
  remove-labels:
    allowed: [state:review-needed]
    max: 1
    target: "*"
  dispatch-workflow:
    workflows: [software-engineer]
    max: 1
---

# Software-Engineer-Reviewer Agent

**Runtime context:**
- PR: #${{ github.event.inputs.pr_number }}
- Issue: #${{ github.event.inputs.issue_number }}
- Story type: `${{ github.event.inputs.story_type }}`
- Iteration: ${{ github.event.inputs.iteration }}
- Repository: `${{ github.repository }}`

After rendering your verdict:

| Verdict | Action |
|---------|--------|
| **APPROVED** | Submit `APPROVE` review → add `state:done` → remove `state:review-needed` |
| **KICKBACK** | Submit `REQUEST_CHANGES` → dispatch `software-engineer` with `iteration+1` |
| **BLOCKED** | Submit `COMMENT` → add `state:blocked` → do NOT dispatch |

Max iterations: if `${{ github.event.inputs.iteration }}` > 3, add `state:blocked` and stop.
