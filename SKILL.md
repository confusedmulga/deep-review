---
name: deep-review
description: Exhaustive, evidence-based review of a codebase or large change that finds all confirmed bugs in one pass, adversarially verifies each, groups them by root cause, and produces a fix plan with a defined stop condition. Use whenever the user asks for a deep review, full audit, "find all bugs", "is this code done/ready/shippable", complains that repeated reviews keep finding new bugs, or wants a review that converges instead of a quick PR-style skim. Prefer this over ad-hoc "review my code" for anything larger than one small diff, even if the user does not say "deep".
---

# Deep review

Ad-hoc "find bugs" prompts never converge, for four mechanical reasons. This skill is built around removing each one.

1. A reviewer asked to find bugs always returns bugs. Nothing in the prompt makes zero an acceptable answer, so the model pads with speculation, style and invented requirements.
2. There is no spec. "Bug" is measured against whatever the reviewer imagines the code should do, and that imagination changes between runs.
3. Recall per pass is a sample, not a census. Each run covers a different slice of the bug population, so the union keeps growing.
4. Every fix is new, unreviewed code. Fixes that refactor or touch extra lines create the next round's findings.

Done is therefore a statistical claim ("N independent passes found nothing new at Medium or above"), never a proof. State it that way in the report.

## Rules that apply to every phase

- A finding needs a concrete trigger (input, call sequence or state), expected vs actual behavior, and file:line. Without a trigger it is a Suspicion. Suspicions are listed separately and never block "done".
- Zero findings from a lens is a valid and expected result. Never pad. Never write "looks good overall" or similar.
- A bug is a deviation from intended behavior. Intent comes from specs, tests, docs, types, names and call sites. If intent cannot be established, record a question or assumption, not a bug.
- Severity gates completion. Only Blocker, High and Medium confirmed findings block "done". Low goes to a backlog. Nit is omitted unless asked. Neither ever triggers another round.
- Read whole files, not grep snippets. Bugs live in the code around the match.

If the user names a path, package or branch, restrict scope to it. For a diff, review the changed hunks plus their direct callers and tests.

## Phase 0: Scope and oracles (alone, before any fan-out)

1. Inventory languages, entry points, modules, external boundaries (network, DB, filesystem, subprocess, user input), config and env, and the build/test commands.
2. Run every objective oracle that exists and record the baseline: test suite, type checker, linter, build, and any static analyzer available (mypy/pyright, ruff, tsc, eslint, flutter analyze, go vet, clippy, bandit, semgrep). A failing test or type error is a confirmed finding with no debate; do not re-derive it in prose.
3. Write an intent sheet: per module, what it is supposed to do, invariants that must hold, trust boundaries, and the assumptions you are making. Source it from README, docs, tests, docstrings and types.
4. Partition into review units, roughly 1,500 lines each, keeping files that call each other together. Under ~1,000 lines total, skip fan-out and run the lenses yourself one after another.

## Phase 1: Fan-out by unit and lens

Cross-module bugs live in the seams, so review along both axes: every unit through every relevant lens, plus one integration pass over module boundaries.

Run lenses in parallel with subagents (Agent/Task tool) when available. Otherwise run them sequentially and finish one lens completely before starting the next. Give each subagent: the intent sheet, its units, one lens, the finding format below, and the zero-is-valid rule.

Lenses:

- **Logic and edge cases**: off-by-one, empty/None/NaN, boundaries, wrong operator, unit or timezone or encoding mismatch, overflow, float precision, wrong defaults.
- **Failure paths**: swallowed exceptions, missing cleanup, partial failure leaving inconsistent state, retries without idempotency, missing timeouts, leaked resources.
- **State and concurrency**: shared mutable state, races, missing await, blocking inside an event loop, ordering assumptions, stale caches, init and lifecycle order (dispose, mounted checks in Flutter).
- **Contracts and integration**: caller/callee mismatch, schema or API drift, serialization, DB model vs migration, config keys that do not exist, code that is unreachable because wiring is missing.
- **Security**: injection, path traversal, authz gaps, secrets in code or logs, unsafe deserialization, SSRF, unvalidated input at every trust boundary.
- **Data and ML correctness** (when present): train/test leakage, wrong axis or shape, nondeterminism, metric computed on the wrong split, silent dtype casts, train/eval mode.
- **Tests and observability**: tests that cannot fail (no assertions, mocks hiding the logic), untested high-severity paths, logs that hide failures.
- **Performance**: only when asked or on a clearly hot path.

Finding format, one per candidate:

```
id | lens | file:line-range | claim (one sentence)
trigger: concrete input / sequence / state
expected vs actual:
evidence: code trace, 1-5 quoted lines
severity guess | confidence
fix: smallest change that resolves it
regression test: sketch
```

Thorough mode: when the user says "thorough", "maximum", or this is the final check before release, run Phase 1 twice as two blind replicas (A and B) on the same snapshot, with different unit partitioning and lens order, and neither seeing the other's output. This enables the estimate in Phase 6.

## Phase 2: Adversarial verification

Verification must run in a different context from the finder: a separate subagent, or when solo, a deliberate second pass whose only goal is to refute. For each candidate, try to disprove it: is it unreachable, guarded elsewhere, or contradicted by the intent sheet? Where possible, write and run a failing test or minimal repro script.

Outcomes: **Confirmed** (reproduced or traced end to end), **Plausible** (trace holds, cannot run it), **Refuted** (drop, one line of why), **Needs-info** (question for the user). Confirmed and strongly traced Plausible findings count. Do not apply any fixes in this phase.

## Phase 3: Dedupe and root cause

Cluster findings by root cause; one cause gets one fix. Record fix ordering dependencies and blast radius (callers and tests touched). Severity rubric:

- **Blocker**: data loss or corruption, security breach, crash on the main path, wrong results in the core function.
- **High**: wrong behavior on common paths or likely edge cases, leaks in long-running processes.
- **Medium**: wrong behavior on rare but realistic inputs, error handling gaps with user-visible failure.
- **Low**: robustness, minor inconsistency.
- **Nit**: style, naming, preference.

## Phase 4: Report

Write `REVIEW.md` in the repo root (or the path the user gives) with, in this order: verdict, baseline oracle results, intent sheet and assumptions, findings table (id, severity, status, location, one line), per-root-cause detail with fix and regression test, suspicions and questions, Low backlog, coverage (what was reviewed, what was not and why), convergence metrics.

Reply in chat with counts by severity, the verdict, and the report path. Nothing else.

If the user did not ask for fixes, stop here and say the report is ready.

## Phase 5: Fix (only on request)

- Work by root cause in dependency order. Write the regression test first, watch it fail, apply the fix, watch it pass.
- Smallest change possible. No refactors, no style changes, no drive-by edits. Every extra line is new unreviewed code.
- Run the full oracle suite after each cluster.
- Then a diff-scoped re-review: changed hunks plus direct callers and tests, through the logic, failure-path and contract lenses only. Not a whole-codebase pass. Anything it finds goes through Phase 2.

## Phase 6: Convergence check

Done means all of these hold:

1. Oracles are green, or every remaining red item is documented as pre-existing.
2. No open Confirmed finding at Medium or above.
3. A fresh-context blind pass over the final code (different partitioning and lens order, no access to earlier findings) returns zero new Confirmed Medium+ findings. If it does not, fix and repeat. Maximum 3 rounds; after that, report the remainder as known risk instead of looping.
4. In thorough mode, estimate remaining bugs with capture-recapture on replicas A and B from the same snapshot: with n1 and n2 confirmed findings and m found by both, estimated total is n1 × n2 / m, and estimated unseen is that minus the union. If m is 0 with both nonzero, recall is too low to estimate; run another replica. Report the numbers.

Final wording in the report: "No confirmed Medium+ findings in N independent passes; estimated unseen Medium+ ≈ X" with the coverage gaps listed. Never "bug-free".
