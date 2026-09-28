# Terraform on AWS: Complete Notes

> Everything covered in our sessions, in one place: concepts, commands, patterns, scenario questions and
> interview lines. Read it top to bottom once, then use the Table of Contents to revise specific topics.
>
> Companion files from the same sessions:
> - `terraform-revision/README.md`: 10 hands-on POCs (Day 0, 1, 2)
> - `terraform-aws-platform/`: the full dev/stage/prod repo with modules and CI
> - `gitlab-terraform-stage/`: GitLab CI pipeline + OIDC roles
> - `terraform-project-story/README.md`: "tell me about a project" interview script

---

## Table of Contents

1. [Core model: how Terraform thinks](#1-core-model-how-terraform-thinks)
2. [Workflow and CLI commands](#2-workflow-and-cli-commands)
3. [Project structure and standard files](#3-project-structure-and-standard-files)
4. [Language building blocks](#4-language-building-blocks)
5. [Variables, tfvars and precedence](#5-variables-tfvars-and-precedence)
6. [Expressions, conditionals, loops and dynamic blocks](#6-expressions-conditionals-loops-and-dynamic-blocks)
7. [Built-in functions](#7-built-in-functions)
8. [Data sources](#8-data-sources)
9. [Providers and multi-region](#9-providers-and-multi-region)
10. [Execution order: the DAG](#10-execution-order-the-dag)
11. [State, backends, keys and locking](#11-state-backends-keys-and-locking)
12. [State operations and recovery](#12-state-operations-and-recovery)
13. [Modules](#13-modules)
14. [Environments: dev, stage, prod](#14-environments-dev-stage-prod)
15. [Sharing values between stacks](#15-sharing-values-between-stacks)
16. [Authentication: profiles, keys, roles and OIDC](#16-authentication-profiles-keys-roles-and-oidc)
17. [Secrets](#17-secrets)
18. [Lifecycle, replacement and resource behaviour](#18-lifecycle-replacement-and-resource-behaviour)
19. [Importing existing infrastructure](#19-importing-existing-infrastructure)
20. [Refactoring and drift](#20-refactoring-and-drift)
21. [Upgrades: Terraform, providers, modules](#21-upgrades-terraform-providers-modules)
22. [Quality gates and testing](#22-quality-gates-and-testing)
23. [CI/CD pipelines: GitLab, GitHub, Terraform Cloud](#23-cicd-pipelines-gitlab-github-terraform-cloud)
24. [AWS resource patterns](#24-aws-resource-patterns)
25. [Team and organisation practices](#25-team-and-organisation-practices)
26. [Scenario questions](#26-scenario-questions)
27. [Command cheat sheet](#27-command-cheat-sheet)

---

## 1. Core model: how Terraform thinks

Terraform is **declarative**: you describe the end state, and Terraform works out the steps.

It always compares **three things**:

```
 Code (.tf)            State file              Real infrastructure
 what you WANT    vs   what Terraform       vs   what actually EXISTS
                       REMEMBERS creating
```

`terraform plan` refreshes reality, compares it with state and code, and shows the difference.

| Plan symbol | Meaning |
|---|---|
| `+` | Create |
| `~` | Update in place |
| `-` | Destroy |
| `-/+` | Destroy then recreate (**look for `# forces replacement`**) |
| `<=` | Read (data source) |

**Key terms**

| Term | Meaning |
|---|---|
| Provider | Plugin that talks to an API (AWS, GCP, Kubernetes…) |
| Resource | Something Terraform creates and manages |
| Data source | Something Terraform only reads |
| State | Terraform's record: code address ↔ real resource ID |
| Module | A folder of `.tf` files with inputs and outputs |
| Root module | The folder where you run `terraform apply` |
| Address | A resource's identity, for example `aws_instance.web` or `module.vpc.aws_subnet.private["ap-southeast-1a"]` |

**Interview line:** "Terraform compares code, state and real infrastructure. Plan refreshes reality, diffs it against
the code, and I always check for `forces replacement` before applying."

---

## 2. Workflow and CLI commands

```bash
terraform init                    # download providers and modules, connect the backend
terraform fmt -recursive          # format code
terraform validate                # syntax and type checks (no API calls)
terraform plan -out=tfplan        # preview and save the plan
terraform apply tfplan            # apply exactly that plan
terraform destroy                 # delete everything in this state
```

**What `terraform apply` does without a saved plan:** it takes the lock, refreshes, plans, **shows the plan and asks
for `yes`**. Nothing changes until you confirm. `-auto-approve` skips the prompt, so avoid it outside automation.

**`terraform fmt -check -recursive`:** checks every `.tf` file in all subfolders without changing anything; a non-zero
exit code fails CI. Fix locally with `terraform fmt -recursive`. It checks **style only**; `validate` checks correctness.

**When to run `init` again:** new or changed module source, new provider, backend change, or a fresh clone. Not for
ordinary resource changes.

---

## 3. Project structure and standard files

| File | Contains |
|---|---|
| `versions.tf` | `required_version`, `required_providers` |
| `providers.tf` | `provider` blocks, `default_tags` (no credentials) |
| `backend.tf` | `backend "s3" {}` (empty) |
| `backend.hcl` | bucket, key, region, `use_lockfile` (one per environment) |
| `variables.tf` | Input **declarations** (type, default, validation). Some teams call it `inputs.tf` |
| `terraform.tfvars` / `env/*.tfvars` | Input **values** |
| `locals.tf` | Computed names, tags, derived maps |
| `data.tf` | Data sources |
| `main.tf` | Resources and module calls (split by domain in big stacks) |
| `outputs.tf` | Values exposed to users and other stacks |

**Special files**

| File | Behaviour |
|---|---|
| `*.tf` | All loaded together; names are only conventions |
| `terraform.tfvars`, `*.auto.tfvars` | Auto-loaded values |
| `*.tftest.hcl` | Tests for `terraform test` |
| `.terraform.lock.hcl` | Provider versions + checksums: **commit it** |
| `.terraform/` | Local cache (`modules/`, `providers/`): **never commit** |
| `*_override.tf` | Merged over other files; avoid |

**Where module code is cached:** `.terraform/modules/` (with a `modules.json` manifest). Local-path modules are used
in place, not copied. Providers go in `.terraform/providers/`.

**Recommended repo layout**

```
repo/
├── modules/            # reusable: vpc/, ec2-instance/, rds/ ...
└── live/
    ├── dev/   network/  servers/  data/
    ├── stage/ network/  servers/  data/
    └── prod/  network/  servers/  data/
```

**.gitignore essentials:** `.terraform/`, `*.tfstate`, `*.tfstate.*`, `tfplan`, secret tfvars.

---

## 4. Language building blocks

### Naming anatomy

```hcl
resource "aws_instance" "web" { ... }
#  keyword   TYPE (from provider)   LOCAL NAME (yours)
# Reference: aws_instance.web.id
```

### Top-level blocks

| Block | Purpose | Referenced as |
|---|---|---|
| `terraform {}` | Versions, providers, backend | — |
| `provider "aws" {}` | Configure a provider | `provider = aws.alias` |
| `resource` | Create and manage | `TYPE.NAME.attr` |
| `data` | Read existing things | `data.TYPE.NAME.attr` |
| `variable` | Input | `var.NAME` |
| `locals` | Computed values | `local.NAME` |
| `output` | Expose values | `module.X.NAME` |
| `module` | Call a module | `module.NAME.output` |
| `moved` | Rename without destroying (1.1+) | — |
| `import` | Adopt existing resources (1.5+) | — |
| `removed` | Stop managing (1.7+) | — |
| `check` | Health assertion; warns only (1.5+) | — |

### Meta-arguments (any resource or module)

| Meta-argument | Use |
|---|---|
| `count` | N copies, or `? 1 : 0` toggle |
| `for_each` | One per map key or set item (stable keys) |
| `depends_on` | Explicit order with no reference |
| `provider` / `providers` | Pick an aliased provider / pass into a module |
| `lifecycle` | Change create/destroy behaviour (see section 18) |

### Built-in references

`var.x`, `local.x`, `each.key`, `each.value`, `count.index`, `self`, `path.module`, `path.root`, `terraform.workspace`.

### Useful helper providers and resources

| Name | Use |
|---|---|
| `terraform_data` (built in, 1.4+) | Provisioners, replacement triggers; replaces `null_resource` |
| `random_*` | Unique names; passwords (end up in state) |
| `time_sleep` | Wait for eventual consistency |
| `archive_file` | Zip code for Lambda |
| `http` data source | Health checks, fetching values |

---

## 5. Variables, tfvars and precedence

```hcl
variable "env" {
  type        = string
  description = "Environment"
  validation {
    condition     = contains(["dev", "stage", "prod"], var.env)
    error_message = "env must be dev, stage or prod."
  }
}

variable "buckets" {
  type = map(object({
    versioning  = optional(bool, false)
    expire_days = optional(number)
  }))
}

variable "db_password" {
  type      = string
  sensitive = true
  ephemeral = true        # 1.10+: never stored in state or plan
}
```

**Types:** `string`, `number`, `bool`, `list()`, `set()`, `map()`, `object({})`, `tuple([])`, `any`. `optional()` gives
defaults inside objects.

**Auto-loaded files:** `terraform.tfvars`, `terraform.tfvars.json`, `*.auto.tfvars`, `*.auto.tfvars.json` (alphabetical).
Other files need `-var-file`.

**Precedence (lowest → highest; later wins)**

1. `default` in the variable block
2. `TF_VAR_<name>` environment variables
3. `terraform.tfvars`
4. `terraform.tfvars.json`
5. `*.auto.tfvars` (alphabetical)
6. `-var` / `-var-file` on the command line (in order given)

Missing value with no default → Terraform prompts (fails in CI with `-input=false`).

**Locals vs variables:** variables are **inputs from outside**; locals are **computed inside**.

**Outputs:** stored **in state**, so newly added outputs only appear after `terraform apply` (it shows "Changes to
Outputs" with 0 resource changes).

---

## 6. Expressions, conditionals, loops and dynamic blocks

### If-then-else (there are no `if` statements)

```hcl
instance_type = var.env == "prod" ? "m6i.large" : "t3.micro"      # both branches same type

# else-if: map + lookup (cleanest)
instance_type = lookup({ dev = "t3.micro", stage = "t3.small", prod = "m6i.large" }, var.env, "t3.micro")

# chained
instance_type = var.env == "prod" ? "m6i.large" : var.env == "stage" ? "t3.small" : "t3.micro"

# null = "as if not set"
kms_key_id = var.encrypt ? aws_kms_key.this[0].arn : null
```

### Conditional resources

```hcl
resource "aws_nat_gateway" "this" {
  count = var.enable_nat ? 1 : 0
  # ...
}

output "nat_id" {
  value = one(aws_nat_gateway.this[*].id)     # null when not created
}
```

### `count` vs `for_each`

| | `count` | `for_each` |
|---|---|---|
| Keys | Index `[0]`, `[1]` | Stable keys `["web"]` |
| Remove a middle item | **Shifts** others → replacements | Only that item removed |
| Use for | 0/1 toggles, identical copies | Collections |

### `for` expressions

```hcl
ids     = [for s in aws_subnet.private : s.id]
by_name = { for k, b in aws_s3_bucket.this : k => b.arn }
public  = [for s in var.subnets : s.id if s.public]
```

### Splat

`aws_instance.app[*].id`

### Dynamic blocks

Generate repeated **nested blocks** inside one resource:

```hcl
dynamic "ingress" {
  for_each = var.ingress_rules
  content {
    from_port   = ingress.value.port
    to_port     = ingress.value.port
    protocol    = "tcp"
    cidr_blocks = ingress.value.cidr_blocks
  }
}

# optional block: one or none
dynamic "ebs_block_device" {
  for_each = var.add_disk ? [1] : []
  content {
    device_name = "/dev/sdf"
    volume_size = 100
  }
}
```

Only for nested blocks (not whole resources or arguments). Use sparingly.

### Templates

`"${var.project}-${var.env}"`, `%{ if var.debug }…%{ endif }`, heredoc `<<-EOT … EOT`.

---

## 7. Built-in functions

Over 100 built in; no user-defined functions in HCL (providers can ship functions since 1.8, called
`provider::aws::arn_parse(...)`). Test in `terraform console`.

| Category | Most used |
|---|---|
| String | `format`, `join`, `split`, `lower`, `upper`, `replace`, `substr`, `trimspace`, `startswith`, `regex` |
| Collection | `length`, `merge`, `concat`, `lookup`, `keys`, `values`, `contains`, `flatten`, `distinct`, `zipmap`, `slice`, `one`, `coalesce`, `alltrue`, `toset` |
| Network | `cidrsubnet`, `cidrhost`, `cidrnetmask` |
| Encoding | `jsonencode`, `jsondecode`, `yamldecode`, `csvdecode`, `base64encode` |
| Files | `file`, `templatefile`, `fileexists`, `filemd5` |
| Types / errors | `try`, `can`, `tostring`, `tonumber`, `sensitive`, `nonsensitive` |
| Numbers / time | `min`, `max`, `ceil`, `floor`, `timestamp` (avoid in resources: permanent diff) |

```hcl
cidrsubnet("10.0.0.0/16", 8, 1)                          # "10.0.1.0/24"
merge(local.tags, { Name = "web" })
templatefile("${path.module}/init.sh.tftpl", { env = var.env })
try(var.cfg.size, 20)
```

---

## 8. Data sources

**Why:** read information that exists outside this configuration (other teams, other stacks, AWS itself) without
managing it. Avoids hardcoding. Read on every plan; never created or destroyed.

| Data source | Use |
|---|---|
| `aws_caller_identity`, `aws_region`, `aws_partition` | Account ID, region, ARN building |
| `aws_availability_zones` | AZs for subnets |
| `aws_ami` (or public SSM AMI parameters) | Latest AMI |
| `aws_vpc`, `aws_subnets`, `aws_security_group` | Existing networks by tag |
| `aws_iam_policy_document` | Generate IAM JSON |
| `aws_ssm_parameter` | Values shared by other stacks |
| `aws_secretsmanager_secret_version` | Secrets (lands in state) |
| `aws_route53_zone`, `aws_acm_certificate` | DNS, certificates |
| `aws_eks_cluster`, `aws_eks_cluster_auth`, `aws_eks_addon_version` | EKS details and Kubernetes provider config |
| `aws_ec2_instance_type` | Architecture and size checks |
| `terraform_remote_state` | Another stack's outputs |

```hcl
data "aws_ami" "al2023" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-2023.*-x86_64"]
  }
}
```

---

## 9. Providers and multi-region

**Provider:** a plugin that translates resources into API calls and defines the resource types you can use.

```hcl
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 6.0" }
  }
}

provider "aws" {
  region = "ap-southeast-1"
  default_tags { tags = { ManagedBy = "terraform" } }
}
```

`terraform init` downloads it and records it in `.terraform.lock.hcl`. Credentials never go in the provider block.

**Version constraints:** `= 1.2.3` exact · `>= 1.2` minimum · `~> 1.2` = `>= 1.2, < 2.0` · `~> 1.2.3` = `>= 1.2.3, < 1.3.0`.
Root stacks pin (`~>`), modules state a minimum (`>=`).

**Multi-region options**

| Option | When |
|---|---|
| Provider **aliases** + `provider = aws.alias` | A few cross-region resources (CloudFront cert in us-east-1, S3 replica) |
| Same **module** per region with `providers = { aws = aws.sydney }` | Repeat infrastructure; `configuration_aliases` when a module needs two regions |
| **Separate stack and state per region**, `region` as a variable | Production and DR: regions independent |
| Per-resource `region` argument (AWS provider v6+) | Fewer aliases |

Provider configurations can't be created dynamically, so you can't `for_each` a module over regions with different
providers. Global services (IAM, Route 53, CloudFront) are managed once; **CloudFront certificates must be in
us-east-1**; plan non-overlapping CIDRs per region.

---

## 10. Execution order: the DAG

- Terraform loads **all `.tf` files in the folder** (not subfolders) as **one configuration**: file names and order don't matter.
- It builds a **Directed Acyclic Graph** from references and `depends_on`.
- It walks the graph in dependency order, **independent resources in parallel** (`-parallelism`, default 10), and in **reverse for destroy**.
- A loop between resources → `Error: Cycle`. Fix by splitting (for example security group rules as separate resources).
- `terraform graph | dot -Tpng > graph.png` shows it.

---

## 11. State, backends, keys and locking

### Why remote state

State maps code to real resource IDs and **contains secrets in plain text**. On a team it must be **shared, locked,
versioned and secured**. Storing it in Git (even private) is unsafe: secrets in history forever, no locking, stale
copies, merge conflicts.

### Backend

The backend defines **where state is stored** and how it's locked. Default: `local`. One per root module; **no
variables allowed** in the block.

```hcl
terraform {
  backend "s3" {}                  # empty: values come from backend.hcl
}
```

```hcl
# backend.hcl
bucket       = "tfstate-prod-<ACCOUNT_ID>-apse1"
key          = "network/terraform.tfstate"
region       = "ap-southeast-1"
encrypt      = true
use_lockfile = true                # S3 native locking, Terraform 1.10+
```

```bash
terraform init -backend-config=backend.hcl                  # connect
terraform init -migrate-state -backend-config=backend.hcl   # copy existing local state up
```

| Backend | Locking |
|---|---|
| `s3` | `use_lockfile = true` (older: DynamoDB table with `LockID`; deprecated) |
| `gcs`, `azurerm`, HCP Terraform (`cloud` block) | Built in |
| `local` | Same machine only |

### The key

The **path of the state file inside the bucket** (`network/terraform.tfstate`). You choose it; Terraform creates the
object on the first write; only the bucket must exist first. **Every stack needs a unique key.** Workspaces add an
`env:/<name>/` prefix.

### Setting up remote state (explain it like this)

1. Create the bucket(s) first: versioning, encryption, public access blocked, `prevent_destroy` (bootstrap stack with
   local state, then migrate its own state in).
2. Add an empty `backend "s3" {}` block.
3. Put values in `backend.hcl` per environment: bucket, **unique key per stack**, region, `encrypt`, `use_lockfile = true`.
4. `terraform init -backend-config=backend.hcl` (add `-migrate-state` to copy existing local state).
5. Verify with `aws s3 ls` and a clean `terraform plan`; delete the local state files.

After that you work exactly as before: edit, `plan`, `apply`. `.terraform/terraform.tfstate` only records the backend
settings, not your infrastructure.

### Locking

- **Automatic** for every command that can write state (`plan`, `apply`, `destroy`, `import`, `state` commands).
- S3: a `<key>.tflock` object exists while a run is active.
- Locally and in CI it's the same; in CI also add a **concurrency group / `resource_group`** per stack and never
  cancel running applies.

```bash
terraform apply -lock-timeout=5m      # wait instead of failing
terraform plan  -lock=false           # read-only jobs only (drift checks), never apply
terraform force-unlock <LOCK_ID>      # stale lock, only after confirming nothing runs
```

**Getting the lock ID:** from the error's `Lock Info: ID:` line, or
`aws s3 cp s3://<bucket>/<key>.tflock - | jq -r .ID`.

**Permissions needed:** `s3:ListBucket`, `GetObject`/`PutObject` on the state, and `GetObject`/`PutObject`/`DeleteObject`
on the `.tflock` object.

---

## 12. State operations and recovery

| Task | Command / block |
|---|---|
| List managed resources | `terraform state list` (filter by address, `-id=<real id>`) |
| Inspect one | `terraform state show ADDR` |
| Everything with attributes | `terraform show`, `terraform show -json` |
| Backup | `terraform state pull > backup.tfstate` |
| Rename / move within state | `moved {}` (old: `terraform state mv`) |
| Stop managing | `removed { destroy = false }` (old: `terraform state rm`) |
| Adopt | `import {}` (old: `terraform import`) |
| Push a state | `terraform state push FILE` (`-force` if serial is lower) |

### Moving state

- **Whole state to a new backend/bucket/key:** change backend config → `terraform init -migrate-state` → clean plan.
  (Manual: `state pull` with old backend, `init -reconfigure` with new, `state push`.)
- **Resources between stacks:** `import` in the new stack **first**, then `removed { destroy = false }` in the old one.

### Corrupted or deleted state

1. **Freeze** all runs on that stack; don't apply.
2. **Back up** what exists: `terraform state pull > broken.tfstate`.
3. **Diagnose:** parse error, empty state (plan wants to create everything), missing resources, serial/lineage errors.
4. **Restore** from S3 versioning: list versions, download the last good one, validate with `jq`, `terraform state push`
   (`-force` if needed). Deleted file → remove the **delete marker**.
5. **Verify** with `plan -refresh-only` and `plan`; reconcile changes made after that version (import / `state rm`).
6. **No backup:** rebuild with `import` blocks.
7. **Prevent:** versioning, locking, CI-only applies, no manual edits, backups before surgery, small states.

### Version rules

State written by a **newer** Terraform can't be used by an older one ("state snapshot was created by Terraform vX,
which is newer"). 0.13 also changed provider addresses in state. Upgrades are one-way; pin versions.

---

## 13. Modules

**A module is a reusable package of Terraform code, like a function:** inputs (variables) → resources → outputs.

**Real-world story:** security rules require every S3 bucket to be encrypted, versioned, private and tagged. A
`secure-s3-bucket` module builds those rules in; teams pass name, env and owner in five lines; every bucket is compliant;
rule changes are made once.

### Creating one

```
modules/secure-s3-bucket/
├── versions.tf     # required_providers with a MINIMUM version
├── variables.tf    # typed, validated inputs
├── main.tf         # resources (no provider, no backend)
├── outputs.tf      # IDs, ARNs, names
└── README.md
```

### Using one

```hcl
module "logs_bucket" {
  source = "../../modules/secure-s3-bucket"
  name   = "logs"
  env    = "dev"
  owner  = "data-team"
}

module "team_buckets" {
  source   = "../../modules/secure-s3-bucket"
  for_each = { logs = "data-team", artifacts = "ci-team" }
  name     = each.key
  env      = "dev"
  owner    = each.value
}

output "logs_arn" {
  value = module.logs_bucket.bucket_arn
}
```

Run `terraform init` after adding or changing a module source.

### Sources

| Source | Example |
|---|---|
| Local path | `../../modules/vpc` |
| Git + tag | `git::https://github.com/acme/tf-modules.git//vpc?ref=vpc-v1.2.0` |
| Registry | `source = "terraform-aws-modules/vpc/aws"`, `version = "~> 5.0"` |
| Private registry | HCP Terraform, GitLab |

### Best practices

One purpose per module; typed and validated inputs; useful outputs; **no provider or backend inside**; preconditions
for assumptions (for example subnet belongs to VPC); README; `terraform test`; pinned versions.

### Two versions at the same time (breaking change)

- **Separate repo/registry:** semantic versioning; old caller pins `v1.x`, new callers use `v2.0.0`; maintain a v1
  branch; upgrade guide; `moved` blocks inside the module.
- **Same repo:** local paths can't be pinned, so either reference the same repo through a **Git source with tags**
  (`?ref=s3-bucket-v1.4.0`), or keep a temporary `s3-bucket-v2/` folder, or make the change **backward compatible**
  (keep old inputs with `coalesce`).

### Generic EC2 module example (from our sessions)

Inputs: `name`, `vpc_id`, `subnet_id`, `security_group_ids`, `create_security_group`, `ingress_rules`, `instance_type`,
`ami_id` (null → latest AL2023), `iam_instance_profile`, `user_data`, `tags`. Features: IMDSv2, encrypted gp3 root,
`ignore_changes = [ami]`, precondition that the subnet is in the VPC and at least one SG is attached, outputs for ID,
IPs and SGs. Called once for a web server and with `for_each` for app servers.

---

## 14. Environments: dev, stage, prod

**Goal:** write the infrastructure once; environments differ only in values, state and credentials.

| Approach | How | Best for |
|---|---|---|
| **Directory per environment** (recommended) | `live/<env>/<stack>` thin roots calling shared modules; own tfvars, `backend.hcl`, role | Real environments |
| One root + tfvars/backend per env | `init -reconfigure -backend-config=env/prod.backend.hcl`, `-var-file=env/prod.tfvars` | Small single-stack projects (wrap in a script) |
| Workspaces | `terraform workspace select prod`, `terraform.workspace` lookups | Short-lived identical copies (per-PR stacks) |
| Terragrunt | Generates backend/provider config, passes inputs | Many stacks and environments |

**Why workspaces aren't used for real environments:** they share one backend and one set of credentials, the active
workspace is invisible (easy to apply to the wrong one), all environments get the same code at once, and PR diffs don't
show which environment is affected. Separating state by directory and backend solves those; the costs are some
duplication and ordering between stacks.

**What differs between environments:** `terraform.tfvars`, `backend.hcl`, role/account, module version, approval rules.
Handle differences with variables, `count` toggles and `lookup` maps, never copied code.

**State layout:** one bucket per environment (ideally in its own account), one key per stack. Separate AWS accounts give
the strongest boundary; in one account, use separate roles with an explicit **deny** on other environments' state.

---

## 15. Sharing values between stacks

Folder B needs values from folder A. **Apply A first.**

| Option | How | Notes |
|---|---|---|
| `terraform_remote_state` | `data.terraform_remote_state.a.outputs.vpc_id` | Only A's **outputs** are visible; B needs read access to A's **whole state** |
| **SSM Parameter Store** (recommended) | A writes `aws_ssm_parameter`; B reads `data "aws_ssm_parameter"` | Clean contract; no state access |
| Data sources by tags | `data "aws_vpc" { tags = {...} }` | No coupling; needs good tagging |
| Variables via CI / Terragrunt `dependency` | Pass A's `terraform output` into B | Needs orchestration |

```hcl
data "terraform_remote_state" "a" {
  backend = "s3"
  config = {
    bucket = "tfstate-dev-<ACCOUNT_ID>-apse1"
    key    = "network/terraform.tfstate"
    region = "ap-southeast-1"
  }
}
# data.terraform_remote_state.a.outputs.<OUTPUT_NAME>   (the output's name, not the resource)
```

---

## 16. Authentication: profiles, keys, roles and OIDC

### Credential chain

Provider arguments → environment variables → `~/.aws` profiles (SSO, roles) → container/instance role. **Keep
credentials out of code.**

### `aws sts get-caller-identity`

Asks AWS "who am I with these credentials?". Needs no permissions. The CLI first **finds credentials locally**, then
signs the request (SigV4; the secret is never sent) and calls STS, which validates them. Returns:

- IAM user: `arn:aws:iam::<acct>:user/<name>`
- Role session: `arn:aws:sts::<acct>:assumed-role/<role>/<session-name>`

First step in any credentials or AccessDenied problem.

### `AWS_PROFILE`

Selects a named profile from `~/.aws` for the CLI **and** Terraform in that terminal. Each developer has **their own**
credentials; the team agrees on profile names; everyone assumes the same deployer role; session names identify people in
CloudTrail. Tip: `direnv` with `.envrc` per folder. Real companies use **IAM Identity Center (SSO)**.

### Two IAM users in one account (our lab pattern)

```
rishabh-admin (AdministratorAccess + MFA)   → bootstrap, IAM, emergencies
tf-runner (ONLY sts:AssumeRole)             → tf-dev-deployer / tf-prod-deployer roles
```

Access keys created **outside** Terraform (otherwise the secret lands in state). Assuming a role needs both the caller's
permission **and** the role's **trust policy**.

### Access keys (what you used first)

| Variable | Meaning |
|---|---|
| `AWS_ACCESS_KEY_ID` | Identifier (`AKIA…` long-lived, `ASIA…` temporary) |
| `AWS_SECRET_ACCESS_KEY` | Secret |
| `AWS_SESSION_TOKEN` | Only for temporary credentials |

(They're not "public/private keys"; key pairs are for EC2 SSH.)

Usage: `export` the three variables (simplest; works for provider **and** backend), or a profile, or `TF_VAR_` into
provider arguments (the backend still needs `AWS_*`). In GitLab: store them as **masked, protected, environment-scoped**
CI/CD variables named `AWS_*`; `TF_VAR_*` is for other secret inputs.

**Problems:** tokens expire and break pipelines, manual rotation, leak risk, same identity for all jobs, weak audit.

### OIDC (the fix): no stored keys

1. Register the CI system as an **OIDC identity provider** in AWS (`https://gitlab.com` or
   `https://token.actions.githubusercontent.com`, audience `sts.amazonaws.com`).
2. Create roles per environment: a **read-only plan role** (any branch / PR) and an **apply role** (only protected
   `main` or an approved environment). The **`sub` condition** is the security boundary.
3. Least-privilege permissions; deny other environments' state.
4. Pipeline requests a token (`id_tokens` in GitLab, `permissions: id-token: write` in GitHub) and Terraform exchanges it
   via `AssumeRoleWithWebIdentity` (`AWS_ROLE_ARN` + `AWS_WEB_IDENTITY_TOKEN_FILE`).
5. Roll out dev → stage → prod, confirm in CloudTrail.
6. Remove CI key variables, **deactivate** then delete the keys.

```hcl
condition {
  test     = "StringEquals"
  variable = "gitlab.com:sub"
  values   = ["project_path:acme/infra:ref_type:branch:ref:main"]
}
```

| OIDC error | Cause |
|---|---|
| Not authorized to perform `sts:AssumeRoleWithWebIdentity` | `sub` mismatch (project, branch, StringLike vs StringEquals) |
| Incorrect token audience | `aud` ≠ `client_id_list` |
| No OpenIDConnect provider found | Provider URL mismatch (self-managed GitLab) |

**Cross-account:** provider `assume_role` or the CI job assuming a role in the target account; OIDC provider in each
account.

---

## 17. Secrets

**Rule:** never in code, tfvars or Git. **State (and saved plans) store every value in plain text**, including
`sensitive` ones.

| Technique | Example |
|---|---|
| Let AWS own the secret | RDS `manage_master_user_password = true` → Secrets Manager; state holds only the ARN |
| Terraform creates the container, not the value | `aws_secretsmanager_secret`, value set outside |
| Apps read secrets at runtime | IAM access to Secrets Manager / SSM |
| Pass at runtime | `TF_VAR_x`, masked/protected CI variables |
| `sensitive = true` | Redacts in plan/apply/output; mark propagates; outputs using it must be sensitive |
| Ephemeral values / write-only args (1.10/1.11+) | `ephemeral = true`, `password_wo` + `password_wo_version`: never persisted |
| Don't generate credentials in Terraform | Avoid `aws_iam_access_key`, `tls_private_key`, `random_password` for real secrets |
| Protect state | KMS encryption, restricted bucket, per-environment state, short-lived plan artifacts |
| Prevent leaks | gitleaks/trufflehog in pre-commit and CI; rotate anything that leaked |
| No stored cloud credentials | OIDC / SSO |

**Where `sensitive` values are still visible:** state, saved plans, `terraform output -raw`/`-json`, `TF_LOG=DEBUG` logs,
your own `echo`.

---

## 18. Lifecycle, replacement and resource behaviour

```hcl
lifecycle {
  create_before_destroy = true                 # replacement without a gap (pair with name_prefix)
  prevent_destroy       = true                 # plan fails if it would destroy
  ignore_changes        = [ami, tags["x"]]     # stop fighting external changes
  replace_triggered_by  = [aws_launch_template.x]
  precondition  { condition = ... error_message = "..." }
  postcondition { condition = ... error_message = "..." }
}
```

### Forcing replacement: `taint` vs `-replace`

- `terraform taint ADDR` marks a resource in state so the next apply recreates it. **Deprecated** (0.15.2): it changes
  state before review.
- Use `terraform apply -replace="ADDR"`: shown in the plan, one reviewed step.
- Terraform **taints automatically** when creation fails halfway; `terraform untaint ADDR` if it's actually healthy.

### Refresh

- `terraform refresh` updates state from reality without touching infrastructure. **Deprecated** (0.15.4): it rewrites
  state without review; wrong region/credentials could drop resources.
- Use `terraform plan -refresh-only` (see drift) and `terraform apply -refresh-only` (accept it). Every plan refreshes
  automatically; `-refresh=false` skips it.

### What replaces vs updates (examples)

| Change | Result |
|---|---|
| S3 **bucket name** | **Replace**: names are immutable; Terraform never copies objects. Without `force_destroy` → `BucketNotEmpty`; with it → **data deleted**. Migrate: new bucket → `aws s3 sync` → cut over → `removed` old |
| S3 tags, versioning, encryption, lifecycle, policy | In place (separate resources) |
| EC2 **`Name` tag** | In place, no downtime |
| Terraform **resource address** rename | Destroy + create unless `moved` |
| EC2 `instance_type` (t3.small → t3.medium) | In place with **stop/start**: brief downtime, public IP changes unless EIP; x86 → ARM needs a new AMI (replace); ASG → change launch template + instance refresh |
| EC2 `user_data` | In place (stop/start), script doesn't re-run; with `user_data_replace_on_change` → replace |
| EC2 `ami`, `subnet_id`, AZ, `key_name`, `private_ip` | Replace (root volume data lost) |
| RDS `identifier` | Replace (new empty DB): protect with `prevent_destroy` + `deletion_protection` |
| RDS instance class | In place |

---

## 19. Importing existing infrastructure

Import only writes to **state**; it never changes the real resource.

**Steps (example: a hand-made EC2 instance)**

1. Find the ID: `aws ec2 describe-instances --filters Name=tag:Name,Values=legacy-web ...`
2. Write an `import` block:

```hcl
import {
  to = aws_instance.legacy_web
  id = "i-0abc1234def567890"
}
```

3. Draft code: `terraform plan -generate-config-out=generated.tf`
4. Clean it: remove defaults, replace hardcoded IDs with references, add `ignore_changes` / `prevent_destroy`.
5. Plan until: **`N to import, 0 to add, 0 to change, 0 to destroy`** (never `forces replacement`).
6. `terraform apply`, then delete the `import` block.
7. Import related resources separately: security groups, EBS volumes + attachments, Elastic IPs.

**ID formats** are in each resource's docs ("Import" section). AWS: instance ID, bucket name, role name. GCP:
`projects/{p}/zones/{z}/instances/{name}`, `projects/{p}/global/networks/{name}`, bucket name.

**Old way:** `terraform import ADDR ID` (no plan, not in Git). **Many resources:** generate import blocks from an
inventory; `for_each` in import blocks (1.7+).

**Bulk tools (drafts only; always clean up):** Terraformer (AWS/GCP/Azure, code + state), `gcloud beta resource-config
bulk-export --resource-format=terraform` (GCP), former2 (AWS), aztfexport (Azure), commercial (Firefly, ControlMonkey).
Clean-up: hardcoded IDs, no modules, noisy defaults, secrets, generated names, stack boundaries.

**Moving state between S3 and GCS** is not an import: change the backend and `init -migrate-state`.

---

## 20. Refactoring and drift

| Tool | Use | Since |
|---|---|---|
| `moved {}` | Rename, move into a module, `count` → `for_each` | 1.1 |
| `import {}` + `-generate-config-out` | Adopt existing | 1.5 |
| `removed { destroy = false }` | Stop managing without deleting | 1.7 |
| `plan/apply -refresh-only` | See / accept drift | 0.15 |
| `apply -replace` | Rebuild one resource | 0.15 |
| `-target` | Emergencies only | — |

**Rule:** a refactor's plan must show **zero destroys** for things that are only moving.

```hcl
moved {
  from = aws_s3_bucket.team[0]
  to   = aws_s3_bucket.team["alpha"]
}
```

### Drift

Reality ≠ code/state (usually console changes). Detect with nightly `terraform plan -detailed-exitcode` (exit 2 =
changes). Inspect with `plan -refresh-only`. Decide per change: **revert** (apply), **codify** (update code),
**`ignore_changes`** (another system owns it), or **accept into state** (`apply -refresh-only`).

**Perpetual diffs:** hand-written JSON (use `jsonencode` / `aws_iam_policy_document`), mixing inline and separate rules,
`default_tags` duplicated in resource tags, AWS defaults not declared, `timestamp()` in arguments.

### Finding who changed a resource

CloudTrail Event history (90 days; trail to S3 for longer) filtered by resource name or event → the identity. Shared
role → session name identifies the person or CI run → Git commit/PR. AWS Config shows before/after.
`plan -refresh-only` shows whether it happened outside Terraform. Prevent: individual SSO identities, per-person session
names, long CloudTrail retention, read-only prod console, drift detection.

---

## 21. Upgrades: Terraform, providers, modules

**Process for all three:** one upgrade per PR → read changelog/upgrade guide → bump constraint → `terraform init
-upgrade` → commit lock file → plan in dev (goal: no changes; **stop on `forces replacement`**) → apply dev → test →
stage → prod. Rollback: revert the PR (constraint + lock file). Automate detection with Renovate/Dependabot.

| Upgrade | Specifics |
|---|---|
| Terraform CLI | `required_version`, tfenv/tfswitch, CI image. **One-way for state**: back up, move everyone together |
| Providers | Minor: `init -upgrade` (lock file decides exact version). Major: read upgrade guide, fix renamed args. Regenerate hashes: `terraform providers lock -platform=linux_amd64 -platform=darwin_arm64` |
| Modules | Bump `version` or `?ref=` tag; `init -upgrade`; read CHANGELOG; good modules ship `moved` blocks. Not in the lock file |

---

## 22. Quality gates and testing

| Guardrail | Runs at | Blocks? | Use |
|---|---|---|---|
| Variable `validation` | Plan | Yes | One input |
| `precondition` | Plan | Yes | Cross-input / data assumptions (ARM type with x86 AMI) |
| `postcondition` | After create/read | Yes | Guarantee the result |
| `check` block | End of plan/apply | **No (warns)** | Ongoing health |
| `terraform test` (`*.tftest.hcl`) | On demand / CI | Yes | Module unit tests (`command = plan`) and integration (`apply`) |
| fmt / validate / tflint (AWS ruleset) | Pre-commit, CI | Yes | Style, provider-aware errors |
| Checkov / Trivy | CI | Configurable | Security misconfigurations |
| OPA / Sentinel | CI / HCP | Yes | Organisation policy |
| Infracost | PR | Advisory | Cost impact |
| gitleaks | Pre-commit, CI | Yes | Leaked secrets |
| terraform-docs | Pre-commit | — | Module README tables |

```hcl
run "rejects_invalid_cidr" {
  command = plan
  variables { cidr = "banana" }
  expect_failures = [var.cidr]
}
```

---

## 23. CI/CD pipelines: GitLab, GitHub, Terraform Cloud

**In plain words:** nobody applies from a laptop. Change → MR/PR → automated checks → plan → review → merge → approve →
apply **exactly the reviewed plan** → nightly drift check.

### GitLab CI

| Piece | How |
|---|---|
| Stages | `validate` (fmt, validate) → `plan` (saved `tfplan` artifact, expire 1 day) → `apply` (manual, main only) |
| Auth | GitLab OIDC: `id_tokens` with `aud: sts.amazonaws.com`, `AWS_ROLE_ARN`, `AWS_WEB_IDENTITY_TOKEN_FILE`; read-only plan role, main-only apply role |
| Locking | Terraform S3 lock + **`resource_group`** per stack + `interruptible: false` on apply |
| Approval | Protected `main`, protected environment, `when: manual` |
| Scope | `rules: changes` on the stack folder and `modules/` |
| Image | `hashicorp/terraform:<version>` with `entrypoint: [""]` |

### GitHub Actions

`permissions: id-token: write`, `aws-actions/configure-aws-credentials` with `role-to-assume`, plan on PR with a plan
comment, apply job with `environment:` approval applying the downloaded `tfplan`, `concurrency` group per stack,
`setup-terraform` with `terraform_wrapper: false`, detect changed stacks, nightly drift job.

### Terraform Cloud (HCP Terraform)

| Piece | How |
|---|---|
| Unit | **Workspace** = one stack in one environment (own state, variables, settings) |
| Code | Workspace linked to the repo + working directory; `cloud { organization, workspaces }` block |
| State and locking | Managed |
| Auth | Dynamic provider credentials: `TFC_AWS_PROVIDER_AUTH=true`, `TFC_AWS_RUN_ROLE_ARN`; trust `app.terraform.io` with `sub` like `organization:acme:project:*:workspace:NAME:run_phase:*` |
| PRs | Automatic speculative plans |
| Approval | Auto-apply for dev; **Confirm & Apply** for prod |
| Extras | Sentinel/OPA policies, cost estimation, run tasks, run triggers between workspaces, drift detection, private registry |

| | GitLab CI | Terraform Cloud |
|---|---|---|
| Pipeline | You write it | Built in |
| State | You set up S3 | Managed |
| Flexibility | High | Lower, less maintenance |

---

## 24. AWS resource patterns

### VPC

`for_each` subnets keyed by AZ, CIDRs with `cidrsubnet`, public/private (/data) tiers, NAT (one in dev, per AZ in
prod) behind a `count` toggle, route table associations, S3 gateway endpoint (free), interface endpoints for ECR/STS,
flow logs, non-overlapping CIDRs per environment/region, EKS subnet tags.

### Security groups

One module: create the group with `name_prefix` + `create_before_destroy`; rules as a **map of objects keyed by stable
names**, each an `aws_vpc_security_group_ingress_rule` via `for_each`, with **either a CIDR or a referenced SG**.
Separate rule resources avoid cycles and noisy diffs; maps avoid index shifts. Rules can come from HCL, `yamldecode`, or
`csvdecode` (CSV values are strings: `tonumber()`). Validation + CI policy block open SSH.

### EC2 software and configuration

| Option | Use |
|---|---|
| `user_data` / cloud-init with `templatefile` | Simple bootstrap at first boot |
| **Packer golden AMI** | Production, ASGs, immutable servers |
| SSM State Manager (`aws_ssm_association`) | Ongoing config, no SSH |
| Ansible after Terraform (inventory from outputs or tags) | Complex config |
| Provisioners (`remote-exec`) | Last resort: creation only, not in state, need SSH, failures taint |

### Lambda

IAM execution role (`AWSLambdaBasicExecutionRole` + least privilege) → package (`archive_file` zip, S3 zip, or ECR image
with `package_type = "Image"`) → `aws_lambda_function` (runtime, handler, timeout, environment, **`source_code_hash`**) →
log group with retention → trigger + `aws_lambda_permission` (EventBridge schedule, API Gateway, function URL, S3
notification; SQS uses `aws_lambda_event_source_mapping`). Production: VPC config, versions/aliases, no secrets in env
vars, layers, CI deploys code with `ignore_changes` on code attributes.

### Other patterns from the project story

EKS (access entries, add-on versions per k8s version, prefix delegation, IRSA/Pod Identity, one-minor upgrades), RDS
(managed password, Multi-AZ, KMS, deletion protection), ElastiCache (encryption in transit/at rest, AUTH token, Multi-AZ),
OpenSearch (VPC domain, fine-grained access, needs service-linked role), S3 + CloudFront (OAC, ACM in us-east-1, Route 53
alias), VPC peering (routes both sides, DNS resolution, cross-account accepter via provider alias).

---

## 25. Team and organisation practices

### 20 engineers across AWS and GCP

Setup: repo(s) with versioned modules and folder per env/stack; remote locked state split per env/stack (S3, GCS);
**applies only from CI** with PR review; SSO + OIDC (AWS roles, GCP Workload Identity Federation) with plan/apply roles;
guardrails (tflint, Checkov, OPA, Infracost); CODEOWNERS and prod approvals; pinned versions; drift detection; audit.

Issues and fixes: state conflicts (locking, CI-only), drift (read-only console, nightly checks), version mismatches
(pinning), credential sprawl (SSO/OIDC), secrets in state (encryption, managed secrets), blast radius (small states),
inconsistent code (modules, review), bottleneck (self-service modules), cost (Infracost, tags), accidental destroys
(protection), throttling (smaller states, `-parallelism`), multi-cloud complexity (separate stacks, common pipeline).

### Introducing Terraform to a new company

Solve up front: remote backend, state split, repo layout, shared modules, pinned versions, environment/account
separation, SSO/OIDC, secrets policy, PR-based CI with checks, drift detection, import strategy, destroy protection,
naming/tagging/cost, CODEOWNERS, docs and training. Roll out: **pilot** stack → foundations → adopt (import gradually)
→ enforce → improve.

---

## 26. Scenario questions

| Scenario | Answer |
|---|---|
| Code, state and cloud in sync; you run `terraform state rm foo`; then `plan`? | Terraform forgot foo only; plan shows **`+ create` (1 to add)**. Apply fails ("already exists") or creates a duplicate and orphans the original. Fix: `import` it back. To stop managing: remove from code too, or use `removed { destroy = false }` |
| Someone edits a resource in the console | Plan detects drift and plans to revert; decide revert/codify/ignore |
| Someone deletes a resource in the console | Refresh drops it; plan shows `+ create` |
| You delete a resource block | Plan shows `- destroy` (use `removed` to keep it) |
| Plan passes but apply fails | Plan doesn't call create APIs: permissions (read-only plan role), existing names, invalid combinations (AZ, ARM/x86), quotas, capacity, IAM eventual consistency, SCPs, timeouts, drift |
| What happens to state when apply fails | Create call rejected → **not in state**, next plan creates again. Created but later step failed → **in state, tainted** → replaced next apply (or `untaint`). Update failure → partial state saved. Other resources stay; **no rollback**. `create_before_destroy` failures leave a "deposed" object |
| Apply without running plan first | Apply plans itself, shows it, waits for `yes`; risks are `-auto-approve` and not reading; in CI apply a saved plan |
| State written by 0.13, use 0.12? | No: newer-state error, provider address format changed; upgrades one-way |
| New outputs added only | `terraform apply` records them (0 resource changes); `-refresh-only` also works |
| Terraform execution order with 10 files | No file order: DAG from references, parallel where independent |
| Who changed a resource? | CloudTrail → identity/session → CI run → Git; AWS Config for what changed |
| State corrupted | Freeze, back up, restore version, verify, reconcile or re-import, prevent |
| Modify two module versions | Pin versions (Git tags/registry), or v2 folder, or backward-compatible change |

---

## 27. Command cheat sheet

```bash
# Workflow
terraform init [-backend-config=backend.hcl] [-upgrade] [-reconfigure] [-migrate-state]
terraform fmt -recursive | terraform fmt -check -recursive
terraform validate
terraform plan [-var-file=x.tfvars] [-out=tfplan] [-refresh-only] [-detailed-exitcode] [-replace=ADDR]
terraform plan -generate-config-out=generated.tf        # with import blocks
terraform apply [tfplan] [-replace=ADDR] [-refresh-only] [-lock-timeout=5m] [-parallelism=N]
terraform destroy
terraform output [-raw NAME] [-json]
terraform console
terraform test
terraform graph | dot -Tpng > graph.png

# State
terraform state list [ADDR] [-id=REAL_ID]
terraform state show ADDR
terraform state pull > backup.tfstate
terraform state push [-force] FILE
terraform state mv SRC DST        # prefer moved {}
terraform state rm ADDR           # prefer removed {}
terraform import ADDR ID          # prefer import {}
terraform force-unlock LOCK_ID
terraform untaint ADDR

# Workspaces
terraform workspace list | new NAME | select NAME | delete NAME

# Providers
terraform providers lock -platform=linux_amd64 -platform=darwin_arm64

# Debugging and identity
export TF_LOG=DEBUG TF_LOG_PATH=./tf.log
aws sts get-caller-identity [--profile NAME]
env | grep AWS_
aws s3 cp s3://<bucket>/<key>.tflock - | jq -r .ID
aws cloudtrail lookup-events --lookup-attributes AttributeKey=ResourceName,AttributeValue=<id>
```

---

## The 60-second summary answer

> "Terraform is declarative: it compares code, state and real infrastructure, builds a dependency graph, and applies
> changes in order. **Day 0**, I set up a versioned, encrypted S3 backend with native locking and a unique key per
> environment and stack, a repo of versioned modules and thin per-environment folders, pinned Terraform and provider
> versions with a committed lock file, and SSO for people and OIDC for pipelines. **Day 1**, I build with small typed,
> validated modules, `for_each` with stable keys, data sources instead of hardcoded IDs, and stateful resources in their
> own stacks with AWS-managed secrets and destroy protection. **Day 2**, every change goes through a pipeline that lints,
> scans, plans and applies the reviewed plan after approval; I detect drift nightly, refactor with `moved`, `import` and
> `removed` blocks so plans show no accidental destroys, upgrade one component at a time through dev first, and can
> recover state from bucket versioning."
