# APIThreshold GitHub Action — proof repo

Minimal OpenAPI spec plus a **workflow** (`.github/workflows/apithreshold-gate.yml`) that runs `apithreshold gate` in CI. The workflow **checks out `apithreshold/backend` with `actions/checkout`** (using your PAT when the repo is private), then **`pip install` that directory**. That avoids `pip install git+https://…` with embedded tokens, which often still clones anonymously and returns **403** on private repos.

Canonical home once published: **`apithreshold/apithreshold-action-demo`** on GitHub ([github.com/apithreshold/apithreshold-action-demo](https://github.com/apithreshold/apithreshold-action-demo)).

## Why this lives outside `backend/`

This is a **standalone git repository** you push under the **`apithreshold` org**. It is not part of the `backend`, `guardrails`, or `portal` repos.

## Quick start

1. Copy this folder somewhere (or use it in place and `git init` here).

   ```bash
   cp -R apithreshold-action-demo ~/apithreshold-action-demo
   cd ~/apithreshold-action-demo
   ```

2. Initialize and push to a new **public** repo under the **`apithreshold` GitHub org** (repo name: `apithreshold-action-demo`).

   ```bash
   git init
   git add .
   git commit -m "Proof: APIThreshold gate in GitHub Actions"
   gh repo create apithreshold/apithreshold-action-demo --public --source=. --remote=origin --push
   ```

   Requires `gh` auth with permission to create repos in **apithreshold**. Or create **apithreshold/apithreshold-action-demo** in the org UI, then:

   ```bash
   git remote add origin https://github.com/apithreshold/apithreshold-action-demo.git
   git push -u origin main
   ```

   **If `gh` prints `Unable to add remote "origin"`:** the repo was created on GitHub, but this clone already has an `origin` (for example from an earlier `git init` in the template). Point it at the new repo and push:

   ```bash
   git remote set-url origin https://github.com/apithreshold/apithreshold-action-demo.git
   git push -u origin main
   ```

   To confirm: `git remote -v`.

3. In GitHub: **Settings → Secrets and variables → Actions → New repository secrets**

   | Name | When |
   |------|------|
   | **`OPENAI_API_KEY`** | Always — used by `apithreshold gate` for the LLM. |
   | **`BACKEND_GITHUB_TOKEN`** | **If `apithreshold/backend` is private** (default for many orgs). CI cannot `git clone` a private repo without credentials. |

   **PAT for `BACKEND_GITHUB_TOKEN`:** GitHub → **Settings** (your account or a bot user) → **Developer settings** → **Personal access tokens**. Create a **fine-grained** token with **Contents: Read** on repository **`apithreshold/backend`** only (minimum scope). Paste the token as secret **`BACKEND_GITHUB_TOKEN`** on **`apithreshold-action-demo`**. If your org uses **SAML SSO**, authorize the token for the org.

   If **`backend` is public**, you can omit creating **`BACKEND_GITHUB_TOKEN`**; `${{ secrets.BACKEND_GITHUB_TOKEN }}` is then empty and the install step clones anonymously.

   **Pull requests from forks** do not receive repository secrets; those workflow runs will fail the clone step unless **`backend`** is public. Same-repo PRs are fine.

4. Open **Actions** and confirm the workflow run completes. The default mode is **`learning`** (set via workflow `env` `APITHRESHOLD_MODE`) so the gate collects data and does not block on score (good for a first green run). Set **`APITHRESHOLD_MODE`** to **`enforcing`** in `.github/workflows/apithreshold-gate.yml` when you want CI to fail on threshold.

## Install source (PyPI vs backend repo)

**`apithreshold` is not on PyPI yet.** The workflow defaults to **`INSTALL_SOURCE: git`**, which checks out **`BACKEND_REPOSITORY`** at ref **`BACKEND_REF`**, then runs `pip install ./_backend_src`.

When the package is published, set **`INSTALL_SOURCE: pypi`** in the workflow `env` and remove or ignore the backend checkout step if you want a slimmer job.

**Pin a commit or tag** (reproducible CI): set workflow `env`, for example:

```yaml
env:
  BACKEND_REPOSITORY: apithreshold/backend
  BACKEND_REF: v0.1.0
```

Use a real tag or branch on **apithreshold/backend**, or a commit SHA.

**403 on checkout:** **`BACKEND_GITHUB_TOKEN`** is missing, invalid, lacks **Contents: Read** on **`apithreshold/backend`**, or is not **SSO-authorized** for the org. Fork PRs do not receive repository secrets unless you use another mechanism.

**Private fork:** set **`BACKEND_REPOSITORY`** to `your-org/your-fork` and use a PAT with access to that fork.

## Costs and runtime

`apithreshold gate` runs the full LLM pipeline (parse → assess → generate → score → evaluate). Expect **minutes** per run and **API spend**; use a tiny spec (this repo’s `openapi.yaml`) to keep both smaller.

## Next steps (product hardening)

- Pin a **released** version: `pip install apithreshold==0.x.y`.
- Add **`--project-id`** stability (already set to `${{ github.repository }}` in the workflow).
- When the CLI supports **`--ci`** and **`GITHUB_OUTPUT` gate state**, extend the workflow and document passing state between runs (see your Pre-Beta technical track).
