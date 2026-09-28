# Terraform CI/CD Pipeline on GitLab + AWS: Every Step Explained

> This explains the pipeline we built (`.gitlab-ci.yml` + `gitlab-oidc.tf`) line by line: what each step does,
> **why** it's there, and what you see when it runs. It ends with the **four real problems** you'll hit while running
> it, how to recognise them and how to fix them, which is exactly what interviewers ask about.

---

## Table of Contents

1. [What the pipeline does, in plain words](#1-what-the-pipeline-does-in-plain-words)
2. [The big picture](#2-the-big-picture)
3. [Prerequisites: AWS side](#3-prerequisites-aws-side)
4. [Prerequisites: GitLab side](#4-prerequisites-gitlab-side)
5. [The pipeline file, step by step](#5-the-pipeline-file-step-by-step)
6. [One change, end to end](#6-one-change-end-to-end)
7. [How locking works (three layers)](#7-how-locking-works-three-layers)
8. [Who can do what: the security model](#8-who-can-do-what-the-security-model)
9. [Extending it: more stacks, prod, scanning, drift](#9-extending-it-more-stacks-prod-scanning-drift)
10. [The four issues you will face](#10-the-four-issues-you-will-face)
11. [Explaining it in an interview](#11-explaining-it-in-an-interview)

---

## 1. What the pipeline does, in plain words

Nobody runs `terraform apply` from a laptop. Instead:

1. A developer changes Terraform code on a branch and opens a **merge request (MR)**.
2. The pipeline **checks** the code (formatting, validity).
3. The pipeline **plans** the change with a **read-only** AWS role and shows exactly what would change.
4. A reviewer reads the plan and **approves the MR**.
5. After **merge to `main`**, the pipeline plans again and **saves** the plan.
6. A person clicks **Apply**, and the pipeline applies **exactly that saved plan** with a separate **apply role**
   that only `main` can use.

No AWS keys are stored anywhere: every job gets **temporary credentials through OIDC**.

---

## 2. The big picture

```
 Developer                 GitLab CI                                   AWS
 ─────────                 ─────────                                   ───
 git push branch
 open MR  ───────────►  [validate]  fmt + validate (no AWS access)
                        [plan]      OIDC token ──► gitlab-stage-plan role (read-only)
                                    terraform plan ──► reads state in S3 (takes .tflock)
                                    plan.txt shown to reviewers
 review + merge ─────►  main pipeline:
                        [validate]
                        [plan]      saves tfplan artifact (1 day)
                        [apply]     ▶ manual button
 click Apply ────────►              OIDC token ──► gitlab-stage-apply role (main only)
                                    terraform apply tfplan ──► changes AWS, writes state
```

**Two files make it work**

| File | Lives in | Purpose |
|---|---|---|
| `gitlab-oidc.tf` | Your IAM/bootstrap stack, applied once by an admin | Lets GitLab jobs assume AWS roles |
| `.gitlab-ci.yml` | Repo root | The pipeline |

---

## 3. Prerequisites: AWS side

These are created **once**, by an admin, usually in a bootstrap stack. They're what `gitlab-oidc.tf` contains.

### Step 3.1: The state bucket (from earlier work)

An S3 bucket per environment, for example `tfstate-stage-<ACCOUNT_ID>-apse1`, with **versioning**, **encryption** and
**public access blocked**. The stack's `backend.hcl` points at it:

```hcl
bucket       = "tfstate-stage-<ACCOUNT_ID>-apse1"
key          = "network/terraform.tfstate"
region       = "ap-southeast-1"
encrypt      = true
use_lockfile = true
```

**Why:** the pipeline runs on throwaway runners, so state must live in a shared, locked, recoverable place.

### Step 3.2: Register GitLab as an identity provider

```hcl
resource "aws_iam_openid_connect_provider" "gitlab" {
  url            = "https://gitlab.com"
  client_id_list = ["sts.amazonaws.com"]
}
```

**What it means:** "AWS, trust tokens signed by gitlab.com, as long as their audience is `sts.amazonaws.com`."
Self-managed GitLab uses its own URL here.

### Step 3.3: The plan role (read-only)

```hcl
data "aws_iam_policy_document" "plan_trust" {
  statement {
    actions = ["sts:AssumeRoleWithWebIdentity"]
    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.gitlab.arn]
    }
    condition {
      test     = "StringEquals"
      variable = "gitlab.com:aud"
      values   = ["sts.amazonaws.com"]
    }
    condition {
      test     = "StringLike"
      variable = "gitlab.com:sub"
      values   = ["project_path:${var.gitlab_project_path}:ref_type:branch:ref:*"]
    }
  }
}

resource "aws_iam_role" "stage_plan" {
  name               = "gitlab-stage-plan"
  assume_role_policy = data.aws_iam_policy_document.plan_trust.json
}

resource "aws_iam_role_policy_attachment" "stage_plan_readonly" {
  role       = aws_iam_role.stage_plan.name
  policy_arn = "arn:aws:iam::aws:policy/ReadOnlyAccess"
}
```

**Two parts to understand:**

- **Trust policy** (who can use the role): tokens from **our project**, from **any branch** (`ref:*` with
  `StringLike`), because merge requests come from feature branches.
- **Permissions** (what the role can do): **read-only**. A malicious or buggy MR can plan, but can't change anything.

### Step 3.4: Let the plan role take the state lock

```hcl
data "aws_iam_policy_document" "stage_plan_lock" {
  statement {
    actions   = ["s3:PutObject", "s3:DeleteObject"]
    resources = ["arn:aws:s3:::${var.stage_state_bucket}/*.tflock"]
  }
}

resource "aws_iam_role_policy" "stage_plan_lock" {
  name   = "state-lock"
  role   = aws_iam_role.stage_plan.id
  policy = data.aws_iam_policy_document.stage_plan_lock.json
}
```

**Why:** `terraform plan` also **locks** the state. `ReadOnlyAccess` can read the state file, but locking means
**writing and deleting** the `.tflock` object. Without this, plan fails (Issue 2 below).

### Step 3.5: The apply role (only `main`)

```hcl
condition {
  test     = "StringEquals"
  variable = "gitlab.com:sub"
  values   = ["project_path:${var.gitlab_project_path}:ref_type:branch:ref:main"]
}
```

With write permissions (admin in the lab; scoped down in real projects) and `max_session_duration = 7200` for long
applies.

**Why `StringEquals` on `main`:** only pipelines on the **protected `main` branch** can get write access. A feature
branch job asking for this role is refused by AWS itself, whatever the YAML says.

---

## 4. Prerequisites: GitLab side

| Setting | Where | Why |
|---|---|---|
| `AWS_ACCOUNT_ID` variable | Settings → CI/CD → Variables | Used to build role ARNs; not a secret |
| **Protected branch** `main` | Settings → Repository → Protected branches | Only merges via MR; ties the apply role to reviewed code |
| **Protected environment** `stage` | Settings → CI/CD → Protected environments | Controls who can run the deploy job (approvals on Premium) |
| MR approval rules / CODEOWNERS | Settings → Merge requests | Someone must review the plan before merge |
| Runners | Shared or your own | Need outbound HTTPS to AWS STS and S3 |

No `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` variables: that's the point of OIDC. If old ones exist, remove them,
because environment credentials can take precedence over the OIDC role and confuse debugging.

---

## 5. The pipeline file, step by step

### Step 5.1: Stages

```yaml
stages:
  - validate
  - plan
  - apply
```

**What:** the order jobs run in. A stage only starts when the previous one succeeds.
**Why three:** cheap checks first (fail fast, no AWS access), then the plan everyone reviews, then the risky apply.

### Step 5.2: When pipelines run at all (`workflow`)

```yaml
workflow:
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
```

**What:** create a pipeline for **merge requests** and for the **default branch (`main`)**, nothing else.
**Why:** without this, a push to a branch with an open MR creates **two** pipelines (branch + MR) that both plan and
fight for the state lock. It also stops pipelines on random branches without MRs.

### Step 5.3: Global variables

```yaml
variables:
  TF_ROOT: live/stage/network
  AWS_REGION: ap-southeast-1
  TF_IN_AUTOMATION: "true"
  TF_INPUT: "false"
```

| Variable | Why |
|---|---|
| `TF_ROOT` | The stack folder this pipeline manages; every job `cd`s into it |
| `AWS_REGION` | Region for the provider and STS |
| `TF_IN_AUTOMATION` | Terraform trims "next steps" hints meant for humans |
| `TF_INPUT=false` | Terraform **fails instead of waiting** for a prompt (a missing variable would otherwise hang the job forever) |

### Step 5.4: Default image

```yaml
default:
  image:
    name: hashicorp/terraform:1.10
    entrypoint: [""]
```

**What:** every job runs in the official Terraform image.
**Why `entrypoint: [""]`:** the image's entrypoint is `terraform`, so GitLab's shell commands would be passed to
Terraform as arguments and fail. Clearing it gives a normal shell.
**Why pin the tag:** the same Terraform version for every run; state written by a newer version can't be used by an
older one. Match it to `required_version`.

### Step 5.5: "Only when this stack changed" (YAML anchor)

```yaml
.stack_changes: &stack_changes
  - $TF_ROOT/**/*
  - modules/**/*
```

**What:** a reusable list of paths (`&stack_changes` defines it, `*stack_changes` reuses it).
**Why:** jobs only run when **this stack or a shared module** changed. A README change doesn't trigger an AWS plan.
A module change **does**, because this stack uses the module.

### Step 5.6: The OIDC login template (`.aws_oidc`)

```yaml
.aws_oidc:
  id_tokens:
    GITLAB_OIDC_TOKEN:
      aud: sts.amazonaws.com
  variables:
    AWS_WEB_IDENTITY_TOKEN_FILE: /tmp/web-identity-token
    AWS_ROLE_SESSION_NAME: gitlab-$CI_PROJECT_ID-$CI_JOB_ID
  before_script:
    - echo "$GITLAB_OIDC_TOKEN" > "$AWS_WEB_IDENTITY_TOKEN_FILE"
    - cd "$TF_ROOT"
    - terraform init -backend-config=backend.hcl
```

A hidden job (starts with `.`): never runs itself; `plan` and `apply` reuse it with `extends`.

| Line | What it does | Why |
|---|---|---|
| `id_tokens: GITLAB_OIDC_TOKEN` | GitLab creates a **signed JWT** for this job and puts it in that variable | This is the identity AWS will check |
| `aud: sts.amazonaws.com` | Sets the token's audience | Must match `client_id_list` on the AWS OIDC provider |
| `AWS_WEB_IDENTITY_TOKEN_FILE` | Where the token file will be | The AWS SDK (inside Terraform) reads the token from a **file** |
| `AWS_ROLE_SESSION_NAME` | Session name with project and job ID | CloudTrail shows **which job** made each change |
| `echo ... > file` | Writes the token to that file | Needed because the SDK wants a file, not a variable |
| `cd "$TF_ROOT"` | Enter the stack folder | The backend and code are there |
| `terraform init -backend-config=backend.hcl` | Downloads providers/modules and connects to the S3 state | Every job starts on a clean runner |

**What happens behind the scenes:** each job also sets `AWS_ROLE_ARN`. When Terraform starts, the AWS provider and the
S3 backend see `AWS_ROLE_ARN` + `AWS_WEB_IDENTITY_TOKEN_FILE`, call **`AssumeRoleWithWebIdentity`**, and AWS checks:
signed by gitlab.com? audience correct? `sub` (project + branch) allowed by the trust policy? If yes, it returns
**temporary credentials** (about an hour). No keys are ever stored.

### Step 5.7: Job 1 — `validate`

```yaml
validate:
  stage: validate
  script:
    - terraform fmt -check -recursive
    - cd "$TF_ROOT"
    - terraform init -backend=false
    - terraform validate
  rules:
    - changes: *stack_changes
```

| Command | What | If it fails |
|---|---|---|
| `terraform fmt -check -recursive` | Checks every `.tf` file in the repo is formatted; changes nothing | Lists unformatted files; developer runs `terraform fmt -recursive` |
| `terraform init -backend=false` | Downloads providers/modules **without** connecting to state | — |
| `terraform validate` | Syntax, types, references, required arguments | Shows the file and line |

**Why no AWS login:** validation doesn't need AWS, so it's fast, free and safe, and fails before anything touches the
cloud.

### Step 5.8: Job 2 — `plan`

```yaml
plan:
  stage: plan
  extends: .aws_oidc
  variables:
    AWS_ROLE_ARN: arn:aws:iam::$AWS_ACCOUNT_ID:role/gitlab-stage-plan
  resource_group: stage-network
  script:
    - terraform plan -lock-timeout=5m -out=tfplan
    - terraform show -no-color tfplan > plan.txt
  artifacts:
    paths:
      - $TF_ROOT/tfplan
      - $TF_ROOT/plan.txt
    expire_in: 1 day
  rules:
    - changes: *stack_changes
```

| Part | What | Why |
|---|---|---|
| `extends: .aws_oidc` | Reuses the OIDC login + `init` | No duplicated login code |
| `AWS_ROLE_ARN: …gitlab-stage-plan` | The **read-only** role | MR pipelines can never change infrastructure |
| `resource_group: stage-network` | Only **one** job with this name runs at a time across all pipelines; others **wait** | Two MRs or an MR + main pipeline don't collide on the state lock |
| `terraform plan -lock-timeout=5m -out=tfplan` | Refreshes, compares, saves the plan file; waits up to 5 min for a lock | The saved file is what will be applied later |
| `terraform show -no-color tfplan > plan.txt` | Human-readable plan | Reviewers read it in the job log or artifacts |
| `artifacts` | Keeps `tfplan` and `plan.txt` after the job | `apply` needs the exact file; people need the text |
| `expire_in: 1 day` | Deletes artifacts after a day | **Plan files can contain sensitive values**, and an old plan is stale anyway |

**What reviewers look for in `plan.txt`:** the summary line (`Plan: X to add, Y to change, Z to destroy`), any
`- destroy`, and especially **`-/+ … # forces replacement`** on important resources.

**Important:** the plan runs **twice**: in the MR pipeline (for review) and again in the `main` pipeline after merge.
The `main` plan is the one that gets applied, because the merged code may include other people's changes too.

### Step 5.9: Job 3 — `apply`

```yaml
apply:
  stage: apply
  extends: .aws_oidc
  variables:
    AWS_ROLE_ARN: arn:aws:iam::$AWS_ACCOUNT_ID:role/gitlab-stage-apply
  needs:
    - job: plan
      artifacts: true
  resource_group: stage-network
  environment:
    name: stage
  interruptible: false
  script:
    - terraform apply -lock-timeout=10m tfplan
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      changes: *stack_changes
      when: manual
```

| Part | What | Why |
|---|---|---|
| `AWS_ROLE_ARN: …gitlab-stage-apply` | The **write** role | AWS only allows it for `main` (trust policy) |
| `needs: plan, artifacts: true` | Downloads `tfplan` from this pipeline's plan job | Applies the **exact reviewed plan** |
| `resource_group: stage-network` | Same group as `plan` | A plan and an apply for this stack never run at the same time |
| `environment: stage` | Marks it as a deployment | GitLab shows deployment history; protected environment rules decide who can run it |
| `interruptible: false` | Never auto-cancelled by a newer pipeline | Killing an apply mid-way leaves a **stale lock** and half-applied changes |
| `terraform apply tfplan` | Applies the saved plan, no prompt | If state changed since the plan, Terraform **refuses** ("stale plan") |
| `if: main` + `when: manual` | Only on `main`, only when someone clicks ▶ | A human confirms after reading the plan |

**What you see:** the pipeline stops with a ▶ **play button** on `apply`. After clicking, the log shows each resource
being created or changed, then `Apply complete! Resources: X added, Y changed, Z destroyed.` The new state is in S3 and
the `.tflock` object is gone.

---

## 6. One change, end to end

**Example:** add a tag to the stage VPC.

| # | Who | Action | What happens |
|---|---|---|---|
| 1 | Developer | Branch `add-vpc-tag`, edit `live/stage/network/main.tf`, push, open MR | `workflow` allows an MR pipeline; `changes` matches the stack folder |
| 2 | Pipeline | `validate` | fmt + validate pass in seconds |
| 3 | Pipeline | `plan` (plan role, `sub` = `…:ref:add-vpc-tag`, allowed by `ref:*`) | `Plan: 0 to add, 1 to change, 0 to destroy`, `~ tags` on the VPC |
| 4 | Reviewer | Opens `plan.txt`, checks no destroys/replacements, approves | — |
| 5 | Developer | Merges | A new pipeline starts on `main` |
| 6 | Pipeline | `validate`, `plan` on `main` | New `tfplan` saved (includes anything else merged) |
| 7 | Approver | Reads the `main` plan, clicks ▶ on `apply` | Waits if another job holds `stage-network` |
| 8 | Pipeline | `apply` (apply role, `sub` = `…:ref:main`) | Tag added; state written to S3; lock released |
| 9 | Anyone | Checks AWS / CloudTrail | Change shows session `gitlab-<project>-<job>` |

---

## 7. How locking works (three layers)

| Layer | Mechanism | Protects against |
|---|---|---|
| **1. Terraform** | `use_lockfile = true`: `network/terraform.tfstate.tflock` exists while a plan/apply runs | Any two Terraform runs (CI or laptop) writing the same state |
| **2. GitLab** | `resource_group: stage-network` | Two pipeline jobs for this stack even starting together; they **queue** instead of failing |
| **3. Process** | `interruptible: false`, `when: manual`, protected `main` and environment | Cancelled applies leaving stale locks; unreviewed or parallel applies |

**Why all three:** Terraform's lock alone would make the second job **fail** on a lock error; `resource_group` makes it
**wait** its turn. And the lock only helps if runs aren't killed halfway.

---

## 8. Who can do what: the security model

| Actor | Can plan? | Can apply? | Enforced by |
|---|---|---|---|
| MR pipeline from a feature branch | ✅ read-only | ❌ | AWS trust policy (`ref:*` only on plan role) |
| `main` pipeline | ✅ | ✅ after manual click | Apply role trusts only `ref:main`; `when: manual` |
| Someone editing `.gitlab-ci.yml` in an MR to use the apply role | — | ❌ | AWS refuses: the token's `sub` is the feature branch |
| Another GitLab project | ❌ | ❌ | `project_path` in `sub` |
| A leaked token | Expires in minutes | — | Short-lived OIDC credentials |
| Developers on laptops | Via their own SSO/deployer role (dev only ideally) | Not for stage/prod | IAM + process |

The key idea: **security lives in AWS trust policies, not in the YAML.** Even if someone changes the pipeline file,
AWS decides which role a job can get.

---

## 9. Extending it: more stacks, prod, scanning, drift

### More stacks: reusable templates

```yaml
.plan:
  stage: plan
  extends: .aws_oidc
  script:
    - terraform plan -lock-timeout=5m -out=tfplan
    - terraform show -no-color tfplan > plan.txt
  artifacts:
    paths: [$TF_ROOT/tfplan, $TF_ROOT/plan.txt]
    expire_in: 1 day

.apply:
  stage: apply
  extends: .aws_oidc
  interruptible: false
  script:
    - terraform apply -lock-timeout=10m tfplan

plan:stage-servers:
  extends: .plan
  variables:
    TF_ROOT: live/stage/servers
    AWS_ROLE_ARN: arn:aws:iam::$AWS_ACCOUNT_ID:role/gitlab-stage-plan
  resource_group: stage-servers
  rules:
    - changes: [live/stage/servers/**/*, modules/**/*]

apply:stage-servers:
  extends: .apply
  variables:
    TF_ROOT: live/stage/servers
    AWS_ROLE_ARN: arn:aws:iam::$AWS_ACCOUNT_ID:role/gitlab-stage-apply
  needs:
    - job: plan:stage-servers
      artifacts: true
  resource_group: stage-servers
  environment: { name: stage }
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      changes: [live/stage/servers/**/*, modules/**/*]
      when: manual
```

Each stack gets its own `resource_group` (stacks lock independently). When one stack depends on another
(servers reads network's SSM values), keep **one layer per MR**, or order jobs with `needs`.

### Prod

Same jobs with `gitlab-prod-plan` / `gitlab-prod-apply` roles (in the prod account if separate), `environment: prod`
with **required approvals**, and a prod job that only runs after stage succeeded.

### Security scanning in `validate`

```yaml
lint:
  stage: validate
  image: { name: ghcr.io/terraform-linters/tflint:latest, entrypoint: [""] }   # pin a version
  script:
    - tflint --init && tflint --recursive
  rules:
    - changes: *stack_changes

checkov:
  stage: validate
  image: { name: bridgecrew/checkov:latest, entrypoint: [""] }                 # pin a version
  script:
    - checkov -d "$TF_ROOT" --quiet
  allow_failure: true          # advisory at first; make it blocking once findings are baselined
  rules:
    - changes: *stack_changes
```

### Nightly drift detection

Create a **pipeline schedule** (CI/CD → Schedules) on `main`, and add:

```yaml
drift:
  stage: plan
  extends: .aws_oidc
  variables:
    AWS_ROLE_ARN: arn:aws:iam::$AWS_ACCOUNT_ID:role/gitlab-stage-plan
  script:
    - set +e
    - terraform plan -lock=false -detailed-exitcode -no-color > drift.txt; CODE=$?
    - cat drift.txt
    - if [ "$CODE" -eq 2 ]; then echo "Drift detected"; exit 1; fi
    - exit $CODE
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
```

And add `- if: $CI_PIPELINE_SOURCE == "schedule"` with `when: never` as the **first** rule of `validate`, `plan` and
`apply`, so the nightly pipeline only runs the drift check. (`-detailed-exitcode`: 0 = no changes, 1 = error,
2 = drift. `-lock=false` is fine because this job only reads.)

---

## 10. The four issues you will face

### Issue 1: OIDC login fails — "Not authorized to perform sts:AssumeRoleWithWebIdentity"

**What you see** (in `terraform init` or `plan`):

```
Error: ... operation error STS: AssumeRoleWithWebIdentity, https response error StatusCode: 403,
api error AccessDenied: Not authorized to perform sts:AssumeRoleWithWebIdentity
```

or `InvalidIdentityToken: Incorrect token audience`, or `No OpenIDConnect provider found in your account`.

**Root causes**

| Cause | Detail |
|---|---|
| `sub` doesn't match | Wrong `project_path` (group/subgroup/project must be exact); apply role uses `StringEquals` on `main` but the job runs on a feature branch or a tag; plan role uses `StringEquals` where `StringLike` with `*` was needed |
| `aud` doesn't match | `aud:` in `id_tokens` ≠ `client_id_list` on the provider |
| Provider URL wrong | Self-managed GitLab: provider must use **your** GitLab URL, and condition keys change from `gitlab.com:sub` to `<your-host>:sub` |
| Old access keys still set | `AWS_ACCESS_KEY_ID` variables in GitLab take precedence and the job isn't even using OIDC |

**How to debug**

```yaml
debug-oidc:
  image: alpine
  id_tokens:
    GITLAB_OIDC_TOKEN: { aud: sts.amazonaws.com }
  script:
    - apk add --no-cache jq coreutils
    - echo "$GITLAB_OIDC_TOKEN" | cut -d. -f2 | base64 -d 2>/dev/null | jq '{aud, sub, iss}'
  when: manual
```

Compare the printed `sub`, `aud` and `iss` with the trust policy character by character. Never print the whole token.

**Fix and prevention:** correct the trust policy (for MR plans: `StringLike` with `ref:*`; for apply: `StringEquals`
on `ref:main`), match `aud`, remove old key variables, and keep the trust policy in Terraform so it's reviewed.

**Interview phrasing:** "Our first OIDC runs failed with AssumeRoleWithWebIdentity denied. Decoding the job token showed
the `sub` was `project_path:…:ref:feature-x`, but our plan role only trusted `ref:main`. We split into a plan role
trusting any branch and an apply role trusting only main, which also improved security."

---

### Issue 2: State lock problems — lock errors and stale locks

**What you see**

```
Error: Error acquiring the state lock
Lock Info:
  ID:        3b8c2f1e-7a4d-...
  Operation: OperationTypeApply
  Who:       root@runner-abc123
```

Three different situations produce it:

| Situation | Cause |
|---|---|
| **Two jobs at once** | An MR plan and a `main` apply ran together (before `resource_group` was added) |
| **Stale lock** | A job was **cancelled** or a runner **died** mid-apply, so the `.tflock` object was never deleted; every later job fails |
| **Plan can't lock at all** (`AccessDenied` on the `.tflock` key) | The read-only plan role could read the state but not **write/delete** the lock file |

**Fix**

1. Concurrent jobs: add `resource_group` per stack (jobs queue) and `-lock-timeout` (brief waits instead of failures).
2. Stale lock: confirm in GitLab that **no job is running** for that stack, read `Who` and `Created`, then:

```bash
terraform force-unlock 3b8c2f1e-7a4d-...
# or find the ID:  aws s3 cp s3://<bucket>/<key>.tflock - | jq -r .ID
```

   Then re-run the pipeline; check the plan carefully, because the interrupted apply may have **partially** changed
   things (and tainted a resource).
3. Plan role: add `s3:PutObject` + `s3:DeleteObject` on `*.tflock` (Step 3.4).

**Prevention:** `interruptible: false` on apply, never cancel running applies, `resource_group`, lock permissions in the
plan role, and applies only from CI.

**Interview phrasing:** "Someone cancelled a running apply to push a fix, which left a stale lock that blocked every
pipeline. I confirmed nothing was running, force-unlocked using the ID from the lock info, reviewed the partial changes
in a fresh plan, and then made apply jobs non-interruptible and added a GitLab resource group per stack."

---

### Issue 3: "Saved plan is stale" — the plan no longer matches state

**What you see** in `apply`:

```
Error: Saved plan is stale

The given plan file can no longer be applied because the state was changed by another
operation after the plan was created.
```

**Root causes**

- Two MRs merged close together: pipeline A planned, pipeline B planned **and applied** first, so A's saved plan is based
  on old state.
- Someone ran `apply` from a laptop, or another stack job wrote the same state, between plan and apply.
- The manual apply was clicked **hours later**, after other changes.
- Related: the plan artifact **expired** (`expire_in: 1 day`), so `apply` can't find `tfplan`.

**Fix:** this error is **Terraform protecting you**: it refuses to apply something nobody reviewed. **Re-run the
pipeline** on `main` (new plan → review → apply).

**Prevention:** `resource_group` per stack (serialises plan and apply), apply soon after the plan, merge MRs one at a
time for the same stack (or use a merge train), and no laptop applies to shared environments.

**Interview phrasing:** "Two MRs for the same stack merged within minutes; the second apply failed with 'saved plan is
stale'. That's by design: Terraform won't apply a plan made against old state. We re-ran the pipeline, reviewed the new
plan and applied it, then serialised the stack with a resource group and asked the team to merge one change per stack at
a time."

---

### Issue 4: Works locally, fails in CI — provider lock file checksum mismatch

**What you see** in `terraform init` on the runner:

```
Error: Failed to install provider

Error while installing hashicorp/aws v6.x.x: the current package for
registry.terraform.io/hashicorp/aws 6.x.x doesn't match any of the checksums previously
recorded in the dependency lock file
```

**Root cause:** the developer ran `terraform init` on a **Mac (darwin_arm64)**, which recorded checksums only for that
platform in `.terraform.lock.hcl`. The GitLab runner is **Linux (linux_amd64)**, so there's no matching checksum.

**Fix**

```bash
terraform providers lock \
  -platform=linux_amd64 \
  -platform=darwin_arm64 \
  -platform=darwin_amd64
git add .terraform.lock.hcl && git commit -m "Lock providers for all platforms"
```

**Prevention:** commit the lock file, always lock every platform the team and CI use, pin the Terraform image in CI to
the team's version, and upgrade providers in their own MR (`terraform init -upgrade` + `providers lock`).

**Interview phrasing:** "A provider upgrade from a Mac broke every pipeline at `init` with a checksum mismatch, because
the lock file only had darwin hashes and our runners are Linux. I regenerated the lock file for linux_amd64 and the Mac
platforms, and we added that to our upgrade checklist."

---

### Bonus: other issues you may meet

| Issue | Cause | Fix |
|---|---|---|
| Plan passes, apply fails with `AccessDenied` | Plan role is read-only; apply role lacks a permission (or an SCP blocks it) | Grant the specific action to the apply role; test applies in dev first |
| Job hangs, then times out | A variable has no value and Terraform waits for input | `TF_INPUT=false` so it fails immediately; provide the value |
| `ParameterNotFound` / missing outputs | A dependent stack planned before the stack it reads from was applied | Apply in order; one layer per MR |
| `ExpiredToken` in very long applies | Role session shorter than the apply | Raise `max_session_duration` on the apply role |
| Jobs don't run after a change | `rules: changes` paths don't match the changed files | Check paths; module changes must be included |
| Secrets visible in job logs | `echo` of variables, `TF_LOG=DEBUG` in shared logs | Masked variables, never debug-log in shared pipelines |

---

## 11. Explaining 

> "Our Terraform pipeline runs in GitLab with three stages. **Validate** runs `fmt -check` and `validate` with no cloud
> access. **Plan** logs into AWS with GitLab OIDC: the job gets a signed token, and Terraform exchanges it through
> `AssumeRoleWithWebIdentity` for temporary credentials on a read-only plan role that trusts any branch of our project,
> so merge requests can plan but never change anything. The plan is saved as a short-lived artifact and shown to
> reviewers. After merge, `main` plans again, and a **manual apply** job uses a separate apply role that AWS only allows
> for the protected main branch, applying exactly the saved plan. Locking has three layers: Terraform's S3 lock file, a
> GitLab `resource_group` per stack so jobs queue, and non-interruptible applies so we never leave stale locks.
>
> The issues we hit were: OIDC failures from a `sub` mismatch, which led us to split plan and apply roles; a stale lock
> after someone cancelled an apply; 'saved plan is stale' when two merges raced, which Terraform correctly blocked; and a
> provider checksum mismatch because the lock file only had Mac hashes. Each fix became part of the pipeline design."
