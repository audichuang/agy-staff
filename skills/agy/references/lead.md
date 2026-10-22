# Leading a task with agy

Read this when the user asks you to coordinate a larger task with agy, or when you split one task into several agy assignments. You are the lead in the current harness.

## Working with AGY

1. **Frame the assignment.** Orient just enough to state the outcome and completion criteria. Discovery itself can be delegated when the right next step is unclear; unknown interfaces or implementation choices need not be settled before discovery starts. Supply relevant background, constraints, settled decisions, and existing authorizations in the brief without expanding their scope. AGY sees its brief and its conversation, not the host's intervening discussion.
2. **Default to delegating substantive work.** Use AGY to advance the task while you own user communication, cross-task decisions, acceptance, integration, and delivery. Handle work directly when it is small or existing context makes handoff and review more expensive. Make routine orchestration choices within the user's existing authorization. Stay within the requested scope and stage: discussing a proposal does not authorize implementing it.
3. **Choose mode and worker scope by outcome.** Default to `staffer`; choose a specialist mode when the user requests it or its guidance materially improves the assignment (`research` for a source-backed survey, `review` for independent critique, `implement` for a scoped code change with verification). Reserve `ask` for smoke tests and genuinely one-shot questions that need no tools; do not route ordinary work to it. Keep related context together and choose worker count by useful independent outcomes. Implementation normally includes necessary verification in the same assignment; avoid splitting one change into implement, test, and review handoffs. Before parallel assignments that share interfaces, naming, or approach, settle those decisions and include them in each affected brief. Workers that edit files in parallel need separate git worktrees, each launched from its own worktree, because job state and continuation are per worktree; otherwise run editing assignments one at a time and parallelize only reads.
4. **Wait by default.** After dispatch, wait for the result through `jobs`. Parallel host work should be an already-identified independent task worth handling directly. Idle time is not a reason to open another investigation into the same problem; avoid duplicating delegated work or delivering conclusions that depend on unavailable worker results.
5. **Require an assessable result.** Specify the artifacts and evidence needed for acceptance. For example: research findings with sources and uncertainties; actual implementation changes with verification results, consequential choices, and remaining issues; or review findings with locations, triggering conditions, evidence, and impact, plus coverage limits even when no issues are found. Adapt these requirements to the task. The worker may analyze and recommend; you retain final judgment.
6. **Assess, then decide the next step.** Read every result against completion criteria and check for conflicting assumptions or interfaces across workers. Inspect relevant diffs and verification for edits; make targeted checks of consequential claims as needed without repeating the whole investigation. Turn substantive gaps into focused follow-ups, continuing the existing conversation when its context helps and supplying new user decisions. Use a fresh conversation for an independent opinion or different context. Handle short, specific checks directly; take over when another handoff is unlikely to help. Integrate the results, disclose material omissions, and complete the requested delivery when the task is satisfied. Add review rounds only when they resolve meaningful uncertainty.

## Dispatch and follow through

Read `jobs.md` for result collection, cancellation, continuation, and recovery. Write the brief to a temporary file and call the shared companion:

```bash
node "<skill-dir>/companion/agy-companion.mjs" staffer --prompt-file "<brief-path>"
```

For a specialist, replace `staffer` with `research`, `review`, or `implement`. When requesting code review, use the review-brief guidance in `code-review.md`. Modes retain their existing model and permission defaults; honor user overrides. Run unsandboxed as described in `troubleshooting.md` (escalated execution in Codex).

Keep each returned job ID with its assignment and collect the result through jobs. Prefer `continue --job <id>` for follow-ups; the companion refuses it while that job is still running. Let useful running work finish when feedback can wait; for an immediate change, follow jobs' cancel, confirm termination, then continue sequence. Account for partial work after interruption and follow the existing timeout recovery rules.

In this workflow you compose briefs and synthesize results. The verbatim-delivery rule in SKILL.md applies to single delegations. Preserve exact quotes, figures, errors, and evidence references when integrating results.
