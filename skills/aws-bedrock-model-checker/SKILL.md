---
name: aws-bedrock-model-checker
description: >
  Check whether AWS Bedrock foundation model IDs used in a project are deprecated or end-of-life,
  and automatically suggest or apply replacements with the latest equivalent models of similar price
  and capability. Use this skill whenever the user mentions Bedrock model IDs, model deprecation,
  model migration, LLM version upgrades, or asks to audit/scan code for outdated AWS Bedrock models.
  Also trigger when the user asks "is my Bedrock model still active?", "which Bedrock models are
  deprecated?", "update my Bedrock model ID", or "replace deprecated models in my code/config".
  Always use this skill for any task involving validating, auditing, or migrating AWS Bedrock model IDs.
---

# AWS Bedrock Model Deprecation Checker

Scans a project (or a single file/model ID) for AWS Bedrock model identifiers, checks each one
against the live AWS Bedrock API for deprecation / EOL status, and proposes (or applies) in-place
replacements using the best currently-active equivalent.

---

## Workflow

### Step 1 — Collect model IDs

**Option A: User provides a model ID directly**
Skip scanning; go straight to Step 2.

**Option B: Scan a file or project directory**
```bash
# Find all Bedrock model ID patterns in common config / code files
grep -rn \
  --include="*.py" --include="*.ts" --include="*.js" \
  --include="*.json" --include="*.yaml" --include="*.yml" \
  --include="*.tf" --include="*.env" --include="*.toml" \
  -E '["\047](([a-z]+\.(claude|llama|titan|nova|mistral|ai21|cohere|stability)[^"'\'']+)|(us\.|eu\.|ap\.)?(anthropic|meta|amazon|mistralai|ai21|cohere|stability)\.[a-z0-9._:-]+)["\047]' \
  <project_root>
```

Deduplicate and collect all matched model IDs.

### Step 2 — Query AWS Bedrock for model lifecycle status

Use the AWS CLI (preferred) or boto3 to get live status.

**AWS CLI (one model):**
```bash
aws bedrock get-foundation-model \
  --model-identifier <model-id> \
  --region us-east-1 \
  --query 'modelDetails.{id:modelId,status:modelLifecycle.status,provider:providerName}' \
  --output json
```

**AWS CLI (all models — list and filter):**
```bash
aws bedrock list-foundation-models \
  --region us-east-1 \
  --output json | python3 -c "
import json, sys
data = json.load(sys.stdin)
for m in data['modelSummaries']:
    lc = m.get('modelLifecycle', {}).get('status', 'UNKNOWN')
    print(json.dumps({'id': m['modelId'], 'status': lc,
                      'provider': m.get('providerName',''),
                      'name': m.get('modelName','')}))
"
```

**boto3 fallback (if CLI unavailable):**
```python
import boto3, json
client = boto3.client('bedrock', region_name='us-east-1')
paginator = client.get_paginator('list_foundation_models')
models = []
for page in paginator.paginate():
    for m in page['modelSummaries']:
        models.append({
            'id': m['modelId'],
            'status': m.get('modelLifecycle', {}).get('status', 'UNKNOWN'),
            'provider': m.get('providerName', ''),
            'name': m.get('modelName', ''),
        })
print(json.dumps(models, indent=2))
```

**Status values:**
- `ACTIVE` — safe to use
- `LEGACY` — deprecated; still callable but EOL is scheduled; **must migrate**
- `DEPRECATED` — EOL reached or imminent; API calls will fail; **migrate immediately**

### Step 3 — Find the best replacement

Run the script at `scripts/find_replacement.py` which implements the matching logic:

```bash
python3 scripts/find_replacement.py \
  --deprecated-id <model-id> \
  --all-models-json <path-to-models.json>
```

The script ranks active models by:
1. **Same provider** (e.g., anthropic → anthropic)
2. **Same model family** (e.g., `claude-3-5-sonnet` → prefer `claude-3-7-sonnet` or `claude-sonnet-4`)
3. **Similar capability tier** (sonnet→sonnet, haiku→haiku, opus→opus, lite→lite)
4. **Recency** (higher version number / newer date suffix wins)
5. **Price parity** (use the reference table in `references/pricing-tiers.md`)

### Step 4 — Report findings

Present a clear table:

| File | Line | Deprecated Model ID | Recommended Replacement | Status |
|------|------|--------------------|-----------------------|--------|
| app/bedrock.py | 14 | `anthropic.claude-3-sonnet-20240229-v1:0` | `anthropic.claude-sonnet-4-5-20251101-v1:0` | LEGACY |

For each replacement, explain **why** it was chosen (family match, tier match, active status).

### Step 5 — Apply replacements (with confirmation)

Always ask before modifying files. If the user confirms:

```bash
# Use sed or Python for safe in-place replacement
python3 scripts/apply_replacements.py \
  --replacements-json <replacements.json> \
  --dry-run false
```

Show a diff before confirming.

---

## Important notes

- **Region matters**: A model active in `us-east-1` may be LEGACY in `eu-west-1`. Default to `us-east-1`
  unless the user specifies otherwise. If cross-region, check each region separately.
- **Cross-region inference IDs**: IDs prefixed with `us.`, `eu.`, `ap.` are cross-region inference
  profiles — check the base model ID for lifecycle status.
- **Provisioned Throughput**: If the model is used with a Provisioned Throughput ARN, flag this
  separately — the user may need to update their PT configuration too.
- **No AWS credentials?** If `aws bedrock` commands fail with auth errors, tell the user and show
  them what to configure (`aws configure` or environment variables). Offer to check against the
  static reference table in `references/known-deprecated-models.md` as a fallback.

---

## Reference files

- `references/known-deprecated-models.md` — Static snapshot of known LEGACY/EOL models (use as
  fallback when AWS credentials are unavailable; always prefer live API data)
- `references/pricing-tiers.md` — Model capability and price tier mapping to guide replacements
- `scripts/find_replacement.py` — Replacement ranking logic
- `scripts/apply_replacements.py` — In-place file patching with dry-run support
