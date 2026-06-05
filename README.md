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
   | **`APITHRESHOLD_LICENSE`** | **Optional** — Ed25519 JWT for **Starter+** tiers. **Omit** on free tier (max **learning** / **warning** modes only). Required if you want **enforcing** / **strict** in CI. |

   **PAT for `BACKEND_GITHUB_TOKEN`:** GitHub → **Settings** (your account or a bot user) → **Developer settings** → **Personal access tokens**. Create a **fine-grained** token with **Contents: Read** on repository **`apithreshold/backend`** only (minimum scope). Paste the token as secret **`BACKEND_GITHUB_TOKEN`** on **`apithreshold-action-demo`**. If your org uses **SAML SSO**, authorize the token for the org.

   If **`apithreshold/backend` is public**, you can omit **`BACKEND_GITHUB_TOKEN`**: in **Settings → Secrets and variables → Actions → Variables**, create **`BACKEND_IS_PUBLIC`** with value **`true`** (exact string). The workflow then clones **`backend`** using **`GITHUB_TOKEN`** only. If you leave **`BACKEND_IS_PUBLIC`** unset, the workflow assumes a **private** backend and **requires** **`BACKEND_GITHUB_TOKEN`** (fail-fast with a clear error if it is missing).

   **Pull requests from forks** do not receive **this** repository’s Action secrets, so **`BACKEND_GITHUB_TOKEN` is empty** on those runs. For a **private** backend the workflow will **fail immediately** with an explanation (not a long git 403 retry). For a **public** backend, set **`BACKEND_IS_PUBLIC=true`** so fork PRs can still clone **`backend`** with **`GITHUB_TOKEN`**. Same-repo PRs receive secrets normally.

   **Organization secrets:** if you store the PAT as an org secret, ensure it is **allowed for repository `apithreshold-action-demo`** (org → Settings → Secrets and variables → Actions → repository access).

4. Open **Actions** and confirm the workflow run completes. **Push to `main`** uses the **progressive** policy (see below). **Same-repo PRs** use **warning** (non-blocking). **`workflow_dispatch`** defaults to **auto** (progressive) or explicit modes for scripted demos.

## Progressive gate in CI (enterprise pattern)

APIThreshold can **tighten the bar over time** instead of flipping straight to a hard **95% enforcing** gate on day one.

| Phase | Default threshold | Fails the job? | When you reach it |
|-------|-------------------|----------------|-------------------|
| **learning** | 50% | No | First run on this cache key |
| **warning** | 80% | No | After **14 days** on the timeline (`learning_days`) |
| **enforcing** | 95% | **Yes** | After **7** warning days with daily best score ≥ **85%** |

**How CI persists state (not Git):**

1. **`actions/cache`** restores `~/.apithreshold/state/<project-id>/gate_state.json` before `apithreshold gate`.
2. On **trusted** runs, **`--mode` is omitted** on `main` / branch push so the CLI calls **`get_current_mode()`** and can auto-advance.
3. At job end, **`save-always: true`** writes updated state back to the cache key  
   `apithreshold-gate-<repo>-<branch>`.
4. Artifact **`apithreshold-gate-state`** (90-day retention) gives auditors a per-run snapshot.

**Policy matrix (this workflow):**

| Trigger | Policy | Cache |
|---------|--------|-------|
| **Push `main` / `master`** | Progressive (auto-advance) | Yes, per branch |
| **Same-repo PR** | **Warning** (observe, exit 0) | Yes, per branch |
| **Fork PR** | **Warning** only | No (no shared cache write) |
| **`workflow_dispatch`** | User choice; **auto** = progressive | Yes when trusted |

**Presenter line:** *“We start in learning so teams aren’t blocked while we measure; warning surfaces gaps in PRs; enforcing on `main` only after stable scores — state lives in Actions cache, not in git history.”*

### Why does the CLI say BLOCKED but CI stays green?

APIThreshold uses **two different outcomes**:

| Layer | Learning / warning | Enforcing / strict |
|-------|--------------------|--------------------|
| **Quality** | Score vs threshold — can show **BLOCKED** if below bar | Same |
| **CI exit code** | Always **0** (observe only) | **1** if below bar → red job |

So **BLOCKED** means “quality did not meet this mode’s threshold,” not “GitHub Actions failed.” Only **enforcing** and **strict** connect a low score to a failed workflow. If you want a low score to fail CI immediately, use **`workflow_dispatch` → mode `enforcing`** (or wait until progressive advance reaches enforcing on `main`).

## Live demo (about 5 minutes)

Use this when you want a **clear narrative** for an audience: policy gate → optional deep remediation → artifacts (and a **sticky PR comment** on same-repo PRs).

1. **Actions** → **APIThreshold gate** → **Run workflow**.
2. **Progressive story:** choose **`openapi.yaml`** and mode **`auto`**, then **Run workflow**. Open **Summary** → **Progressive gate** table + **Progressive gate status** (stored mode, history count, last score). Re-run on **`main`** over time to show cache-backed mode advance (or explain the 14-day / 7-day rules from the Summary).
3. **Happy path (green):** **`demo/openapi-ci-happy-path.yaml`** + **`enforcing`** (or **`warning`**). **Without** **`APITHRESHOLD_LICENSE`**, the workflow auto-uses **warning** + demo **`threshold_warning: 15%`** (free tier blocks **enforcing**). **With** a Starter+ license secret, **enforcing** + **`threshold_enforcing: 15%`** is seeded.
4. **Failure path (red + recommendations):** **`demo/openapi-demo-fail.yaml`** + **`enforcing`** → **explain/report** + **apithreshold-gate-recommendations**.
5. **Summary** + job log steps + **Artifacts** (`apithreshold-gate-full-log`, **`apithreshold-gate-state`**, recommendations on failure).

**Pass path (CI/CD happy path):** **Run workflow** with **`demo/openapi-ci-happy-path.yaml`**. Pick **`enforcing`** if you have **`APITHRESHOLD_LICENSE`** in secrets; otherwise pick **`warning`** (or **`enforcing`** — the workflow will downgrade to **warning** on free tier). The job failed with **`LICENSE_ERROR`** when **enforcing** is requested without a license — that is not a score problem. Demo seeds a **15%** threshold (warning or enforcing) so the gate **meets bar** in logs while the real LLM score may be much lower than product **95%**.

**Fork pull requests** do not get the sticky PR comment (GitHub token cannot update upstream PR threads); use same-repo branches for that part of the demo.

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

On **failure** only, a follow-up step runs **`apithreshold explain`** (from the last gate result in `~/.apithreshold/state/...`) and **`apithreshold report`** when a quality report is present. That output is **`gate_recommendations.log`** in artifact **`apithreshold-gate-recommendations`** and is partially embedded in the job **Summary** under **LLM recommendations (after failure)**. The gate step itself stays short (Rich panels + DEBUG); the explain/report step is what adds prioritized fix suggestions.

**Ways to get *all* details**

1. **Expand the step** **Run APIThreshold gate** in the job log (full stream, as emitted).
2. **Artifacts** on the workflow run → download **apithreshold-gate-full-log**; if the gate failed, also **apithreshold-gate-recommendations** for explain/report text.
3. Run page **⋯** (top right) → **Download log archive** — entire job as files (GitHub-native).
4. **Summary** tab: on **failure**, the workflow embeds the **entire** `gate_full.log` in the Summary when it is under GitHub’s size limit (~950KB); if larger, the Summary includes a long tail and points you to the artifact for the complete file.

## When the APIThreshold gate fails

On failure, the **Summary** includes score headline, remediation checklist, verbatim `gate_full.log`, then (from a separate step) **LLM recommendations** when `OPENAI_API_KEY` is available and state could be read.

## Costs and runtime

`apithreshold gate` runs the full LLM pipeline (parse → assess → generate → score → evaluate). Expect **minutes** per run and **API spend**; use a tiny spec (this repo’s `openapi.yaml`) to keep both smaller.

## Next steps (product hardening)

- Pin a **released** version: `pip install apithreshold==0.x.y`.
- **`--project-id`** is already `${{ github.repository }}`; progressive cache is per **branch** (`github.ref_name`).
- For **compliance-grade** history beyond Actions cache, export **`gate_state.json`** to object storage or the APIThreshold portal when available.
- When the CLI supports **`--ci`** and **`GITHUB_OUTPUT`**, wire structured `GateResult` into required checks and branch protection.
