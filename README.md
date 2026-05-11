# APIThreshold GitHub Action — proof repo

Minimal OpenAPI spec plus a **local composite action** and workflow to prove `apithreshold gate` in CI.

## Why this lives outside `backend/`

This is a **standalone git repository** you push to GitHub. It is not part of the `backend`, `guardrails`, or `portal` repos.

## Quick start

1. Copy this folder somewhere (or use it in place and `git init` here).

   ```bash
   cp -R apithreshold-action-demo ~/apithreshold-action-demo
   cd ~/apithreshold-action-demo
   ```

2. Initialize and push to a new **public** GitHub repository (name is arbitrary).

   ```bash
   git init
   git add .
   git commit -m "Proof: APIThreshold gate in GitHub Actions"
   gh repo create apithreshold-action-demo --public --source=. --remote=origin --push
   ```

   Or create the repo in the UI, then `git remote add origin …` and `git push -u origin main`.

3. In GitHub: **Settings → Secrets and variables → Actions → New repository secret**

   - Name: `OPENAI_API_KEY`
   - Value: your OpenAI API key (or a key for whatever provider you configure via `APITHRESHOLD_LLM_PROVIDER`).

4. Open **Actions** and confirm the workflow run completes. The default mode is **`learning`** so the gate collects data and does not block on score (good for a first green run). Switch to **`enforcing`** in `.github/workflows/apithreshold-gate.yml` when you want CI to fail on threshold.

## Install source

By default the composite action tries **`pip install apithreshold`** (PyPI). If the package is not published yet, set workflow input **`install-source`** to **`git`** so it installs from `https://github.com/apithreshold/backend.git` instead.

## Costs and runtime

`apithreshold gate` runs the full LLM pipeline (parse → assess → generate → score → evaluate). Expect **minutes** per run and **API spend**; use a tiny spec (this repo’s `openapi.yaml`) to keep both smaller.

## Next steps (product hardening)

- Pin a **released** version: `pip install apithreshold==0.x.y`.
- Add **`--project-id`** stability (already set to `${{ github.repository }}` in the workflow).
- When the CLI supports **`--ci`** and **`GITHUB_OUTPUT` gate state**, extend `action.yml` and document passing state between runs (see your Pre-Beta technical track).
