# app-bravo

A **generic starter app** for the platform's preprod environment (team `bravo`). It is the reference shape
a team copies to stand up a new app: a minimal containerized HTTP service plus the policy-compliant
Kubernetes manifests and the thin CI that builds, signs, and ships it.

## What's here

| Path | Purpose |
|------|---------|
| `cmd/server/main.go`, `go.mod` | Minimal stdlib Go HTTP server: `/healthz` (probe) + `/` (JSON). No cloud deps. |
| `Dockerfile` | Multi-stage build → distroless `nonroot` (uid 65532). The Dockerfile is the only language-specific surface. |
| `k8s/base/` + `k8s/overlays/<stage>/` | **The Kubernetes manifests (ADR-067).** Namespace-/host-agnostic `base/` + thin per-stage overlays (`dev`/`test`/`uat`/`staging`/`prod`). The per-Product ApplicationSet syncs `k8s/overlays/<stage>` and injects the namespace + host; `deploy.yml` pins the dev overlay's image digest (promotion to other stages is by PR). |
| `.github/workflows/deploy.yml` / `preview.yml` | **Thin callers** of the shared supply-chain workflows in `asanexample/trusted-ci`. |
| `.github/workflows/validate.yml` / `security.yml` | Shift-left Kyverno gate (shared composite actions) + Trivy/Semgrep SAST. |

## How the supply chain works

`deploy.yml` is ~3 small jobs that call shared, app-team-unwritable reusable workflows:

1. **build** → `trusted-ci/build-sign.yml` — builds the image from the `Dockerfile`, pushes it to the
   product-scoped repo `team-bravo/demo-web` in the platform ECR, cosign-keyless-signs it, and attaches a
   CycloneDX SBOM.
2. **provenance** → `trusted-ci/slsa-provenance.yml` — attaches the SLSA build provenance (isolated
   signer, SLSA Build L3).
3. **deploy** — pins the freshly signed digest into `k8s/overlays/dev/kustomization.yaml` (`images[].digest`)
   and commits it; the per-Product ApplicationSet syncs it to the cluster. Promotion to test/uat/staging/prod
   is by PR (promote-by-PR).

Image signatures, SBOM, and provenance carry this repo's identity (via the `githubWorkflowRepository`
certificate extension), which the platform's Kyverno `verify-images` / `verify-attestations` policies
require at admission. There is nothing per-app to maintain in the signing logic — it lives in `trusted-ci`.

## Starting a new app from this template

1. Copy the repo; rename `app-bravo` → `app-<yourapp>` and `team-bravo`/`demo-web` →
   `team-<yourteam>`/`<product>-<service>` throughout (`k8s/base/`, `k8s/overlays/`, labels, SA name, the
   `app:` input in the workflows).
2. **Do not hardcode a hostname or namespace** — the platform injects both (the per-Product ApplicationSet
   sets the destination namespace and patches the real host onto the `HTTPRoute`). Leave the `placeholder.invalid`
   host and the namespace-agnostic `base/`.
3. Replace `cmd/`/`Dockerfile` with your actual app — keep `/healthz` and listen on `:8080`, or update
   the probes/port in `base/deployment.yaml` to match.
4. The platform onboards the team + product via the git-native `Product` registry entry
   (`gitops/products/<team>/<product>.yaml`) and the developer authors an `Environment` per stage
   (`gitops/environments/<team>/<product>/<stage>.yaml`) — which provisions the ECR repo, Pod-Identity, and
   policies (ADR-067).

See `docs/runbooks/app-supply-chain-onboarding.md` in the platform repo for the full onboarding flow.
