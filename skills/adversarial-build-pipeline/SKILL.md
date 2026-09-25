---
name: adversarial-build-pipeline
description: "Use whenever the user asks to build or implement something non-trivial (\"build X\") — runs the full graph: plan → adversarial plan review → spec → parallel file-disjoint implementation → adversarial diff review → verify."
---

# Adversarial build pipeline

Run this full graph for any build/implement request ("build X", "implement Y"). Use the eval kernel's primitives. Never skip the refutation gates; never let subagents verify their own work.

## Dispatch (existing bundled agents only — no custom agents, no new roles)

- **Doers** — every building step (plan, spec, fan-out implement): bundled `task` agent.
- **Reviewers** — every adversarial step (refute-plan, refute-diffs, spec-screen, second reviewer pools): bundled `reviewer` agent.

## Lens set (mandatory, every adversarial pool)

Every refutation wave is multi-lens, plus any fixed screening items specified in Phase mechanics; screening items are exempt from the lens sweep. Size tiers set granularity (the never-fan-out-suite-runs rule applies at every tier):

- **Trivial** (≤3 files, 0 flagged risks): plan inline, one doer item, ONE refute item per refutation stage (refute-plan reviews the inlined plan; refute-diffs reviews the diff), each explicitly sweeping all five lenses, orchestrator verify — gates shrink, never skip.
- **Small** (≤8 files): one item per diff/claim — and each such item MUST explicitly sweep all five lenses.
- **Larger**: one item per lens per file.

1. **correctness** — logic errors, broken invariants, missed edge cases
2. **readability** — clarity, naming, structure a new reader can follow
3. **maintainability** — coupling, testability, cost of future change
4. **performance** — unnecessary allocation/copying/computation, hot-path waste
5. **language best-practices** — idiom and conventions of the project's language(s)

Lenses apply to the artifact under review: design/architecture in plan waves, code in diff waves. Additional lenses (security, scope creep) only when the request calls for them.

## Graph

```
plan → refute-plan → spec → fan-out implement → refute-diffs → verify + integrate
```

## Phase mechanics

1. **Scope inline first.** Before spawning anything, map the target yourself: read the relevant files, run `git status`, identify the file inventory. Fan-out slices depend on this map.

2. **plan** — `agent("Plan <task>", agent="task", schema=PLAN).wait()`. PLAN schema: `{goal, approach, file_map[], risks[], open_questions[], assumptions[], verification[]}`. The plan brief must include a verification plan: the exact runnable acceptance commands (tests/typecheck/lint/build) that gate "done", chosen before implementation; UI-only work may declare a user-approved manual gate as acceptance, labeled as such. Single node, not a pool. Planning brief requires: every load-bearing external claim (library/API/framework behavior, version availability) is verified against official docs where the agent has `web_search`, with the URL recorded in `assumptions`; anything unchecked is labeled `[UNVERIFIED]` in the plan and queued for orchestrator lookup — never asserted.

3. **refute-plan** — new cell, new pool: `workpool("reviewer", name="refute-plan")`. One item per lens from the lens set. Reviewers do NOT trust the plan's framing: independently re-verify the plan's external assumptions against official docs — via `web_search` if the reviewer's toolset has it, otherwise mark the assumption `[UNVERIFIED]` for orchestrator lookup. A refutation of an external claim cites the doc URL; design refutations cite `file:line` in the existing repo; unevidenced objections are logged and reported as unvalidated, never acted on. The orchestrator confirms every citation before a plan element is altered or killed. Survivors feed forward.

4. **spec** — `agent(f"Write spec given plan + refutations: {...}", agent="task", schema=SPEC).wait()`. SPEC schema: `{interface, files_to_create[], files_to_edit[], invariants[], acceptance[], slice_intents[], frozen_interfaces[]}`. Acceptance items are runnable acceptance commands with expected observable outcomes (exit code/output), not prose; the spec item executes each acceptance command once to record its expected output — that calibration proves each command runs and records its pre-implementation output; verify gates on the declared expected outcome (exit code / declared output), not on similarity to the calibration recording (single-runner rule applies). UI tasks may substitute a manual gate as acceptance only if the user approved it, labeled as such. Every slice declares its intent: exported/edited symbols plus operation class (add/extend/replace/remove). For multi-slice fan-outs, one spec-screen reviewer item in a new cell then pairwise-scans slice boundaries for destructive-vs-additive conflicts (one slice replaces what another extends); single-slice tasks skip the screen. After adjudication, surviving shared interfaces freeze into `frozen_interfaces[]` for the run; they change only via the orchestrator, who re-broadcasts (`write agent://all`) to sibling owners before any fix item runs. Spec must absorb surviving refutations explicitly and may only rely on assumptions that survived verification.

5. **fan-out implement** — before the FIRST fan-out the orchestrator requires a clean `git status`; a dirty tree stops dispatch with a report (checkpoint-commit only if the user authorized commits this session). Corrective waves require only the absence of foreign (non-pipeline) changes. Pool: `workpool("task", name="impl")`. Items are **file-disjoint**; one owner per shared file boundary (serialize shared files to a single item). Every item: self-contained brief (paths, APIs, edge cases, its slice's acceptance commands, the `frozen_interfaces[]` list and what siblings do NOT share, pointers to related code from the plan's `file_map[]` — search before assuming missing: do not reimplement what exists), explicit "no full-suite runs; slice-scoped acceptance commands allowed; never alter tests or harness to make checks pass (a test must fail when the feature is broken); never mutate git state (stash/checkout/reset/clean/commit) — git authority is the orchestrator's". A worktree-per-item layout is an optional harness-dependent variant: file-race isolation, not a security boundary. Editable integration contract via `write agent://<id>` when siblings must coordinate.

6. **refute-diffs** — new named pool: `workpool("reviewer", name="refute-diffs")`. Per the lens set: per-diff items (each sweeping all five lenses) for small changes; per-lens-per-file for broad ones. Reviewer briefs carry the `frozen_interfaces[]` list: cross-slice contradictions are in-scope for the correctness lens. Optional `judge_batch(diffs, RUBRIC, intent="diff scoring")` to rank before human-visible report — if used, it runs only after the second reviewer pool. Freeze the rubric before loading diffs.

7. **verify + integrate** — the orchestrator personally: runs every acceptance command across the union of changed files — subagent prose and test-runner summaries are logging only, the exit code is the gate; reads the touched diffs and scans them for masking edits (new/modified test scaffolding, mocks, app-under-test config, runtime patches); re-checks slice boundaries against `frozen_interfaces[]` on the union; resolves every `[UNVERIFIED]` flag via its own `web_search`. Pool output is evidence, not truth. Negative control (test-tampering gate): for diffs adding/modifying tests, prove a new test fails against a deliberately broken implementation — only when the broken file's pipeline changes are staged or committed (else skip and log the reason); single-test invocation scoped to the touched test files, never the full suite, ≤3 demonstrations per verify run (most load-bearing first); break one load-bearing line → run → observe fail → restore that line to its pre-break content (never `git checkout -- <file>` while the file carries uncommitted pipeline changes) → re-run green. Orchestrator-owned; exempt from the no-inline-edit rule. Fix-ups go through new corrective subagent items, never silent inline edits for substantial gaps; corrective-fix briefs embed the exact failing command, the verbatim failing output up to ~50 lines (beyond that: the file path plus the exact failing lines and the exit code), and the exit code — never a paraphrase.

## Anti-hallucination gates

- Every reviewer brief requires: findings citing `file:line` against the real repo; claims about third-party/library/framework/standard-library behavior MUST be checked against official docs and cite the URL — via `web_search` where the agent's toolset includes it; where it does not, the claim is labeled `[UNVERIFIED]` and routed to the orchestrator, who always has `web_search`. Uncited external claims, as unevidenced ones, are logged and reported as unvalidated and never acted on.
- Reviewers mark anything they could not verify as `[UNVERIFIED]` instead of asserting it.
- The orchestrator re-checks every finding that rests on an external source before acting on it, and discards findings whose citation doesn't check out.
- Plan and spec are held to the same bar: external claims verified at plan time (URLs in `assumptions`) and independently re-verified by refuters — a hallucinated premise must fail twice before it can reach implementation.
- Every finding is a falsifiable claim paired with its cheapest disproving check (command to run, doc URL to fetch, `file:line` to open) — never a conclusion alone. Lookup-class checks (doc fetch, line open) run in-item under this bar; the orchestrator spot-checks a sample of code (`file:line`) citations; external-source findings are re-checked in full per the gate above.

**Adjudication protocol** — the orchestrator is the ONLY bridge between reviewer findings and code changes, applied in order:
1. Evidence gate: act only on findings carrying an executed disproving-check result or a verified citation; everything else is logged and reported as unvalidated — never silently discarded.
2. Intent/scope filter: reject a finding only if it expands scope beyond the user's request, contradicts a stated user constraint, or conflicts with a frozen slice intent — each rejection logged with which test failed.
3. Verdict: structured PASS/FAIL per slice with specific findings (no free-text "LGTM"). The orchestrator judges; it never re-implements.
4. Wave cap: corrective waves default to 2 per slice; a corrective wave counts against the cap of every slice it touches; on cap, stop and report failure evidence to the user.

## Wave discipline

- Wave barrier = cell boundary: push pool items, end the cell, results auto-deliver, next cell builds the next wave from delivered results. `wait()` barriers only `agent()`/`completion()` handles — never pools. Never poll.
- New named pool per wave; a pool closes after its first full drain.
- **Second reviewer pool** — after each refutation pool that produced blocking/major findings: new cell, new named `reviewer` pool. It runs ONLY command-execution disproving checks (execution items, not lens reviews); only findings reviewers marked blocking/major; cap 4 items per pool — `log()` any cap applied. Findings below the bar or beyond the cap are logged and reported as unvalidated, never silently discarded. On the trivial tier the orchestrator runs these checks personally instead of spawning the pool.
- Pool results carry paths, not payloads: artifacts too large for the structured summary go to a file; the result references it.
- Capture phases in `todo`; mark done as waves settle.
- Push a completeness-critic item in the last refutation or second-reviewer pool: "what file/claim/modality remains unchecked?"
- `log()` any sampling caps. New discoveries mid-pool: push into the still-running pool; dedup against SEEN.

## Defaults (tune per request)

- Dispatch: doers = bundled `task`; reviewers = bundled `reviewer`. No new roles.
- Lens set: the five lenses above, always all of them in every refutation item; granularity per the size tiers.
- Evidence bar: `file:line` for code claims; official-docs URL for external claims where the agent has `web_search`; `[UNVERIFIED]` + orchestrator lookup otherwise.
- Concurrency: cap concurrent impl-pool items at 8; exactly one runner (one designated item or the orchestrator) executes full-suite build/test/typecheck per wave — never fan out suite runs; during impl/corrective waves the runner is the orchestrator (impl items never run the full suite) — a designated doer item may be the runner only outside impl pools, e.g. spec calibration, at any tier; subagents never spawn nested pools or background work (grounded in https://www.anthropic.com/engineering/multi-agent-research-system — synchronous subagents are a deliberate simplification — and https://ghuntley.com/ralph/ — only one build/test runner).
- Budget: none unless the user appends `+Nk` / `+Nk!`; an explicit user budget overrides the size tiers.
