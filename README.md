# adversarial-build-pipeline

A skill for [oh-my-pi (omp)](https://omp.sh): a deterministic multi-agent build pipeline with adversarial review baked into every stage. Instead of one agent implementing and grading its own homework, doers build and independent reviewers attack the work — before and after code exists.

## The pipeline

```
plan → refute-plan → spec → fan-out implement → refute-diffs → verify + integrate
```

- **plan** — a doer agent produces a structured plan; load-bearing external claims (library/API behavior) must be verified against official docs with URLs, or labeled `[UNVERIFIED]`
- **refute-plan** — a reviewer pool attacks the plan, one item per lens; refutations need evidence (`file:line` or doc URL) or they're logged as unvalidated
- **spec** — structured spec with per-slice intents; conflicting slice intents are screened out before any code is written
- **fan-out** — parallel file-disjoint implementation slices, one owner per shared file
- **refute-diffs** — reviewers attack the diffs through six lenses: correctness, readability, maintainability, performance, language best-practices, comprehension (could someone who missed the conversation work from the artifacts alone?)
- **verify** — the orchestrator personally runs acceptance commands (exit codes are the gate, agent prose is just logging), scans for test tampering, and re-checks citations

Reviewer findings are falsifiable claims that must survive an executed disproving check; the orchestrator is the only bridge between findings and code. A hallucinated premise must fail twice before it can reach implementation.

## Install

### As an omp plugin (recommended)

```sh
omp plugin marketplace add wvmag/adversarial-build-pipeline
omp plugin install adversarial-build-pipeline@adversarial-build-pipeline
```

Then run `/reload-plugins` (or start a new session) so the skill surfaces.

### Manual

Copy the skill directory into your omp user skills:

```sh
mkdir -p ~/.omp/agent/skills/adversarial-build-pipeline
cp skills/adversarial-build-pipeline/SKILL.md ~/.omp/agent/skills/adversarial-build-pipeline/
```

### Which copy wins?

If you have the skill from more than one source, omp keeps the highest-precedence same-named copy: user/project skills (`~/.omp/agent/skills`) beat installed plugins, which beat managed skills (`~/.omp/agent/managed-skills`). Keep one source of truth — if you install the plugin here, delete any local same-named copy or your local edits will be silently shadowed.

## Usage

Ask omp to build something non-trivial:

```
build a rate limiter for the API layer
```

The skill's description matches build/implement requests, so the agent reads it and runs the pipeline automatically. To invoke it explicitly, use `/skill:adversarial-build-pipeline <task>`.

**Optional classifier triage** (off by default): append `; use classifier triage` to the task, and each reviewer finding gets a cheap classifier label for the harm it claims (data loss / bounded bug / style preference / unclear). Labels only reorder which findings get executed disproving checks first — they never validate evidence or override acceptance results.

**Requirements:** a recent omp (tested on 18.3.0) with the `eval` and `task` tools active — the pipeline runs on the eval kernel's `workpool`/`agent`/`completion` primitives and bundled `task` (doers) and `reviewer` (reviewers) agents. All model choice goes through omp model roles (`modelRoles` in your config) and per-agent overrides; the skill never names a concrete model. No extra roles or configuration needed.

## Tuning

The skill's Defaults section is meant to be edited: size tiers, lens set, concurrency caps, evidence bar. Trivial tasks (≤3 files) automatically collapse to a cheaper graph — gates shrink, never skip.
