# Centralized Model Version Management

## Problem

Anthropic model strings are hardcoded across 54+ files in 9+ projects. When a new model drops (e.g., Opus 4.6), every project needs manual updates. Projects span AWS Lambda, Kubernetes, multiple languages (Python, TypeScript, Go), and three provider formats (Anthropic API, Bedrock, Vertex AI).

## Decision: Single Model, Single Source of Truth

Cost differences between tiers are negligible for current usage. All projects will use one model (currently `claude-opus-4-6`), resolved at deploy time from a central GitHub repo.

## Architecture

```
model-versions repo (public)
├── models.json          ← one model string
├── action.yml           ← composite action, outputs all 3 formats
└── .github/workflows/
    └── propagate.yml    ← triggers downstream deploys on merge
```

### models.json

```json
{
  "default": "claude-opus-4-6"
}
```

### Format Derivation

The composite action takes the canonical Anthropic model ID and derives:

| Provider | Rule | Example |
|----------|------|---------|
| Anthropic | as-is | `claude-opus-4-6` |
| Bedrock | `anthropic.{id}-v1` | `anthropic.claude-opus-4-6-v1` |
| Vertex | `publishers/anthropic/models/{id}` | `publishers/anthropic/models/claude-opus-4-6` |

### Deploy Propagation

On merge to `main` (when `models.json` changes), a workflow triggers `workflow_dispatch` on all downstream repos via `gh workflow run`. Requires a `DEPLOY_PAT` with `actions:write` scope.

Fan-out, not cascade: each downstream deploy is independent. Failures are visible in the model-versions Actions tab.

### Downstream Consumption

Each project's deploy workflow adds:

```yaml
- name: Get model versions
  id: models
  uses: quinnypig/model-versions@main
```

Then passes the appropriate output (`default`, `default-bedrock`, `default-vertex`) as an environment variable to the deploy step.

Application code reads `CLAUDE_MODEL` from the environment with a hardcoded fallback for local dev:

- Python: `os.environ.get("CLAUDE_MODEL", "claude-opus-4-6")`
- TypeScript: `process.env.CLAUDE_MODEL ?? "claude-opus-4-6"`
- Go: `os.Getenv("CLAUDE_MODEL")` with fallback

## Migration Order

| Order | Project | Notes |
|-------|---------|-------|
| 1 | snarknado | Already centralized in constants.ts, good proof of concept |
| 2 | artificialconfidence | Centralized settings.py, tests Vertex format |
| 3 | newsletter-v2 | Six Lambda files, biggest consolidation win |
| 4 | lwia-autoresponder | Python + TS, covers both patterns |
| 5 | secret-agent-corey | Nine specialist files |
| 6 | leakdetector | Two models collapse to one |
| 7 | pricetracker | Bedrock format path |
| 8 | shitposting-trip-reports | Two files |
| 9 | talks | One file |

## What Changes Per Project

1. Add composite action step to deploy workflow
2. Pass model output as env var to deploy target (CDK env, k8s manifest, Docker env)
3. Replace hardcoded model strings with env var read + fallback
4. Remove unused tier constants / multiple model variables
