# Workflow Harness Research Handoff - 2026-06-28

## Purpose

This document captures the current research and design direction for replacing the brittle parts of the user's skill-based automation flow with a deterministic workflow harness.

The project name is intentionally omitted. Naming should remain out of scope unless explicitly reopened.

## Current Context

The user currently has:

- Skills repository: `C:\web\skills`
- Installed skills: `C:\Users\ggumi\.agents\skills`
- New harness workspace: `C:\web\grillmerge`
- GitHub repository: `Gil-1/grillmerge`

The existing workflow roughly is:

1. Research a topic or feature.
2. Run `grill-with-docs` or a Matt Pocock-style grilling session.
3. Create planning docs such as context, glossary, and ADRs.
4. Run `to-prd`.
5. Run `prd-to-prod-autopilot`.
6. Create GitHub issues and PRs.
7. Wait for Codex PR validation.
8. Human reviews and merges the final PRs.

The user wants to preserve the useful Matt workflow outputs:

- PRD issues on GitHub.
- Context files and ADR files in the repository.
- Child implementation issues.
- PRs for completed work.
- Human merge decisions at the end.

The pain point is that the current orchestrator skill works only around 60-70% of the time. It is slow and brittle because long-running workflow state, retries, scheduling, issue/PR tracking, and validation loops live mostly in model context.

## Local Artifacts Reviewed

These files were inspected during the conversation:

- `C:\Users\ggumi\.agents\skills\codebase-design\SKILL.md`
- `C:\Users\ggumi\.agents\skills\prd-to-prod-autopilot\SKILL.md`
- `C:\Users\ggumi\.agents\skills\to-prd\SKILL.md`
- `C:\Users\ggumi\.agents\skills\grill-with-docs\SKILL.md`
- `C:\web\skills\README.md`
- `C:\web\skills\package.json`
- `C:\web\skills\skills\engineering\prd-to-prod-autopilot\SKILL.md`
- `C:\web\skills\skills\engineering\codex-pr-review\scripts\watch-codex-pr.mjs`

The installed `prd-to-prod-autopilot` skill is prompt-heavy and owns many responsibilities:

- issue creation
- worktree policy
- worker scheduling
- implementation supervision
- verification
- review/fix pass
- PR publishing
- Codex review loop
- blocker recovery
- issue reconciliation
- cleanup

That responsibility set is too large for a prompt-only orchestrator. The next design should move deterministic state and transition logic into code.

## Recommendation

Build a TypeScript workflow harness first, then expose it through a thin OpenCode plugin.

The harness should own:

- durable run state
- issue queue state
- step transitions
- retries and idempotency
- scheduling and dependency handling
- Git worktree creation and cleanup
- GitHub issue/PR updates
- local verification
- Codex/OpenCode session IDs
- CI and Codex review monitoring
- timeout handling
- resume/recovery after interrupted sessions

Agent sessions should own bounded judgment-heavy leaf tasks:

- implement one issue in one assigned worktree
- review one PR or diff against one issue brief
- fix concrete review comments
- diagnose one failing command or check
- summarize or update one GitHub item

Do not replace one giant skill with one giant OpenCode plugin. The deep module should be the harness. The OpenCode plugin should be a shallow operator surface over the harness interface.

## Why TypeScript

OpenCode's plugin and SDK surface is TypeScript/JavaScript-first. Official OpenCode docs confirm plugins, custom tools, hooks, and SDK session APIs. A TypeScript harness reduces friction between the core workflow engine and OpenCode integration.

Python remains viable for a later backend service, especially if the project adopts Temporal, Prefect, DBOS, Restate, or heavy data processing. It is not the cleanest first choice for an OpenCode-integrated local harness.

## Proposed Architecture

The likely first version:

```text
workflow harness
  -> GitHub PRD issue
  -> create or discover child issues
  -> create per-issue worktree
  -> run agent implementation session
       adapter: Codex SDK
       adapter: OpenCode SDK
       optional adapter: Claude Code SDK
  -> run local checks
  -> run review/fix pass
  -> open/update PR
  -> monitor CI + Codex review asynchronously
  -> mark PR ready / blocked / needs-fix / timed-out
```

Core interface sketch:

```ts
type WorkflowHarness = {
  startPrdRun(input: {
    repoPath: string
    prdIssue: string
    mode?: "one-pr-per-issue" | "single-pr"
  }): Promise<RunSummary>

  tick(runId: string): Promise<RunSummary>
  resume(runId: string): Promise<RunSummary>
  status(runId: string): Promise<RunSummary>
  retryTask(taskId: string): Promise<RunSummary>
}
```

Possible modules:

```text
src/core/
  workflow.ts
  state-machine.ts
  scheduler.ts
  prompts.ts

src/state/
  event-store.ts
  run-store.ts

src/adapters/
  git-cli.ts
  github-gh.ts
  opencode-sdk.ts
  codex-agent.ts
  codex-review.ts
  checks.ts

src/cli.ts

.opencode/plugins/
  workflow-harness.ts
```

Potential adapters:

- `GitHubAdapter`
- `GitAdapter`
- `CodexAgentRunner`
- `OpenCodeAgentRunner`
- `CodexReviewWatcher`
- `IssuePlanner`
- `WorktreeManager`
- `StateStore`

## State Store

Start simple with local durable files:

```text
.ai-integrator/runs/<run-id>/
  events.jsonl
  state.json
  logs/
  prompts/
  agent-sessions.json
```

Use append-only `events.jsonl` plus `state.json` snapshots before introducing SQLite. This gives visible debug, resume, and retry behavior with minimal infrastructure.

SQLite is a reasonable next step once querying, concurrency, or reporting becomes painful.

## OpenCode Plugin Role

The OpenCode plugin should expose commands/tools only:

```text
/workflow start <prd-issue>
/workflow status
/workflow resume
/workflow retry <task>
/workflow stop
```

Equivalent tool names:

```text
workflow_start_prd_run
workflow_status
workflow_resume
workflow_retry_task
workflow_stop
```

The plugin should call the harness. It should not contain the scheduling, retry, GitHub, worktree, or validation loop logic.

## Codex PR Validation

Codex PR validation should become an asynchronous monitor step, not a synchronous blocker for the entire run.

Suggested flow:

1. Agent implements an issue.
2. Harness runs local checks.
3. Harness runs a cheaper review/fix pass.
4. PR is opened only after local gates pass.
5. Codex PR validation starts.
6. Harness marks the PR as one of:
   - `ready`
   - `needs-fix`
   - `timed-out-waiting-codex`
   - `blocked`

This prevents one slow Codex review from freezing the whole workflow.

## Framework Notes

Recommended first stack:

- TypeScript harness.
- Local JSON event store at first.
- `gh` CLI adapter initially, with room for Octokit later.
- Fake agent runner for tests.
- Real Codex and/or OpenCode runner after the state machine is stable.

Framework tradeoffs:

- Temporal: strongest durable execution story for long-running workflows, retries, timers, and crash recovery. More infrastructure.
- Inngest: pragmatic TypeScript-friendly durable steps, retries, concurrency, throttling, and observability. Easier than Temporal.
- LangGraph: useful for agent graph/stateful reasoning, but probably not the core because most complexity is Git/GitHub/worktree/PR lifecycle state.
- Mastra: interesting TypeScript workflow/agent framework with suspend/resume concepts. Worth watching.
- Pydantic AI: strong Python option with durable execution integrations, but less aligned with OpenCode plugin work.
- GitHub Copilot coding agent / GitHub Agentic Workflows: GitHub-native, but with less local control.
- Codex GitHub Action: useful for CI-side reviews or patch tasks, not the whole orchestrator.
- Claude Code SDK: plausible optional runner adapter.

Useful documentation URLs:

- OpenCode plugins: https://opencode.ai/docs/plugins/
- OpenCode SDK: https://opencode.ai/docs/sdk/
- OpenCode config: https://opencode.ai/docs/config/
- OpenCode skills: https://opencode.ai/docs/skills/
- Codex SDK: https://developers.openai.com/codex/sdk
- Codex GitHub Action: https://developers.openai.com/codex/github-action
- OpenAI Agents SDK orchestration: https://developers.openai.com/api/docs/guides/agents/orchestration
- LangGraph overview: https://docs.langchain.com/oss/python/langgraph/overview
- Pydantic AI durable execution: https://pydantic.dev/docs/ai/integrations/durable_execution/overview/
- Temporal: https://temporal.io/
- Inngest: https://www.inngest.com/docs
- Mastra workflows: https://mastra.ai/docs/workflows/overview
- GitHub Copilot cloud agent: https://docs.github.com/en/copilot/concepts/agents/cloud-agent/about-cloud-agent
- GitHub Agentic Workflows: https://github.github.com/gh-aw/
- Claude Code headless: https://code.claude.com/docs/en/headless

## Suggested Milestones

First milestone:

```text
Given a GitHub PRD issue URL, create a durable run record, discover or create child issues, create worktrees for ready issues, and print a deterministic execution plan without running agents.
```

Second milestone:

```text
Run one issue end-to-end through fake agent -> local checks -> fake PR state -> ready-for-human-merge.
```

Third milestone:

```text
Swap fake agent/PR adapters for real Codex SDK and GitHub adapters.
```

Narrow vertical slice for early validation:

1. Start from an existing GitHub PRD issue.
2. Discover child `ready-for-agent` issues.
3. Create one worktree for one issue.
4. Launch one OpenCode or Codex agent session for that issue.
5. Wait for completion or timeout.
6. Run local verification commands.
7. Record structured status in the run state.
8. Do not push PRs yet.

After that works reliably, add:

- PR publishing
- CI monitoring
- Codex PR review watching
- retry/fix loops
- parallel issue execution
- post-merge issue reconciliation

## Open Questions

- Should the first real runner target Codex, OpenCode, or both via adapters from day one?
- Should run state be local JSON initially or SQLite from the start?
- Should the first GitHub adapter call `gh`, Octokit, or support both?
- Should issue creation continue to be handled by Matt's `to-issues` skill initially?
- How much should the harness update GitHub comments and labels versus only keeping local execution state?
- Should PRD-to-issues become deterministic harness behavior later, or remain a skill-owned planning step?

## Suggested Skills For Continuation

- `codebase-design`: use for the harness interface and module seam decisions.
- `tdd`: useful for building the state machine and adapters test-first.
- `plugin-creator`: useful only if creating a Codex plugin or adapting plugin scaffolding patterns.
- `openai-docs`: use when checking current Codex SDK or OpenAI Agents SDK details.
- `project-folder-structure`: useful once the repo starts accumulating harness code, docs, and adapters.
- `diagnosing-bugs`: useful later when debugging brittle workflow execution or failed PR validation loops.
- `to-prd`: useful after design decisions stabilize and the user wants a PRD issue.
- `to-issues`: useful after a PRD exists and implementation should be split into agent-ready slices.

## Non-Negotiable Constraint

Do not remove the Matt workflow. The goal is to preserve human-readable planning artifacts and the GitHub issue/PR process while making the repetitive delivery loop deterministic, resumable, and easier to debug.
