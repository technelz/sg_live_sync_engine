# Security Group Reconciliation Engine
A dependency‑aware DR parity tool for AWS Security Groups

This project provides a deterministic, dependency‑aware reconciliation engine for synchronizing Security Groups (SGs) from a **source AWS account/VPC** into a **target AWS account/VPC**.  
It is designed for **Disaster Recovery (DR) parity**, ensuring the target environment contains all SGs and SG rules required by the source.

The engine performs:

- SG matching  
- Drift detection  
- Dependency graph analysis  
- Safe rule reconciliation  
- Optional SG shell creation  
- Tag synchronization  
- Terraform‑like execution planning  
- Rollback journaling  
- JSON reporting  

---
## Folder Layout

security-group-sync/
├── sg_reconcile_engine.py        # Main engine
├── sg_config.json                # Manifest for source/target config
├── sg_reports/                   # Generated reports
├── rollback/                     # Rollback journals (apply mode)
└── README.md                     # Documentation

---
## What the Engine Does
The engine:

- Loads source SGs from a live AWS account  
- Loads target SGs from a live AWS account  
- Matches source → target SGs using:
  - exact name match  
  - CloudFormation tag match  
  - optional normalized‑name fallback (`--allow-legacy`)
- Builds a dependency graph of SG‑to‑SG references  
- Detects:
  - missing SGs  
  - missing required rules  
  - tag drift  
  - description drift  
  - unresolved SG references  
  - dependency cycles  
- Generates a Terraform‑like plan:
  - `create` (missing SGs)  
  - `update` (drifted SGs)  
  - `skip` (in‑sync SGs)  
- Executes the plan (apply mode)  
- Writes:
  - reconciliation report (`sg_reports/...`)  
  - rollback journal (`rollback/...`)  

---
## Reconciliation Policy

The engine operates in **source‑required‑only** mode:

### Required  
The target **must** contain all rules that exist in the source.

### Allowed  
The target **may** contain additional rules.

### Not performed  
- Removing extra target rules  
- Deleting target SGs  
- Full rollback automation  

This policy is ideal for DR environments where the target may contain:

- AWS‑managed rules  
- DRS‑managed rules  
- Environment‑specific rules  
- Temporary operational rules  

---
## Matching Logic

The engine attempts to match each source SG to a target SG using:

### 1. Exact name match (strict)
Fastest and safest.

### 2. CloudFormation tag match (strict)
Uses:

- `aws:cloudformation:stack-name`  
- `aws:cloudformation:logical-id`  

### 3. Normalized‑name fallback (legacy)
Enabled with:

--allow-legacy

Normalization removes:

- environment prefixes/suffixes  
- random suffixes  
- account IDs  
- directory IDs  
- non‑alphanumeric characters  

If multiple SGs normalize to the same value, the match becomes **ambiguous** and is blocked in apply mode.

---
## Drift Detection
The engine reports drift when:

- a required ingress rule is missing  
- a required egress rule is missing  
- tags differ  
- descriptions differ  
- a source SG is missing in the target  
- a referenced SG cannot be mapped  

The engine **does not** report drift for extra target rules.

---
## Reports
Reports are written to:

sg_reports/

Each report includes:

- source/target configuration  
- matching results  
- drift analysis  
- dependency issues  
- execution plan  
- apply results (apply mode)  

### Example Summary
json
{
  "source_total": 50,
  "excluded": {
    "directory_service": 1,
    "default_sg": 0
  },
  "evaluated": 49,
  "total": 49,
  "strict_matches": 49,
  "legacy_matches": 0,
  "missing": 0,
  "drifted": 35,
  "unresolved_dependencies": 0,
  "in_sync": 14
}

Summary Field Reference

Field	Meaning
source_total	Total SGs discovered in source (excluding default SG)
excluded.directory_service	SGs skipped because they belong to AWS Directory Service
excluded.default_sg	Default VPC SG excluded
evaluated	SGs actually processed (source_total − excluded)
total	Same as evaluated (backward compatibility)
strict_matches	SGs matched by canonical key
legacy_matches	SGs matched via legacy logic
missing	SGs missing in target
drifted	SGs with rule or tag drift
in_sync	SGs with no drift
unresolved_dependencies	SGs referencing SGs that could not be mapped


Rollback Journals
Apply mode writes rollback entries to:

rollback/
Rollback entries include:
created SGs
added rules
tag sync operations
This is not a full rollback engine, but an audit trail.

CLI Usage
Dry Run (recommended)
python sg_reconcile_engine.py --info-file sg_config.json --dry-run

Dry Run with legacy matching
python sg_reconcile_engine.py --info-file sg_config.json --dry-run --allow-legacy

Report Only
python sg_reconcile_engine.py --info-file sg_config.json --report-only

Apply Mode
python sg_reconcile_engine.py --info-file sg_config.json --yes

Manifest File (sg_config.json)
Example:

json
{
  "source": {
    "profile": "prod",
    "region": "us-west-2",
    "vpc_id": "vpc-xxxxxxxx"
  },
  "target": {
    "profile": "dr",
    "region": "us-east-1",
    "vpc_id": "vpc-yyyyyyyy"
  }
}

The engine loads all of the following from this file unless overridden by CLI flags:
source profile
source region
source VPC ID
target profile
target region
target VPC ID

CLI Flags
Flag	Description
--info-file	Path to manifest JSON
--dry-run	Preview only
--report-only	Generate report only
--yes	Apply changes
--allow-legacy	Enable normalized‑name matching
--allow-unresolved-dependencies	Allow rule application even when SG references are missing
--no-create-missing	Do not create missing SG shells
--no-sync-tags	Skip tag synchronization
--report-path	Custom report output path
--rollback-dir	Custom rollback directory
--workers	Parallelism (future use)


Recommended Workflow
Run dry‑run

Review the report

Fix ambiguous or missing SGs

Run apply

Run dry‑run again to confirm convergence

Expected final state:

missing: 0
drifted: 0
unresolved_dependencies: 0

### Best Practices
Always start with dry‑run

Review drift and dependency issues before apply

Avoid ambiguous SG names

Prefer exact name matches

Use --allow-legacy only when necessary