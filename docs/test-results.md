# Test Results — Lab M5.05

Captured locally on 2026-08-06. Terraform v1.x, AWS provider v5.100.0,
account `697345203222`, region `us-east-1`.

## Layer 1 — Static Analysis

### `terraform fmt -check -recursive`

```
$ terraform fmt -check -recursive
$ echo "Exit code: $?"
Exit code: 0
```

No output and exit `0` means every `.tf` file is already canonically formatted.

### `terraform init -backend=false`

```
Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.0"...
- Installing hashicorp/aws v5.100.0...
- Installed hashicorp/aws v5.100.0 (signed by HashiCorp)

Terraform has been successfully initialized!
```

### `terraform validate`

```
Success! The configuration is valid.
```

### `tflint`

Not run locally — TFLint is not installed on this machine. It executes in the
`static-analysis` CI job, which installs it via the official `install_linux.sh`
script and runs `tflint --init && tflint` against `.tflint.hcl`.

## Layer 2 — Convention Validation

### Positive test (the real configuration)

```
$ ./scripts/validate-conventions.sh
=== Convention Validation ===

Checking naming conventions...
  PASS: All resource names follow conventions

Checking required tags...
  PASS: Tag 'Name' found
  PASS: Tag 'Environment' found
  PASS: Tag 'ManagedBy' found

Checking variable descriptions...
  PASS: All 4 variables have descriptions

Checking for hardcoded AWS regions...
  PASS: No hardcoded regions

=== Results ===
PASSED: All convention checks passed
EXIT: 0
```

### Negative test (deliberately broken config)

A scratch copy of the configuration was given an extra `bad.tf`:

```hcl
resource "aws_sns_topic" "BadName" {
  name   = "oops"
  region = "eu-west-1"
}
```

Result — both violations detected, non-zero exit:

```
=== Convention Validation ===

Checking naming conventions...
  FAIL: bad.tf:1 — resource name 'BadName' must be lowercase with underscores

Checking required tags...
  PASS: Tag 'Name' found
  PASS: Tag 'Environment' found
  PASS: Tag 'ManagedBy' found

Checking variable descriptions...
  PASS: All 4 variables have descriptions

Checking for hardcoded AWS regions...
  FAIL: Hardcoded region found:
  bad.tf:3:  region = "eu-west-1"

=== Results ===
FAILED: 2 convention violation(s) found
EXIT: 1
```

This confirms the script fails the build rather than passing vacuously.

## Layer 3 — Plan Validation

`jq` is not installed on this Windows machine, so `scripts/validate-plan.sh` itself
runs in CI (GitHub's `ubuntu-latest` ships `jq`). The same plan was generated locally
and the identical assertions were evaluated with Python to prove the logic holds.

### `terraform plan -out=tfplan`

```
Plan: 4 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + bucket_arn          = (known after apply)
  + bucket_name         = (known after apply)
  + dynamodb_table_name = "infratest-dev-lock-table"

Saved the plan to: tfplan
```

### Assertions against `terraform show -json tfplan`

```
Resources to create: 4
  PASS: aws_s3_bucket (1 instance(s))
  PASS: aws_s3_bucket_versioning (1 instance(s))
  PASS: aws_s3_bucket_server_side_encryption_configuration (1 instance(s))
  PASS: aws_dynamodb_table (1 instance(s))
Destroys: 0
```

All four expected resource types present, exactly 4 creates, no destroys —
which is what `validate-plan.sh` asserts.

## Pre-Commit Hook

```
$ ./scripts/install-hooks.sh
Pre-commit hook installed successfully.

$ ls -l .git/hooks/pre-commit
-rwxr-xr-x 1 beite 197609 739 Aug  6 09:12 .git/hooks/pre-commit
```

The hook runs `terraform fmt -check`, `terraform validate`, and
`validate-conventions.sh`, aborting the commit on the first failure.

## Summary

| Layer | Check | Status |
|---|---|---|
| 1 | `terraform fmt -check -recursive` | PASS |
| 1 | `terraform init -backend=false` | PASS |
| 1 | `terraform validate` | PASS |
| 1 | `tflint` | Runs in CI |
| 2 | Convention validation (positive) | PASS |
| 2 | Convention validation (negative) | PASS — correctly exits 1 |
| 3 | Plan resource-type assertions | PASS (4/4 types, 0 destroys) |
| 4 | Pre-commit hook installed | PASS |
