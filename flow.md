# Tagging Script Flow (Detailed)

```mermaid
flowchart TD
  A[Start] --> B[Parse CLI args: --profile, --account-id, --region, --apply, --prompt]
  B --> C[Check tools installed: aws, jq]
  C --> D[Load required tags]
  D --> D1{tags_by_account.json exists?}
  D1 -->|Yes| D2[Pick account entry or default]
  D1 -->|No| D3[Fallback to tags.json]
  D2 --> E[Validate tag key/value rules]
  D3 --> E
  E --> F[Write report header missing_tag_report.csv]
  F --> G[Resolve regions list]
  G --> G1{REGIONS preset?}
  G1 -->|Yes| H[Use preset regions (us-east-1, us-west-2)]
  G1 -->|No| I[List regions via ec2:DescribeRegions]
  H --> J[Report-only scan pass]
  I --> J

  J --> K[For each region call get-resources]
  K --> L[For each resource]
  L --> M[Compute missing required tags]
  M --> N[Lookup CloudTrail create events for ARN]
  N --> O{CloudTrail match found?}
  O -->|Yes| P[Set created_by + created_time from CloudTrail]
  O -->|No| Q[Fallback to AWS Config capture time]
  P --> R[Write CSV row]
  Q --> R
  R --> S{Apply mode?}
  S -->|No| T[Log PENDING or missing-value warnings]
  S -->|Yes| U[Call tag-resources for missing tags]
  U --> V[Log tagging success/failure]
  T --> W[Continue to next resource]
  V --> W
  W --> X[Region summary]

  X --> Y{--prompt?}
  Y -->|Yes| Z[Ask user to apply tags]
  Y -->|No| AA{--apply?}
  Z --> AB{User confirmed?}
  AB -->|Yes| AC[Second pass: apply tags]
  AB -->|No| AD[Skip tagging pass]
  AA -->|Yes| AC
  AA -->|No| AD
  AC --> AE[Repeat scan in apply mode]
  AE --> AF[Write final summaries]
  AD --> AF
  AF --> AG[End]
```
