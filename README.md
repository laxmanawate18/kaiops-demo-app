# KaiOps Demo App

A tiny stateless Go HTTP API used to exercise the KaiOps deployment pipeline:
**GitHub CI → Artifact Registry → ArgoCD → GKE**, with health/sync status watched
by KaiOps (ArgoCD poller) which posts `[App] Healthy|Failed` status to Slack.

## Endpoints
- `GET /` -> `{"service":"kaiops-demo-app","status":"ok"}`
- `GET /healthz` -> `{"status":"healthy","app":"kaiops-demo-app"}`
- `GET /api/info` -> app/version/pod/host/timestamp

## Structure
```
main.go                  # Go HTTP API
Dockerfile               # multi-stage distroless build
k8s/deployment.yaml      # Namespace + Deployment + Service (GitOps source)
argocd-application.yaml  # ArgoCD Application (apply after repo exists)
.github/workflows/build-deploy.yml  # CI: build+push image, bump manifest, ArgoCD auto-syncs
```

## Deploy (one-time)
```bash
# 1. Create namespace + register the ArgoCD Application
kubectl apply -f k8s/deployment.yaml       # creates namespace kaiops-demo + app
kubectl apply -f argocd-application.yaml    # registers ArgoCD Application
```
- **I (the assistant) can run these** once you've pushed the repo, since I have GKE kubectl access.

## GitHub repository secrets (set once, in the repo Settings → Secrets → Actions)
| Secret | Value | Purpose |
|---|---|---|
| `WIP_PROVIDER` | `projects/275388304596/locations/global/workloadIdentityPools/kaiops-pool/providers/github-demo-app` | GCP Workload Identity (GitHub Actions → GCP). No key needed. |
| `SERVICE_ACCOUNT` | `kaiops-ci-sa@project-3da8cb5f-328e-44d3-b7a.iam.gserviceaccount.com` | The SA the CI impersonates. |
| `GH_PAT` | a GitHub PAT with `repo` scope | So CI can push the updated image tag back to the repo. |

> The WIF pool `kaiops-pool` + provider `github-demo-app` + SA `kaiops-ci-sa` (with
> roles `artifactregistry.writer`, `container.developer`, `iam.serviceAccountUser`,
> `run.invoker`) are already created in the project. The SA IAM binding grants
> `principalSet://.../attribute.repository_owner/laxmanawate18` permission to
> impersonate it. (Org policy disables SA-key creation, so we use WIF — no key.)

## How the pipeline works
1. `git push` to `main` triggers `.github/workflows/build-deploy.yml`.
2. CI builds the Docker image, pushes to Artifact Registry, and commits the new
   image tag back into `k8s/deployment.yaml`.
3. ArgoCD watches this repo and auto-syncs the new image to the `kaiops-demo`
   namespace on `gcp-demo-cluster` (GKE).
4. KaiOps polls ArgoCD `Application` status; on `Healthy` it replies "deploy OK"
   in the app's Slack thread, on `Failed/Degraded/OutOfSync` it triggers an RCA
   (tagging SRE if infra-related) and surfaces the needed HITL action.

## Notes
- Requires a GKE cluster (`gcp-demo-cluster` @ us-central1-a) and ArgoCD installed.
- The Artifact Registry repo `kaiops-demo-app` must exist in
  `us-central1-docker.pkg.dev/project-3da8cb5f-328e-44d3-b7a/`.
