# Copilot review instructions — care_101_vta_actions

This repository is a reference store of GitHub Actions workflow YAML templates used by Pupilfirst VTA to evaluate CARE course submissions.

These workflows do not run as CI in this repository. Pupilfirst copies the workflow YAML and `submission.json` into per-student private repositories, where GitHub Actions executes them on `submission-*` branches.

Use these instructions when reviewing workflow PRs.

## Review priorities

Prioritize:

- runtime correctness
- grading correctness
- student feedback quality
- idempotency
- secret safety
- safe GitHub Actions usage
- correct CARE/Pupilfirst integration

Do not comment on cosmetic YAML style unless it can cause runtime failure, unsafe behavior, or reviewer confusion.

## Workflow flavors

Every target workflow must be clearly one of these two flavors:

1. Provisioning workflow
2. Evaluation workflow

Do not allow a workflow to mix both flavors.

## Provisioning workflows

Provisioning workflows bootstrap the student’s CARE environment.

They usually:

- parse `submission.json`
- create or reuse the student’s CARE user
- add the user to the required organization
- persist `setup_data.json` to `main`
- send credentials email
- post feedback to Pupilfirst
- do not grade the submission

Provisioning workflows must use:

- `ohc-vta/care-actions/pupilfirst-create-feedback@main`
- unprefixed LMS secrets:
  - `REVIEW_END_POINT`
  - `REVIEW_BOT_USER_TOKEN`
- skip flag convention:
  - `env.skip`
- `continue-on-error: true` for credential email
- multi-line feedback through `env: FEEDBACK_BODY`, never through `with:`

## Evaluation workflows

Evaluation workflows verify that the student performed a task in CARE.

They usually:

- read `setup_data.json` from `main`
- parse `submission.json`
- validate the submitted CARE frontend URL
- extract resource UUIDs
- validate facility/resource ownership
- verify CARE API requirements
- write `report.json`
- grade pass/fail
- optionally persist newly discovered IDs to `setup_data.json`

Evaluation workflows must use:

- `pupilfirst/grade-action@v1`
- `pupilfirst/report-action@v1`
- prefixed LMS secrets:
  - `PF_REVIEW_END_POINT`
  - `PF_REVIEW_BOT_USER_TOKEN`
- skip flag convention:
  - `env.skip_checks`
- `grade-action` with `if: always()`
- `report.json` generated using `jq -n --arg status --arg feedback`

## Blocking review comments

Flag a blocking review comment if any of these are true:

- Workflow file is not named `<target_id>.yml`.
- Workflow is placed under the wrong module folder.
- Workflow flavor is unclear.
- Provisioning and evaluation conventions are mixed.
- Runtime-generated multi-line feedback is passed through `with:`.
- Evaluation workflow does not write `report.json` for every pass/fail/error branch.
- Evaluation workflow does not run `grade-action` with `if: always()`.
- `report-action` success arm is missing.
- `report-action` error arm is missing.
- Wrong LMS secret pair is used for the workflow flavor.
- Post-validation steps are not guarded by the workflow’s skip flag.
- A new secret is introduced without noting that it must be added to the student template repository.
- Student-facing feedback exposes secrets, tokens, passwords, raw API error bodies, or internal IDs students should not see.
- Workflow uses `git push --force`.
- Workflow deletes or mutates `submission.json`.
- Workflow depends on a hardcoded cohort size, course slug, email domain, or student count.
- Shell code uses `set -e` without `set -o pipefail` when pipeline exit status matters.
- Untrusted submission data is interpolated unsafely into shell commands.

## File naming rules

Target workflow files must be named:

```txt
<target_id>.yml
````

Examples:

```txt
facility_management/20823.yml
facility_management/20824.yml
clinical_management/20901.yml
```

Descriptive draft names such as `get-credentials.yml` or `create-facility-evaluation.yml` must be renamed before merge.

## Multi-line content rules

Never pass runtime-generated multi-line feedback or error messages through `with:` inputs.

Bad:

```yaml
with:
  feedback: ${{ steps.build.outputs.feedback }}
```

Good:

```yaml
env:
  FEEDBACK_BODY: ${{ steps.build.outputs.feedback }}
```

Use heredoc when writing multi-line outputs to `GITHUB_OUTPUT`:

```yaml
{
  echo "feedback<<__EOF__"
  echo "$FEEDBACK"
  echo "__EOF__"
} >> "$GITHUB_OUTPUT"
```

## setup_data.json rules

Provisioning workflows create or initialize `setup_data.json`.

For provisioning workflows:

```bash
git add setup_data.json
git diff --cached --quiet -- setup_data.json
```

Evaluation workflows consume an existing `setup_data.json`.

For evaluation workflows:

```bash
git fetch origin main
git checkout origin/main -- setup_data.json
git diff --quiet setup_data.json
```

Use the provisioning pattern for untracked files. Plain `git diff --quiet` can miss newly created files.

## Evaluation failure behavior

Evaluation workflows must distinguish these cases:

### Missing `user.id`

Fail hard because bootstrap has not run or setup is broken.

### Missing prerequisite IDs

Write friendly failure feedback and set `skip_checks=true`.

Example feedback:

```md
Please pass the **Create Facility** target first.
```

Do not `exit 1` for recoverable missing prerequisites.

### Invalid submitted URL

Write failure feedback and skip API checks.

### Wrong facility or parent ownership

Write failure feedback and skip further checks.

Examples:

* submitted facility UUID does not match expected facility ID
* submitted bed does not belong to expected room
* submitted room does not belong to expected facility

### CARE API call failed

Write controlled feedback without leaking raw sensitive API details.

### Checks failed

Show a clear checklist of passed and failed requirements.

### Checks passed

Write success report and persist required IDs to `setup_data.json`.

## CARE API check rules

For `ohc-vta/care-actions/check-api@main`:

* `endpoint` must call the correct CARE API resource.
* `method` must be explicit.
* `jq-filter` should return the count of passed boolean assertions.
* `expected` must equal the total number of assertions.
* `output-jq-filter` should extract per-check booleans and any IDs needed later.
* `label` must be human-readable.

Example pattern:

```yaml
jq-filter: |
  [
    .name == env.EXPECTED_NAME,
    .facility.id == env.EXPECTED_FACILITY_ID
  ] | map(select(.)) | length
assertion: equals
expected: "2"
```

## Idempotency rules

Students may resubmit the same target.

Provisioning workflows must detect or reuse existing side effects where possible.

For user creation:

* check username availability before creating the user
* do not blindly create duplicate users
* do not fail resubmissions that can safely reuse existing setup

Evaluation workflows should be read-only against CARE and only persist IDs to `setup_data.json` after success.

## Secret rules

Use the correct LMS secret pair for the workflow flavor.

Provisioning feedback must use:

```txt
REVIEW_END_POINT
REVIEW_BOT_USER_TOKEN
```

Evaluation grading/reporting must use:

```txt
PF_REVIEW_END_POINT
PF_REVIEW_BOT_USER_TOKEN
```

Any new secret must be configured in the student template repository, not only in `care_101_vta_actions`.

Never expose secrets in:

* logs
* feedback
* `report.json`
* committed files
* email content

## Git rules

Do not allow:

```bash
git push --force
```

Do not allow deleting or mutating:

```txt
submission.json
```

When committing `setup_data.json`, use a safe commit message such as:

```txt
Update setup data [skip ci]
```

## YAML and shell safety

Flag these patterns:

* `description: |` inside composite action metadata
* runtime multi-line values passed through `with:`
* unquoted shell variables
* unsafe interpolation of submitted values
* missing `set -o pipefail` when pipelines matter
* unguarded post-validation steps
* steps that continue after validation failure without checking skip flags

Prefer:

```bash
set -euo pipefail
```

when the script expects strict failure handling.

## Email rules for provisioning workflows

Credential email steps must use:

```yaml
continue-on-error: true
```

If email sending fails:

* do not fail the whole provisioning workflow
* still post feedback explaining that setup succeeded but email delivery failed
* avoid locking the student out of a non-resubmittable provisioning target

Do not duplicate email template content already generated by `send-email`.

Do not emit an empty `Bcc:` header.

## Student feedback rules

Student-facing feedback must be:

* clear
* actionable
* safe
* friendly
* specific enough to fix the issue

Do not expose:

* secrets
* raw API error bodies
* tokens
* passwords
* internal-only IDs
* implementation details students do not need

Prefer feedback like:

```md
We could not verify the room because it does not belong to your assigned facility.
Please submit the room URL from your own CARE facility.
```

Avoid feedback like:

```md
API returned 500 with body: ...
```

## Composite action versioning

All `ohc-vta/care-actions/*` usages are pinned to `@main` today.

When a workflow depends on a newly added composite action, verify that the action already exists on `main`.

If the workflow depends on a not-yet-merged composite action, flag it because the workflow will fail at runtime.

## Final checklist for every new target workflow

Check all of these before approving:

* [ ] Filename is `<target_id>.yml`.
* [ ] File is under the correct module folder.
* [ ] Workflow flavor is clear: provisioning or evaluation.
* [ ] Provisioning and evaluation conventions are not mixed.
* [ ] Correct LMS secret pair is used.
* [ ] Correct skip flag is used consistently.
* [ ] Every post-validation step is guarded.
* [ ] Multi-line feedback is not passed through `with:`.
* [ ] `setup_data.json` read/write pattern matches the workflow flavor.
* [ ] Student-facing feedback is clear and safe.
* [ ] Evaluation workflow writes `report.json` in every branch.
* [ ] Evaluation workflow grades with `if: always()`.
* [ ] `report-action` has both success and error arms.
* [ ] Provisioning email uses `continue-on-error: true`.
* [ ] No force-push.
* [ ] No deletion or mutation of `submission.json`.
* [ ] No cohort-specific assumptions.
* [ ] No new secrets without student template repo note.
