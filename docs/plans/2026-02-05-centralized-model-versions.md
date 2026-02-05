# Centralized Model Versions Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Centralize all Anthropic model strings into a single repo so updating one JSON file triggers deploys across 9 projects.

**Architecture:** A public GitHub repo (`model-versions`) holds one model ID in `models.json`. A composite GitHub Action derives Anthropic/Bedrock/Vertex formats. A propagation workflow triggers downstream deploys on merge. Each project reads `CLAUDE_MODEL` from the environment with a hardcoded fallback.

**Tech Stack:** GitHub Actions (composite action, workflow_dispatch), shell, jq. No new infrastructure.

---

### Task 0: Create the model-versions repo

**Files:**
- Create: `models.json`
- Create: `action.yml`
- Create: `.github/workflows/propagate.yml`
- Create: `README.md`

**Step 1: Create models.json**

```json
{
  "default": "claude-opus-4-6"
}
```

**Step 2: Create the composite action**

Create `action.yml`:

```yaml
name: 'Get Model Versions'
description: 'Provides current Anthropic model IDs in all provider formats'

outputs:
  default:
    description: 'Anthropic API model ID'
    value: ${{ steps.resolve.outputs.default }}
  default-bedrock:
    description: 'AWS Bedrock model ID'
    value: ${{ steps.resolve.outputs.default-bedrock }}
  default-vertex:
    description: 'Vertex AI model ID'
    value: ${{ steps.resolve.outputs.default-vertex }}

runs:
  using: composite
  steps:
    - name: Resolve model versions
      id: resolve
      shell: bash
      run: |
        CONFIG=$(curl -sf "https://raw.githubusercontent.com/quinnypig/model-versions/main/models.json")
        MODEL=$(echo "$CONFIG" | jq -r '.default')

        echo "default=${MODEL}" >> "$GITHUB_OUTPUT"
        echo "default-bedrock=anthropic.${MODEL}-v1" >> "$GITHUB_OUTPUT"
        echo "default-vertex=publishers/anthropic/models/${MODEL}" >> "$GITHUB_OUTPUT"
```

**Step 3: Create the propagation workflow**

Create `.github/workflows/propagate.yml`:

```yaml
name: Propagate model update
on:
  push:
    branches: [main]
    paths: [models.json]

jobs:
  trigger-deploys:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        repo:
          - snarknado
          - newsletter-v2
          - lwia-autoresponder
          - secret-agent-corey
          - leakdetector
          - pricetracker
          - shitposting-trip-reports
    steps:
      - name: Trigger deploy
        env:
          GH_TOKEN: ${{ secrets.DEPLOY_PAT }}
        run: gh workflow run deploy.yml --repo quinnypig/${{ matrix.repo }}
```

Note: `artificialconfidence` (Cloud Build) and `talks` (no deploy) are excluded from propagation. See their tasks for details.

**Step 4: Create the PAT**

Go to GitHub Settings → Developer Settings → Personal access tokens → Fine-grained tokens. Create `model-versions-deploy` with:
- Repository access: select all 7 downstream repos
- Permissions: Actions (Read and write)

Add as `DEPLOY_PAT` secret in the model-versions repo settings.

**Step 5: Init repo and push**

```bash
cd /home/cquinn/src/model-versions
git init
git add models.json action.yml .github/workflows/propagate.yml README.md
git commit -m "Initial commit: centralized model version config"
gh repo create quinnypig/model-versions --public --source=. --push
```

---

### Task 1: Migrate snarknado (proof of concept)

**Files:**
- Modify: `src/lib/ai/constants.ts` (lines 10-14)
- Modify: `.github/workflows/deploy.yml`

**Step 1: Update constants.ts**

Replace the MODELS object at `src/lib/ai/constants.ts:10-14`:

```typescript
// Before:
export const MODELS = {
  FAST: "claude-sonnet-4-5-20250929",
  GENERATION: "claude-sonnet-4-5-20250929",
} as const;

// After:
export const MODEL = process.env.CLAUDE_MODEL ?? "claude-opus-4-6";
```

**Step 2: Find and update all references to MODELS.FAST and MODELS.GENERATION**

Search for `MODELS.FAST` and `MODELS.GENERATION` across the project. Replace with `MODEL`.

**Step 3: Update deploy workflow**

Add the model-versions step to `.github/workflows/deploy.yml`, before the build/deploy step:

```yaml
      - name: Get model versions
        id: models
        uses: quinnypig/model-versions@main

      # Then pass to the deploy step:
      env:
        CLAUDE_MODEL: ${{ steps.models.outputs.default }}
```

Ensure `CLAUDE_MODEL` is passed through to wherever the app runs (Lambda env var in CDK, container env, etc.).

**Step 4: Commit and push**

```bash
git add src/lib/ai/constants.ts .github/workflows/deploy.yml
git commit -m "feat: centralize model version from model-versions repo"
git push
```

**Step 5: Verify deploy succeeds**

Check GitHub Actions for the deploy run. Confirm the model string is correctly resolved.

---

### Task 2: Migrate artificialconfidence (Vertex AI / Cloud Build)

This project uses GCP Cloud Build + Terraform, not GitHub Actions. Two changes needed.

**Files:**
- Modify: `src/artificialconfidence/settings.py` (lines 32-38)
- Modify: `src/artificialconfidence/processing/swarm/agents.py` (lines 42, 121, 141, 175, 195)
- Modify: `infra/cloudrun.tf` (Cloud Run env vars)

**Step 1: Update settings.py to read from environment**

```python
# Before:
claude_model: str = "claude-sonnet-4-5@20250929"
editor_model: str = "claude-opus-4-5@20251101"
swarm_critic_model: str = "claude-sonnet-4-5@20250929"
swarm_gate_model: str = "claude-opus-4-5@20251101"

# After (all tiers collapse to one model):
claude_model: str = os.environ.get("CLAUDE_MODEL", "publishers/anthropic/models/claude-opus-4-6")
editor_model: str = os.environ.get("CLAUDE_MODEL", "publishers/anthropic/models/claude-opus-4-6")
swarm_critic_model: str = os.environ.get("CLAUDE_MODEL", "publishers/anthropic/models/claude-opus-4-6")
swarm_gate_model: str = os.environ.get("CLAUDE_MODEL", "publishers/anthropic/models/claude-opus-4-6")
```

Wait — check: does artificialconfidence use the Vertex AI SDK which expects the `publishers/anthropic/models/` prefix, or just the `@` format? Check `settings.py` and how these strings are consumed. The `@` format suggests the Vertex AI Python SDK's native format. Verify what the SDK expects before choosing the format.

If the SDK uses `@` format: the fallback should be `claude-opus-4-6@YYYYMMDD` or whatever the SDK expects. If it takes the full resource path: use the `publishers/...` format.

**Step 2: Update swarm agents.py default parameters**

Replace all hardcoded model defaults in agent `__init__` methods to use `settings` instead of inline strings. These should already be reading from settings — if they're not, wire them through.

**Step 3: Add CLAUDE_MODEL to Cloud Run environment**

In `infra/cloudrun.tf`, add to both the web service and pipeline job containers:

```hcl
env {
  name  = "CLAUDE_MODEL"
  value = var.claude_model
}
```

Add to `variables.tf`:

```hcl
variable "claude_model" {
  description = "Vertex AI model ID for Claude"
  type        = string
  default     = "publishers/anthropic/models/claude-opus-4-6"
}
```

**Step 4: Propagation path**

Since this project uses Cloud Build (not GitHub Actions), it's excluded from automatic propagation. To update the model:

Option A (manual, quarterly): Update `variables.tf` default and push → Cloud Build triggers.

Option B (automated): Add a minimal `.github/workflows/deploy.yml` with `workflow_dispatch` that runs `gcloud run services update` to set the new env var. Then add this repo to the propagation matrix.

Recommend Option A for now. Automated propagation can be added later if the manual step is annoying.

**Step 5: Commit and push**

```bash
git add settings.py agents.py infra/cloudrun.tf infra/variables.tf
git commit -m "feat: centralize model version via CLAUDE_MODEL env var"
git push
```

---

### Task 3: Migrate newsletter-v2

**Files:**
- Modify: `src/lambda/shared/ai-generation.ts:122`
- Modify: `src/lambda/shared/ai-review.ts:143`
- Modify: `src/lambda/shared/item-evaluation.ts:83`
- Modify: `src/lambda/shared/joke-rating.ts:64`
- Modify: `src/lambda/shared/newsletter-curation.ts:263,347`
- Modify: `src/lambda/api-handler.ts:1562`
- Modify: `.github/workflows/deploy.yml`

**Step 1: Create a shared constant**

Add to `src/lambda/shared/constants.ts` (or whichever shared constants file exists):

```typescript
export const CLAUDE_MODEL = process.env.CLAUDE_MODEL ?? "claude-opus-4-6";
```

**Step 2: Replace all 7 hardcoded model strings**

In each file listed above, replace the inline model string with the imported constant:

```typescript
import { CLAUDE_MODEL } from './constants';
// ...
model: CLAUDE_MODEL,
```

For `api-handler.ts`, adjust the import path as needed.

**Step 3: Update deploy workflow**

Add to `.github/workflows/deploy.yml` before the CDK deploy step:

```yaml
      - name: Get model versions
        id: models
        uses: quinnypig/model-versions@main
```

Pass `CLAUDE_MODEL` to the CDK deploy command as an environment variable. Ensure the CDK stack passes it through to the Lambda environment.

Check: the CDK stack likely already defines Lambda env vars. Add `CLAUDE_MODEL: process.env.CLAUDE_MODEL` to that config.

**Step 4: Commit and push**

---

### Task 4: Migrate lwia-autoresponder

**Files:**
- Modify: `src/lambdas/classifier/constants.py:5`
- Modify: `dashboard/app/api/inbox/[id]/handle/route.ts:130`
- Modify: `dashboard/app/api/drafts/[id]/regenerate/route.ts:139`
- Modify: `dashboard/app/api/drafts/regenerate-all/route.ts:148`
- Modify: `dashboard/app/api/drafts/purge/route.ts:152`
- Modify: `.github/workflows/deploy.yml`

**Step 1: Update Python classifier**

```python
# Before:
CLAUDE_MODEL = "claude-sonnet-4-5"

# After:
import os
CLAUDE_MODEL = os.environ.get("CLAUDE_MODEL", "claude-opus-4-6")
```

**Step 2: Update dashboard API routes**

Create or use a shared constant in the dashboard:

```typescript
// dashboard/app/lib/constants.ts (or similar)
export const CLAUDE_MODEL = process.env.CLAUDE_MODEL ?? "claude-opus-4-6";
```

Replace all 4 route files' hardcoded model strings with this import.

**Step 3: Update deploy workflow**

Add composite action step. Pass `CLAUDE_MODEL` to both Lambda deploy and dashboard deploy steps.

**Step 4: Commit and push**

---

### Task 5: Migrate secret-agent-corey

**Files:**
- Modify: `agents/specialists/aws_cost.py:46`
- Modify: `agents/specialists/authenticity.py:96`
- Modify: `agents/specialists/architecture.py:55`
- Modify: `agents/specialists/career.py:61`
- Modify: `agents/specialists/conference.py:64`
- Modify: `agents/specialists/devops.py:59`
- Modify: `agents/specialists/scorer.py:68`
- Modify: `agents/specialists/snark.py:64`
- Modify: `agents/specialists/tech_news.py:52`
- Modify: `agents/quick_voice.py:42`
- Modify: `.github/workflows/deploy.yml`

**Step 1: Create a shared constant**

```python
# agents/constants.py (or wherever config lives)
import os
CLAUDE_MODEL = os.environ.get("CLAUDE_MODEL", "claude-opus-4-6")
```

**Step 2: Replace all 10 specialist files**

Each file changes from inline `model_id="claude-sonnet-4-..."` to:

```python
from agents.constants import CLAUDE_MODEL
# ...
model_id=CLAUDE_MODEL,
```

`quick_voice.py` was using Haiku — it also gets the same model now (Opus for everything).

**Step 3: Update deploy workflow and commit**

---

### Task 6: Migrate leakdetector

**Files:**
- Modify: `lambdas/llm_analyzer/handler.py:90,132,171`
- Modify: `.github/workflows/deploy.yml`

**Step 1: Replace model strings**

This file currently uses Haiku for triage (lines 90, 171) and Sonnet for deep analysis (line 132). All three become one:

```python
import os
CLAUDE_MODEL = os.environ.get("CLAUDE_MODEL", "claude-opus-4-6")
```

Replace all three `model=` references with `model=CLAUDE_MODEL`.

**Step 2: Update deploy workflow and commit**

---

### Task 7: Migrate pricetracker

**Files:**
- Modify: `lambdas/summarize_notify/handler.py:80`
- Modify: `cdk/stacks/notification.py:79`
- Modify: `.github/workflows/deploy.yml`

**Step 1: Update handler to use env var (Bedrock format)**

```python
import os
CLAUDE_MODEL = os.environ.get("CLAUDE_MODEL_BEDROCK", "anthropic.claude-opus-4-6-v1")
```

Note: this project uses Bedrock, so it needs `default-bedrock` from the composite action, not `default`.

**Step 2: Update CDK IAM policy**

In `cdk/stacks/notification.py:79`, the Bedrock model ARN pattern needs to match the new model. Update the wildcard to be broad enough:

```python
resources=["arn:aws:bedrock:*::foundation-model/anthropic.claude-*"]
```

**Step 3: Update deploy workflow**

Pass `CLAUDE_MODEL_BEDROCK` from the composite action's `default-bedrock` output:

```yaml
      - name: Get model versions
        id: models
        uses: quinnypig/model-versions@main

      - name: CDK Deploy
        env:
          CLAUDE_MODEL_BEDROCK: ${{ steps.models.outputs.default-bedrock }}
```

**Step 4: Add workflow_dispatch trigger**

This is the one repo missing it. Add to the `on:` block:

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:
```

**Step 5: Commit and push**

---

### Task 8: Migrate shitposting-trip-reports

**Files:**
- Modify: `scripts/email-campaign/generate.ts:87`
- Modify: `lib/corey-api/client.ts:130`
- Modify: `.github/workflows/deploy.yml`

**Step 1: Replace model strings with env var read**

In both files:

```typescript
const MODEL = process.env.CLAUDE_MODEL ?? "claude-opus-4-6";
```

Or create a shared constant if there's an existing constants file.

**Step 2: Update deploy workflow and commit**

---

### Task 9: Migrate talks

**Files:**
- Modify: `scripts/lib/image-generator.js:146`

No deploy workflow exists. This is a local dev tool.

**Step 1: Replace model string with env var read**

```javascript
const MODEL = process.env.CLAUDE_MODEL ?? "claude-opus-4-6";
```

**Step 2: Commit and push**

No propagation needed. When the model updates, this picks it up if `CLAUDE_MODEL` is set, otherwise falls back to the hardcoded default. Update the fallback string manually when convenient.

---

### Task 10: Update test files

Several projects have tests that reference specific model strings. These need updating to match the new behavior.

**Files to check:**
- `artificialconfidence/tests/test_settings.py:66-67`
- `artificialconfidence/tests/test_swarm_agents.py`
- `artificialconfidence/tests/test_pipeline.py:30-31`
- Any other test files that assert on model strings

**Step 1:** Search each project for model strings in test files.

**Step 2:** Update assertions to match the new default (`claude-opus-4-6` or the appropriate format).

**Step 3:** Run each project's test suite to confirm nothing breaks.

---

### Task 11: End-to-end validation

**Step 1:** Update `models.json` in the model-versions repo to a test value (e.g., change to `claude-opus-4-6` if not already set).

**Step 2:** Push to main and verify the propagation workflow triggers all downstream deploys.

**Step 3:** Check one deploy per provider format:
- Anthropic: verify snarknado deployment logs show correct model
- Bedrock: verify pricetracker deployment logs show `anthropic.claude-opus-4-6-v1`
- Vertex: verify artificialconfidence uses correct Vertex model ID (manual deploy)

**Step 4:** Spot-check one Lambda and one container to confirm `CLAUDE_MODEL` env var is set correctly at runtime.
