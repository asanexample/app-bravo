# app-bravo

A **generic starter app** for the platform's preprod environment (team `bravo`). It is the reference shape
a team copies to stand up a new app: a minimal containerized HTTP service plus the policy-compliant
Kubernetes manifests and the thin CI that builds, signs, and ships it.

## What's here

| Path | Purpose |
|------|---------|
| `cmd/server/main.go`, `go.mod` | Minimal stdlib Go HTTP server: `/healthz` (probe) + `/` (JSON). No cloud deps. |
| `Dockerfile` | Multi-stage build → distroless `nonroot` (uid 65532). The Dockerfile is the only language-specific surface. |
| `k8s/preprod/` | Kustomize app manifests (Deployment/Service/HTTPRoute/ServiceAccount), policy-compliant for the `team-bravo` namespace. |
| `.github/workflows/deploy.yml` / `preview.yml` | **Thin callers** of the shared supply-chain workflows in `asanexample/trusted-ci`. |
| `.github/workflows/validate.yml` / `security.yml` | Shift-left Kyverno gate (shared composite actions) + Trivy/Semgrep SAST. |

## How the supply chain works

`deploy.yml` is ~3 small jobs that call shared, app-team-unwritable reusable workflows:

1. **build** → `trusted-ci/build-sign.yml` — builds the image from the `Dockerfile`, pushes it to
   `team-bravo/demo` in the platform ECR, cosign-keyless-signs it, and attaches a CycloneDX SBOM.
2. **provenance** → `trusted-ci/slsa-provenance.yml` — attaches the SLSA build provenance (isolated
   signer, SLSA Build L3).
3. **deploy** — pins the freshly signed digest into `k8s/preprod/deployment.yaml` and commits it; ArgoCD
   syncs that to the cluster.

Image signatures, SBOM, and provenance carry this repo's identity (via the `githubWorkflowRepository`
certificate extension), which the platform's Kyverno `verify-images` / `verify-attestations` policies
require at admission. There is nothing per-app to maintain in the signing logic — it lives in `trusted-ci`.

## Starting a new app from this template

1. Copy the repo; rename `app-bravo` → `app-<yourapp>` and `team-bravo` → `team-<yourteam>` throughout
   (`k8s/preprod/`, labels, SA name, the `app:` input in the workflows).
2. Set your team's allow-listed hostname in `k8s/preprod/httproute.yaml` (must match your Tenant claim).
3. Replace `cmd/`/`Dockerfile` with your actual app — keep `/healthz` and listen on `:8080`, or update
   the probes/port in `deployment.yaml` to match.
4. The platform onboards the team (ECR repo, push role, policies) via the Tenant claim + `teams.hcl`.

See `docs/runbooks/app-supply-chain-onboarding.md` in the platform repo for the full onboarding flow.
