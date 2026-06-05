# License files for CI (Starter+ / enforcing)

| File | In git? | Purpose |
|------|---------|---------|
| `dev-public.pem` | **Yes** | Ed25519 **public** key — verifies JWT signatures (`kid: dev`) |
| `*.jwt` / token files | **No** | Never commit license JWTs |

The workflow sets:

```text
APITHRESHOLD_LICENSE_PUBLIC_KEY_FILE=${{ github.workspace }}/license/dev-public.pem
APITHRESHOLD_LICENSE_KID=dev
```

`APITHRESHOLD_LICENSE` (the JWT string) lives only in **GitHub Actions secrets**.

## One-time setup (Option B)

### 1. Issue a Starter JWT (ops / engineering machine)

From the private **`license-issuer`** repo (not committed here):

```bash
cd /path/to/license-issuer
source .venv/bin/activate
apithreshold-license-issue \
  --key-file keys/dev-local.pem \
  --tier starter \
  --days 365 \
  --customer-id apithreshold-action-demo
```

Copy the printed `export APITHRESHOLD_LICENSE="..."` value (JWT only).

**Important:** `dev-public.pem` in this folder must be the public half of the **same** key pair that signed the JWT. The committed file matches `license-issuer/keys/dev-local-public.pem`. If you rotate `dev-local.pem`, refresh both PEMs:

```bash
apithreshold-license-issue --key-file keys/dev-local.pem --tier starter --days 1 \
  --write-public-key /path/to/apithreshold-action-demo/license/dev-public.pem
```

### 2. Add GitHub secret

**apithreshold-action-demo → Settings → Secrets and variables → Actions → New repository secret**

| Name | Value |
|------|--------|
| `APITHRESHOLD_LICENSE` | Full JWT string (one line, no quotes) |

### 3. Run workflow

**workflow_dispatch** → `demo/openapi-ci-happy-path.yaml` + **`enforcing`**.

With the secret set, the workflow keeps **enforcing** and seeds demo `threshold_enforcing: 15%`.

## Sales / `kid: demo` alternative

For prospect demos use `demo-sales-public.pem` from `license-issuer` and `APITHRESHOLD_LICENSE_KID=demo` instead — update workflow `env` if you switch kits.

See [backend/docs/licensing.md](https://github.com/apithreshold/backend/blob/main/docs/licensing.md).
