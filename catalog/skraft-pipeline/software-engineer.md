---
engine: copilot
description: |
  Software-engineer agent for the skraft SDLC pipeline. Triggered by
  workflow_dispatch from acceptance-designer-reviewer (new impl) or
  software-engineer-reviewer (kickback). Implements the plan via
  Outside-In TDD, opens or updates a draft PR, and dispatches the
  software-engineer-reviewer.

on:
  workflow_dispatch:
    inputs:
      issue_number:
        description: The issue this implementation is for.
        required: true
        type: string
      story_type:
        description: functional or technical (propagated from discoverer).
        required: false
        type: string
        default: "functional"
      iteration:
        description: Attempt number in the impl→review loop (1-indexed).
        required: false
        type: string
        default: "1"
      pr_number:
        description: Existing PR to push updates to (set by reviewer on kickback; empty on first attempt).
        required: false
        type: string
        default: ""

concurrency:
  group: skraft-issue-${{ github.event.inputs.issue_number }}
  cancel-in-progress: false

timeout-minutes: 30

permissions: read-all

network:
  allowed:
    - defaults
    - node
    - python
    - dotnet
    - java

checkout:
  fetch-depth: 0

imports:
  - .github/agents/software-engineer.agent.md

tools:
  github:
    toolsets: [default]

safe-outputs:
  threat-detection: false
  add-comment:
    max: 2
    target: "*"
  create-pull-request:
    draft: true
    title-prefix: "[skraft] "
    labels: [skraft, skraft:pr]
    protected-files: fallback-to-issue
    max: 1
  push-to-pull-request-branch:
    target: "*"
    title-prefix: "[skraft] "
    max: 1
  add-labels:
    allowed: [state:review-needed, state:blocked]
    max: 2
    target: "*"
  remove-labels:
    allowed: [state:impl-needed]
    max: 1
    target: "*"
  dispatch-workflow:
    workflows: [software-engineer-reviewer]
    max: 1
---

# Software-Engineer Agent

**Runtime context:**
- Issue: #${{ github.event.inputs.issue_number }}
- Story type: `${{ github.event.inputs.story_type }}`
- Iteration: ${{ github.event.inputs.iteration }}
- Existing PR: ${{ github.event.inputs.pr_number || 'none (first attempt)' }}
- Repository: `${{ github.repository }}`

After executing the full protocol, dispatch `software-engineer-reviewer` with:
- `issue_number`: ${{ github.event.inputs.issue_number }}
- `story_type`: ${{ github.event.inputs.story_type }}
- `pr_number`: (the PR you created or updated)
- `iteration`: ${{ github.event.inputs.iteration }}
