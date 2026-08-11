# Terraform — Deep Dive Reference

---

## 1. How Terraform Actually Works

### Core architecture
Terraform is a **declarative, state-driven, DAG-based** provisioning engine. Four moving parts:

- **HCL (HashiCorp Configuration Language)** — the `.tf` files where you declare *desired* infrastructure state.
- **Providers** — plugins (binaries, gRPC over stdout/stdin) that translate HCL resource blocks into API calls against a target (AWS, Azure, Kubernetes, Harvester, etc). Terraform core itself knows nothing about AWS — the `aws` provider does.
- **State file (`terraform.tfstate`)** — a JSON document mapping your HCL resources to real-world object IDs (e.g. `aws_instance.web` → `i-0abc123`). This is the single source of truth Terraform diffs against.
- **Terraform core** — builds a dependency graph (DAG) from your config, walks it, calls provider CRUD operations (Create/Read/Update/Delete) in the correct order, and reconciles state.

### The core workflow
```
terraform init    → downloads providers/modules, configures backend
terraform plan     → refresh + diff(desired config, state) = execution plan
terraform apply    → executes the plan via provider API calls, updates state
terraform destroy  → tears down everything tracked in state
```

### What actually happens under the hood on `plan`
1. **Init phase**: loads providers, modules, backend config.
2. **Refresh** (unless `-refresh=false`): Terraform calls each provider's `Read` to check real infra vs. state — this is how drift is detected.
3. **Graph build**: resources become nodes; `depends_on`, interpolations (`aws_subnet.id` referenced elsewhere), and implicit references become edges. Terraform walks this graph, parallelizing independent branches.
4. **Diff**: for every resource, compare *state* → *desired config* → produce an action: `create`, `update in-place`, `destroy and recreate` (when an attribute forces replacement, e.g. changing an EC2 `availability_zone`), or `no-op`.
5. Output: a plan (`+`, `~`, `-`, `-/+`) — nothing is touched yet.

### On `apply`
Terraform walks the same graph and calls provider CRUD functions, then **writes results back into state immediately after each resource** (not at the end) — this is why partial applies are recoverable.

### State — the part people misunderstand most
- State is **not optional bookkeeping**, it's Terraform's only memory. Delete it, and Terraform thinks nothing exists.
- **Local state** (`terraform.tfstate` on disk) is fine solo but breaks concurrent/team use — no locking, and secrets sit in plaintext.
- **Remote backends** (S3 + DynamoDB, Terraform Cloud, Azure Blob, GCS) solve two problems:
  - Central shared state.
  - **Locking** — prevents two people running `apply` simultaneously and corrupting state (S3 backend uses a DynamoDB table for this).
- **State file contains sensitive data** (DB passwords, keys) in plaintext by default — encrypt the backend (S3 SSE, etc.), never commit state to git.

### Providers vs modules
- **Provider** = plugin talking to an API (aws, azurerm, kubernetes, harvester).
- **Module** = a reusable folder of `.tf` files. Root module = your working directory; anything in `module "x" {}` is a child module. Modules don't add new capability — they're just packaging/reuse.

---

## 2. First AWS EC2 Instance

```
project/
├── main.tf
├── variables.tf
├── outputs.tf
└── terraform.tfvars
```

**`main.tf`**
```hcl
terraform {
  required_version = ">= 1.7.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket         = "pr-terraform-state-sgp"
    key            = "ec2/first-instance/terraform.tfstate"
    region         = "ap-southeast-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}

provider "aws" {
  region = var.aws_region
}

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

resource "aws_security_group" "web_sg" {
  name        = "web-sg"
  description = "Allow SSH and HTTP"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.ssh_allowed_cidr]
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "web-sg" }
}

resource "aws_instance" "web" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = var.instance_type
  key_name               = var.key_pair_name
  vpc_security_group_ids = [aws_security_group.web_sg.id]

  root_block_device {
    volume_size = 20
    volume_type = "gp3"
    encrypted   = true
  }

  tags = {
    Name = "first-ec2-terraform"
    Env  = "dev"
  }

  lifecycle {
    create_before_destroy = true
  }
}
```

**`variables.tf`**
```hcl
variable "aws_region"      { type = string; default = "ap-southeast-1" }
variable "instance_type"   { type = string; default = "t3.micro" }
variable "key_pair_name"   { type = string }
variable "ssh_allowed_cidr"{ type = string }
```

**`outputs.tf`**
```hcl
output "instance_id"        { value = aws_instance.web.id }
output "public_ip"          { value = aws_instance.web.public_ip }
```

**Run it:**
```bash
terraform init
terraform plan -out=tfplan
terraform apply tfplan
```

Notes worth internalizing since you already run RKE2/EKS: `data.aws_ami` avoids hardcoding AMI IDs that go stale; `lifecycle { create_before_destroy }` matters for anything that can't tolerate downtime on replace; always pin `required_providers` version — unpinned providers are the #1 cause of "it worked yesterday."

---

## 3. Drift Detection & Remediation

**Drift** = real-world infra diverges from state (someone changed a security group in the console, an autoscaler mutated an attribute, a K8s operator flipped a field Terraform also manages).

### Detecting it
```bash
terraform plan -refresh-only          # shows drift without proposing changes to fix it
terraform plan -refresh-only -out=refresh.tfplan
terraform apply -refresh-only refresh.tfplan   # updates STATE to match reality (doesn't touch real infra)
```
Difference from a normal `terraform plan`: a normal plan refreshes AND proposes actions to force reality back to your config. `-refresh-only` only updates Terraform's *knowledge*, useful when the drift was legitimate (e.g. autoscaler-managed desired count) and you want state to stop fighting it.

### Reconciling drift
Two directions to fix it, pick deliberately:
1. **Config wins** — plain `terraform apply` will revert manual changes back to what HCL says. Correct when the drift was unauthorized/accidental.
2. **Reality wins** — update your `.tf` to match what's actually deployed, then `terraform apply -refresh-only` to sync state, or `terraform import`/`terraform state show` to pull the real values in and hand-edit HCL.

### Continuous drift detection at scale
- `terraform plan -detailed-exitcode` in CI (cron): exit code `0` = no changes, `1` = error, `2` = drift/changes present — wire this into a pipeline that alerts (Slack/PagerDuty) without auto-applying.
- **driftctl** (open source, though maintenance has slowed) and **Terraform Cloud/Enterprise's built-in drift detection** scan cloud accounts against state on a schedule.
- Common root causes worth fixing structurally: manual console changes (lock down IAM to deny console writes to TF-managed resources, tag-enforce via SCP), external controllers mutating fields Terraform also owns (use `lifecycle { ignore_changes = [...] }` for fields legitimately owned elsewhere, e.g. `desired_capacity` on an ASG managed by an autoscaler), and out-of-band automation (Ansible/scripts touching the same resources — pick one owner per resource, always).

---

## 4. Importing an Entire Existing Infra into Terraform

Three approaches depending on scale:

### A. Manual `terraform import` (classic, resource-by-resource)
```bash
terraform import aws_instance.web i-0abc123456
```
This only populates **state** — you must hand-write the matching HCL resource block yourself, or Terraform's next plan will show it as "destroy" (since as far as config is concerned it doesn't exist). Tedious at scale — fine for a handful of resources.

### B. Import blocks + config generation (Terraform ≥1.5, much better)
```hcl
import {
  to = aws_instance.web
  id = "i-0abc123456"
}
```
Then:
```bash
terraform plan -generate-config-out=generated.tf
```
This **generates the HCL for you** from the real resource's current attributes. Review it (generated config is often verbose/over-specified), clean it up, then `terraform apply` to finalize the import into state. This is the current recommended path — repeatable, version-controllable, works well when scripted across many resources by listing IDs from a `aws cli` / `boto3` query and generating one `import {}` block per resource.

### C. Bulk reverse-engineering: Terraformer
[GoogleCloudPlatform/terraformer](https://github.com/GoogleCloudPlatform/terraformer) scans an entire AWS account/region (or Azure/GCP/K8s) and generates both HCL **and** state for everything it finds:
```bash
terraformer import aws --resources=vpc,subnet,sg,ec2_instance,rds \
  --regions=ap-southeast-1 --profile=default
```
Pros: fast bootstrap for "we have 500 resources and zero Terraform." Cons: generated HCL is ugly/monolithic, no module structure, over-verbose (every default value gets written explicitly) — always treat the output as a **starting draft**, not final code. Standard next step: refactor into modules, remove default-value noise, replace hardcoded IDs with `data` sources and variables, split into logical state files per environment/service.

### Practical sequencing for a full account import
1. Inventory first (Terraformer's plan mode, or an AWS Config/Resource Explorer export) — know what you're importing before you start.
2. Import in dependency order where it matters for later refactors: networking (VPC/subnet/SG) → IAM → compute → data stores → DNS/edge.
3. After each batch, `terraform plan` must show **zero diff** — if it doesn't, your generated HCL doesn't match reality yet; fix before moving on. A non-empty plan post-import means the next `apply` will silently mutate something you didn't intend to touch.
4. Split into separate state files per boundary (per env, per service) early — one giant state file for an entire account becomes a locking/blast-radius problem fast.

---

## 5. Upgrading Terraform & Managing Code Over Time

### Terraform core version upgrades
```bash
tfenv install 1.9.0        # tfenv = version manager, like nvm for Terraform
tfenv use 1.9.0
```
Steps for a safe core upgrade:
1. Read the **CHANGELOG** for breaking changes between your current and target minor version (major behavior changes are called out explicitly; state format upgrades are automatic but one-way).
2. `terraform init -upgrade` in a **non-prod** workspace first — pulls latest allowed providers, runs any state migrations.
3. `terraform plan` — must show no unexpected diff purely from the version bump.
4. Commit the upgraded `.terraform.lock.hcl`.
5. Roll to prod once validated.

State upgrades are one-directional — once state is written by 1.9, you cannot open it with 1.5 again. Always upgrade lower environments first, and keep a state backup (`terraform state pull > backup.tfstate`) before any core upgrade.

### Provider version management
```hcl
required_providers {
  aws = {
    source  = "hashicorp/aws"
    version = "~> 5.40"   # allows 5.40.x and 5.4x, not 6.x
  }
}
```
- `.terraform.lock.hcl` pins **exact** resolved versions + checksums — commit this to git always, it's what makes builds reproducible across machines/CI.
- `terraform init -upgrade` bumps to the newest version satisfying your constraint and rewrites the lock file — do this deliberately, review the provider's changelog for deprecated/renamed arguments (very common between major provider versions, e.g. AWS provider 4→5 renamed a lot of S3 arguments).

### Managing code quality over time
- **`terraform fmt -recursive`** and **`terraform validate`** in pre-commit/CI — catches syntax and basic logic errors before plan.
- **`tflint`** — catches provider-specific mistakes (invalid instance types, deprecated arguments) that `validate` misses.
- **Module versioning** — pin child module sources to git tags/semver (`source = "git::https://.../vpc.git?ref=v2.3.0"`), never track `main`/`master` in production code.
- **Workspaces vs directory-per-env**: `terraform workspace` (built-in) shares the same `.tf` code across envs with separate state — convenient but a common footgun (easy to `apply` against the wrong workspace by accident). Most mature setups prefer separate directories/state files per environment instead, often via Terragrunt (see §7).

---

## 6. Policy-as-Code: OPA, Sentinel, and Alternatives

Terraform's plan output (JSON) can be evaluated by a policy engine **before** apply — this is how you enforce "no public S3 buckets," "all EC2 must be tagged," "no security group open to 0.0.0.0/0 on port 22," etc., independent of what any individual engineer wrote in HCL.

| Tool | Model | Where it runs | Notes |
|---|---|---|---|
| **OPA / Conftest** | Rego policy language, general-purpose | CI pipeline, `conftest test plan.json` | Open source, not Terraform-specific — same engine used for Kubernetes admission control, which you already use via NeuVector-adjacent policy work. Steepest learning curve (Rego is its own language) but most flexible. |
| **HashiCorp Sentinel** | Policy-as-code, HCL-adjacent syntax | Terraform Cloud/Enterprise only | Tightly integrated (soft-mandatory/hard-mandatory enforcement levels baked into the TFC run pipeline) but locked to paid HCP/TFE tiers. |
| **tfsec / Trivy (tfsec merged into Trivy)** | Static analysis, pre-built rule set | Pre-commit, CI, local | No policy authoring needed — scans HCL directly for known misconfigurations (open SGs, unencrypted volumes, public buckets). Fastest to adopt, least flexible for custom org rules. |
| **Checkov** | Static analysis + custom policy (Python or YAML) | CI, pre-commit | Broad built-in rule library across AWS/Azure/GCP/K8s, plus custom policy support — good middle ground between tfsec's simplicity and OPA's flexibility. |
| **Terraform native `validation`/`check` blocks** | In-HCL, Terraform ≥1.5 `check {}` blocks | `terraform plan`/`apply` itself | Lightweight, no external tool — good for simple invariants (e.g. instance type is in an allowed list) but not a replacement for a real policy engine at scale. |

### Practical OPA + Terraform pipeline
```bash
terraform plan -out=plan.tfplan
terraform show -json plan.tfplan > plan.json
conftest test plan.json --policy ./policy/
```
Example Rego rule (deny open SSH):
```rego
package main

deny[msg] {
  resource := input.resource_changes[_]
  resource.type == "aws_security_group"
  rule := resource.change.after.ingress[_]
  rule.from_port == 22
  rule.cidr_blocks[_] == "0.0.0.0/0"
  msg := sprintf("SG %s allows SSH from anywhere", [resource.address])
}
```
Wire the `conftest` step into your CI/CD gate before `apply` runs — same shift-left principle as NeuVector admission control in your K8s pipelines, just applied to the infra-provisioning layer instead of runtime.

---

## 7. Terraform vs Terragrunt

**Terragrunt is not a replacement for Terraform** — it's a thin wrapper (by Gruntwork) that solves problems Terraform intentionally doesn't solve natively: DRY config across many environments, and remote state/backend boilerplate.

### The problem it solves
Without Terragrunt, a typical multi-env setup duplicates entire `main.tf`/backend blocks across `dev/`, `staging/`, `prod/` folders — any shared module version bump means editing N copies.

### Structure
```
live/
├── terragrunt.hcl              # root: shared backend config, provider generation
├── dev/
│   ├── vpc/terragrunt.hcl
│   └── eks/terragrunt.hcl
├── staging/
│   ├── vpc/terragrunt.hcl
│   └── eks/terragrunt.hcl
└── prod/
    ├── vpc/terragrunt.hcl
    └── eks/terragrunt.hcl
```

**Root `terragrunt.hcl`** (inherited by everything below it):
```hcl
remote_state {
  backend = "s3"
  generate = { path = "backend.tf", if_exists = "overwrite" }
  config = {
    bucket         = "pr-terraform-state-sgp"
    key            = "${path_relative_to_include()}/terraform.tfstate"
    region         = "ap-southeast-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}

generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite"
  contents  = <<EOF
provider "aws" {
  region = "ap-southeast-1"
}
EOF
}
```

**`dev/vpc/terragrunt.hcl`**:
```hcl
include "root" { path = find_in_parent_folders() }

terraform {
  source = "git::https://github.com/org/tf-modules.git//vpc?ref=v1.4.0"
}

inputs = {
  cidr_block  = "10.10.0.0/16"
  environment = "dev"
}
```

Run per-module or across the whole tree:
```bash
cd live/dev/vpc && terragrunt apply
terragrunt run-all apply          # applies entire dependency tree in order
```

### Key Terragrunt features over raw Terraform
- **DRY backend/provider config** — write once at root, inherited everywhere (`generate` blocks).
- **`dependency` blocks** — pull outputs from another module's state without manual `terraform_remote_state` data sources:
```hcl
dependency "vpc" {
  config_path = "../vpc"
}
inputs = { vpc_id = dependency.vpc.outputs.vpc_id }
```
- **`run-all`** — orchestrates apply/plan/destroy across an entire dependency graph of modules in one command, respecting `dependency` ordering.
- **Auto retry / auto init** — handles transient API throttling and lock file staleness without manual reruns.

### When it's worth adopting
Genuinely useful once you have 3+ environments × several modules each — the DRY-ness pays for itself fast. For a single environment or a handful of modules, it's often unnecessary complexity — plain Terraform with workspaces or directory-per-env is simpler to onboard new engineers into. Given the scale you operate at (multi-customer, multi-site Harvester/RKE2 deployments), Terragrunt's `dependency` + `run-all` pattern maps well onto "same module set, different customer site" if you ever move customer infra provisioning into Terraform rather than the current Ansible/manual-BIND-registry approach.

---

## Quick command reference
```bash
terraform init -upgrade
terraform validate
terraform fmt -recursive
terraform plan -out=tfplan -detailed-exitcode
terraform apply tfplan
terraform plan -refresh-only          # drift check, no changes proposed
terraform state list
terraform state show aws_instance.web
terraform import aws_instance.web i-xxxx
terraform plan -generate-config-out=generated.tf   # import + codegen
terraform state pull > backup.tfstate               # before risky ops
terraform force-unlock <lock-id>                     # stuck lock recovery
```