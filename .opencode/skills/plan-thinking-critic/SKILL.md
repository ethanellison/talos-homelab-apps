---
name: plan-thinking-critic
description: Use this skill to evaluate implementation plans and critique the quality of underlying reasoning before execution.
---

# plan-thinking-critic

## When this skill applies

- A plan is proposed in docs, PR notes, or task comments.
- The user asks for plan review, quality check, or critical feedback.
- Execution risk is non-trivial and weak reasoning could cause rework.

## Evaluation dimensions

- Scope clarity: in-scope vs out-of-scope is explicit.
- Logical soundness: steps follow valid cause-and-effect order.
- Completeness: preconditions, dependencies, and rollback are covered.
- Verifiability: checks are command-based and agent-executable.
- Safety: secret handling, generated-file safety, and blast radius are controlled.
- Tradeoffs: alternatives are considered when risk/cost is high.

## Required plan checks

- Presence of: objective, constraints, critical path, and definition of done.
- Presence of task-level guardrails (`Must NOT`) and acceptance criteria.
- Verification strategy includes exact commands and expected outcomes.
- Changed scope maps to repository structure (`apps/`, `infrastructure/`, `clusters/`).
- Includes parent aggregation validation after direct-scope validation.
- Includes Flux reconcile steps when Flux/Kustomization/Helm behavior changes.

## Reasoning critique method

1. Identify explicit claims and unstated assumptions.
2. Test whether each step is necessary, sufficient, and correctly ordered.
3. Find missing failure paths, rollback paths, and operational checks.
4. Flag contradictions between goals, constraints, and proposed actions.
5. Propose minimal, concrete revisions to close critical gaps.

## Scoring rubric

- `plan_quality_score` from 0-10.
- Severity levels: `critical`, `high`, `medium`, `low`.
- Confidence levels: `high`, `medium`, `low`.

## Output contract

Return:

- `plan_quality_score`: integer 0-10.
- `summary`: 1-3 sentence verdict.
- `findings`: list where each item contains:
  - `category`: `scope|logic|completeness|verification|safety|tradeoff`
  - `severity`: `critical|high|medium|low`
  - `issue`: concise problem statement
  - `evidence`: concrete reference (section/line/command)
  - `fix`: minimal actionable revision
- `reasoning_critique`: short paragraph on assumption quality and logic gaps.
- `recommended_rewrite`: ordered replacement steps when major flaws exist.
- `confidence`: `high|medium|low`.

## Guardrails

- Critique the plan, not the person.
- Do not hand-wave; every major claim must have evidence.
- Prefer minimal-change fixes before suggesting full rewrites.
- If safety risk is identified, prioritize safety remediations first.

## Failure handling

- If the plan is underspecified, return low confidence and list missing inputs.
- If verification commands are absent, mark verifiability as `critical`.
- If requirements conflict, call out the conflict explicitly and propose options.
