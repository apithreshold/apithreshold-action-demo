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

   If **`apithreshold/backend` is public**, you can omit **`BACKEND_GITHUB_TOKEN`**: the workflow uses `GITHUB_TOKEN` from **this** repo for `actions/checkout`, which is enough to read a **public** dependency. It is **not** enough to read a **private** sibling repo.

   **Pull requests from forks** do not receive **this** repository’s Action secrets, so `BACKEND_GITHUB_TOKEN` is empty on those runs. Checkout then uses `GITHUB_TOKEN`, which fails with **403** if **`backend`** is private. Same-repo branches and PRs receive secrets normally.

   **Organization secrets:** if you store the PAT as an org secret, ensure it is **allowed for repository `apithreshold-action-demo`** (org → Settings → Secrets and variables → Actions → repository access).

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

### 403 on “Checkout apithreshold backend” (`Write access to repository not granted`)

That message is GitHub’s generic HTTPS denial. Common causes:

1. **`BACKEND_GITHUB_TOKEN` is missing or not visible to this run**  
   - Not created under **apithreshold-action-demo → Settings → Secrets and variables → Actions**.  
   - **Fork PR:** upstream secrets are not passed to workflows from forks (see above).  
   - **Org secret:** not shared with this repository.

2. **Fallback `GITHUB_TOKEN` is being used** (secret empty) and **`apithreshold/backend` is private**  
   The job’s `GITHUB_TOKEN` is scoped to **`apithreshold-action-demo` only**. It cannot read another private org repo. You must supply a PAT with access to **`apithreshold/backend`**.

3. **PAT is wrong or too narrow**  
   - **Fine-grained:** resource owner includes the org; under **Repository access**, **`apithreshold/backend`** is selected; permission **Contents: Read** (and usually **Metadata: Read**).  
   - **SAML SSO:** open **Fine-grained token → Configure SSO → Authorize** for the **`apithreshold`** org.  
   - If fine-grained still misbehaves, try a **classic PAT** with **`repo`** scope limited to that repository (or org policy permitting).

4. **Wrong ref**  
   If **`BACKEND_REF`** does not exist on the remote you get a different error, but confirm the default branch of **`backend`** (e.g. `main`) matches workflow `env` **`BACKEND_REF`**.

**Private fork of backend:** set **`BACKEND_REPOSITORY`** to `your-org/your-fork` and use a PAT that can read that fork.

## Costs and runtime

`apithreshold gate` runs the full LLM pipeline (parse → assess → generate → score → evaluate). Expect **minutes** per run and **API spend**; use a tiny spec (this repo’s `openapi.yaml`) to keep both smaller.

## Next steps (product hardening)

- Pin a **released** version: `pip install apithreshold==0.x.y`.
- Add **`--project-id`** stability (already set to `${{ github.repository }}` in the workflow).
- When the CLI supports **`--ci`** and **`GITHUB_OUTPUT` gate state**, extend the workflow and document passing state between runs (see your Pre-Beta technical track).
