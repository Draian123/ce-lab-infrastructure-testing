# Lab M5.05 - Infrastructure Testing & Validation

**Cloud Engineering Bootcamp — Week 5, Module 5: Cloud Automation & CI/CD**

A multi-layer testing strategy for Terraform code: static analysis, custom convention
validation, plan output parsing, CI enforcement, and pre-commit hooks.

## Repository Structure

```
ce-lab-infrastructure-testing/
├── .github/workflows/ci.yml          # CI pipeline (3 jobs)
├── scripts/
│   ├── validate-conventions.sh       # Layer 2: naming/tagging/docs conventions
│   ├── validate-plan.sh              # Layer 3: plan output parsing
│   └── install-hooks.sh              # Installs the git pre-commit hook
├── docs/test-results.md              # Captured output of every test run
├── main.tf                           # S3 bucket + DynamoDB lock table
├── variables.tf                      # Inputs with descriptions and validation
├── outputs.tf                        # Bucket name/ARN, table name
├── .tflint.hcl                       # TFLint ruleset configuration
├── .pre-commit-config.yaml           # pre-commit framework config
└── .gitignore
```

## Infrastructure Under Test

`main.tf` provisions four resources:

| Resource | Type | Purpose |
|---|---|---|
| `aws_s3_bucket.data_store` | S3 bucket | Data store, named `<project>-<env>-data-store` |
| `aws_s3_bucket_versioning.data_store` | Versioning | Versioning enabled |
| `aws_s3_bucket_server_side_encryption_configuration.data_store` | Encryption | SSE-KMS with bucket key |
| `aws_dynamodb_table.state_lock` | DynamoDB table | State lock table, `PAY_PER_REQUEST` |

Every taggable resource carries `Name`, `Environment`, `ManagedBy`, and `CostCenter`.
The `environment` variable is constrained by a `validation` block to `dev`, `staging`, or `prod`.

## Testing Strategy

Four layers, ordered from fastest/cheapest to slowest. Each catches a different class of
defect, and each runs at a different point in the development cycle.

### Layer 1 — Static Analysis (no credentials, no network to AWS)

| Check | Command | Catches |
|---|---|---|
| Formatting | `terraform fmt -check -recursive` | Inconsistent indentation/alignment |
| Syntax & internal consistency | `terraform validate` | Undefined variables, type errors, bad references |
| Lint | `tflint` | Provider-specific mistakes, deprecated syntax, naming/doc rules |

`terraform validate` requires `terraform init -backend=false` first so the AWS provider
schema is available. `-backend=false` keeps this credential-free.

TFLint is configured in `.tflint.hcl` with the `terraform` recommended preset, the AWS
ruleset plugin, plus explicit `snake_case` naming and documented-variables/outputs rules.

### Layer 2 — Convention Validation (`scripts/validate-conventions.sh`)

Generic tools don't know an organization's standards. This script encodes ours:

1. **Naming** — every resource local name must match `^[a-z][a-z0-9_]+$` (snake_case).
2. **Required tags** — `Name`, `Environment`, `ManagedBy` must be present.
3. **Variable descriptions** — every `variable` block in `variables.tf` needs a `description`.
4. **No hardcoded regions** — a literal like `region = "eu-west-1"` fails; use `var.aws_region`.

Exits `1` on any violation, printing `file:line` for each. Runs in ~1 second, needs no AWS access.

### Layer 3 — Plan Validation (`scripts/validate-plan.sh`)

Static analysis proves the code *parses*; plan parsing proves it *does what we intend*.

```
terraform plan -out=tfplan → terraform show -json tfplan → jq
```

The script then asserts:

- All four expected resource **types** appear in `resource_changes`.
- Reports the count of resources to be created.
- **Warns** if any resource would be destroyed (surfacing accidental replacements).
- Cleans up `tfplan` and `plan.json` afterwards.

This layer **requires AWS credentials**, because `terraform plan` calls the AWS API.
In CI it reads `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` from repository secrets.
It also requires `jq` (preinstalled on GitHub's `ubuntu-latest` runners).

### Layer 4 — CI Pipeline (`.github/workflows/ci.yml`)

Triggers on `pull_request` and `push` to `main`. Three jobs:

| Job | Runs | Depends on |
|---|---|---|
| `static-analysis` | fmt → init → validate → tflint | — |
| `convention-checks` | `validate-conventions.sh` | — (runs in parallel) |
| `plan-validation` | `validate-plan.sh` with AWS secrets | `static-analysis` |

`static-analysis` and `convention-checks` run in parallel since neither needs credentials.
`plan-validation` is gated behind `static-analysis` so a syntax error never burns an AWS
API round-trip. Any failing job fails the check on the PR and blocks merge.

### Pre-Commit Hooks — shifting left

Two options are provided:

- **`scripts/install-hooks.sh`** (zero dependencies) writes a `.git/hooks/pre-commit` that
  runs fmt → validate → convention checks before every commit. Install with:
  ```bash
  chmod +x scripts/install-hooks.sh && ./scripts/install-hooks.sh
  ```
- **`.pre-commit-config.yaml`** for teams already using the `pre-commit` framework
  (`terraform_fmt`, `terraform_validate`, plus the local convention check).

Both deliberately skip Layer 3 — a commit shouldn't need AWS credentials or take 30 seconds.

## Test Coverage Matrix

| Failure mode | Layer that catches it | Where it runs |
|---|---|---|
| Misaligned/unformatted HCL | `terraform fmt -check` | pre-commit + CI |
| Typo in a variable reference | `terraform validate` | pre-commit + CI |
| Deprecated / invalid provider argument | `tflint` | CI |
| `CamelCase` resource name | conventions ch. 1 | pre-commit + CI |
| Missing `Environment` cost tag | conventions ch. 2 | pre-commit + CI |
| Undocumented variable | conventions ch. 3 | pre-commit + CI |
| Region hardcoded instead of variable | conventions ch. 4 | pre-commit + CI |
| Invalid `environment` value | `validation` block in `variables.tf` | validate + plan |
| A resource silently dropped from the config | plan validation | CI |
| A change that would destroy a live resource | plan validation (WARN) | CI |

**Not covered** (out of scope for this lab): post-apply integration tests against real
infrastructure (Terratest), policy-as-code (OPA/Sentinel), drift detection, and security
scanning with Checkov — Checkov is the Extra Mile exercise and was not implemented here.

## Running the Tests Locally

```bash
# Layer 1
terraform fmt -check -recursive
terraform init -backend=false
terraform validate
tflint --init && tflint

# Layer 2
chmod +x scripts/*.sh
./scripts/validate-conventions.sh

# Layer 3 (needs AWS credentials + jq)
./scripts/validate-plan.sh

# Pre-commit hook
./scripts/install-hooks.sh
```

Full captured output from the verification run is in [`docs/test-results.md`](docs/test-results.md).

## Test Failures Found and Resolved

Two real bugs surfaced while getting the convention script to pass. Both are documented
here because the point of the lab is that tests find things.

**1. `grep -c` over multiple files breaks integer comparison.**
`grep -c "pattern" *.tf` prints one `file:count` line *per file*
(`main.tf:4`, `outputs.tf:0`, …), not a single number. Feeding that into
`[ "$TAG_PRESENT" -eq 0 ]` raises `integer expression expected`, and under `set -e` the
script dies before reporting anything.
**Fix:** pipe the files together first — `cat *.tf | grep -c "pattern"` — so exactly one
number comes back.

**2. Tag keys in HCL are unquoted.**
The tag check originally searched for `"Name"` *with* quotes. In a `tags = { ... }` block
the key is written bare (`Name = "..."`), so the pattern matched zero times and every
required tag was reported missing even though all three were present.
**Fix:** match `^\s*Name\s*=` via `grep -P` instead.

**3. Pre-commit repo name.** The `pre-commit` hook repository is
`antonbabenko/pre-commit-terraform`, not `pre-commit-tf`; the wrong URL makes
`pre-commit install` fail to resolve the repo.

The script was also verified **negatively** — a scratch file containing a `BadName`
resource and a hardcoded `eu-west-1` region was correctly rejected with exit code 1.
See `docs/test-results.md`.

## Key Learnings

- **Layering matters more than any single tool.** `fmt`/`validate` catch syntax in
  milliseconds; plan parsing catches intent but costs an API call. Ordering them
  cheapest-first means the expensive check only runs on code that already passed.
- **Generic linters can't know your standards.** Cost-allocation tags and naming schemes
  are organization-specific — a 60-line bash script covers what no off-the-shelf tool will.
- **Plan output is a testable artifact.** `terraform show -json` turns an intended change
  into structured data you can assert against *before* it touches real infrastructure.
- **Testing your tests is not optional.** The convention script passed on clean code while
  being silently broken (bug #1 above); only a deliberate negative test proved it could fail.
- **Pre-commit hooks and CI are complements, not duplicates.** Hooks give instant feedback
  but are bypassable (`--no-verify`); CI is the enforcement boundary that actually blocks merge.

## Submission

1. `git add` → `git commit` → `git push` to your fork
2. Open a Pull Request from your fork to the original lab repo
3. Paste the PR URL into the **Lab Submission** field in the Student Portal
