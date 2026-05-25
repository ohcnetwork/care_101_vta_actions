# Copilot review instructions for `care_101_vta_actions`

This repository is a **reference store for CARE course evaluation actions**. Keep reviews grounded in how these actions are used with **Pupilfirst LMS course targets**, not just how a standalone GitHub Actions repository would normally behave.

## Repository purpose

- `care_101_vta_actions` is used to store and review action/workflow definitions for course-target evaluations.
- Files may be organized by assignment area (for example, `facility_management/20823.yml`) instead of being placed under `.github/workflows/`.
- Do **not** assume that every YAML file in this repo is meant to execute directly inside this repository unless the change explicitly says so.
- When reviewing, prefer repository-context comments over generic suggestions about moving files into `.github/workflows`.

## How the evaluation flow works

- A learner makes a submission on a **school course target** in Pupilfirst LMS.
- That submission provides artifacts such as `submission.json` and is evaluated through GitHub Actions automation.
- The action reads submission data, validates the learner's work against CARE requirements, writes a learner-facing `report.json`, and sends the result back to Pupilfirst using LMS integration actions.
- Reviews should preserve this end-to-end contract: **submission input -> CARE validation -> `report.json` -> grade/report back to LMS**.

## Review priorities

Focus review feedback on correctness, learner experience, and LMS integration:

- Ensure every failure path that can occur before grading still produces a valid `report.json`.
- Prefer helpful learner-facing feedback over raw shell, git, or jq failures.
- Treat `submission.json`, `setup_data.json`, and step outputs as integration contracts; changing their shape or meaning is a breaking change unless explicitly intended.
- Use configured environment values and secrets (for example, `CARE_FE_BASE_URL`, `CARE_API_BASE_URL`) instead of hardcoded hosts or credentials.
- Keep checks deterministic and assignment-specific so reviewers can easily map each rule back to the course target.
- Preserve idempotency when updating shared state such as `setup_data.json` on `main`.
- Be careful with suggestions that add broad abstractions or reusable layers; reuse is welcome only when it clearly reduces duplication without obscuring assignment logic.

## What to avoid in reviews

- Do not flag the non-`.github/workflows` layout by default; this repo is primarily for storing and reviewing action definitions.
- Do not suggest refactors that make assignment rules harder for course authors to read or audit.
- Do not recommend changes that weaken learner feedback, skip grading artifacts, or hide LMS/reporting failures.
- Do not introduce secrets into files, logs, examples, or review suggestions.

## Change style expectations

- Keep changes small and surgical.
- Prefer explicit learner-facing messages when handling invalid submissions or CARE API failures.
- When persisting values back to shared files, preserve unrelated keys and avoid unnecessary commits.
- If a review suggestion depends on how actions are synced or executed outside this repo, call that dependency out explicitly instead of assuming standard GitHub-only behavior.
