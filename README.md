# MultiK8s Beginner Learning Path

This project is a full-stack, containerized app deployed to Kubernetes (GKE) with GitHub Actions CI/CD.  
It is a good beginner project because it covers the full path from local development to production-style deployment.

If you are new, follow this in order and do not skip steps.

## 1. What You Are Building

You are running a multi-service app:

- `client` (React UI)
- `server` (Node/Express API)
- `worker` (Node worker process)
- `redis` (cache/queue)
- `postgres` (database)
- `ingress` (entry point to route traffic)

Infrastructure in this repo:

- Docker for local containerized development
- Kubernetes manifests in `k8s/`
- GitHub Actions deployment workflow in `.github/workflows/deployment.yaml`

You can think of this project as 3 layers: app code, containers, and Kubernetes orchestration.

## 2. Prerequisites

Install:

- Git
- Docker Desktop
- Node.js 18+ and npm
- `kubectl`
- Google Cloud CLI (`gcloud`)

Accounts and access:

- Docker Hub account (for image push)
- Google Cloud project + GKE cluster
- GitHub repository with Actions enabled

Without these accounts/secrets, CI/CD can run tests but will fail when pushing images or deploying.

### 2.1 Create the GKE Cluster First (Google Cloud)

Before using the GitHub Actions deploy workflow, create the Kubernetes cluster in Google Cloud.

Minimum console steps:

1. Open Google Cloud Console and select your project (example: `multi0k8s-487915`).
2. Enable the `Kubernetes Engine API` if prompted.
3. Go to `Kubernetes Engine` -> `Clusters`.
4. Click `Create`.
5. Choose a Standard cluster (recommended for learning this project).
6. Set cluster name to match your workflow (example: `multi-k8s`).
7. Set region to match your workflow (example: `europe-west2`).
8. Choose a small node pool to start (you can scale later).
9. Create the cluster and wait until status is `Running`.

Example `gcloud` command (regional cluster, matches this repo setup):

```bash
gcloud container clusters create multi-k8s \
  --region europe-west2 \
  --project multi0k8s-487915 \
  --num-nodes 1
```

After creation, verify:

```bash
gcloud container clusters list --project multi0k8s-487915
```

Important: The values must match `.github/workflows/deployment.yaml`:

- `PROJECT_ID`
- `CLUSTER_NAME`
- `CLUSTER_LOCATION`

## 3. Project Structure

- `client/` React app
- `server/` API service
- `worker/` background worker
- `k8s/` Kubernetes manifests
- `.github/workflows/deployment.yaml` CI/CD pipeline
- `docker-compose-dev.yml` local dev stack

## 4. Beginner Path (Recommended Order)

1. Run the app locally with Docker Compose.
2. Understand each service and API route.
3. Run tests locally.
4. Build and push Docker images.
5. Deploy to GKE using GitHub Actions.
6. Verify workload health in Google Cloud.
7. Learn rollout and scaling basics.

## 5. Run Locally (Fastest Start)

From repo root:

```bash
docker compose -f docker-compose-dev.yml up --build
```

Open:

- `http://localhost:3050`

Stop:

```bash
docker compose -f docker-compose-dev.yml down
```

## 6. Run Client Tests

Local test command:

```bash
docker build -t react-test -f ./client/Dockerfile.dev ./client
docker run -e CI=true react-test npm test
```

This mirrors your CI test step.

## 7. Kubernetes Manifests

Main files:

- Deployments: `k8s/*-deployment.yaml`
- Services: `k8s/*-cluster-ip-service.yaml`
- Ingress: `k8s/ingress-service.yaml`
- PVC: `k8s/database-persistent-volume-claim.yaml`

Current rollout strategy is capacity-friendly (`maxSurge: 0`, `maxUnavailable: 1`) for small clusters, which reduces deploy failures when resources are limited.

## 8. GitHub Actions Deployment Flow

Workflow file: `.github/workflows/deployment.yaml`

On push to `main`, it does:

1. Checkout code
2. Run client tests
3. Authenticate to Google Cloud (OIDC)
4. Get GKE credentials
5. Create/update Kubernetes secret `pgpassword`
6. Ensure `ingress-nginx` exists
7. Build and push images
8. Apply Kubernetes manifests and roll out deployments

This gives you one repeatable deployment flow every time code changes are merged into `main`.

### 8.1 `deployment.yaml` Explained (Block by Block)

File: `.github/workflows/deployment.yaml`

1. `name: Deploy MultiK8s`  
This is the workflow display name shown in the GitHub Actions UI.

2. `on: push: branches: [main]`  
The pipeline runs automatically every time new commits are pushed to `main`.

3. Top-level `env` block  
`PROJECT_ID`, `CLUSTER_NAME`, and `CLUSTER_LOCATION` are shared variables used by multiple steps so values are centralized.

4. `jobs.build.runs-on: ubuntu-latest`  
GitHub starts a Linux runner VM where all commands execute.

5. `permissions` block  
`id-token: write` is required for OIDC login to Google Cloud.  
`contents: read` allows checkout of repository code.

6. `Checkout` step  
Uses `actions/checkout@v4` to pull your repository into the runner workspace.

7. `Set SHA` step  
Stores the current commit SHA into environment variable `SHA` via `$GITHUB_ENV`, used to tag Docker images uniquely.

8. `Run Tests` step  
Builds test container from `client/Dockerfile.dev` and runs React tests with `CI=true`.  
If tests fail, deployment stops here.

9. `Authenticate to Google Cloud` step  
Uses `google-github-actions/auth@v2` with Workload Identity Federation.  
This avoids using a long-lived JSON key and is the recommended secure approach.

10. `Setup gcloud` and `Configure Docker` steps  
Initializes `gcloud` with your project and configures Docker auth so pushes/pulls to registries can work.

11. `Get GKE Credentials` step  
Fetches Kubernetes cluster credentials and writes kubeconfig so later `kubectl` commands target `multi-k8s` in `europe-west2`.

12. `Create/Update Kubernetes Secrets` step  
Creates or updates secret `pgpassword` from GitHub secret `PGPASSWORD`.  
`--dry-run=client -o yaml | kubectl apply -f -` makes this idempotent (safe on every deploy).

13. `Ensure NGINX Ingress Controller` step  
Checks if `ingressclass nginx` exists.  
If missing, installs ingress-nginx controller and waits until controller deployment is available.

14. `Wait for NGINX Admission Webhook` step  
On a brand-new cluster, the ingress controller can be "running" before its admission webhook is fully ready.  
This step waits for the `ingress-nginx-controller-admission` service to have endpoints before applying your ingress manifest.

15. `Build Images` step  
Builds `client`, `server`, and `worker` images with two tags:
- `latest`
- commit-specific tag `${{ env.SHA }}`

16. `Push Images` step  
Pushes both tags to Docker Hub for all three images.

17. `Deploy to Kubernetes` step  
Applies all manifests in `k8s/`, then updates deployments to use the SHA-tagged images.  
Finally waits for rollout completion for `postgres`, `server`, `client`, and `worker`.

18. `Print App URL When Ready` step  
After rollout, the workflow polls `ingress-service` until an external IP or hostname is available, then prints the app URL in logs.  
It waits up to 10 minutes and fails with a clear message if ingress is still not provisioned.

19. Rollback behavior  
This workflow does not auto-rollback. If rollout fails, GitHub Actions shows the failing step and current cluster state so you can inspect and fix.

### 8.2 How Image Updates Work (`latest` vs `SHA`)

A common confusion is: "If Kubernetes already uses `latest`, how does it know something changed?"

In this project, updates are triggered by the SHA tag, not by `latest` alone:

1. Build step creates two tags for each image:
- `latest`
- `${SHA}` (current commit hash)

2. Push step pushes both tags to Docker Hub.

3. Deploy step runs `kubectl set image ...:${SHA}` for `client`, `server`, and `worker`.

That `kubectl set image` command changes the Deployment spec to a new tag every deploy, so Kubernetes starts a rolling update.

If you only used `latest` and never changed the image tag in the Deployment spec, Kubernetes could treat it as unchanged and skip rollout.

## 9. Branch Workflow (Better CI/CD Practice)

To follow better CI/CD standards, do your changes in a branch first, test them, and only then merge into `main`.

This repo is now set up for that flow:

- `.github/workflows/ci.yaml` runs tests on non-`main` branch pushes and on pull requests to `main`
- `.github/workflows/deployment.yaml` deploys only when code is pushed to `main`

### Recommended flow (`ci-test` example)

1. Create or switch to your working branch:

```bash
git checkout ci-test
```

2. Make code changes and commit them.

3. Push the branch:

```bash
git push -u origin ci-test
```

4. GitHub Actions runs the `CI` workflow (tests only).

5. Open a Pull Request from `ci-test` to `main`.

6. GitHub Actions runs `CI` again for the PR.

7. If tests pass, merge the PR manually into `main`.

8. The `Deploy MultiK8s` workflow runs automatically and deploys the new `main`.

### Why this is better than pushing directly to `main`

- You catch test failures before deployment
- You can review changes in a PR
- `main` stays more stable
- Deployment happens only after an intentional merge

## 10. Required GitHub Secrets

Add these in GitHub:

- `DOCKER_USERNAME`
- `DOCKER_PASSWORD`
- `PGPASSWORD`

Without `PGPASSWORD`, `server` and `postgres` pods fail with `CreateContainerConfigError` because required environment variables cannot be resolved.

## 11. Verify Deployment in Google Cloud Console

Use Google Cloud Console:

1. `Kubernetes Engine` -> `Workloads`
2. Check `client-deployment`, `server-deployment`, `worker-deployment`, `postgres-deployment`, `redis-deployment`
3. Confirm pods are `Running`
4. Open each workload -> check `Events` for errors

Ingress check:

1. `Kubernetes Engine` -> `Services & Ingress`
2. Find `ingress-service`
3. Wait for external IP in `Address`

Important: IPs under cluster "Control plane endpoint" are not your app URL.  
Only the ingress external address is used to access your application in a browser.

## 12. CLI Verification Commands

```bash
kubectl get deployments
kubectl get pods
kubectl get services
kubectl get ingress
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

Rollout status:

```bash
kubectl rollout status deployment/client-deployment
kubectl rollout status deployment/server-deployment
kubectl rollout status deployment/worker-deployment
```

## 13. Common Problems and Fixes

### `CreateContainerConfigError`

Usually missing secret `pgpassword`.

Fix:

- Ensure GitHub secret `PGPASSWORD` exists
- Ensure workflow step creates Kubernetes secret

### `Cannot schedule pods: No preemption victims found`

Cluster has insufficient capacity.

Fix options:

- Increase node pool size
- Reduce replicas
- Use low-surge rollout strategy (already set)

### Ingress has no external IP

Ingress controller missing or still provisioning.

Fix:

- Ensure `ingress-nginx` controller is installed/running
- Wait for cloud load balancer provisioning

### Fresh cluster ingress webhook error (`no endpoints available for service "ingress-nginx-controller-admission"`)

This usually happens right after recreating a cluster. The ingress controller was installed, but the admission webhook service was not fully ready when `k8s/ingress-service.yaml` was applied.

Fix:

- Re-run the workflow once (often works)
- Use a workflow wait step for admission webhook endpoints (already added in this repo)

## 14. Scaling Safely

To scale up without rollout failures:

1. Increase cluster capacity first
2. Add CPU/memory requests/limits
3. Increase `replicas` gradually
4. Keep controlled rollout strategy

## 15. Suggested Next Learning Milestones

1. Add readiness/liveness probes to all deployments.
2. Add resource requests/limits for client/server/worker.
3. Add Horizontal Pod Autoscaler (HPA).
4. Add staging namespace and separate workflow environment.
5. Add monitoring (Cloud Logging + metrics dashboards).

## 16. Quick Reference

Local dev:

```bash
docker compose -f docker-compose-dev.yml up --build
```

Deploy trigger:

- Push to `main`

Main deployment config:

- `.github/workflows/deployment.yaml`

Kubernetes manifests:

- `k8s/`

## 17. Diagrams

### Architecture

```text
                         ┌────────────────────────────┐
                         │        User Browser        │
                         │      (Web Frontend)        │
                         └─────────────┬──────────────┘
                                       │ HTTP
                                       ▼
                         ┌────────────────────────────┐
                         │       NGINX Ingress        │
                         │      ingress-service       │
                         └───────┬──────────┬─────────┘
                                 │          │
                    route "/"    │          │ route "/api"
                                 ▼          ▼
                    ┌──────────────────┐  ┌──────────────────┐
                    │ Client Deployment│  │ Server Deployment│
                    │ (React service)  │  │ (Node API)       │
                    └─────────┬────────┘  └───────┬──────────┘
                              │                   │
                              │ optional calls    │ reads/writes
                              ▼                   ▼
                    ┌──────────────────┐  ┌──────────────────┐
                    │ Worker Deployment│  │ Postgres + Redis │
                    │ (Background Job) │  │ (Data layer)     │
                    └─────────┬────────┘  └──────────────────┘
                              │
                              └────────────► uses Redis queue
```

### Request Flow

```text
Browser            Ingress             Client              Server         Redis/Postgres
   │                  │                   │                  │                   │
   ├── GET / ───────► │                   │                  │                   │
   │                  ├── route "/" ───►  │                  │                   │
   │                  │                   ├── app files ───► │                   │
   │ ◄── HTML/JS/CSS ─┴───────────────────┘                  │                   │
   │
   ├── GET /api/values/current ────────► │                  │                   │
   │                  ├── route "/api" ───────────────────►  │                   │
   │                  │                                      ├── read cache ───► │
   │                  │                                      ├── read db ──────► │
   │ ◄── JSON response ┴─────────────────────────────────────┴───────────────────┘
```

### CI/CD Pipeline

```text
                         ┌─────────────────────────┐
                         │        Developer        │
                         │     git push main       │
                         └─────────────┬───────────┘
                                       │
                                       ▼
                         ┌─────────────────────────┐
                         │         GitHub          │
                         │   (Source + Actions)    │
                         └─────────────┬───────────┘
                                       │ Trigger
                                       ▼
                         ┌─────────────────────────┐
                         │  GitHub Actions Runner  │
                         │  - Run tests            │
                         │  - Auth to GCP (OIDC)   │
                         │  - Get GKE credentials  │
                         │  - Build Docker images  │
                         │  - Push to Docker Hub   │
                         └─────────────┬───────────┘
                                       │
                          ┌────────────┴────────────┐
                          ▼                         ▼
            ┌─────────────────────────┐   ┌─────────────────────────┐
            │       Docker Hub        │   │     GKE Kubernetes      │
            │  latest + SHA image tag │   │ kubectl apply + rollout │
            └─────────────────────────┘   └─────────────────────────┘
```

### Rolling Update

```text
              ┌─────────────────────────────┐
              │ Deployment: replicas = N    │
              │ strategy: maxSurge=0        │
              │           maxUnavailable=1  │
              └──────────────┬──────────────┘
                             │
                             ▼
              ┌─────────────────────────────┐
              │ 1) Terminate one old pod    │
              │    (v1)                     │
              └──────────────┬──────────────┘
                             ▼
              ┌─────────────────────────────┐
              │ 2) Start one new pod (v2)   │
              │    and wait until Ready      │
              └──────────────┬──────────────┘
                             ▼
              ┌─────────────────────────────┐
              │ 3) Repeat pod-by-pod        │
              │    until all pods are v2    │
              └─────────────────────────────┘
```
