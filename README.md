# tagging.sh — End-to-End Explanation (Current Behavior)

This document explains how `tagging.sh` works from start to finish, reflecting
the current logic and file layout.

## 1) Inputs and configuration files

The script depends on these files (all expected in the same folder as
`tagging.sh`):

- `tags_by_account.json`: tag values (default + per-account overrides)
- `required_tags.json`: list of required tag keys
- `tag_definitions.json`: per-tag scope rules and tag source metadata
- `tag_scope.json`: legacy scope rules (ignored if `tag_definitions.json` exists)
- `resource_scope.json`: list of AWS services to scan (optional)
- `accounts_by_env.json`: lists of accounts by environment (optional)

If `tags_by_account.json` is missing, it falls back to `tags.json`.

## 2) Startup defaults and safety

- The script fails fast (`set -euo pipefail`).
- Default regions: `us-east-1` and `us-west-2` (unless overridden).
- Container control-plane resources are excluded unless `--include-containers`.
- Limits:
  - `--max-scan` stops after N missing resources.
  - `--max-tag` stops after N tag updates.

## 3) Load required tags and scope

The script loads tag values and required keys:

- Tag **values** are loaded from `tags_by_account.json` (default + overrides).
- If `required_tags.json` exists, it replaces the required list.

Scope logic:

- If `tag_definitions.json` exists, it provides per-tag scope rules and source.
- If not, `tag_scope.json` is used.
- If `resource_scope.json` exists, only those services are scanned.

## 4) Discover resources

For each region, it calls:

```
aws resourcegroupstaggingapi get-resources --region <region>
```

Then it filters to:
- taggable resources
- in-scope services (if `resource_scope.json` is set)
- optional resource-type filters (`--resource-type`, `--ec2-only`)

## 5) Derive resource metadata

Each resource is normalized to:

- `resource_type`: e.g. `ec2:instance`, `iam:policy`
- `resource_name`: Name tag if present, otherwise ARN tail

These fields are written into the report for easy lookup.

## 6) Find missing tags

For each resource:
- All required keys are checked.
- Per-tag scope rules determine whether the tag applies to that resource.
- The script builds:
  - `missing_tag_keys`: all missing required keys
  - `missing_tag_values`: missing keys that have values configured

If a required tag has no configured value, it appears in
`missing_tag_keys` only.

## 7) Created-by details

The script uses CloudTrail to find creator info. If CloudTrail does not
return data, AWS Config is used to get a creation timestamp.

Output fields:
- `created_timestamp`
- `created_by`
- `created_user_agent`
- `created_role_session`
- `created_via` (`terraform|manual|unknown`)

## 8) Special rules for provisioner tags

The script enforces document-specific logic:

- `aexp-provisioner-workspace`
  - If the resource is an AMI (`ec2:image`), value is `jenkins-packer`.
  - Otherwise, missing values fall back to `na`.

- `aexp-provisioner-repo`
  - Required only for automated provisioning.
  - If `created_via` is not `terraform`, the tag is removed from missing lists.
  - If automation but no configured value, it falls back to `na`.

## 9) Report output

The script writes `missing_tag_report.csv` with:

```
scan_timestamp,account_id,region,resource_arn,resource_type,resource_name,
missing_tag_keys,missing_tag_values,change_number,created_timestamp,created_by,
created_user_agent,created_role_session,created_via
```

This report is always generated first, even in report-only mode.

## 10) Apply tags (optional)

If `--apply` is used (or `--prompt` is accepted), the script:
- calls `tag-resources`
- only applies tags with configured values
- respects `--max-tag` and `--max-scan`

## 11) Multi-account runs

If `--env` or `--accounts-file` is provided, accounts are processed
one by one, with separate report and log files per account.

## 12) Common usage

Report-only:
```
./tagging.sh --profile <profile> --account-id <id> --region us-east-1
```

Apply tags:
```
./tagging.sh --profile <profile> --account-id <id> --region us-east-1 --apply
```

Limit scan size:
```
./tagging.sh --profile <profile> --account-id <id> --region us-east-1 --max-scan 10
```
