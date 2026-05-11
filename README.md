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

   If **`apithreshold/backend` is public**, you can omit **`BACKEND_GITHUB_TOKEN`**: in **Settings → Secrets and variables → Actions → Variables**, create **`BACKEND_IS_PUBLIC`** with value **`true`** (exact string). The workflow then clones **`backend`** using **`GITHUB_TOKEN`** only. If you leave **`BACKEND_IS_PUBLIC`** unset, the workflow assumes a **private** backend and **requires** **`BACKEND_GITHUB_TOKEN`** (fail-fast with a clear error if it is missing).

   **Pull requests from forks** do not receive **this** repository’s Action secrets, so **`BACKEND_GITHUB_TOKEN` is empty** on those runs. For a **private** backend the workflow will **fail immediately** with an explanation (not a long git 403 retry). For a **public** backend, set **`BACKEND_IS_PUBLIC=true`** so fork PRs can still clone **`backend`** with **`GITHUB_TOKEN`**. Same-repo PRs receive secrets normally.

   **Organization secrets:** if you store the PAT as an org secret, ensure it is **allowed for repository `apithreshold-action-demo`** (org → Settings → Secrets and variables → Actions → repository access).

4. Open **Actions** and confirm the workflow run completes. The workflow uses **`enforcing`** for **all** triggers (`push`, `pull_request`, `workflow_dispatch`): if the score is below the threshold, the job **fails**. To run in non-blocking **learning** mode temporarily, set **`APITHRESHOLD_MODE: learning`** in `.github/workflows/apithreshold-gate.yml` `env` (in **learning**, the CLI may still print *Gate Blocked* while exiting **0**).

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

### REST probe returns **404 Not Found** for `GET /repos/apithreshold/backend`

With a **PAT**, GitHub often returns **404** (not 403) when the token **cannot read** a **private** repository — it hides that the repo exists. So **404 = “this identity has no access to that slug”**, not “the URL is wrong” (though a wrong slug also 404s).

Fix the **PAT → repository** binding: fine-grained token must **explicitly list** `apithreshold/backend`, **Contents: Read** + **Metadata: Read**, and **SSO authorize** the token for the `apithreshold` org if SAML is on. Confirm the Actions secret **`BACKEND_GITHUB_TOKEN`** is exactly that token (org secret must be **allowed for `apithreshold-action-demo`**).

The workflow prints an **anonymous** `GET /repos/...` first: **200** = repo metadata is public; **404** = private or wrong path (anonymous never sees private).

### 403 when cloning `apithreshold/backend` (`Write access to repository not granted`)

That message is GitHub’s generic HTTPS denial. The workflow **fails fast** if a private backend is assumed and **`BACKEND_GITHUB_TOKEN`** is missing. Otherwise it runs a **REST probe** (`GET /repos/apithreshold/backend`) with the same token as `git clone`; if that fails, the problem is **token access**, not the clone tool.

Common causes:

1. **`BACKEND_GITHUB_TOKEN` is missing or not visible to this run**  
   - Not created under **apithreshold-action-demo → Settings → Secrets and variables → Actions**.  
   - **Fork PR:** upstream secrets are not passed to workflows from forks (see above); use **`BACKEND_IS_PUBLIC=true`** if **`backend`** is public, or run from a same-repo branch.  
   - **Org secret:** not shared with this repository.

2. **`GITHUB_TOKEN` is being used** (e.g. **`BACKEND_IS_PUBLIC=true`**) but **`apithreshold/backend` is actually private**  
   Clear **`BACKEND_IS_PUBLIC`**, add **`BACKEND_GITHUB_TOKEN`**, and use a PAT with access to **`backend`**.

3. **PAT is wrong or too narrow**  
   - **Fine-grained:** resource owner includes the org; under **Repository access**, **`apithreshold/backend`** is selected; permission **Contents: Read** (and usually **Metadata: Read**).  
   - **SAML SSO:** open **Fine-grained token → Configure SSO → Authorize** for the **`apithreshold`** org.  
   - If fine-grained still misbehaves, try a **classic PAT** with **`repo`** scope limited to that repository (or org policy permitting).

4. **Trailing newline in the secret** (common when pasting into GitHub)  
   The workflow trims CR/LF from the token before `git clone`. If you still use an older workflow without that trim, re-save the secret or upgrade to the latest workflow on `main`.

5. **Wrong ref**  
   If **`BACKEND_REF`** does not exist on the remote you get a different error, but confirm the default branch of **`backend`** (e.g. `main`) matches workflow `env` **`BACKEND_REF`**.

**Private fork of backend:** set **`BACKEND_REPOSITORY`** to `your-org/your-fork` and use a PAT that can read that fork.

## Capturing the full gate output (workflow logs)

Every run uploads **`apithreshold-gate-full-log`** (`gate_full.log`) — the **verbatim** stdout/stderr from `apithreshold gate` (same bytes as the **Run APIThreshold gate** step in the Actions UI). Retention: **30 days**. Upload uses `continue-on-error: true` so **fork PRs** with a read-only token do not fail the job if artifact upload is blocked.

**Ways to get *all* details**

1. **Expand the step** **Run APIThreshold gate** in the job log (full stream, as emitted).
2. **Artifacts** on the workflow run → download **apithreshold-gate-full-log**.
3. Run page **⋯** (top right) → **Download log archive** — entire job as files (GitHub-native).
4. **Summary** tab: on **failure**, the workflow embeds the **entire** `gate_full.log` in the Summary when it is under GitHub’s size limit (~950KB); if larger, the Summary includes a long tail and points you to the artifact for the complete file.

## When the APIThreshold gate fails

On failure, the **Summary** also includes score headline, remediation checklist, then the verbatim log section above.

## Costs and runtime

`apithreshold gate` runs the full LLM pipeline (parse → assess → generate → score → evaluate). Expect **minutes** per run and **API spend**; use a tiny spec (this repo’s `openapi.yaml`) to keep both smaller.

## Next steps (product hardening)

- Pin a **released** version: `pip install apithreshold==0.x.y`.
- Add **`--project-id`** stability (already set to `${{ github.repository }}` in the workflow).
- When the CLI supports **`--ci`** and **`GITHUB_OUTPUT` gate state**, extend the workflow and document passing state between runs (see your Pre-Beta technical track).
