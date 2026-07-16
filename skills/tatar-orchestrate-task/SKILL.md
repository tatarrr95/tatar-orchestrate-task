---
name: tatar-orchestrate-task
description: Orchestrate a parent Linear task and all of its subtasks from implementation through internal multi-agent review, final cross-ticket and thermonuclear review, and transition to In Review. Use when the user invokes /tatar-orchestrate-task or asks to autonomously implement a Linear parent issue such as ALT-123 with subagents.
---

# Tatar Orchestrate Task

Drive one Linear parent issue from its current state to a reviewed implementation. Treat Linear as the task-state source of truth, use `$implement` for implementation, `$tatar-review-ticket` for independent review roles, and `$tatar-thermonuclear-review` for the final whole-task structural audit.

Communicate with the user in Russian. Continue autonomously while safe in-scope work remains.

## Invocation Contract

Require exactly one parent Linear identifier, for example:

```text
/tatar-orchestrate-task ALT-123
```

The invocation authorizes these Linear mutations for the parent and its descendants:

- move an issue to `In Progress` when implementation begins;
- move an issue to `In Review` only after the gates in this skill pass.

Do not set issues to `Done`. Do not write ordinary findings or progress comments to Linear. Do not create follow-up Linear issues unless the user separately asks.

## Required Skills and Context

Before acting:

1. Read the repository `AGENTS.md` and `_bmad-output/project-context.md` completely.
2. Read `$implement`, `$tatar-review-ticket`, and `$tatar-thermonuclear-review` completely.
3. Connect to Linear and use read calls before write calls.
4. Resolve the team's current statuses by name. Require exact statuses `In Progress` and `In Review`; never hardcode status UUIDs.

If a required skill, Linear access, or required status is unavailable, stop before mutating issue state and report the blocker.

## Non-Negotiable Boundaries

- Preserve user changes and unrelated dirty work. Never reset, discard, or silently stash it.
- Never run parallel implementers in the same worktree. Use isolated worktrees/branches for parallel implementation; otherwise serialize.
- Track every worktree and local branch created by this orchestration run. Never delete pre-existing or user-owned worktrees or branches.
- Reviewers are read-only. Only implementers edit product code.
- The orchestrator alone triages findings, writes deferred work, integrates ticket branches, and changes Linear status.
- A current-task regression is never deferred merely because the acceptance criteria did not mention it.
- Keep the parent and any incomplete child in `In Progress` after failures or unresolved findings.
- Do not create or publish a PR unless the user separately requests it.

## Phase 1: Load and Normalize the Linear Task Graph

1. Fetch the parent issue with relations, attachments, and git branch metadata. Fetch all comments.
2. Identify its team and resolve `In Progress` and `In Review` for that team.
3. Refuse to reopen `Done`, `Canceled`, or `Duplicate` without explicit user direction.
4. If the parent is `Todo`, `Backlog`, or `Triage`, move it to `In Progress`. If already `In Progress`, resume. If already `In Review`, inspect state and run only the unfinished final gate unless fixes require returning it to `In Progress`.
5. Recursively fetch child issues, their comments, and their blocking relations. Treat leaf issues as implementation units. If there are no children, treat the parent itself as the single implementation unit.
6. Build a dependency DAG. A ticket enters the executable frontier only when all in-graph blockers are internally complete (`In Review` or `Done`) and their required code is confirmed present in the integrated parent branch. Do not trust status alone after a resumed or externally modified run. Exclude `Canceled` and `Duplicate` tickets and record why.
7. Capture `parent_base_sha` before the first task change. Ensure every ticket can later be tied to an exact base/head range.

If the graph contains a cycle, missing child specification, or ambiguous ownership that materially changes implementation, stop and ask one concise question.

## Phase 2: Dispatch Implementers

For each executable frontier, dispatch as many implementation agents as the runtime safely supports.

Maintain an orchestration resource manifest containing each implementer or integration-fix worktree path, local branch name, ticket, and creation base SHA. Include resources created by subagents. This manifest defines the cleanup scope; discovering a branch by name pattern alone is not sufficient ownership evidence.

Parallelize only when both are true:

- Linear blocking edges permit it;
- each implementer has an isolated worktree/branch based on the current integrated parent branch.

Otherwise serialize. Integrate completed dependency tickets before branching tickets that depend on them.

Immediately before dispatching a ticket, move it to `In Progress`.

Give each implementer:

- the full child issue and comments;
- the parent issue as context, not as expanded scope;
- relevant blockers and already integrated decisions;
- its isolated worktree/branch and `ticket_base_sha`;
- the instruction to invoke `$implement`;
- the repository testing, migration, and validation rules;
- the requirement to make surgical changes and commit them with the ticket identifier;
- the requirement to return commit SHA(s), changed files, tests run, and unresolved risks.

Override one `$implement` behavior for this orchestrated run: the implementer must not invoke its final `/code-review`. Central review is owned by this orchestrator through `$tatar-review-ticket`. Retain `$implement` requirements for TDD where appropriate, regular targeted checks, final affected-app verification, and commits.

If an implementer fails, keep the ticket `In Progress`, preserve its work, inspect the failure, and resume with the same agent when useful. Do not mark it reviewed.

## Phase 3: Review Each Ticket

After an implementer commits, run three independent reviewer agents in parallel. Give them no other reviewers' findings. Each reviewer packet must contain exactly one role, and the agent must load only that role's reference from `$tatar-review-ticket`; never ask one reviewer to combine roles. Each must invoke `$tatar-review-ticket` with the exact ticket base/head range and one role:

1. `requirements` — child acceptance criteria, parent constraints, scope creep, and test evidence.
2. `correctness` — logic, integration contracts, security, project invariants, and regressions.
3. `edge-cases` — exhaustive branching, boundaries, races, timeouts, partial failures, and deletion regressions.

Collect their JSON arrays. Reviewer failure is not a clean review; retry once or report the incomplete layer.

### Triage

For every finding:

1. Open the cited code and enough surrounding callers, guards, validators, types, and tests to verify reachability.
2. Deduplicate findings by underlying behavior, not wording.
3. Disregard reviewer-implied severity or merge decisions.
4. Classify into exactly one bucket:
   - `fix_now`: required by the ticket/parent or introduced/exposed by current work;
   - `decision_needed`: correct behavior cannot be inferred from available requirements;
   - `deferred`: verified pre-existing out-of-scope defect with actionable impact;
   - `dismiss`: false positive, duplicate, speculative improvement, or sibling/future scope.

Ask the user only for genuine `decision_needed` findings. Do not ask about unambiguous fixes.

Send one consolidated, evidence-backed `fix_now` list back to the original implementer. Require fixes, commits, and targeted verification. Apply the review convergence budget below instead of reviewing until no reviewer can discover anything new.

### Review Convergence Budget

Apply this budget independently to each ticket gate in Phase 3 and to the whole-task gate in Phase 4:

- Run one initial review, then allow at most two consolidated fix rounds. A fix round consists of one implementer pass over all accepted `fix_now` findings followed by re-review. Reviewer retries caused by tool or agent failure do not consume a fix round.
- Re-run only affected review roles. Re-run all roles after a high-impact or cross-cutting fix, but require every re-review to focus on the fix delta, unresolved accepted findings, and regressions directly caused by that delta. Do not use re-review to start a fresh open-ended audit of unchanged code.
- Triage genuinely new findings from a re-review normally, but accept them as `fix_now` only when they identify an unmet current requirement or a regression introduced or exposed by the current implementation or fix delta. Dismiss speculative improvements and defer only verified pre-existing out-of-scope defects under the existing deferred-work rules.
- Stop as soon as no accepted `fix_now` or `decision_needed` finding remains; unused rounds are not required.
- After the second fix round, do not start another broad review or accept newly discovered improvements from unchanged code. Freeze the accepted finding set and enter closure mode: keep routing every unresolved `fix_now` to an implementer until it is fixed, and resolve every `decision_needed` through the user when necessary. Verify closure with targeted tests and a narrowly scoped check of the cited behavior and its fix delta; fix any regression directly caused by a closure-mode change. Closure checks must not reopen unrelated code or expand the finding set. Never defer or silently waive a current-task defect merely because the review budget is exhausted.

### Integrate and Advance State

Integrate the reviewed ticket branch into the parent branch in dependency order. If integration changes behavior or creates conflicts, resolve them through an implementer and review the integration delta.

Move the child to `In Review` only when:

- implementation is integrated;
- all accepted findings are fixed or explicitly resolved;
- required migrations have been generated and applied according to project rules;
- required tests, lint, types, and migration checks pass;
- the exact reviewed commit range is known.

If the parent has no children and is itself the implementation unit, record this gate as internally passed but keep the parent `In Progress`. Only Phase 4 completion may move the parent to `In Review`.

Then advance the DAG frontier.

## Deferred Work

Use `_bmad-output/implementation-artifacts/deferred-work.md` as the single ledger. Do not create a second ledger.

Only the orchestrator may append an item, and only after verifying that it is:

- real and reproducible or directly evidenced;
- pre-existing rather than caused by the current parent task;
- outside the current parent scope;
- actionable and significant enough to preserve.

Do not defer current regressions, missing current requirements, cosmetic nits, hypothetical risks, product ideas, or vague refactoring opportunities. Search the ledger first and merge with an existing item instead of duplicating it.

Append compact entries with: source parent/ticket, evidence and location, impact, reason out of scope, suggested owning area, date, and verified commit SHA. Do not write the item to Linear.

## Phase 4: Final Whole-Task Gate

After every non-excluded child is `In Review` or `Done`, review the complete integrated range `parent_base_sha...HEAD` with three independent agents in parallel. Keep the two `$tatar-review-ticket` roles isolated in separate agents and packets:

1. `$tatar-review-ticket` with role `requirements` and scope kind `parent` — prove the parent specification is covered by the union of child implementations.
2. `$tatar-review-ticket` with role `integration` and scope kind `parent` — find cross-ticket contract gaps, incompatible transitions, partial expand/contract states, and integration regressions.
3. `$tatar-thermonuclear-review` — audit the whole task for structural regressions and high-value simplifications.

Triage these findings with the same rules. Route localized fixes to the best original implementer; route cross-ticket fixes to a dedicated integration implementer using `$implement`. Review every fix delta under the Review Convergence Budget; do not repeat final layers beyond its two fix rounds.

Run the complete required verification for every affected application and package. Run `pnpm migrations:check` when database artifacts are touched. Regenerate Graphify once at the end when preparing the task for review, as required by repository policy.

## Phase 5: Cleanup Orchestration Worktrees and Branches

After the complete integrated parent range passes Phase 4, reclaim the temporary Git resources recorded in the orchestration resource manifest. Keep the orchestrator's current worktree and branch; this is the single branch handed to the user for merge.

For every recorded implementer or integration-fix resource:

1. Prove its required commits are present in the orchestrator branch and record the integrated head SHA. Account for the actual integration method: merge, rebase, or cherry-pick.
2. Inspect the worktree for uncommitted or untracked files. If it is dirty, do not remove it or force-delete its branch; treat the unexplained state as a blocker and report it.
3. Remove the linked worktree with `git worktree remove <path>`, never by deleting its directory directly.
4. Delete its local branch after the worktree is removed. Prefer `git branch -d`; use `git branch -D` only when the integration proof from step 1 shows the branch has no required work absent from the orchestrator branch, such as after a verified cherry-pick.
5. Run `git worktree prune`, then verify that every owned temporary worktree path and branch is gone.

Do not delete remote branches, the orchestrator branch, pre-existing resources, resources outside the manifest, or any resource with unintegrated or unexplained work. If the task stops before the final gate passes, preserve all implementation resources for recovery and report them instead of cleaning them up.

## Completion

Move the parent to `In Review` only when:

- every child is internally complete and in `In Review` or `Done`;
- parent requirements coverage passes;
- cross-ticket integration review passes;
- all accepted thermonuclear structural regressions are fixed;
- full required verification passes;
- deferred work has been deduplicated and persisted;
- every owned temporary implementer/integration worktree and local branch has been safely removed;
- only the orchestrator worktree and branch remain from this orchestration run;
- the working tree contains no unexplained orchestration artifacts.

Never move the parent to `Done`.

Report in Russian:

- child ticket status table;
- implementation and review commit ranges;
- accepted/fixed/dismissed/deferred finding counts;
- review rounds used per ticket and for the whole-task gate, including findings resolved in closure mode;
- verification commands and results;
- deferred ledger entries added or updated;
- cleanup summary with removed worktrees/branches and any resources intentionally preserved;
- orchestrator branch retained for merge;
- parent Linear status;
- remaining blockers, if any.
