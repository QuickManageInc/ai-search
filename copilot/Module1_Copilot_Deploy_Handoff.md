# Copilot deploy handoff — dev (platform-gitops + AWS)

**Audience:** Supervisor / platform owner with AWS + ArgoCD access  
**Developer:** Does not have prod/deploy access — please execute the steps below.

---

## Message to supervisor (copy/paste)

> **Subject:** Please deploy QuickManage Copilot (`ai-edge-api`) to **dev** — AWS secrets + GitOps merge
>
> Hi — Copilot is ready for **dev** deployment. I’ve updated `platform-gitops` (dev overlay + ALB SSE timeout). I don’t have AWS/Argo access, so I need you to add the secrets below and merge/sync.
>
> ### 1. AWS Secrets Manager (ca-central-1)
>
> Create these **three** secrets (JSON `SecretString`). **Do not commit values to git.**
>
> | Secret name | JSON body (keys required) |
> |-------------|---------------------------|
> | `quickmanage/dev/ai-edge-api/mongodb` | `{ "MONGODB_URI": "<Atlas URI for aiDB>" }` |
> | `quickmanage/dev/ai-edge-api/redis` | `{ "REDIS_URL": "<redis://...>" }` |
> | `quickmanage/dev/ai-edge-api/google` | `{ "GOOGLE_GENERATIVE_AI_API_KEY": "<Gemini API key>" }` |
>
> **MongoDB notes:** Use the shared Atlas cluster if appropriate, but the app uses database **`aiDB`** (`AI_DB_NAME`). Ensure the DB user can read/write `aiDB` (sessions + copilot metadata).
>
> **Redis notes:** Same ElastiCache/cluster as other dev edge services is fine if network policy allows the `ai-edge-api` pod to reach it.
>
> **Google:** Gemini API key with access to `gemini-2.5-flash-lite` and `gemini-2.0-flash` (fallback).
>
> OTEL is already wired to existing secret `quickmanage/dev/shared/otel` — no new secret unless that one is missing.
>
> ### 2. IAM (IRSA)
>
> Role: `arn:aws:iam::000572870562:role/quickmanage-dev-eks-irsa-ai`  
> ServiceAccount: `microservice-ai` (namespace `quickmanage`)
>
> Ensure the role policy allows:
>
> ```json
> {
>   "Effect": "Allow",
>   "Action": ["secretsmanager:GetSecretValue"],
>   "Resource": [
>     "arn:aws:secretsmanager:ca-central-1:000572870562:secret:quickmanage/dev/ai-edge-api/*",
>     "arn:aws:secretsmanager:ca-central-1:000572870562:secret:quickmanage/dev/shared/otel*"
>   ]
> }
> ```
>
> ### 3. Container image (ECR)
>
> Repo: `000572870562.dkr.ecr.ca-central-1.amazonaws.com/quickmanage/ai-edge-api:latest`
>
> Image is built on push to **`ai-edge-api` `main`** (GitHub Actions `.github/workflows/ecr-build-push.yml`). Please confirm **`latest`** was pushed after the Copilot changes merge to `main`.
>
> ### 4. GitOps merge + ArgoCD sync
>
> Merge PR for **`platform-gitops`** branch with:
>
> - `apps/overlays/dev/services/ai-edge-api/kustomization.yaml` — Copilot env (`AI_TOOL_FILTER=intent`, models, CORS, secret **paths** only)
> - `apps/overlays/dev/ingress/public-ingress.yaml` — ALB `idle_timeout=120` for SSE
>
> ArgoCD ApplicationSet `quickmanage-dev-services` should auto-sync `apps/overlays/dev/services/ai-edge-api`.
>
> Ingress route already exists: `https://api.quickmanage-developer.ca/api/v1/ai/*` → `ai-edge-api-srv:80`
>
> **Dependency:** `analytics-edge-api` must be healthy (Copilot tools call in-cluster `http://analytics-edge-api-srv/api/v1`).
>
> ### 5. Post-deploy smoke (from a machine with dev API access)
>
> ```bash
> cd ai-edge-api
> API_BASE=https://api.quickmanage-developer.ca \
> STORE_ID=67a27db075551b7c50e7a54e \
> AUTH_TOKEN="<merchant JWT>" \
> npm run prod:smoke
>
> # Full 12-ask routing matrix (optional)
> API_BASE=https://api.quickmanage-developer.ca \
> STORE_ID=67a27db075551b7c50e7a54e \
> AUTH_TOKEN="<merchant JWT>" \
> npm run smoke:launch
> ```
>
> Health (no auth): `curl -s https://api.quickmanage-developer.ca/api/v1/ai/health/live` — may 404 if only subpaths are routed; use pod logs or internal probe paths `/health/live` on the deployment.
>
> ### 6. Merchant portal (separate from GitOps)
>
> Portal build already has `VITE_ENABLE_AI_ASSISTANT=true` in `quickmanage-merchant-portal/.env.production`. Deploy portal so merchants see the FAB on dev. API base should remain `https://api.quickmanage-developer.ca/api/v1`.
>
> Thanks!

---

## What changed in GitOps (this PR)

| File | Change |
|------|--------|
| `apps/overlays/dev/services/ai-edge-api/kustomization.yaml` | IRSA, resource limits (512Mi), Copilot env vars, dev secret paths, CORS |
| `apps/overlays/dev/ingress/public-ingress.yaml` | ALB idle timeout 120s for SSE streams |

Base manifests (`apps/base/ai-edge-api/`) were already present; dev overlay was partially configured — now aligned with Copilot launch settings.

---

## Non-secret env (already in GitOps — for reference)

| Variable | Dev value |
|----------|-----------|
| `AI_TOOL_FILTER` | `intent` |
| `AI_TOOL_FILTER_MAX` | `15` |
| `AI_MODEL` | `gemini-2.5-flash-lite` |
| `AI_MODEL_FALLBACK` | `gemini-2.0-flash` |
| `ANALYTICS_EDGE_API_URL` | `http://analytics-edge-api-srv/api/v1` |
| `CORS_ORIGIN` | `https://www.merchants.quickmanage-developer.ca` |
| `AI_BURST_LIMIT_PER_MINUTE` | `8` |
| `AI_DAILY_LIMIT_PER_STORE` | `0` (unlimited) |

---

## Prod (later — not in this handoff)

When promoting to prod:

1. Create `quickmanage/prod/ai-edge-api/{mongodb,redis,google}` secrets (same JSON shape).
2. IRSA role for prod EKS + `apps/overlays/prod` patches (prod ingress currently lacks `/api/v1/ai`).
3. Update `CORS_ORIGIN` to production merchant domain.
4. Run `smoke:launch` against prod API.

See `ai-edge-api/DEPLOY.md`.

---

## Related docs

- Launch plan: `Module1_Copilot_Prod_Launch_Plan.md`
- Deploy notes: `ai-edge-api/DEPLOY.md`
- Copilot index: `README.md`
